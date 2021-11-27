# Docker Running Command 



![Docker엔지니어링](./img/docker-stages.png "RubberDuck")

참고 : 모든 컨테이너 및 이미지 삭제

```
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images -aq)
```

## docker run command

Docker로 컨테이너를 실행할때는 일반적으로 2가지 경우로 생각해볼 수 있습니다.

먼저, Docker로 컨테이너를 실행한 후 컨테이너 안으로 진입하는 형태와

Docker로 컨테이너를 데몬으로 실행하는 형태입니다.

### docker run command - it

아래 명령어는 ubuntu Container를 Container안에 진입하여 인터렉티브 모드로 /bin/bash를 실행하는 명령어입니다.

```bash
root@:~# docker run -it ubuntu /bin/bash

명령어          내용
---------      -----------------------------------
docker        
run            Container를 실행해라(이미지가 없으면 pull한 후 실행)
-it            인터렉티브 터미널 모드로
-d             데몬모드로 실행
--name         실행하는 Container의 이름 지정
-p             포트포워딩으로 Container 외부와 내부의 포트 매핑
-e             컨테이너에 환경변수 전달
ubuntu         ubuntu 이미지 (docker.io/library/ubuntu:latest)
/bin/bash      Container시작후 /bin/bash를 실행해라



Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
32802c0cfa4d: Pull complete 
da1315cffa03: Pull complete 
fa83472a3562: Pull complete 
f85999a86bef: Pull complete 
Digest: sha256:6d0e0c26489e33f5a6f0020edface2727db9489744ecc9b4f50c7fa671f23c49
Status: Downloaded newer image for ubuntu:latest
root@b3c25b397b22:/# 

```

위 명령어를 실행할때, Docker에서 아래와 같은 일이 순차적으로 발생합니다.

1. ubuntu 이미지가 로컬에 없다면, Docker는 Registry로부터 이미지를 다운로드 받습니다.

2. 다운로드 받은 이미지로 신규 Container를 생성합니다.

3. 생성된 Container에 read-write filesystem을 마지막 Layer로 할당합니다. 이를 통해 실행중인 Container는 로컬파일시스템의 파일과 디렉터리를 생성하거나 수정할 수 있습니다.

4. Default Network Interface인 브릿지 네트워크를 생성합니다.

5. Container를 시작하고 /bin/bash를 실행합니다.

6. exit를 입력하여 /bin/bash를 종료하면 컨테이너는 중지되지만 삭제되지 않고, 추후에 삭제하거나 다시 재시작 할 수 있습니다.

   - docker ps -a 명령어로 모든 container를 조회해봅니다.

   - docker images 명령어로 이미지를 조회해봅니다.



### Example docker run command - d

아래 명령어는 nginx Container를 데몬 모드로 실행하는 명령어입니다.

```bash
root@:~# docker run -d --name my-nginx -p 8080:80 nginx

명령어          내용
---------      -----------------------------------
docker        
run            Container를 실행해라(이미지가 없으면 pull한 후 실행)
-it            인터렉티브 터미널 모드로
-d             데몬모드로 실행
--name         실행하는 Container의 이름 지정
-p             포트포워딩으로 Container 외부와 내부의 포트 매핑
-e             컨테이너에 환경변수 전달
ubuntu         ubuntu 이미지 (docker.io/library/ubuntu:latest)
/bin/bash      Container시작후 /bin/bash를 실행해라


Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
a5a6f2f73cd8: Pull complete 
1ba02017c4b2: Pull complete 
33b176c904de: Pull complete 
Digest: sha256:5d32f60db294b5deb55d078cd4feb410ad88e6fe77500c87d3970eca97f54dba
Status: Downloaded newer image for nginx:latest
2a242940c872c37d5125ecf316304c9d0a268534e74e7d0fa6cfd78862761d6f

```

위 명령어를 실행할때, Docker에서 아래와 같은 일 순차적으로 발생합니다.

1. nginx이미지가 로컬에 없다면, Docker는 Registry로부터 이미지를 다운로드 받습니다.
2. 다운로드 받은 이미지로 신규 Container를 생성합니다.
3. 생성된 Container에 read-write filesystem을 마지막 Layer로 할당합니다. 이를 통해 실행중인 Container는 로컬파일시스템의 파일과 디렉터리를 생성하거나 수정할 수 있습니다.
4. Default Network Interface인 브릿지 네트워크를 생성합니다.
5. Container를 데본형식으로 시작하고 컨테이너에 80번 포트가 노출됩니다.

아래 명령어로 nginx가 정상적으로 동작중인지 확인해 봅니다.

