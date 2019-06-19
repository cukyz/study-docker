# 2장
Docker-compose  
도커는 복잡한 설정을 쉽게 관리하기 위해 YAML방식의 설정파일을 이용한 'Docker Compose'라는 툴을 제공

Docker-compose 명령어
```
docker-compose up
docker-compose down
```

# 3장
## 3.1. 애플리케이션과 시스템 내 단일 컨테이너의 적정 비중
컨테이너 안에 애플리케이션을 몇 개나?  
컨테이너 1개 = 프로세스 1개?  
컨테이너가 맡을 역할을 적절히 나누고, 그 역할에 따라 배치한 컨테이너를 복제해도 전체 구조에서 부작용이 일어나지 않는가  

## 3.2. 컨테이너 이식성
도커의 큰 장점은 이식성. 그러나 완벽하지 않다.
- 커널 및 아키텍처: ARM / x86_64 아키텍처 가림
- 네이티브 라이브러리를 동적으로 사용하는 문제

## 3.3. 도커 친화적인 애플리케이션
- 실행인자
- 설정파일사용:  호스트에 대한 의존성이 생긴다.
- 애플리케이션 동작을 환경 변수로 제어 
- docker-compose.yml 의 env 속성.
- 설정파일에 환경변수를 포함

## 3.4. 퍼시스턴스 데이터를 다루는 방법
 데이터 볼륨: 
  - 도커 컨테이너 안의 디렉토리를 디스크에 퍼시스턴스 데이터로 남기기 위한 메커니즘
  - 호스트와 컨테이너 사이의 디렉토리 공유
  - 컨테이너를 파기해도 디스크에 그대로 남는다.
  - 명령어 : docker container run -v
		
 데이터 볼륨 컨테이너:
  - 추천되는 방법
  - 컨테이너 간 디렉토리를 공유한다.
  - 호스트의 스토리지에 저장되는 것은 데이터볼륨과 같다. 데이터 볼륨은 호스트 쪽 특정 디렉토리에 의존성을 갖는다. 데이터 볼륨 컨테이너는 도커가 관리하는 디렉토리 영역에만 영향을 미친다. (/var/lib/docker/volumes)
  - 호스트의 디렉토리를 알 필요가 없다.

## 3.5. 컨테이너 배치 전략
### 3.5.1 도커스웜 (p108)
여러 도커 호스트를 클러스터로 묶어주는 컨테이너 오케스트레이션 도구  
오케스트레이션 도구를 사용하여 컨테이너의 배치나 통신 제어를 조율  
별도로 있었으나 docker에 통합됨  
Master/ slave 와 같이 생각하면 됨  
		
#### 여러 대의 도커 호스트로 스웜 클러스터 구성하기 (p108)
학습용으로 여러 호스트를 구성하기 위해 dind라는 간단한 방법을 사용  
Docker in Docker : dind  
도커 컨테이너 안에서 도커 호스트 실행  

여기서 사용할 컨테이너는 3종류 5개
- registry ×1
- manager ×1
- worker ×3

Registry : 도커 레지스트리 역할, manager 및 worker 컨테이너가 사용하는 컨테이너   
Manager: 스웜 클러스터 전체를 제어, 여러 대 실행되는 도커 호스트(worker)에 서비스가 담긴 컨테이너를 배치  
위의 구성을 도커 컴포즈로 구축.
- 모든 maager 및 worker 컨테이너는 registry 컨테이너에 의존.
- P110 소스 조심. ( manager, worker0x 가 services 하위에 있도록 공백 넣어야 함)
		
#### 컴포즈 실행 (p111)
```
docker-compose up -d
docker container ls
```

`docker comtainer ls` 실행결과

```
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
70f38edc0918        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker03
87b56e609082        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker02
7f1d05c3fb9e        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker01
1951cabcf225        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   4 minutes ago       Up 4 minutes        2375/tcp, 3375/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:8000->80/tcp   manager
052ac84438cb        registry:2.6             "/entrypoint.sh /etc…"   4 minutes ago       Up 4 minutes        0.0.0.0:5000->5000/tcp                                             registry
```
		
