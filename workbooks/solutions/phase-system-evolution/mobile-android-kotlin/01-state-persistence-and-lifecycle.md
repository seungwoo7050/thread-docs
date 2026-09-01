# 상태, 영속성, 생명주기 면접 워크북

이 문서는 현재 프로젝트 기록에서 실제로 확인되는 이름과 동작만 사용한다. 각 백지 구현은 원본을 복원하는 문제가 아니라, 같은 요구사항과 invariant를 만족하는 작은 면접 문제다.

<a id="w01"></a>
## [M01 / `feat(android): establish memory-only Item CRUD`] W01. 안정적인 식별자와 결정론적 로컬 상태

### 면접 질문

`MemoryItemStore`에서 제목을 수정해도 행의 식별자가 유지되어야 하는 이유는 무엇인가요? 제목이나 목록 위치를 식별자로 쓰면 어떤 문제가 생기나요?

꼬리 질문:

- 테스트에서 `nextId`와 `now`를 외부에서 주입한 이유는 무엇인가요?
- 변경 때마다 새 목록 스냅샷을 만드는 방식과 내부 가변 컬렉션을 직접 노출하는 방식의 차이는 무엇인가요?
- M01의 `version = 0`에는 어떤 의미가 있고, 어떤 의미까지 부여하면 안 되나요?

### 30초 모범 답변

행 식별자는 표시 값이나 정렬 위치와 분리해야 합니다. 제목은 수정될 수 있고 목록 위치도 삽입·삭제에 따라 바뀌므로 둘을 키로 쓰면 수정이 삭제 후 재생성처럼 보이거나 잘못된 행에 토글·삭제가 적용될 수 있습니다. M01에서는 ID와 시계를 주입해 테스트를 결정론적으로 만들고, 외부에는 불변 목록 스냅샷을 노출해 임의 변경을 막았습니다. `version = 0`은 아직 원격 동기화 의미가 없는 로컬 기본값일 뿐, 충돌 판정용 버전으로 해석하지 않습니다.

### 답변 핵심 키워드

`stable identity` · 표시 값과 키 분리 · 결정론적 의존성 주입 · 불변 스냅샷 · 위치 기반 키 금지 · `version=0`의 범위

### 백지 구현

**구현 목표**

메모리에서 Item CRUD를 수행하는 저장소를 구현한다. 제목을 바꾸거나 완료 상태를 토글해도 기존 ID가 유지되어야 하며, 외부에서 저장소 내부 목록을 직접 변경할 수 없어야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class Item(
    val id: String,
    val title: String,
    val completed: Boolean,
    val version: Long,
    val updatedAt: Long,
)

