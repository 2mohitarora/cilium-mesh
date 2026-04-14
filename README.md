# Install Tools
```
brew install kubectl helm docker vcluster k9s
```

# Install Orbstack
```
brew install --cask orbstack

# Start Orbstack
```

# Configure docker
```
docker context use orbstack
export DOCKER_HOST="unix:///Users/mua0008/.orbstack/run/docker.sock"
docker context list
```

# Create your first vcluster
Follow instructions in ./cluster-1/README.md

# Create your second cluster 
Follow instructions in ./cluster-2/README.md

# Do some pre-mesh connectivity validations first
Follow instructions in ./pre-mesh-connection-validation.md

# Have both mesh communicate with each other
```
# Get the clustermesh IP from cluster-2
cilium clustermesh status --context vcluster-docker_cluster-2 -n cilium --wait

# Update cluster-1/cilium/cluster-connect-2.yaml with clustermesh IP

# Navigate to cluster-1 folder

helm upgrade cilium cilium/cilium --version 1.19.1 \
  -n cilium --create-namespace --kube-context vcluster-docker_cluster-1 \
  -f ../base-values.yaml \
  -f ../mesh-values.yaml \
  -f cilium/cluster-connect-2.yaml

# Get the clustermesh IP from cluster-1
cilium clustermesh status --context vcluster-docker_cluster-1 -n cilium --wait

# Update cluster-2/cilium/cluster-connect-1.yaml with clustermesh IP

# Navigate to cluster-2 folder

helm upgrade cilium cilium/cilium --version 1.19.1 \
  -n cilium --create-namespace --kube-context vcluster-docker_cluster-2 \
  -f ../base-values.yaml \
  -f ../mesh-values.yaml \
  -f cilium/cluster-connect-1.yaml

# Had to restart opeartor on cluster-2
kubectl --context vcluster-docker_cluster-2 rollout restart daemonset cilium -n cilium
```
# Do some post-mesh connectivity validations
Follow instructions in ./post-mesh-connection-validation.md

# Verify both clusters are running
```
vcluster list
```

# Check all the docker containers that are running
```
docker ps --format "table {{.Names}}\t{{.Status}}"
```

# Describe clusters
```
vcluster describe cluster-1
vcluster describe cluster-1
```

# Cleanup
```
vcluster delete cluster-1
vcluster delete cluster-2
```

# Sample command for logs
```
# View control plane logs
docker exec vcluster.cp.cluster-1 journalctl -u vcluster --no-pager

# View worker node kubelet logs
docker exec vcluster.node.cluster-1.worker-1 journalctl -u kubelet --no-pager
```

# Networking
- Each cluster uses a separate Docker network (vind-cluster-1 and vind-cluster-2) to keep them isolated from each other.
- LoadBalancer is enabled for both clusters
