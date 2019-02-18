# 流量管理

本文主要讲述 Istio 中流量管理的工作原理，包括流量管理原则的有点。本文假设你已经阅读了 Istio 是什么？并熟悉 Istio 的高级架构。

## Pilot 和 Envoy

Istio 流量管理的核心组件是 Pilot，它管理和部署在特定 istio 服务网格中的所以 Envoy 代理实例。它允许自己制定在 Envoy 之间使用什么样的
路由规则，并配合故障恢复功能，如超时、重试和熔断器。它还维护了网格中所有的规范模型，并使用这个模型通过发现服务让 Envoy 了解网格中的
其他实例。

每个 Envoy 实例都会维护负载均衡信息，负载均衡信息是基于从 Pilot 或者的信息，以及其他负载均衡池中的其他实例的定期健康检查。从而允许其在目标实例之间职能分配流量，同事遵循其指定的路由规则。

![Pilot 架构](https://istio.io/docs/concepts/traffic-management/PilotAdapters.svg)

Pilot 公开了用于服务发现、负载均衡池和路由表的动态更新的 API。

运维人员可以通过 Pilot 的 Rules API 指定高级流量管理规则。这些规则被翻译成低级配置，并通过 discovery API 分发到 Envoy 实例。

---

### 路由请求

### 服务之间的通讯

### Ingress 和 Egress

### 服务发现和负载均衡

服务的所有的 HTTP 流量都会通过 Envoy 自动重新路由。Envoy 在负载均衡池中的时候之间分发流量。虽然 Envoy 支持多种复杂的负载负载算法（Round robin，Ring hash，Weighted least request，Random，Original destination），但 istio 目前仅允许三种负载均衡模式：轮询、随机和带权重的最少请求。

除了负载均衡外，Envoy 还会定期检查池中每个实例的运行状况。Envoy 遵循熔断器风格模式，根据健康检查 API 调用的失败率将实例分类为不健康或健康。换句话说，当给定实例的健康检查失败次数超过预定阈值时，它将从负载均衡池中弹出。类似地，当通过的健康检查数超过预定阈值时，该实例将被添加回负载均衡池。您可以在处理故障中了解更多有关 Envoy 的故障处理功能。

### 故障处理

Envoy 提供了一套开箱即用，可选的的故障恢复功能，对应用中的服务大有裨益。这些功能包括：

1. 超时
2. 具备超时预算，并能够在重试之间进行可变抖动（间隔 ）的有限重试功能
3. 并发连接数个上游服务请求数限制
4. 对负载均衡池中的每个成员进行（定期）运行健康检查
5. 细粒度熔断器（被动健康检查）- 适用于负载均衡池中的每个实例

### 微调

Istio 的流量管理规则允许运维人员为每个服务/版本设置故障恢复的全局默认值。然而，服务的消费者也可以通过特殊的 HTTP 头提供的请求级别值覆盖超时和重试的默认值。在 Envoy 代理的实现中，对应的 Header 分别是 x-envoy-upstream-rq-timeout-ms 和 x-envoy-max-retries。

### 故障注入

istio 能在不杀死 Pod 的情况下，将协议特定的故障注入到网络中，在 TCP 层制造数据包的延迟或者损坏。

---

### 规则配置

istio 提供了一个简单的配置模型，用来控制 API 调用以及应用部署多个服务之间的四层通信。istio 中包含有四种流量管理配置资源，分别是 VirtualService、DestinationRule、ServiceEntry 以及 Gateway。

- VirtualService 在 istio 服务网格中定义路由规则，控制路由如何路由到服务上。
- DestinationRule 是 VirtualService 路由生效后，配置应用  与请求成都策略集。
- ServiceEntry 是通常用于在 istio 服务网格之外启用对服务的请求。
- Gateway 为 HTTP/TCP 流量配置负载均衡器，最常见的是在网格的边缘的操作，以启动应用程序的入口流量。

#### Virtual Service

VirtualService 定义了控制在 istio 服务网格中如何路由服务请求的规则。

- 规则的目标描述
- 在服务之间拆分流量
- 超时和重试
- 错误注入 在根据路由规则向选中  目标转发 http 请求的时候，可以向其中注入一个或多个错误。错误可以是延迟，也可以是退出。
- 条件规则
  - 使用工作负载 label 限制特定客户端工作负载。
  - 根据 HTTP Header 选择规则。
  - 根据请求 URI 选择规则。
- 多重匹配条件：可以同事设置多个条件。在这种情况下，根据嵌套，应用 AND 或 OR 语义。如果多个小件嵌套在单个匹配  子句中，则条件为 AND。

AND 条件

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
      # 与 OR 不同
      headers:
        end-user:
          exact: jason
    ...
```

OR 条件

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - match:
    - sourceLabels:
        app: reviews
        version: v2
    # 与 AND 不同
    - headers:
        end-user:
          exact: jason
    ...
```

- 优先级：当对一个目标多个规则时，会按照在 virtualService 中的顺序进行应用，换句话说，列表中的第一条规则具体最高优先级。
  - 为什么优先级重要：当对某个服务的路由是完全基于权重的时候，就可以在单一规则中完成。另一方面，如果有多重条件用来进行路由，就会需要不知一条规则。这样就会出现优先级问题，需要通过优先级来保证根据正确的顺序来执行。

#### 目标规则

在请求被 VirtualService 路由之后，DestinationRule 配置的一系列策略就生效了。这些策略路由服务属主编写， 包含断路器、负载均衡以及 TLS 等的配置内容。

DestinationRule 还定义了对应目标主机的可路由 subset。VirtualService 在向特定服务版本发送请求时会用到这些子集。

- 断路器：可以用一系列的标准，例如连接数和请求数限制来定义简单的断路器。
- 规则评估

#### Service Entry

istio 内部会维护一个服务注册表，可以用 ServiceEntry 向其中加入额外的条目。通常这个对象用来启用对 ISTIO 服务网格之外的服务发出请求。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: foo-ext-svc
spec:
  hosts:
  - *.foo.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
```

ServiceEntry 中使用 hosts 字段来指定目标，字段值可以是一个完全限定名，也可以是通配符域名。其中包含的白名单，包含一个或多个允许网格中服务访问的服务。

ServiceEntry 的配合不仅限于外部服务，它有两种类型：网格内部和网格外部。网格内的条目和其他的内部服务类似，用于显式的将服务加入网格。可以用来把服务作为服务网格扩展的一部分加入不受管理的基础设置。网格外的条目用于表达网格外的服务。对这种条目来说，双向 TLS 认证被禁用，策略实现需要在客户端执行，而不像内部服务请求那样在服务端执行。

#### Gateway

Gateway 为 HTTP/TCP 流量配置了一个负载均衡，多数情况下在网格边缘进行操作，用于启用一个服务的入口（ingress）流量。

和 kubernetes ingress 不同，istio Gateway 只配置四层到六层的功能（例如开放端口或者 TLS 配置）。绑定一个 VirtualService 到 Gateway 上，用户就可以使用标准的 istio 规则来控制进入 HTTP 和 TCP 流量。

#### EnvoyFilter

EnvoyFilter 描述了  针对代理服务的过滤器，用来定制有 istio Pilot 生成的代理配置。一定要谨慎使用此功能。错误的配置内容一旦完成传播，可能会令这个服务网格陷入瘫痪。这配置是用于对 istio 网格系统内部  实现进行变更的，属于高级配置，用于扩展 Envoy 中的过滤器的。

## Istio 中的 Sidecar 注入与流量劫持详解

讲解之前先了解几个概念：

- Sidecar 模式：容器应用模式之一，Service Mesh 架构的一种实现方式。
- init 容器：Pod 中的一种专门的容器，在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。
- IPtable：流量劫持是通过 IPtable 转发实现的。

### Init 容器解析

istio 在 pod 中注入的 Init 容器为 istio-init，可以查看已经注入的 pod 的 init 容器启动参数（根据 YAML 文件描述）：

```shell
-p 15001 -u 1337 -m REDIRECT -i '*' -x "" -b 9080 -d ""
```

可以在源码中查看该容器的[Dockerfile](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxy_init)看看启动命令。

```dockerfile
FROM ubuntu:xenial
RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    iproute2 \
    iptables \
 && rm -rf /var/lib/apt/lists/*

ADD istio-iptables.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/istio-iptables.sh"]
```

我们看到 istio-init 容器的入口是/usr/local/bin/istio-iptables.sh 脚本，该脚本在 istio 源码仓库中是[istio-iptables.sh](https://github.com/istio/istio/blob/master/tools/deb/istio-iptables.sh)

#### init 容器启动入口

init 容器的启动入口是/usr/local/bin/istio-iptables.sh 脚本，该脚本的用法如下：

```shell
istio-iptables.sh -p PORT -u UID -g GID [-m mode] [-b ports] [-d ports] [-i CIDR] [-x CIDR] [-h]
  -p: 指定重定向所有 TCP 流量的 Envoy 端口（默认为 $ENVOY_PORT = 15001）
  -u: 指定未应用重定向的用户的 UID。通常，这是代理容器的 UID（默认为 $ENVOY_USER 的 uid，istio_proxy 的 uid 或 1337）
  -g: 指定未应用重定向的用户的 GID。（与 -u param 相同的默认值）
  -m: 指定入站连接重定向到 Envoy 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 Envoy 中排除（可选）的入站端口列表，以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -i: 指定重定向到 Envoy（可选）的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。

环境变量位于 $ISTIO_SIDECAR_CONFIG（默认在：/var/lib/istio/envoy/sidecar.env）
```

通过查看该脚本你将看到，以上传入的参数都会重新组装成 IPtable 命令参数。

在参考 istio-init 容器的启动参数，完整的启动  命令如下：

```shell
/usr/local/bin/istio-iptables.sh -p 15001 -u 1337 -m REDIRECT -i '*' -x "" -b 9080 -d ""
```

该容器存在的意义就是让 Envoy 代理可以拦截所有进出 pod 的流量，即将入站流量重定向到 Sidecar，在拦截应用容器的出站流量经过 Sidecar 处理后再出站。

**命令解析**

这条启动命令的作用是：

- 将应用容器的所有流量都转发到 Envoy 的 15001 端口。
- 使用 istio-proxy 用于身份运行，UID 为 1337，即 Envoy 所处的用户控件，这也是 istio-proxy 容器默认使用的用户，见 YAML 配置中的 runAsUser 字段。
- 使用默认的 REDIRECT 模式来重定向流量。
- 将所有出站流量重定向到 Envoy 代理。
- 将所有访问 9080 端口的流量重定向到 Envoy 代理。

因为 Init 容器初始化完成就会自动终止，因为我们无法登陆到容器中查看 IPtable 信息，但是 init 容器初始化结果会保留到应用容器和 Sidecar 容器中。

**istio-proxy 容器解析**

为了查看 IPtables 配置，我们需要登录到 Sidecar 容器中使用 root 用用户来查看，因为 kubectl 无法使用特权默认来远程操作 docker 容器，所以我们需要登录到 productpage POD 所在的主机上 docker 命令登录容器中看。

```shell
# 以特权模式进入容器，可以查看IPtable规则。
$ docker exec -i --privileged -t 781b67a78a70 bash

# 查看NAT 表中规则配置的详细信息
$ iptables -t nat -L -v
# PREROUTING 链：用于目标地址转换（DNAT），将所有入站TCP流量跳转到 ISTIO_INBOUND 链上
Chain PREROUTING (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    60 ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere

# INPUT 链：处理输入数据包，非TCP流量将继续OUTPUT链
Chain INPUT (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination

# OUTPUT 链：将所有出站数据包跳转到ISTIO_OUTPUT 链上
Chain OUTPUT (policy ACCEPT 1398 packets, 131K bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere

# POSTROUTING 链：所有数据包流出网卡时都要先进入POSTROUTING 链，内核根据数据包目的地判断是否需要转发出去，我们看到未做任何处理
Chain POSTROUTING (policy ACCEPT 1398 packets, 131K bytes)
 pkts bytes target     prot opt in     out     source               destination

# ISTIO_INBOUND 链：将所有目的端口为9080的端口入站流量重定向到ISTIO_IN_REDIRECT 链上
Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere             tcp dpt:9080

# ISTIO_IN_REDIRECT 链：将所有的入站流量跳转到本地的15001端口，至此成功的拦截了流量到Envoy
Chain ISTIO_IN_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001

# ISTIO_OUTPUT 链：选择需要重定向到Envoy（即本地）的出站流量，所有非localhost的流量全部转发到ISTIO_REDIRECT。
# 为了避免流量在该POD中无限循环，所有到istio-proxy用于空间的的流量都返回到它的调用点中的下一条规则，本例中即
# OUTPUT链，因为跳出ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非localhost 就跳转到
# ISTIO_REDIRECT；如果流量来自istio-proxy用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUTPUT
# 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是localhost 的流量就跳转到
# ISTIO_REDIRECT
Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  any    lo      anywhere            !localhost
    2   120 RETURN     all  --  any    any     anywhere             anywhere             owner UID match istio-proxy
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner GID match istio-proxy
    0     0 RETURN     all  --  any    any     anywhere             localhost
    0     0 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere

# ISTIO_REDIRECT 链：将所有流量重定向到Envoy（即本地）的15001 端口
Chain ISTIO_REDIRECT (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001

```

### 查看 iptables nat 表中注入的规则

init 容器通过向 IPtable nat 表中注入转发规则来劫持流量的，下图显示的是 productpage 服务中 IPtable 流量劫持的详细过程。

![Envoy Sidecar流量劫持流程示意图](https://jimmysong.io/istio-handbook/images/envoy-sidecar-traffic-interception-20190111.png)

init 容器启动时命令行参数中指定了 REDIRECT 模式（使用 REDIRECT 动过可以在本机上进行端口映射），因此只创建了 NAT 表规则。上面是查看 nat 表中的规则，其中链的  名字包含 ISTIO 前缀是有 init 容器注入的，规则匹配是根据下面显示的顺序来执行的，其中会有很多跳转。

## Sidecar 注入及流量劫持步骤

下面是 Sidecar 注入、Pod 启动到 Sidecar proxy 拦截流量及 Envoy 处理路由的步骤概览。

1. kubernetes 通过 Admission Controller 自动注入，或者用户使用 istioctl 命令手动注入 Sidecar 容器。
2. 应用 YAML 配置部署应用，此时 Kubernetes API server 接收到服务创建配置文件中  已经包含了 Init 容器  及 Sidecar proxy。
3. 在 Sidecar proxy 容器和应用程序启动之前，首先运行 Init 容器，Init 容器用于设置 IPtables（istio 中默认的流量拦截方式，还可以使用 BPF、IPVS 等方式）将进入 pod 的流量劫持到 Envoy Sidecar proxy。所有的 TCP 流量（Envoy 目前只支持流量）将被 sidecar 劫持，其他不支持协议的流量按原来的目的地请求。
4. 启动 Pod 中的 Envoy Sidecar proxy 和应用程序。
5. 无论是进入还是从 Pod 发出的 TCP 请求都会被 IPtables 劫持，inbound 流量被劫持后进 Inbound Handler 处理后转交给应用容器处理，outbound 流量被 IPtables 劫持后转交给 Outbound Handler 处理，并确定转发的 upstream 和 Endpoint。
6. Sidecar proxy 请求 Pilot 使用 xDS 协议同步 Envoy 配置，其中包括 LDS、EDS、CDS 等，不过为了保证更新的顺序，Envoy 会直接使用 ADS 向 Pilot 请求配置更新。

### Inbound Handler

Inbound Handler 的作用是将 IPtables 拦截到的 downstream 的流量转交给 localhost，与 pod 内的应用程序建立连接。

### Outbound Handler

Outbound Handler 的作用是将 iptables 拦截到本地应用程序发出的流量，经由 Envoy 判断如何路由到 upstream。

## 流量转移拆分

Envoy 的路由器可以跨两个或更多上游集群将流量拆分到虚拟主机中的路由。

### 在两个上游之间转移流量

路由配置中的运行时对象判断选择特定路由（以及它的集群）的可能性（可以理解为百分比）。通过使用运行时配置，虚拟主机中到特定路由的流量可逐渐从一个集群转移到另一个集群。按照官方给的示例，reviews 有三个版本，我们先用其中两个版本测试。

全部到 v1 版本的：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"hosts":["reviews"],"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"}}]}]}}
  creationTimestamp: "2019-01-28T22:30:48Z"
  generation: 6
  name: reviews
  namespace: default
  resourceVersion: "108464"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/reviews
  uid: 5c1a3e1e-234c-11e9-a865-000c2920f5ac
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
```

在 Envoy 配置中这时并没有设置权重。

30%到 v1 版本，70%到 v3 版本的：(使用 37 分比是为了查看调试方便，也可以使用其他的， 因为配置 json 文件  较大，使用 19 分比可能导致 10 或 90 值  有端口存在，不方便查找)

```yaml
# istio 看到的配置
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"hosts":["reviews"],"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"},"weight":30},{"destination":{"host":"reviews","subset":"v3"},"weight":70}]}]}}
  creationTimestamp: "2019-01-28T22:30:48Z"
  generation: 7
  name: reviews
  namespace: default
  resourceVersion: "108818"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/reviews
  uid: 5c1a3e1e-234c-11e9-a865-000c2920f5ac
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 30
        - destination:
            host: reviews
            subset: v3
          weight: 70
```

30%到 v1 版本，70%到 v3 版本的 Envoy 生成的配置(部分)：

```json
           "prefix": "/"
          },
          "route": {
           "weighted_clusters": {
            "clusters": [
             {
              "name": "outbound|80|v1|reviews.default.svc.cluster.local",
              "weight": 30,
              "per_filter_config": {
               "mixer": {
                "forward_attributes": {
--
               },
             {
              "name": "outbound|80|v3|reviews.default.svc.cluster.local",
              "weight": 70,
              "per_filter_config": {
               "mixer": {
                "forward_attributes": {
--
             "prefix": "/"
          },
          "route": {
           "weighted_clusters": {
            "clusters": [
             {
              "name": "outbound|9080|v1|reviews.default.svc.cluster.local",
              "weight": 30,
              "per_filter_config": {
               "mixer": {
                "mixer_attributes": {
--
               },
             {
              "name": "outbound|9080|v3|reviews.default.svc.cluster.local",
              "weight": 70,
              "per_filter_config": {
               "mixer": {
                "mixer_attributes": {
```

Envoy 使用 first Match 策略来匹配路由。如果路由具有运行对象，则会根据运行时值另外匹配请求（如果未指定值，则为默认值）。因此通过在上述示例中背靠背地放置路由并在第一个路由中指定运行时对象，可以通过更改运行时  值来完成流量转移。

默认情况下，权重的总和必须  精确地为 100。在 v2 API 中，总权重默认为 100，但是可以修改以允许更精确的粒度。

Envoy 大致  配置文件 JSON 结构：

```json
{
    "configs": {
        "listeners": {...},
        "clusters": {...},
        "bootstrap": {...},
        "routes": {
            "@type": "type.googleapis.com/envoy.admin.v2alpha.RoutesConfigDump",
            "static_route_configs": {...},
            "dynamic_route_configs": {...}
        }
    }
}
```

## 运维管理

## 调试 Envoy 和 Pilot

proxy-status 命令允许获取网格的概述并识别导致问题的代理。然后 proxy-status 可以用来检查 Envoy 配置并诊断问题。

### 了解服务网格概况

proxy-status 命令允许获取网格的状态。如果怀疑其中一个 sidecar 没有收到配置或者不同步，那么代理状态会通知你。

```shell
istioctl proxy-status
```

如果此列表中缺少代理，则表示它当前未连接到 Pilot 实例，因此不会接受任何配置。

- SYNCED 表示 Envoy 已经确认了 Pilot 上次发送给它的实例。
- SYNCED(100%) 表示 Envoy 已经成功同步了集群中的所有端点。
- NOT SENT 表示 Pilot 没有发送任何消息给 Envoy。这通常是因为 Pilot 没有任何数据可以发送。
- STALE 表示 Pilot 已向 Envoy 发送更新但尚未收到确认。这通常表明 Envoy 和 Pilot 之间的网络存在问题或者 Istio 本身的错误。

## 关于 Envoy 性能优化

- [浅谈 Service Mesh 体系中的 Envoy](https://yq.aliyun.com/articles/606655)
- [具备 API 感知的网络和安全性管理的开源软件 Cilium](https://jimmysong.io/kubernetes-handbook/concepts/cilium.html)
- [eBPF 简史](https://www.ibm.com/developerworks/cn/linux/l-lo-eBPF-history/index.html)
