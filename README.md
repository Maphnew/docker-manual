# ITS Docker Manual
## Contents
1. Docker in ITS
2. Docker Swarm for Cloud - SaaS

## 1. Docker in ITS
1. API Server 192.168.101.70
- api_server_nginx(192.168.101.70:8083 / 106.255.236.186:8083)
- api_server_standalone(192.168.101.70:9211)
- api_server_gateway(192.168.101.70:9210)
- uyegvkfjdls3988(uyeg-dka)
- uyegg210000000004(uyeg-ka)
- uyegg944d0bb37cf2(uyeg-ka)
- uyeggrtwhgapjbh2r(uyeg-ka)
- uyegga90fd1632be9(uyeg-ka)
- uyegzxcvbn8347(uyeg-dka)

2. App Server 192.168.101.80
- app_server_nginx(192.168.101.80:8082 / 106.255.236.186:8082)
- app_server_mobile(192.168.101.80:9810)
- app_server_web(192.168.101.80:9811)
- app_server_set(192.168.101.80:9812)
- app_server_indexing(192.168.101.80:9813)

3. ML Server 192.168.100.75
- tensorflow/tensorflow:latest-gpu-py3-jupyter(192.168.100.175:8888)
- curritem039(데이터분석)
- activepoweritem005(데이터분석)
- curritem005(데이터분석)
- activepoweritem039(데이터분석)

## 2. Docker Swarm for Cloud - SaaS

### 1. Swarm node 구성
- book server (manager node) / 192.168.101.30
- samsung notebook (worker node) / 192.168.100.46
- msi notebook (worker node) / 192.168.100.47

> Swarm init
```
- input
[root@localhost its]# docker swarm init --advertise-addr 192.168.101.30

- output
Swarm initialized: current node (nhyjv7tmnvzwe92fszb238dtm) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1nkn5ygrdfrcjoowntchb8djsh1ufski1g4805y9b8s1odgp3r-1mcdbhcg831w57bvv25u9wdoy 192.168.101.30:2377

To add a manager to this swarm, run 'docker swarm                  join-token manager' and follow the instructions.
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


### 3. Label on node
1. Labeling on each node
```
docker node update --label-add dmz=true node2
docker service create --name dmz-nginx --constraint node.labels.dmz==true --replicas 2 nginx
```
> on compose file
```
deploy:
  placement:
    constraints:
        - node.labels.disk == ssd
```
> remove the service and labels we created
```
docker node update --label-rm dmz node2

```
2. Global
> Global: Place one task on each node in Swarm
```
docker service create --mode=global nginx
```
> Place one task on each Worker in Swarm
```
docker service create --mode=global --constrant=node.role==worker nginx
```
> compose file
```
deploy:
  mode: global
```
3. preferences
```
deploy:
  placement:
    preferences:
      spread: node.labels.azone
```


참조 블로그
<https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html>