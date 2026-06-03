---
tags: [api, rest, specification, myfin-v2]
created: 2026-06-02
version: "2.0"
status: approved
related:
  - "[[03_ERD]]"
  - "[[01_기획서]]"
  - "[[05_OWASP보안설계서]]"
broker_api_refs:
  - "[[ref_01_한국투자증권_API]]"
  - "[[ref_02_키움증권_REST_API]]"
---

# MyFin V2.0 REST API 명세서

> **Base URL:** `https://api.myfin.local/v1`
> **인증 방식:** `Authorization: Bearer <Access_Token>` (공개 API 제외)
> **콘텐츠 타입:** `Content-Type: application/json`
> **API 버전 전략:** URL 경로 기반 (`/v1/...`). 하위 호환 불가 변경 시 `/v2` 경로 추가.

---

## 브로커 API 참조 자료

myfin의 자동매매 엔진은 아래 증권사·거래소 API를 직접 호출한다. 엔드포인트 구현 시 각 파일을 반드시 참조한다.

| 파일 | 대상 증권사 | 주요 참조 항목 |
| --- | --- | --- |
| [[ref_01_한국투자증권_API]] | 한국투자증권 (KIS OpenAPI) | Access Token 발급·갱신, 주식 주문, 잔고 조회, 실시간 WebSocket 호가 |
| [[ref_02_키움증권_REST_API]] | 키움증권 (Kiwoom REST API) | 국내 주식 주문, 체결 내역, 잔고 조회 |

> [!important] 브로커 API 호출 시 필수 규칙
> 1. **Base URL 고정:** 사용자 입력 URL 절대 사용 금지. `BROKER_BASE_URLS` 딕셔너리에 하드코딩 (OWASP A10 SSRF 방어). 상세 → [[05_OWASP보안설계서]] §8
> 2. **API Key 복호화:** Celery Worker 내에서만 `CryptoService.decrypt()` 호출. FastAPI 라우터에서 직접 복호화 금지.
> 3. **Rate Limit 준수:** 한국투자증권 초당 20건, 키움증권 초당 5건 제한. 초과 시 Circuit Breaker가 자동 차단.

---

## 공통 규칙

### 공통 응답 포맷

**성공 응답:**
```json
{
  "success": true,
  "data": { ... }
}
```

**목록 응답 (페이지네이션):**
```json
{
  "success": true,
  "data": {
    "items": [ ... ],
    "total": 150,
    "page": 1,
    "size": 20,
    "pages": 8
  }
}
```

**에러 응답:**
```json
{
  "success": false,
  "error_code": "ERR_AUTH_001",
  "message": "액세스 토큰이 만료되었습니다.",
  "detail": null
}
```

### 공통 에러 코드 카탈로그

| 에러 코드 | HTTP 상태 | 설명 |
| --- | --- | --- |
| `ERR_AUTH_001` | 401 | Access Token 만료 |
| `ERR_AUTH_002` | 401 | Access Token 서명 불일치 |
| `ERR_AUTH_003` | 401 | Refresh Token 만료 또는 무효화 |
| `ERR_AUTH_004` | 401 | 이메일 또는 비밀번호 불일치 |
| `ERR_AUTH_005` | 409 | 이미 가입된 이메일 |
| `ERR_AUTHZ_001` | 403 | 해당 자원에 대한 접근 권한 없음 |
| `ERR_VALID_001` | 422 | 요청 바디 유효성 검증 실패 |
| `ERR_ACCOUNT_001` | 404 | 계좌를 찾을 수 없음 |
| `ERR_ACCOUNT_002` | 409 | 동일 계좌번호 중복 등록 |
| `ERR_RULE_001` | 404 | 매매 룰을 찾을 수 없음 |
| `ERR_RULE_002` | 409 | 이미 삭제된 룰 |
| `ERR_STOCK_001` | 404 | 종목 코드를 찾을 수 없음 |
| `ERR_WATCHLIST_001` | 404 | 관심목록을 찾을 수 없음 |
| `ERR_REPORT_001` | 404 | 보고서를 찾을 수 없음 |
| `ERR_REPORT_002` | 409 | 생성 중인 보고서가 이미 존재 |
| `ERR_SERVER_001` | 500 | 내부 서버 오류 |
| `ERR_SERVER_002` | 503 | Elasticsearch 응답 없음 |

