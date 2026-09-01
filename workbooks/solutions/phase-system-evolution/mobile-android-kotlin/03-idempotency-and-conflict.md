# 멱등성, 수신확인, 충돌 해결 면접 워크북

이 문서는 네트워크의 "원격 적용 여부를 알 수 없는 실패"와 여러 클라이언트의 동시 변경을 다룬다. 핵심은 요청 식별자, immutable envelope, 원자적 ACK, optimistic concurrency invariant다.

<a id="w09"></a>
## [M06 / 현재 기록에서 커밋 제목 미노출(멱등 재전송·수신확인 구현)] W09. canonical 요청과 mutation identity를 이용한 exact replay

### 면접 질문

원격 저장은 성공했지만 응답이 유실된 경우, 단순히 같은 POST를 다시 보내는 것만으로는 왜 안전하지 않나요? `clientMutationId`와 canonical payload hash를 함께 저장한 설계를 설명해 주세요.

꼬리 질문:

- 같은 `clientMutationId`와 같은 hash가 다시 오면 서버가 왜 새 응답을 만들지 않고 원래 status/body를 반환해야 하나요?
- 같은 ID에 다른 hash가 오면 retry가 아니라 `identity_conflict`로 중단해야 하는 이유는 무엇인가요?
- JSON key 순서, 공백, escape 방식이 hash에 영향을 주지 않도록 canonical form이 필요한 이유는 무엇인가요?
- request metadata를 hash 대상에서 제외하고, method·path·payload를 포함한 이유는 무엇인가요?
- DELETE payload를 명시적 JSON null로 canonicalize한 이유는 무엇인가요?

### 30초 모범 답변

응답 유실 뒤 재전송할 때 서버가 새 요청으로 보면 중복 생성이나 중복 변경이 발생할 수 있습니다. 그래서 intent를 만들 때 한 번 발급한 `clientMutationId`와 method·path·payload의 결정적 canonical hash를 영구 저장합니다. 서버는 처음 처리 결과를 ID와 hash에 묶어 보관하고, 같은 쌍의 재전송에는 side effect 없이 원래 status와 body를 그대로 반환합니다. 같은 ID에 다른 hash가 오면 ID 재사용이나 데이터 손상일 수 있으므로 409 terminal conflict로 중단합니다. canonical form이 안정적이어야 프로세스·언어·재시작을 넘어 동일한 의미의 요청이 동일 hash를 가집니다.

### 답변 핵심 키워드

idempotency key · immutable envelope · canonical JSON · SHA-256 · exact cached response · same ID/same hash · same ID/different hash conflict · uncertain outcome

### 백지 구현

**구현 목표**

