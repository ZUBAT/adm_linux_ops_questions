## Мониторинг и Observability

### Основные понятия

1. Чем мониторинг отличается от observability?

<details>
  <summary>Ответ</summary>

**Мониторинг** отвечает на заранее известные вопросы: «жив ли сервис», «не выросла ли ошибка выше порога». Мы заранее решаем, что измерять и на что алертить (known unknowns).

**Observability (наблюдаемость)** — свойство системы, позволяющее по её внешним сигналам (метрикам, логам, трейсам) ответить на вопросы, которые заранее не формулировались (unknown unknowns): «почему у клиентов из одного региона на одном эндпоинте выросла latency после релиза».

Мониторинг — это практика, которая опирается на наблюдаемость. Чтобы система была наблюдаемой, нужны богатая телеметрия с контекстом (высококардинальные атрибуты в трейсах/логах, корреляция по trace_id) и инструменты для произвольных запросов к ней.

</details>

2. Какие «столпы» observability вы знаете? Для чего каждый из них?

<details>
  <summary>Ответ</summary>

- **Метрики** — числовые временные ряды (счётчики, значения). Дёшево хранить, удобно для дашбордов, алертов и трендов. Мало контекста: видно «что», но не «почему».
- **Логи** — дискретные события с деталями (текст или JSON). Много контекста, но дорого хранить и искать.
- **Трейсы** — путь одного запроса через все сервисы в виде дерева span'ов с длительностями. Показывают, где именно тратится время и где возникла ошибка в распределённой системе.
- **Профили** (часто называют «четвёртым столпом») — на что тратится CPU/память внутри процесса вплоть до функции (flame graph). Continuous profiling: Pyroscope, Parca.

Ценность появляется при корреляции: алерт по метрике → exemplar с trace_id → трейс → логи этого запроса по trace_id → профиль медленного сервиса.

</details>

3. Что такое методы USE, RED и Golden Signals? Когда какой применять?

<details>
  <summary>Ответ</summary>

- **USE** (Brendan Gregg) — для **ресурсов** (CPU, диски, сеть, память, пулы соединений): Utilization (доля занятости), Saturation (очередь/ожидание, например run queue или `await` у дисков), Errors.
- **RED** (Tom Wilkie) — для **сервисов, обрабатывающих запросы**: Rate (RPS), Errors (доля/количество ошибок), Duration (распределение времени ответа, перцентили).
- **Golden Signals** (Google SRE) — Latency, Traffic, Errors, Saturation. По сути RED + насыщенность.

Пример RED на PromQL:
```
sum by (service) (rate(http_requests_total[5m]))                                   # Rate
sum by (service) (rate(http_requests_total{code=~"5.."}[5m]))
  / sum by (service) (rate(http_requests_total[5m]))                               # Errors
histogram_quantile(0.99, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))  # Duration
```

</details>

4. Что такое whitebox и blackbox мониторинг?

<details>
  <summary>Ответ</summary>

- **Whitebox** — метрики изнутри системы: приложение само отдаёт `/metrics`, экспортеры читают внутреннее состояние (node_exporter, postgres_exporter). Помогают понять причину.
- **Blackbox** — проверка «снаружи», как это видит пользователь: HTTP/TCP/ICMP/DNS-пробы, синтетические сценарии. Показывают симптом, даже если внутренний мониторинг сломан вместе с сервисом.

В Prometheus blackbox делается через `blackbox_exporter`:
```
probe_success{job="blackbox-http"} == 0
probe_ssl_earliest_cert_expiry - time() < 86400 * 14   # сертификат истекает через < 14 дней
```
Хорошая практика: алертить по симптомам (blackbox, SLO), а whitebox использовать для диагностики.

</details>

### Prometheus

5. Как устроен Prometheus? Что такое pull-модель и в чём её плюсы и минусы?

<details>
  <summary>Ответ</summary>

Prometheus сам периодически (`scrape_interval`) ходит по HTTP на `/metrics` целей (targets), найденных через статический конфиг или service discovery, и сохраняет ряды в локальную TSDB (блоки по 2 часа + WAL). Поверх данных вычисляются recording/alerting rules, алерты отправляются в Alertmanager.

