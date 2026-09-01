# 소유권과 예외 안전성 면접 워크북

이 문서는 직접 소유 메모리, 다형 객체 lifecycle, 원자 교체, 실패 주입 검증을 한 흐름으로 묶는다. 각 백지 구현은 원본 코드의 복사가 아니라 동일한 요구사항과 불변식을 만족하는 축소 문제다.

<a id="s-01"></a>
## [Thread 02 / `feat(buffer): 깊은 복사와 정규 대입 구현`, `feat(buffer): 결합·비교·출력 연산 제공`] 직접 소유 문자열의 값 의미론과 예외 안전 결합

### 면접 질문

`TextBuffer`는 `char*`를 직접 소유합니다. 복사 생성자, 대입 연산자, 소멸자를 어떤 불변식 아래 구현해야 하며, 대입과 문자열 결합에서 할당이 실패해도 기존 객체를 보존하려면 연산 순서를 어떻게 잡아야 합니까?

꼬리 질문:

- `data_`를 빈 문자열에서도 null로 두지 않고 종료 문자 한 바이트를 소유하게 하면 어떤 장점이 있습니까?
- 자기 대입을 별도 분기로 처리하지 않아도 copy-and-swap이 안전한 이유는 무엇입니까?
- `buffer += buffer`가 안전하려면 기존 저장소를 언제 해제해야 합니까?
- `size_ + other.size_ + 1` 계산 자체에서 오버플로가 발생하지 않도록 무엇을 먼저 확인해야 합니까?
- 복사 생성 중 `new[]`가 실패한 경우와 대입 중 실패한 경우, 원본·대상 객체의 상태는 각각 어떻게 되어야 합니까?

### 30초 모범 답변

`TextBuffer`의 핵심 불변식은 저장소가 항상 유효하고, `size_`는 종료 문자를 제외한 길이이며, `data_[size_]`가 항상 `\0`이라는 점입니다. 복사는 새 저장소를 먼저 완성해 독립 소유권을 만들고, 대입은 복사본을 만든 뒤 `swap`하면 복사 실패 시 대상이 변하지 않아 강한 예외 보장을 얻습니다. 결합도 길이 오버플로를 먼저 검사하고 새 버퍼에 양쪽 내용을 모두 구성한 뒤 기존 저장소와 교체해야 자기 결합과 할당 실패가 안전합니다. 비용은 대입 시 임시 객체와 추가 할당이 생긴다는 점이지만, 구현 단순성과 실패 안전성을 얻습니다.

### 답변 핵심 키워드

Rule of Three, 단일 소유권, 깊은 복사, non-null empty representation, 종료 문자 불변식, copy-and-swap, strong exception guarantee, self-assignment, self-concatenation, 연산 전 오버플로 검사

### 백지 구현

#### 구현 목표

C++98에서 `std::string`을 내부 저장소로 사용하지 않고, null 종료 문자열을 직접 소유하는 `OwnedText`를 구현한다. 복사와 대입은 값 의미론을 제공하고, `append`는 할당 실패 시 기존 값을 보존해야 한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class OwnedText
{
public:
    OwnedText();
    explicit OwnedText(const char *text);
    OwnedText(const OwnedText &other);
    ~OwnedText();

    OwnedText &operator=(const OwnedText &other);
    OwnedText &append(const OwnedText &other);

    std::size_t size() const;
    bool empty() const;
    const char *c_str() const;
    char &at(std::size_t index);
    const char &at(std::size_t index) const;
    void swap(OwnedText &other) throw();

private:
    char *data_;
    std::size_t size_;
};

