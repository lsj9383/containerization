# 推荐使用的标签

除了 kubectl 和 dashboard 之外，你还可以使用其他工具来可视化和管理 Kubernetes 对象。

一组通用的标签可以让多个工具之间相互操作，用所有工具都能理解的通用方式描述对象。

除了支持工具外，推荐的标签还以一种可以查询的方式描述了应用程序。

元数据围绕`应用（application） `的概念进行组织。Kubernetes 不是平台即服务（PaaS），没有或强制执行正式的应用程序概念。

相反，应用程序是非正式的，并使用元数据进行描述。应用程序包含的定义是松散的。

**注意：**

- 共享标签和注解都使用同一个前缀：`app.kubernetes.io`。
- 没有前缀的标签是用户私有的。
- 共享前缀可以确保共享标签不会干扰用户自定义的标签。

## 标签

为了充分利用这些标签，**应该**在每个资源对象上都使用它们（这里是推荐的）。

键 | 描述 | 示例 | 类型
-|-|-|-
app.kubernetes.io/name | 应用程序的名称 | mysql | 字符串
app.kubernetes.io/instance | 用于唯一确定应用实例的名称 | mysql-abcxzy | 字符串
app.kubernetes.io/version | 应用程序的当前版本（例如语义版本、修订版哈希等） | 5.7.21 | 字符串
app.kubernetes.io/component | 架构中的组件 | database | 字符串
app.kubernetes.io/part-of | 此级别的更高级别应用程序的名称 | wordpress | 字符串
app.kubernetes.io/managed-by | 用于管理应用程序的工具 | helm | 字符串


为说明这些标签的实际使用情况，请看下面的 StatefulSet 对象：

```yaml
# 这是一段节选
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```

## 应用和应用实例

Application 可以在 Kubernetes 集群中安装一次或多次。在某些情况下，可以安装在同一命名空间中。 例如，可以不止一次地为不同的站点安装不同的 WordPress。

应用的名称和实例的名称是分别记录的。例如，WordPress 应用的 app.kubernetes.io/name 为 wordpress，而其实例名称 app.kubernetes.io/instance 为 wordpress-abcxzy。 这使得应用和应用的实例均可被识别，应用的每个实例都必须具有唯一的名称。

## 示例

为了说明使用这些标签的不同方式，以下示例具有不同的复杂性。

### 一个简单的无状态服务

考虑使用 Deployment 和 Service 对象部署的简单无状态服务的情况。以下两个代码段表示如何以最简单的形式使用标签。

下面的 Deployment 用于监督运行应用本身的那些 Pod。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

下面的 Service 用于暴露应用。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

### 带有一个数据库的 Web 应用程序

考虑一个稍微复杂的应用：一个使用 Helm 安装的 Web 应用（WordPress）， 其中使用了数据库（MySQL）。以下代码片段说明用于部署此应用程序的对象的开始。

以下 Deployment 的开头用于 WordPress：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

这个 Service 用于暴露 WordPress：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

MySQL 作为一个 StatefulSet 暴露，包含它和它所属的较大应用程序的元数据：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

Service 用于将 MySQL 作为 WordPress 的一部分暴露：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

使用 MySQL StatefulSet 和 Service，你会注意到有关 MySQL 和 WordPress 的信息，包括更广泛的应用程序。
