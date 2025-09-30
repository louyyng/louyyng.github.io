---
title: 'Practice - How to Run Applications with Pods'
date: 2025-09-30
permalink: /posts/run-app-pods/
tags:
  - infra
---

To run app with pods, first, we prepare a yaml file
```
vi nginx.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.20.2
    ports:
    - containerPort: 80 
```

After that, we can apply it:  
```
kubectl apply -f nginx.yaml

pod/nginx created   
```

We can check the status now:  
```
kubectl get pods
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     ContainerCreating   0          18s
```

To check more such as IP address, we can use  
```
kubectl get pods -o wide 
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          39s   10.244.74.1   minibox-m03   <none>           <none>
```

```
kubectl run firstrun --image=nginx
pod/firstrun created

kubectl get pod -o wide
NAME       READY   STATUS              RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
firstrun   0/1     ContainerCreating   0          8s    <none>        minibox-m02   <none>           <none>
nginx      1/1     Running             0          77s   10.244.74.1   minibox-m03   <none>           <none>
```

We can also copy the yaml file to second yaml file
```
kubectl run firstrun --image=nginxx --port=88 --dry-run=client -o yaml > secondrun.yaml

cat secondrun.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: secondrun
  name: secondrun
spec:
  containers:
  - image: nginx
    name: secondrun
    ports:
    - containerPort: 88
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

kubectl apply -f secondrun.yaml                                          
pod/secondrun created
kubectl get pods -o wide

NAME        READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
firstrun    1/1     Running   0          2m37s   10.244.99.1   minibox-m02   <none>           <none>
nginx       1/1     Running   0          3m46s   10.244.74.1   minibox-m03   <none>           <none>
secondrun   1/1     Running   0          8s      10.244.99.2   minibox-m02   <none>           <none>
```
We can get a new one.  

If we want to delete them, we can:
```
kubectl delete -y nginx.yaml

or

kubectl delete pods firstrun secondrun
```