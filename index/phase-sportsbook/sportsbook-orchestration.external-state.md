# External-State Development Gap Audit

## 감사 범위와 판정 요약

첨부된 Thread Index는 **이미 확정된 소유권 체계**로만 사용했습니다. Thread 제목·번호·commit 배치를 유지했으며, 기존 학습 문서나 Thread별 해설서는 분석 입력으로 사용하지 않았습니다. 

분석 범위는 다음과 같습니다.

* GitHub commit history를 마지막 빈 pagination까지 순회
* 대표 commit의 실제 diff 확인
* 현재 recursive tree 전체 확인
* Compose, migration, Kafka bootstrap, fixture, E2E, secret, observability, cold-gate, evidence, CI, history-policy 파일 확인
* 현재 공개 branch/ref 상태와 workflow trigger 대조

현재 recursive tree 응답은 `truncated: false`였으므로 current-state repository contents는 잘리지 않은 상태로 검토했습니다.

### 최종 판정

* **EXISTING_THREAD Gap:** 15개
* **NEW_THREAD:** 0개
* **PROJECT_LEVEL_EXTERNAL_STEP:** 1개
* **보충 Packet 대상 기존 Thread:** 1–9, 14–17
* **별도 Packet이 없는 Thread 10–13:** 이들의 live-state 실행은 Thread 9가 Primary Owner인 cross-store E2E Gap에 Related Threads로 포함했습니다. 같은 실행을 네 Thread에 중복 소유시키지 않았습니다.

### 현재 상태에서 직접 확인되는 두 가지 중요 사항

1. `services.lock`은 8개 service branch와 SHA를 요구하지만, 현재 공개 원격에서 확인되는 branch는 `main` 하나뿐입니다. Materializer는 각 `refs/heads/<branch>` 또는 `refs/remotes/origin/<branch>`가 정확히 lock SHA를 가리켜야 진행합니다. 따라서 **현재 공개 원격의 fresh clone만으로는 source materialization 조건을 충족할 수 없습니다.** 과거 개발자 로컬이나 별도 archive clone에 해당 refs가 있었는지는 확인할 수 없습니다.

2. 현재 workflow에는 `workflow_dispatch`가 정의되어 있지만 자동 push trigger는 `orchestration` branch를 대상으로 합니다. 현재 공개 branch는 `main`뿐이고 `protected: false`입니다. 따라서 **현재 `main` push에 대한 자동 history/cold-gate 집행은 repository 상태로 확인되지 않습니다.** 수동 실행이나 다른 외부 정책이 실제로 사용됐는지는 Git으로 확인할 수 없습니다.

이 두 항목과 달리, 나머지 Gap 대부분은 "현재 잘못 구성되었다"는 판정이 아닙니다. Repository에서 그 실행 필요성은 확인되지만, 실제 Docker/Kafka/PostgreSQL/Redis/secret/evidence 상태가 만들어졌는지는 Git으로 관찰할 수 없다는 판정입니다.

---

# Part I — Gap Index

| Gap ID | 짧은 이름 | Classification / Primary Owner | Related Threads | Repository Evidence → Required External Step | 실제 수행 여부 확인 가능성 | Documentation Action |
| --- | --- | --- | --- | --- | --- | --- |
| **G-01** | 잠긴 Git ref와 detached source materialization   | **EXISTING_THREAD** — Thread 1  | 2, 14, 16 | `services.lock`의 8개 branch/SHA와 materializer의 exact-ref 검증 → 해당 commit object와 branch ref를 실행 clone에 제공하고 8개 detached worktree를 생성 | **현재 공개 원격 미충족은 확인 가능.** 과거·비공개·로컬 ref 제공 및 실제 materialization은 확인 불가 | Thread 1에 Source Availability/Execution Boundary 보충 |
| **G-02** | Run-owned Maven repository와 원자적 service JAR generation | **EXISTING_THREAD** — Thread 2  | 1, 3, 14, 16 | 격리 Maven repository, 정확히 7개 service JAR, generation directory와 `docker/jars` symlink → 실제 빌드·검증·원자적 게시 수행  | 실제 JAR generation, hash, Maven cache는 Git에서 확인 불가   | Thread 2에 Build-State Lifecycle 보충   |
| **G-03** | Compose project와 image/container/network 기동   | **EXISTING_THREAD** — Thread 3  | 2, 4, 5, 6, 7, 14  | `docker compose up --detach --build --wait`와 기동 DAG → image build/pull, private network, containers, bounded gates, named volumes 생성  | 실제 daemon 상태와 기동 성공 여부 확인 불가   | Thread 3에 Runtime Provisioning 보충 |
| **G-04** | PostgreSQL database bootstrap와 Flyway 실행   | **EXISTING_THREAD** — Thread 4  | 3, 9, 10, 11, 16   | init SQL이 4개 DB를 만들고 runtime evidence가 25개 migration을 요구 → 실제 DB 생성 및 service Flyway 실행  | migration 파일·검증 코드는 관찰됨. 실제 적용 시점·DB 상태는 확인 불가   | Thread 4에 DB Bootstrap/Migration Execution 보충 |
| **G-05** | Redis별 격리 persistent state  | **EXISTING_THREAD** — Thread 4  | 3, 9, 10, 16 | `redis-risk`, `redis-odds`, `redis-wallet`, `redis-gateway`의 별도 volume/AOF contract → 실제 volume 생성과 데이터 격리 | 실제 AOF 파일·key state·복구 이력 확인 불가   | G-04와 같은 Thread 4 Packet에 포함   |
| **G-06** | Kafka KRaft state, topic 생성, consumer assignment | **EXISTING_THREAD** — Thread 5  | 3, 8, 9, 12, 16 | 23개 topic manifest, drift rejection, 두 group의 정확한 partition assignment → broker metadata와 topic/group state 실제 구성   | topic·offset·assignment의 실제 실행 상태 확인 불가 | Thread 5에 Broker-State Execution 보충  |
| **G-07** | Runtime credential/keypair와 제한 노출 | **EXISTING_THREAD** — Thread 6  | 3, 9, 13, 14, 15   | 매 run 11개 service key, DB/Grafana password, RSA keypair, loopback port 생성 → 파일·환경변수 주입 및 제한 port binding   | 생성 로직만 관찰. 실제 값은 확인 불가이며 복원·추정 대상이 아님   | Thread 6에 Secret Injection/Exposure Lifecycle 보충 |
| **G-08** | Observability data plane와 Docker discovery 권한 | **EXISTING_THREAD** — Thread 7  | 3, 14, 15, 16   | Prometheus/Loki/Grafana volumes와 Promtail Docker socket discovery → 실제 metrics/log stores 생성 및 socket 접근 허용   | 실제 metric/log/dashboard contents와 socket permission 확인 불가 | Thread 7에 Observability Runtime-State 보충   |
| **G-09** | Fixture JAR staging과 live Kafka publish/probe | **EXISTING_THREAD** — Thread 8  | 2, 5, 9, 12  | exact shaded fixture JAR, broker-ack receipt, exact partition/offset probe → 실제 Kafka records 게시·조회  | 실제 record, offset, ephemeral probe group 확인 불가   | Thread 8에 Fixture Execution Lifecycle 보충   |
| **G-10** | 13개 cross-store E2E scenario 실행   | **EXISTING_THREAD** — Thread 9  | 4, 5, 8, 10, 11, 12, 13, 16 | 고정 순서의 13개 scenario와 DB/Redis/Kafka/HTTP oracle → clean stack에 seed하고 cross-store state transition 검증   | 실제 pass/fail, 생성 row/key/message 확인 불가  | Thread 9에 Live E2E Execution 보충   |
| **G-11** | Toxiproxy fault mutation과 restoration   | **EXISTING_THREAD** — Thread 9  | 10, 14 | allowlisted proxy disable/enable, timeout toxic create/delete → runtime proxy state 변경 후 반드시 복원   | 복구 코드 존재. 특정 실행에서 복구 성공 여부 확인 불가  | G-10과 같은 Thread 9 Packet에 포함   |
| **G-12** | Cold-gate lock, ownership, cleanup lifecycle  | **EXISTING_THREAD** — Thread 14 | 1, 2, 3, 6, 7, 15, 16 | 고유 Compose project, worktree lock, owner markers, scoped teardown → run 소유 상태를 만들고 성공·실패 시 정확히 제거 | 실제 run lock, residual container/volume 여부 확인 불가  | Thread 14에 External Resource Lifecycle 보충  |
| **G-13** | Evidence persistence, redaction, CI artifact retention | **EXISTING_THREAD** — Thread 15 | 14, 16, 17   | write-once evidence directory, secret scan, Git ignore, Actions artifact upload 14일 → local evidence를 남기고 필요 시 외부 artifact store에 업로드 | Git에는 evidence가 의도적으로 없음. 실제 artifact upload/retention 확인 불가 | Thread 15에 Evidence Retention Boundary 보충  |
| **G-14** | Live semantic attestation capture | **EXISTING_THREAD** — Thread 16 | 4, 5, 7, 8, 9, 10, 11, 12, 13, 15 | release identity, compose digest, topic, migration, readiness, scenario, log, cleanup evidence → live state에서 attestation 생성 | schema와 생성 로직만 관찰. 실제 attestation 값 확인 불가  | Thread 16에 Attestation Execution 보충  |
| **G-15** | History policy 실행과 Git/CI control binding  | **EXISTING_THREAD** — Thread 17 | 14, 15, 16   | full-history verifier와 archive workflow → full clone에서 guard를 실행하고 실제 release control point에 연결   | 현재 branch/trigger 불일치는 관찰됨. 수동 실행·외부 enforcement 여부는 확인 불가   | Thread 17에 Repository-Host Enforcement Boundary 보충  |

