---
title: "[CODE STYLE] 자바 코딩 컨벤션"
author: rdyjun
excerpt: "깨끗한 자바 프로그래밍을 위한 코드 작성 협약"

categories: [코드 스타일]
tags: [java, code style, convention, 컨벤션, 스타일]

toc: true
toc_sticky: true

date: 2024-02-28 18:00:00 +0800
last_modified_at: 2024-02-29 18:00:00 +0800

pin: false
---

## 📛 네이밍

공통: 임시 변수 외에는 1글자 사용금지

1. 영어(a-z, A-Z), 숫자(0-9), 언더스코어(\_)만 사용
2. 고유명사를 제외한 한국어 발음 표기 방식 금지
3. 대문자료 표기할 약어 명시
   - API를 대문자로 사용하기로 했을 경우
     HttpApi 가 아닌 HttpAPI
   - HTTP를 대문자로 사용하기로 했을 경우
     HttpApi 가 아닌 HTTPApi
4. 패키지
   - 소문자만 사용 (대문자 및 언더스코어 X)
5. 클래스
   - 명사 사용
   - 대문자로 시작하는 Camel Case
   - Myclass, ClassName
6. 인터페이스
   - 명사/형용사 사용
7. 메서드
   - 동사/전치사로 시작
   - 소문자로 시작하는 Camel Case
   - myMethod, methodName
8. 테스트 케이스
   - 접미사가 Test로 끝남
9. 변수

- 소문자로 시작하는 Camel Case
- myVariable, variableName

11. 상수

- 모두 대문자로 작성하고, 단어 사이는 언더 스코어(\_) 적
- MY_CONSTANT, CONSTANT_NAME

## 🏛️ 형식

1. else, catch, finally 등 닫는 중괄호와 같은 열에 작성

   ```java
   if () {

   } else {

   }

   try {

   } catch () {

   } finally {

   }
   ```

2. 라인 길이

- 코드의 한 줄이 80 ~ 100자를 넘기지 않기

3. 제어문 (for, while, if, switch 등) 사이 공백 추가

   ```java
   if (true) {}

   for (int i = 0; i < n; i++) {}

   while (true) {}
   ```

4. 원시 값 포장

   ```java
   // X
   public int area(int radius) {
       return radius * 3.14;
   }

   // O
   private static final int PI;

   public int sum(int radius) {
       return radius * PI;
   }
   ```

5. else 예약어 지양

   - return 또는 continue 사용

6. 마지막 줄은 개행으로 처리

7. import 및 package는 각 한 줄로

   - package 선언 후 빈 줄 삽입
   - import 선언 순서 및 빈 줄 삽입
     1. static imports
     2. java.
     3. javax.
     4. org.
     5. net.
     6. 8 ~ 10을 제외한 com.\*
     7. 1 ~ 6, 8 ~ 10을 제외한 패키지에 있는 클래스
     8. com.nhmcorp.
     9. com.navercorp.
     10. com.naver.

8. 줄바꿈 후 들여쓰기

   - 이전 줄과 내용이 이어진다면
     최소 한 번 이상 들여쓰기

9. 메서드간 빈 줄 삽입

10. 공백으로 줄 끝내기 X

11. 대괄호 뒤에 공백 삽입

```java
int[] arr = new int[] {0, 1, 2};
```

12. 생성자, 메서드 선언, 호출 및 어노테이션에 쓰이는
    소괄호에 공백을 삽입하지 않는다

13. 타입 캐스팅 시에도 소괄호 내부 공백 미삽입

```java
String text = ( String ) object; X
String text = (String) object; O
```

14. 콤마(,) 및 구분자(/) 뒤 공백 삽입

15. 콜론(:) 앞 뒤 공백 삽입

16. 이항 및 삼항 연산자 앞 뒤 공백 삽입

17. 증감식 공백 미삽입

18. 주석 전후 공백 삽입

```java
/*
 * 위 아래 공백
*/

code // 앞뒤 공백

/* 앞뒤 공백 */
```

## 💬 주석

1. Class/Method 주석

   - `/** ... */`

2. Implementation 주석
   - `//` 또는 `/* ... */` 사용

## 🏰 구현

