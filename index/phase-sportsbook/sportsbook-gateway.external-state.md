# External-State Development Gap Audit — `sportsbook-gateway`

## 감사 범위와 판정 요약

Existing Development Thread의 번호·제목·commit 귀속은 첨부된 Index를 확정 구조로 사용했습니다. 기존 Thread를 재편하거나 기존 학습 문서를 평가하지 않았습니다. 

Git history는 최초 commit `d3f0ba958fdd6ae234233bfba043141375f2109d`부터 현재 HEAD까지 확인했습니다. API 순서상 117번째에 최초 commit이 존재하고 118번째부터는 비어 있으므로, 현재 `main` history는 총 117개 commit입니다.

감사 결과는 다음과 같습니다.

* **EXISTING_THREAD Gap: 13개**
* **NEW_THREAD: 0개**
* **PROJECT_LEVEL_EXTERNAL_STEP: 1개**
* Supplement 대상 기존 Thread: **1, 3, 4, 5, 6, 8, 10, 12, 16, 17, 18**

Database, migration, seed, object storage, OAuth provider 등록, webhook 등록, DNS, TLS certificate, scheduler, backup/restore에 대해서는 현재 및 과거 repository에서 프로젝트별 필요성을 입증하는 source/configuration artifact를 발견하지 못했습니다. 따라서 일반적인 운영 관행만을 근거로 Gap을 추가하지 않았습니다.

---

# Part I — Gap Index

## G-01 — Shared Protocol 빌드 입력과 CI Git Ref 공급

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 1 — Java 17 서비스 빌드, 품질 게이트, 1.0 릴리스
* **Related Threads:** Thread 10, Thread 11
* **Repository Evidence:** `pom.xml`은 `com.sportsbook:shared-protocol:1.0.0`을 빌드 의존성으로 요구합니다. CI는 동일 repository의 `shared-protocol` ref를 checkout한 뒤 임시 Maven repository에 설치하고 gateway를 검증하도록 구현됐습니다.
* **Current External-State Finding:** 현재 repository ref 목록에서 `shared-protocol`과 `gateway`에 일치하는 ref가 확인되지 않으며, 현재 branch 목록은 `main` 하나입니다. CI의 `push` trigger는 `gateway` branch만 대상으로 합니다.
* **Required External Step:** `shared-protocol:1.0.0`을 Maven이 해석할 수 있도록 외부 artifact repository, 사전 로컬 설치, 또는 유효한 Git ref 중 하나로 공급해야 합니다. 현재 CI 방식을 유지한다면 `shared-protocol` ref를 복구해야 하며, `main` push에서도 CI가 필요하다면 trigger/ref 구성을 맞춰야 합니다.
* **실제 수행 여부 확인 가능성:** 현재 ref 부재는 직접 확인할 수 있습니다. 과거 CI에서 dependency 설치가 성공했는지, 별도 Maven registry에 artifact가 게시됐는지는 Git source만으로 확인할 수 없습니다.
* **Documentation Action:** Thread 1에 "빌드 입력 공급 모델 및 CI ref preflight" 보충 절을 추가합니다.

## G-02 — Docker가 활성화된 품질 게이트 실행환경

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 1
* **Related Threads:** Thread 6, Thread 15
* **Repository Evidence:** CI는 gateway 검증 전에 `docker info`를 실행합니다. 전체 검증에는 Redis Testcontainer가 사용되며, JDK 17과 Docker daemon이 build prerequisite로 문서화됐습니다.
* **Required External Step:** 로컬 또는 CI runner에 Java 17과 실제 동작하는 Docker daemon을 제공하고, runner가 container image를 가져오고 Redis container를 시작할 수 있도록 해야 합니다.
* **실제 수행 여부 확인 가능성:** 요구사항과 CI 명령은 관찰됩니다. 특정 runner에서 Docker가 실제로 가동됐는지, `clean verify`가 성공했는지는 repository history만으로 확인되지 않습니다.
* **Documentation Action:** Thread 1에 "quality-gate runtime prerequisites" 체크리스트를 추가합니다.

## G-03 — RS256 키 쌍 생성·분배와 Gateway Public Key 주입

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 3 — RS256 JWT 검증과 정규형 사용자 신원 계약
* **Related Threads:** Thread 8, Thread 9
* **Repository Evidence:** commit `5af27b7e7c5f7b63b0030aa360d410d3a5a5bbee`는 RS256 decoder와 `GATEWAY_JWT_PUBLIC_KEY` 구성을 추가했습니다. Loader는 X.509 `PUBLIC KEY` PEM, 최소 2048-bit RSA key를 요구하며 누락·오형식이면 startup을 실패시킵니다.
* **Required External Step:** Gateway 밖에서 RSA 키 쌍을 생성하거나 기존 signing key를 선택하고, token 발급 측에는 private key를, gateway에는 대응 public key를 secret/configuration mechanism으로 주입해야 합니다. 키 변경 시 새 public key를 주입하고 process를 재시작해야 합니다.
* **실제 수행 여부 확인 가능성:** Gateway가 public key를 필요로 한다는 사실만 확인됩니다. 실제 issuer, private key 소재, key 값, 발급 시점, rotation history는 확인할 수 없습니다.
* **Documentation Action:** Thread 3에 "signing-key counterpart, public-key delivery, rotation boundary"를 추가합니다.

## G-04 — 실제 Downstream Endpoint와 Proxy Base URI 바인딩

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 4 — HTTP 메서드·경로 허용목록과 정확한 프록시 라우팅
* **Related Threads:** Thread 5, Thread 7
* **Repository Evidence:** commit `d61ef88550a4c178d4b600cfd7ad6af6bf7d8f8e`는 betting base URI를 외부 구성으로 받고, 이를 대상으로 허용된 요청을 `/internal/v1/bets` 계열로 rewrite하도록 구현했습니다. Base URI는 absolute HTTP/HTTPS URI이며 user-info, query, fragment, 추가 path를 허용하지 않습니다.
* **Required External Step:** 실제 betting, wallet, odds-feed service를 배치하고, gateway가 접근할 수 있는 base URI를 `GATEWAY_BETTING_URI`, `GATEWAY_WALLET_URI`, `GATEWAY_ODDS_FEED_URI`에 설정해야 합니다. Gateway 실행환경과 각 service 사이의 네트워크 접근도 성립해야 합니다.
* **실제 수행 여부 확인 가능성:** URI property와 routing behavior는 확인됩니다. 실제 service 주소, service discovery 방식, network policy, 배포된 service의 존재와 접근 가능성은 확인되지 않습니다.
* **Documentation Action:** Thread 4에 "route contract를 실제 service endpoint에 결합하는 단계"를 추가합니다.

