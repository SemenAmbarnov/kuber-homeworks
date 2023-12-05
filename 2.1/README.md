# Домашнее задание к занятию «Хранение в K8s. Часть 1»

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке MicroK8S](https://microk8s.io/docs/getting-started).
2. [Описание Volumes](https://kubernetes.io/docs/concepts/storage/volumes/).
3. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1 

**Что нужно сделать**

Создать Deployment приложения, состоящего из двух контейнеров и обменивающихся данными.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Сделать так, чтобы busybox писал каждые пять секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.
4. Продемонстрировать, что multitool может читать файл, который периодоически обновляется.
5. Предоставить манифесты Deployment в решении, а также скриншоты или вывод команды из п. 4.

------

### Задание 2

**Что нужно сделать**

Создать DaemonSet приложения, которое может прочитать логи ноды.

1. Создать DaemonSet приложения, состоящего из multitool.
2. Обеспечить возможность чтения файла `/var/log/syslog` кластера MicroK8S.
3. Продемонстрировать возможность чтения файла изнутри пода.
4. Предоставить манифесты Deployment, а также скриншоты или вывод команды из п. 2.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------


# Выполнение
### Задание 1  

Создадим deployment-volume.yml. Контейнер с busybox будет генерировать таймштампы в файл output.txt  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-volume
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
      - name: busybox
        image: busybox
        command: ['sh', '-c', "while true; do date >> /output/output.txt; sleep 5; done"]
        volumeMounts:
          - name: shared-volume
            mountPath: /output
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
        volumeMounts:
          - name: shared-volume
            mountPath: /input

      volumes:
      - name: shared-volume
        emptyDir: {}
```
Запускаем  
```
microk8s kubectl apply -f deployment-volume.yml
```
Логинимся в наш контейнер с мультитул  
```
microk8s kubectl exec -it netology-deployment-volume-66bc765579-jhtjd -c multitool -- /bin/sh
```
```
cat output.txt
```
Последние 3 строчки
```
Tue Dec  4 07:45:56 UTC 2023
Tue Dec  4 07:46:01 UTC 2023
Tue Dec  4 07:46:06 UTC 2023
```
Подождём какое-то время и запросим их снова. Последние 3 строчки теперь такие  
```
Tue Dec  4 07:56:22 UTC 2023
Tue Dec  4 07:56:27 UTC 2023
Tue Dec  4 07:56:32 UTC 2023
```

### Задание 2  

Создадим daemonSet.yml следующего содержания  
```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: read-node-logs
  labels:
    app: read-node-logs
spec:
  selector:
    matchLabels:
      app: read-node-logs
  template:
    metadata:
      labels:
        app: read-node-logs
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: var-log
          mountPath: /var/log
      volumes:
      - name: var-log
        hostPath:
          path: /var/log
          type: ""
```
Создадим его  
```
microk8s kubectl apply -f daemonSet.yml
```
Подключимся в контейнер пода и прочитаем логи ноды
```
microk8s kubectl exec -it read-node-logs-m5ksw -c multitool -- /bin/sh
```
```
cat /var/log/syslog
```
Фрагмент вывода  
```
Dec  4 15:23:21 ubuntu20 microk8s.daemon-containerd[16638]: time="2023-12-04T15:23:21.533671664+05:00" level=info msg="StartContainer for \"442db2a5df69bbb442de6be42ae94e95f7a1fd73b24c909b7fb2f2c6a008a423\" returns successfully"
Dec  4 15:23:22 ubuntu20 microk8s.daemon-kubelite[16819]: I1204 15:23:22.028963   16819 pod_startup_latency_tracker.go:102] "Observed pod startup duration" pod="default/read-node-logs-m5ksw" podStartSLOduration=-9.223372032825838e+09 pod.CreationTimestamp="2023-12-04 15:23:18 +0500 +05" firstStartedPulling="2023-12-04 15:23:20.13444457 +0500 +05 m=+12086.257522956" lastFinishedPulling="0001-01-01 00:00:00 +0000 UTC" observedRunningTime="2023-12-04 15:23:22.028473962 +0500 +05 m=+12088.151552348" watchObservedRunningTime="2023-12-04 15:23:22.028937562 +0500 +05 m=+12088.152016048"
Dec  4 15:23:26 ubuntu20 systemd[1]: run-containerd-runc-k8s.io-c1cbde7579eb86098f73d855a02ac3b151b981a96293eb09c9c937e7be67624f-runc.EkrSCk.mount: Deactivated successfully.
Dec  4 15:23:36 ubuntu20 systemd[1]: run-containerd-runc-k8s.io-c1cbde7579eb86098f73d855a02ac3b151b981a96293eb09c9c937e7be67624f-runc.fn9DRv.mount: Deactivated successfully.
```

