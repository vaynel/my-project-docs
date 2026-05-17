## API 전체 목록
	
![[API_xlsx]]

---

## 공통 규칙

| 항목           | 내용                                                       |
| ------------ | -------------------------------------------------------- |
| Base URL     | `/api/v1`                                                |
| 인증 방식        | JWT Bearer Token (`Authorization: Bearer <accessToken>`) |
| Content-Type | `application/json`                                       |
| 인코딩          | UTF-8                                                    |
| 통신           | HTTPS 전용                                                 |

### 공통 응답 형식

**성공**
```json
{
  "success": true,
  "data": { },
  "message": "처리 완료"
}
```

**실패**
```json
{
  "success": false,
  "statusCode": 400,
  "message": "에러 메시지",
  "error": "Bad Request"
}
```

### 권한 등급

| 등급 | 설명 |
|---|---|
| `public` | 인증 없이 접근 가능 |
| `user` | 로그인한 일반 사용자 |
| `admin` | 관리자 전용 |

---

## KISA 보안 원칙

> KISA(한국인터넷진흥원) 「암호 알고리즘 및 키 길이 이용 안내서」, 「안전한 비밀번호 저장」 권고 기준 적용

### 비밀번호 저장

| 항목    | 적용 기준                        |
| ----- | ---------------------------- |
| 알고리즘  | PBKDF2-HMAC-SHA256           |
| 반복 횟수 | 310,000회 이상                  |
| Salt  | 암호학적 난수(CSPRNG), 사용자별 고유 값   |
| 금지 방식 | MD5, SHA-1, SHA-256 단독 사용 금지 |

### JWT 토큰 설계

| 구분 | Access Token | Refresh Token |
|---|---|---|
| 서명 알고리즘 | **RS256** (비대칭키, KISA 권고) | RS256 |
| 만료 시간 | **15분** | **7일** |
| 전달 방식 | Response Body | **HttpOnly + Secure 쿠키** |
| 클라이언트 저장 위치 | 메모리(변수)만 허용. `localStorage` / `sessionStorage` **금지** | 브라우저 쿠키 (JS 접근 불가) |
| 서버 저장 | 저장 안 함 (stateless) | **DB 저장** (무효화 가능) |

> **왜 Refresh Token을 쿠키로?**
> Response body로 내려주면 JS에서 읽을 수 있어 XSS 공격 시 탈취 위험.
> `HttpOnly` 쿠키는 JS가 접근 자체를 할 수 없어 XSS 차단.
> `SameSite=Strict`로 CSRF도 방어.

### 2차 인증 (이메일 OTP)

로그인은 **2단계**로 처리됩니다.

| 단계 | 엔드포인트 | 처리 내용 |
|---|---|---|
| 1단계 | `POST /auth/login` 또는 `POST /auth/admin/login` | 아이디+비밀번호 검증 → OTP 이메일 발송 → `sessionToken` 반환 |
| 2단계 | `POST /auth/verify-otp` | `sessionToken` + OTP 검증 → Access Token 발급 + Refresh Token 쿠키 설정 |

- OTP: **6자리 숫자**, 유효시간 **5분**
- OTP는 DB에 SHA-256 해시로 저장
- `sessionToken`: UUID 기반 임시 토큰, 5분 만료, 1회 사용 후 즉시 무효화
- OTP 연속 **5회 오류 시 세션 무효화** (재로그인 필요)

### 브루트포스 방어

- 동일 계정 로그인 1단계 **5회 연속 실패 시 30분 잠금**
- 잠금 상태에서 요청 시 `429 Too Many Requests` 반환

---

## 파일 저장 구조 (File Storage)

### 환경변수 설정 (`.env`)

```env
# 파일 저장 루트 경로 (서버 절대 경로)
FILE_STORAGE_ROOT=/var/data/reports

# 클라이언트에 노출되는 파일 서빙 기본 URL
FILE_BASE_URL=/files
```

### 디렉토리 구조

