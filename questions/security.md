## Безопасность (DevSecOps)

### Общие принципы

1. Что такое DevSecOps и принцип наименьших привилегий? Как его применять на практике?

<details>
  <summary>Ответ</summary>

**DevSecOps** — встраивание безопасности во все этапы жизненного цикла (shift left): проверки в CI (SAST, SCA, сканирование образов и IaC, поиск секретов), политики при деплое, runtime-защита и мониторинг в проде. Безопасность становится общей ответственностью и автоматизируется, а не проверяется вручную перед релизом.

**Принцип наименьших привилегий (least privilege)** — каждый пользователь, сервис и процесс получает ровно те права, которые нужны для задачи, и только на нужное время. Примеры:
- сервисы работают не от root, а от отдельного системного пользователя;
- `sudo` только на конкретные команды (`/etc/sudoers.d/`), без `ALL`;
- в Kubernetes — отдельный ServiceAccount на приложение, Role вместо ClusterRole, `automountServiceAccountToken: false`, если API не нужен;
- в облаке — IAM-политики на конкретные ресурсы (`arn:...:bucket/prefix/*`), а не `*:*`;
- временные доступы (JIT) вместо постоянных, регулярный пересмотр прав.

</details>

2. Что такое модель Zero Trust? Как она реализуется технически?

<details>
  <summary>Ответ</summary>

Zero Trust отказывается от идеи «доверенной внутренней сети»: нахождение внутри периметра (VPN, офис, VPC) само по себе не даёт доступа. Каждый запрос аутентифицируется, авторизуется и шифруется, решение принимается на основе идентичности пользователя/сервиса, состояния устройства и контекста.

Технически:
- **mTLS между сервисами** с короткоживущими сертификатами, выданными по идентичности сервиса (service mesh Istio/Linkerd, SPIFFE/SPIRE);
- **identity-aware proxy / ZTNA** для людей вместо плоского VPN (Cloudflare Access, Google IAP, Teleport, Pomerium) с SSO и MFA;
- микросегментация: NetworkPolicy, security groups по принципу «запрещено всё, что не разрешено»;
- короткоживущие учётные данные (OIDC, SSH-сертификаты, динамические секреты Vault);
- непрерывный аудит и мониторинг доступа.

</details>

3. Кратко перечислите OWASP Top 10. Что из этого важно DevOps-инженеру?

<details>
  <summary>Ответ</summary>

OWASP Top 10 — рейтинг самых распространённых классов уязвимостей веб-приложений. Версия 2021:
1. Broken Access Control; 2. Cryptographic Failures; 3. Injection (SQL, команды, XSS); 4. Insecure Design; 5. Security Misconfiguration; 6. Vulnerable and Outdated Components; 7. Identification and Authentication Failures; 8. Software and Data Integrity Failures; 9. Security Logging and Monitoring Failures; 10. Server-Side Request Forgery (SSRF).

В редакции 2025 года выделена отдельная категория Software Supply Chain Failures, а SSRF вошёл в Broken Access Control.

Зона DevOps: мисконфигурации (открытые бакеты и админки, дефолтные пароли, debug в проде), устаревшие компоненты (сканирование и обновление зависимостей и базовых образов), целостность пайплайна и артефактов (подпись, защита CI), логирование и алертинг по событиям безопасности, защита от SSRF (например, IMDSv2 в AWS, запрет доступа к `169.254.169.254` из подов через NetworkPolicy).

</details>

### Секреты

4. Почему секреты нельзя хранить в git и зашивать в образ или переменные окружения образа? Какие есть варианты хранения?

<details>
  <summary>Ответ</summary>

- **git**: секрет остаётся в истории навсегда, распространяется со всеми клонами, форками и CI-кэшами; удаление в новом коммите не помогает. Боты сканируют публичные репозитории на ключи за минуты.
- **Образ** (`COPY .env`, `ENV DB_PASSWORD=...`, `ARG`): попадает в слои и метаданные, виден через `docker history` / `docker inspect` каждому, у кого есть доступ к registry. Удаление файла в следующем слое не удаляет его из предыдущего.
- **Переменные окружения** в рантайме лучше, но видны в `/proc/<pid>/environ`, `docker inspect`, `kubectl describe`, наследуются дочерними процессами и часто попадают в логи и дампы при ошибках. Файл в tmpfs с ограниченными правами предпочтительнее.

Варианты: HashiCorp Vault / OpenBao, облачные менеджеры (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault), Kubernetes Secrets + External Secrets Operator, SOPS / Sealed Secrets для GitOps. Для сборки — BuildKit secrets: `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci` и `docker build --secret id=npmrc,src=$HOME/.npmrc .`.

</details>

5. Как работает HashiCorp Vault? Что такое динамические секреты?

<details>
  <summary>Ответ</summary>

Vault — централизованное хранилище секретов с аутентификацией, политиками доступа и аудитом. Данные зашифрованы; после старта Vault находится в состоянии sealed, и для работы его нужно распечатать (unseal) ключами Шамира или через auto-unseal облачным KMS/HSM.

- **Auth methods** — как клиент доказывает, кто он: `kubernetes` (по токену ServiceAccount), `approle`, `jwt/oidc` (в т.ч. для GitHub Actions/GitLab CI), `aws`, `ldap`.
- **Policies** — HCL-правила, какие пути доступны (`path "secret/data/app/*" { capabilities = ["read"] }`).
- **Secrets engines**: `kv` (статические), `database`, `aws`, `pki`, `transit` (шифрование как сервис).
- **Динамические секреты** — Vault создаёт учётку в БД/облаке по запросу с ограниченным TTL (lease) и сам удаляет её по истечении. Утёкший пароль быстро перестаёт работать, у каждого клиента своя учётка — видно, кто что делал.

```bash
vault write auth/kubernetes/role/app \
  bound_service_account_names=app bound_service_account_namespaces=prod \
  policies=app ttl=1h
vault read database/creds/app-readonly    # выдаст временные логин/пароль
```
В Kubernetes секреты доставляют через Vault Agent Injector (sidecar пишет файлы в под), Vault Secrets Operator или External Secrets Operator.

</details>

6. Как хранить секреты при GitOps? Что такое SOPS, Sealed Secrets и External Secrets Operator?

<details>
  <summary>Ответ</summary>

- **SOPS** — шифрует значения в YAML/JSON/ENV-файлах (ключи остаются читаемыми, diff осмысленный) через age, PGP или облачный KMS. Зашифрованный файл можно коммитить; расшифровка в CI или в кластере (Flux умеет нативно, для Argo CD — плагины/helm-secrets).
  ```yaml
  # .sops.yaml
  creation_rules:
    - path_regex: .*/prod/.*\.yaml
      encrypted_regex: ^(data|stringData)$
      age: age1qyxz...   # публичный ключ
  ```
  ```bash
  sops -e -i k8s/prod/secret.yaml
  sops -d k8s/prod/secret.yaml
  ```
- **Sealed Secrets** (Bitnami) — `kubeseal` шифрует Secret публичным ключом контроллера в кластере; в git лежит `SealedSecret`, расшифровать его может только контроллер.
- **External Secrets Operator** — в git лежит только ссылка (`ExternalSecret` → `SecretStore`), а само значение оператор забирает из Vault/AWS Secrets Manager/GCP и создаёт обычный Secret, периодически синхронизируя его. Хорошо подходит для ротации, т.к. источник правды — внешний менеджер.

</details>

7. Насколько безопасны Secret'ы в Kubernetes? Как их защитить?

<details>
  <summary>Ответ</summary>

По умолчанию Secret — это просто base64 (кодирование, не шифрование), и в etcd он хранится открыто. Кто имеет `get/list secrets` в namespace или права создать под с монтированием любого Secret — фактически читает все секреты namespace.

Защита:
- **шифрование at rest** в etcd: `--encryption-provider-config` на kube-apiserver, лучше через провайдер `kms` (v2) с ключом во внешнем KMS:
  ```yaml
  apiVersion: apiserver.config.k8s.io/v1
  kind: EncryptionConfiguration
  resources:
    - resources: ["secrets"]
      providers:
        - aescbc:
            keys:
              - name: key1
                secret: <base64 32 байта>
        - identity: {}
  ```
  после включения перешифровать существующие: `kubectl get secrets -A -o json | kubectl replace -f -`;
- строгий RBAC на `secrets` (особенно `list`/`watch`), аудит доступа;
- монтирование как файлов (tmpfs), а не через env;
- шифрование бэкапов etcd и ограничение доступа к узлам control plane;
- внешний источник правды (Vault/ESO) и регулярная ротация.

</details>

