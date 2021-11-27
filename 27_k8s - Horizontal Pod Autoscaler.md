# k8s - Horizontal Pod Autoscaler



*Outline*

[TOC]

## Overview

![HPA](./img/horizontal-pod-autoscaler.svg)

Horizontal Pod Autoscaler는 Pod의 CPU 이용율을 관찰/측정하여 Deployment나 ReplicaSet에서 자동으로 Pods의 수를 Scale해줍니다.



Horizontal Pod Autoscaler controller는 아래와 같이 원하는 기준 값과 측정된 값의 비율로 그 값을 계산합니다.

```
desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
```

예를들어 원하는 값이 `100m`이고 측정된 값이 `200m`이라면 Replicas는 두배가 될 것이고, 반대로 측정된 값이 `50m`이라면 0.5의 값을 얻을 수 ceil에 의해 1의 값으로 계산됩니다.

Horizontal Pod Autoscaler를 사용하기 위해서는 몇가지 작업이 필요한데,

우선 Pod에 CPU 요청 및 제한이 정의되어 있어야 합니다

```
      containers:
      - name: myphp
        image: myphp:1.0
        imagePullPolicy: Never
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

다음으로는 HorizontalPodAutoscaler를 정의한 yaml파일을 생성해주어야합니다.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myphp-hpa
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myphp
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```



## 실습

HPA 실습을 위해 자원을 모니터링해주는 metrics-server를 설치합니다.

```bash
# mkdir -p /lab/metrics
# cd /lab/metrics
# git clone https://github.com/kubernetes-incubator/metrics-server.git
# cd /lab/metrics/metrics-server/deploy/1.8+
# gedit /lab/metrics/metrics-server/deploy/1.8+/metrics-server-deployment.yaml
```

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.1
        args:
        - --kubelet-insecure-tls
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      hostAliases:
      - hostnames:
        - master
        ip: 10.0.2.5
      - hostnames:
        - node
        ip: 10.0.2.6
```

```
# kubectl apply -f /lab/metrics/metrics-server/deploy/1.8+
```

metrics-server 동작 확인

```bash
# kubectl top node
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master   160m         8%     2553Mi          66%       
node1    46m          2%     2289Mi          59%       
node2    42m          2%     2141Mi          55% 
```



아래 예제에서 web에 제곱근을 계산하는 로직을 넣어놓은 Pod를 생성하고, NodePort로 해당 서비스를 노출시켜 VM에서 계속 호출해 부하를 주는 방법으로 HPA를 실습해봅니다.



## Creating a docker image

### Creating a index.php

```bash
# mkdir -p /lab/hpa/dockerfile
# gedit /lab/hpa/dockerfile/index.php
```

```php
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo gethostname();
  echo "\n";
?>
```



### Creating a dockerfile

```
# gedit /lab/hpa/dockerfile/dockerfile
```

```dockerfile
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+rx index.php
EXPOSE 80
```



### Creating a docker images

```bash
# cd /lab/hpa/dockerfile
# docker build -t myphp:3.0 .
```



## Creating a Pod & Replica

Creating a Pod & Exposing pods to the cluster

```bash
# mkdir -p /lab/hpa
# gedit /lab/hpa/myphp-deployment.yaml
```

Deployment yaml파일 생성

CPU자원을 100m으로 요청하는 Deployment를 생성합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myphp
spec:
  selector:
    matchLabels:
      run: myphp
  replicas: 1
  template:
    metadata:
      labels:
        run: myphp
    spec:
      containers:
      - name: myphp
        image: myphp:3.0
        imagePullPolicy: Never
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"
```



## Creating a Service - NodePort

Creating a Service - Node Port

```
# gedit /lab/hpa/myphp-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myphp
  labels:
    run: myphp
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
  selector:
    run: myphp
```

Deployment & Service 생성

```bash
kubectl apply -f /lab/hpa/
```

결과확인

```bash
# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          11h   <none>
myphp     NodePort    10.98.249.250   <none>        8080:31380/TCP   17s   run=myphp

# curl localhost:32016
```



## Creating a Horizontal Pod Autoscaler

Pod 상세정보 확인

```
# gedit /lab/hpa/myphp-hpa.yaml
```

```bash
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myphp-hpa
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myphp
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Deployment & Service & Horizontal Pod Autoscaler 적용

```bash
# kubectl apply -f /lab/hpa/
```

적용확인

```bash
root@master:/# kubectl get hpa -o wide
NAME        REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
myphp-hpa   Deployment/myphp   0%/50%    1         10        1          5m48s


# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          11h   <none>
myphp     NodePort    10.98.249.250   <none>        8080:31380/TCP   17s   run=myphp

# curl localhost:32601
```



## Horizontal Pod Autoscaler 부하테스트

watch  옵션으로 hpa에 대한 모니터링을 시작합니다.

```bash
root@master:/# kubectl get pod -w
NAME                     READY   STATUS    RESTARTS   AGE
curl-5cc7b478b6-vwtmj    1/1     Running   2          170m
myphp-6dff57d978-jrxtl   1/1     Running   0          10m

```

부하테스트

Node VM에서 curl 명령으로 Node에 노출된 컨테이너를 계속 호출합니다.

```bash
# while true; do curl localhost:32601; done

```

서비스 및 deployment 삭제

```bash
# kubectl delete deployment myphp
# kubectl delete svc myphp
```

