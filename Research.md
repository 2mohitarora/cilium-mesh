# Cilium ClusterMesh

### cluster.name
- This is a human-readable string that uniquely identifies the cluster within your mesh.
- Purpose: It is used by Cilium to prefix or identify resources (like nodes and endpoints) coming from this specific cluster.
- Why it matters: When you look at service identities or network policies across a multi-cluster setup, the cluster.name tells you exactly where a workload is running.

### cluster.id
- This is a unique integer (1-255) assigned to the cluster.
- Purpose: Cilium uses this ID to help generate Global Identities. In a Cluster Mesh, security identities are shared. By including the cluster.id in the identity calculation, Cilium ensures that an identity from "Cluster A" doesn't accidentally collide with or get confused for an identity from "Cluster B."
- Constraint: Every cluster in your mesh must have a unique ID. If you accidentally give two clusters the ID 2, the mesh will fail to route traffic correctly because the identities will conflict.

## How Cilium Routing works
When Cluster 1 wants to talk to Cluster 2, Cilium uses one of two primary methods

- Encapsulation (Overlay): This is the most common method. Cilium wraps the Pod-to-Pod traffic inside a standard VXLAN or Geneve packet. Since this packet is addressed to the destination Node IP (which is reachable), the underlying network doesn't need to know anything about your Pod CIDRs (like 10.2.0.0/16).
    - The Packet: A Pod in Cluster 1 sends a request to a Pod in Cluster 2.
    - The Wrapper: The Cilium agent on the source node sees the destination is "remote." It wraps the original packet inside a VXLAN (or Geneve) header.
    - The Delivery: This new "outer" packet is addressed to the IP of the Node in Cluster 2. Because your networks are connected (peered), the underlying routers know how to deliver this node-to-node traffic.
    - The Unwrap: The Cilium agent on the receiving node in Cluster 2 strips off the VXLAN header and delivers the original packet to the destination Pod.
- Native Routing: If your underlying network (like a cloud VPC) is already configured to know how to route your Pod CIDRs between clusters, Cilium can send the traffic directly without wrapping it in a tunnel.

### Role of the "Cilium ClusterMesh API Server"
To make this work, each cluster runs a small service that "advertises" its state to the other clusters.
1. Cluster 1 tells Cluster 2: "I have these Pods with these Identities"
2. When a Pod in Cluster 1 tries to reach a service in Cluster 2, Cilium looks at its local map, sees the destination is in Cluster 2, and knows exactly which Node IP to send the traffic to.

### Verify Node-to-Node Reachability
```
# Get a Node IP from Cluster 2
NODE_CP2=$(kubectl --context $CLUSTER2 get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# Try to ping it from a temporary pod in Cluster 1
kubectl --context $CLUSTER1 run tracer --image=busybox -- restart=Never -- ping -c 4 $NODE_CP2

# Run cilium connectivity test
cilium connectivity test --context $CLUSTER1 --destination-context $CLUSTER2
```

### VXLAN or Geneve, whats the difference?
In the world of Cilium and Cluster Mesh, VXLAN is the reliable old workhorse, while Geneve is the flexible newcomer designed for the future of software-defined networking.

## Mandatory Requirements for Cluster Mesh
- Pod CIDRs must not overlap
  - Cilium routes traffic between pods across clusters using their IP addresses. If Cluster 1 and Cluster 2 both use 10.244.0.0/16, a pod in Cluster 1 trying to talk to 10.244.0.5 will always look locally. It will never realize that the destination might actually be a pod in Cluster 2 with that same IP.
- Node CIDRs must not overlap 
  - Mandatory
- Service CIDRs must not overlap (--service-cluster-ip-range)
  - This is critical for the "Service Discovery" mechanism. When a pod in Cluster A tries to connect to a Service IP (e.g., 10.96.0.1), Cilium needs to know which cluster owns that IP. If both clusters use the same range, the traffic gets "swallowed" by the local cluster (IP shadowing).

## ClusterMesh concepts

### etcd
In a Cilium Cluster Mesh, you spin up a second etcd specifically for the clustermesh-apiserver. This etcd is used to store the cluster mesh configuration and the service information.

  - Local Write: Each cluster has its own clustermesh-apiserver. It watches the local Kubernetes API and writes the relevant networking data into its local kvstore (etcd).
  - Remote Read: When you connect Clusters, the Cilium agents in each cluster create a read-only connection to the other clusters' kvstores.
  - Discovery: Each cluster now "sees" the identities and services of the other clusters as if they were local. It uses this information to build eBPF routing tables.

