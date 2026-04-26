# 08. 어드민 툴

## 8.1 옵션 비교

| 옵션 | 장점 | 단점 | 적합도 |
|------|------|------|--------|
| **자체 어드민 페이지 (Next.js `/admin`)** | 코드/스키마와 같은 레포, 차이 좌표 마킹 같은 *커스텀 UI* 자유, 인증을 Supabase Auth로 간단히 | 초기 구현 1~2일 | ★★★★★ |
| Supabase Studio (대시보드) | 즉시 사용, 무료 | 차이 좌표 마킹 같은 그래픽 작업 불가, 이미지 생성 트리거 불가 | ★★ |
| Retool / Appsmith | 빠른 폼 빌더, RBAC | 비용, 좌표 캔버스가 어색, 외부 SaaS 의존 | ★★ |
| Strapi / Directus 등 헤드리스 CMS | 표준 CMS UX | 이미지 생성 워크플로우, 좌표 마킹은 결국 커스텀 필요 → 이중 작업 | ★ |
| Google Sheets만 + 빌드 스크립트 | 0 인프라 | 좌표/이미지 워크플로우 불가능 | ★ (메타 SoT용으로 보조 사용) |

**추천**: **자체 `/admin` 페이지 + Supabase Auth + Google Sheets는 메타 데이터 SoT로 보조**.

## 8.2 자체 어드민 구성

### 라우트

```
/admin                # 대시보드 (콘텐츠 목록, 통계)
/admin/login          # Supabase Auth (Email magic link 또는 GitHub OAuth)
/admin/contents       # 콘텐츠 목록 (필터: status)
/admin/contents/new   # 신규 생성 (이미지 파이프라인 마법사)
/admin/contents/[id]  # 상세 / 편집 / 좌표 마킹
/admin/reports        # 신고 큐
/admin/jobs           # 이미지 생성 잡 모니터링
/admin/config         # Remote Config 편집
```

### 인증

- Supabase Auth (이메일 매직링크)
- 어드민 화이트리스트는 `admin_users` 테이블 또는 Auth 메타데이터(`role=admin`)로 관리
- 미들웨어에서 매 요청마다 검증 (Edge Function 또는 Pages Function)
- MFA는 GitHub OAuth 사용 시 OAuth 측에 위임

### 권한 모델

- `owner`: 모든 권한
- `editor`: 콘텐츠 CRUD, published 토글, 신고 처리
- 그 외 역할 없음 (필요해지면 그때)

### 핵심 화면 — 차이 좌표 마킹

```
┌────────────────────────────────────────────┐
│ [Base Image]                               │
│   - 클릭 시 새 차이 추가 (cx,cy)              │
│   - 드래그로 반경 r 조절                      │
│   - 더블클릭으로 삭제                         │
│                                            │
│ Diffs:                                     │
│   1. (0.12, 0.34, 0.06)  edit: "remove..." │
│   2. (0.55, 0.20, 0.05)  edit: "add..."    │
│   ...                                      │
│                                            │
│ [Generate Variant] → GPT-Image 호출         │
│ [Preview Side-by-Side]                     │
│ [Save]  [Publish]                          │
└────────────────────────────────────────────┘
```

- 캔버스 라이브러리는 가벼운 것을 직접 구현 (HTML5 Canvas + 단일 컴포넌트)
- 외부 의존성 추가 X (react-konva 등은 과함)

## 8.3 Remote Config

### 무엇을 Remote Config화할까

| 항목 | Remote Config | 이유 |
|------|---------------|------|
| 라운드 기본 시간 | ✓ | 난이도 튜닝을 빌드 없이 |
| 차이 hit-box 기본 반경 | ✓ | 사용자 피드백 반영 |
| 오답 페널티 토글/값 | ✓ | A/B 가능성 |
| 신고 사유 옵션 목록 | ✓ | UX 메시지 변경 |
| BGM 활성화 | ✓ | |
| 콘텐츠 노출 제외 ID 리스트 | ✓ | 긴급 차단 |
| 광고/공지 배너 텍스트 | ✓ | |
| 게임 룰 자체 (정답 수 등) | ✗ | 콘텐츠별 메타에 둠 |
| API 엔드포인트 | ✗ | 환경 변수에 둠 |

### Remote Config의 형태

`config.json`을 manifest와 같은 R2 경로에 둔다.

```json
{
  "version": "2026-04-26T12:00:00Z",
  "default_time_limit_sec": 60,
  "default_hit_radius": 0.05,
  "miss_penalty_sec": 0,
  "report_reasons": [
    { "value": "blurry", "label": "이미지가 흐릿해요" },
    { "value": "wrong_diff", "label": "차이가 잘못됐어요" },
    { "value": "inappropriate", "label": "부적절한 이미지" },
    { "value": "other", "label": "기타" }
  ],
  "blocklist_content_ids": [],
  "banner": null,
  "feature_flags": {
    "miss_shake": true,
    "preview_thumbnails": true
  }
}
```

캐시: `max-age=60, s-maxage=300, stale-while-revalidate=3600`. 더 짧게 굴려 긴급 변경에 반응성 높임.

### 클라이언트 적용

```ts
// /lib/config.ts
const DEFAULT_CONFIG = { ... } satisfies Config;
export async function loadConfig(): Promise<Config> {
  try {
    const res = await fetch(CONFIG_URL, { cache: 'no-store' });
    if (!res.ok) return DEFAULT_CONFIG;
    return { ...DEFAULT_CONFIG, ...(await res.json()) };
  } catch {
    return DEFAULT_CONFIG;
  }
}
```

- 항상 코드에 박힌 `DEFAULT_CONFIG`로 동작 가능해야 한다.
- Remote Config가 죽어도 게임은 멀쩡히 돈다.

## 8.4 어드민 도구의 운영 기능

| 기능 | 설명 |
|------|------|
| 신고 처리 큐 | 신고 카운트 desc로 정렬, 콘텐츠 미리보기, "archive/keep" 1클릭 |
| 콘텐츠 일괄 가져오기 | 스프레드시트 URL 입력 → 메타만 import |
| 이미지 재생성 | 같은 콘텐츠 ID에 대해 *새 ID*로 재생성 (immutable URL) |
| 비용 대시보드 | 이번 달 Replicate/OpenAI 호출 수, 누적 비용 추정 |
| Manifest 미리보기 | 현재 published 콘텐츠로 만들어질 manifest를 dry-run |
| 긴급 차단 | content_id를 blocklist에 추가 → config.json만 갱신 → 5분 내 반영 |
| 변경 이력 | contents 변경 audit log (Supabase Realtime + log 테이블) |

## 8.5 미니 어드민 시작 단계 (MVP에서 줄이는 법)

처음 1주 안엔 풀 어드민이 없어도 된다. 다음 순서로 만든다:

1. (반나절) 콘텐츠 "목록 + published 토글"만 — 신규 생성은 SQL로 직접
2. (반나절) Replicate/GPT-Image 트리거 + 결과 미리보기
3. (1일) 차이 좌표 마킹 캔버스
4. (반나절) 신고 큐
5. (반나절) Remote Config 편집기 (JSON textarea + validate)

이 순서로 만들면 매일 사용 가능한 상태를 유지하면서 점진적으로 풀 어드민이 된다.
