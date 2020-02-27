# 1. k8s 安装文档
* 一个master节点
* 一个node节点
* 两台虚拟机一个运行master节点，一个运行node节点
* 系统: CentOS Linux release 7.7.1908 (Core)
* 目前安装版本 k8s 1.17.3

# 2. 安装步骤
## 2.1 基础检查

安装kubeadm之前，需要确保所有主机满足以下条件:
* 运行支持的操作系统，本文选择使用 CentOS 7搭建
* 内存不低于 2G，CPU 不少于 2核
* 集群中不同主机之间保证网络连通性
* 唯一的 hostname、MAC 地址、product_uuid
* 相关端口开放
* swap 已被禁用
* 所有机器时间一致

## 2.1.1 禁用swap分区
```sh
swapoff -a #关闭交换分区
sed -i '/ swap / s/^/#/' /etc/fstab #禁止重启后自动开启
```
## 2.1.2 关闭防火墙
```sh
systemctl stop firewalld
systemctl disable firewalld
```

## 2.1.3 关闭SELinux
```sh
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## 2.1.4 网络参数配置
某些网络参数可能会导致问题，按官方文档推荐调整
```sh
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# 确保 br_netfilter 模块已经加载
modprobe br_netfilter
```

## 2.1.4 修改hostname
```sh
# 自定义名字
hostnamectl set-hostname xxxx

# 修改/etc/hosts
x.x.x.x xxxx
```

## 2.1.5 校对时间
```
yum install ntp
ntpdate pool.ntp.org
date
# 启动ntpd daemon
systemctl start ntpd
```

## 2.1.6 更换yum源为国内源（墙内用户,可以访问外网，无需操作）
```sh
cd /etc/yum.repos.d  && \
sudo mv CentOS-Base.repo CentOS-Base.repo.bak && \
sudo wget -O CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo && \
yum clean all && \
yum makecache
```

# 2.2 配置容器环境
## 2.2.1 安装Docker环境
K8s 同时支持 Docker、CRI-O、Containered 等多种容器环境，我们使用 Docker，安装过程如下：
```sh
# Install required packages.（安装依赖包）
yum install yum-utils device-mapper-persistent-data lvm2

# Add Docker repository.（添加 Docker 库）
yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker CE.（安装 Docker CE）
yum update && yum install \
  containerd.io-1.2.10 \
  docker-ce-19.03.4 \
  docker-ce-cli-19.03.4

# Create /etc/docker directory.（创建 /etc/docker 目录）
mkdir /etc/docker

# Setup daemon.（配置 daemon）
cat > /etc/docker/daemon.json <<-EOF
{
  "registry-mirrors": [
    "https://a8qh6yqv.mirror.aliyuncs.com",
    "http://hub-mirror.c.163.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
# registry-mirrors 为镜像加速器地址。
# native.cgroupdriver=systemd 表示使用的 cgroup 驱动为 systemd（k8s 使用此方式），默认为 cgroupfs。修改原因是 kubeadm.conf 中修改k8s的驱动方式不成功。
# 创建 docker.service.d
systemctl enable docker.service

# Restart Docker（重启 Docker）
systemctl daemon-reload
systemctl restart docker

# k8s 要求安装 cgroup
# systemctl restart docker
# 查看是否使用systemd
docker info | grep -i cgroup
# 结果
Cgroup Driver: systemd

# 查看版本
docker -v
```

# 2.3 安装kubeadm

kubeadm 安装过程会自动安装 kubelet kubectl kubernetes-cni
kubeadm 负责引导集群，kubelet 在集群的所有节点运行，负责启动 pods 和 containers，kubectl 则负责与集群交互，我们需要在所有节点安装这些组件。

# 2.3.1 配置k8s源
# 2.3.1.1 配置官方源（墙为用户）
```sh
# 配置官方源
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
## 2.3.1.2 配置国内源（墙内用户）
```sh
# 配置国内源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
## 2.3.2 安装 启动
```sh
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

注: 以上步骤 node & master节点都需要

# 2.4 使用kubeadm初始化集群
## 2.4.1 拉取依赖镜像(墙内用户)
```sh
# 这一步可以不做，只是展示需要的 镜像
# 获取依赖镜像列表
kubeadm config images list

# 使用阿里源下载 K8s 依赖镜像
kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' |sh -x

docker images |grep registry.cn-hangzhou.aliyuncs.com/google_containers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#registry.cn-hangzhou.aliyuncs.com/google_containers#k8s.gcr.io#2' |sh -x

docker images |grep registry.cn-hangzhou.aliyuncs.com/google_containers |awk '{print "docker rmi ", $1":"$2}' |sh -x
```
## 2.4.2 master 节点初始化
```sh
kubeadm init --kubernetes-version=1.17.3 --image-repository registry.aliyuncs.com/google_containers

# -image-repository 指定阿里云镜像，下载master节点需要的镜像
```
执行成功后，按照提示操作即可
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

初始化完成后会自动创建一个 token，记录下面加入节点的命令，24小时内可使用该 token 向集群添加节点：
```sh
kubeadm join 192.168.198.129:6443 --token wdagmm.0he40pufw18d1n68 \
    --discovery-token-ca-cert-hash sha256:ba6ba29500a883fe4eef4d2ab575803bc6360c8c73ef03c21d7413ae027d0391
```
# 2.5 添加网络插
* [官方插件说明文档](https://kubernetes.io/zh/docs/concepts/cluster-administration/addons/)
* master节点必须，安装网络插件
* 此时我们使用kubectl get nodes查看集群节点运行情况，会发现节点始终是 NotReady，因为我们没有安装必要的网络组件，可参考参考资料4的官方文档进行选择，这里选择比较精致的 WeaveNet，根据文档进行安装，稍等几分钟后集群就正常运行了：

```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

# 2.6 集群添加node 节点
* 2.1 ~ 2.3 全部执行一遍
```sh
kubeadm join 192.168.198.129:6443 --token wdagmm.0he40pufw18d1n68 --discovery-token-ca-cert-hash sha256:ba6ba29500a883fe4eef4d2ab575803bc6360c8c73ef03c21d7413ae027d0391
```
执行结果
```sh
[root@localhost yum.repos.d]# kubeadm join 192.168.198.129:6443 --token wdagmm.0he40pufw18d1n68 \
>     --discovery-token-ca-cert-hash sha256:ba6ba29500a883fe4eef4d2ab575803bc6360c8c73ef03c21d7413ae027d0391
W0227 14:11:13.980208   50524 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 2.6.1 从master节点查看 集群状态
```sh
[root@localhost yum.repos.d]# kubectl get nodes
NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   13m   v1.17.3
node1     Ready    <none>   64s   v1.17.3
```

# 2.7 部署一个pod
创建一个最简单的pod，使用 busybox 镜像简单测试 pod:
```sh
kubectl run -i --tty busybox --image=latelee/busybox --restart=Never -- sh

```
稍等片刻，即可进入 busybox 命令行：

```sh
[root@localhost yum.repos.d]# kubectl run -i --tty busybox --image=latelee/busybox --restart=Never -- sh
If you don't see a command prompt, try pressing enter.
/ # ls
author.txt  dev         home        lib64       root        tmp         var
bin         etc         lib         proc        sys         usr
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
12: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1376 qdisc noqueue
    link/ether 62:15:b8:99:ce:85 brd ff:ff:ff:ff:ff:ff
    inet 10.44.0.1/12 brd 10.47.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ #
```
在开启一个命令行窗口，查看pods状态
```sh
[root@master1 ~]# kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          104s
```
# TODO
* 网络插件研究



