# External-State Development Gap Audit

## `audience-foundry-travel-readiness`

### 감사 기준

감사는 첨부된 20개 Development Thread Index를 **확정된 구조**로 사용했으며, 기존 Thread의 제목·commit 구성을 재최적화하지 않았다. 

Repository 쪽은 현재 단일 `main` 브랜치의 HEAD `0aef24b0fd3bb0de81f5e645dc534993accf227c`를 기준으로 현재 트리, 200개가 넘는 직선 commit history, migration, management command, 운영 스크립트, release·backup·acceptance 자료와 관련 과거 diff를 대조했다. 현재 원격에는 `main` 하나만 존재한다.

이 보고서에서는 tracked 문서에 “실행했다”는 기록이 있더라도 다음과 같이 해석했다.

* **Directly observed in repository:** 코드·설정·migration·명령 또는 실행을 주장하는 receipt 문서가 실제로 존재함.
* **Required/inferred from repository:** 구현을 성립시키려면 외부에서 해당 단계가 필요함.
* **Actual execution not observable from Git:** disposable DB가 삭제되었거나 산출물이 `/output/`에 있고 Git에서 제외된 경우, Git은 해당 실행의 실제 대상·값·시점·완전성을 독립적으로 증명하지 못함.

현재 완료 보고서도 프로젝트를 production이 아닌 **actual-data local integration candidate**로 규정하며, production DB·scheduler·platform·credential·domain·DNS·deployment·traffic switch를 미실행 경계로 남긴다.

## 판정 요약

총 **19개 Gap**을 채택했다.

* `EXISTING_THREAD`: 16개
* `NEW_THREAD`: 1개
* `PROJECT_LEVEL_EXTERNAL_STEP`: 2개

제안하는 신규 Thread는 하나다.

> **N01 — 역할 분리 PostgreSQL 커미셔닝과 전진 전용 마이그레이션 / Role-Separated PostgreSQL Commissioning and Forward-Only Migration**

운영자 control plane은 Thread 08·12, scheduler와 alerting은 Thread 17, runtime deployment와 public edge는 Thread 18, backup/restore는 Thread 20에 자연스럽게 귀속되므로 별도 신규 Thread로 만들지 않았다.

---

# Part I — Gap Index