// 직접 구현
```

#### 입력과 출력

- 기본 생성자는 빈 문자열을 만든다.
- `const char*`가 null이면 빈 문자열로 취급한다.
- `append`는 현재 값 뒤에 `other`를 붙인 뒤 자기 자신을 반환한다.
- `c_str()`은 항상 null 종료된 읽기 전용 포인터를 반환한다.
- `at()`은 유효한 문자 위치를 참조로 반환한다.

#### 반드시 만족해야 할 조건

- 모든 정상 객체에서 `data_ != 0`이다.
- `size_`는 종료 문자를 포함하지 않는다.
- `data_[size_] == '\0'`이다.
- 복사본을 수정해도 원본이 변하지 않는다.
- 대입과 `append`는 할당 실패 시 호출 전 값을 보존한다.
- `value = value`와 `value.append(value)`가 모두 안전하다.
- 소멸 시 자신이 소유한 배열을 정확히 한 번 해제한다.

#### 경계 조건

- 빈 문자열끼리의 복사·대입·결합
- 빈 값과 비어 있지 않은 값의 양방향 결합
- 자기 대입과 자기 결합
- `at(0)` on empty, `at(size())`
- 최대 `std::size_t`에 가까운 길이 계산

#### 실패 조건

- 범위를 벗어난 `at()`은 범위 오류를 보고한다.
- 필요한 총 바이트 수를 표현할 수 없으면 길이 오류를 보고한다.
- 메모리 할당 실패는 그대로 전파하되 기존 객체를 바꾸지 않는다.

#### 필요한 제약

- C++98만 사용한다.
- 내부에 `std::string`, `std::vector<char>`를 사용하지 않는다.
- 저장소는 `new[]`와 `delete[]`로 관리한다.
- 비교·스트림 연산자는 구현 범위에서 제외한다.

### 구현 후 자가 검증

- [ ] 기본 객체의 `size()`는 0이고 `c_str()`은 빈 C 문자열이다.
- [ ] null 입력 생성자가 정상적인 빈 객체를 만든다.
- [ ] 복사 생성 후 한쪽 문자를 바꿔도 다른 쪽은 그대로다.
- [ ] 대입 결과가 올바르고 대입 연산자가 왼쪽 객체를 반환한다.
- [ ] 자기 대입 전후 값과 불변식이 같다.
- [ ] 빈 값 결합, 일반 결합, 자기 결합 결과가 정확하다.
- [ ] 길이 계산에서 `+ 1`까지 포함해 오버플로를 검사한다.
- [ ] 새 저장소 구성 중 예외가 나면 기존 포인터·길이가 그대로다.
- [ ] 범위 오류 뒤에도 객체를 계속 사용할 수 있다.
- [ ] 생성·복사·대입·소멸 반복 후 이중 해제나 누수가 없다.
- [ ] 복사와 결합의 시간 복잡도가 입력 길이에 선형임을 설명할 수 있다.

### 구현 후 설명할 것

1. 빈 문자열도 한 바이트 저장소를 갖게 한 이유와 null 저장 표현과의 trade-off
2. copy-and-swap이 자기 대입과 강한 예외 보장을 동시에 단순화하는 방식
3. `append`에서 기존 저장소를 마지막에 해제해야 자기 결합이 안전한 이유
4. 길이 오버플로를 연산 후가 아니라 연산 전에 검사한 식
5. 대입마다 추가 할당이 발생하는 방식과 재사용 최적화 사이의 trade-off

### 원본 확인 위치

- Thread 02
- 커밋: `feat(buffer): 깊은 복사와 정규 대입 구현`
- 커밋: `feat(buffer): 결합·비교·출력 연산 제공`
- 파일: `include/cppf/TextBuffer.hpp`, `src/TextBuffer.cpp`
- 클래스·함수: `cppf::TextBuffer`, 복사 생성자, `operator=`, `operator+=`, `swap`, `at`, `c_str`
- 관련 Thread: 12, 13, 14

---

<a id="s-02"></a>
## [Thread 03 / `feat(format): 다형적 formatter 인터페이스 정의`, `test(format): pipeline 복사와 자기 대입 검증`] 다형 객체 clone과 소유형 pipeline 복사

### 면접 질문

서로 다른 파생 `Formatter` 객체를 하나의 `FormatPipeline`이 소유하고, pipeline 자체도 값처럼 복사되어야 합니다. 객체 슬라이싱 없이 동적 타입을 보존하면서 복사·대입·소멸을 안전하게 구현하려면 인터페이스와 소유 규칙을 어떻게 설계하겠습니까?

꼬리 질문:

- 기반 클래스 소멸자가 가상이어야 하는 이유는 무엇입니까?
- `clone()`이 원시 포인터를 반환하는 설계에서 누가 언제 소유권을 취합니까?
- 복사 생성 중 네 번째 clone이 실패하면 앞서 성공한 세 객체는 누가 정리해야 합니까?
- `append(const Formatter&)`가 원본을 참조만 보관하지 않고 clone하는 이유는 무엇입니까?
- 용량 검사를 clone보다 먼저 해야 하는 이유는 무엇입니까?
- C++11 이후라면 어떤 소유 타입으로 바꾸겠습니까?

### 30초 모범 답변

이기종 다형 객체를 값처럼 복사하려면 기반 클래스에 가상 소멸자와 `clone()` 계약이 필요합니다. pipeline은 clone 결과를 독점 소유하고, 복사 생성자는 각 요소를 순서대로 clone하되 중간 실패 시 이미 만든 요소를 모두 정리해야 합니다. 대입은 완성된 복사본과 `swap`하면 대상의 기존 동작을 실패 시 보존할 수 있습니다. `append`도 외부 객체의 lifetime에 의존하지 않도록 clone하며, 용량 초과는 clone 전에 검사해 불필요한 생성과 소유권 혼란을 막습니다.

### 답변 핵심 키워드

가상 소멸자, pure virtual, virtual constructor idiom, clone, 객체 슬라이싱, 독점 소유, heterogeneous collection, 부분 생성 정리, copy constructor cleanup, copy-and-swap, lifetime 분리

### 백지 구현

#### 구현 목표

최대 8개의 다형 변환 단계를 소유하는 pipeline을 구현한다. 각 단계는 입력 문자열을 새 문자열로 변환하며, pipeline 복사본은 원본과 독립적인 파생 객체들을 소유해야 한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class Transform
{
public:
    virtual ~Transform();
    virtual Transform *clone() const = 0;
    virtual std::string apply(const std::string &input) const = 0;
};

class TransformPipeline
{
public:
    enum { max_steps = 8 };

    TransformPipeline();
    TransformPipeline(const TransformPipeline &other);
    ~TransformPipeline();

    TransformPipeline &operator=(const TransformPipeline &other);
    void append(const Transform &step);
    std::string apply(const std::string &input) const;
    std::size_t size() const;
    void swap(TransformPipeline &other) throw();

private:
    Transform *steps_[max_steps];
    std::size_t size_;
};

// 직접 구현
```

