# 개발자 기술면접 워크북 01 — 계약, 영속성, 트랜잭션

이 문서는 DevThread의 HTTP 입력 경계, API 응답 계약, PostgreSQL 매핑, 트랜잭션 경계를 면접 문제로 묶는다. 아래의 함수 시그니처는 **백지 구현 연습용 인터페이스**이며, 원본 구현을 옮긴 정답 코드가 아니다.

---

<!-- coverage: SA-01 -->
<a id="sa-01"></a>
## [Thread 02 / `fix: reject overflowing numeric monitor intervals`] JSON 유형과 값 범위를 동시에 강제하는 입력 경계

### 면접 질문

`interval`을 자바의 `int` 필드가 있는 요청 DTO에 바로 역직렬화하지 않고, JSON 노드의 실제 유형을 먼저 확인한 뒤 정확한 정수 변환을 수행한 이유는 무엇입니까? `60`, `60.0`, `60.5`, `"60"`, 매우 큰 지수 표현은 각각 어떻게 다뤄야 합니까?

꼬리 질문:

- 전송 형식 검증과 도메인 값 검증을 한 단계에서 처리할 때 생기는 장단점은 무엇입니까?
- 변환 중 `ArithmeticException`과 `NumberFormatException`을 모두 고려해야 하는 이유는 무엇입니까?
- 잘못된 요청이 저장소를 한 번도 변경하지 않았음을 어떻게 검증하겠습니까?
- JSON 문서 뒤에 두 번째 토큰이 붙는 경우를 필드 검증만으로 막을 수 있습니까?

### 30초 모범 답변

요청 DTO 자동 바인딩만 사용하면 숫자처럼 보이는 여러 JSON 표현이 같은 자바 값으로 합쳐져, API가 약속한 정확한 전송 유형을 놓칠 수 있습니다. 그래서 루트가 객체인지, 각 필드가 문자열·숫자·불리언인지 먼저 확인하고, `interval`은 소수부가 없는 값만 정확히 `int`로 바꾼 뒤 1초에서 86,400초 범위를 검사했습니다. 변환 실패와 범위 초과는 모두 안전한 400 오류로 통일하고, 저장 호출 전 검증을 끝내 잘못된 입력이 상태를 바꾸지 않게 했습니다. 문서 뒤 추가 토큰은 Jackson의 trailing-token 거부 설정으로 별도 차단했습니다.

### 답변 핵심 키워드

전송 유형, 도메인 제약, exact conversion, 정수 오버플로, trailing token, 검증 선행, 무변경 보장, 400 오류

### 백지 구현

#### 구현 목표

다음 JSON 요청을 검증해 `CreateMonitor`를 만들거나 입력 오류를 반환한다. 편의상 예외 형식은 연습 코드에서 정해도 되지만, 모든 입력 오류가 동일한 공개 오류 범주로 수렴해야 한다.

#### 인터페이스 또는 함수 시그니처

```java
record CreateMonitor(String name, String url, int interval, boolean enabled) {}

static CreateMonitor fromJson(JsonNode body) {
    // 직접 구현
}
```

#### 입력과 출력

- 입력: 역직렬화된 `JsonNode`
- 출력: 정규화가 끝난 `CreateMonitor`
- 실패: 공개용 입력 오류. 내부 변환 예외 메시지는 노출하지 않는다.

#### 반드시 만족해야 할 조건

- 루트는 JSON 객체여야 한다.
- `name`, `url`은 JSON 문자열, `interval`은 JSON 숫자, `enabled`는 JSON 불리언이어야 한다.
- `name`은 앞뒤 공백 제거 후 비어 있지 않고, UTF-16 코드 단위 기준 최대 100이어야 한다.
- `interval`은 수학적으로 정수여야 하며 정확히 `int`로 표현 가능해야 한다.
- `interval`의 허용 범위는 1~86,400이다.
- 입력 검증이 끝나기 전에는 저장소나 외부 I/O를 호출하지 않는다.

#### 경계 조건

