# DevThread 개발자 기술면접 마스터 인덱스

이 문서는 현재 GPT 프로젝트에 축적된 Android/Kotlin DevThread 기록만을 기준으로 작성한 전체 선별표이자 상세 워크북의 기준 문서다. Thread 번호는 원본 문서 내부의 `Mxx` 표기를 따른다. 현재 기록에서 커밋 제목이 직접 노출되지 않은 변경은 임의로 복원하지 않고 `현재 기록에서 커밋 제목 미노출`로 표시했다.

확인 가능한 작업 범위는 M01~M10, M14, M15다. M11~M13에 해당하는 원본 Thread 문서는 현재 프로젝트 기록에서 확인되지 않아 평가·추정하지 않았다. 보안은 이 프로젝트에서 독립적인 핵심 구현 축으로 확인되지 않았으므로 억지로 면접 항목을 만들지 않았고, mutation identity 무결성처럼 실제 기록에 나타난 경계만 관련 항목에 포함했다.

## 우선순위 기준

- **S**: 반드시 준비. 질문과 10~30분 직접 구현 모두 가치가 높다.
- **A**: 준비 가치가 높다. 질문 가능성이 높고 핵심 일부를 직접 구현할 수 있다.
- **B**: 구현보다는 설계, 실패 재현, 시스템 경계 설명이 중요하다. 상세 워크북은 대표 S/A 문제에 통합한다.
- **C**: 보고·하네스·좁은 회귀 수리 성격이 강해 별도 면접 문제로 만들 필요가 낮다.

