# 污点和容忍度

**节点亲和性**是 Pod 的一种属性，它使 Pod 被吸引到一类特定的节点（这可能出于一种偏好，也可能是硬性要求）。

**污点（Taint）**则相反，它使节点能够排斥一类特定的 Pod（相当于是 Node 的反亲和性）：

- 节点但凡被配置了污点，就不能调度 Pod 了。
- **容忍度（Toleration）**是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。容忍度允许调度但并不保证调度。

污点和容忍度（Toleration）相互配合，可以用来避免 Pod 被分配到不合适的节点上。每个节点上都可以应用一个或多个污点，这表示对于那些不能容忍这些污点的 Pod， 是不会被该节点接受的。

## 概念

你可以使用命令 `kubectl taint` 给节点增加一个污点。

比如：

```sh
$ kubectl taint nodes node1 key1=value1:NoSchedule
```

给节点 `node1` 增加一个污点，同时它的键名是 key1，键值是 value1，效果是 NoSchedule。这表示**只有拥有和这个污点相匹配的容忍度的 Pod** 才能够被分配到 `node1` 这个节点。

若要移除上述命令所添加的污点，你可以执行：

```sh
$ kubectl taint nodes node1 key1=value1:NoSchedule-
```

**注意：**

- NoSchedule，Pod 不向该节点调度。
- PreferNoSchedule，Pod 尽量不向该节点调度。
- NoExecute，Pod 不向该节点调度，同时将 Pod 从该节点驱逐。

你可以在 Pod 规约中为 Pod 设置容忍度。

下面两个容忍度均与上面例子中使用 `kubectl taint` 命令创建的污点相匹配，因此如果一个 Pod 拥有其中的任何一个容忍度，都能够被调度到 node1 ：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"

---

tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

这里是一个使用了容忍度的 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

`operator` 的默认值是 `Equal`。

一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

- 如果 operator 是 Exists （此时容忍度不能指定 value），或者
- 如果 operator 是 Equal ，则它们的 value 应该相等

你可以给一个节点添加多个污点，也可以给一个 Pod 添加多个容忍度设置。Kubernetes 处理多个污点和容忍度的过程就像一个过滤器：

- 从一个节点的所有污点开始遍历，过滤掉那些 Pod 中存在与之相匹配的容忍度的污点。
- 余下未被过滤的污点的 effect 值决定了 Pod 是否会被分配到该节点。

**注意：**

- 如果未被忽略的污点中存在至少一个 effect 值为 NoSchedule 的污点， 则 Kubernetes 不会将 Pod 调度到该节点。
- 如果未被忽略的污点中不存在 effect 值为 NoSchedule 的污点，但是存在至少一个 effect 值为 PreferNoSchedule 的污点，则 Kubernetes 会**尝试**不将 Pod 调度到该节点。
- 如果未被忽略的污点中存在至少一个 effect 值为 NoExecute 的污点， 则 Kubernetes 不会将 Pod 调度到该节点（如果 Pod 还未在节点上运行）， 或者将 Pod 从该节点驱逐（如果 Pod 已经在节点上运行）。


例如，假设你给一个节点添加了如下污点

```sh
$ kubectl taint nodes node1 key1=value1:NoSchedule
$ kubectl taint nodes node1 key1=value1:NoExecute
$ kubectl taint nodes node1 key2=value2:NoSchedule
```

假定某个 Pod 有两个容忍度：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

在这种情况下，上述 Pod 不会被调度到上述节点，因为其没有容忍度和第三个污点（key2）相匹配。但是如果在给节点添加上述污点之前，该 Pod 已经在上述节点运行，那么它还可以继续运行在该节点上，因为第三个污点是三个污点中唯一不能被这个 Pod 容忍的。

对于 NoExecute 污点：