8. Что делать, если секрет попал в git-репозиторий?

<details>
  <summary>Ответ</summary>

Порядок важен — сначала обезвредить, потом чистить:
1. **Сразу отозвать/ротировать секрет** (выпустить новый ключ, сменить пароль, отозвать токен). Считать его скомпрометированным, даже если репозиторий приватный и коммит «провисел минуту».
2. **Проверить использование**: аудит-логи облака (CloudTrail), провайдера, БД — не было ли действий с этим ключом.
3. **Очистить историю**, если это требуется политикой:
   ```bash
   git filter-repo --invert-paths --path config/.env          # удалить файл из всей истории
   git filter-repo --replace-text replacements.txt            # или заменить строки
   git push --force --all && git push --force --tags
   ```
   (альтернатива — BFG Repo-Cleaner). Все участники должны переклонировать репозиторий. На GitHub/GitLab могут оставаться кэш, форки и ссылки из PR — для их удаления обращаются в поддержку.
4. **Предотвращение**: pre-commit хуки и CI со сканерами секретов (gitleaks, trufflehog), GitHub push protection, `.gitignore` для `.env`, секреты — только через менеджер секретов.

</details>

### TLS, PKI и SSH

9. Как устроена цепочка доверия TLS? Как проверить сертификат сервера из консоли?

<details>
  <summary>Ответ</summary>

Сертификат сервера подписан промежуточным CA, тот — корневым CA, который есть в trust store клиента. Сервер должен отдавать свой сертификат **вместе с промежуточными** (fullchain); иначе часть клиентов (curl, Java, мобильные) не сможет построить цепочку. Клиент проверяет подписи, срок действия, совпадение имени с SAN (CN давно не используется для проверки имени), статус отзыва (OCSP/CRL).

```bash
# что отдаёт сервер (с SNI), цепочка и результат проверки
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
# сроки и SAN
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
# истечёт ли в ближайшие 30 дней (код возврата 1 — истечёт)
openssl x509 -in cert.pem -noout -checkend 2592000
# соответствует ли ключ сертификату
openssl x509 -in cert.pem -noout -pubkey | sha256sum; openssl pkey -in key.pem -pubout | sha256sum
```

</details>

10. Как автоматизировать выпуск и ротацию сертификатов? Что такое cert-manager?

<details>
  <summary>Ответ</summary>

Ручная ротация — частая причина аварий «сертификат протух». Кроме того, срок жизни публичных сертификатов по решению CA/Browser Forum поэтапно сокращается (до 47 дней к 2029 году), так что автоматизация обязательна.

