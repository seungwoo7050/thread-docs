# External-State Development Gap Audit

**Repository:** `tmp-sportsbook-wallet-service`
**Audit snapshot:** `main` at `c9a05f4d652f24ac97d3e1cd753f69cef2725ff3`

## 감사 범위와 결론

Existing Development Thread Index는 **17개 확정 Thread의 식별자·제목·commit 소유 구조만** 사용했으며, 별도의 Thread 학습 문서나 해설서는 사용하지 않았다. 

Repository는 `main`이 기본 branch인 public repository이며, 감사 시점의 recursive tree는 `truncated:false`로 반환되었다. Commit history는 100개 단위로 끝까지 조회하여 6번째 페이지가 빈 결과임을 확인했다. 따라서 아래 판단은 현재 tree뿐 아니라 repository에 남은 전체 commit history를 대상으로 한다.

최종적으로 채택한 Gap은 **7개**다.

* `EXISTING_THREAD`: 4개
* `NEW_THREAD`: 2개 Gap을 소유하는 신규 Thread 1개
* `PROJECT_LEVEL_EXTERNAL_STEP`: 1개
* Existing Thread 보충 Packet: Thread 3, 13, 15
* Proposed New Thread: `NT-DB-01`
* 기존 17개 Thread의 제목이나 commit 구성을 변경할 필요는 없다.

---

# Part I — Gap Index

## G-01 — PostgreSQL 실행 데이터베이스와 접근 주체 생성

**Classification:** `NEW_THREAD`
**Primary Owner:** `NT-DB-01 — PostgreSQL 프로비저닝과 Flyway 스키마 활성화`
**Related Threads:** 1, 4, 5, 6, 10, 12, 13, 14

### Repository Evidence

Repository는 JPA, PostgreSQL runtime driver, Flyway를 build dependency로 추가하고, 실행 시 `WALLET_DB_URL`, `WALLET_DB_USER`, `WALLET_DB_PASSWORD`를 통해 PostgreSQL에 연결하도록 구성한다. 동일 datasource 위에서 Flyway가 활성화되고 Hibernate는 `ddl-auto: validate`로 최종 schema를 검증한다.

`docs/operations.md`도 PostgreSQL을 필수 runtime dependency로 구분하고, local 기본 password를 production에서 교체하도록 요구한다.

### Required External Step

코드가 실행될 대상 환경에는 다음 외부 상태가 필요하다.

1. 지갑 서비스가 접근 가능한 PostgreSQL server와 database 또는 그에 상응하는 전용 namespace
2. 지갑 서비스의 database identity와 credential
3. `WALLET_DB_URL`, `WALLET_DB_USER`, `WALLET_DB_PASSWORD`의 runtime 주입
4. 동일 datasource identity가 V1–V4 DDL을 적용하고 이후 runtime DML을 수행할 수 있도록 한 권한 구성
5. 애플리케이션 인스턴스와 database 사이의 실제 network reachability

Repository는 별도 Flyway datasource를 구성하지 않으므로, 현재 구성대로라면 애플리케이션 datasource identity가 migration 실행에 필요한 권한도 가져야 한다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

Git으로 확인할 수 없는 항목은 다음과 같다.

* 실제 database host, database name, schema 또는 network
* 실제 role과 grant
* 실제 credential 값과 저장 위치
* staging/production database 생성 여부
* 실제 connection 성공 시점
* database의 HA, backup 또는 restore 구성

### Documentation Action

`NT-DB-01` 신규 Thread의 첫 번째 Gap으로 문서화한다. 개별 영속성 Thread마다 database 생성 절차를 중복하지 않는다.

---

## G-02 — Flyway V1–V4 실제 적용과 migration history 유지

**Classification:** `NEW_THREAD`
**Primary Owner:** `NT-DB-01 — PostgreSQL 프로비저닝과 Flyway 스키마 활성화`
**Related Threads:** 1, 4, 5, 6, 10, 12, 13, 14

### Repository Evidence

Repository에는 다음 네 migration이 있다.

* V1: `account`, `ledger_entry`
* V2: `wallet_operation`
* V3: `outbox_stream`, `outbox_event`
* V4: `wallet_adjustment`

각 migration은 별도 대표 commit으로 추가되었다. V3의 outbox 구조는 V2의 `wallet_operation`을 참조하고, V4의 adjustment 구조도 operation과 연결되므로 이들은 임의 순서로 분리 적용할 수 있는 독립 SQL 파일 집합이 아니다.

현재 설정은 다음 경계를 만든다.

* Flyway enabled
* migration location: `classpath:db/migration`
* `baseline-on-migrate:false`
* Hibernate schema creation이 아니라 `ddl-auto:validate`

따라서 target database에 migration이 성공적으로 적용되지 않으면 애플리케이션의 schema validation과 persistence layer가 성립하지 않는다.

Testcontainers 기반 smoke fixture는 임시 PostgreSQL을 만들고 해당 datasource로 애플리케이션을 시작하도록 구성되어 있으며, migration constraint를 실제 DB에서 검증하는 테스트도 존재한다. 그러나 이것은 production migration의 실행 증거가 아니라 실행 가능성을 검증하는 repository artifact다.

### Required External Step

1. G-01에서 준비된 target database를 Flyway가 관리할 대상으로 결정한다.
2. `baseline-on-migrate:false` 조건과 target의 기존 상태가 양립하는지 확인한다.
3. migration-capable application startup 또는 동등한 Flyway 실행을 통해 V1 → V2 → V3 → V4를 적용한다.
4. Flyway migration metadata와 migration checksum이 현재 artifact와 일치하는지 확인한다.
5. Flyway 완료 뒤 Hibernate validation이 통과하는지 확인한다.
6. migration 실패 시 API traffic, recovery worker, integrity scanner, outbox publisher가 불완전한 schema에 접근하지 않도록 배포 순서를 통제한다.
7. 실패한 migration을 복구할 때 restore, recreate, repair 중 무엇을 사용할지 해당 환경의 운영 규칙으로 결정한다.

마지막 항목의 구체적 복구 방식은 repository가 정하지 않으므로 특정 명령이나 정책을 프로젝트 사실로 단정해서는 안 된다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

확인할 수 없는 정보는 다음과 같다.

* 어떤 환경에 어느 migration까지 적용되었는지
* 실제 `Flyway` metadata 내용
* 적용 시간, 실행자, checksum 상태
* migration 실패·repair·rollback·restore 이력
* schema drift 여부
* production Hibernate validation 성공 여부

### Documentation Action

G-01과 함께 `NT-DB-01`의 핵심 lifecycle로 문서화한다. V1–V4 SQL의 도메인 의미는 기존 Thread가 계속 소유하고, 신규 Thread는 **외부 database에 schema가 실제로 활성화되는 과정과 실패 경계**만 소유한다.

---

