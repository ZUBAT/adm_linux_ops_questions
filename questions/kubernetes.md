## Kubernetes

1. Чем отличается Kubernetes от Openshift?

<details>
  <summary>Ответ</summary>

https://www.redhat.com/cms/managed-files/cl-openshift-and-kubernetes-ebook-f25170wg-202010-en.pdf

1. Openshift имеет более строгие политики безопасности и модели аутентификации.
2. Openshift поддерживает полную интеграцию CI/CD Jenkins.
3. Openshift имеет веб-консоль по-умолчанию. В Kubernetes консоль необходимо дополнительно устанавливать консоль.
4. В Kubernetes возможно устанавливать сторонние сетевые плагины. В Openshift используется собственное сетевое решение Open vSwitch, которое предоставляет 3 различный плагина.
5. Kubernetes может быть установлен практически на любой дистрибутив Linux. Openshift имеет ограничения на устанавливаемые дистрибутивы, преимущественно используются RH-дистрибутивы.
6. Kubernets доступен в большинстве облачных платформ - GCP, AWS, Azure, Yandex.Cloud. Openshift доступен на облачной платформе Azure и облаке от IBM.
7. По-умолчанию, в Openshift поды в кластере могут быть запущены только под обычным пользователем, чтобы запустить под под пользователем root необходимо выдать права для сервисного аккаунта. В Kubernetes по-умолчанию поды могут быть запущены по пользователем root.

</details>

2. Чем отличаются *ReplicationController* от *ReplicaSet*?

<details>
  <summary>Ответ</summary>

ReplicationController гарантирует, что указанное количество реплик подов будут работать одновременно. Другими словами, ReplicationController гарантирует, что под или набор подов всегда активен и доступен.

ReplicaSet - это следующее поколение Replication Controller. Единственная разница между ReplicaSet и Replication Controller - это поддержка селектора. ReplicaSet поддерживает множественный выбор в селекторе, тогда как ReplicationController поддерживает в селекторе только выбор на основе равенства.

</details>

3. Если на каждой ноде Kubernetes кластера нужно запустить контейнер, то какой ресурс Kubernetes вам подойдет?

<details>
  <summary>Ответ</summary>

DaemonSet является контроллером, основным назначением которого является запуск подов на всех нодах кластера. Если нода добавляется/удаляется — DaemonSet автоматически добавит/удалит под на этой ноде.

DaemonSet подходят для запуска приложений, которые должны работать на всех нодах, например — екпортёры мониторинга, сбор логов и так далее.

</details>

4. Как поды разнести на разные ноды?

<details>
  <summary>Ответ</summary>

Необходимо настроить podAntiAffinity. Данное указание определяет, что для определенных подов следует использовать их размещание на разных нодах.

</details>

5. В облаке есть 3 зоны доступности. Как сделать так, чтобы поды приложения распределились по этим зонам доступности равномерно?

<details>
  <summary>Ответ</summary>

