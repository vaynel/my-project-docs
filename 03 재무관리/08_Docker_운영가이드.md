---
tags: [docker, operations, elasticsearch, logstash, myfin-v2]
created: 2026-06-02
updated: 2026-06-02
related:
  - "[[01_기획서]]"
  - "[[07_개발진행현황]]"
  - "[[12_동기화_전략_설계]]"
---

# MyFin 2.0 Docker 운영 가이드

> **8GB VM 주의:** Elasticsearch 같은 무거운 컴포넌트까지 한 번에 올리면 **OOM → SSH 끊김**이 발생할 수 있습니다.
>
> 따라서 **현재 설정은 Docker가 DB(PostgreSQL) + Redis만** 담당하고, backend/frontend은 **로컬에서 실행**합니다.

---

## 1. 사전 준비

```bash
cd /root/myfin_2.0
cp .env.example .env
# .env 에 POSTGRES_PASSWORD, REDIS_PASSWORD, JWT 키, MASTER_ENCRYPTION_KEY 입력

# (선택) TLS 인증서
make certs
```

키 생성:

```bash
pip install cryptography
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
openssl rand -hex 32   # JWT_SECRET_KEY, JWT_REFRESH_SECRET_KEY
```

---

## 2. 안전한 기동 순서 (권장)

### Step 1 — 인프라만 (~1.2GB)

```bash
make infra
# 또는: docker compose up -d postgres redis
```

확인:

```bash
docker compose ps
free -h
pg_isready -h 127.0.0.1 -p 5433 -U myfin_user
```

### Step 2 — FastAPI 추가 (~+512MB)
backend/frontend은 로컬에서 실행합니다.

```bash
# 전체를 한 번에 띄우기(권장)
./start.sh

# 또는 수동 실행(필요 시)
cd backend && pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

cd ../frontend && npm install && npm run dev
```

> 참고: Elasticsearch + Logstash 추가 기동은 아래 §7 참조.

---
## 3. 포트 매핑

| 서비스 | 호스트 | 비고 |
| --- | --- | --- |
| PostgreSQL | 127.0.0.1:5434 | finflow 5432 충돌 회피 |
| Redis | 127.0.0.1:6380 | finflow 6379 충돌 회피 |
| Elasticsearch | 127.0.0.1:9200 | 내부 전용 (외부 노출 금지) |
| Logstash HTTP | 127.0.0.1:5044 | Celery Worker → Logstash 적재 |
| Kibana | 127.0.0.1:5601 | SSH 터널 전용 (프로덕션 외부 노출 금지) |

---

## 4. 중지 · 정리

```bash
make down

# backend/frontend까지 같이 내리기
./stop.sh
```

볼륨까지 삭제 (DB 초기화):

```bash
docker compose down -v   # ⚠ 데이터 삭제됨
```

---

## 6. OOM / SSH 끊김 발생 시
1. VM 콘솔 또는 재접속 후 `docker ps` 확인
2. `make down` 또는 `./stop.sh`로 정리
3. `make infra` 만으로 재시작
4. (선택) `infra/sysctl/99-myfin.conf` 적용 검토: `sudo cp infra/sysctl/99-myfin.conf /etc/sysctl.d/ && sudo sysctl -p /etc/sysctl.d/99-myfin.conf`

---

## 5. .env Docker vs 로컬 개발

| 변수 | Docker Compose 내부 | 호스트 uvicorn |
| --- | --- | --- |
| POSTGRES_HOST | `postgres` | `127.0.0.1` |
| POSTGRES_PORT | `5432` | `5433` |
| REDIS_HOST | `redis` | `127.0.0.1` |
| REDIS_PORT | `6379` | `6380` |

현재는 Docker가 db/redis만 기동하므로, backend(local) 실행 시에는 `.env`에서 `POSTGRES_HOST=127.0.0.1`, `REDIS_HOST=127.0.0.1`을 사용하면 됩니다.

---

## 7. Elasticsearch + Logstash 추가 기동