Плюсы pull:
- Prometheus знает список целей и сразу видит, что цель недоступна (метрика `up == 0`);
- легко вручную посмотреть метрики `curl host:9100/metrics`;
- нагрузку регулирует сервер мониторинга, а не клиенты.

Минусы:
- нужна сетевая доступность целей (NAT, фаерволы);
- неудобно для короткоживущих задач (для них Pushgateway);
- один инстанс Prometheus масштабируется только вертикально и хранит данные локально.

</details>

6. Какие типы метрик есть в Prometheus?

<details>
  <summary>Ответ</summary>

- **Counter** — только растёт (сбрасывается в 0 при рестарте процесса). Пример: `http_requests_total`, `node_network_receive_bytes_total`. Сам по себе почти бесполезен, используется через `rate()`/`increase()`.
- **Gauge** — может расти и падать: `node_memory_MemAvailable_bytes`, температура, размер очереди. С ним используют `avg_over_time`, `delta`, `deriv`, `predict_linear`.
- **Histogram** — раскладывает наблюдения по корзинам (buckets) с верхней границей `le`, экспонирует `_bucket`, `_sum`, `_count`. Квантиль считается на стороне Prometheus и **агрегируется** между инстансами. Точность зависит от выбора корзин. Есть также native histograms с автоматическими корзинами.
- **Summary** — квантили считаются на стороне клиента (`{quantile="0.99"}`) + `_sum` и `_count`. Точны для одного инстанса, но **не агрегируются**: усреднять перцентили разных подов математически некорректно.

На практике для latency почти всегда выбирают histogram.

</details>

7. Что такое labels и кардинальность? Чем опасна высокая кардинальность?

<details>
  <summary>Ответ</summary>

Каждая уникальная комбинация имени метрики и значений меток — это отдельный временной ряд. Кардинальность — количество таких рядов. Например, `http_requests_total{method, path, code}` при 5 методах × 200 путей × 10 кодов × 50 подов = 500 000 рядов.

Опасность: рост потребления RAM (head block держит все активные ряды в памяти), медленные запросы, OOM Prometheus, дорогие хранилища. Типичные ошибки — класть в метки `user_id`, `request_id`, полный URL с параметрами, email, IP клиента.

Как искать виновников:
```
topk(10, count by (__name__)({__name__=~".+"}))
prometheus_tsdb_head_series
```
Также страница Status → TSDB Status (`/api/v1/status/tsdb`) и `promtool tsdb analyze /prometheus`. Лечение: убрать метку в коде, нормализовать путь (`/users/{id}`), удалять метки через `metric_relabel_configs` (`action: labeldrop`), ограничивать `sample_limit` в scrape-конфиге.

</details>

8. Чем отличаются `rate()`, `irate()` и `increase()`?

<details>
  <summary>Ответ</summary>

Все три работают только с counter и корректно обрабатывают сброс счётчика.

- `rate(x[5m])` — средняя скорость роста **в секунду** за всё окно (с экстраполяцией на края). Сглаженная, подходит для алертов и recording rules.
- `irate(x[5m])` — скорость по **двум последним** точкам в окне. Очень чувствительна к всплескам, хороша для детальных графиков, плохо подходит для алертов (дёргается).
- `increase(x[1h])` — насколько счётчик вырос за окно; по сути `rate(x[1h]) * 3600`. Из-за экстраполяции может вернуть дробное число даже для целочисленного счётчика.

```
rate(http_requests_total{job="api"}[5m])          # RPS
increase(http_requests_total{job="api"}[1h])      # запросов за час
```
Правила: сначала `rate`, потом `sum` (не наоборот). Окно должно быть минимум в 4 раза больше `scrape_interval`, иначе в окно может попасть меньше двух точек и ряд пропадёт.

</details>

9. Как посчитать 95-й перцентиль latency по гистограмме?

<details>
  <summary>Ответ</summary>

```
histogram_quantile(
  0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{job="api"}[5m]))
)
```
Если нужен перцентиль в разрезе сервиса/эндпоинта, метку добавляют в `by`, но `le` обязательно сохраняется: `sum by (service, le) (...)`.

Средняя latency считается через `_sum` и `_count`:
```
sum(rate(http_request_duration_seconds_sum[5m])) / sum(rate(http_request_duration_seconds_count[5m]))
```
Доля запросов быстрее 300 мс (удобно для SLO, если есть корзина `le="0.3"`):
```
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) / sum(rate(http_request_duration_seconds_count[5m]))
```
Нюанс: результат `histogram_quantile` — линейная интерполяция внутри корзины, поэтому точность зависит от границ корзин.

