---
title: "알림 시스템에서 배운 트랜잭션 현실: 선점 + 멱등성"
date: 2026-02-26 21:00:00 +0900
categories: [Backend, Architecture]
tags: [outbox, kafka, websocket, idempotency, retry, compensation]
toc: true
---

## 한 줄 요약

외부 시스템이 끼는 순간 완전한 트랜잭션은 없다.  
**핵심만 강하게 보장하고, 나머지는 복구 가능한 구조로 설계하는 것**이 현실적인 해법이다.

---

## 내가 겪은 문제 2가지

Outbox → Kafka → WebSocket 구조를 실제로 운영해보니, 장애 타이밍은 생각보다 훨씬 다양했다.

### 1) 멀티 인스턴스 환경에서 중복 발행

Outbox 테이블에서 PENDING 이벤트를 조회하는 구조는  
서버가 여러 대일 경우 **동시에 같은 이벤트를 가져갈 수 있다.**

예를 들어:

- 서버 A가 PENDING 100건 조회
- 서버 B도 거의 동시에 PENDING 100건 조회
- 같은 이벤트를 각각 Kafka에 발행

결과는 **중복 발행**이다.

---

### 2) 시스템 경계 사이에서의 크래시 문제

아래는 실제로 사용하던 릴레이 코드다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OutboxRelayWorker {

    private final OutboxEventRepository outboxEventRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelayString = "${app.outbox.relay.fixed-delay-ms:2000}")
    public void relayPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxEventRepository
                .findTop100ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);

        if (pendingEvents.isEmpty()) {
            return;
        }

        for (OutboxEvent event : pendingEvents) {
            relayOne(event);
        }
    }

    private void relayOne(OutboxEvent event) {
        String messageKey = String.valueOf(event.getTargetUserId());
        try {
            kafkaTemplate.send(event.getEventType(), messageKey, event.getPayloadJson())
                    .get(5, TimeUnit.SECONDS);

            event.markPublished();
            outboxEventRepository.save(event);

            log.info("아웃박스 이벤트를 발행했습니다. outboxEventId={} eventType={}", 
                     event.getId(), event.getEventType());
        } catch (Exception ex) {
            event.markFailed();
            outboxEventRepository.save(event);

            log.warn("아웃박스 이벤트 발행에 실패했습니다. outboxEventId={} retryCount={}", 
                     event.getId(), event.getRetryCount());
        }
    }
}
```

여기에는 본질적인 경계가 있다.

- `kafkaTemplate.send()`는 Kafka 브로커와의 통신 경계에 있다.
- `event.markPublished()`는 DB 트랜잭션 경계에 있다.

이 둘은 **같은 원자적 트랜잭션 안에 묶여 있지 않다.**

### 1) 현재 코드 순서 (send → DB 저장)에서의 문제

만약 다음 상황이 발생하면?

- Kafka 발행 성공
- DB 업데이트 직전에 서버 크래시

→ DB에는 여전히 `PENDING`  
→ 워커가 다시 집어서 재발행  
→ **중복 발생**

즉, 현재 순서는 유실보다는 **중복 가능성** 쪽으로 기울어져 있다.

---

### 2) 순서를 바꿨을 경우 (DB 저장 → send)의 문제

만약 설계를 이렇게 바꾼다면:

- DB를 먼저 `PUBLISHED`로 변경
- 그 다음 Kafka 발행 시도

그리고 이 순간:

- DB 업데이트 성공
- Kafka 발행 실패 또는 서버 크래시

→ DB에는 이미 `PUBLISHED`  
→ 재시도 대상에서 제외  
→ **유실 발생**

---

결국 문제의 본질은 단순한 예외 처리 여부가 아니라,

**DB 상태 변경과 Kafka 발행이 하나의 원자적 단위로 묶일 수 없다는 것**이다.

분산 시스템 경계를 넘는 순간,  
완벽한 동시 성공을 보장하는 것은 현실적으로 불가능하다.

---

## 그래서 정한 원칙 3가지

### 1) 절대 지켜야 할 것만 강하게 보장

강한 보장은 내가 통제 가능한 경계 안에 둔다. (DB 트랜잭션)

- 도메인 상태 전이
- 유니크 제약 조건
- Outbox 기록의 원자성

핵심 불변식만 DB 안에서 강하게 보장한다.

---

### 2) 나머지는 멱등성 / 재시도 / 보상으로 설계

외부 시스템 연동은 **실패가 기본값**이다.

- 멱등성: `event_id` 기반 dedup
- 재시도: 백오프 + 최대 횟수 + 실패 추적
- 보상: 끝까지 실패하면 취소/재처리 전략

목표는 “실패하지 않게”가 아니라  
**“실패해도 결국 수렴하게” 만드는 것**이다.

---

### 3) 유실 vs 중복 — trade-off 인정

유실과 중복을 동시에 완벽히 제거하려는 시도는  
시스템을 과도하게 복잡하게 만든다.

친구 요청 알림 도메인에서는 다음을 선택했다:

- 알림 유실은 절대 안 된다.
- 내부적으로 at-least-once를 허용한다.
- 대신 소비자에서 멱등 처리로 사용자 체감 중복을 제거한다.

---

## 내가 선택한 답: 선점 + 멱등성

### 1) Relay 선점 (중복 발행 차단)

OutboxStatus에 `PROCESSING` 상태를 추가했다.

핵심은 **원자적 선점(update)** 이다.

```sql
update outbox_event
set status = 'PROCESSING'
where id = ?
  and status = 'PENDING';
```

- affected row = 1 → 선점 성공
- affected row = 0 → 이미 다른 인스턴스가 처리 중

이 방식으로 멀티 인스턴스 동시 접근을 원천 차단했다.

---

### 2) Consumer Dedup (최종 사용자 보호)

모든 이벤트에 UUID 기반 `event_id`를 부여했다.

Consumer는 처리 전에:

```sql
insert into processed_events (event_id)
values (?)
```

여기서 `event_id`는 UNIQUE 제약 조건을 가진다.

- insert 성공 → 최초 처리
- duplicate key 예외 → 이미 처리된 이벤트 → 무시

이로써 **최종 사용자 기준 exactly-once 체감**을 만든다.

---

## 흐름 정리

```
[Domain Tx Commit]
        ↓
[Outbox: PENDING]
        ↓ (atomic update)
[Outbox: PROCESSING]
        ↓
[Kafka Publish]
        ↓
[Consumer]
        ↓ (dedup)
[WebSocket Push]
```

---

## 마무리

외부 시스템이 끼는 순간  
완벽한 트랜잭션은 존재하지 않는다.

대신 선택해야 한다.

- 무엇을 반드시 지킬 것인가?
- 무엇을 인정하고 복구할 것인가?

핵심을 DB 안에 가두고,  
경계를 넘어가는 부분은 멱등성과 재시도로 수습하는 설계.

내가 운영에서 배운 것은 대부분  
이러한 설계였다.