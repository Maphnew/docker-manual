### 1. Swarm node 구성
- book server (manager node) / 192.168.101.30
- samsung notebook (worker node) / 192.168.100.46
- msi notebook (worker node) / 192.168.100.47

> Swarm init
```
- input
[root@localhost its]# docker swarm init --advertise-addr 192.168.101.80

- output
Swarm initialized: current node (j047o4z6w71zwolmvaitbkb6e) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2v3sdrfzc6mkr310wqseoq1cevwkxupqldozhhzxa8jd4d8mmy-6c27bitjryr39a7wzr6j8ik1o \
    192.168.101.80:2377


To add a manager to this swarm, run 'docker swarm join-token manager' and follo                                        w the instructions.

```
> node list 확인
```
- input
[root@localhost its]# docker node ls

- output
ID                            HOSTNAME                STATUS         ENGINE VERSION
nhyjv7tmnvzwe92fszb238dtm *   localhost.localdomain   Ready          19.03.8
```
> worker node에서 join command
```
- input
[root@localhost its]# docker swarm join --token SWMTKN-1-1nkn5ygrdfrcjoowntchb8djsh1ufski1g4805y9b8s1odgp3r-1mcdbhcg831w57bvv25u9wdoy 192.168.101.30:2377


- output
-> error 

Error response from daemon: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial tcp 192.168.101.30:2377: connect: no route to host"
```
> manager node 방화벽 설정 
```
- input
[root@localhost its]# firewall-cmd --zone=public --permanent --add-port=2377/tcp
[root@localhost its]# firewall-cmd --reload
[root@localhost its]# firewall-cmd --zone=public --list-all
```
> worker node에서 재시도
```
- input
[root@localhost its]# docker swarm join --token SWMTKN-1-1nkn5ygrdfrcjoowntchb8djsh1ufski1g4805y9b8s1odgp3r-1mcdbhcg831w57bvv25u9wdoy 192.168.101.30:2377

- output
This node joined a swarm as a worker.
```
> manager node에서 node list 확인
```
- input
[root@localhost its]# docker node ls

- output

ID                            HOSTNAME                STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
ifs68ep6nwj49qpe4yimtxvqe     localhost.localdomain   Ready               Active                                  19.03.8
nhyjv7tmnvzwe92fszb238dtm *   localhost.localdomain   Ready               Active              Leader              19.03.8
xofrbvbruwlw9824jf29auu49     localhost.localdomain   Ready               Active                                  19.03.8
```

> manager node에서 join 명령어 검색
```
[root@localhost its]# docker swarm join-token manager
[root@localhost its]# docker swarm join-token worker
```
> Docker는 고 가용성을 구현하기 위해 클러스터 당 3 개 또는 5 개의 관리자 노드를 권장합니다. 스웜 모드 관리자 노드는 Raft를 사용하여 데이터를 공유하므로 홀수의 관리자가 있어야합니다. 관리자 노드의 절반 이상이 사용 가능한 한 웜은 계속 작동 할 수 있습니다.

### Node에서 제거
> Worker node: Leave Swarm
```
[root@localhost its]# docker swarm leave

Node left the swarm.
```
> Manager node: Remove Node
```
[root@localhost its]# docker node rm node-2
```

### 2. Stack / Service 실행
- git clone API_Server

0. Compose up > create image 
```
docker-compose up -d
```
1. Test - MariaDB
```
[root@bookserver mariadb-docker]# docker stack deploy -c docker-compose.yml mariadb1
Creating service mariadb1_mariadb
```
2. Deploy API Server 
- Nginx
- Gateway (bm4server)
- StandAlone (bm3server)
```
[root@bookserver API_Server]# docker stack deploy --compose-file docker-compose.yml api_server
Ignoring unsupported options: build, restart

Ignoring deprecated options:

container_name: Setting the container name is not supported.

Creating network api_server_backend
Creating service api_server_nginx
Creating service api_server_gateway
Creating service api_server_standalone
```



