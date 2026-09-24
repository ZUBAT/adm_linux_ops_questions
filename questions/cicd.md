## CI / CD

1. Чем отличается Continuous Integration от Continuous Delivery от Continuous Deployment?

<details>
  <summary>Ответ</summary>

Continuous Integration (непрерывная интеграция) - практика интеграции изменений кода из ветки разработки в основную ветку путём инструментов для интеграции.

Continuous Delivery (непрерывная доставка) - практика содержания кода в репозитории в состоянии пригодным для разворачивания на рабочее окружение.

Continuous Deployment (непрерывное разворачивание) - практика доставки каждого изменения в коде продукта на рабочее окружение.

![](imgs/cicdcd.jpg)

Разница между Continuous Delivery и Continuous Deployment очень маленькая. Представим два пайплайна для одного и того же приложения. В каждом есть шаги:

1. Source Control - внесение изменений в систему контроля версий ПО.
2. Build - сборка приложения и прогон unit тестов
3. Staging - деплой на тестовое окружение, прогон интеграционных, нагрузочных и других тестов
4. Production - деплой на окружение с пользователями

Каждый пайплайн запускается автоматически по триггеру из системы контроля версий. В случае Continuous Deployment каждый следующий шаг, будет выполнен автоматически если предыдущий был успешный, включая деплой на Production.

Если же у вас Continuous Delivery, то шаги будут выполняться автоматически только в безопасной среде, а перед деплоем на Production пайплайн остановится и будет ждать ручного подтверждения. Механизм, как это будет реализовано может быть разным. От самого простого, когда ответственный человек должен зайти в пайплайн и нажать кнопку Next, до интерактивного бота с кнопками в корпоративном мессенджере.

</details>

2. Что означает конструкция `when: always` в stage блоке в gitlab CI?

<details>
  <summary>Ответ</summary>

Данная конструкция означает, что stage будет запущен вне зависимости от успешности предыдущего шага.

**Актуально на 2026:** `when` задаётся не для stage, а для отдельной джобы (или внутри `rules`): джоба с `when: always` выполнится независимо от статуса джоб предыдущих стадий — например, для очистки или отправки уведомлений. Другие значения: `on_success` (по умолчанию), `on_failure`, `manual`, `delayed`, `never`.

</details>

3. Что выполняет конструкция `extends: .plan` в gitlab CI?

<details>
  <summary>Ответ</summary>

`extends` используется для повторного использования секции пайплайна (аналог фунции). `.plan` указывает на имя повторяемой секции в пайплайне. Первым в шаге выполняется скрипт из `extends`.

**Актуально на 2026:** точнее, `extends` не выполняет скрипт шаблона перед скриптом джобы, а объединяет ключи: словари сливаются рекурсивно, а массивы (`script`, `before_script`) **заменяются** значением из джобы. Если в джобе есть свой `script`, скрипт из `.plan` не выполнится. Чтобы собрать скрипт из частей шаблона, используют `!reference [.plan, script]`.

</details>

4. В gitlab CI необходимо, чтобы джоба выполнялась всегда только при ручной активации. Что для этого необходимо сделать?

<details>
  <summary>Ответ</summary>

Необходимо добавить `when: manual` в описание заданной джобы. По-умолчанию при использовании `when: manual` параметр `allow_failure` установлен в `true`, поэтому данная джоба будет запускаться автоматически. Чтобы такого не было необходимо также установить параметр `allow_failure: false`.

**Актуально на 2026:** уточнение — ручная джоба не запускается автоматически ни при каком `allow_failure`. `allow_failure: true` (по умолчанию для `when: manual` вне `rules`) означает, что пайплайн не ждёт её и может завершиться без её запуска; с `allow_failure: false` джоба становится блокирующей — следующие стадии ждут ручного запуска. Для `when: manual` внутри `rules` по умолчанию уже `allow_failure: false`.

</details>

5. Из каких этапов состоит типовой CI/CD пайплайн?

