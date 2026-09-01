# External-State Development Gap Audit

분석 기준은 첨부된 **Existing Development Thread Index**를 고정된 1차 Thread 체계로 사용했습니다.  GitHub 저장소는 전체 commit history를 페이지 끝까지 순회했고, 현재 `main` tree도 recursive 조회 결과 `truncated: false`인 전체 파일 목록을 기준으로 확인했습니다. 현재 tree에는 CI workflow, migration, source/test/config는 존재하지만 Docker/Compose/Kubernetes/Terraform 등 production deployment manifest는 없습니다.

감사 결과는 **EXISTING_THREAD 7개 Gap / NEW_THREAD 0개 / PROJECT_LEVEL_EXTERNAL_STEP 1개 Gap**입니다. 외부 상태라는 이유만으로 별도 "Environment/Operations" Thread를 만들 근거는 발견되지 않았습니다.

---

# Part I — Gap Index

| Gap ID  | 짧은 이름  | Classification  | Primary Owner | Related Threads  |
| --- | --- | --- | --- | --- |
| `GAP-ES-01` | Build/Verification Environment Bootstrap | `EXISTING_THREAD` | Thread 1  | Thread 2 |
| `GAP-ES-02` | PostgreSQL Runtime Database Provisioning | `EXISTING_THREAD` | Thread 13 | Threads 10, 11, 12, 14 |
| `GAP-ES-03` | Flyway Migration Execution | `EXISTING_THREAD` | Thread 10 | Threads 11, 12, 13, 14 |
| `GAP-ES-04` | Kafka Broker and Audit Topic Availability  | `EXISTING_THREAD` | Thread 13 | —  |
| `GAP-ES-05` | JWT Trust Material and External Signer Alignment | `EXISTING_THREAD` | Thread 4  | —  |
| `GAP-ES-06` | Trusted Proxy and Operator Network Configuration | `EXISTING_THREAD` | Thread 3  | Thread 4 |
| `GAP-ES-07` | Downstream Origins and Caller Credentials  | `EXISTING_THREAD` | Thread 6  | Threads 7, 8, 16–20  |
| `GAP-ES-08` | Release Runtime Provisioning and Deployment  | `PROJECT_LEVEL_EXTERNAL_STEP` | Project level | Thread 2 |

## GAP-ES-01 — Build/Verification Environment Bootstrap

**Repository Evidence 요약:** 프로젝트는 Java 17 baseline과 released `shared-protocol` 의존성을 전제로 하며, 실제 테스트 스택에는 PostgreSQL/Kafka Testcontainers가 들어 있습니다. `SharedProtocolDependencyTest`는 `com.sportsbook.protocol`의 released contract를 실제로 resolve해야 합니다.  `pom.xml`에는 Testcontainers JUnit/PostgreSQL/Kafka test dependencies가 존재하고, 별도 commit은 Testcontainers archive runtime dependency까지 보정했습니다.

**Required External Step 요약:** 검증을 실제로 수행하려면 Java 17과 container 실행 기능이 있는 host/runner를 제공하고, 요구되는 `shared-protocol` artifact/source를 Maven resolution이 가능한 상태로 만든 뒤 `clean verify`를 수행해야 합니다. Testcontainers 기반 테스트는 실행 중 PostgreSQL/Kafka transient resource를 실제로 생성합니다.

**실제 수행 여부 확인 가능성:** Git에는 CI workflow와 dependency contract만 있습니다. 특정 workflow run이 실제로 성공했는지, container가 실제 생성·정리되었는지는 commit만으로 증명되지 않습니다.

**Documentation Action:** Thread 1에 "재현 가능한 품질 게이트를 성립시키는 external build/test environment bootstrap"을 보충합니다. 새로운 CI/Operations Thread는 만들지 않습니다.

---

## GAP-ES-02 — PostgreSQL Runtime Database Provisioning

