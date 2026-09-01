# 관리자 제어·내부 인증·worker runtime 면접 워크북

이 문서는 운영 명령의 semantic idempotency와 append-only audit, 내부 서비스 인증 경계, scheduler 격리와 bounded shutdown을 다룬다.

<a id="p23"></a>
<!-- POINT:P23 -->
## P23 — [Thread 16 / `feat(admin): validate idempotent replays`] 운영 명령의 request binding과 append-only audit

### 면접 질문

관리자 mutation에 `Idempotency-Key`만 저장하지 않고 action kind, target ID, canonical payload fingerprint를 함께 묶은 이유는 무엇입니까? 같은 key로 다른 요청이 오면 왜 이전 결과를 그대로 반환하면 안 됩니까?

꼬리 질문:

- 같은 key의 동시 요청을 advisory transaction lock으로 직렬화한 이유는 무엇입니까?
- command가 성공한 뒤 audit insert가 실패하면 어떻게 되어야 합니까?
- audit table의 UPDATE와 DELETE를 DB trigger로 막은 이유는 무엇입니까?

### 30초 모범 답변

idempotency key는 한 semantic command를 재시도할 식별자이므로, kind·target·정규화된 payload의 fingerprint에 영구히 bind해야 합니다. 같은 key와 같은 fingerprint는 이전 outcome을 replay하고 command를 다시 실행하지 않지만, 다른 의미의 요청이면 conflict입니다. 같은 key race는 transaction-scoped advisory lock으로 직렬화하고, 대상 상태 변경과 append-only action insert를 한 transaction에 둡니다. DB trigger로 UPDATE/DELETE를 차단해 애플리케이션 버그나 수동 변경도 감사 이력을 덮어쓰지 못하게 했습니다.

### 답변 핵심 키워드

semantic idempotency, key-to-request binding, canonical fingerprint, exact replay, conflict, advisory transaction lock, append-only audit

### 백지 구현

#### 구현 목표