## G-05 — Betting·Wallet 내부 API Credential 발급과 양측 등록

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 5 — 신뢰 헤더 차단과 다운스트림 신원·자격 증명 재구성
* **Related Threads:** Thread 4, Thread 18
* **Repository Evidence:** wallet credential 설정은 commit `f9a42f3f85a3873a52eacdc6b856bd9571ba508d`에서 추가됐고, 각 key는 최소 32자를 요구합니다. commit `cb86b248de9f5fb7f14fa025716f03cbe7ae8fd2`는 betting과 wallet key 재사용을 startup에서 거부합니다.
* **Required External Step:** 서로 다른 두 credential을 외부에서 발급하고, betting 및 wallet service가 각각 해당 credential을 gateway 전용 credential로 인식하도록 등록한 후, gateway에는 `GATEWAY_BETTING_API_KEY`와 `GATEWAY_WALLET_API_KEY`로 주입해야 합니다.
* **실제 수행 여부 확인 가능성:** 길이·분리 정책과 gateway 주입 지점은 확인됩니다. 실제 값, downstream 등록 여부, secret manager 종류, rotation 이력은 확인할 수 없습니다.
* **Documentation Action:** Thread 5에 "credential issuance → downstream registration → gateway injection → coordinated rotation" 절을 추가합니다.

## G-06 — Redis Runtime Resource와 연결 상태 구성

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 6 — 신뢰 프록시 판별과 Fail-Open 복구를 포함한 분산 요청 제한
* **Related Threads:** Thread 17
* **Repository Evidence:** commit `9ee929bf045dde8cbf0773a9de72928bc3909e0e`는 host, port, database, username, password, SSL 설정으로 bounded Redis client를 만듭니다. commit `1108ae51b21e16b244a81ff2b1bb96a784a4b9d3`는 Redis에 분산 token bucket을 생성하고 Redis failure 시 요청을 허용하는 fail-open 동작을 구현합니다.
* **Required External Step:** 접근 가능한 Redis resource를 생성하거나 선택하고 host/port/database를 지정해야 합니다. 선택한 Redis 배치가 인증 또는 TLS를 사용하면 대응 username/password/SSL 설정도 주입해야 합니다. 장애 후에는 연결 복구와 분산 limiting 재개 여부를 운영 측에서 확인해야 합니다.
* **실제 수행 여부 확인 가능성:** Redis connection model과 fail-open behavior는 확인됩니다. 실제 Redis instance, account, password, TLS certificate, 저장된 bucket 상태와 장애·복구 이력은 확인되지 않습니다.
* **Documentation Action:** Thread 6에 "Redis provisioning, connection injection, fail-open recovery verification"을 추가합니다.

## G-07 — 신뢰 Proxy Topology와 Forwarded Header 덮어쓰기 계약

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 6
* **Related Threads:** Thread 5
* **Repository Evidence:** commit `d066c2697d3e41694cb485c5061ebebe1414952a`는 직접 연결 peer가 configured CIDR에 속할 때만 `X-Forwarded-For`를 해석하고, 오른쪽에서 왼쪽으로 신뢰 proxy chain을 제거해 client IP를 선택합니다.
* **Required External Step:** Reverse proxy를 사용하는 배치에서는 gateway와 직접 연결되는 통제된 proxy CIDR만 `GATEWAY_TRUSTED_PROXY_CIDRS`에 등록해야 하며, 해당 proxy가 외부에서 들어온 forwarding header를 신뢰하지 않고 자신의 관찰값으로 덮어쓰도록 구성해야 합니다. Proxy가 없다면 CIDR 목록을 비워야 합니다.
* **실제 수행 여부 확인 가능성:** 신뢰 판정 algorithm은 확인됩니다. 실제 proxy topology, CIDR, header overwrite 설정과 client path는 확인되지 않습니다.
* **Documentation Action:** Thread 6에 Redis와 분리된 "network trust boundary provisioning" 보충 절을 둡니다.

## G-08 — Production WebSocket Origin 허용목록 구성

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 8 — STOMP 연결 인증과 명령·구독 허용목록
* **Related Threads:** Thread 3, Thread 9
* **Repository Evidence:** commit `c75cbbb47472f3ae103247a6e127fb6cf47daa9a`는 `/ws/v1/odds`, `/ws/v1/bets` endpoint에 runtime-configured origin pattern을 적용하며 기본값은 localhost 계열입니다.
* **Required External Step:** 실제 browser/client deployment의 origin을 확정하고 `GATEWAY_WS_ALLOWED_ORIGINS`에 필요한 origin만 구성한 뒤 gateway를 재시작해야 합니다.
* **실제 수행 여부 확인 가능성:** endpoint와 기본 pattern은 확인됩니다. 실제 production frontend origin 및 runtime 주입값은 확인되지 않습니다.
* **Documentation Action:** Thread 8에 "deployment-time origin policy binding"을 추가합니다.

## G-09 — Kafka Input/DLT Topic, Partition, Retention, 권한 Provisioning

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 10 — 원시 Kafka 레코드 소비와 재시도·DLT 복구
* **Related Threads:** Thread 11, Thread 12, Thread 16
* **Repository Evidence:**

  * `f42a78babcc660c88e4f77e1f12afadce11aa9fc`는 네 개의 distinct input topic을 정의합니다.
  * `4744b5ee72caa720323a435ee4168337cb64907c`는 각 input을 `<source>.DLT`의 동일 partition 번호로 매핑합니다.
  * `6f1addd36eba921ffb4e118a36cddb410c04cf4c`는 DLT producer에 `acks=all`, idempotence, bounded send wait와 failure propagation을 설정합니다.