1. 하나의 함수가 한 가지 일만 하도록 가능한 작게 만들기

   - 하나의 테스트 함수 안에 Assertions.assertThat()가
     여러번 사용된다면 분리 검토

2. 소스 파일당 1개의 탑레벨 클래스를 담기

   - 추가 클래스가 필요할 경우
     탑클래스의 내부 클래스로 선언

3. import에 와일드카드 사용하지 않기

4. 제한자 선언 순서
   public protected private abstract static final transient volatile synchronized native strictfp

5. 어노테이션 선언 후 줄바꿈

   ```java
   @Override
   public void method() {}
   ```

6. 한 줄에 한 문장

7. 한 줄의 선언문에는 하나의 변수만

8. 배열 선언 시 대괄호는 타입 뒤

9. long 타입의 값의 접미사에 'L' 붙이기

10. 특수 문자의 전용 선언 방식을 활용

```java
System.out.print("\u000A") // x
System.out.print("\n") // O
```

## ⚙️ 설정

Appendix A : editorconfig 파일 설정

```java
# top-most EditorConfig file
root = true

[*]
# [encoding-utf8]
charset = utf-8

# [newline-lf]
end_of_line = lf

# [newline-eof]
insert_final_newline = true

[*.bat]
end_of_line = crlf

[*.java]
# [indentation-tab]
indent_style = tab

# [4-spaces-tab]
indent_size = 4
tab_width = 4

# [no-trailing-spaces]
trim_trailing_whitespace = true

[line-length-120]
max_line_length = 120
```

Formatter 자동 적용

- https://github.com/naver/hackday-conventions-java/blob/master/rule-config/naver-intellij-formatter.xml 에서 naver-intellij-formatter.xml 다운로드
  네이버 핵데이 컨벤션에서 제공하는 Formatter를 사용해 컨벤션을 쉽게 적용할 수 있다.

들여쓰기 설정

- File -> Settings -> Editor -> Code Style -> Java -> User tab character
- Tab Size, Indent : 4

Import 설정

- File -> Settings -> Editor -> Code Style -> Java -> Imports -> Import Layout
- General
  - Use single class import : 선택
  - Class count to use import with '\*' : 99
  - Names count to use static import with '\*' : 1
- Import Layout
  - import 선언의 순서 및 빈 줄 삽입 순서대로 지정
  - 모든 세부 그룹 사이에도 공백을 넣어야 Eclipse의 Formatter와 동일하게 정렬

줄바꿈 시 연산자 위치

- File -> Settings -> Editor -> Code Style -> Java -> Wrapping and Braces
- Binary expressions 아래 Operation sign on next line : 선택

소스 파일 최하단 공백줄 추가

- File -> Settings -> Editor -> General
- Ensure line feed at file and on Save : 선택

파일을 저장할 때마다 포맷터 자동 적용

- File -> Settings -> Plugins -> Marketplace -> Save Actions 검색
- Save Actions plugin의 상세 설명 화면에서 install 버튼 클릭
- Intellij 재시작
- File -> Settings -> Other Settings -> Save Actions
- Activate save actions on save : 선택
- Optimize imports : 선택
- Reformat file : 선택

일괄 변환

- 프로젝트의 홈디렉토리에 커서를 놓은 채로 아래 메뉴를 실행하면 프로젝트의 모든 소스에 해당 설정을 일괄 적용
- File -> Line Separators
- Code -> Reformat Code
- Code -> Auto-Indent Lines
- Code -> Optimize Imports

CheckStyle 설정

공백 문자 보이기 설정
탭과 스페이스가 섞여있는 프로젝트의 코드 정리 시
탭과 스페이스를 눈에 보이게 표시

- File -> Settings -> Editor -> General -> Appearnce
- Show whitespaces : 선택
- Leading, Inner, Trailing : 선택

들여쓰기에 탭 대신 스페이스 사용

- File -> Settings -> Editor -> Code Style -> Java -> Tabs and Indents
- Use tab character : 미선택
- Tab size : 4

줄바꿈 후 추가 들여쓰기 단계 조정

- File -> Settings -> Editor -> Code Style -> Java -> Tabs and Indents
- Continuation ident : 8
