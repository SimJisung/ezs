# ADR-003: 백엔드 기술 스택 (Supabase)

## 상태
확정 (수정됨)

## 맥락
벡터 검색, 인증, 스토리지가 필요한 백엔드를 빠르게 구축해야 합니다.

### 핵심 요구사항
1. **PostgreSQL**: 관계형 데이터 + 복잡한 쿼리
2. **벡터 검색**: Evidence Bundle (유사 노트 연결)
3. **인증**: OAuth + JWT + 데이터 격리
4. **스토리지**: 음성 파일 저장
5. **서버리스 함수**: AI 처리

## 결정

### Supabase 스택 선택

| 구분 | 선택 | 용도 |
|------|------|------|
| Database | **PostgreSQL 15** | 노트, 결정카드, 리뷰 |
| Vector | **pgvector + HNSW** | Evidence Bundle |
| Auth | **Supabase Auth** | OAuth, JWT, RLS |
| Storage | **Supabase Storage** | 음성 파일 |
| Functions | **Edge Functions (Deno)** | AI 처리 |
| Realtime | **Supabase Realtime** | 향후 협업 기능 |

### 데이터 접근 패턴

```
┌─────────────────────────────────────────────────────────────┐
│                       Frontend (React)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐        ┌─────────────────────────────┐ │
│  │ Supabase Client │        │     Edge Functions          │ │
│  │ (직접 DB 접근)   │        │     (AI 처리)               │ │
│  ├─────────────────┤        ├─────────────────────────────┤ │
│  │ • CRUD 작업     │        │ • 분류/요약 (OpenAI)        │ │
│  │ • 실시간 구독   │        │ • 임베딩 생성               │ │
│  │ • RPC 호출      │        │ • 산출물 생성               │ │
│  └─────────────────┘        └─────────────────────────────┘ │
│          │                            │                      │
│          └────────────┬───────────────┘                      │
│                       ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  PostgreSQL + RLS                        ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   ││
│  │  │  notes   │  │ reviews  │  │  pgvector (HNSW)     │   ││
│  │  └──────────┘  └──────────┘  └──────────────────────┘   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 선택 근거

1. **Supabase Client 직접 접근**
   ```typescript
   // 대부분의 CRUD는 클라이언트에서 직접
   const { data } = await supabase
     .from('notes')
     .select('*')
     .eq('user_id', userId);
   ```
   - 별도 API 서버 불필요
   - RLS로 보안 보장
   - 낮은 지연시간

2. **Edge Functions는 AI 처리만**
   ```typescript
   // AI 작업만 Edge Function으로
   const { data } = await supabase.functions.invoke('ai-classify', {
     body: { noteId, content }
   });
   ```
   - OpenAI API 키 보호
   - 무거운 처리 분리
   - 타임아웃 관리

3. **pgvector + HNSW**
   ```sql
   -- 벡터 인덱스
   CREATE INDEX notes_embedding_idx ON notes
   USING hnsw (embedding vector_cosine_ops);

   -- 유사 노트 검색
   SELECT id, content, 1 - (embedding <=> target_embedding) AS similarity
   FROM notes
   ORDER BY embedding <=> target_embedding
   LIMIT 3;
   ```
   - O(log n) 검색 성능
   - PostgreSQL 네이티브 통합
   - 별도 벡터 DB 불필요

4. **RLS (Row Level Security)**
   ```sql
   CREATE POLICY "Users can access own notes"
     ON notes FOR ALL
     USING (auth.uid() = user_id);
   ```
   - 데이터 격리 자동화
   - 클라이언트 직접 접근에도 안전
   - 정책 중앙 관리

### Edge Function 구조

```typescript
// supabase/functions/ai-classify/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import OpenAI from 'https://esm.sh/openai@4';

serve(async (req) => {
  // 1. 인증 확인
  const authHeader = req.headers.get('Authorization');
  const supabase = createClient(url, anonKey, {
    global: { headers: { Authorization: authHeader } }
  });

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return new Response('Unauthorized', { status: 401 });

  // 2. AI 처리
  const { noteId, content } = await req.json();
  const result = await openai.chat.completions.create({...});

  // 3. DB 업데이트 (service role key 사용)
  const adminClient = createClient(url, serviceRoleKey);
  await adminClient.from('notes').update({...}).eq('id', noteId);

  return new Response(JSON.stringify(result));
});
```

### Cloudflare Workers와 비교

| 항목 | Supabase Edge Functions | Cloudflare Workers |
|------|------------------------|-------------------|
| Runtime | Deno | V8 isolates |
| 위치 | AWS 리전 기반 | 글로벌 Edge |
| DB 접근 | Supabase 네이티브 | 외부 연결 필요 |
| 콜드 스타트 | 있음 (수백 ms) | 거의 없음 |
| 용도 | AI 처리 (빈도 낮음) | 고빈도 API (사용 안 함) |

**결론**: 대부분의 요청은 Supabase Client로 직접 DB 접근하므로, Edge Function의 콜드 스타트는 AI 처리 시에만 영향.

## 결과
- Supabase PostgreSQL + pgvector로 데이터 + 벡터 검색
- Supabase Client로 대부분의 CRUD 직접 처리
- Edge Functions는 AI 처리 전용
- RLS로 데이터 보안 자동화
