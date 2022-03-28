# Sidecar

`Sidecar` 描述了 sidecar 代理的配置，该代理负责协调与它所连接的工作负载实例的 inbound 和 outbound 通信。默认情况下，Istio 会对网格中的所有 sidecar 代理进行必要的配置，以达到网格中的每个工作负载实例，并接受与工作负载相关的所有端口的流量。`Sidecar` 配置提供了一种方法来微调代理在转发工作负载的流量时将接受的一组端口和协议。此外，还可以限制代理在转发工作负载实例的 outbound 流量时可以到达的服务集。

网格中的服务和配置被组织到一个或多个命名空间（例如，Kubernetes 命名空间）。命名空间中的 Sidecar 配置将适用于同一命名空间中的一个或多个工作负载实例，并通过 `workloadSelector` 字段进行选择。在没有 `workloadSelector` 的情况下，它将适用于同一命名空间的所有工作负载实例。在确定应用于工作负载实例的 `Sidecar` 配置时，将优先考虑具有选择该工作负载实例的 `workloadSelector` 的资源，而不是没有任何 `workloadSelector` 的 Sidecar 配置。

## 注意事项

在配置 Sidecar 时需要注意以下事项。

### 注意一

每个命名空间只能有一个没有任何 `workloadSelector` 的 Sidecar 配置，该配置为该命名空间的所有 pod 指定了默认配置。建议对整个命名空间的 sidecar 命名为 `defatult`。如果在一个给定的命名空间中存在多个无选择器的 Sidecar 配置，则系统的行为未定义。如果两个或更多带有 `workloadSelector` 的 `Sidecar` 配置选择了同一个工作负载实例，那么系统的行为将无法定义。

### 注意二

`MeshConfig` 根命名空间中的 `Sidecar` 配置将被默认应用于所有没有 Sidecar 配置的命名空间。这个全局默认 Sidecar 配置不应该有任何 workloadSelector。

## 示例

下面的例子在根命名空间 `istio-config` 中声明了一个全局默认 `Sidecar` 配置，该配置将所有命名空间中的 sidecar 配置为只允许向同一命名空间中的其他工作负载以及 `istio-system` 命名空间中的服务输出流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: istio-config
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

下面的例子在 `prod-us1` 命名空间中声明了一个 `Sidecar` 配置，它覆盖了上面定义的全局默认值，并配置了命名空间中的 sidecar，以允许向 `prod-us1`、`prod-apis` 和 `istio-system` 命名空间的公开服务输出流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  egress:
  - hosts:
    - "prod-us1/*"
    - "prod-apis/*"
    - "istio-system/*"
```

下面的例子在 `prod-us1` 命名空间为所有标签为 `app: ratings` 的 pod 声明了一个 `Sidecar` 配置，这些 pod 属于 `rating.prod-us1` 服务。该工作负载接受 9080 端口的入站 HTTP 流量。然后，该流量被转发到在 Unix 域套接字上监听的附加工作负载实例。在出口方向，除了 `istio-system` 命名空间外，sidecar 只为 `prod-us1` 命名空间的服务代理绑定在 9080 端口的 HTTP 流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: ratings
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: ratings
  ingress:
  - port:
      number: 9080
      protocol: HTTP
      name: somename
    defaultEndpoint: unix:///var/run/someuds.sock
  egress:
  - port:
      number: 9080
      protocol: HTTP
      name: egresshttp
    hosts:
    - "prod-us1/*"
  - hosts:
    - "istio-system/*"
```

