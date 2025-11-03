---
title: '[Java] Garbage Collection (GC)'
excerpt: '가비지 컬렉션은 자바 메모리 관리의 핵심 요소로서 **힙 영역에 동적으로 할당된 메모리 중에서 더 이상 사용되지 않는 메모리 객체(garbage)들을 모아서 주기적으로 제거**하는 매커니즘입니다. 가비지 컬렉터가 메모리를 관리해주기 때문에 개발자가 메모리 누수 문제를 고려하지 않고 개발에만 집중할 수 있게 되었습니다.'

categories: [Java]
tags: [Java]

toc: true
toc_sticky: true

date: 2024-11-4
---

## 가비지 컬렉션이란?

가비지 컬렉션은 자바 메모리 관리의 핵심 요소로서 **힙 영역에 동적으로 할당된 메모리 중에서 더 이상 사용되지 않는 메모리 객체(garbage)들을 모아서 주기적으로 제거**하는 매커니즘입니다. 가비지 컬렉터가 메모리를 관리해주기 때문에 개발자가 메모리 누수 문제를 고려하지 않고 개발에만 집중할 수 있게 되었습니다.

<br>
<br>

## 가비지 판단 방법

가비지 컬렉션 시 더 이상 사용되지 않는 객체를 **Garbage**로 판단하기 위해서 **도달 능력 (Reachability)** 라는 개념을 적용합니다. 객체의 도달 가능한 상태라는 것은 **프로그램의 루트 집합에서 출발하여 참조를 따라갔을 때 도달할 수 있는 객체**를 의미합니다. 이때 루트 집합은 지역변수, 활성 스레드, 정적 변수 등이 포함됩니다. GC는 도달 능력에 따라 아래와 같은 두개의 상태로 객체를 구분합니다.

1. **Reachable**: 객체가 참조되고 있는 상태
2. **Unreachable**: 객체가 참조되고 있지 않은 상태

그리고 Unreachable 상태의 객체를 제거합니다.

