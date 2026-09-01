# 개발자 기술면접 워크북 04 — 큐, 중복 방지, 실행 소유권, 복구

이 문서는 동기 작업을 영속 상태 머신으로 바꾸는 과정과, 여러 API·worker 프로세스가 경쟁할 때 데이터베이스가 지켜야 할 invariant를 다룬다. 프런트의 중복 클릭 방지보다 더 강한 서버 측 보장을 중심으로 연습한다.

---

<!-- coverage: SA-12 -->
<a id="sa-12"></a>
## [Thread 09 / `feat(worker): persist queued checks and execute interval intents`] 영속 실행 의도와 API·worker 분리

### 면접 질문

수동 Check API가 외부 HTTP 완료까지 기다려 200을 반환하던 구조를, 먼저 `QUEUED` 행을 저장하고 202를 반환한 뒤 별도 worker가 같은 ID를 `RUNNING`과 터미널 상태로 진행하는 구조로 바꾼 이유는 무엇입니까?

꼬리 질문:

- API가 202를 반환하기 전에 무엇이 반드시 커밋돼 있어야 합니까?
  - 모범답변: owner 검증과 고유 CheckRun ID를 가진 `QUEUED` intent가 PostgreSQL에 commit돼 있어야 합니다. 202는 실행 완료가 아니라 이 durable 접수 사실의 acknowledgement입니다.
- worker가 `RUNNING`을 커밋하기 전에 외부 I/O를 시작하면 어떤 관찰 불일치가 생깁니까?
  - 모범답변: endpoint는 호출됐는데 DB는 여전히 QUEUED여서 다른 worker가 다시 claim하거나 운영자가 실행 시작을 관찰하지 못할 수 있습니다. claim commit이 I/O보다 선행해야 합니다.
- QUEUED/RUNNING 상태에서 `startedAt`, `finishedAt`, `httpStatus`, `latencyMs`는 어떤 null invariant를 가져야 합니까?
  - 모범답변: QUEUED는 `startedAt`을 포함한 실행·결과 필드가 모두 null이고 queuedAt만 있습니다. RUNNING은 startedAt만 non-null이며 finishedAt과 모든 endpoint 결과는 아직 null입니다.
- API 프로세스와 worker 프로세스를 나누면 장애 격리와 배포에는 어떤 trade-off가 생깁니까?
  - 모범답변: 느린·불안정한 outbound 실행이 API latency와 process lifecycle을 직접 묶지 않지만, 별도 worker 배포·health·queue polling·claim 경쟁·recovery를 운영해야 합니다.

### 30초 모범 답변

동기 API는 외부 대상이 느리거나 멈추면 요청 스레드가 붙잡히고, 응답 전에는 실행 의도 자체가 남지 않아 프로세스 장애 시 추적하기 어렵습니다. 그래서 API 트랜잭션이 고유 ID의 `QUEUED` 행을 커밋한 뒤 202를 반환하고, 별도 worker가 그 행을 `RUNNING`으로 커밋한 다음 외부 I/O를 수행하고 같은 ID를 터미널 상태로 갱신했습니다. 대기·실행 중에는 아직 관측되지 않은 결과 필드를 null로 유지합니다. 이렇게 하면 API 응답성과 실행 상태 관찰은 좋아지지만 worker 운영, 폴링, 중복 claim 방지와 복구 정책이 추가로 필요합니다.

### 답변 핵심 키워드

durable intent, 202 Accepted, state machine, same identity, process separation, committed RUNNING, I/O outside transaction, nullable state fields

### 백지 구현

#### 구현 목표

메모리 구현으로 축소한 실행 저장소와 worker를 작성한다. API는 실행 의도를 저장한 뒤 즉시 반환하고, worker는 같은 실행 ID를 상태 전이시킨다.

#### 인터페이스 또는 함수 시그니처

```java
enum RunState { QUEUED, RUNNING, SUCCEEDED, FAILED }

record CheckRun(
    UUID id,
    UUID monitorId,
    RunState state,
    Instant queuedAt,
    Instant startedAt,
    Instant finishedAt,
    Integer httpStatus,
    Long latencyMs,
    String failureReason
) {}

interface CheckQueue {
    CheckRun enqueue(UUID monitorId, Instant now);
    Optional<CheckRun> startNext(Instant now);
    boolean finish(UUID runId, CheckRun observed);
}

final class CheckWorker {
    boolean executeNext() {
        Optional<CheckRun> claimed = queue.startNext(clock.instant());
        if (claimed.isEmpty()) return false;
        CheckRun running = claimed.get(); // startNext의 상태 변경은 여기 오기 전에 끝난다.
        CheckRun observed = observer.observe(running);
        if (!observed.id().equals(running.id())) {
            throw new IllegalStateException("Observation changed execution identity");
        }
        if (!queue.finish(running.id(), observed)) {
            throw new IllegalStateException("Execution was no longer finishable");
        }
        return true;
    }
}
```

#### 입력과 출력

- API 입력: Monitor ID
- API 출력: 커밋된 `QUEUED` 실행
- worker 출력: 실행할 행이 있었는지 여부
- 외부 관측기: RUNNING 실행과 URL을 받아 결과를 반환한다고 가정

#### 반드시 만족해야 할 조건