| Gap ID  | 짧은 이름  | Classification / Primary Owner  | Related Threads | Repository Evidence 요약 | Required External Step 요약  | 실제 수행 여부 및 Documentation Action |
| --- | --- | --- | --- | --- | --- | --- |
| **G01** | Django runtime secret·host·release 구성  | `EXISTING_THREAD` — **Thread 01** | 18, N01 | 설정은 application secret, DB password, allowed hosts, exact release SHA를 요구하고 누락 시 시작을 거부한다. | 실제 secret 생성, managed store 등록, runtime identity에 주입, host·release·HTTPS 값 설정  | 값과 production 주입은 관찰 불가이며 production runtime은 미구성. **Thread 01 supplement 추가**  |
| **G02** | PostgreSQL service·role·schema commissioning | `NEW_THREAD` — **N01**  | 01, 02, 03, 06–12, 17, 18, 20 | PostgreSQL 18.6, migration/runtime 역할 분리, runtime DDL 거부, data-bearing migration의 forward-fix 경계가 구현되어 있다. | DB service와 database 생성, 역할·grant·credential 생성, migration plan/apply, seed 적용, compatibility와 cutover 검증  | 일부 disposable rehearsal 기록은 있으나 최종 exact-SHA live DDL rehearsal과 production migration은 확인되지 않음. **신규 Thread 작성**  |
| **G03** | 승인 source·rights registry 활성화  | `EXISTING_THREAD` — **Thread 02** | 03, 04, 06, 07, 10, 11, N01 | source는 DRAFT로만 생성되며 exact rights decision 뒤에만 ACTIVE가 된다. `register_approved_sources --apply`가 실제 DB row와 상태를 만든다.  | source 계약과 이용권한을 사람이 확인하고 `--check` 후 `--apply`; revision 변경 시 새 승인과 forward upgrade 수행  | local DB 적용 주장은 있으나 surviving DB나 production registry는 없음. **Thread 02 supplement 추가**  |
| **G04** | 공공데이터 provider key 발급·주입·회전  | `EXISTING_THREAD` — **Thread 04** | 03, 07, 10, 11, 17, 19  | 경보·항공 source가 `DATA_GO_KR_SERVICE_KEY`를 공유하며 legacy reference를 제한적으로 지원한다. 값은 저장하지 않는다.  | 해당 endpoint 권한이 있는 service key 발급, managed secret 등록, worker에만 주입, rotation/revocation 절차 마련 | key 값·발급 계정·승인 범위·회전 이력은 관찰 불가. local child-process 사용 주장만 존재. **Thread 04 supplement 추가**  |
| **G05** | 입국요건 actual ingestion 실행 | `EXISTING_THREAD` — **Thread 06** | 02, 03, 04, 08, 09, 17  | ingestion은 active source와 rights를 확인하고 redacted attempt, artifact, parse evidence와 typed revision을 만든 뒤 `REVIEW_REQUIRED` 또는 replay 결과를 낸다.  | 지원 국가별 실제 명령 실행, 성공·quarantine·terminal failure 분류, review queue 확인  | completion report의 local disposable 실행 주장은 있으나 DB가 삭제되어 독립 확인 불가. **Thread 06 supplement 추가** |
| **G06** | 여행경보 actual ingestion 실행 | `EXISTING_THREAD` — **Thread 07** | 02, 03, 04, 08, 09, 17  | 경보 ingestion은 credential을 process env에서 읽고 원문을 비보존한 채 complete snapshot 후보를 `REVIEW_REQUIRED`로 만든다.  | key를 주입한 worker로 국가별 수집, verified-empty 포함 결과 검증, review queue 전달  | local actual-data 주장은 있으나 production 실행·cadence는 확인 불가. **Thread 07 supplement 추가** |
| **G07** | 정기운항 actual collection·candidate staging | `EXISTING_THREAD` — **Thread 10** | 02, 04, 05, 11, 12, 17  | 이름이 `publish_scheduled_flights`인 명령도 실제로는 API를 호출해 durable evidence와 후보를 `staged` 상태로 만들 뿐 게시하지 않는다. | service key 주입, 대상 국가·기간별 collection 실행, schedule/reference completeness 확인, candidate staging | local candidate 생성 주장은 있으나 production collection은 확인 불가. **Thread 10 supplement 추가**  |
| **G08** | 공식 archive 수집·비행시간 산정 실행 | `EXISTING_THREAD` — **Thread 11** | 02, 04, 10, 12, 17  | worker는 공식 dataset의 변경을 확인하고 archive를 파싱·산정해 review-required 후보를 만든다. 성공은 자동 게시가 아니다.  | archive version 감시, 변경 시 download·parse·derivation 실행, rights revision과 receipt 확인 | local 공식 파일 처리 주장은 있지만 archive와 DB가 Git에 없어 독립 확인 불가. **Thread 11 supplement 추가** |
| **G09** | disposable live parser replay gate | `EXISTING_THREAD` — **Thread 03** | 02, 04, 06, 07, 19, N01 | final replay는 승인 source baseline, exact disposable PG 이름·safety token, canonical key reference, durable evidence와 empty publication pointer를 검사한다. | disposable PG 생성, migration·source registration, key 주입, live replay 실행, raw-free·determinism·cleanup 검증 | tracked receipt 주장은 있으나 DB가 삭제되어 actual execution은 독립 관찰 불가. **Thread 03 supplement 추가**  |
| **G10** | 입국·경보 operator identity와 실제 게시·롤백  | `EXISTING_THREAD` — **Thread 08** | 06, 07, 09, 17, 20  | publication service는 staff actor와 세분화된 권한을 요구하지만 Admin은 현재 URL에 mount되지 않고 모든 접근을 거부한다.  | identity-aware proxy/MFA, operator principal·권한 생성, 승인된 control surface 활성화, publish/reject/rollback 수행  | local 임시 게시 주장은 있으나 production proxy·계정·게시본은 없음. 현재 HEAD에서는 외부 provisioning만으로 Admin 사용 불가. **Thread 08 supplement 추가** |
| **G11** | 항공 operator identity와 실제 게시·롤백 | `EXISTING_THREAD` — **Thread 12** | 08, 10, 11, 13, 17, 20  | 항공 review lifecycle과 publish/reject/rollback 권한·current pointer가 구현되어 있다.  | MFA operator 계정과 항공 권한 생성, sealed candidate 검수, publish/rollback 수행  | local publication 주장은 있으나 production operator와 current pointer는 확인 불가. **Thread 12 supplement 추가**  |
| **G12** | 브라우저 acceptance 산출물·수동 접근성 확인  | `EXISTING_THREAD` — **Thread 16** | 14, 15, 18, 19  | 브라우저 harness가 viewport·state·semantic 검사를 수행하고 screenshot/report를 `/output/`에 만든다. `/output/`은 Git 제외 대상이다.  | exact candidate에 Chrome matrix 실행, report·PNG 보존, 사람이 시각 검수, 실제 VoiceOver smoke 수행 | 12 PNG와 manual review 주장은 있으나 산출물은 Git에 없고 VoiceOver는 remaining gate로 기록됨. **Thread 16 supplement 추가**  |
| **G13** | source scheduler·freshness alert·log transport | `EXISTING_THREAD` — **Thread 17** | 06–12, 18 | `check_freshness`는 flight·entry·warning 셋을 함께 검사하고 비정상이면 nonzero fixed receipt를 낸다. 운영 runbook은 module별 cadence와 alert receiver를 요구한다. | 외부 scheduler/job identity 등록, cadence 적용, alert receiver·owner·log retention 구성  | production scheduler·alerts·log retention은 명시적으로 미구성. **Thread 17 supplement 추가** |
| **G14** | release artifact promotion·runtime platform deployment | `EXISTING_THREAD` — **Thread 18** | 01, 16, 17, 19, N01 | build가 deterministic tar·manifest·digest를 `/output/`에 생성하고 runtime은 exact release SHA와 Gunicorn identity를 요구한다.  | immutable artifact 저장·promotion, platform·process·resource provision, environment 주입, preflight 후 exact artifact 시작  | local build/runtime rehearsal 주장은 있으나 artifact registry와 production platform은 없음. **Thread 18 supplement 추가** |
| **G15** | domain·DNS·TLS·traffic cutover | `EXISTING_THREAD` — **Thread 18** | 01, 17  | settings와 Gunicorn은 host/TLS 경계를 강제하고 runbook은 domain, DNS, certificate, edge header stripping, traffic switch를 외부 handoff로 남긴다. | domain 소유 확인, DNS/TLS 배치, reverse proxy 구성, exact host injection, traffic 전환·rollback  | 모두 production 미실행으로 명시됨. **Thread 18 supplement에 G14와 함께 추가** |
| **G16** | cross-surface sensitive-absence gate 실제 실행 | `EXISTING_THREAD` — **Thread 19** | 03, 14, 16, 18, 20  | gate는 Git 전체 blob, main/restored DB, artifact, runtime files·backup directory를 실제 secret·marker 대상으로 검사하고 fixed receipt만 출력한다. | 실제 candidate artifact·DB·restore·runtime log·backup을 준비하고 credential marker를 주입한 뒤 gate 실행·receipt 보존  | focused checks는 있으나 comprehensive final gate는 remaining item. **Thread 19 supplement 추가** |
| **G17** | production backup role·storage·retention·restore rehearsal | `EXISTING_THREAD` — **Thread 20** | 17, 18, 19, N01 | exact PG18.6 custom backup, read-only role, writer pause, encrypted storage, separate empty restore DB와 cleanup 검증이 정의되어 있다. | backup/restore roles·credentials provision, encrypted retention store, recurring backup, restore drill, RPO/RTO와 PITR 검증 | local disposable rehearsal 주장은 있으나 production backup store·retention·PITR은 미구성. **Thread 20 supplement 추가** |
| **G18** | GitHub remote 생성·push·공개 hosting | `PROJECT_LEVEL_EXTERNAL_STEP` | 18, 19  | tracked README/완료 보고서는 `remote none`·local only라고 적지만 현재 repository는 공개 GitHub remote로 존재한다. | 이미 관찰되는 remote 상태를 handoff 문서에 반영하고 exact pushed HEAD를 기록  | **현재 external state는 직접 관찰 가능**. 다만 생성 명령·credential·주체·절차는 Git으로 확인 불가. **Project handoff 보완** |
| **G19** | Wanted Sans 상업 배포 license sign-off | `PROJECT_LEVEL_EXTERNAL_STEP` | 15, 18  | 완료 보고서가 upstream 공개 이슈 해결 또는 상업 배포 license sign-off를 remaining gate로 남긴다.  | upstream 해결 확인 또는 배포권한의 외부 승인·증빙 확보  | 수행 여부 확인 불가. 독립 Thread를 뒷받침할 구현 commit 집합이 부족함. **Project release checklist 추가**  |

---

# Part II — Existing Thread Supplement Packets

## E01 — Existing Thread 01

### Thread Identity

* **Thread 01**
* **실패 폐쇄형 Django 애플리케이션 셸**
* **Fail-Closed Django Application Shell**
* **Gaps:** G01

### Repository Evidence

`2dce61ec3fb2da42f0deef1b6ee0855d93c9e810` — `build: add minimal Django project shell`

* 관련 파일: `travel_readiness/settings.py`, `manage.py`, `travel_readiness/wsgi.py`
* 관련 diff:

  * application secret과 DB password는 누락 시 `ImproperlyConfigured`를 발생시키는 required environment로 도입됐다.
  * PostgreSQL 외 DB fallback은 존재하지 않는다.
  * secure cookie, HTTPS redirect, HSTS와 redacted logging이 기본 경계로 들어왔다.

Final-state 설정은 여기에 exact `TRAVEL_READINESS_RELEASE_SHA`, fail-closed allowed hosts와 production security check를 추가로 요구한다.

### External Development Steps

1. `TRAVEL_READINESS_SECRET_KEY`에 사용할 application secret을 repository 밖에서 생성한다.
2. secret manager 또는 이에 준하는 관리형 저장소에 등록한다.
3. runtime identity에 application secret, DB connection reference, exact allowed hosts, release SHA와 HTTPS 관련 값을 주입한다.
4. 값이 shell history, dotenv, artifact 또는 log에 들어가지 않는지 확인한다.
5. 동일한 주입 상태에서 Django deployment check와 WSGI startup을 수행한다.

