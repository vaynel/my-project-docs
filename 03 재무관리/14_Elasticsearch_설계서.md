---
tags: [elasticsearch, logging, index-design, myfin-v2]
created: 2026-06-03
version: "1.0"
status: approved
related:
  - "[[12_동기화_전략_설계]]"
  - "[[13_룰_정의서]]"
  - "[[03_ERD]]"
---

# MyFin V2.0 Elasticsearch 설계서

> **목적:** 체결 로그·룰 발동 로그·동기화 이벤트를 ES에 적재하는 인덱스 구조, 문서 스키마, 운영 정책을 정의합니다.
> **ES 버전:** 8.13.x (xpack.security.enabled=false, 단일 노드)
> **전략:** ES는 "선택적(optional)" 컴포넌트입니다. 미연결 시 핵심 트레이딩 기능은 정상 동작하고 로그만 생략됩니다.

---

## 1. 인덱스 목록

| 인덱스 패턴 | 용도 | 롤오버 주기 |
|---|---|---|
| `myfin-trade-logs-YYYY.MM` | 주문 체결 로그 (실거래 + 모의) | 월별 |
| `myfin-sync-logs-YYYY.MM` | 동기화 이벤트 + 룰 발동 로그 | 월별 |

> 월별 롤오버: 인덱스명에 `YYYY.MM`을 포함해 자동 분리. 검색은 와일드카드 (`myfin-trade-logs-*`) 사용.

---

## 2. `myfin-trade-logs-YYYY.MM`

### 2.1 용도
브로커 API를 통한 실제 주문 실행 결과를 기록합니다. 실거래(`is_paper=false`)와 모의(`is_paper=true`) 모두 포함합니다.

### 2.2 적재 시점
`app/worker/tasks/trading.py` — 주문 처리 완료 직후 `index_trade_log()` 호출

### 2.3 문서 스키마

```jsonc
{
  // ── 시간 ───────────────────────────────────────────────────────────
  "@timestamp":          "2026-06-03T09:15:32.412Z",   // UTC ISO8601 (Kibana 기본 time field)
  "ordered_at":          "2026-06-03T09:15:30.000Z",   // 주문 요청 시각 (DB ordered_at과 동일)
  "filled_at":           "2026-06-03T09:15:32.100Z",   // 체결 완료 시각 (미체결 시 null)

  // ── 식별자 ─────────────────────────────────────────────────────────
  "order_uid":           "REAL-A3F8C2D1E4B5",          // 주문 고유 ID (REAL-/PAPER-/SIM- 접두어)
  "user_id":             1,
  "account_id":          3,
  "rule_id":             7,                             // 발동 룰 ID (null=수동 주문)

  // ── 주문 내용 ──────────────────────────────────────────────────────
  "broker_name":         "korea_investment",            // "korea_investment" | "kiwoom"
  "ticker":              "005930",
  "order_side":          "SELL",                        // "BUY" | "SELL"
  "requested_price":     84000,                         // 요청 가격 (시장가=0)
  "requested_quantity":  50,
  "filled_price":        83900,                         // 실제 체결가 (미체결 시 null)
  "filled_quantity":     50,                            // 실제 체결 수량

  // ── 실행 결과 ──────────────────────────────────────────────────────
  "status":              "SUCCESS",
  // SUCCESS | PENDING | ERROR_API | ERROR_FUNDS | TIMEOUT | CANCELLED
  "is_paper":            false,
  "execution_latency_ms": 412,                          // 주문 요청 → 체결 응답까지 ms
  "error_message":       null                           // 오류 시 메시지 (최대 500자)
}
```

### 2.4 status 값 정의

| status | 설명 |
|---|---|
| `SUCCESS` | 브로커 API 체결 확인 완료 |
| `PENDING` | 주문 전송 후 체결 미확인 |
| `ERROR_API` | 브로커 API 오류 (네트워크, 인증 등) |
| `ERROR_FUNDS` | 잔고 부족 |
| `TIMEOUT` | 체결 확인 타임아웃 |
| `CANCELLED` | 룰 비활성화 등으로 취소 |