```
{FILE_STORAGE_ROOT}/
└── {school_name}/          ← 학교명 (공백→언더스코어, 특수문자 제거)
    └── {YYYY-MM-DD}/       ← 업로드 날짜
        └── {filename}      ← 중복 방지 UUID 접두사 포함 원본 파일명
```

**예시**
```
/var/data/reports/
└── 한강초등학교/
    └── 2026-05-11/
        └── a1b2c3d4_report_q1.pdf
```

**클라이언트 접근 URL**
```
GET /files/한강초등학교/2026-05-11/a1b2c3d4_report_q1.pdf
```

> `school_name`은 URL-safe 처리 후 디렉토리명으로 사용. Nginx에서 `/files/` 경로를 `FILE_STORAGE_ROOT`로 내부 라우팅.

---

## 1. 인증 (Auth)

### POST `/auth/login` — 사용자 로그인 (1단계)

| 항목 | 내용 |
|---|---|
| 권한 | public |
| 설명 | 아이디+비밀번호 검증 후 OTP를 등록된 이메일로 발송. JWT는 아직 발급하지 않음 |
| 실패 잠금 | 5회 연속 실패 시 30분 계정 잠금 |

**Request Body**
```json
{
  "username": "user01",
  "password": "plainPassword123"
}
```

**Response `200`** *(OTP 발송 완료, JWT 미발급)*
```json
{
  "success": true,
  "data": {
    "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
    "expiresIn": 300,
    "message": "등록된 이메일로 인증번호를 전송했습니다."
  }
}
```

> `sessionToken`은 2단계(`/auth/verify-otp`) 호출 시 필요. 5분 후 만료.

**Error Cases**

| 상황 | 코드 |
|---|---|
| username 또는 password 불일치 | 401 |
| 계정 잠금 (5회 실패) | 429 |

---

### POST `/auth/admin/login` — 관리자 로그인 (1단계)

| 항목 | 내용 |
|---|---|
| 권한 | public |
| 설명 | admin_id + password 검증 후 OTP를 등록된 이메일로 발송. JWT는 아직 발급하지 않음 |
| 실패 잠금 | 5회 연속 실패 시 30분 계정 잠금 |

**Request Body**
```json
{
  "admin_id": "admin",
  "password": "plainPassword123"
}
```

**Response `200`** *(OTP 발송 완료, JWT 미발급)*
```json
{
  "success": true,
  "data": {
    "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
    "expiresIn": 300,
    "message": "등록된 이메일로 인증번호를 전송했습니다."
  }
}
```

---

### POST `/auth/verify-otp` — 이메일 OTP 2차 인증 (2단계)

| 항목 | 내용 |
|---|---|
| 권한 | public (sessionToken 필요) |
| 설명 | 1단계에서 받은 `sessionToken` + 이메일로 수신한 6자리 OTP 검증. 성공 시 JWT 발급 |
| OTP 오류 잠금 | 5회 연속 오류 시 sessionToken 무효화 (재로그인 필요) |

**Request Body**
```json
{
  "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
  "otpCode": "483921"
}
```

**Response `200`**

*Response Header (Refresh Token 쿠키 설정)*
```
Set-Cookie: refresh_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh; Max-Age=604800
```

*Response Body (Access Token 발급)*
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "tokenType": "Bearer",
    "expiresIn": 900
  }
}
```

**Error Cases**

| 상황 | 코드 |
|---|---|
| sessionToken 없음 또는 만료 | 401 |
| OTP 불일치 | 401 |
| OTP 5회 오류 → 세션 무효화 | 429 |

---

### POST `/auth/logout` — 로그아웃

| 항목 | 내용 |
|---|---|
| 권한 | user / admin |
| 설명 | DB의 Refresh Token 무효화 + 쿠키 삭제. Access Token은 클라이언트가 메모리에서 제거 |

**Response `200`**

*Response Header (쿠키 삭제)*
```
Set-Cookie: refresh_token=; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh; Max-Age=0
```

*Response Body*
```json
{
  "success": true,
  "message": "로그아웃 완료"
}
```

---

### POST `/auth/refresh` — Access Token 갱신

| 항목 | 내용 |
|---|---|
| 권한 | public (쿠키 필요) |
| 설명 | HttpOnly 쿠키의 Refresh Token을 서버가 자동 읽어 검증 후 새 Access Token 발급. Request Body 없음 |

**Request Body** — 없음 *(쿠키는 브라우저가 자동 전송)*

**Response `200`**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "tokenType": "Bearer",
    "expiresIn": 900
  }
}
```

