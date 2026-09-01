# CPP00 개발자 기술면접 워크북 — 마스터 인덱스

이 문서는 현재 프로젝트에 축적된 14개 DevThread 문서와 그 안에서 확인되는 커밋 기록만을 기준으로 작성한 전체 선별표다. 파일·함수·클래스 이름은 작업 기록에서 확인된 경우에만 적었다. 동일 역량이 여러 Thread에서 반복된 경우에는 대표 면접 포인트 하나로 통합하고 나머지를 연관 Thread로 연결했다.

## 우선순위 기준

- **S**: 질문과 10~30분 직접 구현 모두 준비해야 하는 핵심 항목
- **A**: 질문 가치가 높고, 핵심 일부를 직접 구현할 가능성이 높은 항목
- **B**: 직접 구현보다는 개념·설계 판단·경계 설명을 준비할 항목
- **C**: 반복 배선, 얇은 CLI, 단순 설정처럼 별도 면접 문제로 만들 가치가 낮은 항목

## 전체 Thread/커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---:|---|---|---|---|---|---|---|
| **S-01** | 02 | `feat(buffer): 깊은 복사와 정규 대입 구현`<br>`feat(buffer): 결합·비교·출력 연산 제공` | `include/cppf/TextBuffer.hpp`<br>`src/TextBuffer.cpp`<br>`TextBuffer`, `operator=`, `operator+=`, `swap`, `at` | 직접 소유 메모리의 값 의미론, Rule of Three, copy-and-swap, 길이 오버플로, 자기 결합 | C++ 면접에서 가장 직접적으로 소유권·복사 독립성·예외 보장을 확인할 수 있다. 단순 문자열 기능보다 불변식과 실패 시 상태가 핵심이다. | 높음 | 높음 | 03, 04, 12, 13, 14 |
| **S-02** | 03 | `feat(format): 다형적 formatter 인터페이스 정의`<br>`test(format): pipeline 복사와 자기 대입 검증` | `include/cppf/Formatter.hpp`<br>`src/Formatter.cpp`<br>`include/cppf/FormatPipeline.hpp`<br>`src/FormatPipeline.cpp`<br>`Formatter::clone`, `FormatPipeline` | 다형 객체 복제, 가상 소멸자, 객체 슬라이싱 회피, 소유형 이기종 컬렉션, 부분 생성 정리 | raw pointer를 소유하는 다형 컬렉션을 안전하게 복사하는 문제는 언어 원리와 lifecycle 판단을 함께 검증한다. | 높음 | 높음 | 02, 04, 07, 12, 13 |
| **S-03** | 04 | `feat(factory): formatter 임시 소유와 pipeline 교체 구현`<br>`fix(factory): 교체 실패에도 기존 파이프라인 보존` | `include/cppf/Factory.hpp`<br>`src/Factory.cpp`<br>`FormatterOwner`<br>`PipelineBuilder::replace`<br>`DefaultFormatterCreator::create` | 후보 상태를 완성한 뒤 교체하는 트랜잭션, 임시 RAII, 예외 형식 보존 | 실제 코드에서 먼저 대상 객체를 비웠다가 실패 시 상태를 잃는 문제가 수정됐다. 강한 예외 보장을 설명하고 짧게 구현하기에 매우 적합하다. | 높음 | 높음 | 03, 12, 13 |
| **S-04** | 10 | `feat(rpn): signed token과 stack 문법 처리`<br>`feat(rpn): overflow 검사 산술 연산 구현` | `include/cppf/RpnEvaluator.hpp`<br>`src/RpnEvaluator.cpp`<br>`RpnEvaluator::evaluate`<br>`parseLong`, `checkedAdd`, `checkedSubtract`, `checkedMultiply`, `checkedDivide` | 스택 기반 RPN 평가, signed 정수 파싱, UB 없는 오버플로 검사, 피연산자 순서 | 자료구조·파싱·정수 경계·실패 분류를 한 문제에서 검증할 수 있고, 20~30분 구현 과제로 축소하기 좋다. | 높음 | 높음 | 05, 08, 11, 14 |
| **S-05** | 11 | `feat(batch): 입력 문법과 원자 교체 구현`<br>`feat(batch): 결과 정렬과 직렬화 제공` | `include/cppf/BatchEngine.hpp`<br>`src/BatchEngine.cpp`<br>`BatchEngine::replace`, `results`, `write`<br>`parseLine`, `isValidName`, `JobResult` | 다단계 입력 처리의 원자 교체, 중복 검출, 오류 전파, 결정적 정렬·출력 | 파싱·계산·중복 검사·정렬 중 어느 단계가 실패해도 기존 결과를 보존해야 한다. 데이터 정합성과 트랜잭션 면접 질문으로 가장 대표적이다. | 높음 | 높음 | 09, 10, 12, 14 |
| **A-01** | 01 | `feat(contact): 고정 크기 연락처 저장 순서 보존`<br>`fix(contact): 할당 실패에도 저장 상태 보존` | `include/cppf/ContactBook.hpp`<br>`src/ContactBook.cpp`<br>`ContactBook::add`, `at`, `size`<br>`contacts_`, `size_`, `next_` | 고정 용량 원형 상태, 물리·논리 인덱스 분리, 덮어쓰기의 강한 예외 보장 | 원형 버퍼 불변식과 가장 오래된 값 교체를 확인하기 좋다. 복사 실패 전에 인덱스를 바꾸지 않는 순서도 면접 가치가 높다. | 높음 | 높음 | 12, 14 |
| **A-02** | 05 | `feat(scalar): scalar 리터럴 문법과 종류 분류` | `src/ScalarLiteral.cpp`<br>`cppf::scalar_detail::parseScalarLiteral`<br>`rejectInvalidBytes`, `validateFiniteGrammar`, `ScalarLiteral` | 수기 리터럴 파서, 문법 우선순위, 특수값, 음의 0, locale·바이트 경계 | 단순 라이브러리 호출이 아니라 허용 문법을 먼저 확정하고 값 변환과 분리했다. 모호한 입력과 경계 테스트를 설명하기 좋다. | 높음 | 중상 | 06, 10, 14 |
| **A-03** | 06 | `feat(scalar): 부동소수점 표현과 원자 출력 구현` | `include/cppf/ScalarConverter.hpp`<br>`src/ScalarConverter.cpp`<br>`ScalarConverter::write`<br>`finiteNumber`, `canProjectFloat` | 수치 투영 가능성, 음의 0, 부동소수점 underflow/overflow, locale 독립 출력, 임시 렌더링 | 호출자 스트림의 locale·flags·precision에 결과가 흔들리지 않도록 내부 버퍼에서 완성한 뒤 기록한다. I/O 경계와 수치 경계를 함께 묻기 좋다. | 높음 | 중상 | 05, 11, 14 |
| **A-04** | 09 | `feat(template): 임의 접근 container batch 추상화 추가`<br>`test(template): iterator·정렬·복사 실패 계약 검증` | `include/cppf/RandomAccessBatch.hpp`<br>`RandomAccessBatch<T, Container>`<br>`equal_ranges` | 템플릿 container 요구 조건, dependent type, iterator 노출, copy-and-swap, 범위 동등성 | `vector`와 `deque`는 대체되지만 `list`는 계약에 맞지 않는다. 문법 암기보다 요구 연산과 일반화 범위를 설명하는 문제로 적합하다. | 높음 | 중상 | 11, 12, 13 |
| **A-05** | 12 | `test(factory): 생성·복제·할당 실패 정리 검증`<br>`test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증`<br>`test(format): 복제 실패 뒤 부분 객체 정리 검증` | `tests/support/FailingNew.cpp`<br>`tests/support/TestFormatter.cpp`<br>`tests/failure/test_factory_failure.cpp`<br>`tests/failure/test_batch_failure.cpp`<br>`tests/failure/test_pipeline_failure.cpp`<br>`tests/failure/test_contact_failure.cpp` | 실패 지점 sweep, live object 계수, 강한 예외 보장 검증, 오류 형식 전파 | 정상 경로만으로는 소유권 누수와 부분 갱신을 검증할 수 없다. 실패를 각 할당·복제 지점에 주입하는 테스트 설계는 프로젝트 밖에서도 일반화된다. | 높음 | 중간 | 01, 02, 03, 04, 11, 14 |
| **B-01** | 01 | `feat(contact): 검증된 연락처 값 객체 구현`<br>`test(contact): 연락처 값 불변식 검증` | `include/cppf/Contact.hpp`<br>`src/Contact.cpp`<br>`Contact`, `validField`, `empty`, `swap` | 유효하지 않은 입력을 빈 값으로 표현하는 값 객체 불변식 | 입력 길이와 printable ASCII 제한은 명확하지만 구현 난도는 낮다. 오류 표현을 예외·상태형·별도 결과형 중 무엇으로 할지 설명 준비가 더 중요하다. | 중간 | 낮음 | 01, 11, 13 |
| **B-02** | 07 | `test(rtti): pointer·reference 식별 경계 검증`<br>`test(rtti): integer에서 runtime kind로의 암시 변환 거부` | `include/cppf/RuntimeType.hpp`<br>`RuntimeBase`, `RuntimeA`, `RuntimeB`, `RuntimeC`<br>`RuntimeInspector::identify`, `create`, `name` | `dynamic_cast`의 pointer/reference 차이, 등록되지 않은 파생형, 가상 소멸 | C++ 런타임 타입 식별 원리는 질문 가치가 있으나, 실제 식별 체인 구현은 짧고 Thread 03의 다형성 문제와 상당 부분 겹친다. | 높음 | 낮음 | 03, 08, 13 |
| **B-03** | 08 | `test(serialization): null과 주소 동일성 검증`<br>`feat(casts): address token CLI mode 추가` | `include/cppf/Serializer.hpp`<br>`Serializer::raw_type`, `serialize`, `deserialize`<br>`Payload` | 주소 토큰은 직렬화가 아니라 borrowed identity라는 점, 객체 lifetime, null·폭 요구 | 소유권 이전이나 데이터 보존을 하지 않는다는 설명이 핵심이다. 직접 구현은 `reinterpret_cast` 왕복에 가까워 별도 고난도 문제로 만들 필요는 낮다. | 높음 | 낮음 | 07, 10, 13, 14 |
| **B-04** | 11 | `feat(batch): 두 container의 정렬 결과 대조` | `src/BatchEngine.cpp`<br>`RandomAccessBatch<JobResult>`<br>`RandomAccessBatch<JobResult, std::deque<JobResult> >`<br>`equal_ranges` | 서로 다른 구현을 실행시켜 결과를 대조하는 런타임 오라클 | 독립 구현 비교의 장점은 있으나 동일 comparator·동일 알고리즘을 공유하면 결함 독립성이 제한된다. 설계 trade-off 설명용이 적절하다. | 중간 | 낮음 | 09, 14 |
| **B-05** | 13 | `test(contact): 공개 계약과 명령행 세션 검증`<br>`test(casts): 타입·주소 변환의 공개 경계 검증` | `tests/compile/*_fail.cpp`<br>`tests/compile/*_headers.cpp`<br>`tests/integration/test_public_contract.cpp`<br>Makefile `test-contract` | 컴파일 성공·실패를 테스트하는 공개 API 계약, const·explicit·private·abstract 경계 | 런타임 테스트로 잡을 수 없는 잘못된 사용법을 컴파일 실패로 고정한다. 구현보다 어떤 사용을 금지할지 API 관점에서 설명하는 것이 중요하다. | 높음 | 낮음 | 01~12, 14 |
| **B-06** | 14 | `build(check): sanitizer와 portable 검사 계층 구성`<br>`test(portability): 지원 LP64 데이터 모델 검증` | `Makefile`의 `check-build`, `check-portable`, `check-platform`, `test-asan`, `test-ubsan`<br>`tests/portability/test_data_model.cpp` | 검증 계층 분리, sanitizer 역할, 플랫폼 전용 계약, 데이터 모델 가정 | 테스트 종류를 한 타깃에 섞지 않고 portable/platform gate로 분리한 판단은 설명 가치가 높다. 다만 면접 백지 구현보다는 설계 토론이 적합하다. | 높음 | 낮음 | 12, 13, 14 |
| **B-07** | 14 | `test(consumer): 저장소 밖 공개 library 연결 검증` | `tests/check_external_consumer.sh`<br>`tests/consumer/external_main.cpp`<br>`tests/check_archive.sh`<br>`tests/check_dependencies.sh`<br>`tests/manifests/*` | 외부 소비자 관점 링크 검증, 정적 archive 구성·심볼·동적 의존성 release gate | 내부 테스트가 통과해도 공개 헤더·아카이브·링크 경계는 깨질 수 있다. 시스템 경계 설명용으로 좋지만 직접 구현 우선순위는 낮다. | 중상 | 낮음 | 13, 14 |
| **C-01** | 01 | `feat(contact): 연락처 명령행 세션 연결` | `apps/ex00_contact_book.cpp` | 명령 문자열 분기와 얇은 CLI 어댑터 | 핵심 상태와 검증은 `Contact`·`ContactBook`에 있고 CLI는 단순 연결이다. 별도 면접 문제로 만들 필요가 낮다. | 낮음 | 낮음 | 01 |
| **C-02** | 02 | `feat(buffer): 문자열 결합 CLI 제공` | `apps/ex01_text_buffer.cpp` | 인자 검증과 출력 연결 | 핵심 값 의미론을 반복하지 않는 얇은 진입점이다. | 낮음 | 낮음 | 02 |
| **C-03** | 03, 04 | `feat(factory): 명세 기반 파이프라인 CLI 제공` | `apps/ex03_pipeline_factory.cpp` | 명세 배열 구성과 예외를 exit code로 변환 | factory·pipeline의 핵심 소유권 및 원자성보다 면접 가치가 낮다. | 낮음 | 낮음 | 03, 04 |
| **C-04** | 06, 07, 08 | `feat(scalar): type boundary CLI의 scalar mode 제공`<br>`feat(casts): address token CLI mode 추가` | `apps/ex04_type_boundary.cpp`<br>`runScalar`, `runRuntime`, `runAddress`, `parseRuntimeKind`, `parsePayloadId` | 모드 분기와 CLI 오류 출력 | `parsePayloadId`의 오버플로 검사는 Thread 10과 연관해 설명할 수 있지만, 전체 CLI 자체는 반복 어댑터다. | 낮음 | 낮음 | 05, 06, 07, 08, 10 |
| **C-05** | 14 | `build(makefile): C++98 정적 라이브러리 빌드 구성` | `Makefile` 기본 compile/archive/dependency 규칙 | 기본 정적 라이브러리 빌드 배선 | C++98 제약과 strict warning 정책은 배경으로 중요하지만, 기본 패턴 규칙 자체는 프로젝트 핵심 문제보다 우선순위가 낮다. | 낮음 | 낮음 | 13, 14 |

