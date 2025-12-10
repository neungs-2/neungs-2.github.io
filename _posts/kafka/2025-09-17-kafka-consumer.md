---
title: '[Kafka] Apache Kafka Consumer'
excerpt: '브로커의 토픽에서 이벤트 데이터를 가져와서 처리하는 애플리케이션을 카프카 컨슈머라고 합니다. 카프카의 읽기 진입점으로 오프셋 관리, 재처리 전략, 처리 안정성 등에서 중요한 역할을 합니다.'

categories: [Kafka]
tags: [Kafka, Middleware, Event Broker]

toc: true
toc_sticky: true

date: 2025-09-17
---

## 소비자의 역할

브로커의 토픽에서 이벤트 데이터를 가져와서 처리하는 애플리케이션을 카프카 컨슈머라고 합니다. 카프카의 읽기 진입점으로 오프셋 관리, 재처리 전략, 처리 안정성 등에서 중요한 역할을 합니다. 주요 역할은 다음과 같습니다.

- 파티션으로부터 메시지 읽기(poll)
- 역직렬화(deserialize) 수행
- 오프셋 관리
- 컨슈머 그룹을 통한 스케일 아웃 처리
- 브로커 heartbeat 기반 장애 감지

<br>

## 내부 구성 요소

1. Fetcher & ConsumerNetworkClient  
    2개의 컴포넌트를 통해, 파티션의 데이터를 해당 컨슈머 클라이언트로 가져옵니다.

2. ConsumerCoordinator  
   해당 Consumer Group의 리더 컨슈머가 누구인지, 해당 컨슈머의 옵션 등을 관리합니다.

3. HeartBeat Thread  
   하트비트 체크를 위한 별도의 스레드입니다.

4. SubscriptionState  
   파티션의 구독 상태를 관리합니다.

<br>

## 동작 방식

소비자는 한번에 하나 이상의 파티션에서 데이터를 읽을 수 있고 같은 파티션 내의 데이터는 순서대로 읽힙니다. 항상 낮은 오프셋에서 높은 오프셋으로 데이터를 읽고 반대로는 데이터를 읽을 수 없습니다. 소비자는 브로커에 데이터를 요청해서 당겨오는 **Pull 방식**으로 동작합니다. 덕분에 컨슈머가 토픽 소비 속도를 제어할 수 있습니다. 조금 더 구체적으로 설명하자면 아래와 같습니다.

1. 소비자가 consumer group를 명시하여 특정 topic에 대한 `subscribe()` 요청
2. 카프카 클러스터에서 소비자에 대한 `client-id` 및 topic의 특정 파티션을 지정
3. 소비자는 해당 파티션에 주기적으로 `poll()` 요청, 데이터를 배치 단위로 가져옴
4. `poll()` 주기에 배치 안의 모든 데이터를 다 처리하고 `commit()` 요청
   만약 설정된 timeout 시간 내에 `commit()` 하지 않으면, 카프카는 해당 소비자에 장애가 있다고 판단하고 consumer group에서 제외

