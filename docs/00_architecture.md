# Архитектура Homeserver

Документ описывает текущее состояние homeserver, которое разворачивается из этого репозитория. Это эксплуатационное описание фактической конфигурации: хост, роли Ansible, сетевые границы, сервисы и схема доступа.

## Общая картина

Репозиторий управляет одним хостом:

- группа inventory: `homeserver`
- имя узла: `freak`
- адрес: `192.168.1.103`
- пользователь Ansible: `admin`

Полная конфигурация применяется через `playbooks/site.yml`.

Порядок ролей:

1. `common`
2. `users`
3. `docker`
4. `ufw`
5. `traefik`
6. `glances`
7. `syncthing`
8. `homepage`

## Сетевая модель

Сервер рассчитан на работу внутри домашней сети. Доступ к web-интерфейсам организован через `Traefik` по hostname-маршрутизации. Публичная публикация сервисов наружу не используется.

```text
                    Локальная сеть / LAN
                             |
        +--------------------+--------------------+
        |                                         |
   HTTP 80 / SSH 22                         Syncthing 22000/tcp+udp
   Dashboard через Traefik auth             Syncthing discovery 21027/udp
        |                                         |
        +--------------------+--------------------+
                             |
                             v
                       UFW на хосте
             incoming: deny, outgoing: allow, routed: deny
                             |
                             v
                    Ubuntu-хост с Docker
                             |
       +---------------------+----------------------+
       |                                            |
       v                                            v
   homeserver-public                         homeserver-internal
   bridge, internal: false                   bridge, internal: true
       |                                            |
       |                                            +-- Traefik
       |                                            +-- traefik-avahi-helper
       |                                            +-- Glances
       |                                            +-- Syncthing
       |                                            +-- Homepage
       |
       +-- Traefik
       +-- Syncthing
       +-- Homepage
```

Основные принципы:

- HTTP-интерфейсы доступны только через `Traefik`;
- для связи reverse proxy с backend-сервисами используется сеть `homeserver-internal`;
- прямые host-порты открыты только там, где это необходимо:
  - `22/tcp`
  - `80/tcp`
  - `22000/tcp`
  - `22000/udp`
  - `21027/udp`
- HTTPS и ACME пока не включены;
- `Traefik` dashboard публикуется только через hostname и защищён `Basic Auth`.

## Хостовый слой

### Базовая подготовка

Роль `common`:

- обновляет индекс пакетов;
- устанавливает базовые пакеты:
  - `curl`
  - `git`
  - `ca-certificates`
  - `gnupg`
- создаёт базовые директории:
  - `paths.app_root`: `/opt/apps`
  - `paths.app_data_root`: `/srv/data`

### Пользователь и SSH

Роль `users`:

- гарантирует наличие пользователя `admin`;
- добавляет `admin` в группу `sudo`;
- устанавливает SSH public keys из `admin_public_keys`;
- при наличии `admin_password_hash` задаёт пароль пользователя `admin`;
- при `admin_nopasswd_sudo: true` сохраняет для `admin` режим `NOPASSWD sudo`;
- при `ssh_harden: true`:
  - отключает парольную аутентификацию;
  - запрещает `PermitRootLogin`;
  - оставляет `PubkeyAuthentication yes`.

### Docker

Роль `docker`:

- удаляет конфликтующие пакеты из дистрибутива;
- подключает официальный Docker APT repository;
- устанавливает:
  - `docker-ce`
  - `docker-ce-cli`
  - `containerd.io`
  - `docker-buildx-plugin`
  - `docker-compose-plugin`
- включает и запускает сервис `docker`;
- добавляет `admin` в группу `docker`;
- создаёт две пользовательские сети Docker.

### Файрвол

Роль `ufw` включает `UFW` с политикой:

- `incoming: deny`
- `outgoing: allow`
- `routed: deny`

Разрешённые порты в текущем состоянии:

- только из `192.168.1.0/24`:
  - `22/tcp` для SSH
  - `80/tcp` для HTTP через Traefik
  - `22000/tcp` и `22000/udp` для Syncthing sync
  - `21027/udp` для локального discovery Syncthing