- На серверах — ACME-клиенты: `certbot`, `acme.sh`, встроенный ACME в Caddy/Traefik; таймер на продление + reload веб-сервера.
- В Kubernetes — **cert-manager**: CRD `Issuer`/`ClusterIssuer` (Let's Encrypt через ACME HTTP-01 или DNS-01, Vault PKI, собственный CA) и `Certificate`. Сертификат кладётся в Secret и продлевается автоматически (по умолчанию за 1/3 срока до истечения). Для Ingress достаточно аннотации `cert-manager.io/cluster-issuer: letsencrypt-prod` и секции `tls`.
- Внутренний PKI: Vault PKI, step-ca, короткоживущие сертификаты вместо отзыва.

Обязательно мониторить сроки: `probe_ssl_earliest_cert_expiry` из blackbox_exporter, метрика `certmanager_certificate_expiration_timestamp_seconds`.

</details>

11. Как защитить SSH-доступ к серверам?

<details>
  <summary>Ответ</summary>

`/etc/ssh/sshd_config` (или файл в `/etc/ssh/sshd_config.d/`):
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups ssh-users
MaxAuthTries 3
X11Forwarding no
```
Проверить конфиг перед перезапуском: `sshd -t`, и не закрывать текущую сессию, пока не проверен вход в новой.

Дополнительно: ключи ed25519 с passphrase и ssh-agent (или ключи на аппаратных токенах `ed25519-sk`), доступ к порту 22 только из сети bastion/VPN, fail2ban, централизованное управление ключами (не копировать `authorized_keys` вручную), логирование сессий.

</details>

12. Что такое bastion (jump host) и SSH-сертификаты (SSH CA)?

<details>
  <summary>Ответ</summary>

**Bastion** — единственная точка входа во внутреннюю сеть по SSH; сами серверы доступны только с него. На нём централизуются аудит и MFA. Подключение через него без копирования ключей на bastion:
```bash
ssh -J bastion.example.com user@10.0.1.15
ssh -J jump1,jump2,jump3 user@target        # цепочка из нескольких jump-хостов
```
```
# ~/.ssh/config
Host 10.0.*
    ProxyJump bastion.example.com
```
Не стоит использовать `ForwardAgent` через bastion: root на нём сможет воспользоваться вашим агентом.

**SSH CA** — вместо раскладки публичных ключей по `authorized_keys` сервер доверяет центру сертификации, а пользователю выдаётся короткоживущий сертификат:
```bash
ssh-keygen -s user_ca -I alice@corp -n alice,deploy -V +8h ~/.ssh/id_ed25519.pub
# на серверах: TrustedUserCAKeys /etc/ssh/user_ca.pub
```
Плюсы: доступ истекает сам, не нужно чистить ключи уволенных, в сертификате есть принципалы и ограничения. Аналогично хост-сертификаты (`ssh-keygen -h`, в known_hosts `@cert-authority *.example.com ...`) убирают TOFU-вопрос о fingerprint. Готовые решения: Vault SSH secrets engine, Teleport, step-ca.

</details>

### Hardening Linux

13. Как вы проводите hardening Linux-сервера?

<details>
  <summary>Ответ</summary>

- Базовые требования берут из **CIS Benchmarks** (или STIG) для конкретного дистрибутива; проверка — OpenSCAP (`oscap`), Lynis (`lynis audit system`), автоматизация — Ansible-роли (например, dev-sec `ansible-collection-hardening`).
- Минимальная установка, удаление/отключение лишних сервисов: `systemctl list-unit-files --state=enabled`, `ss -tulpn` — что слушает порты.
- Автоматические обновления безопасности (`unattended-upgrades`, `dnf-automatic`), быстрая реакция на критичные CVE.
- SSH hardening, sudo с минимальными правами, отключённые пароли, MFA на bastion.
- Firewall с политикой default deny.
- Параметры ядра через sysctl: `net.ipv4.conf.all.rp_filter=1`, `net.ipv4.conf.all.accept_redirects=0`, `kernel.kptr_restrict=2`, `kernel.dmesg_restrict=1`, `fs.protected_symlinks=1`.
- Опции монтирования `nodev,nosuid,noexec` для `/tmp`, `/dev/shm`.
- SELinux/AppArmor в режиме enforcing, auditd, отправка логов на внешний сервер (чтобы атакующий не мог их затереть), контроль целостности (AIDE).

</details>

14. Что такое fail2ban и auditd? Для чего их используют?

<details>
  <summary>Ответ</summary>

**fail2ban** читает логи (journald, `/var/log/auth.log`), находит по regex повторные неудачные попытки входа и временно банит IP через iptables/nftables.
```ini
# /etc/fail2ban/jail.local
[sshd]
enabled  = true
maxretry = 5
findtime = 10m
bantime  = 1h
```
`fail2ban-client status sshd` — забаненные адреса, `fail2ban-client set sshd unbanip 1.2.3.4` — разбан. Он снижает шум от брутфорса, но не заменяет отключение паролей.

**auditd** — подсистема аудита ядра: журналирует системные вызовы и доступ к файлам по правилам (кто изменил `/etc/passwd`, кто запускал `sudo`).
```
# /etc/audit/rules.d/hardening.rules
-w /etc/passwd -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_exec
```
Поиск: `ausearch -k identity -i`, отчёты: `aureport --auth`. Загрузить правила: `augenrules --load`.

</details>

15. Как настроить firewall на сервере по принципу default deny?

<details>
  <summary>Ответ</summary>

Разрешаем только нужное, остальное отбрасываем. Пример на nftables (`/etc/nftables.conf`):
```
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    ct state invalid drop
    iif "lo" accept
    meta l4proto { icmp, ipv6-icmp } accept
    tcp dport 22 ip saddr 10.0.0.0/24 accept
    tcp dport { 80, 443 } accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output priority 0; policy accept; }
}
```
`nft -c -f /etc/nftables.conf` — проверка синтаксиса, `nft list ruleset` — текущие правила. Обёртки: `ufw` (Ubuntu), `firewalld` с зонами (RHEL). При удалённой настройке — страховка от потери доступа (например, `at now + 5 min` с откатом правил).

Нюанс: Docker сам добавляет правила iptables для опубликованных портов в обход INPUT, поэтому для контейнеров фильтрацию делают в цепочке `DOCKER-USER` или публикуют порты только на `127.0.0.1`.

</details>

16. Что такое SELinux и AppArmor? Чем они отличаются? Можно ли отключить SELinux на лету?

<details>
  <summary>Ответ</summary>

Оба — реализации **MAC** (Mandatory Access Control) через LSM ядра: поверх обычных прав (DAC) ограничивают, что процесс может делать, даже если он работает от root.

- **SELinux** (RHEL, Fedora) — метки (контексты) на всех файлах, процессах и портах (`ls -Z`, `ps -Z`), политика описывает, какой тип процесса к какому типу объекта имеет доступ. Мощный, но сложнее в настройке.
- **AppArmor** (Ubuntu, Debian, SUSE) — профили по путям к файлам для конкретных программ. Проще писать и читать (`aa-status`, `aa-complain`, `aa-enforce`).

SELinux: `getenforce`, `setenforce 0` переводит в **permissive** (нарушения только логируются) без перезагрузки. Полностью отключить можно только через загрузку ядра с `selinux=0` (параметр `SELINUX=disabled` в `/etc/selinux/config` в новых RHEL уже не отключает его полностью) и перезагрузку.

Вместо отключения чинят причину: `ausearch -m avc -ts recent`, `restorecon -Rv /path` (вернуть правильные контексты), `semanage port -a -t http_port_t -p tcp 8080`, `setsebool -P httpd_can_network_connect on`, в крайнем случае `audit2allow` для локального модуля.

</details>

### Supply chain и сканирование

17. Какие атаки на цепочку поставки ПО (supply chain) вы знаете и как от них защищаться?

<details>
  <summary>Ответ</summary>

Примеры атак:
- **typosquatting** — пакет с похожим именем (`reqeusts`) в PyPI/npm;
- **dependency confusion** — публичный пакет с именем внутреннего и более высокой версией, который менеджер пакетов предпочтёт;
- **захват аккаунта мейнтейнера / вредоносный релиз** популярной библиотеки (npm-пакеты с кражей токенов, в т.ч. самораспространяющиеся черви в экосистеме npm);
- **долгосрочное внедрение бэкдора** мейнтейнером (xz-utils, 2024);
- **компрометация CI/CD** или инструментов сборки (SolarWinds, Codecov, подмена тегов GitHub Action tj-actions/changed-files в 2025).

Защита:
- lock-файлы (`package-lock.json`, `poetry.lock`, `go.sum`) и установка строго по ним (`npm ci`), проверка хэшей (`pip install --require-hashes`);
- pinning базовых образов по digest (`FROM python:3.12-slim@sha256:...`) и GitHub Actions по полному SHA коммита, а не по тегу;
- внутренний прокси-репозиторий (Nexus, Artifactory) и явный scope для внутренних пакетов;
- SCA-сканирование, Dependabot/Renovate с ревью обновлений, задержка перед принятием только что вышедших версий;
- минимальные права токенов CI, изолированные раннеры, подпись артефактов и проверка provenance.

</details>

18. Что такое SBOM, подпись образов (cosign/Sigstore) и SLSA?

<details>
  <summary>Ответ</summary>

- **SBOM** (Software Bill of Materials) — перечень всех компонентов и их версий в артефакте в формате SPDX или CycloneDX. Позволяет быстро ответить «где у нас уязвимая версия библиотеки X».
  ```bash
  syft registry.example.com/app:1.4.2 -o cyclonedx-json > sbom.json
  grype sbom:./sbom.json
  ```
- **cosign (Sigstore)** — подпись и проверка образов и аттестаций в OCI-registry. Keyless-режим: CI получает OIDC-токен, Fulcio выдаёт короткоживущий сертификат на эту идентичность, запись о подписи попадает в прозрачный лог Rekor.
  ```bash
  cosign sign --yes registry.example.com/app@sha256:<digest>
  cosign verify registry.example.com/app@sha256:<digest> \
    --certificate-identity-regexp '^https://github.com/org/app/' \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com
  ```
  В кластере проверку подписи при деплое делают admission-политики: Kyverno (`verifyImages`), Sigstore policy-controller, Connaisseur.
- **SLSA** — фреймворк уровней зрелости защиты сборки: наличие provenance (кто, из какого коммита, каким билдом собрал артефакт), его подпись, изолированная и защищённая от вмешательства сборочная платформа. Provenance генерируется, например, `slsa-github-generator` или встроенными аттестациями GitHub.

</details>

19. Чем отличаются SAST, DAST и SCA? Как встроить сканеры в CI?

<details>
  <summary>Ответ</summary>

- **SAST** (Static Application Security Testing) — анализ исходного кода без запуска: инъекции, небезопасные функции. Semgrep, SonarQube, CodeQL.
- **DAST** (Dynamic) — атака на запущенное приложение снаружи (как чёрный ящик): XSS, инъекции, мисконфигурации заголовков. OWASP ZAP, Burp Suite. Запускают на стенде.
- **SCA** (Software Composition Analysis) — поиск известных CVE в сторонних зависимостях и проверка лицензий: Trivy, Grype, Snyk, OWASP Dependency-Check, Dependabot.
- Отдельно: сканирование секретов (gitleaks, trufflehog), IaC-сканирование (Trivy config, Checkov, KICS), сканирование образов.

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 registry.example.com/app:1.4.2
trivy fs --scanners vuln,secret,misconfig .
trivy config ./terraform
gitleaks detect --source .
```
Практика: быстрые проверки (секреты, SAST, SCA) на каждый MR, блокировать сборку только по критичным и исправимым находкам, исключения оформлять явно (`.trivyignore` с комментарием и сроком), регулярно пересканировать уже задеплоенные образы — новые CVE появляются и для старых артефактов.

