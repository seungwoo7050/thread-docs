# 콘텐츠 무결성·게시 보안 면접 워크북

이 문서는 구조 검증, 교차 파일 정합성, URL·파일 경계, 배포 준비도, 구조화 데이터 직렬화처럼 **입력은 받아들이되 잘못된 상태는 애플리케이션 안쪽으로 통과시키지 않는 설계**를 묶는다. 백지 구현 문제는 원본 코드를 재현하는 문제가 아니라 같은 invariant를 다른 방식으로 만족시키는 문제다.

<a id="p01"></a>
## [Thread 02 / `feat(content): JSON schema 파싱 경계 추가`] 런타임 스키마와 정적 타입의 단일 경계

### 면접 질문

JSON 파일을 TypeScript에서 import하면 타입을 붙일 수 있는데도 런타임 스키마 검증이 필요한 이유는 무엇인가? 이 프로젝트에서는 스키마의 출력 타입과 애플리케이션 도메인 타입이 어긋나지 않도록 어떤 경계를 두었는가?

꼬리 질문:

- `as SomeType` 단언과 런타임 파싱은 실패 시점과 신뢰 수준이 어떻게 다른가?
- 이전 형식과 새 형식을 함께 받는 migration union을 오래 유지하면 어떤 문제가 생기는가?
- 파일별 파싱 오류에 파일명과 JSON 경로가 필요한 이유는 무엇인가?
- 파싱 이후 코드에서 반복적으로 `undefined`를 검사하지 않아도 되게 하려면 어디까지 검증해야 하는가?

### 30초 모범 답변

TypeScript 타입은 빌드 뒤 사라지므로 JSON과 환경 입력은 런타임에 여전히 `unknown`입니다. 그래서 파일을 읽는 한 지점에서 스키마로 파싱하고, 그 스키마의 출력 타입을 도메인 타입의 근원으로 삼았습니다. 실패는 파일명과 JSON 경로를 포함해 경계에서 중단시키고, 안쪽 코드는 검증된 값만 받게 했습니다. migration union은 전환 기간에는 유용하지만 모호한 두 계약을 영구 지원하게 되므로 종료 조건과 제거 시점을 함께 정해야 합니다.

### 답변 핵심 키워드

`unknown`, 타입 소거, 런타임 파싱, `z.output`, 단일 진실 공급원, 경계 검증, 위치가 있는 오류, migration 종료 조건

### 백지 구현

**구현 목표**

파일명과 스키마를 받아 런타임 값을 검증하고, 성공 시 스키마 출력 타입을 그대로 반환하는 공통 파싱 경계를 구현한다. 여러 콘텐츠 파일을 조합하는 로더는 이 함수만 통해 원시 입력을 받아야 한다.

**인터페이스 또는 함수 시그니처**

