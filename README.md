## kubefed for itrix-edge project
Support ARM64 architecture edge device

#

#### Docker images for arm64 architecture.
```
  docker pull kevin7674/kubefed:latest
```
#
#### 	kubefedctl
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
#### 	deploy
```
	helm install /root/kubefed/charts/federation-v2  --version=v0.0.4 --namespace kube-federation-system 
``` 
#
  
  
