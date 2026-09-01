# External-State Development Gap Audit

**Repository:** `tmp-sportsbook-settlement-service`

## 감사 기준과 결론

첨부된 Index의 22개 Thread 제목·번호·commit 귀속만 소유권 판정에 사용했으며, 기존 Thread별 해설 내용은 분석 근거로 사용하지 않았습니다. 

저장소의 root commit부터 현재 `main` HEAD까지는 root 이후 505개 commit, 즉 총 506개 commit으로 확인됩니다. 이 전체 이력에서 database/migration, Kafka/topic/DLT, credential/authentication, worker/scheduler, health/metrics, CI/build/release, deployment/backup/seed 등의 외부 상태 신호를 검색하고 관련 diff를 선별했습니다.  현재 원격 branch ref는 `main` 하나입니다.

최종 판정은 다음과 같습니다.

| 구분 |  수 |
| --- | -: |
| 발견된 Gap  |  9 |
| `EXISTING_THREAD` |  8 |
| `NEW_THREAD`   |  1 |
| `PROJECT_LEVEL_EXTERNAL_STEP` |  0 |
| 실제 외부 실행이 Git으로 증명된 Gap |  0 |

---

# Part I — Gap Index

## ESG-DB-01 — PostgreSQL 프로비저닝과 Flyway 스키마 활성화

**Classification:** `NEW_THREAD`
**Primary Owner:** Proposed Thread `NT-23`
**Related Threads:** 1, 2, 10, 11, 13, 14, 15, 16, 19, 21

**Repository Evidence**

* V1에서 JPA, Flyway, PostgreSQL driver를 추가하고 최초 `bet`, `bet_selection` 스키마를 만들었습니다.
* 이후 outbox, match result, settlement attempt, lifecycle, result candidate, revision, admin-action 상태가 V3–V10 migration으로 누적됩니다. 최종 트리에는 V1과 V3–V10이 존재합니다.
* PostgreSQL 16 Testcontainers harness가 datasource URL·username·password를 동적으로 주입하고 Flyway를 활성화합니다.
* 통합시험은 Flyway validation 성공, pending migration 없음, JPA `ddl-auto=validate`를 확인합니다.

**Required External Step**

호환되는 PostgreSQL database/schema와 접속 자격증명을 만들고, 표준 Spring datasource 설정을 주입해야 합니다. Migration 실행 주체는 테이블·인덱스·함수·트리거 생성에 필요한 권한을 가져야 하며, Flyway migration이 적용된 뒤 JPA schema validation이 성공해야 합니다. 배포 후에는 Flyway schema history와 기존 migration checksum을 보존하고 이후 변경을 V11 이상으로 전진시켜야 합니다.

**실제 수행 여부 확인 가능성:** 확인 불가. 특정 database의 생성, role grant, datasource 값, Flyway 실행 결과 및 운영 `flyway_schema_history` 상태는 Git에 남아 있지 않습니다.

**Documentation Action:** `NT-23` 신규 Thread 생성. 기존 도메인 Thread의 migration 설명을 재작성하지 않고, PostgreSQL 외부 상태와 Flyway 진화 수명주기만 독립 문서화합니다.

---

## ESG-KAFKA-01 — Kafka 연결, 기본 토픽 및 Exact DLT 프로비저닝

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 3 — 원시 Kafka 경계와 정확한 DLT 복구
**Related Threads:** 2, 5, 9, 11, 12, 17, 21

**Repository Evidence**

* 여섯 기본 토픽명이 고유해야 하며 DLT 이름은 `<source-topic>.DLT`로 계산됩니다.
* 세 입력 listener는 placement, match result, event lifecycle을 소비하고, 세 terminal/revision 토픽으로 출력합니다.
* Kafka topic 자동 생성은 꺼져 있고 consumer group은 `settlement-service`, 초기 offset 정책은 `earliest`, commit은 수동입니다.
* DLT producer는 idempotence와 `acks=all`을 사용하며, poison record를 원본과 동일한 partition 번호의 DLT로 발행합니다.

**Required External Step**

Kafka cluster에 대한 bootstrap connection을 주입하고 다음 토픽을 명시적으로 만들어야 합니다.

* 기본 토픽 6개: `bet.placed.v1`, `match.result`, `event.lifecycle`, `bet.settled.v1`, `bet.voided.v1`, `bet.resolution.revised.v1`
* 입력 DLT 3개: 각 입력 토픽의 대문자 `.DLT` companion
* 각 source와 DLT 사이의 동일 partition coverage

토픽명을 override한다면 upstream producer, downstream consumer 및 DLT 이름을 함께 정렬해야 합니다. Runtime이 시작되면 `settlement-service` consumer-group offset도 Kafka 외부 상태로 생성·갱신됩니다.

**실제 수행 여부 확인 가능성:** 확인 불가. Broker 주소, 토픽 존재 여부, partition 수, replication, 기존 consumer offset, 접근 권한은 Git에서 확인되지 않습니다.

**Documentation Action:** Thread 3 supplement에 브로커 연결, 9개 토픽 프로비저닝, partition 일치, consumer offset 경계를 추가합니다.

---

## ESG-AUTH-01 — Admin/Wallet 비밀 생성과 교차 서비스 주입

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 18 — 내부 호출 인증과 자격증명 분리
**Related Threads:** 8, 16, 21

**Repository Evidence**

* Admin API key는 필수이며 최소 32자이고 `X-Service-Name: admin-api`, `X-API-Key` 계약에 연결됩니다.
* Wallet key는 `X-Internal-Service: settlement-service`, `X-Internal-Api-Key` 헤더에 기록됩니다.
* Admin key와 Wallet key가 같으면 애플리케이션 시작이 거부됩니다.
* 최종 설정에는 두 값 모두 default 없이 필수 placeholder로 남습니다.

**Required External Step**

서로 다른 최소 32자 비밀 두 개를 생성해야 합니다.

1. `SETTLEMENT_ADMIN_API_KEY`를 Settlement runtime과 Admin API caller에 일치하게 주입
2. `SETTLEMENT_WALLET_API_KEY`를 Settlement runtime에 주입
3. 같은 Wallet 비밀을 Wallet runtime의 대응 설정인 `WALLET_SETTLEMENT_SERVICE_API_KEY`에 주입
4. 값 변경 시 양쪽 서비스를 일치시키는 배포 순서를 결정

