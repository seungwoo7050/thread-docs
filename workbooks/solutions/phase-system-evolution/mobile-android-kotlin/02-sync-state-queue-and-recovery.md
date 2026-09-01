# 동기화 상태, 내구성 큐, 복구 면접 워크북

이 문서는 foreground 동기화에서 시작해 오프라인 변경 큐와 프로세스 사망 후 복구까지 이어지는 대표 면접 지점을 묶는다. 원본 구현을 복사하지 않고, 같은 경계와 invariant를 검증할 수 있는 축소 문제만 제시한다.

<a id="w05"></a>
## [M03 / `feat(android): add M03 foreground sync candidate`] W05. 로컬 변경 계획과 canonical 상태 교체 경계

### 면접 질문

M03의 `ItemSync`가 로컬 목록과 마지막 성공 canonical snapshot을 비교해 POST/PATCH/DELETE를 순차 전송한 뒤, 마지막 GET 결과로 Room을 통째로 교체한 이유는 무엇인가요?

꼬리 질문:

- HTTP 응답 객체를 UI의 두 번째 저장소로 사용하지 않고 Room을 다시 읽은 이유는 무엇인가요?
- 중간 mutation은 원격에서 성공했지만 마지막 GET이나 로컬 저장이 실패하면 어떤 불확실성이 남나요?
- 첫 동기화에서 `version = 0`인 로컬 생성 항목을 별도로 다룬 이유는 무엇인가요?
- HTTP/1.0 fixture와 OkHttp 연결 재사용 문제를 `Connection: close`와 mutation retry 비활성화로 처리한 판단을 설명해 주세요.

### 30초 모범 답변

동기화는 "무엇을 보낼지 결정하는 단계"와 "최종 canonical 상태를 로컬에 커밋하는 단계"를 분리했습니다. 로컬과 직전 snapshot의 차이를 순차 mutation으로 보내고, 마지막에 서버 전체 상태를 받아 Room에 원자적으로 교체한 뒤 다시 읽어 UI에 제공합니다. 이렇게 하면 화면의 진실 공급원은 계속 Room 하나입니다. 다만 M03 snapshot은 메모리에만 있어 프로세스 사망이나 부분 원격 성공을 복구하지 못합니다. 또한 mutation은 자동 재시도하면 중복 적용될 수 있으므로, 닫힌 HTTP/1.0 연결 재사용 문제는 명시적으로 연결을 닫고 재시도하지 않는 쪽을 택했습니다.

### 답변 핵심 키워드

sync plan · local vs previous canonical · serial mutation · final GET · Room single source of truth · atomic replace/readback · partial success ambiguity · mutation retry 위험

### 백지 구현

**구현 목표**

이전 canonical 목록과 현재 로컬 목록을 비교해 원격에 보낼 변경 계획을 생성한다. 생성, 제목 변경, 완료 상태 변경, 삭제를 구분하고, 항상 결정적인 순서로 반환한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class SyncItem(
    val id: String,
    val title: String,
    val completed: Boolean,
    val version: Long,
)

sealed interface RemoteMutation {
    data class Create(val item: SyncItem) : RemoteMutation
    data class Rename(val id: String, val title: String) : RemoteMutation
    data class SetCompleted(val id: String, val completed: Boolean) : RemoteMutation
    data class Delete(val id: String) : RemoteMutation
}

fun planMutations(
    previousCanonical: List<SyncItem>,
    currentLocal: List<SyncItem>,
): List<RemoteMutation> = TODO("직접 구현")

