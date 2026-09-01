# 동기화, 내구성 큐, 멱등성

이 문서는 M03의 명시적 동기화 기준선에서 시작해 M05의 transactional outbox, M06의 멱등성 envelope와 ACK 원자성까지 이어진다. 네 문제는 서로 비슷해 보이지만 각각 막는 실패가 다르다.

---

<a id="p04"></a>
## [M03 / `feat: add explicit foreground synchronization`] 기준 스냅샷 기반 명시적 동기화

### 면접 질문

`ForegroundSync.synchronize`는 첫 동기화에서 원격 snapshot을 가져오고, 이후에는 저장해 둔 baseline과 현재 로컬 상태를 비교해 POST·PATCH·DELETE를 보낸 뒤 마지막 GET snapshot을 SQLite에 교체 저장합니다. 이 흐름에서 baseline, 서버 응답, 커밋된 로컬 저장소가 각각 어떤 역할을 합니까?

꼬리 질문:

- 첫 동기화에서 아직 baseline이 없을 때 로컬 항목을 바로 업로드하지 않은 이유는 무엇입니까?
- PATCH에는 왜 변경된 필드만 넣는 편이 낫습니까?
- 요청 중간에 하나가 실패하면 baseline을 갱신하면 안 되는 이유는 무엇입니까?
- GET 응답 배열의 항목 shape와 중복 ID를 검증해야 하는 이유는 무엇입니까?
- UI가 마지막 HTTP 응답을 바로 렌더링하지 않고 `store.read()` 결과를 사용한 이유는 무엇입니까?
- 두 로컬 DB 인스턴스가 같은 원격으로 수렴하는 것과 서로의 로컬 변경을 즉시 관찰하는 것은 왜 다른 보장입니까?

### 30초 모범 답변

Baseline은 "마지막으로 로컬이 원격과 같다고 확인한 상태"라서 현재 로컬과의 차이를 요청으로 바꾸는 기준입니다. 첫 실행에는 그런 기준이 없으므로 먼저 원격을 가져와 시작점을 정합니다. 업로드가 모두 끝나면 다시 GET해 서버가 정규화한 canonical snapshot을 검증하고 한 트랜잭션으로 저장합니다. 중간 실패에서는 baseline을 전진시키지 않아야 다음 시도에서 누락이 없습니다. UI는 응답 객체가 아니라 커밋된 SQLite를 다시 읽어야 화면, 재시작 후 데이터, 다음 baseline이 같은 source of truth를 공유합니다.

### 답변 핵심 키워드

baseline, three-way boundary, deterministic diff, minimal PATCH, final canonical GET, validation before commit, source of truth, commit-then-publish

### 백지 구현

#### 구현 목표

마지막으로 확인한 baseline과 현재 로컬 목록으로 업로드 작업을 계획하고, 원격 GET 응답을 안전한 Item snapshot으로 검증하는 두 함수를 구현한다. 실제 HTTP와 SQLite 코드는 제공된 coordinator가 담당한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type SyncOperation =
  | {method: 'POST'; path: '/items'; body: {id: string; title: string; completed: boolean}}
  | {method: 'PATCH'; path: string; body: {title?: string; completed?: boolean}}
  | {method: 'DELETE'; path: string; body?: undefined};

export function planUploads(
  baseline: readonly Item[] | null,
  local: readonly Item[],
): readonly SyncOperation[] {
  // 직접 구현
}

