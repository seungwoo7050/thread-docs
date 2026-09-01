# 프런트엔드 캐시·준비 상태·관측·장애 주입 면접 워크북

이 문서는 비동기 캐시의 정합성, 배포 준비 상태, 저카디널리티 메트릭, 오류 이벤트 경계, 안전한 장애 실험을 다룬다.

---

## IM-22. [Thread 20 / `refactor(web): query key와 retry 정책 정의`, `refactor(web): session query와 cache invalidation 추가`] 서버 상태 캐시의 정확한 무효화와 세션 만료

### 면접 질문

React 화면마다 `fetch`와 로컬 state를 직접 두지 않고 query key, 공통 retry 정책, mutation별 무효화 목록을 분리한 이유는 무엇인가요? 로그아웃이나 401에서 세션 캐시를 지울 때 진행 중 요청과 다른 사용자의 데이터까지 고려해야 하는 이유도 설명해 주세요.

꼬리 질문:

- 문자열 prefix로 캐시를 지우는 방식과 exact key 무효화는 어떤 차이가 있습니까?
- 모든 오류를 자동 재시도하면 401, 403, 검증 오류에서 어떤 문제가 생깁니까?
- mutation 성공 직후 낙관적 갱신과 서버 재조회 중 무엇을 선택하겠습니까?
- 사용자 A 로그아웃 직후 사용자 B가 로그인할 때 이전 응답이 늦게 도착하면 어떻게 막습니까?

### 30초 모범 답변

서버 상태는 소유권, stale 시간, 재시도, 동시 요청 합치기, mutation 후 갱신 규칙이 필요하므로 페이지별 로컬 state보다 공통 query 계층이 적합합니다. query key를 도메인 구조로 만들고 mutation이 영향을 주는 정확한 key만 무효화하면 과도한 재요청과 누락을 줄일 수 있습니다. 401·403·입력 오류는 재시도하지 않고, 세션 만료 시 진행 중 세션 요청을 취소하거나 세대가 다른 응답을 무시한 뒤 사용자 범위 캐시를 제거해야 계정 간 데이터가 섞이지 않습니다.

### 답변 핵심 키워드

server state · structured query key · exact invalidation · retry classification · session generation · request cancellation · stale response · user-scoped cache

### 백지 구현

**구현 목표**

도메인별 query key와 mutation 영향 관계를 정의하고, 오류 종류에 따라 재시도 여부를 판단하며, 세션 만료 시 사용자 범위 캐시를 정리하는 작은 캐시 조정기를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
type QueryKey = readonly unknown[];

type ApiFailure = {
  status: number;
  code?: string;
};

interface QueryCache {
  cancel(key: QueryKey): Promise<void>;
  invalidate(key: QueryKey, options?: { exact?: boolean }): Promise<void>;
  remove(key: QueryKey, options?: { exact?: boolean }): void;
}

export const queryKeys = {
  session(): QueryKey {
    // 직접 구현
    return [];
  },
  lobby(): QueryKey {
    // 직접 구현
    return [];
  },
  dashboard(userId: string): QueryKey {
    // 직접 구현
    return [];
  },
  tournament(id: string): QueryKey {
    // 직접 구현
    return [];
  }
};

export function shouldRetryQuery(error: unknown, failureCount: number): boolean {
  // 직접 구현
  throw new Error("not implemented");
}

