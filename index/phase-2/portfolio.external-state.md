# External-State Development Gap Audit — tmp-portfolio

- Audit snapshot: 2026-08-30
- Repository: `seungwoo7050/tmp-portfolio`
- Current default branch: `main`
- Current observed HEAD: `92197c0c78e3b458b10bb2bba37218d96b2450bb`
- Existing Thread Index: attached `Portfolio.md`
- Audit result:
  - Accepted External-State Gaps: **7**
  - `EXISTING_THREAD`: **7**
  - `NEW_THREAD`: **0**
  - `PROJECT_LEVEL_EXTERNAL_STEP`: **0**

## Audit Boundary

이 감사는 Existing Thread Index를 소유권 판정 기준으로만 사용했다. 기존 Thread별 학습 문서나 해설서는 사용하지 않았다.

GitHub API를 통해 현재 파일 트리, 현재 설정·소스, 대표 커밋 diff, 파일별 history, 전체 commit pagination을 조사했다. 전체 commit listing은 page 1–5에 데이터가 있고 page 6이 비어 있는 지점까지 확인했다.

현재 원격 branch 목록에는 `main`만 존재한다. 현재 workflow는 `web/portfolio` branch의 push 및 pull request만 자동 trigger 대상으로 지정한다.

현재 checked-in content는 template marker와 placeholder destination을 포함하며 `PORTFOLIO_CONTENT_MODE`의 기본값도 `template`이다. 그러므로 repository만으로 production build 또는 production deployment가 실제 수행되었다고 판단할 수 없다.

---

# Part I — Gap Index

## GAP-ES-01 — Production 모드·공개 Origin 환경 계약 활성화

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 04 — Template·Production 콘텐츠 준비도 정책
- **Related Threads:** Thread 05, Thread 15, Thread 16
- **Repository Evidence 요약:**
  - `src/lib/content-readiness.ts`는 `PORTFOLIO_CONTENT_MODE`와 `SITE_URL`을 production readiness 입력으로 사용한다.
  - `production` 모드에서는 `SITE_URL`이 필요하며, local·reserved placeholder·credential-bearing URL을 거부한다.
  - `package.json`의 `prebuild`는 `content:check`와 `content:ready`를 실행한다.
  - `Dockerfile` builder는 `PORTFOLIO_CONTENT_MODE`와 `SITE_URL`을 build argument로 받지만 기본 모드는 `template`이다.
  - runner stage에는 이 두 값의 final runtime contract가 별도로 선언되어 있지 않다.
- **Required External Step 요약:**
  - 실제 게시에 사용할 public HTTP(S) origin을 결정한다.
  - production build 실행환경에 `PORTFOLIO_CONTENT_MODE=production`과 동일 origin의 `SITE_URL`을 주입한다.
  - build artifact가 runtime에서 같은 mode/origin 의미를 유지하는지 검증하고, 필요하면 동일 값을 runtime에도 주입한다.
- **실제 수행 여부 확인 가능성:** 값·provider·주입 시점·실제 production build 성공 여부는 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 04에 "production environment activation contract" supplement를 추가한다. 정확한 값이나 provider를 기록하지 않고 build-time 입력, runtime consistency 확인, 실패 경계를 문서화한다.

## GAP-ES-02 — 실제 Origin에서 Canonical·Robots·Sitemap·JSON-LD 게시

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 05 — 검색 메타데이터와 구조화 게시
- **Related Threads:** Thread 04, Thread 15
- **Repository Evidence 요약:**
  - production mode의 metadata, canonical URL, robots host/sitemap, sitemap route URL, site/project JSON-LD는 `SITE_URL` origin으로 구성된다.
  - template mode는 indexing을 차단하고 sitemap을 비운다.
- **Required External Step 요약:**
  - GAP-ES-01에서 선택한 origin과 동일한 origin에서 application을 실제로 서비스한다.
  - 실제 HTTP 응답의 canonical, robots.txt, sitemap.xml, JSON-LD 절대 URL이 배포 origin과 일치하는지 확인한다.
- **실제 수행 여부 확인 가능성:** public endpoint, crawler 접근, 검색엔진 indexing 여부는 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 05에 "real-origin publishing verification" supplement를 추가한다. Search Console 등록·domain verification 등 repository에 없는 작업은 추가하지 않는다.

