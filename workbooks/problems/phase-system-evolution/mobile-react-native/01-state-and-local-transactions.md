# 상태 모델과 로컬 트랜잭션

이 문서는 M01의 순수 상태 전이에서 출발해 M02 이후의 SQLite 트랜잭션과 스키마 진화까지 연결한다. 구현 문제는 원본 코드를 재현하는 문제가 아니라, 기록에서 확인되는 불변식과 실패 경계를 10~30분 안에 다시 설계하는 연습이다.

---

<a id="p01"></a>
## [M01 / `feat: establish in-memory React Native Item UI`] reducer와 Item 불변식

### 면접 질문

`itemsReducer`가 Item 생성·이름 변경·완료 토글·삭제를 모두 처리합니다. 이 상태 전이를 reducer로 모은 이유와, 각 액션 전후에 반드시 유지해야 하는 불변식을 설명해 보세요.

꼬리 질문:

- 렌더링 순서나 배열 인덱스를 Item 식별자로 사용하면 어떤 문제가 생깁니까?
- 공백 제목, 존재하지 않는 Item에 대한 변경, 같은 값으로의 이름 변경은 어떻게 처리해야 합니까?
- reducer가 새 배열을 반환한다는 사실만으로 불변성이 보장됩니까?
- `version`과 `updatedAt`은 어떤 변경에서 증가·갱신되어야 합니까?
- 이후 M02에서 상태 저장소가 SQLite로 바뀌어도 이 도메인 규칙은 어디에 남아 있어야 합니까?

### 30초 모범 답변

Item의 핵심 규칙을 reducer 한 곳에 모으면 UI 이벤트 순서와 무관하게 같은 입력이 같은 상태 전이를 만들고, 테스트도 화면이 아니라 도메인 상태를 기준으로 작성할 수 있습니다. 식별자는 배열 위치가 아니라 안정적인 `id`여야 삭제·정렬 뒤에도 같은 Item을 가리킵니다. 공백 제목이나 대상이 없는 변경은 기존 상태를 훼손하지 않아야 하고, 실제로 반영된 이름 변경·토글만 `version`과 `updatedAt`을 갱신해야 합니다. M02 이후에는 저장 방식이 바뀌어도 이 규칙을 트랜잭션 내부의 도메인 전이로 그대로 유지해야 합니다.

### 답변 핵심 키워드

순수 함수, 안정적인 ID, 불변성, no-op, 입력 정규화, 단일 상태 전이 지점, version, timestamp, UI와 도메인 분리

### 백지 구현

#### 구현 목표

다섯 필드로 구성된 Item 목록과 네 종류의 액션을 받아 다음 상태를 반환하는 순수 reducer를 구현한다. 저장소·React·네트워크 코드는 사용하지 않는다.

#### 인터페이스 또는 함수 시그니처

