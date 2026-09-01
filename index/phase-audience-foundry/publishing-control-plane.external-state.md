# External-State Development Gap Audit

## Audience Foundry Publishing Control Plane

## 감사 기준과 판정 요약

첨부된 Index의 11개 Development Thread를 확정된 구조로 사용했으며, 각 Thread의 기존 학습 문서나 해설서는 입력으로 사용하지 않았다. 

분석 범위는 다음과 같다.

* `main`의 현재 HEAD `ea22de83f36daed6ee9f60862390b1cbf8b232af`
* 루트 커밋부터 현재 HEAD까지의 전체 선형 이력
* 현재 recursive repository tree
* 현재·과거 source, configuration, test, operational documentation 및 committed evidence
* GitHub 저장소 자체의 현재 외부 상태

현재 tree에는 `.wp-env.json`, Decap 설정, WordPress/Public Sites adapter, 감사·승인·게시 코드와 테스트가 있지만, `.github/workflows`, migration, database schema, committed `.publisher` 실행 기록은 없다.

### 최종 분류

* 발견된 Gap: **11개**
* `EXISTING_THREAD`: **9개**
* `NEW_THREAD`: **0개**
* `PROJECT_LEVEL_EXTERNAL_STEP`: **2개**
* 보충 Packet이 필요한 기존 Thread: **01, 03, 05, 06, 08, 09, 10**

핵심 결론은 다음과 같다.

1. Decap 실행, 사람의 승인, 게시 잠금·감사 기록, 외부 renderer checkout, WordPress runtime 및 원격 draft는 모두 코드 밖 상태를 만든다.
2. 그러나 각각은 이미 확정된 Thread의 실제 실행 단계를 완성하는 것이므로 신규 Thread가 필요하지 않다.
3. 공통 dependency materialization과 private GitHub origin의 생성·rename은 독립 Thread보다는 프로젝트 수준 단계가 적합하다.
4. Cloudflare, DNS, public deployment, hosted WordPress account, production credential은 명시적으로 deferred 또는 non-goal이며 현재 구현을 성립시키기 위한 단계가 아니다.

---

# Part I — Gap Index

## ESG-01 — Publisher dependency tree materialization

* **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
* **Primary Owner:** Project level
* **Related Threads:** 03, 05, 06, 08, 10, 11
* **Repository Evidence:** Node `>=22 <25`, npm `11.4.2`, exact package versions와 `package-lock.json`이 존재한다. `node_modules/`는 Git에서 제외되며 Decap, `wp-env`, AJV 등의 실제 실행 파일은 설치 후에만 생긴다. `f49ed9c`는 lockfile integrity에 의한 `npm ci` materialization을 도입했다.
* **Required External Step:** 호환되는 Node/npm/Git 환경을 준비하고 repository root에서 locked dependency tree를 설치한다. 운영 문서는 `npm ci --ignore-scripts`를 사용한다. 설치 결과인 `node_modules`는 외부 로컬 상태다.
* **실제 수행 여부 확인 가능성:** 과거 커밋 메시지에는 테스트와 audit 실행 기록이 있지만, 현재 machine에 올바른 dependency tree가 materialize되어 있는지는 Git으로 확인할 수 없다.
* **Documentation Action:** Thread별로 반복하지 말고 프로젝트 공통 `Local Bootstrap Prerequisite` 한 곳에 기록한다.

---

## ESG-02 — Decap loopback editor와 local Git editorial state 활성화

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 03 — 로컬 Git 편집 워크플로 / Local Git Editorial Workflow
* **Related Threads:** 02, 04, 11
* **Repository Evidence:** `local_backend: true`, proxy backend, `main` branch, `editorial_workflow`, loopback proxy `127.0.0.1:8081`이 구성되어 있다. 별도의 static admin server는 기본적으로 `127.0.0.1:8080`에 bind한다. `cms:proxy`와 `cms:web`은 서로 다른 프로세스다.
* **Required External Step:** dependency 설치 후 proxy와 web server를 동시에 실행하고, loopback editor에서 편집하여 local Git branch/commit 상태를 만든 다음 검토·병합하고 프로세스를 종료해야 한다.
* **실제 수행 여부 확인 가능성:** `0010a75`의 commit message는 HTTP 200 smoke와 `local_git` 확인을 기록하지만, 현재 두 프로세스의 실행 여부나 과거 local editorial branch의 구체적 내용은 확인할 수 없다. 현재 원격에는 `main` 하나만 보인다.
* **Documentation Action:** Thread 03에 두 프로세스의 lifecycle, port ownership, local Git state 생성·병합·종료 단계를 보충한다.

---

## ESG-03 — Human review, TTY decision, approval event commit

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 05 — 불변 소스 승인 수명 주기 / Immutable Source Approval Lifecycle
* **Related Threads:** 01, 06, 11
* **Repository Evidence:** 승인 함수는 clean immutable SHA를 두 번 검사하고 별도 approval/rejection event를 생성한다. CLI는 TTY를 요구하고 `APPROVE <full SHA>` 또는 `REJECT <full SHA>`의 정확한 입력을 요구한다. 게시 gate는 event가 단순히 생성된 상태가 아니라 `HEAD`에 commit된 상태를 요구한다.
* **Required External Step:** 사람이 정확한 source SHA의 내용을 검토하고 interactive terminal에서 결정을 입력한 뒤, 생성된 `.publisher/events/approvals/*.json`을 검토·stage·commit해야 한다.
* **실제 수행 여부 확인 가능성:** TTY 입력은 interactivity만 증명한다. 실제 검토의 충실도, `actor` 문자열과 실제 사람의 일치, 승인자의 권한은 Git으로 확인할 수 없다. 현재 `main` tree에는 committed approval event가 없다.
* **Documentation Action:** Thread 05에 human checkpoint, actor identity boundary, event commit promotion 단계를 추가한다.

---

