# 도메인 경계·라우팅·렌더러 면접 워크북

이 문서는 콘텐츠를 애플리케이션 전체에 그대로 노출하지 않고, facade·route view model·URL context·renderer registry를 통해 **읽기 범위와 상태 조합을 제한하는 설계**를 다룬다. 프레임워크 API 이름보다 aliasing, discriminated union, hydration, lazy boundary를 설명하는 데 초점을 둔다.

<a id="p07"></a>
## [Thread 01·13 / `refactor(content): 검증된 콘텐츠를 portfolio facade에 연결` · `test(portfolio): selector와 presentation 회귀 계약 보강`] Canonical facade와 선택적 복제 경계

### 면접 질문

검증된 콘텐츠를 하나의 facade에서 제공하더라도 호출자끼리 객체를 공유하면 어떤 문제가 생길 수 있는가? 이 프로젝트에서 `getPortfolioContent()` 호출마다 어떤 값은 새 참조로 만들고 어떤 값은 공유하도록 계약한 이유를 설명하라.

꼬리 질문:

- 모든 값을 매번 `structuredClone`하는 것과 필요한 하위 트리만 복제하는 것의 trade-off는 무엇인가?
  - 모범답변: 전체 복제는 격리 범위는 넓지만 매 호출 비용이 크고 어떤 값이 실제로 mutable인지 계약이 흐려집니다. 원본은 프로젝트와 활성 링크처럼 호출자가 다룰 수 있는 하위 트리만 새로 만들고 site·profile·presentation 같은 불변 설정은 구조적으로 공유합니다.
- 배열만 새로 만들고 배열 원소 객체를 공유하면 mutation 격리가 충분한가?
  - 모범답변: 배열의 추가·삭제는 격리되지만 원소의 필드나 중첩 배열을 바꾸면 canonical source와 다른 snapshot에 누출됩니다. 변경 가능한 계약이라면 원소와 그 안의 `links`, `placements`까지 필요한 깊이로 복제해야 합니다.
- 읽기 전용 타입만 붙이면 런타임 mutation을 완전히 막을 수 있는가?
  - 모범답변: `readonly`는 TypeScript에서의 쓰기를 막을 뿐 JavaScript 객체를 freeze하지 않습니다. 타입 우회, 외부 JavaScript, 공유된 mutable API를 통한 변경은 가능하므로 런타임 격리가 필요하면 복제나 `Object.freeze`가 별도로 필요합니다.
- 선택기(selector)가 정렬을 위해 원본 배열을 직접 `sort`하면 어떤 회귀가 생기는가?
  - 모범답변: `sort`는 배열을 제자리에서 바꾸므로 이후 selector와 renderer의 순서가 호출 순서에 따라 달라집니다. 원본의 그룹·여정 정렬처럼 `slice().sort()` 또는 새 배열 정렬로 canonical 입력을 보존해야 합니다.
- facade의 공개 export 목록을 계약 테스트로 고정하는 장단점은 무엇인가?
  - 모범답변: 의도치 않은 내부 helper 노출과 API 제거를 즉시 잡는 장점이 있습니다. 반면 정상적인 export 추가도 테스트 수정을 요구해 리팩터링 자유도를 낮추므로, 정말 안정적으로 제공할 facade에만 적용해야 합니다.

### 30초 모범 답변

검증된 singleton 데이터를 그대로 반환하면 한 테스트나 렌더러의 mutation이 다음 호출에 누출될 수 있습니다. 반대로 전체 deep clone은 비용과 의미가 불분명합니다. 그래서 변경 가능성이 있는 프로젝트·링크 계층은 호출마다 새 배열과 필요한 중첩 객체를 만들고, 실제로 불변으로 취급하는 site·profile·presentation 같은 값은 공유하는 경계를 테스트로 고정했습니다. 선택기는 원본을 정렬하지 않고 복사본을 사용해 deterministic한 결과와 mutation 격리를 함께 보장해야 합니다.

### 답변 핵심 키워드

facade, aliasing, mutation isolation, selective clone, structural sharing, deterministic selector, `slice().sort`, public surface contract

### 백지 구현

**구현 목표**

원본 콘텐츠에서 호출자별 snapshot을 생성한다. 프로젝트와 링크는 중첩 변경이 서로 누출되지 않아야 하고, 명시적으로 불변인 설정 객체는 참조를 공유할 수 있다.

**인터페이스 또는 함수 시그니처**

```ts
type SourceContent = {
  site: Readonly<{ title: string }>;
  profile: Readonly<{ name: string }>;
  presentation: Readonly<{ defaultView: string }>;
  links: Array<{ id: string; placements: string[] }>;
  projects: Array<{
    id: string;
    links: Array<{ type: string; href: string }>;
  }>;
};

export function createContentSnapshot(
  source: SourceContent,
): SourceContent {
  return {
    // 불변으로 계약한 설정은 구조적으로 공유한다.
    site: source.site,
    profile: source.profile,
    presentation: source.presentation,
    links: source.links.map((link) => ({
      ...link,
      placements: [...link.placements],
    })),
    projects: source.projects.map((project) => ({
      ...project,
      links: project.links.map((link) => ({ ...link })),
    })),
  };
}
```