**Error Cases**

| 상황 | 코드 |
|---|---|
| 쿠키 없음 또는 만료 | 401 |
| DB에서 무효화된 토큰 | 401 |

---

## 2. 사용자 관리 (Users)

### POST `/users` — 사용자 등록

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 관리자가 학교 담당자 계정 생성 |

**Request Body**
```json
{
  "username": "user01",
  "password": "plainPassword123",
  "school_id": 1
}
```

**Response `201`**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "username": "user01",
    "school_id": 1,
    "role": "user",
    "created_at": "2026-05-11T00:00:00.000Z"
  }
}
```

---

### GET `/users` — 사용자 목록 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 전체 사용자 목록 반환 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| school_id | INT | N | 특정 학교 사용자만 필터 |
| page | INT | N | 페이지 번호 (기본값: 1) |
| limit | INT | N | 페이지당 개수 (기본값: 20) |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "total": 50,
    "page": 1,
    "limit": 20,
    "items": [
      {
        "id": 1,
        "username": "user01",
        "school_id": 1,
        "school_name": "○○초등학교",
        "role": "user",
        "created_at": "2026-05-11T00:00:00.000Z"
      }
    ]
  }
}
```

---

### GET `/users/:id` — 사용자 단건 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin / 본인 |
| 설명 | 특정 사용자 정보 조회 |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "username": "user01",
    "school_id": 1,
    "school_name": "○○초등학교",
    "role": "user",
    "created_at": "2026-05-11T00:00:00.000Z"
  }
}
```

---

### PATCH `/users/:id` — 사용자 정보 수정

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 비밀번호 또는 소속 학교 수정 |

**Request Body** *(수정할 필드만 전송)*
```json
{
  "password": "newPassword456",
  "school_id": 2
}
```

**Response `200`**
```json
{
  "success": true,
  "message": "사용자 정보가 수정되었습니다."
}
```

---

### DELETE `/users/:id` — 사용자 삭제

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 사용자 계정 삭제 |

**Response `200`**
```json
{
  "success": true,
  "message": "사용자가 삭제되었습니다."
}
```

---

## 3. 학교 관리 (Schools)

### POST `/schools` — 학교 등록

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 새 학교 등록 |

**Request Body**
```json
{
  "school_name": "○○초등학교",
  "region": "서울",
  "contact_number": "02-1234-5678"
}
```

**Response `201`**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "school_name": "○○초등학교",
    "region": "서울",
    "contact_number": "02-1234-5678",
    "created_at": "2026-05-11T00:00:00.000Z"
  }
}
```

---

