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

## Step-10: Run a debugger Pod based on alpine image

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
## This mode is essential when you do not want to risk scaling down very quickly.

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

```t
i.e., the algorithm will:

    gather recommendations for 600 seconds (default: 300 seconds)
    pick the largest one
    scale down no more than 5 pods per minute

Example for CurReplicas = 10 and HPA controller cycle once per a minute:

    First 9 minutes the algorithm will do nothing except gathering recommendations. Let's imagine that we have the following recommendations

    recommendations = [10, 9, 8, 9, 9, 8, 9, 8, 9]

    On the 10th minute, we'll add one more recommendation (let it me 8):

    recommendations = [10, 9, 8, 9, 9, 8, 9, 8, 9, 8]

    Now the algorithm picks the largest one 10. Hence it will not change number of replicas

    On the 11th minute, we'll add one more recommendation (let it be 7) and removes the first one to keep the same amount of recommendations:

    recommendations = [9, 8, 9, 9, 8, 9, 8, 9, 8, 7]

    The algorithm picks the largest value 9 and changes the number of replicas 10 -> 9
```

## Step-13: Exec a command in Pod debugger for create a continuesly request on service

```t
kubectl exec -it debuger --namespace dev -- sh
/ # while sleep 0.01 ; do wget -q -O- http://10.105.199.59 ; done
```

## Step-14: Now run the following command again to see the live consume resource for php-apache service 

```t
kubectl top pod php-apache-5d54745f55-gvrln
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-5d54745f55-gvrln   1m           8Mi
```
