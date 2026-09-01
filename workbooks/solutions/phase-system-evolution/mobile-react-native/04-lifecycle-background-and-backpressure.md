# Lifecycle, 백그라운드 실행, Backpressure

이 문서는 M08의 UI 상태 소유권, M10의 OS 백그라운드 작업, M14의 네트워크 취소를 한 흐름으로 연결한다. 세 문제의 공통 질문은 "이 비동기 작업은 누가 소유하며, 소유자가 사라진 뒤 무엇까지 완료해도 되는가"이다.

---

<a id="p11"></a>
## [M08 / `feat(M08): retain editor state across Activity remounts`] Activity remount 상태 소유권과 stale callback

### 면접 질문

M08은 선택한 Item ID와 제출 전 draft를 `createEditorMemory`가 만든 작은 in-memory owner에 두어 Activity remount에서는 복원하지만 process/runtime termination 뒤에는 복원하지 않습니다. 이 경계를 선택한 이유와, 늦게 완료되는 open/save callback을 어떻게 막아야 하는지 설명해 보세요.

꼬리 질문:

- React component state만 사용하면 Activity recreation에서 어떤 상태가 사라질 수 있습니까?
- draft를 SQLite나 upload queue에 저장하지 않은 이유는 무엇입니까?
- Activity recreation, component remount, JS runtime 재사용, process death는 어떻게 구분합니까?
- 이전 root에서 시작한 Save가 끝난 뒤 새 root의 draft를 비우면 어떤 버그입니까?
- 실패한 Save는 draft와 keyboard 상태를 어떻게 처리해야 합니까?
- `mounted` boolean 하나만으로 같은 runtime 안의 오래된 작업과 새 작업을 모두 구분할 수 있습니까?
- 이미 시작된 durable DB mutation과 UI callback의 생명주기는 왜 별도로 봐야 합니까?

### 30초 모범 답변

Draft는 아직 제출되지 않은 세션 UI 상태라서 Activity remount에는 유지하되 process death까지 내구성을 약속하지 않는 경계를 택했습니다. 그래서 현재 JS runtime이 소유하는 작은 editor memory를 component 밖에 두고 remount 시 초기값으로 읽습니다. 반면 DB에 COMMIT된 mutation은 component보다 오래 사는 저장소 책임입니다. 각 비동기 작업은 시작한 root나 generation을 캡처하고, 완료 시 현재 owner와 일치할 때만 draft를 비우거나 keyboard·status를 바꿉니다. 실패한 Save는 입력을 유지하고, 오래된 root의 callback은 새 editor에 영향을 주지 않습니다.

### 답변 핵심 키워드

state ownership, Activity recreation, JS runtime lifetime, process death boundary, editor memory, generation token, mounted guard, late completion, durable work separation

### 백지 구현

#### 구현 목표

같은 in-memory owner를 공유하는 remount 사이에서 editor 상태를 복원하고, 오래된 Save completion이 새 root의 draft를 지우지 못하게 하는 프레임워크 비의존 controller를 구현한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type EditorSnapshot = {
  editingId: string | null;
  draft: string;
};

export type EditorMemory = {
  current: EditorSnapshot;
};

export function createEditorMemory(): EditorMemory {
  // 직접 구현
}

export type SaveRequest = {
  token: string;
  editingId: string | null;
  title: string;
};

export class EditorController {
  constructor(
    private readonly memory: EditorMemory,
    private readonly generation: string,
    private readonly isCurrentGeneration: (generation: string) => boolean,
  ) {}

  snapshot(): EditorSnapshot {
    // 직접 구현
  }

  beginEdit(id: string, currentTitle: string): void {
    // 직접 구현
  }

  changeDraft(value: string): void {
    // 직접 구현
  }

  cancel(): void {
    // 직접 구현
  }

  beginSave(): SaveRequest | null {
    // 직접 구현
  }