## G-03 — 내부 호출자 API key의 발급·배포·주입·회전

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 15 — **내부 서비스 인증과 권한 경계 / Internal Service Authentication and Authorization**
**Related Threads:** 2, 3, 16

### Repository Evidence

실행 설정은 기본값 없이 다음 다섯 credential을 요구한다.

* `WALLET_PLATFORM_API_KEY`
* `WALLET_GATEWAY_API_KEY`
* `WALLET_BETTING_SERVICE_API_KEY`
* `WALLET_SETTLEMENT_SERVICE_API_KEY`
* `WALLET_ADMIN_API_KEY`

각 값은 nonblank, 최소 32자, 상호 구별되어야 하며 조건이 맞지 않으면 startup validation이 실패한다.

요청 인증은 정확히 하나의 `X-Internal-Service`와 하나의 `X-Internal-Api-Key` header pair를 받아 caller를 결정한다. Credential은 digest로 보관되고 비교된다.

### Required External Step

1. 다섯 caller에 대해 서로 다른 충분한 길이의 secret을 생성한다.
2. 값들을 선택된 secret store 또는 CI/CD/runtime secret mechanism에 저장한다.
3. 지갑 서비스 인스턴스에 정확한 environment variable 이름으로 주입한다.
4. 각 호출 서비스에 자신의 caller name과 일치하는 credential을 안전하게 배포한다.
5. 지갑과 caller 양측을 대상으로 실제 인증 성공 및 잘못된 caller/key 거부를 확인한다.
6. credential 교체 시 wallet side와 caller side의 cutover 순서를 조정한다.

현재 model은 caller별 단일 credential만 나타내며 old/new key overlap을 위한 dual-key 상태는 확인되지 않는다. 따라서 무중단 회전 방식이나 grace period는 repository에서 정해진 사실이 아니다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

Git은 다음을 증명하지 않는다.

* 실제 secret 값
* 생성 시점과 생성 주체
* 사용한 secret manager
* caller service의 실제 설정
* staging/production 주입 여부
* 회전 또는 폐기 이력

### Documentation Action

Thread 15에 External-State Supplement를 추가한다. Secret 값 예시나 복원 시도 없이 **발급 → wallet 주입 → caller 배포 → 검증 → 교체**의 conceptual execution order만 기록한다.

---

## G-04 — Kafka destination 준비와 outbox delivery 명시적 활성화

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 13 — **리스 기반 FIFO 아웃박스 전달 / Leased FIFO Outbox Delivery**
**Related Threads:** 8, 9, 12

### Repository Evidence

Outbox publisher가 사용하는 destination은 source에서 다음 세 topic 이름으로 고정된다.

* `wallet.debited.v1`
* `wallet.debit-failed.v1`
* `wallet.credited.v1`

Kafka record에는 outbox의 partition key가 record key로 전달되고 `event-id` header가 추가된다. Producer는 `acks=all`, idempotence 활성화 및 제한된 in-flight 설정을 사용한다.

중요하게도 outbox scheduling은 `WALLET_OUTBOX_ENABLED`에 연결되고 기본값은 `false`다. Publisher bean 자체도 property가 `true`일 때만 활성화된다.

운영 문서는 production에서 Kafka destination, downstream consumer contract, alerting을 확인한 다음에만 outbox delivery를 명시적으로 활성화하도록 요구한다.

Repository에는 Testcontainers Kafka를 이용해 실제 broker로 publish하고 key/header/payload를 검증하는 테스트가 있다. 이것은 임시 test broker에서의 semantic gate이며 운영 Kafka의 프로비저닝 또는 활성화 증거는 아니다.

### Required External Step

1. 지갑 runtime에서 접근할 수 있는 Kafka bootstrap endpoint를 준비한다.
2. 세 destination이 다음 중 선택된 운영 방식으로 사용할 수 있게 한다.

   * 사전에 topic을 생성하거나
   * 의도적으로 허용된 broker auto-creation으로 생성
3. downstream consumer가 topic 이름과 Avro payload 계약을 처리할 준비가 되었는지 확인한다.
4. 실제 bootstrap endpoint를 `WALLET_KAFKA_BOOTSTRAP`에 주입한다.
5. G-07의 monitoring/alerting 준비를 완료한다.
6. 준비가 끝난 환경에서만 `WALLET_OUTBOX_ENABLED=true`를 주입한다.
7. 활성화 직후 pending, oldest-pending, retry, lease takeover, publish 상태를 확인한다.

Repository는 partition 수, replication factor, retention 기간, broker authentication 방식 또는 ACL 값을 규정하지 않는다. 따라서 이 값들을 임의로 프로젝트 요구사항으로 추가해서는 안 된다.

### 실제 수행 여부 확인 가능성

**운영 환경에 대해서는 확인 불가.**

확인되는 것은 broker integration test를 위한 코드와 설정뿐이다. 다음은 확인되지 않는다.

* 운영 broker 또는 cluster
* 실제 topic 생성
* partition/replication/retention 정책
* consumer deployment
* 운영 bootstrap value
* `WALLET_OUTBOX_ENABLED=true` 적용 여부
* 실제 production event publication
* 실제 backlog 또는 retry 상태

### Documentation Action

Thread 13에 External-State Supplement를 추가한다. Thread 12는 DB transaction 안에서 outbox를 기록하는 단계만 계속 소유하며, **DB에 기록된 event를 실제 Kafka로 보내는 외부 준비와 활성화**는 Thread 13이 단독 소유한다.

---

## G-05 — `shared-protocol` dependency materialization과 Docker 기반 semantic gate 환경

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 3 — **재현 가능한 빌드와 의미 검증 / Reproducible Build and Semantic Verification**
**Related Threads:** 13, 15, 16

### Repository Evidence

`pom.xml`은 `com.sportsbook:shared-protocol:1.0.0`을 compile dependency로 요구하지만, 해당 artifact를 가져올 별도 Maven repository를 정의하지 않는다.

CI workflow는 같은 repository에서 `ref: shared-protocol`을 별도로 checkout하고, 그 결과물을 runner의 Maven repository에 `install`한 다음 wallet build를 실행하도록 구성한다. 또한 Java 17과 semantic verification 실행을 명시한다.

Smoke fixture는 PostgreSQL, Redis, Kafka Testcontainers를 동적으로 준비한다. 운영 문서도 전체 verification에 Docker가 필요하다고 명시한다.

감사 시점의 branch 목록에는 `main`만 존재한다. 반면 workflow의 push filter는 `wallet-service`이고 별도 checkout은 `shared-protocol` ref를 요구한다. 이 사실은 역사적으로 해당 ref가 없었다는 증거는 아니지만, **현재 repository 상태만으로 workflow dependency와 main push trigger가 완결된다고 확인할 수 없다는 증거**다.

### Required External Step