### GET `/schools` — 학교 목록 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 전체 학교 목록 반환 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| region | STRING | N | 지역 필터 |
| page | INT | N | 페이지 번호 |
| limit | INT | N | 페이지당 개수 |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "total": 30,
    "items": [
      {
        "id": 1,
        "school_name": "○○초등학교",
        "region": "서울",
        "contact_number": "02-1234-5678"
      }
    ]
  }
}
```

---

### GET `/schools/:id` — 학교 단건 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin / user (본인 소속 학교) |
| 설명 | 특정 학교 정보 조회 |

---

### PATCH `/schools/:id` — 학교 정보 수정

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 학교 이름·지역·연락처 수정 |

**Request Body** *(수정할 필드만 전송)*
```json
{
  "contact_number": "02-9999-0000"
}
```

---

### DELETE `/schools/:id` — 학교 삭제

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 학교 삭제 (연결된 user, report 확인 필요) |

---

## 4. 보고서 관리 (Reports)

### POST `/reports` — 보고서 등록

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 보고서 메타 정보 등록 (파일은 별도 업로드) |

**Request Body**
```json
{
  "title": "2026년 1분기 환경 검사 보고서",
  "content": "보고서 설명...",
  "school_id": 1,
  "status": "draft"
}
```

**Response `201`**
```json
{
  "success": true,
  "data": {
    "id": 10,
    "title": "2026년 1분기 환경 검사 보고서",
    "school_id": 1,
    "status": "draft",
    "created_at": "2026-05-11T00:00:00.000Z"
  }
}
```

---

### GET `/reports` — 보고서 목록 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin (전체) / user (본인 학교만) |
| 설명 | 보고서 목록. user는 자신의 school_id 기준으로 자동 필터 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| school_id | INT | N | 학교 필터 (admin only) |
| status | STRING | N | 상태 필터 (draft / published / archived) |
| page | INT | N | 페이지 번호 |
| limit | INT | N | 페이지당 개수 |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "total": 5,
    "items": [
      {
        "id": 10,
        "title": "2026년 1분기 환경 검사 보고서",
        "school_name": "○○초등학교",
        "status": "published",
        "file_count": 3,
        "created_at": "2026-05-11T00:00:00.000Z"
      }
    ]
  }
}
```

---

### GET `/reports/:id` — 보고서 단건 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin / user (본인 학교 보고서만) |
| 설명 | 보고서 상세 + 첨부 파일 목록 포함 |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "id": 10,
    "title": "2026년 1분기 환경 검사 보고서",
    "content": "보고서 설명...",
    "school_id": 1,
    "school_name": "○○초등학교",
    "status": "published",
    "created_at": "2026-05-11T00:00:00.000Z",
    "files": [
      {
        "id": 1,
        "file_name": "report_q1.pdf",
        "file_url": "/files/한강초등학교/2026-05-11/a1b2c3d4_report_q1.pdf",
        "file_size": 2048000,
        "file_type": "application/pdf",
        "file_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        "uploaded_at": "2026-05-11T00:00:00.000Z"
      }
    ]
  }
}
```

---

### PATCH `/reports/:id` — 보고서 수정

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 제목·내용·상태 수정 |

**Request Body** *(수정할 필드만 전송)*
```json
{
  "status": "published"
}
```

---

### DELETE `/reports/:id` — 보고서 삭제

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 보고서 및 첨부 파일 전체 삭제 |

---

## 5. 보고서 파일 (Report Files)

### POST `/reports/:reportId/files` — 파일 업로드

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 보고서에 파일 첨부 |
| Content-Type | `multipart/form-data` |

**Request Form Data**

| 키 | 타입 | 필수 | 설명 |
|---|---|---|---|
| file | File | Y | 업로드할 파일 (PDF, 이미지 등) |

**Response `201`**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "report_id": 10,
    "file_name": "report_q1.pdf",
    "file_path": "/var/data/reports/한강초등학교/2026-05-11/a1b2c3d4_report_q1.pdf",
    "file_url": "/files/한강초등학교/2026-05-11/a1b2c3d4_report_q1.pdf",
    "file_size": 2048000,
    "file_type": "application/pdf",
    "file_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "uploaded_at": "2026-05-11T00:00:00.000Z"
  }
}
```

> 저장 경로는 `FILE_STORAGE_ROOT` env 값 기준. 학교명은 URL-safe 처리 후 디렉토리명으로 사용.
> `file_hash`는 서버가 업로드 직후 파일 바이트로 SHA-256을 계산해 DB에 저장. 클라이언트는 이 값으로 다운로드 후 무결성 직접 검증 가능.

---

### GET `/reports/:reportId/files` — 파일 목록 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin / user (본인 학교 보고서만) |
| 설명 | 특정 보고서의 첨부 파일 목록 |

