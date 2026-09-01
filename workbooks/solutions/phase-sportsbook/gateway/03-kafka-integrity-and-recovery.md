# Kafka 정합성과 복구 면접 워크북

raw record가 strict Avro decode와 의미 검증을 통과해 WebSocket으로 전달되거나, 같은 partition DLT에 격리되고 수동 replay되는 전체 경계를 다룬다. 가장 중요한 것은 원본 증거, offset, partition, key와 payload 사이의 불변조건이다.

## KAF-01 · [Thread 10 / `feat(kafka): consume raw event records`; `feat(kafka): classify event failures`] raw byte 소비와 permanent/transient 실패 분류

**우선순위:** A

### 면접 질문

- Kafka key와 value를 generated type으로 바로 deserialize하지 않고 둘 다 `byte[]`로 받은 이유는 무엇입니까?
- auto-commit을 끄고 `RECORD` acknowledgment를 사용하면 성공·실패 시 offset 진행이 어떻게 달라집니까?
- Avro·event contract 위반은 왜 재시도 없이 DLT로 보내고, STOMP 전달 같은 실패는 제한된 재시도를 거치게 했습니까?
- 꼬리 질문: 기본 retry attempts가 2라면 listener 총 호출 횟수는 몇 번이며 설정 이름을 어떻게 설명하겠습니까?

### 30초 모범 답변

raw `byte[]` 소비는 잘못된 key나 value도 consumer deserializer 단계에서 잃지 않고 같은 원본을 DLT 증거로 남기기 위한 선택입니다. auto-commit을 끄고 record ack를 쓰면 한 레코드의 처리·복구가 완료된 뒤에만 그 offset이 진행됩니다. 형식·정규형·key binding 위반은 같은 입력으로 성공할 수 없는 permanent failure라 즉시 격리하고, broker나 local delivery 같은 일시 실패는 bounded backoff 후 격리합니다. retry attempts 2는 최초 시도에 두 번을 더해 최대 세 번의 listener 실행을 뜻합니다.

### 답변 핵심 키워드

`raw evidence`, `byte[] deserializer`, `record ack`, `auto-commit off`, `permanent vs transient`, `bounded backoff`, `failure classification`

### 백지 구현

**구현 목표**

raw Kafka record 처리 실패를 permanent contract failure와 transient delivery failure로 분류하고, 시도 횟수에 따른 다음 행동을 결정한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
enum FailureKind { PERMANENT_CONTRACT, TRANSIENT_DELIVERY }

enum NextAction { RETRY, RECOVER_TO_DLT }

record RetryPolicy(long retryAttempts, Duration interval) {}

final class EventFailurePolicy {
  FailureKind classify(Throwable failure) {
    throw new UnsupportedOperationException("직접 구현");
  }

