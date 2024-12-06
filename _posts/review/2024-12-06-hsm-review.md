---
title: '[네트워크] gRPC'
excerpt: 'HTTP/2와 프로토콜 버퍼를 사용하는, Google에서 개발한 RPC인 gRPC'

categories: [Network]
tags: [Network, API]

toc: true
toc_sticky: true
published: true

date: 2024-09-19
---

## gRPC 란?

gRPC(Google Remote Procedure Call)는 구글에서 개발한 고성능 RPC(Remote Procedure Call)로, 다양한 언어와 플랫폼을 지원하는 마이크로서비스 아키텍처에 최적화된 통신 방식을 제공합니다. gRPC는 클라이언트와 서버 간의 원격 함수 호출을 마치 로컬 함수처럼 쉽게 처리할 수 있게 하며, 복잡한 네트워크 통신을 추상화하여 개발자가 간편하게 분산 시스템을 구축할 수 있게 도와줍니다.

이 기술은 **HTTP/2** 프로토콜을 기반으로 하여, 양방향 스트리밍과 멀티플렉싱을 지원합니다. 이를 통해 클라이언트와 서버가 단일 연결을 통해 동시에 여러 메시지를 주고받을 수 있으며, 전송 효율성을 극대화할 수 있습니다. 이는 기존 REST API가 주로 사용하는 HTTP/1.1 프로토콜의 한계를 극복한 것으로, 특히 대규모 분산 시스템이나 실시간 데이터 전송에 적합합니다.

또한 gRPC는 **프로토콜 버퍼(Protocol Buffers)** 라는 바이너리 직렬화 형식을 사용하여 데이터를 전송합니다. 프로토콜 버퍼는 데이터의 크기를 줄여 전송 속도를 향상시키고, 정의된 인터페이스를 통해 엄격하게 데이터 형식을 관리할 수 있습니다. 이는 REST API에서 사용하는 JSON보다 빠르고 효율적인 데이터 교환을 가능하게 하며, 다양한 프로그래밍 언어에서 쉽게 사용할 수 있도록 합니다.

