# 개발자 기술면접 워크북 03 — 클라이언트 상태, 페이지네이션, 서버 렌더링

이 문서는 브라우저에서 발생하는 중복 요청과 응답 역전, 커서 페이지네이션의 정렬 invariant, URL 상태 복원, 요청별 SSR 보안 경계를 다룬다. React API 암기보다 "어떤 상태가 권위 있는가"를 설명하는 데 초점을 둔다.

---

<!-- coverage: SA-08 -->
<a id="sa-08"></a>
## [Thread 06 / `fix(state): own monitor data and pending mutations together`] 렌더 전에 닫아야 하는 중복 요청 창

### 면접 질문

버튼을 `disabled`로 만드는 것만으로 빠른 이중 제출을 막지 못했던 이유는 무엇입니까? React 상태와 별도로 동기적인 `Set`을 사용해 operation key를 선점한 이유를 설명해 주세요.

꼬리 질문:

- 첫 POST가 서버에서 이미 커밋됐지만 응답만 보류된 상태에서 두 번째 submit이 오면 어떤 문제가 생깁니까?
- create, Monitor별 수정·삭제·Check, logout은 어떤 operation key 관계를 가져야 합니까?
- 서로 무관한 create와 특정 Monitor 수정은 동시에 허용할 수 있습니까?
- 실패한 update가 성공한 적 없는 낙관적 데이터를 남기지 않게 하려면 어떻게 해야 합니까?

### 30초 모범 답변

React 상태 갱신은 즉시 렌더에 반영되지 않기 때문에 첫 submit handler가 끝나기 전에 두 번째 이벤트가 들어오는 짧은 창이 있습니다. 버튼 비활성화는 다음 렌더 이후의 UI만 막습니다. 그래서 handler 진입 시 `useRef`의 `Set`에 operation key를 동기적으로 추가해 같은 키의 요청을 즉시 거부하고, 화면용 pending 상태는 별도로 갱신했습니다. 서버 응답만 권위 있는 데이터로 반영하고, 실패하면 기존 캐시를 유지합니다. Monitor별 작업은 같은 키로 직렬화하되 독립 create는 허용하고, logout은 전체 작업과 충돌하도록 설계했습니다.

### 답변 핵심 키워드

render gap, synchronous guard, operation key, in-flight Set, authoritative response, no optimistic fabrication, independent pending, finally cleanup

### 백지 구현

#### 구현 목표

동일 operation key의 중복 비동기 작업을 막고, 서로 독립된 키는 병렬로 수행하며, 성공·실패·정리 상태를 일관되게 관리하는 작은 실행기를 구현한다.

#### 인터페이스 또는 함수 시그니처

```ts
type Phase = 'pending' | 'success' | 'failure';

type OperationState = Record<string, Phase>;

async function perform(
  key: string,
  work: () => Promise<void>,
): Promise<boolean> {
  // 직접 구현
}
```

연습 환경에는 다음 저장소가 있다고 가정한다.

```ts
const inFlight = new Set<string>();
let operations: OperationState = {};
let loaded = true;
```

#### 입력과 출력

- 입력: 작업 충돌 범위를 나타내는 key와 실제 비동기 작업
- 출력: 작업을 시작해 성공했으면 `true`, 시작하지 못했거나 실패하면 `false`
- 부수 효과: 해당 key의 phase와 오류 상태 갱신

#### 반드시 만족해야 할 조건

- 동일 key는 첫 작업이 끝나기 전 두 번째 작업을 시작하지 않는다.
- 동기 이벤트가 연속으로 들어와도 네트워크 호출은 하나만 발생한다.
- `logout`이 진행 중이면 다른 작업을 시작하지 않는다.
- 정책상 필요한 경우 다른 작업이 진행 중일 때 logout도 거부한다.
- 성공·실패와 관계없이 `inFlight`는 정리된다.
- 실패는 기존 서버 데이터와 다른 key의 pending 상태를 변경하지 않는다.

#### 경계 조건