---

# Part II — Existing Thread Supplement Packets

## Packet E-01 — Thread 1

### Thread Identity

* **Existing Thread**
* **Thread 1**
* **한국어 제목:** 릴리스 입력 잠금과 소스 구체화
* **English title:** Release Input Locking and Source Materialization

### Gaps

* **G-01 — 잠긴 Git ref와 detached source materialization**

### Repository Evidence

**Representative commits**

* `ca6c5af4f18b` — `fix(source): resolve archive remote refs`

  * local branch 또는 `origin` remote-tracking branch를 명시적으로 찾도록 materialization 경계를 강화했습니다.
  * branch ref와 locked commit의 일치를 확인한 뒤 정확한 commit으로 detached worktree를 생성하는 변경입니다.
* `e1e3d4119704` — `test(source): materialize remote-only archive refs`

  * remote-only ref에서도 materialization 계약이 성립하는지 검증합니다.

**관련 파일과 final-state 내용**

* `services.lock`

  * 8개 logical source에 대해 branch, full SHA, artifact를 고정합니다.
* `scripts/materialize-sources.sh`

  * lock entry가 정확히 8개인지 검사
  * commit object가 존재하는지 검사
  * `refs/heads/<branch>` 또는 `refs/remotes/origin/<branch>`가 존재하는지 검사
  * branch ref가 lock SHA와 정확히 같은지 검사
  * 각 source를 detached worktree로 생성
  * cleanup 시 HEAD, detached 상태, clean 상태를 다시 확인한 뒤 제거
* 현재 공개 branch 목록은 `main` 하나이며 `protected: false`입니다. 따라서 lock에 적힌 8개 branch ref는 현재 공개 원격에서 확인되지 않습니다.

**Diff focus**

```text
+ local 또는 origin remote-tracking ref를 명시적으로 resolve
+ branch ref가 lock SHA와 정확히 일치하지 않으면 중단
+ exact SHA로 detached worktree 생성
+ cleanup 전 HEAD·detached·clean 상태 재검증
```

### External Development Steps

1. 실행 clone에 8개 locked commit object를 제공합니다.
2. 각 object를 `services.lock`의 branch 이름과 일치하는 local 또는 `origin` ref로 제공합니다.
3. `materialize-sources.sh`를 실행해 `.runtime/.../sources/<logical-name>` detached worktree를 생성합니다.
4. 해당 worktree를 Maven/JAR build의 유일한 source input으로 사용합니다.
5. build/run 종료 후 각 worktree의 ownership과 clean 상태를 확인하고 `git worktree remove`를 수행합니다.

### Code Connection

* Locked source가 없으면 `ReleaseBuilder.build()`의 첫 command인 materialization이 실패하며 이후 shared install, service JAR build, fixture build는 시작되지 않습니다.
* Thread 2는 생성된 source의 빌드를 소유하고, Thread 14는 run 종료 시 worktree cleanup을 소유합니다.

### Evidence Boundary

* **Directly observed in repository**

  * 8개 branch/SHA lock
  * exact ref와 exact SHA를 요구하는 materializer
  * 현재 공개 원격에 `main`만 존재한다는 상태
* **Required/inferred from repository**

  * cold gate 실행 전에 locked refs/objects가 실행 clone에 제공되어야 함
  * fresh clone에서는 별도 ref provisioning 또는 archive source 제공이 필요함
* **Actual execution not observable from Git**

  * 과거 개발자 clone 또는 CI runner에 해당 branch/ref가 있었는지
  * 누가 언제 refs를 만들거나 fetch했는지
  * 실제로 8개 worktree가 성공적으로 생성됐는지

### Ordering — conceptual execution order

`archive refs/objects 제공 → services.lock 검증 → 8개 detached worktree 생성 → Thread 2 build → Thread 14 cleanup`

---

## Packet E-02 — Thread 2

### Thread Identity

* **Existing Thread**
* **Thread 2**
* **한국어 제목:** 격리 빌드와 원자적 아티팩트 게시
* **English title:** Hermetic Build and Atomic Artifact Publication

### Gaps

* **G-02 — Run-owned Maven repository와 원자적 service JAR generation**

### Repository Evidence

**Representative commit**