**Repository Evidence 요약:** `application.yml` 도입 commit은 datasource를 `ADMIN_DB_URL`, `ADMIN_DB_USER`, `ADMIN_DB_PASSWORD`에 연결하며 Hibernate를 `ddl-auto: validate`로 설정했습니다. 동시에 Flyway가 활성화됩니다.  Readiness도 이후 DB를 포함하도록 명시적으로 구성되었습니다.

**Required External Step 요약:** 실제 실행환경에는 연결 가능한 PostgreSQL database와 application datasource principal이 존재해야 하며, 연결정보를 runtime에 주입해야 합니다. 동일 datasource가 startup Flyway에도 사용되므로 migration에 필요한 schema 변경 권한과 이후 audit DML에 필요한 권한을 충족해야 합니다.

**실제 수행 여부 확인 가능성:** 실제 DB instance, endpoint, account, password, grant, 생성 시점은 repository에서 확인할 수 없습니다.

**Documentation Action:** Thread 13에 "권위 audit DB가 코드 밖에서 먼저 실제 resource로 존재해야 한다"는 단계를 추가합니다. Migration 자체의 실행은 GAP-ES-03/Thread 10에만 소유시킵니다.

---

## GAP-ES-03 — Flyway Migration Execution

**Repository Evidence 요약:** V1은 `audit_log` table과 조회 index를 실제로 생성하는 SQL입니다.  V2는 단순 신규 column 추가가 아니라 기존 audit data의 상태를 변경하는 migration입니다. `occurred_at → started_at` rename, `completed_at` 추가와 backfill, lifecycle CHECK constraints, index rename, stale-STARTED partial index 등이 포함됩니다. 또한 runtime configuration은 Flyway를 활성화하고 Hibernate에는 schema 생성 대신 `validate`만 맡깁니다.

**Required External Step 요약:** 준비된 target PostgreSQL에 repository의 Flyway migration set을 실제로 적용해야 합니다. Clean database에서는 V1→V2가 적용되어야 하며, V1 상태 database의 upgrade라면 V2가 기존 row와 schema를 실제로 변환해야 합니다.

**실제 수행 여부 확인 가능성:** 어떤 DB에 언제 migration이 실행되었는지, 실제 `flyway_schema_history`, 변경된 row 수, migration 성공/실패 또는 수동 복구는 Git에서 확인되지 않습니다.

**Documentation Action:** Thread 10에 migration 파일 작성 이후의 **실제 database state transition**을 별도 external step으로 보충합니다.

---

## GAP-ES-04 — Kafka Broker and Audit Topic Availability

**Repository Evidence 요약:** runtime configuration은 `ADMIN_KAFKA_BOOTSTRAP`을 사용합니다.  Kafka producer는 idempotent producer configuration을 가지고 audit event를 외부 broker로 publish하도록 구현되어 있습니다. 현재 전체 tree에는 Kafka broker/topic provisioning용 infrastructure artifact가 없습니다.

**Required External Step 요약:** 실제 환경에는 admin-api가 접근 가능한 Kafka bootstrap endpoint가 있어야 하며, configured audit topic이 publish 가능한 상태여야 합니다. Topic이 사전 생성되는지 broker auto-creation에 의해 생기는지는 이 repository가 결정하지 않습니다.

**실제 수행 여부 확인 가능성:** 실제 Kafka cluster, broker endpoint, topic existence, partition/replication configuration, ACL 및 event publish 이력은 Git으로 확인할 수 없습니다.

**Documentation Action:** Thread 13에 DB→Kafka best-effort projection을 현실 환경에서 완성하는 broker/topic availability 단계를 보충합니다. 별도 Kafka Operations Thread는 만들지 않습니다.

---

## GAP-ES-05 — JWT Trust Material and External Signer Alignment

**Repository Evidence 요약:** JWT configuration은 `ADMIN_JWT_PUBLIC_KEY`를 필수값으로 만들고 optional issuer를 받습니다.  최종 decoder는 이 공개키를 RSA verification key로 사용하는 구조입니다. 또한 repository는 `*.pem`, `*.p8`, `*-private.key`를 명시적으로 Git에서 제외합니다.