In scenario with 100+ clusters, the "classic" kvstore model has a bottleneck: every single Cilium agent (on every node) would have to maintain 100 separate connections to 100 different remote etcd instances. This does not scale.

To solve this, Cilium uses KVStoreMesh (enabled by default in version 1.16+):

  - The Cache: Instead of every node connecting to every cluster, each cluster runs a local KVStoreMesh pod.
  - The Fan-out: This pod connects to the 99 other clusters, pulls their data, and caches it into a local etcd.
  - The Benefit: Your nodes only talk to their one local cluster cache, drastically reducing network overhead and memory usage across the fleet.

### What is a global service
```
service.cilium.io/global: "true" (identifies the service as a mesh participant)

- What it does: It tells the Cilium clustermesh-apiserver that this service should be "shared" across all clusters in the mesh.
- How it works: When a pod in Cluster A tries to connect to this service, Cilium automatically routes the traffic to the nearest healthy instance of the service in Cluster B (or Cluster A itself).
```

### if a service a in cluster-1 need to talk to service B which only exists on cluster-2, will service B need to be annotated with service.cilium.io/global: "true"?

That alone won't be enough, By default, Kubernetes DNS (core-dns) only knows about services that exist in its local API server. If you don't create the Service resource in Cluster 1, the Pods there will get a NXDOMAIN (Name Not Found) error when they try to look up Service B. You must create the Service B resource in both clusters, even if Cluster 1 has 0 pods for it.
 
### What does service.cilium.io/affinity: "local" mean?
This tells Cilium to prefer local backends and only fall back to remote clusters if no healthy local pods exist. Other values include remote, none (the default, which means no preference). This is commonly used for active/standby failover patterns.

### What does service.cilium.io/shared: "false" mean?
If you set this on a particular cluster's copy of the service, that cluster will consume remote endpoints but won't share its own local endpoints with other clusters. 

### How do headless services work in cluster mesh
When you create a Headless Service (clusterIP: None), Kubernetes does not assign it an IP. Instead, it creates A records in DNS that point directly to the IPs of the backing Pods. if you annotate a Headless Service with service.cilium.io/global: "true" and service.cilium.io/global-sync-endpoint-slices: "true":
- Endpoint Syncing: Cilium’s clustermesh-apiserver synchronizes the Pod IPs (Endpoints) from Cluster B into Cluster A.
- DNS Resolution: When a pod in Cluster A queries the DNS name of the service, it receives the IPs of the pods in Cluster B directly.
- Routing: Cilium uses this information to route traffic directly to the remote pod IPs.

Event in this case a local Service object must exist in each cluster that wants to consume the headless global service.

### What does service.cilium.io/global-sync-endpoint-slices: "true" mean?
This annotation tells Cilium to sync remote cluster endpoints into the local Kubernetes API as actual EndpointSlice objects. This allows the local kube-dns (or CoreDNS) to resolve the service name to the remote pod IPs. For headless global services, this is enabled by default. For ClusterIP global services, this is disabled by default. 

### Does Affinity work for headless services?
Yes, Affinity works for headless services. When you set service.cilium.io/affinity: "local" on a headless service, Cilium will prefer local endpoints over remote endpoints. It uses DNS filtering. If you set affinity to local, Cilium will try to ensure that the DNS response in Cluster A primarily contains Pod IPs from Cluster A.

With ClusterIP services, the fallback can happen per-connection at the datapath level — very fast and transparent. With headless services, the fallback depends on DNS TTLs and re-resolution. If local pods go down, clients won't discover remote backends until they re-query DNS and get an updated response that now includes remote IPs. So the failover is slightly less immediate.

### Above doesn't scale what are my options?
The Scalable Solution: MCS-API, Multi-Cluster Services API (ServiceExport)
Instead of defining a service in every cluster that might need it, you use a "Push" model. You define the service once in the source cluster and "export" it to the entire mesh.

- Target Cluster (Cluster 11): You create a ServiceExport object. This tells Cilium, "Make this service available to the rest of the fleet."

- Client Clusters (1-10): Cilium automatically detects the export and (if configured) can dynamically handle the discovery.

- The DNS Name: Clients use a special cluster-set domain: target-service.namespace.svc.clusterset.local

```
# Enable MCS API in Cilium
clustermesh:
  mcsapi:
    enabled: true
    # This automatically adds the necessary DNS rules to CoreDNS
    corednsAutoConfigure: true

Create a ServiceExport in the source cluster

apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: target-service
  namespace: prod

clients simply change their connection string. Instead of hitting target-service.prod.svc.cluster.local, they hit: <svc>.<ns>.svc.clusterset.local  

```

