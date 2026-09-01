# 런타임·성능·릴리스 면접 워크북

이 문서는 타이머 lifecycle, 사용자 상호작용 측정, 빌드 산출물 예산, 컨테이너 검증, CI 의존 그래프를 묶는다. 공통 주제는 **성공 경로만 구현하지 않고 시간·프로세스·산출물·정리 단계까지 하나의 계약으로 다루는 것**이다.

<a id="p11"></a>
## [Thread 09 / `feat(home): 애니메이션 터미널 상호작용 추가`] 타이머 기반 유한 상태 기계와 effect 정리

### 면접 질문

명령어를 입력하고 잠시 유지한 뒤 지우는 애니메이션을 React effect로 구현할 때, 단순히 문자열 길이만 증감시키는 방식보다 명시적 phase 상태가 필요한 이유는 무엇인가? 타이머 lifecycle과 reduced-motion 정책을 포함해 설명하라.

꼬리 질문:

- `typing | hold | erase` phase가 없으면 전환 조건이 어디에 흩어지는가?
  - 모범답변: 문자열 길이, 마지막 글자 도달 여부, 대기 시간 같은 조건이 여러 timer callback과 render 분기에 흩어집니다. phase를 discriminant로 두면 각 상태의 지연과 다음 상태를 한 전이 함수에서 설명할 수 있습니다.
- effect dependency를 잘못 잡으면 stale closure나 타이머 중복이 어떻게 발생하는가?
  - 모범답변: 현재 command·문자열·phase를 dependency에서 빼면 callback이 이전 값을 캡처하고, cleanup 없이 effect가 다시 실행되면 이전 timer와 새 timer가 함께 상태를 갱신합니다. 원본은 필요한 현재값을 dependency에 두고 매 effect cleanup에서 timeout 하나를 해제합니다.
- command 배열이 바뀌거나 빈 배열이 들어오면 인덱스 invariant를 어떻게 유지할 것인가?
  - 모범답변: 변경 시 현재 index를 새 길이로 정규화하고, 다음 index는 modulo로 순환시킵니다. 빈 배열은 modulo를 수행하지 않고 빈 안정 상태로 전환해야 하며, 원본 컴포넌트는 콘텐츠 스키마가 command 존재를 보장한다는 전제도 갖습니다.
- `setTimeout` cleanup을 빠뜨리면 unmount 뒤 어떤 현상이 생기는가?
  - 모범답변: 제거된 컴포넌트에 상태 갱신이 시도되고 재마운트와 겹쳐 애니메이션 속도가 빨라지거나 순서가 틀어질 수 있습니다. timer와 closure가 예정 시간까지 남는 자원 누수도 생깁니다.
- `prefers-reduced-motion`에서는 애니메이션을 느리게 할 것인가, 제거할 것인가?
  - 모범답변: 이 터미널 애니메이션은 장식이므로 원본처럼 반복 timer를 만들지 않고 첫 command가 완전히 보이는 안정 상태를 제공합니다. 느리게 만드는 것은 움직임 노출 시간을 오히려 늘릴 수 있습니다.

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
  if (commands.length === 0) {
    return {
      next: { commandIndex: 0, visibleText: "", phase: "hold" },
      delayMs: 0,
    };
  }

  const commandIndex =
    ((state.commandIndex % commands.length) + commands.length) %
    commands.length;
  const command = commands[commandIndex] ?? "";

  if (state.phase === "typing") {
    if (state.visibleText.length < command.length) {
      return {
        next: {
          commandIndex,
          visibleText: command.slice(0, state.visibleText.length + 1),
          phase: "typing",
        },
        delayMs: 42,
      };
    }
    return {
      next: { commandIndex, visibleText: command, phase: "hold" },
      delayMs: 520,
    };
  }

  if (state.phase === "hold") {
    return {
      next: { commandIndex, visibleText: command, phase: "erase" },
      delayMs: 1_700,
    };
  }

  if (state.visibleText.length > 0) {
    return {
      next: {
        commandIndex,
        visibleText: command.slice(0, state.visibleText.length - 1),
        phase: "erase",
      },
      delayMs: 24,
    };
  }

  return {
    next: {
      commandIndex: (commandIndex + 1) % commands.length,
      visibleText: "",
      phase: "typing",
    },
    delayMs: 220,
  };
}

