---
title: "return 할 때 await을 붙이면 어떻게 될까?"
author: rdyjun
excerpt: "await 유무에 따른 promise 동작"

categories: [javascript]
tags: [javascript, promise, await, async]

toc: true
toc_sticky: true

date: 2025-02-12 20:57:00 +0900
last_modified_at: 2025-02-12 20:57:00 +0900
pin: false
---

개발하다가 간혹 비동기 함수를 직접 return할 때가 있다.

처음에는 await을 꼭 붙여줬는데 지금은 await을 붙이지 않아도 된다는 사실을 알게되었다.

왜 await을 붙이지 않아도 되는 걸까?

## Promise 중첩

아래 코드의 `asyncFunction()` 함수는 비동기 로직을 수행하고,
가져온 결과 값에 추가 로직(+ 1)을 수행하여 return하기 때문에 await이 사용 되었다.

그리고 `a()` 함수는 이런 `asyncFunction()`함수를 호출하는 상태가 예시이다.

```jsx
async function asyncFunction() {
  const a = await Promise.resolve(2);

  return a + 1;
}

async function a() {
  return asyncFunction(); // await 제거
  return await asyncFunction(); // await 사용
}

a();
```

### Promise와 async, await 동작

먼저 `Promise`와 `await`의 동작을 간단하게 알아보자.

**Promise**는 비동기 처리를 위한 타입으로 비동기 함수를 실행시키면 즉시 **Promise<pending>**을 반환한다.

이후 함수 실행이 끝나고 **resolve(해결)** 상태로 바뀌면서 비동기 함수가 종료된다.

**await**은 비동기 처리를 기다리는 예약어이다.

비동기 로직을 실행시켜 Promise<pending> 상태일 때 다음 로직을 실행하지 않고,

Promise의 상태가 pending이 아닌 다른 상태로 업데이트 되면 다음 로직을 실행한다.

만약 `await new Promise(...);` 구조라면, `await new Promise(...);` 코드 이후의 코드들이
마이크로태스크 큐에 등록된다.

```jsx
// 원래 코드
const a = await PromiseTask();
return a + 1;

// 실제 동작
PromiseTask().then((value) => {
  const a = value;
  return a + 1;
});

// 만약 다음 동작이 없다면
PromiseTask().then((value) => value);
```

async함수에서 반환한 Promise 객체는 함수 호출 시 처음 반환한 Promise와 체이닝된다.

즉, 아래 코드에서 P2와 P1이 논리적으로 연결되며,

Promise.resolve(2)가 실행되어 P2의 값이 변형되면

마찬가지로 P1의 상태도 바뀌는 구조이다.

```jsx
async test() {
	return Promise.resolve(2); // return 시점에 P1과 체이닝 - p2
}

test() // 호출 즉시 Promise<pending> 반환 - P1
```

### await 사용

앞전의 코드를 예시로 `a()` 함수에서 `asyncFunction()` 함수를 호출할 때 `await`을 쓴 상황을 살펴보자.

![await-포함-흐름-설명.png](attachment:423dd082-9b80-40f8-b814-ebbb1d66a22b:await-포함-흐름-설명.png)

```jsx
async function asyncFunction() {
  const a = await Promise.resolve(2);

  return a + 1;
}

async function a() {
  return await asyncFunction(); // await 사용
}

a();
```

1. a()를 호출했을 때 a() 함수도 async 함수이기 때문에 즉시 `Promise<pending>` (P1)를 반환한다.
   - a() = Promise<pending>
2. a()함수의 내부에서 `asyncFunction()` 함수를 실행하는데
   이 함수도 비동기 함수이기 때문에 `Promise<pending>` (P2)를 반환한다. - asyncFunction() = Promise<pending>
3. await이 포함되었기 때문에 `await asyncFunction();` 이후의 작업이
   **마이크로 태스크 큐**에 저장된다. - `await asyncFunction();` 코드 이후에 코드가 없기 때문에
   `return promiseResult.then(value ⇒ value);` 형태로 저장된다.
4. P2의 .then() 결과인 Promise 객체(P3)가 P1과 체이닝된다.
   - 3번에서 마이크로 태스크 큐에 저장된 P2의 .then() 작업의 결과 Promise 객체를 의미
   - P1 = P3
5. asyncFunction() 함수 내부의 `await Promise.resolve(2);` (P4)이 실행된다.
   - Promise.resolve는 pending이 아닌 fulfilled 상태로 즉시 반환된다.
   - 이때 await을 사용했기 때문에 `await Promise.resolve(2);` 이후의 작업이
     마이크로 태스크 큐에 저장된다.
   - 이 작업은 await 코드 이후의 asyncFunction() 코드이다.
     ```jsx
     async function asyncFunction() {
       return Promise.resolve(2).then((value) => {
         const a = value;
         return a + 1;
       });
     }
     ```