특정 secret manager, CI secret store 또는 암호화 방식을 repository가 요구하지는 않습니다.

**실제 수행 여부 확인 가능성:** 확인 불가. 실제 값, 생성 시점, 저장 위치, 주입 플랫폼, 변경 이력은 관찰할 수 없습니다.

**Documentation Action:** Thread 18 supplement에 두 신뢰 경계의 비밀 생성·분리·양방향 주입과 실제 값 비관찰 원칙을 추가합니다.

---

## ESG-WALLET-01 — Wallet origin 배치와 런타임 네트워크 도달성

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 8 — Wallet HTTP 계약과 작업 증거 검증
**Related Threads:** 7, 10, 14, 15, 17, 18, 21

**Repository Evidence**

* Wallet base URL은 기본적으로 `http://localhost:8081`이며, credential/path/query/fragment가 없는 HTTP(S) origin만 허용됩니다.
* 최초 client 구현은 외부 Wallet origin에 idempotent credit 요청을 연결했습니다.
* 최종 client는 같은 origin에서 credit, forfeit, adjustment POST와 adjustment 조회 경로를 사용합니다.
* Connect/read timeout은 양수이고 각각 최대 5초입니다.

**Required External Step**

Wallet service가 해당 HTTP 계약을 제공하도록 배포되어 있어야 하며, Settlement runtime에서 그 origin으로 네트워크 연결이 가능해야 합니다. 실제 환경에 맞는 `SETTLEMENT_WALLET_BASE_URL`과 timeout을 주입해야 합니다. 기본 localhost 주소는 Wallet이 같은 network namespace의 8081에 있을 때만 유효합니다.

TLS는 지원되지만 HTTP도 코드상 허용되므로, repository만으로 인증서 발급을 필수 단계로 만들 수는 없습니다.

**실제 수행 여부 확인 가능성:** 확인 불가. Wallet 배포 주소, service discovery, firewall, route, TLS 사용 여부, 실제 API 응답 상태는 Git에 없습니다.

**Documentation Action:** Thread 8 supplement에 Wallet origin 배치, 네트워크 도달성, timeout, credential Gap과의 연결 관계를 추가합니다.

---

## ESG-TIME-01 — 데이터베이스 시각을 신뢰 시간원으로 운영

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 19 — 데이터베이스 시각 기반 소유권 펜싱
**Related Threads:** 10, 15, 20, 21, NT-23

**Repository Evidence**

* `DatabaseTimeSource`는 애플리케이션 clock 대신 `select current_timestamp`를 읽습니다.
* Settlement claim은 `current_timestamp`로 lease 만료 시각과 생성·갱신 시각을 원자적으로 계산합니다.
* README도 claim과 lease가 PostgreSQL database time을 사용한다고 명시합니다.

**Required External Step**

대상 PostgreSQL 서비스 또는 DB host가 신뢰할 수 있는 clock을 제공하도록 운영해야 합니다. 구체적으로는 DB clock이 과도하게 앞서거나 뒤처지지 않도록 해당 플랫폼의 시간 동기화 메커니즘을 유지해야 합니다. Timezone 문자열 설정이 아니라 `Instant`로 변환되는 실제 database wall-clock 값이 lease와 correction ordering의 기준입니다.

**실제 수행 여부 확인 가능성:** 확인 불가. DB host clock, 동기화 방식, drift, clock correction 이력은 repository에서 볼 수 없습니다.

**Documentation Action:** Thread 19 supplement에 "database clock은 외부 운영 전제"라는 점과 clock 이상이 lease 만료를 앞당기거나 지연시킬 수 있다는 실패 경계를 추가합니다.

---

## ESG-WORKER-01 — Worker 역할 활성화와 종료 유예 구성

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 20 — 작업자 격리와 런타임 제어
**Related Threads:** 4, 10, 11, 15, 17, 19, 21

**Repository Evidence**

* Worker 설정은 runtime property로 제어되며 값이 없으면 기본 활성화됩니다.
* Outbox, lifecycle, base recovery, revision recovery, correction에 각각 독립 scheduler가 있습니다. 각 scheduler는 단일 thread, 최대 10초 종료 대기, 신규 delayed/periodic task 중단 정책을 사용합니다.
* Spring 전체 shutdown phase는 20초로 구성됩니다.
* 종료 중 반복 작업 재개를 막는 정책이 별도 commit으로 강화되었습니다.

**Required External Step**

각 배포 instance의 역할을 정하고 worker 활성화 값을 구성해야 합니다. 분리 배포를 사용한다면 HTTP-only instance에서는 worker를 끄고, 적어도 하나의 instance에서는 recovery/outbox worker를 켜야 합니다. 분리 배포가 repository의 필수 조건은 아닙니다.

Deployment platform의 강제 종료 유예가 Spring의 bounded shutdown과 scheduler 대기 정책을 선점하지 않도록 구성해야 합니다. 구체적인 최소 grace-period 값은 플랫폼 overhead와 shutdown phase 실행 방식까지 repository가 정의하지 않으므로 단정할 수 없습니다.

**실제 수행 여부 확인 가능성:** 확인 불가. Instance 수, worker-enabled 배치, rolling deployment 방식, 실제 termination grace는 Git에 없습니다.

**Documentation Action:** Thread 20 supplement에 worker role matrix, 기본값 `true`, 최소 한 개 worker-enabled runtime 필요성 및 orchestrator shutdown 경계를 추가합니다.

---

## ESG-OBS-01 — Readiness/Liveness 등록과 Prometheus scrape 연결

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 21 — 운영 가시성과 준비 상태
**Related Threads:** 10, 11, 15, 19, 20

**Repository Evidence**

* Prometheus registry와 export가 활성화됩니다.
* Custom dependency health indicator는 DB를 조회하여 paused/exhausted revision과 unpublished outbox 수를 반환하고, DB query 실패 시 `DOWN`을 반환합니다.
* Readiness group은 `readinessState`와 `settlementDependencies`를 포함합니다.
* `health`, `info`, `prometheus` endpoint가 노출됩니다.

