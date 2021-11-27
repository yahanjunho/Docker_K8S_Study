

# k8s - DaemonSet

*Outline*

[TOC]

## Overview

![HPA](https://image.slidesharecdn.com/k8sv1-160316100710/95/new-features-of-kubernetes-v120-beta-14-638.jpg?cb=1458123134)

**DaemonSet은 모든(혹은 몇몇의 지정된) Node들에 특정 Pod 한개를 유지시켜주는 Controller입니다.** 

동작방식은 Node들이 Cluster내에 추가될때, DaemonSet으로 정의된 Pod가 자동으로 생성됩니다. 마차가지로 Node가 Cluster내에서 제거된다면, DaemonSet을 삭제함으로써 생성된 Pod가 제거됩니다.

이러한 특징때문에 DaemonSet은 Cluster내의 logging 및 자원모니터링과 같은 곳에 효과적으로 사용되어 질 수 있습니다.

(K8S cluster에서의 Logging 및 모니터링은 Container가 모든 Node들에 분포되어 배포가 이루어지기때문에 모든 Node에 하나 이상의 Logging할 수 있는 Container가 필요하기 때문입니다.)



## Writing a DaemonSet Spec

DaemonSet의 Spec은 앞서서의 Deployment, Pod의 Spec과 그게 다르지 않습니다. 주요한 정의에 대해서는 아래 예시의 주석을 참고하시기 바랍니다.

### Create a DaemonSet

```bash
# mkdir -p /var/log
# mkdir -p /var/lib/docker/containers
# mkdir -p /lab/daemonset
# gedit /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch # DaemonSet의 Name
  namespace: kube-system      # DaemonSet이 배포되는 namespace
  labels:
    k8s-app: fluentd-logging
spec:
  # DaemonSet이 관리해야할 Pod를 지정하는 Label selector입니다.
  # 아래의 Label과 Pod의 Lavel이 매칭되는 Pod가 관리대상입니다.
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      # Pod의 Lavel(위의 Lavel selector에 매칭됩니다.)
      labels:
        name: fluentd-elasticsearch
    # DaemonSet은 Pod를 관리하는 Object이므로 하위는 Pod의 Spec과 동일합니다.
    spec:
      tolerations: # tolerations에 해당되는 Node를 포함하여 스케쥴링합니다.
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

작성한 daemonset yaml파일을 쿠버네티스 클러스터에 배포합니다.

```bash
# kubectl apply -f /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
```

배포된 내용을 get, describe를 통해 확인해봅니다.

확인 결과 

```bash
root@master:/lab/daemonset# kubectl get daemonset -n kube-system
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3       3            3           <none>          85m
```

```bash
root@master:/lab/daemonset# kubectl get pod -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE
fluentd-elasticsearch-b7pw7      1/1     Running   0          13m   10.38.0.1   master   <none>
fluentd-elasticsearch-kt5cm      1/1     Running   0          13m   10.32.0.2   node2    <none>
fluentd-elasticsearch-qfmbt      1/1     Running   0          13m   10.40.0.3   node1    <none>
```

```bash
# kubectl describe pod fluentd-elasticsearch-cgmnk -n kube-system
Name:               fluentd-elasticsearch-cgmnk
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               node1/10.0.2.22
Start Time:         Tue, 05 Mar 2019 09:53:14 +0900
Labels:             controller-revision-hash=7474489b7b
                    name=fluentd-elasticsearch
                    pod-template-generation=1
Annotations:        <none>
Status:             Running
IP:                 10.40.0.3
Controlled By:      DaemonSet/fluentd-elasticsearch
Containers:
  fluentd-elasticsearch:
    Container ID:   docker://e12d36fe66143f9d14635b5337cddda24b4a7db0a389b9d95f361fdaf0a85437
    Image:          k8s.gcr.io/fluentd-elasticsearch:1.20
    Image ID:       docker-pullable://k8s.gcr.io/fluentd-elasticsearch@sha256:de60e9048b3b79c4e53e66dc3b8ee58b4be2a88395a4b33407f8bbadc9548dfb
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 05 Mar 2019 09:54:49 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     200Mi
    Environment:  <none>
    Mounts:
      /var/lib/docker/containers from varlibdockercontainers (ro)
      /var/log from varlog (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-llfmz (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  varlog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:  
  varlibdockercontainers:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker/containers
    HostPathType:  
  default-token-llfmz:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-llfmz
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
```



### Running Pods on Only Some Nodes

DaemonSet은 Pod를 관리한다는 점에서 Deployment와 유사하지만, 모든 노드나 특정 노드들을 지정해서 스케쥴링할 수 있다는 점에서 차이가 있습니다. 따라서 DaemonSet에서는 아래에서 소개하고 있는 특정 Node에 Pod를 스케쥴링 하는 방법에 대해 알아둘 필요가 있습니다.

[node selector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)와  [node affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)를 통해 특정 Node에만 Pod를 한정하여 배포하는 과정을 알아보고 실습해봅니다.

-  `.spec.template.spec.nodeSelector` : Selector로 지정된 Lavel과 Node에 지정된 Lavel이 매칭되는 Node에만 Pod를 배포합니다. 
- `.spec.template.spec.affinity` : affinity로 지정된 Node명과 일치된 Node명에만 Pod가 배포됩니다.
- `taint & toleration` : Taints가 지정된 Node에는 tolerations이 매칭되는 Pod만 배포할 수 있게됩니다.

nodeSelector와 affinity를 정의하지 않으면 DaemonSet controller에서는 디폴트 설정으로 모든 Node에 Pod를 배포합니다.

#### .spec.template.spec.nodeSelector

nodeSelector는 nodeSelector로 지정된 Lavel로 Node에 정의된 Lavel이 매칭되는 Node를 찾아서 스케쥴링 하는 기능으로 위의 예제에서 nodeSelector를 추가하고 node1에 Lavel을 추가하여 Pod가 node1 노드에만 배포되는 과정을 실습해보겠습니다.

1. Node에 app: dev라는 레이블을 추가합니다.

   ```bash
   root@master:/lab/daemonset# kubectl get node
   NAME     STATUS   ROLES    AGE   VERSION
   master   Ready    master   78d   v1.12.2
   node1    Ready    <none>   78d   v1.12.2
   node2    Ready    <none>   78d   v1.12.2
   
   
   root@master:/lab/daemonset# kubectl edit node node1
   
   # Please edit the object below. Lines beginning with a '#' will be ignored,
   # and an empty file will abort the edit. If an error occurs while saving this file will be
   # reopened with the relevant failures.
   #
   apiVersion: v1
   kind: Node
   metadata:
     annotations:
       kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
       node.alpha.kubernetes.io/ttl: "0"
       volumes.kubernetes.io/controller-managed-attach-detach: "true"
     creationTimestamp: 2018-12-16T23:50:44Z
     labels:
       beta.kubernetes.io/arch: amd64
       beta.kubernetes.io/os: linux
       kubernetes.io/hostname: node1
       app: dev
     name: node1
   
   ```

   

2. DaemonSet Controller에 app:dev을 추가합니다.

   ```bash
   root@master:/lab/daemonset# kubectl get daemonset -n kube-system
   NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   fluentd-elasticsearch   3         3         3       3            3           <none>          94m
   kube-proxy              3         3         3       3            3           <none>          78d
   weave-net               3         3         3       3            3           <none>          78d
   
   root@master:/lab/daemonset# kubectl edit daemonset fluentd-elasticsearch -n kube-system
   
   apiVersion: extensions/v1beta1
   kind: DaemonSet
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"apps/v1","kind":"DaemonSet","metadata":{"annotations":{},"labels":{"k8s-app":"fluentd-logging"},"name":"fluentd-elasticsearch","namespace":"kube-system"},"spec":{"selector":{"matchLabels":{"name":"fluentd-elasticsearch"}},"template":{"metadata":{"labels":{"name":"fluentd-elasticsearch"}},"spec":{"containers":[{"image":"k8s.gcr.io/fluentd-elasticsearch:1.20","name":"fluentd-elasticsearch","resources":{"limits":{"memory":"200Mi"},"requests":{"cpu":"100m","memory":"200Mi"}},"volumeMounts":[{"mountPath":"/var/log","name":"varlog"},{"mountPath":"/var/lib/docker/containers","name":"varlibdockercontainers","readOnly":true}]}],"terminationGracePeriodSeconds":30,"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}],"volumes":[{"hostPath":{"path":"/var/log"},"name":"varlog"},{"hostPath":{"path":"/var/lib/docker/containers"},"name":"varlibdockercontainers"}]}}}}
     creationTimestamp: 2019-03-05T00:53:14Z
     generation: 1
     labels:
       k8s-app: fluentd-logging
     name: fluentd-elasticsearch
     namespace: kube-system
     resourceVersion: "3057"
     selfLink: /apis/extensions/v1beta1/namespaces/kube-system/daemonsets/fluentd-elasticsearch
     uid: 0eafcd79-3ee1-11e9-9e3c-080027b0aab8
   spec:
     revisionHistoryLimit: 10
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     template:
       metadata:
         creationTimestamp: null
         labels:
           name: fluentd-elasticsearch
       spec:
         nodeSelector:
           app: dev
   
   ```

   

3. daemonset, pod를 조회해 결과를 확인해보면  Pod의 갯수가 1개로 줄어들었음을 확인 할 수 있습니다.

   ```bash
   root@master:/lab/daemonset# kubectl get daemonset -n kube-system
   NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   fluentd-elasticsearch   1         1         1       0            1           app=dev         97m
   kube-proxy              3         3         3       3            3           <none>          78d
   weave-net               3         3         3       3            3           <none>          78d
   root@master:/lab/daemonset# kubectl get pod -n kube-system
   NAME                             READY   STATUS    RESTARTS   AGE
   coredns-576cbf47c7-rdxfv         1/1     Running   0          78d
   coredns-576cbf47c7-wp2x6         1/1     Running   0          78d
   etcd-master                      1/1     Running   0          78d
   fluentd-elasticsearch-fhnbf      1/1     Running   0          43s
   kube-apiserver-master            1/1     Running   0          78d
   kube-controller-manager-master   1/1     Running   0          78d
   kube-proxy-8q8qn                 1/1     Running   0          78d
   kube-proxy-9qpxw                 1/1     Running   0          78d
   kube-proxy-ck4xq                 1/1     Running   0          78d
   kube-scheduler-master            1/1     Running   0          78d
   weave-net-j6p5r                  2/2     Running   0          78d
   weave-net-k7b4k                  2/2     Running   0          78d
   weave-net-l5wgs                  2/2     Running   0          78d
   
   ```

4. 이전실습과정을 초기화 하기위해 추가했던 nodeSelector를 삭제합니다.

   ```bash
   # kubectl edit daemonset fluentd-elasticsearch -n kube-system
   ### 아래부분 삭제
         nodeSelector:
           app: dev
   ```

   

#### .spec.template.spec.affinity

affinity또한 nodeSelector와 마찬가지로 affinity로 선언된 Node를 매칭하는 매커니즘으로 동작하지만 nodeSelector와는 다르게 표현식을 사용해 Node의 Lavel과 매칭시킬 수 있는 특징이 있습니다.

예를 들어 아래와 같이 affinity를 정의하면 Node에 정의된 Lavel 중 kubernetes.io/hostname=node1에 매칭되는 Node에 스케쥴링이 이루어지게 됩니다. 자세한 내용은 [node affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)를 참고바랍니다.

```yaml
      affinity: # nodeAffinity를 추가합니다. kubernetes.io/hostname=node1
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - node1
```

그럼 위의 내용데로 예제를 바꾸어서 실습해 보도록하겠습니다.

1. 기존 배포된 daemonSet 삭제

   ```
   root@master:/lab/daemonset# kubectl delete daemonset fluentd-elasticsearch -n kube-system
   daemonset.extensions "fluentd-elasticsearch" deleted
   root@master:/lab/daemonset# kubectl get daemonset -n kube-system
   NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   kube-proxy   3         3         3       3            3           <none>          78d
   weave-net    3         3         3       3            3           <none>          78d
   
   ```

   

2. 위의 예제에서 toleration이 정의된 부분을 추가 후 배포합니다.

   ```
   # gedit /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
   ```

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: fluentd-elasticsearch # DaemonSet의 Name
     namespace: kube-system      # DaemonSet이 배포되는 namespace
     labels:
       k8s-app: fluentd-logging
   spec:
     # DaemonSet이 관리해야할 Pod를 지정하는 Label selector입니다.
     # 아래의 Label과 Pod의 Lavel이 매칭되는 Pod가 관리대상입니다.
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     template:
       metadata:
         # Pod의 Lavel(위의 Lavel selector에 매칭됩니다.)
         labels:
           name: fluentd-elasticsearch
       # DaemonSet은 Pod를 관리하는 Object이므로 하위는 Pod의 Spec과 동일합니다.
       spec:
         affinity: # nodeAffinity를 추가합니다. kubernetes.io/hostname=node1
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname
                   operator: In
                   values:
                   - node1
         containers:
         - name: fluentd-elasticsearch
           image: k8s.gcr.io/fluentd-elasticsearch:1.20
           resources:
             limits:
               memory: 200Mi
             requests:
               cpu: 100m
               memory: 200Mi
           volumeMounts:
           - name: varlog
             mountPath: /var/log
           - name: varlibdockercontainers
             mountPath: /var/lib/docker/containers
             readOnly: true
         terminationGracePeriodSeconds: 30
         volumes:
         - name: varlog
           hostPath:
             path: /var/log
         - name: varlibdockercontainers
           hostPath:
             path: /var/lib/docker/containers
   ```

   ```bash
   # kubectl apply -f /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
   ```

   

3. 배포된 결과를 조회해보면 이전 배포와는 다르게 pod가 node1에만 배포되는 것을 확인 할 수 있습니다.

   ```bash
   root@master:/lab/daemonset# kubectl get pods -n kube-system -o wide
   NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE
   fluentd-elasticsearch-9q8rf      1/1     Running   0          11s   10.40.0.3   node1    <none>
   ```

   

#### taint & toleration

toleration을 가진 Pod들만 taint를 가진 Node를 포함한 모든 노드에 스케쥴링되는 개념으로 taint & toleration로 스케쥴되는 과정은 설명하기 까다로운 부분이 있어, 예시로 대신하겠습니다. 

우선 최초 위의 예제에서 Pod를 조회해보면 모든 노드에 배포되는 것을 확인할 수 있습니다.

```bash
root@master:/lab/daemonset# kubectl get pod -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE
fluentd-elasticsearch-b7pw7      1/1     Running   0          31s   10.38.0.1   master   <none>
fluentd-elasticsearch-kt5cm      1/1     Running   0          31s   10.32.0.2   node2    <none>
fluentd-elasticsearch-qfmbt      1/1     Running   0          31s   10.40.0.3   node1    <none>
```

위의 예제에서는 모든 노드에 DaemonSet이 배포되었지만, 기본적으로 컨테이너는 master node에 배포되지 않도록 설정되어 있습니다. 그 이유는 master node에 taint 가 설정되어 있기 때문입니다. 

```bash
# kubectl describe node master
...
Taints:             node-role.kubernetes.io/master:NoSchedule
...
```

만약 master node에도 컨테이너를 배포하기 위해서는 pod에 master node에 설정된 taint에 해당되는 toleration을 추가해 주어야합니다.

즉 toleration을 가진 Pod들만 taint를 가진 Node를 포함한 모든 노드에 스케쥴링이 가능합니다.

```
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

```bash
# kubectl describe pod fluentd-elasticsearch-b7pw7 -n kube-system
...
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
...
```

그렇타면 반대로 DaemonSet에 정의된 Toleration를 삭제하면 Pod가 어떻게 배포되는 확인해보도록 하겠습니다.

기존에 배포된 daemonSet 을 삭제하고, yaml파일에 정의된 Toleration를 삭제하면 DaemonSet은 Master Node에는 컨테이너를 배포하지 않을 것입니다.

1. 기존 배포된 daemonSet 삭제

   ```bash
   root@master:/lab/daemonset# kubectl get daemonset -n kube-system
   NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   fluentd-elasticsearch   3         3         3       3            3           <none>          4h35m
   kube-proxy              3         3         3       3            3           <none>          78d
   weave-net               3         3         3       3            3           <none>          78d
   root@master:/lab/daemonset# kubectl delete daemonset fluentd-elasticsearch -n kube-system
   daemonset.extensions "fluentd-elasticsearch" deleted
   root@master:/lab/daemonset# kubectl get daemonset -n kube-system
   NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   kube-proxy   3         3         3       3            3           <none>          78d
   weave-net    3         3         3       3            3           <none>          78d
   
   ```

   

2. 위의 예제에서 toleration이 정의된 부분을 삭제 후 배포합니다.

   ```bash
   # gedit /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
   ```

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: fluentd-elasticsearch # DaemonSet의 Name
     namespace: kube-system      # DaemonSet이 배포되는 namespace
     labels:
       k8s-app: fluentd-logging
   spec:
     # DaemonSet이 관리해야할 Pod를 지정하는 Label selector입니다.
     # 아래의 Label과 Pod의 Lavel이 매칭되는 Pod가 관리대상입니다.
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     template:
       metadata:
         # Pod의 Lavel(위의 Lavel selector에 매칭됩니다.)
         labels:
           name: fluentd-elasticsearch
       # DaemonSet은 Pod를 관리하는 Object이므로 하위는 Pod의 Spec과 동일합니다.
       spec:
         containers:
         - name: fluentd-elasticsearch
           image: k8s.gcr.io/fluentd-elasticsearch:1.20
           resources:
             limits:
               memory: 200Mi
             requests:
               cpu: 100m
               memory: 200Mi
           volumeMounts:
           - name: varlog
             mountPath: /var/log
           - name: varlibdockercontainers
             mountPath: /var/lib/docker/containers
             readOnly: true
         terminationGracePeriodSeconds: 30
         volumes:
         - name: varlog
           hostPath:
             path: /var/log
         - name: varlibdockercontainers
           hostPath:
             path: /var/lib/docker/containers
   ```

   ```bash
   # kubectl apply -f /lab/daemonset/fluentd-elasticsearch-daemonset.yaml
   ```

   

3. 배포된 결과를 조회해보면 이전 배포와는 다르게 pod가 master node에는 배포되지 않은 것을 확인 할 수 있습니다.

   ```bash
   root@master:/lab/daemonset# kubectl get pods -n kube-system -o wide
   NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE
   coredns-576cbf47c7-rdxfv         1/1     Running   0          78d   10.40.0.2   node1    <none>
   coredns-576cbf47c7-wp2x6         1/1     Running   0          78d   10.40.0.1   node1    <none>
   etcd-master                      1/1     Running   0          78d   10.0.2.21   master   <none>
   fluentd-elasticsearch-29jsz      1/1     Running   0          97s   10.32.0.2   node2    <none>
   fluentd-elasticsearch-m5d6c      1/1     Running   0          97s   10.40.0.3   node1    <none>
   kube-apiserver-master            1/1     Running   0          78d   10.0.2.21   master   <none>
   kube-controller-manager-master   1/1     Running   0          78d   10.0.2.21   master   <none>
   kube-proxy-8q8qn                 1/1     Running   0          78d   10.0.2.24   node2    <none>
   kube-proxy-9qpxw                 1/1     Running   0          78d   10.0.2.22   node1    <none>
   kube-proxy-ck4xq                 1/1     Running   0          78d   10.0.2.21   master   <none>
   kube-scheduler-master            1/1     Running   0          78d   10.0.2.21   master   <none>
   weave-net-j6p5r                  2/2     Running   0          78d   10.0.2.22   node1    <none>
   weave-net-k7b4k                  2/2     Running   0          78d   10.0.2.24   node2    <none>
   weave-net-l5wgs                  2/2     Running   0          78d   10.0.2.21   master   <none>
   
   ```

   