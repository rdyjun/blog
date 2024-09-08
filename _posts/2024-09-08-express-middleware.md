---
title: "[node] 미들웨어가 뭘까?"
author: rdyjun
excerpt: "요청과 응답 사이의 일꾼"

categories: [node]
tags: [middleware, node, 미들웨어, 노드]

toc: true
toc_sticky: true

date: 2024-09-08 18:31:00 +0800
last_modified_at: 2024-09-08 18:31:00 +0800

pin: false
---

미들웨어란 요청과 응답 사이에서 요청을 가로채거나,  
요청 전후 추가 작업을 수행하는 함수이다.

간단한 예시로, 사용자 인증, 로깅 등의 작업이 있다.

## 🔧 미들웨어는 어떻게 사용하는 것인가?

express 프레임 워크를 기준으로,  
express 모듈의 express() 함수를 호출하게 될 경우  
express 어플리케이션 객체가 반환된다.  
이렇게 반환된 객체에 `.use()` 메소드를 사용하여 미들웨어를 등록할 수 있다.

```js
import express from "express";
const app = express(); // express 어플리케이션 객체 인스턴스 반환

app.use(); // 미들웨어 등록
```

그러면 use()에 들어갈 파라미터는 어떻게 되는지 알아보자.  
`.use()` 메소드는 여러 방식으로 `오버로딩(overloading)`되어,  
다양한 방식으로 미들웨어를 등록할 수 있다.

### ✍️ 기본 미들웨어 함수 등록

미들웨어를 콜백함수 형태로 파라미터에 전달하는 방식이다.

```js
app.use((req, res, next) => {
  console.log(req.body);
  next();
});

app.use(middlewareCallback);
```

파라미터는 순서대로 `request`, `response`, `next`를 지원하며,  
`request`는 요청에 대한 정보를 담고,  
`response`는 응답에 대한 정보를 담는다.  
`next`는 다음 미들웨어를 실행하기 위한 콜백함수이다.

물론 각 파라미터는 사용할 수도 사용하지 않을 수도 있다.  
가장 우측부터 파라미터가 사용되지 않는다면 작성하지 않아도 된다.

```js
app.use(콜백함수);

app.use((req) => {
  console.log(req.body);
});

// 좌측 파라미터가 사용되지 않는다면 _를 명시하는 것도 하나의 방법이다.
app.use((_req, _res, next) => {
  console.log(req.body);
  next();
});
```

### 🛤️ 경로를 지정한 미들웨어 등록

첫 마디에도 요청과 응답 사이에서 동작하는 것이라고 설명했었다.  
이는 `어떤 요청과 응답 사이에서 동작하는 미들웨어이냐?` 에 대한 질문으로도 이어질 수 있을 것 같다.

결론적으로 위에서 처럼 경로를 지정하지 않는 경우 모든 요청에 대한 미들웨어를 말한다.  
아래와 같이 경로를 지정한 경우, 해당 경로로의 요청을 위한 미들웨어를 의미한다.

```js
app.use(경로, 미들웨어콜백함수);

app.use("/test", (_req, res) => {
  res.send("message");
});

app.use("/test", middlewareCallback);
```

### 📦 하나의 경로에 여러 미들웨어 등록

하나의 경로에 대해서 여러 미들웨어를 등록할 수 있다.  
주로 미들웨어의 동작을 순차적으로 나누어서 관리할 때 사용할 수 있다.

```js
app.use(경로, 미들웨어콜백함수1, 미들웨어콜백함수2, ...);

app.use('/test',
  middlewareCallback,    // 미들웨어 1
  (req, res, next) => {  // 미들웨어 2
    res.render(req.body);
    next();
  }
)

app.use('/test/abc');
```

### 🧭 next()는 뭘까?

`next()` 콜백 함수는 다음 미들웨어 또는 라우터 핸들러로 넘어가는 역할을 한다.  
이를 반대로 얘기하자면, `next()` 콜백함수를 실행하지 않는다면,  
다음 미들웨어 또는 라우터 핸들러로 넘어가지 않게 된다.

