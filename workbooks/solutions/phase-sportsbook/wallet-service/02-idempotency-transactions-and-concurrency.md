# 멱등성, 트랜잭션, 동시성

이 문서는 같은 요청의 재시도와 서로 다른 요청의 경쟁을 구분하고, 금전 변경을 하나의 DB 트랜잭션으로 수렴시키는 핵심 흐름을 다룬다.

<a id="w03"></a>
## W03 — [Thread 6 / `feat(operation): reject conflicting request identities`] 정규 요청 지문과 멱등성 충돌

### 면접 질문

멱등성 키가 같으면 무조건 저장된 결과를 돌려주면 안 되는 이유는 무엇입니까? 이 프로젝트에서 요청의 의미를 안정적으로 식별하기 위한 fingerprint가 어떤 성질을 가져야 하는지 설명해 보세요.

꼬리 질문:

- 문자열을 구분자로 이어 붙이는 방식은 어떤 충돌을 만들 수 있습니까?
- JSON 직렬화 결과를 그대로 해시하면 필드 순서·serializer 변경·버전 변경에 어떤 문제가 생깁니까?
- 인증된 호출자도 fingerprint에 포함해야 하는 이유는 무엇입니까?
- idempotency key 자체와 semantic fingerprint의 역할은 어떻게 다릅니까?
- encoding version을 앞에 두면 무엇을 얻습니까?

### 30초 모범 답변

멱등성 키는 저장된 결과를 찾는 주소이고, fingerprint는 그 주소를 사용한 요청의 의미입니다. 같은 키에 호출자·업무 종류·사용자·금액·통화가 달라지면 과거 결과를 재생하지 않고 충돌로 거부해야 합니다. fingerprint는 고정된 필드 순서와 길이 경계가 있는 버전된 이진 인코딩을 사용하고 SHA-256처럼 결정적인 해시로 저장해야 합니다. `toString`, 구분자 연결, serializer 기본값에 의존하면 서로 다른 요청이 같은 바이트가 되거나 배포 뒤 동일 요청의 바이트가 달라질 수 있습니다.

### 답변 핵심 키워드

`키는 주소` · `fingerprint는 의미` · `canonical encoding` · `길이 경계` · `version prefix` · `caller 포함` · `deterministic hash` · `conflict`

### 백지 구현

#### 구현 목표

금전 이체 요청의 의미 필드만 정규 이진 표현으로 인코딩하고 64자 lowercase SHA-256 hex fingerprint를 반환한다.

#### 면접용 축소 인터페이스

