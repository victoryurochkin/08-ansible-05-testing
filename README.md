# Домашнее задание к занятию 5 «Тестирование roles» - Юрочкин В.А.

## Описание

В рамках домашнего задания настроено тестирование Ansible role `vector-role` с использованием Molecule и Tox.

Цель работы:

- изучить пример Molecule-сценариев на готовой роли ClickHouse;
- добавить Molecule-сценарий для роли Vector;
- протестировать роль Vector на нескольких дистрибутивах;
- добавить проверки в `verify.yml`;
- создать облегчённый Molecule-сценарий с Podman;
- настроить запуск тестирования через Tox;
- зафиксировать рабочие состояния роли тегами согласно семантическому версионированию.

## Репозитории

| Назначение | Ссылка |
|---|---|
| Репозиторий с отчётом | <https://github.com/victoryurochkin/08-ansible-05-testing> |
| Репозиторий с ролью Vector | <https://github.com/victoryurochkin/vector-role> |
| Ветка с решением | <https://github.com/victoryurochkin/vector-role/tree/08-ansible-05-testing> |

## Теги решения

| Этап | Тег | Ссылка |
|---|---|---|
| Molecule Docker scenario | `1.1.0` | <https://github.com/victoryurochkin/vector-role/releases/tag/1.1.0> |
| Tox + Molecule Podman scenario | `1.2.0` | <https://github.com/victoryurochkin/vector-role/releases/tag/1.2.0> |

## Используемое окружение

Тестирование выполнялось на виртуальной машине Ubuntu 22.04.

Используемые инструменты:

- Docker;
- Podman;
- Python 3;
- Ansible;
- Molecule;
- molecule-docker;
- molecule-podman;
- Tox;
- yamllint;
- ansible-lint.

Использовался учебный Docker-образ:

```bash
docker pull aragast/netology:latest
```

## Подготовка роли

Для выполнения задания использовалась роль:

```text
https://github.com/victoryurochkin/vector-role
```

Для работы над заданием создана отдельная ветка:

```bash
git checkout -b 08-ansible-05-testing
```

## Структура роли после выполнения задания

После выполнения задания в роли появились два Molecule-сценария и файл `tox.ini`.

```text
vector-role/
├── defaults
│   └── main.yml
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   ├── default
│   │   ├── converge.yml
│   │   ├── molecule.yml
│   │   ├── prepare.yml
│   │   └── verify.yml
│   └── podman
│       ├── converge.yml
│       ├── molecule.yml
│       ├── prepare.yml
│       └── verify.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
│   └── vector.yaml.j2
├── tests
│   ├── inventory
│   └── test.yml
├── tox.ini
└── vars
    └── main.yml
```

<img width="1181" height="1285" alt="image" src="https://github.com/user-attachments/assets/4cc66b91-4d41-4ddb-8f2d-feb45b379e6b" />


## Molecule Docker scenario

Основной Molecule-сценарий расположен в каталоге:

```text
molecule/default/
```

В сценарии используется Docker-драйвер.

Файл `molecule/default/molecule.yml` содержит два тестовых инстанса:

```yaml
platforms:
  - name: vector-ubuntu
    image: ubuntu:latest
    pre_build_image: true
    command: /bin/bash
    tty: true

  - name: vector-oraclelinux
    image: oraclelinux:8
    pre_build_image: true
    command: /bin/bash
    tty: true
```

Для подготовки контейнеров используется `prepare.yml`. В нём устанавливаются необходимые зависимости для выполнения Ansible-модулей внутри контейнеров.

Файл `molecule/default/converge.yml` подключает текущую роль:

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Include vector role
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}"
```

## Verify-проверки

В `verify.yml` добавлены проверки работоспособности роли Vector.

Проверяется:

- наличие бинарного файла Vector;
- успешное выполнение команды `vector --version`;
- наличие конфигурационного файла `/etc/vector/vector.yaml`;
- валидность конфигурации Vector;
- успешное выполнение assert-проверок.

Фрагмент `verify.yml`:

```yaml
---
- name: Verify Vector role
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Check Vector binary exists
      ansible.builtin.command: vector --version
      register: vector_version_result
      changed_when: false

    - name: Assert Vector version command is successful
      ansible.builtin.assert:
        that:
          - vector_version_result.rc == 0
          - "'vector' in vector_version_result.stdout"

    - name: Check Vector config file exists
      ansible.builtin.stat:
        path: /etc/vector/vector.yaml
      register: vector_config_file

    - name: Assert Vector config exists
      ansible.builtin.assert:
        that:
          - vector_config_file.stat.exists
          - vector_config_file.stat.isreg

    - name: Validate Vector config
      ansible.builtin.command: vector validate /etc/vector/vector.yaml
      register: vector_validate_result
      changed_when: false

    - name: Assert Vector config is valid
      ansible.builtin.assert:
        that:
          - vector_validate_result.rc == 0