export async function expireSession(
  cache: QueryCache,
  userScopedKeys: QueryKey[]
): Promise<void> {
  // 직접 구현
}
```

**입력과 출력**

- 입력: API 오류, 실패 횟수, 사용자 범위 query key
- 출력: 재시도 여부, 캐시 취소·제거 부수 효과

**반드시 만족해야 할 조건**

- query key는 서로 다른 리소스와 사용자를 충돌 없이 구분한다.
- 401, 403, 404 또는 명백한 요청 오류는 자동 재시도하지 않는다.
- 일시적 서버·네트워크 오류만 제한된 횟수로 재시도한다.
- 세션 만료 시 관련 in-flight 요청을 먼저 처리해 늦은 응답이 제거된 캐시를 다시 채우지 않게 한다.
- 사용자 범위 데이터는 새 세션으로 넘어가지 않는다.
- 무관한 공개 데이터 캐시는 불필요하게 삭제하지 않는다.

**경계 조건**

- 오류 객체가 예상 구조가 아님
- 실패 횟수 0과 최대 횟수 경계
- 같은 key가 정리 목록에 중복 포함
- cancel 중 일부 요청이 이미 완료
- 로그아웃과 로그인 요청이 매우 가깝게 실행

**실패 조건**

- cache cancel 또는 invalidate 실패
- 사용자 ID가 없는 dashboard key 생성
- 모든 key를 broad prefix로 삭제해 성능이 악화되는 경우

**필요한 제약**

- 특정 UI 컴포넌트에 결합하지 않는다.
- mutation별 영향 key는 선언적으로 확인할 수 있어야 한다.
- 사용자 계정 전환 테스트가 가능해야 한다.

### 구현 후 자가 검증

- [ ] 서로 다른 사용자의 key가 같지 않다.
- [ ] 인증·권한·검증 오류가 재시도되지 않는다.
- [ ] 일시 오류의 최대 재시도 횟수가 제한된다.
- [ ] 세션 만료 뒤 늦은 이전 요청이 사용자 캐시를 복원하지 못한다.
- [ ] 무관한 공개 query는 유지된다.
- [ ] mutation별 invalidate 목록에 누락과 과잉이 없는지 시나리오로 검증했다.
- [ ] exact와 prefix invalidation의 차이를 설명할 수 있다.

### 구현 후 설명할 것

1. 서버 상태와 로컬 UI 상태를 분리한 이유
2. 구조화된 query key가 API 경계를 반영하는 방식
3. 오류 유형별 retry 정책을 둔 이유
4. 세션 만료에서 cancel·remove 순서와 stale response 문제
5. 낙관적 갱신과 invalidate 후 재조회 선택 기준

### 원본 확인 위치

- Thread 20
- 커밋: `refactor(web): query key와 retry 정책 정의`
- 커밋: `refactor(web): session query와 cache invalidation 추가`
- 커밋: `refactor(web): React Query provider 연결`
- `apps/web/src/lib/query.ts`
  - `queryKeys`
  - `mutationInvalidations`
  - `invalidateExactQueries`
  - `shouldRetryQuery`
  - `expireSession`
- `apps/web/src/components/QueryProvider.tsx`
- 관련 Thread: 02, 19

---

## IM-23. [Thread 21 / `feat(db): migration set 상태 검사 추가`, `feat(ops): liveness와 readiness endpoint 추가`] liveness, readiness, migration 집합 비교

### 면접 질문

프로세스가 살아 있다는 것과 트래픽을 받을 준비가 됐다는 것을 왜 별도 endpoint로 나눴나요? 적용된 migration이 단순히 최신 개수인지가 아니라 `current`, `pending`, `diverged`인지 집합으로 비교한 이유도 설명해 주세요.

꼬리 질문:

- DB가 연결되지만 migration이 pending이면 readiness는 어떤 상태여야 합니까?
- 적용 DB에 코드가 모르는 migration이 있으면 왜 `diverged`로 봐야 합니까?
- migration 테이블 자체가 아직 없을 때는 오류와 pending 중 무엇으로 해석합니까?
- readiness가 느린 DB 호출을 매 요청마다 수행할 때의 비용과 캐싱 trade-off는 무엇입니까?

### 30초 모범 답변

liveness는 프로세스 재시작이 필요한지를, readiness는 현재 새 트래픽을 안전하게 받을 수 있는지를 판단합니다. DB가 살아 있어도 코드가 기대하는 migration이 누락되거나 예상 밖 migration이 적용돼 있으면 쿼리 계약이 깨질 수 있으므로 ready가 아닙니다. 예상 이름과 적용 이름을 비교해 누락만 있으면 pending, 예상 밖 항목이 있으면 diverged로 구분합니다. readiness 실패는 503과 제한된 구조로 반환하고 내부 DB 오류 내용은 로그에만 남깁니다.

### 답변 핵심 키워드

liveness/readiness 분리 · dependency health · migration drift · current/pending/diverged · HTTP 503 · fail closed · 배포 순서 · 제한된 오류 노출

### 백지 구현

**구현 목표**

코드에 포함된 migration 이름과 데이터베이스에 적용된 이름을 비교해 상태와 차이를 반환하는 순수 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface MigrationSetComparison {
  status: "current" | "pending" | "diverged";
  missing: string[];
  unexpected: string[];
}

export function compareMigrationSets(
  expectedNames: string[],
  appliedNames: string[]
): MigrationSetComparison {
  // 직접 구현
  throw new Error("not implemented");
}

export function readinessStatus(input: {
  lifecycle: "accepting" | "draining";
  database: "up" | "down";
  migrations: "current" | "pending" | "diverged" | "not_applicable" | "unknown";
}): { statusCode: 200 | 503; status: "ready" | "not_ready" } {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 기대 migration 목록, 적용 migration 목록, lifecycle·DB 상태
- 출력: 차이 목록, readiness HTTP 상태

**반드시 만족해야 할 조건**

- 순서 차이만으로 diverged가 되지 않는다.
- 기대 항목이 모두 적용되고 예상 밖 항목이 없으면 current다.
- 누락만 있으면 pending이다.
- 예상 밖 항목이 하나라도 있으면 diverged다.
- 중복 이름이 들어오면 명시적으로 거부하거나 일관된 정책을 적용한다.
- draining, DB down, pending, diverged, unknown은 ready가 아니다.
- 메모리 저장소의 `not_applicable`은 DB up·accepting이면 ready로 볼 수 있다.

**경계 조건**

- 두 목록 모두 비어 있음
- 기대 목록만 비어 있음
- 적용 목록만 비어 있음
- 같은 항목이 여러 번 들어옴
- 이름의 대소문자 차이

**실패 조건**

- 비문자열 또는 빈 migration 이름
- migration 조회 자체 실패
- 상태 값이 계약 밖임

**필요한 제약**

- 원래 입력 배열을 변경하지 않는다.
- 반환 배열 순서는 입력 또는 정렬 중 하나로 안정적으로 정한다.
- 비교 자체는 O(n) 또는 O(n log n) 범위로 구현한다.

### 구현 후 자가 검증

- [ ] current, pending, diverged 세 상태를 각각 검증했다.
- [ ] 순서가 달라도 결과가 같다.
- [ ] missing과 unexpected가 정확한 목록이다.
- [ ] draining과 DB down이 503을 반환한다.
- [ ] migration 조회 실패가 ready로 오판되지 않는다.
- [ ] 상태 응답에 내부 SQL 오류가 노출되지 않는다.
- [ ] 배포 시 migration job과 API readiness의 순서를 설명할 수 있다.

### 구현 후 설명할 것

1. liveness와 readiness를 합치면 생기는 재시작 루프 위험
2. migration 개수 대신 이름 집합을 비교한 이유
3. pending과 diverged를 구분해 운영자가 얻는 정보
4. readiness의 일시 캐시와 최신성 trade-off
5. drain 중 lifecycle 상태를 readiness에 반영하는 방법

### 원본 확인 위치

- Thread 21
- 커밋: `feat(db): migration set 상태 검사 추가`
- 커밋: `feat(db): repository readiness 경계 추가`
- 커밋: `feat(ops): liveness와 readiness endpoint 추가`
- `packages/db/src/migrator.ts`
  - `compareMigrationSets`
  - `inspectMigrationSet`
- `packages/db/src/index.ts`
  - `checkReadiness`
- `apps/api/src/app.ts`
  - `/health/live`
  - `/health/ready`
- 관련 Thread: 04, 22, 23

---

## IM-24. [Thread 21 / `feat(metrics): repository operation 측정 추가`, `fix(db): idle connection pool 오류에서 복구`] 비동기 계측 wrapper와 실패해도 안전한 오류 이벤트 경계

### 면접 질문

Repository의 모든 메서드를 Proxy로 감싸 지연 시간과 성공·실패를 측정할 때 동기 예외와 Promise rejection을 모두 처리해야 하는 이유는 무엇인가요? 메서드 이름 label을 허용 목록으로 제한하고 PostgreSQL pool 오류의 원본 메시지를 버린 이유도 설명해 주세요.

꼬리 질문:

- label에 user ID, room ID, 예외 메시지를 넣으면 왜 Prometheus에 문제가 됩니까?
- wrapper에서 `this` binding을 잃으면 어떤 버그가 생깁니까?
- EventEmitter의 `error` listener가 없거나 listener가 다시 throw하면 어떤 문제가 생깁니까?
- 관측 코드의 실패가 비즈니스 요청이나 프로세스를 실패시키지 않게 하려면 어떤 원칙이 필요합니까?

### 30초 모범 답변

계측 경계는 원본 메서드의 동작을 바꾸지 않아야 하므로 동기 throw와 비동기 rejection을 각각 실패로 기록한 뒤 같은 오류를 다시 전달하고, 메서드는 원래 target을 `this`로 호출해야 합니다. 메트릭 label은 가능한 값 집합을 제한해 시계열 폭증을 막고 알 수 없는 메서드는 `other`로 모읍니다. pool의 idle client 오류는 반드시 listener에서 소비하되, 연결 문자열이나 원본 메시지 대신 허용된 오류 이름과 코드만 내보냅니다. reporter 자체가 실패해도 EventEmitter 경계를 벗어나 프로세스를 죽이지 않게 합니다.

### 답변 핵심 키워드

transparent instrumentation · sync throw · Promise rejection · Reflect.apply · bounded cardinality · safe labels · EventEmitter error · secret redaction · best-effort observer

### 백지 구현

**구현 목표**

임의 서비스 객체의 함수 호출을 투명하게 계측하고, 외부 오류 객체에서 허용된 짧은 label만 추출하는 두 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface MetricsSink {
  observe(input: {
    operation: string;
    outcome: "success" | "failure";
    durationMs: number;
  }): void;
}

export function instrumentService<T extends object>(
  service: T,
  metrics: MetricsSink,
  allowedOperations: ReadonlySet<string>,
  now?: () => number
): T {
  // 직접 구현
  throw new Error("not implemented");
}

export interface SafeErrorEvent {
  kind: "idle_client_error";
  errorName: string;
  errorCode: string | null;
}

export function toSafeErrorEvent(error: unknown): SafeErrorEvent {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 메서드가 있는 객체, 계측 sink, 허용 operation 집합, 알 수 없는 오류
- 출력: 원본과 같은 API의 wrapper, 제한된 오류 이벤트

**반드시 만족해야 할 조건**

- 함수가 아닌 property는 원래 값으로 반환한다.
- 원본 메서드의 `this`가 유지된다.
- 동기 반환, Promise 성공, 동기 throw, Promise rejection을 각각 한 번만 기록한다.
- 원본 반환값과 오류 identity를 가능한 한 보존한다.
- 허용되지 않은 operation label은 고정값 `other`가 된다.
- duration은 음수가 되지 않는다.
- 오류 이름과 코드는 정해진 문자·길이 규칙을 통과할 때만 사용한다.
- 원본 메시지, 연결 문자열, 임의 property는 안전 이벤트에 포함하지 않는다.
- metrics 또는 reporter 실패를 원본 작업에 전파할지 정책을 명시한다.

**경계 조건**

- getter property
- symbol key
- thenable이지만 실제 Promise가 아닌 반환값
- 메서드가 동기적으로 Promise 생성 전 throw
- error가 문자열, null, proxy 객체인 경우
- 매우 긴 error code

**실패 조건**

- metrics sink가 throw
- `now`가 뒤로 감
- 원본 객체의 getter가 throw

**필요한 제약**

- label 값은 사전에 제한한다.
- wrapper가 비동기 메서드를 불필요하게 다른 타입으로 바꾸지 않게 한다.
- 오류 이벤트는 직렬화해도 비밀 정보가 나오지 않아야 한다.

### 구현 후 자가 검증

- [ ] 동기·비동기 성공과 실패가 각각 정확히 한 번 기록된다.
- [ ] 클래스 메서드가 wrapper에서도 올바른 `this`를 사용한다.
- [ ] unknown operation이 `other`로 집계된다.
- [ ] metrics sink 실패 정책이 일관된다.
- [ ] 악의적 오류 객체에서도 변환 함수가 예외를 밖으로 내보내지 않는다.
- [ ] 안전 이벤트 직렬화에 메시지·비밀번호·연결 URL이 없다.
- [ ] label cardinality 상한을 설명할 수 있다.

### 구현 후 설명할 것

1. Proxy 계측의 장점과 정적 decorator 방식의 trade-off
2. 동기 throw와 Promise rejection을 따로 처리하는 이유
3. `Reflect.apply`로 target binding을 유지한 이유
4. 메트릭 label cardinality가 메모리·성능에 미치는 영향
5. 오류 관측 코드를 fail-safe로 만드는 보안·가용성 원칙

### 원본 확인 위치

- Thread 21
- 커밋: `feat(metrics): repository operation 측정 추가`
- 커밋: `fix(db): idle connection pool 오류에서 복구`
- 커밋: `test(db): 안전한 connection pool 오류 처리 검증`
- `apps/api/src/observability.ts`
  - `ApiMetrics`
  - `instrumentRepository`
- `packages/db/src/poolError.ts`
  - `PostgresPoolErrorEvent`
  - `PostgresPoolErrorReporter`
  - `installPostgresPoolErrorHandler`
- `packages/db/src/poolError.test.ts`
- 관련 Thread: 02, 22, 23

---

## IM-25. [Thread 23 / `test(load): fault recovery 검사 자동화`, `test(load): fault scenario 설정과 report 검증`] 안전한 장애 주입 시나리오와 복구 판정

### 면접 질문

DB 지연·중단과 edge 연결 reset을 자동화하는 fault scenario가 왜 loopback URL만 허용하도록 제한됐나요? baseline, 장애 상태, 복구 상태를 readiness로 관측하고 마지막에 반드시 reset을 시도한 설계를 설명해 주세요.

꼬리 질문:

- 장애 주입 도구의 대상 URL을 환경 변수로 완전히 열어 두면 어떤 위험이 있습니까?
- 한 번의 probe 성공만으로 복구됐다고 판단할 때 생길 수 있는 오탐은 무엇입니까?
- 시나리오 본체와 cleanup이 모두 실패하면 어떤 오류를 반환하거나 보고서에 남겨야 합니까?
- sleep, clock, probe, 명령 실행기를 의존성으로 주입한 이유는 무엇입니까?

### 30초 모범 답변

장애 주입은 정상 테스트보다 피해 반경이 크므로 설정 오류가 외부·공유 환경을 건드리지 않게 loopback HTTP 대상만 허용하는 안전장치를 둡니다. 시나리오는 먼저 baseline readiness를 확인하고, 각 장애를 적용한 뒤 기대 상태를 deadline까지 polling하며, 복구 명령 뒤 ready 상태가 돌아오는지 검증합니다. 성공·실패와 관계없이 마지막 reset을 시도하고, 본래 실패와 cleanup 실패를 함께 보존합니다. 실행기와 시간 의존성을 주입하면 실제 proxy 없이 순서·timeout·보고서를 결정적으로 테스트할 수 있습니다.

### 답변 핵심 키워드

fault injection · blast radius · loopback allowlist · baseline · bounded polling · readiness oracle · finally cleanup · dependency injection · structured report

### 백지 구현

**구현 목표**

안전한 로컬 대상만 허용하는 설정 파서와, `baseline → 장애 → 복구 → reset` 단계를 실행해 구조화된 보고서를 만드는 축소된 fault scenario를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
interface FaultConfig {
  controlUrl: string;
  readinessUrl: string;
  requestTimeoutMs: number;
  recoveryTimeoutMs: number;
  pollIntervalMs: number;
}

interface Observation {
  status: number | null;
  durationMs: number;
  body?: unknown;
  error?: string;
}

interface FaultDependencies {
  apply(command: "reset" | "dependency-down" | "dependency-up"): Promise<void>;
  probe(url: string, timeoutMs: number): Promise<Observation>;
  sleep(ms: number): Promise<void>;
  now(): string;
}

export function createFaultConfig(
  env: Record<string, string | undefined>
): FaultConfig {
  // 직접 구현
  throw new Error("not implemented");
}

export async function runFaultScenario(
  config: FaultConfig,
  dependencies: FaultDependencies
): Promise<{
  schemaVersion: 1;
  startedAt: string;
  finishedAt: string;
  passed: boolean;
  steps: Array<{ name: string; passed: boolean; observation: Observation }>;
}> {
  // 직접 구현
  throw new Error("not implemented");
}
```

