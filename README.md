Для запуска с преварительной проверкой без применения изменений
```
ansible-playbook playbooks/site.yml --check --diff --vault-password-file .vault_pass.txt
```

Для запуска с применением изменений, если нужен пароль к sudo
```
ansible-playbook playbooks/site.yml --ask-become-pass --vault-password-file .vault_pass.txt
```

Для запуска с применением изменений, если НЕ нужен пароль к sudo
```
ansible-playbook playbooks/site.yml --vault-password-file .vault_pass.txt
```

Для обновления сикретов в ansible-vault
```
ansible-vault edit --vault-password-file .vault_pass.txt inventory/group_vars/homeserver/90_vault.yml
```

Для настройки сети на VM
```
  - Subnet: 192.168.122.0/24
  - Address: 192.168.122.50
  - Gateway: 192.168.122.1
  - Name servers: 1.1.1.1,8.8.8.8
  - Search domains: оставить пустым
```

## Syncthing
Экземпляр Syncthing должен быть установлен на всех девайсах где нужна синхронизация папок
Идея в том что на хоумсервере лежит оригинальная папка являющаяся источником истины, а все остальные девайсы через нее синхронизируются подкулючаясь к ней. Сейчас это работает для Obsidian, но в будущем когда появится полноценный NAS можно будет синхронизировать папки с него.

### Omarchy
1. Установить Syncthing из пакета дистрибутива.
2. Запустить user-service, чтобы он жил в твоей сессии и стартовал автоматически.
3. Открыть локальный UI, обычно http://127.0.0.1:8384.
4. Сразу задать GUI user/password.
5. Убедиться, что GUI слушает только локально, если тебе не нужен доступ к нему с других устройств.

```
  sudo pacman -S syncthing
  systemctl --user enable --now syncthing.service
  loginctl enable-linger "$USER"
```

### MacOS
1. Поставить Syncthing через brew или готовое app/service-решение.
2. Настроить автозапуск через brew services или launchd.
3. Открыть локальный UI.
4. Задать GUI user/password.
5. Не публиковать его наружу.

```
  brew install syncthing
  brew services start syncthing
```

### Ios & IpadOs
В текущей схеме из-за ограничений самой apple и песочницы для приложений синхроназиация для этих устройств происходит через icloud. То есть папка на macos настроена так что бы попадать одновременно и в syncthing и в icloud. Поэтому между Omarchy и Macos синхронизация присходит по syncthing а между устройствами apple экосистемы через icloud после попадания туда файлов. 