### Code Connection

설정 import 시 required 값이 없으면 process가 시작하지 않는다. 잘못된 host 또는 release identity는 deployment check와 runtime telemetry의 release binding을 통과하지 못한다.

### Evidence Boundary

* **Directly observed:** required environment 이름, validation 로직, Git에서 `.env`를 제외하는 정책.
* **Required/inferred:** 실제 secret 생성, managed-store 등록과 runtime injection.
* **Actual execution not observable:** 실제 값, secret manager 종류, production 주입 시점과 담당자.

### Ordering — Conceptual Execution Order

`secret 생성 → managed store 등록 → runtime identity 권한 부여 → non-secret config 입력 → deployment check → application 시작`

---

## E02 — Existing Thread 02

### Thread Identity

* **Thread 02**
* **권리 검토 기반 출처 수명 주기**
* **Rights-Governed Source Lifecycle**
* **Gaps:** G03

### Repository Evidence

`d83e894094e86ed6d6cc1a4488de0c159afffcf1` — `feat(sources): enforce the rights activation gate`

* 관련 파일: source activation migration, source lifecycle tests
* 관련 diff:

  * 신규 source는 DRAFT/disabled로만 생성된다.
  * DRAFT → RIGHTS_APPROVED → ACTIVE 순서를 강제한다.
  * exact current revision에 대한 최신 approval 없이는 activation을 거부한다.
  * rejection은 같은 transaction에서 source를 disable한다.
  * rollback은 rights가 없는 empty disposable DB에서만 허용된다.

`2977d092e58c6971ee92578c0711726a713e07a6` — `feat(sources): register approved source contracts`

* 관련 파일: `sources/management/commands/register_approved_sources.py`
* 관련 diff:

  * `--check`는 read-only outcome을 반환한다.
  * `--apply`는 source configuration과 rights decision을 만들고 ACTIVE로 전이한다.
  * PostgreSQL advisory transaction lock으로 동시 등록을 직렬화한다.

`2986879490bcc6185cc2653fc440668e2b836402` — `feat(sources): support reviewed contract revisions`

* prior rights를 보존하면서 새 revision을 DRAFT로 되돌린 후 새 approval과 activation을 수행한다.
* data-bearing 상태에서는 과거 결정을 삭제하거나 같은 revision을 재사용하지 않는다.

### External Development Steps

1. 공식 source locator, 수집 방식, 필드 범위, raw retention 금지와 republication 조건을 사람이 검토한다.
2. 검토 결과를 repository의 exact contract fingerprint와 대조한다.
3. migration 완료 후 production DB에서 `register_approved_sources --check`를 실행한다.
4. 예상 결과가 정확할 때만 migration owner 또는 승인된 bootstrap identity로 `--apply`를 실행한다.
5. source contract 변경 시 기존 row를 직접 덮어쓰지 않고 새 revision과 새 권리 결정을 적용한다.
6. rejection 또는 provider 조건 변경 시 source를 pause/reject하고 이후 collection을 중단한다.

### Code Connection

Thread 06·07·10·11의 모든 ingestion은 source가 ACTIVE이고 exact rights가 최신일 때만 시작된다. 이 외부 등록이 없으면 수집 코드는 존재해도 source gate에서 닫힌다.

### Evidence Boundary

* **Directly observed:** 권리 schema, lifecycle trigger, registration·upgrade 명령.
* **Required/inferred:** 실제 권리 검토자, 외부 이용조건 검토와 production `--apply`.
* **Actual execution not observable:** production source rows, 적용 시각, 검토 자료와 실제 담당자.

### Ordering — Conceptual Execution Order

`외부 이용조건 검토 → contract fingerprint 확인 → DB migration → --check → --apply → ACTIVE 검증 → collection 허용`

---

## E03 — Existing Thread 03

### Thread Identity

* **Thread 03**
* **원문 비보존 증거 상태 머신과 결정적 재현**
* **Raw-Free Evidence State Machine and Deterministic Replay**
* **Gaps:** G09

### Repository Evidence

`c393abfe9ca95c5f17d55d4d7f6ff99109da7d5e` — `test(sources): verify live parser replay`

* 관련 파일: `operations/source_replay.py`, `operations/management/commands/check_live_parser_replay.py`
* 관련 diff:

  * approved live source를 한 번씩 호출한다.
  * 같은 body를 production parser로 두 번 처리해 동일 결과인지 확인한다.
  * receipt에는 source body, typed facts, locator와 secret 값을 넣지 않는다.

`084ca1f02c91ee720684b9a3602aa341c7a003e5` — `fix(ops): align live replay source boundary`

* canonical `DATA_GO_KR_SERVICE_KEY`와 제한된 legacy fallback을 사용한다.
* 현재 승인 source registry 전체를 baseline으로 확인한다.
* exact disposable DB, empty publication pointer, 허용된 typed airport links와 raw-free storage boundary를 검사한다.
* wrapper는 admin password와 provider key를 environment에서 제거한 뒤 필요한 child process에만 전달한다.

### External Development Steps

1. PostgreSQL 18.6에 규칙에 맞는 disposable replay database를 만든다.
2. exact release의 migration을 적용한다.
3. approved source registry를 `--apply`한다.
4. replay admin credential과 provider key를 parent environment에 잠시 넣되 wrapper가 즉시 제거하도록 한다.
5. exact safety token으로 replay command를 실행한다.
6. fixed success receipt, durable attempt·artifact·parse-run 수, typed revision 수와 empty pointer를 확인한다.
7. raw response·secret·publication이 남지 않았는지 검사한다.
8. database와 transient credential을 제거한다.

### Code Connection

Replay는 Thread 06·07 ingestion의 production code path를 호출한다. 단순 parser fixture가 아니라 실제 endpoint response를 transient memory에서 처리하고 durable redacted evidence만 남긴다.

### Evidence Boundary

* **Directly observed:** disposable-target guard, safety token, success/failure receipt 형식과 cleanup wrapper.
* **Required/inferred:** 실제 disposable server, admin credential, provider key와 live endpoint 접근.
* **Actual execution not observable:** tracked report는 성공을 주장하지만 replay DB가 삭제됐으므로 underlying rows와 endpoint response는 확인할 수 없다.

### Ordering — Conceptual Execution Order

`disposable DB → migration → source registration → key 제한 주입 → live replay → receipt·storage 검증 → cleanup`

---

## E04 — Existing Thread 04

### Thread Identity

* **Thread 04**
* **출처 계약에 묶인 제한형 단일 호출 수집기**
* **Contract-Bound Bounded Single-Call Collectors**
* **Gaps:** G04

### Repository Evidence

`2977d092e58c6971ee92578c0711726a713e07a6`은 source registry에 locator가 아닌 **secret reference 이름**만 저장한다.

Final-state 계약은 다음 경계를 둔다.

* canonical key reference: `DATA_GO_KR_SERVICE_KEY`
* 제한된 legacy fallback: `MOFA_TRAVEL_ALARM_SERVICE_KEY`
* 경보·항공 요청이 같은 canonical key를 사용
* request fingerprint와 exception/log에서 key 값 제거
* HTTPS, timeout, response-size와 retry 경계 고정.

