# EZS (Easy Closure System) - 아키텍처 설계 문서

## 1. 개요

### 1.1 서비스 정의
파편 기록을 **자동 구조화 → 결론 확정(Closure) → 문서/실행 → 검증/리텐션**으로 변환하는 풀스택 애플리케이션.

### 1.2 설계 원칙
- **Serverless**: 서버 관리 없이 자동 스케일링
- **Vector-Native**: pgvector로 의미 기반 검색 지원
- **Auth-First**: Supabase Auth로 안전한 인증
- **Cost Efficient**: 사용량 기반 과금, 초기 비용 최소화

---

## 2. 기술 스택

### 2.1 Frontend
| 구분 | 기술 | 선택 이유 |
|------|------|-----------|
| Framework | **React 18** | 컴포넌트 기반, 풍부한 생태계 |
| Build Tool | **Vite** | 빠른 HMR, ESBuild 기반 번들링 |
| Language | **TypeScript** | 타입 안정성, IDE 지원 |
| State | **Zustand** | 경량, 보일러플레이트 최소화 |
| Styling | **Tailwind CSS** | 유틸리티 퍼스트, 빠른 개발 |
| UI Components | **Radix UI** | 접근성, 헤드리스 컴포넌트 |
| Form | **React Hook Form + Zod** | 타입 안전한 폼 검증 |
| Router | **React Router v7** | 표준 라우팅 |
| API Client | **TanStack Query + Supabase Client** | 서버 상태 관리, 실시간 구독 |

### 2.2 Backend (Supabase)
| 구분 | 기술 | 선택 이유 |
|------|------|-----------|
| Platform | **Supabase** | PostgreSQL + 통합 백엔드 서비스 |
| Database | **PostgreSQL 15** | 강력한 SQL, ACID, 확장성 |
| Vector Search | **pgvector** | 네이티브 벡터 검색, Evidence Bundle |
| Auth | **Supabase Auth** | 소셜 로그인, JWT, RLS |
| Edge Functions | **Deno Runtime** | 서버리스 API, TypeScript |
| Realtime | **Supabase Realtime** | WebSocket 기반 실시간 동기화 |
| Storage | **Supabase Storage** | 음성 파일, 첨부파일 |

### 2.3 AI/LLM
| 구분 | 서비스 | 용도 |
|------|--------|------|
| LLM | **OpenAI GPT-4o-mini** | 분류, 요약, 질문 생성, 산출물 생성 |
| Embedding | **OpenAI text-embedding-3-small** | 벡터 임베딩 (1536차원) |
| 음성 전사 | **OpenAI Whisper API** | 음성 → 텍스트 변환 |

### 2.4 인프라
| 구분 | 서비스 | 용도 |
|------|--------|------|
| Frontend Hosting | **Cloudflare Pages** | 글로벌 CDN, 빠른 배포 |
| Backend | **Supabase** | 통합 백엔드 플랫폼 |
| Cron | **Supabase pg_cron** | 주간 리뷰 알림, 배치 작업 |
| Monitoring | **Supabase Dashboard** | 로그, 메트릭, 알림 |

---

## 3. 시스템 아키텍처

### 3.1 전체 구조도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────────────────────────────────────┐  │
│  │   Browser    │    │              Cloudflare Pages                 │  │
│  │   (Client)   │───▶│         React SPA (Static Assets)            │  │
│  └──────────────┘    └──────────────────────────────────────────────┘  │
│         │                                                                │
│         │ Supabase Client (Auth, DB, Realtime, Storage)                 │
│         ▼                                                                │
├─────────────────────────────────────────────────────────────────────────┤
│                              SUPABASE                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Supabase Edge Functions                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │              Deno Runtime (TypeScript)                      │ │  │
│  │  │  /api/notes    /api/ai/classify   /api/ai/embed            │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         │              │                │                │              │
│         ▼              ▼                ▼                ▼              │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     PostgreSQL + pgvector                         │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────────────┐ │  │
│  │  │  notes  │ │decisions│ │ reviews │ │  Vector Index (HNSW)   │ │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         │                                                    │          │
│         ▼                                                    ▼          │
│  ┌──────────────┐                              ┌──────────────────────┐│
│  │   Storage    │                              │    Supabase Auth     ││
│  │ (음성 파일)   │                              │  (JWT + RLS)         ││
│  └──────────────┘                              └──────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │      OpenAI API      │
                         │  GPT-4o / Embedding  │
                         └──────────────────────┘