![gRPC](https://grpc.io/img/landing-2.svg)  
이미지 출처: grpc.io

<br>

## 관련 개념 정리

### API

API는 Application Programming Interface의 줄임말로 응용 프로그램에서 사용할 수 있도록, 운영 체제나 프로그래밍 언어가 제공하는 기능을 제어할 수 있게 만든 인터페이스를 뜻합니다. 인터페이스는 두 애플리케이션 간의 서비스 계약이라고 할 수 있습니다. 이 계약은 요청과 응답을 사용하여 두 애플리케이션이 서로 통신하는 방법을 정의합니다. API의 종류로는 `SOAP, RPC, Websocket, REST, GraphQL` 등이 있습니다.

### RPC

원격 프로시저 호출은 프로세스 간 통신의 형태(IPC)의 한 형태로 떨어져 있거나 분산되어 있는 컴퓨터간의 통신을 위한 기술입니다. 별도의 원격 제어를 위한 코딩 없이 다른 주소 공간에서 함수나 프로시저를 실행할 수 있습니다. 프로그래머는 함수가 실행 프로그램에 로컬 위치에 있든 원격 위치에 있든 동일한 코드를 이용할 수 있습니다.

`Caller`에서 `procedure`를 호출하면, `Request Message`를 통해 함수의 `parameter`가 전달 되고, `Callee`에서 수행한 결과를 `Reply message`를 통해 `Caller`에게 전달됩니다.

이 때, 주고 받는 함수에 대한 명세(parameter, method, .. )에 대해 알아야 하기 때문에 인터페이스를 `IDL(Interface Definition Langauge)`을 활용하여 정의하여야 합니다.

프로그램이 작성된 언어는 모두 다르기 때문에, RPC는 특정 언어에 대한 종속되어있지 않습니다.  
Client와 Server가 서로 다른 언어로 작성되어 있다면, 이들 간의 원활한 소통을 위해 `IDL`이라는 명세가 필요합니다. (`Xml`, `Json`, `Proto`)

![](https://media.geeksforgeeks.org/wp-content/uploads/operating-system-remote-call-procedure-working.png)  
이미지 출처: geeks for geeks

> **프로시저**
> 컴퓨터 프로그래밍에서 특정 작업을 수행하기 위해 일련의 명령어들을 모아놓은 것

> **Stub**
> 서버와 클라이언트는 다른 주소 공간을 사용하고 있는데, 함수에 사용하는 매개변수를 변환하는 작업이 필요.
> 특정 Machine에 종속된 매개변수를 **표준 포맷**으로 변환하는 역할

### Protocol buffer

프로토콜 버퍼란 구조화된 데이터를 직렬화하는 방식입니다. `.proto` 파일을 이용해서 직렬화하려는 데이터의 구조를 정의하는데 key-value 쌍의 필드로 이루어져 있습니다. 스키마는 아래와 같습니다. 여기에서 `1, 2, 3`은 각 데이터의 `field tag` 입니다.

```proto
message Person {
    required string user_name        = 1;
    optional int64  favourite_number = 2;
    repeated string interests        = 3;
}
```

아래의 데이터를 인코딩한 결과는 다음과 같습니다.

```json
// Data
{
  "userName": "Martin",
  "favouriteNumber": 1337,
  "interests": ["daydreaming", "hacking"]
}
```

![encoding result](https://martin.kleppmann.com/2012/12/protobuf_small.png)  
이미지 출처: martin.kleppmann.com

`userName` 같은 속성 값을 `field tag`로 대체하여 데이터를 줄이는 게 핵심입니다.
속성 값과 타입을 조합하여 1바이트 메타정보로 표현할 수 있습니다.

<br>

## gRPC vs Rest

gRPC와 REST는 API 통신을 위한 대표적인 두 방식으로, 각각의 특징과 장단점이 뚜렷합니다. 이 두 방식의 차이를 이해하면, 각각의 방식이 어떤 상황에서 더 적합한지 판단할 수 있습니다.

### 통신 프로토콜

- **REST**는 주로 **HTTP/1.1**을 사용하며, 리소스 기반의 요청-응답 구조를 따릅니다. 이 방식은 JSON 또는 XML과 같은 텍스트 기반의 형식을 사용하여 데이터 전송이 이루어지며, 모든 요청은 개별적으로 처리됩니다.
- **gRPC**는 **HTTP/2**를 기반으로 동작합니다. HTTP/2는 **멀티플렉싱**을 지원하여 하나의 연결로 동시에 여러 요청과 응답을 주고받을 수 있으며, 데이터는 더 효율적인 **프로토콜 버퍼** 형식을 사용해 바이너리로 직렬화되어 전송됩니다. 이로 인해 gRPC는 REST보다 더 빠르고 효율적인 통신이 가능합니다  .

### 데이터 직렬화 형식

- **REST**는 **JSON**을 주로 사용하며, 이는 사람이 읽기 쉽지만 크기가 크고 속도가 느립니다. JSON을 사용하면 직렬화 및 역직렬화 과정에서 더 많은 리소스를 소모하게 됩니다.
- **gRPC**는 **프로토콜 버퍼**를 사용하여 데이터를 직렬화합니다. 이는 바이너리 형식으로 JSON보다 훨씬 더 작은 크기로 데이터를 전송할 수 있으며, 성능 측면에서도 더 효율적입니다 .

### 통신 방식

- **REST**는 주로 **단방향 요청-응답** 패턴을 따르며, 서버는 요청에 따라 단일 응답만을 보냅니다.
- **gRPC**는 더 다양한 통신 방식을 지원합니다.
  - **단일 요청-응답 (Unary RPC)**: REST와 유사하게 단일 요청에 대해 단일 응답을 처리합니다.
  - **서버 스트리밍**: 서버가 클라이언트의 요청에 대해 여러 응답을 스트리밍 방식으로 보냅니다.
  - **클라이언트 스트리밍**: 클라이언트가 여러 요청을 보내고 서버가 단일 응답을 보냅니다.
  - **양방향 스트리밍**: 클라이언트와 서버가 동시에 여러 메시지를 주고받을 수 있습니다.

### 브라우저 지원

- **REST**는 모든 웹 브라우저에서 완벽하게 지원되며, 웹 애플리케이션 개발에 있어 표준으로 자리 잡고 있습니다.
- **gRPC**는 브라우저 지원이 제한적입니다. gRPC를 브라우저에서 사용하려면 추가적으로 `gRPC-web`과 같은 프록시 계층이 필요합니다. 따라서 gRPC는 주로 내부 시스템이나 백엔드 API 통신에 많이 사용됩니다 .

### 사용 사례

- **REST**는 웹 애플리케이션과 API 통신에서 범용적으로 사용되며, 브라우저 호환성과 쉬운 사용성을 이유로 여전히 많은 프로젝트에서 표준으로 사용됩니다.
- **gRPC**는 **실시간 데이터 처리**, **대규모 마이크로서비스 아키텍처**, 또는 **고성능이 요구되는 시스템**에서 더 많이 사용됩니다. 특히 스트리밍 기능과 낮은 대기 시간(Latency)이 필요한 환경에서 유리합니다 .

> [ 참고 ]  
> gRPC  
> <https://grpc.io/docs/>  
> <https://appmaster.io/ko/blog/grpc-dae-hyusig>  
> <https://aws.amazon.com/ko/compare/the-difference-between-grpc-and-rest/>  
> <https://velog.io/@qmflf556/gRPC>
>
> HTTP/2  
> <https://ibocon.tistory.com/257>
>
> RPC  
> <https://ko.wikipedia.org/wiki/%EC%9B%90%EA%B2%A9_%ED%94%84%EB%A1%9C%EC%8B%9C%EC%A0%80_%ED%98%B8%EC%B6%9C>
>
> protocol buffer  
> <https://protobuf.dev/overview/>  
> <https://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html>