- 何不能忍受这个污点的 Pod 都会马上被驱逐，任何可以忍受这个污点的 Pod 都不会被驱逐。
- 如果 Pod 存在一个 effect 值为 NoExecute 的容忍度指定了可选属性 tolerationSeconds 的值，则表示在给节点添加了上述污点之后， Pod 还能继续在节点上运行的时间。

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

这表示如果这个 Pod 正在运行，同时一个匹配的污点被添加到其所在的节点，那么 Pod 还将继续在节点上运行 3600 秒，然后被驱逐。 如果在此之前上述污点被删除了，则 Pod 不会被驱逐。

## 使用例子

通过**污点和容忍度**，可以灵活地让 Pod 避开某些节点或者将 Pod 从某些节点驱逐。

下面是几个使用例子：

- 专用节点：如果想将某些节点专门分配给特定的一组用户使用，你可以给这些节点添加一个污点（`kubectl taint nodes <nodename> dedicated=groupName:NoSchedule`），然后给这组用户的 Pod 添加一个相对应的容忍度（通过编写一个自定义的准入控制器，很容易就能做到）。拥有上述容忍度的 Pod 就能够被调度到上述专用节点，同时也能够被调度到集群中的其它节点。 如果你希望这些 Pod 只能被调度到上述专用节点， 那么你还需要给这些专用节点另外添加一个和上述污点类似的 label （例如：dedicated=groupName）， 同时还要在上述准入控制器中给 Pod 增加节点亲和性要求，要求上述 Pod 只能被调度到添加了 dedicated=groupName 标签的节点上。
- 配备了特殊硬件的节点：在部分节点配备了特殊硬件（比如 GPU）的集群中， 我们希望不需要这类硬件的 Pod 不要被调度到这些特殊节点，以便为后继需要这类硬件的 Pod 保留资源。 要达到这个目的，可以先给配备了特殊硬件的节点添加污点 （例如 kubectl taint nodes nodename special=true:NoSchedule 或 kubectl taint nodes nodename special=true:PreferNoSchedule)， 然后给使用了这类特殊硬件的 Pod 添加一个相匹配的容忍度。 和专用节点的例子类似，添加这个容忍度的最简单的方法是使用自定义 准入控制器。 比如，我们推荐使用扩展资源 来表示特殊硬件，给配置了特殊硬件的节点添加污点时包含扩展资源名称， 然后运行一个 ExtendedResourceToleration 准入控制器。此时，因为节点已经被设置污点了，没有对应容忍度的 Pod 不会被调度到这些节点。 但当你创建一个使用了扩展资源的 Pod 时，ExtendedResourceToleration 准入控制器会自动给 Pod 加上正确的容忍度，这样 Pod 就会被自动调度到这些配置了特殊硬件的节点上。 这种方式能够确保配置了特殊硬件的节点专门用于运行需要这些硬件的 Pod， 并且你无需手动给这些 Pod 添加容忍度。
- 基于污点的驱逐: 这是在每个 Pod 中配置的在节点出现问题时的驱逐行为，接下来的章节会描述这个特性。

## 基于污点的驱逐

前文提到过污点的效果值 NoExecute 会影响已经在节点上运行的 Pod，如下

- 如果 Pod 不能忍受这类污点，Pod 会马上被驱逐。
- 如果 Pod 能够忍受这类污点，但是在容忍度定义中没有指定 tolerationSeconds， 则 Pod 还会一直在这个节点上运行。
- 如果 Pod 能够忍受这类污点，而且指定了 tolerationSeconds， 则 Pod 还能在这个节点上继续运行这个指定的时间长度。

Kubernetes 集群自己也会充分利用该机制进行 Pod 驱逐。当 Kubernetes 集群检测到以下某种条件为真时，节点控制器会自动给节点添加一个污点。当前内置的污点包括：

