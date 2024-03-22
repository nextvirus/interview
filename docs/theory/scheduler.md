## 概述

schedular 是 k8s 的调度工具，调度是指将 Pod 放置到合适的节点上，以便对应节点上的 Kubelet 能够运行这些 Pod。

## kube-scheduler

kube-scheduler 是 Kubernetes 集群的默认调度器，并且是集群 控制面 的一部分。 如果你真得希望或者有这方面的需求，kube-scheduler 在设计上允许你自己编写一个调度组件并替换原有的 kube-scheduler。

Kube-scheduler 选择一个最佳节点来运行新创建的或尚未调度（unscheduled）的 Pod。 由于 Pod 中的容器和 Pod 本身可能有不同的要求，调度程序会过滤掉任何不满足 Pod 特定调度需求的节点。 或者，API 允许你在创建 Pod 时为它指定一个节点，但这并不常见，并且仅在特殊情况下才会这样做。

在一个集群中，满足一个 Pod 调度请求的所有节点称之为可调度节点。 如果没有任何一个节点能满足 Pod 的资源请求， 那么这个 Pod 将一直停留在未调度状态直到调度器能够找到合适的 Node。

调度器先在集群中找到一个 Pod 的所有可调度节点，然后根据一系列函数对这些可调度节点打分， 选出其中得分最高的节点来运行 Pod。之后，调度器将这个调度决定通知给 kube-apiserver，这个过程叫做绑定。

在做调度决定时需要考虑的因素包括：单独和整体的资源请求、硬件/软件/策略限制、 亲和以及反亲和要求、数据局部性、负载间的干扰等等。

## kube-scheduler 节点选择 

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：

- 过滤
- 打分

过滤阶段会将所有满足 Pod 调度需求的节点选出来。 例如，PodFitsResources 过滤函数会检查候选节点的可用资源能否满足 Pod 的资源请求。 在过滤之后，得出一个节点列表，里面包含了所有可调度节点；通常情况下， 这个节点列表包含不止一个节点。如果这个列表是空的，代表这个 Pod 不可调度。

在打分阶段，调度器会为 Pod 从所有可调度节点中选取一个最合适的节点。 根据当前启用的打分规则，调度器会给每一个可调度节点进行打分。

最后，kube-scheduler 会将 Pod 调度到得分最高的节点上。 如果存在多个得分最高的节点，kube-scheduler 会从中随机选取一个。

支持以下两种方式配置调度器的过滤和打分行为：

1. 调度策略 允许你配置过滤所用的 断言（Predicates） 和打分所用的 优先级（Priorities）。
2. 调度配置 允许你配置实现不同调度阶段的插件， 包括：QueueSort、Filter、Score、Bind、Reserve、Permit 等等。 你也可以配置 kube-scheduler 运行不同的配置文件。

### 调度策略

在 Kubernetes v1.23 版本之前，可以使用调度策略来指定 predicates 和 priorities 进程。 例如，可以通过运行 `kube-scheduler --policy-config-file <filename>` 或者 `kube-scheduler --policy-configmap <ConfigMap>` 设置调度策略。

但是从 Kubernetes v1.23 版本开始，不再支持这种调度策略。 同样地也不支持相关的 `policy-config-file`、`policy-configmap`、`policy-configmap-namespace` 和 `use-legacy-policy-config` 标志。 你可以通过使用调度配置来实现类似的行为。


### 调度器配置

你可以通过编写配置文件，并将其路径传给 kube-scheduler 的命令行参数，定制 kube-scheduler 的行为。

调度模板（Profile）允许你配置 kube-scheduler 中的不同调度阶段。每个阶段都暴露于某个扩展点中。插件通过实现一个或多个扩展点来提供调度行为。

你可以通过运行 `kube-scheduler --config <filename>` 来设置调度模板， 使用 KubeSchedulerConfiguration v1 结构体。

