# Sportsbook Gateway 개발자 기술면접 워크북 · 마스터 인덱스

현재 GPT 프로젝트에 축적된 18개 DevThread 학습 기록과 Evidence Packet에서 확인 가능한 커밋·파일·클래스·불변조건만 사용했다. 원격 저장소 상태나 현재 메모리에서 확인되지 않은 구현은 보충하지 않았다. Thread 16의 문서 파일과 설계 내용은 확인되지만 커밋 제목은 확인되지 않아 제목을 만들지 않고 통합 상태로 표시했다.

## 우선순위 기준

| 우선순위 | 의미 | 상세 워크북 |
|---|---|---|
| S | 반드시 준비. 질문과 10~30분 축소 구현 모두에서 핵심 기본기와 불변조건을 확인할 수 있음 | 독립 항목 작성 |
| A | 준비 가치가 높음. 질문 또는 핵심 구현 가능성이 높고 다른 프로젝트에도 일반화됨 | 독립 항목 또는 대표 항목에 명시적 통합 |
| B | 구현보다 설계·개념·운영 trade-off 설명이 중요함 | 별도 상세 항목 없음 |
| C | boilerplate, 반복 검증, 의존성·설정 나열 등으로 별도 면접 준비 효율이 낮음 | 별도 상세 항목 없음 |

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| B | 1 | `build(maven): initialize Java 17 service`<br>`build(wrapper): add Maven wrapper`<br>`build(deps): add integration test support`<br>`build(format): enforce Java formatting`<br>`ci(gateway): verify Java 17 builds`<br>`build(release): release gateway 1.0.0` | `pom.xml`<br>`.mvn/wrapper/maven-wrapper.properties`<br>`mvnw`, `mvnw.cmd`<br>`config/checkstyle/checkstyle.xml`<br>`.github/workflows/ci.yml` | Java 17 빌드 재현성, 검증 lifecycle, 품질 게이트 | 빌드 실패 지점과 재현 가능한 도구 체인을 설명할 가치는 높지만, 별도 백지 구현 문제로 만들면 Maven 설정 암기에 치우친다. | 높음 | 낮음 | 15 |
| C | 1 | `feat(application): start API gateway`<br>`test(application): verify gateway entry point` | `src/main/java/com/sportsbook/gateway/GatewayApplication.java`<br>`src/test/java/com/sportsbook/gateway/GatewayApplicationTest.java` | Spring Boot 진입점과 context smoke test | 프로젝트 기동 확인에는 필요하지만 개발자 판단·불변조건을 깊게 검증하기 어렵다. | 낮음 | 낮음 | — |
| A | 2, 7 | `feat(errors): define gateway problem responses`<br>`test(errors): verify gateway problem shapes`<br>`feat(routing): bound downstream failures`<br>`test(routing): verify upstream failure responses`<br>`test(routing): preserve downstream response contracts` | `GatewayErrorCode.java`<br>`GatewayProblemWriter.java`<br>`DownstreamFailureBoundary.java`<br>`GatewayProblemWriterTest.java`<br>`DownstreamFailureBoundaryTest.java`<br>`ClosedPortProxyTest.java` | **[RES-04](02-rate-limiting-and-http-resilience.md)** · 게이트웨이 소유 실패와 다운스트림 응답 계약의 경계 | 연결·I/O 실패와 read timeout을 구분하면서도 다운스트림의 정상 HTTP 오류는 그대로 보존해야 하는 시스템 경계 문제다. | 높음 | 중간 | 4, 6, 17 |
| S | 3 | `feat(security): verify RS256 user tokens`<br>`test(security): verify token key and lifetime checks` | `JwtDecoderConfiguration.java`<br>`JwtSecurityProperties.java`<br>`RsaPublicKeyLoader.java`<br>`JwtVerificationTest.java` | **[SEC-01](01-security-and-trust-boundaries.md)** · RSA 공개키 검증, RS256 고정, 만료 경계 | 암호화 알고리즘 혼동 방지, 키 형식 검증, exp 경계는 보안 면접에서 질문과 축소 구현 모두 가치가 높다. | 높음 | 높음 | 8, 9 |
| S | 3 | `feat(security): require canonical user claims`<br>`test(security): reject incomplete user identities` | `GatewayClaimsValidator.java`<br>`GatewayJwtAuthenticationConverter.java`<br>`GatewayClaimsValidatorTest.java` | **[SEC-02](01-security-and-trust-boundaries.md)** · canonical UUID subject와 bounded roles | 서명 검증 뒤에도 claim을 정규화·제한하지 않으면 신원 비교와 권한 전달의 불변조건이 깨진다. | 높음 | 높음 | 5, 8, 14 |
| S | 4 | `feat(security): protect HTTP access boundaries`<br>`test(security): verify public and private paths` | `SecurityConfig.java`<br>`HttpAccessBoundaryTest.java` | **[SEC-03](01-security-and-trust-boundaries.md)** · HTTP method/path deny-by-default 허용목록 | 인증 여부와 별개로 노출 가능한 method/path를 정확히 제한하고 ERROR dispatch를 오염시키지 않는 경계 설계다. | 높음 | 높음 | 2, 5, 6 |
| S | 4, 5 | `feat(routing): route betting requests`<br>`test(routing): verify betting route contracts`<br>`feat(routing): route authenticated wallet reads`<br>`test(routing): verify wallet authentication boundary`<br>`feat(routing): expose public event and odds reads`<br>`test(routing): verify public read route boundaries` | `BettingDownstreamProperties.java`<br>`BettingRoutes.java`<br>`WalletDownstreamProperties.java`<br>`WalletRequestAuthentication.java`<br>`WalletRoutes.java`<br>`OddsFeedDownstreamProperties.java`<br>`PublicReadRoutes.java`<br>`GatewayRoutingIntegrationTest.java` | **[SEC-04](01-security-and-trust-boundaries.md)** · 정확한 경로 재작성, subject 강제 주입, 안전한 base URI | prefix 오노출, query 오염, 잘못된 base URI 같은 실제 게이트웨이 취약점을 작은 함수 문제로 축소할 수 있다. | 높음 | 높음 | 3, 5, 7, 15 |
| S | 5 | `test(security): reject spoofed trust headers`<br>`feat(routing): establish downstream identity boundary`<br>`test(routing): verify proxied credential isolation`<br>`feat(routing): validate wallet service credentials`<br>`feat(routing): route authenticated wallet reads`<br>`feat(routing): require betting credentials`<br>`feat(routing): authenticate betting requests`<br>`feat(routing): require distinct downstream credentials`<br>`test(routing): reject reused downstream credentials` | `GatewayHeaders.java`<br>`TrustedHeaderFilter.java`<br>`DownstreamRequestSanitizer.java`<br>`IdentityForwarding.java`<br>`WalletDownstreamProperties.java`<br>`WalletRequestAuthentication.java`<br>`BettingDownstreamProperties.java`<br>`BettingRequestAuthentication.java`<br>`GatewayDownstreamCredentialPolicy.java`<br>`TrustedHeaderFilterTest.java`<br>`DownstreamIdentityBoundaryTest.java`<br>`GatewayDownstreamCredentialPolicyTest.java`<br>`GatewayRoutingIntegrationTest.java` | **[SEC-05](01-security-and-trust-boundaries.md)** · strip-then-rebuild 신뢰 경계 | 호출자 제공 신뢰 헤더·Authorization을 제거한 뒤 검증된 subject와 서비스별 자격 증명만 재구성하는 핵심 보안 불변조건이다. | 높음 | 높음 | 3, 4, 15, 18 |
| S | 6 | `feat(ratelimit): resolve user and trusted peer buckets`<br>`test(ratelimit): verify rate limit key isolation` | `RateLimitKeyResolver.java`<br>`RateLimitKeyResolverTest.java` | **[RES-01](02-rate-limiting-and-http-resilience.md)** · trusted proxy와 X-Forwarded-For 역방향 해석 | 신뢰하지 않은 peer의 spoof를 막고 proxy chain에서 첫 untrusted hop을 찾는 알고리즘·보안 문제다. | 높음 | 높음 | 4, 17 |
| S | 6 | `feat(ratelimit): consume distributed token buckets`<br>`test(ratelimit): verify shared token consumption`<br>`feat(ratelimit): enforce request rate limits`<br>`test(ratelimit): verify HTTP limit responses` | `RateLimiterService.java`<br>`RateLimitFilter.java`<br>`RateLimitHttpConfiguration.java`<br>`DistributedRateLimiterTest.java` | **[RES-02](02-rate-limiting-and-http-resilience.md)** · 분산 토큰 버킷 결과와 HTTP 429 계약 | 상태 원자성, 사용자/IP key 격리, Retry-After 반올림, filter ordering을 함께 묻기 좋다. | 높음 | 높음 | 2, 17 |
| S | 6 | `feat(ratelimit): configure bounded Redis access`<br>`test(ratelimit): verify Redis connection bounds`<br>`feat(ratelimit): consume distributed token buckets` | `RateLimitRedisConfiguration.java`<br>`RateLimiterService.java`<br>`RateLimitRedisConfigurationTest.java`<br>`DistributedRateLimiterTest.java` | **[RES-03](02-rate-limiting-and-http-resilience.md)** · bounded Redis I/O, single-flight 연결, fail-open 복구 | 외부 장애 중 요청 thread 고갈과 연결 폭주를 막고, fail-open의 가용성·보안 trade-off를 설명해야 한다. | 높음 | 높음 | 15, 17 |
| A | 7 | `feat(routing): propagate trace context`<br>`test(routing): verify trace propagation` | `TraceForwarding.java`<br>`TraceForwardingTest.java` | **[RES-05](02-rate-limiting-and-http-resilience.md)** · traceparent 엄격 검증·보존·재구성 | 외부 입력을 그대로 신뢰하지 않으면서 유효한 분산 추적 문맥은 보존하는 경계 parser 문제다. | 높음 | 높음 | 2, 5, 15, 18 |
| S | 8 | `feat(websocket): authenticate STOMP sessions`<br>`test(websocket): verify CONNECT authentication`<br>`feat(websocket): restrict client STOMP commands`<br>`test(websocket): verify destination permissions` | `StompAuthChannelInterceptor.java`<br>`WebSocketConfig.java`<br>`StompAuthChannelInterceptorTest.java` | **[SEC-06](01-security-and-trust-boundaries.md)** · STOMP CONNECT 인증과 command/destination 허용목록 | HTTP handshake와 메시지 프로토콜 인증을 분리하고 SEND·wildcard·비정규 destination을 차단하는 실전 경계다. | 높음 | 높음 | 3, 9, 13, 14 |
| B | 8 | `feat(websocket): configure STOMP transport`<br>`test(websocket): verify endpoint and origin policy` | `WebSocketConfig.java`<br>`src/main/resources/application.yml`<br>`WebSocketEndpointTest.java` | WebSocket endpoint, origin pattern, message/send buffer/time 한도 | 운영·보안 설명은 중요하지만 프레임워크 API 설정을 별도 구현 문제로 만드는 가치는 낮다. | 높음 | 낮음 | 9, 16 |
| S | 9 | `feat(websocket): track raw WebSocket sessions`<br>`test(websocket): verify session registry lifecycle` | `WebSocketSessionRegistry.java`<br>`WebSocketConfig.java`<br>`WebSocketSessionRegistryTest.java` | **[WS-01](04-websocket-lifecycle-and-delivery.md)** · raw 세션 레지스트리의 register/rollback/finally cleanup | 동시 map, callback 예외, 오래된 세션 종료가 새 항목을 지우는 문제까지 포함한 자원 수명 주기 구현이다. | 높음 | 높음 | 8, 16 |
| S | 9 | `feat(websocket): expire authenticated STOMP sessions`<br>`test(websocket): close expired authenticated sessions` | `AuthenticatedSessionExpiryInterceptor.java`<br>`WebSocketSessionRegistry.java`<br>`WebSocketConfig.java`<br>`WebSocketSessionRegistryTest.java` | **[WS-02](04-websocket-lifecycle-and-delivery.md)** · JWT exp 예약, disconnect 취소, orphan task 경쟁 | 장기 연결의 인증 만료와 Future 등록·취소 경쟁을 직접 구현하게 할 가치가 매우 높다. | 높음 | 높음 | 3, 8, 16 |
| A | 10 | `feat(kafka): consume raw event records`<br>`test(kafka): verify record acknowledgment semantics`<br>`feat(kafka): define permanent contract failures`<br>`feat(kafka): classify event failures`<br>`test(kafka): bypass retries for contract failures` | `GatewayKafkaConsumerConfiguration.java`<br>`GatewayKafkaErrorConfiguration.java`<br>`GatewayEventContractException.java` | **[KAF-01](03-kafka-integrity-and-recovery.md)** · raw byte 소비, RECORD ack, permanent/transient 분류 | 역직렬화 경계를 애플리케이션으로 가져온 이유와 재시도 대상 분류를 설명하는 Kafka 기본기다. | 높음 | 중간 | 11, 12, 16 |
| S | 10 | `feat(kafka): route failures by source topic`<br>`test(kafka): verify dead-letter topic mapping`<br>`feat(kafka): bound dead-letter publication`<br>`test(kafka): retain offsets when recovery fails` | `GatewayTopicProperties.java`<br>`GatewayDeadLetterConfiguration.java`<br>`GatewayKafkaProperties.java`<br>`GatewayRecoveryFailureIntegrationTest.java` | **[KAF-02](03-kafka-integrity-and-recovery.md)** · same-partition DLT와 recovery 실패 시 offset 보존 | 실패 레코드가 DLT에 확정되지 않았는데 source offset을 커밋하면 데이터가 사라지는 핵심 정합성 문제다. | 높음 | 높음 | 11, 12, 16 |
| C | 10 | `chore(kafka): define event delivery defaults`<br>`feat(kafka): define event topic inventory` | `src/main/resources/application.yml`<br>`GatewayTopicProperties.java`<br>`GatewayKafkaProperties.java` | Kafka topic·retry·timeout 기본값과 설정 검증 | KAF-01·KAF-02의 조건으로는 중요하지만 설정값 나열 자체는 별도 면접 문제 가치가 낮다. | 낮음 | 낮음 | 11, 12 |
| S | 11 | `feat(events): decode strict Avro records`<br>`test(events): reject malformed Avro records` | `StrictAvroDecoder.java`<br>`StrictAvroDecoderTest.java` | **[KAF-03](03-kafka-integrity-and-recovery.md)** · 정확히 한 record만 허용하는 strict Avro decoder | 성공적으로 한 값을 읽었다는 사실과 입력 전체가 유효하다는 사실을 구분하는 일반화 가능한 parser 원리다. | 높음 | 높음 | 10, 12 |
| S | 11 | `feat(events): validate odds event identities`<br>`test(events): reject invalid odds identities`<br>`feat(events): validate terminal bet identities`<br>`test(events): reject invalid settled bet identities`<br>`test(events): reject invalid voided bet identities` | `GatewayEventContract.java`<br>`GatewayOddsEventContractTest.java`<br>`GatewaySettledEventContractTest.java`<br>`GatewayVoidedEventContractTest.java` | **[KAF-04](03-kafka-integrity-and-recovery.md)** · strict UTF-8 Kafka key와 payload identity 결합 | partition key가 payload 소유권과 다르면 ordering·routing이 깨지므로 canonical identity와 byte decoding을 함께 검증해야 한다. | 높음 | 높음 | 10, 13, 14 |
| S | 11 | `feat(events): validate resolution revisions`<br>`test(events): reject invalid resolution revisions`<br>`fix(events): reject market void lifecycle records`<br>`test(events): quarantine market void records` | `GatewayEventContract.java`<br>`GatewayResolutionRevisionContractTest.java`<br>`GatewayVoidedEventContractTest.java` | **[KAF-05](03-kafka-integrity-and-recovery.md)** · revision snapshot과 terminal lifecycle 의미 불변조건 | 스키마상 읽을 수 있어도 도메인 의미가 모순인 이벤트를 quarantine해야 하는 데이터 정합성 문제다. | 높음 | 높음 | 10, 14 |
| A | 12 | `feat(kafka): sanitize manual replay records`<br>`test(kafka): verify manual replay contract` | `DltReplayRecordFactory.java`<br>`DltReplayRecordFactoryTest.java`<br>`GatewayTopicProperties.java` | **[KAF-06](03-kafka-integrity-and-recovery.md)** · raw record 복원과 framework recovery header 정화 | key/value/application headers는 보존하면서 재복구를 오염시키는 framework headers만 제거해야 하는 선택적 복사 문제다. | 높음 | 높음 | 10, 11, 16 |
| C | 12 | `feat(kafka): define event topic inventory`<br>`feat(kafka): route failures by source topic` | `GatewayTopicProperties.java` | source topic과 paired DLT mapping | KAF-02·KAF-06에 통합되는 지원 규칙이며 단독 문제로 만들면 프로젝트 특수 설정에 치우친다. | 낮음 | 낮음 | 10 |
| A | 13 | `feat(websocket): publish odds updates`<br>`test(websocket): verify odds fan-out` | `GatewayPushPublisher.java`<br>`OddsStreamListener.java`<br>`OddsUpdate.java`<br>`WebSocketStreamIntegrationTest.java`<br>`docs/architecture.md`<br>`docs/realtime-contract.md` | **[WS-03](04-websocket-lifecycle-and-delivery.md)** · 이벤트 범위 공개 fan-out과 single-replica 불변조건 | projection 자체보다 local broker와 Kafka consumer group 결합이 만드는 배치 제약을 설명하는 시스템 설계 가치가 높다. | 높음 | 낮음 | 10, 11, 15, 16 |
| C | 13 | `test(websocket): establish realtime delivery fixture` | `WebSocketStreamFixture.java` | Embedded Kafka·STOMP 통합 테스트 fixture | 테스트 기반은 필요하지만 setup boilerplate는 TEST-01보다 면접 가치가 낮다. | 낮음 | 낮음 | 15 |
| S | 14 | `feat(websocket): project terminal bet updates`<br>`feat(websocket): publish terminal bet updates`<br>`feat(websocket): publish resolution revisions`<br>`test(websocket): verify revision projections` | `BetStatusUpdate.java`<br>`BetStatusStreamListener.java`<br>`GatewayPushPublisher.java`<br>`BetStatusUpdateTest.java`<br>`WebSocketStreamIntegrationTest.java`<br>`docs/realtime-contract.md` | **[WS-04](04-websocket-lifecycle-and-delivery.md)** · owner-scoped 상태 전달과 revision 순서 계약 | 사용자 격리, nullable projection, duplicate/out-of-order 정정 처리, durable replay 부재를 함께 설명해야 한다. | 높음 | 높음 | 3, 8, 10, 11, 15, 16 |
| C | 14 | `test(websocket): verify settled bet fan-out`<br>`test(websocket): verify voided bet fan-out` | `WebSocketStreamIntegrationTest.java` | terminal event별 반복 fan-out 검증 | 핵심 owner isolation과 projection 규칙은 WS-04에 통합되며 이벤트 종류별 반복 테스트는 별도 문제 가치가 낮다. | 낮음 | 낮음 | 15 |
| A | 15 | `test(websocket): observe broker subscription registration`<br>`test(websocket): await broker subscription registration` | `SubscriptionRegistrationProbe.java`<br>`WebSocketStreamFixture.java` | **[TEST-01](05-testing-and-logging.md)** · sleep 없는 구독 등록 완료 barrier | 비동기 테스트를 상태 관찰과 explicit readiness 신호로 결정론화하는 일반화 가능한 테스트 설계다. | 높음 | 높음 | 9, 13, 14 |
| A | 15 | `test(routing): verify concurrent request isolation`<br>`test(realtime): verify concurrent subscriber isolation` | `GatewayRoutingIntegrationTest.java`<br>`WebSocketStreamIntegrationTest.java` | **[TEST-02](05-testing-and-logging.md)** · 동시 request tuple과 subscriber 소유권 교차 오염 검증 | ThreadLocal·security context·header mutation·user destination의 누수는 직렬 테스트로 놓치기 쉬워 동시성 검증 가치가 높다. | 높음 | 중간 | 4, 5, 13, 14 |
| A | 16 | 문서화 작업 — 현재 프로젝트 메모리에서 커밋 제목 미확인 | `README.md`<br>`docs/architecture.md`<br>`docs/http-contract.md`<br>`docs/realtime-contract.md`<br>`docs/operations.md`<br>`docs/build-and-use.md` | **[WS-03](04-websocket-lifecycle-and-delivery.md)**·**[WS-04](04-websocket-lifecycle-and-delivery.md)**에 통합 · process-local 상태, 중복 가능성, 단일 복제본 | 구현 코딩보다 시스템 상태 위치와 consumer/broker topology의 불변조건을 설명하는 것이 핵심이므로 두 대표 실시간 문제에 통합했다. | 높음 | 낮음 | 6, 9, 10, 13, 14, 17, 18 |
| B | 17 | `feat(observability): expose service health and metrics` | `pom.xml`<br>`src/main/resources/application.yml`<br>`RateLimiterService.java` | 외부 의존성과 분리된 liveness/readiness, health/info/prometheus, fail-open metric | 의존성 장애를 재시작 루프로 확대하지 않는 운영 설계는 설명 가치가 높지만 직접 구현은 프레임워크 설정 비중이 크다. | 높음 | 낮음 | 6, 10, 16, 18 |
| C | 17 | `build(deps): add observability support` | `pom.xml` | 관측성 라이브러리 의존성 추가 | 사용 라이브러리 나열은 독립적인 면접 역량을 거의 드러내지 않는다. | 낮음 | 낮음 | 18 |
| A | 18 | `feat(logging): emit redacted structured logs`<br>`test(logging): verify JSON redaction and context` | `RedactedEventJsonProvider.java`<br>`src/main/resources/logback-spring.xml`<br>`StructuredLoggingTest.java` | **[LOG-01](05-testing-and-logging.md)** · message/stack trace 정화와 MDC allowlist | 정규식 치환의 경계와 구조화 로그의 field allowlist를 함께 다루며, 정상 메시지와 예외 경로 모두에서 secret non-leakage를 검증한다. | 높음 | 높음 | 2, 5, 7, 17 |
| C | 18 | `build(deps): add observability support` | `pom.xml` | JSON logging·tracing dependency 구성 | LOG-01의 기반이지만 의존성 추가 자체는 별도 준비 항목으로 만들 필요가 낮다. | 낮음 | 낮음 | 17 |

