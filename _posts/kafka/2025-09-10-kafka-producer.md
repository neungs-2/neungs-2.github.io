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
Kafka 메시지는 압축될 수 있습니다. 압축 유형은 메시지의 일부로 지정할 수 있습니다. 옵션은 none, gzip, lz4, snappy, zstd입니다.

**Headers**  
키-값 쌍 형태의 선택적 Kafka 메시지 헤더 목록이 있을 수 있습니다. 특히 추적을 위해 메시지에 대한 메타데이터를 지정하기 위해 헤더를 추가하는 것이 일반적입니다.

**Partition + Offset**  
메시지가 Kafka 토픽으로 전송되면 파티션 번호와 오프셋 ID를 받습니다. 토픽+파티션+오프셋의 조합은 메시지를 고유하게 식별합니다.

**Timestamp**  
타임스탬프는 사용자 또는 시스템에 의해 메시지에 추가됩니다.

<br>

## 파티셔너

파티셔너는 프로듀서 클라이언트 내부에 위치하며 프로듀서가 메시지를 보낼 때 전송할 파티션을 결정합니다.
만약 **메시지 Key가 존재한다면 Hash를 기반**으로 파티션을 결정하게 됩니다. Hash 특성상 동일한 키를 가졌다면 동일한 파티션에 할당됩니다. 키가 없다면 파티셔너 종류에 따라서 자동으로 파티션을 할당하게 됩니다. Kafka 2.4 버전을 기준으로 기본 파티셔너의 종류가 변경되었고 자동 파티션 할당 방식이 변경되었습니다.

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

<br>

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
> [Producer partitioner 변천사](https://devidea.tistory.com/123)  
> [Producer 생성 및 설정값](https://assu10.github.io/dev/2024/06/16/kafka-producer-1/)  
> [프로듀서의 내부 구조와 최적화 전략](https://jinseong-dev.tistory.com/46)