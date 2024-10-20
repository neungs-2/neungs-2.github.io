---
title: '[Conference] 인프런 Spring 밋업 Josh Long'
excerpt: '이번 밋업은 스프링 씬에서 세계적으로 유명한 Josh Long님이 "Bootiful Spring Boot: A DOGumentary" 라는 주제로 발표를 했습니다. 최신 Java, Spring Boot 기능을 이용해서 성능과 효율 및 안정성을 더 높일 수 있는 방식에 대한 강연이었습니다.'

categories: [Conference]
tags: [Spring, Conference]

toc: true
toc_sticky: true

date: 2024-10-20
---

지난 9월에 인프런과 VMware Tanzu에서 주최한 퇴근길 밋업에 다녀왔습니다. 운좋게 판교쪽으로 출근하는 기간에 밋업 참가자 추첨에 당첨되어서 다녀올 수 있었네요.

- [Josh Long Meetup](https://www.inflearn.com/course/offline/josh-long-meetup)

이번 밋업은 스프링 씬에서 세계적으로 유명한 Josh Long님이 'Bootiful Spring Boot: A DOGumentary' 라는 주제로 발표를 했습니다. 최신 Java, Spring Boot 기능을 이용해서 성능과 효율 및 안정성을 더 높일 수 있는 방식에 대한 강연이었습니다.

강연에서 이야기한 핵심들을 요약하자면 다음과 같습니다.

1. Lombok 대신 Java Record를 사용하자.

2. Sealed class를 사용해서 Sub Class를 제어하자.

3. Pattern matching을 사용해서 코드를 간결하게 만들자.

4. Smart Switch Expression으로 강력해진 Switch 구문을 활용하자.

5. Spring Modulith (Event-Driven)으로 public 클래스를 줄이고 아키텍쳐를 개선하자.

6. Spring AI를 활용해서 생성형 AI를 활용한 어플리케이션을 쉽게 만들자.

7. Virtual Thread를 사용해서 성능을 개선하자.

<br>

---

강연을 본격적으로 시작하면서 Josh가 언급한 개념이 하나 있습니다. 바로 **Data Oriented Programming** 입니다. Java에서 아래의 4가지 기능을 지원하면서 데이터 기반 프로그래밍을 도와준다고 합니다.

- Record
- Sealed class
- pattern matching
- Smart switch expressions

강연의 초반부는 위의 4가지 기능을 코드와 함께 직접 보여주면서 진행되었습니다.

<br>

## Record

Record는 Java 14부터 지원하는 불변객체입니다. 클래스 정의 시 Boilerplate가 될 수 있는 생성자, 접근자, `equals()`, `hashcode()`, `toString()`을 자동 생성해주기 때문에 일반 클래스를 사용하는 것보다 간결하게 코드를 짤 수 있습니다. 물론 일반 객체 사용시에도 Lombok을 사용할 수 있지만 Josh는 Record를 권하고 있습니다.

Record는 Kotlin의 Data class와 거의 유사합니다. 저도 Kotlin을 사용해본적이 있기 때문에 Josh의 말이 굉장히 공감이 되었고, 실제 Java 개발을 할 때에도 Record를 자주 사용합니다. DTO 같은 경우에는 데이터를 전달하는 역할에만 충실하면 된다고 생각하기 때문에 불변객체이면서 Boilerplate를 줄여주는 Record가 적합하다고 생각합니다.

<br>

## Sealed class

Sealed class는 자바 15에서 처음 도입되고, 자바 17에서 정식으로 지원되는 기능입니다. **상속 구조를 제한**할 수 있는 클래스로 어떤 클래스가 상속할 수 있는 **자식 클래스의 범위를 명시적으로 제한**하여, 더 안전하고 예상 가능한 상속 계층을 만들 수 있도록 해줍니다. 세부 사항은 다음과 같습니다.

- Sealed 클래스를 상속(permits)하려면 같은 모듈이나 패키지에 위치해야 함
- `permits` 키워드 뒤에 Sealed 클래스를 상속받을 sub class들을 선언
- **Permitted Subclass**는 3가지 타입으로 선언 가능
  - `sealed`: 상속을 추가적으로 제한 가능
  - `non-sealed`: 다른 클래스가 상속 가능하도록 선언
  - `final`: 더 이상 상속이 불가능한 클래스로 선언

이렇게 제한을 했을 때 Switch 구문과 함께 활용하면, 상속 받은 케이스가 누락된 경우에 컴파일 에러를 통해 케이스를 잡아주어서 더 안정적으로 코드를 짤 수있는 장점이 있습니다. 그 결과 모든 케이스에 대한 커버를 해주기 때문에 `default` 케이스를 생략할 수 있어서 코드도 간결해집니다. Josh 역시 이 부분을 강조했습니다.

```java
Shape rotate(Shape shape) {
	return switch (shape) {   // pattern matching switch
		case Circle c    -> c;
		case Rectangle r -> shape.rotate(angle);
		case Square s    -> shape.rotate(angle);
		// no default needed!
	}
}
```

<br>
저도 Sealed class가 Kotlin에 존재하기 때문에 많이 사용했고, 케이스에 대한 안전성을 많이 보장해주는 것을 느꼈기 때문에 공감이 많이 되었습니다. Kotlin은 `open` 키워드가 없는 클래스는 상속이 불가능하기 때문에 Sub class에 대한 별도의 타입이 없고 Permitted Subclass를 선언해주지 않아도 됩니다. 그래서 Java의 Sealed class 코드가 조금 더 복잡하다고 느꼈지긴 했습니다. 하지만 Java 개발 시에도 적극 활용할 예정입니다.

<br>

## Pattern matching

패턴 매칭은 Java 16부터 도입된 기능으로 `instanceof` 구문을 더 편리하게 사용하게 해줍니다. 기존 자바에서 객체의 타입을 확인한 후, 해당 타입으로 명시적으로 캐스팅하는 작업을 줄일 수 있습니다. Josh 역시 패턴 매칭을 사용해서 코드를 간결하게 만들라고 권했습니다.

```java
// 기존 방식
Object obj = "Hello";
if (obj instanceof String) {
    String s = (String) obj;  // 타입을 확인하고, 캐스팅 필요
    System.out.println(s.length());
}

// pattern matching
Object obj = "Hello";
if (obj instanceof String s) {  // 타입 확인과 캐스팅을 동시에 처리
    System.out.println(s.length());
}
```

<br>

## Smart Switch Expression

스위치 표현식은 Java 12부터 도입된 기능으로 기존 switch 구문을 개선하여 더 표현력 있고 안전한 방식으로 만들어줍니다. 기존의 switch 구문은 `break`을 강제하면서 가독성이 떨어졌는데, 스위치 표현식을 사용하면 가독성을 증가 시킬 수 있습니다. 주요 개선사항은 다음과 같습니다.

- **간결한 표현**: ->를 사용하여 더 간결하게 표현할 수 있습니다.
- **break 불필요**: 기존의 break가 필요 없어졌습니다. 이는 논리적 오류를 줄이고 코드 중복을 없앱니다.
- **값 반환 가능**: switch 구문이 **표현식**이 되어, 값을 반환할 수 있습니다.
- **다중 case 처리**: 여러 case를 한 번에 처리할 수 있습니다.

그리고 Java 13부터는 `yield` 를 사용하면 복잡한 표현식에서도 값을 반환할 수 있습니다.

```java
int day = 2;
String dayName = switch (day) {
    case 1 -> "Sunday";
    case 2 -> {
        System.out.println("Processing Monday...");
        yield "Monday";  // 복잡한 로직을 처리한 후 값을 반환
    }
    case 3 -> "Tuesday";
    default -> "Invalid day";
};
```

### Pattern matching + Smart Switch Expression

Java 17부터는 두가지 기능을 결합 시킬 수 있게 되었습니다. 즉, Switch 문에서 다양한 타입의 패턴 매칭이 가능해졌습니다.

```java
Object obj = "Hello"; // Object 타입.
String result = switch (obj) {
    case Integer i -> "Integer: " + i;
    case String s -> "String: " + s;
    default -> "Unknown type";
};

System.out.println(result);  // 출력: "String: Hello"
```

Josh는 Switch문이 if문보다 성능이 더 좋기 때문에 Switch문에 대한 Java의 새로운 지원이 굉장히 강력하다고 피력했습니다. 저는 사실 Switch 구문의 예전 형태만 알고 있었는데 이 기능을 보니 Kotlin의 `when`과 거의 유사하다고 느꼈습니다.

<br>

## Event-Driven (Spring Modulith)

강연의 중반부에는 Spring Modulith 기반의 **Event-Driven**에 대한 내용이었습니다. 하나의 컨트롤러에서 너무 많은 책임과 의존성을 지닌, 속칭 God Class를 리팩토링하는 과정과 함께 소개했습니다. 저도 이전에 기능 실패 시 데이터를 저장하기 위해 `@TransactionalEventListener`를 프로젝트에서 사용한 경험이 있어서 해당 내용도 공감하면서 들을 수 있었습니다.

**Event-Driven**은 의존성을 분리하고 결합을 느슨하게 만들 수 있는 장점이 있습니다. 로직을 분리해서 코드도 깔끔해지고 단위 테스트를 더 쉽게 만들어줍니다. 만약 이벤트가 비동기적으로 실행되어야 한다면 `@ApplicationModuleListener`를 사용할 수 있습니다. `@Async`, `@Transactional`, `@EventListener` 기능이 모두 결합된 어노테이션입니다. 그리고 시스템이 비정상적으로 종료될 때 메시지를 재처리해주는 기능까지 제공합니다.

**Spring Modulith**는 이렇게 Event 기반의 개발과 함께 어플리케이션의 구조를 개선할 수 있도록 해줍니다. 또 이렇게 만든 아키텍쳐를 문서화해주는 기능도 제공합니다.

Josh가 추가적으로 아키텍처에 대해서 언급한 부분이 있습니다. 바로 `app.controller`, `app.service`, `app.repository`로 이어지는 계층형 구조를 안티패턴으로 규정하고 도메인 기반으로 패키지를 분리하기를 권했습니다. 계층형 구조는 필연적으로 `public`을 사용하며 어디에서든 의존할 수 있는 구조를 만들기 때문입니다.

<br>

## Spring AI

Spring AI는 OpenAI 처럼 생성형 AI를 서비스에 접목시킬 수 있도록 도와줍니다. `RestTemplate`, `WebClient`를 통해 직접 HTTP 요청을 하지 않고도 `ChatClient` 등을 사용해서 편리하게 요청을 보낼 수 있게 해줍니다. 이외에도 `SimpleVectorStore`처럼 AI를 사용하는데 필요한 여러 기능을 지원합니다.

Josh는 AI를 활용한 어플리케이션을 만들 때 확장성이 좋은 Spring이 유리하다고 언급하며 Spring AI를 활용해서 보다 쉽게 코드를 작성할 수 있는 점을 강조했습니다. Spring 쪽에서도 AI 관련된 기능에 대해서 신경을 지속적으로 쓰고 있는 것을 알 수 있었습니다. 개인적으로 AI 모델 생성 등의 코드는 Python이 유리하다고 생각하는데, **AI를 활용한 어플리케이션 개발**에 한정해서는 Spring이 좋은 선택지가 될 수 있을 것 같습니다.

<br>

## Virtual Thread

AI 사용시 필연적으로 네트워크 호출이 필요한데 AI의 처리 및 I/O 작업의 시간이 많이 걸립니다. 그래서 효율성을 위해서 논블로킹, 비동기적으로 동작하게 만들어줄 필요가 있습니다. 웹 플럭스를 사용할 수도 있지만 변경을 위해서는 비용이 들고, 아직 웹 플럭스를 쓰기 싫어하는 개발자도 있습니다. 그래서 이런 경우에 좋은 대안이 바로 **Virtual Thread**입니다.

가상 스레드는 Java 21 부터 적용된 경량 스레드 기술입니다. 기존의 Java 스레드는 OS에 의해서 스케줄링 되지만 가상 스레드는 JVM 위에서 스케줄링 됩니다. 따라서 컨텍스트 스위칭 비용을 많이 줄일 수 있습니다. 또한 스레드 풀 확장이 용이하다는 장점도 존재합니다.

Josh는 강연에서 Hey라는 툴을 사용해서 부하테스트를 진행했습니다. 부하테스트 결과는 기존 Java 스레드보다 가상 스레드를 사용했을 때의 성능이 2배 가까이 좋았습니다.

- **[ 부하테스트 결과 ]**

  - thread 10개, 요청 개수 60개

  - 기존 Java 스레드

    - 작업당 소요시간: 5.2 ~ 17.98s
    - 평균 소요시간: 8.76s
    - 총 작업 소요시간: 36.58s

  - 가상 스레드
    - 작업당 소요시간: 5.2 ~ 7.89s
    - 평균 소요시간: 5.93s
    - 총 작업 소요시간: 19.42s

<br>

가상 스레드는 Java 21 발표시에도 가장 주목을 받는 기술이었고 역시나 강연 이후의 Q&A에서도 관련된 질문이 많이 나왔습니다. Josh는 Web Flux를 사용하지 않을 것이라면 Virtual Thread가 좋은 대안이 될 것이고 CPU 바인딩 작업이 많은 경우가 아니라 대부분 I/O 바인딩일 경우에는 가상 스레드를 쓰는 것이 좋다고 했습니다. Josh의 얘기를 계속 들으면서, 역시 Java 개발의 미래는 Virtual Thread가 되겠다(?)라는 생각이 들었습니다.

<br>

---

지금까지 많지는 않지만 6번 정도 컨퍼런스 혹은 밋업에 참여해봤는데, 이번 밋업이 가장 와닿는게 많고 실질적인 도움이 되는 부분이 많았네요. 제가 이전까지 가본 밋업들은 매번 소주제로 세션을 나누어서 발표하는 형태였습니다. 하지만 이번 강연은 강아지 입양 서비스를 만들면서 여러 기술들은 하나의 세션에서 자연스럽게 이어서 발표하는 것이 인상 깊었습니다. 그리고 Josh가 굉장히 유머러스하고 말을 잘해서 처음부터 끝까지 집중해서 강연을 들을 수 있던 것 같아요!

개발을 하다보면 어느 순간 늘 하던대로, 관성대로 개발하게 되는 순간이 오는 것 같습니다. 특히, 함께 일하는 팀의 분위기가 보수적이라면 더더욱 그런 것 같아요. 이럴 때 컨퍼런스에 한번씩 참여하면 새로운 자극과 동기부여를 받고 갈 수 있어서 참 좋더라구요. 인프런 퇴근길 밋업은 처음 참석해보는데, 다음 밋업도 기대해보려고 합니다 😁
