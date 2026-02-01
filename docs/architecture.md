# EZS (Easy Closure System) - 아키텍처 설계 문서

## 1. 개요

### 1.1 서비스 정의
파편 기록을 **자동 구조화 → 결론 확정(Closure) → 문서/실행 → 검증/리텐션**으로 변환하는 Cloudflare 기반 풀스택 애플리케이션.

### 1.2 설계 원칙
- **Edge-First**: Cloudflare의 글로벌 엣지 네트워크 활용
- **Serverless**: 서버 관리 없이 자동 스케일링
- **Low Latency**: 사용자와 가장 가까운 엣지에서 처리
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
| API Client | **TanStack Query** | 서버 상태 관리, 캐싱 |

### 2.2 Backend (Cloudflare Workers)
| 구분 | 기술 | 선택 이유 |
|------|------|-----------|
| Runtime | **Cloudflare Workers** | V8 isolates, 글로벌 엣지 |
| Framework | **Hono** | 경량, Workers 최적화, Express 유사 API |
| Language | **TypeScript** | 프론트엔드와 타입 공유 |
| Validation | **Zod** | 런타임 타입 검증 |
| Auth | **Cloudflare Access** 또는 **JWT** | Zero Trust 또는 커스텀 인증 |

### 2.3 Database & Storage (Cloudflare)
| 구분 | 서비스 | 용도 |
|------|--------|------|
| Primary DB | **D1** (SQLite) | 노트, 결정카드, 사용자 데이터 |
| Key-Value | **KV** | 세션, 캐시, 설정 |
| Object Storage | **R2** | 음성 파일, 첨부파일 |
| Full-text Search | **D1 FTS5** | 노트 검색 |

### 2.4 AI/LLM
| 구분 | 서비스 | 용도 |
|------|--------|------|
| Primary | **Cloudflare Workers AI** | 분류, 요약, 임베딩 |
| Fallback | **OpenAI API** (선택) | 복잡한 질문 생성, 고품질 요약 |
| Embedding | **@cf/baai/bge-base-en-v1.5** | 관련 메모 연결 (Evidence Bundle) |

### 2.5 인프라
| 구분 | 서비스 | 용도 |
|------|--------|------|
| Hosting | **Cloudflare Pages** | 프론트엔드 배포 |
| API | **Cloudflare Workers** | 백엔드 API |
| Queue | **Cloudflare Queues** | 비동기 작업 (음성 전사 등) |
| Cron | **Cloudflare Cron Triggers** | 주간 리뷰 알림 |
| Analytics | **Cloudflare Analytics** | 사용량 모니터링 |

---

## 3. 시스템 아키텍처

### 3.1 전체 구조도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLOUDFLARE EDGE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────────────────────────────────────┐  │
│  │   Browser    │    │              Cloudflare Pages                 │  │
│  │   (Client)   │───▶│         React SPA (Static Assets)            │  │
│  └──────────────┘    └──────────────────────────────────────────────┘  │
│         │                                                                │
│         │ API Requests                                                   │
│         ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Cloudflare Workers                            │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │                    Hono API Server                          │ │  │
│  │  │  /api/notes    /api/decisions   /api/reviews   /api/auth   │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         │              │                │                │              │
│         ▼              ▼                ▼                ▼              │
│  ┌──────────┐   ┌──────────┐    ┌──────────┐     ┌──────────────┐     │
│  │    D1    │   │    R2    │    │    KV    │     │  Workers AI  │     │
│  │ (SQLite) │   │ (Storage)│    │ (Cache)  │     │ (LLM/Embed)  │     │
│  └──────────┘   └──────────┘    └──────────┘     └──────────────┘     │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Cloudflare Queues                             │  │
│  │     [음성 전사]  [AI 분류]  [임베딩 생성]  [알림 발송]            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 데이터 흐름

```
[캡처] ──▶ [자동분류] ──▶ [요약/질문] ──▶ [Decision Card] ──▶ [산출물]
   │           │              │               │                  │
   ▼           ▼              ▼               ▼                  ▼
 D1/R2     Workers AI      Workers AI        D1              D1/Export
              │                               │
              └──────────▶ Embedding ◀───────┘
                              │
                              ▼
                      Evidence Bundle
```

---

## 4. 데이터 모델

