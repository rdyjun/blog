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

# WAS server와 WAS

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

```bash
> Server webmaster@localhost
> DocumentRoot /var/www/html # 이 부분 수정
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

2-1. Permission 오류
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

### 4. 확인

```bash
vi /etc/apache2/mods-available/jk.conf
```

...추가중