![Garbage 판단 방법](https://github.com/user-attachments/assets/b51f91b8-cd4b-45fb-ba9a-d4c9aa4f3b1d)

<br>
<br>

## Heap 메모리 구조

JVM의 메모리 영역은 `Native Method Stack`,`PC Register`, `Stack Area`, `Heap Area`, `Method Area(with Runtime Constant Pool)`으로 나뉩니다. 그중 Heap 영역은 동적으로 레퍼런스 데이터가 저장되는 공간으로 CG의 대상이 되는 공간입니다.

Heap 영역과 GC는 다음의 두가지 가설을 전제로 설계되었습니다.

> [ **Weak Generational Hypothesis** ]
>
> 1. 대부분의 객체는 금방 접근 불가능한 상태(Unreachable)가 된다.
> 2. 오래된 객체에서 새로운 객체로의 참조는 아주 적게 존재한다.

Reachable 상태로 오래 남는 객체는 매우 적다는 전제 하에 효율적인 메모리 관리를 위해 물리적인 Heap 영역을 Young generation, Old generation 2가지 영역으로 나누게 됩니다.

<img width="727" alt="Heap 메모리 구조" src="https://github.com/user-attachments/assets/c8626757-7fbb-4e4d-81b8-42cfb365769c">

### Young Generation

Young generation 영역은 신규 객체가 할당되는 영역입니다. 대부분의 객체는 Unreachable 상태로 전환되기 때문에 해당 영역에 할당되었다가 GC에 의해 제거됩니다. Young generation 영역에 대한 가비지 컬렉션을 **Minor GC**라고 합니다.

**Eden**

- `new` 키워드로 새롭게 생성된 객체가 할당되는 영역

**Survivor0 & Survivor 1**

- Eden 영역에 Minor GC가 동작한 후 Reachable한 객체가 이동하는 영역
- Survivor0과 1 영역에 번갈아가면서 객체를 이동
  - 둘 중 한 영역은 비워두어야 함
  - **Copy & Scavenge** 방식

### Old Generation

Old generation은 Young generation에서 일정 GC 횟수 이상 살아남은 객체가 이동하는 공간입니다. GC가 동작할 때 살아남은 객체들의 `age`를 1씩 증가시키는데 기존에 설정해둔 임계값에 도달하면 옮기는 방식입니다. Old generation에 대한 가비지 컬렉션을 **Major GC**라고 합니다.

> [ **Permanent 영역** ]
>
> - Java 7까지는 Heap에 있던 Permanent 영역이 Java 8부터는 Native Method Stack에 위치
> - 클래스 로더에 의해 Load되는 Class, Method 들의 메타 정보가 저장되는 영역

<br>
<br>

## 가비지 컬렉션의 동작 과정

### Stop-The-World (STW)

메모리 최적화를 위해서 GC를 자주 돌리면 유리할 것 같지만 사실은 그렇지 않습니다. GC가 동작할 때 JVM이 애플리케이션 실행을 멈추기 때문인데 이를 **Stop-The-World**라고 합니다. GC 동작 시에는 GC를 실행하는 쓰레드를 제외한 나머지 쓰레드는 모두 작업을 멈춥니다. 어떤 GC 알고리즘을 써도 STW는 발생하게 됩니다. 그래서 GC 튜닝이란 STW 시간을 줄이는데 초점을 둡니다.

### Mark and Sweep

**Mark-Sweep**은 GC가 동작하는 아주 기초적인 내부 알고리즘으로 다양한 GC 알고리즘에서 사용됩니다.

1. **Mark (식별)**: Root Space로부터 그래프 순회를 통해 연결된 객체를 순회하며 참조되고 있는(Reachable) 객체들을 마킹
2. **Sweep (제거)**: Unreachable한 객체들을 Heap에서 제거
3. **Compact (압축)**: Sweep 이후 분산된 객체들을 Heap의 시작 주소쪽으로 정렬

참고로 Root Space는 Heap 영역을 참조하는 Stack, Native Method Stack, Method Area (with Runtime Constant Pool)를 의미합니다.

### Minor GC 동작

Minor GC는 **Young generation에서의 GC 동작**입니다. Young generation은 Old generation보다 크기가 작아서 GC가 0.5 ~ 1초 사이에 종료 됩니다. 동작 과정은 다음과 같습니다.

1. 객체가 계속 생성되어 Eden 영역이 차게 되면 Minor GC가 실행
2. Mark-Sweep을 진행하고 Reachable 객체들의 age를 1 증가 시키고 Survivor 영역으로 이동
3. 다시 Eden 영역이 차게 되면 Minor GC가 실행되는데 Survivor 영역에서도 Mark-Sweep을 진행
4. Survivor0의 Reachable한 객체들의 age를 1 증가 시키고 Survivor 1로 이동
   Minor GC가 동작할 때마다 Reachable한 객체들이 Survivor0 과 Survivor1을 번갈아가며 이동
5. age가 임계값에 도달한 객체들을 Old generation으로 이동

### Major GC 동작

Major GC는 **Old generation에서의 GC 동작**입니다. Old generation은 크기가 커서 Major GC가 Minor GC의 10배 이상의 시간이 걸립니다. Old 영역의 메모리가 허용치를 넘어서면 정해진 알고리즘이 동작하는 방식입니다.

<br>
<br>

## GC 알고리즘

### Serial GC

- 가장 단순한 방식의 GC로 CPU가 싱글코어일 때 개발
- Minor GC는 **Mark-Sweep**으로 동작하고 Major GC는 **Mark-Sweep-Compact**를 사용
- GC를 처리하는 쓰레드가 1개라서 STW 시간이 긴 알고리즘

### Parallel GC

- Serial GC와 기본 알고리즘은 동일하지만 **Minor GC를 멀티 쓰레드로 실행**
- Serial GC 보다 STW 시간이 감소
- Java 8의 디폴트 GC

### Parallel Old GC

- **Major GC까지도 멀티 쓰레드**로 수행하는 Parallel GC가 향상된 버전
- 새로운 GC 방식인 **Mark-Summary-Compact** 방식을 사용

### CMS GC

- CG 대상을 파악하는 Mark 단계가 분화되어 APP 쓰레드와 GC 쓰레드가 동시에 실행되는 알고리즘
- 장점: STW 시간이 짧음
- 단점: CPU와 메모리를 많이 사용하고 Compaction 단계가 없어서 메모리 단편화 발생
- Java 9부터 deprecated

### G1 GC

- Young generation에서 Old generation으로 이동하는 단계가 사라진 GC 방식
- **Region 이라는 개념을 도입하여 전체 Heap 영역을 체스판 같이 분할하여 공간을 할당**하는 방식
- 4GB 이상의 힙 메모리와 STW시간이 0.5초 필요한 상황에 적용 (Heap 메모리 충분할 때 사용)
- Java 9 버전의 디폴트 GC

<br>

> [ Reference ]
>
> - https://d2.naver.com/helloworld/1329
> - https://inpa.tistory.com/entry/JAVA-%E2%98%95-%EA%B0%80%EB%B9%84%EC%A7%80-%EC%BB%AC%EB%A0%89%EC%85%98GC-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%F0%9F%92%AF-%EC%B4%9D%EC%A0%95%EB%A6%AC
> - https://velog.io/@haminggu/Java-%EA%B0%80%EB%B9%84%EC%A7%80-%EC%BB%AC%EB%A0%89%EC%85%98-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC
