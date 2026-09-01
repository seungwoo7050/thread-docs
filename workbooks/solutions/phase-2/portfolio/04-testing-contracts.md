# 다차원 테스트 계약 면접 워크북

이 문서는 다섯 디자인과 여러 route가 같은 콘텐츠·탐색·접근성 계약을 지키는지 검증한 작업을 묶는다. 핵심은 테스트 수를 늘리는 것이 아니라 **지원하는 상태 공간을 한 곳에서 정의하고, 각 축의 invariant를 누락 없이 검증하는 것**이다.

<a id="p12"></a>
## [Thread 13 / `test(routes): 홈과 route presentation 계약 검증`] Route × Design 상태 공간과 계약 기반 테스트

### 면접 질문

다섯 디자인과 여러 route를 지원하는 시스템에서 대표 화면 몇 개만 수동으로 테스트하면 어떤 누락이 생기는가? route와 design의 Cartesian product를 무작정 전부 스냅샷으로 만드는 대신, 공통 invariant와 route별 evidence를 어떻게 나누겠는가?

꼬리 질문:

- 활성화된 route 목록을 테스트 파일에 복사하면 production 설정과 어떻게 drift하는가?
  - 모범답변: 페이지 flag나 프로젝트 활성 상태가 바뀌어도 복사된 배열은 그대로라 비활성 route를 계속 검사하거나 새 공개 route를 누락합니다. 원본 `site-matrix.ts`는 `site.json`의 page 설정과 첫 enabled project에서 route 목록을 파생합니다.
- 동적 프로젝트 route를 위한 fixture가 없을 때 테스트를 skip하는 것과 즉시 실패하는 것 중 무엇이 적절한가?
  - 모범답변: 이 프로젝트는 적어도 하나의 enabled project가 제품·readiness 전제이므로 suite 로딩 단계에서 즉시 실패하는 것이 적절합니다. skip하면 가장 중요한 동적 route 계약이 빠진 채 전체가 녹색으로 보입니다.
- 모든 조합에서 확인해야 할 공통 계약과 일부 route만 확인해야 할 세부 계약의 예를 들어라.
  - 모범답변: 모든 조합에는 응답 성공, 기대 design root, main·h1, selector, viewport overflow를 확인합니다. 프로젝트 상세의 stack·highlight·이미지, resume의 training처럼 route 고유 evidence는 해당 경로에서만 검사합니다.
- 개발 서버 cold route 경쟁 때문에 worker를 1로 제한하는 선택의 비용은 무엇인가?
  - 모범답변: compiler가 동시에 cold route를 무효화하는 flakiness는 줄지만 전체 E2E 시간이 늘고 병렬 격리 문제를 가릴 수 있습니다. production build를 먼저 만들거나 worker별 서버·port를 분리하면 비용을 줄일 수 있습니다.
- characterization test가 리팩터링을 보호하는 동시에 구현 세부에 과결합되지 않게 하려면 무엇을 assert해야 하는가?
  - 모범답변: class 이름이나 전체 DOM 문자열보다 heading·landmark·접근 가능한 이름·href·원본 콘텐츠 evidence처럼 사용자와 route가 관찰하는 계약을 assert합니다. 디자인 root 같은 의도적인 public marker만 최소한으로 고정합니다.

### 30초 모범 답변

지원 상태는 design과 enabled route의 조합이므로 테스트 matrix도 같은 원천에서 파생해야 합니다. 각 조합에서는 응답 성공, 선택한 design root, 현재 route와 query 보존, 핵심 landmark처럼 공통 invariant를 확인하고, 프로젝트·이력서·여정 evidence는 해당 route 계약에서 따로 봅니다. 동적 route fixture가 없으면 제품 전제가 깨진 것이므로 suite 시작 단계에서 실패시킵니다. DOM class나 전체 markup보다 사용자에게 보이는 역할·링크·콘텐츠 소유권을 assert해야 리팩터링 내성이 생깁니다.

### 답변 핵심 키워드

state space, Cartesian product, single matrix source, enabled routes, fixture invariant, common contract, route evidence, characterization, semantic assertion, drift prevention

### 백지 구현

**구현 목표**

디자인 목록과 페이지 활성화 설정에서 실행할 route×design case를 결정적으로 생성한다. 동적 프로젝트 route를 만들 수 없는 경우 즉시 실패한다.