#### 입력과 출력

- `append`는 전달받은 변환 객체의 동적 타입을 복제해 pipeline이 소유한다.
- `apply`는 등록 순서대로 변환을 적용한 최종 문자열을 반환한다.
- 복사 생성·대입 결과는 원본과 같은 동작을 하지만 내부 객체는 공유하지 않는다.

#### 반드시 만족해야 할 조건

- `steps_[0..size_)`는 각각 유효한 소유 포인터다.
- `size_ <= max_steps`다.
- pipeline 소멸 시 소유한 모든 단계가 정확히 한 번 소멸한다.
- 기반 포인터로 삭제해도 파생 소멸자가 호출된다.
- 복사 생성 중 실패하면 생성 중인 객체가 누수 없이 폐기되고 원본은 변하지 않는다.
- 대입 중 실패하면 대상 pipeline의 기존 크기와 동작이 보존된다.
- `pipeline = pipeline`이 안전하다.

#### 경계 조건

- 빈 pipeline 적용
- 정확히 최대 용량까지 append
- 최대 용량을 넘기는 append
- clone이 첫 번째, 중간, 마지막 단계에서 실패
- 파생 객체가 pipeline보다 먼저 소멸하는 경우
- 복사 후 원본에 단계를 추가하는 경우

#### 실패 조건

- 최대 용량 초과는 원본 객체를 clone하기 전에 보고한다.
- `clone()`이 던진 예외는 의미를 잃지 않고 전파한다.
- 문자열 변환 중 예외가 발생해도 pipeline의 소유 구조는 유지된다.

#### 필요한 제약

