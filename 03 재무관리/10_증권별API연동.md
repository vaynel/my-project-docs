## 목적
MyFin 2.0에서 **증권사별 API 연동 방식 차이**를 흡수하기 위한 비교 문서입니다.  
계좌 등록/토큰 발급/주문/조회/에러처리/레이트리밋/실전·모의 환경 분리 등을 공통 인터페이스로 수렴시키기 위한 기준을 제공합니다.

> 참고: 본 문서는 `docs/ref_01_한국투자증권_API.xlsx`, `docs/ref_02_키움증권_REST_API.xlsx` 기반으로 정리되었습니다.

---

## 용어
- **KIS**: 한국투자증권(한국투자) API
- **KIWOOM**: 키움증권 REST API
- **Access Token**: API 호출을 위한 단기 토큰 (Bearer 등)
- **App Key / Secret**: 클라이언트 식별/서명용 키
- **실전/모의**: 운영 환경 분리 (Base URL, 헤더, 계좌번호 체계가 다를 수 있음)

---


## 한눈에 보는 차이 요약

### 인증/토큰 방식
- **KIS**
  - **키 구성**: AppKey + AppSecret 기반
  - **토큰**: 일반적으로 **OAuth2 Client Credentials**(또는 유사)로 Access Token 발급
  - **추가 헤더**: `tr_id` 같은 트랜잭션 식별자가 필요할 가능성 높음
  - **토큰 만료**: 만료 시간 기반 갱신 로직 필요
- **KIWOOM**
  - **키 구성**: `appkey` + `secretkey`
  - **토큰**: `/oauth2/token`로 **client_credentials** 발급
  - **추가 헤더(전 구간 공통)**: `api-id`(7자리 TR코드) + `authorization`(Bearer 토큰) + 연속조회(`cont-yn`, `next-key`)

### 환경(실전/모의) 분리
- **KIS**: 실전/모의 **Domain 분리**
  - 실전: `https://openapi.koreainvestment.com:9443`
  - 모의: `https://openapivts.koreainvestment.com:29443`
- **KIWOOM**: 실전/모의 **Domain 분리**
  - 실전: `https://api.kiwoom.com`
  - 모의: `https://mockapi.kiwoom.com` (**KRX만 지원가능**)

### 요청 공통 규칙
- **KIS**: 엔드포인트별 `tr_id`가 **필수**이며, `appkey/appsecret/authorization` 포함. (주문 시 `custtype` 필수)
- **KIWOOM**: 모든 호출에 `api-id`(TR코드) + `authorization` + (연속조회 시) `cont-yn`, `next-key`

### 레이트리밋/차단
- **KIS**: 초당 호출 제한/연속 호출 차단 등 정책 가능 → 백오프/큐잉 필요
- **KIWOOM**: 오류코드로 제한이 명시됨
  - `1700`: 허용된 요청 개수 초과
  - `1687`: 재귀 호출 발생으로 제한

---

## 공통 아키텍처 권장안
증권사별 차이를 아래 3계층으로 흡수합니다.

### 1) `BrokerAdapter` 인터페이스(권장)
- **역할**: “앱 내부 표준 모델” ↔ “증권사 API” 변환
- **필수 메서드 예시**
  - `authenticate()` / `refreshTokenIfNeeded()`
  - `placeOrder(order)` / `cancelOrder(orderId)` / `getOrder(orderId)`
  - `getBalances()` / `getPositions()` / `getQuotes(symbols)`
  - `healthCheck()`

### 2) `AuthStrategy`
- **KIS**: Client Credentials 토큰 발급 + 만료 기반 갱신
- **KIWOOM**: 엑셀 기준으로 전략 구현(서명/세션/토큰교환 등)

### 3) `RateLimiter` / `CircuitBreaker`
- 증권사별 정책을 분리하고, 공통으로 백오프/재시도/차단 상태를 노출

---

## 증권사별 상세 비교

