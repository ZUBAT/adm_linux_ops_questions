# Ответы на вопросы для собеседования Senior SRE Engineer

## 1. Linux Systems & Operations

### Базовые системные знания

**Q: Разница между hard link и symbolic link?**
- **Hard link**: указывает на тот же inode, что и оригинальный файл. Файл существует пока есть хотя бы один hard link. Нельзя создавать для директорий и файлов на разных файловых системах.
- **Symbolic link**: указывает на путь к файлу по имени. Если оригинальный файл удален, symlink становится "битым". Можно создавать для директорий и файлов на разных FS.

**Q: Процесс загрузки Linux системы?**
1. **BIOS/UEFI** - инициализация оборудования, поиск загрузочного устройства
2. **Bootloader** (GRUB) - загрузка ядра и initramfs
3. **Kernel** - инициализация ядра, монтирование initramfs
4. **Init system** (systemd/init) - запуск пользовательского пространства
5. **Services** - запуск системных служб по зависимостям

**Q: Что происходит при выполнении `ls -la`?**
1. Shell парсит команду и аргументы
2. Поиск исполняемого файла в PATH
3. Системный вызов execve() для запуска ls
4. ls читает содержимое директории через opendir()/readdir()
5. Получение метаданных файлов через stat()
6. Форматирование и вывод результата в stdout

**Q: Диагностика высокой загрузки CPU?**
- `top`/`htop` - общий обзор процессов
- `ps aux --sort=-%cpu` - процессы по загрузке CPU
- `sar -u 1 10` - статистика использования CPU
- `perf top` - профилирование на уровне ядра
- `iostat` - проверка I/O wait
- `strace -p PID` - системные вызовы процесса

### Файловые системы и хранение

**Q: Разница между ext4, XFS и Btrfs?**
- **ext4**: стабильная, проверенная временем FS. Хороша для общего использования, поддерживает файлы до 16TB
- **XFS**: высокая производительность для больших файлов, хороша для media/streaming, не поддерживает shrinking
- **Btrfs**: современная FS с снапшотами, сжатием, RAID, но менее стабильна для критичных систем

**Q: RAID уровни и применение?**
- **RAID 0**: striping, увеличение производительности, без отказоустойчивости
- **RAID 1**: mirroring, отказоустойчивость, 50% дискового пространства
- **RAID 5**: parity распределен, минимум 3 диска, отказ одного диска
- **RAID 6**: двойная parity, отказ двух дисков
- **RAID 10**: комбинация 1+0, высокая производительность и надежность

**Q: LVM преимущества?**
- Динамическое изменение размера томов
- Создание снапшотов
- Объединение нескольких дисков в один том
- Перемещение данных между дисками онлайн
- Striping и mirroring

**Q: Что такое inode и диагностика исчерпания?**
- **Inode**: структура данных с метаинформацией о файле (права, размер, время)
- **Диагностика**: `df -i` показывает использование inodes
- **Решение**: удаление ненужных файлов или расширение файловой системы

### Сеть и безопасность

**Q: iptables и пример правила?**
- **iptables**: firewall для фильтрации пакетов по цепочкам (INPUT, OUTPUT, FORWARD)
- **Пример**: `iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT` (разрешить SSH с подсети)

**Q: Диагностика сетевых проблем?**
- `ping` - проверка доступности
- `traceroute` - путь пакетов
- `netstat -tulpn` - открытые порты
- `ss -tulpn` - современная альтернатива netstat
- `tcpdump` - захват пакетов
- `iftop` - мониторинг сетевого трафика

**Q: TCP handshake?**
1. **SYN**: клиент отправляет запрос на соединение
2. **SYN-ACK**: сервер подтверждает и отправляет свой SYN
3. **ACK**: клиент подтверждает получение SYN-ACK

**Q: Безопасная настройка SSH?**
- Отключить root login: `PermitRootLogin no`
- Изменить порт: `Port 2222`
- Использовать ключи: `PasswordAuthentication no`
- Настроить fail2ban
- Ограничить пользователей: `AllowUsers username`

## 2. Мониторинг и Observability

### Концепции мониторинга

**Q: Мониторинг vs логирование vs трейсинг?**
- **Мониторинг**: метрики и алерты в реальном времени (CPU, память, latency)
- **Логирование**: запись событий для анализа и отладки
- **Трейсинг**: отслеживание запросов через распределенную систему

**Q: SLI, SLO, SLA?**
- **SLI** (Service Level Indicator): метрика качества сервиса (availability, latency)
- **SLO** (Service Level Objective): целевое значение SLI (99.9% availability)
- **SLA** (Service Level Agreement): коммерческое соглашение с последствиями

**Q: Мониторинг веб-приложения с высокой нагрузкой?**
- **Golden signals**: latency, traffic, errors, saturation
- **RED метрики**: Rate, Errors, Duration
- **USE метрики**: Utilization, Saturation, Errors
- Application metrics: business KPIs
- Infrastructure metrics: CPU, memory, disk, network

### Практические вопросы