- enqueue는 외부 I/O 없이 고유 ID의 `QUEUED` 상태를 만든다.
- QUEUED에는 `queuedAt`만 있고 실행·결과 필드는 null이다.
- worker는 실행 전에 같은 ID를 `RUNNING`으로 만든다.
- RUNNING에는 `startedAt`이 있고 터미널 결과 필드는 null이다.
- 외부 I/O는 상태 변경을 위한 저장소 임계 구역 밖에서 실행한다.
- finish는 같은 ID를 한 번만 터미널 상태로 바꾼다.
- 터미널 실행은 다시 시작하지 않는다.

#### 경계 조건

- 큐가 비어 있음
- worker 실행 중 Monitor가 삭제됨
- 외부 타임아웃 또는 연결 실패
- finish가 두 번 호출됨
- 외부 결과의 ID가 claim한 ID와 다름
- API 저장 직후 API 프로세스 종료

#### 실패 조건

- API가 외부 I/O 완료를 기다리지 않는다.
- worker가 새 실행 ID를 만들지 않는다.
- RUNNING 전에 외부 요청을 시작하지 않는다.
- 아직 없는 결과를 0이나 빈 문자열로 채우지 않는다.
- finish 실패를 조용히 성공으로 처리하지 않는다.

#### 필요한 제약

- 단일 worker만 가정한다. 경쟁 claim은 다음 문제에서 다룬다.
- 구현 시간은 20~25분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] API 반환 시 행이 이미 QUEUED로 존재한다.
- [ ] worker 시작 전 외부 호출 수가 0이다.
- [ ] RUNNING 상태가 외부 호출 중 관찰 가능하다.
- [ ] 완료 후 동일 ID가 터미널 상태가 된다.
- [ ] 각 상태의 null/non-null 필드 조합이 invariant를 만족한다.
- [ ] finish 재호출이 터미널 값을 덮지 않는다.
- [ ] 외부 I/O 중 저장소 잠금이나 트랜잭션을 유지하지 않는다.

### 구현 후 설명할 것

1. 동기 API에서 durable intent로 바꾼 이유
   - 모범답변: 외부 latency와 API request 수명을 분리하고 process crash 뒤에도 실행 의도와 ID를 조회·복구할 수 있게 합니다.
2. 202 반환 전에 커밋해야 하는 상태
   - 모범답변: 인증 owner와 target Monitor에 결합된 QUEUED row가 commit되어 이후 어떤 worker도 같은 ID를 볼 수 있어야 합니다.
3. 같은 실행 ID를 전 상태에서 유지한 이유
   - 모범답변: enqueue acknowledgement, poll, claim, terminal 결과와 recovery가 하나의 의도를 추적하게 하고 새 ID 생성으로 중복 실행을 숨기지 않습니다.
4. 외부 I/O 전 RUNNING 커밋의 관찰 가능성
   - 모범답변: 실행 중임을 API와 다른 worker가 먼저 보고 claim owner를 존중할 수 있습니다. DB transaction과 lock은 commit 뒤 HTTP 동안 유지하지 않습니다.
5. 프로세스 분리로 새로 생긴 운영·일관성 문제
   - 모범답변: worker readiness·shutdown, queue backlog, atomic claim, idempotency, lease 만료와 늦은 completion fencing이 필요해집니다.

### 원본 확인 위치

