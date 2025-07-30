---
title: "QueryDSL 충돌"
author: rdyjun
excerpt: "Spring Boot 프레임워크에서 DB에 접근할 때 UNION은 어떻게 써야 할까?"

categories: [database]
tags: [java, spring, spring boot, union]

toc: true
toc_sticky: true

date: 2025-07-15 17:55:00 +0900
last_modified_at: 2025-07-15 17:55:00 +0900
pin: false
---

> 참고 사이트  
> https://castle-of-gyu.tistory.com/87  
> https://github.com/querydsl/querydsl/issues/3428

## 문제 정의

QueryDSL을 사용해서 각 행으로 분리된 권한 데이터를 List로 합치는 과정에서 아래와 같은 문제에 직면했다.

> 예를 들어, 아래와 같이 테이블이 있을 경우 권한 부분을 합쳐서 가져온다.
>
> | 게시글ID | 권한     |
> | -------- | -------- |
> | 1        | USER     |
> | 1        | CUSTOMER |
>
> { ID: 1, Role: [USER, CUSTOMER] }

기존에 `string_agg` 방식으로 구현했을 때 발생한 문제는 아니고, `transform`방식으로 변경하려다 발생한 문제이다.

```java
jakarta.servlet.ServletException: Handler dispatch failed: java.lang.NoSuchMethodError: 'java.lang.Object org.hibernate.ScrollableResults.get(int)'
...
Caused by: java.lang.NoSuchMethodError: 'java.lang.Object org.hibernate.ScrollableResults.get(int)'
```

기존 코드에서 QueryDSL의 `Expressions.stringTemplate()` 과 `string_agg`를 사용하여 String 타입의 `권한1,권한2,권한3` 구조로 가져온 후 콤마(,)를 기준으로 배열로 분류하는 구조였다.

```java
// "권한1,권한2,권한3"
Expressions.stringTemplate("string_agg({0}, ',')", memberRole.id.role.roleType)

// [권한1, 권한2, 권한3]
result.split(",");
```

하지만 `string_agg`은 DB 종속적인 함수면서 정적 문자열로 사용해야 한다는 점에서 추후 DB 마이그레이션 과정에서 문제가 생길 것을 우려해 `transform`을 도입하기로 결정했다.

이 transform 코드는 아래와 같다.
transform의 동작은 transform 호출 직전의 쿼리 결과를 가지고 변형하는 동작이다.

> 아래 코드에서는 쿼리 결과를 가져와서 member id로 그룹화하고,  
> 기존 필드는 그대로 constructor에 넣고, List로 표시하려는 role 필드만 GroupBy.list를 사용해주어 권한을 하나의 List로 가져온다.

```java
queryFactory
        .from(member)
        ... // 생략
        .transform(
                groupBy(member.id).list(
                        Projections.constructor(MemberDetails.class,
                                member.id,
                                member.nickname,
                                member.email,
                                member.department.id,
                                GroupBy.list(memberRole.id.role.roleType)))
        ).get(0);
```

하지만 위 쿼리 실행 시 아래와 같이 예외가 발생했다. 예외 내용에서는 `transform`이 반환한 `ScrollableResults` 인스턴스에서 `get(int)` 메서드를 찾을 수 없다는 내용이었다.

![image.png](/assets/img/post/querydsl-crash-1.png)

하지만 IDE에서는 정상적으로 보이고, 컴파일 시에도 아무런 예외나 에러를 반환하지 않았다.

![image.png](/assets/img/post/querydsl-crash-2.png)

## 해결책

이를 검색했을 때, Hibernate 6.x 버전과 QueryDSL의 5.0.0 버전이 호환되지 않아서 생기는 문제인 것을 알 수 있었다.

추가로 해결 방법이 있는데, `JPAQueryFactory` 빈을 등록하는 과정에서 `JPQLTemplates.DEFAULT` 를 추가해주면 문제를 해결할 수 있는 것으로 알게되었다.

### JPQLTemplates.DEFAULT를 넣으면 왜 동작이 될까?

우선 `JPQLTemplates.DEFAULT`를 넣지 않은 상태를 알아보자.

```java
@Bean
public JPAQueryFactory jpaQueryFactory(EntityManager em) {
    return new JPAQueryFactory(em);
}
```

