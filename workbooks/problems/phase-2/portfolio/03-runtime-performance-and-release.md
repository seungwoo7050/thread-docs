# 런타임·성능·릴리스 면접 워크북

이 문서는 타이머 lifecycle, 사용자 상호작용 측정, 빌드 산출물 예산, 컨테이너 검증, CI 의존 그래프를 묶는다. 공통 주제는 **성공 경로만 구현하지 않고 시간·프로세스·산출물·정리 단계까지 하나의 계약으로 다루는 것**이다.

<a id="p11"></a>
## [Thread 09 / `feat(home): 애니메이션 터미널 상호작용 추가`] 타이머 기반 유한 상태 기계와 effect 정리

### 면접 질문

명령어를 입력하고 잠시 유지한 뒤 지우는 애니메이션을 React effect로 구현할 때, 단순히 문자열 길이만 증감시키는 방식보다 명시적 phase 상태가 필요한 이유는 무엇인가? 타이머 lifecycle과 reduced-motion 정책을 포함해 설명하라.

꼬리 질문:

- `typing | hold | erase` phase가 없으면 전환 조건이 어디에 흩어지는가?
- effect dependency를 잘못 잡으면 stale closure나 타이머 중복이 어떻게 발생하는가?
- command 배열이 바뀌거나 빈 배열이 들어오면 인덱스 invariant를 어떻게 유지할 것인가?
- `setTimeout` cleanup을 빠뜨리면 unmount 뒤 어떤 현상이 생기는가?
- `prefers-reduced-motion`에서는 애니메이션을 느리게 할 것인가, 제거할 것인가?

### 30초 모범 답변

입력·유지·삭제는 지연 시간과 전환 조건이 다르므로 phase를 명시한 작은 상태 기계로 모델링하는 편이 안전합니다. 각 tick은 현재 command와 phase에서 다음 문자열·phase·인덱스를 계산하고, effect는 한 번에 타이머 하나만 예약한 뒤 cleanup에서 해제해야 합니다. command 인덱스는 배열 길이로 순환시키고 빈 배열을 별도로 처리합니다. reduced-motion 사용자는 장식 애니메이션을 건너뛰고 즉시 읽을 수 있는 안정된 결과를 제공하는 것이 적절합니다.

### 답변 핵심 키워드

finite-state machine, phase, transition, stale closure, one timer, cleanup, modulo, empty input, reduced motion, deterministic tick

### 백지 구현

**구현 목표**

프레임워크와 분리된 순수 상태 전이 함수와, 한 번에 하나의 타이머만 예약하는 controller를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type Phase = "typing" | "hold" | "erase";

type TerminalState = {
  commandIndex: number;
  visibleText: string;
  phase: Phase;
};

type Transition = {
  next: TerminalState;
  delayMs: number;
};

export function advanceTerminal(
  state: TerminalState,
  commands: readonly string[],
): Transition {
  // 직접 구현
}