**Q: Создание эффективных дашбордов?**
- **Главный дашборд**: SLI/SLO, критичные алерты, система статус
- **Иерархия**: от высокого уровня к детальному
- **Контекст**: временные интервалы, фильтры
- **Actionable**: каждый график должен приводить к действию

**Q: Настройка алертинга без false positives?**
- Использовать percentiles вместо averages
- Настроить hysteresis (разные пороги для включения/выключения)
- Group by релевантным меткам
- Правильные временные окна
- Escalation policies

**Q: MTTR улучшение?**
- Автоматизированное детектирование
- Runbooks и плейбуки
- Быстрый rollback механизм
- Централизованный инцидент менеджмент
- Post-mortem анализ для предотвращения

## 3. Контейнеризация и Оркестрация

### Docker

**Q: Docker image vs container?**
- **Image**: неизменяемый template с приложением и зависимостями
- **Container**: работающий экземпляр image с runtime состоянием

**Q: Оптимизация Dockerfile для production?**
```dockerfile
# Multi-stage builds
FROM node:16-alpine AS builder
# ... build steps

FROM node:16-alpine AS runtime
RUN addgroup -g 1001 -S nodejs && adduser -S app -u 1001
USER app
COPY --from=builder --chown=app:nodejs /app ./
```
- Использовать alpine images
- Multi-stage builds
- Минимизировать количество layers
- Запуск от non-root пользователя
- .dockerignore для исключения файлов

**Q: Multi-stage builds применение?**
- Отделение build dependencies от runtime
- Уменьшение размера финального image
- Улучшение безопасности (меньше attack surface)

**Q: Диагностика проблем производительности контейнера?**
- `docker stats` - ресурсы в реальном времени
- `docker exec -it container_id top` - процессы внутри
- Анализ logs: `docker logs -f container_id`
- cAdvisor для детального мониторинга

### Kubernetes

**Q: Архитектура Kubernetes?**
**Control Plane:**
- **API Server**: центральная точка управления
- **etcd**: хранилище состояния кластера
- **Controller Manager**: поддержание желаемого состояния
- **Scheduler**: размещение pods на nodes

**Q: Deployment vs StatefulSet vs DaemonSet?**
- **Deployment**: stateless приложения, rolling updates
- **StatefulSet**: stateful приложения с persistent storage и сетевой идентификацией
- **DaemonSet**: по одному pod на каждой ноде (мониторинг, логи)

**Q: Service Discovery в Kubernetes?**
- DNS-based: каждый Service получает DNS имя
- Environment variables: автоматически создаются для Services
- Service mesh (Istio): advanced service discovery и routing

**Q: Автоскалирование в Kubernetes?**
- **HPA** (Horizontal Pod Autoscaler): масштабирование по CPU/memory/custom metrics
- **VPA** (Vertical Pod Autoscaler): изменение resource requests
- **Cluster Autoscaler**: добавление/удаление нод

**Q: Ingress Controllers?**
- Управляют внешним доступом к Services
- Layer 7 load balancing
- SSL termination
- Path-based и host-based routing

**Q: Стратегии деплоя в Kubernetes?**
- **Rolling Update**: постепенная замена pods
- **Blue-Green**: полное переключение между версиями
- **Canary**: постепенный rollout для части трафика

### Практические сценарии

**Q: Pod в CrashLoopBackOff диагностика?**
```bash
kubectl describe pod <pod-name>  # События и состояние
kubectl logs <pod-name> --previous  # Логи предыдущего запуска
kubectl get events --sort-by=.metadata.creationTimestamp  # События кластера
kubectl exec -it <pod-name> -- /bin/sh  # Если pod запускается
```

**Q: Высокая доступность в Kubernetes?**
- Multiple replicas с anti-affinity правилами
- Resource limits и requests
- Health checks (liveness/readiness probes)
- PodDisruptionBudgets
- Multi-zone deployment

## 4. Infrastructure as Code и Автоматизация

### IaC концепции

**Q: Организация Terraform кода для большой организации?**
```
terraform/
├── modules/           # Переиспользуемые модули
├── environments/      # Конфиги окружений
│   ├── dev/
│   ├── staging/
│   └── prod/
├── global/           # Общие ресурсы
└── policies/         # Sentinel policies
```

**Q: Immutable infrastructure?**
- Infrastructure components заменяются полностью, а не обновляются
- Исключает configuration drift
- Упрощает rollback и scaling
- Реализуется через AMI, Docker images, Terraform

**Q: Безопасность секретов в IaC?**
- Использовать Vault, AWS Secrets Manager
- Encrypted backends для Terraform state
- Не коммитить secrets в Git
- Использовать data sources для секретов

### CI/CD

**Q: Идеальный CI/CD пайплайн для микросервиса?**
1. **Source**: Git trigger
2. **Build**: компиляция, unit tests
3. **Security**: SAST, dependency scanning
4. **Package**: создание Docker image
5. **Deploy to staging**: automated deployment
6. **Integration tests**: API tests, e2e tests
7. **Security scanning**: DAST, image scanning
8. **Deploy to production**: с approval gates
9. **Monitor**: automated rollback при проблемах

