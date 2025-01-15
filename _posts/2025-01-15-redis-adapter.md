---
title: "다중 인스턴스 환경을 고려해서 소켓 통신 동기화"
author: rdyjun
excerpt: "다중 인스턴스 환경에서의 소켓 통신 동기화"

categories: [Socket]
tags: [socket, redis, nestjs]

toc: true
toc_sticky: true

date: 2025-01-15 14:51:00 +0900
last_modified_at: 2025-01-15 14:51:00 +0900
image: https://github.com/user-attachments/assets/92bc351a-b7a4-4de6-a905-6781fd5938f2
pin: false
---

[Socket.IO Redis adapter 공식문서 링크](https://socket.io/docs/v4/redis-adapter/)

지난 프로젝트를 하면서 채팅, 투표 등 실시간 데이터 반영을 위해 Socket.IO를 사용했었다.  
당장은 하나의 인스턴스로 서버를 돌리고 있기 때문에 문제가 없지만,  
오토 스케일링을 사용한다고 가정하고 이때 발생할 수 있는 문제를 미리 개선해보기 위해 시도하게 되었다.

Socket.IO에서는 `Redis adapter`라는 대표적인 기술을 통해 다중 인스턴스 상황에서도  
연결된 서버에 관계없이 동기화된다고 하여, 선택하게되었다.

## Redis adapter

그래서 Redis adapter는 뭘까?

- Pub/Sub 매커니즘
- Socket.IO 서버 간 메시지 동기화
- 실제 Redis에 데이터가 저장되는 구조는 아니다.
- 레디스 서버가 끊어질 경우 패킷은 각 인스턴스에 연결된 클라이언트로만 전송된다.

위 특징을 봤을 때, 실제 Redis에 데이터가 저장되지 않는 다는 점에서  
소켓 서버 객체 자체를 Redis에 저장하지 않는다는 것을 알 수 있다.

### 흐름

아래 사진을 보면 서버마다 Redis adapter가 하나씩 존재하고, 바깥의 Redis와 통신하고있다.  
그리고 아까 얘기한 Redis adapter의 가장 큰 특징은 Pub/Sub 구조이다.  
쉽게 말해서 각 서버의 Redis adapter가 Redis와 통신하여 동기화한다는 것이다.

![redis adapter](https://github.com/user-attachments/assets/92bc351a-b7a4-4de6-a905-6781fd5938f2)

출처: Socket.io

이를 예시와 함께 얘기해보겠다.
클라이언트가 `채팅 입력 이벤트(Socket)`를 발생시키고, 이에따라 서버에서는 특정 채팅방(A라고 가정) 사용자에게 `채팅 출력 이벤트(Socket)`를 발생시킨다고 가정하자.

이때 클라이언트와 직접 연결된 서버는 자신과 연결된 클라이언트이면서 A라는 채팅방에 참여한 사용자들의 소켓에 `채팅 출력 이벤트(Socket)`를 발생시키고,  
Redis adapter를 통해 위 동작에 대한 메시지를 발행(Publish)한다.

이렇게 Redis에 메시지를 발행하면 다른 인스턴스들의 각 Redis adapter는 Redis를 Subscribe하다가,  
메시지가 발행된 것을 알아채어 각 서버와 연결된 클라이언트에게 `채팅 출력 이벤트(Socket)`를 발생시킨다.

위와같은 흐름으로 Socket 서버가 동기화되는 것을 이해할 수 있다.

### Code

위 흐름대로라면, 각 인스턴스는 소켓 서버가 분리되어있다.  
즉, 기존 gateway 코드는 수정할 필요가 없다는 의미이다.

서버에서 Socket 이벤트를 발생시킬 때 Redis로 발행(Publish)하고 구독(Subscribe)하는 adapter의 구현이 필요하다.

> redis-io.adapter.ts

```js
import { IoAdapter } from '@nestjs/platform-socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { ServerOptions } from 'socket.io';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  // Redis와 연결된 adapter 생성 메소드
  // pub/sub을 위한 client 생성 후 이를 통해 adapter 생성
  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: 'redis://localhost:6379' });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  // main.ts에서 redis 어댑터 생성시 호출됨
  // 소켓 서버 생성 시 adapter를 등록해준다.
  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

---

> main.ts

Redis adapter 인스턴스를 생성 -> Redis와 연결 -> 해당 adapter를 socket adapter로 등록

```js
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();
app.useWebSocketAdapter(redisIoAdapter);
```

## 테스트

테스트를 위해서 두 인스턴스를 기준으로 테스트를 해야 했다.

초기 버전으로 port만 다르게 로컬 서버를 두 개 띄운 상태에서 각 서버에 연결할
**client1(port: 3000)**, **client2(port: 3001)**를 만들고,

client1이 message를 보내면 client2가 응답받는 것을 목표로 한다.

### redis adapter 적용 전 테스트 결과

client1이 message 이벤트를 보냈을 때 client2가 broadcast 이벤트를 받지 못해 timeout되는 것을 볼 수 있다.

![테스트 실패](https://github.com/user-attachments/assets/d6c6b622-bd3a-4456-90eb-28b3828a689d)

### Code

```jsx
import io from "socket.io-client";

describe("Socket.IO Redis Test", () => {
  let client1, client2;

  beforeAll(async () => {
    // 3000포트 서버와 연결할 client1 생성
    client1 = io("http://localhost:3000/rooms", {
      autoConnect: false,
      reconnectionAttempts: 3,
      query: {
        roomId: "room-1"
      }
    });

    // 3001포트 서버와 연결할 client2 생성
    client2 = io("http://localhost:3001/rooms", {
      autoConnect: false,
      reconnectionAttempts: 3,
      query: {
        roomId: "room-1"
      }
    });

    // client1, client2 연결
    await Promise.all([client1.connect(), client2.connect()]);
  });

  test("메시지가 모든 클라이언트에 전달되어야 함", (done) => {
    client2.on("broadcast", (data) => {
      expect(data.message).toBe("test message");
      done();
    });

    client1.emit("message", {
      roomId: "room-1",
      message: "test message"
    });
  });

  afterAll(() => {
    client1.disconnect();
    client2.disconnect();
    client1.removeAllListeners();
    client2.removeAllListeners();
  });
});
```

### 결과

![테스트 성공](https://github.com/user-attachments/assets/5e94a4ea-e6e9-4b47-ab50-46490eb1aa09)
