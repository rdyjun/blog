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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/d51b95ae-3d30-4297-a851-910da7e8fef6/image.png)

## 구조를 수정하는 이유

저장 공간 및 메모리가 부족한 기존 ncloud 서버에서
웹 서버를 돌리기 위해서는 불필요한 리소스를 줄여야 했다.

기존에는 GitHub Actions에서 ncloud 서버에 접근하여,
저장된 git local storage에서 git pull 을 진행했다.
이후에 프로젝트를 도커 파일로 빌드 및 실행한다.

하지만, 도커 파일의 장점은 이미지만 가지고 어느 환경에서나 실행시킬 수 있다는 것이고,
이전까지 그 장점을 활용하지 못하다고 생각되어 구조를 수정하게 되었다.

## 수정 방안

우선 도커 이미지로 빌드하는 역할을 GitHub Actions에 분배하기로 했다.

- 이렇게 될 경우 ncloud 서버에 github 프로젝트 파일을 저장하지 않아도 된다.

위와같이 했을 때, GitHub Actions에서는 프로젝트를 불러와서 docker 이미지로 빌드하고,
Docker Hub에 push하는 과정이 필요하다.

ncloud 서버에서는 Docker Hub에서 최신 이미지를 pull하고 저장된 compose 파일을 실행시켜야 한다.

- 모노레포기 때문에 서버와 클라이언트 두 이미지로 구성이 되어있다.
- 두 이미지를 함께 실행하기 위해 compose로 관리되고 있다.

## 문제 해결 과정

### 이미지를 어떻게 올리지?

[Docker Hub에 Docker Image 올리기](https://www.notion.so/Docker-Hub-Docker-Image-140510ea038e80cfb110eb7309dc666a?pvs=21)

### 백엔드 서버가 아파한다

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/5eedd8f9-da6b-47de-827a-84d431401367/image.png)

위 사진은도커 compose 파일을 실행시키고 뜬 로그이다.

이 로그에서 `6379` 포트에 문제가 있는 것을 보고 redis 연결이 안되어있는 것을 알 수 있었다.

redis 연결을 하기 위해서는 .env 설정이 필요했다.

깃허브 저장소에는 .env 파일이 들어가지 않았기 때문에 서버에 .env 파일을 만들어주고,

가지고 있는 이미지가 .env 파일을 사용할 수 있도록 해주어야 한다고 생각했다.

검색해본 결과, env_file 이라는 옵션을 통해서 .env 파일을 매핑시켜줄 수 있었다.

그렇게 ncloud 서버에 server.env와 client.env 파일을 만들어주고,

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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/a110873e-f700-41a6-b4ff-14e6fb33019f/image.png)

redis 연결이 잘 이루어져 서버가 정상적으로 실행되었다.

### 웹이 안보인다!

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/1544ec96-0fe3-4882-bf93-ec33292e8b99/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/c58f63db-095c-44c1-ae7d-b54177af1ba0/image.png)

위 정보와 같이 서버가 제대로 실행되고 있음에도 실제 서버에 접속이 되지 않는 문제가 발생했다.

자세히 보니, `react-router-dom` 이 제대로 설치되지 않은 문제로 보인다.

생각해보니, 어제(24.11.15) 이 문제를 겪었고, 해결하기 위한 브랜치를 merge했다.

그래서 이 문제에 대해서는 dev 브랜치를 merge 하고 다시 해보면 괜찮아질 것이라고 생각했다.

### 그래서.. 됐어?

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/d583d463-ecaa-41e9-b961-6e324660dddf/image.png)

아까 있었던 오류는 없어졌다.

지금 보이는 화면에서는 문제없이 nginx, react, nest가 실행되는 것을 볼 수 있다.

이때 당시에는 웹 브라우저 접근이 될 줄 알았다…

하지만 어김없이 cloudflare의 Error 페이지만 보였다..

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/365750d7-5807-4156-8a67-79cca8445cbe/image.png)

원인이 뭘까를 고민해봤다.

- server 또는 client 프로젝트가 잘못됐다
  - 이건 내가 보기엔 아니었다.
  - 로그가 정상적으로 출력되었기 때문이다.
- 방화벽?
  - 배포 방식을 바꾸기 전에는 잘 접속됐던 것을 생각해보면, 방화벽은 원인이 아니다.
- 그럼 nginx가 문제인가?
  - 그렇다. nginx가 문제였다.

### nginx 이미지는 뭐가 문제일까?

가장 먼저 이 문제를 이해할 수 없어서 GPT에게 질문했다.

docker ps 결과를 주고 문제를 찾아달라고 했더니,

