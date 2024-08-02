---
title: 'OAuth 2.0과 OIDC(OpenID Connect) 프로토콜'
excerpt: '소셜 로그인 구현을 위해 가장 많이 쓰이고 있는 프로토콜은 OAuth 2.0과 OIDC가 있습니다. 이 두 프로토콜이 어떻게 인증 및 인가를 부여하는지 알아봅시다.'

categories:
  - Blog
tags:
  - OAuth & OIDC

toc: true
toc_sticky: true

date: 2024-08-03
last_modified_at: 2024-08-01T00:23:30+09:00
---

이제는 너무나도 익숙한 소셜 로그인은 유저와 기업 모두에게 매력적인 인증 방법입니다. 유저는 간편하게 로그인할 수 있고 기업은 신규 유저의 가입 장벽을 낮추고 신뢰성 있는 타기업에게 인증의 책임을 미룰 수 있죠. 소셜 로그인 구현을 위해 가장 많이 쓰이고 있는 프로토콜은 OAuth 2.0과 OIDC가 있습니다. 이 두 프로토콜이 어떻게 인증 및 인가를 부여하는지 알아봅시다.

> **인증 (Authentication)**  
> 신원 확인. 유저가 누구인지 확인하는 절차 (로그인, 회원가입)
>
> **인가 (Authorization)**  
> 유저에 대해 특정 리소스나 기능에 액세스 가능한 권한을 부여하는 것 (관리자, VIP)

<br/>

## OAuth 2.0

&nbsp; 위임 권한부여를 위한 표준 프로토콜인 OAuth는 사용자가 비밀번호를 제공하지 않고 서드파티 어플리케이션에게 접근 권한을 부여할 수 있게 해줍니다. 2010년 IETF에서 OAuth 1.0 공식 표준안이 RFC 5849로 발표되었으며, 현재는 OAuth 2.0 (RFC 6749, RFC 6750)이 많이 쓰이고 있습니다.

> **위임 권한부여 (Delegated Authorization)**
>
> - 서드파티 어플리케이션이 사용자의 데이터에 접근하도록 허가해주는 것
> - 서드파티에게 아이디/비밀번호를 주기보다는 주로 OAuth를 통해 위임 권한부여

<br/>

### 용어 정리

OAuth 2.0 의 로직 흐름을 이해하기 위해 몇 가지 용어를 알아야 합니다.

- **Client**: 사용자 데이터에 접근하고 싶어 하는 어플리케이션
- **Resource Owner**: 클라이언트 어플리케이션이 접근하려는 데이터의 사용자 (소유자)
- **Resource Server**: 클라이언트 어플리케이션이 접근하려는 데이터를 저장하고 있는 서버
- **Authorization Server**: 사용자로부터 권한을 부여받아 클라이언트가 사용자 데이터에 접근할 권한을 부여해 주는 권한부여 서버
- **Access Token**: 클라이언트가 리소스 서버의 사용자 데이터에 접근하기 위해 사용 가능한 유일한 키

<br/>

### OAuth 2.0 요청을 위한 파라미터

Client가 Authorization Server에 요청을 보낼 때 주로 다음과 같은 설정값을 Query String을 통해 전달합니다.

- **response_type**: Authorization Server로부터 받길 원하는 응답의 타입 (code, token 등)
- **scope**: Client가 Resource Server에서 접근하고 싶은 리소스 리스트
- **client_id**: OAuth 세팅을 할 때, Authorization Server에 의해 제공. Client가 누구인지 알아내기 위해 사용
- **client_secret**: Authorization Server에 의해 제공. Code와 client_secret을 가지고 Access Token 받게 됨
- **redirect_url**: OAuth 플로우가 끝나고 Authorization Server가 응답을 보내줄 URL

<br/>

### 인증 방식과 동작 흐름

OAuth 2.0에서는 다음의 4가지 인증 방식으로 동작합니다.

> Client는 어플리케이션으로 프론트엔드 + 백엔드 서버를 포괄하는 개념  
> 시퀀스 다이어그램을 보실 때 유의하세요!

<br/>

#### 1. 권한 부여 코드 승인 방식 (Authorization Code Grant)