* **Required External Step:** Kafka cluster와 bootstrap endpoint를 준비하고, 다음 네 input과 각각의 `.DLT` topic을 생성해야 합니다.

  * `odds.changed`
  * `bet.settled.v1`
  * `bet.voided.v1`
  * `bet.resolution.revised.v1`
  * 각 input에 대응하는 `<input>.DLT`

  각 DLT는 source와 동일한 partition 수를 가져야 하며 운영 문서가 요구하는 최소 7일 retention을 적용해야 합니다. Gateway consumer group이 input을 읽고 DLT producer가 DLT에 쓸 수 있는 권한도 필요합니다. 자동 topic 생성은 이 조건을 보장하는 수단으로 간주할 수 없습니다.
* **실제 수행 여부 확인 가능성:** topic naming 및 partition mapping 요구는 확인됩니다. 실제 cluster, topic 존재 여부, partition 수, retention, ACL, record와 consumer offset은 확인되지 않습니다.
* **Documentation Action:** Thread 10에 "Kafka resource provisioning contract"를 추가합니다.

## G-10 — DLT의 통제된 수동 Replay 실행

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 12 — DLT 수동 재처리를 위한 레코드 복원과 헤더 정화
* **Related Threads:** Thread 10, Thread 11
* **Repository Evidence:** commit `716679f240d0c40f86c3b9e3fa276198ab0d2d91`은 정확한 gateway DLT record만 source topic으로 복원하고 framework exception·DLT·deserialization header를 제거하며 원래 partition을 유지합니다. 이 factory는 record를 만들 뿐 실제 publish를 실행하지 않습니다.
* **Required External Step:** 운영자가 원인을 분석·수정한 후 DLT record를 읽고, `DltReplayRecordFactory`와 동등한 정화 규칙을 적용해 원래 source topic의 같은 partition으로 publish하고, 처리 결과를 확인해야 합니다.
* **실제 수행 여부 확인 가능성:** replay record 생성 규칙만 확인됩니다. 실제 DLT record, 운영자 판단, replay 도구, publish 수행, 승인·감사 기록은 확인되지 않습니다.
* **Documentation Action:** Thread 12에 "manual replay runbook and evidence boundary"를 추가합니다.

## G-11 — 정확히 한 개 Replica인 Deployment State 강제

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 16 — Kafka-to-STOMP 전달 파이프라인과 단일 복제본 불변 조건
* **Related Threads:** Thread 9, Thread 13, Thread 14
* **Repository Evidence:** WebSocket broker는 process-local simple broker이고 Kafka consumer group은 partition을 replica 사이에 나눕니다. 따라서 subscriber가 연결된 process와 event를 소비한 process가 달라지는 다중 replica 구성에서는 전달 누락이 발생할 수 있으며, architecture 문서는 deployment configuration이 replica count 1을 강제해야 한다고 명시합니다.
* **Required External Step:** 실제 deployment platform에서 gateway의 활성 replica 수를 1로 강제해야 합니다. 배포·재시작 과정도 둘 이상의 gateway가 동시에 Kafka를 소비하는 상태를 만들지 않도록 deployment semantics를 구성해야 합니다.
* **실제 수행 여부 확인 가능성:** 불변 조건과 이유는 확인됩니다. Deployment manifest, orchestration platform, 실제 replica count와 rollout history는 repository에 없습니다.
* **Documentation Action:** Thread 16에 "deployment enforcement and runtime verification"을 추가합니다.

## G-12 — Probe 등록, Prometheus Scrape, Dependency Alerting

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 17 — 외부 의존성을 제외한 상태 프로브와 운영 지표
* **Related Threads:** Thread 6, Thread 10, Thread 16
* **Repository Evidence:** commit `0a92ec595d11e4e39a5ebbca9f2cbed52787e288`는 `health`, `info`, `prometheus` endpoint와 dependency-independent liveness/readiness group을 구성하고 `gateway.ratelimit.fail.open` counter를 추가했습니다.
* **Required External Step:** Deployment platform에 liveness/readiness probe를 등록하고 Prometheus 또는 호환 collector가 `/actuator/prometheus`를 scrape하도록 해야 합니다. Redis fail-open counter를 경보에 연결하고, probe에서 제외된 Redis, Kafka, betting, wallet, odds-feed는 별도 external monitoring으로 감시해야 합니다.
* **실제 수행 여부 확인 가능성:** endpoint와 metric 생성은 확인됩니다. Scrape target, alert rule, threshold, dashboard, on-call routing과 실제 alert 발생은 확인되지 않습니다.
* **Documentation Action:** Thread 17에 "probe/metric을 운영 시스템에 연결하는 단계"를 추가합니다.

## G-13 — 구조화 JSON 표준출력의 외부 수집

* **Classification:** `EXISTING_THREAD`
* **Primary Owner:** Thread 18 — 비밀정보를 정화하는 구조화 JSON 로깅
* **Related Threads:** Thread 7, Thread 10, Thread 12
* **Repository Evidence:** commit `4eb3e8c0794e20f0c3d111f246408d03d407c123`은 redaction provider와 console JSON encoder를 추가했습니다. 현재 설정은 JSON을 stdout으로 내보내며 MDC에서는 `traceId`, `spanId`만 포함합니다.
* **Required External Step:** Runtime 또는 deployment platform에서 stdout을 수집해 운영 log sink로 전송해야 합니다. 수집 pipeline이 JSON 구조를 훼손하지 않고, 수집 후에도 credential pattern이 노출되지 않는지 확인해야 합니다.
* **실제 수행 여부 확인 가능성:** application의 출력 형식과 redaction logic은 확인됩니다. 실제 collector, sink, retention, 접근권한, 저장된 로그와 end-to-end redaction 결과는 확인되지 않습니다.
* **Documentation Action:** Thread 18에 "stdout collection boundary"를 추가합니다.

---

# Part II — Existing Thread Supplement Packets

## Packet E-01 — Thread 1

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 1
* **한국어 제목:** Java 17 서비스 빌드, 품질 게이트, 1.0 릴리스
* **English title:** Java 17 Service Build, Quality Gates, and 1.0 Release
* **Gaps:** G-01, G-02

### Repository Evidence

1. **`632c39c1c24909982ef449ac8f12cf3fb71158c6` — `build(maven): initialize Java 17 service`**

   * Java 17 Maven service baseline을 만듭니다. 이 commit은 전체 history의 초기 build 단계에 위치합니다.

