---
title: '[Kafka] Apache Kafka Producer'
excerpt: '프로듀서는 애플리케이션에서 발생한 이벤트를 카프카 토픽에 전달하는 주체입니다. 즉, 카프카의 쓰기 진입점이므로 성능, 안정성, 메시지 순서를 보장하는데 중요한 역할을 합니다.'

categories: [Kafka]
tags: [Kafka, Middleware, Event Broker]

toc: true
toc_sticky: true

date: 2025-09-10
---

## Kafka Producer

![Kafka Producer](https://velog.velcdn.com/images/gnlee95/post/27e70aa8-5798-45e6-832e-578b6a2a9cc4/image.png)

프로듀서는 애플리케이션에서 발생한 이벤트를 카프카 토픽에 전달하는 주체입니다. 즉, 카프카의 쓰기 진입점이므로 성능, 안정성, 메시지 순서를 보장하는데 중요한 역할을 합니다. 주요 역할은 다음과 같습니다.
- 메시지를 직렬화(serialize)하여 네트워크로 전송
- 파티셔너를 통해 어느 파티션에 저장할지 결정
- 전송 안정성(acks)과 재시도 전략을 제어

<br>

## 동작 방식

1. **Record 생성** : `ProducerRecord` 객체에 토픽/키/값 지정
2. **Serializer 적용** : 키와 값을 직렬화하여 `byte[]`로 변환
3. **Partition 선택**: **Partitioner**가 키 기반 해시, Round Robin 방식 등으로 파티션 결정
4. **배치처리** : `linger.ms`, `batch.size` 에 따라 배치 단위로 묶어서 전송
5. **네트워크 I/O** : Sender 스레드가 배치 전송, 브로커 응답 확인 (acks 설정)

<br>

## 카프카 메시지 요소

![Kafka Message](https://velog.velcdn.com/images/gnlee95/post/77c7f7c8-d383-43e2-8c01-4dec869a9842/image.png)
<div style="text-align: center;">
  <span align="center" style="color: #D3D3D3;">
    https://learn.conduktor.io/kafka/kafka-producers/
  </span>
</div>

<br>

**Key**  
Kafka 메시지에서 키는 선택 사항이며 null일 수 있습니다. 키는 문자열, 숫자 또는 임의의 객체일 수 있으며, 이진 형식으로 직렬화됩니다.

**Value**  
값은 메시지의 내용을 나타내며 null일 수도 있습니다. 값 형식은 임의적이며 이진 형식으로 직렬화됩니다.

**Compression Type**  
Kafka 메시지는 압축될 수 있습니다. 압축 유형은 메시지의 일부로 지정할 수 있습니다. 옵션은 `none`, `gzip`, `lz4`, `snappy`, `zstd` 입니다.

**Headers**  
키-값 쌍 형태의 선택적 Kafka 메시지 헤더 목록이 있을 수 있습니다. 특히 추적을 위해 메시지에 대한 메타데이터를 지정하기 위해 헤더를 추가하는 것이 일반적입니다.

**Partition + Offset**  
메시지가 Kafka 토픽으로 전송되면 파티션 번호와 오프셋 ID를 받습니다. 토픽+파티션+오프셋의 조합은 메시지를 고유하게 식별합니다.

**Timestamp**  
타임스탬프는 사용자 또는 시스템에 의해 메시지에 추가됩니다.

<br>

## Serializer

카프카는 TCP 상에서 Binary 프로토콜을 사용합니다. 데이터를 전송하거나 브로커에 저장할 때 Byte Array 타입을 사용하고 Producer, Consumer에서 각각 직렬화, 역직렬화를 해서 사용합니다. 그래서 Serializer가 Producer의 구성요소로 존재합니다. 직렬화할 데이터는 Message의 Key, Value 입니다. 

여러 타입의 데이터를 변환하기 위해 다양한 Serializer가 존재합니다.

- StringSerializer
- ShortSerializer
- IntegerSerializer
- LongSerializer
- DoubleSerializer
- BytesSerializer

만약 `JSON Schema`, `Avro`, `Protobuf`처럼 스키마 기반 직렬화를 사용한다면 별도의 Serializer가 필요합니다.
- KafkaJsonSchemaSerializer
- KafkaAvroSerializer
- KafkaProtobufSerializer

<br>

## 파티셔너

파티셔너는 프로듀서 클라이언트 내부에 위치하며 프로듀서가 메시지를 보낼 때 전송할 파티션을 결정합니다.
만약 **메시지 Key가 존재한다면 Hash를 기반**으로 파티션을 결정하게 됩니다. Hash 특성상 동일한 키를 가졌다면 동일한 파티션에 할당됩니다. 즉, 메세지 키를 사용하면 동일한 파티션에 할당해서 순차적인 처리를 보장을 할 수 있습니다. 키가 없다면 파티셔너 종류에 따라서 자동으로 파티션을 할당하게 됩니다. Kafka 2.4 버전을 기준으로 기본 파티셔너의 종류가 변경되었고 자동 파티션 할당 방식이 변경되었습니다.

- **DefaultPartitioner** (Kafka 2.4 이전)
  - Key 존재: Key 해시 기반
  - Key 부재: Round Robin

- **UniformStickyPartitioner** (Kafka 2.4 이후)
  - Key 존재: Key 해시 기반
  - Key 부재: 배치 단위 sticky 방식

<br>

두 파티셔너의 자동 파티션 할당 방식을 비교해보겠습니다.

![파티셔닝 전략](https://velog.velcdn.com/images/gnlee95/post/b4915b4d-84a0-4769-b3f8-0920a5112d5c/image.png)
<div style="text-align: center;">
  <span align="center" style="color: #D3D3D3;">
    https://www.confluent.io/blog/apache-kafka-producer-improvements-sticky-partitioner/
  </span>
</div>

<br>

**라운드 로빈 전략**은 메시지를 순차적으로 파티션에 분배하여 각 파티션에 레코드가 균등하게 나누어집니다. 그러나 이 과정에서 파티션마다 배치 사이즈를 채우는 시간이 길어집니다. 이는 네트워크 전송 효율을 낮추고 오버헤드를 증가시키는 원인입니다. 결국 처리량이 낮아지기 때문에 실시간성이 중요하다면 문제가 될 수 있습니다.

**스티키 전략**은 배치가 가득차거나(`batch.size`) 시간이 경과할 때(`linger.ms`)까지 하나의 파티션에 레코드가 할당됩니다. 이 경우 하나의 파티션부터 채워지기 때문에 라운드 로빈 전략보다 배치 사이즈를 채우는 지연 시간이 덜 소모됩니다. 결과적으로 처리량이 더 높아지기 때문에 라운드 로빈 방식보다 효율적입니다.
  
이외에도 **Custom Partitioner**를 사용해서 국가 코드별로 파티션을 나누거나, 특정 VIP 고객의 메시지는 별도의 파티션으로 보내는 등 파티셔닝을 커스터마이징할 수 있습니다.

<br>

## ACKS 옵션
`acks`는 Producer의 설정값으로, 전송한 데이터가 브로커들에 정상적으로 저장되었는지 성공 여부를 확인하는데에 사용합니다. 신뢰성과 성능 사이의 trade off와 관련된 옵션입니다. Kafka 2.8 버전까지는 `acks = 1` 옵션이 Default였지만 Kafka 3.0 이후로는 `acks = all` 옵션이 기본값입니다.

- `acks = 0`
  - 데이터 전송 이후 응답값을 받지 않고 성공 처리 (Fire and Forget)
  - 리더 파티션은 데이터가 저장되면 해당 오프셋을 리턴하는데 해당 값을 받지 않음
  - 가장 속도가 빠르지만 데이터 유실이 발생할 수 있음
  - 데이터 유실보다 성능이 중요한 **GPS, 네비게이션** 서비스에 적합

- `acks = 1`
  - 리더 파티션에만 정상적으로 적재되었는지 확인
  - 정상적으로 처리되지 않았다면 재시도할 수 있음
  - 팔로워 파티션에 동기화되기 전에 장애가 발생해서 데이터가 유실될 수 있음
  - `acks = 0`옵션에 비해 성능 지연이 있음
  - **At most once** 전송을 보장

- `acks = -1` (all)
  - 전송된 데이터가 리더와 파티션에 모두 정상적으로 적재되었는지 확인
  - 적어도 하나의 브로커만 살아있다면 메시지가 손실되지 않음을 보장
  - 성능 지연이 가장 심함
    - 비동기 프로듀서를 사용하여 성능 지연을 최소화할 수 있음
    - `max.in.flight.requests.per.connection` 옵션으로 요청 개수 설정
  - **At least once** 전송을 보장
    - `enable.idempotence = true` 옵션이라면 **Exactly once** 보장

<br>

만약 확인하고 싶은 팔로워 파티션의 개수를 지정하고 싶다면 `min.insync.replicas` 옵션을 사용할 수 있습니다. 해당 옵션은 프로듀서가 리더 파티션과 팔로워 파티션에 데이터가 적재되었는지 확인하기 위한 ISR 그룹의 파티션 개수를 의미합니다. 따라서 `acks = all`을 사용하려면 옵션값을 2 이상으로 설정해야 합니다.(리더 1개, 팔로워 1개)

`acks = all` 옵션은 성능 지연이 가장 심하기 때문에 이를 최소화하기 위해서는 비동기 프로듀서를 사용하는 것이 좋습니다. `max.in.flight.requests.per.connection` 옵션으로 동시 요청 개수를 설정할 수 있습니다.

> **ISR**(In-Sync-Replicas)  
> - 리더 파티션과 팔로워 파티션이 모두 싱크 된 상태. 
> - 리더 파티션에 존재하는 오프셋이 팔로워 파티션에 존재하는 오프셋과 동일한 상태면 동기화가 완료된 ISR.

<br>

## 재전송과 멱등성

카프카는 리트라이 및 멱등성 옵션을 제공하여 **Exactly Once** 전송을 가능하게 합니다.

### 전송과 재전송 매커니즘

![send mechanism](https://velog.velcdn.com/images/gnlee95/post/9dc35edb-0fd3-4fa0-b7ac-0b5bd4f38641/image.png)
<div style="text-align: center;">
  <span align="center" style="color: #D3D3D3;">
    https://backtony.tistory.com/76
  </span>
</div>

<br>

재전송과 관련된 핵심 사항은 `retries` 옵션을 통해 재시도 횟수를 지정할 수 있고 `delivery.timeout.ms` 옵션값을 `linger.ms` + `request.timeout.ms` 이상으로 설정해야 하는 것입니다. 관련된 옵션 및 동작 흐름은 다음과 같습니다.  

<br>

**주요 옵션**

|옵션|역할|주의할 점|
|:---:|:---|:---|
|`max.block.ms`|`send()`호출 시 내부 버퍼가 가득 차거나 metadata가 준비 안 된 경우, 이 시간만큼 블록해서 기다림. 그 이후에도 여유 없으면 TimeoutException 발생|너무 작게 설정하면 버퍼 부족 시 자주 예외 발생. 너무 크게 하면 send 호출이 오래 블록되어 스레드 응답성이 안 좋아짐|
|`linger.ms`|배치를 모으는 최대 대기 시간. 메시지 수 적을 때도 약간 기다려서 여러 메시지 묶어 보내면 효율 좋음.|대기 시간이 길면 지연(latency)에 영향. 실시간성 필요한 메시지는 낮은 값.|
|`request.timeout.ms`|브로커에게 요청(request)을 보낸 후 응답(ack 등)을 기다리는 최대 시간. 이 시간이 지나면 요청 실패로 간주하고 재시도 가능 또는 예외 발생.|너무 작게 하면, 네트워크 지연이나 브로커 리더 교체(leader election) 시 예외 많이 발생. 너무 크게 하면 실패 감지 느려짐.|
|`delivery.timeout.ms`|메시지 `send()` 호출 후 응답 완료 또는 실패를 최종 판단할 총 시간 상한. 재시도 + 응답 대기 + 배치 대기 시간 등을 모두 포함.|설정값이 `linger.ms` + `request.timeout.ms` 이상이어야 한다는 권고 있음. 설정이 잘못되면 불필요하게 예외가 빨리 날 수 있음.|
|`retries`|브로커 오류 발생 시 재시도 시도 횟수. 중복 가능성 및 순서 영향을 주는 옵션과 함께 생각해야 함.|재시도 많으면 latency 길어지고, 메모리나 네트워크 부하 증가.|
|`retry.backoff.ms`|실패 후 재시도 전에 대기하는 시간. Exponential backoff 또는 최대치 제한 가능. 브로커 장애나 리더 전환 시 retry 빈도를 조절.|너무 짧으면 오류이후 retry 폭주. 너무 길면 복구가 느림.|
|`max.in.flight.requests.per.connection`|연결(connection)당 요청을 얼마나 동시에 보낼지 수.|이 값이 많으면 재시도 시 응답 지연이나 순서 섞임 가능성 있음. 멱등성과 순서 보장이 필요하면 작게 설정.|

<br>

**동작 흐름**

1. 애플리케이션이 producer.send()를 호출  
	• 내부적으로 metadata (파티션 리더 위치 등) 준비 여부 체크  
	•	내부 버퍼(buffer, Record Accumulator)에 레코드 삽입  
	•	만약 버퍼가 가득 차 있거나 metadata 준비가 안 된 상태라면 `max.block.ms` 동안 블록. 이 시간이 지나면 예외 발생  

2. Record Accumulator와 배치 생성  
	•	`linger.ms` 동안 추가 메시지 수집 혹은 `batch.size` 용량 채워질 때까지 대기  
	•	배치가 준비되면 Sender Thread가 브로커 리더에게 요청 전송  

3. 요청 전송 후 응답 대기  
	•	브로커로 보낸 후 request.timeout.ms 만큼 기다림  
	•	응답이 없으면 retransmission 고려하거나, 오류 처리  

4. 실패 시 재시도 로직  
	•	옵션 `retries` 값이 0보다 크면 재시도 가능  
	•	재시도 사이에는 `retry.backoff.ms` 또는 지수 백오프 대기  
	•	재시도 반복하되 전체 동작 시간은 `delivery.timeout.ms`가 한계  

5. 최종 실패 또는 성공  
	•	성공 시, 콜백에서 RecordMetadata 반환  
	•	실패 시 TimeoutException 또는 관련 예외 던짐  
	•	멱등성(enabled) 설정 시, 중복된 sequence 감지. 브로커는 다 저장하지 않고 ack 반환  

<br>

### 멱등성

브로커에서 메시지를 잘 전달 받고 처리했으나 ack 응답이 네트워크 장애 등으로 유실되는 경우가 있습니다. 이 경우에 프로듀서는 메시지를 재전송할 것이고 이는 메시지를 중복 처리하는 위험이 있습니다. 그래서 멱등성, Exactly once 를 보장해야하는 경우에는 멱등성 옵션이 있습니다.

![idempotent producer](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2FzBLoV%2FbtsEkekyRjD%2FAAAAAAAAAAAAAAAAAAAAAI9YB_fVbQTrXfIKeXBLlbj1sZbNH0DICxDTnnwCerod%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1759244399%26allow_ip%3D%26allow_referer%3D%26signature%3DVqWIOGQ7o8YxZa3t7za65U2%252FB20%253D)
<div style="text-align: center;">
  <span align="center" style="color: #D3D3D3;">
    https://medium.com/@shesh.soft/kafka-idempotent-producer-and-consumer-25c52402ceb9
  </span>
</div>

<br>

`enable.idempotence` 옵션을 true로 활성화하면 **각 프로듀서에는 PID**가 할당됩니다. 그리고 프로듀서의 **각 메시지마다 Sequence가 1씩 증가**하며 할당됩니다. 브로커는 메시지 전송 시 PID 별로 가장 큰 Sequence를 확인해서 전송된 메시지와 차이가 1 이상이면 저장하지 않고 ack 응답만 보냅니다.

Kafka 3.0 부터는 **Exactly once**를 위한 옵션들이 Default값으로 설정되어 있습니다. 멱등성 보장을 위한 옵션값 설정은 아래와 같습니다.

- `acks`: -1(all)
- `enable.idempotence`: true
- `max.inflight.requests.per.connection`: 1 ~ 5 사이 권장
- `retries`: 0 이상

위의 값에 `min.insync.replicas`을 2로 설정해주어서 `acks=all`에 따른 성능 저하를 최소화해주는게 좋습니다.

<br>

## 트랜잭션 프로듀서

카프카는 트랜잭션 프로듀서를 지원해서 여러 토픽이나 파티션에 걸친 메시지 전송을 하나의 원자적 단위로 처리할 수 있도록 지원합니다. 예를 들어서 결제 이벤트 발생 시 Payment 토픽과 Order 토픽에 모두 메시지가 커밋되는 것을 보장해야할 때 트랜잭션 프로듀서가 필요합니다.  

트랜잭션 프로듀서는 단순히 애플리케이션이 전송한 데이터를 파티션에 기록하는 것에 그치지 않고, 트랜잭션의 경계를 알리기 위해 별도의 트랜잭션 레코드를 함께 저장합니다. 컨슈머는 파티션에 기록된 이 트랜잭션 레코드를 확인하여 해당 트랜잭션이 정상적으로 커밋(commit)되었는지를 판단합니다. 만약 데이터 레코드는 존재하지만 트랜잭션 레코드가 뒤따르지 않는다면, 아직 트랜잭션이 완료되지 않은 상태로 간주하여 컨슈머는 데이터를 소비하지 않습니다.

트랜잭션 레코드 자체에는 실제 비즈니스 데이터가 포함되어 있지 않고, 단순히 트랜잭션이 종료되었음을 표시하는 메타 정보만 담고 있습니다. 다만, 일반 레코드와 동일한 특성을 가지기 때문에 파티션 내에서 하나의 오프셋을 차지하게 됩니다.

트랜잭션 프로듀서를 사용하기 위해서는 트랜잭션을 식별할 수 있는 `transactional.id`를 고유한 값으로 설정해야 합니다.

<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-7724800898778124" crossorigin="anonymous"></script>
<!-- 수평-반응형-광고 -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="ca-pub-7724800898778124"
     data-ad-slot="2434836777"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>
  (adsbygoogle = window.adsbygoogle || []).push({});
</script>

<br>

> [ Reference ]  
> [Conductor kafkademy - producer](https://learn.conduktor.io/kafka/kafka-producers/)  
> [Confluent - Apache Kafka Producer Improvements with the Sticky Partitioner](https://www.confluent.io/blog/apache-kafka-producer-improvements-sticky-partitioner/)  
> [Kafka protocol guide](https://kafka.apache.org/protocol)  
> [Producer partitioner 변천사](https://devidea.tistory.com/123)  
> [Producer 생성 및 설정값](https://assu10.github.io/dev/2024/06/16/kafka-producer-1/)  
> [프로듀서의 내부 구조와 최적화 전략](https://jinseong-dev.tistory.com/46)
> [올리브영 Tech 블로그](https://oliveyoung.tech/2024-10-16/oliveyoung-scm-oms-kafka/)
> [acks=all 옵션](https://blog.voidmainvoid.net/507)
> [Backtony Dev - Kafka Producer](https://backtony.tistory.com/76)  
> [프로듀서 멱등성 보장하기](https://dkswnkk.tistory.com/741)