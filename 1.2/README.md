# Домашнее задание к занятию «Базовые объекты K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Pod с приложением и подключиться к нему со своего локального компьютера. 

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. Описание [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) и примеры манифестов.
2. Описание [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

------

### Задание 1. Создать Pod с именем hello-world

1. Создать манифест (yaml-конфигурацию) Pod.
2. Использовать image - gcr.io/kubernetes-e2e-test-images/echoserver:2.2.
3. Подключиться локально к Pod с помощью `kubectl port-forward` и вывести значение (curl или в браузере).

------

### Задание 2. Создать Service и подключить его к Pod

1. Создать Pod с именем netology-web.
2. Использовать image — gcr.io/kubernetes-e2e-test-images/echoserver:2.2.
3. Создать Service с именем netology-svc и подключить к netology-web.
4. Подключиться локально к Service с помощью `kubectl port-forward` и вывести значение (curl или в браузере).

------

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода команд `kubectl get pods`, а также скриншот результата подключения.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------

### Критерии оценки
Зачёт — выполнены все задания, ответы даны в развернутой форме, приложены соответствующие скриншоты и файлы проекта, в выполненных заданиях нет противоречий и нарушения логики.

На доработку — задание выполнено частично или не выполнено, в логике выполнения заданий есть противоречия, существенные недостатки.



# Выполнение

Что бы развернуть pod с помощью yml файла с нужным нам названием и образом создадим файл со следующим содержанием  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: netology-web
  labels:
    app: netology-web
spec:
  containers:
  - name: netology-web
    image: gcr.io/kubernetes-e2e-test-images/echoserver:2.2
    ports:
    - containerPort: 8080
```
Само приложение случает порт 8080, поэтому для контейнера прописываем тот же порт.  
Обязательно создаём   `labels:    app: netology-web` по нему в будущем служба будет отрабатывать селектор.  
Создадим pod командой
```
microk8s kubectl apply -f /home/sam/git/kuber/pod.yml
```
В случае успеха мы увидим `pod/netology-web created` и можем проверить это с помощью команды 

![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/a22989f4-3f03-44b2-af37-98448c9888f9)


```
microk8s kubectl get pods
```

![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/88852bee-f4f3-4f77-a959-7e76ee31b03e)


Проверить какие порты слушаются  
```
microk8s kubectl exec -it netology-web -- netstat -tln
```
![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/1baadff0-ee2a-4c20-8561-89cae0f77150)


Пробросим порт командой без указания порта, который мы будем перенаправлять. В этом случае выберется автоматически
```
microk8s kubectl port-forward netology-web :8080
```

Приложение отвечает, отдаёт нам имя пода и прочую информацию


![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/2a665a9e-a8bb-4fb2-8760-a510e7c512f4)


![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/a1331de8-b82b-494e-944c-f8e9fbf1ad9e)



Создаём сервис с привязкой к нашему поду с помощью файла service.yml следующего содержания
```yml
apiVersion: v1
kind: Service
metadata:
  name: netology-svc
spec:
  selector:
    app: netology-web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```
И запускаем  
```
microk8s kubectl apply -f /home/sam/git/kuber/service.yml
```
Видим `service/netology-svc created` и проверяем  
```
microk8s kubectl get services
```

![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/a454c860-efe0-400f-8601-d2099cb472c3)


Служба создалась. Выведем подробную информацию по ней и проверим, что автоматически создались endpoints, это будет означать, что служба нашла нужный нам под  
```
microk8s kubectl describe svc netology-svc
```

![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/c7774c9b-f1f3-4537-88b0-90e4fdc0d539)


Пробрасываем порт, подключаемся и видим тот же самый вывод с нашего пода
```
microk8s kubectl port-forward service/netology-svc :80
```


![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/15539c42-b55a-4b2f-a522-5894ef15219f)

