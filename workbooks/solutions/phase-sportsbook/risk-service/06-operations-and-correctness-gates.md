# 운영 준비성·격리된 정확성 게이트

이 문서는 운영 상태를 단순 프로세스 생존이 아니라 실제 의존성 확인으로 판단하는 readiness, 그리고 동시성·재전달·lifecycle 경계를 격리된 환경에서 검증하는 release gate를 다룬다. Thread 16의 metrics wiring은 설명 우선 B 항목으로 남기고, 구현 가치가 높은 resource lifecycle과 Thread 17의 correctness assertions만 상세화했다.

---

## P20. [Thread 16 / `test(readiness): verify dependency health contracts`] 제한 시간·resource cleanup·interrupt 보존을 갖춘 Kafka readiness

### 면접 질문

Kafka client bean이 생성됐다는 사실이 아니라 cluster metadata 응답을 readiness 조건으로 사용한 이유는 무엇입니까?

꼬리 질문:

- readiness와 liveness에 Redis·Kafka를 동일하게 넣으면 어떤 문제가 생깁니까?
- health 호출마다 만든 `AdminClient`를 모든 경로에서 닫지 않으면 어떤 resource가 누적됩니까?
- metadata future에 timeout을 두지 않으면 readiness endpoint 자체에 어떤 영향이 있습니까?
- `InterruptedException`을 잡고 DOWN을 반환하기 전에 interrupt flag를 복원한 이유는 무엇입니까?
- 의존성 상세 오류를 익명 health 응답에 그대로 노출해도 됩니까?

### 30초 모범 답변

readiness는 새 트래픽을 안전하게 받을 수 있는지를 판단하므로 Kafka 설정 객체의 존재가 아니라 broker가 제한 시간 안에 cluster metadata를 반환하는지 확인합니다. probe마다 만든 `AdminClient`는 try-with-resources로 모든 성공·실패 경로에서 닫고, 2초 같은 명시적 budget을 둬 health thread가 무기한 막히지 않게 합니다. interruption은 상위 shutdown·cancel 신호이므로 flag를 복원한 뒤 DOWN으로 보고합니다. readiness group에는 애플리케이션 상태와 Redis·Kafka를 넣되 liveness는 프로세스 재시작이 실제 해결책인지 기준으로 별도 판단합니다.

### 답변 핵심 키워드

readiness vs liveness · real dependency check · metadata timeout · try-with-resources · interrupt restoration · bounded probe · detail masking

### 백지 구현

#### 구현 목표

Kafka metadata를 제한 시간 안에 확인하고, 모든 경로에서 client를 닫으며, interruption을 보존하는 readiness probe를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
enum HealthStatus {
  UP, DOWN
}

record Health(
    HealthStatus status,
    String reason) {}

interface MetadataFuture {
  String get(long timeout, TimeUnit unit)
      throws InterruptedException, ExecutionException, TimeoutException;
}

interface Admin extends AutoCloseable {
  MetadataFuture clusterId();
  @Override void close();
}

public final class KafkaReadiness {
  public KafkaReadiness(
      Supplier<Admin> admins,
      Duration timeout) {
    // 직접 구현
  }

