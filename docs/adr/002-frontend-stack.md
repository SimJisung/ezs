# ADR-002: 프론트엔드 기술 스택

## 상태
확정

## 맥락
MVP를 빠르게 구축할 수 있으면서, 향후 확장성과 유지보수성이 좋은 프론트엔드 스택을 선택해야 합니다.

### 고려한 대안

#### 프레임워크

| 옵션 | 장점 | 단점 |
|------|------|------|
| **React 18** | 풍부한 생태계, 팀 친숙도, 안정성 | 보일러플레이트, SSR 복잡 |
| Vue 3 | 쉬운 학습곡선, 좋은 DX | 생태계 상대적 작음 |
| Svelte | 번들 크기 작음, 성능 | 생태계 작음, 채용 어려움 |
| Solid | React 유사, 최고 성능 | 신규, 생태계 미성숙 |

#### 빌드 도구

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Vite** | 빠른 HMR, ESBuild, 간단한 설정 | - |
| Create React App | 공식 도구 | 느림, 설정 제한, deprecated 방향 |
| Next.js | SSR/SSG, 풀스택 | Cloudflare Pages 호환 제한 |

#### 상태 관리

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Zustand** | 경량, 간단, 보일러플레이트 없음 | - |
| Redux Toolkit | 강력, 표준화 | 보일러플레이트 많음 |
| Jotai/Recoil | Atomic, React 친화 | 복잡한 상태에서 관리 어려움 |

## 결정

### 선택된 스택

| 구분 | 선택 | 근거 |
|------|------|------|
| Framework | **React 18** | 팀 친숙도, 생태계, 채용 |
| Build | **Vite** | 빠른 개발 경험, Cloudflare Pages 호환 |
| Language | **TypeScript** | 타입 안정성, 백엔드와 타입 공유 |
| State | **Zustand** | 경량, 간단한 API |
| Server State | **TanStack Query** | 캐싱, 재시도, 낙관적 업데이트 |
| Styling | **Tailwind CSS** | 빠른 개발, 일관된 디자인 시스템 |
| UI Components | **Radix UI** | 접근성, 헤드리스 (스타일 자유도) |
| Form | **React Hook Form + Zod** | 성능, 타입 안전한 검증 |
| Router | **React Router v7** | 표준, 안정성 |

### 핵심 결정 근거

1. **React + Vite**
   - Cloudflare Pages와 완벽 호환
   - 빠른 HMR로 개발 생산성 향상
   - SPA로 초기 로딩 후 빠른 내비게이션

2. **Zustand + TanStack Query**
   - 클라이언트 상태(UI)와 서버 상태(API) 명확 분리
   - TanStack Query의 캐싱으로 불필요한 요청 최소화
   - Zustand의 간단한 API로 보일러플레이트 제거

3. **Tailwind CSS + Radix UI**
   - Tailwind로 빠른 스타일링
   - Radix UI로 접근성 확보하면서 커스텀 디자인 적용
   - 디자인 시스템 구축 비용 절감

4. **TypeScript 전역 사용**
   - 프론트엔드/백엔드 타입 공유 (packages/shared)
   - 런타임 에러 사전 방지
   - IDE 자동완성으로 개발 속도 향상

## 결과
- React 18 + Vite + TypeScript 기반 SPA
- Cloudflare Pages에 정적 배포
- 서버 상태는 TanStack Query로 관리
- Tailwind + Radix UI로 일관된 UI
