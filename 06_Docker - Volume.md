# Manage data in Docker

Docker에서 데이터를 관리하는 방법에 관한 내용으로, Docker로 컨테이너를 실행 후 컨테이너 안에서 파일을 생성하면 Docker의 기본설정으로 모든 파일들을 컨테이너 안의 writable layer에 저장됩니다.

하지만 이러한 매커니즘은 실제 시스템 운영환경에서는 많은 문제를 가지고 있습니다. 예를 들어 컨테이너에서 생성한 파일들이 컨테이너가 재시작되더라도 유지되어야 한다면, 아래의 내용을 생각해 보아야 합니다.

- 컨테이너가 중지되면 데이터는 컨테이너와 함께 제거되고, 유지되지 않습니다.
- 다른 프로세스 또는 애플리케이션에서 컨테이너 안의 데이터를 사용하기 위해 컨테이너 밖으로 데이터를 가져오는 것은 번거롭거나 어려운 작업이 될 수 있습니다.
- 컨테이너의 writable layer에 데이터를 기록하기 위해서는 파일스시템을 관리 할 [storage driver](https://docs.docker.com/storage/storagedriver/) 가 필요합니다. storage driver는 리눅스의 커널을 이용해서 union filesystem을 제공하기때문에 이러한 추가적인 추상화로 인해 Host 파일시스템에 직접 데이터를 기록하는 것과 비교해보면 성능측면에서 약점으로 작용합니다.

Docker에서는 컨테이너가 중지되더라도 데이터를 유지하기 위한 방법으로  Host 머신에 파일을 저장하는 3가지 방법을 제공합니다.

- Volume
- Bind mount
- Tmpfs mount (On Linux)



## Choose the right type of mount

위에서 언급한 3가지 유형의 어떤 마운트 방법을 사용하던, 컨테이너에서 보여지는 데이터는 동일하지만, 3가지 유혀의 Mount는 Host의 어떠한 영역에 마운트되느냐에 차이점이 있습니다.

![types of mounts and where they live on the Docker host](./img/types-of-mounts.png)

- **Volumes** : Docker에 의해 관리되는 Host 파일시스템 영역에 데이터를 저장합니다. 

  (`/var/lib/docker/volumes/` on Linux). 

  Docker에 의해 관리되고, Non-Docker process는 이 영역을 수정할 수 없으므로 Volume은 Docker에서 영구데이터를 관리하는 가장 좋은 방법입니다.

- **Bind mounts** : Host 머신의 파일시스템 어디에나 파일을 저장할 수 있는 방법입니다. Docker외의 다른 프로세스에서 데이터의 접근과 수정이 가능합니다.

- **tmpfs mounts** : Host의 메모리에 데이터를 저장하는 방법입니다.

  

### More details about mount types

#### Volume

Docker에 의해 만들어지고 관리되는 Volume은  `docker volume create` 명령어로 명시적으로 Volume을 생성할 수 있습니다.

```bash
root@ubuntu:/# docker volume create volume1
volume1

root@ubuntu:/# docker volume ls
DRIVER              VOLUME NAME
local               volume1
```

Volume을 생성하면 Docker Host 머신의 Docker에서 관리하는 디렉터리에 저장됩니다.

```bash
root@ubuntu:/# docker volume inspect volume1
[
    {
        "CreatedAt": "2019-02-22T02:25:12+09:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/volume1/_data",
        "Name": "volume1",
        "Options": {},
        "Scope": "local"
    }
]

root@ubuntu:/# ls /var/lib/docker/volumes/
metadata.db  volume1
```

Volume을 컨테이너 기동시 마운트하면 이 디렉터리가 컨테이너의 지정된 디렉터리로 마운트됩니다.

```bash
root@ubuntu:/# docker run -it --name myubuntu --mount source=volume1,target=/volumedata ubuntu
root@da5fe2af29c5:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  volumedata
root@da5fe2af29c5:/# cd volumedata/
root@da5fe2af29c5:/volumedata# touch hellovolume
root@da5fe2af29c5:/volumedata# ls
hellovolume
root@da5fe2af29c5:/volumedata# exit
exit

root@ubuntu:/# cd /var/lib/docker/volumes/volume1/_data/
root@ubuntu:/var/lib/docker/volumes/volume1/_data# ls
hellovolume
```

Container 정보를 살펴보면 기본설정으로 Volume의 읽기쓰기모드가 RW(읽기쓰기) 인 것을 알 수 있습니다.

`--mount` 옵션에 `readonly` 옵션을 주어 읽기쓰기 모드를 변경 할 수 있습니다

```
root@ubuntu:/# docker inspect myubuntu
[
    {
        "Id": "f6369bab35f2c89c0e0fed76c024decc4c19707aa833f6407fc0b1f60f42c38c",
        "Created": "2019-02-21T23:40:56.46535649Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 29553,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2019-02-21T23:40:57.125309312Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
       ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "volume1",
                "Source": "/var/lib/docker/volumes/volume1/_data",
                "Destination": "/volumedata",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ],
 
```

```bash
# docker run -it --name myubuntu1 --mount source=volume1,target=/volumedata,readonly ubuntu

# docker inspect myubuntu1
[
    {
        "Id": "47e5ba4a6746ab416e2b1985040805cc7574c368989ae9d9dbe6d2186236ba33",
        "Created": "2019-02-21T23:46:07.024314426Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 29663,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2019-02-21T23:46:07.413027114Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
       ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "volume1",
                "Source": "/var/lib/docker/volumes/volume1/_data",
                "Destination": "/volumedata",
                "Driver": "local",
                "Mode": "z",
                "RW": false,
                "Propagation": ""
            }
        ],
        ...

root@47e5ba4a6746:/volumedata# touch test
touch: cannot touch 'test': Read-only file system


```



Volume은 동시에 여러 컨테이너에 마운트 할 수 있으며, 자동으로 Volume이 제거되지 않습니다.

Volume을 제거하기 위해서는 `docker volume rm`,  `docker volume prune`  명령어로 제거 할 수 있습니다.

```bash
# docker volume rm volume1
```

사용하지 않은 Volume을 제거하기 `docker volume prune`  명령어로 정리 할 수 있습니다.

```bash
# docker volume prune
```

volume 관련 명령어

| Command                                                      | Description                                         |
| ------------------------------------------------------------ | --------------------------------------------------- |
| [docker volume create](https://docs.docker.com/engine/reference/commandline/volume_create/) | Create a volume                                     |
| [docker volume inspect](https://docs.docker.com/engine/reference/commandline/volume_inspect/) | Display detailed information on one or more volumes |
| [docker volume ls](https://docs.docker.com/engine/reference/commandline/volume_ls/) | List volumes                                        |
| [docker volume prune](https://docs.docker.com/engine/reference/commandline/volume_prune/) | Remove all unused local volumes                     |
| [docker volume rm](https://docs.docker.com/engine/reference/commandline/volume_rm/) | Remove one or more volumes                          |



#### Bind mount

Bind mount는 Volume에 비해 기능이 제한되어 있습니다. Bind mount를 사용하면 Container를 Host 머신의 특정 파일이나 디렉터리로 마운트합니다. Host의 마운트 경로는 Full Path를 사용하는데, 해당 파일이나 경로가 존재하지 않을 경우 자동생성되어지기때문에 미리 생성되어 있을 필요는 없습니다.

```bash
# docker run -it -v /volume/bindmount:/data/bindmount ubuntu
root@b1f8e57c3314:/data/bindmount# cd /data/bindmount
root@b1f8e57c3314:/data/bindmount# touch testfile
root@b1f8e57c3314:/data/bindmount# ls
testfile
root@b1f8e57c3314:/data/bindmount# exit
root@ubuntu:/# cd /volume/bindmount/
root@ubuntu:/volume/bindmount# ls
testfile
```



Bind mount는 Volume에 비해 아래와 같은 불리한 점이 있기 때문에 가능하다면 Volume을 사용하는 것이 권장됩니다.

- Bind mount는 Host의 디렉터리 구조에 의존적입니다.
- Docker에 의해 관리되어지지 않습니다.
- Container안의 프로세스가 Host의 파일시스템을 변경할 수 있으므로 보안상 위협이 될 수 있습니다.



#### tmpfs mounts

`tmpfs` mount는 디스크에 데이터를 영구적으로 저장하지 않고 Host의 메모리에 저장합니다. 이러한 특징 때문에 tmfs mount Container의 생명주기와 맞춰서 데이터를 보존하고자 할 때 사용되어질 수 있습니다. 예를 들어, 내부적으로 Docker swarm service는 tmpfs mount를 사용하여 컨테이너 안으로 secret을 마운트합니다.