## 대표 Thread와 연관 Thread 관계

| 대표 면접 축 | 대표 포인트 | 대표 Thread | 연관 Thread | 통합 기준 |
|---|---|---:|---|---|
| JWT와 canonical identity | SEC-01, SEC-02 | 3 | 5, 8, 9, 14 | 서명 검증과 애플리케이션 신원 계약을 분리하되 하나의 인증 경계로 연결 |
| HTTP 노출·신뢰 경계 | SEC-03, SEC-04, SEC-05 | 4, 5 | 2, 3, 7, 15 | allowlist → exact rewrite → strip/rebuild 순서의 요청 경계 |
| 분산 요청 제한 | RES-01, RES-02, RES-03 | 6 | 2, 15, 17 | key 판별, 원자적 소비, 외부 장애 복구를 하나의 상태 흐름으로 연결 |
| HTTP 실패와 추적 | RES-04, RES-05 | 2, 7 | 4, 5, 6, 15, 18 | 게이트웨이 소유 실패만 정규화하고 유효한 trace 문맥은 보존 |
| Kafka 정합성·복구 | KAF-01~KAF-06 | 10, 11, 12 | 13, 14, 16 | raw bytes → strict decode → 의미 검증 → retry/DLT → 수동 replay 흐름 |
| STOMP 인증·연결 수명 | SEC-06, WS-01, WS-02 | 8, 9 | 3, 13, 14, 16 | CONNECT 인증·허용목록과 raw socket resource/exp 수명 주기를 분리 |
| 실시간 projection·topology | WS-03, WS-04 | 13, 14 | 10, 11, 15, 16 | 공개 event fan-out과 owner queue를 분리하고 Thread 16의 single-replica 제약을 통합 |
| 결정적 비동기 검증 | TEST-01, TEST-02 | 15 | 4, 5, 9, 13, 14 | sleep 대신 readiness 신호, 직렬 테스트 대신 동시 격리 invariant를 대표로 선택 |
| 관측 정보의 보안 경계 | LOG-01 | 18 | 2, 5, 7, 17 | 메시지·예외 정화와 MDC field allowlist를 하나의 non-leakage 문제로 통합 |

