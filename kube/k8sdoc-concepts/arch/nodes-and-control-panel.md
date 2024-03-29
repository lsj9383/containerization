# 节点与控制面之间的通信

本文列举 API 服务器（即控制面所用的节点）和 Kubernetes 集群之间的通信路径。

目的是为了让用户能够自定义他们的安装，以实现对网络配置的加固，使得集群能够在不可信的网络上（或者在一个云服务商完全公开的 IP 上）运行。

## 节点到控制面

Kubernetes 采用的是中心辐射型（Hub-and-Spoke）API 模式：

- 来自节点（或运行于其上的 Pod）发出的 API 调用都终止于 API 服务器。
- 其它控制面组件都没有被设计为可暴露远程服务（例如控制器、调度器等都不会对外暴露端口）。

对于节点到控制面的安全方面：

- API 服务器配置为在启用了一种或多种形式的客户端身份验证的情况下侦听安全 HTTPS 端口（通常为 443）上的远程连接。
- 应启用一种或多种形式的授权，尤其是在允许匿名请求或服务帐户令牌的情况下。
- 应为节点提供集群的公共根证书，以便它们可以通过有效的客户端凭据，安全地连接到 API 服务器。
    - 一个好的方法是提供给 kubelet 的客户端凭据采用**客户端证书**的形式。有关 kubelet 客户端证书的自动配置，请参阅 [kubelet TLS 引导](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)。

对于 Pod 连接到 API 服务器的安全方面：

- 连接到 API 服务器的 Pod 可以通过利用**服务帐户**安全地连接，这样 Kubernetes 会在实例化时自动将公共根证书和有效的不记名令牌注入到 Pod 中。
- kubernetes 服务（在默认命名空间中）配置了一个虚拟 IP 地址，该地址被重定向（通过 kube-proxy）到 API 服务器上的 HTTPS 端点。

控制平面组件还通过安全端口与 API 服务器通信。

因此，从节点和 pod 到控制平面的连接的默认操作模式默认是安全的，并且可以在不受信任的和/或公共网络上运行。

## 控制面到节点

从控制面（API 服务器）到节点有两种主要的通信路径：

- 第一种是从 API 服务器到集群中每个节点上运行的 kubelet 进程。
- 第二种是通过 API 服务器的代理功能从 API 服务器到任何节点、pod 或服务。

### API 服务器到 kubelet

从 API 服务器到 kubelet 的连接用于：

- 获取 Pod 日志
- 附加（通常通过 kubectl）到正在运行的 pod。
- 提供 kubelet 的端口转发功能。

这些连接基于 kubelet 的 HTTPS 端点。默认情况下，API 服务器不验证 kubelet 的服务证书，这使得连接容易受到中间人攻击，并且在不受信任和/或公共网络上运行是不安全的。

为了对这个连接进行认证，使用 **--kubelet-certificate-authority** 标志给 API 服务器提供一个根证书包，用于 kubelet 的服务证书。

如果无法实现这点，又要求避免在非受信网络或公共网络上进行连接，可在 API 服务器和 kubelet 之间使用 SSH 隧道。

最后，应该启用 Kubelet 认证/鉴权 来保护 kubelet API。

### API 服务器到节点、Pod 和服务

从 API 服务器到节点、Pod 或服务的连接默认为**纯 HTTP 方式**，因此既没有认证，也没有加密。

这些连接可通过给 API URL 中的节点、Pod 或服务名称添加前缀 https: 来运行在安全的 HTTPS 连接上。不过这些连接既不会验证 HTTPS 末端提供的证书，也不会提供客户端证书。因此，虽然连接是加密的，仍无法提供任何完整性保证。这些连接 目前还不能安全地 在非受信网络或公共网络上运行。

### SSH 隧道

Kubernetes 支持使用 SSH 隧道来保护从控制面到节点的通信路径。

在这种配置下， API 服务器建立一个到集群中各节点的 SSH 隧道（连接到在 22 端口监听的 SSH 服务器） 并通过这个隧道传输所有到 kubelet、节点、Pod 或服务的请求。这一隧道保证通信不会被暴露到集群节点所运行的网络之外。

**注意：**

- SSH 隧道目前已被废弃。除非你了解个中细节，否则不应使用。 Konnectivity 服务是 SSH 隧道的替代方案。

### Konnectivity 服务

特性状态： Kubernetes v1.18 [beta]

作为 SSH 隧道的替代方案，Konnectivity 服务提供 TCP 层的代理，以便支持从控制面到集群的通信。

Konnectivity 服务包含两个部分：

- Konnectivity 服务器
- Konnectivity 代理

分别运行在控制面网络和节点网络中。

Konnectivity 代理建立并维持到 Konnectivity 服务器的网络连接。启用 Konnectivity 服务之后，所有控制面到节点的通信都通过这些连接传输。
