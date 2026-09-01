# 기본 정산·임대 복구·아웃박스 면접 워크북

이 문서는 accepted result fanout에서 시작해 내구성 있는 정산 시도, 임대 소유권, 실패 복구, transactional outbox publication까지의 경계를 다룬다.

<a id="p11"></a>
<!-- POINT:P11 -->
## P11 — [Thread 9 / `feat(result): prepare accepted result claims`] accepted result projection과 동시 완료 race

### 면접 질문

MULTIPLE 또는 SYSTEM 베팅의 서로 다른 event 결과가 거의 동시에 도착해 마지막 unresolved selection을 각각 완료하려고 할 때, 어떻게 정산 시도가 하나만 만들어지도록 했습니까?

꼬리 질문:

- selection에 outcome만 저장하지 않고 source candidate ID도 함께 보존한 이유는 무엇입니까?
- `allSelectionsResolved()`를 메모리에서 확인하는 것만으로 race가 해결됩니까?
- 이미 정산 시도가 있는 `PENDING` 베팅을 fanout 조회에서 제외하는 이유는 무엇입니까?

### 30초 모범 답변

accepted result를 event별 immutable projection으로 읽고, 해당 event의 selections에 outcome과 candidate provenance를 적용합니다. 베팅 행을 잠근 상태에서 현재 상태와 기존 attempt를 다시 확인하고, 모든 selection이 해결된 경우에만 immutable attempt draft를 만듭니다. 정산 시도 테이블은 bet당 한 행이고 insert conflict를 허용하지 않으므로, 두 결과가 동시에 마지막 selection을 채워도 하나만 claim합니다. 메모리 검사는 계산 조건이고 최종 동시성 경계는 행 잠금과 유일 제약입니다.

### 답변 핵심 키워드

immutable projection, candidate provenance, row lock, re-read, all resolved, one attempt per bet, unique conflict

### 백지 구현

#### 구현 목표

