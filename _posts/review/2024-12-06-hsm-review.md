---
title: '[회고] HSM(LunaClient)을 사용한 블록체인 지갑 개발'
excerpt: '이번 프로젝트에서 블록체인 지갑의 Private key 암호화 및 ECDSA 서명을 HSM을 이용하기로 했습니다. 어플리케이션 서버와 HSM 간의 연동은 업체에서 지원해주는 Luna Client 를 사용해서 구현했습니다.'

categories: [Review]
tags: [Review, Security, HSM, PKCS#11]

toc: true
toc_sticky: true
published: true

# image:
#   teaser:
#   feature: 

date: 2024-12-15
---

이번 4분기에 우리은행 WON 앱에 있는 블록체인 지갑 기능을 개발할 기회가 생겼습니다. 저는 솔루션으로 제공되는 코어 부분의 개발을 담당했습니다.
그중에서 가장 기억에 남는 것은 HSM 연동 개발이었어요. 그동안 회사에서 보안 하드웨어를 다룰 기회가 없어서 새로운 경험이 되었습니다.
그리고 ~~가독성 안좋고 불친절한~~ 문서를 제외하면 레퍼런스가 별로 없어서 더 기억에 남았던 것 같습니다.

이번 프로젝트에서 블록체인 지갑의 Private key 암호화 및 ECDSA 서명을 HSM을 이용하기로 했습니다.
어플리케이션 서버와 HSM 간의 연동은 업체에서 지원해주는 <u>Luna Client</u> 를 사용해서 구현했습니다. HSM을 처음 다루느라 고생하시는 분들에게 제 회고가 도움이 되면 좋겠네요.

<br>

## 1. HSM 동작 이해: 연동을 위한 기본 개념

코딩을 하면서 HSM 내부 메모리와 애플리케이션의 메모리를 분리해서 생각할 필요가 있습니다. HSM은 하드웨어이고 LunaClient는 하드웨어 간의 통신을 위해서 쓰이는 라이브러리 입니다. 메모리 또한 한정적이기 때문에 효율적인 메모리 사용을 위해서 명확한 크기를 지정해야 합니다. 그래서 세션 연결을 하고, 버퍼도 잡는 것입니다. 이 점을 고려하지 않고 처음 LunaClient를 사용한 코드를 보면 어색하거나 불필요한 과정이 존재한다고 느낄 수 있습니다.

### Session  
HSM과 세션 연동을 먼저 해주고, 필요한 동작을 수행한 후 연동을 해제하는 과정이 필요합니다.
세션 연동 과정은 Initialize, Open, Login 절차를 거치고 해제는 Logout, Close, Finalize 절차를 거칩니다. 이는 자원이 많이 소요되는 작업으로 레이턴시가 꽤나 있어서 매서드 호출때마다 연결을 하려면 성능상 손해를 많이 보게됩니다. 그래서 저는 `@PostConstruct`, `@PreDestroy` 를 사용해서 프로세스 시작/종료 시에만 연동/해제의 모든 과정을 한번 거치도록 만들었습니다. 미리 세션을 연결해두면 메서드 호출시에는 세션 Open/Close 과정만 거칠 수 있게 됩니다.

세션 연결시에는 `slotId` 값이 필요합니다. 이는 미리 연결해둔 하드웨어(혹은 네트워크) HSM의 slot 번호입니다. 현재 연결된 slot을 알고 싶다면 `safenet/lunaclient/bin`의 설치 경로에서 `./vtl verify` 명령어로 확인하실 수 있습니다. 

### Object Handle
LunaClient에서는 `Object Handle` 객체를 통해 키를 핸들링 합니다. `CryptokiEx` 의 메서드에 키를 직접 주입하는 것이 아니라 핸들러 객체를 통해서 전달하고 받습니다. 이때 주의할 점은 `Object Handle` 객체 내부에는 실제 키값이 들어있지 않다는 사실입니다. 핸들러 객체는 단순히 매핑된 키를 가리키는 용도이고, 요청이 성공적으로 수행됐는지만 알려줍니다. 직관적이지 않아서 처음에 당황할 수 있습니다. 하지만 HSM 연산은 별도의 메모리에서 수행되며 보안을 위한 것임을 생각해보면 이해가 되실겁니다.  

즉, `Object_Handle`은 HSM 메모리에 올라간 객체를 가리키는 용도입니다. 


### 전달 값 형식
이론적으로 키나 민감 데이터는 격리된 HSM 메모리에만 두고 사용하는 것이 가장 바람직하지만 실제 개발할 때에는 다양한 상황이 존재합니다. `Object Handle`을 통해서 Key를 가리키는 것만으로는 부족할 때가 있을 것입니다. 당연히 HSM에서 직접 키를 넘겨주거나 받는 것도 가능합니다.

어플리케이션에서 HSM으로 키를 직접 전달할 때에는 Hexadecimal 형식의 String을 바이너리(`byte[]`)로 변환하여 전달하게 됩니다.

반대로 HSM에서 어플리케이션으로 키를 전달할 때에는 미리 버퍼를 설정해두고 HSM에 요청하면 해당 버퍼에 값을 채워넣는 형식으로 사용이 가능합니다. 버퍼에는 바이너리 데이터가 전달됩니다.

### Attribute 
HSM에서 키를 생성하거나 암복호화, 서명등을 할 때에 많은 설정값들을 포함하게 됩니다. 
로그인 상태에서만 사용이 가능한지, 키의 타입은 무엇이고 파생연산은 가능한지, 어떤 알고리즘으로 서명을 할 것인지, 세션/영구 저장 옵션 등 다양합니다. 이런 속성값 전달을 `Attribute`를 통해서 HSM에게 전달하게 됩니다.

<br>

## 2. 마스터키 등록

저희가 사용한 HSM은 용량이 작아서 개인키 저장공간으로 사용하는 것은 무리였습니다. 그래서 HSM에 마스터키를 두어서 암복호화만 맡기고 암호화된 개인키는 DB에 저장하기로 했습니다. 어차피 암복호화 모두 HSM에서 수행하기 때문에 AES 대칭키를 사용했습니다.

HSM에 마스터키를 어떻게 올릴 것인지도 고민이었습니다. 가장 편리한 방법은 마스터키를 HSM에서 생성하는 것이지만 HSM에 문제가 발생했을 때에 마스터키를 분실할 수도 있고 기존 유저들의 개인키는 쓸 수 없게 되겠죠. 그래서 안전하게 마스터키를 주입하기로 했습니다. 하지만 어째서인지 저희가 쓰는 버전의 LunaClient에서는 AES 키에 대한 `createObject`를 지원하지 않았습니다. PublicKey는 지원을 해주던데 SecretKey(대칭키), PrivateKey(개인키)는 지원을 안해주는 경우가 있다고 합니다.   

굳이 비대칭키를 쓸 이유는 없는데, 안전하게 마스터키를 사용하려면 꼭 주입하는 방식을 사용해야 했습니다. 왜 그렇게 비대칭키가 쓰기 싫었던건지... 방법을 고민했고 그 결과 약간 우회하는 방식을 써서 Master Key를 등록할 수 있었습니다. 

1. HSM 내부에서 Key 하나를 생성 (KEK;Key Encrypt Key 용도)
2. KEK으로 마스터키를 암호화
3. Unwrap을 사용해서 HSM 내부에서 KEK으로 다시 마스터키를 복호화
4. 1 ~ 3 의 결과로 HSM에는 복호화된 마스터키가 등록
5. KEK을 제거

위의 로직을 사용하니 의도한대로 직접 생성한 마스터키를 HSM에 등록할 수 있었습니다.

<br>

## 3. 암복호화

암복호화 함수로는 `C_Encrypt`와 `C_Decrypt`가 있습니다. 특이한 점으로는 `encryptedData` 파라미터에 해당하는 인자를 `null` 값으로 한번 호출한 뒤에 버퍼를 잡고 함수를 재호출해서 버퍼에 데이터를 받아오는 점입니다. 버퍼에 해당하는 데이터 크기를 먼저 `LongRef`를 통해서 받아오고 크기에 맞는 버퍼를 잡아준 뒤에 데이터를 받아오기 때문입니다.

```java
LongRef longRef = new LongRef();

CryptokiEx.C_EncryptInit(session, mechanism, hKey);
CryptokiEx.C_Encrypt(session, plainText, plainText.length, null, longRef);

byte[] cipherText = new byte[(int) longRef.value]; // buffer
CryptokiEx.C_Encrypt(session, plainText, plainText.length, cipherText, longRef);
```

위의 방식이 익숙한 사람들도 있겠지만, 저는 처음보는 방식이라서 낯설게 느껴졌습니다.
이렇게 이중 호출을 하도록 설계한 이유는 다음과 같습니다. HSM은 자원이 특히 한정되어 있기 때문에 최소한의 리소스로 정확한 작업을 수행하기 위해, 출력 크기를 사전에 계산합니다. 모든 작업이 같은 데이터 크기를 리턴한다면 필요 없겠지만 암호화 알고리즘이나 패딩 방식에 따라 출력 데이터의 크기가 달라집니다. 때문에 먼저 크기를 계산하고 받아오는 방식을 사용해서 변동성을 처리합니다.


`C_Encrypt`와 `C_Decrypt`는 HSM에서 암복호화를 수행한 후 바로 값을 받아오는 함수지만, HSM에 키를 등록하지는 않습니다. HSM에서 암복호화를 수행하고 바로 키를 등록하기 위해서는 `C_WrapKey`와 `C_UnwrapKey` 함수를 사용합니다. `C_WrapKey`는 현재 암호화되지 않은 키를 HSM에서 암호화를 한 후에 등록하는 함수이고 `C_UnwrapKey`는 암호화된 키를 HSM에서 암호화한 후에 등록하는 함수입니다.

<br>

## 4. 서명

`C_Sign` 함수는 HSM에서 서명을 하고 값을 리턴해주는 함수입니다. 암복호화와 동일하게 리턴값을 받기 위해 버퍼를 잡고 이중호출하는 로직이 필요합니다. 서명 시 필요한 키도 함께 전달해주어야 하는데 값을 직접 전달하는 것이 아니라 Object Handle 객체를 전달해야 합니다. 그래서 서명 전에 Unwrap을 수행하고 `C_FindObjects` 함수로 마스터 키에 해당하는 Object Handle을 생성해주는 과정이 필요했습니다. 

저희는 이더리움을 사용해야해서 ECDSA 서명을 사용했습니다. 일반적인 키로 ECDSA 서명을 하는 것이라면 문제 없이 쉽게 구현했겠지만, 저희는 블록체인 지갑 생성 과정에서 HSM을 사용했습니다. 그래서 BIP32의 확장키 개념을 적용해야 했습니다. 저희는 HSM 내부 구현을 모르고, HSM 업체는 블록체인 지갑을 몰라서 시간이 조금 걸렸는데, 결론적으로 `xprv`를 사용하여 ECDSA 서명을 성공적으로 해낼 수 있었습니다.

<br>

## 5. 블록체인과 HSM 서명 사이의 이슈

해치웠나...! 라고 생각했지만 역시 아니었습니다. 
이더리움 체인에 트랜젝션을 올릴 때 여러가지 제약조건들을 HSM 서명이 만족시키지 못하는 경우가 발생했습니다. 
해당 이슈들을 아래에 간략히 정리해두었습니다. 만약 HSM을 통한 서명으로 이더리움 체인에 트랜잭션을 올린다면, 사용하시는 HSM이 아래의 제약조건을 만족시킬 수 있는지 확인해보시길 추천드립니다.


### EIP-2 제약조건

- Error: `invalid signature: s-values greater than secp256k1n/2 are considered invalid`
- ECDSA 서명의 `s`값이 `secp256k1n/2` 보다 큰 거래 서명은 무효로 간주 (EIP-2)
- HSM에서 s 값을 별도로 제어할 수 없다면 제약조건을 만족하는 서명이 나올 때까지 재요청

> [Stack Exchange Reference(1)](https://ethereum.stackexchange.com/questions/55245/why-is-s-in-transaction-signature-limited-to-n-21)  
> [Stack Exchange Reference(2)](https://ethereum.stackexchange.com/questions/73192/using-aws-cloudhsm-to-sign-transactions?noredirect=1&lq=1)  
> [EIP-2](https://eips.ethereum.org/EIPS/eip-2)


### Recovery ID 필요

- Error: `invalid sender: invalid transaction v, r, s values`
- HSM에서는 64 byte 서명값을 리턴하지만, 체인에 올릴때에는 recovery id까지 총 65 byte가 필요
- 이것 역시, HSM에서는 제공해줄 수 없어서 예상 id를 대입하여 트랜잭션 재요청

> [Reddit Reference](https://www.reddit.com/r/ethereum/comments/1ewcdcb/help_needed_ethereum_hsm_integration_resulting_in/)

<br>

## 프로젝트를 마치며

[업데이트 예정!]