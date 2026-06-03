---
tags: [security, owasp, myfin-v2]
created: 2026-06-02
version: "2.0"
status: approved
related:
  - "[[01_기획서]]"
  - "[[03_ERD]]"
  - "[[04_API명세서]]"
---

# MyFin V2.0 OWASP 보안 설계서

> **기준:** OWASP Top 10 (2021) + 금융 트레이딩 시스템 특화 보안 요구사항
> **적용 레이어:** 코드 · 인프라 · 운영 전 레이어 동시 적용

---

## 1. 보안 위협 총괄 매핑표

| OWASP ID | 위협명 | 위험도 | 적용 레이어 | 상태 |
| --- | --- | --- | --- | --- |
| A01:2021 | Broken Access Control | 🔴 치명 | FastAPI DI, DB | 설계 반영 |
| A02:2021 | Cryptographic Failures | 🔴 치명 | CryptoService, DB | 설계 반영 |
| A03:2021 | Injection | 🔴 치명 | Pydantic, SQLAlchemy ORM | 설계 반영 |
| A04:2021 | Insecure Design | 🟠 높음 | 아키텍처 전반 | 설계 반영 |
| A05:2021 | Security Misconfiguration | 🟠 높음 | Docker, Nginx, 환경변수 | 설계 반영 |
| A06:2021 | Vulnerable and Outdated Components | 🟡 중간 | CI/CD 파이프라인 | 설계 반영 |
| A07:2021 | Identification and Authentication Failures | 🔴 치명 | JWT, 세션 관리 | 설계 반영 |
| A08:2021 | Software and Data Integrity Failures | 🟡 중간 | Docker 이미지, 의존성 | 설계 반영 |
| A09:2021 | Security Logging and Monitoring Failures | 🟠 높음 | Logstash, Elasticsearch | 설계 반영 |
| A10:2021 | Server-Side Request Forgery (SSRF) | 🟠 높음 | 증권사 API 클라이언트 | 설계 반영 |
| 추가 | CSRF | 🔴 치명 | 쿠키 정책, SameSite | **신규 추가** |
| 추가 | Brute Force | 🟠 높음 | Nginx Rate Limiting | **신규 추가** |
| 추가 | Security Headers | 🟡 중간 | Nginx | **신규 추가** |

---

## 2. A01: Broken Access Control — 접근 제어 오류

### 2.1 위협 시나리오

사용자 A가 B의 `account_id=2`를 URL에 직접 입력하여 B의 API Key 정보, 주문 이력에 접근.

### 2.2 방어 설계

**FastAPI 소유권 검증 의존성 주입 패턴:**

```python
# 모든 계좌·룰·보고서 접근 라우터에 아래 의존성 필수 적용
async def verify_account_owner(
    account_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
) -> UserAccount:
    account = await db.get(UserAccount, account_id)
    if not account or account.deleted_at is not None:
        raise HTTPException(status_code=404, detail="계좌를 찾을 수 없습니다.")
    if account.user_id != current_user.id:
        # 404로 응답하여 자원 존재 여부도 노출하지 않음
        raise HTTPException(status_code=404, detail="계좌를 찾을 수 없습니다.")
    return account
```

> [!important] 403 대신 404 반환 이유
> 403(Forbidden)은 자원이 존재하지만 권한이 없음을 암시한다. 타 사용자 자원 존재 여부 자체를 노출하지 않기 위해 404를 반환한다.

**적용 범위:**

| 자원 | 검증 항목 |
| --- | --- |
| `user_accounts` | `user_id = JWT user_id` |
| `trading_rules` | `account.user_id = JWT user_id` (2단계 체인 검증) |
| `orders` | `account.user_id = JWT user_id` |
| `watchlists` | `user_id = JWT user_id` |
| `ai_reports` | `user_id = JWT user_id` |

---

## 3. A02: Cryptographic Failures — 암호화 오류

### 3.1 위협 시나리오

- DB 덤프 유출 시 API Key 평문 노출
- Master Key가 Git 레포지토리에 커밋됨

### 3.2 방어 설계

**암호화 구현 사양:**

