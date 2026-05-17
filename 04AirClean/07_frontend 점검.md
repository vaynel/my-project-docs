# Frontend 점검

> 점검일: 2026-05-12
> 점검자: 개발팀
> 점검 기준: `logs/frontend.log`
> 상세 패치 현황: [[99_frontend_patch_xlsx]]

---

## 목차

- [[#서버 기동 상태]]
- [[#발견된 오류 및 패치]]
- [[#패치 결과 요약]]

---

## 서버 기동 상태

| 항목 | 상태 | 비고 |
|------|------|------|
| Next.js 개발 서버 기동 | ✅ 정상 | Next.js 16.2.6 (Turbopack), 포트 4001 |
| 빌드 준비 시간 | ✅ 정상 | Ready in 241ms |
| 페이지 렌더링 (패치 전) | ❌ 전체 500 | CSS @import 순서 오류 |
| 페이지 렌더링 (패치 후) | ✅ 정상 | 수정 완료 |

---

## 발견된 오류 및 패치

---

### FE-01. CSS `@import` 순서 오류 — 전체 페이지 500

**심각도:** 🔴 Critical (서비스 전체 불가)
**상태:** ✅ 패치 완료

#### 원인 분석

`src/app/globals.css`에서 `@tailwind base/components/utilities` 디렉티브 이후에 `@import` 구문이 위치함.

PostCSS가 tailwind 디렉티브를 실제 CSS 규칙으로 변환하면, 그 이후에 오는 `@import`는 CSS 명세 위반(`@import rules must precede all rules`)이 되어 파싱 자체가 실패함.

**오류 로그:**
```
⨯ ./src/app/globals.css:1906:8
Parsing CSS source code failed
@import rules must precede all rules aside from @charset and @layer statements
→ GET / 500 in 2.3s
```

#### 변경 내용

**파일:** `frontend/src/app/globals.css`

```diff
- @tailwind base;
- @tailwind components;
- @tailwind utilities;
-
- @import url('https://fonts.googleapis.com/css?family=Roboto:400,300,700&subset=latin,latin-ext');
+ @import url('https://fonts.googleapis.com/css?family=Roboto:400,300,700&subset=latin,latin-ext');
+
+ @tailwind base;
+ @tailwind components;
+ @tailwind utilities;
```

**근거:** CSS 명세 및 PostCSS 처리 순서 — `@import`는 `@charset`, `@layer` 외 모든 규칙보다 앞에 위치해야 함.

---

### FE-02. `next.config.js` 제거된 옵션 경고

**심각도:** 🟡 Medium (서비스 영향 없음, 경고 누적)
**상태:** ✅ 패치 완료

#### 원인 분석

Next.js 15+ 에서 제거된 4개 옵션이 `next.config.js`에 잔존함.

| 옵션 | 제거 버전 | 대체 방안 |
|------|----------|-----------|
| `swcMinify` | Next.js 15 | 항상 활성화 (설정 불필요) |
| `experimental.outputFileTracingRoot` | Next.js 15 | 최상위 `outputFileTracingRoot`로 이동 또는 제거 |
| `serverRuntimeConfig` | Next.js 15 | 환경변수 직접 사용 |
| `publicRuntimeConfig` | Next.js 15 | `NEXT_PUBLIC_*` 환경변수 직접 사용 |

**오류 로그:**
```
⚠ Invalid next.config.js options detected:
⚠   Unrecognized key(s): 'outputFileTracingRoot' at "experimental"
⚠   Unrecognized key(s): 'swcMinify', 'serverRuntimeConfig', 'publicRuntimeConfig'
```

#### 변경 내용

**파일:** `frontend/next.config.js`

제거된 항목:
- `swcMinify: true`
- `experimental: { outputFileTracingRoot: undefined }`
- `serverRuntimeConfig: { maxFileSize: ... }`
- `publicRuntimeConfig: { maxFileSize: ... }`
- `images.domains: []` (deprecated, `remotePatterns`으로 대체)

> `maxFileSize` 설정은 서버 런타임 설정이 아닌 multer/업로드 미들웨어에서 직접 관리하므로 next.config.js에서 제거해도 기능 영향 없음.

---

### FE-03. Cross-Origin 개발 리소스 차단 경고

**심각도:** 🟡 Medium (개발 환경 외부 접근 불가)
**상태:** ✅ 패치 완료

#### 원인 분석

Next.js 15.2부터 개발 서버의 HMR 및 스택 프레임 리소스에 대해 동일 출처 외 접근을 기본 차단함. `192.168.35.252`에서 접근 시 차단.

**오류 로그:**
```
⚠ Blocked cross-origin request to Next.js dev resource /_next/webpack-hmr from "192.168.35.252".
⚠ Blocked cross-origin request to Next.js dev resource /__nextjs_original-stack-frames from "192.168.35.252".
```

#### 변경 내용

**파일:** `frontend/next.config.js`

```diff
+ allowedDevOrigins: ['192.168.35.252'],
```

> 운영 환경(`NODE_ENV=production`)에서는 해당 설정이 무시됨. 개발 환경 전용.

---

## 패치 결과 요약

| ID | 파일 | 변경 내용 | 결과 |
|----|------|-----------|------|
| FE-01 | `src/app/globals.css` | `@import` 구문을 `@tailwind` 디렉티브 위로 이동 | ✅ 500 오류 해소 |
| FE-02 | `next.config.js` | 제거된 옵션 4종 삭제 | ✅ 경고 제거 |
| FE-03 | `next.config.js` | `allowedDevOrigins` 추가 | ✅ 경고 제거 |

### 잔여 점검 항목

> 기능별 상세 점검은 서버 기동 안정화 후 별도 진행

| 항목 | 상태 |
|------|------|
| 로그인 / 인증 흐름 | ⏸ 미점검 |
| API 연동 (backend ↔ frontend) | ⏸ 미점검 |
| 파일 업로드 UI | ⏸ 미점검 |
| 페이지별 렌더링 | ⏸ 미점검 |