### What is clustermesh.defaultGlobalNamespace
Even for mcsapi to work, we need namespace to exist in client cluster. This is very similar challenge to dummy service to exist in client clsuster. With clustermesh.defaultGlobalNamespace cilium assumes that every namespace with the same name across all clusters is the same logical entity. Automatically sync all identities and endpoints for these namespaces across the mesh. It still doesnt solve the dummy namespace problem

### Does MCS API work with headless services?
Yes, MCS API works seamlessly with headless services. When a headless service is exported, the MCS controller creates an EndpointSlice resource in the target cluster that contains the IPs of the pods in the source cluster. The client cluster can then use this EndpointSlice to route traffic directly to the remote pods.

### Traditional Global Services vs MCS API
The traditional "Global Service" (annotation) approach is a mesh-wide broadcast. Every cluster tells every other cluster about its services. In a 100-cluster fleet, that is 100*100 connections—an exponential growth in control-plane noise. MCS-API changes this to a Selective Export model

- Explicit Publishing: Only services you explicitly ServiceExport are shared. This prevents "pollution" of the API servers in your other 99 clusters.
- Lower Control-Plane Stress: Cilium only synchronizes state for exported services. If Cluster 11 exports a database, Clusters 1–10 only receive updates for that specific service, not everything happening in Cluster 11.

### topology.kubernetes.io/region and topology.kubernetes.io/zone labels
Cilium fully supports Topology-Aware Routing. By using the topology.kubernetes.io/region and topology.kubernetes.io/zone labels, Cilium can make even smarter decisions—preferring the same Zone first, then the same Region, and only then crossing to a different Region as a last resort.

### How the "cilium-health" Daemon works
Every Cilium agent runs a side-process called the cilium-health responder. It performs a "full-stack" health check across the mesh:

- ICMP (Ping): Can the nodes see each other at the network layer?
- HTTP (Health Endpoints): Is the Cilium agent on the remote node actually responding?
- Connectivity Paths: It tests both the Node IP and a special Health Endpoint IP on every remote node.

kubectl exec -n kube-system ds/cilium -- cilium-health status --all-clusters

### What do CiliumNetworkPolicy and endpointSelector mean?
CiliumNetworkPolicy allows you to write rules based on Service Identities, DNS names, and Application Protocols (Layer 7). In Cilium's world, an Endpoint is the network representation of a Pod. Every pod in your cluster is assigned a unique Identity based on its labels.

```
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-access
spec:
  endpointSelector:
    matchLabels:
      app: backend  # This selects the Pods we want to protect
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend  # Only allow traffic from the frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

### What is CiliumClusterwideNetworkPolicy
Standard policies are "namespaced," meaning you have to copy them into every namespace. A CCNP is a global object. You can write one policy that says: "Across all 100 clusters, no pod labeled env: dev can talk to a pod labeled env: prod.

### What is EndpointSlice Mirroring
In a large-scale Cilium environment, EndpointSlice Mirroring (often referred to in Cilium as clustermesh.enableEndpointSliceSynchronization) is a mechanism that solves a "visibility" problem for tools that live outside the eBPF datapath. It also helps with the headless services.

While Cilium's eBPF engine knows exactly how to route traffic to a remote cluster, standard Kubernetes controllers (like external Ingress controllers, some Service Meshes, or traditional kube-proxy setups) are "blind" to those remote IPs unless they see a local Kubernetes object representing them.

### How to check clustermesh status
```
cilium clustermesh status # Overall health of the connection
cilium connectivity test --multi-cluster # Tests the actual data path between pods.
kubectl logs -n kube-system -l k8s-app=clustermesh-apiserver # Shows why the control planes aren't syncing.
cilium identity list # Check if you see identities with a cluster-id of 1 and 2.