**Required External Step**

실제 load balancer 또는 orchestrator에 liveness/readiness probe를 등록하고, Prometheus 또는 호환 수집기에 scrape target을 등록해야 합니다. Runtime port가 probe·scraper에서 도달 가능해야 합니다.

현재 custom readiness가 직접 검증하는 외부 dependency는 DB입니다. Kafka broker나 Wallet HTTP 도달성을 readiness가 보장한다고 해석해서는 안 됩니다.

**실제 수행 여부 확인 가능성:** 확인 불가. Probe registration, scrape target, 수집 성공, alert rule 및 dashboard는 Git에 없습니다.

**Documentation Action:** Thread 21 supplement에 actuator 제공과 외부 등록을 구분하고, readiness가 실제로 검증하는 범위를 기록합니다.

---

## ESG-CI-01 — Shared Protocol 공급과 CI 실행환경 활성화

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 22 — 실행 패키징과 이력 무결성
**Related Threads:** 3, 8, 21, NT-23

**Repository Evidence**

* CI는 Java 17을 준비하고, 고정 SHA의 shared protocol을 별도 path에 checkout한 뒤 격리된 Maven local repository에 `install`하고 Settlement의 `clean verify`를 수행하도록 작성되어 있습니다.
* Full history checkout 후 archive-history guard가 실행됩니다.
* 검증은 PostgreSQL Testcontainers를 사용하며 README는 Docker가 필요하다고 명시합니다.
* Workflow의 push filter는 `settlement-service`이지만 현재 원격 branch는 `main`뿐입니다. Pull request와 수동 실행 trigger는 별도로 존재합니다.
* "Checkout fixed shared protocol" 단계에는 `ref`와 `path`만 있고 source `repository`가 명시되어 있지 않습니다. 따라서 이 workflow text만으로는 외부 shared-protocol source 위치를 식별할 수 없습니다.

**Required External Step**

CI platform에서 workflow 실행을 허용하고 다음 자원을 제공해야 합니다.

* Java 17
* Docker/Testcontainers 실행 환경
* 전체 Git history
* 고정 shared-protocol source 및 해당 commit에 대한 checkout 권한
* `com.sportsbook:shared-protocol:1.0.0`을 격리 Maven repository에 설치할 수 있는 build environment
* 실제 사용하는 branch와 일치하는 trigger 또는 PR/수동 실행 절차

**실제 수행 여부 확인 가능성:** 확인 불가. Workflow enablement, 과거 run, runner Docker 상태, dependency checkout 성공, build 결과는 Git만으로 확인할 수 없습니다.

**Documentation Action:** Thread 22 supplement에 CI 외부 prerequisite, shared-protocol source 식별 경계, `main` push trigger 불일치, 실행 여부 비관찰을 기록합니다.

---

## ESG-DEPLOY-01 — Boot JAR 배포와 런타임 활성화

**Classification:** `EXISTING_THREAD`
**Primary Owner:** Thread 22 — 실행 패키징과 이력 무결성
**Related Threads:** 3, 8, 18, 19, 20, 21, NT-23

**Repository Evidence**

* Spring Boot Maven plugin이 추가되어 executable Boot JAR을 생성합니다.
* Project version은 `1.0.0`으로 release되었습니다.
* Runtime contract는 Java 17, 기본 HTTP/Actuator port 8084, PostgreSQL, Kafka 및 Wallet 환경설정을 요구합니다.
* 최종 root에는 Dockerfile, Compose, Kubernetes, Helm 또는 기타 deployment manifest가 없고 Maven/source/configuration만 있습니다.

**Required External Step**

`1.0.0` Boot JAR을 빌드한 뒤 선택한 artifact store 또는 deployment platform으로 전달하고 Java 17 runtime에서 기동해야 합니다. HTTP port를 노출하고, DB·Kafka·Wallet·secret·worker 설정을 해당 환경에 주입해야 합니다. 배포 중에는 graceful shutdown signal과 종료 유예가 전달되어야 합니다.

**실제 수행 여부 확인 가능성:** 확인 불가. JAR의 실제 생성·게시, container image, host, staging/production environment, route, replica 수 및 deployment 시점은 Git에 없습니다.

**Documentation Action:** Thread 22 supplement에 "패키징 완료"와 "실제 artifact 배포·runtime activation"을 분리한 실행 checklist를 추가합니다.

---

# Part II — Existing Thread Supplement Packets

## Packet ET-03

### Thread Identity

* **Existing Thread 3**
* **한국어:** 원시 Kafka 경계와 정확한 DLT 복구
* **English:** Raw Kafka Boundary and Exact DLT Recovery
* **Gaps:** ESG-KAFKA-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거 |
| --- | --- | --- |
| `5a9902c3524f6b78ce4601a1f32ed7662b62e3ff` | `feat(topics): bind settlement topic inventory` | `SettlementTopics.java`; 여섯 토픽의 고유성, 기본명, `<source>.DLT` 계산.  |
| `c0a9cbb4c2583f9f02443089dbeffc80a10a9b06` | `feat(kafka): configure bounded raw DLT producer`  | `RawKafkaProducerConfiguration.java`; byte producer, idempotence, `acks=all`, bounded timeout. |
| `355f83bf773707d1a8eb591151533b6c65cc6c5f` | `feat(kafka): preserve raw records in exact DLT partitions` | `ExactDeadLetterRecoverer.java`; source partition을 동일 번호의 `.DLT` partition으로 재발행.  |

최종 설정의 핵심은 다음과 같습니다.

```yaml
group-id: settlement-service
auto-offset-reset: earliest
enable-auto-commit: false
isolation-level: read_committed
allow.auto.create.topics: false
ack-mode: manual_immediate
```

### External Development Steps