## GAP-ES-03 — 외부 연락처·프로젝트 목적지의 실재성·소유권·도달 가능성

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 04 — Template·Production 콘텐츠 준비도 정책
- **Related Threads:** Thread 02, Thread 03, Thread 05
- **Repository Evidence 요약:**
  - production readiness는 contact placement에 있는 email/GitHub/website 중 하나와, enabled project마다 하나 이상의 public URL을 요구한다.
  - validator는 placeholder·URL parsing·protocol을 확인하지만 외부 endpoint의 소유권, 실제 존재, HTTP health, content truthfulness를 검증하지 않는다.
  - 현재 content에는 `your-handle`, `hello@example.com`, example project와 disabled source link가 남아 있다.
- **Required External Step 요약:**
  - 실제 email/profile/website/source/demo destination을 생성 또는 확보하고 소유권을 확인한다.
  - content에서 enable하거나 `live`/`source-only` 등의 상태를 주장하기 전에 실제 도달 가능성과 주장 일치를 검증한다.
- **실제 수행 여부 확인 가능성:** 외부 계정 생성, 주소 소유권, endpoint health, exact destination은 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 04에 "syntactic readiness vs external truth" 경계를 추가한다.

## GAP-ES-04 — Standalone 컨테이너의 실제 빌드·기동·서비스 상태

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 15 — 재현 가능한 Standalone 컨테이너 패키징과 런타임 검증
- **Related Threads:** Thread 04, Thread 05, Thread 16
- **Repository Evidence 요약:**
  - Next standalone output과 multi-stage non-root Docker image가 정의되어 있다.
  - container verification script는 임시 image를 기본 template mode로 build하고, loopback random port에 container를 기동한 뒤 route·asset·user를 검사하고 image/container를 삭제한다.
  - registry, hosting provider, service definition, ingress, health monitor, release/rollback manifest는 없다.
- **Required External Step 요약:**
  - Docker-compatible build/runtime engine에서 의도한 production input으로 image를 build한다.
  - container 또는 equivalent service를 실제로 기동·감시하고 port 3100을 외부 routing 계층에 연결한다.
  - service readiness와 public assets를 target environment에서 확인한다.
- **실제 수행 여부 확인 가능성:** local verification 실행 여부, image publication, production service creation, ingress, monitoring, rollout/rollback은 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 15에 "ephemeral verification vs durable deployment state" supplement를 추가한다.

## GAP-ES-05 — 브라우저 설치와 시각 기준선 생성 환경

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 13 — 다중 디자인 Route Matrix의 기능·접근성·시각 회귀 계약
- **Related Threads:** Thread 16
- **Repository Evidence 요약:**
  - Playwright production config는 Chromium desktop/mobile projects와 local production server를 정의한다.
  - visual test는 다섯 design의 home desktop/mobile과 project desktop snapshot을 비교하며 baseline PNG가 commit되어 있다.
  - current CI command는 `@visual` tests를 제외한다.
- **Required External Step 요약:**
  - 호환 Chromium과 필요한 OS dependency/font rendering environment를 설치한다.
  - full production visual suite를 명시적으로 실행하고, 의도된 변경만 baseline으로 재생성·검토·commit한다.
- **실제 수행 여부 확인 가능성:** baseline files의 존재는 직접 관찰되지만, 각 baseline을 생성한 정확한 browser/OS/font state와 reviewer acceptance는 Git만으로 완전히 확인할 수 없다.
- **Documentation Action:** Thread 13에 visual baseline provenance와 update/acceptance 절차를 보완한다.

## GAP-ES-06 — Lighthouse Lab 실행환경과 측정 Provenance

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 14 — 클라이언트 비용 최적화와 성능 예산
- **Related Threads:** Thread 13, Thread 16
- **Repository Evidence 요약:**
  - Lighthouse config는 local production server, 5-design × home/project matrix, URL당 3회, median aggregation, desktop budget을 정의한다.
  - committed baseline은 측정 시각, Mac arm64/M1, Chrome user-agent, Node version, route별 runs/median을 기록한다.
  - raw `.lighthouseci` output은 ignore된다.
- **Required External Step 요약:**
  - compatible Chrome을 설치하고 production server를 기동하여 동일 matrix를 실행한다.
  - raw report를 검토하고 environment drift를 기록한 후, 승인된 결과만 tracked baseline으로 갱신한다.