### Rate Limiting (Nginx 레벨)

| 엔드포인트 그룹 | 제한 | 초과 응답 |
| --- | --- | --- |
| `POST /auth/login` | IP당 10회/분 | `429 Too Many Requests` |
| `POST /auth/register` | IP당 5회/시간 | `429 Too Many Requests` |
| 일반 API | IP당 100회/분 | `429 Too Many Requests` |
| `POST /reports/generate` | 사용자당 1회/시간 | `429 Too Many Requests` |

---

## 1. 인증 (Auth) — `/auth`

### `POST /auth/register` — 회원가입

- **인증:** 불필요 (공개 API)

**Request:**
```json
{
  "email": "user@example.com",
  "password": "MyStr0ng!Pass"
}
```

**비밀번호 정책:** 최소 8자, 대문자·소문자·숫자·특수문자 각 1개 이상 포함

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "user_id": 1,
    "email": "user@example.com",
    "created_at": "2026-06-02T13:00:00Z"
  }
}
```

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| 이미 가입된 이메일 | `ERR_AUTH_005` | 409 |
| 비밀번호 정책 미충족 | `ERR_VALID_001` | 422 |

---

### `POST /auth/login` — 로그인

- **인증:** 불필요

**Request:**
```json
{
  "email": "user@example.com",
  "password": "MyStr0ng!Pass"
}
```

**Response `200 OK`:**
```
Set-Cookie: refresh_token=<token>; HttpOnly; Secure; SameSite=Strict; Path=/v1/auth/refresh; Max-Age=2592000
```
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "expires_in": 3600
  }
}
```

> [!note] 보안 설계
> - Refresh Token은 HttpOnly + Secure + SameSite=Strict 쿠키로만 전달. JavaScript에서 접근 불가.
> - Access Token은 응답 바디로 전달. 프론트엔드에서 메모리(Vuex/Pinia)에만 저장. localStorage/sessionStorage 저장 금지.
> - 로그인 실패 시 "이메일 또는 비밀번호가 일치하지 않습니다."로 통일 응답. 이메일 존재 여부 노출 금지.

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| 이메일 또는 비밀번호 불일치 | `ERR_AUTH_004` | 401 |
| Rate Limit 초과 (10회/분) | — | 429 |

---

### `POST /auth/refresh` — Access Token 갱신

- **인증:** HttpOnly 쿠키의 Refresh Token (자동 전송)

**Response `200 OK`:**
```
Set-Cookie: refresh_token=<new_token>; HttpOnly; Secure; SameSite=Strict; Path=/v1/auth/refresh; Max-Age=2592000
```
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "expires_in": 3600
  }
}
```

> [!important] Refresh Token Rotation
> 이 API 호출 시 기존 Refresh Token은 즉시 무효화(`is_revoked = TRUE`)되고 새 토큰이 발급된다. 이미 무효화된 토큰으로 재요청 시 해당 사용자의 모든 세션을 무효화하고 `ERR_AUTH_003` 반환.

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| Refresh Token 만료 또는 무효 | `ERR_AUTH_003` | 401 |

---

### `POST /auth/logout` — 로그아웃

- **인증:** Bearer Access Token

**Response `200 OK`:**
```
Set-Cookie: refresh_token=; HttpOnly; Secure; Path=/v1/auth/refresh; Max-Age=0
```
```json
{
  "success": true,
  "data": { "message": "로그아웃 되었습니다." }
}
```

---

### `GET /auth/me` — 내 프로필 조회

- **인증:** Bearer Access Token

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "user_id": 1,
    "email": "user@example.com",
    "created_at": "2026-06-02T13:00:00Z"
  }
}
```