- Thread 09
- 커밋: `feat(worker): persist queued checks and execute interval intents`
- `backend/src/main/resources/db/migration/V6__queue_check_execution.sql`
- `backend/src/main/java/dev/evolution/monitor/CheckQueue.java`
- `CheckQueue.startNext`, `CheckQueue.finish`
- `backend/src/main/java/dev/evolution/monitor/CheckWorker.java`
- `CheckWorker.executeNext`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `MonitorStore.enqueueCheck`
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java`
- `backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java`
- `tests/browser/worker.spec.ts`
- 관련 Thread: Thread 03의 짧은 트랜잭션, Thread 10의 경쟁 claim, Thread 11의 crash 복구

---

<!-- coverage: SA-13 -->
<a id="sa-13"></a>
## [Thread 09 / `feat(worker): persist queued checks and execute interval intents`] 예약 슬롯의 결정성과 데이터베이스 유일성

### 면접 질문

scheduler가 같은 시각의 tick을 두 번 실행해도 같은 Monitor·예약 슬롯의 실행 의도가 하나만 생기게 한 방법은 무엇입니까? 애플리케이션에서 먼저 존재 여부를 조회하는 것만으로 충분하지 않은 이유를 설명해 주세요.

꼬리 질문:

- Monitor의 생성 시각과 interval로 "현재 슬롯"을 어떻게 정의할 수 있습니까?
  - 모범답변: `elapsed = floor(Duration.between(createdAt, tick).seconds)`를 구하고 첫 interval 이후 `createdAt + floor(elapsed/interval)*interval`로 계산합니다. 원본은 생성 직후가 아니라 첫 interval부터 due입니다.
- worker가 늦게 깨어났을 때 모든 과거 슬롯을 backfill하지 않고 현재 슬롯만 생성한 trade-off는 무엇입니까?
  - 모범답변: 장기 중단 뒤 queue 폭주를 피하지만 놓친 각 interval의 관찰 기록은 복원하지 않습니다. 최신 상태 관찰의 운영 안정성을 과거 완전성보다 우선한 정책입니다.
- 비활성 Monitor와 수동 실행은 예약 유일성에 어떤 영향을 받아야 합니까?
  - 모범답변: disabled Monitor는 scheduler 후보에서 제외하고, manual 실행은 `scheduled_for=null`이라 예약 slot identity와 독립적으로 여러 의도를 가질 수 있습니다.
- partial unique index를 사용한 이유는 무엇입니까?
  - 모범답변: `scheduled_for IS NOT NULL`인 예약 intent에만 `(monitor_id, scheduled_for)` 유일성을 적용해 null인 manual row끼리는 충돌하지 않게 합니다.

### 30초 모범 답변

예약 실행의 논리적 ID는 `monitor_id + scheduled_for`입니다. scheduler는 생성 시각에 interval 배수를 더해 현재 시각이 속한 최신 슬롯을 계산하고, 같은 tick이 반복돼도 그 조합의 행은 하나만 허용합니다. 사전 조회만으로는 두 트랜잭션이 동시에 "없음"을 보고 둘 다 삽입할 수 있으므로 PostgreSQL의 유일 인덱스가 최종 invariant를 보장해야 합니다. 예약 시각이 있는 행에만 제약을 걸어 수동 실행과는 독립시켰고, 늦은 tick은 현재 슬롯 하나만 생성해 폭주성 backfill을 피했습니다.

### 답변 핵심 키워드

logical identity, scheduled slot, deterministic time bucket, unique constraint, race condition, partial index, no catch-up, enabled only

### 백지 구현

#### 구현 목표

Monitor 생성 시각·interval·현재 tick으로 현재 예약 슬롯을 계산하고, 반복 또는 동시 호출에서도 슬롯별 intent가 하나만 생성되도록 인터페이스를 설계한다.

#### 인터페이스 또는 함수 시그니처

```java
record ScheduledMonitor(
    UUID id,
    Instant createdAt,
    int intervalSeconds,
    boolean enabled
) {}

static Optional<Instant> currentSlot(ScheduledMonitor monitor, Instant tick) {
    if (!monitor.enabled()) return Optional.empty();
    long elapsed = Duration.between(monitor.createdAt(), tick).getSeconds();
    if (elapsed < monitor.intervalSeconds()) return Optional.empty();
    long slot = elapsed / monitor.intervalSeconds();
    try {
        return Optional.of(monitor.createdAt().plusSeconds(
                Math.multiplyExact(slot, (long) monitor.intervalSeconds())));
    } catch (ArithmeticException | DateTimeException error) {
        return Optional.empty();
    }
}

