# Docker Network

참고 : 모든 컨테이너 및 이미지 삭제

```
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images -aq)
```



## Overview

Container 를 생성할때 지원되는 network 방식은 크게 4가지입니다. 이번 장에서는 Docker Document 사이트의 예제를 통해 bridge Network구조와 동작에 대해 알아봅니다. 

### Network driver summary

- **bridge networks** are best when you need multiple containers to **communicate on the same Docker host**.

  ![docker bridge network](./img/bridge1.png "docker bridge network")

- **Host networks** are best when the network stack should not **be isolated from the Docker host**, but you want other aspects of the container to be isolated.

- **Overlay networks** are best when you need **containers running on different Docker hosts** to communicate, or when multiple applications work together using swarm services.

  ![docker bridge network](./img/1nNoIXGkJiDax7l5g5GxH7nTzqrqzN7Y9aBZTaXoQ8Q=.png "docker bridge network")

- **Macvlan networks** are best when you are migrating from a VM setup or need your containers to look like physical hosts on your network, each with a unique MAC address.

- **Third-party network plugins** allow you to integrate Docker with specialized network stacks.

*Outline*



## nginx 실행하기

예제를 실습하기 전에 우리에게 친숙한 nginx를 통해 기본 네트워크를 살펴보겠습니다.

[Docker Hub](https://hub.docker.com/)를에 접속하여 nginx 이미지 검색하여 아래 명령어로 이미지를 Pull받습니다.

``` bash
docker pull nginx
```

Docker Container 구동

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

- `--name` : Container 명 지정
- `-p` : 컨테이너 포트지정

접속확인

```bash
curl http://127.0.0.1:8080

브라우저에서 http://127.0.0.1:8080
```

Container 정보 확인

```bash
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
e2a871a929e8        nginx               "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes        0.0.0.0:8080->80/tcp   my-nginx

# docker container inspect e2a871a929e8
```





## bridge network(default)

![](./img/dockerNat.png)

브라우저에서 http://127.0.0.1:8080 입력시 어떤 과정으로 컨테이너 안으로 접속했는지에 대해 알아봅시다.

1. Host에 8080으로 요청이 들어옵니다.
2. 들어온 요청은 Host에 구성된 NAT 테이블을 통해 HostIP:8080 -> ContainerIP:80으로 변환됩니다.
3. 변환된 ContainerIP:80는 Docker0이라는 가상네트워크 인터페이스가 Subnet 172.17.0.1/16으로 구성되어 있으므로  172.17.0.2의 IP로 접근가능하게 됩니다.



기동된 Container의 네트워크 정보를 살펴보면 위에서 설명한 내용되로 구성되어 있음을 확인 할 수 있습니다.

**NAT정보**

iptables -t nat -L -n로 출력된 내용중 

DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80

부분을 확인할 수 있습니다.

```
# iptables -t nat -L -n
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
```

bridge network정보

docker network inspect bridge 명령을 통해 네트워크 정보를 살펴보면

contaner network는 "Subnet": "172.17.0.1/16", "Gateway": "172.17.0.1" 으로 구성되며, Gateway 역할을 하는 인터페이스 네트워크는 docker0임을 알수 있습니다.

``` bash
# docker network inspect bridge

[
    {
        "Name": "bridge",
        "Id": "f7ab26d71dbd6f557852c7156ae0574bbf62c42f539b50c8ebde0f728a253b6f",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.1/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "3386a527aa08b37ea9232cbcace2d2458d49f44bb05a6b775fba7ddd40d8f92c": {
                "Name": "networktest",
                "EndpointID": "647c12443e91faf0fd508b6edfe59c30b642abb60dfab890b4bdccee38750bc1",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "9001"
        },
        "Labels": {}
    }
]
```





