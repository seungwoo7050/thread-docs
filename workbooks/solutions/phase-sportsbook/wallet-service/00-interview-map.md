# Sportsbook Wallet 개발자 기술면접 워크북 — 마스터 인덱스

## 문서 목적과 증거 범위

이 인덱스는 현재 GPT 프로젝트에 축적된 Thread 1–17 학습 문서와 과거 DevThread 작업 기록에서 실제로 확인되는 정보만으로 면접 지점을 선별한 결과다. 원격 저장소, 운영 환경, 배포 이력은 조사하지 않았다. 실제 기록에서 확인되지 않는 파일명·함수명·외부 운영 절차는 적지 않았다.

상세 워크북은 **S와 A 항목만** 다룬다. 여러 Thread가 같은 역량을 반복하는 경우 대표 문제 하나로 통합했고, 나머지는 이 문서의 `상세 워크북 완전성 대조표`에서 통합 상태를 명시했다.

## 우선순위 기준

| 우선순위 | 기준 |
| --- | --- |
| S | 반드시 준비한다. 질문 가치와 10–30분 백지 구현 가치가 모두 높다. |
| A | 준비 가치가 높다. 핵심 구현 또는 설계 설명으로 이어질 가능성이 높다. |
| B | 별도 구현 문제보다 설계 의도·운영 경계·검증 전략 설명이 중요하다. |
| C | 반복 배선, 단순 선언, 빌드 부트스트랩처럼 독립 면접 항목으로 만들 가치가 낮다. |

## 전체 Thread/커밋 선별 결과

