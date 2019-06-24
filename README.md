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
docker image tag ~~~~~~~~~~~
docker image push ~~
```
		
#### 도커 이미지 내려 받기 (p114)
Worker 컨테이너가 registry 컨테이너로부터 도커 이미지를 내려 받을 수 있는지 확인
```
docker image pull ~~~~~
docker container exec -it worker01 docker image pull registry:5000/study01/echo:latest
```
			
### 3.5.2. 서비스
애플리케이션은 단일 컨테이너일 수도 있고 여러 종류의 컨테이너 일 수도 있다. 애플리케이션을 구성하는 일부 컨테이너를 제어하기 위한 단위로 서비스라는 개념이 생김.

서비스 만들기(p115)
```
 docker container exec -it manager docker service create --replicas 1 --publish 8000:8080 --name echo ~~~
```
		
서비스 확인(p116)
```
 docker container exec -it manager docker service ls
```	

스케일아웃(p116)
```
 docker service scale 명령으로 서비스의 컨테이너 수를 조정
 docker container execd -it manager docker service scale echo=6
```
	
확인(p116)
```
 docker container exec -it manager docker service ps echo | grep Run
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
docker container exec -it manager docker stack deploy -c /stack/ch03-weapi.yml echo
```
	
#### 배포된 스택 확인하기 (p120)