```ts
import { z } from "zod";

export function parseContentFile<Schema extends z.ZodType>(
  file: string,
  schema: Schema,
  input: unknown,
): z.output<Schema> {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 논리적 파일명, Zod 스키마, 신뢰하지 않는 런타임 값
- 출력: `z.output<Schema>`
- 실패: 파일명과 가능한 한 구체적인 값 경로를 포함한 오류

**반드시 만족해야 할 조건**

- 타입 단언으로 검증을 우회하지 않는다.
- 반환 타입을 별도의 수동 인터페이스와 중복 선언하지 않는다.
- 중첩 객체와 배열의 실패 위치를 식별할 수 있다.
- 성공한 값만 다음 조합 단계로 넘긴다.
- migration 형식을 허용한다면 허용 범위가 스키마에 명시돼야 한다.

**경계 조건**

- 최상위 값이 `null`, 배열, 문자열인 경우
- 중첩 배열의 여러 원소가 동시에 잘못된 경우
- 선택 필드가 누락된 경우와 필수 필드가 누락된 경우
- union의 어느 분기에도 맞지 않는 경우

**필요한 제약**

- 파일 I/O 자체는 범위 밖이다.
- 파싱 성공 뒤 비즈니스 간 참조 검증은 다음 문제의 책임이다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 입력의 반환 타입이 스키마 출력 타입으로 추론되는가?
- [ ] 잘못된 최상위 타입이 즉시 거절되는가?
- [ ] 깊은 배열 원소의 오류 경로를 확인할 수 있는가?
- [ ] 오류 메시지에 논리적 파일명이 포함되는가?
- [ ] `as` 단언이나 이중 타입 선언 없이 구현했는가?
- [ ] 파싱 성공 이후 호출자가 원시 입력을 다시 참조하지 않아도 되는가?

### 구현 후 설명할 것

1. 컴파일 타임 타입과 런타임 신뢰 경계를 분리한 이유
2. 스키마 출력 타입을 도메인 타입의 근원으로 삼은 이유
3. 예외를 즉시 던지는 방식과 결과 객체를 반환하는 방식의 trade-off
4. migration union의 도입 조건과 제거 조건

### 원본 확인 위치

- Thread 02
- 커밋: `feat(content): JSON schema 파싱 경계 추가`
- 연관 커밋: `feat(content): 콘텐츠 파일 schema 파싱 연결`, `refactor(content): schema 기반 핵심 콘텐츠 타입 연결`, `refactor(content): 프로젝트 컬렉션 migration 경계 추가`
- 파일: `src/lib/content-schema.ts`, `src/lib/content-loader.ts`, `src/lib/portfolio/types.ts`
- 함수·타입: `parseContentFile`, `loadPortfolioSource`, `PortfolioSource`
- 관련 Thread: 01, 03, 04

---

<a id="p02"></a>
## [Thread 03 / `feat(content): 콘텐츠 validation 오류 모델 추가`] 교차 파일 참조를 전부 수집하는 정합성 검증기

### 면접 질문

여러 JSON 파일이 ID로 서로를 참조할 때, 첫 번째 오류에서 멈추지 않고 중복 ID와 끊어진 참조를 한 번에 진단하도록 검증기를 어떻게 설계하겠는가? 구조 스키마 검증과 교차 파일 의미 검증은 왜 분리해야 하는가?

꼬리 질문:

- 배열을 매번 `find`하는 구현과 `Map`·`Set`을 먼저 만드는 구현의 복잡도 차이는 무엇인가?
- 같은 잘못된 ID가 여러 위치에서 참조되면 오류를 하나로 합칠 것인가, 각 사용 위치별로 남길 것인가?
- 오류 순서를 결정적으로 유지해야 하는 이유는 무엇인가?
- 검증기가 원본 데이터를 수정하면 어떤 종류의 테스트가 불안정해질 수 있는가?

### 30초 모범 답변

스키마 검증은 각 파일의 모양을 보장하고, 교차 파일 검증은 유일성·외래키·라우트 같은 의미 invariant를 보장하므로 단계가 다릅니다. 먼저 각 컬렉션의 ID를 `Map`과 `Set`으로 색인하고, 모든 규칙을 순회하면서 `{file, path, message}` 이슈를 누적했습니다. 이렇게 하면 한 번의 실행으로 수정할 목록을 전부 보여주고, 참조 수에 대해 거의 선형으로 검사할 수 있습니다. 이슈는 사용 위치별로 남기고 순서를 결정적으로 유지해 CI 출력도 재현 가능하게 합니다.

### 답변 핵심 키워드

구조 검증/의미 검증 분리, unique invariant, foreign-key invariant, `Map`, `Set`, 누적 진단, JSON path, 결정적 순서, O(N+R)

### 백지 구현

**구현 목표**

프로젝트·그룹·기술·이력서 참조로 축소한 데이터 집합에서 중복 ID와 누락 참조를 모두 찾아 반환하는 순수 검증 함수를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type ValidationIssue = {
  file: string;
  path: string;
  message: string;
};

type ValidationSource = {
  groups: Array<{ id: string }>;
  technologies: Array<{ id: string }>;
  projects: Array<{
    id: string;
    groupId: string;
    stackIds: string[];
  }>;
  resumeProjectIds: string[];
};

export function validateReferences(
  source: ValidationSource,
): ValidationIssue[] {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 구조적으로는 이미 유효한 네 컬렉션
- 출력: 발견된 모든 의미 오류. 오류가 없으면 빈 배열

**반드시 만족해야 할 조건**

- 각 컬렉션의 중복 ID를 찾는다.
- 모든 프로젝트의 `groupId`와 `stackIds`를 검증한다.
- 이력서 프로젝트 ID를 검증한다.
- 한 오류를 발견해도 나머지 규칙을 계속 검사한다.
- 각 이슈는 원래 사용 위치를 가리키는 경로를 가진다.
- 입력 배열과 객체를 변경하지 않는다.
- 같은 입력은 같은 이슈 순서를 만든다.

**경계 조건**

- 빈 컬렉션
- 같은 누락 ID를 여러 프로젝트가 참조하는 경우
- 한 프로젝트가 같은 기술 ID를 여러 번 적은 경우
- 중복으로 선언된 ID를 다른 문서가 참조하는 경우
- ID가 빈 문자열인 경우는 구조 스키마가 이미 처리했다고 가정한다.

**실패 조건**

- 함수 자체는 첫 의미 오류에서 예외를 던지지 않는다.
- 구조가 깨진 입력을 복구하는 것은 범위 밖이다.

**필요한 제약**

- 전체 시간 복잡도 목표는 O(N+R)이다. N은 선언 수, R은 참조 수다.
- 정렬을 추가한다면 그 비용과 필요성을 설명한다.
- 구현 시간은 20~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 데이터에서 빈 배열을 반환하는가?
- [ ] 서로 다른 컬렉션의 중복을 모두 보고하는가?
- [ ] 첫 오류 뒤의 누락 참조도 계속 수집하는가?
- [ ] 각 이슈의 파일과 경로가 실제 원인을 찾기에 충분한가?
- [ ] 입력 순서를 바꾸지 않았고 원본을 변형하지 않았는가?
- [ ] 같은 입력을 두 번 실행했을 때 이슈 순서가 같은가?
- [ ] 중첩 `find` 반복 없이 선언 색인을 재사용하는가?
- [ ] 시간·공간 복잡도를 설명할 수 있는가?

### 구현 후 설명할 것

1. fail-fast 대신 이슈 누적을 선택한 사용자 경험상의 이유
2. `Map`·`Set` 색인이 복잡도와 코드 구조에 주는 효과
3. 구조 검증과 의미 검증을 분리한 책임 경계
4. 중복 선언을 참조하는 경우 어떤 오류 모델이 더 유용한지
5. 결정적 오류 순서가 테스트와 CI에 중요한 이유

### 원본 확인 위치

- Thread 03
- 대표 커밋: `feat(content): 콘텐츠 validation 오류 모델 추가`
- 연관 커밋: `feat(content): JSON 경로 진단 추가`, `feat(content): 중복과 참조 진단 helper 추가`, `feat(content): 프로젝트 내부 참조 검증 추가`, `feat(content): 지표와 Resume 참조 검증 추가`, `feat(content): 여정과 인터뷰 참조 검증 추가`, `feat(content): 큐레이션과 연락 참조 검증 추가`
- 파일: `src/lib/content-loader.ts`
- 함수·타입: `PortfolioContentError`, `ContentValidationIssue`, `jsonPath`, `findDuplicates`, `addDuplicateIssues`, `addMissingReferenceIssue`, `loadPortfolioSource`
- 관련 Thread: 02, 04, 06, 13

---

<a id="p03"></a>
## [Thread 03·07 / `feat(content): 내부 route 참조 검증 추가` · `feat(navigation): 템플릿 URL과 쿼리 해석 추가`] URL 구조와 라우트 상태 invariant

### 면접 질문

디자인 선택값과 디버그 쿼리를 현재 경로에 전파하면서 기존 쿼리와 hash를 보존해야 한다. 문자열 이어 붙이기로 구현하면 어떤 버그가 생기며, 내부 링크 검증과 링크 생성은 어떤 공통 URL 의미를 가져야 하는가?

꼬리 질문:

- `https://...`, `mailto:...`, `//cdn...`과 애플리케이션 내부 경로를 어떻게 구분할 것인가?
- 기본 디자인은 URL에서 생략하고 비기본 디자인만 명시하는 정책의 장단점은 무엇인가?
- 동일 쿼리 키가 배열로 들어오면 어떤 값을 선택할 것인가?
- `/projects/%E...`처럼 인코딩된 동적 ID를 검증할 때 무엇을 주의해야 하는가?
- 비활성 페이지와 존재하지 않는 프로젝트를 모두 404로 취급하는 이유는 무엇인가?

