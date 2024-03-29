# 容器运行时类（Runtime Class）

RuntimeClass 是一个用于选择容器运行时配置的特性，容器运行时配置用于运行 Pod 中的容器。

## 动机

你可以为不同的 Pod 设置不同的 RuntimeClass，以提供性能与安全性之间的平衡。

例如：

- 如果你的部分工作负载需要高级别的信息安全保证，你可以决定在调度这些 Pod 时尽量使它们在使用硬件虚拟化的容器运行时中运行。

这样，你将从这些不同运行时所提供的额外隔离中获益，代价是一些额外的开销。

你还可以使用 RuntimeClass 运行具有相同容器运行时但具有不同设置的 Pod。

## 设置

主要分为两步：

1. 在节点上配置 CRI 的实现（取决于所选用的运行时）
1. 创建相应的 RuntimeClass 资源

### 1. 在节点上配置 CRI 实现

RuntimeClass 的配置依赖于`运行时接口（CRI）`的实现。

根据你使用的 CRI 实现，查阅[相关的文档](https://kubernetes.io/zh-cn/docs/concepts/containers/runtime-class/#cri-configuration)来了解如何配置。

**注意：**

- RuntimeClass 假设集群中的节点配置是同构的（换言之，所有的节点在容器运行时方面的配置是相同的）。
- 如果需要支持异构节点，配置方法请参阅[调度](https://kubernetes.io/zh-cn/docs/concepts/containers/runtime-class/#scheduling)。

所有这些配置都具有相应的 handler 名，并被 RuntimeClass 引用。handler 必须是有效的 DNS 标签名。

### 2. 创建相应的 RuntimeClass 资源

在上面步骤 1 中，每个配置都需要有一个用于标识配置的 handler。针对每个 handler 需要创建一个 RuntimeClass 对象。

RuntimeClass 资源当前只有两个重要的字段：

- RuntimeClass 名 (metadata.name) 
- handler (handler)。 

RuntimeClass 资源对象定义如下所示：

```yaml
# RuntimeClass 定义于 node.k8s.io API 组
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  # 用来引用 RuntimeClass 的名字
  # RuntimeClass 是一个集群层面的资源
  name: myclass
# 对应的 CRI 配置的名称
handler: myconfiguration
```

RuntimeClass 对象的名称必须是有效的 DNS 子域名。

**注意：**

- 建议将 RuntimeClass 写操作（create、update、patch 和 delete）限定于集群管理员使用。通常这是默认配置。

## 使用说明

一旦完成集群中 RuntimeClasses 的配置， 你可以在 Pod spec 中指定 runtimeClassName 来使用它。例如:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```

这一设置会告诉 kubelet 使用所指的 RuntimeClass 来运行该 pod。

如果所指的 RuntimeClass 不存在或者 CRI 无法运行相应的 handler，那么 pod 将会进入 **Failed** 终止阶段。你可以查看相应的事件， 获取执行过程中的错误信息。

如果未指定 runtimeClassName，则将使用默认的 RuntimeHandler，相当于禁用 RuntimeClass 功能特性。

### CRI 配置

关于如何安装 CRI 运行时，请查阅 [CRI 安装](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)。

#### containerd

通过 containerd 的 `/etc/containerd/config.toml` 配置文件来配置运行时 handler。

handler 需要配置在 runtimes 块中：

```txt
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.${HANDLER_NAME}]
```

更详细信息，请查阅 [containerd 的配置指南](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)。

#### CRI-O

通过 CRI-O 的 /etc/crio/crio.conf 配置文件来配置运行时 handler。 handler 需要配置在 crio.runtime 表之下：

```txt
[crio.runtime.runtimes.${HANDLER_NAME}]
  runtime_path = "${PATH_TO_BINARY}"
```

更详细信息，请查阅 [CRI-O 配置文档](https://github.com/cri-o/cri-o/blob/main/docs/crio.conf.5.md)。

## 调度

## Pod 开销

特性状态： Kubernetes v1.24 [stable]

你可以指定与运行 Pod 相关的开销资源。声明开销即允许集群（包括调度器）在决策 Pod 和资源时将其考虑在内。

Pod 开销通过 RuntimeClass 的 overhead 字段定义。

通过使用这个字段，你可以指定使用该 RuntimeClass 运行 Pod 时的开销并确保 Kubernetes 将这些开销计算在内。
