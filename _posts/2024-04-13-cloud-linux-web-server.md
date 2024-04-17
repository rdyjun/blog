---
title: "리눅스를 활용한 웹 서버 익히기"
excerpt: ""

categories: [web]
tags: [web, was, server, apache, tomcat]

toc: true
toc_sticky: true

date: 2024-04-13 20:00:00 +0800
last_modified_at: 2024-04-13 20:00:00 +0800

pin: false
---

# Web server와 WAS

- Web Server의 역할
  - static resource
    - <u>Web Server</u>에서 `static resource`를 다루고,
      <u>WAS Server</u>에서 `dynamic resource`를 다룬다.
  - Security
    - <u>방화벽 바깥</u>에 `Web Server`를 위치하고 `WAS`, `DB서버`는 <u>방화벽 안쪽</u>에 위치
    - SSL 처리
  - Load balancing
    - 하나의 web server가 여러 개의 WAS 서버에게 업무 분배

- Web Server의 장점
  - Static resource 처리를 빨리 해준다
  - WAS 서버의 주소 노출이 안되어 보안이 강화됨 (port 번호 숨길 수 있다)
  - 여러 개의 WAS의 업무를 분배해 주어 자원을 효율적으로 활용

## 웹 서버 설치하기

### 1. Apache 설치하기

```bash
apt-get update # apache2 설치 전 apt-get 업데이트
apt-get install apache2
```

### 2. 설치 후 버전 확인

```bash
apache2 -v
```

### 3. Apache 서비스 가동/중지/상태 확인

service

```bash
service apache2 start
service apache2 stop
service apache2 restart
```

systemd - 더 권장되는 방식

```bash
sudo systemctl start apache2
sudo systemctl restart apache2
sudo systemctl stop apache2
sudo systemctl status apache2 # q를 통해 빠져나올 수 있음
```

### 4. 브라우저를 통한 웹 서버 동작 확인

4-1. Linux 내 FireFox 실행
4-2. 주소창에 `http://localhost/` 또는 `http://127.0.0.1/` 입력

> 아래와 같이 나왔다면 정상

![alt text](/assets/img/apache-1.png)

### 5. netstat을 통한 웹 서버 동작 확인

기본 포트인 ntlp가 정상적으로 작동하는지 확인

```bash
netstat -ntlp
```

![alt text](/assets/img/netstat-1.png)

### 6. Apache Server의 home directory 변경 (선택)

- 내 홈페이지 root로 변경해야 함
- 설정 후 apache2 재시작

```bash
vi /etc/apache2/sites-available/000-default.conf
```

```conf
Server webmaster@localhost
DocumentRoot /var/www/html # 이 부분 수정
```

재시작

```bash
sudo systemctl restart apache2
```

#### 수정한 홈페이지 보여주기

- 새로 지정한 DocumentRoot 폴더의 권한을 일반 사용자에게 read, execute 권한 주기
- 만약 DocumentRoot를 /home/abc/html로 했을 때, 모든 경로에 일반 사용자에게 read와 execute 권한을 준다.

```bash
chmod 755 /home/abc
chmod 755 /home/abc/html
chmod 755 /home/abc/html/index.html
```

### 7. 각종 오류

#### 문제 1. Permission 오류

![alt text](/assets/img/apache-2.png)

#### 문제 2. DocumentRoot를 바꿨음에도 이전 페이지가 나올 경우

- Apache2 재시작 (DocumentRoot 설정을 바꿨기 때문에 필수)
- Cache 지우기
- 재시도

#### 문제 3. 아무것도 나오지 않는 경우

- `https://`가 아닌 `http://` 로 다시 접속

### 8. 오류 해결 방법

#### 8-1. Apache error 확인

```bash
  tail /var/log/apache2/error.log # tail은 가장 마지막에 작성된 내용 확인
```

![alt text](/assets/img/apache-4.png)

#### 8-2. 방화벽 해제

```bash
ufw allow 80
```

