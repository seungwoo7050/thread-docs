# 내구성 있는 멱등성, 정확한 결과 재생, 원격 부작용 증거

이 문서는 요청 멱등성 namespace를 데이터베이스 권위로 만드는 방법과, 리스크·지갑 서비스의 응답을 "성공 추정"이 아니라 검증 가능한 증거로 취급하는 방법을 다룬다. Thread 06과 07은 동일한 역량을 반복하므로 두 번째 항목에 통합했다.

<a id="t05-durable-idempotency"></a>
## [Thread 05 / `feat(placement): replay durable outcomes safely`] 요청 키를 소유하고 정확한 과거 결과를 재생하기

### 면접 질문

`Idempotency-Key`를 단순히 성공한 bet row의 unique column으로 두지 않고, 성공·사전 거부를 모두 표현하는 별도 `PlacementRequest` namespace로 만든 이유는 무엇입니까? 동시에 같은 키가 들어왔을 때 unique 충돌을 어떻게 정상적인 race 해결 경로로 사용했습니까?

꼬리 질문:

- 같은 키와 같은 payload의 재시도, 같은 키와 다른 payload, 다른 actor의 같은 키를 각각 어떻게 처리해야 합니까?
- validation 실패를 왜 일부만 영속화하고 dependency unavailable 같은 실패는 영속화하지 않습니까?
- unique insert가 실패한 뒤 곧바로 원래 예외를 반환하지 않고 authoritative row를 다시 읽는 이유는 무엇입니까?
- Redis 캐시를 멱등성의 source of truth로 쓰지 않은 이유는 무엇입니까?
- DB transaction에서 bet를 먼저 flush하고 request pointer를 쓰는 순서에는 어떤 의미가 있습니까?
- 과거 row에 fingerprint가 없는 migration 호환성은 어느 수준까지 허용할 수 있습니까?

### 30초 모범 답변

멱등성은 "중복 성공 방지"가 아니라 한 키가 어떤 요청과 어떤 결과를 영구히 소유하는 계약입니다. 그래서 성공 bet 포인터와 확정적 사전 거부를 같은 `PlacementRequest` namespace에 저장하고 actor와 canonical fingerprint를 함께 검증합니다. 동시 요청은 unique 제약으로 한 요청만 소유권을 얻고, 패자는 충돌 뒤 DB를 다시 읽어 승자의 정확한 결과를 재생합니다. 일시적 의존성 장애는 나중에 성공할 수 있으므로 영속 verdict로 고정하지 않고, 캐시는 조회 최적화일 뿐 권위가 아닙니다.

### 답변 핵심 키워드

`idempotency namespace`, `request fingerprint`, `actor binding`, `durable verdict`, `unique constraint race`, `reread winner`, `authoritative database`, `transient vs definitive failure`

### 백지 구현

**구현 목표**

동일 키의 요청을 한 번만 소유하고, 같은 actor·같은 fingerprint에는 저장된 결과를 재생하며, 다른 요청에는 충돌을 반환하는 축소형 멱등성 조정기를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
sealed interface Outcome permits Accepted, Rejected {}
record Accepted(UUID resourceId) implements Outcome {}
record Rejected(String code, String detail) implements Outcome {}
record RequestRecord(String key, UUID actorId, String fingerprint, Outcome outcome) {}

interface RequestStore {
  Optional<RequestRecord> find(String key);
  void insert(RequestRecord record) throws UniqueKeyCollision;
}

