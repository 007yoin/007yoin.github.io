---
title: "GC 정리 / 회수 대상 판단, 시점, 방법"
date: 2026-02-24 14:00:00 +0900
categories: [Backend, Java]
tags: [java, jvm, gc, young-generation, old-generation, metaspace]
toc: true
---

## 한 줄 요약

GC는 회수 대상(무엇), 회수 시점(언제), 회수 방법(어떻게)으로 나눠 보면 전체 구조가 깔끔하게 정리된다.

---

## GC를 이해할 때 먼저 볼 3가지

- 회수 대상 판단
- 회수 시점
- 회수 방법

---

## 세대 단위 컬렉션 이론

세대 GC는 아래 가설을 바탕으로 설계된다.

- 대다수 객체는 일찍 사라진다. (약한 세대 가설)
- GC에서 여러 번 살아남은 객체는 앞으로도 생존할 가능성이 높다. (강한 세대 가설)
- 서로 다른 세대 간 참조는 같은 세대 내 참조보다 적다. (세대 간 참조 가설)

HotSpot에서 Young은 보통 `Eden:S0:S1 = 8:1:1`로 본다.
초기 GC에서 많은 객체가 빠르게 사라진다는 관찰이 이 전략의 배경이다.

---

## 영역별 정리

![JVM Heap 영역](/assets/img/posts/gc/jvm-heap-area.png)
_JVM Heap 영역_

### Young Generation (Eden, Survivor)

- Eden
  - 객체가 생성되면 우선 Eden에 배치
  - Minor GC가 발생하면 생존 객체가 Survivor로 이동
- Survivor (S0, S1)
  - Eden + From Survivor의 생존 객체를 To Survivor로 복사
  - S0/S1은 번갈아 From/To 역할 수행
  - 생존 횟수(age)가 임계치 이상이면 Old로 승격

Young은 Copy/Scavenge 계열로 빠르게 회수한다.

### Old Generation

- Young에서 오래 살아남은 객체가 이동하는 영역
- 상대적으로 생존율이 높아 회수 비용이 큼
- Mark-Sweep/Mark-Compact 계열이 주로 사용됨
- Old 압박이 높으면 긴 Stop-The-World가 생길 수 있음

### Permanent / Metaspace

- PermGen은 Java 7까지, Java 8부터는 Metaspace 사용
- Runtime Constant Pool, 필드/메서드 메타 정보, 코드 관련 정보를 관리
- Heap이 아닌 Native 메모리 사용
- 리플렉션/동적 클래스 로딩이 많으면 사용량이 증가할 수 있음

---

## 도달 가능성 분석 (Reachability Analysis)

GC는 도달 가능성 분석으로 회수 대상을 판단한다.
핵심은 "GC Root에서 도달 가능한가"이다.

GC Root가 될 수 있는 것:

- JVM Stack Frame 지역변수 테이블에서 참조하는 객체
- `synchronized`로 잠겨 있는 객체
- JNI가 참조하는 객체
- 메서드 영역의 클래스 정적 필드가 참조하는 객체
- 메서드 영역의 상수가 참조하는 객체
- JVM 내부에서 유지하는 참조

---

## 대표 GC 알고리즘

### Mark and Sweep

- 1단계: Mark (생존 객체 표시)
- 2단계: Sweep (미표시 객체 회수)

단점:

- 힙이 많이 찰수록 비효율 증가
- 파편화 문제 발생

### Mark and Copy

- 메모리 공간을 둘로 나눠 한쪽만 사용
- 생존 객체를 다른 공간으로 복사
- 기존 공간을 비우는 방식으로 회수

단점:

- 여유 공간이 필요
- 생존 객체가 많아지면 복사 비용 증가

### Mark and Compact

- Mark 후 생존 객체를 한쪽으로 압축
- 남은 공간을 연속 빈 영역으로 만듦

단점:

- 객체 이동 비용이 큼
- 이동 과정에서 Stop-The-World 영향이 커질 수 있음

---

## 클래식 GC 종류 일부

### Serial

- 단일 스레드
- GC 중 애플리케이션 중단
- 단순하고 리소스 사용량이 낮음

### ParNew

- Serial Young GC의 병렬 버전

### Parallel Scavenge

- 처리량(Throughput) 중심 설계

---

## G1 (Garbage First) GC

- 대용량 Heap 환경에서 자주 사용
- Heap을 Region(보통 1MB~32MB) 단위로 분할
- 회수 이득이 큰 Region부터 우선 수집

---

## 정리

GC를 정리할 때는 아래 흐름으로 보면 된다.

1. 회수 대상: Reachability로 판단
2. 회수 시점: Young/Old 메모리 압박과 정책으로 결정
3. 회수 방법: Mark-Sweep/Copy/Compact를 세대별로 조합

이 기준으로 보면 세대 구조와 알고리즘이 하나의 그림으로 연결된다.

---