제한된 JSON 값 모델을 deterministic canonical 문자열로 변환하고, HTTP method·path·payload를 묶은 요청 hash를 계산하는 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```kotlin
sealed interface JsonValue {
    data object Null : JsonValue
    data class Bool(val value: Boolean) : JsonValue
    data class Number(val value: Long) : JsonValue
    data class Text(val value: String) : JsonValue
    data class Array(val values: List<JsonValue>) : JsonValue
    data class Object(val values: Map<String, JsonValue>) : JsonValue
}

data class MutationRequest(
    val method: String,
    val path: String,
    val payload: JsonValue,
) {
    fun canonical(): String = TODO("직접 구현")
    fun sha256Hex(): String = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: method, path, 중첩 가능한 JSON payload
- 출력: compact canonical JSON 문자열과 UTF-8 SHA-256 hex

**반드시 만족해야 할 조건**

- Object key는 모든 중첩 단계에서 결정적인 순서로 정렬한다.
- 불필요한 공백을 넣지 않는다.
- 문자열 escape 규칙이 입력 순서나 플랫폼 기본 locale에 의존하지 않아야 한다.
- Boolean, 정수, null을 문자열과 구분한다.
- Array 순서는 보존한다.
- method와 path가 hash 대상에 포함되어야 한다.
- `clientMutationId`, 선언된 `payloadHash` 같은 전송 metadata는 canonical payload에서 제외한다.
- 같은 의미와 같은 값 모델 입력은 언제나 같은 bytes를 만든다.

**경계 조건**

- 빈 object와 빈 array
- 깊게 중첩된 object
- 한글, 이모지, 줄바꿈, 따옴표, 역슬래시
- 서로 다른 insertion order의 Map
- null DELETE payload
- 매우 큰 정수 또는 허용하지 않는 숫자 타입

**실패 조건과 제약**

- 부동소수점은 지원하지 않아도 된다. 지원한다면 정규화 규칙을 명시한다.
- 표준 JSON 라이브러리를 사용해도 되지만 key 정렬과 byte 결과를 직접 통제해야 한다.
- 서버 dedup 저장소는 구현하지 않는다.
- 25~30분 범위다.

### 구현 후 자가 검증

- [ ] Map insertion order가 달라도 canonical과 hash가 같은가?
- [ ] payload는 같지만 method 또는 path가 다르면 hash가 다른가?
- [ ] array 순서가 바뀌면 hash도 달라지는가?
- [ ] 특수문자와 Unicode가 안정적으로 escape·UTF-8 인코딩되는가?
- [ ] null, 문자열 `"null"`, 빈 object가 서로 구분되는가?
- [ ] canonical 출력에 mutation metadata가 섞이지 않는가?
- [ ] 같은 identity/다른 hash를 자동 retry하면 안 되는 이유를 설명할 수 있는가?

### 구현 후 설명할 것

1. 단순 요청 재전송과 application-level idempotency의 차이
2. ID만 비교하지 않고 payload hash까지 확인한 이유
3. 원래 status/body까지 저장해 exact replay한 이유
4. canonical JSON 규칙을 프로토콜 계약으로 고정할 때의 호환성 비용
5. SHA-256은 우발적 충돌·변조 감지에 쓰이며 인증 자체는 아니라는 경계

### 원본 확인 위치

- Thread: M06
- 커밋: 현재 기록에서 커밋 제목 미노출(멱등 재전송·수신확인 구현)
- 파일·컴포넌트: `PendingMutation`, `MutationRequest`, `PendingMutation.request().canonical()`, `hash()`, `clientMutationId`, `payloadHash`, `MutationIdentityConflict`, `fixture/server.py`
- 관련 Thread: M06 기준선 `test(android): freeze M06 lost-acknowledgment baseline`, M09 시작 복구, M10 자동 retry가 이 identity를 재사용한다.

---

<a id="w10"></a>
## [M06 / 현재 기록에서 커밋 제목 미노출(멱등 재전송·수신확인 구현)] W10. receipt 기록과 dequeue의 원자성

### 면접 질문

서버가 성공 응답을 돌려준 뒤 로컬에서 ACK receipt를 기록하고 pending mutation을 제거할 때, 두 작업을 왜 하나의 트랜잭션으로 묶어야 하나요?

꼬리 질문:

- receipt insert는 성공했지만 pending delete가 실패하면 다음 시작에서 어떤 모순이 생기나요?
- pending delete가 먼저 성공하고 receipt insert가 실패하면 어떤 정보가 사라지나요?
- 실패 주입 테스트가 `acknowledged_mutations` INSERT와 `pending_mutations` DELETE 각각에서 rollback을 확인한 이유는 무엇인가요?
- 원격 결과를 적용할 때 이후의 optimistic local edit를 덮어쓰지 않아야 하는 이유는 무엇인가요?

### 30초 모범 답변

ACK는 "이 identity의 원격 결과를 로컬이 영구 수락했고 더 이상 재전송하지 않는다"는 하나의 상태 전이입니다. receipt 저장과 queue 제거가 분리되면 receipt와 pending이 동시에 남거나, 둘 다 없이 intent가 사라지는 중간 상태가 생깁니다. 그래서 원래 status/body 저장, 필요한 canonical 상태 반영, 해당 pending 제거를 한 트랜잭션으로 커밋합니다. 어느 SQL 단계에서 실패해도 전체가 rollback되어 동일 envelope를 다시 보낼 수 있어야 합니다. 이때 오래된 응답이 뒤에 생성된 로컬 optimistic 값을 덮어쓰지 않도록 적용 범위도 제한해야 합니다.

### 답변 핵심 키워드

atomic acknowledgment · receipt + dequeue · all-or-nothing · crash consistency · replayable pending · failure injection · stale response protection

### 백지 구현

**구현 목표**

원격 성공 결과를 수신확인 테이블에 기록하고 정확한 pending entry를 제거하는 트랜잭션 함수를 구현한다. 어느 단계가 실패해도 기존 pending 상태가 유지되어야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class PendingEnvelope(
    val sequence: Long,
    val clientMutationId: String,
    val payloadHash: String,
)

data class RemoteReceipt(
    val statusCode: Int,
    val responseBody: String,
)

data class AcknowledgedMutation(
    val clientMutationId: String,
    val payloadHash: String,
    val statusCode: Int,
    val responseBody: String,
)

interface AckDao {
    suspend fun <T> inTransaction(block: suspend AckDao.() -> T): T
    suspend fun findPending(sequence: Long): PendingEnvelope?
    suspend fun insertAcknowledged(value: AcknowledgedMutation)
    suspend fun deletePending(sequence: Long): Int
}

suspend fun acknowledge(
    dao: AckDao,
    expected: PendingEnvelope,
    receipt: RemoteReceipt,
) {
    TODO("직접 구현")
}
```

