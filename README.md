## kubefed for itrix-edge project
Use kubefed on ARM64 architecture edge device.

kubefed not support K8S 1.16+ now, it only support deployment with helm2 at the moment, especially since helm3 is not officially released yet (still in beta).
#
## requirement
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
- cluster:
    certificate-authority-data: <>
    server: <>
  name: cluster2
contexts:
- context:
    cluster: cluster1
    user: cluster1
  name: cluster1
 - context:
    cluster: cluster2
    user: cluster2
  name: cluster2
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
- name: cluster2
  user:
    client-certificate-data: <>
    client-key-data: <>
```
check cluster contexts
```sh
root@xavier01:~# kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          cluster1                      cluster1     cluster1
          cluster2                      cluster2     cluster2
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```
join cluster to federation
```sh
$ ./kubefedctl join <cluster1> --cluster-context <cluster1> --host-cluster-context <cluster1>
$ ./kubefedctl join <cluster2> --cluster-context <cluster2> --host-cluster-context <cluster1>
$ ./kubefedctl join <kubernetes> --cluster-context <kubernetes-admin@kubernetes> --host-cluster-context <cluster1>
```
check federation cluster
```sh
root@xavier01:~# kubectl -n kube-federation-system get kubefedclusters
NAME       READY   AGE
cluster1     True    5d
cluster2     True    8m
kubernetes   True    1m
```
create testing namespace
```sh
$ kubectl create ns <federation>
$ ./kubefedctl federate ns <federation> --host-cluster-context=<cluster1>
```
### 創建nginx的應用
創建應用有兩種形式
1. 在host集群利用k8s資源創建nginx，然後聯邦到其他的集群
```sh
kubectl apply -f nginx-deployment.yaml
kubefedctl federate deployments.apps nginx -n federation --host-cluster-context=cluster1
```
2. 利用聯邦資源創建,在不同集群分配不同數量的pods 
```sh
kubectl apply -f https://github.com/itrix-edge/kubefed/blob/master/federated-nginx-deployment.yaml

分別到cluster1，cluster2集群查看pods的數量
cluster1 
NAME READY STATUS RESTARTS AGE 
test-deployment-7c867888f-2h4nr 1/1 Running 0 7m24s 
test-deployment-7c867888f-lrmg6 1/1 Running 0 7m24s 
test-deployment-7c867888f-lttql 1/1 Running 0 7m24s

Cluster2
NAME READY STATUS RESTARTS AGE 
test-deployment-59769b7d7d-48sw5 1/1 Running 0 7m43s 
test-deployment-59769b7d7d-86r9k 1/1 Running 0 7m43s 
test-deployment-59769b7d7d-c9lfz 1/1 Running 0 7m43s 
test-deployment-59769b7d7d-shsnr 1/1 Running 0 7m43s 
test-deployment-59769b7d7d-sr6wd 1/1 Running 0 7m43s

可以看到cluster1有3個pod，cluster2有5個pod，利用聯邦override。
```
unjoin cluster form federation
```sh
$ ./kubefedctl unjoin <> --cluster-context <> --host-cluster-context <>
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

### build kubefed on arm64 env

Install GOLANG:
```sh
$ wget https://dl.google.com/go/go1.14.1.linux-arm64.tar.gz
$ tar -C /usr/local -xzf go1.14.1.linux-arm64.tar.gz
$ export PATH=$PATH:/usr/local/go/bin
``` 
build code
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
build docker image:
```sh
$ cd /kubefed
$ mv /kubefed/images/kubefed/Dockerfile . 
$ vi Dockerfile
	<COPY /hyperfed .> ---> <COPY bin/hyperfed .>
$ docker build -t kevin7674/kubefed:latest .
$ docker push kevin7674/kubefed:latest
```


