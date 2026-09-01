# 대규모 로컬 상태와 Release 검증

이 문서는 M15의 bounded list loading을 하나의 종합 문제로 다룬다. 핵심은 `FlatList` 사용 자체가 아니라 DB에서 JS로 materialize하는 row 수, snapshot 정합성, 늦은 page 응답, off-page editor 상태, release 빌드의 실제 자원 해제를 함께 관리하는 것이다.

---

<a id="p14"></a>
## [M15 / `feat(react-native): bound M15 release list loading`] bounded pagination, latest-only publish, release 자원 검증

### 면접 질문

M15는 `ITEM_PAGE_SIZE = 50`으로 `ItemStore.readPage`를 추가하고, 같은 SQLite transaction에서 `COUNT(*)`와 `ORDER BY rowid LIMIT ? OFFSET ?`를 실행합니다. UI는 전체 목록을 읽지 않고 `FlatList`에 현재 페이지만 전달하며, `pageRequest`로 늦은 응답을 버립니다. 이 설계가 해결하는 성능·정합성 문제를 설명해 보세요.

꼬리 질문:

- `FlatList`만 도입하고 저장소가 2,000개 row를 전부 JS 객체로 만들면 문제가 해결됩니까?
- COUNT와 page SELECT를 다른 transaction에서 실행하면 삭제·추가가 겹칠 때 어떤 불일치가 생깁니까?
- 음수 index, 매우 큰 index, 빈 DB, 마지막 페이지 삭제 뒤 index를 어떻게 처리해야 합니까?
- `ORDER BY rowid`를 택한 의미와, 장기적으로 keyset pagination이 더 나을 수 있는 이유는 무엇입니까?
- 사용자가 페이지 2를 눌렀다가 곧바로 페이지 1을 눌렀는데 페이지 2 응답이 늦게 도착하면 어떻게 막습니까?
- 현재 페이지 밖 Item을 편집 중인 draft가 background reload나 page 이동으로 사라지면 어떤 소유권 오류입니까?
- debug가 아닌 release 경로에서 cursor close와 materialized row 수를 검증한 이유는 무엇입니까?
- 성능 계측을 넣는 것 자체가 결과를 왜곡하지 않도록 어떤 범위를 제한해야 합니까?

### 30초 모범 답변

가상화는 렌더링 비용만 줄이므로 DB가 전체 row를 JS로 변환하면 시작 메모리와 bridge 비용은 그대로입니다. M15는 저장소부터 50개로 제한하고, 같은 read transaction에서 total과 page를 읽어 한 snapshot의 index를 계산합니다. index는 안전한 정수로 검증하고 유효 범위로 clamp합니다. UI는 request generation을 증가시켜 가장 최근 page 요청만 publish하고, editor owner는 페이지 데이터와 분리해 off-page draft를 유지합니다. Release 검증에서는 실제 non-debug build에서 materialized row 수와 cursor 해제를 관찰해 host mock이나 debug 전용 동작이 성능 주장을 대신하지 않게 했습니다.

### 답변 핵심 키워드

bounded materialization, database-level pagination, consistent read snapshot, COUNT + LIMIT/OFFSET, clamped index, latest-only publish, FlatList, off-page state ownership, release resource validation

### 백지 구현

#### 구현 목표

고정 크기의 SQLite page를 읽는 저장소 함수와, 여러 page 요청이 겹쳐도 가장 최근 요청만 UI에 publish하는 loader를 구현한다. 실제 SQL driver는 제공된 추상 인터페이스로 대체한다.

#### 인터페이스 또는 함수 시그니처

```ts
export const PAGE_SIZE = 50;

export type ItemPage = {
  items: readonly Item[];
  index: number;
  total: number;
};

export interface PageReadTx {
  countItems(): Promise<number>;
  readItems(limit: number, offset: number): Promise<readonly Item[]>;
}

export interface PageDatabase {
  readTransaction<T>(work: (tx: PageReadTx) => Promise<T>): Promise<T>;
}

export async function readBoundedPage(
  db: PageDatabase,
  requestedIndex = 0,
): Promise<ItemPage> {
  // 직접 구현
}

export interface PagePublisher {
  isMounted(): boolean;
  publish(page: ItemPage): void;
  publishError(message: string): void;
}

export class LatestPageLoader {
  constructor(
    private readonly readPage: (index: number) => Promise<ItemPage>,
    private readonly publisher: PagePublisher,
  ) {}

  async load(index: number): Promise<void> {
    // 직접 구현
  }
}
```

#### 입력과 출력

- `readBoundedPage` 입력: 요청 페이지 index
- 출력: 최대 50개의 Item, 실제 clamp된 index, 전체 row 수
- `LatestPageLoader.load` 입력: 사용자가 보고자 하는 index
- 출력: Promise. 가장 최근 요청이고 화면이 살아 있을 때만 page 또는 오류를 publish한다.

#### 반드시 만족해야 할 조건