interface CanonicalStore {
    suspend fun replaceAllAndReadBack(items: List<SyncItem>): List<SyncItem>
}
```

**입력과 출력**

- 입력: 마지막 성공 canonical 목록, 현재 로컬 목록
- 출력: 원격에 보낼 mutation의 결정적 순서 목록

**반드시 만족해야 할 조건**

- 로컬에만 있는 행은 Create다.
- 이전 canonical에만 있는 행은 Delete다.
- 같은 ID에서 제목과 완료 상태가 모두 바뀌면 두 변경의 순서를 명시한다.
- 단순 정렬 차이는 mutation을 만들지 않는다.
- 같은 입력은 항상 같은 계획을 만든다.
- `version = 0`인 기존 로컬 행을 첫 동기화 생성 후보로 취급할 정책을 명확히 한다.

**경계 조건**

- 두 목록 모두 비어 있음
- ID는 같고 모든 필드가 같음
- 제목과 완료 상태가 동시에 바뀜
- 생성과 삭제가 섞임
- 입력에 중복 ID가 있음

**실패 조건과 제약**

- 네트워크 호출 자체는 구현하지 않는다.
- 중복 ID는 조용히 덮어쓰지 말고 실패시켜도 된다.
- 전체 시간 복잡도를 설명한다. Map을 사용하면 일반적으로 O(n+m), 매번 선형 탐색하면 O(nm)이다.
- 구현 시간은 20~25분이다.

### 구현 후 자가 검증

- [ ] 순서만 다른 동일 목록에서 계획이 비어 있는가?
- [ ] 생성·수정·삭제가 정확한 ID에 매핑되는가?
- [ ] 동일 입력에서 계획 순서가 안정적인가?
- [ ] 중복 ID를 탐지하는가?
- [ ] 두 필드가 바뀐 경우 누락 없이 둘 다 반영되는가?
- [ ] 입력 목록을 변경하지 않는가?
- [ ] 원격 성공 뒤 로컬 canonical 교체가 실패할 수 있는 경계를 설명할 수 있는가?

### 구현 후 설명할 것

1. diff 계획과 실행을 분리한 이유
2. 최종 GET과 Room readback을 통해 UI 저장소를 하나로 유지한 이유
3. 직렬 전송이 단순성을 주는 대신 처리량을 제한하는 trade-off
4. 메모리 snapshot 방식이 프로세스 사망과 부분 성공에 취약한 이유
5. mutation 자동 재시도를 멱등성 없이 켜면 안 되는 이유

### 원본 확인 위치

- Thread: M03
- 커밋: `feat(android): add M03 foreground sync candidate`
- 파일·컴포넌트: `ItemSync`, `HttpItemRemote`, `ItemRemote`, `ItemDatabaseTest#isolatedM02DatabasesCannotShareChangesWithoutSync`, `canonicalReplacementRollsBackIfAnyRowCannotBeInserted`
- 관련 Thread: M03의 HTTP/1.0 연결 수리는 현재 기록에서 커밋 제목이 노출되지 않았으며 `HttpItemRemote`, OkHttp, `fixture/server.py`, `network_security_config.xml`에 연결된다. M05는 메모리 diff를 내구성 mutation queue로 대체한다.

---

<a id="w06"></a>
## [M04 / `feat(android): add M04 refresh status and saved success time`] W06. 세션 상태와 내구성 metadata를 분리한 동기화 상태 머신

### 면접 질문

`SyncState`의 STALE, REFRESHING, FRESH, ERROR와 Room의 `lastSuccessfulRefreshAt`을 왜 같은 방식으로 저장하지 않았나요? 각각 어떤 수명을 갖나요?

꼬리 질문:

- 실패한 refresh가 기존 Item과 이전 성공 시각을 덮어쓰면 안 되는 이유는 무엇인가요?
- canonical rows, 성공 시각, readback을 한 트랜잭션에 넣은 invariant는 무엇인가요?
- 새 `ItemSync` 인스턴스가 저장된 성공 시각을 읽고도 STALE로 시작하는 이유는 무엇인가요?
- `CancellationException`을 일반 오류와 같이 ERROR 상태로 삼키지 않고 다시 던져야 하는 이유는 무엇인가요?

### 30초 모범 답변