**입력과 출력**

- 입력: 한 번 검증된 canonical source
- 출력: 호출자에게 반환할 snapshot

**반드시 만족해야 할 조건**

- 반환 객체 자체는 매 호출 새 참조다.
- `projects`, 각 project 객체, 각 project의 `links` 배열과 원소는 호출 간 mutation을 공유하지 않는다.
- 최상위 `links` 배열과 원소, `placements` 배열도 호출 간 공유하지 않는다.
- `site`, `profile`, `presentation`은 불변 계약 아래 공유해도 된다.
- 입력 source는 변경하지 않는다.
- 값의 내용과 순서는 보존한다.

**경계 조건**

- 빈 프로젝트·링크 배열
- 같은 링크 객체가 여러 위치에서 재사용된 source
- 호출자가 첫 snapshot의 깊은 배열을 수정한 뒤 두 번째 snapshot을 요청하는 경우
- 원본이 이미 frozen된 경우

**필요한 제약**

- 직렬화 기반 deep clone은 사용하지 않는다.
- Date, Map 같은 비JSON 값은 입력에 없다고 가정한다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 두 snapshot의 최상위 참조가 다른가?
- [ ] 프로젝트 배열·프로젝트 객체·중첩 링크가 모두 격리되는가?
- [ ] 첫 snapshot을 수정해도 두 번째 snapshot과 source가 바뀌지 않는가?
- [ ] 공유하기로 한 불변 객체는 실제로 같은 참조인가?
- [ ] 빈 배열과 중복 참조 source에서도 값이 보존되는가?
- [ ] 불필요한 전체 deep clone을 하지 않았는가?
- [ ] 어떤 subtree를 공유하고 복제하는지 계약으로 설명할 수 있는가?

### 구현 후 설명할 것

1. 선택적 복제와 전체 deep clone의 비용·안전성 trade-off
   - 모범답변: 선택적 복제는 변경 가능한 경로만 비용을 내고 불변 객체의 참조 동일성을 유지하지만, 새 mutable 필드를 추가할 때 복제 계약도 갱신해야 합니다. 전체 deep clone은 누락 위험은 줄지만 비용이 크고 지원하지 않는 값 타입과 참조 의미가 달라질 수 있습니다.
2. 타입의 `readonly`와 런타임 mutation 격리의 차이
   - 모범답변: `readonly`는 컴파일 타임 사용 규칙이고 snapshot 복제는 호출자 사이 alias를 실제로 끊는 런타임 동작입니다. 원본은 공유하기로 한 객체와 복제할 배열 경계를 테스트로 함께 고정합니다.
3. facade가 canonical source를 직접 노출하지 않는 이유
   - 모범답변: 한 호출자의 변경이 전역 singleton에 반영되면 다음 렌더와 테스트의 결과가 오염됩니다. facade는 활성 항목을 필터링하고 변경 가능한 컬렉션을 새로 만들어 검증된 원본의 ownership을 유지합니다.
4. 정렬·필터 선택기가 원본을 변형하지 않아야 하는 이유
   - 모범답변: selector는 같은 입력에 같은 결과를 주는 파생 함수여야 조합 순서와 테스트 순서에 독립적입니다. 제자리 정렬은 숨은 상태 전이를 만들기 때문에 복사본을 정렬하고 `filter`처럼 새 배열을 반환하는 연산을 사용합니다.
5. 공개 export 계약 테스트가 리팩터링 자유도에 주는 영향
   - 모범답변: 내부 이름 변경은 자유롭지만 facade의 공개 목록을 바꾸면 의도적 API 변경으로 처리하게 합니다. 안정성은 높아지는 대신 공개 surface 확장·축소마다 계약 테스트 검토 비용이 생깁니다.

### 원본 확인 위치

- Thread 01, 13
- 대표 커밋: `refactor(content): 검증된 콘텐츠를 portfolio facade에 연결`
- 연관 커밋: `feat(content): 여정 정렬과 콘텐츠 인덱스 구성`, `feat(portfolio): 기술과 프로젝트 조회기 추가`, `feat(portfolio): 연락과 프로젝트 링크 선택기 추가`, `test(portfolio): selector와 presentation 회귀 계약 보강`
- 파일: `src/lib/portfolio/content.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio.ts`, `src/lib/portfolio.test.ts`
- 함수: `getPortfolioContent`, `getFeaturedProjects`, `getProjectById`, `getResumeProjects`, `getPreferredContactLinks`, `getProjectCardLinks`, `getProjectDetailLinks`
- 관련 Thread: 02, 06, 13

