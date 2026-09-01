# DevThread Sportsbook Odds 개발자 기술면접 워크북 — 마스터 인덱스

## 사용 범위와 판정 원칙

이 인덱스는 현재 GPT 프로젝트에 축적된 Thread 01~16 작업 기록에서 실제로 확인한 커밋 제목, 파일, 클래스, 함수, 테스트 경계를 기준으로 작성했다. 확인되지 않은 저장소 상태나 구현은 보충하지 않았다.

우선순위는 다음 의미로 사용한다.

- **S**: 질문과 10~30분 백지 구현 모두 준비 가치가 매우 높다.
- **A**: 질문 가치가 높고, 핵심 일부를 축소 구현하기 좋다.
- **B**: 구현보다 설계 판단과 trade-off 설명이 중요하다.
- **C**: 설정·boilerplate·반복 검증 비중이 커 별도 면접 항목으로 만들 필요가 낮다.

같은 역량이 여러 Thread에서 반복되면 대표 항목 하나로 통합했다. 아래 표의 S/A 행은 뒤의 **S/A 완전성 매핑**에서 상세 문서 또는 통합 대상이 반드시 지정되어 있다.

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---:|---|---|---|---|---|---|---|
| C | 01 | `build(protocol): use shared protocol 1.0.0` | `pom.xml` | 공통 계약 artifact 버전 고정 | 의존성 연결 자체는 중요하지만 구현 판단보다 빌드 설정 비중이 크다. 버전 불일치 위험은 Thread 16의 릴리스 설명에 포함하면 충분하다. | 낮음 | 낮음 | 06, 16 |
| A | 01 | `feat(provider): define provider events` | `ProviderEvent`; `ProviderEvent.OddsUpdated`; `ProviderEvent.MarketStatusUpdated`; `ProviderEvent.LifecycleUpdated`; `ProviderEventTest` | sealed 이벤트 계층과 불변 경계 모델 | 허용 이벤트 집합을 닫고 필수 필드를 생성 시점에 검증하는 방식은 언어 핵심 원리, API 경계, exhaustiveness를 함께 확인한다. | 높음 | 중간 | 02, 04, 06 |
| A | 01 | `feat(provider): define odds provider contract` | `OddsProvider`; `EventSummary`; `MatchOutcome` | 공급자 포트와 방어적 복사 | 외부 공급자와 내부 도메인을 분리하고 mutable 컬렉션을 경계에서 복사하는 판단은 다른 시스템에도 그대로 일반화된다. | 높음 | 중간 | 04, 07 |
| B | 02 | `feat(mock): simulate bounded decimal odds` | `OddsSimulator`; `OddsSimulator.initialOdds`; `OddsSimulator.nextOdds` | 범위가 제한된 확률 과정과 숫자 정밀도 | 난수·평균회귀·clamp·반올림은 설명 가치가 있으나 프로젝트 핵심 안정성보다 우선순위가 낮다. | 중간 | 중간 | 03 |
| A | 02 | `feat(mock): seed deterministic event fixtures` | `MockOddsProvider.seed`; `structureRandom`; `oddsRandom`; `MockOddsProviderTest` | 결정적 난수 스트림 분리 | 같은 seed의 재현성과 서로 다른 랜덤 소비 경로의 독립성을 동시에 보장해야 한다. 테스트 가능성과 원인 재현 능력을 직접 확인할 수 있다. | 높음 | 높음 | 03, 04 |
| A | 02 | `feat(mock): advance event lifecycles` | `MockOddsProvider.tick`; `advance`; `transitionTo`; `EventLifecycleStatus` | 시간 기반 lifecycle 상태 머신 | 경계 시각, 큰 시간 점프, terminal 상태의 단조성, 중복 tick을 다루는 기본기를 확인하기 좋다. | 높음 | 높음 | 03, 07, 10 |
| A | 02 | `feat(mock): publish deterministic odds ticks` | `MockOddsProvider`; `Sinks.Many<ProviderEvent>`; `ProviderEvent.OddsUpdated`; `MockOddsProviderTest` | 결정적 이벤트 방출과 replay 의미 | 늦은 구독자가 필요한 이력을 받는 방식, buffer 한계, terminal 이후 방출 금지 등 비동기 경계를 질문하기 좋다. | 높음 | 중간 | 03, 07 |
| B | 03 | `feat(scenarios): crash market odds` | `MockScenario`; `OddsCrash`; `SuddenMarketSuspend`; `MatchPostponed`; `LateGoal`; `ScenarioRotator` | 결정적 장애 주입과 적용 전제 | 테스트하기 어려운 실패를 재현하는 설계는 유용하지만 개별 시나리오 구현은 프로젝트 특수성이 높다. Thread 02의 결정성 문제에 통합해 설명한다. | 중간 | 중간 | 02 |
| S | 04 | `feat(real): enforce request rate limits` | `RateLimiter`; `RateLimiter.tryAcquire`; `RateLimiter.currentUsage`; `Deque<Instant>` | sliding-window rate limiter | 자료구조, 시간 경계, 동시 접근, 복잡도까지 한 문제에서 확인할 수 있고 10~20분 구현에도 적합하다. | 높음 | 높음 | 15, 16 |
| A | 04 | `feat(real): persist monthly request quotas` | `QuotaCounter`; `RedisQuotaCounter`; `increment`; `current`; `currentKey` | 분산 월간 quota와 기간 경계 | process 재시작·다중 호출자에서도 남아야 하는 사용량을 Redis 원자 증가로 다루며 UTC 월 경계와 TTL trade-off를 설명할 수 있다. | 높음 | 높음 | 15 |
| A | 04 | `test(real): verify changed-only polling` | `TheOddsApiProvider.pollSport`; `emitChanges`; `lastSeen`; `deriveEventId`; `deriveMarketId`; `deriveSelectionId` | 안정적 식별자와 변경 감지 | polling 응답의 순서나 객체 재생성에 흔들리지 않는 ID와 snapshot diff는 어댑터 설계의 핵심이다. | 높음 | 높음 | 01, 07 |
| B | 04 | `feat(real): define external provider defaults` | `application-real.yml`; `RealProperties` | typed 설정, secret, blocking I/O 경계 | 설정 검증과 credential 주입은 중요하지만 별도 백지 구현보다 WebClient timeout·scheduler 점유 trade-off 설명이 적절하다. | 중간 | 낮음 | 12, 15, 16 |
| B | 05 | `feat(cache): define projection key contracts` | `CacheKeys` | Redis key namespace와 aggregate 경계 | 안정적인 key 계약은 중요하지만 문자열 조립 자체는 면접 구현 가치가 낮다. 상태 원자성 문제의 전제로만 다룬다. | 중간 | 낮음 | 13, 14 |
| S | 05 | `feat(cache): project feed availability holds` | `RedisOddsCache`; `PROJECT_LATEST_ODDS`; `holdLatestOdds`; `projectLatestOdds`; `isFeedHeld` | 상태 우선순위와 stale update 차단 | terminal, operator override, feed hold, provider 상태가 경쟁한다. last-write-wins가 아닌 명시적 invariant와 원자적 갱신이 필요하다. | 높음 | 높음 | 08, 10, 14 |
| S | 05 | `feat(cache): preserve terminal market closures` | `RedisOddsCache`; `CacheKeys.eventTerminal`; `CacheKeys.marketTerminal`; `storeProviderMarketStatus`; `storeOdds` | terminal latch의 단조성 | 늦은 OPEN·odds update가 종료 상태를 되돌리면 금전적 오류로 이어진다. 무기한 latch와 재시작 내구성을 설명·구현할 가치가 높다. | 높음 | 높음 | 10, 13, 14 |
| S | 05 | `feat(cache): fail close registered event markets` | `RedisOddsCache.closeEventMarkets`; `CacheKeys.eventMarkets`; `CLOSE_EVENT_MARKETS` | registry 기반 원자적 fan-out 종료 | JVM 메모리가 아닌 Redis registry를 source of truth로 삼아 전체 마켓을 한 원자 경계에서 닫는 문제는 정합성·복구·복잡도를 함께 확인한다. | 높음 | 높음 | 10 |
| B | 06 | `test(kafka): verify Avro binary round trips` | `AvroSerializer`; `AvroDeserializer`; `KafkaConfig` | raw Avro 계약과 typed producer | serializer API 암기보다 schema 호환성·null 처리·계약 버전 설명이 중요하다. | 중간 | 낮음 | 01, 16 |
| S | 06 | `feat(publisher): publish thresholded odds changes` | `OddsFeedPublisher.publishOddsChanged`; `OddsFeedPublisher.publish`; `BrokerAvailability`; `KafkaPublishException` | broker acknowledgement, timeout, interruption, partition key | 성공을 send 호출이 아니라 broker ack로 정의하고, interrupt 복구·bounded wait·health 전이를 정확히 처리해야 한다. | 높음 | 높음 | 08, 10, 14 |
| A | 06 | `test(publisher): verify odds thresholds and keys` | `OddsFeedPublisher.isSignificantChange`; force snapshot 인자; event ID Kafka key | 상대 변화 임계값과 강제 snapshot | 작은 변화를 억제하되 복구 중 snapshot은 강제로 발행해야 한다. 숫자 경계와 정책/메커니즘 분리를 확인하기 좋다. | 높음 | 높음 | 08 |
| A | 07 | `feat(feed): discover and seed provider events` | `FeedOrchestrator.refresh`; `seedProjection`; `RedisOddsCache.getEvent`; `EventCatalog.putIfAbsent` | 재시작 시 durable state hydration | provider가 다시 준 초기 상태보다 Redis의 복구 상태를 우선해야 terminal·override 정보가 퇴행하지 않는다. | 높음 | 중간 | 05 |
| S | 07 | `feat(feed): manage provider subscriptions` | `FeedOrchestrator`; `ConcurrentHashMap<EventId, Disposable>`; `doFinally`; `@PreDestroy` | 구독 단일 소유권과 race-safe cleanup | 동일 event 중복 구독, 동기 종료, 교체된 구독을 오래된 callback이 제거하는 경쟁, 종료 시 누수까지 다룬다. | 높음 | 높음 | 02, 15 |
| A | 07 | `feat(feed): retry failed provider streams` | `FeedOrchestrator`; `Retry.backoff`; `doFinally` | bounded exponential backoff와 jitter | 무한 즉시 재시도로 외부 장애를 증폭하지 않으면서 자동 복구해야 한다. backoff 상한과 jitter의 목적을 설명하기 좋다. | 높음 | 높음 | 15 |
| S | 08 | `feat(feed): project acknowledged odds` | `FeedOrchestrator.handleOdds`; `OddsFeedPublisher`; `RedisOddsCache.projectLatestOdds`; `holdLatestOdds` | Kafka ack 후 projection과 fail-close hold | 외부에 알리지 못한 가격을 읽기 모델에 먼저 노출하면 서비스 간 상태가 갈라진다. ack·threshold·hold·stale timestamp가 함께 얽힌 핵심 문제다. | 높음 | 높음 | 05, 06 |
| A | 08 | `feat(feed): suspend markets during broker outages` | `BrokerAvailability`; `FeedOrchestrator`; `RedisOddsCache.isFeedHeld` | publisher health와 실제 hold 해제 조건의 분리 | health가 회복됐다는 사실만으로 기존 hold를 지우면 안 된다. 복구 snapshot의 실제 broker acknowledgement와 도메인 안전 상태를 구분하는 판단이 중요하다. | 높음 | 중간 | 05, 15 |
| B | 09 | `feat(delivery): define critical event envelopes` | `CriticalEvent`; `QueuedCriticalEvent` | 내구 이벤트 envelope와 타입 불변식 | 모델 자체보다 이후 Stream 전달·복구 경계가 더 중요하다. 상세 문제에서는 입력 계약으로만 사용한다. | 중간 | 중간 | 01, 10 |
| S | 09 | `feat(delivery): consume unread critical events` | `CriticalEventQueue.poll`; consumer group; pending 조회; reclaim | Redis Stream unread/pending/reclaim | 새 레코드와 이전 consumer가 남긴 pending을 함께 처리해야 process 교체 후 유실 없이 복구된다. | 높음 | 높음 | 14, 15 |
| S | 09 | `feat(delivery): acknowledge completed deliveries` | `CriticalEventQueue.acknowledge`; Lua `XACK`+`XDEL`; pending meter | 성공 후 ack와 원자적 cleanup | apply 전에 ack하면 유실되고, ack 뒤 delete 실패를 따로 처리하면 잔존 레코드·계측 불일치가 생긴다. at-least-once 경계를 명확히 묻기 좋다. | 높음 | 높음 | 10, 14 |
| S | 10 | `test(delivery): verify restrictive-first ordering` | `FeedOrchestrator.handleMarketStatus`; `CriticalEventProcessor.applyMarketTransition`; `RedisOddsCache` | 제한 상태 선반영과 OPEN 지연 | suspend/close는 실패 시 보수적으로 먼저 닫고, OPEN은 broker ack 전에는 절대 노출하지 않는 비대칭 설계가 핵심이다. | 높음 | 높음 | 05, 08, 09 |
| S | 10 | `test(delivery): verify terminal delivery ordering` | `CriticalEventProcessor.applyLifecycle`; `RedisOddsCache.closeEventMarkets`; `EventCatalog`; `OddsFeedPublisher` | terminal lifecycle, 마켓 종료, 결과 전달 순서 | lifecycle·마켓 상태·결과가 서로 다른 메시지로 퍼질 때 downstream이 관찰할 수 있는 순서를 설계해야 한다. | 높음 | 높음 | 05, 09 |
| S | 10 | `feat(feed): snapshot registered terminal markets` | `FeedOrchestrator.handleLifecycle`; `RedisOddsCache.getRegisteredMarkets`; `MatchOutcome`; terminal market snapshot | mutation 전 snapshot과 재처리 안전성 | 종료 전에 이전 상태를 snapshot해야 정확한 전이 이벤트를 만들 수 있다. process crash와 재처리 시 중복을 허용하되 reopen은 금지해야 한다. | 높음 | 높음 | 05, 09 |
| B | 11 | `feat(api): read current events` | `EventReadController.get`; `OddsReadController.getOdds` | current projection 조회와 404 계약 | 컨트롤러 자체는 단순하다. 없는 projection을 404로 표현하는 경계만 설명하면 충분하다. | 중간 | 낮음 | 12 |
| B | 11 | `feat(api): paginate event summaries` | `EventReadController.list`; `clampSize`; `CursorPage` | page size 기본값과 상한 | 입력 경계 처리는 필요하지만 독립 문제보다 stable cursor 문제의 일부가 적절하다. | 중간 | 중간 | 16 |
| A | 11 | `feat(api): encode stable event cursors` | `EventReadController.encodeCursor`; `decodeCursor`; `indexAfter`; `EventCatalog.orderedByKickoff`; `CursorPage` | `(kickoff, eventId)` 복합 cursor | 동률 정렬 키, 잘못된 cursor, 삭제·삽입 중 page 이동을 다룬다. 자료구조와 API 안정성을 함께 확인할 수 있다. | 높음 | 높음 | 07 |
| A | 12 | `feat(security): authenticate internal callers` | `InternalApiKeyAuthenticationFilter`; `MessageDigest.isEqual`; `SecurityContextHolder` | constant-time credential 비교와 인증 상태 | raw 문자열 비교를 피하고, 누락·오류 credential은 401로 종료하며 context를 비우는 보안 기본기를 확인한다. | 높음 | 높음 | 13 |
| A | 12 | `test(security): verify route authorization` | `SecurityConfig.securityFilterChain`; public GET allowlist; internal POST authority; `anyRequest().denyAll()` | 인증과 권한의 분리, deny-by-default | 올바른 key를 가진 비허용 caller는 인증은 되지만 403이어야 한다. 미등록 route를 기본 거부하는 경계가 중요하다. | 높음 | 중간 | 11, 13 |
| S | 13 | `feat(commands): fingerprint operator requests` | `MarketActionFingerprint.request`; `idempotencyKey`; SHA-256; length-prefixed UTF-8 framing | canonical request fingerprint와 domain separation | 단순 문자열 이어붙이기의 모호성, normalization, 버전·caller 포함, idempotency key 노출 방지를 모두 확인한다. | 높음 | 높음 | 12, 14 |
| S | 13 | `feat(commands): define atomic operator submissions` | `OperatorSubmissionScript`; `OperatorActionQueue.submit`; idempotency/action Redis keys; Stream `XADD` | dedup·queue append·fail-close의 원자 접수 | same request replay, payload conflict, action ID 충돌, restrictive projection, Stream append가 부분 성공하면 안 된다. | 높음 | 높음 | 05, 14 |
| S | 13 | `feat(commands): defer operator reopens` | `OperatorActionQueue`; `OperatorSubmissionScript`; `TerminalMarketReopenException`; market override | terminal reopen 거부와 ack 전 override 유지 | OPEN은 되돌리기 어려운 완화 전이이므로 접수 단계에서 terminal을 재검사하고 기존 제한을 유지해야 한다. | 높음 | 높음 | 10, 14 |
| A | 13 | `feat(api): accept durable market controls` | `MarketAdminController.transition`; `Idempotency-Key`; `X-Admin-Action-Id`; reason validation | 202 durable acceptance와 요청 validation | 202는 마켓이 이미 OPEN/CLOSED가 되었다는 뜻이 아니라 내구 접수 완료다. API 의미와 재시도 책임을 설명할 가치가 높다. | 높음 | 중간 | 12, 14 |
| S | 14 | `feat(commands): chain per-market operator actions` | `OperatorActionQueue`; `sequenceKey`; `committedKey`; `OperatorMarketAction.sequence`; `predecessor` | 마켓별 순서와 교차 마켓 병렬성 | 전체 직렬화 없이 같은 market만 선행 완료를 강제한다. per-key ordering 자료구조와 invariant를 확인하기 좋다. | 높음 | 높음 | 13 |
| S | 14 | `feat(commands): reclaim interrupted operator deliveries` | `OperatorActionQueue.poll`; pending 조회; `claimExpired`; `QueuedOperatorMarketAction.reclaimed` | process crash 후 pending reclaim | Thread 09와 같은 Stream 복구 원리를 운영 명령에 적용한다. 중복 역량이므로 하나의 통합 문제로 다룬다. | 높음 | 높음 | 09, 15 |
| S | 14 | `feat(commands): define acknowledged completion CAS` | `OperatorCompletionScript`; `OperatorActionQueue.complete`; `Completion` | Kafka ack 후 completion CAS | publish 성공 뒤에도 더 최신 명령·terminal·feed hold가 생길 수 있다. 오래된 reopen이 새 restriction을 지우지 못하게 해야 한다. | 높음 | 높음 | 05, 10, 13 |
| S | 14 | `feat(commands): evaluate queued operator actions` | `OperatorDeliveryDecisionScript`; `OperatorActionQueue.deliveryDecision`; `OperatorDeliveryDecision` | publish 직전 상태 재검사와 supersession | 접수 당시 유효했던 reopen도 전달 시점에는 terminal 또는 새 close에 의해 무효일 수 있다. check-then-act race를 줄이는 핵심 지점이다. | 높음 | 높음 | 05, 10, 13 |
| B | 15 | `chore(runtime): configure service management endpoints` | `application.yml`; liveness/readiness groups; health detail policy | liveness와 readiness 기본 설정 | 설정 암기보다 어떤 dependency를 readiness에 포함할지 설명하는 것이 중요하므로 독립 구현 문제로 만들지 않는다. | 중간 | 낮음 | 16 |
| A | 15 | `feat(kafka): probe broker connectivity` | `KafkaBrokerProbe`; `BrokerAvailability`; `KafkaTemplate.partitionsFor` | metadata probe와 실제 publish health의 관계 | background probe는 장기 outage 뒤 자동 회복 신호를 제공하지만 실제 record acknowledgement와 동일하지 않다. probe 결과를 도메인 hold 해제와 직접 연결하지 않는 판단이 중요하다. | 높음 | 중간 | 06, 08 |
| A | 15 | `feat(delivery): report durable queue health` | `CriticalDeliveryHealthIndicator.health`; `CriticalEventQueue.isHealthy`; `OddsFeedPublisher.isHealthy`; processor health; pending count | 도메인 전달 readiness 집계 | process가 살아 있어도 Redis Stream/Kafka/processor가 실패하면 트래픽을 받으면 안 된다. 다만 pending 수치 자체와 실패 여부는 구분해야 한다. | 높음 | 중간 | 09, 14 |
| A | 15 | `test(health): verify operator readiness details` | `OperatorActionQueue.healthy`; `groupReady`; `updatePendingCount`; `CriticalDeliveryHealthIndicator` | Redis 재접속 후 group 재확인과 health 복구 | DataAccessException 뒤 cached group-ready 상태를 버리지 않으면 재접속 후 영구 실패하거나 잘못된 UP을 낼 수 있다. | 높음 | 중간 | 09, 14 |
| B | 15 | `feat(redis): reconnect after address changes` | `RedisClientConfig`; `redisDnsResolverCustomizer`; `SocketAddressResolver`; `DnsResolvers.JVM_DEFAULT` | DNS 재해석을 포함한 Redis endpoint 교체 복구 | managed Redis의 주소가 바뀔 수 있으므로 reconnect 시 DNS를 다시 해석해야 한다. 구현량보다 client resolver lifecycle과 운영 환경 설명 가치가 높다. | 높음 | 낮음 | 07 |
| B | 16 | `test(load): exercise event reads` | `load-test/scenarios/events.js`; constant-arrival-rate; p99/error/check/dropped thresholds | 부하 gate와 측정 방법 | 운영적 가치가 높지만 구현 면접보다 workload 모델, warm-up, percentile, dropped iteration 해석 설명이 적합하다. | 높음 | 낮음 | 11, 15 |
| B | 16 | `test(load): exercise odds reads` | `load-test/scenarios/odds.js`; Redis에서 실제 fixture 탐색; frozen odds 검증 | 현실적인 fixture와 읽기 성능 검증 | 임의 ID를 호출하지 않고 실제 projection을 찾아 응답 정합성까지 확인하는 테스트 설계가 좋다. 별도 알고리즘 문제로 만들 필요는 낮다. | 중간 | 낮음 | 05, 11 |
| C | 16 | `docs(project): document odds feed 1.0 contracts` | `README.md`; `architecture/odds-ingress-and-delivery-paths.md`; `architecture/operator-market-safety.md`; `architecture/runtime-ownership-and-scheduling.md`; `docs/build-run-and-verify.md` | 릴리스 문서와 현재 제한 | 중요한 설명 자료이지만 구현 자체가 아니라 앞선 S/A 항목의 근거와 운영 한계를 정리한 문서다. | 중간 | 낮음 | 전체 |

