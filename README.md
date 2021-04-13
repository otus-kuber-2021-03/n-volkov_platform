# n-volkov_platform
n-volkov Platform repository


## Знакомство с Kubernetes, основные понятия и архитектура

* Установлен и запущен Minikube.
* Настроено рабочее окружение.

При проведение тестов на устойчивость, на команду:

```
$ kubectl get componentstatuses
```

получил такой ответ:

```
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection r
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection r
etcd-0               Healthy     {"health":"true"}
```

Поправил:

```
$ minikube ssh
$ sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml
```

удалена строка конфигурации: `- --port=0`

```
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

удалена строка конфигурации: `- --port=0`

```
$ sudo systemctl restart kubelet.service
```

После проделанных действий получил:

```
$ kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

### Задание

* Разберитесь почему все pod в namespace kube-system восстановились после удаления.

#### Ответ

Pod coredns принадлежит (ownerReferences) контроллеру ReplicaSet.

```
$ kubectl get rs -n kube-system
NAME                DESIRED   CURRENT   READY   AGE
coredns-74ff55c5b   1         1         1       177m
```

Который в свою очередь принадлежит контроллеру Deployment. Данный контроллер ReplicaSet имеет, среди прочих, параметр:

```
spec:
  replicas: 1
```

Т.е. всегда должен быть активен один экземпляр pod-а coredns.

Pod kube-proxy принадлежит (ownerReferences) контроллеру DaemonSet. Который гарантирует, что на каждом узле кластера будет активен экземпляр управляемого им pod-а.

Остальные pod-ы: etcd-minikube, kube-apiserver-minikube, kube-controller-manager-minikube и kube-scheduler-minikube, принадлежат узлу (node), и управляются непосредственно kubelet. Kubelet следит за ними, и активирует вновь при падении.

* Был создан Dockerfile, который:
 * запускает web-сервер на порту 8000;
 * отдает содержимое директории /app внутри контейнера;
 * работающий с UID 1001;
* Dockerfile был помещен в созданную директорию `kubernetes-intro/web`;
* Из Dockerfile был собран образ контейнера и помещен в публичный Container Registry - Docker Hub;
* Был написан манифест web-pod.yaml для создания pod **web**, с использованием  ранее собранного образа с Docker Hub. Манифест был помещен в директорию `kubernetes-intro/`;
* Были опробованы следующие команды:
 * `$ kubectl apply -f web-pod.yaml`
 * `$ kubectl get pod web -o yaml`
 * `$ kubectl describe pod web`
* В манифест web-pod.yaml было добавлено описание init контейнера, генерирующего страницу index.html;
 * *image* init контейнера содержит wget;
 * *command* init контейнера: `['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh'];`
* Для того, чтобы файлы, созданные в init контейнере, были доступны основному контейнеру, в манифест было добавлено описание volume типа *emptyDir*;
* Обновленный pod **web** был запущен, после удаления старого pod:
 * следил за процессом запуска:  
`$ kubectl get pods -w`;
 * проверил работоспособность web сервера:  
`$ kubectl port-forward --address 0.0.0.0 pod/web 8000:8000`  
`$ curl 'http://localhost:8000/index.html'`

* Знакомство с микросервисным приложением Hipster Shop;
* Запуск внутри кластера одного его компонента - микросервиса frontend;
 * был сделан клон репозитория `https://github.com/GoogleCloudPlatform/microservices-demo`;
 * собран образ контейнера для frontend из `microservices-demo/src/frontend/Dockerfile`;
 * собранный образ был помещен на Docker Hub;
 * используя ad-hoc режим, был сгенерирован манифест средствами kubectl:  
`$ kubectl run frontend --image blackwolf292/kubernetes-intro-frontend:v1 --restart=Never --dry-run=client -o yaml > frontend-pod.yaml`


### Задание

* Выясните причину, по которой pod frontend находится в статусе Error.
* Создайте новый манифест frontend-pod-healthy.yaml. При его применении ошибка должна исчезнуть.

#### Ответ

Попытка запуска pod frontend, завершилась ошибкой. Найденная причина - отсутствие описания environment variable в конфигурации контейнера:

```
$ kubectl apply -f frontend-pod.yaml
$ kubectl get pods -w
NAME       READY   STATUS              RESTARTS   AGE
frontend   0/1     ContainerCreating   0          18s
frontend   0/1     Error               0          20s

$ kubectl logs frontend
.....
panic: environment variable "CHECKOUT_SERVICE_ADDR" not set
.....
```

Для устранения ошибки в манифест, в секцию spec, было добавлено:

```
    env:
    - name: PRODUCT_CATALOG_SERVICE_ADDR
      value: "productcatalogservice:3550"
    - name: CURRENCY_SERVICE_ADDR
      value: "currencyservice:7000"
    - name: CART_SERVICE_ADDR
      value: "cartservice:7070"
    - name: RECOMMENDATION_SERVICE_ADDR
      value: "recommendationservice:8080"
    - name: CHECKOUT_SERVICE_ADDR
      value: "checkoutservice:5050"
    - name: SHIPPING_SERVICE_ADDR
      value: "shippingservice:50051"
    - name: AD_SERVICE_ADDR
      value: "adservice:9555"
```

Новый манифест был сохранен как frontend-pod-healthy.yaml. Проверен:

```
$ kubectl apply -f frontend-pod-healthy.yaml
$ kubectl get pods -n default
NAME       READY   STATUS    RESTARTS   AGE
frontend   1/1     Running   0          27s
```

И помещен в директорию kubernetes-intro.

## Механика запуска и взаимодействия контейнеров в Kubernetes

