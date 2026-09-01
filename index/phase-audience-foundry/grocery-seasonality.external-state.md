# External-State Development Gap Audit

- Repository: `seungwoo7050/audience-foundry-grocery-seasonality`
- Audited branch / HEAD: `main` / `5c0707011a5e74f33cce310510fbf920093f364d`
- Existing Thread Index: 사용자가 첨부한 22개 확정 Thread 체계
- Audit purpose: Git/source 중심 학습에서 빠질 수 있는 실제 시스템 성립 과정의 최소 보완

## Audit Scope and Evidence Rules

이번 감사에서는 기존 Thread별 학습 문서나 해설서를 사용하지 않았다. 첨부된 Thread Index는 Thread identity와 ownership 판정에만 사용했고, repository 쪽에서는 다음을 확인했다.

1. GitHub API로 `main`의 전체 commit pagination을 끝까지 열람했다.
2. HEAD의 recursive tree 전체를 확인했다.
3. 현재 migration `0001`~`0028`, source/configuration, management command, Docker Compose, Makefile, runtime settings, security/health/observability code, backup/restore script, 운영·배포 문서를 대조했다.
4. External-State 관련 대표 commit은 개별 diff를 다시 확인했다.
5. repository가 보존한 local completion report나 receipt는 “Git 안에 그러한 실행 기록이 있다”는 증거로만 취급했다. 현재 외부 시스템에 실제 상태가 남아 있는지는 별도로 추정하지 않았다.
6. production 수행 여부는 README의 `production service 미배포`, all-unchecked production checklist, local-only completion-report boundary를 우선했다.

이 보고서에서 사용하는 경계는 다음과 같다.

- **Directly observed in repository**: source, config, migration, command, test, committed receipt/report 또는 commit diff에서 직접 확인되는 사실
- **Required/inferred from repository**: 그 구현을 실제 환경에서 성립시키기 위해 필수임이 repository로부터 확인되는 외부 단계
- **Actual execution not observable from Git**: 실제 account/resource/value/time/operator/result를 repository만으로 확인할 수 없는 부분

---

# Part I — Gap Index

## ESG-01 — Production PostgreSQL 및 역할별 접근 기반

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 22 — 릴리스·복구·배포 관문 / Release, Recovery, and Deployment Gates
- **Related Threads:** 01, 05, 09, 11, 12, 19
- **Repository Evidence 요약:** Django는 `DATABASE_URL` 기반 PostgreSQL만 사용한다. `compose.yaml`은 loopback의 고정 local PostgreSQL과 local credential만 만든다. production checklist는 private managed PostgreSQL, TLS `verify-full` 동등 검증, web·migration·ingestion·reviewer·publisher·backup/restore 역할별 credential·grant 분리를 요구한다.
- **Required External Step 요약:** managed PostgreSQL instance와 database를 만들고, TLS/CA/hostname을 검증하며, 역할별 계정·grant·credential을 발급하고 각 runtime/job에 최소 권한 `DATABASE_URL`을 주입한다.
- **실제 수행 여부 확인 가능성:** local Compose DB와 local/disposable migration·restore 기록은 Git에 있다. production database·role·grant·credential 생성은 확인되지 않으며 repository는 production 미배포를 명시한다.
- **Documentation Action:** Thread 22에 “Production Database Provisioning and Role Matrix” supplement 추가.

## ESG-02 — Immutable release artifact와 static 배포

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 22
- **Related Threads:** 01, 14, 21
- **Repository Evidence 요약:** Makefile은 exact release gate, locked runtime sync, `collectstatic`, production-like process를 제공한다. README와 deployment checklist는 실제 artifact packaging, notice bundle, checksum, vendor upload/release/traffic command를 사람 checkpoint로 남긴다.
- **Required External Step 요약:** clean exact-SHA checkout에서 dependency와 hashed static을 빌드하고, allowlist·license notice·checksum을 고정한 immutable artifact를 platform/registry에 업로드해 release object를 만든다.
- **실제 수행 여부 확인 가능성:** local `collectstatic`과 production-candidate build 기록은 있다. production artifact ID, upload, vendor release 생성은 확인되지 않는다.
- **Documentation Action:** Thread 22 supplement에 artifact provenance와 static deployment evidence 항목 추가.

## ESG-03 — Production forward migration 적용

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 22
- **Related Threads:** 01, 02, 05, 08, 09, 11, 12
- **Repository Evidence 요약:** repository에는 `grocery` migration `0001`~`0028`이 있으며 readiness는 schema가 leaf migration까지 최신인지 확인한다. production checklist는 backup/PITR checkpoint, 복제 DB preflight, migration role, maintenance window, forward-only 적용과 검증을 요구한다.
- **Required External Step 요약:** production 복제본에서 lock·시간·row 적합성·이전 release 호환성을 측정하고, 승인된 window에 migration role로 `migrate --noinput`을 실행한 뒤 `migrate --check`와 deploy check를 통과시킨다.
- **실제 수행 여부 확인 가능성:** 빈 local/disposable DB와 restore DB의 migration 적용 기록은 있다. production DB에 `0001`~`0028`이 적용되었다는 증거는 없다.
- **Documentation Action:** Thread 22 supplement에 production migration execution record template 추가.

## ESG-04 — KAMIS credential 발급·회전 및 worker-only 주입

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 03 — 감사 가능한 KAMIS 수집 경계 / Auditable KAMIS Acquisition Boundary
- **Related Threads:** 04, 09, 20, 21, 22
- **Repository Evidence 요약:** `KAMIS_API_KEY` 값은 repository에 없고 `.env.example`도 빈 값이다. secret loader는 process environment 또는 owner-only ignored `.env.local`만 읽으며 값·길이·부분 문자열을 노출하지 않는다. Makefile은 ambient key를 child process에서 제거하고 명시적 live-source target만 owner-only key를 읽는다.
- **Required External Step 요약:** provider에서 credential을 발급받아 rotation/revocation owner를 정하고 managed secret store에 넣은 뒤, ingestion worker에만 주입하고 web·CI·artifact·log에서 제외한다. approved KAMIS HTTPS endpoint로의 egress도 허용한다.
- **실제 수행 여부 확인 가능성:** committed completion report는 local 개발 key를 사용한 bounded live gate/smoke를 기록하지만 값은 보존하지 않는다. production key 발급·저장·주입·회전 상태는 확인되지 않는다.
- **Documentation Action:** Thread 03 supplement에 “Credential Lifecycle and Runtime Injection Boundary” 추가.

## ESG-05 — Production SourceConfiguration bootstrap

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 03
- **Related Threads:** 04, 08, 20, 22
- **Repository Evidence 요약:** `grocery/source/configuration.py`는 approved KAMIS source의 dataset/endpoint/rights locator와 hash, logical secret name, request bounds, `PLATFORM_SINGLETON` schedule metadata를 deterministic하게 DB에 bootstrap하거나 drift를 거부한다.
- **Required External Step 요약:** production DB에서 exact release로 bootstrap을 실행하고 생성된 `SourceConfiguration`의 state, rights evidence, endpoint, limits, logical secret reference와 schedule metadata를 검증한다.
- **실제 수행 여부 확인 가능성:** bootstrap code와 deterministic desired state는 직접 보인다. production DB에 실제 row가 생성·검증된 사실은 확인되지 않는다.
- **Documentation Action:** Thread 03 supplement에 source-configuration bootstrap evidence record 추가.