---

## 2. 계좌 및 API Key 관리 (Accounts) — `/accounts`

### `GET /accounts` — 계좌 목록 조회

- **인증:** Bearer Access Token

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `is_active` | boolean | — | 활성 계좌만 필터 |
| `broker_name` | string | — | 증권사 필터 |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "account_id": 1,
        "alias": "메인 국장 계좌",
        "broker_name": "korea_investment",
        "account_number": "12345678-01",
        "is_active": true,
        "token_type": "DYNAMIC",
        "expired_at": null,
        "created_at": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 1
  }
}
```

> [!warning] 응답에서 API Key 원문 절대 미포함
> `encrypted_api_key`, `encrypted_secret_key` 컬럼은 어떠한 응답에도 포함하지 않는다.

---

### `POST /accounts` — 계좌 등록

- **인증:** Bearer Access Token

**Request:**
```json
{
  "alias": "메인 국장 계좌",
  "broker_name": "korea_investment",
  "account_number": "12345678-01",
  "api_key": "YOUR_RAW_API_KEY",
  "secret_key": "YOUR_RAW_SECRET_KEY",
  "token_type": "DYNAMIC",
  "expired_at": null
}
```

**`broker_name` 허용값:** `korea_investment`, `upbit`, `binance`, `binance_futures`

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "account_id": 1,
    "alias": "메인 국장 계좌",
    "broker_name": "korea_investment",
    "is_active": true
  }
}
```

> [!note] 보안 처리
> `api_key`, `secret_key`는 백엔드 수신 즉시 Fernet 암호화 처리 후 DB 저장. 로그에도 기록하지 않음.

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| 동일 broker + account_number 중복 | `ERR_ACCOUNT_002` | 409 |
| 유효하지 않은 broker_name | `ERR_VALID_001` | 422 |

---

### `GET /accounts/{account_id}` — 계좌 상세 조회

- **인증:** Bearer Access Token (본인 소유 계좌만 접근 가능)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "account_id": 1,
    "alias": "메인 국장 계좌",
    "broker_name": "korea_investment",
    "account_number": "12345678-01",
    "is_active": true,
    "token_type": "DYNAMIC",
    "expired_at": null,
    "created_at": "2026-06-01T10:00:00Z",
    "updated_at": "2026-06-02T08:00:00Z",
    "latest_balance": {
      "total_balance": 5000000,
      "available_balance": 3200000,
      "currency": "KRW",
      "snapshot_at": "2026-06-02T15:30:00Z"
    }
  }
}
```

---

### `PUT /accounts/{account_id}` — 계좌 정보 수정

- **인증:** Bearer Access Token (소유권 확인 필수)

**Request:** (모든 필드 선택적)
```json
{
  "alias": "새 별칭",
  "api_key": "NEW_API_KEY",
  "secret_key": "NEW_SECRET_KEY",
  "expired_at": "2027-01-01T00:00:00Z"
}
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "account_id": 1,
    "alias": "새 별칭",
    "updated_at": "2026-06-02T14:00:00Z"
  }
}
```

---

### `DELETE /accounts/{account_id}` — 계좌 삭제 (소프트 삭제)

- **인증:** Bearer Access Token (소유권 확인 필수)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "계좌가 삭제되었습니다." }
}
```

> [!note] 소프트 삭제 정책
> `deleted_at = NOW()`로 마킹. 해당 계좌의 주문 이력은 보존된다. 연결된 Trading Rule은 `is_active = FALSE`로 자동 변경.

---

### `PATCH /accounts/{account_id}/kill-switch` — 계좌별 Kill-Switch

- **인증:** Bearer Access Token (소유권 확인 필수)

**Request:**
```json
{ "is_active": false }
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "account_id": 1,
    "is_active": false,
    "message": "계좌 자동매매가 즉시 중단됩니다."
  }
}
```

