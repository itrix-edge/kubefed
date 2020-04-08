## kubefed for itrix-edge project
Support ARM64 architecture edge device.

Not support K8S 1.16+ now, Kubefed only support deployment with helm2 at the moment, especially since helm3 is not officially released yet (still in beta).

#
### Use kubefed

An example join cluster1 to kubefed host cluster :

vi $HOME/.kube/config
```yml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.172.29.1:6443
  name: kubernetes
- cluster:
    certificate-authority-data: <>
    server: <>
  name: cluster1
contexts:
- context:
    cluster: cluster1
    user: cluster1
  name: cluster1
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: cluster1
  user:
    client-certificate-data: <>
    client-key-data: <>
```
check cluster contexts
```sh
$ kubectl config get-contexts
```
```sh
root@xavier01:~# kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          cluster1                      cluster1     cluster1
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```
join cluster to federation
```sh
$ kubefedctl join <cluster1> --cluster-context <cluster1> --host-cluster-context <cluster1>
```
check federation cluster
```sh
$ kubectl -n kube-federation-system get kubefedclusters
```
```sh
root@xavier01:~# kubectl -n kube-federation-system get kubefedclusters
NAME       READY   AGE
cluster1   True    23h
```
#
### Deploy 

#### docker images for arm64 architecture.
```sh
$ docker pull kevin7674/kubefed:latest
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
```sh	
$ helm install /root/kubefed/charts/kubefed  --version=v0.1.0-rc6 --namespace kube-federation-system
```
check deployment
```sh
$ kubectl get all -n kube-federation-system
``` 
#
#### Install helm2 on ARM64 
Install server
```sh
$ wget https://get.helm.sh/helm-v2.16.5-linux-arm64.tar.gz
$ tar -zxvf helm-v2.16.5-linux-arm64.tar.gz
$ mv linux-arm64/helm /usr/local/bin/helm
```
Install client from rebuilt image
```sh
$ helm init --tiller-image=jessestuart/tiller:v2.16.5-arm64
```
check helm
```sh
$ kubectl get pod -n kube-system
```
kubectl create -f helm-rbac.yml
```yml
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
```sh
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
``` 
delete tiller
```sh 
$ kubectl delete deployment tiller-deploy --namespace kube-system
``` 
# 

### Rebuild on arm64 env

Install GOLANG:
```sh
$ wget https://dl.google.com/go/go1.14.1.linux-arm64.tar.gz
$ tar -C /usr/local -xzf go1.14.1.linux-arm64.tar.gz
$ export PATH=$PATH:/usr/local/go/bin
``` 
code build
```sh
$ git clone https://github.com/kubernetes-sigs/kubefed.git
$ cd kubefed
$ git checkout v0.1.0-rc6
$ make build .
``` 	
use kubefedctl
```sh 
$ cd /kubefed/bin
$ ./kubefedctl
$ chmod u+x kubefedctl-linux-arm64
$ sudo cp kubefedctl-linux-arm64 /usr/local/bin
``` 	
Build docker image:
```sh
$ cd /kubefed
$ mv /kubefed/images/kubefed/Dockerfile . 
$ vi Dockerfile
	<COPY /hyperfed .> ---> <COPY bin/hyperfed .>
$ docker build -t kevin7674/kubefed:latest .
$ docker push kevin7674/kubefed:latest
```