1. Kafka cluster 또는 broker endpoint를 준비하고 Spring Kafka bootstrap 설정을 주입합니다.
2. 여섯 기본 토픽을 생성합니다.
3. 세 입력 토픽 각각에 대문자 `.DLT` companion을 생성합니다.
4. Source와 DLT의 partition coverage를 일치시킵니다.
5. Custom topic override를 사용한다면 producer·consumer·DLT 설정을 같은 배포 단위에서 정렬합니다.
6. 첫 runtime 활성화 후 만들어지는 `settlement-service` consumer-group offset을 운영 대상 외부 상태로 관리합니다.

### Code Connection

* 입력 listener가 base source topic을 구독합니다.
* Permanent record failure는 exact DLT producer로 전달됩니다.
* DLT send가 성공해야 source acknowledgment가 진행될 수 있습니다.
* Outbox publisher는 세 output topic으로 terminal/revision event를 발행합니다.

### Evidence Boundary

| 구분 | 내용   |
| --- | --- |
| **Directly observed in repository** | 토픽 이름, DLT naming, 동일 partition 발행, auto-create 비활성화, consumer group과 ack 정책 |
| **Required/inferred from repository**  | Broker connection, 9개 토픽 사전 생성, partition 호환성, runtime consumer-group state  |
| **Actual execution not observable from Git** | Cluster 주소, 실제 토픽/partition, replication, ACL, offset 값, DLT record 존재 여부 |

### Ordering

**Conceptual execution order**

1. Kafka cluster 준비
2. 토픽명과 partition 정책 확정
3. 여섯 base topic 및 세 DLT 생성
4. Bootstrap/topic override 주입
5. Settlement runtime 기동
6. Consumer-group과 DLT publication 상태를 외부 도구로 확인

---

## Packet ET-08

### Thread Identity

* **Existing Thread 8**
* **한국어:** Wallet HTTP 계약과 작업 증거 검증
* **English:** Wallet HTTP Contract and Operation Proof Validation
* **Gaps:** ESG-WALLET-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거  |
| --- | --- | --- |
| `bdab6434a4fe0663e29e77c266dfdc5b71345b94` | `feat(wallet): submit authorized credit movements` | `WalletClient.java`, `WalletEndpointProperties.java`; HTTP(S) root origin과 Wallet credit 연결. |
| `d8c4a9b1b42998b2190ff815009c169980ffe5aa` | `feat(wallet): bound dependency timeouts` | `WalletHttpProperties.java`; connect/read timeout을 `(0, 5s]`로 제한. |

최종 client는 다음 API family를 동일 origin에 연결합니다.

* `/internal/v1/wallet/transactions/credit`
* `/internal/v1/wallet/transactions/forfeit`
* `/internal/v1/wallet/transactions/adjustment`
* `/internal/v1/wallet/transactions/adjustment/{revisionId}`

### External Development Steps

1. 해당 API contract를 제공하는 Wallet runtime을 배포합니다.
2. Settlement에서 도달할 수 있는 network origin을 확정합니다.
3. `SETTLEMENT_WALLET_BASE_URL`을 root origin 형식으로 주입합니다.
4. Network 특성과 recovery cadence를 고려해 허용 범위 안에서 timeout을 선택합니다.
5. ESG-AUTH-01에서 준비한 대응 Wallet credential을 양쪽 runtime에 연결합니다.
6. 배포 전후 실제 HTTP status와 proof contract를 확인합니다.

### Code Connection

Wallet origin이 잘못되거나 도달할 수 없으면 base settlement와 revision execution이 durable plan을 남긴 상태에서 dependency failure 또는 recovery 대상으로 전환됩니다. HTTP 연결 성공만으로 충분하지 않고 user, amount, reason, operation group 등의 proof도 코드 검증을 통과해야 합니다.

### Evidence Boundary

| 구분 | 내용  |
| --- | --- |
| **Directly observed in repository** | 허용 origin 형식, endpoint path, timeout 범위, request/response contract |
| **Required/inferred from repository**  | Wallet runtime 배포, service discovery 또는 고정 origin, network route   |
| **Actual execution not observable from Git** | 실제 URL, DNS, firewall, TLS, Wallet version, endpoint availability  |

### Ordering

**Conceptual execution order**

1. Wallet API 배포
2. Wallet-side matching credential 주입
3. Settlement-side base URL·credential·timeout 주입
4. Network 및 contract 확인
5. Settlement worker 활성화

---

## Packet ET-18

### Thread Identity

* **Existing Thread 18**
* **한국어:** 내부 호출 인증과 자격증명 분리
* **English:** Internal Authentication and Credential Isolation
* **Gaps:** ESG-AUTH-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거 |
| --- | --- | --- |
| `5f476259426be1932711eff9dddc41946c4d997a` | `feat(admin): validate operator credentials` | `AdminCredentials.java`, `application.yml`; 최소 32자 admin key와 caller/header 계약. |
| `18dbc954e3037f86afcdf4599adf6d354d231a64` | `feat(wallet): write exact authentication headers` | `WalletAuthenticationHeaders.java`; Settlement→Wallet의 두 인증 header. |
| `b64b73288f75468a75c54c5890e7941e53e52524` | `fix(security): reject reused settlement credentials` | `SettlementCredentialPolicy.java`; 두 값 재사용 시 startup failure. |

### External Development Steps

1. Admin control-plane 비밀 생성
2. Settlement-to-Wallet 비밀을 별도로 생성
3. 두 비밀이 32자 이상이고 서로 다름을 확인
4. Admin 비밀을 Settlement와 Admin API caller에 일치하게 주입
5. Wallet 비밀을 Settlement와 Wallet runtime에 일치하게 주입
6. 변경 시 상대 서비스와의 인증 불일치 시간을 최소화하는 배포 순서 결정

### Code Connection

* Admin API는 `X-Service-Name`과 `X-API-Key`를 검증합니다.
* Wallet client는 `X-Internal-Service`와 `X-Internal-Api-Key`를 모든 요청에 부착합니다.
* 한쪽 값이 누락·짧음·재사용되면 startup 또는 요청 인증이 실패합니다.

### Evidence Boundary

| 구분 | 내용 |
| --- | --- |
| **Directly observed in repository** | 변수 이름, 최소 길이, 값 분리, caller/header 계약, redacted rendering |
| **Required/inferred from repository**  | 두 비밀 생성, 상대 서비스의 matching 값 주입, 변경 시 양쪽 정렬   |
| **Actual execution not observable from Git** | 비밀 값, secret manager, CI environment, 생성·변경·폐기 시점  |

