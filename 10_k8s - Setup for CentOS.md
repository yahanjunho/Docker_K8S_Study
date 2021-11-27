



*Outline*

[TOC]

# Bootstrapping Kubernetes Clusters

Kubeadm을 사용하여 Kubernetes 클러스터를 구성하는 방법에 대해 설명합니다. Kubernetes는 아래 그림과 같이 Worker Node에 명령을 내리는 Master Node와 실제 Container가 구동되는 Worker Node로 구성되어 있습니다.

![](./img/kubernetes-worker-node.png)

아래 설치과정에서 주의해야 할것은 

- 가이드에서는 Master Node 1대와 Work Node 1대를 가정해서 설치를 가이드합니다.(총 VM 2대)
- Installing kubeadm 부분은 Kubernetes를 설치하는 VM중 Master로 사용할 VM과 Worker로 사용할 VM 모두에서 실행해야하고, 
- Creating a single master cluster 부분은 주로 Master Node에서 수행하고 마지막에 Worker와 Master를 연결하는 부분에서만 Worker Node에서 수행하면 됩니다.
- 해당 설치내용은 Kubernetes 1.13 버전에 대한 설치내용입니다.



## Before you begin

Kubernetes를 설치하기 전  아래 내용을 확인해주시기 바랍니다.

- CentOS 7.4
- 2 GB or more of RAM per machine
- 2 CPUs or more
- Full network connectivity between all machines in the cluster (public or private network is fine)
  - Virtual Box를 사용한다면 클러스터를 구성할 VM이 NAT Network로 구성되어 있어야 합니다.
- Unique hostname, MAC address, and product_uuid for every node. See [here](https://kubernetes.io/docs/setup/independent/install-kubeadm/#verify-the-mac-address-and-product-uuid-are-unique-for-every-node) for more details.
- Certain ports are open on your machines. See [here](https://kubernetes.io/docs/setup/independent/install-kubeadm/#check-required-ports) for more details.
- Swap disabled. You **MUST** disable swap in order for the kubelet to work properly



# **Pre-Installation Steps On Both Master & Slave**

아래의 설치 단계는 Master, Slave 모든 VM에서 수행해야되는 내용입니다.



## 설치계정

k8s는 root계정으로 설치 및 구성됩니다.

root 계정으로 로그인합니다.



SELinux 비활성화

```
# setenforce 0
# sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```



방화벽 비활성화

```
# systemctl stop firewalld
# systemctl disable firewalld
```



### br_netfilter Kernel Module 활성화

```
# modprobe br_netfilter
# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```



## swap disable

Kubernetes 클러스터 초기화시 Host의 swap이 활성화 되어 있으면 클러스터 구성에 실패하게됩니다.  swap은 하드 디스크의 일부 용량을 메모리로 활용하는 것으로, Kubernetes는 배포시 Host의 자원(cpu, ram)을 근간으로 적절한 곳에 배포하는데 이때 기본설정으로 swap된 공간에 배포하지 않습니다.

swap을 disable 합니다.

```
swapoff -a
```

reboot 후에도 swap이 disable되도록 etc/fstab파일을 수정합니다.

```
sed -i '/swap/s/^/#/' /etc/fstab
```



## kernel parameters  설정

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```





## Installing kubeadm, kubelet and kubectl

kubernetes 저장소를 시스템에 추가합니다.

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

kubeadm, kubelet, kubectl을 설치합니다.

```
yum install -y kubelet-1.13.0
yum install -y kubeadm-1.13.0
yum install -y kubectl-1.13.0
```

docker 및 kubelet 서비스를 재시작합니다.

```
systemctl enable --now kubelet
systemctl enable --now docker
systemctl daemon-reload
systemctl restart kubelet
```

# Creating a single master cluster with kubeadm

이번 장은 kubeadm을 이용하여 Kubernetes 클러스터를 구성하는 방법에 대한 내용입니다. 구성된 VM중 **Master VM에서만 이번장의 내용을 수행하면됩니다.**

## Initializing your master

```
# kubeadm init --pod-network-cidr=10.32.0.0/24 --service-cidr=172.168.1.0/24
```

수행결과로써 아래와 같은 내용이 출력되어야만 합니다.

주의할 것은 수행결과로 출력되는 내용중 `kubeadm join 10.0.2.21:6443 --token poz7u...` 부분은 Worker Node와 Master Node를 연결할 때 사용되는 token값입니다. 

```bash
[init] using Kubernetes version: v1.12.2
[preflight] running pre-flight checks
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [172.168.0.1 10.0.2.21]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [master localhost] and IPs [127.0.0.1 ::1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [master localhost] and IPs [10.0.2.21 127.0.0.1 ::1]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[certificates] Generated sa key and public key.
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests" 
[init] this might take a minute or longer if the control plane images have to be pulled
[apiclient] All control plane components are healthy after 25.501379 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.12" in namespace kube-system with the configuration for the kubelets in the cluster
[markmaster] Marking the node master as master by adding the label "node-role.kubernetes.io/master=''"
[markmaster] Marking the node master as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "master" as an annotation
[bootstraptoken] using token: poz7uz.9ni2ee7hi11jbuy6
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.0.2.21:6443 --token poz7uz.9ni2ee7hi11jbuy6 --discovery-token-ca-cert-hash sha256:fdcaf9e670c96754ef429a1364f86b4b664ee5899cc0b97fd6d048e45c7164fb
```



## 인증서 셋팅

접속하는 터미널의 인증서를 셋팅하는 과정으로 일반유저냐 Root계정이냐에 따라 설정이 다르므로 주의합니다. 여기서는 Root유저로 설정했기 때문에 아래 명령어를 수행합니다.

```bash
# export KUBECONFIG=/etc/kubernetes/admin.conf
```



## Installing a pod network add-on

Pod간의 통신을 위해 Pod network add-on 모듈을 설치합니다.

```bash
# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```



## Joining your nodes

실제 Container가 배포되는 VM(worker node)에서 Master Node로의 연결을 설정하는 과정입니다.

Node VM에 로그인하여 터미널을 열어 아래 절차를 수행합니다.

### 계정

아래와 같이 root 계정으로 로그인합니다.

```bash
$ su root
Password: root
root@node0:/home/ubuntu# cd /
root@node0:/# 
```

### Master Joining

위의 Initializing your master 의 수행결과로 출력된 token 값으로 Node VM을 Master VM에 연결합니다.

```bash
# kubeadm join 10.0.2.21:6443 --token poz7uz.9ni2ee7hi11jbuy6 --discovery-token-ca-cert-hash sha256:fdcaf9e670c96754ef429a1364f86b4b664ee5899cc0b97fd6d048e45c7164fb
```