#### Manaer 역할 맡기기(p112)
스웜의 manager 역할을 맡긴다.
``` 
$ docker container exec -it manager docker swarm init
Swarm initialized: current node (p03f5rr1mbcxgsubls54vwiec) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1eo2ygi371y22ixjkhdp6i32qiq47f8udwyol6u2d9e2204907-16vyeunwd6aq7v6kfqqm6ngzu 172.21.0.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
Join 토큰이 생성되는데 이 토큰을 사용하여 worker를 manager가 관리하는 컨테이너로 등록 (스웜클러스터에 등록)
		
#### Join (p113)
Worker가 3개이므로 worker01 ~ 03 까지 등록  
```
$ docker container exec -it worker01 docker swarm join \
> --token SWMTKN-1-1eo2ygi371y22ixjkhdp6i32qiq47f8udwyol6u2d9e2204907-16vyeunwd6aq7v6kfqqm6ngzu manager:2377
This node joined a swarm as a worker.
```


#### 도커 레지스트리에 이미지 등록하기
	
Registry 컨테이너에 도커 이미지 등록 (p113)
외부 도커에서 빌드한 이미지를 사용하기 위해 레지스트리에 등록  
도커허브가 아닌 내부적으로 만든 레지스트리에 등록  
1. Image에 tag를 붙인다.
2. push한다.
```
cuky@cuky:~/dev/study-docker$ docker image tag example/echo:latest localhost:5000/example/echo:latest
cuky@cuky:~/dev/study-docker$ docker image push localhost:5000/example/echo:latest
The push refers to repository [localhost:5000/example/echo]
0f2827fdffbe: Mounted from study01/echo 
a808b3bcd4bf: Mounted from study01/echo 
186d94bd2c62: Mounted from study01/echo 
24a9d20e5bee: Mounted from study01/echo 
e7dc337030ba: Mounted from study01/echo 
920961b94eb3: Mounted from study01/echo 
fa0c3f992cbd: Mounted from study01/echo 
ce6466f43b11: Mounted from study01/echo 
719d45669b35: Mounted from study01/echo 
3b10514a95be: Mounted from study01/echo 
latest: digest: sha256:715b39ea535151213099f586eb206a923d0cec017f4396fcccec9c11d89ab641 size: 2418
```
		
#### 도커 이미지 내려 받기 (p114)
Worker 컨테이너가 registry 컨테이너로부터 도커 이미지를 내려 받을 수 있는지 확인  
docker image pull 
```
cuky@cuky:~/dev/study-docker$ docker container exec -it worker01 docker image pull registry:5000/example/echo:latest
latest: Pulling from example/echo
Digest: sha256:715b39ea535151213099f586eb206a923d0cec017f4396fcccec9c11d89ab641
Status: Image is up to date for registry:5000/example/echo:latest
```
			
### 3.5.2. 서비스
애플리케이션은 단일 컨테이너일 수도 있고 여러 종류의 컨테이너 일 수도 있다. 애플리케이션을 구성하는 일부 컨테이너를 제어하기 위한 단위로 서비스라는 개념이 생김.

서비스 만들기(p115)
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/example/echo:latest
eh3sh32u31il4s9bhe1ay3gz2
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```
		
서비스 확인(p116)
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
eh3sh32u31il        echo                replicated          1/1                 registry:5000/example/echo:latest   *:8000->8080/tcp
```	

스케일아웃(p116)
docker service scale 명령으로 서비스의 컨테이너 수를 조정
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker service scale echo=6
echo scaled to 6
overall progress: 6 out of 6 tasks 
1/6: running   [==================================================>] 
2/6: running   [==================================================>] 
3/6: running   [==================================================>] 
4/6: running   [==================================================>] 
5/6: running   [==================================================>] 
6/6: running   [==================================================>] 
verify: Service converged 
```
	
확인(p116)
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker service ps echo | grep Run
xaa4udz473k8        echo.1              registry:5000/example/echo:latest   ccd45673f4b5        Running             Running about a minute ago                       
l4lyzwqmr274        echo.2              registry:5000/example/echo:latest   739657badf61        Running             Running 20 seconds ago                           
7idn5pgno5eb        echo.3              registry:5000/example/echo:latest   ccd45673f4b5        Running             Running 21 seconds ago                           
vonee6zze4u9        echo.4              registry:5000/example/echo:latest   731b9c121e31        Running             Running 21 seconds ago                           
9iqfdveqf8rm        echo.5              registry:5000/example/echo:latest   7a4cb7c4aedd        Running             Running 21 seconds ago                           
60a76kgxhwln        echo.6              registry:5000/example/echo:latest   739657badf61        Running             Running 19 seconds ago      
```

삭제 (p117)
```
 docker container exec -it manager docker service rm echo