最简单的配置如下：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /etc/srv/kubernetes/kube-scheduler/kubeconfig
```

!!!note

    KubeSchedulerConfiguration v1beta3 在 v1.26 中已被弃用， 并将在 v1.29 中被移除。请将 KubeSchedulerConfiguration 迁移到 v1。

#### 配置文件

通过调度配置文件，你可以配置 kube-scheduler 在不同阶段的调度行为。 每个阶段都在一个扩展点中公开。 调度插件通过实现一个或多个扩展点，来提供调度行为。

你可以配置同一 kube-scheduler 实例使用多个配置文件。

##### 扩展点

调度行为发生在一系列阶段中，这些阶段是通过以下扩展点公开的：

1. queueSort：这些插件对调度队列中的悬决的 Pod 排序。 一次只能启用一个队列排序插件。
2. preFilter：这些插件用于在过滤之前预处理或检查 Pod 或集群的信息。 它们可以将 Pod 标记为不可调度。
3. filter：这些插件相当于调度策略中的断言（Predicates），用于过滤不能运行 Pod 的节点。 过滤器的调用顺序是可配置的。 如果没有一个节点通过所有过滤器的筛选，Pod 将会被标记为不可调度。
4. postFilter：当无法为 Pod 找到可用节点时，按照这些插件的配置顺序调用他们。 如果任何 postFilter 插件将 Pod 标记为可调度，则不会调用其余插件。
5. preScore：这是一个信息扩展点，可用于预打分工作。
6. score：这些插件给通过筛选阶段的节点打分。调度器会选择得分最高的节点。
7. reserve：这是一个信息扩展点，当资源已经预留给 Pod 时，会通知插件。 这些插件还实现了 Unreserve 接口，在 Reserve 期间或之后出现故障时调用。
8. permit：这些插件可以阻止或延迟 Pod 绑定。
9. preBind：这些插件在 Pod 绑定节点之前执行。
10. bind：这个插件将 Pod 与节点绑定。bind 插件是按顺序调用的，只要有一个插件完成了绑定，其余插件都会跳过。bind 插件至少需要一个。
11. postBind：这是一个信息扩展点，在 Pod 绑定了节点之后调用。
12. multiPoint：这是一个仅配置字段，允许同时为所有适用的扩展点启用或禁用插件。

对每个扩展点，你可以禁用默认插件或者是启用自己的插件，例如：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:
        disabled:
        - name: PodTopologySpread
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```

你可以在 disabled 数组中使用 * 禁用该扩展点的所有默认插件。 如果需要，这个字段也可以用来对插件重新顺序。

##### 调度插件

下面默认启用的插件实现了一个或多个扩展点：

* ImageLocality：选择已经存在 Pod 运行所需容器镜像的节点。

    实现的扩展点：score。

* TaintToleration：实现了污点和容忍。

    实现的扩展点：filter、preScore、score。

* NodeName：检查 Pod 指定的节点名称与当前节点是否匹配。

    实现的扩展点：filter。

* NodePorts：检查 Pod 请求的端口在节点上是否可用。

    实现的扩展点：preFilter、filter。

* NodeAffinity：实现了节点选择器 和节点亲和性。

    实现的扩展点：filter、score。

* PodTopologySpread：实现了 Pod 拓扑分布。

    实现的扩展点：preFilter、filter、preScore、score。

* NodeUnschedulable：过滤 .spec.unschedulable 值为 true 的节点。

    实现的扩展点：filter。

* NodeResourcesFit：检查节点是否拥有 Pod 请求的所有资源。 得分可以使用以下三种策略之一：LeastAllocated（默认）、MostAllocated 和 RequestedToCapacityRatio。

    实现的扩展点：preFilter、filter、score。

* NodeResourcesBalancedAllocation：调度 Pod 时，选择资源使用更为均衡的节点。

    实现的扩展点：score。

