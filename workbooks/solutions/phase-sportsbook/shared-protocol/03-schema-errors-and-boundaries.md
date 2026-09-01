# 스키마, 오류, 시스템 경계 면접 워크북

이 문서는 Schema Registry 없이 고정된 Avro binary 계약을 운영하는 방법, 서비스 경계를 넘는 오류 모델, Java·JSON·Avro 표현과 책임 소유권을 다룬다. 단순 직렬화 사용법보다 변경 안전성과 의존성 경계에 초점을 둔다.

<a id="i06"></a>
## [Thread 02 / `test(events): lock wire v1 fingerprints`] Schema Registry 없는 Avro Wire V1 잠금

### 면접 질문

writer schema id나 Schema Registry가 없는 Avro binary 환경에서, 기존 record에 optional field 하나를 추가하는 것도 왜 안전한 rolling change로 볼 수 없나요?

꼬리 질문:

- 필드 순서를 테스트로 고정한 이유는 무엇인가요?
- parsing canonical fingerprint 하나만 검사하면 충분하지 않은 이유는 무엇인가요?
- named schema인 `Money`와 `SettlementResultAvro`의 import·parse 순서가 왜 중요하나요?
- generated class를 직접 수정하면 안 되는 이유는 무엇인가요?
- 직렬화 round trip 테스트에서 encoder flush와 resource cleanup을 빠뜨리면 어떤 문제가 생길 수 있나요?

### 30초 모범 답변

이 환경은 payload에 writer schema 식별자가 없고 consumer가 보유한 generated schema로 바로 읽기 때문에, 일반적인 reader/writer schema resolution을 배포 중에 활용할 수 없습니다. 따라서 V1 record의 이름, namespace, 필드 순서와 타입, enum 순서, default, logical type을 고정해야 합니다. canonical fingerprint는 구조 변경 탐지에 유용하지만 default와 logical type의 의미를 모두 보존하지 않으므로 별도 assertion과 실제 binary round trip이 함께 필요합니다. 새 의미는 기존 record 수정 대신 새 record와 topic으로 확장합니다.

### 답변 핵심 키워드

writer schema 부재, mixed deployment, generated schema, field order, named type, parser order, canonical fingerprint, default assertion, logical type, binary round trip, additive record/topic

### 백지 구현

**구현 목표**

SpecificRecord의 binary round trip과 스키마 계약 핵심 요소를 검증하는 작은 테스트 지원 코드를 구현한다.

**인터페이스 또는 함수 시그니처**

```java
final class WireContractChecks {

  static <T extends SpecificRecord> T roundTrip(T expected, Class<T> type)
      throws IOException {
    // 직접 구현
  }

  static List<String> fieldNames(Schema schema) {
    // 직접 구현
  }

  static long fingerprint64(Schema schema) {
    // 직접 구현
  }

  private WireContractChecks() {}
}
```

**입력과 출력**

- `roundTrip`: generated `SpecificRecord`와 해당 Java 타입을 받아 binary encode/decode 결과를 반환
- `fieldNames`: Avro `Schema`를 받아 선언 순서의 필드 이름 목록을 반환
- `fingerprint64`: Avro schema를 받아 parsing canonical form 기반 64비트 fingerprint를 반환

**반드시 만족해야 할 조건**

- binary encoder에 쓴 모든 내용을 decode 전에 flush한다.
- 임시 출력 resource는 예외가 나도 정리한다.
- writer는 입력 record의 schema를 사용한다.
- reader는 요청한 generated 타입을 복원한다.
- 필드 이름은 정렬하지 않고 선언 순서를 보존한다.
- fingerprint와 별도로 default 유무·값, `timestamp-millis` logical type을 검사할 수 있어야 한다.
- 현재 Wire V1의 top-level record inventory가 14개라는 계약을 별도 테스트로 고정한다.
- named schema 파싱 시 `Money.avsc`, `BetSettled.avsc`, 나머지 schema 순서를 존중한다.

**경계 조건**