### 9. 다른 홈페이지 보여주기

1. DocumentRoot 폴더 하위에 새로운 디렉토리 `project` 만들기 (폴더명은 원하는 이름으로)
2. 그 안에 `a.html` 만들기
3. 파이어 폭스 경로에 `http://localhost/project/a.html` 입력 후 접속

## WAS 설치

### 1. Tomcat9 설치

```bash
apt-get install tomcat9*
```

### 2. Tomcat 실행/중지/상태 확인

```bash
service tomcat9 start
service tomcat9 stop
service tomcat9 status
```

```bash
sudo systemctl start tomcat9
sudo systemctl restart tomcat9
sudo systemctl stop tomcat9
sudo systemctl status tomcat9
```

### 3. 설치 확인

- `http://localhost:8080` - 8080은 tomcat에서 정한 port
- tomcat 기본 홈페이지 경로 `/var/lib/tomcat9/webapps/ROOT/index.html`

### 4. admin 페이지 접근

Tomcat-admin - GUI 관리도구

- Tmcat에 app을 deploy, 설정 변경
- `localhost:8080/manager`

#### manager 페이지를 위한 권한 부여

```bash
<role rolename="manager-gui" />
<user username="myid" password="mypassword" rolename="manager-gui" />
```

![alt text](/assets/img/tomcat-1.png)

### 5. App을 Tomcat에서 Deploy하는 방법

1. Tomcat Admin을 사용하여 Deploy
2. 직접 App을 폴더에 넣어서 Deploy
   - /var/lib/tomcat9/webapps/ 내에 war파일을 위치시킴
   - Tomcat이 압축을 해제하여 directory 형태로 변경함

https://tomcat.apache.org/tomcat-9.0-doc/appdev/sample
위 경로에서 예제 파일 다운로드 후 2번 방법과 같이 `/var/lib/tomcat9/webapps`에 저장

### 6. WAR(Web Application aRchive)

- Web Application을 묶는 확장자
- Application 실행을 위한 컴파일 된 모든 클래스 파일, 설정 파일들이 모두 포함
- Web Application 설정을 정의한 배포 명세서 (web.xml) 존재

### 7. Deploy된 Application 확인

사용자가 Deploy한 Application

```bash
ls /var/lib/tomcat9/webapps
```

사용자가 Deploy한 Application 및 Tomcat 기본 Application

```bash
ls /var/lib/tomcat9/work/Catalina/localhost
```

### 8. Tomcat 서버 설정

1. `/var/lib/tomcat9/conf/server.xml`
2. Tomcat 접속 port 확인
3. Connector? Port? Protocol?

```xml
<Connector port="8080" protocol="HTTP/1.1"
            connectionTimeout="20000"
            redirectPort="8443" />
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

## Apache와 Tomcat 연결

Apache <-> Connector <-> Tomcat

1. apache-tomcat connector에 Tomcat들의 별명을 지어주고, 정보 설정
   - Connector 설정 파일 - `/etc/libapache2-mod-jk/workers.properties
     - <u>Tomcat</u>의 `home directory`, `port`, `통신 protocol`을 설정
2. Web Server Apache에 apache-tomcat connector에 설정된 Tomcat으로 요청을 보내도록 설정(별명 사용)
   - `/etc/apache2/sites-available/000-default.conf`
   - `/etc/apache2/mods-available/jk.conf`
   - `/etc/apache2/mods-available/jk.load`
3. WAS Tomcat에 Apache에서 오는 요청을 받도록 설정
   - 특정 `protocol`과 특정 `port`로 오는 요청을 받도록 설정
     - 실습: AJP1.3으로 연결되고 port 8009로 오는 요청을 받도록 설정
     - 실습을 위한 포트 수정 파일 위치는 `/var/lib/tomcat9/conf/server.xml`
   - Deploy할 App 위치 - `/var/lib/tomcat9/webapps`

### 1. 모듈 설치

