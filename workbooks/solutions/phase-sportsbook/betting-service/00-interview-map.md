# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 범위와 근거 규칙

이 인덱스는 현재 GPT 프로젝트에서 식별 가능한 다음 기록만 사용한다.

- 스포츠북 애플리케이션 Development Thread 01~17의 Markdown 작업 기록
- External-State Evidence Packet으로 제목과 gap 범위가 확인된 Proposed Thread 18~20
- 내용까지 검색 가능한 운영·릴리스 Thread OPS-01, OPS-02, OPS-04
- 프로젝트 대화에서 파일명과 제목만 확인되고 본문은 확인되지 않은 SEED-15, SEED-16, SEED-17

확인되지 않은 중간 OPS/SEED 번호는 존재를 가정하지 않았다. Thread 18~20과 SEED-15~17은 코드·커밋 본문이 현재 컨텍스트에 노출되지 않았으므로, 제목과 Evidence Packet에서 확인되는 범위 이상으로 파일명·함수명·구현을 보충하지 않았다.

우선순위의 의미는 다음과 같다.

- **S**: 질문과 10~30분 축소 구현 모두 준비 가치가 매우 높다.
- **A**: 질문 가치가 높고, 핵심 일부는 직접 구현할 수 있다.
- **B**: 별도 백지 구현보다 설계·운영 원리 설명이 적절하다.
- **C**: 독립 면접 항목으로 만들기보다 다른 항목의 배경으로만 이해하면 충분하다.

## 상세 워크북 파일

