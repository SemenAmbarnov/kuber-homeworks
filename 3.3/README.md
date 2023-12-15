# Домашнее задание к занятию «Как работает сеть в K8s»

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер k8s с установленным сетевым плагином calico

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/)
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy)

-----

### Задание 1. Создать сетевую политику (или несколько политик) для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы
2. В качестве образа использовать network-multitool
3. Разместить поды в namespace app
4. Создать политики чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены
5. Продемонстрировать, что трафик разрешен и запрещен

### Правила приема работы

1. Домашняя работа оформляется в своем Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md


## Выполнение
### Задание 1  

Выполним шаги по разворачиванию kubernetes кластера из прошлого ДЗ.  
Проверяем, что у нас есть подготовленный класстер.  
```
kubectl get nodes
```
```
NAME    STATUS   ROLES           AGE    VERSION
node1   Ready    control-plane   103m   v1.26.6
node2   Ready    <none>          101m   v1.26.6
node3   Ready    <none>          101m   v1.26.6
node4   Ready    <none>          101m   v1.26.6
```
Создадим деплойменты и сервисы под **frontend** **backend** и **cache**

<details>

  <summary><b>deployment-back.yml</b></summary>
  
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
        ports:
        - containerPort: 8080
```
</details>

<details>

  <summary><b>deployment-front.yml</b></summary>
  
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
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
```
</details>

<details>

  <summary><b>deployment-cache.yml</b></summary>
  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-cache
  labels:
    app: netology-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netology-cache
  template:
    metadata:
      labels:
        app: netology-cache
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8090"
        ports:
        - containerPort: 8090

```
</details>

<details>

  <summary><b>service-back.yml</b></summary>
  
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
</details>

<details>

  <summary><b>service-front.yml</b></summary>
  
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
</details>

<details>

  <summary><b>service-cache.yml</b></summary>
  
```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc-cache
spec:
  selector:
    app: netology-cache
  ports:
    - name: multitool-http
      protocol: TCP
      port: 8090
      targetPort: 8090
```
</details>

Создаём деплойменты и проверяем
```
kubectl apply -f deployment-back.yml -f deployment-front.yml -f deployment-cache.yml
```
```
kubectl get deployments
```
```
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment-back    1/1     1            1           12s
netology-deployment-cache   1/1     1            1           11s
netology-deployment-front   3/3     3            3           12s

```
Создаём сервисы и проверяем
```
kubectl apply -f service-back.yml -f service-front.yml -f service-cache.yml
```
```
kubectl get service
```
```
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes           ClusterIP   10.233.0.1      <none>        443/TCP    3h46m
netology-svc-back    ClusterIP   10.233.5.197    <none>        8080/TCP   5s
netology-svc-cache   ClusterIP   10.233.3.138    <none>        8090/TCP   5s
netology-svc-front   ClusterIP   10.233.41.136   <none>        80/TCP     5s
```
Узнаем имена подов
```
kubectl get pods
```
```
NAME                                         READY   STATUS    RESTARTS   AGE
netology-deployment-back-b6645c697-v688q     1/1     Running   0          56s
netology-deployment-cache-776885f5d6-5tbbz   1/1     Running   0          56s
netology-deployment-front-85b977d4d9-94qd2   1/1     Running   0          56s
netology-deployment-front-85b977d4d9-hsdgl   1/1     Running   0          56s
netology-deployment-front-85b977d4d9-nh6dg   1/1     Running   0          56s
```
Подключимся к поду фронта и обратимся к беку и к кешу по их IP и портам сервисов
```
kubectl exec -it netology-deployment-front-85b977d4d9-94qd2 -c multitool -- /bin/bash
```
```
curl 10.233.5.197:8080
```
```
WBITT Network MultiTool (with NGINX) - netology-deployment-back-b6645c697-v688q - 10.233.74.68 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
```
curl 10.233.3.138:8090
```
```
WBITT Network MultiTool (with NGINX) - netology-deployment-cache-776885f5d6-5tbbz - 10.233.75.6 - HTTP: 8090 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
Нам отвечает и бек и кеш.  
Изменим это поведение.  
Для начала создадим политику запрещающую всем любые входящие соединения

<details>

  <summary><b>ingress-deny-all.yml</b></summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```
