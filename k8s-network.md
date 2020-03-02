# docker 网络

## 组网模块
* Bridge（网桥） - 它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端
口（Port）上。
* Veth Pair 虚拟设备 - 它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出
现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网
卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。

![docker通信图](image/network/docker0.jpg)