## ESG-06 — Production recent ingestion 실행

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 04 — 결정적 최근 가격 정규화 / Deterministic Recent Price Normalization
- **Related Threads:** 03, 20, 21, 22
- **Repository Evidence 요약:** `ingest_kamis_recent`는 source config bootstrap, external key load, bounded HTTP fetch, artifact/parse generation persistence, typed fact generation과 safe receipt를 하나의 명령으로 연결한다.
- **Required External Step 요약:** production ingestion identity와 DB credential로 명령을 실행하고 FetchAttempt·SourceArtifact·ParseRun·typed fact IDs/hash/count를 보존하며, failed/quarantined/schema-drift 결과를 공개 pipeline과 분리한다.
- **실제 수행 여부 확인 가능성:** repository는 local live-source smoke와 outer-transaction rollback/전용 DB 삭제를 기록한다. production에 지속된 recent generation은 확인되지 않는다.
- **Documentation Action:** Thread 04 supplement에 first-run/recurrent-run production execution packet 추가.

## ESG-07 — Recent production review·seal·activation·rollback

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 05 — 최근 게시본 수명 주기 / Recent Publication Lifecycle
- **Related Threads:** 04, 12, 13, 19, 22
- **Repository Evidence 요약:** fact-set hash, append-only ReviewDecision, immutable revision seal, publication channel, optimistic CAS activation/rollback/withdraw와 DB trigger가 구현되어 있다. production command는 exact release와 control-plane actor를 요구한다.
- **Required External Step 요약:** 사람이 rights/coverage/reconciliation을 검수하고 approval을 append한 뒤 publisher job에서 revision을 seal하고 expected current/version을 사용해 activate한다. authoritative inspection으로 revision/version/fact-set/activation ID를 남기고 필요 시 별도 rollback/withdraw를 수행한다.
- **실제 수행 여부 확인 가능성:** local Phase 0/test publication의 승인·seal·activation 기록은 repository에 있다. production reviewer decision, sealed revision, current pointer와 rollback 실행은 확인되지 않는다.
- **Documentation Action:** Thread 05 supplement에 production publication evidence checklist 추가.

## ESG-08 — Cross-source identity registry bootstrap

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 08 — 교차 소스 정체성과 타입 사실 / Cross-Source Identity and Typed Facts
- **Related Threads:** 07, 09, 11, 16, 17
- **Repository Evidence 요약:** immutable `HistoricalRetailSeriesKey`, `RetailRegionKey`, `RetailMarketKey`와 evidence revision/code-manifest hash가 모델링된다. `load_historical_dimension_registry()`는 series 또는 region identity가 없거나 manifest/evidence revision이 다르면 ingestion을 중단한다.
- **Required External Step 요약:** recent series와 historical APIs의 exact bridge, official region/market leading-zero code와 표시명을 사람이 검토하고, approved evidence revision과 exact code-manifest hash에 결속된 immutable DB rows를 생성한다.
- **실제 수행 여부 확인 가능성:** schema와 fail-closed loader는 직접 보인다. production의 human-reviewed mapping 값·row·승인 시점은 확인되지 않는다.
- **Documentation Action:** Thread 08 supplement에 identity-registry provisioning packet 추가.

## ESG-09 — Historical monthly/regional/market collection 실행

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 09 — 분할형 히스토리 컬렉션 수집 / Partitioned Historical Collection Ingestion
- **Related Threads:** 07, 08, 10, 17, 20, 21
- **Repository Evidence 요약:** monthly, regional daily, market daily 세 management command가 reviewed identity registry와 KAMIS key를 로드해 bounded collection workflow를 실행하고 validated collection을 저장한다. 테스트는 transport/workflow를 mock하거나 synthetic scope를 사용한다.
- **Required External Step 요약:** reviewed production scope로 세 channel을 독립 실행하고 collection/part/fact/receipt IDs, manifest, counts, coverage window와 reconciliation 결과를 보존한다. 자동 review·seal·activate는 연결하지 않는다.
- **실제 수행 여부 확인 가능성:** 최소 live smoke와 test-only 자동 mapping/publication은 기록되어 있지만 full production coverage의 세 collection은 확인되지 않는다.
- **Documentation Action:** Thread 09 supplement에 independent historical collection execution packet 추가.

## ESG-10 — Historical review·bundle·seal·activation·rollback

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 11 — 히스토리 게시본 수명 주기 / Historical Publication Lifecycle
- **Related Threads:** 08, 09, 10, 12, 13, 16, 17, 19, 22
- **Repository Evidence 요약:** monthly/regional/market의 별도 reviewed decision을 묶는 immutable historical revision, authorized append-only review, seal guard, optimistic CAS activation, last-known-good rollback이 구현되어 있다.
- **Required External Step 요약:** 세 collection을 각각 사람이 승인하고 compatibility report를 승인한 뒤 bundle을 seal한다. authoritative current/version/fact-set을 읽고 publisher job에서 activate하며, 문제 시 last-known-good membership을 검증해 rollback/withdraw한다.
- **실제 수행 여부 확인 가능성:** local/test active historical publication과 SSR evidence는 있다. production review tails, sealed bundle, active pointer와 rollback execution은 확인되지 않는다.
- **Documentation Action:** Thread 11 supplement에 production historical publication/rollback packet 추가.

## ESG-11 — External IAM/MFA control-plane jobs와 actor bootstrap

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 12 — 역할 분리형 게시 제어 평면 / Role-Separated Publication Control Plane
- **Related Threads:** 05, 11, 13, 22
- **Repository Evidence 요약:** control-plane module과 command help는 enable flag와 release lock이 authentication이 아니며, production platform이 external MFA/IAM private job과 role-specific DB credential을 제공해야 한다고 명시한다. `bootstrap_control_plane_actors`는 fixed non-login reviewer/publisher actors를 만들거나 검증한다.
- **Required External Step 요약:** private reviewer/publisher/provisioning jobs를 만들고 external IAM/MFA, immutable audit, exact release lock과 최소 DB credential을 결속한다. 승인된 provisioning job에서 actor bootstrap을 1회 실행한다.
- **실제 수행 여부 확인 가능성:** application defense-in-depth와 bootstrap command는 직접 보인다. platform job, IAM policy, MFA, DB job identity, production actor rows와 audit sink는 확인되지 않는다.
- **Documentation Action:** Thread 12 supplement에 external control-plane provisioning packet 추가.

## ESG-12 — Public domain·DNS·TLS·proxy/runtime security 구성

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 14 — 공개 HTTP 보안과 질의 프라이버시 / Public HTTP Security and Query Privacy
- **Related Threads:** 19, 22
- **Repository Evidence 요약:** production settings는 strong `DJANGO_SECRET_KEY`, non-wildcard allowed hosts, HTTPS CSRF origins, SSL redirect/HSTS와 exact release SHA를 요구한다. public response security headers와 admin hiding이 구현되어 있다. forwarded-proto trust는 명시적 option이다.
- **Required External Step 요약:** domain을 선택·등록하고 DNS를 연결하며 certificate를 발급·배치한다. trusted reverse-proxy hop을 검증하고 host/CSRF/TLS/HSTS/forwarded-proto/Django secret 값을 runtime에 주입한다.
- **실제 수행 여부 확인 가능성:** local settings tests와 header behavior는 확인된다. production domain, DNS records, certificate, proxy topology와 injected values는 확인되지 않는다.
- **Documentation Action:** Thread 14 supplement에 public-edge provisioning evidence 추가.