**입력과 출력**

- 입력: 환경 설정과 명령·probe·시간 의존성
- 출력: 순서가 있는 단계별 JSON 보고서

**반드시 만족해야 할 조건**

- 제어 URL과 readiness URL은 HTTP loopback host만 허용한다.
- timeout과 polling 값은 양의 safe integer여야 한다.
- poll interval은 recovery timeout보다 클 수 없다.
- baseline이 ready가 아니면 장애를 적용하지 않는다.
- 장애 적용 뒤 기대하는 not-ready 상태를 deadline 안에서 확인한다.
- 복구 뒤 ready 상태를 deadline 안에서 확인한다.
- 어느 단계가 실패해도 마지막 `reset`을 시도한다.
- 보고서의 시각과 단계 순서는 주입된 의존성만 사용한다.
- 오류 메시지에 민감한 환경 변수 전체를 노출하지 않는다.

**경계 조건**

- IPv4·IPv6 loopback, `localhost`
- HTTPS 또는 외부 IP
- poll interval과 timeout이 같은 값
- 첫 probe에서 바로 성공
- 마지막 deadline probe에서 성공
- reset 자체가 실패

**실패 조건**

- 잘못된 URL 또는 숫자 설정
- command 실패
- probe timeout 또는 잘못된 응답 body
- 기대 상태가 deadline까지 나타나지 않음
- cleanup 실패

