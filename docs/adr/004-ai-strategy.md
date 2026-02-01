# ADR-004: AI/LLM 전략

## 상태
확정

## 맥락
자동 분류, 요약, 질문 생성, Evidence Bundle(관련 노트 연결) 등 핵심 기능에 AI/LLM이 필요합니다. 비용, 지연시간, 품질의 균형을 맞춰야 합니다.

### 요구사항
1. **자동 분류**: 노트를 5개 유형으로 분류 (idea/decision/todo/learning/journal)
2. **3줄 요약**: 핵심 내용 추출
3. **질문 생성**: 유형별 맞춤 질문 3개
4. **임베딩**: 관련 노트 검색용 벡터 생성
5. **산출물 생성**: ADR, If-Then, 체크리스트

### 고려한 대안

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Cloudflare Workers AI** | Edge 실행, 낮은 지연, 통합 환경 | 모델 제한, 커스터마이징 어려움 |
| OpenAI API | 최고 품질, 다양한 모델 | 비용, 외부 의존성, 지연 |
| Self-hosted (Ollama) | 비용 없음, 프라이버시 | 인프라 관리, 스케일링 어려움 |
| 하이브리드 | 상황별 최적화 | 복잡도 증가 |

## 결정

### 하이브리드 전략 채택

```
┌─────────────────────────────────────────────────────────┐
│                      AI Strategy                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   ┌─────────────────┐     ┌─────────────────────────┐  │
│   │ Workers AI      │     │ OpenAI API (Fallback)   │  │
│   │ (Primary)       │     │                         │  │
│   ├─────────────────┤     ├─────────────────────────┤  │
│   │ • 분류          │     │ • 복잡한 산출물 생성   │  │
│   │ • 간단한 요약    │     │ • Workers AI 실패 시   │  │
│   │ • 임베딩        │     │ • 고품질 필요 시       │  │
│   └─────────────────┘     └─────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 모델 선택

| 용도 | Primary (Workers AI) | Fallback (OpenAI) |
|------|---------------------|-------------------|
| 분류 | `@cf/meta/llama-3.1-8b-instruct` | `gpt-4o-mini` |
| 요약/질문 | `@cf/meta/llama-3.1-8b-instruct` | `gpt-4o-mini` |
| 임베딩 | `@cf/baai/bge-base-en-v1.5` | `text-embedding-3-small` |
| ADR 생성 | - | `gpt-4o` (고품질 필요 시) |

### 구현 전략

#### 1. Workers AI 우선
```typescript
class AIService {
  async classify(content: string): Promise<ClassificationResult> {
    try {
      // Workers AI 시도
      return await this.workersAI.classify(content);
    } catch (error) {
      // OpenAI 폴백
      if (this.openaiKey) {
        return await this.openai.classify(content);
      }
      throw error;
    }
  }
}
```

#### 2. 프롬프트 최적화
```typescript
const CLASSIFICATION_PROMPT = `
당신은 메모 분류 전문가입니다. 다음 메모를 분석하세요:

"{{content}}"

다음 5가지 유형 중 하나로 분류하세요:
- idea: 새로운 아이디어, 영감, 가능성
- decision: 선택/결정이 필요한 상황
- todo: 해야 할 작업, 실행 항목
- learning: 배운 것, 인사이트, 지식
- journal: 일상 기록, 감정, 성찰

JSON으로 응답:
{"type": "...", "confidence": 0.0-1.0, "reason": "한 줄 이유"}
`;
```

#### 3. 임베딩 저장 및 검색
```typescript
// 노트 저장 시 임베딩 생성
async saveNote(note: Note): Promise<void> {
  const embedding = await this.ai.generateEmbedding(note.content);

  await this.db
    .prepare('UPDATE notes SET embedding = ? WHERE id = ?')
    .bind(JSON.stringify(embedding), note.id)
    .run();
}

// 유사 노트 검색 (코사인 유사도)
async findSimilar(noteId: string, limit = 3): Promise<Note[]> {
  // D1에서 벡터 연산 직접 수행 (JSON array)
  // 또는 별도 벡터 DB 사용 (Vectorize - 베타)
}
```

### 비용 분석 (예상 월간)

| 사용량 | Workers AI | OpenAI 폴백 |
|--------|-----------|-------------|
| 분류 1만 건 | 무료 (10만 건/일) | $2-3 |
| 임베딩 1만 건 | 무료 | $0.5 |
| 산출물 1천 건 | - | $5-10 |
| **총 예상** | **$0** | **$10-15** |

### 품질 보장

1. **사용자 피드백 루프**
   - 분류 수정 시 로그 → 프롬프트 개선
   - 주간 리뷰 답변 → 질문 품질 개선

2. **가드레일**
   - 출력 형식 검증 (Zod)
   - 환각 방지: 출처 명시, 사용자 확인 필수
   - 긴 출력 금지 (검수 비용 억제)

3. **A/B 테스트**
   - 프롬프트 버전 관리
   - 분류 정확도 추적

## 결과
- Workers AI를 기본으로 사용 (비용 최소화)
- OpenAI API를 폴백 및 고품질 작업에 사용
- 임베딩 기반 관련 노트 검색
- 프롬프트 최적화로 품질 확보
- 사용자 피드백으로 지속 개선