interface ScheduledIntentStore {
    boolean insertIfAbsent(UUID monitorId, Instant scheduledFor, Instant queuedAt);
}
```

#### 입력과 출력

- 입력: Monitor와 scheduler tick
- 출력: 아직 due가 아니거나 비활성이면 빈 값, 아니면 현재 슬롯 시각
- 저장 출력: 새 행을 만든 경우 `true`, 이미 존재하면 `false`

#### 반드시 만족해야 할 조건

- tick이 생성 시각보다 interval만큼 지나지 않았으면 슬롯이 없다.
- 슬롯은 생성 시각에 interval의 정수 배를 더한 값이다.
- 동일 Monitor·동일 슬롯은 데이터베이스에서 하나만 존재한다.
- 비활성 Monitor는 예약 intent를 만들지 않는다.
- 늦은 tick은 현재 슬롯만 계산하며 과거 모든 슬롯을 반복 생성하지 않는다.
- 수동 실행은 예약 슬롯 유일성의 대상이 아니다.

#### 경계 조건

- 생성 시각과 같은 tick
- 첫 interval 직전과 정확히 같은 시각
- 동일 tick 두 번
- 다음 슬롯 정확한 경계
- 매우 늦은 tick
- interval 1초
- 동시에 두 scheduler가 같은 슬롯 삽입

#### 실패 조건

- 메모리 Set만으로 다중 프로세스 중복을 막지 않는다.
- `SELECT` 후 조건 없는 `INSERT`만 사용하지 않는다.
- 늦은 tick마다 시작 이후 모든 슬롯을 무제한 생성하지 않는다.
- disabled Monitor를 enqueue하지 않는다.

#### 필요한 제약

- 시간 계산은 초 단위 정수 연산으로 수행한다.
- DB 구현은 unique constraint와 충돌 처리 의도를 보여 주면 된다.
- 구현 시간은 15~20분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] T0, T0, T+interval 호출의 총 행 수가 1, 1, 2가 된다.
- [ ] disabled Monitor 행 수는 항상 0이다.
- [ ] 동일 슬롯 재호출에서 기존 실행 ID가 유지된다.
- [ ] 두 동시 삽입 중 하나만 새 행을 만든다.
- [ ] 수동 intent와 예약 intent가 공존한다.
- [ ] 늦은 tick이 과거 슬롯 수에 비례한 작업을 만들지 않는다.
- [ ] 슬롯 계산에서 음수 나눗셈과 오버플로를 고려했다.

### 구현 후 설명할 것

1. 예약 실행의 논리적 유일 키를 정한 근거
   - 모범답변: 고정 interval 격자에서 같은 Monitor의 같은 scheduledFor는 같은 실행 의도이고, 다른 slot이나 Monitor는 독립 의도입니다.
2. 애플리케이션 사전 조회와 DB 유일성의 차이
   - 모범답변: 두 scheduler가 동시에 "없음"을 볼 수 있어 SELECT는 경쟁을 막지 못합니다. unique index와 `ON CONFLICT DO NOTHING`이 commit을 원자적으로 수렴시킵니다.
3. partial unique index가 수동 실행을 방해하지 않는 이유
   - 모범답변: predicate가 non-null scheduled_for row에만 적용되어 null인 manual intent는 해당 index의 유일성 집합에 들어가지 않습니다.
4. backfill을 하지 않은 운영 trade-off
   - 모범답변: queue와 outbound 부하를 bounded하게 유지하지만 downtime 동안의 각 과거 관찰은 영구히 빠집니다. 별도 catch-up 정책이 필요하다면 상한을 설계해야 합니다.
5. 현재 슬롯 계산의 시간 복잡도
   - 모범답변: 경과 초 나눗셈 한 번으로 O(1)이며 놓친 slot 수만큼 반복하지 않습니다.

### 원본 확인 위치

- Thread 09
- 커밋: `feat(worker): persist queued checks and execute interval intents`
- `backend/src/main/java/dev/evolution/monitor/CheckQueue.java`
- `CheckQueue.scheduleDue`
- `backend/src/main/resources/db/migration/V6__queue_check_execution.sql`
- `backend/src/test/java/dev/evolution/monitor/CheckQueueTest.java`
- 관련 Thread: Thread 10의 `ON CONFLICT` 기반 원자성, Thread 11의 scheduler 재기동 후 복구

---

<!-- coverage: SA-14 -->
<a id="sa-14"></a>
## [Thread 10 / `feat(execution): adopt preserved E10 ownership implementation`] 요청 idempotency를 데이터베이스 유일성으로 보장하기

### 면접 질문

수동 Check 요청에 `Idempotency-Key`를 요구하고, `(ownerUserId, idempotencyKey)` 유일성으로 실행을 재사용한 이유는 무엇입니까? 같은 키가 같은 Monitor와 함께 오면 기존 실행을 반환하지만, 다른 Monitor에 재사용되면 409로 거부한 이유도 설명해 주세요.

꼬리 질문:

- "먼저 조회하고 없으면 삽입"은 동시 요청에서 왜 깨집니까?
  - 모범답변: 두 transaction이 서로 commit되지 않은 row를 보지 못하고 모두 없음으로 판단할 수 있습니다. unique constraint가 없으면 둘 다 다른 ID를 삽입합니다.
- `INSERT ... ON CONFLICT DO NOTHING` 뒤 어떤 행을 읽어야 합니까?
  - 모범답변: 새로 삽입했든 경쟁 winner가 만들었든 `(manual_owner_user_id, idempotency_key)`로 권위 row를 다시 읽고 target monitor가 같은지 확인해야 합니다.
- idempotency key가 owner와 결합되지 않으면 어떤 사용자 간 충돌이 생깁니까?
  - 모범답변: Alice와 Bob이 우연히 같은 key를 쓰면 서로의 의도가 충돌하거나 기존 실행 정보가 누출될 수 있습니다. owner를 tenant scope로 포함해야 합니다.
- 키 형식을 가시 ASCII 1~128자로 제한한 이유는 무엇입니까?
  - 모범답변: 빈 값·공백·제어문자·Unicode normalization의 모호성을 피하고 HTTP header와 PostgreSQL C collation에서 같은 identity를 유지하며 크기도 제한합니다.

### 30초 모범 답변

브라우저의 중복 방지는 네트워크 재시도나 여러 클라이언트를 막지 못하므로 서버가 요청 의도의 논리적 ID를 가져야 합니다. 수동 실행은 `(owner, idempotency key)`를 유일 키로 두고 원자적 insert-on-conflict를 사용했습니다. 같은 owner·key·Monitor면 재시도로 보고 기존 CheckRun ID를 반환해 exactly-once 효과를 만들고, 같은 key를 다른 Monitor에 쓰면 서로 다른 의도를 합칠 수 없으므로 409로 거부합니다. 키는 헤더·로그·DB에서 모호한 정규화가 생기지 않도록 제한된 가시 ASCII와 길이로 검증했습니다.

### 답변 핵심 키워드

idempotency key, logical request identity, atomic insert, unique constraint, replay, semantic conflict, owner scope, retry safety

### 백지 구현

#### 구현 목표

동시 요청에서도 수동 실행 intent가 하나만 생기며, 재시도와 의미 충돌을 구분하는 enqueue 함수를 설계한다.

#### 인터페이스 또는 함수 시그니처

```java
enum EnqueueKind { CREATED, REPLAYED, CONFLICT }

record EnqueueResult(EnqueueKind kind, CheckRun run) {}

