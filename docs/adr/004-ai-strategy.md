# ADR-004: AI/LLM 전략

## 상태
확정 (수정됨)

## 맥락
자동 분류, 요약, 질문 생성, Evidence Bundle(관련 노트 연결), 산출물 생성 등 핵심 기능에 AI/LLM이 필요합니다.

### 요구사항
1. **자동 분류**: 노트를 5개 유형으로 분류 (idea/decision/todo/learning/journal)
2. **3줄 요약**: 핵심 내용 추출
3. **질문 생성**: 유형별 맞춤 질문 3개
4. **임베딩**: 관련 노트 검색용 벡터 생성
5. **산출물 생성**: ADR, If-Then, 체크리스트
6. **음성 전사**: 음성 → 텍스트 변환

## 결정

### OpenAI API 단일 스택

| 용도 | 모델 | 비용 (1K 토큰) |
|------|------|---------------|
| 분류/요약/질문 | **gpt-4o-mini** | $0.15 input / $0.6 output |
| 산출물 생성 | **gpt-4o-mini** | $0.15 input / $0.6 output |
| 임베딩 | **text-embedding-3-small** | $0.02 |
| 음성 전사 | **whisper-1** | $0.006/분 |

### 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     Supabase Edge Functions                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────┐     ┌─────────────────────────────┐   │
│   │  ai-classify    │     │       ai-embed              │   │
│   │  ───────────────│     │       ─────────────────     │   │
│   │  • 분류         │     │  • 임베딩 생성              │   │
│   │  • 요약         │     │  • pgvector 저장            │   │
│   │  • 질문 생성    │     │                             │   │
│   └─────────────────┘     └─────────────────────────────┘   │
│            │                           │                     │
│            └───────────┬───────────────┘                     │
│                        ▼                                     │
│               ┌─────────────────┐                           │
│               │   OpenAI API    │                           │
│               └─────────────────┘                           │
│                        │                                     │
│                        ▼                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              PostgreSQL + pgvector                   │   │
│   │   notes.embedding → HNSW Index → 유사 노트 검색      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 구현 상세

#### 1. 분류 + 요약 + 질문 (ai-classify)

```typescript
const SYSTEM_PROMPT = `당신은 메모 분류 전문가입니다.

## 유형 분류 기준
- idea: 새로운 아이디어, 영감, 가능성
- decision: 선택/결정이 필요한 상황
- todo: 해야 할 작업, 실행 항목
- learning: 배운 것, 인사이트, 지식
- journal: 일상 기록, 감정, 성찰

## 유형별 질문 템플릿
- decision: [의도/문제정의], [검토 중인 옵션], [성공 측정 기준]
- idea: [핵심 가치], [구현 방법], [리스크]
- todo: [완료 기준], [우선순위 근거], [의존성]
- learning: [핵심 인사이트], [적용 방법], [추가 학습]
- journal: [느낀 감정], [배운 점], [다음 행동]

JSON으로 응답:
{
  "type": "idea|decision|todo|learning|journal",
  "confidence": 0.0-1.0,
  "summary": "1. 첫 번째 요약\\n2. 두 번째 요약\\n3. 세 번째 요약",
  "questions": ["질문1", "질문2", "질문3"]
}`;

const completion = await openai.chat.completions.create({
  model: 'gpt-4o-mini',
  messages: [
    { role: 'system', content: SYSTEM_PROMPT },
    { role: 'user', content: noteContent }
  ],
  response_format: { type: 'json_object' },
  temperature: 0.3,  // 일관성 높임
  max_tokens: 500
});
```

#### 2. 임베딩 생성 (ai-embed)

```typescript
const embeddingResponse = await openai.embeddings.create({
  model: 'text-embedding-3-small',
  input: content,
  dimensions: 1536  // 기본값, 더 작게 줄일 수도 있음 (512, 256)
});

const embedding = embeddingResponse.data[0].embedding;

// pgvector에 저장
await supabase
  .from('notes')
  .update({ embedding })
  .eq('id', noteId);
```

#### 3. 유사 노트 검색 (Database Function)

