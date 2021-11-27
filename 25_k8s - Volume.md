# Volume



*Outline*

[TOC]

## Overview

![1543905873812](./img/PersistentVolumeClaim.png)

컨테이너는 기본적으로 상태가 없는 상태에서 구동됩니다. 즉, 컨테이너가 실행되서 상태를 가져도 컨테이너가 종료되면 컨테이너에서 생성되었던 모든 데이터는 사라진다는 뜻입니다. 이것은 Kubernetes상에서 자유롭게 Pod를 복제하고 배포하는데 있어  커다란 장점입니다.

하지만, 로그 및 데이터베이스와 같은 애플리케이션은 종료되더라도 데이터가 유지되어야만 하는 특징이 있습니다. 이런 데이터의 상태를 유지할 수 있도록 사용하는 것이 Volume입니다.

Kubernetes에서는 Docker와는 다르게 Pod 단위로 [Volume](https://kubernetes.io/docs/concepts/storage/volumes/)을 관리하며,  Life Cycle과 제공되는 디스크 Type 따라 다양한 옵션과 종류가 존재합니다. 이 장에서는 Kubernetes의 영구 Volume체계인 PersistentVolume(PV)와 PersistentVolumeClaim(PVC)에 대해 살펴봅니다.



## Volume 종류

Volume은 제공되는 디스크 Type에 따라 여러가지 종류의 Volume Type을 지원합니다.

| Temp     | Local    | Network        |
| -------- | -------- | -------------- |
| emptyDir | hostPath | NFS, AWS EBS.. |

- emptyDir은 동일한 Pod내에서 사용가능한 임시 Volume으로 Pod가 삭제되면 Volume의 데이터도 같이 삭제됩니다.
- hostPath는 특정 노드에 mount된 파일시스템을 Volume으로 사용합니다. 본 교육에서는 로컬 VM환경이므로 HostPath Type으로 실습하는 예제입니다.
- Network : 네트워크 또는 클라우드 상에 존재하는 storage를 Volume으로 사용합니다.



## Volume Provisioning 에 따른 분류

Kubernetes에서는 자동으로 Volume을 생성할 수 있느냐 없느냐에 따라 Static Volume Provisioning과 DynamicVolume Provisoning으로 구분할 수 있는데, 

- Static Volume Provisioning : 관리자가 수동으로 Kubernetes에서 Persistent Volume을 생성

- DynamicVolume Provisoning : 개발자의 요청에 의해 자동으로 Kubernetes에서 Persistent Volume을 생성


즉, DynamicVolume Provisoning은 개발자나 클라우드 사용자가 손쉽게 Volum사용요청(PVC) 만으로 Volume을 사용할 수 있습니다.



## Persistent Volume(PV) & Persistent Volume Claim(PVC)

  쿠버네티스에서 볼륨을 사용하는 구조는 스토리지를 매핑시킨 PersistentVolume(PV)과 그 PV를 Pod에서 사용할때 PV로 Volume내역을 요청하는 PersistentVolumeClaim(PVC) 2단계로 분리되어 있습니다.

PV는 Storage와 mount되어 있으므로 쉽게말해 볼륨 또는 Storage 자체를 의미합니다. 

PVC는 개발자 또는 클라우드 사용자가 PV에게 Volume을 할당해달라는 요청입니다. 사용하고 싶은 용량과 각종 Volume에 관한 설정값을 정해서 요청하게 됩니다.

## Persistent Volume

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

### Phase

Volume은 아래와 같은 상태를 가집니다. 해당 상태는 PV이 배포되고, kubectl get pv 명령을 통해 조회할 수 있습니다.

- Available – PV가 생성되고 PVC가 바인딩되기 전 상태
- Bound – PVC가 PV에 바인된 상태
- Released – PVC가 삭제된 상태
- Failed – 실패상태



### Access Modes

- ReadWriteOnce – 단일 노드에 의한 읽기-쓰기로 볼륨이 마운트될 수 있습니다.
- ReadOnlyMany – 여러 노드에 의한 읽기 전용으로 볼륨이 마운트될 수 있습니다.
- ReadWriteMany – 여러 노드에 의한 읽기-쓰기로 볼륨이 마운트될 수 있습니다.



### Reclaim Policy

PV와 바인딩된 PVC를 삭제했을때 PV는 Released 상태로 변경됩니다. 이때 PV와 연결된 디스크의 파일들을 어떻게 처리할 것인지에 대한 설정으로 아래와 같이 설정할 수 있습니다.

- Retain – PVC가 삭제되어도 PV 및 데이터 유지(다시 쓰기 위해서는 PV를 삭제하고 재생성해야함)
- Recycle – PV는 유지하고 파일만 삭제 (`rm -rf /thevolume/*`)
- Delete – 볼륨 및 데이터 삭제



## PersistentVolumeClaims

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```



## 실습

Create a Secret for MySQL Password

실습에서 사용할 mysql 패스워드를 secret으로 생성해두겠습니다.

```
# kubectl create secret generic mysql-pass --from-literal=password=pwd
```



### Volume

#### Create PersistentVolumes

PersistentVolume yaml파일 생성

```
# mkdir -p /lab/volume
# gedit /lab/volume/wp-pv-volume.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wp-pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 3Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: "/data/mysql"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node
```



#### Create PersistentVolumeClaims

PersistentVolumeClaims yaml 파일 생성

```
# gedit /lab/volume/mysql-pv-claim.yaml
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      type: local
```



### Backend

#### Creating a Mysql Pod

Mysql Pod yaml파일 생성

```
# gedit /lab/volume/mysql-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        imagePullPolicy: Always
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```



#### Creating a Mysql Service

Mysql Service yaml 파일 생성

```
gedit /lab/volume/mysql-svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```



### FrontEnd

#### Creating a Wordpress Pod

```
gedit /lab/volume/wp-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        imagePullPolicy: Always
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
```



#### Creating a Wordpress Service

```bash
gedit /lab/volume/wp-svc.yaml
```

Service yaml파일 생성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
  selector:
    app: wordpress
    tier: frontend
```



일괄로 pv, pvc, pod, svc를 생성합니다.

```
kubectl apply -f /lab/volume/
```



결과확인

```bash
root@master:/lab/dashboard# kubectl get svc
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes        ClusterIP   172.168.1.1     <none>        443/TCP          4h56m
wordpress         NodePort    172.168.1.177   <none>        8080:32382/TCP   10m
wordpress-mysql   ClusterIP   None            <none>        3306/TCP         10m

브라우저에서 http://localhost:32382 로 접속해봅니다.

node VM에서 mysql데이터가 생성되었는지 확인해봅니다.
root@node:/data/mysql# ll
total 110616
drwxr-xr-x 5 vboxadd vboxsf     4096 12월  5 15:39 ./
drwxr-xr-x 3 root    root       4096 12월  5 15:12 ../
-rw-rw---- 1 vboxadd vboxsf       56 12월  5 15:38 auto.cnf
-rw-rw---- 1 vboxadd vboxsf 12582912 12월  5 15:38 ibdata1
-rw-rw---- 1 vboxadd vboxsf 50331648 12월  5 15:38 ib_logfile0
-rw-rw---- 1 vboxadd vboxsf 50331648 12월  5 15:38 ib_logfile1
drwx------ 2 vboxadd vboxsf     4096 12월  5 15:38 mysql/
drwx------ 2 vboxadd vboxsf     4096 12월  5 15:38 performance_schema/
drwx------ 2 vboxadd vboxsf     4096 12월  5 15:39 wordpress/
```



# kubernetes dashboard 

이번장에서 앞서 실습해본 내용을 kubernetes에서 제공되는 dashboard에서 내용을 살펴보도록하겠습니다.

![](./img/ui-dashboard.png)

## kubernetes dashboard 설치

```bash
# kubectl apply -f /lab/dashboard/
```



## kubernetes dashboard 접속

```bash
# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   172.168.1.10    <none>        53/UDP,53/TCP   4h52m
kubernetes-dashboard   NodePort    172.168.1.120   <none>        443:31372/TCP   2m2s
metrics-server         ClusterIP   172.168.1.246   <none>        443/TCP         97m

위에 kubernetes-dashboard 로 생성된 svc를 로컬에서 접속하면 됩니다.
브라우저에서 https://localhost:31372 접속해봅니다.
```