**Required External Step 요약:** 유효한 admin JWT를 발행하는 외부 signing side가 존재해야 하며, 그 private signing key에 대응하는 public key를 admin-api runtime에 공급해야 합니다. issuer validation을 사용하는 환경이라면 signer가 생성하는 issuer와 `ADMIN_JWT_ISSUER`도 일치해야 합니다.

**실제 수행 여부 확인 가능성:** private key 생성, actual signer/identity provider, public-key 배포 시점, issuer 값, token 발급 이력은 확인할 수 없습니다.

**Documentation Action:** Thread 4에 **verifier-side code와 external signer/trust-material 상태의 결합**을 추가합니다. Repository에 근거가 없으므로 OAuth application, JWKS endpoint, redirect URI 등록 등이 존재한다고 확대하지 않습니다.

---

## GAP-ES-06 — Trusted Proxy and Operator Network Configuration

**Repository Evidence 요약:** `TrustedProxyResolver`는 remote peer가 trusted proxy일 때만 `X-Forwarded-For`를 신뢰하고, hop을 역방향으로 평가합니다.  `/admin/**`에는 별도 IP allowlist filter가 적용되어 resolved client가 허용 CIDR에 속하지 않으면 요청을 거부합니다.

**Required External Step 요약:** 실제 배포 topology를 기준으로 admin 접근이 허용되는 operator/client CIDR을 결정하고, proxy를 사용하는 경우 신뢰 가능한 direct proxy CIDR도 결정하여 runtime 설정에 주입해야 합니다. `X-Forwarded-For`를 사용하는 실제 ingress/proxy는 코드의 trust model과 일치하는 형태로 header를 전달해야 합니다.

**실제 수행 여부 확인 가능성:** 실제 proxy, load balancer, client network, CIDR 값, ingress topology, firewall state는 repository에서 확인할 수 없습니다.

**Documentation Action:** Thread 3에 code-level CIDR logic과 **actual network topology registration/configuration** 사이의 외부 단계를 추가합니다.

---

## GAP-ES-07 — Downstream Origins and Caller Credentials

**Repository Evidence 요약:** 네 downstream credential은 모두 필수이고 최소 32자이며 서로 중복될 수 없습니다.  테스트도 missing/short/reused credential을 거부하도록 검증합니다.  Wallet/Risk/Odds/Settlement 각각의 base URL은 absolute HTTP(S) origin이어야 하며 별도 runtime variable로 정의됩니다.  각각 전용 RestClient가 credential을 해당 downstream 요청 header에 넣어 전송합니다.

**Required External Step 요약:** 네 downstream service 각각에 대해 실제 service origin을 결정해야 하고, admin-api가 caller로 인증될 수 있는 각각의 distinct credential을 downstream 측에서 발급·등록 또는 동등한 방식으로 준비한 뒤 admin-api runtime에 대응 URL/key를 주입해야 합니다.

**실제 수행 여부 확인 가능성:** 실제 credential value, 발급 방식, downstream-side 등록 상태, rotation history, 실제 production hostname은 Git에 없습니다.

**Documentation Action:** Thread 6에 credential isolation을 실제 서비스 간 trust state로 완성하는 단계를 추가합니다. Wallet/Risk/Market/Settlement 기능 Thread들은 consumer일 뿐 credential lifecycle의 공동 Primary Owner로 만들지 않습니다.

---

## GAP-ES-08 — Release Runtime Provisioning and Deployment

**Repository Evidence 요약:** release commit은 Maven artifact version을 `0.1.0-SNAPSHOT`에서 `1.0.0`으로 고정하여 JAR release boundary를 만듭니다.  그러나 전체 repository tree에는 deployment provider/orchestrator manifest가 없으며 현재 CI도 verification-oriented source입니다.

**Required External Step 요약:** release artifact가 실제로 서비스가 되려면 Java 17 runtime 또는 동등한 application execution environment를 마련하고, 앞선 DB/Kafka/security/downstream runtime inputs를 공급한 뒤 application process를 시작하고 configured HTTP port를 실제 environment routing에 연결해야 합니다.