1. `shared-protocol:1.0.0`을 build environment에서 해석 가능하게 만든다.

   * workflow가 기대하는 ref를 실제로 제공해 local Maven repository에 설치하거나
   * artifact registry에 publish하고 build configuration을 그 방식에 맞춘다.
2. Java 17과 Maven wrapper를 실행할 수 있는 runner를 준비한다.
3. Testcontainers가 사용할 Docker daemon 또는 동등한 container runtime을 제공한다.
4. PostgreSQL, Redis, Kafka test image를 가져올 network 및 registry access를 제공한다.
5. ephemeral container를 실행할 충분한 runner resource와 port/network isolation을 제공한다.
6. 해당 상태에서 unit/integration/semantic gate를 실행한다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

Workflow 파일은 CI 실행 결과가 아니다. Git만으로는 다음을 확인할 수 없다.

* runner에 Docker가 실제로 있었는지
* image pull이 성공했는지
* `shared-protocol` checkout 또는 Maven install 성공 여부
* CI workflow가 실제로 실행된 시점과 결과
* semantic gate 통과 여부
* 현재 `main` push가 자동으로 CI를 트리거하는지

### Documentation Action

Thread 3에 "build inputs outside wallet source tree" 보충 절을 추가한다. Protocol artifact, local Maven repository, CI runner와 Docker resources를 Git 밖의 build state로 명확히 분리한다.

---

## G-06 — Versioned artifact publication과 실제 runtime deployment

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 3 — **재현 가능한 빌드와 의미 검증 / Reproducible Build and Semantic Verification**
**Related Threads:** 1, 13, 15

### Repository Evidence

`build(release): release wallet service 1.0.0` commit은 POM version을 `1.0.0-SNAPSHOT`에서 `1.0.0`으로 변경한다. 이 diff에는 artifact registry publication, container image 생성, deployment target 변경 또는 runtime rollout이 포함되지 않는다.

현재 recursive tree는 complete하지만 다음 deployment artifact는 포함하지 않는다.

* Dockerfile 또는 Compose deployment
* Kubernetes/Helm manifest
* Terraform 또는 cloud resource definition
* CD/deploy workflow
* environment-specific deployment descriptor

Tree에는 단일 CI workflow, source, migration, test, documentation이 포함된다.

현재 GitHub Releases 목록도 비어 있다. 다만 이것은 다른 artifact registry나 별도 deployment system에 publication이 없었다는 증거까지 되지는 않는다.

### Required External Step

소스의 `1.0.0` 표기를 실제 실행 상태로 만들려면 다음 중 프로젝트가 선택한 배포 방식에 해당하는 단계가 필요하다.

1. verified source로 deployable JAR 또는 동등한 artifact를 생성한다.
2. 선택된 artifact registry 또는 deployment system에 artifact를 전달한다.
3. Java 17을 실행할 runtime environment를 준비한다.
4. G-01, G-03, G-04와 관련된 database, secret, Kafka 설정을 runtime에 주입한다.
5. 최초 startup에서 Flyway와 Hibernate validation을 통과시킨다.
6. health endpoint와 실제 API readiness를 확인한다.
7. outbox는 Kafka와 monitoring 준비가 끝난 뒤 별도로 활성화한다.

특정 cloud, container platform, registry 또는 rollout strategy는 repository에서 확인되지 않으므로 하나를 프로젝트 사실로 선정할 수 없다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

* `1.0.0` artifact가 실제 publish되었는지
* JAR인지 container image인지
* registry 또는 deployment platform
* production/staging environment
* 실제 배포 시점과 대상 revision
* rollout·rollback·restart 이력
* 실제 runtime instance 수
* 실제 environment variable 값

### Documentation Action

Thread 3에 "release marker와 release execution의 차이"를 보충한다. Version commit을 실제 배포 완료 증거로 해석하지 않도록 Evidence Boundary를 명시한다.

---

## G-07 — Health·Prometheus signal의 외부 scrape, probe, alert 등록

**Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
**Primary Owner:** Project-level Operations
**Related Threads:** 3, 11, 13, 14

### Repository Evidence

Application은 Actuator health/info/prometheus/metrics endpoint를 노출하도록 구성한다. Observability commit은 Micrometer, Prometheus, structured logging 및 tracing 관련 build dependency를 추가했다.

Outbox는 다음과 같은 delivery signal을 제공한다.

* claimed/published/retried counters
* fenced completion과 lease takeover
* pending/leased gauges
* oldest pending seconds

Integrity scanner는 account, operation, recovery queue, adjustment 관련 drift gauge, scan failure, last checked timestamp를 제공한다. Integrity drift 또는 scan failure가 있으면 health를 `DOWN`, 미실행 상태면 `UNKNOWN`으로 보고한다.

그러나 complete repository tree에는 Prometheus scrape target, alert rule, alert receiver 또는 deployment probe 설정이 없다.

### Required External Step

1. 실제 배포 endpoint를 health probe 대상에 등록한다.
2. `/actuator/prometheus` 또는 선택한 metric export 경로를 monitoring system에 등록한다.
3. 최소한 다음 failure signal에 대한 alerting과 routing을 구성한다.

   * outbox backlog 또는 oldest-pending 증가
   * 반복되는 outbox retry
   * lease takeover/fenced completion 이상
   * integrity total drift
   * integrity scan failure
   * integrity health `DOWN`
   * 장기간 integrity scan 미실행
4. alert가 실제 운영 담당자 또는 incident channel에 도달하는지 검증한다.
5. G-04의 outbox 활성화 전에 outbox 관련 monitoring readiness를 확인한다.

Repository는 threshold, evaluation window, severity, receiver 또는 monitoring product를 정하지 않는다.

### 실제 수행 여부 확인 가능성

**확인 불가.**

Source가 metric을 생성한다는 사실과 외부 monitoring system이 이를 실제로 수집한다는 사실은 다르다. Git은 scrape target, dashboard, alert rule, receiver, notification test 또는 incident history를 증명하지 않는다.

### Documentation Action

신규 Development Thread로 만들지 않고 project-level operations appendix 또는 deployment readiness checklist로 문서화한다.

그 이유는 다음과 같다.

* 관련 signal은 Thread 13과 Thread 14가 이미 소유한다.
* repository에는 in-process instrumentation commit은 있지만 외부 monitoring platform lifecycle을 구현하는 project-specific artifact가 없다.
* 하나의 기존 Thread에 소유시키면 다른 subsystem signal을 중복 설명하게 된다.
* 독립적인 신규 Thread보다 소수의 project-level 외부 실행 단계로 보완하는 것이 최소 변경 원칙에 맞는다.

---

# Part II — Existing Thread Supplement Packets

## Packet ET-03

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 3
* **한국어 제목:** 재현 가능한 빌드와 의미 검증
* **English title:** Reproducible Build and Semantic Verification

### Gaps