## 대표 Thread와 연관 Thread 관계

| 통합 역량 묶음 | 대표 Thread | 연관 Thread | 통합 기준 |
|---|---:|---|---|
| 직접 소유 자원과 강한 예외 보장 | **02** | 03, 04, 12, 13, 14 | 가장 작은 소유 단위인 `TextBuffer`에서 복사·대입·결합·실패 불변식을 먼저 설명하고, 다형 pipeline과 factory로 확장한다. |
| 다형 객체 lifecycle과 런타임 타입 경계 | **03** | 04, 07, 08, 12, 13 | `clone`·가상 소멸·소유 컬렉션을 대표 문제로 두고, RTTI와 주소 토큰은 경계 설명으로 묶는다. |
| 문법·수치 경계·결정적 표현 | **10** | 05, 06, 08, 11, 14 | RPN의 정수 파싱과 오버플로를 대표 직접 구현으로 두고, scalar 문법·투영·출력은 별도 A 문제로 세분화한다. |
| 고정 상태와 트랜잭션 교체 | **11** | 01, 09, 10, 12, 14 | `BatchEngine`의 후보 결과 후 swap을 대표로 삼고, 원형 상태·container 추상화·실패 주입을 연결한다. |
| 계약 검증과 release gate | **12** | 13, 14 | 실패 지점 검증을 코드 수준 대표로 두고, 컴파일 계약·외부 소비자·sanitizer·플랫폼 gate를 설명 수준으로 확장한다. |

