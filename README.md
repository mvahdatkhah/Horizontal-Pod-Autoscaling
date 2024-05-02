## Scale Up As Fast As Possible, Scale Down Very Gradually

## Step-01: Introduction

- create Deployment
- create Service
- create Horizontal Pod Autoscaling
- scale Up As Fast As Possible, Scale Down Very Gradually

## Step-02: Create a Deployment with the following

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: dev
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```
## Step-03: Create a Service with the following

```yaml
apiVersion: v1
kind: Service
metadata:
    name: php-apache-svc
    namespace: dev
    labels:
      run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

## Step-04: Create an HPA with the following

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2
metadata:
  name: php-apache-hpa
  namespace: dev
spec:
  scaleTargetRef:
    kind: Deployment
    name: php-apache
    apiVersion: apps/v1
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

## Step-05: Apply the Kubernetes Manifest files

```t
kubectl apply -f deployment.yaml -f service.yaml -f hpa.yaml
```

## Step-06: Get the live Pod resource on dev namespace

```t
watch kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
php-apache-5d54745f55-gvrln   1/1     Running   0          5m44s   10.10.45.195   kubenode3   <none>           <none>
```

## Step-07: Get the Deployment resource on dev namespace

```t
kubectl get deployments.apps
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           4m22s
```

## Step-08: Get the Services resource on dev namespace

```t
kubectl get services
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
php-apache-svc   ClusterIP   10.105.199.59   <none>        80/TCP    4m27s
```

## Step-09: Get the Horizontal Pod Autoscaling

```t
kubectl get hpa
NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   0%/60%    1         10        1          5m8s
```

## Step-10: Run a debuger Pod based on alpine image

```t
kubectl run debuger --namespace dev --image=alpine --command sleep infinity
```

```t
kubectl get pod -o wide                                
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
debuger                       1/1     Running   0          5m26s   10.10.45.239   kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Running   0          5m44s   10.10.45.195   kubenode3   <none>           <none>
```

## Step-11: Run the following command to see the live consume resources for php-apache Service

```t
kubectl top pod php-apache-5d54745f55-gvrln
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-5d54745f55-gvrln   1m           8Mi
```

## Step-12: Scale Up As Fast As Possible, Scale Down Very Gradually
This mode is essential when you do not want to risk scaling down very quickly.

```yaml
behavior:
  scaleUp:
    policies:
    - type: Percent
      value: 900
      periodSeconds: 60
  scaleDown:
    policies:
    - type: Pods
      value: 1
      periodSeconds: 600 # (i.e., scale down one pod every 10 min)
```

This behavior has the same scale-up pattern as the previous example. However the behavior for scaling down is also specified. The scaleUp behavior will be fast as explained in the previous example. However the target will scale down by only one pod every 10 minutes.

```t
1000 -> 1000 -> 1000 -> … (7 more min) -> 999
```

## Step-13: Exec a command in Pod debuger for create a continuesly request on service

```t
kubectl exec -it debuger --namespace dev -- sh
/ # while sleep 0.01 ; do wget -q -O- http://10.105.199.59 ; done
```

## Step-14: Now run the following command again to see the live consume resource for php-apache service 

```t
kubectl top pod php-apache-5d54745f55-gvrln
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-5d54745f55-gvrln   114m         11Mi
```

Now see the output of the `kubectl get pod -o wide -w` command

```t
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
debuger                       1/1     Running   0          6m43s   10.10.45.239   kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Running   0          7m1s    10.10.45.195   kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Pending   0          0s      <none>         <none>      <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Pending   0          0s      <none>         kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     ContainerCreating   0          0s      <none>         kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     ContainerCreating   0          1s      <none>         kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   1/1     Running             0          11s     10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-jk6l8   0/1     Pending             0          0s      <none>          <none>      <none>           <none>
php-apache-5d54745f55-jk6l8   0/1     Pending             0          0s      <none>          kubenode2   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Pending             0          0s      <none>          <none>      <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Pending             0          0s      <none>          kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   0/1     ContainerCreating   0          0s      <none>          kubenode2   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     ContainerCreating   0          1s      <none>          kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     ContainerCreating   0          1s      <none>          kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   0/1     ContainerCreating   0          1s      <none>          kubenode2   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     ErrImagePull        0          4s      10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   1/1     Running             0          11s     10.10.35.111    kubenode2   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Pending             0          0s      <none>          <none>      <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Pending             0          1s      <none>          kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     ContainerCreating   0          1s      <none>          kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     ContainerCreating   0          2s      <none>          kubenode1   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     ImagePullBackOff    0          17s     10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     ErrImagePull        0          11s     10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-9rg2h   1/1     Running             0          33s     10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     ImagePullBackOff    0          23s     10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   1/1     Running             0          32s     10.10.205.199   kubenode1   <none>           <none>
```