- C++98만 사용한다.
- 스마트 포인터와 표준 컨테이너를 pipeline 내부 저장소로 사용하지 않는다.
- `clone()`은 새로 할당된 객체의 포인터를 반환한다.
- 복사 시 단계 공유나 참조 카운팅을 사용하지 않는다.

### 구현 후 자가 검증

- [ ] 기반 클래스의 소멸자가 가상이다.
- [ ] 서로 다른 파생 단계가 등록 순서대로 호출된다.
- [ ] 외부 단계 객체가 사라진 뒤에도 pipeline이 정상 동작한다.
- [ ] 복사본과 원본의 각 clone 주소가 다르고 동작은 같다.
- [ ] 복사 후 한쪽 구조를 변경해도 다른 쪽 크기와 동작이 변하지 않는다.
- [ ] 첫 clone 실패와 중간 clone 실패에서 live object 수가 기준으로 돌아온다.
- [ ] 대입 실패 시 대상의 기존 단계와 출력이 유지된다.
- [ ] 최대 용량 초과 시 clone 호출 횟수가 증가하지 않는다.
- [ ] 빈 pipeline은 입력을 그대로 반환한다.
- [ ] 소멸 시 파생 소멸자까지 모두 호출된다.

### 구현 후 설명할 것

1. `clone()`이 객체 슬라이싱을 피하면서 값 의미론을 제공하는 방식
2. 복사 생성자 자체는 완성된 객체가 아니므로 부분 clone을 직접 정리해야 하는 이유
3. 대입에서는 copy-and-swap을 사용할 수 있지만 복사 생성에서는 별도 정리가 필요한 이유
4. `append`가 참조 보관이 아니라 clone 소유를 선택한 lifecycle trade-off
5. C++11 이상에서 `std::unique_ptr<Transform>`과 `std::vector`로 단순화할 부분

### 원본 확인 위치

- Thread 03
- 커밋: `feat(format): 다형적 formatter 인터페이스 정의`
- 커밋: `test(format): pipeline 복사와 자기 대입 검증`
- 파일: `include/cppf/Formatter.hpp`, `src/Formatter.cpp`
- 파일: `include/cppf/FormatPipeline.hpp`, `src/FormatPipeline.cpp`
- 클래스·함수: `cppf::Formatter`, `Formatter::clone`, `cppf::FormatPipeline`, `append`, 복사 생성자, `operator=`, `apply`, `swap`
- 관련 Thread: 02, 04, 07, 12, 13

---

<a id="s-03"></a>
## [Thread 04 / `feat(factory): formatter 임시 소유와 pipeline 교체 구현`, `fix(factory): 교체 실패에도 기존 파이프라인 보존`] 명세 기반 pipeline의 원자 교체

### 면접 질문

문자열 명세 배열을 읽어 formatter들을 생성하고 기존 pipeline 전체를 교체하는 `PipelineBuilder::replace`가 있습니다. 중간 명세가 잘못됐거나 생성·clone·할당이 실패해도 기존 pipeline을 그대로 보존하려면 어떤 객체를 언제 변경해야 합니까?

꼬리 질문:

- 기존 pipeline을 먼저 비운 뒤 새 단계를 append하는 구현의 문제는 무엇입니까?
- creator가 반환한 raw pointer와 `append`가 내부에서 만드는 clone 사이의 두 소유 단계를 어떻게 정리합니까?
- 후보 pipeline이 완성되기 전에 예외가 나면 어떤 소멸자가 무엇을 정리합니까?
- `specifications == 0 && count == 0`과 `specifications == 0 && count != 0`을 왜 다르게 처리합니까?
- 생성 실패, 알 수 없는 명세, clone 실패의 예외 형식을 하나로 바꾸지 않고 그대로 전파하는 장점은 무엇입니까?
- creator가 `std::unique_ptr`를 반환할 수 있다면 인터페이스를 어떻게 바꾸겠습니까?

### 30초 모범 답변