## ESG-04 — Publication lock/work state와 stale residue recovery

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 01 — 승인 기반 멱등 게시 파이프라인 / Approval-Gated Idempotent Publication Pipeline
* **Related Threads:** 06, 08, 09, 10
* **Repository Evidence:** `3226d97`은 deterministic receipt별 `.publisher/locks/<receipt-id>` 디렉터리를 만들고, 존재하면 `PUBLICATION_IN_PROGRESS`를 반환하며, 정상적인 control flow에서는 `finally`에서 제거한다. `.publisher/locks/`와 `.publisher/work/`는 Git에서 제외된다.
* **Required External Step:** 동시에 같은 publication을 실행하지 않아야 하며, 프로세스 강제 종료·host crash 이후에는 lock/work residue를 조사하고 retry 가능 여부를 판단해야 한다.
* **실제 수행 여부 확인 가능성:** ignored runtime state이므로 과거 lock이나 build attempt residue는 최종 Git diff에서 확인할 수 없다. 정상 종료 시 cleanup은 구현되어 있지만 abrupt termination 시 stale lock이 남을 수 있다는 것은 코드 구조로부터의 추론이다.
* **Documentation Action:** Thread 01에 `normal completion`, `concurrent rejection`, `abrupt interruption`, `operator recovery` 상태를 추가한다. 안전한 stale-lock 판정·제거 기준은 repository에 정의되어 있지 않다는 점도 명시한다.

---

## ESG-05 — Audit/receipt materialization과 partial transaction recovery

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 06 — 감사 증거와 원자적 게시 기록 / Audit Evidence and Atomic Publication Records
* **Related Threads:** 01, 05, 09
* **Repository Evidence:** 감사 event는 `wx` 방식의 append-only file로 생성된다. 성공 기록은 temporary directory 안에 `event.json`과 `receipt.json`을 쓴 뒤 receipt ID directory로 atomic rename된다. directory가 존재하지만 구성원 중 하나가 없으면 absent가 아니라 `PUBLICATION_TRANSACTION_INVALID`로 fail closed한다.
* **Required External Step:** 게시 후 생성된 event와 receipt pair를 검사하고 Git에 commit해야 한다. partial transaction이 발견되면 adapter를 다시 호출하기 전에 operator가 원인을 조사하고 복구 결정을 내려야 한다.
* **실제 수행 여부 확인 가능성:** 테스트는 temporary repositories에서 동작을 증명하지만 현재 `main`에는 generated `.publisher` event/receipt가 없다. 실제 성공 publication transaction의 수행 여부는 확인되지 않는다.
* **Documentation Action:** Thread 06에 materialization → validation → commit → retry/recovery 절차를 추가하고, repository가 partial-directory repair 명령을 제공하지 않는다는 한계를 기록한다.

---

## ESG-06 — Remote audit-history retention과 origin synchronization

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 06 — 감사 증거와 원자적 게시 기록 / Audit Evidence and Atomic Publication Records
* **Related Threads:** 03, 05, 11
* **Repository Evidence:** 승인 gate는 approval event가 현재 `HEAD`에서 조회되는지를 검사한다. 개발 규칙은 published history rewrite와 force-push를 금지하며, 운영 문서는 생성된 감사 자료를 commit한 뒤 origin과 clean HEAD를 맞추는 절차를 전제로 한다. 현재 저장소는 private이고 기본 branch는 `main`이다.
* **Required External Step:** approval·publication evidence를 private origin에 push하고, 감사에 사용된 SHA와 event가 계속 reachable하도록 원격 history를 보존해야 한다.
* **실제 수행 여부 확인 가능성:** 현재 private origin과 `main`은 관찰되지만, branch API는 `protected:false`를 반환한다. 따라서 non-rewrite invariant가 repository-host 정책으로 강제되는지, 별도 backup이나 retention이 존재하는지는 확인할 수 없다.
* **Documentation Action:** 특정 GitHub 기능을 필수로 가정하지 말고, “audited SHA reachability와 non-rewrite를 누가 어떻게 검증할 것인가”를 Thread 06의 외부 운영 책임으로 명시한다.

---

## ESG-07 — Frozen renderer checkout, repository identity migration, build-state materialization

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 08 — 동결 정적 렌더러 통합 경계 / Frozen Static Renderer Integration Boundary
* **Related Threads:** 07, 01, 04, 06
* **Repository Evidence:** runtime은 `PUBLISHER_PUBLIC_SITES_REPO`를 요구한다. adapter는 별도 renderer checkout의 exact HEAD와 tracked-clean 상태를 검사하고, renderer 밖의 `.publisher/work/public-sites/<attempt>`에 release/report를 만든 후 pinned `fnm`/Corepack/pnpm build를 실행한다.
* **Repository Evidence — recorded execution:** `3b94cad`는 detached renderer SHA `1717326…`, Node `24.19.0`, pnpm `11.22.0`, locked install, 64개 artifact와 output digest를 가진 provider-free viability report를 commit했다. 이는 public deployment가 아니라 local build evidence라고 명시한다.
* **Repository Evidence — identity change:** `9130c1c`는 renderer의 현재 이름과 historical alias를 모두 schema에 허용하도록 변경했고, `ea22de8`은 control-plane 문서와 package identity를 현재 repository family로 변경했다.
* **Required External Step:** 별도 renderer를 현재 이름으로 가져오거나 기존 checkout을 유지하고, 정확한 SHA로 detach하며, tracked-clean 상태와 locked dependencies를 준비하고 환경변수에 checkout path를 주입한 뒤 build를 실행한다.
* **실제 수행 여부 확인 가능성:** one-time viability build를 보고하는 committed artifact는 있다. 그러나 현재 external checkout, installed renderer dependencies, `apps/site-a/out`, ignored attempt directory의 존속은 확인할 수 없다. full approval→publication→receipt loop가 이 report로 증명되는 것도 아니다.
* **Documentation Action:** Thread 08에 외부 checkout bootstrap, repository rename compatibility, exact SHA validation, build output lifecycle을 보충한다. Hosting·DNS·CDN deployment는 추가하지 않는다.

---

## ESG-08 — Docker-backed `wp-env` runtime provisioning과 teardown

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 10 — WordPress 런타임과 전송 계층 격리 / WordPress Runtime and Transport Isolation
* **Related Threads:** 09, 01, 04
* **Repository Evidence:** `.wp-env.json`은 WordPress 7.1, PHP 8.3, port 8888, local environment를 고정한다. 이후 `b44d2c6`은 tests environment를 비활성화해 하나의 development WordPress runtime만 시작하도록 수정했다. default transport는 `wp-env`이며 WP-CLI를 container 안에서 실행한다.
* **Required External Step:** Docker daemon/Desktop을 사용할 수 있게 하고 `wp-env start`가 필요한 WordPress·database runtime을 생성하도록 한 뒤 port 8888과 WP-CLI 실행을 검증하고, 작업 종료 시 `wp-env stop`으로 runtime을 정리한다.
* **실제 수행 여부 확인 가능성:** repository의 completion report는 Docker-backed `wp-env start`가 MySQL/phpMyAdmin image 경로에서 정지해 live mutation을 pass로 표시하지 않았고, 중단 후 WordPress·MariaDB·phpMyAdmin container나 image가 남지 않았다고 명시한다. Playground 대안도 지원되는 `wp-env run cli` 경로를 제공하지 못했다.
* **Documentation Action:** Thread 10에 provision/start/readiness/stop/failure-cleanup lifecycle과 “live closure 미완료” 상태를 보충한다.