* VolumeBinding：检查节点是否有请求的卷，或是否可以绑定请求的卷。 实现的扩展点：preFilter、filter、reserve、preBind 和 score。

    !!!note
        说明：
             VolumeCapacityPriority 特性被启用时，score 扩展点也被启用。 它优先考虑可以满足所需卷大小的最小 PV。

* VolumeRestrictions：检查挂载到节点上的卷是否满足卷提供程序的限制。

    实现的扩展点：filter。

* VolumeZone：检查请求的卷是否在任何区域都满足。

    实现的扩展点：filter。

* NodeVolumeLimits：检查该节点是否满足 CSI 卷限制。

    实现的扩展点：filter。

* EBSLimits：检查节点是否满足 AWS EBS 卷限制。

    实现的扩展点：filter。

* GCEPDLimits：检查该节点是否满足 GCP-PD 卷限制。

    实现的扩展点：filter。

* AzureDiskLimits：检查该节点是否满足 Azure 卷限制。

    实现的扩展点：filter。

* InterPodAffinity：实现 Pod 间亲和性与反亲和性。

    实现的扩展点：preFilter、filter、preScore、score。

* PrioritySort：提供默认的基于优先级的排序。

    实现的扩展点：queueSort。

* DefaultBinder：提供默认的绑定机制。

    实现的扩展点：bind。

* DefaultPreemption：提供默认的抢占机制。

    实现的扩展点：postFilter。

你也可以通过组件配置 API 启用以下插件（默认不启用）：

  * CinderLimits：检查是否可以满足节点的 OpenStack Cinder 卷限制。 实现的扩展点：filter

##### 多配置文件

你可以配置 kube-scheduler 运行多个配置文件。 每个配置文件都有一个关联的调度器名称，并且可以在其扩展点中配置一组不同的插件。

使用下面的配置样例，调度器将运行两个配置文件：一个使用默认插件，另一个禁用所有打分插件。

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
```

对于那些希望根据特定配置文件来进行调度的 Pod，可以在 .spec.schedulerName 字段指定相应的调度器名称。

默认情况下，将创建一个调度器名为 default-scheduler 的配置文件。 这个配置文件包括上面描述的所有默认插件。 声明多个配置文件时，每个配置文件中调度器名称必须唯一。

如果 Pod 未指定调度器名称，kube-apiserver 将会把调度器名设置为 default-scheduler。 因此，应该存在一个调度器名为 default-scheduler 的配置文件来调度这些 Pod。

!!!note
    说明：
        Pod 的调度事件把 .spec.schedulerName 字段值作为 ReportingController。 领导者选举事件使用列表中第一个配置文件的调度器名称。

!!!note
    说明：
        所有配置文件必须在 queueSort 扩展点使用相同的插件，并具有相同的配置参数（如果适用）。 这是因为调度器只有一个保存 pending 状态 Pod 的队列

##### 应用于多个扩展点的插件

从 `kubescheduler.config.k8s.io/v1beta3` 开始，配置文件配置中有一个附加字段 multiPoint，它允许跨多个扩展点轻松启用或禁用插件。 multiPoint 配置的目的是简化用户和管理员在使用自定义配置文件时所需的配置。

考虑一个插件，MyPlugin，它实现了 preScore、score、preFilter 和 filter 扩展点。 要为其所有可用的扩展点启用 MyPlugin，配置文件配置如下所示：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: multipoint-scheduler
    plugins:
      multiPoint:
        enabled:
        - name: MyPlugin
```

这相当于为所有扩展点手动启用 MyPlugin，如下所示：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: non-multipoint-scheduler
    plugins:
      preScore:
        enabled:
        - name: MyPlugin
      score:
        enabled:
        - name: MyPlugin
      preFilter:
        enabled:
        - name: MyPlugin
      filter:
        enabled:
        - name: MyPlugin