![kafka_consumer.png](https://velog.velcdn.com/images/gnlee95/post/26b84f4c-0aa4-4bc4-b1ee-7008a9bf680e/image.png)

`poll()` 동작 시 실제 데이터를 받아오는 역할은 `ConsumerNetworkClient`가 수행하며 비동기로 bytes 형태 데이터를 받아옵니다. 이후  Fetcher에서 주기적으로 bytes 형태의 데이터를 꺼내 역직렬화 후 `completedFetches`에 삽입합니다. 

<br>

## 컨슈머 그룹

동일한 논리적 작업을 수행하는 소비자는 그룹화시킬 수 있습니다. Kafka 컨슈머 그룹은 하나의 토픽을 협력하여 처리한다고 이해할 수 있습니다. 토픽 내에는 여러 파티션이 존재하기 때문에 컨슈머 그룹 내의 소비자들이 담당하는 파티션의 이벤트를 처리합니다. 

![kafka_consumer_group.png](https://velog.velcdn.com/images/gnlee95/post/e60eedf0-5734-4fbe-923b-099092679cdb/image.png)

하나의 토픽은 여러개의 컨슈머 그룹이 구독할 수 있기 때문에 파티션에 대한 Offset을 컨슈머 그룹 단위로 관리합니다. 컨슈머와 파티션 사이의 관계는 `1 : N`으로 하나의 파티션은 하나의 소비자에게 매칭됩니다. 그리고  컨슈머 그룹 내의 소비자 개수가 토픽 내의 파티션 개수보다 많다면 유휴상태(idle)의 소비자가 발생합니다. 결국 파티션 개수와 소비자 개수를 일치시키는 것이 낭비 없이 성능이 가장 좋은 상태가 됩니다.

단일 토픽에 대한 구독을 단일 소비자가 아닌 컨슈머 그룹으로 관리하기 때문에 다음과 같은 장점이 생깁니다.
1. 토픽에 대한 처리를 병렬처리로 분산 (여러 소비자가 동작)
2. 소비자에 문제 생기는 경우 파티션을 다른 소비자에게 할당하여 장애 내성 (Fault Tolerance) 확보
3. 토픽을 여러 컨슈머 그룹이 구독 가능하기 때문에 관심사가 다른 작업에 대해서 디커플링
   - 별도의 작업끼리 장애 영향을 받지 않음
   - 작업별로 필요한 성능에 맞게 소비자 개수 조절 가능

<br>

## Consumer Message Semantics

카프카 소비자 역시 오프셋 커밋 전략을 선택할 수 있습니다. 해당 전략들은 소비자 재시작 시 메시지를 건너뛰거나 두번 읽는 경우에 영향을 미칩니다. 자동 커밋(`enable.auto.commit=true`) 옵션은 주기적 타이머로 오프셋을 커밋하지만 재처리나 메시지 유실 문제가 있습니다. 따라서 실무에서는 필요한 시나리오에 맞는 수동 커밋을 사용하게 됩니다.

### At-most-once
최대 한번 전송의 경우, `poll()` 호출 후 메시지 배치가 수신되는 즉시 오프셋이 커밋됩니다. 만약 후속 처리 실패 시 오프셋이 이미 커밋됐기 때문에 다시 읽을 수 없습니다. 따라서 재처리보다는 성능이 중요하고 손실을 허용하는 네비게이션 등의 시나리오에서 사용됩니다. At-most-once 전략은 짧은 주기의 자동 커밋을 하거나 수동 커밋으로 제어하는 두가지 방식이 있습니다.  

자동 커밋의 경우 아래와 같이 자동 커밋 설정을 해주면 됩니다.
```plain
enable.auto.commit=true
auto.commit.interval.ms=1000   # 자주 커밋해도 됨
```

수동 커밋의 경우 자동 커밋 옵션을 `enable.auto.commit=false` 로 설정하고 커밋 이후에 후처리를 하도록 코드를 작성합니다.

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    // 처리 전에 먼저 커밋
    consumer.commitSync(); // or commitAsync() + 콜백
       
    for (ConsumerRecord<String, String> record : records) {
        process(record);     // 장애나도 메시지는 이미 커밋 → 유실 가능
    }
}
```

### At-least-once

최소 한번 전송은 가장 일반적으로 쓰이는 패턴으로 이벤트의 처리 완료를 보장하는 전략입니다. 오프셋은 메시지 처리 후 커밋됩니다. 다만 재시도로 인해 중복이 발생할 수 있으므로 재시도가 시스템에 영향을 미치지 않도록 **멱등 처리**가 필요합니다. 

`isolation.level=read_uncommitted`

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }

    // 처리 후에 오프셋 커밋
    consumer.commitSync();  // or commitAsync() + 콜백
}
```

위의 코드에서 만약 처리 이후 커밋 전에 컨슈머 장애 발생 시에 기존 오프셋을 다시 읽어오고 재처리를 합니다.

### Exactly-once (EOS)

해당 전략은 금융 거래처럼 정확히 한 번만 메시지를 전송할 필요가 있을 때 사용하게 됩니다. 하지만 사실 소비자 입장에서만 생각해보면 At-least-once와 큰 차이점이 없습니다. Kafka에서 말하는 EOS는 분산 시스템 내에서 최대한 메시지를 한번 처리한 것처럼 보이게 하는 기법으로 크게 2가지 방법으로 구현할 수 있습니다.

- 외부 저장소에 Consume 기록을 저장하는 방식
  - 직접 EOS를 지원할 수 있는 패턴으로 구현
	  ![kafka-eos.png](https://velog.velcdn.com/images/gnlee95/post/0c84f3e0-ef18-47dd-99df-ca9512c593b1/image.png)

- 카프카 옵션을 사용하는 방식 (Kafka 0.11 버전 이상)
  - 멱등성 프로듀서와 트랜잭션 기반 전송을 사용
	  ![kafka_eos_2.png](https://velog.velcdn.com/images/gnlee95/post/e24a15e3-4d2c-4d79-b5ce-ec0f64dad911/image.png)

<br>

## 어사이너

컨슈머 어사이너는 소비자와 파티션 할당 정책을 결정합니다.

### RangeAssignor 
![range_assignor.png](https://velog.velcdn.com/images/gnlee95/post/3dd2ed8b-35cc-4f21-aec1-d5b6beff5632/image.png)

- 각 토픽에서 파티션을 숫자순, 소비자를 사전순으로 정렬하여 할당
- 각 소비자에 동일한 비율의 파티션을 분배 
- 나누어 떨어지지 않는 경우 앞쪽 소비자에 추가로 할당
- [ 예시 ]
	- Consumer 1: (Partition 0, Partition 1)
	- Consumer 2: (Partition 2)

### RoundRobinAssignor
![[round_robin_assignor.png]](https://velog.velcdn.com/images/gnlee95/post/1eab5235-8e0e-4b9a-b0e1-c2c5e506f297/image.png)

- 모든 파티션을 소비자에 번갈아가면서 할당
- 토픽이 여러개인 경우 특정 소비자에 파티션이 몰릴 수 있으므로 주의 
- [ 예시 ]
	- Consumer 1: (Partition 0, Partition 2)
	- Consumer 2: (Partition 1)

### StickyAssignor
- 초기에는 RoundRobin Assinor와 동일
- 컨슈머 장애 발생 시 아래 규칙을 만족하도록 할당
	- 최대한 파티션을 균등하게 배분하면서 할당
		- 소비자에 매칭된 파티션의 개수의 차이가 최대 1개 이상을 넘지 않도록 설정
	- 처음부터 재할당하는 것이 아니라 기존 할당은 최대한 유지
		- 연결이 끊어진 파티션만 균등하게 분배

### CooperativeStickyAssignor 
- Kafka ver 2.4 (stable in 2.5) 부터 적용
- Sticky Assignor와 동일하지만  `Cooperative Rebalance`를 사용하여 다운타임 최소화

<br>

## 리밸런싱

특정 컨슈머의 파티션의 소유권을 다른 컨슈머에게 이관하는 조정 작업을 의미합니다. 브로커 내부의 컴포넌트인 Group Coordinator 는 Consumer Group의 상태를 체크하여 그룹 내 변동이 발생하거나 특정 컨슈머에 장애가 발생한 경우 리밸런싱 명령을 내립니다. 

이때 가장 먼저 응답한 소비자를 Leader로 선출합니다. Leader Consumer는 파티션 할당 전략에 맞게 파티션을 소비자들에게 할당합니다.

리밸런싱 도중에 관련 컨슈머들은 **해당 시간 동안 모든 구독 활동이 중지되기 때문에 메시지를 구독할 수 없습니다.** 따라서 너무 빈번한 리밸런싱은 성능을 저하시킵니다.

**리밸런싱이 발생하는 경우**
1. 컨슈머 생성 / 삭제
2. 컨슈머 장애 발생 (heart beat 미응답)
3.  `max.poll.interval.ms` 초과 (polling ~ commit 까지의 대기시간)  

**리밸런싱 막기 위한 조치**
1. `max.poll.records` 줄이기
   - 리밸런싱 시간 단축
   - poll 시간 지연 예방으로 리밸런싱 발생 가능성 감소
   - 중복 컨슈밍 가능성 감소

2. 수동 커밋 사용

리밸런스 프로토콜은 다음의 두가지 방식이 있으며 최신 버전에서는 협력적 리밸런스 방식으로 완전히 전환되었습니다.

### Eager Rebalance
- 모든 컨슈머가 일시적으로 파티션 할당 해재됨
- 새로운 할당이 완료될 때까지 메시지 소비 중단
- 전체 파티션 재분배되므로 전체 작업중단 발생
    - kafka 4.0부터 완전히 중단됨.

### Cooperative Rebalance
- 점진적으로 파티션을 재할당 
- 필요한 파티션을 재할당하여 서비스 중단 최소화
    - kafka 3.1부터 기본값

<br>

## 주요 옵션

- `key.deserializer`: 메시지 키를 역직렬화하는 클래스 지정
- `value.deserializer`: 메시지 값을 역직렬화하는 클래스 지정
- `enable.auto.commit`: 자동/수동 커밋 지정. 기본값은 `true`
- `auto.commit.interval.ms`: 자동 커밋일 경우 오프셋 커밋 간격 지정. 기본값은 5초(5000)
- `max.poll.records`: `poll()` 메서드를 통해 반환되는 레코드 개수 지정. 기본값은 500
- `max.poll.interval.ms`: `poll()` 메서드를 호출하는 간격의 최대 시간. 기본값은 5분 (300000)
- `isolation.level`: 트랜잭션 프로듀서가 레코드를 트랜잭션 단위로 보낼 경우 사용
- `session.timeout.ms`: 컨슈머가 브로커와 연결이 끊기는 최대 시간. 기본값은 10초 (1000)
- `hearbeat.interval.ms`: 하트비트를 전송하는 시간 간격. 기본값은 3초 (3000)
- `group.id`
  - 컨슈머 그룹 아이디를 지정
  - `subscribe()` 메서드로 토픽을 구독하여 사용할 때에 해당 옵션이 필수
  - 기본값은 `null`
- `auto.offset.reset`: 파티션을 읽을 때 저장된 컨슈머 오프셋이 없는 경우 읽을 오프셋을 선택하는 옵션. 기본값은 `latest`
  - `latest`: 가장 마지막 오프셋부터
  - `earliest`: 가장 처음 오프셋부터
  - `none`: 예외 발생

<br>

## 컨슈머 랙

컨슈머 랙은 파티션의 프로듀서 오프셋과 컨슈머 오프셋 간의 차이를 의미합니다. 프로듀서가 `push`하는 속도와 컨슈머가 처리하는 속도의 차이로서 컨슈머가 정상 동작을 하는지 여부를 알 수 있기 때문에 필수적으로 모니터링이 필요한 지표입니다. 모니터링에는 주로 버로우, 그라파나, 엘라스틱서치, 텔레그래프 등이 사용됩니다.

컨슈머 랙을 통해 컨슈머의 데이터 처리량을 늘려야할지 결정할 수 있고 적절한 튜닝이 필요합니다. 파티션과 컨슈머의 개수를 늘리면 병렬처리를 통해서 데이터 처리량을 늘릴 수 있습니다. 또한 같은 토픽의 파티션 중 유독 컨슈머 랙이 큰 파티션이 있다면 할당된 컨슈머에 이슈가 발생했음을 유추할 수 있습니다.

> [ Reference ]
>  
> [Conduktor - Kafka 문서](https://docs.conduktor.io/learn/advanced/consumers)  
> [Confluent - Message Delivery Semantic](https://docs.confluent.io/kafka/design/delivery-semantics.html)  
> [카프카 컨슈머 파헤치기](https://jngsngjn.tistory.com/51)  
> [리벨런싱 종류와 파티션 할당 전략](https://junior-datalist.tistory.com/387)  
> [Kafka 노션 정리](https://third-edge-020.notion.site/Kafka-e3ab2699193c49259ecdfd46f9ea9758#7ae3c03bc6cf4c22b6eeb0b049f5f98d)  
> [Consumer 내부 동작 원리와 구현](https://devocean.sk.com/community/detail.do?ID=165478&boardType=DEVOCEAN_STUDY&page=1)  
> [카프카 어사이너 1](https://yooloo.tistory.com/95)  
> [카프카 어사이너 2](https://p-bear.tistory.com/56)
