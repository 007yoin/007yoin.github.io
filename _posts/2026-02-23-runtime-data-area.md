---
title: "Runtime Data Area / JVM 메모리 구조"
date: 2026-02-23 15:30:00 +0900
categories: [Backend, Java]
tags: [java, jvm, runtime-data-area, heap, stack, metaspace]
toc: true
---

## 한 줄 요약

Runtime Data Area는 JVM 실행 중 데이터가 실제로 놓이는 메모리 모델이며, 스레드별 영역과 모든 스레드가 공유하는 영역으로 나뉜다.

---

## Runtime Data Area란

Runtime Data Area는 JVM이 프로그램을 실행하는 동안 사용하는 메모리 영역입니다.  
클래스 로딩 결과, 메서드 실행 상태, Object 인스턴스, 상수 정보 등이 여기에 저장됩니다.

보통 아래 5가지로 구분합니다.

1. Method Area (논리적 영역, 구현상 Metaspace 포함)
2. Heap
3. Stack
4. PC Register
5. Native Method Stack

---

## 1) Method Area (메서드 영역)

Method Area는 JVM이 읽어 들인 클래스 수준 정보가 저장되는 영역입니다.

- 타입(클래스/인터페이스) 메타데이터
- 런타임 상수 풀(Runtime Constant Pool)
- 정적 변수(`static`) 정보
- 메서드/필드 관련 구조 정보
- JIT 컴파일 결과 코드 캐시(Code Cache)와 연계되는 실행 정보

핵심 포인트:

- Java 8부터 PermGen은 제거되고 Metaspace를 사용
- Metaspace는 JVM Heap이 아니라 네이티브 메모리에서 관리
- 필요에 따라 동적으로 커질 수 있음 (상한은 옵션으로 제한 가능)

---

## 2) Runtime Constant Pool

Runtime Constant Pool은 각 클래스/인터페이스별로 가지는 런타임 상수 테이블입니다.

- 클래스 파일의 constant_pool 정보를 런타임에 옮겨 둔 영역
- 리터럴(문자열, 숫자 등)과 심벌 참조(클래스/메서드/필드 이름 기반 참조)를 포함
- 클래스 로딩 과정에서 생성되어 Method Area의 일부로 관리

심벌 참조는 실제 메모리 주소가 아니라 "이름으로 가리키는 참조"입니다.
실행 중 해석(Resolution)을 거치면서 실제 참조로 연결됩니다. (Class loader 편 참고)

또한 런타임에는 일부 상수가 동적으로 추가/관리될 수 있어, 완전히 고정된 정적 테이블만은 아닙니다.

---

## 3) Stack

Java Stack은 스레드마다 하나씩 생성되는 실행 스택입니다.  
메서드 호출 시 프레임(Stack Frame)이 쌓이고, 메서드 종료 시 프레임이 제거됩니다.

프레임에는 보통 다음 정보가 들어갑니다.

- 지역변수 테이블(Local Variable Table)
- 피연산자 스택(Operand Stack)
- 동적 링크 정보
- 메서드 반환값/반환 주소 관련 정보

자주 보는 포인트:

- 지역변수 테이블의 0번 슬롯은 인스턴스 메서드에서 `this` 참조
- 슬롯 기반으로 변수 저장 (일부 타입은 2 슬롯 사용)
- 스택 한계를 넘으면 `StackOverflowError` 발생

---

## 4) Native Method Stack

Native Method Stack은 JNI 등으로 네이티브 코드(C/C++)를 실행할 때 사용하는 스택 영역입니다.

- 네이티브 함수의 지역 변수/자동 변수 처리에 사용
- JVM 구현체에 따라 Java Stack과 분리되거나 내부적으로 통합되어 동작 가능

---

## 5) Heap Area

Heap은 GC가 관리하는 메모리 영역으로, Object 인스턴스와 배열이 저장됩니다.

- `new`로 생성한 대부분의 객체가 힙에 배치
- JVM 옵션으로 크기 조절 가능 (`-Xms`, `-Xmx`)
- 메모리 부족 시 `OutOfMemoryError` 발생

HotSpot의 전형적인 세대 기반 관점:

- Young Generation: Eden, Survivor
- Old Generation: 장수 Object 위주
- Metaspace: 클래스 메타데이터(Heap 바깥 네이티브 메모리)

즉, "Object는 Heap", "클래스 메타데이터는 Metaspace"로 나눠 이해하면 정리가 쉽습니다.

---

## 스레드별 vs 공유 영역

정리하면 다음처럼 볼 수 있습니다.

- 스레드별: Stack, PC Register, Native Method Stack
- 스레드 공유: Heap, Method Area(Runtime Constant Pool 포함)

이 구분은 동시성 문제나 메모리 이슈를 분석할 때 매우 중요합니다.

---

## 정리

- 운영 관점
  - 장애가 나면 Heap 부족인지, Metaspace 증가인지, Stack 한계인지를 빠르게 구분할 수 있음
  - GC 로그와 메모리 지표를 볼 때 Young/Old 동작을 근거 있게 해석할 수 있음
- 동시성 관점
  - 스레드별 영역(Stack/PC/Native Stack)과 공유 영역(Heap/Method Area)을 분리해 race condition, 가시성, 상태 공유 문제를 분석하기 쉬움
  - 어떤 데이터를 스레드 로컬로 둘지, 어떤 데이터를 공유 Object로 둘지 설계 판단이 명확해짐
- 성능 관점
  - 객체 생성/소멸 패턴과 GC 비용을 연결해 병목 원인을 찾기 쉬움
  - 불필요한 클래스 로딩, 메타데이터 누적 같은 문제를 조기에 감지할 수 있음
- 디버깅 관점
  - `StackOverflowError`, `OutOfMemoryError`, 클래스 로딩 관련 오류를 같은 기준으로 해석해 원인 후보를 빠르게 좁힐 수 있음

결국 Runtime Data Area 이해는 실제 서비스에서 문제를 더 빨리 찾고 정확히 고치는 데 직접적으로 도움이 됩니다.

---