STALE·REFRESHING·ERROR는 현재 세션의 진행과 실패를 설명하는 일시 상태이고, `lastSuccessfulRefreshAt`은 마지막으로 canonical 데이터가 실제 커밋된 사실이라 프로세스를 넘어 보존할 metadata입니다. 실패한 요청은 새 canonical 결과가 아니므로 기존 Item과 성공 시각을 건드리지 않습니다. 성공 때는 rows, 성공 시각, readback을 한 트랜잭션으로 묶어 서로 다른 시점의 조합이 보이지 않게 합니다. 새 세션은 저장 시각을 읽어도 원격이 그 뒤 바뀌었을 수 있으므로 STALE입니다. 취소는 구조적 제어 흐름이어서 오류 UI로 변환하지 않고 상태를 정리한 뒤 재전파해야 합니다.

### 답변 핵심 키워드

session state vs durable fact · STALE/REFRESHING/FRESH/ERROR · last successful refresh · transactional metadata · cached data retention · cancellation transparency

### 백지 구현

**구현 목표**

명시적 refresh의 상태 전이를 관리하는 작은 상태 머신을 구현한다. 성공한 canonical commit만 durable 성공 시각을 갱신하고, 실패와 취소는 캐시를 보존해야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
enum class RefreshState { STALE, REFRESHING, FRESH, ERROR }

data class RefreshStatus(
    val state: RefreshState,
    val lastSuccessfulRefreshAt: Long?,
    val error: String?,
)

interface RefreshStore {
    suspend fun readLastSuccessfulRefreshAt(): Long?
    suspend fun commitCanonical(
        rows: List<Item>,
        successfulAt: Long,
    ): List<Item>
}