**인터페이스 또는 함수 시그니처**

```ts
type RouteDefinition = {
  id: string;
  path: string | ((fixtureId: string) => string);
  pageFlag?: string;
};

type MatrixCase = {
  designId: string;
  routeId: string;
  path: string;
};

export function buildRouteDesignMatrix(options: {
  designIds: readonly string[];
  routes: readonly RouteDefinition[];
  enabledPages: Readonly<Record<string, boolean | undefined>>;
  fixtureProjectId?: string;
}): MatrixCase[] {
  const duplicate = (values: readonly string[]) => {
    const seen = new Set<string>();
    return values.find((value) => {
      if (seen.has(value)) return true;
      seen.add(value);
      return false;
    });
  };

  const duplicateDesign = duplicate(options.designIds);
  if (duplicateDesign !== undefined) {
    throw new Error(`Duplicate design id "${duplicateDesign}".`);
  }
  const duplicateRoute = duplicate(options.routes.map((route) => route.id));
  if (duplicateRoute !== undefined) {
    throw new Error(`Duplicate route id "${duplicateRoute}".`);
  }

  const enabledRoutes = options.routes
    .filter(
      (route) =>
        route.pageFlag === undefined ||
        options.enabledPages[route.pageFlag] !== false,
    )
    .map((route) => {
      if (typeof route.path === "string") return { ...route, path: route.path };

      // 동적 route를 조용히 빼면 matrix가 거짓 통과하므로 먼저 실패한다.
      const fixtureId = options.fixtureProjectId?.trim();
      if (!fixtureId) {
        throw new Error(`Route "${route.id}" requires a project fixture.`);
      }
      return { ...route, path: route.path(fixtureId) };
    });

  const cases: MatrixCase[] = [];
  for (const designId of options.designIds) {
    for (const route of enabledRoutes) {
      cases.push({ designId, routeId: route.id, path: route.path });
    }
  }
  return cases;
}
```

**입력과 출력**

- 입력: 지원 design ID, route 정의, 페이지 활성화 설정, 선택적 동적 route fixture
- 출력: 각 지원 조합이 정확히 한 번 들어간 case 배열
- 실패: 중복 design·route ID, 필요한 동적 fixture 누락

**반드시 만족해야 할 조건**

- 비활성 페이지의 route를 제외한다.
- page flag가 없는 home 같은 route는 항상 포함한다.
- 각 enabled route와 각 design의 조합을 정확히 한 번 생성한다.
- 결과 순서는 design 순서와 route 정의 순서에 따라 결정적이다.
- 동적 route 함수는 fixture ID가 있을 때만 호출한다.
- 입력 배열을 변경하지 않는다.

**경계 조건**

- 빈 design 목록
- 모든 선택 페이지가 비활성인 경우
- 중복 design ID, 중복 route ID
- 빈 fixture ID
- route path가 query나 hash를 이미 포함한 경우

**필요한 제약**

- 브라우저 실행과 assertion은 범위 밖이다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] enabled route 수 × design 수만큼 case가 생기는가?
- [ ] 비활성 route가 완전히 제외되는가?
- [ ] home route는 페이지 flag와 무관하게 포함되는가?
- [ ] 동적 fixture가 없을 때 조용히 skip하지 않고 실패하는가?
- [ ] 같은 조합이 중복되지 않는가?
- [ ] 입력 순서를 보존한 결정적 결과인가?
- [ ] 원본 입력 배열을 변형하지 않는가?
- [ ] matrix 생성과 실제 assertion 책임이 분리돼 있는가?

### 구현 후 설명할 것

1. 지원 상태 공간을 테스트와 제품 설정에서 함께 파생하는 방법
   - 모범답변: 지원 design ID는 registry의 목록, route 활성 상태는 `site.json`, 동적 fixture는 enabled project에서 가져옵니다. 원본 matrix helper와 presentation 테스트가 같은 다섯 design 계약을 확인해 수동 복사 drift를 줄입니다.
2. 공통 invariant와 route별 evidence를 나눈 기준
   - 모범답변: 응답·design root·landmark·탐색처럼 모든 renderer가 지켜야 할 것은 matrix 전 조합에 적용합니다. 특정 route가 소유한 프로젝트 기술, 이력서 항목, 여정 milestone은 그 route의 characterization helper에서만 확인합니다.
