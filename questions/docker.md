## Docker

1. Что такое Docker? В чем отличие контейнера от образа?

<details>
  <summary>Ответ</summary>

Docker - программное обеспечение для автоматизации развёртывания и управления приложениями в средах с поддержкой контейнеризации.

Образ - шаблон приложения, который содержит слои файловой системы в режиме "только-чтение".

Контейнер - запущенный образ приложения, который кроме нижних слоев в режиме "только чтение" содержит верхний слой в режиме "чтение-запись".

</details>

2. Какие инструкции есть у Dockerfile?
<details>
  <summary>Ответ</summary>

| Инструкция | Описание |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FROM | Задаёт базовый (родительский) образ. |
| LABEL | Описывает метаданные. Например — сведения о том, кто создал и поддерживает образ. |
| ENV | Устанавливает постоянные переменные среды. |
| RUN | Выполняет команду и создаёт слой образа. Используется для установки в контейнер пакетов. |
| COPY | Копирует в контейнер файлы и директории. |
| ADD | Копирует файлы и директории в контейнер, может распаковывать локальные .tar-файлы. |
| CMD | Описывает команду с аргументами, которую нужно выполнить когда контейнер будет запущен. Аргументы могут быть переопределены при запуске контейнера. В файле может присутствовать лишь одна инструкция CMD. |
| WORKDIR | Задаёт рабочую директорию для следующей инструкции. |
| ARG | Задаёт переменные для передачи Docker во время сборки образа. |
| ENTRYPOINT | Предоставляет команду с аргументами для вызова во время выполнения контейнера. Аргументы не переопределяются. |
| EXPOSE | Указывает на необходимость открыть порт. |
| VOLUME | Создаёт точку монтирования для работы с постоянным хранилищем. |

</details>

3. Чем отличается *CMD* от *ENTRYPOINT* в Dockerfile?

<details>
  <summary>Ответ</summary>

Инструкции CMD и ENTRYPOINT выполняются в момент запуска контейнера, тольо инструкция CMD позволяет переопределить передаваемые команде аргументы.

**Пример 1. CMD:**
Опишем сборку образа в Dockerfile.
```
FROM alpine  
CMD ["ping", "8.8.8.8"]  
```
В инструкцию CMD передаются 2 аргумента. Выполним сборку образа `docker build -t test .` и запустим контейнер.
```
$ docker run test
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=43 time=32.976 ms
64 bytes from 8.8.8.8: seq=1 ttl=43 time=31.998 ms
64 bytes from 8.8.8.8: seq=2 ttl=43 time=31.843 ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 31.708/33.316/36.823 ms
```
Теперь передадим 2 новых аргумента для запуска контейнера.
```
$ docker run test traceroute 1.1.1.1
traceroute to 1.1.1.1 (1.1.1.1), 30 hops max, 46 byte packets
 1  172.17.0.1 (172.17.0.1)  0.017 ms  0.016 ms  0.009 ms
 2  192.168.168.1 (192.168.168.1)  0.996 ms  1.553 ms  2.069 ms
 3  *  *  *
 4  lag-2-435.bgw01.samara.ertelecom.ru (85.113.62.125)  1.454 ms  1.427 ms  1.984 ms
 5  172.68.8.3 (172.68.8.3)  19.685 ms  15.722 ms  15.565 ms
 6  172.68.8.2 (172.68.8.2)  15.846 ms  22.696 ms  35.093 ms
 7  one.one.one.one (1.1.1.1)  17.439 ms  17.670 ms  24.202 ms
```
`ping` заменен на traceroute, IP адрес заменен на 1.1.1.1.

**Пример 2. ENTRYPOINT:**
Опишем сборку образа в Dockerfile.
```
FROM alpine  
ENTRYPOINT ["ping", "8.8.8.8"]
```
В инструкцию ENTRYPOINT передаются 2 аргумента. Выполним сборку образа `docker build -t test .` и запустим контейнер.
```
$ docker run test2
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=43 time=36.189 ms
64 bytes from 8.8.8.8: seq=1 ttl=43 time=44.120 ms
64 bytes from 8.8.8.8: seq=2 ttl=43 time=44.584 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 36.189/41.631/44.584 ms
```
Теперь передадим изменим один из аргументов для запуска контейнера.
```
$ docker run test2 ping 1.1.1.1
BusyBox v1.31.1 () multi-call binary.

Usage: ping [OPTIONS] HOST

Send ICMP ECHO_REQUEST packets to network hosts

	-4,-6		Force IP or IPv6 name resolution
	-c CNT		Send only CNT pings
	-s SIZE		Send SIZE data bytes in packets (default 56)
	-i SECS		Interval
	-A		Ping as soon as reply is recevied
	-t TTL		Set TTL
	-I IFACE/IP	Source interface or IP address
	-W SEC		Seconds to wait for the first response (default 10)
			(after all -c CNT packets are sent)
	-w SEC		Seconds until ping exits (default:infinite)
			(can exit earlier with -c CNT)
	-q		Quiet, only display output at start
			and when finished
	-p HEXBYTE	Pattern to use for payload
```
Как видим, аргументы `docker run` не заменяют ENTRYPOINT, а дописываются к нему (выполняется `ping 8.8.8.8 ping 1.1.1.1`), поэтому ping завершается ошибкой. Заменить сам ENTRYPOINT можно только флагом `docker run --entrypoint`.