### 30초 모범 답변

URL은 경로·쿼리·hash가 분리된 구조이므로 문자열 연결 대신 `URL`과 `URLSearchParams`로 다뤄야 합니다. 외부 링크와 protocol-relative 링크는 건드리지 않고, 내부 링크만 기존 쿼리와 hash를 보존한 채 `view`와 `debug`를 설정하거나 삭제합니다. 기본 디자인을 생략하면 canonical URL 수를 줄일 수 있지만 모든 생성 경로가 같은 규칙을 써야 합니다. 검증 쪽도 같은 pathname 해석을 사용해 비활성 라우트와 존재하지 않는 동적 ID를 차단해야 합니다.

### 답변 핵심 키워드

구조화된 URL, 내부/외부 분류, query 보존, hash 보존, canonical state, default 생략, decode 경계, route enablement, 404

### 백지 구현

**구현 목표**

내부 링크를 분류하고, 현재 라우트 상태를 안전하게 반영하는 두 함수를 작성한다. 문자열 연결은 사용하지 않는다.

**인터페이스 또는 함수 시그니처**

```ts
type ParsedInternalHref = {
  kind: "internal";
  pathname: string;
  search: string;
  hash: string;
};

type ParsedExternalHref = {
  kind: "external";
  href: string;
};

export function classifyHref(
  href: string,
): ParsedInternalHref | ParsedExternalHref {
  // 직접 구현
}

export function applyRouteState(
  href: string,
  state: { view?: string; debug?: boolean },
  options: { defaultView: string },
): string {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 상대 또는 절대 href, 선택적 디자인·디버그 상태, 기본 디자인
- 출력: 외부 링크면 원문, 내부 링크면 상태가 반영된 상대 URL

**반드시 만족해야 할 조건**

- 기존 쿼리 키와 hash를 보존한다.
- 비기본 디자인은 `view`에 기록하고 기본 디자인은 제거한다.
- `debug`가 참이면 기록하고 거짓이면 제거한다.
- 외부 scheme 링크와 `//`로 시작하는 링크는 변경하지 않는다.
- 쿼리 키의 순서는 구현이 정한 한 가지 방식으로 결정적이어야 한다.
- 같은 상태를 두 번 적용해도 결과가 더 변하지 않아야 한다.

