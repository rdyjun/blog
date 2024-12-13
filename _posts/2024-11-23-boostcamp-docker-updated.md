---
title: "[Docker] 처음 사용해본 도커로 배포 리팩토링 해보기"
author: rdyjun
excerpt: "어떻게 기존 배포 방식을 더 효율적으로 개선할까?"

categories: [docker]
tags: [Docker, nginx, compose, deploy, github, actions]

toc: true
toc_sticky: true

date: 2024-11-23 23:17:00 +0800
last_modified_at: 2024-11-23 23:17:00 +0800

pin: false
---

![배포 아키텍처](https://github.com/user-attachments/assets/37cda675-2eb6-43f5-aa3c-64e28a225da2)

> 위는 리팩토링 이후 배포 상태이다.

## 배포 방식을 수정하는 이유

기존에는 클라우드 서버에서 `git pull`로 최신 커밋을 내려받고,  
도커 컴포즈 파일로 빌드 후 그린/블루 배포 방식을 통해 실행하는 구조였다.

하지만, 이 방식은 도커의 장점을 살리지 못하고, 굳이 도커를 추가로 가져가는 느낌이었다.  
이런 문제를 해결하고자 도커를 제거할 것인지, 도커로만 운영할 것인지 고민했다.

GitHub 파일로 관리할 경우 환경 설정이나 의존성에 따라 문제가 발생할 수도 있다는 이슈가 있었고,  
도커가 가지는 이미지 버전 관리 및 컨테이너 관리 용이 등의 장점을 경험해보기 위해 도커 사용을 결정했다.

## 수정 방안

우선 도커 이미지로 빌드하는 역할을 GitHub Actions에 분배하기로 했다.  
GitHub Actions에서 프로젝트 파일을 불러와서 docker 이미지로 빌드하고,
Docker Hub에 push하는 빌드 흐름을 가져가기로 했다.

배포는 GitHub Actions에서 ssh로 클라우드 서버에 접근한 후 Docker Hub에서 최신 이미지를 pull하고 저장된 compose 파일을 실행시키는 것으로 결정했다.

모노레포였기 때문에 프론트엔드 도커 이미지와 백엔드 도커 이미지로 나누기로 했다.

> 2024-11-18 프론트엔드 도커 배포 방식 -> 정적 파일로 변경  
> 2024-12-12 프론트엔드/백엔드 전체 재배포 -> 독립적 배포 파이프라인 분리

## 문제 해결 과정

### 이미지를 어떻게 올리지?

[Docker Hub에 Docker Image 올리기](https://www.notion.so/Docker-Hub-Docker-Image-140510ea038e80cfb110eb7309dc666a?pvs=21)

### Redis 연결 실패

![image.png](https://github.com/user-attachments/assets/9a4c17a7-553c-4daa-a89e-aa0525db6afb)

위 사진은도커 compose 파일을 실행시키고 뜬 로그이다.  
이 로그에서 `6379` 포트에 문제가 있는 것을 보고 redis 연결이 안되어있는 것을 알 수 있었다.
redis 연결을 하기 위해서는 .env 설정이 필요했다.

깃허브 저장소에는 .env 파일이 들어가지 않았기 때문에 서버에 .env 파일을 만들어주고,  
가지고 있는 이미지가 .env 파일을 사용할 수 있도록 해주어야 한다고 생각했다.  
검색해본 결과, Docker Compose 파일에서 `env_file` 이라는 옵션을 통해서 로컬의 .env 파일을 매핑시켜줄 수 있었다.

그렇게 클라우드 서버에 server.env와 client.env 파일을 만들어주고,  
각 이미지에 맞게 .env 파일을 가지도록 설정했다.

```bash
  server-green:
    image: "rdyjun/inear-server:latest"
    container_name: server-green
    expose:
      - '3000'
    env_file:
      - ./server.env        # 이 부분
    environment:
      - NODE_ENV=development
    networks:
      - webapp
    # healthcheck 임시 제거
    restart: unless-stopped
  client:
    image: "rdyjun/inear-client:latest"
    container_name: client
    expose:
      - '5173'
    env_file:
      - ./client.env        # 이 부분
    environment:
      - NODE_ENV=development
    networks:
      - webapp
    restart: unless-stopped
```

위 설정을 통한 결과는…

~~효과는 대단했다!~~

![image.png](https://github.com/user-attachments/assets/2b339187-deda-4ab4-b776-fa9957b37ff9)

redis 연결이 잘 이루어져 서버가 정상적으로 실행되었다.

### 웹이 안보인다!

![image.png](https://github.com/user-attachments/assets/2292e90d-681d-4c87-beb4-456d86fe5e24)

![image.png](https://github.com/user-attachments/assets/b6f1c8d6-e65f-4724-8bbe-9cb93d509223)

위 사진과 같이 서버가 제대로 실행되고 있음에도 실제 페이지에 접속이 되지 않는 문제가 발생했다.  
자세히 보니, `react-router-dom` 이 제대로 설치되지 않은 문제가 보인다.

확인해보니 어제(24.11.15) 이 문제가 발생했고, 라이브러리 설치 경로가 잘못되어 그런 것이라고 확인되었다.  
팀원분께서 우선 제거하고 진행한다고 하셔서 제거된 버전을 merge했더니 해당 오류가 없어진 것을 확인할 수 있었다.

![image.png](https://github.com/user-attachments/assets/63501b16-4c0c-477f-887e-41021e66860b)

### 말썽꾸러기 nginx

![image.png](https://github.com/user-attachments/assets/8041d5c2-b64b-4b1e-a471-f2b71d087fdb)

앞에서의 문제가 해결되고 페이지가 제대로 뜰 것이라고 생각했다.  
하지만 위 사진과 같이 아직 문제가 해결되지 않았다.

우선 아까 전 사진에서의 로그 상으로는 문제없이 nginx, react, nest가 실행되는 것을 확인할 수 있다.

원인이 뭘까를 고민해봤다.

- server 또는 client 프로젝트가 잘못됐다
  - 이건 내가 보기엔 아니었다.
  - 로그가 정상적으로 출력되었기 때문이다.
- 방화벽?
  - 배포 방식을 바꾸기 전에는 잘 접속됐던 것을 생각해보면, 방화벽은 원인이 아니다.
- 그럼 nginx가 server 이미지 및 client 이미지와 연결되지 못했나?
  - 그렇다. nginx가 server 이미지 및 client 이미지를 알지 못했다.

### nginx 이미지는 뭐가 문제일까?

가장 먼저 docker ps를 통해 포트를 확인했다.  
아래와 같이 모든 이미지의 포트는 정상적으로 출력중이었다.

```bash
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS         PORTS                NAMES
5f3f397d0de4   rdyjun/inear-nginx:latest    "/docker-entrypoint.…"   5 seconds ago    Up 4 seconds   0.0.0.0:80->80/tcp   nginx
24dd46260274   rdyjun/inear-server:latest   "docker-entrypoint.s…"   5 seconds ago    Up 4 seconds   3000/tcp             server-green
4c464b8d4863   rdyjun/inear-client:latest   "docker-entrypoint.s…"   37 minutes ago   Up 4 seconds   5173/tcp             client
```

혹여 도커 컴포즈 파일에서 server 이미지와 client 이미지에 대해 `expose`가 아닌 `ports` 옵션을 사용하면 해결 될지도 모른다는 생각에 docker-compose.yml 파일에 expose를 지우고 ports: 3000:3000과 ports: 5173:5173을 추가했는데 여전히 작동하지 않았다.

> ports 옵션 자체가 외부 접근 포트를 의미하기 때문에 다시 제거하였다.  
> expose 옵션은 내부 포트로 컨테이너 간 접근 포트를 의미한다.

그러면 도대체 뭐가 문제지?  
이런저런 작업을 해보려고 명령어를 입력하던 중에 ls를 입력하게 되었다.  
그런데, 원래 없었던 nginx 파일이 생겼던 것이다.

그렇게 리팩토링 전 저장소에 저장해둔 nginx 파일을 덮어씌워보면 어떨까라는 생각을 하게 되었다.  
결과적으로는… 성공했다..!

더 자세히 확인해보기 위해 `docker-compose.yml` 내부를 확인했고,  
volumes 옵션에 `./nginx/conf.d:/etc/nginx/conf.d:ro` 경로를 마운트하는 것을 볼 수 있었다.

```bash
services:
  nginx:
    image: "rdyjun/inear-nginx:latest"
    container_name: nginx
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    ports:
      - '80:80'
    networks:
      - webapp
    restart: unless-stopped
```

이로써 nginx 도커 컴포즈 파일을 실행시키는 데 필요한 디렉토리가 존재했다는 사실을 알았고,  
이 디렉토리가 존재하지 않아 자동으로 생성되었다는 사실을 알게되었다.

결과적으로 해당 nginx 컨테이너의 `/etc/nginx/conf.d/default.conf`파일 내용이 없어서 동작하지 않았던 것이다.

도커 컴포즈와 같은 경로에 nginx/conf.d/default.conf에 아래 내용들을 추가해주어서 문제를 해결했다.

```conf
server {
    listen 80;
    server_name localhost;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;

    location / {
        proxy_pass http://client-green:5173;
    }

    location /api {
        proxy_pass http://server-green:3000;
    }
}
```

### 무중단 배포

앞 과정까지 기본적인 자동 배포 사이클을 완성했다.  
이제 기존 green/blue 배포 방식을 지금 배포 방식에 맞게 개선해야 했다.

### 바뀌어버린 redis 포트

서버를 3001번 포트로 실행하려고 deploy.sh에 서버 env 파일의 PORT=3000을 PORT=3001로 바꾸는 스크립트를 작성했다.

```sh
sed -i "s|PORT=.*;|PORT=3001;|g" ./env/server.env
sed -i "s|PORT=.*;|PORT=3000;|g" ./env/server.env
```

그런데, PORT:를 파싱하는 과정에서 REDIS_PORT:도 파싱되어 포트가 바뀌는 문제가 있었다 하하;;  
아래와 같이 수정하여 해당 문제는 해결하게 되었다.

```sh
sed -i "s|^PORT=.*;$|PORT=3001;|g" ./env/server.env
sed -i "s|^PORT=.*;$|PORT=3000;|g" ./env/server.env
```

### 서로 다른 포트는 의미없다.. (도커의 독립적 컨테이너 환경)

클라이언트와 서버 프로젝트를 그린/블루로 띄우려면, 서로 다른 포트로 실행해야 한다고 생각했다.  
그 이유는 이미 한 컨테이너가 실행중인 상태에서 동시에 다른 서버를 띄우고,  
트래픽을 옮기고, 트래픽이 옮겨졌을 때, 기존 컨테이너를 종료하는 방식이기 때문이다.

지금의 구조에서는 서버가 3000포트이고, 클라이언트가 5173 포트를 쓴다.  
그래서 1씩 더한 3001, 5174를 쓰려고 했다.

다만, 여기서의 문제가 server에서는 .env에서 `PORT:3000`, `PORT:3001`과 같이 수정해서 실행하면 해당 포트로 실행이 되지만, 클라이언트는 이미지에 포함되어있는 설정 파일에서 5173 포트를 직접 건드려야 했다...

그렇게 서버 컨테이너와 클라이언트 컨테이너가 포트를 바꾸지 않고 실행되도록 할 방법을 찾고자하였다.

docker-compose 파일의 옵션으로 `expose`와 `ports`가 있었다.  
나는 이때, expose가 컨테이너 내부에서 어떤 포트를 사용할 것인지 설정하는 역할이라고 생각했다.  
그리고 ports는 실제 외부에서 접근할 포트를 설정하는 역할이었다.

우선은 nginx 컨테이너와만 소통해야 하기 때문에 ports는 건드리지 않기로 했다.
그렇게 expose에서 5174:5173을 시도해봤다.  
nginx의 client port 설정을 바꿔줬음에도 bad gateway가 나왔다.

> 당연히 이 부분을 ports에 적용해도 마찬가지였다.

위 시도에서의 문제점은 expose는 port:port 구조가 불가능하다는 것이다.  
port:port 구조에서의 앞 port는 호스트(물리 컴퓨터)가 외부에 열어두는 포트이고  
뒤의 port가 내부 3000번 포트를 가리키는 구조이다.  
expose는 내부에서 사용할 포트를 기록하는 메타데이터 용도였기 때문에 외부 포트를 저장하는 것도 의미없는 행위이고, expose를 바꾼다고 달라지는 일은 없었다...

다만, 계속해서 하다보니 한 가지 깨달은 것이 있었다.  
각 컨테이너가 서로 다른 포트가 아니어도 된다는 사실이다.  
이 사실을 알고 해결할 수 있다는 안도감과 이제서야 알았다는 자괴감을 느꼈지만 시간이 없기에 빠르게 적용해봤다.

결과적으로 client-blue, client-green, server-blue, server-green 4개의 컨테이너를 각각

3000포트와 5173 포트로 열 수 있었다… 🎉

이게 가능한 이유는, 각 컨테이너가 네임스페이스로 관리되기 때문에 포트가 충돌되지 않는다는 것이다.

그렇게 서브 클라이언트와 서브 서버 컨테이너를 띄우고 실행중이던 서버와 클라이언트 컨테이너를 종료하게끔 작성하였다.

### 뒤바뀌어 버린 client 배포 방식…

Vite로 배포했을 때 페이지에 접근하면 2~3초 가량 하얀 화면이 유지되다가 갑자기 페이지가 로드되는 이슈가 있었다.

문제 해결을 위해 이런 사례들을 검색했고, 클라이언트 프로젝트를 빌드한 후 생긴 정적 파일(dist)을 nginx에 마운트하여 사용할 수 있다는 것을 확인했다.  
위 문제에 대한 해결과 도커 이미지가 아닌 빌드된 정적 파일만 가진다는 경량화 이점을 같이 챙길 수 있다고 판단하여 적용하기로 결정했다.

여기서 추가 고민이 생겼다.

1. scp로 넘겨서 실행중인 nginx 컨테이너가 해당 파일을 마운트한다.
2. nginx 이미지 자체에 client 정적 파일을 포함시키고, 서버처럼 blue/green 기법 사용

위 두 가지를 고민했던 이유는 버전관리의 이유 때문이었다.  
client 프로젝트가 nginx 도커 이미지에 포함되면 경량화와 로딩 문제를 해결하고 버전관리까지 가능했기 때문이다.

여기까지의 생각대로 2번 방식을 적용하여 GitHub Actions에 nginx 이미지를 추가했고,  
dist파일도 nginx 이미지의 `/usr/share/nginx/html:ro` 경로에 포함하여 배포했다.

결과적으로 클라우드 서버에서 해당 이미지를 실행했을 때, 페이지는 띄워졌고, 브라우저에서 페이지를 불러올 때 전보다 빠른 속도로 불러올 수 있었다.

여기서 끝인줄 알았으나..

### 중첩된 nginx port

nginx 도커 이미지에 client dist 파일을 포함하고 nginx-green, nginx-blue로 관리하려고 했으나,  
nginx의 80포트는 호스트에 의해 외부에 직접 노출되는 포트로 중복이 불가능 했다..

이에 대해 더 방법을 찾아볼지, 전에 고민했던 방식대로 nginx 이미지가 client dist파일을 마운트하는 방식을 사용할 지를 고민했는데,  
버전 관리가 불가능하다는 점을 제외하면 위 문제를 해결할 수 있고,  
빠르게 도입하여 메인 기능 구현에 집중하기 위해 새로운 방법을 찾지 않고, 바로 이 방식을 적용했다.

최신 버전 배포 시 nginx는 다시 배포할 이유가 없었기 때문에  
GitHub Actions에서 완전히 제거되었다.

> 서버에서 docker-compose-nginx.yml로 실행

build한 client dist 파일에 대해서는 서버의 특정 디렉토리로 scp 명령어를 통해 전송하였다.

실행중인 nginx가 전송된 dist파일을 사용하기 위해서 nginx의 default.conf 파일을 수정해주어야 했다.

```conf
server {
    listen 80;

    # ... 생략됨

    location / {
        root /usr/share/nginx/html/dist-green;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

- root는 nginx가 기본으로 제공할 웹 파일 경로
- 파일이 아닌 디렉토리를 요청 했을 때 디렉토리의 기본 반환 파일을 지정
  - 디렉토리의 기본 반환 파일이란, 해당 디렉토리 내 index.html 파일을 의미한다.
  - 지금은 nginx의 dist파일이 root기 때문에 /로 요청이 오면 dist 디렉토리 내의 index.html이 실행된다. (`dist/index.html`)
- try_files는 들어온 요청에 대해 `$uri` 파일이 존재하지 않으면 `$uri/`로 다시 찾아보고 이마저 없으면
  `/index.html`을 반환하도록 하는 설정이다.

위 세가지 설정 중 가장 중요한 설정은 `root /usr/share/nginx/html/dist-green;`이다.

이게 실제 dist를 가리켜야 작동하기 때문이다.

위 스크립트를 보면 알 수 있듯이 dist-green이 명시되어있다.  
직전 버전을 백업하기 위해 저장된 dist파일에 최신 dist파일을 덮어씌우기만 해서 관리하는 개념보다 dist-green dist-blue를 만들도록 했다.

큰 흐름은 github actions에서 client dist파일을 클라우드 서버의 `./nginx/html/dst`에 전송하여 덮어씌우고,  
deploy.sh가 실행될 때 dist파일을 복사하여
각 컬러에 맞게 dist-green이나 dist-blue로 덮어씌우게 했다.

이렇게 했을 때 방금 배포한 dist에 문제가 있을 경우 nginx가 가리키는 이미지에 따라 반대 이미지로 변경하여 대처할 수 있었다.  
예를 들어 nginx가 dist-blue를 마운트 중이라면 dist-green으로 수정하고 nginx를 reload할 수 있다.

### 기본 dist 디렉토리는 삭제하지 않는다

초기에는 GitHub Actions에서 전송받은 dist 디렉토리를 dist-green이나 dist-blue로 복사하고 제거했었는데, 제거하지 않는 방향으로 수정하였다.

그 이유는 deploy.sh에서 배포할 때 dist 디렉토리가 무조건 있다고 가정하고 dist-blue나 dist-green으로 복사하기 때문이다.

예를 들어 GitHub Actions의 scp 과정없이 서버에서 `./deploy.sh`를 실행하여 green과 blue로 전환하려고 할 때 dist 파일이 없으면 blue/green이 스위칭 되지 않고 에러가 발생하여 제거하지 않고 두게 되었다.

### health check starting...

앞 과정까지 진행 후 브라우저에서 접근했을 때, 페이지는 잘 띄워졌다.  
그런데… 동적 데이터가 전혀 보이지 않는 즉, 서버가 실행되고 있지 않은 것처럼 보여졌다.

더 정확히 얘기하면 docker logs로 본 서버 로그는 정상적으로 nestjs가 실행되었다고 나오지만, api가 작동하지 않은 문제였다.

나는 이 문제가 단순히

1. 서버가 느리게 켜지는 건가?
2. 포트에 문제가 있나?
3. 네트워크가 다른가?
4. 추가한 health check가 문제인가?

라고 생각했다.

우선은 같은 네트워크 환경인지 확인했다.

```bash
docker inspect webapp
```

![Untitled-1.png](https://github.com/user-attachments/assets/35c20720-2339-41b2-86ea-53dcc6c816f6)

이 둘은 같은 네트워크 환경에 있었다…

그럼 포트 문제인가?

이 문제를 해결하던 도중 갑자기 API가 동작하는 것을 확인했다.  
swagger 페이지도 접근이 가능했다. 우선 포트 문제는 아닐거라고 생각했다.

마지막으로 단순히 서버가 느리게 켜지는 건지에 대한 의문을 가지게 되었다.  
서버는 구현을 많이 해놓지 않아서 느리게 동작할 이유가 없었고,  
서버가 켜졌다는 로그도 정상적으로 출력되었다.

결국 아까 추가했던 health check의 문제라고 생각하여 deploy.sh를 열게 되었다.  
health체크 시 `healthy` 상태가 되었을 때 nginx 설정을 수정하도록 했는데,  
health 체크가 `starting`으로 나오던 문제로 확인했다..

그래서 함수를 생성하여 5초 간격으로 health 체크를 하도록 하고, starting이 아닌 상태가 나왔을 때 상태를 반환하기로 했다.

```sh
waiting() {
  while true; do
    health_status_server=$(docker inspect --format '{{.State.Health.Status}}' server-$1)

    if [ "$health_status_server" != "starting" ]; then
      echo "$health_status_server"
      return 0
    fi

    sleep 5
  done
}
```

결과는 브라우저 환경에서 동적 데이터를 확인할 수 있었다.  
이에 더해 docker compose 파일에도 재시도 쿨타임을 설정할 수 있다는 것이 기억나서  
docker compose 파일은 어떻게 설정이 되어있는지 확인해보게 되었다.

docker compose 파일은 1m30s로 되어 있었다..

이 부분 때문에 서버가 이미 정상적으로 실행중임에도 동적 데이터가 안 보였다가 나중에 보여지게 된 것이 아닌가라는 생각이다.

그렇게 아래처럼 수정했고, 7초마다 상태를 확인하여 보다 빠르게 서버 실행 상태를 확인할 수 있었다.

```docker
healthcheck:
  test: ["CMD-SHELL", "wget -q --spider http://server-blue:3000/api/health || exit 1"]
  interval: 7s
  timeout: 10s
  retries: 5
```

## 결과

최신 서버 프로젝트 배포 시 서버 컨테이너는 보조 컨테이너가 실행된 후 건강 상태 확인이 되면 nginx가 보조 컨테이너를 가리키게 된다.  
이후 보조 컨테이너가 메인 컨테이너가 되고, 기존의 메인 컨테이너는 종료 및 제거된다.

최신 클라이언트 프로젝트 배포 시 빌드된 dist 파일을 클라우드 서버에 scp한 후,  
nginx 컨테이너에 마운트 & reload 되어 실행된다.

이 과정에서 의도를 알 수 없는 프로젝트 파일 + docker 조합을 더 효율적으로 개선하는 경험을 가질 수 있었다.  
또한 green/blue 배포 방식을 적용하여 어떤 방식으로 무중단을 실현할 수 있는 지 감을 잡을 수 있었다.

다만, 무중단 배포에 대해 Socket 연결이 약 1초 정도 끊기는 문제가 있었고,  
기존의 메인 서버에서의 요청 처리가 끝나기를 기다리지 않고 서버를 종료하기 때문에  
이 부분을 개선해서 사용자 경험을 해치지 않도록 해야한다.

dist 관련해서도 버전관리가 가능한 방식을 찾아 적용해볼 계획이며,  
클라이언트, 서버 파트에 구분없이 GitHub Actions가 실행되면 모두 재배포하는데,  
클라이언트 디렉토리가 수정되면 클라이언트 디렉토리만,  
서버 디렉토리가 수정되면 서버 디렉토리만 재배포되도록 개선할 예정이다.