---

## ESG-09 — WordPress draft materialization과 remote identity lifecycle

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 09 — WordPress 초안 멱등성과 식별자 보존 / WordPress Draft Idempotency and Identity Preservation
* **Related Threads:** 01, 06, 10
* **Repository Evidence:** adapter는 content marker와 idempotency key를 넣고 draft만 생성한다. receipt가 없어도 marker로 같은 draft를 찾으며, 새 승인 source에서는 이전 receipt의 `remote_post_id`를 가져와 같은 post를 갱신한다. multiple matches, non-draft, identity mismatch, changed remote ID는 fail closed한다.
* **Required External Step:** 준비된 WordPress runtime에서 실제 create/update를 수행하고, draft status·marker·remote post ID·retry 후 단일 post 유지 여부를 원격 상태에서 확인해야 한다.
* **실제 수행 여부 확인 가능성:** memory transport와 command/fetch mock 테스트는 존재하지만, completion report는 live WordPress create/update/retry sequence를 통과한 것으로 기록하지 않는다. 실제 remote post ID나 remote draft는 repository에서 확인되지 않는다.
* **Documentation Action:** Thread 09에 실제 remote mutation verification, duplicate/mismatch 대응, receipt-loss와 new-source update의 외부 검증 단계를 추가한다.

---

## ESG-10 — Hosted WordPress REST endpoint와 runtime credential/TLS injection

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 10 — WordPress 런타임과 전송 계층 격리 / WordPress Runtime and Transport Isolation
* **Related Threads:** 09, 04, 01, 11
* **Repository Evidence:** `PUBLISHER_WP_TRANSPORT=rest`일 때 site source가 지정한 environment-variable 이름에서 URL, username, password를 읽는다. loopback이 아닌 URL은 HTTPS여야 하며, REST transport는 `wp-json/wp/v2/posts`에 authenticated GET/POST를 수행한다.
* **Required External Step:** 이 선택적 transport를 사용할 때만 reachable WordPress REST endpoint를 준비하고, HTTPS 또는 loopback URL과 draft 조회·생성·갱신이 가능한 credential을 발급·주입해야 한다. credential의 정확한 형식은 코드가 고정하지 않으며 실제 값은 Git에 저장해서는 안 된다.
* **실제 수행 여부 확인 가능성:** initial product boundary는 hosted WordPress setup을 prerequisite로 두지 않는다. completion report도 외부 WordPress account, domain, provider identifier, production credential을 사용하지 않았다고 명시한다.
* **Documentation Action:** Thread 10의 conditional/deferred appendix로 기록한다. 현재 MVP의 필수 실행 단계처럼 기술해서는 안 된다.

---

## ESG-11 — Private GitHub origin provisioning과 repository identity rename

* **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
* **Primary Owner:** Project level
* **Related Threads:** 03, 05, 06, 11
* **Repository Evidence:** baseline은 새 private Git repository와 독립 root history를 요구했다. 최종 branding commit은 origin이 `content-foundry-publisher`에서 현재 이름으로 rename되었다고 기록하고 README와 package name을 갱신했다.
* **Required External Step:** GitHub 측 private repository를 생성하고 local repository의 origin을 연결·push하며, rename 시 기존 clone/remote reference를 현재 identity에 맞게 갱신해야 한다.
* **실제 수행 여부 확인 가능성:** GitHub metadata에서 현재 repository가 현재 이름의 private repository이고 기본 branch가 `main`인 것은 직접 확인된다. 다만 최초 생성 UI/API 작업, rename을 수행한 정확한 절차, 모든 developer clone의 origin 갱신 여부는 확인할 수 없다.
* **Documentation Action:** 한 번의 프로젝트-level `Repository Host Provisioning and Rename` 기록으로 보존한다.

---

# Part II — Existing Thread Supplement Packets

중복 소유를 방지하기 위해 아래 Packet은 **Primary Owner Thread에만** 작성한다. Related Thread에는 Part I의 관계만 전달하며 동일 Gap을 다시 소유시키지 않는다.

---

## Packet E01 — Thread 01

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 01
* **한국어 제목:** 승인 기반 멱등 게시 파이프라인
* **English title:** Approval-Gated Idempotent Publication Pipeline

### Gaps

* ESG-04 — Publication lock/work state와 stale residue recovery

### Repository Evidence

#### Representative commits

**`3226d9764e2887e32f82e0256ec6998cc77b4791` — `feat(publication): orchestrate idempotent receipts`**

* 관련 파일:

  * `.gitignore`
  * `src/publication.js`
  * `src/publication-receipts.js`
  * `test/publication.test.js`
* 중요성:

  * receipt identity별 directory lock을 도입한다.
  * started/failed/succeeded 기록 순서와 adapter dispatch를 결합한다.
  * lock/work directory를 Git 밖 runtime state로 만든다.
  * prior WordPress ID를 이전 receipt에서 전달한다.

**`3ea10d8fed78e341baf78a90f5082a84c1e44139` — `feat(publication): resolve adapters from runtime`**

* 관련 파일:

  * `src/runtime-adapters.js`
* 중요성:

  * Public Sites work root를 `.publisher/work/public-sites`에 생성한다.
  * environment에 따라 외부 adapter runtime을 resolve한다.

#### Relevant diff excerpts

```diff
+.publisher/locks/
+.publisher/work/
```

```js
const lockPath = path.join(root, ".publisher/locks", receiptId);
await mkdir(lockPath);
// ...
await rm(lockPath, { force: true, recursive: true });
```

정상적인 JavaScript control flow에서는 `finally` cleanup이 실행된다. 하지만 OS/process 강제 종료는 해당 cleanup을 실행하지 않을 수 있으며, 이 경우 다음 실행은 directory 존재만으로 `PUBLICATION_IN_PROGRESS`를 반환한다. 이는 code structure에서 도출되는 failure mode다.

### External Development Steps

