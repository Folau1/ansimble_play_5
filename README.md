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

### 6. Добавляем тэг.

Сначала, всё закомитим и запушим.

```
git status
fatal: not a git repository (or any of the parent directories): .git
```

Да пока ничего нет, всё добавляем и делаем первый коммит:
Делаем git add и git commit.

```
git log --oneline -3
5fc1d8f (HEAD -> main) Первый коммит
```


Поставить семантический тег:
```
git tag v1.0.0
```

## TOX

### 1. Добавляем файлы для Tox.

По заданию нужно добавить в корень роли Vector файлы tox.ini и requirements.txt из example.
Файл tox.ini:

```
[tox]
minversion = 1.8
basepython = python3.6
envlist = py{37,39}-ansible{210,30}
skipsdist = true

[testenv]
passenv = *
deps =
    -r tox-requirements.txt
    ansible210: ansible<3.0
    ansible30: ansible<3.1
commands =
    {posargs:molecule test -s compatibility --destroy always}
```


Файл `tox-requirements.txt`:

```
selinux
lxml
molecule
molecule_podman
jmespath
```

Проверяем содержимое файлов:

```
cat tox.ini
cat tox-requirements.txt
```

Из tox.ini видно, что Tox должен запускать тестирование в нескольких окружениях Python и Ansible, а для проверки использовать сценарий из molecule.

Далее скачиваем подготовленный учебный Docker-образ:

```
export DOCKER_CONFIG="$HOME/.docker-molecule"
docker pull aragast/netology:latest
```

Образ успешно скачан:

```
Digest: sha256:e44f93d3d9880123ac8170d01bd38ea1cd6c5174832b1782ce8f97f13e695ad5
Status: Downloaded newer image for aragast/netology:latest
docker.io/aragast/netology:latest
```

При повторной проверке Docker сообщает, что образ уже актуален:

```
Status: Image is up to date for aragast/netology:latest
```

Таким образом, файлы для запуска Tox добавлены, docker образ готов.

### 2. Запускаем подготовленный контейнер для Tox

Запускаем docker контейнер и монтируем в него директорию с ролью.
Команда запуска:

```
docker run --privileged=true   -v "/mnt/c/Users/Наталья/Documents/Project_home/ansible_play_5:/opt/vector-role"   -w /opt/vector-role   -it aragast/netology:latest /bin/bash
```

После запуска попадаем внутрь контейнера:

```
[root@93a91329436f vector-role]#
```

Проверяем текущую директорию:
```
pwd
```

Получаем:

```
/opt/vector-role
```

Проверяем содержимое директории:

```
ls -la
```

Внутри контейнера видим примонтированный проект:

```
.git
README.md
defaults
handlers
meta
molecule
tasks
templates
tests
tox-requirements.txt
tox.ini
vars
```

Роль Vector примонтирована внутрь контейнера.
Далее запускаем Tox:

```
tox
```

Tox начинает создавать первое тестовое окружение:

```
py37-ansible210 create: /opt/vector-role/.tox/py37-ansible210
py37-ansible210 installdeps: -rtox-requirements.txt, ansible<3.0
```

Это означает, что Tox создаёт отдельное виртуальное окружение для Python и Ansible и устанавливает в него необходимые зависимости.

После запуска будем сомтреть, появятся ошибки или нет и будем исправлять.

### 3. Первый запуск Tox

Внутри подготовленного контейнера запускаем Tox.
Tox начал создавать первое тестовое окружение:

```
py37-ansible210 create: /opt/vector-role/.tox/py37-ansible210
py37-ansible210 installdeps: -rtox-requirements.txt, ansible<3.0
```

Во время установки зависимостей процесс выполнялся слишком долго, поэтому был остановлен вручную сочетанием Ctrl+C.

После остановки Tox вывел:

```
ERROR: got KeyboardInterrupt signal
summary

ERROR:   py37-ansible210: keyboardinterrupt
ERROR:   py37-ansible30: undefined
ERROR:   py39-ansible210: undefined
ERROR:   py39-ansible30: undefined
```

На этом этапе видно, что Tox корректно прочитал tox.ini, начал создавать окружение и перешёл к установке зависимостей.

Так как запуск был остановлен вручную, ошибки keyboardinterrupt и undefined не являются результатом проверки роли — это следствие незавершённого выполнения Tox.

### 4. Облегчённый вариант Molecule

Для проверки совместимости роли создаём отдельный сценарий compatibility с драйвером molecule_podman.

Итоговый файл molecule/compatibility/molecule.yml:

```
dependency:
  name: galaxy

driver:
  name: podman

platforms:
  - name: instance
    image: quay.io/centos/centos:stream8
    pre_build_image: true
    privileged: true
    command: /usr/sbin/init
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    tmpfs:
      - /run
      - /tmp

provisioner:
  name: ansible

verifier:
  name: ansible

Для запуска роли используется облегчённый molecule/compatibility/converge.yml:

---
- name: Converge
  hosts: all

  tasks:
    - name: Apply Vector role
      ansible.builtin.include_role:
        name: folau1.vector

```

Проверяем сценарий:

```
molecule test -s compatibility
```

Сценарий успешно создал Podman-контейнер и применил роль Vector:

```
PLAY RECAP
instance : ok=12 changed=11 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Проверка идемпотентности также прошла успешно:

```
PLAY RECAP
instance : ok=10 changed=0 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0

INFO     Idempotence completed successfully.
```

Этап verify завершился успешно:

```
TASK [Example assertion]
ok: [instance] => {
    "changed": false,
    "msg": "All assertions passed"
}

INFO     Verifier completed successfully.

```
После завершения тестирования Molecule успешно удалил созданный контейнер.

Отсюда следует, что наш сценарий molecule с драйвером podman создан и проверен. Роль Vector также выполняется в тестовом окружении.

### 5. Настройка запуска облегчённого сценария через Tox

В tox.ini указываем запуск созданного облегчённого сценария compatibility:

```
commands =
    {posargs:molecule test -s compatibility --destroy always}
```

Итоговый tox.ini:

```
[tox]
minversion = 1.8
basepython = python3.6
envlist = py{37,39}-ansible{210,30}
skipsdist = true

[testenv]
passenv = *
deps =
    -r tox-requirements.txt
    ansible210: ansible<3.0
    ansible30: ansible<3.1
commands =
    {posargs:molecule test -s compatibility --destroy always}
```

Таким образом, Tox будет запускать именно сценарий "molecule/compatibility".


### 6. Запуск Tox

Запускаем тестирование командой:

```
tox
```

Перед запуском проверил наличие необходимых версий Python:

```
Python 3.7.10
Python 3.9.2
```

После запуска Tox начал создавать первое тестовое окружение:

```
py37-ansible210 create: /opt/vector-role/.tox/py37-ansible210
py37-ansible210 installdeps: -rtox-requirements.txt, ansible<3.0
py39-ansible30 create: /opt/vector-role/.tox/py39-ansible30
py39-ansible30 installdeps: -rtox-requirements.txt, ansible<3.1
```

Установка зависимостей занимает продолжительное время.

Проверка завершилась успешно: роль Vector применилась без ошибок, проверка идемпотентности завершилась с changed=0, а этап verify прошёл успешно.

### 7. Добавление нового тега
После всех операций, добавляем новый тэг.

Предыдущая версия:

```
v1.0.0
```

Так как в проект добавлен новый функционал тестирования без нарушения обратной совместимости, используем новую minor-версию:

```
v1.1.0
```

Создаём тег:
```
git tag v1.1.0
```

Отправляем тег в удалённый репозиторий:
```
git push origin v1.1.0
```
Проверяем:
```
git tag
```
В списке тегов должны быть:
```
v1.0.0
v1.1.0
```












