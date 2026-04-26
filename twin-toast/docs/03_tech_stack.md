# 03. 기술 스택

## 3.1 한눈에

| 레이어 | 선택 | 비고 |
|--------|------|------|
| 프레임워크 | **Next.js 15 (App Router)** | RSC + Edge 런타임 호환 |
| 렌더링 모드 | **Static Export (`output: 'export'`)** | Cloudflare Pages 정적 호스팅 |
| 스타일 | **Tailwind CSS v4** + `tailwind-variants` | JIT, 다크모드 `class` 전략 |
| 컴포넌트 | shadcn/ui 일부 (Button, Dialog) | 무거운 디자인 시스템 도입 X |
| 상태 | React Server Component + 클라이언트 컴포넌트의 `useReducer` | 전역 상태 라이브러리 미사용 |
| 라우팅 | App Router 파일 기반 | `/`, `/play/[contentId]`, `/about` |
| 폰트 | `next/font` + Pretendard | 가변 폰트 단일 weight 묶음 |
| 아이콘 | `lucide-react` | 트리쉐이킹 |
| 사운드 | `<audio>` 태그 + 작은 wrapper | Tone.js/Howler 미사용 (의존성 감소) |
| 테스트 | **Playwright (MCP)** | E2E 위주, 단위는 Vitest |
| 린트/포맷 | ESLint + Prettier + `@typescript-eslint` | strict |
| 패키지 매니저 | **pnpm** | 모노레포 가능성 대비 |
| 노드 버전 | **Node 20 LTS** | `.nvmrc` |
| 데이터 | **Supabase Postgres** + Storage(선택) | RLS, 어드민만 쓰기 |
| 콘텐츠 배포 | **Cloudflare R2 + Pages CDN** | 이미지 + JSON manifest |
| 호스팅 | **Cloudflare Pages** | Edge Functions 미사용 (정적) |
| CI/CD | **GitHub Actions** → Cloudflare Pages 직배포 | preview deploy 자동 |
| 모니터링 | Cloudflare Web Analytics (쿠키리스) + Sentry(선택) | |

## 3.2 왜 이 조합인가

### Next.js Static Export

서버 런타임이 필요 없다. 모든 콘텐츠는 빌드 타임에 fetch되어 정적 HTML로 굳고, 동적 부분(시계, 클릭 처리)만 클라이언트에서 돈다. Cloudflare Pages는 정적 호스팅에 최적화되어 있고, 비용도 0에 가깝다. SSR/ISR이 필요해지면 Cloudflare Pages Functions(개별 라우트)로 부분 도입 가능.

> **참고**: Next.js의 일부 기능(`next/image` 최적화, Middleware 등)은 정적 export에서 제한적이다. 이미지는 Cloudflare Image Resizing 또는 R2 + 사전 리사이즈로 대응한다.

### Tailwind CSS

클래스 명명에 시간을 안 쓰고, 다크모드 토글이 무료. 게임 한정 디자인이라 디자인 토큰 시스템을 정교하게 짤 동인이 약하다. `tailwind.config.ts`에 색/스페이싱 토큰 5~10개만 추가.

### Supabase

- 어드민 인증을 Supabase Auth로 (어드민 1~3명 대상, GitHub OAuth)
- 콘텐츠 메타데이터 테이블 (`contents`, `differences`, `reports`)
- Storage는 사용 안 함 — 이미지는 R2가 더 싸고 캐시 정책 명확
- 게임 클라이언트는 Supabase에 **직접 접근하지 않는다**. 빌드 타임에만 쿼리해서 JSON 만든다.

### Cloudflare

- Pages: 정적 호스팅, GitHub PR마다 preview URL
- R2: 이미지 + manifest JSON 저장, S3 호환, egress free
- Workers: 어드민 백엔드(이미지 생성 트리거 등)에 한정해 고려. MVP는 미사용.
- Web Analytics: 쿠키리스, 무료, GDPR 친화

## 3.3 폴더 구조 (예고편)

자세한 구조는 `13_appendix.md`. 핵심만:

```
/app             # Next.js App Router
/components      # 게임 UI
/lib             # 유틸 (좌표 검증, 셔플, 스코어)
/content         # 빌드 타임에 만들어지는 JSON manifest (gitignore)
/scripts         # 콘텐츠 빌드, 이미지 생성 파이프라인
/public          # 정적 자산 (sfx, favicon)
/docs            # 본 기획서
/tests           # Playwright 시나리오
```

## 3.4 의존성 가드레일

- `dependencies`는 **20개 이하**를 목표로 한다.
- 게임 로직 자체는 50KB JS 미만. 분석 도구로 측정.
- 라이브러리 추가 시: 같은 일을 하는 더 작은 대안이 있는지, 트리쉐이킹이 가능한지 PR 체크리스트로 확인.

## 3.5 브라우저 지원

- 모바일: iOS Safari 16+, Chrome Android 120+
- 데스크톱: Chrome 120+, Firefox 120+, Safari 17+, Edge 120+
- IE/Legacy: 지원 안 함