Роль также удаляет устаревшие правила, если они остались от предыдущих конфигураций:

- публичные правила для `22/tcp`, `80/tcp`, `22000/tcp`, `22000/udp`
- старый LAN-only доступ к `8080/tcp`
- устаревшее правило `8384/tcp`

## Docker-сети

Сети задаются в `inventory/group_vars/all.yml`.

### `homeserver-internal`

Параметры:

- имя: `homeserver-internal`
- драйвер: `bridge`
- `internal: true`

Назначение:

- трафик между контейнерами;
- доступ `Traefik` к backend-сервисам;
- внутренняя сеть для web UI и служебного взаимодействия.

`Traefik` использует именно эту сеть для маршрутизации:

- в static config задан `providers.docker.network: homeserver-internal`;
- у сервисов с Traefik-роутами выставлен label `traefik.docker.network`.

### `homeserver-public`

Параметры:

- имя: `homeserver-public`
- драйвер: `bridge`
- `internal: false`

Назначение:

- сеть для контейнеров, которым нужен внешний сетевой контур на уровне Docker;
- входная сеть для `Traefik`.

Подключённые сервисы:

- `Traefik`
- `Syncthing`
- `Homepage`

`Glances` работает только во внутренней сети.

## Сервисы

### Traefik

- образ: `traefik:v3.6.7`
- контейнер: `homeserver-traefik`
- compose project: `homelab-traefik`
- директория стека: `/opt/apps/traefik`

Конфигурационные файлы:

- `/opt/apps/traefik/config/traefik.yml`
- `/opt/apps/traefik/config/dynamic.yml`
- `/opt/apps/traefik/config/dashboard-users.htpasswd`
- `/opt/apps/traefik/config/letsencrypt/`

Назначение:

- единая HTTP-точка входа для всех web-интерфейсов;
- автоматическое обнаружение сервисов через Docker labels;
- маршрутизация по hostname;
- доступ к dashboard через `api@internal`.

Текущая конфигурация:

- `api.insecure: false`
- `dashboard` доступен по `http://traefik.home.local/dashboard/`
- доступ к dashboard защищён `Basic Auth`
- `/` перенаправляется на `/dashboard/`
- для dashboard используется middleware `security-headers`
- `ping` доступен на entrypoint `web` и используется healthcheck'ом контейнера

Порты хоста:

- `80:80`

HTTPS entrypoint пока не публикуется, потому что `traefik_enable_https: false`.

### traefik-avahi-helper

- образ: `hardillb/traefik-avahi-helper:latest`
- контейнер: `homeserver-traefik-avahi-helper`
- сеть: `homeserver-internal`

Назначение:

- публикация локальных сервисов через mDNS/Bonjour;
- поддержка обращений к сервисам по именам вида `*.home.local`.

Особенности контейнера:

- `ipc: host`
- `apparmor:unconfined`
- `seccomp:unconfined`
- `cap_add: NET_ADMIN, NET_RAW`
- используется `docker.sock` и системный D-Bus socket

На хосте для этого дополнительно установлены и запущены:

- `avahi-daemon`
- `dbus`

### Glances

- образ: `nicolargo/glances:4.4.1`
- контейнер: `homeserver-glances`
- compose project: `homeserver-glances`
- hostname: `glances.home.local`
- внутренний web port: `61208`
- директория стека: `/opt/apps/glances`
- конфиг: `/opt/apps/glances/config/glances.conf`

Назначение:

- мониторинг CPU, памяти, дисков, сети и процессов хоста;
- публикация web UI через `Traefik`.

Особенности запуска:

- `pid: host`
- bind mounts:
  - `/:/rootfs:ro`
  - `/var/run/docker.sock:/var/run/docker.sock:ro`
  - `/etc/os-release:/etc/os-release:ro`
- `tmpfs: /tmp`

### Syncthing

