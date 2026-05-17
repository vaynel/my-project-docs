## 1. 엔티티 관계도 (ERD)

> 💡 **[ERD 다이어그램 삽입 위치]**

---

## 2. 테이블 명세서 (Table Specification)

### 2-1. `user` — 사용자

> 학교 담당자 계정. 보고서 조회 및 다운로드 권한 보유.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | 사용자 고유 ID |
| username | VARCHAR(50) | UNIQUE, NOT NULL | - | 로그인 아이디 |
| email | VARCHAR(100) | UNIQUE, NOT NULL | - | 이메일 (2차 인증 OTP 수신용) |
| password | VARCHAR(255) | NOT NULL | - | 해시된 비밀번호 (PBKDF2 + Salt) |
| school_id | INT | FK(school.id), NOT NULL | - | 소속 학교 |
| role | ENUM('user','admin') | NOT NULL | 'user' | 권한 |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 계정 생성일시 |
| updated_at | DATETIME | - | ON UPDATE CURRENT_TIMESTAMP | 정보 수정일시 |

---

### 2-2. `admin` — 관리자

> 서비스 내부 관리자 계정. 보고서 등록·수정·삭제, 사용자 관리 권한 보유.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | 관리자 고유 ID |
| admin_id | VARCHAR(50) | UNIQUE, NOT NULL | - | 관리자 로그인 아이디 |
| email | VARCHAR(100) | UNIQUE, NOT NULL | - | 이메일 (2차 인증 OTP 수신용) |
| password | VARCHAR(255) | NOT NULL | - | 해시된 비밀번호 (PBKDF2 + Salt) |
| role | ENUM('admin') | NOT NULL | 'admin' | 권한 (항상 admin) |
| last_login | DATETIME | - | NULL | 마지막 로그인 일시 |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 계정 생성일시 |

---

### 2-3. `school` — 학교

> 보고서의 수신 단위. 한 학교에 여러 user가 매핑될 수 있음.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | 학교 고유 ID |
| school_name | VARCHAR(100) | NOT NULL | - | 학교 이름 |
| region | VARCHAR(50) | - | NULL | 지역 (시/도) |
| contact_number | VARCHAR(20) | - | NULL | 학교 대표 연락처 |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 등록일시 |

---

### 2-4. `report` — 보고서

> 관리자가 등록하는 보고서 메타 정보. 학교 단위로 공개.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | 보고서 고유 ID |
| title | VARCHAR(200) | NOT NULL | - | 보고서 제목 |
| content | TEXT | - | NULL | 보고서 설명 / 본문 |
| school_id | INT | FK(school.id), NOT NULL | - | 대상 학교 |
| admin_id | INT | FK(admin.id), NOT NULL | - | 등록한 관리자 |
| status | ENUM('draft','published','archived') | NOT NULL | 'draft' | 공개 상태 |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 등록일시 |
| updated_at | DATETIME | - | ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

---

### 2-5. `report_file` — 보고서 파일

> 보고서에 첨부된 실제 파일 정보. 로컬 디스크 저장 후 경로 관리.

| 컬럼명         | 타입           | 제약                      | 기본값               | 설명                                                                       |
| ----------- | ------------ | ----------------------- | ----------------- | ------------------------------------------------------------------------ |
| id          | INT          | PK, AUTO_INCREMENT      | -                 | 파일 고유 ID                                                                 |
| report_id   | INT          | FK(report.id), NOT NULL | -                 | 연결된 보고서                                                                  |
| file_name   | VARCHAR(255) | NOT NULL                | -                 | 원본 파일명                                                                   |
| file_path   | VARCHAR(500) | NOT NULL                | -                 | 서버 내 저장 경로 (`{FILE_STORAGE_ROOT}/{school_name}/{YYYY-MM-DD}/{filename}`) |
| file_url    | VARCHAR(500) | NOT NULL                | -                 | 클라이언트 접근 URL                                                             |
| file_size   | BIGINT       | NOT NULL                | -                 | 파일 크기 (bytes)                                                            |
| file_type   | VARCHAR(100) | -                       | NULL              | MIME 타입 (예: `application/pdf`)                                           |
| file_hash   | CHAR(64)     | NOT NULL                | -                 | SHA-256 해시값 (업로드 시 서버가 계산, 무결성 검증용)                                      |
| uploaded_at | DATETIME     | NOT NULL                | CURRENT_TIMESTAMP | 업로드 일시                                                                   |

---

### 2-6. `contact` — 고객 문의

> 랜딩 페이지 문의 폼 제출 데이터. 관리자가 확인.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | 문의 고유 ID |
| name | VARCHAR(50) | NOT NULL | - | 문의자 이름 |
| email | VARCHAR(100) | NOT NULL | - | 이메일 주소 |
| phone | VARCHAR(20) | - | NULL | 연락처 |
| message | TEXT | NOT NULL | - | 문의 내용 |
| is_read | TINYINT(1) | NOT NULL | 0 | 읽음 여부 (0: 미확인, 1: 확인) |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 제출 일시 |

---

### 2-7. `auth_otp` — 이메일 2차 인증

> 로그인 1단계 성공 후 이메일로 발송되는 OTP 정보. 5분 내 미사용 시 자동 만료.

| 컬럼명 | 타입 | 제약 | 기본값 | 설명 |
|---|---|---|---|---|
| id | INT | PK, AUTO_INCREMENT | - | OTP 고유 ID |
| target_id | INT | NOT NULL | - | 인증 대상 ID (user.id 또는 admin.id) |
| target_type | ENUM('user','admin') | NOT NULL | - | 인증 대상 유형 |
| otp_code | VARCHAR(255) | NOT NULL | - | 해시된 6자리 OTP (SHA-256) |
| session_token | VARCHAR(255) | UNIQUE, NOT NULL | - | 1단계↔2단계 연결용 임시 토큰 (UUID) |
| expires_at | DATETIME | NOT NULL | - | 만료 일시 (발급 후 5분) |
| is_used | TINYINT(1) | NOT NULL | 0 | 사용 여부 (0: 미사용, 1: 사용됨) |
| created_at | DATETIME | NOT NULL | CURRENT_TIMESTAMP | 발급 일시 |

---

## 3. 관계 정의 (Relationships)

| From         | To          | 관계    | 설명                                    |
| ------------ | ----------- | ----- | ------------------------------------- |
| school       | user        | 1 : N | 한 학교에 여러 사용자 매핑 가능                    |
| school       | report      | 1 : N | 한 학교에 여러 보고서 등록 가능                    |
| admin        | report      | 1 : N | 한 관리자가 여러 보고서 등록 가능                   |
| report       | report_file | 1 : N | 한 보고서에 여러 파일 첨부 가능                    |
| user / admin | auth_otp    | 1 : N | 로그인 시도마다 OTP 레코드 생성 (만료된 레코드는 주기적 삭제) |
|              |             |       |                                       |
