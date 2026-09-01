# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

이 문서는 현재 GPT 프로젝트에 축적된 DevThread Markdown 기록만을 근거로 작성한 전체 선별표다. 원격 저장소의 현재 상태는 전제로 삼지 않았고, 기록에서 정확한 커밋 제목을 확인하지 못한 경우에는 임의로 복원하지 않고 **`기록에서 정확한 메시지 미확인`**으로 표시했다.

파일명은 작업 묶음의 순서를 따르지만, 원문 내부 마일스톤 표기를 우선한다. 따라서 `11-slow-network-and-sync-backpressure.md`는 **M14**, `12-cold-start-large-local-state-and-release-validation.md`는 **M15**로 적는다.

## 우선순위 기준

- **S**: 질문과 10~30분 직접 구현 모두 가치가 높다. 반드시 준비한다.
- **A**: 질문 또는 핵심 축소 구현으로 준비할 가치가 높다.
- **B**: 별도 구현 문제보다는 설계 설명, 실패 경계, 검증 전략을 준비한다.
- **C**: 설정·증빙·메타데이터 중심이라 별도 면접 문제로 만들 가치가 낮다.

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---|---|---|---|---|---|---|---|
| A | M01 | `feat: establish in-memory React Native Item UI` | `src/items.ts`의 `Item`, `itemsReducer`; `src/App.tsx`의 `App`, `saveTitle` | 순수 상태 전이, 안정적인 식별자, 불변식 | 프레임워크 문법보다 상태 모델과 도메인 불변식을 설명·구현하게 만들 수 있다. 상세 P01. | 높음 | 중간 | M02, M08 |
| B | M01 | `test: verify M01 sequence on actual Android controls` | `android/app/src/androidTest/java/com/mse/reactnative/M01ItemUiTest.java`; `M01ItemUiTest.fixedM01SequenceOnRealReactNativeUi` | 실제 네이티브 UI 경계 검증 | 호스트 렌더러와 실제 Android 컨트롤의 차이를 설명하기 좋지만 독립 구현 문제 가치는 낮다. | 중간 | 낮음 | M02, M08 |
| S | M02 | 기록에서 정확한 메시지 미확인 — SQLite 영속화 구현 구간 | `src/itemStore.ts`의 `ItemStore`, `SqliteItemStore`, `openItemStore`, `mutate`, `itemToRow`, `rowToItem`; `src/App.tsx` | 로컬 트랜잭션, ID 할당, COMMIT 뒤 상태 공개 | Item 변경과 ID allocator를 한 트랜잭션으로 묶고, 실패 시 UI와 디스크 모두 이전 상태를 유지하는 핵심 정합성 문제다. 상세 P02. | 높음 | 높음 | M04, M05, M06 |
| A | M02 | 기록에서 정확한 메시지 미확인 — 스키마 버전 처리 구간 | `src/itemStore.ts`의 `SCHEMA_VERSION`, `openItemStore`; `__tests__/items.test.ts` | 스키마 생성·마이그레이션·미지원 버전 거부 | "업그레이드 실패가 기존 데이터를 훼손하지 않는다"는 일반화 가능한 저장소 설계 주제다. M04 이후 마이그레이션과 통합해 상세 P03. | 높음 | 중간 | M04, M05, M06, M07 |
| C | M02 | `test: verify SQLite compatibility patch without blank context` | `patches/react-native-sqlite-2+3.6.3.patch` | 의존성 호환 패치 검증 | 재현성 측면의 의미는 있지만 특정 패키지·빌드 설정에 치우쳐 별도 면접 준비 효율이 낮다. | 낮음 | 낮음 | — |
| B | M03 | `test: reproduce isolated local SQLite instances` | `__tests__/items.test.ts`의 독립 DB 격리 회귀; `scripts/verify_m03.py` | 로컬 복제본 격리와 동기화 필요성 | 문제를 드러내는 좋은 재현이지만 해법 자체는 다음 기능 커밋에 포함된다. | 중간 | 낮음 | M02, M03 |
| A | M03 | `feat: add explicit foreground synchronization` | `src/sync.ts`의 `JsonRequest`, `SyncSession`, `ForegroundSync`, `exchange`, `synchronize`; `src/itemStore.ts`의 `replaceSnapshot` | 기준 스냅샷 diff, 원격 검증, 저장소를 통한 공개 | 클라이언트 기준선과 현재 로컬 상태로 요청을 만들고, 서버 응답이 아니라 커밋된 SQLite 읽기를 UI에 공개하는 경계가 좋다. 상세 P04. | 높음 | 높음 | M04, M05 |
| B | M03 | `feat: add explicit foreground synchronization`의 Android 통신 경계 | `AndroidManifest.xml`; `network_security_config.xml`; `MainActivity.kt`; `fixture/server.cjs` | 최소 권한·테스트 전용 cleartext 범위·실제 HTTP 경로 | 보안과 테스트 경계를 설명하기 좋지만, 핵심 면접 포인트는 동기화 알고리즘에 통합한다. | 중간 | 낮음 | M04, M06 |
| B | M04 | `test: reproduce missing offline refresh status` | `fixture/server.cjs`; `scripts/verify_m04.py`; `__tests__/App.test.tsx` | 실패 상태 재현과 캐시 보존 검증 | 실패를 먼저 고정한 테스트 전략은 중요하지만 독립 구현보다 P08의 검증 항목으로 다루는 편이 낫다. | 중간 | 낮음 | M03, M04 |
| A | M04 | 기록에서 정확한 메시지 미확인 — 오프라인 읽기·새로고침 구현 구간 | `src/App.tsx`; `src/itemStore.ts`의 `readLastSuccessfulRefresh`, `replaceSnapshot`; `src/sync.ts` | 오프라인 우선 UI 상태, 스냅샷과 성공 시각의 원자성 | 네트워크 실패 중에도 마지막 커밋 스냅샷을 유지하고 "fresh"의 의미를 정확히 제한하는 설계가 면접 가치가 높다. 상세 P08; 마이그레이션은 P03에 통합. | 높음 | 중간 | M03, M09 |
| C | M04 | `docs: record M04 runtime checks and trailer limitation` | `verification/M04.md` | 런타임 증빙과 Git trailer 결함 기록 | 감사 가능성은 높지만 제품 설계·기본기 질문으로 일반화하기 어렵다. | 낮음 | 낮음 | — |
| C | M04 | `docs: record M04 trailer-only repair` | `verification/M04.md` | 커밋 메타데이터 정규화 | 작업 이력 관리에는 유용하지만 별도 기술면접 항목으로 만들 필요가 낮다. | 낮음 | 낮음 | — |
| B | M05 | `test: reproduce lost upload intent after offline process death` | `scripts/verify_m05.py`; `verification/M05-inputs.json`; `__tests__/sync.test.ts` | 프로세스 종료 뒤 업로드 의도 유실 재현 | 실제 실패 경계를 명확히 하지만 해법은 P05의 내구성 큐 문제에 통합한다. | 높음 | 낮음 | M02, M05 |
| S | M05 | 기록에서 정확한 메시지 미확인 — durable sync queue 구현 구간 | `src/itemStore.ts`의 `PendingMutation`, `readPending`; `src/sync.ts`의 `ForegroundSync.synchronize`; `src/App.tsx` | transactional outbox, FIFO drain, 실패 시 보존 | 로컬 변경과 업로드 의도를 한 번에 커밋하고, 재시작·오프라인·부분 실패에서 순서를 지키는 대표적인 시스템 설계 문제다. 상세 P05. | 높음 | 높음 | M02, M06, M09, M10 |
| C | M05 | `docs: record verified M05 Android durability` | `verification/M05.md` | 실기기 내구성 증빙 | 상세 검증 자료로는 가치가 있지만 독립 질문보다 P05 자가 검증에 흡수한다. | 중간 | 낮음 | M05 |
| B | M06 | `test(M06): reproduce lost acknowledgement on verified M05` | `fixture/server.cjs`; `scripts/verify_m06.py`; `verification/M06-inputs.json` | 서버 적용 뒤 응답 유실 재현 | "요청 결과를 모르는 상태"라는 핵심 분산 시스템 실패를 고정하지만, 구현 문제는 다음 두 포인트에 통합한다. | 높음 | 낮음 | M05, M06 |
| S | M06 | `feat(M06): persist mutation identity and atomically acknowledge replay` | `src/mutationProtocol.ts`의 `canonicalJson`, `mutationHash`, `mutationTarget`; `src/itemStore.ts`의 pending identity/hash 필드 | 멱등성 키, canonical JSON, payload hash | 같은 요청의 재전송과 동일 키의 다른 요청을 구분하는 프로토콜 설계다. 해시가 인증이 아니라 동일성 검증이라는 한계까지 묻기 좋다. 상세 P06. | 높음 | 높음 | M05, M07, M10 |
| S | M06 | `feat(M06): persist mutation identity and atomically acknowledge replay` | `src/itemStore.ts`의 `acknowledge`, `rejectIdentity`; `sync_metadata.last_acknowledgement`; `pending_mutations` | ACK·receipt·dequeue 원자성, 순서 검증 | 외부 효과와 로컬 커밋 사이의 틈, out-of-order/foreign ACK, 후속 로컬 편집 보존을 동시에 다루는 고난도 정합성 문제다. 상세 P07. | 높음 | 높음 | M05, M07, M10 |
| B | M07 | `test(M07): preserve frozen baseline evidence and setup failure` | `verification/M07.md`; `scripts/verify_m07.py` | 충돌 기준선과 테스트 입력 동결 | 재현성과 증빙에는 중요하지만 충돌 알고리즘 자체는 P09에서 다룬다. | 중간 | 낮음 | M06, M07 |
| S | M07 | `feat(M07): preserve stale intent with versioned canonical reconciliation` | `src/itemStore.ts`의 `MutationConflict`, `prepareNext`, `rejectVersion`, `readConflicts`; `src/sync.ts` | 낙관적 버전 충돌, canonical-wins, 충돌 증거 보존 | stale write를 단순 재시도하지 않고 서버 canonical을 채택하면서 사용자의 원래 시도를 감사 가능하게 남기는 핵심 trade-off다. 상세 P09. | 높음 | 높음 | M06, M09 |
| C | M07 | `test(M07): preserve exhausted reset-parser failure` | `verification/M07.md` | 검증 도구 실패 보존 | 제품 동작보다 검증 스크립트 복구 기록에 가깝다. | 낮음 | 낮음 | — |
| B | M08 | `test(M08): preserve actual Activity recreation baseline` | `android/app/src/androidTest/java/com/mse/reactnative/M08LifecycleTest.java`; `M08LifecycleTest.draftSurvivesOneBackgroundCycleAndOneActivityRecreation` | Activity 재생성과 프로세스 종료의 차이 검증 | Android lifecycle을 실제로 구분하는 좋은 질문 소재지만 해법은 P11로 통합한다. | 높음 | 낮음 | M08, M10 |
| A | M08 | `feat(M08): retain editor state across Activity remounts` | `src/App.tsx`의 `createEditorMemory`; mounted/disposal guard; `__tests__/App.test.tsx` | 상태 소유권, remount 복원, stale callback 차단 | Activity 재생성에는 유지하되 process death에는 내구성을 약속하지 않는 명시적 경계와 비동기 완료 경쟁을 설명하기 좋다. 상세 P11. | 높음 | 중간 | M10, M14, M15 |
| B | M09 | 기록에서 정확한 메시지 미확인 — process-death 재현 구간 | `scripts/verify_m09.py`; `fixture/server.cjs`; `verification/M09-inputs.json` | 프로세스 종료 경계와 cold-start 관찰 | 실제 프로세스 소멸·재시작을 검증하는 방식은 중요하지만 독립 구현보다 P10 설명·검증에 통합한다. | 높음 | 낮음 | M05, M06, M09 |
| A | M09 | `feat(m09): resume durable uploads on foreground startup` | `src/App.tsx`; `src/itemStore.ts`; `src/sync.ts` | 시작 시 복구 정책, terminal intent 필터링, 중복 실행 방지 | cold start에서 "무엇을 자동으로 재개하고 무엇은 건드리지 않을지"를 정하는 정책 문제다. 상세 P10. | 높음 | 중간 | M05, M06, M10 |
| B | M10 | 기록에서 정확한 메시지 미확인 — unchanged-M09 prerequisite baseline | `scripts/verify_m10_baseline.py`; `verification/M10-inputs.json` | OS 작업이 없으면 process loss 뒤 실행될 수 없다는 전제 검증 | 백그라운드 실행의 필요성을 명확히 하지만 구현은 P12에 통합한다. | 중간 | 낮음 | M09, M10 |
| S | M10 | 기록에서 정확한 메시지 미확인 — persistent background sync 구현 구간 | `src/backgroundSync.ts`의 `BackgroundState`, `backgroundBridge`, `schedulePending`, `serializeSync`, `runBackgroundTask`; `BackgroundSync`, `BackgroundWorker`, `BackgroundSyncModule`; `index.js`의 `ItemBackgroundSync` 등록 | WorkManager 제약, 내구성 retry allowance, foreground/background 직렬화 | JS runtime과 OS Worker가 같은 큐를 만지는 상황에서 네트워크 제약, 최대 시도, 취소, 예약, 결과 보고를 일관되게 만드는 대표 문제다. 상세 P12. | 높음 | 높음 | M05, M09, M14 |
| B | M10 | 기록에서 정확한 메시지 미확인 — 실제 OS dispatch 검증 구간 | `scripts/verify_m10.py`; `android/app/src/test/java/com/mse/reactnative/BackgroundCycleTest.kt`; `verification/M10.md` | real WorkDatabase/job 관찰, retry 시각, cleanup | 구현 정답보다 "가짜 Worker 직접 호출 없이 무엇을 증명했는가"를 설명할 가치가 높다. | 높음 | 낮음 | M10, M14 |
| A | M14 | `feat: cancel foreground uploads on network loss` | `src/backgroundSync.ts`의 `observeForegroundSync`, `serializeSync`; `src/sync.ts`; `src/App.tsx`; `__tests__/App.test.tsx` | AbortSignal 소유권, backpressure, observer cleanup | 느린 요청과 네트워크 손실에서 늦은 응답이 ACK·충돌·스냅샷 커밋을 시작하지 못하게 하는 lifecycle/동시성 문제다. 상세 P13. | 높음 | 높음 | M08, M10 |
| B | M14 | `feat: cancel foreground uploads on network loss`의 압력·소켓 검증 구간 | `fixture/server.cjs`의 pressure 상태; `scripts/verify_m14.py` | 동시 요청 수 계측, idle connection eviction의 한계 | 측정과 네이티브 자원 경계 설명에는 유용하지만 P13에서 trade-off로 다룬다. | 중간 | 낮음 | M10, M14 |
| A | M15 | `feat(react-native): bound M15 release list loading` | `src/itemStore.ts`의 `ITEM_PAGE_SIZE`, `ItemPage`, `readPage`; `src/App.tsx`의 `FlatList`, `pageIndex`, `pageRequest`, `reloadPage`; `android/app/build.gradle` | bounded paging, stale page suppression, release 경로 검증 | 대규모 로컬 상태를 전부 JS 객체로 만들지 않고, 페이지 경계와 lifecycle을 함께 지키는 성능·정합성 문제다. 상세 P14. | 높음 | 높음 | M08, M10, M14 |
| B | M15 | `feat(react-native): bound M15 release list loading`의 native row 계측·release 검증 구간 | Android SQLite bridge 계측의 `MSESqlRows`; `verification/phase-1/M15.md`; `android/app/src/androidTest/assets/m15-inputs.json` | cursor/row materialization, non-debug release 확인 | 구현보다 성능 주장에 필요한 관찰 지점과 측정 오염을 설명하는 항목으로 적절하다. | 높음 | 낮음 | M15 |