  completeSave(token: string, succeeded: boolean): boolean {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 같은 `EditorMemory`를 받은 새 controller는 이전 editor snapshot을 읽는다.
- 다른 `EditorMemory`는 빈 editor에서 시작한다.
- `beginSave`는 현재 draft가 유효할 때 저장 요청 snapshot을 반환한다.
- `completeSave`는 현재 owner가 해당 요청의 UI 후처리를 수행했는지 boolean으로 반환한다.

#### 반드시 만족해야 할 조건

- draft와 editingId는 함께 하나의 snapshot으로 갱신한다.
- begin edit, draft 변경, cancel, 성공한 save 후처리는 memory에 동기적으로 반영한다.
- 같은 memory를 사용하는 remount는 해당 snapshot으로 초기화된다.
- process/runtime 재시작을 표현하는 새 memory는 `{editingId: null, draft: ''}`에서 시작한다.
- 공백 draft는 save request를 만들지 않는다.
- Save는 시작 시 editingId와 정규화된 title을 캡처한다.
- Save가 진행되는 동안 사용자가 새 draft를 입력하거나 다른 Item 편집을 시작하면 과거 completion이 이를 지우지 않는다.
- 실패한 Save는 해당 요청이 여전히 현재 편집이라도 draft를 보존한다.
- 더 이상 current generation이 아닌 controller는 memory를 변경하거나 UI 후처리를 승인하지 않는다.

#### 경계 조건

- create mode와 edit mode
- draft가 공백
- Save 시작 직후 remount
- Save 시작 뒤 새 draft 입력
- Save 시작 뒤 다른 Item 편집
- 실패한 Save
- 이전 root의 completion이 새 root completion보다 늦게 도착
- cancel 뒤 늦은 Save completion
- 같은 callback이 두 번 호출

#### 실패 조건

- process death 뒤 draft가 복원된다고 잘못 약속하기
- 오래된 completion이 새 draft·editingId를 초기화하기
- 실패한 Save가 draft를 비우기
- generation이 폐기된 뒤 새 mutation을 시작하도록 허용하기
- editor memory에 DB COMMIT 여부를 대신 저장하기

#### 필요한 제약

- 실제 React hook, Keyboard API, SQLite는 사용하지 않는다.
- keyboard dismiss는 `completeSave`가 `true`일 때만 현재 UI 계층이 수행한다고 가정한다.
- token 생성은 충돌하지 않으면 형식은 자유다.

### 구현 후 자가 검증

- 정상 경로: edit → draft 변경 → remount → 성공 save에서 상태가 올바르게 이어진다.
- 경계값: 새 memory는 이전 draft를 보지 않는다.
- 실패 경로: 실패한 save가 draft를 보존한다.
- lifecycle: 이전 generation의 모든 메서드와 completion이 새 memory 상태를 바꾸지 않는다.
- 상태 변화: 동일 요청의 중복 completion은 한 번만 후처리한다.
- invariant: editingId와 draft는 서로 다른 세대의 값으로 섞이지 않는다.
- 요구사항 충족 여부: 저장소 내구성과 UI memory 수명을 혼동하지 않는다.

### 구현 후 설명할 것

1. editor state를 DB에 넣지 않은 이유와 그로 인해 포기한 보장
2. Activity recreation과 process death의 차이
3. mounted flag, generation token, operation token의 역할 차이
4. durable mutation 완료와 UI 후처리를 분리한 이유
5. SavedState, 전역 store, SQLite를 쓸 때의 대안 trade-off

### 원본 확인 위치

- Thread: M08
- 커밋: `feat(M08): retain editor state across Activity remounts`
- 파일: `src/App.tsx`, `__tests__/App.test.tsx`, `android/app/src/androidTest/java/com/mse/reactnative/M08LifecycleTest.java`, `android/app/src/androidTest/assets/m08-inputs.json`, `verification/M08.md`
- 함수·클래스·테스트: `createEditorMemory`, `M08LifecycleTest`, `draftSurvivesOneBackgroundCycleAndOneActivityRecreation`
- 관련 Thread: M10, M14, M15

---

<a id="p12"></a>
## [M10 / 기록에서 정확한 커밋 메시지 미확인] WorkManager cycle, retry allowance, foreground/background 직렬화

### 면접 질문

M10은 로컬 mutation COMMIT 뒤 `item-uploads` unique work를 `NetworkType.CONNECTED` 조건으로 등록하고, persistent cycle에 최대 세 번의 HTTP allowance를 예약합니다. Activity와 WorkManager가 같은 JS runtime·queue를 사용할 수 있어 `serializeSync`로 upload와 ACK 전체를 직렬화합니다. 이 설계의 상태 머신과 실패 경계를 설명해 보세요.

꼬리 질문:

- 사용자 저장 완료를 Worker 실행이 끝날 때까지 기다리지 않고 "내구성 있는 작업 등록"까지만 기다린 이유는 무엇입니까?
- 스케줄 등록이 실패했는데 로컬 COMMIT은 성공했다면 UI에 무엇을 표시해야 합니까?
- Worker가 받은 token이 더 이상 active cycle이 아니면 왜 SQLite조차 열지 않아야 합니까?
- HTTP 전에 `reserve`가 내구성 있게 성공해야 하는 이유는 무엇입니까?
- 세 번째 실패 뒤 ordinary remount가 새 cycle이나 네 번째 HTTP를 만들면 왜 안 됩니까?
- foreground 수동 동기화와 background Worker가 동시에 같은 queue head를 처리하면 어떤 문제가 생깁니까?
- `requestFinished`와 `complete(success|retry|failure)`를 분리한 이유를 어떻게 설명하겠습니까?
- WorkManager의 실행 시각·프로세스 생존을 앱 코드가 완전히 보장할 수 있습니까?

### 30초 모범 답변

로컬 저장의 성공 기준은 Item과 outbox COMMIT이고, 백그라운드 전송의 성공 기준은 OS에 내구성 있게 작업이 등록되는 데까지입니다. Worker 실행 시각은 Android가 네트워크와 시스템 상태에 따라 결정합니다. 각 cycle은 persistent token과 attempt 수를 가지며, active token 확인과 reserve가 성공한 경우에만 HTTP를 허용합니다. 세 번 소진되면 deferred로 남겨 ordinary remount가 새 시도를 만들지 않습니다. Activity와 Worker는 같은 queue·ACK 트랜잭션을 만지므로 전체 sync critical section을 하나의 serializer로 감쌉니다. 스케줄 오류는 저장 실패가 아니라 별도 복구 가능 오류입니다.

### 답변 핵심 키워드

WorkManager, unique work, CONNECTED constraint, durable registration, cycle token, reservation, retry ceiling, deferred, shared serializer, scheduler-owned timing

### 백지 구현

#### 구현 목표

백그라운드 cycle의 pure state transition과 foreground/background 공용 serial executor를 구현한다. 실제 WorkManager API는 호출하지 않고, native bridge가 상태를 저장한다고 가정한다.

#### 인터페이스 또는 함수 시그니처

```ts
export type BackgroundState = {
  cycleId: string | null;
  attempts: number;
  status: 'none' | 'active' | 'deferred' | 'complete';
};

export type Reservation = {
  state: BackgroundState;
  allowed: boolean;
};

export function reserveAttempt(
  state: BackgroundState,
  token: string,
  retryCeiling: number,
): Reservation {
  // 직접 구현
}

export function completeCycle(
  state: BackgroundState,
  token: string,
  outcome: 'success' | 'retry' | 'failure',
  retryCeiling: number,
): BackgroundState {
  // 직접 구현
}

export type SerialExecutor = <T>(task: () => Promise<T>) => Promise<T>;

export function createSerialExecutor(): SerialExecutor {
  // 직접 구현
}
```

#### 입력과 출력

- `reserveAttempt`: 현재 durable cycle state, Worker token, 최대 허용 시도 수
- 출력: 갱신된 state와 이번 Worker가 HTTP를 시작해도 되는지
- `completeCycle`: 요청·동기화 결과에 따른 다음 durable state
- `createSerialExecutor`: 호출 순서대로 비동기 critical section을 실행하는 함수

#### 반드시 만족해야 할 조건

- active cycle의 정확한 token만 reservation을 받을 수 있다.
- stale token은 state를 바꾸지 않고 HTTP allowance를 받지 못한다.
- allowance는 HTTP 시작 전에 attempt 수와 함께 내구성 있게 예약된다고 가정한다.
- attempts가 ceiling에 도달한 cycle은 `deferred`이며 ordinary 재호출이 새 allowance를 만들지 않는다.
- success는 cycle을 complete로 만들고 남은 queue가 없다는 외부 조건과 함께 사용한다.
- retry는 ceiling 미만에서만 active로 남을 수 있다.
- permanent failure와 retryable failure의 정책을 구분한다.
- serial executor는 앞 task의 resolve/reject와 관계없이 다음 task가 실행될 수 있어야 한다.
- 한 task가 reject해도 내부 queue가 영구적으로 끊어지지 않는다.
- 동시에 들어온 foreground와 background 작업은 critical section 안에서 겹치지 않는다.

#### 경계 조건

- 상태 `none`
- active token 일치·불일치
- attempts 0, ceiling-1, ceiling
- 같은 Worker callback 중복
- success 뒤 stale retry callback
- task resolve, task reject, 동기 throw
- task 안에서 serializer를 다시 호출하는 경우의 정책
- 여러 호출자가 거의 동시에 enqueue

#### 실패 조건

- stale token이 DB 또는 HTTP를 시작하도록 허용
- ceiling 뒤 네 번째 allowance 생성
- scheduler 오류를 로컬 mutation rollback으로 취급
- serial task 하나의 reject로 뒤 task가 영원히 실행되지 않음
- foreground/background가 같은 head를 동시에 ACK
- UI remount가 deferred cycle을 새 active cycle로 암묵적으로 바꿈

#### 필요한 제약

- 기본 retry ceiling은 기록의 세 번을 사용할 수 있으나 함수 인자로 받는다.
- exponential backoff와 `NetworkType.CONNECTED` 설정은 native scheduler 계약으로 설명하고 pure 함수에 넣지 않는다.
- 재진입을 지원하지 않는다면 명시적으로 금지하고 테스트한다.
- 실제 HTTP·DB 로직은 P05~P07의 sync critical section을 호출한다고 가정한다.

### 구현 후 자가 검증

- 정상 경로: 503, 503, 성공 순서에서 attempts가 증가하고 마지막에 complete가 된다.
- 소진 경로: 세 번 모두 retryable 실패면 deferred가 되고 추가 reservation이 거부된다.
- stale 처리: 오래된 token은 state·DB·HTTP 모두 건드리지 않는다.
- 동시성: foreground와 background task의 실행 구간이 시간상 겹치지 않는다.
- 실패 경로: task reject 뒤 다음 task가 실행된다.
- invariant: attempts는 한 cycle에서 감소하지 않고 ceiling을 넘지 않는다.
- process recovery: state만 다시 읽어도 active/deferred/complete를 판단할 수 있다.
- 요구사항 충족 여부: 등록 실패와 local save 실패를 다른 오류로 표현한다.

### 구현 후 설명할 것

1. local COMMIT, durable schedule registration, Worker completion의 서로 다른 성공 기준
2. reserve-before-HTTP가 시도 예산을 지키는 이유
3. unique work와 cycle token을 함께 사용한 이유
4. foreground/background 전체 sync를 직렬화한 이유
5. OS scheduler에 맡긴 보장과 앱이 직접 보장하는 불변식

### 원본 확인 위치

- Thread: M10
- 커밋: 기록에서 정확한 메시지 미확인
- 파일: `src/backgroundSync.ts`, `src/App.tsx`, `index.js`, `__tests__/App.test.tsx`, `__tests__/sync.test.ts`, `scripts/verify_m10.py`, `scripts/verify_m10_baseline.py`, `verification/M10-inputs.json`, `verification/M10.md`, `android/app/src/test/java/com/mse/reactnative/BackgroundCycleTest.kt`
- 함수·타입·구성요소: `BackgroundState`, `backgroundBridge`, `ownsAutomaticSync`, `schedulePending`, `serializeSync`, `runBackgroundTask`, `BackgroundSync`, `BackgroundWorker`, `BackgroundSyncModule`, `ItemBackgroundSync`
- bridge 메서드: `getState`, `schedule`, `prepareManual`, `isActive`, `reserve`, `requestFinished`, `complete`
- 관련 Thread: M05, M09, M14

---

<a id="p13"></a>
## [M14 / `feat: cancel foreground uploads on network loss`] 네트워크 손실 취소와 backpressure

### 면접 질문

M14는 foreground synchronization이 serial queue에 들어가기 전에 고유 token, `AbortController`, Android default-network observer를 소유합니다. 네트워크 손실이나 화면 dispose 시 queued/active 요청을 취소하고, 늦은 response body가 ACK·conflict·snapshot COMMIT을 시작하지 못하게 합니다. 단순히 fetch에 `AbortSignal`만 전달하는 것으로 충분하지 않은 이유는 무엇입니까?

꼬리 질문:

- observer listener를 native registration Promise가 끝난 뒤 설치하면 어떤 race가 생깁니까?
- 이전 동기화의 늦은 offline event가 새 reconnect 요청을 취소하지 않게 하려면 무엇이 필요합니까?
- dispose가 native registration보다 먼저 호출되거나 registration이 reject해도 exact observer cleanup을 어떻게 보장합니까?
- fetch가 이미 response headers를 받았거나 body parsing 중일 때 abort되면 저장소 전이는 어디서 다시 확인해야 합니까?
- serial queue에 아직 들어가지 못한 작업과 이미 HTTP 중인 작업을 각각 어떻게 취소합니까?
- connection pool의 idle socket eviction은 어떤 문제를 줄이며, 어떤 active-socket 보장까지 제공하지는 않습니까?
- cleanup 실패를 로컬 저장 실패와 같은 메시지로 표시하면 왜 부정확합니까?

### 30초 모범 답변

AbortSignal은 transport에 취소 의사를 전달할 뿐, 이미 resolve된 응답이나 늦은 body callback이 저장소 전이를 실행하지 않는다는 보장은 아닙니다. 그래서 동기화 owner를 queue 진입 전에 만들고 token이 맞는 network event만 현재 controller를 abort하게 합니다. 각 await와 ACK·conflict·snapshot COMMIT 직전에도 signal이 active인지 확인해야 합니다. Dispose는 listener 제거와 controller abort를 즉시 수행하고, native registration의 성공·실패와 무관하게 같은 token의 stop을 한 번만 호출합니다. Idle connection eviction은 stale 재사용을 줄이지만 모든 active socket을 강제로 종료한다는 보장은 아닙니다.

### 답변 핵심 키워드

cancellation ownership, AbortController, token correlation, observer-before-queue, late result guard, idempotent dispose, exact cleanup, backpressure, idle socket eviction

### 백지 구현

#### 구현 목표

고유 token으로 native network observer와 JS event listener를 묶고, 초기 offline·후속 network loss·dispose를 하나의 AbortSignal로 표현하는 observation 객체를 구현한다.

#### 인터페이스 또는 함수 시그니처

```ts
export interface ForegroundNetworkBridge {
  observeForegroundNetwork(token: string): Promise<boolean>;
  stopObservingForegroundNetwork(token: string): Promise<void>;
}

export interface Subscription {
  remove(): void;
}

export interface OfflineEvents {
  addListener(
    name: 'ForegroundSyncOffline',
    listener: (event: {token: string}) => void,
  ): Subscription;
}

export type ForegroundObservation = {
  token: string;
  signal: AbortSignal;
  ready: Promise<void>;
  dispose(): Promise<void>;
};

export function createForegroundObservation(
  bridge: ForegroundNetworkBridge,
  events: OfflineEvents,
  nextToken: () => string,
): ForegroundObservation {
  // 직접 구현
}

export function assertActive(signal: AbortSignal): void {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: native observer bridge, offline event source, 충돌하지 않는 token 공급자
- 출력: token, signal, registration 완료 Promise, idempotent dispose를 가진 owner
- `ready`는 observer 등록 결과가 connected이면 resolve하고, offline이면 signal을 abort한 뒤 완료한다.

#### 반드시 만족해야 할 조건

- token과 listener는 native registration을 기다리기 전에 동기적으로 만들어진다.
- offline event의 token이 현재 owner와 정확히 같을 때만 abort한다.
- initial registration이 `false`를 반환하면 signal을 abort한다.
- dispose는 호출 즉시 disposed 상태 설정, listener 제거, signal abort를 수행한다.
- dispose는 여러 번 호출해도 동일 cleanup Promise를 반환하고 native stop을 한 번만 실행한다.
- dispose가 registration 완료 전 호출되어도 registration resolve/reject 후 exact token stop을 시도한다.
- 등록 실패 뒤에도 stop 호출 정책이 일관되고 unhandled rejection이 남지 않는다.
- 오래된 token event는 새 observation을 취소하지 않는다.
- sync coordinator는 HTTP 전후, body parse 후, ACK/conflict/snapshot 저장 직전에 `assertActive`를 호출한다.

#### 경계 조건

- 시작부터 offline
- queue에서 기다리는 동안 offline
- HTTP 요청 중 offline
- status 수신 뒤 body parsing 중 offline
- ACK 직전 offline
- dispose가 registration보다 먼저 호출
- registration reject
- stop reject
- 이전 token의 늦은 event
- dispose 중복 호출

#### 실패 조건

- listener를 registration 후에 설치해 초기 event를 놓침
- token을 비교하지 않아 이전 작업이 새 작업을 취소
- abort된 뒤 ACK·conflict·snapshot COMMIT 시작
- dispose마다 native stop을 중복 호출
- listener 또는 native observer 누수
- idle socket eviction을 모든 active 요청의 강제 취소라고 과장

#### 필요한 제약

- 실제 fetch 구현은 `signal`을 전달받는다고 가정한다.
- native connection pool 처리는 bridge 밖의 구현으로 남기며, 이 함수는 관찰 소유권만 관리한다.
- stop 오류를 호출자에게 전달할지 별도 cleanup 상태로 기록할지 정책을 명시한다.

### 구현 후 자가 검증

- 정상 경로: connected 등록 뒤 dispose까지 abort되지 않고 exact token이 사용된다.
- 초기 offline: ready 이후 signal이 aborted다.
- token 격리: 이전 token event가 현재 signal을 바꾸지 않는다.
- lifecycle: early dispose의 listener 제거·abort·native stop이 각각 한 번이다.
- 실패 경로: observe resolve/reject와 stop resolve/reject 네 조합에서 누수가 없다.
- 동시성: queued 요청과 active 요청 모두 같은 signal을 관찰한다.
- invariant: signal abort 후 어떤 저장소 상태 전이도 시작되지 않는다.
- resource cleanup: listener와 native observer의 생성·해제 수가 맞는다.

### 구현 후 설명할 것

1. observer를 serial queue 앞에서 소유해야 하는 이유
2. transport abort와 domain commit guard가 둘 다 필요한 이유
3. token correlation으로 오래된 event를 차단한 방법
4. early dispose와 registration rejection을 하나의 cleanup 경로로 만든 이유
5. idle connection eviction의 효과와 한계

### 원본 확인 위치

- Thread: M14
- 커밋: `feat: cancel foreground uploads on network loss`
- 파일: `src/backgroundSync.ts`, `src/sync.ts`, `src/App.tsx`, `__tests__/App.test.tsx`, `fixture/server.cjs`, `scripts/verify_m14.py`
- 함수·상태: `observeForegroundSync`, `serializeSync`, `AbortController`, `ForegroundSyncOffline`
- 관련 Thread: M08, M10
