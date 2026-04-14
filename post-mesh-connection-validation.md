### Check clustermesh status
```
cilium clustermesh status --context vcluster-docker_cluster-1 --namespace cilium
cilium clustermesh status --context vcluster-docker_cluster-2 --namespace cilium
```
### Each cluster should see nodes from the other
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium node list
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium node list
```

### Identities from both clusters should be visible
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium identity list
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium identity list
```

### Full connectivity test across the mesh
```
cilium connectivity test --context vcluster-docker_cluster-1 --multi-cluster vcluster-docker_cluster-2 --test pod-to-pod,pod-to-service --namespace cilium

# Clean up test resources
kubectl --context vcluster-docker_cluster-1 delete ns cilium-test-1
kubectl --context vcluster-docker_cluster-2 delete ns cilium-test-1
```

### Create dns-validator pod on both clusters
```
kubectl --context vcluster-docker_cluster-1 apply -f dns-validator.yaml
kubectl --context vcluster-docker_cluster-2 apply -f dns-validator.yaml
```
### MCS-API Validator, Scearion 1 : Service only on cluster-2
```
# Create mcs-test namespace of cluster-2
kubectl --context vcluster-docker_cluster-2 create namespace mcs-test
kubectl --context vcluster-docker_cluster-2 label namespace mcs-test app.kubernetes.io/part-of=application

# Create deployment, service and serviceexport on cluster-2
kubectl --context vcluster-docker_cluster-2 apply -f cluster-2/mcs-test.yaml

# Check serviceexport and serviceimport objects on cluster-2
kubectl --context vcluster-docker_cluster-2 get serviceexport -n mcs-test
kubectl --context vcluster-docker_cluster-2 get serviceimport -n mcs-test

# Try to resolve the remote service
kubectl --context vcluster-docker_cluster-1 exec dns-validator -- nslookup web.mcs-test.svc.clusterset.local

# Create namespace on cluster-1
kubectl --context vcluster-docker_cluster-1 create namespace mcs-test
kubectl --context vcluster-docker_cluster-1 label namespace mcs-test app.kubernetes.io/part-of=application

# Check serviceimport objects appearing on cluster-1
kubectl --context vcluster-docker_cluster-1 get serviceimport -n mcs-test
kubectl --context vcluster-docker_cluster-1 get svc -n mcs-test

# Try to resolve the remote service again
kubectl --context vcluster-docker_cluster-1 exec dns-validator -- nslookup web.mcs-test.svc.clusterset.local

kubectl --context vcluster-docker_cluster-1 run curl-test --rm -it --image=curlimages/curl --restart=Never -n mcs-test --labels="app=curl-test" -- curl -v -s --max-time 5 http://web.mcs-test.svc.clusterset.local -o /dev/null -w "Response from: %{remote_ip}\n"

What this proves: Cilium’s MCS controller has successfully synced the ServiceImport and CoreDNS is correctly configured with the clusterset stub-domain.
```

### MCS-API Validator, Scearion 2 : Headless service on cluster-1 only and we will resolve it from cluster-2
```
# Create deployment, headless service and serviceexport on cluster-1
kubectl --context vcluster-docker_cluster-1 apply -f cluster-1/mcs-headless-test.yaml

# Check serviceexport and serviceimport objects on cluster-1
kubectl --context vcluster-docker_cluster-1 get serviceexport -n mcs-test
kubectl --context vcluster-docker_cluster-1 get serviceimport -n mcs-test

# Check serviceimport objects appearing on cluster-2 (Had to restart operator cluster-2)
kubectl --context vcluster-docker_cluster-2 get serviceimport -n mcs-test
# Had to restart the operator on cluster-2 for serviceimport to appear
kubectl --context vcluster-docker_cluster-2 rollout restart deployment cilium-operator -n cilium

# Try to resolve the remote service
kubectl --context vcluster-docker_cluster-2 exec dns-validator -- nslookup web-headless.mcs-test.svc.clusterset.local

kubectl --context vcluster-docker_cluster-2 run curl-test --rm -it --image=curlimages/curl --restart=Never -n mcs-test --labels="app=curl-test" -- curl -v -s --max-time 5 http://web-headless.mcs-test.svc.clusterset.local -o /dev/null -w "Response from: %{remote_ip}\n"

# The pod IPs returned by DNS are remote pod IPs, but the actual packets still travel node-to-node through the VXLAN tunnel. The client pod never talks directly to the remote pod over some magic path.
```