## ESG-13 — Four independent scheduler registrations

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 20 — 수집 스케줄과 부하 한계 / Acquisition Scheduling and Load Envelope
- **Related Threads:** 03, 04, 09, 10, 19, 22
- **Repository Evidence 요약:** source configuration은 schedule execution mode `PLATFORM_SINGLETON`, interval, request/page/byte/retry bounds를 저장하지만 실제 scheduler resource를 만들지 않는다. production checklist는 recent/monthly/regional/market을 독립 singleton job으로 등록하고 overlap과 cadence를 검증하라고 요구한다.
- **Required External Step 요약:** 외부 scheduler에 네 job을 생성하고 recent/regional/market 최대 24시간, monthly 최대 168시간 cadence, no-overlap, retry/timeout, network egress와 최소 DB/secret identity를 구성한다. approve·seal·activate를 자동 연결하지 않는다.
- **실제 수행 여부 확인 가능성:** metadata와 load test는 직접 보인다. scheduler vendor/resource ID, trigger, last run, overlap lock과 alert route는 확인되지 않는다.
- **Documentation Action:** Thread 20 supplement에 scheduler registration and run-state packet 추가.

## ESG-14 — External log sink·health/freshness monitoring·alert routing

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 19 — 안전한 관측성과 자료 신선도 / Safe Observability and Publication Freshness
- **Related Threads:** 14, 20, 22
- **Repository Evidence 요약:** allowlisted structured events, request ID와 query-bearing access-log suppression이 구현되고 logging handler는 JSON을 stdout에 쓴다. `/health/live`, `/health/ready`, `/health/freshness`와 recent freshness command가 있다.
- **Required External Step 요약:** stdout을 privacy-safe external log pipeline에 연결하고 retention/access를 설정한다. load balancer/orchestrator probe와 recent/historical freshness monitor, fixed failure thresholds, alert receiver/on-call route를 만든다.
- **실제 수행 여부 확인 가능성:** event emission과 endpoint behavior의 local tests는 확인된다. external sink, dashboards, probes, alert rules, recipients와 actual incidents는 확인되지 않는다.
- **Documentation Action:** Thread 19 supplement에 observability integration packet 추가.

## ESG-15 — Production encrypted backup/PITR와 restore rehearsal

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 22
- **Related Threads:** 05, 11, 13, 19
- **Repository Evidence 요약:** backup script는 repository의 fixed local Docker Compose DB에 의도적으로 제한된 custom-format dump/isolated restore assurance다. committed report도 이를 production encrypted backup/PITR가 아니라고 명시한다. checklist는 encrypted backup, PITR, retention, failure alert와 RPO 24h/RTO 4h restore를 요구한다.
- **Required External Step 요약:** managed backup/PITR/retention을 켜고 backup role·encryption·failure alert를 구성한다. 새 instance에 복원해 schema, rows, publication chains와 canonical hashes를 검증하고 RPO/RTO evidence를 남긴다.
- **실제 수행 여부 확인 가능성:** local isolated restore rehearsal은 Git report로 확인된다. production backup schedule, recovery points, retention와 restore rehearsal은 확인되지 않는다.
- **Documentation Action:** Thread 22 supplement에 managed recovery evidence packet 추가.

## ESG-16 — Production deploy·traffic switch·rollback·evidence closure

- **Classification:** `EXISTING_THREAD`
- **Primary Owner:** Thread 22
- **Related Threads:** 14, 19, 21
- **Repository Evidence 요약:** README는 public repository이지만 production service가 미배포·미활성이라고 명시한다. production checklist의 platform, runtime, publication, acceptance, traffic, rollback, completion 항목은 모두 미체크 상태다. completion report는 local production candidate일 뿐 실제 production deploy/traffic을 주장하지 않는다.
- **Required External Step 요약:** exact artifact를 traffic 없이 시작하고 health/security/static/data/performance acceptance를 통과시킨 뒤 atomic traffic switch를 실행한다. error/latency/DB/scheduler/freshness alert를 관찰하고 application rollback과 publication rollback을 분리하며 immutable completion evidence를 기록한다.
- **실제 수행 여부 확인 가능성:** local production-like process와 browser/load/recovery evidence는 확인된다. 실제 vendor deployment와 traffic switch는 미수행으로 repository에 명시되어 있으며 외부 상태도 확인할 수 없다.
- **Documentation Action:** Thread 22 supplement의 최종 production execution/closure section으로 소유.

## ESG-17 — Approved remote의 required CI/status checks

- **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
- **Primary Owner:** Project level
- **Related Threads:** 01, 21, 22
- **Repository Evidence 요약:** deployment checklist는 approved remote가 exact `RELEASE_SHA`를 포함하고 해당 SHA의 required CI가 통과해야 한다고 요구한다. 현재 recursive tree에는 in-repository `.github/workflows` configuration이 없다.
- **Required External Step 요약:** approved remote/branch rule/status-check policy를 정하고 required CI를 실행해 exact release SHA에 결속된 result locator를 남긴다.
- **실제 수행 여부 확인 가능성:** 외부 branch protection, GitHub settings, 다른 CI provider와 실제 status result는 repository contents만으로 확인할 수 없다. workflow 파일 부재가 “CI가 전혀 없음”을 증명하지는 않는다.
- **Documentation Action:** project release checklist/change record에만 유지. 독립 Thread를 만들지 않음.

## ESG-18 — Change record·운영 책임·maintenance/GO/abort 기록

- **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
- **Primary Owner:** Project level
- **Related Threads:** 12, 19, 22
- **Repository Evidence 요약:** production checklist는 `RELEASE_SHA`, `PREVIOUS_RELEASE_SHA`, change record, deploy/reviewer/publisher/on-call 책임자, maintenance window, rollback 판단 시각, 최종 GO/abort와 completion evidence를 요구한다.
- **Required External Step 요약:** 실제 change/ticket을 만들고 역할 담당자와 승인 범위, window, abort 기준, GO/rollback decision, evidence locators를 기록한다.
- **실제 수행 여부 확인 가능성:** template만 repository에 있다. 실제 ticket, 담당자, 승인 시각과 decision은 확인되지 않는다.
- **Documentation Action:** project-level deployment record로 유지. Development Thread로 승격하지 않음.

---

# Part II — Existing Thread Supplement Packets

## Packet T03 — 감사 가능한 KAMIS 수집 경계 / Auditable KAMIS Acquisition Boundary

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 03
- **한국어 제목:** 감사 가능한 KAMIS 수집 경계
- **English title:** Auditable KAMIS Acquisition Boundary

### Gaps

- ESG-04 — KAMIS credential 발급·회전 및 worker-only 주입
- ESG-05 — Production SourceConfiguration bootstrap

### Repository Evidence

1. **`992ac9e215044d833a78f6dac27b8c378f2a79ea` — `feat(source): load local key without disclosure`**
   - Files: `grocery/source/secrets.py`, related tests/Makefile boundary
   - Relevant diff/excerpt:
     - key source precedence is process environment, then ignored owner-only `.env.local`.
     - missing/unsafe file fails closed.
     - wrapped secret does not expose value, length, representation or equality detail.
   - Importance: repository deliberately models only the secret loading boundary, not credential issuance or value.

