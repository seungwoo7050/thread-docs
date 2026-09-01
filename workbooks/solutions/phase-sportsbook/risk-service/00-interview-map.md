# DevThread Sportsbook Risk 개발자 기술면접 맵

## 작성 범위와 기준

이 인덱스는 현재 프로젝트에 축적된 Thread 01~17의 작업 기록에서 실제로 확인되는 커밋 메시지, 파일, 클래스, 스크립트, 테스트만 사용한다. 원격 저장소 상태나 기록에 없는 구현 세부사항은 전제로 두지 않았다.

우선순위는 다음 의미로 사용한다.

- **S**: 질문과 10~30분 축소 구현 모두 반드시 준비할 지점
- **A**: 질문 또는 핵심 구현 가능성이 높아 상세 워크북으로 준비할 지점
- **B**: 별도 백지 구현보다 설계·trade-off 설명이 중요한 지점
- **C**: boilerplate·반복 wiring·단순 설정으로 별도 면접 항목 가치가 낮은 지점

`P01`~`P21`은 상세 워크북의 대표 면접 포인트다. 같은 역량이 여러 Thread에서 반복된 경우 대표 포인트 하나에 통합했다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| B | 01 | `docs(project): introduce risk ownership` | `README.md`<br>`architecture/runtime-and-consistency-boundaries.md` | Redis 권위 상태와 진단·이벤트 경계 | 프로젝트 전체의 데이터 권위와 책임을 설명하는 핵심 설계지만 직접 코딩보다 trade-off 설명 가치가 큼 | 높음 | 낮음 | 07, 10, 14, 15, 16 |
| B | 01 | `docs(project): document risk 1.0 contracts` | `README.md`<br>`architecture/redis-keyspace-and-reservation-lifecycle.md` | standalone Redis, persistence, bounded replay 계약 | Redis를 유일한 권위 저장소로 둔 결과와 장애·복구 한계를 묻기 좋으나 구현 문제로는 범위가 큼 | 높음 | 낮음 | 08, 09, 11, 12, 14 |
| S | 02 | `test(numeric): verify Redis integer boundaries` | `SafeRedisNumber` | P01 Redis Lua 정확 정수 도메인 | Java와 Lua의 서로 다른 수치 모델을 하나의 invariant로 묶는 일반화 가능한 경계 문제 | 높음 | 높음 | 06, 07, 08, 10, 12, 13 |
| A | 02 | `feat(command): define typed risk candidates` | `RiskCheckCommand` | P02 불변 후보 객체와 입력 invariant | 여러 진입점에서 중복 검증을 줄이고 유효하지 않은 상태를 내부에 만들지 않는 기본기 | 높음 | 높음 | 04, 07, 09, 10, 13 |
| A | 03 | `feat(auth): bind internal caller credentials` | `InternalAuthProperties` | P17 caller별 secret 검증과 digest 비교 | secret lifecycle, 일정 시간 비교, 구성 fail-fast를 함께 확인할 수 있음 | 높음 | 높음 | 04, 16, 17 |
| A | 03 | `feat(auth): authorize internal service routes` | `InternalAuthenticationFilter`<br>`InternalSecurityConfiguration` | P17 인증과 경로 소유권 분리 | 인증 성공만으로 모든 내부 endpoint를 허용하지 않는 최소 권한·deny-by-default 판단 | 높음 | 중간 | 04, 16 |
| C | 03 | `build(security): add internal request security` | `pom.xml`<br>`RiskServiceApplication` | 보안 의존성·자동 구성 변경 | 필요한 기반 작업이지만 dependency 추가 자체는 면접 준비 항목 가치가 낮음 | 낮음 | 낮음 | — |
| A | 04 | `feat(api): expose diagnostic risk checks` | `RiskCheckController`<br>`RiskCheckRequest`<br>`RiskCheckResponse` | P18 진단 결과와 권위 승인 계약 분리 | 정상 업무 거절을 HTTP 오류와 구분하고 advisory endpoint의 한계를 설명할 수 있음 | 높음 | 중간 | 01, 07, 10 |
| A | 04 | `feat(api): render stable problem details` | `RestExceptionHandler`<br>`RiskApiException` | P18 안정적 error code와 opaque 내부 오류 | 클라이언트 계약 안정성·정보 노출 방지·예외 분류를 함께 묻기 좋음 | 높음 | 높음 | 03, 09, 11 |
| A | 05 | `feat(limits): encode user override fields` | `LimitOverrideField` | P19 한도 차원의 타입 모델 | 통화별 금액 한도와 통화 중립 선택 한도를 잘못 조합하지 못하게 하는 값 타입 설계 | 높음 | 높음 | 02, 08, 10 |
| A | 05 | `feat(limits): resolve user limit overrides`<br>`test(limits): reject corrupt Redis overrides` | `LimitResolver`<br>`RedisLimitOverrideStore`<br>`LimitOverrideKeys` | P19 override 우선순위와 저장 손상 fail-closed | 0과 없음의 구분, 기본값 fallback 경계, corrupt state 처리 판단을 확인할 수 있음 | 높음 | 높음 | 07, 10, 12 |
| B | 06 | `feat(pattern): evaluate rules in stable order` | `RuleEngine`<br>`PatternRule` | 결정적 규칙 순서와 중복 이름 방지 | 확장 가능한 규칙 합성과 결정성을 설명하기 좋지만 핵심 구현은 비교적 작고 일반적임 | 높음 | 중간 | 07, 10 |
| S | 06 | `test(pattern): verify sudden stake boundaries` | `SuddenStakeIncreaseRule`<br>`SuddenStakeIncreaseRuleTest` | P08 정확한 홀수·짝수 중앙값 임계 비교 | 중앙값, 최근 표본, 정수 절삭, 오버플로를 동시에 다루는 좋은 알고리즘 문제 | 높음 | 높음 | 02, 07, 10 |
| A | 06 | `feat(pattern): detect rapid betting`<br>`feat(pattern): detect repeated selections` | `RapidBettingRule`<br>`RepeatedSameSelectionRule` | P09 후보 포함 count와 패턴 경계 | off-by-one, 통화 중립 선택 count, confirmed/active 구분을 묻기 좋음 | 높음 | 중간 | 07, 10 |
| A | 06 | `feat(history): configure bounded retention`<br>`test(history): verify bounded hot-user histories` | `RiskHistoryProperties`<br>`HistoryKeys`<br>`history-record.lua` | P13 bounded pattern history | 시간 창·표본 상한·TTL·멱등 member로 hot-user 메모리를 제한하는 자료구조 문제 | 높음 | 높음 | 07, 11, 14 |
| A | 07 | `feat(snapshot): define combined risk facts` | `RiskSnapshot`<br>`RedisRiskSnapshotReader`<br>`risk-snapshot.lua` | P10 원자적 진단 스냅샷 | 여러 GET의 torn read와 side-effecting read의 trade-off를 설명할 수 있음 | 높음 | 중간 | 01, 06, 10, 12 |
| A | 07 | `feat(snapshot): define precision-safe snapshot wire` | `RiskSnapshotWire`<br>`RiskSnapshotWireMapper`<br>`RiskWireNumbers` | P10 strict wire와 문자열 수치 | Lua JSON 경계의 정밀도·버전·필수 slot 검증을 작은 구현 문제로 만들기 좋음 | 높음 | 높음 | 02, 12 |
| B | 07 | `feat(events): publish best-effort limit signals` | `KafkaRiskSignalPublisher`<br>`RiskCheckService` | advisory signal과 outbox 부재 | side effect 실패가 판단 결과를 rollback하지 않는 이유와 전달 보장 한계를 설명하는 항목 | 높음 | 낮음 | 01, 15, 16 |
| A | 08 | `feat(counter): namespace monetary counters by currency` | `LimitKeys`<br>`LimitType` | P06 key 차원과 Redis hash tag | 통화 dimension, 선택 중립 key, 사용자 hash tag를 카운터 설계와 함께 묻기 좋음 | 중간 | 중간 | 05, 10, 14 |
| S | 08 | `feat(counter): expose sliding window counters` | `SlidingWindowCounter`<br>`sliding-window.lua` | P06 sorted set + 파생 합계 슬라이딩 카운터 | 자료구조, 멱등성, 만료, 복잡도, 파생 합계 invariant가 한 문제에 모임 | 높음 | 높음 | 10, 11, 12, 14 |
| S | 09 | `feat(reservation): fingerprint reservation requests` | `ReservationFingerprint` | P04 canonical request fingerprint | 정렬·길이 구분·버전·의미 필드 선택을 통해 replay와 conflict를 구분하는 핵심 | 높음 | 높음 | 10, 11, 13, 14 |
| A | 09 | `feat(reservation): define reservation decisions`<br>`test(reservation): verify reservation decision invariants` | `ReservationDecision` | P05 상태별 결과 shape invariant | nullable 필드 조합을 생성 시 차단하고 wire 오류를 조기에 드러내는 타입 설계 | 높음 | 높음 | 04, 10, 11, 14 |
| S | 10 | `feat(reservation): define atomic reservation entrypoint` | `risk-reserve.lua`<br>`ReservationScriptRequest` | P07 원자적 check-and-reserve | 프로젝트의 핵심 동시성 문제이며 질문과 축소 Lua 구현 모두 가치가 가장 높음 | 높음 | 높음 | 01, 08, 09, 11, 12, 17 |
| S | 10 | `test(reservation): verify concurrent rolling capacity` | `ReservationRollingCapacityScriptTest` | P07 마지막 용량 경쟁의 직렬화 | 20개 병렬 요청에서 정확한 승인 cardinality를 검증해 race 이해를 확인할 수 있음 | 높음 | 높음 | 08, 12, 17 |
| S | 10 | `feat(reservation): evaluate currency stake patterns` | `risk-reserve.lua`<br>`ReservationSuddenPatternScriptTest` | P08·P09 Lua 정확 산술과 원자 패턴 판단 | Java 규칙의 수치 알고리즘을 동시 승인 경계에서 정확히 재현하는 고난도 지점 | 높음 | 높음 | 02, 06, 07 |
| S | 11 | `feat(reservation): define atomic commit lifecycle` | `risk-commit.lua`<br>`ReservationTransition` | P11 token-bound lifecycle 상태 머신 | commit·replay·conflict·expiry와 active→committed 이동을 한 문제로 확인 가능 | 높음 | 높음 | 09, 10, 12, 14 |
| A | 11 | `test(reservation): verify reservation expiry recovery`<br>`test(reservation): verify idempotent release lifecycle` | `risk-release.lua`<br>`ReservationExpiryCleanupScriptTest`<br>`ReservationReleaseLifecycleScriptTest` | P11 lazy expiry·release·tombstone | resource 회수와 replay 기억을 동시에 보장하는 경계 조건이 풍부함 | 높음 | 중간 | 10, 12, 17 |
| S | 12 | `feat(snapshot): validate active aggregate consistency` | `risk-snapshot.lua`<br>`RiskSnapshotAggregateConsistencyScriptTest` | P12 active 파생 집계 preflight | lifecycle·entries·sum·per-selection footprint를 모두 검증한 뒤 mutation하는 핵심 무결성 문제 | 높음 | 높음 | 07, 10, 11 |
| S | 12 | `feat(reservation): validate active commit aggregates` | `risk-commit.lua`<br>`ReservationAggregateConsistencyScriptTest` | P12 commit 전 aggregate 검증 | 부분 footprint가 있는 상태에서 commit을 진행하지 않는 validate-before-mutate 원칙 | 높음 | 높음 | 08, 10, 11 |
| A | 12 | `feat(snapshot): validate committed aggregate consistency` | `risk-snapshot.lua`<br>`RiskSnapshotAggregateConsistencyScriptTest` | P12·P06 committed entry/sum invariant | 파생 sum 오염을 기본값이나 재계산으로 숨기지 않는 fail-closed 판단 | 높음 | 중간 | 07, 08, 14 |
| A | 13 | `feat(events): calculate accepted bet exposure` | `AcceptedBetExposure` | P03 정확한 시스템 베팅 조합식 | 조합 수, shape 검증, 중간·최종 오버플로를 묻는 일반 알고리즘 문제 | 높음 | 높음 | 02, 14 |
| A | 13 | `feat(events): validate accepted event envelopes` | `AcceptedBetEnvelope`<br>`AvroCodec` | P14 strict Kafka/Avro 경계 | schema와 도메인 검증, Kafka key binding, canonical ID, observedAt 판단을 확인 가능 | 높음 | 높음 | 02, 09, 14, 15 |
| S | 14 | `feat(events): project accepted risk capacity` | `risk-project-accepted.lua`<br>`AcceptedProjectionRequest` | P15 first-seen direct projection | 예약 없이 먼저 확정된 이벤트를 모든 counter·history에 한 번만 반영하는 핵심 멱등 처리 | 높음 | 높음 | 08, 09, 11, 13 |
| S | 14 | `feat(events): reconcile accepted reservations` | `ReservationAcceptedBetReconciler`<br>`AcceptedBetReconciliation` | P15 commit-first 재조정 decision matrix | `NOT_FOUND`만 projection으로 fallback하고 terminal/conflict를 분리하는 실패 분류 문제 | 높음 | 높음 | 11, 13, 15 |
| A | 14 | `test(events): prevent accepted identity readmission` | `AcceptedReservationBoundaryScriptTest`<br>`ReservationKeys.acceptedFingerprint` | P15 cross-ingress identity 경계 | event projection 뒤 HTTP admission이 같은 bet를 다시 active로 만들지 못하게 하는 시스템 경계 | 높음 | 중간 | 09, 10, 17 |
| S | 15 | `feat(events): consume accepted bet events` | `BetPlacedConsumer`<br>`KafkaConsumerConfiguration` | P16 source offset ack 순서 | 정상·영구·일시 실패를 source ack, retry, DLT로 분기하는 핵심 비동기 처리 | 높음 | 높음 | 13, 14, 16, 17 |
| S | 15 | `feat(events): publish accepted bet dead letters`<br>`test(events): verify dead letter broker confirmation` | `BetPlacedDeadLetterPublisher` | P16 DLT 확인 후 source ack | 유실과 중복 사이의 crash window, timeout, interrupt 복원을 묻는 가치가 높음 | 높음 | 높음 | 14, 16, 17 |
| A | 15 | `feat(events): retain transient accepted bet failures`<br>`test(events): verify unbounded accepted bet retries` | `KafkaConsumerConfiguration` | P16 transient retry와 partition blocking | 무제한 retry의 보장과 처리량 trade-off를 설명해야 하는 운영 경계 | 높음 | 중간 | 01, 16 |
| B | 16 | `feat(metrics): count reservation decisions`<br>`feat(metrics): count reservation transitions`<br>`feat(metrics): measure reservation script latency` | `RiskReservationService`<br>`RedisRiskReservationStore` | bounded-cardinality 운영 metrics | 관찰 가능성은 중요하지만 구현은 Micrometer wiring 비중이 커 설명 준비가 적합 | 높음 | 낮음 | 10, 11, 15 |
| A | 16 | `test(readiness): verify dependency health contracts` | `KafkaHealthIndicator`<br>`application.yml` | P20 Kafka readiness의 timeout·cleanup·interrupt | resource lifecycle과 interruption semantics를 작은 구현으로 검증하기 좋음 | 높음 | 높음 | 03, 15, 17 |
| A | 17 | `test(load): verify concurrent reservation cardinality` | `load-test/concurrent-admission.sh` | P21 동시성 cardinality gate | 동시성 결과를 순서가 아니라 invariant로 검증하는 테스트 설계 역량 | 높음 | 높음 | 10, 11 |
| A | 17 | `test(events): verify accepted-bet Redis projection` | `BetPlacedKafkaRedisIntegrationTest`<br>`BetPlacedKafkaRedisIntegrationSupport` | P21 redelivery·offset·no-double-count 통합 검증 | broker 전달과 Redis 멱등 projection을 함께 확인하는 까다로운 경계 테스트 | 높음 | 중간 | 14, 15, 16 |
| A | 17 | `test(load): run isolated correctness gate`<br>`ci(risk): verify Java 17 correctness gates` | `load-test/run-gate.sh`<br>`load-test/docker-compose.yml`<br>`.github/workflows/verify.yml` | P21 hermetic release gate | readiness polling, cleanup, dependency isolation, release 차단 조건을 설명·구현할 가치가 높음 | 높음 | 중간 | 10, 11, 14, 15, 16 |
| C | 17 | `build(boot): package executable service` | `pom.xml` | 실행 가능한 패키징 설정 | 서비스 배포에 필요하지만 면접 기본기 검증보다는 빌드 설정 성격이 강함 | 낮음 | 낮음 | — |
| C | 17 | `build(release): release risk service 1.0.0` | `pom.xml` | 릴리스 버전 변경 | 단순 버전 확정으로 별도 면접 준비 가치가 없음 | 낮음 | 낮음 | — |