```bash
root@docker:/# curl localhost:8080

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



다음 실습을 위해 실행되고 있는  컨테이너를 삭제합니다.

```
docker rm -f $(docker ps -aq)
```





## 기본 명령어

### 이미지 받아오기

Docker Hub : https://hub.docker.com/

```bash
# 이미지 받아오기
# docker pull nginx

# 이미지 조회
# docker images

# 이미지 삭제
# docker rmi <이미지명>
```

 

### 컨테이너 실행하기

```bash
# 데몬으로 실행하기
# docker run -d -p 8080:80 --name mynginx nginx
```

```bash
# tty 형태로 실행하기
# docker run -it --name myubuntu ubuntu /bin/bash
```

`-p` 포트 지정

 `-d` 백그라운드로 실행 

 `--name`컨테이너 이름지정

 `-it` 컨테이너 안에서 tty로 실행

컨테이너 조회

```bash
# docker ps -a
```

컨네이서 상세조회

```bash
# docker inspect <컨테이너ID>
```

컨테이너 삭제

```bash
# docker rm -f <컨테이너명>
```



### 실행된 컨테이너에 접속하기

```bash
# 먼저 ubuntu를 터미널 모드로 실행합니다.
docker run -it --name myubuntu ubuntu /bin/bash
# 컨테이너 밖으로 빠져나옵니다.
Ctrl + p + q
# 아래 명령어로 컨테이너가 실행됨을 확인합니다.
docker ps


# 실행된 컨테이너 안으로 접속합니다.
docker attach myubuntu
# 컨테이너 밖으로 빠져나옵니다.
Ctrl + p + q
```



### 실행된 컨테이너에 커맨드 실행하기

```bash
# docker exec myubuntu echo "hello world"
```



### 컨테이너 중지/재시작하기

```bash
root@:/# docker stop myubuntu
root@:/# docker start myubuntu
root@:/# docker restart myubuntu
```

 

### 이미지 Tag하기

```bash
root@:/# docker tag ubuntu localhost:5000/myubuntu:1.0

root@:/# docker images
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
ubuntu                    latest              93fd78260bd1        39 hours ago        86.2MB
localhost:5000/myubuntu   1.0                 93fd78260bd1        39 hours ago        86.2MB
```





## 기타주요 commands

| Command                                                      | Description                                                  |
| ------------------------------------------------------------ | :----------------------------------------------------------- |
| [docker attach](https://docs.docker.com/engine/reference/commandline/attach/) | Attach local standard input, output, and error streams to a running container |
| [docker build](https://docs.docker.com/engine/reference/commandline/build/) | Build an image from a Dockerfile                             |
| [docker commit](https://docs.docker.com/engine/reference/commandline/commit/) | Create a new image from a container’s changes                |
| [docker container](https://docs.docker.com/engine/reference/commandline/container/) | Manage containers                                            |
| [docker exec](https://docs.docker.com/engine/reference/commandline/exec/) | Run a command in a running container                         |
| [docker export](https://docs.docker.com/engine/reference/commandline/export/) | Export a container’s filesystem as a tar archive             |
| [docker images](https://docs.docker.com/engine/reference/commandline/images/) | List images                                                  |
| [docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/) | Return low-level information on Docker objects               |
| [docker kill](https://docs.docker.com/engine/reference/commandline/kill/) | Kill one or more running containers                          |
| [docker login](https://docs.docker.com/engine/reference/commandline/login/) | Log in to a Docker registry                                  |
| [docker logout](https://docs.docker.com/engine/reference/commandline/logout/) | Log out from a Docker registry                               |
| [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) | List containers                                              |
| [docker pull](https://docs.docker.com/engine/reference/commandline/pull/) | Pull an image or a repository from a registry                |
| [docker push](https://docs.docker.com/engine/reference/commandline/push/) | Push an image or a repository to a registry                  |
| [docker restart](https://docs.docker.com/engine/reference/commandline/restart/) | Restart one or more containers                               |
| [docker rm](https://docs.docker.com/engine/reference/commandline/rm/) | Remove one or more containers                                |
| [docker rmi](https://docs.docker.com/engine/reference/commandline/rmi/) | Remove one or more images                                    |
| [docker run](https://docs.docker.com/engine/reference/commandline/run/) | Run a command in a new container                             |
| [docker search](https://docs.docker.com/engine/reference/commandline/search/) | Search the Docker Hub for images                             |
| [docker start](https://docs.docker.com/engine/reference/commandline/start/) | Start one or more stopped containers                         |
| [docker stop](https://docs.docker.com/engine/reference/commandline/stop/) | Stop one or more running containers                          |
| [docker tag](https://docs.docker.com/engine/reference/commandline/tag/) | Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE        |

