# AdChain 오퍼월 서버 연동 가이드

**Version:** 1.0.0  
**최종 수정:** 2025-09-30  

---

## 📑 목차

1. [개요](#개요)
   - [SDK와 서버의 역할](#sdk와-서버의-역할)
   - [연동 플로우](#연동-플로우)
2. [파트너사 필수 구현 사항](#파트너사-필수-구현-사항)
   - [포스트백 수신 API](#1️⃣-포스트백-수신-api-필수)
   - [사용자 정보 조회 API](#2️⃣-사용자-정보-조회-api-필수)
3. [연동 체크리스트](#연동-체크리스트)
4. [포인트 시스템 구현 예시](#포인트-시스템-구현-예시-참고)
   - [DB 구조 설계](#권장-db-구조-설계)
   - [필수 인덱스 설정](#필수-인덱스-설정)
   - [포인트 지급 구현](#포인트-지급-플로우-상세-구현)
   - [사용자 API 구현](#사용자-api-구현-예시)
5. [트러블슈팅](#트러블슈팅)
6. [참고 자료](#참고-자료)

---

## 개요

### SDK와 서버의 역할

- **SDK 역할**: 앱 내 다양한 위젯 제공 + WebView 오퍼월 연결
- **서버 역할**: 포스트백 수신, 보상 검증 및 지급, 트랜잭션 기록, 유저 API 제공

### 연동 플로우

```
[사용자] → [AdChain 오퍼월] → [광고 참여] → [AdChain 검증] 
                                                    ↓
[파트너사 앱] ← [포인트 지급] ← [파트너사 서버] ← [포스트백]
```

---

## 파트너사 필수 구현 사항

파트너사는 **2개의 API만 구현**하시면 AdChain 오퍼월 백엔드 연동이 완료됩니다.

### 1️⃣ 포스트백 수신 API (필수)

AdChain에서 보내는 포스트백을 수신하여 사용자에게 리워드를 지급합니다.

#### 구현 요구사항

1. **엔드포인트**: `POST /callback/offerwall/adchain`
2. **서명 검증**: HMAC SHA256으로 요청 검증
3. **중복 방지**: trans_id를 이용한 중복 지급 방지
4. **응답**: 모든 경우에 HTTP 200 반환 (재시도 방지)

#### ⚠️ 중요 사항

- **trans_id**는 AdChain이 제공하는 고유 ID로, 중복 지급 방지에 필수입니다
- 에러 발생 시에도 HTTP 200을 반환해야 AdChain이 재시도하지 않습니다
- 서명 검증은 보안상 필수입니다

**👉 상세 구현 가이드**: [포스트백 연동 가이드](https://github.com/1selfworld-labs/adchain-guide/blob/master/PUBLISHER_INTEGRATION_GUIDE.md)

### 2️⃣ 사용자 정보 조회 API (필수)

AdChain이 오퍼월 표시를 위해 사용자 정보를 조회합니다.

#### 구현 요구사항

1. **엔드포인트**: `GET /api/adchain/users/:userId`
2. **응답 필드**: userId, status (active/inactive), createdAt
3. **인증**: API Key 또는 Bearer Token

**👉 상세 구현 가이드**: [사용자 API 가이드](https://github.com/1selfworld-labs/adchain-guide/blob/master/PUBLISHER_USER_API_GUIDE.md)

---

## 연동 체크리스트

### ✅ 필수 구현
```
[ ] 포스트백 수신 API 구현
    [ ] 서명 검증 로직
    [ ] trans_id 기반 중복 방지
    [ ] HTTP 200 응답
    
[ ] 사용자 정보 조회 API 구현
    [ ] 사용자 상태 확인
    [ ] 인증 처리
    
[ ] AdChain에 URL 등록
    [ ] 포스트백 URL 설정
    [ ] Secret Key 수령
```

### 🔒 보안 설정
```
[ ] HTTPS 적용
[ ] Secret Key 환경변수 관리
[ ] API 인증 구현
```

---

## 포인트 시스템 구현 예시 (참고)

파트너사의 포인트 시스템 구축을 위한 참고 예시를 제공합니다. 실제 운영 중인 서비스들의 사례를 바탕으로 안정적인 포인트 시스템 설계 방안을 제안드립니다.

### 권장 DB 구조 설계

포인트 시스템은 일반적으로 3개의 핵심 컬렉션으로 구성됩니다:

```
Users (사용자)
  ├─ Accounts (포인트 계정)
  └─ Transactions (거래 내역)
        └─ PostbackLogs (포스트백 원본 로그)
```

#### 컬렉션별 역할

- **Accounts (계정/지갑)**: 사용자별 잔액 관리. 지급/회수는 `$inc`로 원자성 보장
- **Transactions (거래내역)**: 지급/차감 이력 관리. type, partner, status로 출처/목적/진행상태를 투명하게 관리
- **PostbackLogs (원본 로그)**: 포스트백 원문 저장. 분쟁/오지급/보안 검증 대비

#### 1. Accounts 컬렉션 (포인트 지갑)

```javascript
// 사용자별 포인트 잔액 관리
{
  _id: "account_uuid",
  userId: "user_uuid",
  balance: 5000,              // 현재 포인트
  type: "point",              // point, coin 등 구분
  createdAt: ISODate("2025-01-01T00:00:00Z"),
  updatedAt: ISODate("2025-09-30T14:30:00Z")
}
```

#### 2. Transactions 컬렉션 (거래 내역)

**설계 포인트**:
- **type & partner 분리**: type=cs_reward, partner=internal 등 목적 vs 출처를 구분
- **status 필드**: pending→completed/failed로 진행 상태 추적 + UX 표시
- **updatedAt 필드**: 회계적 신뢰성. 상태 변경 이력 기록

```javascript
// 모든 포인트 입출금 내역 기록
{
  _id: "transaction_uuid",
  userId: "user_uuid",
  accountId: "account_uuid",
  type: "offer_wall_reward",
  partner: "adchain",
  amount: 100,                // 양수: 지급, 음수: 차감
  title: "게임 앱 설치 완료",
  subTitle: "3일 플레이 달성",
  trId: "adchain_tx_12345",   // AdChain trans_id (중복 방지 핵심)
  status: "completed",         // pending/completed/failed/cancelled
  createdAt: ISODate("2025-09-30T14:30:00Z"),
  updatedAt: ISODate("2025-09-30T14:30:00Z")
}
```

#### 3. PostbackLogs 컬렉션 (원본 로그)

```javascript
// AdChain 포스트백 원본 저장 (분쟁 대응용)
{
  _id: "log_uuid",
  partner: "adchain",
  userId: "user_uuid",
  transId: "adchain_tx_12345",
  raw: {                       // 원본 파라미터 전체
    user_id: "user_uuid",
    trans_id: "adchain_tx_12345",
    amount: "100",
    signature: "abc123...",
    offer_name: "게임 앱 설치",
    // ... 모든 파라미터
  },
  status: "processed",         // received/verified/processed/failed/duplicated
  errorMessage: null,
  createdAt: ISODate("2025-09-30T14:30:00Z")
}
```

### 필수 인덱스 설정

```javascript
// Accounts
db.accounts.createIndex({ userId: 1, type: 1 }, { unique: true })

// Transactions (가장 중요!)
db.transactions.createIndex({ trId: 1 }, { unique: true })  // 중복 방지 핵심
db.transactions.createIndex({ userId: 1, createdAt: -1 })   // 사용자별 조회
db.transactions.createIndex({ status: 1, createdAt: -1 })   // 상태별 조회

// PostbackLogs
db.postbackLogs.createIndex({ transId: 1 })
db.postbackLogs.createIndex({ partner: 1, createdAt: -1 })
```

### 포인트 지급 플로우 상세 구현

#### 기본 구현 (Simple)

```javascript
// POST /callback/offerwall/adchain 내부 로직
async function processPostback(params) {
  const { user_id, trans_id, amount, offer_name } = params;
  
  try {
    // 1. 포스트백 로그 저장
    await db.collection('postbackLogs').insertOne({
      partner: 'adchain',
      userId: user_id,
      transId: trans_id,
      raw: params,
      status: 'received',
      createdAt: new Date()
    });
    
    // 2. 사용자 확인
    const user = await db.collection('users').findOne({ _id: user_id });
    if (!user) {
      throw new Error('User not found');
    }
    
    // 3. 거래 기록 (unique index가 중복 자동 차단)
    await db.collection('transactions').insertOne({
      _id: generateUUID(),
      userId: user_id,
      type: 'offer_wall_reward',
      partner: 'adchain',
      amount: parseInt(amount),
      title: offer_name || '오퍼월 보상',
      trId: trans_id,  // AdChain trans_id 그대로 사용
      status: 'completed',
      createdAt: new Date()
    });
    
    // 4. 포인트 증가
    await db.collection('accounts').updateOne(
      { userId: user_id },
      { 
        $inc: { balance: parseInt(amount) },
        $set: { updatedAt: new Date() }
      }
    );
    
    // 5. 로그 업데이트
    await db.collection('postbackLogs').updateOne(
      { transId: trans_id },
      { $set: { status: 'processed' } }
    );
    
    return { success: true };
    
  } catch (error) {
    if (error.code === 11000) {  // MongoDB duplicate key error
      console.log('이미 처리된 트랜잭션:', trans_id);
      return { message: 'Already processed' };
    }
    
    // 실패 로그
    await db.collection('postbackLogs').updateOne(
      { transId: trans_id },
      { $set: { status: 'failed', errorMessage: error.message } }
    );
    
    throw error;
  }
}
```

#### 고급 구현 (Transaction 사용)

> 💡 **권장**: MongoDB Transaction을 사용하면 더 안전하고 롤백이 가능합니다.

```javascript
// MongoDB Transaction으로 원자성 보장
async function processPostbackWithTransaction(params) {
  const session = client.startSession();
  
  try {
    await session.withTransaction(async () => {
      const db = client.db('partner_db');
      const { user_id, trans_id, amount, offer_name } = params;
      
      // 1. 계정 조회
      const account = await db.collection('accounts').findOne(
        { userId: user_id, type: 'point' },
        { session }
      );
      
      if (!account) {
        throw new Error('Account not found');
      }
      
      // 2. 잔액 업데이트 ($inc 사용으로 동시성 문제 해결)
      await db.collection('accounts').updateOne(
        { _id: account._id },
        {
          $inc: { balance: parseInt(amount) },
          $set: { updatedAt: new Date() }
        },
        { session }
      );
      
      // 3. 거래 기록
      await db.collection('transactions').insertOne({
        _id: generateUUID(),
        userId: user_id,
        accountId: account._id,
        type: 'offer_wall_reward',
        partner: 'adchain',
        amount: parseInt(amount),
        title: offer_name || '오퍼월 보상',
        trId: trans_id,
        status: 'completed',
        createdAt: new Date()
      }, { session });
      
    });
    
    console.log('포인트 지급 완료:', { user_id, amount });
    return { success: true };
    
  } catch (error) {
    console.error('포인트 지급 실패:', error);
    
    if (error.code === 11000) {
      return { message: 'Already processed' };
    }
    
    throw error;
    
  } finally {
    await session.endSession();
  }
}
```

#### ⚠️ 중요: Balance 업데이트 방식

```javascript
// ❌ Race Condition 발생 가능
const account = await db.collection('accounts').findOne({ userId });
account.balance = account.balance + amount;
await db.collection('accounts').updateOne(
  { _id: account._id },
  { $set: { balance: account.balance } }
);

// ✅ 원자적 업데이트 (권장)
await db.collection('accounts').updateOne(
  { userId },
  { $inc: { balance: amount } }
);
```

### 사용자 API 구현 예시

#### 포인트 잔액 조회

```javascript
// GET /api/v1/users/:userId/balance
app.get('/api/v1/users/:userId/balance', async (req, res) => {
  const { userId } = req.params;
  
  const account = await db.collection('accounts').findOne({
    userId,
    type: 'point'
  });
  
  if (!account) {
    return res.status(404).json({ error: 'Account not found' });
  }
  
  res.json({
    userId,
    balance: account.balance,
    updatedAt: account.updatedAt
  });
});
```

#### 거래 내역 조회

```javascript
// GET /api/v1/users/:userId/transactions
app.get('/api/v1/users/:userId/transactions', async (req, res) => {
  const { userId } = req.params;
  const { limit = 20, offset = 0, type } = req.query;
  
  const filter = { userId };
  if (type) filter.type = type;
  
  const transactions = await db.collection('transactions')
    .find(filter)
    .sort({ createdAt: -1 })
    .skip(parseInt(offset))
    .limit(parseInt(limit))
    .toArray();
  
  const total = await db.collection('transactions').countDocuments(filter);
  
  res.json({
    transactions,
    pagination: {
      total,
      limit: parseInt(limit),
      offset: parseInt(offset),
      hasMore: (parseInt(offset) + parseInt(limit)) < total
    }
  });
});
```

### 거래 유형(type) 정의

확장성을 고려한 거래 유형 설계:

```javascript
const TRANSACTION_TYPES = {
  // 오퍼월 관련
  'offer_wall_reward': '오퍼월 보상',
  
  // 향후 확장 가능
  'event_reward': '이벤트 보상',
  'attendance_reward': '출석 보상',
  'referral_reward': '친구 초대 보상',
  'purchase': '상품 구매',
  'withdrawal': '출금',
  'admin_adjustment': '관리자 조정'
};
```

---

## 트러블슈팅

### 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| 중복 포인트 지급 | trans_id 미사용 또는 unique index 없음 | trans_id를 primary key나 unique로 설정 |
| AdChain 계속 재시도 | HTTP 200이 아닌 응답 | 모든 경우에 200 반환 |
| 서명 검증 실패 | Secret Key 불일치 | AdChain 제공 키 확인 |

---

## 참고 자료

### AdChain 공식 문서
- [포스트백 연동 가이드](https://github.com/1selfworld-labs/adchain-guide/blob/master/PUBLISHER_INTEGRATION_GUIDE.md) - 파라미터 명세, 서명 검증 상세
- [사용자 API 가이드](https://github.com/1selfworld-labs/adchain-guide/blob/master/PUBLISHER_USER_API_GUIDE.md) - 사용자 정보 API 상세

---
