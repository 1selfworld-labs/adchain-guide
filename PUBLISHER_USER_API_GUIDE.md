# 1SelfWorld AdChain 퍼블리셔 유저 정보 API 가이드

## 목차

- [개요](#개요)
- [API 명세](#api-명세)
  - [요청 정보](#요청-정보)
  - [요청 헤더](#요청-헤더)
  - [요청 파라미터](#요청-파라미터)
  - [응답 형식](#응답-형식)
  - [응답 필드 설명](#응답-필드-설명)
- [보안 및 암호화](#보안-및-암호화)
  - [암호화 방식](#암호화-방식)
  - [암호화 구현 예제](#암호화-구현-예제)
  - [복호화 검증](#복호화-검증)
- [구현 가이드](#구현-가이드)
  - [Node.js 구현 예제](#nodejs-구현-예제)
- [에러 처리](#에러-처리)
  - [에러 코드](#에러-코드)
  - [에러 응답 예시](#에러-응답-예시)
- [테스트 가이드](#테스트-가이드)
  - [테스트 시나리오](#테스트-시나리오)
  - [테스트 체크리스트](#테스트-체크리스트)
- [보안 고려사항](#보안-고려사항)
- [FAQ](#faq)
- [문의사항](#문의사항)
- [변경 이력](#변경-이력)

## 개요

1SelfWorld AdChain에서는 퍼블리셔 사용자의 정보를 안전하게 조회하기 위한 API 연동 방식을 제공합니다. 본 가이드는 퍼블리셔가 사용자 정보(이름, 잔액)를 암호화하여 제공하는 API를 구현하는 방법을 설명합니다.

이 API는 1SelfWorld가 퍼블리셔로부터 받은 `user_id`를 통해 해당 사용자의 정보를 조회하고, 안전하게 암호화된 형태로 전달받는 구조입니다.

## 연동을 위한 필수 제공 사항

### 1SelfWorld가 제공하는 정보
- **pub_secret**: 사용자 정보 암호화에 사용할 32자 문자열 키

### 퍼블리셔가 제공해야 하는 정보
- **API 엔드포인트**: 사용자 정보 조회 API URL
- **API 토큰**: API 호출 인증을 위한 토큰

## API 명세

### 요청 정보

- **Method**: `GET`
- **Endpoint**: 퍼블리셔가 제공하는 사용자 정보 조회 URL
- **URL 형식**: `https://api.publisher.com/users/{user_id}`

### 요청 헤더

퍼블리셔에서 API 토큰을 제공해주시면, 1SelfWorld는 해당 토큰을 포함하여 요청을 전송합니다.

| 헤더명          | 타입   | 필수 여부 | 설명                    | 예시                   |
| --------------- | ------ | --------- | ----------------------- | ---------------------- |
| `Authorization` | String | Required  | 퍼블리셔가 제공한 token | `{퍼블리셔 제공 토큰}` |

### 요청 파라미터

| 파라미터명 | 타입   | 위치 | 필수 여부 | 설명                        | 예시                                   |
| ---------- | ------ | ---- | --------- | --------------------------- | -------------------------------------- |
| `user_id`  | String | Path | Required  | 조회할 사용자의 고유 식별자 | `ab0da900-7465-4231-8657-1ef40944a8a2` |

### 응답 형식

#### 성공 응답

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "message": "User information retrieved successfully",
  "result": "U2FsdGVkX1+vupppZksvRf5pq5g5XjFRlipRkwB0K1Y96Qsv2Lu6zPgKbctvRV",
  "timestamp": "2025-09-17T10:30:45Z"
}
```

### 응답 필드 설명

| 필드명      | 타입    | 필수 여부 | 설명                                         | 예시                                                             |
| ----------- | ------- | --------- | -------------------------------------------- | ---------------------------------------------------------------- |
| `success`   | Boolean | Required  | API 처리 성공 여부                           | `true`                                                           |
| `message`   | String  | Required  | 응답 메시지                                  | `successfully`                                                   |
| `result`    | String  | Required  | AES-256 암호화된 사용자 정보 (Base64 인코딩) | `U2FsdGVkX1+vupppZksvRf5pq5g5XjFRlipRkwB0K1Y96Qsv2Lu6zPgKbctvRV` |
| `timestamp` | String  | Required  | 응답 생성 시각 (ISO 8601 형식)               | `2025-09-17T10:30:45Z`                                           |

#### 복호화 후 데이터 형식

`result`를 복호화하면 다음과 같은 JSON 구조를 얻을 수 있습니다:

```json
{
  "user_name": "홍길동",
  "user_balance": 15000
}
```

| 필드명         | 타입   | 설명                    | 예시     |
| -------------- | ------ | ----------------------- | -------- |
| `user_name`    | String | 사용자 이름/닉네임      | `홍길동` |
| `user_balance` | Number | 사용자 보유 포인트/잔액 | `15000`  |

## 보안 및 암호화

### 암호화 방식

사용자 정보는 **AES-256-CBC** 암호화 알고리즘을 사용하여 보호됩니다.

#### 암호화 사양

- **알고리즘**: AES-256-CBC
- **Key**: 1SelfWorld에서 제공하는 32자 문자열 `pub_secret`
- **IV (Initialization Vector)**: 16바이트 랜덤 값
- **Padding**: PKCS7
- **출력 형식**: Base64 인코딩

### 암호화 구현 예제

#### Node.js (crypto 모듈 사용)

```javascript
const crypto = require('crypto');

// 1SelfWorld에서 제공한 pub_secret
const PUB_SECRET = process.env.PUB_SECRET; // 32자 문자열

function encryptUserData(userData) {
  // 암호화할 데이터
  const dataString = JSON.stringify(userData);

  // IV 생성 (16바이트)
  const iv = crypto.randomBytes(16);

  // 암호화
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(PUB_SECRET, 'utf8'), iv);

  let encrypted = cipher.update(dataString, 'utf8', 'base64');
  encrypted += cipher.final('base64');

  // IV와 암호화된 데이터를 함께 저장 (IV:암호문 형식)
  const combined = Buffer.concat([iv, Buffer.from(encrypted, 'base64')]);

  return combined.toString('base64');
}

// API 엔드포인트 구현 예시
app.get('/users/:userId/info', async (req, res) => {
  try {
    const { userId } = req.params;

    // 사용자 정보 조회 (DB에서)
    const userInfo = await getUserFromDatabase(userId);

    if (!userInfo) {
      return res.status(404).json({
        success: false,
        message: 'User not found',
        timestamp: new Date().toISOString(),
      });
    }

    // 암호화할 데이터 준비
    const userData = {
      user_name: userInfo.name,
      user_balance: userInfo.balance,
    };

    // 데이터 암호화
    const encryptedData = encryptUserData(userData);

    // 응답
    return res.json({
      success: true,
      message: 'User information retrieved successfully',
      result: encryptedData,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    console.error('Error processing request:', error);
    return res.status(500).json({
      success: false,
      message: 'Internal server error',
      timestamp: new Date().toISOString(),
    });
  }
});
```

### 복호화 검증

1SelfWorld에서는 다음과 같은 방식으로 데이터를 복호화합니다:

```javascript
function decryptUserData(encryptedData, pubSecret) {
  const combined = Buffer.from(encryptedData, 'base64');

  // IV 추출 (처음 16바이트)
  const iv = combined.slice(0, 16);

  // 암호화된 데이터 추출
  const encrypted = combined.slice(16);

  // 복호화
  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(pubSecret, 'utf8'), iv);

  let decrypted = decipher.update(encrypted, null, 'utf8');
  decrypted += decipher.final('utf8');

  return JSON.parse(decrypted);
}
```

## 구현 가이드

### Node.js 구현 예제

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

// 미들웨어: 인증 검증
function validateAuth(req, res, next) {
  const token = req.headers['authorization'];

  if (!token) {
    return res.status(401).json({
      success: false,
      message: 'Missing authorization header',
      timestamp: new Date().toISOString(),
    });
  }

  // 토큰 검증 로직 (실제 구현 필요)
  const isValid = validateToken(token);
  if (!isValid) {
    return res.status(401).json({
      success: false,
      message: 'Invalid credentials',
      timestamp: new Date().toISOString(),
    });
  }

  next();
}

// 사용자 정보 조회 API
app.get('/users/:userId/info', validateAuth, async (req, res) => {
  try {
    const { userId } = req.params;

    // 사용자 정보 조회
    const user = await getUserInfo(userId);

    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found',
        timestamp: new Date().toISOString(),
      });
    }

    // 암호화
    const userData = {
      user_name: user.name,
      user_balance: user.balance,
    };

    const encryptedData = encryptUserData(userData);

    // 응답
    res.json({
      success: true,
      message: 'User information retrieved successfully',
      result: encryptedData,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    console.error('API Error:', error);
    res.status(500).json({
      success: false,
      message: 'Internal server error',
      timestamp: new Date().toISOString(),
    });
  }
});
```

## 에러 처리

### 에러 코드

| HTTP 상태 코드 | 에러 유형             | 설명                          |
| -------------- | --------------------- | ----------------------------- |
| `400`          | Bad Request           | 잘못된 요청 형식              |
| `401`          | Unauthorized          | 인증 실패 또는 필수 헤더 누락 |
| `500`          | Internal Server Error | 서버 내부 오류                |

### 에러 응답 예시

#### 사용자를 찾을 수 없는 경우

```json
{
  "success": false,
  "message": "User not found",
  "result": "",
  "timestamp": "2025-09-17T10:30:45Z"
}
```

#### 인증 실패

```json
{
  "success": false,
  "message": "Invalid authorization token",
  "result": "",
  "timestamp": "2025-09-17T10:30:45Z"
}
```

#### 서버 내부 오류

```json
{
  "success": false,
  "message": "Internal server error",
  "result": "",
  "timestamp": "2025-09-17T10:30:45Z"
}
```

## 보안 고려사항

### 필수 보안 요구사항

1. **HTTPS 통신 필수**

   - 모든 API 통신은 반드시 HTTPS를 사용해야 합니다
   - TLS 1.2 이상 버전 사용 권장

2. **App Secret 관리**

   - `pub_secret`은 환경변수나 보안 저장소에 보관
   - 코드에 하드코딩 절대 금지

## 변경 이력

| 버전 | 날짜       | 변경 내용      |
| ---- | ---------- | -------------- |
| 1.0  | 2025-09-17 | 초기 버전 발행 |