3. fixture 누락을 fail-fast로 처리한 이유
   - 모범답변: 동적 route를 생성할 수 없다는 것은 테스트 편의 문제가 아니라 "enabled project가 존재한다"는 제품 전제가 깨진 상태입니다. suite 시작 때 실패해야 해당 축이 검사되지 않았다는 사실이 숨지 않습니다.
4. 전체 DOM snapshot보다 semantic assertion을 선호한 이유
   - 모범답변: 전체 snapshot은 wrapper·class·공백 변경에도 흔들리지만 사용자가 보는 의미 회귀를 더 잘 잡는 것은 role, heading, link, visible evidence입니다. 시각 자체가 계약인 소수 화면만 별도 visual snapshot으로 관리합니다.
5. worker 직렬화로 flakiness를 줄일 때 잃는 병렬성과 대안
   - 모범답변: 실행 시간이 늘고 독립 테스트의 병렬 이점을 잃습니다. 원본은 개발 compiler와 두 device project의 cold route 경쟁 때문에 worker 1을 택했으며, 미리 만든 production server나 worker별 격리 server를 쓰면 다시 병렬화할 수 있습니다.

### 원본 확인 위치

- Thread 13
- 대표 커밋: `test(routes): 홈과 route presentation 계약 검증`
- 연관 커밋: `test(ui): 디자인 선택과 프로젝트 링크 계약 검증`, `test(e2e): production server 검증 경로 추가`, `test(portfolio): selector와 presentation 회귀 계약 보강`
- 파일: `src/app/page.test.tsx`, `src/app/routes.test.tsx`, `tests/e2e/site-matrix.ts`, `tests/e2e/portfolio.spec.ts`, `playwright.config.ts`, `playwright.production.config.ts`
- 함수·상수: `designIds`, `enabledRoutes`, `firstEnabledProject`, `withExplicitDesign`
- 관련 Thread: 03, 05, 06, 07, 08, 09, 10, 11, 12

---

<a id="p13"></a>
## [Thread 13 / `fix(a11y): 디자인별 색상 대비 보정`] 자동 접근성 검사와 명시적 키보드·동작 계약

### 면접 질문

모든 route×design 조합에 axe 검사를 실행하는 것만으로 접근성 회귀를 충분히 막을 수 있는가? landmark 개수, skip link focus 이동, reduced-motion 같은 명시적 행동 계약을 왜 별도로 테스트해야 하는가?

꼬리 질문:

- 자동 규칙 검사에서 violation이 0이어도 실제 키보드 사용이 막힐 수 있는 예는 무엇인가?
  - 모범답변: skip link가 DOM과 href를 갖춰 audit은 통과하지만 Enter 뒤 main에 programmatic focus가 가지 않거나, 닫힌 메뉴 안에 focus가 남는 경우가 있습니다. 제품의 실제 키보드 순서는 별도 행동 테스트가 필요합니다.
- `main`, `banner`, `contentinfo`를 정확히 하나로 제한한 이유는 무엇인가?
  - 모범답변: 페이지의 주요 콘텐츠·사이트 header·footer라는 최상위 정보 구조를 보조기기가 모호하지 않게 탐색하도록 하기 위해서입니다. 0개는 구조가 없고 복수는 어느 영역이 대표인지 혼동을 줍니다.
- skip link를 클릭했을 때 URL hash만 바뀌고 main이 focus되지 않으면 어떤 문제가 남는가?
  - 모범답변: 화면은 이동해 보여도 키보드 focus가 navigation에 남아 사용자가 반복 링크를 다시 지나야 할 수 있습니다. 원본 테스트는 첫 Tab이 skip link인지와 Enter 뒤 `main` focus를 모두 확인합니다.
- reduced-motion 검사를 특정 클래스 몇 개가 아니라 계산된 스타일과 pseudo-element까지 확장한 이유는 무엇인가?
  - 모범답변: 디자인별 CSS나 `::before`·`::after` 장식에 새 animation이 추가되면 알려진 클래스 목록은 놓칩니다. 원본은 모든 요소와 pseudo-element의 computed duration, infinite iteration, 실제 running animation을 검사합니다.