<details>
  <summary>Ответ</summary>

Примерный порядок (от быстрых и дешёвых проверок к медленным):
1. **lint / static analysis** — линтеры, форматирование, SAST, проверка секретов в коде (gitleaks);
2. **unit-тесты** и сборка приложения;
3. **build образа/артефакта** — один раз, с неизменяемым тегом (SHA коммита), push в registry;
4. **сканирование** — уязвимости зависимостей и образа (SCA: Trivy, Grype), генерация SBOM, подпись;
5. **deploy в тестовое окружение** + интеграционные, e2e, нагрузочные тесты, DAST;
6. **deploy в production** — автоматически или после ручного approve, с постепенной выкаткой;
7. **post-deploy проверки** — smoke-тесты, мониторинг метрик, автоматический откат.

Принципы: fail fast (быстрые проверки раньше), «собрать один раз — продвигать тот же артефакт» по окружениям, пайплайн описан кодом в репозитории.

</details>

6. Чем артефакты отличаются от кэша в CI?

<details>
  <summary>Ответ</summary>

- **Артефакты** — результат работы джобы, который нужен дальше: бинарник, отчёт тестов, plan-файл Terraform. Гарантированно передаются в последующие джобы и хранятся заданное время (`expire_in`), их можно скачать.
- **Кэш** — ускоритель: зависимости (`node_modules`, `~/.m2`, `.cache/pip`), которые можно в любой момент восстановить заново. Не гарантирован: может отсутствовать или быть устаревшим, поэтому пайплайн обязан работать и без него.

```yaml
# GitLab CI
build:
  stage: build
  cache:
    key:
      files: [package-lock.json]
    paths: [.npm/]
  script:
    - npm ci --cache .npm
    - npm run build
  artifacts:
    paths: [dist/]
    expire_in: 1 week
```

В GitHub Actions аналогично: `actions/cache` для кэша и `actions/upload-artifact` / `actions/download-artifact` для артефактов. Готовые образы и релизные бинарники хранят не в артефактах CI, а в registry (Harbor, Nexus, Artifactory, GHCR).

</details>

7. Чем в GitLab CI `stages` отличается от `needs`?

<details>
  <summary>Ответ</summary>

По умолчанию джобы выполняются по стадиям: джобы одной стадии идут параллельно, а следующая стадия стартует, только когда успешно завершились **все** джобы предыдущей.

`needs` превращает пайплайн в направленный ациклический граф (DAG): джоба стартует, как только завершились перечисленные зависимости, не дожидаясь всей стадии. Заодно `needs` ограничивает, чьи артефакты будут скачаны.

```yaml
stages: [build, test, deploy]

build_backend:
  stage: build
  script: make backend

test_backend:
  stage: test
  needs: [build_backend]   # не ждёт сборки frontend
  script: make test-backend

lint:
  stage: test
  needs: []                # стартует сразу при создании пайплайна
  script: make lint
```

</details>

8. Как управлять тем, когда запускаются джобы и пайплайны в GitLab CI?

<details>
  <summary>Ответ</summary>

Современный способ — `rules` (устаревшие `only`/`except` больше не развиваются). Правила проверяются по порядку, срабатывает первое подходящее:

```yaml
deploy_prod:
  script: ./deploy.sh
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: manual        # только для релизных тегов и по кнопке

docs_build:
  script: make docs
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: [docs/**/*]  # только в MR, где менялась документация
```

Если ни одно правило не подошло, джоба в пайплайн не добавляется.