2. **`36a89b67c43712af21b1647d61838da1f093082e` — `feat(source): fetch kamis within strict bounds`**
   - Files: KAMIS source client and tests
   - Relevant diff/excerpt:
     - HTTPS KAMIS endpoint and `service_key` input are required.
     - timeout/retry/page/byte limits and redacted request evidence are enforced.
   - Importance: valid external credential and approved network egress are runtime prerequisites.

3. **Final state: `grocery/source/configuration.py`**
   - Excerpt: module contains only public configuration metadata and the logical credential name; it does not invoke the credential loader.
   - `bootstrap_kamis_source_configuration()` creates the deterministic DB row or rejects drift in dataset, endpoint, rights evidence, bounds and schedule.

4. **Final state: `.env.example`, Makefile**
   - `KAMIS_API_KEY=` remains blank.
   - ambient KAMIS key is unexported; explicit live source target reads ignored owner-only material only in the worker process.

### External Development Steps

1. Obtain an authorized KAMIS/API portal credential without copying it into Git, command arguments, URLs, fixtures, receipts or logs.
2. Assign a rotation/revocation owner and store the value in a managed secret store for production.
3. Bind the secret only to ingestion identities; do not expose it to the public web process or generic CI/build jobs.
4. Permit outbound HTTPS only to the approved source endpoints and validate TLS/network policy.
5. Run the deterministic source-configuration bootstrap in production and inspect the resulting row for exact desired-state equality.
6. Record only immutable non-secret locators, release SHA, source configuration ID/state and safe receipt hashes.

### Code Connection

- `grocery/source/secrets.py` cannot make a provider credential exist; it only consumes one safely.
- the KAMIS client cannot pass authentication or reach the provider without credential injection and network egress.
- `ingest_kamis_recent` and all three historical ingest commands load this boundary before external fetch.
- source configuration gates acquisition; if its state, rights metadata or limits drift, production ingestion should fail before publication.

### Evidence Boundary

- **Directly observed in repository:** blank env placeholder, fail-closed loader, logical secret name, bounded client, deterministic configuration bootstrap, local report claiming a bounded live gate.
- **Required/inferred from repository:** provider credential issuance, secret-store object, rotation/revocation process, worker-only binding, egress allowlist, execution of production bootstrap.
- **Actual execution not observable from Git:** secret value, current validity, portal account, production secret resource ID, IAM binding, production DB row and the time/operator of bootstrap.

### Ordering

**Conceptual execution order:** credential issuance → managed secret creation → ingestion identity/egress binding → production source configuration bootstrap → bounded connectivity smoke → scheduled/explicit ingestion. This is not asserted as historical production chronology.

---

## Packet T04 — 결정적 최근 가격 정규화 / Deterministic Recent Price Normalization

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 04
- **한국어 제목:** 결정적 최근 가격 정규화
- **English title:** Deterministic Recent Price Normalization

### Gaps

- ESG-06 — Production recent ingestion 실행

### Repository Evidence

1. **`3cd94a72290f444d7c884799e22240cfb03661c5` — `feat(source): orchestrate recent ingestion`**
   - Files: `grocery/management/commands/ingest_kamis_recent.py`, ingestion tests and persistence helpers
   - Relevant diff/excerpt:
     - bootstrap source configuration;
     - load KAMIS key;
     - start bounded fetch and persist attempt/artifact;
     - parse and generate typed recent facts;
     - return a safe count/hash receipt.
   - Importance: code defines the transaction/audit chain, but the persistent external run must still be launched against a real DB and source.

2. **Final state: README/Makefile live-source smoke**
   - creates an exact disposable loopback DB;
   - calls real APIs under explicit owner action;
   - runs test-only mapping/review/publication;
   - rolls back writes and deletes the target DB.
   - Importance: it is connection evidence, not a durable production generation.

### External Development Steps

1. Provide the ingestion job with production DB credential, KAMIS secret and exact release SHA.
2. Launch `ingest_kamis_recent` explicitly or through the approved singleton scheduler.
3. Capture safe identifiers and hashes for FetchAttempt, SourceArtifact, ParseRun and generated fact set.
4. Inspect count, scope, schema and quarantine status before any human review.
5. Preserve failed attempts and diagnostic codes without leaking query or payload.

### Code Connection

- public reads never call KAMIS; therefore a durable production ingestion run is the only way new recent facts reach review/publication.
- normalized fact/replay determinism is useful only after an external fetch and database transaction actually occur.
- scheduling and publication are intentionally separate subsystems, so a successful run does not publish automatically.

### Evidence Boundary

- **Directly observed in repository:** complete orchestration command, test coverage, local live-source smoke report and rollback/delete boundary.
- **Required/inferred from repository:** running the command in the production worker and preserving the resulting database audit chain.
- **Actual execution not observable from Git:** production attempt/artifact/parse IDs, row counts, source response, persistent facts, job time and operator.

### Ordering

**Conceptual execution order:** prerequisites from T03 → launch fetch → persist artifact and parse run → normalize typed facts → inspect safe receipt → hand generation to T05 review. Not claimed as production history.

---

## Packet T05 — 최근 게시본 수명 주기 / Recent Publication Lifecycle

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 05
- **한국어 제목:** 최근 게시본 수명 주기
- **English title:** Recent Publication Lifecycle

### Gaps

- ESG-07 — Recent production review·seal·activation·rollback

### Repository Evidence

1. **`3eea8938331ee0218c20547dc8857da64974a527` — `feat(publication): hash canonical retail facts`**
   - Files: publication-fact canonicalization and tests
   - Relevant diff/excerpt: exact typed facts are serialized and hashed as an immutable publication fact set.
   - Importance: external approval/activation must reference the exact canonical set, not mutable rows by convention.

2. **`97ffb8cc8ed2aa84f61f43d716c8b1a2e7b8af15` — `feat(review): append generation decisions`**
   - Files: review model/migration/services/tests
   - Relevant diff/excerpt: append-only review requires active source/rights evidence, completed validated parse and authorized actor; evidence commitments are stored instead of private material.
   - Importance: an actual human decision is external state represented by a new immutable DB row.

3. **`002d910488347af28e8e9fa988fb36ceea7ba2bc` — `feat(publication): activate revisions atomically`**
   - Files: publication activation migration/models/services/tests
   - Relevant diff/excerpt: DB guard requires a transition capability; channel bootstrap is empty/version 0; activation mutates the pointer only through authorized transition logic.
   - Importance: actual public state changes when the database pointer is transitioned, not when code is merged.

4. **Production commands in final state**
   - `approve_recent_generation`, `seal_recent_publication`, `inspect_recent_publication`, `transition_recent_publication`.
   - exact release and control-plane actor are required; production help text states external MFA is required.

### External Development Steps

1. Reviewer examines source rights, scope, mapping, unit, coverage, reconciliation and quarantine state.
2. Reviewer job appends an approval/rejection decision tied to the exact parse/fact evidence.
3. Publisher job seals the approved revision and records public-copy revision/evidence.
4. Authoritative inspection reads current revision/version/fact-set.
5. Publisher job performs optimistic CAS activation using exact expected current/version and release SHA.
6. On data/publication fault, perform a separate rollback or withdraw with explicit target and expected state.

### Code Connection

- public views resolve only the active `RECENT_RETAIL` pointer.
- a generated fact set is invisible publicly until external review, seal and pointer transition occur.
- freshness/health behavior depends on the active publication state and dates.

### Evidence Boundary