2. **`9e58b1bf991a0419192eca79b6bf1ec34e81dbaf` — `ci(gateway): verify Java 17 builds`**

   * 관련 파일: `.github/workflows/ci.yml`
   * 관련 diff:

     * current gateway SHA checkout
     * `ref: shared-protocol` checkout
     * Temurin 17 설치
     * `docker info`
     * shared protocol의 `clean install`
     * gateway의 `clean verify`
   * CI 자체가 shared protocol source ref와 Docker daemon이라는 외부 상태를 전제로 합니다.

3. **`24817b28bccaa28c77a9c59c5aa9632da2fd6bee` — `build(release): release gateway 1.0.0`**

   * 관련 파일: `pom.xml`
   * 관련 diff: `1.0.0-SNAPSHOT`을 `1.0.0`으로 변경합니다.
   * 이 diff는 version 확정을 증명하지만 artifact registry 게시나 runtime 배포를 증명하지는 않습니다.

4. **Final-state configuration**

   * `pom.xml`은 `com.sportsbook:shared-protocol:1.0.0`을 필수 dependency로 선언합니다.
   * 현재 branch 목록은 `main` 하나이며, CI가 참조하는 `shared-protocol` 및 push 대상으로 지정한 `gateway` ref는 현재 ref lookup에서 확인되지 않습니다.

### External Development Steps

* Shared protocol 공급 방식을 하나로 확정합니다.

  * 유효한 `shared-protocol` Git ref 유지
  * 사전 `mvn install`
  * 또는 접근 가능한 Maven artifact repository에 `1.0.0` 게시
* CI trigger가 실제 기본 branch 정책과 일치하도록 Git ref state 또는 workflow를 맞춥니다.
* Java 17과 Docker daemon을 가진 runner를 준비합니다.
* 동일 Maven repository context에서 shared protocol을 먼저 설치한 뒤 gateway verification을 수행합니다.
* 검증 결과로 생성된 `gateway-1.0.0.jar`을 실제 배포 입력으로 전달합니다.

### Code Connection

* Shared protocol이 없으면 공통 Problem, Money, Avro event type을 compile/link할 수 없습니다.
* Docker가 없으면 Redis Testcontainer 기반 integration test가 실행될 수 없습니다.
* `1.0.0` version 변경만으로는 실행 가능한 artifact가 외부 registry나 server에 존재하게 되지 않습니다.

### Evidence Boundary

* **Directly observed in repository**

  * Java 17, Maven dependency, CI 명령, Docker preflight, release version
  * 현재 `shared-protocol`/`gateway` ref 부재
* **Required/inferred from repository**

  * Shared protocol artifact 또는 source ref의 실제 공급
  * Docker-enabled runner
  * CI ref 정책과 repository branch 정책의 정합화
* **Actual execution not observable from Git**

  * 과거 CI 성공 여부
  * 실제 artifact publication
  * 특정 runner의 Docker 상태
  * 실제 release artifact 전달·배포

### Ordering

**Conceptual execution order**

1. Shared protocol 공급 방식과 ref 정책 확정
2. CI runner에 Java 17 및 Docker 제공
3. Shared protocol `1.0.0` 설치 또는 resolution 확인
4. Gateway `clean verify`
5. `gateway-1.0.0.jar` package 생성
6. Project-level runtime deployment로 전달

---

## Packet E-03 — Thread 3

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 3
* **한국어 제목:** RS256 JWT 검증과 정규형 사용자 신원 계약
* **English title:** RS256 JWT Verification and Canonical User Identity Contract
* **Gaps:** G-03

### Repository Evidence

* **`5af27b7e7c5f7b63b0030aa360d410d3a5a5bbee` — `feat(security): verify RS256 user tokens`**

  * 관련 파일:

    * `JwtDecoderConfiguration.java`
    * `JwtSecurityProperties.java`
    * `RsaPublicKeyLoader.java`
    * `application.yml`
  * 관련 diff:

    * `NimbusJwtDecoder.withPublicKey(...)`
    * RS256 고정
    * `exp` 필수
    * `GATEWAY_JWT_PUBLIC_KEY` external configuration 추가
* Final loader는 누락, 비-PEM 형식, 2048-bit 미만 key를 거부합니다.

### External Development Steps

* Gateway 밖에서 RSA signing key pair를 생성하거나 선택합니다.
* Token 발급 주체에 private key를 안전하게 배치합니다.
* 대응 public key를 gateway secret/configuration에 등록합니다.
* Public key를 multiline PEM 또는 escaped-newline 형식으로 process environment에 주입합니다.
* Rotation 시 token signer와 gateway public key의 전환 순서를 계획하고 gateway를 재시작합니다.

### Code Connection

* HTTP Resource Server와 STOMP `CONNECT` 인증이 같은 decoder와 key material에 의존합니다.
* Public key가 없거나 형식이 잘못되면 servlet startup이 실패합니다.
* Private key는 gateway 코드에 존재하지 않으며 존재해서도 안 됩니다.

### Evidence Boundary

* **Directly observed:** Public key 형식, 최소 bit 수, RS256 algorithm, startup validation
* **Required/inferred:** 대응 private signing key와 token issuer의 존재, public key 전달
* **Unobserved:** 실제 issuer, private/public key 값, secret system, key generation·rotation 이력

### Ordering

**Conceptual execution order**

1. Token issuer와 signing key 결정
2. Private key를 issuer에 배치
3. Public key를 gateway secret store에 등록
4. Gateway에 주입 후 startup validation
5. HTTP 및 STOMP token 검증 확인
6. Rotation 시 issuer와 gateway를 조정해 전환

---

## Packet E-04 — Thread 4

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 4
* **한국어 제목:** HTTP 메서드·경로 허용목록과 정확한 프록시 라우팅
* **English title:** HTTP Method/Path Allowlist and Exact Proxy Routing
* **Gaps:** G-04

### Repository Evidence

* **`d61ef88550a4c178d4b600cfd7ad6af6bf7d8f8e` — `feat(routing): route betting requests`**

  * 관련 파일:

    * `BettingDownstreamProperties.java`
    * `BettingRoutes.java`
    * `application.yml`
  * 관련 diff:

    * `GATEWAY_BETTING_URI`
    * strict HTTP base URI validation
    * public `/api/v1/bets`를 downstream `/internal/v1/bets`로 rewrite
    * configured URI를 실제 proxy target으로 사용