- 같은 tick 안의 연속 두 호출
- `work`가 동기적으로 예외를 던짐
- `work`가 오래 pending인 동안 독립 key 호출
- logout과 다른 key의 동시 호출
- 컴포넌트 언마운트 또는 세션 만료
- 성공 직후 동일 key 재실행

#### 실패 조건

- 렌더된 disabled 값만 검사해 중복을 막지 않는다.
- 요청을 시작한 뒤에야 `inFlight`를 등록하지 않는다.
- 한 작업 실패가 전체 operation map을 초기화하지 않는다.
- 실패한 응답으로 서버 데이터 캐시를 추측 갱신하지 않는다.

#### 필요한 제약

- 상태 관리 라이브러리 없이 15~20분 안에 구현한다.
- operation key 생성 규칙은 호출자가 제공한다고 가정한다.

### 구현 후 자가 검증

- [ ] 동기적으로 두 번 호출해도 `work`가 한 번만 실행된다.
- [ ] 서로 다른 key는 정책대로 동시에 실행된다.
- [ ] logout 충돌 규칙이 양방향으로 적용된다.
- [ ] 성공·실패·예외 모두 `inFlight`를 제거한다.
- [ ] 실패 후 기존 canonical 데이터가 유지된다.
- [ ] 다른 key의 pending 표시가 사라지지 않는다.
- [ ] operation map과 동기 guard의 역할이 구분된다.
- [ ] key 조회·삽입·삭제는 평균 O(1)이다.

### 구현 후 설명할 것

1. UI의 disabled와 동기 guard가 해결하는 문제가 다른 이유
2. operation key의 충돌 범위를 정한 기준
3. 낙관적 갱신을 사용하지 않은 이유
4. 관련 없는 pending과 오류를 독립적으로 보존한 방식
5. 프런트 중복 방지가 서버 idempotency를 대체하지 못하는 이유

### 원본 확인 위치

- Thread 06
- 커밋: `fix(state): own monitor data and pending mutations together`
- `app/monitors/use-monitor-state.ts`
- `useMonitorState`, 내부 `perform`
- `app/monitors/page.tsx`
- `tests/browser/server-state.spec.ts`
- 관련 Thread: Thread 10의 데이터베이스 idempotency, Thread 07의 stale 응답 거부

---

<!-- coverage: SA-09 -->
<a id="sa-09"></a>
## [Thread 07 / `feat(history): bound owner history with stable cursors`] 복합 정렬 키를 보존하는 keyset pagination

### 면접 질문

CheckRun 이력을 `finishedAt DESC, id DESC`로 정렬하면서 offset이 아니라 keyset cursor를 사용한 이유는 무엇입니까? 동률 시각에서 ID를 두 번째 정렬 키로 넣지 않으면 어떤 누락이나 중복이 생깁니까?

꼬리 질문:

- 다음 페이지 조건이 단순히 `finishedAt < cursorTime`이면 무엇이 잘못됩니까?
- 커서에 Monitor ID, 필터, limit를 결합한 이유는 무엇입니까?
- 페이지 사이에 더 최신 행이 삽입돼도 기존 커서 연속성이 유지되는 이유는 무엇입니까?
- `limit + 1`개를 읽는 방식은 무엇을 판별하기 위한 것입니까?

### 30초 모범 답변

offset은 앞쪽에 새 행이 들어오면 페이지 기준이 이동해 중복이나 누락이 생기고, 깊은 페이지일수록 버린 행을 많이 스캔합니다. 이력은 `finishedAt DESC, id DESC`라는 완전한 순서를 만들고, 다음 페이지는 `finishedAt`이 더 작거나 같은 시각에서 `id`가 더 작은 행만 읽었습니다. 커서에는 버전, Monitor, 필터, limit, 마지막 정렬 키를 넣어 다른 쿼리에 재사용하지 못하게 했습니다. `limit + 1`개를 읽어 다음 페이지 존재 여부를 판단하고 실제 응답은 limit까지만 반환합니다.

