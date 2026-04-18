# Ansible Vault

Документ описывает принятую схему работы с `ansible-vault` в этом репозитории: где хранить зашифрованные переменные, какие значения выносить в Vault и как применять их при обычной эксплуатации.

## Назначение

`Ansible Vault` используется для хранения чувствительных значений в зашифрованном виде. Это относится к паролям, токенам и другим секретам, которые нужны playbook'ам, но не должны храниться в открытом виде в inventory или в репозитории.

Vault не заменяет сами секреты. Он только задаёт формат их безопасного хранения в Ansible.

## Базовые понятия

Нужно различать две разные вещи:

- секрет сервиса или хоста:
  - пароль `Traefik` dashboard
  - пароль `Syncthing` GUI
  - пароль пользователя `admin`
  - API tokens
- пароль от Vault:
  - ключ, который используется для шифрования и расшифровки vaulted-файла

Пароль от Vault не должен совпадать с паролем какого-либо сервиса.

## Принятая структура

Для зашифрованных переменных используется файл:

- `inventory/group_vars/homeserver/90_vault.yml`

Выбор именно этой директории связан с тем, что хост находится в группе `homeserver`, а файл `inventory/group_vars/all.yml` уже используется для обычных общих переменных.

Локальный файл с паролем от Vault, если он используется, хранится отдельно:

- `.vault_pass.txt`

Оба файла считаются локальными эксплуатационными артефактами и не должны попадать в git.

## Что выносить в Vault

### Обязательный минимум

Для текущей конфигурации в Vault целесообразно хранить:

- `traefik_dashboard_auth_password`
- `syncthing_gui_password`
- `admin_password_hash`
- `homepage_env`

### По необходимости

Дополнительно можно хранить:

- `ansible_become_password`
- будущие API keys, tokens и production-значения из `.env`

## Почему для `admin` используется hash

Для Linux-пароля пользователя `admin` используется переменная:

- `admin_password_hash`

Роль `users` умеет применять этот hash напрямую при создании или обновлении пользователя. Это предпочтительнее, чем хранить plaintext-пароль пользователя в inventory.

Исходный человеческий пароль при этом должен храниться в password manager, а в Vault попадает только hash.

Для `Syncthing` используется другой подход: в Vault хранится обычный plaintext-пароль `syncthing_gui_password`, а hash уже создаёт сам `Syncthing` при записи в `config.xml`.

## Где хранить исходные значения

Рекомендуемая практика:

- реальные пароли и passphrase хранить в password manager;
- в Vault хранить только те значения, которые нужны Ansible во время деплоя;
- локальный `.vault_pass.txt` использовать только как операторское удобство на своей машине.

Не рекомендуется:

- коммитить `.vault_pass.txt`;
- хранить секреты в `inventory/group_vars/all.yml`;
- использовать один и тот же пароль и как Vault password, и как пароль приложения;
- хранить обычный пароль пользователя `admin` в открытом виде.

## Подготовка

### 1. Создать локальный пароль от Vault

Пароль от Vault выбирается отдельно от всех сервисных паролей.

Пример локального файла:

```bash
printf '%s\n' 'VAULT_PASSWORD' > .vault_pass.txt
chmod 600 .vault_pass.txt
```

Файл используется только локально и не должен публиковаться.

### 2. Создать директорию для vaulted vars

```bash
mkdir -p inventory/group_vars/homeserver
```

### 3. Получить hash для пароля пользователя `admin`

Пример:

```bash
openssl passwd -6
```

Команда попросит ввести пароль и вернёт строку формата `$6$...`. Именно эта строка и должна попасть в `admin_password_hash`.

## Создание vaulted-файла

Вариант с ручным вводом Vault password:

```bash
ansible-vault create inventory/group_vars/homeserver/90_vault.yml
```

Вариант с локальным `.vault_pass.txt`:

```bash
ansible-vault create --vault-password-file .vault_pass.txt inventory/group_vars/homeserver/90_vault.yml
```

## Содержимое vaulted-файла

Минимальный рабочий пример:

```yaml
traefik_dashboard_auth_password: "STRONG_TRAEFIK_PASSWORD"
admin_password_hash: "$6$..."
homepage_env:
  MY_SECRET_TOKEN: "STRONG_HOMEPAGE_TOKEN"
```

Расширенный пример:

```yaml
traefik_dashboard_auth_password: "STRONG_TRAEFIK_PASSWORD"
admin_password_hash: "$6$..."
homepage_env:
  MY_SECRET_TOKEN: "STRONG_HOMEPAGE_TOKEN"
syncthing_gui_password: "STRONG_SYNCTHING_PASSWORD"
# ansible_become_password: "ADMIN_SUDO_PASSWORD"
```

## Применение playbook

### Если Vault password вводится вручную

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Если `admin_nopasswd_sudo` уже выключен и `ansible_become_password` не хранится в Vault:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass --ask-become-pass
```

### Если используется локальный `.vault_pass.txt`

```bash
ansible-playbook playbooks/site.yml --vault-password-file .vault_pass.txt
```

Если `admin_nopasswd_sudo` уже выключен и `ansible_become_password` не хранится в Vault:

```bash
ansible-playbook playbooks/site.yml --vault-password-file .vault_pass.txt --ask-become-pass
```

## Обслуживание vaulted-файла

Редактирование:

```bash
ansible-vault edit --vault-password-file .vault_pass.txt inventory/group_vars/homeserver/90_vault.yml
```

Просмотр:

```bash
ansible-vault view --vault-password-file .vault_pass.txt inventory/group_vars/homeserver/90_vault.yml
```

Смена Vault password:

```bash
ansible-vault rekey --vault-password-file .vault_pass.txt inventory/group_vars/homeserver/90_vault.yml
```

## Использование в текущей конфигурации

Vault в этой конфигурации используется для следующих значений:

- `traefik_dashboard_auth_password` задаёт пароль для `Traefik` dashboard;
- `syncthing_gui_password` задаёт пароль для `Syncthing` GUI;
- `homepage_env` формирует runtime `.env` для `Homepage`;
- `admin_password_hash` задаёт управляемый hash локального пароля пользователя `admin`;
- `ansible_become_password` может использоваться, если `admin_nopasswd_sudo` будет выключен.
