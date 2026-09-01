# External-State Development Gap Audit

**Repository:** `seungwoo7050/tmp-ft-transcendence`
**Existing Thread Index:** `pong.md`

## 감사 범위와 판정 요약

첨부된 Thread Index의 23개 Thread 구조와 commit 배정은 확정된 체계로 취급했으며, 기존 Thread의 제목이나 commit 구성을 재설계하지 않았다. 

저장소 ref를 확인한 결과 현재 Git ref는 `refs/heads/main` 하나이며, default branch 역시 `main`이다. 따라서 다른 활성 branch나 tag에만 존재하는 별도 개발 이력은 확인되지 않았다. main의 commit API를 빈 페이지가 반환될 때까지 순회하고, 현재 재귀 파일 트리와 관련 commit diff를 함께 검사했다.

**최종 판정은 다음과 같다.**

| 판정  |  수 |
| --- | -: |
| `EXISTING_THREAD` | 11 |
| `NEW_THREAD`  |  0 |
| `PROJECT_LEVEL_EXTERNAL_STEP` |  0 |

외부 상태 Gap은 Thread 04, 06, 07, 08, 21, 22, 23에 귀속된다. 별도의 신규 Thread를 만들 만큼 독립적인 external-state lifecycle은 발견되지 않았다.

가장 중요한 공통 경계는 다음과 같다.

> 저장소는 PostgreSQL, migration, seed, secret, container, health probe, Toxiproxy, CI 검증 환경이 **필요하고 어떻게 연결되는지**를 보여준다. 그러나 실제 production 배포 위치, 실제 secret 값, 실제 DB 및 volume, migration·seed 수행 시점, 관리자 지정 이력, metrics 수집기, fault-test 수행 결과는 Git에서 확인할 수 없다.

---

# Part I — Gap Index

## G-01 — PostgreSQL·Compose 네트워크·영속 볼륨의 실제 생성

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 22
**Related Threads:** Thread 04, Thread 21

**Repository Evidence 요약:** 현재 Compose는 PostgreSQL 16, 내부 서비스 네트워크, `pong-pong-db` named volume, 필수 `POSTGRES_PASSWORD`, one-shot migration service, API·Web·Caddy lifecycle을 정의한다. API는 migration 작업이 성공해야 시작하고, `docker compose down`만으로는 named volume이 제거되지 않는다.

초기 Compose 도입 commit은 실제 DB container와 named volume을 추가했고, 이후 production lifecycle commit은 하드코딩된 암호를 제거하고 migration/API/Web의 순서를 명시했다.

**Required External Step 요약:** Docker runtime에서 Compose project를 실제로 시작해 container, network, named volume을 생성하고, `POSTGRES_PASSWORD`를 주입해야 한다. 데이터 유지가 필요하면 volume을 보존하고, 폐기 환경이면 `down --volumes` 또는 동등한 제거 작업을 명시적으로 수행해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 실제 host, Compose project name, volume ID, DB password, 생성·삭제 시점 및 운영 데이터의 존재는 Git에서 알 수 없다.

**Documentation Action:** Thread 22 보충 문서에 "runtime resource materialization and volume retention boundary"를 추가한다.

---

## G-02 — Migration의 실제 실행과 PostgreSQL 실행 권한

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 04
**Related Threads:** Thread 21, Thread 22

**Repository Evidence 요약:** DB CLI의 `migrate` 명령은 `DATABASE_URL`로 실제 PostgreSQL에 접속해 SQL migration set을 실행한다. 현재 migration은 `001`부터 `006`까지 존재하며, `001_initial.sql`은 `pgcrypto` extension 및 주요 table·index를 생성한다. Migrator는 expected/applied set을 비교해 `current`, `pending`, `diverged`를 구분한다.

Compose에서는 migration service가 DB health 이후 실행되고 API가 그 성공을 기다린다.

**Required External Step 요약:** 접근 가능한 PostgreSQL database와 migration을 실행할 수 있는 role을 준비하고, 필요한 경우 `pgcrypto` extension 생성이 허용되도록 한 뒤 migration을 실행해야 한다. 이후 migration set이 `current`인지 확인해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 어느 database에 어느 migration까지 적용되었는지, `pgcrypto`가 실제로 생성되었는지, migration이 실패하거나 재시도되었는지는 Git에서 확인할 수 없다.

**Documentation Action:** Thread 04에 migration execution, 권한, `pending/diverged` 상태 및 실행 결과 비관찰 경계를 보충한다.

---

## G-03 — 개발·데모 Seed Profile의 명시적 실행

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 04
**Related Threads:** Thread 07, Thread 08, Thread 23

**Repository Evidence 요약:** seed는 `development`와 `demo` profile로 분리되어 있다. `development` profile은 예시 사용자, NPC, 관리자 및 예시 rating/stat을 만들고, `demo` profile은 NPC 중심 상태를 만든다. Migration과 seed는 별도 CLI 명령으로 분리되었으며 API startup seed는 최종적으로 제거되었다.

현재 CLI도 `seed:dev`와 `seed:demo`를 별도 명령으로 유지한다.

**Required External Step 요약:** 예시 사용자·NPC·관리자 데이터가 필요한 환경에서 migration 이후 적절한 profile을 선택해 seed CLI를 별도로 실행해야 한다. Production runtime의 성립 자체에는 seed가 요구되지 않으며, guest demo의 process-local guest AI 또한 별도 seed를 전제하지 않는다.

**실제 수행 여부 확인 가능성:** 불가. 어떤 환경에 어느 profile이 실행되었는지, 생성된 row가 현재도 존재하는지는 Git에서 알 수 없다.

**Documentation Action:** Thread 04에 "seed는 startup side effect가 아니라 명시적 환경 준비 작업"이라는 supplement를 추가한다.

---

## G-04 — 전용 Test Database·Schema의 생성과 파괴적 Reset

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 04
**Related Threads:** Thread 23

