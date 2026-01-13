---
title: "Spring Boot에서 카카오 소셜 로그인 적용"
author: rdyjun
excerpt: "OAuth2.0을 활용한 카카오 소셜 로그인"

categories: [oauth]
tags: [social, login, oauth, spring, "부트캠프후기", "멀티캠퍼스부트캠프", "현대이지웰JAVA풀스택"]

toc: true
toc_sticky: true

date: 2026-01-13 08:53:00 +0900
last_modified_at: 2026-01-13 08:53:00 +0900
pin: false
---

Spring Boot 환경에서 OAuth2.0을 도입해보려고 한다.

기존 회원가입이 된 사용자에 대해서만 OAuth 연동을 하도록 할 것이며, 이에 따라 OAuth로만 회원가입은 불가하다.

JWT를 사용중인 환경으로 OAuth2.0을 어떻게 적용할 것인 지에 대해 알아보자.

## 동작 흐름

카카오에서 제공하는 공식 과정은 아래 사진과 같다.

![카카오 공식 흐름도](/assets/img/2026-01-13/kakao-official-process.png)

Spring Boot에서 어디까지 지원이 되는지 어떤 구조로 동작하는지 위 과정과 대조해서 알아보자.

### Kakao API + Spring Boot의 인증 흐름

![나의 흐름도](/assets/img/2026-01-13/my-process.png)

1. Spring Boot는 로그인 요청을 받으면 요청 정보에 담긴 서비스의 API에 요청을 보낸다.
    - 여기서 로그인 요청 주소는 `/oauth2/authorization/{registrationId}` 가 
    Spring Boot의 oauth가 지원하는 기본 형태이다.
2. 서비스 API로부터 받은 로그인 페이지를 서버가 받아 사용자에게 반환
    - https://kauth.kakao.com/oauth/authorize?response_type=code&client_id=client_id&**redirect_uri=http://localhost:8080/login/oauth2/code/kakao**
    - `/login/oauth2/code/kakao`는 Spring Boot의 oauth가 지원하는 기본 형태이다.
3. 사용자가 로그인 후 [확인하고 계속하기] 버튼을 클릭
4. 로그인 페이지 URI의 쿼리에 존재하는 redirect_uri페이지로 요청 전송
5. `OAuth2LoginAuthenticationFilter`가 요청을 가로채어 `DefaultOAuth2UserService` 호출
6. `DefaultOAuth2UserService` 에서는 요청 속 소셜 계정의 식별자를 검증
7. 검증 성공 시 `SimpleUrlAuthenticationSuccessHandler` 를 호출하여 후속 처리
8. 검증 실패 시 `SimpleUrlAuthenticationFailureHandler` 를 호출하여 후속 처리

## API 설정

위 과정을 구현하기 전 카카오 API에 필수 정보를 등록해주어야 한다.

### 가입

아래 경로 또는 카카오 디벨로퍼스 페이지에 접근하여, 회원가입을 진행한다.

https://developers.kakao.com/

### 앱 등록

이후 상단에 [앱] 탭을 클릭하여 앱을 생성해준다.

![앱 생성 화면](/assets/img/2026-01-13/app-create.png)

### API 키 등록

앱을 생성했다면 해당 앱을 클릭하여 들어가, 좌측 메뉴바에서 플랫폼 키로 들어가준다.

![카카오 메뉴탭](/assets/img/2026-01-13/kakao-navigation.png)

![플랫폼 키 생성 버튼](/assets/img/2026-01-13/add-platform-key.png)

이후 REST API키 추가를 통해 키 이름을 간략히 적어주고,
리다이렉트 URI는 2개를 미리 입력해두자.

- https://서버주소/oauth2/authorization/kakao
- https://서버주소/login/oauth2/code/kakao

