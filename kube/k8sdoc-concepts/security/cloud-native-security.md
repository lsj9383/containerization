# 云原生安全概述

定义了一个模型，用于在 Cloud Native 安全性上下文中考虑 Kubernetes 安全性。

**注意：**

- 此容器安全模型只提供建议，而不是经过验证的信息安全策略。

## 云原生安全的 4 个 C

你可以分层去考虑安全性，云原生安全的 4 个 C 分别是：

- 云（Cloud）
- 集群（Cluster）
- 容器（Container）
- 代码（Code）

**注意：**

- 这种分层方法增强了深度防护方法在安全性方面的防御能力，该方法被广泛认为是保护软件系统的最佳实践。

![](assets/4c.png)

云原生安全模型的每一层都建立在下一个最外层之上。

代码层受益于强大的基础（云、集群、容器）安全层，您无法通过解决代码级别的安全性来防止基础层中的安全标准不佳。

## 云

在许多方面，云（或者位于同一位置的服务器，或者是公司数据中心）是 Kubernetes 集群中的[可信计算基础](https://en.wikipedia.org/wiki/Trusted_computing_base)。

如果 Cloud Layer 容易受到攻击（或者被配置成了易受攻击的方式），就不能保证在此基础之上构建的组件是安全的。每个云提供商都会提出安全建议，以在其环境中安全地运行工作负载。

### 云提供商安全性

如果你是在你自己的硬件或者其他不同的云提供商上运行 Kubernetes 集群， 请查阅相关文档来获取最好的安全实践。

下面是一些比较流行的云提供商的安全性文档链接：

IaaS 提供商 | 链接
-|-
Alibaba Cloud | https://www.alibabacloud.com/trust-center
Amazon Web Services | https://aws.amazon.com/security/
Google Cloud Platform | https://cloud.google.com/security/
Huawei Cloud | https://www.huaweicloud.com/securecenter/overallsafety.html
IBM Cloud | https://www.ibm.com/cloud/security
Microsoft Azure | https://docs.microsoft.com/en-us/azure/security/azure-security
Oracle Cloud Infrastructure | https://www.oracle.com/security/
VMWare VSphere | https://www.vmware.com/security/hardening-guides.html

### 基础设施安全

关于在 Kubernetes 集群中保护你的基础设施的建议：

Kubernetes 基础架构关注领域 | 建议
-|-
通过网络访问 API 服务（控制平面） | 所有对 Kubernetes 控制平面的访问不允许在 Internet 上公开，同时应由网络访问控制列表控制，该列表**包含管理集群所需的 IP 地址集**。
通过网络访问 Node（节点） | 节点应配置为仅能从控制平面上通过指定端口来接受（通过网络访问控制列表）连接，以及接受 NodePort 和 LoadBalancer 类型的 Kubernetes 服务连接。如果可能的话，这些节点不应完全暴露在公共互联网上。
Kubernetes 访问云提供商的 API | 每个云提供商都需要向 Kubernetes 控制平面和节点授予不同的权限集。为集群提供云提供商访问权限时，最好遵循对需要管理的资源的最小特权原则。
访问 etcd | 对 etcd（Kubernetes 的数据存储）的访问应仅限于控制平面。根据配置情况，你应该尝试通过 TLS 来使用 etcd。更多信息可以在 etcd 文档中找到。
etcd 加密 | 在所有可能的情况下，最好对所有存储进行静态数据加密，并且由于 etcd 拥有整个集群的状态（包括机密信息），因此其磁盘更应该进行静态数据加密。

## 集群

保护 Kubernetes 有两个方面需要注意：

- 保护可配置的集群组件
- 保护在集群中运行的应用程序

### 集群组件

如果想要保护集群免受意外或恶意的访问，采取良好的信息管理实践，请阅读并遵循有关[保护集群](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/securing-a-cluster/)的建议。

### 你的应用

根据你的应用程序的受攻击面，你可能需要关注安全性的特定面，比如：

- 如果你正在运行中的一个服务（A 服务）在其他资源链中很重要，并且所运行的另一工作负载（服务 B） 容易受到资源枯竭的攻击，则如果你不限制服务 B 的资源的话，损害服务 A 的风险就会很高。
- 
- 下表列出了安全性关注的领域和建议，用以保护 Kubernetes 中运行的工作负载：

工作负载安全性关注领域 | 建议
-|-
RBAC 授权(访问 Kubernetes API) | https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/
认证方式 | https://kubernetes.io/zh-cn/docs/concepts/security/controlling-access/
应用程序 Secret 管理 (并在 etcd 中对其进行静态数据加密) | https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/<br>https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/encrypt-data/
确保 Pod 符合定义的 Pod 安全标准 | https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/#policy-instantiation
服务质量（和集群资源管理） | https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/quality-service-pod/
网络策略 | https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/
Kubernetes Ingress 的 TLS 支持 | https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/#tls

## 容器

容器安全性不在本指南的探讨范围内。下面是一些探索此主题的建议和连接：

容器关注领域 | 建议
-|-
容器漏洞扫描和操作系统依赖安全性 | 作为镜像构建的一部分，你应该扫描你的容器里的已知漏洞。
镜像签名和执行 | 对容器镜像进行签名，以维护对容器内容的信任。
禁止特权用户 | 构建容器时，请查阅文档以了解如何在具有最低操作系统特权级别的容器内部创建用户，以实现容器的目标。
使用带有较强隔离能力的容器运行时 | 选择提供较强隔离能力的容器运行时类。

## 代码

应用程序代码是你最能够控制的主要攻击面之一，虽然保护应用程序代码不在 Kubernetes 安全主题范围内，但以下是保护应用程序代码的建议：

代码关注领域 | 建议
-|-
仅通过 TLS 访问 | 如果你的代码需要通过 TCP 通信，请提前与客户端执行 TLS 握手。除少数情况外，请加密传输中的所有内容。更进一步，加密服务之间的网络流量是一个好主意。这可以通过被称为双向 TLS 或 mTLS 的过程来完成，该过程对两个证书持有服务之间的通信执行双向验证。
限制通信端口范围 | 此建议可能有点不言自明，但是在任何可能的情况下，你都只应公开服务上对于通信或度量收集绝对必要的端口。
第三方依赖性安全 | 最好定期扫描应用程序的第三方库以了解已知的安全漏洞。每种编程语言都有一个自动执行此检查的工具。
静态代码分析 | 大多数语言都提供给了一种方法，来分析代码段中是否存在潜在的不安全的编码实践。只要有可能，你都应该使用自动工具执行检查，该工具可以扫描代码库以查找常见的安全错误，一些工具可以在以下连接中找到： https://owasp.org/www-community/Source_Code_Analysis_Tools
动态探测攻击 | 你可以对服务运行一些自动化工具，来尝试一些众所周知的服务攻击。这些攻击包括 SQL 注入、CSRF 和 XSS。OWASP Zed Attack 代理工具是最受欢迎的动态分析工具之一。
