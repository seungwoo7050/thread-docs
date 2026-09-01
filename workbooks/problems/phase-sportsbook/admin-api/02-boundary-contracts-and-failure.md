# 시스템 경계, 성공 계약과 실패 복구

이 문서는 마스터 인덱스의 W05~W08을 다룬다. 네 항목은 "하위 호출을 했는가"보다 "결과를 얼마나 확실히 아는가", "그 결과를 어떤 상태로 기록할 것인가"를 중심으로 연결된다.

<a id="W05"></a>
## [Thread 07 / `feat(client): classify ambiguous downstream failures`; `feat(api): map unknown downstream outcomes`] downstream 실패 확실성 분류

### 면접 질문

하위 서비스가 `4xx`, `5xx`, read timeout, connection failure를 반환했을 때 왜 같은 "호출 실패"로 처리하지 않았습니까? 특히 변경 요청의 timeout을 자동 재시도하지 않은 이유를 설명해 보세요.

꼬리 질문:

- `4xx`는 왜 명시적 거절로 볼 수 있고, `5xx`는 왜 명령 미적용을 보장하지 못합니까?
- read timeout이 wrapper exception 여러 단계 아래에 있을 때 어떻게 찾습니까?
- JSON 역직렬화 실패는 transport failure와 contract failure 중 어디에 속합니까?
- downstream `4xx`의 status, content type, body를 보존할 때 어떤 방어적 복사가 필요합니까?
- timeout을 외부 API `504`, 다른 불명확 실패를 `502`로 나누는 이유는 무엇입니까?
- 재시도를 허용하려면 idempotency 외에 어떤 조건과 관측 정보가 필요합니까?

### 30초 모범 답변

`4xx`는 하위 서비스가 요청을 수신해 명시적으로 거절한 결과라 `FAILED`로 볼 수 있지만, `5xx`, timeout, transport failure는 명령이 적용된 뒤 응답만 잃었을 수도 있어 `UNKNOWN`입니다. 그래서 원인 사슬에서 timeout을 구분하고, malformed success는 계약 위반으로 별도 분류하며, 변경 요청을 client 계층에서 자동 재시도하지 않습니다. API는 명시적 `4xx`를 중계하고 timeout은 `504`, 나머지 불명확·계약 위반은 `502`로 표현합니다. 재시도는 호출자가 같은 idempotency key로 명시적으로 결정해야 중복 적용 위험을 통제할 수 있습니다.

### 답변 핵심 키워드

`failure certainty` · `explicit rejection` · `unknown outcome` · `response lost after commit` · `cause-chain traversal` · `no blind retry` · `idempotency` · `4xx relay` · `504 timeout` · `502 bad gateway` · `audit UNKNOWN`

### 백지 구현

#### 구현 목표

하위 호출 exception을 결과 확실성 기준으로 분류하고, 외부 API 및 감사 상태 결정을 생성한다. 실제 HTTP client는 구현하지 않는다.

#### 인터페이스 또는 함수 시그니처