**입력과 출력**

- 입력: ACK할 것으로 기대한 immutable pending envelope, 원격 status/body
- 출력: 성공 시 acknowledged row 1개와 pending row 제거, 실패 시 기존 상태 유지

**반드시 만족해야 할 조건**

- 트랜잭션 안에서 현재 pending row가 기대한 sequence·identity·hash와 동일한지 검증한다.
- receipt insert와 pending delete는 하나의 원자적 단위다.
- 삭제 행 수가 정확히 1이 아니면 실패한다.
- mismatch가 있으면 조용히 다른 row를 제거하지 않는다.
- 재실행 정책을 설명한다. 같은 ACK가 이미 기록된 상황을 idempotent하게 받을지 명시적 오류로 볼지 선택한다.

**경계 조건**

- pending row가 없음
- 같은 sequence에 hash가 변조됨
- acknowledged row unique 충돌
- receipt insert 후 delete 실패
- delete 대상 행 수 0 또는 2 이상으로 보고됨
- 트랜잭션 commit 자체 실패

**실패 조건과 제약**

- 실제 SQL은 작성하지 않아도 된다.
- remote 호출은 이 함수 바깥에서 이미 성공한 것으로 가정한다.
- 이후 successor mutation rebasing은 W12에서 별도로 다룬다.

### 구현 후 자가 검증

- [ ] 정상 경로에서 receipt 1개와 pending 제거가 함께 보이는가?
- [ ] insert 실패 시 pending이 남고 receipt가 없는가?
- [ ] delete 실패 시 insert도 rollback되는가?
- [ ] identity 또는 hash mismatch에서 아무 row도 바뀌지 않는가?
- [ ] 다른 pending entry를 실수로 제거하지 않는가?
- [ ] 프로세스가 각 단계 사이에 죽어도 트랜잭션 외부에는 완성 상태만 보이는가?
- [ ] 재시도 시 exact replay와 로컬 ACK가 어떻게 결합되는지 설명할 수 있는가?

### 구현 후 설명할 것

1. ACK를 두 SQL 문이 아니라 하나의 도메인 상태 전이로 본 이유
2. receipt를 보관하면 진단·정확 재생·중복 판정에 어떤 이점이 있는지
3. failure injection이 일반 happy-path 테스트보다 중요한 이유
4. 오래된 원격 응답을 적용할 때 후속 optimistic 변경을 보호해야 하는 이유
5. receipt 보존 기간과 저장 공간 trade-off

### 원본 확인 위치