**Repository Evidence 요약:** `reset:test`는 `NODE_ENV=test`와 `TEST_DATABASE_URL`을 요구한다. 대상은 이름이 test 전용임을 나타내는 database 또는 `test_<32 hex>` 형식의 격리 schema여야 한다. 실제 실행은 대상 schema를 `drop ... cascade`한 뒤 다시 만들고 전체 migration을 적용한다.

**Required External Step 요약:** 파괴해도 되는 전용 test database 또는 규칙에 맞는 격리 schema를 먼저 생성하고, `TEST_DATABASE_URL`을 주입해야 한다. 실행 role에는 해당 schema를 drop/create할 수 있는 권한이 필요하다.

**실제 수행 여부 확인 가능성:** 불가. 실제 test database/schema의 이름, 소유자, reset 실행 횟수 및 삭제된 데이터는 Git에서 확인할 수 없다.

**Documentation Action:** Thread 04에 destructive test reset의 외부 대상 준비와 안전 경계를 보충한다.

---

## G-05 — Guest Session Signing Secret의 생성·주입·교체

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 08
**Related Threads:** Thread 06, Thread 22

**Repository Evidence 요약:** `demo`와 `production` mode는 UTF-8 기준 32byte 이상의 `SESSION_SECRET`이 없으면 환경 구성을 거부한다. Compose는 mode와 관계없이 `SESSION_SECRET` 주입을 필수화한다. `.env`와 `.env.local`은 Git에서 제외된다.

Guest session은 해당 secret으로 HMAC-SHA256 서명되며, 짧은 secret을 거부하는 테스트도 존재한다.

**Required External Step 요약:** 환경별로 32byte 이상인 deployment-specific secret을 생성하고, Git에 commit하지 않은 상태로 API runtime에 주입해야 한다. 교체 시점과 이전 cookie 처리 정책도 외부에서 결정해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 실제 secret 값, 생성 방식, secret manager 사용 여부, 교체 주기와 교체 이력은 확인할 수 없다.

**Documentation Action:** Thread 08에 guest secret provisioning과 rotation boundary를 추가한다. 등록 사용자 DB session token이 이 secret으로 서명된다고 설명해서는 안 된다.

---

## G-06 — Migration에 의한 기존 Session의 일괄 무효화

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 06
**Related Threads:** Thread 04, Thread 22

**Repository Evidence 요약:** `005_expire_legacy_sessions.sql`은 단 한 문장인 `delete from sessions;`를 실행한다. 이를 추가한 commit은 target migration 지원과 함께 legacy session 만료를 명시적으로 도입했다.

**Required External Step 요약:** 해당 migration을 기존 session row가 있는 환경에 적용할 때는 모든 등록 사용자 session이 제거된다는 운영 영향을 반영해야 한다. 적용 후 사용자는 환경에서 제공되는 인증 경로를 통해 새 session을 받아야 한다.

**실제 수행 여부 확인 가능성:** 불가. 이 migration이 실제 production 또는 staging DB에서 실행되었는지, 몇 개의 session이 삭제되었는지, 사용자 안내나 배포 조정이 있었는지는 알 수 없다.

**Documentation Action:** Thread 06에 auth-state migration side effect를 보충하고, migration 실행 자체는 Thread 04를 참조하도록 한다.

---

## G-07 — 최초 관리자 Role의 명시적 할당과 관리자 복구 상태

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 07
**Related Threads:** Thread 04, Thread 06, Thread 22

**Repository Evidence 요약:** 일반 개발 로그인은 handle이 `admin`이어도 자동으로 관리자 role을 부여하지 않도록 변경되었다. 대신 `user:set-role <handle> <user|admin>` CLI가 추가되었고, 이미 존재하는 non-NPC 사용자 row에 대해 role을 갱신한다.

README의 개발 경로도 먼저 개발 로그인을 생성한 후 같은 handle에 관리자 role을 지정하도록 설명하며, 유일한 관리자가 자신을 정지시키면 다른 active admin 또는 직접 DB 복구가 필요하다고 명시한다.

**Required External Step 요약:** 개발 관리자 경로에서는 사용자 row를 먼저 만든 뒤 CLI로 role을 할당해야 한다. 관리자 상태 변경 기능을 지속적으로 사용할 환경이라면 최소한 하나의 별도 active admin 또는 외부 DB 복구 경로를 유지해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 실제 관리자 handle, role 지정자, 지정 시점, 현재 active admin 수와 복구 이력은 Git에 없다.

**Documentation Action:** Thread 07에 initial admin bootstrap과 lockout recovery boundary를 추가한다. Production 인증·계정 생성 절차가 존재하는 것처럼 확장해서는 안 된다.

---

## G-08 — Public URL·WebSocket URL·Secure Transport·Web Build의 정렬

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 22
**Related Threads:** Thread 02, Thread 06, Thread 08, Thread 12

**Repository Evidence 요약:** API의 `WEB_ORIGIN`은 runtime 값이지만, Web의 `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_WS_URL`, `NEXT_PUBLIC_APP_MODE`는 Docker build argument로 Web artifact에 포함된다. Compose는 `PUBLIC_ORIGIN`, `PUBLIC_WS_URL`, `APP_MODE`를 두 경계에 전달한다.

현재 Caddy는 `:8080`의 평문 HTTP reverse proxy만 정의하며 TLS certificate나 domain automation은 없다. 한편 demo guest cookie는 코드에서 `secure: true`로 설정된다.

**Required External Step 요약:** 배포 시 실제 public HTTP origin과 WebSocket URL을 결정하고, 그 값으로 Web image를 다시 빌드한 뒤 API runtime origin과 일치시켜야 한다. localhost가 아닌 browser-facing demo에서는 secure cookie와 HTTPS page의 WebSocket 연결을 성립시키는 HTTPS/WSS termination이 별도로 필요하다는 점이 코드에서 추론된다.

**실제 수행 여부 확인 가능성:** 불가. 실제 domain, public IP, reverse proxy 계층, TLS 종료 위치, certificate, HTTPS/WSS URL 및 해당 값으로 빌드된 image는 Git에서 확인할 수 없다.

