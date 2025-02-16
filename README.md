# manual-k8s-kubeadm
Пошаговая инструкция по созданию кластера Kubernetes из одной мастер-ноды и трех воркер-нод.  
Конфигурация виртуальных машин: 2CPU / RAM 2Гб / HDD 50 Гб / OS AlmaLinux 9.5.


# 1. ВЫПОЛНИТЬ НА ВСЕХ НОДАХ
## 1.1. Добавить имена хостов в FQDN-формате  
(каждому серверу соответствует своя строка)
```console
hostnamectl set-hostname "master-node.a123mc.ru" && exec bash
hostnamectl set-hostname "worker-node1.a123mc.ru" && exec bash
hostnamectl set-hostname "worker-node2.a123mc.ru" && exec bash
hostnamectl set-hostname "worker-node3.a123mc.ru" && exec bash
```

## 1.2. Добавить в `/etc/hosts` IP-адреса и имена хостов в FQDN-формате
```console
127.0.0.1       master-node.a123mc.ru     master-node
192.168.88.31   master-node.a123mc.ru     master-node
192.168.88.28   worker-node1.a123mc.ru    worker-node1
192.168.88.27   worker-node2.a123mc.ru    worker-node2
192.168.88.26   worker-node3.a123mc.ru    worker-node3
```

## 1.3. Добавить в `/etc/resolv.conf` домен поиска
```console
search a123mc.ru
nameserver 192.168.88.1
```

## 1.4. Установить пакеты для траблшутинга и дебага
```bash
dnf check-update && \
dnf update && \
dnf clean all && \
dnf install epel-release -y && \
dnf install -y mc htop atop wget curl sysstat iotop net-tools openssl-devel bzip2-devel zlib openssl grep sed lsof bash-completion perf telnet yum-utils iputils nc tcpdump rsyslog bind-utils jq tar python3 nfs-utils chrony vim git
```

## 1.5. Отключить  файрвол
```bash
systemctl stop firewalld && \
systemctl disable firewalld && \
systemctl status firewalld && \
iptables -F
```

## 1.6. Отключить SWAP
```bash
swapoff -a && \
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 1.7. Отключить SElinux
```bash
setenforce 0 && \
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

## 1.8. Тюнинг ядра линукс
```bash
cat <<EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
vm.swappiness = 0
vm.overcommit_memory = 1
net.ipv4.tcp_tw_recycle = 0
vm.panic_on_oom = 0
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 1048576
fs.file-max = 52706963
fs.nr_open = 52706963
net.ipv6.conf.all.disable_ipv6 = 1
net.netfilter.nf_conntrack_max = 2310720
EOF
```

```bash
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```bash
modprobe overlay
modprobe br_netfilter
sysctl --system
```

## 1.9. Подключить репозиторий кубернетиса
#### Последнюю версию смотреть тут: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management
```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF
```

## 1.10. Установить и настроить containerd
```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && \
dnf install -y containerd.io && \
mkdir -p /etc/containerd && \
(containerd config default | tee /etc/containerd/config.toml) && \
(sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml) && \
systemctl enable containerd && \
systemctl start containerd && \
systemctl status containerd -l
```

## 1.11. Установить бинари кубернетиса
```bash
dnf install -y ipvsadm kubelet kubeadm kubectl --disableexcludes=kubernetes
```

## 1.12. Запустить кублет
```bash
systemctl enable kubelet && \
systemctl start kubelet
```

## 1.13. Запустить chronyd
```bash
systemctl enable chronyd && \
systemctl restart chronyd && \
systemctl status chronyd -l
```

## 1.14. Запустить rsyslog
```bash
systemctl enable rsyslog && \
systemctl start rsyslog && \
systemctl status rsyslog -l
```

## 1.15. Установить crictl для управления контейнерами
#### https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/
```bash
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
EOF
```

### Проверить работоспособность
```bash
crictl images
crictl ps
```

# 2. ВЫПОЛНИТЬ НА МАСТЕР-НОДЕ
## 2.1. Инициализация мастер-ноды, указываем полный хостнейм в FQDN-формате
```bash
kubeadm config images pull
kubeadm init --control-plane-endpoint=master-node.a123mc.ru
```

## Вывод содержит токен для присоединения остальных нод:
```console
    Your Kubernetes control-plane has initialized successfully!
    To start using your cluster, you need to run the following as a regular user:
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    Alternatively, if you are the root user, you can run:
      export KUBECONFIG=/etc/kubernetes/admin.conf
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    You can now join any number of control-plane nodes by copying certificate authorities
    and service account keys on each node and then running the following as root:
      kubeadm join master-node.a123mc.ru:6443 --token gjyfbe.uy9ktl6t5d300lmx \
            --discovery-token-ca-cert-hash sha256:4ecf64eba576190dd1d8aea89da0b336887ec404fd957f7caf4048e5d4cb2cfe \
            --control-plane
    Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join master-node.a123mc.ru:6443 --token gjyfbe.uy9ktl6t5d300lmx \
            --discovery-token-ca-cert-hash sha256:4ecf64eba576190dd1d8aea89da0b336887ec404fd957f7caf4048e5d4cb2cfe