export function startTerminalLoop(options: {
  commands: readonly string[];
  initialState: TerminalState;
  onState: (state: TerminalState) => void;
  reducedMotion: boolean;
}): () => void {
  if (options.commands.length === 0) {
    options.onState({ commandIndex: 0, visibleText: "", phase: "hold" });
    return () => {};
  }

  if (options.reducedMotion) {
    // 장식 loop를 만들지 않고 즉시 읽을 수 있는 첫 command를 고정한다.
    options.onState({
      commandIndex: 0,
      visibleText: options.commands[0] ?? "",
      phase: "hold",
    });
    return () => {};
  }

  // 이 controller가 실행되는 동안 command 목록은 snapshot으로 고정한다.
  // 새 목록을 적용하려면 cleanup 뒤 controller를 다시 시작한다.
  const commands = [...options.commands];
  let state: TerminalState = {
    ...options.initialState,
    commandIndex:
      ((options.initialState.commandIndex % commands.length) +
        commands.length) %
      commands.length,
  };
  let active = true;
  let timer: ReturnType<typeof setTimeout> | undefined;

  const schedule = () => {
    const transition = advanceTerminal(state, commands);
    timer = setTimeout(() => {
      if (!active) return;
      state = transition.next;
      options.onState(state);
      schedule();
    }, transition.delayMs);
  };

  options.onState(state);
  schedule();

  return () => {
    if (!active) return;
    active = false;
    if (timer !== undefined) clearTimeout(timer);
  };
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
   - 모범답변: typing·hold·erase는 전환 조건과 지연이 서로 다른 유한 상태입니다. phase를 명시하면 가능한 상태와 전이를 한 곳에서 검토할 수 있고 문자열 길이에 숨은 의미를 줄입니다.
2. 순수 전이와 부수 효과 scheduler를 분리한 이유
   - 모범답변: 전이 함수는 fake timer 없이 입력과 출력만 테스트할 수 있고, scheduler는 한 timer와 cleanup이라는 lifecycle에 집중합니다. 상태 계산 오류와 자원 관리 오류를 독립적으로 진단할 수 있습니다.
3. effect dependency와 stale closure를 피하는 전략
   - 모범답변: 전이가 읽는 active command, command 길이, phase, visible text를 dependency에 포함하고 functional update가 필요한 index에는 이전값 callback을 씁니다. effect가 재실행될 때 기존 timeout을 항상 취소합니다.
4. reduced-motion에서 제공할 최종 상태를 선택한 기준
   - 모범답변: 정보가 사라지지 않으면서 반복 움직임이 없어야 하므로 첫 command 전체와 출력이 읽히는 hold 상태를 택했습니다. 원본도 matchMedia가 reduce이면 effect가 timer를 예약하지 않습니다.
5. `setTimeout` 재예약과 `setInterval`의 trade-off
   - 모범답변: 재예약은 phase마다 42ms·520ms·1700ms처럼 다른 지연을 직접 표현하고 이전 tick이 끝난 뒤 다음 하나만 만듭니다. interval은 단순 주기에는 편하지만 상태별 지연과 밀린 callback 제어가 어렵습니다.

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
  - 모범답변: 같은 asset이 shared 목록과 route manifest 또는 여러 entry에 반복될 수 있어 경로별 크기를 여러 번 더하게 됩니다. 원본 `assetBytes`는 파일 경로를 `Set`으로 중복 제거한 뒤 실제 크기를 한 번만 합산합니다.
- baseline에 없는 새 route를 자동 통과시키면 어떤 사각지대가 생기는가?
  - 모범답변: 새 route의 비용에는 승인된 기준이 없는데도 무제한으로 들어갈 수 있습니다. 원본 gate는 현재에만 있는 route를 "committed baseline 없음" 위반으로 만들어 명시적 baseline 검토를 요구합니다.
- 절대 바이트 상한과 직전 baseline 대비 성장률 중 어느 것이 더 유용한가?
  - 모범답변: 성장률은 기존 route의 작은 회귀를 민감하게 잡지만 큰 baseline을 계속 허용하고 0에 취약합니다. 절대 상한은 사용자 비용 목표가 명확하지만 route 특성을 반영하기 어려워, 필요하면 성장률과 절대 상한을 함께 둡니다. 이 프로젝트는 승인 baseline 대비 5%를 사용합니다.
- framework의 비공개 manifest 형식을 파싱하는 코드가 왜 별도 contract test를 필요로 하는가?
  - 모범답변: compiler 업그레이드로 assignment 모양이나 `entryJSFiles`·`entryCSSFiles` 구조가 바뀌어도 조용히 빈 목록으로 읽으면 gate가 0바이트로 통과할 수 있습니다. 고정 fixture와 실제 compiler 출력 계약으로 파싱 실패를 즉시 드러내야 합니다.
- `--write-baseline` 같은 갱신 명령이 gate 명령과 분리돼야 하는 이유는 무엇인가?
  - 모범답변: 검사 도중 baseline을 자동 갱신하면 회귀가 새 기준으로 승인되어 항상 통과합니다. 원본은 기본 명령은 read-only 비교만 하고 명시적 `--write-baseline`에서만 파일을 바꿉니다.

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
  const factor = growthFactor ?? 1.05;
  if (!Number.isFinite(factor) || factor < 1) {
    throw new RangeError("growthFactor must be a finite number greater than or equal to 1.");
  }

  const violations: BudgetViolation[] = [];
  const duplicateRoutes = (values: readonly RouteMeasurement[]) => {
    const seen = new Set<string>();
    const duplicates = new Set<string>();
    for (const value of values) {
      if (seen.has(value.route)) duplicates.add(value.route);
      seen.add(value.route);
    }
    return duplicates;
  };

  const currentDuplicates = duplicateRoutes(current);
  const baselineDuplicates = duplicateRoutes(baseline);
  for (const route of [...new Set([...currentDuplicates, ...baselineDuplicates])].sort()) {
    const locations = [
      currentDuplicates.has(route) ? "current" : "",
      baselineDuplicates.has(route) ? "baseline" : "",
    ].filter(Boolean).join(" and ");
    violations.push({
      route,
      kind: "duplicate-route",
      message: `${route}: duplicate route in ${locations}`,
    });
  }

  // 중복은 위에서 진단하고 비교에는 결정적으로 첫 측정을 사용한다.
  const currentByRoute = new Map<string, RouteMeasurement>();
  const baselineByRoute = new Map<string, RouteMeasurement>();
  current.forEach((item) => {
    if (!currentByRoute.has(item.route)) currentByRoute.set(item.route, item);
  });
  baseline.forEach((item) => {
    if (!baselineByRoute.has(item.route)) baselineByRoute.set(item.route, item);
  });

  const routes = [...new Set([...currentByRoute.keys(), ...baselineByRoute.keys()])].sort();
  const validBytes = (value: number) => Number.isFinite(value) && value >= 0;

  for (const route of routes) {
    const actual = currentByRoute.get(route);
    const expected = baselineByRoute.get(route);
    if (!actual) {
      violations.push({
        route,
        kind: "missing-current",
        message: `${route}: current route output is missing`,
      });
      continue;
    }
    if (!expected) {
      violations.push({
        route,
        kind: "missing-baseline",
        message: `${route}: committed baseline is missing`,
      });
      continue;
    }

    for (const [property, kind] of [
      ["jsBytes", "js-growth"],
      ["cssBytes", "css-growth"],
    ] as const) {
      if (!validBytes(actual[property]) || !validBytes(expected[property])) {
        violations.push({
          route,
          kind,
          message: `${route}: ${property} must be finite and non-negative`,
        });
        continue;
      }

      const allowed = Math.floor(expected[property] * factor);
      // 원본 gate처럼 정확히 limit인 값은 통과한다.
      if (actual[property] > allowed) {
        violations.push({
          route,
          kind,
          message: `${route}: ${property} is ${actual[property]} bytes (limit ${allowed})`,
        });
      }
    }
  }

  return violations;
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
   - 모범답변: 실제 전송 비용은 compiler의 code splitting, shared chunk, CSS 추출 결과로 정해져 소스 import 수와 일치하지 않습니다. 원본은 production `.next` manifest가 가리키는 실제 파일 크기를 route별로 측정합니다.
2. shared asset 중복 제거가 필요한 이유
   - 모범답변: 동일 파일이 root main과 route entry 양쪽에 나타날 수 있으므로 경로를 그대로 합산하면 사용자 다운로드 비용을 과대 계산합니다. route 하나의 asset 경로 집합을 만든 뒤 각 파일을 한 번만 더합니다.
3. 성장률 예산과 절대 상한의 장단점
   - 모범답변: baseline 성장률은 작은 변화 검토에 좋지만 이미 큰 route와 0 baseline 문제를 해결하지 못합니다. 절대 상한은 사용자 목표를 직접 표현하지만 route별 정당한 차이를 수용하기 어려워 두 정책을 보완적으로 사용할 수 있습니다.
4. baseline 누락을 실패로 처리하는 fail-closed 정책
   - 모범답변: 새 route나 사라진 build output을 자동 통과시키면 측정 범위가 조용히 축소됩니다. 현재·baseline 어느 한쪽만 있는 route를 모두 위반으로 만들어 사람의 승인이나 원인 조사를 요구합니다.
5. compiler 내부 형식에 의존하는 parser를 격리하고 테스트하는 방법
   - 모범답변: manifest assignment와 route asset 추출을 작은 함수로 분리하고 대표 compiler fixture의 성공·깨진 형식의 실패를 테스트합니다. 실제 build 산출물에서 route가 0개가 아닌지도 계약으로 확인해 업그레이드 회귀를 잡습니다.

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
  - 모범답변: pointer·click 등 여러 event entry가 하나의 사용자 동작에서 나올 수 있어 entry 수를 곧 표본 수로 보면 중복 계산됩니다. 양의 동일 `interactionId`로 묶고 그 그룹의 최대 duration을 한 상호작용의 보수적 값으로 사용합니다.
- duration threshold 아래라 entry가 없는 경우를 0ms로 간주하면 왜 왜곡되는가?
  - 모범답변: 실제 duration이 0이라는 뜻이 아니라 observer가 threshold 아래를 보고하지 않았다는 뜻입니다. 원본은 `<16ms`라는 상한 16ms로 기록해 통계가 인위적으로 좋아지지 않게 합니다.
- 첫 상호작용을 warm-up으로 분리하는 이유는 무엇인가?
  - 모범답변: 첫 실행에는 lazy code, style 계산, cache 준비 같은 일회성 비용이 섞일 수 있습니다. 원본은 한 번 열고 닫아 경로를 준비한 뒤 probe를 reset하고 세 표본을 수집합니다.
- median만 통과하고 max가 크게 튀는 경우 gate는 어떻게 해석해야 하는가?
  - 모범답변: 전형적 경험은 괜찮지만 tail latency 회귀가 있다는 뜻입니다. 원본은 median과 max 모두 200ms 이하를 요구해 일반 성능과 최악 표본을 동시에 제한합니다.
- 개발 서버와 production server의 측정 결과를 같은 baseline으로 봐도 되는가?
  - 모범답변: 개발 compiler, source map, cold compilation과 최적화 수준이 달라 직접 비교하면 안 됩니다. 고정한 production build, 브라우저 프로젝트, viewport 조건에서 수집한 값끼리 비교해야 합니다.
- Lighthouse 결과 여러 회차는 URL별로 어떻게 대표값을 만들 것인가?
  - 모범답변: URL별 run을 먼저 묶고 각 metric의 대표값 정책을 고정해야 합니다. 중앙 경향은 median이 안정적이고 gate는 worst run도 함께 보존할 수 있으며, 서로 다른 URL 표본을 한 분포로 섞지 않습니다.

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
  if (samples.length === 0) {
    throw new Error("At least one interaction sample is required.");
  }

  const isDuration = (value: number) => Number.isFinite(value) && value >= 0;
  const sampleDurationsMs = samples.map((sample, index) => {
    if (sample.trustedClickCount !== 1) {
      throw new Error(`Sample ${index} must contain exactly one trusted click.`);
    }

    if (sample.durationsMs.length === 0) {
      if (
        sample.interactionIds.length !== 0 ||
        sample.noEntryUpperBoundMs === undefined ||
        !isDuration(sample.noEntryUpperBoundMs)
      ) {
        throw new Error(`Sample ${index} needs a valid no-entry upper bound.`);
      }
      return sample.noEntryUpperBoundMs;
    }

    if (!sample.durationsMs.every(isDuration)) {
      throw new Error(`Sample ${index} contains an invalid duration.`);
    }
    const interactionIds = new Set(sample.interactionIds);
    if (
      interactionIds.size !== 1 ||
      [...interactionIds].some((interactionId) =>
        !Number.isInteger(interactionId) || interactionId <= 0
      )
    ) {
      throw new Error(`Sample ${index} must resolve to one positive interaction ID.`);
    }

    // 같은 interaction의 여러 event entry 중 tail을 보존한다.
    return Math.max(...sample.durationsMs);
  });

  const sorted = [...sampleDurationsMs].sort((left, right) => left - right);
  // 원본 측정기와 동일하게 짝수 표본도 upper median을 사용한다.
  const medianMs = sorted[Math.floor(sorted.length / 2)];

  return {
    sampleDurationsMs,
    medianMs,
    maxMs: Math.max(...sampleDurationsMs),
  };
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
   - 모범답변: callback 전후 시각은 browser input 처리와 다음 paint까지의 전체 지연을 담지 못합니다. Event Timing duration은 사용자 interaction과 presentation 경계를 더 가깝게 측정하고 `interactionId`로 관련 event를 묶을 수 있습니다.
2. warm-up과 안정화 대기가 측정 신뢰도에 미치는 영향
   - 모범답변: warm-up은 일회성 lazy 초기화 비용을 반복 표본과 분리합니다. DOM visible 확인 뒤 double `requestAnimationFrame`과 task 경계를 기다리면 관련 paint와 observer entry가 반영되기 전에 읽는 경쟁을 줄입니다.
3. median과 max를 함께 gate하는 이유
   - 모범답변: median은 소수 잡음에 덜 민감한 전형적 표본이고 max는 사용자가 실제로 겪을 tail spike를 보존합니다. 둘 중 하나만 보면 각각 불안정성이나 일반적 회귀를 놓칠 수 있습니다.
4. entry 부재를 censored observation으로 해석한 이유
   - 모범답변: observer threshold 아래라는 정보만 있고 정확한 duration은 모릅니다. 0으로 꾸미지 않고 threshold 상한을 사용하면 보수적이며 표본 의미를 잃지 않습니다.
5. lab 측정과 실제 사용자 모니터링의 차이
   - 모범답변: 이 테스트는 고정 브라우저·viewport·production server의 재현 가능한 회귀 gate입니다. 실제 사용자는 기기·네트워크·입력이 다양하므로 현장 분포와 percentile을 보려면 별도 RUM이 필요합니다.

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
  - 모범답변: 같은 runner에서 여러 실행이 같은 port를 bind해 한쪽 container가 시작하지 못하거나 다른 실행의 서버를 검증할 수 있습니다. 원본은 `127.0.0.1::3100`으로 임의 host port를 받고 실제 매핑을 조회합니다.
- `sleep 5초`와 제한된 readiness polling의 차이는 무엇인가?
  - 모범답변: 고정 sleep은 빠른 시작에도 항상 기다리고 느린 시작에서는 너무 일찍 진행합니다. 원본은 최대 60회, 1초 간격으로 응답 성공을 확인해 빠르면 즉시 진행하고 timeout과 마지막 오류를 진단합니다.
- 프로세스 exit code만 보고 stdout·stderr를 버리면 진단성이 어떻게 떨어지는가?
  - 모범답변: 어떤 command와 Docker 메시지가 실패 원인인지 알 수 없어 재현이 어려워집니다. 원본 wrapper는 command, exit code, capture한 stderr를 오류에 포함하고 필요한 stdout은 port·사용자 조회에 사용합니다.
- 검증 실패 뒤 container log를 언제 수집해야 하는가?
  - 모범답변: container가 실제로 시작된 뒤 readiness나 route·asset 검증이 실패했을 때, container를 제거하기 전에 수집해야 합니다. log 조회 실패는 본 검증 오류를 덮지 않아야 합니다.
- 본 검증이 이미 실패했는데 image 삭제도 실패하면 어떤 오류를 호출자에게 던질 것인가?
  - 모범답변: 본 검증 오류를 주 원인으로 유지하고 cleanup 오류는 부가 원인으로 함께 기록합니다. 원본 스크립트도 이미 실패한 경우 cleanup 예외로 원래 오류를 덮지 않습니다.
- 정적 route 몇 개만 검사하지 않고 콘텐츠에서 자산 경로를 발견하는 이유는 무엇인가?
  - 모범답변: 콘텐츠가 새 이미지·PDF를 참조해도 고정 목록은 갱신 누락될 수 있습니다. 원본은 모든 JSON을 재귀 순회해 `/content`·`/template` 자산을 찾고 상태, 빈 body, MIME을 실제 container에서 검사합니다.

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
  let handle: RuntimeHandle | undefined;
  let primaryError: unknown;
  let logs: string | undefined;
  const auxiliaryErrors: Error[] = [];
  const cleanupErrors: Error[] = [];
  const asError = (error: unknown) =>
    error instanceof Error ? error : new Error(String(error));

  try {
    await operations.build();
    handle = await operations.start();
    await operations.waitUntilReady(handle);
    await operations.verify(handle);
  } catch (error) {
    primaryError = error;
    if (handle) {
      try {
        // 제거 전에 읽되, log 실패가 본 오류를 덮지 않게 수집만 한다.
        logs = await operations.readLogs(handle);
      } catch (logError) {
        auxiliaryErrors.push(asError(logError));
      }
    }
  } finally {
    if (handle) {
      try {
        await operations.stop(handle);
      } catch (error) {
        cleanupErrors.push(asError(error));
      }
    }

    // build가 부분 실패했을 수 있으므로 artifact 제거는 독립적으로 시도한다.
    try {
      await operations.removeBuildArtifact();
    } catch (error) {
      cleanupErrors.push(asError(error));
    }
  }

  if (primaryError !== undefined) {
    const primary = asError(primaryError);
    const details = logs ? `${primary.message}\nRuntime logs:\n${logs}` : primary.message;
    const secondary = [...auxiliaryErrors, ...cleanupErrors];
    if (secondary.length > 0) {
      throw new AggregateError([primary, ...secondary], details);
    }
    if (logs) throw new Error(details, { cause: primary });
    throw primary;
  }

  if (cleanupErrors.length > 0) {
    throw new AggregateError(cleanupErrors, "Ephemeral runtime cleanup failed.");
  }
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
   - 모범답변: orchestration의 순서·오류 우선순위·cleanup을 fake promise로 결정적으로 테스트할 수 있고 Docker command 구성은 adapter에 격리됩니다. 실제 프로세스 없이 모든 실패 조합을 재현할 수 있습니다.
2. readiness polling을 고정 sleep보다 선호한 이유
   - 모범답변: 준비 완료라는 실제 조건을 확인하므로 빠른 실행은 바로 진행하고 느린 실행은 정한 한도까지 기다립니다. 횟수와 간격을 제한해 무한 대기 없이 마지막 오류를 timeout 진단에 남깁니다.
3. 본 오류와 cleanup 오류의 우선순위 정책
   - 모범답변: build·ready·verify 오류를 주 원인으로 유지하고 logs·stop·remove 실패는 보조 정보로 집계합니다. 본 흐름이 성공했을 때 cleanup만 실패하면 그 실패를 호출자에게 노출합니다.
4. 임시 이름·동적 포트가 병렬 실행 안전성에 주는 효과
   - 모범답변: 원본은 PID와 random suffix로 image·container 이름을 유일하게 만들고 Docker가 host port를 선택하게 합니다. 병렬 job끼리 이름 충돌, port bind 실패, 서로의 리소스 정리를 피합니다.
5. route·자산·사용자 권한까지 runtime에서 확인해야 하는 이유
   - 모범답변: build 성공만으로 standalone 파일 복사, public 자산, MIME, 비루트 사용자 설정이 image 안에서 실제로 동작한다는 보장은 없습니다. 원본은 대표 HTML route, 발견한 모든 자산, `node` 사용자까지 실행 container에서 확인합니다.

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
  - 모범답변: 둘이 같은 선행 조건만 가진다면 한쪽 시간만큼 전체 피드백이 늦어지고 runner 병렬성을 쓰지 못합니다. 현재 workflow는 quality 성공 뒤 production과 container를 서로 독립 job으로 실행합니다.
- production build를 E2E, 산출물 검증, bundle 측정마다 다시 만들면 어떤 문제가 생기는가?
  - 모범답변: CPU·시간을 반복 소비하고 서로 다른 build가 섞여 같은 산출물을 검증한다는 보장이 약해집니다. 원본 production job과 Make `verify`는 E2E가 만든 `.next`를 build verify와 bundle check가 이어서 사용합니다.
- 선행 job이 실패해 downstream verify가 skip되면 branch protection에서 어떤 혼동이 생길 수 있는가?
  - 모범답변: 최종 필수 check가 실행되지 않아 실패인지 단순 조건부 skip인지 상태가 분산됩니다. `!cancelled()` final job이 모든 `needs` 결과를 직접 검사하면 하나의 명확한 실패로 정규화됩니다.
- `if: !cancelled()` 상태에서 모든 `needs.*.result`를 검사하는 final gate의 역할은 무엇인가?
  - 모범답변: 선행 실패 때문에 일반 downstream 조건이 false여도 취소가 아니면 실행하고 quality·production·container가 모두 `success`인지 요구합니다. 일부 실패나 skip을 최종 성공으로 오인하지 않게 합니다.
- action을 commit SHA로 고정하고 workflow 권한을 read-only로 제한하는 이유는 무엇인가?
  - 모범답변: mutable tag가 바뀌어 검증 코드가 예고 없이 달라지는 공급망 위험을 줄이고, action이 침해돼도 repository 쓰기 권한을 최소화합니다. 원본은 checkout·setup·upload action을 SHA로 고정하고 `contents: read`만 부여합니다.
- Makefile target과 package script의 책임을 어떻게 나눌 것인가?
  - 모범답변: package script는 각 도구의 구체 명령을 소유하고 Make target은 content→lint→typecheck처럼 로컬·CI가 공유할 workflow 경계를 조합합니다. 그래서 CI YAML은 세부 옵션을 복제하지 않고 안정된 target을 호출합니다.

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
  const orderedGates = [...gates].sort((left, right) =>
    left.id.localeCompare(right.id),
  );
  const byId = new Map<string, Gate>();

  for (const gate of orderedGates) {
    if (byId.has(gate.id)) {
      throw new Error(`Duplicate gate id "${gate.id}".`);
    }
    byId.set(gate.id, gate);
  }

  const indegree = new Map<string, number>();
  const dependents = new Map<string, string[]>();
  for (const gate of orderedGates) {
    const uniqueNeeds = new Set(gate.needs);
    if (uniqueNeeds.size !== gate.needs.length) {
      throw new Error(`Gate "${gate.id}" repeats a dependency.`);
    }
    for (const dependency of [...uniqueNeeds].sort()) {
      if (dependency === gate.id) {
        throw new Error(`Gate "${gate.id}" depends on itself.`);
      }
      if (!byId.has(dependency)) {
        throw new Error(
          `Gate "${gate.id}" needs missing gate "${dependency}".`,
        );
      }
      dependents.set(dependency, [
        ...(dependents.get(dependency) ?? []),
        gate.id,
      ]);
    }
    indegree.set(gate.id, uniqueNeeds.size);
  }

  let ready = orderedGates
    .filter((gate) => indegree.get(gate.id) === 0)
    .map((gate) => gate.id);
  const levels: string[][] = [];
  let plannedCount = 0;

  while (ready.length > 0) {
    const level = [...ready].sort();
    levels.push(level);
    plannedCount += level.length;
    const next: string[] = [];

    for (const gateId of level) {
      for (const dependent of dependents.get(gateId) ?? []) {
        const remaining = (indegree.get(dependent) ?? 0) - 1;
        indegree.set(dependent, remaining);
        if (remaining === 0) next.push(dependent);
      }
    }
    ready = next;
  }

  if (plannedCount !== orderedGates.length) {
    const cyclic = orderedGates
      .filter((gate) => (indegree.get(gate.id) ?? 0) > 0)
      .map((gate) => gate.id);
    throw new Error(`Gate dependency cycle: ${cyclic.join(", ")}.`);
  }

  return {
    levels,
    terminalGates: orderedGates
      .filter((gate) => (dependents.get(gate.id) ?? []).length === 0)
      .map((gate) => gate.id),
  };
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
   - 모범답변: 데이터·산출물 의존성이 없고 자원 충돌이 없는 gate는 병렬화합니다. production build를 소비하는 E2E·산출물·bundle·Lighthouse처럼 같은 artifact가 필요한 단계는 생산자 뒤에 두고 재사용 순서로 실행합니다.
2. production build artifact를 재사용할 때 생기는 이점과 격리 문제
   - 모범답변: build 시간과 변동을 줄이고 같은 결과를 여러 gate가 검증합니다. job을 나누면 artifact 업로드·다운로드, 무결성, 환경 차이를 관리해야 하므로 원본은 관련 gate를 한 production job 안에 묶었습니다.
3. final verify job이 skip·failure 의미를 정규화하는 방식
   - 모범답변: `if: !cancelled()`로 선행 실패 후에도 실행하고 모든 `needs.*.result`가 success인지 검사합니다. branch protection은 분산된 skip 상태 대신 이 최종 check 하나에서 전체 성공 여부를 볼 수 있습니다.
4. 로컬 Make target과 CI workflow의 단일 명령 경계
   - 모범답변: `check-functional`, `verify`, `ci`가 검증 조합을 소유하고 CI도 같은 target이나 package script를 호출합니다. 개발자와 runner가 서로 다른 명령 집합을 실행해 생기는 drift를 줄입니다.
5. action pinning·최소 권한·timeout·concurrency가 보안과 비용에 주는 효과
   - 모범답변: SHA pinning과 read-only 권한은 공급망 변경과 피해 범위를 줄입니다. job timeout은 hung process 비용을 제한하고, 같은 ref의 이전 실행을 취소하는 concurrency 정책은 낡은 검증에 runner를 쓰지 않게 합니다.

### 원본 확인 위치

- Thread 16
- 대표 커밋: `ci: harden portfolio validation`
- 연관 커밋: `ci: 기본 배포 품질 검사 추가`, `ci: standalone 산출물 검증 추가`, `ci: 검증된 bundle과 Lighthouse gate 활성화`, `build: improve Makefile and separate functional portfolio checks`
- 파일: `.github/workflows/web-portfolio-ci.yml`, `Makefile`, `package.json`
- job·target: `quality`, `production`, `container`, `verify`, `check-functional`, `build-verify`, `bundle-check`, `lighthouse`, `container`, `ci`
- 관련 Thread: 03, 04, 13, 14, 15
