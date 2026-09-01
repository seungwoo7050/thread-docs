# 데이터 정합성·트랜잭션·파괴적 작업 경계 면접 워크북

이 문서는 애플리케이션 코드만으로 지키기 어려운 불변식을 데이터베이스 제약, 트랜잭션, 멱등성 키, 안전장치와 함께 보장하는 문제를 다룬다.

---

## IM-06. [Thread 04 / `feat(db): test database reset target guard 추가`, `feat(db): test schema reset과 migration 실행 연결`] 파괴적 테스트 DB 작업의 다중 안전장치

### 면접 질문

테스트 데이터베이스를 초기화하는 CLI에서 `NODE_ENV=test` 확인만으로는 왜 부족한가요? 데이터베이스 이름, `search_path`, URL 옵션까지 검증해 파괴 범위를 제한한 설계를 설명해 주세요.

꼬리 질문:

- 운영 DB URL이 실수로 주입되었을 때 fail-closed 하려면 어떤 조건을 동시에 확인해야 합니까?
  - 모범답변: `NODE_ENV === "test"`, PostgreSQL 프로토콜, 해석 가능한 단일 DB 이름, 전용 테스트 DB 이름 또는 엄격한 격리 schema, 정확히 하나 이하의 허용된 `options` 형식을 모두 확인합니다. 하나라도 빠지거나 애매하면 동일한 안전 오류로 중단합니다.
- SQL 식별자는 값 파라미터처럼 바인딩할 수 없는데, 스키마 이름을 어떻게 안전하게 다루겠습니까?
  - 모범답변: 외부 문자열을 일반화해 escape하기보다 `test_[a-f0-9]{32}` 허용 정규식으로 좁힌 뒤 큰따옴표 식별자로 사용합니다. 프로젝트의 reset 함수도 검증된 schema만 `"schema"` 형태로 drop/create 문에 넣습니다.
- 스키마 삭제·생성은 왜 트랜잭션으로 묶고, migration은 왜 그 뒤 별도 단계로 실행할 수 있습니까?
  - 모범답변: drop과 create 사이 실패로 schema가 사라진 채 남지 않도록 두 DDL을 한 트랜잭션에 둡니다. migration runner는 자체 실행 경계와 연결을 관리하므로 새 schema가 commit된 뒤 적용해 현재 migration 집합을 갖춘 테스트 DB로 만듭니다.
- reset 실패와 cleanup 실패가 동시에 발생하면 어떤 오류를 보존해야 합니까?
  - 모범답변: 본래 reset 실패를 주 오류로 보존하고 rollback·pool 종료 실패는 cause나 `AggregateError`의 보조 오류로 남겨야 합니다. 현재 코드는 rollback 실패를 명시적으로 억제해 원 오류를 보존하지만, 더 엄격한 구현이라면 pool 종료 오류도 함께 관측해야 합니다.

### 30초 모범 답변

파괴적 명령은 한 가지 환경 변수에만 기대면 오설정 한 번으로 운영 데이터를 지울 수 있습니다. 그래서 테스트 런타임인지, 전용 테스트 DB 이름인지 또는 엄격한 형식의 격리 스키마인지, `search_path` 옵션이 단 하나인지까지 모두 확인하고 하나라도 모호하면 중단해야 합니다. 검증된 스키마만 식별자로 인용하고, 삭제와 재생성은 트랜잭션으로 묶습니다. 이후 migration을 다시 적용해 빈 상태가 아니라 현재 스키마 계약을 만족하는 테스트 환경으로 복구합니다.

### 답변 핵심 키워드

파괴적 명령 · fail-closed · 다중 가드 · 전용 DB · 격리 스키마 · 식별자 인용 · 트랜잭션 · migration 재적용 · cleanup 오류 보존

### 백지 구현

**구현 목표**

환경 변수로 받은 PostgreSQL URL이 테스트 초기화 대상으로 안전한지 판정한다. 실제 삭제 코드는 작성하지 않고, 허용 대상의 데이터베이스 이름과 스키마만 반환한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface ResetTarget {
  databaseUrl: string;
  databaseName: string;
  schema: string;
}