- **실제 수행 여부 확인 가능성:** committed JSON은 "측정 결과 기록"의 직접 증거지만 external Chrome process의 실행을 독립적으로 증명하지 않으며, 현재 CI rerun 결과와 raw report는 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 14에 measurement provenance, reproducibility limit, report retention boundary를 추가한다.

## GAP-ES-07 — CI Trigger 정합성·Runner Capability·Artifact 수명주기

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 16 — 재현 가능한 검증 명령과 CI Gate DAG
- **Related Threads:** Thread 13, Thread 14, Thread 15
- **Repository Evidence 요약:**
  - current workflow는 `web/portfolio` branch의 push/PR만 trigger한다.
  - current remote branch 목록에는 `main`만 있다.
  - workflow는 Chromium+OS dependency 설치, `$GITHUB_ENV`의 `CHROME_PATH`, Docker 실행, failure artifact upload, 7-day retention에 의존한다.
  - secret reference는 없다.
- **Required External Step 요약:**
  - committed workflow를 그대로 사용할 경우 `web/portfolio` branch/event path를 실제로 만든다. 대안으로 source에서 trigger target을 변경할 수 있지만 이는 future code change다.
  - repository Actions를 실행 가능하게 하고, browser 설치·Docker daemon·network access가 가능한 runner를 제공한다.
  - run 결과와 failure artifact를 GitHub Actions의 external state에서 확인한다.
- **실제 수행 여부 확인 가능성:** workflow source와 current branch mismatch는 관찰 가능하지만 Actions enablement, run status, artifact existence/retention은 Git에서 확인할 수 없다.
- **Documentation Action:** Thread 16에 "workflow definition vs platform activation" supplement와 trigger-alignment precondition을 추가한다.

---

# Part II — Existing Thread Supplement Packets

## Packet E04 — Thread 04

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 04
- **한국어 제목:** Template·Production 콘텐츠 준비도 정책
- **English title:** Template and Production Content Readiness Policies

### Gaps

- GAP-ES-01
- GAP-ES-03

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `71e7ece7208f07f8a1532b7848d993ab5ba088d6` | `feat(content): 연락 수단과 build readiness 연결` | `src/lib/content-readiness.ts` | production contact requirement와 mode-aware build readiness를 연결한다. |
| `37c0dbc079ff0bc56d1bb2b03054b2b7d8d9834b` | `build(content): readiness 검사를 prebuild에 연결` | `package.json`, `scripts/validate-content-readiness.ts` | build 전에 environment-backed readiness를 실행한다. |
| `fb3d18fd660b6664a41b90997a793d60bd137f01` | `test(content): readiness와 indexing 계약 검증` | `src/lib/content-readiness.test.ts`, `src/lib/site-metadata.test.ts` | template default, production input categories, public site URL, indexing behavior를 고정한다. |

#### Relevant diff excerpts

`71e7ece...`의 핵심 변화:

```diff
+ const hasContactMethod = content.links.some(...)
+ if (!hasContactMethod) {
+   issues.push(... "Enable at least one non-placeholder contact method")
+ }

+ export function validateBuildReadiness(content, environment) {
+   const mode = resolvePortfolioContentMode(environment.PORTFOLIO_CONTENT_MODE)
+   if (mode === "template") return { mode, siteUrl: undefined }
+   return validateProductionReadiness(content, environment)
+ }
```

`37c0dbc...`의 핵심 변화:

```diff
- "prebuild": "npm run content:check"
+ "prebuild": "npm run content:check && npm run content:ready"
+ "content:ready": "node --import tsx scripts/validate-content-readiness.ts"
```

```ts
validateBuildReadiness(loadPortfolioSource(), {
  PORTFOLIO_CONTENT_MODE: process.env.PORTFOLIO_CONTENT_MODE,
  SITE_URL: process.env.SITE_URL,
});
```

#### Final-state source/configuration

`src/lib/content-readiness.ts`:

- undefined/empty/`template` mode는 template로 처리된다.
- production은 explicit `production` value가 필요하다.
- production `SITE_URL`은 absolute HTTP(S) URL이어야 하며 localhost, loopback, reserved placeholder, embedded username/password를 거부한다.
- production content는 placeholder marker가 없어야 한다.
- production assets는 `/content/` 아래를 사용해야 한다.
- enabled project마다 parse 가능한 public URL 하나 이상이 필요하다.
- contact placement에는 email/GitHub/website 중 하나가 필요하다.
- URL validator는 external network request나 ownership proof를 수행하지 않는다.