베팅 snapshot에 한 event의 accepted result를 적용하고, 모든 selection이 해결된 경우에만 정산 draft를 생성하는 순수 도메인 함수를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class ResultSettlementPlanner {
  public PlanningResult applyAndPlan(BetSnapshot bet, AcceptedResult result) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: 현재 bet snapshot, event별 accepted result와 source candidate ID
- 출력: 변경된 snapshot, 변경 여부, 선택적 settlement attempt draft

#### 반드시 만족해야 할 조건

- `PENDING` 베팅만 result projection을 받을 수 있다.
- result의 event ID와 일치하는 selections만 변경한다.
- 각 selection에 outcome과 source candidate ID를 함께 저장한다.
- 같은 candidate의 exact replay는 의미 없는 중복 변경을 만들지 않는다.
- 하나라도 unresolved selection이 남으면 draft를 만들지 않는다.
- 모두 resolved여도 기존 attempt 존재 여부는 저장소 경계에서 다시 확인해야 한다.
- 입력 snapshot을 직접 변경하지 않는다.

#### 경계 조건

- 하나의 selection만 해당 event를 참조
- 같은 event를 참조하는 여러 selections
- 일부 selection이 이미 다른 event 결과로 해결됨
- exact replay
- result에 없는 selection ID
- 결과 적용 후 처음으로 모두 resolved가 되는 순간

#### 실패 조건

- terminal bet에 result 적용
- event ID가 다른 selection을 변경
- outcome과 candidate provenance 중 하나만 저장
- unresolved 상태에서 draft 생성
- 메모리 확인만 믿고 동시 attempt 생성을 허용

#### 필요한 제약

- 동시성 자체는 repository의 `for update`와 unique claim으로 해결한다고 설명한다.
- 순수 함수는 동일 입력에 동일 결과를 반환해야 한다.
- selection 목록의 불변성을 유지한다.

### 구현 후 자가 검증

- 해당 event의 selections만 변경되는가?
- 같은 candidate replay가 updatedAt을 불필요하게 바꾸지 않는가?
- 모든 selection이 해결되기 전에는 draft가 없는가?
- 마지막 unresolved selection이 해결되면 정확히 하나의 draft 후보가 생기는가?
- terminal 상태에서는 아무 projection도 허용하지 않는가?
- 두 thread가 같은 draft 후보를 계산해도 저장소에서 하나만 claim할 수 있다고 설명했는가?

### 구현 후 설명할 것

1. accepted result를 별도 projection으로 읽는 이유
2. selection에 candidate provenance를 남기는 이유
3. 도메인 완결성 검사와 데이터베이스 동시성 제어의 차이
4. fanout query에서 이미 소유된 attempt를 제외하는 이유

### 원본 확인 위치

- Thread: 9 — accepted result projection과 기본 정산 fanout
- 대표 커밋: `feat(result): prepare accepted result claims`
- 관련 커밋: `feat(result): model accepted result projections`, `feat(result): query unowned pending bets`, `feat(result): fan out accepted results`, `test(result): execute concurrent result claims`
- 파일: `src/main/java/com/sportsbook/settlement/result/AcceptedResult.java`
- 파일: `src/main/java/com/sportsbook/settlement/result/AcceptedResultRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/result/ResultSettlementPreparer.java`
- 파일: `src/main/java/com/sportsbook/settlement/result/ResultFanout.java`
- 파일: `src/main/java/com/sportsbook/settlement/domain/Bet.java`
- 파일: `src/main/java/com/sportsbook/settlement/domain/BetSelection.java`
- 파일: `src/main/java/com/sportsbook/settlement/persistence/BetRepository.java`
- 관련 메서드: `Bet.applyAcceptedResult`, `BetSelection.applyCandidate`, `BetRepository.findResultActionableIdsByEvent`, `ResultSettlementPreparer.prepare`
- 관련 Thread: 4, 10, 13

<a id="p12"></a>
<!-- POINT:P12 -->
## P12 — [Thread 10 / `feat(settlement): claim pending bets atomically`] 외부 Wallet 호출 전 immutable attempt 영속화

### 면접 질문

왜 Wallet을 먼저 호출하고 성공 결과만 저장하지 않고, action·result·모든 금액 movement·lease를 가진 `settlement_attempt`를 외부 호출 전에 저장했습니까?

꼬리 질문:

- 프로세스가 Wallet 성공 직후 죽으면 어떤 모호성이 생깁니까?
- retry 때 최신 bet 상태로 money plan을 다시 계산하면 안 되는 이유는 무엇입니까?
- attempt row를 bet당 하나로 제한한 trade-off는 무엇입니까?

### 30초 모범 답변

데이터베이스 transaction과 Wallet 호출은 원자적으로 묶을 수 없으므로, 먼저 실행할 action과 금액 계획을 immutable evidence로 저장했습니다. 그러면 Wallet 성공 후 crash하거나 응답을 잃어도 recovery가 같은 bet·같은 key·같은 금액을 재사용할 수 있습니다. attempt는 `PENDING` bet에서만 생성되고 bet당 한 행이며, action 조합·금액 보존식·lease pair를 DB constraint로도 고정합니다. 비용은 스키마와 복구 로직이 늘지만 금전 재계산 drift와 중복 실행을 막습니다.

### 답변 핵심 키워드

persist-before-side-effect, immutable execution intent, crash ambiguity, stable operation identity, one row per bet, database constraints

### 백지 구현

#### 구현 목표

`PENDING` bet에 대해 한 번만 immutable settlement attempt를 claim하는 저장소 추상화를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface AttemptStore {
  // 직접 구현
  Optional<ClaimedAttempt> claimPending(
      BetState bet,
      AttemptDraft draft,
      Duration leaseDuration);
}
```

#### 입력과 출력

- 입력: 현재 bet 상태, 검증된 action과 money plan, lease duration
- 출력: 최초 claim이면 token과 lease 만료 시각이 포함된 attempt, 이미 소유되었거나 부적격이면 empty

#### 반드시 만족해야 할 조건

- bet가 `PENDING`일 때만 생성한다.
- bet당 attempt는 하나뿐이다.
- SETTLE은 result가 있고 void reason이 없어야 한다.
- VOID는 result가 없고 허용된 void reason이 있어야 한다.
- money plan의 비음수·통화·두 보존식을 검증한다.
- lease token과 lease until은 항상 함께 존재하거나 함께 없어야 한다.
- 최초 attempt count는 1이다.
- conflict가 발생해도 기존 계획을 덮어쓰지 않는다.
- 반환된 attempt는 입력 draft의 의미를 정확히 보존한다.

#### 경계 조건

- 같은 draft의 동시 claim
- 다른 draft가 같은 bet을 동시 claim
- lease duration이 최소 유효값
- payout이 0인 lost settlement
- 전체 슬립 void

#### 실패 조건

- terminal bet claim
- action 필드 조합 오류
- conservation 위반
- lease duration 0 또는 음수
- 기존 attempt overwrite
- race 후 존재하지 않는 row를 성공처럼 반환

#### 필요한 제약

- in-memory 구현이라면 atomic map operation 또는 lock을 사용한다.
- 실제 DB 대응에서는 `INSERT ... SELECT ... WHERE status='PENDING' ON CONFLICT DO NOTHING`과 같은 원자 경계를 설명한다.
- external Wallet 호출은 이 함수 안에서 수행하지 않는다.

### 구현 후 자가 검증

- 같은 bet을 여러 thread가 claim해 하나만 성공하는가?
- 서로 다른 draft race에서도 기존 계획이 바뀌지 않는가?
- terminal bet은 claim되지 않는가?
- attempt count와 lease pair invariant가 유지되는가?
- 모든 money plan 검증이 저장 직전에 다시 수행되는가?
- 반환 객체와 저장 객체가 외부 mutation에 노출되지 않는가?

### 구현 후 설명할 것

1. 외부 side effect 전에 intent를 저장하는 이유
2. attempt를 immutable하게 유지해야 하는 이유
3. one-attempt-per-bet 모델의 장점과 correction을 별도 revision으로 분리한 이유
4. 애플리케이션 검증과 DB CHECK/PK의 중복이 필요한 이유
5. transaction 범위에서 Wallet을 제외한 이유

### 원본 확인 위치

- Thread: 10 — 내구성 있는 기본 정산 실행, 임대 복구, 원자적 확정
- 대표 커밋: `feat(settlement): claim pending bets atomically`
- 관련 커밋: `feat(settlement): define initial attempt drafts`, `feat(settlement): prepare resolved attempts`, `feat(settlement): preserve attempt migration`, `feat(settlement): fence execution with leases`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptDraft.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttempt.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementLease.java`
- 파일: `src/main/resources/db/migration/V5__settlement_attempt.sql`
- 관련 메서드: `SettlementAttemptRepository.claimPending`
- 관련 Thread: 5, 7, 9, 14, 19

<a id="p13"></a>
<!-- POINT:P13 -->
## P13 — [Thread 10 / `feat(recovery): claim ordered attempt batches`] database time·lease token·`SKIP LOCKED` 기반 복구

### 면접 질문

긴 Wallet 호출 동안 데이터베이스 row lock을 유지하지 않고 lease를 사용한 이유는 무엇입니까? lease 만료 판단을 애플리케이션 clock이 아니라 PostgreSQL `current_timestamp`로 통일한 이유도 설명해 보세요.

꼬리 질문:

- `FOR UPDATE SKIP LOCKED`가 없으면 여러 recovery worker가 어떻게 방해합니까?
- lease token 없이 `lease_until`만 비교하면 어떤 stale-owner 문제가 생깁니까?
- lease가 만료된 직후 이전 worker의 Wallet 응답이 돌아오면 어떻게 막습니까?

### 30초 모범 답변

claim transaction은 due rows를 짧게 잠그고 새 token과 만료 시각을 기록한 뒤 끝내고, 느린 Wallet 호출은 transaction 밖에서 수행합니다. `SKIP LOCKED`로 여러 worker가 서로 기다리지 않고 다른 row를 가져가며, 정렬과 batch limit으로 회수 순서를 안정화합니다. 시간 판정은 DB clock 하나를 사용해 인스턴스 clock skew를 제거합니다. 최종 update는 bet/revision ID뿐 아니라 exact lease token과 DB 기준 미만료 조건을 포함해 이전 소유자의 늦은 응답을 거부합니다.

### 답변 핵심 키워드

short claim transaction, lease ownership, database clock, `FOR UPDATE SKIP LOCKED`, bounded batch, stale-owner fence, exact token

### 백지 구현

#### 구현 목표

현재 DB 시각을 입력으로 받아 unowned 또는 expired attempt 중 due rows를 정렬해 제한된 개수만 새 lease로 claim하는 순수 함수를 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class LeaseClaimer {
  public List<ClaimedRow> claimDue(
      List<AttemptRow> rows,
      Instant databaseNow,
      Duration leaseDuration,
      int limit,
      Supplier<UUID> tokenSource) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: attempt rows, 권위 있는 database time, lease duration, batch limit, token 생성기
- 출력: 새 token·leaseUntil·증가된 attempt count가 있는 claim 목록

#### 반드시 만족해야 할 조건

- lease가 없거나 `leaseUntil <= databaseNow`인 row만 claim한다.
- active lease는 건드리지 않는다.
- 복구 우선순위를 `updatedAt`, 그 다음 안정된 식별자 순으로 정한다.
- limit을 넘기지 않는다.
- 각 claimed row는 서로 다른 새 token을 가진다.
- leaseUntil은 databaseNow를 기준으로 계산한다.
- attempt count는 정확히 한 번 증가한다.
- 원본 rows를 직접 변경하지 않는다.

#### 경계 조건

- leaseUntil이 databaseNow와 정확히 같은 row
- 모든 row가 active
- limit보다 due row가 적음
- limit=1
- 동일 updatedAt을 가진 rows
- leaseDuration 더하기 overflow

#### 실패 조건

- active row claim
- 같은 row를 한 batch에서 두 번 반환
- caller clock을 섞어 사용
- limit 범위 위반
- token 재사용
- attempt count overflow

#### 필요한 제약

- 실제 SQL에서는 선택과 update가 한 transaction 안에서 일어나야 한다고 설명한다.
- worker 간 non-blocking 분배는 `SKIP LOCKED`가 담당하고, 함수는 그 선택 의미만 모델링한다.
- batch limit은 1 이상이며 운영상 상한을 둔다.

### 구현 후 자가 검증

- 만료 시각이 현재와 같은 row를 due로 처리하는가?
- active lease가 반환되지 않는가?
- 동일 입력과 token source에서 정렬 순서가 결정적인가?
- limit과 attempt count가 정확한가?
- 각 claim token이 유일한가?
- 이전 token으로 finalize하려는 요청을 별도 predicate에서 거부할 수 있는가?
- process clock skew가 결과에 영향을 주지 않는가?

### 구현 후 설명할 것

1. row lock 대신 lease를 사용한 이유
2. `SKIP LOCKED`의 처리량 이점과 starvation 가능성
3. database time을 single authority로 삼은 이유
4. token과 expiry를 함께 검사해야 하는 이유
5. batch size·lease duration을 조정할 때의 trade-off

### 원본 확인 위치

- Thread: 10 — 내구성 있는 기본 정산 실행, 임대 복구, 원자적 확정
- 대표 커밋: `feat(recovery): claim ordered attempt batches`
- 관련 커밋: `feat(recovery): hydrate durable attempt rows`, `feat(recovery): schedule incomplete settlement attempts`
- Thread 19 관련 커밋: `feat(settlement): claim attempts with database time`, `refactor(settlement): remove caller finalization clock`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementRecoveryRow.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRecovery.java`
- 관련 메서드: `SettlementAttemptRepository.claimRecoveryBatch`, `SettlementAttemptRepository.consumeLease`
- 관련 Thread: 15, 19, 20

<a id="p14"></a>
<!-- POINT:P14 -->
## P14 — [Thread 11 / `feat(outbox): publish locked pending events`] 원자적 정산 확정과 broker acknowledgment 경계

### 면접 질문

베팅 상태 변경과 Kafka publish를 한 distributed transaction으로 묶지 않고 transactional outbox를 사용한 이유는 무엇입니까? 이 구현이 exactly-once publication이 아니라 at-least-once인 crash window를 설명해 보세요.

꼬리 질문:

- Kafka send future가 완료되기 전에 `published_at`을 기록하면 어떤 문제가 생깁니까?
- broker ack 직후 DB transaction이 rollback되면 어떻게 됩니까?
- 여러 publisher 인스턴스가 같은 outbox row를 동시에 보내지 않게 한 방법은 무엇입니까?

### 30초 모범 답변

정산 finalization transaction 안에서 bet terminal 전이, exact lease 소비, outbox insert를 함께 커밋해 도메인 상태와 발행 의도를 원자화했습니다. publisher는 미발행 rows를 `FOR UPDATE SKIP LOCKED`로 claim하고 Kafka send 완료를 bounded wait로 확인한 뒤에만 `published_at`을 기록합니다. send 성공 후 DB commit 전에 죽으면 같은 row를 다시 보내므로 중복은 가능하지만 누락은 피합니다. 따라서 publication은 at-least-once이고 소비자는 event identity로 멱등해야 합니다.

### 답변 핵심 키워드

transactional outbox, atomic state and intent, broker ack, mark-after-send, `SKIP LOCKED`, at-least-once, duplicate window

### 백지 구현

#### 구현 목표

도메인 finalization과 outbox 생성의 원자성, 그리고 broker ack 이후 publication marker를 기록하는 두 단계 경계를 저장소 추상화로 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
public interface SettlementFinalizationService {
  // 직접 구현
  boolean finalizeBase(FinalizeCommand command);
}

public interface OutboxBatchPublisher {
  // 직접 구현
  PublishReport publishBatch(int limit);
}
```

#### 입력과 출력

- finalization 입력: bet ID, action/result, payout, exact lease token
- finalization 출력: 소유권과 상태가 유효해 적용했는지 여부
- publisher 입력: batch limit
- publisher 출력: attempted/published 수와 실패 정보

#### 반드시 만족해야 할 조건

- finalization은 bet row를 잠그고 `PENDING`인지 다시 확인한다.
- exact action과 lease token을 검증하고 DB time 기준으로 lease가 유효해야 한다.
- bet terminal 변경, attempt lease 소비, outbox insert가 한 transaction이다.
- 한 finalization에서 outbox event는 하나만 생성한다.
- event payload는 생성 후 변경되지 않는다.
- publisher는 미발행 row만 제한된 수만큼 claim한다.
- broker acknowledgment를 확인한 뒤에만 published marker를 기록한다.
- send 실패나 timeout이면 row는 미발행 상태로 남는다.
- 동시에 실행되는 publisher가 같은 live transaction의 row를 가져가지 않는다.

#### 경계 조건

- lease가 막 만료된 finalization
- 이미 terminal인 bet
- outbox batch가 비어 있음
- broker ack 직후 publisher process 종료
- 여러 publisher와 row 수가 batch limit보다 많음
- payload가 빈 byte 배열인 경우를 허용할지 여부

#### 실패 조건

- domain update만 커밋되고 outbox가 누락됨
- outbox만 생성되고 terminal update가 rollback됨
- send 시작만으로 published 처리
- 실패한 row를 성공 처리
- stale token으로 finalization
- mutable payload가 publish 전에 변경됨

#### 필요한 제약

- Kafka와 DB의 exactly-once를 주장하지 않는다.
- transaction callback 또는 명시적 unit of work로 원자 경계를 테스트할 수 있어야 한다.
- broker send wait에는 상한이 있어야 한다.

### 구현 후 자가 검증

- finalization transaction 실패 시 bet와 outbox가 모두 원복되는가?
- stale/expired lease가 아무 상태도 바꾸지 않는가?
- send failure 뒤 row가 다시 조회되는가?
- send 완료 뒤에만 marker가 기록되는가?
- 두 publisher가 잠긴 row를 건너뛰어 서로 다른 rows를 처리하는가?
- ack 후 DB rollback에서 duplicate가 생길 수 있음을 테스트나 설명으로 드러냈는가?
- payload byte 배열을 defensive copy하는가?

### 구현 후 설명할 것

1. distributed transaction 대신 outbox를 선택한 이유
2. at-least-once publication의 정확한 duplicate window
3. `SKIP LOCKED`가 publisher 확장성에 주는 효과
4. producer idempotence와 비즈니스 event 멱등성의 차이
5. transaction 안에서 네트워크 send를 기다리는 방식의 장단점

### 원본 확인 위치

- 대표 Thread: 11 — transactional outbox와 broker acknowledgment 경계
- 대표 커밋: `feat(outbox): publish locked pending events`
- 관련 커밋: `build(flyway): add V3 transactional outbox`, `test(outbox): verify broker ack publication boundary`, `test(outbox): verify PostgreSQL skip locked claims`
- Thread 10 관련 커밋: `feat(settlement): finalize resolved bets atomically`, `feat(settlement): finalize whole slip voids atomically`
- Thread 19 관련 커밋: `fix(settlement): fence finalization with database time`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/OutboxEvent.java`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/OutboxEventRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/OutboxPublisher.java`
- 파일: `src/main/java/com/sportsbook/settlement/outbox/SettlementEventFactory.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementFinalizer.java`
- 파일: `src/main/resources/db/migration/V3__outbox.sql`
- 관련 메서드: `SettlementAttemptRepository.consumeLease`, `OutboxEventRepository.lockNextUnpublished`, `OutboxPublisher.publishBatch`
- 관련 Thread: 10, 19, 20

<a id="p15"></a>
<!-- POINT:P15 -->
## P15 — [Thread 10 / `feat(recovery): release failed attempts safely`] batch 실패 격리와 owner-fenced lease 반환

### 면접 질문

recovery batch에서 한 Wallet 호출이 실패했을 때 전체 batch를 중단하지 않고 해당 attempt의 lease만 안전하게 반환한 이유는 무엇입니까? 실패 원문 대신 제한된 error summary만 저장한 이유도 설명해 보세요.

꼬리 질문:

- lease 반환 update가 0건이면 어떤 의미입니까?
- 실패한 작업을 즉시 무한 재시도하면 어떤 문제가 생깁니까?
- 외부 dependency 응답 body를 `last_error`에 그대로 저장하면 안 되는 이유는 무엇입니까?

### 30초 모범 답변

각 bet 정산은 독립된 work item이므로 하나의 Wallet 장애가 batch의 나머지 작업을 굶기지 않게 item 단위로 실행하고 결과를 수집했습니다. 실패 시 `bet_id + exact lease_token`으로만 lease를 해제하고, 이미 소유권을 잃었다면 stale worker가 새 owner 상태를 덮어쓰지 못하게 0건을 반환합니다. 저장하는 오류는 분류 가능한 예외명이나 Wallet error code 정도로 제한해 secret·대용량 body·개인정보가 durable state에 남지 않게 합니다.

### 답변 핵심 키워드

per-item isolation, continue batch, owner-fenced release, lost ownership, sanitized error, bounded retry, no raw dependency body

### 백지 구현

#### 구현 목표

여러 settlement executions를 순서대로 실행하되 각 실패를 격리하고, 실패한 item의 exact lease만 반환하는 batch runner를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class SettlementBatchRunner {
  public BatchResult run(List<SettlementExecution> executions) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: immutable settlement executions
- 출력: item별 `APPLIED`, `RETRY_RELEASED`, `LOST_OWNERSHIP`, `FAILED_TO_RELEASE` 결과

#### 반드시 만족해야 할 조건

- 각 execution은 독립적으로 처리한다.
- 한 item의 runtime failure가 다음 item 실행을 자동으로 막지 않는다.
- 실패 시 해당 bet ID와 exact lease token으로 release를 시도한다.
- release update가 0건이면 lost ownership으로 구분한다.
- 저장할 오류 문자열은 bounded하고 안전하게 canonicalize한다.
- 성공 item에는 release를 호출하지 않는다.
- 결과 목록은 입력 순서와 대응되고 외부에서 변경할 수 없어야 한다.

#### 경계 조건

- 빈 batch
- 첫 item 실패, 나머지 성공
- 모든 item 실패
- Wallet 전용 분류 오류와 일반 runtime 오류
- release 시점에 이미 lease가 바뀐 경우

#### 실패 조건

- 한 실패로 batch 전체 결과가 사라짐
- 다른 item의 lease를 해제
- token 없이 bet ID만으로 release
- raw HTTP body 또는 secret 저장
- release 실패를 정상 retry 가능 상태로 위장

#### 필요한 제약

- 치명적인 JVM 오류까지 잡는 구현을 만들지 않는다.
- interrupt 관련 예외는 상위 shutdown 의미를 보존해야 한다.
- retry cadence와 최대 시도 횟수는 runner 밖의 durable scheduling 정책으로 둔다.

### 구현 후 자가 검증

- 한 item 실패 뒤 다음 item이 실행되는가?
- 성공 item의 상태가 실패 item 때문에 rollback되지 않는가?
- release에 exact token이 전달되는가?
- 0-row release를 lost ownership으로 분류하는가?
- error summary 길이와 허용 문자를 제한하는가?
- 결과 목록이 입력 순서를 보존하는가?
- interrupt 상태를 실수로 지우지 않는가?

### 구현 후 설명할 것

1. batch 전체 transaction보다 item 격리를 선택한 이유
2. lease release에도 owner fence가 필요한 이유
3. 0-row update를 오류가 아니라 상태 신호로 해석하는 방법
4. durable error detail을 최소화한 보안·운영 trade-off
5. retry scheduling을 실행 runner와 분리한 이유

### 원본 확인 위치

- Thread: 10 — 내구성 있는 기본 정산 실행, 임대 복구, 원자적 확정
- 대표 커밋: `feat(recovery): release failed attempts safely`
- 관련 커밋: `feat(settlement): fan out independent executions`, `feat(recovery): schedule incomplete settlement attempts`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementExecutionRunner.java`
- 파일: `src/main/java/com/sportsbook/settlement/execution/SettlementAttemptRecovery.java`
- 관련 메서드: `SettlementAttemptRepository.releaseForRecovery`
- 관련 Thread: 8, 12, 15, 20, 21