### External Development Steps

1. 공공데이터 provider에서 경보·정기운항 API 접근이 허용된 service key를 발급받는다.
2. repository나 source configuration row가 아닌 managed secret store에 저장한다.
3. 수집 worker identity에 read-only secret access를 부여한다.
4. application web process나 release build에는 key를 주입하지 않는다.
5. missing·invalid·revoked key 동작이 terminal authentication failure로 닫히는지 확인한다.
6. rotation 시 이전 key를 폐기하고 canonical reference의 외부 값을 교체한다.
7. incident 발생 시 key를 revoke하고 관련 source를 pause한다.

### Code Connection

Thread 07·10·11의 credential-bound collector는 key가 없으면 수집을 시작할 수 없다. Thread 03 replay와 Thread 19 absence gate도 동일 key reference를 외부 입력으로 요구한다.

### Evidence Boundary

* **Directly observed:** key reference 이름과 redaction 로직.
* **Required/inferred:** provider account, key 발급·승인, managed injection과 rotation.
* **Actual execution not observable:** 실제 key 값, 발급 계정, 허용 API 목록, 만료·회전 기록.

### Ordering — Conceptual Execution Order

`provider 접근 승인 → key 발급 → managed store → worker ACL → child-process 주입 → failure test → rotation/revocation 운영`

---

## E06 — Existing Thread 06

### Thread Identity

* **Thread 06**
* **입국요건 CSV의 엄격 파싱과 내구성 있는 수집**
* **Strict Entry CSV Parsing and Durable Ingestion**
* **Gaps:** G05

### Repository Evidence

`2697145715468174b5752f93d0f95536dc2ed993` — `feat(entry): ingest approved snapshot evidence`

* 관련 파일: `entry_requirements/ingestion.py`, entry ingestion command
* 관련 diff:

  * network call 전에 STARTED attempt를 durable commit한다.
  * active source와 exact current rights를 lock·검증한다.
  * 성공 body는 memory에서 hash·schema·typed projection 검증까지만 유지한다.
  * DB에는 attempt, hash·byte count, parse evidence와 typed fact만 저장한다.
  * 성공 상태는 `REVIEW_REQUIRED` 또는 동일 artifact의 `REPLAY_OBSERVED`다.

Final command는 country 단위 실행 경계를 가지며 게시를 수행하지 않는다.

### External Development Steps

1. G02 migration과 G03 source activation을 완료한다.
2. 해당 국가의 ingestion command를 worker에서 실행한다.
3. `REVIEW_REQUIRED` 또는 `REPLAY_OBSERVED` receipt를 수집한다.
4. retry exhaustion, parse quarantine, source gate failure를 alert 또는 operator queue로 전달한다.
5. review-required fact가 Thread 08의 검수 대상에 나타나는지 확인한다.
6. initial population 후에는 G13 scheduler가 동일 명령을 cadence에 맞게 호출하도록 한다.

### Code Connection

실행은 `FetchAttempt`, `SourceArtifact`, `ParseRun`, `EntryFactRevision`을 실제 PostgreSQL state로 만든다. 코드가 존재하는 것만으로는 이 row들이 생기지 않는다.

### Evidence Boundary

* **Directly observed:** ingestion transaction과 receipt.
* **Required/inferred:** live endpoint 접근과 국가별 command invocation.
* **Actual execution not observable:** completion report의 local actual-data receipt는 직접 관찰되지만 disposable DB가 삭제되어 row와 source body를 독립 확인할 수 없다.

### Ordering — Conceptual Execution Order

`migration/source activation → 국가별 ingestion → receipt 분류 → review queue 확인 → scheduler 편입`

---

## E07 — Existing Thread 07

### Thread Identity

* **Thread 07**
* **여행경보의 단일 사실에서 완전 국가 스냅샷으로의 진화**
* **Travel Warnings from Single Facts to Complete Country Snapshots**
* **Gaps:** G06

### Repository Evidence

`57575ed758f1bc09726d23b1eac7c48e6326ead2` — `feat(warnings): ingest approved alarm evidence`

* 관련 파일: `travel_warnings/ingestion.py`, warning ingestion command
* 관련 diff:

  * key를 named process environment에서만 읽는다.
  * raw response와 credential은 반환하거나 저장하지 않는다.
  * PostgreSQL advisory lock으로 국가별 중복 worker를 차단한다.
  * redacted attempts와 complete snapshot typed facts를 durable하게 기록한다.
  * 성공은 `REVIEW_REQUIRED` 또는 `REPLAY_OBSERVED`이며 게시가 아니다.

Final command도 국가 단위로 실행되고 fixed receipt만 출력한다.

### External Development Steps

1. G04 provider key를 ingestion worker에만 주입한다.
2. 국가별 actual warning ingestion을 실행한다.
3. complete snapshot의 항목 순서·건수 또는 verified-empty 상태를 operator가 공식 source와 대조할 수 있게 한다.
4. authentication, provider error, schema drift와 quarantine을 운영 경로로 전달한다.
5. review-required snapshot을 Thread 08 검수 단계로 넘긴다.
6. 이후 G13 scheduler cadence에 등록한다.

### Code Connection

실행 결과는 `TravelWarningRevision` 및 source evidence rows를 만들며, publication pointer는 Thread 08이 별도로 움직인다.

### Evidence Boundary

* **Directly observed:** key loader, retry·lock·persistence·receipt 경계.
* **Required/inferred:** 실제 key 주입과 source call.
* **Actual execution not observable:** local 실제 수집 주장은 있으나 production row·worker·endpoint response는 없음.

### Ordering — Conceptual Execution Order

`key 준비 → source activation → 국가별 ingestion → complete/empty 검증 → operator queue → scheduler 편입`

---

## E08 — Existing Thread 08

### Thread Identity

* **Thread 08**
* **입국요건·여행경보의 운영자 검수와 원자적 게시·롤백**
* **Operator Review, Atomic Publication, and Rollback for Entry and Warning Facts**
* **Gaps:** G10

### Repository Evidence

`8f85a698430ea29e2dae0155a2c4dbf02ffb0200` — `feat(reviews): publish approved facts atomically`

* publication service가 active staff actor와 exact permission을 요구한다.
* review decision, immutable publication revision, current pointer와 audit event가 한 transaction에서 닫힌다.

Final-state operator surface:

* `reviews/admin.py`는 모든 permission을 거부한다.
* `get_urls`도 approved MFA proxy가 준비될 때까지 실패한다.
* `travel_readiness/urls.py`는 Admin을 mount하지 않는다.

### External Development Steps

1. identity-aware reverse proxy를 provision하고 MFA를 강제한다.
2. proxy가 외부 client의 identity assertion header를 제거하고 자체 검증 assertion만 전달하도록 한다.
3. named operator principal을 만들고 Django staff status와 entry/warning publish·reject·rollback permission을 부여한다.
4. 해당 보안 경계를 전제로 Admin 또는 동등한 승인 control surface를 mount한 release를 배포한다.
5. review-required facts를 공식 source와 사람이 대조한다.
6. approve/reject를 기록하고 publish를 실행해 current pointer와 audit event를 확인한다.
7. stale·오게시 시 rollback을 실행하고 새 audit event와 pointer version을 검증한다.

### Code Connection

