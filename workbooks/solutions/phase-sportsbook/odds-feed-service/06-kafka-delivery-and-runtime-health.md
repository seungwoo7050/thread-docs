# Kafka 전달·publisher health 워크북

이 문서는 Kafka 호출을 "비동기 send를 시작한 것"이 아니라 "제한 시간 안에 broker가 acknowledge한 것"으로 정의한 경계를 다룬다. Thread 08의 projection 안전성과 Thread 10·14의 completion 순서는 이 경계 위에 세워져 있다.

<a id="p21"></a>
## [Thread 06 / `feat(publisher): publish thresholded odds changes`] bounded acknowledgement 대기와 interruption 처리

### 면접 질문

`OddsFeedPublisher`가 `KafkaTemplate.send()`가 반환한 future를 timeout과 함께 기다린 이유는 무엇입니까? `InterruptedException`, `TimeoutException`, `ExecutionException`을 각각 어떻게 처리해야 합니까?

꼬리 질문:

- future를 받았다는 사실만으로 `BrokerAvailability`를 available로 바꾸면 왜 위험합니까?
- `InterruptedException`을 잡고 interrupt flag를 복구하지 않으면 어떤 문제가 생깁니까?
- broker ack timeout과 producer의 `max.block.ms`는 어떤 다른 구간을 제한합니까?
- event ID를 Kafka key로 사용하면 무엇이 보장되고 무엇은 보장되지 않습니까?
- Kafka ack 뒤 application이 죽으면 duplicate가 생길 수 있습니까?

### 30초 모범 답변

send future 생성은 broker가 record를 저장했다는 뜻이 아닙니다. publisher의 성공 기준은 설정된 시간 안의 acknowledgement이므로 future를 bounded wait하고, 그 뒤에만 availability를 available로 바꿉니다. interruption은 상위 shutdown/cancel 신호이므로 현재 thread의 interrupt flag를 복구한 뒤 publish exception으로 전달합니다. timeout, execution failure, runtime failure는 unavailable로 표시합니다. event ID key는 같은 event의 record가 같은 partition으로 가도록 해 partition 내부 순서를 이용하지만, 전체 topic 순서나 exactly-once는 보장하지 않습니다.

### 답변 핵심 키워드

broker acknowledgement, bounded wait, future, interrupt restoration, timeout, health transition, Kafka key, partition ordering, at-least-once

### 백지 구현

**구현 목표**

비동기 broker port에 record를 보내고 제한 시간 안의 acknowledgement를 성공으로 판정하는 publisher를 구현한다.

**인터페이스**