**Documentation Action:** Thread 22에 build-time/public-runtime URL coupling과 조건부 TLS termination을 추가한다. DNS 등록이나 certificate 발급이 실제로 수행되었다고 서술하지 않는다.

---

## G-09 — Health Probe 활성화와 내부 Metrics 수집기 연결

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 21
**Related Threads:** Thread 22, Thread 23

**Repository Evidence 요약:** API는 liveness, readiness, Prometheus-format metrics endpoint를 제공한다. Readiness는 lifecycle, database 연결, migration set 상태를 검사하며, Compose는 API와 Web healthcheck를 실제 dependency gate로 사용한다.

Caddy의 public `/api/metrics` 경로는 의도적으로 404 처리되므로 metrics는 API 내부 접근 경로에서 수집해야 한다.

**Required External Step 요약:** Compose stack을 실행해 healthcheck를 활성화하고, metrics를 실제 관측에 사용할 경우 API service에 내부적으로 접근할 수 있는 scraper를 별도로 연결해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 실제 scraper 제품, network 위치, scrape interval, retention, dashboard, alert rule과 수집 이력은 저장소에 없다.

**Documentation Action:** Thread 21에 endpoint 구현과 실제 monitoring system activation 사이의 경계를 추가한다.

---

## G-10 — Toxiproxy Fault Environment의 생성·상태 전환·정리

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 23
**Related Threads:** Thread 04, Thread 21, Thread 22

**Repository Evidence 요약:** load overlay는 Toxiproxy container와 bootstrap service를 생성하고, API의 DB 경로를 PostgreSQL proxy로 우회시킨다. Control script는 PostgreSQL·edge proxy를 만들고 latency, down, reset, up 상태를 실제 Toxiproxy API에 적용한다.

Fault scenario는 baseline → DB latency → DB down → DB recovery → edge latency/reset/recovery를 수행하며 종료 시 proxy state를 reset하도록 구현되어 있다.

**Required External Step 요약:** 폐기 가능한 DB를 사용하는 base stack과 load overlay를 실제로 시작하고, proxy bootstrap 후 fault scenario를 실행하며, 성공·실패 여부와 관계없이 toxic state 및 container resource를 정리해야 한다.

**실제 수행 여부 확인 가능성:** 불가. 실제 fault-test 실행 시점, host 성능, 주입한 지연값, 생성된 데이터, 장애 관측 결과와 cleanup 성공 여부는 Git에서 확인할 수 없다.

**Documentation Action:** Thread 23에 fault environment lifecycle 및 반드시 수행해야 하는 reset/teardown 경계를 추가한다.

---

## G-11 — CI·Smoke·E2E의 일시적 Resource와 진단 Artifact Lifecycle

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 23
**Related Threads:** Thread 04, Thread 08, Thread 21, Thread 22

**Repository Evidence 요약:** GitHub Actions workflow는 transient PostgreSQL service를 생성하고 migration·`seed:dev`를 실행한 뒤 API, Web, Chromium 기반 E2E를 구동한다. 별도의 demo browser job과 production Compose job도 있으며, production Compose job은 종료 시 volume을 포함해 resource를 삭제한다. 실패 시 Playwright 진단 artifact를 7일간 업로드하도록 정의되어 있다.

이 process/browser 검증 경계는 관련 commit에서 PostgreSQL service, migration, seed, process startup과 browser test 순서로 추가되었다.

**Required External Step 요약:** GitHub Actions runner 또는 동등한 검증 host에서 transient database, process, browser 및 Compose resource를 실제로 생성하고, 실패 artifact와 cleanup lifecycle을 수행해야 한다. 로컬 smoke/E2E도 사용자·경기 데이터를 남기므로 폐기 가능한 DB를 대상으로 실행해야 한다.

**실제 수행 여부 확인 가능성:** Git만으로는 불가. Workflow 정의는 확인되지만 실제 run 성공 여부, required-check 설정, branch protection, artifact 실물 및 runner cleanup 결과는 Git history의 일부가 아니다.

**Documentation Action:** Thread 23에 "검증 코드 존재"와 "외부 검증 환경이 실제로 실행됨"을 구분하는 supplement를 추가한다.

---

# Part II — Existing Thread Supplement Packets

## Packet T04 — 마이그레이션·시드·테스트 데이터베이스 수명 주기

### Thread Identity

**Type:** Existing Thread
**Thread:** 04
**한국어 제목:** 마이그레이션·시드·테스트 데이터베이스 수명 주기
**English title:** Migration, Seed, and Test Database Lifecycle

### Gaps

`G-02`, `G-03`, `G-04`

### Repository Evidence

| Commit | Subject | 관련 파일 | 이 Packet에서 중요한 이유 |
| --- | --- | --- | --- |
| `1140fb8687145ee70465aa734b61c2b990d1d220` | `feat(db): migration 실행 경계 구성`  | `packages/db/src/migrations.ts` | PostgreSQL schema와 `pgcrypto`를 코드에 정의한 최초 실행 경계다.  |
| `8da6edef28ebd57bc6821dd78ff7b27b6e0772fc` | `feat(db): 환경별 seed profile 분리`  | `packages/db/src/index.ts`  | development와 demo seed가 동일한 외부 상태를 만들지 않도록 분리한다.  |
| `981ee655559bcacd17ba563ec125126dc4a86a91` | `refactor(db): migration과 seed CLI 연결` | `packages/db/src/cli.ts`, `packages/db/package.json`, `apps/api/src/index.ts` | Migration과 seed를 별도 명령으로 만들고 PostgreSQL startup seed를 제거했다. |
| `e1a0316fbe8444289fe7b465f9776f97f3d7a69f` | `fix(api): startup seed 생성을 제거`  | `apps/api/src/index.ts` | 최종적으로 memory runtime까지 자동 seed를 제거하여 seed 실행을 완전히 명시적 단계로 만들었다. |
| `113b3c42219235a418ef66fffc3d6993f31b49df` | `feat(db): test database reset target guard 추가` | `packages/db/src/testReset.ts`  | 파괴적 reset 대상을 test 전용 DB/schema로 제한한다. |
| `434403a7c16a0913d079d2dd4b911de13c52baff` | `feat(db): test schema reset과 migration 실행 연결` | `packages/db/src/testReset.ts`, `packages/db/src/cli.ts`  | 외부 schema drop/create와 migration 재적용을 하나의 reset 작업으로 연결한다.  |