```

在这里使用 multiPoint 的一个好处是，如果 MyPlugin 将来实现另一个扩展点，multiPoint 配置将自动为新扩展启用它。

可以使用该扩展点的 disabled 字段将特定扩展点从 MultiPoint 扩展中排除。 这适用于禁用默认插件、非默认插件或使用通配符 ('*') 来禁用所有插件。 禁用 Score 和 PreScore 的一个例子是：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: non-multipoint-scheduler
    plugins:
      multiPoint:
        enabled:
        - name: 'MyPlugin'
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'
```

从 `kubescheduler.config.k8s.io/v1beta3` 开始，所有默认插件都通过 MultiPoint 在内部启用。 但是，仍然可以使用单独的扩展点来灵活地重新配置默认值（例如排序和分数权重）。 例如，考虑两个 Score 插件 DefaultScore1 和 DefaultScore2，每个插件的权重为 1。 它们可以用不同的权重重新排序，如下所示：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: multipoint-scheduler
    plugins:
      score:
        enabled:
        - name: 'DefaultScore2'
          weight: 5
```

在这个例子中，没有必要在 MultiPoint 中明确指定插件，因为它们是默认插件。 Score 中指定的唯一插件是 DefaultScore2。 这是因为通过特定扩展点设置的插件将始终优先于 MultiPoint 插件。 因此，此代码段实质上重新排序了这两个插件，而无需同时指定它们。

配置 MultiPoint 插件时优先级的一般层次结构如下：

1. 特定的扩展点首先运行，它们的设置会覆盖其他地方的设置
2. 通过 MultiPoint 手动配置的插件及其设置
3. 默认插件及其默认设置

为了演示上述层次结构，以下示例基于这些插件：

|插件|扩展点|
|---|---|
| DefaultQueueSort | QueueSort |
| CustomQueueSort  | QueueSort |
| DefaultPlugin1   | Score, Filter |
| DefaultPlugin2   | Score |
| CustomPlugin1	   | Score, Filter |
| CustomPlugin2	   | Score, Filter |

这些插件的一个有效示例配置是：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: multipoint-scheduler
    plugins:
      multiPoint:
        enabled:
        - name: 'CustomQueueSort'
        - name: 'CustomPlugin1'
          weight: 3
        - name: 'CustomPlugin2'
        disabled:
        - name: 'DefaultQueueSort'
      filter:
        disabled:
        - name: 'DefaultPlugin1'
      score:
        enabled:
        - name: 'DefaultPlugin2'
```

请注意，在特定扩展点中重新声明 MultiPoint 插件不会出错。 重新声明被忽略（并记录），因为特定的扩展点优先。

除了将大部分配置保存在一个位置之外，此示例还做了一些事情：

- 启用自定义 queueSort 插件并禁用默认插件
- 启用 CustomPlugin1 和 CustomPlugin2，这将首先为它们的所有扩展点运行
- 禁用 DefaultPlugin1，但仅适用于 filter
- 重新排序 DefaultPlugin2 以在 score 中首先运行（甚至在自定义插件之前）

在 v1beta3 之前的配置版本中，没有 multiPoint，上面的代码片段等同于：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: multipoint-scheduler
    plugins:

      # 禁用默认的 QueueSort 插件
      queueSort:
        enabled:
        - name: 'CustomQueueSort'
        disabled:
        - name: 'DefaultQueueSort'

      # 启用自定义的 Filter 插件
      filter:
        enabled:
        - name: 'CustomPlugin1'
        - name: 'CustomPlugin2'
        - name: 'DefaultPlugin2'
        disabled:
        - name: 'DefaultPlugin1'

      # 启用并重新排序自定义的打分插件
      score:
        enabled:
        - name: 'DefaultPlugin2'
          weight: 1
        - name: 'DefaultPlugin1'
          weight: 3
```

虽然这是一个复杂的例子，但它展示了 MultiPoint 配置的灵活性以及它与配置扩展点的现有方法的无缝集成。