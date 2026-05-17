# 11. 초기 데이터 (Seed Data) 현황

## 개요

airCleanWeb은 초기 데이터를 주입하는 방식이 **두 곳**에 나뉘어 존재한다.

| 구분 | 파일 | 실행 시점 |
|------|------|-----------|
| SQL 초기화 스크립트 | `init-scripts/01-init-database.sql` | Docker 컨테이너 최초 기동 시 자동 실행 |
| Prisma Seed 스크립트 | `backend/prisma/seed.ts` | `npx prisma db seed` 명령 수동 실행 |

> **주의:** 두 스크립트는 같은 데이터를 다루지만 내용에 차이가 있다.  
> 실제 개발·테스트 환경에서는 **Prisma Seed**를 기준으로 사용해야 한다.

---

## 1. SQL 초기화 스크립트 (`init-scripts/01-init-database.sql`)

Docker Compose 실행 시 `docker-entrypoint-initdb.d/` 디렉터리에 마운트되어 **Postgres 컨테이너가 처음 생성될 때 한 번만** 자동 실행된다.

### 역할
- UUID, pg_trgm 확장 기능 활성화
- 테이블 생성 (users, schools, boards, messages, files)
- 인덱스 및 `updated_at` 자동 갱신 트리거 생성
- 개발용 샘플 데이터 삽입

### 삽입되는 학교 데이터

| code | name | region |
|------|------|--------|
| 1001 | 서울초등학교 | 서울 |
| 1002 | 부산중학교 | 부산 |
| 1003 | 대구고등학교 | 대구 |

### 삽입되는 사용자 데이터

| userId | name | role | 비고 |
|--------|------|------|------|
| admin | 관리자 | admin | 비밀번호 해시가 **플레이스홀더**라 로그인 불가 |
| user1 | 김철수 | worker | 비밀번호 해시가 **플레이스홀더**라 로그인 불가 |
| user2 | 이영희 | manager | 비밀번호 해시가 **플레이스홀더**라 로그인 불가 |

> **문제점:** `password_hash` 값이 `$2b$10$example_hash_*` 형태의 더미 값이므로 이 스크립트만으로는 실제 로그인이 불가능하다.

### 삽입되는 게시판·메시지 데이터

- 학교별 샘플 점검 보고서 게시글 3건
- 개발용 채팅 메시지 3건

---

## 2. Prisma Seed 스크립트 (`backend/prisma/seed.ts`)

`package.json`의 `"prisma": { "seed": "node prisma/seed.js" }` 설정에 따라 아래 명령으로 실행한다.

```bash
cd backend
npx prisma db seed
```

### 삽입되는 학교 데이터 (upsert)

| code | name | region |
|------|------|--------|
| 1001 | 서울초등학교 | 서울특별시 |
| 1002 | 부산중학교 | 부산광역시 |
| 1003 | 대구고등학교 | 대구광역시 |

### 삽입되는 사용자 데이터 (upsert, 실제 bcrypt 해시 사용)

| userId | name | role | 비밀번호 | 비고 |
|--------|------|------|----------|------|
| admin | 관리자 | ADMIN | `sbs0303oo$$` | 실제 로그인 가능 |
| ssp67223 | 김재현 | ADMIN | `kjs0303oo$$` | 실제 로그인 가능, 개발자 계정 |
| staff001 | 이직원 | WORKER | `kjs0303oo$$` | 실제 로그인 가능, 테스트용 |

> upsert를 사용하므로 **이미 데이터가 있어도 비밀번호를 포함한 주요 필드를 최신 값으로 업데이트**한다.

### 게시판 데이터

별도 삽입 없음. 게시판은 수동 생성 또는 별도 관리로 처리한다.

---

## 3. 두 스크립트의 차이 요약

| 항목 | SQL 스크립트 | Prisma Seed |
|------|-------------|-------------|
| 실행 방식 | 자동 (Docker 최초 기동) | 수동 (`npx prisma db seed`) |
| 스키마 관리 | SQL DDL 직접 작성 | Prisma Schema 기반 |
| 비밀번호 | 더미 해시 (로그인 불가) | 실제 bcrypt 해시 (로그인 가능) |
| 사용자 계정 | admin, user1, user2 | admin, ssp67223, staff001 |
| 중복 처리 | `ON CONFLICT DO NOTHING` | `upsert` (기존 데이터 업데이트) |
| 게시판·메시지 | 샘플 3건씩 삽입 | 삽입 없음 |

---

## 4. 개발 환경 초기 설정 순서

```bash
# 1. DB 컨테이너 기동 (SQL 스크립트 자동 실행)
docker compose -f docker-compose-dev-db.yml up -d

# 2. Prisma 마이그레이션 적용 (스키마 동기화)
cd backend
npx prisma migrate deploy

# 3. Seed 실행 (실제 로그인 가능한 계정 생성)
npx prisma db seed
```

> `npx prisma migrate dev`를 사용하는 경우 migrate 완료 후 seed가 자동으로 실행된다.

---

## 5. 주의 사항

- SQL 초기화 스크립트는 Prisma Schema와 **별도로 관리**되므로, 스키마 변경 시 두 파일 모두 업데이트해야 한다.
- 현재 SQL 스크립트의 테이블 구조는 Prisma Schema와 **완전히 일치하지 않는다** (예: `report_files`, `audit_logs` 등 누락). 실제 스키마는 Prisma Migration이 관리한다.
- 운영 환경에서는 SQL 초기화 스크립트의 샘플 데이터 삽입 구문을 **반드시 제거**해야 한다.
- Seed 스크립트에 **실제 비밀번호가 평문으로 포함**되어 있으므로 외부에 노출되지 않도록 주의한다.