| 항목 | 적용 기술 | 세부 설정 |
| --- | --- | --- |
| API Key 암호화 | `cryptography.fernet` (Fernet) | AES-128-CBC + HMAC-SHA256. 업계 표준 AES-256 동급 보안 강도 |
| 비밀번호 해싱 | `argon2-cffi` (Argon2id) | `memory_cost=65536`, `time_cost=3`, `parallelism=4` |
| JWT 서명 | HMAC-SHA256 (HS256) | Secret Key 256비트(32바이트) 이상 |
| Refresh Token 저장 | SHA-256 해시만 DB 저장 | 원문은 절대 저장하지 않음 |
| 전송 암호화 | TLS 1.3 (Nginx) | TLS 1.0/1.1 비활성화 |

**Master Key 관리 원칙:**

```bash
# 절대 금지: 코드에 하드코딩
MASTER_ENCRYPTION_KEY = "my-secret-key"  # ❌

# 절대 금지: Git 커밋
echo "MASTER_ENCRYPTION_KEY=..." >> .env && git add .env  # ❌

# 올바른 방법: OS 환경변수 직접 주입
export MASTER_ENCRYPTION_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
# Docker Compose: env_file 또는 secrets 섹션 사용
```

**`.gitignore` 필수 항목:**
```
.env
.env.*
*.pem
*.key
secrets/
```

---

## 4. A03: Injection — 인젝션

### 4.1 위협 시나리오

- SQL Injection: `ticker = "'; DROP TABLE orders; --"`
- NoSQL Injection: ES DSL 조작

### 4.2 방어 설계

**SQL Injection 방어:**

```python
# ❌ 금지: Raw SQL with f-string
query = f"SELECT * FROM orders WHERE ticker = '{ticker}'"

# ✅ 올바른 방법: SQLAlchemy ORM (파라미터 바인딩 자동 적용)
result = await db.execute(
    select(Order).where(Order.ticker == ticker)
)

# ✅ Raw SQL 불가피할 경우: named parameters 사용
result = await db.execute(
    text("SELECT * FROM orders WHERE ticker = :ticker"),
    {"ticker": ticker}
)
```

**Pydantic 입력 유효성 검증:**

```python
from pydantic import BaseModel, validator, Field
import re

class RuleCreateRequest(BaseModel):
    ticker: str = Field(..., min_length=1, max_length=30, pattern=r'^[A-Z0-9\-\/]+$')
    short_term: int = Field(..., ge=1, le=200)
    long_term: int = Field(..., ge=1, le=500)
    value: float = Field(..., gt=0, le=100)

    @validator('long_term')
    def long_must_be_greater_than_short(cls, v, values):
        if 'short_term' in values and v <= values['short_term']:
            raise ValueError('long_term은 short_term보다 커야 합니다.')
        return v
```

**ES Injection 방어:**
- ES 쿼리에 사용자 입력값을 DSL 구조에 직접 삽입하지 않음
- 입력값은 반드시 `term`, `match` 쿼리의 `value` 필드로만 전달
- `query_string` 타입 쿼리 사용 금지 (와일드카드·연산자 삽입 가능)

---

## 5. A05: Security Misconfiguration — 보안 구성 오류

### 5.1 Nginx 보안 헤더 설정

```nginx
# /etc/nginx/conf.d/security-headers.conf
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' wss://api.myfin.local" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

# 서버 버전 정보 노출 차단
server_tokens off;
```

### 5.2 Docker 네트워크 격리

```yaml
# docker-compose.yml 네트워크 설계
networks:
  frontend_net:   # nginx ↔ fastapi
  backend_net:    # fastapi ↔ postgres, redis, celery
    internal: true  # 외부 인터넷 접근 완전 차단
  analytics_net:  # logstash ↔ elasticsearch, kibana
    internal: true

services:
  postgres:
    networks: [backend_net]
    ports: []  # 외부 포트 노출 없음
  redis:
    networks: [backend_net]
    ports: []
  elasticsearch:
    networks: [analytics_net]
    ports: []  # localhost 터널 접근만 허용
  kibana:
    networks: [analytics_net]
    ports:
      - "127.0.0.1:5601:5601"  # 로컬호스트만 바인딩
```

> [!warning] 절대 외부 개방 금지
> myfin 2.0의 PostgreSQL(호스트 **5433**), Redis(호스트 **6380**), Elasticsearch(9200/9300), Kibana(5601) 포트는 어떤 경우에도 `0.0.0.0`에 바인딩하지 않는다. 운영 진단이 필요할 경우 SSH 포트 포워딩만 사용한다. (`ssh -L 5601:localhost:5601 user@myfin-vm`)

