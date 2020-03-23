# docker 网络

## 组网模块
* Bridge（网桥） - 它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端
口（Port）上。
* Veth Pair 虚拟设备 - 它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出
现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网
卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。

![docker通信图](image/network/docker0.jpg)

## 跨主机通信

* Overlay Network（覆盖网络）
![docker通信图](image/network/docker1.jpg)

# 容器跨主机网络

k8s 各种网络插件，主要是为了容器跨主机通信方案。

## Flannel

* VXLAN
* host-gw
* UDP

### udp 模式

* flannel0 - 就是一个TUN设备(Tunnel 设备)，就是修改ip包。

![udp 模式跨主机通信原理](image/network/Flannel/udp.jpg)

# service


![iptables 模式](image/network/)

## service访问pod原理图

![service访问pod](image/network/iptables/k8s-network.png)

* 1. curl访问service ip:port(10.100.2.5:80), 请求数据进入Node1 eth0网卡
* 2. 开始执行iptables规则，prerouting链规则，根据规则随机选择一个pod ip，iptables进行DNAT，修改ip报文目的地址。
* 3. 开始判断目的ip是否是本机
* 4. 数据从forward哪里转发到是其它node
* 5. 通过网络插件，将包转发到对应的pod中