--- DOCKER REGISTRY -----

1. Docker registry 설치
<참조: https://novemberde.github.io/2017/04/09/Docker_Registry_0.html>

```
# registry 이미지를 가져오기
$ docker pull registry

# registry를 실행하기
$ docker run -dit --name docker-registry -p 5000:5000 registry

# hello-world 이미지가 없으니 docker hub에서 pull하자.
$ docker pull hello-world

# localhost/hello-world 이미지를 만들어보자.
$ docker tag hello-world localhost:5000/hello-world

# 이미지 push하기
$ docker push localhost:5000/hello-world

# 이미지 확인하기
$ curl -X GET http://localhost:5000/v2/_catalog
# 출력 {"repositories":["hello-world"]}

# 태그 정보 확인하기
$ curl -X GET http://localhost:5000/v2/hello-world/tags/list
# 출력 {"name":"hello-world","tags":["latest"]}
```

<참조: https://waspro.tistory.com/532>

- server gave HTTP response to HTTPS client 라는 메시지가 출력되는 경우 다음과 같이 daemon.json 파일을 수정합니다.

```
$ vi /etc/docker/daemon.json 
{
    "insecure-registries": ["192.168.101.70:5000"]
}
```
```
$ systemctl daemon-reload
$ systemctl restart docker
$ docker run -dit --name docker-registry -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true registry
```
- 실제 테스트 할 때 image 이름을 test로 사용함
```
$ docker pull 192.168.101.70:5000/test:latest
latest: Pulling from 192.168.101.70:5000/test
68ced04f60ab: Already exists
c4039fd85dcc: Already exists
c16ce02d3d61: Already exists
7469e3cf55fe: Pull complete
Digest: sha256:335079834819530f9746329187a4c26a011ea3f74d4b607106e3d2fbd339d273
Status: Downloaded newer image for 192.168.101.70:5000/test:latest

```

--- DELETE IMAGE IN REGISTRY ---

<참조: https://stackoverflow.com/questions/25436742/how-to-delete-images-from-a-private-docker-registry>

- to delete an image, you should run the registry container with REGISTRY_STORAGE_DELETE_ENABLED=true parameter.
- I set "delete: enabled" value to true in /etc/docker/registry/config.yml file. For this configuration no need to set REGISTRY_STORAGE_DELETE_ENABLED variable

---- REGISTRY VOLUME ----
$ docker run -dit --name docker-registry -p 5000:5000 -v /home/its/data/docker/registry:/var/lib/registry/docker/registry/v2 -e REGISTRY_STORAGE_DELETE_ENABLED=true registry


--

[root@its-apiserver its]# curl -k -I -H Accept:\* http://localhost:5000/v2/test/manifests/latest
HTTP/1.1 200 OK
Content-Length: 12076
Content-Type: application/vnd.docker.distribution.manifest.v1+prettyjws
Docker-Content-Digest: sha256:e034004b3d20f3a95d0dabd48aece8b5db1eaa2bc6d9b02163e5434c7013eb0a
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:e034004b3d20f3a95d0dabd48aece8b5db1eaa2bc6d9b02163e5434c7013eb0a"
X-Content-Type-Options: nosniff
Date: Mon, 06 Apr 2020 08:36:04 GMT


---

[root@its-apiserver its]# curl -X DELETE http://localhost:5000/v2/test/manifests/latest
{"errors":[{"code":"DIGEST_INVALID","message":"provided digest did not match uploaded content"}]}

----

[root@its-apiserver its]# curl -X DELETE http://localhost:5000/v2/test/manifests/sha256:e034004b3d20f3a95d0dabd48aece8b5db1eaa2bc6d9b02163e5434c7013eb0a
{"errors":[{"code":"MANIFEST_UNKNOWN","message":"manifest unknown"}]}

----
permission

----
