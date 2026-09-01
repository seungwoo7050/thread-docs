# 백그라운드 작업, 동시성, 성능 면접 워크북

이 문서는 앱 프로세스가 사라진 뒤에도 이어지는 제한된 자동 동기화, 겹치는 동기화 요청의 직렬화와 취소, 대용량 로컬 상태의 bounded read·가상화 경계를 다룬다.

<a id="w13"></a>
## [M10 / `feat: schedule durable sync with bounded persistent work`] W13. 내구성 있는 unique work와 프로세스 사망을 넘는 retry 예산

### 면접 질문

단순히 앱 시작 때 `recoverPending()`을 호출하는 대신 WorkManager에 하나의 persistent unique work chain을 등록하고, HTTP 시도 횟수를 Room에 예약한 이유는 무엇인가요?

꼬리 질문:

- WorkManager의 `runAttemptCount`만으로 application-level HTTP 최대 3회를 보장하기 어려운 이유는 무엇인가요?
- 실제 HTTP 전에 reservation과 dispatch 표시를 트랜잭션으로 남긴 이유는 무엇인가요?
- 같은 work를 여러 진입점에서 등록해도 하나의 chain을 유지해야 하는 이유는 무엇인가요?
- CONNECTED constraint와 10초·20초 exponential backoff가 해결하는 문제와 해결하지 못하는 문제는 무엇인가요?
- Worker가 database를 열고 `finally`에서 닫아야 하는 이유는 무엇인가요?
- 로컬 저장 커밋 뒤 scheduler 등록이 실패한 경우 저장을 rollback하지 않은 이유는 무엇인가요?

### 30초 모범 답변

M09 시작 복구는 앱이 실행돼야 동작하지만 M10은 프로세스가 죽어도 OS가 다시 실행할 수 있어야 합니다. 그래서 pending queue와 연결된 unique WorkManager 작업을 등록하고, 실제 HTTP 시도 예약과 횟수를 Room에 영구 기록했습니다. WorkManager의 실행 횟수는 Worker 재시작이나 HTTP 호출 전후 실패와 정확히 일치하지 않을 수 있어 application-level 예산을 대신할 수 없습니다. 각 요청 전에 reservation을 커밋해 프로세스 사망 뒤에도 최대 3회가 유지되고, 같은 identity를 재사용합니다. CONNECTED와 backoff는 불필요한 즉시 재시도를 줄이지만 서버 오류 자체를 해결하지는 않습니다. 저장 후 scheduling 실패는 이미 성공한 사용자 편집을 취소하지 않고 별도 복구 오류로 표시합니다.

### 답변 핵심 키워드

persistent OS work · unique chain · durable cycle ID · transactional HTTP reservation · retry ceiling across process loss · CONNECTED · exponential backoff · resource cleanup · post-commit scheduling failure

### 백지 구현

**구현 목표**

최대 3회의 application-level 네트워크 시도를 프로세스 재시작을 넘어 지키는 자동 작업 상태 머신을 구현한다. 각 run은 먼저 영구 저장소에서 시도를 예약한 뒤에만 HTTP를 호출할 수 있다.

**인터페이스 또는 함수 시그니처**

```kotlin
enum class CycleState { ACTIVE, COMPLETE, DEFERRED }

data class AutomaticCycle(
    val cycleId: String,
    val httpAttempts: Int,
    val state: CycleState,
)

enum class WorkerDecision { SUCCESS, RETRY, FAILURE }

interface AutomaticSyncStore {
    suspend fun ensureCycle(): AutomaticCycle?
    suspend fun reserveHttpAttempt(cycleId: String, maxAttempts: Int): AutomaticCycle?
    suspend fun markComplete(cycleId: String)
    suspend fun markDeferred(cycleId: String)
    suspend fun hasRetryablePending(): Boolean
}

class AutomaticSyncRunner(
    private val store: AutomaticSyncStore,
    private val sendOne: suspend () -> SendResult,
) {
    suspend fun runOnce(): WorkerDecision = TODO("직접 구현")
}

sealed interface SendResult {
    data object Applied : SendResult
    data object RetryableFailure : SendResult
    data object TerminalStop : SendResult
}
```