### 4.1 ERD 개요

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   users     │     │     notes       │     │ decision_cards  │
├─────────────┤     ├─────────────────┤     ├─────────────────┤
│ id (PK)     │◀───┤│ user_id (FK)    │     │ id (PK)         │
│ email       │     │ id (PK)         │◀────│ note_id (FK)    │
│ name        │     │ content         │     │ problem         │
│ settings    │     │ type            │     │ options         │
│ created_at  │     │ summary         │     │ criteria        │
└─────────────┘     │ questions       │     │ risks           │
                    │ status          │     │ conclusion      │
                    │ embedding       │     │ status          │
                    │ created_at      │     │ closed_at       │
                    └─────────────────┘     └─────────────────┘
                           │                        │
                           ▼                        ▼
                    ┌─────────────────┐     ┌─────────────────┐
                    │ note_relations  │     │   artifacts     │
                    ├─────────────────┤     ├─────────────────┤
                    │ note_id (FK)    │     │ id (PK)         │
                    │ related_id (FK) │     │ decision_id(FK) │
                    │ similarity      │     │ type (adr/task) │
                    │ reason          │     │ content         │
                    └─────────────────┘     └─────────────────┘

                    ┌─────────────────┐     ┌─────────────────┐
                    │  review_queue   │     │  review_logs    │
                    ├─────────────────┤     ├─────────────────┤
                    │ id (PK)         │     │ id (PK)         │
                    │ note_id (FK)    │     │ queue_id (FK)   │
                    │ user_id (FK)    │     │ question        │
                    │ next_review     │     │ answer          │
                    │ interval_days   │     │ result_status   │
                    │ review_count    │     │ reviewed_at     │
                    └─────────────────┘     └─────────────────┘
```

### 4.2 테이블 상세 스키마

```sql
-- 사용자
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  settings TEXT DEFAULT '{}',  -- JSON: {reviewDay, reviewTime, promptEnabled}
  created_at INTEGER DEFAULT (unixepoch())
);

-- 노트 (메모/기록)
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  content TEXT NOT NULL,
  type TEXT CHECK(type IN ('idea','decision','todo','learning','journal')),
  type_confidence REAL DEFAULT 0.0,
  summary TEXT,                -- 3줄 요약
  questions TEXT,              -- JSON: 결론 질문 3개
  audio_url TEXT,              -- R2 URL (음성인 경우)
  status TEXT DEFAULT 'draft' CHECK(status IN ('draft','processing','ready','closed')),
  embedding BLOB,              -- 벡터 (float32 array)
  created_at INTEGER DEFAULT (unixepoch()),
  updated_at INTEGER DEFAULT (unixepoch())
);

-- 전문 검색 인덱스
CREATE VIRTUAL TABLE notes_fts USING fts5(content, summary, content=notes, content_rowid=rowid);

-- 노트 연결 (Evidence Bundle)
CREATE TABLE note_relations (
  note_id TEXT NOT NULL REFERENCES notes(id),
  related_note_id TEXT NOT NULL REFERENCES notes(id),
  similarity REAL NOT NULL,
  reason TEXT,                 -- 연결 이유
  PRIMARY KEY (note_id, related_note_id)
);

-- 결정 카드
CREATE TABLE decision_cards (
  id TEXT PRIMARY KEY,
  note_id TEXT UNIQUE NOT NULL REFERENCES notes(id),
  problem TEXT,                -- 문제 정의
  options TEXT,                -- JSON: 옵션 배열
  criteria TEXT,               -- JSON: 평가 기준
  risks TEXT,                  -- JSON: 리스크/가정
  conclusion TEXT,             -- 결론
  conclusion_reason TEXT,      -- 결론 근거
  status TEXT DEFAULT 'draft' CHECK(status IN ('draft','confirmed')),
  confirmed_at INTEGER,
  created_at INTEGER DEFAULT (unixepoch()),
  updated_at INTEGER DEFAULT (unixepoch())
);

-- 결정 카드 수정 이력
CREATE TABLE decision_revisions (
  id TEXT PRIMARY KEY,
  decision_id TEXT NOT NULL REFERENCES decision_cards(id),
  previous_state TEXT NOT NULL, -- JSON snapshot
  reason TEXT,
  created_at INTEGER DEFAULT (unixepoch())
);

-- 산출물 (ADR, 체크리스트, If-Then)
CREATE TABLE artifacts (
  id TEXT PRIMARY KEY,
  decision_id TEXT NOT NULL REFERENCES decision_cards(id),
  type TEXT CHECK(type IN ('adr','checklist','if_then','verification')),
  content TEXT NOT NULL,       -- JSON or Markdown
  status TEXT DEFAULT 'active',
  created_at INTEGER DEFAULT (unixepoch())
);