### 5.3 FastAPI CORS 설정

```python
from fastapi.middleware.cors import CORSMiddleware
import os

app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("ALLOWED_ORIGINS", "").split(","),  # 하드코딩 금지
    allow_credentials=True,   # 쿠키 전송 허용 (Refresh Token)
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    expose_headers=["X-Request-ID"],
)
```

> [!warning] `allow_origins=["*"]` 절대 금지
> Wildcard Origin과 `allow_credentials=True` 동시 사용은 CORS 스펙에서 오류. 반드시 명시적 도메인을 ENV로 주입한다.

---

## 6. A07: Identification and Authentication Failures — 인증 오류

### 6.1 JWT 보안 설계

**토큰 보안 정책:**

| 항목 | 설정 | 이유 |
| --- | --- | --- |
| Access Token 유효기간 | 60분 | 탈취 시 피해 최소화 |
| Refresh Token 유효기간 | 30일 | 사용성 균형 |
| Access Token 저장 위치 | 메모리(Vuex/Pinia) | localStorage는 XSS 탈취 가능 |
| Refresh Token 저장 위치 | HttpOnly + Secure + SameSite=Strict 쿠키 | JS 접근 차단 |
| 알고리즘 | HS256 | 단일 서버 환경. RS256은 키 관리 오버헤드 과다 |

**Refresh Token Rotation 로직:**

```python
async def refresh_access_token(refresh_token: str, db: AsyncSession):
    token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
    session = await db.scalar(
        select(UserSession).where(
            UserSession.refresh_token_hash == token_hash,
            UserSession.is_revoked == False,
            UserSession.expires_at > datetime.utcnow()
        )
    )
    if not session:
        # 무효화된 토큰 재사용 의심 → 전체 세션 무효화
        await db.execute(
            update(UserSession)
            .where(UserSession.user_id == session.user_id)
            .values(is_revoked=True)
        )
        raise HTTPException(status_code=401, detail="ERR_AUTH_003")

    # 기존 세션 무효화
    session.is_revoked = True

    # 새 세션 생성
    new_token = secrets.token_urlsafe(32)
    new_session = UserSession(
        user_id=session.user_id,
        refresh_token_hash=hashlib.sha256(new_token.encode()).hexdigest(),
        expires_at=datetime.utcnow() + timedelta(days=30)
    )
    db.add(new_session)
    await db.commit()

    return new_token, create_access_token(session.user_id)
```

### 6.2 CSRF 방어

**설계 원칙:** `SameSite=Strict` 쿠키 정책이 주요 CSRF 방어 수단. 추가적으로 `Origin` 헤더 검증.

```python
# FastAPI CSRF 미들웨어
@app.middleware("http")
async def csrf_protection(request: Request, call_next):
    # 상태 변경 요청 (POST, PUT, PATCH, DELETE)에 대해 Origin 검증
    if request.method in ("POST", "PUT", "PATCH", "DELETE"):
        origin = request.headers.get("origin")
        allowed_origins = os.getenv("ALLOWED_ORIGINS", "").split(",")
        if origin and origin not in allowed_origins:
            return JSONResponse(
                status_code=403,
                content={"success": False, "error_code": "ERR_CSRF", "message": "CSRF 검증 실패"}
            )
    return await call_next(request)
```

**쿠키 속성 완전 명세:**

| 속성 | 값 | 이유 |
| --- | --- | --- |
| `HttpOnly` | True | JavaScript `document.cookie` 접근 차단 |
| `Secure` | True | HTTPS 전송만 허용 |
| `SameSite` | `Strict` | 크로스 사이트 요청 시 쿠키 전송 차단 |
| `Path` | `/v1/auth/refresh` | Refresh Token을 refresh 엔드포인트에서만 사용 |
| `Max-Age` | 2592000 (30일) | Refresh Token 유효기간과 동일 |

---

## 7. A09: Security Logging and Monitoring Failures — 로깅 오류

### 7.1 민감정보 마스킹

**Logstash 파이프라인 마스킹 필터:**