- образ: `syncthing/syncthing:2.0.16`
- контейнер: `homeserver-syncthing`
- compose project: `homeserver-syncthing`
- hostname UI: `syncthing.home.local`
- стек: `/opt/apps/syncthing`
- данные: `/srv/data/syncthing`
- папка синхронизации по умолчанию: `/srv/data/syncthing/Sync`

Назначение:

- синхронизация файлов между локальными устройствами;
- web GUI через `Traefik`;
- прямые sync/discovery порты на хосте.

Порты хоста:

- `22000/tcp`
- `22000/udp`
- `21027/udp`

GUI:

- слушает внутри контейнера `0.0.0.0:8384`
- доступен в LAN по `http://syncthing.home.local`
- host port `8384` не публикуется

Конфигурация GUI:

- при отсутствии явно заданного пароля роль генерирует пароль при первом деплое;
- логин и пароль сохраняются в:
  - `/opt/apps/syncthing/gui-user.txt`
  - `/opt/apps/syncthing/gui-password.txt`

Принудительно задаваемые параметры:

- `insecureAdminAccess = false`
- `startBrowser = false`
- `autoUpgradeIntervalH = 0`
- `globalAnnounceEnabled = false`
- `localAnnounceEnabled = true`
- `localAnnouncePort = 21027`
- `relaysEnabled = false`
- `natEnabled = false`

### Homepage

Роль называется `homepage`, но разворачивает приложение `Glance`.

- образ: `glanceapp/glance:v0.8.4`
- контейнер: `homeserver-homepage`
- compose project: `homeserver-homepage`
- hostname: `home.local`
- внутренний web port: `8080`
- директория стека: `/opt/apps/homepage`
- конфиг: `/opt/apps/homepage/config`
- assets: `/opt/apps/homepage/assets`

Назначение:

- стартовая страница для локальной сети;
- агрегированный доступ к основным сервисам;
- сервисный мониторинг для `Traefik`, `Glances` и `Syncthing`.

Особенности:

- публикуется через `Traefik`;
- по умолчанию не получает доступ к `docker.sock`;
- использует локальные конфиги, assets и runtime `.env`, разворачиваемые ролью на хост.

## Потоки трафика

### Web-интерфейсы

Для `home.local`, `glances.home.local`, `syncthing.home.local` и `traefik.home.local` используется одна схема:

1. клиент в локальной сети обращается к hostname сервиса;
2. запрос приходит на хост по `80/tcp`;
3. `UFW` пропускает трафик только из LAN-подсети;
4. `Traefik` выбирает router по Docker labels;
5. backend открывается через сеть `homeserver-internal`;
6. ответ возвращается клиенту.

### Dashboard Traefik

Сценарий доступа:

1. клиент открывает `http://traefik.home.local/`;
2. `Traefik` перенаправляет запрос на `/dashboard/`;
3. middleware `dashboard-auth` требует логин и пароль;
4. после успешной аутентификации открывается `api@internal`.

### Синхронизация Syncthing

Сценарий обмена:

1. локальные устройства работают через `22000/tcp` и `22000/udp`;
2. локальное discovery использует `21027/udp`;
3. sync-трафик идёт напрямую в контейнер `Syncthing`;
4. web GUI при этом остаётся за reverse proxy.

## Данные на диске

### `/opt/apps`

Здесь лежат compose-файлы и конфигурация стеков:

- `/opt/apps/traefik`
- `/opt/apps/glances`
- `/opt/apps/syncthing`
- `/opt/apps/homepage`

### `/srv/data`

Здесь лежат runtime-данные приложений:

- `/srv/data/syncthing`
- `/srv/data/syncthing/Sync`

Сейчас прикладные данные в `paths.app_data_root` хранит только `Syncthing`.

## Эксплуатационные замечания

- Каталог для Let's Encrypt уже создаётся, но TLS и ACME не включены.
- Доступ к сервисам рассчитан на локальную сеть и/или VPN-модель, а не на прямую публикацию наружу.
- `admin_nopasswd_sudo` пока остаётся включённым, пока не будет введён управляемый `sudo`-пароль через Vault.