export function resolveSafeResetTarget(
  env: Record<string, string | undefined>
): ResetTarget {
  const unsafe = (): never => {
    // 자격 증명이 포함될 수 있는 원본 URL은 오류 메시지에 넣지 않는다.
    throw new Error("Unsafe test reset target");
  };
  if (env.NODE_ENV !== "test" || !env.TEST_DATABASE_URL) unsafe();

  let url: URL;
  try {
    url = new URL(env.TEST_DATABASE_URL);
  } catch {
    return unsafe();
  }
  if (url.protocol !== "postgres:" && url.protocol !== "postgresql:") unsafe();

  let databaseName: string;
  try {
    databaseName = decodeURIComponent(url.pathname.slice(1));
  } catch {
    return unsafe();
  }
  if (!databaseName || databaseName.includes("/")) unsafe();

  const optionValues = url.searchParams.getAll("options");
  if (optionValues.length > 1) unsafe();
  let schema = "public";
  if (optionValues.length === 1) {
    const match = /^-c search_path=(test_[a-f0-9]{32})$/.exec(optionValues[0]);
    if (!match) unsafe();
    schema = match[1];
  }

  const dedicatedDatabase = /^(?:test(?:_[a-z0-9][a-z0-9_-]*)?|[a-z0-9][a-z0-9_-]*_test)$/;
  const isolatedSchema = /^test_[a-f0-9]{32}$/;
  if (schema === "public" && !dedicatedDatabase.test(databaseName)) unsafe();
  if (schema !== "public" && !isolatedSchema.test(schema)) unsafe();

  return { databaseUrl: env.TEST_DATABASE_URL, databaseName, schema };
}
```

**입력과 출력**

- 입력: `NODE_ENV`, `TEST_DATABASE_URL`
- 출력: 검증된 URL, 데이터베이스 이름, 초기화할 스키마
- 거부 시에는 하나의 안전한 오류를 던진다.

**반드시 만족해야 할 조건**

- `NODE_ENV`가 정확히 `test`여야 한다.
- URL 프로토콜은 PostgreSQL 계열만 허용한다.
- URL 경로에서 데이터베이스 이름을 안전하게 해석할 수 있어야 한다.
- `public` 스키마를 허용하려면 데이터베이스 이름이 명확한 테스트 전용 규칙을 만족해야 한다.
- 일반 데이터베이스를 쓸 때는 자동 생성된 격리 스키마 이름만 허용한다.
- `options`가 여러 개이거나 `search_path` 외 설정이 섞이면 거부한다.
- 검증이 애매한 경우 허용하지 않는다.

**경계 조건**

- URL 디코딩에 실패하는 데이터베이스 이름
- 빈 데이터베이스 경로
- `public,other`처럼 여러 스키마가 지정된 경우
- 이름에 `test`가 들어가지만 테스트 전용 규칙에는 맞지 않는 경우
- 대소문자 또는 특수 문자가 섞인 스키마 이름

**실패 조건**

- 필수 환경 변수가 없음
- URL 형식 또는 프로토콜이 잘못됨
- 일반 DB의 `public` 스키마를 가리킴
- 생성 규칙과 맞지 않는 임의 스키마를 가리킴

**필요한 제약**

- 허용 목록 방식으로 구현한다.
- 호출자가 임의 식별자를 SQL 문자열에 삽입하지 못하도록 출력 규칙을 좁게 유지한다.
- 오류 메시지에 비밀번호가 포함된 전체 URL을 출력하지 않는다.

### 구현 후 자가 검증

- [ ] 전용 테스트 DB의 `public` 스키마만 허용된다.
- [ ] 일반 DB는 엄격한 형식의 격리 스키마일 때만 허용된다.
- [ ] 운영 DB와 애매한 이름의 DB는 모두 거부된다.
- [ ] 복수 `options`, 복수 스키마, 다른 PostgreSQL 옵션이 거부된다.
- [ ] 잘못된 URL을 넣어도 비밀번호가 오류에 노출되지 않는다.
- [ ] 모든 불명확한 입력이 허용이 아니라 거부로 끝난다.

### 구현 후 설명할 것

1. 환경 변수 하나가 아니라 서로 독립적인 여러 가드를 둔 이유
   - 모범답변: `NODE_ENV`나 변수 이름은 배포 설정 실수로 쉽게 잘못 주입됩니다. 런타임 모드, 프로토콜, DB 이름, schema 생성 규칙을 독립적으로 모두 만족시켜 한 번의 오설정이 파괴 권한으로 이어지지 않게 합니다.
2. 데이터 값과 SQL 식별자의 안전한 처리 방식 차이
   - 모범답변: 값은 query parameter로 바인딩해 DB driver가 escape하게 할 수 있지만 schema·table 식별자는 같은 방식으로 바인딩할 수 없습니다. 따라서 식별자는 좁은 허용 목록으로 검증한 뒤 안전하게 quote합니다.
3. 스키마 단위 격리가 테스트 병렬성에 주는 이점
   - 모범답변: 각 실행이 `test_<32 hex>` schema와 `search_path`를 쓰면 같은 DB 서버에서도 reset 범위와 migration 객체가 분리됩니다. 테스트 worker끼리 public schema를 drop하거나 데이터를 지우는 충돌을 줄입니다.
4. reset 트랜잭션과 migration 단계를 분리한 이유
   - 모범답변: reset 트랜잭션은 drop/create의 원자성만 책임지고 commit합니다. 그 다음 기존 migration runner가 새 연결과 자체 순서로 전체 migration을 적용하게 해 DDL 초기화와 schema 진화 책임을 분리합니다.
5. 본래 실패와 cleanup 실패를 함께 보존하는 오류 정책
   - 모범답변: reset 오류가 원인이므로 이를 primary로 유지하고 cleanup 오류는 cause·구조화 로그·`AggregateError`로 함께 남깁니다. cleanup의 새 예외가 원래 stack을 덮어쓰면 진단 방향이 바뀌기 때문입니다.

### 원본 확인 위치

- Thread 04
- 커밋: `feat(db): test database reset target guard 추가`
- 커밋: `feat(db): test schema reset과 migration 실행 연결`
- 커밋: `test(db): test database reset guard 검증`
- `packages/db/src/testReset.ts`
  - `resolveTestResetTarget`
  - `resetTestDatabase`
- `packages/db/src/testReset.test.ts`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 21, 23

---

## IM-07. [Thread 15 / `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`, `refactor(game): 경기 결과 확정 boundary 사용`] 멱등한 경기 확정과 원자적 부수 효과

### 면접 질문

경기 종료 저장을 단순 `INSERT`가 아니라 `resultKey` 기반의 멱등한 확정 명령으로 만든 이유는 무엇인가요? 경기 행, 승패·레이팅, 레이팅 이력, 토너먼트 진행을 한 트랜잭션에서 처리해야 하는 이유도 설명해 주세요.

꼬리 질문:

- 네트워크 타임아웃 뒤 같은 요청을 재시도했을 때 새 경기인지 기존 경기인지 어떻게 구분합니까?
  - 모범답변: 방 종료 사실에서 만든 안정적인 `resultKey`로 insert를 시도하고, 최초 insert가 행을 반환하면 새 경기, unique conflict로 반환 행이 없으면 기존 `resultKey`의 경기 ID를 조회합니다. 응답의 `created`가 그 차이를 전달합니다.
- 애플리케이션에서 먼저 조회한 뒤 삽입하는 방식은 왜 경쟁 상태를 막지 못합니까?
  - 모범답변: 두 트랜잭션이 동시에 "없음"을 읽은 뒤 둘 다 insert할 수 있기 때문입니다. `matches.result_key`의 unique constraint와 `ON CONFLICT DO NOTHING`이 최종 직렬화 지점이어야 합니다.
- 중복 요청이 들어왔을 때 이전 결과와 입력 내용이 충돌한다면 어떤 정책이 필요합니까?
  - 모범답변: 멱등성 키가 같은 요청은 같은 명령이어야 한다는 계약을 두고, payload fingerprint를 저장·비교해 다르면 conflict로 거부하는 것이 안전합니다. 현재 원본은 기존 경기 ID를 그대로 반환하므로 호출자가 안정적인 방 종료 명령을 재사용한다는 전제에 의존합니다.
- 트랜잭션 안에서 참가자 행을 어떤 순서로 잠그는 것이 좋습니까?
  - 모범답변: winner·loser ID에서 null과 중복을 제거한 뒤 문자열 정렬 순서로 `FOR UPDATE`해야 합니다. 반대 참가자 조합의 동시 경기 확정도 같은 잠금 순서를 사용해 교착 가능성을 낮춥니다.

### 30초 모범 답변

경기 종료는 저장 성공 여부를 클라이언트나 프로세스가 확실히 알지 못한 채 재시도될 수 있으므로 자연스러운 멱등성 키가 필요합니다. `resultKey`에 유일 제약을 두고 삽입 결과로 최초 생성 여부를 판정하면 동시 요청에서도 한 건만 생성됩니다. 경기 기록, 사용자 통계, 레이팅 이력, 토너먼트 상태는 모두 같은 사실의 파생 상태이므로 하나라도 실패하면 전부 롤백해야 합니다. 여러 사용자 행을 잠글 때는 일관된 순서를 사용해 교착 가능성도 줄입니다.

### 답변 핵심 키워드

멱등성 키 · unique constraint · 원자성 · 재시도 · 파생 상태 · 트랜잭션 · 행 잠금 순서 · created 플래그 · 충돌 정책

### 백지 구현

**구현 목표**

메모리 저장소에서 동일한 `resultKey`를 가진 경기 확정 명령을 여러 번 호출해도 경기와 사용자 통계가 한 번만 반영되도록 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
import { randomUUID } from "node:crypto";

type MatchMode = "queue" | "ai" | "tournament";

export interface FinalizeMatchCommand {
  resultKey: string;
  mode: MatchMode;
  winnerId: string | null;
  loserId: string | null;
  scoreLeft: number;
  scoreRight: number;
}

export interface FinalizeMatchResult {
  matchId: string;
  resultKey: string;
  created: boolean;
}

export class InMemoryMatchStore {
  private matches = new Map<string, { id: string; command: FinalizeMatchCommand }>();
  private users = new Map<string, { wins: number; losses: number; rating: number }>();

  constructor(
    initialUsers: Record<string, { wins: number; losses: number; rating: number }>,
    private readonly createId: () => string = randomUUID,
    private readonly beforeCommit: () => void = () => undefined
  ) {
    for (const [id, user] of Object.entries(initialUsers)) this.users.set(id, { ...user });
  }

  finalize(command: FinalizeMatchCommand): FinalizeMatchResult {
    if (!command.resultKey.trim() || command.resultKey.length > 200) {
      throw new Error("invalid match result key");
    }
    if (command.winnerId && command.winnerId === command.loserId) {
      throw new Error("match participants must be different");
    }
    if (!Number.isSafeInteger(command.scoreLeft) || command.scoreLeft < 0) {
      throw new Error("invalid left score");
    }
    if (!Number.isSafeInteger(command.scoreRight) || command.scoreRight < 0) {
      throw new Error("invalid right score");
    }

    const existing = this.matches.get(command.resultKey);
    if (existing) {
      const samePayload = JSON.stringify(existing.command) === JSON.stringify(command);
      if (!samePayload) throw new Error("result key conflicts with another match result");
      return { matchId: existing.id, resultKey: command.resultKey, created: false };
    }

    const nextUsers = new Map(
      [...this.users].map(([id, user]) => [id, { ...user }] as const)
    );
    const winner = command.winnerId ? nextUsers.get(command.winnerId) : undefined;
    const loser = command.loserId ? nextUsers.get(command.loserId) : undefined;
    if (command.winnerId && !winner) throw new Error("winner not found");
    if (command.loserId && !loser) throw new Error("loser not found");

    if (winner) {
      winner.wins += 1;
      winner.rating += 16;
    }
    if (loser) {
      loser.losses += 1;
      loser.rating = Math.max(800, loser.rating - 12);
    }

    const matchId = this.createId();
    const nextMatches = new Map(this.matches);
    nextMatches.set(command.resultKey, { id: matchId, command: { ...command } });
    this.beforeCommit();
    // 준비한 두 snapshot을 마지막에 교체해 중간 예외의 부분 반영을 막는다.
    this.users = nextUsers;
    this.matches = nextMatches;
    return { matchId, resultKey: command.resultKey, created: true };
  }
}
```