---

<a id="p08"></a>
## [Thread 06 / `refactor(content): 홈 route view model 경계 추가`] Route별 최소 view model과 선계산된 참조

### 면접 질문

여러 renderer가 동일한 전체 `PortfolioContent`를 받아 내부에서 필요한 프로젝트를 다시 찾도록 두지 않고, route별 view model을 만드는 이유는 무엇인가? 타입 수준과 런타임 수준에서 데이터 최소화를 어떻게 보장할 수 있는가?

꼬리 질문:

- `Pick`으로 허용 필드를 선언하는 것과 제외 필드를 `never`로 만드는 방식은 각각 무엇을 잡아내는가?
  - 모범답변: `Pick`은 view model을 만들 때 허용한 원본 필드만 타입에 포함시킵니다. 원본의 `never` 매핑은 기존 renderer가 넓은 타입을 기대하는 전환기에도 금지 필드 접근이 컴파일 오류가 되게 하며, 실제 런타임 객체에는 그 값을 넣지 않습니다.
- ID 배열을 순서대로 실제 객체에 해석할 때 `Map`을 쓰면 어떤 복잡도 이점이 있는가?
  - 모범답변: 객체 N개를 한 번 색인하고 참조 R개를 순회하면 O(N+R)이며 ID 배열의 순서도 그대로 보존됩니다. 매 ID마다 `find`하면 O(N×R)이 될 수 있습니다.
- 존재하지 않는 참조를 `null`, 빈 배열, 예외 중 무엇으로 표현할지 어떻게 정하는가?
  - 모범답변: 단일 선택 항목의 부재가 정상 상태면 `null`, 0개 이상 관계면 빈 배열이 자연스럽습니다. 검증 완료가 사전 조건인 필수 참조가 빠졌다면 예외로 경계 위반을 드러내고, 원본의 route creator는 route 자체가 없을 때 `null`을 반환합니다.
- view model에 현재 연도를 넣을 때 테스트 가능성을 위해 시간을 어떻게 주입할 것인가?
  - 모범답변: 원본 `createHomeViewModel(content, now = new Date())`처럼 `Date`를 선택 인자로 받아 기본은 현재 시간, 테스트는 고정 시간을 넘깁니다. 전역 clock을 직접 읽는 지점을 함수 경계 하나로 제한합니다.
- renderer에서 추가 조인을 허용하면 어떤 아키텍처 회귀가 생기는가?
  - 모범답변: renderer가 다시 전체 콘텐츠와 ID lookup에 의존해 디자인마다 조인 규칙과 누락 처리 방식이 갈라집니다. view model의 최소 데이터·선계산 계약도 무력화돼 route와 presentation 경계가 다시 결합됩니다.

### 30초 모범 답변

전체 콘텐츠를 renderer에 넘기면 각 화면이 임의 필드에 결합되고 같은 ID 조인을 반복합니다. route별 view model은 필요한 원본 필드만 `Pick`하고 파생 값과 참조 해석을 서버 경계에서 한 번 수행해 renderer를 단순한 소비자로 만듭니다. 순서가 있는 참조는 `Map`으로 O(N+R)에 해석하고, 누락 처리 규칙은 route 계약에 맞게 `null` 또는 제외로 고정했습니다. 테스트에서는 공개 키 목록과 "전체 content spread 금지"를 함께 검사해 타입과 런타임 경계를 지켰습니다.

### 답변 핵심 키워드

least privilege, route projection, `Pick`, excluded `never`, precomputed join, `Map`, order preservation, nullability contract, time injection

### 백지 구현

**구현 목표**

프로젝트 목록 화면에 필요한 데이터만 반환하는 view model 생성기를 작성한다. renderer가 그룹 정렬, featured 필터, metric 조회를 다시 수행하지 않게 한다.

**인터페이스 또는 함수 시그니처**