```

### 3.2 데이터 흐름

```
[캡처] ──▶ [자동분류] ──▶ [요약/질문] ──▶ [Decision Card] ──▶ [산출물]
   │           │              │               │                  │
   ▼           ▼              ▼               ▼                  ▼
 Supabase   OpenAI API    OpenAI API     PostgreSQL        PostgreSQL
 Storage                                      │
                                              │
              ┌───────────────────────────────┘
              ▼
        [Embedding 생성]
              │
              ▼
        [pgvector 저장]
              │
              ▼
        [Evidence Bundle]
        (유사 노트 검색)
```

### 3.3 인증 흐름

```
┌──────────┐     ┌──────────────┐     ┌─────────────────┐
│  Client  │────▶│ Supabase Auth│────▶│   PostgreSQL    │
│          │     │   (OAuth)    │     │   (RLS Policy)  │
└──────────┘     └──────────────┘     └─────────────────┘
     │                 │                      │
     │    JWT Token    │      user_id         │
     │◀────────────────│──────────────────────│
     │                                        │
     │         Row Level Security             │
     │      (자동으로 user_id 필터링)          │
     └────────────────────────────────────────┘
```

---

## 4. 데이터 모델

### 4.1 ERD 개요

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  auth.users     │     │     notes       │     │ decision_cards  │
│  (Supabase)     │     ├─────────────────┤     ├─────────────────┤
├─────────────────┤     │ id (PK)         │     │ id (PK)         │
│ id (UUID)       │◀────│ user_id (FK)    │     │ note_id (FK)    │◀┐
│ email           │     │ content         │────▶│ problem         │ │
│ ...             │     │ type            │     │ options         │ │
└─────────────────┘     │ summary         │     │ criteria        │ │
                        │ questions       │     │ risks           │ │
┌─────────────────┐     │ embedding       │     │ conclusion      │ │
│   profiles      │     │ status          │     │ status          │ │
├─────────────────┤     │ created_at      │     │ confirmed_at    │ │
│ id (FK→auth)    │     └─────────────────┘     └─────────────────┘ │
│ display_name    │            │                        │           │
│ settings        │            │                        ▼           │
│ created_at      │            │                ┌─────────────────┐ │
└─────────────────┘            │                │   artifacts     │ │
                               │                ├─────────────────┤ │
                               │                │ id (PK)         │ │
                               │                │ decision_id(FK) │─┘
                               │                │ type            │
                               ▼                │ content         │
                        ┌─────────────────┐     └─────────────────┘
                        │  review_queue   │
                        ├─────────────────┤     ┌─────────────────┐
                        │ id (PK)         │     │  review_logs    │
                        │ note_id (FK)    │     ├─────────────────┤
                        │ user_id (FK)    │◀────│ queue_id (FK)   │
                        │ next_review_at  │     │ question        │
                        │ interval_days   │     │ answer          │
                        │ review_count    │     │ result_status   │
                        └─────────────────┘     └─────────────────┘
```

### 4.2 테이블 상세 스키마 (PostgreSQL)

