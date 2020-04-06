## kubefed for itrix-edge project
Support ARM64 architecture edge device

#

#### docker images for arm64 architecture.
```
docker pull kevin7674/kubefed:latest
```
#
#### kubefedctl
```
chmod u+x kubefedctl
sudo mv kubefedctl /usr/local/bin/
```
```
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com." \
    --apiserver-enable-basic-auth=true \
    --apiserver-enable-token-auth=true \
    --apiserver-arg-overrides="--anonymous-auth=false,--v=4"
```
#
#### deploy 
```
helm install /root/kubefed/charts/federation-v2  --version=v0.0.4 --namespace kube-federation-system 
``` 
#
### Install helm2 on ARM64 
Install server
```
wget https://get.helm.sh/helm-v2.16.5-linux-arm64.tar.gz
tar -zxvf helm-v2.16.5-linux-arm64.tar.gz
mv linux-arm64/helm /usr/local/bin/helm
```
Install client from rebuilt image
```
 helm init --tiller-image=jessestuart/tiller:v2.16.5-arm64
```
check
```
kubectl get pod -n kube-system
```
kubectl create -f helm-rbac.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
Error : no available release name found
```
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
``` 
delete tiller
``` 
kubectl delete deployment tiller-deploy --namespace kube-system
``` 
# 
  