코드는 권한을 가진 actor가 존재해야 publication service를 실행한다. 현재 HEAD에서는 web Admin이 의도적으로 비활성화되어 있으므로 **외부 proxy/MFA provision만으로는 충분하지 않으며**, 이후 승인된 mount/configuration release도 필요하다.

### Evidence Boundary

* **Directly observed:** 권한 모델, transaction과 unmounted Admin.
* **Required/inferred:** IdP/MFA/proxy, 실제 operator accounts와 권한 부여.
* **Actual execution not observable:** production actor, login, 실제 publish·rollback과 현재 pointer.

### Ordering — Conceptual Execution Order

`MFA/IdP → proxy/header boundary → operator principal → 승인된 surface release → review → publish → rollback rehearsal`

---

## E10 — Existing Thread 10

### Thread Identity

* **Thread 10**
* **순서 보존 항공 증거 수집과 계약 버전 관리**
* **Ordered Aviation Evidence Collection and Contract Versioning**
* **Gaps:** G07

### Repository Evidence

`38b28c1a5e34543b32dffc83e866b53d07ff367a` — `feat(source): publish scheduled flight evidence`

* 지원 국가·공항 seed와 항공 evidence·pointer 관련 migration을 도입한다.
* commit 제목의 “publish”와 달리 final collection command는 current publication을 바꾸지 않는다.

`travel_windows/management/commands/publish_scheduled_flights.py`:

* provider key를 로드한다.
* scheduled departure/arrival과 reference를 수집한다.
* sealed candidate를 만들고 `result=staged`를 출력한다.

### External Development Steps

1. G03의 aviation source contracts를 활성화한다.
2. G04 key를 aviation worker에 주입한다.
3. 대상 국가와 schedule period별 collection command를 실행한다.
4. departure·arrival completeness, order, codeshare와 reference receipt를 확인한다.
5. candidate가 sealed/staged 상태인지 확인하고 Thread 12 queue로 전달한다.
6. recurring refresh는 G13 scheduler에 등록한다.

### Code Connection

실행은 source attempts, schedule evidence와 review candidate를 실제 DB에 만든다. public recommendation은 current publication만 읽으므로 staging만으로 사용자 결과는 변하지 않는다.

### Evidence Boundary

* **Directly observed:** 수집·staging command와 durable evidence 모델.
* **Required/inferred:** actual API invocation, period selection과 worker execution.
* **Actual execution not observable:** local staging 주장은 있으나 production schedule generation과 current API response는 확인 불가.

### Ordering — Conceptual Execution Order

`source/key 준비 → schedule collection → completeness 검증 → sealed candidate → operator review`

---

## E11 — Existing Thread 11

### Thread Identity

* **Thread 11**
* **공식 아카이브 기반 노선별 비행시간 산출**
* **Route Duration Derivation from Official Archives**
* **Gaps:** G08

### Repository Evidence

`61e98a1bec5498c7428ed41e14ad25cfff92b2f4` — `docs: define actual data readiness contract`

* 공식 dataset page에서 현재 attachment를 찾는다.
* archive hash가 바뀔 때만 다시 해석한다.
* raw bytes는 memory에서만 사용하고 source hash·size·typed derivation만 남긴다.
* 산정 성공도 자동 게시가 아니라 review-required candidate다.

`travel_windows/management/commands/collect_route_duration_reference.py`는 변경이 없으면 no-op receipt를, 변경이 있으면 review-required derivation을 stage한다.

### External Development Steps

1. dataset page에 대한 anonymous HTTPS 접근 가능성을 확인한다.
2. source contract의 현재 rights revision을 활성화한다.
3. 외부 worker에서 archive-check command를 실행한다.
4. attachment가 변경된 경우 download·nested archive parsing·route derivation을 수행한다.
5. dataset date, source hash, parser·derivation contract와 표본·제외 건수를 확인한다.
6. 결과를 Thread 12의 항공 candidate에 결합한다.
7. dataset page가 login, CAPTCHA 또는 예상하지 못한 redirect를 요구하면 우회하지 않고 source를 중단한다.

### Code Connection

항공 candidate는 schedule evidence와 approved duration receipt가 모두 있어야 검수 가능하다. archive worker가 실행되지 않으면 code가 있어도 새 duration revision은 생성되지 않는다.

### Evidence Boundary

* **Directly observed:** dataset identity, derivation 계약과 collection command.
* **Required/inferred:** 실제 dataset page inspection·download와 worker 실행.
* **Actual execution not observable:** local archive 처리 주장은 있으나 archive, source hash 대상과 DB rows가 Git에 없다.

### Ordering — Conceptual Execution Order

`source rights → archive check → 변경 감지 → parse·derive → receipt 검증 → flight candidate 결합`

---

## E12 — Existing Thread 12

### Thread Identity

* **Thread 12**
* **봉인된 항공 후보의 검수·게시·롤백**
* **Sealed Flight Candidate Review, Publication, and Rollback**
* **Gaps:** G11

### Repository Evidence

`70e47ab69a55f50ee5ecb8413917b92433e7ad59` — `feat(domain): model flight review lifecycle`

* 항공 review decision, current publication pointer와 audit lifecycle을 도입한다.
* publish·reject·rollback permission을 분리한다.

`travel_windows/admin.py`는 sealed candidate에 대한 publish·reject와 prior publication rollback action을 정의하고 staff permission을 검사한다.

### External Development Steps

1. G10과 같은 identity-aware proxy·MFA 경계를 준비한다.
2. 항공 전용 operator principal과 publish/reject/rollback 권한을 부여한다.
3. schedule와 route-duration receipt가 같은 candidate에 결합되었는지 확인한다.
4. 공식 운항 사실과 산정 근거를 사람이 대조한다.
5. publish를 실행하고 current flight pointer와 generation을 확인한다.
6. 잘못된 publication 또는 stale source 상황에서 rollback을 rehearsal한다.

### Code Connection

Thread 13의 ranking은 published flight pointer를 읽는다. 실제 operator action이 없으면 staged candidate는 recommendation 결과에 참여하지 않는다.

### Evidence Boundary

* **Directly observed:** review·permission·pointer·rollback 코드.
* **Required/inferred:** operator 계정, MFA, 실제 검수와 action invocation.
* **Actual execution not observable:** local temporary publication 외 production publication과 actor는 확인되지 않는다.

### Ordering — Conceptual Execution Order

`operator security → candidate completeness → human review → publish → read-model 확인 → rollback rehearsal`

---

## E16 — Existing Thread 16

### Thread Identity

* **Thread 16**
* **재현 가능한 브라우저 인수 검증**
* **Reproducible Browser Acceptance Verification**
* **Gaps:** G12

### Repository Evidence

`554dccce21603e76afca386368a8738beb7b96eb` — `test(frontend): align live warning browser contract`

* browser harness가 module별 ready·stale·server-error 격리, verified-empty warning, multiple warning facts와 source links를 검증한다.
* 실제 publication shape을 test scenario에 복제해 semantic·visual boundary를 확인한다.

Repository는 `/output/`, screenshot과 acceptance report를 추적하지 않는다.

완료 보고서는 Chrome 4 viewport × 3 state의 12 PNG와 manual review를 주장하지만, VoiceOver smoke는 남은 gate로 기록한다.

### External Development Steps