```ts
export type Item = {
  id: string;
  title: string;
  completed: boolean;
  version: number;
  updatedAt: number;
};

export type ItemAction =
  | {type: 'create'; id: string; title: string; now: number}
  | {type: 'rename'; id: string; title: string; now: number}
  | {type: 'toggle'; id: string; now: number}
  | {type: 'delete'; id: string};

export function reduceItems(
  state: readonly Item[],
  action: ItemAction,
): readonly Item[] {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 현재 Item 배열과 하나의 액션
- 출력: 액션이 반영된 다음 Item 배열
- 함수 밖의 상태, 현재 시각, 난수에 의존하지 않는다. 필요한 값은 액션에 포함한다.

#### 반드시 만족해야 할 조건

- Item의 `id`는 생성 뒤 이름 변경·토글 과정에서 바뀌지 않는다.
- 제목은 앞뒤 공백을 제거한 값을 사용한다.
- 공백 제목으로는 생성하거나 기존 제목을 지울 수 없다.
- 생성된 Item은 `completed: false`로 시작하고, 버전과 시각은 문제에서 정한 규칙대로 초기화한다.
- 실제 이름 변경과 토글은 해당 Item 하나만 변경한다.
- 기존 Item 객체와 배열을 직접 수정하지 않는다.
- 삭제는 대상 Item만 제거하며 다른 Item의 ID나 버전을 바꾸지 않는다.

#### 경계 조건

- 빈 목록
- 공백만 있는 제목
- 존재하지 않는 ID에 대한 rename, toggle, delete
- 기존 제목과 정규화 후 같은 제목으로 rename
- 목록의 첫 번째·마지막 Item 삭제
- 동일한 `now`가 연속으로 들어오는 경우

#### 실패 조건

이 문제에서는 사용자 입력 오류를 예외로 던지지 않고 상태를 바꾸지 않는 no-op으로 처리한다. 단, `version`이 음수인 손상된 입력처럼 함수 계약 자체가 깨진 상태를 받을 때의 정책은 구현 전에 명시한다.

#### 필요한 제약

- 시간 복잡도는 액션당 O(n) 이하여야 한다.
- 별도 인덱스나 전역 ID 생성기는 만들지 않는다.
- React hook이나 저장소 API를 호출하지 않는다.

### 구현 후 자가 검증

- 정상 경로: create → rename → toggle → delete 순서가 기대한 다섯 필드를 만든다.
- 경계값: 빈 제목, 같은 제목, 없는 ID가 목록과 메타데이터를 바꾸지 않는다.
- 상태 변화: 변경 대상 외 Item의 객체 값이 보존된다.
- invariant: 모든 Item의 ID가 안정적이고 제목이 비어 있지 않으며 version이 감소하지 않는다.
- 요구사항 충족 여부: `Date.now()`나 난수를 함수 내부에서 호출하지 않는다.
- 시간·공간 복잡도: 한 액션에서 불필요한 중첩 순회나 전체 깊은 복사를 하지 않는다.

### 구현 후 설명할 것

1. 유효하지 않은 액션을 예외 대신 no-op으로 선택했는지, 그 이유는 무엇인지
2. 배열 인덱스가 아니라 `id`를 상태·렌더링 키로 사용해야 하는 이유
3. `version`과 `updatedAt`을 도메인 상태에 둔 이유
4. reducer가 저장소 구현과 분리되어야 하는 이유
5. 목록이 매우 커졌을 때 O(n) reducer를 어떤 경계에서 다른 구조로 바꿀지

### 원본 확인 위치

- Thread: M01
- 커밋: `feat: establish in-memory React Native Item UI`
- 파일: `src/items.ts`, `src/App.tsx`, `__tests__/items.test.ts`, `__tests__/App.test.tsx`
- 함수·컴포넌트: `Item`, `itemsReducer`, `App`, `saveTitle`
- 관련 Thread: M02, M08

---

<a id="p02"></a>
## [M02 / 기록에서 정확한 커밋 메시지 미확인] Item 변경·ID allocator·UI 공개의 원자성

### 면접 질문

M02에서는 Item 변경과 `local_identity.next_id` 갱신을 SQLite 트랜잭션으로 묶고, React 상태는 네이티브 트랜잭션의 성공 콜백 뒤에만 갱신합니다. 왜 세 동작을 같은 정합성 경계로 봐야 합니까?

꼬리 질문:

- Item INSERT는 성공했지만 COMMIT이 실패하면 UI와 다음 ID는 어떤 상태여야 합니까?
- 삭제된 ID를 재사용하지 않기 위해 `MAX(id) + 1` 대신 별도 allocator를 둔 이유는 무엇입니까?
- ID를 먼저 메모리에서 증가시키고 나중에 저장하면 process death에서 어떤 문제가 생깁니까?
- 저장 성공 전에 낙관적으로 UI를 바꾸는 설계도 가능한가요? 가능하다면 어떤 보상 상태가 추가로 필요합니까?
- 두 개의 mutation이 동시에 들어올 수 있다면 `busyRef`만으로 충분합니까, 아니면 저장소도 직렬화를 보장해야 합니까?

### 30초 모범 답변

사용자가 본 Item, 디스크의 Item, 다음 ID는 하나의 논리 상태입니다. 생성 중 INSERT나 allocator UPDATE, 마지막 COMMIT 중 하나라도 실패하면 셋 모두 이전 상태여야 다음 재시도에서 ID가 건너뛰거나 중복되지 않습니다. 그래서 ID 예약과 Item 쓰기를 한 DB 트랜잭션에 두고, React 상태는 성공 콜백 이후 커밋된 값을 읽어 공개합니다. `MAX(id)+1`은 삭제·형식 변경·동시 실행에 취약하므로 별도 high-water mark가 낫습니다. 낙관적 UI도 가능하지만 pending/rollback 표시가 필요해 이 단계의 단순한 confirmed-state 모델보다 복잡합니다.

### 답변 핵심 키워드

atomic commit, high-water mark, ID 비재사용, write-ahead intent, commit callback, confirmed UI, rollback, process death, linearization point

### 백지 구현

#### 구현 목표

주어진 트랜잭션 API 위에서 Item mutation을 수행하는 저장소를 구현한다. 생성 시 Item INSERT와 다음 ID 증가가 반드시 함께 커밋되어야 하며, 호출자는 COMMIT이 끝난 뒤의 목록만 받아야 한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type ItemMutation =
  | {type: 'create'; title: string; now: number}
  | {type: 'rename'; id: string; title: string; now: number}
  | {type: 'toggle'; id: string; now: number}
  | {type: 'delete'; id: string};

export interface ItemTx {
  readItems(): Promise<readonly Item[]>;
  readNextId(): Promise<number>;
  writeNextId(next: number): Promise<void>;
  insertItem(item: Item): Promise<void>;
  updateItem(item: Item): Promise<void>;
  deleteItem(id: string): Promise<void>;
}

export interface TransactionalDatabase {
  transaction<T>(work: (tx: ItemTx) => Promise<T>): Promise<T>;
}

export class AtomicItemStore {
  constructor(private readonly db: TransactionalDatabase) {}

  async mutate(action: ItemMutation): Promise<readonly Item[]> {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 생성·이름 변경·토글·삭제 중 하나의 mutation
- 출력: 해당 트랜잭션이 성공적으로 COMMIT된 뒤의 전체 Item 목록
- 트랜잭션 API는 work 함수가 성공해도 실제 COMMIT 실패 시 Promise를 reject할 수 있다고 가정한다.

#### 반드시 만족해야 할 조건

- 생성 시 현재 `nextId`로 ID를 만들고, Item INSERT와 `nextId + 1` 저장을 한 트랜잭션에 둔다.
- 삭제 뒤에도 allocator를 되감지 않는다.
- rename·toggle은 현재 row를 기준으로 새 version과 `updatedAt`을 계산한다.
- 공백 생성·공백 rename·없는 ID 변경은 디스크와 allocator를 바꾸지 않는다.
- SQL 문 중간 실패와 COMMIT 실패 모두 호출자에게 실패로 보이고, 이전 데이터가 남아야 한다.
- 반환값은 COMMIT이 끝난 뒤에만 외부로 전달한다.
- 실패한 생성의 같은 입력을 다시 실행했을 때 원래 사용하려던 ID를 정상적으로 쓸 수 있어야 한다.

#### 경계 조건

- 첫 생성
- 여러 번 생성 후 중간 Item 삭제와 재시작
- INSERT 실패
- allocator UPDATE 실패
- UPDATE·DELETE 대상 없음
- 트랜잭션 본문은 끝났지만 COMMIT 실패
- 동시에 두 생성 요청이 들어오는 경우

#### 실패 조건

- 손상된 allocator 값, 중복 기본 키, DB I/O 오류, COMMIT 오류는 reject한다.
- 실패를 성공한 빈 변경으로 삼키지 않는다.
- UI 계층이 실패를 "저장됨"으로 표시할 수 있는 값은 반환하지 않는다.

#### 필요한 제약

- 전역 메모리 카운터를 두지 않는다.
- ID 형식은 `item-` 접두사와 고정 폭 숫자처럼 문제에서 정한 규칙을 사용하되, 형식과 allocator 의미를 분리한다.
- DB가 트랜잭션을 직렬화하지 않는다고 가정한다면 필요한 동시성 제약을 인터페이스 계약에 명시한다.
- 원본 SQL을 복사하지 말고 제공된 추상 인터페이스만 사용한다.

### 구현 후 자가 검증

- 정상 경로: 생성·수정·토글·삭제·재시작 뒤 다섯 필드와 다음 ID가 보존된다.
- 실패 경로: 각 쓰기와 COMMIT 위치에 실패를 주입해 Item과 allocator가 함께 롤백된다.
- 상태 변화: 성공 전에는 외부 confirmed state가 바뀌지 않는다.
- invariant: 존재했던 ID를 삭제·재시작 뒤 다시 발급하지 않는다.
- 중복·누락 처리: 동시에 들어온 두 생성이 같은 ID를 받지 않는다.
- resource cleanup: 실패 뒤 트랜잭션과 statement 자원이 열린 채 남지 않는다는 DB 계약을 확인한다.
- 요구사항 충족 여부: 성공 반환 시점이 실제 COMMIT 이후인지 확인한다.

### 구현 후 설명할 것

1. 논리적 linearization point를 COMMIT 성공으로 둔 이유
2. allocator row를 Item INSERT와 같은 트랜잭션에 둔 이유
3. `MAX(id)+1`, UUID, 별도 high-water mark의 trade-off
4. confirmed UI와 optimistic UI 중 어떤 모델을 택했는지
5. UI의 `busyRef`와 DB 수준 직렬화가 각각 막는 경쟁 조건

### 원본 확인 위치

- Thread: M02
- 커밋: 기록에서 정확한 메시지 미확인
- 파일: `src/itemStore.ts`, `src/App.tsx`, `__tests__/items.test.ts`, `__tests__/App.test.tsx`, `__tests__/sqliteNative.ts`, `scripts/verify_m02.py`
- 함수·클래스·컴포넌트: `ItemStore`, `SqliteItemStore`, `openItemStore`, `mutate`, `itemToRow`, `rowToItem`, `App`, `failNextSql`
- 관련 Thread: M04, M05, M06

---

<a id="p03"></a>
## [M02 / 기록에서 정확한 커밋 메시지 미확인] 비파괴 스키마 마이그레이션

### 면접 질문

M02는 새 DB 생성과 미지원 스키마 거부를 도입했고, M04~M07은 `sync_metadata`, `pending_mutations`, mutation identity, conflict/version 테이블을 순차적으로 추가했습니다. 마이그레이션 실패가 기존 데이터와 `user_version`을 훼손하지 않게 하려면 어떤 순서와 불변식이 필요합니까?

꼬리 질문:

- `PRAGMA user_version`을 먼저 올리고 테이블을 만든다면 어떤 복구 불가능 상태가 생깁니까?
- 최신 버전보다 높은 DB를 자동으로 삭제·재생성하지 않은 이유는 무엇입니까?
- 각 과거 버전의 literal schema를 테스트하는 이유는 무엇입니까?
- 긴 마이그레이션에서 하나의 큰 트랜잭션과 단계별 트랜잭션 중 무엇을 택하겠습니까?
- 앱 코드가 여러 버전을 건너뛰어 업그레이드할 때 어떤 테스트 행렬이 필요합니까?

### 30초 모범 답변

마이그레이션의 핵심 불변식은 성공하면 새 스키마와 기존 데이터가 함께 보이고, 실패하면 이전 버전 그대로 다시 열 수 있어야 한다는 것입니다. 따라서 지원 여부를 쓰기 전에 판단하고, 테이블·컬럼·백필과 `user_version` 갱신을 같은 트랜잭션의 마지막에 둡니다. 더 높은 미지원 버전은 앱이 의미를 모르므로 삭제하지 않고 명시적으로 거부합니다. 테스트는 현재 ORM 모델로 만든 DB가 아니라 실제 과거 DDL과 데이터를 seed해 중간 statement와 COMMIT 실패를 주입해야, 누락된 컬럼·allocator·queue 순서를 검증할 수 있습니다.

### 답변 핵심 키워드

forward-only migration, atomic schema change, user_version last, literal old schema, non-destructive rejection, reopenability, backfill, high-water mark 보존

### 백지 구현

#### 구현 목표

버전 0부터 목표 버전까지 순차적으로 올리는 마이그레이션 실행기를 작성한다. 각 단계는 기존 row와 메타데이터를 보존하며, 실패 시 열기 전 상태로 돌아가야 한다.

#### 인터페이스 또는 함수 시그니처

```ts
export interface MigrationTx {
  execute(statement: string, params?: readonly unknown[]): Promise<void>;
}