`Dockerfile`:

```dockerfile
ARG PORTFOLIO_CONTENT_MODE=template
ARG SITE_URL
ENV PORTFOLIO_CONTENT_MODE=$PORTFOLIO_CONTENT_MODE
ENV SITE_URL=$SITE_URL
RUN npm run build && npm run build:verify
```

Current content evidence:

- `site.json`에 template title/name marker가 남아 있다.
- `links.json`에 `your-handle`, `hello@example.com`이 있다.
- `projects.json`에 example project와 disabled placeholder source URL이 있다.

### External Development Steps

1. 실제 contact/profile/project destination을 생성 또는 확보한다.
2. destination의 소유권·도달 가능성·내용 일치를 확인한다.
3. 실제 게시 origin을 선택한다. custom domain일 필요는 없으며 repository는 provider를 지정하지 않는다.
4. build environment에 `PORTFOLIO_CONTENT_MODE=production`과 exact `SITE_URL`을 주입한다.
5. final runtime이 동일 mode/origin을 해석하는지 확인한다. repository는 build artifact가 모든 relevant environment semantics를 완전히 고정하는지 명시하지 않으므로, runtime injection 또는 artifact behavior verification 중 하나가 필요하다.
6. readiness와 build를 실행하여 실패가 없는지 확인한다.

Tracked prerequisite: template content와 assets를 production content로 교체하는 작업은 code/content diff로 남으므로 External-State Gap 자체가 아니다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| public origin 선택 | `parsePublicSiteUrl`, metadata, robots, sitemap, JSON-LD의 absolute URL |
| production mode 주입 | `resolvePortfolioContentMode`, `validateBuildReadiness`, `prebuild` |
| external destination 확보 | `isUsablePublicUrl`, `isUsableContactHref`, project/contact readiness |
| runtime consistency 확인 | `layout.tsx`, `robots.ts`, `sitemap.ts`의 server-side `process.env` read |
| readiness 실행 | `scripts/validate-content-readiness.ts`, `package.json#prebuild` |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | environment variable names, allowed mode values, URL rejection policy, readiness rules, prebuild wiring, Docker builder arguments, current template markers |
| **Required/inferred from repository** | real destination ownership, production origin selection, environment injection, build/runtime semantic consistency |
| **Actual execution not observable from Git** | exact values, provider, account owner, creation time, environment configuration, production build result, endpoint health |

### Ordering

**Conceptual execution order**, not an observed historical chronology:

1. tracked production content/assets 준비
2. external destinations 확보·검증
3. public origin 선택
4. build-time environment injection
5. production readiness/build
6. runtime mode/origin consistency 확인

---

## Packet E05 — Thread 05

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 05
- **한국어 제목:** 검색 메타데이터와 구조화 게시
- **English title:** Search Metadata and Structured Publishing

### Gaps

- GAP-ES-02

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `cb61450ad922d263584de60179f5c0dc3d920461` | `feat(seo): 콘텐츠 mode별 robots 정책 추가` | `src/app/robots.ts`, `src/lib/site-metadata.ts` | production mode와 public origin을 robots policy에 연결한다. |
| `70b69f04e8c7f45aa8d04708954123535a1ff393` | `feat(seo): 공개 route sitemap 생성` | `src/app/sitemap.ts`, `src/lib/site-metadata.ts` | public route를 origin-qualified sitemap으로 게시한다. |
| `c5938ea4b4f82ee8eeeca2bb4269be66b846d33c` | `test(seo): JSON-LD 계약과 직렬화 검증` | `src/lib/site-metadata.test.ts` | Person/WebSite/CreativeWork records와 safe serialization을 검증한다. |

#### Relevant diff excerpts

`cb61450...`:

```diff
+ const mode = resolvePortfolioContentMode(process.env.PORTFOLIO_CONTENT_MODE)
+ const siteUrl =
+   mode === "production"
+     ? resolveProductionSiteUrl(process.env.SITE_URL)
+     : undefined
+ return createRobots({ mode, siteUrl })
```

`70b69f...`:

```diff
+ if (mode === "template") return []
+ if (!siteUrl) throw new Error(...)
+ return routes.map((path) => ({ url: absoluteSiteUrl(path, siteUrl) }))
```