## 상세 워크북 파일 배치

| 파일 | 포함 면접 포인트 |
|---|---|
| [`01-state-consistency-and-fail-close.md`](01-state-consistency-and-fail-close.md) | P01 상태 우선순위, P02 ack 후 projection, P03 제한 전이 순서, P04 terminal fan-out |
| [`02-durable-streams-and-recovery.md`](02-durable-streams-and-recovery.md) | P05 Redis Stream reclaim 통합, P06 성공 후 ack, P07 readiness·재접속 |
| [`03-idempotency-ordering-and-safe-reopen.md`](03-idempotency-ordering-and-safe-reopen.md) | P08 canonical fingerprint, P09 원자 접수, P10 마켓별 순서, P11 completion CAS |
| [`04-concurrency-determinism-and-lifecycle.md`](04-concurrency-determinism-and-lifecycle.md) | P12 구독 소유권, P13 retry backoff, P14 hydration, P15 난수 스트림, P16 lifecycle |
| [`05-provider-boundaries-and-external-budgeting.md`](05-provider-boundaries-and-external-budgeting.md) | P17 provider 경계, P18 rate limiter, P19 월 quota, P20 안정 ID·diff |
| [`06-kafka-delivery-and-runtime-health.md`](06-kafka-delivery-and-runtime-health.md) | P21 broker-ack publisher, P22 threshold·force snapshot |
| [`07-http-pagination-and-security.md`](07-http-pagination-and-security.md) | P23 stable cursor, P24 내부 인증·route boundary |

