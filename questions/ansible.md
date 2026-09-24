## Ansible

1. Чем отличаются Ansible модули *raw*, *command* и *shell*?

<details>
  <summary>Ответ</summary>

Модуль *raw* отличается от *command* и *shell* тем, что не выполняет дополнительную обработку выполнения команды. Эти дополнительные обработки присутствуют в почти любом модуле Ansible. Модуль *raw* передает команду, как есть, в "сыром" (raw) виде без проверок.
Модули *command* и *shell* отличаются тем, что в модуле *command* команда выполняется без прохождения через командную оболочку `/bin/sh`. Поэтому переменные определенные в оболочке и перенаправления - конвееры работать не будут. Модуль *shell* выполняет команды через оболочку по умолчанию `/bin/sh`. Поэтому там будут доступны переменные оболочки и перенаправления.

</details>

2. На всех серверах должен быть набор пользователей, с доступом по ssh-ключу, стандартный модуль user не позволяет вносить ssh ключ в authorized_keys. Предложите решение.

<details>
  <summary>Ответ</summary>

1. Использовать модуль `authorized_key` для добавления ключей.
2. Использовать модуль `shell`, чтобы вручную с использованием команды `cat {{ PUBLIC_SSH_KEY }} >> /home/{{ USER }}/.ssh/authorized_keys` добавить ключ. В данном случае шаблоны Jinja2 PUBLIC_SSH_KEY и USER должны быть заданы.

**Актуально на 2026:** модуль теперь находится в коллекции `ansible.posix` и вызывается как `ansible.posix.authorized_key` (с `exclusive: true` можно удалить все неперечисленные ключи). Вариант 2 с `shell` не идемпотентен — при каждом запуске ключ будет дописываться повторно, поэтому его лучше не использовать.

</details>

3. Есть группы пользователей, которые должны заводиться не на всех серверах. Как ограничить заведение пользователей?

<details>
  <summary>Ответ</summary>

Сгруппировать сервера, на которых должны заводиться группы пользователей, в инвентори или написать в плейбуке условие, которому передаётся список серверов, на которых необходимо выполнить задачу.

</details>

4. На новом сервере не установлен Python, который требуется для работы Ansible. Как выполнить установку Python на сервере используя Ansible?

<details>
  <summary>Ответ</summary>

Использовать модуль `raw`, которому необходимо передать команду для установки python на сервере. Модуль `raw` принимает команду без дополнительной обработки Python и выполняет её на сервере.

</details>

5. Что такое роль в Ansible? Что содержит в себе Ansible роль?

<details>
  <summary>Ответ</summary>

Ansible роль представляет собой структурированный плейбук, содержащий, как минимум, набор задач (tasks) и дополнительно - обработчики событий (handlers), переменных (default и vars), файлов (files), шаблонов (templates), описание и зависимости (metadata) и тесты (tests).

</details>

6. В Ansible роли есть директории *vars* и *default*. Что они содержат и чем отличаются?

<details>
  <summary>Ответ</summary>

Ansible применяет порядок приоритета переменных. Ниже представлен список в порядке повышения приоритета.