**Q: GitOps подход?**
- Git как single source of truth для декларативной инфраструктуры
- ArgoCD или Flux для синхронизации
- Pull-based deployment вместо push-based
- Автоматический drift detection

## 5. Отказоустойчивость и Катастрофоустойчивость

### Теория и практика

**Q: Circuit Breaker pattern?**
- Предотвращает каскадные сбои
- **Closed**: нормальная работа
- **Open**: блокировка запросов при сбоях
- **Half-Open**: проверка восстановления
- Используется при интеграции с внешними сервисами

**Q: Graceful degradation vs fail-fast?**
- **Graceful degradation**: система продолжает работать с ограниченной функциональностью
- **Fail-fast**: быстрый отказ при обнаружении проблемы для предотвращения распространения

**Q: Система в нескольких зонах доступности?**
- Load balancer с health checks
- Database replication across AZ
- Stateless application design
- Shared storage или data replication
- Network connectivity между AZ

### Disaster Recovery

**Q: План восстановления после катастрофы?**
1. **Assessment**: классификация критичности систем
2. **Backup strategy**: регулярные бэкапы в отдельную зону
3. **Recovery procedures**: документированные шаги
4. **Testing**: регулярное тестирование DR процедур
5. **Communication plan**: уведомления stakeholders

**Q: RTO и RPO?**
- **RTO** (Recovery Time Objective): максимально допустимое время восстановления
- **RPO** (Recovery Point Objective): максимально допустимая потеря данных

## 6. Производительность и Масштабирование

### Анализ производительности

**Q: Диагностика медленного веб-приложения?**
1. **Frontend**: browser dev tools, Core Web Vitals
2. **Network**: CDN, DNS lookup время
3. **Load balancer**: распределение нагрузки
4. **Application**: APM tools (New Relic, DataDog)
5. **Database**: query performance, indexing
6. **Infrastructure**: CPU, memory, I/O

**Q: Типы кеширования?**
- **Browser cache**: статические ресурсы
- **CDN**: географически распределенный cache
- **Application cache**: Redis, Memcached
- **Database cache**: query result cache
- **HTTP cache**: reverse proxy cache

### Масштабирование

**Q: Vertical vs Horizontal scaling?**
- **Vertical**: увеличение мощности одного сервера (scale up)
- **Horizontal**: добавление серверов (scale out)
- Horizontal предпочтительнее для отказоустойчивости

**Q: Автоскалирование для непредсказуемой нагрузки?**
- Реактивное: на основе CPU/memory metrics
- Прогностическое: machine learning для предсказания нагрузки
- Scheduled: известные пики нагрузки
- Queue-based: длина очереди сообщений

## 7. Безопасность

**Q: Защита Kubernetes кластера?**
- Network policies для micro-segmentation
- Pod Security Standards
- RBAC для доступа к API
- Secrets management (Vault)
- Image scanning
- Runtime security monitoring

**Q: Zero trust security model?**
- "Never trust, always verify"
- Каждый запрос должен быть аутентифицирован и авторизован
- Micro-segmentation сети
- Continuous verification

## 8. Cloud Platforms

**Q: Высокодоступная архитектура в AWS?**
- Multi-AZ deployment
- Auto Scaling Groups
- Application Load Balancer
- RDS Multi-AZ
- CloudFront CDN
- Route 53 health checks

**Q: Оптимизация cloud затрат?**
- Reserved instances для предсказуемой нагрузки
- Spot instances для batch workloads
- Right-sizing instances
- Automated resource cleanup
- Cost allocation tags

## 9. Ситуационные вопросы

### Практические сценарии

**Q: Веб-сайт медленно отвечает - действия?**
1. **Проверить мониторинг**: алерты, дашборды
2. **Load balancer**: распределение трафика, health checks
3. **Application servers**: CPU, memory, процессы
4. **Database**: active queries, locks, performance
5. **Network**: latency, packet loss
6. **CDN/Cache**: cache hit ratio
7. **Escalate** если не удается быстро локализовать

**Q: Потеря master ноды Kubernetes?**
1. **Проверить статус кластера**: `kubectl get nodes`
2. **Etcd consistency**: проверить quorum
3. **API Server**: доступность и работоспособность
4. **Восстановление**: запуск новой master ноды
5. **Мониторинг**: убедиться в полном восстановлении

## 10. Лидерство и командная работа

**Q: Онбординг junior SRE?**
- Структурированный план обучения
- Менторинг и pair programming
- Постепенное увеличение ответственности
- Регулярные 1-на-1 встречи
- Обратная связь и цели развития

**Q: Развитие культуры reliability?**
- SLO-based approach
- Blameless post-mortems
- Error budgets для баланса между reliability и фичами
- Knowledge sharing sessions
- Investment в tooling и автоматизацию

**Q: Приоритизация SRE задач?**
- **Инциденты**: highest priority
- **Toil reduction**: автоматизация рутины
- **Reliability projects**: улучшение SLO
- **Capacity planning**: предотвращение проблем
- 50/50 правило: 50% операционной работы, 50% проектной