- `1`, `86400`
- `60.0`
- `60.5`
- `"60"`
- 필드 누락, `null`, 잘못된 루트 배열
- `int` 범위를 벗어나는 숫자와 매우 큰 지수 표현
- 공백만 있는 이름, 길이 100과 101인 이름

#### 실패 조건

- 형식 또는 범위가 하나라도 맞지 않으면 전체 요청을 거부한다.
- 변환 라이브러리가 던진 예외 문자열을 응답에 포함하지 않는다.
- 일부 필드만 적용하거나 기본값으로 보정하지 않는다.

#### 필요한 제약

- 정규식으로 숫자 문자열을 다시 해석하지 않는다.
- 부동소수점으로 먼저 변환한 뒤 반올림하지 않는다.
- 구현 시간은 15~20분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 정상 입력이 정확한 필드 값으로 변환된다.
- [ ] `60`과 `60.0`의 정책이 요구사항과 일치한다.
- [ ] 소수, 문자열 숫자, 불리언 숫자가 거부된다.
- [ ] 최솟값·최댓값과 그 바로 바깥 값이 구분된다.
- [ ] 매우 큰 숫자가 500이 아니라 입력 오류가 된다.
- [ ] 모든 실패에서 저장 함수 호출 횟수가 0이다.
- [ ] 오류 응답에 파서·변환기 예외 상세가 없다.
- [ ] 시간 복잡도는 입력 필드 수와 문자열 길이에 선형이고 별도 대용량 복사가 없다.

### 구현 후 설명할 것

1. 자동 DTO 바인딩보다 명시적 노드 검증을 택한 이유와 그 비용
2. 전송 형식 제약과 도메인 값 제약을 나눈 기준
3. exact conversion으로 반올림·오버플로를 막은 방식
4. 잘못된 입력에서 상태 변화가 없도록 만든 호출 순서
5. trailing-token 검사는 함수 바깥의 파서 설정이 담당하는 이유

### 원본 확인 위치

- Thread 02
- 커밋: `fix: reject overflowing numeric monitor intervals`
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `MonitorController.CreateMonitor.fromJson`
- `backend/src/main/resources/application.properties`
- `backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java`
- 관련 Thread: Thread 01의 초기 요청 처리, Thread 03의 저장 전 검증 경계

---

<!-- coverage: SA-02 -->
<a id="sa-02"></a>
## [Thread 02 / `fix: reject overflowing numeric monitor intervals`] 안전한 API 오류 봉투와 클라이언트 런타임 검증

### 면접 질문

서버가 TypeScript 타입과 같은 모양의 JSON을 준다고 가정하지 않고, 브라우저에서 성공 응답과 오류 응답을 다시 검증한 이유는 무엇입니까? 예상하지 못한 서버 예외를 그대로 직렬화하지 않고 고정된 `INTERNAL_ERROR` 봉투로 바꾼 이유도 설명해 주세요.

꼬리 질문:

- HTTP 200이지만 성공 payload가 깨져 있다면 UI는 성공으로 처리해야 합니까?
- 오류 메시지 문구 대신 상태 코드와 오류 코드를 기준으로 분기해야 하는 이유는 무엇입니까?
- 네트워크 실패, 비정상 JSON, 잘못된 오류 봉투를 어떻게 같은 안전 범주로 수렴시키겠습니까?
- 서버 로그에 남길 정보와 클라이언트에 공개할 정보를 어떻게 나누겠습니까?

### 30초 모범 답변

TypeScript 타입은 컴파일 시점의 약속일 뿐 네트워크에서 들어오는 JSON을 검증하지 않습니다. 그래서 HTTP 상태, 최상위 `data` 또는 `error`의 단일 봉투, 그리고 payload의 실제 필드 유형을 런타임에서 확인해야 합니다. 성공 상태라도 payload가 깨졌다면 캐시나 화면을 갱신하지 않고 실패로 처리합니다. 서버의 예상하지 못한 예외는 내부에 기록하되 응답에는 고정된 `INTERNAL_ERROR` 코드와 일반 메시지만 보내 구현 세부사항과 민감한 값을 숨깁니다. UI 분기는 변할 수 있는 문구가 아니라 상태와 안정된 오류 코드에 둡니다.