### Ordering

**Conceptual execution order**

1. 두 trust boundary별 비밀 생성
2. 대상 secret store에 저장
3. Wallet과 Admin caller에 matching 값 배치
4. Settlement에 두 값을 분리 주입
5. Startup validation과 실제 인증 호출 확인

---

## Packet ET-19

### Thread Identity

* **Existing Thread 19**
* **한국어:** 데이터베이스 시각 기반 소유권 펜싱
* **English:** Database-Time Ownership Fencing
* **Gaps:** ESG-TIME-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거   |
| --- | --- | --- |
| `de4b6b08b3799744c40467bbf2b927dfb7911d6b` | `feat(persistence): expose transaction database time` | `DatabaseTimeSource.java`; `select current_timestamp`를 `Instant`로 반환. |
| `f9beb189faf432b33b79763e94014583518ea33a` | `feat(settlement): claim attempts with database time` | `SettlementAttemptRepository.java`; lease 및 row timestamp를 DB에서 계산.   |

관련 SQL의 핵심 형태는 다음과 같습니다.

```sql
current_timestamp + (? * interval '1 millisecond')
current_timestamp
```

### External Development Steps

1. 대상 PostgreSQL 또는 managed DB의 clock 동작을 신뢰 가능한 상태로 운영합니다.
2. DB platform의 시간 동기화와 clock correction 정책을 확인합니다.
3. Runtime 활성화 전에 `current_timestamp`가 기대하는 실제 시각 범위인지 검증합니다.
4. DB failover나 host 교체가 있을 때 새로운 primary의 clock도 같은 신뢰 전제를 만족시켜야 합니다.

마지막 항목의 구체적인 failover 절차는 repository에 정의되어 있지 않으므로, 특정 방식을 요구하지는 않습니다.

### Code Connection

Database time은 단순 audit timestamp가 아니라 lease ownership, stale-owner 판정, retry scheduling 및 correction chronology의 입력입니다. 따라서 DB clock은 코드 밖에 존재하지만 알고리즘의 일부인 외부 상태입니다.

### Evidence Boundary

| 구분 | 내용  |
| --- | --- |
| **Directly observed in repository** | `current_timestamp` 조회 및 lease/timestamp 계산   |
| **Required/inferred from repository**  | 신뢰 가능한 DB wall clock, 운영 플랫폼의 시간 동기화 |
| **Actual execution not observable from Git** | 실제 DB 시각, drift, 동기화 방식, failover 당시 clock 상태 |

### Ordering

**Conceptual execution order**

1. PostgreSQL instance 준비
2. DB clock 신뢰성 확인
3. Migration 적용
4. Worker 활성화
5. Lease/recovery 동작 중 database time 이상 여부 운영 확인

---

## Packet ET-20

### Thread Identity

* **Existing Thread 20**
* **한국어:** 작업자 격리와 런타임 제어
* **English:** Worker Isolation and Runtime Control
* **Gaps:** ESG-WORKER-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거  |
| --- | --- | --- |
| `b8ed2d413d099e023dfef58e3bf0b180e3a6e061` | `feat(config): make workers runtime-switchable` | `SettlementWorkerConfiguration.java`; property missing 시 worker 기본 활성화. |
| `399a5fb95473f494468dc3fc3a9446f329623c98` | `feat(workers): bound scheduler shutdown` | 종료 후 delayed/periodic task 재실행 방지.  |

최종 configuration은 다섯 scheduler를 격리하며 각 executor가 최대 10초 동안 종료를 기다립니다.

### External Development Steps

1. Deployment unit별 worker role을 결정합니다.
2. 명시적인 role 분리가 필요하면 `SETTLEMENT_WORKERS_ENABLED`를 주입합니다.
3. 전체 배포 중 적어도 하나의 instance가 recovery와 outbox publication을 실행하도록 합니다.
4. Rolling shutdown 중 새 instance가 준비되기 전에 모든 worker를 동시에 제거하지 않도록 배포 정책을 정합니다.
5. Platform의 강제 kill timeout이 bounded graceful shutdown보다 먼저 끝나지 않도록 합니다.

### Code Connection

Worker가 모두 꺼지면 listener가 즉시 처리한 경로와 별개로 stale attempt recovery, correction catch-up, revision recovery 및 outbox publication이 진행되지 않습니다. 반대로 여러 instance에서 worker를 켜는 것은 DB lease와 `SKIP LOCKED`로 조정되므로 단일 worker instance만을 코드가 요구하지는 않습니다.

### Evidence Boundary

| 구분 | 내용   |
| --- | --- |
| **Directly observed in repository** | 다섯 scheduler, 기본 활성화, pool size, shutdown policy  |
| **Required/inferred from repository**  | 배포별 role 선택, worker coverage, termination grace   |
| **Actual execution not observable from Git** | Replica 수, 실제 enablement 값, rolling strategy, kill timeout |

### Ordering

**Conceptual execution order**

1. Instance role 설계
2. Worker enablement 및 cadence/lease 설정 주입
3. 종료 유예 구성
4. Runtime 배포
5. Outbox와 recovery backlog가 실제로 감소하는지 확인

---

## Packet ET-21

### Thread Identity

* **Existing Thread 21**
* **한국어:** 운영 가시성과 준비 상태
* **English:** Operational Observability and Readiness
* **Gaps:** ESG-OBS-01

### Repository Evidence

| Commit | Subject | 관련 파일 및 최소 증거   |
| --- | --- | --- |
| `0ec03233e4407d853a5fad43d7b66ee4c6dd0501` | `build(observability): enable Prometheus registry`   | Prometheus registry dependency와 export 설정. |
| `00e249ffff72debf2aabbeeeb612e0d51c87c18d` | `feat(health): report settlement dependencies` | DB 및 durable backlog health indicator.  |
| `494f7413af4d1f526c456b86baea077a13f350c5` | `feat(health): include settlement readiness details` | Readiness group에 custom indicator 연결.   |

### External Development Steps