* G-05 — `shared-protocol` dependency materialization과 Docker 기반 semantic gate 환경
* G-06 — Versioned artifact publication과 실제 runtime deployment

---

### Repository Evidence

#### Representative commits

| Commit | Subject | 관련 파일 | 이 Packet에서의 의미 |
| --- | - | -- | -- |
| `34927fdb76a9aa417b59dccae309242c5ada7f79` | `ci(wallet): verify Java 17 builds`   | `.github/workflows/wallet-ci.yml`   | Java 17, 별도 protocol checkout/install, wallet verification이라는 CI 실행 계약을 만든다.  |
| `53caad57cf6cbf6bd93bd1214b3692ca2658e132` | `test(gate): provision live wallet dependencies`  | `WalletSmokeFixture.java`, smoke test config | semantic gate가 임시 PostgreSQL·Redis·Kafka container라는 실행 중 외부 상태를 필요로 함을 보여준다. |
| `30f643cfabf3cc9450b8b2a1e0d54af3046683fb` | `test(gate): publish through a real Kafka broker` | `KafkaOutboxDeliveryTest.java` 등 | mock이 아니라 ephemeral broker에서 publish semantics를 검증하는 gate를 만든다.   |
| `944e302273586b8b1a7ffcfbf3a5249cbc48b4a4` | `build(release): release wallet service 1.0.0` | `pom.xml`   | release version 표기는 남기지만 publication/deployment는 남기지 않는 경계를 보여준다. |

CI와 release diff는 각각 workflow orchestration과 POM version 변경을 직접 보여준다.

Test fixture는 ephemeral dependency를 동적으로 만들지만 scheduler는 test에서 제어한다. 따라서 test resource 생성은 production resource provisioning과 동일한 사실이 아니다.

#### 관련 final-state configuration

`pom.xml`:

```text
com.sportsbook:shared-protocol:1.0.0
Java 17
Spring Boot packaging/build plugins
Testcontainers PostgreSQL, Kafka, Redis dependencies
```

`.github/workflows/wallet-ci.yml`:

```text
push branch: wallet-service
checkout ref: shared-protocol
install shared protocol into runner Maven repository
run clean verify and semantic gates
```

현재 repository branch는 `main` 하나만 확인된다. 따라서 workflow 파일을 현재 repository 상태에서 실행 가능한 완성된 CI 이력으로 간주해서는 안 된다.

---

### External Development Steps

#### G-05

1. Protocol source ref 또는 published artifact를 준비한다.
2. 동일 build에서 `shared-protocol:1.0.0`을 해석할 수 있게 한다.
3. Java 17과 Maven을 갖춘 runner를 준비한다.
4. Docker/Testcontainers 실행 권한과 image registry 접근을 준비한다.
5. PostgreSQL·Redis·Kafka test containers가 정상 기동되는지 확인한다.
6. repository가 정의한 semantic gate를 수행한다.

#### G-06

1. verified revision을 deployable artifact로 package한다.
2. 선택된 registry 또는 deployment target으로 artifact를 전달한다.
3. runtime에 database·secret·Kafka·monitoring 설정을 주입한다.
4. startup migration과 validation을 확인한다.
5. health/API readiness를 확인한다.
6. outbox는 별도의 readiness 조건을 충족한 후 활성화한다.

---

### Code Connection

| External step | Code/configuration connection  |
| - | --- |
| Protocol artifact materialization  | Wallet source가 protocol value/event type을 compile dependency로 사용하며 POM이 `shared-protocol:1.0.0`을 요구한다. |
| Docker-capable verification runner | Smoke fixture와 Kafka delivery test가 Testcontainers resource를 동적으로 만든다.   |
| Java 17 runtime  | POM과 CI가 Java 17을 build/runtime baseline으로 사용한다. |
| Artifact publication   | `1.0.0` POM version은 package identity를 만들지만 배포 target을 만들지 않는다. |
| Runtime deployment  | `application.yml`의 datasource, API key, Kafka, scheduler, Actuator 설정은 실행 인스턴스에서만 실제 상태가 된다.  |
| Readiness verification | Flyway 완료, Hibernate validation, security property validation, health endpoint가 startup/runtime 성공 여부와 연결된다. |

---

### Evidence Boundary

**Directly observed in repository**

* Java 17 build requirement
* `shared-protocol:1.0.0` dependency
* protocol source checkout/install CI 단계
* Testcontainers 기반 PostgreSQL·Redis·Kafka fixture
* `1.0.0` version commit
* application runtime 설정
* 현재 complete tree에 deployment manifest가 없음

**Required/inferred from repository**

* build runner에 protocol artifact와 Docker runtime이 필요함
* container image 접근이 필요함
* deployable artifact의 전달과 runtime 시작이 필요함
* runtime에 external configuration을 주입해야 함

**Actual execution not observable from Git**

* CI 수행과 성공 여부
* protocol artifact publication
* container 실행 결과
* release artifact publication
* 실제 배포 또는 rollback
* staging/production runtime state

---

### Ordering

다음은 **conceptual execution order**이며 실제 과거 수행 이력이 아니다.

1. Protocol source/artifact materialization
2. Java 17 + Maven + Docker runner preparation
3. Unit/integration/semantic verification
4. Versioned artifact packaging
5. Artifact publication 또는 deployment system 전달
6. Runtime resource와 secret 연결
7. Startup migration 및 validation
8. Health/API verification
9. Kafka·monitoring readiness 이후 outbox activation

---

## Packet ET-13

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 13
* **한국어 제목:** 리스 기반 FIFO 아웃박스 전달
* **English title:** Leased FIFO Outbox Delivery

### Gaps

* G-04 — Kafka destination 준비와 outbox delivery 명시적 활성화

---

### Repository Evidence

#### Representative commits

| Commit | Subject | 관련 파일 | 중요성   |
| --- | - | -- | -- |
| `1384125c368d1cad2b70d07aff4ce5b17c60b527` | `config(kafka): configure an idempotent producer` | `KafkaProducerConfig.java`, `application.yml`   | broker endpoint와 producer delivery semantics를 runtime configuration에 연결한다. |
| `8ebceef58ddd65f757dd0d92bc1d4c4322ab66d3` | `config(outbox): activate safe delivery polling`  | `WalletServiceApplication.java`, `application.yml` | scheduling을 도입하지만 outbox delivery를 기본 비활성 상태로 둔다.  |
| `30f643cfabf3cc9450b8b2a1e0d54af3046683fb` | `test(gate): publish through a real Kafka broker` | Kafka integration test  | 실제 ephemeral broker에서 record contract를 검증한다. |
| `d368eb06d3d6b59d39a5f0b08c11cb9cc269fd9c` | `feat(outbox): expose lease retry and oldest-pending metrics` | `OutboxMetrics.java`, `OutboxBacklogSampler.java`  | 활성화 후 외부 monitoring이 수집해야 할 delivery signal을 제공한다. |