**Пример 3. ENTRYPOINT и CMD:**
Опишем сборку образа в Dockerfile.
```
FROM alpine  
ENTRYPOINT ["ping"]
CMD ["8.8.8.8"]
```
В инструкцию ENTRYPOINT передаётся аргумент `ping`, в CMD передаётся аргумент 8.8.8.8. Выполним сборку образа `docker build -t test .` и запустим контейнер.
```
$ docker run test3
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=43 time=41.176 ms
64 bytes from 8.8.8.8: seq=1 ttl=43 time=32.875 ms
64 bytes from 8.8.8.8: seq=2 ttl=43 time=40.395 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 32.875/38.148/41.176 ms
```
Пробуем изменить 2 аргумента.
```
$ docker run test3 traceroute 1.1.1.1
BusyBox v1.31.1 () multi-call binary.

Usage: ping [OPTIONS] HOST

Send ICMP ECHO_REQUEST packets to network hosts

	-4,-6		Force IP or IPv6 name resolution
	-c CNT		Send only CNT pings
	-s SIZE		Send SIZE data bytes in packets (default 56)
	-i SECS		Interval
	-A		Ping as soon as reply is recevied
	-t TTL		Set TTL
	-I IFACE/IP	Source interface or IP address
	-W SEC		Seconds to wait for the first response (default 10)
			(after all -c CNT packets are sent)
	-w SEC		Seconds until ping exits (default:infinite)
			(can exit earlier with -c CNT)
	-q		Quiet, only display output at start
			and when finished
	-p HEXBYTE	Pattern to use for payload
```
Изменить 2 аргумента невозможно. Заменим аргумент инструкции CMD.
```
$ docker run test3 1.1.1.1    
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: seq=0 ttl=58 time=31.412 ms
64 bytes from 1.1.1.1: seq=1 ttl=58 time=19.400 ms
64 bytes from 1.1.1.1: seq=2 ttl=58 time=15.814 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 15.814/22.208/31.412 ms
```
При такой сборке образа CMD передаётся как аргументы по умолчанию в ENTRYPOINT (выполняется одна команда `ping 8.8.8.8`), а аргументы `docker run` заменяют только CMD.

</details>

4. Чем отличается *COPY* от *ADD* в Dockerfile?

<details>
  <summary>Ответ</summary>

Инструкция *COPY* копируют файлы и директории с хостовой машины внутрь контейнера, инструкция *ADD* копирует файлы и директории с хостовой машины внутрь контейнера и может распаковывать .tar архивы.

</details>

5. Какие есть best practices для написания Dockerfile?

<details>
  <summary>Ответ</summary>

1. Запускать только один процесс на контейнер.
2. Стараться объединять несколько команд RUN в одну для уменьшения количества слоёв образа.
3. Частоизменяемые слои образа необходимо располагать ниже по уровню, чтобы ускорить процесс сборки, т.к. при изменении верхнего слоя, все нижеследующие слои будут пересобираться.
4. Указывать явные версии образов в инструкции FROM, чтобы избежать случая, когда выйдет новая версия образа с тегом latest.
5. При установке пакетов указывать версии пакетов.
6. Очищать кеш пакетного менеджера и удалять ненужные файлы после выполненной инструкции.
7. Использовать multistage build для сборки артифакта в одном контейнере и размещении его в другом.

</details>

6. Какие типы сетевых драйверов используются в docker?

<details>
  <summary>Ответ</summary>

Основные драйвера сетей docker: bridge, host, overlay, ipvlan, macvlan, none

