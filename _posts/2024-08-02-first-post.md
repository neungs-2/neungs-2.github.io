---
title: '전략 패턴 (Strategy Pattern)'
excerpt: '소셜 로그인 구현을 위해 가장 많이 쓰이고 있는 프로토콜은 OAuth 2.0과 OIDC가 있습니다. 이 두 프로토콜이 어떻게 인증 및 인가를 부여하는지 알아봅시다.'

categories:
  - Design Pattern
tags:
  - Design Pattern
last_modified_at: 2024-08-02T03:00:00+09:00
---

## 전략 패턴이란?

객체가 할 수 있는 유사한 행위들을 캡슐화하는 인터페이스를 정의하고, 각각의 행위에 대한 전략 클래스를 생성하여 **객체의 행위를 동적으로 바꾸고 싶을 때** 직접 행위를 수정하지 않고 전략을 바꿔주는 디자인 패턴.

전략을 사용하는 클라이언트와는 독립적으로 생성.
객체의 행위를 유연하게 확장하기 위한 디자인 패턴.
클라이언트는 시스템에 영향을 주지 않고 런타임에 사용되는 알고리즘을 변경 가능

- 특정 계열의 알고리즘을 정의 및 캡슐화 (전략 클래스 및 행위 인터페이스)
- 동일 계열의 알고리즘들을 상호 교체 가능하게 만듦

<br>

## 구조

![](https://velog.velcdn.com/images/gnlee95/post/df139964-6a32-4e7f-ae42-03dcf75183b6/image.png)

- **컨텍스트**: 가지고 있는 전략 콘크리트에 따라 행동이 달라짐
- **전략**: 제공하는 알고리즘에 대한 공동의 연산들을 인터페이스로 정의
- **전략 콘크리트 클래스**: 실제 알고리즘을 구현

<br>

## 구현 예시

- Robot은 `move()`, `temperature()` 라는 2개의 행위가 가능하고 각각의 행위는 2종류의 기능이 가능
  - (걷기/뛰기, 뜨거움/차가움)
    <br>
- 결국 총 4종류의 구현 클래스를 생성 가능
  - (걷기+뜨거움, 걷기+차가움, 뛰기+뜨거움, 뛰기+차가움)
    <br>

#### 전략 패턴 없이 상속만을 이용 시

- 행위의 종류 및 기능이 늘어날 수록 구현해야 하는 클래스 수가 급격하게 증가
- Method 수정 시 각각의 구현한 클래스의 모두 수정
- 새로운 행위 추가 시 각각의 구현한 클래스에 모두 추가

```java
public abstract class Robot {
    public abstract void move();
    public abstract void temperature();
}

public class HotAndWalkingRobot extends Robot {
    @Override
    public void move() {
    	System.out.println("I'm walking now.");
    }

    @Override
    public void temperature() {
    	System.out.println("It's hot.");
    }
}

public class ColdAndWalkingRobot extends Robot {
    @Override
    public void move() {
    	System.out.println("I'm walking now.");
    }

    @Override
    public void temperature() {
    	System.out.println("It's cold.");
    }
}

public class HotAndRunningRobot extends Robot {
    @Override
    public void move() {
    	System.out.println("I'm running now.");
    }

    @Override
    public void temperature() {
    	System.out.println("It's hot.");
    }
}

public class ColdAndRunningRobot extends Robot {
    @Override
    public void move() {
    	System.out.println("I'm running now.");
    }

    @Override
    public void temperature() {
    	System.out.println("It's cold.");
    }
}

public class Main {
    public static void main(String[] args) {
    	Robot robot = new HotAndWalkingRobot();
        robot.move();
        robot.temperature();
    }
}
```

<br>

#### 전략 패턴 사용 시

- Method 추가 및 수정 시 하나의 클래스만 생성, 수정
- 기본적인 상속만을 이용할 때보다 구현할 클래스 개수 감소
  - **기본적인 상속 구조의 클래스 수**: A행위의 기능 수 \* B행위의 기능 수
  - **전략 패턴 구조의 클래스 수**: A행위의 기능수 + B행위의 기능 수 + 행위 개수(인터페이스)

```java
// 전략을 사용하는 클라이언트
public class Robot {
    private MoveStrategy moveStrategy;
    private TemperatureStrategy temperatureStrategy;

    public Robot(MoveStrategy moveStrategy, TemperatureStrategy temperatureStrategy) {
        this.moveStrategy = moveStrategy;
        this.temperatureStrategy = temperatureStrategy;
    }

    public void move() {
        moveStrategy.move();
    }

    public void temperature() {
        temperatureStrategy.temperature();
    }
}

// 유사한 행위를 하나의 인터페이스로 정의하고 각 행위를 클래스로 구현
public interface MoveStrategy {
    void move();
}

public class Walk implements MoveStrategy {
    @Override
    public move() {
         System.out.println("I'm walking now.");
    }
}

public class Run implements MoveStrategy {
    @Override
    public move() {
         System.out.println("I'm running now.");
    }
}

public interface TemperatureStrategy {
    void temperature();
}

public class Hot implements TemperatureStrategy {
    @Override
    public temperature() {
         System.out.println("It's hot.");
    }
}

public class Cold implements TemperatureStrategy {
    @Override
    public temperature() {
         System.out.println("It's cold.");
    }
}

// 사용할 전략을 객체 생성 시 전달
public class Main {
    public static void main(String[] args) {
        Robot robot = new Robot(new Walk(), new Cold());
        robot.move();
        robot.temperature();
    }
}
```

<br>
<br>

---

전략 패턴은 이전에 NestJs로 개발을 할때 Passport.js를 사용하면서 공부했던 기억이 납니다. 정리를 하자면, 알고리즘 집합을 캡슐화하여 클라이언트와 결합도를 낮추고 런타임에 동적으로 전략을 선택할 수 있게 해주는 패턴입니다.

서비스마다 다르겠지만 저의 경우 생각보다 기획의 변경으로 로직이 변경되는 상황이 빈번했습니다. 그래서인지 전략 패턴의 핵심인 전략의 동적 선택을 알아보기 위해서 공부했었는데, 각 알고리즘을 캡슐화하여 유지보수를 쉽게 만들어주는 점이 더 인상 깊었던 것 같습니다 😄