### 한국투자증권(KIS)

#### 기본 정보
- **Base URL(실전)**: `https://openapi.koreainvestment.com:9443`
- **Base URL(모의)**: `https://openapivts.koreainvestment.com:29443`
- **인증(토큰 발급)**: `POST /oauth2/tokenP`
- **인증(토큰 폐기)**: `POST /oauth2/revokeP`

#### 인증(토큰 발급)
- **요청 방식**: `POST /oauth2/tokenP`
- **Request Body**
  - `grant_type`: `client_credentials`
  - `appkey`: 앱키
  - `appsecret`: 앱시크릿
- **응답 주요 필드**
  - `access_token`
  - `token_type`: `"Bearer"` (API 호출 시 `"Bearer <token>"` 형태)
  - `expires_in`: (초)
  - `access_token_token_expired`: `"YYYY-MM-DD HH:MM:SS"` 형태 만료 시각 문자열
- **저장/갱신 규칙**
  - **유효기간 24시간**, “1일 1회 발급 원칙”
  - **갱신발급주기 6시간**: 6시간 이내 재호출 시 기존 토큰이 반환될 수 있음(잦은 발급 제어)
  - 만료 \(T\) 이전 선갱신(예: 5~10분) + 401/403 시 1회 재발급 후 재시도(무한루프 방지)

#### 공통 헤더(예시)
- `authorization`: `Bearer <access_token>`
- `appkey`: `<앱키>`
- `appsecret`: `<앱시크릿>`
- `tr_id`: 엔드포인트별 TR_ID (실전/모의 값이 다름)
- (주문 시) `custtype`: `P`(개인) / `B`(법인)

#### 주문/조회 차이 포인트
- 주문/조회마다 `tr_id`가 다르므로 **엔드포인트×(매수/매도)×(실전/모의)** 매핑 테이블로 관리 권장
- **주문(현금)**: `POST /uapi/domestic-stock/v1/trading/order-cash`
  - TR_ID
    - 실전: (매도) `TTTC0011U` / (매수) `TTTC0012U`
    - 모의: (매도) `VTTC0011U` / (매수) `VTTC0012U`
  - **주의**: `POST` Body의 key를 **대문자**로 전달해야 함(문서 명시)
  - **주의**: `ORD_QTY`, `ORD_UNPR` 등을 **String**으로 전달(문서 명시)
- **잔고조회**: `GET /uapi/domestic-stock/v1/trading/inquire-balance`
  - TR_ID: 실전 `TTTC8434R` / 모의 `VTTC8434R`
  - 연속조회
    - Request Header: `tr_cont` (`공백`=초기, `N`=다음)
    - Query: `CTX_AREA_FK100`, `CTX_AREA_NK100`
    - Response Header: `tr_cont`가 `M`이면 다음 데이터 존재
    - Response Body: `ctx_area_fk100`, `ctx_area_nk100` 반환

---

### 키움증권(KIWOOM)

#### 기본 정보
- **Base URL(실전)**: `https://api.kiwoom.com`
- **Base URL(모의)**: `https://mockapi.kiwoom.com` (**KRX만 지원가능**)
- **인증(토큰 발급)**: `POST /oauth2/token` (API ID: `au10001`)
- **인증(토큰 폐기)**: `POST /oauth2/revoke` (API ID: `au10002`)

#### 인증(토큰 발급/세션)
- **요청 방식**: `POST /oauth2/token` (Content-Type: `application/json;charset=UTF-8`)
  - Header
    - `api-id`: TR코드(7자리) — 문서 예: `ka00001`
    - `authorization`: `Bearer <token>` (문서상 Required로 표기되어 있음)
    - `cont-yn`, `next-key`: 연속조회용(선택)
  - Body
    - `grant_type`: `client_credentials`
    - `appkey`
    - `secretkey`