**bridge:** это сетевой драйвер по умолчанию. Бридж сеть используется, когда ваши приложения запускаются в автономных контейнерах, которые должны взаимодействовать между собой. 
![docker-bridge](imgs/docker-bridge.png)
Взаимодействие с хостом выполняется через мост docker0 и конфигурацию таблицы iptables nat. В этом режиме будет выделено сетевое пространство имен, задан IP-адрес для каждого контейнера, а контейнер Docker на хосте будет подключен к виртуальному мосту. Виртуальный мост работает как физический коммутатор, поэтому все контейнеры на хосте подключены к сети уровня 2 через коммутатор.

**host:** использует сеть хоста напрямую без изоляции контейнера и хоста.

**none:** этот режим помещает контейнер в свой собственный сетевой стек, но не выполняет никакой настройки. Фактически, этот режим отключает сетевую функцию контейнера, что полезно в следующих двух ситуациях: контейнер не требует сети (например, только для пакетной задачи записи дисковых томов).

**macvlan:** в режиме Macvlan Bridge каждый контейнер имеет уникальный MAC-адрес, который используется для отслеживания сопоставления MAC-адреса с портом хоста Docker. Сеть драйвера Macvlan подключается к родительскому интерфейсу хоста Docker. Примерами являются физические интерфейсы, такие как eth0, субинтерфейс eth0.10 для тегирования VLAN 802.1q (.10 означает VLAN 10) или даже связанный хост-адаптер, который объединяет два интерфейса Ethernet в единый логический интерфейс. Назначенный шлюз является внешним по отношению к хосту, предоставляемому сетевой инфраструктурой. Каждая сеть Docker в режиме Macvlan Bridge изолирована друг от друга, и только одна сеть может быть подключена к родительскому узлу одновременно. Каждый хост-адаптер имеет теоретический предел, и каждый хост-адаптер может подключаться к сети Docker. Любой контейнер в той же подсети может взаимодействовать с любым другим контейнером в той же сети без шлюзового моста macvlan. Та же сетевая команда docker применяется к драйверу vlan. В режиме Macvlan без внешней маршрутизации процессов между двумя сетями / подсетями контейнеры в разных сетях не могут получить доступ друг к другу. Это также относится к нескольким подсетям в одной и той же терминальной сети.

**overlay:** Оверлейные сети соединяют несколько демонов Docker вместе и позволяют сервисам swarm взаимодействовать друг с другом. Вы также можете использовать оверлейные сети для облегчения связи между сервисом swarm и автономным контейнером или между двумя автономными контейнерами в разных демонах Docker. Эта стратегия устраняет необходимость выполнять маршрутизацию между этими контейнерами на уровне ОС.

**ipvlan:** Сети ipvlan предоставляют пользователям полный контроль над адресацией IPv4 и IPv6. Драйвер VLAN построен на основе этой возможности, предоставляя операторам полный контроль над тегированием VLAN уровня 2 и даже маршрутизацией IPvlan L3 для пользователей.

</details>

7. Что такое эфемерные контейнеры?

<details>
  <summary>Ответ</summary>

[Эфемерные контейнеры](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) стали бета-функцией в Kubernetes v1.23 и теперь включены по умолчанию.
Эфемерные контейнеры предназначены для транзитных задач, когда вам нужно временно [подключить дополнительный контейнер к существующему поду](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container). Это идеально подходит для отладочных операций, когда вы хотите проверить поды, не затрагивая живые экземпляры контейнеров.

</details>

8. Какие механизмы ядра Linux делают контейнер контейнером? Чем namespaces отличаются от cgroups?

<details>
  <summary>Ответ</summary>

Контейнер - это обычный процесс хоста, которому ядро ограничило "видимость" и ресурсы:
- **namespaces** - изоляция того, *что процесс видит*: `pid` (своё дерево процессов, PID 1), `net` (свои интерфейсы, маршруты, iptables), `mnt` (свои точки монтирования), `uts` (hostname), `ipc`, `user` (свои UID/GID), `cgroup`, `time`.
- **cgroups** (в современных дистрибутивах - cgroups v2, единая иерархия в `/sys/fs/cgroup`) - ограничение и учёт того, *сколько процесс потребляет*: CPU, память, I/O, число процессов.
- Дополнительно: **capabilities** (урезанные права root), **seccomp** (фильтр системных вызовов), **AppArmor/SELinux**, **overlayfs** (слоистая файловая система образа).

Проверить: `docker inspect -f '{{.State.Pid}}' app`, затем `ls -l /proc/<PID>/ns` и `cat /proc/<PID>/cgroup`, либо `lsns -p <PID>`.