**내부 처리 순서:**
1. PostgreSQL UPDATE `is_active`
2. Redis PUBLISH `user:{uid}:events` (Celery Worker 즉시 반응)
3. WebSocket PUSH → 클라이언트 UI 업데이트

---

### `POST /system/kill-switch` — 글로벌 Kill-Switch (전체 계좌 즉시 정지)

- **인증:** Bearer Access Token

**Request:**
```json
{ "activate": true }
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "stopped_accounts": 3,
    "message": "모든 자동매매 봇이 즉시 중단되었습니다."
  }
}
```

---

## 3. 자동 매매 룰 (Trading Rules) — `/rules`

### `GET /rules` — 룰 목록 조회

- **인증:** Bearer Access Token

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `account_id` | integer | — | 특정 계좌 필터 |
| `is_active` | boolean | — | 활성 룰만 필터 |
| `is_paper_mode` | boolean | — | 모의투자 룰만 필터 |
| `page` | integer | 1 | 페이지 번호 |
| `size` | integer | 20 | 페이지 크기 (최대 100) |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "rule_id": 5,
        "account_id": 1,
        "rule_name": "비트코인 골든크로스 추세매매",
        "conditions": {
          "ticker": "KRW-BTC",
          "indicator": "MA_CROSS",
          "short_term": 15,
          "long_term": 50,
          "trigger": "UPWARD",
          "interval": "1m"
        },
        "actions": {
          "order_side": "BUY",
          "amount_type": "PERCENTAGE",
          "value": 10,
          "order_type": "MARKET"
        },
        "is_active": true,
        "is_paper_mode": false,
        "created_at": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 1,
    "page": 1,
    "size": 20,
    "pages": 1
  }
}
```

---

### `POST /rules` — 룰 생성

- **인증:** Bearer Access Token

**Request:**
```json
{
  "account_id": 1,
  "rule_name": "비트코인 골든크로스 추세매매",
  "conditions": {
    "ticker": "KRW-BTC",
    "indicator": "MA_CROSS",
    "short_term": 15,
    "long_term": 50,
    "trigger": "UPWARD",
    "interval": "1m"
  },
  "actions": {
    "order_side": "BUY",
    "amount_type": "PERCENTAGE",
    "value": 10,
    "order_type": "MARKET",
    "stop_loss_pct": 3.0,
    "take_profit_pct": 5.0
  },
  "is_paper_mode": false
}
```

**지원 지표 (`indicator`) 목록 (v1 범위):**

| 값 | 설명 | 필수 파라미터 |
| --- | --- | --- |
| `MA_CROSS` | 이동평균선 골든/데드크로스 | `short_term`, `long_term`, `trigger` (UPWARD/DOWNWARD) |
| `RSI` | RSI 과매수/과매도 | `period`, `threshold`, `trigger` (ABOVE/BELOW) |
| `BOLLINGER_BAND` | 볼린저밴드 이탈 | `period`, `std_dev`, `trigger` (UPPER/LOWER) |
| `PRICE_BREAKOUT` | 특정 가격 돌파 | `price`, `trigger` (ABOVE/BELOW) |

**`amount_type` 허용값:**

| 값 | 설명 |
| --- | --- |
| `PERCENTAGE` | 가용 잔고의 `value`% |
| `FIXED_KRW` | 고정 금액 (원화) |
| `FIXED_QUANTITY` | 고정 수량 |

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "rule_id": 5,
    "rule_name": "비트코인 골든크로스 추세매매",
    "is_active": true,
    "created_at": "2026-06-02T14:00:00Z"
  }
}
```

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| 타인 소유 account_id | `ERR_AUTHZ_001` | 403 |
| 지원하지 않는 indicator | `ERR_VALID_001` | 422 |

---

### `GET /rules/{rule_id}` — 룰 상세 조회

- **인증:** Bearer Access Token (소유권 확인)

**Response `200 OK`:** (룰 생성 응답과 동일한 전체 필드)

---

### `PUT /rules/{rule_id}` — 룰 수정

- **인증:** Bearer Access Token (소유권 확인)