`질문형`과 `구현형`은 해당 커밋 자체를 독립 면접 지점으로 다룰 때의 가치다. `높음`, `중간`, `낮음`은 상대적 우선순위이며 채점 기준이 아니다.

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| A | M01 | `feat(android): establish memory-only Item CRUD` | `MemoryItemStore`, `Item`, `ItemRow`, `toRows()`, `ItemScreen` | 안정적인 식별자, 결정론적 상태 전이, ID·시계 주입, 불변 스냅샷 | 작은 구현이지만 이후 영속성·동기화·UI 복원의 기반 invariant를 만든다. 자료구조와 상태 모델 기본기를 직접 확인하기 좋다. | 높음 | 높음 | M02, M08 |
| S | M02 | `feat(android): persist Items across app process restart` | `ItemStore`, `ItemDao`, `ItemDatabase`, `MainActivity`, `verification/process_restart.py` | 저장소 경계, read-after-write, 트랜잭션, DB 생명주기, 프로세스 사망 후 복원 | 메모리 상태와 내구성 상태의 경계를 실제로 바꾼 대표 커밋이다. 오류 시 rollback과 resource cleanup까지 함께 질문할 수 있다. | 높음 | 높음 | M01, M04, M05, M15 |
| A | M03 | `feat(android): add M03 foreground sync candidate` | `ItemSync`, `HttpItemRemote`, `ItemRemote`, `ItemDatabaseTest#canonicalReplacementRollsBackIfAnyRowCannotBeInserted` | 로컬 diff 계획, 순차 mutation, canonical GET, Room 단일 진실 공급원, 부분 성공 | 동기화 알고리즘과 시스템 경계를 설명하기 좋다. 다만 메모리 snapshot 한계는 M05 이후 구조로 대체되므로 축소 구현이 적절하다. | 높음 | 중간 | M04, M05, M06, M07 |
| B | M03 | 현재 기록에서 커밋 제목 미노출(HTTP/1.0 연결 재사용 수리) | `HttpItemRemote`, OkHttp, `fixture/server.py`, `network_security_config.xml` | 연결 재사용, EOF 진단, mutation 재시도 위험, transport contract | 원인 분리와 재시도 판단은 중요하지만 실제 제품 변경은 좁고 fixture 특성이 강하다. W05의 꼬리 질문으로 통합한다. | 높음 | 낮음 | M03, M06 |
| S | M04 | `feat(android): add M04 refresh status and saved success time` | `SyncState`, `SyncStatus`, `readSavedStatus()`, `markLocalChange()`, `synchronize()`, `sync_metadata` | 동기화 상태 머신, transient error와 durable metadata 분리, 취소, 원자적 canonical 교체 | 상태 invariant와 실패 의미가 명확하며, UI·저장소·네트워크 경계를 한 문제에서 확인할 수 있다. | 높음 | 높음 | M03, M05, M08 |
| C | M04 | `docs(android): record failed M04 initial verification` | `verification/M04.md` | 실패 원장, 실행 증거, 미검증 범위 표시 | 검증 정직성과 추적성은 중요하지만 제품 설계나 일반화 가능한 독립 구현 문제는 아니다. | 낮음 | 낮음 | M04 |
| C | M04 | `test(android): repair M04 harness assertions` | `ItemDatabaseTest#v1MigrationPreservesItemsAndRefreshTimeSurvivesReopen`, `ItemUiTest#assertM04Status` | 테스트 메서드 contract, 정확한 오류 표시 assertion | 회귀 하네스의 좁은 수리다. 테스트 설계 설명에는 참고할 수 있으나 별도 면접 포인트로 만들 가치가 낮다. | 낮음 | 낮음 | M04 |
| C | M04 | 현재 기록에서 커밋 제목 미노출(UI readiness 대기 수리) | `ItemUiTest#awaitCrudText` | Compose idleness, IME·focus readiness, 비동기 UI 테스트 동기화 | 실제 flaky 경계를 보여 주지만 테스트 전용 대기 조건에 가깝다. W03의 실패 재현·검증 항목에 통합한다. | 중간 | 낮음 | M01, M04 |
| A | M04 | 현재 기록에서 커밋 제목 미노출(focus 순서 수리) | `MainActivity.kt`, `ItemScreen`, `accessStorage`, `focus.clearFocus()` | focus target 생명주기, 비활성화 전 정리 순서, async UI 상태 전이 | 프레임워크 암기보다 resource/lifecycle 순서를 판단하게 하는 좋은 문제다. 작은 구현과 설명 모두 가능하다. | 높음 | 중간 | M01, M08 |
| B | M05 | 현재 기록에서 커밋 제목 미노출(오프라인 intent 유실 기준선) | `verification/offline_queue_restart.py`, `M05ScenarioInstrumentation` | 저장된 최종 상태와 유실된 사용자 의도, 실제 PID 교체 기반 실패 재현 | M04 방식의 한계를 명확히 드러내지만 제품 구현은 없다. W07의 문제 배경과 실패 조건으로 통합한다. | 높음 | 낮음 | M04, M05 |
| S | M05 | `feat(android): persist ordered offline mutation intents` | `ItemStore`, `ItemDao`, `pending_mutations`, `M05ScenarioInstrumentation` | transactional outbox, 로컬 변경과 intent의 원자성, FIFO drain, 삭제 intent 보존 | 오프라인 우선 시스템의 핵심 invariant다. 트랜잭션·순서·복구를 10~30분 문제로 축소하기 좋다. | 높음 | 높음 | M04, M06, M09 |
| B | M06 | `test(android): freeze M06 lost-acknowledgment baseline` | `verification/M06-inputs.json`, `fixture/server.py` | 원격 commit 후 응답 유실, 결과 불확실성, exact replay 필요성 | 분산 시스템의 중요한 실패 창을 고정하지만 구현 커밋은 아니다. W09·W10의 실패 조건으로 통합한다. | 높음 | 낮음 | M05, M06 |
| S | M06 | 현재 기록에서 커밋 제목 미노출(멱등 재전송·수신확인 구현) | `PendingMutation`, `MutationRequest`, `AcknowledgedMutation`, `MutationResult`, `ItemStore.acknowledge`, `ItemSync.synchronize` | canonical 요청 hash, idempotency key, exact cached response, receipt/dequeue 원자성, terminal identity conflict | 네트워크의 불확실한 결과를 로컬·서버 양쪽 invariant로 해결한다. 질문과 직접 구현 가치가 모두 매우 높다. | 높음 | 높음 | M05, M07, M09, M10 |
| C | M06 | `M06 preserve unverified matcher repair at global stop` | `ItemUiTest#m06ActualHttpCollisionStaysVisibleAndTerminalAfterRoomReopen` | UI matcher 수리와 미검증 상태 보존 | 제품 invariant를 바꾸지 않는 제한된 테스트 수리다. 원본 실패 기록은 W09의 검증 관점에서만 참고한다. | 낮음 | 낮음 | M06 |
| B | M07 | `test(M07): freeze stale mutation baseline` | `verification/M07.md`, `M05ScenarioInstrumentation` | stale base로 보낸 mutation, 409 canonical conflict, 프로세스 경계 재현 | 동시 변경 실패를 구체화하지만 제품 구현은 없다. W11·W12의 시나리오로 통합한다. | 높음 | 낮음 | M06, M07 |
| S | M07 | `feat(M07): preserve stale mutation conflicts and acknowledged bases` | `PendingMutation.baseVersion`, `prepareDispatch(sequence)`, `ItemStore.acknowledge`, `tombstones` | optimistic concurrency, immutable prepared envelope, terminal conflict, successor rebase, tombstone | 같은 Item의 연속 로컬 intent와 원격 변경을 함께 다루는 고밀도 invariant 문제다. | 높음 | 높음 | M06, M08, M09 |
| B | M08 | `test(M08): freeze actual Activity state baseline` | `M08ActivityStateTest#frozenM08ActivityLifecycleSequence` | Activity recreation과 background/foreground 경계, 실제 UI·DB checkpoint | 모바일 lifecycle 경계를 신뢰성 있게 재현하지만 제품 변경은 없다. W04의 검증 조건으로 통합한다. | 높음 | 낮음 | M04, M08, M09 |
| A | M08 | `feat(M08): retain editor state across Activity recreation` | `ItemEditor`, `ItemEditor.Saver`, `ItemEditor.submit`, `rememberSaveable`, `flowWithLifecycle` | saveable UI state, lifecycle-aware collection, 미제출 draft와 영속 데이터 분리 | Android lifecycle 기본기와 상태 소유권 판단을 확인하기 좋다. 구현 범위는 editor reducer와 saver로 축소한다. | 높음 | 중간 | M04, M09, M10 |
| B | M09 | `test(M09): preserve actual process-death baseline` | `verification/startup_recovery.py`, `M05ScenarioInstrumentation` | ordinary cold startup, in-flight kill, test injection과 제품 경계 | 실제 프로세스 사망을 재현하는 가치가 크지만 구현은 없다. W08의 실패 시나리오로 통합한다. | 높음 | 낮음 | M06, M08, M09 |
| S | M09 | 현재 기록에서 커밋 제목 미노출(일반 시작 복구 구현) | `ItemSync.recoverPending`, `ItemScreen`, `LaunchedEffect`, `accessStorage` | 로컬 우선 시작 순서, nonterminal queue 자동 복구, 1회 시도, terminal 건너뛰기 | 프로세스 사망 뒤 저장된 intent를 제품의 일반 진입점에서 복구하는 핵심 경계다. 작고 명확한 상태 문제로 구현할 수 있다. | 높음 | 높음 | M05, M06, M08, M10 |
| B | M10 | `test(M10): preserve accepted foreground-only baseline` | `verification/background_baseline.py` | 앱 진입점이 없을 때 foreground recovery가 동작하지 않는 한계, OS scheduler 경계 | M09와 M10의 책임 차이를 설명하기 좋지만 제품 변경은 없다. W13의 문제 배경으로 통합한다. | 높음 | 낮음 | M09, M10 |
| S | M10 | `feat: schedule durable sync with bounded persistent work` | `ItemSyncWorker`, `ItemWorkScheduler`, `AutomaticItemSync`, `AutomaticResult`, `ItemApplication`, `item-automatic-sync` | WorkManager unique work, durable HTTP reservation, process 사망을 넘는 retry ceiling, network constraint, DB cleanup | OS·앱·저장소 경계를 모두 다루며 일반화 가치가 높다. retry와 resource invariant를 직접 구현할 수 있다. | 높음 | 높음 | M09, M14 |
| B | M14 | `test: verify M14 pressure cancellation on unchanged product` | `ItemSync.synchronize`, `itemSyncMutex`, `M14-inputs.json` | single-flight, 느린 네트워크 역압력, durable ACK 뒤 취소, bounded concurrency | 제품 변경 없이 M10의 동시성 invariant를 검증한 Thread다. 설명 가치가 높고 구현은 W14에 M10 대표 문제로 통합한다. | 높음 | 중간 | M10 |
| S | M15 | `feat: bound M15 release reads and virtualize item pages` | `ItemDao.countItems`, `ItemDao.readPageRows`, `ItemStore.page`, `LazyColumn`, `M15StorageTest` | bounded I/O, `LIMIT/OFFSET`, 성공 후 offset 공개, UI virtualization, 시간·공간 복잡도 | 대용량 로컬 상태에서 실제 병목을 제거한다. DB·상태·UI의 경계를 한정된 페이징 문제로 직접 구현하기 좋다. | 높음 | 높음 | M02, M10, M14 |

