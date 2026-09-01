# External-State Development Gap Audit

**Repository:** `seungwoo7050/tmp-sportsbook-betting-service`
**Existing Development Threads:** 17개 

## 감사 범위와 판정 요약

Git commit history를 100개 단위로 끝까지 조회했으며, 네 번째 페이지가 빈 배열임을 확인한 뒤 현재 repository tree, migration, runtime configuration, CI workflow, integration-test support, 운영 문서를 함께 대조했습니다.

기존 Thread 문서는 사용하지 않았으며, 첨부된 문서에서는 **확정된 Thread 제목·번호·commit 구성만** 사용했습니다.

최종 판정은 다음과 같습니다.

* `EXISTING_THREAD`: 4개
* `NEW_THREAD`: 3개
* `PROJECT_LEVEL_EXTERNAL_STEP`: 2개
* 총 Gap: **9개**

제안하는 신규 Thread는 다음 세 가지입니다.

1. **NT-18 — 데이터베이스 스키마 배포와 복구 준비**
2. **NT-19 — 방향별 서비스 자격증명과 런타임 바인딩**
3. **NT-20 — 고정 프로토콜 아티팩트 부트스트랩과 재현 가능한 검증**

---

# Part I — Gap Index

## GAP-ES-01 — Redis 유효 시세 스냅샷 공급

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 3 — 실패 폐쇄형 베팅 접수 검증
* **Related Threads:** Thread 8 — 체크포인트 기반 베팅 접수 사가와 보상
* **Repository Evidence 요약:**

  * `OddsSnapshotReader`는 다음 Redis 값을 직접 읽습니다.

    * `market:{eventId}:{marketId}` → 정확히 `OPEN`
    * `odds:{eventId}:{marketId}:{selectionId}` → 파싱 가능한 decimal
  * market 값이 없거나 `OPEN`이 아니고, odds 값이 없거나 숫자가 아니면 접수를 폐쇄합니다.
  * canonical key 형식은 별도의 테스트로 고정되어 있습니다.
  * repository architecture는 이 Redis 상태의 writer를 Betting Service가 아니라 **Odds Feed**로 지정합니다.
* **Required External Step 요약:**

  * Betting Service가 접근할 Redis runtime을 준비합니다.
  * 대상 Redis 주소를 `BETTING_REDIS_HOST`, `BETTING_REDIS_PORT`에 바인딩합니다.
  * 외부 Odds Feed가 위 key 형식으로 market 상태와 selection odds를 실제로 기록해야 합니다.
  * 접수 traffic을 활성화하기 전에 필요한 market/odds 값이 존재해야 합니다.
* **실제 수행 여부 확인 가능성:**
  Redis writer 구현, 실제 Redis instance, 기록 시점, 값, 갱신 주기, TTL은 repository에서 확인되지 않습니다.
* **Documentation Action:**
  Thread 3에 External-State 보충 문서를 추가합니다. Redis TTL, authentication, persistence mode 등 repository가 명시하지 않은 사항은 추가하지 않습니다.

---

## GAP-ES-02 — Kafka publication target과 `bet.placed.v1` 준비

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 9 — 트랜잭셔널 아웃박스와 승인 기반 최소 한 번 전달
* **Related Threads:** Thread 2, Thread 8, Thread 12
* **Repository Evidence 요약:**

  * Outbox publisher는 DB의 미발행 row를 읽어 `event.topic()`과 `event.partitionKey()`로 Kafka record를 생성하고 broker acknowledgement를 기다린 뒤에만 `published_at`을 기록합니다.
  * accepted-bet publication topic은 `bet.placed.v1`입니다.
  * Kafka bootstrap address는 `BETTING_KAFKA_BOOTSTRAP`으로 바인딩됩니다.
  * 운영 구성은 `bet.placed.v1`을 consumer 활성화 전 사전 생성할 topic 목록에 포함합니다.
* **Required External Step 요약:**

  * 접근 가능한 Kafka cluster 또는 broker endpoint가 존재해야 합니다.
  * `bet.placed.v1` topic을 사전 생성해야 합니다.
  * target environment의 bootstrap address를 Betting Service에 주입해야 합니다.
  * publisher 활성화 전에 topic metadata 조회와 broker acknowledgement 경로를 검증해야 합니다.
* **실제 수행 여부 확인 가능성:**
  실제 broker, topic 생성 이력, partition 수, replication factor, retention, 실제 publication 실행 여부는 repository에서 확인되지 않습니다.
* **Documentation Action:**
  Thread 9에 "아웃박스 row 생성 이후 실제 Kafka publication이 성립하기 위한 외부 조건"을 보충합니다.

---

## GAP-ES-03 — Kafka consumed-topic/DLT topology 사전 구성

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 12 — Kafka 영구 오류 분류와 승인 기반 DLT 복구
* **Related Threads:** Thread 10, Thread 15, Thread 16
* **Repository Evidence 요약:**

  * commit `931a0b3134efd813453511c925211dbf76f3fbc6`는 consumer의 `allow.auto.create.topics`를 `false`로 변경합니다.
  * commit `2c3d5abfaf456bfb17d0ee1326c067052f719404`는 해당 값이 계속 `false`인지 검증합니다.
  * DLT recoverer는 source record의 partition을 그대로 사용해 `record.topic() + ".DLT"`에 전송하며, send failure를 성공으로 처리하지 않습니다.
  * repository가 사용하는 Kafka topic은 여섯 개로 고정되어 있습니다.
* **Required External Step 요약:**

  * 다음 consumed source topic을 사전 생성합니다.

    * `wallet.debited.v1`
    * `wallet.debit-failed.v1`
    * `bet.settled.v1`
    * `bet.voided.v1`
    * `bet.resolution.revised.v1`
  * 각각에 대응하는 uppercase-suffix DLT를 생성합니다.

    * `wallet.debited.v1.DLT`
    * `wallet.debit-failed.v1.DLT`
    * `bet.settled.v1.DLT`
    * `bet.voided.v1.DLT`
    * `bet.resolution.revised.v1.DLT`
  * 각 source topic과 DLT의 partition 수를 일치시킵니다.
  * 이 구성을 마친 후 consumer를 활성화합니다.
