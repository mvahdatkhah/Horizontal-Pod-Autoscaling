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

```kubectl apply -f deployment.yaml -f service.yaml -f hpa.yaml
```

## Step-0: Scale Up As Fast As Possible, Scale Down Very Gradually
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