**Request:** (수정할 필드만 포함)
```json
{
  "rule_name": "새 룰 이름",
  "conditions": { ... },
  "actions": { ... }
}
```

> [!warning] 활성 룰 수정 정책
> `is_active = TRUE`인 룰을 수정하면 수정 전 Celery Worker에서 실행 중인 태스크를 안전하게 종료하고, 수정된 룰로 재시작한다.

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "rule_id": 5,
    "updated_at": "2026-06-02T15:00:00Z"
  }
}
```

---

### `DELETE /rules/{rule_id}` — 룰 삭제 (소프트 삭제)

- **인증:** Bearer Access Token (소유권 확인)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": { "message": "매매 룰이 삭제되었습니다." }
}
```

---

### `PATCH /rules/{rule_id}/toggle` — 룰 활성/비활성 토글

- **인증:** Bearer Access Token

**Request:**
```json
{ "is_active": false }
```

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "rule_id": 5,
    "is_active": false
  }
}
```

---

## 4. 종목 마스터 (Stocks) — `/stocks`

### `GET /stocks` — 종목 목록 조회

- **인증:** Bearer Access Token

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `market_type` | string | — | `KOSPI`, `KOSDAQ`, `UPBIT`, `BINANCE` |
| `is_trading` | boolean | true | 거래 가능 종목만 |
| `q` | string | — | 종목명 또는 티커 검색 |
| `page` | integer | 1 | |
| `size` | integer | 50 | 최대 200 |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "stock_id": 1,
        "ticker": "KRW-BTC",
        "name": "Bitcoin",
        "market_type": "UPBIT",
        "is_trading": true
      }
    ],
    "total": 1,
    "page": 1,
    "size": 50,
    "pages": 1
  }
}
```

---

### `GET /stocks/{ticker}` — 종목 상세 조회

- **인증:** Bearer Access Token

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "stock_id": 1,
    "ticker": "KRW-BTC",
    "name": "Bitcoin",
    "market_type": "UPBIT",
    "is_trading": true,
    "updated_at": "2026-06-02T00:00:00Z"
  }
}
```

---

## 5. 관심종목 (Watchlists) — `/watchlists`

### `GET /watchlists` — 관심목록 그룹 조회

- **인증:** Bearer Access Token

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "watchlist_id": 1,
        "group_name": "성장주 테마",
        "item_count": 5,
        "created_at": "2026-06-01T10:00:00Z"
      }
    ],
    "total": 1
  }
}
```

---

### `POST /watchlists` — 관심목록 그룹 생성

**Request:**
```json
{ "group_name": "코인 단기 스윙" }
```

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "watchlist_id": 2,
    "group_name": "코인 단기 스윙"
  }
}
```

---

### `GET /watchlists/{watchlist_id}` — 관심목록 상세 (종목 포함)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "watchlist_id": 1,
    "group_name": "성장주 테마",
    "items": [
      {
        "stock_id": 1,
        "ticker": "KRW-BTC",
        "name": "Bitcoin",
        "added_at": "2026-06-01T10:00:00Z"
      }
    ]
  }
}
```

---

### `DELETE /watchlists/{watchlist_id}` — 관심목록 그룹 삭제

**Response `200 OK`:**
```json
{ "success": true, "data": { "message": "관심목록이 삭제되었습니다." } }
```

---

### `POST /watchlists/{watchlist_id}/items` — 종목 추가

**Request:**
```json
{ "ticker": "KRW-BTC" }
```

**Response `201 Created`:**
```json
{
  "success": true,
  "data": {
    "watchlist_id": 1,
    "stock_id": 1,
    "ticker": "KRW-BTC"
  }
}
```

---

### `DELETE /watchlists/{watchlist_id}/items/{stock_id}` — 종목 제거

**Response `200 OK`:**
```json
{ "success": true, "data": { "message": "종목이 관심목록에서 제거되었습니다." } }
```

---

## 6. 매매 로그 (Logs) — `/logs`