* **실제 수행 여부 확인 가능성:**
  topic 존재 여부, 실제 partition 수, DLT 생성 시점, retention 또는 replication 설정은 repository에서 확인되지 않습니다.
* **Documentation Action:**
  Thread 12에 topic/DLT provisioning checklist와 "DLT send acknowledgement 이전 source offset을 복구하지 않는다"는 외부 전제조건을 추가합니다.

---

## GAP-ES-04 — Resolution revision의 cross-service rollout 순서

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 16 — 순서화된 정산 수정, 갭, 충돌 격리
* **Related Threads:** Thread 12, Thread 15
* **Repository Evidence 요약:**

  * revision listener는 `bet.resolution.revised.v1`을 소비하며 payload의 `betId`와 Kafka key가 일치하는지 확인합니다. Base settlement/void topic은 `eventId`를 key로 검증합니다.
  * 운영 절차는 **Betting과 Gateway의 revision consumer를 먼저 배포한 뒤 Settlement의 revision producer를 활성화**하도록 명시합니다. 또한 revision topic은 `betId`, base topic은 `eventId`로 keying할 것을 요구합니다.
* **Required External Step 요약:**

  * Betting revision consumer 배포
  * Gateway revision consumer 배포
  * 두 consumer의 topic/key 계약 검증
  * 그 이후 Settlement revision producer 활성화
  * Settlement producer가 `betId`를 record key로 사용하도록 구성
* **실제 수행 여부 확인 가능성:**
  Gateway와 Settlement repository의 배포 상태, producer 활성화 시점, 실제 rollout 순서는 이 repository에서 확인되지 않습니다.
* **Documentation Action:**
  Thread 16에 cross-service deployment ordering supplement를 추가합니다. 이를 실제 수행 이력으로 서술하지 않고 **conceptual execution order**로만 기록합니다.

---

## GAP-ES-05 — PostgreSQL schema delivery, migration safety, restore readiness

* **Classification:** `NEW_THREAD`
* **Primary Owner:** Proposed Thread NT-18
* **Related Threads:** Thread 1, 5, 6, 8, 9, 10, 11, 15, 16, 17
* **Repository Evidence 요약:**

  * repository에는 Flyway V1–V10 migration이 존재합니다.
  * application은 Flyway를 활성화하고 Hibernate를 `ddl-auto=validate`로 설정합니다.
  * integration test는 Testcontainers PostgreSQL 위에서 `flyway_schema_history`의 성공 migration 수가 10인지 검증합니다.
  * 운영 절차에는 backup, checksum 검증, migration role 분리, canary startup, restore 후 FK 및 outbox/receipt 확인이 포함됩니다.
* **Required External Step 요약:**

  * PostgreSQL database와 runtime/migration 권한을 준비합니다.
  * 배포 전 backup을 만듭니다.
  * released migration checksum을 검증합니다.
  * V1–V10을 순차 적용합니다.
  * Hibernate schema validation을 통과시킵니다.
  * canary instance를 확인한 후 traffic을 추가합니다.
  * restore 시 전체 Betting-owned table을 일관되게 복구하고 Flyway/FK/outbox/receipt 상태를 검증합니다.
* **실제 수행 여부 확인 가능성:**
  실제 database, role, backup, migration 실행, canary, restore 수행 여부는 Git에서 확인되지 않습니다.
* **Documentation Action:**
  Proposed Thread NT-18을 생성합니다.

---

## GAP-ES-06 — 방향별 서비스 credential 발급·등록·주입

* **Classification:** `NEW_THREAD`
* **Primary Owner:** Proposed Thread NT-19
* **Related Threads:** Thread 6, 7, 8, 13, 14
* **Repository Evidence 요약:**

  * Risk와 Wallet URL은 서로 다른 absolute HTTP(S) origin이어야 하고, 두 API key는 각각 32자 이상이면서 서로 달라야 합니다.
  * 두 outbound client는 별도 client instance와 별도 key로 구성됩니다.
  * outbound request에는 `X-Internal-Service: betting-service`와 direction-specific API key가 주입됩니다.
  * ingress에서는 `X-Internal-Service: gateway`와 `BETTING_GATEWAY_API_KEY`가 요구되고, Gateway key가 outbound key와 같으면 startup 구성이 거부됩니다.
  * production configuration에는 Risk/Wallet key의 default가 없습니다.
* **Required External Step 요약:**

  * 세 개의 서로 다른 secret을 생성합니다.

    * Gateway → Betting
    * Betting → Risk
    * Betting → Wallet
  * 각 secret을 송신 측과 수신 측에 동일하게 등록합니다.
  * Betting runtime에는 세 secret을 모두 안전하게 주입합니다.
  * Risk와 Wallet의 서로 다른 runtime origin을 구성합니다.
  * 각 방향의 authentication을 별도로 검증합니다.
* **실제 수행 여부 확인 가능성:**
  secret 값, 생성 도구, secret manager, 수신 서비스의 실제 등록 상태, 교체 시점, runtime injection 여부는 확인되지 않습니다.
* **Documentation Action:**
  Proposed Thread NT-19를 생성합니다.

---

## GAP-ES-07 — 고정 shared-protocol source와 Maven artifact 준비

* **Classification:** `NEW_THREAD`
* **Primary Owner:** Proposed Thread NT-20
* **Related Threads:** Thread 2, 6, 7, 8, 9, 10, 12, 15, 16
* **Repository Evidence 요약:**

  * `pom.xml`은 `com.sportsbook:shared-protocol:1.0.0`을 직접 요구합니다.
  * CI workflow는 SHA `f9de6bc1e533761ab4bb1454d8d4ab8175cdf001`을 `shared-protocol` 경로에 checkout하고, 그 경로에서 Maven install을 실행한 뒤 Betting Service를 검증하도록 구성되어 있습니다.
  * commit `a0ce606e4d5a455d08c2eb2e2c0d34fb5de313a2`가 이 고정 SHA/Java 17/install/verify pipeline을 도입합니다.
  * commit `d6a73625429eb4cd856e38127bf51d8a5a9f9434`가 고정 입력을 테스트로 잠급니다.