* `f4a48d911ada` — `build(jars): stage exact release artifacts atomically`

  * 임시 generation을 만들고 정확히 7개 service JAR만 허용한 뒤 generation을 게시하고 `docker/jars` symlink를 원자적으로 연결하는 변경입니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/build.py`

  * host의 reserved Maven/Java/build 환경변수 override를 거부
  * run-owned `.runtime/.../m2/repository` 생성
  * locked source materialization
  * shared protocol install
  * service JAR staging
  * fixture publisher staging
  * `docker/jars`가 symlink인지 검사
  * service generation에 정확히 7개 JAR만 있는지 검사
* `.gitignore`는 `.runtime/`, `docker/.jars/`, `docker/jars`를 제외합니다. 실제 build outputs가 Git에 남지 않는 구조입니다.

**Diff focus**

```text
+ run-owned Maven repository 사용
+ pending JAR generation에서 exact artifact inventory 검증
+ 완성된 generation만 최종 경로로 이동
+ docker/jars symlink를 generation에 원자적으로 연결
```

### External Development Steps

1. JDK 17과 executable Maven wrapper를 사용해 shared protocol을 run-owned repository에 install합니다.
2. 7개 application source를 같은 repository를 사용해 package합니다.
3. 임시 generation에서 artifact 이름·개수·JAR identity를 검증합니다.
4. 완성된 generation을 `docker/.jars/<generation>`에 게시합니다.
5. `docker/jars` symlink를 해당 generation으로 연결합니다.
6. Compose image build가 그 generation을 읽도록 합니다.
7. run 종료 시 symlink, generation, Maven repository를 제거합니다.

Fixture publisher의 artifact·Kafka 실행 semantics는 중복을 피하기 위해 **Thread 8 / G-09**가 소유합니다.

### Code Connection

* Compose application image는 현재 source tree가 아니라 `docker/jars`가 가리키는 exact generation을 입력으로 사용합니다.
* `ColdStack.start()`의 `docker compose up --build`는 이 외부 artifact state가 이미 완성되어 있다는 전제에서 image build를 시작합니다.

### Evidence Boundary

* **Directly observed in repository**

  * build command sequence
  * run-owned Maven path
  * 정확히 7개 JAR requirement
  * atomic generation/symlink contract
* **Required/inferred from repository**

  * 실제 Maven dependency resolution
  * JAR compilation·packaging
  * generation과 symlink 생성
* **Actual execution not observable from Git**

  * 실제 생성된 JAR bytes와 hash
  * Maven cache contents
  * 어느 run에서 generation이 게시됐는지
  * 실패 후 pending artifact가 완전히 제거됐는지

### Ordering — conceptual execution order

`locked sources → isolated Maven repository → shared install → 7개 service build → generation 검증 → atomic symlink publication → image build → scoped removal`

---

## Packet E-03 — Thread 3

### Thread Identity

* **Existing Thread**
* **Thread 3**
* **한국어 제목:** 풀스택 토폴로지와 기동 DAG
* **English title:** Full-Stack Topology and Startup DAG

### Gaps

* **G-03 — Compose project와 image/container/network 기동**

### Repository Evidence

**Representative commit**

* `4b3c66663326` — `build(startup): enforce full dependency DAG`

  * persistence, preflight, application, consumer-assignment, settlement, admin 순서가 Compose dependency graph로 명시되도록 강화했습니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/stack.py`

  * 기존 project resources가 없어야 시작
  * `docker compose config --quiet`
  * `docker compose up --detach --build --wait --wait-timeout 900`
  * Gateway, Toxiproxy, Grafana의 실제 loopback publication을 조회·검증
* README가 정의하는 final topology는 18개 long-running container와 3개 bounded gate, 총 21개 service입니다. 기동 순서는 persistence/cache → secret/topic gate → Wallet/Risk/Odds → Betting → Gateway → consumer assignment → Settlement → Admin입니다.

**Diff focus**

```text
+ 각 application에 explicit depends_on/health condition 추가
+ bounded gate 완료를 후속 service의 시작 조건으로 사용
+ 전체 project를 --build --wait로 한 번에 materialize
```

### External Development Steps

1. Docker daemon이 필요한 base image를 pull하거나 기존 cache에서 resolve합니다.
2. 7개 application image를 staged JAR로 build합니다.
3. 고유 Compose project의 private network를 생성합니다.
4. PostgreSQL, Kafka, Redis, observability용 named volumes를 생성합니다.
5. container와 bounded gate를 dependency order에 따라 생성·기동합니다.
6. healthcheck와 one-shot gate completion을 기다립니다.
7. 세 개의 허용된 host exposure만 loopback에 binding되었는지 확인합니다.

Volume의 domain contents는 다음 Primary Owner가 소유합니다.

* PostgreSQL/Redis contents: Thread 4
* Kafka broker/topic contents: Thread 5
* Observability contents: Thread 7
* 전체 teardown: Thread 14

### Code Connection

* `ReleaseChecks.run()`은 Compose config를 기록한 뒤 stack을 기동하고, 그 이후에만 topic, migration, E2E, final-state 검증을 수행합니다.

### Evidence Boundary

* **Directly observed in repository**

  * Compose topology와 DAG
  * `up --build --wait`
  * project-scoped port verification
* **Required/inferred from repository**

  * Docker image, network, container, volume의 실제 생성
  * healthcheck 통과와 one-shot gate 완료
* **Actual execution not observable from Git**

  * image pull/build 성공
  * container ID와 runtime health
  * 실제 network/volume ID
  * host port allocation 결과

### Ordering — conceptual execution order

`Compose config 검증 → image pull/build → project network/volumes 생성 → DAG 기동 → health/gate 대기 → runtime port 검증`

---

## Packet E-04 — Thread 4

### Thread Identity

* **Existing Thread**
* **Thread 4**
* **한국어 제목:** 저장소 격리와 마이그레이션 무결성
* **English title:** Storage Isolation and Migration Integrity

### Gaps

* **G-04 — PostgreSQL database bootstrap와 Flyway 실행**
* **G-05 — Redis별 격리 persistent state**

### Repository Evidence

**Representative commits**

* `f57b610f2637` — `build(postgres): bootstrap service databases`

  * PostgreSQL named volume과 bootstrap SQL을 통해 service별 database를 만들도록 도입했습니다.
* `ceb9d6f668f8` — `test(redis): verify risk storage contract`

  * Redis의 AOF, `everysec`, `noeviction`, dedicated volume contract를 실제 Compose 실행으로 검증합니다.
* `6572947a55d5` — `build(evidence): verify Flyway release history`

  * live Flyway history를 source migration과 비교하는 evidence path를 도입했습니다.
* `f4ba5bd4ad86` — `test(evidence): reject migration history drift`

  * migration version·script·checksum·success drift를 거부하는 경계를 검증합니다.

**관련 파일과 final-state 내용**

* `docker/postgres-init.sql`은 `wallet`, `betting`, `settlement`, `admin` database를 `sportsbook` owner로 조건부 생성합니다.
* `scripts/cold_gate/migration_evidence.py`

  * 각 DB의 `flyway_schema_history` 조회
  * materialized source migration의 version·filename·checksum 계산
  * observed history와 source-derived expected history의 exact equality 요구
  * 전체 inventory가 정확히 25개 row인지 요구
* Redis contract는 네 개 service store를 구분하고, E2E runtime은 허용된 Redis instance만 직접 질의합니다.

**Diff focus**

```text
+ PostgreSQL 첫 초기화 시 네 service database 생성
+ application startup에서 Flyway가 schema를 실제 적용
+ live flyway_schema_history를 source checksum과 비교
+ Redis service마다 별도 named volume 및 persistence contract 부여
```

### External Development Steps

#### PostgreSQL/Flyway

1. PostgreSQL named volume을 생성합니다.
2. 비어 있는 volume의 최초 PostgreSQL 기동에서 init SQL을 실행합니다.
3. 네 database를 생성합니다.
4. Wallet, Betting, Settlement, Admin이 각 DB에 연결합니다.
5. 각 service의 Flyway startup이 migration을 실제 적용합니다.
6. 모든 service가 시작한 뒤 live `flyway_schema_history`를 조회합니다.
7. 정확히 25개 성공 row와 source checksum 일치를 확인합니다.

#### Redis

1. 네 개의 별도 Redis named volume을 생성합니다.
2. 각 Redis를 AOF `everysec`, `noeviction` 설정으로 기동합니다.
3. Risk/Odds/Wallet/Gateway가 자기 store에만 연결하도록 합니다.
4. E2E 실행 중 생성되는 key와 state가 store 간에 섞이지 않는지 확인합니다.
5. cold gate 종료 시 네 volume을 project-scoped cleanup으로 제거합니다.

### Code Connection