```ts
type Content = {
  site: { title: string };
  profile: { name: string };
  presentation: { defaultView: string };
  contact: { email?: string };
  projects: Array<{
    id: string;
    groupId: string;
    featured: boolean;
  }>;
  projectGroups: Array<{ id: string; order: number; label: string }>;
  privateNotes: string[];
};

type ProjectIndexViewModel = {
  route: "projects";
  site: Content["site"];
  profile: Content["profile"];
  presentation: Content["presentation"];
  contact: Content["contact"];
  groups: Array<{
    id: string;
    label: string;
    projects: Content["projects"];
  }>;
  featuredProjects: Content["projects"];
};

export function createProjectIndexViewModel(
  content: Content,
): ProjectIndexViewModel {
  const orderedGroups = content.projectGroups
    .map((group, sourceIndex) => ({ group, sourceIndex }))
    .sort(
      (left, right) =>
        left.group.order - right.group.order ||
        left.sourceIndex - right.sourceIndex,
    )
    .map(({ group }) => group);
  const projectsByGroup = new Map<string, Content["projects"]>();

  for (const project of content.projects) {
    projectsByGroup.set(project.groupId, [
      ...(projectsByGroup.get(project.groupId) ?? []),
      project,
    ]);
  }

  const configuredGroups = orderedGroups
    .map((group) => ({
      id: group.id,
      label: group.label,
      projects: projectsByGroup.get(group.id) ?? [],
    }))
    .filter((group) => group.projects.length > 0);
  const configuredGroupIds = new Set(
    configuredGroups.map((group) => group.id),
  );
  // 사전 validation을 우회한 ID도 임의 그룹에 넣지 않고 자체 ID로 보존한다.
  const unconfiguredGroups = [...projectsByGroup.entries()]
    .filter(([groupId]) => !configuredGroupIds.has(groupId))
    .sort(([left], [right]) => left.localeCompare(right))
    .map(([groupId, projects]) => ({
      id: groupId,
      label: groupId,
      projects,
    }));

  return {
    route: "projects",
    site: content.site,
    profile: content.profile,
    presentation: content.presentation,
    contact: content.contact,
    groups: [...configuredGroups, ...unconfiguredGroups],
    // filter는 원본 순서를 보존하며 입력을 변경하지 않는다.
    featuredProjects: content.projects.filter((project) => project.featured),
  };
}
```

**입력과 출력**

- 입력: 구조와 참조가 이미 검증된 전체 콘텐츠
- 출력: 프로젝트 목록 route 전용 view model

**반드시 만족해야 할 조건**

- `privateNotes`와 전체 `content` 객체를 반환하지 않는다.
- 그룹은 `order` 오름차순으로 나온다.
- 각 프로젝트는 자신의 그룹에 정확히 한 번 포함된다.
- `featuredProjects`는 원본 프로젝트 순서를 유지한다.
- 입력을 정렬하거나 수정하지 않는다.
- renderer가 추가 lookup 없이 소비할 수 있는 구조다.

**경계 조건**

- 프로젝트가 없는 그룹
- featured 프로젝트가 없는 경우
- 동일 order를 가진 그룹은 원본 순서 등 결정적 tie-break 규칙을 정한다.
- 검증 단계를 우회해 존재하지 않는 groupId가 들어온 경우의 정책을 명시한다.

**실패 조건**

- 잘못된 참조를 조용히 임의 그룹으로 이동하지 않는다.
- view model 생성 중 전체 객체를 spread하지 않는다.

**필요한 제약**

- UI 문자열 조립은 범위 밖이다.
- 구현 시간은 20~25분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 공개 필드가 요구한 키로만 제한되는가?
- [ ] 그룹 순서와 프로젝트 원래 순서가 결정적인가?
- [ ] 모든 프로젝트가 정확히 한 번 포함되는가?
- [ ] featured가 없는 경우 빈 배열을 반환하는가?
- [ ] 입력 배열의 순서가 변하지 않는가?
- [ ] renderer가 ID로 다시 검색할 필요가 없는가?
- [ ] 전체 content spread나 불필요한 깊은 복제가 없는가?
- [ ] 시간·공간 복잡도를 설명할 수 있는가?

### 구현 후 설명할 것

1. view model을 API 경계이자 least-privilege 경계로 본 이유
   - 모범답변: route가 실제로 렌더링할 필드와 파생값만 공개하면 renderer가 `privateNotes` 같은 무관한 데이터에 결합할 수 없습니다. 타입의 허용 키와 런타임 반환 키를 모두 제한해야 이 경계가 실효성이 있습니다.
2. 조인을 renderer가 아니라 view model 생성 단계에 둔 이유
   - 모범답변: 서버 경계에서 한 번 색인하고 순서를 보존해 참조를 해석하면 모든 디자인이 같은 결과와 누락 정책을 소비합니다. renderer는 표현만 담당하고 추가 lookup을 반복하지 않습니다.
3. 누락 참조를 처리하는 정책과 사전 validation의 관계
   - 모범답변: group 참조는 loader가 먼저 보장하지만 원본 view model은 우회 입력도 임의의 기존 그룹으로 옮기지 않고 자체 ID의 후행 그룹으로 보존합니다. 선택적 단일 관계는 `null`, 순서 있는 선택적 목록은 누락 항목을 제외하는 식으로 route 계약을 따로 명시합니다.
4. 객체 참조를 유지하는 것과 복제하는 것의 선택 기준
   - 모범답변: view model이 읽기 전용 투영이고 호출자가 수정하지 않는 계약이면 프로젝트 객체를 공유해 불필요한 복제를 피할 수 있습니다. mutation 격리가 필요한 facade 경계에서는 P07처럼 변경 가능한 하위 트리를 복제합니다.
5. 공개 필드 계약을 테스트하는 방법
   - 모범답변: `Object.keys`로 route별 허용 키를 확인하고 금지된 전체 content 필드가 런타임에 없는지 검사합니다. 타입 테스트에서는 `Pick`과 `never` 필드 접근이 컴파일되지 않는지도 확인합니다.

