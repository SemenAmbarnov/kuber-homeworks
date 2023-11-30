# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------


# Выполнение  
### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod
Создадим deployment.yml следующего содержания  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment
  labels:
    app: netology-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netology-app
  template:
    metadata:
      labels:
        app: netology-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 1180
        - containerPort: 11443
```
Создаём  
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment.yml
```
После вывода `deployment.apps/netology-deployment created` проверим  
```
microk8s kubectl get deployments
```
```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment   0/1     1            0           81s
```
Мы увидим наш деплоймент, но он ещё не готов. Что бы отследить прогресс воспользуемся командой  
```
microk8s kubectl rollout status deployment/netology-deployment
```
Дождёмся окончания развёртывания  
```
Waiting for deployment "netology-deployment" rollout to finish: 0 of 1 updated replicas are available...
deployment "netology-deployment" successfully rolled out
```
Проверим с помощью `microk8s kubectl get deployments`
```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment   0/1     1            0           2m
```
Деплоймент всё равно не готов, начнём траблшутинг.  
Посмотрим на поды привязанные к селектору деплоймента  

```
microk8s kubectl get pods -l app=netology-app
```
```
NAME                                  READY   STATUS             RESTARTS         AGE
netology-deployment-c769bfd5b-nxv6n   1/2     CrashLoopBackOff   10 (3m31s ago)   30m
```
Под создан, в нём готов один контейнер и ещё один не смог запуститься. Выведем более подробную информацию о нём  
```
microk8s kubectl describe pod netology-deployment-c769bfd5b-nxv6n
```
```
Name:             netology-deployment-c769bfd5b-nxv6n
Namespace:        default
Priority:         0
Service Account:  default
Node:             ubuntu20/10.236.64.25
Start Time:       Thur, 30 Nov 2023 14:01:55 +0500
Labels:           app=netology-app
                  pod-template-hash=c769bfd5b
Annotations:      cni.projectcalico.org/containerID: 46a2f3fc2b0c9d2773ff0418b0c3d36cd77415e4533d7882dbd830063651333e
                  cni.projectcalico.org/podIP: 10.1.219.208/32
                  cni.projectcalico.org/podIPs: 10.1.219.208/32
Status:           Running
IP:               10.1.219.208
IPs:
  IP:           10.1.219.208
Controlled By:  ReplicaSet/netology-deployment-c769bfd5b
Containers:
  nginx:
    Container ID:   containerd://7ea0cd7837e331759e7f998915e798a07d052316c698763c02510a32e3c7bbb5
    Image:          nginx:1.14.2
    Image ID:       docker.io/library/nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thur, 30 Nov 2023 14:01:55 +0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4r729 (ro)
  multitool:
    Container ID:   containerd://a8d76e9c81ebcc9fd787a87ef591a105e67f8ce57e780137e30fa1ffa613b5c6
    Image:          wbitt/network-multitool
    Image ID:       docker.io/wbitt/network-multitool@sha256:82a5ea955024390d6b438ce22ccc75c98b481bf00e57c13e9a9cc1458eb92652
    Ports:          1180/TCP, 11443/TCP
    Host Ports:     0/TCP, 0/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Thur, 30 Nov 2023 14:01:55 +0500
      Finished:     Thur, 30 Nov 2023 14:01:55 +0500
    Ready:          False
    Restart Count:  11
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4r729 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-4r729:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                    From     Message
  ----     ------   ----                   ----     -------
  Warning  BackOff  3m40s (x137 over 33m)  kubelet  Back-off restarting failed container multitool in pod netology-deployment-c769bfd5b-nxv6n_default(c401aff4-f30a-490f-b439-fbfcfebaacd7)
```

Из вывода мы видим, что у нас проблемы с контейнером multitool.  
Попробуем выяснить подробнее, посмотрев логи конкретно этого контейнера в поде.  
```
microk8s kubectl logs netology-deployment-c769bfd5b-nxv6n -c multitool
```
Из всего вывода нас интересует строчка:  
```
2023/11/30 14:49:26 [emerg] 1#1: bind() to 0.0.0.0:80 failed (98: Address in use)
```
Она говорит о том, что порт 80 уже занят.  
Согласно документации порты для прослушивания можно переназначить с помощью переменных окружения.  
Сделаем это, добавив необходимые строчки в наш deployment.yml приведя его к виду:  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment
  labels:
    app: netology-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netology-app
  template:
    metadata:
      labels:
        app: netology-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "1180"
          - name: HTTPS_PORT
            value: "11443"
        ports:
        - containerPort: 1180
        - containerPort: 11443