interface ManualIntentRepository {
    EnqueueResult enqueue(
        UUID ownerId,
        UUID monitorId,
        String idempotencyKey,
        Instant now
    );
}
```

#### 입력과 출력

- 입력: owner, Monitor, key, 현재 시각
- 출력: 새 실행, 기존 실행 재사용, 의미 충돌
- 공개 상태 예시: CREATED/REPLAYED는 같은 실행 payload, CONFLICT는 409

#### 반드시 만족해야 할 조건

- key는 1~128자의 가시 ASCII만 허용한다.
- 유일성 범위는 owner와 key의 조합이다.
- 동일 owner·key·Monitor 동시 요청은 하나의 행과 하나의 ID로 수렴한다.
- 동일 owner·key·다른 Monitor는 conflict다.
- 다른 owner는 같은 key를 독립적으로 사용할 수 있다.
- 유일성 판단과 삽입은 데이터베이스 원자 연산에 의존한다.
- invalid key는 행과 외부 호출을 만들지 않는다.

#### 경계 조건

- key 없음, 빈 문자열, 공백 포함
- 길이 1과 128, 길이 129
- 비ASCII 문자
- 같은 요청 두 스레드 동시 실행
- 첫 트랜잭션이 아직 커밋되지 않은 상태에서 충돌 요청
- 같은 key·다른 Monitor
- 같은 key·다른 owner

#### 실패 조건

- JVM 내 Map이나 mutex만으로 다중 인스턴스를 제어하지 않는다.
- 조회 후 조건 없는 삽입으로 경쟁 창을 만들지 않는다.
- conflict에서 기존 다른 Monitor 실행을 성공으로 반환하지 않는다.
- invalid key를 trim하거나 변환해 다른 key와 합치지 않는다.

#### 필요한 제약

- SQL은 설명용 한 문장 또는 repository 계약으로 표현해도 된다.
- 구현 시간은 20~25분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 동일 요청 순차 재시도가 같은 ID를 반환한다.
- [ ] 동일 요청 동시 재시도가 행 하나로 수렴한다.
- [ ] 같은 key·다른 Monitor가 conflict다.
- [ ] 다른 owner의 같은 key는 충돌하지 않는다.
- [ ] invalid key에서 저장·외부 호출 횟수가 0이다.
- [ ] 첫 트랜잭션 commit 대기 후에도 올바른 기존 행을 읽는다.
- [ ] idempotency 결과가 worker claim 횟수를 늘리지 않는다.

### 구현 후 설명할 것

1. 프런트 중복 방지와 서버 idempotency의 차이
   - 모범답변: 프런트 Set은 한 document의 동시 클릭만 막고 DB identity는 여러 client·인스턴스·재시작·ack loss 재전송을 수렴시킵니다.
2. 유일 키에 owner를 포함한 이유
   - 모범답변: key 문자열은 전역 발급자가 보장하지 않으므로 사용자 namespace 안에서만 의미가 있습니다. 다른 owner는 같은 문자열을 독립적으로 사용할 수 있어야 합니다.
3. 같은 key·다른 Monitor를 conflict로 본 의미론
   - 모범답변: idempotency는 같은 의도의 replay만 합쳐야 합니다. target이 다르면 한 key의 의미가 바뀐 것이므로 기존 row를 반환하거나 새로 만들지 않고 409로 닫습니다.
4. 원자 insert와 사전 조회 방식의 경쟁 차이
   - 모범답변: insert-on-conflict는 unique index가 동시에 들어온 commit의 winner를 정하지만 사전 조회에는 조회와 insert 사이 gap이 있습니다.
5. key 문자·길이 제한의 호환성과 운영상 이점
   - 모범답변: 여러 HTTP stack·DB collation에서 byte 표현이 안정되고 과도한 index 크기와 로그 cardinality 위험을 제한합니다. 원문 key는 로그 label로 쓰지 않습니다.

### 원본 확인 위치

- Thread 10
- 커밋: `feat(execution): adopt preserved E10 ownership implementation`
- `backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `MonitorStore.enqueueCheck`
- `backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java`
- `backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java`
- `backend/src/test/java/dev/evolution/monitor/SessionClient.java`
- 관련 Thread: Thread 06의 UI 중복 제출, Thread 09의 durable intent, Thread 10의 UI intent lifecycle

---

<!-- coverage: SA-15 -->
<a id="sa-15"></a>
## [Thread 10 / `feat(execution): adopt preserved E10 ownership implementation`] 경쟁 worker claim과 소유자 조건부 완료

### 면접 질문

여러 worker가 같은 QUEUED 실행을 동시에 가져가지 않게 `FOR UPDATE SKIP LOCKED`와 `claim_owner`를 사용한 이유는 무엇입니까? 완료 시 실행 ID만으로 업데이트하지 않고 `state = RUNNING AND claim_owner = :worker` 조건을 둔 이유도 설명해 주세요.

꼬리 질문:

- `SKIP LOCKED`가 처리량에는 어떤 장점이 있고 공정성에는 어떤 trade-off가 있습니까?
  - 모범답변: 다른 worker가 잡은 첫 row를 기다리지 않고 다음 row를 처리해 병렬 처리량이 높습니다. 대신 엄격한 FIFO가 아니며 오래 잠긴 row가 반복해서 건너뛰어질 수 있습니다.
- claim 상태와 worker owner를 한 트랜잭션에서 기록해야 하는 이유는 무엇입니까?
  - 모범답변: QUEUED 선택과 RUNNING·claim_owner 저장 사이의 관찰 가능한 무소유 중간 상태를 없애 한 worker만 실행 권리를 commit하게 합니다.
