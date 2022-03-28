# DestinationRule

`DestinationRule` 定义了在路由发生后适用于服务流量的策略。这些规则指定了负载均衡的配置、来自 sidecar 的连接池大小，以及用于检测和驱逐负载均衡池中不健康主机的异常检测设置。

## 示例

例如，ratings 服务的一个简单的负载均衡策略看起来如下。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

可以通过定义一个 `subset` 并覆盖在服务级来配置特定版本的策略。下面的规则对前往由带有标签（`version:v3`）的端点（如 pod）组成的名为 `testversion` 的子集的所有流量使用轮询负载均衡策略。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
  subsets:
  - name: testversion
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

**注意**：只有当路由规则明确地将流量发送到这个子集，为子集指定的策略才会生效。

流量策略也可以针对特定的端口进行定制。下面的规则对所有到 80 号端口的流量使用最少连接的负载均衡策略，而对 9080 号端口的流量使用轮流负载均衡设置。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: bookinfo-ratings-port
spec:
  host: ratings.prod.svc.cluster.local
  trafficPolicy: # 应用到所有端口
    portLevelSettings:
    - port:
        number: 80
      loadBalancer:
        simple: LEAST_CONN
    - port:
        number: 9080
      loadBalancer:
        simple: ROUND_ROBIN
```

## 配置项

下图是 DestinationRule 资源的配置拓扑图。

![DestinationRule 配置项拓扑图](../../images/destinationrule.png)

DestinationRule 资源的顶级配置项如下：

- `host`：字符串类型。来自服务注册表的服务名称。服务名称从平台的服务注册表（例如，Kubernetes 服务、Consul 服务等）和 ServiceEntry 声明的主机中查找。为服务注册表中不存在的服务定义的规则将被忽略。

  Kubernetes 用户的注意事项。当使用短名称时（例如 `reviews` 而不是 `reviews.default.svc.cluster.local`），Istio 将根据规则的命名空间而不是服务来解释短名称。在 `default` 命名空间中包含主机 `reviews` 的规则将被解释为 `reviews.default.svc.cluster.local`，而不考虑与 reviews 服务相关的实际命名空间。为了避免潜在的错误配置，建议总是使用完全限定名而不是短名称。

  注意，主机字段适用于 HTTP 和 TCP 服务。

- `trafficPolicy`：要应用的流量策略（负载均衡策略、连接池大小、异常值检测）。

- `subsets`：一个或多个命名的集合，代表一个服务的单独版本。流量策略可以在子集级别被覆盖。

- `exportTo`：`DestinationRule` 被导出的命名空间的列表。应用于服务的 DestinationRule 的解析是在命名空间的层次结构中进行的。导出 DestinationRule 允许它被包含在其他命名空间的服务的解析层次中。该功能为服务所有者和网格管理员提供了一种机制，以控制 VirtualService 在命名空间边界的可见性。

  如果没有指定命名空间，那么默认情况下，VirtualService 会被输出到所有命名空间。

  值 `.` 是保留的，它定义了导出到 DestinationRule 声明的同一命名空间。同样，值 `*` 也是保留的，它定义了导出到所有命名空间。

关于 DestinationRule 配置的详细用法请参考 [Istio 官方文档](https://istio.io/latest/docs/reference/config/networking/destination-rule/)。

## 参考

- [Destination Rule - istio.io](https://istio.io/latest/docs/reference/config/networking/destination-rule/)