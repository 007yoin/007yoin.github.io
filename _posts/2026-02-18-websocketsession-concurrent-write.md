---
title: "WebSocketSession에 두 스레드가 동시에 write하면 왜 실패할까"
date: 2026-02-18 21:00:00 +0900
categories: [Backend, Spring]
tags: [spring, websocket, concurrency, tomcat]
---

## 한 줄 요약

Spring WebSocket에서 **같은 `WebSocketSession`에 대해 한 전송이 끝나기 전에(전송이 진행 중인 동안) 다른 스레드에서 `sendMessage()`가 호출되면**, 컨테이너(예: Tomcat)의 송신 상태 머신과 충돌해서 `TEXT_PARTIAL_WRITING` 같은 예외로 실패할 수 있다.  
해결의 핵심은 **"세션 단위 송신(write)을 직렬화(serialize)하는 것"**이다.

---

## 증상: 메시지가 간헐적으로 누락되고 예외가 찍힌다

처음엔 "특정 메시지가 안 온다"처럼 보일 수 있다.  
하지만 로그를 보면 실제로는 **전송 시도 자체가 예외로 실패**하는 경우가 많다.

아래는 대표적인 예외 형태다(불필요한 줄은 생략).

```text
java.lang.IllegalStateException: The remote endpoint was in state [TEXT_PARTIAL_WRITING]
which is an invalid state for called method
    at org.springframework.web.socket.adapter.standard.StandardWebSocketSession.sendTextMessage(StandardWebSocketSession.java:206)
    at org.springframework.web.socket.adapter.AbstractWebSocketSession.sendMessage(AbstractWebSocketSession.java:106)
    ...
```

`TEXT_PARTIAL_WRITING`은 말 그대로 **텍스트 프레임을 아직 쓰는 중(부분 write/flush 전)**인 상태다.  
이때 같은 세션에 대해 또 다른 전송이 들어오면 "지금 그 메서드 호출하면 안 된다"는 식으로 예외가 발생할 수 있다.

---

## 문제의 본질: "무슨 값을 보냈는가"가 아니라 "누가 동시에 쓰는가"

이 문제는 데이터 포맷이나 메시지 내용이 아니라, **동시성 모델**에서 발생한다.

대표적으로 아래처럼 "보내는 경로가 둘 이상"일 때 흔하게 나타난다.

- 요청/메시지 처리 스레드에서 즉시 응답(ack)을 보냄
- 스케줄러/비동기 작업/이벤트 콜백 등 다른 실행 컨텍스트에서 추가 메시지를 보냄

예시(문제 패턴을 단순화):

```java
@Override
protected void handleTextMessage(WebSocketSession session, TextMessage message) {
    try {
        // 스레드 A (예: nio-8080-exec-*)
        session.sendMessage(new TextMessage("ACK"));

        // 스레드 B (예: pool-*-thread-*)
        scheduledExecutorService.schedule(() -> {
            try {
                session.sendMessage(new TextMessage("DONE"));
            } catch (Exception e) {
                log.error("Failed to send DONE", e);
            }
        }, 1, TimeUnit.SECONDS);
    } catch (Exception e) {
        log.error("Failed to send ACK", e);
    }
}
```

이 구조에서 **스레드 A와 B의 전송 타이밍이 겹치면**,
컨테이너의 송신 엔드포인트가 "이미 쓰는 중인데 또 쓰려고 한다"고 판단하여 예외를 던질 수 있다.

> 핵심: 한 세션의 송신(write)은 **한 번에 하나씩** 진행되어야 안전하다.

---

## 왜 이런 제약이 생기나

WebSocket 전송은 내부적으로 "그냥 문자열을 출력 스트림에 쓰기"보다 복잡하다.

- 텍스트 프레임 생성/인코딩
- 네트워크 버퍼로의 write
- 플러시/완료까지의 상태 관리

특히 Tomcat 같은 구현에서는 송신 엔드포인트가 **상태 머신을 통해 전송 단계(예: partial writing)를 관리**한다.  
컨테이너 구현별 내부 정책(상태/락/버퍼 처리)은 다를 수 있지만, 한 세션에서 write가 겹치면 비슷한 충돌 위험이 있다.

---

## 해결 전략: 세션별 송신을 직렬화하라

아래는 실무/학습에서 많이 쓰는 대표 선택지들이다.