`workflow:rules` решает, создавать ли пайплайн целиком (например, чтобы не было дублей для ветки и для MR):

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
```

Также доступны `rules:exists` (наличие файла) и `rules:variables` (установить переменные при срабатывании правила).

</details>

9. Что такое environments и runners в GitLab CI?

<details>
  <summary>Ответ</summary>

**Environment** — логическая цель деплоя (staging, production, review/$CI_COMMIT_REF_SLUG). GitLab хранит историю деплоев в окружение, показывает, какая версия где работает, позволяет повторить предыдущий деплой (откат). Protected environments ограничивают, кто может деплоить, и позволяют требовать approve. `resource_group` не даёт двум деплоям в одно окружение идти одновременно.

```yaml
deploy_prod:
  stage: deploy
  environment:
    name: production
    url: https://app.example.com
  resource_group: production
  tags: [prod-deployer]
  script: ./deploy.sh
```

**Runner** — агент, который выполняет джобы. Бывает instance (shared), group и project runner; executor — `docker`, `kubernetes`, `shell`, `docker-autoscaler` и др. Джобы попадают на нужные runner-ы по `tags`. Для prod-деплоя используют отдельный runner с доступом только к prod и только для protected-веток/тегов.

</details>

10. Из чего состоит workflow в GitHub Actions? Что такое matrix?

<details>
  <summary>Ответ</summary>

Workflow — YAML в `.github/workflows/`, запускается событиями из `on` (push, pull_request, schedule, workflow_dispatch). Состоит из **jobs**, которые по умолчанию идут параллельно на разных runner-ах; порядок задаётся `needs`. Job — последовательность **steps**: либо `run` (shell-команда), либо `uses` (готовый action).

`strategy.matrix` запускает одну job для всех комбинаций параметров:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        python: ["3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
      - run: pytest
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

</details>

11. Как переиспользовать код пайплайнов в GitHub Actions и GitLab CI?

<details>
  <summary>Ответ</summary>

GitHub Actions:
- **reusable workflow** — целый workflow с `on: workflow_call` (с inputs и secrets), вызывается как job; подходит для стандартизации «сборка + деплой» во всех репозиториях;
- **composite action** — набор steps в `action.yml`, вызывается как step.

```yaml
jobs:
  deploy:
    uses: my-org/ci-templates/.github/workflows/deploy.yml@v2
    with:
      environment: production
    secrets: inherit
```

GitLab CI: `include` (local, project, remote, template), `extends`, YAML-якоря, `!reference` для вставки отдельных секций и **CI/CD components** с `spec:inputs`, публикуемые в CI/CD Catalog:

```yaml
include:
  - component: $CI_SERVER_FQDN/my-org/ci-components/docker-build@1.2.0
    inputs:
      image_name: my-app
```

Шаблоны версионируют тегами, чтобы изменение шаблона не ломало все пайплайны разом.

</details>

12. Как дать CI доступ к облаку без долгоживущих ключей? Что такое OIDC?

<details>
  <summary>Ответ</summary>

Вместо хранения `AWS_ACCESS_KEY_ID`/JSON-ключа сервисного аккаунта в секретах CI используют federation через OpenID Connect: CI-система выдаёт джобе короткоживущий подписанный JWT, в котором указаны репозиторий, ветка, окружение. Облако (AWS IAM, GCP Workload Identity Federation, Azure) доверяет издателю токена и по условиям на claims (например, только `repo:my-org/app:ref:refs/heads/main`) выдаёт временные креды на минуты.

```yaml
# GitHub Actions
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::111111111111:role/gha-deploy
      aws-region: eu-central-1
