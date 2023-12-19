# Домашнее задание к занятию «Обновление приложений»

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.
3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.
4. Откатиться после неудачного обновления.

## Дополнительные задания — со звёздочкой*

Задания дополнительные, необязательные к выполнению, они не повлияют на получение зачёта по домашнему заданию. **Но мы настоятельно рекомендуем вам выполнять все задания со звёздочкой.** Это поможет лучше разобраться в материале.   

### Задание 3*. Создать Canary deployment

1. Создать два deployment'а приложения nginx.
2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.


# Выполнение  
### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор  

Так как новые версии приложения не умеют работать со старыми, а так же мы очень ограничены в ресурсах, при этом нет упоминания о том, что для нас критична доступность приложения, то нам подойдёт стратегия **Recreate** при которой все старые поды удаляются, а затем поднимаются новые вместо них.  

Если наши приложения не конфликтовали бы друг с другом, то в условиях ограниченных ресурсов нам подошла бы стратегия постепенного обновления - **Rolling update**  
В нашем деплойменте нам необходимо было бы указать значения **maxSurge** (максимальное число подов, которое может быть создано свыше желаемого количества) и **maxUnavailable** ( максимальное число подов, которые могут быть недоступны в процессе обновления) по **20%** так как мы распологаем только этими мощностями для совершения обновления.
```yml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 20%
    maxUnavailable: 20%
```

### Задание 2. Обновить приложение  
Создадим деплоймент с nginx 1.19.  
Для максимально быстрого обновления мы выставили значение **maxSurge** в **100%**, для сохранения доступности выставили **maxUnavailable: 50%**

<details>

  <summary><b>deployment-rollingupdate.yml</b></summary>
  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-back
  labels:
    app: netology-back
spec:
  replicas: 5
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 50%
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
        ports:
        - containerPort: 8080      
      - name: nginx119
        image: nginx:1.19
        ports:
        - containerPort: 80
```
</details>

  
Создадим и проверим
```
kubectl apply -f "/home/yc-user/deployment-rollingupdate.yml"
```
```
kubectl get deployments.apps
```
```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment-back   5/5     5            5           4m50s
```
Теперь изменим версию nginx 
```yml
- name: nginx120
  image: nginx:1.20
```
В отдельном окне можно ввести команду для отображения состояния подов в реальном времени
```
kubectl get pod -w
```
Применим изменения, тем самым запустив обновление
```
kubectl apply -f "/home/yc-user/deployment-rollingupdate.yml"
```
Когда вывод завершится, посмотрим на наши поды
```
kubectl get pods
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
netology-deployment-back-7574fd5b7-5ngwx   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-82ncn   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-n6xr4   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-rtbsm   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-ztw92   2/2     Running   0          84s
```
Подключимся к поду и проверим текущую версию nginx
```
kubectl exec -it netology-deployment-back-7574fd5b7-5ngwx -c nginx120 -- /bin/bash
```
```
nginx -version
```
```
nginx version: nginx/1.20.2
```
Приложение обновлено.  
Проверим историю обновлений командой
```
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Попытаемся обновиться на версию 1.28
```yml
      - name: nginx128
        image: nginx:1.28
```
```
kubectl apply -f "/home/yc-user/deployment-rollingupdate.yml"
```
```
kubectl get pods
```
```
NAME                                        READY   STATUS             RESTARTS   AGE
netology-deployment-back-6ff467cd6c-6jmkj   1/2     ErrImagePull       0          76s
netology-deployment-back-6ff467cd6c-94p49   1/2     ImagePullBackOff   0          76s
netology-deployment-back-6ff467cd6c-bnzbr   1/2     ImagePullBackOff   0          76s
netology-deployment-back-6ff467cd6c-c7szx   1/2     ErrImagePull       0          76s
netology-deployment-back-6ff467cd6c-tnb26   1/2     ImagePullBackOff   0          76s
netology-deployment-back-7574fd5b7-82ncn    2/2     Running            0          25m
netology-deployment-back-7574fd5b7-n6xr4    2/2     Running            0          25m
netology-deployment-back-7574fd5b7-ztw92    2/2     Running            0          25m
```
100% нашего приложения это 5 реплик.
Они попытались подняться, но у них не получилось из-за ошибки образа (такого образа не существует).
Половина старого приложения (округлённая до бОльшего значения, то есть до 3 реплик) осталась функционировать.  
Посмотрим на количество наших ревизий
```
kubectl get rs
```
```
NAME                                  DESIRED   CURRENT   READY   AGE
netology-deployment-back-6ff467cd6c   5         5         0       7m42s
netology-deployment-back-7574fd5b7    3         3         3       31m
netology-deployment-back-7d6db7d77f   0         0         0       36m
```
```
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
Выполним откат
```
kubectl rollout undo deployment/netology-deployment-back
```
```
kubectl get pods
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
netology-deployment-back-7574fd5b7-82ncn   2/2     Running   0          42m
netology-deployment-back-7574fd5b7-8c65g   2/2     Running   0          37s
netology-deployment-back-7574fd5b7-98h75   2/2     Running   0          37s
netology-deployment-back-7574fd5b7-n6xr4   2/2     Running   0          42m
netology-deployment-back-7574fd5b7-ztw92   2/2     Running   0          42m
```
Мы выполнили откат.  
Посмотрим на историю ревизий
```
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```
Вывод говорит о том, что ревизия, которая шла под номером 2 теперь актуальна и ей был присвоен номер 4
