# 概述

k8s中最小的调度单位是pod，真实的一个应用服务一般是一个pod来抽象表达。
本文仔细讲述了从k8s中访问pod 的几种方式, 列表如下：

* hostNetwork
* hostPort
* NodePort
* LoadBalancer
* Ingress