export function validateRemoteSnapshot(value: unknown): readonly Item[] {
  // 직접 구현
}
```

#### 입력과 출력

- `baseline === null`: 아직 원격 기준선을 확립하지 않은 세션
- `local`: 현재 커밋된 로컬 Item 목록
- `planUploads` 출력: 실행 순서가 결정된 HTTP 작업 목록
- `validateRemoteSnapshot` 입력: 파싱된 JSON의 `items` 값이라고 가정하지 않고 전체 unknown 값
- `validateRemoteSnapshot` 출력: 다섯 필드가 검증되고 ID가 중복되지 않은 Item 배열

#### 반드시 만족해야 할 조건

- baseline이 없으면 업로드 작업을 만들지 않는다.
- baseline에 없고 local에 있는 ID는 POST 대상이다.
- 두 목록에 모두 있는 ID는 `title`, `completed`의 실제 차이만 PATCH body에 넣는다.
- baseline에는 있고 local에는 없는 ID는 DELETE 대상이다.
- `version`, `updatedAt`은 이 축소 문제의 업로드 body에 넣지 않는다.
- 작업 순서는 입력 배열 우연에 의존하지 않고 ID 기준으로 결정적이어야 한다.
- snapshot의 각 Item은 `id`, `title`, `completed`, `version`, `updatedAt` 타입과 값 범위를 검증한다.
- 중복 ID가 하나라도 있으면 전체 snapshot을 거부한다.
- 입력 객체와 배열을 수정하지 않는다.

#### 경계 조건

- baseline과 local이 모두 비어 있음
- 첫 동기화
- 새 항목만 있음
- 삭제만 있음
- 제목만 변경, 완료 여부만 변경, 둘 다 변경
- 같은 값이라 작업이 필요 없는 항목
- URL encoding이 필요한 ID
- 빈 배열, 잘못된 필드, 중복 ID, 매우 큰 version, 음수 timestamp

#### 실패 조건

- 잘못된 원격 snapshot은 일부만 채택하지 않고 전체를 거부한다.
- PATCH body가 비어 있으면 요청을 만들지 않는다.
- 중복 ID나 잘못된 Item을 조용히 덮어쓰지 않는다.

#### 필요한 제약

- 시간 복잡도는 O(n log n) 이하를 목표로 한다.
- 요청 실행, retry, queue, 충돌 해결은 이 문제 범위가 아니다.
- coordinator는 모든 operation 성공 뒤 GET → validate → `replaceSnapshot` → `read` 순서를 지킨다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: create, rename, toggle, delete가 각각 정확한 작업으로 변환된다.
- 경계값: 첫 동기화와 no-op diff에서 업로드가 0개다.
- 중복·누락 처리: 같은 ID를 map으로 덮어써 숨기지 않고 명시적으로 발견한다.
- 실패 경로: snapshot 하나의 필드가 잘못되면 커밋 후보 전체가 거부된다.
- invariant: 동일 입력은 동일한 작업 순서를 만든다.
- 시간·공간 복잡도: 항목마다 전체 배열을 다시 검색하는 O(n²) 구현인지 확인한다.
- 요구사항 충족 여부: 서비스 응답을 직접 UI 상태로 반환하지 않는다.

### 구현 후 설명할 것

1. baseline이 필요한 이유와 첫 동기화 정책
2. 전체 PUT 대신 최소 PATCH를 만든 이유
3. 마지막 GET을 생략했을 때 놓치는 서버 canonicalization
4. 요청 성공과 snapshot COMMIT 사이의 실패를 어떻게 다룰지
5. 배열 기반 diff와 Map 기반 diff의 복잡도·메모리 trade-off

### 원본 확인 위치

- Thread: M03
- 커밋: `feat: add explicit foreground synchronization`
- 파일: `src/sync.ts`, `src/itemStore.ts`, `src/App.tsx`, `__tests__/sync.test.ts`, `fixture/server.cjs`, `scripts/verify_m03.py`
- 함수·클래스: `JsonRequest`, `SyncSession`, `ForegroundSync`, `exchange`, `synchronize`, `replaceSnapshot`, `remoteItem`, `sessionIdentity`
- 관련 Thread: M04, M05

---

<a id="p05"></a>
## [M05 / 기록에서 정확한 커밋 메시지 미확인] transactional outbox와 FIFO drain

### 면접 질문

M05는 로컬 Item 변경과 `pending_mutations` 추가를 같은 SQLite 트랜잭션에 넣고, 명시적 동기화가 queue head부터 하나씩 업로드합니다. 왜 로컬 변경과 업로드 의도를 분리해서 커밋하면 안 되며, 왜 queue를 임의로 합치지 않고 FIFO로 drain했습니까?

꼬리 질문:

- Item은 저장됐지만 queue INSERT가 실패하면 어떤 사용자-visible 문제가 생깁니까?
- queue는 저장됐지만 Item 변경이 롤백되면 어떤 잘못된 원격 변경이 생깁니까?
- 첫 요청이 네트워크 오류로 실패했을 때 뒤 요청을 계속 보내면 어떤 순서 문제가 생깁니까?
- 같은 Item의 rename 두 개를 하나로 coalesce하면 어떤 의미를 잃을 수 있습니까?
- pending이 남은 상태에서 원격 snapshot으로 로컬 Items를 통째로 교체하면 왜 안 됩니까?
- 이 설계만으로 "서버는 적용했지만 ACK를 잃은 경우"까지 안전합니까?

### 30초 모범 답변

로컬 상태와 outbox는 하나의 사용자 의도를 두 방식으로 표현하므로 반드시 함께 커밋해야 합니다. 둘 중 하나만 남으면 화면에는 반영됐지만 영원히 업로드되지 않거나, 화면에 없는 변경을 서버로 보내게 됩니다. Drain은 head 하나를 성공시킨 뒤에만 다음으로 넘어가야 create 뒤 rename·delete 같은 의존 순서를 지킬 수 있습니다. M05는 의도 보존을 우선해 coalescing하지 않았고, 실패하면 head와 뒤 항목을 모두 남깁니다. 다만 서버 적용 뒤 응답 유실은 같은 요청을 재전송할 수 있으므로 M06의 멱등성 키가 추가로 필요합니다.

### 답변 핵심 키워드

transactional outbox, atomic local+intent, durable FIFO, head-of-line ordering, no coalescing, stop on failure, pending snapshot barrier, at-least-once gap

### 백지 구현

#### 구현 목표

로컬 mutation과 업로드 intent를 원자적으로 저장하는 메서드와, queue를 한 번만 앞에서부터 비우는 foreground drain 함수를 구현한다. 멱등성 키와 lost-ACK 복구는 다음 문제 범위로 남긴다.

#### 인터페이스 또는 함수 시그니처

```ts
export type PendingMutation = {
  sequence: number;
  kind: 'create' | 'rename' | 'toggle' | 'delete';
  itemId: string;
  payload: unknown;
};

