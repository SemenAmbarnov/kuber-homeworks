# Домашнее задание к занятию "Установка кластера K8s"

### Цель задания

Установить кластер K8s.

### Чеклист готовности к домашнему заданию

1. Развернутые ВМ с ОС Ubuntu 20.04-lts


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция по установке kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
2. [Документация kubespray](https://kubespray.io/)

-----

### Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.

## Дополнительные задания (со звездочкой*)

**Настоятельно рекомендуем выполнять все задания под звёздочкой.**   Их выполнение поможет глубже разобраться в материале.   
Задания под звёздочкой дополнительные (необязательные к выполнению) и никак не повлияют на получение вами зачета по этому домашнему заданию. 

------
### Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA
2. Использовать нечетное кол-во Master-node
3. Для cluster ip использовать keepalived или другой способ

### Правила приема работы

1. Домашняя работа оформляется в своем Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl get nodes`, а также скриншоты результатов
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md

# Выполнение
## Задание 1  kubeadm
Сгенерируем ключи для использования их на облачных машинах
```
ssh-keygen -t rsa -b 4096 -f /home/sam/.ssh/yandex/ya_key
```
Создадим мастер ноду в яндекс клауде пробросив наши ключи  
```
yc compute instance create \
--name masternode \
--hostname masternode \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/yandex/ya_key.pub \
--format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address'
```
```
yc compute instance create \
--name masternode \
--hostname masternode \
--zone ru-central1-a \
--folder-id b1gcj17iv37qg7h91dfe
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/yandex/ya_key.pub \
--format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address'
```
Подключимся по ssh
```
ssh -i /home/sam/.ssh/yandex/ya_key yc-user@158.160.53.7
```
В документации сказано, что для корректной работы обязательно нужно отключить swap.  
Отключим до перезагрузки  
```
sudo swapoff -a
```
Так же откорректируем конфигурационный файл, который будет перечитываться при перезагрузке, что бы свап не включился обратно  
```
sudo nano /etc/fstab
```
Закомментируем строку
```
#UUID=be2c7c06-cc2b-4d4b-96c6-e3700932b129 /               ext4    errors=remount-ro 0       1
```
Воспользуемся коммандами из инструкции
```
mkdir -p 0755 /etc/apt/keyrings && \ #for ubuntu 22.04 and latest
sudo apt-get update && \
sudo apt-get install -y apt-transport-https ca-certificates curl && \
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg && \
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
sudo apt-get update && \
sudo apt-get install -y kubelet kubeadm kubectl containerd && \
sudo apt-mark hold kubelet kubeadm kubectl containerd
```
Включаем forwarding. Если необходимо зайдём под рутом `sudo -i`  
```
modprobe br_netfilter && \
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && \
echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf && \
echo "net.bridge.bridge-nf-call-arptables=1" >> /etc/sysctl.conf && \
echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.conf && \
sysctl -p /etc/sysctl.conf
```
Инициализируем мастер ноду с помощью комманды
```
kubeadm init \
--apiserver-advertise-address=10.128.0.17 \ #внутренний IP нашей машины
--pod-network-cidr 10.244.0.0/16 \ #подсеть, в данном случае стандартная для кубера
--apiserver-cert-extra-sans=158.160.53.7 #внешний IP на случай, если мы хотим подключаться к API кластера из вне
#--control-plane-endpoint=cluster_ip_address #Этот флаг необходим только в случае, если у нас несколько мастер нод и нам нужна единая точка входа для него
```
Например  
```
kubeadm init \
--apiserver-advertise-address=10.128.0.17 \
--pod-network-cidr 10.244.0.0/16 \
--apiserver-cert-extra-sans=158.160.53.7
```
В выводе много полезных команд.  
В них описано, как кубер обращается к конфигам, а так же дана команда для присоединения нод к кластеру.  
Для простоты мы используем
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
Но если мы будем переподключаться, то лучше выбрать более стабильный вариант для нашего пользователя
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Проверим, что API получает наши команды
```
kubectl get nodes
```
```
NAME         STATUS     ROLES           AGE   VERSION
masternode   NotReady   control-plane   13m   v1.27.3
```
Что б наш кластер был готов к использованию нам нужно доставить сетевой плагин flanel
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Теперь наш статус изменился
```
NAME         STATUS   ROLES           AGE   VERSION
masternode   Ready    control-plane   27m   v1.27.3
```
Из вывода команды `kubectl describe nodes masternode` мы видим, что etcd запущен на нашей мастер ноде (приведён частичный вывод)  
```
Namespace                   Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                  ------------  ----------  ---------------  -------------  ---
  kube-flannel                kube-flannel-ds-j6tss                 100m (5%)     0 (0%)      50Mi (2%)        0 (0%)         14m
  kube-system                 coredns-5d78c9869d-p7cwd              100m (5%)     0 (0%)      70Mi (3%)        170Mi (9%)     39m
  kube-system                 coredns-5d78c9869d-td2cq              100m (5%)     0 (0%)      70Mi (3%)        170Mi (9%)     39m
  kube-system                 etcd-masternode                       100m (5%)     0 (0%)      100Mi (5%)       0 (0%)         39m
```

Создаём ещё машины в облаке, повторяем все шаги, а так же выполняем команду которая была сгенерирована после инициализации мастера
```
kubeadm join 10.128.0.17:6443 --token wy9j89.el2pmnbyzf2ikyj1 \
	--discovery-token-ca-cert-hash sha256:9586599443ed75d146859327e1e94f6582ad7fdc66c23927e6511529d24
```
Для того что бы выполнить требования ДЗ (1 мастер нода и 4 воркер ноды) нам нужно было бы прикручивать к этому ansible.  
Вместо этого воспользуемся kubespray

## Задание 1  kubespray  

Создадим 5 машин на яндексе
```
masternode=$(yc compute instance create \
--name masternode \
--hostname masternode \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/id_rsa.pub \
--async \
--format json)

workernode1=$(yc compute instance create \
--name workernode1 \
--hostname workernode1 \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/id_rsa.pub \
--async \
--format json)

workernode2=$(yc compute instance create \
--name workernode2 \
--hostname workernode2 \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/id_rsa.pub \
--async \
--format json)

workernode3=$(yc compute instance create \
--name workernode3 \
--hostname workernode3 \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/id_rsa.pub \
--async \
--format json)

workernode4=$(yc compute instance create \
--name workernode4 \
--hostname workernode4 \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=10 \
--ssh-key /home/sam/.ssh/id_rsa.pub \
--async \
--format json)
```
По умолчанию плейбуки ориентированы на работу с внутренними IP адресами, поэтому работать будем внутри сети.  
Скопируем наш приватный ключ на мастер ноду, он должен подходить к публичным ключам на воркерах
```
scp /home/sam/.ssh/id_rsa yc-user@51.250.75.58:~/.ssh/
```
Заходим на мастер ноду 
```
ssh yc-user@51.250.75.58
```
Определим права для ключа
```
sudo chmod 600 ~/.ssh/id_rsa
```
Установим минимально необходимое
```
sudo apt-get install git pip -y
```
Склонируем репозиторий в любую нужную нам папку на то устройство, с которого мы будем заниматься настройкой
```
git clone https://github.com/kubernetes-sigs/kubespray.git
```
Скопируем папку с примерами, что бы настроить под себя
```
cd kubespray \
cp -rfp inventory/sample inventory/mycluster
```
Во время выполнения ДЗ обнаружил, что репозиторий был изменён, и теперь там указаны версии пакетов, которых ещё нет в зеркалах яндекса.  
Что бы установить свежие зависимости нам нужен pip3.9, установим его
```
sudo apt update
sudo apt install python3.9 build-essential libssl-dev libffi-dev python3.9-dev
wget https://bootstrap.pypa.io/get-pip.py
sudo python3.9 get-pip.py
```
Установим зависимости на том же устройстве с его помощью
```
sudo pip3.9 install -r requirements.txt
```
Дождёмся создания машин и посмотрим на адреса
```
yc compute instances list
```
```
+----------------------+-------------+---------------+---------+----------------+-------------+
|          ID          |    NAME     |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+-------------+---------------+---------+----------------+-------------+
| fhm6pc5q6kio57ojru95 | workernode2 | ru-central1-a | RUNNING | 51.250.94.204  | 10.128.0.36 |
| fhmb0ukemj16ehljoa9g | workernode1 | ru-central1-a | RUNNING | 158.160.63.51  | 10.128.0.20 |
| fhmbr8b41jrjk2g7ec0i | workernode3 | ru-central1-a | RUNNING | 158.160.62.146 | 10.128.0.28 |
| fhmdnppvkgd99hj4glar | masternode  | ru-central1-a | RUNNING | 158.160.44.191 | 10.128.0.10 |
| fhmnmm4to2f3h2h9ef0c | workernode4 | ru-central1-a | RUNNING | 84.201.174.151 | 10.128.0.4  |
+----------------------+-------------+---------------+---------+----------------+-------------+
```

Для автоматического создания инвентарника для ансибл, которым будет пользоваться kubespray пойдём по инструкции.  
Объявим наши IP адреса в порядке от мастер ноды до ноды 4.  
Это ручной способ  
```
declare -a IPS=(158.160.44.191 158.160.63.51 51.250.94.204 158.160.62.146 84.201.174.151)
```
Мы можем сделать это автоматически с помощью команды.  
Для EXTERNAL IP
```
declare -a IPS=($(yc compute instances list --format=json | jq -r 'map(select(.name == "masternode" or .name == "workernode1" or .name == "workernode2" or .name == "workernode3" or .name == "workernode4")) | sort_by(.name) | .[].network_interfaces[0].primary_v4_address.one_to_one_nat.address'))
```
Для INTERNAL IP
```
declare -a IPS=($(yc compute instances list --format=json | jq -r 'map(select(.name == "masternode" or .name == "workernode1" or .name == "workernode2" or .name == "workernode3" or .name == "workernode4")) | sort_by(.name) | .[].network_interfaces[0].primary_v4_address.address'))
```
Вывести команду для формирования адресов
```
echo "declare -a IPS=(""${IPS[@]}"")"
```
Запустим билдер.  
Команда из документации выглядит так
```
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```
Команда может не сработать, если мы устанавливали пакеты под другой версией Python.  
В нашем случае мы пользовались pip3.9 поэтому команда будет выглядеть так
```
CONFIG_FILE=inventory/mycluster/hosts.yaml python3.9 contrib/inventory_builder/inventory.py ${IPS[@]}
```
Редактируем файл с нашими хостами `inventory/mycluster/hosts.yaml`  
В нём мы укажем какие ноды будут мастерами, воркерами и на каких хостах будет храниться etcd. 
Так же укажем **ansible_user**, в случае с яндекс облаком это **yc-user**.  
Запускаем плейбук по настройке хостов
```
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v
```
После завершения работы плейбука не забываем скопировать конфиги и дать права, а так же перелогиниться, если это необходимо
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Наш кластер готов
```
sudo kubectl get nodes
```
```
NAME    STATUS   ROLES           AGE    VERSION
node1   Ready    control-plane   149m   v1.26.6
node2   Ready    <none>          148m   v1.26.6
node3   Ready    <none>          148m   v1.26.6
node4   Ready    <none>          148m   v1.26.6
node5   Ready    <none>          148m   v1.26.6
```
Не забываем удалить машины если они нам больше не нужны
```
yc compute instance delete --name masternode --async
yc compute instance delete --name workernode1 --async
yc compute instance delete --name workernode2 --async
yc compute instance delete --name workernode3 --async
yc compute instance delete --name workernode4 --async
```