1. Runtime port에 대한 liveness/readiness probe를 deployment platform에 등록합니다.
2. Readiness failure가 traffic 제거로 연결되도록 설정합니다.
3. Prometheus endpoint를 scrape target에 등록합니다.
4. Network policy가 probe와 scrape traffic을 허용하도록 합니다.
5. DB backlog metric의 수집 여부를 확인합니다.

Repository는 alert threshold나 dashboard를 정의하지 않으므로 이를 필수 단계로 추가하지 않습니다.

### Code Connection

* Readiness indicator는 DB query 성공 여부와 durable backlog 세부정보를 제공합니다.
* Prometheus endpoint는 application이 제공하지만 수집기는 application 내부에 없습니다.
* Kafka와 Wallet 상태는 이 readiness indicator의 직접 검사 대상이 아닙니다.

### Evidence Boundary

| 구분 | 내용   |
| --- | --- |
| **Directly observed in repository** | Actuator exposure, Prometheus export, DB health query, readiness group |
| **Required/inferred from repository**  | Probe와 scrape의 외부 등록, runtime port 도달성   |
| **Actual execution not observable from Git** | Probe 설정, scrape 성공, metric retention, alerts, dashboards  |

### Ordering

**Conceptual execution order**

1. Application endpoint 설정 확인
2. Runtime service/port 노출
3. Liveness/readiness 등록
4. Prometheus scrape 등록
5. 실제 응답과 metric 수집 확인

---

## Packet ET-22

### Thread Identity

* **Existing Thread 22**
* **한국어:** 실행 패키징과 이력 무결성
* **English:** Executable Packaging and History Integrity
* **Gaps:** ESG-CI-01, ESG-DEPLOY-01

### Repository Evidence

| Commit | Subject  | 관련 파일 및 최소 증거 |
| --- | --- | --- |
| `00fe5c784b4a21627e751f8241b84662ff42e81b` | `ci(build): verify settlement on Java 17` | Java 17, fixed shared-protocol checkout/install, isolated Maven repo, `clean verify`. |
| `531a2910d78a1b5b2d8338eab4bd1248d67fff3d` | `ci(history): guard settlement archive history` | `fetch-depth: 0`, history script 실행.  |
| `ff981a7b22a1a8ca87a9634a8e069adb0e51823a` | `fix(build): package settlement as a Boot JAR`  | Spring Boot Maven plugin 추가. |
| `599df4606c9cb1621413767351497f28f22e32ee` | `build(release): release settlement 1.0.0`   | Maven version `1.0.0`. |

### External Development Steps

#### ESG-CI-01

1. CI workflow execution을 repository/platform에서 허용합니다.
2. Java 17과 Docker를 갖춘 runner를 제공합니다.
3. Full history checkout을 유지합니다.
4. 실제 shared-protocol source repository와 고정 commit 접근을 구성합니다.
5. Protocol artifact를 isolated Maven repository에 설치합니다.
6. 현재 `main` branch와 push filter의 불일치를 해소하거나 PR/수동 trigger를 공식 실행 경로로 정합니다.

#### ESG-DEPLOY-01

1. Boot JAR을 생성합니다.
2. Artifact store 또는 runtime host로 전달합니다.
3. Java 17 runtime을 준비합니다.
4. ESG-DB/KAFKA/AUTH/WALLET/TIME/WORKER/OBS 설정을 주입합니다.
5. HTTP/Actuator port를 외부 service 또는 route에 연결합니다.
6. Graceful shutdown을 지원하는 방식으로 process를 기동합니다.

### Code Connection

CI는 코드의 재현 가능한 검증과 executable artifact 생성까지 담당합니다. 실제 artifact publication과 process deployment는 정의되어 있지 않습니다. 또한 shared-protocol dependency는 일반 remote repository에서 자동 해결되도록 구성되지 않고, workflow가 먼저 local Maven repository에 설치하는 구조입니다.

### Evidence Boundary

| 구분 | 내용  |
| --- | --- |
| **Directly observed in repository** | Workflow, Java version, fixed ref, Docker 기반 test dependency, Boot JAR plugin, release version   |
| **Required/inferred from repository**  | CI enablement, runner resources, dependency source access, artifact 전달, runtime 기동   |
| **Actual execution not observable from Git** | Workflow run 결과, JAR 생성·게시, deployment environment, process 상태  |
| **Repository under-specification**  | Shared-protocol checkout source repository가 workflow에 명시되지 않음; 현재 push filter가 유일한 remote branch인 `main`과 다름 |

### Ordering

**Conceptual execution order**

1. Shared-protocol source와 fixed commit 확인
2. CI runner에서 protocol install
3. Full-history guard와 `clean verify`
4. Boot JAR 생성
5. Artifact 게시 또는 전달
6. Runtime 환경변수·연결정보 주입
7. Process 배포
8. Readiness 확인 후 traffic 활성화

---

# Part III — Proposed New Thread Packets

## Proposed Thread NT-23

### Thread Identity

* **Proposed New Thread:** `NT-23`
* **한국어 제목:** PostgreSQL 스키마 프로비저닝과 Flyway 진화 수명주기
* **English title:** PostgreSQL Schema Provisioning and Flyway Evolution Lifecycle
* **Gaps:** ESG-DB-01

### NEW_THREAD 판정 근거

이 관점은 단순히 여러 Thread가 같은 datasource 환경변수를 사용한다는 이유로 제안된 것이 아닙니다.

1. **독립 상태가 존재합니다.**
   Target PostgreSQL의 database/schema, role grant, 실제 table/index/function/trigger, Flyway 적용 이력은 Git source와 별개로 수명을 가집니다.

2. **여러 단계가 연결됩니다.**
   DB 생성 → credential 주입 → Flyway 적용 → validation → JPA schema validation → 이후 전진 migration이라는 하나의 실행 흐름이 있습니다.

3. **독립적인 실패·복구 조건이 있습니다.**
   Connection failure, DDL 권한 부족, migration failure, checksum mismatch, pending migration, entity/schema mismatch가 application startup을 막을 수 있습니다.