#### Final-state source/configuration

- template mode:
  - metadata robots: `index: false`, `follow: false`
  - robots: `/` disallow
  - sitemap: empty
  - site JSON-LD: omitted
- production mode:
  - metadata and canonical use `SITE_URL`
  - robots contains origin host and origin-qualified sitemap URL
  - sitemap contains origin-qualified enabled routes
  - Person/WebSite/CreativeWork JSON-LD uses the same origin

### External Development Steps

1. GAP-ES-01의 origin으로 production artifact를 실제 서비스한다.
2. public response에서 homepage and representative project routes를 확인한다.
3. rendered canonical/Open Graph/JSON-LD absolute URLs를 확인한다.
4. `/robots.txt`와 `/sitemap.xml`의 host, policy, route set을 확인한다.
5. deployed origin과 configured `SITE_URL`이 다르면 배포를 중단하거나 재build/reconfigure한다.

Search engine crawler가 실제로 crawl/index했다는 사실은 별도 external observation이며 repository는 이를 요구하거나 기록하지 않는다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| real-origin serving | `createPortfolioMetadata`, `absoluteSiteUrl` |
| robots verification | `src/app/robots.ts`, `createRobots` |
| sitemap verification | `src/app/sitemap.ts`, `createSitemap` |
| JSON-LD verification | `layout.tsx`, `createSiteStructuredData`, `createProjectStructuredData` |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | mode-dependent indexing policy, absolute URL generation, sitemap route derivation, JSON-LD contracts |
| **Required/inferred from repository** | app must be served at the configured origin for canonical/host/sitemap statements to be truthful |
| **Actual execution not observable from Git** | public deployment URL, HTTP responses, crawler access, indexing state, social preview cache |

### Ordering

**Conceptual execution order:**

1. GAP-ES-01 complete
2. production artifact/service start
3. public origin routing
4. metadata/robots/sitemap/JSON-LD response verification
5. only then treat production publishing contract as externally complete

---

## Packet E13 — Thread 13

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 13
- **한국어 제목:** 다중 디자인 Route Matrix의 기능·접근성·시각 회귀 계약
- **English title:** Functional, Accessibility, and Visual Regression Contracts for the Multi-Design Route Matrix

### Gaps

- GAP-ES-05

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `882a2f9d753ea8ff97cc8ce6a202aeb0e394597d` | `test(visual): 다섯 디자인 회귀 기준 추가` | `playwright.config.ts`, `playwright.production.config.ts`, `tests/e2e/visual.spec.ts`, snapshot PNG files | exact visual matrix, stabilization, threshold, tracked baselines를 추가한다. |

Relevant contract:

- design IDs: design, classic, editorial, brutalist, cinematic
- per design:
  - home desktop
  - home mobile
  - project desktop
- total expected baseline manifest: 15 PNG files
- reduced motion, `document.fonts.ready`, image completion wait
- full-page screenshots
- `maxDiffPixelRatio: 0.01`
- production CI command excludes `@visual`

### External Development Steps

1. Playwright-compatible Chromium과 OS-level dependency를 설치한다.
2. production server를 build/start한다.
3. full `test:e2e:production` visual suite를 deterministic environment에서 실행한다.
4. mismatch를 inspect하고 application regression과 environment drift를 구분한다.
5. 의도된 visual change일 때만 baseline을 재생성하고 reviewer가 승인한 PNG를 commit한다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| browser provisioning | `@playwright/test`, Playwright Chromium projects |
| production server | `playwright.production.config.ts#webServer`, port 3200 |
| baseline comparison | `visual.spec.ts#toHaveScreenshot` |
| baseline manifest | `visual-regression-contract.test.ts` |
| CI boundary | `test:e2e:ci`의 `--grep-invert @visual` |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | visual test code, thresholds, projects, 15 PNG baselines, manifest check, CI exclusion |
| **Required/inferred from repository** | compatible browser/OS/font environment and intentional baseline review are necessary to apply the contract |
| **Actual execution not observable from Git** | exact capture host, installed browser binary state, font rasterization state, reviewer decision, current run result |

### Ordering

**Conceptual execution order:**

1. install browser/dependencies
2. build/start production server
3. run full visual suite
4. inspect diff
5. accept application change or repair regression
6. update tracked baseline only after approval

---