</details>

10. Приведите несколько полезных PromQL-запросов для инфраструктуры (node_exporter).

<details>
  <summary>Ответ</summary>

```
# Загрузка CPU, %
100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# Доступная память, %
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Диск закончится в ближайшие 24 часа (прогноз по тренду за 6 часов)
predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24 * 3600) < 0

# Недоступные цели
up == 0

# Метрика вообще перестала приходить
absent(up{job="node"})

# Рестарты контейнеров за час (kube-state-metrics)
increase(kube_pod_container_status_restarts_total[1h]) > 3
```

</details>

11. Что такое recording rules и alerting rules? Зачем нужен `for`?

<details>
  <summary>Ответ</summary>

**Recording rules** заранее вычисляют тяжёлое выражение и сохраняют результат как новый ряд — дашборды и алерты работают быстрее. Имена по соглашению `level:metric:operations`.

**Alerting rules** — выражение, при непустом результате которого создаётся алерт. `for` — сколько условие должно держаться непрерывно: до этого алерт в состоянии `pending`, после — `firing`. Это защита от флаппинга на кратковременных всплесках.

```yaml
groups:
  - name: api
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Ошибок 5xx больше 5% на {{ $labels.job }}"
          runbook_url: https://wiki.example.com/runbooks/high-error-rate
```
Проверка синтаксиса: `promtool check rules rules.yml`, юнит-тесты правил: `promtool test rules tests.yml`.

</details>

12. Что делает Alertmanager? Объясните grouping, inhibition и silences.

<details>
  <summary>Ответ</summary>

Alertmanager принимает алерты от Prometheus, дедуплицирует (в т.ч. от HA-пары Prometheus), группирует, маршрутизирует по получателям (Slack, Telegram, PagerDuty, email, webhook) и повторяет уведомления.

- **Grouping** — объединение похожих алертов в одно уведомление по `group_by`. `group_wait` — сколько ждать перед первой отправкой группы, `group_interval` — как часто слать изменения в группе, `repeat_interval` — как часто напоминать о неизменной группе.
- **Inhibition** — подавление одних алертов при наличии других. Например, если нода лежит, не нужно слать warning'и по всем сервисам на ней.
- **Silences** — временное ручное заглушение по матчерам (плановые работы). Создаются в UI или `amtool silence add alertname="DiskFull" instance="db01" --duration=2h --comment="замена диска"`.

```yaml
route:
  receiver: team-slack
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: ['severity="critical"']
      receiver: oncall-pager
inhibit_rules:
  - source_matchers: ['alertname="NodeDown"']
    target_matchers: ['severity="warning"']
    equal: ['instance']
```

</details>

13. Что такое exporter? Какие экспортеры вы использовали?

<details>
  <summary>Ответ</summary>

Exporter — процесс, который собирает метрики из системы, не умеющей отдавать их в формате Prometheus, и публикует их на `/metrics`. Примеры:
- `node_exporter` — метрики ОС Linux (CPU, память, диски, сеть, filesystem), порт 9100;
- `blackbox_exporter` — пробы HTTP/TCP/ICMP/DNS;
- `postgres_exporter`, `mysqld_exporter`, `redis_exporter`, `kafka_exporter`;
- `kube-state-metrics` — состояние объектов Kubernetes (deployments, pods, restarts), cAdvisor (встроен в kubelet) — ресурсы контейнеров;
- `nginx-prometheus-exporter`, `process-exporter`, `snmp_exporter`.

Собственные метрики приложения лучше отдавать напрямую через клиентскую библиотеку (или OpenTelemetry SDK), а для нестандартных источников на хосте удобен textfile collector node_exporter: скрипт по cron пишет файл `*.prom` в каталог `--collector.textfile.directory`.

</details>

14. Что такое Pushgateway и когда его стоит (и не стоит) использовать?

<details>
  <summary>Ответ</summary>

Pushgateway — промежуточный сервис, куда короткоживущие задачи (cron, бэкап, batch-job) **пушат** метрики перед завершением, а Prometheus уже обычным образом скрейпит сам Pushgateway.