- **Directly observed in repository:** append-only decision model, canonical hash, seal/activation/rollback commands and DB guards; committed local Phase 0 publication evidence.
- **Required/inferred from repository:** human review, private evidence examination, reviewer/publisher job execution and recording exact production IDs.
- **Actual execution not observable from Git:** production decision/revision/activation IDs, active pointer/version, reviewer/publisher identity, approval evidence and rollback history.

### Ordering

**Conceptual execution order:** inspect generation → human review → append decision → seal revision → authoritative inspect → CAS activate → monitor → rollback/withdraw if necessary.

---

## Packet T08 — 교차 소스 정체성과 타입 사실 / Cross-Source Identity and Typed Facts

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 08
- **한국어 제목:** 교차 소스 정체성과 타입 사실
- **English title:** Cross-Source Identity and Typed Facts

### Gaps

- ESG-08 — Cross-source identity registry bootstrap

### Repository Evidence

1. **`82d58e1bfb696a8ed8083ff400dd173655d73009` — `feat(history): identify exact retail sources`**
   - Files: historical identity models, migration and tests
   - Relevant diff/excerpt:
     - immutable bridge from historical series identity to recent series;
     - official region/market codes and names;
     - evidence revision and exact code-manifest SHA-256.
   - Importance: repository models reviewed identity facts but does not select production mappings automatically.

2. **Final state: `grocery/historical_registry.py`**
   - `HistoricalRetailSeriesKey` rows must exist and all match the supplied code-manifest hash.
   - region rows must exist; region and market evidence revisions must be exactly one consistent revision.
   - any drift raises validation before historical ingestion.

### External Development Steps

1. Obtain the approved official code/manifests for the exact API revision.
2. Human-review recent-to-historical series identity, unit and coverage compatibility.
3. Human-review official region and market codes/names, preserving leading zeros.
4. Generate and retain the code-manifest hash and evidence revision.
5. Insert immutable production registry rows through an approved provisioning path.
6. Re-run the loader with the exact release manifest and stop on any mismatch.

### Code Connection

- all historical parsers/generation depend on the loaded registry.
- missing or mismatched rows prevent collection ingestion, making registry creation a required external precursor.
- the active recent series identity is the cross-source anchor used by public historical routes.

### Evidence Boundary

- **Directly observed in repository:** schema, constraints, loader and fail-closed validation.
- **Required/inferred from repository:** human mapping process, evidence acquisition, manifest generation and production row insertion.
- **Actual execution not observable from Git:** full production mapping values, approval artifacts, DB row IDs, evidence reviewer, time and current manifest binding.

### Ordering

**Conceptual execution order:** acquire official manifests → review series/region/market identities → freeze evidence revision/hash → provision registry rows → validate loader → permit T09 ingestion.

---

## Packet T09 — 분할형 히스토리 컬렉션 수집 / Partitioned Historical Collection Ingestion

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 09
- **한국어 제목:** 분할형 히스토리 컬렉션 수집
- **English title:** Partitioned Historical Collection Ingestion

### Gaps

- ESG-09 — Historical monthly/regional/market collection 실행

### Repository Evidence

1. **`8dec9c8ceb5b575a97c4b9749317d0810f8e9bed` — `feat(history): expose three bounded ingest commands`**
   - Files: `ingest_kamis_monthly.py`, `ingest_kamis_regional_daily.py`, `ingest_kamis_market_daily.py`, shared management workflow/tests
   - Relevant diff/excerpt:
     - each command loads reviewed identity registry and source credential;
     - binds one dataset mode/scope;
     - executes bounded fetch/parse/persistence;
     - emits a fixed safe receipt.
   - Importance: three collections are separate external executions, not one implicit history import.

2. **Final-state command tree and checklist**
   - all three commands are present independently.
   - production checklist requires independent receipt/audit and rejects incomplete/quarantined/schema-mismatch collections.

### External Development Steps

1. Freeze the reviewed identity registry and approved collection scope.
2. Run monthly, regional-daily and market-daily commands as independent jobs/runs.
3. Preserve each collection ID, part membership, source artifact/parse metadata, manifest and validation receipt.
4. Reconcile expected vs completed partitions and inspect quarantine/schema failures.
5. Do not automatically approve, bundle or activate any collection.

### Code Connection

- monthly charts, regional ledgers and market ledgers read only approved historical publication facts produced from these collections.
- collection integrity/concurrency guards operate on database state created by the external ingestion runs.
- failure in one channel must not manufacture success in another channel.

### Evidence Boundary

- **Directly observed in repository:** separate commands, bounded workflows, tests and local live minimum smoke.
- **Required/inferred from repository:** production execution with reviewed full scope and persistent collection/audit rows.
- **Actual execution not observable from Git:** production collection IDs, full provider coverage, part counts, run times, failures/retries and reconciliation results.

### Ordering

**Conceptual execution order:** T08 registry ready → schedule/manual launch each channel → persist/reconcile each collection → independent human reviews → T11 compatibility/bundle.

---

## Packet T11 — 히스토리 게시본 수명 주기 / Historical Publication Lifecycle

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 11
- **한국어 제목:** 히스토리 게시본 수명 주기
- **English title:** Historical Publication Lifecycle

### Gaps

- ESG-10 — Historical review·bundle·seal·activation·rollback

### Repository Evidence

1. **`94fa78121e02a9c31f9b54d4f20494f7203e5f56` — `feat(history): define publication bundles`**
   - Files: historical publication revision model/migration/services/tests
   - Relevant diff/excerpt: one immutable revision binds separate monthly, regional and market reviewed decisions plus exact hashes/counts/windows.

2. **`e57d6953e631b54a698138d894eb20bfe728d2b5` — `fix(history): authorize review decisions`**
   - Files: historical review service and DB triggers
   - Relevant diff/excerpt: append-only review mutations require authorized service capability/actor; update/delete are prevented.

3. **`18116a50ba6eba31ee43869f59f6a2104b703070` — `fix(history): activate publications through CAS`**
   - Files: historical activation service/migration/tests
   - Relevant diff/excerpt: activate requires authorized publisher, sealed approved bundle, expected current/version and activation evidence.

4. **`dd307fb6d7f6c0c196ae7638253ee90410585728` — `fix(history): preserve last-known-good rollback`**
   - Relevant diff/excerpt: previous known-good target can be rolled back even when current review tails advanced, while ordinary activation remains strict.

### External Development Steps

1. Review monthly, regional and market collections separately and append three decisions.
2. Produce and approve a compatibility report over identity, coverage, units, windows and fact hashes.
3. Build and seal one historical revision referencing the exact three decisions.
4. Perform authoritative repeatable-read inspection of current/version/fact-set.
5. Activate by optimistic CAS through the external publisher job.
6. Monitor historical freshness/read health.
7. On incident, validate last-known-good membership and perform rollback/withdraw as a separate change.

### Code Connection

- public historical reads depend on the active `HISTORICAL_RETAIL` revision.
- the three channels remain independently acquired/reviewed but become public atomically through the bundle pointer.
- rollback rules preserve immutable history and avoid editing source facts or prior revisions.

### Evidence Boundary

- **Directly observed in repository:** bundle schema, authorized review/seal/activation guards, CAS and last-known-good rollback implementation; local/test SSR evidence.
- **Required/inferred from repository:** actual human review/compatibility approval and private publisher job execution.
- **Actual execution not observable from Git:** production decisions, compatibility report locator, revision/seal/activation IDs, active pointer/version and rollback/withdraw events.

### Ordering