**실제 수행 여부 확인 가능성:** staging/production environment, host, process manager, deployment time, rollout/rollback, public routing 등은 확인할 수 없습니다.

**Documentation Action:** 특정 Thread의 소유 단계로 과도하게 확장하지 않고 **Project-Level External Development Step**으로 기록합니다. Thread 2는 release boundary를 제공하지만 production deployment implementation 자체의 Thread라고 볼 repository 근거가 없습니다.

---

# Part II — Existing Thread Supplement Packets

## Packet E01 — Thread 1

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 1
* **한국어 제목:** 재현 가능한 Java 서비스 기반과 품질 게이트
* **English title:** Reproducible Java Service Baseline and Quality Gates
* **Gaps:** `GAP-ES-01`

### Repository Evidence

| Commit  | Subject | 관련 파일 | 의미 |
| --- | --- | --- | --- |
| `0a2672ec195d...` | `build(maven): establish Java 17 baseline`  | `pom.xml` | Java runtime/compiler baseline |
| `98c178a7bbb6...` | `test(maven): resolve shared protocol 1.0.0`  | `SharedProtocolDependencyTest.java` | 외부 Maven dependency가 실제 resolve되어야 함 |
| `f9438af1149b...` | `build(test): align Testcontainers archive runtime` | `pom.xml` | container-based test runtime 보강  |

`98c178...`은 released Money/Currency contract를 직접 import합니다.  `pom.xml`의 final state에는 PostgreSQL/Kafka Testcontainers가 포함됩니다.

### External Development Steps

**Conceptual execution order:**

1. Java 17을 사용할 수 있는 build runner/host를 준비한다.
2. container 실행이 가능한 runtime을 제공한다.
3. required `shared-protocol` release/source를 Maven resolution 가능한 상태로 준비한다.
4. admin-api의 verification command를 실행한다.
5. integration test가 transient PostgreSQL/Kafka resource를 생성하도록 허용한다.
6. 검증 종료 후 transient resource를 정리한다.

### Code Connection

* shared protocol이 resolution되지 않으면 application test compilation 자체가 성립하지 않습니다.
* PostgreSQL/Kafka integration test는 source만으로 실행되지 않고 container runtime state를 요구합니다.
* 이 단계는 Thread 1의 "재현 가능한 quality gate"를 실제 실행 상태로 완성합니다.

### Evidence Boundary

**Directly observed in repository**

* Java baseline
* external shared-protocol dependency test
* Testcontainers dependencies와 runtime alignment

**Required/inferred from repository**

* Java 17 runtime
* usable Maven dependency resolution
* container-capable test environment

**Actual execution not observable from Git**

* 특정 CI run
* image pull
* transient PostgreSQL/Kafka instance
* test execution outcome와 cleanup 여부

---

## Packet E03 — Thread 3

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 3
* **한국어 제목:** 신뢰 프록시 해석과 IP 허용 목록
* **English title:** Trusted Proxy Resolution and IP Allowlisting
* **Gaps:** `GAP-ES-06`

### Repository Evidence

* `06c95ea1215c...` — `feat(security): resolve trusted client addresses`

  * `TrustedProxyResolver.java`
  * trusted peer일 때만 XFF chain을 해석합니다.
* `76953723b621...` — `feat(security): enforce the admin IP allowlist`

  * `IpAllowlistFilter.java`, `SecurityConfig.java`
  * `/admin/**`를 network policy로 차단합니다.

### External Development Steps

**Conceptual execution order:**

1. 실제 admin client/operator network를 식별한다.
2. 허용해야 할 CIDR을 결정한다.
3. proxy가 존재한다면 admin-api와 직접 연결되는 trusted proxy CIDR을 결정한다.
4. 두 runtime configuration을 주입한다.
5. XFF가 실제 사용된다면 proxy forwarding behavior를 코드의 trust model과 정렬한다.
6. 실제 ingress 경로를 통해 resolved address와 allow/deny 결과를 확인한다.