* Hibernate는 migrated schema를 생성하는 대신 validate하는 계약이므로 Flyway 실행 누락은 application readiness 실패로 이어집니다.
* E2E의 bet placement/recovery/oracle은 PostgreSQL row와 여러 Redis key를 함께 관찰하므로 G-04/G-05가 성립하지 않으면 Thread 9의 cross-store oracle도 성립하지 않습니다.

### Evidence Boundary

* **Directly observed in repository**

  * 네 DB bootstrap SQL
  * 네 Redis service와 storage contract
  * 25개 migration inventory 검증 코드
* **Required/inferred from repository**

  * PostgreSQL init script의 실제 실행
  * 각 service Flyway의 실제 migration 적용
  * Redis volumes와 AOF state의 실제 생성
* **Actual execution not observable from Git**

  * 특정 DB에 언제 migration이 적용됐는지
  * 실제 PostgreSQL volume contents
  * Redis key/AOF contents와 복구 이력
  * 실제 `migrations.tsv`

### Ordering — conceptual execution order

`PostgreSQL/Redis volume 생성 → PostgreSQL bootstrap → service DB 연결 → Flyway 적용 → application readiness → migration/history evidence → E2E storage use → scoped volume cleanup`

---

## Packet E-05 — Thread 5

### Thread Identity

* **Existing Thread**
* **Thread 5**
* **한국어 제목:** Kafka 토픽과 소비자 준비 계약
* **English title:** Kafka Topic and Consumer Readiness Contract

### Gaps

* **G-06 — Kafka KRaft state, topic 생성, consumer assignment**

### Repository Evidence

**Representative commit**

* `f9e15158d474` — `build(kafka): provision topics without mutation`

  * topic-init service가 missing topic만 생성하고 existing topic의 partition·replication·retention drift는 자동 변경하지 않고 실패하도록 구성했습니다.

**관련 파일과 final-state 내용**

* `docker/kafka/topics.manifest`

  * source topic 14개
  * DLT 9개
  * 모든 topic은 partition 3, replication factor 1
  * DLT retention은 최소 `604800000` ms
* `docker/kafka/wait-consumer-assignments.sh`

  * `gateway-bets`와 `betting-resolution` group이 지정된 9개 topic-partition을 정확히 소유해야 성공
  * 180초 timeout 안에 exact assignment가 되지 않으면 실패

**Diff focus**

```text
+ authoritative topic manifest 추가
+ missing topic만 생성
+ existing topic drift는 mutate하지 않고 fail
+ fixed consumer groups의 exact active assignment를 startup gate로 사용
```

### External Development Steps

1. Kafka KRaft data volume과 broker metadata를 생성합니다.
2. auto topic creation이 비활성화된 broker를 기동합니다.
3. topic-init을 실행해 23개 topic을 생성하거나 existing configuration을 검증합니다.
4. Gateway와 Betting consumer를 시작합니다.
5. 두 consumer group이 기대한 9개 partition을 실제로 assign받을 때까지 기다립니다.
6. assignment gate가 성공한 뒤 Settlement와 후속 service를 시작합니다.
7. E2E 종료 후 topic configuration과 Kafka state를 evidence로 캡처합니다.

### Code Connection

* Fixture publisher와 E2E scenario는 manifest에 정의된 topic이 실제 존재한다는 전제에서 record를 게시합니다.
* Settlement는 consumer assignment gate 이후에만 시작하므로 assignment는 단순 관찰값이 아니라 startup dependency입니다.
* Thread 12의 ordering/replay/DLT invariants는 이 broker state 위에서 검증됩니다.

### Evidence Boundary

* **Directly observed in repository**

  * exact topic manifest
  * idempotent/fail-closed topic-init
  * exact consumer assignment gate
* **Required/inferred from repository**

  * Kafka cluster metadata와 topic의 실제 생성
  * consumer group join과 partition assignment
* **Actual execution not observable from Git**

  * broker cluster ID
  * actual topic IDs/configs
  * partition offsets와 records
  * 실제 group generation/assignment 시점

### Ordering — conceptual execution order

`KRaft volume/broker 생성 → topic-init → producer/consumer service 기동 → exact group assignment → Settlement 기동 → E2E records → topic evidence → volume cleanup`

---

## Packet E-06 — Thread 6

### Thread Identity

* **Existing Thread**
* **Thread 6**
* **한국어 제목:** 서비스 신뢰 경계와 제한 노출
* **English title:** Service Trust Boundaries and Controlled Exposure

### Gaps

* **G-07 — Runtime credential/keypair와 제한 노출**

### Repository Evidence

**Representative commit**

