## kubefed for itrix-edge project
Use kubefed on ARM64 architecture edge device.

kubefed not support K8S 1.16+ now, it only support deployment with helm2 at the moment, especially since helm3 is not officially released yet (still in beta).
#
## requirement
### Use kubefed to federate clusters

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
unjoin cluster form federation
```sh
$ ./kubefedctl unjoin <> --cluster-context <> --host-cluster-context <>
```
### 創建nginx的應用
新增一個Namespace，測試用
```sh
$kubectl create namespace federation
namespace/federation created
```
在剛剛新增的Namespace底下再創建一個FederatedNamespace
```sh
$vim fed-namespace.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: federation
  namespace: federation
spec:
  placement:
    clusters:
    - name: cluster1
    - name: cluster2
```
調度它
```sh
$ kubectl create -f fed-namespace.yaml
federatednamespace.types.kubefed.io/test created
```
檢查
```sh
root@xavier01:~/kubefed/bin# kubectl -n federation get federatednamespaces
NAME         AGE
federation   13m
```
創建應用有幾種形式
1. **水平擴展**

在host集群利用k8s資源創建nginx，然後利用kubefedctl水平擴展到其他的聯邦集群。(不用修改原有的yaml檔)
```sh
$ kubectl create -f https://github.com/itrix-edge/kubefed/blob/master/nginx-deployment.yaml
$ ./kubefedctl federate deployments.apps nginx-deployment -n federation --host-cluster-context=cluster1
```
分別到cluster1，cluster2集群查看，可以看到namespace federation底下都有nginx的pod。
```sh
root@xavier01:~/kubefed/bin# kubectl --context cluster1 -n federation get all
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           4s

root@xavier01:~/kubefed/bin# kubectl --context cluster2 -n federation get all
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           4s
```
部署Deployment 之後，可以通過federateddeployment.types.kubefed.io查看部署狀態。
```sh
$ kubectl -n federation get federateddeployment.types.kubefed.io
$ kubectl -n federation describe federateddeployment.types.kubefed.io/nginx-deployment
```
刪除聯邦的服務
```sh
root@xavier01:~/kubefed/bin# kubectl -n federation delete federateddeployment.types.kubefed.io/nginx-deployment
federateddeployment.types.kubefed.io "nginx-deployment" deleted
```
2. **placement**

利用聯邦placement,創建nginx到指定的cluster2。
```sh
$ kubectl create -f https://github.com/itrix-edge/kubefed/blob/master/nginx-placement-sample.yaml
```
分別到cluster1，cluster2集群查看，可看到 : 
```sh
cluster1 沒有nginx的pod
cluster2 有nginx的pod
```
3. **overrides**

利用聯邦overrides,在不同集群分配不同數量且版本不同的pods。 
```sh
$ kubectl create -f https://github.com/itrix-edge/kubefed/blob/master/nginx-overrides-sample.yaml
```
分別到cluster1，cluster2集群查看，可看到 :
```sh
cluster1 有3個pod 版本是nginx:1.14.2
cluster2 有5個pod 版本是nginx:1.17.0-alpine
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


