# AdChain S2S 게임 연동 가이드

## 목차

- [1. 개요](#1-개요)
- [2. 시작 전 준비사항](#2-시작-전-준비사항)
- [3. 연동 흐름](#3-연동-흐름)
- [4. 인증](#4-인증)
- [5. API 명세](#5-api-명세)
  - [5.1 게임 목록 조회](#51-게임-목록-조회)
  - [5.2 랜딩 URL 생성](#52-랜딩-url-생성)
- [6. 에러 응답](#6-에러-응답)
  - [6.1 응답 포맷](#61-응답-포맷)
  - [6.2 errorCode 목록](#62-errorcode-목록)
  - [6.3 엔드포인트별 발생 가능 에러](#63-엔드포인트별-발생-가능-에러)
- [7. 포스트백 수신](#7-포스트백-수신)
- [8. 서명 검증](#8-서명-검증)
- [9. 구현 가이드](#9-구현-가이드)
- [10. 데이터 예시](#10-데이터-예시)
- [11. 테스트 가이드](#11-테스트-가이드)
- [12. 문의](#12-문의)

---

## 1. 개요

### S2S 연동이란

S2S(Server-to-Server) 연동은 매체사 서버에서 AdChain API를 직접 호출하여 게임 캠페인을 연동하는 방식입니다. AdChain으로부터 게임 미션 물량을 받아 하위 매체에 재공급하는 대대행 구조를 지원합니다.

### 연동 대상

- AdChain의 게임 미션 캠페인을 하위 매체에 공급하려는 매체사(네트워크 공급사)
- 매체사 서버에서 게임 목록을 직접 관리하고 하위 매체에 전달하는 경우
- 자체 UI/UX로 게임 광고를 표시하는 경우
- SDK 연동이 어려운 환경

---

## 2. 시작 전 준비사항

| # | 항목 | 설명 | 담당 |
|---|------|------|------|
| 1 | 연동 요청 | AdChain 담당자에게 S2S 연동 요청 | 매체사 → AdChain |
| 2 | `pub_secret` / `app_key` 발급 | 매체사 등록 완료 후 키 전달 | AdChain → 매체사 |
| 3 | 콜백 URL 전달 | 포스트백을 수신할 매체사 서버 URL 전달 | 매체사 → AdChain |
| 4 | 연동 테스트 | 스테이지 환경에서 API 호출 확인 | 매체사 |

> `pub_secret`과 `app_key`는 매체사에서 직접 생성하는 것이 아니라, 연동 요청 시 AdChain에서 발급하여 전달합니다. `app_key`는 앱(OS)별로 구분되는 고유 식별자입니다.
> 기술 지원: developer@1self.world

---

## 3. 연동 흐름

| 단계 | 방향 | 설명 |
| --- | --- | --- |
| 1 | 매체사 → AdChain | 게임 목록 조회 (GET /game/pub) |
| 2 | 매체사 → AdChain | 랜딩 URL 요청 (POST /game/pub/landing-url) |
| 3 | AdChain → 매체사 | 미션 완료 포스트백 수신 |

### 상세 설명

1. **게임 목록 조회**: 매체사에 할당된 활성 게임 이벤트 목록과 각 게임의 미션 정보를 가져옵니다
2. **랜딩 URL 생성**: 최종 유저가 게임에 참여할 때 사용할 트래킹 URL을 생성합니다
3. **포스트백 수신**: 미션 완료 시 AdChain에서 매체사 서버로 포스트백을 전송합니다

---

## 4. 인증

### 인증 헤더

S2S API는 `x-pub-secret`과 `x-app-key` 헤더를 사용하여 인증합니다.

```http
x-pub-secret: {발급받은_pub_secret}
x-app-key: {발급받은_app_key}
```

| 헤더 | 필수 | 설명 |
|---|---|---|
| x-pub-secret | Y | 매체사 시크릿 키 |
| x-app-key | Y | 앱 식별자 (OS별 구분, AdChain에서 발급) |

### 주의사항

- `pub_secret`과 `app_key`는 서버 환경변수로 안전하게 관리해야 합니다
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

### 5.1 게임 목록 조회

매체사에 할당된 전체 활성 게임 이벤트 목록을 조회합니다. 비활성 캠페인은 목록에서 제외됩니다.

**Endpoint**

```
GET /v1/api/game/pub
```

**Headers**

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| x-pub-secret | Y | 매체사 시크릿 키 |
| x-app-key | Y | 앱 식별자 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| country | enum | N | 국가 코드 (예: `KR`). 미전달 시 `KR` 기본 적용 |
| ~~platform~~ | ~~enum~~ | ~~Y~~ | ~~`android` 또는 `ios`~~ **(Deprecated)** — `x-app-key` 헤더로 앱(OS)이 구분되므로 더 이상 사용하지 않습니다. 하위 호환을 위해 전달할 수 있으나, 향후 제거될 예정입니다. |

**Request 예시**

```bash
curl -X GET "https://adchain-api.1self.world/v1/api/game/pub" \
  -H "x-pub-secret: your_pub_secret_here" \
  -H "x-app-key: 100000001"
```

> **참고**: `platform` 쿼리 파라미터는 Deprecated되었습니다. `x-app-key`로 앱(OS)이 구분되므로 별도 전달이 필요하지 않습니다.

**Response**

```json
{
  "events": [
    {
      "eventId": "550e8400-e29b-41d4-a716-446655440000",
      "title": "에픽 RPG 어드벤처",
      "description": "판타지 세계에서 모험을 떠나세요!",
      "iconImageUrl": "https://cdn.example.com/icon.png",
      "imageUrl": "https://cdn.example.com/cover.png",
      "packageName": "com.example.epicrpg",
      "platform": "android",
      "category": "GAMES",
      "subCategory": "RPG",
      "totalRewardPoints": 5000,
      "missionCount": 3,
      "previewUrl": "https://play.google.com/store/apps/details?id=com.example.epicrpg",
      "biddingModel": "fixed_rate",
      "type": "Standard",
      "updateDate": "2026-04-26T10:30:00.000Z",
      "score": 85.5,
      "subtitle": "신규 유저 전용 이벤트",
      "footerDescription": "<p>**신규 유저만 참여 가능합니다</p>",
      "action": "지금 설치하기",
      "missions": [
        {
          "missionId": "mission-uuid-1",
          "name": "앱 설치",
          "type": "Install",
          "rewardPoints": 500,
          "description": "게임을 설치하세요",
          "sortOrder": 1,
          "missionGroup": "Standard",
          "maxCountOccurences": 1,
          "maxCountTimeframeInSeconds": 0,
          "trackerDelay": 0,
          "attributionWindowDays": 30
        },
        {
          "missionId": "mission-uuid-2",
          "name": "레벨 10 달성",
          "type": "Level",
          "rewardPoints": 1000,
          "description": "캐릭터 레벨 10을 달성하세요",
          "sortOrder": 2,
          "missionGroup": "Standard",
          "maxCountOccurences": 1,
          "maxCountTimeframeInSeconds": 0,
          "trackerDelay": 0,
          "attributionWindowDays": 14
        },
        {
          "missionId": "mission-uuid-3",
          "name": "첫 결제",
          "type": "Purchase",
          "rewardPoints": 3500,
          "description": "게임 내 첫 결제를 완료하세요",
          "sortOrder": 3,
          "missionGroup": "Bonus",
          "maxCountOccurences": 1,
          "maxCountTimeframeInSeconds": 0,
          "trackerDelay": 86400,
          "attributionWindowDays": 30
        }
      ]
    }
  ]
}
```

**Response 필드 설명**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| eventId | string | 게임 이벤트 고유 ID (UUID) |
| title | string | 게임 제목 |
| description | string | 게임 설명 |
| iconImageUrl | string | 아이콘 이미지 URL |
| imageUrl | string | 커버 이미지 URL |
| packageName | string | 앱 패키지명 |
| platform | string | 플랫폼 (android/ios) |
| category | string | 카테고리 (GAMES, APPS) |
| subCategory | string | 서브 카테고리 (RPG, Puzzle 등) |
| totalRewardPoints | number | 전체 미션 완료 시 총 보상 포인트 |
| missionCount | number | 미션 개수 |
| missions | array | 미션 목록 |
| previewUrl | string | 앱스토어 URL |
| biddingModel | string | 입찰 모델. `CPI`: 설치당 지급, `fixed_rate`: 멀티리워드 rev share 모델 |
| type | string | 오퍼 타입. `Standard`: 일반 오퍼, `Play2Earn`: 플레이투언 오퍼 |
| updateDate | string | 오퍼 마지막 업데이트 시간 (ISO 8601 형식) |
| score | number | 캠페인 성과 점수 (높을수록 퍼포먼스가 좋은 캠페인) |
| subtitle | string | 부제목 (선택적) |
| footerDescription | string | 푸터 추가 설명. HTML 태그 포함 가능 |
| action | string | CTA(Call-to-Action) 버튼 텍스트 (예: "지금 설치하기") |

**Mission 필드 설명**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| missionId | string | 미션 고유 ID |
| name | string | 미션 이름 |
| type | string | 미션 타입 (Install, Level, Purchase, Achievement 등) |
| rewardPoints | number | 미션 완료 보상 포인트 |
| description | string | 미션 설명 |
| sortOrder | number | 정렬 순서 |
| missionGroup | string | 미션 그룹 (Standard, Bonus) |
| maxCountOccurences | number | 유저당 최대 달성 횟수. 이 횟수를 채우면 이벤트가 완료됨 |
| maxCountTimeframeInSeconds | number | 시간 프레임 (초). `maxCountOccurences`와 함께 사용. `0`이면 총 횟수 제한, `>0`이면 해당 시간 후 카운터 리셋 |
| trackerDelay | number | 트래커 지연 시간 (초). 일부 광고주는 이벤트를 배치로 전송하여 최대 24시간(86400초) 지연될 수 있음 |
| attributionWindowDays | number | 귀속 기간 (일). 이 기간 내에 발생한 이벤트만 유효하게 추적됨 |

**반복 이벤트 vs 재발생 이벤트**

미션이 여러 번 완료될 수 있는 경우, `maxCountOccurences`와 `maxCountTimeframeInSeconds` 조합으로 동작이 결정됩니다.

**1. 단순 반복 (Repeatable Event)**

- 조건: `maxCountOccurences > 1` AND `maxCountTimeframeInSeconds = 0`
- 동작: 유저는 `attributionWindowDays` 기간 내 총 `maxCountOccurences`번까지 보상 수령 가능
- 예시:
  ```json
  { "maxCountOccurences": 3, "maxCountTimeframeInSeconds": 0, "attributionWindowDays": 7 }
  ```
  → 7일 내 총 3번까지 보상. 3번 달성 시 완료.

**2. 시간 프레임 반복 (Recurring Event)**

- 조건: `maxCountOccurences >= 1` AND `maxCountTimeframeInSeconds > 0`
- 동작: `maxCountTimeframeInSeconds` 간격마다 카운터가 리셋되어 `maxCountOccurences`번씩 반복 가능
- 예시:
  ```json
  { "maxCountOccurences": 3, "maxCountTimeframeInSeconds": 86400, "attributionWindowDays": 7 }
  ```
  → 매일(86400초) 3번씩, 7일간 반복. 총 3×7=21번 가능

---

### 5.2 랜딩 URL 생성

최종 유저가 게임에 참여할 때 트래킹 URL을 생성합니다. 이 URL로 최종 유저를 리다이렉트하면 설치/미션이 추적됩니다.

**Endpoint**

```
POST /v1/api/game/pub/landing-url
```

**Headers**

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| x-pub-secret | Y | 매체사 시크릿 키 |
| x-app-key | Y | 앱 식별자 |
| Content-Type | Y | application/json |

**Request Body**

| 필드 | 타입 | 필수 | 최대 길이 | 설명 |
| --- | --- | --- | --- | --- |
| eventId | string | Y | 36자 | 게임 이벤트 ID (UUID) |
| userId | string | Y | 128자 | 최종 유저 식별자 |
| platform | enum | Y | - | `android` 또는 `ios` |
| ifa | string | **Y** | 36자 | 광고 식별자 (GAID/IDFA) |
| extra1 | string | N | 128자 | 매체사 자유 필드 1 — 포스트백 시 그대로 반환 |
| extra2 | string | N | 128자 | 매체사 자유 필드 2 — 포스트백 시 그대로 반환 |
| extra3 | string | N | 128자 | 매체사 자유 필드 3 — 포스트백 시 그대로 반환 |
| extra4 | string | N | 128자 | 매체사 자유 필드 4 — 포스트백 시 그대로 반환 |
| extra5 | string | N | 128자 | 매체사 자유 필드 5 — 포스트백 시 그대로 반환 |

> **중요**: `ifa`는 **필수** 파라미터입니다. 게임 캠페인의 정확한 어트리뷰션을 위해 반드시 전달해야 합니다.

> **extra1~5**: 매체사가 내부 추적 용도로 자유롭게 사용할 수 있는 페이로드 필드입니다. 랜딩 URL 생성 시 전달한 값이 포스트백에 그대로 포함되어 반환됩니다.

**Request 예시**

```bash
curl -X POST "https://adchain-api.1self.world/v1/api/game/pub/landing-url" \
  -H "x-pub-secret: your_pub_secret_here" \
  -H "x-app-key: 100000001" \
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

**에러 응답**

실패 시 HTTP `400` 또는 `401`이 반환됩니다. 상세 에러 포맷 및 발생 가능한 errorCode는 [6. 에러 응답](#6-에러-응답)을 참고하세요.

---

## 6. 에러 응답

### 6.1 응답 포맷

실패 응답은 아래 포맷을 따릅니다. 기존 필드(`statusCode`/`success`, `message`)는 그대로 유지되며, 매체사는 `errorCode` 필드(숫자)로 에러 케이스를 구분할 수 있습니다.

**예시**

```json
HTTP/1.1 400 Bad Request

{
  "statusCode": 400,
  "message": "Event not found",
  "errorCode": 451
}
```

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| statusCode | number | Y | HTTP status code (응답 헤더와 동일) |
| message | string | Y | 사람이 읽을 수 있는 에러 설명 (디버깅/로깅용) |
| errorCode | number | Y | AdChain 에러 코드 (3자리 숫자) — 매체사 매핑에 사용 |

> **주의**: `message` 문자열은 로깅/디버깅 용도입니다. 매체사 내부 로직 분기는 반드시 `errorCode` 기준으로 작성해 주세요. `message`는 사전 공지 없이 문구가 보강될 수 있습니다.
>
> `errorCode`는 HTTP status와 **별개의 값**입니다. 예를 들어 HTTP `400`이더라도 `errorCode`는 `450`, `451`, `452` 등 세부 원인에 따라 달라집니다.

### 6.2 errorCode 목록

| errorCode | HTTP | 이름 | 설명 |
| --- | --- | --- | --- |
| `401` | 401 | AUTH_MISSING_SECRET | `x-pub-secret` 헤더가 요청에 없음 |
| `402` | 401 | AUTH_INVALID_SECRET | `x-pub-secret` 값이 유효하지 않거나 비활성화된 매체사 |
| `400` | 400 | VALIDATION_MISSING_APP_KEY | `x-app-key` 헤더가 요청에 없음 (GET `/pub`) |
| `450` | 400 | ALREADY_PARTICIPATED | 동일 유저/기기가 이미 해당 이벤트에 참여함 |
| `451` | 400 | EVENT_NOT_FOUND | `eventId`에 해당하는 이벤트가 없거나 비활성 상태 |
| `452` | 400 | CAMPAIGN_ENDED | 캠페인이 광고주 측에서 종료 처리됨 |
| `453` | 400 | QUOTA_EXHAUSTED | 당일 캠페인 물량이 모두 소진됨 (익일 재개 가능) |
| `454` | 400 | EVENT_APP_KEY_MISMATCH | 요청 `x-app-key`가 해당 이벤트의 소속 앱과 다름 |
| `461` | 400 | PROVIDER_ERROR | 외부 광고 네트워크 호출 실패 또는 자격 미달 (기기/유저 조건 등) |
| `500` | 500 | SERVER_INTERNAL_ERROR | AdChain 서버 내부 예외 — 재시도 권장 |

### 6.3 엔드포인트별 발생 가능 에러

#### `GET /v1/api/game/pub`

| errorCode | HTTP | 이름 |
| --- | --- | --- |
| `401` | 401 | AUTH_MISSING_SECRET |
| `402` | 401 | AUTH_INVALID_SECRET |
| `400` | 400 | VALIDATION_MISSING_APP_KEY |
| `500` | 500 | SERVER_INTERNAL_ERROR |

#### `POST /v1/api/game/pub/landing-url`

| errorCode | HTTP | 이름 |
| --- | --- | --- |
| `401` | 401 | AUTH_MISSING_SECRET |
| `402` | 401 | AUTH_INVALID_SECRET |
| `450` | 400 | ALREADY_PARTICIPATED |
| `451` | 400 | EVENT_NOT_FOUND |
| `452` | 400 | CAMPAIGN_ENDED |
| `453` | 400 | QUOTA_EXHAUSTED |
| `454` | 400 | EVENT_APP_KEY_MISMATCH |
| `461` | 400 | PROVIDER_ERROR |
| `500` | 500 | SERVER_INTERNAL_ERROR |

---

## 7. 포스트백 수신

최종 유저가 게임 미션을 완료하면 AdChain에서 매체사 서버로 포스트백을 전송합니다.

### 포스트백 수신 URL 설정

포스트백을 받을 Endpoint를 AdChain 팀에 전달해 주세요. (developer@1self.world)

### 요청 정보

- **Method**: `POST`
- **Content-Type**: `application/json`

### Request Body

```json
{
  "callback_id": "b6fcca4e-e7b8-4a70-94fd-810b1b6a256b",
  "type": "game",
  "revenue_type": "cpi",
  "user_id": "user_12345",
  "amount": "500",
  "campaign_key": "ext-campaign-9281",
  "campaign_name": "에픽 RPG 어드벤처",
  "signed_value": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "mission_id": "mission-uuid-1",
  "mission_name": "앱 설치",
  "install_id": "92598636",
  "campaign_model": "CPI",
  "extra1": "sub_media_001",
  "extra2": "campaign_group_A"
}
```

### 필드 설명

| 필드명 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| callback_id | string | Y | 고유 트랜잭션 ID (UUID) - 중복 처리 방지용 |
| type | string | Y | 콘텐츠 유형 (`game`: 게임 미션 완료) |
| revenue_type | string | Y | 수익화 모델 (`cpi`: 설치, `cpa`: 액션, `cps`: 구매) |
| user_id | string | Y | 최종 유저 식별자 |
| amount | string | Y | 보상 포인트 |
| campaign_key | string | Y | 외부 광고 네트워크 측 캠페인 키 |
| campaign_name | string | N | 캠페인 명칭 (게임 제목) |
| signed_value | string | Y | HMAC-MD5 서명값 |
| app_key | string | N | 앱 식별자 (서명 검증 시 app_secret 선택에 사용) |
| os | string | N | 운영체제 (android/ios) |
| ifa | string | N | 광고 식별자 (GAID/IDFA) — 랜딩 URL 생성 시 전달한 값이 있으면 그대로 반환 |
| event_id | string | Y | AdChain 내부 게임 이벤트 ID (랜딩 URL 생성 시 전달한 `eventId`) |
| mission_id | string | Y | 완료한 미션 ID |
| mission_name | string | N | 완료한 미션 이름 |
| install_id | string | N | 광고 네트워크 설치 ID |
| campaign_model | string | N | 광고주 입찰 모델 (CPI, fixed_rate 등) |
| extra1 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 1 |
| extra2 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 2 |
| extra3 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 3 |
| extra4 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 4 |
| extra5 | string | N | 랜딩 URL 생성 시 전달한 매체사 자유 필드 5 |

> **`campaign_key` vs `event_id`**: `campaign_key`는 광고 네트워크(외부) 측 캠페인 식별자이고, `event_id`는 AdChain 내부 게임 이벤트 ID입니다. 서명 검증과 매체사 측 캠페인 매핑은 `campaign_key`를 기준으로 합니다.

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

## 8. 서명 검증

### 서명 검증 필요성

포스트백의 `signed_value`를 검증하여 요청이 AdChain으로부터 온 유효한 요청인지 확인해야 합니다.

### 서명 생성 방식

1. **서명 대상 문자열 생성**:

```
plainText = callback_id + user_id + amount + campaign_key
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
4. **문자열 순서**: `callback_id → user_id → amount → campaign_key` 순서가 중요합니다

---

## 9. 구현 가이드

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
app.post('/game-postback', async (req, res) => {
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

## 10. 데이터 예시

### 게임 설치 미션 포스트백 (CPI)

```json
{
  "callback_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "type": "game",
  "revenue_type": "cpi",
  "user_id": "user_12345",
  "amount": "500",
  "campaign_key": "ext-campaign-9281",
  "campaign_name": "에픽 RPG 어드벤처",
  "signed_value": "7f8a9b2c3d4e5f6a1b2c3d4e5f6a7b8c",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "mission_id": "mission-uuid-1",
  "mission_name": "앱 설치",
  "install_id": "92598636",
  "campaign_model": "CPI",
  "extra1": "sub_media_001",
  "extra2": "campaign_group_A"
}
```

### 레벨 달성 미션 포스트백 (CPA)

```json
{
  "callback_id": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "type": "game",
  "revenue_type": "cpa",
  "user_id": "user_12345",
  "amount": "1000",
  "campaign_key": "ext-campaign-9281",
  "campaign_name": "에픽 RPG 어드벤처",
  "signed_value": "9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "mission_id": "mission-uuid-2",
  "mission_name": "레벨 10 달성",
  "install_id": "92598636",
  "campaign_model": "fixed_rate",
  "extra1": "sub_media_001"
}
```

### 인앱 결제 미션 포스트백 (CPS)

```json
{
  "callback_id": "c3d4e5f6-a7b8-9012-cdef-345678901234",
  "type": "game",
  "revenue_type": "cps",
  "user_id": "user_12345",
  "amount": "3500",
  "campaign_key": "ext-campaign-9281",
  "campaign_name": "에픽 RPG 어드벤처",
  "signed_value": "2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e",
  "app_key": "100000001",
  "os": "android",
  "ifa": "38400000-8cf0-11bd-b23e-10b96e40000d",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "mission_id": "mission-uuid-3",
  "mission_name": "첫 결제",
  "install_id": "92598636",
  "campaign_model": "fixed_rate",
  "extra3": "promo_2026Q1"
}
```

---

## 11. 테스트 가이드

### 테스트 환경

**실제 서비스 적용 전 AdChain 개발팀과 함께 테스트를 진행해 주세요.**

1. Staging 환경 URL: `https://adchain-stage-api.1self.world`
2. 테스트용 `pub_secret` / `app_key` 발급 요청
3. 포스트백 수신 Endpoint 전달

### 테스트 시나리오

| 단계 | 테스트 항목 | 확인 사항 |
| --- | --- | --- |
| 1 | 게임 목록 조회 | API 응답 정상 수신, missions 배열 및 미션 메타데이터 확인 |
| 2 | 랜딩 URL 생성 | landingUrl 정상 생성, ifa 필수 전달 확인 |
| 3 | 포스트백 수신 | 서명 검증 성공 |
| 4 | 중복 처리 | 동일 callback_id 중복 요청 시 스킵 |
| 5 | 에러 처리 | 잘못된 서명 시 401 응답 |
| 6 | extra 필드 검증 | 랜딩 URL에 전달한 extra 값이 포스트백에 그대로 반환되는지 확인 |
| 7 | mission_id 검증 | 미션별로 다른 `mission_id` / `mission_name`이 포스트백에 포함되는지 확인 |

### 테스트 요청

테스트 관련 모든 문의는 developer@1self.world로 연락 주시면 빠르게 지원해 드리겠습니다.

---

## 12. 문의

연동 관련 문의사항이 있으시면 아래 채널로 연락 주시기 바랍니다:

- **기술 지원**: developer@1self.world

---