```java
enum Caller { PLATFORM, BETTING, SETTLEMENT, ADMIN }
enum Kind { DEPOSIT, WITHDRAW, BET_DEBIT, BET_PAYOUT, BET_REFUND, BET_FORFEIT }

enum Currency { KRW, USD }

record SemanticTransfer(
    Caller caller,
    Kind kind,
    UUID userId,
    long amount,
    Currency currency) {}

final class RequestFingerprint {
  static String sha256(SemanticTransfer request) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: null이 아닌 호출자, 업무 종류, 사용자 UUID, 양의 금액, 통화
- 출력: 같은 semantic input에는 항상 같은 64자 lowercase hex 문자열

#### 반드시 만족해야 할 조건

- 인코딩 형식의 version을 명시적으로 포함한다.
- 각 필드는 고정된 순서로 들어간다.
- 가변 길이 값은 서로 이어 붙였을 때 경계가 모호하지 않아야 한다.
- UUID는 정규 16바이트 값으로 처리한다.
- `long`의 바이트 순서를 구현 안에서 고정한다.
- enum은 우연히 바뀔 수 있는 `ordinal()`에 의존하지 않는다.
- platform 기본 charset, locale, 객체 `toString()`에 의존하지 않는다.
- idempotency key는 조회 주소이므로 이 축소 문제의 semantic hash에는 넣지 않는다.

#### 경계 조건

- 한 필드만 다른 두 요청
- 값 조합이 문자열 연결에서는 모호해질 수 있는 입력
- `Long.MAX_VALUE` 금액
- 서로 다른 caller가 같은 나머지 필드를 보낸 경우
- enum 선언 순서를 바꿔도 기존 wire code가 유지되는 설계
- null 또는 비양수 금액

#### 실패 조건

- 필수 필드 누락
- 지원하지 않는 값
- 양수가 아닌 금액
- 요청 객체를 안정적으로 인코딩할 수 없는 경우

#### 필요한 제약

- 표준 Java API만 사용한다.
- 해시 함수 호출 전 canonical bytes가 하나로 결정되어야 한다.
- 시간 복잡도와 공간 복잡도는 인코딩 길이에 대해 O(n)이다.

### 구현 후 자가 검증

- [ ] 같은 객체와 같은 값을 가진 새 객체가 같은 fingerprint를 낸다.
- [ ] caller, kind, userId, amount, currency를 하나씩 바꾸면 fingerprint가 달라진다.
- [ ] JVM 기본 charset과 locale을 바꿔도 결과가 같다.
- [ ] enum ordinal을 사용하지 않았다.
- [ ] 출력이 정확히 64자 lowercase hex다.
- [ ] 길이 경계가 없는 문자열 연결을 사용하지 않았다.
- [ ] version 값이 인코딩에 실제로 포함된다.
- [ ] 비양수 금액과 null 입력을 거부한다.

### 구현 후 설명할 것

1. idempotency key와 fingerprint를 분리한 이유.
2. JSON이나 Java serialization 대신 직접 정의한 canonical binary 형식을 택한 이유.
3. encoding version 변경 시 기존 저장 결과와의 호환 전략.
4. caller를 요청 의미에 포함해 cross-caller replay를 막는 이유.
5. SHA-256 collision 위험과 애플리케이션에서 받아들이는 trade-off.

### 원본 확인 위치

- Thread: **6 — 영속 멱등성과 결과 재생**
- 커밋: `feat(operation): encode canonical request fields`
- 커밋: `feat(operation): reject conflicting request identities`
- 파일: `src/main/java/com/sportsbook/wallet/service/CanonicalRequestEncoder.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletRequestIdentity.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/OperationFingerprint.java`
- 파일: `src/main/java/com/sportsbook/wallet/domain/error/IdempotencyConflictException.java`
- 클래스·메서드: `CanonicalRequestEncoder`, `WalletRequestIdentity.requireMatching()`, `OperationFingerprint`
- 관련 Thread: 2, 10, 16, 17

---

<a id="w04"></a>
## W04 — [Thread 6 / `feat(operation): resolve fingerprints inside execution transactions`] 영속 최초 작성자 직렬화와 결과 재생

### 면접 질문

서로 다른 프로세스에서 같은 idempotency key 요청이 동시에 들어왔을 때 정확히 하나의 최초 작성자만 업무 로직을 실행하고 나머지는 같은 영속 결과로 수렴하게 하려면 어떤 순서와 경계가 필요합니까?

꼬리 질문:

- 최초 read에서 결과가 없었는데도 트랜잭션 잠금 뒤 다시 읽어야 하는 이유는 무엇입니까?
- unique constraint만 두고 insert 충돌을 처리하는 방식과 advisory lock 방식의 trade-off는 무엇입니까?
- 최초 작성 시도가 예외로 끝난 뒤 결과를 한 번 더 읽는 이유는 무엇입니까?
- 같은 key와 다른 fingerprint가 동시에 들어오면 어떤 결과여야 합니까?
- Redis marker가 없어도 PostgreSQL 결과를 찾을 수 있어야 하는 이유는 무엇입니까?
- 이 executor를 이미 열린 외부 트랜잭션 안에서 호출하지 못하게 한 이유는 무엇입니까?

### 30초 모범 답변

먼저 저장된 결과가 있으면 fingerprint를 확인해 바로 재생합니다. 결과가 없으면 새 트랜잭션에서 key 단위의 transaction-scoped advisory lock을 잡고 다시 읽어 승자가 생겼는지 확인한 뒤, 여전히 없을 때만 최초 업무 쓰기와 outcome 저장을 수행합니다. 시도가 실패하면 트랜잭션 밖에서 결과를 재조회해 실제로 다른 승자가 커밋했는지 구분합니다. Redis는 조회 힌트일 뿐 결과 존재 여부를 결정하지 않으며, 같은 key의 다른 fingerprint는 409 성격의 충돌입니다.

### 답변 핵심 키워드

`fast replay` · `transaction-scoped advisory lock` · `double check` · `single first writer` · `failure reread` · `durable outcome` · `cache is a hint` · `fingerprint conflict`

### 백지 구현

#### 구현 목표

주어진 저장소·트랜잭션·key lock 추상화를 사용해 다중 스레드와 다중 프로세스에서도 같은 key가 하나의 영속 결과로 수렴하는 executor를 작성한다.

#### 면접용 축소 인터페이스

```java
record Request(String key, String fingerprint) {}
record Outcome(String key, String fingerprint, Status status) {}