  public Health check() {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 매 probe마다 client를 제공하는 supplier와 timeout
- 출력: metadata 확인 성공 시 UP, 그 외 DOWN
- caller에게 client 예외를 그대로 던지지 않음

#### 반드시 만족해야 할 조건

- timeout은 양수이며 무한 대기를 허용하지 않는다.
- `check`마다 새 `Admin`을 얻는다.
- cluster ID future를 지정한 timeout 안에서 기다린다.
- 성공·실패·timeout·interruption 모든 경로에서 `Admin.close()`가 호출된다.
- `InterruptedException`을 잡으면 현재 thread의 interrupt flag를 복원한다.
- null 또는 빈 cluster ID를 성공으로 볼지 정책을 명시한다.
- 실패 reason은 bounded하며 credential·broker 내부 상세를 노출하지 않는다.
- supplier 자체가 실패해도 DOWN을 반환한다.

#### 경계 조건

- 즉시 성공
- timeout 직전 성공
- timeout
- `ExecutionException`
- `InterruptedException`
- supplier가 exception을 던짐
- `close()`가 exception을 던짐
- timeout 0·음수

#### 실패 조건

- 잘못된 구성 timeout
- client 생성 실패
- metadata failure/timeout
- interruption
- close failure

#### 필요한 제약

- background thread나 자체 retry loop를 만들지 않는다.
- 하나의 health 호출은 한 번의 metadata 시도만 한다.
- `Admin`을 singleton 필드로 재사용하지 않는다.
- 15~20분 안에 테스트 가능한 순수 클래스로 구현한다.

### 구현 후 자가 검증

- [ ] 정상 metadata 응답에서 UP이다.
- [ ] broker failure와 timeout에서 DOWN이다.
- [ ] 모든 경로에서 `close()`가 정확히 한 번 호출된다.
- [ ] interruption 뒤 `Thread.currentThread().isInterrupted()`가 true다.
- [ ] supplier 실패도 DOWN으로 변환된다.
- [ ] timeout 0·음수는 구성 시 거부된다.
- [ ] health reason에 비밀이나 전체 stack trace가 없다.
- [ ] probe가 무제한 blocking할 경로가 없다.
- [ ] liveness와 readiness에 포함할 의존성을 별도로 설명할 수 있다.

### 구현 후 설명할 것

1. bean 존재 검사보다 metadata round trip이 더 나은 readiness 신호인 이유
2. probe마다 client를 만드는 비용과 resource 격리의 trade-off
3. timeout budget이 너무 짧거나 길 때의 false negative·복구 지연
4. interruption flag 복원이 executor·shutdown 동작에 미치는 영향
5. Redis·Kafka 장애를 liveness가 아니라 readiness에 반영한 이유

### 원본 확인 위치

- Thread 16
- 커밋: `test(readiness): verify dependency health contracts`
- 파일: `src/main/java/com/sportsbook/risk/event/KafkaHealthIndicator.java`, `src/main/resources/application.yml`
- 클래스: `KafkaHealthIndicator`
- 테스트: `KafkaHealthIndicatorTest`, `DeployedRiskConfigurationTest`
- 관련 Thread: 01, 03, 15, 17

---

## P21. [Thread 17 / `test(load): verify concurrent reservation cardinality`] 동시성·replay·projection을 검증하는 격리 correctness gate

### 면접 질문

동시성 correctness test에서 단순히 "모든 요청이 200이었다"가 아니라 어떤 cardinality와 invariant를 검증해야 합니까?

꼬리 질문:

- 같은 bet 요청 100개에서는 왜 하나의 created와 99개의 replay를 기대합니까?
- 서로 다른 60 금액 요청 두 개가 100 한도를 경쟁할 때 어떤 결과 조합만 허용됩니까?
- thread 시작을 barrier로 맞추지 않으면 race test의 신뢰도가 왜 낮아집니까?
- sleep 기반 대기 대신 readiness polling과 명시적 process cleanup을 사용한 이유는 무엇입니까?
- accepted event 재전달 테스트에서 offset 증가와 Redis 합계 둘 다 확인해야 하는 이유는 무엇입니까?

### 30초 모범 답변

동시성 게이트는 HTTP 성공 여부가 아니라 상태 cardinality를 검증해야 합니다. 동일 bet 100개라면 lifecycle은 하나이고 최초 created 1개, 같은 token의 replay 99개여야 하며 합계도 한 번만 증가해야 합니다. 서로 다른 60 요청 두 개가 100 한도를 경쟁하면 정확히 하나만 승인되고 하나는 한도 거절이어야 합니다. 시작 barrier로 실제 경합을 만들고 결과 순서가 아닌 집합 invariant를 검증합니다. 격리된 Redis·Kafka를 readiness까지 기다린 뒤 실행하고, 실패해도 cleanup trap으로 환경을 회수합니다.

### 답변 핵심 키워드

cardinality assertion · start barrier · order-independent assertion · same token replay · exact capacity winner · hermetic dependencies · readiness polling · cleanup

### 백지 구현

#### 구현 목표

예약 서비스 인터페이스에 대해 두 가지 동시성 invariant를 검증하는 JUnit 테스트를 작성한다.

1. 같은 identity 요청 100개는 상태를 한 번만 생성한다.
2. 한도 100에서 서로 다른 60 요청 두 개는 정확히 하나만 승인한다.

#### 인터페이스 또는 함수 시그니처

```java
record Request(
    String betId,
    long amount) {}

record Decision(
    Status status,
    boolean replayed,
    String token,
    String rejectionReason) {}

enum Status {
  APPROVED, REJECTED, CONFLICT
}

interface ReservationClient {
  Decision reserve(Request request);
}

class ConcurrentReservationGateTest {
  @Test
  void createsOneIdentityAndReplaysTheRest() throws Exception {
    // 직접 구현
  }

  @Test
  void admitsExactlyOneLastCapacityCompetitor() throws Exception {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: thread-safe해야 하는 `ReservationClient`
- 출력: 테스트 assertion
- 테스트 자체의 thread failure: 원래 원인을 보존해 실패

#### 반드시 만족해야 할 조건

- 각 시나리오의 동시 요청 수와 같은 크기의 고정 executor를 사용한다.
- 각 worker가 준비됐음을 알리는 `ready` latch와 동시에 출발시키는 `start` latch를 분리한다.
- 모든 작업을 제출한 뒤 제한 시간 안에 모든 worker의 준비를 확인하고 `start`를 해제한다.
- 같은 identity 100개 결과에서:
  - 승인 결과는 모두 같은 token을 가진다.
  - `replayed=false`는 정확히 1개다.
  - `replayed=true`는 정확히 99개다.
  - conflict와 rejection은 없다.
- 서로 다른 두 60 요청 결과에서:
  - 승인은 정확히 1개다.
  - 일간 한도 거절은 정확히 1개다.
  - 승인 금액 총합은 100을 넘지 않는다.
- 결과 도착 순서나 특정 요청의 승자를 가정하지 않는다.
- 모든 future를 수집하고 worker 예외를 assertion failure 원인으로 보존한다.
- `finally`에서 executor를 종료한다.
- timeout을 두어 deadlock이 테스트를 무기한 막지 않게 한다.

#### 경계 조건

- worker 하나가 준비 단계에서 예외를 내거나 `ready`에 도달하지 못함
- 모든 worker가 준비된 뒤 같은 시점에 `start` 해제
- future 하나가 예외
- 테스트 timeout
- 승자가 매 실행마다 달라짐
- replay token이 하나만 다름

#### 실패 조건

- created가 0개 또는 2개 이상
- replay 수 불일치
- token 불일치
- 두 60 요청이 모두 승인되거나 모두 거절
- worker 예외가 삼켜짐
- executor leak 또는 테스트 hang

#### 필요한 제약

- `Thread.sleep`으로 race를 유도하지 않는다.
- 특정 실행 순서를 assertion하지 않는다.
- 테스트에서 서비스 내부 락이나 Redis 상태를 직접 조작하지 않는다.
- 20~30분 안에 helper를 포함해 구현한다.

### 구현 후 자가 검증

- [ ] `ready`와 `start`를 분리해 모든 worker가 준비된 뒤 실제 경합을 시작한다.
- [ ] 동일 identity에서 created 1, replay 99를 정확히 검증한다.
- [ ] 모든 replay token이 최초 token과 같다.
- [ ] capacity 경쟁에서 승인 1, 거절 1만 허용한다.
- [ ] 어느 bet가 승자인지는 assertion하지 않는다.
- [ ] future 예외의 원인이 테스트 출력에 남는다.
- [ ] executor가 성공·실패 모두에서 종료된다.
- [ ] 테스트 전체에 합리적인 timeout이 있다.
- [ ] 반복 실행해도 결과 cardinality가 안정적이다.
- [ ] 실제 E2E 확장 시 lifecycle replay, Kafka redelivery, source offset, Redis no-double-count를 함께 검증할 항목을 설명할 수 있다.

### 구현 후 설명할 것

1. 동시성 테스트에서 결과 순서보다 cardinality invariant가 중요한 이유
2. `ready`/`start` 이중 latch와 큰 반복 횟수가 race 재현 확률을 높이는 방식
3. 같은 identity replay 테스트와 서로 다른 identity capacity 경쟁 테스트가 확인하는 역량의 차이
4. 단위 Redis script 테스트, embedded Kafka 테스트, Docker gate를 계층화한 이유
5. 격리 환경의 readiness polling·timeout·cleanup이 flaky test를 줄이는 방식

### 원본 확인 위치

- Thread 17
- 커밋: `test(load): verify concurrent reservation cardinality`
- 파일: `load-test/concurrent-admission.sh`, `load-test/lifecycle-replay.sh`, `load-test/run-gate.sh`, `load-test/docker-compose.yml`, `.github/workflows/verify.yml`
- 파일: `src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationSupport.java`, `src/test/java/com/sportsbook/risk/event/BetPlacedKafkaRedisIntegrationTest.java`
- 클래스: `BetPlacedKafkaRedisIntegrationSupport`, `BetPlacedKafkaRedisIntegrationTest`
- 관련 Thread: 10, 11, 14, 15, 16