```bash
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS         PORTS                NAMES
5f3f397d0de4   rdyjun/inear-nginx:latest    "/docker-entrypoint.…"   5 seconds ago    Up 4 seconds   0.0.0.0:80->80/tcp   nginx
24dd46260274   rdyjun/inear-server:latest   "docker-entrypoint.s…"   5 seconds ago    Up 4 seconds   3000/tcp             server-green
4c464b8d4863   rdyjun/inear-client:latest   "docker-entrypoint.s…"   37 minutes ago   Up 4 seconds   5173/tcp             client
```

위 문장에서 PORTS를 보고 호스트에 노출되지 않은 상태라고 했다.

그리하여 docker-compose.yml 파일에 expose를 지우고 ports: 3000:3000과 ports: 5173:5173을 추가했다.

여전히 작동하지 않았다.

> 이 부분에 있어서는 배포 방식을 수정하기 전에도 이 방식으로 배포했었기에, 괜찮을 것이라고 생각하고 다시 돌려 두었다.

그러면 도대체 뭐가 문제지?

이런저런 작업을 해보려고 명령어를 입력하던 중에 ls를 입력하게 되었다.

그런데, 원래 없었던 nginx 파일이 생겼던 것이다.

그렇게 원래 저장소에 기록한 nginx 파일을 덮어씌워보면 어떨까라는 생각을 하게 되었다.

결과적으로는… 성공했다..!

Dockerfile 내부에 COPY nginx.conf만 작성되어있는 것을 확인했다.

```bash
FROM nginx:alpine

COPY nginx.conf
```

nginx.conf는 이미지에서 가지고 있지만, `conf.d/default.conf` 가 누락되어 그런 문제였다.

docker compose를 확인해보니 아래와 같이
로컬의 `./nginx/conf.d`를 이미지에 가져와, `/etc/nginx/conf.d:ro` 와 같이 사용하는 것을 볼 수 있다.

```jsx
volumes:
  - ./nginx/conf.d:/etc/nginx/conf.d:ro
```

이러한 원인으로 test 디렉토리에 nginx 디렉토리가 존재하지 않았고,
`/etc/nginx/conf.d/default.conf` 가 초기화되어 제대로 nginx가 역할을 수행하지 못했던 것이다.

이 내용에 대한 증명은, 기존 `nginx/conf.d/default.conf` 파일을 그대로 test 디렉토리로 복사하고,

docker compose를 실행시킨 결과로 볼 수 있었다.

결과는 제대로 페이지가 나오게 되었다.

### 이러면 또 고민해야 할 게 있을텐데..

무중단 배포를 위해서 서버 이미지는 그린/블루 배포 방식을 사용하기로 했다.

그런데 클라이언트는? 클라이언트도 똑같이 그린/블루 배포 방식처럼

5173/5174로 나누어서 배포해야 한다.

물론 클라이언트 특성상 꼭 그린/블루로 나누지 않아도 된다.

GitHub Actions에서 build한 클라이언트 파일을 서버로 복사하고,

해당 파일이 nginx에서 정적 파일로 사용되기만 하면 된다.

다만, 이 방식을 사용했을 때의 문제점은 버전관리가 어렵다는 것이다.

우선은 가능한 커밋 버전에 맞추어 롤백할 수 있도록

클라이언트 프로젝트 파일도 5173 / 5174로 포트를 바꾸어 진행해볼 계획이다.

### 바뀌어버린 redis 포트

서버를 3001번 포트로 실행하려고 deploy.sh에 서버 env 파일의 PORT:3000을 PORT:3001로 바꾸는 스크립트를 작성했다.

그런데, PORT:를 파싱하는 과정에서 REDIS_PORT:도 파싱되어 포트가 바뀌는 문제가 있었다 하하;;

이후 과정에서 PORT를 바꾸어서 실행하지 않아도 되어서 이 부분이 제거되었고, 결국 해결되었다.

### 포트는 의미없다..

클라이언트와 서버를 그린/블루로 띄우려면, 서로 다른 포트로 실행해야 한다.

그 이유는 이미 한 서버가 실행중인 상태에서 동시에 다른 서버를 띄우고,

트래픽을 옮기고, 트래픽이 옮겨졌을 때, 기존 서버를 종료하는 방식이기 때문이다.

지금의 구조에서는 서버가 3000포트이고, 클라이언트가 5173 포트를 쓴다.

그래서 1씩 더한 3001, 5174를 쓰려고 했다.

다만, 여기서의 문제가 server에서는 .env에서 `PORT:3000`, `PORT:3001`과 같이 수정해서

실행하면 해당 포트로 실행이 되지만, 클라이언트는 이미지에 포함되어있는 설정 파일들을 건드려야 했다….

