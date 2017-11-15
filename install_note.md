# 安装前准备：

1. 使用虚拟机
1. 使用RHEL 7.3
1. 各节点 Hostname、MAC地址和product_uuid（/sys/class/dmi/id/product_uuid）要唯一
1. 各节点关闭 Swap：在/etc/fstab中将Swap行注释掉
1. 各节点关闭防火墙：systemctl disable firewalld
1. 各节点关闭 Selinux：编辑/etc/selinux/config
1. 为了使用方便，建议各节点安装bash-completion：yum install bash-completion
1. 各节点重启
1. 使用Docker 1.12.6（目前K8s对该版本的测试最为充分）和K8s 1.8.1

# 开始安装：

## 在各节点安装Docker

以下操作需要在各节点分别进行，使用 root 用户

1. yum install docker（**此时不要启动Docker**）
1. 配置 Direct-lvm：
    1. yum install lvm2
    1. 可能需要安装 thin-provisioning-tools，CentOS的对应包是device-mapper-persistent-data，RHEL待确认
    1. 添加一个新盘，并配置 Thinpool，步骤为：
        ```bash
        pvcreate /dev/sdb
        vgcreate docker /dev/sdb
        lvcreate --wipesignatures y -n thinpool docker -l 95%VG
        lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
        lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
        ```
    1. 在 /etc/lvm/profile/docker-thinpool.profile 中添加：
        ```bash
        activation {
        thin_pool_autoextend_threshold=80
        thin_pool_autoextend_percent=20
        }
        ```
    1. 执行 lvchange --metadataprofile docker-thinpool docker/thinpool，使 autoextend 策略生效
    1. 执行 lvs -o+seg_monitor，确认 Thinpool 创建成功
    1. 为 Docker 配置 devicemapper option，在 /etc/docker/daemon.json 中添加： 
        ```bash
        {
        "storage-driver": "devicemapper",
        "storage-opts": [
                "dm.thinpooldev=/dev/mapper/docker-thinpool",
                "dm.use_deferred_removal=true",
                "dm.use_deferred_deletion=true"
                ]
        }
        ```
1. 启动 Docker：systemctl enable docker && systemctl start docker
1. 执行 journalctl -fu dm-event.service，确认 devicemapper 运行正常
1. 执行 lsblk 确认卷结构配置正确
1. 执行 docker info 确认 Docker 正在使用Thinpool，输出中需要有如下内容：
    ```bash
    ...
    ...
    Storage Driver: devicemapper
    Pool Name: docker-thinpool
    ...
    ...
    ```
    
**上述步骤的详细说明参见 https://docs.docker.com/v1.12/engine/userguide/storagedriver/device-mapper-driver/**

**上述配置使用了Thinpool方式，生产上可能会不稳定，非Thinpool配置方式我还没来得及试，你们如果有时间可以试一下，谢谢**

**后续步骤中需要从墙外下载大量Docker镜像，如果翻墙不便，可以实现将我提供的镜像导入到本地镜像库中**

## 在各节点安装kubeadm、kubelet和kubectl

以下操作需要在各节点分别进行，使用root用户

1. 添加K8s仓库（该库在墙外）：
    ```bash
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
    ```
1. yum install kubelet kubeadm kubectl
1. 配置 k8s autocompletion：echo "source <(kubectl completion bash)" >> ~/.bashrc
1. systemctl enable kubelet && systemctl start kubelet
1. 配置 iptables:
    ```bash
    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl --system
    ```

# 创建 K8s 集群

## 配置 K8s Master 节点

以下操作在 Master 节点上进行，使用 root 用户

1. 执行 kubeadm init --kubernetes-version=1.8.1 --pod-network-cidr=10.244.0.0/16，过程中会从墙外下载大量镜像，如果执行失败，则需要先进行 Teardown，才能再次执行 kubeadm init，所以init之前建议先抓个虚机快照以备份环境；如果执行成功，会输出如下内容：
    ```bash
    ....
    ....
    Your Kubernetes master has initialized successfully!
    ....
    ....
    You can now join any number of machines by running the following on each node as root:
    kubeadm join --token 635bda.161f8b136c04ba23 10.66.65.192:6443 --discovery-token-ca-cert-hash sha256:9e69fb0a246ab60c4d1686e0c60d8e1e3c6eee759d58d72d7a54a072cec35883
    ```
1. 安装 Flannel：kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.0/Documentation/kube-flannel.yml
1. 为 root 用户添加环境变量 KUBECONFIG=/etc/kubernetes/admin.conf

## 添加工作节点：

以下操作分别在各工作节点上进行，使用 root 用户

1. kubeadm join....，完整命令参见kubeadm init的输出

**上述步骤的说明详见 https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/**

**上述安装方式启用了RBAC**

# 安装辅助工具

## 安装 Dashboard

1. kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml (该 YAML 启用了RBAC)

**详见 https://github.com/kubernetes/dashboard**

## 安装 Heapster、Grafana 和 Influxdb

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

**详见 https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md**

## 安装 Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml
```

**详见 https://github.com/kubernetes/ingress-nginx/blob/master/deploy/README.md**

## 安装 Microservice Demo App

1. kubectl create namespace sock-shop
2. kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"

## 安装 Weave Scope

1. kubectl apply --namespace kube-system -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"

1. 如果需要 Nodeport 访问，则执行：kubectl apply --namespace kube-system -f "https://cloud.weave.works/k8s/scope.yaml?k8s-service-type=NodePort&k8s-version=$(kubectl version | base64 | tr -d '\n')"

**详见 https://www.weave.works/docs/scope/latest/installing/#k8s**