```bash
sudo apt-get install libapache2-mod-jk*
```

### 2. apache-tomcat connector에 Tomcat들의 별명 지어주기

```bash
vi /etc/libapache2-mod-jk/workers.properties
```

```properties
workers.tomcat_home=/var/lib/tomcat9
workers.java_home=/usr/lib/jvm/default-java
worker.list=ajp13_worker
worker.ajp13_worker.port=8009
worker.ajp13_worker.host=localhost
worker.ajp13_worker.type=ajp13

# 아래 항목들 다 주석처리
```

수정 후 재시작

```bash
sudo systemctl restart apache2
```

### 3. webserver apache에 apache-tomcat connector에 설정된 tomcat으로 요청을 보내도록 설정(별명 활용)

```bash
vi /etc/apache2/sites-available/000-default.conf
```

```conf
<VirtualHost *:80>

      ServerAdmin webmaster@localhost
      #DocumentRoot /var/www/html
      JkMount /* ajp13_worker # 2번에 나온 Tomcat 별명(ajp13_worker)
```

설정 변경 후 재시작

```bash
sudo systemctl restart apache2
```

### 4. 확인

```bash
cat /etc/apache2/mods-available/jk.conf
cat /etc/apache2/mods-available/jk.laod
```

### 5. WAS Tomcat에 특정 Port와 특정 Protocol로 오는 요청을 받도록 설정

```bash
vi /var/lib/tomcat9/conf/server.xml
```

아래 내용 주석 해제

```xml
<Connector  protocol="AJP/1.3"
            port="8009"
            redirectPort="8443" />
```

#### 사용자 지정

> +는 새로 추가된 줄로 실제 작성할 때는 +는 무시하자

```xml
<Connector  protocol="AJP/1.3"
            address="0.0.0.0"           +
            port="8009"
            redirectPort="8443"
            pacaketsize="65536"         +
            secretRequired="false" />   +
```

마찬가지로 설정 변경 후 재시작

```bash
sudo systemctl restart tomcat9
```

#### 정상 작동 확인

- 주석만 해제하고 재시작한 경우
  `http://localhost` 로 접속 시 `service unavailable error, 503 error` 가 뜬다면 정상
- 주석 제거 후 내용을 추가한 경우
  `http://localhost` 와 `http://localhost:8080` 의 접속 결과가 동일 - <u>Tomcat 화면</u>

## 여기서 궁금증

1. Apache와 Tomcat이 다른 Server에 있는 경우?
   - Tomcat의 host를 나타내는 ip인 `127.0.0.1(or localhost)`를 Tomcat 서버의 ip로 변경
2. Apache와 Tomcat을 연동했는데, http://localhost 했을 때, Tomcat으로 연결이 안되는 경우?

   - `https://`로 한 건 아닌지 확인
   - `netstat -ntlp` 로 8009 포트가 열려있는지 확인
   - Tomcat이 구동중인지 `systemctl status tomcat9` 확인

3. `http://localhost:8009` 가 안되는 이유

4. Apache의 DocumentRoot에 있는 index.html이 동작하지 않는 이유

## WAS가 여러 개이고, 각각 다른 App이 있는 경우

```
        / WAS1
Apache -  WAS2
        \ WAS3
```

### 1. Connector 설정

```bash
vi /etc/libapache2-mod-jk/workers-properties
```

```properties
workers.tomcat_home=/var/lib/tomcat9
workers.java_home=/usr/lib/jvm/default-java

worker.list=was1,was2,was3
worker.was1.port=port1(해당 port)
worker.was1.host=ip1(해당 ip)
worker.was1.type=ajp13

worker.was2.port=port2(해당 port)
worker.was2.host=ip2 (해당 ip)
worker.was2.type=ajp13
```

### 2. 특정 경로에 각 Tomcat Mount 하기

```bash
vi /etc/apache2/sites-available/000-default.conf
```