-- 주간 리뷰 큐
CREATE TABLE review_queue (
  id TEXT PRIMARY KEY,
  note_id TEXT NOT NULL REFERENCES notes(id),
  user_id TEXT NOT NULL REFERENCES users(id),
  next_review_at INTEGER NOT NULL,
  interval_days INTEGER DEFAULT 7,
  review_count INTEGER DEFAULT 0,
  created_at INTEGER DEFAULT (unixepoch())
);

-- 리뷰 로그
CREATE TABLE review_logs (
  id TEXT PRIMARY KEY,
  queue_id TEXT NOT NULL REFERENCES review_queue(id),
  question TEXT NOT NULL,
  answer TEXT,
  result_status TEXT CHECK(result_status IN ('confirmed','in_progress','discarded','needs_info')),
  reviewed_at INTEGER DEFAULT (unixepoch())
);

-- 인덱스
CREATE INDEX idx_notes_user ON notes(user_id);
CREATE INDEX idx_notes_type ON notes(type);
CREATE INDEX idx_notes_status ON notes(status);
CREATE INDEX idx_notes_created ON notes(created_at DESC);
CREATE INDEX idx_review_queue_next ON review_queue(user_id, next_review_at);
```

---

## 5. API 설계

### 5.1 RESTful API 엔드포인트

```
Base URL: /api/v1
```

#### 인증
| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/auth/login` | 로그인 (이메일/소셜) |
| POST | `/auth/logout` | 로그아웃 |
| GET | `/auth/me` | 현재 사용자 정보 |

#### 노트 (Notes)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/notes` | 노트 목록 (필터/페이지네이션) |
| POST | `/notes` | 노트 생성 (텍스트/음성) |
| GET | `/notes/:id` | 노트 상세 |
| PATCH | `/notes/:id` | 노트 수정 |
| DELETE | `/notes/:id` | 노트 삭제 |
| GET | `/notes/:id/relations` | 관련 노트 (Evidence Bundle) |
| POST | `/notes/:id/reclassify` | 수동 재분류 |

#### 결정 카드 (Decision Cards)
| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/notes/:id/decision` | Decision Card 생성 |
| GET | `/decisions/:id` | Decision Card 상세 |
| PATCH | `/decisions/:id` | Decision Card 수정 |
| POST | `/decisions/:id/confirm` | 결론 확정 (Closure) |
| POST | `/decisions/:id/revise` | 개정 생성 |

#### 산출물 (Artifacts)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/decisions/:id/artifacts` | 산출물 목록 |
| GET | `/artifacts/:id` | 산출물 상세 |
| PATCH | `/artifacts/:id` | 산출물 수정 (체크리스트 완료 등) |
| GET | `/artifacts/:id/export` | 내보내기 (Markdown) |

#### 주간 리뷰 (Weekly Review)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/reviews/pending` | 이번 주 리뷰 대상 (최대 5개) |
| POST | `/reviews/:queueId/answer` | 리뷰 답변 제출 |
| GET | `/reviews/history` | 리뷰 이력 |

#### 검색 (Search)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/search` | 통합 검색 (키워드 + 필터) |
| GET | `/search/similar` | 유사 노트 검색 (벡터) |

#### 설정 (Settings)
| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/settings` | 사용자 설정 조회 |
| PATCH | `/settings` | 사용자 설정 수정 |

### 5.2 Request/Response 예시

#### POST /api/v1/notes
```json
// Request
{
  "content": "NL2SQL message bus 적용 방안 검토",
  "type": "text"  // "text" | "audio"
}

// Response
{
  "id": "note_abc123",
  "content": "NL2SQL message bus 적용 방안 검토",
  "type": "decision",
  "typeConfidence": 0.87,
  "summary": "1. NL2SQL 아키텍처에 message bus 도입 검토\n2. 비동기 처리로 응답 시간 개선 기대\n3. 복잡도 증가와 트레이드오프 필요",
  "questions": [
    "이 결정을 내리게 된 핵심 문제는 무엇인가요?",
    "검토 중인 대안은 무엇이 있나요?",
    "이 결정의 성공을 어떻게 측정할 수 있나요?"
  ],
  "relatedNotes": [
    {
      "id": "note_xyz789",
      "summary": "기존 동기 처리 방식의 성능 이슈",
      "similarity": 0.82,
      "reason": "message bus 관련 이전 논의"
    }
  ],
  "status": "ready",
  "createdAt": "2026-02-01T10:30:00Z"
}
```

#### POST /api/v1/decisions/:id/confirm
```json
// Request
{
  "conclusion": "Kafka 기반 message bus 도입",
  "conclusionReason": "확장성과 팀 경험 고려"
}

