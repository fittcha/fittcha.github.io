---
title: "OAuth 2.0부터 HMAC까지 실무 인증 패턴 정리"
date: 2025-12-10 12:00:00 +0900
categories: [Development, Auth]
tags: [인증, OAuth, JWT, HMAC, Basic Auth]
---

1편에서 세션과 토큰 개념을 정리했는데요, 이번에는 실무에서 실제로 사용하는 인증 패턴들을 정리해보겠습니다.

OAuth 2.0, Basic Auth, HMAC... 정확히 파악하지 못한 채 일해온 사람.. 저뿐이지는 않겠죠//_//

---

## OAuth 2.0

### OAuth가 뭔가요?

**Open Authorization**의 약자로, 제3자 앱에 **비밀번호 없이 권한을 위임**하는 표준 프로토콜입니다.

### 왜 필요한가요?

OAuth 없던 시절을 생각해봅시다.
```
[사용자] "저 앱에서 내 카카오 친구 목록 가져오고 싶어"
[앱] "그러면 카카오 ID/PW 알려주세요"
[사용자] "여기요... abc@kakao.com / mypassword123"
[앱] (카카오 로그인해서 친구 목록 가져옴)
```

문제점이 보이죠?
- 앱이 내 카카오 비밀번호를 알게 됨
- 앱이 카카오에서 뭘 해도 모름 (메일 읽기, 삭제 등)
- 권한 취소하려면 비밀번호를 바꿔야 함

OAuth 방식은 이렇습니다:
```
[사용자] "저 앱에서 내 카카오 친구 목록 가져오고 싶어"
[앱] "카카오한테 허락받아오세요"
[사용자] → [카카오] "앱이 친구 목록 요청하는데 허락할까요?"
[사용자] "허락!"
[카카오] → [앱] "여기 토큰. 친구 목록만 볼 수 있어요"
[앱] (토큰으로 친구 목록만 조회)
```

장점:
- 앱은 비밀번호 모름
- 친구 목록만 허용, 다른 건 못함
- 카카오에서 언제든 권한 취소 가능

### OAuth 2.0 용어 정리

| 용어 | 설명 | 예시 |
|------|------|------|
| **Resource Owner** | 리소스 주인 (사용자) | 카카오 회원인 나 |
| **Client** | 리소스 접근하려는 앱 | 우리 서비스 |
| **Authorization Server** | 인증/토큰 발급 서버 | 카카오 인증 서버 |
| **Resource Server** | 실제 리소스 보유 서버 | 카카오 API 서버 |

### Grant Types (인증 방식)

OAuth 2.0에는 4가지 표준 방식이 있어요.

| Grant Type | 사용 케이스 | 보안 수준 |
|------------|-------------|-----------|
| **Authorization Code** | 웹 서버 앱 (가장 많이 사용) | 높음 |
| **PKCE** | 모바일/SPA (Authorization Code 확장) | 높음 |
| **Client Credentials** | 서버 간 통신 (사용자 없음) | 중간 |
| **Implicit** | ~~SPA용~~ (현재 비권장) | 낮음 |

### Authorization Code Grant (가장 중요!)

카카오 로그인, 네이버 로그인이 이 방식입니다. 가장 많이 쓰이는 플로우예요.
```
[1] 사용자가 "카카오 로그인" 클릭

[2] 카카오 인증 페이지로 리다이렉트
    https://kauth.kakao.com/oauth/authorize
    ?client_id=xxx
    &redirect_uri=https://myservice.com/callback
    &response_type=code

[3] 사용자가 카카오에서 로그인 + 권한 동의

[4] Authorization Code와 함께 우리 서비스로 리다이렉트
    https://myservice.com/callback?code=AUTH_CODE_HERE

[5] 서버에서 code + client_secret으로 토큰 요청
    POST /oauth/token
    { code, client_id, client_secret, redirect_uri }

[6] Access Token + Refresh Token 발급
    { access_token, refresh_token, expires_in }

[7] 토큰으로 사용자 정보 요청
    GET /v2/user/me
    Authorization: Bearer {access_token}
```

**왜 Authorization Code를 거칠까요?**

"그냥 바로 Access Token 주면 안 되나요?" 라고 생각할 수 있는데요.

브라우저 리다이렉트로 토큰 전달하면 URL에 노출됩니다. 브라우저 히스토리나 로그에 토큰이 남게 돼요. 탈취 위험이 있죠.