</details>

### Контейнеры и Kubernetes

20. Как ограничить привилегии контейнера в Kubernetes через securityContext?

<details>
  <summary>Ответ</summary>

```yaml
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: registry.example.com/app@sha256:<digest>
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]   # только если реально нужно слушать порт < 1024
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```
- `runAsNonRoot` — kubelet откажется запускать контейнер от UID 0;
- `readOnlyRootFilesystem` — нельзя дописать бинарник или изменить конфиг; для записи — отдельные `emptyDir`;
- `capabilities.drop: ALL` — убирает root-привилегии ядра (`CAP_NET_RAW`, `CAP_SYS_ADMIN` и т.д.);
- `allowPrivilegeEscalation: false` — запрет получения прав через setuid;
- избегать `privileged: true`, `hostNetwork`, `hostPID`, `hostPath` (особенно монтирования `/var/run/docker.sock` или containerd-сокета — это фактически root на ноде).

</details>

21. Что такое Pod Security Admission? Чем он заменил PodSecurityPolicy?

<details>
  <summary>Ответ</summary>

PodSecurityPolicy удалён в Kubernetes 1.25. На смену пришёл встроенный **Pod Security Admission** (PSA), который применяет стандарты Pod Security Standards на уровне namespace:
- `privileged` — без ограничений (системные компоненты, CNI);
- `baseline` — запрещает очевидно опасное: privileged, hostNetwork/hostPID, hostPath, добавление опасных capabilities;
- `restricted` — плюс обязательные runAsNonRoot, drop ALL, seccomp, запрет privilege escalation.