동일 idempotency key의 동시 요청을 직렬화하고, exact replay는 이전 receipt를 반환하며, conflicting reuse는 거부하는 command executor를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class IdempotentAdminExecutor {
  public AdminReceipt execute(
      UUID idempotencyKey,
      AdminRequest request,
      AdminCommand command) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: UUID idempotency key, action kind·target·payload를 가진 요청, 실제 상태 변경 command
- 출력: 최초 실행 또는 replay 여부가 포함된 immutable receipt

#### 반드시 만족해야 할 조건

- kind, target ID, canonical payload로 fingerprint를 만든다.
- key 단위 lock을 획득한 뒤 기존 action을 조회한다.
- 기존 action이 exact fingerprint면 command를 실행하지 않고 이전 receipt를 반환한다.
- 같은 key가 다른 kind/target/fingerprint에 묶였으면 conflict다.
- 기존 action이 없을 때만 command를 한 번 실행한다.
- 상태 변경과 audit append는 같은 transaction 경계다.
- audit record는 생성 후 수정하거나 삭제하지 않는다.
- rejection reason을 payload에 사용한다면 앞뒤 공백 정책, 1~256자, printable 문자 제약을 적용한다.
- 실행 token이 필요한 action과 필요하지 않은 action의 shape를 구분한다.

#### 경계 조건

- exact replay
- 같은 key·다른 target
- 같은 key·같은 target·다른 rejection reason
- 같은 key의 동시 최초 요청
- command 결과가 no-op인 유효 요청
- fingerprint 계산에 빈 payload가 들어가는 action

#### 실패 조건

- 기존 action 조회 전에 command 실행
- conflicting reuse를 replay로 처리
- replay에서 command 재실행
- command 성공과 audit append가 분리 commit
- audit UPDATE/DELETE 허용
- canonicalization이 공백이나 필드 순서에 따라 우연히 달라짐

#### 필요한 제약

- in-memory 구현이면 key별 lock 또는 atomic compute를 사용한다.
- 실제 DB 대응에서는 transaction-scoped lock과 append-only schema를 설명한다.
- audit record에 credential이나 dependency response body를 저장하지 않는다.

### 구현 후 자가 검증

- exact replay에서 command 호출 횟수가 증가하지 않는가?
- key reuse conflict를 kind·target·payload 각각에 대해 검증했는가?
- 동시 최초 요청에서 command와 audit가 하나만 생성되는가?
- command 실패 시 audit 성공 record가 남지 않는가?
- audit append 실패 시 대상 상태 변경도 rollback된다고 설명했는가?
- reason validation과 canonical fingerprint가 일관적인가?
- 저장된 action을 수정하는 API가 존재하지 않는가?

### 구현 후 설명할 것

1. idempotency key와 request fingerprint를 분리한 이유
2. advisory transaction lock과 unique key만 사용하는 방식의 차이
3. command와 audit를 같은 transaction에 둔 이유
4. append-only를 애플리케이션 규칙뿐 아니라 DB trigger로 보강한 이유
5. exact replay receipt를 저장하는 비용과 운영 이점

### 원본 확인 위치

- Thread: 16 — 멱등 운영 제어와 감사 로그
- 대표 커밋: `feat(admin): validate idempotent replays`
- 관련 커밋: `build(flyway): add V10 admin actions`, `feat(admin): canonicalize operator actions`, `feat(admin): approve result candidates`, `feat(admin): claim manual revision retries`
- 파일: `src/main/resources/db/migration/V10__admin_action.sql`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminAction.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminRequestFingerprint.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminActionRepository.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminActionReplay.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminCandidateApproval.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminRevisionRetryRepository.java`
- 관련 메서드: `AdminActionReplay.requireExact`, `AdminActionRepository.append`, `AdminCandidateApproval.decide`, `AdminRevisionRetryRepository.queue`
- 관련 Thread: 13, 14, 15, 18, 19

<a id="p24"></a>
<!-- POINT:P24 -->
## P24 — [Thread 18 / `feat(admin): authenticate internal requests`] 경로 한정 인증과 자격증명 분리

### 면접 질문

내부 admin API에서 header가 없을 때 401, 중복되거나 잘못됐을 때 403을 반환하고, header가 정확히 하나씩 있는지 검사한 이유는 무엇입니까? admin credential과 Wallet credential을 startup에 서로 다르게 강제한 이유도 설명해 보세요.

꼬리 질문:

- `String.equals` 대신 `MessageDigest.isEqual`을 사용한 이유는 무엇입니까?
- filter를 모든 actuator 경로까지 적용하면 어떤 운영 문제가 생깁니까?
- API key 길이 검증과 실제 secret strength는 같은 개념입니까?

### 30초 모범 답변

filter는 `/internal/admin` 경계에만 적용하고 service-name과 API-key header가 각각 정확히 하나인지 확인합니다. 누락은 인증 정보가 없으므로 401, 중복이나 불일치는 제시된 인증 정보가 유효하지 않으므로 403으로 처리하고 secret은 constant-time 비교를 사용합니다. admin 제어면과 settlement-to-Wallet 호출은 권한과 유출 범위가 다르므로 별도 32자 이상 secret과 caller header를 쓰고, 같은 값을 재사용하면 startup을 실패시킵니다. `toString`과 오류 응답에는 값을 노출하지 않습니다.

### 답변 핵심 키워드

path-scoped filter, exactly-one header, 401 vs 403, constant-time compare, credential separation, fail-fast startup, redaction

### 백지 구현

#### 구현 목표

경로와 multi-value headers를 받아 admin 요청의 인증 결과를 판정하고, 두 서비스 credential의 재사용을 검증하는 순수 컴포넌트를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class InternalAdminAuthentication {
  public AuthDecision authenticate(
      String requestPath,
      Map<String, List<String>> headers,
      ExpectedCredentials expected) {
    // 직접 구현
  }

  public void requireDistinct(String adminSecret, String walletSecret) {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: request path, multi-value header map, expected caller와 secret
- 출력: `BYPASS`, `CONTINUE`, `UNAUTHORIZED`, `FORBIDDEN`

#### 반드시 만족해야 할 조건

- `/internal/admin` 및 그 하위 경로에만 filter를 적용한다.
- caller header와 secret header가 모두 없거나 하나라도 누락되면 unauthorized다.
- 각 header가 두 개 이상이면 forbidden이다.
- caller가 expected identity와 다르면 forbidden이다.
- secret은 byte 기반 constant-time 비교를 사용한다.
- 성공한 경우에만 downstream chain을 계속한다.
- credential은 null·blank·최소 길이 미만이면 startup validation에 실패한다.
- admin과 Wallet secret이 같으면 startup validation에 실패한다.
- 오류 결과와 객체 문자열 표현에 secret을 넣지 않는다.

#### 경계 조건

- 정확히 `/internal/admin`
- `/internal/admin/...` 하위 경로
- `/internal/administrator`처럼 prefix가 비슷하지만 범위 밖인 경로
- 같은 header가 두 번 전달됨
- 빈 문자열 secret
- 같은 길이지만 한 문자만 다른 secret

#### 실패 조건

- admin 외 경로를 잘못 차단
- duplicate header 중 첫 값만 신뢰
- secret을 일반 문자열 메시지에 포함
- caller는 맞지만 secret이 틀린 요청을 통과
- admin과 Wallet secret 재사용 허용
- 비교 중 조기 종료하는 직접 구현

#### 필요한 제약

- HTTP response body 작성은 별도 problem writer로 분리할 수 있다.
- 이 문제의 API key 인증이 사용자 인증, mTLS, 네트워크 정책 전체를 대체한다고 주장하지 않는다.
- secret 최소 길이는 형식 정책일 뿐 엔트로피 보장을 완전히 대체하지 않는다.

### 구현 후 자가 검증

- 범위 밖 경로가 bypass되는가?
- header 누락과 중복이 각각 401/403으로 구분되는가?
- exact caller와 secret만 통과하는가?
- duplicate header 순서를 바꿔도 통과하지 않는가?
- 오류 detail과 `toString`에 secret이 없는가?
- 동일 admin/Wallet secret을 거부하는가?
- constant-time 비교 도구를 실제로 사용했는가?

### 구현 후 설명할 것

1. filter 적용 경계를 좁힌 이유
2. 401과 403을 구분한 계약
3. multi-value header를 정확히 하나로 제한한 이유
4. credential 분리와 최소 권한 원칙의 관계
5. API key 방식의 한계와 보완 가능한 외부 통제

### 원본 확인 위치

- Thread: 18 — 내부 호출 인증과 자격증명 분리
- 대표 커밋: `feat(admin): authenticate internal requests`
- 관련 커밋: `feat(admin): validate operator credentials`, `feat(wallet): write exact authentication headers`, `feat(wallet): require isolated settlement credentials`, `fix(security): reject reused settlement credentials`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminCredentials.java`
- 파일: `src/main/java/com/sportsbook/settlement/admin/AdminAuthenticationFilter.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletAuthenticationHeaders.java`
- 파일: `src/main/java/com/sportsbook/settlement/client/WalletCredentials.java`
- 파일: `src/main/java/com/sportsbook/settlement/config/SettlementCredentialPolicy.java`
- 파일: `src/main/resources/application.yml`
- 관련 메서드: `AdminAuthenticationFilter.shouldNotFilter`, `AdminAuthenticationFilter.doFilterInternal`, `WalletAuthenticationHeaders.apply`
- 관련 Thread: 8, 16, 22

<a id="p25"></a>
<!-- POINT:P25 -->
## P25 — [Thread 20 / `feat(scheduling): isolate settlement workers`] scheduler starvation 격리와 bounded shutdown

### 면접 질문

outbox, lifecycle catch-up, base recovery, revision recovery, correction catch-up을 하나의 큰 thread pool이 아니라 서로 독립된 single-thread scheduler로 분리한 이유는 무엇입니까? shutdown 정책에서 delayed/periodic task 재실행을 막은 이유도 설명해 보세요.

꼬리 질문:

- 공용 pool 크기만 충분히 키우면 격리가 필요 없지 않습니까?
- single-thread scheduler가 각 flow에 주는 장점과 병목은 무엇입니까?
- process shutdown 중 lease를 가진 작업이 끝나지 않으면 어떻게 됩니까?

### 30초 모범 답변

각 worker는 Wallet·Kafka·DB 지연 특성이 달라 한 flow의 block이 다른 복구를 굶길 수 있습니다. 그래서 flow별 pool size 1 scheduler를 둬 failure domain과 cadence를 분리하고, 실제 다중 인스턴스 동시성은 DB lease와 `SKIP LOCKED`가 담당하게 했습니다. 테스트나 제어 환경에서는 property로 workers 전체를 끌 수 있습니다. shutdown은 진행 중 작업을 제한된 시간만 기다리고, 예약된 delayed·periodic task를 새로 실행하지 않으며 cancelled task를 queue에서 제거해 process 종료를 bounded하게 만듭니다.

### 답변 핵심 키워드

failure-domain isolation, starvation prevention, single-thread cadence, database lease scaling, runtime switch, graceful bounded shutdown, cancel cleanup

### 백지 구현

#### 구현 목표

worker 종류마다 독립된 named single-thread scheduled executor를 만들고, 진행 중 작업을 bounded wait한 뒤 안전하게 종료하는 registry를 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
public final class WorkerSchedulerRegistry implements AutoCloseable {
  public ScheduledExecutorService scheduler(WorkerKind kind) {
    // 직접 구현
  }

  @Override
  public void close() {
    // 직접 구현
  }
}
```

#### 입력과 출력

- 입력: worker kind
- 출력: 해당 kind 전용 scheduler
- 종료: 모든 scheduler를 bounded time 안에 종료

#### 반드시 만족해야 할 조건

- 서로 다른 worker kind는 서로 다른 executor instance를 사용한다.
- 각 executor의 pool size는 1이다.
- thread 이름으로 worker 종류를 식별할 수 있다.
- 동일 kind 요청에는 같은 registry-owned executor를 반환한다.
- 한 scheduler의 blocked task가 다른 scheduler task 실행을 막지 않는다.
- close 시 새 task 접수를 막는다.
- 진행 중 task 완료를 제한된 시간 동안 기다린다.
- 시간 안에 끝나지 않으면 취소 또는 강제 종료 정책을 적용한다.
- shutdown 이후 delayed/periodic task가 새로 실행되지 않게 한다.
- `InterruptedException` 발생 시 interrupt 상태를 복원한다.
- close를 여러 번 호출해도 안전하다.

#### 경계 조건

- scheduler를 한 번도 사용하지 않고 close
- 한 worker가 무기한 block
- close와 task 제출 race
- close 두 번 호출
- shutdown 대기 중 현재 thread interrupt
- worker enablement가 false인 구성

#### 실패 조건

- 모든 kind에 같은 executor 반환
- blocked outbox가 lifecycle 실행을 막음
- shutdown 후 periodic task 계속 실행
- 무한 await로 process 종료 불가
- interrupt flag 소실
- executor thread leak

#### 필요한 제약

- 구현 테스트에서 latch를 사용해 한 scheduler를 의도적으로 block하고 다른 scheduler의 실행을 검증한다.
- 실제 Spring 설정과 동일한 이름·정책을 복제할 필요는 없지만 의미는 유지한다.
- scheduler 격리가 DB-level 중복 실행 방지를 대체하지 않는다.

### 구현 후 자가 검증

- kind별 executor가 실제로 다른가?
- blocked task가 있는 동안 다른 kind task가 완료되는가?
- pool size 1로 같은 flow 작업이 직렬 실행되는가?
- close가 설정한 상한 안에 반환되는가?
- shutdown 이후 delayed/periodic task가 시작되지 않는가?
- 강제 종료 뒤 thread가 남지 않는가?
- interrupt 상태를 복원하는가?
- runtime disable 시 scheduler bean 자체가 생성되지 않는 설계를 설명할 수 있는가?

### 구현 후 설명할 것

1. 공용 pool 확장보다 failure-domain 격리를 선택한 이유
2. flow별 single-thread와 DB lease 기반 수평 확장의 역할 분담
3. thread 수 증가와 starvation 방지 사이의 trade-off
4. graceful shutdown에서 대기 상한이 필요한 이유
5. scheduler isolation을 검증하는 동시성 테스트 설계

### 원본 확인 위치

- Thread: 20 — worker 격리와 runtime 제어
- 대표 커밋: `feat(scheduling): isolate settlement workers`
- 관련 커밋: `feat(config): make workers runtime-switchable`, `feat(workers): isolate correction schedulers`, `feat(workers): route correction workers independently`, `feat(workers): bound scheduler shutdown`, `test(scheduling): verify worker isolation`
- 파일: `src/main/java/com/sportsbook/settlement/config/SettlementWorkerConfiguration.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/CorrectionCatchupScanner.java`
- 파일: `src/main/java/com/sportsbook/settlement/correction/RevisionRecoveryScanner.java`
- 클래스·컴포넌트: `SettlementWorkerConfiguration.OUTBOX`, `LIFECYCLE`, `RECOVERY`, `REVISION_RECOVERY`, `CORRECTION`
- 테스트 위치: `SettlementWorkerConfigurationTest`, `SettlementRecoveryIsolationTest`
- 관련 Thread: 10, 11, 15, 17, 21, 22
