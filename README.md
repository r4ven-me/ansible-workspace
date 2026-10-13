# Ansible Workspace

Рабочая база для inventory, playbooks и устанавливаемых ролей Galaxy.
Все команды ниже выполняются из `ansible-workspace/`. Самостоятельная исходная
роль находится в соседнем `../firewall/`; установленная копия — в `roles/firewall/`.

```text
ansible.cfg
requirements.txt
requirements.yml
inventory/
  hosts.yml
  group_vars/all/firewall.yml
  host_vars/firewall_example.yml
playbooks/firewall.yml
roles/firewall/
  defaults/
  meta/
  tasks/
  templates/
  tests/
  README.md
```

## Подготовка

На управляющей машине нужны Python 3.11+ и SSH. На целевом хосте — Debian/LMDE,
Python 3, systemd и sudo. Пример использует Ansible Core 2.19.

```sh
export ANSIBLE_CONFIG=./ansible.cfg
python3 -m venv .venv
. .venv/bin/activate
python3 -m pip install -r requirements.txt
mkdir -p .ansible
tar --exclude='./.ansible' --exclude='./.venv' --exclude='./.git' --exclude='./__pycache__' -czf .ansible/firewall.tar.gz -C ../firewall .
ansible-galaxy install -r requirements.yml -p ./roles
```

`requirements.yml` устанавливает `firewall` из локального архива соседней роли.
После изменения исходников пересоздайте архив и выполните
`ansible-galaxy role install -r requirements.yml -p ./roles --force`.
`ANSIBLE_CONFIG` явно выбирает настройки workspace, даже если в окружении
ранее был указан другой конфигурационный файл.
Установленные роли игнорируются Git; в `roles/` отслеживается только `.gitkeep`.
Коллекции пока не требуются: роль использует `ansible.builtin`.
После публикации замените `src` на Git URL, добавьте `scm: git` и
`version` с закреплённым тегом релиза.

## Запуск

1. Замените документационный адрес `192.0.2.10` и пользователя в `inventory/hosts.yml`.
2. Проверьте базовую политику в `inventory/group_vars/all/firewall.yml`.
3. Укажите реальный порт sshd в `ansible_port`: `ssh_port` берётся из него.
4. Включите `firewall_enabled: true` в host vars только после проверки доступа.

```sh
ansible-inventory --graph
ansible-playbook playbooks/firewall.yml --syntax-check
ansible-playbook playbooks/firewall.yml -l firewall_example --check --diff
ansible-playbook playbooks/firewall.yml -l firewall_example
```

Если sudo требует пароль, добавьте `--ask-become-pass`. Проверка SSH host key
включена; проверьте отпечаток и установите доверие до запуска Ansible.
Секреты храните через Ansible Vault, а не в публичном inventory.

## Важные ограничения

- Фаервол выключен по умолчанию. Пример адреса не является рабочим сервером.
- `hash_behaviour = merge` нужен для объединения `nft_rules_extra` между group
  и host vars. При `replace` словарь хоста заменит групповой целиком.
- Базовая политика блокирует приватные источники на WAN **до** established/trusted.
  На сервере с WAN в приватной сети это может заблокировать администратора.
  Осознанно настройте WAN-интерфейс, наборы сетей и правила; примеры `null`
  overrides и ограничения описаны в [README роли](roles/firewall/README.md).
- LAN-политика разрешает весь входящий трафик: добавление управляющей подсети
  в LAN — не эквивалент проверки фильтрации WAN.
- `--check` без установленного nftables показывает diff, но не проверяет синтаксис
  nft, runtime ruleset и сервис. `-t nft_apply` требует установленного nftables.
- Независимый таймер отката снижает риск потери доступа, но не заменяет снапшот
  ВМ и проверку на тестовом стенде.

Подробности DSL, rollback и бэкапов: [roles/firewall/README.md](roles/firewall/README.md).
Деструктивные VM-тесты: [roles/firewall/tests/README.md](roles/firewall/tests/README.md).
Не запускайте их на production: они блокируют SSH, удаляют nftables и могут перезагрузить ВМ.