export function startTerminalLoop(options: {
  commands: readonly string[];
  initialState: TerminalState;
  onState: (state: TerminalState) => void;
  reducedMotion: boolean;
}): () => void {
  // 직접 구현
}
```

**입력과 출력**

- `advanceTerminal`: 현재 상태와 command 목록을 받아 다음 상태와 지연 시간 반환
- `startTerminalLoop`: loop를 시작하고 모든 예약 작업을 중단하는 cleanup 반환

**반드시 만족해야 할 조건**

- phase별 전환 조건이 한 곳에 모인다.
- command 인덱스는 유효 범위를 벗어나지 않는다.
- 한 시점에 활성 타이머는 최대 하나다.
- cleanup 뒤에는 `onState`가 더 호출되지 않는다.
- 빈 command 목록을 안전하게 처리한다.
- reduced motion이면 반복 타이핑을 수행하지 않고 안정된 표시 상태를 제공한다.

**경계 조건**

- 빈 문자열 command
- 한 개 command만 있는 목록
- 마지막 command에서 첫 command로 순환
- hold 중 cleanup
- loop 실행 중 command 배열이 바뀌는 경우의 정책
- cleanup을 여러 번 호출하는 경우

**필요한 제약**

- DOM 렌더링은 범위 밖이다.
- `setInterval`보다 각 전환별 지연을 표현할 수 있는 구조를 선택한다.
- 구현 시간은 25~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] typing이 전체 문자열에 도달한 뒤 hold로 전환되는가?
- [ ] hold 뒤 erase, 빈 문자열 뒤 다음 command typing으로 전환되는가?
- [ ] 한 command와 빈 command 목록에서 무한 예외가 발생하지 않는가?
- [ ] command 인덱스가 항상 범위 안인가?
- [ ] cleanup 뒤 예약된 callback이 상태를 갱신하지 않는가?
- [ ] 타이머가 중복 예약되지 않는가?
- [ ] reduced-motion 경로가 애니메이션 loop를 만들지 않는가?
- [ ] 전이 함수 자체를 fake timer 없이 순수하게 테스트할 수 있는가?

### 구현 후 설명할 것

1. 암묵적 조건문 대신 phase 상태 기계를 선택한 이유
2. 순수 전이와 부수 효과 scheduler를 분리한 이유
3. effect dependency와 stale closure를 피하는 전략
4. reduced-motion에서 제공할 최종 상태를 선택한 기준
5. `setTimeout` 재예약과 `setInterval`의 trade-off

### 원본 확인 위치

- Thread 09
- 커밋: `feat(home): 애니메이션 터미널 상호작용 추가`
- 파일: `src/components/portfolio/animated-terminal.tsx`
- 함수·컴포넌트: `formatTerminalLine`, `AnimatedTerminal`
- 관련 Thread: 07, 13, 14

---

<a id="p14"></a>
## [Thread 14 / `build(perf): route별 client asset 측정 추가`] Build manifest 기반 route bundle 예산

### 면접 질문

페이지 소스만 보고 client 비용을 추정하지 않고 실제 production build manifest에서 route별 JS·CSS 바이트를 측정한 이유는 무엇인가? shared asset 중복, baseline 누락, compiler output 변화까지 포함해 예산 gate를 어떻게 신뢰할 수 있게 만들겠는가?

꼬리 질문:

- shared chunk와 route chunk를 단순 합산하면 중복 계산이 어떻게 생기는가?
- baseline에 없는 새 route를 자동 통과시키면 어떤 사각지대가 생기는가?
- 절대 바이트 상한과 직전 baseline 대비 성장률 중 어느 것이 더 유용한가?
- framework의 비공개 manifest 형식을 파싱하는 코드가 왜 별도 contract test를 필요로 하는가?
- `--write-baseline` 같은 갱신 명령이 gate 명령과 분리돼야 하는 이유는 무엇인가?

### 30초 모범 답변

실제 사용자가 받는 비용은 import 줄 수가 아니라 compiler가 만든 route별 JS·CSS 산출물로 결정됩니다. 그래서 build manifest에서 shared와 route asset 경로를 모으고 `Set`으로 중복 제거한 뒤 실제 파일 크기를 합산했습니다. 현재 측정과 baseline을 route별로 비교하고, 초과뿐 아니라 baseline 누락과 산출물 누락도 실패로 처리해야 새 route가 사각지대가 되지 않습니다. 다만 manifest parser는 compiler 내부 형식에 결합되므로 고정 fixture와 compiler 계약 테스트로 변화가 조용히 0바이트로 측정되지 않게 해야 합니다.

### 답변 핵심 키워드

production artifact, manifest, deduplication, byte size, route baseline, growth factor, missing baseline, fail closed, compiler contract, explicit baseline update

### 백지 구현

**구현 목표**

이미 수집된 route별 JS·CSS 측정값과 baseline을 비교해 모든 위반을 반환하는 순수 예산 평가기를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type RouteMeasurement = {
  route: string;
  jsBytes: number;
  cssBytes: number;
};

type BudgetViolation = {
  route: string;
  kind:
    | "missing-current"
    | "missing-baseline"
    | "js-growth"
    | "css-growth"
    | "duplicate-route";
  message: string;
};

export function evaluateRouteBudgets(
  current: readonly RouteMeasurement[],
  baseline: readonly RouteMeasurement[],
  growthFactor?: number,
): BudgetViolation[] {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 현재 production build 측정, 승인된 baseline, 허용 성장 배수
- 출력: 모든 route별 위반. 통과하면 빈 배열

**반드시 만족해야 할 조건**

- 현재와 baseline의 중복 route를 감지한다.
- baseline에만 있는 route와 현재에만 있는 route를 모두 명시적으로 처리한다.
- JS와 CSS를 독립적으로 비교한다.
- 정확히 임계값인 값의 통과 여부를 명확히 정의한다.
- 입력 순서와 무관하게 결정적인 결과 순서를 만든다.
- NaN, 음수, 무한대 바이트를 유효 측정으로 취급하지 않는다.
- 첫 위반에서 멈추지 않는다.

**경계 조건**

- 0바이트 baseline
- 빈 current 또는 빈 baseline
- route 순서가 서로 다른 경우
- 성장 배수가 1보다 작은 경우
- 매우 큰 정수와 부동소수점 비교
- JS만 증가하거나 CSS만 증가한 경우

**필요한 제약**

- manifest parsing과 파일 크기 수집은 범위 밖이다.
- baseline을 함수 안에서 자동 갱신하지 않는다.
- 구현 시간은 20~25분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 동일 측정값이 통과하는가?
- [ ] 임계값 직전·정확히 임계값·직후를 구분하는가?
- [ ] 새 route가 `missing-baseline`으로 드러나는가?
- [ ] 사라진 route가 `missing-current`로 드러나는가?
- [ ] JS와 CSS 위반을 동시에 수집하는가?
- [ ] 중복 route와 잘못된 숫자를 조용히 덮어쓰지 않는가?
- [ ] 입력 순서를 바꿔도 위반 순서가 결정적인가?
- [ ] 시간 복잡도가 route 수에 대해 선형에 가까운가?

### 구현 후 설명할 것

1. source-level 추정보다 production artifact 측정을 선택한 이유
2. shared asset 중복 제거가 필요한 이유
3. 성장률 예산과 절대 상한의 장단점
4. baseline 누락을 실패로 처리하는 fail-closed 정책
5. compiler 내부 형식에 의존하는 parser를 격리하고 테스트하는 방법

### 원본 확인 위치

- Thread 14
- 대표 커밋: `build(perf): route별 client asset 측정 추가`
- 연관 커밋: `build(perf): route bundle 성장 예산 평가 추가`, `build(perf): bundle budget CLI 연결`, `fix(perf): webpack route manifest parser 보강`, `test(build): compiler와 manifest parser 계약 검증`
- 파일: `scripts/route-budgets.mjs`, `scripts/route-budgets.d.mts`, `src/performance/build-manifest-contract.test.ts`, `src/performance/performance-gates.test.ts`
- 함수·타입: `parseClientReferenceManifest`, `collectRouteBundleMeasurements`, `evaluateRouteBudgets`, `ClientReferenceManifest`
- 관련 Thread: 08, 13, 15, 16

---

<a id="p15"></a>
## [Thread 14 / `test(perf): 사용자 상호작용 지연 측정 추가`] 상호작용 측정의 표본·동기화·통계 경계

### 면접 질문

브라우저에서 메뉴 열기 같은 상호작용 지연을 자동화할 때 클릭 직후 `Date.now()` 차이 하나만 재는 방식이 왜 신뢰하기 어려운가? Event Timing, paint 안정화, warm-up, 반복 표본, median과 max를 어떻게 조합해야 하는가?

꼬리 질문:

- 같은 사용자 상호작용에 여러 event entry가 생기면 `interactionId`로 왜 묶어야 하는가?
- duration threshold 아래라 entry가 없는 경우를 0ms로 간주하면 왜 왜곡되는가?
- 첫 상호작용을 warm-up으로 분리하는 이유는 무엇인가?
- median만 통과하고 max가 크게 튀는 경우 gate는 어떻게 해석해야 하는가?
- 개발 서버와 production server의 측정 결과를 같은 baseline으로 봐도 되는가?
- Lighthouse 결과 여러 회차는 URL별로 어떻게 대표값을 만들 것인가?

### 30초 모범 답변

클릭 callback 시간만 재면 브라우저의 입력 처리와 다음 paint까지 포함하지 못합니다. Event Timing entry를 `interactionId`로 묶고, 메뉴가 실제로 보인 뒤 두 번의 animation frame과 짧은 task 경계를 지나 안정화합니다. 첫 회는 warm-up으로 제외하고 여러 표본의 median과 max를 함께 봐 일반 성능과 tail을 모두 제한합니다. entry가 없으면 0이 아니라 관측 threshold보다 작다는 상한으로 기록해야 합니다. 측정은 production build와 고정된 브라우저 조건에서 반복해야 비교가 가능합니다.

### 답변 핵심 키워드

Event Timing, `interactionId`, next paint, double `requestAnimationFrame`, warm-up, repeated samples, median, max, censored observation, production environment

### 백지 구현

**구현 목표**

상호작용별 duration 표본을 검증하고 median·max를 계산하는 순수 요약기를 작성한다. 관측 entry가 없는 표본은 측정 threshold 미만이라는 상한값으로 전달된다.

**인터페이스 또는 함수 시그니처**

```ts
type InteractionSample = {
  trustedClickCount: number;
  interactionIds: number[];
  durationsMs: number[];
  noEntryUpperBoundMs?: number;
};