```
# logstash/pipeline/myfin.conf
filter {
  # API Key / Secret Key 패턴 마스킹
  mutate {
    gsub => [
      "message", "(?i)(api_key|secret_key|access_token|password)[\"': ]+[^\s\"',}]+", '\1: ****',
      "raw_response", "(?i)(api_key|secret_key|token)[\"': ]+[^\s\"',}]+", '\1: ****'
    ]
  }
}
```

**Python 로거 민감정보 필터:**

```python
import logging
import re

class SensitiveDataFilter(logging.Filter):
    PATTERNS = [
        (re.compile(r'(?i)(api_key|secret_key|password)[=: ]+\S+'), r'\1: ****'),
        (re.compile(r'(?i)Bearer\s+\S+'), 'Bearer ****'),
    ]

    def filter(self, record):
        record.msg = self._mask(str(record.msg))
        return True

    def _mask(self, text: str) -> str:
        for pattern, replacement in self.PATTERNS:
            text = pattern.sub(replacement, text)
        return text

# 모든 로거에 필터 등록
logging.getLogger().addFilter(SensitiveDataFilter())
```

### 7.2 보안 감사 로그 (Security Audit Log)

아래 이벤트는 반드시 구조화 로그로 기록하고 ES에 적재한다.

| 이벤트 | 기록 항목 |
| --- | --- |
| 로그인 성공 | `user_id`, `ip_address`, `user_agent`, `timestamp` |
| 로그인 실패 | `email(마스킹)`, `ip_address`, `fail_reason`, `timestamp` |
| Kill-Switch 실행 | `user_id`, `scope` (global/account), `account_ids` |
| API Key 등록·삭제 | `user_id`, `account_id`, `broker_name`, `action` |
| Refresh Token Rotation 탈취 감지 | `user_id`, `ip_address`, `all_sessions_revoked: true` |

---

## 8. A10: SSRF — 서버 측 요청 위조

### 8.1 위협 시나리오

증권사 API URL을 사용자가 입력하도록 설계하면, 공격자가 `http://169.254.169.254/metadata`(클라우드 메타데이터) 또는 내부망 Redis/PostgreSQL 주소를 입력하여 내부 자원에 접근.

### 8.2 방어 설계

```python
# 증권사별 Base URL은 코드·ENV에 고정. 사용자 입력 URL 사용 금지.
BROKER_BASE_URLS = {
    "korea_investment": "https://openapi.koreainvestment.com:9443",
    "upbit": "https://api.upbit.com",
    "binance": "https://api.binance.com",
    "binance_futures": "https://fapi.binance.com",
}

async def call_broker_api(broker_name: str, endpoint: str, **kwargs):
    base_url = BROKER_BASE_URLS.get(broker_name)
    if not base_url:
        raise ValueError(f"지원하지 않는 증권사: {broker_name}")
    # 사용자 입력값은 endpoint에 절대 포함되지 않음
    url = f"{base_url}{endpoint}"
    async with httpx.AsyncClient(timeout=30.0) as client:
        return await client.request(**kwargs, url=url)
```

---

## 9. Rate Limiting — Brute Force 방어

### 9.1 Nginx Rate Limiting 설정

```nginx
# /etc/nginx/nginx.conf
http {
    # 로그인 전용 Zone (IP 기준, 엄격)
    limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;

    # 회원가입 Zone
    limit_req_zone $binary_remote_addr zone=register:10m rate=5r/h;

    # 일반 API Zone
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;

    # AI 보고서 생성 (사용자 기준)
    limit_req_zone $http_authorization zone=reports:10m rate=1r/h;
}

server {
    location = /v1/auth/login {
        limit_req zone=login burst=5 nodelay;
        limit_req_status 429;
        proxy_pass http://fastapi:8000;
    }

    location = /v1/auth/register {
        limit_req zone=register burst=2 nodelay;
        limit_req_status 429;
        proxy_pass http://fastapi:8000;
    }

    location /v1/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://fastapi:8000;
    }
}
```

---

## 10. HTTPS / TLS 설정 (내부망 자체 서명 인증서)

> [!note] Let's Encrypt 미사용 이유
> 내부망 전용 시스템으로 공인 CA의 도메인 소유권 검증(ACME challenge)이 불가능하다. OpenSSL로 자체 CA를 구성하고 내부 기기에 CA 인증서를 신뢰 등록한다. 상세 발급 절차는 [[01_기획서]] §6.4를 참조.