- optional union 필드가 null인 record
- timestamp logical type 필드가 있는 record
- 공통 named type을 재사용하는 record
- 빈 collection 또는 빈 map을 가진 record
- 예상 inventory에서 schema 하나가 추가·삭제된 경우
- 필드 이름은 같지만 순서가 달라진 경우
- fingerprint는 같다고 판단하기 어려운 default 또는 logical type 변화

**실패 조건**

- flush 전에 byte 배열을 읽어 payload 일부가 누락됨
- 필드 이름 목록을 정렬해 순서 변경을 놓침
- canonical fingerprint만으로 모든 계약 의미를 검증했다고 가정함
- schema 파일 순서를 무시해 named type 중복 정의·미해결 오류 발생
- generated source를 수정해 다음 빌드에서 변경이 사라짐

**필요한 제약**

- Apache Avro의 SpecificDatumWriter/SpecificDatumReader 계열을 사용한다.
- 테스트 지원 코드가 Kafka client나 Schema Registry에 의존하지 않는다.
- 실제 expected fingerprint 상수 목록은 원본을 보기 전에 작성하지 않아도 된다.

### 구현 후 자가 검증

- [ ] 대표 record가 encode/decode 후 값 동등성을 유지한다.
- [ ] encoder flush가 decode 전에 수행된다.
- [ ] 출력 resource가 정상·예외 경로에서 정리된다.
- [ ] 필드 순서 변경을 테스트가 탐지한다.
- [ ] top-level schema 추가·삭제를 inventory 테스트가 탐지한다.
- [ ] named type fullname과 재사용 여부를 검사한다.
- [ ] null default와 required field를 구분한다.
- [ ] timestamp logical type 누락을 별도 검사한다.
- [ ] fingerprint의 한계를 설명할 수 있다.
- [ ] 전체 검증 비용이 schema·필드 수에 선형적으로 비례하는지 확인했다.

### 구현 후 설명할 것

1. Registry와 writer schema id 부재가 rolling deployment 안전성에 미치는 영향
2. round trip, inventory, fingerprint, default/logical type 검증이 서로 보완하는 관계
3. parsing canonical fingerprint가 포함하지 않는 의미를 별도로 잠그는 이유
4. named schema 정의를 한 곳에 두고 import 순서를 관리하는 이유
5. 기존 record 수정 대신 새 record/topic을 추가하는 비용과 장점

### 원본 확인 위치

- Thread 02
- 커밋: `build(avro): generate protocol records`
- 커밋: `feat(events): define wire monetary amounts`
- 커밋: `test(events): verify wire monetary amounts`
- 커밋: `build(avro): order shared named schemas`
- 커밋: `test(events): lock wire v1 schema inventory`
- 커밋: `test(events): lock wire v1 fingerprints`
- 파일: `src/main/avro/com/sportsbook/protocol/event/Money.avsc`
- 파일: `src/test/java/com/sportsbook/protocol/event/AvroRecordTestSupport.java`
- 파일: `src/test/java/com/sportsbook/protocol/event/WireSchemaTestSupport.java`
- 파일: `src/test/java/com/sportsbook/protocol/event/WireSchemaInventoryTest.java`
- 파일: `src/test/java/com/sportsbook/protocol/event/WireSchemaFingerprintTest.java`
- 함수·클래스·컴포넌트: `AvroRecordTestSupport.assertFields`, `AvroRecordTestSupport.roundTrip`, `WireSchemaTestSupport.loadSchemas`, `WireSchemaTestSupport.schemaOrder`
- 관련 Thread: 01, 10, 11, 12, 13, 14, 16

---

<a id="i07"></a>
## [Thread 09 / `feat(errors): define problem details`] 안정적인 오류 어휘와 프레임워크 중립 Problem Detail

### 면접 질문

공통 라이브러리에서 Spring의 `ProblemDetail`을 직접 사용하지 않고 자체 JSON record를 둔 이유는 무엇인가요? 어떤 오류만 공통 `ErrorCode`에 포함해야 하나요?

꼬리 질문:

- HTTP status만으로 클라이언트가 오류를 안정적으로 분기하기 어려운 이유는 무엇인가요?
- `errorCode`, `type URI`, `title` 중 어떤 값이 기계 판별용이고 어떤 값이 표시용인가요?
- `SERVICE_UNAVAILABLE`과 업무 거절 오류의 retry 의미는 어떻게 달라야 하나요?
- `detail`, `instance`, `correlationId`를 nullable로 두고 JSON에서 생략하는 장단점은 무엇인가요?
- 공통 enum이 모든 서비스 내부 오류를 흡수하면 어떤 결합이 생기나요?

### 30초 모범 답변

공통 계약은 웹 서버뿐 아니라 background consumer와 batch도 사용하므로 spring-web 타입을 노출하지 않고 프레임워크 중립 record로 유지했습니다. 클라이언트는 가변적인 메시지가 아니라 안정적인 `errorCode`와 type URI로 분기하고, title과 detail은 설명에 사용합니다. 공통 enum에는 서비스 경계를 넘어 동일한 의미가 필요한 오류만 두며 내부 구현 오류는 각 서비스가 소유합니다. 일시적 의존성 장애와 영구적인 업무 거절은 status와 retry 정책에서 구분해야 합니다.

### 답변 핵심 키워드

stable machine code, RFC 7807 shape, framework-neutral, transitive dependency, retry semantics, boundary vocabulary, optional extension, correlation, shared vs local error

### 백지 구현

**구현 목표**

안정적인 오류 메타데이터를 가진 enum과, 선택적 요청 문맥을 담는 프레임워크 중립 Problem Detail 값을 구현한다.

**인터페이스 또는 함수 시그니처**

```java
public enum ErrorCode {
  // 공통 경계 오류를 직접 정의

  ErrorCode(int httpStatus, String typeSuffix, String title) {
    // 직접 구현
  }

  public ProblemDetail toProblemDetail(
      String detail,
      URI instance,
      String correlationId) {
    // 직접 구현
  }
}

public record ProblemDetail(
    URI type,
    String title,
    int status,
    String errorCode,
    String detail,
    URI instance,
    String correlationId) {

  public ProblemDetail {
    // 직접 구현
  }
}
```

**입력과 출력**

- 입력: 오류별 HTTP status, type suffix, title와 요청별 선택 문맥
- 출력: stable code와 메타데이터를 가진 `ProblemDetail`

**반드시 만족해야 할 조건**

- 모든 오류의 status는 400 이상 600 미만이다.
- type URI는 오류별로 고유하고 안정적이다.
- title은 비어 있지 않다.
- `errorCode`는 기계가 안정적으로 판별할 수 있는 값이다.
- `SERVICE_UNAVAILABLE` 계열은 일시 장애임을 표현할 수 있는 5xx 의미를 가진다.
- 검증 실패, 잔액 부족, 한도 초과 같은 업무 거절은 4xx로 분류한다.
- `type`, `title`, `errorCode`는 null이 아니다.
- 선택적 `detail`, `instance`, `correlationId`는 없을 때 JSON에서 생략할 수 있다.
- Spring Web 타입이나 HTTP adapter 구현을 공통 모델에 넣지 않는다.

**경계 조건**

- 최소 정보만 가진 Problem Detail
- 세 선택 필드가 모두 채워진 값
- 중복 type URI
- 빈 title
- 399 또는 600 같은 범위 밖 status
- 내부 서비스 전용 오류를 공통 enum에 추가하려는 경우
- 같은 `errorCode`에 표시 문구만 바꾸는 경우

**실패 조건**

- 클라이언트가 자연어 `detail`을 파싱해야 함
- 프레임워크 타입이 공통 JAR의 전이 의존성으로 노출됨
- 동일 오류가 서비스마다 다른 code나 status를 사용함
- 모든 내부 예외를 공통 enum에 넣어 배포 결합이 커짐
- correlation id를 필수로 만들어 비관측성 경로의 직렬화를 깨뜨림

**필요한 제약**

- HTTP adapter가 이 값을 status와 body로 변환한다고 가정한다.
- retry 실행 자체는 구현하지 않는다.
- 사용자 현지화 문구는 공통 enum의 책임으로 두지 않는다.