Producer configuration commit은 idempotence 및 delivery constraint를 설정하며, scheduling commit은 `WALLET_OUTBOX_ENABLED:false`를 명시한다.

`WalletEventFactory`가 정한 세 topic과 dispatcher가 설정하는 key/header가 external Kafka contract의 직접 근거다.

#### 필요한 final-state excerpt

```text
WALLET_KAFKA_BOOTSTRAP
WALLET_OUTBOX_ENABLED=false   # default

wallet.debited.v1
wallet.debit-failed.v1
wallet.credited.v1
```

Outbox publisher는 enable property가 true일 때만 생성되고, lease owner·poll interval·batch/in-flight 값을 이용해 scheduled publish를 수행한다.

---

### External Development Steps

1. Kafka endpoint와 network path를 준비한다.
2. 세 topic을 선택된 provisioning policy 아래 사용할 수 있게 한다.
3. consumer contract와 consumer deployment readiness를 확인한다.
4. 실제 bootstrap endpoint를 runtime에 주입한다.
5. G-07 monitoring target과 alerting을 준비한다.
6. `WALLET_OUTBOX_ENABLED=true`를 의도적으로 설정한다.
7. startup 후 publisher instance owner와 scheduling 상태를 확인한다.
8. pending, oldest pending, retries, lease takeover, publish count를 관찰한다.
9. Kafka 장애 시 DB outbox가 보존되고 retry가 진행되는지 확인한다.
10. 비정상 backlog가 지속되면 활성화 유지·비활성화·broker 복구 중 운영 대응을 결정한다.

---

### Code Connection

| External step | Code/runtime behavior   |
| - | -- |
| Kafka endpoint 준비   | `spring.kafka.bootstrap-servers`가 `WALLET_KAFKA_BOOTSTRAP`에 연결된다. |
| Topic availability  | Outbox row의 topic 값이 세 exact destination 중 하나로 생성된다.  |
| Partition/key contract | Dispatcher가 `partitionKey`를 Kafka record key로 전달한다.   |
| Consumer readiness  | Avro payload와 event-id header가 외부 consumer contract가 된다. |
| 명시적 enable | `ConditionalOnProperty` 때문에 enable 전에는 publisher가 생성되지 않는다. |
| Monitoring | Backlog sampler와 metrics는 outbox enable 상태에서 delivery health를 나타낸다.  |
| Failure/retry | Lease, retry policy, fenced completion은 broker failure 뒤 재전송 lifecycle과 연결된다. |

---

### Evidence Boundary

**Directly observed in repository**

* 세 topic 이름
* producer idempotence/acks 설정
* record key와 event-id header
* outbox default disabled
* property-controlled publisher
* ephemeral broker integration test
* backlog/retry/lease metrics

**Required/inferred from repository**

* 접근 가능한 Kafka endpoint
* destination availability
* consumer contract readiness
* enable flag의 runtime 설정
* production monitoring

**Actual execution not observable from Git**

* 운영 cluster와 topic
* broker policy와 ACL
* actual consumer state
* enablement 시점
* production event publication
* retry/backlog incident 또는 recovery

---

### Ordering

**Conceptual execution order**

1. Kafka endpoint 결정
2. Topic availability와 consumer contract 확인
3. Runtime bootstrap configuration 주입
4. Monitoring/alerting 연결
5. Wallet deployment
6. Outbox를 disabled 상태로 health 확인
7. `WALLET_OUTBOX_ENABLED=true`
8. 초기 publish와 metric 확인
9. 장애·재시작·lease takeover 조건 검증

Thread 12의 outbox DB 기록은 이 순서보다 먼저 동작할 수 있다. Publisher가 비활성화되어도 outbox row는 생성될 수 있으므로, "outbox 기록 성공"을 "외부 event 전달 성공"으로 표현해서는 안 된다.

---

## Packet ET-15

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 15
* **한국어 제목:** 내부 서비스 인증과 권한 경계
* **English title:** Internal Service Authentication and Authorization

### Gaps

* G-03 — 내부 호출자 API key의 발급·배포·주입·회전

---

### Repository Evidence

#### Representative commits

| Commit | Subject   | 관련 파일 | 중요성 |
| --- | --- | -- | --- |
| `96a19d2128ff995b58ae8133c29c8e50a18793ae` | `feat(security): validate caller API keys` | `WalletSecurityProperties.java`  | caller별 key, 최소 길이, 상호 구별 startup invariant를 만든다.   |
| `371455c29b31015ab7f42bbf3b1ab7fdc4919297` | `feat(security): authenticate internal API keys` | `InternalApiKeyAuthenticationFilter.java` | 외부 caller identity와 credential header를 runtime 인증으로 연결한다. |
| `e293f84bc719a8c823ddb28fdc926c078d70ee4e` | `test(security): verify environment-bound caller keys` | `application.yml`, binding test  | 실제 environment variable 이름과 누락 시 startup failure를 고정한다.   |
| `d4e6cfdefa3709c35b2ad69ad021b1cbc8b42887` | `feat(security): compare caller credential digests` | `WalletCredentials.java`   | raw key를 장기 보유하지 않고 digest 기반 비교를 사용한다. |

Validation commit과 environment binding test는 실제 secret이 repository에 없어야 하며 실행환경에서 공급되어야 함을 직접 보여준다.

Authentication filter는 `X-Internal-Service`, `X-Internal-Api-Key`가 각각 정확히 하나여야 한다는 request contract를 만든다.

#### 필요한 final-state excerpt

```text
WALLET_PLATFORM_API_KEY
WALLET_GATEWAY_API_KEY
WALLET_BETTING_SERVICE_API_KEY
WALLET_SETTLEMENT_SERVICE_API_KEY
WALLET_ADMIN_API_KEY

X-Internal-Service
X-Internal-Api-Key
```

Repository가 확인하는 invariant:

```text
각 key는 nonblank
각 key는 최소 32자
다섯 값은 서로 달라야 함
caller name과 해당 key가 일치해야 함
```

---

### External Development Steps

1. 다섯 caller identity별 secret을 생성한다.
2. 각 secret을 승인된 secret store에 저장한다.
3. wallet runtime에 다섯 environment variable을 주입한다.
4. gateway, platform, betting, settlement, admin caller에 각각 대응 credential을 배포한다.
5. wallet startup validation을 확인한다.
6. caller별 허용 route에 정상 요청을 보내 인증과 authorization을 검증한다.
7. 잘못된 caller/key, 중복 header, 누락 header가 거부되는지 검증한다.
8. 교체가 필요하면 wallet과 caller 사이의 coordinated cutover를 수행한다.
9. 사용 종료된 값을 secret store와 caller configuration에서 폐기한다.

---

### Code Connection