```
Удалим наш деплоймент, создадим его заново и проверим результат  
```
microk8s kubectl delete deployment netology-deployment
```
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment.yml
```
```
microk8s kubectl get deployments
```
```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment   1/1     1            1           17s
```
Всё получилось. Проверим количество подов в нём
```
microk8s kubectl get pods -l app=netology-app
```
```
NAME                                   READY   STATUS    RESTARTS   AGE
netology-deployment-6c6857d877-94hxb   2/2     Running   0          4m36s
```
У нас есть под с двумя контейнерами.  
В нашем deployment.yml изменим количество реплик до двух  
```
spec:
  replicas: 2
```
И применим изменения  
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment.yml
```
После вывода `deployment.apps/netology-deployment configured` проверим количесто подов
```
microk8s kubectl get pods -l app=netology-app
```
```
NAME                                   READY   STATUS    RESTARTS   AGE
netology-deployment-6c6857d877-94hxb   2/2     Running   0          18m
netology-deployment-6c6857d877-gbv75   2/2     Running   0          74s
```
2 пода по 2 контейнера в каждом.  
Создадим сервис, объединяющий эти приложения.  
Опишим его в файле service2.yml следующего содержания  
```
apiVersion: v1
kind: Service
metadata:
  name: netology-svc2
spec:
  selector:
    app: netology-app
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80
    - name: multitool-http
      protocol: TCP
      port: 1180
      targetPort: 1180
    - name: multitool-https
      protocol: TCP
      port: 11443
      targetPort: 11443

```
И запустим  
```
microk8s kubectl apply -f /home/sam/git/kuber/service2.yml
```
Создадим отдельный под с помощью pod2.yml следующего содержания  
```
apiVersion: v1
kind: Pod
metadata:
  name: netology-multitool
  labels:
    app: netology-multitool
spec:
  containers:
  - name: netology-multitool
    image: wbitt/network-multitool
    env:
      - name: HTTP_PORT
        value: "2280"
      - name: HTTPS_PORT
        value: "22443"
    ports:
    - containerPort: 2280
    - containerPort: 22443
```
И запустим  
```
microk8s kubectl apply -f /home/sam/git/kuber/pod2.yml
```
С помощью `microk8s kubectl get services` мы узнаём, что наша служба `netology-svc2` работает с IP `10.152.183.132`  

Так как в отдельном поде, который мы создали запущен только один контейнер, команда для подключения к нему будет выглядеть так:  
```
microk8s kubectl exec -it netology-multitool  -- /bin/bash
```
Курлом стучимся в нашу службу  
```
bash-5.1# curl http://10.152.183.132:80
```
Nxinx отвечает  
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
```
bash-5.1# curl http://10.152.183.132:1180
```
Nginx в контейнере multitool отвечает  
```
bash-5.1# curl http://10.152.183.132:1180
WBITT Network MultiTool (with NGINX) - netology-deployment-6c6857d877-gbv75 - 10.1.219.205 - HTTP: 1180 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий
Удаляем все ранее созданные поды, службы и деплойменты для экономии ресурсов и избежания конфликтов.  
Создаём **deployment-for-init.yml** в котором помимо контейнера с **nginx** будет описан **init** контейнер на основе **busybox** с ожиданием нашего сервиса **netology-svc-for-init**  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-for-init
  labels:
    app: netology-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netology-app
  template:
    metadata:
      labels:
        app: netology-app
    spec:
      initContainers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'until nslookup netology-svc-for-init.default.svc.cluster.local; do echo waiting for netology-svc-for-init; sleep 2; done;']

      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
В команде для инит контейнера мы ищем службу **netology-svc-for-init.default.svc.cluster.local** где default это имя неймспейса  
Запускаем
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment-for-init.yml
```
Сотрим на поды
```
microk8s kubectl get pods -l app=netology-app
```
```
NAME                                            READY   STATUS     RESTARTS   AGE
netology-deployment-for-init-6975ff8787-n8bn2   0/1     Init:0/1   0          2m13s
```
Создаём service-for-init.yml
```
apiVersion: v1
kind: Service
metadata:
  name: netology-svc-for-init
spec:
  selector:
    app: netology-app
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80
```
Запускаем  
```
microk8s kubectl apply -f /home/sam/git/kuber/service-for-init.yml
```
Снова посмотрим на поды  
```
microk8s kubectl get pods -l app=netology-app
```
Поначалу увидим  
```
NAME                                            READY   STATUS            RESTARTS   AGE
netology-deployment-for-init-8596fb4877-htk9m   0/1     PodInitializing   0          34s
```
А затем  
```
NAME                                            READY   STATUS    RESTARTS   AGE
netology-deployment-for-init-8596fb4877-htk9m   1/1     Running   0          46s
```
Под запустился только после старта службы, как мы и задумывали.