**입력과 출력**

- 입력: durable cycle, pending 유무, 한 번의 전송 결과
- 출력: Worker의 SUCCESS/RETRY/FAILURE 결정과 갱신된 cycle 상태

**반드시 만족해야 할 조건**

- retryable pending이 없으면 새 cycle을 만들거나 HTTP를 보내지 않는다.
- 실제 HTTP 호출 전 `reserveHttpAttempt`가 성공해야 한다.
- 프로세스가 reservation 직후 죽더라도 횟수는 소비된 것으로 남는다.
- `httpAttempts`가 3이면 네 번째 HTTP를 호출하지 않는다.
- retryable failure이며 예산이 남으면 RETRY다.
- 성공하면 COMPLETE, 예산 소진 후 미완료면 DEFERRED다.
- terminal conflict는 자동으로 뒤 intent를 건너뛰지 않는다.
- 한 run에서 DB나 네트워크 자원을 열었다면 모든 경로에서 닫는다.

**경계 조건**

- pending 없음
- 첫 시도 성공
- 세 번째 시도 성공
- 세 번 모두 retryable failure
- reservation 뒤 프로세스 사망을 모사한 재실행
- terminal conflict
- cycle 저장 실패
- cancellation

**실패 조건과 제약**

- WorkManager API 자체는 구현하지 않는다.
- backoff 계산은 `attempt`를 받아 지연을 반환하는 별도 함수로 작성해도 된다.
- HTTP 요청은 한 번에 하나의 immutable envelope를 사용한다고 가정한다.
- 25~30분 범위다.

### 구현 후 자가 검증

- [ ] reservation 없이 HTTP가 호출되는 경로가 없는가?
- [ ] 재실행 후에도 누적 시도가 3을 넘지 않는가?
- [ ] 503,503,성공에서 호출 3회와 COMPLETE가 되는가?
- [ ] 503 세 번 뒤 DEFERRED이며 네 번째 호출이 없는가?
- [ ] 같은 pending identity가 모든 retry에서 유지되는가?
- [ ] terminal conflict 뒤 다음 intent가 자동 전송되지 않는가?
- [ ] cancellation과 예외에서도 자원이 정리되는가?
- [ ] 로컬 저장 성공과 scheduler 등록 실패를 별도 결과로 다룰 수 있는가?

### 구현 후 설명할 것

1. OS scheduler 실행 횟수와 실제 HTTP 예산을 분리한 이유
2. request 전 reservation이 제공하는 crash consistency와 보수적 과금
3. unique work가 중복 등록과 병렬 chain을 막는 방식
4. retry ceiling·backoff·CONNECTED constraint의 역할 분담
5. post-commit scheduling failure를 저장 실패로 되돌리지 않은 사용자 데이터 invariant

### 원본 확인 위치

- Thread: M10
- 커밋: `feat: schedule durable sync with bounded persistent work`
- 파일·컴포넌트: `ItemSyncWorker`, `ItemWorkScheduler`, `AutomaticItemSync`, `AutomaticResult`, `ItemApplication`, `AutomaticCycle`, unique work 이름 `item-automatic-sync`
- 관련 Thread: M09의 시작 복구, M06의 stable mutation identity, M07의 terminal conflict, M14의 압력·취소 검증과 연결된다.

---

<a id="w14"></a>
## [M10 / `feat: schedule durable sync with bounded persistent work`; M14 / `test: verify M14 pressure cancellation on unchanged product`] W14. single-flight 동기화, 역압력, 취소 경계

### 면접 질문

동일 프로세스에서 UI, startup recovery, Worker 등 여러 진입점이 동시에 `synchronize()`를 호출할 수 있을 때 `itemSyncMutex.withLock`으로 전체 drain을 직렬화한 이유는 무엇인가요?

꼬리 질문:

- 요청 하나마다만 잠그고 drain 전체를 잠그지 않으면 어떤 queue/ACK race가 생길 수 있나요?
- M14에서 느린 응답 중 추가 sync 호출이 들어와도 mutation 동시 실행 최고치가 1이었던 결과가 무엇을 증명하나요?
- ACK 3개가 내구성 있게 커밋된 뒤 취소되면 남은 9개가 그대로 pending이어야 하는 이유는 무엇인가요?
- 코루틴 취소 시 lock과 cursor·observer 같은 자원을 어떻게 정리해야 하나요?
- 프로세스 내부 Mutex가 여러 프로세스의 동시 실행까지 막아 주지 않는다는 한계를 어떻게 보완할 수 있나요?

### 30초 모범 답변

여러 sync 진입점이 같은 FIFO queue를 동시에 drain하면 같은 envelope를 중복 전송하거나 ACK와 dequeue 순서가 뒤섞일 수 있습니다. 그래서 프로세스 안에서는 drain 전체를 single-flight로 직렬화했습니다. M14는 500ms 지연 중 겹친 호출이 있어도 실제 mutation 동시성은 1이고, 취소 시 이미 커밋된 ACK 3개만 제거되며 나머지 9개가 내구성 있게 남는 것을 확인했습니다. 취소는 삼키지 않고 재전파하되 `finally`/구조적 동시성으로 lock, cursor, observer를 해제해야 합니다. 다만 Mutex는 프로세스 범위이므로 cross-process 정합성은 DB transaction·reservation 같은 영구 상태가 담당해야 합니다.

### 답변 핵심 키워드

single-flight · drain-level mutex · backpressure · durable ACK barrier · cooperative cancellation · cleanup · no lost remainder · process-local limit

### 백지 구현

**구현 목표**

동시에 여러 호출자가 들어와도 실제 drain 함수가 한 번에 하나만 실행되도록 하는 취소 가능한 single-flight gate를 설계한다. 구현 언어는 Kotlin이며 실제 네트워크는 fake 함수로 대체한다.

**인터페이스 또는 함수 시그니처**

```kotlin
interface SyncGate {
    suspend fun <T> exclusive(block: suspend () -> T): T
}

class SyncCoordinator(
    private val gate: SyncGate,
    private val readPending: suspend () -> List<PendingEnvelope>,
    private val sendAndAcknowledge: suspend (PendingEnvelope) -> Unit,
) {
    suspend fun synchronize() {
        TODO("직접 구현")
    }
}
```

**입력과 출력**

- 입력: 동시에 호출될 수 있는 `synchronize()`, 순서 있는 pending 목록, 취소 가능한 전송 함수
- 출력: 성공한 entry만 제거된 queue; 함수 성공, 실패 또는 취소

**반드시 만족해야 할 조건**

- 같은 프로세스에서 `sendAndAcknowledge`가 동시에 두 개 실행되지 않는다.
- drain 순서는 pending sequence를 따른다.
- 각 ACK가 커밋된 뒤에만 다음 entry로 진행한다.
- 중간 취소 시 이미 ACK된 entry는 되돌리지 않고, 아직 시작하지 않은 entry는 남긴다.
- 취소 예외를 일반 sync 오류로 바꾸지 않는다.
- lock은 성공·실패·취소에서 모두 해제된다.
- 한 호출이 기다리는 동안 queue가 바뀔 수 있다는 점을 고려해 lock 안에서 읽을 시점을 정한다.

**경계 조건**

- 빈 큐
- 동시에 5개 호출
- 첫 entry에서 실패
- 세 번째 ACK 직후 취소
- 기다리던 호출 자체가 취소됨
- terminal entry가 큐 앞에 있음

**실패 조건과 제약**

- `Mutex` 구현 자체를 다시 만들 필요는 없다. `SyncGate` 구현에 표준 Mutex를 써도 된다.
- process 간 동기화는 범위 밖이지만 한계를 설명해야 한다.
- timeout이나 임의 delay로 동시성 문제를 숨기지 않는다.