### 답변 핵심 키워드

컴파일 타입과 런타임 데이터, discriminated envelope, fail closed, stable error code, 정보 노출 방지, 성공 오인 방지, 캐시 무결성

### 백지 구현

#### 구현 목표

`Response`가 성공이면 검증된 `data`만 반환하고, 실패이면 검증된 오류 코드 또는 안전한 대체 오류를 던지는 제네릭 판독기를 구현한다.

#### 인터페이스 또는 함수 시그니처

```ts
type ApiErrorCode =
  | 'INVALID_INPUT'
  | 'NOT_FOUND'
  | 'UNAUTHENTICATED'
  | 'FORBIDDEN'
  | 'CONFLICT'
  | 'INTERNAL_ERROR';

type ApiFailure = { code: ApiErrorCode; status: number };

async function readData<T>(
  response: Response,
  valid: (value: unknown) => value is T,
): Promise<T> {
  // 직접 구현
}
```

#### 입력과 출력

- 입력: 실제 `fetch` 응답과 payload 타입 가드
- 출력: 검증이 끝난 `T`
- 실패: 안정된 코드와 HTTP 상태만 가진 오류 객체

#### 반드시 만족해야 할 조건

- 성공 응답은 최상위에 `data`만 있어야 하며 `valid(data)`가 참이어야 한다.
- 실패 응답은 최상위에 `error`만 있어야 하고 허용된 오류 코드를 가져야 한다.
- HTTP 상태와 봉투 의미가 모순되면 성공으로 간주하지 않는다.
- JSON 파싱 실패, 알 수 없는 오류 코드, 깨진 성공 payload는 안전한 내부 오류 범주로 처리한다.
- 서버가 보낸 임의 문구를 분기 조건이나 예외 식별자로 사용하지 않는다.

#### 경계 조건

- 200 + 정상 `data`
- 200 + `error`
- 500 + 정상 `error`
- 500 + HTML 또는 빈 본문
- 200 + 배열이어야 하는데 객체인 `data`
- `data`와 `error`가 동시에 존재
- 알려지지 않은 오류 코드
- 네트워크 예외로 `Response` 자체가 없는 경우와의 조합

#### 실패 조건

- 불완전한 payload를 부분적으로 캐시에 넣지 않는다.
- 서버 메시지나 스택 추적을 사용자용 오류 객체에 보존하지 않는다.
- 상태가 실패인데 기본값을 반환하지 않는다.

#### 필요한 제약

- 외부 스키마 검증 라이브러리 없이 20분 안에 작성한다.
- 타입 단언만으로 검증을 대체하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 성공만 `T`로 반환된다.
- [ ] 깨진 2xx 응답이 성공처럼 보이지 않는다.
- [ ] 4xx·5xx의 허용 코드가 상태와 함께 보존된다.
- [ ] 서버 문구가 달라져도 UI 분류 결과가 같다.
- [ ] JSON 파싱 실패와 네트워크 실패가 안전한 오류로 귀결된다.
- [ ] `data`와 `error` 동시 존재를 거부한다.
- [ ] 실패 후 기존 화면·캐시 상태를 유지할 수 있는 인터페이스다.
- [ ] 검증 비용이 payload 크기에 선형이다.

### 구현 후 설명할 것

1. TypeScript 정적 타입이 네트워크 신뢰 경계를 넘지 못하는 이유
2. 성공 상태와 payload 모양을 함께 검증하는 방식
3. 오류 문구 대신 코드에 의존하는 호환성 이점
4. 서버 내부 예외와 공개 오류를 분리한 기준
5. fail-open보다 fail-closed를 택한 이유

### 원본 확인 위치

- Thread 02
- 커밋: `fix: reject overflowing numeric monitor intervals`
- `backend/src/main/java/dev/evolution/monitor/ApiErrors.java`
- `backend/src/test/java/dev/evolution/monitor/ApiErrorBoundaryTest.java`
- `app/monitors/api.ts`
- `tests/browser/errors.spec.ts`
- 관련 Thread: Thread 06의 실패 시 캐시 보존, Thread 08의 서버 초기 데이터 검증, Thread 14의 오류 상세 비공개