## 상세 워크북 파일

| 파일 | 포함 포인트 | 역할 |
|---|---|---|
| [`01-numeric-domain-and-identity.md`](01-numeric-domain-and-identity.md) | P01~P05 | Redis 정확 정수, 불변 후보, 조합식 노출액, canonical fingerprint, 결과 shape invariant |
| [`02-redis-counters-and-admission.md`](02-redis-counters-and-admission.md) | P06~P10 | 슬라이딩 카운터, 원자적 승인, 중앙값, 동시 패턴 용량, 원자 스냅샷 |
| [`03-lifecycle-and-integrity.md`](03-lifecycle-and-integrity.md) | P11~P13 | commit/release/expiry 상태 머신, 파생 집계 fail-closed, bounded history |
| [`04-events-reconciliation-and-recovery.md`](04-events-reconciliation-and-recovery.md) | P14~P16 | Avro/Kafka 경계, accepted 재조정, source ack·retry·DLT |
| [`05-security-api-and-policy.md`](05-security-api-and-policy.md) | P17~P19 | 내부 인증·인가, 안정적 HTTP 오류, 사용자 한도 override |
| [`06-operations-and-correctness-gates.md`](06-operations-and-correctness-gates.md) | P20~P21 | Kafka readiness와 격리 동시성·통합 correctness gate |