현재 final state의 실행 계약은 다음과 같다.

```text
DATABASE_URL + migrate
  -> bundled SQL migration을 순서대로 실행

DATABASE_URL + seed:dev
  -> 개발 사용자·관리자·NPC·예시 통계 생성

DATABASE_URL + seed:demo
  -> demo profile의 seed 상태 생성

NODE_ENV=test + TEST_DATABASE_URL + reset:test
  -> 안전한 test target 검증
  -> schema drop cascade
  -> schema recreate
  -> migration 재적용
```

이 계약은 현재 CLI, migrator 및 test reset 구현에서 직접 확인된다.

현재 SQL migration set은 `001`부터 `006`까지이며, 그중 `001`은 `pgcrypto` 및 주요 schema를 생성한다. `005`의 session 삭제 부작용은 이 Packet에서 중복 소유하지 않고 Thread 06 Packet으로 넘긴다.

### External Development Steps

**Conceptual execution order:**

1. PostgreSQL database와 접속 role을 외부에서 준비한다.
2. `DATABASE_URL`을 migration process에 주입한다.
3. role이 bundled SQL과 `pgcrypto` extension 생성을 수행할 수 있게 한다.
4. `migrate`를 실행한다.
5. API가 요청을 받기 전에 migration set이 `current`인지 확인한다.
6. 예시 데이터가 필요한 경우에만 `seed:dev` 또는 `seed:demo`를 선택해 실행한다.
7. 테스트 격리가 필요하면 전용 test database 또는 규칙에 맞는 isolated schema를 생성한다.
8. `TEST_DATABASE_URL`을 주입하고 `reset:test`를 실행한다.

이는 실제 historical execution 순서가 아니라 repository가 요구하는 **conceptual execution order**다.

### Code Connection

* PostgreSQL repository는 migration으로 생성되는 table을 직접 조회·갱신한다.
* API readiness는 DB 연결뿐 아니라 applied migration set이 repository의 expected set과 일치하는지 검사한다.
* Seed 결과는 leaderboard, 개발 로그인, 관리자 테스트, NPC 상대와 테스트 fixture의 실제 row로 연결된다.
* Test reset은 integration test가 이전 실행의 row와 schema 영향을 이어받지 않게 한다.
* Production API는 `DATABASE_URL`이 없으면 시작하지 않는다.

### Evidence Boundary

**Directly observed in repository**

* Migration SQL 파일과 실행 코드가 존재한다.
* `pgcrypto`, table, index 생성문이 존재한다.
* Migration과 seed는 별도 CLI다.
* Startup seed는 제거되었다.
* Test reset은 schema를 drop/recreate한다.
* Migration set은 `current`, `pending`, `diverged`로 검사된다.

**Required/inferred from repository**

* 실제 database와 접속 role이 먼저 존재해야 한다.
* Migration role은 필요한 DDL과 extension 관련 명령을 실행할 수 있어야 한다.
* Seed는 migration 이후 실행해야 한다.
* Test reset 대상은 폐기 가능한 격리 환경이어야 한다.

**Actual execution not observable from Git**

* 실제 DB endpoint와 credential
* Migration 적용 시점과 적용 대상
* Applied migration table의 현재 내용
* Seed profile과 실행 횟수
* Reset으로 삭제된 실제 데이터
* 실패 시 수동 복구 또는 rollback 절차

---

## Packet T06 — 쿠키 세션에서 일회용 WebSocket 티켓까지의 인증 경계

### Thread Identity

**Type:** Existing Thread
**Thread:** 06
**한국어 제목:** 쿠키 세션에서 일회용 WebSocket 티켓까지의 인증 경계
**English title:** Authentication Boundary from Cookie Sessions to One-Time WebSocket Tickets

### Gaps

`G-06`

### Repository Evidence

대표 commit은 다음과 같다.

| Commit | Subject  | 관련 파일  | 관련 diff |
| --- | --- | --- | --- |
| `b93910708330c0e07c44b0f6ae44c4a8959a66e2` | `feat(db): legacy session을 안전하게 만료` | `packages/db/migrations/005_expire_legacy_sessions.sql`, `packages/db/src/migrator.ts` | `+ delete from sessions;` 및 target migration 지원. |

현재 migration 파일도 다음 외부 상태 변경을 그대로 유지한다.

```sql
delete from sessions;
```

### External Development Steps

**Conceptual execution order:**

1. 대상 DB에 기존 session row가 있는지 여부와 관계없이 migration의 효과를 사전에 인지한다.
2. 인증 관련 배포와 DB migration의 순서를 조정한다.
3. Migration을 실행한다.
4. 기존 cookie가 참조하던 server-side session row가 사라졌음을 전제로 인증 상태를 처리한다.
5. 해당 환경에서 제공되는 인증 방식이 있다면 사용자가 새 session을 생성하도록 한다.

Production에는 OAuth 또는 별도 가입·로그인 공급자가 포함되어 있지 않으므로, "migration 후 production 사용자가 자동으로 다시 로그인한다"고 서술할 근거는 없다.

### Code Connection

* 등록 사용자 cookie는 server-side `sessions` table의 token row와 연결된다.
* `delete from sessions` 이후 기존 cookie 문자열이 남아 있어도 대응 row가 없어 인증되지 않는다.
* 새 WebSocket ticket 발급은 인증된 HTTP session을 요구하므로, 기존 session 삭제는 이후 ticket 발급에도 연결된다.
* Migration은 `ws_tickets` 자체를 삭제하는 것으로 확인되지 않으므로 그 이상의 영향을 추정하지 않는다.

### Evidence Boundary

**Directly observed in repository**

* Migration이 `sessions` 전체 row를 삭제한다.
* 해당 migration은 ordinary migration set의 일부다.
* API는 server-side session row를 조회한다.