### 답변 핵심 키워드

keyset pagination, total order, composite cursor, tie breaker, cursor binding, limit+1, insert stability, no offset

### 백지 구현

#### 구현 목표

`finishedAt DESC, id DESC` 순서를 사용하는 이력 API의 커서 인코딩·검증과 다음 페이지 조건을 설계한다. 데이터베이스 접근 코드는 조건 객체 또는 SQL 조각으로 표현해도 된다.

#### 인터페이스 또는 함수 시그니처

```java
record HistoryWindow(
    int limit,
    String state,
    Instant beforeTime,
    UUID beforeId
) {}

static HistoryWindow parse(
    UUID monitorId,
    String rawLimit,
    String state,
    String cursor
) {
    // 직접 구현
}

static String nextCursor(
    UUID monitorId,
    HistoryWindow window,
    CheckRun last
) {
    // 직접 구현
}
```

#### 입력과 출력

- 입력: Monitor ID, 선택적 limit·state·cursor
- 출력: 검증된 쿼리 창과 다음 cursor
- 공개 실패: 잘못된 limit·state·cursor를 같은 입력 오류로 처리

#### 반드시 만족해야 할 조건

- 기본 limit는 20, 허용 범위는 1~100이다.
- state는 없거나 `SUCCEEDED`, `FAILED` 중 하나다.
- 커서는 버전, Monitor ID, state, limit, 마지막 `finishedAt`, 마지막 ID를 포함한다.
- 커서의 Monitor·state·limit가 현재 요청과 다르면 거부한다.
- 정렬은 `finishedAt DESC, id DESC`로 완전해야 한다.
- 다음 조건은 두 정렬 키의 사전식 순서를 정확히 반영한다.
- 조회는 최대 `limit + 1`행으로 제한한다.

#### 경계 조건

- 같은 `finishedAt`을 가진 여러 UUID
- 마지막 페이지가 정확히 limit개인 경우
- limit보다 한 개 많은 경우
- 첫 페이지에서 cursor 없음
- 잘못된 Base64, 필드 수, 버전, UUID, 시각
- 다른 Monitor나 필터에서 만든 cursor
- 페이지 사이 최신 행 삽입

#### 실패 조건

- offset을 cursor 내부에 넣지 않는다.
- 시각만으로 다음 페이지를 자르지 않는다.
- cursor를 권한 증명으로 사용하지 않는다.
- 잘못된 cursor를 첫 페이지로 조용히 되돌리지 않는다.
- 무제한 조회 후 메모리에서 자르지 않는다.

#### 필요한 제약

- cursor 서명·암호화는 요구하지 않는다. 변조는 검증 실패로 닫는다.
- 구현 시간은 25분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 동률 시각 행이 페이지를 넘어가도 중복·누락이 없다.
- [ ] 7개 행을 limit 3으로 읽으면 3·3·1이 된다.
- [ ] 페이지 사이 최신 행 삽입이 기존 연속 결과에 끼어들지 않는다.
- [ ] Monitor·state·limit가 다른 cursor가 거부된다.
- [ ] 마지막 페이지에서는 next cursor가 없다.
- [ ] 모든 쿼리가 owner 조건과 limit을 유지할 수 있다.
- [ ] 정렬·조건과 인덱스 선두 컬럼을 설명할 수 있다.
- [ ] 페이지 비용이 전체 앞쪽 행 수에 비례하지 않는다.

### 구현 후 설명할 것

1. 완전한 정렬 순서를 만든 이유
2. offset 대비 keyset의 삽입 안정성과 성능 특성
3. cursor를 쿼리 조건에 결합한 이유
4. cursor가 인가 수단이 아닌 이유
5. `limit + 1` 방식의 비용과 이점

### 원본 확인 위치