* `43d20c34e2eb` — `build(gate): generate isolated runtime secrets`

  * cold-gate별 service keys, RSA keypair, PostgreSQL/Grafana password와 environment injection을 도입했습니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/secrets.py`

  * `config/required-secrets.txt`에 정의된 정확히 11개 key 생성
  * 모든 key가 서로 다르고 최소 길이를 만족해야 함
  * OpenSSL로 RSA 2048-bit private/public key 생성
  * private/public key file mode를 `0600`으로 설정
  * 독립적인 PostgreSQL/Grafana password 생성
  * Gateway와 Toxiproxy용 사용 가능한 loopback port 두 개 선택
  * `COMPOSE_PROJECT_NAME`과 모든 runtime value를 Compose environment에 주입
* README와 Compose contract상 host publication은 Gateway, Toxiproxy control, Grafana만 loopback으로 제한됩니다.

**Diff focus**

```text
+ 매 run fresh service credentials 생성
+ RSA private/public PEM 생성
+ DB/Grafana password와 host ports를 environment에 결합
+ internal backend와 loopback-only publication을 runtime contract로 사용
```

### External Development Steps

1. run-owned private directory를 `0700`으로 생성합니다.
2. 11개 service key와 두 password를 생성합니다.
3. RSA private/public keypair를 생성하고 key files를 `0600`으로 제한합니다.
4. public PEM을 Gateway/Admin runtime environment에 주입합니다.
5. caller/callee별 key mapping을 정확한 environment variable에 주입합니다.
6. 두 개의 사용 가능한 loopback port를 선택하고 Compose에 전달합니다.
7. internal network와 loopback publication이 계약대로 형성됐는지 확인합니다.
8. cleanup에서 private key와 모든 runtime secret-bearing directory를 제거합니다.

### Code Connection

* Secret preflight gate는 required secret이 누락되면 application startup 전에 실패합니다.
* Service-to-service authentication, Admin JWT, PostgreSQL startup, Grafana startup은 모두 이 generated state를 필요로 합니다.
* EvidenceStore는 같은 generated values를 redaction set으로 사용합니다.

### Evidence Boundary

* **Directly observed in repository**

  * secret inventory와 생성 알고리즘
  * RSA generation command
  * environment mapping
  * loopback port allocation
* **Required/inferred from repository**

  * key files의 실제 생성
  * Compose process와 container environment로의 실제 주입
  * host port의 실제 binding
* **Actual execution not observable from Git**

  * 모든 실제 key/password/PEM 값
  * 실제 port 번호
  * credential 사용 시점과 rotation 이력
  * cleanup 후 host memory에 값이 남았는지 여부

실제 secret 값은 추정하거나 복원할 수 없으며, 이 Packet에도 포함하지 않습니다.

### Ordering — conceptual execution order

`run ownership 확보 → secrets/keypair 생성 → permissions 설정 → environment 구성 → preflight → service startup/loopback binding → evidence redaction → secret directory 제거`

---

## Packet E-07 — Thread 7

### Thread Identity

* **Existing Thread**
* **Thread 7**
* **한국어 제목:** 프로젝트 범위 메트릭과 로그
* **English title:** Project-Scoped Metrics and Logs

### Gaps

* **G-08 — Observability data plane와 Docker discovery 권한**

### Repository Evidence

**Representative commit**

* `aa55201ffca6` — `build(observability): compose isolated metrics and logs`

  * Prometheus, Loki, Grafana, Promtail과 project-owned volumes/healthchecks를 Compose topology에 추가했습니다.

**관련 파일과 final-state 내용**

* `compose.observability.yaml`

  * Prometheus 72시간 retention과 `prometheus-data`
  * Loki와 `loki-data`
  * Grafana와 `grafana-data`
  * generated Grafana password
  * loopback-only Grafana port
  * Promtail에 read-only Docker socket bind
* `observability/promtail/promtail.yml`

  * Docker service discovery
  * exact `COMPOSE_PROJECT_NAME` label filter
  * project/service/container label 부여
  * Docker/JSON pipeline parsing

**Diff focus**

```text
+ metrics/logging services를 cold Compose project에 포함
+ Prometheus/Loki/Grafana별 persistent volume 추가
+ Promtail이 Docker socket으로 동일 Compose project container만 발견
```

### External Development Steps

1. 세 observability named volume을 생성합니다.
2. Prometheus, Loki, Grafana, Promtail containers를 시작합니다.
3. Grafana generated admin password를 주입합니다.
4. Promtail container에 Docker socket을 read-only로 mount할 수 있도록 host permission을 제공합니다.
5. Promtail이 정확한 Compose project label의 container만 발견하게 합니다.
6. live application metrics를 scrape하고 logs를 Loki로 전송합니다.
7. final-state evidence와 bounded raw logs를 수집합니다.
8. cold-gate cleanup에서 project observability containers와 volumes를 제거합니다.

### Code Connection

* Grafana health는 전체 Compose wait boundary에 포함됩니다.
* Promtail Docker discovery는 run-specific Compose label을 사용하므로 Thread 14의 project ownership과 직접 연결됩니다.
* 수집된 logs는 Thread 15의 redaction/storage와 Thread 16의 attestation에 연결됩니다.

### Evidence Boundary

* **Directly observed in repository**

  * observability service definitions
  * data volumes
  * Docker socket bind
  * project label filter
* **Required/inferred from repository**

  * Docker socket의 실제 존재와 접근 허용
  * metrics/logs의 실제 생성·수집
  * Grafana datasource/provisioning의 실제 활성화
* **Actual execution not observable from Git**

  * Prometheus TSDB contents
  * Loki chunks
  * Grafana database와 session state
  * 실제 수집 로그와 metric values

### Ordering — conceptual execution order

`host socket permission 확인 → observability volumes 생성 → Loki/Prometheus 기동 → Grafana/Promtail 기동 → project-scoped discovery → metrics/log 수집 → evidence capture → volume cleanup`

---

## Packet E-08 — Thread 8

### Thread Identity

* **Existing Thread**
* **Thread 8**
* **한국어 제목:** 결정적 Avro 픽스처와 Kafka 프로브
* **English title:** Deterministic Avro Fixtures and Kafka Probes

### Gaps

* **G-09 — Fixture JAR staging과 live Kafka publish/probe**

### Repository Evidence

**Representative commits**

* `269cf445cb2a` — `build(fixtures): stage executable publisher`

  * Java 17, shared protocol identity, shaded dependencies, main class를 검사한 뒤 fixture JAR을 run-owned output에 게시합니다.
* `f350328d23ea` — `feat(fixtures): publish broker-acknowledged records`

  * Kafka producer가 `acks=all`, idempotence enabled로 record를 전송하고 topic/key/partition/offset/hash/fingerprint receipt를 반환합니다.
* `e64f9040551c` — `feat(fixtures): probe exact Kafka records`

  * 지정된 topic/partition/offset을 직접 seek하고 payload, headers, hash, 선택적 Avro decode를 반환하는 probe를 추가합니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/fixtures.py`

  * fixture JAR 경로가 현재 cold-gate 소유인지 검증
  * canonical JSON input을 임시 read-only 파일로 생성
  * `docker compose run --rm --no-deps`로 JAR과 input을 read-only mount
  * fixed fixture type/topic/fingerprint/partition 검증
  * poison record의 topic/key/partition/hash/fingerprint 검증
  * temporary input 제거

**Diff focus**

```text
+ exact shaded fixture publisher를 run-owned path에 stage
+ broker acknowledgement가 포함된 publication receipt 생성
+ exact topic/partition/offset을 읽는 isolated probe group 사용
+ payload/hash/header/Avro identity를 live record와 대조
```

### External Development Steps

1. materialized shared source와 run-owned Maven repository를 사용해 fixture JAR을 실제 build합니다.
2. JAR manifest, Java bytecode, embedded dependencies와 shared version을 검증합니다.
3. E2E별 canonical fixture input을 임시 파일로 생성합니다.
4. live Compose network 안에서 fixture publisher를 일회성 container로 실행합니다.
5. Kafka broker acknowledgement와 topic/partition/offset receipt를 받습니다.
6. 필요 시 새로운 isolated probe consumer를 생성해 정확한 offset의 record를 읽습니다.
7. key, payload hash, schema fingerprint, headers, Avro contents를 검증합니다.
8. 임시 input과 transient publisher build output을 제거합니다.

### Code Connection

* Thread 5가 Kafka topic/assignment를 소유하지만, 특정 Avro record를 만드는 책임은 Thread 8에 있습니다.
* Thread 9가 scenario 전체를 소유하지만, deterministic encoding, publication receipt, exact record probe는 Thread 8의 독립적인 관점입니다.
* Thread 12의 DLT invariants는 poison record와 partition-preserving probe 결과를 사용합니다.

### Evidence Boundary

* **Directly observed in repository**

  * fixture schema/type whitelist
  * deterministic publisher/probe implementation
  * receipt 검증
  * run-owned JAR staging
* **Required/inferred from repository**

  * live Kafka에 record를 실제 publish
  * probe consumer를 생성하고 exact offset을 read
* **Actual execution not observable from Git**

  * 실제 topic offset
  * record bytes와 headers
  * ephemeral probe group ID
  * broker acknowledgement receipt
  * 실제 fixture JAR hash

### Ordering — conceptual execution order

`fixture JAR build/stage → canonical input 생성 → one-shot publisher 실행 → broker receipt 검증 → scenario processing → exact record probe → temporary input 제거 → Kafka volume cleanup`

---

## Packet E-09 — Thread 9

### Thread Identity

* **Existing Thread**
* **Thread 9**
* **한국어 제목:** 다중 저장소 E2E 오라클과 장애 주입
* **English title:** Cross-Store E2E Oracles and Fault Injection

### Gaps

* **G-10 — 13개 cross-store E2E scenario 실행**
* **G-11 — Toxiproxy fault mutation과 restoration**

### Repository Evidence

**Representative commits**

* `28a36bf8d802` — `test(e2e): define isolated scenario identities`

  * scenario별 canonical UUIDv7-style identity를 분리해 live stores 간 fixture 충돌을 막습니다.
* `6c10ff7682d9` — `build(gate): control scoped dependency faults`

  * allowlisted Toxiproxy proxy만 조작하고 timeout toxic을 제한된 형태로 만들도록 control client를 도입합니다.
* `d827a4d76c50` — `test(gate): restore every injected dependency fault`

  * injected fault가 모든 경로에서 복원되는 계약을 검증합니다.

**관련 파일과 final-state 내용**

* `e2e/scenarios.py`

  * 정확히 13개 scenario
  * 고정 순서 실행
  * 시작 전 Settlement consumer assignment 대기
  * 각 scenario가 성공한 뒤 이름을 pass inventory에 추가
