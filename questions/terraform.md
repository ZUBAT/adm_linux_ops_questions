## Terraform

1. Что содержит код Terraform?

<details>
  <summary>Ответ</summary>

Ресурсы облачного провайдера, а также провижининг для создаваемых ресурсов.

</details>

2. Как хранить состояние инфраструктуры в Terraform?

<details>
  <summary>Ответ</summary>

Например, можно хранить tfstate в git-репозитории команды. Другой вариант - хранить в специализированном Terraform Backend.

**Актуально на 2026:** хранить tfstate в git не рекомендуется: в state секреты лежат открытым текстом, а git не даёт блокировки от одновременного `apply`. Стандарт — remote backend с шифрованием и locking (см. вопросы 13–14).

</details>

3. Terraform Backend. Какой лучше?

<details>
  <summary>Ответ</summary>

Зависит от требованиям к хранению состояния.

- AWS S3 — Standard (с блокировкой через DynamoDB). Сохраняет состояние в виде заданного ключа в заданном сегменте на Amazon S3. Этот бэкэнд также поддерживает блокировку состояния и проверку согласованности через DynamoDB.

- terraform enterprise — Standard (без блокировки).

- etcd — Standard (без блокировки). Сохраняет состояние в etcd 2.x по заданному пути.

- etcdv3 — Standard (с блокировкой). Сохраняет состояние в хранилище etcd в виде K/V с заданным префиксом.

- gcs — Standard (с блокировкой). Сохраняет состояние как объект в настраиваемом префиксе в заданном сегменте в Google Cloud Storage (GCS). Этот бэкэнд также поддерживает блокировку состояния.

- Gitlab Terraform state (с блокировкой). Хранит состояние в Gitlab Terraform state хранилище, используя HTTP протокол и права Gitlab для доступа.

Существуют также и другие Backend для Terraform.

**Актуально на 2026:** backend-ы `etcd` и `etcdv3` удалены из Terraform в версии 1.3. Для S3 блокировка через DynamoDB (`dynamodb_table`) объявлена устаревшей: с Terraform 1.10/1.11 используется нативная блокировка S3 — `use_lockfile = true` (см. вопрос 14).

</details>

4. Как добавить имеющиеся ресурсы в tfstate?

<details>
  <summary>Ответ</summary>

```
terraform import [options] ADDRESS ID
```
1. Например, создаем директорию и инициализируем будущую инфраструктуру:
```
mkdir terraform-test
cd terraform-test
terraform init
vi main.tf
```
2. Добавляем в файл main.tf следующий код:
```
provider "aws" {
  region = "us-west-1"
  profile = "tyx-local"
}
resource "aws_s3_bucket" "sample_bucket" {
  bucket = "tyx-local-bucket"
  acl = "public"
}
```
3. Выполняем импорт ресурса:
```
terraform import aws_s3_bucket.sample_bucket tyx-local-bucket
```

**Актуально на 2026:** с Terraform 1.5 импорт удобнее описывать блоком `import` с генерацией кода через `terraform plan -generate-config-out` (см. вопрос 16). В примере выше `acl = "public"` — недопустимое значение, а в AWS provider v4+ ACL бакета задаётся отдельным ресурсом `aws_s3_bucket_acl`.

</details>

5. Зачем нужен `terraform taint`?

<details>
  <summary>Ответ</summary>

Команда `terraform taint` пометит ресурс инфраструктуры, который будет удален и заново создан при следующем применении команды `terraform apply`. 

**Актуально на 2026:** `terraform taint` устарел (с Terraform 0.15.2). Вместо него используют `terraform apply -replace="aws_instance.web"` — пересоздание сразу видно в плане и не требует отдельного изменения state.

</details>

6. Как проводить тестирование terraform?

<details>
  <summary>Ответ</summary>

`terraforn plan` выполнит проверку действующего кода. Работу с облачными ресурсами выполнит 

**Актуально на 2026:** базовые проверки — `terraform fmt -check` и `terraform validate`, линтер tflint, сканеры Checkov/Trivy; `terraform plan` показывает изменения, но не проверяет логику. С Terraform 1.6 есть встроенный `terraform test` (см. вопрос 28), для интеграционных тестов также применяют Terratest.