**경계 조건**

- `/`, 빈 query, query만 있는 URL, hash만 있는 URL
- 이미 `view`와 `debug`가 있는 URL
- 공백이나 퍼센트 인코딩이 포함된 값
- 배열형 검색 파라미터의 해석은 별도 호출부에서 첫 값을 사용한다고 명시한다.
- 잘못된 URL 문자열은 외부 원문 보존 또는 명시적 오류 중 하나를 선택하고 설명한다.

**필요한 제약**

- 실제 라우트 목록 조회는 범위 밖이다.
- URL 상태 변환은 순수 함수로 작성한다.
- 구현 시간은 20~25분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 내부 경로에서 query와 hash가 보존되는가?
- [ ] 기본 디자인을 적용하면 기존 `view`가 제거되는가?
- [ ] 외부 URL, `mailto:`, protocol-relative URL이 그대로 남는가?
- [ ] 같은 함수를 두 번 적용해도 결과가 동일한가?
- [ ] `?`나 `&`가 중복으로 생기지 않는가?
- [ ] 인코딩된 값을 임의로 이중 인코딩하지 않는가?
- [ ] 상태 삭제와 상태 추가가 모두 테스트됐는가?

### 구현 후 설명할 것

1. 문자열 연결 대신 URL 파서를 선택한 이유
2. 기본 상태를 URL에서 생략하는 canonical 정책
3. 외부 링크를 보존하는 신뢰 경계
4. 링크 생성 규칙과 콘텐츠 라우트 검증 규칙을 일치시키는 방법
5. 잘못된 입력에서 원문 보존과 예외 중 무엇을 선택했는지

### 원본 확인 위치

