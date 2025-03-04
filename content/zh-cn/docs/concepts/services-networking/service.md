---
title: 服务（Service）
feature:
  title: 服务发现与负载均衡
  description: >
    无需修改你的应用程序去使用陌生的服务发现机制。Kubernetes 为容器提供了自己的 IP 地址和一个 DNS 名称，并且可以在它们之间实现负载均衡。
description: >-
  将在集群中运行的应用程序暴露在单个外向端点后面，即使工作负载分散到多个后端也是如此。
content_type: concept
weight: 10
---
<!--
reviewers:
- bprashanth
title: Service
feature:
  title: Service discovery and load balancing
  description: >
    No need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.
description: >-
  Expose an application running in your cluster behind a single outward-facing
  endpoint, even when the workload is split across multiple backends.
content_type: concept
weight: 10
-->

<!-- overview -->

{{< glossary_definition term_id="service" length="short" >}}

<!--
With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism.
Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods,
and can load-balance across them.
-->
使用 Kubernetes，你无需修改应用程序去使用不熟悉的服务发现机制。
Kubernetes 为 Pod 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名，
并且可以在它们之间进行负载均衡。

<!-- body -->

<!--
## Motivation

Kubernetes {{< glossary_tooltip term_id="pod" text="Pods" >}} are created and destroyed
to match the desired state of your cluster. Pods are nonpermanent resources.
If you use a {{< glossary_tooltip term_id="deployment" >}} to run your app,
it can create and destroy Pods dynamically.

Each Pod gets its own IP address, however in a Deployment, the set of Pods
running in one moment in time could be different from
the set of Pods running that application a moment later.

This leads to a problem: if some set of Pods (call them "backends") provides
functionality to other Pods (call them "frontends") inside your cluster,
how do the frontends find out and keep track of which IP address to connect
to, so that the frontend can use the backend part of the workload?

Enter _Services_.
-->
## 动机   {#motivation}

创建和销毁 Kubernetes {{< glossary_tooltip term_id="pod" text="Pod" >}} 以匹配集群的期望状态。
Pod 是非永久性资源。
如果你使用 {{< glossary_tooltip term_id="deployment">}}
来运行你的应用程序，则它可以动态创建和销毁 Pod。

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

这导致了一个问题： 如果一组 Pod（称为“后端”）为集群内的其他 Pod（称为“前端”）提供功能，
那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用提供工作负载的后端部分？

进入 **Services**。

<!--
## Service resources {#service-resource}
-->
## Service 资源 {#service-resource}

