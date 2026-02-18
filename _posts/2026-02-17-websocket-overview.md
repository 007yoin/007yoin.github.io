---
title: "WebSocket이란 무엇인가"
date: 2026-02-17 21:00:00 +0900
categories: [Network]
tags: [websocket, http, realtime, network]
toc: true
---

## 한 줄 요약

**WebSocket은 클라이언트–서버가 하나의 연결을 오래 유지하면서, 양방향(Full-duplex)으로 실시간 메시지를 주고받기 위한 프로토콜**이다.  
HTTP로 “연결 업그레이드(Upgrade)”를 한 뒤에는, HTTP 요청/응답이 아니라 **WebSocket 프레임**으로 통신한다.

---

## 왜 WebSocket이 필요할까

HTTP는 기본적으로 **요청(Request)이 있어야 응답(Response)이 오는** 모델이다.  
즉, 서버가 “갑자기” 클라이언트에게 데이터를 보내고 싶어도, 전통적인 HTTP만으로는 매끄럽지 않다.

그래서 실시간이 필요한 서비스들은 보통 아래 같은 우회 방법을 썼다.

- **Polling**: 클라이언트가 주기적으로 “새 소식 있어?”라고 요청  
  - 단점: 불필요한 요청이 많아지고 지연(latency)이 생김
- **Long Polling**: 서버가 바로 응답하지 않고, 이벤트가 생길 때까지 연결을 잡아둔 뒤 응답  
  - 단점: 구현/리소스 관리가 까다롭고, 헤더 오버헤드가 큼

WebSocket은 이런 문제를 해결하기 위해,
- 연결을 **한 번 맺어 두고**
- 그 연결 위에서 **서버와 클라이언트가 자유롭게 데이터를 주고받는**
방식을 제공한다.

---

## 방식 비교: Polling vs Long Polling vs WebSocket

| 방식 | 실시간성 | 서버 부하 | 특징 |
|---|---|---|---|
| Polling | 낮음 | 높음 | 단순하지만 비효율적 |
| Long Polling | 중간 | 중간 | 이벤트 발생 시까지 대기 |
| WebSocket | 매우 높음 | 낮음 | 초기 연결 후 자유로운 양방향 통신 |

---

## WebSocket의 핵심 특징

### 1) 하나의 연결을 오래 유지(Persistent Connection)

한 번 연결이 성립되면 끊기기 전까지 같은 TCP 연결 위에서 통신한다.  
요청/응답처럼 매번 연결을 새로 잡지 않는다.

### 2) 양방향(Full-duplex) 통신

- 서버 → 클라이언트: 서버가 원할 때 push 가능
- 클라이언트 → 서버: 클라이언트도 언제든 send 가능

### 3) 메시지 오버헤드가 작다

HTTP는 요청마다 헤더가 반복되지만, WebSocket은 한 번 연결 후 작은 프레임 단위로 통신하므로 오버헤드가 줄어든다.  
(물론 애플리케이션 메시지 포맷(JSON 등)에 따라 실제 효율은 달라질 수 있다)

### 4) 텍스트뿐 아니라 바이너리도 전송 가능

WebSocket 메시지는 텍스트(JSON 문자열 등)만 다루는 게 아니다.

- 텍스트: `String`
- 바이너리: `Blob`, `ArrayBuffer` (브라우저 기준)

이미지/음성/파일 조각 전송, 혹은 커스텀 바이너리 프로토콜 설계 시 이 점이 중요하다.

---

## 동작 방식: “HTTP로 시작해서 WebSocket으로 전환”

WebSocket 연결은 보통 다음 순서로 진행된다.

1. 클라이언트가 HTTP 요청을 보낸다 (핸드셰이크)
2. 서버가 “이 연결을 WebSocket으로 업그레이드하자”고 응답한다 (`101 Switching Protocols`)
3. 이후부터는 HTTP가 아니라 WebSocket 프레임으로 통신한다

즉, **시작만 HTTP고, 그 다음부터는 WebSocket**이다.

아래처럼 도식으로 보면 이해가 훨씬 빠르다.

![HTTP vs WebSocket 비교](/assets/img/posts/websocket-overview/http-vs-websocket-modern.png)
_HTTP의 요청-응답 모델과 WebSocket의 지속 연결/양방향 모델 비교_

![WebSocket 핸드셰이크 시퀀스](/assets/img/posts/websocket-overview/websocket-handshake-sequence-modern.png)
_시작은 HTTP 업그레이드, 이후에는 WebSocket 프레임으로 양방향 통신_

---

## WebSocket만으로는 부족할 때: 서브 프로토콜(STOMP)

WebSocket은 "연결과 프레임 전송"을 제공하지만,
"메시지의 의미/라우팅/구독 규칙"까지 표준화하진 않는다.

그래서 실무에서는 STOMP 같은 서브 프로토콜을 올려서 쓰는 경우도 있다.