Again see the output of the `kubectl top pod php-apache-5d54745f55-gvrln` command

```t
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-5d54745f55-gvrln   200m         11Mi
```

Also see the output of the `kubectl get hpa` command

```t
NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   58%/60%   1         10        5          20m
```

After create a continuesly request on service see the output of the `kubectl get pod -o wide | grep -i running` command

```t
debuger                       1/1     Running   0          22m     10.10.45.239    kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   1/1     Running   0          7m36s   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   1/1     Running   0          7m51s   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Running   0          23m     10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   1/1     Running   0          7m36s   10.10.35.111    kubenode2   <none>           <none>
php-apache-5d54745f55-vktqq   1/1     Running   0          7m21s   10.10.205.199   kubenode1   <none>           <none>
```

## Step-15: Now drop the continuesly request on service and see the scale down one pod every 10 min

Now see the output of the `kubectl top pod php-apache-5d54745f55-gvrln` and `kubectl get hpa` commands

```t
kubectl get hpa
NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache-hpa   Deployment/php-apache   0%/60%    1         10        5          39m
```

```t
kubectl top pod php-apache-5d54745f55-gvrln
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-5d54745f55-gvrln   1m           11Mi

```

Now see the output of the `kubectl get pod -o wide -w` command

```t
kubectl get pod -o wide -w
NAME                          READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
debuger                       1/1     Running   0          41m   10.10.45.239    kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   1/1     Running   0          26m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   1/1     Running   0          26m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Running   0          41m   10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   1/1     Running   0          26m   10.10.35.111    kubenode2   <none>           <none>
php-apache-5d54745f55-vktqq   1/1     Running   0          26m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   1/1     Terminating   0          27m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   1/1     Terminating   0          27m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Terminating   0          27m   <none>          kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Terminating   0          27m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Terminating   0          27m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-vktqq   0/1     Terminating   0          27m   10.10.205.199   kubenode1   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Terminating   0          43m   10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   1/1     Terminating   0          27m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   1/1     Terminating   0          27m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-9rg2h   1/1     Terminating   0          27m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   1/1     Terminating   0          43m   10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   1/1     Terminating   0          27m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Terminating   0          27m   <none>          kubenode1   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Terminating   0          27m   <none>          kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Terminating   0          27m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   0/1     Terminating   0          43m   <none>          kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Terminating   0          27m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-9rg2h   0/1     Terminating   0          27m   10.10.45.224    kubenode3   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Terminating   0          27m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Terminating   0          27m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gq8kj   0/1     Terminating   0          27m   10.10.205.239   kubenode1   <none>           <none>
php-apache-5d54745f55-gvrln   0/1     Terminating   0          43m   10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   0/1     Terminating   0          43m   10.10.45.195    kubenode3   <none>           <none>
php-apache-5d54745f55-gvrln   0/1     Terminating   0          43m   10.10.45.195    kubenode3   <none>           <none>
```

However the target will scale down by only one pod every 10 minutes.
Now only one apache Pod run at the moment

Run  the `kubectl get pod -o wide -w` command to see the number of running Pod

```t
kubectl get pod -o wide                                
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
debuger                       1/1     Running   0          46m   10.10.45.239   kubenode3   <none>           <none>
php-apache-5d54745f55-jk6l8   1/1     Running   0          31m   10.10.35.111   kubenode2   <none>           <none>
```

## Step-16: Clean up the manidest files

The simplest method of deleting any resource in Kubernetes is to use the specific manifest file used to create it. With the manifest file on hand, we can use the kubectl delete command with the -f flag.
To delete Kubernetes Resources run the `kubectl delete -f deployment.yaml -f service.yaml -f hpa.yaml` command.

kubectl delete -f deployment.yaml -f service.yaml -f hpa.yaml

```t
deployment.apps "php-apache" deleted
service "php-apache-svc" deleted
horizontalpodautoscaler.autoscaling "php-apache-hpa" deleted
```

And for deleting the debuger Pod run the `kubectl delete pod debuger` command.

kubectl delete pod debuger 

```t
pod "debuger" deleted
```