**입력과 출력**

- 입력: 결과 키, 모드, 승자·패자, 점수
- 출력: 안정적인 경기 ID와 최초 생성 여부

**반드시 만족해야 할 조건**

- 같은 `resultKey`의 두 번째 호출은 기존 경기 ID와 `created: false`를 반환한다.
- 중복 호출은 승리·패배·레이팅을 다시 변경하지 않는다.
- 승자와 패자는 같은 사용자일 수 없다.
- 점수는 0 이상의 정수여야 한다.
- 명령 검증이 실패하면 어떤 상태도 변경되지 않는다.
- 상태 변경 도중 실패를 주입해도 부분 반영이 남지 않도록 구조화한다.

**경계 조건**

- 승자 또는 패자가 `null`인 경기
- 0 대 0, 큰 정수 점수
- 존재하지 않는 참가자
- 동일 키에 서로 다른 명령 내용이 재전송된 경우
- 거의 동시에 같은 키로 호출되는 상황을 어떻게 확장할지 설명

**실패 조건**

- 참가자 조회 실패
- 통계 변경 중 예외
- 결과 키 충돌인데 기존 내용과 새 내용이 모순됨

**필요한 제약**

- 원본 상태를 복사해 커밋하거나 undo 가능한 방식으로 작성한다.
- 구현 크기는 메모리 저장소에 한정하되, 데이터베이스로 옮길 때 필요한 제약을 설명한다.

### 구현 후 자가 검증