```

## Если мастер-нода не проинициализировалась и команда завершилась с ошибкой, перед повтором следует отменить инициализацию:
```bash
kubeadm reset
ipvsadm --clear
rm -rf /etc/kubernetes/manifests/*
rm -rf /var/lib/etcd/*
```

## Для траблшутинга смотреть логи:
```bash
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps -a | grep kube | grep -v pause
    88434fe3d4b07       1372127edc9da       37 seconds ago       Exited              kube-apiserver            17                  17f520fe51121       kube-apiserver-master-node.a123mc.ru
    d315034cceb43       1372127edc9da       About a minute ago   Exited              kube-apiserver            16                  17f520fe51121       kube-apiserver-master-node.a123mc.ru
    859183259291f       5f23cb154eea1       5 minutes ago        Running             kube-controller-manager   1                   a5ff48d2a6beb       kube-controller-manager-master-node.a123mc.ru
    945edcce05315       9195ad415d31e       5 minutes ago        Running             kube-scheduler            1                   83b56b7de206e       kube-scheduler-master-node.a123mc.ru
```
```bash
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock logs 17f520fe51121
```


## 2.2. После инициализации мастер-ноды копировать конфиги
```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```


# 3. ВЫПОЛНИТЬ НА ВСЕХ НОДАХ (КРОМЕ МАСТЕР-НОДЫ)
## 3.1. После инициализации мастер-ноды на остальных нодах выполнить по очереди подключение к мастер-ноде
```bash
kubeadm join <master_fqdn>:6443 --token <token> --discovery-token-ca-cert-hash <sha256_hash>
```
```bash
kubeadm join master.a123mc.ru:6443 --token gjyfbe.uy6ktl6t5d500lmx \
        --discovery-token-ca-cert-hash sha256:4ecf64eba576160dd1d8aea89da0b336887ec404fd957f7caf3048e5d4cb2cfe
```

## 3.2. Проверить присоединение нод
```bash
kubectl get no
```
```console
    NAME                   STATUS     ROLES           AGE   VERSION
    kube-node1.a123mc.ru   NotReady   <none>          26s   v1.32.2
    kube-node2.a123mc.ru   NotReady   <none>          16s   v1.32.2
    kube-node3.a123mc.ru   NotReady   <none>          10s   v1.32.2
    master.a123mc.ru       NotReady   control-plane   11m   v1.31.6
```


# 4. ВЫПОЛНИТЬ НА МАСТЕР-НОДЕ
## 4.1. Установить CNI (Calico)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml
```

## 4.2. Проверить установку Calico
```bash
watch -n5 kubectl get pods -n kube-system
```
```console
    NAME                                            READY   STATUS    RESTARTS   AGE
    calico-kube-controllers-cc5755cd7-mfqz6         1/1     Running   0          2m7s
    calico-node-krlfp                               1/1     Running   0          2m7s
    calico-node-rq988                               1/1     Running   0          2m7s
    calico-node-vglrd                               1/1     Running   0          2m7s
    calico-node-wc6sf                               1/1     Running   0          2m7s
    coredns-7c65d6cfc9-cw2mr                        1/1     Running   0          15m
    coredns-7c65d6cfc9-xxkxm                        1/1     Running   0          15m
    etcd-master-node.a123mc.ru                      1/1     Running   2          15m
    kube-apiserver-master-node.a123mc.ru            1/1     Running   26         15m
    kube-controller-manager-master-node.a123mc.ru   1/1     Running   2          15m
    kube-proxy-2vns4                                1/1     Running   0          4m52s
    kube-proxy-d5nh9                                1/1     Running   0          5m2s
    kube-proxy-kgzhm                                1/1     Running   0          4m46s
    kube-proxy-sc69q                                1/1     Running   0          15m
    kube-scheduler-master-node.a123mc.ru            1/1     Running   2          15m
```

## 4.3. Проверить статус всех нод
### Проверяем, что мастер- и воркер-ноды перешли в статус Ready, это значит, что сеть настроена. Если долго висит NotReady (больше 5 минут), то VM следует перезагрузить
```bash
kubectl get no
```
```console
    NAME                     STATUS   ROLES           AGE     VERSION
    worker-node1.a123mc.ru   Ready    <none>          6m6s    v1.32.2
    worker-node2.a123mc.ru   Ready    <none>          5m56s   v1.32.2
    worker-node3.a123mc.ru   Ready    <none>          5m50s   v1.32.2
    master-node.a123mc.ru    Ready    control-plane   16m     v1.32.2
```


# Кластер Kubernetes из одной мастер-ноды и трех воркер-нод создан и настроен, установлен CNI, все ноды в статусе Ready