* `scripts/cold_gate/chaos.py`

  * 허용 proxy: `betting_to_risk`, `betting_to_wallet`, `settlement_to_wallet`
  * proxy enable/disable
  * downstream timeout toxic create/delete
* `scenario_02_risk_recovery.py`

  * Risk proxy disable
  * pending placement과 cross-store non-effects 검증
  * `finally`에서 proxy enable 복원
  * 이후 recovery state 검증

**Diff focus**

```text
+ scenario별 격리 identity 제공
+ 13개 scenario를 fixed order로 실행
+ HTTP·PostgreSQL·Redis·Kafka state를 함께 oracle로 사용
+ scoped Toxiproxy state를 변경하고 finally에서 복원
```

### External Development Steps

#### Cross-store E2E

1. 완전히 기동된 clean cold stack을 준비합니다.
2. consumer assignment가 준비됐는지 확인합니다.
3. scenario별 account·market·event fixture를 실제 service/DB/Kafka에 seed합니다.
4. HTTP 요청, database row, Redis key, Kafka record와 outbox state를 함께 관찰합니다.
5. 13개 scenario를 정확한 순서로 한 번 실행합니다.
6. 각 scenario pass를 `scenarios.tsv`의 입력으로 전달합니다.
7. E2E 이후 final readiness와 integrity scan을 다시 수행합니다.

#### Fault injection

1. allowlisted proxy의 현재 state를 확인합니다.
2. Risk 또는 Wallet 경로를 disable하거나 response timeout toxic을 추가합니다.
3. failure-state application behavior와 durable recovery checkpoint를 확인합니다.
4. `finally` 경로에서 proxy를 다시 enable하거나 toxic을 삭제합니다.
5. 복원 후 recovery와 terminal cleanup이 진행되는지 확인합니다.

### Code Connection

* Thread 10의 placement recovery, Thread 11의 settlement revision, Thread 12의 ordering/replay/DLT, Thread 13의 audit/correlation은 모두 이 하나의 live E2E execution에서 검증됩니다.
* 따라서 동일한 external run을 Thread 10–13에 다시 소유시키지 않고 Thread 9를 Primary Owner로 지정했습니다.
* Thread 16은 이 실행을 수행하는 것이 아니라 그 결과를 semantic evidence로 직렬화합니다.

### Evidence Boundary

* **Directly observed in repository**

  * 13개 scenario inventory와 순서
  * cross-store query/assertion 코드
  * Toxiproxy mutation과 `finally` restoration
* **Required/inferred from repository**

  * live databases/Redis/Kafka에 fixture와 state가 실제 생성
  * failure state를 실제 주입하고 recovery를 기다림
* **Actual execution not observable from Git**

  * scenario별 실제 bet/account/event IDs
  * 생성된 rows, keys, records
  * 실제 pass/fail 결과
  * fault restoration이 특정 run에서 완료됐는지
  * `scenarios.tsv` contents

### Ordering — conceptual execution order

`clean stack → assignments 준비 → scenario seed → 정상/장애 요청 → cross-store oracle → fault 복원 → recovery oracle → 13개 pass 집계 → final readiness/evidence`

---

## Packet E-14 — Thread 14

### Thread Identity

* **Existing Thread**
* **Thread 14**
* **한국어 제목:** 소유권 기반 콜드 릴리스 게이트
* **English title:** Ownership-Scoped Cold Release Gate

### Gaps

* **G-12 — Cold-gate lock, ownership, cleanup lifecycle**

### Repository Evidence

**Representative commits**

* `5ef2d1349379` — `build(gate): own cold release lifecycle`

  * context 생성, secrets, build, checks, success cleanup과 failure cleanup을 하나의 lifecycle로 결합했습니다.
* `110d7ea31c1e` — `ci(archive): verify the cold release once`

  * archive workflow가 history, unit tests, cold gate, evidence upload를 한 번의 ordered run으로 수행하도록 정의했습니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/context.py`

  * `sb-gate-<sha12>-<nonce8>` project name
  * `.runtime/cold-gate.lock`
  * run-specific runtime/evidence directory
  * owner marker
  * 동시에 두 gate가 같은 worktree를 소유하지 못하도록 방지
* `scripts/cold_gate/gate.py`

  * context → secrets → Compose binding → evidence store → build → checks → cleanup
  * 실패 시 logs capture와 best-effort scoped cleanup
  * primary error, log error, cleanup error를 함께 보존
* `scripts/cold_gate/cleanup.py`

  * `compose down --volumes --remove-orphans --rmi local`
  * project resources가 모두 사라졌는지 재검증
  * detached sources 제거
  * staged JAR symlink/generation 제거
  * runtime directory와 lock 제거
  * evidence는 남김

**Diff focus**

```text
+ exact commit과 nonce에서 run identity 생성
+ global worktree lock과 owner markers 생성
+ 성공·오류·interrupt 모두 같은 scoped cleanup 경로 사용
+ Docker resources, sources, JARs, secrets/runtime을 ownership 검증 후 제거
```

### External Development Steps

1. cold-gate global lock을 획득합니다.
2. unique project identity와 run-owned runtime/evidence paths를 만듭니다.
3. 모든 Compose invocation에 같은 `COMPOSE_PROJECT_NAME`을 사용합니다.
4. source, artifacts, secrets, Docker resources를 그 project/run에 귀속시킵니다.
5. success 시 complete evidence를 확인한 뒤 scoped cleanup을 실행합니다.
6. build/startup/check/interrupt failure 시 logs를 가능한 범위에서 수집하고 같은 cleanup을 실행합니다.
7. containers, networks, volumes, local images, source worktrees, staged JARs, secrets, runtime directory와 lock이 제거됐는지 확인합니다.
8. evidence directory만 보존합니다.

### Code Connection

* Threads 1–9의 external state는 모두 이 lifecycle 안에서 생성됩니다.
* Thread 15의 evidence는 cleanup 대상에서 의도적으로 제외됩니다.
* Thread 16의 `cleanup.tsv`는 cleanup이 끝난 상태를 attestation합니다.

### Evidence Boundary

* **Directly observed in repository**

  * project naming과 lock
  * ownership markers
  * success/failure cleanup algorithm
  * exact cleanup command와 target validation
* **Required/inferred from repository**

  * lock directory와 Docker resources의 실제 생성
  * cleanup command의 실제 실행
* **Actual execution not observable from Git**

  * 특정 run의 project name
  * 실제 container/network/volume/image ID
  * cleanup 후 residual resources
  * 실패 시 best-effort cleanup의 성공 여부

### Ordering — conceptual execution order

`lock/context 생성 → secrets/build/stack/checks → evidence complete 검증 → Docker/source/JAR/runtime cleanup → zero-residual 확인 → lock 제거 → evidence 유지`

---

## Packet E-15 — Thread 15

### Thread Identity

* **Existing Thread**
* **Thread 15**
* **한국어 제목:** 증거 저장과 비밀 제거
* **English title:** Evidence Storage and Secret Redaction

### Gaps

* **G-13 — Evidence persistence, redaction, CI artifact retention**

### Repository Evidence

**Representative commit**

* `627c34edbd44` — `build(evidence): redact runtime credentials`

  * generated service keys, passwords, PEM/JWT-like material이 evidence에 남지 않도록 redaction과 rejection을 도입했습니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/evidence.py`

  * required evidence filename whitelist
  * bounded file size
  * write-once semantics
  * temporary file 후 `os.replace`를 통한 atomic write
  * logs는 허용된 21개 service만 기록
  * 모든 write와 final verify에서 redactor clean check
  * complete mode에서는 required inventory 누락을 거부