export interface OutboxTx {
  applyLocal(action: ItemMutation): Promise<readonly Item[]>;
  appendPending(input: Omit<PendingMutation, 'sequence'>): Promise<PendingMutation>;
}

export interface OutboxDatabase {
  transaction<T>(work: (tx: OutboxTx) => Promise<T>): Promise<T>;
  readPending(): Promise<readonly PendingMutation[]>;
  removeHead(sequence: number): Promise<void>;
}

export class DurableOutboxStore {
  constructor(private readonly db: OutboxDatabase) {}

  async commitMutation(
    action: ItemMutation,
    intent: Omit<PendingMutation, 'sequence'>,
  ): Promise<{items: readonly Item[]; queued: PendingMutation}> {
    // 직접 구현
  }
}

export async function drainPendingOnce(
  db: OutboxDatabase,
  send: (operation: PendingMutation) => Promise<void>,
): Promise<{sent: number; remaining: number}> {
  // 직접 구현
}
```

#### 입력과 출력

- `commitMutation`: 하나의 로컬 액션과 그 액션에 대응하는 wire intent
- 출력: COMMIT된 Item 목록과 새 queue row
- `drainPendingOnce`: 현재 저장된 queue와 한 번의 전송 함수
- 출력: 이번 호출에서 성공적으로 dequeue한 개수와 남은 개수

#### 반드시 만족해야 할 조건

- 로컬 변경과 queue append는 같은 트랜잭션에 들어간다.
- 유효하지 않은 no-op 액션은 queue row를 만들지 않는다.
- `sequence`는 내구성 있고 단조 증가하며 재시작 뒤 순서를 보존한다.
- drain은 가장 작은 sequence부터 한 번에 하나만 보낸다.
- `send`가 성공한 head만 제거한다.
- 전송 또는 dequeue가 실패하면 즉시 중단하고 뒤 항목을 건드리지 않는다.
- queue 내용을 재정렬하거나 같은 Item의 연속 작업을 합치지 않는다.
- pending이 하나라도 있을 때 full snapshot replacement를 허용할지 여부를 저장소 계약에 명시하며, 이 문제에서는 거부한다.

#### 경계 조건

- 빈 queue
- create → rename → toggle → delete가 같은 Item에 연속됨
- 첫 head 실패
- 중간 head 실패
- 로컬 statement 실패
- queue append 실패
- COMMIT 실패
- 성공 직후 앱 프로세스 종료

#### 실패 조건

- 로컬 변경만 남거나 queue만 남는 부분 성공
- 실패한 head를 제거하고 뒤 요청을 계속하는 동작
- 재시작 뒤 sequence 순서 변경
- pending이 있는데 remote snapshot으로 현재 로컬 optimistic state를 덮어쓰기

#### 필요한 제약

- 자동 retry, timer, background scheduler를 만들지 않는다.
- `send`가 성공을 반환한 뒤 응답을 잃는 문제는 범위 밖이며 P06·P07에서 해결한다.
- drain 중 queue 전체를 메모리에서 반복 복사하지 않아도 되지만, 축소 구현에서는 명확성을 우선할 수 있다.

### 구현 후 자가 검증

- 정상 경로: 네 종류의 intent가 생성 순서대로 전송되고 0개가 남는다.
- 실패 경로: local write, queue append, COMMIT, 첫/중간 send, dequeue 실패를 각각 주입한다.
- 상태 변화: 로컬 Items와 pending count가 같은 COMMIT 세대를 가리킨다.
- invariant: queue sequence가 엄격히 증가하고 drain은 head 외 row를 먼저 제거하지 않는다.
- 중복·누락 처리: no-op 액션이 intent를 만들지 않고 실제 액션은 정확히 하나를 만든다.
- process recovery: DB를 닫고 다시 열어도 Items와 queue가 동일하다.
- 요구사항 충족 여부: 실패 뒤 수동 재시도 가능 상태가 남아 있다.

### 구현 후 설명할 것

1. transactional outbox가 해결하는 dual-write 실패
2. FIFO와 head-of-line blocking을 택한 이유
3. coalescing으로 얻는 이점과 잃는 감사 가능성·도메인 의미
4. pending 중 full snapshot replacement를 막은 이유
5. M05만으로 lost ACK가 해결되지 않는 이유

### 원본 확인 위치

- Thread: M05
- 커밋: 기록에서 정확한 메시지 미확인
- 파일: `src/itemStore.ts`, `src/sync.ts`, `src/App.tsx`, `__tests__/items.test.ts`, `__tests__/sync.test.ts`, `scripts/verify_m05.py`, `verification/M05-inputs.json`
- 타입·메서드: `PendingMutation`, `ItemStore.readPending`, `ForegroundSync.synchronize`
- 관련 Thread: M02, M06, M09, M10

---

<a id="p06"></a>
## [M06 / `feat(M06): persist mutation identity and atomically acknowledge replay`] 멱등성 envelope와 canonical payload hash

### 면접 질문

M06는 각 pending mutation에 `clientMutationId`와 `payloadHash`를 내구성 있게 저장하고, `{method, path, payload}`를 canonical JSON으로 만든 뒤 SHA-256을 계산합니다. 왜 ID만으로는 부족하고, 왜 hash 계산 전에 canonicalization이 필요합니까?

꼬리 질문:

- 같은 `clientMutationId`와 같은 hash가 다시 오면 서버는 무엇을 반환해야 합니까?
- 같은 ID인데 다른 hash라면 단순한 중복으로 처리하면 왜 위험합니까?
- 객체 key 순서, 배열 순서, 숫자처럼 보이는 key, Unicode와 escape는 canonicalization에서 어떻게 다뤄야 합니까?
- 서버는 클라이언트가 선언한 hash만 믿어도 됩니까?
- SHA-256 hash가 있다고 요청 인증이나 위변조 방지가 해결됩니까?
- ID와 hash를 전송 시점에 새로 만들지 않고 local mutation과 함께 저장한 이유는 무엇입니까?

### 30초 모범 답변

Mutation ID는 재시도되는 논리 요청을 찾는 키이고, hash는 그 키가 같은 내용에만 재사용됐는지 확인하는 값입니다. 같은 ID·같은 hash면 서버는 side effect를 다시 만들지 않고 최초 status와 body를 반환해야 합니다. 같은 ID·다른 hash는 충돌이므로 terminal 오류로 남겨 자동 재시도를 막습니다. 객체 key 순서가 다른 같은 JSON이 같은 hash를 가져야 하므로 모든 object level을 정렬하고 배열 순서·UTF-8·escape는 보존합니다. 서버는 실제 method·path·body로 hash를 재계산해야 하며, 이 hash는 인증이 아니라 동일성 검증입니다.

### 답변 핵심 키워드

idempotency key, content binding, canonical JSON, recursive key sort, array order, UTF-8 bytes, server recomputation, duplicate replay, identity conflict, not authentication

### 백지 구현

#### 구현 목표

JSON 값의 결정적 직렬화, mutation envelope 생성, 서버 측 중복 registry의 핵심 전이를 구현한다. 암호 해시는 주어진 `sha256Utf8` 함수로 계산한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type JsonValue =
  | null
  | boolean
  | number
  | string
  | readonly JsonValue[]
  | {[key: string]: JsonValue};

export function canonicalJson(value: JsonValue): string {
  // 직접 구현
}

export type MutationEnvelope = {
  clientMutationId: string;
  payloadHash: string;
  method: 'POST' | 'PATCH' | 'DELETE';
  path: string;
  payload: JsonValue;
};

export function buildEnvelope(
  clientMutationId: string,
  method: MutationEnvelope['method'],
  path: string,
  payload: JsonValue,
  sha256Utf8: (text: string) => string,
): MutationEnvelope {
  // 직접 구현
}

export type StoredResult = {status: number; body: JsonValue; payloadHash: string};

export class IdempotencyRegistry {
  async handle(
    envelope: MutationEnvelope,
    applyOnce: () => Promise<{status: number; body: JsonValue}>,
  ): Promise<{kind: 'applied' | 'duplicate' | 'identity_conflict'; status: number; body: JsonValue}> {
    // 직접 구현
  }
}
```