```sql
-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- 사용자 프로필 (Supabase Auth 연동)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name TEXT,
  settings JSONB DEFAULT '{
    "reviewDay": 0,
    "reviewTime": "09:00",
    "promptEnabled": true,
    "timezone": "Asia/Seoul"
  }'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 노트 (메모/기록)
CREATE TABLE notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  type TEXT CHECK(type IN ('idea','decision','todo','learning','journal')),
  type_confidence REAL DEFAULT 0.0,
  summary TEXT,
  questions JSONB DEFAULT '[]'::jsonb,
  audio_url TEXT,
  status TEXT DEFAULT 'draft' CHECK(status IN ('draft','processing','ready','closed')),
  embedding vector(1536),  -- OpenAI text-embedding-3-small
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 벡터 검색을 위한 HNSW 인덱스
CREATE INDEX notes_embedding_idx ON notes
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 전문 검색 인덱스
CREATE INDEX notes_content_search_idx ON notes
USING gin (to_tsvector('korean', content));

-- 결정 카드
CREATE TABLE decision_cards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  note_id UUID UNIQUE NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
  problem TEXT,
  options JSONB DEFAULT '[]'::jsonb,
  criteria JSONB DEFAULT '[]'::jsonb,
  risks JSONB DEFAULT '[]'::jsonb,
  conclusion TEXT,
  conclusion_reason TEXT,
  status TEXT DEFAULT 'draft' CHECK(status IN ('draft','confirmed')),
  confirmed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 결정 카드 수정 이력
CREATE TABLE decision_revisions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  decision_id UUID NOT NULL REFERENCES decision_cards(id) ON DELETE CASCADE,
  previous_state JSONB NOT NULL,
  reason TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 산출물 (ADR, 체크리스트, If-Then)
CREATE TABLE artifacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  decision_id UUID NOT NULL REFERENCES decision_cards(id) ON DELETE CASCADE,
  type TEXT CHECK(type IN ('adr','checklist','if_then','verification')),
  content JSONB NOT NULL,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 주간 리뷰 큐
CREATE TABLE review_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  next_review_at TIMESTAMPTZ NOT NULL,
  interval_days INTEGER DEFAULT 7,
  review_count INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 리뷰 로그
CREATE TABLE review_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  queue_id UUID NOT NULL REFERENCES review_queue(id) ON DELETE CASCADE,
  question TEXT NOT NULL,
  answer TEXT,
  result_status TEXT CHECK(result_status IN ('confirmed','in_progress','discarded','needs_info')),
  reviewed_at TIMESTAMPTZ DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_notes_user ON notes(user_id);
CREATE INDEX idx_notes_type ON notes(type);
CREATE INDEX idx_notes_status ON notes(status);
CREATE INDEX idx_notes_created ON notes(created_at DESC);
CREATE INDEX idx_review_queue_next ON review_queue(user_id, next_review_at);
CREATE INDEX idx_decision_cards_note ON decision_cards(note_id);
CREATE INDEX idx_artifacts_decision ON artifacts(decision_id);

-- RLS (Row Level Security) 정책
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;
ALTER TABLE decision_cards ENABLE ROW LEVEL SECURITY;
ALTER TABLE artifacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE review_queue ENABLE ROW LEVEL SECURITY;
ALTER TABLE review_logs ENABLE ROW LEVEL SECURITY;

-- 사용자별 데이터 접근 정책
CREATE POLICY "Users can access own profile"
  ON profiles FOR ALL
  USING (auth.uid() = id);

CREATE POLICY "Users can access own notes"
  ON notes FOR ALL
  USING (auth.uid() = user_id);

CREATE POLICY "Users can access own decisions"
  ON decision_cards FOR ALL
  USING (
    note_id IN (SELECT id FROM notes WHERE user_id = auth.uid())
  );

CREATE POLICY "Users can access own artifacts"
  ON artifacts FOR ALL
  USING (
    decision_id IN (
      SELECT dc.id FROM decision_cards dc
      JOIN notes n ON dc.note_id = n.id
      WHERE n.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can access own review queue"
  ON review_queue FOR ALL
  USING (auth.uid() = user_id);

CREATE POLICY "Users can access own review logs"
  ON review_logs FOR ALL
  USING (
    queue_id IN (SELECT id FROM review_queue WHERE user_id = auth.uid())
  );

-- 프로필 자동 생성 트리거
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id)
  VALUES (NEW.id);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE FUNCTION handle_new_user();

-- updated_at 자동 갱신 트리거
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER notes_updated_at
  BEFORE UPDATE ON notes
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER decision_cards_updated_at
  BEFORE UPDATE ON decision_cards
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();
```

### 4.3 벡터 검색 함수