- [ ] 최초 호출만 새 경기와 통계를 만든다.
- [ ] 순차 중복과 병렬 중복을 가정한 invariant를 설명할 수 있다.
- [ ] 잘못된 점수나 동일 참가자는 상태 변경 전에 거부된다.
- [ ] 중간 실패 뒤 경기와 사용자 통계가 모두 이전 상태다.
- [ ] 동일 키·다른 내용 충돌 정책이 명확하다.
- [ ] 결과 키 조회와 통계 변경의 시간·공간 복잡도를 설명할 수 있다.

### 구현 후 설명할 것

1. 단순 재시도와 멱등한 재시도의 차이
   - 모범답변: 단순 재시도는 같은 명령을 다시 실행해 통계와 rating을 중복 변경할 수 있습니다. 멱등 재시도는 같은 `resultKey`를 기존 결과로 수렴시켜 외부 관측 결과와 부수 효과가 한 번만 생깁니다.
2. 애플리케이션 선조회보다 데이터베이스 유일 제약이 강한 이유
   - 모범답변: 선조회와 insert 사이에는 다른 트랜잭션이 끼어들 수 있지만 unique index는 모든 writer가 통과해야 하는 DB의 단일 invariant입니다. 원본은 conflict insert 결과로 최초 생성 여부를 판정합니다.
3. 한 트랜잭션에 포함해야 하는 상태와 포함하지 말아야 할 외부 I/O
   - 모범답변: 경기 행, 사용자 승패·rating, rating history, tournament match/final 전이는 같은 종료 사실의 파생 상태라 함께 commit합니다. WebSocket broadcast나 외부 API 호출은 잠금을 길게 잡고 DB rollback로 되돌릴 수도 없으므로 commit 뒤 처리합니다.
4. `created`를 호출자에게 반환하는 이유
   - 모범답변: GameHub와 관측 계층이 최초 확정과 중복 재시도를 구분하면서도 같은 match ID를 사용할 수 있습니다. 중복이면 알림·메트릭 같은 후속 효과를 다시 만들지 결정할 근거가 됩니다.
5. 동일 키에 상충하는 payload가 들어왔을 때의 선택지
   - 모범답변: 가장 안전한 선택은 저장된 payload hash와 비교해 conflict를 반환하는 것입니다. 현재 원본처럼 기존 결과를 그대로 반환하는 단순 정책은 `resultKey` 생성자가 같은 방 종료 내용을 절대 바꾸지 않는다는 더 강한 호출자 계약이 필요합니다.

### 원본 확인 위치

- Thread 15
- 커밋: `feat(db): 경기 확정 command 계약 정의`
- 커밋: `feat(db): PostgreSQL 경기 결과 중복 생성을 차단`
- 커밋: `refactor(game): 경기 결과 확정 boundary 사용`
- `packages/db/migrations/003_match_finalization.sql`
- `packages/db/src/index.ts`
  - `FinalizeMatchCommand`
  - `FinalizeMatchResult`
  - `finalizeMatch`
  - `assertFinalizeMatchCommand`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 07, 13, 16, 21, 22

---

## IM-08. [Thread 17 / `feat(db): memory friendship invariant 적용`, `test(db): friendship와 tournament 경쟁 상태 검증`] 방향 없는 관계의 정규화와 멱등 상태 전이

### 면접 질문

친구 관계는 요청자와 수신자의 방향이 있지만, 두 사용자 사이에는 논리적으로 하나만 존재해야 합니다. 같은 방향 재요청과 반대 방향 동시 요청을 하나의 관계로 수렴시키려면 어떤 invariant와 데이터 모델이 필요합니까?

꼬리 질문:

- `(A, B)`와 `(B, A)`를 모두 막는 유일 제약은 어떻게 표현할 수 있습니까?
  - 모범답변: PostgreSQL에서 `least(requester_id, addressee_id), greatest(...)` 표현식에 unique index를 둡니다. 별도의 `requester_id <> addressee_id` check로 자기 관계도 차단합니다.
- 반대 방향의 pending 요청이 들어오면 자동 수락하는 정책의 장단점은 무엇입니까?
  - 모범답변: 양쪽이 서로 요청한 순간 같은 행을 accepted로 바꿔 추가 승인 단계를 줄이고 자연스럽게 수렴합니다. 반면 명시적 수락 기록이 없고 사용자가 pending을 취소하려던 시점과 경쟁할 수 있어 제품 정책과 감사 요구를 먼저 정해야 합니다.
- 메모리 저장소와 PostgreSQL 구현이 같은 의미를 갖는지 어떤 계약 테스트로 확인하겠습니까?
  - 모범답변: 자기 요청 거부, 같은 방향 반복, 반대 방향 pending의 accepted 전이, 수신자 전용 수락, 양쪽 목록 결과를 같은 테스트 suite로 두 저장소에 실행합니다. PostgreSQL에는 반대 방향 동시 요청을 `Promise.all`로 추가해 행이 하나인지도 확인합니다.
- 수락 API에서 요청자도 수락할 수 있는 권한 버그를 어떻게 막습니까?
  - 모범답변: update 조건에 friendship ID뿐 아니라 `addressee_id = 현재 사용자`를 포함하고 반환 행이 없으면 not found/forbidden으로 처리합니다. 애플리케이션에서 읽고 비교하는 것보다 DB mutation 조건에 권한을 결합하는 편이 경쟁에도 안전합니다.

### 30초 모범 답변

친구 관계의 핵심 invariant는 두 사용자 쌍마다 행이 최대 하나이고 자기 자신과는 관계를 만들 수 없다는 것입니다. 저장 시 두 ID를 정렬한 canonical pair를 별도 키로 사용하거나, 데이터베이스에서 정규화된 표현에 유일 제약을 둡니다. 같은 방향 재요청은 기존 상태를 그대로 반환하고, 반대 방향 pending은 정책에 따라 같은 행을 accepted로 전이시킵니다. 수락 권한은 현재 사용자가 실제 addressee인지 확인해야 합니다.

### 답변 핵심 키워드

canonical pair · 대칭 관계 · unique constraint · 자기 관계 금지 · 멱등 요청 · 상태 전이 · 권한 검증 · 저장소 parity

### 백지 구현

**구현 목표**