---

<!-- coverage: SA-03 -->
<a id="sa-03"></a>
## [Thread 03 / `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록`] 도메인·ORM·PostgreSQL 표현의 정규형 유지

### 면접 질문

도메인 객체, JPA 엔티티, PostgreSQL 행 사이에 명시적인 변환 함수를 둔 이유는 무엇입니까? 특히 마이크로초 정밀도의 시각, `0`과 `null`, 불리언, nullable 결과 필드를 첫 응답부터 데이터베이스의 정규형과 일치시켜야 하는 이유를 설명해 주세요.

꼬리 질문:

- 애플리케이션이 나노초 시각을 응답한 뒤 DB가 마이크로초로 잘라 저장하면 어떤 문제가 생깁니까?
- 데이터베이스 세션 시간대가 `Asia/Seoul`이어도 같은 `Instant`를 복원하려면 무엇을 확인해야 합니까?
- 컬럼이 빠졌거나, ORM이 모르는 필수 컬럼이 추가된 상태에서 서버를 띄우면 어떤 실패 방식이 바람직합니까?
- `0`과 `null`을 구분하지 못하는 매핑 버그를 어떻게 찾겠습니까?

### 30초 모범 답변

도메인과 저장 표현을 암묵적으로 섞으면 재시작 전후 값이 달라지거나 `0`, `false`, `null` 같은 경계값이 손실될 수 있습니다. 이 작업에서는 엔티티 변환을 명시하고, PostgreSQL이 보존하는 마이크로초 정밀도로 첫 응답 전부터 정규화해 메모리 값과 재조회 값을 같게 만들었습니다. 시간은 `Instant`와 `timestamp with time zone` 의미를 기준으로 검증하고, nullable 실행 결과는 상태에 맞게 유지했습니다. 또 시작 시 스키마 호환성을 확인해 누락 컬럼이나 매핑되지 않은 필수 컬럼이 있으면 요청을 받기 전에 실패하게 했습니다.

### 답변 핵심 키워드

canonical representation, round trip, timestamp precision, UTC instant, null semantics, explicit mapping, schema compatibility, fail fast

### 백지 구현

#### 구현 목표

도메인 모델과 영속 엔티티 간 왕복 변환을 구현하고, 저장소가 허용하는 시각 정밀도와 nullable 필드 invariant를 지킨다.

#### 인터페이스 또는 함수 시그니처

```java
record CheckRun(
    UUID id,
    UUID monitorId,
    String state,
    Integer httpStatus,
    Long latencyMs,
    String failureReason,
    Instant startedAt,
    Instant finishedAt
) {}

final class CheckRunEntity {
    static CheckRunEntity fromDomain(CheckRun run) {
        // 직접 구현
    }

    CheckRun toDomain() {
        // 직접 구현
    }
}
```

#### 입력과 출력

- 입력: 유효한 도메인 `CheckRun` 또는 데이터베이스에서 읽은 엔티티
- 출력: 반대 표현
- 목표: `toDomain(fromDomain(x))`가 저장 가능한 정규형의 `x`와 동일

#### 반드시 만족해야 할 조건

- 시각은 데이터베이스가 보존하는 마이크로초 정밀도로 정규화한다.
- `0`인 지연 시간은 `null`로 바뀌지 않는다.
- 아직 실행되지 않은 상태의 결과 필드는 `null`을 유지한다.
- 터미널 상태의 필수 결과 필드는 누락되지 않는다.
- ID와 외래키는 왕복 과정에서 바뀌지 않는다.
- 시스템 기본 시간대에 의존하지 않는다.

#### 경계 조건

- 나노초가 포함된 `Instant`
- epoch 근처 시각과 음수 epoch 시각
- `latencyMs = 0`
- `httpStatus = null`
- 모든 nullable 필드가 비어 있는 대기 상태
- 실패 상태에서 HTTP 상태가 없는 경우

#### 실패 조건