污点 | 作用
-|-
node.kubernetes.io/not-ready | 节点未准备好。这相当于节点状况 Ready 的值为 "False"。
node.kubernetes.io/unreachable | 节点控制器访问不到节点. 这相当于节点状况 Ready 的值为 "Unknown"。
node.kubernetes.io/memory-pressure | 节点存在内存压力。
node.kubernetes.io/disk-pressure | 节点存在磁盘压力。
node.kubernetes.io/pid-pressure | 节点的 PID 压力。
node.kubernetes.io/network-unavailable | 节点网络不可用。
node.kubernetes.io/unschedulable | 节点不可调度。
node.cloudprovider.kubernetes.io/uninitialized | 如果 kubelet 启动时指定了一个“外部”云平台驱动，它将给当前节点添加一个污点将其标志为不可用。在 cloud-controller-manager 的一个控制器初始化这个节点后，kubelet 将删除这个污点。

在节点被驱逐时，节点控制器或者 kubelet 会添加带有 `NoExecute effect` 的相关污点。如果异常状态恢复正常，kubelet 或节点控制器能够移除相关的污点。

**注意：**

- Control Panel 会限制向节点添加新污点的速率。这一速率限制可以管理多个节点同时不可达时（例如出现网络中断的情况），可能触发的驱逐的数量。

你可以为 Pod 设置 tolerationSeconds，以指定当节点失效或者不响应时，Pod 维系与该节点间绑定关系的时长。

比如，你可能希望在出现网络分裂事件时，对于一个与节点本地状态有着深度绑定的应用而言， 仍然停留在当前节点上运行一段较长的时间，以等待网络恢复以避免被驱逐。 你为这种 Pod 所设置的容忍度看起来可能是这样：

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

**注意：**

- Kubernetes 会自动给 Pod 添加针对 `node.kubernetes.io/not-ready` 和 `node.kubernetes.io/unreachable` 的容忍度，且配置 tolerationSeconds=300， 除非用户自身或者某控制器显式设置此容忍度。
- 这些自动添加的容忍度意味着 Pod 可以在检测到对应的问题之一时，在 5 分钟内保持绑定在该节点上。

DaemonSet 中的 Pod 被创建时， 针对以下污点自动添加的 NoExecute 的容忍度将不会指定 tolerationSeconds：

- node.kubernetes.io/unreachable
- node.kubernetes.io/not-ready

这保证了出现上述问题时 DaemonSet 中的 Pod 永远不会被驱逐。

## 基于节点状态添加污点

Control Panel 使用节点控制器自动创建与节点状况对应的、效果为 NoSchedule 的污点（早期的 Kubernetes 是直接以节点状况来判断）。

调度器在进行调度时**检查污点**，而**不是检查节点状况**。这确保节点状况不会直接影响调度。

例如，如果 DiskPressure 节点状况处于活跃状态，则控制平面添加 **node.kubernetes.io/disk-pressure:NoSchedule** 污点，因此并且不会调度新的 Pod 到该受影响的节点。如果 MemoryPressure 节点状况处于活跃状态，则控制平面添加 **node.kubernetes.io/memory-pressure:NoSchedule** 污点。

对于新创建的 Pod，可以通过添加相应的 Pod 容忍度来忽略节点状况。

Control Panel 还在具有除 BestEffort 之外的 QoS 类的 Pod 上添加 `node.kubernetes.io/memory-pressure` 容忍度。

这是因为：

- Kubernetes 将 Guaranteed 或 Burstable QoS 类中的 Pod（甚至没有设置内存请求的 Pod）视为能够应对内存压力
- 而新创建的 BestEffort Pod 不会被调度到受影响的节点上。

DaemonSet 控制器自动为所有守护进程添加如下 NoSchedule 容忍度，以防 DaemonSet 崩溃：

- node.kubernetes.io/memory-pressure
- node.kubernetes.io/disk-pressure
- node.kubernetes.io/pid-pressure (1.14 或更高版本)
- node.kubernetes.io/unschedulable (1.10 或更高版本)
- node.kubernetes.io/network-unavailable (只适合主机网络配置)

添加上述容忍度确保了向后兼容，你也可以选择自由向 DaemonSet 添加容忍度。