교체 연산은 기존 대상에 직접 쓰지 않고 지역 `candidate`를 완성한 뒤 마지막에 `swap`해야 합니다. creator가 반환한 포인터는 append가 clone을 끝낼 때까지만 임시 RAII 객체가 소유하고, append 후에는 임시 원본을 지워도 candidate가 독립 clone을 갖습니다. 명세 검증·생성·clone 중 어디서 실패해도 지역 객체의 소멸자가 부분 결과를 정리하고 target은 손대지 않았으므로 강한 예외 보장이 성립합니다. 마지막 swap은 예외를 던지지 않아야 commit 지점으로 사용할 수 있습니다.

### 답변 핵심 키워드

transactional update, candidate then commit, no-throw swap, local RAII, temporary ownership, creator/clone ownership boundary, strong guarantee, exception transparency, validation before mutation

### 백지 구현

#### 구현 목표

`Creator`가 문자열 명세로 다형 `Transform` 객체를 동적 생성하고, `replacePipeline`이 명세 전체를 성공적으로 처리한 경우에만 대상 pipeline을 교체하도록 구현한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class Creator
{
public:
    virtual ~Creator();
    virtual Transform *create(const std::string &specification) const = 0;
};

void replacePipeline(TransformPipeline &target,
                     const Creator &creator,
                     const std::string *specifications,
                     std::size_t count);