두 사용자 사이의 대칭 관계를 하나의 레코드로 유지하는 메모리 친구 저장소를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
import { randomUUID } from "node:crypto";

type FriendshipStatus = "pending" | "accepted";

export interface Friendship {
  id: string;
  requesterId: string;
  addresseeId: string;
  status: FriendshipStatus;
}

export class FriendshipStore {
  private readonly byId = new Map<string, Friendship>();
  private readonly idByPair = new Map<string, string>();
  private readonly idsByUser = new Map<string, Set<string>>();

  constructor(private readonly createId: () => string = randomUUID) {}

  request(requesterId: string, addresseeId: string): Friendship {
    if (requesterId === addresseeId) throw new Error("cannot friend yourself");
    const pairKey = this.pairKey(requesterId, addresseeId);
    const existingId = this.idByPair.get(pairKey);
    const existing = existingId ? this.byId.get(existingId) : undefined;
    if (existing) {
      if (
        existing.status === "pending"
        && existing.requesterId === addresseeId
        && existing.addresseeId === requesterId
      ) {
        existing.status = "accepted";
      }
      return { ...existing };
    }

    const friendship: Friendship = {
      id: this.createId(),
      requesterId,
      addresseeId,
      status: "pending"
    };
    this.byId.set(friendship.id, friendship);
    this.idByPair.set(pairKey, friendship.id);
    this.addUserIndex(requesterId, friendship.id);
    this.addUserIndex(addresseeId, friendship.id);
    return { ...friendship };
  }

  accept(userId: string, friendshipId: string): Friendship {
    const friendship = this.byId.get(friendshipId);
    if (!friendship || friendship.addresseeId !== userId) {
      throw new Error("friendship not found");
    }
    friendship.status = "accepted";
    return { ...friendship };
  }

  listFor(userId: string): Friendship[] {
    return [...(this.idsByUser.get(userId) ?? [])]
      .map((id) => this.byId.get(id))
      .filter((friendship): friendship is Friendship => friendship !== undefined)
      .map((friendship) => ({ ...friendship }));
  }

  private pairKey(left: string, right: string): string {
    return left < right ? `${left}\u0000${right}` : `${right}\u0000${left}`;
  }

