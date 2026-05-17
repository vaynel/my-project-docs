# 개발 TODO 리스트 (우선순위 기반)

## M0. 기준선 확정 (1회성)

* [ ] 거래 분류 축 정의 확정

  * [ ] 축 A(카테고리) 기본 목록 작성
  * [ ] 축 B(목적/사용처) 기본 목록 작성
  * [ ] 축 B 복수 허용 여부 정책 확정(기본 1개 + 예외 N개)
* [ ] 거래 상태 머신 확정 (`INGESTED → ENRICHED → CLASSIFIED/REVIEW_REQUIRED → CONFIRMED → POSTED`)
* [ ] 중복 방지 정책 확정(idempotency key 구성 요소 정의)

---

## M1. Staging(임시) 데이터 계층 구축

### 1) 스키마/모델

* [ ] Staging DB 선택/준비(원장 DB와 분리 권장)
* [ ] 테이블 설계 및 생성

  * [ ] `staging_transactions`
  * [ ] `staging_labels` (label_type: CATEGORY/CONTEXT)
  * [ ] `staging_audit_logs` (수정 이력)
  * [ ] `merchant_dictionary` (raw → normalized)
  * [ ] `classification_rules` (룰 기반 분류)
* [ ] 인덱스 설계

  * [ ] `idempotency_key` unique
  * [ ] `status`, `merchant_norm_id`, `datetime` 검색 최적화

### 2) 데이터 표준(정규화)

* [ ] 거래 표준 포맷 정의(내부 공통 JSON 스키마)

  * [ ] amount, currency, datetime, merchant_raw, merchant_norm_id
  * [ ] payment_method, account_hint, source(SMS/CSV/MANUAL)
* [ ] merchant 정규화 규칙 정의(공백/특수문자/브랜드 매핑)

---

## M2. 수집(Ingestion) 경로 최소 구현 (1개만 먼저)

> “SMS/CSV/수동” 중 하나만 먼저 뚫고 end-to-end를 만든다.

* [ ] 수집 API(Webhook) 스펙 정의

  * [ ] 입력 JSON 스키마
  * [ ] 인증 방식(토큰/서명)
* [ ] 수집 이벤트 → `staging_transactions` 저장

  * [ ] idempotency_key 생성 및 중복 차단
  * [ ] 상태 `INGESTED` 기록
* [ ] (선택) 수집 실패/재시도 전략 기록

---

## M3. 정규화(Enrichment) 파이프라인

* [ ] Enrichment 워커(또는 n8n 워크플로) 설계
* [ ] `INGESTED` → `ENRICHED` 변환 처리

  * [ ] merchant_dictionary 기반 `merchant_norm_id` 채우기
  * [ ] 결제수단/계정 힌트 매핑
  * [ ] 시간대/포맷 통일
* [ ] 신규 merchant 발견 시 “정규화 미완료 큐” 생성(운영 편의)

---

## M4. 2축 자동 분류 엔진 v1 (룰 기반)

### 1) 분류 결과 모델

* [ ] `staging_labels`에 아래 저장

  * [ ] CATEGORY(축 A) 추천 1~3개 + confidence
  * [ ] CONTEXT(축 B) 추천 1~3개 + confidence
  * [ ] source=RULE

### 2) 룰 설계

* [ ] 룰 우선순위 정책(merchant_norm_id > 결제수단 > 시간대/요일 > 과거 확정)
* [ ] 룰 테이블(`classification_rules`) 스키마 설계
* [ ] confidence 기준 정책 정의

  * [ ] auto-classify 기준값(예: ≥0.85)
  * [ ] review 기준값(예: <0.85)

### 3) 상태 전이

* [ ] `ENRICHED` → `CLASSIFIED` 또는 `REVIEW_REQUIRED` 전이

---

## M5. 검수/분류 웹 UI (운영 핵심)

> 여기서 “자동 + 사람이 빠르게 확정”이 완성됨.

### 1) 화면/기능

* [ ] 큐 화면: `REVIEW_REQUIRED` 목록(필터/검색)
* [ ] 상세 화면:

  * [ ] 축 A 카테고리 선택(필수, 단일)
  * [ ] 축 B 목적 선택(기본 1개, 필요 시 복수)
  * [ ] 추천(top3) + confidence 노출
  * [ ] “추천대로 확정” 버튼
  * [ ] “수정 후 확정” 버튼
* [ ] 일괄 처리:

  * [ ] 동일 merchant_norm_id 묶음 확정
  * [ ] 최근 N건 빠른 확정

### 2) 기록

* [ ] 확정 시 `staging_audit_logs`에 변경 이력 저장
* [ ] 확정 상태로 전환: `CONFIRMED`

---

## M6. 원장(가계부) 반영(Post) 파이프라인

* [ ] Posting 워커 설계: `CONFIRMED` → 원장 반영 → `POSTED`
* [ ] 매핑 규칙 확정

  * [ ] Firefly Category = 축 A 최종값
  * [ ] Firefly Tags = 축 B 최종값(들)
* [ ] 실패 처리

  * [ ] 원장 반영 실패 시 재시도/데드레터 상태
  * [ ] 동일 거래 중복 반영 방지(원장에도 idempotency 관리)

---

## M7. 재무관리/습관 분석 대시보드(초기 KPI)

> “돈관리 상태”를 실제로 보이게 만드는 단계.

* [ ] KPI 정의 확정(최소 10개)

  * [ ] 월 총지출/총수입/순저축/저축률
  * [ ] 축 A 지출 Top N
  * [ ] 축 B 지출 Top N
  * [ ] 축 A×축 B 매트릭스
  * [ ] 요일/시간대 패턴
  * [ ] 반복 merchant(구독/습관)
* [ ] 데이터 접근 방식 결정

  * [ ] Firefly DB 직접 조회 vs Staging+원장 조합 뷰
* [ ] Grafana 대시보드 v1 구성

---

## M8. 운영 고도화 (정확도/편의성)

* [ ] “확정 결과 → 룰 자동 학습” 기능

  * [ ] merchant_norm_id별 기본 CATEGORY/CONTEXT 자동 추천 강화
* [ ] 목적(축 B) 관리 기능

  * [ ] 목적 추가/병합/이름 변경
* [ ] 예산 기능(선택)

  * [ ] 축 A 예산
  * [ ] 축 B 예산
  * [ ] 축 A×축 B 예산(선택)
* [ ] 알림 정책(예산 초과/급증/이상결제) 설계 및 적용

---

# 추천 시작 순서 (실제로 빠르게 성과 내는 루트)

1. **M1 Staging DB**
2. **M2 수집 경로 1개**(SMS든 CSV든 하나만)
3. **M3 정규화**
4. **M4 룰 기반 분류 v1**
5. **M5 검수 UI**
6. **M6 원장 반영**
7. **M7 대시보드**
