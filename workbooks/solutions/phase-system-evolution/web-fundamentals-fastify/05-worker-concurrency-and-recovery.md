# 워커 동시성, 멱등성과 crash recovery 면접 워크북

Thread 09·10·11은 하나의 연속된 실행 상태 머신이다. API는 durable intent만 만들고, scheduler는 중복 없는 slot을 만들며, worker는 원자적으로 소유권을 얻고, idempotency와 lease가 ack loss·동시 claim·crash 뒤의 의미를 고정한다.

## 문서 내 면접 포인트

- [P21 API와 endpoint I/O를 분리한 durable CheckRun 상태 머신](#p21)
- [P22 시간 slot과 unique key로 만드는 scheduler 멱등성](#p22)
- [P23 `FOR UPDATE SKIP LOCKED` 기반 원자적 claim과 실행 소유권](#p23)
- [P24 commit 후 응답 유실을 견디는 요청 idempotency와 client intent 수명](#p24)
- [P25 lease 만료 복구, stale worker 차단과 terminal write 원자성](#p25)
- [P26 SIGTERM에서 신규 claim 중단, 현재 작업 drain과 강제 종료](#p26)

---

<a id="p21"></a>
## [Thread 09 (E09) / 제목 미노출 — 기록상 persisted queue와 별도 worker 구현] API와 endpoint I/O를 분리한 durable CheckRun 상태 머신

### 면접 질문

- 수동 Check API가 즉시 외부 HTTP를 실행하는 대신 `QUEUED` row를 저장하고 202를 반환하도록 바꾼 이유는 무엇입니까?
  - 꼬리 질문: `QUEUED → RUNNING → SUCCEEDED/FAILED/ABORTED` 상태에서 각 필드의 nullability invariant를 설명해 보세요.
    - 모범답변: `QUEUED`는 시작·종료·결과가 모두 null, `RUNNING`은 `startedAt`과 lease만 있고 결과는 null입니다. 성공·실패는 종료 시각과 일관된 관찰 필드를 가지며, `ABORTED`는 종료 시각만 있고 endpoint 결과는 null입니다.
  - 꼬리 질문: worker가 내려가도 브라우저가 새 terminal 결과를 꾸며내지 않고 persisted 상태를 다시 읽게 한 이유는 무엇입니까?
    - 모범답변: polling 실패는 endpoint 실패의 증거가 아닙니다. PostgreSQL 상태가 권위 원천이므로 브라우저는 durable ID를 다시 조회하고, lease recovery가 불확실한 RUNNING을 ABORTED로 확정할 때까지 사실을 추측하지 않습니다.

### 30초 모범 답변

외부 HTTP를 API 요청 안에서 실행하면 응답 지연, 프로세스 종료, 중복 제출이 DB 상태와 섞입니다. 그래서 API는 인증·권한을 확인한 뒤 실행 intent인 `QUEUED` row만 원자적으로 저장하고 202를 반환하며, 별도 worker가 실행을 소유합니다. `QUEUED`는 시작·종료·결과가 없고, `RUNNING`은 시작만 있으며, terminal만 종료 시각과 결과 의미를 가집니다. 이 상태를 PostgreSQL 제약과 코드 전이로 함께 고정하면 재시작 뒤에도 사실을 복원할 수 있고, 브라우저는 polling 실패 시 결과를 추측하지 않습니다.

### 답변 핵심 키워드

- durable queue
- API/worker separation
- 202 Accepted
- state machine
- persisted intent
- field invariants
- no fabricated terminal
- restart recovery

### 백지 구현

#### 구현 목표

CheckRun 상태와 이벤트를 받아 허용된 다음 상태만 만드는 순수 상태 전이 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`transitionCheck(run, event, now): CheckRun`
#### 입력과 출력

- 입력: 현재 CheckRun, claim·succeed·fail·abort 이벤트, 현재 시각
- 출력: invariant를 만족하는 새 CheckRun
- 실패: 허용되지 않은 전이나 필드 조합에 대한 상태 오류
#### 반드시 만족해야 할 조건

- `QUEUED`는 startedAt, finishedAt, 결과 필드가 모두 `null`이다.
- `RUNNING`은 startedAt만 존재하고 terminal 결과는 없다.
- `SUCCEEDED`는 2xx status와 latency, finishedAt을 가진다.
- `FAILED`는 HTTP 실패 또는 transport 실패의 일관된 필드 조합을 가진다.
- `ABORTED`는 실행 중단 사실을 나타내며 성공·실패로 다시 열리지 않는다.
- terminal 상태는 불변이며 같은 이벤트 재적용이 상태를 바꾸지 않는다.
#### 경계 조건

- QUEUED에서 바로 terminal로 가려는 이벤트
- RUNNING에 두 번 claim
- 2xx인데 FAILED
- transport 실패인데 httpStatus가 존재
- finishedAt이 startedAt보다 이른 값
#### 실패 조건

- 상태별 nullability를 optional 필드 관습에만 맡긴다.
- terminal row를 재시도하려고 다시 RUNNING으로 바꾼다.
- API가 queue 저장 전에 endpoint I/O를 시작한다.
- polling 오류를 terminal 실패로 저장한다.
#### 필요한 제약

- 함수는 순수하며 DB claim·lease ownership은 후속 문제로 분리한다.
- 현재 모델의 terminal 상태와 실패 이유만 사용한다.

```ts
export function transitionCheck(
  run: CheckRun,
  event: CheckEvent,
  now: string,
): CheckRun {
  if (['SUCCEEDED', 'FAILED', 'ABORTED'].includes(run.state)) return run;
  if (run.state === 'QUEUED') {
    if (event.type !== 'claim') throw new Error('A queued check must be claimed first');
    return {
      ...run, state: 'RUNNING', startedAt: now, finishedAt: null,
      httpStatus: null, latencyMs: null, failureReason: null,
    };
  }
  if (run.state !== 'RUNNING' || run.startedAt === null ||
      Date.parse(now) < Date.parse(run.startedAt)) {
    throw new Error('Invalid running check');
  }
  if (event.type === 'claim') throw new Error('A running check cannot be claimed again');
  if (event.type === 'abort') {
    return { ...run, state: 'ABORTED', finishedAt: now,
      httpStatus: null, latencyMs: null, failureReason: null };
  }
  if (event.latencyMs < 0) throw new Error('Latency must be non-negative');
  if (event.type === 'succeed') {
    if (event.httpStatus < 200 || event.httpStatus >= 300) {
      throw new Error('A successful check requires a 2xx status');
    }
    return { ...run, state: 'SUCCEEDED', finishedAt: now,
      httpStatus: event.httpStatus, latencyMs: event.latencyMs, failureReason: null };
  }
  if (event.failureReason === 'HTTP_STATUS') {
    if (event.httpStatus === null ||
        (event.httpStatus >= 200 && event.httpStatus < 300)) {
      throw new Error('HTTP failure requires a non-2xx status');
    }
  } else if (event.httpStatus !== null) {
    throw new Error('Transport failure cannot have an HTTP status');
  }
  return { ...run, state: 'FAILED', finishedAt: now,
    httpStatus: event.httpStatus, latencyMs: event.latencyMs,
    failureReason: event.failureReason };
}
```

### 구현 후 자가 검증

- 각 상태의 정상 전이가 정확한 필드를 채운다.
- 모든 금지 전이가 명시적으로 거부된다.
- terminal에 이벤트를 다시 적용해도 row가 바뀌지 않는다.
- HTTP 실패와 transport 실패의 `httpStatus` 조합이 다르다.
- 시각 순서와 latency 음수 경계를 거부한다.
- API handler 테스트에서 queue INSERT 전후 endpoint 호출 수가 0이다.

### 구현 후 설명할 것

- API 요청과 외부 실행을 분리한 이유
  - 모범답변: API는 권한 확인과 durable intent 저장까지만 짧게 수행해 endpoint 지연·장애를 request transaction에서 떼어냅니다. worker가 commit된 intent를 별도로 claim해 실행 수명을 소유합니다.
- 상태 enum만이 아니라 필드 조합 invariant가 필요한 이유
  - 모범답변: 같은 `FAILED`라도 HTTP 응답을 본 경우와 transport 실패는 `httpStatus` 의미가 다르고, QUEUED/RUNNING의 시각도 다릅니다. enum만 검사하면 모순된 사실을 저장할 수 있습니다.
- DB CHECK constraint와 TypeScript union을 둘 다 유지하는 이유
  - 모범답변: TypeScript는 애플리케이션 작성 시 도움을 주지만 다른 writer와 migration, runtime 값은 막지 못합니다. DB constraint는 모든 write의 최종 경계이고 union은 코드의 상태별 사용을 명확하게 합니다.
- 202 응답이 실행 성공이 아니라 접수 성공을 뜻한다는 점
  - 모범답변: 202는 QUEUED row가 commit됐고 조회할 ID가 생겼다는 acknowledgement입니다. 실제 HTTP 결과는 worker가 이후 terminal 상태로 저장해야 알 수 있습니다.

### 원본 확인 위치

- Thread 09
- `server/model.ts` — CheckRun 상태와 `TerminalCheckRun`
- `server/app.ts` — 수동 Check 202와 QUEUED INSERT
- `server/worker.ts` — worker 실행
- `server/migrations/006_check_queue.sql`
- `test/execution.test.ts`
- 관련 Thread: 10(E10) claim ownership, 11(E11) lease recovery
---

<a id="p22"></a>
## [Thread 09 (E09) / 제목 미노출 — 기록상 interval scheduler 구현] 시간 slot과 unique key로 만드는 scheduler 멱등성

### 면접 질문

- 스케줄러가 단순히 `now - lastRun >= interval`을 계산하지 않고 Monitor 생성 시각에 정렬된 slot을 만든 이유는 무엇입니까?
  - 꼬리 질문: `(monitor_id, scheduled_for)` unique 제약과 `ON CONFLICT DO NOTHING`이 어떤 중복을 막습니까?
    - 모범답변: 반복 tick이나 여러 scheduler가 같은 Monitor·slot을 동시에 INSERT하는 중복을 DB에서 원자적으로 막습니다. manual Check는 `scheduled_for`가 null이라 이 자연 key와 별도입니다.
  - 꼬리 질문: 두 scheduler가 같은 시각에 실행되거나 한 scheduler가 같은 tick을 반복해도 정확히 한 intent만 남는다는 것을 어떻게 검증합니까?
    - 모범답변: 두 실제 connection을 동시에 실행하고 같은 고정 `now`로 반복 호출한 뒤 `(monitor_id, scheduled_for)`별 row 수가 1인지 확인합니다. connector 호출은 0이어야 합니다.

### 30초 모범 답변

마지막 실행 시각을 기준으로 하면 worker 지연이나 실패가 다음 실행 시각을 계속 밀어 drift가 생깁니다. 생성 시각과 interval에 정렬된 결정적 slot을 계산하면 같은 `now`와 Monitor는 같은 `scheduled_for`를 얻습니다. 이를 `(monitor_id, scheduled_for)` unique key로 만들고 conflict를 무시하면 여러 scheduler와 반복 tick이 동시에 같은 slot을 제안해도 한 row만 남습니다. scheduler는 due intent만 저장하고 실제 endpoint 실행은 claim worker가 담당합니다.

### 답변 핵심 키워드

- deterministic time slot
- drift avoidance
- database clock
- unique natural key
- ON CONFLICT
- idempotent enqueue
- multiple schedulers
- no endpoint I/O

### 백지 구현

#### 구현 목표

Monitor의 생성 시각·interval·현재 시각으로 가장 최근 due slot을 계산하고, 이미 존재하는 slot은 중복 생성하지 않는 scheduler를 작성한다.
#### 인터페이스 또는 함수 시그니처

`scheduledFor(monitor, now): string | null`, `enqueueDueChecks(db, now): Promise<number>`
#### 입력과 출력

- 입력: enabled Monitor, createdAt, interval seconds, 현재 시각
- 출력: due하지 않으면 `null`, due하면 canonical slot 시각, enqueue된 새 row 수
#### 반드시 만족해야 할 조건

- createdAt 이전에는 slot을 만들지 않는다.
- slot은 createdAt 기준 interval 경계에 정렬된다.
- 같은 Monitor와 now는 항상 같은 slot을 계산한다.
- 같은 `(monitorId, slot)`은 여러 호출·프로세스에서도 하나만 저장된다.
- disabled Monitor는 enqueue하지 않는다.
- scheduler는 endpoint I/O를 수행하지 않는다.
#### 경계 조건

- now가 createdAt과 정확히 같다.
- interval 경계 1ms 전·정확히 경계·1ms 후
- 긴 scheduler 중단 뒤 여러 slot이 지나간 경우
- 두 scheduler transaction의 동시 INSERT
- Monitor가 scheduler 조회 직후 disabled 또는 삭제되는 경우
#### 실패 조건

- 애플리케이션 메모리의 lastRun만으로 중복을 막는다.
- 매 tick마다 random ID만 생성하고 자연 key가 없다.
- worker 실행 완료 시각을 다음 schedule anchor로 사용해 drift를 만든다.
- scheduler가 Check 실행까지 맡는다.
#### 필요한 제약

- 현재 문제는 가장 최근 due slot 하나를 enqueue하는 정책으로 축소한다.
- 프로덕션 시각 권위는 DB clock으로 두고, 테스트에는 명시적 now를 주입한다.

```ts
export function scheduledFor(
  monitor: Pick<Monitor, 'createdAt' | 'interval' | 'enabled'>,
  now: string,
): string | null {
  if (!monitor.enabled) return null;
  const created = Date.parse(monitor.createdAt);
  const current = Date.parse(now);
  if (!Number.isFinite(created) || !Number.isFinite(current) || current < created) return null;
  const intervalMs = monitor.interval * 1000;
  const slot = created + Math.floor((current - created) / intervalMs) * intervalMs;
  return new Date(slot).toISOString();
}

export async function enqueueDueChecks(
  db: SchedulerDatabase,
  now: string,
): Promise<number> {
  const result = await db.query(
    `INSERT INTO check_runs
       (id, monitor_id, trigger, state, queued_at, scheduled_for)
     SELECT gen_random_uuid(), id, 'SCHEDULED', 'QUEUED', $1,
       date_bin(interval_seconds * interval '1 second', $1::timestamptz, created_at)
       FROM monitors
      WHERE enabled AND created_at <= $1
     ON CONFLICT (monitor_id, scheduled_for) DO NOTHING`,
    [now],
  );
  return result.rowCount ?? 0;
}
```

### 구현 후 자가 검증

- createdAt 이전과 disabled Monitor는 enqueue 0이다.
- interval 경계 전·정확히·후의 slot이 기대대로다.
- 같은 tick을 여러 번 호출해 row가 하나만 남는다.
- 두 scheduler를 동시에 시작해도 같은 slot row가 하나다.
- 한 Monitor의 slot이 다른 Monitor를 막지 않는다.
- enqueue 전후 endpoint connector 호출 수가 0이다.

### 구현 후 설명할 것

- last completion 기반 schedule과 고정 slot 기반 schedule의 차이
  - 모범답변: 완료 시각을 anchor로 쓰면 실행 지연만큼 다음 시각이 계속 밀립니다. 생성 시각 기반 slot은 지연과 무관하게 동일한 interval 격자를 유지합니다.
- 자연 unique key가 application lock보다 강한 이유
  - 모범답변: DB unique 제약은 여러 프로세스와 재시작을 가로질러 동시 commit을 직렬화합니다. 프로세스 메모리 lock은 다른 인스턴스나 crash 뒤 상태를 알지 못합니다.
- 놓친 여러 slot을 catch up할지 최신 하나만 만들지의 정책 trade-off
  - 모범답변: 원본은 현재 가장 최근 slot 하나만 계산해 장기 중단 뒤 폭주를 피합니다. 모든 slot catch-up은 관찰 완전성은 높지만 queue burst와 오래된 Check의 가치 문제를 다뤄야 합니다.
- 애플리케이션 시계와 DB 시계가 어긋날 때의 문제
  - 모범답변: 여러 scheduler의 wall clock이 다르면 서로 다른 slot을 만들거나 이른 실행을 만들 수 있습니다. production은 DB clock을 권위로 삼고 테스트만 고정 시각을 명시적으로 주입하는 편이 안전합니다.

### 원본 확인 위치

- Thread 09
- `server/worker.ts` — `scheduleDueChecks`
- `server/migrations/006_check_queue.sql`
- `test/execution.test.ts`
- 관련 Thread: 10(E10) 다중 worker claim
---

<a id="p23"></a>
## [Thread 10 (E10) / `feat: persist manual identities and atomic check ownership`] `FOR UPDATE SKIP LOCKED` 기반 원자적 claim과 실행 소유권

### 면접 질문

- worker가 `SELECT next row`와 `UPDATE RUNNING`을 별도 autocommit으로 수행하면 어떤 duplicate execution race가 생깁니까?
  - 꼬리 질문: `FOR UPDATE SKIP LOCKED`를 사용한 claim transaction의 동작과 장단점을 설명해 보세요.
    - 모범답변: 한 worker가 후보 row lock을 잡으면 다른 worker는 기다리지 않고 다음 unlocked row를 찾으므로 병렬 처리량이 좋습니다. 반면 엄격한 FIFO는 깨질 수 있고 오래 잠긴 row가 반복해서 건너뛰어질 수 있습니다.
  - 꼬리 질문: 완료 UPDATE에 `worker_id`와 현재 상태 조건을 다시 넣은 이유는 무엇입니까?
    - 모범답변: claim winner만 자신의 RUNNING row를 terminal로 바꾸게 하는 compare-and-set입니다. 다른 worker나 이미 완료·복구된 row의 늦은 결과는 row count 0으로 거부됩니다.

### 30초 모범 답변

두 worker가 같은 QUEUED row를 SELECT한 뒤 각각 실행하면 endpoint가 중복 호출됩니다. claim은 한 transaction에서 후보 row를 잠그고, 이미 잠긴 row는 건너뛰며, 같은 row를 RUNNING·worker owner로 갱신한 결과를 반환해야 합니다. `SKIP LOCKED`는 대기 대신 다른 작업을 진행해 throughput을 높이지만 엄격한 FIFO나 starvation 방지는 보장하지 않습니다. completion도 `id`, RUNNING, worker_id 조건을 만족한 owner만 허용해 늦은 worker나 다른 worker가 terminal 결과를 덮지 못하게 합니다.

### 답변 핵심 키워드

- atomic claim
- row lock
- FOR UPDATE SKIP LOCKED
- single transaction
- worker ownership
- compare-and-set completion
- duplicate execution
- fairness trade-off

### 백지 구현

#### 구현 목표

동시에 여러 worker가 호출해도 한 QUEUED row를 한 worker만 RUNNING으로 소유하고, owner만 완료할 수 있는 repository 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`claimNext(db, workerId, now): Promise<ClaimedRun | null>`, `completeOwned(db, workerId, runId, result): Promise<boolean>`
#### 입력과 출력

- 입력: worker ID, 시각, terminal 관찰 결과
- 출력: 소유권이 기록된 claim row 또는 없음, 완료 적용 여부
- DB row: 상태와 worker_id가 함께 전이
#### 반드시 만족해야 할 조건

- 후보 선택과 RUNNING update가 같은 transaction·connection에 있다.
- 잠긴 후보는 대기하지 않고 건너뛴다.
- 한 row는 한 worker ID로만 claim된다.
- completion은 같은 worker ID와 RUNNING 상태를 조건으로 한다.
- loser worker의 completion은 row를 바꾸지 않고 false를 반환한다.
- 모든 외부 I/O는 성공한 claim 이후에만 시작한다.
#### 경계 조건

- queue가 비어 있음
- 첫 후보가 다른 transaction에 잠겨 있음
- 두 worker가 같은 순간 claim
- owner worker가 completion을 두 번 호출
- 다른 worker가 같은 run ID로 완료 시도
#### 실패 조건

- SELECT와 UPDATE 사이에 transaction 경계가 있다.
- worker_id 없이 상태만 RUNNING으로 바꾼다.
- completion이 현재 owner와 상태를 확인하지 않는다.
- lock을 잡은 채 endpoint HTTP를 실행한다.
#### 필요한 제약

- claim transaction은 짧게 끝내고 네트워크 I/O는 commit 이후 수행한다.
- 엄격한 공정성은 범위 밖이며 ordering 정책과 `SKIP LOCKED`의 한계를 설명한다.

```ts
export async function claimNext(
  db: WorkerDatabase,
  workerId: string,
  now: string,
): Promise<ClaimedRun | null> {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    const result = await client.query<ClaimedRun>(
      `UPDATE check_runs
          SET state = 'RUNNING', started_at = $2, worker_id = $1
        WHERE id = (
          SELECT id FROM check_runs
           WHERE state = 'QUEUED'
           ORDER BY queued_at, id
           LIMIT 1 FOR UPDATE SKIP LOCKED
        )
          AND state = 'QUEUED'
      RETURNING *`,
      [workerId, now],
    );
    await client.query('COMMIT');
    return result.rows[0] ?? null;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

export async function completeOwned(
  db: WorkerDatabase,
  workerId: string,
  runId: string,
  result: ObservedResult,
): Promise<boolean> {
  const saved = await db.query(
    `UPDATE check_runs
        SET state = $3, http_status = $4, latency_ms = $5,
            failure_reason = $6, finished_at = $7
      WHERE id = $1 AND worker_id = $2 AND state = 'RUNNING'`,
    [runId, workerId, result.state, result.httpStatus,
      result.latencyMs, result.failureReason, result.finishedAt],
  );
  return (saved.rowCount ?? 0) === 1;
}
```

### 구현 후 자가 검증

- 두 실제 connection에서 동시에 claim해 같은 row owner가 하나뿐이다.
- loser는 같은 row를 실행하지 않고 다른 unlocked row 또는 null을 받는다.
- endpoint 호출 횟수가 claim된 row 수와 같다.
- 다른 worker completion과 중복 completion이 row를 바꾸지 않는다.
- claim lock은 HTTP 실행 동안 유지되지 않는다.
- rollback 뒤 row가 다시 QUEUED로 남아 다른 worker가 claim할 수 있다.

### 구현 후 설명할 것

- claim을 compare-and-set 성격의 transaction으로 본 이유
  - 모범답변: "현재 QUEUED인 다음 row"라는 조건을 잠근 상태에서 RUNNING·owner로 함께 바꿔야 한 worker만 성공합니다. 읽기와 쓰기가 분리되면 두 worker가 같은 이전 상태를 볼 수 있습니다.
- `SKIP LOCKED`의 처리량 이점과 fairness 단점
  - 모범답변: 잠금 대기 없이 다른 row를 처리해 worker가 놀지 않지만 queue 순서를 엄격히 지키거나 starvation이 없다고 보장하지는 않습니다.
- DB lock 범위와 외부 I/O 범위를 분리한 이유
  - 모범답변: claim transaction을 commit한 뒤 HTTP를 실행해야 row lock과 DB connection을 느린 네트워크 동안 점유하지 않습니다. durable worker_id가 lock 이후의 소유권을 이어받습니다.
- worker owner 조건이 lease 도입 전후에 어떤 역할을 하는지
  - 모범답변: worker_id는 어느 claimant의 completion인지 막고, lease는 그 소유권의 시간 유효성을 추가합니다. lease 이후에는 owner가 맞아도 만료된 completion은 거부됩니다.

### 원본 확인 위치

- Thread 10
- `server/worker.ts` — `runNextCheck`, `completeCheck`
- `server/migrations/007_check_ownership.sql`
- `test/ownership.test.ts`
- 관련 Thread: 09(E09) queue, 11(E11) lease 만료
---

<a id="p24"></a>
## [Thread 10 (E10) / `feat: persist manual identities and atomic check ownership`] commit 후 응답 유실을 견디는 요청 idempotency와 client intent 수명

### 면접 질문

- 브라우저 중복 방지 gate가 있어도 `Idempotency-Key`가 필요한 이유는 무엇입니까?
  - 꼬리 질문: 같은 사용자·같은 key가 같은 Monitor에 재전송되면 기존 CheckRun을 반환하고, 다른 Monitor에 쓰이면 409를 반환한 이유는 무엇입니까?
    - 모범답변: 같은 Monitor면 같은 실행 의도의 replay라 기존 ID가 정답입니다. 다른 Monitor면 한 key가 두 의미를 갖게 되어 어느 결과를 replay할지 모호하므로 semantic conflict로 거부합니다.
  - 꼬리 질문: 클라이언트가 응답을 받기 전과 받은 후에 key를 언제 재사용·폐기해야 합니까?
    - 모범답변: acknowledgement 전에는 응답 유실 가능성이 있으므로 같은 사용자 의도에 같은 key를 유지합니다. 성공 응답을 받은 뒤 다음 클릭은 새 의도이므로 기존 key를 폐기하고 새 UUID를 발급합니다.

### 30초 모범 답변

브라우저 gate는 한 탭의 동시 submit만 막고, 서버 commit 뒤 응답이 유실돼 사용자가 재시도하거나 다른 인스턴스로 요청이 가는 상황은 막지 못합니다. 그래서 사용자와 idempotency key에 unique 제약을 두고, 같은 의도의 재전송은 같은 persisted CheckRun을 돌려줍니다. 같은 key를 다른 Monitor에 쓰면 의미 충돌이므로 409입니다. 클라이언트는 결과를 확인하기 전까지 같은 intent key를 유지하고, 성공 acknowledgement를 받은 다음 사용자 새 클릭에는 새 key를 발급합니다.

### 답변 핵심 키워드

- idempotency key
- ack loss
- persisted intent
- user-scoped uniqueness
- semantic conflict
- same result replay
- client key lifecycle
- 409 Conflict

### 백지 구현

#### 구현 목표

동일 사용자와 idempotency key의 재요청을 같은 manual CheckRun으로 수렴시키는 enqueue handler를 작성한다.
#### 인터페이스 또는 함수 시그니처

`enqueueManualCheck(db, userId, monitorId, key, now): Promise<QueuedRun>`
#### 입력과 출력

- 입력: 인증 사용자 ID, owner-validated Monitor ID, printable key, 현재 시각
- 출력: 최초 생성 또는 이전에 생성된 동일 CheckRun
- 실패: 같은 user/key가 다른 의미에 결박됐으면 CONFLICT
#### 반드시 만족해야 할 조건

- key는 길이와 허용 문자 범위를 검증한다.
- unique identity는 최소한 사용자와 key를 포함한다.
- 동일 user/key/monitor 재요청은 새 row를 만들지 않고 같은 ID를 반환한다.
- 동일 user/key가 다른 monitor에 쓰이면 충돌한다.
- 서로 다른 사용자는 같은 key 문자열을 독립적으로 사용할 수 있다.
- 동시 최초 요청 두 개도 한 row로 수렴한다.
#### 경계 조건

- 응답이 commit 후 유실되고 같은 key로 재요청
- 같은 key의 동시 POST
- 같은 사용자·같은 key·다른 Monitor
- 다른 사용자·같은 key
- 빈 key, 제어 문자, 최대 길이 경계
#### 실패 조건

- key를 프로세스 메모리 Map에만 저장한다.
- unique violation을 무조건 500으로 바꾼다.
- 같은 key면 요청 의미를 확인하지 않고 아무 기존 row나 반환한다.
- 성공 응답 전 client가 key를 지워 재시도 때 새 row를 만든다.
#### 필요한 제약

- 현재 idempotency 범위는 manual Check enqueue 하나로 한정한다.
- key를 비밀로 취급할 필요는 없지만 로그 cardinality와 개인정보 측면에서 원문 로깅은 피한다.

```ts
export async function enqueueManualCheck(
  db: CheckDatabase,
  userId: string,
  monitorId: string,
  key: string,
  now: string,
): Promise<QueuedRun> {
  if (!/^[!-~]{1,128}$/.test(key)) {
    throw new ApiError('INVALID_INPUT', 'Invalid Idempotency-Key');
  }
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    const parent = await client.query(
      'SELECT id FROM monitors WHERE id = $1 AND owner_user_id = $2 FOR KEY SHARE',
      [monitorId, userId],
    );
    if (!parent.rows[0]) throw new ApiError('NOT_FOUND', 'Monitor not found');
    const inserted = await client.query<CheckRunRow>(
      `INSERT INTO check_runs
         (id, monitor_id, trigger, state, queued_at, request_user_id, idempotency_key)
       VALUES (gen_random_uuid(), $1, 'MANUAL', 'QUEUED', $4, $2, $3)
       ON CONFLICT (request_user_id, idempotency_key) DO NOTHING
       RETURNING *`,
      [monitorId, userId, key, now],
    );
    // 경쟁 winner가 commit된 뒤 같은 transaction의 다음 statement에서 읽는다.
    const saved = inserted.rows[0] ?? (await client.query<CheckRunRow>(
      'SELECT * FROM check_runs WHERE request_user_id = $1 AND idempotency_key = $2',
      [userId, key],
    )).rows[0];
    if (saved.monitor_id !== monitorId) {
      throw new ApiError('CONFLICT', 'Idempotency-Key has another meaning');
    }
    await client.query('COMMIT');
    return checkRunFromRow(saved) as QueuedRun;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### 구현 후 자가 검증

- 최초 요청은 row 하나를 만들고 replay는 같은 ID를 반환한다.
- 동시 두 요청에서도 row count가 1이다.
- 같은 key의 다른 Monitor 요청은 409 의미로 실패한다.
- 다른 사용자의 같은 key는 충돌하지 않는다.
- commit 뒤 응답 유실 시 replay가 기존 terminal/active 상태를 그대로 반환한다.
- 성공 acknowledgement 뒤 새 클릭은 새 key와 새 row를 사용한다.

### 구현 후 설명할 것

- UI gate와 서버 idempotency의 장애 범위 차이
  - 모범답변: UI gate는 현재 document의 동시 handler만 다룹니다. DB idempotency는 다른 탭·인스턴스·재시작과 commit 후 ack loss 재전송까지 같은 row로 수렴시킵니다.
- idempotency identity에 사용자 범위를 포함한 이유
  - 모범답변: 서로 다른 사용자가 우연히 같은 key 문자열을 써도 독립된 의도입니다. `(request_user_id, idempotency_key)`가 tenant 경계를 보존합니다.
- replay와 semantic conflict를 구분한 이유
  - 모범답변: 같은 target의 replay는 기존 결과를 반환할 수 있지만 다른 Monitor는 동일 key의 의미가 바뀐 것입니다. 새 row도 기존 row도 임의로 선택하지 않고 409로 호출자 오류를 드러냅니다.
- key 보관 기간과 DB 정리 정책을 추가할 때 고려할 점
  - 모범답변: key row를 너무 일찍 지우면 늦은 replay가 중복 실행을 만듭니다. 최대 재전송 기간, terminal history 보존, 개인정보·storage 비용을 기준으로 tombstone 또는 retention을 정해야 합니다.

### 원본 확인 위치

- Thread 10
- `server/contracts.ts` — `idempotencyKey`
- `server/app.ts` — manual Check enqueue
- `server/migrations/007_check_ownership.sql`
- `app/monitors/use-monitors.ts` — `checkIntents`
- `test/ownership.test.ts`
- 관련 Thread: 06(E06) browser admission, 11(E11) ABORTED replay
---

<a id="p25"></a>
## [Thread 11 (E11) / 제목 미노출 — 기록상 lease recovery와 terminal transaction 구현] lease 만료 복구, stale worker 차단과 terminal write 원자성

### 면접 질문

- worker_id만 있고 lease가 없으면 crash 뒤 RUNNING row가 왜 영구 정체됩니까?
  - 꼬리 질문: completion transaction에서 row lock을 얻은 뒤 lease를 다시 검사한 이유는 무엇입니까?
    - 모범답변: UPDATE predicate를 평가한 시점에는 lease가 유효해도 다른 transaction의 row lock을 기다리는 동안 만료될 수 있습니다. lock을 얻은 뒤 DB clock으로 다시 확인하고 만료면 transaction 전체를 rollback합니다.
  - 꼬리 질문: 만료된 RUNNING을 같은 ID의 ABORTED로 만들고 자동 requeue하지 않은 설계 trade-off를 설명해 보세요.
    - 모범답변: endpoint 요청이 실제로 전송됐는지 불확실하므로 자동 재실행은 side effect를 중복시킬 수 있습니다. 같은 ID를 ABORTED로 보존해 안전성을 우선하지만 일시 장애의 자동 복구 가용성은 낮아집니다.

### 30초 모범 답변

worker ID는 누가 소유했는지만 말하고 그 worker가 살아 있는지는 말하지 못합니다. claim 시 DB clock으로 lease 만료를 기록하고, recovery는 `lease_expires_at <= now`인 RUNNING만 같은 ID의 ABORTED로 종료합니다. completion은 owner·RUNNING·lease 조건을 가진 update를 한 explicit transaction에서 수행하고, lock 대기 중 시간이 지났을 수 있으므로 row lock 후 lease를 다시 확인합니다. 자동 requeue를 하지 않아 중복 endpoint 실행 가능성을 피하고, 같은 idempotency key는 ABORTED 사실을 재생하며 새 의도만 새 key로 시작합니다.

### 답변 핵심 키워드

- lease
- database clock
- exact expiry
- stale worker fencing
- row lock recheck
- transactional terminal write
- ABORTED same ID
- no automatic retry

### 백지 구현

#### 구현 목표

live lease를 가진 owner만 terminal 결과를 commit하고, 만료된 RUNNING을 안전하게 ABORTED로 복구하는 repository 함수를 작성한다.
#### 인터페이스 또는 함수 시그니처

`completeWithLease(db, workerId, runId, result, now): Promise<boolean>`, `recoverExpiredChecks(db, now): Promise<number>`
#### 입력과 출력

- 입력: worker ID, run ID, terminal 관찰 결과, 명시적 또는 DB 현재 시각
- 출력: completion 적용 여부와 recovery row 수
- 복구 결과: 같은 CheckRun ID, ABORTED, finishedAt 설정, endpoint 결과 필드 비움
#### 반드시 만족해야 할 조건

- claim은 RUNNING·worker ID·lease expiry를 함께 commit한다.
- completion은 같은 owner, RUNNING, `lease_expires_at > now`를 요구한다.
- row lock을 얻은 뒤 현재 시각 기준 lease를 다시 검사한다.
- terminal update와 commit이 한 explicit transaction 안에 있다.
- recovery는 `lease_expires_at <= now`인 RUNNING만 ABORTED로 바꾼다.
- terminal row는 recovery와 stale completion에서 바뀌지 않는다.
- 복구는 endpoint I/O와 자동 retry를 수행하지 않는다.
#### 경계 조건

- lease 정확히 만료 시각
- completion이 lock을 기다리는 동안 lease 만료
- recovery와 owner completion의 동시 경쟁
- stale worker가 recovery 후 늦게 결과 제출
- 같은 idempotency key replay
#### 실패 조건

- 애플리케이션 시계만 사용해 worker 간 skew를 허용한다.
- lock 전 조건만 보고 lock 후 만료를 재검사하지 않는다.
- autocommit terminal UPDATE가 worker 종료와 애매한 중간 상태를 만든다.
- 만료 row를 무조건 QUEUED로 되돌려 중복 side effect를 유발한다.
#### 필요한 제약

- lease renewal은 현재 범위 밖이며 HTTP deadline이 lease보다 짧다는 전제를 명시한다.
- 복구 결과의 실패 이유는 정책상 null일 수 있으며 관찰 실패와 구분한다.

```ts
export async function completeWithLease(
  db: WorkerDatabase,
  workerId: string,
  runId: string,
  result: ObservedResult,
  now: string,
): Promise<boolean> {
  void now; // production completion은 worker 시계가 아니라 원본처럼 DB clock을 사용한다.
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    const saved = await client.query(
      `UPDATE check_runs
          SET state = $3, http_status = $4, latency_ms = $5,
              failure_reason = $6, finished_at = $7
        WHERE id = $1 AND worker_id = $2 AND state = 'RUNNING'
          AND lease_expires_at > clock_timestamp()
      RETURNING id`,
    [runId, workerId, result.state, result.httpStatus, result.latencyMs,
        result.failureReason, result.finishedAt],
    );
    // row lock 대기 중에도 흐르는 DB clock으로 lease를 다시 확인한다.
    const lease = saved.rows[0] && await client.query<{ valid: boolean }>(
      'SELECT lease_expires_at > clock_timestamp() AS valid FROM check_runs WHERE id = $1',
      [runId],
    );
    if (!lease?.rows[0]?.valid) {
      await client.query('ROLLBACK');
      return false;
    }
    await client.query('COMMIT');
    return true;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

export async function recoverExpiredChecks(
  db: WorkerDatabase,
  now: string,
): Promise<number> {
  const result = await db.query(
    `UPDATE check_runs
        SET state = 'ABORTED', http_status = NULL, latency_ms = NULL,
            failure_reason = NULL, finished_at = $1
      WHERE state = 'RUNNING' AND lease_expires_at <= $1
      RETURNING id`,
    [now],
  );
  return result.rowCount ?? 0;
}
```

### 구현 후 자가 검증

- live owner completion은 terminal로 한 번 전이한다.
- 다른 worker와 만료 owner completion은 false이며 row가 그대로다.
- 정확히 expiry와 같은 시각은 recovery 대상이다.
- lock 대기 중 만료되는 시나리오에서 stale completion이 commit되지 않는다.
- recovery와 completion 경쟁 결과가 terminal 하나로 수렴한다.
- recovery 뒤 idempotency replay가 같은 ABORTED ID를 반환한다.
- 복구 중 endpoint connector 호출이 0이다.

### 구현 후 설명할 것

- worker ID와 lease가 각각 해결하는 문제
  - 모범답변: worker ID는 다른 claimant의 completion을 막는 소유자 표식이고, lease는 crash한 owner의 권리가 언제 끝나는지 정합니다. 둘이 함께 identity와 liveness 경계를 만듭니다.
- DB clock을 권위 시각으로 둔 이유
  - 모범답변: worker별 wall clock skew로 같은 lease를 서로 다르게 판단하지 않게 합니다. 원본 production SQL은 `clock_timestamp()`를 사용하고 명시적 시각은 고정 테스트에만 씁니다.
- lock 후 lease 재검사가 필요한 race
  - 모범답변: 조건 평가와 실제 update/commit 사이 row lock 대기로 시간이 흐를 수 있습니다. 재검사 없이 commit하면 만료 뒤 recovery와 경쟁한 stale 결과가 확정될 수 있습니다.
- 자동 retry 대신 ABORTED로 확정한 안전성·가용성 trade-off
  - 모범답변: 중복 endpoint 호출을 피하고 불확실성을 보존하지만 자동으로 업무를 끝내지 못합니다. 새 사용자 의도나 별도 안전한 retry 정책이 필요합니다.
- lease renewal을 추가할 때 필요한 fencing token 또는 epoch
  - 모범답변: 단순 만료 연장만으로는 이전 owner 메시지를 완전히 구분하기 어렵습니다. claim 세대가 증가하는 fencing token을 모든 renewal·completion 조건에 넣어 stale 세대를 거부해야 합니다.

### 원본 확인 위치

- Thread 11
- `server/worker.ts` — `recoverExpiredChecks`, `completeCheck`, lease 상수
- `server/migrations/008_check_lease.sql`
- `test/recovery.test.ts`
- 관련 Thread: 10(E10) worker ownership·idempotency, 12(E12) total HTTP deadline
---

<a id="p26"></a>
## [Thread 11 (E11) / 제목 미노출 — 기록상 graceful shutdown drain 구현] SIGTERM에서 신규 claim 중단, 현재 작업 drain과 강제 종료

### 면접 질문

- SIGTERM을 받자마자 프로세스를 종료하지 않고 신규 claim만 중단한 뒤 현재 작업을 drain한 이유는 무엇입니까?
  - 꼬리 질문: shutdown grace가 DB close까지 포함돼야 하는 이유는 무엇입니까?
    - 모범답변: endpoint 관찰이 끝나도 terminal transaction이나 pool 종료가 남아 있을 수 있습니다. 그 시간을 제외하면 표면상 drain 성공 뒤 프로세스가 계속 살아 전체 종료 상한이 깨집니다.
  - 꼬리 질문: 제한 시간 안에 작업이 끝나지 않으면 0이 아닌 종료 코드로 끝내고 lease recovery에 맡긴 이유는 무엇입니까?
    - 모범답변: completion을 확정하지 못한 상태를 정상 종료로 위장하지 않고 운영에 실패로 드러냅니다. RUNNING row는 유한 lease를 유지하므로 다른 프로세스가 만료 뒤 ABORTED로 복구합니다.

### 30초 모범 답변

즉시 종료하면 현재 endpoint 결과와 DB completion 사이가 끊겨 RUNNING row가 남습니다. 신호를 받으면 polling과 신규 claim을 먼저 멈추고 이미 소유한 작업 하나만 끝내 terminal commit을 시도한 뒤 DB를 닫습니다. grace timer는 작업뿐 아니라 completion transaction과 pool close까지 포함해야 실제 프로세스 종료 상한이 됩니다. 상한을 넘으면 성공으로 위장하지 않고 비정상 종료해 lease가 만료된 뒤 다른 프로세스가 ABORTED로 복구하게 합니다.

### 답변 핵심 키워드

- graceful shutdown
- stop intake
- drain in-flight
- bounded grace
- DB close
- nonzero forced exit
- lease handoff
- signal idempotency

### 백지 구현

#### 구현 목표

worker loop의 신규 작업 수락과 현재 작업 수명을 분리하고, SIGTERM 후 제한 시간 내 drain 또는 비정상 종료를 결정하는 controller를 작성한다.
#### 인터페이스 또는 함수 시그니처

`runWorkerLoop(deps, options): Promise<ExitResult>`, `requestShutdown(signal): void`
#### 입력과 출력

- 입력: claim 함수, execute/complete 함수, DB close 함수, grace 시간, signal
- 출력: 정상 drain 또는 강제 종료를 나타내는 종료 결과
- 상태: accepting, draining, closed 중 명시적 lifecycle
#### 반드시 만족해야 할 조건

- 첫 shutdown 신호 이후 신규 claim을 시작하지 않는다.
- 이미 claim한 작업은 grace 안에서 완료·commit 기회를 가진다.
- grace는 작업 drain과 DB close 전체를 덮는다.
- 반복 signal은 중복 close·중복 timer를 만들지 않는다.
- 제한 초과는 정상 종료로 보고하지 않는다.
- 강제 종료 뒤 row 복구는 lease 만료 정책에 맡긴다.
#### 경계 조건

- 작업이 없는 idle worker에서 신호
- claim 직전·직후 신호
- completion commit 중 신호
- DB close가 지연되는 경우
- SIGTERM 두 번 또는 SIGINT 뒤 SIGTERM
#### 실패 조건

- 신호 뒤에도 polling timer가 새 claim을 만든다.
- 현재 작업을 기다리지만 timer 상한이 없다.
- DB close를 기다리지 않고 성공 종료를 보고한다.
- 강제 종료를 code 0으로 기록한다.
#### 필요한 제약

- 현재 worker concurrency 1을 기준으로 구현한다.
- 실제 `process.exit` 호출 대신 테스트 가능한 lifecycle 결과와 주입된 종료 함수를 사용한다.

```ts
export async function runWorkerLoop(
  deps: WorkerLoopDependencies,
  options: { shutdownGraceMs: number },
): Promise<ExitResult> {
  let inFlight: Promise<void> | null = null;
  const stopped = new Promise<void>(resolve => {
    if (deps.signal.aborted) resolve();
    else deps.signal.addEventListener('abort', () => resolve(), { once: true });
  });

  while (!deps.signal.aborted) {
    const claimed = await deps.claimNext(deps.signal);
    if (!claimed || deps.signal.aborted) {
      if (!claimed && !deps.signal.aborted) await Promise.race([deps.waitForWork(), stopped]);
      continue;
    }
    inFlight = deps.executeAndComplete(claimed);
    // 신호가 먼저 오면 작업은 background에서 drain 대상으로 계속 둔다.
    await Promise.race([inFlight, stopped]);
    if (!deps.signal.aborted) {
      await inFlight;
      inFlight = null;
    }
  }

  const drainAndClose = (async () => {
    if (inFlight) await inFlight;
    await deps.close();
    return { kind: 'drained', exitCode: 0 } as ExitResult;
  })();
  const deadline = new Promise<ExitResult>(resolve => {
    const timer = setTimeout(() => resolve(
      { kind: 'deadline-exceeded', exitCode: 1 } as ExitResult,
    ), options.shutdownGraceMs);
    void drainAndClose.finally(() => clearTimeout(timer));
  });
  return Promise.race([drainAndClose, deadline]);
}
```

### 구현 후 자가 검증

- idle 신호는 신규 claim 없이 빠르게 정상 종료한다.
- 작업 중 신호는 추가 claim 0, 현재 작업 completion 1이다.
- completion과 DB close가 grace 안이면 정상 종료다.
- stuck 작업 또는 close는 상한 뒤 비정상 종료다.
- 중복 signal이 timer·close·completion을 중복 호출하지 않는다.
- 강제 종료 뒤 lease recovery가 같은 RUNNING row를 ABORTED로 만든다.

### 구현 후 설명할 것

- intake 중단과 in-flight 취소를 분리한 이유
  - 모범답변: 신규 claim은 즉시 막을 수 있지만 이미 endpoint I/O를 시작한 작업은 결과를 terminal commit할 기회를 주는 편이 불확실 RUNNING을 줄입니다. 강제 취소는 별도 의미 설계가 필요합니다.
- grace 범위에 DB close를 포함한 이유
  - 모범답변: completion commit과 pool의 열린 handle까지 닫혀야 프로세스가 실제 종료할 수 있습니다. 작업 Promise만 기준으로 삼으면 종료 수명을 과소 측정합니다.
- 강제 종료를 실패로 관측 가능하게 한 이유
  - 모범답변: deadline 초과는 정상 drain이 아니므로 exit 1과 구조화 event로 orchestration과 운영자가 구분해야 합니다. durable lease가 데이터 복구 경로를 제공합니다.
- 작업 취소를 추가할 때 endpoint I/O와 DB 상태를 어떻게 연결할지
  - 모범답변: abort가 실제 요청 전인지 후인지 구분하고, authoritative 관찰 결과가 없으면 RUNNING을 임의 FAILED로 만들지 않아야 합니다. 취소 ownership·terminal 상태·lease fencing을 함께 정의해야 합니다.

### 원본 확인 위치

- Thread 11
- `server/worker.ts` — worker loop, `SHUTDOWN_GRACE_MS`
- `server/main.ts` — API signal 정리와 비교 가능
- `test/recovery.test.ts`
- 관련 Thread: 14(E24) frontend SIGTERM·container shutdown
