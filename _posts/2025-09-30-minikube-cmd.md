---
title: 'Practice - Minikube'
date: 2025-09-30
permalink: /posts/minikube-cmd/
tags:
  - infra
---

# Installation on Mac (Apple Silicon)
```
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

### Basic Cluster Lifecycle
`minikube start`  
Usage: When we start this, it helps to create a simple node of kuberentes.  
Normally, if using M chip, we use docker as driver.   
`minikube start --driver=docker`   

Driver can be virtualbox `minikube start --driver=virtualbox`  

`minikube status`  
Usage: It is used to check the kubernetes status.  

`minikube stop`  
`minikube delete`  

### Custom Cluster Creation  
To create cluster, we can use  
`minikube start` with different options such as   
- `-p <name> or --profile <name>`  
It is for defining cluster name. The default name is minikube  

For example: We want to create a profile call 'minibox' with 3 nodes, and use the driver docker.  
`minikube start --driver=docker -n 3 --container-runtime=containerd --cni=calico -p minibox`  

To check the node, we can use  
```
minikube node list -p minibox  

minibox	192.168.49.2  
minibox-m02	192.168.49.3  
minibox-m03	192.168.49.4  
```

To check the default one, we can use  
```
minikube ip  
```

To check the cluster status, we can use  
```
minikube -p minibox status
```

### How to use command `minikube kubectl`  
If using `minikube kubectl -- get nodes`, we will get the default information.  
To specific profile, we can use  
```
minikube -p minibox kubectl -- get nodes
```

If we just want to use kubectl, first, we need to check the name  
```
kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
          minibox    minibox    minibox    default
*         minikube   minikube   minikube   default
```
  
Then, switch the profile to minibox:
```
kubectl config use-context minibox
```

Finally, we can get nodes  
```
kubectl config use-context minibox
Switched to context "minibox".

kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
minibox       Ready    control-plane   27m   v1.34.0
minibox-m02   Ready    <none>          27m   v1.34.0
minibox-m03   Ready    <none>          27m   v1.34.0
```

### Enable Kubernetes Dashboard
We can enable it through:
```
minikube addons enable metrics-server
minikube addons enable dashboard
```

Then, we can use `minikube addons list` to check it is enabled.

Normally, we can get the url
```
minikube dashboard --url
```

`kubectl proxy` command, kubectl authenticates with the API server on the control plane node and makes services available on the default proxy port 8001.
When `kubectl proxy` is running, we can use `curl http://localhost:8001/` to check the api endpoints
```
{
 "paths": [
   "/api",
   "/api/v1",
   "/apis",
   "/apis/apps",
   ......
   ......
   "/logs",
   "/metrics",
   "/openapi/v2",
   "/version"
 ]
}
```

If we are not using `kubectl proxy`, we need to authenticate to the API Server when sending API requests.  
We can authenticate by providing a Bearer Token when issuing a curl command, or by providing a set of keys and certificates.  

```
export TOKEN=$(kubectl create token default)
kubectl create clusterrole api-access-root --verb=get --non-resource-url=/*
kubectl create clusterrolebinding api-access-root --clusterrole api-access-root --serviceaccount=default:default
```

The above commands:
- Create an access token is for the default ServiceAccount, and grant special permission to access the root directory of the API. 
- The special permission will be set through a Role Based Access Control (RBAC) policy. 
- The special permission is only needed to access the root directory of the API, but not needed to access /api, /apis, or other subdirectories.  

After that, we can 
```
export APISERVER=$(kubectl config view | grep https | cut -f 2- -d ":" | tr -d " ")
```
And find the APIServer URL
```
echo $APISERVER
https://127.0.0.1:54126
https://127.0.0.1:54281
```

Then, we can test the API is applied token or not:
```
curl https://127.0.0.1:54126 --header "Authorization: Bearer $TOKEN" --insecure

{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
    .......
  ]
}
```

If just query with `curl https://127.0.0.1:54126 --insecure`, we will get Forbidden.  

Then, we can query the API   
```
curl https://127.0.0.1:54126/version --header "Authorization: Bearer $TOKEN" --insecure
{
  "major": "1",
  "minor": "34",
  "emulationMajor": "1",
  "emulationMinor": "34",
  "minCompatibilityMajor": "1",
  "minCompatibilityMinor": "33",
  "gitVersion": "v1.34.0",
  "gitCommit": "f28b4c9efbca5c5c0af716d9f2d5702667ee8a45",
  "gitTreeState": "clean",
  "buildDate": "2025-08-27T10:09:04Z",
  "goVersion": "go1.24.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}                                                        
```

If we don't want to use access token, we can extract the client certificate, client key, and certificate authority data from the .kube/config file.  