```java
public enum FailureKind {
  EXPLICIT_REJECTION,
  TIMEOUT,
  SERVER_ERROR,
  TRANSPORT,
  CONTRACT_VIOLATION
}

public record FailureDecision(
    FailureKind kind,
    int apiStatus,
    String auditOutcome,
    Optional<Integer> downstreamStatus,
    Optional<String> contentType,
    byte[] responseBody) {}

public final class DownstreamFailureClassifier {
  public static FailureDecision classify(Throwable failure) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: HTTP client 또는 converter가 던진 exception. 테스트용 custom exception에 downstream status/content type/body를 담아도 된다.
- 출력: 실패 종류, 외부 API status, 감사 outcome, 보존 가능한 downstream 응답 정보

#### 반드시 만족해야 할 조건

- downstream `4xx`는 `EXPLICIT_REJECTION`, 외부 status 동일, 감사 outcome `FAILED`다.
- `4xx`의 content type과 body를 보존하되 반환 객체가 원본 mutable byte array를 공유하지 않는다.
- downstream `5xx`는 `SERVER_ERROR`, 외부 `502`, 감사 outcome `UNKNOWN`이다.
- cause chain 어디에든 `SocketTimeoutException`이 있으면 `TIMEOUT`, 외부 `504`, 감사 `UNKNOWN`이다.
- converter/역직렬화 실패가 cause chain에 있으면 `CONTRACT_VIOLATION`, 외부 `502`, 감사 `UNKNOWN`이다.
- 그 밖의 연결·I/O 실패는 `TRANSPORT`, 외부 `502`, 감사 `UNKNOWN`이다.
- exception message나 출력에 downstream body 전체를 자동 포함하지 않는다.
- classifier는 호출을 재실행하지 않는다.
- 원인 사슬이 순환하는 비정상 exception에도 무한 반복하지 않는다.

#### 경계 조건

- status `400`, `404`, `409`, `499`, `500`, `503`
- timeout이 최상위 원인인 경우와 3단계 안쪽인 경우
- converter exception과 timeout이 동시에 감싼 비정상 chain
- null content type, empty body
- 호출자가 원본 body array를 나중에 변경하는 경우
- cause chain 순환

#### 실패 조건

- 분류할 수 없는 null exception
- 상태코드가 HTTP 범위를 벗어난 custom exception
- 명시적 응답이라고 표시됐지만 status 정보가 없음

#### 필요한 제약

- 분류 시간은 cause chain 길이에 선형이어야 한다.
- body는 defensive copy한다.
- 자동 retry, sleep, network I/O를 수행하지 않는다.

### 구현 후 자가 검증

- [ ] `409`가 status/body/content type을 유지한 `FAILED`로 분류된다.
- [ ] `503`이 명령 미적용으로 단정되지 않고 `UNKNOWN`이 된다.
- [ ] wrapped timeout이 `504`로 분류된다.
- [ ] 일반 connection failure는 `502` transport로 분류된다.
- [ ] malformed JSON converter exception은 contract violation이다.
- [ ] 반환 body를 수정해도 원본 exception의 body가 변하지 않는다.
- [ ] 원본 body를 수정해도 분류 결과가 변하지 않는다.
- [ ] exception text에 민감한 response body가 노출되지 않는다.
- [ ] supplier나 네트워크 호출 횟수가 늘어나지 않는다.
- [ ] cause cycle에서도 종료한다.

### 구현 후 설명할 것

1. HTTP 성공/실패가 아니라 "명령 결과 확실성"으로 분류한 이유
2. `4xx`와 `5xx`의 감사 outcome을 다르게 둔 이유
3. timeout 자동 재시도를 금지한 이유
4. cause chain 탐색 우선순위가 필요한 이유
5. downstream body 보존과 비밀정보 노출 방지의 trade-off

### 원본 확인 위치

- Thread 07
- `feat(client): classify ambiguous downstream failures`
- `feat(client): recognize wrapped read timeouts`
- `feat(api): map unknown downstream outcomes`
- `DownstreamFailureMapper`
- `DownstreamStatusException`
- `DownstreamUnavailableException`
- `AdminExceptionHandler`
- `Rfc7807Writer`
- 관련 Thread: 05, 09, 11

---

<a id="W06"></a>
## [Thread 08 / `feat(client): reject malformed success responses`; `feat(client): classify malformed success bodies`] 구조·의미 성공 계약

### 면접 질문

하위 서비스가 `2xx`를 반환했는데 왜 그대로 성공 처리하지 않고 exact status, body 존재 여부, 도메인 proof를 다시 검증했습니까? `204` 응답에 body가 있거나 `200` body가 역직렬화되지만 요청과 상관없는 값이면 각각 어떻게 처리해야 합니까?

꼬리 질문:

- `200`과 `202`를 모두 2xx로 묶지 않고 exact status를 요구할 때 얻는 이점은 무엇입니까?
- body가 null인 경우와 zero-length body인 경우를 어떻게 구분합니까?
- DTO 필드가 모두 non-null이어도 semantic proof가 필요한 이유는 무엇입니까?
- predicate 기반 공통 계약과 도메인별 `verify` 메서드의 책임은 어떻게 나눕니까?
- 역직렬화 실패가 `retrieve()` 단계에서 발생하면 공통 계약에 도달하기 전에 어떻게 분류합니까?
- 계약 위반을 재시도 대상으로 보지 않는 이유는 무엇입니까?

### 30초 모범 답변

HTTP `2xx`는 전송 계층의 성공일 뿐, 호출자가 기대한 명령 계약이 지켜졌다는 증거는 아닙니다. 공통 계층은 exact status와 body 존재·비어 있음 같은 구조 계약을 검사하고, 도메인 verifier는 요청 ID·금액·상태 조합 같은 의미 계약을 검사합니다. `204`에 body가 있거나 `200` body가 요청과 상관없으면 모두 downstream contract violation으로 `502` 처리하며 감사 결과는 `UNKNOWN`입니다. 역직렬화 실패도 같은 계약 위반으로 분류해 잘못된 성공을 내부 상태로 확정하지 않습니다.

### 답변 핵심 키워드

`2xx is not proof` · `exact status` · `body presence` · `empty response` · `structural contract` · `semantic proof` · `request-response correlation` · `deserialization failure` · `contract violation` · `UNKNOWN`

### 백지 구현

#### 구현 목표

typed body 응답과 empty 응답의 공통 구조 계약을 검증하는 작은 utility를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public record HttpResult<T>(int status, T body, byte[] rawBody) {}

public final class SuccessContract {
  public static <T> T requireBody(
      HttpResult<T> result,
      int expectedStatus,
      Predicate<T> semanticProof) {
    // 직접 구현
  }

  public static void requireEmpty(
      HttpResult<?> result,
      int expectedStatus) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- `requireBody`: HTTP 결과, exact expected status, 도메인 proof. 성공하면 body 반환.
- `requireEmpty`: HTTP 결과와 exact expected status. 성공하면 반환값 없음.

#### 반드시 만족해야 할 조건

- 두 함수 모두 status가 expected status와 정확히 같아야 한다.
- `requireBody`는 typed body가 null이면 실패한다.
- `requireBody`는 semantic proof가 `true`여야 한다.
- `requireEmpty`는 typed body가 없어야 한다.
- `requireEmpty`는 raw body가 null 또는 길이 0인 경우만 허용한다.
- status가 맞아도 body 계약이 틀리면 contract exception을 던진다.
- contract exception message는 실제 body나 credential을 포함하지 않는다.
- utility는 response body를 수정하지 않는다.
- converter에서 이미 발생한 decoding exception은 호출자가 contract violation으로 분류할 수 있는 별도 오류 유형으로 감싸야 한다.

#### 경계 조건

- expected `200`인데 actual `201`, `202`, `204`
- body null, empty object, raw zero bytes
- `204` + whitespace bytes
- semantic proof false
- proof가 요청 ID mismatch를 발견한 경우
- raw body와 typed body가 모순되는 test fixture

#### 실패 조건

- result null
- invalid HTTP status 값
- proof null
- proof 실패
- empty 계약에 body 존재

#### 필요한 제약

- 공통 utility는 도메인 필드 이름을 알지 않는다.
- proof 실행은 한 번만 한다.
- 시간·공간 복잡도는 body 자체의 도메인 검증을 제외하면 `O(1)`이어야 한다.

### 구현 후 자가 검증

- [ ] exact `200` + valid body가 통과한다.
- [ ] 같은 2xx라도 expected와 다른 status는 실패한다.
- [ ] null body가 실패한다.
- [ ] proof false가 실패한다.
- [ ] exact `204` + null/zero body가 통과한다.
- [ ] `204` + 한 byte라도 있으면 실패한다.
- [ ] 오류 메시지에 raw body가 들어가지 않는다.
- [ ] proof가 정확히 한 번 실행된다.
- [ ] 도메인 verifier를 교체해도 공통 utility가 수정되지 않는다.
- [ ] decoding failure를 transport failure로 오분류하지 않는다.

### 구현 후 설명할 것

1. exact status를 계약 일부로 본 이유
2. 구조 검증과 의미 검증을 두 계층으로 나눈 이유
3. `204` body를 관대하게 무시하지 않은 이유
4. contract violation을 `UNKNOWN`으로 감사하는 이유
5. predicate abstraction의 장점과 지나치게 일반화할 때의 단점

### 원본 확인 위치

- Thread 08
- `feat(client): reject malformed success responses`
- `feat(client): classify malformed success bodies`
- `DownstreamContract`
- `DownstreamContractException`
- `DownstreamFailureMapper`
- 관련 Thread: 16, 17, 19, 20

---

<a id="W07"></a>
## [Thread 16 / `feat(wallet): delegate refunds with exact proof`; `test(wallet): send exact refund request`] 검증 가능한 wallet refund

### 면접 질문

wallet가 `200`과 JSON body를 반환했는데, 어떤 필드가 맞아야 관리자 환불이 실제로 적용되었다는 증거로 인정할 수 있습니까? 요청 payload의 `source=HOUSE_POOL`, `reason=REFUND`와 응답의 ledger reason `BET_REFUND`가 각각 어떤 역할을 합니까?

꼬리 질문:

- operation group ID만 있으면 충분하지 않은 이유는 무엇입니까?
- 요청 user가 다르거나 금액은 같지만 currency가 다르면 어떻게 처리합니까?
- 응답 timestamp를 필수로 둔 이유는 무엇입니까?
- 외부 operator reason과 wallet ledger reason을 섞지 않아야 하는 이유는 무엇입니까?
- raw idempotency key를 변형 없이 전달해야 하는 이유는 무엇입니까?
- proof 실패 후 같은 key로 조회 또는 재시도를 설계한다면 어떤 추가 API가 필요합니까?

### 30초 모범 답변

환불 성공은 단순 `200`이 아니라 요청과 연결되는 완전한 ledger 증거여야 합니다. operation group ID가 존재하고, user·amount·currency가 요청과 정확히 같으며, ledger reason이 `BET_REFUND`이고 처리 시각이 있어야 합니다. 하위 요청은 관리자 서비스가 통제하는 `HOUSE_POOL` source와 `REFUND` reason을 고정하고, 외부 idempotency key는 그대로 전달합니다. 하나라도 불일치하면 잘못된 계정·통화·거래 응답일 수 있으므로 성공으로 확정하지 않고 contract violation으로 처리합니다.

### 답변 핵심 키워드

`monetary proof` · `operation group` · `exact user` · `exact amount/currency` · `fixed source` · `fixed ledger reason` · `timestamp evidence` · `raw idempotency key` · `correlation` · `fail closed`

### 백지 구현

#### 구현 목표

환불용 wallet command를 생성하고, wallet response가 그 command의 완전한 증거인지 검증한다.

#### 인터페이스 또는 함수 시그니처

```java
public record Money(long amount, String currency) {}

