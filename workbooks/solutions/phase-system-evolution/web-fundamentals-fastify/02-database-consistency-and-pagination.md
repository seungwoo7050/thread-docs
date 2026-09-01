# 데이터 정합성, 마이그레이션과 페이지네이션 면접 워크북

Thread 03·07·13(E20)의 공통 주제는 데이터 계약을 코드 밖의 PostgreSQL까지 확장하는 것이다. 마이그레이션 이력, 실제 schema, row mapping, cursor, index가 같은 invariant를 가리키도록 설계했는지에 초점을 둔다.

## 문서 내 면접 포인트

- [P07 체크섬을 포함한 순차 마이그레이션과 단일 트랜잭션](#p07)
- [P08 애플리케이션 시작 전 실제 DB schema의 fail-fast 검증](#p08)
- [P09 명시적 row mapping과 PostgreSQL을 단일 진실 원천으로 두는 설계](#p09)
- [P10 조건에 결박된 cursor와 안정적인 seek pagination](#p10)
- [P11 실제 query plan으로 부분 인덱스를 선택하는 판단](#p11)

---

<a id="p07"></a>
## [Thread 03 (E03) / `Introduce checked PostgreSQL schema migrations`] 체크섬을 포함한 순차 마이그레이션과 단일 트랜잭션

### 면접 질문

- `schema_migrations`의 version만 저장하지 않고 SQL checksum까지 저장한 이유는 무엇입니까?
  - 꼬리 질문: 적용 이력이 예상 목록의 prefix인지 검사해야 하는 이유와, 중간 version 누락·순서 변경을 허용하면 생기는 문제를 설명해 보세요.
    - 모범답변: 마이그레이션은 앞 단계의 schema를 전제로 하므로 적용 이력은 명시된 목록의 정확한 prefix여야 합니다. 집합만 같거나 중간 version이 빠지면 같은 version 이름 아래 서로 다른 schema가 생길 수 있습니다.
  - 꼬리 질문: DDL과 migration 이력 INSERT를 같은 명시적 connection과 transaction에서 처리한 이유는 무엇입니까?
    - 모범답변: PostgreSQL transaction은 connection에 귀속됩니다. 같은 client에서 DDL과 이력 INSERT를 직렬 실행해야 둘 중 하나만 남는 반쪽 적용을 rollback할 수 있습니다.

### 30초 모범 답변

version만 기록하면 이미 적용된 파일의 내용이 나중에 바뀌어도 같은 마이그레이션으로 오인합니다. 그래서 애플리케이션이 기대하는 순서와 checksum을 적용 이력과 비교하고, 정확한 prefix만 허용해 append-only 규칙을 지킵니다. 새 DDL과 이력 기록은 한 connection의 transaction에서 직렬로 실행해야 DDL만 적용되거나 이력만 남는 반쪽 상태를 막을 수 있습니다. 오류 시 rollback하고, 성공·실패 모두에서 client를 release하고 pool을 닫는 lifecycle도 계약에 포함됩니다.

### 답변 핵심 키워드

- append-only migration
- ordered prefix
- checksum
- single connection
- transactional DDL
- rollback
- client release
- fail fast

### 백지 구현

#### 구현 목표

예상 마이그레이션 목록과 적용 이력을 검증하고, 남은 항목을 한 트랜잭션에서 순서대로 적용하는 runner를 작성한다.
#### 인터페이스 또는 함수 시그니처

`applyMigrations(db, expected): Promise<string[]>`
#### 입력과 출력

- `expected`: version, checksum, SQL을 가진 순서 있는 목록
- `db`: 한 connection transaction을 제공하는 어댑터
- 출력: 이번 실행에서 새로 적용된 version 목록
#### 반드시 만족해야 할 조건

- 현재 이력은 예상 목록의 정확한 prefix여야 한다.
- 같은 version의 checksum 불일치, 알 수 없는 version, 순서 변경, 적용 이력 과잉은 시작 전에 거부한다.
- 남은 SQL과 이력 INSERT는 같은 transaction·connection에서 직렬 실행한다.
- 한 항목이라도 실패하면 전체 신규 적용분을 rollback한다.
- 반복 실행은 빈 적용 목록을 반환한다.
- 모든 경로에서 connection과 pool 수명을 정리한다.
#### 경계 조건

- 적용 이력이 비어 있는 fresh schema
- 모든 마이그레이션이 이미 적용된 schema
- 첫 항목 checksum 불일치
- 중간 version이 빠지고 뒤 version만 있는 이력
- 마지막 DDL은 성공했지만 이력 INSERT가 실패하는 경우
#### 실패 조건

- 여러 pool connection에 transaction 작업을 분산한다.
- checksum 불일치를 경고만 하고 계속 실행한다.
- 부분 적용을 commit한다.
- rollback 실패가 원래 오류를 완전히 가린다.
#### 필요한 제약

- 마이그레이션 파일 목록은 runner가 임의 디렉터리 정렬로 추론하지 않고 명시적 순서를 사용한다.
- 동시 runner 조정은 별도 문제로 두되, 현재 범위와 한계를 설명한다.

```ts
type Migration = { version: string; checksum: string; sql: string };

export async function applyMigrations(
  db: MigrationDatabase,
  expected: readonly Migration[],
): Promise<string[]> {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    const applied = await client.appliedMigrations();
    if (applied.length > expected.length || applied.some((row, index) =>
      row.version !== expected[index]?.version ||
      row.checksum !== expected[index]?.checksum)) {
      throw new Error('Migration history does not match this application');
    }

    const added: string[] = [];
    for (const migration of expected.slice(applied.length)) {
      // DDL과 이력은 반드시 이 transaction의 같은 connection에 기록한다.
      await client.query(migration.sql);
      await client.query(
        'INSERT INTO schema_migrations (version, checksum) VALUES ($1, $2)',
        [migration.version, migration.checksum],
      );
      added.push(migration.version);
    }
    await client.query('COMMIT');
    return added;
  } catch (error) {
    try { await client.query('ROLLBACK'); }
    catch { /* 원래 적용 실패를 보존한다. */ }
    throw error;
  } finally {
    client.release();
    await db.end();
  }
}
```

### 구현 후 자가 검증

- fresh, 부분 적용, 완전 적용 상태에서 결과가 맞다.
- checksum·version·순서가 하나라도 다르면 DDL 실행 전 실패한다.
- 두 번째 실행은 아무 변경도 만들지 않는다.
- 중간 SQL 또는 이력 INSERT 실패 후 신규 테이블·이력 행이 남지 않는다.
- 성공과 실패 모두 connection release와 pool 종료가 한 번씩 일어난다.
- 적용 순서가 입력 목록과 정확히 같다.

### 구현 후 설명할 것

- checksum을 version과 함께 저장한 이유
  - 모범답변: version이 같아도 이미 적용된 SQL 파일이 수정되면 실행 이력과 현재 코드의 기대 schema가 달라집니다. SHA-256 checksum 비교로 이런 사후 변경을 시작 전에 거부합니다.
- prefix 검사가 단순 집합 비교보다 강한 이유
  - 모범답변: prefix 검사는 누락·재정렬·알 수 없는 version을 모두 막고 오직 뒤에 새 migration을 추가하는 append-only 이력만 허용합니다.
- 한 connection을 transaction 전체에 고정한 이유
  - 모범답변: `BEGIN`의 원자성은 해당 PostgreSQL session에만 적용됩니다. pool의 다른 connection으로 DDL이나 이력 INSERT가 나가면 하나의 rollback 경계가 아닙니다.
- 동시 마이그레이션 실행을 허용하려면 추가로 필요한 잠금 전략
  - 모범답변: 현재 원본에는 runner 간 조정이 없습니다. 여러 인스턴스가 동시에 실행할 수 있다면 advisory lock이나 동등한 단일 실행 잠금을 transaction 초기에 획득해야 합니다.

### 원본 확인 위치

- Thread 03
- `server/migrate.ts` — `migrationFiles`, `checkMigrationHistory`, `migrate`
- `server/database.ts` — `databaseConfig`, `databasePool`, `schemaIdentifier`
- `server/migrations/001_monitors.sql`, `002_check_runs.sql` 및 이후 순차 파일
- `test/persistence.test.ts`
- 관련 Thread: 04·05·07·09·10·11·12·13에서 migration chain 확장
---

<a id="p08"></a>
## [Thread 03 (E03) / `Reject unexpected PostgreSQL columns before API startup`] 애플리케이션 시작 전 실제 DB schema의 fail-fast 검증

### 면접 질문

- 마이그레이션 이력이 맞아도 `information_schema`와 catalog를 다시 검사한 이유는 무엇입니까?
  - 꼬리 질문: 컬럼 이름·타입만 확인하고 nullability, timestamp precision, PK/FK, cascade를 생략하면 어떤 오류를 놓칠 수 있습니까?
    - 모범답변: 필수값이 `null`이 되거나 `timestamptz(3)` 정밀도가 달라지고, 중복 ID·고아 CheckRun·부모 삭제 후 잔존 row가 생기는 계약 위반을 놓칩니다. 이름과 타입만 같아도 데이터 invariant는 다를 수 있습니다.
  - 꼬리 질문: 호환되지 않는 schema에서 요청을 받아 일부 기능만 실패하게 두는 대신 listen 전에 종료한 trade-off를 설명해 보세요.
    - 모범답변: 배포 시 가용성은 낮아질 수 있지만 호환되지 않는 인스턴스가 잘못된 row를 쓰거나 요청별로 불규칙하게 실패하지 않습니다. 이 프로젝트는 schema 계약을 시작 조건으로 선택했습니다.

### 30초 모범 답변

마이그레이션 이력은 해당 runner가 기록한 역사일 뿐, 운영자가 수동으로 schema를 바꾸거나 다른 도구가 제약을 수정한 상태까지 증명하지 못합니다. 그래서 시작 시 실제 컬럼, 타입, nullability, timestamp 정밀도, PK·FK·삭제 규칙과 예상 밖 컬럼을 확인합니다. 계약이 다르면 listen 전에 실패해 잘못된 데이터 쓰기나 요청별 랜덤 장애를 막습니다. 가용성은 낮아질 수 있지만, 데이터 계약이 깨진 인스턴스를 트래픽에 넣지 않는 쪽을 선택한 것입니다.

### 답변 핵심 키워드

- schema drift
- catalog introspection
- fail fast
- exact contract
- nullability
- constraint verification
- startup gate
- readiness 이전 검증

### 백지 구현

#### 구현 목표

DB에서 읽은 schema snapshot을 기대 명세와 비교해 호환 여부를 결정하는 순수 검증기를 작성한다.
#### 인터페이스 또는 함수 시그니처

`verifySchema(actual, expected): void`
#### 입력과 출력

- `actual`: 테이블별 컬럼·타입·nullability·정밀도·PK·FK·삭제 규칙 snapshot
- `expected`: 애플리케이션이 요구하는 정확한 schema 명세
- 출력: 일치하면 없음, 불일치하면 진단 가능한 시작 오류
#### 반드시 만족해야 할 조건

- 필수 테이블과 컬럼의 누락을 잡는다.
- 타입, nullability, timestamp precision 차이를 구분한다.
- PK·FK 대상과 삭제 규칙 차이를 잡는다.
- 예상하지 않은 필수 계약 침해 컬럼도 정책에 따라 거부한다.
- 여러 차이가 있어도 민감 데이터 없이 충분한 진단을 제공한다.
#### 경계 조건

- 타입 이름의 DB 표기 차이
- 컬럼 순서는 다르지만 계약은 같은 경우
- nullable만 다른 경우
- FK 대상은 같지만 `ON DELETE`가 다른 경우
- 애플리케이션이 아직 모르는 새 migration이 적용된 경우
#### 실패 조건

- 첫 요청이 오기 전까지 오류를 숨긴다.
- 테이블 존재 여부만 보고 호환으로 판정한다.
- 실제 row 데이터나 비밀 값을 진단 로그에 넣는다.
#### 필요한 제약

- 검증기는 허용할 차이와 금지할 차이를 명시해야 한다.
- DB catalog 조회와 snapshot 비교 로직을 분리한다.

```ts
type SchemaSnapshot = {
  tables: Record<string, TableSnapshot>;
};

export function verifySchema(
  actual: SchemaSnapshot,
  expected: SchemaSnapshot,
): void {
  const differences: string[] = [];
  const compare = (path: string, left: unknown, right: unknown): void => {
    if (left === right) return;
    if (left === null || right === null || typeof left !== 'object' ||
        typeof right !== 'object' || Array.isArray(left) || Array.isArray(right)) {
      differences.push(path);
      return;
    }
    const leftRecord = left as Record<string, unknown>;
    const rightRecord = right as Record<string, unknown>;
    const keys = new Set([...Object.keys(leftRecord), ...Object.keys(rightRecord)]);
    for (const key of [...keys].sort()) {
      if (!Object.hasOwn(leftRecord, key) || !Object.hasOwn(rightRecord, key)) {
        differences.push(`${path}.${key}`);
      } else {
        compare(`${path}.${key}`, leftRecord[key], rightRecord[key]);
      }
    }
  };

  // 객체 key 순서는 무시하지만 컬럼·제약의 값과 집합은 정확히 비교한다.
  compare('schema', actual.tables, expected.tables);
  if (differences.length > 0) {
    throw new Error(`Incompatible database schema: ${differences.join(', ')}`);
  }
}
```

### 구현 후 자가 검증

- 정확히 일치하는 snapshot만 통과한다.
- 누락·추가·타입·nullable·precision·PK·FK·cascade 차이를 각각 독립적으로 잡는다.
- 컬럼 순서만 다른 경우 정책대로 처리한다.
- 여러 불일치가 있을 때 진단이 어느 계약이 깨졌는지 식별 가능하다.
- 비밀 값과 실제 row 내용은 오류에 포함되지 않는다.
- 시작 코드가 listen 전에 검증 실패를 전파한다.

### 구현 후 설명할 것

- migration history 검사와 실제 schema 검사가 서로 대체되지 않는 이유
  - 모범답변: 이력은 runner가 기록한 과거를 검증하고 catalog 검사는 현재 물리 schema를 검증합니다. 수동 변경이나 다른 도구의 drift는 이력이 맞아도 존재할 수 있습니다.
- 정확한 schema 검증과 전·후방 호환 배포의 충돌
  - 모범답변: 예상 밖 컬럼까지 거부하면 expand/contract 배포 중 구·신 버전의 동시 실행이 어렵습니다. 그 방식이 필요하면 버전별로 허용할 superset을 명시해야 하며, 현재 구현은 exact contract와 fail-fast를 택했습니다.
- 예상 밖 컬럼을 허용할지 거부할지 정한 기준
  - 모범답변: 원본은 네 핵심 테이블의 예상 밖 컬럼도 drift로 거부합니다. raw row를 직접 노출하지 않더라도 migration chain 밖의 schema를 운영하지 않겠다는 강한 정책입니다.
- 검증 실패를 readiness 503으로 둘지 프로세스 시작 실패로 둘지의 trade-off
  - 모범답변: 일시적 저장소 장애는 readiness로 표현할 수 있지만 구조적으로 호환되지 않는 schema는 자동 회복되지 않습니다. 원본은 listen 전 시작 실패로 배포 오류를 즉시 드러냅니다.

### 원본 확인 위치

- Thread 03
- `Reject unexpected PostgreSQL columns before API startup`
- `server/schema.ts` — `verifySchema`
- `server/migrate.ts` — migration history 검사
- `server/migrations/001_monitors.sql`, `002_check_runs.sql`
- 관련 Thread: 05(E05) owner 컬럼, 09~13 후속 제약·인덱스 검증
---

<a id="p09"></a>
## [Thread 03 (E03) / `Make PostgreSQL authoritative for Monitor lifecycle`] 명시적 row mapping과 PostgreSQL을 단일 진실 원천으로 두는 설계

### 면접 질문

- `SELECT *` 결과를 그대로 API에 내보내지 않고 `monitorFromRow`, `checkRunFromRow`를 둔 이유는 무엇입니까?
  - 꼬리 질문: `false`, `0`, `null`, timezone이 포함된 값을 매핑할 때 자주 생기는 버그를 설명해 보세요.
    - 모범답변: `value || default`는 `false`와 `0`을 없애고, nullable timestamp에 무조건 메서드를 호출하면 실패합니다. 원본은 필드를 직접 옮기고 `Date.toISOString()`과 명시적 null 분기로 UTC instant를 보존합니다.
  - 꼬리 질문: Monitor 삭제 시 CheckRun을 애플리케이션 루프로 지우는 대신 FK cascade로 묶은 이유와 단점은 무엇입니까?
    - 모범답변: DB가 부모 삭제와 모든 자식 삭제를 한 transaction에서 보장해 재시작·다중 인스턴스에도 고아 row가 없습니다. 대신 삭제 영향이 schema에 있으므로 migration·catalog 검증과 테스트 없이는 동작을 파악하기 어렵습니다.

### 30초 모범 답변

DB row는 snake_case, `Date`, nullable 컬럼 같은 저장 표현이고 API 모델은 camelCase, UTC ISO 문자열, 명시적 union을 사용하므로 경계에서 정확히 매핑해야 합니다. truthiness 기반 기본값을 쓰면 `false`, `0`, `null`이 다른 값으로 바뀌고, timezone 변환을 생략하면 인스턴스별 결과가 달라집니다. PostgreSQL을 단일 진실 원천으로 두고 FK cascade를 사용하면 재시작과 여러 인스턴스에서도 원자적으로 부모·자식 정합성을 지킬 수 있습니다. 대신 삭제 영향이 DB schema에 숨으므로 제약과 테스트를 함께 관리해야 합니다.

### 답변 핵심 키워드

- explicit mapping
- storage model vs API model
- falsy preservation
- UTC ISO
- single source of truth
- foreign key
- cascade
- restart persistence

### 백지 구현

#### 구현 목표

DB row와 공개 도메인 모델 사이를 왕복하는 명시적 mapper를 작성한다.
#### 인터페이스 또는 함수 시그니처

`monitorFromRow(row): Monitor`, `monitorToValues(model): readonly unknown[]`, `checkRunFromRow(row): CheckRun`
#### 입력과 출력

- 입력: snake_case 필드, Date·number·boolean·null을 포함한 row
- 출력: 허용된 공개 필드만 가진 camelCase 모델과 파라미터 배열
- 실패: 필수 필드나 상태 조합이 잘못된 row에 대한 명시적 mapping 오류
#### 반드시 만족해야 할 조건

- `false`, `0`, `null`을 손실 없이 보존한다.
- timestamp는 millisecond UTC ISO 문자열로 정규화한다.
- DB 전용 필드와 credential·owner 내부 정보는 공개 모델에 섞지 않는다.
- 필드 순서에 의존하는 파라미터 배열은 한곳에서 정의한다.
- 알 수 없는 상태나 모순된 nullable 조합을 조용히 통과시키지 않는다.
#### 경계 조건

- latency 0
- HTTP status `null`
- enabled `false`
- UTC가 아닌 offset을 가진 입력 Date
- 예상 밖 추가 컬럼
#### 실패 조건

- spread 연산으로 row 전체를 API 모델에 노출한다.
- `value || default`로 정상 falsy 값을 바꾼다.
- 로컬 timezone 문자열을 저장·반환한다.
- 부모 삭제와 자식 삭제를 별도 autocommit으로 처리한다.
#### 필요한 제약

- SQL 값은 파라미터 바인딩을 사용한다.
- mapper는 네트워크나 전역 설정에 의존하지 않는 순수 함수로 작성한다.

```ts
export function monitorFromRow(row: MonitorRow): Monitor {
  return {
    id: row.id,
    name: row.name,
    url: row.url,
    interval: row.interval_seconds,
    enabled: row.enabled,
    createdAt: row.created_at.toISOString(),
    updatedAt: row.updated_at.toISOString(),
  };
}

export function monitorToValues(
  monitor: Monitor,
): readonly unknown[] {
  // SQL placeholder 순서의 ownership을 mapper 한곳에 둔다.
  return [
    monitor.id,
    monitor.name,
    monitor.url,
    monitor.interval,
    monitor.enabled,
    new Date(monitor.createdAt),
    new Date(monitor.updatedAt),
  ];
}
```

### 구현 후 자가 검증

- 왕복 후 UUID, 이름, URL, interval, enabled, timestamp가 유지된다.
- `false`, `0`, `null` fixture가 그대로 남는다.
- offset timestamp가 동일한 UTC instant로 변환된다.
- 예상 밖 내부 컬럼이 API 객체에 포함되지 않는다.
- 부모 삭제 뒤 자식이 남지 않고, 실패 시 둘 다 유지된다.
- API 재시작 뒤에도 이전 데이터와 history가 동일하게 조회된다.

### 구현 후 설명할 것

- row와 wire model을 분리한 이유
  - 모범답변: PostgreSQL의 snake_case·`Date`·내부 owner 필드와 공개 API의 camelCase·UTC ISO·허용 필드는 서로 다른 계약입니다. 명시적 mapper가 그 신뢰 경계를 고정합니다.
- 명시적 mapper의 중복 비용과 안정성 이점
  - 모범답변: 필드를 양쪽에 반복 작성하는 비용은 있지만 schema 추가 시 공개 여부와 변환을 의식적으로 결정하게 하고, `SELECT *`의 새 내부 컬럼이 우연히 노출되는 일을 막습니다.
- DB cascade와 애플리케이션 명시 삭제의 trade-off
  - 모범답변: cascade는 원자성과 다중 writer 정합성이 강하지만 영향 범위가 SQL schema에 숨습니다. 애플리케이션 삭제는 절차가 보이지만 중간 실패와 경합을 별도로 처리해야 합니다.
- PostgreSQL을 authoritative state로 정한 뒤 브라우저 cache가 가져야 할 역할
  - 모범답변: 브라우저 cache는 응답성과 렌더링을 위한 파생 복사본일 뿐 진실 원천이 아닙니다. mutation 후 무효화하고 stale 응답을 거부하며 필요하면 DB 기반 API에서 다시 채워야 합니다.

### 원본 확인 위치

- Thread 03
- `server/mapping.ts` — `monitorFromRow`, `checkRunFromRow`, `monitorToValues`, `checkRunToValues`
- `server/app.ts` — PostgreSQL 기반 CRUD·history
- `server/migrations/001_monitors.sql`, `002_check_runs.sql`
- `test/persistence.test.ts` — canonical row round-trip과 mapping 검증
- 관련 Thread: 06(E06) 브라우저 server-state cache
---

<a id="p10"></a>
## [Thread 07 (E07) / 제목 미노출 — 기록상 bounded cursor history 구현] 조건에 결박된 cursor와 안정적인 seek pagination

### 면접 질문

- 이 프로젝트가 OFFSET 대신 `(finished_at, id)` seek cursor를 사용한 이유를 삽입 동시성과 복잡도 관점에서 설명해 보세요.
  - 꼬리 질문: cursor에 monitorId, state, limit을 함께 넣고 continuation 요청과 일치시키는 이유는 무엇입니까?
    - 모범답변: cursor 위치는 해당 필터와 페이지 크기에서만 의미가 있습니다. 조건을 payload에 넣고 요청과 대조해야 다른 Monitor나 필터에 위치를 재사용해 생기는 누락·중복을 막을 수 있습니다.
  - 꼬리 질문: timestamp만 cursor로 쓰지 않고 UUID를 tie-breaker로 둔 이유는 무엇입니까?
    - 모범답변: 여러 row가 같은 millisecond `finished_at`을 가질 수 있어 timestamp만으로는 전체 순서가 아닙니다. UUID를 두 번째 내림차순 키로 사용해 엄격한 seek 경계를 만듭니다.

### 30초 모범 답변

OFFSET은 앞쪽에 새 row가 들어오면 같은 row가 중복되거나 빠질 수 있고, 깊은 페이지일수록 건너뛰는 비용이 커집니다. `(finished_at DESC, id DESC)`의 마지막 tuple보다 작은 row를 찾는 seek 방식은 새로 들어온 더 최신 row의 영향을 받지 않고 인덱스 순서를 그대로 사용할 수 있습니다. timestamp 동률을 UUID로 완전 정렬하고, cursor에 monitor·filter·limit을 넣어 다른 조건에서 재사용하는 것을 거부합니다. cursor는 위치 정보일 뿐 권한 증명이 아니므로 owner predicate는 매 요청 다시 적용합니다.

### 답변 핵심 키워드

- keyset pagination
- seek cursor
- total ordering
- tie-breaker
- condition-bound token
- canonical base64url
- limit+1
- cursor는 권한이 아님

### 백지 구현

#### 구현 목표

history query를 검증하고 마지막 `(finishedAt, id)` 위치를 담은 canonical cursor를 인코딩·디코딩한다.
#### 인터페이스 또는 함수 시그니처

`parseHistoryQuery(monitorId, raw): HistoryQuery`, `encodeHistoryCursor(query, last): string`
#### 입력과 출력

- 입력: monitor ID와 `limit`, `state`, `cursor` query 값
- 출력: 검증된 limit·state·seek tuple 또는 새 cursor 문자열
- 실패: 형식, 길이, 타입, 조건 불일치에 대한 입력 오류
#### 반드시 만족해야 할 조건

- 허용 query key만 받고 limit은 문자열 정수 범위 안이어야 한다.
- state는 지원하는 terminal 상태 또는 없음만 허용한다.
- cursor 길이를 제한하고 canonical base64url인지 round-trip으로 확인한다.
- cursor JSON은 정확한 version·monitorId·limit·state·finishedAt·id를 가져야 한다.
- timestamp는 유효한 millisecond UTC ISO, id는 유효한 UUID여야 한다.
- 페이지 조회는 정렬 tuple과 같은 방향의 엄격한 seek 조건, `limit + 1` look-ahead를 사용한다.
#### 경계 조건

- 동일 timestamp를 가진 여러 row
- limit 1과 최대 limit
- 새 최신 row가 첫 페이지 뒤 삽입되는 경우
- cursor의 state·limit·monitorId를 바꾼 요청
- 패딩된 base64url, 비정규 timestamp, 잘못된 날짜
#### 실패 조건

- cursor payload를 타입 단언만 하고 사용한다.
- timestamp 하나로만 정렬해 동률 순서가 불안정하다.
- cursor를 소유권 토큰으로 간주한다.
- continuation에서 `<=`를 사용해 마지막 row를 중복한다.
#### 필요한 제약

- cursor 최대 길이는 512자, 기본 limit 20, 최대 100이라는 현재 기록의 경계를 사용한다.
- 서명·암호화는 요구하지 않으며, tamper 방지는 엄격한 검증과 owner SQL로 다룬다.

```ts
type HistoryQuery = {
  monitorId: string;
  limit: number;
  state: 'SUCCEEDED' | 'FAILED' | null;
  after: { finishedAt: string; id: string } | null;
};

export function parseHistoryQuery(
  monitorId: string,
  raw: unknown,
): HistoryQuery {
  const invalid = (): never => { throw new ApiError('INVALID_INPUT', 'Invalid history query'); };
  if (raw === null || typeof raw !== 'object' || Array.isArray(raw)) invalid();
  const value = raw as Record<string, unknown>;
  if (Object.keys(value).some(key => !['limit', 'state', 'cursor'].includes(key))) invalid();

  let limit = 20;
  if (value.limit !== undefined) {
    if (typeof value.limit !== 'string' || !/^[0-9]{1,3}$/.test(value.limit)) invalid();
    limit = Number(value.limit);
    if (limit < 1 || limit > 100) invalid();
  }
  const state = value.state ?? null;
  if (state !== null && state !== 'SUCCEEDED' && state !== 'FAILED') invalid();
  const base: HistoryQuery = { monitorId, limit, state, after: null };
  if (value.cursor === undefined) return base;

  const token = value.cursor;
  if (typeof token !== 'string' || token.length > 512 ||
      !/^[A-Za-z0-9_-]+$/.test(token)) invalid();
  const bytes = Buffer.from(token, 'base64url');
  if (bytes.toString('base64url') !== token) invalid();
  let decoded: unknown;
  try { decoded = JSON.parse(bytes.toString('utf8')); }
  catch { invalid(); }
  if (decoded === null || typeof decoded !== 'object' || Array.isArray(decoded)) invalid();
  const cursor = decoded as Record<string, unknown>;
  if (Object.keys(cursor).length !== 6 || cursor.version !== 1 ||
      cursor.monitorId !== monitorId || cursor.limit !== limit || cursor.state !== state ||
      typeof cursor.finishedAt !== 'string' ||
      !/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/.test(cursor.finishedAt) ||
      !Number.isFinite(Date.parse(cursor.finishedAt)) ||
      new Date(cursor.finishedAt).toISOString() !== cursor.finishedAt ||
      typeof cursor.id !== 'string' ||
      !/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/.test(cursor.id)) invalid();
  return { ...base, after: { finishedAt: cursor.finishedAt, id: cursor.id } };
}

export function encodeHistoryCursor(
  query: HistoryQuery,
  last: { finishedAt: string; id: string },
): string {
  return Buffer.from(JSON.stringify({
    version: 1,
    monitorId: query.monitorId,
    state: query.state,
    limit: query.limit,
    finishedAt: last.finishedAt,
    id: last.id,
  })).toString('base64url');
}
```

### 구현 후 자가 검증

- 기본·최소·최대 limit과 허용 state가 정상 처리된다.
- 알 수 없는 key, 배열 query, 숫자형 limit을 거부한다.
- malformed·oversized·noncanonical cursor를 거부한다.
- monitor/state/limit 중 하나라도 바꾸면 같은 cursor를 거부한다.
- 동률 timestamp row가 UUID tie-breaker 순서로 중복·누락 없이 순회된다.
- 첫 페이지 뒤 새 최신 row를 삽입해도 기존 continuation이 그대로 이어진다.
- foreign owner는 cursor가 있어도 404 경계를 넘지 못한다.

### 구현 후 설명할 것

- OFFSET과 keyset pagination의 정확성·성능 차이
  - 모범답변: OFFSET은 앞쪽 삽입으로 page 경계가 이동하고 깊을수록 많은 row를 건너뜁니다. keyset은 마지막 tuple보다 작은 row를 index에서 seek하므로 최신 삽입에 안정적이고 비용이 페이지 깊이에 덜 의존합니다.
- 정렬 열과 index 열 순서를 맞춘 이유
  - 모범답변: SQL의 `monitor_id` equality 뒤 `(finished_at DESC, id DESC)` 정렬·seek 순서와 인덱스를 일치시켜 별도 sort 없이 연속 범위를 읽게 합니다.
- cursor에 조건을 포함한 이유와 서명하지 않은 trade-off
  - 모범답변: monitor·state·limit 결박으로 우발적 재사용은 막지만 서명이 없어 클라이언트가 payload를 만들 수 있습니다. 그래서 모든 값을 엄격히 검증하고 owner SQL을 매번 다시 적용하며 cursor를 권한으로 보지 않습니다.
- 삽입에는 안정적이지만 삭제·row 변경에 snapshot을 보장하지 않는 한계
  - 모범답변: 새 최신 row는 기존 continuation 앞에 생겨 영향을 주지 않지만 이미 본 row의 삭제나 정렬 키 변경은 전체 시점 snapshot을 보장하지 않습니다. snapshot이 필요하면 별도 기준 시각이나 DB snapshot 전략이 필요합니다.

### 원본 확인 위치

- Thread 07
- `server/history.ts` — `historyQuery`, `historyCursor`, limit 상수
- `server/app.ts` — owner-scoped history seek query
- `server/migrations/005_check_history_index.sql`
- `test/unit.test.ts`, `test/contracts.test.ts`, `test/browser/history.spec.ts`
- 관련 Thread: 05(E05) owner predicate, 13(E20) 부분 인덱스
---

<a id="p11"></a>
## [Thread 13 (내부 E20) / `perf: index sparse failed history without changing pagination`] 실제 query plan으로 부분 인덱스를 선택하는 판단

### 면접 질문

- 기존 `(monitor_id, finished_at DESC, id DESC)` 인덱스가 있는데 FAILED history용 부분 인덱스를 추가한 이유는 무엇입니까?
  - 꼬리 질문: rows removed by filter와 shared buffer hit/read가 줄었다는 결과를 어떻게 해석하며, 무엇을 latency 개선으로 과장하면 안 됩니까?
    - 모범답변: FAILED 후보를 찾기 위해 버린 SUCCEEDED row와 접근 page가 줄었다는 구조적 근거입니다. 고정 fixture의 한 plan 관찰일 뿐 실제 production 지연·I/O·p95 개선률로 일반화하면 안 됩니다.
  - 꼬리 질문: parameterized query에서 partial index가 선택된 결과를 generic prepared plan 전체에 일반화할 수 없는 이유는 무엇입니까?
    - 모범답변: partial index는 planner가 `state = 'FAILED'`가 predicate를 함의한다고 증명할 때만 쓸 수 있습니다. generic plan은 파라미터 값을 모를 수 있어 같은 SQL도 전체 인덱스를 선택할 수 있습니다.

### 30초 모범 답변

전체 history 인덱스는 정렬에는 맞지만 FAILED가 1%인 skewed 데이터에서 원하는 21개를 얻기 전에 많은 성공 row를 필터링합니다. 같은 정렬 열에 `state='FAILED'` 조건을 둔 부분 인덱스는 후보 집합 자체를 줄여 rows removed와 buffer 접근을 낮춥니다. 다만 한 번의 고정 데이터·ANALYZE에서 본 plan과 buffer 수치이지 지연 시간 일반화를 뜻하지 않습니다. 또한 실제 unnamed parameterized query에서 predicate가 증명돼 선택된 것이므로, generic prepared plan처럼 planner가 파라미터 값을 모르는 상황까지 보장하면 안 됩니다.

### 답변 핵심 키워드

- EXPLAIN ANALYZE
- partial index
- data skew
- predicate selectivity
- rows removed by filter
- buffer hits/reads
- no sort
- generic vs custom plan

### 백지 구현

#### 구현 목표

주어진 history query와 데이터 분포를 보고 후보 인덱스 하나를 제안하고, 같은 결과·pagination 의미를 유지하는 검증 계획을 작성한다.
#### 인터페이스 또는 함수 시그니처

코드 함수가 아니라 `CREATE INDEX` 후보 1개와 `EXPLAIN (ANALYZE, BUFFERS)` 비교 계획을 작성한다. SQL 정답을 복사하는 문제가 아니라, 주어진 query·분포·plan 근거로 하나의 가설을 세우는 축소 설계 문제다.

```sql
-- 후보 인덱스 DDL 1개를 작성한다.
-- 적용 전·후에 실행할 EXPLAIN 비교 문장을 작성한다.
-- 기존 전체 history 인덱스는 All/SUCCEEDED 조회를 위해 유지한다.
CREATE INDEX check_runs_failed_history_idx
  ON check_runs (monitor_id, finished_at DESC, id DESC)
  WHERE state = 'FAILED' AND finished_at IS NOT NULL;

ANALYZE check_runs;

EXPLAIN (ANALYZE, BUFFERS)
SELECT c.*
FROM check_runs c
JOIN monitors m ON m.id = c.monitor_id
WHERE m.id = $1
  AND m.owner_user_id = $2
  AND c.finished_at IS NOT NULL
  AND c.state = 'FAILED'
  AND ($3::timestamptz IS NULL
       OR (c.finished_at, c.id) < ($3::timestamptz, $4::uuid))
ORDER BY c.finished_at DESC, c.id DESC
LIMIT 21;
```

#### 입력과 출력

- 입력: owner-scoped monitor history query, `state` 선택 filter, `(finished_at, id)` 내림차순, limit+1, FAILED 1%의 고정 분포
- 출력: 후보 DDL, 적용 전·후 확인 지표, 결과 동등성 검증 항목
#### 반드시 만족해야 할 조건

- 현재 정렬과 seek 조건을 깨지 않는다.
- All history와 다른 상태 query가 사용할 기존 인덱스를 제거하지 않는다.
- 인덱스 predicate가 실제 query predicate와 논리적으로 맞아야 한다.
- 첫 페이지와 continuation의 row ID·nextCursor 의미가 전후 동일해야 한다.
- plan 선택뿐 아니라 rows removed, buffers, sort 여부를 비교한다.
#### 경계 조건

- FAILED 비율이 크게 높아지는 데이터
- 특정 monitor만 극단적으로 큰 skew
- state 파라미터가 runtime에 알려지지 않는 generic plan
- 통계가 오래된 상태
#### 실패 조건

- 무조건적인 복합 인덱스를 추가해 write·storage 비용을 무시한다.
- 한 번의 wall-clock 결과만으로 개선을 단정한다.
- 결과 row나 cursor 의미가 달라졌는데 plan만 빠르다고 판단한다.
#### 필요한 제약

- 프로젝트 기록의 99,000 terminal row 고정 dataset을 기준으로 설명한다.
- 여러 후보를 무작정 탐색하지 않고 가설 하나와 반증 조건을 명시한다.

### 구현 후 자가 검증

- 후보 index 열 순서가 WHERE·ORDER BY·seek tuple과 맞는다.
- 부분 predicate가 FAILED query에서만 적용된다.
- 적용 전·후 반환 ID와 cursor가 동일하다.
- rows removed by filter, buffer hit/read, sort 유무를 기록한다.
- All/SUCCEEDED query의 plan과 write 비용이 악화될 가능성을 검토한다.
- 통계와 plan 종류를 명시하고 latency 일반화를 피한다.

### 구현 후 설명할 것

- 전체 인덱스와 부분 인덱스를 함께 유지한 이유
  - 모범답변: 부분 인덱스는 FAILED query에만 적용되므로 All과 SUCCEEDED history에는 기존 전체 인덱스가 필요합니다. 새 인덱스는 기존 pagination 계약을 대체하지 않고 희소 필터만 보완합니다.
- 데이터 분포가 index 가치에 미치는 영향
  - 모범답변: FAILED가 1%이면 작은 부분 인덱스로 후보를 빠르게 찾지만 비율이 높아지면 크기·선택도 이점이 줄고 쓰기 비용만 남을 수 있습니다. 실제 분포와 monitor별 skew를 함께 봐야 합니다.
- buffer 지표와 wall-clock latency의 차이
  - 모범답변: buffer hit/read는 접근한 page와 캐시 상태를 설명하지만 CPU 경쟁·스토리지·네트워크·측정 잡음을 모두 나타내지 않습니다. 한 번의 실행 시간보다 plan 구조와 반복 측정을 함께 해석해야 합니다.
- prepared statement plan mode가 partial index 선택에 미치는 영향
  - 모범답변: custom plan은 실제 `FAILED` 값을 알아 predicate를 증명할 수 있지만 generic plan은 이를 모를 수 있습니다. 원본에서 관찰한 unnamed parameterized 실행을 모든 prepared 실행 모드의 보장으로 확대하면 안 됩니다.

### 원본 확인 위치

- Thread 13 — 파일명 기준, 내부 기록 식별자는 E20
- `server/app.ts` — 실제 history query
- `server/migrations/010_failed_history_index.sql`
- `13-checkrun-query-plans-and-index-selection-01.md`
- 관련 Thread: 07(E07) cursor pagination과 기본 history 인덱스
