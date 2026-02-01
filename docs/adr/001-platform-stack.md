# ADR-001: 플랫폼 스택 선택 (Cloudflare Pages + Supabase)

## 상태
확정 (수정됨)

## 맥락
MVP를 빠르게 구축하고, **벡터 검색(Evidence Bundle)** 기능을 안정적으로 지원하면서, 운영 부담을 최소화해야 합니다.

### 핵심 요구사항
1. **벡터 검색**: 관련 노트 자동 연결 (Evidence Bundle)
2. **빠른 개발**: 인증, 스토리지, DB를 직접 구축하지 않음
3. **낮은 운영 부담**: 서버리스, 자동 스케일링
4. **비용 효율**: 초기 비용 최소화

### 고려한 대안

| 옵션 | 벡터 검색 | Auth | 개발 속도 | 비용 |
|------|----------|------|----------|------|
| **Cloudflare Pages + Supabase** | pgvector (프로덕션 레디) | 빌트인 | 빠름 | 중간 |
| Cloudflare 전체 (D1+Workers) | Vectorize (베타) | 직접 구현 | 중간 | 저렴 |
| Vercel + Supabase | pgvector | 빌트인 | 빠름 | 높음 |
| AWS (Lambda+RDS+OpenSearch) | OpenSearch | Cognito | 느림 | 높음 |

## 결정
**Cloudflare Pages (Frontend) + Supabase (Backend)** 하이브리드 구성을 선택합니다.

### 구성

```
┌─────────────────┐     ┌─────────────────────────────────┐
│ Cloudflare      │     │           Supabase              │
│ Pages           │     ├─────────────────────────────────┤
│ (React SPA)     │────▶│ PostgreSQL + pgvector           │
│                 │     │ Edge Functions (Deno)           │
│ 글로벌 CDN      │     │ Auth (OAuth, JWT, RLS)          │
│ 빠른 배포       │     │ Storage (음성 파일)              │
│                 │     │ Realtime (향후 협업)            │
└─────────────────┘     └─────────────────────────────────┘
         │                           │
         └───────────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │     OpenAI API      │
         │  (LLM + Embedding)  │
         └─────────────────────┘
```

### 선택 이유

1. **pgvector 네이티브 지원**
   - Supabase PostgreSQL에 pgvector 확장 기본 포함
   - HNSW 인덱스로 빠른 유사도 검색
   - Cloudflare Vectorize는 베타 상태로 프로덕션에 부적합

2. **빌트인 Auth**
   - Google, GitHub 등 OAuth 지원
   - JWT 자동 관리
   - RLS(Row Level Security)로 데이터 격리

3. **통합 백엔드**
   - DB, Auth, Storage, Edge Functions 통합
   - 단일 대시보드로 모니터링
   - Supabase CLI로 마이그레이션 관리

4. **Cloudflare Pages 장점 유지**
   - 글로벌 CDN (300+ PoP)
   - 빠른 빌드 및 배포
   - 무료 티어 충분

### 역할 분담

| 역할 | 서비스 | 이유 |
|------|--------|------|
| Frontend Hosting | Cloudflare Pages | 글로벌 CDN, 무료 |
| Database | Supabase PostgreSQL | pgvector, RLS |
| Vector Search | Supabase pgvector | 프로덕션 레디 |
| Auth | Supabase Auth | 빌트인, RLS 연동 |
| Storage | Supabase Storage | Auth 연동, RLS |
| Edge Functions | Supabase Edge Functions | AI 처리, DB 접근 |
| AI/LLM | OpenAI API | GPT-4o-mini, Embedding |

### 트레이드오프

| 장점 | 단점 |
|------|------|
| 벡터 검색 안정성 | Supabase Edge는 Cloudflare Workers만큼 글로벌하지 않음 |
| 빠른 개발 | 두 플랫폼 관리 필요 |
| 빌트인 Auth/RLS | Supabase Pro 비용 ($25/월) |
| PostgreSQL 강력함 | Vendor lock-in (마이그레이션 가능하지만 수고) |

### 리스크 및 대응

| 리스크 | 대응 |
|--------|------|
| Supabase Edge 지연 | 대부분의 요청은 Supabase Client로 직접 DB 접근 (Edge 우회) |
| 비용 증가 | Pro 플랜 $25/월로 시작, 사용량 모니터링 |
| pgvector 성능 | HNSW 인덱스, 필요시 별도 벡터 DB 검토 |

## 결과
- Frontend: Cloudflare Pages (React SPA)
- Backend: Supabase (PostgreSQL + pgvector + Auth + Storage + Edge Functions)
- AI: OpenAI API (GPT-4o-mini + text-embedding-3-small)