### Code Connection

잘못된 외부 CIDR 또는 proxy topology는 소스 변경이 없어도 legitimate admin request 거부 또는 신뢰하면 안 되는 forwarded address의 채택으로 이어질 수 있습니다.

### Evidence Boundary

* **Directly observed:** CIDR parsing, trusted-proxy traversal, allowlist filter.
* **Required/inferred:** deployment-specific CIDR/topology selection.
* **Actual execution not observable:** proxy 종류, 실제 CIDR, routing/firewall configuration, 적용 이력.

---

## Packet E04 — Thread 4

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 4
* **한국어 제목:** JWT 운영자 신뢰와 역할 기반 접근 경계
* **English title:** JWT Operator Trust and Role-Based Access Boundary
* **Gaps:** `GAP-ES-05`

### Repository Evidence

* `dc8684c07a11...` — `feat(security): validate JWT settings`

  * `AdminJwtProperties.java`, `application.yml`
  * public key가 mandatory runtime trust input입니다.
* `2a83687fa99d...` — `chore(repo): ignore generated and secret files`

  * private-key 계열 file을 Git에서 제외합니다.

### External Development Steps

**Conceptual execution order:**

1. admin JWT를 서명할 외부 signing authority/key material을 마련한다.
2. 해당 private key에 대응하는 public verification key를 admin-api에 공급한다.
3. issuer 검증을 사용할 경우 발급자와 configured issuer를 정렬한다.
4. code가 요구하는 subject/role/time claim을 갖는 실제 token을 발급한다.
5. admin-api에서 그 token의 trust boundary를 검증한다.

### Code Connection

JWT verification code는 signer를 생성하지 않습니다. 따라서 public-key verifier와 실제 token signer 사이의 외부 신뢰관계가 형성되어야 Thread 4의 authentication boundary가 실제 환경에서 성립합니다.

### Evidence Boundary

* **Directly observed:** public verification key와 issuer configuration.
* **Required/inferred:** matching signer/signing key.
* **Actual execution not observable:** private key, 생성·회전, signer/provider, token issuance.
* **Not inferred:** OAuth application, redirect URI, JWKS service.

---

## Packet E06 — Thread 6

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 6
* **한국어 제목:** 하위 서비스 자격 증명과 RestClient 격리
* **English title:** Downstream Credential and RestClient Isolation
* **Gaps:** `GAP-ES-07`

### Repository Evidence

* `e81bdc7b5f7a...` — 네 credential의 mandatory/distinct/minimum-length contract.
* `79830c14b06c...` — missing/short/reused key rejection test.
* `ad5e3a24ca6b...` — 네 HTTP(S) origin의 runtime contract.
* final `DownstreamClientConfiguration`은 각각 다른 client에 해당 API key를 붙입니다.

### External Development Steps

**Conceptual execution order:**

1. Wallet/Risk/Odds/Settlement의 실제 target origin을 결정한다.
2. 각 downstream에서 admin-api caller credential을 독립적으로 준비한다.
3. 네 credential이 서로 재사용되지 않도록 한다.
4. 각 origin/key를 admin-api runtime에 주입한다.
5. 각 downstream에서 해당 caller/header contract가 실제 수락되는지 확인한다.

### Code Connection

Credential isolation은 admin-api 내부 객체 분리만으로 끝나지 않습니다. 상대 서비스가 같은 credential을 caller identity로 인식해야 실제 cross-service authentication이 성립합니다.

### Evidence Boundary

* **Directly observed:** key shape/isolation, URL validation, HTTP header insertion.
* **Required/inferred:** downstream-side matching credential state.
* **Actual execution not observable:** 값, 발급/등록 방법, rotation, actual production origins.

---

## Packet E10 — Thread 10

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 10
* **한국어 제목:** 감사 스키마의 호환 진화와 데이터베이스 불변 조건
* **English title:** Compatible Audit Schema Evolution and Database Invariants
* **Gaps:** `GAP-ES-03`