위처럼 `EntityManager`만으로 생성하는 구조가 된다. 이렇게 되면 `EntityManager`를 통해 현재 사용중인 JPA 구현체를 감지한다.

> 나의 프로젝트에서 JPA 구현체는 `Hibernate`이다.

`Hibernate`가 감지되면, `Querydsl`은 Hibernate에 특화된 기능 사용을 위해 `HibernateJPQLTemplates`과 같은 전용 템플릿을 로드한다.

이 Hibernate 전용 템플릿은 `transform`과 같은 그룹화 작업을 수행할 때, 메모리를 효율적으로 사용하기 위해 Hibernate가 제공하는 `ScrollableResults`라는 특수한 기능을 사용하도록 설계되어있다.

> 이는 모든 결과를 메모리에 올리지 않고 스트리밍 방식으로 처리하여 성능을 높이는 최적화 기법이라고 한다.

하지만 이 `ScrollableResults` 기능과 그 안의 `get(int)` 메소드는 Hibernate 5.x 버전에 있던 기능이다. 현재 프로젝트에 사용중인 Spring Boot 3.4.3과 함께 사용중인 Hibernate 6.x 버전에서는 이 메소드가 사라졌다. 결국 구버전 Querydsl이 신버전 Hibernate에 제거된 기능을 호출하려다 `NoSuchMethodError`를 발생시킨 것이 문제였다.

---

이제 `JPQLTemplates.DEFAULT`를 넣었을 때를 살펴보자.

```java
@Bean
public JPAQueryFactory jpaQueryFactory(EntityManager em) {
    return new JPAQueryFactory(JPQLTemplates.DEFAULT, em);
}
```

`JPQLTemplates.DEFAULT`라는 표준 JPQL 템플릿을 직접 주입했으므로, Querydsl은 JPA 프로바이더를 감지하거나 Hibernate 전용 템플릿을 로드하는 과정을 건너뛴다.

이렇게 되면 표준 JPQL 방식으로 fallback 되면서 Hibernate 전용 기능을 사용하지 않게 된다.
즉, 위 `transform` 코드에서 `Hibernate` 전용 최적화 기능인 `ScrollableResults`의 존재 자체를 모르고, 표준 방식을 쓰게 된다.

대신, 어떤 환경에서나 동작하는 가장 안전하고 표준적인 방식으로 데이터를 처리한다. 보통은 `fetch()`를 통해 일단 모든 결과를 `List`로 메모리에 가져온 다음, 메모리 상에서 그룹화 작업을 수행하는 방식이다.

결과적으로 문제가 되었던 Hibernate 구버전의 `ScrollableResults.get(int)` 메소드를 전혀 호출하지 않으므로 `NoSuchMethodError`가 발생하지 않고 코드가 정상적으로 실행된다.

## 결론

구 버전 Hibernate에 최적화된 QueryDSL을 특정 환경에 최적화 하지 않고, 모든 환경에 안정적인 상태로 바꾸어주면서 문제를 해결할 수 있었다. 하지만 표준적인 방식으로 처리하기 때문에 Hibernate에서 얻을 수 있는 일부 성능적 이점이 사라질 가능성을 고려하여, STRING_AGG 방식을 유지하기로 결정했다.

QueryDSL이 개발 중단되어 생긴 문제라고 생각된다. `OpenFeign QueryDSL`이라는 새로운 선택지가 있는 것을 확인했고, 추후 마이그레이션을 통해 해당 문제를 해결할 수 있는지 점검할 계획이다.

### string agg 방식

String 타입으로 포맷을 넣어줘야 한다는 불안정성과 String 타입에 대해 리스트로 파싱이 필요하다.

```java
queryFactory.select(Projections.constructor(dto.class,
                ... // 생략
                Expressions.stringTemplate("string_agg({0}, ',')", memberRole.id.role.roleType))) // stringTemplate 사용으로 불필요한 파싱 필요
        .from(member)
        ...
        .fetchOne();

```

### transform

타입 안정성을 유지하면서 QueryDSL에 맞게 코드를 작성할 수 있다.

```java
queryFactory
        .from(member)
        ...
        .transform(
                groupBy(member.id).list(
                        Projections.constructor(dto.class,
                                ... // 생략
                                GroupBy.list(memberRole.id.role.roleType)))
        ).get(0);
```