<!--
In Kubernetes, a Service is an abstraction which defines a logical set of Pods
and a policy by which to access them (sometimes this pattern is called
a micro-service). The set of Pods targeted by a Service is usually determined
by a {{< glossary_tooltip text="selector" term_id="selector" >}}.
To learn about other ways to define Service endpoints,
see [Services _without_ selectors](#services-without-selectors).
-->
Kubernetes Service 定义了这样一种抽象：逻辑上的一组 Pod，一种可以访问它们的策略 —— 通常称为微服务。
Service 所针对的 Pod 集合通常是通过{{< glossary_tooltip text="选择算符" term_id="selector" >}}来确定的。
要了解定义服务端点的其他方法，请参阅[不带选择算符的服务](#services-without-selectors)。

<!--
For example, consider a stateless image-processing backend which is running with
3 replicas.  Those replicas are fungible&mdash;frontends do not care which backend
they use.  While the actual Pods that compose the backend set may change, the
frontend clients should not need to be aware of that, nor should they need to keep
track of the set of backends themselves.

The Service abstraction enables this decoupling.
-->
举个例子，考虑一个图片处理后端，它运行了 3 个副本。这些副本是可互换的 ——
前端不需要关心它们调用了哪个后端副本。
然而组成这一组后端程序的 Pod 实际上可能会发生变化，
前端客户端不应该也没必要知道，而且也不需要跟踪这一组后端的状态。

Service 定义的抽象能够解耦这种关联。

<!--
### Cloud-native service discovery

If you're able to use Kubernetes APIs for service discovery in your application,
you can query the {{< glossary_tooltip text="API server" term_id="kube-apiserver" >}}
for matching EndpointSlices. Kubernetes updates the EndpointSlices for a Service
whenever the set of Pods in a Service changes.

For non-native applications, Kubernetes offers ways to place a network port or load
balancer in between your application and the backend Pods.
-->
### 云原生服务发现   {#cloud-native-discovery}

如果你想要在应用程序中使用 Kubernetes API 进行服务发现，则可以查询
{{< glossary_tooltip text="API 服务器" term_id="kube-apiserver" >}}
用于匹配 EndpointSlices。只要服务中的 Pod 集合发生更改，Kubernetes 就会为服务更新 EndpointSlices。

对于非本机应用程序，Kubernetes 提供了在应用程序和后端 Pod 之间放置网络端口或负载均衡器的方法。

<!--
## Defining a Service

A Service in Kubernetes is a REST object, similar to a Pod.  Like all of the
REST objects, you can `POST` a Service definition to the API server to create
a new instance.
The name of a Service object must be a valid
[RFC 1035 label name](/docs/concepts/overview/working-with-objects/names#rfc-1035-label-names).

For example, suppose you have a set of Pods where each listens on TCP port 9376
and contains a label `app.kubernetes.io/name=MyApp`:
-->
## 定义 Service   {#defining-a-service}

Service 在 Kubernetes 中是一个 REST 对象，和 Pod 类似。
像所有的 REST 对象一样，Service 定义可以基于 `POST` 方式，请求 API server 创建新的实例。
Service 对象的名称必须是合法的
[RFC 1035 标签名称](/zh-cn/docs/concepts/overview/working-with-objects/names#rfc-1035-label-names)。

例如，假定有一组 Pod，它们对外暴露了 9376 端口，同时还被打上 `app.kubernetes.io/name=MyApp` 标签：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

<!--
This specification creates a new Service object named "my-service", which
targets TCP port 9376 on any Pod with the `app.kubernetes.io/name=MyApp` label.

Kubernetes assigns this Service an IP address (sometimes called the "cluster IP"),
which is used by the Service proxies
(see [Virtual IP addressing mechanism](#virtual-ip-addressing-mechanism) below).

The controller for the Service selector continuously scans for Pods that
match its selector, and then POSTs any updates to an Endpoint object
also named "my-service".
-->
上述配置创建一个名称为 "my-service" 的 Service 对象，它会将请求代理到使用
TCP 端口 9376，并且具有标签 `app.kubernetes.io/name=MyApp` 的 Pod 上。

Kubernetes 为该服务分配一个 IP 地址（有时称为 “集群 IP”），该 IP 地址由服务代理使用。
(请参见下面的[虚拟 IP 寻址机制](#virtual-ip-addressing-mechanism)).

服务选择算符的控制器不断扫描与其选择算符匹配的 Pod，然后将所有更新发布到也称为
“my-service” 的 Endpoint 对象。

{{< note >}}
<!--
A Service can map _any_ incoming `port` to a `targetPort`. By default and
for convenience, the `targetPort` is set to the same value as the `port`
field.
-->
需要注意的是，Service 能够将一个接收 `port` 映射到任意的 `targetPort`。
默认情况下，`targetPort` 将被设置为与 `port` 字段相同的值。
{{< /note >}}

<!--
Port definitions in Pods have names, and you can reference these names in the
`targetPort` attribute of a Service. For example, we can bind the `targetPort`
of the Service to the Pod port in the following way:
-->
Pod 中的端口定义是有名字的，你可以在 Service 的 `targetPort` 属性中引用这些名称。
例如，我们可以通过以下方式将 Service 的 `targetPort` 绑定到 Pod 端口：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

<!--
This works even if there is a mixture of Pods in the Service using a single
configured name, with the same network protocol available via different
port numbers. This offers a lot of flexibility for deploying and evolving
your Services. For example, you can change the port numbers that Pods expose
in the next version of your backend software, without breaking clients.
-->
即使 Service 中使用同一配置名称混合使用多个 Pod，各 Pod 通过不同的端口号支持相同的网络协议，
此功能也可以使用。这为 Service 的部署和演化提供了很大的灵活性。
例如，你可以在新版本中更改 Pod 中后端软件公开的端口号，而不会破坏客户端。

<!--
The default protocol for Services is 
[TCP](/docs/reference/networking/service-protocols/#protocol-tcp); you can also
use any other [supported protocol](/docs/reference/networking/service-protocols/).

As many Services need to expose more than one port, Kubernetes supports multiple
port definitions on a Service object.
Each port definition can have the same `protocol`, or a different one.
-->
服务的默认协议是 [TCP](/zh-cn/docs/reference/networking/service-protocols/#protocol-tcp)；
你还可以使用任何其他[受支持的协议](/zh-cn/docs/reference/networking/service-protocols/)。

由于许多服务需要公开多个端口，因此 Kubernetes 在服务对象上支持多个端口定义。
每个端口定义可以具有相同的 `protocol`，也可以具有不同的协议。

<!--
### Services without selectors

Services most commonly abstract access to Kubernetes Pods thanks to the selector,
but when used with a corresponding set of
{{<glossary_tooltip term_id="endpoint-slice" text="EndpointSlices">}}
objects and without a selector, the Service can abstract other kinds of backends,
including ones that run outside the cluster.

For example:

* You want to have an external database cluster in production, but in your
  test environment you use your own databases.
* You want to point your Service to a Service in a different
  {{< glossary_tooltip term_id="namespace" >}} or on another cluster.
* You are migrating a workload to Kubernetes. While evaluating the approach,
  you run only a portion of your backends in Kubernetes.

In any of these scenarios you can define a Service _without_ a Pod selector.
For example:
-->
### 没有选择算符的 Service   {#services-without-selectors}

由于选择算符的存在，服务最常见的用法是为 Kubernetes Pod 的访问提供抽象，
但是当与相应的 {{<glossary_tooltip term_id="endpoint-slice" text="EndpointSlices">}}
对象一起使用且没有选择算符时，
服务也可以为其他类型的后端提供抽象，包括在集群外运行的后端。

例如：

  * 希望在生产环境中使用外部的数据库集群，但测试环境使用自己的数据库。
  * 希望服务指向另一个 {{< glossary_tooltip term_id="namespace" >}} 中或其它集群中的服务。
  * 你正在将工作负载迁移到 Kubernetes。在评估该方法时，你仅在 Kubernetes 中运行一部分后端。

在任何这些场景中，都能够定义没有选择算符的 Service。
实例:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

<!--
Because this Service has no selector, the corresponding EndpointSlice (and
legacy Endpoints) objects are not created automatically. You can manually map the Service
to the network address and port where it's running, by adding an EndpointSlice
object manually. For example:
-->
由于此服务没有选择算符，因此不会自动创建相应的 EndpointSlice（和旧版 Endpoint）对象。
你可以通过手动添加 EndpointSlice 对象，将服务手动映射到运行该服务的网络地址和端口：

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1 # 按惯例将服务的名称用作 EndpointSlice 名称的前缀
  labels:
    # 你应设置 "kubernetes.io/service-name" 标签。
    # 设置其值以匹配服务的名称
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: '' # 留空，因为 port 9376 未被 IANA 分配为已注册端口
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6" # 此列表中的 IP 地址可以按任何顺序显示
      - "10.1.2.3"
```

<!--
#### Custom EndpointSlices

When you create an [EndpointSlice](#endpointslices) object for a Service, you can
use any name for the EndpointSlice. Each EndpointSlice in a namespace must have a
unique name. You link an EndpointSlice to a Service by setting the
`kubernetes.io/service-name` {{< glossary_tooltip text="label" term_id="label" >}}
on that EndpointSlice.
-->
#### 自定义 EndpointSlices

当为服务创建 [EndpointSlice](#endpointslices) 对象时，可以为 EndpointSlice 使用任何名称。
命名空间中的每个 EndpointSlice 必须有一个唯一的名称。通过在 EndpointSlice 上设置
`kubernetes.io/service-name` {{< glossary_tooltip text="label" term_id="label" >}}
可以将 EndpointSlice 链接到服务。

{{< note >}}
<!--
The endpoint IPs _must not_ be: loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or
link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).

The endpoint IP addresses cannot be the cluster IPs of other Kubernetes Services,
because {{< glossary_tooltip term_id="kube-proxy" >}} doesn't support virtual IPs
as a destination.
-->
端点 IP 地址**必须不是** ：本地回路地址（IPv4 的 127.0.0.0/8、IPv6 的 ::1/128）
或链路本地地址（IPv4 的 169.254.0.0/16 和 224.0.0.0/24、IPv6 的 fe80::/64）。

端点 IP 地址不能是其他 Kubernetes 服务的集群 IP，因为
{{< glossary_tooltip term_id ="kube-proxy">}} 不支持将虚拟 IP 作为目标。
{{< /note >}}

<!--
For an EndpointSlice that you create yourself, or in your own code,
you should also pick a value to use for the [`endpointslice.kubernetes.io/managed-by`](/docs/reference/labels-annotations-taints/#endpointslicekubernetesiomanaged-by) label.
If you create your own controller code to manage EndpointSlices, consider using a
value similar to `"my-domain.example/name-of-controller"`. If you are using a third
party tool, use the name of the tool in all-lowercase and change spaces and other
punctuation to dashes (`-`).
If people are directly using a tool such as `kubectl` to manage EndpointSlices,
use a name that describes this manual management, such as `"staff"` or
`"cluster-admins"`. You should
avoid using the reserved value `"controller"`, which identifies EndpointSlices
managed by Kubernetes' own control plane.
-->
对于你自己或在你自己代码中创建的 EndpointSlice，你还应该为
[`endpointslice.kubernetes.io/managed-by`](/zh-cn/docs/reference/labels-annotations-taints/#endpointslicekubernetesiomanaged-by)
标签拣选一个值。如果你创建自己的控制器代码来管理 EndpointSlice，
请考虑使用类似于 `"my-domain.example/name-of-controller"` 的值。
如果你使用的是第三方工具，请使用全小写的工具名称，并将空格和其他标点符号更改为短划线 (`-`)。
如果人们直接使用 `kubectl` 之类的工具来管理 EndpointSlices，请使用描述这种手动管理的名称，
例如 `"staff"` 或 `"cluster-admins"`。你应该避免使用保留值 `"controller"`，
该值标识由 Kubernetes 自己的控制平面管理的 EndpointSlices。

<!--
#### Accessing a Service without a selector {#service-no-selector-access}

Accessing a Service without a selector works the same as if it had a selector.
In the [example](#services-without-selectors) for a Service without a selector,
traffic is routed to one of the two endpoints defined in
the EndpointSlice manifest: a TCP connection to 10.1.2.3 or 10.4.5.6, on port 9376.
-->
#### 访问没有选择算符的 Service   {#service-no-selector-access}

访问没有选择算符的 Service，与有选择算符的 Service 的原理相同。
在没有选择算符的 Service [示例](#services-without-selectors)中，
流量被路由到 EndpointSlice 清单中定义的两个端点之一：
通过 TCP 协议连接到 10.1.2.3 或 10.4.5.6 的端口 9376。

<!--
An ExternalName Service is a special case of Service that does not have
selectors and uses DNS names instead. For more information, see the
[ExternalName](#externalname) section later in this document.
-->
ExternalName Service 是 Service 的特例，它没有选择算符，但是使用 DNS 名称。
有关更多信息，请参阅本文档后面的 [ExternalName](#externalname)。

<!--
### EndpointSlices
-->
### EndpointSlices

{{< feature-state for_k8s_version="v1.21" state="stable" >}}

<!--
[EndpointSlices](/docs/concepts/services-networking/endpoint-slices/) are objects that
represent a subset (a _slice_) of the backing network endpoints for a Service.

Your Kubernetes cluster tracks how many endpoints each EndpointSlice represents.
If there are so many endpoints for a Service that a threshold is reached, then
Kubernetes adds another empty EndpointSlice and stores new endpoint information
there.
By default, Kubernetes makes a new EndpointSlice once the existing EndpointSlices
all contain at least 100 endpoints. Kubernetes does not make the new EndpointSlice
until an extra endpoint needs to be added.

See [EndpointSlices](/docs/concepts/services-networking/endpoint-slices/) for more
information about this API.
-->
[EndpointSlices](/zh-cn/docs/concepts/services-networking/endpoint-slices/)
这些对象表示针对服务的后备网络端点的子集（**切片**）。

你的 Kubernetes 集群会跟踪每个 EndpointSlice 表示的端点数量。
如果服务的端点太多以至于达到阈值，Kubernetes 会添加另一个空的 EndpointSlice 并在其中存储新的端点信息。
默认情况下，一旦现有 EndpointSlice 都包含至少 100 个端点，Kubernetes 就会创建一个新的 EndpointSlice。
在需要添加额外的端点之前，Kubernetes 不会创建新的 EndpointSlice。

参阅 [EndpointSlices](/zh-cn/docs/concepts/services-networking/endpoint-slices/)
了解有关该 API 的更多信息。

<!--
### Endpoints

In the Kubernetes API, an
[Endpoints](/docs/reference/kubernetes-api/service-resources/endpoints-v1/)
(the resource kind is plural) defines a list of network endpoints, typically
referenced by a Service to define which Pods the traffic can be sent to.

The EndpointSlice API is the recommended replacement for Endpoints.
-->
### Endpoints

在 Kubernetes API 中，[Endpoints](/zh-cn/docs/reference/kubernetes-api/service-resources/endpoints-v1/)
（该资源类别为复数）定义了网络端点的列表，通常由 Service 引用，以定义可以将流量发送到哪些 Pod。

推荐用 EndpointSlice API 替换 Endpoints。

<!--
#### Over-capacity endpoints

Kubernetes limits the number of endpoints that can fit in a single Endpoints
object. When there are over 1000 backing endpoints for a Service, Kubernetes
truncates the data in the Endpoints object. Because a Service can be linked
with more than one EndpointSlice, the 1000 backing endpoint limit only
affects the legacy Endpoints API.
-->
#### 超出容量的端点

Kubernetes 限制单个 Endpoints 对象中可以容纳的端点数量。
当一个服务有超过 1000 个后备端点时，Kubernetes 会截断 Endpoints 对象中的数据。
由于一个服务可以链接多个 EndpointSlice，所以 1000 个后备端点的限制仅影响旧版的 Endpoints API。

<!--
In that case, Kubernetes selects at most 1000 possible backend endpoints to store
into the Endpoints object, and sets an
{{< glossary_tooltip text="annotation" term_id="annotation" >}} on the
Endpoints:
[`endpoints.kubernetes.io/over-capacity: truncated`](/docs/reference/labels-annotations-taints/#endpoints-kubernetes-io-over-capacity).
The control plane also removes that annotation if the number of backend Pods drops below 1000.
-->
这种情况下，Kubernetes 选择最多 1000 个可能的后端端点来存储到 Endpoints 对象中，并在
Endpoints: [`endpoints.kubernetes.io/over-capacity: truncated`](/zh-cn/docs/reference/labels-annotations-taints/#endpoints-kubernetes-io-over-capacity)
上设置{{< glossary_tooltip text="注解" term_id="annotation" >}}。
如果后端 Pod 的数量低于 1000，控制平面也会移除该注解。

<!--
Traffic is still sent to backends, but any load balancing mechanism that relies on the
legacy Endpoints API only sends traffic to at most 1000 of the available backing endpoints.

The same API limit means that you cannot manually update an Endpoints to have more than 1000 endpoints.

### Application protocol
-->
流量仍会发送到后端，但任何依赖旧版 Endpoints API 的负载均衡机制最多只能将流量发送到 1000 个可用的后备端点。

相同的 API 限制意味着你不能手动将 Endpoints 更新为拥有超过 1000 个端点。

### 应用协议    {#application-protocol}

{{< feature-state for_k8s_version="v1.20" state="stable" >}}

<!-- 
The `appProtocol` field provides a way to specify an application protocol for
each Service port. The value of this field is mirrored by the corresponding
Endpoints and EndpointSlice objects.

This field follows standard Kubernetes label syntax. Values should either be
[IANA standard service names](https://www.iana.org/assignments/service-names) or
domain prefixed names such as `mycompany.com/my-custom-protocol`.
-->
`appProtocol` 字段提供了一种为每个 Service 端口指定应用协议的方式。
此字段的取值会被映射到对应的 Endpoints 和 EndpointSlices 对象。

该字段遵循标准的 Kubernetes 标签语法。
其值可以是 [IANA 标准服务名称](https://www.iana.org/assignments/service-names)
或以域名为前缀的名称，如 `mycompany.com/my-custom-protocol`。


<!--
## Multi-Port Services

For some Services, you need to expose more than one port.
Kubernetes lets you configure multiple port definitions on a Service object.
When using multiple ports for a Service, you must give all of your ports names
so that these are unambiguous.
For example:
-->
## 多端口 Service   {#multi-port-services}

对于某些服务，你需要公开多个端口。
Kubernetes 允许你在 Service 对象上配置多个端口定义。
为服务使用多个端口时，必须提供所有端口名称，以使它们无歧义。
例如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

{{< note >}}
<!--
As with Kubernetes {{< glossary_tooltip term_id="name" text="names">}} in general, names for ports
must only contain lowercase alphanumeric characters and `-`. Port names must
also start and end with an alphanumeric character.

For example, the names `123-abc` and `web` are valid, but `123_abc` and `-web` are not.
-->
与一般的 Kubernetes 名称一样，端口名称只能包含小写字母数字字符 和 `-`。
端口名称还必须以字母数字字符开头和结尾。

例如，名称 `123-abc` 和 `web` 有效，但是 `123_abc` 和 `-web` 无效。
{{< /note >}}

<!--
## Choosing your own IP address

You can specify your own cluster IP address as part of a `Service` creation
request.  To do this, set the `.spec.clusterIP` field. For example, if you
already have an existing DNS entry that you wish to reuse, or legacy systems
that are configured for a specific IP address and difficult to re-configure.

The IP address that you choose must be a valid IPv4 or IPv6 address from within the
`service-cluster-ip-range` CIDR range that is configured for the API server.
If you try to create a Service with an invalid clusterIP address value, the API
server will return a 422 HTTP status code to indicate that there's a problem.
-->
## 选择自己的 IP 地址   {#choosing-your-own-ip-address}

在 `Service` 创建的请求中，可以通过设置 `spec.clusterIP` 字段来指定自己的集群 IP 地址。
比如，希望替换一个已经已存在的 DNS 条目，或者遗留系统已经配置了一个固定的 IP 且很难重新配置。

用户选择的 IP 地址必须合法，并且这个 IP 地址在 `service-cluster-ip-range` CIDR 范围内，
这对 API 服务器来说是通过一个标识来指定的。
如果 IP 地址不合法，API 服务器会返回 HTTP 状态码 422，表示值不合法。

<!--
## Discovering services

Kubernetes supports 2 primary modes of finding a Service - environment
variables and DNS.
-->
## 服务发现  {#discovering-services}

Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS。

<!--
### Environment variables

When a Pod is run on a Node, the kubelet adds a set of environment variables
for each active Service. It adds `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT` variables,
where the Service name is upper-cased and dashes are converted to underscores.
It also supports variables (see [makeLinkVariables](https://github.com/kubernetes/kubernetes/blob/dd2d12f6dc0e654c15d5db57a5f9f6ba61192726/pkg/kubelet/envvars/envvars.go#L72))
that are compatible with Docker Engine's
"_[legacy container links](https://docs.docker.com/network/links/)_" feature.

For example, the Service `redis-primary` which exposes TCP port 6379 and has been
allocated cluster IP address 10.0.0.11, produces the following environment
variables:
-->
### 环境变量   {#environment-variables}

当 Pod 运行在 `Node` 上，kubelet 会为每个活跃的 Service 添加一组环境变量。
kubelet 为 Pod 添加环境变量 `{SVCNAME}_SERVICE_HOST` 和 `{SVCNAME}_SERVICE_PORT`。
这里 Service 的名称需大写，横线被转换成下划线。
它还支持与 Docker Engine 的 "**[legacy container links](https://docs.docker.com/network/links/)**" 特性兼容的变量
（参阅 [makeLinkVariables](https://github.com/kubernetes/kubernetes/blob/dd2d12f6dc0e654c15d5db57a5f9f6ba61192726/pkg/kubelet/envvars/envvars.go#L72)) 。

举个例子，一个名称为 `redis-primary` 的 Service 暴露了 TCP 端口 6379，
同时给它分配了 Cluster IP 地址 10.0.0.11，这个 Service 生成了如下环境变量：

```shell
REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
REDIS_PRIMARY_SERVICE_PORT=6379
REDIS_PRIMARY_PORT=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP_PROTO=tcp
REDIS_PRIMARY_PORT_6379_TCP_PORT=6379
REDIS_PRIMARY_PORT_6379_TCP_ADDR=10.0.0.11
```

{{< note >}}
<!--
When you have a Pod that needs to access a Service, and you are using
the environment variable method to publish the port and cluster IP to the client
Pods, you must create the Service *before* the client Pods come into existence.
Otherwise, those client Pods won't have their environment variables populated.

If you only use DNS to discover the cluster IP for a Service, you don't need to
worry about this ordering issue.
-->
当你具有需要访问服务的 Pod 时，并且你正在使用环境变量方法将端口和集群 IP 发布到客户端
Pod 时，必须在客户端 Pod 出现 **之前** 创建服务。
否则，这些客户端 Pod 将不会设定其环境变量。

如果仅使用 DNS 查找服务的集群 IP，则无需担心此设定问题。
{{< /note >}}

### DNS

<!--
You can (and almost always should) set up a DNS service for your Kubernetes
cluster using an [add-on](/docs/concepts/cluster-administration/addons/).

A cluster-aware DNS server, such as CoreDNS, watches the Kubernetes API for new
Services and creates a set of DNS records for each one.  If DNS has been enabled
throughout your cluster then all Pods should automatically be able to resolve
Services by their DNS name.
-->
你可以（几乎总是应该）使用[附加组件](/zh-cn/docs/concepts/cluster-administration/addons/)
为 Kubernetes 集群设置 DNS 服务。

支持集群的 DNS 服务器（例如 CoreDNS）监视 Kubernetes API 中的新服务，并为每个服务创建一组 DNS 记录。
如果在整个集群中都启用了 DNS，则所有 Pod 都应该能够通过其 DNS 名称自动解析服务。

<!--
For example, if you have a Service called `my-service` in a Kubernetes
namespace `my-ns`, the control plane and the DNS Service acting together
create a DNS record for `my-service.my-ns`. Pods in the `my-ns` namespace
should be able to find the service by doing a name lookup for `my-service`
(`my-service.my-ns` would also work).

Pods in other namespaces must qualify the name as `my-service.my-ns`. These names
will resolve to the cluster IP assigned for the Service.
-->
例如，如果你在 Kubernetes 命名空间 `my-ns` 中有一个名为 `my-service` 的服务，
则控制平面和 DNS 服务共同为 `my-service.my-ns` 创建 DNS 记录。
`my-ns` 命名空间中的 Pod 应该能够通过按名检索 `my-service` 来找到服务
（`my-service.my-ns` 也可以工作）。

其他命名空间中的 Pod 必须将名称限定为 `my-service.my-ns`。
这些名称将解析为为服务分配的集群 IP。

<!--
Kubernetes also supports DNS SRV (Service) records for named ports.  If the
`my-service.my-ns` Service has a port named `http` with the protocol set to
`TCP`, you can do a DNS SRV query for `_http._tcp.my-service.my-ns` to discover
the port number for `http`, as well as the IP address.

The Kubernetes DNS server is the only way to access `ExternalName` Services.
You can find more information about `ExternalName` resolution in
[DNS Pods and Services](/docs/concepts/services-networking/dns-pod-service/).
-->
Kubernetes 还支持命名端口的 DNS SRV（服务）记录。
如果 `my-service.my-ns` 服务具有名为 `http`　的端口，且协议设置为 TCP，
则可以对 `_http._tcp.my-service.my-ns` 执行 DNS SRV 查询以发现该端口号、`"http"` 以及 IP 地址。

Kubernetes DNS 服务器是唯一的一种能够访问 `ExternalName` 类型的 Service 的方式。
更多关于 `ExternalName` 信息可以查看
[DNS Pod 和 Service](/zh-cn/docs/concepts/services-networking/dns-pod-service/)。

<!--
## Headless Services  {#headless-services}

Sometimes you don't need load-balancing and a single Service IP.  In
this case, you can create what are termed "headless" Services, by explicitly
specifying `"None"` for the cluster IP (`.spec.clusterIP`).

You can use a headless Service to interface with other service discovery mechanisms,
without being tied to Kubernetes' implementation.

For headless `Services`, a cluster IP is not allocated, kube-proxy does not handle
these Services, and there is no load balancing or proxying done by the platform
for them. How DNS is automatically configured depends on whether the Service has
selectors defined:
-->
## 无头服务（Headless Services）  {#headless-services}

有时不需要或不想要负载均衡，以及单独的 Service IP。
遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"`
来创建 `Headless` Service。

你可以使用一个无头 Service 与其他服务发现机制进行接口，而不必与 Kubernetes 的实现捆绑在一起。

对于无头 `Services` 并不会分配 Cluster IP，kube-proxy 不会处理它们，
而且平台也不会为它们进行负载均衡和路由。
DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符。

<!--
### With selectors

For headless Services that define selectors, the Kubernetes control plane creates
EndpointSlice objects in the Kubernetes API, and modifies the DNS configuration to return
A or AAAA records (IPv4 or IPv6 addresses) that point directly to the Pods backing
the Service.
-->
### 带选择算符的服务 {#with-selectors}

对定义了选择算符的无头服务，Kubernetes 控制平面在 Kubernetes API 中创建 EndpointSlice 对象，
并且修改 DNS 配置返回 A 或 AAA 条记录（IPv4 或 IPv6 地址），通过这个地址直接到达 `Service` 的后端 Pod 上。

<!--
### Without selectors

For headless Services that do not define selectors, the control plane does
not create EndpointSlice objects. However, the DNS system looks for and configures
either:

* DNS CNAME records for [`type: ExternalName`](#externalname) Services.
* DNS A / AAAA records for all IP addresses of the Service's ready endpoints,
  for all Service types other than `ExternalName`.
  * For IPv4 endpoints, the DNS system creates A records.
  * For IPv6 endpoints, the DNS system creates AAAA records.
-->
### 无选择算符的服务  {#without-selectors}

对没有定义选择算符的无头服务，控制平面不会创建 EndpointSlice 对象。
然而 DNS 系统会查找和配置以下之一：

* 对于 [`type: ExternalName`](#externalname) 服务，查找和配置其 CNAME 记录
* 对所有其他类型的服务，针对 Service 的就绪端点的所有 IP 地址，查找和配置 DNS A / AAAA 条记录
  * 对于 IPv4 端点，DNS 系统创建 A 条记录。
  * 对于 IPv6 端点，DNS 系统创建 AAAA 条记录。

<!--
## Publishing Services (ServiceTypes) {#publishing-services-service-types}

For some parts of your application (for example, frontends) you may want to expose a
Service onto an external IP address, that's outside of your cluster.

Kubernetes `ServiceTypes` allow you to specify what kind of Service you want.

`Type` values and their behaviors are:
-->
## 发布服务（服务类型）      {#publishing-services-service-types}

对一些应用的某些部分（如前端），可能希望将其暴露给 Kubernetes 集群外部的 IP 地址。

Kubernetes `ServiceTypes` 允许指定你所需要的 Service 类型。

`Type` 的取值以及行为如下：

<!--
* `ClusterIP`: Exposes the Service on a cluster-internal IP. Choosing this value
  makes the Service only reachable from within the cluster. This is the
  default that is used if you don't explicitly specify a `type` for a Service.
* [`NodePort`](#type-nodeport): Exposes the Service on each Node's IP at a static port
  (the `NodePort`).
  To make the node port available, Kubernetes sets up a cluster IP address,
  the same as if you had requested a Service of `type: ClusterIP`.
* [`LoadBalancer`](#loadbalancer): Exposes the Service externally using a cloud
  provider's load balancer.
* [`ExternalName`](#externalname): Maps the Service to the contents of the
  `externalName` field (e.g. `foo.bar.example.com`), by returning a `CNAME` record
  with its value. No proxying of any kind is set up.
-->
* `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。
  这也是你没有为服务显式指定 `type` 时使用的默认值。
* [`NodePort`](#type-nodeport)：通过每个节点上的 IP 和静态端口（`NodePort`）暴露服务。
  为了让节点端口可用，Kubernetes 设置了集群 IP 地址，这等同于你请求 `type: ClusterIP` 的服务。
* [`LoadBalancer`](#loadbalancer)：使用云提供商的负载均衡器向外部暴露服务。
  外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
* [`ExternalName`](#externalname)：通过返回 `CNAME` 记录和对应值，可以将服务映射到
  `externalName` 字段的内容（例如，`foo.bar.example.com`）。
  无需创建任何类型代理。

  {{< note >}}
  <!--
  You need either `kube-dns` version 1.7 or CoreDNS version 0.0.8 or higher
  to use the `ExternalName` type.
  -->
  你需要使用 kube-dns 1.7 及以上版本或者 CoreDNS 0.0.8 及以上版本才能使用 `ExternalName` 类型。
  {{< /note >}}

<!--
The `type` field was designed as nested functionality - each level adds to the
previous.  This is not strictly required on all cloud providers (for example: Google
Compute Engine does not need to allocate a node port to make `type: LoadBalancer` work,
but another cloud provider integration might do). Although strict nesting is not required,
but the Kubernetes API design for Service requires it anyway.
-->
`type` 字段被设计为嵌套功能 - 每个级别都添加到前一个级别。
这并不是所有云提供商都严格要求的（例如：Google Compute Engine
不需要分配节点端口来使 `type: LoadBalancer` 工作，但另一个云提供商集成可能会这样做）。
虽然不需要严格的嵌套，但是 Service 的 Kubernetes API 设计无论如何都需要它。

<!--
You can also use [Ingress](/docs/concepts/services-networking/ingress/) to expose your Service.
Ingress is not a Service type, but it acts as the entry point for your cluster.
It lets you consolidate your routing rules into a single resource as it can expose multiple
services under the same IP address.
-->
你也可以使用 [Ingress](/zh-cn/docs/concepts/services-networking/ingress/) 来暴露自己的服务。
Ingress 不是一种服务类型，但它充当集群的入口点。
它可以将路由规则整合到一个资源中，因为它可以在同一 IP 地址下公开多个服务。

<!--
### Type NodePort {#type-nodeport}

If you set the `type` field to `NodePort`, the Kubernetes control plane
allocates a port from a range specified by `--service-node-port-range` flag (default: 30000-32767).
Each node proxies that port (the same port number on every Node) into your Service.
Your Service reports the allocated port in its `.spec.ports[*].nodePort` field.

Using a NodePort gives you the freedom to set up your own load balancing solution,
to configure environments that are not fully supported by Kubernetes, or even
to expose one or more nodes' IP addresses directly.
-->
### NodePort 类型  {#type-nodeport}

如果你将 `type` 字段设置为 `NodePort`，则 Kubernetes 控制平面将在
`--service-node-port-range` 标志指定的范围内分配端口（默认值：30000-32767）。
每个节点将那个端口（每个节点上的相同端口号）代理到你的服务中。
你的服务在其 `.spec.ports[*].nodePort` 字段中报告已分配的端口。

使用 NodePort 可以让你自由设置自己的负载均衡解决方案，
配置 Kubernetes 不完全支持的环境，
甚至直接暴露一个或多个节点的 IP 地址。

<!--
For a node port Service, Kubernetes additionally allocates a port (TCP, UDP or
SCTP to match the protocol of the Service). Every node in the cluster configures
itself to listen on that assigned port and to forward traffic to one of the ready
endpoints associated with that Service. You'll be able to contact the `type: NodePort`
Service, from outside the cluster, by connecting to any node using the appropriate
protocol (for example: TCP), and the appropriate port (as assigned to that Service).
-->
对于 NodePort 服务，Kubernetes 额外分配一个端口（TCP、UDP 或 SCTP 以匹配服务的协议）。
集群中的每个节点都将自己配置为监听分配的端口并将流量转发到与该服务关联的某个就绪端点。
通过使用适当的协议（例如 TCP）和适当的端口（分配给该服务）连接到所有节点，
你将能够从集群外部使用 `type: NodePort` 服务。

<!--
#### Choosing your own port {#nodeport-custom-port}

If you want a specific port number, you can specify a value in the `nodePort`
field. The control plane will either allocate you that port or report that
the API transaction failed.
This means that you need to take care of possible port collisions yourself.
You also have to use a valid port number, one that's inside the range configured
for NodePort use.

Here is an example manifest for a Service of `type: NodePort` that specifies
a NodePort value (30007, in this example).
-->
#### 选择你自己的端口   {#nodeport-custom-port}

如果需要特定的端口号，你可以在 `nodePort` 字段中指定一个值。
控制平面将为你分配该端口或报告 API 事务失败。
这意味着你需要自己注意可能发生的端口冲突。
你还必须使用有效的端口号，该端口号在配置用于 NodePort 的范围内。

以下是 `type: NodePort` 服务的一个示例清单，它指定了一个 NodePort 值（在本例中为 30007）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
      # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80
      # 可选字段
      # 默认情况下，为了方便起见，Kubernetes 控制平面会从某个范围内分配一个端口号（默认：30000-32767）
      nodePort: 30007
```

<!--
#### Custom IP address configuration for `type: NodePort` Services {#service-nodeport-custom-listen-address}

You can set up nodes in your cluster to use a particular IP address for serving node port
services. You might want to do this if each node is connected to multiple networks (for example:
one network for application traffic, and another network for traffic between nodes and the
control plane).

If you want to specify particular IP address(es) to proxy the port, you can set the
`--nodeport-addresses` flag for kube-proxy or the equivalent `nodePortAddresses`
field of the
[kube-proxy configuration file](/docs/reference/config-api/kube-proxy-config.v1alpha1/)
to particular IP block(s).
-->
#### 为 `type: NodePort` 服务自定义 IP 地址配置  {#service-nodeport-custom-listen-address}

你可以在集群中设置节点以使用特定 IP 地址来提供 NodePort 服务。
如果每个节点都连接到多个网络（例如：一个网络用于应用程序流量，另一个网络用于节点和控制平面之间的流量），
你可能需要执行此操作。

如果你要指定特定的 IP 地址来代理端口，可以将 kube-proxy 的 `--nodeport-addresses` 标志或
[kube-proxy 配置文件](/zh-cn/docs/reference/config-api/kube-proxy-config.v1alpha1/)的等效
`nodePortAddresses` 字段设置为特定的 IP 段。

<!--
This flag takes a comma-delimited list of IP blocks (e.g. `10.0.0.0/8`, `192.0.2.0/25`)
to specify IP address ranges that kube-proxy should consider as local to this node.

For example, if you start kube-proxy with the `--nodeport-addresses=127.0.0.0/8` flag,
kube-proxy only selects the loopback interface for NodePort Services.
The default for `--nodeport-addresses` is an empty list.
This means that kube-proxy should consider all available network interfaces for NodePort.
(That's also compatible with earlier Kubernetes releases.)
-->
此标志采用逗号分隔的 IP 段列表（例如 `10.0.0.0/8`、`192.0.2.0/25`）来指定 kube-proxy 应视为该节点本地的
IP 地址范围。

例如，如果你使用 `--nodeport-addresses=127.0.0.0/8` 标志启动 kube-proxy，
则 kube-proxy 仅选择 NodePort 服务的环回接口。
`--nodeport-addresses` 的默认值是一个空列表。
这意味着 kube-proxy 应考虑 NodePort 的所有可用网络接口。
（这也与早期的 Kubernetes 版本兼容。）

{{< note >}}
<!-- 
This Service is visible as `<NodeIP>:spec.ports[*].nodePort` and `.spec.clusterIP:spec.ports[*].port`.
If the `--nodeport-addresses` flag for kube-proxy or the equivalent field
in the kube-proxy configuration file is set, `<NodeIP>` would be a filtered node IP address (or possibly IP addresses).
-->
此服务呈现为 `<NodeIP>:spec.ports[*].nodePort` 和 `.spec.clusterIP:spec.ports[*].port`。
如果设置了 kube-proxy 的 `--nodeport-addresses` 标志或 kube-proxy 配置文件中的等效字段，
则 `<NodeIP>` 将是过滤的节点 IP 地址（或可能的 IP 地址）。
{{< /note >}}

<!--
### Type LoadBalancer {#loadbalancer}

On cloud providers which support external load balancers, setting the `type`
field to `LoadBalancer` provisions a load balancer for your Service.
The actual creation of the load balancer happens asynchronously, and
information about the provisioned balancer is published in the Service's
`.status.loadBalancer` field.
For example:
-->
### LoadBalancer 类型  {#loadbalancer}

在使用支持外部负载均衡器的云提供商的服务时，设置 `type` 的值为 `"LoadBalancer"`，
将为 Service 提供负载均衡器。
负载均衡器是异步创建的，关于被提供的负载均衡器的信息将会通过 Service 的
`status.loadBalancer` 字段发布出去。

实例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

<!--
Traffic from the external load balancer is directed at the backend Pods.
The cloud provider decides how it is load balanced.
-->
来自外部负载均衡器的流量将直接重定向到后端 Pod 上，不过实际它们是如何工作的，这要依赖于云提供商。

<!--
Some cloud providers allow you to specify the `loadBalancerIP`. In those cases, the load-balancer is created
with the user-specified `loadBalancerIP`. If the `loadBalancerIP` field is not specified,
the loadBalancer is set up with an ephemeral IP address. If you specify a `loadBalancerIP`
but your cloud provider does not support the feature, the `loadbalancerIP` field that you
set is ignored.
-->
某些云提供商允许设置 `loadBalancerIP`。
在这些情况下，将根据用户设置的 `loadBalancerIP` 来创建负载均衡器。
如果没有设置 `loadBalancerIP` 字段，将会给负载均衡器指派一个临时 IP。
如果设置了 `loadBalancerIP`，但云提供商并不支持这种特性，那么设置的
`loadBalancerIP` 值将会被忽略掉。

<!--
To implement a Service of `type: LoadBalancer`, Kubernetes typically starts off
by making the changes that are equivalent to you requesting a Service of
`type: NodePort`. The cloud-controller-manager component then configures the external load balancer to
forward traffic to that assigned node port.

_As an alpha feature_, you can configure a load balanced Service to
[omit](#load-balancer-nodeport-allocation) assigning a node port, provided that the
cloud provider implementation supports this.
-->
要实现 `type: LoadBalancer` 的服务，Kubernetes 通常首先进行与请求 `type: NodePort` 服务等效的更改。
cloud-controller-manager 组件然后配置外部负载均衡器以将流量转发到已分配的节点端口。

**作为 Alpha 特性**，你可以将负载均衡服务配置为[忽略](#load-balancer-nodeport-allocation)分配节点端口，
前提是云提供商实现支持这点。

{{< note >}}
<!--
On **Azure**, if you want to use a user-specified public type `loadBalancerIP`, you first need
to create a static type public IP address resource. This public IP address resource should
be in the same resource group of the other automatically created resources of the cluster.
For example, `MC_myResourceGroup_myAKSCluster_eastus`.

Specify the assigned IP address as loadBalancerIP. Ensure that you have updated the
`securityGroupName` in the cloud provider configuration file.
For information about troubleshooting `CreatingLoadBalancerFailed` permission issues see,
[Use a static IP address with the Azure Kubernetes Service (AKS) load balancer](https://docs.microsoft.com/en-us/azure/aks/static-ip)
or [CreatingLoadBalancerFailed on AKS cluster with advanced networking](https://github.com/Azure/AKS/issues/357).
-->
在 **Azure** 上，如果要使用用户指定的公共类型 `loadBalancerIP`，
则首先需要创建静态类型的公共 IP 地址资源。
此公共 IP 地址资源应与集群中其他自动创建的资源位于同一资源组中。
例如，`MC_myResourceGroup_myAKSCluster_eastus`。

将分配的 IP 地址设置为 loadBalancerIP。确保你已更新云提供程序配置文件中的 securityGroupName。
有关对 `CreatingLoadBalancerFailed` 权限问题进行故障排除的信息，
请参阅[与 Azure Kubernetes 服务（AKS）负载均衡器一起使用静态 IP 地址](https://docs.microsoft.com/zh-cn/azure/aks/static-ip)
或[在 AKS 集群上使用高级联网时出现 CreatingLoadBalancerFailed](https://github.com/Azure/AKS/issues/357)。
{{< /note >}}
<!--
#### Load balancers with mixed protocol types

{{< feature-state for_k8s_version="v1.24" state="beta" >}}

By default, for LoadBalancer type of Services, when there is more than one port defined, all
ports must have the same protocol, and the protocol must be one which is supported
by the cloud provider.

The feature gate `MixedProtocolLBService` (enabled by default for the kube-apiserver as of v1.24) allows the use of
different protocols for LoadBalancer type of Services, when there is more than one port defined.
-->
#### 混合协议类型的负载均衡器

{{< feature-state for_k8s_version="v1.20" state="alpha" >}}

默认情况下，对于 LoadBalancer 类型的服务，当定义了多个端口时，
所有端口必须具有相同的协议，并且该协议必须是受云提供商支持的协议。

当服务中定义了多个端口时，特性门控 `MixedProtocolLBService`（在 kube-apiserver 1.24 版本默认为启用）允许
LoadBalancer 类型的服务使用不同的协议。

{{< note >}}
<!--
The set of protocols that can be used for LoadBalancer type of Services is still defined by the cloud provider. If a
cloud provider does not support mixed protocols they will provide only a single protocol.
-->
可用于 LoadBalancer 类型服务的协议集仍然由云提供商决定。
如果云提供商不支持混合协议，他们将只提供单一协议。
{{< /note >}}

<!--
#### Disabling load balancer NodePort allocation {#load-balancer-nodeport-allocation}
-->
### 禁用负载均衡器节点端口分配 {#load-balancer-nodeport-allocation}

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

<!--
Starting in v1.20, you can optionally disable node port allocation for a Service Type=LoadBalancer by setting
the field `spec.allocateLoadBalancerNodePorts` to `false`. This should only be used for load balancer implementations
that route traffic directly to pods as opposed to using node ports. By default, `spec.allocateLoadBalancerNodePorts`
is `true` and type LoadBalancer Services will continue to allocate node ports. If `spec.allocateLoadBalancerNodePorts`
is set to `false` on an existing Service with allocated node ports, those node ports will **not** be de-allocated automatically.
You must explicitly remove the `nodePorts` entry in every Service port to de-allocate those node ports.
-->
你可以通过设置 `spec.allocateLoadBalancerNodePorts` 为 `false`
对类型为 LoadBalancer 的服务禁用节点端口分配。
这仅适用于直接将流量路由到 Pod 而不是使用节点端口的负载均衡器实现。
默认情况下，`spec.allocateLoadBalancerNodePorts` 为 `true`，
LoadBalancer 类型的服务继续分配节点端口。
如果现有服务已被分配节点端口，将参数 `spec.allocateLoadBalancerNodePorts`
设置为 `false` 时，这些服务上已分配置的节点端口**不会**被自动释放。
你必须显式地在每个服务端口中删除 `nodePorts` 项以释放对应端口。

<!--
#### Specifying class of load balancer implementation {#load-balancer-class}
-->
#### 设置负载均衡器实现的类别 {#load-balancer-class}

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

<!--
`spec.loadBalancerClass` enables you to use a load balancer implementation other than the cloud provider default.
By default, `spec.loadBalancerClass` is `nil` and a `LoadBalancer` type of Service uses
the cloud provider's default load balancer implementation if the cluster is configured with
a cloud provider using the `--cloud-provider` component flag.
If `spec.loadBalancerClass` is specified, it is assumed that a load balancer
implementation that matches the specified class is watching for Services.
Any default load balancer implementation (for example, the one provided by
the cloud provider) will ignore Services that have this field set.
`spec.loadBalancerClass` can be set on a Service of type `LoadBalancer` only.
Once set, it cannot be changed.
-->
`spec.loadBalancerClass` 允许你不使用云提供商的默认负载均衡器实现，转而使用指定的负载均衡器实现。
默认情况下，`.spec.loadBalancerClass` 的取值是 `nil`，如果集群使用 `--cloud-provider` 配置了云提供商，
`LoadBalancer` 类型服务会使用云提供商的默认负载均衡器实现。
如果设置了 `.spec.loadBalancerClass`，则假定存在某个与所指定的类相匹配的负载均衡器实现在监视服务变化。
所有默认的负载均衡器实现（例如，由云提供商所提供的）都会忽略设置了此字段的服务。`.spec.loadBalancerClass`
只能设置到类型为 `LoadBalancer` 的 Service 之上，而且一旦设置之后不可变更。

<!--
The value of `spec.loadBalancerClass` must be a label-style identifier,
with an optional prefix such as "`internal-vip`" or "`example.com/internal-vip`".
Unprefixed names are reserved for end-users.
-->
`.spec.loadBalancerClass` 的值必须是一个标签风格的标识符，
可以有选择地带有类似 "`internal-vip`" 或 "`example.com/internal-vip`" 这类前缀。
没有前缀的名字是保留给最终用户的。

<!--
#### Internal load balancer

In a mixed environment it is sometimes necessary to route traffic from Services inside the same
(virtual) network address block.

In a split-horizon DNS environment you would need two Services to be able to route both external
and internal traffic to your endpoints.

To set an internal load balancer, add one of the following annotations to your Service
depending on the cloud Service provider you're using.
-->
#### 内部负载均衡器 {#internal-load-balancer}

在混合环境中，有时有必要在同一(虚拟)网络地址块内路由来自服务的流量。

在水平分割 DNS 环境中，你需要两个服务才能将内部和外部流量都路由到你的端点（Endpoints）。

如要设置内部负载均衡器，请根据你所使用的云运营商，为服务添加以下注解之一。

{{< tabs name="service_tabs" >}}
{{% tab name="Default" %}}
<!--
Select one of the tabs.
-->
选择一个标签。
{{% /tab %}}
{{% tab name="GCP" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        cloud.google.com/load-balancer-type: "Internal"
[...]
```

{{% /tab %}}
{{% tab name="AWS" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-internal: "true"
[...]
```

{{% /tab %}}
{{% tab name="Azure" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/azure-load-balancer-internal: "true"
[...]
```

{{% /tab %}}
{{% tab name="IBM Cloud" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: "private"
[...]
```

{{% /tab %}}
{{% tab name="OpenStack" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/openstack-internal-load-balancer: "true"
[...]
```

{{% /tab %}}
{{% tab name="Baidu Cloud" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/cce-load-balancer-internal-vpc: "true"
[...]
```

{{% /tab %}}
{{% tab name="Tencent Cloud" %}}

```yaml
[...]
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxx
[...]
```

{{% /tab %}}
{{% tab name="Alibaba Cloud" %}}

```yaml
[...]
metadata:
  annotations:
    service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
[...]
```

{{% /tab %}}
{{% tab name="OCI" %}}

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/oci-load-balancer-internal: true
[...]
```
{{% /tab %}}
{{< /tabs >}}

<!--
#### TLS support on AWS {#ssl-support-on-aws}

For partial TLS / SSL support on clusters running on AWS, you can add three
annotations to a `LoadBalancer` service:
-->
### AWS TLS 支持 {#ssl-support-on-aws}

为了对在 AWS 上运行的集群提供 TLS/SSL 部分支持，你可以向 `LoadBalancer`
服务添加三个注解：

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

<!--
The first specifies the ARN of the certificate to use. It can be either a
certificate from a third party issuer that was uploaded to IAM or one created
within AWS Certificate Manager.
-->
第一个指定要使用的证书的 ARN。 它可以是已上载到 IAM 的第三方颁发者的证书，
也可以是在 AWS Certificate Manager 中创建的证书。

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
```

<!--
The second annotation specifies which protocol a Pod speaks. For HTTPS and
SSL, the ELB expects the Pod to authenticate itself over the encrypted
connection, using a certificate.

HTTP and HTTPS selects layer 7 proxying: the ELB terminates
the connection with the user, parses headers, and injects the `X-Forwarded-For`
header with the user's IP address (Pods only see the IP address of the
ELB at the other end of its connection) when forwarding requests.

TCP and SSL selects layer 4 proxying: the ELB forwards traffic without
modifying the headers.

In a mixed-use environment where some ports are secured and others are left unencrypted,
you can use the following annotations:
-->
第二个注解指定 Pod 使用哪种协议。对于 HTTPS 和 SSL，ELB 希望 Pod
使用证书通过加密连接对自己进行身份验证。

HTTP 和 HTTPS 选择第7层代理：ELB 终止与用户的连接，解析标头，并在转发请求时向
`X-Forwarded-For` 标头注入用户的 IP 地址（Pod 仅在连接的另一端看到 ELB 的 IP 地址）。

TCP 和 SSL 选择第4层代理：ELB 转发流量而不修改报头。

在某些端口处于安全状态而其他端口未加密的混合使用环境中，可以使用以下注解：

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
```

<!--
In the above example, if the Service contained three ports, `80`, `443`, and
`8443`, then `443` and `8443` would use the SSL certificate, but `80` would be proxied HTTP.

From Kubernetes v1.9 onwards you can use
[predefined AWS SSL policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html)
with HTTPS or SSL listeners for your Services.
To see which policies are available for use, you can use the `aws` command line tool:
-->
在上例中，如果服务包含 `80`、`443` 和 `8443` 三个端口， 那么 `443` 和 `8443` 将使用 SSL 证书，
而 `80` 端口将转发 HTTP 数据包。

从 Kubernetes v1.9 起可以使用
[预定义的 AWS SSL 策略](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/classic/elb-security-policy-table.html)
为你的服务使用 HTTPS 或 SSL 侦听器。
要查看可以使用哪些策略，可以使用 `aws` 命令行工具：

```bash
aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
```

<!--
You can then specify any one of those policies using the
"`service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy`"
annotation; for example:
-->
然后，你可以使用 "`service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy`"
注解; 例如：

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
```

<!--
#### PROXY protocol support on AWS

To enable [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)
support for clusters running on AWS, you can use the following service
annotation:
-->
#### AWS 上的 PROXY 协议支持

为了支持在 AWS 上运行的集群，启用
[PROXY 协议](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)。
你可以使用以下服务注解：

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

<!--
Since version 1.3.0, the use of this annotation applies to all ports proxied by the ELB
and cannot be configured otherwise.
-->
从 1.3.0 版开始，此注解的使用适用于 ELB 代理的所有端口，并且不能进行其他配置。
<!--
#### ELB Access Logs on AWS

There are several annotations to manage access logs for ELB Services on AWS.

The annotation `service.beta.kubernetes.io/aws-load-balancer-access-log-enabled`
controls whether access logs are enabled.

The annotation `service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval`
controls the interval in minutes for publishing the access logs. You can specify
an interval of either 5 or 60 minutes.

The annotation `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name`
controls the name of the Amazon S3 bucket where load balancer access logs are
stored.

The annotation `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix`
specifies the logical hierarchy you created for your Amazon S3 bucket.
-->
#### AWS 上的 ELB 访问日志

有几个注解可用于管理 AWS 上 ELB 服务的访问日志。

注解 `service.beta.kubernetes.io/aws-load-balancer-access-log-enabled` 控制是否启用访问日志。

注解 `service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval`
控制发布访问日志的时间间隔（以分钟为单位）。你可以指定 5 分钟或 60 分钟的间隔。

注解 `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name`
控制存储负载均衡器访问日志的 Amazon S3 存储桶的名称。

注解 `service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix`
指定为 Amazon S3 存储桶创建的逻辑层次结构。

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
        # 指定是否为负载均衡器启用访问日志
        service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"
        # 发布访问日志的时间间隔。你可以将其设置为 5 分钟或 60 分钟。
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"
        # 用来存放访问日志的 Amazon S3 Bucket 名称
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"
        # 你为 Amazon S3 Bucket 所创建的逻辑层次结构，例如 `my-bucket-prefix/prod`
```

<!--
#### Connection Draining on AWS

Connection draining for Classic ELBs can be managed with the annotation
`service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled` set
to the value of `"true"`. The annotation
`service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout` can
also be used to set maximum time, in seconds, to keep the existing connections open before
deregistering the instances.
-->
#### AWS 上的连接排空

可以将注解 `service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled`
设置为 `"true"` 来管理 ELB 的连接排空。
注解 `service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout`
也可以用于设置最大时间（以秒为单位），以保持现有连接在注销实例之前保持打开状态。

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"
```

<!--
#### Other ELB annotations

There are other annotations to manage Classic Elastic Load Balancers that are described below.
-->
#### 其他 ELB 注解

还有其他一些注解，用于管理经典弹性负载均衡器，如下所述。

```yaml
    metadata:
      name: my-service
      annotations:
        # 按秒计的时间，表示负载均衡器关闭连接之前连接可以保持空闲
        # （连接上无数据传输）的时间长度
        service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"

        # 指定该负载均衡器上是否启用跨区的负载均衡能力
        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

        # 逗号分隔列表值，每一项都是一个键-值耦对，会作为额外的标签记录于 ELB 中
        service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "environment=prod,owner=devops"

        # 将某后端视为健康、可接收请求之前需要达到的连续成功健康检查次数。
        # 默认为 2，必须介于 2 和 10 之间
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: ""

        # 将某后端视为不健康、不可接收请求之前需要达到的连续不成功健康检查次数。
        # 默认为 6，必须介于 2 和 10 之间
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"

        # 对每个实例进行健康检查时，连续两次检查之间的大致间隔秒数
        # 默认为 10，必须介于 5 和 300 之间
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"

        # 时长秒数，在此期间没有响应意味着健康检查失败
        # 此值必须小于 service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval
        # 默认值为 5，必须介于 2 和 60 之间
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"

        # 由已有的安全组所构成的列表，可以配置到所创建的 ELB 之上。
        # 与注解 service.beta.kubernetes.io/aws-load-balancer-extra-security-groups 不同，
        # 这一设置会替代掉之前指定给该 ELB 的所有其他安全组，也会覆盖掉为此
        # ELB 所唯一创建的安全组。 
        # 此列表中的第一个安全组 ID 被用来作为决策源，以允许入站流量流入目标工作节点
        # (包括服务流量和健康检查）。
        # 如果多个 ELB 配置了相同的安全组 ID，为工作节点安全组添加的允许规则行只有一个，
        # 这意味着如果你删除了这些 ELB 中的任何一个，都会导致该规则记录被删除，
        # 以至于所有共享该安全组 ID 的其他 ELB 都无法访问该节点。
        # 此注解如果使用不当，会导致跨服务的不可用状况。
        service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-53fae93f"

        # 额外的安全组列表，将被添加到所创建的 ELB 之上。
        # 添加时，会保留为 ELB 所专门创建的安全组。
        # 这样会确保每个 ELB 都有一个唯一的安全组 ID 和与之对应的允许规则记录，
        # 允许请求（服务流量和健康检查）发送到目标工作节点。
        # 这里顶一个安全组可以被多个服务共享。
        service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f,sg-42efd82e"

        # 用逗号分隔的一个键-值偶对列表，用来为负载均衡器选择目标节点
        service.beta.kubernetes.io/aws-load-balancer-target-node-labels: "ingress-gw,gw-name=public-api"
```

<!--
#### Network Load Balancer support on AWS {#aws-nlb-support}
-->
#### AWS 上网络负载均衡器支持 {#aws-nlb-support}

{{< feature-state for_k8s_version="v1.15" state="beta" >}}

<!--
To use a Network Load Balancer on AWS, use the annotation `service.beta.kubernetes.io/aws-load-balancer-type` with the value set to `nlb`.
-->
要在 AWS 上使用网络负载均衡器，可以使用注解
`service.beta.kubernetes.io/aws-load-balancer-type`，将其取值设为 `nlb`。

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

{{< note >}}
<!--
NLB only works with certain instance classes; see the
[AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-register-targets.html#register-deregister-targets)
on Elastic Load Balancing for a list of supported instance types.
-->
NLB 仅适用于某些实例类。有关受支持的实例类型的列表，
请参见
[AWS 文档](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/network/target-group-register-targets.html#register-deregister-targets)
中关于所支持的实例类型的 Elastic Load Balancing 说明。
{{< /note >}}

<!--
Unlike Classic Elastic Load Balancers, Network Load Balancers (NLBs) forward the
client's IP address through to the node. If a Service's `.spec.externalTrafficPolicy`
is set to `Cluster`, the client's IP address is not propagated to the end
Pods.

By setting `.spec.externalTrafficPolicy` to `Local`, the client IP addresses is
propagated to the end Pods, but this could result in uneven distribution of
traffic. Nodes without any Pods for a particular LoadBalancer Service will fail
the NLB Target Group's health check on the auto-assigned
`.spec.healthCheckNodePort` and not receive any traffic.
-->
与经典弹性负载均衡器不同，网络负载均衡器（NLB）将客户端的 IP 地址转发到该节点。
如果服务的 `.spec.externalTrafficPolicy` 设置为 `Cluster` ，则客户端的 IP 地址不会传达到最终的 Pod。

通过将 `.spec.externalTrafficPolicy` 设置为 `Local`，客户端 IP 地址将传播到最终的 Pod，
但这可能导致流量分配不均。
没有针对特定 LoadBalancer 服务的任何 Pod 的节点将无法通过自动分配的
`.spec.healthCheckNodePort` 进行 NLB 目标组的运行状况检查，并且不会收到任何流量。

<!--
In order to achieve even traffic, either use a DaemonSet, or specify a
[pod anti-affinity](/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
to not locate on the same node.

You can also use NLB Services with the [internal load balancer](/docs/concepts/services-networking/service/#internal-load-balancer)
annotation.

In order for client traffic to reach instances behind an NLB, the Node security
groups are modified with the following IP rules:
-->
为了获得均衡流量，请使用 DaemonSet 或指定
[Pod 反亲和性](/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
使其不在同一节点上。

你还可以将 NLB 服务与[内部负载均衡器](/zh-cn/docs/concepts/services-networking/service/#internal-load-balancer)
注解一起使用。

为了使客户端流量能够到达 NLB 后面的实例，使用以下 IP 规则修改了节点安全组：

| Rule | Protocol | Port(s) | IpRange(s) | IpRange Description |
|------|----------|---------|------------|---------------------|
| Health Check | TCP | NodePort(s) (`.spec.healthCheckNodePort` for `.spec.externalTrafficPolicy = Local`) | Subnet CIDR | kubernetes.io/rule/nlb/health=\<loadBalancerName\> |
| Client Traffic | TCP | NodePort(s) | `.spec.loadBalancerSourceRanges` (默认值为 `0.0.0.0/0`) | kubernetes.io/rule/nlb/client=\<loadBalancerName\> |
| MTU Discovery | ICMP | 3,4 | `.spec.loadBalancerSourceRanges` (默认值为 `0.0.0.0/0`) | kubernetes.io/rule/nlb/mtu=\<loadBalancerName\> |

<!--
In order to limit which client IP's can access the Network Load Balancer,
specify `loadBalancerSourceRanges`.
-->
为了限制哪些客户端 IP 可以访问网络负载均衡器，请指定 `loadBalancerSourceRanges`。

```yaml
spec:
  loadBalancerSourceRanges:
    - "143.231.0.0/16"
```

{{< note >}}
<!--
If `.spec.loadBalancerSourceRanges` is not set, Kubernetes
allows traffic from `0.0.0.0/0` to the Node Security Group(s). If nodes have
public IP addresses, be aware that non-NLB traffic can also reach all instances
in those modified security groups.
-->
如果未设置 `.spec.loadBalancerSourceRanges` ，则 Kubernetes 允许从 `0.0.0.0/0` 到节点安全组的流量。
如果节点具有公共 IP 地址，请注意，非 NLB 流量也可以到达那些修改后的安全组中的所有实例。
{{< /note >}}

<!--
Further documentation on annotations for Elastic IPs and other common use-cases may be found
in the [AWS Load Balancer Controller documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/).
-->
有关弹性 IP 注解和更多其他常见用例，
请参阅[AWS 负载均衡控制器文档](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/)。

<!--
### Type ExternalName {#externalname}

Services of type ExternalName map a Service to a DNS name, not to a typical selector such as
`my-service` or `cassandra`. You specify these Services with the `spec.externalName` parameter.

This Service definition, for example, maps
the `my-service` Service in the `prod` namespace to `my.database.example.com`:
-->

### ExternalName 类型         {#externalname}

类型为 ExternalName 的服务将服务映射到 DNS 名称，而不是典型的选择算符，例如 `my-service` 或者 `cassandra`。
你可以使用 `spec.externalName` 参数指定这些服务。

例如，以下 Service 定义将 `prod` 名称空间中的 `my-service` 服务映射到 `my.database.example.com`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

{{< note >}}
<!--
ExternalName accepts an IPv4 address string, but as a DNS name comprised of digits, not as an IP address.
ExternalNames that resemble IPv4 addresses are not resolved by CoreDNS or ingress-nginx because ExternalName
is intended to specify a canonical DNS name. To hardcode an IP address, consider using
[headless Services](#headless-services).
-->
ExternalName 服务接受 IPv4 地址字符串，但作为包含数字的 DNS 名称，而不是 IP 地址。
类似于 IPv4 地址的外部名称不能由 CoreDNS 或 ingress-nginx 解析，因为外部名称旨在指定规范的 DNS 名称。
要对 IP 地址进行硬编码，请考虑使用[无头 Services](#headless-services)。
{{< /note >}}

<!--
When looking up the host `my-service.prod.svc.cluster.local`, the cluster DNS Service
returns a `CNAME` record with the value `my.database.example.com`. Accessing
`my-service` works in the same way as other Services but with the crucial
difference that redirection happens at the DNS level rather than via proxying or
forwarding. Should you later decide to move your database into your cluster, you
can start its Pods, add appropriate selectors or endpoints, and change the
Service's `type`.
-->
当查找主机 `my-service.prod.svc.cluster.local` 时，集群 DNS 服务返回 `CNAME` 记录，
其值为 `my.database.example.com`。
访问 `my-service` 的方式与其他服务的方式相同，但主要区别在于重定向发生在 DNS 级别，而不是通过代理或转发。
如果以后你决定将数据库移到集群中，则可以启动其 Pod，添加适当的选择算符或端点以及更改服务的 `type`。

{{< warning >}}
<!--
You may have trouble using ExternalName for some common protocols, including HTTP and HTTPS.
If you use ExternalName then the hostname used by clients inside your cluster is different from
the name that the ExternalName references.

For protocols that use hostnames this difference may lead to errors or unexpected responses.
HTTP requests will have a `Host:` header that the origin server does not recognize;
TLS servers will not be able to provide a certificate matching the hostname that the client connected to.
-->
对于一些常见的协议，包括 HTTP 和 HTTPS，你使用 ExternalName 可能会遇到问题。
如果你使用 ExternalName，那么集群内客户端使用的主机名与 ExternalName 引用的名称不同。

对于使用主机名的协议，此差异可能会导致错误或意外响应。
HTTP 请求将具有源服务器无法识别的 `Host:` 标头；
TLS 服务器将无法提供与客户端连接的主机名匹配的证书。
{{< /warning >}}

{{< note >}}
<!--
This section is indebted to the [Kubernetes Tips - Part
1](https://akomljen.com/kubernetes-tips-part-1/) blog post from [Alen Komljen](https://akomljen.com/).
-->
有关这部分内容，我们要感谢 [Alen Komljen](https://akomljen.com/) 刊登的
[Kubernetes Tips - Part1](https://akomljen.com/kubernetes-tips-part-1/) 这篇博文。
{{< /note >}}

<!--
### External IPs

If there are external IPs that route to one or more cluster nodes, Kubernetes Services can be exposed on those
`externalIPs`. Traffic that ingresses into the cluster with the external IP (as destination IP), on the Service port,
will be routed to one of the Service endpoints. `externalIPs` are not managed by Kubernetes and are the responsibility
of the cluster administrator.

In the Service spec, `externalIPs` can be specified along with any of the `ServiceTypes`.
In the example below, "`my-service`" can be accessed by clients on "`80.11.12.10:80`" (`externalIP:port`)
-->
### 外部 IP  {#external-ips}

如果外部的 IP 路由到集群中一个或多个 Node 上，Kubernetes Service 会被暴露给这些 `externalIPs`。
通过外部 IP（作为目的 IP 地址）进入到集群，打到 Service 的端口上的流量，
将会被路由到 Service 的 Endpoint 上。
`externalIPs` 不会被 Kubernetes 管理，它属于集群管理员的职责范畴。

根据 Service 的规定，`externalIPs` 可以同任意的 `ServiceType` 来一起指定。
在上面的例子中，`my-service` 可以在  "`80.11.12.10:80`"(`externalIP:port`) 上被客户端访问。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

<!--
## Session stickiness

If you want to make sure that connections from a particular client are passed to
the same Pod each time, you can configure session affinity based on the client's
IP address. Read [session affinity](/docs/reference/networking/virtual-ips/#session-affinity)
to learn more.
-->
## 粘性会话   {#session-stickiness}

如果你想确保来自特定客户端的连接每次都传递到同一个 Pod，你可以配置根据客户端 IP 地址来执行的会话亲和性。
阅读[会话亲和性](/zh-cn/docs/reference/networking/virtual-ips/#session-affinity)了解更多。

<!--
## API Object

Service is a top-level resource in the Kubernetes REST API. You can find more details
about the [Service API object](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#service-v1-core).

-->
## API 对象   {#api-object}

Service 是 Kubernetes REST API 中的顶级资源。你可以找到有关
[Service 对象 API](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#service-v1-core)
的更多详细信息。

<!-- preserve existing hyperlinks -->
<a id="shortcomings" /><a id="#the-gory-details-of-virtual-ips" />

<!--
## Virtual IP addressing mechanism

Read [Virtual IPs and Service Proxies](/docs/reference/networking/virtual-ips/) to learn about the
mechanism Kubernetes provides to expose a Service with a virtual IP address.
-->
## 虚拟 IP 寻址机制   {#virtual-ip-addressing-mechanism}

阅读[虚拟 IP 和 Service 代理](/zh-cn/docs/reference/networking/virtual-ips/)以了解
Kubernetes 提供的使用虚拟 IP 地址公开服务的机制。

## {{% heading "whatsnext" %}}

<!--
* Follow the [Connecting Applications with Services](/docs/tutorials/services/connect-applications-service/) tutorial
* Read about [Ingress](/docs/concepts/services-networking/ingress/)
* Read about [EndpointSlices](/docs/concepts/services-networking/endpoint-slices/)

For more context:
* Read [Virtual IPs and Service Proxies](/docs/reference/networking/virtual-ips/)
* Read the [API reference](/docs/reference/kubernetes-api/service-resources/service-v1/) for the Service API
* Read the [API reference](/docs/reference/kubernetes-api/service-resources/endpoints-v1/) for the Endpoints API
* Read the [API reference](/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/) for the EndpointSlice API
-->
* 遵循[使用 Service 连接到应用](/zh-cn/docs/tutorials/services/connect-applications-service/)教程
* 阅读了解 [Ingress](/zh-cn/docs/concepts/services-networking/ingress/)
* 阅读了解[端点切片（Endpoint Slices）](/zh-cn/docs/concepts/services-networking/endpoint-slices/)

更多上下文：
* 阅读[虚拟 IP 和 Service 代理](/zh-cn/docs/reference/networking/virtual-ips/)
* 阅读 Service API 的 [API 参考](/zh-cn/docs/reference/kubernetes-api/service-resources/service-v1/)
* 阅读 EndpointSlice API 的 [API 参考](/zh-cn/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/)
