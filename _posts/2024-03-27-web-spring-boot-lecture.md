---
title: "스프링 웹 이론 기초"
excerpt: ""

categories: [웹]
tags: [web, spring, spring boot]

toc: true
toc_sticky: true

date: 2024-03-26 13:00:00 +0800
last_modified_at: 2024-03-26 13:00:00 +0800

pin: false
---

# 뷰 템플릿과 MVC 패턴

#### 뷰 템플릿

`웹 페이지(View)`를 하나의 `틀(Template)`로 만들고 여기에 변수를 삽입해 서로 다른 페이지로 보여주는 기술이다

> 예시로는 회원 로그인 시 마이페이지의 구성이 바뀐다는 점이 있다.

## MPA vs SPA

#### MPA - Multi Page Application

클라이언트가 서버에 요청을 보냈을 때, 서버에서 html 코드를 반환한다

#### SPA - Single Page Application

클라이언트가 서버에 요청을 보냈을 때, 서버에서 페이지에 필요한 키와 키에 대한 값을 반환한다

## 자바 웹 프로그래밍 기술의 발전 흐름

- Servlet -> HTML 코드 출력 문제 해결 -> JSP
- JSP -> 불필요한 코드의 반복 해결 -> JSP MVC 모델
- JSP MVC 모델 -> 규격화/모듈화 -> 스프링
- 스프링 -> 더 간단하게 설정 -> 스프링 부트

```
┌────────────── 스프링 부트 ──────────────┐
|                 스프링 　      　   ───────── 스프링 컨테이너
|                                        |
|      ┌────── JSP MVC 모델 ──────┐ 　　　|
|      |           JSP            |      |
|      |                      ──────────────── 서블릿 컨테이너
|      |      ┌───Servlet───┐     |      |
|      |      |             |     |      |
|      |      |   ┌─JDK─┐   |     |      |
|      |      |   |   ──────────────────────── JVM
|      |      |   └─────┘   |     |      |
|      |      └─────────────┘     |      |
|      └──────────────────────────┘      |
|                                        |
└────────────────────────────────────────┘
```

## MVC 패턴

MVC 패턴의 최대 장점은 사용자에게 보여지는 `프레젠테이션 영역`과 `비즈니스 로직`, `데이터 구조`가 서로 완전히 `분리`되어있다는 점이다

#### 모델 1

```
Client -> Server -> JSP(View & Controller) -> JavaBean(Model & Controller) -> DB -> 이후 역순
```

#### 모델 2

```
1 Client -> 2 Server -> 3 Servlet(Controller) <-> 4 JavaBean(Model)
　　↑　　　　　　　　　　　　　　　 └> 5 JSP(View) -> 6 JavaBean(Model) -> 7 DB
    └────────────── 8 ───────────────────┘
```

|                         | 모델1             | 모델2                                        |
| ----------------------- | ----------------- | -------------------------------------------- |
| 컨트롤러와 뷰 분리 여부 | 통합 (JSP 파일)   | 분리 (JSP, Servlet)                          |
| 장점                    | 쉽고 빠른 개발    | 디자이너/개발자 분업 유리<br>유지보수에 유리 |
| 단점                    | 유지보수가 어렵다 | 설계가 어렵다<br>개발 난이도가 더 높다       |

## 서블릿 동작

웹 서버 <---> WAS <---> java_file

#### 모델1 동작

1. Web Server
2. login.jsp 실행
3. login_jsp.java로 변환
4. login_jsp.class로 컴파일

#### 모델2 동작

1. login.java 실행
2. login.class로 컴파일

### 서블릿 컨테이너 동작 방식

#### 1. Xml 설정 파일에 등록

1. `WEB-INF/web.xml` 파일을 로딩하여 구동

   ```xml
   <web-app>
      <servlet>
         <servlet-name>example</servlet-name>
         <servlet-class>example.ExampleServlet</servlet-class>
      </servlet>

      <servlet-mapping>
        <servlet-name>example</servlet-name>
        <url-pattern>/example</url-pattern>
    </servlet-mapping>
   </web-app>
   ```