- 상태와 결과 필드 조합이 모순되면 조용히 보정하지 않고 거부한다.
- 시각을 로컬 날짜·시간 문자열로 변환해 저장하지 않는다.
- primitive 기본값 때문에 `null`이 `0`으로 바뀌지 않게 한다.

#### 필요한 제약

- ORM 어노테이션 작성보다 변환 로직과 invariant에 집중한다.
- 구현 시간은 20분을 기준으로 한다.

### 구현 후 자가 검증

- [ ] 나노초 입력이 예상한 마이크로초 값으로 한 번만 정규화된다.
- [ ] 왕복 후 ID, 상태, 외래키가 동일하다.
- [ ] `0`, `false`, `null`이 서로 바뀌지 않는다.
- [ ] JVM 기본 시간대를 바꿔도 같은 `Instant`가 나온다.
- [ ] 대기·실행·성공·실패 상태별 필드 조합이 invariant를 만족한다.
- [ ] 재조회 값과 첫 API 응답을 직접 비교할 수 있다.
- [ ] 잘못된 상태 조합을 저장하기 전에 발견한다.

### 구현 후 설명할 것

1. 첫 응답부터 DB 정밀도로 정규화해야 하는 이유
2. 엔티티를 API 모델로 직접 노출하지 않은 이유
3. nullable wrapper 타입을 사용한 필드와 primitive를 사용한 필드의 기준
4. 시간대와 시각 정밀도를 별개 문제로 본 이유
5. 스키마 호환성 검사를 애플리케이션 준비 완료 전에 수행하는 이유

### 원본 확인 위치

- Thread 03
- 커밋: `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록`
- `backend/src/main/java/dev/evolution/monitor/Monitor.java`
- `backend/src/main/java/dev/evolution/monitor/MonitorEntity.java`
- `MonitorEntity.fromDomain`, `MonitorEntity.toDomain`
- `backend/src/main/java/dev/evolution/monitor/CheckRunEntity.java`
- `CheckRunEntity.fromDomain`, `CheckRunEntity.toDomain`
- `backend/src/main/java/dev/evolution/monitor/SchemaCompatibility.java`
- `backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java`
- 관련 Thread: Thread 09의 상태별 nullable 필드, Thread 11의 `ABORTED` 상태

---

<!-- coverage: SA-04 -->
<a id="sa-04"></a>
## [Thread 03 / `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록`] 짧은 트랜잭션과 외부 I/O 경계

### 면접 질문

수동 HTTP Check를 처리할 때 Monitor를 읽는 트랜잭션, 외부 HTTP 요청, 결과를 저장하는 트랜잭션을 분리한 이유는 무엇입니까? 하나의 `@Transactional` 메서드 안에서 전부 수행하면 어떤 문제가 생깁니까?

꼬리 질문:

- 외부 I/O 중 트랜잭션과 DB 커넥션을 오래 잡으면 어떤 자원 고갈과 잠금 문제가 생깁니까?
- Spring 프록시 기반 트랜잭션에서 같은 클래스의 메서드를 내부 호출하면 어떤 함정이 있습니까?
- 외부 호출은 성공했지만 결과 저장이 실패하면 어떤 의미론이 남습니까?
- 이 경계는 Thread 09의 영속 큐로 어떻게 발전했습니까?

### 30초 모범 답변

외부 HTTP는 지연과 실패 시간이 데이터베이스보다 훨씬 불확실하므로 트랜잭션 안에 두면 커넥션과 잠금을 불필요하게 오래 점유합니다. 그래서 소유권과 URL을 짧은 읽기 트랜잭션에서 확인하고 커밋한 뒤, 외부 호출은 트랜잭션 없이 수행하고, 관측 결과만 별도의 짧은 쓰기 트랜잭션으로 저장했습니다. Spring 프록시를 실제로 통과하도록 경계를 분리해야 `@Transactional`이 적용됩니다. 다만 외부 호출 후 저장 실패라는 빈틈은 남기 때문에 이후 Thread에서는 먼저 실행 의도를 영속화하고 worker가 같은 ID를 진행하는 구조로 보강했습니다.

### 답변 핵심 키워드