그리하여 서버와 같이 PORT 번호를 수정하지 않고, 도커 컨테이너의 외부 포트, 내부 포트를 건드려 보기로 했다.

docker-compose 파일의 옵션으로 `expose`와 `ports`가 있었다.

expose는 컨테이너간 연결하기 위한 포트이고

ports는 실제 외부에서 접근할 포트를 설정할 수 있었다.

우선은 nginx와만 소통해야 하기 때문에 ports는 건드리지 않기로 했다.

그렇게 expose에서 5174:5173을 시도해봤다.

nginx의 client port 설정을 바꿔줬음에도 bad gateway가 나왔다.

이 부분을 ports에 적용해도 마찬가지였다.

위 시도에서의 문제점은 expose는 port:port 구조가 불가능하다는 것이다.

port:port 구조에서의 앞 port는 호스트(물리 컴퓨터)가 외부에 열어두는 포트이고

뒤의 port가 내부 3000번 포트를 가리키는 구조이다.

결과적으로 위 방식은 오랜 시간 끝에 불가능하다는 것을 인지했다…

다만, 계속해서 하다보니 한 가지 깨달은 것이 있었다.

각 컨테이너가 서로 다른 포트가 아니어도 된다는 사실이다.

이 사실을 알고 해결할 수 있다는 안도감과 이제서야 알았다는 자괴감을 느꼈지만

시간이 없기에 빠르게 적용해봤다.

결과적으로 client-blue, client-green, server-blue, server-green 4개의 컨테이너를 각각

3000포트와 5173 포트로 열 수 있었다… 🎉

이게 가능한 이유는, 각 컨테이너가 네임스페이스로 관리되기 때문에 포트가 충돌되지 않는다는 것이다.

그렇게 서브 클라이언트와 서브 서버 컨테이너를 띄우고 실행중이던 서버와 클라이언트 컨테이너를 종료하게끔 작성하였다.

### 뒤바뀌어 버린 client 배포 방식…

위에서 클라이언트 프로젝트 파일을 5173/5174로 한다고 했었다.
이걸 실제로 구현했으나 한 가지 문제가 있었다.

페이지가 너무 느리게 로드된다.

사용자에게 있어서 흰 화면을 2초 이상 노출하는 것이 썩 좋은 경험은 아니라고 생각했다.

결국 도커 허브에서의 버전관리를 유지하면서 서버를 유지하겠다는 꿈은 무너졌다.

그렇게 방법을 찾다가 서버에서 5173번 포트로 클라이언트 파일을 실행시키는 것이
일반적인 선택은 아니라는 것을 알게되었다.

그렇게 client 이미지를 지우기로 결정했다.

그러면 여기서 갈림길이 생긴다.

1. scp로 넘겨서 실행중인 nginx가 해당 파일을 가리키게 한다.
2. nginx 이미지에 client 정적 파일을 포함시키고, 서버처럼 blue/green 기법 사용

이전의 선택의 갈림길에서도 scp가 버전 관리가 안되기 때문에

nginx 이미지에 포함시키고 이 이미지를 버전으로 관리하려고 했다.

그렇게 docker actions 스크립트에 client 파일을 build 시키고 nginx Dockerfile에서 COPY 했다.

> client를 빌드시킬 때 yarn --cwd로 해당 디렉토리에서 실행되도록 했다.
> 그리고 yarn build를 위해서 yarn install이 선행되어야 했다.

결과적으로 페이지가 띄워졌고, 브라우저에서 페이지를 불러올 때 전보다 빠른 속도로 불러올 수 있었다.

여기서 끝인줄 알았으나..

### 중첩된 nginx port

nginx에 client 이미지를 포함하고 nginx-green, nginx-blue로 관리하려고 했으나,

nginx의 80포트는 호스트에 의해 외부에 직접 노출되는 포트로 중복이 불가능 했다..

이에 대해 더 방법을 찾아볼지, nginx에서 client 파일을 분리할 지를 고민했는데,

nginx와 독립적으로 돌아가는 것이 문제 해결도 가능하고, 구조상으로 더 나을 것 같아서 분리하기로 하였다.

nginx 이미지는 딱히 코드 파트에서 바뀔 부분이 없다고 생각되어 github actions에서 완전히 제거하였다.

> 서버에서 docker-compose-nginx.yml로 관리

build한 client dist 파일에 대해서는 서버의 특정 디렉토리로 scp 명령어를 통해 전송하였다.

실행중인 nginx가 전송된 dist파일을 사용하기 위해서 nginx의 default.conf 파일을 수정해주어야 했다.