- Thread 07
- 커밋: `feat(history): bound owner history with stable cursors`
- `backend/src/main/java/dev/evolution/monitor/HistoryQuery.java`
- `HistoryQuery.parse`, `HistoryQuery.nextCursor`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `MonitorStore.historyPage`
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java`
- 관련 Thread: Thread 05의 owner scope, Thread 13의 실행 계획과 인덱스 선택

---

<!-- coverage: SA-10 -->
<a id="sa-10"></a>
## [Thread 07 / `feat(history): restore filter and page state from URLs`] URL을 선택 상태의 권위로 두고 오래된 응답 거부하기

### 면접 질문

선택한 Monitor, 필터, limit, cursor를 컴포넌트의 로컬 상태만이 아니라 URL query로 관리한 이유는 무엇입니까? 이전 필터 요청이 늦게 도착해 현재 화면을 덮어쓰는 문제는 어떻게 막아야 합니까?

꼬리 질문:

- 필터를 바꿀 때 cursor는 왜 지워야 합니까?
- back/forward와 reload에서 같은 행을 복원하려면 무엇이 URL에 있어야 합니까?
- AbortController만 사용하면 stale 응답 문제를 완전히 해결할 수 있습니까?
- 응답을 적용할 때 요청 시 selection과 현재 selection 중 어느 것을 비교해야 합니까?

### 30초 모범 답변

이력 선택을 URL에 두면 deep link, 새로고침, back/forward가 동일한 쿼리를 재구성할 수 있고 로컬 상태와 주소가 어긋나지 않습니다. 필터가 바뀌면 기존 cursor는 이전 쿼리에 결합돼 있으므로 제거해야 합니다. 네트워크 응답은 완료 순서가 보장되지 않으므로 각 요청이 시작될 때의 selection을 보존하고, 응답 시 현재 URL selection과 정확히 일치할 때만 상태에 적용합니다. 취소는 자원 절약에 도움은 되지만 이미 완료 중인 응답 경쟁을 모두 보장하지 않으므로 적용 시점의 권위 검사가 필요합니다.

### 답변 핵심 키워드

URL as state, deep link, navigation restoration, query identity, stale response, response ordering, compare-and-apply, cursor reset

### 백지 구현

#### 구현 목표

URL에서 이력 selection을 읽고, selection이 바뀔 때 URL을 갱신하며, 이전 요청의 늦은 응답이 현재 화면을 덮지 못하게 하는 조정기를 구현한다.

#### 인터페이스 또는 함수 시그니처

```ts
type HistorySelection = {
  monitorId: string;
  limit: string;
  state: string | null;
  cursor: string | null;
};

type HistoryPage = { items: CheckRun[]; nextCursor: string | null };

function sameSelection(a: HistorySelection | null, b: HistorySelection | null): boolean {
  // 직접 구현
}