```bash
echo "backup_last_success_timestamp_seconds $(date +%s)" \
  | curl --data-binary @- http://pushgateway:9091/metrics/job/backup/instance/db01
```
Алерт: `time() - backup_last_success_timestamp_seconds > 26 * 3600`.

Когда **не** использовать: как способ превратить pull в push для долгоживущих сервисов. Проблемы: Pushgateway — единая точка отказа, он не удаляет старые метрики сам (метрики умершего job'а висят вечно), теряется смысл `up` для конкретного инстанса. В scrape-конфиге для него обычно ставят `honor_labels: true`, чтобы сохранить метки `job`/`instance` из пуша.

</details>

15. Как Prometheus находит цели в Kubernetes?

<details>
  <summary>Ответ</summary>

Через `kubernetes_sd_configs` с ролями `node`, `pod`, `service`, `endpoints`, `endpointslice`, `ingress`. Prometheus опрашивает API-сервер и получает список целей с метаданными в метках `__meta_kubernetes_*`, которые затем фильтруются и переименовываются через `relabel_configs`:

```yaml
- job_name: pods
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: "true"
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
    - source_labels: [__meta_kubernetes_pod_name]
      target_label: pod
```
На практике чаще используют **Prometheus Operator** (kube-prometheus-stack): цели описываются CRD `ServiceMonitor` / `PodMonitor` (выбор по label selector), правила — `PrometheusRule`, конфиг Alertmanager — `AlertmanagerConfig`. Prometheus'у нужны RBAC-права на `get/list/watch` pods, services, endpoints.

</details>

16. Как организовать долговременное хранение метрик и HA для Prometheus? Сравните Thanos, VictoriaMetrics, Mimir.

<details>
  <summary>Ответ</summary>

Локальная TSDB Prometheus рассчитана на недели (`--storage.tsdb.retention.time`), не реплицируется и не масштабируется горизонтально. Для HA ставят два одинаковых Prometheus, а для длинного хранения и глобального запроса — одно из решений:

- **Thanos** — sidecar рядом с Prometheus выгружает 2-часовые блоки в объектное хранилище (S3/GCS/MinIO). Компоненты: Querier (глобальный запрос + дедупликация HA-пары по external label `replica`), Store Gateway (чтение из бакета), Compactor (компакция и downsampling 5m/1h), Ruler. Есть и режим Receive (remote_write).
- **VictoriaMetrics** — отдельная TSDB, совместимая с PromQL (расширение MetricsQL), принимает `remote_write`. Single-node или кластер `vminsert`/`vmselect`/`vmstorage`. Отличается экономным потреблением диска и RAM, простотой эксплуатации; `vmagent` может заменить Prometheus как сборщик.
- **Grafana Mimir** (развитие Cortex) — горизонтально масштабируемое multi-tenant хранилище поверх объектного хранилища, приём через `remote_write`.

```yaml
remote_write:
  - url: http://vminsert:8480/insert/0/prometheus/api/v1/write
```

</details>

17. Для чего используется Grafana? Как хранить дашборды «как код»?

<details>
  <summary>Ответ</summary>

Grafana — платформа визуализации поверх множества data source'ов: Prometheus/VictoriaMetrics/Mimir (метрики), Loki/Elasticsearch (логи), Tempo/Jaeger (трейсы), SQL-базы. Умеет переменные (templating: `$namespace`, `$instance`), переходы между сигналами (метрика → трейс по exemplar, трейс → логи), собственный алертинг, аннотации (например, отметки деплоев).

Отличие от Kibana: Kibana заточена под Elasticsearch (поиск по логам, аналитика документов), Grafana — универсальная, в первую очередь для временных рядов.

«Как код»:
- provisioning: YAML-файлы в `/etc/grafana/provisioning/datasources` и `/dashboards` + JSON-дашборды из git;
- в Kubernetes — ConfigMap с меткой `grafana_dashboard: "1"`, которую подхватывает sidecar в helm-чарте;
- Terraform provider `grafana`, Grafonnet (Jsonnet) для генерации дашбордов.

</details>

### Логи

18. Сравните ELK/EFK и Grafana Loki.

<details>
  <summary>Ответ</summary>

**ELK/EFK**: агент (Filebeat/Fluent Bit/Fluentd) → (опционально Logstash/Kafka) → Elasticsearch/OpenSearch → Kibana. Elasticsearch строит **полнотекстовый индекс** по содержимому: мощный поиск и аналитика по любому полю, но большой расход CPU/RAM/диска и сложная эксплуатация кластера (шарды, ILM, heap).

**Loki**: агент (Promtail — устарел, сейчас Grafana Alloy, либо Fluent Bit/OTel Collector) → Loki → Grafana. Индексируются **только метки** (`namespace`, `app`, `level`), сами строки сжатыми чанками лежат в объектном хранилище. Намного дешевле, но поиск по содержимому — это grep по чанкам в выбранном потоке. Метки должны быть низкокардинальными.

```
{namespace="prod", app="nginx"} |= "error" | json | status >= 500
sum by (status) (count_over_time({app="nginx"} | json [5m]))
```
Как сохранить логи при недоступности хранилища: буфер на агенте (filesystem buffer у Fluent Bit/Vector) или очередь Kafka между агентами и хранилищем.

</details>

19. Что такое структурированные логи и почему они лучше?

<details>
  <summary>Ответ</summary>

Структурированный лог — запись в машиночитаемом формате (обычно JSON) с фиксированными полями вместо свободного текста:
```json
{"ts":"2026-09-24T10:15:02Z","level":"error","service":"payments","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","user_id":42,"msg":"card declined","duration_ms":183}
```
Плюсы: не нужно писать хрупкие regex/grok-парсеры, можно фильтровать и агрегировать по полям, легко коррелировать с трейсами по `trace_id`.

Практики: писать в stdout/stderr (в контейнерах), единые имена полей во всех сервисах, уровни логирования и их динамическое переключение, не писать секреты и персональные данные (маскирование), сэмплировать шумные debug-логи.

</details>

20. Как настраивается ротация логов на Linux-сервере и в контейнерах?

<details>
  <summary>Ответ</summary>

**logrotate** (`/etc/logrotate.d/app`):
```
/var/log/app/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 app app
    postrotate
        systemctl kill -s HUP app.service
    endscript
}
```
`postrotate` + сигнал заставляет приложение переоткрыть файл. Если приложение не умеет, используют `copytruncate` (с риском потерять строки в момент копирования). Проверка: `logrotate -d /etc/logrotate.d/app` (dry-run), `logrotate -f` — принудительно.

**journald**: `SystemMaxUse=2G` в `/etc/systemd/journald.conf`, ручная чистка `journalctl --vacuum-size=1G` / `--vacuum-time=7d`.

**Docker**: драйвер json-file с `"log-opts": {"max-size": "100m", "max-file": "3"}` в `daemon.json`. **Kubernetes**: kubelet `containerLogMaxSize` и `containerLogMaxFiles`.

Классическая проблема: удалили большой лог через `rm`, а место не освободилось — процесс держит дескриптор. Найти: `lsof +L1`, освободить: `: > /proc/<pid>/fd/<N>` или рестарт процесса.

</details>

### Трейсинг и OpenTelemetry

21. Что такое distributed tracing? Что такое span и context propagation?

<details>
  <summary>Ответ</summary>

Распределённый трейсинг позволяет проследить один запрос через цепочку сервисов. **Trace** — дерево **span'ов**; span — одна операция (входящий HTTP-запрос, SQL-запрос, вызов другого сервиса) с началом, длительностью, статусом, атрибутами и ссылкой на родительский span.

**Context propagation** — передача `trace_id` и `span_id` между сервисами, обычно в HTTP-заголовке W3C `traceparent`:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```
(версия-trace_id-parent_span_id-флаги, последний бит — sampled). Для очередей контекст кладут в заголовки сообщений.

Бэкенды: **Jaeger** (хранилища Cassandra/Elasticsearch/OpenSearch, в v2 построен на OTel Collector), **Grafana Tempo** (хранит трейсы в объектном хранилище, индексирует минимум, поиск по trace_id и через TraceQL), Zipkin, коммерческие APM.

</details>

22. Что такое OpenTelemetry? Из чего он состоит?

<details>
  <summary>Ответ</summary>

OpenTelemetry (OTel) — вендоронезависимый стандарт и набор инструментов CNCF для генерации и передачи телеметрии (трейсы, метрики, логи, профили в развитии).

- **API и SDK** для языков + **автоинструментирование** (Java agent, Python/Node.js auto-instrumentation, OTel Operator в k8s инжектит его в поды по аннотации).
- **OTLP** — протокол передачи: gRPC на порту 4317, HTTP/protobuf на 4318.
- **Collector** — отдельный процесс-конвейер: `receivers` → `processors` → `exporters`. Разворачивается как агент (DaemonSet/sidecar) и/или как gateway (Deployment).
- Semantic conventions — единые имена атрибутов (`http.request.method`, `service.name`).

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
  batch: {}
exporters:
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```
Плюс Collector'а: приложения шлют в один стандартный протокол, а куда дальше уходят данные и как они фильтруются/обогащаются — решается конфигом без изменения кода.

</details>

23. Что такое сэмплирование трейсов? Чем head sampling отличается от tail sampling?

<details>
  <summary>Ответ</summary>

Хранить 100% трейсов при высоком RPS слишком дорого, поэтому часть отбрасывают.

- **Head sampling** — решение принимается в начале трейса (в корневом сервисе) и передаётся дальше флагом в `traceparent`. Просто и дёшево, но решение принимается «вслепую»: редкая ошибка или медленный запрос могут не попасть в выборку.
  ```
  OTEL_TRACES_SAMPLER=parentbased_traceidratio
  OTEL_TRACES_SAMPLER_ARG=0.1
  ```
- **Tail sampling** — решение принимается после завершения трейса (в OTel Collector, процессор `tail_sampling`): можно сохранить все трейсы с ошибками, все медленные и, например, 5% остальных. Требует буферизации в памяти, а все span'ы одного трейса должны попадать на один и тот же Collector (перед ним ставят слой с `loadbalancing` exporter по trace_id).

```yaml
tail_sampling:
  decision_wait: 10s
  policies:
    - name: errors
      type: status_code
      status_code: {status_codes: [ERROR]}
    - name: slow
      type: latency
      latency: {threshold_ms: 500}
    - name: rest
      type: probabilistic
      probabilistic: {sampling_percentage: 5}
```

</details>

24. Что такое exemplars и как связать метрики, логи и трейсы?

<details>
  <summary>Ответ</summary>

**Exemplar** — пример конкретного наблюдения, прикреплённый к точке метрики, чаще всего к корзине гистограммы, вместе с `trace_id`. На графике latency в Grafana видны точки-exemplar'ы, клик по которым открывает конкретный медленный трейс в Tempo/Jaeger. В Prometheus включается флагом `--enable-feature=exemplar-storage`, приложение должно отдавать метрики в формате OpenMetrics.

Общий принцип корреляции — единые идентификаторы и метки во всех сигналах:
- `trace_id`/`span_id` пишутся в каждую строку лога (OTel SDK делает это через интеграции с логгерами);
- одинаковые `service.name`, `namespace`, `pod` в метриках, логах и трейсах;
- в Grafana настраиваются переходы: trace → logs (по trace_id), trace → metrics, logs → trace (derived fields).

</details>

### Алертинг и SLO

25. Что такое алертинг по SLO и burn rate? Как выглядит multi-window multi-burn-rate алерт?

<details>
  <summary>Ответ</summary>

Вместо алертов по «CPU > 80%» алертят по тому, насколько быстро расходуется **error budget**. При SLO 99.9% на 30 дней бюджет — 0.1% ошибочных запросов. **Burn rate** — во сколько раз текущая доля ошибок превышает допустимую: burn rate 1 — бюджет закончится ровно к концу окна, 14.4 — за ~2 дня (30 / 14.4 ≈ 2.08).

Рекомендация из Google SRE Workbook — несколько пар окон (длинное — для значимости, короткое — чтобы алерт быстро гас после починки):
- burn rate 14.4 за 1h и 5m → page (сожжено 2% бюджета за час);
- burn rate 6 за 6h и 30m → page;
- burn rate 1 за 3d и 6h → тикет.

```
(
  sum(rate(http_requests_total{job="api",code=~"5.."}[1h])) / sum(rate(http_requests_total{job="api"}[1h])) > (14.4 * 0.001)
)
and
(
  sum(rate(http_requests_total{job="api",code=~"5.."}[5m])) / sum(rate(http_requests_total{job="api"}[5m])) > (14.4 * 0.001)
)
```
Генерировать такие правила удобно через Sloth или Pyrra.

</details>

26. Что такое alert fatigue и как с ним бороться?

<details>
  <summary>Ответ</summary>

Alert fatigue — ситуация, когда алертов так много (и большинство не требует действий), что дежурные начинают их игнорировать и пропускают действительно важные.

Что делать:
- алертить по **симптомам**, влияющим на пользователя (SLO, ошибки, latency), а не по каждой причине; причины — на дашборды;
- каждый page-алерт должен быть **actionable** и иметь runbook (`runbook_url` в аннотациях);
- разделять severity: page (разбудить ночью) и ticket/warning (в рабочее время);
- `for`, гистерезис, окна `rate` побольше против флаппинга;
- группировка и inhibition в Alertmanager, дедупликация;
- регулярный разбор статистики: какие алерты срабатывали, сколько из них привели к действию; шумные — чинить, переделывать или удалять.

</details>

27. Как мониторить сам мониторинг?

<details>
  <summary>Ответ</summary>

- **Watchdog / Dead man's switch** — алерт, который срабатывает всегда (`expr: vector(1)`) и отправляется во внешний сервис (Healthchecks.io, Dead Man's Snitch, PagerDuty heartbeat). Если уведомления перестали приходить — сломан Prometheus, Alertmanager или путь доставки.
- Два Prometheus в HA-паре, которые скрейпят друг друга; Alertmanager в кластере из 2–3 узлов (gossip).
- Метрики самого Prometheus:
  ```
  up{job="prometheus"} == 0
  rate(prometheus_rule_evaluation_failures_total[5m]) > 0
  rate(prometheus_notifications_dropped_total[5m]) > 0
  prometheus_tsdb_head_series                       # рост кардинальности
  rate(prometheus_tsdb_compactions_failed_total[1h]) > 0
  ```
