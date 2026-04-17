# Архитектура Homeserver

Этот документ описывает фактическую архитектуру, которую сейчас разворачивает репозиторий. Описание основано на текущих `playbooks`, `inventory`, ролях и шаблонах `Docker Compose`, а не на планируемом состоянии.

## Область ответственности

Сейчас проект управляет одним хостом из inventory:

- группа: `homeserver`
- узел: `freak`
- адрес: `192.168.1.103`
- пользователь Ansible: `admin`

Полная конфигурация накатывается playbook'ом `playbooks/site.yml`.

Порядок ролей:

1. `common`
2. `users`
3. `docker`
4. `ufw`
5. `traefik`
6. `glances`
7. `syncthing`
8. `homepage`

## Общая схема

```text
                   Локальная сеть / LAN
                            |
         +------------------+------------------+
         |                                     |
   HTTP 80 / Dashboard 8080              Syncthing 22000/tcp+udp
   SSH 22                                Syncthing discovery 21027/udp
         |                                     |
         +------------------+------------------+
                            |
                            v
                      UFW на хосте
            default incoming: deny, outgoing: allow
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
      |                                            +-- Homepage (Glance)
      |
      +-- Traefik
      +-- Syncthing
      +-- Homepage (Glance)
```

Ключевая идея текущей схемы:

- вход в HTTP-интерфейсы идёт через `Traefik` по host-based routing;
- для маршрутизации Traefik использует сеть `homeserver-internal`;
- прямые host-порты публикуются только там, где это действительно нужно:
  `80/tcp`, `8080/tcp`, `22000/tcp`, `22000/udp`, `21027/udp`, `22/tcp`;
- HTTPS и ACME-выдача сертификатов пока не включены, хотя заготовка под них уже есть.

## Хостовый слой

### Базовая подготовка системы

Роль `common`:

- обновляет `apt` cache;
- ставит базовые пакеты: `curl`, `git`, `ca-certificates`, `gnupg`;
- создаёт корневые директории:
  - `paths.app_root`: `/opt/apps`
  - `paths.app_data_root`: `/srv/data`

### Пользователь и SSH

Роль `users`:

- гарантирует наличие пользователя `admin`;
- добавляет его в группу `sudo`;
- устанавливает SSH public keys из `admin_public_keys`;
- при `admin_nopasswd_sudo: true` включает `NOPASSWD` sudo;
- при `ssh_harden: true`:
  - отключает парольную аутентификацию;
  - запрещает `PermitRootLogin`;
  - оставляет только `PubkeyAuthentication yes`.

### Docker

Роль `docker`:

- удаляет конфликтующие пакеты `docker.io`, `docker-compose`, `containerd`, `runc` и т.д.;
- подключает официальный Docker APT repository для Ubuntu;
- ставит:
  - `docker-ce`
  - `docker-ce-cli`
  - `containerd.io`
  - `docker-buildx-plugin`
  - `docker-compose-plugin`
- включает и запускает сервис `docker`;
- добавляет `admin` в группу `docker`;
- создаёт две пользовательские сети Docker.

### Файрвол

Роль `ufw` включает `UFW` с такими правилами:

- политика по умолчанию:
  - `incoming: deny`
  - `outgoing: allow`
  - `routed: deny`
- открыто для всех:
  - `22/tcp` для SSH
  - `80/tcp` для HTTP через Traefik
  - `22000/tcp` и `22000/udp` для Syncthing sync
- открыто только из LAN `192.168.1.0/24`:
  - `8080/tcp` для прямого доступа к Traefik dashboard
  - `21027/udp` для локального discovery Syncthing
- удаляется устаревшее правило для `8384/tcp`

Это означает, что GUI Syncthing не торчит наружу напрямую: он доступен через Traefik, а не через host port `8384`.

## Docker-сети

Общие сети задаются в `inventory/group_vars/all.yml`.

### `homeserver-internal`

Параметры:

- имя: `homeserver-internal`
- драйвер: `bridge`
- `internal: true`

Назначение:

- основной east-west трафик между контейнерами;
- сеть, которую Traefik использует для доступа к backend-сервисам;
- сеть по умолчанию для внутренних web UI.

Важно:

- в Traefik явно задан `providers.docker.network: homeserver-internal`;
- у сервисов с Traefik-роутами также выставлен label `traefik.docker.network={{ docker_networks.internal.name }}`;
- значит, фактическая маршрутизация идёт именно через `homeserver-internal`, даже если контейнер дополнительно подключён к `homeserver-public`.

### `homeserver-public`

