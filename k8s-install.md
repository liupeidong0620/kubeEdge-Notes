# 1. k8s 安装文档
本次安装单master 和 一个node节点

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

## 更换yum源为国内源（墙内用户,可以访问外网，无需操作）
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
# docker info | grep -i cgroup
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

# 2.4 使用kubeadm初始化集群
## 2.4.1 拉取依赖镜像(墙内用户)
```sh
# 获取依赖镜像列表
kubeadm config images list

# 使用阿里源下载 K8s 依赖镜像
kubeadm config images list |sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' |sh -x

docker images |grep registry.cn-hangzhou.aliyuncs.com/google_containers |awk '{print "docker tag ",$1":"$2,$1":"$2}' |sed -e 's#registry.cn-hangzhou.aliyuncs.com/google_containers#k8s.gcr.io#2' |sh -x

docker images |grep registry.cn-hangzhou.aliyuncs.com/google_containers |awk '{print "docker rmi ", $1":"$2}' |sh -x
```
## 2.4.2 master 节点初始化
```sh
kubeadm init --kubernetes-version=1.17.3
```
执行成功后，按照提示操作即可
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
初始化完成后会自动创建一个 token，记录下面加入节点的命令，24小时内可使用该 token 向集群添加节点：
# 2.5 添加网络插件
此时我们使用kubectl get nodes查看集群节点运行情况，会发现节点始终是 NotReady，因为我们没有安装必要的网络组件，可参考参考资料4的官方文档进行选择，这里选择比较精致的 WeaveNet，根据文档进行安装，稍等几分钟后集群就正常运行了：

```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

# 2.6 想集群添加node 节点



