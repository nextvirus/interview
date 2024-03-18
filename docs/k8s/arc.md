## k8s 架构

![](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

上图为 k8s 的架构图，是目前官方给出的最新架构图。主要工具为 Cloud-control-manage、etcd、kube-api-server、scheduler 、 Contriller Manager、kubelet 和 kube-proxy。

在最新的架构图中，没有强调 master 和 worker 节点之间的区别，取而代之的就是引入了 Cloud-control-manager 这个组件。Cloud-contorl-manager 这个工具主要传播的理念为 Controller As Service，主要代表的开源项目是 Kamaji。它的主要理念是在搭建 Kubernetes 集群时，如何像公有云提供商那样快速生成一个 k8s 的控制平面，比如 AWS 的 EKS、阿里云的 ACK 和 谷歌的 GKE 。

接着就是控制层面所包含的组件：

- k8s 的主要访问接口 kube-api-server，在通常情况下 kube-api-server 为了保证 k8s 的高可用，要保证其的高可用。
- k8s 的持久化存储 etcd，在通常情况下为了数据的安全和持久化存储，etcd 也通常是需要高可用的，而 etcd 使用算法是 raft 的分布式算法，所以在扩展的时候需要进行单数扩展，比如：3、5、7、9 这样。
- k8s 的调度组件 kube-schdular，主要负责整个集群的资源调度，指定 Pod 运行的节点。指定的方法是：
  - 判断资源容量
  - 判断优先级
  - 判断容忍度、亲和度和污点
- controller-manager: k8s 的管理层面，k8s 内置了多种 controller，比如 service-controller、replica-controller、endpoinrt-controller 和 deployment-controoler 等等。直接管理各个服务的增删改查。

工作层包含的组件：

- kubelet：k8s agent 直接管理 Pod 的组件。
- kubeproxy：在之前直接作为 k8s 的负载均衡器，在现在主要管理 ipvs 或者 iptables 规则进行流量转发。主要利用内核的 netfilter 模块实现，未来可能会使用 eBPF 去实现。


其他组件：

- cni 插件：网络管理插件，目前比较流行的有：flannel、calico 和 cilium 等。
- csi 插件：存储管理插件，目前比较流行的有：efs-csi 和 openEBS等。
- cri ：目前比较流行是 containerd，如果想使用 docker，需要安装 cri-o。