## 대표 Thread와 연관 Thread 관계

| 대표 문제군 | 대표 포인트·Thread | 함께 읽을 Thread | 통합 기준 |
|---|---|---|---|
| 정확한 숫자와 노출액 | P01·P03 / 02·13 | 06, 07, 08, 10, 12 | 모든 금액·합계·조합값이 Redis Lua 정확 정수 범위를 공유 |
| 입력 불변식과 canonical identity | P02·P04·P05 / 02·09 | 04, 10, 11, 13, 14 | 생성 시 invariant, 정규 지문, 상태별 결과 shape를 한 경계 전략으로 묶음 |
| rolling counter와 원자 승인 | P06·P07 / 08·10 | 07, 11, 12, 17 | sorted set+sum primitive와 check-and-reserve 경쟁을 대표 문제로 통합 |
| 패턴 알고리즘과 동시 패턴 용량 | P08·P09·P13 / 06·10 | 07, 11, 14 | 순수 규칙, atomic admission, confirmed history의 역할을 분리해 묶음 |
| 스냅샷과 파생 집계 무결성 | P10·P12 / 07·12 | 08, 10, 11, 14 | atomic view와 validate-before-mutate를 하나의 정합성 축으로 묶음 |
| 예약 lifecycle | P11 / 11 | 09, 10, 12, 14, 17 | commit·release·expiry·replay의 전이를 한 상태 머신 문제로 통합 |
| accepted event 경계·재조정·오프셋 | P14·P15·P16 / 13·14·15 | 09, 11, 16, 17 | bytes→domain→Redis→source ack까지 처리 완료 시점을 한 흐름으로 묶음 |
| 내부 신뢰 경계 | P17·P18·P19 / 03·04·05 | 02, 16, 17 | caller identity, protocol error, 관리 override를 입력 신뢰 경계로 묶음 |
| 운영 준비성과 release gate | P20·P21 / 16·17 | 10, 11, 14, 15 | 의존성 readiness와 실제 동시성·재전달 검증을 운영 correctness로 묶음 |
| 전체 권위·복구 아키텍처 | Thread 01, B | 07~17 | 개별 구현을 관통하는 Redis 권위, advisory Kafka, bounded replay trade-off |

