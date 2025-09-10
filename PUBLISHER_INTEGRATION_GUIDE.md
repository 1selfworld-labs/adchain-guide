# 1SelfWorld AdChain 퍼블리셔 포스트백 연동 가이드

## 목차

- [개요](#개요)
- [API 명세](#api-명세)
  - [요청 정보](#요청-정보)
  - [요청 본문 (Request Body)](#요청-본문-request-body)
  - [필드 설명](#필드-설명)
  - [응답 처리](#응답-처리)
- [보안 및 검증](#보안-및-검증)
  - [서명 검증 (Signature Validation)](#서명-검증-signature-validation)
  - [Secret Key 관리](#secret-key-관리)
  - [검증 구현 예제](#검증-구현-예제)
  - [주의사항](#주의사항)
- [구현 가이드](#구현-가이드)
  - [중복 처리 방지 (필수)](#중복-처리-방지-필수)
  - [푸시 알림 구성 가이드](#푸시-알림-구성-가이드)
- [데이터 타입별 예시](#데이터-타입별-예시)
- [테스트 가이드](#테스트-가이드)
  - [테스트 환경 구성](#테스트-환경-구성)
  - [테스트 시나리오](#테스트-시나리오)
- [보안 고려사항](#보안-고려사항)
- [문의사항](#문의사항)
- [변경 이력](#변경-이력)

## 개요

1SelfWorld AdChain에서는 퍼블리셔에게 사용자의 광고 참여 완료 정보를 실시간으로 전달하기 위해 포스트백(Postback) 시스템을 제공합니다.

아래 가이드를 따라 진행하시면 포스트백을 연동하실 수 있으며, 연동 과정에서 기술적인 문의사항이나 도움이 필요하신 경우 언제든지 개발팀(developer@1self.world)으로 문의해 주시면 신속하게 답변 드리겠습니다.

## API 명세

### 요청 정보

- **Method**: `POST`
- **Content-Type**: `application/json`
- **Endpoint**: 퍼블리셔가 제공한 포스트백 수신 URL

### 요청 본문 (Request Body)

```json
{
  "callback_id": "b6fcca4e-e7b8-4a70-94fd-810b1b6a256b",
  "type": "campaign",
  "revenue_type": "cpa",
  "user_id": "ab0da900-7465-4231-8657-1ef40944a8a2",
  "amount": "100",
  "campaign_key": "12352221",
  "campaign_name": "[초간단] 이마트 24 구독하기",
  "signed_value": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "app_key": "100000001",
  "os": "android",
  "ifa": "9ee20401-14bf-4569-a8d3-dc577be8d07f"
}
```

### 필드 설명

| 필드명          | 타입   | 필수 여부 | 설명                                                                                                                                                                                                                                                     | 예시                                   |
| --------------- | ------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `callback_id`   | String | Required  | 고유 트랜잭션 ID (UUID)<br>중복 지급 방지용                                                                                                                                                                                                              | `b6fcca4e-e7b8-4a70-94fd-810b1b6a256b` |
| `type`          | String | Required  | 콘텐츠/활동 유형<br>- `campaign`: 광고 캠페인 참여<br>- `mission`: 미션 완료 보상<br>- `quiz`: 퀴즈 참여 보상                                                                                                                                            | `campaign`                             |
| `revenue_type`  | String | Required  | 수익화 모델<br>- `cpa`: 일반 참여형<br>- `cpq`: 퀴즈형 광고<br>- `cps`: 쇼핑/구매형<br>- `cpx`: 설문형<br>- `cpi`: 앱 설치형<br>- `none`: -                                                                                                              | `cpa`                                  |
| `user_id`       | String | Required  | 퍼블리셔 유저 식별값                                                                                                                                                                                                                                     | `ab0da900-7465-4231-8657-1ef40944a8a2` |
| `amount`        | String | Required  | 사용자 보상 금액                                                                                                                                                                                                                                         | `100`                                  |
| `campaign_key`  | String | Required  | 캠페인 고유 키<br>퀴즈의 경우 이벤트 ID 사용                                                                                                                                                                                                             | `12352221`                             |
| `campaign_name` | String | Optional  | 캠페인 명칭                                                                                                                                                                                                                                              | `[초간단] 이마트 24 구독하기`          |
| `signed_value`  | String | Required  | HMAC-MD5 서명값<br>데이터 무결성 검증용                                                                                                                                                                                                                  | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`     |
| `app_key`       | String | Optional  | 앱 식별자<br>멀티 앱 구분용<br>**OS별로 구분되는 고유 키**                                                                                                                                                                                               | `100000001`                            |
| `os`            | String | Optional  | 운영체제 타입<br>- `android`<br>- `ios`                                                                                                                                                                                                                  | `android`                              |
| `ifa`           | String | Optional  | 광고 식별값 (Identifier for Advertisers)<br>OS에 따라 다른 값이 전달됩니다:<br>- Android 앱: GAID/ADID 값<br>- iOS 앱: IDFA 값<br><br>하나의 앱은 단일 OS에서만 동작하며, `app_key`를 통해 어떤 OS의 앱인지 구분 가능하므로 단일 필드로 통합 제공됩니다. | `9ee20401-14bf-4569-a8d3-dc577be8d07f` |

### 응답 처리

#### 응답 형식 (필수)

모든 응답은 반드시 다음 필드를 포함해야 합니다:

| 필드명    | 타입    | 필수 여부 | 설명                            |
| --------- | ------- | --------- | ------------------------------- |
| `success` | Boolean | Required  | 처리 성공 여부 (true/false)     |
| `message` | String  | Required  | 응답 메시지 (빈 문자열 "" 가능) |

#### 성공 응답

퍼블리셔는 포스트백을 정상적으로 수신하고 처리했을 경우, HTTP 상태 코드 `200 OK` 또는 `201 Created`를 반환해야 합니다.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "success": true,
    "message": "Postback received"
}
```

#### 실패 응답

처리 실패 시 적절한 HTTP 상태 코드와 에러 메시지를 반환 부탁드리겠습니다.

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
    "success": false,
    "message": "Invalid user_id"
}
```

## 보안 및 검증

### 서명 검증 (Signature Validation)

데이터 무결성과 보안을 위해 모든 포스트백에는 `signed_value` 필드가 포함됩니다. 퍼블리셔는 이 값을 검증하여 요청이 AdChain으로부터 온 유효한 요청인지 확인이 필요합니다.

#### 서명 생성 방식

1. **서명 대상 문자열 생성**:

   ```
   plainText = callback_id + user_id + amount + campaign_key
   ```

2. **HMAC-MD5 해시 생성**:
   - 알고리즘: HMAC-MD5
   - Secret Key: AdChain에서 발급한 `app_secret` (OS 또는 app_key 별로 구분)
   - 결과: 32자리 16진수 문자열 (소문자)

### Secret Key 관리

퍼블리셔는 app_key와 os중에 선택하여 다른 `app_secret`을 사용해야 합니다.

1. **App Key 기반 구분 (권장)**: `app_key` 필드가 제공된 경우, 해당 앱에 맞는 secret 사용
2. **OS 기반 구분**: `os` 필드 값에 따라 Android용 secret과 iOS용 secret을 구분하여 사용

```javascript
function getAppSecretByAppKey(payload) {
  const APP_SECRETS = {
    100000001: process.env.ANDROID_APP_SECRET,
    100000002: process.env.IOS_APP_SECRET,
  };

  return APP_SECRETS[payload.app_key];
}

function getAppSecretByOs(payload) {
  const OS_SECRETS = {
    android: process.env.ANDROID_DEFAULT_SECRET,
    ios: process.env.IOS_DEFAULT_SECRET,
  };

  return payload.os === 'ios' ? IOS_APP_SECRET : ANDROID_APP_SECRET;
}
```

### 검증 구현 예제

```javascript
const crypto = require('crypto');

// 환경변수에서 OS별/앱별 Secret 관리
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
  const { callback_id, user_id, amount, campaign_key, signed_value } = payload;

  // 적절한 app_secret 선택
  const appSecret = getAppSecret(payload);

  // 서명 대상 문자열 생성
  const plainText = `${callback_id}${user_id}${amount}${campaign_key}`;

  // HMAC-MD5 해시 생성
  const hmac = crypto.createHmac('md5', appSecret);
  hmac.update(plainText);
  const generatedValue = hmac.digest('hex');

  // 서명 검증
  return generatedValue === signed_value;
}

// 사용 예시
const payload = {
  callback_id: 'b6fcca4e-e7b8-4a70-94fd-810b1b6a256b',
  user_id: 'ab0da900-7465-4231-8657-1ef40944a8a2',
  amount: '100',
  campaign_key: '12352221',
  signed_value: '5d8b2a9f1c3e7b4a6d9f2e8c1a5b3d7f',
  os: 'android',
  app_key: '100000001',
};

if (validateSignature(payload)) {
  console.log('✅ 서명 검증 성공 - 유효한 요청입니다.');
} else {
  console.log('❌ 서명 검증 실패 - 요청을 거부해야 합니다.');
}
```

### 주의사항

1. **app_secret 보안**: app_secret은 절대 외부에 노출되어서는 안 됩니다
2. **검증 필수**: 모든 포스트백 요청에 대해 서명 검증을 반드시 수행해야 합니다
3. **검증 실패 시**: 서명 검증이 실패한 요청은 반드시 거부해야 합니다 (HTTP 401 응답)
4. **문자열 순서**: 서명 생성 시 필드 순서가 중요합니다 (callback_id → user_id → amount → campaign_key)
5. **Secret 관리**: OS별 또는 앱별로 다른 secret을 사용하므로 환경변수 등을 통해 안전하게 관리해야 합니다
6. **우선순위**: app_key가 제공된 경우 해당 앱의 secret을 우선 사용하고, 없으면 OS별 기본 secret을 사용합니다

## 구현 가이드

### 중복 처리 방지 (필수)

사용자 보상이 직접적으로 연결되어 있기 때문에, 중복 처리 방지 로직을 반드시 구현이 필요합니다.

#### 핵심 구현 요구사항

1. **고유 식별자 관리**

   - `callback_id`는 각 트랜잭션의 고유 식별자(UUID)입니다
   - 동일한 `callback_id`가 여러 번 수신될 경우, **반드시 첫 번째 요청만 처리**하고 이후 요청은 무시해야 합니다

2. **구현 예시**

   ```javascript
   // 중복 처리 방지 로직 예시
   async function processPostback(payload) {
     const { callback_id, user_id, amount } = payload;

     // 1. 중복 체크 (필수)
     const isDuplicate = await checkDuplicateCallback(callback_id);
     if (isDuplicate) {
       console.log(`[중복 요청] callback_id: ${callback_id} - 처리 스킵`);
       return { success: true, message: 'Already processed' };
     }

     // 2. callback_id 저장
     await saveCallbackId(callback_id);

     // 3. 실제 포인트 지급 처리
     await grantPointsToUser(user_id, amount);

     return { success: true, message: 'Postback processed' };
   }
   ```

### 푸시 알림 구성 가이드

사용자에게 리워드 지급 푸시 알림을 보낼 때는 `campaign_name`과 `amount` 필드를 활용하여 메시지를 구성하는 것을 권장합니다.

#### 푸시 메시지 예시

```
// 일반 캠페인
"[초간단] 이마트 24 구독하기 참여 완료! 100포인트가 지급되었습니다."

// 미션 완료
"3회 미션 완료 보상으로 500포인트가 지급되었습니다."

// 퀴즈 참여
"일일 상식 퀴즈 참여 보상 50포인트가 지급되었습니다."
```

#### 구현 예시

```javascript
function createPushMessage(payload) {
  const { campaign_name, amount } = payload;

  // campaign_name이 있는 경우
  if (campaign_name) {
    return `${campaign_name} 참여 완료! ${amount}포인트가 지급되었습니다.`;
  }

  // campaign_name이 없는 경우 type 기반 메시지
  switch (payload.type) {
    case 'mission':
      return `미션 완료 보상 ${amount}포인트가 지급되었습니다.`;
    case 'quiz':
      return `퀴즈 참여 보상 ${amount}포인트가 지급되었습니다.`;
    default:
      return `${amount}포인트가 지급되었습니다.`;
  }
}
```

## 데이터 타입별 예시

### 1. CPA 광고 캠페인 (일반 참여형)

```json
{
  "callback_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "campaign",
  "revenue_type": "cpa",
  "user_id": "ab0da900-7465-4231-8657-1ef40944a8a2",
  "os": "android",
  "ifa": "abc123def456",
  "amount": "150",
  "campaign_key": "camp_001",
  "campaign_name": "[초간단] 이마트 24 구독하기",
  "signed_value": "7f8a9b2c3d4e5f6a1b2c3d4e5f6a7b8c",
  "app_key": "100000001"
}
```

### 2. 미션 완료 보상

```json
{
  "callback_id": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "type": "mission",
  "revenue_type": "none",
  "user_id": "ab0da900-7465-4231-8657-1ef40944a8a2",
  "os": "ios",
  "ifa": "xyz789abc123",
  "amount": "500",
  "campaign_key": "mission_daily",
  "campaign_name": "3회 미션 완료 보상",
  "signed_value": "9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a",
  "app_key": "100000002"
}
```

### 3. 퀴즈 참여 보상

```json
{
  "callback_id": "c3d4e5f6-a7b8-9012-cdef-345678901234",
  "type": "quiz",
  "revenue_type": "none",
  "user_id": "user_345678",
  "os": "android",
  "ifa": "def456ghi789",
  "amount": "50",
  "event_id": "quiz_2024_01",
  "campaign_key": "quiz_2024_01",
  "campaign_name": "일일 상식 퀴즈",
  "signed_value": "2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e",
  "app_key": "100000002"
}
```

## 테스트 가이드

### 테스트 환경 구성

**실제 서비스 적용 전 1SelfWorld 개발팀과 함께 테스트를 진행해 주세요.**

테스트 포스트백을 받기 위한 Endpoint를 알려주시면 테스트 진행을 도와드리겠습니다.

### 테스트 시나리오

1. **기본 포스트백 수신 테스트**: 정상적인 포스트백 처리 확인
2. **중복 처리 테스트**: 동일한 `callback_id`로 반복 테스트가 필요하신 경우, 개발팀에 요청해 주시면 중복 포스트백을 보내드립니다
3. **서명 검증 테스트**: 올바른 서명과 잘못된 서명 처리 확인
4. **에러 처리 테스트**: 잘못된 데이터 형식 처리 확인

테스트 관련 모든 문의사항은 developer@1self.world로 연락 주시면 빠르게 지원해 드리겠습니다.

## 보안 고려사항

1. **HTTPS 필수**: 모든 포스트백 전송은 HTTPS 프로토콜을 사용합니다
2. **서명 검증 필수**: 모든 포스트백 요청에 대해 `signed_value` 검증을 반드시 수행해야 합니다
3. **Secret Key 관리**: OS별/앱별 `app_secret`을 환경변수나 보안 저장소에 안전하게 분리 보관해야 합니다

## 문의사항

연동 관련 문의사항이 있으시면 아래 채널로 연락 주시기 바랍니다:

- 기술 지원: developer@1self.world

## 변경 이력

| 버전 | 날짜       | 변경 내용                                |
| ---- | ---------- | ---------------------------------------- |
| 1.1  | 2025-09-10 | 문서 구조 개선, IFA 필드 통합, 목차 추가 |
| 1.0  | 2025-09-09 | 초기 버전 발행                           |