Параметры:

- имя: `homeserver-public`
- драйвер: `bridge`
- `internal: false`

Назначение:

- сеть для контейнеров, которым потенциально нужен внешний сетевой доступ;
- сеть, к которой подключён Traefik как входная точка.

В текущем состоянии к ней подключены:

- `Traefik`
- `Syncthing`
- `Homepage`

`Glances` работает только во внутренней сети.

## Каталог сервисов

### Traefik

- образ: `traefik:v3.6.7`
- контейнер: `homeserver-traefik`
- compose project: `homelab-traefik`
- директория стека: `/opt/apps/traefik`
- конфиги:
  - `/opt/apps/traefik/config/traefik.yml`
  - `/opt/apps/traefik/config/dynamic.yml`
  - `/opt/apps/traefik/config/letsencrypt/`
- сети:
  - `homeserver-internal`
  - `homeserver-public`

Функции:

- reverse proxy для всех HTTP-интерфейсов;
- Docker provider с `exposedByDefault: false`;
- file provider для динамической конфигурации;
- роутинг dashboard по `Host("traefik.home.local")`;
- redirect `/` -> `/dashboard/` через middleware `dashboard-to-root`;
- включён `ping` для healthcheck.

Порты хоста:

- `80:80`
- `{{ actual_ip }}:8080:8080`, где `actual_ip` вычисляется из `ansible_facts['default_ipv4']['address']`
- `443:443` пока не публикуется, потому что `traefik_enable_https: false`

Текущий доступ:

- через hostname: `http://traefik.home.local/dashboard/`
- напрямую из LAN: `http://<ip_хоста>:8080/dashboard/`

Ограничения текущей конфигурации:

- `api.insecure: true`
- basic auth для dashboard не настроен
- middleware `security-headers` определён, но в шаблонах сервисов пока не используется
- каталог `letsencrypt` создаётся заранее, но TLS/ACME пока выключены

### traefik-avahi-helper

- образ: `hardillb/traefik-avahi-helper:latest`
- контейнер: `homeserver-traefik-avahi-helper`
- сеть: `homeserver-internal`

Назначение:

- интеграция с `avahi-daemon` и публикация сервисов в локальной сети через mDNS/Bonjour;
- дополнение к схеме с доменами вида `*.home.local`.

Хостовые зависимости:

- пакет `avahi-daemon`
- пакет `dbus`
- systemd service `avahi-daemon` в состоянии `enabled` и `started`

Особенности контейнера:

- `ipc: host`
- `apparmor:unconfined`
- `seccomp:unconfined`
- `cap_add: NET_ADMIN, NET_RAW`
- монтируются:
  - `/var/run/docker.sock:ro`
  - `/var/run/dbus/system_bus_socket`

Примечание:

- в самом репозитории нет отдельного набора `avahi.*` labels для приложений;
- логика публикации `.home.local` завязана на поведение helper-контейнера и Traefik-роутов.

### Glances

- образ: `nicolargo/glances:4.4.1`
- контейнер: `homeserver-glances`
- compose project: `homeserver-glances`
- hostname сервиса: `glances.home.local`
- внутренний web port: `61208`
- директория стека: `/opt/apps/glances`
- конфиг: `/opt/apps/glances/config/glances.conf`
- сеть контейнера: только `homeserver-internal`

Функции:

- мониторинг CPU, RAM, дисков, сети и процессов хоста;
- мониторинг Docker через read-only доступ к `docker.sock`;
- публикация web UI через Traefik.

Особенности запуска:

- `pid: host`, чтобы видеть процессы хоста;
- bind mounts:
  - `/:/rootfs:ro`
  - `/var/run/docker.sock:/var/run/docker.sock:ro`
  - `/etc/os-release:/etc/os-release:ro`
- `tmpfs: /tmp`

Доступ:

- `http://glances.home.local`

### Syncthing

- образ: `syncthing/syncthing:2.0.16`
- контейнер: `homeserver-syncthing`
- compose project: `homeserver-syncthing`
- hostname UI: `syncthing.home.local`
- директории:
  - стек: `/opt/apps/syncthing`
  - данные: `/srv/data/syncthing`
  - дефолтная синхронизируемая папка: `/srv/data/syncthing/Sync`
- сети:
  - `homeserver-internal`
  - `homeserver-public`

Функции:

- peer-to-peer синхронизация файлов между устройствами;
- web GUI публикуется через Traefik;
- sync/discovery порты публикуются напрямую на хост.

Порты хоста:

- `22000/tcp`
- `22000/udp`
- `21027/udp`

