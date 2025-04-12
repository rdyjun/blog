---
title: "싱글톤 클래스 관리: Bean vs Static 방식의 차이점"
author: rdyjun
excerpt: "Bean과 Static의 싱글톤 클래스 관리 방식 알아보기"

categories: [singleton]
tags: [java, spring, bean, static]

toc: true
toc_sticky: true

date: 2025-03-07 16:05:00 +0900
last_modified_at: 2025-03-07 16:05:00 +0900
pin: false
---

Java와 Spring Boot 프로젝트 개발 시 싱글톤은 자주 사용되는 패턴 중 하나이다.

인스턴스를 하나만 생성하여 메모리 효율성을 높이고 전역적으로 동일한 객체를 사용하기 위해 사용된다.

특히 데이터를 저장하기보다 로직을 처리하는 클래스에서 주로 사용된다.

이 싱글톤 클래스를 관리하는 방법은 여러가지이고, 이 글에서는 Bean 등록 방식과 static 메서드 방식을 비교하여 각자 어떤 상황에 적합한지 알아보자.

## Bean으로 등록하는 이유?

```java
interface Service {
	int sub(int a, int b);
}

@Service
class ServiceImpl implements Service {
	public int sub(int a, int b) {
		return a + b;  // 로직 변경 시 새 Service 구현체를 구현
	}
}

@Component
class Hello {
	private Service service;

	public Hello(Service service) {   // Service 의존성 주입
		this.service = service;
	}

	public int sub(int a, int b) {
		return this.service.sub(a, b);  // Service의 sub 메서드 호출
	}
}
```

위 코드에서 `ServiceImpl`은 `Service`를 구현하는 구현체로 Bean으로 등록되어 `Spring Container`에 의해 관리된다.

이는 로직 변경 시 새로운 구현체로 교체할 수 있도록 한다.

`Hello` 클래스는 `Service` 인터페이스를 의존하며 Spring의 DI를 통해 Service 구현체를 주입받는다.

이렇게 하면 sub 메서드의 로직이 변경되어도 새로운 구현체를 생성하여 쉽게 교체할 수 있다.

위처럼 구현체를 교체할 수 있다는 점은 유지보수성, **OCP(개방 폐쇄 원칙)**에 유리하다.

> 여기서 OCP는 확장에 열려있고 수정에 닫혀있다는 의미이다.
> 즉, 기존 코드는 수정하지 않고 기능을 확장할 수 있어야 한다는 의미이다.

## Static Method

```jsx
class Service {
	public static int sub(int a, int b) {
		return a - b;  // 로직 변경 시 내용 수정
	};
}

class Hello {
	public int sub(int a, int b) {
		return Service.sub(a, b);
	}
}
```

`Service` 클래스의 `sub` 메서드는 `static`으로 정의되어 있어 인스턴스 생성 없이 호출 가능하다.

그러나 로직 변경 시 직접 메서드 내부를 수정해야 하므로 **OCP(개방 폐쇄 원칙)**를 위반한다.

또한 static 메서드는 테스트 환경에서 Mocking이 어렵기 때문에 테스트 코드 작성에 불편함이 따른다.

### OCP 위반

Static 메서드는 객체 지향적 설계의 유연성을 제한한다.

예를 들어 `MathUtil.add()` 메서드를 호출하는 코드를 테스트하려고 할 때, 반환 값을 임의로 설정하거나 동작을 변경하려면 직접 메서드 내부를 수정해야 한다.

반면 Bean 방식에서는 DI를 통해 Mock 객체를 주입하여 테스트 환경을 쉽게 구성할 수 있다.

### 테스트 불편성

Mocking이 어렵다는 점에서 테스트 환경에 적용하기 어렵다.

예를 들어 아래 코드를 보면 performAddition 메소드가`MathUtil.add()를 호출하는 상황이다.

테스트 상황에서 MathUtil.add()의 반환 값을 10으로 임의 지정하고 싶을 때 static 메서드이기 때문에 override가 불가능하다.

```java
public class MathUtil {
    public static int add(int a, int b) {
        return a + b;
    }
}

public class Calculator {
    public int performAddition(int a, int b) {
        // 정적 메서드 호출
        return MathUtil.add(a, b);
    }
}

```

## Static은 쓰지 않는가?

그렇다고 static을 쓰지 않는 것은 아니다.

단순한 로직(sum, compare 등)을 담는 util은 static으로 구현하는 것이 더 직관적이다.

```java
public class MathUtil {
    public static int add(int a, int b) {
        return a + b;
    }

    public static int max() {
	    return Integer.MAX_VALUE;
    }

    public static int compare(int a, int b) {
	    if (a == b) {
		    return 0;
	    }

	    if (a > b) {
		    return 1;
	    }

	    return -1;
    }
}
```

## 결론

싱글톤 관리 방식에 따라 프로젝트의 유지보수성과 테스트 코드 작성에 영향을 끼친다.

비즈니스 로직을 담은 클래스들에 대해서는 Spring Boot Bean 등록 방식을 사용해서 클래스간 결합도를 낮추고,

단순한 연산 로직을 담은 클래스에 대해서는 static 방식을 사용해서 프로젝트의 유지보수성을 높이자.