1. exact release candidate와 exact browser/runtime version을 준비한다.
2. ready·empty·stale/error state matrix를 모두 실행한다.
3. JSON report와 PNG screenshot을 immutable evidence 위치에 보존한다.
4. 사람이 clipping, overlap, Korean typography, official link, disclosure와 state isolation을 검수한다.
5. macOS VoiceOver 등 실제 assistive technology로 핵심 입력·결과·source history 흐름을 smoke-test한다.
6. release SHA·browser version·scenario digest와 승인자를 포함한 비민감 receipt를 남긴다.

### Code Connection

Harness는 public request path가 외부 source를 호출하거나 DB를 쓰지 않는지도 검사한다. 그러나 실제 browser rendering과 assistive technology 동작은 code diff 자체가 증명할 수 없다.

### Evidence Boundary

* **Directly observed:** harness, scenario matrix와 output 경로.
* **Required/inferred:** 실제 browser process, 화면 산출물과 사람의 시각·접근성 검수.
* **Actual execution not observable:** PNG/report가 Git에 없으며 VoiceOver 완료 증거도 없다.

### Ordering — Conceptual Execution Order

`candidate 배포 → automated browser matrix → artifact sealing → manual visual review → assistive-tech smoke → receipt 승인`

---

## E17 — Existing Thread 17

### Thread Identity

* **Thread 17**
* **제한된 상태 신호·신선도 감시·비식별 텔레메트리**
* **Bounded Health Signals, Freshness Monitoring, and Redacted Telemetry**
* **Gaps:** G13

### Repository Evidence

`14d801688068908aa4d6bbf30b64c3fe83e89708` — `test(ops): align three-module freshness contract`

* `flight`, `entry`, `warning`을 독립적으로 평가한다.
* 하나의 module failure가 다른 module state를 덮지 않는다.
* 비정상 상태에서는 detail-free fixed alert receipt와 nonzero exit를 반환한다.

운영 runbook은 개념적으로 다음 cadence를 둔다.

* entry: 24시간 cadence, 최대 36시간
* warning: 6시간 cadence, 최대 8시간
* flight: 7일 cadence, 최대 8일

또한 scheduler, alert receiver, owner와 log retention을 production handoff 대상으로 남긴다.

### External Development Steps

1. 수집 worker와 freshness checker에 사용할 non-interactive identity를 만든다.
2. module별 ingestion/collection command를 외부 scheduler에 등록한다.
3. concurrent run을 허용하지 않도록 timeout·overlap 정책을 구성한다.
4. 각 job 뒤에 `check_freshness`를 실행하거나 별도 고빈도 checker를 등록한다.
5. nonzero receipt를 실제 alert receiver에 연결한다.
6. alert owner, escalation, acknowledgement와 recovery procedure를 지정한다.
7. redacted structured log의 transport와 retention을 구성한다.

### Code Connection

코드는 fixed receipt를 출력할 뿐 job을 스스로 예약하거나 paging service로 전송하지 않는다. scheduler와 alert transport가 없으면 freshness 상태는 자동으로 운영자에게 도달하지 않는다.

### Evidence Boundary

* **Directly observed:** commands, state aggregation, fixed receipt와 runbook cadence.
* **Required/inferred:** scheduler provider, job identity, alert receiver와 log platform.
* **Actual execution not observable:** production job, cadence, alert delivery·acknowledgement 이력.

### Ordering — Conceptual Execution Order

`job identity → collection schedules → freshness schedule → alert receiver → log retention → failure·recovery rehearsal`

---

## E18 — Existing Thread 18

### Thread Identity

* **Thread 18**
* **결정적 릴리스 산출물과 재현 가능한 런타임**
* **Deterministic Release Artifacts and Reproducible Runtime**
* **Gaps:** G14, G15

### Repository Evidence

`7b42a6aa87a42dd985959334edde9ed4d036b414` — `build(release): create deterministic artifact`

* `travel-readiness-release.tar`, manifest와 SHA-256 receipt를 만든다.
* 두 build가 byte-identical이어야 한다.
* tracked source, migrations, static, runtime와 dependencies를 manifest로 봉인한다.
* parent environment의 provider key와 임의 secret이 build로 넘어가면 실패한다.

`eb10145b3865c1dfa18c614639e895da2d8b8999` — `build(web): pin the production process`

* Gunicorn process를 고정하지만 platform worker·resource와 public edge는 외부 경계로 남긴다.

`3a67e65ac8d2f83dbd2bef4624a7cc622a746594` — `feat(release): bind runtime bootstrap identity`

* runtime telemetry에 exact release SHA를 넣고 bootstrap runtime tree identity를 검증한다.

Final runtime은 exact host, release SHA와 security flag를 요구하고 forwarded headers를 기본 신뢰하지 않는다. TLS는 direct cert/key pair 또는 external edge가 필요하다.

### External Development Steps

#### G14 — Artifact Promotion and Runtime

1. clean exact HEAD에서 deterministic build를 실행한다.
2. artifact digest와 manifest를 검증한다.
3. tar·manifest·digest를 immutable artifact store/registry에 업로드한다.
4. Linux production platform, process type, resource limits와 network policy를 provision한다.
5. G01 runtime config와 G02 DB credentials를 주입한다.
6. G02 migration cutover 후 exact digest의 artifact만 배포한다.
7. deployment checks와 readiness를 통과한 뒤 process를 시작한다.

#### G15 — Public Edge

1. domain 소유와 사용할 hostname을 확정한다.
2. DNS record와 TLS certificate를 발급·배치한다.
3. reverse proxy가 client-supplied forwarded header를 제거하도록 한다.
4. application allowed hosts와 public URL을 exact hostname에 맞춘다.
5. health/readiness가 통과한 뒤 traffic을 새 release로 전환한다.
6. 문제가 생기면 previous artifact 또는 새 restore DB로 traffic을 되돌린다.

### Code Connection

Artifact는 자체적으로 배포되지 않고 runtime process도 domain/TLS를 생성하지 않는다. exact release SHA, host와 edge 정책이 일치하지 않으면 application startup 또는 deployment check가 실패한다.

### Evidence Boundary

* **Directly observed:** deterministic builder, runtime process config, deployment checks와 edge requirements.
* **Required/inferred:** artifact registry, production compute, domain, DNS, certificate, proxy와 traffic mechanism.
* **Actual execution not observable:** local build/rehearsal receipt는 있으나 artifact 자체는 ignored이며 production platform·traffic은 미실행이다.

### Ordering — Conceptual Execution Order

`deterministic build → artifact seal/promotion → platform provision → DB migration → runtime config → deploy/start → DNS/TLS edge → traffic cutover → rollback readiness`

---

## E19 — Existing Thread 19

### Thread Identity

* **Thread 19**
* **Git·DB·산출물·런타임을 가로지르는 민감정보 부재 검증**
* **Cross-Surface Sensitive-Data Absence Verification**
* **Gaps:** G16

### Repository Evidence

`001ee495989e219fb0399b701619ac47c93248cc` — `ops: add silent sensitive-absence release gate`

* 관련 파일: sensitive-absence scanner와 release-manifest validator
* 관련 diff:

  * Git blob 전체, main/restored DB, artifact tree, runtime roots/files와 backup directory를 검사한다.
  * 실제 secret, trip marker와 raw-body marker를 encoded variant·fragment까지 탐색한다.
  * path, match, fragment, SQL result 또는 exception detail을 출력하지 않는다.
  * receipt는 `git/db/artifact/runtime` 성공·실패 상태만 갖는다.