## 상세 워크북 파일 위치

| 파일 | 포함 포인트 | 역할 |
|---|---|---|
| [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | SEC-01~SEC-06 | JWT, HTTP allowlist, 정확한 프록시, trust-header strip/rebuild, STOMP 인증 |
| [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | RES-01~RES-05 | trusted proxy, 분산 token bucket, Redis 장애 복구, problem contract, traceparent |
| [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | KAF-01~KAF-06 | raw record, strict Avro, 의미 검증, same-partition DLT, 수동 replay |
| [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | WS-01~WS-04 | raw session lifecycle, JWT expiry scheduler, public/user fan-out, single-replica invariant |
| [05-testing-and-logging.md](05-testing-and-logging.md) | TEST-01, TEST-02, LOG-01 | 비동기 준비 완료 신호, 동시 격리, 구조화 로그 정화 |

## S/A 완전성 검증

상세 워크북의 독립 항목은 24개이며, 마스터 표의 Thread 16 A 항목은 WS-03·WS-04에 명시적으로 통합했다. 독립 항목이나 통합 대상이 없는 S/A 항목은 없다.

| 포인트 | 우선순위 | 상태 | 상세 문서 | 통합·연관 Thread |
|---|---|---|---|---|
| SEC-01 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 3 |
| SEC-02 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 3 |
| SEC-03 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 4 |
| SEC-04 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 4 + 5 통합 |
| SEC-05 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 5 |
| SEC-06 | S | 독립 상세 항목 | [01-security-and-trust-boundaries.md](01-security-and-trust-boundaries.md) | Thread 8 |
| RES-01 | S | 독립 상세 항목 | [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | Thread 6 |
| RES-02 | S | 독립 상세 항목 | [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | Thread 6 |
| RES-03 | S | 독립 상세 항목 | [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | Thread 6 |
| RES-04 | A | 독립 상세 항목 | [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | Thread 2 + 7 통합 |
| RES-05 | A | 독립 상세 항목 | [02-rate-limiting-and-http-resilience.md](02-rate-limiting-and-http-resilience.md) | Thread 7 |
| KAF-01 | A | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 10 |
| KAF-02 | S | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 10 |
| KAF-03 | S | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 11 |
| KAF-04 | S | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 11 |
| KAF-05 | S | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 11 |
| KAF-06 | A | 독립 상세 항목 | [03-kafka-integrity-and-recovery.md](03-kafka-integrity-and-recovery.md) | Thread 12 |
| WS-01 | S | 독립 상세 항목 | [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | Thread 9 |
| WS-02 | S | 독립 상세 항목 | [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | Thread 9 |
| WS-03 | A | 독립 상세 항목 | [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | Thread 13 + 16 통합 |
| WS-04 | S | 독립 상세 항목 | [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | Thread 14 + 16 통합 |
| TEST-01 | A | 독립 상세 항목 | [05-testing-and-logging.md](05-testing-and-logging.md) | Thread 15; Thread 13 fixture 연관 |
| TEST-02 | A | 독립 상세 항목 | [05-testing-and-logging.md](05-testing-and-logging.md) | Thread 15; Thread 4 + 5 + 13 + 14 연관 |
| LOG-01 | A | 독립 상세 항목 | [05-testing-and-logging.md](05-testing-and-logging.md) | Thread 18 |
| Thread 16 topology·운영 계약 | A | WS-03·WS-04에 명시적 통합 | [04-websocket-lifecycle-and-delivery.md](04-websocket-lifecycle-and-delivery.md) | Thread 13 + 14의 실제 fan-out 구현과 결합 |

## 백지 구현 우선순위

1. **KAF-02** — same-partition DLT 전송 성공 전 source offset을 보존하는 복구 상태 전이
2. **WS-02** — JWT 만료 Future의 등록·교체·취소 경쟁과 orphan task 방지
3. **RES-03** — Redis cold-connect single-flight, bounded I/O, fail-open 후 재복구
4. **SEC-05** — caller trust headers를 제거하고 검증된 identity·service credential만 재구성
5. **KAF-03** — 입력 전체 소비를 확인하는 strict one-record decoder
6. **KAF-04** — strict UTF-8 key, canonical UUID, payload identity 일치 검증
7. **SEC-06** — STOMP CONNECT credential parser와 command/destination allowlist
8. **SEC-04** — exact route rewrite, query 보존, subject overwrite, safe base URI
9. **RES-01** — trusted proxy CIDR와 X-Forwarded-For right-to-left 판별
10. **WS-01** — raw session register/rollback/finally cleanup과 conditional remove
11. **KAF-05** — revision snapshot·terminal lifecycle 의미 불변조건 검사
12. **RES-02** — token bucket 결과를 200/429·remaining·Retry-After로 변환
13. **WS-04** — owner-scoped projection과 revision update 축소 구현
14. **TEST-01** — subscription registration을 명시적 Future로 기다리는 테스트 probe
15. **TEST-02** — 동시 request/subscriber tuple의 누수 없는 검증 harness
16. **SEC-01, SEC-02, RES-05, KAF-06, LOG-01, KAF-01** — parser·validator·copy-filter 형태의 보조 구현 순환 연습

## 설명 연습 우선순위

1. **WS-03 + Thread 16** — Kafka consumer group과 process-local simple broker가 왜 단일 복제본을 요구하는가
2. **RES-03** — Redis 장애에서 fail-open을 택한 이유, 남는 공격 면과 관측·복구 조건
3. **KAF-02** — DLT publish 실패 시 offset을 커밋하면 왜 데이터가 소실되는가
4. **RES-04** — 게이트웨이 소유 transport 실패와 다운스트림 application response의 소유권 경계
5. **WS-04** — WebSocket 비내구성, 중복·순서 뒤바뀜, revision을 이용한 client 수렴
6. **SEC-05** — strip-then-rebuild와 service별 credential 분리의 보안 모델
7. **SEC-01 + SEC-02** — 서명 검증과 canonical claim validation을 분리해야 하는 이유
8. **KAF-01 + KAF-03~KAF-05** — raw bytes에서 permanent contract failure까지의 검증 단계
9. **TEST-01 + TEST-02** — sleep 기반 테스트가 만드는 허위 성공·실패와 invariant 중심 동시성 테스트
10. **LOG-01 + Thread 17** — 로그 정화·MDC allowlist와 dependency-independent health의 운영 trade-off

## 한 문제로 통합한 Thread 묶음

- **Thread 2 + Thread 7 → RES-04**: 공통 Problem 응답과 다운스트림 실패 정규화를 "오류 소유권과 응답 계약 보존" 한 문제로 통합했다.
- **Thread 13 + Thread 16 → WS-03**: 공개 odds fan-out 구현과 process-local broker의 단일 복제본 제약을 한 설계 문제로 통합했다.
- **Thread 14 + Thread 16 → WS-04**: 사용자 범위 bet projection과 비내구·중복 가능·revision 수렴 계약을 한 문제로 통합했다.
- **Thread 13 + Thread 15 → TEST-01**: 초기 실시간 fixture와 이후 subscription registration probe를 "sleep 없는 준비 완료 동기화" 한 문제로 통합했다.
- **Thread 4 + Thread 5 + Thread 15 → TEST-02**: exact routing, trust-header 재구성, 동시 request isolation을 "요청별 신원·idempotency·trace tuple 비오염" 한 문제로 통합했다.