**Required/inferred from repository**

* Migration이 적용된 환경의 기존 등록 사용자 session은 무효가 된다.
* 운영상 재인증 또는 session 재수립을 고려해야 한다.

**Actual execution not observable from Git**

* 해당 migration의 실제 적용 여부
* 삭제된 session 수
* 사용자 안내, maintenance window, rollout 방식
* Production 재인증 수단 또는 운영자 대응

---

## Packet T07 — 계정 정지·감사 기록·실시간 권한 철회

### Thread Identity

**Type:** Existing Thread
**Thread:** 07
**한국어 제목:** 계정 정지·감사 기록·실시간 권한 철회
**English title:** Account Suspension, Audit Logging, and Live Revocation

### Gaps

`G-07`

### Repository Evidence

| Commit | Subject  | 관련 파일  | 중요한 diff  |
| --- | --- | --- | --- |
| `45225adcfcd905aebfaea4e8ec5a080dbd531847` | `feat(db): 명시적 사용자 role 할당 추가` | `packages/db/src/cli.ts`, `packages/db/src/index.ts`, `apps/api/src/admin.test.ts` | 개발 로그인 사용자의 기본 role을 `user`로 고정하고 `user:set-role` CLI를 추가했다. |

현재 CLI 계약은 다음과 같다.

```text
user:set-role <existing-handle> user|admin
```

대상 사용자가 없거나 NPC이면 성공하지 않는다.

README에 기록된 개발 관리자 경로도 "개발 로그인으로 사용자 생성 → 같은 handle에 role 지정" 순서이며, 유일한 관리자가 정지되면 다른 active admin 또는 DB 복구가 필요하다고 명시한다.

### External Development Steps

**Conceptual execution order:**

1. Development mode에서 개발 로그인으로 대상 사용자 row를 만든다.
2. 같은 `DATABASE_URL`을 사용하는 DB CLI에서 `user:set-role <handle> admin`을 실행한다.
3. 다음 session 조회에서 최신 repository role이 반영되는지 확인한다.
4. 관리자 상태 변경을 운용할 경우 다른 active admin 또는 DB 수준 복구 경로를 유지한다.
5. 관리자 자신을 정지시키는 경우 현재 session으로는 복구할 수 없다는 실패 조건을 인지한다.

### Code Connection

* Admin API authorization은 사용자의 현재 role과 status에 연결된다.
* Role은 source code의 상수가 아니라 실제 DB row 상태다.
* `user:set-role`은 관리 UI가 없어도 DB 외부 실행을 통해 관리자 권한 상태를 만든다.
* Seed가 관리자 row를 만들 수 있지만, 문서화된 수동 개발 경로는 로그인 후 명시적 role 지정이다.
* Production에는 사용자 생성·인증 provider가 없으므로 이 CLI를 production bootstrap 절차로 일반화하지 않는다.

### Evidence Boundary

**Directly observed in repository**

* `admin` handle 자체는 관리자 권한을 부여하지 않는다.
* Role 변경 CLI가 있다.
* CLI는 기존 non-NPC 사용자 row를 갱신한다.
* 관리자 정지 상태가 authorization에 영향을 준다.

**Required/inferred from repository**

* 최초 관리자 상태를 사용할 환경에서는 role row를 실제로 만들어야 한다.
* 단일 관리자의 정지에 대비한 별도 복구 주체가 필요하다.

**Actual execution not observable from Git**

* 현재 admin 사용자 목록
* 실제 role 변경 이력
* CLI 실행 주체
* 정지·해제 이력
* 직접 DB 복구 방법과 수행 여부

---

## Packet T08 — 비회원 체험의 신원·자원·데이터 격리

### Thread Identity

**Type:** Existing Thread
**Thread:** 08
**한국어 제목:** 비회원 체험의 신원·자원·데이터 격리
**English title:** Guest Demo Identity, Resource, and Data Isolation

### Gaps

`G-05`

### Repository Evidence

| Commit | Subject  | 관련 파일 | 중요한 내용  |
| --- | --- | --- | --- |
| `f801ccd09cf023c62a34f164b72998fe32f08f26` | `feat(guest): guest runtime 환경 경계 구성`  | `.env.example`, `apps/api/src/env.ts` | Demo/production의 minimum secret length, APP_MODE, TRUST_PROXY 및 public URL 환경 계약을 추가했다. |
| `f877ff676d656ad6ac7fafb55e19ca2fa5aabb7b` | `test(auth): guest session secret 요구 검증` | `apps/api/src/auth-boundary.test.ts`  | 짧은 guest session secret을 거부하는 검증이다. |

Final state에서는 다음이 직접 관찰된다.

```text
demo/production:
  SESSION_SECRET absent or UTF-8 byte length < 32
  -> runtime configuration failure
```

Guest cookie payload은 secret 기반 HMAC-SHA256으로 서명되고 검증된다.

### External Development Steps

**Conceptual execution order:**

1. 환경마다 별도의 deployment-specific `SESSION_SECRET` 값을 생성한다.
2. 값이 UTF-8 기준 최소 32byte 조건을 충족하는지 확인한다.
3. Git에 commit하지 않고 API runtime 또는 Compose environment에 주입한다.
4. 모든 API instance가 같은 guest session을 검증해야 하는 배포 형태라면 동일한 active secret을 공유하도록 구성한다.
5. Secret 교체 시 기존 guest cookie의 처리 및 사용자 재접속 영향을 결정한다.

### Code Connection

* Guest cookie의 서명 검증은 `SESSION_SECRET`에 직접 연결된다.
* Guest cookie에는 만료 시점과 IP가 포함되고, signature가 일치하지 않으면 인증되지 않는다.
* 따라서 secret 교체 후 이전 secret으로 서명된 cookie가 검증되지 않는다는 것은 코드에서 도출되는 결과다.
* 등록 사용자의 server-side session token은 DB row 기반이며, 이 secret으로 서명된다고 볼 근거가 없다.
* Production 환경도 현재 env contract상 secret을 요구하지만, `GuestAccess`의 직접적인 생성은 demo mode에 한정된다.