```sql
-- 유사 노트 검색 (Evidence Bundle)
CREATE OR REPLACE FUNCTION find_similar_notes(
  target_note_id UUID,
  match_count INT DEFAULT 3
)
RETURNS TABLE (
  id UUID,
  content TEXT,
  summary TEXT,
  similarity FLOAT
) AS $$
DECLARE
  target_embedding vector(1536);
  target_user_id UUID;
BEGIN
  -- 대상 노트의 임베딩과 user_id 조회
  SELECT embedding, user_id INTO target_embedding, target_user_id
  FROM notes
  WHERE notes.id = target_note_id;

  -- 유사 노트 반환
  RETURN QUERY
  SELECT
    n.id,
    n.content,
    n.summary,
    1 - (n.embedding <=> target_embedding) AS similarity
  FROM notes n
  WHERE n.id != target_note_id
    AND n.user_id = target_user_id
    AND n.embedding IS NOT NULL
  ORDER BY n.embedding <=> target_embedding
  LIMIT match_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 텍스트로 유사 노트 검색
CREATE OR REPLACE FUNCTION search_notes_by_embedding(
  query_embedding vector(1536),
  user_uuid UUID,
  match_count INT DEFAULT 10
)
RETURNS TABLE (
  id UUID,
  content TEXT,
  summary TEXT,
  type TEXT,
  similarity FLOAT
) AS $$
BEGIN
  RETURN QUERY
  SELECT
    n.id,
    n.content,
    n.summary,
    n.type,
    1 - (n.embedding <=> query_embedding) AS similarity
  FROM notes n
  WHERE n.user_id = user_uuid
    AND n.embedding IS NOT NULL
  ORDER BY n.embedding <=> query_embedding
  LIMIT match_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 5. API 설계

### 5.1 Supabase Client (직접 호출)

대부분의 CRUD는 Supabase Client로 직접 처리합니다.

```typescript
// Frontend에서 직접 호출
const { data, error } = await supabase
  .from('notes')
  .select('*')
  .order('created_at', { ascending: false });
```

### 5.2 Edge Functions (AI 처리)

AI 관련 작업은 Edge Functions로 처리합니다.

```
Base URL: https://<project-ref>.supabase.co/functions/v1
```

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/ai/classify` | 노트 자동 분류 + 요약 + 질문 생성 |
| POST | `/ai/embed` | 임베딩 생성 + 저장 |
| POST | `/ai/generate-artifacts` | ADR/체크리스트/If-Then 생성 |
| POST | `/ai/transcribe` | 음성 전사 (Whisper) |

### 5.3 Database Functions (RPC)

복잡한 쿼리는 Database Functions로 처리합니다.

| Function | 설명 |
|----------|------|
| `find_similar_notes(note_id, count)` | Evidence Bundle 조회 |
| `search_notes_by_embedding(embedding, user_id, count)` | 벡터 검색 |
| `get_pending_reviews(user_id, limit)` | 주간 리뷰 대상 조회 |
| `update_review_interval(queue_id, status)` | 스페이싱 간격 조정 |

### 5.4 Request/Response 예시

#### POST /functions/v1/ai/classify

```typescript
// Request
{
  "noteId": "uuid-xxx",
  "content": "NL2SQL message bus 적용 방안 검토"
}

// Response
{
  "type": "decision",
  "confidence": 0.87,
  "summary": "1. NL2SQL 아키텍처에 message bus 도입 검토\n2. 비동기 처리로 응답 시간 개선 기대\n3. 복잡도 증가와 트레이드오프 필요",
  "questions": [
    "이 결정을 내리게 된 핵심 문제는 무엇인가요?",
    "검토 중인 대안은 무엇이 있나요?",
    "이 결정의 성공을 어떻게 측정할 수 있나요?"
  ]
}
```

#### RPC: find_similar_notes

```typescript
const { data: relatedNotes } = await supabase
  .rpc('find_similar_notes', {
    target_note_id: noteId,
    match_count: 3
  });

// Response
[
  {
    "id": "uuid-yyy",
    "content": "기존 동기 처리 방식의 성능 이슈",
    "summary": "...",
    "similarity": 0.82
  }
]
```

---

## 6. 프로젝트 구조

### 6.1 모노레포 구조