그래서 Authorization Code를 먼저 받고, 서버 간 통신으로 토큰을 교환하는 방식입니다. Code가 노출돼도 client_secret 없이는 토큰 교환이 불가능하거든요.

### PKCE (모바일/SPA용)

모바일 앱이나 SPA는 **client_secret을 안전하게 보관할 수 없습니다.**

- 모바일 앱 → APK 디컴파일하면 노출
- SPA → 브라우저 소스에서 노출

PKCE는 이 문제를 해결해요:
```
[1] 클라이언트가 랜덤 값 생성
    code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

[2] 해시해서 code_challenge 생성
    code_challenge = BASE64URL(SHA256(code_verifier))

[3] 인증 요청 시 code_challenge 전송

[4] 토큰 요청 시 code_verifier 전송

[5] 서버가 검증
    SHA256(code_verifier) == code_challenge? → OK!
```

핵심은 code_verifier를 알아야 토큰 교환이 가능하다는 점. Authorization Code가 탈취돼도 소용없어요.

### Client Credentials Grant

사용자 없이 **서버 간 통신**할 때 사용하는 방식입니다.
```
[우리 서버] → [외부 API]
"나 OOO 서비스야, 데이터 줘"
```

사용자 개입 없이 Client ID + Client Secret으로 바로 토큰을 받습니다. 배치 작업이나 서버 간 API 연동에 많이 쓰여요.

---

## Basic Authentication

가장 단순한 HTTP 표준 인증 방식. 결제 API에서 많이 씁니다.

### 동작 원리
```
[1] 인증 정보 준비
    ID: live_sk_xxx
    Password: (보통 비어있거나 동일)

[2] Base64 인코딩
    "live_sk_xxx:" → "bGl2ZV9za194eHg6"

[3] HTTP 요청
    GET /v1/payments
    Authorization: Basic bGl2ZV9za194eHg6

[4] 서버에서 검증
    Base64 디코딩 → ID:Password 추출 → 유효성 검증
```

### 예시 코드
```java
String secretKey = "live_sk_xxxxxxx";
String encodedKey = Base64.getEncoder()
                          .encodeToString((secretKey + ":").getBytes());

HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestProperty("Authorization", "Basic " + encodedKey);
```

###️ 주의사항

| 주의사항 | 이유 |
|----------|------|
| Base64는 암호화가 아님 | 누구나 디코딩 가능 |
| HTTPS 필수 | 평문 전송과 동일 |
| 매 요청마다 인증 정보 전송 | 노출 기회 많음 |

Basic Auth는 단순하지만 HTTPS 없이 쓰면 위험합니다. 꼭 HTTPS와 함께 사용하세요.

---

## API Key 인증

가장 간단한 방식. 발급받은 키를 요청에 포함해서 보내면 끝입니다.

### 전달 방식

| 방식 | 예시 | 보안성 |
|------|------|--------|
| **Header** | `X-API-Key: abc123` | 권장 |
| **Query Param** | `/api?apiKey=abc123` | URL에 노출 |
| **Body** | `{ "apiKey": "abc123" }` | 보통 |

### 예시 코드
```java
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestProperty("X-API-Key", apiKey);
```

### API Key vs OAuth 2.0

| 항목 | API Key | OAuth 2.0 |
|------|---------|-----------|
| **복잡도** | 단순 | 복잡 |
| **권한 범위** | 전체 또는 없음 | Scope로 세분화 가능 |
| **만료** | 보통 없음 | Access Token 자동 만료 |
| **사용 케이스** | 서버 간 통신, 내부 API | 사용자 권한 위임 |

간단한 서버 간 통신에는 API Key, 사용자 권한 위임이 필요하면 OAuth 2.0이 적합해요.

---

## HMAC (Hash-based Message Authentication Code)

비밀키를 네트워크로 전송하지 않는 방식입니다. 채팅 서비스나 웹훅에서 많이 써요.

### 동작 원리
```
[클라이언트]
메시지: "userId=12345"
비밀키: "secret_key_abc"
         ↓
HMAC-SHA256(메시지, 비밀키)
         ↓
해시값: "a1b2c3d4e5f6..."

→ 서버로 전송: 메시지 + 해시값 (비밀키는 안 보냄!)

[서버]
받은 메시지로 같은 방식으로 해시 생성
         ↓
받은 해시값 == 계산한 해시값?
         ↓
일치하면 인증 성공!
```