- nginx가 기본으로 제공할 웹 파일 경로
- 파일이 아닌 디렉토리를 요청 했을 때 디렉토리의 기본 반환 파일을 지정
  - 예를 들어 /경로도 디렉토리이다. /에대한 기본 반환 파일을 index.html로 지정하면
    각 디렉토리로 요청이 왔을 때, 디렉토리 내부의 index.html을 반환하는 구조이다.
  - 지금은 nginx의 dist파일이 root기 때문에 /로 요청이 오면 dist 디렉토리 내의 index.html이 실행된다.
- try_files는 들어온 요청에 대해 `$uri` 파일이 존재하지 않으면 `$uri/`로 다시 찾아보고 이마저 없으면
  `/index.html`을 반환하도록 하는 설정이다.

위 세가지 설정 중 가장 중요한 설정은 `root /usr/share/nginx/html/dist-green;`이다.

이게 실제 dist를 가리켜야 작동하기 때문이다.

```bash
location / {
    root /usr/share/nginx/html/dist-green;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
}
```

위 스크립트를 보면 알 수 있듯이 dist-green이 명시되어있다.

직전 버전을 백업하기 위해 dist파일을 덮어씌우는 개념보다 dist-green dist-blue를 만들도록 했다.

큰 흐름은 github actions에서 nginx의 dist파일로 전송하고, deploy.sh에서 dist파일을 cp하여
각 컬러에 맞게 dist-green이나 dist-blue로 덮어씌우게 했다.

이렇게 했을 때 클라이언트에 문제가 있을 경우 dist-blue를 실행중이라면 dist-green을 실행하고 dist-blue를 종료할 수 있다.

초기에는 dist-green이나 dist-blue로 cp하고 rm했었는데, 그렇게 하지 않은 이유는

deploy.sh에서 실행될 때 dist 디렉토리가 무조건 있다고 가정하고 blue나 green으로 복사하는데,

GitHub Actions의 scp 없이 서버에서 green과 blue로 전환할 때 위와같은 이유로 dist 파일이 필요해서 rm하지 않고 두게 되었다.

### 서버는 왜 또 말썽인가

위 과정 후 브라우저에서 접근했을 때, 페이지는 잘 띄워졌다.

그런데… 서버가 안 띄워지는 것 처럼 보였다..

정확히 얘기하면 docker logs로 본 서버 로그는 정상적으로 nestjs가 실행되었다고 나오지만,

api가 작동하지 않은 문제였다.

나는 이 문제가 단순히

1. 서버가 느리게 켜지는 건가?
2. 포트에 문제가 있나?
3. 네트워크가 다른가?

라고 생각했다.

우선은 같은 네트워크 환경인지 확인했다.

```bash
docker inspect webapp
```

![Untitled-1.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2b7fd40e-1f24-4d91-a1a0-590a51716cd1/494fa800-0b74-456f-8772-7836e6de8627/Untitled-1.png)

이 둘은 같은 네트워크 환경에 있었다…

그럼 포트 문제인가?

사실 포트 문제인지 확인하기 전에 갑자기 swagger의 /api 가 접속되는 것을 확인했다. 포트의 문제는 아니라고 생각했다.

마지막으로 서버가 느리게 켜지는 건지에 대한 의문을 가지게 되었다.

서버는 구현을 많이 해놓지 않아서 느리게 동작할 이유가 없었고,

서버가 켜졌다는 로그도 정상적으로 출력되었다.

그래서 일단 파일을 하나씩 열었다. 그러다가 deploy.sh를 열게 되었다.

health체크 후 정상적으로 health 체크가 되었을 때 nginx 설정을 수정하도록 했는데,

health 체크가 starting으로 나오던 문제였다..

그래서 함수를 생성하여 5초 간격으로 health 체크를 하도록 하고, starting이 아닌 상태가 나왔을 때

이 함수 값을 반환하기로 했다.

결과는 브라우저 환경에서 /api 를 확인해볼 수 있었다.

이에 더해 docker compose 파일에도 재시도 쿨타임을 설정할 수 있다는 것이 기억나서

docker compose 파일은 어떻게 설정이 되어있는지 확인해보게 되었다.

docker compose 파일은 1m30s로 되어 있었다..

이 부분 때문에도 서버가 늦게 시작된 것이 아닌가라는 생각이든다.

## 결과

server 컨테이너는 실행되고 얼마 지나지 않아 healthy한 상태로 바뀌게 되었다.

또한 blue/green 변경 과정에서 health 체크 후 기존 서버를 닫게끔 작성했다 보니,

request도 끊임없이 전달되었다.

프론트 또한 설정 파일을 바꾸고 reload하는 즉시 적용되기 때문에 문제 없이 배포가 되고 있다.