* **Required External Step 요약:**

  * 정확한 protocol source commit을 확보합니다.
  * 해당 source에서 `shared-protocol:1.0.0`을 build합니다.
  * local 또는 CI Maven repository에 install합니다.
  * 그 후 Betting Service의 `clean verify`를 실행합니다.
* **실제 수행 여부 확인 가능성:**
  workflow는 두 번째 checkout의 별도 `repository:` 값을 기록하지 않습니다. 따라서 이 repository만으로 protocol source repository identity, 실제 checkout 성공, 설치된 artifact bytes/checksum, CI 실행 결과를 확인할 수 없습니다.
* **Documentation Action:**
  Proposed Thread NT-20을 생성하고 source repository 식별과 artifact provenance의 Evidence Boundary를 명시합니다.

---

## GAP-ES-08 — Target runtime deployment와 운영 수집 경로 구성

* **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
* **Primary Owner:** Project Level
* **Related Threads:** 전체 runtime Thread
* **Repository Evidence 요약:**

  * release artifact는 `target/betting-service-1.0.0.jar`로 정의되어 있습니다.
  * HTTP port, graceful shutdown, health probes, Prometheus/metrics endpoint가 application configuration에 존재합니다.
  * `json` profile은 structured console logging을 제공합니다.
  * 현재 repository root에는 Dockerfile, Compose, Kubernetes, Helm 또는 Terraform deployment artifact가 없습니다.
* **Required External Step 요약:**

  * JAR을 실행할 target runtime을 준비합니다.
  * dependency endpoint와 secret을 주입합니다.
  * HTTP port와 dependency network path를 구성합니다.
  * health/readiness probe를 연결합니다.
  * Prometheus scrape 또는 metric collector를 연결합니다.
  * console JSON log를 수집할 runtime logging path를 준비합니다.
  * canary 확인 후 traffic을 전환합니다.
* **실제 수행 여부 확인 가능성:**
  배포 platform, replicas, service discovery, ingress, collector, 실제 traffic shift는 확인되지 않습니다.
* **Documentation Action:**
  프로젝트 수준 Deployment Checklist로 기록합니다. 플랫폼 구현 근거가 없으므로 신규 Thread로 승격하지 않습니다.

---

## GAP-ES-09 — Integration test용 container runtime 준비

* **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
* **Primary Owner:** Project Level
* **Related Threads:** Thread 8, 9, 11, 12
* **Repository Evidence 요약:**

  * integration-test support는 `postgres:16-alpine` Testcontainers image를 시작하고 동적 datasource 값을 주입합니다.
  * Maven에는 Testcontainers PostgreSQL/Kafka dependency가 포함됩니다.
  * CI는 `./mvnw -B clean verify`를 실행합니다.
* **Required External Step 요약:**

  * local 개발환경과 CI runner에 Testcontainers가 사용할 수 있는 container runtime을 제공합니다.
  * `postgres:16-alpine` image를 확보할 수 있어야 합니다.
  * container 생성과 ephemeral port 접근이 허용되어야 합니다.
* **실제 수행 여부 확인 가능성:**
  runner의 container engine, image cache/pull, 실제 integration-test 실행 성공 여부는 Git에서 확인되지 않습니다.
* **Documentation Action:**
  프로젝트 수준 Build/Test Prerequisite로 기록합니다. production lifecycle이 아니므로 독립 Development Thread로 만들지 않습니다.

---

# Part II — Existing Thread Supplement Packets

동일 Gap의 중복 소유를 피하기 위해 이 Part에는 **Primary Owner인 Existing Thread만** 포함합니다. Related Thread에는 Part I의 관계만 기록하고 같은 Gap을 다시 소유시키지 않습니다.

---

## Packet ET-03

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 3
* **한국어 제목:** 실패 폐쇄형 베팅 접수 검증
* **English title:** Fail-Closed Bet Admission Validation

### Gaps

* `GAP-ES-01`

### Repository Evidence

#### Commit `3f8b94d3c84d970a8b7580f9e7e915847124f430`

* **Subject:** `feat(odds): read effective market snapshots`
* **Files:**

  * `src/main/java/com/sportsbook/betting/validation/OddsSnapshotReader.java`
* **Relevant diff:**

  * Redis `market:{eventId}:{marketId}` read
  * value must equal `OPEN`
  * Redis `odds:{eventId}:{marketId}:{selectionId}` read
  * missing or invalid decimal becomes `MarketClosedException`

#### Commit `7cf01b5d66fe1fe103e0eaff1e4962866de81639`

* **Subject:** `test(odds): verify canonical snapshot keys`
* **Files:**

  * `src/test/java/com/sportsbook/betting/validation/OddsSnapshotReaderTest.java`
* **Relevant diff:**

  * exact market/odds key shape and successful decimal read are locked by the test.

#### Runtime configuration

```yaml
spring:
  data:
    redis:
      host: ${BETTING_REDIS_HOST:localhost}
      port: ${BETTING_REDIS_PORT:6379}
      timeout: 200ms
```

#### Final-state ownership evidence

Repository architecture assigns effective odds snapshot authority to **Redis data written by Odds Feed**, not to Betting Service.

### External Development Steps

1. Redis runtime을 준비합니다.
2. Betting Service의 Redis endpoint를 target environment에 바인딩합니다.
3. Odds Feed 또는 동등한 외부 writer가 canonical key/value를 기록하도록 연결합니다.
4. 필요한 market과 selection 값이 존재하는지 확인합니다.
5. 그 후 betting placement traffic을 활성화합니다.

### Code Connection

* Redis market 값이 없거나 `OPEN`이 아니면 bet admission은 닫힙니다.
* selection odds가 없거나 decimal이 아니어도 동일하게 접수를 거부합니다.
* 따라서 이 외부 Redis 상태는 단순 cache 최적화가 아니라 placement decision의 직접 입력입니다.