type InteractionSummary = {
  sampleDurationsMs: number[];
  medianMs: number;
  maxMs: number;
};

export function summarizeInteractionSamples(
  samples: readonly InteractionSample[],
): InteractionSummary {
  // 직접 구현
}
```

**입력과 출력**

- 입력: warm-up 이후 수집한 여러 상호작용 표본
- 출력: 표본별 대표 duration, median, max
- 실패: 신뢰할 수 없는 클릭 수, 복수 interaction ID, 잘못된 숫자, 빈 표본

**반드시 만족해야 할 조건**

- 각 표본은 trusted click이 정확히 하나여야 한다.
- event entry가 있으면 interaction ID가 하나로 수렴해야 한다.
- 같은 interaction의 여러 entry는 가장 보수적인 대표값을 선택한다.
- entry가 없으면 명시된 상한을 사용하고 0으로 만들지 않는다.
- 짝수 표본 수의 median 정의를 명확히 한다.
- 원본 표본 배열을 정렬하거나 수정하지 않는다.

**경계 조건**

- 표본 한 개
- duration entry 여러 개
- interaction ID가 0이거나 서로 다른 경우
- entry도 상한도 없는 경우
- NaN, 음수, 무한대
- 동일 duration만 있는 표본

**필요한 제약**

- PerformanceObserver 설치와 브라우저 자동화는 범위 밖이다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 여러 표본의 median과 max가 맞는가?
- [ ] 짝수·홀수 표본 수의 median을 의도대로 계산하는가?
- [ ] 원본 배열 순서를 변경하지 않는가?
- [ ] entry 없는 표본을 0으로 축소하지 않는가?
- [ ] 복수 interaction ID나 trusted click 불일치를 거절하는가?
- [ ] NaN·음수·무한대를 거절하는가?
- [ ] tail spike가 max에 유지되는가?
- [ ] 표본 수가 적을 때 결과 해석의 한계를 설명할 수 있는가?

### 구현 후 설명할 것

1. callback 시간보다 Event Timing을 선호한 이유
2. warm-up과 안정화 대기가 측정 신뢰도에 미치는 영향
3. median과 max를 함께 gate하는 이유
4. entry 부재를 censored observation으로 해석한 이유
5. lab 측정과 실제 사용자 모니터링의 차이

### 원본 확인 위치

- Thread 14
- 대표 커밋: `test(perf): 사용자 상호작용 지연 측정 추가`
- 연관 커밋: `build(perf): Lighthouse 결과 요약기 추가`, `chore(perf): 최종 lab 성능 측정 결과 기록`
- 파일: `tests/e2e/interaction-performance.spec.ts`, `scripts/summarize-lighthouse.mjs`, `lighthouserc.cjs`, `performance/lighthouse-baseline.json`
- 관련 Thread: 07, 09, 13, 16

---

<a id="p16"></a>
## [Thread 15 / `test(docker): runtime route와 public 자산 검증 자동화`] 외부 프로세스·임시 리소스·cleanup 우선순위

### 면접 질문

Docker image를 build하고 임시 container를 실행해 route와 자산을 검증한 뒤 항상 정리하는 스크립트를 설계하라. readiness polling, 임의 포트, 로그 수집, 본 오류와 cleanup 오류의 우선순위를 어떻게 처리할 것인가?

꼬리 질문:

- 고정 host port를 쓰면 병렬 CI에서 어떤 경쟁이 생기는가?
- `sleep 5초`와 제한된 readiness polling의 차이는 무엇인가?
- 프로세스 exit code만 보고 stdout·stderr를 버리면 진단성이 어떻게 떨어지는가?
- 검증 실패 뒤 container log를 언제 수집해야 하는가?
- 본 검증이 이미 실패했는데 image 삭제도 실패하면 어떤 오류를 호출자에게 던질 것인가?
- 정적 route 몇 개만 검사하지 않고 콘텐츠에서 자산 경로를 발견하는 이유는 무엇인가?

### 30초 모범 답변

임시 image·container 이름과 host port를 실행마다 유일하게 만들고, `spawn` 결과의 exit code와 stdout·stderr를 함께 보존해야 합니다. container 시작 뒤 고정 sleep 대신 횟수와 간격이 제한된 readiness polling을 하고, 준비되면 HTML route와 콘텐츠에서 발견한 모든 공개 자산의 상태·본문·MIME을 검사합니다. 전체 흐름은 `try/catch/finally`로 감싸 검증 실패 시 log를 남기고 항상 container와 image를 제거합니다. cleanup도 실패할 수 있지만 본 오류가 있으면 그 원인을 보존하고 cleanup 오류는 부가 정보로 다루는 편이 진단에 유리합니다.

### 답변 핵심 키워드

subprocess lifecycle, unique resource, dynamic port, readiness polling, stdout/stderr, runtime contract, dynamic asset discovery, `finally`, error precedence, idempotent cleanup

### 백지 구현

**구현 목표**

특정 Docker 명령에 종속되지 않은 임시 런타임 검증 orchestrator를 작성한다. 외부 작업은 주입된 함수로 실행하고, 성공·실패와 무관하게 정리한다.

**인터페이스 또는 함수 시그니처**

```ts
type RuntimeHandle = {
  id: string;
  baseUrl: string;
};

