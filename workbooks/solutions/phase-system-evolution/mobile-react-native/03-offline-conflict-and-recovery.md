# 오프라인 읽기, 충돌 해결, 프로세스 복구

이 문서는 실패한 네트워크 요청에서도 기존 데이터를 잃지 않는 M04, 동시에 변경된 원격 상태와의 충돌을 명시적으로 처리한 M07, process death 뒤 저장된 의도를 foreground 시작에서 복구한 M09를 묶는다.

---

<a id="p08"></a>
## [M04 / 기록에서 정확한 커밋 메시지 미확인] 오프라인 우선 새로고침 상태와 성공 메타데이터

### 면접 질문

M04의 앱은 시작할 때 네트워크를 호출하지 않고 SQLite를 읽어 `stale`로 표시합니다. 사용자가 명시적으로 동기화하면 `refreshing`이 되고, snapshot과 `last_successful_refresh_at`이 함께 커밋된 뒤에만 `fresh`가 됩니다. 이 상태 모델이 해결하는 문제와 `fresh`의 정확한 의미를 설명해 보세요.

꼬리 질문:

- 네트워크가 끊겼거나 GET이 503일 때 기존 목록과 마지막 성공 시각을 유지한 이유는 무엇입니까?
- snapshot row 저장은 성공했지만 성공 시각 UPDATE가 실패하면 둘 중 어느 것을 공개해야 합니까?
- 로컬 편집 직후 상태를 다시 `stale`로 만든 이유는 무엇입니까?
- 로컬 저장 오류와 원격 refresh 오류를 한 개의 `error` 상태로 합치면 어떤 UX·디버깅 문제가 생깁니까?
- `fresh`가 "현재 서버와 계속 동일하다"는 뜻이 아닌 이유는 무엇입니까?
- 앱 시작 때 자동 GET을 하지 않은 선택의 장점과 단점은 무엇입니까?

### 30초 모범 답변

오프라인 우선 화면에서 목록의 가용성과 원격 확인 상태는 분리해야 합니다. 시작 시에는 커밋된 SQLite를 즉시 보여 주되 아직 서버를 확인하지 않았으므로 `stale`입니다. 명시적 GET과 응답 검증, snapshot·성공 시각의 같은 트랜잭션이 끝난 뒤에만 `fresh`로 바꿉니다. 실패하면 기존 목록과 이전 성공 시각을 보존하고 refresh 오류만 표시합니다. 로컬 편집도 마지막 원격 확인 이후 상태를 바꾸므로 다시 stale입니다. 따라서 fresh는 마지막 확인이 성공했다는 뜻이지 지속적인 최신성 보장은 아닙니다.

### 답변 핵심 키워드

offline-first, cached availability, freshness state machine, stale-on-edit, atomic snapshot+metadata, last known success, explicit refresh, failure isolation

### 백지 구현

#### 구현 목표

캐시된 목록을 유지하는 refresh 상태 머신과, 원격 snapshot·성공 시각을 원자적으로 커밋하는 coordinator를 구현한다. UI 렌더링 코드는 작성하지 않는다.

#### 인터페이스 또는 함수 시그니처

```ts
export type RefreshState =
  | {status: 'stale'; lastSuccessfulAt: number | null}
  | {status: 'refreshing'; lastSuccessfulAt: number | null}
  | {status: 'fresh'; lastSuccessfulAt: number}
  | {status: 'error'; lastSuccessfulAt: number | null; message: string};

export type RefreshEvent =
  | {type: 'opened'; lastSuccessfulAt: number | null}
  | {type: 'local_changed'}
  | {type: 'refresh_started'}
  | {type: 'refresh_succeeded'; at: number}
  | {type: 'refresh_failed'; message: string};

export function reduceRefresh(
  state: RefreshState,
  event: RefreshEvent,
): RefreshState {
  // 직접 구현
}

export interface SnapshotStore {
  readItems(): Promise<readonly Item[]>;
  readLastSuccessfulRefresh(): Promise<number | null>;
  replaceSnapshot(items: readonly Item[], successfulAt: number): Promise<void>;
}

export async function refreshItems(
  store: SnapshotStore,
  fetchSnapshot: () => Promise<unknown>,
  validate: (value: unknown) => readonly Item[],
  now: () => number,
): Promise<{items: readonly Item[]; successfulAt: number}> {
  // 직접 구현
}
```