**필요한 제약**

- 실제 장애 도구나 네트워크 호출을 단위 테스트에서 실행하지 않는다.
- 각 단계는 이름과 관측값을 구조화해 기록한다.
- 무한 polling을 금지한다.

### 구현 후 자가 검증

- [ ] 외부 host와 HTTPS 대상이 거부된다.
- [ ] 잘못된 숫자와 poll/timeout 관계가 거부된다.
- [ ] baseline 실패 시 장애 명령이 실행되지 않는다.
- [ ] 성공 시 명령과 단계 순서가 정확하다.
- [ ] timeout 경계에서 polling이 무한 반복되지 않는다.
- [ ] 중간 실패와 reset 실패가 모두 관측 가능하다.
- [ ] fake probe·clock·sleep으로 빠르고 결정적인 테스트가 가능하다.

### 구현 후 설명할 것

1. 장애 테스트에 일반 테스트보다 강한 대상 제한이 필요한 이유
2. readiness를 복구 oracle로 사용한 이유와 한계
3. polling deadline과 개별 request timeout의 차이
4. cleanup 실패를 숨기지 않는 오류·보고서 설계
5. 실제 부하·장애 실험 전에 단위 계약 테스트를 두는 이유

### 원본 확인 위치

- Thread 23
- 커밋: `test(load): fault recovery 검사 자동화`
- 커밋: `test(load): fault scenario 설정과 report 검증`
- `tests/load/fault-scenario.mjs`
  - `createFaultScenarioConfig`
  - `runFaultScenario`
  - `formatFaultReport`
- `tests/load/fault-scenario.test.mjs`
- `tests/load/toxiproxy-control.mjs`
- 관련 Thread: 21, 22
