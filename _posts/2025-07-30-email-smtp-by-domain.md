---
title: "개인 도메인으로 무료 메일 보내기"
author: rdyjun
excerpt: "Daum 스마트워크를 통해 내 도메인으로 메일 보내기"

categories: [smtp]
tags: [spring boot, smtp, daum, 스마트워크]

toc: true
toc_sticky: true

date: 2025-07-30 17:30:00 +0900
last_modified_at: 2025-07-30 17:30:00 +0900
pin: false
---

회원 가입 과정에서 이메일 인증이 필요하며 이를 백엔드 서버에서 요청을 받아 인증 메일을 보내도록 할 것이다. 인증 메일은 서비스 운영 중 실제 사용자에게 보내질 메일이기 때문에 개인 메일로 보내지 않고, 사용중인 도메인으로 메일을 보낼 계획이다.

본 글은 도메인의 네임서버가 Cloudflare인 환경에서의 과정이다.

무료로 도메인으로 이메일을 보낼 수 있는 서비스는 다음의 스마트워크가 있었다.

스마트워크는 다음 계정이 있는 경우 이 계정에 도메인을 연결해서 쓸 수 있는 구조다.

## 스마트워크 등록

다음(Daum) 메일에 들어가서 좌측 하단에 스마트워크를 클릭해준다.

![image.png](/assets/img/post/email-smtp-by-domain/image-2.png)

그러면 서비스 신청 화면이 나오는데, 여기서 본인의 보유 도메인을 넣어주면 된다.

![image.png](/assets/img/post/email-smtp-by-domain/image-1.png)

다음 단계로 넘어가서 기업/단체명을 작성해주고, 이메일 주소는 도메인으로 보낼 이메일의 이름만 넣어주면 된다.

예를 들어, `example.com`이 도메인이라고 할 때 이메일 발송 시 발송인을
`no-reply@example.com` 로 설정하고 싶다면, `no-reply`를 적으면 된다.

이름 및 휴대폰 번호는 담당자의 이름 및 번호를 작성하면 되는 것 같으며,

약관까지 체크하면 신청이 끝난다.

그러면 아래와 같은 화면이 나올텐데, 아래 MX 서버 주소를 본인의 네임 서버에 등록해주어야 한다.

![image.png](/assets/img/post/email-smtp-by-domain/image-3.png)

## MX 서버주소 등록 by Cloudflare

위에서 받은 MX 키를 현재 도메인의 네임서버에 등록해주어야 한다.

나는 https 설정을 위해 네임 서버를 Cloudflare를 이용하고 있었고, 이 MX키를 Cloudflare에 등록해보겠다.

> 여기서 Cloudflare에 도메인을 등록하는 과정은 포함되어있지 않다.

먼저 Cloudflare에 설정하고자 하는 도메인을 선택하여 세부 정보에 들어온다.

이 페이지 좌측에 DNS가 보일텐데 여기서 Records를 선택한다.

![image.png](/assets/img/post/email-smtp-by-domain/image-4.png)

그러면 Add record라는 버튼이 보이고, 그 아래 도메인과 ip등이 보일 것이다.

여기서 Add record를 누르면 아래와 같이 등록할 수 있는 요소들이 나타난다.

![image.png](/assets/img/post/email-smtp-by-domain/image-5.png)

아까 스마트워크에서 알려준대로 Type은 MX, Name은 @, IPv4는 그대로 붙여넣고, Priority도 마찬가지로 스마트워크에 나온 그대로 넣어주면 된다.

이후 스마트워크에 다시 접속해서 연결 확인을 시도하면 정상적으로 넘어갈 것이다.

> 시간이 조금 소요되는 경우도 있다고 한다.

## SMTP(Simple Mail Transfer Protocol) 설정

이메일 클라이언트(Daum)가 이메일을 보낼 때 SMTP를 거쳐서 보내게 되는데, SMTP를 지정하지 않으면 이메일을 보낼 수 없기 때문에 꼭 설정해주어야 한다.

Daum에서는 SMTP만 단일로 설정할 수 없고, IMAP 또는 POP3와 함께 설정해야 한다.

나는 페이지에 제일 먼저 나온 IMAP을 사용하기로 했다.

Daum 메일의 좌측 하단 `설정`에 들어가, 상단에 `IMAP/POP3`탭을 선택하고, `IMAP / SMTP`사용함을 누르면 앱 비밀번호를 준다.

이 앱 비밀번호는 Spring Boot 설정 시 필요하니 잘 기억해두자.

## Spring Boot 설정

백엔드에서 메일을 보내야하기 때문에 Spring Boot에서 연결해보겠다.

먼저 아래 의존성을 `build.gradle`에 추가한다.

```java
implementation 'org.springframework.boot:spring-boot-starter-mail'
```

이후 application 파일에 mail 설정을 넣어야 한다.

username은 도메인을 연결한 자신의 다음 계정을 넣어주면 되고,

password는 위 IMAP/SMTP 설정 때 받은 앱 비밀번호를 넣어주면 된다.

```java
spring:
  mail:
    host: smtp.daum.net
    port: 465
    username: daum 이메일 (@daum.net)
    password: 앱 비밀번호
    protocol: smtp
    properties:
      mail:
        smtp:
          auth: true
          ssl:
            enable: true
```

properties의 auth는 SMTP가 id/pw를 요구하는 상황을 의미한다.

false로 둘 경우 인증되지 않아 거부될 것이다.

ssl의 경우 “암호화된 통신을 사용하겠다”라는 의미이다.

그 다음은 아래와 같이 코드를 작성하여 보낼 수 있다.

```java
  MimeMessage message = sender.createMimeMessage();
  MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
  helper.setFrom(senderEmail); // 발송자 이메일
  helper.setTo(userEmail);     // 수신자 이메일
  helper.setSubject(subject);  // 제목
  helper.setText(content, true); // 내용

  sender.send(message);
```

MessageHelper의 역할은 UTF-8 인코딩, HTML 설정, 멀티파트 지원 등의 이유가 있다고 한다.

현재 프로젝트에서 이메일을 HTML로 보내기 위해 Helper를 사용하기로 했다.

`new MimeMessageHelper(message, true, "UTF-8");` 에서 2번째 인자는 멀티 파트 사용 여부를 의미한다.

첨부파일, HTML 등 여러 컨텐츠 형식을 포함할 수 있도록 한다.

`helper.setText`에서 2번째 인자에 true가 들어가는 것은, 이 내용을 html로 보낼 것인지에 대한 여부를 선택하는 것이다.

단순히 문자만 보낼 거라면, false로 두면 된다.

## 결론

앞 단계까지 완성된다면, 이제 백엔드에서 사용자에게 메일을 보낼 수 있게된다.

개인 메일로 보낸다면 빠르게 끝났겠지만, 실제 운영을 위한 서비스기 때문에 도메인으로 처리하였다.

개인 프로젝트를 진행하는 경우 위와 같은 방식으로 도메인 메일을 사용하는 것이 비용 절약 측면에서 좋아보인다.