```

## Запуск Molecule Docker scenario

Проверка сценария выполняется командой:

```bash
molecule test
```

Результат выполнения:

```text
INFO     default ➜ converge: Executed: Successful
INFO     default ➜ idempotence: Executed: Successful
INFO     default ➜ verify: Executed: Successful
INFO     default ➜ destroy: Executed: Successful
```

Во время тестирования успешно проверены оба инстанса:

```text
vector-ubuntu
vector-oraclelinux
```

После успешного выполнения Molecule Docker-сценария создан тег:

```bash
git tag 1.1.0
git push origin 1.1.0
```

## Molecule Podman scenario

Дополнительно создан облегчённый Molecule-сценарий с Podman-драйвером.

Сценарий расположен в каталоге:

```text
molecule/podman/
```

Файл `molecule/podman/molecule.yml`:

```yaml
---
dependency:
  name: galaxy

driver:
  name: podman

platforms:
  - name: vector-podman-ubuntu
    image: ubuntu:latest
    pre_build_image: true
    command: /bin/bash
    tty: true

provisioner:
  name: ansible
  playbooks:
    prepare: prepare.yml
    converge: converge.yml
  inventory:
    host_vars:
      vector-podman-ubuntu:
        ansible_python_interpreter: /usr/bin/python3

verifier:
  name: ansible
```

Файл `molecule/podman/converge.yml`:

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Include vector role
      ansible.builtin.include_role:
        name: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') }}"
```

Файл `molecule/podman/verify.yml` содержит проверки:

- `vector --version`;
- наличие `/etc/vector/vector.yaml`;
- `vector validate /etc/vector/vector.yaml`;
- assert-проверки.

## Запуск Molecule Podman scenario

Проверка Podman-сценария выполняется командой:

```bash
molecule test -s podman
```

Результат выполнения:

```text
INFO     podman ➜ converge: Executed: Successful
INFO     podman ➜ idempotence: Executed: Successful
INFO     podman ➜ verify: Executed: Successful
INFO     podman ➜ destroy: Executed: Successful
```

## Tox

В корень роли добавлен файл `tox.ini`.

Файл `tox.ini`:

```ini
[tox]
minversion = 3.20
envlist = py39
skipsdist = true

[testenv]
passenv =
    HOME
    USER
    PATH
    TERM
    container
deps =
    ansible-core==2.14.18
    ansible-compat==2.2.7
    molecule==4.0.4
    molecule-podman==2.0.3
    yamllint==1.37.1
commands =
    ansible-galaxy collection install containers.podman
    molecule test -s podman
```

Tox запускает облегчённый Molecule-сценарий с Podman-драйвером:

```bash
molecule test -s podman
```

## Запуск Tox в контейнере

Для запуска Tox использовался образ `aragast/netology:latest`.

Команда запуска контейнера:

```bash
docker run --privileged=True \
  -v /root/vector-role:/opt/vector-role \
  -w /opt/vector-role \
  -it aragast/netology:latest \
  /bin/bash
```

Внутри контейнера выполнена команда:

```bash
tox
```

Результат выполнения:

```text
py39: commands succeeded
congratulations :)
```

Это подтверждает, что Tox успешно создал окружение `py39`, установил необходимые зависимости и выполнил команду:

```bash
molecule test -s podman
```

После успешного выполнения Tox создан тег:

```bash
git tag 1.2.0
git push origin 1.2.0
```

## Итоговая проверка тегов

Команда:

```bash
git tag
```

Результат:

```text
1.0.0
1.1.0
1.2.0
```

Проверка истории Git:

```bash
git log --oneline --decorate -5
```

Результат:

```text
a646a49 (HEAD -> 08-ansible-05-testing, tag: 1.2.0, origin/08-ansible-05-testing) Add tox and podman molecule scenario
9db975c (tag: 1.1.0) Add molecule docker scenario
8b9739d (tag: 1.0.0, origin/main, origin/HEAD, main) Initial vector role
```

## Результат выполнения

В результате выполнения домашнего задания:

- изучен пример Molecule-сценария на роли ClickHouse;
- для роли `vector-role` создан Molecule Docker-сценарий;
- в Docker-сценарий добавлены инстансы `ubuntu:latest` и `oraclelinux:8`;
- роль Vector успешно протестирована на обоих инстансах;
- в `verify.yml` добавлены assert-проверки;
- выполнена проверка идемпотентности роли;
- создан тег `1.1.0` для Molecule-этапа;
- создан облегчённый Molecule-сценарий с Podman-драйвером;
- добавлен файл `tox.ini`;
- через Tox успешно запущен Molecule Podman-сценарий;
- создан тег `1.2.0` для Tox-этапа;
- все изменения загружены в ветку `08-ansible-05-testing`.

## Ссылки для проверки

- Репозиторий с ролью: <https://github.com/victoryurochkin/vector-role>
- Ветка с решением: <https://github.com/victoryurochkin/vector-role/tree/08-ansible-05-testing>
- Тег Molecule: <https://github.com/victoryurochkin/vector-role/releases/tag/1.1.0>
- Тег Tox: <https://github.com/victoryurochkin/vector-role/releases/tag/1.2.0>
- Репозиторий с отчётом: <https://github.com/victoryurochkin/08-ansible-05-testing>