- Thread 03, 07
- 대표 커밋: `feat(content): 내부 route 참조 검증 추가`
- 연관 커밋: `feat(content): 사이트와 링크 route 참조 검증 추가`, `feat(routes): 비활성 페이지 route 차단`, `feat(navigation): 템플릿 URL과 쿼리 해석 추가`, `feat(home): 쿼리 기반 디자인 전환 연결`
- 파일: `src/lib/content-loader.ts`, `src/lib/portfolio/template-href.ts`, `src/lib/portfolio/selectors.ts`, `src/lib/portfolio/page-context.ts`
- 함수·타입: `addInternalRouteIssue`, `createTemplateHref`, `getTemplateHref`, `resolveHomeTemplateId`, `resolveContentDebug`, `PortfolioPagePath`
- 관련 Thread: 05, 08, 13

---

<a id="p04"></a>
## [Thread 03 / `feat(content): 저장소 자산 참조 경계 검증`] 공개 자산 경로의 디렉터리 탈출 방지

### 면접 질문

콘텐츠가 `/content/...` 또는 `/template/...` 자산 경로를 가리킬 때, 파일 존재 여부만 검사하면 왜 부족한가? 공개 루트 밖으로 빠지는 경로를 어떻게 검증하겠는가?

꼬리 질문:

- `resolvedPath.startsWith(publicRoot)`가 `/public`과 `/publicity`를 구분하지 못하는 이유는 무엇인가?
- 정규화 전 문자열에서 `..`만 찾는 방식이 충분하지 않은 이유는 무엇인가?
- 심볼릭 링크까지 방어하려면 현재의 lexical containment 검사에 무엇을 더해야 하는가?
- 누락 파일과 루트 탈출을 같은 오류로 처리해야 하는가?

### 30초 모범 답변

허용 prefix와 파일 존재만 보면 `..`가 섞인 경로가 공개 루트 밖 파일을 가리킬 수 있습니다. 먼저 허용된 논리 경로인지 확인하고, 루트와 참조를 절대 경로로 정규화한 뒤 `relative(root, candidate)`가 `..`로 시작하거나 절대 경로가 되는지 검사해야 합니다. 그 다음 존재 여부를 확인하고 원본 JSON 경로에 이슈를 붙입니다. 이 방식은 lexical 경계이므로 심볼릭 링크 공격까지 다루려면 `realpath` 기반 검사가 별도로 필요합니다.

### 답변 핵심 키워드

path traversal, normalize, `resolve`, `relative`, sibling-prefix 오류, 허용 prefix, lexical containment, `realpath`, 위치 진단

### 백지 구현

**구현 목표**

공개 디렉터리 아래의 콘텐츠 자산만 허용하고, 경로 탈출과 누락 파일을 구분해 반환하는 함수를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type AssetResolution =
  | { ok: true; absolutePath: string }
  | {
      ok: false;
      reason: "unsupported-prefix" | "outside-root" | "missing";
    };

