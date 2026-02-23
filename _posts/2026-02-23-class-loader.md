---
title: "Class Loader / Java 클래스 로딩 과정"
date: 2026-02-23 12:00:00 +0900
categories: [Backend, Java]
tags: [java, jvm, classloader, linking, initialization]
toc: true
---

## 한 줄 요약

Class Loader는 `.class` 바이트 스트림을 JVM으로 가져와 런타임에 연결하고(링킹), 초기화 후 실제 실행까지 이어지게 만드는 시작점이다.

---

## Class Loader란

Class Loader는 이름을 알고 있는 특정 클래스의 정의(`byte stream`)를 읽어, JVM의 Method Area(메타데이터 영역) 에 클래스 메타데이터를 정의합니다.

JVM 관점에서 클래스 로딩은 컴파일 타임이 아니라 런타임에 일어나며, 이 특성 덕분에 유연성과 확장성을 얻습니다.

- 인터페이스만 맞으면 런타임에 구현 클래스를 바꿔 끼울 수 있음
- 필요하면 네트워크 등 외부 소스에서 클래스 코드를 가져올 수도 있음
- 해석(Resolution) 시점을 늦춰 동적 바인딩(Late Binding)을 지원할 수 있음

---

## 클래스 로더 3계층

### 1) 부트스트랩 클래스 로더(Bootstrap Class Loader)

- 가장 저수준 로더
- JVM 실행에 반드시 필요한 최소 코어 라이브러리를 로드
- 예: `java.lang.*` 같은 핵심 클래스

### 2) 플랫폼 클래스 로더(Platform Class Loader)

- 최소 코어 위에서 동작하는 자바 플랫폼 기능 클래스를 로드
- 예: 표준 API 중 확장/플랫폼 영역

### 3) 애플리케이션 클래스 로더(Application Class Loader)

- 우리가 작성한 코드와 외부 의존성을 로드
- 예:
  - 내 프로젝트 코드 `com.myapp.*`
  - Maven/Gradle 의존성 라이브러리

---

## Java 클래스 로딩 전체 흐름

![Java 클래스 로딩 흐름](/assets/img/posts/java-class-loading/java-class-loading-1.png)
_Loading → Linking(Verification, Preparation, Resolution) → Using(Initialization, Using) → Unloading_

클래스 라이프사이클은 크게 아래 순서로 볼 수 있습니다.

1. `Loading`
2. `Linking` (`Verification` → `Preparation` → `Resolution`)
3. `Using` (`Initialization` → 실제 사용)
4. `Unloading`

---

## Linking 단계 자세히

### 1) Verification (검증)

JVM 명세가 정한 제약을 만족하는지 확인합니다.

- `.class` 파일 형식이 올바른지
- 메타데이터가 유효한지
- 바이트코드가 안전한지
- 심벌 참조가 유효한지

### 2) Preparation (준비)

- `java.lang.Class` 메타데이터가 JVM 메모리에 준비됨
- 클래스 변수(정적 멤버) 메모리를 기본값(0, `null`, `false`)으로 초기화
- 일반 `static` 필드는 이 단계에서 "기본값"만 들어감
  - 예: `static int x = 10;` 이어도 `Preparation` 시점에는 `x = 0`
  - 실제 `x = 10` 대입은 다음 단계인 `Initialization`에서 `clinit` 실행으로 수행됨
- 그 다음 `static final`을 보면, 경우에 따라 동작이 갈림
  - 컴파일타임 상수: `static final int A = 10;`
    - 리터럴처럼 컴파일 시점에 값이 확정되면 상수로 취급되어 초기값처럼 바로 사용될 수 있음
  - 런타임 계산값: `static final int B = Math.abs(-10);`
    - 메서드 호출 결과처럼 실행이 필요하면 `Preparation`이 아니라 `Initialization` 시점에 값이 결정됨

### 3) Resolution (해석)

- 먼저 용어를 짚으면:
  - 상수 풀(Constant Pool): 클래스 파일 안에 있는 "문자열/타입/메서드 이름" 같은 테이블
  - 심벌 참조(Symbolic Reference): 실제 주소가 아니라 `"java/lang/String"`, `"println:(Ljava/lang/String;)V"` 같은 이름 기반 참조
- `Resolution`은 이 심벌 참조를 JVM이 바로 사용할 수 있는 실제 참조(Direct Reference)로 바꾸는 과정
  - 예: "어떤 클래스의 어떤 메서드"라는 문자열 정보 -> 실제 로드된 클래스 메타데이터/메서드 엔트리 포인터
- JVM 구현/전략에 따라 초기화 이후로 지연될 수 있음 (지연 해석)

---

## Initialization과 객체 생성

`Initialization` 단계에서 클래스 초기화 코드(`clinit`)가 실행되고, 정적 필드의 명시적 초기값 할당이 완료됩니다.

그 다음 실제 객체를 만들 때는 보통 다음 순서를 거칩니다.

1. 힙에서 객체 메모리 할당
2. 객체 헤더를 제외한 영역을 기본값으로 초기화
3. 객체 헤더 설정(클래스 메타 정보 포인터, 해시코드 관련 정보, GC age 등)
4. 생성자 실행으로 필드 초기화

---

## 정리

Class Loader는 단순히 클래스 파일을 "읽어오는 기능"이 아니라, JVM이 안전하게 클래스를 검증하고 연결하고 초기화해 실행 가능한 상태로 만드는 런타임 진입점입니다.

특히 `Linking`(검증/준비/해석)과 `Initialization`의 구분을 잡아두면, 클래스 초기화 타이밍 이슈나 정적 필드 동작을 이해할 때 훨씬 수월해집니다.

---
