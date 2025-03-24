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


### **1.1** Как найти топ-5 процессов по потреблению памяти?
```bash
ps aux --sort=-%mem | head -n 6
```

### **1.2** Как добавить статический маршрут для конкретного интерфейса?
```bash
ip route add 192.168.1.0/24 dev eth0
```

### **1.3** Как проверить, какой процесс использует открытый файл?
```bash
lsof /path/to/file
```

### **1.4** Как настроить автоматическое монтирование NFS при загрузке?
```bash
# В /etc/fstab:
nfs-server:/export /mnt nfs defaults 0 0
```

### **1.5** Как изменить hostname без перезагрузки?
```bash
hostnamectl set-hostname newname
```

### **1.6** Как найти все файлы размером >100MB?
```bash
find / -type f -size +100M
```

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


### **2.1** Как посмотреть все Pods в конкретном namespace?
```bash
kubectl get pods -n <namespace>
```

### **2.2** Как создать ConfigMap из файла?
```bash
kubectl create configmap my-config --from-file=config.properties
```

### **2.3** Как подключиться к работающему Pod в debug-режиме?
```bash
kubectl exec -it <pod> -- /bin/bash
```

### **2.4** Как масштабировать Deployment?
```bash
kubectl scale deployment/my-app --replicas=5
```

### **2.5** Как посмотреть логи нескольких Pods одновременно?
```bash
kubectl logs -l app=my-app --tail=100
```

### **2.6** Как обновить образ в Deployment?
```bash
kubectl set image deployment/my-app my-app=nginx:1.21
```

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


### **3.1** Как посмотреть активные подключения?
```sql
SELECT * FROM pg_stat_activity;
```

### **3.2** Как создать read-only пользователя?
```sql
CREATE ROLE readonly WITH LOGIN PASSWORD 'pass';
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
```

### **3.3** Как сделать backup одной таблицы?
```bash
pg_dump -t my_table mydb > table_backup.sql
```

### **3.4** Как перестроть все индексы в базе?
```sql
REINDEX DATABASE mydb;
```

### **3.5** Как посмотреть размер всех баз данных?
```sql
SELECT datname, pg_size_pretty(pg_database_size(datname)) 
FROM pg_database;
```

### **3.6** Как проверить прогресс длительного запроса?
```sql
SELECT pid, query, now() - query_start AS duration 
FROM pg_stat_activity 
WHERE state = 'active';
```

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


### **4.1** Как запустить плейбук только на определенных хостах?
```bash
ansible-playbook playbook.yml -l webservers
```

### **4.2** Как использовать переменные из файла?
```yaml
- hosts: all
  vars_files:
    - vars/secrets.yml
```

### **4.3** Как создать кастомный модуль?
```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(argument_spec={})
    module.exit_json(changed=False)

if __name__ == '__main__':
    main()
```

### **4.4** Как сделать обработку ошибок?
```yaml
- name: Try something
  command: /bin/false
  ignore_errors: yes
```

### **4.5** Как использовать шаблоны Jinja2?
```yaml
- name: Configure app
  template:
    src: templates/app.conf.j2
    dest: /etc/app.conf
```

### **4.6** Как создать динамический инвентарь для AWS?
```bash
ansible-inventory -i aws_ec2.yml --graph
```

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


### **5.1** Как инициализировать новый проект?
```bash
terraform init
```

### **5.2** Как проверить синтаксис без применения?
```bash
terraform validate
```

### **5.3** Как посмотреть план изменений?
```bash
terraform plan
```

### **5.4** Как применить изменения?
```bash
terraform apply
```

### **5.5** Как удалить все ресурсы?
```bash
terraform destroy
```

### **5.6** Как использовать переменные окружения?
```bash
export TF_VAR_region="us-west-1"
```
---  

### **Дополнительные вопросы (Soft Skills & Troubleshooting)**  
- Опишите, как вы бы debug'или проблему с "503 Service Unavailable" в Kubernetes.  
- Как бы вы автоматизировали резервное копирование PostgreSQL?  
- Как вы организуете CI/CD для инфраструктурного кода (Terraform + Ansible)?  


### **6.1** Как бы вы настроили CI/CD для инфраструктурного кода?
**Ответ:** GitLab CI + Terraform Cloud + Atlantis

### **6.2** Как мониторить состояние кластера K8s?
**Ответ:** Prometheus + Grafana + Alertmanager

### **6.3** Как организовать backup PostgreSQL в K8s?
**Ответ:** WAL-G + Kubernetes CronJob

### **6.4** Как обеспечить безопасность Ansible плейбуков?
**Ответ:** Ansible Vault + RBAC + минимальные привилегии

### **6.5** Как оптимизировать стоимость облачных ресурсов?
**Ответ:** Autoscaling + Spot Instances + Reserved Instances

### **6.6** Как бы вы развернули высокодоступный кластер PostgreSQL?
**Ответ:** Patroni + etcd + HAProxy