### 예시 코드
```java
public static String generateHmac(String message, String secretKey) {
    Mac sha256Hmac = Mac.getInstance("HmacSHA256");
    SecretKeySpec keySpec = new SecretKeySpec(
        secretKey.getBytes(StandardCharsets.UTF_8), 
        "HmacSHA256"
    );
    sha256Hmac.init(keySpec);
    
    byte[] hash = sha256Hmac.doFinal(message.getBytes(StandardCharsets.UTF_8));
    return Hex.encodeHexString(hash);
}
```

### HMAC의 장점

| 장점 | 설명 |
|------|------|
| **비밀키 미전송** | 네트워크에서 탈취 불가 |
| **무결성 검증** | 메시지 변조 감지 가능 |
| **재사용 공격 방지** | timestamp 포함 시 |

Basic Auth보다 안전해요. 비밀키가 네트워크에 노출되지 않으니까요.

---

## SSO (Single Sign-On)

한 번 로그인으로 여러 시스템에 접근하는 방식. 회사 내부 시스템에서 많이 쓰입니다.

### SSO 없을 때 vs 있을 때
```
[SSO 없을 때]
시스템 A 로그인 → ID/PW 입력
시스템 B 로그인 → ID/PW 또 입력
시스템 C 로그인 → ID/PW 또또 입력
→ 귀찮음, 비밀번호 관리 복잡

[SSO 있을 때]
SSO 서버 로그인 → ID/PW 한 번만 입력
시스템 A 접근 → 자동 인증
시스템 B 접근 → 자동 인증
시스템 C 접근 → 자동 인증
→ 편리함, 중앙 집중 관리
```

### 동작 플로우
```
[사용자] → [우리 서비스] → [SSO 서버]
                              ↓
                         통합 계정으로 로그인
                              ↓
                    인증 토큰 + 사용자 정보 반환
                              ↓
[우리 서비스] ← 토큰 검증 후 세션 생성
```

회사 내부 시스템이나 그룹사 서비스 통합에 많이 사용돼요.

---

## 인증 방식 비교 총정리

| 방식 | 복잡도 | 보안 | 사용 케이스 |
|------|--------|------|-------------|
| **Basic Auth** | ⭐ | ⭐⭐ | 결제 API |
| **API Key** | ⭐ | ⭐⭐ | 간단한 외부 연동 |
| **HMAC** | ⭐⭐ | ⭐⭐⭐ | 무결성 중요한 경우 |
| **OAuth 2.0** | ⭐⭐⭐ | ⭐⭐⭐⭐ | SNS 로그인, 권한 위임 |
| **JWT** | ⭐⭐ | ⭐⭐⭐ | Stateless 인증 |
| **SSO** | ⭐⭐⭐ | ⭐⭐⭐⭐ | 사내 시스템 통합 |

---

## 실무에서 선택 기준

어떤 인증 방식을 써야 할지 고민될 때 참고하세요.
```
사용자 권한 위임이 필요해?
→ OAuth 2.0

서버 간 통신이야?
→ API Key 또는 Client Credentials

메시지 무결성이 중요해?
→ HMAC

레거시 시스템이랑 연동해야 해?
→ Basic Auth (HTTPS 필수!)

사내 여러 시스템 통합 로그인?
→ SSO

Stateless + 확장성이 필요해?
→ JWT
```

한 프로젝트에서 여러 방식을 혼용하는 경우도 많아요. 결제 API는 Basic Auth, SNS 로그인은 OAuth 2.0, 내부 통신은 API Key 이런 식으로요. 상황에 맞게 선택하면 됩니다.

---

## 정리

- **OAuth 2.0**: 제3자에게 비밀번호 없이 권한 위임. SNS 로그인의 표준
- **Basic Auth**: ID:Password를 Base64 인코딩. 단순하지만 HTTPS 필수
- **API Key**: 가장 단순한 방식. 서버 간 통신에 적합
- **HMAC**: 비밀키 미전송. 무결성 검증 가능
- **SSO**: 한 번 로그인으로 여러 시스템 접근

다음 편에서는 실제로 인증을 적용해보면서 정리하겠습니다.

---

*이 글은 인증 시리즈 2편입니다.*
- 1편: 세션 vs 토큰, 알고 사용하기
- 2편: OAuth 2.0부터 HMAC까지 실무 인증 패턴 정리 (현재 글)
- 3편: [try] 인증 실제로 적용해보기