function applyHistoryResponse(
  requested: HistorySelection,
  current: HistorySelection | null,
  page: HistoryPage,
): HistoryPage | null {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 요청 시작 시 selection, 응답 시 현재 URL selection, 응답 page
- 출력: 적용할 page 또는 무시를 뜻하는 `null`
- 별도 함수: URL query를 변경해 selection을 생성

#### 반드시 만족해야 할 조건

- selection의 네 필드를 모두 비교한다.
- 필터 또는 Monitor 변경 시 cursor를 제거한다.
- 다음 페이지 이동 시 서버가 준 cursor만 URL에 넣는다.
- back/forward와 reload는 URL만으로 동일 요청을 만든다.
- 이전 selection의 응답은 pending·error·rows를 현재 selection에 적용하지 않는다.
- 현재 요청 실패와 이전 요청 실패를 구분한다.

#### 경계 조건

- A 필터 요청을 보류한 채 All로 변경
- 더 새로운 All 응답이 먼저 완료되고 A 응답이 나중에 완료
- 같은 Monitor와 필터지만 다른 cursor
- 같은 cursor지만 다른 limit
- 선택된 Monitor 삭제
- URL에 부분적으로 잘못된 query 값이 들어옴

#### 실패 조건

- 응답 도착 순서만으로 최신성을 판단하지 않는다.
- cursor만 비교하고 필터·limit을 무시하지 않는다.
- 오래된 실패가 현재 성공 상태를 오류로 바꾸지 않는다.
- URL과 별도 로컬 선택 상태를 독립적으로 갱신해 이중 권위를 만들지 않는다.

#### 필요한 제약

- 라우터 라이브러리 세부 API는 생략한다.
- 순수 함수와 한 개의 비동기 호출 흐름으로 20분 안에 구현한다.

### 구현 후 자가 검증

- [ ] deep link에서 올바른 첫 화면이 만들어진다.
- [ ] 필터 변경 시 cursor가 사라지고 limit는 보존된다.
- [ ] back/forward와 reload가 같은 selection과 rows를 복원한다.
- [ ] 오래된 성공 응답이 최신 rows를 덮지 않는다.
- [ ] 오래된 실패가 최신 pending/error 상태를 바꾸지 않는다.
- [ ] Monitor 삭제 시 관련 query가 정리된다.
- [ ] selection 비교가 모든 필드를 포함한다.

### 구현 후 설명할 것

1. URL을 선택 상태의 단일 권위로 둔 이유
2. query identity를 구성하는 필드의 기준
3. AbortController와 응답 적용 검사의 역할 차이
4. 필터 변경 시 cursor를 무효화한 이유
5. navigation 복원성과 컴포넌트 복잡도 사이 trade-off

### 원본 확인 위치

- Thread 07
- 커밋: `feat(history): restore filter and page state from URLs`
- `app/monitors/api.ts`
- `HistorySelection`, `HistoryPage`, `loadChecks`
- `app/monitors/page.tsx`
- `app/monitors/use-monitor-state.ts`
- `tests/browser/history.spec.ts`
- 관련 Thread: Thread 06의 operation authority, Thread 08의 SSR 초기 selection

---

<!-- coverage: SA-11 -->
<a id="sa-11"></a>
## [Thread 08 / `feat(rendering): seed request-scoped server state for hydration`] 요청별 SSR 데이터와 hydration 보안 경계

### 면접 질문

인증된 Monitor 목록과 선택된 이력을 서버에서 먼저 렌더링하면서도, Alice의 HTML에 Bob 데이터나 세션·CSRF 비밀이 섞이지 않게 하려면 어떤 경계를 지켜야 합니까? 서버 초기 데이터와 hydration 후 클라이언트 상태가 달라지면 어떤 문제가 생깁니까?

꼬리 질문:

- 서버 컴포넌트가 API를 호출할 때 어떤 쿠키만 전달해야 합니까?
- 사용자별 HTML을 캐시하면 왜 위험합니까?
- SSR 직후 클라이언트가 같은 목록을 다시 가져오는 동안 오래된 응답이 오면 어떻게 처리해야 합니까?
- 초기 HTML과 RSC/클라이언트 props에 자격 증명이 없음을 어떻게 검증하겠습니까?

### 30초 모범 답변

SSR은 요청마다 현재 세션의 권한으로 데이터를 읽어야 하며 사용자 간 공유 캐시를 사용하면 안 됩니다. 서버 전용 로더가 요청 쿠키에서 필요한 세션 쿠키만 내부 API로 전달하고, 응답에서는 제품 데이터와 안정된 오류 코드만 클라이언트 props로 넘깁니다. 비밀번호, CSRF 토큰, 세션 값은 HTML과 RSC payload에 포함하지 않습니다. 클라이언트 상태는 서버가 준 initial state로 시작해 첫 hydration 결과가 같은 마크업을 만들게 하고, 이후 revalidation 응답도 현재 selection과 요청 세대를 확인해 오래된 데이터가 초기 권위를 덮지 못하게 합니다.

### 답변 핵심 키워드

request-scoped SSR, per-user authority, no-store, minimal cookie forwarding, secret-free props, hydration consistency, initial state, revalidation race

### 백지 구현

#### 구현 목표

요청 쿠키와 URL selection을 받아 사용자별 초기 Monitor 상태를 읽는 서버 전용 함수를 작성한다. 반환값은 브라우저로 직렬화해도 안전한 데이터만 포함해야 한다.

#### 인터페이스 또는 함수 시그니처

```ts
type InitialMonitorState = {
  monitors: MonitorView[];
  history: { selection: HistorySelection; page: HistoryPage } | null;
  loaded: boolean;
  error: ApiErrorCode | null;
};

async function loadInitialMonitorState(
  sessionCookie: string | null,
  selection: HistorySelection | null,
): Promise<InitialMonitorState> {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 현재 요청에서 추출한 세션 쿠키 값과 URL selection
- 출력: 직렬화 가능한 초기 제품 상태
- 실패: 인증 실패 또는 안전한 오류 코드가 있는 초기 상태

#### 반드시 만족해야 할 조건

- 함수는 서버에서만 실행된다.
- 내부 API 요청은 현재 요청의 필요한 세션 쿠키만 전달한다.
- 모든 사용자별 fetch는 공유 캐시 없이 수행한다.
- Monitor 목록과 선택된 history가 같은 요청 권한 아래에서 읽힌다.
- 다른 사용자 데이터, 비밀번호, CSRF 토큰, 전체 Cookie 헤더를 반환값에 포함하지 않는다.
- history 대상이 목록 권한과 맞지 않으면 안전한 오류 상태를 만든다.
- 클라이언트는 이 반환값으로 첫 렌더 상태를 초기화할 수 있다.

#### 경계 조건

- 인증 쿠키 없음
- 만료된 세션
- Alice와 Bob이 같은 URL 접근
- foreign Monitor가 URL에 있음
- 목록 성공 후 history 실패
- API가 깨진 성공 payload를 반환
- 클라이언트 revalidation이 SSR보다 늦거나 빠르게 완료

#### 실패 조건

- 전역 변수나 모듈 캐시에 사용자별 데이터를 저장하지 않는다.
- 전체 요청 헤더를 내부 API에 그대로 전달하지 않는다.
- 세션·CSRF 값을 클라이언트 props에 넣지 않는다.
- 인증 실패 시 이전 사용자의 초기 상태를 재사용하지 않는다.
- hydration 전후에 서로 다른 기본 selection을 사용하지 않는다.

#### 필요한 제약

- 프레임워크의 세부 캐시 API 대신 `cache: 'no-store'` 수준의 의도를 표현해도 된다.
- 구현 시간은 20~25분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] Alice와 Bob의 같은 URL HTML에 각자 데이터만 존재한다.
- [ ] 익명 HTML에 보호 데이터가 없다.
- [ ] HTML과 직렬화 props에 세션·CSRF·비밀번호가 없다.
- [ ] 서버 HTML에 목록과 선택 history가 JavaScript 전부터 존재한다.
- [ ] hydration 경고나 중복 `<main>`·`<h1>`가 없다.
- [ ] 초기 상태와 첫 클라이언트 상태가 동일하다.
- [ ] 오래된 revalidation 응답이 현재 selection을 덮지 않는다.
- [ ] 사용자별 응답이 공유 캐시에 남지 않는다.

### 구현 후 설명할 것

1. SSR이 성능 기능이면서 동시에 권한 경계인 이유
2. 요청 쿠키 중 최소한만 전달한 기준
3. 사용자별 `no-store`가 필요한 이유
4. secret-free initial props를 만든 방식
5. hydration 일치와 이후 revalidation을 분리한 설계

### 원본 확인 위치

- Thread 08
- 커밋: `feat(rendering): seed request-scoped server state for hydration`
- `app/monitors/server-data.ts`
- `loadInitialMonitorState`
- `app/monitors/page.tsx`
- `app/monitors/monitor-controls.tsx`
- `app/monitors/api.ts`
- `historyPath`, `readHistoryPage`, `readMonitors`
- `tests/browser/rendering.spec.ts`
- 관련 Thread: Thread 04의 세션, Thread 05의 소유권, Thread 07의 URL selection, Thread 06의 stale 응답 처리
