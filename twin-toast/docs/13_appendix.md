# 13. 부록

## 13.1 디렉토리 구조

```
twin-toast/
├─ app/
│  ├─ layout.tsx                # 글로벌 레이아웃 (폰트, 다크모드)
│  ├─ page.tsx                  # 홈
│  ├─ play/
│  │  ├─ page.tsx               # 랜덤 진입
│  │  └─ [contentId]/
│  │     └─ page.tsx            # 라운드 (CSR)
│  ├─ about/
│  │  └─ page.tsx
│  └─ admin/
│     ├─ layout.tsx             # 인증 가드
│     ├─ page.tsx               # 대시보드
│     ├─ contents/
│     ├─ reports/
│     └─ config/
├─ components/
│  ├─ Game/
│  │  ├─ GameBoard.tsx          # 좌우 이미지 + overlay
│  │  ├─ Timer.tsx
│  │  ├─ DifferenceMarker.tsx
│  │  └─ ResultDialog.tsx
│  ├─ Admin/
│  │  ├─ DiffMarkerCanvas.tsx
│  │  └─ ContentEditor.tsx
│  └─ ui/                       # shadcn/ui
├─ lib/
│  ├─ manifest.ts               # 로드/검증
│  ├─ config.ts                 # remote config
│  ├─ hit-detection.ts          # 좌표 → diff idx
│  ├─ score.ts
│  ├─ shuffle.ts
│  ├─ supabase.ts               # 서버 전용
│  └─ replicate.ts              # 서버 전용
├─ scripts/
│  ├─ build-content.ts          # DB → manifest
│  ├─ validate-content.ts       # 무결성 검사
│  └─ sync-r2.ts                # 이미지/manifest 동기화
├─ content/
│  └─ manifest.json             # 빌드 산출물 (gitignore)
├─ public/
│  ├─ sfx/                      # 효과음
│  └─ favicon.ico
├─ tests/
│  ├─ unit/                     # Vitest
│  ├─ component/                # Vitest + RTL
│  └─ e2e/                      # Playwright
├─ supabase/
│  └─ migrations/
├─ docs/                        # ← 본 기획서들
├─ .github/
│  └─ workflows/
│     ├─ preview.yml
│     ├─ production.yml
│     └─ scheduled-build.yml
├─ next.config.mjs              # output: 'export'
├─ tailwind.config.ts
├─ tsconfig.json
├─ package.json
└─ pnpm-lock.yaml
```

## 13.2 환경 변수

`/.env.example` (커밋), `.env.local` (개인 머신에만)

```
# Public (클라이언트 노출 OK)
NEXT_PUBLIC_R2_BASE_URL=https://cdn.twintoast.app
NEXT_PUBLIC_MANIFEST_URL=https://cdn.twintoast.app/manifest/v1.json
NEXT_PUBLIC_CONFIG_URL=https://cdn.twintoast.app/config.json
NEXT_PUBLIC_CF_ANALYTICS_TOKEN=

# Server only
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_ANON_KEY=             # 어드민 클라이언트용
REPLICATE_API_TOKEN=
OPENAI_API_KEY=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_PROD=twin-toast-prod
R2_BUCKET_DEV=twin-toast-dev
```

## 13.3 명명 규칙

- 디렉토리: kebab-case
- React 컴포넌트 파일: PascalCase
- 그 외 TS 파일: kebab-case
- DB 테이블/컬럼: snake_case
- 상수: SCREAMING_SNAKE_CASE
- React hook: `useFooBar`
- 테스트 파일: `*.test.ts` (단위), `*.spec.ts` (E2E)

## 13.4 Git 전략

- 단일 main 브랜치
- 기능 브랜치는 `feat/`, `fix/`, `docs/`, `chore/` 접두사
- PR 머지는 squash merge
- 커밋 메시지는 한국어 가능. 단 첫 줄은 50자 이내, 동사 시작

## 13.5 기본 색상 토큰 (Tailwind)

```ts
// tailwind.config.ts theme.extend.colors
{
  toast: {
    50:  '#FFF8EC',
    100: '#FFE9C2',
    400: '#F2A65A',
    600: '#C97A2B',
    900: '#5A2A06',
  },
  ink: {
    50:  '#F7F7F8',
    900: '#0E0F12',
  },
  hit: {
    success: '#22C55E',
    miss:    '#EF4444',
  }
}
```

다크모드 토큰은 동일 색의 다른 명도. `dark:` 변형으로 적용.

## 13.6 키보드 단축키 (선택)

| 키 | 동작 |
|----|------|
| Space | 일시정지 / 재개 (MVP+) |
| R | 라운드 재시작 (결과 화면에서) |
| N | 다음 라운드 (결과 화면에서) |
| Esc | 홈으로 |

## 13.7 용어집

- **콘텐츠 (content)**: 한 라운드를 구성하는 단위. 베이스 이미지 + 변형 이미지 + 차이 좌표 N개의 묶음.
- **차이 (difference, diff)**: 두 이미지 사이의 시각적 차이 1개. `(cx, cy, r)` 정의.
- **manifest**: 게시된 콘텐츠 목록을 담은 JSON. CDN으로 배포되는 게임의 데이터 부분.
- **Remote Config**: manifest와 별도로, 게임의 *설정* 값(시간, 페널티 등)을 담은 JSON. 빌드 없이 변경 가능.
- **베이스 이미지**: Replicate로 생성한 원본.
- **변형 이미지**: GPT-Image로 베이스를 편집한 결과.
- **세트**: 콘텐츠 1개를 부르는 비공식 용어 (= 1 라운드 분량).

## 13.8 참고 링크 (개발자가 직접 확인)

- Next.js Static Export 가이드
- Cloudflare Pages 빌드 환경
- Cloudflare R2 + Custom Domain
- Replicate Node SDK
- OpenAI Images Edit API
- Supabase Auth + RLS
- Playwright MCP

## 13.9 문서 유지 지침

- 본 docs는 *살아있는 문서*. 의사결정이 바뀌면 즉시 수정.
- 큰 변경은 문서 상단에 "Last updated: YYYY-MM-DD" 코멘트 추가.
- 새 문서 추가 시 `00_overview.md`의 인덱스 갱신.
- mermaid는 GitHub에서 바로 렌더되므로 적극 활용.
- 분량이 늘면 한 파일을 더 쪼개되, 인덱스에서는 한 줄 요약을 유지.