- 비유: **WebSocket은 도로**, **STOMP는 그 도로 위를 달리는 차의 규격**
- Spring에서도 `spring-websocket + STOMP` 조합이 매우 흔하다
- WebSocket은 "전송 채널"일 뿐이고, STOMP는 그 위에서 "메시지 규칙"을 정의한다.

---

## ws:// 와 wss:// 차이

- `ws://` : 평문(WebSocket over TCP)
- `wss://` : TLS 적용(WebSocket over TLS) — **실서비스에서는 대부분 wss 사용**

웹소켓도 결국 네트워크 상에서 데이터가 오가기 때문에,
인증/세션/개인정보가 오간다면 `wss://`가 기본이라고 생각하는 편이 안전하다.

---

## 연결 생명주기: open → message → close

WebSocket 통신은 크게 네 가지 이벤트 흐름이 있다.

- **open**: 연결 성립
- **message**: 메시지 수신
- **error**: 에러 발생 (구현체/환경마다 의미가 조금 다를 수 있음)
- **close**: 연결 종료 (Close Code / Reason 포함 가능)

또한 WebSocket 표준에는 **ping/pong**이 있어,
연결이 살아있는지 확인(keep-alive)하거나, 중간 장비(LB/프록시) 타임아웃을 피하는 데 활용한다.

---

## 언제 WebSocket을 쓰면 좋을까

실시간 “쌍방향” 통신이 본질인 경우에 특히 적합하다.

- 채팅/메신저
- 실시간 알림(서버 push)
- 온라인 게임/멀티플레이 동기화
- 협업 편집(문서 공동 편집 등)
- 실시간 시세/대시보드 스트리밍
- IoT 기기 상태 스트리밍, 제어 채널

---

## 언제는 WebSocket이 과할까

WebSocket은 강력하지만, 만능은 아니다.

- 단순 조회/CRUD처럼 **요청-응답이 자연스러운 API**는 HTTP가 더 적합
- 서버 → 클라이언트 단방향 “푸시”만 필요하다면 **SSE(Server-Sent Events)**가 더 단순할 수 있음
- 캐시/프록시/관측(Observability) 등 HTTP 생태계의 장점을 그대로 쓰고 싶으면 HTTP가 유리

> 결론: “실시간”이라는 이유만으로 무조건 WebSocket을 선택하기보다  
> “양방향/연결 유지가 정말 필요한가?”를 먼저 따지는 게 좋다.

---

## 구현할 때 자주 부딪히는 포인트

### 1) 인증/인가(보안)

WebSocket은 연결 이후에 HTTP 헤더를 계속 주고받지 않는다.  
그래서 보통은 **핸드셰이크 단계에서 인증**을 하거나, 연결 후 첫 메시지로 토큰을 교환하는 식으로 설계한다.

- 쿠키 기반 인증을 쓰면 쿠키가 자동으로 포함될 수 있어 **CSWSH(Cross-Site WebSocket Hijacking)** 이슈를 고려해야 한다.
- WebSocket은 기본적으로 브라우저의 Same-Origin Policy에 의해 자동 차단되지 않기 때문에,
서버 측에서 `Origin` 헤더 검증을 직접 수행하는 것이 안전하다.
- 토큰(JWT 등)을 쿼리스트링에 넣는 방식은 로그/히스토리 노출 위험이 있어 주의  
  (가능하면 헤더/서브프로토콜/첫 메시지 교환 등 대안을 검토)

### 2) 확장성(Scale-out)

서버 인스턴스가 여러 대라면,
- 같은 사용자의 연결이 어디로 붙는지(Sticky Session)
- 메시지를 다른 서버로 라우팅해야 하는지(브로커/Redis PubSub 등)

같은 운영 관점 문제가 나온다.

### 3) Backpressure(느린 클라이언트)

클라이언트가 느리면 서버의 send 버퍼가 쌓이고, 결국 메모리/지연 문제가 생길 수 있다.

- 버퍼 제한
- 메시지 드롭 전략
- 전송 큐/단일 writer
- 느린 소비자 감지 후 연결 정리

같은 설계가 중요해진다.

### 4) “동시 send” 같은 동시성 문제

한 세션에 대해 여러 스레드가 동시에 write하면,
컨테이너 구현에 따라 예외(`TEXT_PARTIAL_WRITING` 등)가 발생할 수 있다.

- 세션별 write 직렬화(락/큐)
- Spring의 `ConcurrentWebSocketSessionDecorator` 같은 도구 활용

---

## 정리

WebSocket은 “실시간”을 위해 만들어진 **양방향, 지속 연결 기반 프로토콜**이다.  
하지만 실서비스에서는 단순히 연결만 열면 끝이 아니라,

- 인증/인가
- 스케일링
- 느린 소비자(backpressure)
- 동시성(send 직렬화)
- 연결 유지(ping/pong, 타임아웃)

같은 운영/설계 요소까지 함께 고려해야 안정적으로 쓸 수 있다.

---
