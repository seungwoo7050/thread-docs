# Sportsbook Settlement 개발자 기술면접 마스터 맵

## 사용 범위와 증거 경계

이 워크북은 현재 GPT 프로젝트에 축적된 Thread 1~22 학습 문서와 NT-23 External-State Evidence Packet에서 직접 확인된 정보만 사용한다. 원격 저장소, 실제 배포 환경, 운영 secret, production database 상태는 사용하지 않았다. 파일·클래스·메서드 이름은 프로젝트 기록에서 확인된 경우에만 적었고, 외부 상태의 실제 생성 여부가 확인되지 않은 NT-23은 설명형 B 항목으로만 분류했다.

## 우선순위 기준

| 우선순위 | 기준 |
| --- | --- |
| S | 질문과 10~30분 백지 구현 모두 가치가 높고, 상태·금액·동시성·복구의 핵심 invariant를 직접 검증할 수 있다. |
| A | 준비 가치가 높다. 질문 가능성이 높거나 축소 구현으로 핵심 판단을 확인하기 좋다. |
| B | 별도 구현 문제보다 설계 선택, 운영 경계, 통합 구조를 설명하는 준비가 중요하다. |
| C | boilerplate·반복 wiring·단순 설정·릴리스 표면에 가까워 별도 면접 항목으로 만들 필요가 낮다. |

## 상세 워크북 파일

| 파일 | 역할 | 포함 항목 |
| --- | --- | --- |
| [01-contracts-and-state.md](01-contracts-and-state.md) | aggregate 전이, placement 계약, Kafka ack/DLT, 순서 역전, lifecycle latch | P01~P06 |
| [02-algorithms-and-money.md](02-algorithms-and-money.md) | K-of-N 조합, payout 반올림, 금액 보존식, Wallet 응답 판정 | P07~P10 |
| [03-base-settlement-and-messaging.md](03-base-settlement-and-messaging.md) | accepted result fanout, durable attempt, lease 복구, outbox, batch 실패 격리 | P11~P15 |
| [04-result-candidates-and-revisions.md](04-result-candidates-and-revisions.md) | 결과 후보 fingerprint, causal CAS, immutable revision plan, 상태 기계 | P16~P19 |
| [05-correction-proof-and-recovery.md](05-correction-proof-and-recovery.md) | adjustment proof, GET-first recovery, 원자적 revision 확정 | P20~P22 |
| [06-admin-security-and-runtime.md](06-admin-security-and-runtime.md) | 관리자 멱등성, 내부 인증, worker 격리와 종료 | P23~P25 |

## 전체 Thread·커밋 선별 결과