## Packet E14 — Thread 14

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 14
- **한국어 제목:** 클라이언트 비용 최적화와 성능 예산
- **English title:** Client-Cost Optimization and Performance Budgets

### Gaps

- GAP-ES-06

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `1529ccf225c10028607b6a8963c3280f4ab56a42` | `build(perf): desktop Lighthouse 실행 경계 추가` | `lighthouserc.cjs`, `package.json`, `.gitignore` | local production Lighthouse runner와 budget을 정의하고 raw output을 ignore한다. |
| `a39856cf734a309a32f5c7f239ba6bec90e1c259` | `chore(perf): 최종 lab 성능 측정 결과 기록` | `performance/lighthouse-baseline.json` | environment metadata와 route별 3-run result/median을 tracked record로 남긴다. |

Final-state contract:

- local server: `http://localhost:3300`
- 5 design IDs × home/project routes
- 3 runs per URL
- median aggregation
- desktop preset
- performance and accessibility only
- thresholds:
  - accessibility ≥ 0.95
  - performance ≥ 0.90
  - CLS ≤ 0.1
  - LCP ≤ 2500 ms
  - TBT ≤ 200 ms

Committed baseline reports:

- measured at `2026-08-13T02:52:54.086Z`
- `darwin`, `arm64`, Apple M1
- Node `v24.18.0`
- Headless Chrome 147 user agent
- route-level raw runs and medians

### External Development Steps

1. compatible Chrome binary를 설치한다.
2. production build와 local server를 port 3300에서 기동한다.
3. exact route matrix를 URL당 3회 실행한다.
4. generated `.lighthouseci` reports를 검토한다.
5. machine/browser drift와 code regression을 구분한다.
6. 승인된 measurement summary만 tracked baseline으로 반영한다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| Chrome provisioning | `lighthouserc.cjs#chromeFlags`, workflow `CHROME_PATH` |
| server start | `startServerCommand: npm run start:performance` |
| route matrix | `designIds`, first enabled project |
| budget decision | `ci.assert.assertions` |
| provenance | `performance/lighthouse-baseline.json` |
| raw report lifecycle | `.gitignore#/.lighthouseci`, CI failure artifact upload |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | runner configuration, budget values, committed baseline/result metadata |
| **Required/inferred from repository** | Chrome process, server process, three actual lab runs, report review are needed to reproduce or update the record |
| **Actual execution not observable from Git** | external process logs, raw current reports, current CI run, machine load, independent proof that the recorded run occurred exactly as described |

### Ordering

**Conceptual execution order:**

1. build production artifact
2. provision Chrome
3. start performance server
4. execute 3-run matrix
5. inspect raw reports and environment
6. enforce budget
7. update tracked baseline only when intentionally accepted

---

## Packet E15 — Thread 15

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 15
- **한국어 제목:** 재현 가능한 Standalone 컨테이너 패키징과 런타임 검증
- **English title:** Reproducible Standalone Container Packaging and Runtime Verification

### Gaps

- GAP-ES-04

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `29508f4668eaed37c393c8c2ef2e80d0e6c8e2f2` | `build: standalone server 산출물 생성` | `next.config.ts` | Next standalone output을 활성화한다. |
| `b87a2b4537418771530ae520df448ca84142f80c` | `build(docker): public 자산을 포함한 비루트 standalone image 추가` | `Dockerfile`, `.dockerignore` | pinned Node/npm, builder args, non-root runner, static/public copy, port/CMD를 정의한다. |
| `b94fa6dd0118322ff57dc84b180b1c179ca8a867` | `test(docker): runtime route와 public 자산 검증 자동화` | `scripts/verify-container-runtime.mjs`, `package.json`, historical workflow | ephemeral image/container test와 cleanup을 자동화한다. |

#### Relevant diff excerpts

`29508f...`:

```diff
 const nextConfig = {
   devIndicators: false,
+  output: "standalone",
 }
```

`b87a2b...`:

```dockerfile
ARG PORTFOLIO_CONTENT_MODE=template
ARG SITE_URL
RUN npm run build && npm run build:verify

ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3100
USER node
EXPOSE 3100
CMD ["node", "server.js"]
```

`b94fa6...` runtime test:

```text
docker build --tag <temporary-image> .
docker run --detach --publish 127.0.0.1::3100 <temporary-image>
wait until ready
inspect Config.User == node
verify two HTML routes and every content asset
remove container and image
```