**Conceptual execution order:** three independent reviews → compatibility approval → bundle build/seal → authoritative inspect → CAS activate → monitor → last-known-good rollback/withdraw when approved.

---

## Packet T12 — 역할 분리형 게시 제어 평면 / Role-Separated Publication Control Plane

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 12
- **한국어 제목:** 역할 분리형 게시 제어 평면
- **English title:** Role-Separated Publication Control Plane

### Gaps

- ESG-11 — External IAM/MFA control-plane jobs와 actor bootstrap

### Repository Evidence

1. **`35153e168a5304777a1aacc173d5adb520f8d231` — `feat(ops): gate production publication commands`**
   - Files: `.env.example`, settings, production commands, `grocery/management/control_plane.py`, actor bootstrap command/tests
   - Relevant diff/excerpt:
     - `CONTROL_PLANE_OPERATIONS_ENABLED=0` by default;
     - exact expected release SHA required;
     - module states it is not an authentication boundary;
     - production platform must provide external MFA/IAM and separate role-specific DB credentials;
     - command help repeats that the flag is not authentication.

2. **`ce204525902e359a89dbcd61f2fe4e898d369076` — `feat(review): authorize local phase zero publication`**
   - Files: local-only operator/bootstrap and local publication commands
   - Importance: demonstrates a deliberate local rehearsal path, not a production IAM solution.

3. **Final state: `bootstrap_control_plane_actors.py`, `control_plane.py`**
   - fixed non-login reviewer/publisher actors and explicit permissions.
   - external platform DB grant matrix remains a production checkpoint.

### External Development Steps

1. Create private provisioning, reviewer and publisher job surfaces inaccessible to the public web.
2. Enforce external IAM and MFA for human invocation.
3. Attach separate minimum DB credentials/grants to migration, ingestion, reviewer, publisher and backup jobs.
4. Bind jobs to exact release SHA and enable control-plane flag only inside approved control jobs.
5. Send invocation identity/result to immutable audit storage.
6. Run actor bootstrap once and validate fixed actor IDs/permissions.

### Code Connection

- application permissions are defense in depth; without external authentication and DB grants, production role separation is incomplete.
- all recent/historical approval, seal, activation and rollback commands call this boundary.
- public web should keep control-plane disabled and should never possess publisher credentials.

### Evidence Boundary

- **Directly observed in repository:** fixed actor model, permission checks, default-off flag, release lock and explicit external-auth boundary statement.
- **Required/inferred from repository:** platform jobs, IAM/MFA policies, job identities, role credentials/grants and immutable audit sink.
- **Actual execution not observable from Git:** cloud/platform job resources, principals, MFA rules, grant SQL, production actor rows, invocation history and auditors.

### Ordering

**Conceptual execution order:** provision DB roles/grants → create IAM/MFA jobs → bind exact release and credentials → bootstrap actors → execute reviewer/publisher operations → retain audit.

---

## Packet T14 — 공개 HTTP 보안과 질의 프라이버시 / Public HTTP Security and Query Privacy

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 14
- **한국어 제목:** 공개 HTTP 보안과 질의 프라이버시
- **English title:** Public HTTP Security and Query Privacy

### Gaps

- ESG-12 — Public domain·DNS·TLS·proxy/runtime security 구성

### Repository Evidence

1. **`c1e0e76492538dee075cb51fc5d018018449e501` — `feat(security): enforce public response boundaries`**
   - Files: public security middleware/helpers, routes and tests
   - Relevant diff/excerpt: fixed CSP/no-referrer/nosniff/frame/COOP/CORP/permissions policy, generic admin 404 and no user-specific public state.

2. **Final state: `config/settings.py`, `.env.example`**
   - production rejects weak/missing secret, wildcard/empty hosts and non-HTTPS/empty CSRF origin.
   - secure cookies, SSL redirect, HSTS and optional forwarded-proto trust are runtime settings.

3. **Production checklist**
   - requires approved domain/DNS/certificate, trusted proxy-hop decision and HSTS scope approval.

### External Development Steps

1. Select and register the production domain.
2. Create DNS records and complete any required domain verification.
3. Issue, renew and deploy TLS certificate/chain.
4. Configure the reverse proxy/load balancer and prove clients cannot spoof trusted forwarded-proto headers.
5. Generate/store a strong Django secret and inject exact allowed hosts/HTTPS CSRF origins/SSL/HSTS settings.
6. Validate headers, no-cookie/no-referrer behavior and generic admin response through the real edge.

### Code Connection

- production settings fail startup when required external values are absent or unsafe.
- correct `is_secure()` behavior, redirects, CSRF and HSTS depend on real TLS termination and proxy trust.
- public security tests at loopback do not prove DNS/certificate/edge behavior.

### Evidence Boundary

- **Directly observed in repository:** fail-closed settings and response-security implementation/tests.
- **Required/inferred from repository:** domain/DNS/certificate/proxy resources and runtime secret/config injection.
- **Actual execution not observable from Git:** domain owner, DNS values, certificate serial/expiry, proxy topology, HSTS decision and production environment values.

### Ordering

**Conceptual execution order:** approve domain/edge topology → DNS/certificate → secret/runtime configuration → no-traffic edge validation → public acceptance → traffic switch.

---

## Packet T19 — 안전한 관측성과 자료 신선도 / Safe Observability and Publication Freshness

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 19
- **한국어 제목:** 안전한 관측성과 자료 신선도
- **English title:** Safe Observability and Publication Freshness

### Gaps

- ESG-14 — External log sink·health/freshness monitoring·alert routing

### Repository Evidence

1. **`cc36f566fe7a56357f73c03b54cbbb2243c742fe` — `feat(ops): add redacted structured events`**
   - Files: `grocery/observability.py`, tests
   - Relevant diff/excerpt: allowlisted event schema forbids raw payload, query, secret and user-input fields.

2. **`dbd60224d47d3e1ed257810e4efda28a9bc25d4e` — `feat(ops): wire safe request logging`**
   - Files: settings/middleware/tests
   - Relevant diff/excerpt: request ID is attached; structured JSON handler writes to stdout; query-bearing generic access logs are disabled.
   - Importance: repository stops at safe emission, not external collection.

3. **Final state: `grocery/health.py`**
   - `/health/live` checks bounded process response.
   - `/health/ready` checks DB, migration currency and a readable sealed publication.
   - `/health/freshness` returns fixed CURRENT/STALE/UNAVAILABLE states and 503 on stale/unavailable.

4. **Production checklist**
   - requires log pipeline, liveness/recent readiness/freshness, historical freshness monitor and alert receiver verification.

### External Development Steps

1. Route stdout to a privacy-safe centralized sink without ingress query/IP/User-Agent/search terms.
2. Set retention, access control and immutable evidence locators.
3. Configure platform liveness/readiness probes against the exact health paths.
4. Configure recent and historical freshness checks with fixed thresholds.
5. Route acquisition/freshness/error alerts to a named on-call destination and verify delivery.
6. Observe the deployment/traffic window and preserve non-secret alert/test evidence.

### Code Connection

- logs do not leave stdout without platform integration.
- health endpoints do not become probes or alerts without external registration.
- readiness/freshness is a release and traffic gate in Thread 22.

### Evidence Boundary

- **Directly observed in repository:** safe event schema, stdout handler, request-ID behavior, health endpoints and tests.
- **Required/inferred from repository:** log agent/sink, monitor resources, alert rules, routing and on-call process.
- **Actual execution not observable from Git:** sink/account IDs, retained logs, probe settings, alert recipients, test-delivery result and incident history.

