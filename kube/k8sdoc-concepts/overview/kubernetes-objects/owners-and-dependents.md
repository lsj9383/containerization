# 属主与附属

在 Kubernetes 中，一些对象是其他对象的 “属主（Owner）”，与属主 Owner 相对的那批对象被称为 “附属（Dependent）”。

例如：ReplicaSet 是一组 Pod 的属主，这些 Pod 是附属。

## 对象规约中的属主引用

附属对象有一个 **metadata.ownerReferences** 字段，用于引用其属主对象。

一个有效的属主引用，包含与附属对象同在一个命名空间下的`1) 对象名称`和`2) UID`。

Kubernetes 自动为一些对象的附属资源设置属主引用的值，这些对象包含: ReplicaSet、DaemonSet、Deployment、Job、CronJob、ReplicationController 等。

你也可以通过改变这个字段的值，来手动配置这些关系。 然而，通常不需要这么做，你可以让 Kubernetes 自动管理附属关系。

附属对象还有一个 **ownerReferences.blockOwnerDeletion** 字段，该字段使用布尔值， 用于控制特定的附属对象是否可以阻止垃圾收集删除其属主对象。

如果控制器（例如 Deployment 控制器） 设置了 **metadata.ownerReferences** 字段的值，Kubernetes 会自动设置 **blockOwnerDeletion** 的值为 true。 你也可以手动设置 blockOwnerDeletion 字段的值，以控制哪些附属对象会阻止垃圾收集。

Kubernetes 准入控制器根据属主的删除权限控制用户访问，以便为附属资源更改此字段。 这种控制机制可防止未经授权的用户延迟属主对象的删除。

**注意：**

- 根据设计，kubernetes 不允许跨 namespace 指定属主。
- 名字空间范围的附属可以指定集群范围的或者名字空间范围的属主。
- 名字空间范围的属主必须和该附属处于相同的名字空间。
- 如果名字空间范围的属主和附属不在相同的名字空间，那么该属主引用就会被认为是缺失的，并且当附属的所有属主引用都被确认不再存在之后，该附属就会被删除。

## 属主关系与 Finalizer

当你告诉 Kubernetes 删除一个资源，API 服务器允许管理控制器处理该资源的任何 Finalizer 规则。 Finalizer 防止意外删除你的集群所依赖的、用于正常运作的资源。 

例如，如果你试图删除一个仍被 Pod 使用的 PersistentVolume，该资源不会被立即删除， 因为 PersistentVolume 有 kubernetes.io/pv-protection Finalizer。 相反，它将进入 Terminating 状态，直到 Kubernetes 清除这个 Finalizer， 而这种情况只会发生在 PersistentVolume 不再被挂载到 Pod 上时。
