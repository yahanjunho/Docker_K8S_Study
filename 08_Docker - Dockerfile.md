

# Docker File Instruction

[TOC]



참고 : 모든 컨테이너 및 이미지 삭제

```bash
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images -aq)
```



## 예제1:ubuntu

![Docker엔지니어링](./img/docker-stages.png "RubberDuck")

Docker에서 이미지를 생성하는 방법은 3가지가 있습니다.

- Dockerfile로 Build하기
- Container로부터 Commit하기
- Image로부터 Tag하기

이중 가장 일반적이고 CI/CD로 활용되어지는 방법은 Dockerfile로 빌드해 이미지를 생성하는 방법입니다.

Dockerfile은 사용자가 이미지를 생성하기위해 순차적으로 작성한 명령어로 된 Text문서이며, *docker build* 명령어로 Dockerfile과 Context를 통해 이미지를 생성합니다.

```dockerfile
# mkdir -p /lab/dockerfile
# cd /lab/dockerfile
```

index.php파일을 생성합니다.

```
# gedit /lab/dockerfile/index.php
```

```
<?php
  echo gethostname();
  echo "\n";
?>
```

dockerfile을 생성합니다.

```
# gedit /lab/dockerfile/dockerfile
```

```dockerfile
FROM php:5-apache
ADD index.php /var/www/html/index.php
RUN chmod a+rx index.php
EXPOSE 80
```

이미지 빌드

```bash
# docker build --tag myphp:1.0 .
# docker images
```

- `-t, --tag` : 저장소 이름, 이미지 이름, 태그를 설정합니다. <저장소 이름>/<이미지 이름>:<태그> 형식
- `.` : Build Context

컨테이너 실행

```
docker run -d -p 8080:80 myphp:1.0
```

--name : 컨테이너 명 지정

-d : 컨테이너를 백그라운드로 실행

-it : 터미널모드로 실행

-p 호스트와 컨테이너 포트를 연결해 노출

--rm 실행 후 컨테이너 자동삭제



실행되고 있는 컨테이너 목록 출력

```
docker ps
```



## FROM

Base image를 지정하는 Instruction으로 지정된 Images를 Docker Hub와 같은 Registry에서 Pull합니다. Base image를 지정할때는 `ubuntu:18.04` 처럼 Image명과 버젼까지 지정해주는것이 좋습니다.

### Syntax

```dockerfile
FROM <image> [AS <name>]
```

```dockerfile
FROM <image>[:<tag>] [AS <name>]
```

```dockerfile
FROM <image>[@<digest>] [AS <name>]
```

### Example

```
# cd /dockerfile
# gedit dockerfile
```

```dockerfile
FROM ubuntu:18.04

# Container에서 실행할 명령
CMD ["/bin/echo", "hello docker"]
```

```
# docker build -t localhost:5000/myununtu:v2 .
# docker run localhost:5000/myununtu:v2
```



## LABEL

`LABEL` instruction는 Lavel로 지정한 문자열을 이미지에 메타데이터로 추가합니다. 