// Response
{
  "id": "dec_def456",
  "status": "confirmed",
  "confirmedAt": "2026-02-01T11:00:00Z",
  "artifacts": {
    "adr": {
      "id": "art_001",
      "content": "# ADR-001: Kafka Message Bus 도입\n\n## 상태\n확정\n\n## 맥락\n..."
    },
    "checklist": {
      "id": "art_002",
      "items": [
        { "text": "Kafka 클러스터 설정", "done": false },
        { "text": "Producer/Consumer 구현", "done": false }
      ]
    },
    "ifThen": {
      "id": "art_003",
      "plans": [
        { "if": "다음 스프린트 시작", "then": "Kafka PoC 환경 구성" },
        { "if": "PoC 완료", "then": "프로덕션 마이그레이션 계획 수립" }
      ]
    },
    "verification": {
      "id": "art_004",
      "question": "메시지 처리 지연이 100ms 이하로 유지되고 있는가?",
      "scheduledAt": "2026-02-08T10:00:00Z"
    }
  }
}
```

---

## 6. 프로젝트 구조

### 6.1 모노레포 구조 (Turborepo)

```
ezs/
├── apps/
│   ├── web/                    # React Frontend (Cloudflare Pages)
│   │   ├── src/
│   │   │   ├── components/     # UI 컴포넌트
│   │   │   │   ├── ui/         # 기본 UI (Button, Input, Card...)
│   │   │   │   ├── notes/      # 노트 관련 컴포넌트
│   │   │   │   ├── decisions/  # Decision Card 컴포넌트
│   │   │   │   ├── reviews/    # 주간 리뷰 컴포넌트
│   │   │   │   └── layout/     # 레이아웃 컴포넌트
│   │   │   ├── pages/          # 페이지 컴포넌트
│   │   │   ├── hooks/          # 커스텀 훅
│   │   │   ├── stores/         # Zustand 스토어
│   │   │   ├── lib/            # 유틸리티
│   │   │   ├── api/            # API 클라이언트
│   │   │   └── types/          # 타입 정의
│   │   ├── public/
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   └── package.json
│   │
│   └── api/                    # Cloudflare Workers Backend
│       ├── src/
│       │   ├── routes/         # API 라우트
│       │   │   ├── auth.ts
│       │   │   ├── notes.ts
│       │   │   ├── decisions.ts
│       │   │   ├── artifacts.ts
│       │   │   ├── reviews.ts
│       │   │   └── search.ts
│       │   ├── services/       # 비즈니스 로직
│       │   │   ├── note.service.ts
│       │   │   ├── ai.service.ts
│       │   │   ├── decision.service.ts
│       │   │   └── review.service.ts
│       │   ├── repositories/   # 데이터 접근
│       │   ├── middleware/     # 미들웨어 (auth, cors, error)
│       │   ├── lib/            # 유틸리티
│       │   │   ├── db.ts       # D1 헬퍼
│       │   │   ├── ai.ts       # Workers AI 래퍼
│       │   │   └── storage.ts  # R2 헬퍼
│       │   ├── types/
│       │   └── index.ts        # 엔트리포인트
│       ├── wrangler.toml       # Cloudflare 설정
│       └── package.json
│
├── packages/
│   ├── shared/                 # 공유 코드
│   │   ├── types/              # 공유 타입 (Note, Decision, etc.)
│   │   ├── constants/          # 상수
│   │   ├── validators/         # Zod 스키마
│   │   └── package.json
│   │
│   └── ui/                     # 공유 UI 컴포넌트 (선택)
│       └── package.json
│
├── docs/                       # 문서
│   ├── architecture.md         # 이 문서
│   ├── api.md                  # API 문서
│   └── setup.md                # 개발 환경 설정
│
├── turbo.json                  # Turborepo 설정
├── pnpm-workspace.yaml
├── package.json
└── README.md
```

### 6.2 주요 설정 파일

#### turbo.json
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {},
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

#### apps/api/wrangler.toml
```toml
name = "ezs-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "ezs-db"
database_id = "<YOUR_D1_DATABASE_ID>"

[[r2_buckets]]
binding = "STORAGE"
bucket_name = "ezs-storage"