| 파일 | 역할 | 포함 면접 포인트 |
|---|---|---|
| [01-domain-invariants-and-money.md](01-domain-invariants-and-money.md) | 도메인 구조, 조합·금액, 접수 검증, 키셋 조회 | Thread 01, 02, 03, 04 |
| [02-idempotency-and-remote-evidence.md](02-idempotency-and-remote-evidence.md) | 내구성 멱등성, 리스크·지갑 원격 증거 | Thread 05, 통합 Thread 06·07 |
| [03-saga-concurrency-and-locking.md](03-saga-concurrency-and-locking.md) | 사가 상태 머신, recovery lease, JPA 잠금 경계 | Thread 08, 11, 17 |
| [04-messaging-delivery-and-recovery.md](04-messaging-delivery-and-recovery.md) | outbox, inbox, Kafka 영구 오류·DLT | Thread 09, 10, 12 |
| [05-security-and-api-boundaries.md](05-security-and-api-boundaries.md) | default-deny gateway 경계와 trusted actor HTTP 계약 | 통합 Thread 13·14 |
| [06-settlement-ordering-and-consistency.md](06-settlement-ordering-and-consistency.md) | 기본 정산과 순서화된 revision | Thread 15, 16 |
| [07-release-integrity-and-publication.md](07-release-integrity-and-publication.md) | 고정 릴리스 입력과 generation 단위 원자적 게시 | OPS-01, OPS-02 |

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---|---|---|---|---|---|---|---|---|
| S | 01 | `feat(domain): attach structurally valid legs`<br>`feat(database): create bet aggregate schema` | `Bet`, `BetLeg`, `Bet.pending`, `BetLeg.assignTo`, `V1__bet_and_leg.sql` | 집계 루트, valid-by-construction, 자식 순서·소유권, 이중 invariant | 프레임워크를 넘어 모든 상태 중심 도메인에 일반화된다. 잘못된 부분 집계가 금액·정산 오류로 증폭되는 경계를 직접 구현할 수 있다. | 높음 | 높음 | 02, 03, 15, 16 | [D01](01-domain-invariants-and-money.md#t01-aggregate-invariants) |
| S | 02 | `feat(domain): calculate system exposure`<br>`feat(domain): calculate maximum system payout` | `SystemBetCalculator`, `lineCount`, `totalStake`, `maxPayout`, `binomial` | 조합론, overflow, 통화 의미론, 반올림, 복잡도 | 자료구조·알고리즘과 금액 invariant를 동시에 검증할 수 있다. unit stake와 total exposure 구분은 프로젝트 전반의 계약이다. | 높음 | 높음 | 01, 06, 07, 09, 15 | [D02](01-domain-invariants-and-money.md#t02-system-combinatorics) |
| S | 03 | `feat(odds): read effective market snapshots`<br>`feat(odds): enforce user-protective slippage` | `BetSlipValidator`, `OddsSnapshotReader`, `OddsSlippageChecker`, `BetAssembler` | 실패 폐쇄형 I/O, 십진 비교, 검증 순서, 경계값 | 돈이 걸린 승인 경로에서 근거 부재를 성공으로 추정하지 않는 판단과 정확한 slippage 경계를 묻기 좋다. | 높음 | 높음 | 01, 02, 05, 06 | [D03](01-domain-invariants-and-money.md#t03-fail-closed-admission) |
| A | 04 | `feat(identifier): generate time-ordered bet ids`<br>`feat(api): query actor scoped bet history` | `UuidV7`, `CursorPage`, `BetQueryService`, actor-scoped `BetRepository` queries | UUIDv7, 인덱스 지역성, 키셋 페이지, tenant predicate | 식별자 설계와 조회 복잡도·보안을 한 번에 설명할 수 있다. 비트 단위 UUID 구현보다 키셋 구현의 실전 가치가 높다. | 높음 | 중간 | 05, 13, 14, 17 | [D04](01-domain-invariants-and-money.md#t04-time-keyset) |
| S | 05 | `feat(placement): replay durable outcomes safely` | `PlacementRequest`, `PlacementReplay`, `BetPlacementService.place`, `BetStore` | durable idempotency namespace, actor·fingerprint 결합, unique race, 정확한 결과 재생 | 분산 API 면접에서 반복되는 핵심 주제다. 성공뿐 아니라 확정 거부까지 한 namespace로 소유하고 race 패자가 winner를 다시 읽는 설계가 강하다. | 높음 | 높음 | 07, 08, 09, 14 | [D05](02-idempotency-and-remote-evidence.md#t05-durable-idempotency) |
| A | 06 | `feat(risk): require explicit reservation approval`<br>`feat(risk): bound dependency failures with circuit breaker` | `RiskClient`, reservation wire models, risk token migration | 명시적 원격 verdict, opaque proof, strict status, fallback 분류 | Thread 07과 같은 "원격 성공을 증거로 채택" 역량을 반복한다. 리스크 고유 lifecycle 질문은 가치가 있으나 구현 문제는 Thread 07과 통합하는 편이 낫다. | 높음 | 중간 | 05, 07, 08 | [D06에 통합](02-idempotency-and-remote-evidence.md#t06-t07-remote-evidence) |
| S | 07 | `feat(wallet): debit full exposure idempotently`<br>`feat(wallet): refund locked exposure idempotently` | `WalletClient`, operation proof, debit lookup, refund path | 원격 operation proof, idempotency conflict lookup, actor·금액·reason 검증 | 응답 유실·재시도·충돌 상황에서 "성공 추정"을 피하는 구현 문제로 매우 좋다. Thread 06의 공통 역량을 대표한다. | 높음 | 높음 | 05, 06, 08, 10 | [D06](02-idempotency-and-remote-evidence.md#t06-t07-remote-evidence) |
| S | 08 | `feat(placement): checkpoint external side effects`<br>`feat(placement): commit risk before atomic acceptance` | `Bet`, `BetPlacementService`, `BetStore`, `V6__placement_compensation_and_verdict.sql` | durable saga, 단조 상태 전이, 보상 action, 재시작·중복 실행 | 프로젝트의 핵심 orchestration이다. 상태 머신, invariant, 실패 복구, local atomicity를 모두 직접 검증할 수 있다. | 높음 | 높음 | 05, 06, 07, 09, 10, 11 | [D07](03-saga-concurrency-and-locking.md#t08-saga-checkpoints) |
| S | 09 | `feat(database): add transactional outbox`<br>`feat(outbox): publish acknowledged pending events` | `OutboxEvent`, `OutboxPublisher`, `BetStore.acceptAndEnqueue`, `KafkaConfig`, `V2__outbox.sql` | transactional outbox, broker ack, at-least-once, interruption·timeout | DB와 broker 사이의 원자성 간극을 다루는 대표 시스템 설계 문제이며, 축소 publisher 구현도 가능하다. | 높음 | 높음 | 05, 08, 10, 12 | [D10](04-messaging-delivery-and-recovery.md#t09-transactional-outbox) |
| S | 10 | `feat(wallet-events): deduplicate reconciliation hints`<br>`feat(wallet-events): preserve raw record identity` | `WalletEventReceipt`, `WalletEventInbox`, `WalletEventListener`, `Bet.requestReconciliation` | durable inbox, 동일 재생과 동일 ID 충돌, receipt-first, wake hint | outbox의 소비 측 대칭이면서 별도의 중요한 판단을 포함한다. payload hash와 actor ownership으로 충돌을 격리하는 구현 가치가 높다. | 높음 | 높음 | 07, 08, 09, 11, 12 | [D11](04-messaging-delivery-and-recovery.md#t10-durable-inbox) |
| S | 11 | `feat(recovery): claim fair reconciliation batches`<br>`feat(recovery): consume owner-fenced reconciliation claims` | `BetRepository.claimReconciliationBatch`, `clearReconciliationClaim`, `BetReconciliationJob`, `V10__reconciliation_lease.sql` | `SKIP LOCKED`, DB 시간, lease expiry, owner fencing, 공정성 | 동시 worker, crash recovery, bounded resource 사용을 한 문제에서 다룬다. SQL과 순수 상태 구현 모두 면접 가치가 높다. | 높음 | 높음 | 08, 10, 17 | [D08](03-saga-concurrency-and-locking.md#t11-owner-fenced-leases) |
| S | 12 | `feat(messaging): require acknowledged permanent recovery`<br>`feat(messaging): require preprovisioned topics` | `KafkaMessageValidator`, `PermanentKafkaException`, `KafkaRecoveryConfig`, Kafka recovery integration test | transient·permanent 분류, raw DLT, DLT ack와 offset 안전성 | retry·DLT를 설정 암기가 아니라 데이터 손실 경계로 설명하게 한다. 실패 분류 coordinator로 축소 구현하기 좋다. | 높음 | 높음 | 09, 10, 15, 16 | [D12](04-messaging-delivery-and-recovery.md#t12-kafka-recovery) |
| S | 13 | `feat(api): authenticate the gateway boundary`<br>`feat(api): enforce exact gateway ingress` | `GatewayAuthFilter`, `GatewayAuthProperties` | default deny, exact-one header, route allowlist, constant-time secret, credential 분리 | 보안 경계의 판단이 구체적이고 일반화 가능하다. Thread 14의 actor mapping을 함께 묻는 것이 더 완전하다. | 높음 | 높음 | 04, 14, 19 | [D13](05-security-and-api-boundaries.md#t13-t14-trust-boundary) |
| A | 14 | `feat(api): map trusted placement commands`<br>`feat(api): expose trusted placement and history routes` | `PlaceBetRequest`, `BetResponse`, `BetController` | caller와 actor 분리, body spoofing 방지, 안정적 status·Location 계약 | DTO 암기 자체는 낮은 가치지만, 신뢰된 actor를 body와 분리하는 판단은 중요하다. Thread 13의 경계 문제에 통합했다. | 높음 | 중간 | 04, 05, 13 | [D13에 통합](05-security-and-api-boundaries.md#t13-t14-trust-boundary) |
| S | 15 | `feat(settlement): project base resolution snapshots`<br>`feat(settlement): preserve raw resolution keys` | `BetSettlementService`, `SettlementResultListener`, `Bet.settleBase`, `Bet.voidBase` | base snapshot, duplicate·conflict·superseded, whole-slip refund 검증 | 금액 확정 입력의 identity와 invariant를 다룬다. SYSTEM unit stake와 total exposure를 정산에 연결하는 좋은 교차 Thread 문제다. | 높음 | 높음 | 02, 12, 16, 17 | [D14](06-settlement-ordering-and-consistency.md#t15-base-settlement) |
| S | 16 | `feat(settlement): apply full revision snapshots`<br>`feat(settlement): consume ordered resolution revisions`<br>`fix(settlement): validate revision chronology` | `Bet.applyRevision`, `BetSettlementService`, `SettlementResultListener`, `V9__resolution_revision_projection.sql` | 순서화 revision, 낮은 값·중복·갭·충돌, full snapshot, chronology | 상태 머신과 이벤트 순서 문제 중 가장 밀도가 높다. gap 가용성과 일관성 검증 trade-off를 직접 구현할 수 있다. | 높음 | 높음 | 10, 12, 15, 17 | [D15](06-settlement-ordering-and-consistency.md#t16-ordered-revisions) |
| A | 17 | `fix(persistence): lock bet roots before loading legs` | `BetRepository.findLockedByBetId`, `findLockedRootByBetId`, `findWithLegsByBetId` | pessimistic root lock, graph loading, lock amplification, transaction lifecycle | JPA 사용법 암기보다 DB 잠금 경계를 이해하는지 확인할 수 있다. 구현은 작지만 설명 가치가 높다. | 높음 | 중간 | 08, 11, 15, 16 | [D09](03-saga-concurrency-and-locking.md#t17-root-locking) |
| B | 18 | 커밋 제목 미노출 | Evidence Packet NT-18, `GAP-ES-05` | DB provisioning, migration authority 분리, immutable migration, restore readiness | 운영·복구 설계 질문은 가치가 높지만 현재 컨텍스트에서 실제 코드·커밋 경계를 확인할 수 없어 독립 구현 문제로 만들지 않는다. | 중간 | 낮음 | OPS-04 | — |
| B | 19 | 커밋 제목 미노출 | Evidence Packet NT-19, `GAP-ES-06` | 방향별 credential namespace, pairwise distinctness, runtime binding | Thread 13의 보안 경계를 확장하는 중요한 개념이나, 현재 확인 가능한 정보는 Evidence Packet의 설계 범위뿐이다. | 중간 | 낮음 | 13, 14 | — |
| B | 20 | 커밋 제목 미노출 | Evidence Packet NT-20, `GAP-ES-07` | pinned protocol source, local·CI Maven bootstrap, 재현 검증 | OPS-01·02와 강하게 연관되지만 독립 구현 식별자가 확인되지 않는다. 릴리스 설명 연습의 연관 항목으로만 남긴다. | 중간 | 낮음 | OPS-01, OPS-02 | — |
| A | OPS-01 | `build(lock): pin service release inputs`<br>`build(source): materialize detached release worktrees` | `services.lock`, `scripts/materialize-sources.sh`, materializer tests | exact source identity, manifest preflight, detached worktree, 안전한 rollback | 움직이는 branch를 릴리스 입력으로 쓰는 위험과 filesystem cleanup 보안을 함께 묻는 일반화 가능한 문제다. | 높음 | 중간 | OPS-02, NT-20 | [D16](07-release-integrity-and-publication.md#ops01-pinned-materialization) |
| A | OPS-02 | `build(shared): install the locked protocol artifact`<br>`build(jars): stage exact release artifacts atomically` | `scripts/install-shared.sh`, `scripts/stage-release-jars.sh`, generation tests | hermetic build, complete release set, checksum, atomic symlink swap | build·게시 lifecycle, 부분 실패, resource cleanup을 확인할 수 있다. 순수 filesystem 문제로 축소 구현 가능하다. | 높음 | 중간 | OPS-01, OPS-04, NT-20 | [D17](07-release-integrity-and-publication.md#ops02-atomic-publication) |
| B | OPS-04 | `build(gate): query owned release databases` | `04-storage-isolation-and-migration-integrity-01.md`, `PostgresClient`, `MigrationEvidence`, `flyway_checksum`, Redis isolation tests | 저장소 격리, DB inventory allowlist, migration checksum·history evidence | 운영 설명 가치는 높지만 현재 선별된 S/A의 DB·릴리스 문제와 겹치며, 별도 10~30분 구현 문제로는 프로젝트 특수성이 크다. | 중간 | 낮음 | 18, OPS-02 | — |
| B | SEED-15 | 커밋 메시지 현재 컨텍스트에서 미확인 | `15-evidence-storage-and-secret-redaction.md` | 증거 저장과 secret redaction | 제목으로 보안·감사 주제는 확인되지만 본문·식별자가 확인되지 않아 상세 문제를 만들 수 없다. | 중간 | 낮음 | 13, 19 | — |
| B | SEED-16 | 커밋 메시지 현재 컨텍스트에서 미확인 | `16-semantic-release-attestation-01.md`~`03.md` | semantic release attestation | 공급망 설명 가치가 예상되지만 현재 확인 가능한 것은 문서명뿐이다. OPS-01·02의 설명 배경으로만 둔다. | 중간 | 낮음 | OPS-01, OPS-02, 20 | — |
| C | SEED-17 | 커밋 메시지 현재 컨텍스트에서 미확인 | `17-linear-development-history-policy-01.md`, `02.md` | 선형 개발 이력 정책 | 독립 구현보다 팀 정책·프로세스 설명에 가깝고, 현재 본문이 확인되지 않아 별도 면접 준비 항목으로 만들지 않는다. | 낮음 | 낮음 | SEED-16 | — |

## 대표 면접 포인트와 연관 Thread 관계

| 상세 ID | 대표 Thread | 통합·연관 Thread | 관계 | 상세 위치 |
|---|---|---|---|---|
| D01 | 01 | 02, 03, 15, 16 | 집계 구조 invariant가 계산·접수·정산의 선행조건이다. | [집계 루트 invariant](01-domain-invariants-and-money.md#t01-aggregate-invariants) |
| D02 | 02 | 01, 06, 07, 09, 15 | unit stake와 total exposure 의미가 원격 요청·이벤트·void 검증까지 이어진다. | [시스템 조합·금액](01-domain-invariants-and-money.md#t02-system-combinatorics) |
| D03 | 03 | 01, 02, 05, 06 | 도메인 구조를 확인한 뒤 외부 snapshot 근거를 실패 폐쇄형으로 읽는다. | [접수 검증](01-domain-invariants-and-money.md#t03-fail-closed-admission) |
| D04 | 04 | 05, 13, 14, 17 | 시간 순서 ID를 actor-scoped keyset 조회 경계에 사용한다. | [UUIDv7·키셋](01-domain-invariants-and-money.md#t04-time-keyset) |
| D05 | 05 | 07, 08, 09, 14 | 요청 결과 소유권이 사가 시작과 외부 부작용 재실행을 통제한다. | [내구성 멱등성](02-idempotency-and-remote-evidence.md#t05-durable-idempotency) |
| D06 | 07 | **06 통합**, 05, 08, 10 | 리스크와 지갑 모두 원격 성공을 명시적 proof로 검증한다. Thread 07을 대표 구현으로 사용한다. | [원격 operation 증거](02-idempotency-and-remote-evidence.md#t06-t07-remote-evidence) |
| D07 | 08 | 05, 06, 07, 09, 10, 11 | 멱등성·외부 proof·outbox·recovery job을 durable 사가가 결합한다. | [사가 체크포인트](03-saga-concurrency-and-locking.md#t08-saga-checkpoints) |
| D08 | 11 | 08, 10, 17 | wake hint와 stale 상태를 여러 worker가 owner-fenced lease로 나눈다. | [복구 lease](03-saga-concurrency-and-locking.md#t11-owner-fenced-leases) |
| D09 | 17 | 08, 11, 15, 16 | 모든 쓰기 전이는 동일한 root lock 경계를 통과한다. | [루트 잠금](03-saga-concurrency-and-locking.md#t17-root-locking) |
| D10 | 09 | 05, 08, 10, 12 | local acceptance와 event 생성은 원자적이고 broker 전달은 at-least-once다. | [transactional outbox](04-messaging-delivery-and-recovery.md#t09-transactional-outbox) |
| D11 | 10 | 07, 08, 09, 11, 12 | 소비 측 중복·충돌을 영속화하고 authoritative reconciliation을 깨운다. | [durable inbox](04-messaging-delivery-and-recovery.md#t10-durable-inbox) |
| D12 | 12 | 09, 10, 15, 16 | consumer record validation 실패를 permanent로 격리하고 DLT ack를 offset 증거로 사용한다. | [Kafka 복구](04-messaging-delivery-and-recovery.md#t12-kafka-recovery) |
| D13 | 13 | **14 통합**, 04, 05, 19 | caller 인증·route 인가와 trusted actor command mapping을 한 경계 문제로 묶는다. | [보안·API 경계](05-security-and-api-boundaries.md#t13-t14-trust-boundary) |
| D14 | 15 | 02, 12, 16, 17 | 첫 정산 snapshot의 identity와 SYSTEM committed exposure를 검증한다. | [기본 정산](06-settlement-ordering-and-consistency.md#t15-base-settlement) |
| D15 | 16 | 10, 12, 15, 17 | revision 순서·갭·충돌과 permanent Kafka 격리를 한 projection 규칙으로 다룬다. | [순서화 revision](06-settlement-ordering-and-consistency.md#t16-ordered-revisions) |
| D16 | OPS-01 | OPS-02, NT-20 | 정확한 source identity와 protocol bootstrap의 입력 고정 단계다. | [릴리스 입력 고정](07-release-integrity-and-publication.md#ops01-pinned-materialization) |
| D17 | OPS-02 | OPS-01, OPS-04, NT-20 | 고정 입력을 격리 빌드하고 complete generation으로 게시하는 단계다. | [원자적 게시](07-release-integrity-and-publication.md#ops02-atomic-publication) |

## S/A 완전성 검증

| 마스터 S/A 항목 | 상태 | 상세 워크북 위치 |
|---|---|---|
| Thread 01 | 독립 항목 작성됨 | [D01](01-domain-invariants-and-money.md#t01-aggregate-invariants) |
| Thread 02 | 독립 항목 작성됨 | [D02](01-domain-invariants-and-money.md#t02-system-combinatorics) |
| Thread 03 | 독립 항목 작성됨 | [D03](01-domain-invariants-and-money.md#t03-fail-closed-admission) |
| Thread 04 | 독립 항목 작성됨 | [D04](01-domain-invariants-and-money.md#t04-time-keyset) |
| Thread 05 | 독립 항목 작성됨 | [D05](02-idempotency-and-remote-evidence.md#t05-durable-idempotency) |
| Thread 06 | Thread 07 대표 문제에 명시적으로 통합됨 | [D06](02-idempotency-and-remote-evidence.md#t06-t07-remote-evidence) |
| Thread 07 | 독립 대표 항목 작성됨 | [D06](02-idempotency-and-remote-evidence.md#t06-t07-remote-evidence) |
| Thread 08 | 독립 항목 작성됨 | [D07](03-saga-concurrency-and-locking.md#t08-saga-checkpoints) |
| Thread 09 | 독립 항목 작성됨 | [D10](04-messaging-delivery-and-recovery.md#t09-transactional-outbox) |
| Thread 10 | 독립 항목 작성됨 | [D11](04-messaging-delivery-and-recovery.md#t10-durable-inbox) |
| Thread 11 | 독립 항목 작성됨 | [D08](03-saga-concurrency-and-locking.md#t11-owner-fenced-leases) |
| Thread 12 | 독립 항목 작성됨 | [D12](04-messaging-delivery-and-recovery.md#t12-kafka-recovery) |
| Thread 13 | 독립 대표 항목 작성됨 | [D13](05-security-and-api-boundaries.md#t13-t14-trust-boundary) |
| Thread 14 | Thread 13 대표 문제에 명시적으로 통합됨 | [D13](05-security-and-api-boundaries.md#t13-t14-trust-boundary) |
| Thread 15 | 독립 항목 작성됨 | [D14](06-settlement-ordering-and-consistency.md#t15-base-settlement) |
| Thread 16 | 독립 항목 작성됨 | [D15](06-settlement-ordering-and-consistency.md#t16-ordered-revisions) |
| Thread 17 | 독립 항목 작성됨 | [D09](03-saga-concurrency-and-locking.md#t17-root-locking) |
| OPS-01 | 독립 항목 작성됨 | [D16](07-release-integrity-and-publication.md#ops01-pinned-materialization) |
| OPS-02 | 독립 항목 작성됨 | [D17](07-release-integrity-and-publication.md#ops02-atomic-publication) |

## 백지 구현 우선순위

1. Thread 08 — durable 사가 checkpoint와 compensation 상태 머신
2. Thread 05 — durable idempotency namespace와 unique race 후 정확한 재생
3. Thread 16 — ordered full-snapshot revision 적용 규칙
4. Thread 11 — owner-fenced bounded lease claim
5. Thread 02 — K-of-N 조합 수, total exposure, 최대 payout
6. Thread 09 — broker ack 기반 outbox publisher
7. Thread 06·07 — 원격 operation proof와 idempotency conflict lookup
8. Thread 13·14 — default-deny ingress와 trusted actor command mapping
9. Thread 12 — permanent/transient 분류와 DLT ack coordinator
10. Thread 10 — duplicate와 conflict를 구분하는 durable inbox
11. Thread 03 — 실패 폐쇄형 market snapshot·slippage 검증
12. Thread 01 — valid-by-construction 집계 조립
13. Thread 15 — base settlement duplicate·conflict·whole-slip void 검증
14. Thread 04 — actor-scoped keyset page
15. Thread 17 — root lock과 graph loading 분리
16. OPS-01 — manifest preflight와 detached materialization rollback
17. OPS-02 — complete generation과 atomic publication

## 설명 연습 우선순위

1. Thread 08 — 왜 긴 transaction이 아니라 durable saga이며, 각 보상 branch가 무엇인지
2. Thread 05 — 멱등성이 중복 성공 방지가 아니라 요청 결과 소유권인 이유
3. Thread 09·10·12 — outbox, inbox, DLT가 각각 어떤 유실·중복 경계를 담당하는지
4. Thread 16 — 낮은 revision, duplicate, conflict, gap을 다르게 처리하는 이유
5. Thread 06·07 — HTTP 성공과 검증된 원격 operation evidence의 차이
6. Thread 13·14 — caller credential과 end-user actor를 분리하는 이유
7. Thread 11 — `SKIP LOCKED`, DB 시간, owner fencing, lease expiry의 결합
8. Thread 02·15 — unit stake, total exposure, payout, whole-slip refund의 금액 의미론
9. Thread 17 — root row 잠금과 collection fetch를 분리한 DB·JPA 이유
10. Thread 03 — 돈이 걸린 승인 경로에서 fail closed를 선택한 이유
11. OPS-01·02 — exact source identity에서 complete release generation까지의 재현성·원자성
12. Thread 04 — UUIDv7과 keyset 조회의 이점과 보장하지 않는 것

## 한 문제로 통합한 Thread 묶음

- **Thread 06 + Thread 07** → "원격 성공을 검증 가능한 operation evidence로 채택하고 idempotency conflict를 authoritative lookup으로 복구하기". Thread 07을 대표 구현으로 사용하고 리스크 reservation·commit·release의 차이는 꼬리 질문으로 유지했다.
- **Thread 13 + Thread 14** → "default-deny gateway 경계에서 caller를 인증하고, 신뢰된 actor만 안정적인 HTTP command로 매핑하기". 필터 보안과 DTO 매핑을 분리 문제로 반복하지 않았다.
