---
title: "ConcurrentWebSocketSessionDecorator 내부 구현을 알아보자"
date: 2026-02-19 16:00:00 +0900
categories: [Backend, Spring]
tags: [spring, websocket, concurrency]
toc: true
---

## 한 줄 요약

`ConcurrentWebSocketSessionDecorator`는 같은 세션에 대한 동시 `sendMessage()` 호출을  
"단일 실제 송신 + 버퍼링 + 한도 초과 보호 종료" 흐름으로 바꿔주는 클래스다.

---

## 이 글에서 볼 것

코드를 위에서 아래로 따라가며 아래 4개만 잡으면 내부 동작이 거의 끝난다.

1. `sendMessage()`: 버퍼 적재 + flush 시도
2. `tryFlushMessageBuffer()`: 실제 직렬 송신
3. `checkSessionLimits()`: 느린 전송/버퍼 폭증 보호
4. `close()`: 필요 시 `SESSION_NOT_RELIABLE`로 상태 승격

---

## 먼저 큰 흐름

```java
sendMessage(message):
  if (shouldNotSend()) return

  buffer.add(message)
  bufferSize += payloadLength
  callback?.accept(message)

  do {
    if (!tryFlushMessageBuffer()) {
      checkSessionLimits()
      break
    }
  } while (!buffer.isEmpty() && !shouldNotSend())
```

핵심은 단순하다.

- 호출 스레드는 먼저 메시지를 버퍼에 넣는다.
- `flushLock`을 잡은 스레드만 실제 `delegate.sendMessage()`를 수행한다.
- 락을 못 잡은 스레드는 `checkSessionLimits()`에서 안전장치만 확인하고 빠진다.

즉, 다중 스레드 호출을 "단일 송신자 + 버퍼" 모델로 강제한다.

---

## `tryFlushMessageBuffer()`: 실제 직렬 송신 엔진

```java
private boolean tryFlushMessageBuffer() throws IOException {
    if (this.flushLock.tryLock()) {
        try {
            while (true) {
                WebSocketMessage<?> message = this.buffer.poll();
                if (message == null || shouldNotSend()) {
                    break;
                }
                this.bufferSize.addAndGet(-message.getPayloadLength());
                this.sendStartTime = System.currentTimeMillis();
                getDelegate().sendMessage(message);
                this.sendStartTime = 0;
            }
        }
        finally {
            this.sendStartTime = 0;
            this.flushLock.unlock();
        }
        return true;
    }
    return false;
}
```

여기서 많이 헷갈리는 포인트 2개:

### 1) 왜 `while (true)`로 계속 비우나?

코드 기반으로 보면 의도는 명확하다.

- 한 번 `flushLock`을 잡은 스레드가 큐를 최대한 배출해서 backlog를 빠르게 낮춘다.
- 메시지마다 lock/unlock 반복하는 비용을 줄인다.
- FIFO 큐를 단일 송신자가 비우기 때문에 순서 보존도 자연스럽다.

즉, "락 한 번 획득 -> 여러 메시지 연속 송신"으로 효율을 택한 설계다.

### 2) 왜 `sendStartTime`을 `now -> 0`으로 토글하나?

이 값은 `checkSessionLimits()`가 "현재 송신이 얼마나 오래 걸리는지"를 계산하는 기준값이다.

- 실제 send 직전에 `now` 기록
- send 끝나면 `0`으로 복원
- `finally`에서 다시 `0` 처리해서 예외가 나도 상태값이 남지 않게 보장

즉, `sendStartTime`은 전송 시간 제한(`sendTimeLimit`) 감지를 위한 상태 마커다.

> ⚠️ 중요한 점
>
> `sendTimeLimit`은 "추가 send 시도가 발생했을 때"만 검사된다.
> 즉, 단 한 번의 `delegate.sendMessage(...)`가 내부에서 오래 블로킹되는 경우,
> 다음 send 시도가 없으면 이 제한으로 즉시 끊기지 않을 수 있다.
>
> 따라서 실제 블로킹 방지는 컨테이너/서버 타임아웃 설정과 함께 고려해야 한다.

---

## `checkSessionLimits()`: 보호 정책의 핵심

```java
private void checkSessionLimits() {
    if (!shouldNotSend() && this.closeLock.tryLock()) {
        try {
            if (getTimeSinceSendStarted() > getSendTimeLimit()) {
                limitExceeded(...);
            }
            else if (getBufferSize() > getBufferSizeLimit()) {
                switch (this.overflowStrategy) {
                    case TERMINATE -> limitExceeded(...);
                    case DROP -> {
                        while (getBufferSize() > getBufferSizeLimit()) {
                            WebSocketMessage<?> message = this.buffer.poll();
                            if (message == null) {
                                break;
                            }
                            this.bufferSize.addAndGet(-message.getPayloadLength());
                        }
                    }
                    default -> throw new IllegalStateException(...);
                }
            }
        }
        finally {
            this.closeLock.unlock();
        }
    }
}
```

흐름을 그대로 풀면:

1. 이미 종료 상태면 검사하지 않는다 (`shouldNotSend`)
2. `closeLock.tryLock()`이 실패하면 대기하지 않고 빠진다
3. 먼저 전송 시간 초과를 검사한다
4. 그다음 버퍼 크기 초과를 검사한다
5. 초과 시 정책 분기:
   - `TERMINATE`: 즉시 `limitExceeded(...)`
   - `DROP`: 오래된 메시지부터 제거해 버퍼를 한도 아래로 내림

이 메서드는 "문제 발생 후 복구"보다 "세션 품질 악화 조기 차단"에 가깝다.

---

## `limitExceeded(...)`와 예외 의미

```java
private void limitExceeded(String reason) {
    this.limitExceeded = true;
    throw new SessionLimitExceededException(
        reason, CloseStatus.SESSION_NOT_RELIABLE);
}
```

단순 예외가 아니라 close status까지 같이 들고 간다.  
핵심 의미는 "이 세션은 더 이상 신뢰 가능한 상태가 아니다"다.

---

## `close()`가 close status를 바꿀 수 있는 이유

```java
@Override
public void close(CloseStatus status) throws IOException {
    if (this.closeLock.tryLock()) {
        try {
            if (this.closeInProgress) {
                return;
            }
            if (!CloseStatus.SESSION_NOT_RELIABLE.equals(status)) {
                try {
                    checkSessionLimits();
                }
                catch (SessionLimitExceededException ex) {
                    // ignore
                }
                if (this.limitExceeded) {
                    status = CloseStatus.SESSION_NOT_RELIABLE;
                }
            }
            this.closeInProgress = true;
            super.close(status);
        }
        finally {
            this.closeLock.unlock();
        }
    }
}
```

즉, 애플리케이션이 일반 close를 요청해도 내부 상태가 이미 한도 초과라면  
close status를 `SESSION_NOT_RELIABLE`로 바꿔서 종료 의미를 맞춘다.

---

## 운영 관점에서 꼭 볼 설정

기본값 (Spring WebSocket transport 설정 기준):

- `sendTimeLimit`: 10초
- `sendBufferSizeLimit`: 512KB

(설정은 WebSocketTransportRegistration에서 변경 가능)

1. `sendTimeLimit`
- 너무 작으면 일시적 네트워크 지연에도 끊긴다.
- 너무 크면 느린 세션이 오래 살아서 버퍼 누적을 키운다.

2. `bufferSizeLimit`
- 대략 `피크 유입량(bytes/s) x 허용 지연(s)`로 시작해서 실측으로 조정한다.

3. `overflowStrategy`
- `TERMINATE`: 유실보다 일관성이 중요할 때
- `DROP`: 최신 상태 전달이 중요한 스트리밍일 때

---

## 코드 리뷰 체크리스트

1. `sendMessage()`가 먼저 버퍼 적재 후 flush 시도하는지
2. 실제 송신 경로가 `flushLock.tryLock()`으로 단일화되는지
3. `sendStartTime`이 send 전후/예외 시점에 정확히 정리되는지
4. flush 실패 경로에서 `checkSessionLimits()`가 호출되는지
5. 버퍼 초과 시 `TERMINATE`/`DROP` 분기가 의도와 맞는지
6. `limitExceeded`가 `SESSION_NOT_RELIABLE`와 함께 처리되는지
7. `close()`에서 필요 시 close status 승격이 일어나는지

---

## 결론

`ConcurrentWebSocketSessionDecorator`는 단순한 락 래퍼가 아니다.

- 동시 send를 단일 송신으로 직렬화
- 경합 시 버퍼링
- 느린 전송/버퍼 폭증 보호
- close status 의미 보존

을 한 클래스에서 묶어 제공하는, 세션 단위 송신 안정성 정책 구현체다.

---

## 한계

- 세션 내부 송신 안정성은 보장하지만, 애플리케이션 전역 백프레셔/유량 제어까지 해결하지는 못한다.
- 종료 조건은 버퍼 초과만이 아니다. `sendTimeLimit` 초과 시에도 `SessionLimitExceededException`과 함께 `SESSION_NOT_RELIABLE` 종료로 이어진다.
- 데코레이터가 직렬화해도 실제 `delegate.sendMessage(...)` 자체에서 `IOException`/컨테이너 예외가 나면 전송은 실패한다.
- `DROP` 전략은 안정성을 위해 오래된 메시지를 버리므로, 전달 보장을 일부 포기한다.
- `sendTimeLimit`/`bufferSizeLimit` 값이 트래픽 특성과 맞지 않으면 과도한 종료 또는 과도한 유실이 생길 수 있다.

---

## 참고

- WebSocketSession Javadoc  
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/socket/WebSocketSession.html
- ConcurrentWebSocketSessionDecorator Javadoc (current)  
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/socket/handler/ConcurrentWebSocketSessionDecorator.html
- ConcurrentWebSocketSessionDecorator source (main)  
  https://raw.githubusercontent.com/spring-projects/spring-framework/main/spring-websocket/src/main/java/org/springframework/web/socket/handler/ConcurrentWebSocketSessionDecorator.java