  private addUserIndex(userId: string, friendshipId: string): void {
    const ids = this.idsByUser.get(userId) ?? new Set<string>();
    ids.add(friendshipId);
    this.idsByUser.set(userId, ids);
  }
}
```

**입력과 출력**

- 입력: 요청자·수신자 ID, 관계 ID, 현재 사용자 ID
- 출력: 생성되거나 전이된 하나의 친구 관계

**반드시 만족해야 할 조건**

- 자기 자신에게 요청할 수 없다.
- 두 사용자 쌍에는 방향과 무관하게 관계가 하나만 존재한다.
- 같은 방향 재요청은 새 ID를 만들지 않는다.
- 반대 방향 pending 요청은 같은 레코드에서 정해진 정책대로 전이한다.
- 수신자만 pending 관계를 명시적으로 수락할 수 있다.
- 양쪽 사용자의 목록에서 같은 관계 ID와 상대 사용자가 보인다.

**경계 조건**

- 이미 accepted인 관계에 대한 재요청
- 없는 관계 수락
- 요청자가 자기 요청을 수락하려는 경우
- 사용자 ID의 사전순과 실제 요청 방향이 반대인 경우

**실패 조건**

- 자기 관계
- 권한 없는 수락
- 존재하지 않는 사용자 또는 관계

**필요한 제약**

- 쌍 조회가 선형 검색이 되지 않도록 보조 키를 설계한다.
- ID 생성기는 주입하거나 테스트에서 고정할 수 있어야 한다.

### 구현 후 자가 검증

- [ ] `(A, B)`와 `(B, A)`가 같은 논리 키로 수렴한다.
- [ ] 반복 요청이 새 레코드를 만들지 않는다.
- [ ] 반대 방향 pending 전이가 정확히 한 번 일어난다.
- [ ] 잘못된 사용자가 수락할 수 없다.
- [ ] 양쪽 목록의 상대 사용자와 상태가 일관된다.
- [ ] 쌍 조회의 평균 시간 복잡도를 설명할 수 있다.

### 구현 후 설명할 것

1. 요청 방향과 관계 정체성을 분리한 이유
   - 모범답변: requester/addressee는 pending을 누가 수락할지 정하는 상태 전이 정보이고, 관계 정체성은 정렬된 두 사용자 쌍입니다. 둘을 분리해야 방향을 보존하면서도 `(A,B)`와 `(B,A)`가 한 행으로 수렴합니다.
2. canonical pair를 애플리케이션과 DB 중 어디에서 강제할지
   - 모범답변: 애플리케이션 보조 key는 빠른 조회와 명확한 의도를 제공하지만 모든 writer를 통제하지 못합니다. 최종 invariant는 원본처럼 DB의 least/greatest unique index와 distinct-users check로 강제해야 합니다.
3. 자동 수락 정책과 명시적 수락 정책의 trade-off
   - 모범답변: 자동 수락은 쌍방 요청을 한 번의 관계로 매끄럽게 수렴시키지만 별도 동의 시점이 없습니다. 명시적 수락은 의사와 감사를 분명히 하지만 양쪽 pending 충돌을 어떤 상태로 둘지 추가 모델이 필요합니다.
4. 메모리 구현이 단일 프로세스 동시성만 다룬다는 한계
   - 모범답변: JavaScript 한 프로세스의 동기 Map 변경은 중간 interleave가 없지만 여러 API 인스턴스는 map을 공유하지 않습니다. 분산 writer에는 DB unique constraint와 원자 upsert가 필요합니다.
5. 실제 DB에서 동시 요청을 검증할 테스트 형태
   - 모범답변: 격리된 test schema에서 A→B와 B→A를 동시에 실행하고 둘 다 완료된 뒤 canonical pair 행 수가 1, 상태가 accepted인지 확인합니다. 같은 방향 동시 요청과 수신자 아닌 사용자의 accept도 함께 검증합니다.

### 원본 확인 위치

- Thread 17
- 커밋: `feat(db): memory friendship invariant 적용`
- 커밋: `test(db): friendship와 tournament 경쟁 상태 검증`
- `packages/db/migrations/004_friendship_tournament_invariants.sql`
- `packages/db/src/index.ts`
  - `listFriends`
  - `requestFriend`
  - `acceptFriend`
- `packages/db/src/index.test.ts`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 05, 19

---

## IM-09. [Thread 16·17 / `feat(tournament): 대진 경기 schema 추가`, `test(db): friendship와 tournament 경쟁 상태 검증`] 마지막 자리 경쟁과 대진표 단일 생성

### 면접 질문

정원 4명인 토너먼트에 3명이 참가한 상태에서 10명이 동시에 마지막 자리에 참가하려고 합니다. 정확히 한 명만 입장시키고, 시드 1~4와 준결승 두 경기를 한 번만 생성하려면 어떤 트랜잭션 경계와 제약이 필요합니까?

꼬리 질문:

- `SELECT count(*)` 뒤 `INSERT`하는 구현이 왜 초과 입장을 허용할 수 있습니까?
  - 모범답변: 기준 행을 잠그지 않으면 여러 트랜잭션이 모두 count 3을 읽고 각자 네 번째라고 판단해 insert할 수 있습니다. count와 insert를 토너먼트 행의 `FOR UPDATE` 아래 직렬화해야 합니다.
- 토너먼트 행 잠금과 원자적 조건부 업데이트 중 어떤 방식을 선택하겠습니까?
  - 모범답변: 현재 모델은 entry 수·next seed·bracket 생성까지 함께 결정하므로 원본처럼 토너먼트 한 행을 잠그는 방식이 단순하고 잠금 범위도 한 대회로 좁습니다. player_count를 기준 행에 유지한다면 capacity 조건부 update로 자리만 선점하는 대안도 가능합니다.
- 같은 사용자의 재시도는 실패가 아니라 멱등 성공으로 처리할 수 있습니까?
  - 모범답변: 할 수 있으며, row lock을 얻은 뒤 capacity 검사보다 먼저 기존 entry를 조회해 있으면 변경 없이 현재 view를 반환해야 합니다. 그래야 이미 참가한 사용자가 대회가 찬 뒤 재시도해도 `full`로 오인되지 않습니다.
- 참가자 삽입은 성공했는데 대진 생성이 실패한 경우 무엇이 보여야 합니까?
  - 모범답변: 둘은 같은 트랜잭션이므로 외부에는 참가자와 대진 모두 이전 상태로 보여야 합니다. 네 번째 참가만 남고 running bracket이 없는 상태는 허용하지 않습니다.

### 30초 모범 답변

정원 검사, 참가자 삽입, 시드 배정, 대진 생성은 하나의 invariant이므로 한 트랜잭션에서 직렬화해야 합니다. 토너먼트 기준 행을 잠그거나 정원 조건을 포함한 원자 연산으로 마지막 자리를 선점하고, 참가자와 시드에는 유일 제약을 둡니다. 네 번째 참가자가 확정된 동일 트랜잭션에서 `(tournament, round, slot)` 유일 제약 아래 준결승 두 경기를 생성합니다. 같은 참가자의 재시도는 기존 참가를 반환하고, 중간 실패는 전부 롤백합니다.

### 답변 핵심 키워드

check-then-act race · row lock · capacity invariant · seed uniqueness · bracket uniqueness · 멱등 참가 · 단일 트랜잭션 · 롤백

### 백지 구현

**구현 목표**

고정 정원의 토너먼트에 참가자를 추가하고, 정원이 찬 순간 첫 라운드 슬롯을 정확히 한 번 만드는 동시성 안전 API의 저장소 계약을 작성한다.

**인터페이스 또는 함수 시그니처**

```ts
export interface TournamentView {
  id: string;
  capacity: number;
  playerCount: number;
  entries: Array<{ userId: string; seed: number }>;
  matches: Array<{ round: "semifinal"; slot: 1 | 2 }>;
}

export interface TournamentTransaction {
  joinTournament(tournamentId: string, userId: string): Promise<TournamentView>;
}