| ID | 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| P01 | A | 1 | `feat(bet): create pending placement aggregate` | `Bet`, `BetSelection`, `SettlementStatus`<br>`Bet.pending`, `recordSettled`, `recordVoided` | aggregate terminal 전이와 result/status 분리 | 단방향 상태 전이, 통화·payout invariant, 결과 `VOID`와 전체 슬립 `VOIDED`의 차이는 프로젝트 밖에서도 일반화된다. 동시성 자체는 저장소 경계와 함께 설명해야 한다. | 높음 | 중간 | 5, 9, 10 | [P01](01-contracts-and-state.md#p01) |
| P02 | S | 2 | `feat(placement): persist exact replays idempotently` | `BetPlacementValidator`, `BetPlacementFingerprinter`, `BetReadModelWriter.record` | 경계 검증과 semantic replay 멱등성 | 동일 키 재사용을 exact replay와 semantic conflict로 나누고 canonical form·hash·유일 제약을 연결한다. 직접 구현과 설계 질문 모두 강하다. | 높음 | 높음 | 1, 4, 13, 16 | [P02](01-contracts-and-state.md#p02) |
| P03 | A | 2 | `test(placement): verify listener key and ack boundaries` | `BetPlacedListener.receive`, `StrictAvroDecoder`, `KafkaUuidKeyValidator` | raw key·payload identity·manual ack 순서 | key/payload 계약, durable write, catch-up, acknowledgment의 순서가 명확하다. 프레임워크 암기보다 실패 시 offset을 언제 남길지 판단을 묻기 좋다. | 높음 | 높음 | 3, 4, 12 | [P03](01-contracts-and-state.md#p03) |
| P04 | A | 3 | `feat(kafka): wire retry recovery handler` | `ExactDeadLetterRecoverer`, `KafkaRecoveryConfiguration`, `RawKafkaProducerConfiguration`, `MessageFailureClassifier` | exact raw DLT와 bounded retry | 원본 bytes·partition·headers를 증거로 보존하고, DLT broker ack 후에만 recovered commit하는 경계는 오류 처리·I/O·interrupt 기본기를 함께 확인한다. | 높음 | 높음 | 2, 8, 11 | [P04](01-contracts-and-state.md#p04) |
| P05 | A | 4 | `feat(lifecycle): catch up tombstones before results` | `BetPlacedListener`, `LifecycleStore.findTombstone`, `AcceptedResultRepository.findByEventId`, `LifecycleFanout`, `ResultFanout` | topic 간 순서 역전과 replay catch-up | 서로 다른 topic의 전역 순서 부재, durable early fact, lifecycle-first pass, exact replay 재사용을 하나의 문제로 묻는다. | 높음 | 높음 | 2, 5, 9, 10 | [P05](01-contracts-and-state.md#p05) |
| P06 | A | 5 | `feat(lifecycle): capture semantic observations` | `LifecycleFingerprinter`, `LifecycleObservation`, `LifecycleStore.record`, `LifecycleFanout` | first-terminal latch와 전체 노출액 환불 | observation dedup과 decision tombstone을 구분하고, concurrent first-wins와 SYSTEM total exposure를 연결한다. 상태·동시성 경계 면접 가치가 높다. | 높음 | 높음 | 4, 6, 7, 10, 19 | [P06](01-contracts-and-state.md#p06) |
| P07 | S | 6 | `feat(resolver): expand deterministic system combinations` | `SettlementLineFactory.lines`, `SettlementLine`, `ResolvedSelection` | K-of-N 결정적 조합 생성 | backtracking, 중복 방지, 안정된 순서, defensive copy, `O(C(N,K)×K)`를 직접 구현으로 검증하기 좋다. | 높음 | 높음 | 5, 7, 12 | [P07](02-algorithms-and-money.md#p07) |
| P08 | S | 6 | `feat(resolver): calculate unit stake line payouts` | `SettlementPayoutCalculator.calculate`, `PayoutCalculation`, `SettlementResolver.resolve` | line 합산 후 한 번만 반올림 | LOST/WON/PUSH/VOID 의미, decimal 계산, line별 rounding bias와 overflow를 함께 묻는 금액 알고리즘 핵심이다. | 높음 | 높음 | 7, 12, 14 | [P08](02-algorithms-and-money.md#p08) |
| P09 | S | 7 | `feat(settlement): enforce money plan conservation` | `WalletMovementPlanner.plan`, `SettlementMoneyPlan`, `SettlementWalletExecutor` | stake·payout 보존식과 movement 순서 | 두 보존식, SYSTEM exposure, zero movement 생략, release→forfeit→profit 순서와 안정된 key를 통해 금액 invariant와 외부 효과를 직접 검증한다. | 높음 | 높음 | 5, 6, 8, 10, 14 | [P09](02-algorithms-and-money.md#p09) |
| P10 | S | 8 | `feat(wallet): classify dependency failures` | `WalletClient`, `WalletFailurePolicy`, `WalletHttpProperties`, `WalletHttpConfiguration` | HTTP failure taxonomy와 exact proof | 429/5xx, permanent non-2xx, malformed 2xx, unexpected status, bounded timeout을 transport truth와 business proof로 분리한다. | 높음 | 높음 | 3, 7, 15, 18 | [P10](02-algorithms-and-money.md#p10) |
| P11 | A | 9 | `feat(result): prepare accepted result claims` | `AcceptedResultRepository`, `Bet.applyAcceptedResult`, `ResultSettlementPreparer.prepare`, `BetRepository.findResultActionableIdsByEvent` | accepted projection과 concurrent one-attempt claim | 여러 event가 마지막 selection을 동시에 해결하는 race에서 row lock·재검증·bet당 하나의 attempt를 연결한다. | 높음 | 높음 | 4, 10, 13 | [P11](03-base-settlement-and-messaging.md#p11) |
| P12 | S | 10 | `feat(settlement): claim pending bets atomically` | `SettlementAttemptDraft`, `SettlementAttempt`, `SettlementAttemptRepository.claimPending`, `V5__settlement_attempt.sql` | Wallet 전 immutable execution intent | DB와 외부 Wallet 사이 crash ambiguity를 durable plan으로 해결한다. action·금액·lease constraint와 one-attempt-per-bet는 직접 구현·설명 모두 핵심이다. | 높음 | 높음 | 5, 7, 9, 14, 19 | [P12](03-base-settlement-and-messaging.md#p12) |
| P13 | S | 10·19 | `feat(recovery): claim ordered attempt batches`<br>`feat(settlement): claim attempts with database time` | `SettlementAttemptRepository.claimRecoveryBatch`, `consumeLease`, `SettlementRecoveryRow`, `SettlementAttemptRecovery` | DB-time owner-fenced lease와 `SKIP LOCKED` | 긴 I/O 밖의 짧은 claim transaction, clock skew 제거, token+expiry stale-owner fence, bounded batch를 묻는 대표 동시성·복구 문제다. | 높음 | 높음 | 15, 19, 20 | [P13](03-base-settlement-and-messaging.md#p13) |
| P14 | S | 11·10·19 | `feat(outbox): publish locked pending events`<br>`fix(settlement): fence finalization with database time` | `SettlementFinalizer`, `OutboxEventRepository.lockNextUnpublished`, `OutboxPublisher.publishBatch`, `SettlementEventFactory` | 원자적 finalization과 at-least-once outbox | domain state와 publication intent는 원자화하지만 broker 전송은 중복 가능하다는 정확한 crash window를 설명·구현할 수 있다. | 높음 | 높음 | 10, 19, 20 | [P14](03-base-settlement-and-messaging.md#p14) |
| P15 | A | 10 | `feat(recovery): release failed attempts safely` | `SettlementExecutionRunner`, `SettlementAttemptRepository.releaseForRecovery`, `SettlementAttemptRecovery` | item 실패 격리와 안전한 lease 반환 | 한 dependency 실패가 batch 전체를 굶기지 않게 하고, exact token으로만 release하며, durable error를 sanitize하는 판단을 확인한다. | 높음 | 높음 | 8, 12, 15, 20, 21 | [P15](03-base-settlement-and-messaging.md#p15) |
| P16 | A | 13 | `feat(correction): fingerprint semantic resolutions` | `ResultCandidate`, `ResultCandidateFingerprinter`, `ResultCandidateIntake.ingest`, `ResultCandidateStore.record` | immutable result candidate와 decision taxonomy | overwrite 대신 evidence를 쌓고 map order를 제거한 semantic hash, `VOIDED` mode identity, replay·future·late·stale 상태를 구분한다. | 높음 | 높음 | 2, 16, 19 | [P16](04-result-candidates-and-revisions.md#p16) |
| P17 | S | 13·16·19 | `feat(correction): enforce candidate causal order` | `ResultCandidateStore.replaceAccepted`, `acceptFirst`, `lockForAdmin`, `AdminCandidateApproval` | predecessor CAS와 단조 sequence | expected accepted ID, replaces ID, sequence, event, pending state, DB-time future fence를 한 atomic predicate로 묶어 lost update와 인과 역전을 막는다. | 높음 | 높음 | 16, 19 | [P17](04-result-candidates-and-revisions.md#p17) |
| P18 | S | 14·19 | `feat(correction): persist plans before wallet` | `RevisionTarget`, `RevisionPlan.allocate`, `RevisionSnapshot`, `RevisionPlanRepository.persist`, `CorrectionRevisionPreparer` | immutable correction snapshot과 delta | 이미 지급된 금액 변경에서 previous/new 상태, source candidate, revision ID, selections를 고정하고 retry가 같은 plan을 사용하게 한다. | 높음 | 높음 | 13, 15, 17, 19 | [P18](04-result-candidates-and-revisions.md#p18) |
| P19 | A | 14 | `feat(correction): define revision lifecycle` | `RevisionState.canTransitionTo`, `RevisionPlan`, `RevisionPlanRepository`, `settlement_revision` constraints | revision 상태 기계와 DB invariant | `PENDING/BLOCKED/EXHAUSTED/APPLIED/REJECTED` 의미와 lease·proof·schedule shape를 앱과 DB에서 이중 방어한다. 구현보다 설계 설명 비중이 조금 더 높다. | 높음 | 중간 | 15, 16, 19, 21 | [P19](04-result-candidates-and-revisions.md#p19) |
| P20 | S | 15·8 | `feat(correction): validate wallet adjustment proofs` | `RevisionProofValidator.requireExact`, `RevisionWalletGateway.submit`, `WalletAdjustmentProof` | adjustment proof identity와 금액 대수 | revision·bet·user·number·previous/new·delta·currency와 status별 proof shape를 모두 맞춰야 금전 확정을 허용한다. | 높음 | 높음 | 8, 14, 19 | [P20](05-correction-proof-and-recovery.md#p20) |
| P21 | S | 15 | `feat(correction): recover ambiguous adjustments` | `RevisionWalletGateway.recoverAmbiguous`, `RevisionExecutionRunner`, `RevisionRecoveryRepository.claimDue`, `RevisionRecoveryScanner` | GET-first ambiguous recovery와 proof-aware exhaustion | timeout 후 blind POST를 금지하고 exact not-found에서만 same-ID POST, BLOCKED proof 보존, 12회 자동 상한을 연결한다. | 높음 | 높음 | 8, 14, 16, 19, 20 | [P21](05-correction-proof-and-recovery.md#p21) |
| P22 | S | 17·15·19 | `feat(correction): finalize revisions atomically` | `RevisionFinalizer.apply`, `RevisionPlanRepository.markApplied`, `BetRepository.findForUpdateById`, `OutboxEventRepository` | stale-target 재검증과 revised event 원자성 | exact Wallet proof만으로 충분하지 않으며 predecessor bet snapshot·lease·DB time을 다시 fence하고 bet/revision/outbox를 함께 커밋한다. | 높음 | 높음 | 11, 13, 14, 15, 19 | [P22](05-correction-proof-and-recovery.md#p22) |
| P23 | S | 16 | `feat(admin): validate idempotent replays` | `V10__admin_action.sql`, `AdminActionRepository`, `AdminActionReplay.requireExact`, `AdminRequestFingerprint` | 운영 명령 request binding과 append-only audit | idempotency key를 semantic request에 bind하고 advisory transaction lock, exact replay, conflicting reuse, same-transaction audit, DB trigger를 함께 검증한다. | 높음 | 높음 | 13, 14, 15, 18, 19 | [P23](06-admin-security-and-runtime.md#p23) |
| P24 | A | 18 | `feat(admin): authenticate internal requests` | `AdminCredentials`, `AdminAuthenticationFilter`, `WalletCredentials`, `WalletAuthenticationHeaders`, `SettlementCredentialPolicy` | 내부 인증과 credential isolation | 정확히 하나의 header, 401/403, constant-time compare, redaction, admin/Wallet secret 분리를 묻는다. 보안 기본기와 경계 설계 가치가 높다. | 높음 | 중간 | 8, 16, 22 | [P24](06-admin-security-and-runtime.md#p24) |
| P25 | A | 20 | `feat(scheduling): isolate settlement workers` | `SettlementWorkerConfiguration`, `CorrectionCatchupScanner`, `RevisionRecoveryScanner`, worker isolation tests | scheduler starvation 격리와 bounded shutdown | flow별 single-thread executor, runtime enablement, delayed/periodic shutdown 정책, interrupt·thread cleanup을 작은 동시성 구현으로 검증할 수 있다. | 높음 | 높음 | 10, 11, 15, 17, 21, 22 | [P25](06-admin-security-and-runtime.md#p25) |
| B01 | B | 12 | `feat(event): consume match result events` | `BetPlacedListener`, `MatchResultListener`, `EventLifecycleListener`, base settlement components | 기본 정산 end-to-end wiring | placement/result/lifecycle listener부터 Wallet·finalizer·outbox까지의 전체 흐름은 설명 가치가 있지만, 핵심 역량은 P03·P05·P08·P09·P11~P15에 이미 분해돼 있다. | 높음 | 낮음 | 2~11, 19 | 통합: P03, P05, P08, P09, P11~P15 |
| B02 | B | 17 | `feat(event): fan out accepted corrections` | `MatchResultListener`, `CorrectionFanout`, `CorrectionRevisionPreparer`, revision execution components | correction end-to-end orchestration | candidate intake부터 revision finalization까지의 연결을 설명하기 좋지만, 독립 면접 역량은 P16~P22에서 더 정확히 평가된다. | 높음 | 낮음 | 13~16, 19 | 통합: P16~P22 |
| B03 | B | 21 | `feat(health): report settlement dependencies` | `SettlementBacklogMetrics`, `SettlementDependenciesHealthIndicator`, `application.yml` readiness group | backlog observability와 readiness 의미 | pending·blocked·exhausted·outbox backlog와 DB reachability를 운영 상태로 해석하는 설명 가치가 있다. 구현은 단순 SQL gauge/health wiring 비중이 높다. | 높음 | 낮음 | 10, 11, 15, 20 | 상세 문서 없음 |
| B04 | B | 22 | `ci(history): guard settlement archive history`<br>`fix(build): package settlement as a Boot JAR` | `.github/scripts/verify-history.sh`, `.github/workflows/settlement-ci.yml`, `pom.xml`, `BuildBaselineTest`, `SettlementHistoryGuardTest` | 재현 가능한 build·실행 artifact·history policy | Java 17, fixed protocol, Maven wrapper, executable Boot JAR, full-history CI의 이유는 설명할 가치가 있으나 핵심 runtime 구현보다 우선순위가 낮다. | 중간 | 낮음 | 18, 20, NT-23 | 상세 문서 없음 |
| B05 | B | NT-23 | `test(integration): verify PostgreSQL schema bootstrap` | Flyway V1·V3~V10, `PostgresSchemaIntegrationTest`, `pom.xml`, datasource·JPA validation 설정 | PostgreSQL provisioning과 Flyway evolution lifecycle | schema/history/checksum·pending migration·JPA validate 경계는 중요하지만 실제 DB 생성·권한·적용 이력은 프로젝트에서 관찰되지 않았다. 설명형으로만 준비한다. | 높음 | 낮음 | 1, 11, 14, 16, 19, 21, 22 | 상세 문서 없음 |
| C01 | C | 3 | `feat(topics): bind settlement topic inventory` | `SettlementTopics`, `application.yml` | topic 이름과 inventory binding | 명시적 topic 계약은 필요하지만 대부분 설정·wiring이며 P03·P04·P14에서 더 일반화된 경계를 이미 다룬다. | 낮음 | 낮음 | 2, 11, 22 | 상세 문서 없음 |
| C02 | C | 21 | `build(observability): enable Prometheus registry` | `pom.xml`, actuator·Prometheus 설정 | metrics registry 활성화 | dependency와 export 설정 자체는 boilerplate 성격이 강하고, 면접 가치는 backlog metric의 의미를 설명하는 B03에 있다. | 낮음 | 낮음 | 20, 22 | 상세 문서 없음 |
| C03 | C | 22 | `build(quality): configure formatting and static analysis` | `pom.xml`, formatting·static analysis plugin 설정 | build quality tooling | 프로젝트 품질에는 유효하지만 도구 설정 암기 비중이 높고 개발 기본기 직접 검증에는 적합하지 않다. | 낮음 | 낮음 | — | 상세 문서 없음 |
| C04 | C | 22 | `build(release): release settlement 1.0.0`<br>`docs(project): document settlement service` | `pom.xml`, `README.md` | release version과 문서 표면 | 릴리스 숫자 변경과 최종 문서화는 프로젝트 맥락 파악에는 유용하지만 별도 면접 문제로 만들 가치가 낮다. | 낮음 | 낮음 | 전 Thread | 상세 문서 없음 |

## 대표 Thread와 연관 Thread 관계

| 묶음 | 대표 면접 포인트 | 대표 Thread | 연관 Thread | 통합 기준 |
| --- | --- | --- | --- | --- |
| G01 | P02 | 2 | 1, 13, 16 | placement·candidate·admin command에 반복되는 semantic idempotency 중 최초 입력 snapshot의 canonical replay를 대표 문제로 삼았다. |
| G02 | P03·P04 | 2·3 | 12, 8, 11 | raw decode/key/manual ack와 retry/DLT publication을 서로 다른 실패 경계로 분리하되 Kafka boundary 묶음으로 배치했다. |
| G03 | P05·P06 | 4·5 | 2, 9, 10, 19 | topic 간 순서 역전 복구와 first-terminal latch는 연결되지만, catch-up orchestration과 tombstone 동시성은 별도 구현 역량이라 두 문제로 유지했다. |
| G04 | P07·P08·P09 | 6·7 | 5, 10, 12, 14 | 조합 생성, payout rounding, Wallet movement 보존식은 계산 단계별로 다른 invariant를 가지므로 하나의 문서에 묶되 문제는 분리했다. |
| G05 | P11~P15 | 9·10·11 | 8, 12, 19, 20, 21 | accepted projection → durable attempt → DB-time lease → atomic outbox → failure isolation의 기본 정산 실행 체인이다. Thread 12의 통합 pipeline은 이 묶음에 흡수했다. |
| G06 | P16·P17 | 13 | 16, 19 | semantic candidate dedup과 causal replacement CAS는 서로 다른 race를 막으므로 별도 문제로 두고, admin approval와 DB time을 P17에 통합했다. |
| G07 | P18·P19 | 14 | 13, 15, 16, 19 | immutable plan 생성과 durable lifecycle invariant를 분리했다. DB-time persistence는 P18에, proof/schedule state는 P19에 연결했다. |
| G08 | P20~P22 | 15·17 | 8, 11, 14, 19, 20 | proof 검증 → ambiguous recovery → atomic finalization의 correction execution chain이다. Thread 17의 전체 pipeline은 이 세 문제와 P16~P19로 흡수했다. |
| G09 | P23·P24 | 16·18 | 8, 13, 14, 15, 19 | command idempotency/audit와 요청 인증/credential isolation은 보안 목적은 같지만 검증 대상이 달라 별도 문제로 유지했다. |
| G10 | P25 | 20 | 10, 11, 15, 17, 21, 22 | 여러 worker Thread에 반복된 starvation·shutdown 우려를 `SettlementWorkerConfiguration` 하나의 대표 runtime 문제로 통합했다. |

## S/A 완전성 검증

아래 25개 S/A 항목은 모두 독립 상세 워크북 항목으로 작성되었다. 여러 Thread의 동일 역량을 흡수한 항목은 통합 범위를 함께 명시했다.

| ID | 우선순위 | 상세 위치 | 상태 | 명시적으로 통합한 범위 |
| --- | --- | --- | --- | --- |
| P01 | A | [01-contracts-and-state.md#p01](01-contracts-and-state.md#p01) | 독립 상세 항목 | Thread 5·9·10의 terminal 사용처를 연관 위치로 연결 |
| P02 | S | [01-contracts-and-state.md#p02](01-contracts-and-state.md#p02) | 독립 상세 항목 | Thread 13·16의 fingerprint 멱등성과 구분 |
| P03 | A | [01-contracts-and-state.md#p03](01-contracts-and-state.md#p03) | 독립 상세 항목 | Thread 3 raw boundary, Thread 12 listener orchestration 일부 통합 |
| P04 | A | [01-contracts-and-state.md#p04](01-contracts-and-state.md#p04) | 독립 상세 항목 | retry classifier와 raw DLT producer 통합 |
| P05 | A | [01-contracts-and-state.md#p05](01-contracts-and-state.md#p05) | 독립 상세 항목 | Thread 4의 lifecycle/result catch-up과 Thread 12 흐름 통합 |
| P06 | A | [01-contracts-and-state.md#p06](01-contracts-and-state.md#p06) | 독립 상세 항목 | lifecycle observation·tombstone·whole-slip void claim 통합 |
| P07 | S | [02-algorithms-and-money.md#p07](02-algorithms-and-money.md#p07) | 독립 상세 항목 | SYSTEM line expansion의 대표 알고리즘 |
| P08 | S | [02-algorithms-and-money.md#p08](02-algorithms-and-money.md#p08) | 독립 상세 항목 | payout 계산·classification·rounding test 통합 |
| P09 | S | [02-algorithms-and-money.md#p09](02-algorithms-and-money.md#p09) | 독립 상세 항목 | Thread 6 line counts와 Thread 10 durable execution 연결 |
| P10 | S | [02-algorithms-and-money.md#p10](02-algorithms-and-money.md#p10) | 독립 상세 항목 | Wallet failure·timeout·malformed success·status/proof coupling 통합 |
| P11 | A | [03-base-settlement-and-messaging.md#p11](03-base-settlement-and-messaging.md#p11) | 독립 상세 항목 | accepted projection·fanout·concurrent attempt test 통합 |
| P12 | S | [03-base-settlement-and-messaging.md#p12](03-base-settlement-and-messaging.md#p12) | 독립 상세 항목 | attempt draft·migration·atomic first claim 통합 |
| P13 | S | [03-base-settlement-and-messaging.md#p13](03-base-settlement-and-messaging.md#p13) | 독립 상세 항목 | Thread 10 recovery와 Thread 19 DB-time ownership 통합 |
| P14 | S | [03-base-settlement-and-messaging.md#p14](03-base-settlement-and-messaging.md#p14) | 독립 상세 항목 | Thread 10 finalizer, Thread 11 outbox, Thread 19 finalization fence 통합 |
| P15 | A | [03-base-settlement-and-messaging.md#p15](03-base-settlement-and-messaging.md#p15) | 독립 상세 항목 | runner failure isolation과 owner-fenced release 통합 |
| P16 | A | [04-result-candidates-and-revisions.md#p16](04-result-candidates-and-revisions.md#p16) | 독립 상세 항목 | candidate snapshot·fingerprint·decision taxonomy 통합 |
| P17 | S | [04-result-candidates-and-revisions.md#p17](04-result-candidates-and-revisions.md#p17) | 독립 상세 항목 | Thread 13 causal CAS, Thread 16 admin approval, Thread 19 DB-time fence 통합 |
| P18 | S | [04-result-candidates-and-revisions.md#p18](04-result-candidates-and-revisions.md#p18) | 독립 상세 항목 | target·snapshot·plan allocation·DB-time persistence 통합 |
| P19 | A | [04-result-candidates-and-revisions.md#p19](04-result-candidates-and-revisions.md#p19) | 독립 상세 항목 | application state graph와 V9 DB constraints 통합 |
| P20 | S | [05-correction-proof-and-recovery.md#p20](05-correction-proof-and-recovery.md#p20) | 독립 상세 항목 | Thread 8 Wallet boundary와 Thread 15 revision proof 통합 |
| P21 | S | [05-correction-proof-and-recovery.md#p21](05-correction-proof-and-recovery.md#p21) | 독립 상세 항목 | GET-first, blocked proof, recovery scanner, attempt ceiling 통합 |
| P22 | S | [05-correction-proof-and-recovery.md#p22](05-correction-proof-and-recovery.md#p22) | 독립 상세 항목 | Thread 15 execution, Thread 17 finalizer, Thread 19 DB-time fence, Thread 11 outbox 통합 |
| P23 | S | [06-admin-security-and-runtime.md#p23](06-admin-security-and-runtime.md#p23) | 독립 상세 항목 | V10 append-only schema, exact replay, candidate/retry command 통합 |
| P24 | A | [06-admin-security-and-runtime.md#p24](06-admin-security-and-runtime.md#p24) | 독립 상세 항목 | admin inbound auth, Wallet outbound headers, credential reuse policy 통합 |
| P25 | A | [06-admin-security-and-runtime.md#p25](06-admin-security-and-runtime.md#p25) | 독립 상세 항목 | base·outbox·lifecycle·revision·correction scheduler 격리와 종료 정책 통합 |

## 백지 구현 우선순위

1. P09 — 금액 보존식과 Wallet movement 계획
2. P07 — 결정적 K-of-N 조합 생성
3. P13 — DB-time lease claim과 stale-owner fence
4. P02 — placement canonical validation과 semantic replay
5. P17 — predecessor CAS와 causal sequence fence
6. P20 — adjustment proof identity·금액 대수 검증
7. P21 — GET-first recovery state reducer
8. P14 — 원자적 finalization과 transactional outbox
9. P23 — 운영 명령 semantic idempotency와 append-only audit
10. P08 — line payout 합산과 최종 한 번 rounding
11. P12 — 외부 호출 전 immutable attempt claim
12. P22 — stale target를 막는 revision finalization
13. P10 — Wallet HTTP 응답 분류와 exact success 판정
14. P18 — immutable revision plan과 payout delta
15. P04 — exact raw DLT publication
16. P11 — accepted result projection과 one-attempt planning
17. P06 — concurrent first-terminal latch
18. P25 — 격리 scheduler와 bounded shutdown
19. P05 — lifecycle-first out-of-order catch-up
20. P03 — raw Kafka key·decode·ack 호출 순서
21. P24 — 경로 한정 내부 인증
22. P16 — result candidate fingerprint와 intake policy
23. P15 — batch 실패 격리와 exact lease release
24. P01 — aggregate terminal transition
25. P19 — revision state graph와 row invariant validator

## 설명 연습 우선순위

1. P14 — outbox가 보장하는 것과 보장하지 않는 것
2. P21 — timeout 후 blind retry를 금지하는 이유
3. P13 — lease, DB time, `SKIP LOCKED`, stale owner의 관계
4. P22 — Wallet proof가 있어도 target 재검증이 필요한 이유
5. P17 — predecessor ID와 sequence가 각각 막는 인과 오류
6. P12 — persist-before-side-effect가 crash ambiguity를 줄이는 방식
7. P23 — idempotency key를 semantic request에 bind하는 이유
8. P10 — transport status와 business proof의 차이
9. P09 — committed stake와 payout을 두 보존식으로 나눈 이유
10. P05 — 서로 다른 Kafka topic 사이에 전역 순서가 없는 영향
11. P18 — correction plan을 immutable snapshot으로 고정한 이유
12. P19 — `BLOCKED`와 `EXHAUSTED`의 차이와 DB constraint의 역할
13. P24 — 401/403, duplicate header, credential separation
14. P25 — scheduler isolation과 DB lease 기반 수평 확장의 역할 분담
15. P04 — DLT publish 성공 전 source offset을 커밋하면 안 되는 이유
16. P02 — exact replay와 conflicting idempotency-key reuse의 차이
17. P11 — 메모리의 all-resolved 검사만으로 concurrent claim을 막을 수 없는 이유
18. P06 — observation history와 first-terminal tombstone의 차이
19. P01 — 결과 `VOID`와 aggregate `VOIDED`의 차이
20. P20 — exact adjustment proof가 금전 확정 권한인 이유
21. P08 — line마다 반올림하지 않는 이유
22. P07 — 결정적 조합 순서와 output-sensitive complexity
23. P15 — item 단위 실패 격리와 owner-fenced release
24. P16 — immutable result candidate와 mode-dependent fingerprint
25. P03 — key identity와 payload identity를 모두 검증하는 이유

## 한 문제로 통합한 Thread 묶음

1. P03: Thread 2의 placement listener + Thread 3의 raw Kafka 경계 + Thread 12의 listener orchestration
2. P04: Thread 3의 retry policy·failure classifier·raw DLT producer·recoverer
3. P05: Thread 4의 placement catch-up + Thread 5 lifecycle tombstone + Thread 9 accepted result fanout
4. P06: Thread 5 observation/latch + Thread 6·7의 SYSTEM total exposure + Thread 10 durable void claim
5. P09: Thread 6 line/surviving counts + Thread 7 money conservation/movement order + Thread 10 immutable attempt
6. P13: Thread 10 base attempt recovery + Thread 19 database-time ownership fencing + Thread 20 multi-worker 실행 환경
7. P14: Thread 10 base finalization + Thread 11 transactional outbox + Thread 19 DB-time lease consumption
8. P15: Thread 10 failure release + Thread 8 Wallet failure taxonomy + Thread 20 recovery worker 실행
9. P17: Thread 13 causal candidate replacement + Thread 16 admin candidate approval + Thread 19 database-time future fence
10. P18: Thread 14 target/snapshot/plan persistence + Thread 19 DB-time source validation
11. P20: Thread 15 revision proof validator + Thread 8 malformed Wallet success policy
12. P21: Thread 15 GET-first recovery·BLOCKED proof·attempt ceiling + Thread 16 operator retry queue + Thread 20 recovery scheduler
13. P22: Thread 15 revision execution + Thread 17 correction finalization + Thread 19 DB-time fence + Thread 11 revised outbox
14. P23: Thread 16 candidate approval·rejection·revision retry를 하나의 semantic idempotency/audit 문제로 통합
15. P24: Thread 18 admin inbound authentication + Wallet outbound authentication + credential reuse rejection
16. P25: Thread 20의 OUTBOX·LIFECYCLE·RECOVERY·REVISION_RECOVERY·CORRECTION scheduler와 Thread 22의 runtime 종료 표면
17. B01 Thread 12 전체 기본 정산 pipeline은 P03, P05, P08, P09, P11~P15로 분해·통합
18. B02 Thread 17 전체 correction pipeline은 P16~P22로 분해·통합