```

	
### 3.5.3. 스택
하나 이상의 서비스를 그룹으로 묶은 단위  
서비스는 애플리케이션 이미지를 하나밖에 다루지 못하지만, 여러 서비스가 상호작용하여 동작하는 형태로 다양한 애플리케이션을 구성할 수 있게 하기 위한 상위 개념이 스택이다.
	
	
#### Overlay 네트워크 (p117)
- 여러 도커 호스트에 걸쳐 배포된 컨테이너 그룹을 같은 네트워크에 배치하기 위한 기술
- 스택은 overlay네트워크를 사용해야 서로 다른 호스트에 위치한 컨테이너끼리 통신할 수 있다.
		
#### 스택생성
		
		
#### 스택 배포하기 (p119)
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
Creating service echo_nginx
Creating service echo_api
```
	
#### 배포된 스택 확인하기 (p120)
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker stack services echo
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
6nynot2d1p00        echo_nginx          replicated          3/3                 gihyodocker/nginx-proxy:latest      
7ucjjpo68aik        echo_api            replicated          3/3                 registry:5000/example/echo:latest   
```

#### 스택에 배포된 컨테이너 확인하기
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker stack ps echo
ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
yj43qbggxy3g        echo_api.1          registry:5000/example/echo:latest   4236174bcf06        Running             Running 36 minutes ago                       
2aj6iodumhwe        echo_nginx.1        gihyodocker/nginx-proxy:latest      4236174bcf06        Running             Running 36 minutes ago                       
swmp4gmq6t3h        echo_api.2          registry:5000/example/echo:latest   5baf1b2ea0ff        Running             Running 36 minutes ago                       
ywl5t09d7uhu        echo_nginx.2        gihyodocker/nginx-proxy:latest      5c442309ea00        Running             Running 36 minutes ago                       
ti787eyutjj0        echo_api.3          registry:5000/example/echo:latest   5c442309ea00        Running             Running 36 minutes ago                       
w0nksb4mjyaf        echo_nginx.3        gihyodocker/nginx-proxy:latest      20cc15285958        Running             Running 36 minutes ago     
```

#### visualizer를 사용해 컨테이너 배치 시각화하기
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer
Creating network visualizer_default
Creating service visualizer_app
```
#### 스택 삭제하기
```
cuky@cuky:~/dev/study-docker/stack$ docker container exec -it manager docker stack rm echo
Removing service echo_api
Removing service echo_nginx
```

### 3.5.4 스웜 클러스터 외부에서 서비스 사용하기
HAProxy를 사용하여 스워 클러스터 외부에서 echo_nginx 서비스 접근.  
localhost:8000 -> HAProxy 80 -> 

#### ch03-webapi.yml을 스택echo로 다시 배포
```
cuky@cuky:~/dev/study-docker/stack$ docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
Updating service echo_api (id: upmc74mds1bmoyxdd0ap5sa9f)
Updating service echo_nginx (id: muox61bzuq81vfohr246nb381)
```

#### ch03-ingress.yml을 스택 ingress로 배포
```
cuky@cuky:~/dev/study-docker/stack$ docker container exec -it manager docker stack deploy -c /stack/ch03-ingress.yml ingress
Creating service ingress_haproxy
```

# 4장. 스웜을 이용한 실전 애플리케이션 개발
## 4.1. 웹 애플리케이션 구성

## 4.2. MySQL 서비스 구축

### 빌드 및 스웜 클러스터에서 사용하기
```
cuky@cuky:~/dev/tododb$ docker image build -t ch04/tododb:latest .
Sending build context to Docker daemon  116.2kB
Step 1/16 : FROM mysql:5.7
5.7: Pulling from library/mysql
fc7181108d40: Pull complete 
787a24c80112: Pull complete 
a08cb039d3cd: Pull complete 
4f7d35eb5394: Pull complete 
5aa21f895d95: Pull complete 
a742e211b7a2: Pull complete 
0163805ad937: Pull complete 
62d0ebcbfc71: Pull complete 
559856d01c93: Pull complete 
c849d5f46e83: Pull complete 
f114c210789a: Pull complete 
Digest: sha256:c3594c6528b31c6222ba426d836600abd45f554d078ef661d3c882604c70ad0a
Status: Downloaded newer image for mysql:5.7
 ---> a1aa4f76fab9
Step 2/16 : RUN apt-get update
 ---> Running in d8057bd8d2ce