#### 입력과 출력

- reducer 입력: 현재 refresh 표시 상태와 하나의 이벤트
- reducer 출력: 다음 표시 상태
- coordinator 입력: 저장소, 원격 요청 함수, snapshot validator, 성공 시각 공급자
- coordinator 출력: COMMIT 뒤 다시 읽은 목록과 성공 시각

#### 반드시 만족해야 할 조건

- 앱 open은 네트워크 요청 없이 저장된 목록과 마지막 성공 시각을 읽는다.
- open 뒤 상태는 마지막 성공 시각 유무와 관계없이 `stale`이다.
- refresh 시작은 기존 목록을 지우지 않는다.
- network, HTTP, JSON validation, DB write 중 하나라도 실패하면 기존 snapshot과 성공 시각이 유지된다.
- 성공 시각은 원격 응답 수신 시점이 아니라 검증된 snapshot을 커밋하는 성공 세대를 나타낸다.
- snapshot과 성공 시각은 같은 저장소 트랜잭션에서 성공하거나 함께 롤백된다.
- 성공 뒤 반환하는 목록은 HTTP body가 아니라 커밋된 저장소 읽기다.
- 로컬 변경 이벤트는 마지막 성공 시각을 보존하면서 상태를 stale로 만든다.

#### 경계 조건

- 최초 실행이라 성공 시각이 없음
- 캐시는 있지만 네트워크가 없음
- HTTP 503
- JSON shape 오류 또는 중복 ID
- 빈 원격 snapshot
- snapshot row 중간 실패
- metadata UPDATE 실패
- COMMIT 실패
- refresh 성공 직후 로컬 편집
- 중복 refresh 버튼 입력

#### 실패 조건

- 실패한 원격 응답으로 캐시를 비우기
- snapshot만 새 세대이고 성공 시각은 이전 세대인 부분 커밋
- 오류 뒤 마지막 성공 시각을 현재 시각으로 갱신
- `fresh` 상태를 지속적인 서버 구독처럼 설명하기

#### 필요한 제약

- 자동 retry와 polling은 구현하지 않는다.
- 한 번에 하나의 refresh만 실행된다고 가정하거나, 중복 실행을 막는 gate를 별도로 명시한다.
- `now()`는 테스트에서 고정 가능한 의존성으로 받는다.

### 구현 후 자가 검증

- 정상 경로: open → refreshing → fresh와 마지막 성공 시각이 일치한다.
- 실패 경로: offline, 503, invalid snapshot, 각 DB write, COMMIT 실패에서 캐시와 시각이 그대로다.
- 상태 변화: 로컬 편집은 fresh/error 어느 상태에서도 stale로 전환하되 목록을 유지한다.
- invariant: `fresh.at`은 저장소의 `last_successful_refresh_at`과 같은 snapshot 세대를 설명한다.
- 동시성·비동기: 두 refresh가 겹칠 가능성을 계약이나 gate로 처리한다.
- 요구사항 충족 여부: open 경로에서 `fetchSnapshot`을 호출하지 않는다.

### 구현 후 설명할 것

1. 데이터 가용성과 freshness를 분리한 이유
2. snapshot과 성공 시각을 한 트랜잭션에 둔 이유
3. HTTP body 대신 커밋된 저장소를 다시 읽는 이유
4. 자동 refresh 대신 명시적 refresh를 택한 trade-off
5. fresh의 의미와 TTL·push가 추가될 때 달라질 점