export interface SchemaDatabase {
  readVersion(): Promise<number>;
  transaction(work: (tx: MigrationTx) => Promise<void>): Promise<void>;
}

export type Migration = {
  from: number;
  to: number;
  run(tx: MigrationTx): Promise<void>;
};

export async function ensureSchema(
  db: SchemaDatabase,
  targetVersion: number,
  migrations: readonly Migration[],
): Promise<void> {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 현재 버전을 읽을 수 있는 DB, 목표 버전, 단일 단계 마이그레이션 목록
- 출력: 성공 시 `void`; 지원하지 않는 버전·경로 누락·DB 오류 시 reject

#### 반드시 만족해야 할 조건

- 음수 버전과 목표보다 높은 현재 버전은 어떤 write도 하기 전에 거부한다.
- 버전 0은 명시적인 초기 스키마 단계로 처리한다.
- `from → to`는 한 단계씩 연결되어야 하며 중간 경로가 빠지면 실패한다.
- 각 단계의 DDL·데이터 백필·버전 갱신은 하나의 트랜잭션 안에서 수행한다.
- 버전 값은 해당 단계의 모든 작업이 성공한 뒤 마지막에 갱신한다.
- 실패 뒤 재호출하면 같은 이전 버전에서 정상적으로 다시 시작할 수 있어야 한다.
- 기존 Item 필드, `local_identity.next_id`, refresh 시각, pending 순서처럼 문제에서 지정한 보존 대상이 유지되어야 한다.

#### 경계 조건

- 새 DB 버전 0
- 이미 목표 버전인 DB
- 목표보다 높은 DB
- migration 목록 순서가 뒤섞인 경우
- 한 단계가 누락된 경우
- 테이블 생성·백필·인덱스 생성·버전 기록 각각의 실패
- 여러 버전을 한 번에 건너뛰는 업그레이드

#### 실패 조건

- 자동 삭제, 빈 DB 재생성, 알 수 없는 버전의 강제 downgrade는 금지한다.
- migration 함수가 성공했더라도 COMMIT이 실패하면 버전이 오른 것으로 취급하지 않는다.
- 중복 실행으로 데이터가 두 번 삽입되거나 queue sequence가 바뀌면 실패다.

#### 필요한 제약

- 문자열 SQL의 구체 내용은 문제에서 제공된 migration 객체 안에 있다고 가정한다.
- 실행기는 migration을 추측하거나 스키마를 introspection해 자동 수리하지 않는다.
- 메모리 사용량은 데이터 전체 크기에 비례하지 않아야 한다. 대량 백필이 필요하면 별도 운영 전략을 설명하되 이 구현에는 넣지 않는다.

### 구현 후 자가 검증

- 정상 경로: 0→목표, 중간 버전→목표, 이미 최신인 경우를 확인한다.
- 실패 경로: 각 단계의 첫 statement·중간 statement·버전 갱신·COMMIT 실패 뒤 이전 DB를 다시 연다.
- invariant: 실패 시 데이터와 버전이 같은 세대에 남는다.
- 중복·누락 처리: 경로 중복과 누락을 실행 전에 발견한다.
- 상태 변화: 성공한 단계까지만 남기는 정책인지 전체 업그레이드를 한 번에 롤백하는 정책인지 계약과 구현이 일치한다.
- 시간·공간 복잡도: migration 탐색이 불필요한 중첩 검색이 되지 않는다.
- 요구사항 충족 여부: 더 높은 스키마를 비파괴적으로 거부한다.

### 구현 후 설명할 것

1. `user_version`을 마지막에 기록해야 하는 이유
2. 단계별 트랜잭션과 전체 업그레이드 단일 트랜잭션의 trade-off
3. literal 과거 스키마 fixture가 필요한 이유
4. 미지원 미래 버전을 삭제하지 않는 이유
5. 큰 데이터 백필을 앱 시작 경로에서 처리할 때의 위험과 완화책

### 원본 확인 위치

- 대표 Thread: M02
- 대표 커밋: 기록에서 정확한 메시지 미확인
- 파일: `src/itemStore.ts`, `__tests__/items.test.ts`, `__tests__/sqliteNative.ts`
- 함수·상수: `SCHEMA_VERSION`, `openItemStore`, `failNextSql`
- 관련 Thread와 확인된 스키마 요소:
  - M04: `sync_metadata`, `readLastSuccessfulRefresh`
  - M05: `pending_mutations`, `PendingMutation`
  - M06: `client_mutation_id`, `payload_hash`, `terminal_error`
  - M07: `remote_versions`, `mutation_conflicts`, `MutationConflict`