- 외부 I/O 결과를 가진 worker가 claim을 잃었다면 어떻게 해야 합니까?
  - 모범답변: 결과를 저장하지 않고 폐기해야 합니다. 원격 결과가 있어도 DB 상태·owner 조건을 덮을 권한은 없으며 terminal 또는 ABORTED row를 다시 열면 안 됩니다.
- 행 잠금만 있고 claim owner가 없다면 늦은 worker의 결과를 어떻게 막겠습니까?
  - 모범답변: HTTP 동안 lock을 유지하지 않으므로 commit 뒤에는 claimant를 식별할 표식이 없습니다. owner나 더 강한 fencing token 없이는 늦은 completion을 안전하게 구분할 수 없습니다.

### 30초 모범 답변

경쟁 worker가 먼저 같은 후보를 읽는 문제를 막기 위해 후보 행을 잠그고 이미 잠긴 행은 건너뛰게 했습니다. 선택한 행을 `RUNNING`으로 바꾸고 `claim_owner`를 기록하는 작업은 하나의 트랜잭션이어야 다른 worker가 중간 상태를 보지 않습니다. 외부 I/O는 커밋 뒤 수행합니다. 완료는 실행 ID만으로 하지 않고 현재 상태가 RUNNING이며 claim owner가 자기 자신일 때만 조건부 업데이트해, 중복 claim이나 늦게 도착한 다른 worker가 결과를 덮지 못하게 합니다. 업데이트 행 수가 0이면 소유권을 잃은 것으로 보고 결과를 폐기합니다.

### 답변 핵심 키워드

FOR UPDATE SKIP LOCKED, atomic claim, claim owner, conditional update, fencing, lost ownership, no overwrite, throughput vs fairness

### 백지 구현

#### 구현 목표

두 개 이상의 worker가 공유 큐에서 실행을 claim하고 완료하되, 한 실행은 한 worker만 소유하고 소유자만 터미널 업데이트할 수 있도록 저장소 계약을 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
record Claim(UUID runId, UUID workerId, String url, CheckRun run) {}