</details>

Применим её
```
kubectl apply -f ingress-deny-all.yml
```
Проверяем
```
kubectl get networkpolicies
```
```
NAME               POD-SELECTOR   AGE
ingress-deny-all   <none>         34s
```
Снова подключимся к фронту и попробуем обратиться к беку
```
kubectl exec -it netology-deployment-front-85b977d4d9-94qd2 -c multitool -- /bin/bash
```
```
curl 10.233.5.197:8080
```
В ответ нам ничего не приходит, так как мы запретили все входящие соединения.  
Стоит понимать, что сетевые политики это объект неймспейса и так как мы его не указали, описанные правила касаются только дефолтного неймспейса.  
Создадим правила согласно ДЗ.
На мой взгляд задача не совсем корректна.  
Связь должна быть не односторонней, как описано в ДЗ  
```frontend -> backend -> cache```  
А двусторонней, что бы приложения могли общаться.  
В моём понимании правильнее будет создать правила по типу:  
```frontend <-> backend <-> cache```  
То есть мы будем создавать правила, что бы могли общаться друг с другом сервисы фронта и бека, сервисы бека и кеша, а сервисы фронта и кеша не имели бы доступа друг к другу.   
Создадим манифесты с описанием сетевых политик для фронта, бека и кеша
<details>

  <summary><b>front-np.yml</b></summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
spec:
  podSelector:
    matchLabels:
      app: netology-front
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: netology-back
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: netology-back
```
</details>

<details>

  <summary><b>back-np.yml</b></summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
spec:
  podSelector:
    matchLabels:
      app: netology-back
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchExpressions:
              - {key: app, operator: In, values: [netology-front, netology-cache]}
  egress:
    - to:
        - podSelector:
            matchExpressions:
              - {key: app, operator: In, values: [netology-front, netology-cache]}
```
</details>

<details>

  <summary><b>cache-np.yml</b></summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache-network-policy
spec:
  podSelector:
    matchLabels:
      app: netology-cache
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: netology-back
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: netology-back
```
</details>

Применим

```
kubectl apply -f back-np.yml -f front-np.yml -f cache-np.yml
```
Проверим
```
kubectl get networkpolicies
```
```
NAME                      POD-SELECTOR         AGE
backend-network-policy    app=netology-back    103m
cache-network-policy      app=netology-cache   7m6s
frontend-network-policy   app=netology-front   103m
ingress-deny-all          <none>               166m
```
Подключимся к фронту и обратимся к беку и к кешу
```
kubectl exec -it netology-deployment-front-85b977d4d9-94qd2 -c multitool -- /bin/bash
```
```
bash-5.1# curl 10.233.5.197:8080
WBITT Network MultiTool (with NGINX) - netology-deployment-back-b6645c697-v688q - 10.233.74.68 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
bash-5.1# curl 10.233.3.138:8090
^C
bash-5.1#
```
Как и задумывалось мы получили ответ от бека, но не получили от кеша.  
Подключимся к беку и обратимся к фронту и к кешу
```
kubectl exec -it netology-deployment-back-b6645c697-v688q -c multitool -- /bin/bash
```
```
bash-5.1# curl 10.233.41.136:80
WBITT Network MultiTool (with NGINX) - netology-deployment-front-85b977d4d9-94qd2 - 10.233.75.5 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
bash-5.1# curl 10.233.3.138:8090
WBITT Network MultiTool (with NGINX) - netology-deployment-cache-776885f5d6-5tbbz - 10.233.75.6 - HTTP: 8090 , HTTPS: 443 . (Formerly praqma/network-multitool)
bash-5.1#
```
Как мы и задумывали мы получили ответ от обоих.
Подключимся к кешу и обратимся к беку и к фронту
```
kubectl exec -it netology-deployment-cache-776885f5d6-5tbbz -c multitool -- /bin/bash
```
```
bash-5.1# curl 10.233.5.197:8080
WBITT Network MultiTool (with NGINX) - netology-deployment-back-b6645c697-v688q - 10.233.74.68 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
bash-5.1# curl 10.233.41.136:80
^C
bash-5.1#
```
Мы получили ответ от бека, но не получили от фронта, как мы и задумывали.