6. 마이크로 태스크 큐에 저장된 P4의 .then() 작업은 P5이다.
   - P5는 P2와 체이닝 된다.
   - .then()은 언제나 독립적이고 새로운 Promise를 반환한다는 것을 명심하자.
7. P5 로직 즉, `asyncFunction()`의 나머지 코드들이 추가되고 실행된다.
   - P5의 값이 업데이트됨 3
8. P5 결과 3이 체이닝된 P2에도 업데이트된다.
9. P2의 상태 변경에 따라 P2.then() = P3을 실행한다.
10. P3이 업데이트 됨에 따라 P1도 업데이트 된다.
11. 종료

위 흐름대로라면 P1은 P3과 연결되고, P3은 P2의 종료를 기다리는 구조가 된다.

또한 8번에서 `asyncFunction()`함수의 결과 값을

불필요하게 `.then(value ⇒ value);` 형태로 실행하는 과정이 추가된다.

### await 미사용

반대로 await을 쓰지 않는 상황을 보자.

![await-흐름-설명.png](attachment:4f286be5-b629-4e3a-85e7-dfe61c42b166:await-흐름-설명.png)

```jsx
async function asyncFunction() {
  const a = await Promise.resolve(2);

  return a + 1;
}

async function a() {
  return asyncFunction(); // await 미사용
}

a();
```

1. a()를 호출했을 때 a() 함수도 async 함수이기 때문에 즉시 `Promise<pending>` (P1)를 반환한다.
   - a() = Promise<pending>
2. a()함수의 내부에서 `asyncFunction()` 함수를 실행하는데
   이 함수도 비동기 함수이기 때문에 `Promise<pending>` (P2)를 반환한다. - asyncFunction() = Promise<pending>
3. await이 없기 때문에 P2는 P1과 체이닝된다.
4. asyncFunction() 함수 내부의 `await Promise.resolve(2);` (P3)이 실행된다.
   - 이때 await을 사용했기 때문에 `await Promise.resolve(2);` 이후의 작업이
     마이크로 태스크 큐에 저장된다.
   - 이 작업은 await 코드 이후의 asyncFunction() 코드이다.
     ```jsx
     async function asyncFunction() {
       return Promise.resolve(2).then((value) => {
         const a = value;
         return a + 1;
       });
     }
     ```
5. 마이크로 태스크 큐에 저장된 P3의 .then() 작업은 P4이다.
   - .then()은 언제나 독립적이고 새로운 Promise를 반환한다는 것을 명심하자.
6. P4 로직 즉, `asyncFunction()`의 나머지 코드들이 추가되고 실행된다.
   - 실행된 결과는 P4에 업데이트 되었다.
7. 6번의 P4 결과 3이 체이닝된 P2, P1에도 업데이트된다.
8. 종료

위 과정에서 P4 값이 업데이트 되면 체이닝 된 P1, P2 값도 업데이트 되어
불필요하게 각 Promise에 값을 기다리지 않고,

await을 사용했을 때 추가되는 마이크로태스크 큐 작업이 제거되어 더 효율적이다.

## 내용을 작성하면서 궁금했던 점

async 함수에 Promise를 return하는 것과,
일반 값(number, string 등)을 직접 반환하는 것은 무슨 차이가 있을까?

```jsx
async test() {
	return 1;
}

async test() {
	return Promise.resolve(1);
}
```

두 코드의 공통점은 호출 즉시 Promise<pending>을 반환한다는 점이다.

`return 1`부터 동작을 살펴보니, 호출 시점에서 반환된 Promise 객체에 1이라는 값을 저장하여 상태를 업데이트한다.

`return Promise.resolve(1);`의 동작을 살펴보니, Promise 객체를 생성하고, 호출 시점에서 반환 된 Promise 객체와 체이닝된다.

즉, 서로 다른 객체지만 논리적으로 값이 연동된 상태이다.

## 결론

비동기 로직 호출 즉시 return하는 경우 await을 사용하지 않고 바로 return 하는 것이

`promiseResult.then(value ⇒ value)`와 같은 불필요한 작업을 줄이고,

실행 후 반환한 Promise 객체(P1)와 호출 즉시 반환한 Promise 객체(P2)가

직접 체이닝하여 더 자연스럽게 동작하게 된다.

만약 return한 Promise 데이터를 써야 한다면 가장 상위에서 await을 사용하는 것이 적합해 보인다.