#### 입력과 출력

- canonicalization 입력은 JSON으로 표현 가능한 값만 허용한다.
- `buildEnvelope`는 method, path, payload를 hash에 결합한 내구성 envelope를 반환한다.
- registry는 최초 적용, 동일 재전송, 동일 ID의 다른 요청을 구분한다.

#### 반드시 만족해야 할 조건

- 객체 key는 모든 중첩 level에서 코드 포인트 기준의 결정적인 순서로 정렬한다.
- 배열 element 순서는 바꾸지 않는다.
- 불필요한 공백 없는 JSON 문자열을 만든다.
- 문자열의 Unicode와 JSON escape 결과가 표준 JSON 의미를 유지한다.
- 숫자처럼 보이는 key도 런타임의 기본 object 열거 순서에 맡기지 않는다.
- hash 대상은 compact canonical `{method, path, payload}` 전체다.
- 서버는 envelope의 선언 hash와 실제 method·path·payload로 재계산한 hash가 같은지 확인한다.
- 최초 요청만 `applyOnce`를 실행한다.
- 같은 ID·같은 hash는 저장된 최초 status/body를 그대로 돌려준다.
- 같은 ID·다른 hash는 side effect 없이 `identity_conflict`를 반환한다.

#### 경계 조건

- 빈 객체와 빈 배열
- 여러 level의 중첩 객체
- 숫자형 key 문자열
- `null`, `false`, 음수, 0
- 한글, 조합 문자, 따옴표, 역슬래시, 개행, 제어 문자
- 배열 안의 객체와 배열
- 동일 값이지만 객체 삽입 순서만 다른 입력
- 중복 요청이 동시에 도착하는 경우

