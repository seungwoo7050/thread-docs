# 감사 상태, 데이터 정합성과 동시성

이 문서는 마스터 인덱스의 W09~W12를 다룬다. 네 항목은 하나의 감사 시스템을 서로 다른 층에서 본다. W09는 저장 가능한 상태를 정의하고, W10은 terminal 전이를 원자화하며, W11은 중단된 상태의 회수를 병렬화하고, W12는 권위 레코드를 이벤트로 투영한다.

<a id="W09"></a>
## [Thread 10 / `feat(audit): preserve the V1 audit migration`; `feat(audit): migrate to fail-closed lifecycle states`] 호환 가능한 감사 스키마 진화

### 면접 질문

이미 운영 DB에 적용된 `V1__audit_log.sql`을 왜 직접 수정하지 않고 checksum을 고정한 뒤 V2 migration을 추가했습니까? 기존 단일 시점 audit row를 `STARTED`와 terminal lifecycle 모델로 바꾸면서 legacy row와 신규 invariant를 어떻게 함께 보존했습니까?

꼬리 질문:

- Flyway migration 파일을 byte-for-byte 고정하는 것과 schema validation의 역할은 어떻게 다릅니까?
- legacy row의 `completed_at`을 `started_at`과 같게 backfill하는 이유는 무엇입니까?
- `STARTED`, `SUCCESS`, `FAILED`, `UNKNOWN`별 nullable column 규칙을 DB constraint로 둘 이유는 무엇입니까?
- application validation만으로는 부족한 이유는 무엇입니까?
- partial index를 전체 index 대신 선택한 이유는 무엇입니까?
- clean install과 V1 upgrade를 모두 어떤 테스트로 검증해야 합니까?

### 30초 모범 답변

적용된 Flyway migration을 수정하면 기존 DB의 checksum과 새 설치의 schema가 갈라지므로 V1은 immutable하게 고정하고 V2로 진화시켰습니다. V2는 기존 완료 row의 시각 증거를 잃지 않도록 legacy timestamp를 시작·완료 시각으로 backfill하고, 이후 `STARTED`와 terminal 상태의 nullability·status 범위·시간 순서를 check constraint로 강제합니다. stale scan은 `STARTED`만 찾으므로 해당 상태에 대한 partial index를 둡니다. clean V1→V2와 실제 V1 데이터 upgrade를 모두 PostgreSQL에서 검증하고 Hibernate는 생성이 아니라 `validate`만 수행합니다.

### 답변 핵심 키워드

`immutable migration` · `Flyway checksum` · `forward-only evolution` · `legacy backfill` · `database invariant` · `CHECK constraint` · `nullability by state` · `completed >= started` · `partial index` · `clean install and upgrade` · `Hibernate validate`

### 백지 구현

#### 구현 목표

다음 V1 schema를 lifecycle schema로 안전하게 변환하는 V2 SQL migration을 작성한다. 원본 V1 파일은 수정하지 않는다.

#### 입력 schema

`audit_log`에 다음 핵심 column이 이미 존재한다고 가정한다.

- `action_id UUID PRIMARY KEY`
- actor/action/target/reason/trace 정보
- `outcome VARCHAR NOT NULL`
- `http_status INTEGER NOT NULL`
- `occurred_at TIMESTAMPTZ NOT NULL`

#### 목표 schema와 invariant

- lifecycle 시각은 `started_at`, `completed_at`으로 표현한다.
- 허용 outcome은 `STARTED`, `SUCCESS`, `FAILED`, `UNKNOWN`이다.
- `STARTED`: `completed_at IS NULL`, `http_status IS NULL`
- `SUCCESS`, `FAILED`: `completed_at IS NOT NULL`, `http_status IS NOT NULL`
- `UNKNOWN`: `completed_at IS NOT NULL`, `http_status`는 null일 수 있다.
- status가 있으면 `100..599` 범위다.
- 완료 시각이 있으면 시작 시각보다 빠를 수 없다.
- 기존 V1 row는 완료된 legacy evidence로 남아야 하며 데이터가 삭제되면 안 된다.
- stale scan을 위해 `STARTED` row의 `started_at` 조회를 지원한다.