### 구현 후 자가 검증

- [ ] 오류별 type URI가 중복되지 않는다.
- [ ] 모든 title이 비어 있지 않다.
- [ ] status 범위를 검증한다.
- [ ] 일시 장애와 업무 거절의 분류가 구분된다.
- [ ] 필수 필드 null을 거부한다.
- [ ] 선택 필드가 없을 때 wire에서 생략된다.
- [ ] 선택 필드가 있을 때 round trip으로 보존된다.
- [ ] Spring 또는 특정 web framework에 대한 의존이 없다.
- [ ] 공통 오류와 서비스 내부 오류의 경계를 설명할 수 있다.

### 구현 후 설명할 것

1. HTTP status와 stable business error code가 각각 맡는 역할
2. 자체 Problem Detail이 비웹 consumer와 의존성 관리에 주는 이점
3. retry 가능 오류를 업무 거절과 구분하는 기준
4. 선택 필드를 생략하는 wire 설계의 장단점
5. 공통 오류 목록을 작게 유지해야 하는 이유

### 원본 확인 위치

- Thread 09
- 커밋: `feat(errors): define protocol error codes`
- 커밋: `test(errors): verify error classifications`
- 커밋: `test(errors): verify retry semantics`
- 커밋: `feat(errors): define problem details`
- 커밋: `test(errors): verify problem detail invariants`
- 파일: `src/main/java/com/sportsbook/protocol/error/ErrorCode.java`
- 파일: `src/main/java/com/sportsbook/protocol/error/ProblemDetail.java`
- 클래스·컴포넌트: `ErrorCode`, `ProblemDetail`, `ErrorCode.toProblemDetail`
- 테스트: `ErrorCodeTest`, `ErrorRetryTest`, `ProblemDetailTest`
- 관련 Thread: 06, 11, 15, 16

---

<a id="i08"></a>
## [Thread 15 / `docs(project): document shared protocol`] Java·JSON·Avro 표현과 계약 소유권

### 면접 질문

같은 `Money`나 `BetSlip` 개념을 Java domain, JSON API, Avro event에서 하나의 자동 공유 모델로 취급하지 않고 경계별 표현과 adapter를 둔 이유는 무엇인가요?

꼬리 질문:

- 공통 라이브러리가 "여러 서비스가 공유해야 하는 최소 계약"을 넘어서면 어떤 문제가 생기나요?
- `Money`가 안전한 산술은 제공하지만 balance·ledger·payout 정책을 소유하지 않는 이유는 무엇인가요?
- `BetSlip`의 구조적 자기 일관성과 betting 서비스의 승인 정책을 어떻게 구분하나요?
- event schema가 존재한다고 producer·consumer·topic 운영까지 공통 라이브러리 책임이 되는 것은 왜 아닌가요?
- 자동 매핑과 명시적 adapter 중 무엇을 선택하고 어떤 테스트를 두겠나요?

### 30초 모범 답변

Java, JSON, Avro는 목적과 변경 제약이 다릅니다. Java 모델은 타입 안전성과 생성 불변식, JSON은 동기 API의 공개 shape, Avro는 record 이름·필드 순서·named type까지 포함한 비동기 계약을 책임집니다. 세 표현을 하나로 묶으면 한 경계의 변경이 다른 경계를 강제로 바꾸므로 adapter에서 명시적으로 변환하고 경계별 테스트를 둡니다. 공통 라이브러리는 wire shape와 여러 소비자에게 필요한 최소 불변식만 소유하고, 저장소·트랜잭션·업무 정책은 각 서비스에 남깁니다.

### 답변 핵심 키워드

representation boundary, explicit adapter, anti-corruption layer, minimal shared kernel, ownership, change coupling, domain vs wire, framework-neutral, boundary test

### 백지 구현

이 항목은 코딩보다 설계 설명이 적절하다. 다음 표와 변환 경로를 백지에 작성한다.

**구현 목표**

