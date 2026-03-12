# AdChain S2S 캠페인 연동 가이드

## 목차

- [1. 개요](#1-개요)
- [2. 시작 전 준비사항](#2-시작-전-준비사항)
- [3. 연동 흐름](#3-연동-흐름)
- [4. 인증](#4-인증)
- [5. API 명세](#5-api-명세)
  - [5.1 이벤트 목록 조회](#51-이벤트-목록-조회)
  - [5.2 랜딩 URL 생성](#52-랜딩-url-생성)
- [6. 포스트백 수신](#6-포스트백-수신)
- [7. 서명 검증](#7-서명-검증)
- [8. 구현 가이드](#8-구현-가이드)
- [9. 데이터 예시](#9-데이터-예시)
- [10. 테스트 가이드](#10-테스트-가이드)
- [11. 문의](#11-문의)

---

## 1. 개요

### S2S 연동이란

S2S(Server-to-Server) 연동은 매체사 서버에서 AdChain API를 직접 호출하여 CPA 캠페인을 연동하는 방식입니다. AdChain으로부터 캠페인 물량을 받아 하위 매체에 재공급하는 대대행 구조를 지원합니다.

### 연동 대상

- AdChain의 CPA 캠페인을 하위 매체에 공급하려는 매체사(네트워크 공급사)
- 매체사 서버에서 캠페인 목록을 직접 관리하고 하위 매체에 전달하는 경우
- SDK 연동이 어려운 환경

---

## 2. 시작 전 준비사항

| # | 항목 | 설명 | 담당 |
|---|------|------|------|
| 1 | 연동 요청 | AdChain 담당자에게 S2S 연동 요청 | 매체사 → AdChain |
| 2 | `pub_secret` 발급 | 매체사 등록 완료 후 키 전달 | AdChain → 매체사 |
| 3 | 콜백 URL 전달 | 포스트백을 수신할 매체사 서버 URL 전달 | 매체사 → AdChain |
| 4 | 연동 테스트 | 스테이지 환경에서 API 호출 확인 | 매체사 |

> `pub_secret`은 매체사에서 직접 생성하는 것이 아니라, 연동 요청 시 AdChain에서 발급하여 전달합니다.
> 기술 지원: developer@1self.world

---

## 3. 연동 흐름

| 단계 | 방향 | 설명 |
| --- | --- | --- |
| 1 | 매체사 → AdChain | 캠페인 목록 조회 (GET /campaign/pub) |
| 2 | 매체사 → AdChain | 랜딩 URL 요청 (POST /campaign/pub/landing-url) |
| 3 | AdChain → 매체사 | 완료 포스트백 수신 |

### 상세 설명

1. **캠페인 목록 조회**: 매체사에 할당된 활성 캠페인 이벤트 목록을 가져옵니다
2. **랜딩 URL 생성**: 최종 유저가 이벤트에 참여할 때 사용할 트래킹 URL을 생성합니다
3. **포스트백 수신**: 액션 완료 시 AdChain에서 매체사 서버로 포스트백을 전송합니다

---

## 4. 인증

### x-pub-secret 헤더

S2S API는 `x-pub-secret` 헤더를 사용하여 인증합니다.

```http
x-pub-secret: {발급받은_pub_secret}
```

### 주의사항

- `pub_secret`은 서버 환경변수로 안전하게 관리해야 합니다
- 클라이언트 코드에 절대 노출하지 마세요
- 유출 시 즉시 재발급 요청해 주세요

---

## 5. API 명세

### Base URL

| 환경 | URL |
| --- | --- |
| Production | `https://adchain-api.1self.world` |
| Staging | `https://adchain-stage-api.1self.world` |

---

### 5.1 이벤트 목록 조회

활성화된 CPA 캠페인 이벤트 목록을 조회합니다. 비활성 캠페인은 목록에서 제외되며, 일일 물량이 소진된 캠페인은 `isAvailable: false`로 표시됩니다.

**Endpoint**

```
GET /v1/api/campaign/pub
```

**Headers**

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| x-pub-secret | Y | 매체사 시크릿 키 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| platform | enum | Y | `android` 또는 `ios` |
| page | string | N | 페이지 번호 (기본: 1) |
| limit | string | N | 페이지 크기 (기본: 10) |

**Request 예시**

```bash
curl -X GET "https://adchain-api.1self.world/v1/api/campaign/pub?platform=android&page=1&limit=10" \
  -H "x-pub-secret: your_pub_secret_here"
```

**Response**

```json
{
  "events": [
    {
      "eventId": "550e8400-e29b-41d4-a716-446655440000",
      "title": "[최초실행] 레진스낵 실행하기",
      "description": "세상의 모든 재미! 숏드라마는 레진스낵에서!",
      "guide": "1. 앱을 설치합니다\n2. 앱을 실행합니다.\n3. 보상지급!",
      "notice": "신규 유저만 참여 가능합니다. 기존 설치 이력이 있는 경우 보상이 지급되지 않습니다.",
      "iconImageUrl": "https://cdn.example.com/icon.png",
      "imageUrl": "https://cdn.example.com/cover.png",
      "rewardPoints": 300,
      "packageName": "com.example.shopping",
      "platform": "android",
      "category": "",
      "labelText": "최초실행",
      "revenueType": "cpa",
      "isAvailable": true
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25,
    "totalPages": 3
  }
}
```

**Response 필드 설명**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| eventId | string | 이벤트 고유 ID (UUID) |
| title | string | 이벤트 제목 |
| description | string | 이벤트 설명 |
| guide | string | 참여 가이드 (단계별 안내) |
| notice | string | 유의사항 |
| iconImageUrl | string | 아이콘 이미지 URL |
| imageUrl | string | 커버 이미지 URL |
| rewardPoints | number | 보상 포인트 |
| packageName | string | 앱 패키지명 |
| platform | string | 플랫폼 (android/ios) |
| category | string | 카테고리 (SHOPPING, FINANCE, LIFESTYLE 등) |
| labelText | string | 액션 라벨 (회원가입, 설치, 카드발급 등) |
| revenueType | string | 수익 모델 (`cpa`, '...') |
| isAvailable | boolean | 참여 가능 여부 (일일 물량 소진 시 `false`) |

---

### 5.2 랜딩 URL 생성

최종 유저가 이벤트에 참여할 때 트래킹 URL을 생성합니다. 이 URL로 최종 유저를 리다이렉트하면 설치/액션이 추적됩니다.

**Endpoint**

```
POST /v1/api/campaign/pub/landing-url
```

**Headers**

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| x-pub-secret | Y | 매체사 시크릿 키 |
| Content-Type | Y | application/json |

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| eventId | string | Y | 이벤트 ID (UUID) |
| userId | string | Y | 최종 유저 식별자 |
| platform | enum | Y | `android` 또는 `ios` |
| ifa | string | **Y** | 광고 식별자 (GAID/IDFA) |
| extra1 | string | N | 매체사 자유 필드 1 — 포스트백 시 그대로 반환 |
| extra2 | string | N | 매체사 자유 필드 2 — 포스트백 시 그대로 반환 |
| extra3 | string | N | 매체사 자유 필드 3 — 포스트백 시 그대로 반환 |
| extra4 | string | N | 매체사 자유 필드 4 — 포스트백 시 그대로 반환 |
| extra5 | string | N | 매체사 자유 필드 5 — 포스트백 시 그대로 반환 |

> **중요**: `ifa`는 **필수** 파라미터입니다. CPA 캠페인의 정확한 어트리뷰션을 위해 반드시 전달해야 합니다.

> **extra1~5**: 매체사가 내부 추적 용도로 자유롭게 사용할 수 있는 페이로드 필드입니다. 랜딩 URL 생성 시 전달한 값이 포스트백에 그대로 포함되어 반환됩니다.

**Request 예시**

```bash
curl -X POST "https://adchain-api.1self.world/v1/api/campaign/pub/landing-url" \
  -H "x-pub-secret: your_pub_secret_here" \
  -H "Content-Type: application/json" \
  -d '{
    "eventId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_12345",
    "platform": "android",
    "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
    "extra1": "sub_media_001",
    "extra2": "campaign_group_A"
  }'
```

**Response**

```json
{
  "success": true,
  "landingUrl": "https://adchain-track.1self.world/click?campaign=abc123&user_id=user_12345&gaid=38400000-8cf0-11bd-b23e-10b96e40000d"
}
```

**Response 필드 설명**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| success | boolean | 성공 여부 |
| landingUrl | string | 트래킹이 적용된 랜딩 URL (유저를 이 URL로 리다이렉트) |

---

## 6. 포스트백 수신

최종 유저가 이벤트 액션을 완료하면 AdChain에서 매체사 서버로 포스트백을 전송합니다.

### 포스트백 수신 URL 설정

포스트백을 받을 Endpoint를 AdChain 팀에 전달해 주세요. (developer@1self.world)

### 요청 정보

- **Method**: `POST`
- **Content-Type**: `application/json`

### Request Body

```json
{
  "callback_id": "b6fcca4e-e7b8-4a70-94fd-810b1b6a256b",
  "type": "campaign",
  "revenue_type": "cpa",
  "user_id": "user_12345",
  "amount": "1500",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_name": "[최초실행] 레진스낵 실행하기",
  "signed_value": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "extra1": "sub_media_001",
  "extra2": "campaign_group_A"
}
```

### 필드 설명

| 필드명 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| callback_id | string | Y | 고유 트랜잭션 ID (UUID) - 중복 처리 방지용 |
| type | string | Y | 콘텐츠 유형 (`campaign`: 캠페인 완료) |
| revenue_type | string | Y | 수익화 모델 (`cpa`: 액션 완료) |
| user_id | string | Y | 최종 유저 식별자 |
| amount | string | Y | 보상 포인트 |
| event_id | string | Y | 이벤트 ID (eventId) |
| event_name | string | N | 이벤트 제목 |
| signed_value | string | Y | HMAC-MD5 서명값 |
| os | string | N | 운영체제 (android/ios) |
| ifa | string | N | 광고 식별자 (GAID/IDFA) |
| app_key | string | N | 앱 식별 키 (서명 검증 시 app_secret 선택에 사용) |
| extra1 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 1 |
| extra2 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 2 |
| extra3 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 3 |
| extra4 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 4 |
| extra5 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 5 |

### 응답 처리

#### 성공 응답

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "message": "Postback received"
}
```

#### 실패 응답

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "success": false,
  "message": "Invalid user_id"
}
```

#### 응답 필수 필드

| 필드명 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| success | boolean | Y | 처리 성공 여부 |
| message | string | Y | 응답 메시지 (빈 문자열 가능) |

---

## 7. 서명 검증

### 서명 검증 필요성

포스트백의 `signed_value`를 검증하여 요청이 AdChain으로부터 온 유효한 요청인지 확인해야 합니다.

### 서명 생성 방식

1. **서명 대상 문자열 생성**:

```
plainText = callback_id + user_id + amount + event_id
```

2. **HMAC-MD5 해시 생성**:
   - 알고리즘: HMAC-MD5
   - Secret Key: AdChain에서 발급한 `app_secret`
   - 결과: 32자리 16진수 문자열 (소문자)

### Secret Key 관리

`app_key` 또는 `os`에 따라 다른 `app_secret`을 사용합니다:

```javascript
// app_key 기반 (권장)
const APP_SECRETS = {
  100000001: process.env.ANDROID_APP_SECRET,
  100000002: process.env.IOS_APP_SECRET,
};

// os 기반
const OS_SECRETS = {
  android: process.env.ANDROID_DEFAULT_SECRET,
  ios: process.env.IOS_DEFAULT_SECRET,
};
```

### 검증 구현 예제 (JavaScript)

```javascript
const crypto = require('crypto');

// Secret Key 설정
const APP_SECRETS = {
  100000001: process.env.ANDROID_APP_SECRET,
  100000002: process.env.IOS_APP_SECRET,
};

const OS_SECRETS = {
  android: process.env.ANDROID_DEFAULT_SECRET,
  ios: process.env.IOS_DEFAULT_SECRET,
};

function getAppSecret(payload) {
  // app_key가 있는 경우 app별 secret 사용
  if (payload.app_key && APP_SECRETS[payload.app_key]) {
    return APP_SECRETS[payload.app_key];
  }
  // OS별 secret 사용
  return OS_SECRETS[payload.os] || OS_SECRETS['android'];
}

function validateSignature(payload) {
  const { callback_id, user_id, amount, event_id, signed_value } = payload;

  // 적절한 app_secret 선택
  const appSecret = getAppSecret(payload);

  // 서명 대상 문자열 생성
  const plainText = `${callback_id}${user_id}${amount}${event_id}`;

  // HMAC-MD5 해시 생성
  const hmac = crypto.createHmac('md5', appSecret);
  hmac.update(plainText);
  const generatedValue = hmac.digest('hex');

  // 서명 검증
  return generatedValue === signed_value;
}

// 사용 예시
app.post('/postback', (req, res) => {
  const payload = req.body;

  if (!validateSignature(payload)) {
    return res.status(401).json({
      success: false,
      message: 'Invalid signature',
    });
  }

  // 서명 검증 성공 - 포스트백 처리 진행
  // ...
});
```

### 주의사항

1. **app_secret 보안**: 절대 외부에 노출하지 마세요
2. **검증 필수**: 모든 포스트백에 대해 서명 검증을 수행하세요
3. **검증 실패 시**: HTTP 401 응답을 반환하세요
4. **문자열 순서**: `callback_id → user_id → amount → event_id` 순서가 중요합니다

---

## 8. 구현 가이드

### 중복 처리 방지 (필수)

`callback_id`를 사용하여 중복 처리를 방지해야 합니다.

```javascript
async function processPostback(payload) {
  const { callback_id } = payload;

  // 1. 중복 체크 (필수)
  const isDuplicate = await checkDuplicateCallback(callback_id);
  if (isDuplicate) {
    console.log(`[중복 요청] callback_id: ${callback_id} - 처리 스킵`);
    return { success: true, message: 'Already processed' };
  }

  // 2. callback_id 저장
  await saveCallbackId(callback_id);

  // 3. 매체사 내부 처리 로직
  // ...

  return { success: true, message: 'Postback processed' };
}
```

### 에러 처리

```javascript
app.post('/campaign-postback', async (req, res) => {
  try {
    const payload = req.body;

    // 1. 서명 검증
    if (!validateSignature(payload)) {
      return res.status(401).json({
        success: false,
        message: 'Invalid signature',
      });
    }

    // 2. 필수 필드 검증
    if (!payload.callback_id || !payload.user_id || !payload.amount) {
      return res.status(400).json({
        success: false,
        message: 'Missing required fields',
      });
    }

    // 3. 포스트백 처리
    const result = await processPostback(payload);
    return res.status(200).json(result);
  } catch (error) {
    console.error('Postback processing error:', error);
    return res.status(500).json({
      success: false,
      message: 'Internal server error',
    });
  }
});
```

---

## 9. 데이터 예시

### CPA 앱 설치 포스트백

```json
{
  "callback_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "campaign",
  "revenue_type": "cpa",
  "user_id": "user_12345",
  "amount": "800",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_name": "배달 앱 설치",
  "signed_value": "7f8a9b2c3d4e5f6a1b2c3d4e5f6a7b8c",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "extra1": "sub_media_001",
  "extra2": "campaign_group_A"
}
```

### CPA 액션 완료 포스트백

```json
{
  "callback_id": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "type": "campaign",
  "revenue_type": "cpa",
  "user_id": "user_12345",
  "amount": "3000",
  "event_id": "660f9511-f30c-52e5-b827-557766551111",
  "event_name": "증권 앱 계좌 개설",
  "signed_value": "9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "extra1": "sub_media_002",
  "extra3": "promo_2026Q1"
}
```

---

## 10. 테스트 가이드

### 테스트 환경

**실제 서비스 적용 전 AdChain 개발팀과 함께 테스트를 진행해 주세요.**

1. Staging 환경 URL: `https://adchain-stage-api.1self.world`
2. 테스트용 `pub_secret` 발급 요청
3. 포스트백 수신 Endpoint 전달

### 테스트 시나리오

| 단계 | 테스트 항목 | 확인 사항 |
| --- | --- | --- |
| 1 | 이벤트 목록 조회 | API 응답 정상 수신, isAvailable 필드 확인 |
| 2 | 랜딩 URL 생성 | landingUrl 정상 생성, ifa 필수 전달 확인 |
| 3 | 포스트백 수신 | 서명 검증 성공 |
| 4 | 중복 처리 | 동일 callback_id 중복 요청 시 스킵 |
| 5 | 에러 처리 | 잘못된 서명 시 401 응답 |
| 6 | extra 필드 검증 | 랜딩 URL에 전달한 extra 값이 포스트백에 그대로 반환되는지 확인 |

### 테스트 요청

테스트 관련 모든 문의는 developer@1self.world로 연락 주시면 빠르게 지원해 드리겠습니다.

---

## 11. 문의

연동 관련 문의사항이 있으시면 아래 채널로 연락 주시기 바랍니다:

- **기술 지원**: developer@1self.world

---