interface ClaimQueue {
    Optional<Claim> claimNext(UUID workerId, Instant now);
    boolean finish(UUID workerId, CheckRun result, Instant now);
}
```

#### 입력과 출력

- claim 입력: worker ID와 현재 시각
- claim 출력: 소유권이 기록된 하나의 실행 또는 빈 값
- finish 출력: 자신이 여전히 소유한 RUNNING 실행을 갱신했는지 여부

#### 반드시 만족해야 할 조건

- 하나의 QUEUED 행은 한 worker만 RUNNING으로 전환한다.
- claim 선택과 상태·owner 기록은 한 트랜잭션이다.
- 외부 I/O는 claim 트랜잭션 커밋 뒤에 수행된다.
- finish 조건은 최소한 실행 ID, RUNNING 상태, claim owner를 포함한다.
- 소유권이 맞지 않으면 0행 갱신과 `false`를 반환한다.
- 터미널 행은 다시 claim하거나 덮지 않는다.

#### 경계 조건

- 큐에 한 행, worker 두 개 동시 claim
- 큐에 여러 행, 일부 행 잠김
- worker A claim 후 worker B의 잘못된 finish
- worker A finish 두 번
- claim 후 부모 Monitor 삭제
- worker가 외부 I/O 중 오래 정지

#### 실패 조건

- `SELECT`와 `UPDATE`를 별도 트랜잭션으로 나누지 않는다.
- ID만으로 무조건 터미널 업데이트하지 않는다.
- 0행 업데이트를 성공으로 간주하지 않는다.
- 잠금 중 외부 HTTP를 호출하지 않는다.

#### 필요한 제약

- PostgreSQL을 가정하고 SQL 키워드를 사용해도 된다.
- lease 만료는 다음 문제에서 추가한다.
- 구현 시간은 25~30분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 한 행에 대해 동시 claim 성공 수가 정확히 1이다.
- [ ] 여러 행에서는 worker가 서로 다른 행을 가져갈 수 있다.
- [ ] claim 커밋 전 외부 호출이 시작되지 않는다.
- [ ] 다른 worker의 finish가 0행을 갱신한다.
- [ ] 같은 worker의 중복 finish가 터미널 값을 바꾸지 않는다.
- [ ] 큐가 비었을 때 잠금 대기 없이 빈 결과를 반환한다.
- [ ] 잠금 보유 시간에 외부 I/O가 포함되지 않는다.

### 구현 후 설명할 것

1. `SKIP LOCKED`를 선택한 처리량상의 이유
   - 모범답변: 잠금 대기 대신 다른 QUEUED row를 즉시 claim해 worker connection이 blocked되지 않게 합니다.
2. claim과 상태 전이를 원자적으로 묶은 이유
   - 모범답변: 같은 locked row를 RUNNING과 owner로 함께 commit해야 중복 실행과 무소유 RUNNING 상태가 없습니다.
3. claim owner가 fencing 역할을 하는 방식
   - 모범답변: completion WHERE에 run ID·RUNNING·owner를 넣어 claim하지 않은 worker와 이미 상태가 바뀐 늦은 worker를 row count 0으로 거부합니다.
4. 조건부 UPDATE의 행 수를 결과로 사용한 이유
   - 모범답변: 1행은 현재 소유권이 유효해 terminal 전이가 commit됐다는 뜻이고 0행은 ownership/state 상실을 원자적으로 나타냅니다.
5. 공정성·고갈 가능성과 운영 보완책
   - 모범답변: queued age metric과 오래된 row alert로 starvation을 관찰하고, 필요하면 partition·priority aging·claim ordering을 보강합니다. `SKIP LOCKED` 자체는 fairness 보장이 아닙니다.

### 원본 확인 위치

- Thread 10
- 커밋: `feat(execution): adopt preserved E10 ownership implementation`
- `backend/src/main/resources/db/migration/V7__execution_ownership_and_manual_identity.sql`
- `backend/src/main/java/dev/evolution/monitor/CheckQueue.java`
- `CheckQueue.startNext`, `CheckQueue.finish`
- `backend/src/main/java/dev/evolution/monitor/CheckWorker.java`
- `backend/src/test/java/dev/evolution/monitor/ExecutionOwnershipTest.java`
- `backend/src/test/java/dev/evolution/monitor/E10WorkerProcess.java`
- 관련 Thread: Thread 09의 단일 worker 큐, Thread 11의 lease fencing

---

<!-- coverage: SA-16 -->
<a id="sa-16"></a>
## [Thread 11 / `feat: recover expired worker runs and drain shutdown`] lease 만료, 늦은 완료 차단, graceful drain

### 면접 질문

worker가 `RUNNING` 상태에서 죽으면 해당 실행을 어떻게 복구해야 합니까? 만료된 실행을 새 worker가 재실행하지 않고 `ABORTED`로 확정한 이유와, 기존 worker의 늦은 완료가 복구 결과를 덮지 못하게 한 조건을 설명해 주세요.

꼬리 질문:

- 애플리케이션 시계가 아니라 데이터베이스의 권위 있는 시각을 잠금 획득 후 확인한 이유는 무엇입니까?
  - 모범답변: worker clock skew를 제거하고 lock 대기 시간까지 포함한 실제 DB 상태 전이 시점에 lease 유효성을 판정하기 위해서입니다.
- lease가 만료되기 직전에 finish가 잠금을 기다리면 어떤 시각을 기준으로 판단해야 합니까?
  - 모범답변: row lock을 실제 획득한 뒤의 `clock_timestamp()`를 기준으로 해야 합니다. 그때 expiry와 같거나 늦으면 원래 predicate 시점과 무관하게 finish를 거부합니다.
- `ABORTED`와 `FAILED`를 구분한 이유는 무엇입니까?
  - 모범답변: FAILED는 endpoint 응답·timeout 같은 관찰 결과이고 ABORTED는 crash 때문에 결과를 확정할 수 없다는 상태입니다. ABORTED의 HTTP·latency·reason은 null입니다.
- graceful shutdown에서 "새 claim 중지"와 "현재 작업 drain" 순서는 어떻게 가져가야 합니까?
  - 모범답변: 먼저 accepting flag를 내려 scheduler가 새 claim을 만들지 못하게 하고, 이미 claim한 작업의 HTTP와 finish commit을 제한 시간 동안 기다린 뒤 process 자원을 닫습니다.

### 30초 모범 답변

RUNNING 중 worker가 죽으면 실제 외부 부수효과가 있었는지 알 수 없으므로 자동 재실행은 중복 효과를 만들 수 있습니다. 그래서 claim 시 lease 만료를 저장하고, 복구 worker가 만료된 RUNNING 행을 `ABORTED`로 바꿀 때 복구 시각을 `finishedAt`에 기록하되 HTTP 상태·지연·실패 사유는 null로 두어 "endpoint 결과 불명"을 표현했습니다. finish는 행 잠금을 얻은 뒤 데이터베이스 현재 시각을 다시 읽고, 같은 claim owner이며 lease가 아직 미래일 때만 허용합니다. 이미 복구됐거나 만료됐다면 늦은 결과를 폐기합니다. 종료 시에는 먼저 새 claim을 막고 현재 작업을 제한 시간 동안 drain하며, 남은 미확정 작업은 lease 복구가 처리합니다.

### 답변 핵심 키워드

lease, authoritative DB clock, lock-then-time, fenced finish, ABORTED unknown outcome, no replay, graceful drain, accepting claims

### 백지 구현

#### 구현 목표

claim에 lease를 부여하고, 만료된 RUNNING 실행을 `ABORTED`로 복구하며, 만료 또는 소유권 상실 후 늦은 finish를 거부하는 저장소를 설계한다. 종료 상태도 함께 관리한다.

#### 인터페이스 또는 함수 시그니처

```java
interface LeaseQueue {
    Optional<Claim> claimNext(UUID workerId, Duration lease);
    boolean finish(UUID workerId, CheckRun result);
    int recoverExpired();
}

final class WorkerLifecycle {
    private volatile boolean accepting = true;

    void beginDrain() {
        accepting = false;
    }