### Evidence Boundary

* **Directly observed in repository**

  * Redis key 형식
  * `OPEN` 요구
  * decimal odds 요구
  * fail-closed behavior
  * Redis endpoint properties
  * Odds Feed가 writer라는 repository ownership 선언
* **Required/inferred from repository**

  * Redis instance와 Odds Feed writer가 실제 환경에 있어야 함
  * traffic 활성화 전에 필요한 key가 준비되어야 함
* **Actual execution not observable from Git**

  * 실제 Redis host
  * 실제 key/value
  * writer deployment
  * update cadence, TTL, persistence 정책

### Ordering

**Conceptual execution order**

1. Redis 준비
2. endpoint 주입
3. Odds Feed writer 연결
4. canonical key/value 검증
5. Betting Service 시작
6. placement traffic 활성화

---

## Packet ET-09

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 9
* **한국어 제목:** 트랜잭셔널 아웃박스와 승인 기반 최소 한 번 전달
* **English title:** Transactional Outbox and Acknowledged At-Least-Once Delivery

### Gaps

* `GAP-ES-02`

### Repository Evidence

#### Commit `b540b2a17cfac77c8841e93a696946842b49f2ab`

* **Subject:** `feat(database): add transactional outbox`
* **File:** `src/main/resources/db/migration/V2__outbox.sql`
* **Relevant change:** durable outbox table과 unpublished-row 조회용 index 추가.

#### Commit `0f29ae08ed3bc53be1aff67dedcf41d798bd231c`

* **Subject:** `feat(outbox): publish acknowledged pending events`
* **Final-state file:** `src/main/java/com/sportsbook/betting/outbox/OutboxPublisher.java`
* **Relevant behavior:**

  * pending row batch 조회
  * `ProducerRecord(topic, partitionKey, payload)` 생성
  * broker send result 대기
  * acknowledgement 이후에만 row를 published로 변경

#### Topic and connection evidence

* Produced topic: `bet.placed.v1`
* Broker endpoint: `BETTING_KAFKA_BOOTSTRAP`

### External Development Steps

1. Kafka cluster 또는 broker endpoint를 준비합니다.
2. `bet.placed.v1`을 사전 생성합니다.
3. bootstrap address를 Betting runtime에 주입합니다.
4. topic metadata 조회와 publication acknowledgement 경로를 검증합니다.
5. outbox scheduler가 실행되는 Betting instance를 시작합니다.

### Code Connection

* DB transaction은 outbox row까지만 생성합니다.
* Kafka publication 완료는 broker와 topic이 실제로 존재하고 acknowledgement를 반환해야 성립합니다.
* broker 장애 시 row는 pending으로 남고 다음 scheduler 실행에서 재시도됩니다.

### Evidence Boundary

* **Directly observed in repository**

  * outbox table
  * publication topic
  * acknowledgement 대기
  * publication 성공 후 `published_at` 기록
* **Required/inferred from repository**

  * Kafka broker와 `bet.placed.v1` 존재
  * bootstrap address 주입
* **Actual execution not observable from Git**

  * topic 생성
  * 실제 broker acknowledgement
  * 실제 outbox row의 publication
  * partition/replication/retention 설정

### Ordering

**Conceptual execution order**

1. Kafka target 준비
2. `bet.placed.v1` 생성
3. bootstrap address 주입
4. Betting instance 시작
5. metadata/ack smoke test
6. placement traffic 활성화

---

## Packet ET-12

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 12
* **한국어 제목:** Kafka 영구 오류 분류와 승인 기반 DLT 복구
* **English title:** Kafka Permanent Failure Classification and Acknowledged DLT Recovery

### Gaps

* `GAP-ES-03`

### Repository Evidence

#### Commit `931a0b3134efd813453511c925211dbf76f3fbc6`

* **Subject:** `feat(messaging): require preprovisioned topics`
* **File:** `src/main/resources/application.yml`
* **Relevant diff:**

```yaml
spring:
  kafka:
    consumer:
      properties:
        allow.auto.create.topics: false
```

#### Commit `2c3d5abfaf456bfb17d0ee1326c067052f719404`

* **Subject:** `test(messaging): require preprovisioned topics`
* **File:** `RuntimeConfigurationTest.java`
* **Relevant diff:** configuration test가 auto-create 비활성화를 고정합니다.

#### DLT partition behavior

```java
new TopicPartition(record.topic() + ".DLT", record.partition())
```

DLT send failure가 있으면 source recovery 성공으로 처리하지 않으며 send result를 기다립니다.

### External Development Steps

1. consumed source topic 다섯 개를 생성합니다.
2. 각 source topic에 대응하는 `.DLT` topic을 생성합니다.
3. source/DLT partition 수를 일치시킵니다.
4. topic metadata를 확인합니다.
5. 이후 Betting consumer를 활성화합니다.
6. malformed test record로 same-partition DLT routing과 acknowledgement를 검증합니다.

### Code Connection

* DLT target partition은 source partition 번호로 강제됩니다.
* 따라서 source에는 존재하지만 DLT에는 없는 partition이 있으면 recovery가 완료될 수 없습니다.
* consumer auto-creation이 비활성화되어 있으므로 listener 실행이 topology 생성 절차를 대신하지 않습니다.

### Evidence Boundary

* **Directly observed in repository**

  * topic 이름
  * auto-create 비활성화
  * `.DLT` suffix
  * source partition 보존
  * DLT acknowledgement 요구
* **Required/inferred from repository**

  * source/DLT 사전 생성
  * partition 수 일치
* **Actual execution not observable from Git**

  * topic 존재 여부
  * partition 수
  * DLT record 생성
  * retention, replication, broker security 설정

### Ordering

**Conceptual execution order**

1. source topic 생성
2. matching DLT 생성
3. metadata/partition 검증
4. consumer 배포
5. DLT acknowledgement smoke test
6. upstream producer 활성화