```conf
<VirtualHost *:80>
              ServerName 127.0.0.1
              ServerAdmin webmaster@localhost
              DocumentRoot /var/www/html
              JkMount /app1/* was1
              JkMount /app2/* was2
              JkMount /app3/* was3
</VirtualHost>
```

여기서 JkMount의 사용법은 `JkMount {URL_PATTERN} {WORKER}` 이다.

- `URL_PATTERN` 주소로 요청이 들어왔을 때 `WORKER`로 보낸다는 의미

#### 여기서 잠깐!

위에서 주석처리 해둔 DocumentRoot를 주석 해제한 이유?

- 서버의 정적 파일을 액세스 하기 위해 DocumentRoot를 사용해야하기 때문이다
  - 즉, `/app1/*`, `/app2/*`, `/app3/*` 외의 경로로 서버의 정적 파일을 확인하고자 할 때 DocumentRoot 설정이 필요하기 때문이다.
- 기본 값으로 `/var/ww/html` 를 설정
- 꼭 DocumentRoot를 설정해야하는 것은 아니다. 위 3가지 경로만 액세스하게 하고싶다면 주석처리된 상태 그대로 두어도 된다.

### 3. 테스트

#### 현재 상태

```
Linux-1         Linux-2
apache
  |   \------▷ Tomcat
  ▽             WAS-2 = app2
Tomcat
 WAS-1 = app1
```

#### 접속

아래 경로 모두 잘 나오는 것을 확인

1. `http://localhost/`
2. `http://localhost/app1/`
3. `http://localhost/app2/`
4. `http://{1번 LINUX IP}.001/app1/`
5. `http://{2번 LINUX IP}.001/app2/`

## Load Balancing

- `네트워크 트래픽`을 하나 이상의 `서버`나 `장비`로 분산하기 위해 사용되는 기술이다
- S/W나 H/W를 통해 로드 밸런싱을 수행할 수 있다.
- 서비스 사용자의 요청을 서버로 분산해서 전달한다

```
사용자1 →―――――――――――――┐      ┌――――――→ Tomcat1 (00.00.00.1)
                      ↓     ↑
사용자2 →――――――――→ Apache Server ―――→ Tomcat2 (00.00.00.2)
                   00.00.00.00
                      ↑     ↓
사용자3 →―――――――――――――┘      └――――――→ Tomcat3 (00.00.00.3)
```

### 웹 트래픽 증가에 따른 대응 방법

#### 1. Scale Up

- CPU/RAM/Disk 성능 및 Network 대역폭 등 증가
- 비싸고 성능이 좋은 서버로 변경

```
             ┌────┐
     ┌──┐    │    │
□ -> └──┘ -> └────┘
```

#### 2. Scale Out

- 로드 밸런싱과 함께 활용
- 부하를 처리할 서버 대수를 늘리는 방식
- 저렴한 서버 여러 대를 이용해 더 많은 부하를 감당한다

```
ㅁ + ㅁ + ㅁ
```

- `같은 그룹`에 있는 `서버`들에게 요청이 `정책`에 맞게 골고루 분배될 수 있도록 한다.
- `같은 그룹`에 있는 `서버`들은 동일한 기능을 `서비스`한다

### 로드 밸런싱 방식

#### 1. Round Robin

- 로드 밸런서에서 서버 선택 시, `순차적`으로 서버를 선택하는 방법

```
==============================
          1st Request
                       apache1
                      /
user -> load balancer  apache2

                       apache3

==============================
          2nd Request
                       apache1

user -> load balancer ━ apache2

                       apache3

==============================
          3rd Request
                       apache1

user -> load balancer  apache2
                      \
                       apache3
==============================
```

#### 2. Weighted Round Robin

- 로드 밸런서에서 서버 선택 시 `비중`에 따라서 서버를 선택하는 방식

```
user1                        apache1 (weight 0.5)
                          /
user2 ----> load balancer    apache2 (weight 0.2)

user3                        apache3 (weight 0.3)
```

추가중...
