---
title: "웹 서버 공부"
excerpt: ""

categories: [웹]
tags: [web, spring, spring boot]

toc: true
toc_sticky: true

date: 2024-03-26 13:00:00 +0800
last_modified_at: 2024-03-26 13:00:00 +0800

pin: false
---

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
-  DI - Dependency Injection
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

|어노테이션    |위치           |의미                              |
|-----------|--------------|---------------------------------|
|@Service   |xxxServiceImpl|비즈니스 로직을 처리하는 Service 클래스  |
|@Repository|xxxDAO        |데이터베이스 연동을 처리하는 DAO 클래스   |
|@Controller|xxxController |사용자 요청을 제어하는 Controller 클래스|

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