type RuntimeOperations = {
  build: () => Promise<void>;
  start: () => Promise<RuntimeHandle>;
  waitUntilReady: (handle: RuntimeHandle) => Promise<void>;
  verify: (handle: RuntimeHandle) => Promise<void>;
  readLogs: (handle: RuntimeHandle) => Promise<string>;
  stop: (handle: RuntimeHandle) => Promise<void>;
  removeBuildArtifact: () => Promise<void>;
};

export async function verifyEphemeralRuntime(
  operations: RuntimeOperations,
): Promise<void> {
  // 직접 구현
}
```

**입력과 출력**

- 입력: build, start, readiness, verify, log, cleanup 작업 집합
- 출력: 모두 성공하면 `void`
- 실패: 가장 의미 있는 본 오류를 보존하되 cleanup 실패 정보도 잃지 않는다.

**반드시 만족해야 할 조건**

- build 성공 뒤에만 start한다.
- start가 성공한 경우에만 runtime stop을 시도한다.
- verify 실패 시 가능한 경우 log를 읽는다.
- stop과 build artifact 제거는 독립적으로 시도한다.
- 본 오류가 있으면 cleanup 오류가 그 원인을 덮어쓰지 않는다.
- 본 오류가 없고 cleanup만 실패하면 cleanup 실패를 보고한다.
- cleanup 호출은 가능한 한 idempotent하게 취급한다.

**경계 조건**

- build 실패
- start가 일부 리소스를 만들고 실패하는 경우
- readiness timeout
- verify 실패와 log 조회 실패가 동시에 발생
- stop 실패와 artifact 제거 실패가 동시에 발생
- cleanup 함수를 두 번 호출해도 안전해야 하는 구현

**필요한 제약**

- 실제 Docker CLI 문자열 작성은 범위 밖이다.
- 구현 시간은 25~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 경로에서 build→start→ready→verify→stop→remove 순서를 지키는가?
- [ ] build 실패 시 start와 stop을 호출하지 않는가?
- [ ] readiness·verify 실패 뒤에도 가능한 cleanup을 모두 시도하는가?
- [ ] verify 실패 시 log 수집을 시도하는가?
- [ ] log 수집 실패가 원래 verify 오류를 덮지 않는가?
- [ ] 본 오류가 없고 cleanup만 실패하면 그 실패가 노출되는가?
- [ ] stop과 remove 중 하나가 실패해도 다른 하나를 시도하는가?
- [ ] 호출 순서를 fake operation으로 결정적으로 테스트할 수 있는가?

### 구현 후 설명할 것

1. 외부 명령을 주입 가능한 operations로 분리한 이유
2. readiness polling을 고정 sleep보다 선호한 이유
3. 본 오류와 cleanup 오류의 우선순위 정책
4. 임시 이름·동적 포트가 병렬 실행 안전성에 주는 효과
5. route·자산·사용자 권한까지 runtime에서 확인해야 하는 이유

### 원본 확인 위치

- Thread 15
- 대표 커밋: `test(docker): runtime route와 public 자산 검증 자동화`
- 연관 커밋: `build: standalone server 산출물 생성`, `test(build): standalone 산출물 완전성 검증`, `build(docker): public 자산을 포함한 비루트 standalone image 추가`, `fix(build): production build에 webpack compiler 고정`
- 파일: `scripts/verify-container-runtime.mjs`, `scripts/verify-build-output.mjs`, `Dockerfile`, `.dockerignore`, `next.config.ts`
- 관련 Thread: 03, 04, 13, 14, 16

---

<a id="p17"></a>
## [Thread 16 / `ci: harden portfolio validation`] 재현 가능한 CI gate DAG와 최종 집계

### 면접 질문

lint·typecheck·content·unit·production E2E·bundle·Lighthouse·container 검증을 한 workflow에 넣을 때, 단순한 직렬 shell script 대신 job DAG와 최종 verify job을 두는 이유는 무엇인가? build 재사용과 실패 집계까지 설명하라.

꼬리 질문:

- 서로 독립적인 quality와 container 검증을 직렬화하면 어떤 비용이 생기는가?
- production build를 E2E, 산출물 검증, bundle 측정마다 다시 만들면 어떤 문제가 생기는가?
- 선행 job이 실패해 downstream verify가 skip되면 branch protection에서 어떤 혼동이 생길 수 있는가?
- `if: !cancelled()` 상태에서 모든 `needs.*.result`를 검사하는 final gate의 역할은 무엇인가?
- action을 commit SHA로 고정하고 workflow 권한을 read-only로 제한하는 이유는 무엇인가?
- Makefile target과 package script의 책임을 어떻게 나눌 것인가?

### 30초 모범 답변

검증은 의존 관계가 다른 DAG입니다. lint·typecheck·unit 같은 quality와 container build는 병렬화하고, production build가 필요한 E2E·bundle·Lighthouse는 같은 산출물을 재사용하도록 경계를 잡아야 합니다. 마지막 verify job은 취소가 아닌 경우 항상 실행해 모든 선행 결과가 success인지 명시적으로 판정하므로, 중간 실패가 단순 skip으로 숨지 않습니다. 로컬과 CI는 같은 Make target을 호출하고, action SHA 고정·최소 권한·timeout·concurrency 취소로 공급망과 자원 사용도 통제합니다.

### 답변 핵심 키워드

DAG, dependency, parallelism, artifact reuse, final aggregator, skipped vs failed, reproducible command, pinned action, least privilege, timeout, concurrency

### 백지 구현

**구현 목표**

검증 gate 정의를 받아 누락 의존성과 cycle을 찾고, 병렬 실행 가능한 level 순서로 계획하는 DAG 검사기를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type Gate = {
  id: string;
  needs: string[];
};

type GatePlan = {
  levels: string[][];
  terminalGates: string[];
};

export function validateAndPlanGates(
  gates: readonly Gate[],
): GatePlan {
  // 직접 구현
}
```

