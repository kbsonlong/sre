# 通过Gateway API不断演变的Kubernetes网络


Ingress 资源是 Kubernetes 众多成功故事之一。它创建了一个不同的 Ingress 控制器生态系统，这些控制器以标准化和一致的方式在成千上万的集群中使用。这种标准化帮助用户采用 Kubernetes。然而，在 Ingress 创建 5 年后，有迹象表明，分裂为不同但惊人相似的 CRD 和超载的注释。使 Ingress 普及的可移植性同样也限制了它的未来。
<!--more-->
在 2019 年圣地亚哥 Kubecon 大会上，一群热情的贡献者聚集在一起讨论 Ingress 的演变。讨论蔓延到了街对面的酒店大厅，结果就是后来被称为 Gateway API 的东西。这一讨论是基于以下几个关键假设：

1.  作为路由匹配、流量管理和服务暴露基础的 API 标准已经商品化，作为自定义 API 对其实现者和用户几乎没有提供什么价值
    
2.  可以通过共同的核心 API 资源来表示 L4/L7 路由和流量管理
    
3.  以一种不牺牲核心 API 的用户体验的方式，为更复杂的功能提供可扩展性是可能的
    

## 引入 Gateway API

这就引出了允许 Gateway API 在 Ingress 基础上改进的设计原则：

-   表达能力——除了 HTTP 主机/路径匹配和 TLS 之外，Gateway API 还可以表达 HTTP 头操作、流量加权和镜像、TCP/UDP 路由以及其他只能在 Ingress 中通过自定义注释才能实现的功能。
    
-   面向角色的设计——API 资源模型反映了在路由和 Kubernetes 服务网络中常见的职责分离。
    
-   可扩展性——资源允许在 API 的不同层上附加任意的配置。这使得在最合适的地方可以进行细粒度定制。
    
-   灵活的一致性——Gateway API 定义了不同的一致性级别——核心（强制支持）、扩展（如果支持则可移植）和自定义（没有可移植性保证），统称为灵活的一致性\[1\]。这促进了一个高度可移植的核心 API（如 Ingress），它仍然为网关控制器实现者提供灵活性。
    

### Gateway API 是什么样子的？

Gateway API 引入了一些新的资源类型：

-   GatewayClasses 是集群范围的资源，作为模板来显式定义从它们派生的 Gateways 的行为。这在概念上类似于 StorageClasses，但用于联网数据平面。
    
-   Gateways 是 GatewayClasses 的部署实例。它们是执行路由的数据平面的逻辑表示，可以是集群内代理、硬件 LB 或云 LB。
    
-   路由不是单一的资源，而是代表许多不同协议特定的 Route 资源。HTTPRoute 具有匹配、过滤和路由规则，这些规则应用于能够处理 HTTP 和 HTTPS 流量的 Gateways。类似地，TCPRoutes、UDPRoutes 和 TLSRoutes 也具有特定于协议的语义。此模型还允许 Gateway API 将来增量地扩展其协议支持。
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/GpkQxibjhkJwBvJMF3973oiawUVU1rBTicty0ibCD5Ieou3L0W77KRPicgZBicYBjiav6EAibGPRKHky4za62LuicmALfnA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 网关控制器实现

好消息是，尽管 Gateway 是在Alpha\[2\]阶段，但已经有几个你可以运行的Gateway 控制器实现\[3\]。由于它是一个标准化的规范，下面的示例可以在它们中的任何一个上运行，并且应该以完全相同的方式工作。查看入门手册\[4\]，了解如何安装和使用这些网关控制器之一。

## Gateway API 例子

在下面的例子中，我们将演示不同 API 资源之间的关系，并带你浏览一个常见的用例：

-   团队 foo 将他们的应用部署在 foo 命名空间中。他们需要控制应用程序不同页面的路由逻辑。
    
-   团队 bar 运行在 bar 命名空间中。他们希望能够对他们的应用进行蓝绿发布以降低风险。
    
-   平台团队负责管理 Kubernetes 集群中所有应用的负载均衡器和网络安全。
    

下面的 foo-route 对 foo 命名空间中的各种服务进行路径匹配，并且还有一个到 404 服务器的默认路由。这将分别通过 foo.example.com/login 和 foo.example.com/home 暴露 foo-auth 和 foo-home 服务：

```auto
kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: foo-route
  namespace: foo
  labels:
    gateway: external-https-prod
spec:
  hostnames:
  - "foo.example.com"
  rules:
  - matches:
    - path:
        type: Prefix
        value: /login
    forwardTo:
    - serviceName: foo-auth
      port: 8080
  - matches:
    - path:
        type: Prefix
        value: /home
    forwardTo:
    - serviceName: foo-home
      port: 8080
  - matches:
    - path:
        type: Prefix
        value: /
    forwardTo:
    - serviceName: foo-404
      port: 8080
```

