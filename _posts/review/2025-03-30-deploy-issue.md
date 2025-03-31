---
title: '[회고] NGINX 무중단 배포 이슈 및 스크립트 개선'
excerpt: '이번 프로젝트에서 블록체인 지갑의 Private key 암호화 및 ECDSA 서명을 HSM을 이용하기로 했습니다. 어플리케이션 서버와 HSM 간의 연동은 업체에서 지원해주는 Luna Client 를 사용해서 구현했습니다.'

categories: [Review]
tags: [Review, Deploy, ClassLoader]

toc: true
toc_sticky: true
published: true

# image:
#   teaser:
#   feature:

date: 2025-03-30
---

지난번 배포 과정 중 발생한 이슈와 해결 과정을 공유해보고자 합니다. 분명 배포중 문제가 발생했고, 기존 실행중이던 프로세스가 계속 동작중이었지만 일부 기능이 동작하지 않았습니다. 결론부터 말하자면, **실행중인 jar 파일을 SFTP로 덮어쓸 때 일부 클래스 로딩에 실패하는 문제**가 있었습니다. 실행중인 파일을 덮어쓰지 않는 것은 에러를 방지하기 위한 기본적인 사항이지만 사소한만큼 놓치기 쉬운 포인트라고 생각되어 오랜만에 포스트를 남깁니다.

<br>

## 1. 이슈 발생

### 배포 프로세스

저희 서비스의 트래픽 중 대부분은 레거시 프로젝트의 트래픽이기 때문에 리뉴얼 프로젝트는 간단한 인프라를 유지하고 있습니다. 그래서 배포 방식도 다음과 같이 단순한 방식으로 구현되어 있습니다.

Jenkins에서 배포 파이프라인이 실행되면 CICD 서버에서 git clone을 받고 AWS Secret Manager에서 환경변수를 로드해서 빌드를 진행합니다. 이후에 SFTP로 jar 파일을 어플리케이션 서버로 전송하고 배포 스크립트를 실행합니다. jar 파일 실행에 성공하면 NGINX 연결 포트를 배포된 신규 프로세스로 변경하고 기존 프로세스는 graceful shutdown을 진행합니다.(Blue-Green) 만약 배포에 실패하면 NGINX 포트는 그대로 기존 프로세스에 연결된 채로 유지되기 때문에 사용자에게 영향은 없습니다.

### 무중단이 아닌 반중단(?) 배포...

위에 설명한 바와 같이 저희는 NGINX를 사용한 무중단 Blue-Green 배포를 구현했지만 간혹 배포중에 발생한 장애가 문제가 되었습니다. 이슈 공유시에는 NGINX 문제로 파악된다는 전달을 받았습니다.

최근 저도 환경변수 설정 오류 때문에 배포 중 실행 단계에서 문제가 발생했습니다. 기존의 프로세스를 shutdown 시키지 않고 포트도 그대로 잘 연결되어 있었기에 어플리케이션이 정상동작을 할 것으로 예상했습니다. 하지만 **Swagger 새로고침 시 White Label 페이지가 응답했고 API도 일부만 동작하지 않았습니다.** 여기서 이상한 점은 일부 기능만 동작하지 않는 점입니다. 과거 전달받은 대로 NGINX 리로드에 문제가 있었다면 요청을 아예 포워딩해주지 않아야 합니다. 그래서 NGINX가 아니라 무중단 배포 구현에 문제가 있음을 깨닫고 에러 메시지를 분석했습니다.

<br>

## 2. 원인 파악

### 에러 메시지

에러 메시지는 `NoClassDefFoundError` 혹은 `AssertionError` 두 가지 유형으로 발생했습니다.

- `NoClassDefFoundError`: 컴파일 시점에 존재했던 클래스를 런타임 시에 찾지 못한 경우에 발생
- `AssertionError`: 패키지가 이미 JVM에는 등록되었지만, 클래스 로더의 패키지 맵에서는 찾을 수 없는 비정상 상태에서 발생

```
# 1.
Caused by: jakarta.servlet.ServletException: Handler dispatch failed: java.lang.NoClassDefFoundError: org/springframework/validation/DefaultMessageCodesResolver
...
Caused by: java.lang.ClassNotFoundException: org.springframework.validation.DefaultMessageCodesResolver

# 2.
Handler processing failed: java.lang.AssertionError: Package org.springframework.web.server has already been defined but it could not be found
```

<br>
두 메시지는 공통적으로 **클래스 로딩 실패**를 의미합니다. JVM의 ClassLoader가 .jar 내부에서 클래스를 로딩하지 못했을 때 발생하며 보통은 jar 자체가 손상되었거나, zip entry 구조에 문제가 있을 때 발생합니다. 이외에도 의존성 버전을 잘못 맞추는 등 원인은 다양합니다.

ClassLoader는 `loadClass()` 내부에서 Class를 정의하기 전에 `definePackageIfNecessary()`를 호출하여 Package를 먼저 정의합니다. 그리고 그 과정 중에 jar 파일에 접근해서 데이터를 동적으로 로딩합니다. 그래서 배포 당시의 타이밍에 따라 아래와 같은 에러 발생 케이스로 나뉜다고 추론할 수 있었습니다.

- Class가 이미 로딩된 Case: 정상동작
- Package만 정의된 Case: `NoClassDefFoundError` 발생
- ClassLoader의 `Packages`에 Package가 정의되지 않은 Case: `AssertionError` 발생

> AssertionError의 경우에 ClassLoader 패키지 맵이 손상된 것인지, 상위 ClassLoader에서 `IllegalArgumentException`이 발생한 것인지 불분명합니다.

