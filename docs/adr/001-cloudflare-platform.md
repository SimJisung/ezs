# ADR-001: Cloudflare 플랫폼 선택

## 상태
확정

## 맥락
MVP를 빠르게 구축하고 운영 부담을 최소화하면서, 글로벌 사용자에게 낮은 지연시간을 제공해야 합니다.

### 고려한 대안

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Cloudflare (Workers + Pages)** | Edge 배포, 서버리스, 통합 생태계, 저렴한 비용 | 일부 Node.js API 미지원, D1 성숙도 |
| AWS (Lambda + S3 + RDS) | 풍부한 서비스, 성숙한 생태계 | 복잡한 설정, 콜드 스타트, 비용 예측 어려움 |
| Vercel + Supabase | 빠른 개발, 좋은 DX | 비용 급증 가능, Edge 제한 |
| Self-hosted (VPS) | 완전한 제어 | 운영 부담, 스케일링 직접 관리 |

## 결정
**Cloudflare 플랫폼**을 선택합니다.

### 선택 이유

1. **Edge-First 아키텍처**
   - 전 세계 300+ PoP에서 코드 실행
   - 사용자와 가장 가까운 곳에서 응답 (< 50ms)

2. **통합 생태계**
   - Workers (컴퓨팅), D1 (SQL), R2 (스토리지), KV (캐시), AI (LLM)
   - 단일 CLI(wrangler)로 모든 리소스 관리

3. **비용 효율성**
   - Workers: 일 10만 요청 무료, 이후 $5/백만 요청
   - D1: 5GB 무료, $0.75/GB-월
   - R2: 10GB 무료, egress 비용 없음

4. **운영 부담 제로**
   - 서버 프로비저닝/관리 불필요
   - 자동 스케일링, DDoS 방어 기본 제공

5. **Workers AI**
   - 추가 인프라 없이 LLM 사용 가능
   - 분류/요약/임베딩을 Edge에서 처리

### 리스크 및 대응

| 리스크 | 대응 |
|--------|------|
| D1 성숙도 (GA 직후) | 중요 데이터 백업 정책, 복잡한 쿼리 회피 |
| Workers AI 모델 제한 | OpenAI API 폴백 옵션 |
| Vendor lock-in | 표준 SQL/S3 API 사용, 추상화 레이어 |

## 결과
- Cloudflare 생태계 기반으로 MVP 개발
- Hono 프레임워크로 Workers 최적화된 API 구축
- 필요시 OpenAI API 폴백 준비
