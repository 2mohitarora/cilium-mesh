# Instructions for second cluster

# Extract CA from cluster-1
```
kubectl get secret -n cert-manager cilium-ca-secret -o yaml > cilium-ca-secret.yaml
```

# Create cluster-2
```
sudo vcluster create cluster-2 --driver docker --values cluster-2.yaml
```
# Add the Jetstack repo
```
helm repo add cilium https://helm.cilium.io/
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
# Upgrade Coredns as Ciliummesh is not able to modify older version of coredns
```
kubectl set image deployment/coredns \
  -n kube-system \
  coredns=registry.k8s.io/coredns/coredns:v1.14.2
```
# Install Gateway API CRD
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/experimental-install.yaml
```
# Install cilium 
```
helm install cilium cilium/cilium --version 1.19.1 \
  -n cilium --create-namespace \
  -f ../base-values.yaml \
  -f cilium/cluster.yaml

```
# After CNI is installed, wait for pods to become Ready:
```
kubectl get pods --all-namespaces -w -o wide
```
# Install cert-manager
```
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```
# After cert-manager, wait for pods to become Ready:
```
kubectl get pods --all-namespaces -w -o wide
```
# Import the CA certificate
```
kubectl apply -f cilium-ca-secret.yaml -n cert-manager
kubectl apply -f cert-manager-issuer.yaml
kubectl get clusterissuer cilium-ca-issuer 
```

# Upgrade Cilium with ClusterMesh
```
helm upgrade cilium cilium/cilium --version 1.19.1 \
  -n cilium --create-namespace \
  -f ../base-values.yaml \
  -f ../mesh-values.yaml \
  -f cilium/cluster.yaml
  
# After Mesh is installed, wait for pods to become Ready:
kubectl get pods --all-namespaces -w -o wide

# Check cilium status
cilium status --namespace cilium
```
# Update CoreDNS ConfigMap. We have to do this because helm flag corednsAutoConfigure is not working as expected
```
kubectl edit configmap coredns -n kube-system

# Add the following to the configmap
kubernetes cluster.local clusterset.local in-addr.arpa ip6.arpa {
multicluster clusterset.local

kubectl edit clusterrole system:coredns

# Add following config in rules section
- apiGroups:
  - multicluster.x-k8s.io
  resources:
  - serviceimports
  verbs:
  - list
  - watch
```

# Check Gateway Class and Create Cilium Gateway
```
kubectl get gatewayclasses -o wide
kubectl apply -f ../cilium-gateway.yaml
kubectl get gateways -n cilium
kubectl get svc -l io.cilium.gateway/owning-gateway=default-gateway -n cilium
```

# Install Dapr Inject Dapr sidecar into Traefik deployment
```
dapr init -k

dapr status -k

# Make sure Dapr containers are ready
kubectl get pods -o wide -n dapr-system

# Run DAPR UI

dapr dashboard -k

```
# Configure Registry for second cluster
```
# Start a local registry on the same Docker network as your vind cluster
docker run -d --name registry-2 --network vind-cluster-2 -p 5051:5000 registry:2

# Configure registry for cluster-2 so that nodes can pull from insecure registry
./cluster-2-script.sh
```
# Disconnect from first cluster
```
vcluster disconnect
```