- Thread: M06
- 커밋: 현재 기록에서 커밋 제목 미노출(멱등 재전송·수신확인 구현)
- 파일·컴포넌트: `AcknowledgedMutation`, `MutationResult`, `ItemStore.acknowledge`, `acknowledged_mutations`, `pending_mutations`, `ItemSync.synchronize`
- 관련 Thread: M07은 predecessor ACK와 즉시 뒤따르는 successor의 baseVersion 갱신까지 같은 원자적 전이로 확장한다.

---

<a id="w11"></a>
## [M07 / `feat(M07): preserve stale mutation conflicts and acknowledged bases`] W11. baseVersion 기반 optimistic concurrency와 terminal conflict 보존

### 면접 질문

로컬 UPDATE·DELETE intent에 `baseVersion`을 저장하고, 서버 409에서 original intent를 삭제하지 않고 `version_conflict` terminal 상태로 남긴 이유는 무엇인가요?

꼬리 질문:

- 현재 로컬 Item의 version을 전송 직전에 다시 읽어 base로 쓰면 어떤 race가 생기나요?
- `dispatched` 이후 identity, payload, base, hash를 immutable하게 유지해야 하는 이유는 무엇인가요?
- conflict에서 서버 canonical Item 또는 tombstone을 로컬에 저장하면서도 원래 로컬 시도를 남기는 이유는 무엇인가요?
- terminal conflict를 평범한 drain이 건너뛰고, 이후 사용자 명시적 편집은 새 identity와 새 base로 만들어야 하는 이유는 무엇인가요?
- 마이그레이션으로 base를 알 수 없는 기존 UPDATE·DELETE를 `base_version_unknown` terminal로 둔 이유는 무엇인가요?

### 30초 모범 답변

`baseVersion`은 사용자가 변경을 만든 순간 관찰한 원격 버전을 나타냅니다. 전송 직전에 현재 값을 다시 읽으면 그 사이 다른 ACK나 refresh가 version을 바꿔 원래 의도와 다른 조건으로 요청할 수 있습니다. 그래서 envelope 생성 시 base를 고정하고 dispatch 뒤에는 identity·payload·base·hash를 바꾸지 않습니다. 409는 일시 네트워크 오류가 아니라 자동 재시도로 해결되지 않는 의미 충돌이므로 original intent를 terminal로 보존하고, 서버 canonical Item이나 tombstone을 저장해 현재 진실도 보여 줍니다. 사용자가 다시 수정하면 새 identity와 최신 base로 새 의도를 만듭니다.

### 답변 핵심 키워드

optimistic concurrency · observed baseVersion · immutable dispatched envelope · terminal conflict · canonical remote state · tombstone · explicit resolution · unknown-base migration

### 백지 구현

**구현 목표**

버전 충돌 응답을 처리하는 순수 전이 함수를 구현한다. 원래 pending intent는 terminal로 남기고, 서버가 제공한 canonical Item 또는 삭제 tombstone을 현재 로컬 상태에 반영한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class VersionedItem(
    val id: String,
    val title: String,
    val completed: Boolean,
    val version: Long,
)

data class VersionedPending(
    val sequence: Long,
    val itemId: String,
    val clientMutationId: String,
    val payloadHash: String,
    val baseVersion: Long?,
    val dispatched: Boolean,
    val terminalError: String?,
)

sealed interface CanonicalConflictValue {
    data class Existing(val item: VersionedItem) : CanonicalConflictValue
    data class Deleted(val id: String, val version: Long) : CanonicalConflictValue
}

data class ConflictState(
    val items: Map<String, VersionedItem>,
    val pending: List<VersionedPending>,
    val tombstones: Map<String, Long>,
)