`LABEL` 은 key-value 쌍으로 작성하며 , space를 포함시키기 위해서는 따옴표(`""`)를 이어쓰기를 위해서는 백슬래쉬(`\`)를 사용하면 됩니다. 

### Syntax

```dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

### Example

```
# cd /dockerfile
# gedit dockerfile
```

```dockerfile
FROM ubuntu:18.04

LABEL multi.label1="value1" multi.label2="value2" other="value3"
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
      
# Container에서 실행할 명령
CMD ["/bin/echo", "hello docker"]
```

```
# docker build -t localhost:5000/myununtu:v2 .
# docker run -it localhost:5000/myununtu:v2 /bin/bash
```

이렇게 추가된 Lavel은 컨테이너 실행시 `docker inspect` 명령어로 내용을 확인 할 수 있습니다.

다만 주의할 것은  Base 이미지에 포함된 Lavel 값이 상속된다는 점이고, 만약 같은 Lavel값이 존재한다면 가장 최근에 적용된 Lavel값이 우선합니다.

```dockerfile
"Labels": {
    "com.example.vendor": "ACME Incorporated"
    "com.example.label-with-value": "foo",
    "version": "1.0",
    "description": "This text illustrates that label-values can span multiple lines.",
    "multi.label1": "value1",
    "multi.label2": "value2",
    "other": "value3"
},
```



## RUN

 `RUN` instruction는 Base Image위에 필요한 패키지를 설치할 때 사용하는 명령어입니다.. `Run`명령어가 수행된 이후에는 새로운 Layer가 Base이미지 위에 생성됩니다.

### Syntax

shell form : command 로 입력받은 명령어는 쉘에서 수행되며 디폴트로 리눅스에서는 /bin/sh -c 이 윈도우에서는 cmd /S /C 가 사용됩니다.

```dockerfile
RUN <command>
```

exec form

```dockerfile
RUN ["executable", "param1", "param2"]
```

### Example

`RUN` instruction은 현재생성된 이미지의 위의 새로운 레이어를 생성하여 그 안에서 명령어를 수행하고 그 결과를 레이어에 commit합니다. 따라서 \ (backslash)를 사용하여 RUN을 수행한 것과 여러개의 RUN을 수행한 것은 결과는 동일하나 생성된 레이어의 갯수가 차이가 있게 됩니다.

```dockerfile
FROM ubuntu:14.04
RUN apt-get update
RUN apt-get install -y package-a
RUN apt-get install -y package-b
RUN apt-get install -y package-c
```

```dockerfile
FROM ubuntu:14.04
RUN apt-get update && apt-get install -y \
        package-a \
        package-b \
        package-c
```



## CMD

 `CMD` instruction은 Docker Container가 시작할때 실행 할 커맨드를 지정하는 지시자이며 아래와 같은 특징과 기능을 제공합니다.

- **CMD의 주용도는 컨테이너를 실행할 때 사용할 default 명령어를 설정**하는 것입니다.`docker run` 실행 시 실행할 커맨드를 주지 않으면 CMD로 지정한 default 명령이 실행됩니다. 
- `ENTRYPOINT`의 파라미터를 설정할 수도 있습니다. 
-  `RUN` instruction 과 기능은 비슷하지만 차이점은 `CMD`는 docker image를 빌드할때 실행되는 것이 아니라 docker container가 시작될때 실행됩니다. 주로 docker image로 빌드된 application을 실행할때 사용됩니다.

### Syntax

`CMD` instruction 는 아래와 같이 3가지 포맷을 지원합니다.

- `CMD ["executable","param1","param2"]` (*exec* form, this is the preferred form)
- `CMD ["param1","param2"]` (as *default parameters to ENTRYPOINT*)
- `CMD command param1 param2` (*shell* form)

### Example

```dockerfile
FROM ubuntu
...
CMD ["apache2","-DFOREGROUND"]
```

```dockerfile
FROM ubuntu
...
CMD ["python"]
```

```dockerfile
FROM ubuntu
CMD echo "This is a test."
```

```dockerfile
FROM ubuntu
CMD ["/usr/bin/wc","--help"]
```



## ENTRYPOINT

Docker image가 실행될때 기본 command를 지정합니다. `CMD`와 비슷하지만 `CMD`는 override 가 가능하지만 `ENTRYPOINT` 는 override 할 수 없습니다. 대신에 `docker run` 커맨드로 추가하는 커맨드들은 `ENTRYPOINT` instruction에 지정된 커맨드에 옵션으로 추가됩니다.

### Syntax

`ENTRYPOINT` instruction 는 아래와 같이 2가지 포맷을 지원합니다.

- `ENTRYPOINT ["executable", "param1", "param2"]` (*exec* form, preferred)
- `ENTRYPOINT command param1 param2` (*shell* form)

### Example

```dockerfile
FROM ubuntu
ENTRYPOINT ["/bin/echo", "Hello world"]
```



### ENTRYPOINT & CMD Instruction 예제

```dockerfile
FROM centos
ENTRYPOINT ["/bin/echo", "Hello world"]
CMD ["world"]
```

```bash
# gedit dockerfile

# docker build --tag localhost:5000/hello:1.1 .

컨테이너를 실행에 어떤 내용이 출력되는지 확인해봅시다.
# docker run -it localhost:5000/hello:1.1
# docker run -it localhost:5000/hello:1.1 cmdtest
```



## EXPOSE

`EXPOSE` 명령은 Container가 Runtime시 수신대기할 포트를 지정하는 명령어입니다. 포트번호 및 TCP 또는 UDP를 지정할 수 있으며 프로토콜이 지정되지 않을 경우 기본값은 TCP입니다.

주의해야할 점은 EXPOSE 명령어 자체가 실제로 포트를 열지 않는다는 점입니다. EXPOSE 명령은 이미지를 만드는 사람과 이미지를 사용하는 사람이 포트 및 프로토콜 규약을 명시해놓은 문서기능을 하는 것이고 실제 Container의 포트는 docker run -p 옵션으로 Container를 실행하는 시점에 -p 옵션으로 지정된 포트가 개방됩니다.

### Syntax

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

### Example

```Dockerfile
EXPOSE 80/udp
```

```Dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```



## ENV

`ENV`명령어는 환경변수 `<key>` 를 `<value>`로 설정하는 지시자입니다. 쉽게 일반 프로그래밍에서 변수를 지정하는 것과 동일하다고 생각하면 됩니다. 

`ENV`를 사용하여 셋팅된 환경변수는 Container Runtime시에도 셋팅된 변수가 유지됩니다.

### Syntax

```dockerfile
ENV <key> <value>
ENV <key>=<value> ...
```

### Example

```dockerfile
FROM busybox
ENV FOO /bar
WORKDIR ${FOO}   # WORKDIR /bar
COPY . /quux # COPY all to /quux
```



## COPY

호스트의 Context 내의 파일 또는 디렉터리들을 Container의 파일시스템으로 복사하는 명령어입니다. 또한 `--chown` 옵션으로 파일 및 디렉터리에 대한 유저와 그룹을 지정할 수 있습니다.

### Syntax

- `COPY [--chown=<user>:<group>] <src>... <dest>`
- `COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]` (this form is required for paths containing whitespace)

### Example

```dockerfile
COPY . /bar/             # adds all files and directories
COPY test.jar /bin/test/ # adds a test.jar file
COPY hom* /mydir/        # adds all files starting with "hom"
COPY hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"
COPY --chown=55:mygroup files* /somedir/
COPY --chown=bin files* /somedir/
COPY --chown=1 files* /somedir/
COPY --chown=10:11 files* /somedir/
```

`COPY --chown`에 의해 주어진 유저명과 그룹명으로 지정되지 않은 모든 파일 및 디렉터리들은 Container안에서 UID, GID 0번으로 생성됩니다.



## ADD

ADD는 Syntax 및 기능면에서 COPY와 유사 URL을 지정해 파일을 복사 할 수 있고 압축파일(tar)을 을 자동으로 풀어서 복사한다는 점에서 차이가 있습니다.

### Syntax

- `ADD [--chown=<user>:<group>] <src>... <dest>`
- `ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]`(this form is required for paths containing whitespace)

### Example

```dockerfile
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```



## USER

Docker로 Container를 수행할때 Container 실행 유저는 Docker의 기본설정으로 root계정입니다.

```bash
root@ubuntu:/# docker run ubuntu whoami
root
```

USER 명령어는 이러한 Container안에서 명령을 실행 할 유저명과 유저그룹을 설정하는 명령어이며, Dockerfile 안에서 USER명령어를 셋팅한 이후의 RUN, CMD, ENTRYPONT 명령어에 적용됩니다.

### Syntax

```dockerfile
USER <user>[:<group>] or
USER <UID>[:<GID>]
```

### Example

```dockerfile
# 시스템 그룹 및 유저로 postgres를 추가한다. 
RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres

# Run the rest of the commands as the `postgres` user
USER postgres
```



## WORKDIR

`WORKDIR`명령어는 `WORKDIR`명령어에 뒤따르는`WORKDIR`, `RUN`, `CMD`, `ENTRYPOINT`, `COPY` , `ADD`명령어의 작업디렉터리를 지정하는 명령어입니다. 쉽게 말해 리눅스 cd명령어와 비슷하다고 생각하면 됩니다.

### Syntax

```dockerfile
WORKDIR /path/to/workdir
```

### Example

```dockerfile
WORKDIR /a   # 작업디렉터리를 /a로 이동합니다
WORKDIR b    # /a에서 하위의 b디렉터리로 이동합니다.
WORKDIR c    # /a/b에서 하위의 c디렉터리로 이동합니다.
RUN pwd      # RUN명령어가 실행되는 디렉터리는 WORKDIR로 이동한 /a/b/c가 됩니다.
```

주의할 점은, Linux cd명령어와는 다르게 만약 `WORKDIR`로 지정한 디렉터리가 존재하지 않을 경우, 그 디렉터리를 Container 안에 생성한다는 점에서 다릅니다.

아래와 같이 `ENV`로 지정한 환경변수와 함께 쓰일 수 있다는 점도 참고바랍니다.

```dockerfile
ENV DIRPATH /path
ENV DIRNAME dname
WORKDIR $DIRPATH/$DIRNAME
RUN pwd                   # 출력결과 /path/dname
```



## VOLUME

### Syntax

```dockerfile
VOLUME ["/data"]
```

`VOLUME` 은 컨테이너에서 생성된 데이터를 영구보존하고자 할때 사용되는 명령어로 컨테이너가 실행될때 Volume 명령어로 선언된 Container 안의 지점을 Container 밖의 특정지점과 자동으로 마운트되어 컨테이너가 실행됩니다.

이때 Host에 생성되는 경로는 /var/lib/docker/volumes/{volume_name}이고, volume_name은 Docker에서 생성하는 해쉬값으로 생성이됩니다.

### Example

```bash
# cd /dockerfile
# gedit dockerfile
```

```dockerfile
FROM ubuntu
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
CMD ["/bin/bash"]
```

```bash
# docker build -t localhost/volumetest:v1 .
root@ubuntu:/dockerfile# docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
localhost/volumetest   v1                  0e537c84a0d2        12 seconds ago      88.1MB
```

생성한 이미지를 실행합니다.

```bash
# docker run -it localhost/volumetest:v1 /bin/bash;
```

볼륨을 조회해 보면 신규로 볼륨을 조회해보면 신규로 한개가 생긴것을 확인할 수 있고 생성된 볼륨디렉터리로 들어가면 dockerfile에서 생성한 file을 확인 할 수 있습니다.

```bash
root@ubuntu:/dockerfile# docker volume ls
DRIVER              VOLUME NAME
local               fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9
root@ubuntu:/dockerfile# cd /var/lib/docker/volumes && ls
fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9  metadata.db

root@ubuntu:/var/lib/docker/volumes# cd fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9
root@ubuntu:/var/lib/docker/volumes/fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9# ls
_data
root@ubuntu:/var/lib/docker/volumes/fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9# cd _data/
root@ubuntu:/var/lib/docker/volumes/fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9/_data# ls
greeting

```

이렇게 dockerfile에서만 Volume을 명시해두고 docker실행시 디폴트로 마운트하게되면, docker에서 자동으로 volume을 생성해주지만, Hash값으로 생성하기 때문에 관리하기가 쉽지 않습니다.

따라서 앞서 배운 내용을 바탕으로 Volume을 사전에 미리 생성해두고, 실행시 dockerfile 안에 명시된 경로로 마운트해보도록하겠습니다.

```bash
root@ubuntu:/# docker volume create volume2
volume2
root@ubuntu:/# docker volume ls
DRIVER              VOLUME NAME
local               fbfdbb0b0236c712bd8f2034aa699e0a23a6449915fad90cc0ae07e7f570b0a9
local               volume2
```

Volume을 컨테이너 기동시 마운트하면 이 디렉터리가 컨테이너의 지정된 디렉터리로 마운트됩니다.

```bash
# docker run -it --name volumetest --mount source=volume2,target=/myvol localhost/volumetest:v1 /bin/bash
root@a52260c673a7:/# cd /myvol && ls
greeting
root@a52260c673a7:/myvol# exit

root@ubuntu:/# cd /var/lib/docker/volumes/volume2/_data/ && ls
greeting
```



## 혼동하지 말자

```
COPY vs ADD
```

```
RUN vs CMD
```

```
ENTRYPOINT & CMD
```





## 실습

아래 3번의 Dockerfile에 주석부분에 해당되는 Dockerfile을 완성하고, 이미지를 만든 후 실행해 봅시다.



1. java 애플리케이션을 빌드 할 디렉터리를 생성합니다.

   ```bash
   # mkdir /lab/dockerfile
   # cd /lab/dockerfile
   ```

2. 간단한 java 애플리케이션을 하나 코딩합니다.

   ```bash
   # gedit /lab/dockerfile/HelloDocker.java
   public class HelloDocker {
   	public static void main(String[] args) {
   		System.out.println("Hello Docker!!!");
   	}
   }
   ```

3. dockerfile을 생성합니다.

   ```bash
   # gedit /lab/dockerfile/dockerfile
   ```

   ```dockerfile
   # Base이미지는 java:8 입니다.
   
   
   # 빌드디렉터리의 모든 파일들은 컨테이너 /home/user/hello 위치로 복사합니다.
   
   
   # 컨테이너 안의 위치를 /home/user/hello로 이동합니다.
   
   
   # javac HelloDocker.java 명령어로 자바파일을 컴파일합니다.
   
   
   # 컨테이너가 시작되면 java HelloDocker을 실행합니다.(디폴트 옵션으로)
   
   ```

4. 이미지를 생성합니다.

   ```
   docker build -t localhost:5000/hellodocker:v1 .
   ```

5. 빌드한 이미지를 싫행합니다.

   ```
   docker run localhost:5000/hellodocker:v1
   ```

   