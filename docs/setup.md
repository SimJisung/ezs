# EZS 개발 환경 설정 가이드

## 1. 사전 요구사항

### 1.1 필수 설치
```bash
# Node.js 20+ (LTS)
# https://nodejs.org/

# pnpm (패키지 매니저)
npm install -g pnpm

# Wrangler (Cloudflare CLI)
npm install -g wrangler
```

### 1.2 Cloudflare 계정 설정
1. [Cloudflare Dashboard](https://dash.cloudflare.com/) 계정 생성
2. Wrangler 로그인:
   ```bash
   wrangler login
   ```

---

## 2. 프로젝트 초기 설정

### 2.1 저장소 클론 및 의존성 설치
```bash
git clone <repository-url>
cd ezs
pnpm install
```

### 2.2 Cloudflare 리소스 생성

#### D1 데이터베이스
```bash
# 데이터베이스 생성
wrangler d1 create ezs-db

# 출력된 database_id를 apps/api/wrangler.toml에 입력
```

#### R2 버킷
```bash
# 버킷 생성
wrangler r2 bucket create ezs-storage
```

#### KV 네임스페이스
```bash
# 네임스페이스 생성
wrangler kv:namespace create CACHE

# 출력된 id를 apps/api/wrangler.toml에 입력
```

### 2.3 환경 변수 설정

#### apps/api/.dev.vars
```env
# 로컬 개발용 환경 변수
JWT_SECRET=your-dev-secret-key
OPENAI_API_KEY=sk-xxx  # Workers AI 대신 사용 시
```

#### apps/web/.env.local
```env
VITE_API_URL=http://localhost:8787/api/v1
```

---

## 3. 데이터베이스 마이그레이션

### 3.1 마이그레이션 파일 생성 위치
```
apps/api/migrations/
├── 0001_init.sql
└── 0002_add_indexes.sql
```

### 3.2 마이그레이션 실행
```bash
# 로컬 D1에 마이그레이션 적용
cd apps/api
wrangler d1 execute ezs-db --local --file=./migrations/0001_init.sql

# 원격 D1에 마이그레이션 적용 (스테이징/프로덕션)
wrangler d1 execute ezs-db --file=./migrations/0001_init.sql
```

---

## 4. 로컬 개발 실행

### 4.1 전체 개발 서버 시작
```bash
# 루트에서 실행 (Turborepo)
pnpm dev
```

### 4.2 개별 앱 실행
```bash
# API 서버만
cd apps/api
pnpm dev
# → http://localhost:8787

# Web 앱만
cd apps/web
pnpm dev
# → http://localhost:5173
```

### 4.3 로컬 D1/R2/KV 사용
Wrangler는 로컬 개발 시 자동으로 `.wrangler/` 폴더에 로컬 스토리지를 생성합니다.

```bash
# 로컬 데이터 확인
ls apps/api/.wrangler/state/
```

---

## 5. 테스트

### 5.1 테스트 실행
```bash
# 전체 테스트
pnpm test

# 특정 패키지 테스트
pnpm --filter api test
pnpm --filter web test
```

### 5.2 E2E 테스트 (Playwright)
```bash
cd apps/web
pnpm test:e2e
```

---

## 6. 코드 품질

### 6.1 린트
```bash
pnpm lint
```

### 6.2 타입 체크
```bash
pnpm typecheck
```

### 6.3 포맷팅
```bash
pnpm format
```

---

## 7. 빌드 및 배포

### 7.1 로컬 빌드
```bash
pnpm build
```

### 7.2 수동 배포

#### API (Workers)
```bash
cd apps/api

# 스테이징
wrangler deploy --env staging

# 프로덕션
wrangler deploy --env production
```

#### Web (Pages)
```bash
cd apps/web
pnpm build

# Pages 배포
wrangler pages deploy dist --project-name=ezs-web
```

### 7.3 자동 배포 (CI/CD)
`main` 또는 `staging` 브랜치에 푸시 시 GitHub Actions가 자동 배포합니다.

---

## 8. 유용한 명령어

### Wrangler
```bash
# Workers 로그 확인
wrangler tail

# D1 데이터 조회
wrangler d1 execute ezs-db --local --command="SELECT * FROM notes LIMIT 10"

# KV 데이터 조회
wrangler kv:key list --binding=CACHE

# R2 파일 목록
wrangler r2 object list ezs-storage
```

### Turborepo
```bash
# 특정 패키지만 빌드
pnpm --filter api build

# 의존성 그래프 시각화
pnpm turbo run build --graph
```

---

## 9. 트러블슈팅

### Workers AI 로컬 테스트
Workers AI는 로컬에서 실행되지 않습니다. 대안:
1. `wrangler dev --remote` 사용 (원격 Workers 환경)
2. 로컬에서는 OpenAI API로 폴백

### D1 스키마 변경
D1은 일부 ALTER TABLE 명령을 지원하지 않습니다.
- 새 테이블 생성 → 데이터 복사 → 기존 테이블 삭제 → 이름 변경

### CORS 에러
`apps/api/src/middleware/cors.ts`에서 허용 도메인 확인:
```typescript
const allowedOrigins = [
  'http://localhost:5173',
  'https://ezs.pages.dev',
];
```

---

## 10. 프로젝트 구조 요약

```
ezs/
├── apps/
│   ├── web/          # React Frontend
│   └── api/          # Cloudflare Workers Backend
├── packages/
│   └── shared/       # 공유 타입/유틸리티
├── docs/             # 문서
├── turbo.json        # Turborepo 설정
├── pnpm-workspace.yaml
└── package.json
```