## S/A 완전성 검증

아래 표의 모든 S/A 커밋 묶음은 독립 상세 항목이거나 대표 항목에 명시적으로 통합되어 있다.

| 포인트 | 우선순위 | 포함한 S/A 커밋·Thread | 상태 | 상세 문서 |
|---|---:|---|---|---|
| P01 | S | 02 `test(numeric): verify Redis integer boundaries` | 독립 상세 항목 | `01-numeric-domain-and-identity.md` |
| P02 | A | 02 `feat(command): define typed risk candidates` | 독립 상세 항목 | `01-numeric-domain-and-identity.md` |
| P03 | A | 13 `feat(events): calculate accepted bet exposure` | 독립 상세 항목 | `01-numeric-domain-and-identity.md` |
| P04 | S | 09 `feat(reservation): fingerprint reservation requests` | 독립 상세 항목 | `01-numeric-domain-and-identity.md` |
| P05 | A | 09 `feat(reservation): define reservation decisions`, invariant test | 독립 상세 항목 | `01-numeric-domain-and-identity.md` |
| P06 | S/A | 08 currency namespace·sliding counter, 12 committed aggregate consistency | Thread 12 항목을 대표 카운터 문제에 통합 | `02-redis-counters-and-admission.md` |
| P07 | S | 10 atomic entrypoint·concurrent rolling capacity | 독립 상세 항목 | `02-redis-counters-and-admission.md` |
| P08 | S | 06 sudden boundary, 10 currency stake pattern exact arithmetic | 두 런타임 구현을 하나의 정확 중앙값 문제로 통합 | `02-redis-counters-and-admission.md` |
| P09 | S/A | 06 rapid/repeated rules, 10 atomic pattern capacity | 순수 규칙과 동시 승인 경계를 하나의 대표 항목에 통합 | `02-redis-counters-and-admission.md` |
| P10 | A | 07 combined facts·precision-safe wire | 두 커밋을 atomic snapshot과 strict mapper로 통합 | `02-redis-counters-and-admission.md` |
| P11 | S/A | 11 commit lifecycle·expiry recovery·idempotent release | commit/release/expiry를 한 상태 머신으로 통합 | `03-lifecycle-and-integrity.md` |
| P12 | S/A | 12 active snapshot aggregate·active commit aggregate·committed aggregate | 세 검증 지점을 validate-before-mutate 문제로 통합 | `03-lifecycle-and-integrity.md` |
| P13 | A | 06 bounded retention·hot-user history | 독립 상세 항목 | `03-lifecycle-and-integrity.md` |
| P14 | A | 13 accepted envelope validation | 독립 상세 항목 | `04-events-reconciliation-and-recovery.md` |
| P15 | S/A | 14 accepted projection·reconciliation·readmission prevention | first-seen projection과 cross-ingress identity를 통합 | `04-events-reconciliation-and-recovery.md` |
| P16 | S/A | 15 consume·DLT broker confirmation·transient retry | ack ordering과 실패 분류로 통합 | `04-events-reconciliation-and-recovery.md` |
| P17 | A | 03 caller credentials·route ownership | 인증과 인가를 한 신뢰 경계 문제로 통합 | `05-security-api-and-policy.md` |
| P18 | A | 04 diagnostic contract·stable problem details | 정상 업무 결과와 protocol 오류 문제로 통합 | `05-security-api-and-policy.md` |
| P19 | A | 05 override field·resolver·corrupt store | 차원 타입과 fallback/fail-closed를 통합 | `05-security-api-and-policy.md` |
| P20 | A | 16 dependency health contract | 독립 상세 항목 | `06-operations-and-correctness-gates.md` |
| P21 | A | 17 concurrent cardinality·accepted projection·isolated gate·CI | 단위 race와 격리 E2E gate를 하나의 테스트 전략으로 통합 | `06-operations-and-correctness-gates.md` |