```sql
-- PostgreSQL 함수로 벡터 검색
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
  SELECT embedding, user_id INTO target_embedding, target_user_id
  FROM notes WHERE notes.id = target_note_id;

  RETURN QUERY
  SELECT n.id, n.content, n.summary,
    1 - (n.embedding <=> target_embedding) AS similarity
  FROM notes n
  WHERE n.id != target_note_id
    AND n.user_id = target_user_id
    AND n.embedding IS NOT NULL
  ORDER BY n.embedding <=> target_embedding
  LIMIT match_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

#### 4. 산출물 생성 (ai-generate-artifacts)

```typescript
const ARTIFACT_PROMPT = `Decision Card를 기반으로 산출물을 생성하세요.

Decision Card:
${JSON.stringify(decisionCard)}

JSON으로 응답:
{
  "adr": "# ADR: 제목\\n\\n## 상태\\n확정\\n\\n## 맥락\\n...\\n\\n## 결정\\n...\\n\\n## 결과\\n...",
  "checklist": [
    { "text": "할 일 1", "done": false },
    { "text": "할 일 2", "done": false }
  ],
  "ifThen": [
    { "if": "조건 1", "then": "행동 1" },
    { "if": "조건 2", "then": "행동 2" }
  ],
  "verification": {
    "question": "1주 뒤 검증 질문",
    "scheduledDays": 7
  }
}`;
```

### 비용 분석 (예상 월간)

| 용도 | 호출 수 | 토큰/호출 | 예상 비용 |
|------|---------|----------|-----------|
| 분류/요약 (gpt-4o-mini) | 1,000 | ~300 in + ~200 out | ~$0.17 |
| 산출물 생성 (gpt-4o-mini) | 500 | ~500 in + ~500 out | ~$0.19 |
| 임베딩 | 1,000 | ~200 | ~$0.004 |
| 음성 전사 (10분 평균) | 100 | - | ~$0.60 |
| **총합** | | | **~$1/월** (소규모) |

*스케일업 시: 10K 노트/월 → ~$5-10/월*

### 품질 보장

#### 1. 출력 검증 (Zod)
```typescript
import { z } from 'zod';

const ClassificationSchema = z.object({
  type: z.enum(['idea', 'decision', 'todo', 'learning', 'journal']),
  confidence: z.number().min(0).max(1),
  summary: z.string().min(10),
  questions: z.array(z.string()).length(3)
});

const result = ClassificationSchema.parse(JSON.parse(response));
```

#### 2. 재시도 로직
```typescript
async function classifyWithRetry(content: string, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await openai.chat.completions.create({...});
      return ClassificationSchema.parse(JSON.parse(response.choices[0].message.content));
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

#### 3. 환각 방지
- 출처 명시: Evidence Bundle에 원본 노트 링크
- 사용자 확인: Decision Card 확정은 사용자 액션 필수
- 짧은 출력: 3줄 요약, 질문 3개로 제한 (검수 비용 최소화)

### Workers AI 대안 (비용 절감)

향후 비용 절감이 필요할 경우:

| 용도 | OpenAI | Cloudflare Workers AI |
|------|--------|----------------------|
| 분류/요약 | gpt-4o-mini | @cf/meta/llama-3.1-8b-instruct |
| 임베딩 | text-embedding-3-small | @cf/baai/bge-base-en-v1.5 |

```typescript
// Cloudflare Workers AI 폴백 예시
async function classifyNote(content: string, ai?: Ai) {
  if (ai) {
    // Workers AI 사용 (무료)
    return ai.run('@cf/meta/llama-3.1-8b-instruct', { prompt });
  } else {
    // OpenAI 사용 (유료)
    return openai.chat.completions.create({...});
  }
}
```

## 결과
- OpenAI API 단일 스택으로 시작 (품질 우선)
- gpt-4o-mini로 분류/요약/산출물 생성
- text-embedding-3-small + pgvector로 벡터 검색
- Zod로 출력 검증, 환각 방지 설계
- 필요시 Workers AI로 폴백 가능