## 대표 면접 포인트와 상세 문서

| ID | 우선순위 | 대표 Thread | 면접 포인트 | 상세 워크북 | 통합한 연관 기록 |
|---|---|---|---|---|---|
| P01 | A | M01 | reducer와 도메인 불변식 | [01-state-and-local-transactions.md#p01](01-state-and-local-transactions.md#p01) | M02의 저장소 전환 전후 상태 모델 |
| P02 | S | M02 | Item 변경·ID allocator·UI 공개의 원자성 | [01-state-and-local-transactions.md#p02](01-state-and-local-transactions.md#p02) | M04 snapshot/metadata, M05 enqueue, M06 ACK 트랜잭션의 기초 |
| P03 | A | M02 | 비파괴 스키마 마이그레이션 | [01-state-and-local-transactions.md#p03](01-state-and-local-transactions.md#p03) | M04 v1→v2, M05 queue, M06 identity, M07 conflict 스키마 |
| P04 | A | M03 | 기준 스냅샷 기반 명시적 동기화 | [02-sync-queue-and-idempotency.md#p04](02-sync-queue-and-idempotency.md#p04) | M04 refresh, M05 queue 도입 전 기준선 |
| P05 | S | M05 | transactional outbox와 FIFO drain | [02-sync-queue-and-idempotency.md#p05](02-sync-queue-and-idempotency.md#p05) | M02 로컬 트랜잭션, M06 replay, M09 startup, M10 background |
| P06 | S | M06 | 멱등성 envelope와 canonical payload hash | [02-sync-queue-and-idempotency.md#p06](02-sync-queue-and-idempotency.md#p06) | M07 version conflict, M10 background 재전송 |
| P07 | S | M06 | ACK·receipt·dequeue의 원자적 상태 전이 | [02-sync-queue-and-idempotency.md#p07](02-sync-queue-and-idempotency.md#p07) | M07 successor 보존, M10 background ACK promotion |
| P08 | A | M04 | 오프라인 우선 새로고침 상태와 성공 메타데이터 | [03-offline-conflict-and-recovery.md#p08](03-offline-conflict-and-recovery.md#p08) | M03 snapshot, M09 cold-start 표시 정책 |
| P09 | S | M07 | 낙관적 버전 충돌과 canonical reconciliation | [03-offline-conflict-and-recovery.md#p09](03-offline-conflict-and-recovery.md#p09) | M06 terminal identity, M09 재개 정책 |
| P10 | A | M09 | process death 뒤 foreground startup 복구 정책 | [03-offline-conflict-and-recovery.md#p10](03-offline-conflict-and-recovery.md#p10) | M05 durable queue, M06 terminal intent, M10 OS background 전 단계 |
| P11 | A | M08 | Activity remount 상태 소유권과 stale callback | [04-lifecycle-background-and-backpressure.md#p11](04-lifecycle-background-and-backpressure.md#p11) | M10 event listener cleanup, M14 dispose, M15 page race |
| P12 | S | M10 | WorkManager cycle, retry allowance, 직렬 실행 | [04-lifecycle-background-and-backpressure.md#p12](04-lifecycle-background-and-backpressure.md#p12) | M05 queue, M09 startup ownership, M14 cancellation |
| P13 | A | M14 | 네트워크 손실 취소와 backpressure | [04-lifecycle-background-and-backpressure.md#p13](04-lifecycle-background-and-backpressure.md#p13) | M08 lifecycle guard, M10 shared serializer |
| P14 | A | M15 | bounded pagination, latest-only publish, release 자원 검증 | [05-pagination-and-release-validation.md#p14](05-pagination-and-release-validation.md#p14) | M08 off-page draft, M10/M14 비동기 직렬화·취소 |

## 대표 Thread와 연관 Thread 관계

| 대표 축 | 대표 포인트 | 연관 Thread를 별도 문제로 늘리지 않은 이유 |
|---|---|---|
| 순수 상태 → 트랜잭션 상태 | P01, P02 | M01의 reducer 불변식이 M02 이후 저장소 경계에서도 유지되므로, UI CRUD를 Thread별로 반복 출제하지 않는다. |
| 스키마 진화 | P03 | M04~M07의 각 테이블 추가는 모두 "기존 데이터 보존 + 실패 시 이전 버전 유지"라는 같은 역량을 반복한다. |
| 동기화 기준선과 캐시 | P04, P08 | M03의 snapshot 동기화와 M04의 offline refresh는 요청 흐름은 다르지만 동일한 source-of-truth 경계를 공유한다. |
| 내구성 큐와 재전송 | P05, P06, P07 | M05의 outbox, M06의 멱등성·ACK는 한 시스템의 서로 다른 불변식이므로 세 문제로만 나누고 이후 Thread의 반복은 통합한다. |
| 충돌과 복구 정책 | P09, P10 | M07은 데이터 충돌 규칙, M09는 시작 시 실행 정책이다. 동일한 "무엇을 자동 재시도하지 않는가"를 꼬리 질문으로 연결한다. |
| lifecycle·OS 작업·취소 | P11, P12, P13 | M08의 UI 소유권, M10의 OS Worker, M14의 Abort/observer는 실행 주체가 다르므로 구분하되 cleanup·stale callback 역량은 교차 검증한다. |
| 대규모 로컬 상태 | P14 | M15의 핵심은 페이지 쿼리 하나가 아니라 DB snapshot, UI publish race, editor 소유권, release 계측을 함께 지키는 것이다. |

## S/A 완전성 대조

아래 표의 각 행은 위 선별표의 S/A 항목이 상세 문서에서 누락되지 않았음을 확인하는 기준이다.

| 선별 항목 | 상태 | 상세 위치 또는 통합 위치 |
|---|---|---|
| M01 reducer·불변식 | 독립 상세 항목 | P01 |
| M02 로컬 mutation·allocator 트랜잭션 | 독립 상세 항목 | P02 |
| M02 스키마 처리 | 독립 상세 항목 | P03 |
| M03 명시적 foreground sync | 독립 상세 항목 | P04 |
| M04 offline refresh 구현 | 독립 상세 항목 | P08 |
| M04 마이그레이션 구간 | 다른 대표 항목에 명시적 통합 | P03 |
| M05 durable sync queue | 독립 상세 항목 | P05 |
| M06 멱등성 identity/hash | 독립 상세 항목 | P06 |
| M06 atomic acknowledgement | 독립 상세 항목 | P07 |
| M07 version conflict reconciliation | 독립 상세 항목 | P09 |
| M08 editor restoration | 독립 상세 항목 | P11 |
| M09 startup recovery | 독립 상세 항목 | P10 |
| M10 persistent background sync | 독립 상세 항목 | P12 |
| M14 foreground cancellation/backpressure | 독립 상세 항목 | P13 |
| M15 bounded release list loading | 독립 상세 항목 | P14 |

## 백지 구현 우선순위

1. P05 transactional outbox와 FIFO drain
2. P07 ACK·receipt·dequeue 원자적 전이
3. P06 canonical JSON과 멱등성 envelope
4. P02 로컬 mutation·ID allocator 트랜잭션
5. P09 version conflict reconciliation
6. P12 background cycle·retry allowance·직렬화
7. P13 token 기반 네트워크 취소·cleanup
8. P14 bounded page 읽기와 stale publish 차단
9. P04 기준선 diff와 원격 snapshot 검증
10. P08 refresh 상태 머신과 성공 메타데이터
11. P11 remount editor owner와 late completion 차단
12. P10 startup recovery 결정 함수
13. P03 비파괴 마이그레이션 계획
14. P01 reducer 불변식

## 설명 연습 우선순위

1. P12: OS 스케줄러가 실행 시각을 소유할 때 앱이 보장할 수 있는 것과 없는 것
2. P09: canonical-wins를 택한 이유, 사용자 시도 보존, 자동 재시도 중단 조건
3. P06·P07: 멱등성 키와 ACK 원자성이 각각 막는 실패가 왜 다른가
4. P05: 로컬 변경과 outbox를 한 트랜잭션으로 묶는 이유
5. P13: Abort가 네트워크 요청 취소와 상태 커밋 금지를 모두 의미하려면 필요한 guard
6. P10: cold start 자동 복구 범위와 terminal intent 제외 이유
7. P08: `fresh`가 지속적인 원격 최신성을 의미하지 않는 이유
8. P14: 페이지 크기·OFFSET·FlatList·release 계측의 trade-off
9. P02·P03: COMMIT 뒤 UI 공개와 마이그레이션 실패 보존
10. P11: Activity recreation, component remount, JS runtime/process death의 차이
11. P04: 서비스 응답 대신 커밋된 저장소 읽기를 공개하는 이유
12. P01: 안정적인 ID와 reducer 불변식

## 한 문제로 통합한 Thread 묶음

1. **M02 + M04 + M05 + M06**: 로컬 트랜잭션과 스키마 진화 — P02, P03
2. **M03 + M04**: 원격 snapshot, 캐시, refresh 상태 — P04, P08
3. **M05 + M06 + M09 + M10**: 내구성 큐, replay, startup/OS 복구 — P05, P06, P07, P10, P12
4. **M06 + M07**: mutation identity와 version conflict — P06, P07, P09
5. **M08 + M10 + M14**: lifecycle 소유권, listener cleanup, cancellation — P11, P12, P13
6. **M08 + M15**: editor 상태와 페이지 이동·비동기 결과 경쟁 — P11, P14
7. **M14 + M15**: 느린 I/O에서 bounded work와 release 자원 검증 — P13, P14