* Используя frontend-replicaset.yaml, попытались создать объект ReplicaSet. Получили ошибку.

### Задание

Изменить frontend-replicaset.yaml так, что бы устранить ошибку. И применить его.

#### Ответ

```
error: error validating "frontend-replicaset.yaml":  
error validating data: ValidationError(ReplicaSet.spec): missing required field "selector"  
in io.k8s.api.apps.v1.ReplicaSetSpec
```

Добавил в манифест поле selector

```
  selector:
    matchLabels:
      app: frontend
```

* Изменили манифест frontend-replicaset.yaml так, что из него сразу разворачивается 3 реплики сервиса.

* Изменили версию образа в манифесте frontend-replicaset.yaml. Применили манифест.

### Вопрос

Почему обновление ReplicaSet (изменена версия образа контейнера) не повлекло обновление запущенных pod.

#### Ответ

ReplicaSet следит только за тем, что запущено указанное количество pod-ов. Т.к. указанное количество pod-ов уже было запущено, никаких других изменений не произошло.

* Сделал образы для микросервиса paymentService, и залил на Docker Hub. Сделал манифест paymentservice-replicaset.yaml для развертывания трех реплик из образа v1.
* Подготовил и применил Deployment манифест paymentservice-deployment.yaml. В результате чего появилось 3 реплики сервиса payment в состоянии Ready.
* Убедился, на смени версии образа в манифесте Deployment, что работает, как обновление pod, так и откат на предыдущую версию. Увидел, что при обновлении использовалась стратегия по умолчанию - Rolling Update, и наличие двух ReplicaSet.
* Подготовил манифест frontend-deployment.yaml с описанием readinessProbe. Из которого успешно развернул 3 экземпляра pod frontend.
* Изменил URL проверки readinessProbe на `/_health`. Получил ошибку при применении манифеста:  
`Readiness probe failed: HTTP probe failed with statuscode: 404`  
А при использовании команды: `kubectl rollout status deployment frontend-deployment --timeout=60s`, получил:  
`Waiting for deployment "frontend-deployment" rollout to finish: 1 out of 3 new replicas have been updated...`  `error: timed out waiting for the condition`

* Подготовлен манифест node-exporter-daemonset.yaml.

### Задание

Изменить node-exporter-daemonset.yaml так, что бы Node Exporter был развернут не только на worker узлах, но и на master.

#### Ответ

Надо либо отключить политику `Taints: node-role.kubernetes.io/master:NoSchedule` для master узлов, либо дать разрешение pod-ам Node Exporter разворачиваться на master узлах. Например так:

```
tolerations:
  operator: "Exists"
```

## Что стоит знать о безопасности и управлении доступом в Kubernetes

* Рассмотрена авторизация в Кubernetes на основе RBAC.
* Созданы манифесты для создания объектов в Кubernetes.
* Созданы манифесты для управления доступом к Namespaces и к кластеру в целом.

### Task01

1. Создать Service Account **bob**, дать ему роль `admin` в рамках всего кластера;
2. Создать Service Account **dave** без доступа к кластеру;

В ходе выполнения задания были подготовлены и применены манифесты: `01-sa-bob.yaml`, `02-bob-clusteradmin-rolebinding.yaml`, `03-sa-dave.yaml`.

```
# kubectl get sa -n default
NAME            SECRETS   AGE
bob             1         3m32s
dave            1         52s
default         1         3d2h
ingress-nginx   1         44h
```

```
# kubectl get clusterrolebindings bob-rolebinding
NAME              ROLE                        AGE
bob-rolebinding   ClusterRole/cluster-admin   13m
```

### Task02

1. Создать Namespace `prometheus`.
2. Создать Service Account **carol** в этом Namespace.
3. Дать всем Service Account в Namespace `prometheus` возможность делать *get*, *list*, *watch* в отношении Pods всего кластера.

В ходе выполнения задания были подготовлены и применены манифесты: `01-ns-prometheus.yaml`, `02-sa-carol.yaml`, `03-pod-info-role.yaml`, `04-pod-info-rolebinding.yaml`.

```
# kubectl get ns prometheus
NAME         STATUS   AGE
prometheus   Active   3h27m
```

```
# kubectl get sa -n prometheus
NAME      SECRETS   AGE
carol     1         3h30m
default   1         3h33m
```

```
# kubectl get clusterrole pod-info
NAME       CREATED AT
pod-info   2021-04-13T12:02:30Z
```

```
# kubectl get clusterrolebindings -n prometheus pod-info-rolebinding
NAME                   ROLE                   AGE
pod-info-rolebinding   ClusterRole/pod-info   172m
```

### Task03

1. Создать Namespace `dev`
2. Создать Service Account **jane** в Namespace `dev`
3. Дать **jane** роль *admin* в рамках Namespace `dev`
4. Создать Service Account **ken** в Namespace `dev`
5. Дать **ken** роль *view* в рамках Namespace `dev`


В ходе выполнения задания были подготовлены и применены манифесты: `01-ns-dev.yaml`, `02-sa-jane.yaml`, `03-jane-admin-rolebinding.yaml`, `04-sa-ken.yaml`, `05-ken-view-rolebinding.yaml`.

```
# kubectl get ns dev
NAME   STATUS   AGE
dev    Active   3s
```

```
# kubectl get sa -n dev
NAME      SECRETS   AGE
default   1         119s
jane      1         81s
ken       1         4s
```

```
# kubectl get rolebindings -n dev
NAME                     ROLE                AGE
jane-admin-rolebinding   ClusterRole/admin   21s
ken-view-rolebinding     ClusterRole/view    6s
```