### External Development Steps

1. Docker-compatible daemon/build engine을 제공한다.
2. production mode와 origin을 명시하여 image를 build한다.
3. resulting image를 target runtime에 배치하고 non-root container/service를 기동한다.
4. port 3100을 environment의 actual routing mechanism에 연결한다.
5. readiness, HTML routes, public asset MIME/body를 target environment에서 확인한다.
6. failed process를 restart하거나 failed rollout을 회수할 operational mechanism을 마련한다.

Step 6의 구체적 provider/rollback implementation은 repository가 정의하지 않는다. 따라서 특정 registry, orchestrator, load balancer, health-check syntax를 추정해서는 안 된다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| image build | Dockerfile dependencies/builder stages |
| production input | Docker `ARG` and prebuild readiness |
| non-root start | runner `USER node`, `CMD node server.js` |
| port routing | `HOSTNAME=0.0.0.0`, `PORT=3100`, `EXPOSE 3100` |
| runtime verification | `verify-container-runtime.mjs` |
| cleanup | test script finally block removes temporary container/image |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | image recipe, standalone output, non-root user, port, test build/run/inspect/fetch/cleanup sequence |
| **Required/inferred from repository** | an external container engine and durable service/routing state are needed for real deployment |
| **Actual execution not observable from Git** | image build/run history, registry publication, service provisioning, public ingress, health monitor, restart, rollout, rollback |

### Ordering

**Conceptual execution order:**

1. GAP-ES-01 inputs ready
2. image build
3. standalone artifact verification
4. image transfer or target placement
5. service/container start
6. route/asset/readiness verification
7. establish operational restart/recovery boundary

The repository's verification script proves the intended temporary lifecycle, not a durable production lifecycle.

---

## Packet E16 — Thread 16

### Thread Identity

- **Kind:** Existing Thread
- **Thread:** 16
- **한국어 제목:** 재현 가능한 검증 명령과 CI Gate DAG
- **English title:** Reproducible Validation Commands and CI Gate DAG

### Gaps

- GAP-ES-07

### Repository Evidence

| Commit | Subject | Relevant files | Importance |
|---|---|---|---|
| `abbd530368a03b8dc40221d663377c1f45e254ee` | `ci: 검증된 bundle과 Lighthouse gate 활성화` | historical `.github/workflows/ci.yml`, `package.json` | production E2E, bundle check, Chrome path, Lighthouse gate를 CI에 연결한다. |
| `61b07d6391a9f0d276f506873992109a2275cd56` | `ci: harden portfolio validation` | `.github/workflows/web-portfolio-ci.yml`, `lighthouserc.cjs`, `playwright.production.config.ts` | generic workflow를 branch-specific DAG로 교체하고 jobs, action pinning, diagnostics, final gate를 강화한다. |

Current workflow source:

```yaml
on:
  push:
    branches: [web/portfolio]
  pull_request:
    branches: [web/portfolio]
```

Current remote branch state:

```text
main
```

Current DAG:

```text
quality
  ├─> production ─┐
  └─> container  ─┴─> verify
```

External runner/platform state required by steps:

- npm dependency download
- Chromium and OS dependency installation
- executable Chromium path export through `$GITHUB_ENV`
- Docker daemon/CLI
- local server ports
- GitHub artifact store
- failure artifact retention for 7 days

No current workflow step references a secret.

### External Development Steps

1. committed trigger를 사용할 경우 remote `web/portfolio` branch와 push/PR event path를 만든다.
2. repository에서 GitHub Actions execution을 허용한다.
3. job에 필요한 runner/network/browser-install/Docker capabilities를 제공한다.
4. matching event를 발생시켜 DAG를 실행한다.
5. quality, production, container, final verify results를 platform state에서 확인한다.
6. failure 시 uploaded diagnostics를 7-day retention window 안에 수집한다.

Trigger를 `main`으로 바꾸는 대안은 possible future code/config diff이므로 이 audit의 external step으로 수행되었다고 간주하지 않는다.

### Code Connection

| External step | Code/config/runtime connection |
|---|---|
| branch/event alignment | workflow `on.push.branches`, `on.pull_request.branches` |
| Actions activation | `.github/workflows/web-portfolio-ci.yml` discovery/execution |
| browser capability | Playwright install and Lighthouse `CHROME_PATH` |
| Docker capability | container job `npm run test:container` |
| DAG result | `needs` and final `verify` job |
| artifact lifecycle | `actions/upload-artifact`, `retention-days: 7` |

