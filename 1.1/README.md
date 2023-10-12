# Домашнее задание к занятию «Kubernetes. Причины появления. Команда kubectl»

### Задание 1. Установка MicroK8S

1. Установить MicroK8S на локальную машину или на удалённую виртуальную машину.
2. Установить dashboard.
3. Сгенерировать сертификат для подключения к внешнему ip-адресу.

------

### Задание 2. Установка и настройка локального kubectl
1. Установить на локальную машину kubectl.
2. Настроить локально подключение к кластеру.
3. Подключиться к дашборду с помощью port-forward.


# Выполнение
## Установка на Linux

```bash
sudo apt-get update
```
Обязательно ставим последнюю **стабильную** версию
```
sudo snap install microk8s --classic --channel=1.27/stable
```
Добавляем текущего пользователя в группу, назначаем владельцем, применяем
```
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```
Стартуем
```
microk8s start
```
Проверить статус
```
microk8s status
```
Проверить на готовность к работе
```
microk8s inspect
```
Смотрим на нашу ноду
```
microk8s kubectl get nodes
```
![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/30f20270-6e09-4f39-acbe-cc78e9efbbca)

Добавляем аддон для дашборда, и пару стандартных
```
microk8s enable dashboard dns registry
```
Установим kubectl

```
sudo snap install kubectl --classic
```
Обновим сертификаты
```
sudo microk8s refresh-certs --cert front-proxy-client.crt
```
Дождёмся когда все поды поднимутся и выведем дефолтный токен  
```
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
```
Пробрасываем порты
```
microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443
```
Теперь можем подключиться по https://127.0.0.1:10443 к дашборду

![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/fb2a9540-a647-481a-b941-de4583e03fe9)

В файле **/var/snap/microk8s/current/certs/csr.conf.template** уже прописан IP, он же высвечивается у нас как Cluster IP
```
[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = 10.152.183.1
#MOREIPS
```