* Final configuration은 betting, wallet, odds-feed의 base URI를 각각 runtime environment에서 받습니다.

### External Development Steps

* 각 downstream service의 실제 runtime endpoint를 준비합니다.
* Gateway runtime에서 해당 endpoint로 연결 가능한 network path를 구성합니다.
* 세 개 base URI를 environment에 주입합니다.
* URI가 service root만 가리키고 user-info, query, fragment, 추가 path를 포함하지 않는지 검증합니다.
* Gateway startup 이후 각 허용 route가 정확한 downstream contract로 도달하는지 확인합니다.

### Code Connection

* Route handler는 configured base URI에 의존해 proxy target을 생성합니다.
* 잘못된 URI는 startup configuration validation에서 거부됩니다.
* Service가 존재하지 않거나 network path가 막혀 있으면 요청 시 502/504 failure normalization 경로로 들어갑니다.

### Evidence Boundary

* **Directly observed:** URI property, 형식 제약, rewrite behavior
* **Required/inferred:** 실제 downstream deployment와 network reachability
* **Unobserved:** 실제 URI, service discovery, network policy, DNS/TLS 사용 여부, production reachability

### Ordering

**Conceptual execution order**

1. Downstream service 배치
2. 각 service의 root base URI 결정
3. Gateway-to-service network path 허용
4. Runtime URI 주입
5. Allowed route별 end-to-end proxy 검증

---

## Packet E-05 — Thread 5

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 5
* **한국어 제목:** 신뢰 헤더 차단과 다운스트림 신원·자격 증명 재구성
* **English title:** Trust Header Isolation and Downstream Identity/Credential Reconstruction
* **Gaps:** G-05

### Repository Evidence

* **`f9a42f3f85a3873a52eacdc6b856bd9571ba508d` — `feat(routing): validate wallet service credentials`**

  * `GATEWAY_WALLET_API_KEY`를 추가하고 최소 32자를 요구합니다.
* `BettingDownstreamProperties`도 betting key를 최소 32자로 검증하고 `toString()`에서 값을 redaction합니다.
* **`cb86b248de9f5fb7f14fa025716f03cbe7ae8fd2` — `feat(routing): require distinct downstream credentials`**

  * betting과 wallet key가 같으면 startup을 중단합니다.

### External Development Steps

* Betting 전용 key와 wallet 전용 key를 서로 독립적으로 생성합니다.
* Betting service에 betting key를, wallet service에 wallet key를 등록합니다.
* Gateway secret system에 두 key를 별도 항목으로 저장하고 process environment에 주입합니다.
* Rotation 시 downstream service가 새 key를 인식하는 시점과 gateway 재시작 시점을 조정합니다.
* 이전 key 폐기 여부와 시점은 실제 downstream credential policy에 따라 처리합니다.

### Code Connection

* Gateway는 외부 caller가 보낸 internal credential header를 제거합니다.
* 검증된 요청에만 `X-Internal-Service: gateway`와 route별 key를 다시 구성합니다.
* Downstream이 같은 credential을 인식하지 않으면 gateway code가 정상이어도 private route가 성립하지 않습니다.

### Evidence Boundary

* **Directly observed:** Key 길이, distinctness, gateway 주입 point, redaction
* **Required/inferred:** Downstream 측 key 등록과 양측의 coordinated rotation
* **Unobserved:** Key 값, 생성 방법, downstream 저장 위치, secret manager, rotation 실행 이력

### Ordering

**Conceptual execution order**

1. 두 개의 독립 credential 발급
2. 각 downstream service에 대응 key 등록
3. Gateway secret store에 저장
4. Gateway runtime에 주입
5. Startup distinctness validation
6. 인증된 proxy call 검증
7. 필요 시 coordinated rotation

---

## Packet E-06 — Thread 6

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 6
* **한국어 제목:** 신뢰 프록시 판별과 Fail-Open 복구를 포함한 분산 요청 제한
* **English title:** Distributed Rate Limiting with Trusted-Proxy Resolution and Fail-Open Recovery
* **Gaps:** G-06, G-07

### Repository Evidence

1. **`9ee929bf045dde8cbf0773a9de72928bc3909e0e` — `feat(ratelimit): configure bounded Redis access`**

   * Redis host, port, database, optional username/password, SSL을 외부 구성으로 받습니다.
   * 300 ms connect timeout, 500 ms command timeout과 reconnect behavior를 설정합니다.

2. **`1108ae51b21e16b244a81ff2b1bb96a784a4b9d3` — `feat(ratelimit): consume distributed token buckets`**

   * Redis에 Bucket4j state를 생성합니다.
   * Redis exception은 요청 허용으로 전환됩니다.

3. **`d066c2697d3e41694cb485c5061ebebe1414952a` — `feat(ratelimit): resolve user and trusted peer buckets`**

   * 인증 사용자는 subject, 익명 사용자는 trusted-proxy chain에서 도출한 client IP를 bucket key로 사용합니다.

4. Final state는 Redis failure마다 `gateway.ratelimit.fail.open`을 증가시키고 요청을 허용합니다.

### External Development Steps

#### Redis

* Gateway가 접근할 Redis resource를 준비합니다.
* 실제 host, port, database를 구성합니다.
* 배치가 인증 또는 SSL을 사용하면 대응 credential과 SSL flag를 주입합니다.
* Redis outage 및 recovery 시나리오에서 fail-open과 제한 재개를 확인합니다.

#### Trusted proxy

* 실제 network topology에서 gateway와 직접 연결되는 proxy를 식별합니다.
* 해당 proxy만 CIDR 목록에 넣습니다.
* Proxy가 caller-provided `X-Forwarded-For`를 그대로 통과시키지 않고 자신의 관찰값으로 덮어쓰도록 구성합니다.
* Direct client 배치라면 trusted CIDR을 비웁니다.

### Code Connection

* Redis에는 사용자·IP 단위 distributed bucket이 저장됩니다.
* Redis outage는 service outage가 아니라 제한 우회 상태를 만듭니다.
* 익명 bucket key는 proxy trust 설정에 따라 달라지므로 잘못된 CIDR은 rate-limit identity를 손상시킬 수 있습니다.

### Evidence Boundary

