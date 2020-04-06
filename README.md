## kubefed for itrix-edge project
Support ARM64 architecture edge device.

Not support K8S 1.16+ now, Kubefed only support deployment with helm2 at the moment, especially since helm3 is not officially released yet (still in beta).

#
### Use kubefed
#### kubefedctl
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
### Deploy 

#### docker images for arm64 architecture.
```
docker pull kevin7674/kubefed:latest
```
vi values.yaml
```
cd /kubefed/charts/kubefed
vi values.yaml
	repository: kevin7674
	image: kubefed
	tag: latest
```
#### deploy from local helm chart
```	
helm install /root/kubefed/charts/kubefed  --version=v0.1.0-rc6 --namespace kube-federation-system
```
check deployment
```
kubectl get all -n kube-federation-system
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
check helm
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

### Rebuild on arm64 env

Install GOLANG:
``` 
wget https://dl.google.com/go/go1.14.1.linux-arm64.tar.gz
tar -C /usr/local -xzf go1.14.1.linux-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin
``` 
code build
``` 
git clone https://github.com/kubernetes-sigs/kubefed.git
cd kubefed
git checkout v0.1.0-rc6
make build .
``` 	
use kubefedctl
``` 
cd /kubefed/bin
./kubefedctl
chmod u+x kubefedctl-linux-arm64
sudo cp kubefedctl-linux-arm64 /usr/local/bin
``` 	
Build docker image:
```
cd /kubefed
mv /kubefed/images/kubefed/Dockerfile . 
vi Dockerfile
	<COPY /hyperfed .> ---> <COPY bin/hyperfed .>
docker build -t kevin7674/kubefed:latest .
docker push kevin7674/kubefed:latest
```