### 2.5 매핑 (PUT `myfin-trade-logs-YYYY.MM`)

```json
{
  "mappings": {
    "properties": {
      "@timestamp":           { "type": "date" },
      "ordered_at":           { "type": "date" },
      "filled_at":            { "type": "date" },
      "order_uid":            { "type": "keyword" },
      "user_id":              { "type": "integer" },
      "account_id":           { "type": "integer" },
      "rule_id":              { "type": "integer" },
      "broker_name":          { "type": "keyword" },
      "ticker":               { "type": "keyword" },
      "order_side":           { "type": "keyword" },
      "requested_price":      { "type": "double" },
      "requested_quantity":   { "type": "integer" },
      "filled_price":         { "type": "double" },
      "filled_quantity":      { "type": "integer" },
      "status":               { "type": "keyword" },
      "is_paper":             { "type": "boolean" },
      "execution_latency_ms": { "type": "integer" },
      "error_message":        { "type": "text", "index": false }
    }
  }
}
```

---

## 3. `myfin-sync-logs-YYYY.MM`

### 3.1 용도
두 종류의 이벤트를 하나의 인덱스에 혼합 적재합니다. `event_type` 필드로 구분합니다.

| event_type | 설명 | 적재 위치 |
|---|---|---|
| `SYNC_SUCCESS` / `SYNC_ERROR` 등 | 계좌 동기화 이벤트 | `es_client.py::index_sync_event()` |
| `RULE_TRIGGERED` | 룰 발동 이벤트 | `rule_watcher.py::_log_rule_triggered()` |

### 3.2 문서 스키마 — 동기화 이벤트 (`SYNC_*`)

```jsonc
{
  "@timestamp":      "2026-06-03T09:05:00.000Z",
  "event_type":      "SYNC_SUCCESS",
  // SYNC_SUCCESS | SYNC_ERROR | SYNC_TIMEOUT | SYNC_SKIPPED

  "user_id":         1,
  "account_id":      3,
  "broker_name":     "kiwoom",
  "sync_mode":       "realtime",          // "realtime" | "daily" | "manual"
  "is_paper":        false,

  // 성공 시
  "duration_ms":     823,                 // 동기화 소요 시간
  "positions_count": 5,                   // 동기화된 보유 종목 수
  "total_balance":   15420000.0,          // 총 평가금액 (KRW)

  // 오류 시 (null 생략)
  "error_code":      null,
  "error_message":   null
}
```

### 3.3 문서 스키마 — 룰 발동 이벤트 (`RULE_TRIGGERED`)

```jsonc
{
  "@timestamp":  "2026-06-03T09:15:28.000Z",
  "event_type":  "RULE_TRIGGERED",

  "account_id":  3,
  "rule_id":     7,
  "rule_name":   "매도기본룰(10%)",
  "rule_type":   "TRAILING_STOP",
  // "TRAILING_STOP" | "PRICE_RISE_ALERT" | "PRICE_BUY"

  "ticker":      "307950",
  "action_type": "SELL",
  // "SELL" | "BUY" | "NOTIFY"

  "reason":      "최고가 ₩984,000 대비 10% 하락 → 현재가 ₩808,200",  // 최대 200자
  "price":       808200.0,          // 발동 시점 현재가
  "quantity":    12                  // 수량 (NOTIFY 타입은 null 가능)
}
```

### 3.4 매핑 (PUT `myfin-sync-logs-YYYY.MM`)

```json
{
  "mappings": {
    "properties": {
      "@timestamp":      { "type": "date" },
      "event_type":      { "type": "keyword" },
      "user_id":         { "type": "integer" },
      "account_id":      { "type": "integer" },
      "broker_name":     { "type": "keyword" },
      "sync_mode":       { "type": "keyword" },
      "is_paper":        { "type": "boolean" },
      "duration_ms":     { "type": "integer" },
      "positions_count": { "type": "integer" },
      "total_balance":   { "type": "double" },
      "error_code":      { "type": "keyword" },
      "error_message":   { "type": "text", "index": false },
      "rule_id":         { "type": "integer" },
      "rule_name":       { "type": "keyword" },
      "rule_type":       { "type": "keyword" },
      "ticker":          { "type": "keyword" },
      "action_type":     { "type": "keyword" },
      "reason":          { "type": "text", "index": false },
      "price":           { "type": "double" },
      "quantity":        { "type": "integer" }
    }
  }
}
```