* **Directly observed:** Redis client 설정, bucket key prefix, timeout, fail-open, proxy resolution algorithm
* **Required/inferred:** 실제 Redis resource, network access, proxy overwrite policy
* **Unobserved:** Redis endpoint·credential·bucket content, proxy CIDR, 실제 outage/recovery, forwarding chain

### Ordering

**Conceptual execution order**

1. Direct connection인지 proxy connection인지 topology 결정
2. Redis resource 및 접근 방식 준비
3. Redis runtime configuration 주입
4. 필요한 경우 trusted proxy CIDR 및 header overwrite 구성
5. Gateway 배포
6. 사용자/IP bucket 분리 확인
7. Redis outage 및 recovery 관측
8. Fail-open metric alert 연결

---

## Packet E-08 — Thread 8

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 8
* **한국어 제목:** STOMP 연결 인증과 명령·구독 허용목록
* **English title:** STOMP Connection Authentication and Command/Subscription Allowlisting
* **Gaps:** G-08

### Repository Evidence

* **`c75cbbb47472f3ae103247a6e127fb6cf47daa9a` — `feat(websocket): configure STOMP transport`**

  * `/ws/v1/odds`, `/ws/v1/bets`
  * `setAllowedOriginPatterns(allowedOrigins)`
  * `GATEWAY_WS_ALLOWED_ORIGINS`
  * localhost-only 기본 origin pattern을 추가했습니다.
* Final configuration은 process-local `/topic`, `/queue` simple broker를 사용합니다.

### External Development Steps

* 실제 WebSocket client가 실행되는 origin 목록을 확정합니다.
* 필요한 origin만 runtime configuration에 등록합니다.
* Origin 변경 시 gateway process를 재시작합니다.
* 허용 origin과 비허용 origin의 handshake 결과를 실제 edge 경로에서 확인합니다.

### Code Connection

* Origin configuration은 STOMP command authorization 이전의 WebSocket handshake boundary에 적용됩니다.
* Localhost 기본값을 production에서 유지하면 실제 browser client가 연결되지 않을 수 있습니다.
* 과도한 wildcard는 repository가 구현한 origin boundary를 약화시킵니다.

### Evidence Boundary

* **Directly observed:** Endpoint, origin property, localhost 기본값
* **Required/inferred:** 실제 client origin과 production configuration
* **Unobserved:** Frontend URL, runtime 값, 외부 edge configuration, 실제 handshake 결과

### Ordering

**Conceptual execution order**

1. Client origin inventory 작성
2. 최소 허용목록 결정
3. Runtime environment에 주입
4. Gateway 시작·재시작
5. Positive/negative handshake 검증
6. 이후 STOMP authentication 및 subscription contract 검증

---

## Packet E-10 — Thread 10

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 10
* **한국어 제목:** 원시 Kafka 레코드 소비와 재시도·DLT 복구
* **English title:** Raw Kafka Record Consumption, Retry, and DLT Recovery
* **Gaps:** G-09

### Repository Evidence

1. **`f42a78babcc660c88e4f77e1f12afadce11aa9fc` — `feat(kafka): define event topic inventory`**

   * 네 input topic이 nonblank이고 서로 달라야 합니다.

2. **`4744b5ee72caa720323a435ee4168337cb64907c` — `feat(kafka): route failures by source topic`**

   * 각 source를 `<source>.DLT`의 동일 partition 번호로 보냅니다.

3. **`6f1addd36eba921ffb4e118a36cddb410c04cf4c` — `feat(kafka): bound dead-letter publication`**

   * Raw byte producer
   * `acks=all`
   * idempotence
   * DLT send failure propagation
   * bounded wait

4. Final consumers는 raw byte records와 record-level acknowledgement를 사용합니다. Odds listener group은 `gateway-odds`, bet listener group은 `gateway-bets`입니다.

### External Development Steps

* Kafka cluster 및 bootstrap endpoint 준비
* 네 input topic 생성
* 네 `<source>.DLT` topic 생성
* 각 DLT partition 수를 source와 동일하게 설정
* DLT에 최소 7일 retention 적용
* Gateway consumer와 DLT producer에 필요한 접근권한 설정
* DLT 접근권한을 운영상 필요한 주체로 제한
* Topic 자동 생성에 의존하지 않고 startup 전에 inventory를 검증

### Code Connection

* 동일 partition이 없으면 source partition 번호를 지정한 DLT publication이 실패할 수 있습니다.
* DLT publish가 실패하면 source offset을 성공 처리할 수 없으므로 consumer는 해당 record에서 진행하지 못합니다.
* Fixed consumer group과 process-local STOMP broker 때문에 replica count는 Thread 16의 불변 조건과 결합됩니다.

### Evidence Boundary

* **Directly observed:** Topic naming, source→DLT mapping, fixed groups, producer/consumer semantics
* **Required/inferred:** Cluster, topic, partition, retention, access권한 provisioning
* **Unobserved:** 실제 broker, topic metadata, ACL, offset, lag, DLT record, provisioning 시점

### Ordering

**Conceptual execution order**

1. Production topic 이름 확정
2. Input topic 생성
3. 동일 partition 수의 DLT 생성
4. Retention과 접근권한 설정
5. Bootstrap configuration 주입
6. Gateway 한 replica 배포
7. Consumption 및 DLT publication 검증
8. Lag/DLT 운영 모니터링 연결

---

## Packet E-12 — Thread 12

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 12
* **한국어 제목:** DLT 수동 재처리를 위한 레코드 복원과 헤더 정화
* **English title:** Record Restoration and Header Sanitization for Manual DLT Replay
* **Gaps:** G-10

### Repository Evidence

* **`716679f240d0c40f86c3b9e3fa276198ab0d2d91` — `feat(kafka): sanitize manual replay records`**

  * 관련 파일: `DltReplayRecordFactory.java`
  * 정확한 gateway DLT만 허용
  * DLT·exception·deserialization·delivery-attempt header 제거
  * 원래 key/value를 복제
  * DLT partition과 동일한 source partition 지정
* Operations 문서는 root cause 수정 후 operator가 record를 읽고, 정화하고, source로 publish하고, 결과를 확인하는 절차를 설명합니다. Factory 자체는 publish하지 않습니다.

### External Development Steps

