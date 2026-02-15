---
title: "[클라우드] 로드 밸런싱"
author: rdyjun
excerpt: "서버 부하 분산을 위한 로드 밸런싱"

categories: [cloud]
tags: [loadbalancing, elb]

toc: true
toc_sticky: true

date: 2024-06-16 09:25:00 +0800
last_modified_at: 2024-06-16 09:25:00 +0800

pin: false
---

로드 밸런싱은 여러 서버를 두고, 부하를 각 서버로 분산하기 위해 사용되는 기술이다.<br>
이 로드 밸런싱의 종류와 특징을 알아볼 것이다.

# 로드밸런싱

- 네트워크 트래픽을 하나 이상의 서버나 장비로 분산하기 위해 사용되는 기술
- S/W나 H/W를 통해 로드 밸런싱을 수행할 수 있다.
- 서비스 사용자의 요청을 서버로 분산해서 전달

## ELB(Elastic Load Balancing)

### 트래픽 증가에 따른 대응 방법

1. Scale Up (로드밸런싱 X)
   - CPU/RAM/Disk 성능 & Network 대역폭 등 증가
   - 비싸고 좋은 서버로 변경
2. Scale Out (로드밸런싱 O)
   - 부하를 처리할 서버 대수를 늘림
   - 저렴한 서버 여러 대를 이용해 더 많은 부하 감당
   - 같은 그룹에 있는 Server 들에게 요청이 골고루 분배될 수 있도록 한다.
   - 같은 그룹에 있는 server들은 동일한 기능을 서비스한다.

### 로드 밸런싱 방식

1. Round Robin
   로드 밸런서에서 서버 선택 시, 다음 순차적으로 서버를 선택한다.
2. Hash(Sticky Session)
   로드 밸런서에서 서버 선택 시, 클라이언트가 서버와 한 번 연결되면, 그 이후에는 항상 같은 서버로 연결한다.
   > 처음 연결할 때에는 클라이언트의 고유 값을 해시 함수에 입력하여 해시 값을 계산한다.
   > 이렇게 나온 해시 값을 서버 대수로 나누어 서버를 선택한다.
3. Least Connection
   로드 밸런서에서 서버 선택 시, 연결 수가 가장 적은 서버를 선택한다.
   > a 서버에 3명이 연결되어 있고, b 서버에 2명이 연결되어 있다면, b 서버로 연결된다.
4. 응답시간(Response Time)
   로드 밸런서에서 서버 선택 시, 응답속도가 가장 빠른 서버를 선택한다.

### ELB 유형

1. ALB(Application Load Balancer)
   - OSI 7 Layer 중 `Application Layer`에 속하는 패킷을 처리할 수 있다.
   - `Http/Https` 프로토콜
2. NLB(Network Load Balancer)
   - OSI 7 Layer 중 `Transport Layer`에 속하는 패킷을 처리할 수 있다.
   - `TCP` 프로토콜
3. CLB(Classic Load Balancer)
   - OSI 7 Layer 중 `Network` 및 `Transport Layer`에 속하는 패킷을 처리할 수 있다.
   - 과거에 사용되던 방식으로 호환성을 위해 유지되는 상태이다.

### ELB 옵션

1. 인터넷 트래픽용(Internet-facing)
   - 인터넷을 통해 ELB에 접근하는 경우이다.
     > 즉, Public IP를 가진 클라이언트가 접근하는 경우를 의미한다.
   - 단, ELB가 `Private IP:Public IP` 와 같이 쌍을 가지므로, `Private IP 클라이언트`에서도 사용 가능하다.
2. 내부 네트워크 트래픽용(Internal)
   - 내부 네트워크에서 ELB에 접근하는 경우
     > 즉, Private IP를 가진 클라이언트에서 접근하는 경우
   - 이 경우는 주로 전용선을 통해 인터넷을 거치지 않고 VPC에 연결되는 경우나,<br>
     VPN을 통해 Private IP를 이용해서 통신하는 경우에 활용된다.

### ELB 동작 특징

#### 상태확인(Health Check) 서비스

- ELB에서 연결할 서버들의 상태를 상시 체크한다.
- 비정상 상태의 서버로 트래픽을 전달하지 않는다.
- ELB에서 서버 측에 요청을 전달해서 응답을 받음으로써 처리되므로,<br>
  서버의 `보안그룹`에서 해당 요청을 받을 수 있도록 `지정된 포트`에 대해 접근을 허용해야 한다.
- 고가용성(High Availabilty) 지원