![플랫폼 키 정보 입력 화면](/assets/img/2026-01-13//platform-key-input.png)

이 설정까지 끝냈다면 만들어진 **클라이언트 아이디**와 **클라이언트 시크릿**을 기억해두자.

## Spring Boot

### 의존성 설정

Gradle을 사용중이기 때문에 아래 한 줄을 `build.gradle` 에 추가해주었다.

```json
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
```

### application 설정

https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#oidc-discovery-sample-response

위 문서에 따르면 인가 코드 요청 API, 토큰 요청 API, 사용자 정보 조회 API가 존재한다.

위 API 경로를 provider.kakao에 작성해주어야 제대로 동작한다.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-id: 클라이언트 아이디
            client-secret: 클라이언트 시크릿
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8080/login/oauth2/code/kakao
            scope: email
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
```

<aside>
💡

registration

</aside>

> client-id, client-secret
> 

Rest API 키 추가 시 받은 정보를 입력한다.

> authorization-grant-type
> 

https://oauth.net/2/grant-types/

토큰 교환 API에서 사용되는 속성으로 교환 방식을 정한다.

- authorization_code
    - 안전한 방식으로, 인가 코드를 통해 토큰을 발급하는 구조이다.
- client_credentials
    - 코드가 필요하지 않아 사용자 인증에 사용되지 않고, 서버간 통신 시 사용된다.
- refresh_token
    - 발급받았던 토큰을 재전송하여 새 토큰을 발급받는다. (로그인 연장)
- pkce
    - CSRF 및 인증 코드 주입 공격을 방지하기 위한 인증 코드 흐름의 확장이다.
    - 클라이언트 인증을 대체하는 것이 아니므로 공개 클라이언트를 기밀 클라이언트처럼 취급할 수 없다.
- device_code
    - 브라우저가 없거나 입력이 제한된 장치가 장치 흐름 에서 이전에 획득한 장치 코드를 액세스 토큰으로 교환하는 데 사용된다.

> redirect-uri
> 

Spring Boot oauth 라이브러리에서 제공되는 `/login/oauth2/code/kakao`를 사용한다.

백엔드 주소를 앞에 덧붙이면 된다.

> scope
> 

로그인 후 가져올 사용자 데이터 범위이다.

profile,  account_email 등 다양한 데이터가 있다.

<aside>
💡

provider

</aside>

> 인가 코드 요청 API
> 

카카오 인증을 호출해서 사용자의 로그인 및 로그인 동의 후 인가 코드를 서비스 서버로 전송한다.

> 토큰 요청 API
> 

인가 코드를 검증하여 토큰을 발급한다.

사용자 정보 조회 시 사용되는 토큰이다.

> 사용자 정보 조회 API
> 

scope에 기록된 사용자 데이터를 조회하기 위한 API 이다.

사용자가 카카오로 로그인 시 email 정보 제공 동의를 한 경우 해당 API를 통해 데이터를 받아올 수 있다.

### 서비스 구현

OAuth2 권한을 검증하기 위해 Service를 구현해야 한다.

`OAuth2LoginAuthenticationFilter`는 `DefaultOAuth2UserService`를 호출하여 검증하기 때문에

`DefaultOAuth2UserService`를 상속받아서 구현한다.

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final MemberSocialAccountRepository memberSocialAccountRepository;
    private final ReportValidator reportValidator;
    private final OAuth2UserParser oAuth2UserParser;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        // 소셜 로그인 API를 통해 사용자 정보 가져오기
        OAuth2User oAuth2User = super.loadUser(userRequest);

        // 소셜 계정에 따라 id 추출
        String registrationId = userRequest.getClientRegistration()
                .getRegistrationId();
        String providerId = this.oAuth2UserParser.extractProviderId(oAuth2User, registrationId);

        // 소셜 계정과 연동된 회원이 있는지 확인
        MemberSocialAccountDto socialAccount = memberSocialAccountRepository.findMemberSocialAccountDTO(providerId)
                .orElseThrow(() -> {
                    log.info("social account not linked: providerId={}", providerId);
                    return new AccountNotLinkedException();
                });

        // 소셜 계정이 회원과 매칭되지 않은 경우 예외 처리
        if (socialAccount.isMemberNotMatched()) {
            log.info("social account not linked: providerId={}", providerId);
            throw new AccountNotLinkedException();
        }

        // 소셜 계정과 연결된 계정이 탈퇴된 경우 예외 처리
        if (socialAccount.isMemberDeleted()) {
            log.info("linked account was deleted: providerId={} memberId={}", providerId,
                    socialAccount.member().getId());
            throw new LinkedAccountAlreadyDeletedException();
        }

        // 신고당한 회원인지 검증
        reportValidator.checkMemberAccessById(socialAccount.member().getId());

        return new CustomOAuth2User(oAuth2User, socialAccount.member().getId(), socialAccount.roleType());
    }
}

```

위 코드에서는 서비스 정책에 맞게 토큰을 검증하는 과정이다.

본 서비스는 소셜 계정이 사용자와 연결되어있어야 하고, 제재를 받지 않은 상태여야 한다는 정책에 따라 검증을 진행했다.

중간에 providerId를 가져오기 위해 parser를 사용했는데, 
각 공급사마다 담겨있는 속성의 이름이 다르기 때문에 parser로 분리하여 구현하였다.

```java
    public String extractProviderId(OAuth2User oAuth2User, String registrationId) {
        Map<String, Object> attributes = oAuth2User.getAttributes();

        return String.valueOf(attributes.get(ATTRIBUTE_NAME));
    }
```

> 반환 값
> 

마지막에 CustomOAuth2User를 반환하고 있다.

OAuth2User를 사용해도 되지만, 로그인 성공 시 memberId가 필요해서 Custom 클래스를 추가했다.

```java
public class CustomOAuth2User implements OAuth2User {

    private final OAuth2User delegate; // 기존 OAuth2User (구글, 카카오 등에서 받은 정보)

    @Getter
    private final Long memberId; // 우리 DB의 PK

    @Getter
    private final List<RoleType> roleTypeList;

    public CustomOAuth2User(OAuth2User delegate, Long memberId, List<RoleType> roleTypeList) {
        this.delegate = delegate;
        this.memberId = memberId;
        this.roleTypeList = roleTypeList;
    }

    // OAuth2User 메서드 위임
    @Override
    public Map<String, Object> getAttributes() {
        return delegate.getAttributes();
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return delegate.getAuthorities(); // 혹은 role 기반으로 생성해서 반환
    }

    @Override
    public String getName() {
        return delegate.getName();
    }
}
```

### Access Handler

검증이 성공했을 때 SuccessHandler를 통해 처리해주어야 한다.

앞서 반환한 CustomOAuth2User로 Authentication 인스턴스로 만들어주었다.

Authentication 가 필요한 이유는 Authentication를 기반으로 JWT 를 발급하기 때문이다.

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class OAuth2LoginSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    private static final String SOCIAL_TOKEN_PARAM = "socialToken";

    private final TokenGenerator tokenGenerator;

    @Value("${oauth.authorize-uri}")
    private String targetUrl;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
                                        Authentication authentication) throws IOException {

        CustomOAuth2User oAuth2User = (CustomOAuth2User) authentication.getPrincipal();
        List<GrantedAuthority> authorityList = oAuth2User.getRoleTypeList().stream().map(RoleType::getAuthority)
                .toList();
        Authentication auth = new UsernamePasswordAuthenticationToken(oAuth2User.getMemberId(), null, authorityList);

        String targetUrl = determineTargetUrl(auth);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
        log.info("OAuth2 login success. memberId: {}, redirect to: {}",
                oAuth2User.getMemberId(), targetUrl);
    }

    protected String determineTargetUrl(Authentication authentication) {

        String socialToken = tokenGenerator.generateSocialToken(authentication);

        return UriComponentsBuilder.fromUriString(targetUrl)
                .queryParam(SOCIAL_TOKEN_PARAM, socialToken)
                .build().toUriString();
    }
}
```

발급받은 JWT는 redirect 주소에 포함시켜 응답하면, 사용자는 redirect 페이지로 이동하게 된다.
이후 클라이언트에서 JWT를 포함한 재요청을 보내어 일반 로그인(소셜 로그인이 아닌)에서의 처리(알림 구독/해지)를 진행한다.

이 부분은 각 서비스 정책에 따라 커스텀하면 될 것 같다.

### FailureHandler

인증에 실패했을 때 redirect할 페이지를 추가해준다.

대부분 기존 회원과 관련된 문제로 로그인 페이지로 리다이렉트하고 있다.

```java
@Component
@Slf4j
public class OAuth2LoginFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
                                        AuthenticationException exception) throws IOException {
        // 에러 메시지와 함께 로그인 페이지로 리다이렉트
        String targetUrl = UriComponentsBuilder.fromUriString("login")
                .queryParam("error", "NOT_REGISTERED")
                .build().toUriString();

        getRedirectStrategy().sendRedirect(request, response, targetUrl);
        log.info("OAuth2 login failed. redirect to: {}", targetUrl);
    }
}
```

### Security Config 등록

이렇게 필수 기능들을 구현했다면 Spring Security가 인식하고 사용할 수 있도록 처리해주어야 한다.

Security Config에서 아래와 같이 인스턴스를 등록해주면, 실제 Spring Security가 기본 클래스를 사용하지 않고 내가 등록한 클래스 인스턴스를 사용하게된다.

```java
private final CustomOAuth2UserService customOAuth2UserService;
private final OAuth2LoginSuccessHandler oAuth2LoginSuccessHandler;
private final OAuth2LoginFailureHandler oAuth2LoginFailureHandler;