fun applyVersionConflict(
    state: ConflictState,
    sequence: Long,
    canonical: CanonicalConflictValue,
): ConflictState = TODO("직접 구현")
```

**입력과 출력**

- 입력: 현재 Item/pending/tombstone 상태, 충돌한 sequence, 서버 canonical 값
- 출력: canonical 현재 상태와 terminal 원본 intent를 모두 보존한 새 상태

**반드시 만족해야 할 조건**

- 정확한 sequence의 pending 하나만 `terminalError = "version_conflict"`로 바뀐다.
- identity, payloadHash, baseVersion, sequence는 바뀌지 않는다.
- canonical Existing은 해당 Item을 서버 버전으로 교체하고 관련 tombstone을 정리한다.
- canonical Deleted는 Item을 제거하고 tombstone을 기록한다.
- terminal pending은 삭제하지 않는다.
- 이미 terminal인 intent에 중복 conflict가 왔을 때의 정책을 명시한다.

**경계 조건**

- sequence가 없음
- pending itemId와 canonical ID가 다름
- 이미 terminal 상태
- canonical version이 현재보다 낮음
- Existing과 tombstone이 동시에 존재하는 손상 상태

**실패 조건과 제약**

- 실제 DB 트랜잭션은 구현하지 않는다. 함수 전체가 원자적으로 저장된다고 가정한다.
- 자동 merge는 하지 않는다.
- 사용자의 새 편집 생성은 별도 함수로 남긴다.

### 구현 후 자가 검증

- [ ] original identity/base/hash가 보존되는가?
- [ ] terminal intent가 ordinary retry 대상으로 남지 않는 구조인가?
- [ ] Existing conflict에서 canonical Item이 보이는가?
- [ ] Deleted conflict에서 Item이 제거되고 tombstone이 남는가?
- [ ] 다른 pending이 수정되지 않는가?
- [ ] ID 불일치나 손상 상태를 탐지하는가?
- [ ] 새 사용자 편집은 기존 terminal intent를 재활성화하지 않고 새 envelope를 만들어야 함을 설명할 수 있는가?

### 구현 후 설명할 것

1. baseVersion을 mutation 생성 시점에 고정한 이유
2. conflict를 retryable error와 분리해 terminal로 저장한 이유
3. 원격 canonical 상태와 실패한 로컬 의도를 동시에 보존한 이유
4. tombstone이 삭제 충돌의 버전 정보를 유지하는 방식
5. 자동 merge를 하지 않은 보수적 선택과 사용자 해결 UX의 trade-off

### 원본 확인 위치

- Thread: M07
- 커밋: `feat(M07): preserve stale mutation conflicts and acknowledged bases`
- 파일·컴포넌트: `PendingMutation.baseVersion`, `PendingMutation.dispatched`, `prepareDispatch(sequence)`, `terminalError`, `tombstones`, `ItemStore.acknowledge`
- 관련 Thread: M07 기준선 `test(M07): freeze stale mutation baseline`; M06의 immutable identity/hash와 ACK 원자성을 확장한다.

---

<a id="w12"></a>
## [M07 / `feat(M07): preserve stale mutation conflicts and acknowledged bases`] W12. predecessor ACK 뒤 successor base 재설정

### 면접 질문

같은 Item에 CREATE 다음 RENAME이 큐에 있는 상황에서 CREATE ACK가 version 1을 반환하면, 뒤 RENAME의 `baseVersion`을 어떻게 처리해야 하나요? 왜 최종 GET까지 기다리지 않았나요?

꼬리 질문:

- CREATE 응답의 canonical title로 로컬 Item 전체를 덮어쓰면 이후 optimistic rename이 왜 사라지나요?
- ACK transaction에서 "관찰된 version만 올리고 newer local fields/time은 유지"하는 invariant를 설명해 주세요.
- 이미 dispatch된 successor와 아직 dispatch되지 않은 successor의 변경 가능 범위는 어떻게 달라야 하나요?
- predecessor receipt, dequeue, Item known version, immediate successor base 준비를 한 원자적 전이로 묶은 이유는 무엇인가요?

### 30초 모범 답변

뒤 RENAME은 CREATE가 성공하기 전에는 원격 base를 알 수 없지만, CREATE ACK가 version 1을 주는 순간 정확한 predecessor version을 알 수 있습니다. 최종 GET을 기다리지 않고 ACK transaction 안에서 아직 dispatch되지 않은 immediate successor의 base를 1로 준비해야 순차 전송이 이어집니다. 이때 CREATE 응답 전체로 Item을 덮어쓰면 사용자가 이미 입력한 newer rename이 사라지므로, 로컬 optimistic 필드와 시각은 유지하고 원격에서 확인된 known version만 전진시킵니다. receipt·dequeue·known version·successor base는 함께 커밋되어야 중간 모순이 없습니다.

### 답변 핵심 키워드

predecessor ACK · immediate successor · rebase · known remote version · preserve optimistic fields · atomic chain advancement · no final GET dependency

### 백지 구현

**구현 목표**

하나의 Item에 대한 predecessor 성공 결과를 적용하면서, 로컬 optimistic 필드는 보존하고 아직 dispatch되지 않은 바로 다음 mutation의 baseVersion만 새 원격 version으로 준비하는 순수 전이 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class LocalItemState(
    val item: VersionedItem,
    val updatedAt: Long,
)

data class AckChainState(
    val item: LocalItemState,
    val pending: List<VersionedPending>,
    val acknowledgedIds: Set<String>,
)

data class AppliedAck(
    val sequence: Long,
    val clientMutationId: String,
    val canonicalVersion: Long,
)

fun applyAcknowledgmentAndPrepareSuccessor(
    state: AckChainState,
    ack: AppliedAck,
): AckChainState = TODO("직접 구현")
```