---

## Packet ET-16

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 16
* **한국어 제목:** 순서화된 정산 수정, 갭, 충돌 격리
* **English title:** Ordered Settlement Revisions, Gaps, and Conflict Isolation

### Gaps

* `GAP-ES-04`

### Repository Evidence

#### Commit `7718e82047dc8e7a7be1bef8860c4550fa38d2f3`

* **Subject:** `feat(settlement): consume ordered resolution revisions`
* **Files:**

  * `BetSettlementService.java`
  * `SettlementResultListener.java`
* **Relevant diff:**

  * `bet.resolution.revised.v1` listener 추가
  * revision payload 적용
  * revision record key가 payload의 `betId`인지 검증
  * base settlement/void는 `eventId` key 검증

#### Operational rollout contract

* Betting과 Gateway의 revision consumer를 먼저 배포
* 이후 Settlement revision producer를 활성화
* revision topic key는 `betId`
* base resolution topic key는 `eventId`

### External Development Steps

1. revision source/DLT topic을 준비합니다.
2. Betting revision consumer를 배포합니다.
3. Gateway revision consumer를 배포합니다.
4. 두 consumer가 `betId` key를 처리하는지 검증합니다.
5. 이후 Settlement producer를 활성화합니다.
6. 순차 revision, duplicate, gap, conflicting equal revision을 smoke-test합니다.

### Code Connection

* Betting listener는 key mismatch를 정상 revision으로 적용하지 않습니다.
* Settlement producer가 잘못된 key를 사용하면 permanent failure 및 DLT 경로로 이어집니다.
* producer를 먼저 활성화하면 준비되지 않은 consumer 또는 downstream 계약으로 revision이 유입될 수 있습니다.

### Evidence Boundary

* **Directly observed in repository**

  * Betting consumer topic
  * key 검증
  * 운영 문서의 요구 rollout 순서
* **Required/inferred from repository**

  * Gateway consumer 선배포
  * Settlement producer 후활성화
* **Actual execution not observable from Git**

  * 다른 서비스의 배포 버전
  * 실제 producer key
  * 실제 rollout 시점과 순서

### Ordering

위 단계는 실제 history가 아니라 **cross-service conceptual execution order**입니다.

---

# Part III — Proposed New Thread Packets

---

## Proposed Thread NT-18

### Thread Identity

* **Type:** Proposed New Thread
* **Temporary ID:** NT-18
* **한국어 제목:** 데이터베이스 스키마 배포와 복구 준비
* **English title:** Database Schema Delivery and Restore Readiness

### Gaps

* `GAP-ES-05`

### NEW_THREAD 판정 근거

이 관점은 개별 domain table의 설계가 아니라 다음을 하나의 lifecycle로 다룹니다.

* database provisioning
* migration authority와 application authority 분리
* immutable migration 검증
* 순차 schema rollout
* schema validation
* canary startup
* pre-rollout backup
* partial/failed migration 대응
* full-state restore 검증

여러 기존 Thread의 테이블을 관통하고, 독립적인 실패·복구 조건을 가지며, repository에서 명확한 representative commit 집합을 선정할 수 있으므로 `NEW_THREAD`로 판정합니다.

### Representative Commits

| Commit | Subject | 이 Thread에서 중요한 이유 |
| --- | --- | --- |
| `6cf6cec95ef146a260d7a4a97e6e849453d32a39` | `feat(database): create bet aggregate schema`    | V1에서 `bet`, `bet_leg`, constraints와 indexes를 생성합니다.  |
| `b540b2a17cfac77c8841e93a696946842b49f2ab` | `feat(database): add transactional outbox`       | V2에서 outbox durable state를 추가합니다.    |
| `682bd691d92f3af0106f088cf84cafd586de124b` | `feat(database): add settlement outcome columns` | V3에서 settlement projection state를 추가합니다.     |
| `7e2b8bfe47b1a7ea72bc16386f30711d23e3c675` | `feat(database): add whole-slip void reason`     | V4에서 void projection schema를 확장합니다.  |
| `b28d0652abafffbaa71757006390146b988ba3cd` | `feat(database): add placement recovery evidence`        | V5에서 recovery phase와 request fingerprint 등 durable evidence를 추가합니다.  |
| `90f1207d649e2de4629cc0e270c427f2ed8ffc14` | `feat(database): add compensation verdict schema`        | V6에서 compensation/wallet/risk proof와 placement request state를 추가합니다. |
| `21abf421338bfe8161a226de1358651c6b221937` | `feat(database): persist risk reservation tokens`        | V7에서 opaque risk token을 영속화합니다.      |
| `2a628dbb54c1e2ee72f67e6f66cbf02ba8718aec` | `feat(database): track wallet reconciliation hints`      | V8에서 wallet receipt와 reconciliation checkpoint를 추가합니다.       |
| `8947d22de5f23bdecdf799b07268562d114c81de` | `feat(database): persist resolution revisions`   | V9에서 revision number/identity/hash/proof를 추가합니다.     |
| `2076b8e489ca8f27f38a1d3ff0b8aaef42791cfe` | `feat(recovery): add owner-fenced reconciliation leases` | V10에서 owner/lease/eligibility 상태와 partial index를 추가합니다.      |
| `46a227c02b64c3083dc54073b82bfd0bde95c7ca` | `feat(runtime): configure durable service dependencies`  | Flyway, datasource, `ddl-auto=validate`와 connection binding을 구성합니다.  |
| `2de78ea64d1d91062a8bb33c3765d36f6423b1ce` | `test(integration): provide PostgreSQL runtime`  | `postgres:16-alpine` Testcontainers runtime을 도입합니다.  |
| `b1ffdd534bff223696245a4f52048c73033ab985` | `test(integration): verify PostgreSQL placement outbox`  | 실제 PostgreSQL 위에서 migration과 saga/outbox persistence를 검증하기 시작합니다.    |

### Thread-Direct Diff Packet

#### Runtime migration authority

