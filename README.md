# Домашнее задание "Тестирование roles"

## Цели домашней работы.

Настроить автоматическое тестирование Ansible-роли Vector с помощью Molecule и Tox.

В качестве основы использована роль "vector" из предыдущего домашнего задания.

## Подготовка.

Для выполнения домашней работы нам необходимо:

- Molecule — для создания тестовых окружений и проверки Ansible-роли;
- Docker — для запуска тестовых Linux-контейнеров;
- Tox — для запуска тестирования в нескольких Python/Ansible-окружениях.

Для выполнения задания используется WSL2 с Ubuntu.
Docker запущен через Docker Desktop с интеграцией WSL2.
Для изоляции Python-зависимостей создано отдельное виртуальное окружение.
В нём устанавливаются Ansible, Molecule, Docker-драйвер Molecule и Tox.

# Molecule

### 1. Запускаем molecule test

Как в задании написано, сначала запускаем готовый сценарий Molecule для роли ClickHouse, чтобы посмотреть, как устроено тестирование роли и из каких файлов состоит сценарий.

На данном этапе команда может выполниться с ошибками — это допустимо, так как наша задача посмотреть на работу готового сценария.

Проверяем какие сценарии уже есть:

```
le -maxdepth 2 -type f -print
```

Видим, что среди них есть нужный нам сценарий molecule/ubuntu_xenial/molecule.yml

Пробуем запустить его и получаем ошибку:

```
ERROR Failed to validate molecule/ubuntu_xenial/molecule.yml
```

Чтобы понять, что за ошибка, запускаем тест с отладкой:

```
molecule --debug test -s ubuntu_xenial
```

Что видем в выводе:

```
provisioner.playbooks found in molecule.yml,
this can be defined in ansible.playbooks
```

Погуглив можно выяснить, что готовый сценарий ClickHouse написан под более старую версию Molecule, а у меня установлена Molecule 26.8.0
Из-за изменений в формате конфигурации старый molecule.yml не проходит валидацию.
Сам сценарий ClickHouse исправлять не стал, так как дальше по заданию необходимо создать уже свой сценарий тестирования для роли Vector.

### 2 Создаём новый сценарий тестирования для Vector

Переходим в директорию с ролью Vector:

```
cd "/mnt/c/Users/Наталья/Documents/Project_home/ansible_play_5"
```

Переходим в директорию с ролью Vector и нам по заданию предлагается выполнить команду: 
molecule init scenario --driver-name docker
Но в установленной версии Molecule 26.8.0 параметр --driver-name уже не поддерживается:

```
Error: No such option '--driver-name'.
```

Поэтому созадем сценарий стандартной командой molecule init scenario.

```
INFO default ➜ init: Initialized scenario in .../molecule/default successfully.
```

Как мы видим, всё успешно создалось!
Ищем созданные файлы:

```
find molecule -maxdepth 2 -type f -print
molecule/centos_7/converge.yml
molecule/centos_7/molecule.yml
molecule/centos_7/verify.yml
molecule/centos_8/converge.yml
molecule/centos_8/molecule.yml
molecule/centos_8/verify.yml
molecule/debian_bullseye/molecule.yml
molecule/debian_buster/molecule.yml
molecule/debian_jessie/molecule.yml
molecule/debian_stretch/molecule.yml
molecule/resources/Dockerfile.j2
molecule/resources/Dockerfile_jessie.j2
molecule/ubuntu_bionic/molecule.yml
molecule/ubuntu_focal/converge.yml
molecule/ubuntu_focal/molecule.yml
molecule/ubuntu_focal/verify.yml
molecule/ubuntu_xenial/molecule.yml
```

Таким образом, стандартный сценарий default для тестирования роли Vector создан.

### 3. Добавляем разные дистрибутивы и тестируем роль
Для тестирования роли добавляем два Docker-окружения:

- "ubuntu:latest";
- "oraclelinux:8".

Добавляем Docker драйвера в окружении и две платформы.

При первом запуске:
```
molecule create
```

Molecule успешно начал создавать оба окружения, но контейнер Ubuntu завершился с ошибкой:

```
"/sbin/init": stat /sbin/init: no such file or directory
```

Причина в том, что стандартный образ `ubuntu:latest` не содержит настроенный systemd.
Для Ubuntu был создан отдельный Dockerfile:

```
molecule/default/Dockerfile-ubuntu.j2
```

В него были добавлены "systemd", Python и необходимые утилиты.
Также в "molecule.yml" для Ubuntu был указан созданный Dockerfile:

```yaml
dockerfile: Dockerfile-ubuntu.j2
FROM ubuntu:latest

ENV container=docker
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        systemd \
        systemd-sysv \
        python3 \
        python3-apt \
        sudo \
        tar \
        gzip \
        ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

STOPSIGNAL SIGRTMIN+3

CMD ["/sbin/init"]
```

После удаления старого локального образа и повторной сборки запускаем:

```
molecule create
```

На этот раз создание окружений завершается успешно:

```
INFO default ➜ create: Executed: Successful
```

Проверяем контейнеры:

```
docker ps
molecule_local/oraclelinux:8   "/sbin/init"   Up   oraclelinux8
molecule_local/ubuntu:latest   "/sbin/init"   Up   ubuntu
```

Далее настраиваем converge.yml для применения роли Vector к обоим контейнерам и запускаем:

```
molecule converge
```

Роль успешно отрабатывает на Ubuntu и Oracle Linux:

