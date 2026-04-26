# Twin Toast — 다른 그림 찾기 (Spot the Difference)

> "두 장의 토스트, 하나는 살짝 탔다." — 모던 웹 기반 캐주얼 게임

이 폴더는 Twin Toast 프로젝트의 기획 문서 모음입니다. 문서는 작은 단위로 쪼개져 있으며, 번호 순서대로 읽으면 기획의 전체 흐름을 따라갈 수 있도록 구성했습니다.

## 문서 인덱스

| # | 문서 | 내용 |
|---|------|------|
| 00 | [overview](00_overview.md) | 본 문서. 전체 인덱스 및 한 줄 요약 |
| 01 | [product_spec](01_product_spec.md) | 제품 비전, 타겟 유저, MVP 스코프, 성공 지표 |
| 02 | [game_design](02_game_design.md) | 게임 룰, UX 플로우, 화면 설계, 상호작용 |
| 03 | [tech_stack](03_tech_stack.md) | 기술 스택 선정 근거, 의존성, 라이브러리 |
| 04 | [architecture](04_architecture.md) | 시스템 아키텍처, 데이터 흐름, mermaid 다이어그램 |
| 05 | [data_model](05_data_model.md) | Supabase 스키마, JSON 구조, 캐시 전략 |
| 06 | [image_pipeline](06_image_pipeline.md) | Replicate + GPT-Image 이미지 생성 파이프라인 |
| 07 | [content_ops](07_content_ops.md) | 스프레드시트 → JSON → CDN 운영, Remote Config |
| 08 | [admin_tools](08_admin_tools.md) | 어드민 툴 옵션 비교, 추천 구성 |
| 09 | [deployment](09_deployment.md) | Cloudflare Pages/Workers/R2, 환경 분리, 도메인 |
| 10 | [testing](10_testing.md) | Playwright MCP 기반 E2E, 단위/시각 회귀 테스트 |
| 11 | [maintenance](11_maintenance.md) | 운영, 모니터링, 콘텐츠 추가, 신고 처리, 비용 관리 |
| 12 | [roadmap](12_roadmap.md) | MVP → 베타 → 정식 출시 마일스톤, 작업 단위 |
| 13 | [appendix](13_appendix.md) | 디렉토리 구조, 환경 변수, 명명 규칙, 용어집 |

## 한 줄 요약

> Replicate `z-image-turbo`로 베이스 그림을 만들고, GPT-Image 2로 "다른 곳"을 편집한 두 장을 비교해 정해진 시간 내 N개의 차이를 찾는 미니 게임. 로그인 없음, 사용자 데이터 미저장. Next.js + Tailwind로 모던한 룩, Supabase에 콘텐츠 메타만 보관, Cloudflare로 정적 호스팅 + CDN.

## 설계 원칙

1. **MVP 최소주의** — 핵심 루프(고른다 → 푼다 → 결과)만 잘 만든다.
2. **콘텐츠는 정적, 게임 상태는 클라이언트** — 서버는 메타데이터만, 진행 상태는 메모리.
3. **JSON-as-CMS** — 구글 스프레드시트를 단일 진실 소스(SoT)로 두고 빌드/배포 시 JSON으로 굳혀 CDN 캐시.
4. **이미지 생성은 오프라인 파이프라인** — 런타임에 LLM/이미지 모델을 호출하지 않는다. 비용·지연·실패 위험 회피.
5. **사용자 식별 0** — 로그인, 쿠키, 트래킹 없음. 차이 좌표는 클라이언트에서 검증.
6. **유지보수 우선** — 콘텐츠 1세트 추가가 *5분 내*에 끝나는 워크플로우를 목표로 한다.