| ID | 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| P01 | B | 1 | `docs(project): introduce wallet ownership` | `README.md`, `docs/architecture.md` | 권위 상태와 시스템 경계 | PostgreSQL이 정합성의 유일한 권위이고 Redis·Kafka는 결정을 소유하지 않는다는 전제가 이후 설계 전체를 설명한다. 독립 구현보다 아키텍처 설명에 적합하다. | 높음 | 낮음 | 2, 6, 12, 13, 14 |
| P02 | B | 1 | `feat(persistence): read transaction timestamps from PostgreSQL` | `DatabaseClock`, `DatabaseClock.now()` | DB 시계와 인과 순서 | 마감 시각은 DB 시계로 비교하지만 업무 인과 순서는 상태·sequence·version으로 표현한다. 분산 시스템 설명 가치가 높지만 구현 범위는 작다. | 높음 | 낮음 | 10, 11, 13 |
| P03 | B | 2 | `feat(operation): resolve fingerprints inside execution transactions` | `WalletOperationExecutor`, `WalletTransferExecutor`, `InternalApiKeyAuthenticationFilter`, `WalletSecurityConfig` | 금전 요청 처리 파이프라인의 경계 | 인증→권한→요청 식별→최초 작성자→계정 잠금→원장·결과·아웃박스라는 전체 흐름을 설명하는 대표 Thread다. 세부 구현은 Thread 6·7·15에 통합한다. | 높음 | 중간 | 6, 7, 15, 16, 17 |
| P04 | C | 3 | `build(maven): add the Maven wrapper` | `.mvn/wrapper/maven-wrapper.properties`, `mvnw` | 빌드 도구 부트스트랩 | 재현성에는 필요하지만 독립적인 개발 기본기나 프로젝트 핵심 판단을 검증하기에는 boilerplate 비중이 높다. | 낮음 | 낮음 | 없음 |
| P05 | B | 3 | `test(gate): provision live wallet dependencies` | `WalletSmokeFixture`, `@Tag("wallet-semantic-gate")` 적용 테스트, PostgreSQL·Redis·Kafka Testcontainers | 의미 검증 게이트와 실제 의존성 테스트 | 인메모리 대체물이 놓치는 잠금·SQL·브로커 의미를 검증한다. 직접 구현보다 테스트 경계와 비용을 설명하는 항목이다. | 높음 | 낮음 | 6, 7, 11, 13, 15 |
| P06 | S | 4 | `test(domain): verify Long.MAX aggregate boundaries` | `Account`, `EmbeddedMoney`, `AccountBalanceLimitTest`, `V1__account_and_ledger.sql` | 잔액 전이와 표현 범위 invariant | 비음수·단일 통화·두 버킷 합계 범위·실패 시 무변경을 동시에 다룬다. 언어의 정수 오버플로와 도메인 상태 전이를 직접 구현하기 좋다. | 높음 | 높음 | 1, 5, 7, 11 |
| P07 | S | 5 | `feat(domain): construct matched debit-credit pairs` | `LedgerEntry`, `LedgerEntry.pair()`, `LedgerEntry.TransferLeg`, `LedgerEntry.Pair` | 이중부기 쌍 구성 | 한 자금 이동을 양쪽 원장 행으로 강제하고 공유 식별자를 보존하는 핵심 모델이다. 작은 구현으로 정합성 판단을 검증할 수 있다. | 높음 | 높음 | 4, 7, 9, 14 |
| P08 | S | 5 | `feat(service): derive results from complete ledger pairs` | `WalletOperationResult.fromExisting()`, `WalletTransferTopologyTest` | 원장 쌍 복원과 업무 토폴로지 검증 | 행 수와 side만 맞아도 잘못된 상대 계정·버킷이 들어갈 수 있다. 이유별 토폴로지 검증은 일반화 가능한 방어적 재구성 문제다. | 높음 | 높음 | 7, 9, 14 |
| P09 | S | 6 | `feat(operation): reject conflicting request identities` | `CanonicalRequestEncoder`, `WalletRequestIdentity`, `OperationFingerprint`, `IdempotencyConflictException` | 정규 요청 지문과 멱등성 충돌 | 같은 키를 다른 의미의 요청에 재사용하는 문제를 막는다. 결정적·비모호 인코딩과 버전 관리까지 질문과 구현 가치가 높다. | 높음 | 높음 | 2, 10, 16, 17 |
| P10 | S | 6 | `feat(operation): resolve fingerprints inside execution transactions` | `WalletOperationExecutor.execute()`, `IdempotencyKeyLock.acquire()`, `WalletOperationRepository` | 영속 최초 작성자 직렬화와 결과 재생 | 빠른 재생, 트랜잭션 범위 advisory lock, 잠금 후 재확인, 실패 후 승자 재조회가 하나의 경쟁 알고리즘을 이룬다. | 높음 | 높음 | 1, 2, 7, 10 |
| P11 | A | 6 | `feat(cache): treat Redis idempotency data as a fallible hint` | `IdempotencyCache.mightContain()`, `IdempotencyCache.mark()` | 캐시와 권위 상태 분리 | 캐시 유실·장애가 이미 커밋된 결과를 바꾸지 못하게 만드는 경계가 중요하다. P10의 대표 문제에 통합한다. | 높음 | 중간 | 1, 10 |
| P12 | S | 7 | `feat(service): execute durable transfer outcomes` | `WalletTransferExecutor`, `WalletTransferWriter`, `WalletTransferPlan`, `AccountRepository.findByUserIdForUpdate()` | 계정 잠금과 단일 트랜잭션 이체 | 잔액·원장 쌍·업무 결과·아웃박스가 함께 커밋되거나 모두 롤백되어야 한다. 동시 exact-balance 경쟁까지 직접 구현 가치가 높다. | 높음 | 높음 | 4, 5, 6, 8, 12 |
| P13 | S | 7 | `test(operation): roll back outcomes with failed money transactions` | `WalletPersistenceTest`, `OperationCommitted`, `WalletTransferWriter` | 중간 실패와 전체 롤백 | 커밋 전 알림 실패를 주입해 잔액·원장·결과가 모두 사라지는지 검증한다. P12의 원자성 문제에 통합한다. | 높음 | 높음 | 5, 8, 12 |
| P14 | A | 8 | `feat(debit): commit debit success or durable rejection atomically` | `WalletTransferExecutor.executeDebit()`, `WalletEventFactory`, `WalletTransferReceipt` | 성공과 업무 거절의 영속 사실화 | 성공은 잔액·원장·결과·성공 이벤트를, 잔액 부족은 무변경·거절 결과·실패 이벤트를 같은 트랜잭션에 묶는다. 아웃박스 기록 문제에 통합한다. | 높음 | 중간 | 6, 7, 12, 17 |
| P15 | A | 9 | `feat(service): prepare semantic credit transfers` | `WalletTransferExecutor.requireAllowedCredit()`, `CreditCommand`, `CreditReason`, `WalletTransferTopologyTest` | 호출자·자금 출처·업무 이유의 의미 권한 | HTTP 경로 권한만으로는 잘못된 지급 출처를 막을 수 없다. 원장 토폴로지와 업무 권한을 함께 묻기 좋다. | 높음 | 중간 | 5, 15, 16 |
| P16 | C | 9 | `feat(command): classify credit reasons` | `CreditReason` | 업무 enum 선언 | 이후 권한·토폴로지의 입력 vocabulary로 필요하지만 enum 선언만 떼어 내면 독립 면접 가치가 낮다. | 낮음 | 낮음 | 5, 15 |
| P17 | S | 10 | `feat(adjustment): choose locked correction outcomes` | `AdjustmentFirstWriter.write()`, `AdjustmentProofWriter`, `WalletAdjustmentService.adjust()` | 조정 최초 판정 상태 머신 | 양수 즉시 반영, 음수 즉시 회수, FIFO 선행 작업·잔액 부족 시 BLOCKED, 업무 오류 시 REJECTED를 원자적으로 판정한다. | 높음 | 높음 | 6, 7, 11 |
| P18 | S | 10 | `feat(adjustment): serialize bet revision claims` | `AdjustmentPairLock.acquire()`, `WalletAdjustmentRepository.findByBetIdAndRevisionNumber()` | 복합 업무 식별자의 동시 선점 | 멱등성 키와 별개로 `(betId, revisionNumber)` 중복을 직렬화한다. P17 상태 머신의 경쟁 조건으로 통합한다. | 높음 | 높음 | 6, 11 |
| P19 | S | 11 | `feat(recovery): claim one FIFO head transactionally` | `RecoveryWorker.recoverOne()`, `RecoveryHeadProcessor.process()`, `WalletAdjustmentRepository.findOldestBlockedForUpdate()` | 복구 부채 FIFO 회수 | 계정별 oldest head만 처리하고 두 작업자가 같은 부채를 이중 회수하지 않게 한다. 부족 자금 재시도와 성공 전이를 함께 구현할 가치가 높다. | 높음 | 높음 | 4, 7, 10 |
| P20 | A | 11 | `feat(recovery): bound automatic retry delays` | `RecoveryRetryPolicy`, `WalletAdjustment.deferUntil()` | 포화형 지수 backoff와 무효과 재시도 | 지수 계산의 오버플로를 막고 cap을 보장하며, 자금 부족 시 retry metadata 외에는 바꾸지 않는 경계가 중요하다. P19에 통합한다. | 높음 | 중간 | 10, 13 |
| P21 | A | 12 | `feat(outbox): append one event per operation` | `OutboxAppender.append()`, `OutboxStreamLock.nextSequence()`, `OutboxEvent`, `OutboxEventRepository` | 트랜잭셔널 아웃박스 기록 | DB 커밋과 메시지 사실을 하나로 묶고 스트림별 단조 sequence를 배정한다. 직접 Kafka 전송과 대비되는 핵심 설계다. | 높음 | 높음 | 8, 9, 13 |
| P22 | S | 13 | `feat(repository): claim strict FIFO outbox stream heads` | `OutboxDeliveryRepository.claim()` | 다중 작업자 스트림별 FIFO 선점 | `NOT EXISTS` 선행 미발행 행, DB 시계, `FOR UPDATE SKIP LOCKED`, lease 갱신을 결합한다. 동시 큐 처리 면접 문제로 가치가 높다. | 높음 | 높음 | 12 |
| P23 | S | 13 | `feat(outbox): model fenced delivery leases` | `OutboxLease`, `LeasedOutboxMessage`, `OutboxDeliveryRepository.markPublished()`, `releaseForRetry()` | lease version fencing과 at-least-once | 만료 lease를 탈취한 뒤 늦게 도착한 이전 작업자의 완료를 막는다. 브로커 ACK와 DB 완료 사이 중복 가능성까지 설명·구현 가치가 높다. | 높음 | 높음 | 12, 17 |
| P24 | A | 13 | `feat(outbox): schedule indefinitely retried bounded backoff` | `OutboxRetryPolicy`, `OutboxPublisher` | 비동기 완료·bounded in-flight·무제한 재시도 | permit 누수, 오류 문자열 경계, 포화형 backoff, send 완료 생명주기를 다룬다. P23에 통합한다. | 높음 | 중간 | 11, 17 |
| P25 | C | 13 | `config(outbox): activate safe delivery polling` | `src/main/resources/application.yml` | 스케줄러 활성화 설정 | 실제 운영에는 필요하지만 설정 토글 자체는 별도 구현 문제로 만들 가치가 낮다. 안전 조건은 P23 설명에 포함한다. | 중간 | 낮음 | 3 |
| P26 | A | 14 | `feat(integrity): reconcile account ledger snapshots` | `AccountIntegrityRepository.findSnapshotDrift()`, `findOrphanLedgerAccountIds()` | 잔액 스냅샷과 append-only 원장 대사 | 파생 스냅샷을 권위 원장과 재계산해 drift를 찾는다. 부호·통화·오버플로·고아 행을 다루는 구현 문제로 적합하다. | 높음 | 높음 | 4, 5 |
| P27 | A | 14 | `feat(integrity): scan durable wallet invariants` | `WalletIntegrityScanner.scan()`, `WalletIntegritySnapshot` | 하나의 repeatable-read 뷰에서 다중 invariant 스캔 | 여러 drift query가 서로 다른 시점을 보면 거짓 양성이 생긴다. P26의 대표 문제에 통합한다. | 높음 | 중간 | 1, 6, 10, 11 |
| P28 | B | 14 | `feat(integrity): publish scan metrics` | `WalletIntegrityMetrics`, `WalletIntegrityHealth`, `WalletIntegrityScheduler` | bounded 관측 상태와 health | scrape 때 DB를 읽지 않고 마지막 완료 스냅샷의 저카디널리티 count만 노출한다. 구현보다 운영 설명이 중요하다. | 높음 | 낮음 | 3 |
| P29 | A | 15 | `feat(security): compare caller credential digests` | `WalletCredentials.authenticate()`, `WalletSecurityProperties` | 내부 API key 검증과 상수시간 비교 | 원문 key 보유를 줄이고 unknown caller도 동일 비교 경로로 처리한다. 정확한 입력 검증과 비밀 비노출을 구현하기 좋다. | 높음 | 높음 | 16, 17 |
| P30 | A | 15 | `feat(security): authenticate internal API keys` | `InternalApiKeyAuthenticationFilter.doFilterInternal()` | 중복 헤더 없는 호출자 인증 | 헤더가 없을 때와 일부만 있거나 여러 값인 경우를 구분하고, key를 SecurityContext에 남기지 않는다. P29에 통합한다. | 높음 | 높음 | 16, 17 |
| P31 | A | 15 | `feat(security): authorize wallet route capabilities` | `WalletSecurityConfig`, `WalletSecurityConfigTest` | 닫힌 allowlist와 최소 권한 | method+path+caller를 명시하고 나머지를 `denyAll`로 닫는다. Thread 9의 의미 권한과 합쳐 방어 계층을 묻기 좋다. | 높음 | 높음 | 9, 16 |
| P32 | A | 16 | `feat(api): parse wallet request identities` | `WalletRequestHeaders.requireIdempotencyKey()`, `requireCanonicalDebitId()` | 단일값 헤더와 정규 식별자 | `getHeader`가 숨길 수 있는 중복 값과 UUID의 여러 textual form을 거부해 요청 identity를 하나로 고정한다. | 높음 | 높음 | 6, 15 |
| P33 | C | 16 | `feat(api): define account representations` | `AccountResponse`, `BalanceResponse`, 요청·응답 DTO | DTO 변환과 반복 컨트롤러 배선 | 내부 필드를 숨기는 계약 판단은 중요하지만 개별 record와 단순 매핑은 독립 구현 문제로 만들 가치가 낮다. | 중간 | 낮음 | 15, 17 |
| P34 | B | 16 | `feat(settlement): expose adjustment endpoints` | `AdjustmentController`, `AdjustmentProofResponse` | 동기 200·비동기 202·proof 조회 의미 | BLOCKED 요청의 `202 + Location`과 이후 GET은 좋은 API 설계 질문이다. 상태 머신 구현은 Thread 10에서 준비한다. | 높음 | 낮음 | 10, 11, 17 |
| P35 | A | 17 | `feat(errors): shape wallet problem details` | `WalletError`, `WalletProblems`, `WalletExceptionHandler` | 안정적인 Problem Details와 비밀 비노출 | 저장된 업무 실패 facts를 그대로 재생하면서 key·SQL·예외 메시지를 반사하지 않는다. API 안정성과 보안을 함께 검증한다. | 높음 | 높음 | 6, 15, 16 |
| P36 | A | 17 | `feat(errors): map retryable database outages` | `PostgresFailureTranslator`, `WalletExceptionHandler` | 재시도 가능 DB 실패 분류 | lock·timeout·deadlock·serialization·연결 장애는 503으로, 영구 오류는 안전한 500으로 구분한다. P35에 통합한다. | 높음 | 높음 | 6, 7, 13 |

