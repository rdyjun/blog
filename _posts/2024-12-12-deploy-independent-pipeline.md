---
title: "모노레포 배포 파이프라인 분리하기"
author: rdyjun
excerpt: "독립적 배포 파이프라인 구축하기"

categories: [docker]
tags: [Docker, nginx, compose, deploy, github, actions]

toc: true
toc_sticky: true

date: 2024-12-13 15:39:00 +0800
last_modified_at: 2024-12-13 15:39:00 +0800

pin: false
---

[게시글, 처음 사용해본 도커로 배포 리팩토링 해보기](https://rdyjun.github.io/posts/boostcamp-docker-updated/)

지난번에 도커 자동 배포 방식을 개선했었는데, 모노레포였기 때문에 push했을 때 수정된 파일과 관계없이 클라이언트 프로젝트와 서버 프로젝트를 모두 다시 배포하는 상태였다.  
하지만 커밋 변경사항에 따라 클라이언트 프로젝트와 서버 프로젝트를 독립적으로 배포할 수 있을 것 같아 개선하게 되었다.

## 흐름

아래 코드는 수정 전 코드이다.  
여기에 server 파일과 client 파일이 수정되었는 지 boolean 형태의 환경변수를 저장하는 로직을 추가하고,  
해당 변수에 따라 배포되거나 배포되지 않도록 수정해보려고 한다.

```yml
name: Docker Image CD

on:
  push:
    branches: ["dev"]

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      # 소스 코드 체크아웃
      - name: Checkout code
        uses: actions/checkout@v3

      # Docker Hub 로그인
      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # SSH 접속을 위한 SSH 키 설정
      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      # 원격 서버의 SSH 호스트 키를 신뢰할 수 있는 목록에 추가
      - name: Add known hosts
        run: ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts

      # 서버 이미지 빌드 및 태그 지정
      - name: Build and tag Docker image
        run: |
          docker build -f server/Dockerfile -t ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }} ./
          docker tag ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/inear-server:latest
      # 클라이언트 및 nginx 이미지 빌드 및 태그 지정
      - name: Build and tag client image
        run: |
          yarn --cwd ./client install
          echo ${{ secrets.CLIENT_ENV_PRODUCTION }} > ./client/.env.production
          echo ${{ secrets.CLIENT_ENV_DEVELOPMENT }} > ./client/.env.development
          yarn --cwd ./client build
          scp -r ./client/dist/ ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }}:/${{ secrets.SSH_USER }}/web18-inear-test/nginx/html/
      # Docker Compose로 전체 애플리케이션 빌드 및 실행 (테스트용)
      - name: Run Docker Compose for testing
        run: |
          docker compose up --build -d
        continue-on-error: false

      # 서버 이미지 푸시 (Docker Hub 또는 다른 레지스트리)
      - name: Push server image to Docker Hub
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }}
          docker push ${{ secrets.DOCKER_USERNAME }}/inear-server:latest
      # 서버에서 파일을 최신화 하고, 배포 스크립트를 통해 실제 배포 진행
      - name: Deploy with Docker Compose on server
        run: |
          ssh ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} " \
            mkdir -p /${{ secrets.SSH_USER }}/web18-inear-test && \
            cd /${{ secrets.SSH_USER }}/web18-inear-test && \
            docker system prune -af && \
            docker pull ${{ secrets.DOCKER_USERNAME }}/inear-server:latest && \
            ./deploy.sh
          "
```

## 프로젝트 수정 여부 확인

### 어떤 방식으로 수정 여부를 확인할 것인가?

GitHub Actions에서 사용 가능한 여러 액션을 사용할지, `git diff`를 사용해서 비교할지 고민했다.

GitHub Actions에서 지원되는 액션들을 사용하면 편리는 하겠지만, 어떤 원리에서 동작하는 지 깊이 알지 못하고,  
어떤 이슈들이 있을 수 있을지 모르기 때문에 git diff를 적용해보기로 결정했다.

### git diff를 사용해 수정 여부 확인

처음엔 프로젝트가 Squash merge 방식으로 운영되어왔기 때문에 단순하게 `git diff HEAD^ HEAD` 명령으로 비교하면 될 것이라고 생각했다.

하지만 간혹 dev에 바로 merge하는 일이 있어서 위 명령을 사용하지 않기로 했다.  
그럼 push 전 코드와 push 후 코드를 어떻게 비교할 것인지를 고민해야 했다.

```yml
- name: Checkout code
  uses: actions/checkout@v3
```

위 방식으로 깃허브 코드를 가져오면 가장 최신 커밋만 가져오게 되어, 비교가 불가능했다.  
그래서 `with: fetch-depth: 0`옵션을 추가했다.

> 이 옵션을 추가했을 때 모든 커밋을 가져온다는 의미이다.

```yml
- name: Checkout code
  uses: actions/checkout@v3
  with:
    fetch-depth: 0
```

자 이제 모든 커밋을 가져왔다. 하지만 아직 push 전과 push 후를 확인할 방법을 모른다.  
검색을 해보니, GitHub Actions에서 `github.event.before`라는 컨텍스트 변수를 지원해주는 것을 알 수 있었다.

> `github.event.before`는 push하기 전 최신 커밋 해시값을 가지는 변수이다.

이를 통해 git diff를 아래와 같이 쓸 수 있었다.

> quiet 옵션은 종료 코드를 반환하도록 하는 옵션이다.  
> 여기서는 수정사항이 있을 때 1을 반환하고, 없을 때 0을 반환한다.

```sh
git diff --quiet ${{ github.event.before }} HEAD client
git diff --quiet ${{ github.event.before }} HEAD server
```

이제 이 값을 다른 job에서 쓰기 위해 변수로 저장하여야 한다.  
아래와 같이 `$GITHUB_ENV(환경 변수파일)`에 저장하면 다른 job에서도 쓸 수 있다.

```sh
echo "CLIENT_FILE_CHANGED=$CLIENT_FILE_CHANGED" > $GITHUB_ENV
echo "SERVER_FILE_CHANGED=$SERVER_FILE_CHANGED" >> $GITHUB_ENV

# 사용법
${{ env.SERVER_FILE_CHANGED }}
```

이제 기본적으로 클라이언트 파일과 서버 파일이 수정되었는 지에 대해 변수에 저장하고 관리되고 있다.  
하지만 여기서 추가로 고민했던 점은 두 파일이 수정되지 않고 루트의 설정 파일만 수정되었을 때의 시나리오이다.

만약 루트의 설정 파일을 merge하지 않아 hotfix로 merge한다고 했을 때,  
클라이언트와 서버 파일 중 어떤 파일을 위해 수정된 것인지 모르기 때문에 둘 다 다시 배포해주기로 결정했다.

그래서 우선 파일 수정사항을 확인한 후 로컬 변수로 저장하고,  
이 로컬 변수가 둘 다 false일 때 둘 다 true로 환경 변수 파일에 저장되도록,  
하나라도 true면 로컬 변수 그대로 환경 변수에 저장되도록 했다.

```yml
- name: Check file changes
  run: |
    CLIENT_FILE_CHANGED=$(git diff --quiet ${{ github.event.before }} HEAD client && echo 'false' || echo 'true')
    SERVER_FILE_CHANGED=$(git diff --quiet ${{ github.event.before }} HEAD server && echo 'false' || echo 'true')

    if [ "$CLIENT_FILE_CHANGED" == 'false' ] && [ "$SERVER_FILE_CHANGED" == 'false' ]; then
      echo "CLIENT_FILE_CHANGED=true" > $GITHUB_ENV
      echo "SERVER_FILE_CHANGED=true" >> $GITHUB_ENV
    else
      echo "CLIENT_FILE_CHANGED=$CLIENT_FILE_CHANGED" > $GITHUB_ENV
      echo "SERVER_FILE_CHANGED=$SERVER_FILE_CHANGED" >> $GITHUB_ENV
    fi
```

## 실제 배포 파이프라인 분리

각 프로젝트에 대해 수정 여부를 변수에 저장했다.  
이제 이 변수 값을 통해 배포를 진행할지, 말지에 대한 로직을 추가해야 한다.

앞 단계에서 정의한 변수를 통해 조건문만 달아주면 되는 문제로 어렵지 않게 수행할 수 있었다.

먼저 서버 이미지 빌드 및 도커 허브에 push하는 job에 if문을 추가하여,  
`env.SERVER_FILE_CHANGED`가 `true`인 경우에만 빌드하구 도커 허브에 업로드하도록 했다.

```yml
- name: # 서버 이미지 빌드 및 태깅
  run: |
    if [ "${{ env.SERVER_FILE_CHANGED }}" == 'true' ]; then
      docker build -f server/Dockerfile -t ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }} ./
      docker tag ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }} ${{ secrets.DOCKER_USERNAME }}/inear-server:latest
    fi

- name: # 서버 이미지 도커 허브에 push
  run: |
    if [ "${{ env.SERVER_FILE_CHANGED }}" == 'true' ]; then
      docker push ${{ secrets.DOCKER_USERNAME }}/inear-server:${{ github.sha }}
      docker push ${{ secrets.DOCKER_USERNAME }}/inear-server:latest
    fi
```

마찬가지로 클라이언트 프로젝트를 빌드하고 scp 명령을 수행하기 전에  
if문을 통해 `env.CLIENT_FILE_CHANGED`가 `true`인지 검사한 후 실행되도록 했다.

```yml
- name: # 클라이언트 프로젝트 빌드 및 scp
  run: |
    if [ "${{ env.CLIENT_FILE_CHANGED }}" == 'true' ]; then
      yarn --cwd ./client install
      echo ${{ secrets.CLIENT_ENV_PRODUCTION }} > ./client/.env.production
      echo ${{ secrets.CLIENT_ENV_DEVELOPMENT }} > ./client/.env.development
      yarn --cwd ./client build
      scp -r ./client/dist/ ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }}:/${{ secrets.SSH_USER }}/web18-inear-test/nginx/html/
    fi
```

앞 과정 이후 클라우드 서버에서 배포할 때, 파일 수정 여부에 대한 변수를 ssh 세션에 저장하고,  
해당 변수를 통해 각 프로젝트에 대해 배포 스크립트를 실행할지 결정해주었다.

```yml
- name: # 배포
  run: |
    ssh ${{ secrets.SSH_USER }}@${{ secrets.SERVER_IP }} " \
      SERVER_FILE_CHANGED=${{ env.SERVER_FILE_CHANGED }} && \ # 변수 가져오기
      CLIENT_FILE_CHANGED=${{ env.CLIENT_FILE_CHANGED }} && \ # 변수 가져오기
      mkdir -p /${{ secrets.SSH_USER }}/web18-inear-test && \
      cd /${{ secrets.SSH_USER }}/web18-inear-test && \
      docker system prune -af && \

      # 서버 파일이 수정되면 Docker Hub에서 이미지 pull 및 서버 배포 스크립트 실행
      if [ \"\$SERVER_FILE_CHANGED\" == true ]; then
        docker pull ${{ secrets.DOCKER_USERNAME }}/inear-server:latest && \
        ./backend-deploy.sh
      fi

      # 클라이언트 파일이 수정되면 클라이언트 배포 스크립트 실행
      if [ \"\$CLIENT_FILE_CHANGED\" == true ]; then
        ./frontend-deploy.sh
      fi
    "
```

## 결과

이후 일정에 따라 커밋을 너무 많이 추가하기에 무리가 있다고 생각되어,  
별도의 각 레포 수정 테스트는 진행되지 않았다.

다만 위 결과에서 루트의 `/.github/workflows` 디렉토리가 수정되었기 때문에  
클라이언트 및 서버가 모두 다시 배포되는 것으로 끝내고자 했다.

아래는 서버 이미지 빌드 및 Docker Hub로 push한 결과 사진이다.  
두 번의 job 모두 `env.SERVER_FILE_CHANGED` 변수에 `true`가 저장되어 실행되는 것을 볼 수 있다.

> 서버 이미지 빌드 결과

![서버 이미지 빌드 결과](https://github.com/user-attachments/assets/51aca995-4968-4234-b1c2-99b3c94648f8)

<br>

> 서버 이미지 업로드 결과

![서버 이미지 업로드 결과](https://github.com/user-attachments/assets/1f9dafac-8097-47f4-a874-036bfda3cadb)

<br>
<br>

아래는 클라이언트 빌드 및 scp에 대한 내용이다.  
마찬가지로 `env.CLIENT_FILE_CHANGED`가 `true`로 변환되어 실행되는 결과를 볼 수 있다.

![클라이언트 이미지 빌드 결과](https://github.com/user-attachments/assets/30d43138-d8f8-4ad4-8021-a59b3a3bb45b)

마지막으로 클라우드 서버에서의 실제 배포 결과이다.  
env에 `CLIENT_FILE_CHANGED`와 `SERVER_FILE_CHANGED` 변수가 잘 넘어간 것을 볼 수 있고,  
바로 `docker pull` 명령을 실행하는 것을 확인할 수 있다.

![클라우드 서버 배포 결과](https://github.com/user-attachments/assets/cb01dcb6-2188-4f9f-ae52-43bd137ea30a)

## 회고
