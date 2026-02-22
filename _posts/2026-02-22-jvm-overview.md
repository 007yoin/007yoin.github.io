---
title: "JVM이란 무엇인가 / JVM 구성요소"
date: 2026-02-22 21:00:00 +0900
categories: [Backend, Java]
tags: [java, jvm, classloader, runtime-data-area, execution-engine]
toc: true
---

## 한 줄 요약

JVM은 OS 위에서 하나의 프로세스로 동작하며, 자바 바이트코드를 해석(Interpret)하거나 JIT 컴파일을 통해 네이티브 기계어로 변환하여 CPU에서 실행되도록 하는 가상 머신이다.

---

## JVM은 무엇인가

JVM(Java Virtual Machine)은 OS 위에서 하나의 process로 동작하는, 자바 바이트코드를 실행하기 위한 가상 머신입니다.  
보통 컴퓨터 공학에서 말하는 Machine은 CPU, 메모리, 명령어 집합을 포함한 컴퓨터 전체 시스템을 의미하며,  
JVM에서는 자바 바이트코드를 실행할 수 있도록 구현된 가상의 실행 환경 전체를 의미합니다.  
JVM은 바이트코드를 인터프리터로 실행하거나, JIT 컴파일을 통해 네이티브 기계어로 변환하여 CPU에 전달되어 실행됩니다.
또한 클래스 로딩, 메모리 관리, GC 등 자바 런타임 환경을 함께 제공합니다.

---

## 왜 JVM을 쓰는가

OS 에 Process 형태로 떠있기때문에, 바이트코드로 컴파일이 가능하도록 본인이 자바 코드만 잘 작성하면 OS가 뭐가 됐든 대부분 잘 작동하는 장점이 있습니다.

---

## JVM 구성 요소

JVM 구성 요소로는  
크게 보면  
Class loader, Runtime data area, Execution engine 정도가 있습니다.

이 글에서는 큰 그림만 먼저 보고, 각각은 다음 글에서 따로 정리해보려고 합니다.

![JVM 구성요소](/assets/img/posts/jvm/jvm-components.png)
_JVM 구성요소_

### 1) Class Loader

- `.class` 바이트코드를 JVM 메모리로 로딩
- 필요한 시점에 클래스를 찾아서 읽어옴
- 로딩된 클래스 정보가 런타임에서 사용할 수 있는 형태가 됨

### 2) Runtime Data Area

- JVM이 실행되는 동안 사용하는 Java의 메모리 영역
- Metaspace(Method area), Heap area, Stack area, PC register, Native method stack 이 있음
- Thread 별로 쓰는 영역과 모든 Thread가 공유하는 영역이 존재
- 객체, 클래스 메타데이터, 스택 프레임 같은 실행 데이터가 저장됨

### 3) Execution Engine

- 로딩된 바이트코드를 실제로 실행하는 엔진
- 인터프리터 방식으로 실행하거나, 자주 실행되는 코드를 JIT 컴파일
- 컴파일된 코드는 네이티브 코드로 CPU에서 실행됨
- 그 유명한 Garbage Collector 도 여기에 포함하여 설명하기도 하고, 따로 구분하기도 함

---

## 정리

JVM은 단순히 "자바 실행기"가 아니라,  
바이트코드를 실행하기 위한 가상 머신 + 클래스 로딩 + 메모리 관리 + GC를 포함한 런타임 환경입니다.

그리고 JVM을 이해할 때는  
Class Loader, Runtime Data Area, Execution Engine 이 세 가지 축으로 나눠서 보면 훨씬 이해가 쉽습니다.

다음 글에서는 각 구성 요소를 하나씩 따로 정리해보겠습니다.

---