```
ezs/
├── apps/
│   ├── web/                      # React Frontend (Cloudflare Pages)
│   │   ├── src/
│   │   │   ├── components/       # UI 컴포넌트
│   │   │   │   ├── ui/           # 기본 UI (Button, Input, Card...)
│   │   │   │   ├── notes/        # 노트 관련 컴포넌트
│   │   │   │   ├── decisions/    # Decision Card 컴포넌트
│   │   │   │   ├── reviews/      # 주간 리뷰 컴포넌트
│   │   │   │   └── layout/       # 레이아웃 컴포넌트
│   │   │   ├── pages/            # 페이지 컴포넌트
│   │   │   ├── hooks/            # 커스텀 훅
│   │   │   ├── stores/           # Zustand 스토어
│   │   │   ├── lib/
│   │   │   │   ├── supabase.ts   # Supabase 클라이언트
│   │   │   │   └── utils.ts      # 유틸리티
│   │   │   └── types/            # 타입 정의
│   │   ├── public/
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   └── package.json
│   │
│   └── supabase/                 # Supabase 설정 및 Edge Functions
│       ├── functions/            # Edge Functions
│       │   ├── ai-classify/
│       │   │   └── index.ts
│       │   ├── ai-embed/
│       │   │   └── index.ts
│       │   ├── ai-generate-artifacts/
│       │   │   └── index.ts
│       │   └── ai-transcribe/
│       │       └── index.ts
│       ├── migrations/           # 데이터베이스 마이그레이션
│       │   ├── 20240201000000_init.sql
│       │   ├── 20240201000001_vector.sql
│       │   └── 20240201000002_functions.sql
│       ├── seed.sql              # 시드 데이터
│       └── config.toml           # Supabase 설정
│
├── packages/
│   └── shared/                   # 공유 코드
│       ├── types/                # 공유 타입 (Note, Decision, etc.)
│       ├── constants/            # 상수
│       ├── validators/           # Zod 스키마
│       └── package.json
│
├── docs/                         # 문서
│   ├── architecture.md           # 이 문서
│   ├── setup.md                  # 개발 환경 설정
│   └── adr/                      # Architecture Decision Records
│
├── turbo.json                    # Turborepo 설정
├── pnpm-workspace.yaml
├── package.json
└── README.md
```

### 6.2 주요 설정 파일

#### apps/supabase/config.toml
```toml
[project]
project_id = "your-project-ref"

[api]
enabled = true
port = 54321
schemas = ["public", "graphql_public"]
extra_search_path = ["public", "extensions"]
max_rows = 1000

[db]
port = 54322
shadow_port = 54320
major_version = 15

[studio]
enabled = true
port = 54323

[auth]
enabled = true
site_url = "http://localhost:5173"
additional_redirect_urls = ["https://ezs.pages.dev"]

[auth.external.google]
enabled = true
client_id = "env(GOOGLE_CLIENT_ID)"
secret = "env(GOOGLE_CLIENT_SECRET)"

[storage]
enabled = true
file_size_limit = "50MiB"
```

#### apps/web/.env.local
```env
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```

---

## 7. 핵심 모듈 설계

### 7.1 Supabase 클라이언트 설정

```typescript
// apps/web/src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';
import type { Database } from '@/types/database';

export const supabase = createClient<Database>(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY
);
```

### 7.2 AI 분류 Edge Function

```typescript
// apps/supabase/functions/ai-classify/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import OpenAI from 'https://esm.sh/openai@4';

const openai = new OpenAI({
  apiKey: Deno.env.get('OPENAI_API_KEY'),
});

serve(async (req) => {
  const { noteId, content } = await req.json();

  // 분류 + 요약 + 질문 생성
  const completion = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    messages: [
      {
        role: 'system',
        content: `당신은 메모 분류 전문가입니다. 다음 메모를 분석하세요.

유형 분류 (idea/decision/todo/learning/journal):
- idea: 새로운 아이디어, 영감, 가능성
- decision: 선택/결정이 필요한 상황
- todo: 해야 할 작업, 실행 항목
- learning: 배운 것, 인사이트, 지식
- journal: 일상 기록, 감정, 성찰

JSON으로 응답:
{
  "type": "...",
  "confidence": 0.0-1.0,
  "summary": "3줄 요약 (줄바꿈으로 구분)",
  "questions": ["질문1", "질문2", "질문3"]
}`,
      },
      {
        role: 'user',
        content,
      },
    ],
    response_format: { type: 'json_object' },
  });

  const result = JSON.parse(completion.choices[0].message.content);

  // 데이터베이스 업데이트
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  await supabase
    .from('notes')
    .update({
      type: result.type,
      type_confidence: result.confidence,
      summary: result.summary,
      questions: result.questions,
      status: 'ready',
    })
    .eq('id', noteId);

  return new Response(JSON.stringify(result), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

### 7.3 임베딩 생성 Edge Function

```typescript
// apps/supabase/functions/ai-embed/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import OpenAI from 'https://esm.sh/openai@4';