| External step  | Code/runtime behavior |
| -- | --- |
| Key 생성   | `WalletSecurityProperties`의 최소 길이·상호 구별 validation  |
| Wallet 주입   | `application.yml`의 무기본 environment placeholder   |
| Caller 배포   | Filter가 caller header와 API key를 함께 인증   |
| Secret 보호   | `WalletCredentials`가 digest 비교를 수행   |
| Coordinated rotation | Caller별 하나의 configured key만 나타나므로 양측 값 불일치는 즉시 인증 실패로 연결됨 |
| 검증 | Security filter와 route authorization이 request를 허용 또는 거부   |

---

### Evidence Boundary

**Directly observed in repository**

* 다섯 caller와 environment variable 이름
* 최소 길이와 distinct constraint
* header 이름과 cardinality
* digest comparison
* 누락/잘못된 설정의 startup 또는 request rejection behavior

**Required/inferred from repository**

* 실제 secret 생성
* wallet과 caller에 대한 안전한 배포
* 양측 설정의 일치
* 교체 시 coordinated cutover

**Actual execution not observable from Git**

* secret 값
* secret manager
* 생성자와 생성 시점
* caller 측 설정
* 배포 환경
* 회전·폐기·incident history

---

### Ordering

**Conceptual execution order**

1. Caller 목록 확정
2. 다섯 secret 생성
3. Secret store 등록
4. Wallet environment 주입
5. Caller environment 주입
6. Wallet과 caller 배포
7. Positive/negative authentication 검증
8. 운영 monitoring
9. 필요 시 coordinated rotation 및 이전 값 폐기

실제 secret이나 과거 rotation chronology는 이 Packet에 포함하지 않는다.

---

# Part III — Proposed New Thread Packets

## Packet NT-DB-01

### Thread Identity

* **Type:** Proposed New Thread
* **Temporary ID:** `NT-DB-01`
* **한국어 제목:** PostgreSQL 프로비저닝과 Flyway 스키마 활성화
* **English title:** PostgreSQL Provisioning and Flyway Schema Activation

### Gaps

* G-01 — PostgreSQL 실행 데이터베이스와 접근 주체 생성
* G-02 — Flyway V1–V4 실제 적용과 migration history 유지

---

### 신규 Thread 판정 이유

이 관점은 기존 persistence Thread의 단순 부수 설정보다 넓다.

1. **자체 lifecycle이 있다.**
   Database와 role 생성 → connectivity → migration → validation → application traffic 허용 → schema upgrade/recovery 순서가 있다.

2. **여러 단계가 하나의 개발 문제를 형성한다.**
   PostgreSQL dependency만 추가하거나 SQL 파일만 보유한다고 시스템이 성립하지 않는다. 실제 target, credential, DDL 권한, migration state, validation이 연결되어야 한다.

3. **별도 실패·복구 조건이 있다.**
   Connection failure, 권한 부족, nonempty schema와 `baseline-on-migrate:false` 충돌, migration failure, checksum mismatch, Hibernate validation failure가 API 비즈니스 로직과 다른 실패 축을 형성한다.

4. **여러 subsystem을 관통한다.**
   Account, ledger, operation outcome, adjustment, outbox, recovery, integrity scanner가 모두 동일 schema activation에 의존한다.

5. **대표 commit 집합을 선정할 수 있다.**
   PostgreSQL/Flyway build dependency, runtime configuration, V1–V4 migration, live dependency fixture가 서로 연결된 commit 연쇄로 존재한다.

따라서 이를 Thread 1, 4, 5, 6, 10, 12 등에 분산 소유시키면 동일 외부 database lifecycle이 반복된다. 반대로 project-level 체크리스트 하나로 축소하면 migration 순서·failure boundary·schema validation이라는 독립 학습 관점이 손실된다.

---

### Representative Commits

| Commit | Subject  | 이 Thread에서 중요한 이유  |
| --- | -- | --- |
| `f7204c41b5c2ca8f2e43f5b5c909efc67c9e091d` | `build(storage): add JPA PostgreSQL and Flyway` | 애플리케이션이 in-memory 상태가 아니라 PostgreSQL과 Flyway runtime을 요구하게 된 build 경계다.  |
| `446164f375d8ffdf4b4b000df4219e7826d33ac5` | `config(runtime): configure PostgreSQL JPA and Flyway`   | DB URL/user/password, Flyway enable, baseline policy, Hibernate validation을 하나의 startup contract로 만든다. |
| `ec784c955a3e9cbc1ae82d8b20830f33859deab5` | `build(flyway): create the final account and ledger schema` | V1에서 account 및 ledger authoritative tables와 constraints를 만든다.   |
| `15b73def43a912c403313c20617186c3235ae235` | `build(flyway): create authoritative wallet outcomes` | V2에서 durable idempotency/outcome table을 만든다.  |
| `c942e3d7cb148732f87638780371ffc3eed9c6c4` | `build(flyway): create an ordered transactional outbox`  | V3에서 V2 operation에 연결된 outbox schema를 만든다. |
| `f9c7f6ab6b61860164340ca5e3fc5e48b3f23559` | `build(flyway): create adjustment proof and recovery table` | V4에서 operation과 연결된 adjustment/recovery proof schema를 만든다.   |
| `53caad57cf6cbf6bd93bd1214b3692ca2658e132` | `test(gate): provision live wallet dependencies`   | ephemeral PostgreSQL에 runtime configuration을 연결해 schema startup을 검증하는 fixture를 만든다.  |
| `c2bf2dc351c2a600b5a01893050b02ef2d846547` | `test(flyway): reject orphan and duplicate revision proofs` | migration 결과의 FK와 uniqueness가 실제 PostgreSQL에서 작동해야 함을 검증한다.  |

Dependency 및 runtime configuration의 관련 diff는 PostgreSQL driver, Flyway, datasource environment binding, `baseline-on-migrate:false`, Hibernate validation을 추가한다.

V1–V4의 대표 migration commit은 schema가 단계적으로 확장되었음을 보여준다.

---

### Thread-Direct Repository Evidence

### 1. Build/runtime dependency

Relevant `pom.xml` state:

```text
spring-boot-starter-data-jpa
org.postgresql:postgresql
org.flywaydb:flyway-core
```

이 dependency set은 source에서 SQL 파일을 단순 보관하는 것이 아니라 PostgreSQL datasource와 Flyway migration runner를 application startup에 포함한다.

### 2. Runtime datasource contract

Relevant `application.yml` state:

```text
WALLET_DB_URL
WALLET_DB_USER
WALLET_DB_PASSWORD
WALLET_DB_CONNECTION_TIMEOUT_MS

spring.jpa.hibernate.ddl-auto: validate
spring.flyway.enabled: true
spring.flyway.locations: classpath:db/migration
spring.flyway.baseline-on-migrate: false
```

이 설정은 다음 순서를 요구한다.