public record WalletCredit(
    UUID userId,
    Money amount,
    String source,
    String reason) {}

public record WalletProof(
    UUID operationGroupId,
    UUID userId,
    Money amount,
    String reason,
    Instant at) {}

public final class RefundProof {
  public static WalletCredit command(UUID userId, Money amount) {
    // 직접 구현
  }

  public static UUID verify(WalletCredit command, WalletProof proof) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- `command`: 환불 대상 user와 money를 받아 내부 wallet credit command 생성
- `verify`: command와 response proof를 받아 검증된 operation group ID 반환

#### 반드시 만족해야 할 조건

- command user와 money는 null이 아니어야 한다.
- amount는 양수여야 한다.
- command source는 환불에 고정된 값이어야 한다.
- command reason은 환불에 고정된 값이어야 한다.
- proof와 operation group ID는 null이면 안 된다.
- proof user는 command user와 정확히 같아야 한다.
- proof amount와 currency는 command와 정확히 같아야 한다.
- proof reason은 wallet가 반환하는 환불 ledger reason과 정확히 같아야 한다.
- proof 처리 시각은 null이면 안 된다.
- 검증 실패 시 operation group ID를 반환하지 않는다.
- idempotency key 전달 로직을 붙일 경우 원문을 정규화하거나 새 key로 바꾸지 않는다.

#### 경계 조건

- amount 0, 1
- 같은 amount의 다른 currency
- 같은 operation group이지만 다른 user
- null timestamp
- reason 대소문자·공백 차이
- proof null

#### 실패 조건

- invalid command
- incomplete proof
- request-response mismatch
- 예상하지 않은 ledger reason

#### 필요한 제약

- 금액 비교에서 부동소수점을 사용하지 않는다.
- proof를 수정하지 않는다.
- 검증은 `O(1)`이어야 한다.

### 구현 후 자가 검증

- [ ] 정상 command에 고정 source/reason이 들어간다.
- [ ] 완전한 proof가 operation group ID를 반환한다.
- [ ] user mismatch가 거부된다.
- [ ] amount mismatch와 currency mismatch가 각각 거부된다.
- [ ] operation group ID 누락이 거부된다.
- [ ] timestamp 누락이 거부된다.
- [ ] 다른 ledger reason이 거부된다.
- [ ] amount 0이 command 생성 단계에서 거부된다.
- [ ] 검증 실패가 partial success를 반환하지 않는다.
- [ ] idempotency key를 연결한 테스트에서 raw 값이 유지된다.

### 구현 후 설명할 것

1. 금전 변경에서 일반 2xx 검증보다 강한 proof를 요구한 이유
2. 요청 payload의 고정 필드와 operator 입력을 분리한 이유
3. user·amount·currency를 모두 상관 검증한 이유
4. timestamp와 operation group의 감사 가치
5. proof 실패 시 자동 재시도를 하지 않는 이유

### 원본 확인 위치

- Thread 16
- `test(wallet): reject incomplete refund proofs`
- `feat(wallet): delegate refunds with exact proof`
- `test(wallet): provide exact request capture fixture`
- `test(wallet): send exact refund request`
- `WalletOperationProof`
- `WalletOperationResponse`
- `WalletCreditPayload`
- `WalletClient`
- `WalletClientExactRequestTest`
- 관련 Thread: 05, 06, 08

---

<a id="W08"></a>
## [Thread 09·11 / `feat(audit): weave the fail-closed action lifecycle`; `feat(audit): order audit before method authorization`] end-to-end fail-closed 명령 파이프라인

### 면접 질문

관리자 변경 명령을 실행할 때 감사 `STARTED` insert, 권한 검사, 하위 호출, terminal update의 순서를 어떻게 정했고, 왜 감사 DB의 시작 또는 종료 기록 실패가 API 성공보다 우선합니까?

꼬리 질문:

- `STARTED` insert가 실패하면 하위 호출을 실행하면 안 되는 이유는 무엇입니까?
- 하위 호출은 실패했고 terminal update도 실패한 경우 어떤 exception을 밖으로 내보내고 원래 실패는 어떻게 보존합니까?
- method authorization denial까지 감사하려면 aspect ordering을 어떻게 잡아야 합니까?
- `@ResponseStatus(202)` 같은 선언된 성공 status를 audit에 어떻게 반영합니까?
- timeout과 malformed success를 `FAILED`가 아니라 `UNKNOWN`으로 기록하는 이유는 무엇입니까?
- 통합 테스트에서 `STARTED`가 하위 응답 전 외부 DB에서 관찰되는 것을 왜 확인했습니까?

### 30초 모범 답변

감사 대상 메서드에 들어가기 전에 별도 transaction으로 `STARTED`를 먼저 저장하고, 실패하면 하위 명령을 실행하지 않습니다. 이후 권한 검사와 명령 실행을 aspect가 감싸며 결과를 `SUCCESS`, 명시적 거절을 `FAILED`, 불명확 결과를 `UNKNOWN`으로 분류해 guarded terminal update를 한 번 수행합니다. terminal update가 실패하면 이미 실행된 명령의 결과를 신뢰할 수 없으므로 성공 응답 대신 finalization 오류를 내고, 원래 application failure는 suppressed context로 보존합니다. 이 순서를 통합 테스트에서 DB 상태·응답 action ID·하위 요청 identity까지 함께 검증합니다.

### 답변 핵심 키워드

`fail closed` · `STARTED before side effect` · `separate transaction` · `AOP ordering` · `authorization denial audited` · `terminal classification` · `guarded completion` · `finalization failure wins` · `suppressed exception` · `externally visible STARTED`

### 백지 구현

#### 구현 목표

감사 저장소와 실제 operation을 조합해 fail-closed lifecycle을 보장하는 orchestration 함수를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface AuditStore {
  void begin(UUID actionId);
  void complete(UUID actionId, String outcome, Integer httpStatus);
}

public interface AuditedOperation<T> {
  T run() throws Exception;
}

public interface OutcomeClassifier<T> {
  Decision fromResult(T result);
  Decision fromFailure(Throwable failure);
}

public record Decision(String outcome, Integer httpStatus) {}

public final class FailClosedExecutor {
  public static <T> T execute(
      UUID actionId,
      AuditStore audits,
      AuditedOperation<T> operation,
      OutcomeClassifier<T> classifier) throws Exception {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: action ID, 감사 저장소, 실제 operation, 결과 분류기
- 출력: 성공 시 operation 결과. 실패 시 원래 operation 또는 감사 예외.

#### 반드시 만족해야 할 조건

- `begin`은 operation보다 먼저 정확히 한 번 호출된다.
- `begin`이 실패하면 operation과 `complete`를 호출하지 않는다.
- operation은 최대 한 번 실행된다.
- operation 결과 또는 실패를 classifier로 정확히 한 번 분류한다.
- `complete`는 operation이 시작된 경우 정확히 한 번 시도한다.
- `complete`가 성공하고 operation이 실패했다면 원래 failure를 다시 던진다.
- `complete`가 실패하면 finalization failure를 우선 던진다.
- operation failure와 complete failure가 모두 있으면 operation failure를 suppressed로 보존한다.
- 성공 결과는 complete가 성공한 뒤에만 반환한다.
- classifier 실패도 불명확 outcome으로 terminal 처리할 수 있도록 정책을 명확히 정한다.
- authorization denial을 operation 내부 failure로 전달하면 `FAILED`로 terminal 처리할 수 있어야 한다.

#### 경계 조건

- begin 실패
- operation 성공 + complete 성공
- operation 명시적 4xx failure + complete 성공
- operation timeout + complete 성공
- operation 성공 + complete 실패
- operation 실패 + complete 실패
- classifier 자체 실패
- operation이 null을 성공 결과로 반환

#### 실패 조건

- action ID 또는 dependency 누락
- begin/complete persistence exception
- operation exception
- outcome 분류 불가

#### 필요한 제약

- operation과 각 persistence 단계의 호출 횟수를 테스트 가능하게 해야 한다.
- 예외를 message 문자열로 분류하지 않는다.
- 원래 stack trace를 잃지 않는다.
- cleanup이 필요한 operation이라면 그 책임 경계를 명시한다.

### 구현 후 자가 검증

- [ ] begin 실패 시 side effect가 전혀 실행되지 않는다.
- [ ] 정상 경로 순서가 `begin → operation → complete → return`이다.
- [ ] operation failure가 complete 뒤 원래 타입으로 다시 던져진다.
- [ ] complete failure가 성공 결과를 밖으로 내보내지 않는다.
- [ ] 이중 실패에서 suppressed exception이 정확히 하나 보존된다.
- [ ] complete는 모든 operation 경로에서 최대 한 번이다.
- [ ] timeout/5xx/contract failure가 `UNKNOWN`으로 분류된다.
- [ ] explicit 4xx와 authorization denial이 `FAILED`로 분류된다.
- [ ] 선언된 `202` 성공이 `200`으로 잘못 기록되지 않는다.
- [ ] 동시 지연 테스트에서 operation 완료 전 `STARTED`가 관찰 가능하다.
- [ ] 응답·DB·하위 요청이 동일 action ID로 상관된다.

### 구현 후 설명할 것

1. 감사 실패가 비즈니스 성공보다 우선하는 이유
2. `REQUIRES_NEW` 성격의 시작/종료 transaction이 필요한 이유
3. AOP가 method security보다 바깥에서 실행되어야 하는 이유
4. 이중 실패에서 finalization failure를 대표 exception으로 선택한 이유
5. 단위 테스트와 실제 PostgreSQL·HTTP 통합 테스트가 각각 잡는 오류

### 원본 확인 위치

- Thread 09
- `feat(audit): weave the fail-closed action lifecycle`
- `feat(error): fail closed when audit finalization fails`
- `test(audit): correlate downstream action identities`
- Thread 11
- `feat(audit): surface persistence failure phases`
- `feat(audit): classify terminal admin outcomes`
- `feat(audit): honor declared response statuses`
- `feat(audit): order audit before method authorization`
- `AuditAspect`
- `AuditService`
- `AuditOutcomeClassifier`
- `AuditPersistenceException`
- `AdminExceptionHandler`
- `AuditHttpIntegrationTest`
- `AuditHttpTestEnvironment`
- `AuditMethodSecurityOrderingTest`
- 관련 Thread: 05, 07, 10, 12