GUI:

- слушает внутри контейнера `0.0.0.0:8384`
- доступен снаружи через `http://syncthing.home.local`
- прямой host port `8384` не публикуется

Управление конфигом:

- если пароль GUI не задан явно, роль генерирует его при первом деплое;
- credential-файлы сохраняются в:
  - `/opt/apps/syncthing/gui-user.txt`
  - `/opt/apps/syncthing/gui-password.txt`
- `config.xml` создаётся через `docker run ... generate`, затем правится Ansible-модулем `community.general.xml`

Жёстко выставляемые настройки:

- `insecureAdminAccess = false`
- `startBrowser = false`
- `autoUpgradeIntervalH = 0`
- настраиваются `globalAnnounceEnabled`, `localAnnounceEnabled`, `localAnnouncePort`, `relaysEnabled`, `natEnabled`

### Homepage

В проекте роль называется `homepage`, но фактически разворачивает приложение `Glance`.

- образ: `glanceapp/glance:v0.8.4`
- контейнер: `homeserver-homepage`
- compose project: `homeserver-homepage`
- hostname: `home.local`
- внутренний web port: `8080`
- директории:
  - стек: `/opt/apps/homepage`
  - конфиг: `/opt/apps/homepage/config`
  - assets: `/opt/apps/homepage/assets`
- сети:
  - `homeserver-internal`
  - `homeserver-public`

Функции:

- домашняя стартовая страница для локальной сети;
- показывает сервисный мониторинг для:
  - Traefik
  - Glances
  - Syncthing
- содержит погодный виджет для Kaliningrad;
- работает за Traefik по hostname `home.local`.

Артефакты, которые роль копирует на хост:

- каталог `config/`
- каталог `assets/`
- файл `.env`
- файл `compose.yml`

Доступ:

- `http://home.local`

## Потоки трафика

### 1. Доступ к локальным web UI

Сценарий для `home.local`, `glances.home.local`, `syncthing.home.local`, `traefik.home.local`:

1. клиент в LAN идёт на hostname сервиса;
2. запрос приходит на хост по `80/tcp`;
3. UFW пропускает порт `80`;
4. `Traefik` принимает HTTP-запрос;
5. Traefik находит нужный router по Docker labels;
6. backend-сервис выбирается в сети `homeserver-internal`;
7. ответ возвращается клиенту.

### 2. Прямой доступ к Traefik dashboard

1. клиент в LAN идёт на `http://<ip_хоста>:8080/dashboard/`;
2. порт `8080/tcp` разрешён только из `192.168.1.0/24`;
3. запрос попадает напрямую в контейнер `Traefik`.

Этот путь отдельный от hostname `traefik.home.local` и существует как запасной вариант диагностики.

### 3. Синхронизация Syncthing

1. внешние peers или локальные устройства подключаются к `22000/tcp` и `22000/udp`;
2. локальное discovery использует `21027/udp`;
3. трафик попадает напрямую в контейнер `Syncthing`, минуя Traefik;
4. web GUI Syncthing при этом остаётся за reverse proxy.

## Раскладка данных на диске

### `/opt/apps`

Здесь лежат compose-файлы и статические артефакты приложений:

- `/opt/apps/traefik`
- `/opt/apps/glances`
- `/opt/apps/syncthing`
- `/opt/apps/homepage`

### `/srv/data`

Здесь лежат runtime-данные приложений:

- `/srv/data/syncthing`
- `/srv/data/syncthing/Sync`

Пока это единственный сервис в репозитории, который хранит прикладные данные в `paths.app_data_root`.

## Ограничения и текущее состояние

Подготовлено, но ещё не доведено до боевого состояния:

- каталог для Let's Encrypt уже создаётся;
- в dynamic config Traefik уже есть `security-headers`;
- схема с `.home.local` опирается на `traefik-avahi-helper`, но отдельная repo-level конфигурация публикации сервисов минимальна.

## Краткий итог

Текущая архитектура — это один Ubuntu-хост с Docker, двумя пользовательскими сетями и Traefik в роли единой HTTP-точки входа. Поверх него развернуты три прикладных сервиса:

- `Homepage` как стартовая страница;
- `Glances` как мониторинг;
- `Syncthing` как сервис синхронизации.

Безопасность на базовом уровне обеспечивается через `UFW`, SSH hardening и отказ от прямой публикации web UI большинства сервисов. При этом TLS, аутентификация Traefik dashboard и более строгая mDNS/edge-конфигурация пока остаются следующими шагами развития.