</details>

7. Что такое модуль в terraform? Для чего он нужен?

<details>
  <summary>Ответ</summary>

Модуль в Terraform - пакет конфигурации Terraform, который можно использовать при повторной конфигурации компонентов инфраструктуры, а также базовой организации кода Terraform в директориях. При подключения модуля, ему даётся имя.

</details>

8. Как хранить переменные в terraform?

<details>
  <summary>Ответ</summary>

*main.tf* - основной конфигурационный файл, описывающий какие инстансы необходимо создать.
*variables.tf* - конфигурация с описанием переменных и значениями по-умолчанию. Если значения по-умолчанию не задано, то они являются обязательными.
*terraform.tfvars* - конфигурация со значениями переменных. Часто является секретным файлом, поэтому нужно с осторожностью пушить в публичные репозитарии.
*outputs.tf* - описание выходных переменных. Необязательный файл, но очень удобно выделять нужные параметры из созданного инстанса, например IP созданного в облаке инстанса.

</details>

9. Как конвертировать Kubernetes yaml-манифест в HCL средствами Linux и terraform?

<details>
  <summary>Ответ</summary>

Например:
```
echo 'yamldecode(file("filename.yaml"))' | terraform console
```

</details>

10. Что такое Workspaces в Terraform?

<details>
  <summary>Ответ</summary>

[Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces#using-workspaces) в Terraform - это возможность управления state файлами. Workspace содержит все что необходимо для управления набором инфраструктуры, а отдельные рабочие области функционируют как полностью отдельные рабочие каталоги. С помощью Workspaces возможно управлять несколькими средами инфраструктуры.

</details>

11. Для чего нужен terragrunt?

<details>
  <summary>Ответ</summary>

Terragrunt — это обертка для Terraform, позволяющая решать проблемы, связанные с масштабированием и переиспользованием кода для настройки инфраструктуры. Он позволяет повторно использовать конфигурационные параметры и поддерживает многоуровневые конфигурации и зависимости.

</details>

12. Чем отличается `count` от `for_each`?

<details>
  <summary>Ответ</summary>

`count` — это итерация по списку, который содержит целочисленные элементы, `for_each` — это итерация по корневым ключам словаря, которые могут содержать данные любого типа.

```
resource "aws_instance" "web" {
  count = 3
 
  instance_type = "t2.micro"
  ami           = data.aws_ami.debian_buster.id
  tags = {
    Name = "WebServer-${count.index + 1}"
  }
}
```
Описание ресурса выше создаст 3 одинаковых EC2 инстанса, изменив имя с указанием номера текущего состояния счётчика. `count` начинает отсчет с 0, поэтому чтобы 1 EC2 инстанс был с индексом 1 в имени ему прибавили `1`.

```
resource "aws_instance" "server" {
  for_each = {
    web = { type = "t2.micro", public_ip = true },
    db  = { type = "m5.large", public_ip = false }
  }
 
  instance_type = each.value["type"]
  ami           = data.aws_ami.debian_buster.id
  associate_public_ip_address = each.value["public_ip"]
  tags = {
    Name = "each.key"
  }
}
```
Ресурс выше создаст 2 EC2 инстанса с итерацией по ключам `each.key` и использовав значения вложенных словарей в конфигурации EC2.

**Актуально на 2026:** в примере опечатка — `Name = "each.key"` задаст всем инстансам буквальное имя `each.key`; нужно `Name = each.key` (или `"server-${each.key}"`). `for_each` принимает map или set строк (список приводят через `toset()`). Почему `for_each` надёжнее `count` при удалении элементов — см. вопрос 19.

</details>

13. Что хранится в state-файле и почему к нему нужно относиться как к секрету?

<details>
  <summary>Ответ</summary>

State (`terraform.tfstate`) — JSON, в котором Terraform связывает адреса ресурсов из кода (`aws_instance.web`) с реальными объектами (их ID), хранит все атрибуты ресурсов, зависимости, outputs, а также служебные `serial` и `lineage`. По нему строится план: без state Terraform не знает, чем он уже управляет.

В state попадают значения атрибутов **открытым текстом**: пароли БД, приватные ключи из `tls_private_key`, токены. `sensitive = true` лишь скрывает значение в выводе CLI, но не в state и не в plan-файле.

Практики:
- хранить state в remote backend с шифрованием (например, S3 + SSE-KMS), включить версионирование бакета и ограничить доступ по IAM;
- не коммитить в git: в `.gitignore` добавить `*.tfstate`, `*.tfstate.*`, `.terraform/`, `*.tfplan`;
- не редактировать руками — использовать `terraform state list/show/mv/rm`;
- по возможности не пропускать секреты через state вовсе (ephemeral-значения, write-only аргументы, см. ниже).

</details>

14. Как устроена блокировка state? Чем S3 native locking отличается от блокировки через DynamoDB?

<details>
  <summary>Ответ</summary>

Блокировка не даёт двум процессам (двум инженерам или двум джобам CI) одновременно выполнить `plan`/`apply` над одним state и испортить его. Terraform берёт lock автоматически перед любой операцией, меняющей state.

Раньше для S3 backend блокировку делали через отдельную таблицу DynamoDB (`dynamodb_table`). Начиная с Terraform 1.10 (GA в 1.11) S3 умеет блокировку сам: рядом со state создаётся объект `<key>.tflock` с помощью условной записи S3. Параметр `dynamodb_table` объявлен устаревшим и будет удалён.

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tf-state"
    key          = "prod/network/terraform.tfstate"
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Для плавной миграции можно временно указать и `use_lockfile`, и `dynamodb_table` одновременно. Если процесс упал и lock «завис» — `terraform force-unlock LOCK_ID` (только убедившись, что никто не работает). Подождать освобождения lock можно флагом `-lock-timeout=5m`.

</details>

15. Что такое drift и как его обнаружить и устранить?

<details>
  <summary>Ответ</summary>

Drift — расхождение между реальной инфраструктурой и тем, что записано в state/коде (кто-то поменял security group руками в консоли, ресурс удалили и т.п.).

- `terraform plan -refresh-only` — показывает, чем реальность отличается от state, не предлагая изменений по коду;
- `terraform apply -refresh-only` — принимает эти изменения в state (команда `terraform refresh` устарела и эквивалентна `apply -refresh-only -auto-approve` без возможности просмотра);
- дальше либо обновляют код под реальность, либо обычным `terraform apply` возвращают инфраструктуру к описанному состоянию.

Для регулярного обнаружения запускают по расписанию в CI `terraform plan -detailed-exitcode`: код возврата `0` — изменений нет, `1` — ошибка, `2` — есть изменения (drift), по которому шлют алерт. В HCP Terraform для этого есть health assessments.

</details>

16. Как импортировать существующий ресурс декларативно, через блок `import`?

<details>
  <summary>Ответ</summary>

Начиная с Terraform 1.5 импорт можно описать в коде, а не только выполнить командой `terraform import`:

```hcl
import {
  to = aws_s3_bucket.logs
  id = "my-company-logs"
}
```

Если ресурса `aws_s3_bucket.logs` в коде ещё нет, Terraform может сгенерировать его описание:

```
terraform plan -generate-config-out=generated.tf
```

Преимущества перед CLI-командой: импорт проходит через обычный `plan`/`apply` и ревью в PR, можно импортировать много ресурсов разом, а с Terraform 1.7 внутри `import` поддерживается `for_each`. Сгенерированный код нужно вычистить вручную, а после успешного `apply` блок `import` можно удалить. OpenTofu поддерживает тот же синтаксис.

</details>

17. Как переименовать ресурс или перенести его в модуль, не пересоздавая? Зачем нужен блок `moved`?

<details>
  <summary>Ответ</summary>

Если просто поменять имя ресурса в коде, Terraform увидит «удалить старый адрес, создать новый». Блок `moved` (Terraform 1.1+) говорит, что это тот же объект под новым адресом:

```hcl
moved {
  from = aws_instance.web
  to   = module.web.aws_instance.this
}

moved {
  from = aws_instance.worker[0]
  to   = aws_instance.worker["a"]
}
```

Второй пример — типичная миграция с `count` на `for_each`. В отличие от императивной `terraform state mv`, `moved` проходит ревью, виден в `plan` и автоматически применяется у всех, кто использует модуль. Блоки `moved` оставляют в коде на какое-то время, пока все окружения не обновятся.

</details>

18. Как убрать ресурс из-под управления Terraform, не удаляя его в облаке?

<details>
  <summary>Ответ</summary>

Императивно — `terraform state rm ADDRESS`. Декларативно (Terraform 1.7+) — блок `removed`, при этом сам блок `resource` из кода удаляется:

```hcl
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false
  }
}
```

С `destroy = false` объект просто «забывается» state, реальный ресурс остаётся. Это нужно, например, при передаче ресурса в другой state/репозиторий (там его затем импортируют) или при выводе ресурса из управления Terraform. Как и `moved`, этот способ виден в плане и проходит ревью.

</details>

19. Какая проблема возникает у `count` при удалении элемента из середины списка?

<details>
  <summary>Ответ</summary>

Экземпляры с `count` адресуются по индексу: `aws_instance.vm[0]`, `[1]`, `[2]`. Если из `var.names = ["a", "b", "c"]` удалить `"b"`, то `"c"` сдвинется на индекс `1`: Terraform изменит (или пересоздаст) `vm[1]` под `"c"` и удалит `vm[2]` — вместо того чтобы удалить только `"b"`.

С `for_each` ключом служит само значение, адреса стабильны:

```hcl
resource "aws_instance" "vm" {
  for_each      = toset(var.names)
  instance_type = "t3.micro"
  ami           = var.ami_id
  tags          = { Name = each.key }
}
```

Поэтому `count` оставляют для однотипных взаимозаменяемых ресурсов и для условного создания (`count = var.enabled ? 1 : 0`), а для коллекций именованных объектов используют `for_each`.

</details>

20. Как версионировать модули и провайдеры? Что такое `.terraform.lock.hcl`?

<details>
  <summary>Ответ</summary>

Модули из registry фиксируют через `version`, из git — через `ref` (тег или SHA коммита):

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

module "app" {
  source = "git::https://git.example.com/infra/tf-modules.git//app?ref=v1.4.2"
}
```

Для своих модулей используют SemVer и CHANGELOG, ломающие изменения — только в major-версии.

Провайдеры ограничивают в `required_providers` (`version = "~> 6.0"`), а точные выбранные версии и их хеши Terraform записывает в `.terraform.lock.hcl`. Этот файл **коммитят в git**, чтобы у всех и в CI были одинаковые провайдеры. Обновление — `terraform init -upgrade`; хеши для других ОС добавляются через `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64`. Версии модулей в lock-файл не попадают — поэтому их важно пинить явно.

</details>

21. Workspaces, отдельные директории или Terragrunt — как разделять окружения (dev/stage/prod)?

<details>
  <summary>Ответ</summary>

- **CLI workspaces** — один код и один backend, но отдельный state на каждый workspace. Удобно для одинаковых временных окружений (например, на feature-ветку). Минусы: общие backend и доступы у prod и dev, легко сделать `apply` не в том workspace, различия между окружениями приходится делать условиями по `terraform.workspace`. HashiCorp не рекомендует workspaces для изоляции prod.
- **Отдельные директории** (`envs/prod`, `envs/stage`), которые вызывают общие модули с разными переменными, — отдельный backend/аккаунт на окружение, явные различия, изоляция доступов. Дополнительно state режут по компонентам (network, db, app), чтобы уменьшить blast radius и время `plan`.
- **Terragrunt** убирает дублирование backend/provider-конфигурации между директориями, описывает зависимости между стеками и позволяет запустить план сразу по всему дереву (`terragrunt run --all plan`, в старых версиях `run-all`).

</details>

22. Что такое data source и чем он отличается от resource?

<details>
  <summary>Ответ</summary>

`resource` создаёт и управляет объектом, а `data` только **читает** информацию о существующем объекте (созданном вручную, другой командой или другим state) при каждом `plan`:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```

Для чтения outputs другой конфигурации есть `data "terraform_remote_state"`, но он требует доступа ко всему чужому state (с секретами). Чаще лучше публиковать нужные значения в SSM Parameter Store/Consul или искать ресурсы data source-ами по тегам.

</details>

23. Какие настройки есть в блоке `lifecycle`?

<details>
  <summary>Ответ</summary>

- `create_before_destroy = true` — при замене сначала создать новый объект, потом удалить старый (меньше простоя; имя ресурса должно допускать сосуществование двух копий, например через `name_prefix`);
- `prevent_destroy = true` — `plan` падает с ошибкой, если ресурс будет удалён или пересоздан (защита для БД, бакетов со state). Не спасает, если блок ресурса удалить из кода целиком;
- `ignore_changes = [tags, desired_capacity]` — игнорировать изменения указанных атрибутов вне Terraform (например, autoscaler меняет число инстансов);
- `replace_triggered_by = [terraform_data.release]` — пересоздать ресурс при изменении другого ресурса/атрибута;
- `precondition` / `postcondition` — проверки с понятным `error_message` до и после создания.

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [engine_version] # минорные обновления делает AWS
  }
}
```

</details>

24. Когда нужен явный `depends_on`?

<details>
  <summary>Ответ</summary>

Обычно зависимости неявные: если ресурс ссылается на атрибут другого (`subnet_id = aws_subnet.a.id`), Terraform сам построит граф и создаст их в правильном порядке. Граф можно посмотреть командой `terraform graph`.

`depends_on` нужен, когда зависимость есть, но ссылки в коде нет. Классический пример — IAM-политика должна быть прикреплена к роли раньше, чем Lambda/EC2 с этой ролью начнёт обращаться к S3:

```hcl
resource "aws_lambda_function" "app" {
  # ...
  role       = aws_iam_role.app.arn
  depends_on = [aws_iam_role_policy_attachment.app_s3]
}
```

Злоупотреблять не стоит: `depends_on` на уровне модуля делает зависимым всё содержимое модуля и откладывает чтение его data sources до `apply`, из-за чего план становится менее точным (много `known after apply`).

</details>

25. Что такое provisioners и почему их советуют использовать только в крайнем случае?

<details>
  <summary>Ответ</summary>

Provisioners (`local-exec`, `remote-exec`, `file`) выполняют команды на машине с Terraform или на созданном ресурсе по SSH/WinRM.

Почему их избегают:
- их действия не видны в `plan` и не отражаются в state — Terraform не знает, что они сделали, и не отследит drift;
- выполняются только при создании (или удалении) ресурса, не идемпотентны;
- при ошибке ресурс помечается как tainted и будет пересоздан;
- требуют сетевого доступа и ключей от раннера до ВМ.

Альтернативы: `user_data`/cloud-init, готовые образы через Packer, конфигурирование Ansible после создания ВМ, нативные ресурсы провайдера. Если без запуска команды не обойтись — вместо `null_resource` используют встроенный `terraform_data` (1.4+) с `triggers_replace`.

</details>

26. Как не хранить секреты в state? Чем `sensitive` отличается от `ephemeral`?

<details>
  <summary>Ответ</summary>

- `sensitive = true` у переменной/output только скрывает значение в выводе CLI; в state и plan оно сохраняется.
- `ephemeral = true` (Terraform 1.10+) у переменных и outputs, а также `ephemeral`-ресурсы — значения существуют только во время выполнения и **не пишутся** ни в plan, ни в state.
- Write-only аргументы (Terraform 1.11+, суффикс `_wo`) принимают ephemeral-значения; поскольку Terraform не видит старое значение, для обновления увеличивают парный `*_wo_version`.

```hcl
ephemeral "random_password" "db" {
  length = 24
}

resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1
}
```

В OpenTofu ephemeral-значения и write-only атрибуты есть с 1.11, а state дополнительно можно шифровать целиком.

</details>

27. Чем OpenTofu отличается от Terraform?

<details>
  <summary>Ответ</summary>

В августе 2023 HashiCorp перевела Terraform (с версии 1.6) с open source лицензии MPL 2.0 на Business Source License 1.1, которая запрещает строить на нём конкурирующие коммерческие продукты. Сообщество сделало форк **OpenTofu** под MPL 2.0, проект находится под управлением Linux Foundation и принят в CNCF. В 2025 HashiCorp вошла в состав IBM.

- OpenTofu совместим с конфигурациями, state и провайдерами Terraform уровня 1.5–1.6, CLI — `tofu`, свой registry.
- Возможности, которых нет в Terraform: шифрование state и plan на стороне клиента (1.7), переменные и locals в конфигурации backend и в `source` модулей (1.8), `for_each` в блоках provider и флаг `-exclude` (1.9).
- У Terraform — интеграция с HCP Terraform (Stacks, Sentinel) и своя линейка новых функций; со временем языки расходятся.

Миграция: бэкап state, `tofu init`, `tofu plan` должен показать отсутствие изменений.

</details>

28. Как работает встроенный фреймворк `terraform test`?

<details>
  <summary>Ответ</summary>

С Terraform 1.6 тесты пишут на HCL в файлах `*.tftest.hcl` (обычно в каталоге `tests/`) и запускают `terraform test`. Каждый блок `run` выполняет `plan` или `apply` и проверяет условия `assert`:

```hcl
variables {
  bucket_name = "ci-test-bucket"
}

