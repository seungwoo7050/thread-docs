# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 검토 범위와 선별 원칙

이 인덱스는 현재 GPT 프로젝트에 축적된 Thread 01~16 학습 문서와 확인 가능한 커밋 작업 기록만을 근거로 작성했다. 기록에서 확인되지 않은 함수명·파일명·구현 세부사항은 넣지 않았다.

선별 기준은 다음과 같다.

- **S**: 질문과 10~30분 직접 구현 모두 준비해야 하는 핵심 경계
- **A**: 질문 가치가 높고, 축소 구현이나 테스트 helper 구현 가능성이 높은 지점
- **B**: 별도 코딩 문제보다는 설계 판단·운영 trade-off 설명이 중요한 지점
- **C**: 반복 UI, boilerplate, 스타일 조립, 단순 설정처럼 독립 면접 항목으로 만들 가치가 낮은 지점

같은 역량이 여러 Thread에서 반복되면 대표 Thread를 정하고 상세 워크북 한 항목으로 통합했다. 상세 문서의 P01~P17이 S/A 완전성 검증의 기준이다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---:|---|---|---|---|---|---|---|
| A | 01·13 | `refactor(content): 검증된 콘텐츠를 portfolio facade에 연결`<br>`test(portfolio): selector와 presentation 회귀 계약 보강` | `src/lib/portfolio/content.ts`<br>`src/lib/portfolio/selectors.ts`<br>`src/lib/portfolio.ts`<br>`src/lib/portfolio.test.ts`<br>`getPortfolioContent` | Canonical facade, deterministic selector, 선택적 복제와 aliasing | 호출 간 mutation 격리와 structural sharing의 경계를 설명·구현할 가치가 높다. 상세: [P07](02-domain-routing-and-renderers.md#p07) | 높음 | 중간 | 02, 06, 13 |
| S | 02 | `feat(content): JSON schema 파싱 경계 추가` | `src/lib/content-schema.ts`<br>`src/lib/content-loader.ts`<br>`src/lib/portfolio/types.ts`<br>`parseContentFile`<br>`loadPortfolioSource` | 런타임 입력 검증과 정적 타입의 단일 근원 | TypeScript 타입 소거, `unknown` 경계, 스키마 출력 타입 추론을 한 문제로 확인할 수 있다. 상세: [P01](01-content-integrity-and-security.md#p01) | 높음 | 높음 | 01, 03, 04 |
| B | 02 | `refactor(content): 프로젝트 컬렉션 migration 경계 추가` | `src/lib/content-schema.ts` | 이전·신규 형식의 migration union | 전환기 호환성의 종료 조건과 모호성 설명은 중요하지만 독립 구현 문제로는 P01과 중복된다. | 높음 | 중간 | 01, 03 |
| S | 03 | `feat(content): 콘텐츠 validation 오류 모델 추가`<br>`feat(content): 중복과 참조 진단 helper 추가` | `src/lib/content-loader.ts`<br>`PortfolioContentError`<br>`ContentValidationIssue`<br>`jsonPath`<br>`findDuplicates`<br>`addMissingReferenceIssue` | 중복·외래 참조 invariant와 누적 진단 | 다중 문서 정합성, O(N+R) 색인, fail-fast와 누적 오류의 trade-off를 직접 구현하기 좋다. 상세: [P02](01-content-integrity-and-security.md#p02) | 높음 | 높음 | 02, 04, 06, 13 |
| S | 03·07 | `feat(content): 내부 route 참조 검증 추가`<br>`feat(navigation): 템플릿 URL과 쿼리 해석 추가` | `src/lib/content-loader.ts`<br>`src/lib/portfolio/template-href.ts`<br>`src/lib/portfolio/selectors.ts`<br>`addInternalRouteIssue`<br>`createTemplateHref`<br>`resolveHomeTemplateId` | 내부·외부 URL 분류, query/hash 보존, route state canonicalization | 문자열 연결 버그, default 상태 생략, 동적 route 검증을 하나의 일반화된 URL 문제로 묶을 수 있다. 상세: [P03](01-content-integrity-and-security.md#p03) | 높음 | 높음 | 05, 08, 13 |
| S | 03 | `feat(content): 저장소 자산 참조 경계 검증` | `src/lib/content-assets.ts`<br>`collectAssetReferences`<br>`validatePortfolioAssets` | 공개 루트 containment와 path traversal 방지 | 파일 존재 여부를 넘어 경로 정규화·탈출 방지·오류 구분을 확인하는 보안 구현 문제다. 상세: [P04](01-content-integrity-and-security.md#p04) | 높음 | 높음 | 04, 15 |
| B | 03 | `feat(routes): 비활성 페이지 route 차단` | `src/app/projects/page.tsx`<br>`src/app/projects/[projectId]/page.tsx`<br>`src/app/contact/page.tsx`<br>`src/app/resume/page.tsx`<br>`isSitePageEnabled` | 콘텐츠 설정과 실제 route 가시성 일치 | 404 정책과 metadata·navigation 일관성 설명은 중요하지만 구현은 반복적인 route guard에 가깝다. | 중간 | 낮음 | 05, 07, 13 |
| S | 04·05 | `feat(content): production readiness 기본 검사 추가`<br>`feat(seo): 콘텐츠 mode별 metadata 정책 추가` | `src/lib/content-readiness.ts`<br>`scripts/validate-content-readiness.ts`<br>`src/lib/site-metadata.ts`<br>`validateProductionReadiness`<br>`createRobots`<br>`createSitemap` | Template·Production 상태 기계와 공개 정책 | validity와 publishability 분리, 공개 origin 신뢰 경계, build fail-closed 정책이 시스템 전반에 일반화된다. 상세: [P05](01-content-integrity-and-security.md#p05) | 높음 | 높음 | 02, 03, 05, 15, 16 |
| A | 05 | `feat(seo): JSON-LD 안전 직렬화 경계 추가` | `src/lib/site-metadata.ts`<br>`src/components/portfolio/structured-data.tsx`<br>`serializeStructuredData`<br>`StructuredData` | HTML script 문맥의 안전한 JSON 직렬화 | `JSON.stringify`의 문맥 한계와 injection 방어를 짧은 구현으로 확인할 수 있다. 상세: [P06](01-content-integrity-and-security.md#p06) | 높음 | 중간 | 04, 13 |
| B | 05 | `feat(seo): 콘텐츠 mode별 robots 정책 추가`<br>`feat(seo): 공개 route sitemap 생성` | `src/lib/site-metadata.ts`<br>`src/app/robots.ts`<br>`src/app/sitemap.ts`<br>`createPortfolioMetadata`<br>`createRobots`<br>`createSitemap` | metadata·robots·sitemap의 동일 상태 정책 | 검색 게시 정책의 일관성은 설명 가치가 높지만 핵심 구현 판단은 P05에 통합됐다. | 높음 | 낮음 | 03, 04, 13 |
| S | 06 | `refactor(content): 홈 route view model 경계 추가`<br>`test(content): route view model 파생 규칙 검증` | `src/lib/portfolio/view-models.ts`<br>`src/lib/portfolio/view-models.test.ts`<br>`PortfolioRouteViewModel`<br>`createProjectIndexViewModel`<br>`createProjectDetailViewModel` | Route별 least-privilege view model과 선계산 join | 전체 도메인 노출 방지, `Map` 기반 참조 해석, 순서·nullability invariant를 동시에 검증한다. 상세: [P08](02-domain-routing-and-renderers.md#p08) | 높음 | 높음 | 01, 08, 09, 10, 11, 12 |
| B | 07 | `refactor(routes): 홈 page context 통합`<br>`refactor(projects): 프로젝트 page context 통합` | `src/lib/portfolio/page-context.ts`<br>`resolvePortfolioPageContext`<br>`PortfolioPagePath` | route context의 서버 경계 통합 | 중복 해소와 shell props 조립은 좋은 설계 질문이지만 직접 구현은 P03·P09와 겹친다. | 중간 | 낮음 | 03, 06, 08 |
| S | 07 | `refactor(ui): 디자인 선택기를 server markup으로 전환` | `src/components/portfolio/design-switcher.tsx`<br>`src/components/portfolio/design-switcher-close.tsx`<br>`src/components/portfolio/design-switcher.test.tsx`<br>`DesignSwitcher` | Native state, hydration race, 최소 client island, focus 복구 | hydration 전에 바뀐 DOM 상태 보존과 progressive enhancement를 직접 설명·구현할 가치가 높다. 상세: [P09](02-domain-routing-and-renderers.md#p09) | 높음 | 중간 | 09, 13, 14 |
| S | 08 | `refactor(routes): renderer view model 요청 타입 추가`<br>`refactor(designs): 모든 route를 registry renderer로 위임` | `src/designs/config.ts`<br>`src/designs/types.ts`<br>`src/designs/registry.tsx`<br>`DesignRouteRequestProps`<br>`renderDesignRoute` | Correlated discriminated union과 lazy registry | mapped type·`Extract`·exhaustive registry·동적 import를 한 TypeScript 구현 문제로 확인할 수 있다. 상세: [P10](02-domain-routing-and-renderers.md#p10) | 높음 | 높음 | 06, 09, 10, 11, 12, 13, 14 |
| A | 09 | `feat(home): 애니메이션 터미널 상호작용 추가` | `src/components/portfolio/animated-terminal.tsx`<br>`formatTerminalLine`<br>`AnimatedTerminal` | 타이머 기반 FSM, stale closure, cleanup, reduced motion | UI 효과 자체보다 phase invariant와 lifecycle 정리가 일반적인 비동기 상태 문제다. 상세: [P11](03-runtime-performance-and-release.md#p11) | 높음 | 높음 | 07, 13, 14 |
| C | 09 | `feat(app): 콘텐츠 기반 디자인 홈 연결` | `src/designs/design/`<br>`src/designs/classic/` | 반복 컴포넌트 조립과 스타일 변형 | 콘텐츠 표시 구조는 풍부하지만 같은 역량이 반복되고 프로젝트 고유 UI 비중이 크다. | 낮음 | 낮음 | 06, 08, 13 |
| C | 10 | `style(editorial): footer와 hero 활자 체계 구성`<br>`feat(editorial): About 정체성과 원칙 소개 추가` | `src/designs/editorial/editorial-route.tsx`<br>`src/designs/editorial/editorial-route.module.css` | 편집형 레이아웃과 타이포그래피 | 시각 설계 설명은 가능하지만 핵심 언어·상태·실패 경계 구현 문제로는 약하다. | 낮음 | 없음 | 06, 08, 13 |
| B | 11 | `refactor(brutalist): 내부 helper 공개 범위 정리` | `src/designs/brutalist/brutalist-route.tsx`<br>`brutalistHref`<br>`renderCopyTemplate`<br>`groupProjects`<br>`isCurrentNavigation` | 모듈 공개 표면과 helper 캡슐화 | 불필요한 export를 줄이는 판단은 설명 가치가 있으나 helper 구현은 단순하고 프로젝트 특화다. | 중간 | 낮음 | 06, 08, 13 |
| C | 12 | `feat(cinematic-home): 소개와 대표 프로젝트 구성`<br>`feat(designs): Cinematic renderer 활성화` | `src/designs/cinematic/cinematic-route.tsx`<br>`src/designs/cinematic/cinematic.module.css` | 장면·chapter 기반 서버 renderer | renderer contract의 소비 예시이지만 독립 면접 역량은 P08·P10에 이미 대표된다. | 낮음 | 없음 | 06, 08, 13 |
| A | 13 | `test(routes): 홈과 route presentation 계약 검증`<br>`test(e2e): production server 검증 경로 추가` | `src/app/page.test.tsx`<br>`src/app/routes.test.tsx`<br>`tests/e2e/site-matrix.ts`<br>`tests/e2e/portfolio.spec.ts`<br>`enabledRoutes`<br>`withExplicitDesign` | Route × Design 상태 공간과 invariant 기반 테스트 | 조합 누락·fixture drift·semantic assertion을 다루는 테스트 설계 문제다. 상세: [P12](04-testing-contracts.md#p12) | 높음 | 중간 | 03, 05, 06, 07, 08, 09, 10, 11, 12 |
| A | 13 | `fix(a11y): 디자인별 색상 대비 보정` | `tests/e2e/accessibility.spec.ts`<br>`tests/e2e/site-matrix.ts`<br>`src/app/globals.css`<br>`src/designs/editorial/editorial-route.module.css`<br>`expectAccessibleRoute` | 자동 접근성 검사, keyboard focus, reduced-motion 계약 | axe만으로 증명되지 않는 행동 경계와 다차원 진단을 묻기 좋다. 상세: [P13](04-testing-contracts.md#p13) | 높음 | 중간 | 07, 09, 10, 11, 12, 14 |
| B | 13 | `test(visual): 다섯 디자인 회귀 기준 추가` | `tests/e2e/visual.spec.ts`<br>`src/designs/visual-regression-contract.test.ts`<br>`playwright.config.ts` | 안정화된 visual snapshot과 manifest 계약 | 폰트·이미지·motion 안정화 설명은 중요하지만 스냅샷 자체는 면접 구현 가치가 낮다. | 높음 | 낮음 | 09, 10, 11, 12, 14 |
| S | 14 | `build(perf): route별 client asset 측정 추가`<br>`build(perf): route bundle 성장 예산 평가 추가` | `scripts/route-budgets.mjs`<br>`scripts/route-budgets.d.mts`<br>`src/performance/build-manifest-contract.test.ts`<br>`src/performance/performance-gates.test.ts`<br>`evaluateRouteBudgets` | Production manifest 측정, dedupe, baseline completeness | 실제 산출물 I/O와 순수 예산 알고리즘, compiler 계약 위험을 함께 다룬다. 상세: [P14](03-runtime-performance-and-release.md#p14) | 높음 | 높음 | 08, 13, 15, 16 |
| A | 14 | `test(perf): 사용자 상호작용 지연 측정 추가`<br>`build(perf): Lighthouse 결과 요약기 추가` | `tests/e2e/interaction-performance.spec.ts`<br>`scripts/summarize-lighthouse.mjs`<br>`lighthouserc.cjs`<br>`performance/lighthouse-baseline.json` | Event Timing, 표본 안정화, median·max, lab gate | 측정값을 만드는 동기화와 통계 판단을 설명하고 축소 집계기를 구현할 가치가 높다. 상세: [P15](03-runtime-performance-and-release.md#p15) | 높음 | 중간 | 07, 09, 13, 16 |
| B | 14 | `refactor(ui): reveal 콘텐츠를 server에서 즉시 표시`<br>`perf(font): route별 글꼴 로딩 비용 축소` | `src/components/portfolio/reveal.tsx`<br>`src/app/layout.tsx` | client 제거, font preload, prefetch 비용 선택 | 성능 판단은 중요하지만 설정·markup 변경이 중심이며 P09·P14·P15에 핵심 원리가 통합됐다. | 높음 | 낮음 | 07, 09, 13 |
| S | 15 | `test(docker): runtime route와 public 자산 검증 자동화` | `scripts/verify-container-runtime.mjs`<br>`scripts/verify-build-output.mjs`<br>`Dockerfile`<br>`next.config.ts` | subprocess lifecycle, readiness polling, 동적 리소스, cleanup 오류 우선순위 | 외부 프로세스와 임시 자원의 성공·실패·정리를 모두 묻는 대표적인 실전 문제다. 상세: [P16](03-runtime-performance-and-release.md#p16) | 높음 | 높음 | 03, 04, 13, 14, 16 |
| B | 15 | `build: standalone server 산출물 생성`<br>`build(docker): public 자산을 포함한 비루트 standalone image 추가` | `next.config.ts`<br>`Dockerfile`<br>`.dockerignore` | standalone 산출물, multi-stage image, 비루트 실행 | 패키징·권한·복사 경계 설명은 중요하지만 직접 Dockerfile 암기보다 P16 lifecycle 판단이 대표적이다. | 높음 | 낮음 | 04, 14, 16 |
| A | 16 | `ci: harden portfolio validation`<br>`build: improve Makefile and separate functional portfolio checks` | `.github/workflows/web-portfolio-ci.yml`<br>`Makefile`<br>`package.json`<br>`quality`<br>`production`<br>`container`<br>`verify` | 검증 DAG, 병렬화, build 재사용, final aggregate, 공급망 hardening | 의존 그래프·실패 전파·재현 가능한 명령을 일반 DAG 구현과 시스템 설계 질문으로 연결할 수 있다. 상세: [P17](03-runtime-performance-and-release.md#p17) | 높음 | 높음 | 03, 04, 13, 14, 15 |

## 대표 면접 포인트와 연관 Thread

| 포인트 | 우선순위 | 대표 Thread | 명시적으로 통합·연결한 Thread | 상세 워크북 |
|---|---|---|---|---|
| P01 런타임 스키마와 정적 타입 | S | 02 | 01, 03, 04 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p01) |
| P02 교차 파일 정합성 검증기 | S | 03 | 02, 04, 06, 13 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p02) |
| P03 URL 구조와 route state | S | 03·07 | 05, 08, 13 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p03) |
| P04 공개 자산 containment | S | 03 | 04, 15 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p04) |
| P05 Template·Production readiness | S | 04·05 | 02, 03, 15, 16 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p05) |
| P06 안전한 JSON-LD 직렬화 | A | 05 | 04, 13 | [01-content-integrity-and-security.md](01-content-integrity-and-security.md#p06) |
| P07 Canonical facade와 clone 경계 | A | 01·13 | 02, 06 | [02-domain-routing-and-renderers.md](02-domain-routing-and-renderers.md#p07) |
| P08 Route별 최소 view model | S | 06 | 01, 08, 09, 10, 11, 12 | [02-domain-routing-and-renderers.md](02-domain-routing-and-renderers.md#p08) |
| P09 Hydration-safe native state | S | 07 | 09, 13, 14 | [02-domain-routing-and-renderers.md](02-domain-routing-and-renderers.md#p09) |
| P10 타입 안전 lazy registry | S | 08 | 06, 09, 10, 11, 12, 13, 14 | [02-domain-routing-and-renderers.md](02-domain-routing-and-renderers.md#p10) |
| P11 타이머 FSM과 cleanup | A | 09 | 07, 13, 14 | [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md#p11) |
| P12 Route × Design matrix | A | 13 | 03, 05, 06, 07, 08, 09, 10, 11, 12 | [04-testing-contracts.md](04-testing-contracts.md#p12) |
| P13 접근성·keyboard·motion 계약 | A | 13 | 07, 09, 10, 11, 12, 14 | [04-testing-contracts.md](04-testing-contracts.md#p13) |
| P14 Route bundle budget | S | 14 | 08, 13, 15, 16 | [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md#p14) |
| P15 상호작용·lab 성능 측정 | A | 14 | 07, 09, 13, 16 | [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md#p15) |
| P16 컨테이너 runtime lifecycle | S | 15 | 03, 04, 13, 14, 16 | [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md#p16) |
| P17 CI gate DAG | A | 16 | 03, 04, 13, 14, 15 | [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md#p17) |

## 상세 워크북 파일 안내

- [01-content-integrity-and-security.md](01-content-integrity-and-security.md): P01~P06. 스키마, 교차 참조, URL, 자산 경계, readiness, JSON-LD 보안
- [02-domain-routing-and-renderers.md](02-domain-routing-and-renderers.md): P07~P10. facade, 최소 view model, hydration, lazy renderer registry
- [03-runtime-performance-and-release.md](03-runtime-performance-and-release.md): P11, P14~P17. 타이머 lifecycle, bundle·interaction 성능, 컨테이너, CI DAG
- [04-testing-contracts.md](04-testing-contracts.md): P12~P13. route×design matrix, 접근성·keyboard·motion 계약

## S/A 완전성 검증

| 포인트 | 우선순위 | 마스터 선별 행의 처리 상태 | 상세 위치 |
|---|---|---|---|
| P01 | S | 독립 상세 항목 | [01 문서 P01](01-content-integrity-and-security.md#p01) |
| P02 | S | 독립 상세 항목 | [01 문서 P02](01-content-integrity-and-security.md#p02) |
| P03 | S | Thread 03의 route 검증과 Thread 07의 URL 상태 생성을 하나로 통합 | [01 문서 P03](01-content-integrity-and-security.md#p03) |
| P04 | S | 독립 상세 항목 | [01 문서 P04](01-content-integrity-and-security.md#p04) |
| P05 | S | Thread 04 readiness와 Thread 05 게시 정책을 하나로 통합 | [01 문서 P05](01-content-integrity-and-security.md#p05) |
| P06 | A | 독립 상세 항목 | [01 문서 P06](01-content-integrity-and-security.md#p06) |
| P07 | A | Thread 01 facade와 Thread 13 clone regression contract를 통합 | [02 문서 P07](02-domain-routing-and-renderers.md#p07) |
| P08 | S | 독립 상세 항목. Thread 09~12 renderer의 반복 조회 문제는 이 항목에 귀속 | [02 문서 P08](02-domain-routing-and-renderers.md#p08) |
| P09 | S | 독립 상세 항목. hydration·focus 회귀 검증은 Thread 13과 연결 | [02 문서 P09](02-domain-routing-and-renderers.md#p09) |
| P10 | S | 독립 상세 항목. Thread 09~12 renderer family는 registry 소비자로 통합 | [02 문서 P10](02-domain-routing-and-renderers.md#p10) |
| P11 | A | 독립 상세 항목. reduced-motion·성능 경계는 Thread 13·14와 연결 | [03 문서 P11](03-runtime-performance-and-release.md#p11) |
| P12 | A | 독립 상세 항목 | [04 문서 P12](04-testing-contracts.md#p12) |
| P13 | A | 독립 상세 항목 | [04 문서 P13](04-testing-contracts.md#p13) |
| P14 | S | 독립 상세 항목 | [03 문서 P14](03-runtime-performance-and-release.md#p14) |
| P15 | A | interaction 표본과 Lighthouse 반복 집계를 하나의 측정 신뢰성 항목으로 통합 | [03 문서 P15](03-runtime-performance-and-release.md#p15) |
| P16 | S | standalone packaging 설명을 runtime lifecycle 대표 문제에 통합 | [03 문서 P16](03-runtime-performance-and-release.md#p16) |
| P17 | A | workflow job DAG와 Makefile·package 명령 재현성을 통합 | [03 문서 P17](03-runtime-performance-and-release.md#p17) |

검증 결과: S 10개와 A 7개가 모두 독립 상세 항목 또는 명시적 통합 항목으로 연결됐다. 미연결 S/A 항목은 없다.

## 백지 구현 우선순위

1. **P02** 교차 파일 참조를 모두 수집하는 정합성 검증기
2. **P10** discriminated union 기반 lazy renderer registry
3. **P16** 외부 프로세스·임시 리소스·cleanup orchestrator
4. **P03** 내부 URL 분류와 route state canonicalization
5. **P14** route bundle baseline 예산 평가기
6. **P08** route별 최소 view model과 선계산 join
7. **P01** 런타임 schema 파싱 경계
8. **P17** CI gate DAG 검사와 병렬 level 계획
9. **P04** 공개 자산 경로 containment 검사
10. **P05** Template·Production readiness 검증기
11. **P11** 타이머 기반 유한 상태 기계와 cleanup
12. **P07** selective clone snapshot
13. **P12** Route × Design matrix 생성기
14. **P06** script 문맥 안전 JSON 직렬화
15. **P15** interaction 표본 median·max 요약기
16. **P09** native `<details>` 최소 controller
17. **P13** 접근성 route-case 검증 helper

## 설명 연습 우선순위

1. **P05** "유효한 콘텐츠"와 "공개 가능한 콘텐츠"를 분리한 이유
2. **P09** hydration 전 native DOM 상태를 React가 덮지 않게 한 이유
3. **P14** 실제 production manifest를 측정하고 baseline 누락도 실패시킨 이유
4. **P16** 본 오류와 cleanup 오류의 우선순위를 정한 방식
5. **P10** route와 view model의 상관관계를 mapped union으로 표현한 원리
6. **P02** fail-fast 대신 모든 정합성 이슈를 누적한 이유
7. **P13** axe 결과 외에 keyboard·focus·reduced-motion 계약이 필요한 이유
8. **P17** 검증 DAG, build 재사용, final verify job의 역할
9. **P08** renderer에 전체 도메인을 넘기지 않은 이유
10. **P03** URL 문자열 연결을 금지하고 default 상태를 생략한 이유
11. **P15** Event Timing, warm-up, median과 max를 함께 사용한 이유
12. **P07** 전체 deep clone 대신 선택적 복제를 계약한 이유
13. **P01** 스키마 출력 타입을 도메인 타입 근원으로 삼은 이유
14. **P06** JSON 유효성과 HTML script 문맥 안전성이 다른 이유
15. **P11** 타이머 애니메이션을 phase 상태 기계로 만든 이유
16. **P12** 전체 상태 공간과 공통 invariant를 분리한 테스트 전략
17. **P04** lexical containment와 `realpath` containment의 차이

## 한 문제로 통합한 Thread 묶음

- **P03 — Thread 03 + 07:** 콘텐츠 내부 route 검증과 UI 링크의 query·hash·디자인 상태 생성을 "구조화된 URL invariant" 하나로 통합
- **P05 — Thread 04 + 05:** production readiness와 metadata·robots·sitemap 게시 정책을 "명시적 콘텐츠 mode 상태 기계" 하나로 통합
- **P07 — Thread 01 + 13:** canonical facade와 호출별 clone regression contract를 "aliasing·mutation isolation" 하나로 통합
- **P08 — Thread 06 + 09~12:** route view model 생성과 여러 renderer의 데이터 소비를 "least-privilege projection·선계산 join" 하나로 통합
- **P09 — Thread 07 + 13:** server markup·native state 설계와 hydration 회귀 테스트를 "hydration-safe progressive enhancement" 하나로 통합
- **P10 — Thread 06 + 08 + 09~12:** view model discriminant, lazy registry, renderer family를 "상관관계가 있는 타입 안전 dispatch" 하나로 통합
- **P11 — Thread 09 + 13 + 14:** 타이머 FSM, reduced-motion 회귀, 상호작용 비용을 "시간 기반 UI lifecycle" 하나로 통합
- **P12 — Thread 03 + 05~13:** route enablement, mode, URL, view model, registry, renderer를 "지원 상태 공간의 contract matrix" 하나로 통합
- **P13 — Thread 07 + 09~14:** native focus, 여러 renderer의 semantic 구조, motion 정책을 "행동 중심 접근성 계약" 하나로 통합
- **P14 — Thread 14 + 16:** route bundle 측정·baseline gate와 CI 실행을 "production artifact 예산" 하나로 통합
- **P15 — Thread 13 + 14 + 16:** production E2E 환경, interaction 표본, Lighthouse 반복 실행을 "재현 가능한 성능 측정" 하나로 통합
- **P16 — Thread 03 + 04 + 15 + 16:** 자산·readiness 전제, standalone image, runtime route 검증, CI cleanup을 "임시 배포 리소스 lifecycle" 하나로 통합
- **P17 — Thread 13~16:** 기능·접근성·성능·컨테이너 검증 명령을 "의존성과 실패 의미가 명시된 CI gate DAG" 하나로 통합
