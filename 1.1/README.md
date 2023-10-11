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
Проверить статус
```
microk8s status
```
Проверить на готовность к работе
```
microk8s inspect
```
Стартуем
```
microk8s start
```
Смотрим на нашу ноду
```
microk8s kubectl get nodes
```
![image](https://github.com/SemenAmbarnov/kuber-homeworks/assets/92155007/30f20270-6e09-4f39-acbe-cc78e9efbbca)



Добавляем аддон для дашборда, и пару стандартных
```
microk8s enable dashboard dns registry