1. DLT record와 exception metadata 분석
2. Poison record의 근본 원인 수정
3. 승인된 operator 또는 replay 도구로 DLT record 읽기
4. Framework header 제거 및 source record 복원
5. 동일 source partition으로 publish
6. Consumer가 record를 성공 처리했는지 확인
7. Replay 대상과 결과를 운영 기록으로 남김

### Code Connection

* Factory는 replay할 `ProducerRecord`만 생성합니다.
* Kafka client 실행, DLT read, source publish, 승인 및 결과 확인은 코드 밖의 운영 행위입니다.
* 잘못된 DLT 또는 partition 변경은 ordering과 event semantics를 손상시킬 수 있습니다.

### Evidence Boundary

* **Directly observed:** 복원·정화 algorithm
* **Required/inferred:** 운영자가 수행하는 read/publish/verification
* **Unobserved:** 실제 poison record, root cause, replay 횟수, operator, approval, audit trail

### Ordering

위 1~7은 **conceptual execution order**이며 실제 replay history를 뜻하지 않습니다.

---

## Packet E-16 — Thread 16

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 16
* **한국어 제목:** Kafka-to-STOMP 전달 파이프라인과 단일 복제본 불변 조건
* **English title:** Kafka-to-STOMP Delivery Pipeline and Single-Replica Invariant
* **Gaps:** G-11

### Repository Evidence

* WebSocket configuration은 외부 shared broker가 아니라 process-local simple broker를 사용합니다.
* Architecture는 Kafka consumer group이 partition을 process 사이에 나누지만 local broker는 자기 process의 subscriber만 알기 때문에 gateway 1.0은 정확히 한 replica여야 한다고 설명합니다.
* **`8248a3233f0fce7ca36a503ee71b7a8a0802d733` — `docs(project): document API gateway`**

  * `Exactly one gateway replica`
  * `Deployment configuration must enforce a replica count of one`
  * scaling은 shared broker 또는 cross-instance fan-out 설계 이후의 범위로 규정합니다.

### External Development Steps

* Deployment platform에서 desired replica count를 1로 설정합니다.
* 실제 활성 replica가 1인지 배포 후 확인합니다.
* Rollout 과정에서도 동시에 Kafka를 소비하는 gateway가 둘 이상 활성화되지 않도록 rollout configuration을 검토합니다.
* Autoscaling을 활성화하지 않거나 최소·최대 replica를 모두 1로 제한합니다.
* 다중 replica로 변경하려면 먼저 shared broker/cross-instance fan-out 설계를 별도 구현해야 합니다.

### Code Connection

* Kafka partition을 받은 process가 해당 event를 local STOMP broker에 게시합니다.
* Subscriber가 다른 process에 연결돼 있으면 그 process-local broker로 event가 전달되지 않습니다.
* Replica 수는 단순 성능 설정이 아니라 delivery correctness 조건입니다.

### Evidence Boundary

* **Directly observed:** Local broker, fixed consumer groups, single-replica requirement
* **Required/inferred:** Deployment platform에서 replica state 강제
* **Unobserved:** Platform 종류, manifest, actual replica count, autoscaling, rollout history

### Ordering

**Conceptual execution order**

1. Process-local fan-out 제약 확인
2. Deployment desired replicas를 1로 설정
3. Autoscaling/rollout 설정 검토
4. Gateway 배포
5. 활성 process 수와 Kafka group assignment 확인
6. 실제 subscriber delivery 검증

---

## Packet E-17 — Thread 17

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 17
* **한국어 제목:** 외부 의존성을 제외한 상태 프로브와 운영 지표
* **English title:** Dependency-Independent Health Probes and Operational Metrics
* **Gaps:** G-12

### Repository Evidence

* **`0a92ec595d11e4e39a5ebbca9f2cbed52787e288` — `feat(observability): expose service health and metrics`**

  * `health`, `info`, `prometheus` 노출
  * liveness에는 `livenessState`만 포함
  * readiness에는 `readinessState`만 포함
  * `gateway.ratelimit.fail.open` counter 추가
* Architecture는 Redis, Kafka, downstream reachability가 readiness에 포함되지 않으므로 dependency outage를 metrics/logs concern으로 다뤄야 한다고 명시합니다.

### External Development Steps

* Liveness probe를 `/actuator/health/liveness`에 등록
* Readiness probe를 `/actuator/health/readiness`에 등록
* Prometheus scrape target을 `/actuator/prometheus`에 등록
* `gateway.ratelimit.fail.open` 증가에 대한 alert 설정
* Kafka consumer lag/DLT, Redis availability, downstream reachability를 별도 monitor로 구성
* Alert destination과 운영 대응 절차 연결

### Code Connection

* Application은 metric과 endpoint를 제공할 뿐 외부 monitoring system에 스스로 등록하지 않습니다.
* Dependency outage가 readiness failure를 만들지 않으므로 외부 monitor가 없으면 fail-open이나 Kafka 장애가 process health만으로 드러나지 않습니다.

### Evidence Boundary

* **Directly observed:** Endpoint, health group, metric name
* **Required/inferred:** Probe registration, scrape, dependency monitor, alert connection
* **Unobserved:** Collector, scrape interval, threshold, dashboard, notification channel, 실제 incident

### Ordering

**Conceptual execution order**

1. Gateway 배포 endpoint 확정
2. Liveness/readiness 등록
3. Prometheus scrape 등록
4. Redis/Kafka/downstream 별도 monitor 구성
5. Alert rule과 destination 설정
6. 장애 주입 또는 통제된 검증으로 signal 확인

---

## Packet E-18 — Thread 18

### Thread Identity

* **Type:** Existing Thread
* **Thread:** 18
* **한국어 제목:** 비밀정보를 정화하는 구조화 JSON 로깅
* **English title:** Secret-Redacting Structured JSON Logging
* **Gaps:** G-13

### Repository Evidence

* **`4eb3e8c0794e20f0c3d111f246408d03d407c123` — `feat(logging): emit redacted structured logs`**

  * 관련 파일:

    * `RedactedEventJsonProvider.java`
    * `logback-spring.xml`
  * message와 stack trace에서 authorization, API key, password, token pattern을 `[REDACTED]`로 치환합니다.
  * Console appender로 JSON을 출력합니다.
* Final configuration은 service, timestamp, level, logger, message, optional stack trace 및 `traceId`/`spanId`를 stdout에 기록합니다.

### External Development Steps

