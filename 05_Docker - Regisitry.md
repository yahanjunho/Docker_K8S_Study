# 1. Docker Registry
모든 컨테이너 및 이미지 삭제

```
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images -aq)
```



## 1.1 What it is

Docker Registry는 Docker의 이미지 저장소로써 운영환경에 따라
* public으로 운영되는 Docker Hub
* 사내에서 운영되는 Redii
* 로컬서버에 설치해 사용할 수 있는 Private Local Registry
로 구분할 수 있다.



## 1.2 Why use it

* Docker image가 저장되는 위치를 통제 할 수 있습니다.
* Docker image 저장 및 배포에 대해 개발 워크 플로우에 맞춰 pipeline을 구성할 수 있습니다.

![Docker엔지니어링](./img/docker-stages.png "RubberDuck")

# 2 Local Registry 구성
- Local Private Registry 구성하여
- Public Registry에서 image를 pull해 나만의 image로 만들어 Local Private Registry에 Push 하기

Local Registry 구성 및 실행

```
docker run -d -p 5000:5000 --restart=always --name registry registry:2.5
```

## 2.1 Docker Hub에서 Local registry로 이미지 옴기기

1. Docker Hub에서 이미지  Pull하기.

   ```
   root@node0:/# docker pull ubuntu:16.04
   
   root@node0:/# docker images
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   ubuntu              16.04               a51debf7e1eb        2 days ago          116MB
   registry            2.5                 96ca477b7e56        2 months ago        37.8MB
   ```

2. 이미지 Tag하고 확인하기

   ```
   docker tag ubuntu:16.04 localhost:5000/my-ubuntu:1.0
   ```

   ```
   docker images
   ```

   > ```
   > Docker Image Tag규칙
   > Docker에서 Push또는 Pull시 Tag된 명명규칙을 참조하기 때문에 반드시 알아두어야 한다.
   > > 도메인명/레파지토리명:버전정보
   > ```

3. 이미지 Push하기

   ```
   docker push localhost:5000/my-ubuntu:1.0
   ```

4. 생성한 이미지 삭제하기

   ```
   # docker images
   # docker rmi -f localhost:5000/my-ubuntu:1.0
   ```

5. Local registry에서 이미지 Pull하기.

   ```
   root@node0:/# docker pull localhost:5000/my-ubuntu:1.0
   
   1.0: Pulling from my-ubuntu
   Digest: sha256:078d30763ae6697b7d55e49f4cb9e61407abd2137cafb5625f131aa47c1a47af
   Status: Downloaded newer image for localhost:5000/my-ubuntu:1.0
   
   ```