### 원본 확인 위치

- Thread 06
- 대표 커밋: `refactor(content): 홈 route view model 경계 추가`
- 연관 커밋: `refactor(content): 프로젝트 목록 파생 모델 추가`, `refactor(content): 여정 근거 view model 추가`, `refactor(content): 인터뷰 근거 view model 추가`, `refactor(content): 홈 view model 공개 필드 제한`, `refactor(content): 프로젝트 route 공개 필드 제한`, `refactor(content): 소개·이력·연락 공개 필드 제한`, `refactor(content): 여정·인터뷰 공개 필드 제한`, `test(content): route view model 파생 규칙 검증`
- 파일: `src/lib/portfolio/view-models.ts`, `src/lib/portfolio/view-models.test.ts`
- 함수·타입: `createHomeViewModel`, `createProjectIndexViewModel`, `createProjectDetailViewModel`, `createAboutViewModel`, `createResumeViewModel`, `createContactViewModel`, `createJourneyViewModel`, `createInterviewMapViewModel`, `PortfolioRouteViewModel`
- 관련 Thread: 01, 08, 09, 10, 11, 12

---

<a id="p09"></a>
## [Thread 07 / `refactor(ui): 디자인 선택기를 server markup으로 전환`] Hydration-safe native state와 최소 client island

### 면접 질문

디자인 선택기를 React state로 완전히 제어하지 않고 서버가 `<details>` markup을 렌더링하고 작은 client island만 닫기·focus 복구를 담당하게 한 이유는 무엇인가? hydration 전에 사용자가 바꾼 native `open` 상태를 어떻게 보존해야 하는가?

꼬리 질문:

- 서버 HTML과 첫 클라이언트 render가 다른 `open` 값을 만들면 어떤 문제가 생기는가?
  - 모범답변: hydration mismatch 경고가 나거나 React가 DOM을 다시 맞추며 사용자가 hydration 전에 연 메뉴를 닫을 수 있습니다. 원본은 `<details>`를 uncontrolled로 두고 hydration 경고 억제도 해당 native 상태 경계에만 적용합니다.
- uncontrolled native state를 선택하면 테스트해야 할 경계는 무엇인가?
  - 모범답변: 연결·hydration 전후 `open` 보존, summary의 native toggle, 명시적 닫기, 닫은 뒤 focus, unmount cleanup을 확인해야 합니다. React state 값 대신 실제 DOM 속성과 focus를 계약으로 봅니다.
- JavaScript가 로드되지 않아도 기본 상호작용이 가능한 구조의 장점은 무엇인가?
  - 모범답변: 브라우저 기본 `<details>/<summary>` 동작과 링크 탐색이 즉시 가능해 hydration 지연·실패에도 핵심 기능이 남습니다. client JavaScript는 focus 복구 같은 보강만 담당해 전송·실행 범위도 줄어듭니다.
- 닫기 뒤 focus를 `<summary>`로 돌려야 하는 이유는 무엇인가?
  - 모범답변: 닫힌 패널 안 버튼에 focus가 남으면 키보드 사용자는 현재 위치를 잃고 보이지 않는 요소에 머물 수 있습니다. 메뉴를 다시 열 수 있는 summary로 돌리면 탐색 흐름이 예측 가능합니다.
- 이벤트 listener를 직접 설치한다면 cleanup을 놓쳤을 때 어떤 문제가 생기는가?
  - 모범답변: 재연결 때 handler가 중복 실행되고 제거된 DOM이나 오래된 closure가 남아 메모리와 상태를 오염시킵니다. 설치한 각 listener를 cleanup에서 정확히 제거하고 반복 cleanup도 안전하게 만들어야 합니다.

### 30초 모범 답변

`<details>`는 브라우저가 자체 상태와 키보드 동작을 제공하므로 서버 markup만으로도 열고 닫을 수 있습니다. React가 같은 상태를 다시 소유하면 hydration 전 사용자가 연 메뉴를 초기값으로 덮어쓸 수 있습니다. 그래서 서버 컴포넌트는 semantic markup과 링크를 만들고, 작은 client island는 명시적 닫기와 summary focus 복구만 담당했습니다. 테스트는 hydration 전에 DOM의 `open`을 바꾼 뒤 오류 없이 상태가 보존되는지, unmount 시 자원이 정리되는지 확인해야 합니다.

### 답변 핵심 키워드

progressive enhancement, native state, uncontrolled DOM, hydration race, minimal island, focus restoration, semantic markup, cleanup

### 백지 구현

**구현 목표**

서버가 이미 렌더링한 `<details>` 요소에 최소한의 동작만 추가하는 DOM controller를 작성한다. 초기 `open` 상태를 설정하거나 React state로 미러링하면 안 된다.