run "bucket_name_is_passed" {
  command = plan

  assert {
    condition     = aws_s3_bucket.this.bucket == var.bucket_name
    error_message = "Имя бакета не совпадает с переменной"
  }
}
```

По умолчанию `command = apply` — создаются настоящие ресурсы, которые удаляются после теста. С 1.7 есть `mock_provider`, позволяющий тестировать без облака. `expect_failures` проверяет, что валидация переменных действительно срабатывает. Дополнительно: `terraform fmt -check`, `terraform validate`, tflint, для сложных интеграционных сценариев — Terratest (Go).

</details>

29. Как построить CI/CD для Terraform? Что такое Atlantis и policy as code?

<details>
  <summary>Ответ</summary>

Типовой пайплайн на merge request:
1. `terraform fmt -check`, `terraform init`, `terraform validate`, `tflint`;
2. сканирование безопасности конфигурации: Checkov, Trivy (в него влился tfsec), KICS;
3. `terraform plan -out=tfplan` и публикация плана комментарием в PR;
4. проверка политик по `terraform show -json tfplan`: OPA/Conftest, Sentinel или OPA в HCP Terraform (например, «запрещены публичные бакеты», «обязательные теги»);
5. после ревью и approve — `terraform apply tfplan`, то есть применяется именно тот план, который проверили.

**Atlantis** — self-hosted сервис, который по комментариям `atlantis plan` / `atlantis apply` в PR выполняет Terraform и пишет результат в PR, блокируя директорию/workspace до мержа. Аналоги: HCP Terraform, Spacelift, env0, Digger. Доступ CI к облаку лучше давать через OIDC, без долгоживущих ключей, и не допускать параллельных `apply` в один state.

</details>

30. Как работать с несколькими регионами или аккаунтами одного провайдера?

<details>
  <summary>Ответ</summary>

Объявить несколько конфигураций провайдера с `alias` и явно указывать их в ресурсах и модулях:

```hcl
provider "aws" {
  region = "eu-central-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/terraform"
  }
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.us # сертификат для CloudFront должен быть в us-east-1
  domain_name       = "example.com"
  validation_method = "DNS"
}

module "dr" {
  source    = "./modules/app"
  providers = { aws = aws.us }
}
```

Внутри модуля провайдеры не объявляют, а получают от вызывающего кода через `providers`; если модулю нужны сразу несколько, их перечисляют в `configuration_aliases` в `required_providers`.

</details>