### Ordering

**Conceptual execution order:** create safe sink → attach app stdout → register probes/freshness checks → configure alerts/on-call → test delivery → use during no-traffic acceptance and traffic observation.

---

## Packet T20 — 수집 스케줄과 부하 한계 / Acquisition Scheduling and Load Envelope

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 20
- **한국어 제목:** 수집 스케줄과 부하 한계
- **English title:** Acquisition Scheduling and Load Envelope

### Gaps

- ESG-13 — Four independent scheduler registrations

### Repository Evidence

1. **`a9645d08f0562c52632d0739813fd9fb31dc77bf` — `feat(source): record bounded ingestion schedule`**
   - Files: source configuration model/migration/tests
   - Relevant diff/excerpt: schedule execution mode `PLATFORM_SINGLETON`, interval and retry/request/page/byte bounds are persisted.
   - Importance: data describes a platform job contract; it does not create one.

2. **`f4758244ef6d8ce0acd09c03bfc4056c3778a3ea` — `test(perf): add bounded phase zero profile`**
   - Files: load-profile script/tests/docs
   - Relevant diff/excerpt: fixed 900-second/10-RPS/20-logical-user local envelope.
   - Importance: capacity evidence does not register production acquisition jobs.

3. **Final state: production checklist**
   - requires independent recent/monthly/regional/market singleton jobs, cadence bounds, overlap prevention and no automatic publication transition.

### External Development Steps

1. Choose the platform scheduler and create four independent job resources.
2. Bind each job to the correct command, exact artifact/release, worker identity, DB role and KAMIS secret.
3. Enforce singleton/no-overlap semantics and approved cadence.
4. Configure timeout/retry/failure handling within source bounds.
5. Ensure jobs stop after acquisition; review/seal/activation remain private human control-plane actions.
6. Connect run failure and freshness signals to Thread 19 monitoring.

### Code Connection

- `PLATFORM_SINGLETON` is metadata interpreted by operators/platform; no scheduler loop exists in the app.
- actual job runs create the data consumed by T04/T09.
- overlapping or over-frequent jobs could violate quota, collection integrity and publication freshness assumptions.

### Evidence Boundary

- **Directly observed in repository:** schedule contract fields, bounds, load profile and production checklist.
- **Required/inferred from repository:** scheduler resources, job identities, triggers, locks, retries and alert bindings.
- **Actual execution not observable from Git:** vendor, job IDs, cron/trigger values, next/last run, overlap prevention result and production load behavior.

### Ordering

**Conceptual execution order:** provision worker identities/secrets → create four jobs → set cadence/singleton/retry → test manual run → enable schedule → monitor failure/freshness.

---

## Packet T22 — 릴리스·복구·배포 관문 / Release, Recovery, and Deployment Gates

### Thread Identity

- **Type:** Existing Thread
- **Thread:** 22
- **한국어 제목:** 릴리스·복구·배포 관문
- **English title:** Release, Recovery, and Deployment Gates

### Gaps

- ESG-01 — Production PostgreSQL 및 역할별 접근 기반
- ESG-02 — Immutable release artifact와 static 배포
- ESG-03 — Production forward migration 적용
- ESG-15 — Production encrypted backup/PITR와 restore rehearsal
- ESG-16 — Production deploy·traffic switch·rollback·evidence closure

### Repository Evidence

1. **`148cd294b4e9884b874528af3670b3e10753fd64` — `docs: define phase zero release gate`**
   - Files: release/operations documentation
   - Importance: establishes release gate as an explicit development concern rather than an implicit final command.

2. **`5062903a15f6373e18e5c35759c9cecaebbcf90f` — `feat(ops): rehearse postgres recovery`**
   - Files: `scripts/postgres_backup_restore.py`, tests/docs
   - Relevant diff/excerpt: local PostgreSQL custom-format backup and isolated restore assurance bound to the repository’s fixed Docker Compose DB.

3. **`4fb41059a40c2cbd7b916bd3cf8daea5fab5cfae` — `fix(ops): harden postgres recovery boundaries`**
   - Relevant diff/excerpt: identifies and pins the exact local container/target, requires manifest evidence and limits cleanup to the invocation-created disposable target.
   - Importance: strong local recovery evidence, explicitly not managed production PITR.

4. **`5c0707011a5e74f33cce310510fbf920093f364d` — `docs(ops): add production deployment checklist`**
   - Files: `docs/PRODUCTION-DEPLOYMENT-CHECKLIST.md`, README reference
   - Relevant diff/excerpt: one ordered all-unchecked checklist for platform/DB/TLS/roles/secrets/domain/IAM/logging/monitoring/backup, exact release/build/migration, scheduler, production data publication, traffic acceptance, rollback and closure.
   - Importance: repository itself marks these as human-approved production gates rather than completed code changes.

5. **Final state: `compose.yaml`, `.env.example`, `config/settings.py`, Makefile**
   - local loopback DB is reproducible but local-only.
   - production startup is fail-closed on secret/host/CSRF/release/database requirements.
   - exact release build/migration/serve commands exist but do not create vendor resources.

6. **Final state: migrations `0001`~`0028`**
   - schema artifacts exist for all implemented subsystems.
   - presence of migration files does not prove production application.

7. **Final state: README and completion report**
   - repository explicitly says production service is not deployed/active.
   - local candidate includes migration, collectstatic, health/browser/load and isolated restore evidence.
   - local report explicitly says the local dump is not production encrypted backup/PITR and vendor artifact/upload/traffic/rollback remain human checkpoints.

### External Development Steps

#### A. Production database and roles

1. Approve compatible platform and private managed PostgreSQL.
2. Configure TLS hostname/CA verification.
3. Create web, migration, ingestion, reviewer, publisher and backup/restore roles with least-privilege grants.
4. Issue/store credentials and bind each to the correct process/job.

#### B. Immutable artifact

1. Check out clean exact `RELEASE_SHA` and pass locked verification.
2. Build runtime and `collectstatic` output.
3. Package only allowlisted code/template/migration/lock/static/license/release evidence.
4. Record checksum/size/notices and upload to the approved platform/registry.

#### C. Migration

1. Take/verify managed backup/PITR checkpoint.
2. Rehearse `0001`~`0028` on a production clone and verify previous-release read compatibility.
3. Run forward migration with migration role in the maintenance window.
4. Verify migration currency and deploy checks without opening traffic.

#### D. Recovery

1. Configure encrypted scheduled backup, PITR, retention and failure alerts.
2. Restore into a new isolated instance.
3. Validate migrations, row counts, active publication chains and canonical payload/hash.
4. Prove RPO 24h and RTO 4h, then retain immutable non-secret evidence.

#### E. Deploy, traffic and closure

1. Start exact artifact/Gunicorn with no traffic.
2. Inject web runtime settings while keeping KAMIS and control plane out of public web.
3. Complete production data review/seal/activation through T03–T12 packets.
4. Run health, public SSR, static, security, accessibility, load and historical acceptance.
5. Execute approved atomic traffic switch.
6. Observe app/DB/scheduler/freshness/error alerts.
7. Roll back application traffic separately from publication rollback when criteria trigger.
8. Record artifact, migration, publication, health, backup, alert and rollback evidence in a follow-up docs commit without changing the deployed SHA.

### Code Connection

