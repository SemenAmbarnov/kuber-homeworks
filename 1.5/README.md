# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 2»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к двум приложениям снаружи кластера по разным путям.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://microk8s.io/docs/getting-started) по установке MicroK8S.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Service.
3. [Описание](https://kubernetes.io/docs/concepts/services-networking/ingress/) Ingress.
4. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment приложений backend и frontend

1. Создать Deployment приложения _frontend_ из образа nginx с количеством реплик 3 шт.
2. Создать Deployment приложения _backend_ из образа multitool. 
3. Добавить Service, которые обеспечат доступ к обоим приложениям внутри кластера. 
4. Продемонстрировать, что приложения видят друг друга с помощью Service.
5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

------

### Задание 2. Создать Ingress и обеспечить доступ к приложениям снаружи кластера

1. Включить Ingress-controller в MicroK8S.
2. Создать Ingress, обеспечивающий доступ снаружи по IP-адресу кластера MicroK8S так, чтобы при запросе только по адресу открывался _frontend_ а при добавлении /api - _backend_.
3. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
4. Предоставить манифесты и скриншоты или вывод команды п.2.

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------

# Выполнение  

Опишем **deployment** для **frontend** в файле deployment-front.yml  

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-front
  labels:
    app: netology-front
spec:
  replicas: 3
  selector:
    matchLabels:
      app: netology-front
  template:
    metadata:
      labels:
        app: netology-front
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
Опишем **deployment** для **backend** в файле deployment-back.yml  

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-back
  labels:
    app: netology-back
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netology-back
  template:
    metadata:
      labels:
        app: netology-back
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8080"
          - name: HTTPS_PORT
            value: "11443"
        ports:
        - containerPort: 8080
        - containerPort: 11443
```

Опишем **service** для **frontend** в файле service-front.yml  

```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc-front
spec:
  selector:
    app: netology-front
  ports:
    - name: nginx
      protocol: TCP
      port: 80
      targetPort: 80
```
Опишем **service** для **backend** в файле service-back.yml  

```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc-back
spec:
  selector:
    app: netology-back
  ports:
    - name: multitool-http
      protocol: TCP
      port: 8080
      targetPort: 8080
```

Запускаем деплойменты и сервис  
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment-front.yml
```
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment-back.yml
```
```
microk8s kubectl apply -f /home/sam/git/kuber/service-back.yml
```
```
microk8s kubectl apply -f /home/sam/git/kuber/service-front.yml
```
Проверяем
```
microk8s kubectl get deployments
```
    NAME                        READY   UP-TO-DATE   AVAILABLE   AGE  
    netology-deployment-front   3/3     3            3           23s  
    netology-deployment-back    1/1     1            1           15s  

```
microk8s kubectl get svc
```
```
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes           ClusterIP   10.152.183.1     <none>        443/TCP    15d
netology-svc-back    ClusterIP   10.152.183.169   <none>        8080/TCP   25s
netology-svc-front   ClusterIP   10.152.183.229   <none>        80/TCP     20s
```
Узнаём название пода бэкенда, подключаемся к нему и проверяем доступ до сервиса фронта
```
microk8s kubectl get pods
```
```
microk8s kubectl exec -it netology-deployment-back-5f7f4dd46c-9m5xk -- /bin/bash
```
```
bash-5.1# curl http://netology-svc-front:80
```
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
Фронт ответил. Постучимся к бэку через сервис  
```
bash-5.1# curl http://netology-svc-back:8080
```
```
WBITT Network MultiTool (with NGINX) - netology-deployment-back-5f7f4dd46c-9m5xk - 10.1.219.233 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```
Бэк ответил.  
Устанавливаем ingress в microk8s  
```
microk8s enable ingress
```

Создаём ingress.yml  
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: netologty-ingress
spec:
  rules:
    - host: netology-portal.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: netology-svc-front
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: netology-svc-back
                port:
                  number: 8080
```
Запускаем  
```
microk8s kubectl apply -f /home/sam/git/kuber/ingress.yml
```
Проверим с каким адресом светится ингресс
```
microk8s kubectl get ingress
```
```
NAME                CLASS    HOSTS                ADDRESS     PORTS   AGE
netologty-ingress   public   netology-portal.io   127.0.0.1   80      3h42m
```
Это адрес нашего локалхоста. Так как в правилах ингресса нельзя указать IP мы указали маршртуризировать запросы к `host: netology-portal.io`  
Добавим это в локальный DNS машины поправив файл hosts
```
echo '127.0.0.1    netology-portal.io' | sudo tee -a /etc/hosts
```
Теперь можем попробовать обратиться с нашей машины и убедиться, что правила ingress отрабатывают  
```
sam@netology:~$ curl http://netology-portal.io
```
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
Корректно отработало правило для корня /  
```
sam@netology:~$ curl http://netology-portal.io/api
```
```
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.20.2</center>
</body>
</html>
```
Правило для /api так же отработало корректно.  
Нам откликнулся nginx версии 1.20.2, именно эта версия установлена у нас в бэке.  
Мы видим другой вывод, отличный от обращения через службу из-за особенностей работы приложения, которое генерирует ответ исходя из данных о клиенте, но не получает эту информацию, так как запрос пришел не от самого клиента, а через ingress
