### **Список вопросов для интервью на позицию L3 Engineer (Linux, K8s, PostgreSQL, Ansible, Terraform)**  

#### **1. Linux**  
**Вопрос:** Как проверить загрузку CPU, памяти и дискового I/O в Linux? Какие утилиты вы используете?  
**Ответ:**  
- `top`, `htop` – мониторинг процессов и CPU.  
- `vmstat`, `free -h` – информация о памяти.  
- `iostat`, `iotop` – мониторинг дискового ввода-вывода.  
- `df -h`, `du -sh` – проверка использования дискового пространства.  

**Вопрос:** Как найти и убить зависший процесс?  
**Ответ:**  
- `ps aux | grep <process_name>` – найти PID процесса.  
- `kill -9 <PID>` – принудительное завершение.  
- `pkill <process_name>` – убить по имени.  

**Вопрос:** Как настроить статический маршрут в Linux?  
**Ответ:**  
```bash
ip route add <network>/<mask> via <gateway_ip> dev <interface>
```
Или через `/etc/network/interfaces` (в Debian) или `/etc/sysconfig/network-scripts/` (в RHEL).  

---  

#### **2. Kubernetes (K8s)**  
**Вопрос:** Как развернуть StatefulSet для PostgreSQL в Kubernetes? Какие особенности нужно учесть?  
**Ответ:**  
- StatefulSet нужен для управления состоянием (PersistentVolumes).  
- Требуется `storageClass` для динамического выделения PV.  
- Важно настроить `initContainers` для инициализации БД.  
- Желательно использовать `headless Service` для стабильных DNS-записей.  

**Вопрос:** Как посмотреть логи пода, который упал и был перезапущен?  
**Ответ:**  
```bash
kubectl logs <pod_name> --previous
```
Или через `kubectl describe pod <pod_name>` для анализа событий.  

**Вопрос:** Как обновить Deployment без downtime?  
**Ответ:**  
- Стратегия `RollingUpdate` (по умолчанию).  
- Можно настроить `maxSurge` и `maxUnavailable` в `spec.strategy`.  

---  

#### **3. PostgreSQL**  
**Вопрос:** Как настроить репликацию в PostgreSQL?  
**Ответ:**  
- **Streaming Replication** (асинхронная/синхронная):  
  - На мастере: `wal_level = replica`, создать пользователя для репликации.  
  - На реплике: `primary_conninfo` в `recovery.conf` (или через `pg_basebackup`).  

**Вопрос:** Как найти самые медленные запросы в PostgreSQL?  
**Ответ:**  
```sql
SELECT query, total_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;
```
(Требуется включить `pg_stat_statements` в `postgresql.conf`).  

**Вопрос:** Что такое VACUUM и AUTOVACUUM в PostgreSQL?  
**Ответ:**  
- `VACUUM` – очистка "мертвых" строк (не освобождает место на диске).  
- `VACUUM FULL` – перезаписывает таблицу, освобождая место (блокирует таблицу).  
- `AUTOVACUUM` – автоматический фоновый процесс для поддержания БД в оптимальном состоянии.  

---  

#### **4. Ansible**  
**Вопрос:** Как создать плейбук для установки и настройки Nginx?  
**Пример ответа:**  
```yaml
- name: Install and configure Nginx
  hosts: webservers
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    - name: Copy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

**Вопрос:** Как использовать динамический инвентарь в Ansible?  
**Ответ:**  
- Можно использовать скрипт (Python, Bash), который возвращает JSON с хостами.  
- Или интеграцию с облачными провайдерами (AWS, GCP) через плагины (`aws_ec2`, `gcp_compute`).  

---  

#### **5. Terraform**  
**Вопрос:** Как создать модуль в Terraform для развертывания виртуальной машины в AWS/GCP?  
**Пример ответа:**  
```hcl
module "ec2_instance" {
  source        = "./modules/ec2"
  ami           = "ami-123456"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.main.id
}
```

**Вопрос:** Что такое Terraform State и как его обезопасить?  
**Ответ:**  
- `terraform.tfstate` – файл с текущим состоянием инфраструктуры.  
- **Обезопасить можно:**  
  - Хранить state в S3/GCS с блокировкой через DynamoDB.  
  - Использовать `backend "remote"` (Terraform Cloud).  
  - Шифровать sensitive данные (например, через `s3_kms_key`).  

---  

### **Дополнительные вопросы (Soft Skills & Troubleshooting)**  
- Опишите, как вы бы debug'или проблему с "503 Service Unavailable" в Kubernetes.  
- Как бы вы автоматизировали резервное копирование PostgreSQL?  
- Как вы организуете CI/CD для инфраструктурного кода (Terraform + Ansible)?  

Этот список можно адаптировать под конкретные требования вакансии. Если нужно углубиться в какую-то тему – дайте знать!