bar 团队在同一个 Kubernetes 集群的 bar 命名空间中操作，也希望将他们的应用程序暴露给互联网，但他们也希望控制自己的灰度和蓝绿发布。下面的 HTTPRoute 被配置为以下行为：

-   bar.example.com 的流量：
    

-   将 90%的流量发送到 bar-v1
    
-   将 10%的流量发送给 bar-v2
    

-   对于使用 HTTP 头 env:canary 到 bar.example.com 的流量：
    

-   将所有流量发送到 bar-v2
    

![图片](https://mmbiz.qpic.cn/mmbiz_png/GpkQxibjhkJwBvJMF3973oiawUVU1rBTictgeVnETPtq36HWeH3Hf2HUfeaMCLZCOTYicVZTBGoCmQEfvHdvZmictiaw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

```auto
kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: bar-route
  namespace: bar
  labels:
    gateway: external-https-prod
spec:
  hostnames:
  - "bar.example.com"
  rules:
  - forwardTo:
    - serviceName: bar-v1
      port: 8080
      weight: 90
    - serviceName: bar-v2
      port: 8080
      weight: 10
  - matches:
    - headers:
        values:
          env: canary
    forwardTo:
    - serviceName: bar-v2
      port: 8080
```

### 路由和网关绑定

因此，我们有两个匹配的 HTTPRoutes，并将流量路由到不同的服务。你可能想知道，在哪里可以访问这些服务？它们通过哪些网络或 IP 暴露？

路由如何向客户端暴露由路由绑定\[5\]来管理，该绑定描述了路由和网关之间如何创建双向关系。当 Routes 被绑定到一个 Gateway 时，这意味着它们的集合路由规则被配置在底层的负载均衡器或代理上，并且路由可以通过网关访问。因此，网关是可以通过路由配置的网络数据平面的逻辑表示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/GpkQxibjhkJwBvJMF3973oiawUVU1rBTict5IN33XVPiaQDkMibzNkGDmjibUSkya44P5ZB2gsXTPjsm9MJFUaJWxtyw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 行政委托

Gateway 和 Route 资源之间的分离允许集群管理员将一些路由配置委派给各个团队，同时仍然保持集中控制。以下网关资源在端口 443 上暴露 HTTPS，并使用由集群管理员控制的证书终止端口上的所有通信流。

```auto
kind: Gateway
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: prod-web
spec:
  gatewayClassName: acme-lb
  listeners:
  - protocol: HTTPS
    port: 443
    routes:
      kind: HTTPRoute
      selector:
        matchLabels:
          gateway: external-https-prod
      namespaces:
        from: All
    tls:
      certificateRef:
        name: admin-controlled-cert
```

下面的 HTTPRoute 展示了 Route 如何通过它的 kind（HTTPRoute）和资源标签（gateway=external-https-prod）来确保它匹配网关的选择器。

```auto
# Matches the required kind selector on the Gateway
kind: HTTPRoute
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: foo-route
  namespace: foo-ns
  labels:

    # Matches the required label selector on the Gateway
    gateway: external-https-prod
...
```

### 面向角色的设计

当你将它们放在一起时，你就拥有了一个可以被多个团队安全地共享的负载平衡基础设施。Gateway API 不仅是用于高级路由的更具表现力的 API，而且是面向角色的 API，专为多租户基础设施设计。它的可扩展性确保了它将在保持可移植性的同时为未来的用例发展。最终，这些特性将允许 Gateway API 适应不同的组织模型和实现，直到未来。

### 尝试一下，并参与其中

有许多资源可以查看以了解更多。

-   查看入门手册，看看可以解决哪些用例。
    
-   尝试使用现有的网关控制器之一
    
-   或者参与\[6\]并帮助设计和影响 Kubernetes 服务网络的未来！
    

### 参考资料

\[1\]

灵活的一致性: _https://gateway-api.sigs.k8s.io/concepts/guidelines/#conformance_

\[2\]

Alpha: _https://github.com/kubernetes-sigs/gateway-api/releases_

\[3\]

Gateway 控制器实现: _https://gateway-api.sigs.k8s.io/references/implementations/_

\[4\]

入门手册: _https://gateway-api.sigs.k8s.io/guides/getting-started/_

\[5\]

路由绑定: _https://gateway-api.sigs.k8s.io/concepts/api-overview/#route-binding_

\[6\]

参与: _https://gateway-api.sigs.k8s.io/contributing/community/_