### 분포

- **S 8개**: M02, M04 상태, M05, M06, M07, M09, M10, M15
- **A 4개**: M01, M03, M04 focus, M08
- **B 8개**: M03 transport, M05·M06·M07·M08·M09·M10 기준선, M14
- **C 4개**: M04 보고·하네스·readiness 수리, M06 matcher 수리

## 대표 Thread와 연관 Thread 관계

| 상세 ID | 대표 Thread | 대표 면접 주제 | 명시적으로 연결한 연관 Thread | 상세 문서 |
|---|---|---|---|---|
| W01 | M01 | 안정적인 ID와 결정론적 메모리 상태 | M02의 영속성 경계, M08의 UI identity | [01-state-persistence-and-lifecycle.md#w01](01-state-persistence-and-lifecycle.md#w01) |
| W02 | M02 | 저장소 추상화와 process-safe 영속성 | M01의 메모리 모델, M04의 metadata, M05의 트랜잭션, M15의 bounded query | [01-state-persistence-and-lifecycle.md#w02](01-state-persistence-and-lifecycle.md#w02) |
| W03 | M04 | focus 제거와 비활성화 순서 | M01 UI 회귀, M04 readiness 수리, M08 lifecycle | [01-state-persistence-and-lifecycle.md#w03](01-state-persistence-and-lifecycle.md#w03) |
| W04 | M08 | saveable editor와 lifecycle-aware 상태 수집 | M04 sync 상태, M08 baseline, M09 process death | [01-state-persistence-and-lifecycle.md#w04](01-state-persistence-and-lifecycle.md#w04) |
| W05 | M03 | diff 기반 foreground sync와 canonical 교체 | M03 HTTP/1.0 transport 수리, M04 상태, M05 큐 | [02-sync-state-queue-and-recovery.md#w05](02-sync-state-queue-and-recovery.md#w05) |
| W06 | M04 | 동기화 상태 머신과 durable success time | M03 canonical sync, M05 intent, M08 UI 상태 | [02-sync-state-queue-and-recovery.md#w06](02-sync-state-queue-and-recovery.md#w06) |
| W07 | M05 | transactional outbox와 FIFO drain | M05 intent 유실 기준선, M04 local state, M06 idempotency, M09 recovery | [02-sync-state-queue-and-recovery.md#w07](02-sync-state-queue-and-recovery.md#w07) |
| W08 | M09 | cold-start pending recovery | M09 baseline, M05 queue, M06 exact replay, M08 local-first rendering, M10 scheduler | [02-sync-state-queue-and-recovery.md#w08](02-sync-state-queue-and-recovery.md#w08) |
| W09 | M06 | canonical hash와 exact replay | M06 lost-ACK 기준선, M05 immutable intent, M07 conflict identity | [03-idempotency-and-conflict.md#w09](03-idempotency-and-conflict.md#w09) |
| W10 | M06 | receipt 기록과 dequeue의 원자성 | M05 outbox transaction, M09 replay recovery, M10 worker retry | [03-idempotency-and-conflict.md#w10](03-idempotency-and-conflict.md#w10) |
| W11 | M07 | optimistic concurrency와 terminal conflict | M07 stale baseline, M06 identity conflict, M08/M09 재시작 | [03-idempotency-and-conflict.md#w11](03-idempotency-and-conflict.md#w11) |
| W12 | M07 | ACK 뒤 successor rebase | M06 exact receipt, M07 baseVersion·tombstone, M09 replay | [03-idempotency-and-conflict.md#w12](03-idempotency-and-conflict.md#w12) |
| W13 | M10 | persistent unique work와 durable retry budget | M10 foreground-only 기준선, M09 startup recovery, M14 pressure | [04-background-concurrency-and-performance.md#w13](04-background-concurrency-and-performance.md#w13) |
| W14 | M10·M14 | single-flight, cancellation, backpressure | M10 scheduler/worker, M14 unchanged-product 검증 | [04-background-concurrency-and-performance.md#w14](04-background-concurrency-and-performance.md#w14) |
| W15 | M15 | bounded paging와 UI virtualization | M02 Room 경계, M14 압력 관찰, M10 background I/O 책임 | [04-background-concurrency-and-performance.md#w15](04-background-concurrency-and-performance.md#w15) |

## 상세 워크북 파일 구성

### `01-state-persistence-and-lifecycle.md`

- [W01. 안정적인 식별자와 결정론적 로컬 상태](01-state-persistence-and-lifecycle.md#w01)
- [W02. 저장소 경계, 원자적 영속성, resource lifetime](01-state-persistence-and-lifecycle.md#w02)
- [W03. focus target을 제거하기 전 명시적으로 focus를 정리하는 순서](01-state-persistence-and-lifecycle.md#w03)
- [W04. Activity recreation에서 미제출 editor 상태 복원](01-state-persistence-and-lifecycle.md#w04)

### `02-sync-state-queue-and-recovery.md`

- [W05. 로컬 변경 계획과 canonical 상태 교체 경계](02-sync-state-queue-and-recovery.md#w05)
- [W06. 동기화 상태 머신과 저장된 성공 시각](02-sync-state-queue-and-recovery.md#w06)
- [W07. 로컬 변경과 mutation intent를 함께 커밋하는 durable queue](02-sync-state-queue-and-recovery.md#w07)
- [W08. ordinary cold start에서 nonterminal pending work 복구](02-sync-state-queue-and-recovery.md#w08)

### `03-idempotency-and-conflict.md`

- [W09. canonical 요청과 mutation identity를 이용한 exact replay](03-idempotency-and-conflict.md#w09)
- [W10. acknowledgment receipt와 dequeue의 원자적 커밋](03-idempotency-and-conflict.md#w10)
- [W11. baseVersion을 이용한 terminal version conflict](03-idempotency-and-conflict.md#w11)
- [W12. 앞선 ACK를 반영하면서 뒤 intent를 rebase하는 방법](03-idempotency-and-conflict.md#w12)

### `04-background-concurrency-and-performance.md`

- [W13. 내구성 있는 unique work와 프로세스 사망을 넘는 retry 예산](04-background-concurrency-and-performance.md#w13)
- [W14. 겹치는 drain의 single-flight와 cancellation/backpressure](04-background-concurrency-and-performance.md#w14)
- [W15. bounded local read와 UI virtualization](04-background-concurrency-and-performance.md#w15)

## S/A 완전성 검증

아래 표는 위 전체 선별표의 모든 S/A 항목을 대조한 결과다. B/C 항목은 별도 상세 문제를 만들지 않고 관련 S/A의 배경·꼬리 질문·검증 조건으로만 사용했다.

| 선별 항목 | 우선순위 | 상세 워크북 대응 | 상태 |
|---|---|---|---|
| M01 `feat(android): establish memory-only Item CRUD` | A | W01 | 독립 상세 작성 완료 |
| M02 `feat(android): persist Items across app process restart` | S | W02 | 독립 상세 작성 완료 |
| M03 `feat(android): add M03 foreground sync candidate` | A | W05 | 독립 상세 작성 완료. 같은 Thread의 transport 수리는 꼬리 질문에 통합 |
| M04 `feat(android): add M04 refresh status and saved success time` | S | W06 | 독립 상세 작성 완료 |
| M04 focus 순서 수리 | A | W03 | 독립 상세 작성 완료. readiness·IME 하네스 경계도 검증 항목에 통합 |
| M05 `feat(android): persist ordered offline mutation intents` | S | W07 | 독립 상세 작성 완료. 기준선은 실패 조건에 통합 |
| M06 멱등 재전송·수신확인 구현 | S | W09, W10 | canonical identity와 local ACK transaction을 두 독립 상세 항목으로 작성 완료 |
| M07 `feat(M07): preserve stale mutation conflicts and acknowledged bases` | S | W11, W12 | terminal conflict와 successor rebase를 두 독립 상세 항목으로 작성 완료 |
| M08 `feat(M08): retain editor state across Activity recreation` | A | W04 | 독립 상세 작성 완료. baseline lifecycle sequence를 자가 검증에 통합 |
| M09 일반 시작 복구 구현 | S | W08 | 독립 상세 작성 완료. actual process-death baseline을 실패 조건에 통합 |
| M10 `feat: schedule durable sync with bounded persistent work` | S | W13, W14 | persistent retry와 single-flight/backpressure를 두 독립 상세 항목으로 작성 완료 |
| M15 `feat: bound M15 release reads and virtualize item pages` | S | W15 | 독립 상세 작성 완료 |

**검증 결과:** 누락된 S/A 항목 없음. 상세 항목은 총 15개이며 모든 항목에 면접 질문, 30초 답변, 핵심 키워드, 백지 구현, 자가 검증, 구현 후 설명, 원본 확인 위치가 포함되어 있다.

## 백지 구현 우선순위

1. [W09 canonical 요청과 exact replay](03-idempotency-and-conflict.md#w09) — deterministic serialization과 idempotency invariant
2. [W10 receipt/dequeue 원자성](03-idempotency-and-conflict.md#w10) — 트랜잭션 실패 창과 compare-and-delete
3. [W12 successor rebase](03-idempotency-and-conflict.md#w12) — 연속 intent와 observed version 전파
4. [W07 durable mutation queue](02-sync-state-queue-and-recovery.md#w07) — local state와 outbox의 원자적 커밋
5. [W11 optimistic version conflict](03-idempotency-and-conflict.md#w11) — terminal conflict와 canonical winner
6. [W13 persistent retry budget](04-background-concurrency-and-performance.md#w13) — 프로세스 사망을 넘는 reservation 상태 머신
7. [W15 bounded paging](04-background-concurrency-and-performance.md#w15) — DB query, offset invariant, complexity
8. [W06 sync 상태 머신](02-sync-state-queue-and-recovery.md#w06) — transient/durable 상태 분리와 cancellation
9. [W02 영속 저장소 경계](01-state-persistence-and-lifecycle.md#w02) — transaction, readback, close
10. [W05 sync plan](02-sync-state-queue-and-recovery.md#w05) — 결정적 diff와 canonical replace
11. [W14 single-flight와 cancellation](04-background-concurrency-and-performance.md#w14) — mutex와 durable progress
12. [W08 cold-start 복구](02-sync-state-queue-and-recovery.md#w08) — local-first, terminal filtering, 1회 시도
13. [W04 saveable editor](01-state-persistence-and-lifecycle.md#w04) — reducer·Saver·lifecycle
14. [W01 안정적인 로컬 상태](01-state-persistence-and-lifecycle.md#w01) — ID·시계 주입과 불변 스냅샷
15. [W03 focus 순서](01-state-persistence-and-lifecycle.md#w03) — 작은 lifecycle ordering 문제

## 설명 연습 우선순위

1. **W09·W10**: 응답 유실, exactly-once가 아닌 at-least-once 전송, idempotent effect, local ACK transaction의 차이
2. **W11·W12**: optimistic concurrency, terminal conflict, prepared envelope 불변성, successor rebase
3. **W13·W14**: 앱 시작 복구와 OS persistent work의 책임 차이, retry budget, unique work, single-flight, 취소
4. **W07·W08**: 최종 상태 저장과 사용자 intent 저장의 차이, 프로세스 사망 후 복구 순서
5. **W02·W06**: durable state와 session state, transaction/readback, 성공 시각의 의미
6. **W15**: unbounded read의 비용, `LIMIT/OFFSET` trade-off, virtualization과 데이터 정합성
7. **W05**: diff 기반 초기 동기화의 장점과 부분 성공·프로세스 사망 한계
8. **W04**: configuration recreation과 process death, saveable state와 DB state의 책임 분리
9. **W03**: focus target 제거 전에 resource를 정리해야 하는 이유와 테스트 idleness의 한계
10. **W01**: stable identity, deterministic dependency injection, immutable state exposure

## 한 문제로 통합한 Thread 묶음

- **M03 feature + M03 HTTP/1.0 연결 수리 → W05**: foreground sync 알고리즘을 대표 문제로 두고 transport 수리는 재시도·연결 contract 꼬리 질문으로 통합했다.
- **M04 상태 feature + 실패 보고·하네스·readiness 기록 → W06·W03**: 상태 머신은 W06, 실제 focus/lifecycle 순서는 W03으로 분리했으며 보고·테스트 전용 커밋은 자가 검증 근거로만 사용했다.
- **M05 intent 유실 기준선 + durable queue feature → W07**: 기준선은 "왜 현재 Item만 저장해서는 안 되는가"라는 실패 조건으로 통합했다.
- **M06 lost-ACK 기준선 + 멱등 구현 + matcher 수리 → W09·W10**: 서버 exact replay와 클라이언트 receipt/dequeue transaction을 서로 다른 invariant로 분리했다.
- **M07 stale baseline + conflict/rebase feature → W11·W12**: terminal conflict 보존과 성공 ACK 뒤 후속 intent 재기반화를 별도 문제로 구성했다.
- **M08 lifecycle baseline + editor restoration feature → W04**: 실제 Activity sequence를 자가 검증 경계로 사용했다.
- **M09 process-death baseline + ordinary startup recovery → W08**: 테스트용 초기 주입과 일반 제품 진입점의 책임 차이를 하나의 복구 문제로 통합했다.
- **M10 foreground-only baseline + persistent work feature + M14 pressure 검증 → W13·W14**: durable scheduler/retry와 single-flight/cancellation을 분리하되 같은 자동 동기화 축으로 묶었다.
- **M15 baseline과 paging fix → W15**: 2,000행 unbounded read 재현과 25행 bounded page/virtualized UI를 하나의 성능 문제로 구성했다.
