# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 1»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к приложению, установленному в предыдущем ДЗ и состоящему из двух контейнеров, по разным портам в разные контейнеры как внутри кластера, так и снаружи.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Описание Service.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера

1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.
2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080.
3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры.
4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса.
5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

------

### Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.
2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

------

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.


# Выполнение  
### Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера
Удаляем все сервисы, деплойменты и поды из предыдущих заданий.  
Создаём **deployment.yml** c nginx слушающим 80 порт, multitool слушающим 8080 порт и количеством реплик 3
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment
  labels:
    app: netology-app
spec:
  replicas: 3
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
            value: "8080"
          - name: HTTPS_PORT
            value: "11443"
        ports:
        - containerPort: 8080
        - containerPort: 11443

```
Создаём деплоймент
```
microk8s kubectl apply -f /home/sam/git/kuber/deployment.yml
```
Проверяем
```
microk8s kubectl get deployments
```
```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment   3/3     3            3           51s
```
Создаём service3.yml перенаправляющий порты 9001 и 9002 до приложений
```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc3
spec:
  selector:
    app: netology-app
  ports:
    - name: nginx
      protocol: TCP
      port: 9001
      targetPort: 80
    - name: multitool-http
      protocol: TCP
      port: 9002
      targetPort: 8080

```
Создаём service
```
microk8s kubectl apply -f /home/sam/git/kuber/service3.yml
```
Проверяем
```
microk8s kubectl get service
```
```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes      ClusterIP   10.152.183.1     <none>        443/TCP             7d21h
netology-svc3   ClusterIP   10.152.183.128   <none>        9001/TCP,9002/TCP   21s
```
Для выполнения практической работы нам подойдёт наш pod2.yml из предыдущих практик
```yml
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
Запускаем
```
microk8s kubectl apply -f /home/sam/git/kuber/pod2.yml
```
Проверяем
```
microk8s kubectl get pods
```
```
NAME                                   READY   STATUS    RESTARTS   AGE
netology-deployment-85c859f79c-kdwjh   2/2     Running   0          18m
netology-deployment-85c859f79c-z6chs   2/2     Running   0          18m
netology-deployment-85c859f79c-jg2lc   2/2     Running   0          18m
netology-multitool                     1/1     Running   0          2m5s
```
Подключаемся к контейнеру
```
microk8s kubectl exec -it netology-multitool  -- /bin/bash
```
Курлом стучимся в нашу службу по её доменному имени
```
bash-5.1# curl http://netology-svc3:9001
```
Получаем ответ
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
Стучимся по второму порту снова используя доменное имя
```
bash-5.1# curl http://netology-svc3:9002
```
Получаем ответ
```
WBITT Network MultiTool (with NGINX) - netology-deployment-85c859f79c-kdwjh - 10.1.219.250 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```
### Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

Создаём **service-external.yml** для создания сервиса с типом **NodePort** для доступа к приложению **nginx** из вне кластра
```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc-external
spec:
  selector:
    app: netology-app
  ports:
    - name: nginx
      protocol: TCP
      port: 9001
      targetPort: 80
      nodePort: 30001
  type: NodePort

```
При обращении к службе по порту 9001 из вне кластера нас перенаправит на нодпорт, который в свою очередь перенаправит на приложение.
Нодпорт мы можем не указывать, тогда он выделится автоматически из диапозона 30000 - 32767

Запускаем
```
microk8s kubectl apply -f /home/sam/git/kuber/service-external.yml
```
Проверяем
```
microk8s kubectl get service
```
```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes              ClusterIP   10.152.183.1     <none>        443/TCP             7d22h
netology-svc3           ClusterIP   10.152.183.128   <none>        9001/TCP,9002/TCP   47m
netology-svc-external   NodePort    10.152.183.141   <none>        9001:30001/TCP      23s
```
Обращение из вне кластера выдаёт корректный вывод:  
```
sam@netology:~$ curl http://10.152.183.141:9001

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