결국, 클래스 로딩이 실패하는 에러 메시지를 통해서 jar 파일과 관련된 문제가 있다는 예상을 할 수 있었습니다.

<br>

### 젠킨스 파이프라인 및 배포 스크립트 분석

역시 예상대로 젠킨스 파이프라인에서 파일을 전송한 경로와 배포 스크립트에서 실행하는 파일의 경로가 동일했습니다. 즉, **기존 프로세스가 참조하는 파일 경로에 신규 jar를 덮어쓰고 있던 것**입니다. 실행중인 파일을 덮어쓰는 것은 혹시 모를 에러를 유발할 수 있기에 피해야합니다. SFTP 전송 자체가 원자적이지 않기 때문에 전송 과정에서도 파일을 참조할 수 있습니다. 그래서 실제로 Atomic하게 SFTP를 사용하기 위해서 임시 디렉토리에 파일을 업로드한 다음 단일 원자적 작업으로 최종 목적지로 파일을 이동시키는 방법을 사용할 수 있습니다.

우선 가설 검증을 위해 파일 전송 경로와 실행 파일 경로를 분리하고 실행중인 jar 파일 이름과 새로운 jar 파일 이름을 분리해서 기존 처럼 배포 실패 상황을 재현했습니다. 그 결과, 기존 프로세스에서 에러가 발생하지 않고 정상 동작함을 확인할 수 있었습니다.

<!-- SFTP와 CP의 차이점에 대해서 쓰려고 했지만 관련 문서나 내용을 찾지 못해서 생략.. -->
<!--하지만 여기서 또 의문이 생깁니다. 실행중인 프로세스는 기존의 파일 디스크립터를 유지하고 있을 것이고 기존의 inode를 계속 참조할 것입니다. 파일이 덮어씌워져도 리눅스 내부에서는 진짜 삭제된 것이 아닐텐데 왜 Jar 파일을 참조할 때 오류가 발생할까요? 🧐 아무래도 Zip Entry 파일 내부의 offset 정보와 새로 빌드된 jar 구조의 불일치로 문제가 발생한 것 같았습니다. 그래서 inode가 변경되지 않은 것이 이유라는 가설과 함께 로컬 환경에서 다시 재현을 해보았습니다.

<br>

### SFTP와 CP, MV의 차이점

재현 방법은 다음과 같습니다.

1. 별도의 디렉토리에 jar 파일을 두고 배포 디렉토리에 위치한 또 다른 jar 파일을 실행합니다.
2. 실행중인 jar 파일을 별도의 디렉토리에 위치한 jar 파일로 덮어씌웁니다.
3. 이때 `cp`와 `mv` 명령어로 각각 실행해봅니다.
4. 실행중인 경로에 위치한 파일의 inode가 유지된 상태와 변경된 상태를 비교합니다.
    (`mv`는 inode가 유지된다고 헷갈리지 마세요! 비교 대상이 실행 경로에 위치한 파일의 inode 입니다.)

결과가 어땠을까요? 제 가설대로 `cp`를 통해 덮어씌운 경우에 inode가 변경되지 않아서 문제 상황이 재현됐을까요? 두 경우 모두 에러가 발생하지 않았습니다. 로컬 환경이 아닌 실제 Dev 서버에서도 마찬가지였습니다.

놓치고 있는 부분을 확인하기 위해 다시 젠킨스 파이프라인을 살펴보았고, 젠킨스에서는 `ssh`를 통해 파일을 전송하는 사실을 알아챘습니다. 그래서 Dev 서버에서 실행중인 프로세스가 참조한 파일을 `SFTP`로 덮어씌워보니 비로소 문제 상황을 재현할 수 있었습니다. `SFTP` 역시 덮어쓰는 파일의 inode를 변경하지 않았습니다. -->

<br>

## 3. 스크립트 수정

파일 덮어쓰기를 막기 위해 우선 SFTP를 통해 전송되는 파일 경로를 배포와 무관한 별도의 디렉토리로 수정했습니다. SFTP 파일 전송 자체가 원자적이지 않기 때문에 안전한 파일 전송을 위해서라도 분리할 필요가 있습니다. 전송이 완료된 후에 다른 디렉토리로 파일을 복사해서 사용하는 것이 안전합니다.

이후에 가장 간단한 파일명 변경 방식을 적용시켰습니다. 배포 시 저희는 매번 2개의 포트를 번갈아 사용중이기 때문에 다운로드 받은 파일에 포트번호를 추가해서 실행중인 jar 파일을 덮어쓰는 문제를 해결할 수 있었습니다.

```sh
JAR_NAME="${JAR_NAME_BASE}.${PORT_TO_RUN}.jar"
cp "${JAR_NAME_BASE}.jar" ${JAR_NAME}
nohup java -jar "${JAR_NAME}" ... 2>&1 &
```

<br>
  
파일명 변경보다 주로 많이 사용하는 심볼릭 링크 방식도 있습니다. 스크립트도 더 간결하고 자바 실행 경로를 항상 동일하게 유지할 수 있기 때문에 파일명 변경 방식보다 더 좋은 방식이라고 생각해서 수정하여 적용했습니다.

```sh
ln -Tfs "${NEW_JAR_PATH}" "${LINK_NAME}"
nohup java -jar "${LINK_NAME}" ... 2>&1 &
```

<br>

> [References]  
> [Spring Boot LaunchedURLClassLoader](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-tools/spring-boot-loader-classic/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java)  
> [Stack overflow](https://stackoverflow.com/questions/32477145/java-lang-classnotfoundexception-ch-qos-logback-classic-spi-throwableproxy)  
> [심볼릭 링크로 무중단 배포하기](https://11st-tech.github.io/2023/12/11/spring-batch-non-stop-deploy/)