// 직접 구현
```

#### 입력과 출력

- 입력은 대상 pipeline, creator, 명세 배열, 원소 수다.
- 모든 명세가 성공하면 target은 같은 순서의 새 단계들로 완전히 교체된다.
- `count == 0`이면 target은 빈 pipeline으로 교체된다.
- 반환값은 없고 오류는 예외로 전달한다.

#### 반드시 만족해야 할 조건

- 명세 전체가 성공하기 전까지 target을 변경하지 않는다.
- creator가 반환한 각 포인터는 성공·실패와 관계없이 정확히 한 번 해제된다.
- pipeline이 소유하는 객체는 creator의 원본 포인터와 독립적인 clone이다.
- 성공 시 이전 target 자원은 교체 과정 뒤 자동으로 정리된다.
- 실패 시 target의 크기와 동작이 호출 전과 같다.
- 입력 명세 순서가 pipeline 실행 순서로 보존된다.

#### 경계 조건

- null 배열과 0개 명세
- null 배열과 1개 이상 명세
- 정확히 최대 단계 수
- 최대 단계 수 초과
- 첫 명세 생성 실패
- 중간 명세 생성 실패
- 생성은 성공했지만 append의 clone이 실패
- 기존 target이 비어 있는 경우와 이미 단계를 가진 경우

#### 실패 조건

- 포인터/개수 조합이 잘못되거나 최대 단계를 넘으면 명세 오류를 보고한다.
- creator가 던진 오류와 clone이 던진 오류는 target을 보존한 채 전파한다.
- 임시 소유 객체를 만들기 전에 null 포인터 역참조가 일어나면 안 된다.

#### 필요한 제약

- C++98만 사용한다.
- creator 인터페이스는 owning raw pointer를 반환한다.
- `TransformPipeline::append`는 전달 객체를 clone하며 포인터 자체를 인수하지 않는다.
- `TransformPipeline::swap`은 예외를 던지지 않는다고 가정한다.

### 구현 후 자가 검증

- [ ] 정상 명세 여러 개가 원래 순서로 적용된다.
- [ ] 0개 명세가 target을 빈 상태로 교체한다.
- [ ] 잘못된 null/개수 조합은 target을 보존한다.
- [ ] 첫 생성 실패와 중간 생성 실패에서 target이 보존된다.
- [ ] clone 실패에서 생성된 임시 원본과 candidate clone이 모두 정리된다.
- [ ] 실패 예외의 실제 형식이 다른 오류로 뭉개지지 않는다.
- [ ] 성공 후 이전 target의 객체가 누수 없이 정리된다.
- [ ] commit 이전에는 target의 관찰 가능한 상태를 바꾸는 문장이 없다.
- [ ] commit에 사용하는 연산이 예외를 던지지 않는다.
- [ ] creator 호출 횟수와 append/clone 호출 횟수가 실패 지점과 일치한다.

### 구현 후 설명할 것

1. 대상 선삭제 방식이 basic guarantee조차 약화할 수 있는 실패 시나리오
2. candidate의 lifetime이 rollback을 자동화하는 이유
3. creator 원본과 pipeline clone이 잠시 동시에 존재하는 비용과 소유권 명확성의 trade-off
4. 마지막 `swap`을 트랜잭션 commit으로 볼 수 있는 조건
5. 오류를 변환하지 않고 전파할 때 호출자가 얻는 진단 정보

### 원본 확인 위치

- Thread 04
- 커밋: `feat(factory): formatter 임시 소유와 pipeline 교체 구현`
- 커밋: `fix(factory): 교체 실패에도 기존 파이프라인 보존`
- 파일: `include/cppf/Factory.hpp`, `src/Factory.cpp`
- 클래스·함수: `FormatterOwner`, `cppf::FormatterCreator`, `cppf::DefaultFormatterCreator::create`, `cppf::PipelineBuilder::replace`
- 관련 Thread: 03, 12, 13

---

<a id="a-05"></a>
## [Thread 12 / `test(factory): 생성·복제·할당 실패 정리 검증`, `test(format): 복제 실패 뒤 부분 객체 정리 검증`] 실패 지점 sweep와 강한 예외 보장 검증

### 면접 질문

코드가 copy-and-swap이나 후보 객체를 사용한다고 해서 강한 예외 보장이 자동으로 입증되지는 않습니다. 여러 번의 할당·복제가 일어나는 연산에서 "어느 단계가 실패해도 누수와 부분 갱신이 없다"는 것을 테스트로 어떻게 검증하겠습니까?

꼬리 질문:

- 성공 경로에서 먼저 총 할당 또는 clone 횟수를 계측하는 이유는 무엇입니까?
- 1번째 실패부터 마지막 실패까지 모두 주입해야 하는 이유는 무엇입니까?
- 단순히 예외가 발생했다는 사실 외에 어떤 상태를 기록해야 합니까?
- live object 수와 destroyed count는 각각 어떤 종류의 결함을 찾습니까?
- 실패 주입을 해제하는 코드 자체가 예외 때문에 건너뛰지 않도록 어떻게 구성하겠습니까?
- 전역 `operator new` 실패 주입은 어떤 테스트 격리 문제를 만들 수 있습니까?

### 30초 모범 답변

먼저 동일 입력의 성공 실행에서 할당이나 clone 시도 횟수를 측정하고, 그다음 1번부터 마지막 시도까지 각 지점을 하나씩 실패시킵니다. 매 반복에서 예상 예외 형식, 정확한 실패 시도 번호, live object 기준값, 대상 객체의 이전 출력과 크기, 이후 재사용 가능성을 함께 확인합니다. 실패 주입은 반드시 해제하고 각 반복을 새 fixture에서 시작해야 서로 영향을 주지 않습니다. 이 방식은 특정 한 지점만 고른 테스트보다 부분 생성 누수와 commit 시점 오류를 체계적으로 찾습니다.

### 답변 핵심 키워드

fault injection, failure sweep, deterministic failure point, baseline, live object count, allocation count, exception type preservation, rollback invariant, fixture isolation, post-failure usability

### 백지 구현

#### 구현 목표

복사할 때마다 카운터를 증가시키고 지정된 시도에서 예외를 던지는 `ThrowingStep`을 이용해, `TransformPipeline` 복사 생성과 대입의 모든 clone 실패 지점을 검증하는 테스트 함수를 작성한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class CloneFailure
{
};

class ThrowingStep : public Transform
{
public:
    explicit ThrowingStep(const std::string &prefix);
    virtual ~ThrowingStep();
    virtual Transform *clone() const;
    virtual std::string apply(const std::string &input) const;

    static void failCloneOn(std::size_t attempt);
    static void disableFailure();
    static void resetCounters();
    static std::size_t cloneAttempts();
    static int liveCount();
};

void verifyCopyFailureSweep(const TransformPipeline &source);
void verifyAssignmentFailureSweep(const TransformPipeline &source,
                                  const TransformPipeline &original_target);

// 테스트 본문을 직접 구현
```

