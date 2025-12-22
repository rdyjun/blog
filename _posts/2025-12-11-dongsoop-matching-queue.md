---
title: "실시간 매칭 구조 리팩터링: 낙관적 락에서 MQ로 전환하기까지"
author: rdyjun
excerpt: "MQ기반 대기열로 매칭 동시성 처리하기"

categories: [mq]
tags: [javascript, messqgequeue, mq, redis]

toc: true
toc_sticky: true

date: 2025-12-11 14:58:00 +0900
last_modified_at: 2025-12-11 14:58:00 +0900
pin: false
---

## 문제 상황

- 두 명 이상이 동시에 입장할 때, 세션 정원을 충족해도 입장이 가능하다고 판단하여 정원보다 많은 사용자가 한 세션에 매칭되는 문제
- 두 명 이상이 동시에 입장할 때, 새로운 세션을 생성하는 과정이 중복으로 실행되며 세션이 두 개 만들어지고, 먼저 생성된 세션에는 사용자가 정상적으로 매칭되지 않는 문제

## 낙관적 락

처음에는 Redis의 WATCH 기능을 이용해 입장·퇴장 시점의 데이터를 스냅샷처럼 보호하고, 변경 충돌이 발생하면 재시도하는 낙관적 락 방식을 선택했다.

```jsx
for (let i = 0; i < 3; i++) {
  this.redis.watch([participantsKey, memberKey]);
	const result = this.redis.multi()
	    .hAdd(participants, memberId)
	 		.hSet(memberKey, name, '익명')
	 		...
	 		.exec(); // 호출 시 자동으로 unwatch

	// 실패 시 재시도
  if (result == null) {
	  continue;
  }
}
```

하지만 실제 운영 시 다음과 같은 문제가 있었다.

### 감시해야 할 키가 많다.

- 회원 정보 키
- 세션 참여자 목록 키
- 세션 상태 키
- 세션 포인터 키

이처럼 여러 키를 동시에 감시하면 누구 하나만 갱신해도 전체 트랜잭션이 실패하기 때문에 동시 입장 상황에서 재시도가 빈번했다.

### 충돌 재시도는 순서를 보장하지 않는다.

WATCH는 충돌 여부만 알려줄 뿐 순서를 보장하지 않는다.

- 퇴장 이벤트
- 입장 이벤트

이 두 이벤트가 거의 동시에 발생하면 퇴장이 먼저 와야 하는 상황에서 입장이 먼저 처리되는 반전 현상이 발생할 수 있었다.

### 유지보수 난이도가 높다.

감시 대상 키가 많아지고, 재시도 로직도 고려해야 하며, 입장/퇴장/재입장의 흐름까지 세밀하게 관리해야 한다.
장기적으로 복잡한 상태 머신이 되어 실수 발생 가능성이 커질 것으로 판단했다.

이런 이유로 보다 안전하고 예측 가능한 구조가 필요했고, 결과적으로 이벤트를 직렬화하여 처리할 수 있는 대기열을 고려하게 되었다.

## 대기열 적용

문제의 본질은 **“동시에 여러 이벤트가 들어오고, 이를 Redis 상태 조회와 함께 처리한다”**는 점이었다.
동시성을 자연스럽게 해결하려면, 이벤트를 한 줄로 세워 순서대로 처리하는 구조가 필요했고, 이에 따라 Redis Message Queue를 도입했다.

우선 입장/퇴장 이벤트를 큐에 push하도록 구현했다.

```jsx
  async pushEnterQueue(data: { memberId: number; socketId: string }) {
    const json = JSON.stringify({
      type: BlindDateQueue.ENTER,
      timestamp: Date.now(),
      ...data,
    });

    await this.redisClient.lPush('blinddate:queue', json);
    await this.redisClient.publish('blinddate:queue:signal', 'new');
  }

  async pushLeaveQueue(data: { memberId: number; socketId: string }) {
    const json = JSON.stringify({
        type: BlindDateQueue.LEAVE,
        timestamp: Date.now(),
        ...data,
      });

    await this.redisClient.lPush('blinddate:queue', json);
    await this.redisClient.publish('blinddate:queue:signal', 'new');
  }
```