...

protected SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    ...
				.oauth2Login(oauth2 -> oauth2
				        .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService))
				        .successHandler(oAuth2LoginSuccessHandler)
				        .failureHandler(oAuth2LoginFailureHandler))
    ...
}
```

## 결과

서버 실행 후 아래 주소로 접근할 경우 로그인 동의 페이지가 띄워진다.

/oauth2/authorization/kakao 

![카카오 로그인 페이지](/assets/img/2026-01-13/login.png)

확인하고 계속하기를 진행할 경우

카카오 서버 내 인증 과정을 거치고 각 핸들러(Success, Failure)에 따른 동작이 실행된다.

> 실패
> 

![실패 로그](/assets/img/2026-01-13/failure-log.png)

> 성공
> 

아래와 같이 데이터를 추가하여 재시도했을 때 정상 로그인처리 되는 것을 볼 수 있다.

![DB 저장 이미지](/assets/img/2026-01-13/save-device.png)

![성공 로그](/assets/img/2026-01-13/success-log.png)

## 결론

OAuth2를 적용하는 과정에서 Redirect 파트가 많이 헷갈렸다

과정 중 Redirect URL을 잘못 입력해서 실패 요청을 API서버에 계속 보내기도 했다.

한동안 429 응답 코드가 반환되어 아무것도 못했지만 의미있는 경험이었다.

계정 아이디, 비밀번호 없이 버튼 한 번에 다른 플랫폼의 계정을 연결할 수 있다는 점은 큰 메리트라고 생각한다.

위 과정을 구현하면서 Flutter를 사용할 때의 구현 구조가 다르다는 것을 알게 되었고, 프론트에서 Flutter를 사용중인 경우, 위보다 간단하게 구현할 수 있다.

> Flutter는 앱에 직접 접근해서 토큰을 받아올 수 있다.
서버에서는 토큰 검증만 진행하면 된다.
>