### Evidence Boundary

| Boundary | Finding |
|---|---|
| **Directly observed in repository** | workflow source, current branch list, trigger mismatch, job DAG, runner image, action pins, artifact config |
| **Required/inferred from repository** | Actions must be enabled and a matching branch/event plus capable runner must exist |
| **Actual execution not observable from Git** | workflow run IDs/status, runner instance, installed packages, Docker state, generated reports, uploaded artifact existence/expiration |

### Ordering

**Conceptual execution order:**

1. align branch/event path
2. enable/permit Actions
3. ensure runner capabilities
4. trigger event
5. execute DAG
6. inspect final status
7. retrieve failure diagnostics within retention window

---

# Part III — Proposed New Thread Packets

## No NEW_THREAD proposed

외부 상태 감사로 확인된 모든 accepted gap은 기존 Thread 04, 05, 13, 14, 15, 16의 관점을 실제 환경에서 완성하는 단계다.

"Production Deployment Operations" 또는 "Environment Provisioning" 같은 신규 Thread는 제안하지 않는다. Repository에는 다음이 없다.

- provider-specific deployment manifest
- infrastructure-as-code
- registry/release promotion model
- staging/production environment model
- deployment credential lifecycle
- rollout/rollback implementation
- external health monitoring or incident recovery source
- domain/DNS/TLS provisioning source

Container와 CI 관련 representative commits는 존재하지만 이미 Thread 15와 Thread 16의 핵심 commit set으로 확정되어 있다. 이들을 다시 묶어 신규 Thread로 중복 소유시키면 Existing Thread 구조를 재정의하게 된다.

따라서 NEW_THREAD 기준인 독립 lifecycle, source-backed implementation, failure/recovery model, representative commit set을 충족하는 추가 관점은 발견되지 않았다.

---

# Part IV — Project-Level External Steps

## Accepted items

**없음.**

중요한 external step은 모두 자연스러운 Primary Owner를 가진 기존 Thread supplement로 귀속되었다.

## Evidence가 없어 채택하지 않은 후보

| Candidate | Exclusion reason |
|---|---|
| database creation, migration, seed | database client dependency, connection code, schema/migration/seed script, compose service가 없다. |
| secret/credential issuance or CI secret injection | current code/workflow에 secret/token/API key reference가 없다. `SITE_URL`과 content mode는 non-secret configuration이다. |
| OAuth application/provider or redirect URI registration | OAuth integration code/config가 없다. |
| webhook registration | webhook endpoint/consumer/signature verification/config가 없다. |
| external API credential | external API client or credential reference가 없다. |
| object storage, bucket, IAM | storage SDK/config/resource reference가 없다. assets는 repository의 `public` directory에서 image에 복사된다. |
| custom DNS, TLS certificate, domain verification | code는 real public HTTP(S) origin만 요구하며 custom domain/provider/DNS/TLS mechanism을 지정하지 않는다. origin publication itself는 GAP-ES-01/02에 포함했다. |
| Search Console or search engine property registration | sitemap/robots generation은 있지만 provider registration source가 없다. |
| scheduler/cron | scheduled job code/config가 없다. |
| backup/restore | persisted runtime data와 backup/restore script가 없다. |
| analytics/observability account setup | analytics or remote telemetry integration source가 없다. |
| production content/asset replacement | tracked JSON/public files를 바꾸는 source/content change이므로 External-State Gap이 아니다. |
| Vercel project setup | `.vercel` ignore rule만으로 provider use를 입증할 수 없다. |

---

# Final Audit Conclusion

이 repository의 External-State Development Gap은 **production content mode를 활성화하고, real origin에서 실제로 게시하며, temporary validation environments를 실제 browser/container/CI platform state에서 실행하는 과정**에 집중된다.

다만 Git은 다음을 증명하지 않는다.

- production environment values
- external destination ownership or reachability
- actual production image/service deployment
- public origin response
- search crawler/indexing
- exact visual baseline capture environment
- current Lighthouse raw runs
- GitHub Actions run status or artifact existence

따라서 후속 문서는 "실제로 수행된 역사"가 아니라 각 Existing Thread의 **conceptual execution order, required external state, verification boundary, unobserved execution**을 보완해야 한다.
