---
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
---


<!-- overview -->

{{< glossary_definition term_id="service" length="short" >}}

With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism.
Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods,
and can load-balance across them.

<!-- body -->

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

## Service resources {#service-resource}

In Kubernetes, a Service is an abstraction which defines a logical set of Pods
and a policy by which to access them (sometimes this pattern is called
a micro-service). The set of Pods targeted by a Service is usually determined
by a {{< glossary_tooltip text="selector" term_id="selector" >}}.
To learn about other ways to define Service endpoints,
see [Services _without_ selectors](#services-without-selectors).

For example, consider a stateless image-processing backend which is running with
3 replicas.  Those replicas are fungible&mdash;frontends do not care which backend
they use.  While the actual Pods that compose the backend set may change, the
frontend clients should not need to be aware of that, nor should they need to keep
track of the set of backends themselves.

The Service abstraction enables this decoupling.

### Cloud-native service discovery

If you're able to use Kubernetes APIs for service discovery in your application,
you can query the {{< glossary_tooltip text="API server" term_id="kube-apiserver" >}}
for matching EndpointSlices. Kubernetes updates the EndpointSlices for a Service
whenever the set of Pods in a Service changes.

For non-native applications, Kubernetes offers ways to place a network port or load
balancer in between your application and the backend Pods.

## Defining a Service

A Service in Kubernetes is a REST object, similar to a Pod.  Like all of the
REST objects, you can `POST` a Service definition to the API server to create
a new instance.
The name of a Service object must be a valid
[RFC 1035 label name](/docs/concepts/overview/working-with-objects/names#rfc-1035-label-names).

For example, suppose you have a set of Pods where each listens on TCP port 9376
and contains a label `app.kubernetes.io/name=MyApp`:

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

This specification creates a new Service object named "my-service", which
targets TCP port 9376 on any Pod with the `app.kubernetes.io/name=MyApp` label.

Kubernetes assigns this Service an IP address (sometimes called the "cluster IP"),
which is used by the Service proxies
(see [Virtual IP addressing mechanism](#virtual-ip-addressing-mechanism) below).

The controller for the Service selector continuously scans for Pods that
match its selector, and then POSTs any updates to an Endpoint object
also named "my-service".

{{< note >}}
A Service can map _any_ incoming `port` to a `targetPort`. By default and
for convenience, the `targetPort` is set to the same value as the `port`
field.
{{< /note >}}

Port definitions in Pods have names, and you can reference these names in the
`targetPort` attribute of a Service. For example, we can bind the `targetPort`
of the Service to the Pod port in the following way:

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


This works even if there is a mixture of Pods in the Service using a single
configured name, with the same network protocol available via different
port numbers. This offers a lot of flexibility for deploying and evolving
your Services. For example, you can change the port numbers that Pods expose
in the next version of your backend software, without breaking clients.

The default protocol for Services is
[TCP](/docs/reference/networking/service-protocols/#protocol-tcp); you can also
use any other [supported protocol](/docs/reference/networking/service-protocols/).

As many Services need to expose more than one port, Kubernetes supports multiple
port definitions on a Service object.
Each port definition can have the same `protocol`, or a different one.

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

Because this Service has no selector, the corresponding EndpointSlice (and
legacy Endpoints) objects are not created automatically. You can map the Service
to the network address and port where it's running, by adding an EndpointSlice
object manually. For example:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1 # by convention, use the name of the Service
                     # as a prefix for the name of the EndpointSlice
  labels:
    # You should set the "kubernetes.io/service-name" label.
    # Set its value to match the name of the Service
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: '' # empty because port 9376 is not assigned as a well-known
             # port (by IANA)
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6" # the IP addresses in this list can appear in any order
      - "10.1.2.3"
```

#### Custom EndpointSlices

When you create an [EndpointSlice](#endpointslices) object for a Service, you can
use any name for the EndpointSlice. Each EndpointSlice in a namespace must have a
unique name. You link an EndpointSlice to a Service by setting the
`kubernetes.io/service-name` {{< glossary_tooltip text="label" term_id="label" >}}
on that EndpointSlice.

{{< note >}}
The endpoint IPs _must not_ be: loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or
link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).

The endpoint IP addresses cannot be the cluster IPs of other Kubernetes Services,
because {{< glossary_tooltip term_id="kube-proxy" >}} doesn't support virtual IPs
as a destination.
{{< /note >}}

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

#### Accessing a Service without a selector {#service-no-selector-access}

Accessing a Service without a selector works the same as if it had a selector.
In the [example](#services-without-selectors) for a Service without a selector, traffic is routed to one of the two endpoints defined in
the EndpointSlice manifest: a TCP connection to 10.1.2.3 or 10.4.5.6, on port 9376.

{{< note >}}
The Kubernetes API server does not allow proxying to endpoints that are not mapped to
pods. Actions such as `kubectl proxy <service-name>` where the service has no
selector will fail due to this constraint. This prevents the Kubernetes API server
from being used as a proxy to endpoints the caller may not be authorized to access.
{{< /note >}}

An ExternalName Service is a special case of Service that does not have
selectors and uses DNS names instead. For more information, see the
[ExternalName](#externalname) section later in this document.

### EndpointSlices

{{< feature-state for_k8s_version="v1.21" state="stable" >}}

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

### Endpoints

In the Kubernetes API, an
[Endpoints](/docs/reference/kubernetes-api/service-resources/endpoints-v1/)
(the resource kind is plural) defines a list of network endpoints, typically
referenced by a Service to define which Pods the traffic can be sent to.

The EndpointSlice API is the recommended replacement for Endpoints.

#### Over-capacity endpoints

Kubernetes limits the number of endpoints that can fit in a single Endpoints
object. When there are over 1000 backing endpoints for a Service, Kubernetes
truncates the data in the Endpoints object. Because a Service can be linked
with more than one EndpointSlice, the 1000 backing endpoint limit only
affects the legacy Endpoints API.

In that case, Kubernetes selects at most 1000 possible backend endpoints to store
into the Endpoints object, and sets an
{{< glossary_tooltip text="annotation" term_id="annotation" >}} on the
Endpoints:
[`endpoints.kubernetes.io/over-capacity: truncated`](/docs/reference/labels-annotations-taints/#endpoints-kubernetes-io-over-capacity).
The control plane also removes that annotation if the number of backend Pods drops below 1000.

Traffic is still sent to backends, but any load balancing mechanism that relies on the
legacy Endpoints API only sends traffic to at most 1000 of the available backing endpoints.

The same API limit means that you cannot manually update an Endpoints to have more than 1000 endpoints.

### Application protocol

{{< feature-state for_k8s_version="v1.20" state="stable" >}}

The `appProtocol` field provides a way to specify an application protocol for
each Service port. The value of this field is mirrored by the corresponding
Endpoints and EndpointSlice objects.

This field follows standard Kubernetes label syntax. Values should either be
[IANA standard service names](https://www.iana.org/assignments/service-names) or
domain prefixed names such as `mycompany.com/my-custom-protocol`.

## Multi-Port Services

For some Services, you need to expose more than one port.
Kubernetes lets you configure multiple port definitions on a Service object.
When using multiple ports for a Service, you must give all of your ports names
so that these are unambiguous.
For example:

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
As with Kubernetes {{< glossary_tooltip term_id="name" text="names">}} in general, names for ports
must only contain lowercase alphanumeric characters and `-`. Port names must
also start and end with an alphanumeric character.

For example, the names `123-abc` and `web` are valid, but `123_abc` and `-web` are not.
{{< /note >}}

## Choosing your own IP address

You can specify your own cluster IP address as part of a `Service` creation
request.  To do this, set the `.spec.clusterIP` field. For example, if you
already have an existing DNS entry that you wish to reuse, or legacy systems
that are configured for a specific IP address and difficult to re-configure.

The IP address that you choose must be a valid IPv4 or IPv6 address from within the
`service-cluster-ip-range` CIDR range that is configured for the API server.
If you try to create a Service with an invalid clusterIP address value, the API
server will return a 422 HTTP status code to indicate that there's a problem.

## Discovering services

Kubernetes supports 2 primary modes of finding a Service - environment
variables and DNS.

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
When you have a Pod that needs to access a Service, and you are using
the environment variable method to publish the port and cluster IP to the client
Pods, you must create the Service *before* the client Pods come into existence.
Otherwise, those client Pods won't have their environment variables populated.

If you only use DNS to discover the cluster IP for a Service, you don't need to
worry about this ordering issue.
{{< /note >}}

### DNS

You can (and almost always should) set up a DNS service for your Kubernetes
cluster using an [add-on](/docs/concepts/cluster-administration/addons/).

A cluster-aware DNS server, such as CoreDNS, watches the Kubernetes API for new
Services and creates a set of DNS records for each one.  If DNS has been enabled
throughout your cluster then all Pods should automatically be able to resolve
Services by their DNS name.

For example, if you have a Service called `my-service` in a Kubernetes
namespace `my-ns`, the control plane and the DNS Service acting together
create a DNS record for `my-service.my-ns`. Pods in the `my-ns` namespace
should be able to find the service by doing a name lookup for `my-service`
(`my-service.my-ns` would also work).

Pods in other namespaces must qualify the name as `my-service.my-ns`. These names
will resolve to the cluster IP assigned for the Service.

Kubernetes also supports DNS SRV (Service) records for named ports.  If the
`my-service.my-ns` Service has a port named `http` with the protocol set to
`TCP`, you can do a DNS SRV query for `_http._tcp.my-service.my-ns` to discover
the port number for `http`, as well as the IP address.

The Kubernetes DNS server is the only way to access `ExternalName` Services.
You can find more information about `ExternalName` resolution in
[DNS for Services and Pods](/docs/concepts/services-networking/dns-pod-service/).

## Headless Services

Sometimes you don't need load-balancing and a single Service IP.  In
this case, you can create what are termed "headless" Services, by explicitly
specifying `"None"` for the cluster IP (`.spec.clusterIP`).

You can use a headless Service to interface with other service discovery mechanisms,
without being tied to Kubernetes' implementation.

For headless `Services`, a cluster IP is not allocated, kube-proxy does not handle
these Services, and there is no load balancing or proxying done by the platform
for them. How DNS is automatically configured depends on whether the Service has
selectors defined:

### With selectors

For headless Services that define selectors, the Kubernetes control plane creates
EndpointSlice objects in the Kubernetes API, and modifies the DNS configuration to return
A or AAAA records (IPv4 or IPv6 addresses) that point directly to the Pods backing
the Service.

### Without selectors

For headless Services that do not define selectors, the control plane does
not create EndpointSlice objects. However, the DNS system looks for and configures
either:

* DNS CNAME records for [`type: ExternalName`](#externalname) Services.
* DNS A / AAAA records for all IP addresses of the Service's ready endpoints,
  for all Service types other than `ExternalName`.
  * For IPv4 endpoints, the DNS system creates A records.
  * For IPv6 endpoints, the DNS system creates AAAA records.

## Publishing Services (ServiceTypes) {#publishing-services-service-types}

For some parts of your application (for example, frontends) you may want to expose a
Service onto an external IP address, that's outside of your cluster.

Kubernetes `ServiceTypes` allow you to specify what kind of Service you want.

`Type` values and their behaviors are:

* `ClusterIP`: Exposes the Service on a cluster-internal IP. Choosing this value
  makes the Service only reachable from within the cluster. This is the
  default that is used if you don't explicitly specify a `type` for a Service.
  You can expose the service to the public with an [Ingress](/docs/concepts/services-networking/ingress/) or the
  [Gateway API](https://gateway-api.sigs.k8s.io/).
* [`NodePort`](#type-nodeport): Exposes the Service on each Node's IP at a static port
  (the `NodePort`).
  To make the node port available, Kubernetes sets up a cluster IP address,
  the same as if you had requested a Service of `type: ClusterIP`.
* [`LoadBalancer`](#loadbalancer): Exposes the Service externally using a cloud
  provider's load balancer.
* [`ExternalName`](#externalname): Maps the Service to the contents of the
  `externalName` field (e.g. `foo.bar.example.com`), by returning a `CNAME` record
  with its value. No proxying of any kind is set up.
  {{< note >}}
  You need either `kube-dns` version 1.7 or CoreDNS version 0.0.8 or higher
  to use the `ExternalName` type.
  {{< /note >}}

The `type` field was designed as nested functionality - each level adds to the
previous.  This is not strictly required on all cloud providers (for example: Google
Compute Engine does not need to allocate a node port to make `type: LoadBalancer` work,
but another cloud provider integration might do). Although strict nesting is not required,
but the Kubernetes API design for Service requires it anyway.

You can also use [Ingress](/docs/concepts/services-networking/ingress/) to expose your Service.
Ingress is not a Service type, but it acts as the entry point for your cluster.
It lets you consolidate your routing rules into a single resource as it can expose multiple
services under the same IP address.

### Type NodePort {#type-nodeport}

If you set the `type` field to `NodePort`, the Kubernetes control plane
allocates a port from a range specified by `--service-node-port-range` flag (default: 30000-32767).
Each node proxies that port (the same port number on every Node) into your Service.
Your Service reports the allocated port in its `.spec.ports[*].nodePort` field.

Using a NodePort gives you the freedom to set up your own load balancing solution,
to configure environments that are not fully supported by Kubernetes, or even
to expose one or more nodes' IP addresses directly.

For a node port Service, Kubernetes additionally allocates a port (TCP, UDP or
SCTP to match the protocol of the Service). Every node in the cluster configures
itself to listen on that assigned port and to forward traffic to one of the ready
endpoints associated with that Service. You'll be able to contact the `type: NodePort`
Service, from outside the cluster, by connecting to any node using the appropriate
protocol (for example: TCP), and the appropriate port (as assigned to that Service).

#### Choosing your own port {#nodeport-custom-port}

If you want a specific port number, you can specify a value in the `nodePort`
field. The control plane will either allocate you that port or report that
the API transaction failed.
This means that you need to take care of possible port collisions yourself.
You also have to use a valid port number, one that's inside the range configured
for NodePort use.

Here is an example manifest for a Service of `type: NodePort` that specifies
a NodePort value (30007, in this example).

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
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

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

This flag takes a comma-delimited list of IP blocks (e.g. `10.0.0.0/8`, `192.0.2.0/25`)
to specify IP address ranges that kube-proxy should consider as local to this node.

For example, if you start kube-proxy with the `--nodeport-addresses=127.0.0.0/8` flag,
kube-proxy only selects the loopback interface for NodePort Services.
The default for `--nodeport-addresses` is an empty list.
This means that kube-proxy should consider all available network interfaces for NodePort.
(That's also compatible with earlier Kubernetes releases.)
{{< note >}}
This Service is visible as `<NodeIP>:spec.ports[*].nodePort` and `.spec.clusterIP:spec.ports[*].port`.
If the `--nodeport-addresses` flag for kube-proxy or the equivalent field
in the kube-proxy configuration file is set, `<NodeIP>` would be a filtered node IP address (or possibly IP addresses).
{{< /note >}}

### Type LoadBalancer {#loadbalancer}

On cloud providers which support external load balancers, setting the `type`
field to `LoadBalancer` provisions a load balancer for your Service.
The actual creation of the load balancer happens asynchronously, and
information about the provisioned balancer is published in the Service's
`.status.loadBalancer` field.
For example:

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

Traffic from the external load balancer is directed at the backend Pods.
The cloud provider decides how it is load balanced.

Some cloud providers allow you to specify the `loadBalancerIP`. In those cases, the load-balancer is created
with the user-specified `loadBalancerIP`. If the `loadBalancerIP` field is not specified,
the loadBalancer is set up with an ephemeral IP address. If you specify a `loadBalancerIP`
but your cloud provider does not support the feature, the `loadbalancerIP` field that you
set is ignored.

To implement a Service of `type: LoadBalancer`, Kubernetes typically starts off
by making the changes that are equivalent to you requesting a Service of
`type: NodePort`. The cloud-controller-manager component then configures the external load balancer to
forward traffic to that assigned node port.

_As an alpha feature_, you can configure a load balanced Service to
[omit](#load-balancer-nodeport-allocation) assigning a node port, provided that the
cloud provider implementation supports this.


{{< note >}}

On **Azure**, if you want to use a user-specified public type `loadBalancerIP`, you first need
to create a static type public IP address resource. This public IP address resource should
be in the same resource group of the other automatically created resources of the cluster.
For example, `MC_myResourceGroup_myAKSCluster_eastus`.

Specify the assigned IP address as loadBalancerIP. Ensure that you have updated the
`securityGroupName` in the cloud provider configuration file.
For information about troubleshooting `CreatingLoadBalancerFailed` permission issues see,
[Use a static IP address with the Azure Kubernetes Service (AKS) load balancer](https://docs.microsoft.com/en-us/azure/aks/static-ip)
or [CreatingLoadBalancerFailed on AKS cluster with advanced networking](https://github.com/Azure/AKS/issues/357).

{{< /note >}}

#### Load balancers with mixed protocol types

{{< feature-state for_k8s_version="v1.24" state="beta" >}}

By default, for LoadBalancer type of Services, when there is more than one port defined, all
ports must have the same protocol, and the protocol must be one which is supported
by the cloud provider.

The feature gate `MixedProtocolLBService` (enabled by default for the kube-apiserver as of v1.24) allows the use of
different protocols for LoadBalancer type of Services, when there is more than one port defined.

{{< note >}}

The set of protocols that can be used for LoadBalancer type of Services is still defined by the cloud provider. If a
cloud provider does not support mixed protocols they will provide only a single protocol.

{{< /note >}}

#### Disabling load balancer NodePort allocation {#load-balancer-nodeport-allocation}

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

You can optionally disable node port allocation for a Service of `type=LoadBalancer`, by setting
the field `spec.allocateLoadBalancerNodePorts` to `false`. This should only be used for load balancer implementations
that route traffic directly to pods as opposed to using node ports. By default, `spec.allocateLoadBalancerNodePorts`
is `true` and type LoadBalancer Services will continue to allocate node ports. If `spec.allocateLoadBalancerNodePorts`
is set to `false` on an existing Service with allocated node ports, those node ports will **not** be de-allocated automatically.
You must explicitly remove the `nodePorts` entry in every Service port to de-allocate those node ports.

#### Specifying class of load balancer implementation {#load-balancer-class}

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

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
The value of `spec.loadBalancerClass` must be a label-style identifier,
with an optional prefix such as "`internal-vip`" or "`example.com/internal-vip`".
Unprefixed names are reserved for end-users.

#### Internal load balancer

In a mixed environment it is sometimes necessary to route traffic from Services inside the same
(virtual) network address block.

In a split-horizon DNS environment you would need two Services to be able to route both external
and internal traffic to your endpoints.

To set an internal load balancer, add one of the following annotations to your Service
depending on the cloud Service provider you're using.

{{< tabs name="service_tabs" >}}
{{% tab name="Default" %}}
Select one of the tabs.
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

#### TLS support on AWS {#ssl-support-on-aws}

For partial TLS / SSL support on clusters running on AWS, you can add three
annotations to a `LoadBalancer` service:

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

The first specifies the ARN of the certificate to use. It can be either a
certificate from a third party issuer that was uploaded to IAM or one created
within AWS Certificate Manager.

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
```

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

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
```

In the above example, if the Service contained three ports, `80`, `443`, and
`8443`, then `443` and `8443` would use the SSL certificate, but `80` would be proxied HTTP.

From Kubernetes v1.9 onwards you can use
[predefined AWS SSL policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html)
with HTTPS or SSL listeners for your Services.
To see which policies are available for use, you can use the `aws` command line tool:

```bash
aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
```

You can then specify any one of those policies using the
"`service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy`"
annotation; for example:

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
```

#### PROXY protocol support on AWS

To enable [PROXY protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)
support for clusters running on AWS, you can use the following service
annotation:

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

Since version 1.3.0, the use of this annotation applies to all ports proxied by the ELB
and cannot be configured otherwise.

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

```yaml
    metadata:
      name: my-service
      annotations:
        # Specifies whether access logs are enabled for the load balancer
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"

        # The interval for publishing the access logs. You can specify an interval of either 5 or 60 (minutes).
        service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"

        # The name of the Amazon S3 bucket where the access logs are stored
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"

        # The logical hierarchy you created for your Amazon S3 bucket, for example `my-bucket-prefix/prod`
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"
```

#### Connection Draining on AWS

Connection draining for Classic ELBs can be managed with the annotation
`service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled` set
to the value of `"true"`. The annotation
`service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout` can
also be used to set maximum time, in seconds, to keep the existing connections open before
deregistering the instances.

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"
```

#### Other ELB annotations

There are other annotations to manage Classic Elastic Load Balancers that are described below.

```yaml
    metadata:
      name: my-service
      annotations:
        # The time, in seconds, that the connection is allowed to be idle (no data has been sent
        # over the connection) before it is closed by the load balancer
        service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"

        # Specifies whether cross-zone load balancing is enabled for the load balancer
        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

        # A comma-separated list of key-value pairs which will be recorded as
        # additional tags in the ELB.
        service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "environment=prod,owner=devops"

        # The number of successive successful health checks required for a backend to
        # be considered healthy for traffic. Defaults to 2, must be between 2 and 10
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: ""

        # The number of unsuccessful health checks required for a backend to be
        # considered unhealthy for traffic. Defaults to 6, must be between 2 and 10
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"

        # The approximate interval, in seconds, between health checks of an
        # individual instance. Defaults to 10, must be between 5 and 300
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"

        # The amount of time, in seconds, during which no response means a failed
        # health check. This value must be less than the service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval
        # value. Defaults to 5, must be between 2 and 60
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"

        # A list of existing security groups to be configured on the ELB created. Unlike the annotation
        # service.beta.kubernetes.io/aws-load-balancer-extra-security-groups, this replaces all other
        # security groups previously assigned to the ELB and also overrides the creation
        # of a uniquely generated security group for this ELB.
        # The first security group ID on this list is used as a source to permit incoming traffic to
        # target worker nodes (service traffic and health checks).
        # If multiple ELBs are configured with the same security group ID, only a single permit line
        # will be added to the worker node security groups, that means if you delete any
        # of those ELBs it will remove the single permit line and block access for all ELBs that shared the same security group ID.
        # This can cause a cross-service outage if not used properly
        service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-53fae93f"

        # A list of additional security groups to be added to the created ELB, this leaves the uniquely
        # generated security group in place, this ensures that every ELB
        # has a unique security group ID and a matching permit line to allow traffic to the target worker nodes
        # (service traffic and health checks).
        # Security groups defined here can be shared between services.
        service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f,sg-42efd82e"

        # A comma separated list of key-value pairs which are used
        # to select the target nodes for the load balancer
        service.beta.kubernetes.io/aws-load-balancer-target-node-labels: "ingress-gw,gw-name=public-api"
```

#### Network Load Balancer support on AWS {#aws-nlb-support}

{{< feature-state for_k8s_version="v1.15" state="beta" >}}

To use a Network Load Balancer on AWS, use the annotation `service.beta.kubernetes.io/aws-load-balancer-type` with the value set to `nlb`.

```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

{{< note >}}
NLB only works with certain instance classes; see the
[AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-register-targets.html#register-deregister-targets)
on Elastic Load Balancing for a list of supported instance types.
{{< /note >}}

Unlike Classic Elastic Load Balancers, Network Load Balancers (NLBs) forward the
client's IP address through to the node. If a Service's `.spec.externalTrafficPolicy`
is set to `Cluster`, the client's IP address is not propagated to the end
Pods.

By setting `.spec.externalTrafficPolicy` to `Local`, the client IP addresses is
propagated to the end Pods, but this could result in uneven distribution of
traffic. Nodes without any Pods for a particular LoadBalancer Service will fail
the NLB Target Group's health check on the auto-assigned
`.spec.healthCheckNodePort` and not receive any traffic.

In order to achieve even traffic, either use a DaemonSet or specify a
[pod anti-affinity](/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
to not locate on the same node.

You can also use NLB Services with the [internal load balancer](/docs/concepts/services-networking/service/#internal-load-balancer)
annotation.

In order for client traffic to reach instances behind an NLB, the Node security
groups are modified with the following IP rules:

| Rule | Protocol | Port(s) | IpRange(s) | IpRange Description |
|------|----------|---------|------------|---------------------|
| Health Check | TCP | NodePort(s) (`.spec.healthCheckNodePort` for `.spec.externalTrafficPolicy = Local`) | Subnet CIDR | kubernetes.io/rule/nlb/health=\<loadBalancerName\> |
| Client Traffic | TCP | NodePort(s) | `.spec.loadBalancerSourceRanges` (defaults to `0.0.0.0/0`) | kubernetes.io/rule/nlb/client=\<loadBalancerName\> |
| MTU Discovery | ICMP | 3,4 | `.spec.loadBalancerSourceRanges` (defaults to `0.0.0.0/0`) | kubernetes.io/rule/nlb/mtu=\<loadBalancerName\> |

In order to limit which client IP's can access the Network Load Balancer,
specify `loadBalancerSourceRanges`.

```yaml
spec:
  loadBalancerSourceRanges:
    - "143.231.0.0/16"
```

{{< note >}}
If `.spec.loadBalancerSourceRanges` is not set, Kubernetes
allows traffic from `0.0.0.0/0` to the Node Security Group(s). If nodes have
public IP addresses, be aware that non-NLB traffic can also reach all instances
in those modified security groups.

{{< /note >}}

Further documentation on annotations for Elastic IPs and other common use-cases may be found
in the [AWS Load Balancer Controller documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/).

### Type ExternalName {#externalname}

Services of type ExternalName map a Service to a DNS name, not to a typical selector such as
`my-service` or `cassandra`. You specify these Services with the `spec.externalName` parameter.

This Service definition, for example, maps
the `my-service` Service in the `prod` namespace to `my.database.example.com`:

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
ExternalName accepts an IPv4 address string, but as a DNS name comprised of digits, not as an IP address.
ExternalNames that resemble IPv4 addresses are not resolved by CoreDNS or ingress-nginx because ExternalName
is intended to specify a canonical DNS name. To hardcode an IP address, consider using
[headless Services](#headless-services).
{{< /note >}}

When looking up the host `my-service.prod.svc.cluster.local`, the cluster DNS Service
returns a `CNAME` record with the value `my.database.example.com`. Accessing
`my-service` works in the same way as other Services but with the crucial
difference that redirection happens at the DNS level rather than via proxying or
forwarding. Should you later decide to move your database into your cluster, you
can start its Pods, add appropriate selectors or endpoints, and change the
Service's `type`.

{{< warning >}}
You may have trouble using ExternalName for some common protocols, including HTTP and HTTPS.
If you use ExternalName then the hostname used by clients inside your cluster is different from
the name that the ExternalName references.

For protocols that use hostnames this difference may lead to errors or unexpected responses.
HTTP requests will have a `Host:` header that the origin server does not recognize;
TLS servers will not be able to provide a certificate matching the hostname that the client connected to.
{{< /warning >}}

{{< note >}}
This section is indebted to the [Kubernetes Tips - Part
1](https://akomljen.com/kubernetes-tips-part-1/) blog post from [Alen Komljen](https://akomljen.com/).
{{< /note >}}

### External IPs

If there are external IPs that route to one or more cluster nodes, Kubernetes Services can be exposed on those
`externalIPs`. Traffic that ingresses into the cluster with the external IP (as destination IP), on the Service port,
will be routed to one of the Service endpoints. `externalIPs` are not managed by Kubernetes and are the responsibility
of the cluster administrator.

In the Service spec, `externalIPs` can be specified along with any of the `ServiceTypes`.
In the example below, "`my-service`" can be accessed by clients on "`80.11.12.10:80`" (`externalIP:port`)

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

## Session stickiness

If you want to make sure that connections from a particular client are passed to
the same Pod each time, you can configure session affinity based on the client's
IP address. Read [session affinity](/docs/reference/networking/virtual-ips/#session-affinity)
to learn more.

## API Object

Service is a top-level resource in the Kubernetes REST API. You can find more details
about the [Service API object](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#service-v1-core).

<!-- preserve existing hyperlinks -->
<a id="shortcomings" /><a id="#the-gory-details-of-virtual-ips" />

## Virtual IP addressing mechanism

Read [Virtual IPs and Service Proxies](/docs/reference/networking/virtual-ips/) to learn about the
mechanism Kubernetes provides to expose a Service with a virtual IP address.

## {{% heading "whatsnext" %}}

Learn more about the following:
* Follow the [Connecting Applications with Services](/docs/tutorials/services/connect-applications-service/) tutorial
* [Ingress](/docs/concepts/services-networking/ingress/) exposes HTTP and HTTPS routes from outside the cluster to services within the cluster.
* [EndpointSlices](/docs/concepts/services-networking/endpoint-slices/)

For more context:
* [Virtual IPs and Service Proxies](/docs/reference/networking/virtual-ips/)
* [API reference](/docs/reference/kubernetes-api/service-resources/service-v1/) for the Service API
* [API reference](/docs/reference/kubernetes-api/service-resources/endpoints-v1/) for the Endpoints API
* [API reference](/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/) for the EndpointSlice API