```
oraclelinux8 : ok=13 changed=11 unreachable=0 failed=0
ubuntu       : ok=13 changed=11 unreachable=0 failed=0
```

Molecule также подтверждает успешное выполнение этапа:

```
INFO default ➜ converge: Executed: Successful
```

Таким образом, роль Vector успешно устанавливается и запускается на `ubuntu:latest` и `oraclelinux:8`.

### 4. Добавляем проверки в verify.yml

После успешного запуска роли на Ubuntu и Oracle Linux добавляем проверки, чтобы убедиться, что Vector действительно установился и работает корректно.

Для этого редактируем файл:

```
molecule/default/verify.yml
```

Добавляем несколько проверок:

```
---
- name: Verify
  hosts: all
  become: true
  gather_facts: false

  tasks:
    - name: Check Vector binary
      ansible.builtin.stat:
        path: /usr/bin/vector
      register: vector_binary

    - name: Assert Vector binary exists
      ansible.builtin.assert:
        that:
          - vector_binary.stat.exists
        fail_msg: "Vector binary was not found"
        success_msg: "Vector binary exists"

    - name: Validate Vector config
      ansible.builtin.command:
        cmd: /usr/bin/vector validate /etc/vector/vector.yaml
      register: vector_config_validation
      changed_when: false

    - name: Assert Vector config is valid
      ansible.builtin.assert:
        that:
          - vector_config_validation.rc == 0
        fail_msg: "Vector configuration is invalid"
        success_msg: "Vector configuration is valid"

    - name: Check Vector service
      ansible.builtin.command:
        cmd: systemctl is-active vector
      register: vector_service
      changed_when: false

    - name: Assert Vector service is running
      ansible.builtin.assert:
        that:
          - vector_service.stdout == "active"
        fail_msg: "Vector service is not running"
        success_msg: "Vector service is active"
```

Здесь проверяем:
существует ли бинарник /usr/bin/vector;
проходит ли конфигурация /etc/vector/vector.yaml проверку Vector;
находится ли сервис vector в состоянии active.

После этого запускаем:

```
molecule verify
```

Проверки успешно проходят и на Ubuntu, и на Oracle Linux:

```
oraclelinux8 : ok=6 changed=0 unreachable=0 failed=0
ubuntu       : ok=6 changed=0 unreachable=0 failed=0
```

Также Molecule сообщает:

```
INFO default ➜ verify: Executed: Successful
INFO Molecule executed 1 scenario (1 successful)
```

Таким образом, проверка показала, что на обоих дистрибутивах бинарник Vector существует, конфигурация валидна, а сервис успешно запущен.

### 5.  Повторно запускаем тестирования ролей.

После настройки сценария и добавления проверок запускаем полный цикл тестирования:

```
molecule test
```

При первом запуске появилась ошибка Docker Desktop:

```
Exec format error: '/usr/bin/docker-credential-desktop.exe'
```

Проблема была связана не с ролью Vector, а с Docker credential helper внутри WSL.
Чтобы Molecule заработал, создал отдельный Docker-конфиг:

```
mkdir -p ~/.docker-molecule
cat > ~/.docker-molecule/config.json <<'EOF'
{
  "auths": {}
}
EOF

export DOCKER_CONFIG="$HOME/.docker-molecule"
```
После этого повторно запускаем:

```
molecule test
```


На этот раз полный сценарий проходит успешно.
Вывод из консоли:

```
INFO     default ➜ destroy: Executed: Successful
INFO     default ➜ scenario: Pruning extra files from scenario ephemeral directory
WARNING  Molecule executed 1 scenario (1 missing files)

DETAILS                                                                        
default ➜ dependency: Executed: 2 missing (Remove from test_sequence to suppress)
default ➜ cleanup: Executed: Missing playbook (Remove from test_sequence to suppress)
default ➜ destroy: Executed: Successful
default ➜ syntax: Executed: Successful
default ➜ create: Executed: Successful
default ➜ prepare: Executed: Missing playbook (Remove from test_sequence to suppress)
default ➜ converge: Executed: Successful
default ➜ idempotence: Executed: Successful
default ➜ side_effect: Executed: Missing playbook (Remove from test_sequence to suppress)
default ➜ verify: Executed: Successful
default ➜ cleanup: Executed: Missing playbook (Remove from test_sequence to suppress)
default ➜ destroy: Executed: Successful

SCENARIO RECAP                                                                 
default                   : actions=12  successful=7  disabled=0  skipped=0  missing=6  failed=0
```

Итог:
```
SCENARIO RECAP
default : actions=12 successful=7 disabled=0 skipped=0 missing=6 failed=0
```
Таким образом, полный цикл Molecule успешно прошёл для ubuntu и oracle роль Vector корректно устанавливается, является идемпотентной, конфигурация валидна, а сервис успешно запускается.














# Vector Role

Ansible role для установки и настройки Vector.

Роль выполняет:

- скачивание указанной версии Vector;
- распаковку Vector в `/opt/vector`;
- создание системного пользователя `vector`;
- создание каталогов конфигурации и данных;
- создание ссылки на бинарный файл Vector;
- установку systemd unit;
- разворачивание конфигурационного файла;
- запуск и включение сервиса Vector.

## Role Variables

Версия Vector задаётся в `defaults/main.yml`:

```yaml
vector_version: "0.58.0"

Версию Vector можно переопределить при использовании роли.

Внутренние параметры роли находятся в `vars/main.yml`:

```yaml
vector_user: vector
vector_install_dir: /opt/vector
vector_config_dir: /etc/vector
vector_data_dir: /var/lib/vector