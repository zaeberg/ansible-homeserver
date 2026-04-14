Для запуска с преварительной проверкой без применения изменений
```
ansible-playbook playbooks/site.yml --check --diff
```

Для запуска с применением изменений, если нужен пароль к sudo
```
ansible-playbook playbooks/site.yml --ask-become-pass
```

Для запуска с применением изменений, если НЕ нужен пароль к sudo
```
ansible-playbook playbooks/site.yml
```