- 자체 생성한 Authorization Code를 전달하는 방식
- OAuth2.0에서 가장 기본이 되는 방식
- `response_type=code`, `grant_type=authoration_code` 형식으로 요청
- Authoration Server가 Redirect 시 엑세스 토큰을 전달하면 브라우저에 토큰이 바로 노출되기 때문에 프론트앤드에서 code를 받아서 서버로 전달하면 서버에서 엑세스 토큰을 요청
- Code를 Access Token으로 변환할 때 client_secret이 필요 (결국, 백엔드 서버 필요)
- 백엔드 사이에서만 토큰이 이동하여 비교적 안전한 방식

![권한 부여 코드 승인 방식](https://velog.velcdn.com/images/gnlee95/post/af188245-cddf-4a3b-9a22-d4cfae48caf9/image.png)

<br/>

#### 2. 암묵적 승인 방식 (Implicit Grant)

- 자격 증명을 안전하게 저장하기 힘든 클라이언트 사이드에서 OAuth2.0 인증에 최적화된 방식.
- `response_type=token` 형식으로 요청.
- Autorization Code 발급 없이 바로 Access Token 발급되기 때문에 만료 기간이 짧아야 함.
- 절차가 비교적 간단하지만 Access Token이 URI를 통해 전달되어 보안에 취약

![암묵적 승인 방식](https://velog.velcdn.com/images/gnlee95/post/13d366d0-4d5e-4467-88a9-bf9860a567a7/image.png)

<br/>

#### 3. 자원 소유자 자격 증명 방식 (Resource Owner Password Credentials Grant)

- Authorization Server, Resource Server, Client가 모두 같은 시스템에 속해 있을 때만 사용 가능
- ID, Password로만 Access Token을 발급받는 방식
- `grant_type=password` 형식으로 요청

![자원 소유자 자격 증명 방식](https://velog.velcdn.com/images/gnlee95/post/3c5cc40f-55a1-4838-8928-ca434a4f0173/image.png)

<br/>

#### 4. 클라이언트 자격 증명 방식 (Client Credentials Grant)

- 클라이언트의 자격 증명만으로 Access Token을 획득하는 방식
- User가 아닌 Client에 대한 인가가 필요할 때 사용
- 즉, Client에 대해 리소스 접근 권한이 설정된 경우 사용
- 자격 증명을 안전하게 보관할 수 있는 클라이언트에서만 사용

![클라이언트 자격증명 방식](https://velog.velcdn.com/images/gnlee95/post/fd06c8de-6829-4471-b3f2-d39117bcb1bc/image.png)

<br/>
<br/>

## OpenID Connect

&nbsp; OIDC는 OAuth 2.0을 기반으로 만들어진 유저의 인증(Authentication)을 위한 프로토콜 입니다. OIDC는 OAuth 2.0을 확장하여 인증 방식을 표준화 합니다. OpenID를 관리하는 OpenID Foundation에서 정의한 OpenID의 개념은 다음과 같습니다.

![OIDC](https://velog.velcdn.com/images/gnlee95/post/7ef86684-99c4-45a5-bd0d-72281f21fc11/image.png)

> OpenID Connect 1.0은 OAuth 2.0 프로토콜 위에서 동작하는 간단한 ID 레이어입니다. 이를 통해 클라이언트는 인증 서버에서 수행한 인증을 기반으로 최종 사용자의 신원을 확인할 수 있을 뿐만 아니라, 최종 사용자에 대한 기본 프로필 정보를 상호 운용 가능하며 REST와 유사한 방식으로 얻을 수 있습니다. OpenID Connect를 사용하면 웹 기반, 모바일, 자바스크립트 클라이언트 등을 포함한 모든 유형의 클라이언트에서 인증 세션과 최종 사용자에 대한 정보를 요청하고 받을 수 있습니다. 스펙은 확장 가능하므로 필요에 따라 참가자들에게 ID 데이터 암호화, OpenID 제공자 확인, 로그아웃 등을 이용할 수 있습니다.

<br/>

### 인증 방식과 동작 흐름

기존 OAuth 2.0의 동작 흐름과 거의 유사하며 **ID token**을 추가 발급한다는 차이점이 있습니다.
추가로, OpenID 문서를 읽다 보면 IDP, RP라는 용어가 등장하는데 각각 다음과 같습니다.

- **IDP** (IDentity Provider): Google, Apple 같은 간편 로그인 서비스 제공사 (OpenID 제공자)
- **RP** (Relying Party): 사용자를 인증하기 위해 IDP에 의존하는 주체 (어플리케이션)

  ![OIDC 동작 흐름](https://velog.velcdn.com/images/gnlee95/post/90e46121-d37b-4667-aaf4-913f6d16faa5/image.png)

<br/>
<br/>

## OAuth 2.0 과 OIDC의 차이점

#### 1. 스코프 (Scopes)

OAuth 2.0에서는 제공자가 원하는 대로 요청 범위를 설정 가능하여 유연한 사용이 가능했지만 상호 운용에 취약했습니다. OIDC는 요청범위를 profile, email, address, phone으로 표준화했습니다.  
[참고](https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims)

<br/>

#### 2. 클레임 (claims)

OIDC에서 ID토큰의 payload는 claims라고 알려진 필드들을 포함합니다. OIDC는 이런 클레임들을 표준화했습니다. 기본적인 클레임은 다음과 같습니다.

- iss: 토큰 발행자
- sub: 사용자의 유니크한 식별자
- email: 사용자 이메일
- iat: 토큰 발행 시간 (Unix time)
- exp: 토큰 만료 시간(Unix time)

[참고](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)

<br/>

#### 3. 사용자 정보 요청 엔드포인트 통일

OIDC는 사용자가 요청하는 엔드포인트도 표준화했습니다. 예를 들어 /userinfo를 통해 사용자 메타데이터 정보를 검증합니다.  
[참고](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo)

<br/>

#### 4. ID token

OIDC의 동작 흐름에서 가장 눈에 띄는 차이가 ID token의 유무입니다. ID token은 JWT로 생성이 되어 Payload 내부에 클레임을 포함합니다. 즉, ID token을 복호화하여 사용자 정보를 얻을 수 있습니다. OAuth 2.0에서 액세스 토큰을 얻고 다시 사용자 정보를 요청하는 것보다 네트워크 통신 비용이 절감됩니다.

<br/>
<br/>

## OAuth 2.0, OIDC 중 무엇을 사용해야 할까?

&nbsp; 정리하자면, **가능하면 OIDC를 쓰되 필요로 하는 IDP에서 제공하는 방식을 사용**하면 된다고 생각합니다. OAuth 2.0은 인가, OIDC는 인증에 중점을 둔다고는 하지만 큰 차이점은 없었습니다. (ID 토큰과 더 강력한 표준화를 곁들인?) 오히려 OAuth와 OIDC 보다는 자사 서비스의 유저들이 어떤 IDP로 연결했을 때 회원가입 없이 로그인할 수 있을지가 더 중요할 것입니다. 이때 OAuth 2.0 만 제공하는 IDP라면 OAuth 2.0을 쓸 수 밖에 없을 것이고 구글처럼 OpenID가 적용된 OAuth가 기본 사양이라면 OIDC를 쓰지 않을 수 없을 것 입니다.

만약 OAuth 2.0과 OIDC 모두를 지원한다면 OAuth 2.0이 레거시일 확률이 큽니다. 표준화가 적용된 OIDC가 확장성도 좋고 네트워크 비용도 적게 들 테니 OIDC를 쓰지 않을 이유는 없어 보입니다. 하지만 두 프로토콜 사이의 큰 차이는 없습니다.

<br/>

### 참고문서

> [RFC 표준](https://www.rfc-editor.org/rfc/rfc6750)  
> [OIDC](https://openid.net/specs/openid-connect-core-1_0.html)  
> [구글](https://developers.google.com/identity/openid-connect/openid-connect?hl=ko#getcredentials)  
> [네이버](https://developers.naver.com/docs/login/devguide/devguide.md)  
> [카카오](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api)  
> [애플](https://developer.apple.com/documentation/sign_in_with_apple/sign_in_with_apple_rest_api/authenticating_users_with_sign_in_with_apple)  
> <https://velog.io/@jakeseo_me/Oauth-2.0%EA%B3%BC-OpenID-Connect-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C-%EC%A0%95%EB%A6%AC> > <https://tecoble.techcourse.co.kr/post/2022-10-24-openID-Oauth/>  
> <https://hudi.blog/open-id/>  
> <https://wildeveloperetrain.tistory.com/247>  
> <https://velog.io/@wlsh44/OpenID-Connect%EC%99%80-OAuth2.0>  
> <https://www.samsungsds.com/kr/insights/oidc.html>