```yaml
spring:
  datasource:
    url: ${BETTING_DB_URL:jdbc:postgresql://localhost:5432/betting}
    username: ${BETTING_DB_USER:betting}
    password: ${BETTING_DB_PASSWORD:betting}
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
```

#### Schema chain

* **V1:** `bet`, `bet_leg`, idempotency/reference uniqueness, slip/status constraints, lookup indexes
* **V2:** `outbox_event`
* **V3:** settlement outcome/payout projection
* **V4:** whole-slip void reason
* **V5:** request fingerprint, placement phase, recovery evidence
* **V6:** compensation state/action, wallet proof, risk-commit observation, durable placement request
* **V7:** risk reservation token
* **V8:** wallet-event receipt와 reconciliation request time
* **V9:** base/revision identity, number, payload hash, source result time
* **V10:** database-time eligibility, claim owner, claim expiry, claim consistency check와 partial index

Current migration directory에는 V1–V10이 모두 존재합니다.

#### Final-state verification excerpt

```java
select count(*)
from flyway_schema_history
where success
```

Expected count is `10`.

### External Development Steps

1. Target PostgreSQL database를 준비합니다.
2. migration process와 application runtime의 권한 경계를 설정합니다.
3. `BETTING_DB_URL`, user, password를 target에 바인딩합니다.
4. 배포 전 일관된 backup을 생성합니다.
5. released V1–V10 checksum을 검증합니다.
6. migration process가 V1–V10을 순차 적용합니다.
7. application startup에서 Hibernate validation을 수행합니다.
8. 한 instance를 canary로 시작해 Flyway/Hibernate 상태를 확인합니다.
9. 이후 instance 및 traffic을 확대합니다.
10. 장애 또는 restore 시 Betting-owned 전체 table을 함께 복구하고 Flyway/FK/outbox/receipt/reconciliation 상태를 검증합니다.

### Code Connection

* Thread 1의 aggregate persistence는 V1에 의존합니다.
* Thread 5/8의 placement request와 compensation은 V5–V6에 의존합니다.
* Thread 6의 risk proof는 V7에 의존합니다.
* Thread 9의 outbox는 V2에 의존합니다.
* Thread 10의 wallet-event inbox는 V8에 의존합니다.
* Thread 11의 owner fencing은 V10에 의존합니다.
* Thread 15/16의 settlement projection은 V3, V4, V9에 의존합니다.
* Thread 17의 row-locking은 실제 PostgreSQL schema/index/transaction state를 전제로 합니다.

### Evidence Boundary

#### Directly observed in repository

* V1–V10 SQL
* migration을 만든 commit
* Flyway 활성화
* Hibernate `validate`
* Testcontainers PostgreSQL
* 성공 migration 수 10 검증
* backup/checksum/canary/restore 요구가 적힌 repository 운영 절차

#### Required/inferred from repository

* target database 생성
* role/permission 구성
* backup 생성
* V1–V10 실행
* canary startup
* restore 검증

#### Actual execution not observable from Git

* 실제 database 이름과 host
* 실제 production/staging role
* 실제 migration 실행 시점과 결과
* backup 및 restore artifact
* canary deployment
* migration failure/rollback 이력

### Ordering

다음은 실제 수행 이력이 아니라 **conceptual execution order**입니다.

1. database와 권한 준비
2. target connection binding
3. pre-rollout backup
4. released checksum 검증
5. V1–V10 적용
6. Hibernate validation
7. canary startup
8. traffic 확대
9. runtime monitoring
10. 장애 시 full-state restore와 post-restore validation

---

## Proposed Thread NT-19

### Thread Identity

* **Type:** Proposed New Thread
* **Temporary ID:** NT-19
* **한국어 제목:** 방향별 서비스 자격증명과 런타임 바인딩
* **English title:** Direction-Scoped Service Credentials and Runtime Binding

### Gaps

* `GAP-ES-06`

### NEW_THREAD 판정 근거

이 관점은 단순히 여러 환경변수를 나열하는 것이 아닙니다.

* ingress와 두 outbound direction의 credential namespace 분리
* pairwise distinctness
* 송신자 identity header
* 수신자별 client isolation
* endpoint origin validation
* startup fail-fast
* secret redaction
* 송신·수신 서비스 사이의 bilateral registration
* mismatch 시 startup failure 또는 authentication failure

라는 자체 수명주기와 실패조건을 가집니다. 또한 기존 Thread에 포함되지 않은 명확한 commit cluster가 있어 `NEW_THREAD` 요건을 충족합니다.

### Representative Commits

| Commit     | Subject  | 이 Thread에서 중요한 이유       |
| --- | --- | --- |
| `b444712f40f55e4af6e88f915ed65f1f06c3f963` | `feat(client): validate dependency credentials`  | Risk/Wallet origin과 API key를 검증하고 direction 간 중복을 금지합니다.        |
| `50e3d596727413ea5e6617c90348cbdcf7c92060` | `test(client): reject unsafe dependency credentials`     | short/shared key와 unsafe/shared endpoint가 거부되는지 검증합니다.  |
| `4444fee8de8ac53e3bcce57b5b89948046b2b663` | `feat(client): inject betting caller identity`   | outbound request에 caller와 direction-specific key header를 삽입합니다. |
| `b5cce3e4c867f769c529380841d730cc42e84241` | `feat(client): isolate authenticated dependency clients` | Risk와 Wallet client instance를 분리합니다.    |
| `a397d3c1fa2ca80c5bc24ed024f12fcda553c548` | `feat(api): authenticate the gateway boundary`   | Gateway ingress key, caller identity, key distinctness를 도입합니다.  |
| `46a227c02b64c3083dc54073b82bfd0bde95c7ca` | `feat(runtime): configure durable service dependencies`  | Risk/Wallet endpoint와 key environment binding을 정의합니다.   |
| `b34b0d5f39ba47891d25e923adbd86075cad10fa` | `test(runtime): verify production dependency wiring`     | outbound key에 production default가 없음을 검증합니다.    |

### Thread-Direct Diff Packet