[[kv_namespaces]]
binding = "CACHE"
id = "<YOUR_KV_NAMESPACE_ID>"

[ai]
binding = "AI"

[[queues.producers]]
binding = "TASK_QUEUE"
queue = "ezs-tasks"

[[queues.consumers]]
queue = "ezs-tasks"
max_batch_size = 10
max_batch_timeout = 30

[vars]
ENVIRONMENT = "development"

# Cron Triggers (주간 리뷰 알림)
[triggers]
crons = ["0 9 * * 0"]  # 매주 일요일 09:00 UTC
```

---

## 7. 핵심 모듈 설계

### 7.1 AI 분류 서비스

```typescript
// apps/api/src/services/ai.service.ts

interface ClassificationResult {
  type: 'idea' | 'decision' | 'todo' | 'learning' | 'journal';
  confidence: number;
  summary: string;
  questions: string[];
}

export class AIService {
  constructor(private ai: Ai) {}

  async classifyAndSummarize(content: string): Promise<ClassificationResult> {
    const prompt = `다음 메모를 분석하세요:

"${content}"

1. 유형 분류 (idea/decision/todo/learning/journal)
2. 3줄 요약
3. 유형별 질문 3개:
   - decision: 의도/옵션/검증 기준
   - idea: 핵심 가치/구현 방법/리스크
   - todo: 완료 기준/우선순위/의존성
   - learning: 핵심 인사이트/적용 방법/추가 학습
   - journal: 감정/배운 점/다음 행동

JSON 형식으로 응답:
{"type": "...", "confidence": 0.0-1.0, "summary": "...", "questions": ["...", "...", "..."]}`;

    const response = await this.ai.run('@cf/meta/llama-3.1-8b-instruct', {
      prompt,
      max_tokens: 500,
    });

    return JSON.parse(response.response);
  }

  async generateEmbedding(text: string): Promise<number[]> {
    const result = await this.ai.run('@cf/baai/bge-base-en-v1.5', {
      text: [text],
    });
    return result.data[0];
  }

  async generateArtifacts(decision: DecisionCard): Promise<Artifacts> {
    // ADR, If-Then, Checklist, Verification 생성
    // ...
  }
}
```

### 7.2 Evidence Bundle 서비스

```typescript
// apps/api/src/services/evidence.service.ts

export class EvidenceService {
  constructor(
    private db: D1Database,
    private ai: AIService
  ) {}

  async findRelatedNotes(noteId: string, limit = 3): Promise<RelatedNote[]> {
    const note = await this.db
      .prepare('SELECT embedding FROM notes WHERE id = ?')
      .bind(noteId)
      .first();

    if (!note?.embedding) return [];

    // 코사인 유사도 기반 검색 (D1에서 직접 계산)
    const related = await this.db
      .prepare(`
        SELECT
          id,
          summary,
          embedding,
          (
            SELECT SUM(a.v * b.v) / (
              SQRT(SUM(a.v * a.v)) * SQRT(SUM(b.v * b.v))
            )
            FROM json_each(notes.embedding) a, json_each(?) b
            WHERE a.key = b.key
          ) as similarity
        FROM notes
        WHERE id != ? AND user_id = (SELECT user_id FROM notes WHERE id = ?)
        ORDER BY similarity DESC
        LIMIT ?
      `)
      .bind(note.embedding, noteId, noteId, limit)
      .all();

    return related.results.map((r) => ({
      id: r.id,
      summary: r.summary,
      similarity: r.similarity,
      reason: this.generateReason(r),
    }));
  }
}
```

### 7.3 주간 리뷰 스케줄러

```typescript
// apps/api/src/services/review.service.ts

export class ReviewService {
  constructor(private db: D1Database) {}

  async getPendingReviews(userId: string, limit = 5): Promise<ReviewItem[]> {
    const now = Date.now() / 1000;

    return this.db
      .prepare(`
        SELECT rq.*, n.content, n.summary, dc.conclusion
        FROM review_queue rq
        JOIN notes n ON n.id = rq.note_id
        LEFT JOIN decision_cards dc ON dc.note_id = n.id
        WHERE rq.user_id = ? AND rq.next_review_at <= ?
        ORDER BY rq.next_review_at ASC
        LIMIT ?
      `)
      .bind(userId, now, limit)
      .all();
  }

