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
kubectl delete -f nginx.yaml

or

kubectl delete pods firstrun secondrun
```

### Deployment
yaml file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.29.1-alpine-slim
        ports:
        - containerPort: 80
```

command
```
kubectl create deployment nginx --image=nginx:1.29.1-alpine-slim
```

```
kubectl get deploy,rs,po -l app=nginx

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           93s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-58dd7d557   1         1         1       93s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-58dd7d557-h4wbl   1/1     Running   0          93s
```

If we want to scale the deployment up to three replicas
```
kubectl scale deploy nginx --replicas=3'

deployment.apps/nginx scaled
```

Now, we get update
```
kubectl get deploy,rs,po -l app=nginx  
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/3     3            3           6m46s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-58dd7d557   3         3         3       6m46s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-58dd7d557-g76g5   1/1     Running   0          22s
pod/nginx-58dd7d557-gc6h7   1/1     Running   0          22s
pod/nginx-58dd7d557-h4wbl   1/1     Running   0          6m46s
```

Remember the name of replica set - nginx-58dd7d557 

```
kubectl describe deploy nginx

Name:                   nginx
Namespace:              default
CreationTimestamp:      Tue, 30 Sep 2025 20:54:56 +0800
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:         nginx:1.29.1-alpine-slim
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-58dd7d557 (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  8m59s  deployment-controller  Scaled up replica set nginx-58dd7d557 from 0 to 1
  Normal  ScalingReplicaSet  2m35s  deployment-controller  Scaled up replica set nginx-58dd7d557 from 1 to 3
```

#### Rollback
Check history
```
kubectl rollout history deploy nginx

deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
```

If we add option `--revision 1`, we can get more details
```
deployment.apps/nginx with revision #1
Pod Template:
  Labels:	app=nginx
	pod-template-hash=58dd7d557
  Containers:
   nginx:
    Image:	nginx:1.29.1-alpine-slim
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
  Node-Selectors:	<none>
  Tolerations:	<none>
```

we can update the image version, for example:
```
kubectl set image deployment nginx nginx=nginx:1.16-alpine  

deployment.apps/nginx image updated 
```

Then, use `kubectl rollout history deploy nginx` again, we will have 2 revisions.
```
kubectl rollout history deploy nginx --revision 2

deployment.apps/nginx with revision #2
Pod Template:
  Labels:	app=nginx
	pod-template-hash=6f5c45796b
  Containers:
   nginx:
    Image:	nginx:1.16-alpine
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
  Node-Selectors:	<none>
  Tolerations:	<none>
```

To rollback revision 1
```
kubectl rollout undo deployment nginx --to-revision=1  

deployment.apps/nginx rolled back
```

### Different between Daemonset and Deployment
For example:
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-agent
  labels:
    k8s-app: fluentd-agent
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-agent
  template:
    metadata:
      labels:
        k8s-app: fluentd-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-agent  
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
      terminationGracePeriodSeconds: 30
```

It set `DaemonSet`, every nodes will have a pod to run. It is not need to apply replicas in yaml file.  
Since the pod number is defined by worker node.  
e.g Log Collector is suitable for it.

In deployment, you need to set replicas, kubernetes will help to find suitable worker node to implement the pods.  
If 1 pod is died, it helps to start a new one in  another node.