- **응답 주요 필드**
  - `expires_dt`: 만료일(예: `20241107083713`)
  - `token_type`: 예: `bearer`
  - `token`: 접근토큰
  - `return_code`, `return_msg`

#### 공통 헤더/서명(있다면)
- `api-id`: 7자리 TR코드 (예: `ka00001`, `kt10000`)
- `authorization`: `Bearer <token>`
- 연속조회
  - `cont-yn`, `next-key`를 다음 호출에 그대로 전달
  - Response Header에서도 `cont-yn`, `next-key`가 내려옴

#### 주문/조회 차이 포인트
- **주식 매수주문**: `POST /api/dostk/ordr` (API ID: `kt10000`)
  - Body: `dmst_stex_tp`(KRX/NXT/SOR), `stk_cd`, `ord_qty`, `ord_uv`(선택), `trde_tp`, `cond_uv`(선택)
  - Response Body: `ord_no`(7자리 주문번호), `return_code`, `return_msg`
- **예수금상세현황요청**: `POST /api/dostk/acnt` (API ID: `kt00001`)
  - Body: `qry_tp` (3:추정조회, 2:일반조회)
  - Response 예: `entr`(예수금), `ord_alow_amt`(주문가능금액), `pymn_alow_amt`(출금가능금액) 등 (좌측 0-padding 포함 15자리 문자열)

#### 대표 오류코드(문서 “오류코드” 시트)
- **헤더/요청 형식**
  - `1513`: authorization 헤더 필수
  - `1514~1516`: authorization 형식/타입/토큰 누락
- **인증**
  - `8001/8002`: App Key/Secret 검증 실패
  - `8005`: Token 유효하지 않음
  - `8010`: 토큰 발급 IP와 요청 IP 불일치
  - `8030/8031`: 실전/모의 투자구분 불일치(appkey/token 사용 불가)
  - `8103`: 토큰 또는 단말기 인증 실패
- **제한**
  - `1687`: 재귀 호출로 제한
  - `1700`: 허용된 요청 개수 초과
- **모의 제한**
  - `8104`: 모의투자에서 지원하지 않는 API

---

## 공통 데이터 모델(권장)
- **OrderRequest**
  - `broker`: `KIS | KIWOOM`
  - `accountId`
  - `symbol`
  - `side`: `BUY | SELL`
  - `type`: `MARKET | LIMIT`
  - `qty` / `price`
- **OrderResult**
  - `brokerOrderId`
  - `status`
  - `requestedAt`
  - `filledQty` / `avgFillPrice`
- **BrokerError**
  - `code` (증권사 원본)
  - `message` (원본)
  - `category` (AUTH/RATE_LIMIT/VALIDATION/NETWORK/UNKNOWN)
  - `retryable` (boolean)

---

## 체크리스트(실제 연동 전)
- **키/시크릿 보관**: DB 저장 시 암호화(서버 측 KMS 또는 앱 레벨 암호화)
- **토큰 저장**: 만료/갱신/동시 갱신 경쟁 조건(락) 처리
- **레이트리밋**: 증권사별 limiter 설정
- **재시도 정책**: 네트워크/429/5xx vs 비즈니스 에러 구분
- **시간 동기화**: timestamp/nonce 기반 서명이 있다면 NTP 동기화 필요

---

## 구현 시 주의사항(엑셀 기반으로 확정된 포인트)
- **KIS**
  - 주문 Body key는 **대문자**(문서 명시)
  - `tr_id`는 호출 종류/매수·매도/실전·모의에 따라 달라짐 → 코드 상수화 필수
  - 연속조회는 `tr_cont`(헤더) + `CTX_AREA_*`(쿼리/바디) 조합
- **KIWOOM**
  - 모든 호출에 `api-id` 헤더 필요(7자리 TR코드)
  - 연속조회는 `cont-yn`/`next-key`를 헤더로 전달
  - `8030/8031`로 **실전/모의 키/토큰 혼용이 즉시 차단**될 수 있음 → 환경 분리 강제