class MemoryItemStore(
    private val nextId: () -> String,
    private val now: () -> Long,
) {
    fun items(): List<Item> = TODO("직접 구현")
    fun create(title: String): Item = TODO("직접 구현")
    fun rename(id: String, title: String): Item = TODO("직접 구현")
    fun setCompleted(id: String, completed: Boolean): Item = TODO("직접 구현")
    fun delete(id: String): Boolean = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: 생성 제목, 대상 ID, 새 제목, 완료 여부
- 출력: 생성·수정된 Item 또는 삭제 성공 여부, 현재 목록의 읽기 전용 스냅샷

**반드시 만족해야 할 조건**

- 생성할 때만 새 ID를 발급한다.
- 이름 변경과 완료 상태 변경은 기존 ID를 유지한다.
- 각 성공한 생성·수정은 `now()` 값을 기록한다.
- M01 범위에서 `version`은 항상 0이다.
- `items()`를 통해 받은 목록이나 요소를 조작해 저장소 상태를 바꿀 수 없어야 한다.
- 존재하지 않는 ID를 수정할 때의 정책은 일관되어야 한다. 예외 또는 명시적 실패 중 하나를 선택하고 설명한다.

**경계 조건**

- 빈 목록
- 첫 번째/마지막 행 삭제
- 같은 완료 값을 다시 설정
- 같은 제목으로 다시 이름 변경
- 존재하지 않는 ID

**실패 조건과 제약**

- 디스크, 네트워크, 코루틴은 사용하지 않는다.
- 제목 중복은 허용하되 ID가 충돌해서는 안 된다.
- 10~15분 안에 구현할 수 있는 범위로 유지한다.

### 구현 후 자가 검증

- [ ] 두 Item의 제목이 같아도 ID가 서로 다른가?
- [ ] rename, toggle 뒤에도 대상 ID가 그대로인가?
- [ ] 삭제가 다른 행의 상태를 바꾸지 않는가?
- [ ] 외부에서 받은 목록을 바꿔 내부 상태를 훼손할 수 없는가?
- [ ] `nextId()`는 생성 성공마다 정확히 한 번만 호출되는가?
- [ ] `now()` 호출 규칙이 생성·변경 작업마다 일관적인가?
- [ ] 시간 복잡도를 설명할 수 있는가? 현재 자료구조에서 ID 조회·수정은 보통 O(n)이다.

### 구현 후 설명할 것

1. 제목이나 인덱스가 아니라 별도 ID를 도메인 식별자로 둔 이유
2. 불변 Item과 목록 스냅샷이 상태 변경 경계를 명확히 만드는 방식
3. `nextId`와 `now` 주입이 테스트 결정론을 높이는 이유
4. List 기반 O(n) 조회를 받아들인 범위와 Map/순서 목록으로 바꿀 시점

### 원본 확인 위치

- Thread: M01
- 커밋: `feat(android): establish memory-only Item CRUD`
- 파일·컴포넌트: `MemoryItemStore`, `Item`, `ItemRow`, `toRows()`, `ItemScreen`
- 관련 Thread: M02는 이 메모리 모델을 Room 영속 모델로 교체한다.

---

<a id="w02"></a>
## [M02 / `feat(android): persist Items across app process restart`] W02. 커밋된 상태만 노출하는 영속 CRUD와 리소스 수명

### 면접 질문

로컬 CRUD를 Room에 저장할 때 UI를 먼저 낙관적으로 갱신하지 않고, 쓰기 후 데이터베이스를 다시 읽은 값을 화면에 반영한 이유는 무엇인가요?

꼬리 질문:

- Item 변경과 UI 성공 표시 사이의 invariant는 무엇인가요?
- 쓰기가 실패했을 때 입력 draft와 마지막으로 확인된 목록을 각각 어떻게 다뤄야 하나요?
- `MainActivity`가 보유한 데이터베이스 핸들을 닫는 시점과 그 이유는 무엇인가요?
- 알 수 없는 스키마 버전에서 destructive fallback을 쓰지 않은 선택은 어떤 데이터를 보호하나요?

### 30초 모범 답변

화면의 성공 상태는 데이터베이스 커밋과 일치해야 합니다. UI를 먼저 바꾸면 쓰기 실패 뒤 화면만 성공한 것처럼 보이는 이중 상태가 생깁니다. 그래서 한 변경을 저장하고, 커밋된 행을 다시 읽은 뒤에만 확인된 목록을 갱신합니다. 실패하면 사용자가 입력한 draft와 이전 확인 목록을 유지해 재시도가 가능하게 하고, 저장소 오류를 별도로 보여 줍니다. 데이터베이스 핸들은 Activity 수명에 맞춰 닫아 누수와 열린 파일 핸들을 방지하고, 알 수 없는 스키마는 자동 삭제보다 실패를 택해 사용자 데이터를 보존합니다.

### 답변 핵심 키워드

커밋된 상태가 진실 · optimistic publish 금지 · read-after-write · draft 보존 · last-known-good · Room transaction · close 책임 · 비파괴 마이그레이션

### 백지 구현

**구현 목표**

영속 저장소 작업을 실행하고, 성공한 경우에만 커밋된 최신 목록을 UI 상태로 공개하는 작은 조정 계층을 구현한다. 저장 실패 시 draft와 이전 목록은 유지한다.

**인터페이스 또는 함수 시그니처**

```kotlin
interface ItemPersistence {
    suspend fun insert(title: String)
    suspend fun rename(id: String, title: String)
    suspend fun readAll(): List<Item>
    fun close()
}

data class ScreenState(
    val confirmedItems: List<Item>,
    val draft: String,
    val busy: Boolean,
    val storageError: String?,
)

class ItemEditCoordinator(
    private val persistence: ItemPersistence,
) {
    suspend fun submitCreate(state: ScreenState): ScreenState = TODO("직접 구현")
    suspend fun submitRename(state: ScreenState, id: String): ScreenState = TODO("직접 구현")
    fun close() = TODO("직접 구현")
}
```

**입력과 출력**

- 입력: 현재 화면 상태, 생성/수정 요청
- 출력: 성공 시 커밋된 목록과 비워진 draft, 실패 시 기존 목록·draft와 오류가 있는 새 상태

**반드시 만족해야 할 조건**

- 성공으로 간주하기 전에 저장 쓰기와 후속 읽기가 모두 끝나야 한다.
- 쓰기 또는 읽기 중 하나라도 실패하면 `confirmedItems`와 `draft`를 변경하지 않는다.
- busy 상태는 모든 경로에서 해제한다.
- 저장 오류와 입력 검증 오류를 같은 상태로 뭉개지 않는다.
- `close()` 호출 이후 추가 작업을 허용할지 금지할지 정책을 정한다.

**경계 조건**

- 쓰기는 성공했지만 후속 읽기가 실패하는 경우
- 읽기 결과가 빈 목록인 경우
- 작업 도중 예외가 발생하는 경우
- close가 두 번 호출되는 경우
- 존재하지 않는 ID 수정

**실패 조건과 제약**

- 실제 Room 코드는 작성하지 않아도 된다. 인터페이스 뒤의 저장소는 fake로 테스트한다.
- UI 프레임워크 API를 사용하지 않는다.
- 20분 이내 구현 범위다.

### 구현 후 자가 검증

- [ ] 성공 상태가 저장 완료 전 노출되지 않는가?
- [ ] 쓰기 실패 때 이전 목록과 draft가 그대로인가?
- [ ] 후속 읽기 실패 때도 성공처럼 draft를 지우지 않는가?
- [ ] busy가 정상·실패·취소 경로에서 모두 해제되는가?
- [ ] close가 누락되거나 중복 호출되어도 정책이 명확한가?
- [ ] 오류가 다음 성공 후 적절히 지워지는가?
- [ ] 저장소가 한 변경에 여러 SQL 문을 요구한다면 트랜잭션 경계를 어디에 둘지 설명할 수 있는가?

### 구현 후 설명할 것

1. UI가 아닌 데이터베이스를 단일 진실 공급원으로 둔 이유
2. 쓰기 후 재조회가 주는 정합성과 추가 I/O 비용의 trade-off
3. 실패 때 draft와 마지막 확인 목록을 보존하는 사용자 경험·복구 이점
4. Activity 수명에 묶인 핸들 관리와 더 긴 수명 스코프로 옮길 때의 변화
5. destructive migration을 피한 이유와 명시적 마이그레이션 비용

### 원본 확인 위치

- Thread: M02
- 커밋: `feat(android): persist Items across app process restart`
- 파일·컴포넌트: `ItemStore`, `ItemDao`, `ItemDatabase`, `MainActivity`, `process_restart.py`
- 관련 Thread: M04의 canonical rows·refresh metadata 트랜잭션, M05의 Item·pending intent 동시 커밋으로 확장된다.

---

<a id="w03"></a>
## [M04 / 현재 기록에서 커밋 제목 미노출(focus 순서 수리)] W03. UI 상태 전이 순서와 포커스 자원 invariant

### 면접 질문

Compose 입력창이 포커스를 가진 상태에서 `busy=true`로 컨트롤을 비활성화한 뒤 포커스를 해제하면 왜 문제가 될 수 있었나요? 포커스 해제를 먼저 수행한 수정의 핵심 invariant를 설명해 주세요.

꼬리 질문:

- 저장 성공 후에만 draft를 비우는 규칙과 포커스를 작업 시작 전에 해제하는 규칙은 왜 서로 다른 시점에 적용되나요?
- 단순히 테스트 대기 시간을 늘리는 방식이 근본 해결이 아닌 이유는 무엇인가요?
- 포커스, IME, 비동기 저장, Compose recomposition이 얽힌 테스트는 어떤 관찰 조건을 기다려야 하나요?

### 30초 모범 답변

포커스를 가진 입력 컴포넌트를 먼저 비활성화하면 그 포커스 대상이 트리에서 제거되는 동안 포커스 계층은 아직 활성 경로를 가리킬 수 있습니다. 이 프로젝트에서는 그 순서가 `ActiveParent with no focused child` 상태로 이어졌습니다. 따라서 작업 시작의 invariant를 "포커스 해제 완료 후 컨트롤 비활성화"로 두었습니다. 반면 draft는 저장 성공 여부와 연결된 사용자 데이터라 성공 콜백에서만 지웁니다. 테스트도 임의 지연 대신 커밋된 텍스트, 활성화된 빈 편집기, 포커스 해제, IME 숨김 같은 실제 준비 조건을 확인해야 합니다.

### 답변 핵심 키워드

순서 invariant · focus target 수명 · disable 전 clear · IME 외부 비동기 · 성공 전용 draft clear · 조건 기반 동기화 · sleep 금지

### 백지 구현

**구현 목표**

입력 중인 편집기를 비동기 저장 상태로 전환하는 순수 상태 전이기를 구현한다. 포커스 해제 요청이 컨트롤 비활성화보다 먼저 일어나고, draft는 저장 성공일 때만 초기화되어야 한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class EditorUiState(
    val draft: String,
    val focused: Boolean,
    val busy: Boolean,
    val error: String?,
)

sealed interface EditorEvent {
    data object SubmitRequested : EditorEvent
    data object FocusCleared : EditorEvent
    data object SaveSucceeded : EditorEvent
    data class SaveFailed(val message: String) : EditorEvent
}

data class EditorTransition(
    val state: EditorUiState,
    val effects: List<EditorEffect>,
)

sealed interface EditorEffect {
    data object RequestFocusClear : EditorEffect
    data object StartSave : EditorEffect
}

fun reduceEditor(state: EditorUiState, event: EditorEvent): EditorTransition =
    TODO("직접 구현")
```

**입력과 출력**

- 입력: 현재 편집기 상태와 사용자/플랫폼/저장 이벤트
- 출력: 다음 상태와 실행해야 할 효과 목록

**반드시 만족해야 할 조건**

- 포커스가 있는 상태의 제출은 먼저 `RequestFocusClear` 효과를 내야 한다.
- `FocusCleared`를 받은 뒤에만 busy 전환과 저장 시작을 허용한다.
- 저장 성공일 때만 draft와 오류를 초기화한다.
- 저장 실패 시 draft를 보존하고 busy를 해제한다.
- busy 중 중복 제출의 정책을 명확히 한다.

**경계 조건**

- 이미 포커스가 없는 상태에서 제출
- 포커스 해제 이벤트가 중복 전달됨
- 저장 성공/실패 이벤트가 늦게 또는 중복 전달됨
- busy 중 제출
- 빈 draft

**실패 조건과 제약**

- 실제 Compose API나 FocusManager는 호출하지 않는다.
- 효과 실행 순서는 반환된 목록과 상태 전이 계약으로만 표현한다.
- 15~20분 범위다.

### 구현 후 자가 검증

- [ ] `StartSave`가 `RequestFocusClear`보다 먼저 나오지 않는가?
- [ ] 저장 실패 뒤 draft가 남아 있는가?
- [ ] 저장 성공 전 draft가 비워지지 않는가?
- [ ] 중복/지연 이벤트가 잘못된 상태로 되돌리지 않는가?
- [ ] busy 중 중복 저장을 차단하거나 직렬화하는 정책이 있는가?
- [ ] 상태와 일회성 효과가 분리되어 있는가?
- [ ] 테스트가 임의 sleep 없이 이벤트 순서와 최종 invariant를 검증할 수 있는가?

### 구현 후 설명할 것

1. 포커스를 UI 장식이 아니라 수명과 순서를 가진 자원으로 다룬 이유
2. 작업 시작 전 효과와 저장 성공 후 데이터 정리 효과를 분리한 이유
3. 단순 timeout 증가보다 상태 기반 동기화가 안정적인 이유
4. reducer 방식과 Composable 내부 명령형 코드의 trade-off

### 원본 확인 위치

- Thread: M04
- 커밋: 현재 기록에서 커밋 제목 미노출(focus 순서 수리)
- 파일·컴포넌트: `MainActivity.kt`, `ItemScreen`, `accessStorage`, `focus.clearFocus()`, `busy`
- 관련 Thread: M04의 `ItemUiTest#awaitCrudText` 수리는 테스트 준비 조건을 명시했고, M08은 편집기 상태를 Activity 재생성까지 보존한다.

---

<a id="w04"></a>
## [M08 / `feat(M08): retain editor state across Activity recreation`] W04. UI 임시 상태와 도메인 영속 상태의 분리

### 면접 질문

선택된 Item ID와 draft 제목을 `rememberSaveable`로 복원하면서도 Room이나 pending mutation에는 저장하지 않은 이유는 무엇인가요?

꼬리 질문:

- Activity 재생성과 OS 프로세스 사망은 어떤 보장 차이가 있나요?
- `ItemEditor.Saver`가 보존해야 하는 최소 상태는 무엇인가요?
- 복원만으로는 데이터 변경이 발생하지 않고 명시적 Save에서만 `ItemStore`를 호출해야 하는 이유는 무엇인가요?
- 동기화 상태 Flow를 `STARTED` 생명주기에서 수집한 이유는 무엇인가요?

### 30초 모범 답변

선택과 draft는 아직 커밋되지 않은 UI 상태이므로 Item 테이블이나 동기화 큐에 넣으면 사용자가 Save하지 않은 값이 도메인 변경으로 오염됩니다. M08은 선택 ID와 draft만 `Saver`로 보존해 같은 프로세스의 Activity 재생성을 통과시키고, 실제 저장은 명시적 Save에서 기존 `ItemStore` 경로로 한 번만 수행합니다. 이 보장은 프로세스 사망까지의 내구성을 뜻하지 않습니다. Flow 수집도 화면이 `STARTED` 이상일 때만 활성화해 보이지 않는 UI가 불필요하게 상태를 소비하지 않도록 했습니다.

### 답변 핵심 키워드

UI state vs durable state · 최소 저장 상태 · explicit submit · side-effect-free restore · Activity recreation ≠ process death · `rememberSaveable` · `flowWithLifecycle(STARTED)`

### 백지 구현

**구현 목표**

편집기 선택과 draft를 직렬화·복원하되, 복원 자체는 도메인 저장을 수행하지 않는 모델을 구현한다. 사용자가 명시적으로 submit할 때만 저장소의 create 또는 rename을 호출한다.

**인터페이스 또는 함수 시그니처**

```kotlin
data class ItemEditor(
    val itemId: String? = null,
    val title: String = "",
) {
    fun saveState(): List<String?> = TODO("직접 구현")

    companion object {
        fun restoreState(values: List<String?>): ItemEditor = TODO("직접 구현")
    }

    suspend fun submit(store: ItemStorePort) {
        TODO("직접 구현")
    }
}

interface ItemStorePort {
    suspend fun create(title: String)
    suspend fun rename(id: String, title: String)
}
```

**입력과 출력**

- 입력: 선택된 ID, draft, 저장된 값 목록, 저장소
- 출력: 복원된 편집기 또는 한 번의 create/rename 호출

**반드시 만족해야 할 조건**

- 선택 ID가 null이면 생성 모드, 값이 있으면 수정 모드다.
- 저장·복원 왕복 후 모드와 draft가 동일해야 한다.
- `restoreState`는 저장소를 호출하지 않는다.
- `submit`은 현재 모드에 맞는 저장소 메서드를 정확히 하나 호출한다.
- 잘못된 저장 형식의 처리 정책을 명시한다.

**경계 조건**

- null ID와 빈 제목
- 비어 있거나 길이가 다른 저장 값 목록
- 두 번째 값이 null인 손상된 상태
- 복원 뒤 대상 Item이 이미 삭제된 경우
- submit 실패

**실패 조건과 제약**

- 실제 Compose `Saver` 타입을 쓰지 않아도 된다.
- draft를 데이터베이스에 저장하지 않는다.
- submit 성공 후 편집기를 초기화하는 책임이 모델과 호출자 중 어디에 있는지 선택하고 설명한다.

### 구현 후 자가 검증

- [ ] 생성 모드와 수정 모드가 모두 왕복 복원되는가?
- [ ] 복원 전후에 저장소 호출 횟수가 0인가?
- [ ] 한 번의 submit이 정확히 한 변경만 만드는가?
- [ ] submit 실패 때 draft를 유지할 수 있는 구조인가?
- [ ] 대상 Item이 삭제된 경우의 정책이 명확한가?
- [ ] 저장 형식 손상 시 조용히 잘못된 편집을 만들지 않는가?
- [ ] Activity 재생성 보장과 프로세스 사망 보장을 구분해 설명할 수 있는가?

### 구현 후 설명할 것

1. 저장할 UI 상태를 ID와 draft로 최소화한 이유
2. 복원과 커밋을 분리해 의도치 않은 변경을 막는 방식
3. 같은 프로세스 Activity 재생성만 보장하는 한계
4. lifecycle-aware Flow 수집이 자원 사용과 중복 관찰을 줄이는 방식
5. draft까지 영구 저장해야 하는 제품 요구가 생길 때 필요한 별도 설계

### 원본 확인 위치

- Thread: M08
- 커밋: `feat(M08): retain editor state across Activity recreation`
- 파일·컴포넌트: `ItemEditor`, `ItemEditor.Saver`, `ItemEditor.submit`, `rememberSaveable`, `flowWithLifecycle`, `M08ActivityStateTest#frozenM08ActivityLifecycleSequence`
- 관련 Thread: M02는 커밋된 Item의 프로세스 재시작 내구성을 다루며, M09는 pending sync의 프로세스 사망 복구를 다룬다. M08은 draft의 프로세스 사망 내구성을 주장하지 않는다.