### 구현 후 자가 검증

- [ ] 동시 호출에서 active send 최대값이 1인가?
- [ ] 동일 envelope가 병렬로 두 번 전송되지 않는가?
- [ ] 취소 전에 커밋된 ACK는 유지되는가?
- [ ] 취소 이후 남은 queue가 누락 없이 보존되는가?
- [ ] 기다리는 호출이 lock 획득 전에 취소될 수 있는가?
- [ ] 모든 예외 경로에서 lock과 테스트 observer가 정리되는가?
- [ ] 이 gate가 cross-process 중복을 막지 못한다는 점을 설명할 수 있는가?

### 구현 후 설명할 것

1. 요청 단위가 아니라 queue drain 단위로 잠근 이유
2. single-flight가 처리량을 제한하지만 정합성을 단순화하는 trade-off
3. durable ACK가 취소 경계의 commit point가 되는 방식
4. 구조적 취소와 자원 정리 원칙
5. process-local Mutex와 DB 기반 reservation의 책임 분리

### 원본 확인 위치

- 대표 Thread: M10
- 대표 커밋: `feat: schedule durable sync with bounded persistent work`
- 연관 Thread: M14
- 연관 커밋: `test: verify M14 pressure cancellation on unchanged product`
- 파일·컴포넌트: `ItemSync.synchronize()`, `itemSyncMutex`, `AutomaticItemSync`, `M14-inputs.json`
- 통합 이유: M14는 제품 코드를 바꾸지 않고 M10의 직렬화·취소·내구성 queue invariant를 느린 네트워크 압력에서 검증한다.

---

<a id="w15"></a>
## [M15 / `feat: bound M15 release reads and virtualize item pages`] W15. bounded database page와 UI 가상화

### 면접 질문

2000개 Item을 시작할 때 두 번 전부 읽던 구현을 `count + LIMIT/OFFSET` 25개 페이지와 `LazyColumn`으로 바꾼 이유는 무엇인가요? 데이터베이스 paging과 UI virtualization은 각각 어떤 병목을 해결하나요?

꼬리 질문:

- count와 page rows를 한 트랜잭션에서 읽은 이유는 무엇인가요?
- page offset을 읽기 성공 뒤에만 공개한 invariant는 무엇인가요?
- 마지막 페이지에서 마지막 행을 삭제하거나 저장소가 비었을 때 offset을 어떻게 clamp해야 하나요?
- `ORDER BY rowid`를 고정한 이유와 안정적인 비즈니스 정렬 키를 도입할 때의 차이는 무엇인가요?
- OFFSET paging이 큰 offset에서 느려질 수 있는 이유와 keyset paging으로 바꾸는 기준은 무엇인가요?
- 실제 release에서 관찰한 first-usable 시간을 SLA나 benchmark 결과로 주장하면 안 되는 이유는 무엇인가요?

### 30초 모범 답변

전체 읽기는 DB materialization, 객체 생성, UI composition 비용을 모두 데이터 크기에 비례해 시작 경로에 몰아넣습니다. M15는 Room에서 count와 25행만 읽어 I/O와 메모리를 제한하고, `LazyColumn`으로 화면에 필요한 행만 compose해 UI 비용도 제한했습니다. count와 rows는 한 snapshot에서 읽고, 새 offset은 읽기가 성공한 뒤에만 공개해 실패 시 현재 페이지를 잃지 않습니다. 삭제로 범위를 벗어나면 마지막 유효 페이지나 0으로 clamp합니다. OFFSET은 구현이 단순하지만 깊은 페이지에서 비용이 커질 수 있어 데이터가 더 커지거나 순차 탐색이 중심이면 stable key 기반 keyset paging을 고려합니다.

### 답변 핵심 키워드

bounded read · count + LIMIT/OFFSET · transactional snapshot · publish-after-success · offset clamp · stable ordering · LazyColumn virtualization · OFFSET vs keyset · release validation ≠ benchmark SLA

### 백지 구현

**구현 목표**