4. **여러 기존 Thread를 관통합니다.**
   V1 read model, V3 outbox, V9 revision plan, V10 admin audit 등 서로 다른 Thread의 상태가 하나의 Flyway history에 결합됩니다.

5. **대표 commit을 선정할 수 있습니다.**
   Migration 생성 commit과 PostgreSQL bootstrap 검증 commit이 모두 존재합니다.

Thread 19는 database time과 lease fencing을 소유하지만 schema version lifecycle을 소유하지 않습니다. Thread 22는 executable packaging과 Git history integrity를 소유하지만 target database의 외부 상태 전환을 소유하지 않습니다. 따라서 어느 기존 Thread의 부수 단계만으로 귀속하기 어렵습니다.

### Representative Commits

| Commit | Subject | 이 Thread에서 중요한 이유 |
| --- | --- | --- |
| `9dd3cd7d91e8957ca36e4e64d903112679799a86` | `build(flyway): add V1 bet read model`   | JPA·Flyway·PostgreSQL dependency와 최초 schema를 함께 도입하여 DB lifecycle의 출발점을 만듭니다. 기존 Thread 1 소속 commit입니다.  |
| `4d1f54790ccf14478263395e03b6ff40154d92f3` | `build(flyway): add V3 transactional outbox`   | Domain state뿐 아니라 broker publication state도 같은 DB migration history에 들어감을 보여 줍니다. 기존 Thread 11 소속 commit입니다.   |
| `cefbc14d9f0e369162bf1e6d50af48fc86f8b884` | `build(flyway): add V9 revision plans`   | Lease, retry, Wallet proof, terminal state를 다수 constraint로 보존하는 correction schema를 추가합니다. 기존 Thread 14 소속 commit입니다. |
| `b93b996c58bfd8ecf9d507b4759efe536a05ff23` | `build(flyway): add V10 admin actions`   | PostgreSQL function과 trigger까지 migration 실행 권한에 포함되며 admin audit를 append-only로 만듭니다. 기존 Thread 16 소속 commit입니다.   |
| `cde6128a36ac85345d5dfb77d82e1e9f1278ea26` | `test(integration): configure PostgreSQL harness` | 실제 PostgreSQL 16 container를 만들고 datasource 속성을 외부 주입하는 검증 경계를 제공합니다. 기존 Index 미사용 commit입니다. |
| `3bd69585dc7664065cad11bc82d049c24725e0b1` | `test(integration): verify PostgreSQL schema bootstrap` | Flyway validation, pending 없음, JPA validation을 하나의 bootstrap 완료 조건으로 확인합니다. 기존 Index 미사용 commit입니다.   |

### 관련 Diff Packet

### 1. 최초 DB 기술 경계

`9dd3cd7...`은 build dependency와 V1 schema를 동시에 추가합니다.

```diff
+ spring-boot-starter-data-jpa
+ flyway-core
+ postgresql

+ CREATE TABLE bet (...)
+ CREATE TABLE bet_selection (...)
+ CREATE INDEX ix_bet_selection_event ...
+ CREATE INDEX ix_bet_pending ...
```

이는 단순 SQL 파일 추가가 아니라 application startup이 PostgreSQL/Flyway 외부 상태에 의존하기 시작한 지점입니다.

### 2. 여러 subsystem을 한 schema history에 결합

`4d1f547...`은 terminal state와 Kafka publication 사이의 내구 경계를 `outbox_event` table로 추가합니다.

```diff
+ CREATE TABLE outbox_event (...)
+ CREATE INDEX ix_outbox_unpublished
+  ON outbox_event (created_at)
+  WHERE published_at IS NULL;
```

`cefbc14...`은 revision planning과 Wallet proof, lease, retry schedule을 하나의 DB schema로 고정합니다.

```diff
+ CREATE TABLE settlement_revision (...)
+ CONSTRAINT ck_settlement_revision_lease ...
+ CONSTRAINT ck_settlement_revision_wallet_state ...
+ CONSTRAINT ck_settlement_revision_schedule ...
+ CREATE INDEX ix_settlement_revision_recovery ...
```

### 3. PostgreSQL-specific DDL 권한

`b93b996...`은 table/index뿐 아니라 PL/pgSQL function과 trigger를 생성합니다.

```diff
+ CREATE FUNCTION reject_settlement_admin_action_mutation()
+ RETURNS TRIGGER
+ LANGUAGE plpgsql ...

+ CREATE TRIGGER settlement_admin_action_append_only
+ BEFORE UPDATE OR DELETE ON settlement_admin_action ...
```

따라서 migration executor는 적어도 이 DDL을 수행할 수 있어야 합니다. 실제 role 이름이나 grant 방식은 repository에 없습니다.

### 4. 외부 datasource 주입과 bootstrap 검증

`cde6128...`의 test harness는 다음 외부 연결정보를 runtime property로 넣습니다.

```java
registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
registry.add("spring.datasource.username", POSTGRES::getUsername);
registry.add("spring.datasource.password", POSTGRES::getPassword);
```

또한 `postgres:16-alpine` container를 실제 DB engine으로 사용합니다.

`3bd69585...`는 bootstrap 완료 조건을 다음과 같이 고정합니다.

```java
flyway.validateWithResult().validationSuccessful
flyway.info().pending().isEmpty()
spring.jpa.hibernate.ddl-auto == validate
```

### Final-State Configuration and Source Excerpts

최종 `application.yml`은 Hibernate가 schema를 생성하지 않고 검증만 하도록 설정합니다.

```yaml
spring:
  jpa:
 hibernate:
   ddl-auto: validate
```

Datasource URL, username, password는 이 파일에 고정되어 있지 않습니다. README는 표준 Spring Boot datasource 설정을 사용하고 Flyway 적용 후 Hibernate validation을 수행한다고 설명합니다.

최종 migration inventory는 다음과 같습니다.