#### 실패 조건

- `NaN`, `Infinity`, `undefined`, 함수, symbol, bigint, 순환 참조처럼 JSON 계약 밖의 값은 명시적으로 거부한다.
- 클라이언트 선언 hash가 실제 body hash와 다르면 registry 조회 전에 거부한다.
- identity conflict에서 기존 저장 결과를 덮어쓰지 않는다.
- 동시 최초 요청 두 개가 `applyOnce`를 각각 실행하면 실패다.

#### 필요한 제약

- hash 함수 구현 자체는 제공된다.
- registry의 영속 저장 방식은 추상화해도 되지만, ID 단위 원자적 insert-or-read가 가능하다고 가정하거나 필요한 lock을 명시한다.
- HMAC, 사용자 인증, 권한 검사는 이 문제 범위가 아니며 별도 계층이 필요하다.

### 구현 후 자가 검증

- 정상 경로: 최초 요청은 한 번 적용되고 결과가 저장된다.
- 중복 처리: 같은 ID·같은 내용의 두 번째 요청에서 apply count가 증가하지 않는다.
- 충돌 처리: 같은 ID·다른 method/path/payload 중 하나라도 바뀌면 terminal conflict다.
- 경계값: key 순서만 바뀐 중첩 객체와 Unicode fixture가 같은 canonical/hash를 만든다.
- 동시성: 같은 ID의 동시 요청에서도 side effect가 한 번뿐이다.
- 보안: hash를 인증 수단으로 설명하지 않는다.
- 시간·공간 복잡도: object key 정렬 비용과 중첩 깊이를 설명할 수 있다.