다섯 계약의 공통 라이브러리 책임과 서비스 책임을 분리하고, Java↔JSON↔Avro 변환 지점을 정의한다.

**작성할 인터페이스 또는 산출물**

```text
계약: Money | Odds | BetSlip | Event schema | ErrorCode

각 계약마다 작성:
- 공통 라이브러리가 보장할 것
- 각 서비스가 보장할 것
- Java 표현
- JSON 표현
- Avro 표현 또는 해당 없음
- 변환 adapter 위치
- 경계 테스트
```

**입력과 출력**

- 입력: 위 다섯 계약과 현재 프로젝트에서 확인된 책임 목록
- 출력: 책임 매트릭스와 대표 데이터 흐름 한 개

**반드시 만족해야 할 조건**

- `Money`: wire shape와 안전한 Java 산술 / balance·ledger·payout 정책 분리
- `Odds`: 정규화·표시 변환 / 가격 source·drift 허용 정책 분리
- `BetSlip`: 구조적 자기 일관성 / 승인·위험 확인·wallet saga 분리
- event schema: 직렬화 형식 / topic 설정·publish·consume·idempotency 분리
- `ErrorCode`: 공통 식별자와 HTTP 의미 / logging·retry 실행·사용자 문구 분리
- 공통 라이브러리에 repository, Spring component, Kafka client, DB model을 넣지 않는다.
- 표현 간 변환은 adapter가 명시적으로 수행한다.
- 각 경계에서 round trip 또는 shape 검증 방법을 적는다.

**경계 조건**

- Java 모델에는 helper method가 있지만 JSON에는 노출하면 안 되는 경우
- JSON에는 선택 필드가 있지만 Avro V1은 새 record가 필요한 경우
- Java sealed type을 Avro enum+보조 필드로 표현하는 경우
- 서비스 정책 변경이 공통 artifact 배포를 요구하지 않아야 하는 경우
- 공통 오류와 서비스 내부 오류가 갈리는 경우

**실패 조건**

- 하나의 클래스가 JPA, Jackson, Avro, Spring Web 책임을 동시에 가짐
- 공통 JAR이 서비스 저장소나 메시징 런타임을 포함함
- adapter 없이 "필드 이름이 같으니 동일하다"고 가정함
- 서비스별 정책 변경 때문에 모든 서비스가 공통 라이브러리를 다시 배포해야 함

**필요한 제약**

- 현재 확인된 컴포넌트와 문서만 사용한다.
- 존재가 확인되지 않은 adapter 클래스명은 만들지 않는다.
- 자동 코드 생성 도구를 전제로 답하지 않는다.

### 구현 후 자가 검증

- [ ] 다섯 계약 모두 공통 책임과 서비스 책임이 구분됐다.
- [ ] Java·JSON·Avro의 목적 차이가 드러난다.
- [ ] 최소 한 개의 명시적 변환 흐름을 설명할 수 있다.
- [ ] 공통 라이브러리에 들어가면 안 되는 구성요소를 표시했다.
- [ ] 정책 변경과 wire 변경의 배포 범위를 구분했다.
- [ ] 경계별 테스트 전략이 있다.
- [ ] 확인되지 않은 클래스나 파일을 만들어내지 않았다.

### 구현 후 설명할 것

1. shared kernel을 작게 유지하는 이유와 중복 코드 허용의 trade-off
2. 명시적 adapter가 주는 독립 배포성과 추가 코드 비용
3. 구조적 불변식과 업무 정책을 가르는 판단 기준
4. Java·JSON·Avro 변경이 각각 어떤 소비자에게 영향을 주는지
5. 프레임워크 중립성을 유지하면 얻는 장기적 선택권

### 원본 확인 위치

- Thread 15
- 커밋: `docs(project): document shared protocol`
- 파일: `README.md`
- 파일: `architecture/contract-ownership-and-representation-boundaries.md`
- 문서 내 컴포넌트: `Money`, `Odds`, `BetSlip`, event schema, `ErrorCode`, `ProblemDetail`
- 관련 Thread: 02, 03, 04, 06, 08, 09, 13, 16
