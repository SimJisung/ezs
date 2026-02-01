# ADR-003: 백엔드 기술 스택

## 상태
확정

## 맥락
Cloudflare Workers 환경에서 효율적으로 동작하면서, 빠른 개발과 유지보수가 가능한 백엔드 스택을 선택해야 합니다.

### 고려한 대안

#### Workers 프레임워크

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Hono** | 경량(12KB), Workers 최적화, Express 유사 API, 미들웨어 풍부 | 상대적으로 신규 |
| itty-router | 초경량(< 1KB) | 기능 제한, 미들웨어 부족 |
| Cloudflare Workers 직접 | 의존성 없음 | 보일러플레이트 많음 |

#### 검증 라이브러리

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Zod** | TypeScript 친화, 런타임 검증, 스키마 재사용 | 번들 크기 |
| Yup | 널리 사용됨 | TypeScript 지원 약함 |
| AJV | 빠름, JSON Schema | 사용성 복잡 |

## 결정

### 선택된 스택

| 구분 | 선택 | 근거 |
|------|------|------|
| Runtime | **Cloudflare Workers** | ADR-001 결정 |
| Framework | **Hono** | Workers 최적화, 경량, 좋은 DX |
| Language | **TypeScript** | 프론트엔드와 타입 공유 |
| Validation | **Zod** | 타입 추론, 프론트/백엔드 공유 |
| ORM | **Drizzle ORM** (선택) | D1 지원, 타입 안전, 경량 |
| Auth | **JWT** | Stateless, 표준 |

### 핵심 결정 근거

1. **Hono 프레임워크**
   ```typescript
   // Express와 유사한 직관적인 API
   import { Hono } from 'hono';
   import { cors } from 'hono/cors';
   import { jwt } from 'hono/jwt';

   const app = new Hono();

   app.use('/api/*', cors());
   app.use('/api/*', jwt({ secret: env.JWT_SECRET }));

   app.get('/api/notes', async (c) => {
     const notes = await c.env.DB.prepare('SELECT * FROM notes').all();
     return c.json(notes);
   });
   ```

2. **Zod 스키마 공유**
   ```typescript
   // packages/shared/validators/note.ts
   import { z } from 'zod';

   export const createNoteSchema = z.object({
     content: z.string().min(1).max(10000),
     type: z.enum(['idea', 'decision', 'todo', 'learning', 'journal']).optional(),
   });

   export type CreateNoteInput = z.infer<typeof createNoteSchema>;
   ```

3. **D1 직접 사용 vs ORM**
   - MVP에서는 D1 직접 쿼리 사용 (간단함)
   - 쿼리 복잡해지면 Drizzle ORM 도입 검토
   ```typescript
   // 직접 사용
   const note = await c.env.DB
     .prepare('SELECT * FROM notes WHERE id = ?')
     .bind(id)
     .first();

   // Drizzle ORM (필요시)
   const note = await db.select().from(notes).where(eq(notes.id, id));
   ```

4. **JWT 인증**
   - Cloudflare Access 없이 자체 인증
   - Stateless로 Edge 환경에 적합
   - KV에 리프레시 토큰 저장

### 아키텍처 패턴

```
src/
├── routes/           # 라우트 핸들러 (Controller 역할)
├── services/         # 비즈니스 로직
├── repositories/     # 데이터 접근 추상화
├── middleware/       # 공통 미들웨어
├── lib/              # 유틸리티
└── types/            # 타입 정의
```

- **Routes**: HTTP 요청/응답 처리만
- **Services**: 비즈니스 로직 집중
- **Repositories**: DB/Storage 접근 추상화

## 결과
- Hono 기반 Workers API
- Zod로 입력 검증 및 타입 공유
- JWT 기반 Stateless 인증
- 레이어드 아키텍처로 관심사 분리