2. 브라우저로부터 `/example` 요청 수신
3. 해당 Servlet 클래스(예시: example.ExampleServlet)를 찾아 객체 생성 후 doGet() 메소드 호출
4. doGet() 메소드 실행 결과를 클라이언트 브라우저로 전송

#### 2. 클래스 파일에 어노테이션 표기

```java
@WebServlet("/example.do")
or
@WebServlet(name="action", urlPatterns={"example.do"})
```

### 서블릿 클래스 파일 위치

```
C:/apache-tomcat9.0.33/webapps/ROOT/WEB-INF/classes/ExampleServlet.class
```

## 엔터프라이즈급 어플리케이션

- 개발을 위한 API

#### 1. 분산형/기업형 응용프로그램

- 결합력을 낮추는 DI, Transaction 처리, 로그 처리 등

#### 2. 일반적인 로컬 응용프로그램

- 파일 I/O, 콘솔 I/O, 윈도우 I/O, 네트워크 I/O, Thread 등

## 프레임워크 개요

- Framework는 어플리케이션을 개발할 때, 아키텍처에 해당하는 골격 코드를 제공한다.
- Solution이 완제품이라면 Framework는 반제품에 해당한다.

### 장점

#### 1. 빠른 구현 시간

- 개발자는 비즈니스 로직만 구현하면 되므로 제한된 시간 내에 많은 기능을 구현할 수 있다.

#### 2. 쉬운 관리

- 같은 프레임워크가 적용된 어플리케이션은 아키텍쳐가 같으므로 관리가 수월
- 유지보수의 시간도 절약

#### 3. 개발자들의 역량 획일화

- 숙련된 개발자와 초급 개발자가 생성한 코드가 비슷해진다.

#### 4. 검증된 아키텍쳐의 재사용과 일관성 유지

### Spring Framework

- 로드 존슨이 2004년에 만든 오픈소스 프레임워크
- `IoC`와 `AOP`를 지원하는 경량의 프레임 워크
- JAVA개발을 편하게 해주는 오픈 소스 프레임워크
- 객체지향에 충실한 설계가 가능함

# 스프링 콘셉트

## 스프링 컨테이너

- 스프링 안에서 동작하는 빈 생성 및 관리 주체

## 빈

- 스프링 컨테이너가 생성하고 관리하는 객체
- 빈 등록 방법은 여러가지

## 관점 지향 프로그래밍

- AOP - Aspect Oriented Programming
- 핵심 관점, 부가 관점으로 나누어 프로그래밍하는 것

ex)
핵심관점: 계좌이체 & 고객관리
부가관점: 로깅 & 데이터베이스 연결

## 제어의 역전과 의존성 주입

### 제어의 역전

- IoC - Inversion of Control
- 객체를 직접 관리하는 것이 아닌 외부(프레임워크)에서 관리하는 것

### 의존성 주입

- DI - Dependency Injection
- 빈: 스프링 컨테이너가 관리하는 객체
- 빈을 스프링 컨테이너로부터 주입 받아 사용 (직접 객체 생성 X)

```java
class A {

    @Autowired
    private final B b;

}
```

## 이직 가능한 서비스 추상화

- PSA - Portable Service Abstraction
- 스프링에서 제공하는 다양한 기술을 추상화해 개발자가 쉽게 사용하는 인터페이스
  - 데이터베이스를 위한 PSA: `JPA`, `MyBatis`, `JDBC`
  - 실행을 위한 PSA: `WAS`

# 분류

`JDK` `Sevlet` `JSP` `JSP MVC 모델` -> `Spring` -> `Spring Boot`

# 스프링 특징

`IoC`와 `AOP`를 지원하는 `경량`의 컨테이너 프레임워크

## 경량

- 여러 개의 모듈로 구성 : 각 모듈은 하나 이상의 JAR 파일로 구성
- 스프링 프레임워크가 `POJO(Plain Old Java Object)` 형태의 객체를 관리하기 때문에 가능

## 제어의 역행(Inversion of Control: IoC): DI 지원