### Repository Evidence

`7e9a046f...`의 V1은 실제 table/index를 생성합니다.  이후 V2는 기존 audit schema와 row를 lifecycle model로 변환합니다. Runtime에서는 Flyway가 활성화되며 Hibernate는 `validate`만 수행합니다.

### External Development Steps

**Conceptual execution order:**

1. `GAP-ES-02`의 PostgreSQL target을 준비한다.
2. target schema/Flyway history가 expected migration lineage와 호환되는지 확인한다.
3. application startup의 Flyway lifecycle을 target database에 실제 실행한다.
4. V1 및 필요한 후속 migration을 적용한다.
5. V1→V2 upgrade에서는 기존 row backfill 및 constraints/index transition까지 완료한다.
6. 이후 Hibernate schema validation이 통과하는 상태가 되어야 한다.

### Code Connection

Migration SQL이 Git에 있다는 사실과 database에 SQL 효과가 존재한다는 사실은 다릅니다. Runtime repository/entity는 migration 결과의 실제 schema를 전제로 동작합니다.

### Evidence Boundary

* **Directly observed:** migration SQL, Flyway enablement, Hibernate validate.
* **Required/inferred:** target DB에서 migration의 실제 적용.
* **Actual execution not observable:** 적용 DB, 시간, Flyway history, affected row count, failure/rollback.

---

## Packet E13 — Thread 13

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 13
* **한국어 제목:** 권위 감사 DB와 best-effort Kafka 투영
* **English title:** Authoritative Audit Database and Best-Effort Kafka Projection
* **Gaps:** `GAP-ES-02`, `GAP-ES-04`

### Repository Evidence

`2655b69d...`은 PostgreSQL datasource와 Kafka bootstrap을 runtime configuration으로 도입했습니다.  DB는 readiness에 포함됩니다.  현재 전체 tree에는 migration은 존재하지만 PostgreSQL resource provisioning이나 Kafka topic creation용 infrastructure source는 없습니다.

### External Development Steps

**Conceptual execution order:**

1. PostgreSQL database와 datasource principal을 마련한다.
2. 필요한 migration/DML 권한을 부여한다.
3. DB endpoint/user/password를 runtime에 주입한다.
4. `GAP-ES-03`에서 schema migration을 수행한다.
5. 별도로 reachable Kafka cluster/bootstrap endpoint를 마련한다.
6. configured audit topic이 producer에게 usable한 상태가 되도록 한다.
7. Kafka bootstrap/topic runtime configuration을 적용한다.

DB와 Kafka 준비는 서로 독립적으로 진행될 수 있습니다. DB는 readiness와 authoritative state에 연결되지만 Kafka projection은 best-effort이므로 둘의 운영 중요도 역시 동일하지 않습니다.

### Code Connection

* DB 없음 → readiness와 authoritative audit persistence가 성립하지 않음.
* Kafka 없음 → authoritative DB row와 별개로 best-effort projection이 실패할 수 있음.
* Topic provisioning 방식은 repository가 선택하지 않음.

### Evidence Boundary

**Directly observed**

* datasource binding
* Flyway
* DB-based readiness
* Kafka producer/bootstrap configuration

**Required/inferred**

* actual PostgreSQL resource
* datasource principal/permissions
* reachable Kafka cluster
* usable audit topic

**Actual execution not observable**

* DB/Kafka addresses
* credentials
* DB grants
* Kafka ACL
* topic creation method
* actual runtime availability

---

# Part III — Proposed New Thread Packets

## 제안 없음

이번 감사에서는 **`NEW_THREAD`를 제안하지 않습니다.**

이유는 각 external-state 관점이 이미 명확한 기존 소유자를 갖기 때문입니다.

* build/test external state → Thread 1
* network trust configuration → Thread 3
* external JWT signing trust → Thread 4
* downstream credential/origin state → Thread 6
* migration execution → Thread 10
* PostgreSQL/Kafka runtime resources → Thread 13