final class IdempotencyCoordinator {
  Outcome execute(
      String key,
      UUID actorId,
      String fingerprint,
      Supplier<Outcome> operation,
      Predicate<Outcome> durable,
      RequestStore store) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: key, actor ID, canonical fingerprint, 실제 작업, 결과의 영속 가능 여부, 저장소
- 출력: 새로 계산했거나 과거에 저장된 정확한 `Outcome`

**반드시 만족해야 할 조건**

- 기존 key가 있으면 실제 작업을 다시 실행하지 않는다.
- 기존 actor 또는 fingerprint가 다르면 결과를 노출하지 않고 충돌한다.
- durable 결과만 저장한다.
- insert unique 충돌 뒤에는 저장소를 다시 읽어 승자의 결과를 검증·재생한다.
- 충돌 뒤에도 row가 없다면 원래 저장 실패를 숨기지 않는다.
- 저장된 rejection은 원래 code와 detail을 유지한다.
- key, actor, fingerprint가 null이거나 비정규형이면 작업 전 거부한다.

**경계 조건**

- 첫 요청
- 같은 요청의 즉시 재시도
- 동시에 실행된 두 요청
- 같은 key와 다른 fingerprint
- 같은 key와 다른 actor
- operation이 transient exception으로 실패
- operation 결과는 생성됐지만 저장이 실패
- unique collision 뒤 row 가시성이 늦거나 존재하지 않는 경우

**실패 조건**

소유권 충돌, transient operation failure, 비정상 저장소 상태를 서로 구분한다. 어느 실패에서도 다른 actor의 저장 결과를 반환하지 않는다.

**제약**

실제 데이터베이스 transaction 코드는 제외하고 store 계약으로 축소한다. thread sleep이나 전역 lock으로 race를 숨기지 않는다.

### 구현 후 자가 검증

- [ ] 동일 actor·동일 fingerprint 재시도는 같은 outcome을 반환한다.
- [ ] 재생 경로에서 operation 호출 횟수가 늘지 않는다.
- [ ] 같은 key의 다른 payload가 거부된다.
- [ ] 다른 actor가 같은 key의 결과를 볼 수 없다.
- [ ] unique race의 패자가 winner row를 다시 읽는다.
- [ ] durable이 아닌 실패는 저장되지 않아 다음 요청이 다시 시도할 수 있다.
- [ ] 저장된 rejection의 code와 detail이 바뀌지 않는다.
- [ ] cache가 없어도 정합성이 유지되는 설계다.

### 구현 후 설명할 것

1. key만 비교하지 않고 actor와 fingerprint를 함께 묶은 이유
2. unique constraint를 동시성 제어 primitive로 사용한 이유
3. 확정적 실패와 일시적 실패의 영속 정책
4. 충돌 후 reread가 필요한 이유와 transaction isolation의 영향
5. legacy fingerprint가 없는 row를 허용할 때의 보안·호환성 trade-off

### 원본 확인 위치

- Thread: 05, 내구성 있는 멱등성 namespace와 정확한 결과 재생
- 커밋: `feat(placement): own idempotency request namespace`, `feat(placement): create durable request outcomes`, `feat(placement): claim durable request keys`, `feat(placement): persist request ownership`, `feat(placement): replay durable outcomes safely`
- 파일: `PlacementRequest.java`, `PlacementRequestRepository.java`, `PlacementReplay.java`, `BetStore.java`, `BetPlacementService.java`
- 함수·컴포넌트: `BetPlacementService.place(...)`, `PlacementReplay.bet(...)`, `PlacementReplay.request(...)`, `BetStore.savePending(...)`, `BetStore.savePreflightRejection(...)`
- 관련 Thread: 07, 08, 09, 14

---

<a id="t06-t07-remote-evidence"></a>
## [Thread 06·07 / 대표 커밋 `feat(wallet): debit full exposure idempotently`] 원격 성공을 검증 가능한 증거로 채택하기

### 면접 질문

리스크 예약과 지갑 debit은 모두 원격 side effect입니다. HTTP가 2xx라는 사실만으로 성공을 채택하지 않고, 리스크의 명시적 approval·reservation token과 지갑의 operation proof를 검증한 이유는 무엇입니까? `IDEMPOTENCY_CONFLICT`가 왔을 때 왜 무조건 성공이나 실패로 결정하지 않고 authoritative lookup을 수행합니까?

꼬리 질문:

- JSON에서 primitive `boolean` 대신 nullable `Boolean`을 사용해 missing verdict를 구분한 이유는 무엇입니까?
- 리스크 release에서 204와 409가 각각 어떤 확정적 의미를 가질 수 있습니까?
- 지갑 응답에서 operation ID만 확인하면 부족한 이유는 무엇입니까?
- debit POST의 201, 202, 204를 성공으로 넓게 인정하지 않은 이유는 무엇입니까?
- lookup의 404가 "operation 없음"인지 "account 없음"인지 어떻게 구분해야 합니까?
- circuit breaker fallback이 도메인 거부까지 service unavailable로 덮어쓰면 왜 문제가 됩니까?
- 원격 side effect는 성공했지만 응답이 유실된 경우 어떤 증거로 복구합니까?

### 30초 모범 답변

원격 호출은 응답 유실과 재시도가 있으므로 상태 코드만으로는 실제 side effect를 확정할 수 없습니다. 리스크는 `approved`, 상태, 만료 시각, canonical token을 모두 검증하고, 지갑은 operation ID뿐 아니라 actor, 금액, reason이 요청과 일치하는 proof만 채택합니다. 멱등성 충돌은 이미 같은 키로 작업이 있었음을 뜻할 뿐 그 작업이 내 요청과 같은지는 보장하지 않으므로 authoritative lookup으로 증거를 다시 읽습니다. missing·mismatch는 성공 추정이 아니라 격리하거나 dependency failure로 처리합니다.

### 답변 핵심 키워드

`remote operation evidence`, `explicit verdict`, `proof matching`, `idempotency conflict recovery`, `authoritative lookup`, `fail closed`, `ambiguous outcome`, `strict status contract`, `circuit breaker classification`

### 백지 구현

**구현 목표**

지갑형 원격 작업의 응답 proof를 검증하고, idempotency conflict 시 authoritative lookup 결과만 채택하는 순수 로직을 구현한다. 같은 구조를 리스크 reservation에도 일반화할 수 있어야 한다.

**인터페이스 또는 함수 시그니처**

```java
record OperationRequest(UUID key, UUID actorId, long amount, String reason) {}
record OperationProof(UUID operationId, UUID actorId, long amount, String reason) {}

sealed interface RemoteReply permits SuccessReply, ConflictReply, ProblemReply {}
record SuccessReply(int httpStatus, OperationProof proof) implements RemoteReply {}
record ConflictReply() implements RemoteReply {}
record ProblemReply(int httpStatus, String errorCode, String detail) implements RemoteReply {}

interface OperationLookup {
  Optional<OperationProof> find(UUID key);
}

final class EvidenceResolver {
  UUID resolve(OperationRequest request, RemoteReply reply, OperationLookup lookup) {
    // 직접 구현
    return null;
  }
}
```

**입력과 출력**

- 입력: 원래 요청, 원격 직접 응답, authoritative lookup
- 출력: 검증된 operation ID

**반드시 만족해야 할 조건**

- 직접 성공은 계약에서 허용한 정확한 status에서만 채택한다.
- proof의 operation ID, actor, amount, reason이 모두 유효해야 한다.
- conflict에서는 lookup을 수행하고 동일한 proof 검증을 반복한다.
- conflict 뒤 lookup 결과가 없거나 불일치하면 성공으로 간주하지 않는다.
- `operation not found`와 다른 404 문제를 분리할 수 있어야 한다.
- 금액은 양수이고 overflow 없는 정수 단위로 비교한다.
- 원격의 예상 밖 문제는 definitive rejection으로 임의 변환하지 않는다.

**경계 조건**

- 정상 proof
- null proof 또는 null operation ID
- 다른 actor
- 금액 1 차이
- debit 대신 refund reason
- conflict 뒤 정확한 proof
- conflict 뒤 lookup empty
- lookup 자체 실패
- 계약 밖의 2xx status

**실패 조건**

- proof mismatch: 외부 증거 충돌로 격리
- lookup unavailable: 일시적 dependency failure
- 계약상 확정 거부: 도메인 rejection
- 예상 밖 status·error: dependency/protocol failure

**제약**

HTTP 클라이언트 구현은 제외한다. 예외 타입 또는 결과 타입으로 실패 분류를 명확히 표현한다.

### 구현 후 자가 검증

- [ ] 정확한 proof만 operation ID로 채택된다.
- [ ] operation ID가 있어도 actor·amount·reason mismatch는 거부된다.
- [ ] conflict가 곧 성공으로 처리되지 않는다.
- [ ] conflict 뒤 정확한 lookup proof는 안전하게 채택된다.
- [ ] lookup empty와 lookup failure가 구분된다.
- [ ] 계약 밖의 2xx가 성공으로 새지 않는다.
- [ ] 같은 검증 규칙을 direct reply와 lookup reply에 재사용한다.
- [ ] 원격 도메인 거부가 circuit breaker fallback에 의해 다른 오류로 덮이지 않는다.

### 구현 후 설명할 것

1. 상태 코드와 비즈니스 증거를 별도로 검증한 이유
2. proof에 actor·amount·reason까지 포함해야 하는 이유
3. idempotency conflict를 복구 가능한 ambiguity로 본 이유
4. strict success status가 공급자 변경에 민감하지만 안전한 이유
5. 리스크와 지갑에서 공통 추상화할 부분과 각각 남겨야 할 도메인 차이

### 원본 확인 위치

- 대표 Thread: 07, 지갑 작업 증거와 멱등성 충돌 복구
- 통합 Thread: 06, 증거 기반 리스크 예약 수명 주기
- Thread 06 커밋: `feat(risk): define reservation wire contract`, `feat(risk): reserve full betting exposure`, `feat(risk): require explicit reservation approval`, `feat(risk): bound dependency failures with circuit breaker`, `feat(database): persist risk reservation tokens`
- Thread 07 커밋: `feat(wallet): debit full exposure idempotently`, `feat(wallet): refund locked exposure idempotently`
- 파일: `RiskClient.java`, `RiskReservationRequest.java`, `RiskReservationResponse.java`, `WalletClient.java`, `WalletOperationResponse.java`, `WalletProblemMapper.java`
- 함수·컴포넌트: `RiskClient.reserve(...)`, `RiskClient.commit(...)`, `RiskClient.release(...)`, `WalletClient.debit(...)`, `WalletClient.findDebit(...)`, `WalletClient.refund(...)`
- 관련 Thread: 05, 08, 10, 11
