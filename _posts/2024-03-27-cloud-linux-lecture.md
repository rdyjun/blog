---
title: "리눅스 서버 통신 기본"
excerpt: ""

categories: [리눅스]
tags: [linux, telnet, port, ssh, ftp]

toc: true
toc_sticky: true

date: 2024-03-26 13:00:00 +0800
last_modified_at: 2024-03-26 13:00:00 +0800

pin: false
---

# port

- 프로세스마다 network 연결이 가능하도록 하는 번호
- 포트는 16bit로 이루어짐 (0-65535)

| port            | 설명                                       | 영역        |
| --------------- | ------------------------------------------ | ----------- |
| Well-Known port | System에서 사용하는 예약 영역              | 0-1023      |
| Registered port | 서버 소켓으로 사용(회사 프로그램들이 사용) | 1024-49151  |
| Dynamic port    | 매번 할당했다가 해제했다가 사용            | 49151-65535 |

# Telnet

- 리눅스에서 원격 접속을 하기 위한 어플리케이션
- 서버에서 Telnet Server를 설치하고 원격지 PC에는 Telnet Client 프로그램 설치
- 전통적인 원격 접속 방법인 텔넷은 보안에 취약하기 때문에 최근에는 보안 기능을 추가
- 원격지의 PC(Telnet Client)에서 리눅스 서버에 접속하면 서버에서 직접 작업하는 것과 동일하게 작업 가능
- 23번 port 사용

# OpenSSH

### SSH(Secure Shell Protocl)

> 컴퓨터와 컴퓨터가 인터넷과 같은 Public Network를 통해 서로 통신을 할 때 보안적으로 안전하게 통신을 하기 이해 사용하는 `네트워크 프로토콜`

- 텔넷은 서버, 클라이언트 사이에 데이터를 전송할 때, 암호화를 하지 않아 해킹 위험
- OpenSSH 서버는 텔넷 서버와 기능은 비슷하지만 데이터를 전송할 때 `패킷을 암호화`하여 보안이 우수하기에 이러한 텔넷의 단점을 보완하기 위해 사용
- 22번 port 사용

## OpenSSH 사용

### 1. 설치하기

```bash
apt-get -y install openssh-server
```

### 2. OpenSSH 상태 관리

```bash
systemctl restart ssh # 재시작
systemctl enable ssh # 상시 가동
systemctl status ssh # 서비스 가동 여부 확인 (q로 종료)
```

### 3. 방화벽 열기

```bash
ufw allow 22/tcp # 22번 포트 허용
```

### 4. SSH Server의 IP 주소 확인

```bash
ifconfig ens32 # ens32는 서버 ip등 정보를 담는 네트워크 인터페이스
```

### 5. 서버 종료

```bash
exit
```

## 파일 전송하기 - SCP(Ssh CoPy)

SSH 접속 + COPY

### 옵션

`-p`: port가 ssh의 기본 port인 22가 아닌 경우, 따로 지정
`-r`: 하위 디렉토리 모두 copy 함

```bash
scp [option] [source] [destination]
```

### 원격 서버에 있는 파일을 로컬 컴퓨터로 가져오기

상황: User1 계정으로 접속하여 111.111.111.111에 /home/user1/index.html을 내 컴퓨터 /home/abc로 옮김

```bash
scp user1@111.111.111.111:/home/user1/index.html /home/abc/

# port가 9999일 때
scp -p 9999 user1@111.111.111.111:/home/user1/index.html /home/abc/
```

### 로컬 컴퓨터에 있는 파일을 원격 서버에 전송

상황: 내 컴퓨터에 /home/abc/를 user1로 접근하여 111.111.111.111에 /home/user1/로 옮김

```bash
# -r 옵션을 통해 하위 모든 file을 전송
scp -r /home/abc/ user1@111.111.111.111:/home/user1/

# 포트가 다른 경우 (ex: 9999)
scp -p 9999 -r /home/abc user1@111.111.111.111:/home/user1/
```

# Windows에서 Linux로 연결

- SSH client program 설치
  - Putty
  - NetSarang X Shell

# 파일 전송 서비스 FTP

- File을 주고 받을 수 있는 서버를 운영한다
- Server/client 시스템
- 21번 포트 사용

### FTP 프로그램

- vsftpd
- proftpd

## vsftpd 설치 및 사용

### 1. 설치

```bash
apt-get -y install vsftpd
```

### 2. 재시작

```bash
systemctl restart vsftpd

systemctl enable vsftpd

systemctl status vsftpd
```

### 3. 포트 접근 허용

```bash
ufw allow ftp

# 임시로 원활한 외부 접속 허용 시
systemctl stop ufw
```

### 4. IP 주소 확인

```bash
ifconfig ens32
```

### 5. Client에서 FTP 서버에 접속하여 파일 다운로드/업로드하기

#### 5-1. filezilla 클라이언트 설치

```bash
sudo apt-get -y install filezilla
```

#### 5-2. filezilla 실행, 접속

```bash
filezilla
```

![alt text](/assets/img/filezilla-1.png)

호스트: Server IP 주소
사용자명: Server id
비밀번호: 사용자에 따른 비밀번호
포트: 21
![alt text](image.png)

#### 5-3. 파일 다운로드

![alt text](/assets/img/filezilla-3.png)

#### 5-4. 파일 업로드

![alt text](/assets/img/filezilla-4.png)

## anonymous(익명) 사용자의 접속 및 파일 업로드 허용 설정

### 1. 설정파일 수정

```bash
vi /etc/vsftpd.conf
```

anonymous_enable=YES - 익명 사용자 접근 허용
write_enable=YES - 작성 권한 허용
anon_upload_enable=YES - 업로드 권한 허용
anon_mkdir_write_enable=YES - 글 쓰기 권한 허용
![alt text](/assets/img/vsftpd.png)

### 2. 익명 사용자 폴더

```bash
/srv/ftp
```

### 3. 재시작 및 확인

```bash
systemctl restart vsftpd

systemctl enable vsftpd

systemctl status vsftpd
```
