controller-manager 是 K8S 的控制枢纽，在我们安装好 k8s 之后，controller-manager 中包含了很多控制平面，详情可以点击[链接](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller)，其中最具有代表性的功能如下：

- replication-controller-manager
- daemonset-controller-manager
- endpoint-controller-manager
- deployment-controller-manager

kube-controller-manager 就是控制以上 controller-manager 进行正常运作，而下面的 replication-controller-manager，daemonset-controller-manager 和 endpoint-controller-manager 这些控制层，他们的主要作用就是对相关服务的增删改查。