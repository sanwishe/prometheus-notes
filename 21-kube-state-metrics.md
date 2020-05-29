## kubernetes监控体系架构

### 核心通道

由kubelet、metrics-server和apiserver的metrics api等构成，这个监控通道用于kubernetes的核心组件，不依赖于第三方监控组件，如调度或者kubectl top等。
### 监控通道

用于收集各种对用户暴露的指标，包括用于HPA的自定义指标等，基于核心指标，用户可以自行选择第三方的监控系统。
![monitoring system of kubernetes](https://upload-images.jianshu.io/upload_images/1889435-6ef55c32828a19af.png?imageMogr2/auto-orient/strip|imageView2/2/w/960)

- 核心流程（黑色部分）：这是 Kubernetes正常工作所需要的核心度量，从 Kubelet、cAdvisor 等获取度量数据，再由metrics-server提供给 Dashboard、HPA 控制器等使用。
- 监控流程（蓝色部分）：基于核心度量构建的监控流程，比如 Prometheus 可以从 metrics-server 获取核心度量，从其他数据源（如 Node Exporter 等）获取非核心度量，再基于它们构建监控告警系统


## kube-state-metrics

### overview

- 监控对象：kubernetes集群上的资源对象，如node、pod、service等。
- 工作原理：使用client-go和kubernetes apiserver交互，list/watch kubernetes 对象得到指标，通过http暴露prometheus 文本格式的指标。
- 认证方式：kubecofig或者serviceaccount。

#### 和metrics-server的比较

- metrics-server

取代heapster，是kubernetes核心指标的聚合器，它关注于cpu、内存使用率这种监控指标。metrics-server采用kubernetes metrics api(和其他的Kubernetes APIs一样)暴露指标。
metrics-server的工作原理是定是调用kubelet的Summary API获取指标，然后对指标进行汇聚。然后通过apiserver以metrics api格式(/apis/metrics.k8s.io/)对外暴露。

- kube-state-metrics

用于监控kubernetes上的资源对象，如node、pod、service等。通过prometheus 文本格式对外暴露指标。它关注于获取k8s各种资源的最新状态。

### 指标

#### 按阶段分类

| Stage        | Description           |
| ------------ | ------------------ |
| EXPERIMENTAL | 实验性质的：k8s api中alpha阶段的或者spec的字段。可能随时更改。 |
| STABLE       | 稳定版本的：k8s中不向后兼容的主要版本的更新。  |
| DEPRECATED   | 被废弃的：已经不在维护的。这些指标随时会被清除掉。  |

#### 指标详解

略

### 使用

#### 资源要求

常规需求：

- 200MiB memory
- 0.1 cores

当node超过100时，每增加一个node，需要额外：

- 2MiB memory per node
- 0.001 cores per node

#### 重要启动参数

| No. | Name | Type | Description |
|:---:|:---:|:---:|:---:|
| 1 | --port | int | 暴露指标的端口，默认80 |
| 2 | --telemetry-port | int | 自监控指标暴露的端口，默认81 |
| 3 | --apiserver | string | 集群kube-apiserver的地址 |
| 4 | --kubeconfig | string | kubeconfig文件的绝对路径 |

对于kubeconfig认证方式，因为kubeconfig文件中已有apiserver地址，此时--apiserver参数可以忽略。

如果--apiserver和 --kubeconfig都为空，则系统认为当前kube-state-metrics以pod部署在系统内部，通过环境变量(KUBERNETES_SERVICE_HOST和KUBERNETES_SERVICE_PORT)发现kubernetes service，并通过serviceaccount认证。

#### 部署

官方提供了基于serviceaccount认证的部署方式，将kube-state-metrics以deployment部署，并通过headless服务对外暴露pod IP地址。

![official deployment yamls](https://icenterapi.zte.com.cn/group2/M00/7D/5F/Ch-0XV4Jzb-AcDKsAAAntI5Q3Gc871.png)

所需权限：[ClusteRole、对应资源的watch和list权限](https://github.com/kubernetes/kube-state-metrics/blob/master/examples/standard/cluster-role.yaml)。

此外，还可以基于kubeconfig认证部署kube-state-metrics。通过启动时指定kubeconfig绝对路径来发现apiserver和认证。这种方式适于以进程或者静态pod方式部署到master节点上。


## 问题及解决

问题：在穿刺过程中发现日志报错：

```
E1229 01:20:51.898553       1 reflector.go:156] pkg/mod/k8s.io/client-go@v0.0.0-20191109102209-3c0d1af94be5/tools/cache/reflector.go:108: Failed to list *v1.Node: unexpected EOF
E1229 01:20:52.902618       1 reflector.go:156] pkg/mod/k8s.io/client-go@v0.0.0-20191109102209-3c0d1af94be5/tools/cache/reflector.go:108: Failed to list *v1.Node: unexpected EOF
E1229 01:20:53.906819       1 reflector.go:156] pkg/mod/k8s.io/client-go@v0.0.0-20191109102209-3c0d1af94be5/tools/cache/reflector.go:108: Failed to list *v1.Node: unexpected EOF
```

并且暴露的metrics中，kube-node*指标缺失。

经查，内部kubernetes曾经修改过node对象，增加了`DeviceInfo`，而且这个`Deviceinfo`字段占用了社区版本的其它字段的序号11，导致外部client-go和内部kubernetes版本不兼容，watch node资源失败。

```go
// NodeStatus is information about the current status of a node.
type NodeStatus struct {
    ... ...

    // Devices info
    // +optional
    DeviceInfo NodeDeviceInfo `json:"deviceInfo,omitempty" protobuf:"bytes,11,rep,name=deviceInfo"`
    // Status of the config assigned to the node via the dynamic Kubelet config feature.
    // +optional
    Config *NodeConfigStatus `json:"config,omitempty" protobuf:"bytes,12,opt,name=config"`
}
```

解决方法:
找到与当前kube-state-metrics兼容的k8s.io/api版本，然后找到与该k8s.io/api版本兼容的kubernetes版本，修改node对象增加上述deviceInfo后，重新编译生成k8s.io/api包，将该package替换到kube-state-metrics的vendor下即可。

## pod事件监控

###  pod生命周期

- 指标名称：kube_pod_status_phase
阶段（label：phase）：Pending|Running|Succeeded|Failed|Unknown
 
- 指标名称： kube_pod_status_scheduled
调度状态(label: condition)：true|false|unknown

- 指标名称： kube_pod_status_unschedulable

- 指标名称： kube_pod_status_ready
ready状态(label: condition)： true|false|unknown
用来指示readiness探针的就绪状态

### 业务容器生命周期

- 指标名称: kube_pod_container_status_waiting_reason
等待原因枚举包括： ContainerCreating|CrashLoopBackOff|ErrImagePull|ImagePullBackOff|CreateContainerConfigError|InvalidImageName|CreateContainerError

- 指标名称： kube_pod_container_status_ready

- 指标名称： kube_pod_container_status_terminated

- 指标名称： kube_pod_container_status_terminated_reason
其中容器终结原因枚举值包括：OOMKilled|Error|Completed|ContainerCannotRun|DeadlineExceeded

- 指标名称：kube_pod_container_status_last_terminated_reason

- 指标名称： kube_pod_container_status_restarts_total


### 初始化容器生命周期

- 指标： kube_pod_init_container_status_ready

- 指标： kube_pod_init_container_status_terminated_reason

其他如kube_pod_init_container_status_waiting_reason等，指标设置和业务容器类似，不再举例。

### 总结

kube-state-metrics能够监控大多数pod生命周期的各种状态变迁和事件，包括pod级别的状态变迁和业务容器、初始化容器的各种事件。

其中pod级别可观测的事件包括创建、调度、就绪阶段的状态变迁：
- kube_pod_status_phase，pod状态变迁
- kube_pod_status_ready，pod就绪与否
- kube_pod_status_scheduled/kube_pod_status_unschedulable，pod调度状态

容器级别的事件，包括容器的创建等待、运行以及终结阶段的事件：

- kube_pod_container_status_ready/running/waiting/terminated，容器的就绪、运行、等待、终结状态
- kube_pod_container_status_waiting_reason，**容器处于waiting状态的原因**，原因枚举：ContainerCreating|CrashLoopBackOff|ErrImagePull|ImagePullBackOff|CreateContainerConfigError|InvalidImageName|CreateContainerError
- kube_pod_container_status_terminated_reason，**容器terminate原因**，原因枚举：OOMKilled|Error|Completed|ContainerCannotRun|DeadlineExceeded
- kube_pod_container_status_restarts_total，容器重启次数，counter类型指标
- kube_pod_container_status_last_terminated_reason，容器最后一次terminate原因

**需要注意的问题**：

- **prometheus是基于指标采样的监控系统，如果采样周期大于事件持续时间，即事件发生在两次采样点之间，prometheus无法通过单一指标感知事件的发生，比如容器启动的ContainerCreating状态。**
- **pod删除过程中，可以观察到极其短暂的pod及容器状态的跃迁，pod删除后指标不再上报。对pod删除事件的监控可以说是不可用。**
