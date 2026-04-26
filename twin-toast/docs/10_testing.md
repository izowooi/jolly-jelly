# 10. 테스트 전략

## 10.1 테스트 피라미드

```
         E2E (Playwright MCP)        ←  10
       ────────────────────────
       시각 회귀 (Playwright)         ←  20
     ──────────────────────────
   컴포넌트 (RTL + Vitest)            ←  30
 ────────────────────────────────
 단위 (Vitest pure)                   ← 100+
```

게임 핵심은 *좌표 검증* 같은 순수 함수 → 단위 테스트가 매우 유리. 비주얼 회귀는 모바일/데스크톱 두 뷰포트만.

## 10.2 단위 테스트 (Vitest)

대상:

- `lib/hit-detection.ts` — 클릭 좌표 → 차이 ID 매핑
- `lib/score.ts` — 시간/콤보 → 점수
- `lib/shuffle.ts` — 결정적 셔플 시드
- `lib/manifest.ts` — schema 검증, fallback
- `lib/coords.ts` — px ↔ normalized

샘플:

```ts
import { describe, it, expect } from 'vitest';
import { detectHit } from '@/lib/hit-detection';

describe('detectHit', () => {
  const diffs = [
    { idx: 1, cx: 0.5, cy: 0.5, r: 0.05 },
    { idx: 2, cx: 0.8, cy: 0.2, r: 0.05 },
  ];
  it('정답 영역 안 클릭은 idx 반환', () => {
    expect(detectHit({ x: 0.5, y: 0.5 }, diffs)).toBe(1);
    expect(detectHit({ x: 0.81, y: 0.21 }, diffs)).toBe(2);
  });
  it('영역 밖은 null', () => {
    expect(detectHit({ x: 0.0, y: 0.0 }, diffs)).toBeNull();
  });
  it('이미 발견된 idx는 skip 옵션', () => {
    expect(detectHit({ x: 0.5, y: 0.5 }, diffs, { skip: [1] })).toBeNull();
  });
});
```

## 10.3 컴포넌트 테스트

- React Testing Library + Vitest
- 대상: `<GameBoard>`, `<Timer>`, `<ResultDialog>`, `<DifferenceMarker>`
- 시간 제어: `vi.useFakeTimers()`
- 사용자 입력: `@testing-library/user-event`

## 10.4 Playwright (MCP)

테스트는 `tests/e2e/*.spec.ts`로 구성. Playwright MCP를 통해 어시스턴트가 직접 실행/디버그 가능.

### 시나리오

1. **smoke**: 홈 진입 → "랜덤 시작" → 첫 콘텐츠 로드 확인
2. **happy-path**: 시드된 스텁 manifest로 정답 5회 클릭 → 결과 화면 "성공" 확인
3. **timeout**: 시계 빠르게 돌려 60초 경과 → "실패" 화면
4. **miss**: 빈 영역 클릭 → 흔들림 효과 + 발견 카운트 변동 없음
5. **reload**: 라운드 중 새로고침 → 사용자 데이터 0이므로 홈으로 (의도된 동작)
6. **manifest fallback**: manifest URL을 가짜 404로 → 백업 manifest 동작
7. **schema mismatch**: schema=999인 manifest → 안전 모드 메시지
8. **mobile viewport**: 375x812, 두 이미지 세로 스택 확인
9. **a11y smoke**: `prefers-reduced-motion` 시 흔들림 0
10. **report**: 신고 버튼 → 사유 선택 → POST 성공 (네트워크 인터셉트)

### 좌표 시뮬레이션

- 테스트에서는 manifest를 인터셉트(`page.route`)해 정해진 좌표를 가진 콘텐츠를 주입
- `page.locator('[data-testid="image-base"]').click({ position: { x: 512, y: 512 } })`로 정답 위치 클릭

### 시각 회귀

- `await expect(page).toHaveScreenshot('home.png')` 형식
- 뷰포트: 375x812 (iPhone), 1280x800 (데스크톱)
- 마스킹: 시계 영역은 `mask` 옵션으로 비교 제외

## 10.5 어드민 테스트

- 인증이 필요하므로 Playwright `storageState`에 stub Supabase JWT
- 핵심 플로우: 로그인 → 콘텐츠 생성(이미지 API는 mock) → 좌표 5개 마킹 → published 토글
- Replicate/OpenAI는 절대 진짜로 호출하지 않는다. `msw`로 인터셉트.

## 10.6 성능 테스트

- Lighthouse CI: PR마다 모바일/데스크톱 점수 기록, 회귀 시 fail
- bundlesize CI: 게임 페이지 JS gzipped 80KB 상한
- WebPageTest 1회/주 (수동 트리거 또는 cron)

## 10.7 데이터 무결성 테스트 (콘텐츠 빌드)

`scripts/validate-content.ts`:

- 모든 published 콘텐츠에 대해:
  - base/variant URL HEAD 200
  - differences row 수 == diff_count
  - 좌표가 [0,1]
  - hit-box 두 개가 겹치지 않음 (사용자 혼란 방지)
- CI에서 빌드 전 실행. 실패하면 배포 중단.

## 10.8 수동 QA 체크리스트 (콘텐츠 추가 시)

- [ ] 좌우 이미지 사이즈/비율 동일
- [ ] 차이 5개 모두 사람이 *어느 정도* 알아볼 수 있음 (너무 어렵지 않음)
- [ ] 5개 외에 의도하지 않은 차이가 없음 (편집 모델이 다른 영역도 건드리는 경우)
- [ ] 차이 영역들이 너무 한 모서리에 몰려있지 않음
- [ ] 부적절한 요소 없음
- [ ] 모바일/데스크톱 둘 다에서 플레이 가능