전체 개수와 한 페이지 rows를 읽고, 요청 offset을 유효 범위로 보정하는 page loader를 구현한다. 읽기가 실패하면 호출자가 가진 이전 offset을 바꾸지 않아야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class Page<T>(
    val totalCount: Int,
    val offset: Int,
    val limit: Int,
    val rows: List<T>,
)

interface PagingDao<T> {
    suspend fun <R> inReadTransaction(block: suspend PagingDao<T>.() -> R): R
    suspend fun countItems(): Int
    suspend fun readPageRows(limit: Int, offset: Int): List<T>
}

class PageLoader<T>(
    private val dao: PagingDao<T>,
    private val pageSize: Int = 25,
) {
    suspend fun load(requestedOffset: Int): Page<T> = TODO("직접 구현")
    fun nextOffset(current: Page<T>): Int = TODO("직접 구현")
    fun previousOffset(current: Page<T>): Int = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: 요청 offset, page size, DB total count와 rows
- 출력: 유효 offset으로 정규화된 Page

**반드시 만족해야 할 조건**

- count와 rows는 하나의 read transaction 안에서 읽는다.
- page size는 양수여야 한다.
- requestedOffset이 음수면 0으로 보정한다.
- totalCount가 0이면 offset 0, 빈 rows다.
- offset이 마지막 유효 페이지를 넘으면 마지막 페이지 시작점으로 clamp한다.
- 반환 rows 수는 pageSize 이하이며 offset 순서와 일치한다.
- loader 내부에 현재 offset을 저장한다면 DB 읽기 성공 뒤에만 변경한다.
- 동일 snapshot에서 안정적인 ordering이 전제되어야 한다.

**경계 조건**

- 0행, 1행, 정확히 25행, 26행
- 마지막 페이지가 1행
- 마지막 페이지 행 삭제 후 같은 offset 재조회
- 매우 큰 requestedOffset
- count 성공 후 rows 읽기 실패
- pageSize 0 또는 음수

**실패 조건과 제약**

- UI 코드는 구현하지 않는다. `LazyColumn`의 stable key가 왜 필요한지만 설명한다.
- OFFSET SQL 자체는 DAO가 제공한다고 가정한다.
- 시간 복잡도는 DB 구현에 따라 달라지며 깊은 OFFSET의 한계를 언급한다.
- 20~25분 범위다.

### 구현 후 자가 검증

- [ ] 0행에서 offset 0과 빈 page인가?
- [ ] 25·26행 경계에서 next/previous가 정확한가?
- [ ] 범위를 넘는 offset이 마지막 유효 페이지로 clamp되는가?
- [ ] 마지막 행 삭제 뒤 빈 잘못된 페이지에 머물지 않는가?
- [ ] rows read 실패 시 이전 UI offset을 유지할 수 있는 API인가?
- [ ] 한 페이지 읽기에서 전체 Item을 materialize하지 않는가?
- [ ] 안정적인 ordering과 UI key를 분리해 설명할 수 있는가?
- [ ] OFFSET에서 keyset으로 바꿀 성장 기준을 제시할 수 있는가?

### 구현 후 설명할 것

1. DB bounded read와 UI virtualization이 서로 다른 비용을 줄이는 방식
2. count+rows snapshot과 offset publish-after-success invariant
3. 삭제·빈 저장소에서 offset clamp 정책
4. OFFSET의 단순성 대 deep-page 성능 trade-off
5. release 관찰 시간을 benchmark/SLA로 과장하지 않고 기능 경계로 검증한 이유

### 원본 확인 위치

- Thread: M15
- 커밋: `feat: bound M15 release reads and virtualize item pages`
- 파일·컴포넌트: `ItemDao.countItems`, `ItemDao.readPageRows`, `ItemStore.page`, `LazyColumn`, `M15StorageTest`
- 관련 Thread: M01의 stable row identity, M02의 Room readback, M05 이후의 pending queue를 그대로 유지하면서 읽기·렌더링 경계만 제한한다.