### 10.1 인증서 갱신 관리

자체 서명 인증서는 825일 유효기간으로 발급한다 (Apple/Chrome 정책 준수). 갱신 알림을 수동으로 관리한다.

```bash
# 인증서 만료일 확인 (주기적으로 실행)
openssl x509 -in /etc/nginx/certs/myfin.crt -noout -dates

# 만료 30일 전 재발급 (발급 스크립트 재실행)
# 재발급 후 nginx 재시작
docker compose restart nginx
```

**만료 사전 알림 (cron):**
```bash
# /etc/cron.d/myfin-cert-check
0 9 * * 1 root openssl x509 -in /etc/nginx/certs/myfin.crt -noout -checkend 2592000 \
  || curl -s -X POST "$DISCORD_WEBHOOK_URL" \
     -H "Content-Type: application/json" \
     -d '{"content":"[MyFin] TLS 인증서 만료 30일 이내. 갱신 필요."}'
```

### 10.2 Nginx TLS 강화 설정

```nginx
ssl_protocols       TLSv1.2 TLSv1.3;
ssl_ciphers         ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;
ssl_session_timeout 1d;
ssl_session_cache   shared:SSL:10m;

# 자체 서명 인증서는 OCSP stapling 불가 (CA가 공인 OCSP 서버 없음)
# ssl_stapling on;  ← 사용 금지
```

> [!important] `ssl_stapling` 비활성화 필수
> 자체 서명 인증서는 공인 OCSP 응답 서버가 없다. `ssl_stapling on` 설정 시 Nginx가 OCSP 서버에 접근을 시도하며 타임아웃이 발생해 TLS 핸드셰이크가 지연된다.

---

## 11. 의존성 취약점 관리 (A06)

### 11.1 자동화 스캔 설정

```bash
# pip-audit: Python 패키지 CVE 스캔
pip install pip-audit
pip-audit --requirement requirements.txt --output json > audit-report.json

# bandit: Python 소스코드 정적 분석
pip install bandit
bandit -r app/ -f json -o bandit-report.json

# 실행 방법 (로컬 또는 CI)
make security-check
```

**Makefile 예시:**
```makefile
security-check:
	pip-audit --requirement requirements.txt
	bandit -r app/ -ll  # 중간 이상 심각도만 리포트
```

### 11.2 의존성 업데이트 정책

| 유형 | 정책 |
| --- | --- |
| 보안 패치 (패치 버전) | 발견 즉시 업데이트 |
| 기능 업데이트 (마이너 버전) | 월 1회 검토 |
| 주요 버전 업그레이드 | 분기별 검토 + 통합 테스트 후 반영 |

---

## 12. 보안 체크리스트 (배포 전 필수 확인)

| 항목 | 확인 방법 | 담당 |
| --- | --- | --- |
| `.env` 파일 Git 미포함 | `git ls-files .env` 결과 없음 확인 | 개발자 |
| Master Key 32바이트 이상 | `len(MASTER_ENCRYPTION_KEY) >= 32` | 코드 리뷰 |
| PostgreSQL 외부 포트 미노출 | `ss -tlnp \| grep 5433` → `127.0.0.1` 바인딩 확인 | 인프라 |
| Redis AUTH 설정 | `redis-cli -p 6380 AUTH 패스워드 PING` | 인프라 |
| Elasticsearch 외부 포트 미노출 | `curl http://서버IP:9200` 타임아웃 확인 | 인프라 |
| HTTPS 인증서 유효 | `openssl x509 -in /etc/nginx/certs/myfin.crt -noout -dates` | 인프라 |
| Nginx Rate Limit 동작 | `ab -n 20 -c 5 .../auth/login` 후 429 확인 | QA |
| HttpOnly 쿠키 확인 | 브라우저 DevTools → Cookies | QA |
| SameSite=Strict 쿠키 확인 | 브라우저 DevTools → Cookies | QA |
| 보안 헤더 확인 | `curl -I https://myfin.local` | QA |
| CORS Wildcard 미사용 | FastAPI 설정 코드 리뷰 | 개발자 |
| 로그 내 API Key 마스킹 | ES 인덱스 샘플 쿼리 | QA |
| pip-audit 통과 | CI 파이프라인 그린 | CI |
| bandit 중간 이상 0건 | CI 파이프라인 그린 | CI |