1. command line values (for example, -u my_user, these are not variables)
2. role defaults (defined in role/defaults/main.yml)
3. inventory file or script group vars
4. inventory group_vars/all
5. playbook group_vars/all
6. inventory group_vars/*
7. playbook group_vars/*
8. inventory file or script host vars
9. inventory host_vars/*
10. playbook host_vars/*
11. host facts / cached set_facts
12. play vars
13. play vars_prompt
14. play vars_files
15. role vars (определяемые в role/vars/main.yml)
16. block vars (только для задач в `block`)
17. task vars (только для задач)
18. include_vars
19. set_facts / registered vars
20. role (и include_role) params
21. include params
22. extra vars (например, -e "user=my_user")(всегда приоритетнее)

Соответственно переменные в *vars* будут приорететнее, чем в *defaults*.

</details>

7. В Ansible роли есть директории *file* и *templates*. Что они содержат и чем отличаются?

<details>
  <summary>Ответ</summary>

*files* - содержит файлы, которые будут скопированы на настраиваемые хосты; так же — может содержать скрипты, которые позже будут запускаться на хостах.

*templates* - содержит шаблоны файлов с переменными.

</details>

8. По-умолчанию, в Ansible все задачи из списка выполняются параллельно на всех хостах, которые указаны в `hosts`. Как сделать так, чтобы задачи выполнялись последовательно по хостам?

<details>
  <summary>Ответ</summary>

Необходимо установить параметр `serial: 1`, чтобы определить количество хостов, на которых будут выполняться паралелльно задачи. Значение 1 будет значить, что все задачи будут проходить параллельно по 1 хосту за раз.

Ссылка на документацию: https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html#setting-the-batch-size-with-serial

</details>

9. Как работает Ansible: push или pull? Нужен ли агент на управляемых хостах?

<details>
  <summary>Ответ</summary>

Ansible по умолчанию работает по **push-модели** и без агента: control node подключается к хостам по SSH (Windows — WinRM/SSH), копирует на них Python-код модуля во временный каталог, выполняет его и забирает результат в JSON. На хостах нужны только SSH-доступ и Python (кроме модулей `raw`/`script` и сетевых устройств).

Параллельность задаётся `forks` (по умолчанию 5) в `ansible.cfg` или флагом `-f`. Ускорить работу помогают `pipelining = True` и SSH ControlPersist.

Есть и pull-вариант — `ansible-pull`: хост сам по cron клонирует репозиторий с плейбуками и применяет их к себе (`ansible-pull -U https://git.example.com/infra.git local.yml`). Подходит для большого числа машин без постоянного доступа с control node.

</details>

10. Что такое идемпотентность в Ansible и как сохранить её при использовании `command`/`shell`?

<details>
  <summary>Ответ</summary>

Идемпотентность — повторный запуск плейбука на уже настроенной системе ничего не меняет и показывает `changed=0`. Большинство модулей (`package`, `copy`, `template`, `user`) проверяют текущее состояние и действуют только при расхождении.

`command` и `shell` всегда выполняют команду и всегда возвращают `changed`, поэтому их делают идемпотентными вручную:

```yaml
- name: Инициализировать кластер один раз
  ansible.builtin.command: kubeadm init --config /etc/kubeadm.yaml
  args:
    creates: /etc/kubernetes/admin.conf

- name: Проверить версию приложения
  ansible.builtin.command: /opt/app/bin/app --version
  register: app_version
  changed_when: false
```

Также используют `removes`, `changed_when`/`failed_when` по выводу команды, а по возможности — специализированный модуль вместо команды.

</details>

11. Чем статический inventory отличается от динамического? Где хранить переменные групп и хостов?

<details>
  <summary>Ответ</summary>

Статический inventory — файл INI/YAML со списком хостов и групп. Динамический строится на лету из внешнего источника (облако, CMDB, NetBox) с помощью inventory-плагинов. Пример для AWS (имя файла обязано заканчиваться на `aws_ec2.yml`):

```yaml
# inventory/prod.aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - eu-central-1
keyed_groups:
  - key: tags.Role
    prefix: role
```

Переменные кладут рядом с inventory (или с плейбуком) в каталоги `group_vars/<группа>.yml`, `group_vars/all.yml` и `host_vars/<хост>.yml` — так они не смешиваются с описанием хостов. Проверить итоговое дерево групп и переменные: `ansible-inventory -i inventory/ --graph` и `ansible-inventory -i inventory/ --host web1`.

</details>

12. Что такое handlers и как работает `notify`?

<details>
  <summary>Ответ</summary>

Handler — задача, которая выполняется только если её «уведомила» задача со статусом `changed`. Типичный пример — перезапуск сервиса после изменения конфига:

```yaml
tasks:
  - name: Конфиг nginx
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reload nginx

handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

Особенности: handler запускается один раз, сколько бы задач его ни уведомили, и по умолчанию в конце секции play (после `tasks`), а не сразу. Выполнить накопленные handlers раньше можно через `- meta: flush_handlers`. Несколько handlers можно подписать на одно событие через `listen`. Если play упал, handlers не выполнятся, если не задан `force_handlers: true` (или `--force-handlers`).

</details>

13. Для чего нужны check mode и diff mode?

<details>
  <summary>Ответ</summary>

`ansible-playbook site.yml --check --diff` — «сухой прогон»: модули, поддерживающие check mode, сообщают, что **изменили бы**, но ничего не меняют, а `--diff` показывает построчную разницу для файлов и шаблонов. Удобно для ревью перед применением на prod и в CI.

Нюансы:
- `command`/`shell` в check mode пропускаются (если не заданы `creates`/`removes`), поэтому задачи, зависящие от их результата через `register`, могут вести себя иначе;
- `check_mode: false` у задачи заставляет её выполняться даже в check mode (например, для чтения данных), `check_mode: true` — всегда работать «насухо»;
- в шаблонах и условиях доступна переменная `ansible_check_mode`;
- `diff: false` у задачи отключает вывод diff (для файлов с секретами).

</details>

14. Как работают tags?

<details>
  <summary>Ответ</summary>

Теги позволяют выполнять только часть плейбука:

```yaml
- name: Установить пакеты
  ansible.builtin.package:
    name: nginx
  tags: [nginx, packages]
```

```
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --skip-tags packages
ansible-playbook site.yml --list-tags
```

Теги можно вешать на задачи, блоки, роли и play целиком — они наследуются вложенными задачами (у `import_*`; у динамических `include_*` тег относится только к самой include-задаче, для передачи используют `apply: { tags: ... }`). Специальные теги: `always` — задача выполняется всегда, кроме явного `--skip-tags always`; `never` — только если её тег указан явно.

</details>

15. Как с помощью `serial`, `delegate_to` и `run_once` сделать rolling update за балансировщиком?

<details>
  <summary>Ответ</summary>

- `serial` — обновлять хосты партиями (число, процент или список `[1, "25%", "100%"]`);
- `max_fail_percentage` / `any_errors_fatal` — остановить выкатку, если партия упала;
- `delegate_to` — выполнить задачу на другом хосте, но в контексте текущего (например, на балансировщике);
- `run_once: true` — выполнить задачу один раз на всю партию (миграция БД, уведомление).

```yaml
- hosts: web
  serial: "25%"
  max_fail_percentage: 0
  tasks:
    - name: Вывести хост из балансировщика
      community.general.haproxy:
        state: disabled
        backend: app
        host: "{{ inventory_hostname }}"
      delegate_to: "{{ item }}"
      loop: "{{ groups['lb'] }}"

    - name: Обновить приложение
      ansible.builtin.package:
        name: myapp
        state: latest

    - name: Вернуть хост в балансировщик
      community.general.haproxy:
        state: enabled
        backend: app
        host: "{{ inventory_hostname }}"
      delegate_to: "{{ item }}"
      loop: "{{ groups['lb'] }}"
```

</details>

16. Что такое Ansible Vault и как с ним работать?

<details>
  <summary>Ответ</summary>

Ansible Vault шифрует (AES256) файлы переменных или отдельные значения, чтобы секреты можно было хранить в git:

```
ansible-vault create group_vars/prod/vault.yml
ansible-vault edit group_vars/prod/vault.yml
ansible-vault encrypt_string 'S3cr3t' --name 'db_password'
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-id prod@~/.vault_pass_prod
```

Практики:
- разделять `vars.yml` (открытый, `db_password: "{{ vault_db_password }}"`) и `vault.yml` (зашифрованный) — так grep по именам переменных продолжает работать;
- разные пароли для разных окружений через `--vault-id`;
- в CI пароль Vault передавать через секрет или скрипт, получающий его из внешнего хранилища;
- на задачах, которые работают с секретами, ставить `no_log: true`, чтобы значения не попали в вывод и логи.

Альтернатива — не хранить секреты в репозитории вовсе и получать их через lookup из HashiCorp Vault (`community.hashi_vault`) или облачного Secrets Manager.

</details>

17. Как обрабатывать ошибки: `block`, `rescue`, `always`?

<details>
  <summary>Ответ</summary>

`block` группирует задачи (общие `when`, `become`, `tags`), а `rescue` и `always` работают как try/catch/finally:

```yaml
- block:
    - name: Обновить приложение
      ansible.builtin.unarchive:
        src: "app-{{ version }}.tar.gz"
        dest: /opt/app
    - name: Проверить healthcheck
      ansible.builtin.uri:
        url: http://localhost:8080/health
  rescue:
    - name: Откатиться на предыдущую версию
      ansible.builtin.command: /opt/app/bin/rollback
  always:
    - name: Отправить уведомление
      ansible.builtin.debug:
        msg: "Деплой на {{ inventory_hostname }} завершён"
```

`rescue` выполняется, если любая задача в `block` упала; `always` — в любом случае. В `rescue` доступны `ansible_failed_task` и `ansible_failed_result`. Для отдельных задач также есть `ignore_errors: true` и `failed_when`.

</details>

18. Как выполнить долгую задачу, не упираясь в таймаут SSH? Что делают `async` и `poll`?

<details>
  <summary>Ответ</summary>

`async` задаёт максимальное время выполнения задачи в секундах, а `poll` — как часто Ansible проверяет её статус. Задача запускается в фоне на хосте, и SSH-сессия не держится всё это время.

```yaml
- name: Запустить долгий бэкап
  ansible.builtin.command: /usr/local/bin/backup.sh
  async: 3600
  poll: 0
  register: backup_job

- name: Дождаться завершения
  ansible.builtin.async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 120
  delay: 30
```

`poll: 0` — «запустил и пошёл дальше» (fire-and-forget), статус проверяется позже через `async_status`. Так можно запустить долгие операции параллельно на разных хостах или не упираться в таймаут соединения.

</details>

19. Что такое facts? Как ускорить их сбор и кэшировать?

<details>
  <summary>Ответ</summary>

Facts — сведения о хосте (ОС, IP, диски, память), которые собирает модуль `setup` в начале play; доступны как `ansible_facts['distribution']`. Посмотреть: `ansible web1 -m ansible.builtin.setup`. Свои факты можно положить на хост в `/etc/ansible/facts.d/*.fact` (local facts).

Ускорение:
- `gather_facts: false`, если факты не нужны;
- `gather_subset: [network, hardware]` — собирать только нужное;
- кэширование между запусками в `ansible.cfg`:

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /var/cache/ansible/facts
fact_caching_timeout = 86400
```

С `gathering = smart` факты собираются только для хостов, которых нет в кэше. Кэш также позволяет обращаться к фактам хостов, которых нет в текущем play, через `hostvars`.

</details>

20. Чем роль отличается от коллекции? Как управлять зависимостями через Ansible Galaxy?

<details>
  <summary>Ответ</summary>

**Роль** — переиспользуемый набор tasks/handlers/templates/defaults для одной задачи (например, настроить nginx). **Коллекция** — формат дистрибуции, в который упаковываются модули, плагины (inventory, filter, lookup), роли и плейбуки под общим пространством имён. С Ansible 2.10 большинство модулей вынесено из ядра в коллекции, поэтому модули пишут по FQCN: `ansible.builtin.copy`, `community.general.haproxy`.

Зависимости фиксируют в `requirements.yml`:

```yaml
collections:
  - name: community.general
    version: ">=9.0.0"
  - name: amazon.aws
roles:
  - name: nginx
    src: https://git.example.com/infra/ansible-role-nginx.git
    scm: git
    version: v1.2.0
```

```
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

Источники — Ansible Galaxy, Automation Hub или приватный Galaxy NG / git.

</details>

21. Как тестировать роли? Что такое Molecule и ansible-lint?

<details>
  <summary>Ответ</summary>

**ansible-lint** — статический анализ плейбуков и ролей: FQCN, `name` у задач, `no-changed-when` для `command`, небезопасные права файлов, устаревший синтаксис. Уровень строгости задаётся профилями (`min`, `basic`, `moderate`, `safety`, `shared`, `production`). Запускается в pre-commit и CI.

**Molecule** — фреймворк для тестирования ролей в одноразовом окружении (контейнеры Docker/Podman, ВМ или внешний «delegated» драйвер). `molecule test` проходит (упрощённо) последовательность: `create` → `prepare` → `converge` (применить роль) → **`idempotence`** (повторный прогон должен дать `changed=0`) → `verify` (проверки через Ansible-задачи или testinfra) → `destroy`.

```
molecule init scenario default
molecule converge   # применить роль и оставить окружение для отладки
molecule test       # полный цикл
```

</details>

22. Что такое execution environments и ansible-navigator?

<details>
  <summary>Ответ</summary>

**Execution environment (EE)** — контейнерный образ, в котором собраны ansible-core, нужные коллекции, Python-библиотеки и системные пакеты. Он решает проблему «у меня работает»: на ноутбуке, в CI и в AWX/Automation Controller используется один и тот же набор зависимостей нужных версий.

- **ansible-builder** собирает образ по файлу `execution-environment.yml` (базовый образ, `requirements.yml`, `requirements.txt`, `bindep.txt`): `ansible-builder build -t registry.example.com/ee-infra:1.0`.
- **ansible-navigator** запускает плейбуки внутри EE (и имеет TUI для разбора результатов):

```
ansible-navigator run site.yml -i inventory/ --eei registry.example.com/ee-infra:1.0 --mode stdout
```

</details>

23. Чем Ansible отличается от Terraform и как их использовать вместе?

<details>
  <summary>Ответ</summary>

- **Terraform** — инструмент провижининга: создаёт и удаляет облачные ресурсы (сети, ВМ, БД, DNS) декларативно, хранит state и по нему строит план изменений, знает зависимости ресурсов и умеет удалять то, что исчезло из кода.
- **Ansible** — управление конфигурацией и оркестрация: настраивает ОС и софт внутри уже существующих машин, выполняет задачи по порядку, state не хранит — каждый раз сравнивает с фактическим состоянием хоста. Удалённое из плейбука само не откатится.

Типичная связка: Terraform создаёт инфраструктуру и отдаёт IP/теги, Ansible через динамический inventory (по тем же тегам) настраивает хосты. При immutable-подходе Ansible используют внутри Packer для сборки образов, а Terraform только разворачивает готовые образы.

</details>

24. Какие фильтры Jinja2 чаще всего используются в Ansible?

<details>
  <summary>Ответ</summary>

```yaml
# значение по умолчанию и пропуск параметра
port: "{{ app_port | default(8080) }}"
mode: "{{ file_mode | default(omit) }}"
# обязательная переменная — понятная ошибка, если не задана
token: "{{ api_token | mandatory }}"
# слияние словарей (recursive=true — глубокое)
settings: "{{ defaults_cfg | combine(env_cfg, recursive=true) }}"
# выборка из списка словарей
admins: "{{ users | selectattr('admin', 'equalto', true) | map(attribute='name') | list }}"
# прочее
line: "{{ hostname | regex_replace('[.]example[.]com$', '') }}"
cfg: "{{ config_dict | to_nice_yaml }}"
flag: "{{ enabled | bool | ternary('on', 'off') }}"
```

Также полезны `unique`, `flatten`, `dict2items`/`items2dict`, `b64decode`, `password_hash`, `ipaddr`-фильтры из `ansible.utils`. Тип переменной проверяют через `{{ var | type_debug }}`. В ansible-core 2.19 шаблонизатор стал строже: условие в `when` должно возвращать булево значение, поэтому вместо `when: my_list` лучше писать `when: my_list | length > 0`.

</details>