**검증 결과:** 선별표의 모든 S/A 항목이 P01~P21 중 하나에 연결되었다. 연결되지 않은 S/A 항목은 없다.

## 백지 구현 우선순위

1. P07 원자적 check-and-reserve
2. P12 파생 집계 validate-before-mutate
3. P11 commit·release·expiry 상태 머신
4. P16 Kafka source ack·DLT 확인 순서
5. P15 commit-first accepted 재조정
6. P06 슬라이딩 윈도 counter
7. P09 동시 패턴 임계값 승인
8. P04 canonical request fingerprint
9. P08 정확한 홀수·짝수 중앙값 비교
10. P03 시스템 베팅 조합식 노출액
11. P21 동시성 cardinality gate
12. P14 strict accepted event envelope
13. P10 precision-safe snapshot mapper
14. P17 caller별 내부 인증과 route ownership
15. P13 bounded pattern history
16. P01 Redis 정확 정수 연산
17. P20 timeout·cleanup·interrupt readiness
18. P19 typed override resolver
19. P05 상태별 결과 shape
20. P02 불변 후보 객체
21. P18 HTTP 결과·오류 mapper

## 설명 연습 우선순위

1. Thread 01의 Redis 권위 상태, diagnostic advisory, Kafka accepted fact 경계
2. P07 committed+active를 포함한 원자적 승인과 standalone Redis trade-off
3. P11 bounded replay를 갖춘 lifecycle 상태 머신
4. P15 reservation commit과 first-seen projection의 재조정 순서
5. P16 at-least-once, source ack, DLT 중복 crash window
6. P12 자동 보정 대신 fail-closed를 택한 이유
7. P10 atomic snapshot과 side-effecting read
8. P17 caller별 secret, 최소 권한, deny-by-default
9. P06 sorted set+sum의 성능·무결성 trade-off
10. P20 readiness와 liveness, timeout, interruption
11. P21 race·redelivery를 검증하는 테스트 계층
12. P18 업무 거절 200과 stable problem error의 분리
13. Thread 07 best-effort signal에서 outbox를 두지 않은 보장
14. Thread 16 bounded-cardinality metrics와 outcome 관찰
15. Thread 01의 reservation retention 이후 replay 기억 상실과 복구 한계