**입력과 출력**

- 입력: 현재 optimistic Item, 순서 있는 pending 목록, predecessor ACK와 canonical version
- 출력: predecessor가 제거되고 receipt 표시가 추가되며, 필요하면 successor base가 준비된 상태

**반드시 만족해야 할 조건**

- ACK 대상은 정확한 sequence·identity와 일치해야 한다.
- ACK 대상 pending만 제거한다.
- local title/completed/updatedAt은 ACK 전 값을 유지한다.
- Item의 known version은 `canonicalVersion`으로 전진한다.
- 같은 Item의 가장 가까운 아직 dispatch되지 않은 successor만 baseVersion을 준비한다.
- 이미 dispatch된 successor의 base/hash를 변경하지 않는다.
- 다른 Item의 pending은 변경하지 않는다.

**경계 조건**

- successor 없음
- 다른 Item의 pending이 사이에 있음
- 같은 Item successor가 여러 개 있음
- successor가 이미 dispatched
- ACK version이 현재 known version보다 낮음
- ACK identity가 이미 acknowledged됨

**실패 조건과 제약**

- 실제 response body 파싱은 생략한다.
- receipt 세부 값 대신 acknowledged ID 집합만 써도 된다.
- invalid version regression은 실패시켜도 된다.
- 구현 시간은 20~25분이다.

### 구현 후 자가 검증

- [ ] predecessor만 제거되는가?
- [ ] optimistic title/completed/time이 그대로인가?
- [ ] known version만 ACK version으로 전진하는가?
- [ ] 같은 Item의 정확한 immediate successor만 갱신되는가?
- [ ] dispatched successor는 immutable한가?
- [ ] 다른 Item의 큐가 보존되는가?
- [ ] 이 전이가 receipt/dequeue와 함께 저장되지 않으면 생길 중간 상태를 설명할 수 있는가?

### 구현 후 설명할 것

1. 최종 GET 없이 ACK version을 successor base로 전달한 이유
2. canonical 응답 전체를 덮어쓰지 않고 known version만 반영한 이유
3. immediate successor만 갱신하는 순서 invariant
4. dispatched envelope immutability와 재전송 일관성
5. 이 로직을 DB 트랜잭션으로 묶을 때 잠금 범위와 큐 길이 trade-off

### 원본 확인 위치

- Thread: M07
- 커밋: `feat(M07): preserve stale mutation conflicts and acknowledged bases`
- 파일·컴포넌트: `ItemStore.acknowledge`, `PendingMutation.baseVersion`, `PendingMutation.dispatched`, `prepareDispatch(sequence)`, `AcknowledgedMutation`
- 관련 Thread: M06의 receipt+dequeue 원자성을 확장하며, M05의 ordered queue가 predecessor/successor 순서를 제공한다.