#### skeleton

```sql
-- V2__audit_lifecycle.sql
-- 직접 구현
```

#### 반드시 만족해야 할 조건

- V1 파일을 변경하지 않는다.
- existing row를 잃거나 action ID를 바꾸지 않는다.
- migration 중간 단계에서도 `NOT NULL` 추가 순서 때문에 existing row가 실패하지 않도록 한다.
- legacy `occurred_at`을 사용해 유효한 시작·완료 시각을 만든다.
- legacy outcome을 신규 허용 상태로 일관되게 매핑한다.
- 위 상태별 nullability와 status/시간 범위를 DB constraint로 강제한다.
- invalid row insert/update를 DB가 직접 거부한다.
- `STARTED` 대상 partial index를 만든다.
- clean schema와 existing V1 schema 모두에서 V2 적용이 성공해야 한다.
- application ORM은 schema를 자동 수정하지 않아도 mapping validation을 통과해야 한다.

#### 경계 조건

- V1 table이 비어 있음
- legacy row가 다수 존재함
- 동일한 occurred timestamp를 가진 row
- status 100과 599
- completed 시각과 started 시각이 같은 legacy row
- migration transaction rollback

#### 실패 조건

- status 99 또는 600
- `STARTED`에 completed/status 존재
- `SUCCESS`/`FAILED`에 completed 또는 status 누락
- `UNKNOWN`에 completed 누락
- `completed_at < started_at`
- V1 checksum 변경

#### 필요한 제약

- PostgreSQL과 Flyway를 대상으로 한다.
- migration은 forward-only이며 기존 V1을 재작성하지 않는다.
- 테스트는 in-memory DB가 아니라 실제 PostgreSQL semantics를 사용한다.

### 구현 후 자가 검증

- [ ] 빈 DB에서 V1과 V2가 순서대로 적용된다.
- [ ] V1만 적용된 DB에 legacy row를 넣은 뒤 V2가 적용된다.
- [ ] legacy action ID, actor, action, status가 보존된다.
- [ ] legacy completed/start 시각이 유효한 순서를 가진다.
- [ ] 네 outcome별 valid row가 저장된다.
- [ ] 각 invalid null 조합이 DB constraint에서 거부된다.
- [ ] status 범위 밖 값이 거부된다.
- [ ] 완료 시각 역전이 거부된다.
- [ ] stale 조회 실행 계획이 partial index를 활용할 수 있다.
- [ ] V1 파일 digest가 변경되지 않았다.
- [ ] Hibernate `validate`가 최종 schema와 일치한다.

### 구현 후 설명할 것

1. 적용된 migration을 수정하지 않은 이유
2. backfill 순서와 constraint 추가 순서를 정한 기준
3. application validation과 DB constraint를 중복으로 둔 이유
4. partial index의 쓰기 비용과 stale scan 이득
5. clean install 테스트와 upgrade 테스트가 서로 대체되지 않는 이유

### 원본 확인 위치

- Thread 10
- `feat(audit): preserve the V1 audit migration`
- `test(audit): lock the V1 migration checksum`
- `feat(audit): migrate to fail-closed lifecycle states`
- `src/main/resources/db/migration/V1__audit_log.sql`
- `src/main/resources/db/migration/V2__audit_lifecycle.sql`
- `AuditV1ChecksumTest`
- `AuditMigrationTest`
- `AuditLogEntity`
- `AuditOutcome`
- 관련 Thread: 11, 12, 14

---

<a id="W10"></a>
## [Thread 11 / `feat(audit): guard the terminal lifecycle transition`; `test(audit): allow one terminal transition under race`] 원자적 STARTED→terminal 전이

### 면접 질문