class RefreshCoordinator(
    private val store: RefreshStore,
    private val fetchRemote: suspend () -> List<Item>,
    private val now: () -> Long,
) {
    var status: RefreshStatus = TODO("초기 상태 직접 정의")
        private set

    suspend fun loadSavedStatus(): List<Item> = TODO("직접 구현")
    fun markLocalChange() = TODO("직접 구현")
    suspend fun refresh(): List<Item> = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: 저장된 성공 시각, 원격 fetch 결과, 현재 시각
- 출력: 저장된/새 canonical rows와 갱신된 `RefreshStatus`

**반드시 만족해야 할 조건**

- 저장 상태를 읽은 새 인스턴스는 성공 시각을 보존하지만 STALE로 시작한다.
- refresh 시작 시 REFRESHING이다.
- fetch와 commit이 모두 성공한 뒤에만 FRESH와 새 성공 시각을 공개한다.
- 일반 실패는 기존 성공 시각을 유지한 ERROR가 된다.
- 취소는 캐시와 이전 성공 시각을 유지하고 반드시 호출자에게 다시 전달한다.
- 로컬 변경은 FRESH 상태를 STALE로 바꾼다.

**경계 조건**

- 저장된 성공 시각이 없음
- 원격 빈 목록 성공
- fetch 실패
- fetch 성공 후 commit 실패
- refresh 중 취소
- ERROR 이후 재시도 성공

**실패 조건과 제약**

- 실제 StateFlow는 구현하지 않아도 된다.
- 캐시 데이터 자체는 coordinator가 복제해 들고 있지 않아도 된다.
- `commitCanonical`의 내부 원자성은 인터페이스 계약으로 가정하되 무엇이 함께 커밋되어야 하는지 설명한다.

### 구현 후 자가 검증

- [ ] 실패 전후 Item과 이전 성공 시각이 유지되는가?
- [ ] FRESH가 commit 완료 전에 보이지 않는가?
- [ ] 새 세션이 저장 시각을 읽고 STALE인가?
- [ ] local change가 상태를 STALE로 만드는가?
- [ ] 취소가 ERROR 문자열로 바뀌지 않고 재전파되는가?
- [ ] ERROR 뒤 성공하면 오류가 지워지는가?
- [ ] rows와 metadata의 원자적 커밋 invariant를 테스트할 수 있는가?

### 구현 후 설명할 것

1. "최근 성공 사실"과 "현재 세션의 freshness 판단"을 분리한 이유
2. 실패가 캐시를 무효화하지 않는 offline-first 읽기 원칙
3. rows와 timestamp를 한 트랜잭션으로 묶은 이유
4. cancellation을 정상적인 제어 신호로 보존하는 이유
5. FRESH가 영구적인 원격 최신성을 보장하지 않는다는 한계

### 원본 확인 위치

- Thread: M04
- 커밋: `feat(android): add M04 refresh status and saved success time`
- 파일·컴포넌트: `SyncState`, `SyncStatus`, `readSavedStatus()`, `markLocalChange()`, `synchronize()`, `sync_metadata`, `lastSuccessfulRefreshAt`
- 관련 Thread: M02의 Room 단일 진실 공급원, M03의 canonical 교체, M05의 내구성 변경 큐와 연결된다.

---

<a id="w07"></a>
## [M05 / `feat(android): persist ordered offline mutation intents`] W07. 도메인 변경과 outbox intent의 원자적 커밋

### 면접 질문

오프라인 CRUD에서 현재 Item 상태만 저장하고 나중에 서버 변경을 재구성하지 않고, 각 작업의 payload를 `pending_mutations`에 같은 트랜잭션으로 저장한 이유는 무엇인가요?

꼬리 질문:

- rename 뒤 complete가 이어진 경우 현재 Item 하나만 보고 원래 두 intent를 정확히 복원할 수 있나요?
- 삭제할 때 Item을 지우더라도 DELETE intent를 남겨야 하는 이유는 무엇인가요?
- 큐를 FIFO로 전송하고 각 성공 응답 후 해당 entry만 제거하는 장점과 단점은 무엇인가요?
- schema 2→3 migration에서 기존 `version = 0` 생성 항목만 큐에 넣을 수 있었던 이유와, 잃어버린 과거 intent는 왜 복원할 수 없었나요?

### 30초 모범 답변

현재 Item은 최종 로컬 상태만 나타내므로 여러 사용자 의도와 순서를 잃습니다. 예를 들어 rename 후 complete를 한 행만 보고는 원래 두 요청의 순서와 payload를 복원할 수 없습니다. 그래서 각 CRUD와 해당 outbox intent를 한 Room 트랜잭션으로 커밋해 "보이는 로컬 변경에는 반드시 전송할 intent가 있다"는 invariant를 만들었습니다. 큐는 순서대로 보내고 성공 응답 뒤에만 하나씩 제거합니다. 삭제도 로컬 행과 별개로 DELETE intent를 유지합니다. 이 방식은 내구성과 추적성이 좋지만 큐가 길어지고, M05만으로는 원격 성공 후 로컬 ACK 전에 죽는 중복 전송 창을 닫지 못합니다.

### 답변 핵심 키워드

transactional outbox · intent at mutation time · FIFO · item+intent atomicity · delete tomb intent · no reconstruction · ACK-after-success · crash window

### 백지 구현

**구현 목표**

간단한 Item 저장소와 FIFO outbox를 구현한다. 각 로컬 변경과 그에 대응하는 immutable intent는 반드시 같은 트랜잭션 결과로 생성되어야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
enum class Operation { CREATE, RENAME, COMPLETE, DELETE }

data class PendingMutation(
    val sequence: Long,
    val itemId: String,
    val operation: Operation,
    val title: String?,
    val completed: Boolean?,
)

interface TransactionalItemDao {
    suspend fun <T> inTransaction(block: suspend TransactionalItemDao.() -> T): T
    suspend fun readItem(id: String): Item?
    suspend fun upsertItem(item: Item)
    suspend fun deleteItem(id: String)
    suspend fun insertPending(mutation: PendingMutation)
    suspend fun readPendingInOrder(): List<PendingMutation>
    suspend fun deletePending(sequence: Long)
}

class OfflineItemStore(
    private val dao: TransactionalItemDao,
    private val nextSequence: suspend () -> Long,
) {
    suspend fun rename(id: String, title: String) = TODO("직접 구현")
    suspend fun setCompleted(id: String, completed: Boolean) = TODO("직접 구현")
    suspend fun delete(id: String) = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: Item ID와 변경 payload
- 출력: 커밋된 Item 상태와 순서가 부여된 pending mutation

**반드시 만족해야 할 조건**

- Item 변경과 pending insert는 같은 트랜잭션이다.
- 둘 중 하나라도 실패하면 둘 다 반영되지 않는다.
- intent payload는 변경 당시 사용자 작업을 표현해야 하며 이후 Item에서 재구성하지 않는다.
- sequence는 큐 전체에서 증가하고 FIFO 순서를 정의한다.
- DELETE는 Item을 지운 뒤에도 pending entry를 유지한다.
- drain 측에서는 성공한 원격 응답 뒤에만 해당 sequence를 제거한다.

**경계 조건**

- 존재하지 않는 Item 수정/삭제
- 같은 Item에 연속 rename, complete, delete
- intent insert 실패
- Item write 실패
- 큐 중간 entry 전송 실패

**실패 조건과 제약**

- coalescing은 구현하지 않는다.
- 원격 멱등 키는 아직 요구하지 않는다. 그 한계를 설명한다.
- 20~30분 안에 rename과 delete를 중심으로 구현해도 된다.

### 구현 후 자가 검증

- [ ] Item write가 성공하고 intent insert가 실패했을 때 둘 다 rollback되는가?
- [ ] intent insert가 성공하고 Item write가 실패해도 orphan intent가 남지 않는가?
- [ ] 연속 작업의 sequence와 payload가 정확한가?
- [ ] 삭제 뒤 DELETE intent가 남아 있는가?
- [ ] 전송 실패 때 현재 entry와 뒤 entry가 유지되는가?
- [ ] 성공한 entry만 제거되고 순서가 바뀌지 않는가?
- [ ] 현재 Item만으로 과거 intent를 복원할 수 없는 사례를 설명할 수 있는가?

### 구현 후 설명할 것

1. transactional outbox가 해결하는 "로컬 상태는 바뀌었지만 보낼 일이 없음" 문제
2. 최종 상태가 아니라 변경 시점 intent를 저장한 이유
3. FIFO의 단순성·일관성과 처리량·큐 팽창 trade-off
4. migration이 확실히 알 수 있는 version-zero 생성만 보강한 이유
5. M05에서 남은 remote-success/local-ACK crash window

### 원본 확인 위치

- Thread: M05
- 커밋: `feat(android): persist ordered offline mutation intents`
- 파일·컴포넌트: `ItemStore`, `ItemDao`, `pending_mutations`, `PendingMutation`, `M05ScenarioInstrumentation`
- 관련 Thread: M05 기준선은 `verification/offline_queue_restart.py`로 이전 구현의 intent 유실을 재현했다. M06은 mutation identity와 ACK receipt로 중복 전송 창을 보완한다.

---

<a id="w08"></a>
## [M09 / 현재 기록에서 커밋 제목 미노출(일반 시작 복구 구현)] W08. 프로세스 사망 뒤 local-first 시작과 보수적 재전송

### 면접 질문

일반 앱 시작에서 먼저 저장된 Item과 상태를 읽고, 저장 오류가 없으며 nonterminal pending mutation이 있을 때만 `recoverPending()`을 실행한 이유는 무엇인가요?

꼬리 질문:

- 앱이 시작되자마자 항상 GET 또는 Sync를 실행하지 않은 이유는 무엇인가요?
- 원격이 요청을 적용했지만 응답 전에 프로세스가 죽은 경우, 재전송이 안전하려면 M06의 어떤 보장이 필요하나요?
- terminal conflict만 있는 큐에서 자동 복구가 요청을 보내면 왜 안 되나요?
- 이 시작 복구와 WorkManager 기반 OS 백그라운드 복구의 차이는 무엇인가요?

### 30초 모범 답변

시작 경로는 먼저 Room의 저장 데이터를 화면에 제공해야 네트워크가 느리거나 없어도 앱이 usable합니다. 그 다음 저장소가 정상이고 재시도 가능한 pending intent가 있을 때만 한 번 복구를 시도합니다. 큐가 비었거나 terminal conflict만 있으면 네트워크를 보내지 않습니다. 원격 적용 후 응답 유실 같은 불확실한 결과는 같은 mutation identity와 hash로 정확히 재전송하고 서버가 원래 응답을 돌려주는 M06 멱등성이 있어야 안전합니다. M09는 앱 프로세스가 실제로 시작됐을 때의 복구일 뿐, 프로세스가 없는 동안 OS가 실행을 보장하는 백그라운드 작업은 아닙니다.

### 답변 핵심 키워드

local-first startup · recover only retryable pending · terminal skip · uncertain outcome · exact replay dependency · one recovery attempt · startup ≠ background guarantee

### 백지 구현

**구현 목표**

앱 시작 시 로컬 데이터를 먼저 읽고, 재시도 가능한 pending mutation이 있을 때만 동기화를 한 번 예약하는 시작 조정기를 구현한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class StartupSnapshot(
    val items: List<Item>,
    val storageError: String?,
    val recoveryAttempted: Boolean,
)

data class StartupPending(
    val terminalError: String?,
)

interface StartupStore {
    suspend fun readItems(): List<Item>
    suspend fun readPending(): List<StartupPending>
}

class StartupRecovery(
    private val store: StartupStore,
    private val synchronize: suspend () -> Unit,
) {
    suspend fun start(): StartupSnapshot = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: 저장된 Item, pending queue, 동기화 함수
- 출력: 즉시 사용 가능한 로컬 snapshot과 복구 시도 여부

**반드시 만족해야 할 조건**

- Item 읽기가 성공하면 네트워크 결과와 무관하게 먼저 반환 가능한 구조여야 한다.
- storage read가 실패하면 동기화를 시작하지 않는다.
- `terminalError == null`인 pending이 하나 이상일 때만 synchronize를 호출한다.
- 빈 큐 또는 terminal-only 큐에서는 네트워크 호출이 0회다.
- 한 번의 `start()`에서 synchronize를 중복 호출하지 않는다.
- synchronize 실패가 저장된 Item을 지우거나 숨기지 않는다.

**경계 조건**

- 빈 로컬 DB
- 빈 pending queue
- terminal과 nonterminal이 섞인 큐
- 저장 읽기 실패
- synchronize 실패 또는 취소
- `start()`가 동시에 두 번 호출됨

**실패 조건과 제약**

- WorkManager나 알람은 구현하지 않는다.
- 멱등 재전송은 synchronize의 계약으로 가정하고, 없을 때의 위험을 설명한다.
- 화면에 local snapshot을 언제 공개할지 API 형태를 바꿔도 되지만 local-first 원칙을 지켜야 한다.

### 구현 후 자가 검증

- [ ] 로컬 Item이 네트워크 완료를 기다리지 않고 이용 가능한가?
- [ ] storage 오류에서 네트워크가 호출되지 않는가?
- [ ] empty/terminal-only 큐에서 호출 수가 0인가?
- [ ] nonterminal이 있을 때 호출 수가 정확히 1인가?
- [ ] sync 실패 뒤에도 Item이 유지되는가?
- [ ] 동시에 시작할 때 중복 복구를 막을 필요와 범위를 설명할 수 있는가?
- [ ] 원격 적용/응답 유실 재전송이 M06 identity에 의존함을 설명할 수 있는가?

### 구현 후 설명할 것

1. local-first로 화면 사용 가능성과 복구를 분리한 이유
2. terminal intent를 자동 재전송하지 않는 안전 규칙
3. M06의 exact replay가 M09의 프로세스 사망 복구를 가능하게 하는 방식
4. 시작 때 한 번 시도하는 정책과 지속 백그라운드 스케줄링의 차이
5. 시작 동기화가 느린 경우 UI 상태와 취소를 어떻게 표현할지

### 원본 확인 위치

- Thread: M09
- 커밋: 현재 기록에서 커밋 제목 미노출(일반 시작 복구 구현)
- 파일·컴포넌트: `ItemSync.recoverPending()`, `ItemScreen`, `LaunchedEffect`, `accessStorage`, `verification/startup_recovery.py`, `M05ScenarioInstrumentation`
- 관련 Thread: M06의 mutation identity·exact replay, M07의 terminal conflict 제외 규칙, M10의 OS 지속 작업과 연결된다.