## 대표 Thread와 연관 Thread 관계

| 대표 면접 축 | 대표 Thread | 연관 Thread | 통합 이유 |
|---|---|---|---|
| 상태 precedence와 terminal 단조성 | 05 | 08, 10, 13, 14 | 모두 "더 제한적인 상태를 늦은 완화 전이가 되돌리지 못한다"는 같은 invariant를 사용한다. |
| ack 이후에만 완화 상태 반영 | 08 | 06, 10, 14 | Kafka send 호출이 아니라 broker acknowledgement를 경계로 OPEN·odds projection을 허용한다. |
| Redis Stream 재시작 복구 | 09 | 14, 15 | unread, pending, idle reclaim, 성공 후 ack, Redis 재접속 시 group 재확인이 동일한 전달 원리다. |
| 운영 명령의 멱등성과 순서 | 13 | 14 | 접수 시 원자 dedup과 fail-close를 만들고, 전달 시 per-market predecessor와 completion CAS를 적용한다. |
| 결정적 테스트 환경 | 02 | 03 | seed 분리, clock 주입, 장애 시나리오 전제와 재현 가능한 이벤트 순서를 하나의 테스트 가능성 축으로 본다. |
| 외부 provider 경계 | 01 | 04, 07 | sealed 내부 모델, 외부 DTO 정규화, stable ID, hydration과 subscription이 하나의 adapter/port 경계를 형성한다. |
| 구독·재시도·readiness | 07 | 15 | lifecycle 소유권과 자동 복구가 잘못되면 process는 살아 있어도 실제 feed는 준비되지 않는다. |
| HTTP 노출 경계 | 11 | 12, 13 | public current read, 내부 인증·권한, durable 202 command acceptance를 서로 다른 계약으로 구분한다. |