**인터페이스 또는 함수 시그니처**

```ts
const detailsControllers = new WeakMap<HTMLDetailsElement, () => void>();

export function attachDetailsController(
  details: HTMLDetailsElement,
): () => void {
  // 중복 연결은 이전 controller를 정리한 뒤 교체하는 정책이다.
  detailsControllers.get(details)?.();

  const summary = details.querySelector<HTMLElement>(":scope > summary");
  const closeButton = details.querySelector<HTMLButtonElement>("button");
  if (!summary || !closeButton) {
    throw new Error("Details controller requires a summary and close button.");
  }

  const closeAndRestoreFocus = () => {
    details.removeAttribute("open");
    summary.focus();
  };
  const closeForNavigation = () => {
    details.removeAttribute("open");
  };
  const links = [...details.querySelectorAll<HTMLAnchorElement>("a[href]")];

  closeButton.addEventListener("click", closeAndRestoreFocus);
  links.forEach((link) => link.addEventListener("click", closeForNavigation));

  let active = true;
  const cleanup = () => {
    if (!active) return;
    active = false;
    closeButton.removeEventListener("click", closeAndRestoreFocus);
    links.forEach((link) =>
      link.removeEventListener("click", closeForNavigation),
    );
    if (detailsControllers.get(details) === cleanup) {
      detailsControllers.delete(details);
    }
  };

  detailsControllers.set(details, cleanup);
  return cleanup;
}
```

**입력과 출력**

- 입력: 내부에 직접 자식 `<summary>`, 닫기 버튼, 탐색 링크가 있는 `<details>`
- 출력: 설치한 listener를 모두 해제하는 cleanup 함수

**반드시 만족해야 할 조건**

- 연결 시점의 `details.open` 값을 변경하지 않는다.
- 닫기 버튼을 누르면 닫고 summary로 focus를 복구한다.
- 내부 탐색 링크가 활성화되면 메뉴를 닫는다.
- `<summary>`의 native toggle 동작을 막지 않는다.
- cleanup 이후에는 controller가 추가한 동작이 실행되지 않는다.
- 같은 요소에 중복 연결하는 경우의 정책을 명시한다.

**경계 조건**

- summary 또는 닫기 버튼이 없는 잘못된 markup
- 이미 닫힌 상태에서 닫기 버튼을 누르는 경우
- 링크 클릭의 기본 탐색이 테스트에서 취소된 경우
- controller 연결 전 사용자가 이미 열어 둔 경우
- cleanup을 두 번 호출하는 경우

**필요한 제약**

- animation 구현은 범위 밖이다.
- native accessibility semantics를 다시 구현하지 않는다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 연결 직전 `open` 값이 연결 뒤 그대로인가?
- [ ] 열림·닫힘 모두에서 닫기 버튼이 안전하게 동작하는가?
- [ ] 닫힌 뒤 summary가 focus를 받는가?
- [ ] 내부 링크 활성화 시 메뉴가 닫히는가?
- [ ] summary 클릭의 기본 toggle이 유지되는가?
- [ ] cleanup 뒤 listener가 더 실행되지 않는가?
- [ ] cleanup을 반복해도 예외가 나지 않는가?
- [ ] JavaScript 없이도 `<details>` 자체는 사용할 수 있는가?

### 구현 후 설명할 것

1. controlled React state보다 native state를 선택한 이유
   - 모범답변: `<details>`가 open 상태와 키보드 toggle semantics를 이미 제공하므로 React가 중복 소유할 필요가 없습니다. 원본은 서버가 semantic markup을 만들고 작은 client 버튼만 명시적 닫기와 focus를 담당합니다.
2. hydration 전 사용자 입력을 보존해야 하는 이유
   - 모범답변: 서버 응답 뒤 hydration까지도 사용자는 native control을 조작할 수 있습니다. 첫 client render가 초기값을 다시 쓰면 실제 사용자 행동을 잃으므로 연결 시 `open`을 읽거나 설정하지 않습니다.
3. client island의 책임을 최소화한 기준
   - 모범답변: 브라우저가 제공하지 않는 닫기 버튼 동작과 focus 복구만 client 책임으로 뒀습니다. 목록·링크·현재 선택 표시와 URL 생성은 서버에서 완성해 hydration 없이도 탐색할 수 있습니다.
4. focus 복구가 접근성 계약인 이유
   - 모범답변: 패널이 닫힐 때 내부 control은 보이지 않게 되므로 focus도 사용 가능한 trigger로 옮겨야 합니다. 그렇지 않으면 키보드와 보조기기 사용자가 다음 위치를 예측하기 어렵습니다.
5. 직접 listener 방식과 React event handler 방식의 trade-off
   - 모범답변: 직접 listener는 이미 렌더된 DOM에 작게 붙일 수 있지만 selector·중복 연결·cleanup을 직접 책임져야 합니다. React handler는 lifecycle 통합이 쉽지만 해당 subtree를 client component 경계로 만들 수 있습니다. 원본은 닫기 버튼만 client component로 택했습니다.

