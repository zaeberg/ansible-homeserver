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
```
```

Для настройки сети на VM
```
  - Subnet: 192.168.122.0/24
  - Address: 192.168.122.50
  - Gateway: 192.168.122.1
  - Name servers: 1.1.1.1,8.8.8.8
  - Search domains: оставить пустым
```
```
```