---

## 4. 인덱스 템플릿

매월 새 인덱스 생성 시 매핑이 자동 적용되도록 Index Template을 등록합니다.

```bash
# 체결 로그 템플릿 등록
PUT _index_template/myfin-trade-logs
{
  "index_patterns": ["myfin-trade-logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "30s"
    },
    "mappings": { ... }   // 2.5항의 매핑
  }
}

# 동기화 로그 템플릿 등록
PUT _index_template/myfin-sync-logs
{
  "index_patterns": ["myfin-sync-logs-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "30s"
    },
    "mappings": { ... }   // 3.4항의 매핑
  }
}
```

> **단일 노드 설정:** `number_of_replicas: 0` (복제본 없음), `refresh_interval: 30s` (쓰기 부하 최소화)

---

## 5. 데이터 보존 정책

| 인덱스 | 보존 기간 | 삭제 방식 |
|---|---|---|
| `myfin-trade-logs-*` | 12개월 | 수동 또는 ILM |
| `myfin-sync-logs-*` | 3개월 | 수동 또는 ILM |

```bash
# 3개월 이전 sync-logs 삭제 예시 (cron)
DELETE myfin-sync-logs-2026.01
DELETE myfin-sync-logs-2026.02
```

---

## 6. 현재 코드 vs 문서 불일치 사항 (기술 부채)

| 항목 | 현재 코드 | 설계 기준 | 우선순위 |
|---|---|---|---|
| time field | `trading.py`에서 `"timestamp"` 사용 | `"@timestamp"` 통일 | 중 |
| ES 클라이언트 이중화 | `elasticsearch.py` / `es_client.py` 분리 | 단일 `es_client.py`로 통합 | 중 |
| 인덱스 템플릿 미등록 | 동적 매핑 (자동) | 명시적 매핑 등록 | 중 |
| `ordered_at` 미포함 | trade-logs에 누락 | 추가 필요 | 하 |
| SYNC 이벤트 `user_id` | sync-logs 일부 누락 가능 | 항상 포함 | 하 |

---

## 7. 주요 쿼리 예시

### 특정 계좌의 최근 체결 로그
```json
GET myfin-trade-logs-*/_search
{
  "query": { "bool": { "must": [
    { "term": { "user_id": 1 } },
    { "term": { "account_id": 3 } },
    { "range": { "@timestamp": { "gte": "now-7d" } } }
  ]}},
  "sort": [{ "@timestamp": "desc" }],
  "size": 50
}
```

### 특정 룰의 발동 이력
```json
GET myfin-sync-logs-*/_search
{
  "query": { "bool": { "must": [
    { "term": { "event_type": "RULE_TRIGGERED" } },
    { "term": { "rule_id": 7 } }
  ]}},
  "sort": [{ "@timestamp": "desc" }],
  "size": 20
}
```

### 오늘 에러 현황
```json
GET myfin-trade-logs-*/_search
{
  "query": { "bool": { "must": [
    { "terms": { "status": ["ERROR_API", "ERROR_FUNDS", "TIMEOUT"] } },
    { "range": { "@timestamp": { "gte": "now/d" } } }
  ]}},
  "aggs": {
    "by_status": { "terms": { "field": "status" } }
  },
  "size": 0
}
```

---

## 8. 적재 흐름 요약

```
브로커 체결 완료
  └─ trading.py
       └─ index_trade_log()          → myfin-trade-logs-YYYY.MM

계좌 동기화 완료/실패
  └─ scheduler.py → _sync_single_account()
       └─ index_sync_event()         → myfin-sync-logs-YYYY.MM (event_type: SYNC_*)

룰 평가 → 조건 충족
  └─ rule_watcher.py → _log_rule_triggered()
       └─ es.index()                 → myfin-sync-logs-YYYY.MM (event_type: RULE_TRIGGERED)
```