## S/A 완전성 매핑

아래 상태가 상세 워크북 완전성 검증의 기준이다. 모든 S/A 선별 행은 독립 항목 또는 대표 항목에 명시적으로 통합됐다.

| 선별 대상 | 상태 | 상세 위치 |
|---|---|---|
| T01 `feat(provider): define provider events` | P17에 통합 | `05-provider-boundaries-and-external-budgeting.md#p17` |
| T01 `feat(provider): define odds provider contract` | P17에 통합 | `05-provider-boundaries-and-external-budgeting.md#p17` |
| T02 `feat(mock): seed deterministic event fixtures` | 독립 P15 | `04-concurrency-determinism-and-lifecycle.md#p15` |
| T02 `feat(mock): advance event lifecycles` | 독립 P16 | `04-concurrency-determinism-and-lifecycle.md#p16` |
| T02 `feat(mock): publish deterministic odds ticks` | P15·P16에 통합 | `04-concurrency-determinism-and-lifecycle.md#p15`, `#p16` |
| T04 `feat(real): enforce request rate limits` | 독립 P18 | `05-provider-boundaries-and-external-budgeting.md#p18` |
| T04 `feat(real): persist monthly request quotas` | 독립 P19 | `05-provider-boundaries-and-external-budgeting.md#p19` |
| T04 `test(real): verify changed-only polling` | 독립 P20 | `05-provider-boundaries-and-external-budgeting.md#p20` |
| T05 `feat(cache): project feed availability holds` | 독립 P01 | `01-state-consistency-and-fail-close.md#p01` |
| T05 `feat(cache): preserve terminal market closures` | P01·P04에 통합 | `01-state-consistency-and-fail-close.md#p01`, `#p04` |
| T05 `feat(cache): fail close registered event markets` | 독립 P04 | `01-state-consistency-and-fail-close.md#p04` |
| T06 `feat(publisher): publish thresholded odds changes` | 독립 P21 | `06-kafka-delivery-and-runtime-health.md#p21` |
| T06 `test(publisher): verify odds thresholds and keys` | 독립 P22 | `06-kafka-delivery-and-runtime-health.md#p22` |
| T07 `feat(feed): discover and seed provider events` | 독립 P14 | `04-concurrency-determinism-and-lifecycle.md#p14` |
| T07 `feat(feed): manage provider subscriptions` | 독립 P12 | `04-concurrency-determinism-and-lifecycle.md#p12` |
| T07 `feat(feed): retry failed provider streams` | 독립 P13 | `04-concurrency-determinism-and-lifecycle.md#p13` |
| T08 `feat(feed): project acknowledged odds` | 독립 P02 | `01-state-consistency-and-fail-close.md#p02` |
| T08 `feat(feed): suspend markets during broker outages` | P02·P07에 통합 | `01-state-consistency-and-fail-close.md#p02`, `02-durable-streams-and-recovery.md#p07` |
| T09 `feat(delivery): consume unread critical events` | 독립 P05 | `02-durable-streams-and-recovery.md#p05` |
| T09 `feat(delivery): acknowledge completed deliveries` | 독립 P06 | `02-durable-streams-and-recovery.md#p06` |
| T10 `test(delivery): verify restrictive-first ordering` | 독립 P03 | `01-state-consistency-and-fail-close.md#p03` |
| T10 `test(delivery): verify terminal delivery ordering` | 독립 P04 | `01-state-consistency-and-fail-close.md#p04` |
| T10 `feat(feed): snapshot registered terminal markets` | P04에 통합 | `01-state-consistency-and-fail-close.md#p04` |
| T11 `feat(api): encode stable event cursors` | 독립 P23 | `07-http-pagination-and-security.md#p23` |
| T12 `feat(security): authenticate internal callers` | 독립 P24 | `07-http-pagination-and-security.md#p24` |
| T12 `test(security): verify route authorization` | P24에 통합 | `07-http-pagination-and-security.md#p24` |
| T13 `feat(commands): fingerprint operator requests` | 독립 P08 | `03-idempotency-ordering-and-safe-reopen.md#p08` |
| T13 `feat(commands): define atomic operator submissions` | 독립 P09 | `03-idempotency-ordering-and-safe-reopen.md#p09` |
| T13 `feat(commands): defer operator reopens` | P09·P11에 통합 | `03-idempotency-ordering-and-safe-reopen.md#p09`, `#p11` |
| T13 `feat(api): accept durable market controls` | P08·P09에 통합 | `03-idempotency-ordering-and-safe-reopen.md#p08`, `#p09` |
| T14 `feat(commands): chain per-market operator actions` | 독립 P10 | `03-idempotency-ordering-and-safe-reopen.md#p10` |
| T14 `feat(commands): reclaim interrupted operator deliveries` | P05에 통합 | `02-durable-streams-and-recovery.md#p05` |
| T14 `feat(commands): define acknowledged completion CAS` | 독립 P11 | `03-idempotency-ordering-and-safe-reopen.md#p11` |
| T14 `feat(commands): evaluate queued operator actions` | P11에 통합 | `03-idempotency-ordering-and-safe-reopen.md#p11` |
| T15 `feat(kafka): probe broker connectivity` | P07에 통합 | `02-durable-streams-and-recovery.md#p07` |
| T15 `feat(delivery): report durable queue health` | 독립 P07 | `02-durable-streams-and-recovery.md#p07` |
| T15 `test(health): verify operator readiness details` | P07에 통합 | `02-durable-streams-and-recovery.md#p07` |

