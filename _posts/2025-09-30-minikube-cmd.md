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
Normally, if using M chip, we use docker as driver. `minikube start --driver=docker`   

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