1. 대상 article/source SHA에 대해 동일 receipt publication이 이미 실행 중인지 확인한다.
2. publication command를 단일 operator/process 경로에서 실행한다.
3. 정상 완료 또는 정상 예외 시 lock cleanup을 확인한다.
4. 비정상 종료 후에는 `.publisher/locks/<receipt-id>`와 관련 work attempt를 조사한다.
5. success transaction의 존재 여부, adapter의 외부 side effect 여부, failure event를 대조한 뒤에만 retry 여부를 결정한다.
6. stale lock의 안전한 제거 기준이 없으면 임의로 삭제하지 않고 별도 recovery 판단으로 전환한다.

### Code Connection

| External state | 연결되는 code behavior |
| --- | --- |
| Receipt-specific lock directory  | 같은 publication의 concurrent dispatch 방지 |
| `.publisher/work/public-sites/<attempt>` | release와 build report의 transient materialization |
| Existing stale lock  | `PUBLICATION_IN_PROGRESS`  |
| Existing success receipt | adapter 재호출 없이 idempotent reuse  |
| Failure event만 존재  | retry 가능하지만 외부 adapter side effect 조사 필요 |

### Evidence Boundary

**Directly observed in repository**

* lock directory의 생성·충돌 검사·정상 cleanup
* lock/work directory가 Git에서 제외됨
* existing success가 있으면 adapter를 다시 호출하지 않음

**Required/inferred from repository**

* abrupt termination 이후 lock/work residue 조사가 필요함
* 외부 adapter side effect와 local receipt의 불일치를 확인한 뒤 retry해야 함

**Actual execution not observable from Git**

* 과거 실행 중 실제 lock이 생성되었는지
* stale lock이 남았는지
* 특정 attempt directory에 어떤 transient build data가 있었는지
* 어떤 operator가 recovery를 수행했는지

### Ordering

**Conceptual execution order**

1. committed approval 확인
2. existing receipt 확인
3. lock 취득
4. started event 생성
5. adapter external step 실행
6. success pair 또는 failed event 생성
7. lock cleanup
8. 비정상 종료 시 별도 residue investigation

---

## Packet E03 — Thread 03

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 03
* **한국어 제목:** 로컬 Git 편집 워크플로
* **English title:** Local Git Editorial Workflow

### Gaps

* ESG-02 — Decap loopback editor와 local Git editorial state 활성화

### Repository Evidence

#### Representative commits

**`f49ed9cb908216f665cafd2d297f4818915e104e` — `build(deps): pin publisher toolchain`**

* `decap-cms-app@3.15.1`, `decap-server@3.10.0`과 lockfile integrity를 도입했다.
* Decap dependency의 residual advisory 때문에 trusted private checkout과 loopback binding만 허용한다.

**`0010a75442b1b0f47173cb23df9a7e05b20b6d23` — `feat(editor): configure Decap local workflow`**

* `admin/config.yml`, `admin/index.html`, `src/admin-server.js`, package scripts를 도입했다.
* local Git proxy, `editorial_workflow`, loopback-only admin server를 하나의 workflow boundary로 만들었다.

**`c2e4a582239fc170e7e9351264a4b734f53d3090` — `feat(wordpress): configure local draft target`**

* Decap에서 `wordpress-local` site record도 편집 가능하게 추가했다.
* 값 자체가 아니라 runtime environment-variable 이름만 content configuration에 남긴다.

#### Relevant diff excerpts

```yaml
local_backend: true
backend:
  name: proxy
  proxy_url: http://127.0.0.1:8081/api/v1
  branch: main
publish_mode: editorial_workflow
```

```json
"cms:proxy": "BIND_HOST=127.0.0.1 MODE=git GIT_REPO_DIRECTORY=. decap-server",
"cms:web": "node src/admin-server.js"
```

### External Development Steps

1. locked npm dependencies를 설치한다.
2. repository root에서 `cms:proxy`를 실행한다.
3. 별도 terminal에서 `cms:web`을 실행한다.
4. `127.0.0.1:8080`에서 editor를 열고 proxy `127.0.0.1:8081` 연결을 확인한다.
5. editor가 만든 local Git branch/commit을 검토한다.
6. canonical source validation과 review를 거쳐 `main`에 병합한다.
7. 두 loopback process를 명시적으로 종료한다.
8. port를 변경했다면 `PUBLISHER_CMS_PORT`의 실제 runtime 값을 별도 관리한다.

### Code Connection

* admin server는 `node_modules/decap-cms-app`의 bundle을 읽으므로 dependency materialization 없이는 실행되지 않는다.
* `editorial_workflow`의 draft/review state는 database가 아니라 Git branch/commit 상태로 표현된다.
* `backend.branch: main`은 최종 merge target이다.
* loopback binding은 단순 편의가 아니라 dependency risk boundary다.

### Evidence Boundary

**Directly observed in repository**

* 두 개의 process command
* loopback host와 default ports
* proxy backend 및 editorial workflow
* canonical article/site collection mapping

**Required/inferred from repository**

* 두 프로세스를 동시에 실행해야 editor가 정상 작동함
* 생성된 local branch/commit을 사람이 검토·병합해야 함
* port conflict 발생 시 runtime port coordination이 필요함

**Actual execution not observable from Git**

* 현재 process가 실행 중인지
* 실제 browser session
* local-only branch 또는 abandoned draft의 내용
* commit message의 smoke validation 이후 환경이 동일한지

### Ordering

**Conceptual execution order**

dependency install → proxy start → web start → local editing → Git review/merge → process shutdown

---

## Packet E05 — Thread 05

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 05
* **한국어 제목:** 불변 소스 승인 수명 주기
* **English title:** Immutable Source Approval Lifecycle

### Gaps

* ESG-03 — Human review, TTY decision, approval event commit

### Repository Evidence

#### Representative commits

**`2aaa27eef9f5db88b53242122903dbafb8fa7788` — `feat(approval): record explicit decisions`**

* reviewed Git object에서 article/site를 읽는다.
* human confirmation 전후로 clean SHA를 검사한다.
* approval/rejection을 별도 event로 생성한다.

**`f736167285b77466c7972bd0315b4168e02d818c` — `feat(approval): require interactive SHA confirmation`**

* non-TTY approval을 거부한다.
* decision과 full source SHA를 정확히 입력하도록 한다.

**`5cde980b51f0bda67e52f12056beaad069a72cdd` — `feat(publication): enforce committed approval gate`**

* source SHA의 reachability를 확인한다.
* approval event가 `HEAD`에 commit되어 있어야 한다.
* latest matching decision이 `approved`여야 한다.

#### Relevant diff excerpts