- no public route is useful in production without a reachable migrated DB and active publication.
- readiness explicitly checks DB/migrations and publication readability.
- WhiteNoise manifest static and exact release lock depend on a real build/deploy artifact.
- recovery scripts validate data semantics but are intentionally tied to local Compose; managed service configuration is external.
- source push/merge is explicitly not deployment or publication activation.

### Evidence Boundary

- **Directly observed in repository:** local Compose environment, all migration files, release/build commands, fail-closed settings, local backup/restore implementation and committed local candidate evidence; production checklist remains unchecked; README states production un-deployed.
- **Required/inferred from repository:** platform/account/app, managed DB, role grants/credentials, artifact registry/release, production migration, managed backup/PITR, no-traffic deploy, traffic switch, monitoring and rollback evidence.
- **Actual execution not observable from Git:** resource IDs, cloud/vendor, database endpoint, credentials, deployed artifact, migration run, backup schedule/recovery point, traffic state, production publication IDs and final GO/rollback decisions.

### Ordering

**Conceptual execution order:** change approval and owners → platform/DB/roles/secrets/domain/observability/recovery prerequisites → exact build/artifact → backup/preflight → forward migration → no-traffic runtime → source/data/publication operations → acceptance → traffic switch → observation → application/publication rollback if needed → evidence closure. This is an intended execution order, not a claim that production history occurred.

---

# Part III — Proposed New Thread Packets

## 판정: 제안 없음

이번 audit에서는 `NEW_THREAD` 조건을 충족하는 항목을 발견하지 못했다.

### 이유

1. **Production platform provisioning/deployment lifecycle**은 자체 lifecycle과 복구조건을 가지지만 이미 Thread 22가 release, recovery, deployment gate를 명시적으로 소유하고 있으며 `148cd294`, `5062903a`, `4fb41059`, `5c070701` 등 충분한 대표 commit이 기존 Thread에 배정되어 있다. 새 Thread를 만들면 기존 확정 관점을 중복한다.
2. **Secret lifecycle**은 하나의 독립 구현군이라기보다 KAMIS credential은 Thread 03, public runtime secret/edge config는 Thread 14, DB/platform secret provisioning은 Thread 22의 외부 단계로 자연스럽게 분리된다. repository에 이들을 통합하는 별도 rotation service나 독립 commit cluster가 없다.
3. **External scheduler**는 Thread 20의 제목·모델·commit이 직접 소유한다.
4. **External IAM/MFA control plane**은 Thread 12가 명시적으로 소유한다.
5. **External observability integration**은 Thread 19의 안전한 event/health/freshness 관점을 실제 환경에서 완성하는 단계다.
6. CI status/branch governance와 change-management record는 중요하지만 독립 subsystem 구현과 representative commit 집합이 부족해 project-level step으로 남겼다.

따라서 Part III에는 representative commits를 가진 신규 Packet이 없다.

---

# Part IV — Project-Level External Steps

## ESG-17 — Approved remote required CI/status checks

### Repository Evidence

- `docs/PRODUCTION-DEPLOYMENT-CHECKLIST.md`의 exact release gate는 approved remote가 `RELEASE_SHA`를 포함하고 해당 SHA의 required CI가 통과할 것을 요구한다.
- HEAD의 recursive tree에는 in-repository `.github/workflows` 경로가 없다.

### Required External Step

1. approved remote와 protected branch/status policy를 정한다.
2. exact `RELEASE_SHA`에 대해 required CI를 실행한다.
3. required checks의 성공 locator를 change record에 고정한다.
4. branch protection과 CI provider가 repository 밖에서 관리된다면 그 configuration evidence도 별도로 보존한다.

### Code Connection

- Thread 21의 source-to-SSR assurance와 Thread 22의 exact release gate가 remote result를 release 판단에 사용한다.
- 그러나 repository 내부에는 해당 remote policy/resource를 생성하는 독립 pipeline implementation이 없다.

### Evidence Boundary

- **Directly observed in repository:** required-CI requirement와 current tree의 workflow-file 부재.
- **Required/inferred from repository:** remote branch/status policy와 exact-SHA CI execution.
- **Actual execution not observable from Git:** current branch protection, external CI configuration, status check result, approver and result locator.

### Classification Rationale

독립적인 lifecycle을 repository commit 집합으로 구성하기 어렵고 특정 기존 Thread의 개발 내용으로 재정의할 수도 없다. 따라서 project release governance의 external step으로 남긴다.

### Documentation Action

Production deployment checklist와 change record에 required-check 이름, result locator, SHA, approver를 기록한다. 별도 Development Thread를 만들지 않는다.

---

## ESG-18 — Change record·운영 책임·maintenance/GO/abort 기록

### Repository Evidence

- production checklist section 0은 release/previous SHA, change record, deploy/reviewer/publisher/on-call, maintenance window, rollback 판단 시각과 final state를 요구한다.
- traffic section은 최종 GO, abort 기준, application rollback/publication rollback 분리를 요구한다.
- closure section은 artifact/migration/publication/health/backup/alert/rollback evidence locator를 요구한다.

### Required External Step

1. 실제 change/ticket을 생성한다.
2. 배포·reviewer·publisher·on-call 담당자를 지정한다.
3. maintenance window와 abort/rollback 기준을 승인한다.
4. GO/STOP/ROLLBACK 결정을 시각·담당자·근거와 함께 기록한다.
5. non-secret completion evidence와 deployed SHA/follow-up evidence-doc SHA를 연결한다.

### Code Connection

- control-plane role separation, monitoring response, deployment and rollback은 실제 인간 책임과 승인 창구 없이 완성되지 않는다.
- 이 기록은 코드 실행을 허용하거나 중단하는 운영 상태지만 application schema나 runtime subsystem은 아니다.

### Evidence Boundary

- **Directly observed in repository:** 빈 checklist/template와 요구 필드.
- **Required/inferred from repository:** actual ticket, owner assignment, window and decision record.
- **Actual execution not observable from Git:** 담당자 이름, ticket ID, approval time, GO/abort decision, incident/rollback result.

### Classification Rationale

여러 Thread를 관통하지만 자체 구현 commit/lifecycle을 가진 Development Thread가 아니라 project governance record다.

### Documentation Action

Thread별 학습 문서가 아니라 project-level production change record와 deployment completion evidence로 유지한다.

---

# Final Audit Conclusion

- **총 Gap:** 18
- **EXISTING_THREAD:** 16
- **NEW_THREAD:** 0
- **PROJECT_LEVEL_EXTERNAL_STEP:** 2
- **영향받는 Existing Threads:** 03, 04, 05, 08, 09, 11, 12, 14, 19, 20, 22
- **핵심 경계:** repository는 local source/live-smoke, migration, publication rehearsal, browser/load, backup/restore의 상당한 증거를 보존하지만, production platform·managed DB·role grants·secret resources·domain/DNS/TLS·IAM/MFA jobs·scheduler·monitoring/alerts·managed PITR·actual deployment/traffic/publication state는 소스 코드나 commit만으로 성립하거나 확인되지 않는다.
- **구조 판정:** 이러한 외부 단계는 이미 확정된 Thread 관점을 실제 환경에서 완성하는 단계로 자연스럽게 귀속된다. 새 Thread를 만드는 것보다 위 supplement packets를 최소 추가 문서로 제공하는 것이 기존 체계를 가장 덜 변경하면서 외부 상태 공백을 메운다.