### 구현 후 설명할 것

1. ID와 hash가 각각 제공하는 보장
2. canonicalization 규칙을 프로토콜로 고정해야 하는 이유
3. 서버가 실제 body를 다시 hash해야 하는 이유
4. 중복 결과를 최초 status/body까지 저장하는 이유
5. SHA-256 hash와 HMAC·서명·인증의 차이

### 원본 확인 위치

- Thread: M06
- 커밋: `feat(M06): persist mutation identity and atomically acknowledge replay`
- 파일: `src/mutationProtocol.ts`, `src/itemStore.ts`, `src/sync.ts`, `fixture/server.cjs`, `verification/M06-inputs.json`, `scripts/verify_m06.py`
- 함수·메서드: `canonicalJson`, `mutationHash`, `mutationTarget`, `ItemStore.rejectIdentity`
- 관련 Thread: M05, M07, M10

---

<a id="p07"></a>
## [M06 / `feat(M06): persist mutation identity and atomically acknowledge replay`] ACK·receipt·dequeue의 원자적 상태 전이

### 면접 질문

서버 응답을 검증한 뒤 M06는 마지막 acknowledgement 기록과 pending head 제거를 한 트랜잭션으로 묶습니다. 이후 기록에서는 후속 로컬 편집이 없을 때 canonical ACK 결과를 Item과 `remote_versions`에도 원자적으로 반영합니다. 어떤 조건을 확인한 뒤 어떤 상태들을 함께 커밋해야 합니까?

꼬리 질문:

- acknowledgement 기록은 성공했지만 dequeue가 실패하면 다음 시작에서 무엇이 일어납니까?
- dequeue만 성공하고 receipt 기록이 실패하면 lost-ACK 분석과 복구가 왜 어려워집니까?
- queue의 두 번째 항목에 대한 out-of-order ACK를 허용하면 어떤 의도가 사라질 수 있습니까?
- 응답 Item ID가 요청 target과 다르면 왜 전체 ACK를 거부해야 합니까?
- 첫 mutation의 응답이 늦게 왔는데 같은 Item에 더 최신 로컬 rename이 pending이라면 canonical Item을 바로 덮어써도 됩니까?
- terminal identity/version conflict로 표시된 intent가 ACK 경로에 들어오면 어떻게 해야 합니까?

### 30초 모범 답변

ACK의 linearization point는 receipt 기록, 검증된 head dequeue, 필요한 canonical metadata 갱신이 함께 COMMIT되는 순간입니다. sequence·mutation ID·hash·target이 현재 nonterminal head와 정확히 맞아야 하고 foreign·out-of-order ACK는 거부합니다. 트랜잭션 일부가 실패하면 head와 receipt 모두 이전 상태여서 동일 envelope로 안전하게 재시도할 수 있어야 합니다. 같은 Item에 더 최신 pending 편집이 있으면 이전 canonical 응답으로 현재 optimistic Item을 덮어쓰지 않고 receipt만 남깁니다. 후속 편집이 없을 때만 canonical 결과를 함께 승격할 수 있습니다.

### 답변 핵심 키워드

atomic acknowledgement, head validation, receipt, dequeue, exact identity/hash/target, stale response, successor preservation, canonical promotion, rollback

### 백지 구현

#### 구현 목표

검증된 서버 mutation 결과를 로컬 저장소에 반영하는 `acknowledge` 함수를 구현한다. queue head 검증, receipt 저장, dequeue, 안전한 canonical promotion이 하나의 트랜잭션으로 성공하거나 모두 실패해야 한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type Acknowledgement = {
  clientMutationId: string;
  payloadHash: string;
  status: number;
  item?: Item;
  deletedId?: string;
};

export interface AckTx {
  readPendingInOrder(): Promise<readonly PendingMutation[]>;
  readItem(id: string): Promise<Item | null>;
  writeLastAcknowledgement(value: Acknowledgement): Promise<void>;
  deletePending(sequence: number): Promise<void>;
  upsertCanonicalItem(item: Item): Promise<void>;
  recordRemoteVersion(item: Item): Promise<void>;
}

export interface AckDatabase {
  transaction<T>(work: (tx: AckTx) => Promise<T>): Promise<T>;
}