  NextAction next(FailureKind kind, long failedAttemptIndex, RetryPolicy policy) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: listener에서 발생한 예외와 nonnegative retry-attempts 정책
- 출력: 실패 종류와 RETRY 또는 RECOVER_TO_DLT
- `failedAttemptIndex`의 의미를 문서화하고 테스트에서 일관되게 사용

**반드시 만족해야 할 조건**

- 확인된 `GatewayEventContractException`은 permanent로 분류한다.
- 그 밖의 delivery 실패는 무조건 permanent로 추측하지 않는다.
- permanent는 첫 실패 후 바로 DLT 복구로 간다.
- transient는 설정된 추가 retry 수만큼만 재시도한 뒤 DLT로 간다.
- negative retry count와 nonpositive interval은 설정 오류다.

**경계 조건**

- contract exception이 cause chain 안에 감싸진 경우를 분류할지 정책으로 명시
- retryAttempts 0, 1, 2
- 마지막 허용 retry 직전과 직후의 attempt index
- 분류되지 않은 RuntimeException

**실패 조건**

- permanent 입력을 반복 재시도해 poison record로 partition을 오래 막지 않는다.
- unknown failure를 조용히 성공으로 취급하지 않는다.
- retry count off-by-one으로 총 호출 횟수가 늘어나지 않는다.

**필요한 제약**

- 실제 Kafka listener container API는 구현하지 않는다.
- sleep을 호출하지 않고 행동 결정만 반환한다.
- 원본 key/value를 decode하거나 변경하지 않는다.

### 구현 후 자가 검증

- [ ] contract failure가 첫 실패에서 DLT 행동으로 간다.
- [ ] retryAttempts 0에서 transient도 즉시 DLT로 간다.
- [ ] retryAttempts 2에서 최초 포함 총 세 번의 실패 뒤 DLT로 간다.
- [ ] unknown failure의 분류 정책이 명시적이다.
- [ ] negative count와 0 이하 interval이 거부된다.
- [ ] cause wrapping 정책에 대한 테스트가 있다.
- [ ] 분류 함수가 입력 예외를 변경하지 않는다.

### 구현 후 설명할 것

- raw bytes가 poison record의 운영 증거 보존에 주는 이점
- record ack와 batch ack의 실패 격리 차이
- 재시도 가능성을 예외 타입으로 표현할 때 생기는 결합
- retry 횟수 이름에서 발생하기 쉬운 off-by-one
- DLT도 실패할 수 있으므로 recovery 완료 전 ack하면 안 된다는 연결점

### 원본 확인 위치

- Thread 10
- 커밋: `chore(kafka): define event delivery defaults`
- 커밋: `feat(kafka): consume raw event records`
- 커밋: `test(kafka): verify record acknowledgment semantics`
- 커밋: `feat(kafka): define permanent contract failures`
- 커밋: `feat(kafka): classify event failures`
- 커밋: `test(kafka): bypass retries for contract failures`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfiguration.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayEventContractException.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaErrorConfiguration.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java`
- 테스트: `src/test/java/com/sportsbook/gateway/kafka/GatewayKafkaConsumerConfigurationTest.java`
- 관련 Thread: 11, 12, 13, 14, 16

## KAF-02 · [Thread 10 / `feat(kafka): route failures by source topic`; `feat(kafka): bound dead-letter publication`] same-partition DLT와 recovery 실패 시 offset 보존

**우선순위:** S

### 면접 질문

- 왜 DLT destination을 source topic의 `.DLT`와 정확히 같은 partition 번호로 정했습니까?
- DLT가 source보다 partition 수가 적어 publication이 실패하면 source offset을 커밋하지 않아야 하는 이유는 무엇입니까?
- `acks=all`, idempotent producer, bounded send wait를 함께 사용한 의도를 설명해 보세요.
- partition preflight를 생략하고 실제 send failure를 경계로 사용한 trade-off는 무엇입니까?

### 30초 모범 답변

같은 partition으로 격리하면 source partition 안의 순서·key locality와 장애 위치를 유지해 운영자가 원본을 추적하기 쉽습니다. DLT publish가 실패했는데 source offset을 커밋하면 원본과 격리본이 모두 사라지므로 recovery는 fail-closed여야 합니다. 그래서 send result 오류를 다시 container로 올리고 offset을 남겨 redelivery하게 하며, producer는 `acks=all`과 idempotence를 사용합니다. 다만 무한 대기는 gateway를 멈추므로 producer와 result wait에 상한을 두고, 운영에서 DLT partition 수를 source와 같게 사전 프로비저닝해야 합니다.

### 답변 핵심 키워드

`same partition`, `ordering locality`, `fail-closed recovery`, `offset invariant`, `acks=all`, `idempotent producer`, `bounded publication`

### 백지 구현

**구현 목표**

source record와 DLT publish 결과를 받아 destination과 source offset commit 가능 여부를 결정하는 recovery coordinator를 작성한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record RawRecord(
    String topic,
    int partition,
    long offset,
    byte[] key,
    byte[] value) {}

record TopicPartition(String topic, int partition) {}

interface DltPublisher {
  PublishResult publish(TopicPartition destination, RawRecord source);
}

final class RecoveryCoordinator {
  RecoveryOutcome recover(RawRecord source, Throwable failure) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: configured source record와 처리 실패
- 의존성: 성공/실패/timeout을 반환하는 DLT publisher
- 출력: DLT 성공 후 ACK 가능 또는 recovery failure 후 ACK 금지

**반드시 만족해야 할 조건**

- 정확히 등록된 source topic만 `<source>.DLT`로 매핑한다.
- destination partition은 source partition과 동일하다.
- publish 성공을 확인하기 전에는 ACK 가능 상태를 반환하지 않는다.
- publish 오류·timeout·잘못된 partition은 recovery failure로 반환한다.
- source key와 value bytes를 변환하지 않고 publisher에 전달한다.

**경계 조건**

- source partition 0과 큰 partition 번호
- 알 수 없는 source topic
- publisher가 즉시 예외, future 실패, timeout을 반환하는 경우
- 같은 record가 recovery failure로 여러 번 redelivery되는 경우

**실패 조건**

- DLT 실패를 성공 처리해 offset이 전진하지 않게 한다.
- 알 수 없는 topic을 임의 `.DLT` 이름으로 자동 수용하지 않는다.
- timeout 뒤 늦은 성공과 caller 결과 사이의 정책을 명확히 한다.

**필요한 제약**

- DLT topic 생성이나 partition 증설은 구현하지 않는다.
- source offset을 직접 조작하지 않고 ACK 가능 여부만 반환한다.
- publish 기다림은 설정된 유한 시간 안에 끝나야 한다.

### 구현 후 자가 검증

- [ ] 네 configured source가 올바른 `.DLT`로 매핑된다.
- [ ] source partition 번호가 그대로 보존된다.
- [ ] 알 수 없는 source와 이미 `.DLT`인 source가 거부된다.
- [ ] publish 성공에서만 ACK 가능하다.
- [ ] send failure·timeout·partition mismatch에서 ACK 금지다.
- [ ] recovery 실패 후 같은 source offset이 다시 처리될 수 있다.
- [ ] key/value byte 배열의 내용이 보존된다.

### 구현 후 설명할 것

- DLT 성공과 source offset commit 사이의 핵심 invariant
- same-partition이 순서와 조사 가능성에 주는 장점
- preflight 없는 실제 send 검증의 단순성·지연 trade-off
- idempotent producer가 consumer-to-DLT exactly-once를 보장하지는 않는 이유
- 운영 프로비저닝을 코드 invariant의 일부로 문서화해야 하는 이유

### 원본 확인 위치

- Thread 10
- 커밋: `feat(kafka): route failures by source topic`
- 커밋: `test(kafka): verify dead-letter topic mapping`
- 커밋: `feat(kafka): bound dead-letter publication`
- 커밋: `test(kafka): retain offsets when recovery fails`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayDeadLetterConfiguration.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayKafkaProperties.java`
- 테스트: `src/test/java/com/sportsbook/gateway/kafka/GatewayRecoveryFailureIntegrationTest.java`
- 관련 Thread: 12, 16

## KAF-03 · [Thread 11 / `feat(events): decode strict Avro records`] 정확히 한 개의 Avro record만 허용하는 strict decoder

**우선순위:** S

### 면접 질문

- Avro decoder가 객체 하나를 읽었다고 해서 payload 전체가 유효하다고 볼 수 없는 이유는 무엇입니까?
- trailing bytes를 계약 실패로 처리하지 않으면 어떤 producer 버그나 데이터 오염을 놓칠 수 있습니까?
- null, truncated, schema 불일치, trailing payload를 하나의 permanent contract exception으로 경계화한 이유는 무엇입니까?
- 꼬리 질문: 매 호출마다 `SpecificDatumReader`를 만드는 구현과 reader cache의 trade-off는 무엇입니까?

### 30초 모범 답변

binary decoder는 객체 하나를 읽고도 입력 뒤에 바이트가 남아 있을 수 있으므로 decode 성공만 확인하면 concatenated record나 잘못된 envelope를 수용하게 됩니다. 이 gateway 계약은 generated type의 정확히 한 레코드이므로 읽은 뒤 decoder가 끝인지 확인합니다. null·truncated·schema/런타임 decode 오류·trailing bytes는 같은 입력으로 재시도해도 고쳐지지 않는 permanent contract failure로 바꿔 DLT로 보냅니다. 구현은 단순성을 위해 reader를 매번 만들 수 있지만 고처리량에서는 type별 안전한 cache를 검토할 수 있습니다.

### 답변 핵심 키워드

`complete consumption`, `trailing bytes`, `SpecificDatumReader`, `binary decoder`, `permanent contract`, `schema boundary`, `reader cache trade-off`

### 백지 구현

**구현 목표**

주어진 generated Avro type으로 payload를 읽고, 정확히 한 레코드를 완전히 소비했을 때만 반환하는 decoder를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
final class StrictRecordDecoder {
  <T extends SpecificRecord> T decode(byte[] payload, Class<T> type) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: nullable raw payload, 기대하는 generated `SpecificRecord` class
- 출력: 완전히 decode된 한 객체
- 실패: 원인을 감싼 단일 event contract exception

**반드시 만족해야 할 조건**

- null payload를 거부한다.
- 기대 type의 datum reader로 한 레코드를 읽는다.
- 읽은 뒤 decoder가 입력 끝에 도달했는지 확인한다.
- truncated·malformed·runtime decode 실패를 contract exception으로 변환한다.
- 이미 만든 contract exception을 불필요하게 이중 wrapping하지 않는다.

**경계 조건**

- 정상 payload, 빈 배열, 마지막 byte가 잘린 payload
- 정상 payload 뒤 0 byte 하나가 붙은 경우
- 다른 generated type의 payload
- 유효한 레코드 두 개를 이어 붙인 payload

**실패 조건**

- 부분 객체를 반환하지 않는다.
- trailing bytes를 무시하지 않는다.
- payload 원문을 예외 메시지에 출력하지 않는다.

**필요한 제약**

- schema registry envelope나 framing은 추가하지 않는다.
- generic record가 아니라 확인된 generated specific type을 사용한다.
- 성능 최적화가 필요하면 reader의 thread-safety를 먼저 검토한다.

### 구현 후 자가 검증

- [ ] 정상 generated record 하나가 동일 내용으로 복원된다.
- [ ] null·empty·truncated·random bytes가 거부된다.
- [ ] 정상 뒤 한 byte와 두 record concatenation이 거부된다.
- [ ] 다른 type payload가 정상 객체로 오인되지 않는다.
- [ ] 예외 타입이 retry classifier가 permanent로 알아볼 수 있는 타입이다.
- [ ] 예외에 raw payload가 노출되지 않는다.
- [ ] reader cache를 사용했다면 동시 호출 안전성이 검증된다.

### 구현 후 설명할 것

- decode 성공과 complete consumption의 차이
- trailing byte 검사가 protocol framing 오류를 잡는 방식
- 모든 decode 실패를 permanent로 본 프로젝트 계약의 전제
- reader 생성 비용과 cache 복잡도의 trade-off

### 원본 확인 위치

- Thread 11
- 커밋: `feat(events): decode strict Avro records`
- 커밋: `test(events): reject malformed Avro records`
- 파일: `src/main/java/com/sportsbook/gateway/events/StrictAvroDecoder.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/StrictAvroDecoderTest.java`
- 테스트 지원: `src/test/java/com/sportsbook/gateway/events/AvroTestSupport.java`
- 관련 Thread: 10, 12, 13, 14

## KAF-04 · [Thread 11 / `feat(events): validate odds event identities`; `feat(events): validate terminal bet identities`] strict UTF-8 key와 payload identity 결합

**우선순위:** S

### 면접 질문

- Kafka key를 `new String(bytes, UTF_8)`로 만드는 것과 malformed input을 REPORT하도록 decoder를 설정하는 것은 무엇이 다릅니까?
- 왜 odds와 terminal bet에서 key를 `eventId`, revision에서 `betId`와 정확히 일치시켰습니까?
- payload 안 모든 식별자를 canonical UUID로 검증하는 것이 partition key 일치 검사와 별도로 필요한 이유는 무엇입니까?
- 꼬리 질문: key가 null이거나 mismatch인 record를 자동 보정해 다시 publish하지 않은 이유는 무엇입니까?

### 30초 모범 답변

기본 문자열 생성은 잘못된 UTF-8을 replacement character로 바꿀 수 있어 원본 key 오염을 숨깁니다. 그래서 malformed와 unmappable을 REPORT하고, 복원된 문자열이 payload의 계약상 identity와 byte 의미상 정확히 같아야 합니다. 동시에 payload의 betId·userId·eventId·marketId·selectionId도 canonical UUID인지 검증해 같은 ID의 여러 표현과 잘못된 user routing을 막습니다. key를 자동 보정하면 partition ordering과 producer 오류 증거를 바꾸므로 permanent failure로 격리합니다.

### 답변 핵심 키워드

`strict UTF-8`, `replacement character risk`, `partition-key binding`, `canonical IDs`, `ordering`, `no silent repair`, `routing integrity`

### 백지 구현

**구현 목표**

raw key와 이미 strict-decode된 event identity들을 검증해 partition key와 payload 의미가 일치하는지 판정한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record EventIdentity(
    String keyFieldName,
    String expectedKey,
    Map<String, String> uuidFields) {}

final class KafkaIdentityContract {
  void validate(byte[] rawKey, EventIdentity identity, String topic) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: nullable raw key, 기대 key 문자열, 이름별 UUID field, topic
- 출력: 성공 시 반환 없음
- 실패: event contract exception

**반드시 만족해야 할 조건**

- raw key가 반드시 존재해야 한다.
- UTF-8 malformed/unmappable 입력을 replacement 없이 거부한다.
- 모든 uuid field는 parse 후 canonical round-trip을 만족해야 한다.
- decoded key가 expectedKey와 정확히 같아야 한다.
- topic별 expectedKey 선택은 호출 전에 명시적으로 결정되어야 한다.

**경계 조건**

- null key, empty key, malformed 2-byte sequence
- 대문자 UUID key와 소문자 payload
- payload key field는 정상이나 다른 identity field가 invalid인 경우
- eventId와 betId가 우연히 같은 값인 revision record

**실패 조건**

- invalid key를 replacement 문자열로 계속 처리하지 않는다.
- mismatch를 payload나 key 중 하나로 조용히 수정하지 않는다.
- 오류 메시지에 전체 binary key를 출력하지 않는다.

**필요한 제약**

- Kafka partition hash를 재구현하지 않는다.
- UUID 외 business field 검증은 다음 항목과 분리한다.
- 필드별 expected key 규칙을 topic 이름 추측으로 확장하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 UTF-8 canonical key와 모든 UUID가 통과한다.
- [ ] null·empty·malformed UTF-8 key가 거부된다.
- [ ] 다른 canonical UUID key도 mismatch로 거부된다.
- [ ] 대문자·비정규형·비UUID payload field가 거부된다.
- [ ] odds/settled/voided는 eventId, revision은 betId 규칙을 사용한다.
- [ ] 입력 key 배열과 event 객체가 변경되지 않는다.
- [ ] 오류가 permanent contract exception으로 분류된다.

### 구현 후 설명할 것

- strict decoder가 Unicode replacement로 인한 충돌을 막는 방식
- partition key가 ordering과 user/event fan-out의 의미 계약인 이유
- canonical identity가 HTTP·JWT·Kafka·STOMP 전 경계에 반복되는 이유
- 자동 repair보다 DLT 격리를 선택한 데이터 거버넌스 판단

### 원본 확인 위치

- Thread 11
- 커밋: `feat(events): validate odds event identities`
- 커밋: `test(events): reject invalid odds identities`
- 커밋: `feat(events): validate terminal bet identities`
- 커밋: `test(events): reject invalid settled bet identities`
- 커밋: `test(events): reject invalid voided bet identities`
- 파일: `src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/GatewayOddsEventContractTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/GatewaySettledEventContractTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/GatewayVoidedEventContractTest.java`
- 관련 Thread: 3, 13, 14

## KAF-05 · [Thread 11 / `feat(events): validate resolution revisions`; `fix(events): reject market void lifecycle records`] revision snapshot과 terminal lifecycle 의미 불변조건

**우선순위:** S

### 면접 질문

- revision event에서 canonical ID와 key 일치 외에 revision number, currency, timestamp 관계를 왜 검증해야 합니까?
- `newPayout`의 currency가 `previousPayout`과 다르면 gateway가 격리해야 하는 이유는 무엇입니까?
- `revisedAt`이 원래 settlement 시각보다 앞설 수 없다는 조건은 어떤 정합성을 보호합니까?
- `BetVoided`의 `MARKET_VOID`를 거부하고 settled `VOID` 결과를 요구한 이유를 설명해 보세요.

### 30초 모범 답변

revision은 단순 알림이 아니라 최신 terminal snapshot을 갱신하는 사건이므로 revision number는 양수이고, 이전·새 payout의 currency가 같으며, revisedAt은 source settlement보다 이르지 않아야 합니다. 그렇지 않으면 클라이언트가 서로 비교할 수 없는 금액이나 시간 역전 상태를 받습니다. key는 betId와 묶어 같은 bet의 순서를 유지합니다. 또한 market void는 refund 중심 `BetVoided`가 아니라 settlement 결과 `VOID`로 표현하는 계약이므로 잘못된 lifecycle record는 즉시 격리합니다.

### 답변 핵심 키워드

`semantic invariant`, `positive revision`, `currency continuity`, `monotonic time`, `bet key`, `terminal lifecycle`, `market void contract`

### 백지 구현

**구현 목표**

축소된 revision/void event 모델이 gateway가 전달 가능한 의미 계약을 만족하는지 검증한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record Money(long amount, String currency) {}

record RevisionEvent(
    String revisionId,
    long revisionNumber,
    String betId,
    String userId,
    String eventId,
    Money previousPayout,
    Money newPayout,
    Instant sourceResultSettledAt,
    Instant revisedAt) {}

record VoidedEvent(String betId, String reason) {}

final class BetLifecycleContract {
  void validateRevision(RevisionEvent event, String decodedKafkaKey) {
    throw new UnsupportedOperationException("직접 구현");
  }