Final CLI는 canonical·legacy provider key, main/restored DB password와 marker를 environment에서 제거한 후 scanner에 전달한다.

### External Development Steps

1. exact candidate repository와 release artifact를 준비한다.
2. representative actual-data main DB와 Thread 20으로 복구한 restored DB를 준비한다.
3. runtime log, generated files와 backup directory를 읽기 전용 검사 대상으로 지정한다.
4. 실제 credential을 외부 environment에 넣되 scanner가 즉시 제거하도록 한다.
5. 충분히 구별되는 synthetic trip/raw markers를 준비한다.
6. exact safety token과 대상 digest로 gate를 실행한다.
7. fixed receipt만 보존하고 target 값이나 match detail은 보존하지 않는다.
8. 한 surface라도 실패하면 release를 중단하고 수정 후 전체 gate를 다시 실행한다.

### Code Connection

정적 source 검사만으로는 DB, restored DB, built artifact, logs와 backup에 민감정보가 없는지 증명할 수 없다. 해당 external artifacts가 실제로 존재해야 scanner의 전체 범위가 실행된다.

### Evidence Boundary

* **Directly observed:** scanner 범위, bounds, safety token과 detail-free receipt.
* **Required/inferred:** 실제 DB·artifact·runtime·backup과 credential marker.
* **Actual execution not observable:** 완료 보고서는 comprehensive gate를 남은 항목으로 명시하며 final receipt도 repository에 없다.

### Ordering — Conceptual Execution Order

`candidate surfaces 준비 → credentials/markers 제한 주입 → read-only scan → fixed receipt → 실패 시 수정 → 전체 재실행`

---

## E20 — Existing Thread 20

### Thread Identity

* **Thread 20**
* **PostgreSQL 정확 백업·복구 리허설**
* **Exact PostgreSQL Backup and Restore Rehearsal**
* **Gaps:** G17

### Repository Evidence

`e246ae33d7e1b45ce24e259c1e81f21051f7ec23` — `test(operations): rehearse exact backup restore`

* PostgreSQL 18.6 custom-format backup, exact schema/data integrity manifest와 separate restore target을 도입한다.
* backup role은 read-only로 preprovision되어야 하며 script가 역할을 자동 생성하지 않는다.
* writer pause, encrypted controlled storage, RPO 24시간·RTO 4시간과 외부 scheduler/alert 경계를 문서화한다.

Final runbook은 다음을 요구한다.

* repository 밖의 mode `0600` credential file
* 별도 backup·restore 역할
* restore 전 empty database
* 복구 후 exact integrity 검증
* disposable target·roles·credentials cleanup
* production에서는 provider PITR과 encrypted retention 검증.

### External Development Steps

1. 최소 권한 backup role과 별도 restore role을 생성한다.
2. credential을 repository 밖의 관리형 저장소 또는 제한된 password file로 배치한다.
3. encrypted backup destination과 retention·deletion policy를 만든다.
4. writer를 pause하고 source DB의 exact custom backup을 생성한다.
5. archive·manifest digest를 기록하고 backup store에 업로드한다.
6. 별도의 empty database에 restore한다.
7. schema, constraints, trigger functions, sequence state, table digests와 publication pointers를 비교한다.
8. restored application smoke test를 실행한다.
9. target DB·temporary role·credential이 제거됐는지 독립적으로 확인한다.
10. backup cadence, alerting, PITR와 RPO/RTO rehearsal을 운영한다.

### Code Connection

Backup·restore script는 기존 PostgreSQL state를 입력으로 요구하며 storage와 role을 스스로 provision하지 않는다. Thread 19의 comprehensive gate도 복구된 DB와 backup directory를 대상으로 필요로 한다.

### Evidence Boundary

* **Directly observed:** exact backup·restore contract와 validation code.
* **Required/inferred:** production roles, encrypted storage, scheduled backups와 PITR.
* **Actual execution not observable:** local disposable rehearsal 주장은 있으나 archive·DB가 삭제되었고 production store·retention은 미구성이다.

### Ordering — Conceptual Execution Order

`backup/restore roles → encrypted store → writer pause → backup → seal/store → empty restore DB → integrity·runtime 검증 → cleanup → cadence/PITR rehearsal`

---

# Part III — Proposed New Thread Packets

## N01 — Proposed New Thread

### Thread Identity

* **Proposed New Thread N01**
* **역할 분리 PostgreSQL 커미셔닝과 전진 전용 마이그레이션**
* **Role-Separated PostgreSQL Commissioning and Forward-Only Migration**
* **Gaps:** G02

### 신규 Thread 판정 이유

이 관점은 단순히 `DATABASE_URL` 또는 DB password를 설정하는 부수 단계가 아니다.

* PostgreSQL service 선택부터 database·role 생성, migration, fixed seed, grant, artifact compatibility와 traffic cutover까지 고유 lifecycle이 있다.
* migration owner와 runtime role이 서로 다른 권한·credential·실패 조건을 가진다.
* populated schema의 reverse migration이 거부되는 경우가 반복적으로 존재하며 forward-fix 또는 restore가 별도 복구 전략이다.
* 모든 source, publication, ranking, operation subsystem을 관통한다.
* Thread 18의 artifact/runtime deployment 및 Thread 20의 backup/restore와 연결되지만 어느 한쪽의 사소한 하위 단계가 아니다.
* 독립적인 representative commit 집합을 선정할 수 있다.

Thread 20에 포함된 commit을 근거로 재사용하지만, 기존 Thread 20의 제목이나 commit 소유를 변경하지 않는다. Thread 20은 **기존 상태의 정확한 보존·복구**를, N01은 **상태의 최초 성립과 schema evolution/cutover**를 소유한다.

### Representative Commits

### 1. `9dfa352aab2b749fdc5821ea8c72535c05a35b09`

`build: pin travel readiness runtime`

* 관련 파일: runtime version baseline, dependency lock
* 중요 이유:

  * PostgreSQL 18.6, Python·Django·psycopg baseline을 고정한다.
  * runtime package와 production execution environment가 외부 경계임을 드러낸다.

### 2. `816b56ca8c6798b7f63306099f99f3d5bf9a2fdb`

`test(database): rehearse separated runtime roles`

* 관련 파일: `scripts/check-database-roles`, database-role tests
* 관련 diff:

  * loopback-only disposable PostgreSQL에서 migration owner, runtime role과 database를 생성한다.
  * public 접근을 회수하고 runtime role에는 DML만 허용한다.
  * runtime의 DDL 및 migration 실행을 거부한다.
  * previous artifact와 forward schema compatibility를 검사한다.
  * 종료 시 database와 roles를 제거한다.

### 3. `a3d779f484093b2df593af4899228c95aecf3cfd`

`feat(sources): add immutable source rights decisions`

* 관련 파일: source rights migration, models, rollback tests
* 중요 이유:

  * empty disposable DB에서는 migration reverse를 검증하지만 data-bearing 환경에서는 history를 유지하고 source를 disable한 뒤 forward-fix하도록 명시한다.
  * schema rollback과 business-state rollback이 동일하지 않음을 대표한다.