```js
app.use(경로, 미들웨어콜백함수1, 미들웨어콜백함수2, ...);

app.use('/test',
  (req, res, next) => {  // 미들웨어 1, next()를 실행하지 않음
    res.render(req.body);
  },
  middlewareCallback    // 미들웨어 2, 실행되지 않음
)

app.use('/test/abc');
```

`next()` 콜백함수는 `.use()` 메소드로 등록된 미들웨어로만 이동하지 않는다.  
`.get()`, `.delete()` 등의 메소드로 등록된 `라우트 핸들러`로도 이동한다.

```js
app.use(경로, 미들웨어콜백함수1, 미들웨어콜백함수2, ...);

app.use('/test',
  (req, res, next) => {  // 미들웨어
    res.render(req.body);
    next(); // 라우트 핸들러 실행
  }
)

// 라우트 핸들러
// 위에서 실행된 next()를 통해 라우터 핸들러의 콜백함수가 실행
app.get('/test/abc', (req, res, next()) => {
  console.log('route handler');
});
```

## 🚦 라우트 핸들러는 또 뭐지?

라우트 핸들러는 경로(route)에 대해 처리하는 로직이다.  
앞에서는 `app.use(route, middleware)`를 통해 미들웨어를 등록시켜줬다면,  
`app.get()`, `app.post()`와 같이 라우트 핸들러를 사용할 수 있다.

```js
app.get("/test", (_, res) => {
  res.send("라우트 핸들러 테스트");
});

app.delete("/test", (_, res) => {
  res.send("라우트 핸들러 삭제 테스트");
});

app
  .get("/test", callback)
  .post("/test", callback)
  .delete("/test", callback)
  .put("/test", callback);
```

### ⚖️ 미들웨어 등록과의 차이점?

위 코드를 보면 미들웨어를 등록했을 때와 다른 점이,  
바로 `method`를 설정해줄 수 있다는 점이다.

미들웨어는 `요청 경로`까지만 범위를 적용할 수 있다면,  
라우트 핸들러는 `메소드`까지 지정해줄 수 있다.

이렇게 세부적으로 지정할 수 있는 만큼 실행 우선순위는 미들웨어에게 밀린다.

```js
app.use(경로, 미들웨어콜백함수1, 미들웨어콜백함수2, ...);

// 라우트 핸들러는 미들웨어에서 next()가 호출되고 실행된다.
app.get('/test', (req, res, next()) => {
  console.log('route handler');
});

// 미들웨어가 먼저 실행된다.
app.use('/test',
  (req, res, next) => {  // 미들웨어
    res.render(req.body);
    next(); // 라우트 핸들러 실행
  }
)
```

### 📂 라우트 핸들러를 app.js 외부에서 등록할 방법 없나..?

있다. 바로 `Router()`를 사용하는 것이다.  
우리는 여태 express의 application 인스턴스에 라우트 핸들러를 등록했다.  
하지만 express는 `.Router()` 메소드를 통해  
라우트 핸들러를 등록하는 새로운 방법을 지원하고 있다.

```js
// router.js ------------------------------------------------
import express from "express";

const router = express.Router(); // 미들웨어 생성

router
  .get(경로, 라우트핸들러) // 라우트 핸들러 등록
  .post(경로, 라우트핸들러)
  .delete(경로, 라우트핸들러);

export default router; // 라우트 핸들러가 등록된 미들웨어 반환

// app.js ---------------------------------------------------
app.use("/", indexRouter);
```

위와같이 작성한다면, 다양한 경로에 대해 Router 미들웨어를 만들고  
각 경로에 해당 미들웨어를 적용할 수 있다.  
그러면 여기서 `app 자체에 저런식으로 등록하지 못하나?`라고 생각할 수 있다.

```js
// app.js --------------------------------------
import express from "express";

const app = express();

export default app;

// route.js ------------------------------------
import app from "app.js";

app
  .get(경로, 라우트핸들러) // 라우트 핸들러 등록
  .post(경로, 라우트핸들러)
  .delete(경로, 라우트핸들러);
```