Необходимо настроить [podAntiAffinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#an-example-of-a-pod-that-uses-pod-affinity). Либо, более новый вариант для данной задачи, настроить [topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) с указание ключа лейбла зон.

</details>

6. Как контейнеры одного пода разнести на разные ноды?

<details>
  <summary>Ответ</summary>

Никак. Под - минимальная и неделимая сущность, Kubernetes оперирует подами, а не отдельными контейнерами. 

</details>

7. Как обеспечить, чтобы поды никогда не перешли в состояние Evicted на ноде? 

<details>
  <summary>Ответ</summary>

Когда узлу (node) кластера не хватает памяти или дискового пространства, он активирует флаг, сигнализирующий о данной проблеме. Данное действие блокирует любое новое выделение ресурсов на ноде и запускает процесс "выселения" (evicted) пода с ноды.

В этот момент kubelet начинает восстанавливать ресурсы, удаляя контейнеры и объявляя поды, как Failed, пока использование ресурсов снова не станет ниже порога "выселения".

Сначала kubelet пытается освободить ресурсы узла, особенно диск, путем удаления мертвых модулей и их контейнеров, а затем неиспользуемых образов. Если этого недостаточно, kubelet начинает выселять поды конечных пользователей в следующем порядке:

1. Best Effort.
2. Burstable поды, использующие больше ресурсов, чем запрос истощенного ресурса.
3. Burstable поды, использующие меньше ресурсов, чем запрос истощенного ресурса.

Чтобы под не был удален при "выселении", необходимо настроить политики QoS для пода как Guaranteed.

Подробнее в документации Kubernetes: [Create a Pod that gets assigned a QoS class of Guaranteed](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed)

Кроме того, можно использовать сущность кубернетиса PodDisruptionBudget, которая позволит регулировать количество вытесняемых подов и обеспчивать гарантированную доступность для конкретного микросервиса https://kubernetes.io/docs/tasks/run-application/configure-pdb/

</details>

8. За что отвечает kube-proxy?

<details>
  <summary>Ответ</summary>

Kube-proxy отвечает за взаимодействие между сервисами на разных нодах кластера.

</details>

9. Что находится на master ноде?

<details>
  <summary>Ответ</summary>

- Kube-apiserver отвечает за оркестрацию всех операций кластера.
- Controller-manager (Node controller + Replication Controller) Controller отвечает за функции контроля за нодами, репликами.
- ETCD cluster (распределенное хранилище ключ-значение) ETCD хранит информацию о кластере и его конфигурацию.
- Kube-sheduler отвечает за планирование приложений и контейнеров на нодах.

По-умолчанию на master ноде не размещаются контейнеры приложений, но данный фунционал возможно настроить.

</details>

10. Что находится на worker ноде?

<details>
  <summary>Ответ</summary>

- Kubelet слушает инструкции от kube-apiserver и разворачивает или удаляет контейнеры на нодах.
- Kube-proxy отвечает за взаимодействие между сервисами на разных нодах кластера.

На worker нодах по-умолчанию размещаются контейнеры приложений. На каждой ноде кластера устанавливается Docker или другая платформа контейнеризации (например RKT или containterd). На Master ноде также устанавливается Docker, если необходимо использовать компоненты Kubernetes в контейнерах.

**Актуально на 2026:** поддержка Docker Engine через dockershim удалена в Kubernetes 1.24, rkt давно заброшен. На нодах используется CRI-совместимый runtime — containerd или CRI-O (Docker возможен только через внешний адаптер cri-dockerd). Компоненты control plane в kubeadm запускаются как static pods через тот же runtime.

</details>

11. Как установить Kubernetes?

<details>
  <summary>Ответ</summary>

1. Следовать инструкции [установки kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

2. Установка с [использованием kubespray](https://github.com/kubernetes-sigs/kubespray).

</details>

12. Чем отличается *StatefulSet* от *Deployment*?

<details>
  <summary>Ответ</summary>

*Deployment* - ресурс Kubernetes предназнваенный для развертывания приложения без сохранения состояния. При использовании PVC все реплики будут использовать один и тот же том, и ни один из них не будет иметь собственного состояния.

*StatefulSet* - поддерживают состояние приложений за пределами жизненного цикла отдельных модулей pod, например для хранилища. Используется для приложений с отслеживанием состояния, каждая реплика модуля будет иметь собственное состояние и будет использовать свой собственный том.

</details>

13. Что такое *операторы* в понятиях Kubernetes?

<details>
  <summary>Ответ</summary>

Операторы -- это программные расширения Kubernetes,призванное автоматизировать выполнение рутинных действий над объектами кластера при определённых событиях.

Оператор работает по подписке на события к API Kubernetes.

</details>

14. Почему *DaemonSet* не нужен scheduler?

<details>
  <summary>Ответ</summary>

DaemonSet гарантирует, что определенный под будет запущен на всех нодах кластера. При наличии DaemonSet в кластере на любой из существующих и будущих нод в кластере зарезервированы ресурсы для пода на ноде.

Здесь стоит сделать оговорку насчет того, что DaemonSet может работать не на всех нодах кластера, а на некоторых, выбранных, например, по nodeSelector. К примеру, у нас есть GPU ноды и нам нужно на все эти ноды задеплоить микросервис выполняющий вычисления на GPU.

</details>

15. В каких случаях не отработает перенос пода на другую ноду?

<details>
  <summary>Ответ</summary>

Если на другой ноде нет ресурсов для размещения пода или нет сетевой доступности до ноды.

</details>

16. Что делает *ControllerManager*?

<details>
  <summary>Ответ</summary>

Controller выполняет постоянный процесс мониторинга состояния кластера и различных компонент.

Controller-manager (Node controller + Replication Controller) - Controller отвечает за функции контроля за нодами, репликами.

</details>

17. Администратор выполняет команду `kubectl apply -f deployment.yaml`. Опишите по порядку что происходит в каждом из узлов Kubernetes и в каком порядке.

<details>
  <summary>Ответ</summary>

Клиент kubectl обращается к мастер-серверу kube-apiserver (стандартно на порт 6443), адрес мастер сервер задан в *.config* файле. В запросе передаётся информация, которую нужно применить в кластере обращения. API-сервер обращается к etcd хранилищу, проверяет наличие конфигурации запрашиваемого ресурса. Если конфигурация в хранилище etcd есть, то API-сервер сравнивает новую конфигурацию с конфигурацией в базе данных: если конфигурация одинаковая, то изменений в кластере не происходит, клиенту отдается ответ об успешности запрашиваемого действия, если конфигурации нет в etcd, то если требуемое действие касается создания сущностей, которые требуют ресурсов кластера (создания подов, хранилища pv/pvc и т.д.), scheduler проверяет возможность размещения подов на нодах и после чего происходит создание подов, при этом controll-manager контроллирует создание нужного поличества реклик сущности. После создания трубуемой сущности, происходит запись в etcd, controll-manager продолжает отслеживать состояние сущностей на протяжении всего цикла его жизни.

**Актуально на 2026:** точнее цепочка выглядит так: kubectl (для `apply` — с вычислением patch, либо server-side apply) → kube-apiserver: аутентификация, авторизация (RBAC), mutating admission, валидация, validating admission → запись Deployment в etcd. Дальше все компоненты работают через watch к API-серверу, а не к etcd напрямую: Deployment controller (в kube-controller-manager) создаёт ReplicaSet, ReplicaSet controller — объекты Pod без ноды; kube-scheduler выбирает ноду и записывает binding; kubelet этой ноды через CRI (containerd) создаёт sandbox, CNI настраивает сеть, запускаются контейнеры, kubelet обновляет статус пода. Endpoints-контроллер добавляет готовый под в EndpointSlice сервиса. В etcd пишет только kube-apiserver.

</details>

18. Как выполнить обновление Kubernetes в контуре где нет интернета?

<details>
  <summary>Ответ</summary>

Предварительно с рабочего кластера с новой версией Kubernetes и доступом в Интернет необходимо скачать требуемые пакеты kubeadm и образы api, controllmanager, etcd, scheduler, kubelet, docker-ce. Скачать пакеты с разрешением зависимостей возможно командой `yumdownloader --resolve kubeadm`. Образы скачиваются локально в архив `docker save <имя_образа> > <имя_образа>`.tar.

1. Удалить приложения из кластера.
```sh
helm delete --purge all
```

2. После того, как все необходимые компоненты скачены и загружены в контур без Интернета, выполняет команду сброса kubeadm.
```
kubeadm reset
```

3. Удаляем CNI-плагин Kubernetes.
```
yum remove kubernetes-cni-plugins
```

4. Локально устанавливаем необходимые пакеты.
```
yum install ./kubernetes_packages/*.rpm
```

5. Загружаем образы сервисов Kubernetes.
```
docker load < <имя_образа>.tar
```

6. Отключаем SELinux.
```sh
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

7. Определяем IP адрес master сервера.
```
IP=$(ip route get 1 | awk '{print $NF;exit}')
```

8. Инициализируем кластер Kubernetes.
```
kubeadm init --apiserver-advertise-address=$IP
```

9. Далее необходимо установить CNI-плагин, например Weave.

10. Разрешить на master ноде запускать контейнеры приложения.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

На worker ноде выполняются аналогичные действия, кроме того, что устанавливается только kubelet. При инициализации master ноды выдаётся token для подключения worker нод, его необходимо сохранить, чтобы позже включить woker ноду в кластер.

**Актуально на 2026:** описанный способ — это переустановка кластера с потерей состояния, а не обновление. Штатно обновляются через `kubeadm upgrade plan/apply` по одной минорной версии, образы для закрытого контура заранее получают через `kubeadm config images list/pull` и загружают в локальный реестр (`imageRepository` в конфиге kubeadm) или в containerd (`ctr -n k8s.io images import`), т.к. Docker больше не используется как runtime. Taint `node-role.kubernetes.io/master` заменён на `node-role.kubernetes.io/control-plane`, Weave Net больше не развивается, `helm delete --purge` — синтаксис Helm 2.

</details>

19. Чем Router в Openshift отличается от Ingress в Kubernetes?

<details>
  <summary>Ответ</summary>

Router Openshift использует haproxy, как прокси-вебсервер. Ingress как в Kubernetes, так и OpenShift может быть разным (nginx, haproxy, caddy, etc).

</details>

20. Почему для установки Kubernetes требуется отключить swap?

<details>
  <summary>Ответ</summary>

Планировщик Kubernetes определяет наилучший доступный узел для развертывания вновь созданных модулей. Если в хост-системе разрешена подкачка памяти, это может привести к проблемам с производительностью и стабильностью в Kubernetes. По этой причине Kubernetes требует, чтобы вы отключили swap в хост-системе.

**Актуально на 2026:** требование больше не абсолютное. Поддержка swap на Linux-нодах стабилизирована: kubelet запускается с `failSwapOn: false`, а `memorySwap.swapBehavior` задаёт режим — `NoSwap` (по умолчанию, поды swap не используют) или `LimitedSwap` (swap доступен подам класса Burstable пропорционально их memory request). Нужна cgroup v2. На практике многие по-прежнему отключают swap для предсказуемости.

</details>

21. Что такое *Pod* в Kubernetes?

<details>
  <summary>Ответ</summary>

Минимальная сущность в Kubernetes и является абстракцией над контейнерами. Pod представляет собой запрос на запуск одного или более контейнеров на одном узле.

</details>

22. Сколько контейнеров запускается в одном поде?

<details>
  <summary>Ответ</summary>

По умолчанию при запуске одного контейнера в одном поде запускается еще *pause* контейнер. Итого, в одном поде может быть запущено *n+1* контейнеров.

</details>

23. Для чего нужен *pause* контейнер в каждом поде?

<details>
  <summary>Ответ</summary>

Контейнер *pause* запускается первым в поде и создаёт сетевое пространство имен для пода. Затем Kubernetes выполняет CNI плагин для присоединения контейнера *pause* к сети. Все контейнеры пода используют сетевое пространство имён (netns) этого *pause* контейнера.

</details>

24. Чем отличается *Deployment* от *DeploymentConfig* (Openshift)?

<details>
  <summary>Ответ</summary>

https://docs.openshift.com/container-platform/4.1/applications/deployments/what-deployments-are.html

</details>

25. Для чего нужны *Startup*, *Readiness*, *Liveness* пробы? Чем отличаются?

<details>
  <summary>Ответ</summary>

Kubelet использует **Liveness** пробу для проверки, когда перезапустить контейнер. Например, Liveness проба должна поймать блокировку, когда приложение запущено, но не может ничего сделать. В этом случае перезапуск приложения может помочь сделать приложение доступным, несмотря на баги.

Kubelet использует **Readiness** пробы, чтобы узнать, готов ли контейнер принимать траффик. Pod считается готовым, когда все его контейнеры готовы.

Одно из применений такого сигнала - контроль, какие Pod будут использованы в качестве бекенда для сервиса. Пока Pod не в статусе ready, он будет исключен из балансировщиков нагрузки сервиса.

Kubelet использует **Startup** пробы, чтобы понять, когда приложение в контейнере было запущено. Если проба настроена, он блокирует Liveness и Readiness проверки, до того как проба становится успешной, и проверяет, что эта проба не мешает запуску приложения. Это может быть использовано для проверки работоспособности медленно стартующих контейнеров, чтобы избежать убийства kubelet'ом прежде, чем они будут запущены.

</details>

26. Чем отличаются *Taints* и *Tolerations* от *Node Afiinity*?

<details>
  <summary>Ответ</summary>

*Node Affinity* - это свойство подов, которое позволяет нодам выбирать необходимый под. Node Affinity позволяет ограничивать для каких узлов под может быть запланирован, на основе меток на ноде. Node Affinity требует указания nodeSelector для пода с необходимым label ноды кластера.

Типы Node Affinity:
`<Требование 1><Момент 1><Требование 2><Момент 2>
requiredDuringSchedulingRequiredDuringExecution`

| Тип \ Момент | DuringScheduling | DuringExecution |
|-|-|-|
| Тип 1 | Required | Ignored |
| Тип 2 | Preferred | Ignored |
| Тип 3 | Required | Required |

Существуют определенные операторы nodeAffinity: In, NotIn, Exists, DoesNotExist, Gt или Lt.

**Актуально на 2026:** в API реально существуют только `requiredDuringSchedulingIgnoredDuringExecution` и `preferredDuringSchedulingIgnoredDuringExecution`; вариант `...RequiredDuringExecution` (выселение при смене меток ноды) так и не реализован. Для выселения подов с ноды используют taint с эффектом `NoExecute`.

---

*Taints* - это свойство нод, которое позволяет поду выбирать необходимую ноду. Tolerations применяеются к подам и позволяют (но не требуют) планировать модули на нодах с соответствующим Taints.

Установить для ноды Taints:
```
kubectl taint nodes <node-name> key=value:taint-effect
```
Taint-effect принимает значения - NoSchedule, PreferNoSchedule, NoExecute.

Пример:
```
kubectl taint nodes node1 app=blue:NoSchedule
```

- NoSchedule означает, что пока в спецификации пода не будет соответствующей записи tolerations, он не сможет быть развернут на ноде (в данном примере node10).

- PreferNoSchedule— упрощённая версия NoSchedule. В этом случае планировщик попытается не распределять поды, у которых нет соответствующей записи tolerations на ноду, но это не жёсткое ограничение. Если в кластере не окажется ресурсов, то поды начнут разворачиваться на этой ноде.

- NoExecute — этот эффект запускает немедленную эвакуацию подов, у которых нет соответствующей записи tolerations.

Taints и Tolerations работают вместе, чтобы гарантировать, что поды не запланированы на несоответствующие ноды. На ноду добавляется один или несколько Taints и это означает, что нода не должна принимать никакие поды, не относящиеся к Taints.

---

Taints и Tolerations не гарантирует, что определенный под будет размещен на нужной ноде. NodeAffinity - не гарантирует, что на определенной ноде, кроме выбранных подов, не будет размещены другие поды. 

</details>

27. Чем отличаются *Statefulset* и *Deployment* в плане стратегии обновления подов Rolling Update?

<details>
  <summary>Ответ</summary>

Стратегия обновления Rolling Update в **Deployment** предполагает последовательное обновление подов: сначала будет создан новый под, затем будет переключен трафик на новый под и затем удален старый под.

Стратегия обновления Rolling Update в **StatefulSet** предполагает обновление подов в обратном порядке, то есть под сначала будет удален, а потом установлен новый.

</details>

28. Для чего в Kubernetes используются порты 2379 и 2380?

<details>
  <summary>Ответ</summary>

2379 и 2380 - порты, которые используются etcd. 
2379 используется для взаимодействия etcd с компонентами control plane. 2380 используется только для взаимодействия компонентов etcd в кластере, при наличии множества master нод в кластере.

</details>

29. Задан следующий yaml файл для создания пода Test. Как сделать так, чтобы контейнеры nginx и redis пода test были размещены разных нодах кластера при условии, что существуют лейблы нод `disk=ssd` и `disk=hard`?
```
apiVersion: v1
kind: Pod
metadata:
  name: Test
spec:
  containers:
  - name: nginx
    image: nginx
  - name: redis
    image: redis
  nodeSelector:
    disk: ssd
```

<details>
  <summary>Ответ</summary>

Никак. Контейнеры одного пода могут размещаться только на одной ноде. Под является неделимой сущностью Kubernetes.

</details>

30. Какую функцию выполняют `indent` и `nindent` в Helm чартах?

<details>
  <summary>Ответ</summary>

`indent` делает отступ каждой строки в заданном списке до указанной ширины отступа.
`nindent` аналогична функции `indent`, но добавляет символ новой строки в начало каждой строки в списке.

</details>

31. Чем отличается *Deployment* от *ReplicaSet*?

<details>
  <summary>Ответ</summary>

ReplicaSet гарантирует, что определенное количество экземпляров подов (Pods) будет запущено в кластере Kubernetes.

Deployment предоставляет возможность декларативного обновления для объектов типа поды (Pods) и наборы реплик (ReplicaSets).

Deployment - уровень абстрации над ReplicaSet. Deployment будет создавать объект ReplicaSet, но с возможностью rolling-update и rollback.

Чтобы сохранить состояние при разворачивании Deployment необходимо установить ключ `--record` при применении манифеста.

**Актуально на 2026:** флаг `--record` объявлен устаревшим. Причину изменения в `kubectl rollout history` записывают аннотацией: `kubectl annotate deployment/web kubernetes.io/change-cause="image 1.2.3"` (или в `metadata.annotations` манифеста).

</details>

32. Чем отличается *Deployment* от *StatefulSet*?

<details>
  <summary>Ответ</summary>

Deployment выполняет обновление подов и RelicaSets, и является наиболее используемым ресурсом Kubernetes для деплоя приложений, как правило – stateless приложений, но если подключить Persistent Volume – приложение можно использовать как stateful, но все поды деплоймента будут совместно использовать это хранилище и данные из него. Для PVC можно указать режим доступа как `ReadWriteMany`, так и `ReadOnlyMany`.

StatefulSet используются для управления stateful-приложениями. Создаёт не ReplicaSet, а Pod напрямую с уникальным именем. В связи с этим – при использовании StatefulSet нет возможности выполнить откат версии, но можно его удалить или выполнить скейлинг. При обновлении StatefulSet – будет выполнено RollingUpdate всех подов. StatefulSet использует `volumeClaimTemplates` для описания хранилища и при использовании PVC для каждого пода будет создан уникальный PVC и режимом доступа `ReadWriteOnce`.

</details>

33. Что такое HPA (Horizontal Pod Autoscaling)? Как он работает и что для этого нужно?

<details>
  <summary>Ответ</summary>

[HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) - механизм, который позволяет указать нужную метрику(и) настроить автоматический порог масштабирования Pod’ов в зависимости от изменения её значений.

Чтобы HPA работал необходимо, чтобы в кластере был установлен metrics-server, чтобы считывать меетрики потребления ресурсов. По умолчанию HPA можно настроить для метрики потребления CPU и/или памяти. Возможно расширение функционала HPA с помощью [keda](https://keda.sh/).

</details>

34. Что такое Headless сервис?

<details>
  <summary>Ответ</summary>

При указании `ClusterIP: None` для сервиса мы создаём "безголовый сервис", у данного сервиса не будет виртуального IP адреса. Headless сервис это просто А-запись в системе DNS, таким образом имя сервиса преобразуется не в виртуальный IP сервиса, а сразу в IP пода. Headless сервисы полезны, когда приложение само должно управлять тем, к какому Pod подключаться. Например, gRPC-клиенты держат по одному соединению с сервисами и сами управляют запросами, мультиплексируя запросы к одному серверу. В случае использования ClusterIP клиент может создать одно подключение и нагружать ровно один Pod сервера.

</details>

35. Что такое ExternalName сервис?

<details>
  <summary>Ответ</summary>

Сервис типа ExternalName добавляет запись типа CNAME во внутренний DNS сервер Kubernetes. Например:
```
apiVersion: v1
kind: Service
metadata:
  name: ya-ru
spec:
  type: ExternalName
  externalName: ya.ru
```
Для сервиса `ya-ru` не создаётся endpoint. Поэтому сразу переходим к запросам к DNS.
```
ya-ru.default.svc.cluster.local. 5 IN CNAME   ya.ru.
```

</details>

36. Что такое ExternalIP сервис?

<details>
  <summary>Ответ</summary>

При определении сервиса можно добавить поле externalIPs, в котором можно указать IP адрес машины кластера. При обращении на этот IP и указанный в сервисе порт, запрос будет переброшен на соответствующий сервис.
Например:
```
apiVersion: v1
kind: Service
metadata:
  name: external-svc-nginx
  labels:
    app: nginx
spec:
  ports:
    - name: http-main
      port: 8080
      protocol: TCP
      targetPort: 8090
  selector:
    app: nginx
  externalIPs:
    - 192.168.218.178
```
При обращении к 192.168.218.178:8080 запрос будет переброшен к сервису external-svc-nginx:8080

</details>

37. Что такое NodePort сервис?

<details>
  <summary>Ответ</summary>

Сервисы типа NodePort открывают порт на каждой ноде кластера на сетевых интерфейсах хоста. Все запросы, приходящие на этот порт, будут пересылаться на endpoints, связанные с данным сервисом.
Диапазон портов, который можно использовать в NodePort — 30000-32767. Но его можно изменить при конфигурации кластера.

</details>

38. Что такое capabilities? Для чего нужно их описывать?

<details>
  <summary>Ответ</summary>

[Capabilities](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container) - это разрешения на уровне ядра, которые позволяют гранулярно управлять разрешениями на вызовы ядра, вместо того, чтобы запускать все от имени пользователя root. Capabilities позволяют изменять права доступа к файлам, управлять сетевой подсистемой и выполняет общесистемные функции администрирования.

</details>

39. Через какие этапы проходит запрос в kube-apiserver перед тем, как объект попадёт в etcd?

<details>
  <summary>Ответ</summary>

1. **Аутентификация** — кто делает запрос: клиентский x509-сертификат, bearer-токен (в т.ч. токен ServiceAccount), OIDC, webhook.
2. **Авторизация** — разрешено ли действие: обычно режимы `Node` и `RBAC` (флаг `--authorization-mode`).
3. **Mutating admission** — встроенные плагины (например, `DefaultStorageClass`, `ServiceAccount`) и mutating-вебхуки могут изменить объект (подставить sidecar, дефолты).
4. **Валидация схемы** объекта.
5. **Validating admission** — встроенные плагины (`ResourceQuota`, `PodSecurity`), validating-вебхуки и `ValidatingAdmissionPolicy` (правила на CEL прямо в API, без внешнего вебхука).
6. Запись в **etcd**. Дальше контроллеры и scheduler узнают об изменении через watch.

Проверить свои права можно так: `kubectl auth can-i create deployments -n prod`.

</details>

40. Какие бывают фазы пода и как на них влияет `restartPolicy`?

<details>
  <summary>Ответ</summary>

Фазы (`status.phase`): `Pending` (принят, но не все контейнеры запущены: ждёт планирования, скачивания образа, томов), `Running` (привязан к ноде, хотя бы один контейнер работает), `Succeeded` (все контейнеры завершились с кодом 0 и не будут перезапущены), `Failed` (все завершились, хотя бы один — с ошибкой), `Unknown` (нет связи с нодой).

`CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating` — это не фазы, а причины (`reason`) состояния контейнера `waiting`.

`restartPolicy` задаётся на уровне пода:
- `Always` (по умолчанию, единственный вариант для Deployment/StatefulSet/DaemonSet);
- `OnFailure` — перезапуск только при ненулевом коде выхода (типично для Job);
- `Never`.

Перезапуски выполняет kubelet на той же ноде с экспоненциальной задержкой (10s, 20s, 40s … до 5 минут), под при этом на другую ноду не переезжает.

</details>

41. Какие типичные ошибки допускают при настройке liveness/readiness/startup проб?

<details>
  <summary>Ответ</summary>

- Liveness проверяет внешние зависимости (БД, другой сервис): при их падении kubelet перезапускает все поды — каскадный отказ. Liveness должна проверять только сам процесс.
- Одинаковые liveness и readiness: под не успевает выйти из балансировки и сразу перезапускается.
- Слишком маленькие `timeoutSeconds` (по умолчанию 1с) и `failureThreshold` — перезапуски под нагрузкой или во время GC.
- Медленный старт без startupProbe: liveness убивает приложение раньше, чем оно поднялось. Правильно — `startupProbe` с `failureThreshold * periodSeconds` больше максимального времени старта.
- Отсутствие readinessProbe: трафик идёт в ещё не готовый под при rollout.
- Тяжёлая проба (exec со скриптом) раз в секунду — лишняя нагрузка на ноду.

```yaml
startupProbe:
  httpGet: {path: /healthz, port: 8080}
  failureThreshold: 30
  periodSeconds: 10
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
readinessProbe:
  httpGet: {path: /ready, port: 8080}
```

</details>

42. Чем отличаются requests и limits? Какие существуют QoS-классы?

<details>
  <summary>Ответ</summary>

- **requests** — сколько ресурсов гарантировано контейнеру; scheduler размещает под только на ноду, где сумма requests помещается в `allocatable`. Для CPU requests также определяют вес (`cpu.weight` в cgroup v2) при конкуренции.
- **limits** — верхняя граница: при превышении memory limit контейнер убивается OOM killer'ом, CPU limit приводит к троттлингу.

QoS-класс вычисляется автоматически (`kubectl get pod <pod> -o jsonpath='{.status.qosClass}'`):
- **Guaranteed** — у каждого контейнера заданы CPU и memory, и requests = limits;
- **Burstable** — хотя бы у одного контейнера есть request или limit, но условия Guaranteed не выполнены;
- **BestEffort** — ни requests, ни limits не заданы.

QoS влияет на `oom_score_adj` и на порядок выселения при нехватке ресурсов на ноде (первыми страдают BestEffort).

</details>

43. Что означает статус `OOMKilled` и чем он отличается от CPU throttling?

<details>
  <summary>Ответ</summary>

**OOMKilled** (код выхода 137 = 128 + SIGKILL) — процесс контейнера превысил memory limit своей cgroup, и ядро убило его. Память — несжимаемый ресурс, поэтому процесс именно убивается. Диагностика:
```
kubectl describe pod <pod>   # Last State: Terminated, Reason: OOMKilled
kubectl top pod <pod> --containers
```
Решения: поднять limit, найти утечку, для JVM/Go учитывать лимит контейнера (`-XX:MaxRAMPercentage`, `GOMEMLIMIT`).

Если OOM произошёл на уровне ноды (не хватило памяти всей ноде), kubelet может выселить поды — это уже Evicted.

**CPU throttling** — CPU сжимаемый ресурс: при достижении CPU limit процесс не убивается, а ограничивается CFS quota (`cpu.max` в cgroup v2) и ждёт следующего периода (100 мс). Проявляется ростом latency. Метрика — `container_cpu_cfs_throttled_periods_total`. Часто для latency-чувствительных сервисов CPU limit не задают, оставляя только requests.

</details>

44. Что такое нативные sidecar-контейнеры и чем они лучше обычного второго контейнера в поде?

<details>
  <summary>Ответ</summary>

Нативный sidecar — это init-контейнер с `restartPolicy: Always` (включено по умолчанию с 1.29, стабильно с 1.33):
```yaml
spec:
  initContainers:
  - name: log-shipper
    image: fluent/fluent-bit
    restartPolicy: Always
  containers:
  - name: app
    image: myapp
```
Отличия от обычного контейнера в `containers`:
- стартует **до** основных контейнеров (после того как пройдёт его startupProbe), значит прокси/агент уже готов, когда стартует приложение;
- останавливается **после** основных контейнеров;
- не мешает завершению Job: раньше sidecar в Job продолжал работать и под никогда не переходил в `Succeeded`;
- поддерживает пробы и перезапускается независимо.

Используется для service mesh прокси, сборщиков логов, агентов секретов.

</details>

45. Как происходит корректное завершение пода и как добиться zero-downtime при rollout?

<details>
  <summary>Ответ</summary>

При удалении пода параллельно происходят две вещи:
1. Под помечается `Terminating`, и его адрес убирается из EndpointSlice — kube-proxy/Ingress-контроллеры перестают слать новый трафик (с задержкой).
2. kubelet выполняет `preStop` хук, затем шлёт SIGTERM процессу PID 1 контейнера. По истечении `terminationGracePeriodSeconds` (по умолчанию 30с, отсчёт включает preStop) — SIGKILL.

Из-за гонки между п.1 и п.2 приложение может получить запросы уже после SIGTERM. Типичные меры:
```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        sleep:
          seconds: 10
```
(действие `sleep` в preStop есть в новых версиях; раньше использовали `exec: command: ["sleep","10"]`). Плюс: приложение должно обрабатывать SIGTERM (дорабатывать активные запросы), PID 1 должен пробрасывать сигналы (tini/dumb-init или exec-форма CMD), обязательна readinessProbe и PDB.

</details>

46. Какие стратегии обновления есть у Deployment и как откатить неудачный релиз?

<details>
  <summary>Ответ</summary>

- `RollingUpdate` (по умолчанию) — новый ReplicaSet поднимается, старый сворачивается. Параметры: `maxSurge` (сколько подов сверх `replicas` можно создать) и `maxUnavailable` (сколько может быть недоступно), по умолчанию оба 25%.
- `Recreate` — сначала удаляются все старые поды, потом создаются новые (даунтайм, но нет одновременной работы двух версий).

```
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=3
kubectl rollout pause|resume deploy/web
```
`progressDeadlineSeconds` (600с) задаёт, когда rollout считается зависшим, `revisionHistoryLimit` (10) — сколько старых ReplicaSet хранить.

Canary и blue-green встроенными средствами Deployment не делаются — для них используют Argo Rollouts, Flagger, веса в Gateway API/service mesh или два Deployment за одним Service.

</details>

47. Чем отличаются Job и CronJob? Какие у них важные параметры?

<details>
  <summary>Ответ</summary>

**Job** запускает поды до успешного завершения нужного числа раз:
- `completions`, `parallelism`, `completionMode: Indexed` (каждый под получает свой индекс);
- `backoffLimit` (по умолчанию 6) — число повторов, `activeDeadlineSeconds` — общий таймаут;
- `podFailurePolicy` — правила по кодам выхода (например, не повторять при коде 42);
- `ttlSecondsAfterFinished` — автоудаление завершённого Job;
- `restartPolicy` пода — только `OnFailure` или `Never`.

**CronJob** создаёт Job по расписанию:
- `schedule` (cron-формат), `timeZone: "Europe/Moscow"`;
- `concurrencyPolicy`: `Allow` (по умолчанию), `Forbid`, `Replace`;
- `startingDeadlineSeconds` — сколько можно опоздать с запуском;
- `successfulJobsHistoryLimit` (3) и `failedJobsHistoryLimit` (1).

Ручной запуск: `kubectl create job manual-1 --from=cronjob/backup`.

</details>

48. Что такое static pods? Где лежат их манифесты?

<details>
  <summary>Ответ</summary>

Static pod — под, которым управляет напрямую kubelet конкретной ноды, а не API-сервер. kubelet читает манифесты из каталога `staticPodPath` (в kubeadm-кластерах `/etc/kubernetes/manifests`) и запускает поды сам, даже если API-сервер недоступен. Так в kubeadm запускаются kube-apiserver, kube-controller-manager, kube-scheduler и etcd.

В API kubelet создаёт «зеркальный» под (mirror pod) с суффиксом имени ноды, например `kube-apiserver-master1`. Удалить его через `kubectl delete` нельзя — он пересоздастся. Чтобы остановить static pod, нужно убрать манифест из каталога; чтобы изменить — отредактировать файл, kubelet перезапустит под.

Посмотреть путь: `grep staticPodPath /var/lib/kubelet/config.yaml`.

</details>

49. Какие режимы работы есть у kube-proxy и чем они отличаются?

<details>
  <summary>Ответ</summary>

kube-proxy следит за Service и EndpointSlice и программирует на каждой ноде правила ядра для трансляции ClusterIP/NodePort в IP подов.
- **iptables** — режим по умолчанию. Для каждого сервиса и эндпоинта создаются цепочки в таблице nat; балансировка случайная (`statistic`). На десятках тысяч сервисов обновление правил и линейный поиск становятся медленными.
- **IPVS** — балансировщик L4 в ядре на хеш-таблицах, поддерживает алгоритмы rr, lc, sh и др. Лучше масштабировался, чем iptables, но требует модулей ipvs и всё равно использует iptables/ipset для части функций.
- **nftables** — новый режим (стабилен с 1.33), использует verdict maps nftables: быстрые инкрементальные обновления и масштабирование. Рекомендуется для новых кластеров на современных ядрах; в будущем планируется сделать его режимом по умолчанию.

Режим задаётся в `KubeProxyConfiguration.mode` (ConfigMap `kube-proxy` в `kube-system`). Cilium и некоторые другие CNI могут полностью заменить kube-proxy своей eBPF-реализацией.

</details>

50. Что такое EndpointSlice и зачем он заменил Endpoints?

<details>
  <summary>Ответ</summary>

Объект `Endpoints` хранил все адреса бэкендов сервиса в одном объекте. При тысячах подов каждое изменение одного пода приводило к пересылке огромного объекта всем kube-proxy кластера — нагрузка на API и etcd, лимит размера объекта.

`EndpointSlice` (`discovery.k8s.io/v1`) делит бэкенды на куски, по умолчанию до 100 адресов в срезе (`--max-endpoints-per-slice`, максимум 1000). Изменение затрагивает только один срез. Также хранит состояния `ready`/`serving`/`terminating`, зону и имя ноды (нужно для topology-aware routing), поддерживает dual-stack.

```
kubectl get endpointslices -l kubernetes.io/service-name=web
```
С 1.33 API v1 Endpoints объявлен устаревшим; kube-proxy и контроллеры работают через EndpointSlice.

</details>

51. Как работает Service типа LoadBalancer и что даёт `externalTrafficPolicy: Local`?

<details>
  <summary>Ответ</summary>

Service `LoadBalancer` — это NodePort + ClusterIP, плюс cloud-controller-manager (или MetalLB/Cilium LB-IPAM/kube-vip на bare metal) создаёт внешний балансировщик и записывает его адрес в `status.loadBalancer`.

`externalTrafficPolicy`:
- `Cluster` (по умолчанию) — трафик, пришедший на любую ноду, может уйти в под на другой ноде. Лишний сетевой хоп, и из-за SNAT под видит IP ноды вместо IP клиента.
- `Local` — трафик обслуживается только подами на той ноде, куда пришёл. Сохраняется исходный IP клиента, нет лишнего хопа. Ноды без подов не проходят health-check балансировщика (`healthCheckNodePort`), но при неравномерном размещении подов нагрузка распределяется неравномерно.

Аналогичная настройка для внутреннего трафика — `internalTrafficPolicy: Local`.

</details>

52. Чем Gateway API отличается от Ingress? Что сейчас стоит использовать?

<details>
  <summary>Ответ</summary>

**Ingress** — простой ресурс для HTTP(S)-маршрутизации по host/path. Всё, что сложнее (canary-веса, заголовки, таймауты, rewrite), задаётся аннотациями, которые у каждого контроллера свои, т.е. непереносимы. Нет TCP/UDP/gRPC как первоклассных сущностей.

**Gateway API** (`gateway.networking.k8s.io`, GA с 2023 года) — ролевая модель:
- `GatewayClass` — тип реализации (задаёт провайдер инфраструктуры);
- `Gateway` — конкретная точка входа: слушатели, порты, TLS (платформенная команда);
- `HTTPRoute`, `GRPCRoute`, `TLSRoute` и др. — маршруты (команды приложений), в т.ч. из других namespace с разрешения через `ReferenceGrant`/`allowedRoutes`.

Разделение по весам, сопоставление по заголовкам, зеркалирование трафика — в стандартной спецификации.

**Актуально на 2026:** проект ingress-nginx (kubernetes/ingress-nginx) выведен из поддержки — обновления, в том числе по безопасности, прекратились в марте 2026 года. Для новых установок рекомендуется Gateway API (реализации: Envoy Gateway, Cilium, Istio, NGINX Gateway Fabric, Traefik, Kong и др.) либо другой поддерживаемый Ingress-контроллер.

</details>

53. Что такое CNI? Чем отличаются Flannel, Calico и Cilium?

<details>
  <summary>Ответ</summary>

CNI (Container Network Interface) — спецификация плагинов, которые container runtime вызывает при создании/удалении sandbox пода: создать veth, выдать IP (IPAM), настроить маршруты. Конфиги лежат в `/etc/cni/net.d/`, бинарники — в `/opt/cni/bin/`. Модель сети Kubernetes: у каждого пода свой IP, поды общаются друг с другом без NAT.

- **Flannel** — простой overlay (VXLAN), без поддержки NetworkPolicy.
- **Calico** — маршрутизация L3 с BGP или overlay IPIP/VXLAN, полноценные NetworkPolicy и свои расширенные политики; есть dataplane на iptables, nftables и eBPF.
- **Cilium** — dataplane на eBPF: может заменить kube-proxy, политики на уровне L3–L7 (HTTP, DNS), прозрачное шифрование (WireGuard/IPsec), наблюдаемость через Hubble, Cluster Mesh для мультикластера.

Если сеть не работает — под висит в `ContainerCreating` с ошибкой `failed to setup network for sandbox`, нода может быть `NotReady` с `NetworkPluginNotReady`.

</details>

54. Как работают NetworkPolicy? Как сделать политику «запретить всё по умолчанию»?

<details>
  <summary>Ответ</summary>

По умолчанию все поды могут общаться со всеми. NetworkPolicy — это белый список: как только под выбран хотя бы одной политикой с типом `Ingress` (или `Egress`), для него разрешено только то, что явно указано во всех применимых политиках (правила складываются). Политики применяет CNI-плагин; если он их не поддерживает (Flannel), объекты создаются, но ничего не делают.

Запрет всего входящего и исходящего трафика в namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```
После этого нужно явно разрешить DNS (UDP/TCP 53 к подам CoreDNS в `kube-system`), иначе перестанут резолвиться имена. Селекторы в правилах: `podSelector`, `namespaceSelector` (например, по метке `kubernetes.io/metadata.name`), `ipBlock`. Для соединения нужно разрешение и на egress источника, и на ingress получателя.

</details>

55. Как работает DNS внутри кластера и в чём проблема `ndots:5`?

<details>
  <summary>Ответ</summary>

DNS обслуживает CoreDNS (Service `kube-dns` в `kube-system`). kubelet прописывает в `/etc/resolv.conf` пода адрес этого сервиса, search-домены `<ns>.svc.cluster.local svc.cluster.local cluster.local` и `options ndots:5`. Имена: `<svc>.<ns>.svc.cluster.local`, для подов headless-сервиса — `<pod>.<svc>.<ns>.svc.cluster.local`.

`ndots:5` означает: если в имени меньше 5 точек, резолвер сначала перебирает все search-домены. Запрос `api.example.com` превращается в 4+ запроса (по A и AAAA), большинство из которых получают NXDOMAIN — лишняя нагрузка на CoreDNS и задержки.

Решения: писать внешние FQDN с точкой в конце (`api.example.com.`), снизить `ndots` через `dnsConfig.options` пода, установить NodeLocal DNSCache.

Отладка: `kubectl run -it --rm dns --image=busybox:1.36 -- nslookup kubernetes.default`.

</details>

56. Как связаны PV, PVC, StorageClass и CSI?

<details>
  <summary>Ответ</summary>

- **PersistentVolume** — объект, описывающий реальный том (диск облака, NFS, Ceph RBD). Cluster-scoped.
- **PersistentVolumeClaim** — запрос приложения на том: размер, `accessModes` (`ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, `ReadWriteOncePod`), класс. Живёт в namespace, под ссылается на PVC.
- **StorageClass** — шаблон для динамического создания PV: `provisioner` (CSI-драйвер), `parameters`, `reclaimPolicy` (`Delete` по умолчанию или `Retain`), `allowVolumeExpansion`, `volumeBindingMode`.
- **CSI** — стандартный интерфейс драйверов хранилищ; драйвер состоит из controller-части (создание/подключение томов) и node-плагина (DaemonSet, монтирование на ноде).

`volumeBindingMode: WaitForFirstConsumer` откладывает создание тома до планирования пода — иначе диск может быть создан в зоне, где под не сможет запуститься.

```
kubectl get pvc,pv
kubectl describe pvc data-db-0   # события provisioner'а
```

</details>

57. Как передать ConfigMap/Secret в под и обновится ли конфигурация без перезапуска?

<details>
  <summary>Ответ</summary>

Способы: переменные окружения (`env.valueFrom`, `envFrom`) или файлы через том (`volumes.configMap`/`volumes.secret`).

- **Переменные окружения** читаются только при старте контейнера — после изменения ConfigMap нужен перезапуск пода.
- **Смонтированный том** kubelet обновляет автоматически (с задержкой до минуты — период синхронизации плюс кеш), но приложение должно само перечитать файл.
- **subPath**-монтирование не обновляется никогда.
- ConfigMap/Secret с `immutable: true` изменить нельзя, зато они снижают нагрузку на API (kubelet не следит за ними).

Перезапуск после изменения: `kubectl rollout restart deploy/web`. Для автоматики — хэш конфига в аннотации шаблона пода (Helm: `checksum/config`) или Stakater Reloader.

Лимит размера ConfigMap/Secret — 1 МиБ.

</details>

58. Насколько безопасны Secret в Kubernetes и как их защищают?

<details>
  <summary>Ответ</summary>

Данные в Secret закодированы base64, это не шифрование. По умолчанию они хранятся в etcd открытым текстом, их может прочитать любой, у кого есть `get`/`list` на secrets в namespace или право создать под, монтирующий этот Secret.

Меры:
- **шифрование at rest**: `--encryption-provider-config` на kube-apiserver с `EncryptionConfiguration` (`aescbc`/`secretbox` или лучше KMS v2 с ключом во внешнем KMS);
- строгий RBAC, особенно на `list`/`watch` secrets (list отдаёт содержимое);
- не хранить секреты в Git открыто: **Sealed Secrets**, **SOPS** (с age/KMS), **External Secrets Operator** (синхронизирует из Vault, AWS Secrets Manager, Yandex Lockbox и т.п. в обычный Secret), **Secrets Store CSI Driver** (монтирует секрет файлом из внешнего хранилища);
- защищать бэкапы etcd, включить audit log.

Проверить, что секрет зашифрован в etcd: `etcdctl get /registry/secrets/<ns>/<name> | hexdump -C` — должен быть префикс `k8s:enc:`.

</details>

59. Как устроен RBAC и как работают токены ServiceAccount в современных версиях?

<details>
  <summary>Ответ</summary>

RBAC разрешающий (запрещающих правил нет):
- `Role` (в namespace) / `ClusterRole` (на кластер или для переиспользования) — набор правил `apiGroups`/`resources`/`verbs`;
- `RoleBinding` / `ClusterRoleBinding` — привязка роли к субъекту: User, Group, ServiceAccount. RoleBinding может ссылаться на ClusterRole — права получаются только в его namespace.

```
kubectl create sa ci -n build
kubectl create rolebinding ci-edit --clusterrole=edit --serviceaccount=build:ci -n build
kubectl auth can-i --list --as=system:serviceaccount:build:ci -n build
```

Токены ServiceAccount: с 1.24 долгоживущие токены в Secret больше не создаются автоматически. Под получает короткоживущий токен, привязанный к поду и аудитории, через projected volume (TokenRequest API), kubelet его ротирует. Разовый токен: `kubectl create token ci -n build --duration=1h`. Если под не обращается к API, лучше отключить монтирование: `automountServiceAccountToken: false`.

</details>

60. Что пришло на смену PodSecurityPolicy и как им пользоваться?

<details>
  <summary>Ответ</summary>

PodSecurityPolicy удалён в 1.25. Встроенная замена — **Pod Security Admission** (PSA), который проверяет поды по стандартам Pod Security Standards:
- `privileged` — без ограничений;
- `baseline` — запрещает очевидные эскалации (privileged, hostNetwork/hostPID, hostPath и т.д.);
- `restricted` — жёсткий профиль: `runAsNonRoot`, запрет эскалации привилегий, `drop: [ALL]` capabilities, seccomp `RuntimeDefault`.

Включается метками на namespace, режимы `enforce` (отклонить), `audit` (записать в audit log), `warn` (предупреждение пользователю):
```
kubectl label ns prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```
Проверить заранее: `kubectl label --dry-run=server --overwrite ns prod pod-security.kubernetes.io/enforce=restricted`.

Для более гибких правил (обязательные метки, разрешённые реестры, лимиты) используют Kyverno, OPA Gatekeeper или встроенный `ValidatingAdmissionPolicy`.

</details>

61. Какой `securityContext` считается хорошей практикой для контейнера?

<details>
  <summary>Ответ</summary>

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```
- `runAsNonRoot` — kubelet откажется запускать контейнер от UID 0;
- `allowPrivilegeEscalation: false` — выставляет `no_new_privs`, setuid-бинарники не дадут root;
- `readOnlyRootFilesystem` — для записи монтируются `emptyDir`;
- `drop: ["ALL"]` и при необходимости `add` только нужных (например, `NET_BIND_SERVICE`);
- seccomp `RuntimeDefault` — фильтр системных вызовов runtime.

Также: не использовать `privileged: true`, `hostPath`, `hostNetwork` без необходимости. Эти настройки соответствуют профилю `restricted` Pod Security Standards.

</details>

62. Что такое PriorityClass и preemption?

<details>
  <summary>Ответ</summary>

`PriorityClass` — cluster-scoped объект с числовым приоритетом, под ссылается на него через `priorityClassName`. Если под с высоким приоритетом не помещается ни на одну ноду, scheduler может **вытеснить** (preempt) поды с меньшим приоритетом, чтобы освободить место. Приоритет также учитывается kubelet при выселении при нехватке ресурсов.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: business-critical
value: 100000
preemptionPolicy: PreemptLowerPriority   # или Never
globalDefault: false
```
Встроенные классы `system-cluster-critical` и `system-node-critical` предназначены для компонентов кластера (CoreDNS, CNI и т.п.).

Приём с «балластом»: поды-заглушки с отрицательным приоритетом резервируют место на нодах и вытесняются первыми при появлении реальной нагрузки (overprovisioning для autoscaler).

</details>

63. Для чего нужны ResourceQuota и LimitRange?

<details>
  <summary>Ответ</summary>

**ResourceQuota** ограничивает суммарное потребление в namespace: `requests.cpu`, `limits.memory`, количество подов, PVC, сервисов LoadBalancer, объём хранилища по StorageClass и т.д. Если квота на ресурсы задана, под без соответствующих requests/limits будет отклонён.

**LimitRange** задаёт правила для отдельных объектов в namespace: requests/limits по умолчанию (`default`, `defaultRequest`), а также min/max на контейнер или PVC.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
  - type: Container
    defaultRequest: {cpu: 100m, memory: 128Mi}
    default: {memory: 256Mi}
```
Использование квоты: `kubectl describe resourcequota -n team-a`. Вместе они используются для мультиарендных кластеров: команде выдаётся namespace с квотой, LimitRange страхует от подов без ресурсов.

</details>

64. Что такое PodDisruptionBudget и от чего он защищает?

<details>
  <summary>Ответ</summary>

PDB ограничивает количество одновременно недоступных подов приложения при **добровольных** прерываниях — вызовах Eviction API: `kubectl drain`, обновление нод, консолидация Karpenter/Cluster Autoscaler. От падения ноды или OOM PDB не защищает.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
spec:
  minAvailable: 2        # или maxUnavailable: 1
  selector:
    matchLabels: {app: web}
  unhealthyPodEvictionPolicy: AlwaysAllow
```
Частые ошибки: `minAvailable` равен числу реплик (или `maxUnavailable: 0`) — drain зависает навсегда; PDB для Deployment с одной репликой. `unhealthyPodEvictionPolicy: AlwaysAllow` разрешает выселять неготовые поды (например, в CrashLoopBackOff), чтобы они не блокировали обслуживание нод.

Проверка: `kubectl get pdb` — колонка `ALLOWED DISRUPTIONS`.

</details>

65. Что такое VPA и KEDA? Чем они дополняют HPA?

<details>
  <summary>Ответ</summary>

**VPA** (Vertical Pod Autoscaler, ставится отдельно) — подбирает requests контейнеров по истории потребления. Компоненты: recommender, updater, admission controller. Режимы `updateMode`: `Off` (только рекомендации — удобно для анализа), `Initial` (только при создании пода), `Recreate`, а в новых версиях — `InPlaceOrRecreate`, использующий in-place resize (изменение ресурсов пода без пересоздания, стабильно с 1.35). Нельзя одновременно масштабировать HPA и VPA по одной метрике (CPU/память).

**KEDA** — event-driven автоскейлинг: ресурс `ScaledObject` с триггерами (длина очереди Kafka/RabbitMQ, метрики Prometheus, cron и десятки других). KEDA сама создаёт и управляет HPA, умеет масштабировать в ноль и обратно; для разовых задач есть `ScaledJob`.

Итог: HPA — число реплик по CPU/памяти/кастомным метрикам, VPA — размер подов, KEDA — число реплик по внешним событиям с поддержкой нуля.

</details>

66. Чем отличаются Cluster Autoscaler и Karpenter?

<details>
  <summary>Ответ</summary>

Оба добавляют ноды, когда появляются поды в `Pending` из-за нехватки ресурсов, и удаляют недогруженные.

**Cluster Autoscaler** работает с заранее заданными группами нод (ASG, managed node groups, группы узлов облака): меняет их желаемый размер. Типы инстансов фиксированы в группе, для разнообразия нужно много групп. Поддерживает большинство облаков.

**Karpenter** не использует группы: по требованиям ожидающих подов (ресурсы, affinity, зоны, архитектура, spot/on-demand) сам выбирает подходящий тип инстанса и создаёт ноду напрямую через API облака. Настраивается ресурсами `NodePool` и `NodeClass` (например, `EC2NodeClass`). Умеет **консолидацию** — переупаковывает поды и заменяет ноды на более дешёвые, учитывает disruption budgets и PDB. Изначально AWS, есть провайдеры для Azure и других.

В обоих случаях корректные requests обязательны: автоскейлер планирует по ним, а не по фактической нагрузке.

</details>

67. Как безопасно вывести ноду на обслуживание?

<details>
  <summary>Ответ</summary>

```
kubectl cordon node1          # запретить планирование новых подов
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data --timeout=10m
# ... обслуживание / перезагрузка / обновление ...
kubectl uncordon node1
```
- `drain` выполняет cordon и выселяет поды через Eviction API, поэтому учитываются PDB — если бюджет исчерпан, drain ждёт.
- Поды DaemonSet не выселяются (`--ignore-daemonsets`), static pods тоже.
- Поды с `emptyDir` требуют флага `--delete-emptydir-data` — данные будут потеряны.
- «Голые» поды без контроллера не пересоздадутся — нужен `--force`, и они будут потеряны.
- Поды с локальными PV (local-path, local PV) не смогут запуститься на другой ноде.

Перед обслуживанием полезно проверить: `kubectl get pdb -A` и `kubectl get pods -A -o wide --field-selector spec.nodeName=node1`.

</details>

68. Чем отличаются Helm и Kustomize? Когда что использовать?

<details>
  <summary>Ответ</summary>

**Helm** — пакетный менеджер: чарт (шаблоны Go template + `values.yaml` + `Chart.yaml`) и релиз — установленный экземпляр чарта с историей ревизий. Состояние релиза хранится в кластере в Secret `sh.helm.release.v1.<release>.v<N>`. Есть зависимости, хуки, откат:
```
helm upgrade --install web ./chart -n prod -f values-prod.yaml --atomic
helm history web -n prod
helm rollback web 3 -n prod
helm template ./chart | kubectl diff -f -
```
В конце 2025 года вышел Helm 4 (улучшенная система плагинов, server-side apply); Helm 2 с Tiller давно не поддерживается.

**Kustomize** — без шаблонов: базовые манифесты (`base`) + оверлеи (`overlays/prod`) с патчами, `namePrefix`, `images`, `configMapGenerator`. Встроен в kubectl: `kubectl apply -k overlays/prod`. Нет понятия релиза и истории.

Helm удобен для распространения сторонних приложений с множеством параметров, Kustomize — для своих манифестов с небольшими различиями между окружениями. Их часто сочетают (Kustomize поверх вывода Helm), в т.ч. в Argo CD и Flux.

</details>

69. Что такое GitOps? Чем отличаются Argo CD и Flux?

<details>
  <summary>Ответ</summary>

GitOps — подход, при котором желаемое состояние кластера декларативно описано в Git, а агент внутри кластера постоянно сверяет его с фактическим и приводит к нему (pull-модель). Изменения — через merge request, откат — revert коммита, CI не нужен доступ к кластеру, дрейф (ручные правки) виден и исправляется автоматически.

**Argo CD**: ресурс `Application` (источник: Git/Helm/OCI, путь, целевой кластер и namespace), `ApplicationSet` для генерации множества приложений, `AppProject` для ограничений. Сильный веб-интерфейс, SSO, управление многими кластерами из одного экземпляра, sync waves и хуки, `selfHeal` и `prune`.

**Flux**: набор контроллеров (source, kustomize, helm, notification, image automation) и CRD `GitRepository`/`OCIRepository`, `Kustomization`, `HelmRelease`. Более «кубернетес-нативный», без обязательного UI, удобен для автоматического обновления образов в Git и для модели «каждый кластер тянет свою конфигурацию».

Секреты в GitOps хранят через SOPS, Sealed Secrets или External Secrets.

</details>

70. Что такое CRD, контроллер и finalizer? Почему namespace может зависнуть в `Terminating`?

<details>
  <summary>Ответ</summary>

**CRD** (CustomResourceDefinition) регистрирует в API новый тип ресурса со своей OpenAPI-схемой (например, `Certificate` у cert-manager). Сам по себе CRD только хранит объекты.

**Контроллер** (оператор) следит за объектами через watch/informer и в цикле **reconcile** приводит фактическое состояние к описанному в `spec`, записывая результат в `status`. Цикл должен быть идемпотентным. Фреймворки: controller-runtime/Kubebuilder, Operator SDK.

**Finalizer** — строка в `metadata.finalizers`. Пока список не пуст, объект при удалении получает `deletionTimestamp`, но не удаляется: контроллер успевает выполнить очистку (удалить облачный балансировщик, диск) и снимает finalizer.

Если контроллер удалён или сломан, finalizer никто не снимет — объект, а вместе с ним и namespace, висит в `Terminating`. Диагностика:
```
kubectl get ns stuck -o jsonpath='{.status.conditions}'
kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get -n stuck --ignore-not-found
```
Правильно — починить контроллер. Крайняя мера — снять finalizer вручную (`kubectl patch ... --type=merge -p '{"metadata":{"finalizers":null}}'`), понимая, что внешние ресурсы останутся.

</details>

71. Что такое CRI? Почему Kubernetes больше не использует Docker и как отлаживать контейнеры на ноде?

<details>
  <summary>Ответ</summary>

CRI (Container Runtime Interface) — gRPC-API между kubelet и container runtime. Основные реализации — **containerd** и **CRI-O**; они, в свою очередь, запускают контейнеры через низкоуровневый OCI-runtime (runc, crun, либо gVisor/Kata для песочниц через `RuntimeClass`).

Docker Engine не реализует CRI, поэтому kubelet работал с ним через прослойку dockershim, которую удалили в 1.24. Образы, собранные Docker, по-прежнему работают — это стандартные OCI-образы. Если нужен именно Docker Engine, используют внешний адаптер cri-dockerd.

На ноде вместо `docker` используют `crictl` (конфиг `/etc/crictl.yaml`):
```
crictl ps -a
crictl pods
crictl logs <container-id>
crictl inspect <container-id>
crictl images; crictl pull nginx:1.27
```
У containerd также есть `ctr` (образы Kubernetes в namespace `k8s.io`: `ctr -n k8s.io images ls`) и `nerdctl` с Docker-подобным интерфейсом.

</details>

72. Как обновить кластер, развёрнутый kubeadm? Какие есть ограничения по версиям?

<details>
  <summary>Ответ</summary>

Правила: обновление только на **одну минорную версию** за раз (1.33 → 1.34 → 1.35), сначала control plane, потом worker-ноды. kubelet может отставать от kube-apiserver не более чем на 3 минорные версии, но не может быть новее него. Перед обновлением: бэкап etcd, проверка удалённых API (`kubectl deprecations`/pluto/kubent), чтение changelog.

1. Первая control plane нода: обновить пакет `kubeadm` (сменить репозиторий `pkgs.k8s.io` на нужную минорную версию), затем
```
kubeadm upgrade plan
kubeadm upgrade apply v1.34.x
```
2. Остальные control plane ноды: `kubeadm upgrade node`.
3. На каждой ноде поочерёдно: `kubectl drain`, обновить `kubelet` и `kubectl`, `systemctl daemon-reload && systemctl restart kubelet`, `kubectl uncordon`. Для worker — перед этим `kubeadm upgrade node`.
4. Обновить CNI и аддоны согласно их совместимости.

kubeadm при `upgrade apply` также продлевает сертификаты control plane (проверить: `kubeadm certs check-expiration`).

</details>

73. Как сделать резервную копию etcd и восстановить её?

<details>
  <summary>Ответ</summary>

Снапшот (на control plane ноде kubeadm-кластера):
```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%F).db
etcdutl snapshot status /backup/etcd-2026-09-01.db -w table
```
Восстановление (в etcd 3.6 команда `etcdctl snapshot restore` удалена, используется `etcdutl`):
```
etcdutl snapshot restore /backup/etcd.db --data-dir /var/lib/etcd-restore
```
Затем остановить kube-apiserver и etcd (убрать их манифесты из `/etc/kubernetes/manifests`), в манифесте etcd указать новый `hostPath` для data-dir, вернуть манифесты. В кластере из нескольких членов etcd восстанавливают каждый член из одного снапшота с новым `--initial-cluster`.

Бэкап содержит все Secret — его нужно шифровать и хранить вне кластера. Данные PV в etcd не входят — для них Velero/снапшоты CSI.

</details>

74. Под в статусе `CrashLoopBackOff`. Как искать причину?

<details>
  <summary>Ответ</summary>

CrashLoopBackOff означает, что контейнер запускается и падает, а kubelet перезапускает его с растущей задержкой.
```
kubectl describe pod <pod>          # Last State, Exit Code, Reason, Events
kubectl logs <pod> -c <container> --previous
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState}'
```
Интерпретация кода выхода:
- `1` и прочие — ошибка приложения: неверный конфиг, нет переменной окружения/секрета, недоступна БД;
- `137` — SIGKILL: OOMKilled или убит по liveness-пробе (смотреть события `Liveness probe failed`);
- `139` — segfault;
- `0` с `restartPolicy: Always` — процесс просто завершился (например, запущен в фоне или команда не та).

Если логов нет — проверить `command`/`args`, права на файлы, `readOnlyRootFilesystem`. Для отладки можно запустить копию пода с другой командой: `kubectl debug <pod> -it --copy-to=dbg --container=app -- sh`.

</details>

75. Под в статусе `ImagePullBackOff`/`ErrImagePull`. Какие причины и как проверить?

<details>
  <summary>Ответ</summary>

Смотреть события: `kubectl describe pod <pod>` — сообщение от kubelet обычно прямо называет причину.

Типичные причины:
- опечатка в имени образа или тега, тег удалён (`not found`/`manifest unknown`);
- приватный реестр без `imagePullSecrets` или с неверным секретом (`unauthorized`, `401`/`403`);
- секрет в другом namespace (должен быть в namespace пода) или не привязан к ServiceAccount;
- лимиты реестра (Docker Hub rate limit, `429 Too Many Requests`);
- нет сетевого доступа с ноды к реестру, прокси, DNS, самоподписанный сертификат реестра;
- образ не для архитектуры ноды (`no match for platform`).

```
kubectl create secret docker-registry regcred --docker-server=registry.example.com \
  --docker-username=ci --docker-password=... -n prod
```
Проверить с ноды: `crictl pull registry.example.com/app:1.2.3`. Рекомендуется использовать неизменяемые теги или digest (`image@sha256:...`) и зеркало/pull-through cache реестра.

</details>

76. Под висит в `Pending`. Что проверить?

<details>
  <summary>Ответ</summary>

`kubectl describe pod <pod>` → Events, сообщение `FailedScheduling` от scheduler объясняет, почему не подошла каждая группа нод, например `0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had untolerated taint`.

Причины:
- не хватает ресурсов по **requests** (`Insufficient cpu/memory`) — `kubectl describe node` → Allocated resources;
- taints без tolerations, неудовлетворимые `nodeSelector`/`nodeAffinity`/`podAntiAffinity`/`topologySpreadConstraints` с `DoNotSchedule`;
- PVC не привязан (нет StorageClass, том в другой зоне, `WaitForFirstConsumer` ждёт) — `kubectl get pvc`;
- превышена ResourceQuota (тогда пода вообще нет, ошибка в событиях ReplicaSet);
- нет портов при `hostPort`, ноды cordoned;
- все ноды NotReady.

Если событий нет совсем — проверить, работает ли kube-scheduler и не задан ли у пода `schedulerName` несуществующего планировщика. При наличии Cluster Autoscaler/Karpenter в событиях видно, пытается ли он добавить ноду.

</details>

77. Почему поды получают статус `Evicted` и как это предотвратить?

<details>
  <summary>Ответ</summary>

Evicted — под выселен kubelet'ом из-за нехватки ресурсов на ноде (node-pressure eviction) или через Eviction API (drain, preemption). Для первого случая пороги задаются в конфиге kubelet (`evictionHard`, по умолчанию, например, `memory.available<100Mi`, `nodefs.available<10%`, `imagefs.available<15%`). Нода получает condition `MemoryPressure`/`DiskPressure`/`PIDPressure` и taint, новые поды туда не планируются.

Порядок выселения: сначала поды, у которых потребление превышает requests, затем по приоритету (PriorityClass), затем по превышению потребления над requests. Поэтому корректные requests (и QoS Guaranteed для критичных подов) снижают риск.

```
kubectl get pods -A --field-selector=status.phase=Failed
kubectl describe node <node> | grep -A5 Conditions
kubectl delete pods -A --field-selector=status.phase=Failed   # очистка
```
Профилактика: лимиты на `ephemeral-storage`, ротация логов контейнеров (`containerLogMaxSize`), чистка образов, `systemReserved`/`kubeReserved` в kubelet, мониторинг диска `/var/lib/containerd` и `/var/lib/kubelet`.

</details>

78. Нода перешла в `NotReady`. Как диагностировать?

<details>
  <summary>Ответ</summary>

NotReady означает, что kubelet не обновляет статус/lease ноды (объекты `Lease` в `kube-node-lease`) или сам сообщает о проблеме.
```
kubectl describe node <node>    # Conditions: Ready, MemoryPressure, DiskPressure, NetworkUnavailable
```
На самой ноде:
```
systemctl status kubelet containerd
journalctl -u kubelet --since "30 min ago"
crictl ps -a
df -h; free -m
```
Частые причины: kubelet остановлен или падает (ошибка конфига, истёк клиентский сертификат kubelet), упал container runtime, не работает CNI (`NetworkPluginNotReady`), закончился диск или память, нет сети до API-сервера, расхождение времени (NTP) ломает TLS.

Через ~5 минут (`tolerationSeconds: 300` для taint `node.kubernetes.io/not-ready`/`unreachable`) поды будут выселены и пересозданы на других нодах; поды StatefulSet на недоступной ноде автоматически не пересоздаются, пока нода не подтверждена как выключенная.

</details>

79. Как отладить под, в образе которого нет shell и утилит?

<details>
  <summary>Ответ</summary>

Использовать **эфемерные контейнеры** через `kubectl debug` — во время работы в под добавляется временный контейнер с нужным образом, без перезапуска:
```
kubectl debug -it pod/web --image=busybox:1.36 --target=app
```
`--target` подключает отладочный контейнер к пространству имён процессов указанного контейнера — видны его процессы, файловая система доступна через `/proc/<pid>/root`. Сеть у всех контейнеров пода общая.

Другие варианты:
- копия пода с изменённой командой/образом: `kubectl debug pod/web -it --copy-to=web-dbg --container=app --image=ubuntu -- bash`;
- отладка ноды: `kubectl debug node/node1 -it --image=ubuntu` — под в host-namespaces, корень ноды смонтирован в `/host`;
- профили прав: `--profile=general|baseline|restricted|netadmin|sysadmin`.

Эфемерный контейнер нельзя удалить из пода — он останется до пересоздания пода.

</details>