- 접근성 violation 출력에 design, path, rule ID, target, failure summary를 넣어야 하는 이유는 무엇인가?
  - 모범답변: 같은 rule이 수십 route×design 조합에서 실행되므로 matrix 좌표와 정확한 DOM target이 없으면 재현 위치를 찾기 어렵습니다. impact와 failure summary까지 있으면 수정 우선순위와 원인을 로그만으로 판단할 수 있습니다.
- 색상 대비 수정과 시각 회귀 snapshot 갱신을 어떻게 구분할 것인가?
  - 모범답변: 먼저 axe와 계산된 색상으로 접근성 요구를 충족한 이유를 확인하고, 그 변경으로 생긴 의도적 픽셀 차이만 리뷰 후 baseline에 반영합니다. snapshot 갱신 자체를 수정으로 취급해 무심코 회귀를 승인하지 않습니다.

### 30초 모범 답변

axe는 표준 규칙 위반을 넓게 잡지만 키보드 흐름이나 제품별 상호작용 의도를 모두 증명하지는 못합니다. 그래서 모든 enabled route와 design에 자동 규칙을 적용하면서, landmark가 하나씩 존재하는지, 첫 Tab이 skip link로 가고 Enter 뒤 main에 focus가 이동하는지 별도 시나리오로 검증했습니다. reduced-motion은 CSS 선언 누락이 디자인별로 생길 수 있어 실제 계산된 animation·transition과 pseudo-element까지 확인해야 합니다. 실패 메시지는 어떤 조합과 DOM target이 원인인지 바로 알 수 있어야 합니다.

### 답변 핵심 키워드

automated audit limits, semantic landmark, keyboard flow, skip link, programmatic focus, reduced motion, computed style, pseudo-element, actionable diagnostics

### 백지 구현

이 항목의 백지 구현은 제품 UI가 아니라 **접근성 계약을 실행하는 테스트 helper**다.

**구현 목표**

추상화된 브라우저 driver를 사용해 한 route×design 조합의 기본 접근성 계약을 검사하고, violation을 진단 가능한 문자열로 만든다.

**인터페이스 또는 함수 시그니처**

```ts
type A11yViolation = {
  id: string;
  impact?: string;
  nodes: Array<{
    targets: string[];
    failureSummary?: string;
  }>;
};

type AccessibilityDriver = {
  goto: (url: string) => Promise<{ ok: boolean }>;
  countRole: (role: "main" | "banner" | "contentinfo") => Promise<number>;
  activeDesign: () => Promise<string | null>;
  runAudit: () => Promise<A11yViolation[]>;
};

export function formatAccessibilityViolations(options: {
  designId: string;
  path: string;
  violations: readonly A11yViolation[];
}): string {
  return options.violations
    .map(
      (violation) =>
        `${options.designId} ${options.path}: ${violation.id} (${violation.impact ?? "unknown"})\n${violation.nodes
          .map((node) => {
            const target =
              node.targets.length > 0
                ? node.targets.join(" ")
                : "<unknown target>";
            return `  ${target}\n  ${node.failureSummary ?? "No failure summary."}`;
          })
          .join("\n")}`,
    )
    .join("\n\n");
}

export async function verifyAccessibilityCase(options: {
  driver: AccessibilityDriver;
  designId: string;
  path: string;
}): Promise<void> {
  const { driver, designId, path } = options;
  const context = `${designId} ${path}`;
  let response: { ok: boolean };

  try {
    response = await driver.goto(path);
  } catch (error) {
    throw new Error(`${context}: navigation failed`, { cause: error });
  }
  if (!response?.ok) throw new Error(`${context}: response was not successful`);

  const actualDesign = await driver.activeDesign();
  if (actualDesign !== designId) {
    throw new Error(
      `${context}: expected active design "${designId}", received ${JSON.stringify(actualDesign)}`,
    );
  }

  for (const role of ["main", "banner", "contentinfo"] as const) {
    const count = await driver.countRole(role);
    if (count !== 1) {
      throw new Error(`${context}: expected one ${role} landmark, received ${count}`);
    }
  }

  let violations: A11yViolation[];
  try {
    violations = await driver.runAudit();
  } catch (error) {
    // audit reject는 violation 0개와 다른 실행 실패다.
    throw new Error(`${context}: accessibility audit failed`, { cause: error });
  }
  if (violations.length > 0) {
    throw new Error(
      formatAccessibilityViolations({ designId, path, violations }),
    );
  }
}
```

**입력과 출력**