* Runtime platform에서 container/process stdout collection을 활성화합니다.
* 한 줄 JSON이 분할·재포맷되지 않도록 collector parser를 구성합니다.
* 운영 log sink로 전달하고 검색 가능한 service/trace field를 보존합니다.
* 실제 failure message와 stack trace를 사용해 end-to-end redaction을 검증합니다.
* Retention과 접근제어는 선택한 외부 logging platform에서 별도로 결정합니다.

### Code Connection

* Application은 file이나 remote sink에 직접 저장하지 않습니다.
* Collector가 없으면 process 종료와 함께 운영 로그가 소실될 수 있습니다.
* Application-side redaction이 있어도 collector의 enrichment나 다른 platform log source까지 자동으로 정화되는 것은 아닙니다.

### Evidence Boundary

* **Directly observed:** JSON schema 일부, console appender, redaction patterns
* **Required/inferred:** stdout collector와 외부 sink 연결
* **Unobserved:** Collector 종류, sink, retention, permissions, 실제 저장 로그, end-to-end redaction 결과

### Ordering

**Conceptual execution order**

1. Gateway stdout 형식 확인
2. Runtime log collector 구성
3. JSON parser와 field mapping 설정
4. Sink 및 접근정책 설정
5. Test secret pattern으로 redaction 검증
6. Trace ID 기반 검색과 incident workflow 확인

---

# Part III — Proposed New Thread Packets

## 제안 없음

이번 감사에서는 `NEW_THREAD` 조건을 충족하는 External-State 관점을 발견하지 못했습니다.

판정 이유는 다음과 같습니다.

* 키 material lifecycle은 Thread 3에 직접 귀속됩니다.
* Downstream endpoint와 credential은 각각 Thread 4와 Thread 5가 소유합니다.
* Redis와 trusted proxy 외부 상태는 Thread 6의 핵심 문제를 실제 환경에서 완성하는 단계입니다.
* Kafka provisioning과 DLT replay는 이미 Thread 10과 Thread 12가 독립 관점으로 분리돼 있습니다.
* Process-local STOMP의 deployment 제약은 Thread 16 제목과 commit 집합에 이미 명시돼 있습니다.
* Metrics와 logging의 외부 연결은 Thread 17과 Thread 18의 운영 완성 단계입니다.
* 일반적인 runtime deployment는 여러 Thread를 관통하지만 자체 개발 관점으로 선정할 새로운 대표 commit 집합이나 독립 source artifact가 없으므로 프로젝트 수준 단계가 적절합니다.

따라서 기존 commit을 재조합해 별도의 "Infrastructure" 또는 "Deployment" Thread를 억지로 만들지 않습니다.

---

# Part IV — Project-Level External Steps

## PL-01 — 실제 Runtime Environment Provisioning과 Service Deployment

* **Classification:** `PROJECT_LEVEL_EXTERNAL_STEP`
* **Primary Owner:** 없음
* **Related Threads:** Thread 1, Thread 4, Thread 8, Thread 16, Thread 17, Thread 18

### Repository Evidence

* Gateway는 Java 17 executable Spring Boot artifact `gateway-1.0.0.jar`로 package됩니다. Version 확정 commit은 artifact version만 변경하며 외부 배포를 수행하지 않습니다.
* Build/use 문서는 `java -jar target/gateway-1.0.0.jar` 실행과 production dependency configuration을 요구합니다.
* Repository에는 Dockerfile, Compose, Kubernetes/Helm manifest, Terraform 또는 특정 cloud deployment configuration이 포함돼 있지 않습니다.
* 실제 runtime platform과 staging/production environment를 생성하는 자동화도 확인되지 않습니다.

### Required External Step

1. Java 17을 실행할 host, VM, container 또는 managed runtime을 생성합니다.
2. 검증된 `gateway-1.0.0.jar`을 runtime에 전달합니다.
3. Component별 external step을 완료합니다.

   * JWT public key
   * Downstream URI와 API keys
   * Redis
   * Trusted proxy configuration
   * Kafka topic/DLT
   * WebSocket origins
4. Runtime environment variables를 주입합니다.
5. Gateway process를 시작하고 HTTP port를 외부 transport boundary에 연결합니다.
6. Thread 16의 요구대로 한 replica만 활성화합니다.
7. Probe, metrics scrape, stdout log collection을 운영 platform에 등록합니다.
8. HTTP, STOMP, Kafka-to-STOMP, Redis fail-open의 최소 smoke test를 수행합니다.

### Code Connection

이 단계가 없으면 repository에는 실행 가능한 code와 configuration contract만 존재하고 실제 public gateway process는 생성되지 않습니다. 다만 single-replica, probe, logging처럼 명확한 subsystem 제약은 각각 기존 Thread가 소유하며, PL-01은 이들을 한 runtime에 실제로 배치하는 잔여 project-level 단계만 다룹니다.

### Evidence Boundary

* **Directly observed in repository**

  * Java 17 executable artifact
  * HTTP port와 runtime properties
  * 필요한 external dependencies
  * 한 replica 요구
  * probes, metrics, stdout logs
* **Required/inferred from repository**

  * 실제 runtime resource 생성
  * artifact 전달
  * process 시작
  * environment와 network 연결
* **Actual execution not observable from Git**

  * Staging/production environment의 존재
  * 실제 배포 시점과 version
  * Host/container/platform
  * Runtime environment-variable 값
  * Public endpoint
  * DNS 또는 TLS 사용 여부
  * 현재 실행 중인 replica와 process 상태

Repository가 DNS나 TLS certificate를 구체적으로 요구하거나 관리하는 artifact를 포함하지 않으므로, 이 감사에서는 DNS 등록이나 certificate 발급을 별도 Gap으로 주장하지 않습니다.

### Ordering

**Conceptual execution order**

1. Thread 1에 따라 build input과 quality gate 성립
2. Release artifact 생성
3. G-03부터 G-10까지 component-specific 외부 상태 준비
4. Runtime environment 생성
5. Secret/configuration 주입
6. G-11에 따라 한 replica 배포
7. G-12 및 G-13의 observability 연결
8. Smoke test 및 운영 인수

이 순서는 Git에서 확인된 실제 배포 chronology가 아니라, repository contract를 성립시키기 위한 **conceptual execution order**입니다.