</details>

9. Как устроена файловая система контейнера (overlay2) и что такое copy-on-write?

<details>
  <summary>Ответ</summary>

Драйвер `overlay2` (в новых установках Docker Engine - snapshotter `overlayfs` в containerd image store, механизм тот же) объединяет каталоги в одно дерево:
- `lowerdir` - слои образа, только чтение, общие для всех контейнеров из этого образа;
- `upperdir` - тонкий слой контейнера для записи;
- `merged` - то, что видит процесс.

**Copy-on-write**: при первом изменении файла из нижнего слоя он целиком копируется в `upperdir` (copy-up). Удаление файла из нижнего слоя создаёт в верхнем "whiteout"-маркер - место на диске не освобождается. Поэтому `RUN rm` в отдельном слое не уменьшает образ, а базы данных и активно пишущие приложения должны писать в volume, а не в слой контейнера.

Посмотреть размер записываемого слоя: `docker ps -s`.

</details>

10. Чем контейнер отличается от виртуальной машины? Когда что выбрать?

<details>
  <summary>Ответ</summary>

- **ВМ** эмулирует железо через гипервизор, у каждой ВМ своё ядро и полноценная ОС. Изоляция сильнее, но выше накладные расходы, старт - десятки секунд, образы - гигабайты.
- **Контейнер** - изолированный процесс, использующий **общее ядро хоста**. Старт - доли секунды, образ - мегабайты, плотность размещения выше.

Следствия: в Linux-контейнере нельзя запустить ядро другой версии или Windows; уязвимость ядра затрагивает все контейнеры на хосте. Для недоверенного кода (multi-tenant) используют ВМ или "песочницы" - gVisor, Kata Containers, Firecracker microVM. На macOS/Windows Docker Desktop запускает Linux-контейнеры внутри небольшой ВМ.

</details>

11. Как работает кэш сборки образа и как правильно упорядочить инструкции Dockerfile?

<details>
  <summary>Ответ</summary>

Каждая инструкция `RUN`, `COPY`, `ADD` создаёт слой. При повторной сборке слой берётся из кэша, если не изменились сама инструкция, родительский слой и (для `COPY`/`ADD`) контрольные суммы копируемых файлов. Как только один слой инвалидирован, **все последующие пересобираются**.

Поэтому редко меняющееся - выше, часто меняющееся - ниже:
```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                     # кэшируется, пока не изменились зависимости
COPY . .                       # код меняется часто
CMD ["node", "server.js"]
```
Для `RUN` кэш сверяется только по тексту команды: `RUN apt-get update` сам по себе не обновится, поэтому его объединяют с `apt-get install` в одну инструкцию. Сбросить кэш: `docker build --no-cache`. С BuildKit можно кэшировать каталоги пакетных менеджеров: `RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt`.

</details>

12. Что такое multi-stage build и зачем он нужен?

<details>
  <summary>Ответ</summary>

Несколько `FROM` в одном Dockerfile: в первой стадии есть компилятор, SDK, dev-зависимости; в финальную стадию копируется только готовый артефакт. В итоговый образ не попадают инструменты сборки, исходники и секреты промежуточных стадий - образ меньше, поверхность атаки уже.
```dockerfile
FROM golang:1.24 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```
Собрать только определённую стадию (например, для тестов): `docker build --target build .`. BuildKit собирает независимые стадии параллельно и пропускает стадии, не нужные для цели.

</details>

13. Что такое BuildKit и `docker buildx`? Как собрать multi-arch образ?

<details>
  <summary>Ответ</summary>

**BuildKit** - движок сборки, используемый по умолчанию начиная с Docker Engine 23.0: параллельная сборка стадий, `RUN --mount` (cache, secret, ssh), внешний кэш (`--cache-to/--cache-from`, например в registry), аттестации (SBOM, provenance).

**buildx** - CLI-плагин для BuildKit, умеющий собирать под несколько платформ и работать с удалёнными билдерами.

Multi-arch образ - это **manifest list (OCI image index)**: под одним тегом лежат манифесты для разных архитектур, `docker pull` сам выбирает подходящий.
```sh
docker buildx create --use --name multi
docker buildx build --platform linux/amd64,linux/arm64 -t registry.example.com/app:1.4.0 --push .
docker buildx imagetools inspect registry.example.com/app:1.4.0
```
Чужие архитектуры собираются через эмуляцию QEMU (binfmt) - медленно; быстрее нативные ноды-билдеры или кросс-компиляция (`--platform=$BUILDPLATFORM` + `TARGETOS/TARGETARCH`).