#### 입력과 출력

- 입력은 여러 단계를 가진 source와 기존 내용을 가진 target fixture다.
- 출력값은 없으며, 테스트 assertion으로 결과를 검증한다.
- 각 실패 시도마다 독립된 복사 또는 target을 사용한다.

#### 반드시 만족해야 할 조건

- 성공 실행에서 실제 clone 횟수를 먼저 얻는다.
- 실패 지점 1부터 성공 실행의 마지막 clone까지 빠짐없이 순회한다.
- 설정한 지점에서 정확히 `CloneFailure`가 발생한다.
- 예상하지 않은 예외는 별도로 실패 처리한다.
- 반복 전후 live object 수가 기준값과 같다.
- source는 모든 실패 뒤에도 같은 크기와 동작을 유지한다.
- 대입 대상은 모든 실패 뒤에도 원래 크기와 동작을 유지한다.
- 실패 주입을 끈 뒤 동일 객체를 다시 정상 사용할 수 있다.

#### 경계 조건

- 첫 clone 실패
- 마지막 clone 실패
- source가 한 단계뿐인 경우
- target이 빈 경우와 기존 단계가 있는 경우
- 자기 대입 fixture
- 성공 경로의 clone 횟수가 0인 잘못된 fixture

#### 실패 조건

- 설정한 실패 지점까지 도달하지 않으면 테스트 실패다.
- live count가 기준보다 크거나 작으면 누수 또는 이중 소멸 가능성으로 처리한다.
- source나 target의 관찰 가능한 결과가 달라지면 rollback 실패다.
- 실패 주입 상태가 다음 반복에 남아 있으면 테스트 격리 실패다.

#### 필요한 제약

- production 구현을 테스트 전용 분기로 수정하지 않는다.
- 임의 sleep이나 비결정적 스케줄에 의존하지 않는다.
- 포인터 주소만 비교하지 말고 크기와 동작도 함께 검증한다.
- 테스트 종료 시 모든 static failure 설정을 초기화한다.

### 구현 후 자가 검증

- [ ] 성공 경로의 clone 횟수를 실제로 측정했다.
- [ ] 첫 번째부터 마지막 clone까지 모든 실패 인덱스를 실행한다.
- [ ] 각 반복 fixture가 이전 반복의 상태를 공유하지 않는다.
- [ ] 예상 예외와 예상하지 않은 예외를 구분한다.
- [ ] clone 시도 횟수가 설정한 실패 번호와 같다.
- [ ] source의 크기와 출력이 실패 전과 같다.
- [ ] 대입 target의 크기와 출력이 실패 전과 같다.
- [ ] live object 수가 반복 전 기준으로 돌아온다.
- [ ] 실패 후 정상 복사·대입이 다시 성공한다.
- [ ] 실패 주입 해제가 테스트 실패 경로에서도 보장된다.

### 구현 후 설명할 것

1. 성공 횟수 계측 후 전 지점 sweep가 경로 누락을 줄이는 방식
2. "예외가 났다"와 "강한 예외 보장이 지켜졌다" 사이에 필요한 추가 assertion
3. live object 계수만으로는 상태 변조를 찾을 수 없어 출력·크기도 확인해야 하는 이유
4. 전역 할당 실패 주입과 타입별 clone 실패 주입의 범위·격리 trade-off
5. Thread 01·02·03·04·11에 같은 검증 패턴을 재사용할 수 있는 이유

### 원본 확인 위치

- Thread 12
- 커밋: `test(factory): 생성·복제·할당 실패 정리 검증`
- 커밋: `test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증`
- 커밋: `test(format): 복제 실패 뒤 부분 객체 정리 검증`
- 파일: `tests/support/FailingNew.cpp`, `tests/support/TestFormatter.cpp`
- 파일: `tests/failure/test_factory_failure.cpp`, `tests/failure/test_batch_failure.cpp`, `tests/failure/test_pipeline_failure.cpp`, `tests/failure/test_contact_failure.cpp`
- 관련 Thread: 01, 02, 03, 04, 11, 14