```text
reachable database
→ datasource authentication
→ Flyway migration
→ Hibernate schema validation
→ application runtime
```

### 3. Migration topology

#### V1 — Account and ledger

직접 생성되는 주요 상태:

* `account`
* `ledger_entry`
* account balance constraints
* ledger sequence와 transfer constraints
* query/index structures

V1은 HOUSE 및 EXTERNAL_PAYMENT UUID를 사용자 account row에서 제외한다. 따라서 이 repository를 근거로 시스템 상대계정용 row seed가 필요하다고 판단해서는 안 된다.

#### V2 — Wallet operation

직접 생성되는 주요 상태:

* durable operation identity
* idempotency key
* request fingerprint
* committed/rejected outcome
* failure snapshot과 operation indexes

#### V3 — Transactional outbox

직접 생성되는 주요 상태:

* outbox stream
* outbox event
* stream ordering와 lease/delivery fields
* operation과의 관계

V3는 V2 operation state와 연결되므로 V2보다 먼저 독립적으로 적용하는 schema로 해석할 수 없다.

#### V4 — Adjustment proof/recovery

직접 생성되는 주요 상태:

* adjustment revision identity
* original/compensating operation references
* blocked/recovery lifecycle state
* revision uniqueness와 lookup indexes

### 4. Live database verification artifact

`WalletSmokeFixture`는 Testcontainers PostgreSQL을 만들고 동적 datasource property를 application context에 연결한다. Adjustment migration test는 orphan proof와 duplicate revision이 database constraint에 의해 거부되는지 확인한다. 이는 migration SQL의 의미를 실제 PostgreSQL engine에서 검증하는 repository evidence다.

단, test fixture의 존재는 해당 test가 특정 CI에서 성공했다거나 production schema가 동일 상태라는 실행 증거가 아니다.

---

### External Development Steps

### G-01 — Database provisioning

1. PostgreSQL runtime을 선택하고 생성한다.
2. Wallet용 database 또는 namespace를 결정한다.
3. Wallet datasource identity를 생성한다.
4. Network access를 구성한다.
5. Migration DDL과 runtime DML에 필요한 권한을 부여한다.
6. URL/user/password를 secret/configuration system에 등록한다.
7. Wallet runtime에 환경변수로 주입한다.
8. Connection과 PostgreSQL-specific behavior를 확인한다.

### G-02 — Schema activation

1. Target database의 기존 object 및 migration metadata 상태를 확인한다.
2. `baseline-on-migrate:false` 조건을 충족하는지 판단한다.
3. Wallet artifact의 V1–V4 migration set을 확정한다.
4. Application startup 또는 동등한 controlled process로 migration을 실행한다.
5. V1 → V2 → V3 → V4 적용 완료를 확인한다.
6. Flyway metadata와 migration checksum을 확인한다.
7. Hibernate validation 성공을 확인한다.
8. API traffic과 scheduled worker를 허용한다.
9. 이후 release에서 새 migration을 적용할 때 동일 절차를 반복한다.
10. 실패 시 해당 환경의 restore/recreate/repair 정책에 따라 복구한다.

---

### Code Connection

| External state/step   | 직접 연결되는 source 또는 runtime behavior |
| --- | - |
| PostgreSQL endpoint   | JDBC datasource와 모든 JPA repository |
| DB identity/credential   | `WALLET_DB_URL`, `WALLET_DB_USER`, `WALLET_DB_PASSWORD` |
| DDL capability  | Startup Flyway가 V1–V4 `CREATE TABLE`, constraint, index를 실행   |
| V1 activation   | Account balance와 double-entry ledger persistence  |
| V2 activation   | Durable idempotency와 outcome replay   |
| V3 activation   | Transactional outbox recording 및 delivery leases  |
| V4 activation   | Adjustment proof, first-write decision, recovery debt   |
| Migration completion  | Hibernate `ddl-auto:validate` startup gate  |
| DB availability | API transactions, recovery worker, integrity scanner, outbox publisher |
| DB clock/locking   | PostgreSQL-specific lock와 transaction behavior를 사용하는 persistence code  |
| Schema constraint integrity | Orphan/duplicate operation·adjustment·ledger state의 DB-level rejection |

---

### Failure and Recovery Boundary

### Repository에서 직접 확인되는 실패 조건

* datasource connection 실패
* PostgreSQL driver/runtime 부재
* migration 실행 실패
* 기존 schema와 `baseline-on-migrate:false`의 비호환
* migration 결과와 JPA mapping 불일치
* DB constraint 위반
* connection/statement/lock timeout

### Repository로부터 요구됨을 확인할 수 있는 대응

* incomplete schema에서 application traffic을 허용하지 않음
* migration 성공 후 validation
* constraint test를 PostgreSQL engine에서 수행
* database credential과 timeout을 환경별로 설정

### Repository가 정의하지 않는 복구 결정

* migration rollback SQL
* Flyway repair 승인 절차
* backup restore 지점
* database point-in-time recovery
* blue/green database strategy
* production cutover 또는 rollback runbook

이 항목들은 필요할 수 있지만, repository에 근거가 없으므로 신규 Thread Packet에서 프로젝트 고유 사실로 채워 넣지 않는다.

---

### Evidence Boundary

### Directly observed in repository

* PostgreSQL/Flyway/JPA dependency
* datasource environment-variable contract
* Flyway enabled, baseline disabled
* Hibernate schema validation
* V1–V4 migration SQL
* migration 간 FK/ordering dependency
* PostgreSQL Testcontainers fixture
* migration constraint tests

### Required/inferred from repository

* live database와 datasource identity 생성
* network reachability
* migration DDL 권한
* environment-variable 주입
* migration 실행
* migration metadata/checksum 보존
* migration 완료 후 traffic 허용
* 실패 시 환경별 recovery decision

### Actual execution not observable from Git

* 실제 database instance
* 실제 URL, user, password
* 실제 role/grants
* migration 적용 시점과 실행 결과
* Flyway metadata
* production schema 상태
* schema drift
* backup, restore, repair 또는 rollback 이력

---

### Ordering

다음은 **conceptual execution order**다. Git이 증명하는 실제 chronological deployment history가 아니다.

1. PostgreSQL platform/runtime 선택
2. Database/namespace 생성
3. Wallet identity와 credential 생성
4. Network 및 권한 구성
5. Runtime secret/config 등록
6. Target schema 상태 검사
7. V1–V4 artifact 확정
8. Flyway 실행
9. Flyway metadata/checksum 확인
10. Hibernate validation
11. API readiness
12. Recovery/integrity workers 확인
13. Kafka 준비 후 outbox publisher 활성화
14. 이후 release의 incremental migration lifecycle 반복

---

### New Thread Source Packet의 비범위

이 Packet은 다음을 다시 설명하지 않는다.