## 대표 Thread와 연관 Thread 관계

| 대표 면접 축 | 대표 Thread | 함께 묶은 Thread | 통합 이유 |
| --- | --- | --- | --- |
| 계정 상태와 원장 | 4, 5 | 1, 7, 9, 14 | 잔액 전이는 원장 쌍·업무 토폴로지·대사 규칙과 분리해서 설명할 수 없다. |
| 멱등성 최초 작성자 | 6 | 1, 2, 7, 10, 16, 17 | 권위 DB, 정규 request identity, 트랜잭션 잠금, 충돌·재생·오류 계약이 하나의 경쟁 프로토콜을 이룬다. |
| 원자적 금전 처리 | 7 | 4, 5, 6, 8, 12 | 계정 잠금 이후 잔액·원장·결과·이벤트 사실을 한 트랜잭션으로 커밋한다. |
| 조정과 복구 | 10, 11 | 4, 6, 7 | 최초 판정에서 BLOCKED proof를 만들고, 이후 FIFO worker가 같은 영속 상태를 완료한다. |
| 아웃박스 수명주기 | 12, 13 | 8, 9, 17 | 기록 원자성, 스트림 순서, lease 선점, 브로커 전송, fenced 완료, 재시도가 하나의 흐름이다. |
| 내부 보안 경계 | 15 | 9, 16, 17 | 자격 증명 인증, route capability, 업무 의미 권한, 요청 identity, 안전한 오류 응답을 계층별로 묶는다. |
| 영속 상태 대사 | 14 | 1, 4, 5, 6, 10, 11 | 정상 쓰기 경로의 invariant를 독립적인 repeatable-read 검증 경로로 다시 계산한다. |