### 초기 구현: polling 기반 처리

음에는 단순한 `while` 루프를 이용해 큐를 계속 확인하는 방식으로 구현했다.

```jsx
  private async start() {
    console.log('[QueueConsumer] started');

    while (true) {
      const job = await this.redisClient.rPop('blinddate:queue');
      if (!job) {
	      await setTimeout(2000);
	      continue;
      }
      await this.process(JSON.parse(job) as JobType);
    }
  }
```

큐가 비어 있으면 2초 쉬었다가 다시 확인하는 방식이다.

### 스레드 점유 문제

하지만 무한 루프는 메인 스레드를 점유하는 문제로 이어졌다.

HTTP 요청 처리 지연 및 Socket.IO 이벤트 수신이 누락되었다.

이 구조는 단일 서버 환경에서는 괜찮아 보이지만 Node.js의 이벤트 루프 특성과 맞지 않아 전체 서버 응답성이 떨어지는 문제가 있었다.

### Pub/Sub 구조로 전환

이 문제를 해결하기 위해 **Message Queue에 이벤트가 들어왔을 때만 처리하는 이벤트 기반 구조**로 변경했다.

큐에 작업을 넣을 때 `publish`를 함께 실행하고,

subscriber는 해당 신호를 받았을 때만 큐를 확인하도록 구현했다.

```jsx
  private async start() {
    console.log('[QueueConsumer] started');

    const subscriber = this.redisClient.duplicate();
    await subscriber.connect();

    await subscriber.subscribe('blinddate:queue:signal', async () => {
      while (true) {
        const job = await this.redisClient.rPop('blinddate:queue');
        if (!job) {
          // 더 이상 처리할 게 없으면 중단
          break;
        }
        await this.process(JSON.parse(job) as JobType);
      }
    });
  }
```

적용 후 아래와 같이 문제들이 해결되었다.

- 불필요한 루프 제거
- 메인 스레드 점유 문제 해소
- HTTP 요청과 Socket.IO 이벤트 정상 처리
- <span style="color:red">~~모든 이벤트가 순서대로 안전하게 처리됨~~</span>

<span style="color:red">모든 이벤트가 순서대로 처리될 것이라고 생각했으나, 그렇지 않았다.</span>

### 안정화

위 구조에서 문제는 2가지이다.

1. while (true)는 문법적으로 종료 구문이 없어 실제 서비스에서 불안할 수 있다고 판단했다.
2. 동시에 signal이 들어오면 콜백함수가 동시에 처리된다.

이 문제를 다시 해결하기 위해 redis에서 지원하는 blocking pop방식을 사용해 기존 signal 을 제거하고, queue에 작업 목록이 들어올 때까지 대기하도록 구현했다.

여기서 while은 true가 아닌 AbortController를 사용해서 비동기 로직의 종료 시점을 검사하도록 수정했다.

> `signal.aborted`는 `controller.abort()`가 호출되기 전까지 `false`이기 때문에,
> 현재 구조에서는 `while (true)`와 동작상 큰 차이는 없다.
> 다만, 중단 가능성을 명시적으로 표현하고 외부 제어 지점을 열어두는 구조이므로,
> 확장성과 유지보수 측면에서 signal 기반 제어가 더 안전하다고 판단했다.

```jsx
public initServer(server: Server) {
    this.server = server;

    const { signal } = new AbortController();
    this.signal = signal;

    const subscriber = this.redisClient.duplicate();
    await subscriber.connect();

    while (!this.signal.aborted) {
        try {
            const queue = await subscriber.brPop('blinddate:queue', 0);
            if (!queue?.element) {
                continue;
            }
            await this.process(JSON.parse(queue.element) as JobType);
        } catch (e) {
            console.error(e);
        }
    }
}

```

## 결론

락을 사용해서 해결해보려고 했지만, 복잡도를 봤을 때 문제 해결 후에도 문제가 될 수 있다고 생각되었다.

결과적으로 대기열을 사용해 안정적으로 사용자 경험을 만들어 내는 것이 중요하다고 판단했다.
