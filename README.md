# Ansible Workspace

Рабочая база для inventory, playbooks и устанавливаемых ролей Galaxy.
Все команды ниже выполняются из `ansible-workspace/`. Роль `firewall` ставится
из своего репозитория в `roles/firewall/`.

```text
ansible.cfg
requirements.txt
requirements.yml
inventory/
  hosts.yml
  group_vars/all/firewall.yml
  host_vars/crow.r4ven.me.yml
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
Актуальная локальная роль в `../firewall/` указывает минимум Ansible Core 2.17
в `meta/main.yml`; её README называет его протестированным минимумом.
Это сведения актуальной роли, а не проверка совместимости установленной роли
или закреплённого тега `v1.0.0` в этом workspace.

```sh
export ANSIBLE_CONFIG=./ansible.cfg
python3 -m venv .venv
. .venv/bin/activate
python3 -m pip install -r requirements.txt
ansible-galaxy role install -r requirements.yml -p ./roles
```

`requirements.yml` устанавливает `firewall` из Git-репозитория
<https://github.com/r4ven-ansible/firewall> с закреплённым тегом релиза
(`version: v1.0.0`). Эквивалент одной командой:
`ansible-galaxy role install git+https://github.com/r4ven-ansible/firewall.git,v1.0.0 -p ./roles`.
Чтобы перейти на другую версию, измените `version` и выполните установку с `--force`.
`ANSIBLE_CONFIG` явно выбирает настройки workspace, даже если в окружении
ранее был указан другой конфигурационный файл.
Установленные роли игнорируются Git; в `roles/` отслеживается только `.gitkeep`.
Коллекции пока не требуются: роль использует `ansible.builtin`.

## Запуск

1. Замените документационный адрес `192.0.2.10` и пользователя в `inventory/hosts.yml`.
2. Проверьте базовую политику в `inventory/group_vars/all/firewall.yml`.
3. Укажите реальный порт sshd в `ansible_port`: `ssh_port` берётся из него.
4. Включите `firewall_enabled: true` в host vars только после проверки доступа.

```sh
ansible-inventory --graph
ansible-playbook playbooks/firewall.yml --syntax-check
ansible-playbook playbooks/firewall.yml -l crow.r4ven.me --check --diff
ansible-playbook playbooks/firewall.yml -l crow.r4ven.me
```

Если sudo требует пароль, добавьте `--ask-become-pass`. Проверка SSH host key
включена; проверьте отпечаток и установите доверие до запуска Ansible.
Секреты храните через Ansible Vault, а не в публичном inventory.

## Групповой baseline и примеры overrides

`inventory/group_vars/all/firewall.yml` содержит полный групповой baseline:
таблицы, наборы адресов, цепочки и правила input/forward/log-drop; необязательные
правила и NAT-заготовки остаются закомментированными. YAML-коллекции записаны
в block-стиле. Пустой словарь `{}` сохранён там, где это действительно пустая
коллекция; фигурные скобки внутри Jinja и строк nftables — не flow YAML.

В `inventory/host_vars/crow.r4ven.me.yml` сохранены выключенный фаервол и текущие
значения. Закомментированный `nft_rules_host` показывает три независимых операции:
добавление доступа Prometheus с `203.0.113.20` к TCP/9100, удаление `wan_allow_ssh`
через `null` и изменение IPv4 SSH flood limit на `10/minute` через recursive merge.
Выбирайте нужные операции, а не раскомментируйте всё автоматически: удаление SSH
accept может закрыть управляющий доступ. Оно не удаляет established/trusted/LAN
accept и не отменяет автоматически сгенерированные разрешения портов.

**Совместимость:** установленная сейчас роль в `roles/firewall/` объединяет
`nft_rules`, сгенерированные правила портов и `nft_rules_extra`, но не читает
`nft_rules_host`. Примеры последнего требуют версии роли, поддерживающей этот
слой (он объединяется последним). В текущей версии выбранные overrides можно
перенести в существующий `nft_rules_extra` вместо пустого словаря; не создавайте
дублирующий YAML-ключ. Эти примеры не меняют версию роли и ничего не включают.

## Важные ограничения

- Фаервол выключен по умолчанию. Пример адреса не является рабочим сервером.
- `hash_behaviour = merge` нужен для объединения `nft_rules_extra` между group
  и host vars. При `replace` словарь хоста заменит групповой целиком.
- Порядок `input`: invalid drop (`10`), ICMP/ICMPv6 redirect drops (`11/12`),
  established/related accept (`15`), WAN spoof guards (`20/21`), затем
  loopback/trusted и остальные правила. Порядок `forward`: invalid drop (`200`),
  established/related accept (`203`), WAN spoof guards (`205/206`).
  Guards намеренно идут **после established/related**, но до trusted/LAN:
  ответы от приватных DNS/NTP/gateway на uplink не отбрасываются этими guards.
  Новый/untracked трафик с приватным источником на WAN по-прежнему блокируется;
  новый управляющий доступ через приватный WAN может быть заблокирован.
  Осознанно настройте WAN-интерфейс, наборы сетей и правила; примеры `null`
  overrides и ограничения описаны в [README роли](roles/firewall/README.md).
- **Изменение поведения baseline:** раньше spoof guards проверялись до
  established/related в обеих цепочках; теперь established/related обходят их.
  Добавлены явные drops ICMP redirects до established, поэтому redirects
  отбрасываются и при состоянии related.
- **`forward` намеренно сохраняет `policy: accept`, это не default deny.**
  Новый routed-трафик, не попавший под явные drops, разрешён этой цепочкой.
  Правила `input` защищают локальный хост, но не forwarded-трафик и не
  контейнерные published ports, проходящие через `forward`.
  Для ограничения маршрутизации/контейнеров нужны отдельные forward-правила;
  другие цепочки и правила контейнерного runtime могут дополнительно фильтровать
  трафик. Закомментированный `fwd_default_drop` не активен.
- LAN-политика разрешает весь входящий трафик: добавление управляющей подсети
  в LAN — не эквивалент проверки фильтрации WAN.
- `--check` без установленного nftables показывает diff, но не проверяет синтаксис
  nft, runtime ruleset и сервис. `-t nft_apply` требует установленного nftables.
- Независимый таймер отката снижает риск потери доступа, но не заменяет снапшот
  ВМ и проверку на тестовом стенде.

Подробности DSL, rollback и бэкапов: [roles/firewall/README.md](roles/firewall/README.md).
Деструктивные VM-тесты: [roles/firewall/tests/README.md](roles/firewall/tests/README.md).
Не запускайте их на production: они блокируют SSH, удаляют nftables и могут перезагрузить ВМ.