| Version | 파일 | 상태 관점 |
| --- | --- | --- |
| V1   | `V1__bet_read_model.sql`   | Bet snapshot 및 selection   |
| V3   | `V3__outbox.sql`  | Transactional outbox |
| V4   | `V4__match_result.sql`  | Accepted match result   |
| V5   | `V5__settlement_attempt.sql`  | Base settlement attempt |
| V6   | `V6__event_lifecycle.sql`  | Lifecycle observation/tombstone  |
| V7   | `V7__result_candidate.sql` | Result candidate  |
| V8   | `V8__source_revision.sql`  | Source revision identity   |
| V9   | `V9__settlement_revision.sql` | Immutable revision plan 및 Wallet evidence |
| V10  | `V10__admin_action.sql` | Append-only operator action   |

V2 파일이 최종 트리에 없다는 사실은 직접 관찰되지만, repository가 별도의 V2 외부 작업을 요구한다는 근거는 없습니다. 따라서 이를 별도 Gap이나 migration 실행 사실로 해석하지 않습니다.

Migration immutability test는 V1, V3, V4, V5의 Git blob ID와 핵심 constraint를 고정합니다.  README는 released migration을 수정하지 않고 향후 변경을 V11 이상으로 추가하도록 요구합니다.

### External Development Steps

1. PostgreSQL-compatible target environment를 선택합니다.
2. Database 또는 schema를 생성합니다.
3. Runtime/migration credential을 생성하고 필요한 권한을 부여합니다.
4. `spring.datasource.url`, username, password에 해당하는 설정을 runtime에 주입합니다.
5. Flyway가 V1 및 V3–V10을 적용하도록 실행합니다.
6. Flyway validation 성공과 pending migration 없음 상태를 확인합니다.
7. JPA `ddl-auto=validate` startup을 통과시킵니다.
8. 배포 후 Flyway history와 released migration checksum을 보존합니다.
9. 미래 schema 변경은 V11 이상으로 추가합니다.
10. Migration 실패 시 rollback SQL을 추정하지 말고, 실제 DB 상태를 먼저 조사한 뒤 repository가 허용하는 전진 migration으로 복구 계획을 수립합니다.

마지막 단계의 구체적인 복구 명령이나 데이터 복원 절차는 repository에 존재하지 않습니다.

### Code Connection

| External Step | 연결되는 코드/행동  |
| --- | --- |
| DB/schema 생성  | JPA repositories, JDBC repositories, health query의 실행 대상 |
| Datasource 주입 | Spring Boot datasource, Flyway, JPA가 공통 연결 사용   |
| Migration 실행  | 모든 settlement state table/index/constraint/function/trigger 생성 |
| Validation | `PostgresSchemaIntegrationTest`, runtime `ddl-auto=validate`   |
| History/checksum 보존 | Migration blob test와 append-only V11 규칙   |
| DDL 권한  | V1–V10의 table/index/function/trigger 생성   |
| DB availability  | Readiness indicator와 worker claim/recovery   |

### Evidence Boundary

### Directly observed in repository

* Flyway와 PostgreSQL dependency
* V1 및 V3–V10 SQL
* PostgreSQL 16 integration harness
* Datasource property 주입 방식
* Flyway validation과 pending 확인
* JPA `validate`
* Released migration blob immutability
* V11 이상 전진 변경 규칙

### Required/inferred from repository

* 실제 database/schema 생성
* Connection credential 발급
* Migration executor의 DDL 권한
* Datasource runtime 주입
* Target DB에 대한 Flyway 실행
* Target DB의 Flyway history/checksum 보존
* Migration 완료 후 application 활성화

### Actual execution not observable from Git

* Database provider, host, port, database/schema 이름
* Username/password 또는 인증 방식
* Role grant
* Production/staging database 존재 여부
* 어떤 환경에 어떤 migration이 적용되었는지
* 실제 `flyway_schema_history` 내용
* Migration 실행 시점과 실행 주체
* 실패·복구 이력
* Backup 또는 restore 수행 여부

### Ordering

다음은 실제 과거 수행 이력이 아니라 **conceptual execution order**입니다.

1. PostgreSQL runtime 선택 및 생성
2. DB/schema와 credential/권한 구성
3. Datasource 설정 주입
4. Flyway migration 실행
5. Flyway validation 및 pending 확인
6. JPA schema validation
7. Application과 worker 활성화
8. Readiness 확인
9. 이후 변경을 V11 이상으로 누적
10. 외부 DB의 migration history와 checksum 보존

---

# Part IV — Project-Level External Steps

## 채택 항목 없음

이번 감사에서는 중요 외부 단계가 모두 특정 기존 Thread 또는 제안된 `NT-23`에 자연스럽게 귀속되었습니다. 별도의 `PROJECT_LEVEL_EXTERNAL_STEP`을 만들면 다음 내용이 기존 소유권과 중복되므로 채택하지 않았습니다.

| 후보  | 미채택 이유  |
| --- | --- |
| 일반적인 staging/production provisioning | Executable 배포 관점은 Thread 22, DB/Kafka/Wallet 자원은 각 소유 Thread로 귀속 가능 |
| Runtime seed 실행 | 이력의 `seed` 표현은 통합시험용 row 삽입이며 운영 seed/init artifact 근거가 없음 |
| 외부 cron 등록   | Worker는 Spring 내부 scheduler로 구현되어 Thread 20에 귀속됨  |
| Backup/restore  | Script, configuration, commit 또는 복구 절차 근거가 없음  |
| OAuth application/redirect URI | 관련 integration 근거가 없음  |
| Webhook 등록   | 관련 endpoint 또는 provider configuration 근거가 없음   |
| Object storage/bucket/IAM   | 관련 source/configuration 근거가 없음  |
| DNS/domain verification  | 구체적인 domain 또는 verification artifact가 없음 |
| TLS certificate 발급 | Wallet origin은 HTTP와 HTTPS를 모두 허용하며 TLS를 필수로 만드는 근거가 없음 |
| Docker/Kubernetes/Helm/Terraform provisioning | 최종 repository에 해당 deployment/IaC artifact가 없음. |

따라서 이번 보완의 최소 범위는 **기존 Thread supplement 7개, 그 안의 Gap 8개, 신규 Thread 1개**입니다. 실제 secret 값, 외부 endpoint 값, database connection 값 또는 이미 수행되었다고 확인할 수 없는 deployment/migration 실행은 어떠한 경우에도 복원하거나 과거 사실로 서술하지 않았습니다.
