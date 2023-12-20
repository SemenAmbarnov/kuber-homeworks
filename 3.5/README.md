# Домашнее задание к занятию Troubleshooting

### Цель задания

Устранить неисправности при деплое приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Задание. При деплое приложение web-consumer не может подключиться к auth-db. Необходимо это исправить

1. Установить приложение по команде:
```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
```
2. Выявить проблему и описать.
3. Исправить проблему, описать, что сделано.
4. Продемонстрировать, что проблема решена.


### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

# Выполнение
Применяем команду
```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
```
Получаем ответ
```
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "web" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
```
Действительно в манифесте есть указания на эти неймспейсы, создадим их и попробуем снова
```
kubectl create namespace web
kubectl create namespace data
```
```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
```
```
deployment.apps/web-consumer created
deployment.apps/auth-db created
service/auth-db created
```
Посмотрим на логи
```
kubectl logs web-consumer-577d47b97d-vzfxw -n web
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
```
Приложение не может подключиться к "auth-db".  
Узнаем IP этого сервиса
```
kubectl get service -n data
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
auth-db   ClusterIP   10.233.58.165   <none>        80/TCP    14m
```
Подключимся к контейнеру с нашим веб приложением
```
kubectl exec -n web -it web-consumer-577d47b97d-xg7qd -- /bin/sh
```
Проверим
```
[ root@web-consumer-577d47b97d-xg7qd:/ ]$ curl auth-db
curl: (6) Couldn't resolve host 'auth-db'
[ root@web-consumer-577d47b97d-xg7qd:/ ]$ curl 10.233.58.165
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
```
Мы смогли подключиться по IP, но не смогли по имени.  
Это говорит о том, что вызывающий контейнер не знает куда ведёт имя **auth-db**  
В таких случаях нужно прикручивать DNS либо прописать пару "имя - IP" в локальный DNS контейнера, то есть в файл hosts.  
Но если нам нужно решить проблему подключения прямо сейчас просто изменим этот деплоймент
```
kubectl -n web edit deployments web-consumer
```
Поменяем строчку
```
- while true; do curl auth-db; sleep 5; done
```
На
```
- while true; do curl 10.233.58.165; sleep 5; done
```
Снова посмотрим в логи (приведён частичный вывод)
```
kubectl logs -n web web-consumer-7c589bc659-mbhk5
```
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   101k      0 --:--:-- --:--:-- --:--:--  119k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
В логах мы видим, что доступ к сервису появился, проблема решена