    boolean acceptingClaims() {
        return accepting;
    }
}
```

#### 입력과 출력

- claim: worker와 lease 기간을 받아 DB 시각 기준 만료가 포함된 Claim 반환
- finish: 행 잠금 후 권위 시각을 검사해 성공 여부 반환
- recover: 만료 RUNNING을 ABORTED로 바꾼 행 수 반환
- lifecycle: 새 claim 허용 여부 관리

#### 반드시 만족해야 할 조건

- claim은 owner와 `lease_expires_at`을 RUNNING 전이와 함께 저장한다.
- finish는 해당 행을 잠근 후 데이터베이스 현재 시각을 평가한다.
- finish 조건은 RUNNING, 같은 owner, `lease_expires_at > dbNow`를 모두 만족한다.
- recover는 만료된 RUNNING만 ABORTED로 바꾼다.
- ABORTED는 `startedAt`과 복구 시각인 `finishedAt`이 있고, `httpStatus`·`latencyMs`·`failureReason`은 null이어야 한다.
- 복구는 외부 I/O를 재실행하지 않는다.
- drain 시작 후 새 claim은 없고 이미 시작된 작업만 완료 기회를 가진다.

#### 경계 조건

- lease 만료 1마이크로초 전·정확한 시점·직후
- finish가 행 잠금을 기다리는 동안 lease 만료
- recover와 finish 동시 실행
- 다른 worker의 finish
- SIGKILL로 finally가 실행되지 않음
- SIGTERM 중 외부 호출이 정상 종료·타임아웃
- shutdown 제한 시간 초과

#### 실패 조건

- 애플리케이션에서 미리 읽은 시각만으로 잠금 대기 후 완료를 허용하지 않는다.
- 만료 실행을 무조건 재실행하지 않는다.
- ABORTED를 일반 HTTP 실패로 꾸며 결과를 만들지 않는다.
- drain 중 새 claim을 계속 받지 않는다.
- 소유권·lease 조건 없는 finish를 허용하지 않는다.

#### 필요한 제약

- PostgreSQL의 `clock_timestamp()` 또는 동등한 권위 시각을 가정한다.
- 종료 신호 처리 자체보다 상태·호출 순서에 집중한다.
- 구현 시간은 25~30분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] SIGKILL 뒤 RUNNING 행이 남고 lease 전에는 복구되지 않는다.
- [ ] 정확한 만료 시점부터 finish가 거부된다.
- [ ] recover와 finish 경쟁에서 하나의 최종 상태만 남는다.
- [ ] 늦은 원래 worker 결과가 ABORTED를 덮지 않는다.
- [ ] recover가 외부 요청을 발생시키지 않는다.
- [ ] ABORTED 필드 조합이 invariant를 만족한다.
- [ ] drain 시작 후 새 claim 수가 0이다.
- [ ] 현재 작업은 제한 시간 안에 완료될 기회를 갖고, 미완료는 lease로 수렴한다.

### 구현 후 설명할 것

1. 만료 실행을 재시도하지 않고 ABORTED로 둔 이유
   - 모범답변: 기존 worker가 원격 요청을 보냈는지 알 수 없어 자동 재실행이 side effect를 중복시킬 수 있습니다. 불확실성을 같은 ID에 보존합니다.
2. 잠금 획득 후 DB 시각을 읽어야 하는 경쟁 시나리오
   - 모범답변: finish가 recovery나 다른 transaction의 lock을 기다리는 동안 lease가 만료될 수 있습니다. 획득 후 현재 시각으로 재평가해야 stale owner가 commit하지 못합니다.
3. owner와 lease를 함께 검사하는 fencing 조건
   - 모범답변: `state=RUNNING`, `claim_owner=:worker`, `lease_expires_at > clock_timestamp()`가 모두 참인 UPDATE만 1행을 바꿉니다. identity와 시간 권한을 함께 확인합니다.
4. graceful drain과 crash recovery가 서로 보완하는 방식
   - 모범답변: 정상 SIGTERM은 in-flight completion 기회를 높이고, grace 초과·SIGKILL처럼 finally가 없는 경우에는 finite lease와 recovery가 RUNNING을 ABORTED로 수렴시킵니다.
5. lease 길이를 너무 짧거나 길게 잡았을 때의 trade-off
   - 모범답변: 너무 짧으면 정상 HTTP가 끝나기 전에 결과가 fenced되고, 너무 길면 crash 뒤 RUNNING과 readiness 이상이 오래 남습니다. outbound total deadline과 shutdown grace보다 충분히 길되 복구 목표 안이어야 합니다.

### 원본 확인 위치

- Thread 11
- 커밋: `feat: recover expired worker runs and drain shutdown`
- `backend/src/main/java/dev/evolution/monitor/CheckQueue.java`
- `CheckQueue.finish`, `CheckQueue.recoverExpired`
- `backend/src/main/java/dev/evolution/monitor/CheckWorker.java`
- `CheckWorker.acceptingClaims`
- `backend/src/main/java/dev/evolution/monitor/WorkerProcess.java`
- `backend/src/test/java/dev/evolution/monitor/WorkerRecoveryTest.java`
- `app/monitors/api.ts`
- `app/monitors/use-monitor-state.ts`
- `tests/browser/worker.spec.ts`
- 관련 Thread: Thread 10의 claim owner, Thread 14의 readiness와 worker active metric