```text
Type exactly: APPROVE <40-character-source-sha>
```

```js
const listed = await runGit(root, [
  "ls-tree", "-r", "--name-only", "HEAD",
  "--", ".publisher/events/approvals",
]);
```

### External Development Steps

1. reviewer가 exact source SHA의 article과 site configuration을 검토한다.
2. worktree가 그 SHA와 일치하고 clean한지 확인한다.
3. interactive terminal에서 approval 또는 rejection command를 실행한다.
4. 표시된 article/site/SHA를 다시 확인한다.
5. full confirmation phrase를 입력한다.
6. 생성된 approval event가 secret을 포함하지 않는지 확인한다.
7. event를 stage하고 별도 Git commit으로 보존한다.
8. publication 전에 해당 commit이 현재 audited `HEAD`에서 reachable한지 확인한다.

### Code Connection

* human confirmation이 `true`가 아니면 event가 생성되지 않는다.
* event 생성 직전에 clean HEAD를 다시 검사하므로 prompt 중 source 변경을 막는다.
* uncommitted approval event는 publication gate에서 dirty state로 거부된다.
* latest rejection은 이전 approval을 무효화한다.

### Evidence Boundary

**Directly observed in repository**

* TTY requirement
* exact phrase requirement
* actor/decision/source SHA event fields
* committed event gate
* latest decision ordering

**Required/inferred from repository**

* reviewer가 실제 콘텐츠를 충분히 읽고 판단해야 함
* `actor` 값을 조직이 신뢰할 수 있는 방식으로 입력해야 함
* event를 publication 전에 commit해야 함

**Actual execution not observable from Git**

* reviewer의 실제 신원
* review의 범위·품질
* terminal 앞의 사람이 `actor`와 같은 사람인지
* approval command가 실제로 수행된 시점
* 현재 `main`에 실제 approval event가 없는 이유

### Ordering

**Conceptual execution order**

source commit → human review → TTY confirmation → approval event generation → event review → Git commit → publication gate

실제 과거 chronology로 주장할 수 있는 것은 commit에서 구현이 도입된 순서뿐이며, 특정 콘텐츠에 대한 승인 실행 chronology는 확인되지 않는다.

---

## Packet E06 — Thread 06

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 06
* **한국어 제목:** 감사 증거와 원자적 게시 기록
* **English title:** Audit Evidence and Atomic Publication Records

### Gaps

* ESG-05 — Audit/receipt materialization과 partial transaction recovery
* ESG-06 — Remote audit-history retention과 origin synchronization

### Repository Evidence

#### Representative commits

**`98f68a79a8a4c2cfd18bb159423c154b29588b5d` — `feat(audit): write append-only events`**

* event file을 exclusive-create mode로 쓴다.
* event identity 재사용과 sensitive material을 거부한다.

**`d767ac2c3ed2ba35098078af0e75d7ef3ebe3df4` — `feat(receipt): store success atomically`**

* temporary transaction directory에 event와 receipt를 쓴다.
* complete pair를 receipt ID directory로 atomic rename한다.
* identical retry는 기존 pair를 재사용하고 collision은 거부한다.

**`e3b8dd3ab591c9ec64e4804313a5ec8bce861e0d` — `fix(receipt): reject partial transactions`**

* transaction directory 자체의 부재와 구성원 누락을 구별한다.
* partial directory를 “게시하지 않음”으로 취급해 adapter를 재호출하지 않고 fail closed한다.

**`3226d9764e2887e32f82e0256ec6998cc77b4791` — `feat(publication): orchestrate idempotent receipts`**

* started/failed event와 succeeded transaction을 실제 orchestration 순서에 연결한다.
* WordPress receipt의 remote ID를 후속 source publication에 전달한다.

#### Final-state excerpts

```js
await writeFile(absolutePath, serialized, {
  flag: "wx",
  mode: 0o600,
});
```

```js
await Promise.all([
  writeFile("receipt.json", ...),
  writeFile("event.json", ...),
]);
await rename(temporary, destination);
```

현재 `.gitignore`는 locks/work를 제외하지만 `.publisher/events/`와 `.publisher/publications/`를 제외하지 않는다. 따라서 이 파일들은 transient cache가 아니라 Git으로 승격할 수 있는 audit evidence다.

### External Development Steps

#### ESG-05

1. publication command가 생성한 started/failed event 또는 success pair를 확인한다.
2. success directory에 `event.json`과 `receipt.json`이 모두 있는지 확인한다.
3. schema validation과 receipt/source/engine identity를 확인한다.
4. non-secret generated records를 stage·commit한다.
5. partial directory가 발견되면 adapter의 실제 외부 side effect를 먼저 조사한다.
6. repository에 자동 repair command가 없으므로 operator가 보존·복구·폐기 중 하나를 명시적으로 결정한다.

#### ESG-06

1. approval 및 publication evidence commit을 private origin에 push한다.
2. audited source SHA와 event SHA가 계속 reachable한지 확인한다.
3. latest decision ordering을 바꾸는 history rewrite를 하지 않는다.
4. local clean HEAD와 origin `main`의 동기화를 검증한다.
5. 조직의 retention/enforcement 방법은 repository 밖에서 정하되 실제 방법을 문서화한다.

### Code Connection

| External step | Code connection  |
| --- | --- |
| Event/receipt inspection  | schema validation과 sensitive-material rejection  |
| Git commit  | approval gate와 future receipt lookup이 committed evidence를 사용 |
| Remote push | 다른 checkout에서도 audited state를 재현하기 위한 조건 |
| Non-rewrite retention | source SHA reachability와 latest-decision semantics 유지  |
| Partial transaction investigation | incomplete directory가 adapter retry를 차단  |
| Existing WordPress receipts 보존  | `priorWordPressRemoteId`가 이전 remote ID를 찾음 |

### Evidence Boundary

**Directly observed in repository**

* append-only event write
* atomic success pair
* partial transaction fail-closed
* committed approval lookup
* non-rewrite policy 문구
* 현재 private origin과 `main`

**Required/inferred from repository**

* 생성된 records를 commit/push해야 shared audit trail이 됨
* remote history reachability를 운영적으로 유지해야 함
* partial state repair 전 external side effect 조사가 필요함

**Actual execution not observable from Git**

* 현재 tree에는 실제 approval/publication records가 없음
* 과거 real publication에서 어떤 event가 생성됐는지
* force-push 방지 또는 backup 정책
* partial transaction 복구가 실제로 수행된 적이 있는지

