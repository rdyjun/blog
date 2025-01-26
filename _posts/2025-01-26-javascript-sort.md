---
title: "자바스크립트의 sort"
author: rdyjun
excerpt: "의도되지 않은 숫자 정렬(?)"

categories: [Socket]
tags: [javascript, sort]

toc: true
toc_sticky: true

date: 2025-01-26 23:11:25 +0900
last_modified_at: 2025-01-26 23:11:25 +0900
pin: false
---

[MDN 참조 링크](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)

코딩테스트 문제 풀이 중 계속 실패하여 엣지 케이스를 찾으려 했으나, 아무리 찾아도 엣지 케이스가 나오지 않는 문제가 발생했다.  
AI의 도움을 받아 `array.sort()` 부분이 문제라는 것을 알아냈다.

## 이 sort는 타입을 고려하지 않는다

자바스크립트에서 `array.sort()`는 배열의 요소를 `문자열`로 변환한 다음 UTF-16 코드 값 시퀀스를 비교한다고 한다.  
이 말은 즉, 배열에 숫자가 들어있을 때 문자열로써 비교된다는 의미이다.

```js
const array1 = [1, 30, 4, 21, 100000];
array1.sort();
console.log(array1);
// Expected output: Array [1, 100000, 21, 30, 4]
```

## 해결 방법

단순히 파라미터에 콜백함수를 추가하면 된다.  
이 콜백함수는 비교할 두 값에 대해 어떻게 정렬할 것인지에 대한 콜백함수이다.

```js
const array = [1, 30, 4, 21, 100000];
array.sort((a, b) => a - b); // 콜백함수 추가
console.log(array);
// Expected output: Array [1, 4, 21, 30, 100000]
```