### 원본 확인 위치

- Thread 07
- 대표 커밋: `refactor(ui): 디자인 선택기를 server markup으로 전환`
- 연관 커밋: `refactor(ui): reveal 콘텐츠를 server에서 즉시 표시`
- 파일: `src/components/portfolio/design-switcher.tsx`, `src/components/portfolio/design-switcher-close.tsx`, `src/components/portfolio/design-switcher.test.tsx`
- 컴포넌트: `DesignSwitcher`
- 관련 Thread: 09, 13, 14

---

<a id="p10"></a>
## [Thread 08 / `refactor(routes): renderer view model 요청 타입 추가`] Discriminated union 기반 lazy renderer registry

### 면접 질문

다섯 디자인이 여러 route view model을 렌더링할 때, `route`와 `viewModel.route`가 타입 수준에서 반드시 일치하고 선택한 디자인 모듈만 동적으로 로드되게 하려면 registry 타입을 어떻게 설계하겠는가?

꼬리 질문:

- 단순히 `{ route: RouteId; viewModel: PortfolioRouteViewModel }`로 선언하면 어떤 잘못된 조합이 통과하는가?
  - 모범답변: 두 union이 독립이라 `route: "home"`과 `viewModel.route: "project-detail"`도 각각의 타입에는 맞아 통과합니다. 두 discriminant가 같은 literal이라는 상관관계를 타입에 표현해야 합니다.
- mapped type과 `Extract`를 사용해 상관관계가 있는 union을 만드는 원리를 설명하라.
  - 모범답변: 각 route literal을 mapped type의 key로 순회하면서 그 route와 `Extract<Union, {route: Route}>`만 한 객체에 묶습니다. 마지막에 mapped object를 route key로 인덱싱하면 올바른 쌍들의 union이 됩니다.
- registry key 누락을 컴파일 타임에 잡으려면 어떤 타입을 써야 하는가?
  - 모범답변: `Record<DesignId, Loader>` 또는 `satisfies Record<DesignId, Loader>`를 사용합니다. 새 디자인을 union에 추가하거나 key를 빼면 registry 선언 위치에서 컴파일 오류가 납니다.
- lazy import가 실패하거나 모듈의 default export가 계약을 지키지 않으면 어디서 실패시킬 것인가?
  - 모범답변: 선택한 loader를 호출하는 registry 경계에서 import rejection을 전파하고, JavaScript 모듈처럼 런타임 검증이 필요하면 `default`가 함수인지 확인합니다. 화면별 renderer 내부에서 조용히 `null`로 숨기지 않습니다.
- renderer가 `null`을 반환하는 방어 로직과 exhaustive 타입 검사의 역할은 어떻게 다른가?
  - 모범답변: exhaustive 타입은 지원 route 누락과 잘못된 조합을 빌드에서 막습니다. `null` 반환은 데이터 부재 등 명시된 런타임 상태를 표현할 뿐 타입 계약 누락을 보완하지 못합니다.

### 30초 모범 답변

route ID와 view model을 독립 union으로 두면 `route: "home"`에 project-detail model을 넘기는 조합도 타입상 허용됩니다. 각 route literal을 순회하는 mapped type에서 `Extract<Union, {route: Route}>`로 짝을 만든 뒤 다시 union으로 펼치면 상관관계가 유지됩니다. registry는 모든 design ID를 key로 갖는 lazy loader record로 만들고, 선택된 loader만 import합니다. 타입은 잘못된 조합과 누락을 빌드에서 막고, 로드 실패 같은 런타임 오류는 명시적인 경계에서 처리합니다.

### 답변 핵심 키워드

discriminated union, correlated union, mapped type, `Extract`, exhaustive `Record`, lazy import, module boundary, compile-time/runtime split

### 백지 구현

**구현 목표**

세 route와 두 디자인으로 축소한 타입 안전 lazy registry를 작성한다. 잘못된 route/view model 조합은 컴파일되지 않아야 한다.

**인터페이스 또는 함수 시그니처**

```ts
type RouteViewModel =
  | { route: "home"; headline: string }
  | { route: "projects"; projectIds: string[] }
  | { route: "project-detail"; projectId: string };

type DesignId = "classic" | "editorial";

type RendererProps = {
  [Route in RouteViewModel["route"]]: {
    route: Route;
    viewModel: Extract<RouteViewModel, { route: Route }>;
  };
}[RouteViewModel["route"]];

type RendererModule = {
  default: (props: RendererProps) => unknown;
};

type RendererRegistry = Record<
  DesignId,
  () => Promise<RendererModule>
>;

const rendererRegistry: RendererRegistry = {
  classic: () => import("./classic"),
  editorial: () => import("./editorial"),
};

export async function renderDesignRoute(
  designId: DesignId,
  request: RendererProps,
): Promise<unknown> {
  const loader: (() => Promise<RendererModule>) | undefined =
    rendererRegistry[designId];
  if (!loader) {
    throw new Error(`Unsupported design: ${String(designId)}`);
  }

  // 선택된 디자인 모듈 하나만 이 시점에 로드한다.
  const module = await loader();
  if (typeof module.default !== "function") {
    throw new Error(`Design "${designId}" has no default renderer.`);
  }

  return module.default(request);
}
```