</details>

14. Как передать секрет (токен, ключ) при сборке образа, чтобы он не остался в образе?

<details>
  <summary>Ответ</summary>

`ARG` и `ENV` не подходят: их значения видны в `docker history` и метаданных образа. Удаление файла в следующем `RUN` тоже не помогает - файл остаётся в предыдущем слое.

Правильно - secret mount BuildKit: секрет монтируется только на время одной инструкции и не пишется в слой.
```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```sh
docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
```
Для приватных git-репозиториев - `RUN --mount=type=ssh` и `docker build --ssh default .`. Секреты времени выполнения передают при запуске (переменные окружения, файлы, Docker/Kubernetes secrets, Vault), а не зашивают в образ.

</details>

15. Что такое build context и зачем нужен `.dockerignore`?

<details>
  <summary>Ответ</summary>

Build context - набор файлов (обычно каталог `.` в `docker build .`), который передаётся билдеру. `COPY`/`ADD` могут брать файлы только из контекста.

`.dockerignore` исключает файлы из контекста (синтаксис похож на `.gitignore`):
```
.git
node_modules
*.log
.env
**/__pycache__
```
Зачем: 
- быстрее сборка (меньше передаётся данных);
- `COPY . .` не утащит в образ `.git`, локальные `.env`, ключи;
- кэш не инвалидируется из-за изменения нерелевантных файлов.

</details>

16. Чем отличается shell form от exec form в `CMD`/`ENTRYPOINT`? Как это связано с PID 1 и сигналами?

<details>
  <summary>Ответ</summary>

- **exec form** `CMD ["nginx", "-g", "daemon off;"]` - процесс запускается напрямую и становится PID 1.
- **shell form** `CMD nginx -g 'daemon off;'` - запускается `/bin/sh -c "..."`, PID 1 становится shell.

`docker stop` шлёт SIGTERM процессу PID 1, ждёт 10 секунд (`docker stop -t`, `STOPSIGNAL`) и шлёт SIGKILL. В shell form SIGTERM получает `sh`, который обычно не пересылает его приложению - контейнер останавливается долго и "грязно".

Особенности PID 1: ядро не применяет к нему действия сигналов по умолчанию (если приложение не установило обработчик SIGTERM, оно его просто проигнорирует) и он должен забирать зомби-процессы. Решения: exec form; `exec "$@"` в конце entrypoint-скрипта; лёгкий init - `docker run --init` (встроенный tini) или `tini`/`dumb-init` в `ENTRYPOINT`.

</details>

17. Контейнер завершился с кодом 137 (или 143, 139). Что это значит и как найти причину?

<details>
  <summary>Ответ</summary>

Коды > 128 означают завершение сигналом: `код - 128 = номер сигнала`.
- **137** = SIGKILL (9): OOM killer из-за лимита памяти, либо `docker kill`/истёк таймаут `docker stop`.
- **143** = SIGTERM (15): штатная остановка.
- **139** = SIGSEGV (11): падение приложения.
- **125/126/127** - ошибка самого `docker run`, команда не исполняемая, команда не найдена.

Диагностика:
```sh
docker inspect -f '{{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}' app
docker logs --tail 100 app
journalctl -k | grep -i oom        # сообщения OOM killer в ядре
```

</details>

18. Что такое distroless и минимальные образы? Чем `alpine` отличается от `distroless` и `scratch`?

<details>
  <summary>Ответ</summary>

- **scratch** - пустой образ. Подходит для статически собранных бинарников (Go с `CGO_ENABLED=0`, Rust musl). Нет даже CA-сертификатов и `/etc/passwd` - их нужно скопировать самим.
- **distroless** (например, `gcr.io/distroless/static`, `base`, `java`, `python`) - только runtime-зависимости, без shell и пакетного менеджера; есть вариант `:nonroot`. Меньше CVE и меньше инструментов для атакующего.
- **alpine** - полноценный маленький дистрибутив (~5-8 МБ) с `apk` и `sh`, но на **musl libc**: возможны отличия в DNS-резолвинге, производительности и совместимости с бинарниками под glibc.
- Также есть `debian:*-slim`, `ubi-minimal`, hardened-образы (Chainguard/Wolfi, Docker Hardened Images).

Минус образов без shell - сложнее отладка (см. вопрос про отладку контейнера без shell); у distroless для этого есть теги `:debug` с busybox.

</details>

19. Образ весит несколько гигабайт. Как понять, почему, и как его уменьшить?

<details>
  <summary>Ответ</summary>

Анализ:
```sh
docker image ls app
docker history --no-trunc app:latest   # размер каждого слоя и создавшая его инструкция
dive app:latest                         # интерактивный просмотр содержимого слоёв
```
Типичные причины и решения:
1. Толстый базовый образ -> `*-slim`, `alpine`, distroless.
2. Инструменты сборки в финальном образе -> multi-stage build.
3. Кэш пакетного менеджера -> `apt-get install --no-install-recommends ... && rm -rf /var/lib/apt/lists/*` в том же `RUN`, `pip install --no-cache-dir`, `npm ci --omit=dev`.
4. Файлы удаляются в отдельном слое -> удалять в той же инструкции, где они появились.
5. В контекст попали `.git`, `node_modules`, дампы -> `.dockerignore`.
6. `COPY` + `chown` отдельным `RUN` дублирует файлы -> `COPY --chown=app:app`.

</details>

20. Почему не стоит использовать тег `latest`? Что такое digest образа?

<details>
  <summary>Ответ</summary>

Тег - изменяемый указатель: `latest` (как и `1.27`) сегодня и завтра может указывать на разные образы. `latest` - это просто тег по умолчанию, а не "самая свежая версия". Итог - невоспроизводимые сборки и разные версии на разных нодах.

**Digest** (`sha256:...`) - хеш манифеста образа, неизменяемый идентификатор содержимого:
```sh
docker pull nginx@sha256:<digest>
docker image ls --digests nginx
docker buildx imagetools inspect nginx:1.27
```
Практика: в продакшене использовать конкретные версии, для критичных мест - закреплять по digest (`FROM nginx:1.27@sha256:...`), а обновления автоматизировать (Renovate/Dependabot). В registry включать immutable tags, если поддерживается.

</details>

21. Чем `EXPOSE` отличается от `-p`? Почему опубликованный порт может быть доступен из интернета, несмотря на ufw/firewalld?

<details>
  <summary>Ответ</summary>

- `EXPOSE 8080` - только документация/метаданные образа, порт не публикуется.
- `-p 8081:80` - публикация: Docker добавляет DNAT-правило (iptables/nftables, цепочка `DOCKER`), трафик на порт 8081 хоста уходит на порт 80 контейнера. `-P` публикует все EXPOSE-порты на случайные.

По умолчанию `-p 8081:80` слушает **на всех интерфейсах** (`0.0.0.0`). Правила Docker срабатывают в nat PREROUTING и цепочке FORWARD раньше правил ufw, поэтому ufw такой порт не закрывает. Решения:
```sh
docker run -p 127.0.0.1:8081:80 nginx   # только локально
```
свои правила фильтрации - в цепочку `DOCKER-USER`; для баз данных порт вообще не публиковать, а обращаться к контейнеру по внутренней сети.

</details>

22. Как контейнеры находят друг друга по имени? Чем пользовательская bridge-сеть отличается от сети по умолчанию?

<details>
  <summary>Ответ</summary>

В **пользовательской** bridge-сети работает встроенный DNS Docker (`127.0.0.11` внутри контейнера): контейнеры резолвят друг друга по имени контейнера и по сетевым алиасам. В сети по умолчанию `bridge` (docker0) DNS по именам нет, только IP (устаревший `--link`).
```sh
docker network create app-net
docker run -d --name db --network app-net postgres:17
docker run -d --name api --network app-net -e DB_HOST=db my/api
docker network inspect app-net
```
Пользовательские сети также дают изоляцию: контейнеры из разных сетей не видят друг друга. Docker Compose по умолчанию создаёт такую сеть для проекта, и сервисы доступны по имени сервиса.

</details>

23. В чём разница между volume, bind mount и tmpfs?

<details>
  <summary>Ответ</summary>

- **volume** - управляется Docker (`/var/lib/docker/volumes/...`), переживает удаление контейнера, не зависит от структуры каталогов хоста, поддерживает драйверы (NFS и т.п.). Рекомендуемый способ хранить данные (БД). При подключении пустого volume в каталог образа содержимое каталога копируется в volume.
- **bind mount** - произвольный путь хоста монтируется в контейнер. Удобно для конфигов и разработки, но зависит от хоста, прав и UID, а также может дать контейнеру доступ к чувствительным путям хоста.
- **tmpfs** - в оперативной памяти, исчезает при остановке. Для временных и чувствительных данных.
```sh
docker run -v pgdata:/var/lib/postgresql/data postgres:17
docker run --mount type=bind,src=/etc/app/conf.yml,dst=/app/conf.yml,readonly my/app
docker run --read-only --tmpfs /tmp:rw,size=64m my/app
```

</details>

24. Для чего нужен `HEALTHCHECK` и чем он отличается от проверки "контейнер запущен"?

<details>
  <summary>Ответ</summary>

Статус `running` говорит лишь о том, что процесс жив. `HEALTHCHECK` периодически выполняет команду внутри контейнера и проверяет, что приложение реально работает; статус - `starting`, `healthy`, `unhealthy`.
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -fsS http://localhost:8080/health || exit 1
```
Посмотреть: `docker ps` (колонка STATUS) или `docker inspect -f '{{json .State.Health}}' app`.

Важно: сам Docker Engine **не перезапускает** unhealthy-контейнер (это делает Swarm или внешняя автоматика). Статус используется в `depends_on: condition: service_healthy` в Compose. В Kubernetes `HEALTHCHECK` игнорируется - там свои liveness/readiness/startup probes. Для distroless-образов без curl проверку делают отдельным бинарником.

</details>

25. Как ограничить ресурсы контейнера и что будет при превышении лимитов?

<details>
  <summary>Ответ</summary>

```sh
# память 512 МБ без swap, не более 1.5 CPU (квота CFS), не более 200 процессов (защита от fork-бомбы)
docker run -d --name app --memory 512m --memory-swap 512m --cpus 1.5 --pids-limit 200 my/app
docker stats                           # текущее потребление
docker update --memory 1g --memory-swap 1g app   # изменить на лету
```
Под капотом - cgroups. Превышение памяти -> OOM killer убивает процесс (код 137, `OOMKilled=true`). Превышение CPU -> троттлинг (приложение тормозит, но не падает). `--cpu-shares` - относительный вес, действует только при конкуренции за CPU. Без лимитов один контейнер может исчерпать память всего хоста.

</details>

26. Какие практики безопасности вы применяете при запуске контейнеров?

<details>
  <summary>Ответ</summary>

1. Не запускать процесс от root: `USER 10001` в Dockerfile или `docker run --user`.
2. Минимум capabilities: `--cap-drop ALL --cap-add NET_BIND_SERVICE`.
3. `--security-opt no-new-privileges` - запрет повышения прав через setuid.
4. Не использовать `--privileged` - он даёт почти полный доступ к хосту (все capabilities, устройства, отключает seccomp/AppArmor).
5. Не монтировать `/var/run/docker.sock` - доступ к сокету равен root на хосте.
6. Оставить профиль seccomp по умолчанию, AppArmor/SELinux включены.
7. `--read-only` + `tmpfs` для каталогов записи, лимиты ресурсов.
8. Минимальные базовые образы, сканирование на уязвимости, подпись, секреты не в образе.
9. Rootless-режим или user namespaces.

</details>

27. Что такое rootless Docker и user namespaces (`userns-remap`)? Какие у них ограничения?

<details>
  <summary>Ответ</summary>

Проблема: по умолчанию root в контейнере - это UID 0 на хосте; при побеге из контейнера атакующий получает root.

- **userns-remap** - демон работает от root, но UID 0 контейнера отображается на непривилегированный диапазон из `/etc/subuid`/`/etc/subgid`: `{"userns-remap": "default"}` в `/etc/docker/daemon.json`.
- **rootless Docker** - сам демон и контейнеры работают от обычного пользователя (установка: `dockerd-rootless-setuptool.sh install`), используется user namespace, сеть через RootlessKit (slirp4netns/pasta). Podman работает rootless изначально.

Ограничения: нельзя публиковать порты < 1024 без `net.ipv4.ip_unprivileged_port_start`, ниже производительность сети, нет части функций (некоторые драйверы хранилища, `--privileged` в полном смысле, AppArmor), сложности с правами на bind mount.

</details>

28. Как проверить образ на уязвимости и гарантировать его происхождение (сканирование, SBOM, подпись)?

<details>
  <summary>Ответ</summary>

- **Сканирование CVE**: `trivy image registry/app:1.4.0`, `grype registry/app:1.4.0`, `docker scout cves registry/app:1.4.0`. В CI - падать на уязвимостях HIGH/CRITICAL (`trivy image --exit-code 1 --severity HIGH,CRITICAL ...`), дополнительно сканировать Dockerfile на ошибки конфигурации (`trivy config`, hadolint).
- **SBOM** (Software Bill of Materials) - список всех пакетов в образе в формате SPDX или CycloneDX: `syft registry/app:1.4.0 -o spdx-json`, либо `docker buildx build --sbom=true --provenance=true ...` - аттестации прикрепляются к образу.
- **Подпись** - Sigstore cosign: `cosign sign --key cosign.key registry/app@sha256:...`, проверка `cosign verify --key cosign.pub ...`; возможен keyless-режим через OIDC-идентичность CI. Подписывают digest, а не тег.
- На стороне кластера - admission-политики (Kyverno, Sigstore policy-controller), разрешающие только подписанные образы из доверенных registry.

</details>

29. Что такое OCI? Как соотносятся Docker, containerd, runc и Podman?

<details>
  <summary>Ответ</summary>

**OCI** (Open Container Initiative) - стандарты: image spec (формат образа), runtime spec (как запускать контейнер из bundle), distribution spec (API registry). Благодаря им образ, собранный Docker, запускается в Podman, containerd, Kubernetes.

Цепочка Docker: `docker` CLI -> `dockerd` (API, сети, volumes, сборка) -> **containerd** (high-level runtime: образы, snapshot'ы, жизненный цикл) -> `containerd-shim` -> **runc** (low-level OCI runtime: создаёт namespaces/cgroups и запускает процесс, после чего завершается). Благодаря shim контейнеры могут продолжать работать при перезапуске демона (`"live-restore": true`).

**Podman** - альтернатива без центрального демона (daemonless), по умолчанию rootless, CLI совместим с docker, умеет pod'ы и интеграцию с systemd (Quadlet). Kubernetes с версии 1.24 работает с runtime напрямую через CRI (containerd, CRI-O), без dockershim - но образы, собранные Docker, работают, так как это OCI-образы.

</details>

30. Что нужно знать про Docker Compose v2? Как дождаться готовности зависимого сервиса?

<details>
  <summary>Ответ</summary>

Compose v2 - плагин CLI на Go, команда `docker compose` (через пробел); старый Python `docker-compose` v1 больше не поддерживается. Поле `version:` в `compose.yaml` устарело и игнорируется.

Простой `depends_on` задаёт только порядок старта, не готовность. Для ожидания используется `condition`:
```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
  migrate:
    image: my/app
    command: ["./migrate"]
    depends_on:
      db: { condition: service_healthy }
  api:
    image: my/app
    depends_on:
      migrate: { condition: service_completed_successfully }
```
Полезное: `docker compose up -d --wait`, `docker compose config` (итоговый конфиг), `profiles`, `docker compose watch` для разработки. Приложение всё равно должно уметь переподключаться к БД.

</details>

31. Как отлаживать контейнер, в образе которого нет shell и утилит (distroless, scratch)?

<details>
  <summary>Ответ</summary>

1. Запустить отладочный контейнер в namespaces целевого:
```sh
docker run --rm -it --pid=container:app --network=container:app nicolaka/netshoot
# процессы app видны в ps, файловая система app - через /proc/<pid>/root
```
2. С хоста через `nsenter`:
```sh
PID=$(docker inspect -f '{{.State.Pid}}' app)
nsenter -t "$PID" -n ss -tlnp      # сетевой namespace контейнера с утилитами хоста
```
3. `docker cp app:/path/file .` - забрать файлы; `docker logs`, `docker inspect`, `docker events`.
4. `docker debug app` в Docker Desktop - подключает shell с инструментами без изменения образа.
5. Временно собрать образ с `:debug`-вариантом базового образа.

В Kubernetes аналог - `kubectl debug` с эфемерным контейнером.

</details>

32. На сервере закончилось место из-за Docker. Как найти причину и что настроить, чтобы не повторилось?

<details>
  <summary>Ответ</summary>

```sh
docker system df -v                 # образы, контейнеры, volumes, build cache
du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail   # логи
docker image prune -a               # неиспользуемые образы
docker builder prune                # кэш сборки
docker container prune              # остановленные контейнеры
```
`docker system prune -a --volumes` удалит и неиспользуемые volumes (данные!) - применять осознанно.

Частая причина - логи: драйвер `json-file` по умолчанию **без ротации**. Настроить в `/etc/docker/daemon.json`:
```json
{ "log-driver": "local", "log-opts": { "max-size": "10m", "max-file": "3" } }
```
Или `json-file` с теми же `log-opts`. Применяется только к **новым** контейнерам (существующие нужно пересоздать). Также - мониторинг заполнения диска и вынос `/var/lib/docker` на отдельный раздел (`data-root`).

</details>