  async submitReview(
    queueId: string,
    answer: string,
    status: ReviewStatus
  ): Promise<void> {
    // 리뷰 로그 저장
    await this.db
      .prepare(`
        INSERT INTO review_logs (id, queue_id, question, answer, result_status)
        VALUES (?, ?, ?, ?, ?)
      `)
      .bind(generateId(), queueId, question, answer, status)
      .run();

    // 다음 리뷰 간격 조정 (간단 룰)
    const intervalMultiplier = status === 'confirmed' ? 1.5 : 0.75;
    await this.db
      .prepare(`
        UPDATE review_queue
        SET
          interval_days = MAX(1, MIN(30, interval_days * ?)),
          next_review_at = unixepoch() + (interval_days * 86400),
          review_count = review_count + 1
        WHERE id = ?
      `)
      .bind(intervalMultiplier, queueId)
      .run();
  }
}
```

---

## 8. 보안 고려사항

### 8.1 인증/인가
- **방식**: JWT 기반 또는 Cloudflare Access
- **세션 관리**: KV에 세션 토큰 저장 (TTL 24시간)
- **API 보호**: 모든 엔드포인트에 인증 미들웨어 적용

### 8.2 데이터 보호
- **전송 암호화**: HTTPS 강제 (Cloudflare 기본)
- **저장 암호화**: D1/R2 기본 암호화
- **개인 데이터 분리**: user_id 기반 데이터 격리

### 8.3 API 보안
- **Rate Limiting**: Cloudflare Rate Limiting 규칙
- **Input Validation**: Zod 스키마로 모든 입력 검증
- **CORS**: 허용 도메인만 접근

---

## 9. 성능 최적화

### 9.1 프론트엔드
- **Code Splitting**: React.lazy + Suspense
- **이미지 최적화**: Cloudflare Images (필요시)
- **캐싱**: TanStack Query + staleTime 설정

### 9.2 백엔드
- **Edge Caching**: KV 캐시 (검색 결과, 설정)
- **DB 최적화**: 적절한 인덱스, 페이지네이션
- **비동기 처리**: Queues로 무거운 작업 분리

### 9.3 AI 처리
- **배치 처리**: 여러 노트 동시 분류
- **캐싱**: 동일 내용 분류 결과 캐시
- **Fallback**: Workers AI 실패 시 대체 로직

---

## 10. 모니터링 & 로깅

### 10.1 메트릭
- **Cloudflare Analytics**: 요청 수, 에러율, 응답 시간
- **커스텀 메트릭**: KV에 집계 (DAU, 캡처 수, Closure율)

### 10.2 로깅
- **Workers Logs**: console.log → Cloudflare Dashboard
- **에러 추적**: Sentry 또는 Cloudflare Logpush (선택)

### 10.3 알림
- **에러 알림**: 임계값 초과 시 Slack/Email
- **성능 저하**: 응답 시간 P95 초과 시 알림

---

## 11. 배포 전략

### 11.1 환경 구성
| 환경 | 용도 | Cloudflare 프로젝트 |
|------|------|---------------------|
| Development | 로컬 개발 | - |
| Staging | 테스트/QA | ezs-staging |
| Production | 실서비스 | ezs-prod |

### 11.2 CI/CD 파이프라인 (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  deploy:
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
        run: pnpm build

      - name: Deploy API to Workers
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          workingDirectory: apps/api

      - name: Deploy Web to Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          projectName: ezs-web
          directory: apps/web/dist
```

---

## 12. MVP 개발 로드맵

### Phase 1: 기반 구축 (Week 1-2)
- [ ] 프로젝트 초기 설정 (Turborepo, TypeScript)
- [ ] Cloudflare 리소스 생성 (D1, R2, KV)
- [ ] 기본 인증 플로우
- [ ] 데이터베이스 스키마 마이그레이션
- [ ] API 기본 구조 (Hono)
- [ ] 프론트엔드 기본 구조 (React + Vite)

### Phase 2: 캡처 & 자동 구조화 (Week 3-4)
- [ ] 텍스트 캡처 (입력/저장)
- [ ] 음성 캡처 (녹음/업로드/전사)
- [ ] AI 자동 분류
- [ ] 3줄 요약 + 질문 생성
- [ ] Evidence Bundle (관련 노트 연결)

### Phase 3: Decision Card & Closure (Week 5-6)
- [ ] Decision Card UI/API
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

## 13. 참고 자료

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 Documentation](https://developers.cloudflare.com/d1/)
- [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/)
- [Hono Framework](https://hono.dev/)
- [TanStack Query](https://tanstack.com/query)
- [Zustand](https://zustand-demo.pmnd.rs/)