export async function joinWithCapacityInvariant(
  tx: TournamentTransaction,
  tournamentId: string,
  userId: string
): Promise<TournamentView> {
  // 저장소 구현은 tournament 기준 행을 FOR UPDATE로 잠근 같은 트랜잭션에서
  // 기존 참가 확인 → capacity/next seed 계산 → entry 삽입 → bracket 생성을 수행한다.
  // (tournament_id, user_id), (tournament_id, seed), (tournament_id, round, slot)
  // 유일 제약이 애플리케이션 판단의 마지막 방어선이다.
  return tx.joinTournament(tournamentId, userId);
}
```

**입력과 출력**

- 입력: 토너먼트 ID, 사용자 ID
- 출력: 참가 뒤 일관된 토너먼트 뷰

**반드시 만족해야 할 조건**

- 참가자 수가 정원을 넘지 않는다.
- 같은 사용자는 한 번만 참가한다.
- 시드는 1부터 중복 없이 연속 배정된다.
- 정원이 찬 경우 준결승 슬롯 1, 2가 정확히 한 번 존재한다.
- 같은 사용자의 재호출은 상태를 늘리지 않는다.
- 참가와 대진 생성 중 하나가 실패하면 둘 다 남지 않는다.

**경계 조건**

- 이미 꽉 찬 토너먼트
- 마지막 자리를 향한 다수 동시 호출
- 같은 사용자의 동시 중복 호출
- 정원에 도달하기 전과 정확히 도달한 순간
- 대진이 이미 생성된 상태에서 재호출

**실패 조건**

- 토너먼트 또는 사용자 없음
- 정원 초과
- 유일 제약 충돌
- 대진 생성 실패

**필요한 제약**

- 저장소 계층의 트랜잭션 또는 원자 연산을 사용한다.
- 동시성 결과는 호출 순서에 의존해도 되지만 invariant는 항상 유지해야 한다.
- 전체 토너먼트 테이블을 잠그는 해법은 피하고 범위를 설명한다.

### 구현 후 자가 검증

- [ ] 3명 상태에서 10개 동시 시도 중 정확히 한 건만 새로 성공한다.
- [ ] 최종 참가자 수, 사용자 중복, 시드 연속성이 유지된다.
- [ ] 준결승 슬롯이 두 개만 존재한다.
- [ ] 이미 참가한 사용자의 재시도가 무해하다.
- [ ] 대진 생성 실패를 주입했을 때 참가도 롤백된다.
- [ ] 잠금 범위와 교착 가능성을 설명할 수 있다.

### 구현 후 설명할 것

1. `count 후 insert`가 경쟁 상태에 취약한 이유
   - 모범답변: 잠금 없는 count는 뒤따르는 insert와 하나의 원자 연산이 아니므로 여러 요청이 같은 빈 자리를 관측합니다. 원본은 tournament ID의 기준 행을 `FOR UPDATE`한 뒤 count와 insert를 수행해 대회별로 직렬화합니다.
2. 잠글 기준 행과 잠금 범위를 선택한 이유
   - 모범답변: capacity와 status를 소유한 tournament 한 행이 모든 참가 요청이 공유하는 안정적인 기준입니다. 해당 대회 행만 잠그므로 서로 다른 토너먼트 참가까지 직렬화하지 않습니다.
3. 애플리케이션 검증과 DB 제약을 함께 두는 이유
   - 모범답변: 애플리케이션은 `already joined`, `full` 같은 의미 있는 분기와 오류를 제공하고, DB의 사용자·seed·round/slot unique 제약은 버그나 다른 writer가 invariant를 깨는 것을 최종 차단합니다.
4. 멱등 재시도와 정원 초과 오류를 구분하는 순서
   - 모범답변: 잠금 뒤 먼저 `(tournament,user)` 기존 entry를 조회해 있으면 성공으로 반환하고, 없을 때만 count를 capacity와 비교합니다. 순서가 반대면 이미 참가한 사용자의 재시도가 가득 찬 대회에서 실패합니다.
5. 정원이 커지거나 가변 대진이 될 때의 확장 방향
   - 모범답변: 고정 4명에서는 seed 1↔4, 2↔3 두 행이면 충분합니다. 가변 정원에서는 bracket size와 bye 규칙을 명시하고 `(round,slot)` 생성기를 별도 순수 함수로 두되, 최종 entry 확정과 bracket materialization은 같은 트랜잭션에 유지합니다.

### 원본 확인 위치

- Thread 16
- 커밋: `feat(tournament): 대진 경기 schema 추가`
- Thread 17
- 커밋: `test(db): friendship와 tournament 경쟁 상태 검증`
- `packages/db/migrations/004_friendship_tournament_invariants.sql`
- `packages/db/src/index.ts`
  - `joinTournament`
  - `getTournamentMatch`
  - `startTournamentMatch`
  - `completeTournamentMatch`
  - `ensureFinalMatch`
- `packages/db/src/postgres.integration.test.ts`
- 관련 Thread: 11, 15

---

## IM-10. [Thread 18 / `fix(game): 매치 채팅의 좌석과 audience 검증`, `test(game): 타 경기방 채팅 주입 차단 검증`] 타입·저장소·실시간 라우팅의 다층 채팅 권한

### 면접 질문

클라이언트가 유효한 다른 경기방 UUID를 알고 있을 때, 왜 이벤트 스키마 검증과 데이터베이스 `scope/roomId` 제약만으로는 채팅 주입을 막을 수 없나요? 현재 연결의 좌석과 실제 방 소속까지 검증해야 하는 이유를 설명해 주세요.

꼬리 질문:

- 로비 채팅과 매치 채팅을 하나의 느슨한 타입으로 표현했을 때 어떤 잘못된 상태가 가능합니까?
  - 모범답변: 로비 메시지에 room ID가 붙거나 매치 메시지에 room ID가 없는 상태, scope와 room이 모순된 상태가 타입상 가능해집니다. 원본 codec은 scope 판별 유니온으로, DB는 check constraint로 이 조합을 각각 차단합니다.
- 발신 권한과 수신 audience를 각각 어디에서 검증해야 합니까?
  - 모범답변: GameHub가 현재 client의 `roomId`와 실제 `sideFor(room, client)`로 발신 권한을 검사합니다. 수신 audience도 client가 보낸 문자열이 아니라 서버의 room.clients에서 계산해 그 방 두 좌석에만 broadcast합니다.
- 방이 종료되거나 재연결 중일 때 매치 채팅 정책은 어떻게 정의하겠습니까?
  - 모범답변: 현재 구현은 room이 map에 존재하고 client가 그 room의 실제 좌석일 때만 허용하므로 종료 후에는 거부됩니다. 재연결은 새 client가 room side를 복구한 뒤에만 허용되어야 하며 예약만 있고 좌석 복구 전인 상태의 정책을 명시해야 합니다.
- 저장 성공 뒤 broadcast 실패와 broadcast 뒤 저장 실패 중 어떤 순서가 더 적절합니까?
  - 모범답변: 원본처럼 권한 검사 후 저장하고 성공한 메시지만 broadcast하는 순서가 적절합니다. 그러면 보였지만 저장되지 않은 메시지는 막지만, 저장 후 전송 실패는 남으므로 재조회·재전송이 필요한 전달 보장은 별도로 설계해야 합니다.

### 30초 모범 답변

문법적으로 유효한 room ID는 권한을 증명하지 않습니다. 프로토콜에서는 로비와 매치 메시지를 서로 다른 형태로 만들어 잘못된 조합을 막고, 저장소에서는 `scope`와 `roomId`의 관계를 다시 검증합니다. 마지막으로 GameHub에서 연결이 그 방에 속하고 실제 좌석을 점유했는지 확인한 뒤 그 방 audience에만 전송해야 합니다. 즉 형식, 도메인 invariant, 인가, 배포 범위를 서로 다른 층에서 각각 지킵니다.

### 답변 핵심 키워드

문법과 권한 분리 · discriminated union · scope invariant · room membership · seat ownership · audience 제한 · 다층 방어 · 저장·방송 순서

### 백지 구현

**구현 목표**

로비 또는 경기방 채팅 명령을 처리하되, 경기방 메시지는 현재 연결이 실제 참가자인 경우에만 저장·방송하는 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```ts
type ChatCommand =
  | { scope: "lobby"; body: string }
  | { scope: "match"; roomId: string; body: string };