> **메모리 영향:** ES 2,500MB + Logstash 640MB = 추가 ~3.1GB 사용
> 기존 서비스(~2.5GB) + 신규 = 총 **~5.6GB**. OS 예약 1GB 포함 시 한도 내 안전.
> 반드시 `free -h` 확인 후 기동.

### 7.1 `docker-compose.yml` 추가 서비스 정의

아래 내용을 `docker-compose.yml`의 `services:` 블록에 추가:

```yaml
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4
    container_name: myfin_elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xmx2g -Xms1g -XX:+UseG1GC
      - xpack.security.enabled=false          # 내부 전용, TLS 불필요
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data:/usr/share/elasticsearch/data
      - ./infra/elasticsearch/init:/usr/share/elasticsearch/init:ro
    ports:
      - "127.0.0.1:9200:9200"
    networks:
      - myfin_net
    deploy:
      resources:
        limits:
          memory: 2700M
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9200/_cluster/health | grep -qE '\"status\":\"(green|yellow)\"'"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.4
    container_name: myfin_logstash
    restart: unless-stopped
    environment:
      - LS_JAVA_OPTS=-Xmx512m -Xms256m
    volumes:
      - ./infra/logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./infra/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "127.0.0.1:5044:5044"
    networks:
      - myfin_net
    depends_on:
      elasticsearch:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 640M
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9600/ | grep -q '\"status\":\"green\"'"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 45s

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.4
    container_name: myfin_kibana
    restart: unless-stopped
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "127.0.0.1:5601:5601"       # SSH 터널로만 접근: ssh -L 5601:localhost:5601 user@server
    networks:
      - myfin_net
    depends_on:
      elasticsearch:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 768M
```

`volumes:` 블록에도 추가:
```yaml
  es_data:
```

### 7.2 필수 디렉터리 및 파일 생성

```bash
mkdir -p infra/elasticsearch/init
mkdir -p infra/logstash/pipeline
mkdir -p infra/logstash/config
```

**`infra/logstash/config/logstash.yml`:**
```yaml
http.host: "0.0.0.0"
log.level: warn
pipeline.workers: 1
pipeline.batch.size: 125
```

**`infra/logstash/pipeline/myfin.conf`:**
→ 파이프라인 전체 명세는 [[12_동기화_전략_설계]] §6.2 참조

### 7.3 ES 초기화 (최초 1회)

```bash
# ES가 healthy 상태가 된 후 실행
docker compose up -d elasticsearch
docker compose ps   # elasticsearch: healthy 확인

# ILM 정책 등록
curl -X PUT "localhost:9200/_ilm/policy/myfin-24m-policy" \
  -H 'Content-Type: application/json' \
  -d @infra/elasticsearch/init/ilm_24m.json

curl -X PUT "localhost:9200/_ilm/policy/myfin-6m-policy" \
  -H 'Content-Type: application/json' \
  -d @infra/elasticsearch/init/ilm_6m.json

# 인덱스 템플릿 등록
curl -X PUT "localhost:9200/_index_template/myfin-trade-logs-template" \
  -H 'Content-Type: application/json' \
  -d @infra/elasticsearch/init/template_trade_logs.json

curl -X PUT "localhost:9200/_index_template/myfin-sync-logs-template" \
  -H 'Content-Type: application/json' \
  -d @infra/elasticsearch/init/template_sync_logs.json
```

### 7.4 Logstash 포함 기동 순서

```bash
# 순서 중요: postgres/redis → elasticsearch (healthy) → logstash → 앱
docker compose up -d postgres redis
docker compose up -d elasticsearch   # ~60초 대기 필요
docker compose up -d logstash kibana

# 상태 확인
curl localhost:9200/_cluster/health
curl localhost:9600/          # Logstash status
```

### 7.5 메모리 부족 시 대응

```bash
free -h   # available < 1GB 이면 기동 중단

# Logstash 메모리 줄이기 (최소 구성)
# docker-compose.yml에서 LS_JAVA_OPTS=-Xmx256m -Xms128m 변경

# Kibana 비활성화 (모니터링 불필요 시)
# docker compose --profile no-kibana up -d
```