### Evidence Boundary

**Directly observed in repository**

* Minimum 32byte 검사가 있다.
* Compose가 secret 주입을 요구한다.
* Guest session은 HMAC-SHA256으로 서명된다.
* `.env` 파일은 Git에서 제외된다.

**Required/inferred from repository**

* 실제 secret 값을 외부에서 생성하고 runtime에 주입해야 한다.
* Secret 변경은 기존 guest cookie 검증에 영향을 준다.
* 다중 instance가 같은 guest cookie를 검증해야 한다면 secret 정렬이 필요하다.

**Actual execution not observable from Git**

* 실제 secret 값 및 entropy
* 저장 매체와 secret manager
* 접근 권한
* 교체 주기와 이전 secret 병행 여부
* 실제 guest cookie 무효화 사건

---

## Packet T21 — 준비 상태·메트릭·데이터베이스 장애 관측

### Thread Identity

**Type:** Existing Thread
**Thread:** 21
**한국어 제목:** 준비 상태·메트릭·데이터베이스 장애 관측
**English title:** Readiness, Metrics, and Database Failure Observability

### Gaps

`G-09`

### Repository Evidence

| Commit | Subject | 관련 파일  | 중요한 내용  |
| --- | --- | --- | --- |
| `15002e229acbe16a1fd89c1064c0c3e6aab2cff7` | `feat(ops): liveness와 readiness endpoint 추가` | `apps/api/src/app.ts`, `packages/shared/src/http.ts` | DB와 migration 상태를 포함하는 readiness 계약을 만든다.  |
| `69278d8fc4564d0c283173d5191b42fa343f279f` | `feat(metrics): runtime gauge registry 추가`  | `apps/api/src/observability.ts`  | WebSocket 연결, queue, room 및 Node runtime 지표를 Prometheus registry에 등록한다. |

Final state에서 readiness 성공 조건은 다음과 같다.

```text
lifecycle == accepting
database == up
migrations == current | not_applicable
```

API는 `/metrics`를 노출하지만 Caddy의 public `/api/metrics`는 404로 차단된다. Compose healthcheck는 `/health/ready`를 주기적으로 호출한다.

### External Development Steps

**Conceptual execution order:**

1. DB와 migration이 준비된 runtime stack을 시작한다.
2. Compose healthcheck가 API readiness를 실제로 호출하도록 한다.
3. Public Caddy가 아닌 내부 API endpoint에 접근할 수 있는 metrics collector를 준비한다.
4. Collector가 `/metrics`를 주기적으로 scrape하도록 외부에서 등록한다.
5. 수집 장애와 API 장애를 구분할 수 있도록 collector의 network reachability를 관리한다.

### Code Connection

* DB가 down이면 readiness가 503으로 바뀐다.
* Migration set이 pending 또는 diverged이면 API는 traffic-ready가 아니다.
* Compose는 이 readiness를 API 이후 Web·Caddy startup ordering에 사용한다.
* Metrics endpoint는 호출되어야 registry의 현재 값을 외부 시스템이 보존할 수 있다.
* Caddy 차단 때문에 인터넷 공개 URL을 scraper target으로 사용할 수 없다.

### Evidence Boundary

**Directly observed in repository**

* Liveness, readiness, metrics endpoint가 있다.
* Compose healthcheck가 readiness를 호출한다.
* Caddy가 public metrics 경로를 차단한다.
* Prometheus-format registry가 있다.

**Required/inferred from repository**

* Metrics의 지속적 관측에는 내부 scraper가 필요하다.
* 다른 orchestrator를 사용할 경우 동등한 probe 등록이 필요하다.

**Actual execution not observable from Git**

* 실제 Prometheus 또는 다른 collector 배포 여부
* Scrape target, interval, retention
* Dashboard와 alert
* 실제 readiness failure 또는 recovery 이력

---

## Packet T22 — 프로덕션 컨테이너·영속 저장소·정상 종료

### Thread Identity

**Type:** Existing Thread
**Thread:** 22
**한국어 제목:** 프로덕션 컨테이너·영속 저장소·정상 종료
**English title:** Production Containers, Persistent Storage, and Graceful Shutdown

### Gaps

`G-01`, `G-08`

### Repository Evidence

| Commit | Subject | 관련 파일 | 중요한 내용 |
| --- | --- | --- | --- |
| `19b4d9f9083d8f08c3066d8a9bd667828d470473` | `build(runtime): Compose와 Caddy 라우팅 추가`  | `docker-compose.yml`, `Caddyfile` | DB container, named volume, API/Web/Caddy network와 공개 port를 최초로 연결했다. |
| `2c44cb7cd71fb540a836f11a59a848c1c6145757` | `build(docker): production container lifecycle 구성` | `docker-compose.yml`  | 필수 password/secret, one-shot migration, health ordering, Web build args와 production images를 추가했다. |

현재 final-state Compose의 핵심 계약은 다음과 같다.

```text
db healthy
  -> migrate completes successfully
  -> api healthy
  -> web healthy
  -> caddy starts

persistent state:
  pong-pong-db -> /var/lib/postgresql/data

runtime inputs:
  POSTGRES_PASSWORD
  SESSION_SECRET
  APP_MODE
  PUBLIC_ORIGIN
  PUBLIC_WS_URL
```

Web image는 public API, WebSocket URL과 mode를 builder stage에서 artifact에 고정한다.

현재 Caddy는 plain HTTP `:8080`만 제공하며, demo cookie는 `secure: true`로 설정된다.

### External Development Steps

**Conceptual execution order:**

1. 대상 deployment mode를 정한다.
2. PostgreSQL password와 session secret을 외부에서 준비한다.
3. 실제 public origin과 WebSocket URL을 결정한다.
4. localhost가 아닌 demo deployment라면 HTTPS/WSS를 제공할 TLS termination 계층을 준비한다.
5. 결정한 public URL과 mode로 Web image를 빌드한다.
6. 같은 origin을 API runtime의 `WEB_ORIGIN`에 주입한다.
7. Compose stack을 시작해 DB, network, volume, migration, API, Web, Caddy를 생성한다.
8. Readiness가 통과한 뒤 traffic을 허용한다.
9. 종료 시 API의 room drain과 container grace period를 거친다.
10. 환경 폐기 여부에 따라 PostgreSQL volume을 보존하거나 명시적으로 제거한다.