const openai = new OpenAI({
  apiKey: Deno.env.get('OPENAI_API_KEY'),
});

serve(async (req) => {
  const { noteId, content } = await req.json();

  // 임베딩 생성
  const embeddingResponse = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: content,
  });

  const embedding = embeddingResponse.data[0].embedding;

  // 데이터베이스 저장
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  await supabase
    .from('notes')
    .update({ embedding })
    .eq('id', noteId);

  return new Response(JSON.stringify({ success: true }), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

### 7.4 Evidence Bundle 훅

```typescript
// apps/web/src/hooks/useRelatedNotes.ts
import { useQuery } from '@tanstack/react-query';
import { supabase } from '@/lib/supabase';

export function useRelatedNotes(noteId: string | undefined) {
  return useQuery({
    queryKey: ['related-notes', noteId],
    queryFn: async () => {
      if (!noteId) return [];

      const { data, error } = await supabase.rpc('find_similar_notes', {
        target_note_id: noteId,
        match_count: 3,
      });

      if (error) throw error;
      return data;
    },
    enabled: !!noteId,
  });
}
```

### 7.5 주간 리뷰 서비스

```typescript
// apps/web/src/hooks/useReviews.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { supabase } from '@/lib/supabase';

export function usePendingReviews() {
  return useQuery({
    queryKey: ['pending-reviews'],
    queryFn: async () => {
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) throw new Error('Not authenticated');

      const { data, error } = await supabase
        .from('review_queue')
        .select(`
          *,
          notes (id, content, summary),
          decision_cards (id, conclusion)
        `)
        .eq('user_id', user.id)
        .lte('next_review_at', new Date().toISOString())
        .order('next_review_at')
        .limit(5);

      if (error) throw error;
      return data;
    },
  });
}

export function useSubmitReview() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      queueId,
      question,
      answer,
      status,
    }: {
      queueId: string;
      question: string;
      answer: string;
      status: 'confirmed' | 'in_progress' | 'discarded' | 'needs_info';
    }) => {
      // 리뷰 로그 저장
      await supabase.from('review_logs').insert({
        queue_id: queueId,
        question,
        answer,
        result_status: status,
      });

      // 다음 리뷰 간격 조정
      const intervalMultiplier = status === 'confirmed' ? 1.5 : 0.75;

      const { data: queue } = await supabase
        .from('review_queue')
        .select('interval_days, review_count')
        .eq('id', queueId)
        .single();

      if (queue) {
        const newInterval = Math.max(1, Math.min(30,
          Math.round(queue.interval_days * intervalMultiplier)
        ));

        await supabase
          .from('review_queue')
          .update({
            interval_days: newInterval,
            next_review_at: new Date(
              Date.now() + newInterval * 24 * 60 * 60 * 1000
            ).toISOString(),
            review_count: queue.review_count + 1,
          })
          .eq('id', queueId);
      }
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['pending-reviews'] });
    },
  });
}
```

---

## 8. 보안 고려사항

### 8.1 인증/인가
- **Supabase Auth**: OAuth (Google, GitHub), Magic Link
- **JWT**: 자동 관리, 만료 시 자동 갱신
- **RLS (Row Level Security)**: 모든 테이블에 적용, user_id 기반 격리

### 8.2 데이터 보호
- **전송 암호화**: HTTPS 강제
- **저장 암호화**: Supabase 기본 암호화
- **API 키 분리**: anon key (공개), service role key (서버만)

### 8.3 Edge Function 보안
```typescript
// 인증 확인
const authHeader = req.headers.get('Authorization');
const supabase = createClient(url, anonKey, {
  global: { headers: { Authorization: authHeader } },
});

const { data: { user }, error } = await supabase.auth.getUser();
if (error || !user) {
  return new Response('Unauthorized', { status: 401 });
}
```

---

## 9. 성능 최적화

### 9.1 프론트엔드
- **Code Splitting**: React.lazy + Suspense
- **캐싱**: TanStack Query (staleTime: 5분)
- **Optimistic Updates**: 즉각적인 UI 반응

### 9.2 데이터베이스
- **인덱스**: 쿼리 패턴에 맞는 인덱스 설계
- **HNSW 인덱스**: 벡터 검색 가속 (O(log n))
- **Connection Pooling**: Supabase Pooler (Supavisor)

### 9.3 벡터 검색 최적화
```sql
-- HNSW 인덱스 파라미터
-- m: 각 노드의 연결 수 (높을수록 정확, 느림)
-- ef_construction: 인덱스 구축 시 탐색 범위
CREATE INDEX notes_embedding_idx ON notes
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 검색 시 ef_search 조정 (정확도 vs 속도)
SET hnsw.ef_search = 40;
```

---

## 10. 모니터링 & 로깅

### 10.1 Supabase Dashboard
- **Database**: 쿼리 성능, 연결 수, 스토리지
- **Auth**: 로그인 시도, 활성 사용자
- **Edge Functions**: 실행 횟수, 에러, 지연 시간
- **Storage**: 사용량, 대역폭

### 10.2 커스텀 로깅
```typescript
// Edge Function 로깅
console.log(JSON.stringify({
  event: 'ai_classify',
  noteId,
  type: result.type,
  confidence: result.confidence,
  latencyMs: Date.now() - startTime,
}));
```

### 10.3 알림
- Supabase Webhooks → Slack/Discord

---

## 11. 배포 전략

### 11.1 환경 구성
| 환경 | Frontend | Supabase 프로젝트 |
|------|----------|-------------------|
| Development | localhost:5173 | ezs-dev |
| Staging | ezs-staging.pages.dev | ezs-staging |
| Production | ezs.pages.dev | ezs-prod |

### 11.2 CI/CD 파이프라인 (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  deploy-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v2

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install

      - name: Build
        run: pnpm --filter web build
        env:
          VITE_SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          VITE_SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          projectName: ezs-web
          directory: apps/web/dist

  deploy-supabase:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Supabase CLI
        uses: supabase/setup-cli@v1

      - name: Deploy migrations
        run: supabase db push --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}

      - name: Deploy Edge Functions
        run: supabase functions deploy --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