Get:1 http://repo.mysql.com/apt/debian stretch InRelease [21.7 kB]
Get:5 http://repo.mysql.com/apt/debian stretch/mysql-5.7 amd64 Packages [5716 B]
Ign:2 http://cdn-fastly.deb.debian.org/debian stretch InRelease
Get:3 http://cdn-fastly.deb.debian.org/debian stretch-updates InRelease [91.0 kB]
Get:4 http://security-cdn.debian.org/debian-security stretch/updates InRelease [94.3 kB]
Get:6 http://cdn-fastly.deb.debian.org/debian stretch Release [118 kB]
Get:7 http://cdn-fastly.deb.debian.org/debian stretch Release.gpg [2434 B]
Get:8 http://security-cdn.debian.org/debian-security stretch/updates/main amd64 Packages [499 kB]
Get:9 http://cdn-fastly.deb.debian.org/debian stretch-updates/main amd64 Packages [27.2 kB]
Get:10 http://cdn-fastly.deb.debian.org/debian stretch/main amd64 Packages [7082 kB]
Fetched 7941 kB in 7s (1063 kB/s)
Reading package lists...
.
.
.
.
.
Step 9/16 : COPY add-server-id.sh /usr/local/bin/
 ---> d827352da9ff
Step 10/16 : COPY etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/
 ---> 55964a5978c5
Step 11/16 : COPY etc/mysql/conf.d/mysql.cnf /etc/mysql/conf.d/
 ---> 3cd44d4dfdbf
Step 12/16 : COPY prepare.sh /docker-entrypoint-initdb.d
 ---> 44a9929d0823
Step 13/16 : COPY init-data.sh /usr/local/bin/
 ---> 26c9e0602250
Step 14/16 : COPY sql /sql
 ---> 7314560ddaff
Step 15/16 : ENTRYPOINT [   "prehook",     "add-server-id.sh",     "--",   "docker-entrypoint.sh" ]
 ---> Running in d2326624fe6c
Removing intermediate container d2326624fe6c
 ---> d8fd8b8ba222
Step 16/16 : CMD ["mysqld"]
 ---> Running in 4b0599ff2630
Removing intermediate container 4b0599ff2630
 ---> e691a406b435
Successfully built e691a406b435
Successfully tagged ch04/tododb:latest

```


#### 이미지 registry 등록 (tag, push)
```
cuky@cuky:~/dev/tododb$ docker image tag ch04/tododb:latest localhost:5000/ch04/tododb:latest
cuky@cuky:~/dev/tododb$ 
cuky@cuky:~/dev/tododb$ 
cuky@cuky:~/dev/tododb$ 
cuky@cuky:~/dev/tododb$ docker image push localhost:5000/ch04/tododb:latest
The push refers to repository [localhost:5000/ch04/tododb]
b78ae42b3a42: Pushed 
3389cb3012bd: Pushed 
a34ce42f2f0f: Pushed 
ed4f470f5e72: Pushed 
a40c726235a8: Pushed 
89cff9dff1b5: Pushed 
627b81839d00: Pushed 
4657d31ba1bd: Pushed 
f985ccf437ca: Pushed 
e9651980f89e: Pushed 
1e9b0de5a957: Pushed 
b7ca84812a9e: Pushed 
b68cb4281fa3: Pushed 
fe502c068612: Pushed 
099a334da0eb: Pushed 
013702f09688: Pushed 
de99e027e630: Pushed 
4a6063139efa: Pushed 
7dd55886f8d3: Pushed 
17d2cfdb93fc: Pushed 
cf1d44aef62d: Pushed 
b983f63f4c27: Pushed 
b686f68f8333: Pushed 
cf5b3c6798f7: Pushed 
latest: digest: sha256:574e4780f01f2b38a3b4de415b1c1f664f13ffc208e09e817a451e0b5ffca068 size: 5334

```

### 스웜에서 마스터 및 슬레이브 실행
todo-mysql.yml에 정의된 서비스를 todo_mysql 스택으로 manager 컨테이너에 배포.  
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager docker stack deploy -c /stack/todo-mysql.yml todo_mysql
Creating service todo_mysql_master
Creating service todo_mysql_slave

```

배포확인.  
```
cuky@cuky:~/dev/study-docker$ docker container exec -it manager \
> docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
upmc74mds1bm        echo_api            replicated          3/3                 registry:5000/example/echo:latest   
muox61bzuq81        echo_nginx          replicated          3/3                 gihyodocker/nginx-proxy:latest      
slplzp0h24oo        ingress_haproxy     global              1/1                 dockercloud/haproxy:latest          *:80->80/tcp, *:1936->1936/tcp
2ezcara97fjr        todo_mysql_master   replicated          0/1                 registry:5000/ch04/tododb:latest    
d43ku8mfwxsf        todo_mysql_slave    replicated          0/2                 registry:5000/ch04/tododb:latest    
zqdhc7uxsjmy        visualizer_app      global              1/1                 dockersamples/visualizer:latest     *:9000->8080/tcp
cuky@cuky:~/dev/study-docker$ 

```


## 4.3. API 서비스 구축