두 thread 또는 복구 worker가 같은 action을 동시에 완료하려 할 때 왜 "먼저 SELECT해서 STARTED인지 확인한 뒤 UPDATE"하지 않고, 조건부 `UPDATE ... WHERE outcome='STARTED' RETURNING ...` 형태를 사용했습니까?

꼬리 질문:

- 애플리케이션의 `synchronized`나 JVM lock으로는 왜 충분하지 않습니까?
- update count가 0인 경우 "이미 완료됨"을 성공으로 간주하지 않은 이유는 무엇입니까?
- terminal outcome으로 `STARTED`가 전달되면 어디서 거부해야 합니까?
- `SUCCESS/FAILED`와 `UNKNOWN`의 status nullability 차이는 어디서 검증합니까?
- 두 경쟁자 중 한 명만 성공했다는 것을 어떻게 테스트합니까?
- `RETURNING`으로 갱신된 row를 받는 것이 별도 SELECT보다 어떤 이점이 있습니까?

### 30초 모범 답변

읽기와 쓰기를 분리하면 두 transaction이 모두 `STARTED`를 읽고 terminal update를 시도하는 TOCTOU race가 생깁니다. 그래서 DB의 단일 조건부 update가 `STARTED`인 행만 바꾸도록 하고, `RETURNING` 결과가 정확히 한 행일 때만 성공으로 인정합니다. JVM lock은 여러 instance를 막지 못하지만 DB row 조건은 모든 writer에 공통입니다. 두 경쟁자가 동시에 실행되면 한 명만 terminal record를 받고 나머지는 "claim 상실"로 실패해야 중복 발행과 서로 다른 최종 상태를 막을 수 있습니다.

### 답변 핵심 키워드

`atomic conditional update` · `TOCTOU` · `compare-and-set` · `database as serialization point` · `WHERE outcome=STARTED` · `RETURNING` · `exactly one row` · `multi-instance safety` · `lost claim` · `race test`

### 백지 구현

#### 구현 목표

하나의 audit action을 `STARTED`에서 terminal 상태로 정확히 한 번 전이하는 repository method를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public enum Outcome { STARTED, SUCCESS, FAILED, UNKNOWN }

public record TerminalRecord(
    UUID actionId,
    Outcome outcome,
    Integer httpStatus,
    Instant startedAt,
    Instant completedAt) {}

public interface SqlExecutor {
  List<TerminalRecord> updateAndReturn(
      String sql,
      List<Object> parameters);
}