* Account balance 계산 규칙
* Double-entry ledger의 도메인 topology
* Idempotency fingerprint 또는 replay 의미
* Adjustment decision semantics
* Outbox lease/retry 알고리즘
* Integrity scanner의 각 drift query

그 내용은 기존 Thread가 계속 소유한다. `NT-DB-01`은 해당 모델들이 **실제 PostgreSQL 외부 상태로 생성되고 버전 관리되는 과정**만 소유한다.

---

# Part IV — Project-Level External Steps

## PL-01 / G-07 — Monitoring target·probe·alert 외부 등록

### 관련 Existing Threads

* Thread 3 — build/runtime observability dependency
* Thread 11 — recovery worker 상태
* Thread 13 — outbox delivery backlog/retry
* Thread 14 — durable state integrity scan

### Repository Evidence

* Actuator health/info/prometheus/metrics exposure

* Prometheus/Micrometer build dependency

* outbox pending, leased, retry, oldest-pending metrics

* integrity drift, scan-failure, last-checked metrics

* integrity health `UP`, `DOWN`, `UNKNOWN` behavior

* operations 문서의 "outbox 활성화 전 alerting 확인" 요구

### Required External Step

1. 실제 wallet deployment endpoint를 monitoring inventory에 등록한다.
2. health probe와 Prometheus scrape를 구성한다.
3. outbox와 integrity failure signal의 alert rule을 만든다.
4. alert receiver와 routing을 등록한다.
5. notification test를 수행한다.
6. outbox production activation의 사전 조건으로 기록한다.

### Code Connection

Monitoring system이 없더라도 application은 metric을 생성할 수 있다. 그러나 외부 scrape가 없으면 signal은 수집되지 않으며, alert receiver가 없으면 failure가 운영자에게 전달되지 않는다. 즉, source instrumentation과 operational observability는 별도의 상태다.

### Evidence Boundary

**Directly observed:** metric/health 생성 코드와 endpoint exposure
**Required/inferred:** 외부 scrape, probe, alerting, notification route
**Not observable:** 실제 monitoring product, target, threshold, receiver, dashboard, notification 결과

### Documentation Action

Development Thread를 추가하지 않고 project-level deployment/operations readiness 문서에 한 번만 기록한다.

---

## 채택하지 않은 External-State 후보

### 1. Redis의 필수 프로비저닝 — 채택하지 않음

Redis는 `IdempotencyCache`의 24시간 best-effort marker에 사용되지만, lookup/write failure를 잡아 PostgreSQL authoritative state로 fallback한다. 운영 문서도 Redis를 optional/best-effort로 구분한다. 따라서 Redis가 없으면 시스템 자체가 성립하지 않는다고 볼 수 없으며, mandatory External-State Gap으로 채택하지 않았다.

Redis를 실제로 사용하려는 환경에서는 endpoint availability가 필요하지만, 이는 optional acceleration의 활성화이며 이번 감사의 "필수 외부 성립 단계"에는 포함하지 않는다.

### 2. HOUSE·EXTERNAL_PAYMENT account seed — 채택하지 않음

두 ID는 ledger counterparties로 정의된 고정 UUID다. V1은 이 UUID들이 사용자 `account` row로 생성되는 것을 오히려 금지한다. Repository에는 이들을 insert하는 seed도 없다. 따라서 system account row 생성 또는 seed 실행을 요구하면 repository evidence와 반대되는 설명이 된다.

### 3. 외부 cron/scheduler 등록 — 채택하지 않음

Recovery, integrity, outbox worker는 외부 cron service가 아니라 Spring의 in-process `@Scheduled`로 실행된다. Shared scheduler pool도 application configuration으로 생성된다. 따라서 cloud scheduler나 OS cron 등록을 요구하는 근거가 없다.

Outbox의 enable flag는 G-04에 포함했고, recovery/integrity는 기본 활성 상태이므로 별도 외부 등록 Gap으로 분리하지 않았다.

### 4. OAuth application, redirect URI, webhook registration — 채택하지 않음

Repository의 인증 방식은 internal API key header이고, complete tree에서 OAuth client/provider 또는 webhook endpoint registration artifact가 확인되지 않는다. 이를 일반적인 서비스 운영 관행만으로 추가하지 않았다.

### 5. Object storage, bucket, IAM — 채택하지 않음

Source와 configuration에서 object storage client, bucket name 또는 cloud IAM binding 요구를 확인할 수 없다. 따라서 관련 resource 생성 단계는 없다.

### 6. DNS, domain verification, TLS certificate — 채택하지 않음

Repository에 ingress, domain, certificate, reverse proxy 또는 deployment platform configuration이 없다. 실제 배포에서 필요할 수 있으나 이 repository만으로 구체적 필요성을 확정할 수 없으므로 Gap으로 만들지 않았다.

### 7. Database backup/restore 실행 — 채택하지 않음

Repository에는 backup script, restore script, snapshot policy 또는 recovery command가 없다. Database가 금전 상태를 보유한다는 이유만으로 특정 backup/restore 작업이 실제 수행되었거나 repository가 구체적으로 요구한다고 서술하지 않았다.

`NT-DB-01`에서는 migration 실패 시 recovery policy가 외부에서 결정되어야 한다는 boundary만 남기고, 구체적 backup/restore 절차는 프로젝트 사실로 추가하지 않았다.

### 8. Kafka SASL/TLS/ACL과 topic topology 값 — 채택하지 않음

Repository가 확인하는 것은 bootstrap endpoint, producer delivery configuration, exact topic 이름, key/header contract까지다. 다음 값은 확인되지 않는다.

* SASL mechanism
* TLS certificate
* principal 또는 ACL
* partition count
* replication factor
* retention
* min ISR

따라서 G-04에서는 "destination availability와 consumer readiness"만 요구하고 구체 값을 추정하지 않았다.

---

# 최종 Documentation Plan

기존 Thread 체계에는 다음 최소 보완만 필요하다.

| Action | 대상  | 포함 Gap  |
| --- | --- | - |
| Existing Thread Supplement  | Thread 3 — 재현 가능한 빌드와 의미 검증 | G-05, G-06 |
| Existing Thread Supplement  | Thread 13 — 리스 기반 FIFO 아웃박스 전달 | G-04 |
| Existing Thread Supplement  | Thread 15 — 내부 서비스 인증과 권한 경계   | G-03 |
| Proposed New Thread   | `NT-DB-01` — PostgreSQL 프로비저닝과 Flyway 스키마 활성화 | G-01, G-02 |
| Project-Level Operations Appendix | Monitoring target·probe·alert 등록  | G-07 |

이 구성은 기존 17개 Thread의 도메인 관점을 변경하지 않는다. 추가되는 것은 Git에 남은 source/configuration을 실제 시스템 상태로 성립시키기 위해 필요한 **database activation, secret distribution, Kafka enablement, build/runtime materialization, deployment, monitoring registration**의 증거 경계뿐이다.