Even if your clusters are peered, specific ports must be open between all nodes in both clusters. If these are blocked, the clusters will look connected but traffic will simply vanish.
- Port 4240 (TCP): Cluster health checks. Without this, Cilium thinks the remote nodes are dead.
- Port 2379/2380 (TCP): The ClusterMesh API Server (etcd) ports. This is how the "brains" of the clusters talk to each other.
- Port 8472 (UDP): If you are using VXLAN (Overlay). This is the "pipe" your actual Pod data travels through.
- ICMP (Ping): Cilium uses this for health monitoring; if disabled, you’ll see "unreachable" errors.
- Cilium Cluster Mesh relies on a Shared Certificate Authority (CA).
```
### Does cilium connectivity test Service CIDR overlap as well (service-cluster-ip-range)
No it does not check for service cidr overlap

### What is cilium identity list
cilium identity list command allows you to see every identity currently known to a specific node’s Cilium agent.

### Is cilium agent an envoy proxy? 
No, the Cilium Agent is not an Envoy proxy, but it manages one. In your environment, it's helpful to think of the Cilium Agent as the Control Plane for each node, while Envoy is a specialized worker it calls upon when eBPF isn't enough.

Cilium Agent (cilium-agent): This is a Go-based daemon. Its primary job is to watch the Kubernetes API, manage eBPF programs in the Linux kernel, and handle identity-based networking. It operates mostly at Layers 3 and 4 (IP and Ports).Cilium Envoy (cilium-envoy): This is a specialized distribution of the Envoy proxy. The Cilium Agent starts and configures this Envoy instance to handle Layer 7 (Application layer) tasks like HTTP/gRPC routing, retries, and Kafka protocol parsing. Envoy runs as its own dedicated DaemonSet (cilium-envoy). This is often preferred in large-scale production (like yours) because it allows you to restart the Cilium Agent (to update BPF logic) without dropping live L7 traffic being handled by Envoy.

### What is loadbalancerMode: dedicated
By default, Cilium often uses a "shared" Envoy instance on each node to handle all Layer 7 traffic. When you switch a Gateway to loadbalancerMode: dedicated, you are telling Cilium to spin up a unique, isolated deployment of Envoy proxies specifically for that one Gateway resource. Traffic for this Gateway never touches the "Shared" Envoy that handles internal mesh traffic. It stays within its own process and resource boundaries.

### Can clusters across regions be on mesh?

### Can ClusterMesh be enabled during helm install?
```
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set cluster.name=cluster-11 \
  --set cluster.id=11 \
  --set clustermesh.useAPIServer=true \
  --set clustermesh.apiserver.service.type=LoadBalancer \
  --set clustermesh.apiserver.tls.auto.enabled=true

While Helm can enable the Cluster Mesh (it starts the clustermesh-apiserver and generates the certs), Helm cannot automatically connect Cluster 1 to Cluster 11. You still need to run the `cilium clustermesh connect` command on both clusters to establish the peering relationship.  
```

### Can Hubble be enabled during helm install?
```
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    # Enable common metrics for Prometheus scraping
    enabled:
    - dns
    - drop
    - tcp
    - flow
    - icmp
    - http

helm install cilium cilium/cilium \
  --namespace kube-system \
  -f values.yaml    
```
### Does Hubble need to run on every cluster?
The Core Components

- Hubble: The local observer on every node (low-level eBPF data collection). The core Hubble component is embedded inside the cilium-agent. Mandatory for data capture.

- Hubble Relay: The aggregator that talks to all nodes to give you a "cluster-wide" view. You can configure a Single Hubble Relay to aggregate flows from multiple clusters in the mesh, but not recommended. 

- Hubble UI: The graphical dashboard for service maps.

- Hubble Timescape: you'll eventually need a persistent store (like Grafana Tempo or ClickHouse) because Hubble's "live" buffer is tiny.

### ServiceExport and ServiceImport flow

Cluster-2                                          Cluster-1
─────────                                          ─────────

1. User creates ServiceExport
        │
        ▼
2. cilium-operator watches it
        │
        ▼
3. operator writes service export
   info into local ClusterMesh etcd
        │
        ▼
4. ClusterMesh etcd stores it at
   cilium/state/serviceexports/v1/
        │
        │
        ├──── KVStoreMesh in cluster-1 has a
        │     read-only connection to cluster-2's
        │     ClusterMesh etcd
        │
        ▼
5. KVStoreMesh in cluster-1 detects
   the new key, pulls it, caches it
   into cluster-1's local etcd
        │
        ▼
6. cilium-operator in cluster-1 watches
   local etcd for remote service exports
        │
        ▼
7. operator creates ServiceImport
   in cluster-1's Kubernetes API
        │
        ▼
8. operator creates derived-$hash
   Service in cluster-1
        │
        ▼
9. CoreDNS sees the ServiceImport
   via multicluster plugin
        │
        ▼
10. web.mcs-test.svc.clusterset.local
    is now resolvable in cluster-1

# CiliumAgent

cilium-agent
    │
    ├──→ Kubernetes API (for local cluster data)
    │    - Local pods, services, endpoints
    │    - Local CiliumIdentity CRDs
    │    - Local CiliumNetworkPolicy
    │    - Local CiliumEndpointSlice
    │
    ├──→ Local ClusterMesh etcd (for remote cluster data)
    │    - Remote identities
    │    - Remote nodes
    │    - Remote services
    │    - Remote endpoints
    │
    ✗ Never talks to remote ClusterMesh etcd directly
      (KVStoreMesh does that on its behalf)