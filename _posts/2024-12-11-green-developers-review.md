---
title: "[Naver Cloud] Green Developers 후기"
author: rdyjun
excerpt: "네이버 클라우드 예비 개발자 지원 프로그램 참여 후기"

categories: [naver-cloud]
tags: [naver, cloud, green-developers]

toc: true
toc_sticky: true

date: 2024-12-11 20:00:00 +0800
last_modified_at: 2024-12-11 20:00:00 +0800
image: https://github.com/user-attachments/assets/a5df995e-4142-43d9-8f6c-9aa554a420b6

pin: false
---

네이버 부스트캠프 과정에서 참여하게 된 Naver Cloud Green Developers 후기이다.  
간단한 프로젝트 소개를 시작으로 해당 프로젝트에서의 Naver Cloud 적용 과정을 설명하겠다.

## 프로젝트 소개 - inear

![프로젝트 소개 배너](https://github.com/user-attachments/assets/354d7a48-d2aa-445e-a15f-9b4c19aa4200)

[Github](https://github.com/boostcampwm-2024/web18-inear)

[Inear Web Link](https://inear.live)

오직 음악으로만 소통할 수 있는 공간을 필요로 하는 아티스트와 팬들을 위한 실시간 앨범 감상 서비스이다.

앨범 발매 시간에 맞춰 열리는 라이브 세션에서 음악을 감상하며, 다른 팬들과 채팅으로 감상평을 나눌 수 있다.

### 기획의도

유튜브, 치지직과 같은 영상 스트리밍 플랫폼은 시각적 요소가 큰 비중을 가지고 있고,  
스포티파이, 멜론과 같은 음악 스트리밍 플랫폼은 경우 실시간으로 여러 사용자들이 함께 참여할 수 있는 공간이 존재하지 않는다.

우리는 시각적인 요소를 접어두고 오직 음악에만 집중하여 사용자들이 실시간으로 재생된 음악에 대해 공감하고 공유하길 바랐다.

결과적으로 이 서비스는 각 사용자가 응원하는 아티스트의 신곡이 발매되었을 때, 실시간으로 음악을 함께 듣고 서로의 감정을 공유하는 공간이 되길 기대한다.

## Ncloud 서비스 적용기

프로젝트의 서버를 띄우고, 음악 파일을 저장하는 등 클라우드 서버가 필요했다.  
우리는 Ncloud에서 지원하는 여러 서비스를 사용하게 되었다.

1. Server
2. DB for Redis
3. Object Storage

위 서비스를 활용하여 아래와 같은 아키텍처로 구성할 수 있었다.

![시스템 아키텍처](https://github.com/user-attachments/assets/838b3842-cc9f-4bf2-9dc9-6570d746f4af)

### Server

백엔드는 nest.js를 프론트는 react.js를 선택했고, 배포를 위해 nginx를 사용하게 되었다.  
배포를 위한 서버가 필요했고, 제한된 비용에서 6주라는 기간동안 계속 서버를 실행해야 했기에 가장 저렴한 G3 서버를 사용하기로 했다.  
그렇게 Ncloud G3 서버 인스턴스에 우분투 리눅스를 사용하게 되었다.

여기서 서버는 public VPC와 private VPC로 나누었다.

public VPC에는 nginx와 react.js, nest.js를 두게 되었다.  
여기서 nest.js는 배포 환경을 빠르게 구성하고 확인하기 위해 두게 되었다.

private VPC에는 보안을 위해 MySQL만 띄워두었다.

배포는 nginx, nest.js를 도커 컴포즈 파일로 배포했고,  
react 파일은 우선 정적 파일로 nginx가 관리하도록 설정했다.

> 프로젝트 이후에 vercel에 react 파일을 배포할 수 있다는 사실을 알았다.  
> G3 용량이 약 10GB로 크지 않기 때문에 react 파일을 별도로 배포해볼 계획이다.

Ncloud의 서버 인스턴스를 처음 생성할 때 당황했던 것은,  
기본 계정이 비활성화 되어있어서 루트 계정으로 접근해야 했던 점과,  
ssh 비밀번호를 비활성화 할 경우 두 개의 파일을 수정해야 한다는 점이다.

계정 목록을 확인해보니, 아래와 같이 로그인 셸이 /sbin/nologin으로 되어 있었다.  
우선은 별도의 계정을 생성하지 않고, root로 진행했다.

> 아마 이 부분은 보안 때문에 설정된 것으로 추측된다.

![Ncloud 리눅스 기본 사용자](https://github.com/user-attachments/assets/7badaa37-12a8-419a-bb8c-8d68cbb0c4ac)

다음으로 ssh의 비밀번호를 비활성화 할 때 `/etc/ssh`의 `sshd_config`에서 `PasswordAuthentication no`를 설정해도 적용되지 않는다는 점이었다.

부스트캠프의 피어세션에서 `/etc/ssh/sshd_config.d/50-cloud-init.conf`의  
`PasswordAuthentication` 설정도 `no`로 바꿔야 한다는 사실을 알게 되었고,  
이를 적용했을 때 계정이 비활성화 된 것을 볼 수 있었다.

> 이 파일은 클라우드 환경에서 인스턴스가 부팅될 때 생성된다고 한다.

![키 없이 ssh 접근 실패](https://github.com/user-attachments/assets/c6ad5a72-4bf0-4271-ad2f-ed5762973e16)

![키를 통한 ssh 접근 성공](https://github.com/user-attachments/assets/2df24726-54e7-4342-bd4c-dc69d99e2e31)

### Object Storage

음악을 스트리밍하기 위해서 음악 파일을 저장해야 했다.  
서버의 여유 용량이 넉넉치 않았기도 하고, 이후 CDN 적용을 위해 Object Storage를 선택했다.

> Ncloud 자체에서도 Object Storage의 CDN을 지원하는 것 같지만  
> 아직 프로젝트에 적용해보지 못했다.

현재는 Object storage에 음원의 .m3u8 파일과 .ts 파일들을 저장하고,  
각 음원에 대한 앨범 커버나, 배너도 함께 저장하고있다.

> .m3u8 파일과 .ts 파일은 HLS(Http Live Streaming) 프로토콜을 위해 변환된 음원 파일이다.

### Cloud DB for Redis

각 스트리밍 세션에 대한 정보를 저장하기 위해 redis를 추가하고자 했다.  
다만, 설치 과정을 생략하고자 Ncloud의 DB for Redis를 선택하게 되었다.

Redis 도입으로 인해 실시간으로 빠르게 사용자 수, 재생 시간 등의 데이터를 저장하고 클라이언트에 반환할 수 있었다.

## 만족했던 점

- 한글로 작성된 공식문서
- 다른 클라우드 서비스와 크게 다르지 않아 익숙하게 사용 가능

공식문서가 한글로 작성되었기 때문에 잘 알지 못하는 개념도  
공식문서를 보고 쉽게 이해하여 쉽게 적용해볼 수 있었다.

AWS 경험과 비교했을 때, 크게 다르지 않아서 보다 익숙하고 빠르게 적용할 수 있었다고 생각한다.  
이 점은 AWS에서 Ncloud로, Ncloud에서 AWS로 환경을 전환하는 데 어려움이 없을 것이라는 의미로 생각된다.

## 아쉬운 점

- 레퍼런스 부족

사용하면서 유일하게 아쉬웠던 부분이, Ncloud 사용 중 발생한 문제를 검색했을 때 문제 해결에 대한 레퍼런스가 부족했다.  
공식 문서에서 찾기 어려운 문제들에 대해서 레퍼런스를 참조하면 좋았겠지만,  
서치된 레퍼런스가 몇 개 없다보니 문제 해결에 시간을 많이 사용한 것 같다.

## AI 확장성

우리 팀에서는 아직 AI를 도입하지 않았다.  
Ncloud에서 지원하는 Clova AI를 사용한다면,  
사용자 간 불순한 채팅을 검열할 수 있고,  
기능 중 스트리밍이 끝난 앨범에 대한 정보를 출력하는 페이지가 있는데,  
이 페이지에 세션에서의 채팅 기록을 요약하고 노출할 수 있을 것으로 기대한다.

> 아래는 AI 적용 후에 대한 단순 예시이다.

<section style="display: flex; flex-direction: column; align-items: center">
  <img src="https://github.com/user-attachments/assets/e2160e3b-9abe-4cd8-85c7-4fa24698822c" alt="AI 적용 후 채팅창" width="200px">
  <img src="https://github.com/user-attachments/assets/b91abce2-b494-4c3d-a612-a26e084a236e" alt="AI 적용 후 상세 페이지" width="600px">
</section>

## Green Developers 참여 소감

부스트캠프 덕분에 Green Developers에 참여할 수 있었는데, 평소 부담되었던 클라우드 서비스를 부담 없이 이용해 볼 수 있었다.

여태까지 사용해보지 못했던 Ncloud에서 많은 서비스들이 존재하고,  
DB for Redis과 같이 환경 분리와 빠른 환경 설치라는 두 가지 이점을 가지고 편리하게 사용할 수 있었다.

Ncloud에서 지원하는 Clova AI를 아직 사용해보지 못했지만,  
해당 AI가 지원하는 기능들을 통해 프로젝트에 많은 확장성을 고려해볼 수 있었다고 생각한다.

이번 Green Developers를 통해 Ncloud가 서비스하는 기능들을 알게 되었고,  
다 활용하기도 어려울만큼 많은 서비스들이 존재하여 부스트 캠프 일정 외 추가로 학습하고  
어떻게 확장 및 적용해볼 수 있을 지 고민하고자한다.