> **데이터 소스:** Elasticsearch `myfin-trade-logs-*` 인덱스. FastAPI가 ES DSL 쿼리를 생성하여 Proxy.

### `GET /logs/trades` — 체결 로그 목록 (페이지네이션)

- **인증:** Bearer Access Token

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `account_id` | integer | — | 계좌 필터 |
| `rule_id` | integer | — | 룰 필터 |
| `ticker` | string | — | 종목 필터 |
| `order_side` | string | — | `BUY` 또는 `SELL` |
| `status` | string | — | 상태 필터 |
| `start_date` | date | — | `YYYY-MM-DD` |
| `end_date` | date | — | `YYYY-MM-DD` |
| `is_paper` | boolean | — | 모의투자 필터 |
| `page` | integer | 1 | |
| `size` | integer | 20 | 최대 100 |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "order_uid": "20260602-KIS-0001",
        "account_id": 1,
        "rule_id": 5,
        "rule_name": "비트코인 골든크로스 추세매매",
        "ticker": "KRW-BTC",
        "order_side": "BUY",
        "requested_price": 85000000,
        "requested_quantity": 0.001,
        "filled_price": 85050000,
        "filled_quantity": 0.001,
        "status": "SUCCESS",
        "is_paper": false,
        "execution_latency_ms": 124,
        "ordered_at": "2026-06-02T13:45:00Z",
        "filled_at": "2026-06-02T13:45:00.124Z"
      }
    ],
    "total": 45,
    "page": 1,
    "size": 20,
    "pages": 3
  }
}
```

---

### `GET /logs/trades/{order_uid}` — 체결 로그 상세

**Response `200 OK`:** (위 항목 + `error_message` 필드 포함)

---

### `GET /logs/errors` — 에러 로그만 필터링 조회

**Query Parameters:** `/logs/trades`와 동일 (status 고정: `ERROR_API`, `ERROR_FUNDS`, `TIMEOUT`)

---

## 7. 대시보드 (Dashboard) — `/dashboard`

### `GET /dashboard/summary` — 통합 자산 현황 요약

- **인증:** Bearer Access Token
- **데이터 소스:** PostgreSQL `account_balance_snapshots` (최근 스냅샷 집계)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "total_balance_krw": 15000000,
    "daily_return_pct": 1.2,
    "active_bots_count": 3,
    "total_accounts": 2,
    "accounts": [
      {
        "account_id": 1,
        "alias": "메인 국장 계좌",
        "broker_name": "korea_investment",
        "total_balance": 10000000,
        "currency": "KRW",
        "is_active": true,
        "snapshot_at": "2026-06-02T15:30:00Z"
      }
    ]
  }
}
```

---

## 8. AI 보고서 (Reports) — `/reports`

### `GET /reports` — 보고서 목록

- **인증:** Bearer Access Token

**Query Parameters:**

| 파라미터 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `period_type` | string | — | `WEEKLY`, `MONTHLY`, `CUSTOM` |
| `page` | integer | 1 | |
| `size` | integer | 10 | |

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "report_id": 10,
        "title": "2026년 5월 4주차 트레이딩 분석",
        "period_type": "WEEKLY",
        "period_start": "2026-05-25",
        "period_end": "2026-05-31",
        "performance_summary": {
          "win_rate": 65.5,
          "total_return_pct": 2.4,
          "total_trades": 38
        },
        "created_at": "2026-05-31T09:00:00Z"
      }
    ],
    "total": 10,
    "page": 1,
    "size": 10,
    "pages": 1
  }
}
```

---

### `GET /reports/{report_id}` — 보고서 상세 (마크다운 내용 포함)

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "report_id": 10,
    "title": "2026년 5월 4주차 트레이딩 분석",
    "period_type": "WEEKLY",
    "period_start": "2026-05-25",
    "period_end": "2026-05-31",
    "content_markdown": "## 이번 주 요약\n\n비트코인 골든크로스 전략이 ...",
    "performance_summary": { ... },
    "created_at": "2026-05-31T09:00:00Z"
  }
}
```