```

В GitLab CI токен запрашивают ключевым словом `id_tokens` (с нужным `aud`) и обменивают его на креды облака или токен HashiCorp Vault. Плюсы: нечего красть и ротировать, доступ привязан к конкретному репозиторию/ветке.

</details>

13. Как правильно работать с секретами в CI?

<details>
  <summary>Ответ</summary>

- Не хранить секреты в репозитории и в `.gitlab-ci.yml`/workflow; для проверки — gitleaks/trufflehog в pre-commit и CI.
- Использовать хранилище CI с ограничением области: masked и protected переменные в GitLab (доступны только на protected-ветках/тегах), секреты уровня environment в GitHub с обязательными reviewers.
- Лучше — внешний secret manager (HashiCorp Vault, AWS Secrets Manager) + OIDC, короткоживущие креды.
- Не выводить секреты в лог (`set -x`, `env`), не передавать их в аргументах команд, не класть в артефакты и слои Docker-образа (использовать `RUN --mount=type=secret`).
- Не давать секреты пайплайнам из форков: в GitHub опасен `pull_request_target` вместе с checkout кода из PR.
- Минимальные права токена (`permissions:` в GitHub Actions) и регулярная ротация.

</details>

14. Что такое supply chain security в CI/CD? Зачем пинить actions по SHA, что такое SLSA и SBOM?

<details>
  <summary>Ответ</summary>

Атака на цепочку поставки — компрометация не вашего кода, а зависимостей, инструментов сборки или CI. Меры:
- **Пининг** зависимостей и сторонних actions по полному SHA коммита, а не по тегу: тег можно переписать. В марте 2025 так был скомпрометирован `tj-actions/changed-files` — злоумышленник перевесил теги на вредоносный коммит, и тысячи пайплайнов слили секреты в логи.

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

- **SBOM** (Software Bill of Materials, форматы SPDX/CycloneDX) — список всех компонентов образа/артефакта; генерируется Syft или Trivy и позволяет быстро найти, где используется уязвимая библиотека.
- **Подпись** образов и артефактов (Sigstore cosign, в т.ч. keyless через OIDC) и проверка подписи при деплое (admission-политики Kyverno и т.п.).
- **SLSA** — фреймворк уровней зрелости сборки: provenance (аттестация «кто, из какого коммита и каким пайплайном собрал»), изолированная и защищённая от подмены сборка. В GitHub это делается через `actions/attest-build-provenance`.
- Обновление пинов через Dependabot/Renovate, минимальные права токенов.

</details>

15. Какие есть стратегии деплоя?

<details>
  <summary>Ответ</summary>

- **Recreate** — остановить старую версию и запустить новую; простой, но с даунтаймом.
- **Rolling** — постепенно заменять экземпляры (стандарт Deployment в Kubernetes: `maxSurge`/`maxUnavailable`); без даунтайма, но какое-то время работают обе версии, и откат тоже постепенный.
- **Blue-green** — рядом с текущим (blue) поднимается полный новый (green) стек, трафик переключается разом на балансировщике/DNS; мгновенный откат обратным переключением, но нужны двойные ресурсы.
- **Canary** — новая версия получает небольшую долю трафика (1%, 10%, 50%...), по метрикам ошибок и латентности решается продолжать или откатить; автоматизируется Argo Rollouts, Flagger, service mesh.
- **A/B-тестирование** и **shadow (mirroring)** — маршрутизация по признакам пользователя / копия трафика на новую версию без ответа клиенту.
- **Feature flags** (LaunchDarkly, Unleash, OpenFeature) — отделяют деплой от релиза: код уже в prod, но функция включается флагом для части пользователей и выключается без передеплоя.

Для всех схем, где версии работают одновременно, изменения схемы БД должны быть обратно совместимыми.

</details>

16. Что такое GitOps? Чем pull-модель отличается от push-модели деплоя?

<details>
  <summary>Ответ</summary>

GitOps — подход, при котором желаемое состояние системы (манифесты Kubernetes, Helm values, Kustomize) декларативно описано в git, а агент внутри кластера постоянно сверяет с ним реальное состояние и приводит его в соответствие (reconciliation). Изменения — только через merge request, git служит аудитом и точкой отката (`git revert`).

- **Push**: CI-пайплайн сам выполняет `kubectl apply`/`helm upgrade` в кластер — CI нужны креды от кластера, ручные изменения в кластере никто не откатывает.
- **Pull**: агент (**Argo CD** или **Flux**) внутри кластера забирает изменения из git/OCI-registry и применяет их; наружу креды кластера не выдаются, drift обнаруживается и исправляется автоматически (self-heal).

Типичная схема: CI собирает образ, пушит его и коммитит новый тег в репозиторий с манифестами (или это делает Argo CD Image Updater / Flux image automation), дальше деплой делает GitOps-агент.

</details>

17. Как организовать откат при неудачном деплое?

<details>
  <summary>Ответ</summary>

- **Откат на предыдущий артефакт**, а не пересборка: образы с неизменяемыми тегами позволяют быстро вернуть прошлую версию (`kubectl rollout undo deployment/app`, `helm rollback app 41`, повтор предыдущего деплоя в environment GitLab, `git revert` в GitOps-репозитории).
- **Автоматический откат** по health checks и метрикам: readiness-пробы блокируют rolling update, Argo Rollouts/Flagger прерывают canary при росте ошибок.
- **Blue-green** — обратное переключение трафика.
- **Feature flag** — выключить проблемную функцию без деплоя.
- **Миграции БД** делают по схеме expand/contract: сначала обратно совместимое расширение схемы, затем код, и только потом, в отдельном релизе, удаление старых колонок. Иначе откат кода сломается о новую схему.
- Иногда быстрее **roll forward** — выкатить исправление, если пайплайн быстрый.

</details>

18. Как настроить CI для монорепозитория?

<details>
  <summary>Ответ</summary>

Главное — запускать только джобы для изменившихся частей:

```yaml
# GitHub Actions
on:
  push:
    paths:
      - "services/billing/**"
      - "libs/common/**"