---

### GET `/reports/:reportId/files/:fileId/download` — 파일 다운로드

| 항목 | 내용 |
|---|---|
| 권한 | admin / user (본인 학교 보고서만) |
| 설명 | 파일 스트리밍 다운로드 |

**Response Header**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="report_q1.pdf"
X-File-Hash: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

> 클라이언트는 다운로드 완료 후 수신 파일을 SHA-256으로 해싱하여 `X-File-Hash` 값과 비교, 파일 무결성 검증 가능.

---

### GET `/reports/:reportId/files/:fileId/verify` — 파일 무결성 서버 검증

| 항목 | 내용 |
|---|---|
| 권한 | admin / user (본인 학교 보고서만) |
| 설명 | 서버가 디스크의 실제 파일을 직접 SHA-256으로 재계산하여 DB에 저장된 해시와 비교. 백업 복원 후 또는 디스크 이상 의심 시 사용 |

**Response `200`** *(무결성 정상)*
```json
{
  "success": true,
  "data": {
    "file_id": 1,
    "file_name": "report_q1.pdf",
    "status": "ok",
    "db_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "disk_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "verified_at": "2026-05-12T00:00:00.000Z"
  }
}
```

**Response `409`** *(무결성 불일치 — 파일 손상)*
```json
{
  "success": false,
  "statusCode": 409,
  "data": {
    "file_id": 1,
    "file_name": "report_q1.pdf",
    "status": "corrupted",
    "db_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "disk_hash": "aaaa1234..."
  },
  "message": "파일 해시 불일치: 파일이 손상되었을 수 있습니다."
}
```

---

### DELETE `/reports/:reportId/files/:fileId` — 파일 삭제

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 서버 디스크 파일 및 DB 레코드 함께 삭제 |

---

## 6. 고객 문의 (Contact)

### POST `/contact` — 문의 제출

| 항목 | 내용 |
|---|---|
| 권한 | public |
| 설명 | 랜딩 페이지 문의 폼 제출 |

**Request Body**
```json
{
  "name": "홍길동",
  "email": "hong@example.com",
  "phone": "010-1234-5678",
  "message": "서비스 도입 관련 문의드립니다."
}
```

**Response `201`**
```json
{
  "success": true,
  "message": "문의가 접수되었습니다."
}
```

---

### GET `/contact` — 문의 목록 조회

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 접수된 문의 목록 조회 |

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| is_read | BOOLEAN | N | 읽음 여부 필터 (true / false) |
| page | INT | N | 페이지 번호 |
| limit | INT | N | 페이지당 개수 |

**Response `200`**
```json
{
  "success": true,
  "data": {
    "total": 12,
    "items": [
      {
        "id": 1,
        "name": "홍길동",
        "email": "hong@example.com",
        "phone": "010-1234-5678",
        "message": "서비스 도입 관련 문의드립니다.",
        "is_read": false,
        "created_at": "2026-05-11T00:00:00.000Z"
      }
    ]
  }
}
```

---

### PATCH `/contact/:id/read` — 문의 읽음 처리

| 항목 | 내용 |
|---|---|
| 권한 | admin |
| 설명 | 문의 읽음 상태를 true로 변경 |

**Response `200`**
```json
{
  "success": true,
  "message": "읽음 처리 완료"
}
```

---

## 7. HTTP 상태 코드 정의

| 코드 | 의미 | 사용 케이스 |
|---|---|---|
| 200 | OK | 조회·수정·삭제 성공 |
| 201 | Created | 등록 성공 |
| 400 | Bad Request | 요청 파라미터 오류 |
| 401 | Unauthorized | 인증 토큰 없음 또는 만료 |
| 403 | Forbidden | 권한 없음 |
| 404 | Not Found | 리소스 없음 |
| 409 | Conflict | 중복 데이터 (username 등) |
| 429 | Too Many Requests | 로그인 실패 횟수 초과 (계정 잠금) |
| 500 | Internal Server Error | 서버 오류 |