interface ConnectionContext {
  userId: string;
  currentRoomId: string | null;
}

interface ChatDependencies {
  isSeated(roomId: string, userId: string): boolean;
  save(input: {
    scope: "lobby" | "match";
    roomId: string | null;
    senderId: string;
    body: string;
  }): Promise<{ id: string }>;
  broadcastLobby(messageId: string): void;
  broadcastRoom(roomId: string, messageId: string): void;
}

export async function handleChat(
  context: ConnectionContext,
  command: ChatCommand,
  dependencies: ChatDependencies
): Promise<void> {
  const body = command.body.trim();
  if (body.length === 0 || body.length > 240) throw new Error("invalid chat body");

  if (command.scope === "lobby") {
    const message = await dependencies.save({
      scope: "lobby",
      roomId: null,
      senderId: context.userId,
      body
    });
    dependencies.broadcastLobby(message.id);
    return;
  }

  if (
    context.currentRoomId !== command.roomId
    || !dependencies.isSeated(command.roomId, context.userId)
  ) {
    // 문법적으로 유효한 room ID와 현재 좌석 소유권은 별개의 조건이다.
    throw new Error("forbidden match chat");
  }
  const message = await dependencies.save({
    scope: "match",
    roomId: command.roomId,
    senderId: context.userId,
    body
  });
  dependencies.broadcastRoom(command.roomId, message.id);
}
```

**입력과 출력**

- 입력: 검증된 명령, 현재 연결 문맥, 저장·방송 의존성
- 출력: 성공 시 없음, 권한 또는 저장 실패 시 오류

**반드시 만족해야 할 조건**

- 로비 메시지는 `roomId` 없이 저장하고 로비에만 보낸다.
- 매치 메시지는 연결의 현재 방과 명령의 방이 같아야 한다.
- 사용자가 해당 방의 좌석을 점유해야 한다.
- 권한 검증 실패 시 저장과 방송이 모두 일어나지 않는다.
- 매치 메시지는 다른 방과 로비에 전파되지 않는다.
- 빈 본문과 최대 길이는 처리 전에 검증됐다고 가정하거나 함수 안에서 명시적으로 검증한다.

**경계 조건**

- 존재하지 않는 방
- 현재 방은 같지만 spectator인 연결
- 방 이동 직후 이전 room ID를 보낸 경우
- 저장 직후 연결이 끊긴 경우

**실패 조건**

- 권한 없음
- 저장소 실패
- 방송 콜백 실패

**필요한 제약**

- 권한 검증 이전에는 외부 부수 효과를 만들지 않는다.
- 어떤 audience에 보낼지는 명령의 문자열만 믿지 않고 서버 상태에서 결정한다.

### 구현 후 자가 검증

- [ ] 로비와 매치 입력의 가능한 필드 조합이 명확하다.
- [ ] 다른 방 UUID를 넣은 공격이 저장 전에 거부된다.
- [ ] 좌석이 없는 연결은 같은 방 ID를 알아도 전송할 수 없다.
- [ ] 성공한 메시지는 정확한 audience 한 곳에만 전파된다.
- [ ] 저장 실패 시 방송되지 않는다.
- [ ] 방 상태가 동시에 바뀌는 경우의 일관성 범위를 설명할 수 있다.

### 구현 후 설명할 것

1. 형식 검증과 객체 수준 인가가 다른 문제인 이유
   - 모범답변: codec은 room ID가 문자열·UUID이고 필드 조합이 맞는지만 증명합니다. 그 ID의 방에 현재 사용자가 참여할 권한은 서버의 최신 room과 seat 상태를 조회해야만 알 수 있습니다.
2. 프로토콜, 저장소, 실시간 허브에 각각 둔 방어선
   - 모범답변: 공유 protocol은 lobby/match 구조와 길이를, 저장소와 DB check는 scope-room 조합을, GameHub는 현재 room membership·seat ownership과 실제 broadcast audience를 검증합니다. 각 층이 서로 다른 잘못된 상태를 막습니다.
3. 저장 후 방송 순서를 선택한 이유와 전달 보장의 한계
   - 모범답변: 저장 성공 전에는 broadcast하지 않아 수신자가 본 메시지가 DB에 없는 상태를 피합니다. 다만 commit 후 socket 전송 실패는 가능하므로 정확히 한 번 전달은 보장하지 않고, 필요하면 message ID 기반 재조회나 outbox를 추가해야 합니다.
4. 참가자·관전자 정책을 타입과 런타임 상태에 표현하는 방법
   - 모범답변: 명령 타입은 match room ID를 요구하되 발신 주체의 역할은 서버 context로 판단합니다. 현재 프로젝트는 `sideFor`가 반환되는 좌석 참가자만 허용하며 spectator 발신 경로는 정의하지 않습니다.
5. 재연결·방 종료와 채팅 권한 사이의 경쟁 상태 처리
   - 모범답변: 메시지 처리 시점에 `client.roomId`, 현재 room 객체, 그 객체의 side identity를 함께 확인합니다. 방 제거 또는 client 교체가 먼저 끝났으면 실패하고, 비동기 저장 중 방이 끝나는 경우 저장은 남을 수 있으므로 더 강한 원자성이 필요하면 room lifecycle과 저장 경계를 조정해야 합니다.

### 원본 확인 위치

- Thread 18
- 커밋: `fix(game): 매치 채팅의 좌석과 audience 검증`
- 커밋: `test(game): 타 경기방 채팅 주입 차단 검증`
- `packages/shared/src/ws.ts`
- `packages/db/migrations/006_chat_invariants.sql`
- `packages/db/src/index.ts`
  - `assertChatRoom`
  - `createChatMessage`
- `apps/api/src/gameHub.ts`
- `apps/api/src/gameHub.chat.test.ts`
- 관련 Thread: 02, 03, 12
