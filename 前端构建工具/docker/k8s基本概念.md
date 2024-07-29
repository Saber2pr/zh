# k8s 基本概念

[k8s基本概念](http://www.dockone.io/article/932)

Cluster 集群，由一个 Master 控制的一组机器

Master 主节点，控制中心机器

api-server 操作集群的 API

Replication Controller 多实例保活

[Node 部署机器](https://www.cnblogs.com/fengdejiyixx/p/15568056.html)

kubelet 主节点代理

kube-proxy 和 Service 链接

pod 一个应用，重启/滚动更新

containers 一个应用的多个容器实例，共享 localhost:port

app 应用容器实例

istio-proxy

label 标签，用来 selector 批量选择

[Service 控制一组 pod 的对集群内/外 IP 分配](https://www.cnblogs.com/moonlight-lin/p/14553119.html)

NodePort 节点机器 IP 端口，外部访问机器 externalIP:nodePort，解析到 pod:targetPort

ClusterIP 默认类型，集群内 IP，访问 clusterIP:port，解析到 pod:targetPort

LoadBalancer 公网 IP，解析到 pod:NodePort

Services without selectors 对接 Endpoints 可以转发到外部 IP

StatefulSet 定义有状态 pod，可对接 Headless

Headless Services 绕过 Service 转发，直接访问 StatefulSet 的 pod:NodePort

ExternalName 将一个 Service 映射到域名

Ingress 将 Service 映射到公网 IP/域名

[k8s插件工具](https://www.kubernetes.org.cn/kubernetes%E8%AE%BE%E8%AE%A1%E6%9E%B6%E6%9E%84)

kube-dns

Ingress Controller

Heapster

Dashboard

Federation

Fluentd-elasticsearch

# FAQ

部署的两个 pod 之间无法访问

可能是都没有暴露到公网，都是内部域名，然后内部域名不在一个集群

同一个集群下的内部域名 ClusterIP 是可以互相访问的

# 资料

[k8s网络模型](https://www.cnblogs.com/fengdejiyixx/p/15568056.html)

[k8s核心概念](http://www.dockone.io/article/932)

[k8s架构](https://www.kubernetes.org.cn/kubernetes%E8%AE%BE%E8%AE%A1%E6%9E%B6%E6%9E%84)

[istio 微服务，jaeger链路](https://docs.k8stech.net/kubernetes-he-istio-wei-fu-wu-jia-gou)

[k8s配置ingress](https://www.jianshu.com/p/c726ed03562a)

[port、targetPort、nodePort区别](https://www.cnblogs.com/devilwind/p/8881201.html)

[Kubernetes的三种外部访问方式：NodePort、LoadBalancer 和 Ingress](https://dockone.io/article/4884)