현재 GitHub branch metadata는 `main`을 protected branch로 표시하지 않는다. 이는 특정 보호 기능을 반드시 써야 한다는 뜻은 아니지만, repository code만으로 non-rewrite invariant가 강제된다고 주장할 수 없다는 증거다.

### Ordering

**Conceptual execution order**

approval evidence commit → publication attempt → started event → adapter result → failed event 또는 atomic success pair → local validation → Git commit → origin push → long-term reachability preservation

---

## Packet E08 — Thread 08

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 08
* **한국어 제목:** 동결 정적 렌더러 통합 경계
* **English title:** Frozen Static Renderer Integration Boundary

### Gaps

* ESG-07 — Frozen renderer checkout, repository identity migration, build-state materialization

### Repository Evidence

#### Representative commits

**`6019ff898805d0fd5e780fc42be88c0d8dc03e4c` — `feat(public-sites): add frozen build adapter`**

* checkout 검증, environment isolation, exact build command ordering, report handoff를 하나의 failure/retry boundary로 구현했다.
* work root가 renderer 내부에 있으면 거부한다.

**`3b94cad353efcd2783de91fb9b39549a709c79b5` — `test(public-sites): capture viability build`**

* 실제 article과 exact renderer SHA를 사용했다고 기록한 report를 commit했다.
* 64 artifacts와 output digest를 보존한다.
* 명시적으로 provider-free local evidence이며 public deployment가 아니다.

**`9130c1c8bfe7bfb968351d1f88bd7ef265cfc519` — `refactor(public-sites): migrate renderer identity`**

* current renderer repository name과 legacy report name을 모두 허용한다.
* frozen SHA는 변경하지 않는다.

#### Final-state configuration/source excerpts

```js
export const PUBLIC_SITES_SHA =
  "1717326cda8262d7f7f56d544b3a9d0215b71d51";
```

```js
const repository = environment.PUBLISHER_PUBLIC_SITES_REPO;
if (!repository) {
  throw new RuntimeAdapterError("PUBLIC_SITES_REPOSITORY_REQUIRED", ...);
}
```

```text
fnm exec --using=24.19.0 -- corepack pnpm ...
```

### External Development Steps

1. 별도 renderer repository를 가져온다.
2. exact SHA `1717326…`로 detach한다.
3. tracked working tree가 clean한지 확인한다.
4. Node `24.19.0`, Corepack, pnpm `11.22.0` 및 renderer locked dependencies를 준비한다.
5. 기존 checkout이 old repository name을 사용한다면 remote identity를 점검한다. Adapter 자체는 local path와 SHA를 검증하며 remote URL을 검증하지 않는다.
6. `PUBLISHER_PUBLIC_SITES_REPO`에 absolute 또는 resolvable checkout path를 주입한다.
7. control-plane의 `.publisher/work/public-sites/<attempt>`에 release/report를 생성한다.
8. renderer에서 prerequisite와 target build를 실행한다.
9. `apps/site-a/out`과 build report를 검증한다.
10. transient attempt/build output의 보존 또는 정리 정책을 적용한다.

### Code Connection

* wrong SHA 또는 tracked diff → frozen checkout rejection
* work root가 renderer 내부 → `WORK_ROOT_INSIDE_RENDERER`
* missing env path → `PUBLIC_SITES_REPOSITORY_REQUIRED`
* failed command → redacted `PUBLIC_SITES_BUILD_FAILED`
* verified output → receipt에는 full output이 아니라 build-report SHA-256만 저장
* legacy viability report → old repository name을 유지하면서 schema validation 가능

### Evidence Boundary

**Directly observed in repository**

* exact SHA pin
* expected target와 build command
* environment sanitization
* output digest/report schema
* committed viability report
* repository-name migration compatibility

**Required/inferred from repository**

* 외부 renderer checkout materialization
* external repository dependency installation
* runtime path injection
* transient output cleanup 또는 retention 결정

**Actual execution not observable from Git**

* 현재 renderer checkout의 위치와 상태
* 현재 installed renderer dependency tree
* 현재 `apps/site-a/out`의 존재
* current attempt directories
* report 생성 당시 external filesystem 전체
* public host로의 배포

### Ordering

**Conceptual execution order**

external clone → identity/SHA selection → clean checkout check → locked install → env path injection → release compilation → target build → report verification → receipt creation

`3b94cad`의 committed report는 이 순서의 한 차례 수행을 기록하지만, 현재 환경의 지속 상태를 의미하지 않는다.

---

## Packet E09 — Thread 09

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 09
* **한국어 제목:** WordPress 초안 멱등성과 식별자 보존
* **English title:** WordPress Draft Idempotency and Identity Preservation

### Gaps

* ESG-09 — WordPress draft materialization과 remote identity lifecycle

### Repository Evidence

#### Representative commits

**`c2e4a582239fc170e7e9351264a4b734f53d3090` — `feat(wordpress): configure local draft target`**

* `wordpress-local` site와 draft-only policy를 도입했다.
* runtime URL과 credential 값은 source가 아니라 environment-variable 이름으로 남긴다.

**`fb55fba7071d676afe095db2af0ad253890dbafd` — `feat(wordpress): upsert idempotent drafts`**

* marker를 포함한 HTML을 만들고 create 또는 update를 수행한다.
* receipt loss 뒤 marker recovery, prior remote ID update, duplicate/mismatch rejection을 구현한다.

**`3226d9764e2887e32f82e0256ec6998cc77b4791` — `feat(publication): orchestrate idempotent receipts`**

* successful receipt의 `remote_post_id`를 보존한다.
* 새 source publication에서 이전 WordPress receipt의 remote ID를 찾는다.

#### Relevant source excerpt

```js
const remote = existing
  ? await transport.updateDraft(existing.id, payload)
  : await transport.createDraft(payload);
```

```js
return {
  kind: "wordpress",
  remote_post_id: remote.id,
  remote_status: "draft",
};
```

### External Development Steps

1. Thread 10에서 정의된 `wp-env` 또는 REST runtime을 준비한다.
2. approved source로 publication을 실행한다.
3. WordPress에 실제 draft가 하나 생성되었는지 확인한다.
4. draft content에 article identity와 idempotency marker가 있는지 확인한다.
5. receipt의 `remote_post_id`와 실제 WordPress post ID가 같은지 확인한다.
6. 같은 source를 retry하여 post 수와 ID가 변하지 않는지 확인한다.
7. 새 approved source를 게시하여 같은 remote ID가 update되는지 확인한다.
8. receipt loss 상황에서는 slug/marker recovery가 duplicate 없이 같은 post를 찾는지 확인한다.
9. multiple matching drafts, non-draft status, mismatched article이 발견되면 자동 수정하지 않고 fail-closed 상태를 조사한다.

