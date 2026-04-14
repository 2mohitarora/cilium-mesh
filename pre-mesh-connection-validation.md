# Node to Node connectivity works
```
- On cluster-1 run 
  kubectl --context vcluster-docker_cluster-1 get nodes -o wide
- Pick internal IP of any node in cluster-1
- On cluster-2 run 
  kubectl --context vcluster-docker_cluster-2 run ping-test --rm -it --image=busybox --restart=Never -- ping -c 4 <cluster-1-node-ip>
```

# Pod to Pod connectivity doesnt work
```
- On cluster-1 run 
  kubectl --context vcluster-docker_cluster-1 get pods --all-namespaces -w -o wide
- Pick internal IP of any pod in cluster-1
- On cluster-2 run 
  kubectl --context vcluster-docker_cluster-2 run cross-test --rm -it --image=busybox --restart=Never -- ping -c 3 <cluster-1-pod-ip>
``` 

# Check cilium and clustermesh status
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium status --brief
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium status --brief

kubectl --context vcluster-docker_cluster-1 get cm cilium-config -n cilium -o yaml | grep -E "cluster-name|cluster-id|routing-mode|tunnel-protocol|kube-proxy-replacement|max-connected-clusters|clustermesh-enable-mcs-api"

kubectl --context vcluster-docker_cluster-2 get cm cilium-config -n cilium -o yaml | grep -E "cluster-name|cluster-id|routing-mode|tunnel-protocol|kube-proxy-replacement|max-connected-clusters|clustermesh-enable-mcs-api"

cilium clustermesh status --context vcluster-docker_cluster-1 -n cilium --wait
cilium clustermesh status --context vcluster-docker_cluster-2 -n cilium --wait
```

# Check cluster nodes
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium node list
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium node list
```

# Check cluster identities
```
kubectl --context vcluster-docker_cluster-1 exec -n cilium ds/cilium -- cilium identity list | head -20
kubectl --context vcluster-docker_cluster-2 exec -n cilium ds/cilium -- cilium identity list | head -20
```

# Compare the Cert-manager CA
```
kubectl --context vcluster-docker_cluster-1 get secret cilium-ca-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | md5sum
kubectl --context vcluster-docker_cluster-2 get secret cilium-ca-secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | md5sum

# CA in the server cert (what etcd trusts)
kubectl --context vcluster-docker_cluster-1 get secret clustermesh-apiserver-server-cert -n cilium -o jsonpath='{.data.ca\.crt}' | md5sum
kubectl --context vcluster-docker_cluster-2 get secret clustermesh-apiserver-server-cert -n cilium -o jsonpath='{.data.ca\.crt}' | md5sum

# CA in the remote cert (what kvstoremesh presents)
kubectl --context vcluster-docker_cluster-1 get secret clustermesh-apiserver-remote-cert -n cilium -o jsonpath='{.data.ca\.crt}' | md5sum
kubectl --context vcluster-docker_cluster-2 get secret clustermesh-apiserver-remote-cert -n cilium -o jsonpath='{.data.ca\.crt}' | md5sum
```