  void validateVoided(VoidedEvent event) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: revision snapshot과 strict UTF-8로 복원된 Kafka key
- 입력: voided event의 확인된 reason
- 출력: 성공 또는 permanent contract exception

**반드시 만족해야 할 조건**

- revisionId, betId, userId, eventId가 canonical UUID다.
- revisionNumber는 1 이상이다.
- previousPayout과 newPayout의 currency가 정확히 같다.
- `revisedAt >= sourceResultSettledAt`이다.
- revision Kafka key는 betId와 같다.
- `MARKET_VOID` reason을 가진 voided event는 거부한다.

**경계 조건**

- revisionNumber 0, 음수, 1
- timestamp가 같음, 1나노초 이전, 1나노초 이후
- currency 대소문자만 다른 경우
- null money 또는 timestamp가 가능한 축소 모델에서의 정책
- 다른 void reason

**실패 조건**

- 서로 다른 currency의 amount를 변환해 보정하지 않는다.
- 시간 역전을 clock skew로 추측해 허용하지 않는다.
- MARKET_VOID를 일반 void로 전달하지 않는다.

**필요한 제약**

- revision 연속 번호의 전역 누락 검사는 state store가 없으므로 범위 밖이다.
- 금액의 business-valid range는 현재 확인된 계약 이상으로 추측하지 않는다.
- event 객체를 수정해 정상화하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 revision과 같은 timestamp 경계가 통과한다.
- [ ] revision 0·음수가 거부된다.
- [ ] currency 불일치와 timestamp 역전이 거부된다.
- [ ] 모든 비정규형 ID와 betId key mismatch가 거부된다.
- [ ] MARKET_VOID voided event가 거부되고 다른 확인된 reason은 통과한다.
- [ ] 검증 실패가 DLT 직행 가능한 permanent exception이다.
- [ ] 상태 저장 없이 검증할 수 없는 revision gap을 거짓으로 보장하지 않는다.

### 구현 후 설명할 것

- schema 유효성과 semantic validity의 차이
- state 없는 gateway가 검증할 수 있는 invariant와 검증할 수 없는 invariant
- currency continuity와 시간 단조성이 클라이언트 snapshot 적용에 필요한 이유
- 한 비즈니스 사건을 두 event type으로 중복 표현하지 않게 한 lifecycle 계약

### 원본 확인 위치

- Thread 11
- 커밋: `feat(events): validate resolution revisions`
- 커밋: `test(events): reject invalid resolution revisions`
- 커밋: `fix(events): reject market void lifecycle records`
- 커밋: `test(events): quarantine market void records`
- 파일: `src/main/java/com/sportsbook/gateway/events/GatewayEventContract.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/GatewayResolutionRevisionContractTest.java`
- 테스트: `src/test/java/com/sportsbook/gateway/events/GatewayVoidedEventContractTest.java`
- 관련 Thread: 10, 14, 16

## KAF-06 · [Thread 12 / `feat(kafka): sanitize manual replay records`] DLT 수동 replay의 원본 복원과 framework header 정화

**우선순위:** A

### 면접 질문

- DLT record를 source로 replay할 때 key와 value를 clone해야 하는 이유는 무엇입니까?
- application header는 duplicate와 null value까지 보존하면서 recovery/exception/deserializer header만 제거한 이유를 설명해 보세요.
- `kafka_application`처럼 framework prefix와 비슷한 이름을 prefix 규칙으로 지우지 않은 이유는 무엇입니까?
- 왜 factory는 `ProducerRecord`만 만들고 자동 consume·publish나 consumer-group offset reset을 하지 않습니까?

### 30초 모범 답변

DLT는 원본 처리 실패의 증거이므로 replay는 raw key·value와 application header의 의미를 최대한 그대로 복원해야 합니다. 배열과 header 값을 clone해 source record의 mutable storage와 replay record가 별개가 되게 하고, 중복·null도 Kafka header 계약대로 보존합니다. 제거 대상은 알려진 Spring Kafka recovery·exception·delivery attempt·deserializer metadata의 정확한 이름뿐이며 prefix 추측으로 application evidence를 잃지 않습니다. 실제 publish는 운영자가 원인과 destination을 확인한 뒤 같은 partition tail에 명시적으로 수행하고 group offset은 건드리지 않습니다.

### 답변 핵심 키워드

`defensive copy`, `evidence preservation`, `exact header denylist`, `duplicate/null headers`, `source-DLT mapping`, `operator-controlled replay`, `no offset reset`

### 백지 구현

**구현 목표**

허용된 DLT record 하나를 paired source topic과 같은 partition으로 보낼 새 producer record로 변환한다.

**인터페이스 또는 함수 시그니처**

```java
// 직접 구현
record HeaderValue(String name, byte[] value) {}

record DeadLetterRecord(
    String topic,
    int partition,
    byte[] key,
    byte[] value,
    List<HeaderValue> headers) {}

record ReplayRecord(
    String topic,
    int partition,
    byte[] key,
    byte[] value,
    List<HeaderValue> headers) {}

final class DltReplayRecordFactory {
  ReplayRecord replay(DeadLetterRecord deadLetter) {
    throw new UnsupportedOperationException("직접 구현");
  }
}
```

**입력과 출력**

- 입력: 정확한 configured DLT 이름의 record
- 출력: paired source topic, 같은 partition, cloned payload와 sanitized headers
- 알 수 없는 DLT: 거부

**반드시 만족해야 할 조건**

- 허용된 네 DLT 이름만 정확히 source topic으로 역매핑한다.
- partition 번호를 보존한다.
- key와 value가 null이 아니면 새 배열로 복사한다.
- application header의 순서·중복·null 값과 byte 내용을 보존하고 non-null 값은 clone한다.
- 확인된 recovery·exception·delivery-attempt·deserializer exception header의 정확한 이름만 제거한다.

**경계 조건**

- null key/value/header value
- 같은 application header 여러 개
- framework 이름과 접두사만 비슷한 application header
- `source.DLT.DLT`, 대소문자가 다른 DLT, 알 수 없는 DLT
- 빈 byte 배열

**실패 조건**

- 잘못된 DLT를 이름 잘라내기로 source에 보내지 않는다.
- framework metadata가 replay record에 남아 새 failure history와 섞이지 않게 한다.
- 입력 배열·header 값과 출력이 같은 reference를 공유하지 않는다.

**필요한 제약**

- broker publish와 acknowledgment wait는 구현하지 않는다.
- consumer group offset을 읽거나 변경하지 않는다.
- application payload를 decode·검증·수정하지 않는다.

### 구현 후 자가 검증

- [ ] 네 정확한 DLT가 paired source와 같은 partition으로 변환된다.
- [ ] 알 수 없는 이름과 `.DLT.DLT`가 거부된다.
- [ ] key/value/header byte 내용은 같고 reference는 다르다.
- [ ] duplicate·null application header가 순서대로 보존된다.
- [ ] 모든 확인된 framework header가 제거된다.
- [ ] `kafka_application` 같은 유사 이름은 보존된다.
- [ ] 입력 record와 입력 배열이 변하지 않는다.

### 구현 후 설명할 것

- defensive copy가 운영 도구의 부수 효과를 막는 이유
- header allowlist가 아니라 exact framework denylist를 쓴 이유
- same partition tail replay가 원래 offset 위치로 되돌리는 것은 아니라는 점
- 자동 replay보다 operator-controlled replay가 안전한 이유
- DLT 접근 권한을 민감한 운영 증거로 취급해야 하는 이유

### 원본 확인 위치

- Thread 12
- 커밋: `feat(kafka): define event topic inventory`
- 커밋: `feat(kafka): route failures by source topic`
- 커밋: `feat(kafka): sanitize manual replay records`
- 커밋: `test(kafka): verify manual replay contract`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/GatewayTopicProperties.java`
- 파일: `src/main/java/com/sportsbook/gateway/kafka/DltReplayRecordFactory.java`
- 테스트: `src/test/java/com/sportsbook/gateway/kafka/DltReplayRecordFactoryTest.java`
- 관련 Thread: 10, 11, 16