이 순서는 repository가 요구하는 **conceptual execution order**이며, 실제 배포 이력을 의미하지 않는다.

### Code Connection

* `pong-pong-db`가 유지되어야 PostgreSQL data가 container 재생성 후에도 남는다.
* API는 migration service가 실패하면 시작하지 않는다.
* Web public URL은 runtime 환경변수만 변경해서 바뀌지 않고 image rebuild가 필요하다.
* API origin과 Web artifact origin이 다르면 CORS, cookie, HTTP 및 WebSocket 접속 경계가 일치하지 않을 수 있다.
* `TRUST_PROXY=1`은 Compose에서 실제 Caddy reverse proxy가 앞에 있는 구성과 연결된다.
* Secure guest cookie와 browser-facing remote demo를 함께 사용하려면 repository 밖의 secure transport가 필요하다는 점이 추론된다.
* Current Compose는 TLS certificate를 발급하거나 배치하지 않는다.

### Evidence Boundary

**Directly observed in repository**

* Compose services, network dependency, named volume이 정의되어 있다.
* Password와 secret은 required interpolation이다.
* Web public values는 build args다.
* Caddy는 `:8080` HTTP proxy다.
* Demo cookie는 secure flag를 사용한다.
* API stop grace period는 70초다.

**Required/inferred from repository**

* Compose stack을 실제로 시작해야 runtime resource가 생성된다.
* Public URL 변경 시 Web image rebuild가 필요하다.
* Remote demo의 secure cookie 사용에는 HTTPS/WSS termination이 필요하다.
* Volume 유지·폐기는 외부 운영자가 선택해야 한다.

**Actual execution not observable from Git**

* 실제 deployment host, VM, container IDs
* Network와 volume IDs 및 현재 데이터
* Public domain과 DNS
* TLS provider, certificate, renewal
* 실제 image digest와 build args
* 실제 shutdown 또는 drain 결과
* Production/staging 존재 여부

---

## Packet T23 — 계층형 검증·부하 시험·장애 주입

### Thread Identity

**Type:** Existing Thread
**Thread:** 23
**한국어 제목:** 계층형 검증·부하 시험·장애 주입
**English title:** Layered Verification, Load Testing, and Fault Injection

### Gaps

`G-10`, `G-11`

### Repository Evidence

| Commit | Subject | 관련 파일 | 중요한 내용 |
| --- | --- | --- | --- |
| `7b0b5f086b4193b9585e5a8bf6e55ec6e52fb447` | `test(load): 실시간 fault injection 도구 추가` | `docker-compose.load.yml`, `tests/load/toxiproxy-control.mjs`, load scripts | Toxiproxy resource, proxy bootstrap, realtime load/fault 검증을 추가했다. |
| `3367b4266049a4ec3d8a24f6d2c72de87c444fd4` | `ci(repo): process와 browser 검증 job 추가`  | GitHub Actions workflow | Transient PostgreSQL, migration, seed, API/Web process, Chromium, smoke/E2E 실행 순서를 추가했다. |

Current fault environment는 PostgreSQL과 edge proxy를 생성하고, 다음 상태 변화를 지원한다.

```text
postgres:
  latency
  down
  up

edge:
  latency
  reset_peer
  down
  up

cleanup:
  remove all toxics
  enable proxies
```

Automated fault scenario는 `finally` 성격의 reset을 수행하도록 구현되어 있다.

현재 CI workflow는 transient PostgreSQL과 Compose volume을 만들며, production Compose job은 `docker compose down --volumes --remove-orphans`로 정리한다. 실패한 browser test의 진단 artifact는 7일 보존된다.

### External Development Steps

**Fault/load conceptual execution order:**

1. 폐기 가능한 PostgreSQL data target을 준비한다.
2. Base Compose와 load overlay를 함께 시작한다.
3. Toxiproxy control API와 proxy definitions를 생성한다.
4. Baseline readiness를 확인한다.
5. DB latency와 DB down 상태를 주입한다.
6. DB를 복구하고 readiness recovery를 확인한다.
7. 필요한 경우 edge latency/reset을 주입한다.
8. Proxy를 정상 상태로 reset한다.
9. Container, network 및 테스트 DB state를 정리한다.

**CI/process conceptual execution order:**

1. GitHub Actions runner 또는 동등한 host를 할당한다.
2. Transient PostgreSQL service를 시작한다.
3. Dependency와 browser runtime을 설치한다.
4. Build 후 migration·seed를 실행한다.
5. API/Web process를 시작하고 readiness를 기다린다.
6. HTTP, WebSocket, browser test를 실행한다.
7. 실패 시 진단 artifact를 저장한다.
8. Process와 Compose resource를 정리한다.

### Code Connection

* Fault scenario는 API readiness의 DB 상태 변화를 직접 검증한다.
* Toxiproxy control state는 Git file이 아니라 실행 중인 proxy process 안에 존재한다.
* Load, smoke, E2E는 실제 사용자·session·match row를 생성할 수 있다.
* CI PostgreSQL service와 Compose volume은 workflow run 동안만 존재하는 external state다.
* Failure artifact는 Git commit이 아니라 GitHub Actions storage에 남는다.
* Workflow 정의가 존재한다는 사실은 실제 run 성공을 증명하지 않는다.

### Evidence Boundary

**Directly observed in repository**

* Fault proxy, control commands와 cleanup 코드가 있다.
* CI workflow가 transient DB와 processes를 정의한다.
* Migration과 seed가 CI process test 전에 실행된다.
* Production Compose test가 volume까지 제거한다.
* Failure diagnostics upload와 retention이 정의된다.

**Required/inferred from repository**

