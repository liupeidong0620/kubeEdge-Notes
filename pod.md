# Projected Volume
k8s 中有几种特殊的Volume存在的意义不是为了存放容器里的数据，也不是为了用来进行容器和宿主机器之间的数据交换。
这些特殊的Volume作用，是为了容器提供预先定义好的数据。
从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投
射”（Project）进入容器当中的。这正是 Projected Volume 的含义。
## Kubernetes 支持的 Projected Volume 一共有四种：
* Secret
* ConfigMap
* Downward API
* ServiceAccountToken

## Secret

它的作用，是帮你把 Pod 想要访问的加密数据，
存放到 Etcd 中。然后，你就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret
里保存的信息了。
Secret 最典型的使用场景，莫过于存放数据库的 user pass信息

## ConfigMap

与 Secret 类似的是 ConfigMap，它与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、
应用所需的配置信息。而 ConfigMap 的用法几乎与 Secret 完全相同：你可以使用 kubectl create
configmap 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件。

## Downward API
API，它的作用是：让 Pod 里的容器能够直接获取到这个 Pod API 对象本身
的信息。

## ServiceAccountToken
ServiceAccountToken，只是一种特殊的 Secret 而已。

在一个 Pod 里安装一个 Kubernetes的 Client，这样就可以从容器里直接访问并且操作这个 Kubernetes 的 API 了。
ServiceAccountToken 就是 为了解决默认api 授权问题。

* 默认容器中的存放位置
```sh
 # ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt     namespace  token
```

# Probe (探针)

Pod 的另一个重要的配置：容器健康检查和恢复机制。

# PodPreset（Pod 预设置）

给 pod 的字段特别多，每一次 都要设置一遍太浪费时间，比如一些公共设置，可以设置预先设置好。
