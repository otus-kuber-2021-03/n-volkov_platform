## n-volkov_platform
n-volkov Platform repository


### Знакомство с Kubernetes, основные понятия и архитектура

- Установлен и запущен Minikube.
- Настроено рабочее окружение.

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
удалена строка конфигурации: "- --port=0"
```
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```
удалена строка конфигурации: "- --port=0"
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