### MCS-API Validator, Scearion 3 : CiliumNetworkPolicy and L7 Policy and CiliumClusterwideNetworkPolicy 
 CiliumNetworkPolicy (CNP) and CiliumClusterwideNetworkPolicy (CCNP) are not automatically replicated across clusters in a Cilium Cluster Mesh. Cluster Mesh synchronizes identities, pods, and services to allow cross-cluster communication, security policies must be managed separately in each cluster

### Deny all remote cluster traffic (CiliumClusterwideNetworkPolicy)
```
kubectl --context vcluster-docker_cluster-1 apply -f policies/deny-all-cluster-1.yaml
kubectl --context vcluster-docker_cluster-2 apply -f policies/deny-all-cluster-2.yaml

kubectl --context vcluster-docker_cluster-1 get ccnp
kubectl --context vcluster-docker_cluster-2 get ccnp

# Try connecting from cluster-1 to cluster-2 and cluster-2 to cluster-1, it will fail
```

### Allow cluster-1 to access web (CiliumNetworkPolicy)
```
kubectl --context vcluster-docker_cluster-2 apply -f policies/allow-web.yaml
kubectl --context vcluster-docker_cluster-2 get cnp -n mcs-test
```

### Allow cluster-2 to access web-headless (CiliumNetworkPolicy)
```
kubectl --context vcluster-docker_cluster-1 apply -f policies/allow-web-headless.yaml
kubectl --context vcluster-docker_cluster-1 get cnp -n mcs-test
```

### Only allow prod to prod (CiliumClusterwideNetworkPolicy)
```
kubectl --context vcluster-docker_cluster-1 apply -f policies/only-allow-prod-to-prod.yaml
kubectl --context vcluster-docker_cluster-2 apply -f policies/only-allow-prod-to-prod.yaml
```

### Helpful commands
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium endpoint list
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium endpoint list

kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium monitor --type drop
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium monitor --type drop

# After applying allow policy, had to restart cilium pods and recreate web to make it work
kubectl --context vcluster-docker_cluster-2 delete pods -n cilium -l k8s-app=cilium
kubectl --context vcluster-docker_cluster-2 get pods -n cilium -l k8s-app=cilium -w
kubectl --context vcluster-docker_cluster-2 rollout restart deployment web -n mcs-test
```

### The Hubble "Flow-Watch" Command
Finally, use Hubble to see the "Identity" magic in action. This command allows you to see traffic filtered by the numeric Identity we discussed earlier.
```
### Port-forward Hubble if not already open
cilium hubble port-forward -n cilium

### Watch flows showing the Identity IDs
hubble observe --follow --output json | jq '{
  time: .flow.time,
  source_identity: .flow.source.identity,
  source_labels: .flow.source.labels,
  source_pod: .flow.source.pod_name,
  dest_identity: .flow.destination.identity,
  dest_labels: .flow.destination.labels,
  dest_pod: .flow.destination.pod_name,
  verdict: .flow.verdict
}'

### Compact version
hubble observe --follow --output json | jq -c '{
  src: (.flow.source.pod_name // .flow.source.labels[0]),
  dst: (.flow.destination.pod_name // .flow.destination.labels[0]),
  verdict: .flow.verdict,
  port: .flow.l4.TCP.destination_port
}'

### Cross cluster traffic only
hubble observe --follow --output json | jq -c 'select(.flow.source.cluster_name != .flow.destination.cluster_name) | {
  src_cluster: .flow.source.cluster_name,
  src_pod: .flow.source.pod_name,
  dst_cluster: .flow.destination.cluster_name,
  dst_pod: .flow.destination.pod_name,
  verdict: .flow.verdict
}'

### Filter for drops only
hubble observe --follow --output json | jq -c 'select(.flow.verdict == "DROPPED") | {
  src: .flow.source.pod_name,
  dst: .flow.destination.pod_name,
  drop_reason: .flow.drop_reason_desc,
  verdict: .flow.verdict
}'
```
### CiliumIdentity
```
1. Pod gets created
2. Local cilium-agent computes an identity from the pod's labels
3. Agent writes that identity to the local Kubernetes API (CiliumIdentity CRD)
4. cilium-operator syncs it into the local KVStore (etcd)
5. KVStoreMesh in remote clusters pulls it into their local KVStore
6. Remote cilium-agents read it from their local KVStore
7. Now remote clusters can recognize traffic from that identity
```

### What labels does identity 169568 have?
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium identity get 169568
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium identity get 169568
```