```

```yaml
# GitLab CI
billing_test:
  script: make -C services/billing test
  rules:
    - changes:
        paths: [services/billing/**/*, libs/common/**/*]
        compare_to: refs/heads/main
```

Также используют дочерние пайплайны (`trigger: include:` в GitLab, в т.ч. динамически сгенерированные), фильтры внутри workflow (`dorny/paths-filter`), и build-системы с графом зависимостей и удалённым кэшем (Bazel, Nx, Turborepo, Pants), которые сами определяют затронутые цели. Нужно учитывать общие библиотеки: их изменение должно запускать тесты всех зависимых сервисов.

</details>

19. Что такое DORA-метрики?

<details>
  <summary>Ответ</summary>

Метрики эффективности доставки ПО из исследований DORA (DevOps Research and Assessment, Google Cloud):
- **Deployment frequency** — как часто изменения выкатываются в production;
- **Lead time for changes** — время от коммита до работы в production;
- **Change failure rate** — доля деплоев, вызвавших инцидент/откат/хотфикс;
- **Failed deployment recovery time** (ранее MTTR) — как быстро восстанавливаются после неудачного деплоя.

Первые две показывают скорость (throughput), вторые — стабильность. В отчётах DORA с 2024 года добавлена пятая метрика — **rework rate** (доля незапланированных деплоев для исправления проблем в production). Метрики используют для отслеживания трендов команды, а не для сравнения людей; источники данных — CI/CD, git и система инцидентов.

</details>

20. Какие тесты и проверки запускают в пайплайне?

<details>
  <summary>Ответ</summary>

По пирамиде тестирования — много быстрых внизу, мало медленных наверху:
- **unit** — на каждый коммит, секунды-минуты;
- **integration** — с реальной БД/брокером (сервисы в CI, Testcontainers);
- **contract** (Pact) — совместимость API между сервисами;
- **e2e / UI** — на стенде, дольше и нестабильнее, поэтому их меньше;
- **performance / load** (k6, JMeter) — перед релизом или по расписанию;
- **smoke** — после деплоя в каждое окружение.

Проверки безопасности: **SAST** (анализ исходного кода: Semgrep, SonarQube, CodeQL), **SCA** (уязвимые зависимости), **секреты в коде**, сканирование образов и IaC, **DAST** (атаки на запущенное приложение, OWASP ZAP). Нестабильные (flaky) тесты изолируют и чинят, а не перезапускают бесконечно; отчёты (JUnit XML, покрытие) публикуют в MR.

</details>
