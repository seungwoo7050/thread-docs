## `docs(project): document odds feed 1.0 contracts`

diff --git a/README.md b/README.md
index 0fb837d..61baad4 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,124 @@
-# Odds Feed Service
+# odds-feed-service
 
-Owns sportsbook event discovery, odds ingestion, market state projection, and feed event publication.
+`odds-feed-service`는 외부 배당 공급자를 내부 경기·마켓·배당 계약으로 정규화하는
+Java 17 서비스다. 현재 조회 상태는 Redis에 투영하고, 변경 알림과 정산 입력은 raw Avro
+Kafka record로 발행한다. 베팅 접수, 지갑 처리, 위험 판정과 정산은 소유하지 않는다.
+
+Maven 좌표는 `com.sportsbook:odds-feed-service:1.0.0`이며
+`com.sportsbook:shared-protocol:1.0.0`을 사용한다.
+
+## 책임 경계
+
+```text
+mock provider ─┐
+               ├─ discovery / subscription ─ odds ─ Kafka ack ─ Redis projection
+real provider ─┘                         └ critical ─ Redis Stream ─ fail-close ─ Kafka
+
+admin-api ─ authenticated command ─ operator Stream ─ async Kafka ─ ordered completion
+
+gateway / betting ─ public GET and Redis read model
+settlement       ─ lifecycle and result topics
+```
+
+- `mock` profile은 결정적인 경기, 1X2 마켓, 배당 walk, lifecycle과 결과를 제공한다.
+- `real` profile은 The Odds API의 EPL/NBA 첫 bookmaker `h2h` 가격을 정규화한다.
+- 네 Kafka topic의 payload와 key 계약은 [전달 경로](architecture/odds-ingress-and-delivery-paths.md)에 정리한다.
+- 운영자 suspend/close/reopen은 동기 Kafka API가 아니라 내구 command 접수 API다.
+
+## 상태 안전성
+
+Redis 유효 마켓 상태는 다음 우선순위를 따른다.
+
+```text
+event terminal 또는 provider terminal CLOSED
+  > operator CLOSED/SUSPENDED
+  > feed-availability SUSPENDED
+  > provider OPEN/SUSPENDED
+```
+
+terminal latch와 provider-CLOSED latch에는 TTL이 없다. 활성 경기의 마켓 registry는
+Redis에 유지하므로 process가 교체되어도 terminal lifecycle이 기존 마켓 전체를 닫는다.
+restrictive 상태는 내구 enqueue 뒤 Redis에 먼저 반영되고 Kafka에 발행된다. `OPEN`은
+broker ack 이후에만 반영된다.
+
+Kafka 장애가 발생한 마켓에는 feed hold가 생긴다. broker probe 성공만으로 hold를
+해제하지 않으며, 공급자의 최신 odds가 성공적으로 투영된 뒤에만 해제한다. 이 과정은
+terminal latch나 operator override를 지우지 않는다.
+
+## HTTP API
+
+| method | path | 접근 | 결과 |
+|---|---|---|---|
+| `GET` | `/api/v1/events?cursor=&size=` | anonymous | cursor 경기 목록 |
+| `GET` | `/api/v1/events/{eventId}` | anonymous | 경기 상세 또는 404 |
+| `GET` | `/api/v1/odds/{eventId}/{marketId}/{selectionId}` | anonymous | 현재 배당 또는 404 |
+| `POST` | `/internal/v1/events/{eventId}/markets/{marketId}/suspend` | `admin-api` | 내구 접수 202 |
+| `POST` | `/internal/v1/events/{eventId}/markets/{marketId}/close` | `admin-api` | 내구 접수 202 |
+| `POST` | `/internal/v1/events/{eventId}/markets/{marketId}/reopen` | `admin-api` | 내구 접수 202 |
+
+내부 요청은 `X-Internal-Service: admin-api`, `X-Internal-Api-Key`, 안정적인
+`Idempotency-Key`, UUID `X-Admin-Action-Id`와 trim된 1~256자 `reason`을 요구한다.
+credential 누락·불일치는 401, 올바른 key를 가진 비허용 caller는 403이다. 동일
+idempotency key와 동일 요청은 같은 202를 replay하고 payload가 다르면 409를 반환한다.
+terminal 마켓 reopen도 409다.
+
+health와 Prometheus는 anonymous다. health detail은 인증되지 않은 요청에 노출하지 않으며,
+그 밖의 management 및 미등록 경로는 허용하지 않는다. 자세한 접수·순서·재시도 경계는
+[운영자 마켓 안전성](architecture/operator-market-safety.md)을 따른다.
+
+## 빌드와 실행
+
+Java 17과 Maven Wrapper를 사용한다. 먼저 같은 저장소의 `shared-protocol` 브랜치가 만든
+1.0.0 artifact를 Maven repository에 설치한다.
+
+```sh
+export ODDS_M2=/tmp/odds-feed-m2
+(cd ../shared-protocol && ./mvnw -Dmaven.repo.local="${ODDS_M2}" clean install)
+ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)" \
+  ./mvnw -Dmaven.repo.local="${ODDS_M2}" clean verify
+```
+
+로컬 mock 실행에는 Redis와 Kafka가 필요하다.
+
+```sh
+ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)" \
+  ./mvnw spring-boot:run -Dspring-boot.run.profiles=mock
+```
+
+`real` profile은 `THE_ODDS_API_KEY`를 추가로 요구한다. 내부 API key는 환경변수로만
+주입하며 설정 파일과 로그에 기록하지 않는다. 전체 설정과 검증 명령은
+[빌드·실행·검증](docs/build-run-and-verify.md)에 있다.
+
+## 관측과 검증
+
+- `/actuator/health/liveness`: process 생존성
+- `/actuator/health/readiness`: application, Redis, Kafka publisher, critical/operator delivery 상태
+- `/actuator/prometheus`: Spring 기본 meter와 critical/operator Stream 처리 meter
+- `KafkaPublishThroughputTest`: broker acknowledgement 기준 초당 50건 하한
+- `load-test/run-http-gate.sh`: event/odds 각각 60초 예열과 60초 측정 5회
+
+HTTP gate 결과는 저장소 외부 release artifact로만 생성한다. 실행 방법과 합격 기준은
+[부하 검증](load-test/README.md)에 있다.
+
+## 현재 지원 제한
+
+- Redis Stream poison record 격리와 단계별 delivery checkpoint는 제공하지 않는다.
+- 여러 instance의 provider 생산을 조정하는 leader lease는 제공하지 않는다.
+- provider poll, subscription retry, critical/operator drain은 전용 scheduler로 분리되지 않는다.
+- readiness는 전체 unread backlog, provider freshness와 scheduler lag를 완전히 증명하지 않는다.
+- real provider는 가격 polling만 지원하며 lifecycle과 result를 생성하지 않는다.
+- mock 경기 집합은 process 시작 시 생성되며 장기 실행 중 새 세대를 보충하지 않는다.
+- 종료 시 HTTP graceful shutdown은 사용하지만 모든 provider/Kafka/Stream 작업의 drain을 보장하지 않는다.
+- lifecycle과 placement가 서로 다른 topic에서 경쟁하는 문제는 settlement의 terminal tombstone과
+  late-placement catch-up이 함께 해결해야 한다.
+
+이 제한은 [런타임 소유권과 스케줄링](architecture/runtime-ownership-and-scheduling.md)에 운영
+영향과 함께 정리한다.
+
+## 문서
+
+- [provider·Redis·Kafka 전달 경로](architecture/odds-ingress-and-delivery-paths.md)
+- [운영자 마켓 안전성](architecture/operator-market-safety.md)
+- [런타임 소유권과 스케줄링](architecture/runtime-ownership-and-scheduling.md)
+- [빌드·실행·검증](docs/build-run-and-verify.md)
+- [HTTP 부하 검증](load-test/README.md)
diff --git a/architecture/odds-ingress-and-delivery-paths.md b/architecture/odds-ingress-and-delivery-paths.md
new file mode 100644
index 0000000..744ea91
--- /dev/null
+++ b/architecture/odds-ingress-and-delivery-paths.md
@@ -0,0 +1,105 @@
+# 배당 유입과 전달 경로
+
+## 입력 계약
+
+`OddsProvider`는 경기 검색, 경기별 event stream과 선택적 결과 조회를 제공한다. orchestrator는
+`EventSummary`로 경기를 발견하고 한 event에 하나의 subscription을 유지한다.
+
+| profile | 경기와 가격 | lifecycle/result |
+|---|---|---|
+| `mock` | 고정 seed의 3경기, 1X2 HOME/DRAW/AWAY | scheduled, in-play, terminal과 결정적 결과 |
+| `real` | EPL/NBA, 첫 bookmaker `h2h`, stable internal ID | 제공하지 않음 |
+
+real adapter의 process-local limiter와 Redis 월 quota는 discovery와 odds polling이 공유한다.
+quota는 외부 호출 전에 소비되므로 provider 오류도 사용량에 포함된다.
+
+## 일반 odds 경로
+
+```text
+provider OddsUpdated
+  → terminal/operator/feed 상태 확인
+  → 변화율 기준 판정
+  → 필요한 경우 odds.changed broker ack
+  → odds와 provider/effective market Redis projection
+  → 최신 projection 성공 시 해당 feed hold 해제
+```
+
+`odds.changed`가 필요한 갱신은 broker ack 전에 Redis 가격을 노출하지 않는다. 기준 미만의
+변경은 Kafka record 없이 현재 가격을 갱신할 수 있다. Kafka 전송 실패는 해당 마켓을
+feed-availability `SUSPENDED`로 만든다. 독립 broker probe는 publisher 재시도를 허용하지만
+hold 해제 근거가 아니다. 이후 공급자의 현재 odds가 broker와 Redis 경계를 모두 통과해야
+hold가 사라진다.
+
+일반 odds 경로에는 durable retry queue가 없다. provider가 같은 마켓의 최신 snapshot을
+다시 제공하는 것이 회복 입력이다.
+
+## 중요 경기·마켓 경로
+
+market `SUSPENDED`/`CLOSED`와 terminal lifecycle은 Redis Stream에 먼저 보관된다.
+
+```text
+XADD critical envelope
+  → restrictive Redis projection
+  → consumer group unread/pending poll
+  → raw Avro Kafka publish and ack
+  → 후속 projection
+  → XACK
+  → best-effort XDEL
+```
+
+consumer가 종료되면 idle 시간이 지난 pending record를 다른 poll이 reclaim한다. Kafka ack 뒤
+XACK 전에 종료된 record는 다시 전달되므로 Kafka 출력은 at-least-once다. terminal lifecycle은
+`event:markets:{eventId}` registry를 읽어 모든 알려진 마켓을 먼저 닫는다. registry와
+terminal latch가 Redis에 있으므로 새 process도 같은 폐쇄 집합을 복원한다.
+
+`OPEN`은 Kafka ack가 완료된 뒤에만 유효 projection으로 승격한다. 다음 Redis 우선순위는
+모든 provider와 operator write에 공통이다.
+
+```text
+event terminal 또는 market terminal
+  > operator CLOSED/SUSPENDED
+  > feed hold
+  > provider OPEN/SUSPENDED
+```
+
+## Kafka 출력
+
+모든 record는 Schema Registry framing 없이 shared protocol의 raw Avro binary를 사용하고,
+Kafka key는 `eventId` 문자열이다.
+
+| topic | Avro record | 주요 consumer |
+|---|---|---|
+| `odds.changed` | `OddsChanged` | gateway fan-out |
+| `market.status.changed` | `MarketStatusChanged` | market 상태 구독자 |
+| `event.lifecycle` | `EventLifecycle` | settlement |
+| `match.result` | `MatchResult` | settlement |
+
+같은 key는 한 topic 안에서만 순서를 제공한다. 서로 다른 topic 사이에는 전역 순서가 없다.
+따라서 settlement는 terminal lifecycle을 내구 상태로 남기고, 그 뒤 늦게 도착한 placement도
+다시 판정해야 한다.
+
+## Redis projection
+
+| key | 의미 | 만료 |
+|---|---|---|
+| `odds:{event}:{market}:{selection}` | 현재 decimal odds | cache TTL |
+| `event:{event}` | 현재 경기 요약 | cache TTL |
+| `market:{event}:{market}` | betting이 읽는 유효 상태 | cache TTL |
+| `market:provider:{event}:{market}` | 마지막 provider 상태 | cache TTL |
+| `market:override:{event}:{market}` | operator restrictive 상태 | 명시적 reopen까지 |
+| `event:markets:{event}` | terminal closure용 market registry | active cache 수명과 갱신 |
+| `event:terminal:{event}` | terminal event latch | 없음 |
+| `market:terminal:{event}:{market}` | provider-CLOSED latch | 없음 |
+| `market:feed-hold:{event}:{market}` | broker 장애 후 가격 차단 | 최신 projection 또는 cache TTL까지 |
+| `oddsfeed:critical-events` | 중요 event envelope Stream | ack 뒤 cleanup |
+
+terminal latch는 provider의 late `OPEN`과 operator reopen을 거절한다. feed recovery는
+operator override와 terminal latch를 변경하지 않는다.
+
+## 전달 보장의 한계
+
+- critical Stream은 unread/pending 회수와 ack를 제공하지만 poison 격리와 단계 checkpoint는 없다.
+- critical cleanup은 XACK 뒤 XDEL을 best-effort로 수행한다.
+- raw Avro consumer는 duplicate를 idempotently 처리해야 한다.
+- single process 안에서의 event subscription만 조정하며 여러 replica의 provider 생산을 조정하지 않는다.
+- ordinary odds의 Kafka와 여러 Redis key는 하나의 cross-system transaction이 아니다.
diff --git a/architecture/operator-market-safety.md b/architecture/operator-market-safety.md
new file mode 100644
index 0000000..e26889f
--- /dev/null
+++ b/architecture/operator-market-safety.md
@@ -0,0 +1,82 @@
+# 운영자 마켓 안전성
+
+## 요청 계약
+
+```http
+POST /internal/v1/events/{eventId}/markets/{marketId}/{suspend|close|reopen}
+X-Internal-Service: admin-api
+X-Internal-Api-Key: <환경변수로 주입한 32자 이상 secret>
+Idempotency-Key: <안정적인 요청 key>
+X-Admin-Action-Id: <UUID>
+Content-Type: application/json
+
+{"reason":"trim 후 1~256자"}
+```
+
+`ADMIN_API_INTERNAL_KEY`가 없거나 32자 미만이면 애플리케이션 시작이 실패한다. filter는
+caller 이름만 신뢰하지 않고 API key digest를 constant-time으로 비교한다. credential이
+없거나 틀리면 401, 올바른 key를 가진 caller가 `admin-api`가 아니면 403이다. secret과 인증
+header는 로그에 남기지 않는다.
+
+public event/odds GET, health와 Prometheus만 anonymous다. 다른 route는 기본 거절한다.
+
+## 202의 의미
+
+202는 Kafka publish 완료가 아니다. 다음 두 상태가 하나의 Redis Lua 경계에서 저장됐다는
+뜻이다.
+
+1. `oddsfeed:operator:*` idempotency 상태
+2. 처리할 command의 Redis Stream record
+
+close와 suspend는 같은 원자 경계에서 operator override와 유효 restrictive 상태도
+fail-close한다. reopen은 command를 넣되 기존 override를 유지한다. 따라서 broker 장애나
+process 교체가 접수된 restrictive 상태를 되돌리지 않는다.
+
+## Idempotency와 fingerprint
+
+fingerprint는 다음 canonical UTF-8 값을 SHA-256으로 계산한다.
+
+```text
+format version
+authenticated caller
+action
+lowercase event UUID
+lowercase market UUID
+requested status
+normalized reason
+```
+
+같은 idempotency key와 같은 fingerprint는 새 Stream record 없이 기존 202를 반환한다.
+같은 key의 fingerprint가 다르면 409다. action ID는 감사 상관관계이며 idempotency key를
+대신하지 않는다.
+
+완료 상태는 7일 뒤 만료한다. pending 상태는 처리되기 전에는 만료하지 않는다.
+
+## 마켓별 순서
+
+각 command는 마켓별 증가 sequence를 가진다. processor는 predecessor가 완료된 command만
+적용하므로 동시에 접수된 close/reopen도 Redis 접수 순서를 보존한다.
+
+```text
+submit → Stream pending → predecessor 확인 → Kafka ack → completion CAS → XACK+XDEL
+```
+
+- close/suspend: 접수 시 이미 restrictive projection이며 ack 뒤 command를 완료한다.
+- reopen: terminal latch가 있으면 접수 단계에서 409다.
+- reopen: Kafka ack 뒤 predecessor와 현재 override가 예상값일 때만 CAS로 override를 지운다.
+- 뒤따르는 restrictive command가 있으면 오래된 reopen이 새 restrictive 상태를 지우지 못한다.
+- 완료된 operator record는 하나의 Lua 경계에서 XACK와 XDEL을 수행한다.
+
+Kafka ack 후 Redis completion 전에 process가 종료되면 같은 command가 Kafka에 다시
+발행될 수 있다. 이는 의도한 at-least-once 경계이며 consumer가 event duplicate를 처리해야
+한다.
+
+## 호출자 책임
+
+admin-api는 사용자 요청의 재시도 전반에 같은 `Idempotency-Key`와
+`X-Admin-Action-Id`를 유지한다. 202를 market OPEN 완료로 해석하지 않고 접수 완료로
+해석한다. 401은 credential 설정 문제, 403은 caller 권한 문제, 409는 key 충돌 또는
+terminal 불변식 위반으로 다룬다.
+
+orchestration은 odds-feed와 admin-api에 같은 `ADMIN_API_INTERNAL_KEY` 값을 secret으로
+주입한다. 실제 값은 Compose 파일, 로그와 문서에 저장하지 않는다.
diff --git a/architecture/runtime-ownership-and-scheduling.md b/architecture/runtime-ownership-and-scheduling.md
new file mode 100644
index 0000000..47d14ad
--- /dev/null
+++ b/architecture/runtime-ownership-and-scheduling.md
@@ -0,0 +1,81 @@
+# 런타임 소유권과 스케줄링
+
+## process 내부 소유권
+
+| 구성요소 | 소유 자원 | 종료 시 동작 |
+|---|---|---|
+| Spring MVC/Tomcat | public GET, internal POST, Actuator | graceful HTTP shutdown |
+| `FeedOrchestrator` | event discovery와 provider subscription | 등록 subscription dispose |
+| mock provider | 경기 상태, random walk, result | process와 함께 종료 |
+| real provider | WebClient, limiter, quota 호출 | 진행 중 HTTP의 별도 drain 없음 |
+| critical processor | Redis Stream consumer identity와 poll | ack 전 record는 pending 유지 |
+| operator processor | command Stream poll과 market sequence | ack 전 record는 pending 유지 |
+| Kafka producer | 네 raw Avro topic 전송 | Spring lifecycle에서 close |
+| Lettuce | projection, registry, latch, Stream 연결 | Spring lifecycle에서 close |
+
+## 스케줄링
+
+provider refresh, mock tick, scenario rotation, real poll, critical drain, operator drain과 broker
+probe는 Spring scheduled task다. 별도 task pool을 구성하지 않은 실행에서는 한 scheduler
+thread를 공유한다. WebClient 대기, Redis 호출 또는 Kafka ack 대기가 길어지면 다른 cadence도
+늦어질 수 있다.
+
+provider subscription은 Reactor worker에서 재시도 backoff를 수행한다. retry는 process-local이며
+여러 replica 사이에서 event ownership을 조정하지 않는다. Redis consumer group은 이미
+enqueue된 Stream record를 나눌 수 있지만 provider poll과 enqueue 중복을 막는 leader lease는
+아니다.
+
+## 시작 순서
+
+1. 내부 API key와 typed 설정을 검증한다.
+2. Redis/Kafka client와 provider를 만든다.
+3. mock provider는 고정 seed 경기 집합을 만들고 real provider는 credential을 검증한다.
+4. orchestrator가 경기를 발견하고 Redis event/market registry를 hydrate한다.
+5. event subscription과 critical/operator poll이 시작된다.
+6. broker probe와 delivery health가 readiness에 반영된다.
+
+Redis가 유지된 process 교체에서는 terminal latch와 market registry가 source of truth다.
+JVM map은 조회 가속과 active subscription 소유권일 뿐 terminal 안전성의 유일한 저장소가
+아니다.
+
+## Health와 meter
+
+- liveness는 process가 요청을 처리할 수 있는지 나타낸다.
+- readiness는 Spring readiness state, Redis, Kafka publisher와 critical/operator delivery를 묶는다.
+- Prometheus는 Spring 기본 meter와 다음 application meter를 노출한다.
+
+```text
+oddsfeed.critical.delivery.enqueued
+oddsfeed.critical.delivery.acknowledged
+oddsfeed.critical.delivery.reclaimed
+oddsfeed.critical.delivery.failure
+oddsfeed.critical.delivery.pending
+oddsfeed.operator.action.processed
+oddsfeed.operator.action.processing.failure
+oddsfeed.operator.action.pending
+```
+
+readiness는 전체 unread Stream 길이, poison record, provider freshness, scheduler 지연,
+catalog drift 또는 downstream consumer lag를 모두 증명하지 않는다. broker probe 성공도
+기존 feed hold를 즉시 해제하지 않는다.
+
+## 배포 전제와 종료
+
+현재 안전한 provider 생산 모델은 단일 active instance다. 여러 instance를 사용하려면
+discovery/poll/subscription에 leader lease 또는 event partition ownership이 필요하다.
+consumer group만 추가해 이 조건을 충족할 수 없다.
+
+HTTP graceful shutdown과 Spring bean close는 제공한다. 그러나 provider의 진행 중 WebClient,
+scheduled callback, Kafka late future와 모든 Stream backlog가 종료 제한 시간 안에 drain되는
+것은 보장하지 않는다. ack 전 critical/operator record는 Redis pending state에서 다음
+process가 reclaim한다.
+
+## 외부 의존성
+
+- Redis 7: projection, terminal/override/feed state, quota와 두 Stream
+- Kafka: `odds.changed`, `market.status.changed`, `event.lifecycle`, `match.result`
+- The Odds API: `real` profile에서만 사용
+- OpenTelemetry endpoint: tracing exporter이며 기능 성공의 필수 의존성은 아님
+
+`ADMIN_API_INTERNAL_KEY`와 `THE_ODDS_API_KEY`는 deployment secret으로만 주입한다. 서비스
+설정 파일에는 실제 값을 두지 않는다.
diff --git a/docs/build-run-and-verify.md b/docs/build-run-and-verify.md
new file mode 100644
index 0000000..773c443
--- /dev/null
+++ b/docs/build-run-and-verify.md
@@ -0,0 +1,132 @@
+# 빌드·실행·검증
+
+## 도구와 artifact
+
+- Temurin Java 17
+- Maven Wrapper 3.9.11
+- Redis 7
+- Kafka
+- Docker Compose와 k6는 HTTP release gate에만 필요
+
+서비스 artifact는 `com.sportsbook:odds-feed-service:1.0.0`이고 실행 파일 이름은
+Maven `project.build.finalName`이 결정한다. 공통 값 객체와 Avro record는
+`com.sportsbook:shared-protocol:1.0.0` 하나만 사용한다.
+
+## 격리 Maven repository 빌드
+
+같은 Git 저장소의 `shared-protocol` 브랜치를 별도 디렉터리에 checkout한 뒤 먼저 설치한다.
+
+```sh
+export ODDS_M2=/tmp/odds-feed-m2
+(cd ../shared-protocol && \
+  ./mvnw -Dmaven.repo.local="${ODDS_M2}" clean install)
+
+ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)" \
+  ./mvnw -Dmaven.repo.local="${ODDS_M2}" clean verify
+```
+
+`verify`는 compile, unit/MVC/WireMock, Embedded Kafka, Testcontainers, Spotless와 Checkstyle을
+실행하고 Spring Boot executable jar를 만든다. 테스트를 생략하지 않는다.
+
+최종 파일 이름은 버전 문자열을 script에 고정하지 않고 Maven에서 읽는다.
+
+```sh
+FINAL_NAME=$(./mvnw -q -Dstyle.color=never -DforceStdout \
+  help:evaluate -Dexpression=project.build.finalName)
+test -f "target/${FINAL_NAME}.jar"
+```
+
+dependency 확인에서는 shared protocol이 정확히 1.0.0 한 건이어야 한다.
+
+```sh
+./mvnw dependency:tree \
+  -Dincludes=com.sportsbook:shared-protocol \
+  -Dverbose
+```
+
+## 로컬 실행
+
+mock profile도 Redis와 Kafka가 필요하다. 내부 API key는 32자 이상의 process 환경변수로만
+전달한다.
+
+```sh
+export ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)"
+export REDIS_HOST=localhost
+export REDIS_PORT=6379
+export KAFKA_BOOTSTRAP_SERVERS=localhost:9092
+./mvnw spring-boot:run -Dspring-boot.run.profiles=mock
+```
+
+real profile은 외부 provider key가 추가로 필요하다.
+
+```sh
+export ADMIN_API_INTERNAL_KEY="$(openssl rand -hex 32)"
+export THE_ODDS_API_KEY='<provider secret>'
+./mvnw spring-boot:run -Dspring-boot.run.profiles=real
+```
+
+실제 secret을 shell history, 설정 파일, test report와 애플리케이션 로그에 저장하지 않는다.
+
+## 주요 환경 변수
+
+| 변수 | 의미 | 기본값 또는 요구사항 |
+|---|---|---|
+| `ADMIN_API_INTERNAL_KEY` | admin-api 내부 인증 | 필수, 32자 이상 |
+| `REDIS_HOST`, `REDIS_PORT` | projection/Stream Redis | `localhost`, `6379` |
+| `KAFKA_BOOTSTRAP_SERVERS` | 네 output topic broker | `localhost:9092` |
+| `SERVER_PORT` | HTTP/Actuator port | `8085` |
+| `KAFKA_BROKER_ACK_TIMEOUT` | publisher ack 대기 | `5s` |
+| `CRITICAL_EVENT_STREAM` | critical Redis Stream | `oddsfeed:critical-events` |
+| `CRITICAL_EVENT_GROUP` | critical consumer group | `oddsfeed-publisher` |
+| `CRITICAL_EVENT_CLAIM_IDLE` | pending reclaim 최소 idle | `5s` |
+| `ODDSFEED_MOCK_RANDOM_SEED` | 결정적 mock seed | `424242` |
+| `ODDSFEED_MOCK_TICK_INTERVAL_MS` | mock tick | `500` |
+| `THE_ODDS_API_KEY` | real provider credential | real profile에서 필수 |
+| `OTEL_EXPORTER_OTLP_ENDPOINT` | trace collector | local collector URL |
+| `OTEL_SAMPLING_PROBABILITY` | trace sampling | `1.0` |
+
+operator Stream의 key, group, consumer name, poll과 reclaim은 application 설정으로 조정할
+수 있다. 완료 mapping은 7일 동안 유지하며 pending idempotency 상태에는 만료를 주지 않는다.
+
+## API 점검
+
+```sh
+curl --fail http://localhost:8085/actuator/health/readiness
+curl --fail 'http://localhost:8085/api/v1/events?size=20'
+```
+
+운영자 요청 예시의 key는 재시도에서도 그대로 유지한다.
+
+```sh
+curl --request POST \
+  --header 'Content-Type: application/json' \
+  --header 'X-Internal-Service: admin-api' \
+  --header "X-Internal-Api-Key: ${ADMIN_API_INTERNAL_KEY}" \
+  --header 'Idempotency-Key: admin:market:example-action' \
+  --header 'X-Admin-Action-Id: 5c3ba0a8-f08a-49f4-8e5e-a1ad95ea37d3' \
+  --data '{"reason":"manual risk review"}' \
+  http://localhost:8085/internal/v1/events/00000000-0000-4000-8000-000000000001/markets/00000000-0000-4000-8000-000000000002/suspend
+```
+
+202는 command와 idempotency 상태가 Redis에 저장됐다는 뜻이다. Kafka 완료 여부를 응답과
+동일시하지 않는다.
+
+## 검증 층
+
+| 검증 | 실행 | 보장 범위 |
+|---|---|---|
+| 전체 Maven | `./mvnw clean verify` | Java 17 compile, test, format, static analysis, jar |
+| provider | JUnit/WireMock | 정규화, rate/quota, stable ID와 polling diff |
+| Redis | Testcontainers | projection, registry, latch, hold, Stream reclaim |
+| Kafka | Embedded Kafka | raw Avro, eventId key, ack와 초당 50건 하한 |
+| MVC/security | MockMvc | public/401/403/202/409와 header 계약 |
+| operator | Redis integration | atomic submit, replay/conflict, sequence와 reopen CAS |
+| HTTP gate | Docker/k6 | events/odds read latency와 오류율 |
+
+HTTP gate는 [load-test 안내](../load-test/README.md)대로 저장소 밖에 결과를 만든다.
+
+## CI
+
+CI는 같은 repository의 `shared-protocol` 브랜치를 checkout하고 1.0.0인지 확인한 뒤
+`clean install`한다. 그 다음 실행마다 새 64자리 test key를 만들어 odds-feed의
+`clean verify` process에만 전달한다.
diff --git a/load-test/README.md b/load-test/README.md
new file mode 100644
index 0000000..ed9bac1
--- /dev/null
+++ b/load-test/README.md
@@ -0,0 +1,76 @@
+# HTTP release gate
+
+이 gate는 public 경기 목록과 단일 배당 조회를 독립된 Redis/Kafka 상태에서 검사한다.
+Kafka publisher의 초당 50건 검사는 Maven의 `KafkaPublishThroughputTest`가 별도로 담당한다.
+
+## 요구 도구
+
+- Java 17
+- Docker Compose
+- k6
+- curl과 jq
+- OpenSSL
+- local Maven repository의 `shared-protocol:1.0.0`
+
+## 실행
+
+`RESULT_ROOT`는 존재하지 않는 절대 경로이며 Git 저장소 밖이어야 한다. script는 결과를
+tracked tree에 만들지 않는다.
+
+```sh
+RESULT_ROOT=/tmp/odds-http-gate-result \
+  ./load-test/run-http-gate.sh
+```
+
+선택 환경 변수:
+
+| 변수 | 기본값 | 의미 |
+|---|---|---|
+| `SERVER_PORT` | `8085` | service HTTP port |
+| `REDIS_PORT` | `6392` | Docker Redis host port |
+| `KAFKA_PORT` | `9096` | Docker Kafka host port |
+| `REQUEST_RATE` | `1000` | endpoint별 초당 도착 요청 수 |
+| `PREALLOCATED_VUS` | `200` | k6 pre-allocated VU |
+| `MAX_VUS` | `500` | k6 VU 상한 |
+| `COMPOSE_PROJECT_NAME` | `odds-feed-http-gate` | Compose 격리 이름 |
+| `MAVEN_REPO_LOCAL` | Maven 기본값 | shared 1.0.0이 설치된 repository |
+
+script는 `clean verify`로 jar를 만들고 Maven `project.build.finalName`에서 실행 파일 이름을
+읽는다. event와 odds를 각각 fresh Redis/Kafka에서 검사하며, 고정 seed와 긴 tick interval로
+한 process의 read model을 측정 동안 유지한다. 내부 API key는 build와 service 시작마다 새로
+만들며 결과와 로그에 출력하지 않는다.
+
+## 측정 절차
+
+각 endpoint는 다음 순서로 단독 검사한다.
+
+1. 60초 warm-up
+2. 60초 measurement 5회
+3. 각 measurement의 k6 threshold 판정
+
+measurement 하나라도 다음 조건을 만족하지 못하면 script가 실패한다.
+
+- `http_req_duration` p99 < 50ms
+- `http_req_failed` < 0.1%
+- checks > 99.9%
+- dropped iterations = 0
+
+warm-up은 동일 request rate를 사용하지만 threshold를 release 판정에 넣지 않는다.
+
+## 외부 결과
+
+결과 디렉터리는 endpoint별 k6 summary와 service log만 담는다.
+
+```text
+RESULT_ROOT/
+  events-service.log
+  events/warmup.json
+  events/measure-1.json ... measure-5.json
+  odds-service.log
+  odds/warmup.json
+  odds/measure-1.json ... measure-5.json
+```
+
+이 gate는 고정 mock read model의 HTTP 성능을 검사한다. real provider, multi-instance,
+critical/operator Stream end-to-end 처리량, downstream consumer와 운영 cluster 용량을
+증명하지 않는다. 결과 보존과 폐기는 release 환경이 담당한다.