- 자바 코드로 하는 것이 아니라 객체 생성을 컨테이너가 대신 처리하기 때문에 소스에 의존관계가 표시되지 않으므로 결합도가 낮아짐
- 프로그램의 흐름을 개발자가 아닌 프레임워크가 주도하게 된다는 디자인 패턴
- 객체의 생성에서 소멸까지 프레임워크가 관리를 하면서 DI(Dependency Injection)나 AOP(Aspect Oriented Programming)가 가능

### DI가 아닌 경우

개발자가 직접 new로 객체를 생성해야함

### DI의 경우

스프링 컨테이너가 객체를 생성하여 개발자 코드에 주입

## 관점지향 프로그래밍(Aspect Oriented Programming: AOP) 지원

- 공통으로 사용하는 기능들을 독립된 클래스로 분리하고, 해당 기능을 프로그램 코드에 명시하지 않고 선언적으로 처리하는 것

## 컨테이너 역할

- 특정 객체의 생성과 관리를 담당하며 다양한 기능을 제공
- ex) Servlet 컨테이너, EJB 컨테이너, Spring 컨테이너

# Spring Layered Architecture 구조

> 모든 단계에서 `VO(DTO)`와 양방향 통신

## 1. Presentation Layer

Spring MVC 객체

> 사용자와 커뮤니케이션 담당

### 동작

`Dispatcher Servlet` -> `Handler Mapping`(Controller 찾기) -> `Controller` -> `ModelAndView`(값 반환) -> `Dispatcher Servlet` -> `View Resolver`(View 찾기) -> `View`

## 2. Bussiness Layer

Presentation Layer에서 요청을 보내면 실제로 비지니스 로직 수행

> 사용자의 요청에 대한 비즈니스 로직 담당 (IOC, AOP)

### 동작

`Presentation Layer - Controller` -> `Service Interface` -> `Service Implementations`

## 3. Data Access Layer(Repository Layer)

DB에 값을 저장하거나 가져오기 위해 `JDBC`, `Mybatis`, `JPA` 등을 사용해 구현한 `DAO`

### 동작

`Bussiness Layer - Service Implementations` -> `DAO Interface` -> `DAO Implementations` -> `Database`

# Layer별 bean 객체 선언 Annotaion

| 어노테이션  | 위치           | 의미                                     |
| ----------- | -------------- | ---------------------------------------- |
| @Service    | xxxServiceImpl | 비즈니스 로직을 처리하는 Service 클래스  |
| @Repository | xxxDAO         | 데이터베이스 연동을 처리하는 DAO 클래스  |
| @Controller | xxxController  | 사용자 요청을 제어하는 Controller 클래스 |

# Java 기반의 Framework

<table>
    <tr>
        <th>\</th>
        <th>Business</th>
        <th>Persistence</th>
    </tr>
    <tr>
        <th rowspan="2">Presentation</th>
        <td>Struts</td>
        <td>UI Layer에 중점을 두고 개발된 MVC(Model View Controller) 프레임워크임</td>
    </tr>
    <tr>
        <td>Spring(MVC)</td>
        <td>sruts와 동일하게 MVC 아키텍처를 제공하는 UI Layer 프레임워크<br>
        Struts처럼 독립된 프레임워크는 아니고 Spring 프레임워크에 포함되어 있음</td>
    </tr>
    <tr>
        <th>Business</th>
        <td>Spring (IoC, AOP)</td>
        <td>Spring의 IoC와 AOP 모듈을 이용하여 Spring 컨테이너에서 동작하는 엔터프라이즈 비즈니스 컴포넌트를 개발할 수 있음</td>
    </tr>
    <tr>
        <th rowspan="2">Persistence</th>
        <td>Hibernate or JPA</td>
        <td>Hibernate는 완벽한 ORM(Object Relation Mapping) 프레임워크</td>
    </tr>
    <tr>
        <td>iBatis or Mybatis</td>
        <td>iBatis 프레임워크는 개발자가 작성한 SQL 명령어와 자바 객체(VO 혹은 DTO)를 매핑해주는 기능을 제공<br>
        Mybatis는 iBatis에서 파생된 프레임워크로서 기본 개념과 문법은 거의 유사</td>
    </tr>
</table>