Режимы: `enforce` (отклонять), `audit` (писать в аудит-лог), `warn` (предупреждение пользователю).
```bash
kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted
```
Для более гибких политик (разрешённые registry, обязательные лимиты, запрет тега `latest`, проверка подписи образов) используют Kyverno, OPA Gatekeeper или встроенные ValidatingAdmissionPolicy на CEL.

</details>

22. Как ограничить сетевой доступ и права доступа к API в Kubernetes (NetworkPolicy, RBAC, аудит)?

<details>
  <summary>Ответ</summary>

**NetworkPolicy** — по умолчанию все поды могут общаться со всеми. Работает, только если CNI поддерживает политики (Calico, Cilium; flannel — нет). Базовый шаг — default deny в namespace, затем явные разрешения:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: prod
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```
После этого нужно явно разрешить egress на DNS (kube-dns, порт 53 UDP/TCP), иначе сломается резолв.

**RBAC** — Role/ClusterRole (что можно) + RoleBinding/ClusterRoleBinding (кому). Избегать `cluster-admin` для людей и CI, wildcard'ов `*`, прав на `secrets`, `pods/exec`, `escalate`, `bind`, `impersonate`. Проверка:
```bash
kubectl auth can-i --list --as=system:serviceaccount:prod:app -n prod
kubectl auth can-i create pods/exec --as=jane -n prod
```
**Аудит-логи** API-сервера: `--audit-policy-file` и `--audit-log-path`; политика задаёт уровень детализации (`None`, `Metadata`, `Request`, `RequestResponse`) по ресурсам. Для Secret'ов — только `Metadata`, чтобы значения не попадали в лог. Логи отправляют в SIEM и алертят на `exec` в продовые поды, изменения RBAC, доступ к секретам.

</details>

23. Что такое Falco и зачем нужна runtime-безопасность?

<details>
  <summary>Ответ</summary>

Сканирование образов ловит известные уязвимости до запуска, но не видит, что происходит в работающем контейнере. **Falco** (CNCF) отслеживает системные вызовы через eBPF и по правилам генерирует события: запуск shell в контейнере, чтение `/etc/shadow`, запись в `/bin` или `/etc`, неожиданные исходящие соединения, запуск privileged-контейнера, доступ к сокету container runtime.

Правило состоит из `condition` (выражение над полями вроде `evt.type`, `proc.name`, `container.id`, `fd.name`), `output` (текст события) и `priority`. События отправляются через Falcosidekick в Slack, SIEM, Loki или на автоматическую реакцию (например, удалить под).

Альтернативы и дополнения: **Tetragon** (Cilium) — не только наблюдает, но и может блокировать действие в ядре; KubeArmor; коммерческие CNAPP-решения.

</details>

### Облако и реагирование на инциденты

24. Как правильно выдавать доступ к облаку из CI/CD и из подов без статических ключей?

<details>
  <summary>Ответ</summary>

Статические ключи (`AWS_ACCESS_KEY_ID`) живут долго, копируются и утекают. Вместо них — **федерация через OIDC**: CI или кластер выдаёт подписанный JWT о своей идентичности, облако проверяет его и выдаёт временные учётные данные роли (минуты–часы).

- **GitHub Actions → AWS**: в workflow `permissions: id-token: write`, шаг `aws-actions/configure-aws-credentials` с `role-to-assume`. В trust policy роли — провайдер `token.actions.githubusercontent.com` и условие на `sub`, например `repo:org/app:ref:refs/heads/main`, чтобы роль мог получить только нужный репозиторий и ветка. Аналогично GitLab CI (`id_tokens`), GCP Workload Identity Federation, Azure federated credentials.
- **Поды в Kubernetes**: EKS IRSA или EKS Pod Identity, GKE Workload Identity, Azure Workload Identity — ServiceAccount пода связывается с облачной ролью.

Плюс общие правила IAM: отдельные роли на сервис, least privilege, запрет root-аккаунта для работы, MFA, SCP/Organization Policies как ограничители сверху, аудит (CloudTrail) и периодический поиск неиспользуемых прав (IAM Access Analyzer).

</details>

25. Сервер скомпрометирован. Ваши действия?

<details>
  <summary>Ответ</summary>

1. **Не паниковать и не удалять**: не перезагружать и не переустанавливать сразу — пропадут улики в памяти. Сообщить ответственным (security, руководитель), начать вести хронологию действий.
2. **Изолировать**: отключить от сети через security group/firewall (разрешив доступ только для расследования), вывести из балансировщика. Для облака — снапшот дисков.
3. **Собрать данные**: дамп памяти (LiME, AVML), список процессов и соединений (`ps auxf`, `ss -tupan`, `ls -l /proc/*/exe`), пользователи и ключи (`/etc/passwd`, `~/.ssh/authorized_keys`), cron и systemd-юниты, недавно изменённые файлы (`find / -mtime -3 -type f`), логи (`/var/log/auth.log`, `journalctl`, `last`, `lastb`, auditd). Встроенным утилитам на скомпрометированной машине нельзя доверять полностью (rootkit) — лучше анализировать снапшот диска на отдельной машине.
4. **Оценить масштаб**: какие секреты были на сервере (ключи, токены, пароли к БД) — считать их скомпрометированными и **ротировать**; проверить соседние хосты на lateral movement.
5. **Восстановить**: не «лечить» сервер, а пересоздать с чистого образа (IaC), закрыв вектор входа (патч, закрытый порт, отозванный ключ); данные — из проверенного бэкапа.
6. **Postmortem**: как проникли, почему не заметили раньше, какие меры и алерты добавить.

</details>