**완전성 결과:** S/A 선별 행 가운데 상세 문서 또는 명시적 통합 대상이 없는 항목은 없다.

## 백지 구현 우선순위

1. **P11** 마켓별 completion CAS와 오래된 reopen 차단
2. **P01** 상태 precedence reducer와 terminal 단조성
3. **P09** 멱등 명령의 원자 접수
4. **P05** Redis Stream unread/pending/reclaim 선택
5. **P02** Kafka ack 후 projection과 feed hold
6. **P12** race-safe subscription registry와 cleanup
7. **P21** bounded Kafka acknowledgement 대기와 interrupt 처리
8. **P18** sliding-window rate limiter
9. **P23** 복합 정렬 키 기반 cursor pagination
10. **P08** canonical fingerprint framing
11. **P10** per-market predecessor ordering
12. **P06** 성공 후 ack와 at-least-once 처리
13. **P24** constant-time 내부 인증과 401/403 분리
14. **P20** stable ID와 changed-only snapshot diff
15. **P15** 독립 난수 스트림으로 재현성 보장

## 설명 연습 우선순위

1. 제한 상태를 먼저 반영하고 OPEN을 지연하는 이유
2. terminal latch가 TTL 없이 남아야 하는 이유와 운영 비용
3. broker probe 성공과 기존 feed hold 해제가 다른 이유
4. Kafka ack 후 Redis completion 전 crash가 만드는 at-least-once 중복
5. 같은 idempotency key의 replay와 conflict를 구분하는 canonical fingerprint
6. per-market ordering을 전체 시스템 직렬화 없이 보장하는 방법
7. Redis Stream의 unread, pending, claim, ack, delete 의미
8. subscription map에서 동기 종료와 오래된 cleanup callback 경쟁 처리
9. readiness가 liveness와 다르고, 현재 readiness가 증명하지 못하는 것
10. stable cursor에서 tie-breaker가 필수인 이유
11. 인증 성공과 권한 부재를 401/403으로 구분하는 이유
12. deterministic test에서 구조·가격·결과 난수를 분리하는 이유
13. sliding-window limiter의 시간/공간 복잡도와 lock 범위
14. Redis 기반 월 quota의 UTC 경계·TTL·다중 instance 의미
15. raw Avro와 event ID Kafka key를 선택한 이유

## 한 문제로 통합한 Thread 묶음

- **Thread 05 + 08 + 10 + 14:** 상태 precedence, fail-close, ack 후 완화, safe reopen을 P01~P04와 P11로 통합
- **Thread 09 + 14 + 15:** Redis Stream unread/pending/reclaim, 성공 후 ack, 재접속 health를 P05~P07로 통합
- **Thread 13 + 14:** canonical idempotency, 원자 접수, per-market sequence, completion CAS를 P08~P11로 통합
- **Thread 02 + 03:** seed 분리, time-driven lifecycle, 결정적 장애 주입을 P15~P16으로 통합
- **Thread 01 + 04 + 07:** provider port, immutable internal model, adapter normalization, hydration을 P14·P17·P20으로 통합
- **Thread 06 + 08 + 10:** broker acknowledgement와 projection/terminal 전달 순서를 P02·P03·P04·P21~P22로 통합
- **Thread 07 + 15:** subscription ownership, retry, readiness를 P12~P13·P07로 통합
- **Thread 11 + 12 + 13:** public read, stable cursor, 내부 인증, durable command acceptance를 P23~P24·P08~P09로 통합