## 상세 워크북 파일 위치

| 워크북 ID | 대표 주제 | 우선순위 | 상세 파일 | 포함된 선별 ID |
| --- | --- | --- | --- | --- |
| W01 | 계정 잔액 전이와 표현 범위 invariant | S | [`01-domain-invariants-and-ledger.md`](01-domain-invariants-and-ledger.md#w01) | P06 |
| W02 | 이중부기 쌍 복원과 업무 토폴로지 | S | [`01-domain-invariants-and-ledger.md`](01-domain-invariants-and-ledger.md#w02) | P07, P08, P15 일부 |
| W03 | 정규 요청 지문과 멱등성 충돌 | S | [`02-idempotency-transactions-and-concurrency.md`](02-idempotency-transactions-and-concurrency.md#w03) | P09 |
| W04 | 영속 최초 작성자 직렬화와 결과 재생 | S | [`02-idempotency-transactions-and-concurrency.md`](02-idempotency-transactions-and-concurrency.md#w04) | P10, P11 |
| W05 | 계정 잠금과 단일 트랜잭션 debit | S | [`02-idempotency-transactions-and-concurrency.md`](02-idempotency-transactions-and-concurrency.md#w05) | P12, P13 |
| W06 | 조정 최초 판정과 revision 선점 | S | [`03-adjustment-and-recovery-state-machines.md`](03-adjustment-and-recovery-state-machines.md#w06) | P17, P18 |
| W07 | 복구 부채 FIFO 회수와 bounded retry | S | [`03-adjustment-and-recovery-state-machines.md`](03-adjustment-and-recovery-state-machines.md#w07) | P19, P20 |
| W08 | 업무 트랜잭션 안의 아웃박스 append | A | [`04-outbox-recording-and-delivery.md`](04-outbox-recording-and-delivery.md#w08) | P14, P21 |
| W09 | 다중 작업자의 스트림별 FIFO 선점 | S | [`04-outbox-recording-and-delivery.md`](04-outbox-recording-and-delivery.md#w09) | P22 |
| W10 | lease fencing과 at-least-once 완료 | S | [`04-outbox-recording-and-delivery.md`](04-outbox-recording-and-delivery.md#w10) | P23, P24 |
| W11 | repeatable-read 원장 대사 | A | [`06-integrity-scanning.md`](06-integrity-scanning.md#w11) | P26, P27 |
| W12 | 내부 API key 인증 | A | [`05-security-http-and-errors.md`](05-security-http-and-errors.md#w12) | P29, P30 |
| W13 | 닫힌 route와 업무 의미 권한 | A | [`05-security-http-and-errors.md`](05-security-http-and-errors.md#w13) | P15 일부, P31 |
| W14 | 단일값 헤더와 canonical UUID | A | [`05-security-http-and-errors.md`](05-security-http-and-errors.md#w14) | P32 |
| W15 | 안정적인 오류 응답과 DB 실패 분류 | A | [`05-security-http-and-errors.md`](05-security-http-and-errors.md#w15) | P35, P36 |

## 상세 워크북 완전성 대조표

| 선별 ID | 우선순위 | 상태 | 상세 위치 또는 통합 대상 |
| --- | --- | --- | --- |
| P06 | S | 독립 작성 | W01 |
| P07 | S | 통합 작성 | W02의 원장 쌍 구성 조건 |
| P08 | S | 대표 작성 | W02 |
| P09 | S | 독립 작성 | W03 |
| P10 | S | 대표 작성 | W04 |
| P11 | A | 명시적 통합 | W04의 캐시 비권위 조건 |
| P12 | S | 대표 작성 | W05 |
| P13 | S | 명시적 통합 | W05의 중간 실패·전체 롤백 검증 |
| P14 | A | 명시적 통합 | W08의 성공·거절 이벤트 원자성 |
| P15 | A | 분할 통합 | W02의 원장 토폴로지, W13의 의미 권한 |
| P17 | S | 대표 작성 | W06 |
| P18 | S | 명시적 통합 | W06의 `(betId, revisionNumber)` 선점 |
| P19 | S | 대표 작성 | W07 |
| P20 | A | 명시적 통합 | W07의 포화형 지수 backoff |
| P21 | A | 대표 작성 | W08 |
| P22 | S | 독립 작성 | W09 |
| P23 | S | 대표 작성 | W10 |
| P24 | A | 명시적 통합 | W10의 retry·비동기 permit lifecycle |
| P26 | A | 구현 중심 통합 | W11의 snapshot/ledger 대사 |
| P27 | A | 대표 작성 | W11의 repeatable-read scanner |
| P29 | A | 대표 작성 | W12 |
| P30 | A | 명시적 통합 | W12의 헤더 모호성·SecurityContext 경계 |
| P31 | A | 대표 작성 | W13 |
| P32 | A | 독립 작성 | W14 |
| P35 | A | 대표 작성 | W15의 Problem Details |
| P36 | A | 명시적 통합 | W15의 retryable/permanent DB 분류 |

**검증 결과:** 모든 S/A 선별 항목은 독립 항목으로 작성되었거나 대표 항목에 명시적으로 통합되었다. 미배치 S/A 항목은 없다.

## 백지 구현 우선순위

1. **W04** — 영속 최초 작성자 직렬화와 결과 재생
2. **W05** — 계정 잠금과 단일 트랜잭션 debit
3. **W09** — 다중 작업자의 스트림별 FIFO 선점
4. **W06** — 조정 최초 판정과 revision 선점
5. **W07** — 복구 부채 FIFO 회수와 bounded retry
6. **W10** — lease fencing과 at-least-once 완료
7. **W01** — 계정 잔액 전이와 표현 범위 invariant
8. **W02** — 이중부기 쌍 복원과 업무 토폴로지
9. **W03** — 정규 요청 지문과 멱등성 충돌
10. **W08** — 업무 트랜잭션 안의 아웃박스 append
11. **W12** — 내부 API key 인증
12. **W13** — 닫힌 route와 업무 의미 권한
13. **W15** — 안정적인 오류 응답과 DB 실패 분류
14. **W11** — repeatable-read 원장 대사
15. **W14** — 단일값 헤더와 canonical UUID

## 설명 연습 우선순위

1. PostgreSQL만 정합성 권위를 갖고 Redis·Kafka는 힌트·전달 수단으로 제한한 이유
2. 멱등성 최초 작성자의 빠른 read, advisory lock, 잠금 후 재확인, 실패 후 승자 재조회 흐름
3. 계정 행 잠금과 잔액·원장·결과·아웃박스를 한 트랜잭션으로 묶은 이유
4. strict FIFO outbox와 lease version fencing이 각각 막는 실패
5. at-least-once 중복이 발생하는 정확한 crash window와 소비자 deduplication 책임
6. BLOCKED adjustment와 recovery debt/FIFO head/outbound freeze 사이 invariant
7. route 권한과 credit source/reason 의미 권한을 둘 다 둔 이유
8. 저장된 업무 거절 facts를 재계산하지 않고 그대로 재생하는 이유
9. repeatable-read integrity scan과 저카디널리티 metric snapshot의 역할
10. timestamp가 인과 순서 권위가 아니고 sequence·status·version이 권위인 이유
11. Testcontainers 기반 semantic gate가 단위 테스트만으로 찾기 어려운 경계를 검증하는 이유

## 한 문제로 통합한 Thread 묶음

- **W01–W02:** Thread 4 + 5 + 9의 잔액·원장·토폴로지 규칙
- **W03–W04:** Thread 1 + 2 + 6 + 16의 권위 상태·정규 identity·최초 작성자 재생
- **W05:** Thread 4 + 5 + 6 + 7 + 8 + 12의 원자적 debit 커밋
- **W06–W07:** Thread 6 + 10 + 11의 조정 proof와 후속 recovery 상태 머신
- **W08–W10:** Thread 8 + 9 + 12 + 13의 이벤트 기록·선점·전달·완료 lifecycle
- **W11:** Thread 1 + 4 + 5 + 6 + 10 + 11 + 14의 영속 invariant 대사
- **W12–W14:** Thread 9 + 15 + 16의 인증·route capability·업무 의미 권한·요청 identity
- **W15:** Thread 6 + 15 + 16 + 17의 영속 실패 재생·비밀 비노출·재시도 계약