## S/A 상세 워크북 연결

| ID | 상세 면접 포인트 | 상세 문서 |
|---|---|---|
| S-01 | 직접 소유 문자열의 값 의미론과 예외 안전 결합 | [`01-ownership-and-exception-safety.md#s-01`](01-ownership-and-exception-safety.md#s-01) |
| S-02 | 다형 객체 clone과 소유형 pipeline 복사 | [`01-ownership-and-exception-safety.md#s-02`](01-ownership-and-exception-safety.md#s-02) |
| S-03 | 명세 기반 pipeline의 원자 교체 | [`01-ownership-and-exception-safety.md#s-03`](01-ownership-and-exception-safety.md#s-03) |
| A-05 | 실패 지점 sweep와 강한 예외 보장 검증 | [`01-ownership-and-exception-safety.md#a-05`](01-ownership-and-exception-safety.md#a-05) |
| A-02 | scalar 리터럴 문법 파서 | [`02-parsing-numeric-and-rendering.md#a-02`](02-parsing-numeric-and-rendering.md#a-02) |
| A-03 | 수치 투영과 결정적 원자 렌더링 | [`02-parsing-numeric-and-rendering.md#a-03`](02-parsing-numeric-and-rendering.md#a-03) |
| S-04 | 오버플로 검사 RPN 평가기 | [`02-parsing-numeric-and-rendering.md#s-04`](02-parsing-numeric-and-rendering.md#s-04) |
| A-01 | 고정 용량 원형 상태와 실패 안전 덮어쓰기 | [`03-state-containers-and-transactions.md#a-01`](03-state-containers-and-transactions.md#a-01) |
| A-04 | 대체 가능한 임의 접근 container 템플릿 | [`03-state-containers-and-transactions.md#a-04`](03-state-containers-and-transactions.md#a-04) |
| S-05 | 배치 입력의 트랜잭션 평가와 결정적 결과 | [`03-state-containers-and-transactions.md#s-05`](03-state-containers-and-transactions.md#s-05) |

## 완전성 검증

| ID | 우선순위 | 처리 상태 | 검증 결과 |
|---|---|---|---|
| S-01 | S | 독립 상세 항목 | `01-ownership-and-exception-safety.md`에 작성됨 |
| S-02 | S | 독립 상세 항목 | `01-ownership-and-exception-safety.md`에 작성됨 |
| S-03 | S | 독립 상세 항목 | `01-ownership-and-exception-safety.md`에 작성됨 |
| S-04 | S | 독립 상세 항목 | `02-parsing-numeric-and-rendering.md`에 작성됨 |
| S-05 | S | 독립 상세 항목 | `03-state-containers-and-transactions.md`에 작성됨 |
| A-01 | A | 독립 상세 항목 | Thread 12의 contact 실패 보장까지 명시적으로 통합됨 |
| A-02 | A | 독립 상세 항목 | Thread 06과의 파싱/표현 경계를 분리해 작성됨 |
| A-03 | A | 독립 상세 항목 | Thread 11의 결정적 출력 원칙을 연관 항목으로 통합함 |
| A-04 | A | 독립 상세 항목 | Thread 11의 vector/deque 대조는 B-04로 분리하고 container 계약만 상세화함 |
| A-05 | A | 독립 상세 항목 | Thread 01·02·03·04·11의 구체 실패 사례를 하나의 실패 주입 문제로 통합함 |

누락된 S/A 항목은 없다. B/C 항목은 상세 워크북 문제를 별도로 만들지 않았다.

## 백지 구현 우선순위

1. **S-04** 오버플로 검사 RPN 평가기 — 스택, 파싱, 산술 경계를 한 번에 연습한다.
2. **S-01** 직접 소유 문자열 — Rule of Three, copy-and-swap, 자기 결합, 길이 오버플로를 구현한다.
3. **S-05** 배치 트랜잭션 교체 — 파싱·중복·계산·정렬 완료 전에는 기존 상태를 건드리지 않는다.
4. **S-02** 다형 clone pipeline — 가상 소멸과 부분 복제 실패 정리를 구현한다.
5. **S-03** factory 기반 원자 교체 — 임시 소유와 후보 객체 후 swap을 구현한다.
6. **A-01** 고정 용량 원형 상태 — 물리 인덱스와 논리 순서를 분리한다.
7. **A-02** scalar 문법 파서 — 허용 문법과 모호성 우선순위를 상태 전이로 구현한다.
8. **A-04** 임의 접근 batch 템플릿 — container 요구 조건과 iterator API를 구현한다.
9. **A-03** 결정적 scalar 렌더링 — locale·stream 상태와 수치 투영 경계를 분리한다.
10. **A-05** 실패 지점 sweep 테스트 — 각 실패 시점에서 상태·live object·오류 형식을 검증한다.

## 설명 연습 우선순위

1. 후보 객체를 완성한 뒤 `swap`해야 강한 예외 보장이 성립하는 이유
2. 다형 객체 컬렉션에서 `clone`과 가상 소멸자가 함께 필요한 이유
3. signed overflow를 연산 후 검사하면 안 되고 연산 전에 경계를 증명해야 하는 이유
4. `BatchEngine::replace`가 기존 결과와 기존 원소 참조까지 실패 시 보존하는 방식
5. `TextBuffer`의 `data_ != 0`, `data_[size_] == '\0'`, `size_` 의미 불변식
6. scalar 파싱과 scalar 표현을 분리한 이유, 음의 0과 NaN/무한대 처리
7. caller stream의 locale·flags·precision을 출력 계약에서 격리한 이유
8. `RandomAccessBatch`가 `vector`·`deque`는 허용하고 `list`는 허용하지 않는 연산 계약
9. 실패 횟수를 먼저 계측한 뒤 1번째부터 마지막 실패 지점까지 sweep하는 테스트 전략
10. 컴파일 실패 테스트, 외부 소비자 링크 테스트, sanitizer, 플랫폼 gate가 서로 대체할 수 없는 이유

## 한 문제로 통합한 Thread 묶음

- **Thread 02 + 03 + 04 + 12**: raw 자원 소유, 다형 복제, 임시 RAII, copy-and-swap, 후보 후 교체를 "소유권과 강한 예외 보장" 묶음으로 통합
- **Thread 05 + 06**: 입력 문법 분류와 출력 투영을 분리하되 "scalar 경계 처리" 연속 문제로 구성
- **Thread 10 + 11**: RPN 계산 실패가 배치 트랜잭션에 전파되는 흐름을 "계산 엔진과 원자 결과 교체" 묶음으로 통합
- **Thread 01 + 12**: 원형 저장소의 정상 덮어쓰기와 할당 실패 시 상태 보존을 하나의 문제로 통합
- **Thread 09 + 11 + 14**: container 대체성, 결과 대조, property/release 검증은 대표 구현을 `RandomAccessBatch`와 `BatchEngine`에 두고 검증 전략으로 연결