### Code Connection

| Remote condition  | Adapter behavior  |
| --- | --- |
| Prior remote ID 있음  | 해당 post를 조회하고 update  |
| Prior receipt 없음  | slug와 exact marker로 기존 draft 검색 |
| Matching draft 없음 | 새 draft 생성  |
| Matching draft 둘 이상 | `DUPLICATE_REMOTE_DRAFTS` |
| Remote status가 draft 아님 | `REMOTE_NOT_DRAFT`  |
| 다른 article marker | `REMOTE_IDENTITY_MISMATCH`  |
| Update 후 ID 변경  | `REMOTE_ID_CHANGED` |

### Evidence Boundary

**Directly observed in repository**

* marker format
* create/update selection
* draft-only assertion
* receipt remote ID preservation
* memory/mocked transport tests

**Required/inferred from repository**

* 실제 WordPress에서 remote status와 ID를 검증해야 함
* local receipt와 remote post의 identity를 함께 보존해야 함
* ambiguous remote state에서는 operator investigation이 필요함

**Actual execution not observable from Git**

* 실제 WordPress post
* 실제 remote ID
* create/update API 또는 WP-CLI의 성공
* receipt-loss recovery의 live 실행
* duplicate remote draft의 실제 발생 여부

completion report는 이 live sequence를 통과한 것으로 표시하지 않는다. 따라서 tests를 실제 external mutation evidence로 승격해서는 안 된다.

### Ordering

**Conceptual execution order**

runtime ready → first create → remote verification → receipt commit → same-source retry → new-source update → mismatch/duplicate recovery

---

## Packet E10 — Thread 10

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 10
* **한국어 제목:** WordPress 런타임과 전송 계층 격리
* **English title:** WordPress Runtime and Transport Isolation

### Gaps

* ESG-08 — Docker-backed `wp-env` runtime provisioning과 teardown
* ESG-10 — Hosted WordPress REST endpoint와 runtime credential/TLS injection

### Repository Evidence

#### Representative commits

**`c2e4a582239fc170e7e9351264a4b734f53d3090` — `feat(wordpress): configure local draft target`**

* WordPress 7.1, PHP 8.3, port 8888의 local runtime contract를 도입했다.
* `wp:start`와 `wp:stop` command를 추가했다.

**`d2067c072b7d45ca2188c8a09d2e2b62039720a6` — `feat(wordpress): isolate REST runtime credentials`**

* URL·username·password를 named environment entries에서만 읽는다.
* non-loopback HTTPS를 요구하고 response body를 error에 노출하지 않는다.

**`3da96dfaaa4bc1f25eb9baed0b904f66d74352f7` — `feat(wordpress): add credential-free wp-env transport`**

* local `node_modules/.bin/wp-env run cli wp`를 통해 container 내부 WP-CLI를 사용한다.
* inherited environment를 allowlist하며 command failure를 redact한다.

**`b44d2c67c39d969e74b26688f14831c184096a2b` — `fix(wordpress): start one wp-env runtime`**

* deprecated tests environment를 끄고 development instance 하나만 시작하도록 한다.

**`3ea10d8fed78e341baf78a90f5082a84c1e44139` — `feat(publication): resolve adapters from runtime`**

* 기본 transport는 `wp-env`다.
* `rest`는 명시적 mode와 runtime mapping이 있을 때만 선택된다.

#### Final-state configuration excerpts

```json
{
  "core": "https://downloads.wordpress.org/release/wordpress-7.1.zip",
  "phpVersion": "8.3",
  "port": 8888,
  "testsEnvironment": false
}
```

```yaml
wordpress:
  base_url_env: PUBLISHER_WP_BASE_URL
  username_env: PUBLISHER_WP_USERNAME
  password_env: PUBLISHER_WP_PASSWORD
  default_status: draft
```

### External Development Steps

#### Mode A — default `wp-env`

1. Docker daemon/Desktop의 readiness를 확인한다.
2. project dependencies를 설치한다.
3. `wp-env start`를 실행한다.
4. development WordPress runtime 하나와 그 database dependency가 준비되었는지 확인한다.
5. loopback port 8888의 readiness를 확인한다.
6. supported `wp-env run cli wp`가 작동하는지 확인한다.
7. publication을 수행한다.
8. 종료 후 `wp-env stop`을 실행한다.
9. `.wp-env` runtime data는 disposable external state로 유지하고 Git에 넣지 않는다.
10. image pull 또는 startup 실패 후 남은 container/image/runtime state를 조사한다.

#### Mode B — conditional `rest`

1. `PUBLISHER_WP_TRANSPORT=rest`를 명시한다.
2. WordPress REST posts endpoint를 준비한다.
3. non-loopback endpoint라면 HTTPS를 제공한다.
4. draft 조회·생성·갱신이 가능한 user credential을 준비한다.
5. URL, username, password를 site mapping이 지시하는 environment names에 주입한다.
6. secret이 `.env`, log, receipt, source에 들어가지 않는지 확인한다.
7. authenticated GET/POST와 error redaction을 검증한다.
8. credential rotation·revocation은 repository 밖 runtime responsibility로 유지한다.

### Code Connection

| Runtime state | Code behavior  |
| --- | --- |
| Docker/`wp-env` 미시작 | WP-CLI command failure |
| 잘못된 transport mode  | `WORDPRESS_TRANSPORT_INVALID`  |
| REST env 값 누락 | `WORDPRESS_RUNTIME_MISSING`  |
| non-loopback HTTP | `WORDPRESS_URL_INSECURE` |
| REST 4xx/5xx  | status만 포함한 `WORDPRESS_REQUEST_FAILED` |
| malformed response  | `WORDPRESS_RESPONSE_INVALID` |
| valid transport | Thread 09의 draft upsert 실행 |

### Evidence Boundary

**Directly observed in repository**

* 두 transport 구현
* default `wp-env` selection
* exact local runtime configuration
* environment-variable names
* HTTPS requirement
* environment sanitization과 error redaction

**Required/inferred from repository**

* Docker-backed runtime의 실제 provisioning
* REST endpoint 및 credential 발급
* endpoint가 필요한 read/write 권한을 제공해야 함
* credential rotation/revocation과 Docker cleanup