- `requestedIndex`가 safe integer가 아니면 reject한다.
- 음수 safe integer는 첫 페이지로 clamp한다.
- total이 0이면 `{items: [], index: 0, total: 0}` 형태의 유효한 빈 페이지를 반환한다.
- total이 양수면 마지막 유효 index를 계산해 너무 큰 요청을 clamp한다.
- COUNT와 page SELECT는 같은 read transaction에서 실행한다.
- page SELECT는 `limit = PAGE_SIZE`, `offset = index * PAGE_SIZE`를 사용한다.
- 정렬 기준은 삽입 순서를 나타내는 안정적인 저장소 기준이라고 계약한다. 원본에서는 `ORDER BY rowid`가 사용됐다.
- 반환 Item 수는 PAGE_SIZE를 넘지 않는다.
- 전체 Item 목록을 읽는 API를 호출하지 않는다.
- loader는 각 요청에 증가하는 generation을 부여하고, 가장 최신 generation만 publish한다.
- unmount 뒤에는 성공·실패 결과를 publish하지 않는다.
- 페이지 데이터와 editor draft의 소유권을 분리해, 현재 page 밖 Item의 draft가 page reload 때문에 초기화되지 않게 한다.

#### 경계 조건

- total 0, 1, 49, 50, 51
- 정확히 마지막 페이지
- 음수 index
- 매우 큰 index
- `Number.MAX_SAFE_INTEGER`와 safe integer가 아닌 값
- 마지막 페이지의 마지막 Item 삭제 뒤 같은 index 재요청
- 앞 page 요청이 느리고 뒤 요청이 빠름
- 뒤 요청이 실패하고 앞 요청이 늦게 성공
- load 도중 unmount
- background event와 사용자의 page 이동이 거의 동시에 발생

#### 실패 조건

- 전체 row를 먼저 읽고 배열을 잘라 page를 만들기
- COUNT와 SELECT 사이에 다른 snapshot을 보아 `index`, `total`, `items`가 모순됨
- 오래된 요청 결과가 최신 화면을 덮어쓰기
- page 오류가 editor draft나 durable pending queue를 초기화
- empty DB에서 음수 last index 계산
- cursor/transaction 자원이 실패 경로에서 해제되지 않음

#### 필요한 제약

- 저장소는 read transaction 종료 시 cursor와 statement를 닫는다고 계약하거나 명시적인 cleanup을 제공해야 한다.
- loader는 요청을 실제로 취소하지 않아도 되지만 stale publish를 반드시 막아야 한다. Abort를 추가한다면 P13의 owner 규칙을 재사용한다.
- OFFSET 비용이 매우 큰 데이터에서 증가할 수 있음을 인정한다. 이 문제에서는 50개 fixed page와 기록의 로컬 목록 규모를 대상으로 한다.
- 정렬 기준 변경이나 keyset cursor 설계는 구현 후 trade-off로 설명한다.

### 구현 후 자가 검증

- 정상 경로: 51개 입력에서 첫 요청은 50개, 큰 index 요청은 마지막 1개를 반환한다.
- 경계값: 빈 DB와 음수 index가 page 0을 반환한다.
- 실패 경로: count/read 실패가 partial page를 publish하지 않는다.
- 상태 변화: 마지막 page 삭제 후 다시 읽으면 유효한 이전/현재 마지막 index로 clamp된다.
- 동시성·비동기: 요청 A 후 B를 시작하고 B가 먼저 끝났을 때 A가 publish되지 않는다.
- lifecycle: unmount 뒤 page와 오류 모두 publish되지 않는다.
- invariant: `items.length <= PAGE_SIZE`, `0 <= index <= lastIndex`, `total >= items.length`가 유지된다.
- resource cleanup: 성공·실패·stale 결과에서 transaction/cursor 해제가 보장된다.
- 시간·공간 복잡도: JS materialization 공간이 O(PAGE_SIZE)인지 확인한다.
- 요구사항 충족 여부: full-read API가 호출되지 않는다.

### 구현 후 설명할 것

1. FlatList와 DB-level pagination이 각각 줄이는 비용
2. COUNT와 page query를 같은 transaction에 둔 이유
3. OFFSET pagination을 택한 이유와 keyset pagination으로 바꿀 기준
4. request generation을 이용한 latest-only publish와 실제 취소의 차이
5. release 빌드에서 row materialization·cursor close를 검증한 이유

### 원본 확인 위치

- Thread: M15
- 커밋: `feat(react-native): bound M15 release list loading`
- 파일: `src/itemStore.ts`, `src/App.tsx`, `__tests__/items.test.ts`, `__tests__/App.test.tsx`, `android/app/build.gradle`, `android/app/src/androidTest/assets/m15-inputs.json`, `verification/phase-1/M15.md`
- 타입·상수·함수·상태: `ITEM_PAGE_SIZE`, `ItemPage`, `ItemStore.readPage`, `SqliteItemStore.readPage`, `FlatList`, `pageIndex`, `pageRequest`, `reloadPage`, `MSESqlRows`
- 관련 Thread: M08, M10, M14