transaction scope, connection pool, lock duration, external I/O, proxy boundary, partial failure, durable intent, short transaction

### 백지 구현

#### 구현 목표

외부 HTTP 작업을 데이터베이스 트랜잭션 밖에서 실행하면서, 읽기와 결과 저장은 각각 명확한 트랜잭션 경계를 갖도록 서비스 흐름을 작성한다.

#### 인터페이스 또는 함수 시그니처

```java
interface MonitorTransactions {
    MonitorTarget loadAuthorizedTarget(UUID ownerId, UUID monitorId);
    CheckRun saveObservedResult(UUID ownerId, CheckRun result);
}

interface OutboundChecker {
    CheckRun observe(UUID monitorId, String url);
}

final class CheckService {
    CheckRun execute(UUID ownerId, UUID monitorId) {
        // 직접 구현
    }
}
```

#### 입력과 출력

- 입력: 인증된 사용자 ID와 Monitor ID
- 출력: 저장이 완료된 `CheckRun`
- 실패: 권한 없음·대상 없음·외부 실패·저장 실패를 구분 가능한 내부 흐름

#### 반드시 만족해야 할 조건

- 대상 조회가 끝난 뒤 읽기 트랜잭션이 종료되어야 한다.
- 외부 I/O 시점에는 활성 DB 트랜잭션이 없어야 한다.
- 결과 저장은 별도 쓰기 트랜잭션에서 수행한다.
- 외부 호출 전 소유권을 검증한다.
- 결과 저장 시에도 사용자와 Monitor 관계를 다시 확인하거나, 동등한 무결성 장치를 둔다.

#### 경계 조건

- Monitor가 조회 직후 삭제됨
- 외부 HTTP 타임아웃
- 결과 저장 직전 DB 장애
- 같은 Monitor에 대한 동시 실행
- 외부 성공 후 저장 실패

#### 실패 조건

- 외부 요청 중 DB 커넥션을 점유하지 않는다.
- 권한 없는 대상에 외부 요청을 보내지 않는다.
- 저장 실패를 성공으로 응답하지 않는다.
- 예외를 삼켜 가짜 성공 결과를 만들지 않는다.

#### 필요한 제약

- 실제 Spring 설정 없이 인터페이스와 호출 순서만 15분 안에 구현한다.
- 트랜잭션 어노테이션은 `MonitorTransactions` 구현에 있다고 가정한다.

### 구현 후 자가 검증

- [ ] 정상 경로에서 조회 → 외부 I/O → 저장 순서가 지켜진다.
- [ ] 외부 I/O 시 활성 트랜잭션이 없음을 테스트할 수 있다.
- [ ] 조회 실패 시 외부 호출과 저장 호출이 모두 0회다.
- [ ] 외부 실패 시 정책에 맞는 실패 결과만 저장되거나 예외가 전파된다.
- [ ] 저장 실패가 클라이언트 성공으로 바뀌지 않는다.
- [ ] 삭제·변경 경합에서 데이터 invariant가 깨지지 않는다.
- [ ] DB 커넥션 점유 시간은 두 짧은 구간으로 제한된다.

### 구현 후 설명할 것

1. 트랜잭션을 외부 I/O 바깥으로 밀어낸 자원·잠금상의 이유
2. 읽기와 쓰기를 한 트랜잭션으로 묶지 않은 trade-off
3. 프록시 기반 트랜잭션의 self-invocation 함정
4. 외부 성공 후 저장 실패를 완전히 없애지 못하는 이유
5. 그 빈틈을 영속 큐와 상태 머신으로 개선하는 방법

### 원본 확인 위치

- Thread 03
- 커밋: `PostgreSQL 재시작과 스키마 거부의 실제 검증 결과 기록`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `MonitorStore`의 읽기·쓰기 메서드와 `@Transactional` 경계
- `backend/src/main/java/dev/evolution/monitor/MonitorController.java`
- `backend/src/test/java/dev/evolution/monitor/PostgresPersistenceTest.java`
- 관련 Thread: Thread 01의 동기 Check, Thread 09의 API/worker 분리, Thread 10의 실행 소유권, Thread 11의 lease 복구