#### Outbound credentials and endpoints

```java
riskBaseUrl = requireEndpoint(riskBaseUrl, "riskBaseUrl");
walletBaseUrl = requireEndpoint(walletBaseUrl, "walletBaseUrl");

if (riskEndpoint.equals(walletEndpoint)) {
  throw new IllegalArgumentException(
      "Risk and Wallet destinations must be distinct");
}

riskApiKey = requireSecret(riskApiKey, "BETTING_RISK_API_KEY");
walletApiKey = requireSecret(walletApiKey, "BETTING_WALLET_API_KEY");

if (riskApiKey.equals(walletApiKey)) {
  throw new IllegalArgumentException(
      "Dependency API keys must be distinct");
}
```

#### Outbound caller headers

```text
X-Internal-Service: betting-service
X-Internal-Api-Key: <direction-specific key>
```

#### Gateway ingress

```text
X-Internal-Service: gateway
X-Internal-Api-Key: <BETTING_GATEWAY_API_KEY>
```

Gateway key는 32자 이상이어야 하며 Risk/Wallet key와 달라야 합니다.

#### Runtime binding

```yaml
betting:
  clients:
    risk-base-url: ${RISK_BASE_URL:http://localhost:8083}
    wallet-base-url: ${WALLET_BASE_URL:http://localhost:8081}
    risk-api-key: ${BETTING_RISK_API_KEY}
    wallet-api-key: ${BETTING_WALLET_API_KEY}
```

Gateway key는 `@Value("${BETTING_GATEWAY_API_KEY}")`로 직접 주입됩니다.

### External Development Steps

1. Risk와 Wallet의 target origin을 결정합니다.
2. 세 방향에 각각 사용할 서로 다른 secret을 생성합니다.
3. Gateway와 Betting에 Gateway→Betting secret을 등록합니다.
4. Betting과 Risk에 Betting→Risk secret을 등록합니다.
5. Betting과 Wallet에 Betting→Wallet secret을 등록합니다.
6. Betting runtime에 세 secret과 두 outbound URL을 주입합니다.
7. Gateway ingress, Risk outbound, Wallet outbound를 각각 smoke-test합니다.
8. mismatch 시 새 요청을 보내기 전에 구성 값을 수정하고 관련 instance를 재시작합니다.

### Code Connection

* Gateway secret이 없거나 너무 짧으면 Betting startup 구성이 성립하지 않습니다.
* Risk/Wallet secret이 없거나 중복되면 client configuration이 성립하지 않습니다.
* 송신 측과 수신 측 값이 다르면 HTTP authentication이 실패합니다.
* Risk key와 Wallet key를 바꾸어 주입해도 client isolation 때문에 올바른 상대 서비스 인증을 통과하지 못합니다.
* Risk와 Wallet URL이 동일한 canonical origin이면 startup 시 거부됩니다.

### Evidence Boundary

#### Directly observed in repository

* 세 secret의 이름
* 최소 길이
* pairwise distinctness
* caller header
* separate client
* endpoint validation
* secret redaction
* production default 부재

#### Required/inferred from repository

* secret 생성
* 송신·수신 양측 등록
* runtime injection
* target endpoint/network 구성
* direction별 authentication smoke test

#### Actual execution not observable from Git

* secret 값
* secret manager/provider
* 외부 서비스의 등록 상태
* 실제 endpoint
* secret 교체 이력
* 인증 성공 여부

### Ordering

**Conceptual execution order**

1. service origin 결정
2. 세 secret 생성
3. 수신 서비스에 expected secret 등록
4. 송신 서비스에 corresponding secret 주입
5. Betting runtime 구성
6. direction별 authentication 검증
7. traffic 활성화

Repository에는 dual-key overlap이나 자동 rotation protocol이 없으므로, 무중단 rotation 방법을 임의로 추가하지 않습니다.

---

## Proposed Thread NT-20

### Thread Identity

* **Type:** Proposed New Thread
* **Temporary ID:** NT-20
* **한국어 제목:** 고정 프로토콜 아티팩트 부트스트랩과 재현 가능한 검증
* **English title:** Pinned Protocol Artifact Bootstrap and Reproducible Verification

### Gaps

* `GAP-ES-07`

### NEW_THREAD 판정 근거

이 관점은 다음 독립 단계를 가집니다.

* cross-repository source identity 고정
* source commit 확보
* protocol artifact build
* local/CI Maven repository 설치
* consumer repository build
* version/SHA mismatch 검출
* missing artifact 시 재부트스트랩

artifact installation은 Git working tree가 아니라 Maven repository state를 변경하며, 해당 상태가 없으면 Betting Service build가 성립하지 않습니다. 명확한 CI와 검증 commit이 있으므로 `NEW_THREAD`로 판정합니다.

### Representative Commits

#### `a0ce606e4d5a455d08c2eb2e2c0d34fb5de313a2`

* **Subject:** `ci(verify): pin Java and protocol inputs`
* **Files:** `.github/workflows/verify.yml`
* **왜 중요한가:**

  * protocol SHA 고정
  * `shared-protocol` working directory 지정
  * Java 17 고정
  * protocol Maven install
  * Betting `clean verify` 순서 정의

#### `d6a73625429eb4cd856e38127bf51d8a5a9f9434`

* **Subject:** `test(ci): verify pinned build inputs`
* **Files:** `ContinuousIntegrationTest.java`
* **왜 중요한가:**

  * fixed SHA, Java 17, install, verify command가 workflow에서 사라지지 않도록 검증합니다.

#### `647da6058a560472a064e1271596fc5764d68256`

* **Subject:** `chore(release): release betting service 1.0.0`
* **Files:** `pom.xml`
* **왜 중요한가:**

  * Betting Service 자체를 `1.0.0` release artifact로 고정합니다.

### Thread-Direct Diff Packet

#### Consumer dependency

```xml
<dependency>
  <groupId>com.sportsbook</groupId>
  <artifactId>shared-protocol</artifactId>
  <version>1.0.0</version>
</dependency>
```