* `.gitignore`는 `evidence/`와 `.runtime/`을 제외합니다. 따라서 actual run evidence는 일반 Git diff에 남지 않습니다.
* `.github/workflows/archive.yml`은 `evidence/cold-gate/**`를 GitHub artifact로 업로드하고 retention을 14일로 설정합니다.

**Diff focus**

```text
+ generated secret set을 evidence redactor에 전달
+ evidence path·filename·size·service inventory 제한
+ write-once atomic file publication
+ cleanup 이후 evidence만 유지
+ CI 성공/실패 후 external artifact store로 업로드
```

### External Development Steps

1. run-owned evidence directory를 생성합니다.
2. Thread 16이 전달하는 contents를 write-once file로 기록합니다.
3. 모든 generated secret과 secret-like material을 redact합니다.
4. 작성 직후와 complete verification 시 전체 evidence를 다시 scan합니다.
5. Docker/runtime cleanup 이후에도 evidence directory를 남깁니다.
6. GitHub Actions에서 workflow가 실제 실행된 경우 evidence tree를 artifact service에 업로드합니다.
7. artifact service가 configured retention 기간 동안 artifact를 보존하고 만료 처리하도록 합니다.

### Code Connection

* EvidenceStore는 Thread 6이 생성한 secret values를 redaction 입력으로 사용합니다.
* Thread 16은 evidence의 의미와 내용을 생성하지만 storage 안전성은 Thread 15가 소유합니다.
* Thread 17의 workflow activation이 없으면 GitHub artifact upload 단계도 발생하지 않습니다.

### Evidence Boundary

* **Directly observed in repository**

  * evidence inventory와 path
  * redaction/write-once/size-bound implementation
  * Git ignore
  * Actions upload와 14일 retention 설정
* **Required/inferred from repository**

  * local evidence files의 실제 생성
  * Actions artifact service로의 실제 upload
* **Actual execution not observable from Git**

  * 특정 run evidence contents
  * redaction 결과
  * artifact ID, upload 시각, download 이력
  * retention 만료 여부

### Ordering — conceptual execution order

`evidence directory 생성 → live contents capture → redact/atomic write → inventory 검증 → runtime cleanup → cleanup evidence 작성 → local evidence 유지 → CI artifact upload/retention`

---

## Packet E-16 — Thread 16

### Thread Identity

* **Existing Thread**
* **Thread 16**
* **한국어 제목:** 의미 기반 릴리스 증명
* **English title:** Semantic Release Attestation

### Gaps

* **G-14 — Live semantic attestation capture**

### Repository Evidence

**Representative commit**

