# 클라이언트 상태, URL과 서버 렌더링 면접 워크북

Thread 06·07·08은 UI 프레임워크 사용법보다 비동기 상태의 권위와 수명에 초점을 둔다. 동기 admission, 늦은 응답 무효화, URL query identity, 인증된 SSR cache 경계가 하나의 상태 흐름으로 이어진다.

## 문서 내 면접 포인트

- [P17 첫 `await` 전에 수행하는 동기식 중복 mutation 차단](#p17)
- [P18 generation·query version으로 늦은 비동기 응답 무효화](#p18)
- [P19 서버 query 조건을 URL과 cache identity에 일치시키기](#p19)
- [P20 인증된 SSR의 no-store, coherent initial payload와 account cache 경계](#p20)

---

<a id="p17"></a>
## [Thread 06 (E06) / 제목 미노출 — 기록상 단일 server-state owner와 mutation admission 구현] 첫 `await` 전에 수행하는 동기식 중복 mutation 차단

### 면접 질문

- 버튼의 `disabled` 속성만으로 중복 submit을 막을 수 없는 실제 반례를 설명해 보세요.
  - 꼬리 질문: `pending`을 React state가 아니라 동기적으로 갱신되는 ref 기반 Set에도 기록한 이유는 무엇입니까?
  - 꼬리 질문: 모든 mutation을 전역 직렬화하지 않고 Monitor별 conflict key를 둔 이유는 무엇입니까?

### 30초 모범 답변

렌더된 버튼이 disabled여도 같은 form의 `requestSubmit()`이나 이미 큐에 들어온 이벤트가 두 번째 handler를 실행할 수 있고, React state 반영 전 같은 tick에 두 호출이 모두 통과할 수 있습니다. 그래서 handler 시작 직후, CSRF fetch나 네트워크 전 첫 `await`보다 앞에서 ref Set을 검사·추가해 동기 admission을 결정합니다. 같은 Monitor의 충돌 작업은 한 key로 막고, 서로 다른 Monitor와 create는 병렬로 허용합니다. logout은 account boundary이므로 진행 중 mutation과 서로 충돌시킵니다.

### 답변 핵심 키워드

- synchronous admission
- before first await
- disabled is presentation
- ref Set
- conflict key
- per-resource concurrency
- logout barrier
- finally cleanup

### 백지 구현

#### 구현 목표

비동기 작업을 시작하기 전에 동기적으로 충돌 여부를 판정하고, 작업 종료 시 반드시 해제하는 mutation gate를 작성한다.
#### 인터페이스 또는 함수 시그니처

`tryEnter(action): AdmissionToken | null`, `leave(token): void`, `clear(): void`
#### 입력과 출력

- 입력: create, logout, 특정 resource의 update/delete/check action
- 출력: 허용된 작업의 token 또는 거부를 뜻하는 `null`
- gate 상태: 현재 진행 중인 conflict key 집합
#### 반드시 만족해야 할 조건

- 검사와 등록은 같은 동기 호출 안에서 첫 `await` 전에 끝난다.
- 같은 resource의 충돌 mutation은 하나만 허용한다.
- 서로 다른 resource의 mutation은 동시에 허용한다.
- logout은 다른 모든 진행 작업과 상호 배타적이다.
- 성공·실패·throw·취소 경로에서 token을 정확히 한 번 해제한다.
- UI disabled 여부와 무관하게 gate가 correctness를 보장한다.
#### 경계 조건

- 같은 call stack에서 두 번 연속 `tryEnter`
- 작업 시작 직후 동기 throw
- network rejection
- logout과 resource mutation이 거의 동시에 시작
- `clear` 뒤 늦은 `leave` 호출
#### 실패 조건

- React render가 반영된 뒤에야 pending을 등록한다.
- 전역 boolean 하나로 모든 독립 작업을 막는다.
- `finally` 누락으로 영구 pending이 된다.
- 오래된 token이 새 작업의 key를 해제한다.
#### 필요한 제약

- gate 자체는 React에 의존하지 않는 순수 객체로 구현한다.
- 서버 멱등성을 대체하지 않으며 브라우저 내 중복 admission만 다룬다.

```ts
type MutationAction =
  | { kind: 'create' }
  | { kind: 'logout' }
  | { kind: 'update' | 'delete' | 'check'; id: string };

export class MutationGate {
  tryEnter(action: MutationAction): AdmissionToken | null {
    // 직접 구현
  }

  leave(token: AdmissionToken): void {
    // 직접 구현
  }

  clear(): void {
    // 직접 구현
  }
}
```

### 구현 후 자가 검증

- 동일 action을 같은 tick에 두 번 호출하면 하나만 허용된다.
- 같은 Monitor의 update와 delete는 충돌한다.
- 서로 다른 Monitor의 작업은 동시에 들어간다.
- create와 resource mutation의 정책이 요구사항과 맞다.
- logout과 기존 pending이 양방향으로 충돌한다.
- 성공·실패·동기 throw 후 gate가 다시 열리고, stale token은 새 작업을 해제하지 못한다.

### 구현 후 설명할 것

- disabled UI와 correctness gate를 분리한 이유
- ref/동기 자료구조가 React state보다 admission에 적합한 이유
- conflict key의 granularity를 정한 기준
- 클라이언트 gate와 서버 idempotency가 함께 필요한 이유

### 원본 확인 위치

- Thread 06
- `app/monitors/use-monitors.ts` — `useMonitors`, `mutate`, `pending`
- `evidence/E06/browser-scenario.ts` — held response 뒤 `requestSubmit()` 반례
- `test/browser/server-state.spec.ts`
- 관련 Thread: 05(E05) mutation 전 CSRF, 10(E10) 서버 idempotency
---

<a id="p18"></a>
## [Thread 06 (E06) / 제목 미노출 — 기록상 cache invalidation과 stale response 억제 구현] generation·query version으로 늦은 비동기 응답 무효화

### 면접 질문

- logout 뒤 이전 계정의 fetch 응답이 도착해도 새 화면 상태를 덮지 못하게 한 방법을 설명해 보세요.
  - 꼬리 질문: Monitor 삭제 또는 Check 완료 뒤 이전 history 응답이 늦게 도착하는 문제를 어떻게 막았습니까?
  - 꼬리 질문: Check POST는 commit됐지만 뒤따른 history refresh가 실패한 경우 mutation 자체를 실패로 표시하면 왜 잘못입니까?

### 30초 모범 답변

비동기 응답의 유효성은 요청을 보낸 시점이 아니라 적용 시점의 state epoch와 query version으로 판단해야 합니다. session lifetime generation을 캡처해 logout·unmount에서 증가시키면 이전 계정 응답을 무시할 수 있습니다. history는 Monitor별 version과 query string을 함께 기록해 삭제·filter 변경·새 Check 뒤 이전 read가 복원되지 않게 합니다. POST acknowledgement와 후속 refresh는 다른 작업이므로 commit된 mutation은 성공으로 유지하고 refresh 오류는 query 상태로만 표시해야 실제 서버 사실을 뒤집지 않습니다.

### 답변 핵심 키워드

- generation token
- epoch
- per-query version
- stale response suppression
- query identity
- invalidation
- commit ack vs refresh
- account boundary

### 백지 구현

#### 구현 목표

scope별 version token을 발급하고 reset·invalidate 이후 늦은 결과가 적용되지 않게 하는 비동기 결과 gate를 작성한다.
#### 인터페이스 또는 함수 시그니처

`begin(scope, identity): RequestToken`, `isCurrent(token): boolean`, `invalidate(scope): void`, `resetAll(): void`
#### 입력과 출력

- 입력: session scope 또는 Monitor history scope, query identity
- 출력: 요청 시점 generation/version을 담은 token과 적용 가능 여부
- 사용처: fetch 성공·실패 모두 state update 직전에 확인
#### 반드시 만족해야 할 조건

- `resetAll` 이후 기존 모든 token은 무효다.
- scope invalidate는 해당 scope의 기존 token만 무효화한다.
- 같은 scope라도 query identity가 바뀌면 이전 응답을 적용하지 않는다.
- 성공과 오류 응답 모두 current 검사를 통과해야 state에 반영된다.
- mutation commit 성공과 refresh 결과를 서로 다른 상태로 저장한다.
#### 경계 조건

- logout 직전 시작한 collection fetch
- 삭제 전 시작한 history fetch
- filter A 응답을 hold한 뒤 filter B 요청 완료
- Check commit 뒤 refresh 실패
- 동일 query의 연속 reload
#### 실패 조건

- 응답 도착 순서만 믿고 마지막 callback을 적용한다.
- 오류 응답에는 current 검사를 생략한다.
- global version 하나로 독립 Monitor query를 모두 무효화한다.
- refresh 실패가 이미 commit된 mutation을 실패로 되돌린다.
#### 필요한 제약

- 요청 취소는 최적화일 뿐 correctness는 token 검사로 보장한다.
- 한 Monitor에는 현재 요청한 history page 하나만 cache한다는 프로젝트 범위를 따른다.

```ts
export class AsyncVersionGate {
  begin(scope: string, identity: string): RequestToken {
    // 직접 구현
  }

  isCurrent(token: RequestToken): boolean {
    // 직접 구현
  }

  invalidate(scope: string): void {
    // 직접 구현
  }

  resetAll(): void {
    // 직접 구현
  }
}
```

### 구현 후 자가 검증

- logout/reset 뒤 과거 success와 error가 모두 무시된다.
- 한 Monitor invalidate가 다른 Monitor 응답을 막지 않는다.
- filter A의 늦은 응답이 filter B 데이터를 덮지 않는다.
- 삭제된 Monitor history가 복원되지 않는다.
- Check mutation은 성공, refresh는 오류인 상태를 동시에 표현할 수 있다.
- 취소 기능이 없어도 state correctness가 유지된다.

### 구현 후 설명할 것

- AbortController만으로 충분하지 않은 이유
- session generation과 query version을 분리한 이유
- query identity에 search string을 포함한 이유
- mutation 상태와 read 상태를 분리한 이유

### 원본 확인 위치

- Thread 06
- `app/monitors/use-monitors.ts` — `lifetime`, `historyVersions`, `historySearches`, `loadHistory`, `clearSession`
- `test/browser/server-state.spec.ts`
- 관련 Thread: 07(E07) URL query identity, 08(E08) account-boundary document replacement
---

<a id="p19"></a>
## [Thread 07 (E07) / 제목 미노출 — 기록상 history URL state와 브라우저 navigation 구현] 서버 query 조건을 URL과 cache identity에 일치시키기

### 면접 질문

- 선택한 Monitor, state filter, limit, cursor를 component local state가 아니라 URL에 둔 이유는 무엇입니까?
  - 꼬리 질문: filter가 바뀔 때 기존 cursor를 제거해야 하는 이유는 무엇입니까?
  - 꼬리 질문: `pushState`, `useSearchParams`, fetch cache key가 서로 다른 조건을 가리키지 않게 하려면 어떤 invariant가 필요합니까?

### 30초 모범 답변

history 조건을 URL에 두면 직접 링크, reload, back/forward가 같은 서버 query를 재구성할 수 있고 사용자 탐색 기록과 UI 상태가 일치합니다. cursor는 monitor·state·limit에 결박된 위치이므로 filter나 page size가 바뀌면 반드시 제거해야 합니다. URL에서 파생한 canonical search string을 fetch identity와 client cache key로 함께 사용하고, 변경 시 이전 page 데이터를 비우거나 version을 올립니다. `pushState`는 새 탐색을 기록하고 popstate로 돌아왔을 때 같은 조건을 다시 읽게 해야 합니다.

### 답변 핵심 키워드

- URL as state
- deep link
- back/forward
- query identity
- cursor reset
- canonical search
- pushState
- single source of navigation truth

### 백지 구현

#### 구현 목표

현재 query string에서 history 선택을 파싱하고, 사용자 변경을 반영한 다음 URL을 생성하는 순수 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`parseHistoryLocation(search): HistorySelection`, `updateHistoryLocation(search, change): string`
#### 입력과 출력

- 입력: 현재 query string과 monitor/state/limit/cursor 변경
- 출력: 정규화된 selection 또는 새 query string
- 브라우저 계층: 실제 navigation에서 `pushState`를 한 번 호출
#### 반드시 만족해야 할 조건

- monitor, state, limit, cursor 네 조건을 URL에서 복원한다.
- monitor·state·limit 변경은 기존 cursor를 제거한다.
- Next page는 cursor를 추가하고 First page는 cursor를 제거한다.
- history와 무관한 query parameter가 있다면 손실하지 않는다.
- 생성된 search string이 fetch identity와 cache identity에 그대로 사용된다.
- back/forward에서는 새 history entry를 만들지 않고 현재 URL을 반영한다.
#### 경계 조건

- monitor 없음
- 유효하지 않은 state·limit·cursor
- 동일 변경을 반복 적용
- percent-encoding과 parameter 순서
- 삭제한 Monitor가 URL에 남아 있는 경우
#### 실패 조건

- local state와 URL을 서로 독립적으로 갱신한다.
- filter 변경 뒤 이전 cursor를 유지한다.
- render마다 `pushState`를 호출해 history loop를 만든다.
- URL은 바뀌었지만 fetch가 이전 identity를 사용한다.
#### 필요한 제약

- 서버의 `historyQuery`가 최종 유효성 권위자이며 클라이언트 파서는 편의 계층이다.
- 프레임워크 router 전체 재설계 없이 현재 `useSearchParams` 경계에 맞춘다.

```ts
type HistorySelection = {
  monitorId: string | null;
  state: 'SUCCEEDED' | 'FAILED' | null;
  limit: number | null;
  cursor: string | null;
};

export function parseHistoryLocation(search: string): HistorySelection {
  // 직접 구현
}

export function updateHistoryLocation(
  search: string,
  change: HistoryLocationChange,
): string {
  // 직접 구현
}
```

### 구현 후 자가 검증

- 직접 URL, reload, back, forward가 같은 조건을 복원한다.
- state·limit·monitor 변경 시 cursor가 사라진다.
- Next와 First가 예상한 history entry를 만든다.
- 알 수 없는 서버 cursor는 클라이언트가 신뢰하지 않고 서버 오류를 처리한다.
- query identity 변경 뒤 이전 page 응답이 덮어쓰지 못한다.
- Monitor 삭제 뒤 URL과 열린 history가 정리된다.

### 구현 후 설명할 것

- URL을 탐색 상태의 권위 원천으로 둔 이유
- `pushState`와 router navigation의 trade-off
- query string 전체를 cache identity로 사용한 이유
- 클라이언트 검증과 서버 검증의 역할 차이

### 원본 확인 위치

- Thread 07
- Thread 07 당시 `app/monitors/page.tsx` — `useSearchParams`, history navigation
- 현재 `app/monitors/monitor-workspace.tsx` — `useSearchParams`, history 선택과 `pushState` navigation
- `app/monitors/initial-state.ts` — `historyLocation`
- `app/monitors/use-monitors.ts` — `historySearches`, query별 `loadHistory`
- `test/browser/history.spec.ts`
- 관련 Thread: 06(E06) stale response version, 08(E08) server initial payload와 client island 분리
---

<a id="p20"></a>
## [Thread 08 (E08) / 제목 미노출 — 기록상 authenticated Server Component와 hydration 경계 구현] 인증된 SSR의 no-store, coherent initial payload와 account cache 경계

### 면접 질문

- 인증된 `/monitors`를 Server Component로 바꾸면서 `headers()`와 `cache: 'no-store'`를 사용한 이유는 무엇입니까?
  - 꼬리 질문: Monitor collection을 읽은 뒤 selected history를 읽는 사이 session이 사라지면 두 결과를 함께 버리고 login으로 보낸 이유는 무엇입니까?
  - 꼬리 질문: login/logout에서 client router 전환보다 document replacement를 선택한 이유를 account-boundary cache 관점에서 설명해 보세요.

### 30초 모범 답변

인증된 HTML과 RSC는 사용자별 데이터이므로 static/shared cache에 들어가면 계정 간 노출 위험이 있습니다. 요청 시 `headers()`로 현재 Cookie를 읽고 trusted API에 Cookie와 필요한 Origin만 전달하며 모든 fetch를 no-store로 둡니다. 초기 collection과 선택 history는 한 화면 payload이므로 중간 session loss가 나면 일부만 렌더하지 않고 전부 버려 login으로 이동합니다. client는 이 공개 payload로 바로 초기화해 hydration 중 중복 GET을 피하고, login/logout은 document replacement로 이전 계정의 route·RSC client cache를 폐기합니다.

### 답변 핵심 키워드

- authenticated SSR
- request-time headers
- no-store
- HTML/RSC privacy
- coherent initial payload
- hydration without refetch
- client islands
- account cache eviction

### 백지 구현

#### 구현 목표

현재 요청의 인증 문맥으로 initial Monitor와 선택 history를 읽고, 안전한 공개 payload 또는 redirect를 반환하는 server loader를 작성한다.
#### 인터페이스 또는 함수 시그니처

`loadInitialWorkspace(requestHeaders, selectedQuery): Promise<InitialWorkspace | Redirect>`
#### 입력과 출력

- 입력: 현재 요청 Headers, 선택된 history query, trusted API base
- 출력: 공개 Monitor/history payload 또는 login redirect
- 클라이언트에는 cookie, session hash, password, CSRF 값을 전달하지 않음
#### 반드시 만족해야 할 조건

- 현재 요청의 raw Cookie와 필요할 때 Origin만 trusted API로 전달한다.
- 모든 인증 데이터 fetch는 shared cache를 사용하지 않는다.
- collection과 selected history 중 하나가 인증 상실을 알리면 둘 다 폐기한다.
- 다른 사용자의 데이터나 credential 값이 HTML/RSC props에 직렬화되지 않는다.
- client state owner는 initial payload로 바로 시작하고 같은 collection/history를 hydration 직후 다시 읽지 않는다.
- account 전환은 이전 client route cache를 남기지 않는다.
#### 경계 조건

- Cookie 없음 또는 중복 cookie
- collection 성공 후 history에서 401
- selected Monitor가 없음 또는 foreign
- JavaScript 비활성 상태의 initial HTML
- hydration 도중 logout
#### 실패 조건

- 인증 HTML을 장기 shared cache에 저장한다.
- 서버가 session token을 client prop으로 넘긴다.
- collection만 보여 주고 history 401을 별도 오류로 렌더해 혼합 계정 화면을 만든다.
- hydration 직후 무조건 같은 GET을 다시 보내며 loading shell로 교체한다.
#### 필요한 제약

- Fastify API가 인증·owner 판정의 권위 원천으로 남는다.
- 브라우저 API는 handler/effect에서만 사용하고 Server Component에서 참조하지 않는다.

```ts
export async function loadInitialWorkspace(
  requestHeaders: Headers,
  selectedQuery: string | null,
): Promise<InitialWorkspace | RedirectResult> {
  // 직접 구현
}
```

### 구현 후 자가 검증

- JavaScript 없이도 owner의 Monitor와 선택 history가 initial HTML에 있다.
- 응답과 RSC payload에 cookie, token hash, password, CSRF 값이 없다.
- 초기 hydration 동안 collection/history 중복 GET이 0이다.
- collection 후 session loss를 주입하면 일부 private payload 없이 login으로 간다.
- 다른 owner의 데이터가 HTML·RSC·client state에 나타나지 않는다.
- login/logout 뒤 이전 계정 화면이 browser back 또는 client cache에서 재사용되지 않는다.

### 구현 후 설명할 것

- Server Component와 Client Component 경계를 나눈 기준
- `no-store`와 요청 시 header 접근이 필요한 이유
- 여러 server read를 하나의 화면 단위로 취급한 이유
- document replacement의 비용과 account cache 안전성 이점

### 원본 확인 위치

- Thread 08
- `app/monitors/page.tsx` — Server Component 조합과 인증 실패 redirect 경계
- `app/monitors/server-data.ts` — `readInitialMonitors`, request-time `headers()`, `cache: 'no-store'`, collection/history 일관성 처리
- `app/monitors/initial-state.ts` — `InitialMonitors`, `historyLocation`
- `app/monitors/monitor-workspace.tsx` — client island, URL navigation과 account-boundary document replacement
- `app/monitors/use-monitors.ts` — initial public payload로 client state 초기화
- `test/browser/rendering.spec.ts`
- `test/e08-rsc-transport.mjs`
- 관련 Thread: 04(E04) session, 05(E05) owner privacy, 06·07 client state