#### CI bootstrap

```yaml
- name: Check out fixed shared protocol
  uses: actions/checkout@v4
  with:
    ref: f9de6bc1e533761ab4bb1454d8d4ab8175cdf001
    path: shared-protocol

- name: Install shared protocol 1.0.0
  working-directory: shared-protocol
  run: ./mvnw -B -DskipTests install

- name: Verify betting service
  run: ./mvnw -B clean verify
```

#### Final-state project instruction

README 역시 fixed protocol artifact를 local Maven repository에 먼저 설치한 후 Betting build를 실행하도록 요구합니다.

### External Development Steps

1. SHA `f9de6bc1e533761ab4bb1454d8d4ab8175cdf001`의 protocol source를 확보합니다.
2. source가 의도한 repository와 commit에 해당하는지 검증합니다.
3. protocol project에서 Maven install을 실행합니다.
4. local/CI Maven repository에 `com.sportsbook:shared-protocol:1.0.0`이 생성되었는지 확인합니다.
5. 같은 Java 17 toolchain에서 Betting Service `clean verify`를 실행합니다.
6. missing 또는 mismatched artifact가 있으면 임의 버전으로 대체하지 않고 정확한 source에서 다시 install합니다.

### Code Connection

다음 runtime contract가 모두 shared protocol artifact의 type/schema에 의존합니다.

* SYSTEM monetary semantics
* Risk/Wallet request와 response types
* Bet placement events
* Wallet event inbox
* settlement/void/revision Kafka payload
* durable error codes와 verdict mapping
* strict Avro serialization/deserialization

### Evidence Boundary

#### Directly observed in repository

* dependency coordinates `com.sportsbook:shared-protocol:1.0.0`
* fixed commit SHA
* Java 17
* protocol install → Betting verify 순서
* CI test가 해당 입력을 검증함

#### Required/inferred from repository

* protocol source의 실제 확보
* source build
* Maven repository install
* Betting build 이전 artifact availability

#### Actual execution not observable from Git

* protocol source repository identity
* checkout 성공
* installed artifact bytes/checksum
* local Maven repository 상태
* CI run 성공 여부

특히 workflow에는 별도 `repository:` 값이 없으므로, 이 repository만으로 fixed SHA가 어느 외부 repository에 속하는지 확정할 수 없습니다.

### Ordering

**Conceptual execution order**

1. source repository identity 확인
2. fixed SHA checkout
3. Java 17 준비
4. protocol `install`
5. artifact coordinates 확인
6. Betting `clean verify`
7. release artifact 생성

---

# Part IV — Project-Level External Steps

## PL-01 — Runtime deployment, health/metrics/log collection, traffic activation

### Repository Evidence

* Spring Boot artifact: `target/betting-service-1.0.0.jar`
* HTTP port: `BETTING_HTTP_PORT`, default `8082`
* graceful shutdown
* health probe 활성화
* health/info/prometheus/metrics endpoint 노출
* `service=betting-service` metric tag
* `json` profile의 structured console log

### External Development Steps

1. 실행 host/container/platform을 준비합니다.
2. JAR을 배치합니다.
3. DB, Redis, Kafka, Risk, Wallet network path와 environment를 주입합니다.
4. health/readiness probe를 deployment platform에 연결합니다.
5. Prometheus 또는 동등한 collector를 연결합니다.
6. JSON console output을 log collector에 연결합니다.
7. canary 검증 후 production traffic을 전환합니다.

### Evidence Boundary

* **Directly observed:** application-side endpoint와 log format
* **Required/inferred:** surrounding deployment/collector/network 구성
* **Not observable:** 실제 platform, replicas, probes, scrape target, log sink, traffic shift

### 왜 NEW_THREAD가 아닌가

Repository에는 Dockerfile, Compose, Kubernetes, Helm, Terraform 또는 platform-specific deployment artifact가 없습니다. Application runtime 요구는 확인되지만 독립 deployment implementation을 설명할 commit 집합이 부족하므로 프로젝트 수준 단계로 유지합니다.

---

## PL-02 — Container-capable test runner

### Repository Evidence

* `PostgresIntegrationSupport`가 `postgres:16-alpine` Testcontainers instance를 생성합니다.
* integration test가 실제 PostgreSQL과 Flyway V1–V10을 사용합니다.
* CI gate는 `clean verify`입니다.

### External Development Steps

1. Docker-compatible container runtime이 있는 runner를 준비합니다.
2. `postgres:16-alpine` image를 사용할 수 있게 합니다.
3. ephemeral container와 mapped port 사용을 허용합니다.
4. 그 환경에서 `clean verify`를 실행합니다.

### Evidence Boundary

* **Directly observed:** Testcontainers class, image name, dynamic datasource wiring
* **Required/inferred:** container runtime과 image availability
* **Not observable:** 실제 local/CI runtime, image pull/cache, test execution 결과

### 왜 NEW_THREAD가 아닌가

이는 production system 관점이 아니라 integration-test 실행 전제이며, 자체적인 장기 runtime state를 소유하지 않습니다.

---

## 명시적으로 채택하지 않은 항목

다음 항목은 repository에서 구체적인 필요성을 확인할 수 없어 Gap으로 만들지 않았습니다.

* database seed 실행
* 별도 bootstrap/init SQL 실행
* OAuth application/provider 등록
* webhook endpoint 외부 등록
* object storage 또는 bucket/IAM 구성
* DNS/domain verification
* TLS certificate 발급·배치
* Kafka SASL/TLS credential 또는 ACL provisioning
* Redis authentication, TTL 또는 persistence 정책
* 외부 cron/scheduler 등록

특히 outbox와 reconciliation은 외부 cron이 아니라 process 내부의 별도 `ThreadPoolTaskScheduler`에서 실행됩니다. 따라서 external scheduler registration Gap은 만들지 않았습니다.

또한 실제 secret 값, DB credential, runtime host, Kafka topic 설정값, Redis contents, migration 실행 이력, deployment 이력은 어느 것도 추정하지 않았습니다.