### 원본 확인 위치

- Thread: M04
- 커밋: 기록에서 정확한 메시지 미확인
- 파일: `src/App.tsx`, `src/itemStore.ts`, `src/sync.ts`, `__tests__/App.test.tsx`, `__tests__/sync.test.ts`, `fixture/server.cjs`, `scripts/verify_m04.py`
- 함수·메서드: `readLastSuccessfulRefresh`, `replaceSnapshot`, `ForegroundSync.synchronize`
- 관련 Thread: M03, M09

---

<a id="p09"></a>
## [M07 / `feat(M07): preserve stale intent with versioned canonical reconciliation`] 낙관적 버전 충돌과 canonical reconciliation

### 면접 질문

M07은 mutation마다 기준 version과 dispatch 여부를 저장하고, 서버가 `version_conflict`를 반환하면 원격 canonical 상태를 로컬에 채택하면서 원래 시도를 `MutationConflict`로 보존하고 자동 재시도에서 제외합니다. 왜 단순한 last-write-wins나 무조건 재시도 대신 이런 정책을 택했습니까?

꼬리 질문:

- 클라이언트 시각의 `updatedAt`으로 승자를 정하면 어떤 문제가 있습니까?
- 충돌한 원래 intent를 삭제만 하지 않고 증거로 남긴 이유는 무엇입니까?
- canonical-wins가 사용자 입력을 무시하는 정책처럼 보일 때 어떤 후속 UX가 필요합니까?
- 같은 Item에 아직 서버로 보내지 않은 후속 편집이 있으면 baseVersion을 안전하게 바꿀 수 있는 조건은 무엇입니까?
- 이미 dispatch된 successor를 자동 rebase하면 왜 위험합니까?
- 원격에서 Item이 삭제된 tombstone 충돌은 현재 Item row와 queue에 어떻게 반영해야 합니까?
- conflict 기록, terminal 처리, canonical Item/version 갱신 중 하나가 실패하면 어떤 상태가 남아야 합니까?

### 30초 모범 답변

Version precondition은 서버가 본 상태를 기준으로 stale write를 검출하므로 클라이언트 시계보다 신뢰할 수 있습니다. 충돌 시 원래 요청을 그대로 재시도하면 새 원격 변경을 반복해서 덮으려 하므로 terminal 처리하고, 서버 canonical을 로컬 기준으로 채택합니다. 대신 사용자의 원래 payload와 canonical 결과를 conflict 증거로 보존해 손실을 숨기지 않습니다. 후속 편집은 아직 dispatch되지 않았고 같은 Item에 속할 때만 새 canonical version에 rebase할 수 있습니다. 이미 전송된 요청의 의미는 바꾸지 않으며, 새 사용자 편집은 별도 identity로 다시 시도합니다.

### 답변 핵심 키워드

optimistic concurrency, baseVersion, server canonical, terminal conflict, evidence preservation, no blind retry, tombstone, undispatched successor, explicit retry as new intent

### 백지 구현

#### 구현 목표

현재 queue head에 대한 version conflict 응답을 받아 로컬 canonical 상태, conflict evidence, pending queue를 한 번에 갱신하는 순수 전이 함수를 구현한다. 이후 DB 트랜잭션에 넣을 수 있도록 입력과 출력을 명시적인 상태 값으로 표현한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type VersionedPending = PendingMutation & {
  baseVersion: number;
  dispatched: boolean;
  terminalError: null | 'version_conflict' | 'identity_conflict';
};

export type CanonicalRemote =
  | {kind: 'item'; item: Item}
  | {kind: 'deleted'; id: string; version: number};

export type ConflictEvidence = {
  operation: VersionedPending;
  canonical: CanonicalRemote;
};

export type ConflictState = {
  items: readonly Item[];
  pending: readonly VersionedPending[];
  conflicts: readonly ConflictEvidence[];
};