public final class AuditCompletionRepository {
  public static TerminalRecord complete(
      SqlExecutor sql,
      UUID actionId,
      Outcome outcome,
      Integer httpStatus) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: SQL executor, action ID, terminal outcome, 선택적 HTTP status
- 출력: DB가 실제로 갱신해 반환한 terminal record

#### 반드시 만족해야 할 조건

- `STARTED`를 terminal outcome으로 받을 수 없다.
- `SUCCESS`와 `FAILED`는 유효한 HTTP status가 필수다.
- `UNKNOWN`은 status가 null일 수 있다.
- SQL은 action ID가 같고 현재 outcome이 `STARTED`인 row만 갱신해야 한다.
- completion timestamp는 DB 시각으로 기록한다.
- 갱신과 record 반환은 하나의 원자적 DB statement/transaction 경계여야 한다.
- 결과 row가 정확히 하나가 아니면 성공으로 간주하지 않는다.
- 이미 terminal인 row를 같은 값으로 다시 완료해도 성공으로 숨기지 않는다.
- 반환 record 자체도 terminal 상태와 시간 invariant를 만족해야 한다.
- 한 action에 대한 동시 두 호출 중 최대 하나만 성공해야 한다.

#### 경계 조건

- status 100, 599
- `UNKNOWN` + null status
- action ID 없음
- 이미 SUCCESS/FAILED/UNKNOWN인 row
- 두 thread가 같은 outcome으로 경쟁
- 두 thread가 서로 다른 outcome으로 경쟁
- DB timeout

#### 실패 조건

- non-terminal outcome
- terminal/status 조합 위반
- update result 0행 또는 2행 이상
- DB exception
- 반환 row의 action ID 또는 상태 불일치

#### 필요한 제약

- process-local lock에 의존하지 않는다.
- SELECT 후 UPDATE 두 단계로 구현하지 않는다.
- transaction timeout을 둘 수 있어야 한다.
- 실패가 발생해도 다른 action row에는 영향을 주지 않는다.

### 구현 후 자가 검증

- [ ] STARTED row가 SUCCESS로 한 번 전이된다.
- [ ] STARTED row가 UNKNOWN/null status로 전이된다.
- [ ] STARTED outcome 입력이 거부된다.
- [ ] SUCCESS/FAILED의 null status가 거부된다.
- [ ] 없는 action ID가 성공으로 처리되지 않는다.
- [ ] 이미 terminal인 row의 재완료가 실패한다.
- [ ] 같은 action을 두 thread가 동시에 완료하면 성공 수가 정확히 1이다.
- [ ] 서로 다른 terminal outcome 경쟁에서도 DB에는 하나만 남는다.
- [ ] 반환 completedAt이 startedAt보다 빠르지 않다.
- [ ] 다른 action row의 상태는 변하지 않는다.
- [ ] SQL 실패가 부분 terminal record를 반환하지 않는다.

### 구현 후 설명할 것

1. 조건부 UPDATE를 compare-and-set처럼 사용한 이유
2. update count 0을 idempotent success로 보지 않은 이유
3. DB clock을 사용한 이유
4. process lock과 database serialization의 차이
5. race 테스트가 단순 repository unit test보다 필요한 이유

### 원본 확인 위치

- Thread 11
- `feat(audit): guard the terminal lifecycle transition`
- `test(audit): allow one terminal transition under race`
- `AuditWriteRepository`
- `AuditTerminalRecord`
- `AuditTerminalRaceTest`
- `AuditOutcome`
- 관련 Thread: 10, 12

---

<a id="W11"></a>
## [Thread 12 / `feat(audit): claim stale STARTED rows safely`; `feat(audit): recover stale actions on schedule`] 동시성 안전한 stale action 복구

### 면접 질문

오래 `STARTED`에 머문 action을 여러 scheduler instance가 동시에 복구할 때 `FOR UPDATE SKIP LOCKED`를 사용한 이유를 설명해 보세요. 단순히 오래된 행을 SELECT한 뒤 하나씩 UPDATE하면 어떤 중복과 blocking 문제가 생깁니까?

꼬리 질문:

- stale cutoff와 batch size를 왜 설정값으로 두었습니까?
- `ORDER BY started_at`은 무엇을 보장하고 무엇을 보장하지 못합니까?
- `SKIP LOCKED`가 공정성을 완전히 보장합니까?
- claim 시 바로 `UNKNOWN` terminal로 바꾸는 이유는 무엇입니까?
- scan 중 DB exception이 나도 scheduler 자체가 계속 살아 있어야 하는 이유는 무엇입니까?
- claimed count, scan failure, publish failure를 왜 서로 다른 metric으로 봅니까?

### 30초 모범 답변

stale recovery도 terminal transition의 한 경쟁자이므로 여러 worker가 같은 `STARTED` row를 잡지 못하게 해야 합니다. CTE에서 오래된 row를 시작 시각 순으로 제한하고 `FOR UPDATE SKIP LOCKED`로 각 worker가 잠긴 행을 기다리지 않고 다른 batch를 claim한 뒤, 같은 statement에서 `UNKNOWN`으로 바꾸고 반환합니다. 이렇게 해야 duplicate recovery와 긴 lock wait를 줄일 수 있습니다. scheduler는 scan 실패를 metric으로 기록하고 예외를 격리해 다음 주기에 다시 진행하며, claim과 Kafka 발행의 실패는 별도 의미로 관측합니다.

### 답변 핵심 키워드

`stale lease recovery` · `batch claim` · `FOR UPDATE SKIP LOCKED` · `oldest first` · `no duplicate workers` · `UNKNOWN terminal` · `bounded scan` · `fixed delay` · `failure isolation` · `progress metrics` · `starvation trade-off`

### 백지 구현

#### 구현 목표

PostgreSQL에서 stale `STARTED` row를 제한된 batch로 원자적으로 `UNKNOWN` 처리해 반환하고, 이를 호출하는 scheduler wrapper를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface StaleAuditRepository {
  List<TerminalRecord> claimStale(Duration staleAfter, int batchSize);
}

public interface TerminalPublisher {
  void publish(TerminalRecord record);
}

public interface Metrics {
  void increment(String name, double amount);
}

public final class StaleAuditScanner {
  public static void scan(
      StaleAuditRepository repository,
      TerminalPublisher publisher,
      Metrics metrics,
      Duration staleAfter,
      int batchSize) {
    // 직접 구현
  }
}
```

repository SQL skeleton:

```sql
-- 오래된 STARTED row를 서로 다른 worker가 중복 claim하지 않고
-- 같은 statement에서 UNKNOWN으로 전이해 반환한다.
-- 직접 구현
```

#### 입력과 출력

- repository 입력: stale duration과 batch size
- repository 출력: 이번 transaction에서 실제로 `UNKNOWN`으로 전이한 terminal records
- scanner 출력: 없음. metric과 publisher side effect를 발생시킨다.

#### 반드시 만족해야 할 조건

- `staleAfter`는 양수, `batchSize`는 1 이상이어야 한다.
- 기준 시각보다 충분히 오래된 `STARTED` row만 대상이다.
- fresh `STARTED`와 이미 terminal인 row는 변경하지 않는다.
- 오래된 순서로 최대 batch size만 선택한다.
- 선택 단계에서 row lock과 `SKIP LOCKED`를 사용한다.
- 선택된 row를 같은 DB statement/transaction에서 `UNKNOWN`, null status, 현재 completed time으로 바꾼다.
- 실제 갱신한 row만 반환한다.
- 여러 worker가 동시에 호출해도 같은 action ID가 두 결과에 나타나지 않는다.
- scanner는 반환된 각 record를 한 번씩 publisher에 전달한다.
- scanner는 claimed 수를 metric에 누적한다.
- repository 또는 scanner-level 예외가 발생해도 예외를 격리하고 scan failure metric을 올려 다음 호출이 가능해야 한다.
- publisher 자체가 best-effort failure를 내부 격리한다는 계약을 명확히 한다.

#### 경계 조건

- stale row 0개
- batch size보다 적거나 많은 row
- cutoff와 정확히 같은 started time
- lock된 가장 오래된 row와 lock되지 않은 다음 row
- 두 scanner가 동시에 실행
- repository가 첫 scan 실패 후 다음 scan 성공
- publisher가 일부 record 발행 실패

#### 실패 조건

- staleAfter 0/음수
- batch size 0/음수
- DB unavailable
- transaction timeout
- 반환 record가 terminal invariant를 위반

#### 필요한 제약

- worker가 lock 해제까지 무한 대기하지 않게 한다.
- batch당 메모리 사용은 `O(batchSize)`다.
- scheduler thread가 한 번의 예외로 종료되지 않게 한다.
- claim과 publish는 같은 원자 transaction이라고 가장하지 않는다.

### 구현 후 자가 검증

- [ ] stale STARTED만 UNKNOWN으로 바뀐다.
- [ ] fresh STARTED와 기존 terminal row는 그대로다.
- [ ] 반환 수가 batch size를 넘지 않는다.
- [ ] 가장 오래된 unlocked row부터 선택된다.
- [ ] 두 worker 결과의 action ID 교집합이 비어 있다.
- [ ] locked row 때문에 전체 batch가 멈추지 않는다.
- [ ] 완료 시각과 null status가 terminal invariant에 맞다.
- [ ] 빈 결과에서도 claimed metric이 잘못 증가하지 않는다.
- [ ] DB 실패 후 다음 scan이 정상 실행된다.
- [ ] scan failure와 publish failure metric이 혼동되지 않는다.
- [ ] scanner가 반환 record를 누락·중복 publish하지 않는다.

### 구현 후 설명할 것

1. `SKIP LOCKED`가 throughput을 높이는 방식
2. oldest-first와 starvation 가능성의 trade-off
3. claim 시 UNKNOWN으로 terminal 처리한 이유
4. claim transaction과 Kafka publish를 분리한 이유
5. scheduler에서 예외를 삼키되 metric을 남기는 이유

### 원본 확인 위치

- Thread 12
- `feat(audit): claim stale STARTED rows safely`
- `feat(audit): recover stale actions on schedule`
- `feat(audit): stream recovered stale actions`
- `test(audit): stream every stale transition once`
- `AuditWriteRepository.claimStale`
- `AuditStaleScheduler`
- `AuditStaleSchedulerTest`
- 관련 Thread: 10, 11, 13

---

<a id="W12"></a>
## [Thread 13 / `feat(audit): define terminal audit event schema`; `feat(audit): publish terminal actions best effort`] 권위 DB와 best-effort Kafka 투영

### 면접 질문

감사 row와 Kafka event를 모두 남겨야 하는데 왜 Kafka 발행 실패가 API 및 DB 성공을 뒤집지 않도록 설계했습니까? 이 선택이 제공하는 보장과 잃는 보장을 설명하고, transactional outbox와 비교해 보세요.

꼬리 질문:

- terminal DB update보다 먼저 event를 발행하면 어떤 ghost event가 생길 수 있습니까?
- `KafkaTemplate.send` 호출 자체가 던지는 실패와 반환 future가 나중에 실패하는 경우를 각각 어떻게 처리합니까?
- event key를 actor ID로 둔 선택은 partition ordering에 어떤 의미가 있습니까?
- producer idempotence와 application-level exactly once는 왜 같은 말이 아닙니까?
- Avro schema fingerprint를 고정하는 테스트는 무엇을 막습니까?
- 실제 Kafka 통합 테스트에서 producer factory, template, consumer, admin client를 명시적으로 정리한 이유는 무엇입니까?

### 30초 모범 답변

PostgreSQL row가 감사의 권위 기록이고 Kafka는 검색·분석을 위한 파생 투영입니다. 따라서 guarded terminal update가 성공한 뒤에만 event를 보내고, 동기 send 실패와 비동기 ack 실패는 metric·로그로 남기되 DB 결과와 API 응답을 뒤집지 않습니다. 이 구조는 broker 장애가 관리자 명령 가용성을 막지 않지만, DB 성공 후 event가 유실될 수 있어 완전한 전달 보장은 없습니다. 그 보장이 필요하면 outbox와 relay를 도입해야 하며, 현재 설계는 단순성과 가용성을 선택한 best-effort trade-off입니다.

### 답변 핵심 키워드

`authoritative source` · `derived projection` · `publish after commit` · `best effort` · `sync failure` · `async completion failure` · `failure metric` · `Avro compatibility` · `producer idempotence` · `outbox alternative` · `resource lifecycle`

### 백지 구현

#### 구현 목표

terminal record를 직렬화해 asynchronous broker로 보내되, 모든 발행 실패를 격리하고 관측하는 best-effort publisher를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface AsyncBroker {
  CompletionStage<Void> send(String topic, String key, byte[] value);
}

public interface EventEncoder<T> {
  byte[] encode(T value);
}

public interface FailureRecorder {
  void record(String metricName, UUID actionId, Throwable failure);
}

public final class BestEffortTerminalPublisher {
  public static void publish(
      TerminalRecord record,
      AsyncBroker broker,
      EventEncoder<TerminalRecord> encoder,
      FailureRecorder failures,
      String topic,
      String actorId) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 이미 DB에서 terminal로 확정된 record, 비동기 broker, encoder, failure recorder, topic, event key
- 출력: 없음. 발행 결과는 best-effort side effect다.

#### 반드시 만족해야 할 조건

- null 또는 non-terminal record를 발행하지 않는다.
- DB completion이 성공한 뒤 호출된다는 상위 계약을 문서화한다.
- serialization 실패를 호출자에게 전파하지 않고 failure metric에 기록한다.
- `send`가 즉시 던지는 exception을 격리한다.
- 반환 stage가 나중에 exceptional completion되는 경우도 기록한다.
- 성공한 future에 failure metric을 올리지 않는다.
- 동일 실패를 동기·비동기 경로에서 이중 집계하지 않는다.
- action ID와 action type 같은 안전한 식별 정보만 로그에 남기고 event payload/credential을 그대로 남기지 않는다.
- topic과 key가 null/blank이면 구성 오류로 기록한다.
- publisher 실패가 authoritative terminal record나 API 결과를 변경하지 않는다.

#### 경계 조건

- encoder exception
- `send` 즉시 exception
- future exceptional completion
- future 정상 완료
- broker가 null stage를 반환하는 비정상 구현
- 매우 큰 payload
- 같은 terminal record의 중복 publish 호출

#### 실패 조건

- invalid terminal record
- invalid topic/key
- serialization/broker failure
- callback executor rejection

#### 필요한 제약

- `publish`는 broker ack를 동기 blocking wait하지 않는다.
- 실패 기록 자체가 exception을 던질 때의 방어 정책을 명시한다.
- delivery exactly-once를 주장하지 않는다.
- 실제 broker test에서 생성한 resource를 명시적으로 종료한다.

### 구현 후 자가 검증

- [ ] 정상 record가 지정 topic/key로 한 번 send된다.
- [ ] DB completion 실패를 모사한 경로에서는 publisher가 호출되지 않는다.
- [ ] encoder 실패가 호출자에게 전파되지 않고 metric이 증가한다.
- [ ] sync send 실패가 격리된다.
- [ ] async ack 실패가 격리된다.
- [ ] 정상 ack에는 failure metric이 증가하지 않는다.
- [ ] payload나 secret이 실패 로그에 노출되지 않는다.
- [ ] callback 실패가 request thread를 나중에 깨뜨리지 않는다.
- [ ] schema round-trip과 fingerprint 검증이 통과한다.
- [ ] 실제 Kafka test 종료 후 producer/consumer/admin resource가 남지 않는다.
- [ ] best-effort라서 가능한 event loss window를 문서화했다.

### 구현 후 설명할 것

1. DB를 권위 저장소로 선택한 이유
2. publish-after-commit이 막는 ghost event와 막지 못하는 loss window
3. sync/async failure를 모두 처리해야 하는 이유
4. producer idempotence와 outbox의 보장 차이
5. Kafka를 readiness dependency로 두지 않은 설계 trade-off

### 원본 확인 위치

- Thread 13
- `feat(audit): define terminal audit event schema`
- `test(audit): pin terminal event compatibility`
- `feat(audit): publish terminal actions best effort`
- `test(audit): contain Kafka publish failures`
- `test(audit): deliver terminal events to Kafka`
- `test(audit): publish terminal events through real Kafka`
- `AdminActionRecorded.avsc`
- `AdminActionPublisher`
- `AuditKafkaConfiguration`
- `AuditService`
- `AdminActionPublisherFailureTest`
- `AdminActionPublisherBrokerTest`
- `AdminActionPublisherRealKafkaTest`
- `RealKafkaAuditFixture`
- 관련 Thread: 10, 11, 12, 15