**입력과 출력**

- 입력: 디자인 ID와 route에 맞는 view model 요청
- 출력: 선택한 renderer의 결과
- 실패: loader import 실패 또는 잘못된 런타임 design ID

**반드시 만족해야 할 조건**

- route literal과 view model 변형이 항상 같은 조합이어야 한다.
- registry는 모든 `DesignId`를 정확히 한 번 포함한다.
- 선택하지 않은 디자인 모듈을 미리 import하지 않는다.
- `any`와 광범위한 타입 단언을 사용하지 않는다.
- 새 route 변형을 추가하면 필요한 타입 또는 renderer 수정 지점이 컴파일 오류로 드러난다.
- 런타임에 외부 문자열 design ID를 받을 경우 별도 검증 경계를 둔다.

**경계 조건**

- 존재하지 않는 design ID
- loader가 reject하는 경우
- renderer module에 default export가 없는 경우
- route union에 새 변형을 추가한 경우
- legacy props를 함께 지원한다면 어느 시점에 제거할지 명시한다.

**필요한 제약**

- 실제 React JSX는 필수가 아니다. 타입 관계와 lazy 호출이 핵심이다.
- 구현 시간은 25~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 올바른 세 route 요청이 모두 타입 검사되는가?
- [ ] `home` route와 project-detail model을 섞으면 컴파일 오류가 나는가?
- [ ] registry에서 한 디자인을 빼면 컴파일 오류가 나는가?
- [ ] 호출 시 선택된 loader 하나만 실행되는가?
- [ ] loader reject가 호출자에게 명확히 전파되는가?
- [ ] 새 route를 추가했을 때 누락 지점이 드러나는가?
- [ ] `any`나 강제 cast로 상관관계를 숨기지 않았는가?

### 구현 후 설명할 것

1. 독립 union과 correlated discriminated union의 차이
   - 모범답변: 독립 union 두 개는 각 필드가 허용 집합에 속하는지만 확인합니다. correlated union은 각 객체 변형 안에서 `route`와 해당 view model을 한 쌍으로 묶어 잘못된 교차 조합을 제거합니다.
2. mapped type과 `Extract`가 route/model 관계를 만드는 방식
   - 모범답변: view model union의 route literal마다 객체 변형을 만들고, `Extract`로 그 literal을 가진 model만 선택합니다. 이를 다시 union으로 펼치면 discriminant narrowing도 그대로 작동합니다.
3. exhaustive registry가 확장성에 주는 장점과 수정 비용
   - 모범답변: 지원 디자인의 loader 누락을 즉시 컴파일 오류로 보여 주고 registry가 단일 탐색 지점이 됩니다. 대신 새 디자인을 추가할 때 모듈과 registry를 함께 완성해야 하므로 의도적인 수정 비용이 생깁니다.
4. lazy import의 bundle 이점과 로드 실패 처리
   - 모범답변: 현재 선택한 디자인만 동적으로 import해 다른 renderer 코드를 초기 경로에 싣지 않습니다. 네트워크·모듈 실패는 registry 호출에서 명시적으로 전파해 상위 오류 경계가 처리하게 합니다.
5. legacy 입력 계약을 union에 함께 둘 때의 migration 위험
   - 모범답변: 구형 전체-content props를 함께 허용하면 새 최소 view model을 우회하는 renderer가 계속 생길 수 있습니다. 호환 기간과 호출부 전환 목록, 제거 조건을 정하고 장기 union으로 남기지 않아야 합니다.

### 원본 확인 위치

- Thread 08
- 대표 커밋: `refactor(routes): renderer view model 요청 타입 추가`
- 연관 커밋: `feat(designs): site design 정의 registry 추가`, `feat(designs): route renderer 계약 추가`, `refactor(designs): 확장 renderer lazy registry 추가`, `test(design): view model 기반 renderer matrix 검증`, `refactor(designs): 모든 route를 registry renderer로 위임`
- 파일: `src/designs/config.ts`, `src/designs/types.ts`, `src/designs/registry.tsx`, `src/designs/route-view-models.test.tsx`
- 함수·타입: `SITE_DESIGNS`, `SITE_DESIGN_IDS`, `SiteDesignDefinition`, `PortfolioRouteId`, `DesignRouteRequestProps`, `renderDesignRoute`
- 관련 Thread: 06, 09, 10, 11, 12, 13, 14