```

---

## 12. MVP 개발 로드맵

### Phase 1: 기반 구축 (Week 1-2)
- [ ] 프로젝트 초기 설정 (Turborepo, TypeScript)
- [ ] Supabase 프로젝트 생성
- [ ] 데이터베이스 스키마 마이그레이션
- [ ] Supabase Auth 설정 (Google OAuth)
- [ ] 프론트엔드 기본 구조 (React + Vite)
- [ ] Supabase 클라이언트 설정

### Phase 2: 캡처 & 자동 구조화 (Week 3-4)
- [ ] 텍스트 캡처 (입력/저장)
- [ ] 음성 캡처 (녹음/업로드/전사)
- [ ] AI 분류 Edge Function
- [ ] 임베딩 생성 Edge Function
- [ ] Evidence Bundle (pgvector 검색)

### Phase 3: Decision Card & Closure (Week 5-6)
- [ ] Decision Card UI/로직
- [ ] 질문 기반 필드 자동 채움
- [ ] 결론 확정 (Closure) 플로우
- [ ] 산출물 자동 생성 (ADR, If-Then, 체크리스트)

### Phase 4: 주간 리뷰 & 마무리 (Week 7-8)
- [ ] 주간 리뷰 큐 로직
- [ ] 리뷰 UI (질문/답변)
- [ ] 스페이싱 간격 조정
- [ ] 검색/필터
- [ ] 설정 화면
- [ ] 버그 수정 & 최적화

---

## 13. 비용 예측 (월간)

| 서비스 | 무료 티어 | 예상 사용량 | 예상 비용 |
|--------|-----------|-------------|-----------|
| Supabase (Pro) | - | DB + Auth + Storage | $25/월 |
| Cloudflare Pages | 무제한 | 정적 호스팅 | $0 |
| OpenAI API | - | 10K 분류 + 10K 임베딩 | $5-10/월 |
| **총합** | | | **$30-35/월** |

---

## 14. 참고 자료

- [Supabase Documentation](https://supabase.com/docs)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [OpenAI API](https://platform.openai.com/docs)
- [TanStack Query](https://tanstack.com/query)
- [Cloudflare Pages](https://developers.cloudflare.com/pages/)