* `6184fc6137c` — `build(evidence): record locked release identities`

  * orchestration SHA, source SHA, lock identity, JAR hash를 runtime-derived release evidence로 기록합니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/checks.py`의 capture 순서:

  1. release identity
  2. rendered Compose identity
  3. stack startup
  4. Kafka topics
  5. Flyway migration history
  6. 13개 E2E scenarios
  7. final runtime images/container state
  8. readiness와 Wallet integrity
  9. scenario pass inventory
  10. bounded logs
* `scripts/cold_gate/migration_evidence.py`는 live database history를 materialized source-derived expected history와 대조합니다.
* `EvidenceStore.REQUIRED_FILES`에는 `run.tsv`, `services.lock`, `jars.sha256`, `images.tsv`, `compose.sha256`, `topics.tsv`, `migrations.tsv`, `readiness.tsv`, `scenarios.tsv`, `compose-ps.json`, `cleanup.tsv`가 포함됩니다.

**Diff focus**

```text
+ source/JAR/config identity를 run 시작 시 기록
+ Kafka/Flyway/E2E/readiness를 live system에서 관찰
+ final image/container/log state와 cleanup zero-state를 함께 기록
+ required evidence inventory가 완성되지 않으면 release 실패
```

### External Development Steps

1. exact orchestration SHA와 clean worktree 상태를 확인합니다.
2. materialized source SHA와 staged JAR hash를 실제 filesystem에서 계산합니다.
3. generated secrets가 적용된 Compose rendering을 redact한 뒤 digest합니다.
4. live broker에서 topic configuration을 조회합니다.
5. live PostgreSQL에서 Flyway history를 조회합니다.
6. 13개 scenario를 실행하고 pass inventory를 받습니다.
7. E2E 이후 container image, embedded JAR, health, readiness, Wallet integrity를 다시 관찰합니다.
8. 모든 service log를 bounded/redacted 형태로 캡처합니다.
9. cleanup 이후 scoped resource가 0인지 확인하고 `cleanup.tsv`를 작성합니다.
10. required inventory가 완전한지 최종 검증합니다.

### Code Connection

* 단순 config snapshot이 아니라 source → artifact → rendered config → live runtime → E2E → cleanup까지 하나의 semantic chain으로 연결됩니다.
* G-14는 external state를 "만드는" 책임보다, 이미 생성된 external state를 의미 있는 evidence로 관찰·연결하는 책임입니다.
* Storage/redaction은 Thread 15가 소유합니다.

### Evidence Boundary

* **Directly observed in repository**

  * evidence schema와 capture sequence
  * live query·hash·comparison 코드
  * required file inventory
* **Required/inferred from repository**

  * 실제 runtime에서 각 query/capture 수행
  * observed values를 evidence files로 직렬화
* **Actual execution not observable from Git**

  * source/JAR/image/config hashes
  * live topic/migration/readiness values
  * scenario pass inventory
  * cleanup zero-state
  * 최종 attestation bundle

### Ordering — conceptual execution order

`release identity → rendered config identity → live stack → topic/migration evidence → E2E → final runtime/readiness/log evidence → cleanup → cleanup evidence → complete attestation 검증`

---

## Packet E-17 — Thread 17

### Thread Identity

* **Existing Thread**
* **Thread 17**
* **한국어 제목:** 선형 개발 히스토리 정책
* **English title:** Linear Development History Policy

### Gaps

* **G-15 — History policy 실행과 Git/CI control binding**

### Repository Evidence

**Representative commits**

* `f969a81afbda` — `build(history): expose the archive history guard`

  * local repository의 history를 읽고 policy verifier를 실행하는 CLI를 추가합니다.
* `9d30ca7a03c9` — `build(history): parse measured linear records`

  * commit별 parent, path, numstat를 policy input으로 파싱합니다.
* `110d7ea31c1e` — `ci(archive): verify the cold release once`

  * workflow에서 full checkout 후 history guard를 첫 번째 control로 실행합니다.

**관련 파일과 final-state 내용**

* `scripts/cold_gate/history_repository.py`

  * `git log --reverse --no-renames ... --numstat`로 현재 checkout에서 reachable한 전체 history를 읽습니다.
* `scripts/cold_gate/history_policy.py`

  * minimum commit depth
  * single-parent linearity
  * conventional subject
  * generated artifact exclusion
  * production/test adjacency
  * terminal release/docs pair를 검증합니다.
* 현재 workflow:

  * `workflow_dispatch` 정의
  * push trigger는 `orchestration`
  * full-history checkout
  * history guard → unit tests → cold gate → artifact upload 순서
* 현재 공개 branch:

  * `main` 하나
  * `protected: false`

**Diff focus**

```text
+ full reachable history를 verifier input으로 읽음
+ history guard를 release CLI로 노출
+ CI workflow 첫 단계에 history guard 배치
```

### External Development Steps

1. shallow clone이 아닌 full reachable history를 checkout/fetch합니다.
2. 검증 대상 exact orchestration SHA를 checkout합니다.
3. `history_guard.py`를 release build보다 먼저 실행합니다.
4. policy failure 시 cold gate가 실행되지 않게 release control을 구성합니다.
5. GitHub Actions를 사용할 경우 workflow가 실제 active release branch 또는 수동 dispatch에서 실행되게 합니다.
6. 지속적인 server-side enforcement가 목적이라면 active branch의 required status/check 정책과 verifier run을 연결합니다.
7. 성공한 history check와 cold gate가 같은 exact SHA를 대상으로 했는지 유지합니다.

현재 상태에서는 `orchestration` push trigger와 실제 `main` branch가 일치하지 않습니다. 다만 `workflow_dispatch`가 있으므로 "workflow를 전혀 실행할 수 없다"고 단정할 수는 없습니다.

### Code Connection

* Verifier는 repository 안의 코드이지만, 그 코드의 존재 자체는 정책 집행이 아닙니다.
* Full history fetch, exact SHA checkout, workflow invocation, required check binding은 Git diff 밖의 control-plane state입니다.
* History check 이후의 cold gate와 artifact upload는 Threads 14–16과 연결됩니다.

### Evidence Boundary

* **Directly observed in repository**

  * full-history verifier
  * workflow 순서와 trigger
  * 현재 공개 branch가 `main` 하나이며 unprotected라는 상태
* **Required/inferred from repository**

  * exact SHA의 full-history checkout
  * release control point에서 verifier 실행
* **Actual execution not observable from Git**

  * 수동 workflow 실행 여부
  * 과거 Actions run 결과
  * 별도 외부 CI나 local release process에서의 guard 실행
  * 조직·repository rule 또는 다른 enforcement mechanism
  * 실제 required status check 구성

### Ordering — conceptual execution order

`full history fetch → exact SHA checkout → history guard → unit tests → cold release gate → evidence upload → external release decision`

---

# Part III — Proposed New Thread Packets

## NEW_THREAD 판정 없음

독립적인 신규 Thread는 제안하지 않습니다.

### 판정 이유

1. **외부 resource lifecycle 전체**

   * 이미 Thread 14가 unique project, lock, ownership, failure cleanup과 recovery lifecycle을 소유합니다.
   * 별도의 "Infrastructure Lifecycle" Thread를 만드는 것은 Thread 14의 재정의가 됩니다.

2. **CI 및 evidence 보존**

   * Evidence storage/upload는 Thread 15, semantic contents는 Thread 16, history/control binding은 Thread 17로 자연스럽게 분리됩니다.
   * CI가 여러 Thread를 실행한다는 이유만으로 독립된 Thread를 만들 필요가 없습니다.

3. **Database/Kafka/observability/fixture external state**

   * 각각 Thread 4, 5, 7, 8이 명확한 Primary Owner입니다.
   * 자체 commit 집합은 존재하지만 이미 확정된 기존 Thread의 핵심 관점을 실제 환경에서 완성하는 단계입니다.

4. **Execution-host provisioning**

   * 여러 subsystem을 관통하지만, repository 안에 독립된 provisioning implementation이나 자체 lifecycle을 형성하는 representative commit 집합이 없습니다.
   * 따라서 Part IV의 project-level step으로 분류하는 것이 적절합니다.

5. **Thread 10–13의 runtime state**

   * 실제 실행은 동일한 13-scenario cold-stack run에서 함께 만들어집니다.
   * 별도 external-state Thread나 각 Thread별 중복 Gap으로 만들지 않고 Thread 9를 Primary Owner로 정했습니다.

---

# Part IV — Project-Level External Steps

## P-01 — Cold-Gate Execution Host Provisioning

### Classification

* **PROJECT_LEVEL_EXTERNAL_STEP**

### 관련 Threads

* 1, 2, 3, 6, 7, 8, 14, 15, 16, 17

### Repository Evidence

README가 명시하는 실행 전제는 다음과 같습니다.

* Git
* Docker와 Compose v2
* OpenSSL
* Python 3.12
* complete JDK 17
* 21개 container와 7개 application image build를 감당할 Docker daemon capacity
* clean exact commit
* reserved build environment override 부재

추가로 final-state source가 요구하는 host interaction은 다음과 같습니다.

* detached Git worktree를 만들 수 있는 filesystem
* `.runtime`, `docker/.jars`, `evidence`를 만들 수 있는 write permission
* Docker daemon 및 Compose plugin 접근
* Promtail에 전달할 read-only Docker socket
* Maven dependencies와 container images를 resolve할 수 있는 network 또는 사전 cache
* loopback host ports 할당 가능성

### Required External Steps

1. local host 또는 CI runner에 필요한 toolchain versions를 설치·선택합니다.
2. Docker daemon을 시작하고 caller에게 필요한 permission을 제공합니다.
3. Docker image build, 21-container runtime, named volumes, Kafka/PostgreSQL/observability data를 감당할 CPU·memory·disk를 제공합니다.
4. Git worktree와 runtime/evidence directory를 만들 수 있는 filesystem permission을 제공합니다.
5. Promtail이 Docker socket을 read-only로 mount할 수 있게 합니다.
6. Maven dependency와 public container image가 network 또는 cache에서 resolve되게 합니다.
7. reserved build environment variables가 외부 shell/runner에서 주입되지 않게 합니다.
8. full Git history와 G-01의 locked service refs/objects를 execution clone에 제공합니다.

### Code Connection

* 이 전제가 없으면 source materialization, JAR build, key generation, Compose startup, observability, evidence capture 중 하나 이상이 시작 전에 실패합니다.
* 그러나 host provisioning 자체를 수행하는 Terraform, image-builder, runner bootstrap 또는 다른 독립적인 infrastructure implementation은 repository에서 확인되지 않습니다.

### Evidence Boundary

* **Directly observed in repository**

  * tool/version prerequisites
  * Docker capacity requirement
  * filesystem·socket·build command 사용
* **Required/inferred from repository**

  * 실제 host/runner provisioning
  * permissions, capacity, network/cache availability
* **Actual execution not observable from Git**

  * 설치된 실제 versions
  * Docker daemon configuration
  * CPU/RAM/disk capacity
  * Maven/container cache contents
  * socket 권한
  * runner provisioning 이력

### Documentation Action

신규 Development Thread를 만들지 않고, 프로젝트 시작부에 한 번만 적용되는 **"Cold-Gate Execution Environment Preflight"** 프로젝트 수준 체크리스트를 추가하는 것이 적절합니다.

### Ordering — conceptual execution order

`toolchain/daemon 준비 → capacity/permissions 확인 → full history 및 locked refs 제공 → reserved environment 검사 → Thread 1 source materialization부터 실행`

---

## 채택하지 않은 External-State 범주

전체 history와 repository contents에서 구체적인 implementation 또는 requirement evidence를 찾지 못했으므로 다음 항목은 Gap으로 추가하지 않았습니다.

* production/staging deployment
* Kubernetes, Helm, Terraform 또는 cloud resource provisioning
* DNS/domain verification
* TLS certificate 발급·배치
* OAuth application/provider 등록
* redirect URI 등록
* webhook 외부 등록
* object storage 또는 IAM
* 외부 API credential 발급
* scheduler/cron 외부 등록
* backup/restore lifecycle

이 프로젝트에서 확인되는 release boundary는 **production deployment가 아니라, exact source를 materialize하고 private cold Compose stack을 한 번 기동·검증한 뒤 scoped cleanup하고 evidence를 남기는 경계**입니다. Repository evidence 없이 일반적인 운영 단계를 추가하지 않았습니다.