**Actual execution not observable from Git**

* Docker daemon의 현재 상태
* 실제 image/container/database
* 실제 WordPress REST account
* 실제 credential 값
* TLS certificate와 endpoint 운영 상태
* 성공한 live create/update sequence

completion report는 Docker-backed proof가 완료되지 않았으며 외부 account나 production credential을 사용하지 않았다고 명시한다. 따라서 Mode A는 **미완료 external closure**, Mode B는 **conditional/deferred external closure**다.

### Ordering

#### Default local mode

Docker readiness → `wp-env start` → WordPress/WP-CLI readiness → publication → remote verification → `wp-env stop`

#### Hosted REST mode

external WordPress provisioning → HTTPS/read-write credential → environment injection → connection verification → publication → remote verification → credential lifecycle management

두 mode를 하나의 실제 chronology로 합쳐서는 안 된다. REST mode는 local proof 이후의 선택적 경로다.

---

# Part III — Proposed New Thread Packets

## 제안 없음

이번 감사에서는 `NEW_THREAD` 조건을 만족하는 관점이 발견되지 않았다.

판정 근거는 다음과 같다.

* Decap process와 local Git state는 Thread 03의 실행 완성 단계다.
* human approval과 approval-event commit은 Thread 05의 핵심 lifecycle이다.
* lock/work state는 Thread 01, audit/receipt persistence와 history retention은 Thread 06에 이미 자연스럽게 귀속된다.
* frozen renderer checkout과 build output은 Thread 08의 integration boundary를 실제 환경에서 성립시키는 단계다.
* WordPress runtime/credential은 Thread 10, 실제 remote draft identity는 Thread 09가 각각 명확한 Primary Owner다.
* 공통 dependency bootstrap과 GitHub repository provisioning은 여러 Thread를 지원하지만 독립적인 개발 문제로 확장될 만큼의 구현·실패·복구 commit series를 갖지 않으므로 프로젝트 수준 단계가 적절하다.

따라서 외부 상태가 존재한다는 이유만으로 별도 “Operations” 또는 “Infrastructure” Thread를 추가하지 않는다.

---

# Part IV — Project-Level External Steps

## ESG-01 — Publisher dependency tree materialization

### Repository Evidence

* `f49ed9c`가 exact package versions, npm version, Node range와 lockfile integrity를 도입했다.
* `node_modules/`는 Git에 저장되지 않는다.
* Decap admin server, `wp-env` transport, CLI와 test commands는 materialized local binaries와 modules를 직접 사용한다.

### Required External Steps

**Conceptual execution order**

1. Git과 호환 Node version을 준비한다.
2. package manager version boundary를 확인한다.
3. repository root에서 locked install을 수행한다.
4. lockfile과 installed dependency tree의 일치를 확인한다.
5. `npm run check` 등 repository gate를 수행한다.
6. `node_modules`는 local materialized state로 유지하고 commit하지 않는다.
7. dependency upgrade 때 Decap의 loopback-only residual-risk boundary를 다시 검토한다.

### Evidence Boundary

* **Directly observed:** package manifest, lockfile, engine range, ignored `node_modules`.
* **Required/inferred:** install이 없으면 editor, `wp-env`, schema validation이 실행될 수 없음.
* **Actual execution not observable:** 현재 machine의 package contents, registry/cache 상태, 과거 install 시점.

### Documentation Action

프로젝트 root에 한 개의 bootstrap checklist를 두고 Thread 03·08·10에서 참조한다. Thread마다 같은 `npm ci` 설명을 복제하지 않는다.

---

## ESG-11 — Private GitHub origin provisioning과 repository identity rename

### Repository Evidence

* root baseline은 새 private repository와 독립 root history를 전제로 했다.
* final branding commit은 이전 origin name에서 현재 name으로 rename되었다고 기록하고 package, README, completion report를 갱신했다.
* 현재 repository-host metadata는 현재 이름, private visibility, default branch `main`을 직접 보여준다.

### Required External Steps

**Conceptual execution order**

1. private GitHub repository를 생성한다.
2. local root history에 origin을 연결한다.
3. main history를 push한다.
4. repository family rename을 GitHub 측에서 수행한다.
5. local clones, scripts, documentation의 remote identity를 현재 이름으로 갱신한다.
6. current `main`과 local clean HEAD를 비교한다.
7. audited history가 계속 reachable하도록 origin을 유지한다.

### Evidence Boundary

* **Directly observed in source repository:** old/new identity를 설명하는 commit과 documentation.
* **Directly observed in repository host:** 현재 repository name, private visibility, default branch.
* **Required/inferred:** developer clone의 origin 갱신과 push synchronization.
* **Actual execution not observable:** rename UI/API 절차, 각 clone에서 실행한 `remote set-url`, 모든 과거 operator action.

### Documentation Action

이 단계는 특정 Development Thread가 아니라 `Project Repository Host State` 항목으로 한 번만 기록한다.

---

# 채택하지 않은 외부 단계

다음 항목은 일반적으로 가능하다는 이유만으로 Gap에 포함하지 않았다.

* **Database migration 또는 seed:** migration, schema, seed script, `DATABASE_URL` 사용 근거가 없다.
* **CI/CD secret 또는 environment:** current tree에 GitHub Actions나 다른 CI configuration이 없다.
* **OAuth application 또는 redirect URI:** OAuth integration이 없다.
* **Webhook registration:** webhook endpoint나 handler가 없다.
* **Object storage, bucket, IAM:** 해당 integration이 없다.
* **Cloudflare, DNS, domain, analytics, public deployment:** 현재 phase에서 명시적으로 deferred다.
* **Static renderer output의 public hosting:** committed viability evidence가 스스로 local provider-free build이며 public deployment가 아니라고 한정한다.
* **Hosted WordPress account creation:** default local MVP의 prerequisite가 아니며, REST mode를 선택할 때만 ESG-10의 conditional step이 된다.
* **TLS provisioning 일반론:** remote REST mode의 HTTPS requirement에 필요한 범위만 ESG-10에 포함했고, 특정 certificate provider나 DNS 절차는 repository가 정하지 않으므로 추가하지 않았다.
* **Backup/restore:** backup artifact, restore script, retention policy를 요구하는 구체적 repository evidence가 없다.

따라서 이 감사의 최소 보완 범위는 **기존 Thread 7개에 대한 external-state supplement + 프로젝트 수준 2개 운영 항목**이며, 기존 11개 Thread 구조를 변경할 필요는 없다.