enum Status { SUCCEEDED, REJECTED, BLOCKED }

interface OutcomeStore {
  Optional<Outcome> find(String key);
  Outcome insertAndFlush(Outcome outcome);
}

interface TransactionRunner {
  <T> T run(Supplier<T> work);
  boolean isTransactionActive();
}

interface TransactionKeyLock {
  void acquire(String key);
}

interface HintCache {
  boolean mightContain(String key);
  void mark(String key);
}

final class FirstWriterExecutor {
  Outcome execute(Request request, Supplier<Outcome> firstWriter) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: idempotency key와 canonical fingerprint, 최초 작성 결과 생성 함수
- 출력: 해당 key에 최종적으로 저장된 동일한 영속 outcome
- 충돌: 저장 outcome의 fingerprint가 요청과 다르면 별도 conflict 예외

#### 반드시 만족해야 할 조건

- 이미 저장된 동일 요청은 최초 작성 함수를 호출하지 않고 재생한다.
- 같은 key의 동시 요청 중 최초 작성 함수는 성공적으로 한 번만 결과를 소유해야 한다.
- key 잠금은 업무 쓰기와 outcome insert가 들어가는 트랜잭션 범위에 묶여야 한다.
- 잠금을 얻기 전 관찰한 부재만 믿지 않고, 잠금 경계 안에서 승자 존재 여부를 다시 확인한다.
- 저장 outcome과 요청 fingerprint가 다르면 성공·거절 여부와 무관하게 충돌이다.
- 쓰기 시도 중 예외가 발생했더라도 다른 승자가 커밋했을 가능성을 구분해야 한다.
- 실제로 아무 outcome도 커밋되지 않았다면 원래 실패를 숨기지 않는다.
- cache miss 또는 cache 장애는 PostgreSQL 조회·최초 작성 경로를 막지 않는다.
- cache mark는 커밋된 outcome 뒤의 best-effort 동작이며 실패해도 결과를 바꾸지 않는다.
- 이미 열린 외부 트랜잭션에서 호출되는 경우 이 축소 executor는 명시적으로 거부한다.

#### 경계 조건

- 결과가 이미 있는 빠른 재생
- 같은 key·같은 fingerprint 요청 100개
- 같은 key·서로 다른 fingerprint 두 요청
- 최초 read 뒤 다른 작업자가 커밋한 경쟁
- insert/flush 예외 뒤 실제 승자 결과가 존재하는 경우
- insert/flush 예외 뒤 아무 결과도 없는 경우
- cache가 false를 반환하거나 예외를 던지는 경우
- 최초 작성 함수가 null을 반환하는 경우

#### 실패 조건

- key 또는 fingerprint 누락
- 외부 트랜잭션이 이미 활성화된 경우
- fingerprint 충돌
- 저장소·잠금·트랜잭션 인프라 오류이고 승자 결과도 없는 경우
- first writer가 유효하지 않은 outcome을 만든 경우

#### 필요한 제약

- JVM 로컬 `synchronized`나 `ConcurrentHashMap`만으로 프로세스 간 경쟁을 해결하지 않는다.
- cache가 correctness branch를 소유하지 않는다.
- 같은 key에 대한 lock namespace가 다른 업무 lock과 의도치 않게 충돌하지 않아야 한다.
- 정상 replay는 불필요한 write transaction을 열지 않는 방향을 유지한다.

### 구현 후 자가 검증

- [ ] 저장된 동일 outcome을 lock과 first writer 없이 재생한다.
- [ ] 잠금 경계 안에서 두 번째 read가 존재한다.
- [ ] 100개 동시 요청이 같은 outcome을 돌려받고 first writer 소유 결과는 하나다.
- [ ] 같은 key의 다른 fingerprint가 어떤 타이밍에서도 conflict가 된다.
- [ ] 실패 후 winner가 발견되면 원래 예외 대신 winner를 검증해 반환한다.
- [ ] 실패 후 winner가 없으면 인프라 오류를 숨기지 않는다.
- [ ] cache miss·장애·mark 실패가 영속 결과를 바꾸지 않는다.
- [ ] 외부 트랜잭션 중첩을 거부한다.
- [ ] lock은 트랜잭션 종료 시 자동 해제되는 모델이다.

### 구현 후 설명할 것

1. 빠른 replay read와 잠금 후 double-check를 함께 둔 이유.
2. transaction-scoped advisory lock과 unique-insert 경쟁 방식의 장단점.
3. 실패 뒤 winner reread가 timeout·connection reset 같은 모호한 결과를 다루는 방법.
4. cache를 negative authority로 사용하지 않은 이유.
5. executor가 외부 트랜잭션을 금지해 소유하는 transaction boundary를 명확히 한 이유.

### 원본 확인 위치

- Thread: **6 — 영속 멱등성과 결과 재생**
- 커밋: `feat(locking): acquire transaction-scoped idempotency locks`
- 커밋: `feat(operation): resolve fingerprints inside execution transactions`
- 커밋: `feat(cache): treat Redis idempotency data as a fallible hint`
- 커밋: `test(concurrency): converge one hundred requests under one key`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletOperationExecutor.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/IdempotencyKeyLock.java`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/WalletOperationRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/IdempotencyCache.java`
- 클래스·메서드: `WalletOperationExecutor.execute()`, `IdempotencyKeyLock.acquire()`, `IdempotencyCache.mightContain()`, `IdempotencyCache.mark()`
- 관련 Thread: 1, 2, 7, 10, 16, 17

---

<a id="w05"></a>
## W05 — [Thread 7 / `feat(service): execute durable transfer outcomes`] 계정 잠금과 단일 트랜잭션 debit

### 면접 질문

베팅 debit을 처리할 때 잔액 변경, 두 원장 행, 영속 operation outcome, 아웃박스 이벤트를 어떤 단위로 커밋해야 합니까? 잔액이 정확히 100인 계정에 서로 다른 key로 100원 debit 두 건이 동시에 들어오는 경우까지 설명해 보세요.

꼬리 질문:

- 계정을 일반 조회한 뒤 애플리케이션에서 검사하면 어떤 lost-update 또는 double-spend가 생깁니까?
- idempotency key lock과 account row lock은 서로 어떤 경쟁을 막습니까?
- 잔액 부족은 왜 인프라 실패와 다르게 영속 `REJECTED` outcome이 될 수 있습니까?
- debit 실패 이벤트는 남기면서 원장 행은 남기지 않아야 하는 이유는 무엇입니까?
- 트랜잭션 안의 알림 listener가 예외를 던졌다면 어떤 행도 남지 않아야 합니까?

### 30초 모범 답변

key 경쟁은 멱등성 lock으로, 서로 다른 key가 같은 잔액을 쓰는 경쟁은 account row의 `FOR UPDATE` 계열 잠금으로 막습니다. 잠근 최신 계정 상태에서 debit 가능 여부를 판정하고, 성공이면 available→locked 변경·matched ledger pair·SUCCEEDED outcome·성공 outbox를 한 트랜잭션에 넣습니다. 잔액 부족은 잔액과 ledger를 건드리지 않고 REJECTED outcome과 실패 outbox만 함께 커밋합니다. 그 외 예외는 전체 롤백되어 key가 실제로 커밋되지 않았다면 안전하게 재시도할 수 있어야 합니다.

### 답변 핵심 키워드

`idempotency lock` · `pessimistic row lock` · `latest balance` · `single transaction` · `business rejection commits facts` · `infrastructure failure rolls back` · `exact-balance race`

### 백지 구현

#### 구현 목표

트랜잭션·계정 잠금·저장소 추상화가 제공된다고 가정하고 `BET_DEBIT` 한 종류를 원자적으로 처리한다. 멱등성 최초 작성자 처리는 W04가 담당한다고 가정한다.

#### 면접용 축소 인터페이스

```java
record DebitCommand(UUID userId, long amount, Currency currency, String operationKey) {}

sealed interface DebitOutcome {
  record Succeeded(UUID groupId) implements DebitOutcome {}
  record Rejected(String code, String detail) implements DebitOutcome {}
}

interface AccountStore {
  Account lockByUserId(UUID userId);
  void flush(Account account);
}

interface DebitCommitStore {
  void insertLedgerPair(LedgerPair pair);
  void insertOutcome(String operationKey, DebitOutcome outcome);
  void appendSuccessEvent(String operationKey, UUID userLedgerEntryId);
  void appendFailureEvent(String operationKey, String reason);
}

interface TransactionRunner {
  <T> T run(Supplier<T> work);
}

final class DebitWriter {
  DebitOutcome debit(DebitCommand command) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 사용자, 양의 debit 금액, 통화, operation key
- 출력: 성공 group ID 또는 저장 가능한 업무 거절 snapshot
- 인프라 실패: outcome을 가장하지 않고 예외

#### 반드시 만족해야 할 조건

성공 경로:

- 쓰기 판정 전에 해당 사용자 계정을 row lock으로 읽는다.
- 통화가 맞고 available이 충분해야 한다.
- `available -= amount`, `locked += amount`가 같은 계정 상태에 반영된다.
- `BET_DEBIT` 토폴로지의 정확한 두 원장 행을 만든다.
- 성공 outcome과 성공 outbox event를 기록한다.
- 위 변경은 하나의 트랜잭션에서 모두 커밋된다.

업무 거절 경로:

- 계정 없음, 통화 불일치, 잔액 부족 중 프로젝트가 영속 업무 거절로 취급하는 조건을 안정적인 snapshot으로 만든다.
- 잔액과 ledger는 바꾸지 않는다.
- `REJECTED` outcome을 기록한다.
- debit 실패 outbox event를 기록한다.
- 거절 facts도 하나의 트랜잭션으로 커밋된다.

공통:

- 저장·flush·트랜잭션 내부 알림 중 하나라도 인프라 예외를 던지면 해당 시도의 변경은 모두 롤백된다.
- 서로 다른 key의 두 exact-balance debit이 경쟁하면 하나만 성공할 수 있다.
- 외부 Kafka send를 이 트랜잭션 안에서 호출하지 않는다.

#### 경계 조건

- available과 amount가 정확히 같은 성공
- 1원 부족한 거절
- 서로 다른 key의 동시 exact-balance 요청
- 같은 key 재시도는 W04를 통해 이 writer가 두 번 실행되지 않는 조건
- account row lock 획득 대기 또는 timeout
- ledger 저장 뒤 outcome 저장 실패
- outcome 저장 뒤 outbox 저장 실패
- transaction-bound listener 실패

#### 실패 조건

- 비양수 금액
- 통화 불일치
- 계정 없음
- 잔액 부족
- DB lock·timeout·connection·constraint 오류
- 잘못된 ledger pair 생성

#### 필요한 제약

- 계정 lock 이전에 읽은 잔액을 판정에 사용하지 않는다.
- 서로 다른 계정의 독립 요청까지 전역 JVM lock으로 직렬화하지 않는다.
- 성공과 업무 거절을 구분하되 둘 다 일관된 영속 facts를 남긴다.
- 전체 시간 복잡도는 저장 행 수 기준 O(1)이다.

### 구현 후 자가 검증

- [ ] 충분한 잔액에서 available과 locked가 정확히 반대 방향으로 바뀐다.
- [ ] 성공 시 ledger가 정확히 두 행이고 outcome·outbox가 각각 하나다.
- [ ] 잔액 부족 시 account와 ledger는 그대로이고 rejection outcome·failure event만 남는다.
- [ ] 중간 저장 실패를 각 지점에 주입했을 때 어떤 부분 행도 남지 않는다.
- [ ] 정확한 잔액을 놓고 동시 요청 두 개를 실행하면 성공은 하나뿐이다.
- [ ] 두 요청이 서로 다른 사용자라면 불필요하게 서로를 직렬화하지 않는다.
- [ ] 잠근 최신 상태를 기준으로 판정한다.
- [ ] 직접 Kafka를 호출하지 않고 outbox 사실만 기록한다.

### 구현 후 설명할 것

1. idempotency key lock과 account row lock이 각각 보호하는 자원이 다른 이유.
2. 업무 거절은 커밋하지만 인프라 실패는 롤백하는 기준.
3. 계정 mutation, ledger pair, outcome, outbox의 transaction boundary를 나눌 수 없는 이유.
4. pessimistic lock을 택했을 때의 처리량·대기·deadlock trade-off.
5. fault injection test가 단순 정상 경로 테스트보다 보여 주는 보장.

### 원본 확인 위치

- Thread: **7 — 원자적 자금 이체**
- 커밋: `feat(repository): lock accounts for balance writes`
- 커밋: `feat(service): persist matched transfers and commit notifications`
- 커밋: `feat(service): execute durable transfer outcomes`
- 커밋: `test(operation): roll back outcomes with failed money transactions`
- 커밋: `test(concurrency): converge exact-balance debit races`
- 파일: `src/main/java/com/sportsbook/wallet/persistence/AccountRepository.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletTransferPlan.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletTransferWriter.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletTransferExecutor.java`
- 파일: `src/main/java/com/sportsbook/wallet/service/WalletService.java`
- 클래스·메서드: `AccountRepository.findByUserIdForUpdate()`, `WalletTransferExecutor`, `WalletTransferWriter.write()`·`writeReceipt()`
- 테스트: `WalletPersistenceTest`
- 관련 Thread: 4, 5, 6, 8, 12, 17