如果工作负载的部署没有基于 IPTables 的流量捕获，`Sidecar` 配置是配置连接到工作负载实例的代理上的端口的唯一方法。下面的例子在 `prod-us1` 命名空间中为属于 `productpage.prod-us1` 服务的标签为 `app: productpage` 的所有 pod 声明了一个 `Sidecar` 配置。假设这些 pod 的部署没有 IPtable 规则（即 `istio-init` 容器），并且代理元数据 `ISTIO_META_INTERCEPTION_MODE` 被设置为 `NONE`，下面的规范允许这样的 pod 在 9080 端口接收 HTTP 流量（在 Istio mutual TLS 内包装），并将其转发给在 `127.0.0.1:8080` 监听的应用程序。它还允许应用程序与 `127.0.0.1:3306` 上的 MySQL 数据库进行通信，然后被代理到 `mysql.foo.com:3306` 上的外部托管 MySQL 服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: no-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - port:
      number: 9080 #绑定到 proxy_instance_ip:9080（0.0.0.0:9080，如果实例没有可用的单播 IP）。
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    captureMode: NONE #如果为整个代理设置了元数据，则不需要。
  egress:
  - port:
      number: 3306
      protocol: MYSQL
      name: egressmysql
    captureMode: NONE #如果为整个代理设置了元数据，则不需要。
    bind: 127.0.0.1
    hosts:
    - "*/mysql.foo.com"
```

以及路由到 `mysql.foo.com:3306` 的相关 ServiceEntry。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mysql
  namespace: ns1
spec:
  hosts:
  - mysql.foo.com
  ports:
  - number: 3306
    name: mysql
    protocol: MYSQL
  location: MESH_EXTERNAL
  resolution: DNS
```

也可以在一个代理中混合和匹配流量捕获模式。例如有一个设置，内部服务在 `192.168.0.0/16` 子网。因此，在虚拟机上设置 IPTables 以捕获 `192.168.0.0/16` 子网的所有出站流量。假设虚拟机在 `172.16.0.0/16` 子网有一个额外的网络接口，用于入站流量。下面的 `Sidecar` 配置允许虚拟机在 `172.16.1.32:80`（虚拟机的 IP）上为从 `172.16.0.0/16` 子网到达的流量暴露一个监听器。

注意：虚拟机中代理上的 `ISTIO_META_INTERCEPTION_MODE` 元数据应包含 `REDIRECT` 或 `TPROXY` 作为其值，这意味着基于 IPTables 的流量捕获是激活的。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: partial-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - bind: 172.16.1.32
    port:
      number: 80 #绑定到 172.16.1.32:80
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    captureMode: NONE
  egress:
  #使用系统检测到的默认值设置配置，以处理到 192.168.0.0/16 子网的服务的出站流量，基于服务注册表提供的信息
  - captureMode: IPTABLES
    hosts:
    - "*/*"
```

## 配置项

下图是 Sidecar 资源的配置拓扑图。

![Sidecar 资源配置拓扑图](../../images/sidecar.png)

Sidecar 资源的顶级配置项如下：

- `workloadSelector`：用于选择应用该 Sidecar 配置的特定 pod/VM 集合的标准。如果省略，Sidecar 配置将被应用于同一命名空间的所有工作负载实例。
- `ingress`：Ingress 指明用于处理连接工作负载实例的入站流量的 sidecar 配置。如果省略，Istio 将根据从编排平台获得的工作负载信息（例如，暴露的端口、服务等）自动配置 sidecar。如果指定，当且仅当工作负载实例与服务相关联时，将配置入站端口。
- `egress`：Egress 指定 sidecar 的配置，用于处理从附加工作负载实例到网格中其他服务的出站流量。如果不指定，则继承系统检测到的全命名空间或全局默认 Sidecar 的默认值。
- `outboundTrafficPolicy`：出站流量策略的配置。如果你的应用程序使用一个或多个事先不知道的外部服务，将策略设置为`ALLOW_ANY`，将导致 sidecar 将任何来自应用程序的未知流量路由到其请求的目的地。如果没有指定，则继承系统检测到的来自全命名空间或全局默认 Sidecar 的默认值。

关于 Sidecar 配置的详细用法请参考 [Istio 官方文档](https://istio.io/latest/docs/reference/config/networking/sidecar/)。

## 参考

- [Sidecar - Istio 官网文档](https://istio.io/latest/docs/reference/config/networking/sidecar/)