---

### `POST /reports/generate` — AI 보고서 수동 생성 트리거

- **인증:** Bearer Access Token
- **Rate Limit:** 사용자당 1회/시간

**Request:**
```json
{
  "period_type": "WEEKLY",
  "period_start": "2026-05-25",
  "period_end": "2026-05-31"
}
```

**Response `202 Accepted`:**
```json
{
  "success": true,
  "data": {
    "task_id": "celery-task-uuid",
    "message": "보고서 생성이 시작되었습니다. 완료 시 알림을 발송합니다.",
    "estimated_seconds": 30
  }
}
```

**에러:**

| 조건 | 에러 코드 | HTTP |
| --- | --- | --- |
| 이미 생성 중인 보고서 존재 | `ERR_REPORT_002` | 409 |
| Rate Limit 초과 (1회/시간) | — | 429 |

---

## 9. 알림 설정 (Notifications) — `/notifications`

### `GET /notifications/settings` — 알림 설정 조회

**Response `200 OK`:**
```json
{
  "success": true,
  "data": {
    "discord_webhook_url": "https://discord.com/api/webhooks/****",
    "slack_webhook_url": null,
    "events": {
      "ORDER_SUCCESS": true,
      "ORDER_ERROR": true,
      "API_KEY_EXPIRY_7D": true,
      "API_KEY_EXPIRY_3D": true,
      "API_KEY_EXPIRY_TODAY": true,
      "CIRCUIT_BREAKER_OPEN": true,
      "AI_REPORT_READY": true,
      "KILL_SWITCH_ACTIVATED": true
    }
  }
}
```

---

### `PUT /notifications/settings` — 알림 설정 변경

**Request:**
```json
{
  "discord_webhook_url": "https://discord.com/api/webhooks/...",
  "events": {
    "ORDER_SUCCESS": false
  }
}
```

**Response `200 OK`:**
```json
{ "success": true, "data": { "message": "알림 설정이 업데이트되었습니다." } }
```

---

## 10. WebSocket — 실시간 이벤트 스트림

### `WS /ws/events` — 실시간 이벤트 구독

- **인증:** Query Parameter `?token=<access_token>` (WebSocket은 Authorization 헤더 지원 불일치)
- **프로토콜:** WebSocket (WSS 필수)

**연결 후 수신 이벤트 포맷:**
```json
{
  "event": "ORDER_EXECUTED",
  "timestamp": "2026-06-02T13:45:00Z",
  "data": {
    "account_id": 1,
    "ticker": "KRW-BTC",
    "order_side": "BUY",
    "status": "SUCCESS",
    "filled_price": 85050000
  }
}
```

**이벤트 타입 목록:**

| event | 트리거 조건 |
| --- | --- |
| `ORDER_EXECUTED` | 주문 체결 완료 또는 실패 |
| `KILL_SWITCH_CHANGED` | Kill-Switch 상태 변경 |
| `CIRCUIT_BREAKER_STATE` | Circuit Breaker 상태 전환 |
| `BOT_STATUS_CHANGED` | Celery 봇 시작/중지 |
| `REPORT_READY` | AI 보고서 생성 완료 |

**Ping/Pong:** 서버는 30초마다 Ping 전송. 클라이언트는 Pong 응답 필수. 60초 무응답 시 연결 종료.

---

## 11. 시스템 헬스체크 — `/health`

### `GET /health` — 시스템 상태 확인

- **인증:** 불필요 (공개)

**Response `200 OK`:**
```json
{
  "status": "healthy",
  "timestamp": "2026-06-02T13:00:00Z",
  "components": {
    "database": "ok",
    "redis": "ok",
    "elasticsearch": "ok"
  }
}
```

**Response `503 Service Unavailable`:** (의존 서비스 장애 시)
```json
{
  "status": "degraded",
  "components": {
    "database": "ok",
    "redis": "ok",
    "elasticsearch": "error"
  }
}
```