위 코드를 보고 잘못된 것을 감지해야 한다.  
app 객체가 전역 상태가 된다면 어떠한 사이드이펙트가 발생할지 모른다.  
순환참조가 생길수도 있고, 여러 파일에서 app객체를 수정하다보면 동작을 추적하기도 어려워진다.

그러면 또 한 가지 아이디어가 떠올랐을 것이다.  
다른 파일에서 `express()`를 호출하여 `express application`을 호출하는 것이다.  
하지만 새롭게 `express()`를 호출하면, 기존에 생성한 `express application`과  
`상호 독립적`으로 관리되어, `app`과 아무 관계없는 `express application` 인스턴스가 생성된다.

결과적으로 Router 미들웨어를 사용하여, 라우트 핸들러를 등록하는 방식이  
로직을 분리하면서 관리하기에 용이하다고 할 수 있다.

### 🔀 그렇다고 한다면 두 라우트 핸들러 등록 방식의 차이가 있는지?

`express.Router()`에 등록하는 것과 `express()`에 등록하는 방식에는  
로직적으로 큰 차이는 없으나 몇 가지 장/단점이 존재한다.

#### 🗂️ 모듈화

위에서의 내용과 같이 `express()` 등록하는 방식은`app.js`에 모두 등록해야 한다.  
즉, 모듈화가 불가능하다는 의미이다.  
반대로 `express.Router()`를 사용한다면 별도 .js에 관리할 수 있다.

#### 🛣️ 경로 관리

경로 관리에 있어서는 1번의 모듈화와 연결되는 부분이다.  
`express()`에 등록하는 방식은 모듈화가 불가능하다 보니,  
경로에 대해서 세부 경로 모두 지정해주어야 한다.

```js
import express from "express";

const app = express();

app
  .get("/test", 라우트핸들러) // 라우트 핸들러 등록
  .post("/test", 라우트핸들러)
  .delete("/test/:id", 라우트핸들러);
```

반대로 `express.Router()` 방식은 `app.js`에서 한 번에 경로 설정이 가능하다.

```js
// route.js -----------------------------------
import express from "express";

const router = express.Router();

router
  .get("/", 라우트핸들러)
  .post("/", 라우트핸들러)
  .delete("/:id", 라우트핸들러);

export default router;

// app.js -------------------------------------
import express from "express";
import router from "router.js";

const app = express();

app.use("/test", router); // 경로 일괄 등록
```

#### ♻️ 재사용성

재사용성에서도 마찬가지로 모듈화 특징과 연관된다.  
`express.Router()`를 사용한 경우 독립적인 테스트가 가능하고,  
다른 모듈에서 라우터를 재사용하게 될 경우 활용이 가능하다.

반대로 `express()`로 라우트 핸들러를 등록한 경우  
test에서 `app.js`를 통째로 import하여 사용해야 하기 때문에  
테스트 단위를 분리할 때 문제가 생길 수 있다.

```js
// express.Router() 방식 테스트 -----------------------------
const request = require("supertest");
const router = require("./router.js");
const express = require("express");

const app = express();
app.use("/api", router); // 라우트 미들웨어 단위로 테스트 가능

describe("router test", () => {
  it("test", (done) => {
    request(app).get("/test").expect(200, "success", done);
  });
});

// express() 방식 테스트 ------------------------------------
const request = require("supertest");
const app = require("./app.js"); // 모든 라우트 핸들러가 등록된 app을 import하여 테스트

describe("Test app routes", () => {
  it("test", (done) => {
    request(app).get("/test").expect(200, "success", done);
  });
});
```

## 🎯 마무리

미들웨어가 참 직관적이고 편리하게 사용할 수 있다는 것이 좋았다.

라우트 핸들러를 등록하는 방식이 두 가지가 있었는데,  
Router 미들웨어를 통해 모듈화하여 등록할 수 있다는 점이 좋았다.