export function resolveVersionConflict(
  state: ConflictState,
  expectedHead: VersionedPending,
  canonical: CanonicalRemote,
): ConflictState {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 현재 Item 목록, 순서가 있는 pending 목록, 누적 conflict 증거, 전송했던 head, 서버 canonical 결과
- 출력: 충돌 처리가 완료된 다음 논리 상태
- 실제 저장에서는 출력 전체가 하나의 트랜잭션으로 반영된다고 가정한다.

#### 반드시 만족해야 할 조건

- `expectedHead`는 현재 첫 nonterminal pending과 identity, hash, target, baseVersion이 일치해야 한다.
- 서버 canonical의 Item ID 또는 tombstone ID는 mutation target과 일치해야 한다.
- 원래 충돌 operation은 더 이상 자동 전송 대상이 아니어야 한다.
- 원래 operation과 canonical 결과를 conflict evidence에 정확히 한 번 추가한다.
- canonical이 Item이면 그 Item을 로컬 기준 상태로 채택한다.
- canonical이 tombstone이면 해당 Item을 로컬 목록에서 제거한다.
- 다른 Item의 pending operation은 순서와 내용을 유지한다.
- 같은 Item의 successor를 자동 rebase하려면 아직 dispatch되지 않았다는 조건을 만족해야 한다.
- 이미 dispatch된 successor의 identity, payload, baseVersion을 조용히 바꾸지 않는다.
- 사용자가 canonical 상태를 본 뒤 새로 편집할 수 있도록 conflict 기록과 현재 Item 상태를 분리한다.

#### 경계 조건

- queue가 비어 있음
- expected head가 아님
- terminal head
- canonical target 불일치
- 같은 Item successor 없음
- 같은 Item의 undispatched successor 하나 이상
- 같은 Item의 dispatched successor
- 다른 Item operation이 사이에 있음
- remote tombstone
- 동일 conflict 응답이 두 번 도착

#### 실패 조건

- conflict operation을 정상 retry queue에 그대로 남기기
- conflict 증거 없이 원격 상태로 덮어쓰기
- 서버 canonical을 무시하고 로컬 timestamp로 승자 결정
- 이미 dispatch된 successor의 의미 변경
- 다른 Item의 queue 순서 변경

#### 필요한 제약

- 이 문제는 conflict 한 건의 논리 전이에 집중한다.
- DB 구현에서는 conflict INSERT, pending terminal/dequeue, Item/tombstone, remote version 갱신을 한 트랜잭션으로 묶어야 한다.
- 자동 merge 정책은 제공하지 않는다. 별도 도메인 merge가 가능하다면 canonical 채택 뒤 새 intent를 만드는 방식과 비교 설명한다.

### 구현 후 자가 검증

- 정상 경로: stale rename conflict가 canonical Item과 conflict evidence를 남기고 자동 retry에서 빠진다.
- tombstone: stale update 대상이 원격 삭제됐을 때 로컬 row가 사라지고 증거는 남는다.
- 상태 변화: unrelated pending과 다른 Item은 바뀌지 않는다.
- invariant: 동일한 원래 intent가 conflict evidence에 중복 기록되지 않는다.
- 중복·누락 처리: foreign/out-of-order conflict 응답은 상태를 바꾸지 않는다.
- 동시성: dispatch marker와 rebase 판단이 같은 저장소 직렬화 경계 안에 있는지 확인한다.
- 요구사항 충족 여부: 사용자의 새 explicit edit는 과거 terminal identity를 재사용하지 않는다.

### 구현 후 설명할 것

1. version precondition을 클라이언트 timestamp보다 신뢰한 이유
2. canonical-wins와 conflict evidence를 함께 사용한 이유
3. automatic retry를 중단하는 기준
4. undispatched successor만 rebase할 수 있는 이유
5. 자동 merge, 사용자 선택 UI, server-wins 각각의 trade-off

### 원본 확인 위치

- Thread: M07
- 커밋: `feat(M07): preserve stale intent with versioned canonical reconciliation`
- 파일: `src/itemStore.ts`, `src/sync.ts`, `src/App.tsx`, `__tests__/items.test.ts`, `__tests__/sync.test.ts`, `scripts/verify_m07.py`, `verification/M07.md`
- 타입·함수·메서드: `MutationConflict`, `prepareNext`, `rejectVersion`, `readConflicts`, `remoteTombstone`, `decodeMutation`, `matchesHead`, `writeConflict`, `readVersions`, `acceptsVersion`, `writeVersion`
- 관련 Thread: M06, M09

---

<a id="p10"></a>
## [M09 / `feat(m09): resume durable uploads on foreground startup`] process death 뒤 foreground startup 복구 정책

### 면접 질문

M09는 앱 시작 때 저장된 nonterminal pending intent가 있을 때만 foreground synchronization을 한 번 시도합니다. 빈 queue나 terminal-only queue는 캐시만 표시하고, 앱이 켜진 뒤 새로 생긴 편집은 자동 전송하지 않습니다. 이 복구 범위를 이렇게 제한한 이유는 무엇입니까?

꼬리 질문:

- process death 뒤 durable queue가 있어도 앱 시작 코드가 아무 동작을 하지 않으면 어떤 한계가 남습니까?
- terminal identity/version conflict를 startup에서 다시 보내면 왜 안 됩니까?
- 시작 직후 오프라인이면 head의 dispatch marker와 queue는 어떤 상태여야 합니까?
- open effect가 끝나기 전에 component가 unmount되거나 새 root가 만들어지면 오래된 callback을 어떻게 막습니까?
- startup 자동 시도와 사용자의 수동 Synchronize가 겹치면 무엇을 직렬화해야 합니까?
- polling·connectivity listener·OS scheduler 없이 foreground startup만 사용한 설계의 장단점은 무엇입니까?
- M10에서 WorkManager가 queue를 소유하게 되면 M09 startup 정책은 어떻게 달라져야 합니까?

### 30초 모범 답변

M09의 목표는 process death로 중단된 이미 커밋된 의도를 다음 foreground 시작에서 한 번 복구하는 것입니다. 그래서 nonterminal durable queue가 있을 때만 자동 시도하고, terminal conflict는 사용자 판단 없이는 재전송하지 않습니다. 빈 queue는 불필요한 GET을 피하고, 실행 중 새 편집은 기존 수동 정책을 유지해 예측 가능성을 지킵니다. 오프라인 실패에서도 queue는 남아 다음 시도를 허용합니다. effect의 generation·mounted guard와 busy gate로 오래된 root나 수동 동기화와의 중복 실행을 막습니다. 다만 앱이 다시 열리지 않으면 실행되지 않는 한계가 있어 M10의 OS scheduler가 필요합니다.

### 답변 핵심 키워드

cold-start recovery, persisted nonterminal intent, one-shot automatic attempt, terminal exclusion, lifecycle generation, busy gate, no polling, foreground-only limitation

### 백지 구현

#### 구현 목표

저장소 open 뒤 캐시를 공개하고, queue 내용에 따라 시작 시 동기화를 정확히 한 번 수행할지 결정하는 coordinator를 구현한다. component가 폐기된 뒤 늦은 비동기 결과는 새 화면 상태를 바꾸지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type StartupDecision = 'show_cached_only' | 'resume_once';

export function decideStartupRecovery(
  pending: readonly {terminalError: string | null}[],
): StartupDecision {
  // 직접 구현
}

export interface StartupStore {
  readItems(): Promise<readonly Item[]>;
  readPending(): Promise<readonly {terminalError: string | null}[]>;
}

export interface StartupUi {
  isCurrent(): boolean;
  publishCached(items: readonly Item[]): void;
  publishRefreshState(state: 'stale' | 'fresh' | 'error'): void;
}

export async function initializeAndRecover(
  openStore: () => Promise<StartupStore>,
  synchronize: (store: StartupStore) => Promise<void>,
  ui: StartupUi,
): Promise<void> {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: durable store opener, 한 번의 synchronization 함수, 현재 root인지 확인하고 상태를 공개하는 UI adapter
- 출력: 완료를 나타내는 Promise. 저장·네트워크 오류는 계약에 따라 상태로 공개하거나 reject한다.

#### 반드시 만족해야 할 조건

- 저장소를 연 뒤 먼저 커밋된 cached Items를 읽어 현재 root에 공개한다.
- pending 중 `terminalError === null`인 항목이 하나라도 있을 때만 startup sync를 시도한다.
- 빈 queue와 terminal-only queue는 네트워크를 호출하지 않는다.
- 한 번의 mount/open attempt에서 자동 sync는 최대 한 번이다.
- startup sync가 실패해도 pending을 지우거나 cached Items를 비우지 않는다.
- unmount되거나 더 새 root가 생긴 뒤에는 오래된 callback이 UI 상태를 공개하지 않는다.
- 이미 시작한 DB 트랜잭션의 내구성은 저장소가 책임지되, 폐기된 root가 draft·keyboard·status를 변경하지 않는다.
- 앱이 준비된 뒤 새로 생기는 live mutation은 이 함수가 자동 전송하지 않는다.

#### 경계 조건

- open 실패
- 빈 DB와 빈 queue
- cached Items와 빈 queue
- nonterminal head 하나
- terminal-only queue
- terminal row 뒤에 nonterminal row
- startup sync 중 네트워크 실패
- open 직후 unmount
- cached publish 뒤 sync 중 unmount
- 수동 synchronize가 거의 동시에 시작됨

#### 실패 조건

- terminal intent 전송
- 같은 mount에서 자동 sync 반복
- 오래된 root가 새 root의 상태를 stale/fresh/error로 덮어쓰기
- 네트워크 실패를 로컬 저장 실패로 표시하기
- 빈 queue에서 불필요한 HTTP 요청

#### 필요한 제약

- polling, timer, network listener, WorkManager는 구현하지 않는다.
- foreground와 수동 sync 직렬화 primitive가 외부에 있다면 그 계약을 명시한다.
- queue drain 세부 로직은 P05~P07의 저장소·sync 구현을 재사용한다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: nonterminal queue에서 cached publish 후 자동 sync가 정확히 한 번 실행된다.
- 경계값: empty/terminal-only queue에서 HTTP가 0회다.
- 실패 경로: offline과 open 오류가 cached state·pending을 훼손하지 않는다.
- lifecycle: 각 await 사이에서 `isCurrent()`가 false가 되는 경우를 검증한다.
- 중복·누락 처리: 수동 sync와 startup sync가 같은 head를 동시에 처리하지 않는다.
- invariant: startup recovery 여부는 메모리 flag가 아니라 durable queue 내용에서 결정된다.
- 요구사항 충족 여부: 앱이 열린 뒤 새 mutation을 자동으로 보내지 않는다.

### 구현 후 설명할 것

1. nonterminal queue만 startup 복구 대상으로 삼은 이유
2. one-shot foreground 복구와 background scheduler의 차이
3. cached Items를 먼저 공개한 이유
4. mounted/generation guard와 저장소 내구성의 책임 분리
5. 수동 sync와 startup sync를 직렬화하는 위치

### 원본 확인 위치

- Thread: M09
- 커밋: `feat(m09): resume durable uploads on foreground startup`
- 파일: `src/App.tsx`, `src/itemStore.ts`, `src/sync.ts`, `__tests__/App.test.tsx`, `__tests__/sync.test.ts`, `scripts/verify_m09.py`, `fixture/server.cjs`, `verification/M09-inputs.json`
- 관련 Thread: M05, M06, M10