export async function acknowledge(
  db: AckDatabase,
  expected: PendingMutation,
  result: Acknowledgement,
): Promise<void> {
  // 직접 구현
}
```

#### 입력과 출력

- `expected`: 네트워크 전송에 사용한 durable pending row
- `result`: HTTP status/body 검증을 마친 acknowledgement
- 출력: 성공 시 `void`; 현재 queue와 맞지 않거나 DB 쓰기가 실패하면 reject

#### 반드시 만족해야 할 조건

- 현재 queue의 첫 번째 nonterminal row가 `expected.sequence`, `clientMutationId`, `payloadHash`, kind, target과 일치해야 한다.
- terminal row, foreign identity, 다른 target, out-of-order row의 ACK는 어떤 write도 하기 전에 거부한다.
- create/rename/toggle 결과 Item의 ID는 요청 target과 일치해야 한다.
- delete 결과의 `deletedId`도 target과 일치해야 한다.
- receipt 저장과 head 삭제는 같은 트랜잭션이다.
- 같은 Item에 더 최신 pending successor가 있으면 현재 optimistic Item을 과거 canonical 결과로 덮어쓰지 않는다.
- 같은 Item successor가 없고 결과가 Item을 포함하면 canonical Item과 remote version을 같은 트랜잭션에서 승격할 수 있다.
- 어느 statement나 COMMIT이 실패해도 receipt, queue, Item, remote version이 모두 이전 세대에 남는다.

#### 경계 조건

- queue가 비어 있음
- expected가 head와 sequence만 다름
- identity/hash만 다름
- 응답 Item ID가 다름
- head가 terminal error 상태
- 같은 Item의 후속 rename이 있음
- 다른 Item의 후속 operation만 있음
- delete acknowledgement
- 같은 ACK를 두 번 호출
- receipt UPDATE, dequeue, Item UPDATE, remote version INSERT, COMMIT 각각의 실패

#### 실패 조건

- 일부 상태만 커밋되는 부분 성공
- out-of-order ACK가 앞선 intent를 건너뛰어 제거
- 과거 ACK가 최신 로컬 편집을 덮어쓰기
- terminal intent가 정상 성공으로 변환되기
- target 불일치 응답을 receipt로 기록하기

#### 필요한 제약

- 네트워크 호출은 트랜잭션 밖에서 이미 끝났다고 가정한다.
- queue 크기가 큰 경우 successor 탐색을 어떻게 인덱싱할지는 설명하되, 축소 구현은 정렬된 배열을 사용할 수 있다.
- duplicate server response는 P06 registry가 원래 결과를 돌려준 상태라고 가정한다.

### 구현 후 자가 검증

- 정상 경로: head ACK 뒤 receipt가 남고 head만 제거된다.
- 상태 변화: successor 없음/있음에 따라 canonical promotion 여부가 달라진다.
- 실패 경로: 모든 write 지점과 COMMIT 실패 뒤 DB를 다시 열어 동일 이전 상태인지 확인한다.
- invariant: queue는 prefix만 제거할 수 있고 중간 row를 건너뛰지 않는다.
- 중복·누락 처리: foreign, out-of-order, terminal, target mismatch ACK가 intent를 지우지 않는다.
- process recovery: 실패 뒤 동일 envelope를 다시 보내 성공적으로 마무리할 수 있다.
- 동시성: foreground와 background가 같은 head를 ACK하려 할 때 한쪽만 상태를 전진시킨다.

### 구현 후 설명할 것

1. ACK 트랜잭션의 linearization point와 포함한 테이블
2. receipt와 dequeue 중 하나만 커밋될 때 생기는 장애
3. head-only 검증을 택한 이유
4. 후속 local successor가 있을 때 canonical promotion을 막는 기준
5. 네트워크 I/O를 DB 트랜잭션 안에 넣지 않은 이유

### 원본 확인 위치

- Thread: M06
- 커밋: `feat(M06): persist mutation identity and atomically acknowledge replay`
- 파일: `src/itemStore.ts`, `src/sync.ts`, `__tests__/items.test.ts`, `__tests__/sync.test.ts`
- 메서드·상태: `ItemStore.acknowledge`, `ItemStore.rejectIdentity`, `sync_metadata.last_acknowledgement`, `pending_mutations`, `remote_versions`
- 관련 Thread: M05, M07, M10