export function resolvePublicAsset(
  publicRoot: string,
  assetPath: string,
): AssetResolution {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 공개 루트 절대 또는 상대 경로, 콘텐츠에 적힌 URL형 자산 경로
- 출력: 안전한 절대 경로 또는 구체적인 실패 이유

**반드시 만족해야 할 조건**

- `/content/`와 `/template/` 아래만 허용한다.
- 정규화된 후보가 공개 루트 안에 있는지 경로 단위로 검사한다.
- `/publicity` 같은 sibling-prefix를 내부로 오인하지 않는다.
- 파일이 없으면 `missing`, 루트 밖이면 `outside-root`로 구분한다.
- 입력 문자열을 수정하지 않는다.

**경계 조건**

- `..`, 반복 구분자, `.` 세그먼트
- 공개 루트 자체를 가리키는 경로
- 절대 파일 시스템 경로가 입력된 경우
- query나 hash가 자산 경로에 붙은 경우는 거절하거나 제거하는 정책을 명시한다.
- Windows와 POSIX 구분자 차이를 고려한다.

**필요한 제약**

- 기본 문제는 lexical containment까지만 요구한다.
- 심볼릭 링크 해석은 확장 질문으로 남긴다.
- 구현 시간은 15~20분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 정상 `/content/...`와 `/template/...` 파일을 허용하는가?
- [ ] `../` 조합으로 루트 밖에 나가는 경로를 거절하는가?
- [ ] 루트 이름을 prefix로만 공유하는 sibling 디렉터리를 거절하는가?
- [ ] 누락 파일과 경로 탈출을 구분하는가?
- [ ] 플랫폼별 경로 구분자를 사용해도 invariant가 유지되는가?
- [ ] 허용하지 않은 prefix를 파일 존재 여부와 무관하게 거절하는가?
- [ ] 심볼릭 링크를 현재 구현이 보장하는지 여부를 정확히 설명할 수 있는가?

### 구현 후 설명할 것

1. 문자열 prefix 검사 대신 경로 상대화 검사를 선택한 이유
2. 논리 URL 경로와 파일 시스템 경로를 분리한 이유
3. 누락과 보안 경계 위반을 다른 실패로 모델링한 이유
4. lexical containment와 `realpath` containment의 차이

### 원본 확인 위치

- Thread 03
- 커밋: `feat(content): 저장소 자산 참조 경계 검증`
- 파일: `src/lib/content-assets.ts`, `src/lib/portfolio.test.ts`
- 함수: `collectAssetReferences`, `validatePortfolioAssets`
- 관련 Thread: 04, 15

---

<a id="p05"></a>
## [Thread 04·05 / `feat(content): production readiness 기본 검사 추가` · `feat(seo): 콘텐츠 mode별 metadata 정책 추가`] Template·Production 상태 기계와 공개 정책

### 면접 질문

스키마와 참조가 모두 유효한 콘텐츠가 왜 아직 배포 가능한 콘텐츠는 아닐 수 있는가? Template과 Production을 명시적 상태로 두고 build, 공개 URL, 연락처, 자산, metadata, robots, sitemap을 일관되게 통제하는 설계를 설명하라.

꼬리 질문:

- 환경 변수가 없을 때 암묵적으로 production으로 간주하면 어떤 사고가 날 수 있는가?
- placeholder 탐색을 재귀적으로 수행할 때 배열·객체 경로는 어떻게 기록할 것인가?
- localhost, 예약 도메인, URL credentials를 production origin에서 거절하는 이유는 무엇인가?
- template 모드에서 `noindex`만 넣고 robots.txt나 sitemap을 다르게 두면 어떤 불일치가 생기는가?
- readiness 검사를 요청 시점이 아니라 prebuild에 연결한 이유는 무엇인가?

### 30초 모범 답변

구조적으로 유효해도 예시 문구, 로컬 URL, template 자산, 비어 있는 공개 연락처가 남아 있으면 배포 준비가 된 것은 아닙니다. 그래서 `template | production`을 명시적 상태로 두고, production에서만 placeholder·공개 origin·자산 prefix·연락 수단을 모두 검사해 이슈를 누적했습니다. 같은 mode를 metadata, robots, sitemap에도 전달해 template은 검색 차단, production은 검증된 origin만 사용하게 했습니다. 이 검사를 prebuild에 연결해 잘못된 공개 상태가 산출물로 만들어지기 전에 실패시켰습니다.

### 답변 핵심 키워드

명시적 상태, readiness vs validity, fail-closed, 재귀 순회, 공개 origin, reserved host, credentials 거절, 정책 일관성, prebuild gate

### 백지 구현

**구현 목표**

축소된 배포 환경과 콘텐츠를 받아 template 모드에서는 통과시키고, production 모드에서는 모든 준비도 문제를 누적하는 검증기를 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
type ContentMode = "template" | "production";

type ReadinessIssue = {
  file: string;
  path: string;
  message: string;
};

type ReadinessInput = {
  mode: ContentMode;
  siteUrl?: string;
  content: unknown;
  assetPaths: string[];
  contactHrefs: string[];
};

export function validateReadiness(
  input: ReadinessInput,
): ReadinessIssue[] {
  // 직접 구현
}
```

**입력과 출력**

- 입력: 명시적 콘텐츠 mode, 선택적 사이트 URL, 중첩 콘텐츠, 자산 경로, 연락 링크
- 출력: 모든 준비도 이슈. template이면 빈 배열

**반드시 만족해야 할 조건**

- 알 수 없는 mode는 호출 전 또는 별도 resolver에서 거절한다.
- production에서는 중첩 문자열의 placeholder 표식을 모두 찾는다.
- production 사이트 URL은 `http` 또는 `https`, credentials 없음, 로컬·예약 host 아님을 요구한다.
- production 자산은 공개 콘텐츠 경계에 있어야 한다.
- 최소 하나의 사용 가능한 공개 연락 수단을 요구한다.
- 첫 오류에서 중단하지 않고 파일·경로별로 누적한다.
- template 모드와 production 모드의 결과가 명확히 다르다.

**경계 조건**

- 빈 문자열과 공백 문자열
- 배열 안 객체, 객체 안 배열
- 잘못된 URL, 상대 URL, `mailto:`, `tel:`
- `localhost`, loopback IP, `.test`, `.invalid`, 예시 도메인
- URL에 사용자명이나 비밀번호가 포함된 경우
- 연락 링크가 여러 개지만 모두 placeholder인 경우

**필요한 제약**

- 실제 파일 존재 검사는 P04의 함수를 호출한다고 가정할 수 있다.
- metadata 객체 생성은 범위 밖이지만 mode 정책을 설명해야 한다.
- 구현 시간은 25~30분을 목표로 한다.

### 구현 후 자가 검증

- [ ] template 모드는 예시 값이 남아 있어도 공개 허용으로 오인하지 않고 별도 상태로 통과하는가?
- [ ] production에서 여러 파일의 placeholder를 전부 보고하는가?
- [ ] 정상 HTTPS origin과 금지된 local/reserved origin을 구분하는가?
- [ ] credentials가 있는 URL을 거절하는가?
- [ ] 연락 수단 하나가 유효하면 전체 연락 조건을 만족하는가?
- [ ] 자산·URL·placeholder 오류가 동시에 있어도 모두 수집되는가?
- [ ] mode가 metadata·robots·sitemap 정책에 동일하게 전달돼야 함을 설명할 수 있는가?

### 구현 후 설명할 것

1. validity와 publishability를 별도 단계로 나눈 이유
2. 기본값을 template로 두는 fail-closed 정책
3. 재귀 placeholder 탐색의 복잡도와 경로 표현 방식
4. 공개 URL 허용 목록과 거부 목록 접근의 trade-off
5. 런타임 경고보다 build gate를 선택한 이유

### 원본 확인 위치

- Thread 04, 05
- 대표 커밋: `feat(content): production readiness 기본 검사 추가`
- 연관 커밋: `feat(content): 콘텐츠 mode와 readiness 오류 모델 추가`, `feat(content): template placeholder 탐색 경계 추가`, `feat(content): public origin과 자산 경계 검증 추가`, `feat(content): 공개 URL과 연락 링크 검증 추가`, `build(content): readiness 검사를 prebuild에 연결`, `feat(seo): 콘텐츠 mode별 metadata 정책 추가`, `feat(seo): 콘텐츠 mode별 robots 정책 추가`, `feat(seo): 공개 route sitemap 생성`
- 파일: `src/lib/content-readiness.ts`, `scripts/validate-content-readiness.ts`, `src/lib/site-metadata.ts`, `src/app/robots.ts`, `src/app/sitemap.ts`
- 함수·타입: `resolvePortfolioContentMode`, `collectPlaceholderIssues`, `parsePublicSiteUrl`, `resolveProductionSiteUrl`, `validateProductionReadiness`, `validateBuildReadiness`, `createPortfolioMetadata`, `createRobots`, `createSitemap`
- 관련 Thread: 02, 03, 15, 16

---

<a id="p06"></a>
## [Thread 05 / `feat(seo): JSON-LD 안전 직렬화 경계 추가`] `<script>` 문맥의 안전한 JSON 직렬화

### 면접 질문

구조화 데이터를 `<script type="application/ld+json">`에 넣을 때 `JSON.stringify`만 사용하면 왜 충분하지 않을 수 있는가? 직렬화 함수는 어떤 보안 invariant를 보장해야 하는가?

꼬리 질문:

- JSON 파서와 HTML 파서가 같은 문자열을 다르게 해석하는 사례는 무엇인가?
- React의 `dangerouslySetInnerHTML`을 사용하더라도 안전한 경계를 만들 수 있는가?
- 모든 문자열을 일반 HTML entity로 치환하면 JSON 의미가 유지되는가?
- 직렬화 책임을 각 컴포넌트에 흩어 놓으면 어떤 회귀가 생기는가?

### 30초 모범 답변

`JSON.stringify` 결과는 JSON으로는 유효해도 HTML 파서가 `</script>`를 먼저 만나 script 요소를 닫을 수 있습니다. 따라서 객체만 입력받는 중앙 직렬화 함수를 두고, JSON 의미는 유지하면서 HTML script 문맥을 깨는 원문 문자를 남기지 않아야 합니다. 컴포넌트는 이 직렬화 결과만 주입하고 임의 문자열을 직접 넣지 않게 했습니다. 핵심은 "파싱하면 원래 데이터와 같고, HTML 파서에는 새 태그 경계가 생기지 않는다"는 두 invariant입니다.

### 답변 핵심 키워드

JSON 문맥, HTML parser, script breakout, 중앙 serializer, semantic equivalence, `dangerouslySetInnerHTML`, 신뢰 경계

### 백지 구현

**구현 목표**

임의의 JSON 직렬화 가능 값을 받아, JSON으로 다시 파싱했을 때 원본과 같고 HTML script 문맥을 탈출할 수 없는 문자열을 반환한다.

**인터페이스 또는 함수 시그니처**

```ts
export function serializeStructuredData(value: unknown): string {
  // 직접 구현
}
```

**입력과 출력**

- 입력: JSON으로 표현 가능한 객체
- 출력: `<script type="application/ld+json">`의 텍스트로 사용할 문자열
- 실패: 순환 참조 등 JSON 직렬화 자체가 불가능한 값

**반드시 만족해야 할 조건**

- `JSON.parse(result)`가 원본의 JSON 표현과 동등하다.
- 결과 안에 HTML parser가 새 태그 경계로 해석할 수 있는 원문이 남지 않는다.
- 객체가 아닌 사전 직렬화 문자열을 받아 이중 직렬화하는 API로 만들지 않는다.
- 컴포넌트 호출부가 별도 치환을 추가할 필요가 없다.
- 정상 Unicode 문자열을 불필요하게 손실하지 않는다.

**경계 조건**

- `</script><script>`가 포함된 제목
- `<`, `>`, `&`가 연속된 문자열
- 따옴표, 역슬래시, 줄바꿈, 비ASCII 문자
- `null`, 배열, 중첩 객체
- 순환 참조

**필요한 제약**

- CSP nonce 관리와 schema.org 모델 설계는 범위 밖이다.
- 구현 시간은 10~15분을 목표로 한다.

### 구현 후 자가 검증

- [ ] 일반 객체를 직렬화하고 다시 파싱했을 때 값이 같은가?
- [ ] 악의적인 `</script>` 포함 문자열이 script 문맥을 닫지 못하는가?
- [ ] 따옴표와 역슬래시를 포함한 값이 손상되지 않는가?
- [ ] 배열과 `null`을 처리하는가?
- [ ] 순환 참조 실패를 숨기지 않는가?
- [ ] 호출부가 원시 객체만 넘기도록 API가 제한되는가?
- [ ] 보안 invariant와 데이터 동등성 invariant를 각각 테스트했는가?

### 구현 후 설명할 것

1. JSON 유효성과 HTML 문맥 안전성이 별개인 이유
2. 중앙 직렬화 경계를 둔 이유
3. 일반 HTML escaping과 JSON-safe escaping의 차이
4. 순환 참조나 직렬화 불가 값에서 실패를 전파하는 방식

### 원본 확인 위치

- Thread 05
- 커밋: `feat(seo): JSON-LD 안전 직렬화 경계 추가`
- 연관 커밋: `feat(seo): 사이트 소유자 JSON-LD 모델 추가`, `feat(seo): 프로젝트 CreativeWork JSON-LD 모델 추가`, `feat(seo): 프로젝트 상세에 JSON-LD 연결`, `test(seo): JSON-LD 계약과 직렬화 검증`
- 파일: `src/lib/site-metadata.ts`, `src/components/portfolio/structured-data.tsx`
- 함수·컴포넌트: `serializeStructuredData`, `createSiteStructuredData`, `createProjectStructuredData`, `StructuredData`
- 관련 Thread: 04, 13
