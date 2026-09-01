# 운영 상태와 관측 가능성 면접 워크북

Thread 14(E24)는 운영 endpoint와 로그를 단순한 부가 기능이 아니라 시스템 경계로 다룬다. 프로세스 생존, 트래픽 수용 가능성, 의존성의 불확실성, 로그·metric label의 cardinality와 비밀 노출을 서로 다른 계약으로 구분하는지가 핵심이다.

## 문서 내 면접 포인트

- [P27 liveness·readiness·metric unknown을 구분하는 운영 계약](#p27)
- [P28 allowlist 기반 구조화 로그와 bounded cardinality](#p28)

---

<a id="p27"></a>
## [Thread 14 (E24) / 제목 미노출 — 기록상 readiness·metrics와 운영 endpoint 구현] liveness·readiness·metric unknown을 구분하는 운영 계약

### 면접 질문

- PostgreSQL이 중단됐을 때 liveness는 200을 유지하고 readiness는 503을 반환하도록 나눈 이유는 무엇입니까?
  - 꼬리 질문: readiness가 단순 설정 확인이 아니라 실제 `SELECT 1`을 수행해야 하는 이유는 무엇입니까?
    - 모범답변: 설정이 존재해도 DNS·접속·인증·DB 실행이 실패할 수 있습니다. 실제 query 성공은 이 인스턴스가 현재 권위 저장소와 최소한 통신할 수 있다는 더 직접적인 준비 증거입니다.
  - 꼬리 질문: DB를 읽을 수 없을 때 queue age를 `0`으로 내보내지 않고 metric 자체를 생략하는 이유를 설명해 보세요.
    - 모범답변: 0은 "확인 결과 대기열이 비었거나 age가 0"이라는 관찰값이고 DB 장애는 unknown입니다. 둘을 합치면 장애 중에도 queue가 건강하다는 잘못된 신호를 만듭니다.
  - 꼬리 질문: 현재 기록에서 `Operations.ready`의 별도 probe timeout은 확인되지 않습니다. 이 상태의 위험과 어디에 상한을 둘지 설명해 보세요.
    - 모범답변: 연결이 끊기지 않은 채 지연되면 health handler와 connection이 오래 묶이고 shutdown도 늦어질 수 있습니다. PostgreSQL statement/query timeout이나 취소 가능한 request deadline을 readiness query 경계에 둬야 합니다.
  - 꼬리 질문: health endpoint에 `Cache-Control: no-store`가 필요한 이유는 무엇입니까?
    - 모범답변: health는 순간 상태라 proxy나 browser cache의 과거 200·503이 재사용되면 traffic 제어 판단이 틀립니다. 각 probe가 현재 인스턴스를 직접 관찰하게 해야 합니다.

### 30초 모범 답변

확인된 구현은 liveness가 DB를 호출하지 않고, readiness가 실제 `SELECT 1` 성공 여부로 200과 503을 나누며, DB를 읽지 못한 queue age를 생략합니다. 이 구분은 DB 장애를 process restart가 아니라 traffic removal로 다루고 unknown을 0으로 왜곡하지 않는다는 장점이 있습니다. 다만 `Operations.ready` 자체의 별도 query timeout은 기록에서 확인되지 않습니다. 운영 계약으로는 statement/query timeout이나 취소 가능한 deadline을 보강해 probe가 handler와 shutdown을 무기한 붙잡지 않게 해야 합니다.

### 답변 핵심 키워드

- liveness와 readiness 분리
- 현재 구현: `SELECT 1` dependency probe
- probe timeout 보강
- traffic removal
- restart loop 방지
- unknown ≠ zero
- `no-store`
- safe degradation

### 백지 구현

#### 구현 목표

프로세스 생존, DB 준비 상태, metric snapshot을 서로 독립적으로 계산하는 운영 상태 모듈을 작성한다.
#### 인터페이스 또는 함수 시그니처

- `live(): HealthResponse`
- `ready(database, timeoutMs): Promise<ReadyResponse>`
- `collectMetrics(snapshot, dependencyState): MetricSample[]`
#### 입력과 출력

- `database`: 짧은 probe query를 실행할 수 있는 어댑터
- `timeoutMs`: readiness 확인의 최대 시간
- `snapshot`: HTTP·worker의 메모리 내 누적 계수
- `dependencyState`: DB 준비 여부와, 준비된 경우에만 읽은 queue 상태
- 출력: liveness 응답, readiness 응답, 노출 가능한 metric sample 목록
#### 반드시 만족해야 할 조건

- liveness는 DB 호출 없이 프로세스 자체 상태만 보고 판단한다.
- readiness는 실제 DB probe를 수행한다. 이 백지 구현에서는 현재 기록의 보강 과제로 명시적 상한도 둔다.
- DB 실패·timeout은 readiness 실패로 변환하되 내부 오류·접속 문자열을 노출하지 않는다.
- 모든 health·metrics 응답은 공유 cache에 저장되지 않도록 표시한다.
- DB 상태를 나타내는 metric은 명시적으로 0/1을 제공한다.
- DB를 읽지 못한 queue age·queue depth는 0으로 조작하지 않고 생략하거나 unknown으로 유지한다.
- 한 probe가 영구히 끝나지 않아 요청 handler와 shutdown을 붙잡지 않는다.
#### 경계 조건

- 프로세스는 정상이나 DB가 연결 거부하는 경우
- 연결은 되지만 query가 timeout을 넘기는 경우
- readiness probe 직후 DB가 다시 내려가는 경우
- queue가 실제로 비어 있어 age가 정의되지 않는 경우
- DB 장애 중 metrics endpoint 자체는 응답 가능한 경우
#### 실패 조건

- liveness와 readiness가 같은 함수·같은 의존성 결과를 그대로 공유한다.
- DB 장애를 process restart 신호로만 표현한다.
- 불확실한 queue 값을 0으로 내보낸다.
- readiness 오류에 SQL, hostname, credential 또는 stack을 포함한다.
- probe timeout 뒤에도 미해결 작업이 계속 자원을 점유한다.
#### 필요한 제약

- 면접 범위에서는 단일 PostgreSQL 의존성만 다룬다.
- readiness 결과는 순간 관찰일 뿐 이후 요청의 성공을 보장하지 않는다는 한계를 설명한다.
- metric text serialization보다 상태 의미와 sample 선택을 우선 구현한다.

```ts
type DependencyState =
  | { ready: true; queueAgeSeconds: number | null }
  | { ready: false };

export async function ready(
  database: ReadinessDatabase,
  timeoutMs: number,
): Promise<ReadyResponse> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    await database.query('SELECT 1', { signal: controller.signal });
    return { status: 'ok', cacheControl: 'no-store' };
  } catch {
    // SQL·host·credential·stack을 공개 응답에 포함하지 않는다.
    return { status: 'unavailable', cacheControl: 'no-store' };
  } finally {
    clearTimeout(timer);
  }
}

export function collectMetrics(
  snapshot: OperationsSnapshot,
  dependency: DependencyState,
): MetricSample[] {
  const samples: MetricSample[] = [
    { name: 'postgres_ready', value: dependency.ready ? 1 : 0,
      labels: { role: snapshot.role } },
  ];
  for (const sample of snapshot.http) {
    const labels = { role: snapshot.role, route: sample.route,
      method: sample.method, status: String(sample.status) };
    samples.push(
      { name: 'http_request_duration_seconds_sum', value: sample.seconds, labels },
      { name: 'http_request_duration_seconds_count', value: sample.count, labels },
      { name: 'http_errors_total', value: sample.errors, labels },
    );
  }
  // dependency unknown과 실제 queue age 0을 구분한다.
  if (dependency.ready && dependency.queueAgeSeconds !== null) {
    samples.push({ name: 'check_queue_age_seconds',
      value: Math.max(0, dependency.queueAgeSeconds), labels: { role: snapshot.role } });
  }
  if (snapshot.role === 'worker') {
    samples.push(
      { name: 'worker_active', value: snapshot.active, labels: { role: 'worker' } },
      { name: 'worker_claims_total', value: snapshot.claims, labels: { role: 'worker' } },
      { name: 'worker_recovery_runs_total', value: snapshot.recoveryRuns, labels: { role: 'worker' } },
      { name: 'worker_recovered_checks_total', value: snapshot.recovered, labels: { role: 'worker' } },
    );
  }
  return samples;
}
```

### 구현 후 자가 검증

- DB 정상 시 liveness와 readiness가 모두 성공한다.
- DB 연결 거부·query 오류·timeout에서 liveness는 유지되고 readiness만 실패한다.
- readiness가 지정한 상한 안에 항상 종료된다.
- 실패 응답과 metric에 DB URL, SQL 오류, stack이 없다.
- DB 장애 시 준비 상태 metric은 실패를 나타내고 queue age sample은 존재하지 않는다.
- 실제 빈 queue와 DB unknown이 서로 다른 출력으로 구분된다.
- 모든 endpoint가 `no-store` 정책을 유지한다.
- 반복 probe가 timer·connection·promise를 누적하지 않는다.

### 구현 후 설명할 것

- liveness와 readiness를 분리한 장애 대응 효과
  - 모범답변: DB 장애 때 liveness를 실패시키면 정상 프로세스를 반복 재시작해 장애를 키울 수 있습니다. readiness만 실패시키면 인스턴스를 traffic에서 빼면서 진단 endpoint는 유지합니다.
- 실제 probe의 정확성과 의존성 부하 사이 trade-off
  - 모범답변: 실제 query는 end-to-end 준비 상태를 잘 반영하지만 probe 빈도만큼 pool과 DB에 부하를 줍니다. 짧고 읽기 전용인 query, 제한된 빈도와 deadline이 필요합니다.
- unknown을 0으로 바꾸지 않은 이유
  - 모범답변: 0은 측정된 정상 값이고 unknown은 측정 자체가 실패한 상태입니다. alert와 dashboard가 둘을 구분해야 DB 장애를 queue 정상으로 오판하지 않습니다.
- readiness를 시작 검증, 요청 처리 오류, metric과 어떻게 구분했는지
  - 모범답변: 시작 schema 검증은 호환성 gate, readiness는 현재 dependency 통신의 순간 관찰, 요청 오류는 개별 작업 결과, metric은 누적·관찰 상태입니다. 서로 다른 수명과 조치가 필요합니다.
- 여러 의존성이 생길 때 준비 상태를 합성할 기준
  - 모범답변: 해당 인스턴스가 traffic을 안전하게 처리하는 데 필수인 의존성만 AND로 묶고 선택 기능은 degraded 상태로 분리해야 합니다. 단순히 모든 외부 시스템 장애를 readiness 실패로 만들면 blast radius가 커집니다.

### 원본 확인 위치

- Thread 14(E24)
- `server/operations.ts` — `Operations.ready`, `Operations.metrics`, `workerHealth`
- `server/app.ts` — API `/health/live`, `/health/ready`, `/metrics` 연결
- `server/worker.ts` — worker health endpoint와 종료 lifecycle
- `server/main.ts` — API process 시작·종료 lifecycle
- 확인된 현재 구현: `Operations.ready`는 `SELECT 1`을 사용하며 별도 probe timeout은 기록에서 확인되지 않음
- 관련 Thread: 03(E03) DB 시작 검증, 11(E11) worker lifecycle

---

<a id="p28"></a>
## [Thread 14 (E24) / 제목 미노출 — 기록상 구조화 로그와 metric 경계 구현] allowlist 기반 구조화 로그와 bounded cardinality

### 면접 질문

- request 로그에 실제 URL 대신 route template을 사용한 이유는 무엇입니까?
  - 꼬리 질문: 모든 request나 error 객체를 JSON 직렬화하지 않고 허용 필드만 새 객체에 복사한 이유는 무엇입니까?
    - 모범답변: request/error 객체에는 cookie·authorization·body·URL·stack처럼 새 필드가 계속 생길 수 있어 denylist가 누락되기 쉽습니다. allowlist projection은 출력 가능한 필드 집합 자체를 고정합니다.
  - 꼬리 질문: `processId`, role, requestId는 어떤 문제를 해결하며 서로 대체할 수 있습니까?
    - 모범답변: role은 API/worker 종류, processId는 한 프로세스 수명의 ready·stopping 사건, requestId는 한 HTTP 요청의 사건을 연결합니다. 범위가 달라 서로 대체할 수 없습니다.
  - 꼬리 질문: metric label에 monitor ID, check ID, URL 또는 오류 문자열을 넣으면 어떤 운영 문제가 생깁니까?
    - 모범답변: 사용자·요청 수에 비례해 시계열이 무한히 늘어 메모리·저장·query 비용이 폭증하고 민감 값도 노출될 수 있습니다. label은 route template 같은 유한 차원만 사용합니다.

### 30초 모범 답변

운영 로그는 검색 가능해야 하지만 입력 전체를 신뢰하면 안 됩니다. 실제 URL이나 resource ID를 label로 쓰면 요청 수에 비례해 시계열과 색인 cardinality가 늘고, query·header·body·오류 객체를 그대로 기록하면 credential과 사용자 입력이 새어 나갈 수 있습니다. 그래서 event별 allowlist로 time, role, processId, requestId, route template, method, status, duration 같은 안전한 필드만 새 객체에 담습니다. processId는 한 프로세스 수명을, requestId는 한 요청을 연결하며, route template은 관측 가능성을 유지하면서 cardinality를 제한합니다.

### 답변 핵심 키워드

- structured event
- field allowlist
- data minimization
- route template
- bounded cardinality
- request correlation
- process lifecycle correlation
- secret suppression
- stable dimensions

### 백지 구현

#### 구현 목표

신뢰하지 않는 request·error 입력에서 운영상 필요한 값만 선택하고, 로그와 metric label을 유한한 집합으로 제한하는 변환기를 작성한다.
#### 인터페이스 또는 함수 시그니처

- `operationalEvent(input): SafeOperationalEvent`
- `metricDimensions(input): SafeDimensions`
#### 입력과 출력

- 입력: event 이름, process role, process ID, 선택적 request ID, route template, method, status, duration와 신뢰하지 않는 원본 request/error
- 출력: 허용된 필드만 가진 JSON event와 bounded label 집합
#### 반드시 만족해야 할 조건

- 출력 key는 event별 명시적 allowlist를 따른다.
- 원본 request, response, error 객체를 spread하거나 그대로 직렬화하지 않는다.
- route는 등록된 template만 허용하며 실제 path parameter·query를 넣지 않는다.
- method, role, event, status category는 유한한 허용 집합으로 정규화한다.
- header, cookie, authorization, CSRF token, idempotency key, body, URL, stack, DB 오류 전문은 출력하지 않는다.
- duration과 status 같은 수치는 유효 범위를 검사한다.
- requestId와 processId의 용도를 분리하고, 누락 가능한 필드는 누락으로 표현한다.
- 알 수 없는 route·method는 원문 대신 고정된 안전 범주로 접는다.
#### 경계 조건

- 등록되지 않은 route 또는 404 요청
- path에 UUID나 비밀 문자열이 포함된 요청
- error message 안에 URL·credential이 들어 있는 경우
- duration이 음수·NaN·무한대인 경우
- requestId가 없거나 형식이 잘못된 경우
- 같은 route template에 서로 다른 수천 개의 resource ID가 들어오는 경우
#### 실패 조건

- `JSON.stringify(request)` 또는 `{ ...error }`로 직렬화한다.
- 실제 pathname이나 error message를 metric label로 사용한다.
- 알 수 없는 값마다 새 label을 만든다.
- 로그 필터링을 출력 sink 한 곳에만 의존한다.
- 안전성을 위해 모든 상관관계 필드까지 제거해 장애 추적이 불가능해진다.
#### 필요한 제약

- redaction 정규식보다 allowlist projection을 기본 전략으로 사용한다.
- 입력 값을 변경하지 않는 순수 변환기로 만든다.
- 로그 transport·보관 기간·sampling은 별도 설계 문제로 둔다.

```ts
type OperationalEventInput = {
  event: unknown;
  role: unknown;
  processId: unknown;
  requestId?: unknown;
  route?: unknown;
  method?: unknown;
  status?: unknown;
  durationMs?: unknown;
  request?: unknown;
  error?: unknown;
};

export function operationalEvent(
  input: OperationalEventInput,
): SafeOperationalEvent {
  const events = new Set([
    'http_request', 'api_ready', 'api_stopping', 'api_shutdown_deadline',
    'worker_ready', 'worker_stopping', 'worker_shutdown_deadline',
    'check_claimed', 'check_completed', 'checks_recovered',
    'worker_iteration_failed', 'database_connection_lost',
  ]);
  const routes = new Set([
    '/health', '/health/live', '/health/ready', '/metrics', '/auth/login',
    '/auth/session', '/auth/csrf', '/auth/logout', '/monitors',
    '/monitors/:id', '/monitors/:id/checks', '/checks/:id',
  ]);
  const methods = new Set(['GET', 'HEAD', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS']);
  const event = typeof input.event === 'string' && events.has(input.event)
    ? input.event : 'unknown_event';
  const role = input.role === 'api' || input.role === 'worker' ? input.role : 'unknown';
  const processId = typeof input.processId === 'string' &&
    /^[0-9a-f-]{36}$/i.test(input.processId) ? input.processId : 'unknown';
  const output: Record<string, unknown> = {
    time: new Date().toISOString(), role, processId, event,
  };
  if (event === 'http_request') {
    output.route = typeof input.route === 'string' && routes.has(input.route)
      ? input.route : 'unmatched';
    output.method = typeof input.method === 'string' && methods.has(input.method)
      ? input.method : 'OTHER';
    output.status = Number.isInteger(input.status) &&
      Number(input.status) >= 100 && Number(input.status) <= 599
      ? input.status : 500;
    output.durationMs = typeof input.durationMs === 'number' &&
      Number.isFinite(input.durationMs) ? Math.max(0, input.durationMs) : 0;
    if (typeof input.requestId === 'string' && /^[A-Za-z0-9_-]{1,128}$/.test(input.requestId)) {
      output.requestId = input.requestId;
    }
  }
  // input.request와 input.error는 의도적으로 어떤 참조도 복사하지 않는다.
  return output as SafeOperationalEvent;
}
```

### 구현 후 자가 검증

- 정상 request event에 필요한 상관관계·status·duration만 남는다.
- URL, query, UUID, cookie, authorization, CSRF, idempotency key, body, stack이 어떤 입력 경로에서도 출력되지 않는다.
- 서로 다른 resource ID 수천 개가 동일한 route template label로 합쳐진다.
- unknown route·method가 고정된 범주로 정규화되어 cardinality가 늘지 않는다.
- 잘못된 숫자와 긴 문자열이 안전하게 거부·생략된다.
- processId로 시작·ready·stopping event를 연결하고 requestId로 한 요청 event를 연결할 수 있다.
- 출력 객체가 원본 request·error에 대한 참조를 보유하지 않는다.
- 로그 필드와 metric label이 각각 허용된 차원만 사용한다.

### 구현 후 설명할 것

- denylist redaction보다 allowlist projection을 선택한 이유
  - 모범답변: 객체에 미래에 추가될 민감 필드를 denylist가 알 수 없지만 allowlist는 새 객체에 승인된 값만 복사합니다. 기본값이 비노출인 구조입니다.
- route template이 개인정보 보호와 cardinality를 동시에 개선하는 방식
  - 모범답변: `/monitors/:id`는 실제 UUID와 query를 숨기면서 모든 동일 handler 요청을 하나의 안정된 차원으로 묶습니다. 검색 가능성은 유지하고 시계열 폭증은 막습니다.
- requestId와 processId를 나눈 이유
  - 모범답변: requestId는 한 요청의 짧은 흐름을, processId는 재시작 사이를 구분하는 프로세스 lifecycle을 연결합니다. 하나로 합치면 요청 상관관계나 재시작 경계 중 하나를 잃습니다.
- 진단 정보와 비밀 최소화 사이의 기준
  - 모범답변: 조치 가능한 route·method·status·duration·안전한 ID는 남기고 사용자가 제공한 값과 credential·오류 전문은 제외합니다. 상세 원인은 내부 bounded code나 별도 접근 통제된 추적 수단으로 다룹니다.
- 고유 resource 차원의 분석이 필요할 때 metric이 아닌 다른 관측 수단을 선택하는 방법
  - 모범답변: high-cardinality ID는 metric label 대신 접근 통제·보존 기간·sampling을 둔 구조화 trace나 감사 로그에서 조회합니다. 집계 metric의 유한 시계열 계약을 깨지 않습니다.

### 원본 확인 위치

- Thread 14(E24)
- `server/operations.ts` — `Operations`, `PROCESS_ID`, `recordHttp`, 구조화 event와 metric 차원
- `server/app.ts` — HTTP request 관측 hook과 `/metrics` 연결
- `server/worker.ts` — claim·complete·recovery·iteration lifecycle event
- `server/main.ts` — process ready·stopping event
- 관련 Thread: 04·05의 session·CSRF 비밀 경계, 11의 worker lifecycle