### 4. `b65995fd511af84ffa19ec4c8c8c2e4ae33e0ddd`

`fix(migrations): preserve populated flight upgrades`

* 관련 파일: populated flight migration과 migration tests
* 중요 이유:

  * existing data가 있는 상태에서 immutability trigger를 통제된 순서로 교체하고 backfill한 뒤 guard를 복원해야 한다.
  * production migration이 단순 `migrate` 호출이 아니라 writer·guard·compatibility를 함께 다루는 cutover임을 보여준다.

### Final-State Configuration and Source Evidence

* application은 PostgreSQL backend만 지원하고 DB password를 required runtime environment로 요구한다.
* role rehearsal은 migration owner와 runtime DML role을 분리하며 runtime role의 DDL·migration을 실패시킨다.
* 운영 runbook은 production handoff에서 별도 migration/runtime/backup/restore 역할, migration plan, writer pause와 traffic control을 요구한다.
* 여러 migration은 empty DB에 한해서만 reverse를 허용하고 populated DB에서는 forward-fix를 요구한다.

### External Development Steps

1. PostgreSQL 18.6 service와 region·availability·storage profile을 선택한다.
2. required encoding·locale·collation profile로 application database를 생성한다.
3. migration owner와 runtime role을 별도로 만든다.
4. public schema/database 기본 권한을 회수하고 default privilege를 명시한다.
5. 두 역할의 credential을 서로 분리해 managed secret store에 등록한다.
6. exact release artifact와 migration leaf set을 검증한다.
7. 현재 schema와 `migrate --plan`, migration drift를 비교한다.
8. writer와 traffic을 migration contract에 맞게 pause한다.
9. migration owner credential로 migration을 적용한다.
10. migration에 포함된 country, airport, policy와 empty publication pointer seed가 정확한지 확인한다.
11. source registry bootstrap은 G03에 따라 별도로 실행한다.
12. runtime role이 필요한 SELECT/INSERT/UPDATE/DELETE를 수행하지만 DDL·migration은 수행하지 못하는지 검증한다.
13. previous artifact가 forward schema에서 비파괴적으로 시작·중지되는지 확인한다.
14. current artifact를 시작하고 readiness 후 traffic을 전환한다.
15. migration failure 시 blind reverse를 수행하지 않고 traffic을 철회한다. 데이터가 없는 경우에만 검증된 reverse를 사용하며, 그 외에는 forward-fix 또는 Thread 20 restore 경로를 사용한다.

### Code Connection

* 모든 Django model과 trigger는 migration 적용 전에는 존재하지 않는다.
* migration 안의 fixed seed는 application이 기대하는 immutable UUID와 empty publication pointer를 실제 DB에 만든다.
* G03 source activation은 migration 완료 후에만 수행할 수 있다.
* Thread 18 runtime은 schema compatibility가 확인된 뒤에만 안전하게 시작할 수 있다.
* Thread 20 restore 결과도 동일 migration/runtime role 경계에서 검증되어야 한다.

### Failure and Recovery Conditions

| Failure  | Required response  |
| --- | --- |
| PostgreSQL version·locale 불일치  | migration 중단, 올바른 empty DB 재생성 |
| migration plan에 예상하지 못한 leaf·drift | 배포 중단, artifact·source 재검토 |
| runtime role이 DDL 가능 | grant 회수 후 재검증; traffic 금지 |
| populated reverse가 guard에 의해 거부  | reverse 강행 금지; forward-fix 또는 restore  |
| migration 후 previous artifact 비호환  | traffic 전환 금지; migration 또는 compatibility release 수정 |
| seed identity 충돌 | 기존 DB를 덮어쓰지 않고 원인 분석 후 forward migration |
| current artifact readiness 실패  | previous artifact 또는 검증된 restore DB로 traffic 유지/복귀 |

### Evidence Boundary

* **Directly observed in repository**

  * PostgreSQL-only settings
  * role rehearsal script와 tests
  * migration graph, fixed seed와 trigger guards
  * populated migration compatibility와 reverse refusal
  * runbook의 migration role·cutover 계약
* **Required/inferred from repository**

  * production PostgreSQL provision
  * 역할·credential 생성
  * actual migration execution
  * writer pause, compatibility check와 traffic cutover
* **Actual execution not observable from Git**

  * provider와 instance identity
  * DB host·name·credential 값
  * production role grants
  * migration 적용 시점·operator
  * actual production schema와 seed state
  * actual cutover 또는 recovery

### Ordering — Conceptual Execution Order

`PG service → database/roles → credential injection → artifact/schema inspection → backup coordination → writer/traffic pause → migration → seed/source bootstrap → runtime-role verification → compatibility test → current runtime → traffic cutover`

---

# Part IV — Project-Level External Steps

## PL01 / G18 — GitHub Remote 생성·Push·공개 Hosting

### Repository Evidence

Tracked README와 완료 보고서는 final local candidate 시점에 remote가 없고 unpublished라고 기록한다.

그러나 현재 GitHub metadata에서는 repository가 public remote로 존재하고 `main`을 default branch로 제공한다. 따라서 **remote 생성과 push라는 external-state 변화 자체는 현재 직접 관찰된다.**

### Required External Step

이미 발생한 현재 상태를 다음 handoff 문서에서 정확히 반영해야 한다.

* remote repository identity
* pushed HEAD
* public/private visibility
* local-only 문서가 더 이상 current state가 아니라는 점

Repository가 요구하지 않는 branch protection, CI workflow 또는 release policy는 이번 Gap에 추가하지 않는다.

### Evidence Boundary

* **Directly observed:** 현재 public remote와 `main` HEAD.
* **Required/inferred:** tracked handoff 문서 갱신.
* **Not observable:** remote 생성 명령, 사용 credential, 정확한 operator와 push 절차.

### Documentation Action

Development Thread로 만들지 않고 **Project Handoff / Repository State** 보충 항목으로 기록한다.

---

## PL02 / G19 — Wanted Sans 상업 배포 License Sign-Off

### Repository Evidence

완료 보고서는 Wanted Sans와 관련해 upstream 공개 이슈의 해결 또는 상업 배포를 허용하는 license sign-off를 미완료 release gate로 명시한다.

### Required External Step

다음 중 repository의 배포 형태에 맞는 외부 증빙이 필요하다.

* upstream issue가 해소되어 사용·재배포 범위가 명확해졌음을 확인
* 적절한 라이선스를 별도로 확보
* 법무·권리 보유자로부터 상업 배포 가능 sign-off 확보
* 증빙 identity와 적용 font version을 release checklist에 기록

실제 계약 내용이나 license 조건은 repository만으로 추정해서는 안 된다.

### Evidence Boundary

* **Directly observed:** unresolved release gate가 문서에 존재함.
* **Required/inferred:** 상업 배포 전 외부 권리 확인.
* **Actual execution not observable:** 라이선스 취득·승인 여부와 계약 조건.

### Classification Rationale

runtime state와 연결된 release gate이지만, 자체 implementation lifecycle을 보여주는 representative commit 집합이 없고 특정 기존 Thread의 개발 관점을 완성하는 기술 단계도 아니다. 따라서 `NEW_THREAD`가 아닌 `PROJECT_LEVEL_EXTERNAL_STEP`으로 유지한다.
