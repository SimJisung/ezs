# EZS 개발 환경 설정 가이드

## 1. 사전 요구사항

### 1.1 필수 설치
```bash
# Node.js 20+ (LTS)
# https://nodejs.org/

# pnpm (패키지 매니저)
npm install -g pnpm

# Supabase CLI
npm install -g supabase

# Wrangler (Cloudflare CLI) - Pages 배포용
npm install -g wrangler
```

### 1.2 계정 설정
1. [Supabase](https://supabase.com) 계정 생성
2. [Cloudflare](https://dash.cloudflare.com) 계정 생성
3. [OpenAI](https://platform.openai.com) API 키 발급

---

## 2. Supabase 프로젝트 설정

### 2.1 프로젝트 생성
1. [Supabase Dashboard](https://app.supabase.com) 접속
2. "New Project" 클릭
3. 프로젝트 정보 입력:
   - Name: `ezs-dev` (또는 원하는 이름)
   - Database Password: 안전한 비밀번호 설정
   - Region: `Northeast Asia (Seoul)` 권장
4. "Create new project" 클릭

### 2.2 프로젝트 정보 확인
Dashboard → Settings → API에서 확인:
- **Project URL**: `https://xxx.supabase.co`
- **anon/public key**: 프론트엔드에서 사용
- **service_role key**: Edge Functions에서 사용 (비공개)

### 2.3 Supabase CLI 로그인
```bash
supabase login

# 프로젝트 연결
supabase link --project-ref <your-project-ref>
```

---

## 3. 프로젝트 초기 설정

### 3.1 저장소 클론 및 의존성 설치
```bash
git clone <repository-url>
cd ezs
pnpm install
```

### 3.2 환경 변수 설정

#### apps/web/.env.local
```env
VITE_SUPABASE_URL=https://xxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJxxx...
```

#### apps/supabase/.env
```env
OPENAI_API_KEY=sk-xxx
```

---

## 4. 데이터베이스 마이그레이션

### 4.1 pgvector 확장 활성화
Supabase Dashboard → SQL Editor에서 실행:
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### 4.2 마이그레이션 파일 생성
```bash
cd apps/supabase

# 새 마이그레이션 생성
supabase migration new init

# migrations/xxx_init.sql 파일에 스키마 작성
```

### 4.3 마이그레이션 실행
```bash
# 로컬 개발 환경에 적용
supabase db reset

# 원격 프로젝트에 적용
supabase db push
```

### 4.4 주요 테이블 확인
```sql
-- 테이블 목록 확인
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';

-- 벡터 인덱스 확인
SELECT indexname FROM pg_indexes WHERE tablename = 'notes';
```

---

## 5. Supabase Auth 설정

### 5.1 Google OAuth 설정
1. [Google Cloud Console](https://console.cloud.google.com) 접속
2. OAuth 2.0 클라이언트 ID 생성
3. Authorized redirect URIs 추가:
   - `https://xxx.supabase.co/auth/v1/callback`

4. Supabase Dashboard → Authentication → Providers → Google:
   - Client ID 입력
   - Client Secret 입력
   - Enable 활성화

### 5.2 로컬 개발용 Redirect URL
Dashboard → Authentication → URL Configuration:
- Site URL: `http://localhost:5173`
- Redirect URLs: `http://localhost:5173/**`

---

## 6. Edge Functions 설정

### 6.1 로컬 개발
```bash
cd apps/supabase

# Edge Functions 개발 서버 시작
supabase functions serve

# 특정 함수만 실행
supabase functions serve ai-classify --env-file .env
```

### 6.2 시크릿 설정
```bash
# OpenAI API 키 설정 (원격)
supabase secrets set OPENAI_API_KEY=sk-xxx

# 시크릿 목록 확인
supabase secrets list
```

### 6.3 배포
```bash
# 모든 함수 배포
supabase functions deploy

# 특정 함수만 배포
supabase functions deploy ai-classify
```

---

## 7. 로컬 개발 실행

### 7.1 전체 개발 서버 시작
```bash
# 루트에서 실행 (Turborepo)
pnpm dev
```

### 7.2 개별 앱 실행
```bash
# Frontend만
cd apps/web
pnpm dev
# → http://localhost:5173

# Supabase 로컬 환경
cd apps/supabase
supabase start
# → Local Studio: http://localhost:54323
```

### 7.3 로컬 Supabase 정보
```bash
supabase status
```
출력:
```
API URL: http://localhost:54321
GraphQL URL: http://localhost:54321/graphql/v1
DB URL: postgresql://postgres:postgres@localhost:54322/postgres
Studio URL: http://localhost:54323
Inbucket URL: http://localhost:54324
anon key: eyJ...
service_role key: eyJ...
```

---

## 8. 테스트

### 8.1 테스트 실행
```bash
# 전체 테스트
pnpm test

# 특정 패키지 테스트
pnpm --filter web test
```

### 8.2 Edge Function 테스트
```bash
# 로컬에서 함수 호출
curl -i --location --request POST 'http://localhost:54321/functions/v1/ai-classify' \
  --header 'Authorization: Bearer <anon-key>' \
  --header 'Content-Type: application/json' \
  --data '{"noteId":"xxx","content":"테스트 메모"}'
```

---

## 9. 빌드 및 배포

### 9.1 로컬 빌드
```bash
pnpm build
```

### 9.2 Frontend 배포 (Cloudflare Pages)
```bash
cd apps/web
pnpm build

# Cloudflare Pages 배포
wrangler pages deploy dist --project-name=ezs-web
```

### 9.3 Supabase 배포
```bash
cd apps/supabase

# 마이그레이션 배포
supabase db push

# Edge Functions 배포
supabase functions deploy
```

### 9.4 자동 배포 (CI/CD)
`main` 또는 `staging` 브랜치에 푸시 시 GitHub Actions가 자동 배포합니다.

---

## 10. 유용한 명령어

### Supabase CLI
```bash
# 로컬 환경 시작/중지
supabase start
supabase stop

# 데이터베이스 리셋 (주의: 데이터 삭제됨)
supabase db reset

# SQL 실행
supabase db execute --sql "SELECT * FROM notes LIMIT 10"

# 타입 생성 (TypeScript)
supabase gen types typescript --local > apps/web/src/types/database.ts

# 로그 확인
supabase logs --edge-functions
```

### Turborepo
```bash
# 특정 패키지만 빌드
pnpm --filter web build

# 의존성 그래프 시각화
pnpm turbo run build --graph
```

---

## 11. 트러블슈팅

### Edge Functions 401 에러
- Authorization 헤더 확인
- anon key가 올바른지 확인
- RLS 정책이 올바르게 설정되었는지 확인

### pgvector 관련 에러
```sql
-- 확장이 활성화되어 있는지 확인
SELECT * FROM pg_extension WHERE extname = 'vector';

-- 인덱스 재생성
REINDEX INDEX notes_embedding_idx;
```

### CORS 에러
Supabase Dashboard → Settings → API → CORS:
- `http://localhost:5173` 추가
- 프로덕션 도메인 추가

### 로컬 Supabase 연결 문제
```bash
# Docker 상태 확인
docker ps

# Supabase 재시작
supabase stop && supabase start
```

---

## 12. 프로젝트 구조 요약

```
ezs/
├── apps/
│   ├── web/              # React Frontend (Cloudflare Pages)
│   │   ├── src/
│   │   ├── public/
│   │   └── package.json
│   │
│   └── supabase/         # Supabase 설정
│       ├── functions/    # Edge Functions
│       ├── migrations/   # DB 마이그레이션
│       └── config.toml   # 설정
│
├── packages/
│   └── shared/           # 공유 타입/유틸리티
│
├── docs/                 # 문서
├── turbo.json            # Turborepo 설정
├── pnpm-workspace.yaml
└── package.json
```

---

## 13. 참고 링크

- [Supabase Documentation](https://supabase.com/docs)
- [Supabase CLI Reference](https://supabase.com/docs/reference/cli)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Cloudflare Pages](https://developers.cloudflare.com/pages/)
- [Turborepo](https://turbo.build/repo)
