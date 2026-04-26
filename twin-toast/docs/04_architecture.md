# 04. 시스템 아키텍처

## 4.1 컴포넌트 다이어그램

```mermaid
flowchart TB
    subgraph Author[콘텐츠 제작 환경]
        SS[Google Sheets<br/>콘텐츠 SoT]
        REPL[Replicate API<br/>z-image-turbo]
        GPTI[GPT-Image 2.0<br/>edit endpoint]
        QA[휴먼 QA<br/>로컬 리뷰 UI]
    end

    subgraph Build[빌드 파이프라인]
        SCR[scripts/build-content.ts]
        NEXT[Next.js Build]
    end

    subgraph Backend[운영 백엔드]
        SB[(Supabase Postgres)]
        ADMIN[Admin Web<br/>/admin Next.js]
    end

    subgraph CDN[Cloudflare]
        R2[R2 Bucket<br/>images + manifest]
        PAGES[Cloudflare Pages<br/>game.example.com]
        ANL[Web Analytics]
    end

    subgraph Client[유저 브라우저]
        GAME[Game UI]
    end

    SS --> SCR
    SB --> SCR
    REPL --> QA
    GPTI --> QA
    QA --> SB
    QA --> R2
    SCR --> NEXT
    NEXT --> PAGES
    R2 --> PAGES
    PAGES --> GAME
    GAME -.쿠키리스 핑.-> ANL
    ADMIN --> SB
    ADMIN --> REPL
    ADMIN --> GPTI
    ADMIN --> R2
```

## 4.2 데이터 흐름

### 4.2.1 콘텐츠 추가 (오프라인)

```mermaid
sequenceDiagram
    participant Admin as 어드민
    participant Sheet as Google Sheet
    participant AdminApp as /admin
    participant Repl as Replicate
    participant GPT as GPT-Image 2
    participant R2 as R2 Bucket
    participant DB as Supabase

    Admin->>Sheet: 1. 행 추가 (id, prompt, diff_count, ...)
    Admin->>AdminApp: 2. "이미지 생성" 클릭
    AdminApp->>Repl: 3. z-image-turbo 호출 (1024x1024)
    Repl-->>AdminApp: base.png
    AdminApp->>GPT: 4. edit (base + diff prompts)
    GPT-->>AdminApp: variant.png
    AdminApp->>Admin: 5. 두 이미지 + 차이 마킹 UI
    Admin->>AdminApp: 6. 차이 좌표 N개 클릭으로 정의
    AdminApp->>R2: 7. base/variant 업로드 (immutable URL)
    AdminApp->>DB: 8. contents + differences row 저장
    Admin->>AdminApp: 9. "게시" 토글
```

### 4.2.2 빌드 & 배포

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as GitHub Actions
    participant DB as Supabase
    participant R2 as R2
    participant CFP as Cloudflare Pages

    Dev->>GH: push to main
    GH->>CI: workflow_dispatch
    CI->>DB: SELECT published contents
    CI->>CI: scripts/build-content.ts<br/>→ /content/manifest.json
    CI->>CI: next build (static export)
    CI->>R2: rclone sync /content/* (manifest)
    CI->>CFP: deploy /out
    CFP-->>CI: preview/prod URL
```

> **메모**: manifest.json은 R2에도 올리지만 1차로는 Pages 정적 자산으로 함께 배포한다 (CDN HIT 빠름). R2는 어드민이 *빌드 없이* 콘텐츠를 push할 때를 위한 fallback/Remote Config 채널.

### 4.2.3 런타임 (게임 실행)

```mermaid
sequenceDiagram
    participant U as User
    participant CFP as Cloudflare Pages
    participant R2 as R2 (manifest)

    U->>CFP: GET /
    CFP-->>U: HTML + JS (정적)
    U->>CFP: GET /content/manifest.json (or R2)
    CFP-->>U: 콘텐츠 목록 (캐시 1h)
    U->>CFP: GET /play/abc123
    CFP-->>U: HTML + 클라이언트 부트스트랩
    U->>R2: GET /img/abc123/base.webp
    U->>R2: GET /img/abc123/variant.webp
    Note over U: 모든 정답 검증은 클라이언트에서<br/>좌표 비교만 수행
```

## 4.3 환경 분리

| 환경 | 도메인 | Supabase | R2 버킷 | 용도 |
|------|--------|----------|---------|------|
| local | localhost:3000 | local Supabase or staging | `twin-toast-dev` | 개발 |
| preview | `*.pages.dev` | staging | `twin-toast-dev` | PR preview |
| production | `twintoast.app` | production | `twin-toast-prod` | 출시 |

`.env.local` 키:

```
NEXT_PUBLIC_R2_BASE_URL=https://cdn.twintoast.app
NEXT_PUBLIC_MANIFEST_URL=https://cdn.twintoast.app/manifest/v1.json
SUPABASE_URL=...                # 서버/빌드 전용
SUPABASE_SERVICE_ROLE_KEY=...   # 서버/빌드 전용 (CI secret)
REPLICATE_API_TOKEN=...         # 어드민 전용
OPENAI_API_KEY=...              # 어드민 전용 (GPT-Image)
```

게임 클라이언트는 `NEXT_PUBLIC_*`만 사용한다. 다른 키가 클라이언트에 노출되면 빌드 실패하도록 ESLint custom rule을 둔다.

## 4.4 캐시 전략

| 자원 | 캐시 정책 | 무효화 |
|------|-----------|--------|
| HTML | `cache-control: public, max-age=0, must-revalidate` | 배포 시 즉시 |
| JS/CSS (해시 파일명) | `immutable, max-age=31536000` | 파일명 변경 |
| 이미지 (R2, 해시 경로) | `immutable, max-age=31536000` | 경로 변경 |
| manifest.json | `public, max-age=300, s-maxage=3600` | 의도적으로 짧게 — Remote Config 효과 |
| sfx | `immutable, max-age=31536000` | |

manifest.json만 짧게 캐시하는 이유: 어드민이 새 콘텐츠를 게시하면 *빌드 없이* CDN purge로 반영 가능. 자세한 전략은 `07_content_ops.md`.