- 입력: 브라우저 driver, 예상 design ID, route path
- 출력: 계약을 만족하면 `void`
- 실패: 응답, design root, landmark, audit violation 중 하나라도 잘못되면 위치 정보가 있는 오류

**반드시 만족해야 할 조건**

- 네트워크 응답이 성공해야 한다.
- 실제 활성 design이 기대값과 같아야 한다.
- `main`, `banner`, `contentinfo`가 각각 정확히 하나여야 한다.
- 모든 audit violation을 한 번에 포맷한다.
- 빈 target이나 failure summary가 있어도 formatter가 실패하지 않는다.
- 오류 문자열에 design ID와 path를 포함한다.
- skip link와 reduced-motion 시나리오는 이 helper와 별도 행동 테스트로 남겨야 함을 설명한다.

**경계 조건**

- 응답 객체가 없거나 실패인 경우
- landmark 0개 또는 2개 이상
- violation 여러 개와 node 여러 개
- impact·summary 누락
- audit 자체가 reject하는 경우

**필요한 제약**

- 실제 axe API와 Playwright locator 문법은 범위 밖이다.
- 구현 시간은 20~25분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 case가 통과하는가?
- [ ] 잘못된 design root를 감지하는가?
- [ ] 각 landmark의 0개·복수 개를 구분해 실패하는가?
- [ ] 여러 violation과 node가 모두 오류 문자열에 포함되는가?
- [ ] 누락된 impact·summary 때문에 formatter가 예외를 내지 않는가?
- [ ] design과 path가 모든 실패 진단에 포함되는가?
- [ ] audit reject를 violation 0개로 오인하지 않는가?
- [ ] 자동 audit로 대체할 수 없는 키보드·focus·reduced-motion 테스트를 설명할 수 있는가?

### 구현 후 설명할 것

1. 자동 접근성 규칙과 수동·행동 계약의 역할 차이
   - 모범답변: axe는 WCAG 규칙으로 정적으로 판별 가능한 문제를 모든 조합에 넓게 적용합니다. skip focus, menu keyboard flow, reduced-motion처럼 실제 입력과 시간에 의존하는 제품 계약은 명시적 Playwright 시나리오가 보완합니다.
2. landmark를 정확히 하나로 제한한 정보 구조상의 이유
   - 모범답변: 각 디자인이 같은 shell 정보를 제공한다는 공통 계약입니다. main·banner·contentinfo가 하나씩 있어야 보조기기가 대표 영역을 일관되게 찾고 renderer가 중첩 shell을 만든 회귀도 잡을 수 있습니다.
3. skip link의 표시와 실제 focus 이동을 함께 봐야 하는 이유
   - 모범답변: focus했을 때 링크가 보이는 것은 발견 가능성이고, activation 뒤 main이 focus되는 것은 반복 navigation을 실제로 건너뛰는 동작입니다. 둘 중 하나만 확인하면 키보드 흐름이 완성됐다고 할 수 없습니다.
4. reduced-motion을 계산된 스타일 수준에서 검증한 이유
   - 모범답변: 선언 파일이나 class 목록이 아니라 브라우저가 최종 적용한 animation·transition을 봐야 cascade, 디자인별 override, pseudo-element 누락을 잡습니다. 원본은 duration 1ms 이하, infinite 없음, scroll auto까지 확인합니다.
5. 실패 메시지를 matrix 좌표와 DOM target 중심으로 설계한 이유
   - 모범답변: 동일 audit이 많은 조합을 순회하므로 design과 path가 재현 좌표이고 target과 summary가 수정 위치입니다. 모든 violation을 한 번에 포맷하면 한 실행으로 관련 노드를 함께 고칠 수 있습니다.

### 원본 확인 위치

- Thread 13
- 대표 커밋: `fix(a11y): 디자인별 색상 대비 보정`
- 연관 커밋: `test(routes): 홈과 route presentation 계약 검증`, `test(e2e): production server 검증 경로 추가`
- 파일: `tests/e2e/accessibility.spec.ts`, `tests/e2e/site-matrix.ts`, `tests/e2e/portfolio.spec.ts`, `src/app/globals.css`, `src/designs/editorial/editorial-route.module.css`
- 함수: `formatViolations`, `expectAccessibleRoute`
- 관련 Thread: 07, 09, 10, 11, 12, 14