**입력과 출력**

- 입력: gate ID와 선행 gate ID 목록
- 출력: 같은 level은 병렬 실행 가능한 결정적 계획, downstream이 없는 terminal gate 목록
- 실패: 중복 ID, 존재하지 않는 dependency, 자기 의존, cycle

**반드시 만족해야 할 조건**

- 모든 gate를 정확히 한 번 계획한다.
- 어떤 gate도 dependency보다 앞 level에 오지 않는다.
- 독립 gate는 같은 level에 배치할 수 있다.
- 같은 입력 집합은 입력 배열 순서와 무관하게 결정적 계획을 만든다.
- cycle과 누락 dependency를 구분해 진단한다.
- terminal gate를 식별한다.
- final verify gate를 별도로 요구한다면 모든 terminal 결과를 의존하도록 검증하는 확장 규칙을 설명한다.

**경계 조건**

- 빈 DAG
- gate 하나
- diamond dependency
- 여러 개의 독립 terminal gate
- 긴 chain
- 자기 cycle과 다중 node cycle
- 동일 dependency가 중복 기재된 경우

**필요한 제약**

- 실제 CI YAML 생성은 범위 밖이다.
- 시간 복잡도 목표는 O(V+E)다.
- 구현 시간은 25~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 독립 gate가 같은 level에 배치되는가?
- [ ] diamond DAG에서 공통 downstream이 두 선행 뒤에 배치되는가?
- [ ] 중복 ID와 누락 dependency를 거절하는가?
- [ ] 자기 cycle과 일반 cycle을 탐지하는가?
- [ ] 모든 gate가 한 번만 결과에 포함되는가?
- [ ] 입력 순서를 바꿔도 각 level의 정렬이 결정적인가?
- [ ] terminal gate 계산이 맞는가?
- [ ] 시간·공간 복잡도를 O(V+E)로 설명할 수 있는가?

### 구현 후 설명할 것

1. 어떤 검증을 병렬화하고 어떤 검증을 직렬화할지 정하는 기준
2. production build artifact를 재사용할 때 생기는 이점과 격리 문제
3. final verify job이 skip·failure 의미를 정규화하는 방식
4. 로컬 Make target과 CI workflow의 단일 명령 경계
5. action pinning·최소 권한·timeout·concurrency가 보안과 비용에 주는 효과

### 원본 확인 위치

- Thread 16
- 대표 커밋: `ci: harden portfolio validation`
- 연관 커밋: `ci: 기본 배포 품질 검사 추가`, `ci: standalone 산출물 검증 추가`, `ci: 검증된 bundle과 Lighthouse gate 활성화`, `build: improve Makefile and separate functional portfolio checks`
- 파일: `.github/workflows/web-portfolio-ci.yml`, `Makefile`, `package.json`
- job·target: `quality`, `production`, `container`, `verify`, `check-functional`, `build-verify`, `bundle-check`, `lighthouse`, `container`, `ci`
- 관련 Thread: 03, 04, 13, 14, 15
