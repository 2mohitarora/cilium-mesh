# Instructions for first cluster
```
sudo vcluster create cluster-1 --driver docker --values cluster-1.yaml
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
# Install cert-manager issuer
```
kubectl apply -f cert-manager-issuer.yaml
kubectl get clusterissuer cilium-ca-issuer 
kubectl get certificates -n cert-manager 
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

# Disconnect from first cluster
```
vcluster disconnect
```