### 1) 빠른 대응: 세션 단위 락으로 `sendMessage()` 직렬화

가장 빠르게 불을 끄는 방법은 "한 세션에 대해 send는 한 번에 하나"가 되도록 잠그는 것이다.

```java
private void sendSafely(WebSocketSession session, TextMessage message) {
    synchronized (session) {
        try {
            session.sendMessage(message);
        } catch (Exception e) {
            // 예외는 stacktrace 포함해서 남기는 게 디버깅에 유리하다
            log.error("Failed to send message. sessionId={}, payload={}",
                      session.getId(), message.getPayload(), e);
        }
    }
}
```

장점: 적용이 가장 빠르다.  
한계: 병목, 느린 클라이언트 영향, 백프레셔 제어 부재. 장기 운영 해법으로는 약하다.

> 참고: 더 명시적으로 관리하고 싶다면 `session.getId()`를 키로 별도 `Lock`(또는 `Map<sessionId, Lock>`)을 두는 방식도 많이 쓴다.

---

### 2) 기본: `ConcurrentWebSocketSessionDecorator` 사용 (Spring 제공)

Spring은 동시 send와 버퍼링을 제어하기 위한 데코레이터를 제공한다.
내부 동작을 코드 기준으로 자세히 보고 싶다면
[`ConcurrentWebSocketSessionDecorator 내부 구현을 알아보자`]({% post_url 2026-02-19-concurrentwebsocketsessiondecorator-internals %})를 이어서 보면 된다.

```java
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.ConcurrentWebSocketSessionDecorator;

WebSocketSession safeSession =
    new ConcurrentWebSocketSessionDecorator(session, 5000, 1024 * 1024);
```

생성자 시그니처는 `ConcurrentWebSocketSessionDecorator(delegate, sendTimeLimit, bufferSizeLimit)` 형태다.

- `delegate`: 실제 전송을 수행할 원본 `WebSocketSession`
- `sendTimeLimit`: 전송 제한 시간(ms)
- `bufferSizeLimit`: 버퍼 제한 크기(bytes)

장점: 락 방식보다 안정적이고, 느린 소비자 상황 보호까지 가능하다.

> 주의: `safeSession`을 만들기만 하면 자동 적용되는 게 아니라, **이후 모든 전송 경로에서 반드시 `safeSession`으로 send**해야 효과가 있다.

---

### 3) 정석: 아웃바운드 큐 + 단일 송신자(Writer)로 아키텍처 분리

조금 더 구조적으로 가면, 세션마다 outbound queue를 두고 **단일 소비자(송신자)**만 실제 `sendMessage()`를 호출하게 만든다.

- 여러 스레드(생산자)는 큐에 메시지만 넣는다
- 한 스레드(소비자)가 순서대로 꺼내서 send한다

장점:
- 송신 순서 보장
- 동시 send 제거
- 확장성/관측성(큐 길이 등) 개선

단점:
- 구현이 조금 더 커짐

---

## 운영/디버깅 팁

- `sendMessage()` 예외는 **stacktrace 포함 로그**로 남겨야 원인 추적이 쉽다.
- "메시지가 안 온다"는 증상은 실제로는 **전송 실패(예외)**인 경우가 많다.
- 서로 다른 실행 컨텍스트(예: 메시지 핸들러, 스케줄러, 비동기 작업)가 같은 세션에 write하는 경로를 먼저 의심하는 게 빠르다.
- 코드 베이스에서 send 경로가 여러 곳에 흩어져 있지 않은지 점검하자.
  - 메시지 핸들러
  - 스케줄러/비동기 작업
  - 이벤트 리스너/콜백
  - 서비스 계층의 임의 send

---

## 정리

`WebSocketSession`에서 발생하는 `TEXT_PARTIAL_WRITING`류 예외는
대부분 "특정 입력값" 문제가 아니라 **동일 세션에 대한 동시 write** 문제다.

문제를 해결하려면, 세션별 송신을 한 곳으로 모으고(단일 경로),
반드시 **write를 직렬화**하도록 설계하는 것이 가장 확실하다.

---

## 참고

- Spring WebSocket `WebSocketSession` Javadoc: 동시 send 관련 주의 및 데코레이터 언급  
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/socket/WebSocketSession.html
- `ConcurrentWebSocketSessionDecorator` Javadoc  
  https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/socket/handler/ConcurrentWebSocketSessionDecorator.html
