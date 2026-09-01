# Sportsbook SharedProtocol 개발자 기술면접 맵

이 문서는 현재 프로젝트에 축적된 16개 DevThread 문서와 확인 가능한 커밋 기록만을 기준으로 작성한 마스터 인덱스다. 파일·함수·클래스는 프로젝트 기록에서 이름이 확인된 경우에만 적었다. 서비스 내부 구현이 문서에 없을 때는 계약과 아키텍처 원칙만 선별하고 실제 구현 위치를 추측하지 않았다.

## 상세 워크북 문서

| 파일 | 역할 | 상세 항목 |
| --- | --- | --- |
| [`01-value-objects.md`](01-value-objects.md) | 수치 정확성, 값 동등성, overflow, 표시 변환 | [I01 Money](01-value-objects.md#i01), [I02 Odds](01-value-objects.md#i02) |
| [`02-domain-state-and-invariants.md`](02-domain-state-and-invariants.md) | sealed type, 복합 객체 불변식, 정산 상태 조합 | [I03 BetSlipType](02-domain-state-and-invariants.md#i03), [I04 BetSlip 구조](02-domain-state-and-invariants.md#i04), [I05 정산 상태](02-domain-state-and-invariants.md#i05) |
| [`03-schema-errors-and-boundaries.md`](03-schema-errors-and-boundaries.md) | Avro 변경 안전성, 오류 어휘, 표현·소유권 경계 | [I06 Wire V1](03-schema-errors-and-boundaries.md#i06), [I07 오류 모델](03-schema-errors-and-boundaries.md#i07), [I08 계약 소유권](03-schema-errors-and-boundaries.md#i08) |
| [`04-distributed-delivery-and-revisions.md`](04-distributed-delivery-and-revisions.md) | 멱등 요청, 동기·비동기 경계, at-least-once, 전달 간극, 정산 정정 | [I09 멱등 요청](04-distributed-delivery-and-revisions.md#i09), [I10 토폴로지](04-distributed-delivery-and-revisions.md#i10), [I11 멱등 consumer](04-distributed-delivery-and-revisions.md#i11), [I12 Redis·Kafka](04-distributed-delivery-and-revisions.md#i12), [I13 정산 정정](04-distributed-delivery-and-revisions.md#i13) |

## 우선순위 기준

- **S**: 질문과 10~30분 직접 구현 모두 가치가 높아 반드시 준비할 지점
- **A**: 질문 가치가 높고, 핵심 일부를 직접 구현하거나 설계 산출물로 검증하기 좋은 지점
- **B**: 개별 구현보다 계약 의미, 경계, trade-off 설명이 중요한 지점
- **C**: boilerplate·반복·설정 중심이거나 다른 대표 문제에 이미 흡수되어 별도 준비 효율이 낮은 지점

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C | 01 | `build(maven): initialize Java 17 library`<br>`build(wrapper): add Maven wrapper` | `pom.xml`, `.mvn/wrapper/maven-wrapper.properties`, `mvnw`, `mvnw.cmd` | 재현 가능한 빌드 초기화 | 기본 프로젝트 bootstrap과 wrapper 스크립트 자체는 면접 변별력이 낮다. wrapper 사용 이유는 설명 정도면 충분하다. | 낮음 | 낮음 | 15 |
| B | 01 | `build(avro): generate protocol records` | `pom.xml`, `src/main/avro`, `target/generated-sources/avro` | generated source lifecycle과 빌드 phase | schema가 source of truth이고 generated code를 직접 수정하지 않는 원칙은 중요하지만, 플러그인 XML 작성 자체는 구현 문제 가치가 낮다. I06의 전제에 통합한다. | 중간 | 낮음 | 02, 14, 16 |
| C | 01 | `build(format): enforce Java formatting`<br>`build(checkstyle): enforce static analysis` | `pom.xml`, `config/checkstyle/checkstyle.xml` | formatting과 semantic static analysis 분리 | 팀 운영상 유용하지만 규칙 설정 암기보다 실제 도메인·분산 문제의 면접 가치가 높다. | 낮음 | 낮음 | 15 |
| B | 01 | `ci(protocol): verify Java 17 builds`<br>`build(release): release shared protocol 1.0.0` | `.github/workflows/ci.yml`, `pom.xml` | 검증 gate와 공통 artifact 릴리스 | `clean verify`를 release gate로 두는 판단은 설명 가치가 있으나, 독립 코딩 문제로 만들 필요는 낮다. I06·I08과 연계한다. | 중간 | 낮음 | 02, 15, 16 |
| A | 02 | `build(avro): generate protocol records`<br>`feat(events): define wire monetary amounts`<br>`test(events): verify wire monetary amounts` | `Money.avsc`, `AvroRecordTestSupport.java`, `MoneyRecordTest.java`; `assertFields`, `roundTrip` | Avro SpecificRecord binary round trip | schema·generated class·binary codec의 경계를 직접 설명하고 작은 테스트 지원 코드를 구현하기 좋다. [I06]에 통합한다. | 높음 | 중간 | 01, 03, 10, 13, 14 |
| S | 02 | `build(avro): order shared named schemas`<br>`test(events): lock wire v1 schema inventory`<br>`test(events): lock wire v1 fingerprints` | `pom.xml`, `WireSchemaTestSupport.java`, `WireSchemaInventoryTest.java`, `WireSchemaFingerprintTest.java`; `loadSchemas`, `schemaOrder` | Registry 없는 Avro V1 변경 안전성 | writer schema id가 없는 mixed deployment에서 필드 순서·named type·default·logical type·fingerprint를 함께 잠그는 판단은 프로젝트 밖에서도 일반화된다. [I06] | 높음 | 높음 | 01, 10, 11, 12, 13, 14, 16 |
| C | 03 | `feat(money): define supported currencies`<br>`test(money): verify supported currencies` | `Currency.java`, `CurrencyTest.java` | 닫힌 통화 집합 | enum 두 개 정의 자체는 단순하다. 통화 판별자의 필요성은 Money 대표 문제 [I01]에 흡수한다. | 낮음 | 낮음 | 15, 16 |
| S | 03 | `feat(money): define monetary amounts`<br>`test(money): verify arithmetic and currency safety` | `Money.java`, `MoneyArithmeticTest.java`; `add`, `subtract`, `multiply`, `negate`, `compareTo` | minor unit, 통화 안전성, exact arithmetic | cross-currency 차단, overflow, 음수 허용 책임 경계까지 한 번에 확인할 수 있는 대표 값 객체 문제다. [I01] | 높음 | 높음 | 02, 10, 13, 14, 15, 16 |
| A | 03 | `test(money): verify monetary JSON shape` | `MoneyJsonTest.java`, `Money.java` | helper method의 JSON 누출 방지 | 언어 객체의 편의 API가 wire shape를 오염시킬 수 있다는 경계 문제다. 독립 문제 대신 [I01]에 통합한다. | 높음 | 중간 | 09, 15, 16 |
| S | 04 | `feat(odds): define normalized decimal odds`<br>`test(odds): verify decimal odds invariants` | `Odds.java`, `OddsTest.java`; `equals`, `hashCode`, `ofDecimal` | BigDecimal scale, equality/hashCode 계약 | `compareTo` 기반 동등성과 hash 정규화, rounding, 최소값 검증은 Java 핵심 원리와 수치 정확성을 함께 묻기 좋다. [I02] | 높음 | 높음 | 08, 12, 15, 16 |
| A | 04 | `feat(odds): convert display formats`<br>`test(odds): verify American and fractional conversions` | `Odds.java`, `OddsConversionTest.java`; `toAmerican`, `toFractional`, `ofAmerican` | 수치 변환·반올림·기약분수·경계값 | 공식 암기보다 분기, 반올림, gcd, 1.0000 같은 경계 처리 판단을 검증할 수 있다. [I02]에 통합한다. | 높음 | 높음 | 12, 15, 16 |
| B | 05 | `feat(identity): define event identities`<br>`test(identity): verify event identity invariants` | `EventId.java`, `MarketId.java`, `SelectionId.java`, `EventIdentityTest.java` | UUID primitive obsession 제거 | 컴파일 타임 타입 안전성과 canonical JSON은 좋은 설명 주제지만 구현은 반복 record 작성에 가깝다. | 높음 | 낮음 | 08, 12, 13, 15, 16 |
| B | 05 | `feat(identity): define account identities`<br>`test(identity): verify account identity invariants` | `BetId.java`, `UserId.java`, `AccountIdentityTest.java` | 동일 원시값의 의미 타입 분리 | 앞 행과 같은 역량을 반복한다. EventId 계열을 대표로 보고 연관 위치로 묶는다. | 중간 | 낮음 | 06, 08, 10, 13 |
| S | 06 | `feat(idempotency): define request keys`<br>`test(idempotency): verify canonical request keys`<br>`test(idempotency): reject malformed request keys` | `IdempotencyKey.java`, `IdempotencyKeyTest.java`, `IdempotencyKeyValidationTest.java` | wire key 검증과 durable idempotency의 차이 | 문자열 검증, payload fingerprint, 결과 재생, unique constraint race를 연결해 API·DB·분산 처리 기본기를 확인할 수 있다. [I09] | 높음 | 높음 | 10, 13, 15, 16 |
| A | 07 | `feat(bet): classify bet slips`<br>`test(bet): verify slip type invariants` | `BetSlipType.java`, `BetSlipTypeTest.java`; `Single`, `Multiple`, `System` | sealed hierarchy와 K-of-N 모델링 | 유효하지 않은 nullable 조합을 타입으로 제거하고 조합 폭발 상한과 정책 경계를 설명하기 좋다. [I03] | 높음 | 높음 | 08, 13, 15, 16 |
| B | 08 | `test(bet): verify state semantics` | `MarketType.java`, `BetStatus.java`, `SettlementResult.java`, `DomainEnumsTest.java` | 도메인 어휘 집합 고정 | enum vocabulary drift 방지는 중요하지만 값 집합 암기는 면접 변별력이 낮다. 상태 조합 문제 [I05]의 배경으로 사용한다. | 중간 | 낮음 | 09, 13, 14 |
| C | 08 | `feat(bet): classify bet slips` | `BetSlipType.java` | Thread 07 역량 반복 | 동일 구현을 다시 독립 문제로 만들지 않고 Thread 07을 대표로 선택했다. [I03] 연관 위치다. | 낮음 | 낮음 | 07 |
| B | 08 | `feat(bet): define bet selections`<br>`test(bet): verify selection invariants` | `BetSelection.java`, `BetSelectionTest.java` | placement snapshot과 의도적 denormalization | odds·market type snapshot을 보관하는 이유는 설명 가치가 있으나 record null 검증 자체는 단순하다. [I04]의 구성요소로 통합한다. | 높음 | 낮음 | 04, 12, 13 |
| S | 08 | `feat(bet): compose self-consistent slips`<br>`test(bet): verify structural slip invariants` | `BetSlip.java`, `BetSlipStructureTest.java` | aggregate 구조 불변식과 정책 경계 | 타입별 선택 수, 양수 stake, 필수 필드, 구조 규칙과 업무 정책 분리를 한 문제에서 검증할 수 있다. [I04] | 높음 | 높음 | 03, 04, 07, 13, 15, 16 |
| S | 08 | `test(bet): verify settlement slip invariants` | `BetSlip.java`, `BetSlipSettlementTest.java` | 상태 의존 nullable 조합 제거 | SETTLED·결과·시각·payout의 합법 조합과 VOID/VOIDED 의미를 묻는 대표 상태 invariant 문제다. [I05] | 높음 | 높음 | 13, 14, 16 |
| A | 08 | `test(bet): isolate slip selection state` | `BetSlipIsolationTest.java`, `BetSlip.java` | record의 shallow immutability와 defensive copy | record도 가변 collection alias를 가질 수 있다는 Java 핵심 원리를 직접 테스트하기 좋다. [I04]에 통합한다. | 높음 | 중간 | 07, 15 |
| A | 09 | `feat(errors): define protocol error codes`<br>`test(errors): verify error classifications`<br>`test(errors): verify retry semantics` | `ErrorCode.java`, `ErrorCodeTest.java`, `ErrorRetryTest.java` | stable error code와 retry 의미 | status·type URI·stable code, 업무 거절과 일시 장애의 구분은 API 경계와 복구 판단을 확인하기 좋다. [I07] | 높음 | 중간 | 06, 11, 15, 16 |
| A | 09 | `feat(errors): define problem details`<br>`test(errors): verify problem detail invariants` | `ProblemDetail.java`, `ProblemDetailTest.java`, `ErrorCode.toProblemDetail` | 프레임워크 중립 오류 계약 | spring-web 전이 의존성을 피하고 비웹 consumer까지 쓰는 공통 표현을 설계한 trade-off가 명확하다. [I07] | 높음 | 중간 | 15, 16 |
| B | 10 | `feat(events): define wallet debit confirmations`<br>`test(events): verify wallet debit confirmations` | `WalletDebited.avsc`, `WalletDebitedRecordTest.java` | 원장 commit 이후 outcome event | 개별 record 정의보다 ledgerTxId·idempotencyKey·outbox 의미를 설명하는 가치가 높다. [I11] 연관 위치다. | 높음 | 낮음 | 06, 13, 16 |
| B | 10 | `feat(events): define wallet credit confirmations`<br>`test(events): verify wallet credit confirmations` | `WalletCredited.avsc` | credit outcome 계약 | debit과 동일 역량이 반복된다. wallet outcome 이벤트 묶음으로 설명한다. | 중간 | 낮음 | 13, 14, 16 |
| B | 10 | `feat(events): define wallet debit failures`<br>`test(events): verify wallet debit failures` | `WalletDebitFailed.avsc` | 성공·실패 event 의미 분리 | 실패를 사실 이벤트로 보존하는 설계는 유용하지만 독립 구현 문제로는 record boilerplate 비중이 크다. | 중간 | 낮음 | 09, 13, 16 |
| B | 11 | `feat(events): define risk limit signals`<br>`test(events): verify risk limit signals` | `RiskLimitViolated.avsc` | hard rejection과 관측 event | 승인 경로의 한도 거절과 비동기 신호를 구분하는 설명 주제다. [I10] 연관 위치다. | 높음 | 낮음 | 09, 13, 16 |
| B | 11 | `feat(events): define risk pattern signals`<br>`test(events): verify risk pattern signals` | `RiskPatternSuspected.avsc` | heuristic signal과 best-effort 전달 | 탐지 신호를 hard rejection으로 취급하지 않는 판단과 delivery class 차이가 중요하다. [I12]에 연계한다. | 높음 | 낮음 | 12, 16 |
| B | 12 | `feat(events): define market status changes`<br>`test(events): verify market status changes` | `MarketStatusChanged.avsc` | 상태 변화 사실 이벤트 | previous/new snapshot과 event key 의미는 설명 가치가 있으나 개별 record 구현은 반복적이다. | 중간 | 낮음 | 13, 16 |
| B | 12 | `feat(events): define event lifecycle changes`<br>`test(events): verify lifecycle changes` | `EventLifecycle.avsc` | 경기 단계와 downstream trigger | lifecycle과 결과 데이터를 분리한 의미는 [I10]에서 MatchResult와 함께 묻는 편이 효율적이다. | 높음 | 낮음 | 13, 14, 16 |
| B | 12 | `feat(events): define odds changes`<br>`test(events): verify odds changes` | `OddsChanged.avsc` | 동기 Redis projection과 비동기 push 계약 | record 필드보다 Redis·Kafka 전달 간극이 핵심이므로 [I12]로 통합한다. | 높음 | 낮음 | 04, 16 |
| A | 12 | `feat(events): define match results`<br>`test(events): verify match results` | `MatchResult.avsc` | lifecycle와 settlement input 분리 | phase transition과 정산 데이터가 다른 사실이라는 경계는 event modeling과 역순 처리 질문에 적합하다. [I10]에 통합한다. | 높음 | 중간 | 13, 14, 16 |
| A | 13 | `feat(events): define accepted bet placements`<br>`test(events): verify bet placement records` | `BetPlacedRequested.avsc` | 승인 완료 이벤트, idempotency, outbox | risk·wallet 확인 뒤 durable state와 outbox를 기록하고 at-least-once consumer를 요구하는 핵심 흐름이다. [I09]·[I10]·[I11]에 명시적으로 통합한다. | 높음 | 중간 | 06, 08, 10, 11, 16 |
| A | 13 | `feat(events): define settled bet outcomes`<br>`test(events): verify settled bet outcomes` | `BetSettled.avsc`, `BetSettledRecordTest.java` | 정산 완료 snapshot과 downstream projection | settlement result·stake·payout·시각의 사실 계약은 상태 invariant와 revision 0의 기반이다. [I10]·[I13]에 통합한다. | 높음 | 중간 | 08, 12, 14, 16 |
| A | 13 | `feat(events): define voided bet outcomes`<br>`test(events): verify voided bet outcomes` | `BetVoided.avsc`, `BetVoidedRecordTest.java` | lifecycle void와 settle-time VOID 분리 | 전체 slip 환급 event와 per-selection 정산 결과를 구분하는 모델링 질문 가치가 높다. [I05]·[I10]에 통합한다. | 높음 | 낮음 | 08, 12, 14, 16 |
| C | 14 | `feat(events): define settled bet outcomes`<br>`build(avro): order shared named schemas` | `BetSettled.avsc`, `pom.xml` | Thread 02·13 반복 기반 | named type과 초기 settlement 정의는 필요하지만 대표 면접 포인트는 Thread 02와 13에서 이미 선택했다. | 낮음 | 낮음 | 02, 13 |
| S | 14 | `feat(events): define resolution revision snapshots`<br>`test(events): verify revision schema contract`<br>`test(events): verify payout increase revisions`<br>`test(events): verify payout decrease revisions` | `BetResolutionRevised.avsc`, `BetResolutionRevisedSchemaTest.java`, `BetResolutionPayoutIncreaseTest.java`, `BetResolutionPayoutDecreaseTest.java` | full snapshot revision, 중복·역순 reducer | revision 단조성, duplicate/conflict, delta 대신 replacement snapshot, 증가·감소 보정은 분산 상태 처리의 대표 구현 문제다. [I13] | 높음 | 높음 | 02, 10, 12, 13, 16 |
| C | 15 | `build(maven): initialize Java 17 library`<br>`feat(money): define monetary amounts`<br>`feat(events): define wire monetary amounts`<br>`feat(idempotency): define request keys`<br>`feat(bet): compose self-consistent slips`<br>`feat(errors): define problem details` | 앞선 Thread에서 확인된 동일 파일·클래스 | 횡단 관점에서 반복된 구현 이력 | Thread 15의 가치는 개별 구현 반복보다 표현·소유권을 하나의 관점으로 묶은 문서에 있다. 각 구현은 I01·I04·I06·I07·I09의 연관 위치다. | 낮음 | 낮음 | 02, 03, 06, 08, 09 |
| A | 15 | `docs(project): document shared protocol` | `README.md`, `architecture/contract-ownership-and-representation-boundaries.md` | 최소 shared kernel과 Java·JSON·Avro 경계 | 공통 계약과 서비스 정책을 분리하고 framework·DB·Kafka runtime 의존을 배제한 설계 판단은 시스템 경계 면접에 적합하다. [I08] | 높음 | 중간 | 02, 03, 04, 06, 08, 09, 13, 16 |
| C | 16 | wallet·risk·bet·event schema를 정의한 앞선 `feat(events): ...` 커밋들 | Thread 10~14에서 확인된 `.avsc`와 record tests | 개별 이벤트 계약의 종합 반복 | 개별 record는 대표 Thread 10~14에서 이미 평가했다. Thread 16에서는 토폴로지와 보장 차이를 대표로 선택한다. | 낮음 | 낮음 | 10, 11, 12, 13, 14 |
| A | 16 | `docs(project): document shared protocol` | `architecture/event-flow-and-consumer-map.md` | 동기 승인과 비동기 사실 전파 | 사용자 승인 조건과 후속 projection을 분리하고 명령과 이벤트를 구분하는 시스템 설계 질문 가치가 높다. [I10] | 높음 | 중간 | 10, 11, 12, 13, 14 |
| S | 16 | `docs(project): document shared protocol` | `architecture/event-flow-and-consumer-map.md` | outbox, redelivery, 멱등 consumer | side effect와 offset 사이 장애, event identity, no exactly-once assumption은 분산 처리의 핵심 구현·설명 지점이다. [I11] | 높음 | 높음 | 06, 10, 11, 12, 13, 14 |
| A | 16 | `docs(project): document shared protocol` | `architecture/event-flow-and-consumer-map.md` | Redis projection과 Kafka delivery gap | Redis write·Kafka publish의 부분 성공, queue ack 시점, readiness를 다루는 운영·복구 판단이 뚜렷하다. [I12] | 높음 | 중간 | 11, 12, 15 |
| A | 16 | `docs(project): document shared protocol` | `architecture/event-schema-evolution.md` | fixed Wire V1의 변경 절차 | 기존 record 수정 대신 새 record/topic, named schema 재사용, fingerprint 한계를 정리한 설명은 [I06]에 통합한다. | 높음 | 중간 | 01, 02, 10, 11, 12, 13, 14 |
| A | 16 | `docs(project): document shared protocol` | `README.md`, `architecture/contract-ownership-and-representation-boundaries.md` | 공통 라이브러리 책임의 음수 경계 | repository·Spring component·Kafka client·DB model을 넣지 않는 원칙은 [I08]의 연관 설명으로 통합한다. | 높음 | 낮음 | 15 |

## 대표 면접 포인트와 연관 Thread

| ID | 우선순위 | 대표 Thread·커밋 | 대표 역량 | 연관 Thread | 상세 위치 |
| --- | --- | --- | --- | --- | --- |
| I01 | S | Thread 03 / `feat(money): define monetary amounts` | 통화 안전 금액, overflow, wire shape | 02, 10, 13, 14, 15, 16 | [`01-value-objects.md#i01`](01-value-objects.md#i01) |
| I02 | S | Thread 04 / `feat(odds): define normalized decimal odds` | BigDecimal equality/hash, 반올림, 배당 변환 | 08, 12, 15, 16 | [`01-value-objects.md#i02`](01-value-objects.md#i02) |
| I03 | A | Thread 07 / `feat(bet): classify bet slips` | sealed type, tagged union, K-of-N | 08, 13, 15, 16 | [`02-domain-state-and-invariants.md#i03`](02-domain-state-and-invariants.md#i03) |
| I04 | S | Thread 08 / `feat(bet): compose self-consistent slips` | aggregate 구조 불변식, defensive copy | 03, 04, 07, 13, 15, 16 | [`02-domain-state-and-invariants.md#i04`](02-domain-state-and-invariants.md#i04) |
| I05 | S | Thread 08 / `test(bet): verify settlement slip invariants` | 상태 의존 필드 조합, VOID 의미 | 13, 14, 16 | [`02-domain-state-and-invariants.md#i05`](02-domain-state-and-invariants.md#i05) |
| I06 | S | Thread 02 / `test(events): lock wire v1 fingerprints` | Registry 없는 Avro 계약 잠금 | 01, 10, 11, 12, 13, 14, 16 | [`03-schema-errors-and-boundaries.md#i06`](03-schema-errors-and-boundaries.md#i06) |
| I07 | A | Thread 09 / `feat(errors): define problem details` | stable 오류 어휘, retry 의미, framework 중립 | 06, 11, 15, 16 | [`03-schema-errors-and-boundaries.md#i07`](03-schema-errors-and-boundaries.md#i07) |
| I08 | A | Thread 15 / `docs(project): document shared protocol` | Java·JSON·Avro 표현과 책임 소유권 | 02, 03, 04, 06, 08, 09, 13, 16 | [`03-schema-errors-and-boundaries.md#i08`](03-schema-errors-and-boundaries.md#i08) |
| I09 | S | Thread 06·16 / idempotency key와 문서 | durable 결과 재생, payload conflict, race | 10, 13, 15 | [`04-distributed-delivery-and-revisions.md#i09`](04-distributed-delivery-and-revisions.md#i09) |
| I10 | A | Thread 16 / `docs(project): document shared protocol` | 동기 승인·비동기 전파, 명령·이벤트 분리 | 10, 11, 12, 13, 14 | [`04-distributed-delivery-and-revisions.md#i10`](04-distributed-delivery-and-revisions.md#i10) |
| I11 | S | Thread 16 / `docs(project): document shared protocol` | outbox, at-least-once, 멱등 consumer | 06, 10, 11, 12, 13, 14 | [`04-distributed-delivery-and-revisions.md#i11`](04-distributed-delivery-and-revisions.md#i11) |
| I12 | A | Thread 16 / `docs(project): document shared protocol` | Redis·Kafka 부분 성공과 복구·readiness | 11, 12, 15 | [`04-distributed-delivery-and-revisions.md#i12`](04-distributed-delivery-and-revisions.md#i12) |
| I13 | S | Thread 14 / `feat(events): define resolution revision snapshots` | full snapshot revision reducer | 02, 10, 12, 13, 15, 16 | [`04-distributed-delivery-and-revisions.md#i13`](04-distributed-delivery-and-revisions.md#i13) |

## S/A 완전성 검증

| 선별된 S/A 지점 | 상태 | 상세 워크북 처리 |
| --- | --- | --- |
| Thread 02 Avro round trip·named schema·inventory·fingerprint | 독립 항목으로 작성됨 | I06 |
| Thread 03 Money 산술·JSON shape | 독립 항목 및 통합 | I01에 산술과 wire 누출 경계를 함께 작성 |
| Thread 04 Odds 정규화·표시 변환 | 독립 항목 및 통합 | I02에 equality/hash와 변환을 함께 작성 |
| Thread 06 IdempotencyKey와 durable dedup 의미 | 독립 항목 및 통합 | I09에 wire 검증과 저장소 race를 함께 작성 |
| Thread 07 sealed BetSlipType | 독립 항목으로 작성됨 | I03 |
| Thread 08 BetSlip 구조·selection isolation | 독립 항목 및 통합 | I04에 defensive copy를 통합 |
| Thread 08 정산 필드 invariant | 독립 항목으로 작성됨 | I05 |
| Thread 09 ErrorCode·ProblemDetail | 하나의 대표 항목에 통합 | I07 |
| Thread 12 MatchResult와 lifecycle 의미 분리 | 다른 대표 항목에 명시적으로 통합 | I10 |
| Thread 13 accepted placement | 세 대표 항목에 명시적으로 통합 | I09, I10, I11 |
| Thread 13 settled·voided outcome | 대표 상태·토폴로지·revision 항목에 통합 | I05, I10, I13 |
| Thread 14 resolution revision | 독립 항목으로 작성됨 | I13 |
| Thread 15 계약 소유권 | 독립 항목으로 작성됨 | I08 |
| Thread 16 동기·비동기 topology | 독립 항목으로 작성됨 | I10 |
| Thread 16 outbox·redelivery·no exactly-once | 독립 항목으로 작성됨 | I11 |
| Thread 16 Redis·Kafka delivery gap | 독립 항목으로 작성됨 | I12 |
| Thread 16 schema evolution 규칙 | 다른 대표 항목에 명시적으로 통합 | I06 |
| Thread 16 공통 라이브러리 음수 경계 | 다른 대표 항목에 명시적으로 통합 | I08 |

**검증 결과:** 위 선별표의 모든 S/A 지점은 I01~I13 중 하나의 독립 상세 항목으로 작성되었거나, 대표 항목에 명시적으로 통합되었다. 상세 문서에 연결되지 않은 S/A 지점은 없다.

## 백지 구현 우선순위

1. **I13** — revision number 기반 full snapshot reducer: 중복·역순·충돌을 한 문제에서 검증
2. **I04** — BetSlip 구조 invariant와 defensive copy: Java 객체 모델링·상태 안전성의 대표 문제
3. **I09** — 멱등 요청 판정: key·payload fingerprint·durable race 연결
4. **I06** — Avro round trip과 계약 잠금: I/O·resource·schema 변경 경계
5. **I05** — 상태별 정산 필드 조합 검증: illegal state 차단
6. **I01** — Money exact arithmetic: overflow와 cross-currency 실패
7. **I02** — Odds equality/hash와 표시 변환: BigDecimal·반올림·gcd
8. **I11** — 멱등 consumer handler: side effect·dedup marker·offset 경계
9. **I03** — sealed K-of-N 타입: 닫힌 모델과 입력 경계
10. **I07** — stable ErrorCode와 ProblemDetail: API 경계 구현

## 설명 연습 우선순위

1. **I06** — 왜 optional field 추가도 안전하지 않은지, fingerprint가 충분하지 않은 이유
2. **I11** — side effect 후 offset 전 장애와 exactly-once 적용 경계
3. **I10** — 승인 조건은 동기, 후속 사실 전파는 비동기인 이유
4. **I13** — delta가 아니라 full snapshot을 택한 이유와 같은 revision 충돌
5. **I08** — shared kernel에 넣을 것과 서비스에 남길 것의 기준
6. **I09** — wire key 검증과 실제 durable idempotency의 차이
7. **I12** — Redis 성공·Kafka 실패의 관측 차이와 ack/readiness 설계
8. **I04·I05** — 구조적 invariant와 업무 정책, 상태별 nullable 조합
9. **I07** — stable machine code, HTTP status, retry 의미, 프레임워크 중립성
10. **I02·I01·I03** — 수치 표현과 닫힌 타입 모델의 trade-off

## 한 문제로 통합한 Thread 묶음

1. **금액의 domain·JSON·Avro 표현**: Thread 03을 대표로 Thread 02·10·13·14·15·16을 I01·I06·I08에 연결
2. **배당 값과 전달 snapshot**: Thread 04를 대표로 Thread 08·12·15·16을 I02에 연결
3. **베팅 조합과 상태 invariant**: Thread 07·08을 대표로 Thread 13·14·15·16을 I03·I04·I05에 연결
4. **요청 멱등성과 이벤트 멱등성**: Thread 06을 시작점으로 Thread 10·13·16을 I09·I11에 연결
5. **Wire V1 계약 검증**: Thread 02를 대표로 Thread 01·10·11·12·13·14·16을 I06에 통합
6. **공통 오류와 경계 소유권**: Thread 09·15·16을 I07·I08로 통합
7. **승인·outbox·consumer topology**: Thread 10·11·12·13·16을 I10·I11로 통합
8. **Redis projection과 Kafka fan-out 간극**: Thread 11·12·16을 I12로 통합
9. **초기 정산과 정정 revision**: Thread 02·13·14·16을 I13으로 통합