* Fault test는 실제 container와 proxy state를 만들어야 의미가 있다.
* 검증 데이터가 남으므로 폐기 가능한 DB가 필요하다.
* 중단된 fault test에서도 proxy reset과 teardown이 필요하다.

**Actual execution not observable from Git**

* 실제 workflow run과 성공 여부
* Runner image의 실제 상태
* Fault injection 수행 결과
* 측정된 latency·throughput
* 생성된 테스트 row
* Artifact 실물과 다운로드 이력
* Cleanup이 실제로 완료되었는지 여부

---

# Part III — Proposed New Thread Packets

## 제안 없음

이번 감사에서는 `NEW_THREAD`를 제안하지 않는다.

그 이유는 다음과 같다.

1. DB resource, migration, seed, test reset은 이미 Thread 04와 Thread 22의 명확한 관점 안에 있다.
2. Session invalidation은 독립 운영 Thread라기보다 Thread 06의 authentication lifecycle을 migration 실행으로 완성하는 단계다.
3. Guest secret은 Thread 08의 identity boundary를 실제 runtime에 성립시키는 단계다.
4. 관리자 role bootstrap은 Thread 07의 authorization state를 실제 DB에 만드는 단계다.
5. Public URL, Web build, TLS termination, volume, Compose resource는 Thread 22의 deployment 관점에 자연스럽게 귀속된다.
6. Metrics collector는 Thread 21의 observability를 실제 환경에서 활성화하는 단계다.
7. Toxiproxy와 CI transient resources는 Thread 23의 검증·장애 주입 lifecycle 자체다.
8. `SESSION_SECRET`과 public environment 변수가 여러 subsystem에서 사용된다는 이유만으로 별도 "환경변수 Thread"를 만드는 것은 신규 Thread 독립성 기준을 충족하지 않는다.
9. Secret 생성·저장·교체를 독립적인 Thread로 만들 만큼 secret manager, multi-key rotation, credential renewal, provider integration 등의 commit 집합은 존재하지 않는다.

따라서 기존 Thread 체계를 변경하지 않고 supplement packet만 추가하는 것이 가장 작은 보완이다.

---

# Part IV — Project-Level External Steps

## 채택 항목 없음

Repository evidence로 확인된 중요한 외부 단계는 모두 Thread 04, 06, 07, 08, 21, 22, 23에 자연스럽게 귀속되었다. 특정 Thread에 귀속할 수 없으면서도 별도로 기록해야 하는 project-level external step은 발견되지 않았다.

## 검토했지만 Gap으로 채택하지 않은 항목

### OAuth/OIDC/SAML Provider 등록 및 Redirect URI

Production에는 OAuth나 별도 인증 provider가 포함되어 있지 않으며, 이 저장소만으로는 새 production session을 만들 수 없다고 명시되어 있다. 따라서 "외부 provider를 등록해야 한다"가 아니라 **provider integration 자체가 구현되지 않은 상태**다. 이를 수행된 또는 필요한 외부 등록 단계로 변환하지 않았다.

### DNS·Domain Verification

Repository에는 특정 domain, DNS record 또는 domain verification artifact가 없다. G-08의 HTTPS/WSS는 secure guest cookie와 remote browser 배포에서 추론되는 transport requirement이지만, 특정 DNS provider나 domain verification 절차까지 요구된다고 판단할 근거는 없다.

### Caddy의 자동 TLS Certificate

현재 Caddy 설정은 `:8080` HTTP listener이며 domain-based HTTPS site block이 없다. 따라서 certificate가 실제로 발급되었거나 Caddy가 자동 TLS를 수행했다고 서술하지 않았다.

### Webhook·Object Storage·Bucket·IAM·Scheduler

현재 재귀 repository tree와 integration source에서 webhook registration, object storage, bucket, IAM policy, scheduler 또는 cron provider 연결을 찾을 수 없다. 일반적인 웹 서비스에서 사용할 수 있다는 이유만으로 Gap에 추가하지 않았다.

### Backup/Restore

PostgreSQL named volume은 존재하지만 backup, restore, snapshot, dump 또는 recovery script는 없다. 영속 volume의 존재만으로 수행된 backup이나 필수 backup 절차를 추정하지 않았다.

### Cloud Production Provisioning 및 실제 Deployment

Compose와 production image는 존재하지만 Terraform, Kubernetes manifest, cloud provider deployment 설정 또는 production deploy workflow는 없다. 실제 production/staging environment가 존재한다고 판단할 수 없으며, container resource의 실제 생성은 G-01과 Thread 22 범위에만 기록했다.

### CI/CD Secret 또는 GitHub Environment 설정

현재 workflow에는 `${{ secrets.* }}` 또는 GitHub Environment를 요구하는 production deploy 단계가 없다. Workflow의 password와 session secret은 transient CI 검증용 environment 값으로 정의되어 있다. 따라서 별도의 GitHub Actions secret 등록이 실제로 필요하거나 수행되었다고 판단하지 않았다.

---

## 최종 Documentation Action

기존 Thread 체계를 유지하면서 다음 7개 supplement만 추가하는 것이 최소 보완안이다.

| Existing Thread | 추가할 External-State Supplement |
| --- | --- |
| Thread 04 | DB provisioning 전제, migration 실제 실행, seed profile 선택, test reset 대상  |
| Thread 06 | `005_expire_legacy_sessions` 실행에 따른 session 일괄 무효화 |
| Thread 07 | 명시적 관리자 role bootstrap 및 관리자 lockout 복구  |
| Thread 08 | Guest signing secret 생성·주입·교체  |
| Thread 21 | Health probe 활성화와 내부 metrics collector 연결  |
| Thread 22 | Compose resource·volume materialization, public URL build coupling, 조건부 TLS |
| Thread 23 | Toxiproxy 상태 lifecycle, disposable verification DB, CI transient resource와 artifact |

이 보완안은 기존 Development Thread의 제목이나 commit 소유 구조를 변경하지 않으면서, Git/source 중심 학습만으로는 빠질 수 있는 **실제 시스템 성립 단계와 그 비관찰 경계**를 추가한다.