```java
record OutboundRecord(
    String topic,
    String key,
    byte[] payload) {}

interface AsyncBroker {
  CompletableFuture<Void> send(OutboundRecord record);
}

interface Availability {
  void markAvailable();
  void markUnavailable();
}

final class PublishFailure extends RuntimeException {
  PublishFailure(String message, Throwable cause) {
    super(message, cause);
  }
}

final class AcknowledgedPublisher {
  private final AsyncBroker broker;
  private final Availability availability;
  private final Duration acknowledgementTimeout;

  AcknowledgedPublisher(
      AsyncBroker broker,
      Availability availability,
      Duration acknowledgementTimeout) {
    // 직접 구현
  }

  void publish(OutboundRecord record) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: topic, partition key, payload
- 출력: acknowledgement 성공 시 정상 반환, 그 외 `PublishFailure`
- health side effect: ack 성공 후 available, 실패 후 unavailable

**반드시 만족해야 할 조건**

- timeout은 양수
- broker `send` 자체의 synchronous exception 처리
- future가 null이면 실패
- 제한 시간 안에 future 정상 완료되어야 성공
- 성공 전에 `markAvailable` 호출 금지
- `InterruptedException` 시 `Thread.currentThread().interrupt()`
- interruption/timeout/execution/runtime failure 모두 unavailable
- 원인 예외를 cause로 보존
- payload/key null 정책 명시

**경계 조건**

- 이미 완료된 future
- timeout과 거의 동시에 완료
- future가 exceptionally completed
- `send`가 즉시 예외
- thread가 호출 전 이미 interrupted
- health callback 자체가 예외를 던지는 경우
- acknowledgement 성공 후 caller가 다음 저장 단계에서 실패

**실패 조건**

- unbounded `get()`
- interrupt flag 손실
- send 호출만 성공하면 available
- 예외를 로그만 남기고 정상 반환
- 모든 record에 null/random key 사용

**필요한 제약**

- 15~25분
- standard `CompletableFuture` 사용
- retry는 publisher 내부에서 하지 않고 caller/queue 책임으로 둠

### 구현 후 자가 검증

- [ ] completed future 정상 반환 후 available
- [ ] exceptionally completed future가 unavailable + `PublishFailure`
- [ ] timeout이 실제 무한 대기를 막음
- [ ] interruption 후 현재 thread interrupt flag가 true
- [ ] synchronous send failure 처리
- [ ] null future 처리
- [ ] 실패 원인이 exception cause로 유지
- [ ] available이 ack 전에 호출되지 않음을 recording fake로 검증
- [ ] 같은 event key가 record에 그대로 전달
- [ ] retry를 내부에서 몰래 수행하지 않음

### 구현 후 설명할 것

1. producer send, metadata wait, broker ack의 서로 다른 대기 구간
2. interrupt flag 복구가 executor/shutdown에 필요한 이유
3. publisher health를 마지막 시도 결과로 표현하는 한계
4. event ID partition key와 ordering 범위
5. ack 후 Redis completion 전 crash가 만드는 중복

### 원본 확인 위치

- Thread 06
- 커밋: `feat(publisher): publish thresholded odds changes`
- 커밋: `feat(publisher): define Kafka publish failures`
- 파일/컴포넌트: `OddsFeedPublisher`, `KafkaPublishException`, `BrokerAvailability`, `KafkaConfig`, `PublishProperties`
- 함수: `publishOddsChanged`, 내부 `publish`, `isHealthy`
- Kafka 설정: event ID key, bounded acknowledgement timeout, producer `max.block.ms`
- 관련 Thread: 08, 10, 14, 15

---

<a id="p22"></a>
## [Thread 06 / `test(publisher): verify odds thresholds and keys`] 상대 변화 임계값과 force snapshot

### 면접 질문

odds 변화 발행 여부를 절대 차이가 아니라 상대 변화율로 판단한 이유는 무엇입니까? feed 복구 중 `forceSnapshot`이 threshold를 무시해야 하는 이유도 설명해 주세요.

꼬리 질문:

- `2.00 → 2.01`과 `20.00 → 20.01`을 같은 절대 차이로 보면 어떤 문제가 있습니까?
- threshold와 정확히 같은 변화는 발행합니까?
- `BigDecimal.equals`와 `compareTo` 차이는 무엇입니까?
- division의 scale과 rounding을 어디서 고정합니까?
- previous 값이 0이면 상대 변화율을 어떻게 처리합니까?

### 30초 모범 답변

odds 크기에 따라 같은 절대 변화의 의미가 달라지므로 `abs(next - previous) / previous` 같은 상대 변화로 노이즈를 억제했습니다. threshold 이상일 때만 일반 update를 발행하고, 비교는 `BigDecimal` 수치 의미와 고정 rounding 정책으로 안정화해야 합니다. 다만 feed hold 복구는 downstream이 최신 snapshot을 반드시 받아야 하므로 변화가 작아도 `forceSnapshot`이면 발행합니다. 정책상 억제된 변화와 전달 실패로 누락된 변화를 같은 것으로 취급하지 않는 것이 핵심입니다.

### 답변 핵심 키워드

relative change, significance threshold, BigDecimal, scale, rounding, force snapshot, noise suppression, recovery semantics

### 백지 구현

**구현 목표**

두 양수 odds 값과 상대 threshold, force flag로 발행 여부를 결정하는 순수 함수를 구현한다.

**인터페이스**

```java
final class OddsPublicationPolicy {
  static boolean shouldPublish(
      BigDecimal previous,
      BigDecimal next,
      BigDecimal relativeThreshold,
      boolean forceSnapshot) {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 이전/새 odds, 0 이상 threshold, force flag
- 출력: 발행 여부

**반드시 만족해야 할 조건**

- force가 true면 유효 입력에 대해 항상 true
- force가 false면 상대 변화 절댓값이 threshold 이상일 때 true
- odds는 0보다 커야 함
- threshold는 0 이상
- scale만 다른 동일 수치는 변화 없음
- rounding/precision 정책이 명시적
- input object 변경 없음
- locale/string formatting에 의존하지 않음

**경계 조건**

- previous와 next 동일
- threshold 0
- 변화율이 threshold와 정확히 같음
- 매우 작은 odds 차이
- scale이 매우 큰 BigDecimal
- previous 0 또는 음수
- next 음수
- force true + invalid input

**실패 조건**

- `double`로 변환해 정밀도 손실
- `BigDecimal.equals`만으로 수치 동일성 판단
- division에서 `ArithmeticException`
- force가 validation까지 우회
- threshold 단위를 퍼센트 정수와 소수로 혼동

**필요한 제약**

- 10~15분
- 외부 publish는 구현하지 않음
- division 또는 cross-multiplication 선택 이유 설명

### 구현 후 자가 검증

- [ ] 동일 수치, 다른 scale은 false
- [ ] threshold 미만 false
- [ ] threshold 정확히 같은 변화 true
- [ ] threshold 초과 true
- [ ] 감소도 절댓값으로 판정
- [ ] force true가 작은 변화도 true
- [ ] zero/negative odds 거부
- [ ] negative threshold 거부
- [ ] repeating decimal에서도 예외 없음
- [ ] large precision 입력에서 double 변환 없음

### 구현 후 설명할 것

1. 상대 임계값과 절대 임계값의 trade-off
2. division과 cross-multiplication 비교
3. rounding mode가 경계 판정에 미치는 영향
4. force snapshot을 domain recovery 신호로 둔 이유
5. threshold 억제된 local projection과 downstream 상태 차이 관리

### 원본 확인 위치

- Thread 06
- 커밋: `test(publisher): verify odds thresholds and keys`
- 파일/컴포넌트: `OddsFeedPublisher`, `PublishProperties`
- 함수: `isSignificantChange`, `publishOddsChanged`
- 입력 정책: relative threshold, force snapshot, event ID Kafka key
- 관련 Thread: 08