## 한 문제로 통합한 Thread 묶음

- **P01·P03:** Thread 02의 정확 정수 모델 + Thread 13의 조합식 accepted exposure
- **P04·P05:** Thread 09의 canonical fingerprint + replay 결과 shape invariant
- **P06:** Thread 08 sliding counter + Thread 12 committed aggregate consistency
- **P07:** Thread 10 atomic entrypoint + concurrent rolling capacity + Thread 17 경쟁 gate
- **P08·P09:** Thread 06 패턴 알고리즘 + Thread 10 원자적 패턴 승인
- **P10·P12:** Thread 07 atomic snapshot + Thread 12 파생 집계 하드닝
- **P11:** Thread 11 commit·release·expiry 전체 lifecycle
- **P13:** Thread 06 bounded history + Thread 11 commit history + Thread 14 accepted history
- **P14:** Thread 13 Avro codec·envelope·exposure 경계
- **P15:** Thread 14 first-seen projection + reservation reconciliation + accepted identity readmission 차단
- **P16:** Thread 15 consume·retry·DLT + Thread 16 관련 reconciliation metrics
- **P17·P18·P19:** Thread 03 인증·인가 + Thread 04 HTTP 계약 + Thread 05 관리 override
- **P20·P21:** Thread 16 dependency readiness + Thread 17 격리 correctness/release gate