반면 actual deployment는 새로운 Thread로 만들 만큼 repository-backed implementation이 없습니다. 현재 저장소에는 provider-specific provisioning, environment promotion, rollout/rollback, registry, service orchestration, ingress 등의 commit 집합을 선정할 수 없습니다. 따라서 `NEW_THREAD`의 독립 lifecycle + representative commits 조건을 만족하지 않습니다.

---

# Part IV — Project-Level External Steps

## GAP-ES-08 — Release Runtime Provisioning and Deployment

**Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`

### Repository Evidence

`7d09112a...`는 `admin-api`를 `1.0.0` release JAR 경계로 전환합니다.  그러나 current complete tree에는 durable deployment/provisioning artifact가 없습니다.

### Required External Development Steps

**Conceptual execution order:**

1. target environment에 필요한 DB/Kafka/JWT/network/downstream external state를 앞선 owner Gap들에 따라 준비한다.
2. 검증된 `admin-api-1.0.0` artifact를 준비한다.
3. Java 17 application runtime을 마련한다.
4. 각 owner Gap의 runtime configuration을 target process에 주입한다.
5. application을 configured HTTP port로 실제 기동한다.
6. 선택된 environment의 routing/process supervision에 연결한다.
7. readiness를 확인한 뒤 service traffic을 허용한다.

### Evidence Boundary

**Directly observed in repository**

* executable Java service
* `1.0.0` release artifact boundary
* HTTP/runtime configuration
* readiness endpoint configuration

**Required/inferred from repository**

* artifact를 실제 process로 실행할 target runtime
* runtime configuration injection
* service port에 대한 실제 reachability

**Actual execution not observable from Git**

* production/staging 존재 여부
* deploy host
* process/container/orchestrator 종류
* release deployment 시점
* ingress/load balancer
* rollout/rollback
* restart policy
* public domain

따라서 Kubernetes, Docker deployment, VM, cloud service, DNS, TLS 등의 특정 구현을 이 Gap의 사실로 추가해서는 안 됩니다.

---

## 채택하지 않은 External-State 후보

Repository 근거를 기준으로 다음 항목은 Gap으로 만들지 않았습니다.

* **OAuth application/provider 및 redirect URI 등록:** 관련 integration source가 없음.
* **Webhook 외부 등록:** webhook registration/signature/config 근거 없음.
* **Object storage/bucket/IAM:** 관련 SDK/resource configuration 없음.
* **DNS/domain verification/TLS certificate:** 해당 external resource를 요구하는 deployment source 없음.
* **외부 cron/scheduler 등록:** stale audit recovery는 외부 cron이 아니라 Spring `@Scheduled`로 process 내부에서 수행됩니다.
* **Seed 실행:** repository에 production/dev seed workflow가 확인되지 않음.
* **Backup/restore:** 관련 script 또는 lifecycle source 없음.
* **특정 secret manager / GitHub Actions Secret 구성:** secrets가 외부에 있어야 한다는 것은 확인되지만 어느 platform을 사용했는지는 확인할 수 없음.
* **Prometheus collector 등록:** endpoint 노출은 있지만 외부 collector 계정/resource 구성을 요구한다고 볼 repository 근거가 부족함.

## 최종 판정

이 프로젝트의 External-State Gap은 새로운 개발 관점을 추가하기보다, **이미 존재하는 Thread들이 실제 시스템 상태와 만나는 마지막 구간을 보완하는 성격**이 강합니다. 특히 핵심은 `DB resource → migration state → authoritative audit`, `external signer → JWT verifier`, `real proxy/network → CIDR trust`, `downstream-side credential → isolated RestClient`, `Kafka runtime → best-effort projection`의 다섯 경계입니다.

따라서 후속 문서 작업에서는 **Thread 1, 3, 4, 6, 10, 13만 supplement**하고, 실제 deployment는 한 개의 project-level external step으로 유지하는 것이 현재 증거에 가장 보수적으로 부합합니다.