- Внешний blackbox-мониторинг из другой площадки/облака: если упал весь ЦОД вместе с мониторингом, кто-то снаружи должен это заметить.
- Если используется SaaS (Datadog, New Relic) — дублировать критичные проверки независимым простым инструментом.

</details>

### Профилирование и eBPF

28. Какие eBPF-инструменты наблюдаемости вы знаете? Чем eBPF удобен?

<details>
  <summary>Ответ</summary>

eBPF позволяет безопасно выполнять небольшие программы в ядре на событиях (kprobes, tracepoints, uprobes, сетевые хуки) без модулей ядра и изменения кода приложений, с низкими накладными расходами. Нужно ядро с поддержкой BTF (обычно 5.x+) и права root/`CAP_BPF`.

- **bcc-tools**: `execsnoop` (новые процессы), `opensnoop` (открытие файлов), `biolatency` (гистограмма latency дисков), `tcplife`, `tcpretrans`, `runqlat` (ожидание в очереди CPU), `profile`.
- **bpftrace** — однострочники:
  ```bash
  bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
  bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'
  ```
- В Kubernetes: **Cilium Hubble** (сетевые потоки L3–L7), **Pixie**, **Grafana Beyla** (автоинструментирование HTTP/gRPC без изменения кода), **Parca / Pyroscope** (continuous profiling), **Tetragon** и **Falco** (безопасность/runtime).

</details>

29. Как найти, на что тратит CPU процесс? Что такое flame graph?

<details>
  <summary>Ответ</summary>

Профилировщик с заданной частотой снимает стеки вызовов; функции, которые чаще встречаются в стеках, потребляют больше CPU.

```bash
perf top -p <pid>                                    # в реальном времени
perf record -F 99 -g -p <pid> -- sleep 30            # запись профиля со стеками
perf report
```
**Flame graph** — визуализация стеков: по оси X ширина = доля сэмплов (не время!), по оси Y — глубина стека. Широкие «плато» наверху — горячие функции. Строится из `perf script` скриптами Брендана Грегга (`stackcollapse-perf.pl` + `flamegraph.pl`) или сразу в Pyroscope/Grafana.

Для управляемых языков нужны свои средства или символы: Go — `net/http/pprof` и `go tool pprof`, Java — async-profiler, Python — py-spy. **Continuous profiling** (Pyroscope, Parca) постоянно снимает профили со всего кластера с низким оверхедом, что позволяет сравнивать профили до и после релиза.

</details>
