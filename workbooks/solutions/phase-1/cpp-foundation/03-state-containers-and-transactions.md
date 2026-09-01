# 상태·컨테이너·트랜잭션 면접 워크북

이 문서는 제한된 저장 공간의 불변식, 템플릿 container 계약, 여러 단계 처리 결과의 원자 교체를 다룬다.

<a id="a-01"></a>
## [Thread 01 / `feat(contact): 고정 크기 연락처 저장 순서 보존`, `fix(contact): 할당 실패에도 저장 상태 보존`] 고정 용량 원형 상태와 실패 안전 덮어쓰기

### 면접 질문

용량 8인 `ContactBook`은 새 값을 계속 추가하되 가득 차면 가장 오래된 값을 교체하고, `at(0)`부터는 항상 가장 오래된 값에서 가장 최신 값 순서로 보여야 합니다. `size_`, `next_`, 물리 배열 인덱스의 불변식을 정의하고, 교체할 값의 복사가 실패해도 기존 순서와 내용이 보존되도록 `add`를 어떻게 구성하겠습니까?

꼬리 질문:

- 저장소가 아직 가득 차지 않았을 때와 가득 찼을 때 논리적 첫 원소의 물리 인덱스는 각각 무엇입니까?
  - 모범답변: 프로젝트 구현에서 `size_ < capacity`이면 값들이 앞에서부터 채워졌으므로 첫 인덱스는 0입니다. 가득 차면 `next_`가 다음 교체 위치이자 현재 가장 오래된 원소의 물리 인덱스입니다.
- `next_`는 "다음 빈 칸"과 "가장 오래된 원소" 중 언제 어떤 의미를 갖습니까?
  - 모범답변: 공통 불변식은 항상 "다음 성공 push가 쓸 위치"입니다. 가득 차기 전에는 그 위치가 빈 칸이고, 가득 찬 뒤에는 빈 칸이 없으므로 그 위치의 가장 오래된 원소를 교체합니다.
- 배열 슬롯에 바로 대입한 뒤 복사가 실패하면 어떤 부분 상태가 남을 수 있습니까?
  - 모범답변: `T::operator=`가 대상 내부의 일부 필드를 먼저 바꾼 뒤 던질 수 있어 해당 슬롯이 옛 값도 새 값도 아닌 상태가 될 수 있습니다. 메타데이터가 그대로여도 논리 원소 내용은 이미 손상됩니다.
- 새 값을 지역 객체에 먼저 복사한 뒤 슬롯과 swap하는 방식이 왜 안전합니까?
  - 모범답변: 실패 가능한 복사는 저장소 밖의 지역 replacement에서 끝냅니다. 복사가 성공한 뒤 no-throw swap으로 슬롯을 한 번에 교체하면, 실패 시 저장소를 전혀 건드리지 않고 성공 시 옛 값은 지역 객체 소멸로 정리됩니다.
- 값 교체가 성공하기 전에 `next_`나 `size_`를 바꾸면 안 되는 이유는 무엇입니까?
  - 모범답변: 복사 실패 후 메타데이터만 전진하면 논리 첫 위치와 실제 저장 내용이 어긋나 순서 불변식이 깨집니다. 슬롯 교체가 성공한 뒤 두 메타데이터를 commit해야 값과 인덱스가 같은 상태 전이를 나타냅니다.
- 고정 배열 방식과 동적 `std::deque` 방식의 trade-off는 무엇입니까?
  - 모범답변: 고정 배열은 용량과 메모리 사용이 예측 가능하고 push·조회가 O(1)이지만 `N`개의 `T`를 미리 생성해야 하고 용량을 바꿀 수 없습니다. deque는 크기 조절과 양끝 연산이 유연하지만 동적 할당과 구현별 블록 관리 비용이 있습니다.

### 30초 모범 답변

불변식은 `0 <= size_ <= capacity`, `0 <= next_ < capacity`이고, 가득 차지 않았을 때 논리 첫 원소는 0, 가득 찼을 때는 `next_`가 가장 오래된 원소이자 다음 교체 위치입니다. 논리 인덱스 `i`는 `(first + i) % capacity`로 물리 인덱스에 매핑합니다. 추가 시에는 입력을 지역 replacement에 먼저 완전히 복사하고, 성공한 뒤 대상 슬롯과 no-throw swap한 다음에만 `next_`와 `size_`를 갱신합니다. 그러면 복사 실패 전에는 저장 배열과 메타데이터를 건드리지 않아 내용과 순서가 모두 보존됩니다.

### 답변 핵심 키워드

bounded state, ring buffer, logical vs physical index, oldest element, next write position, modulo mapping, invariant, prepare before mutate, no-throw swap, commit order, strong exception guarantee

### 백지 구현

#### 구현 목표

고정 용량 `N`의 최근 값 저장소를 구현한다. 최대 용량을 넘으면 가장 오래된 값을 교체하고, 읽기는 오래된 값부터 최신 값 순서로 제공한다. 값 복사가 실패하면 저장소 전체가 호출 전 상태를 유지해야 한다.

#### 인터페이스 또는 함수 시그니처

```cpp
template <typename T, std::size_t N>
class FixedHistory
{
public:
    FixedHistory();

    void push(const T &value);
    std::size_t size() const;
    const T &at(std::size_t logical_index) const;

private:
    T values_[N];
    std::size_t size_;
    std::size_t next_;
};

template <typename T, std::size_t N>
FixedHistory<T, N>::FixedHistory() : values_(), size_(0), next_(0)
{
}

template <typename T, std::size_t N>
void FixedHistory<T, N>::push(const T &value)
{
    T replacement(value); // 실패 가능한 복사는 상태 밖에서 끝낸다.

    // T의 member swap은 예외를 던지지 않는다는 지원 계약이다.
    values_[next_].swap(replacement);
    next_ = (next_ + 1) % N;
    if (size_ < N)
        ++size_;
}

template <typename T, std::size_t N>
std::size_t FixedHistory<T, N>::size() const
{
    return size_;
}

template <typename T, std::size_t N>
const T &FixedHistory<T, N>::at(std::size_t logical_index) const
{
    if (logical_index >= size_)
        throw std::out_of_range("history index");

    const std::size_t first = size_ == N ? next_ : 0;
    return values_[(first + logical_index) % N];
}
```

#### 입력과 출력

- `push`는 새 값을 가장 최신 값으로 추가한다.
- 저장소가 가득 차면 가장 오래된 값 하나를 버린다.
- `at(0)`은 현재 가장 오래된 값, `at(size()-1)`은 가장 최신 값을 반환한다.
- 반환 참조의 lifetime은 다음 성공적인 교체나 저장소 소멸 전까지다.

#### 반드시 만족해야 할 조건

- `N > 0`을 전제로 한다.
- 항상 `size_ <= N`, `next_ < N`이다.
- 가득 차지 않았을 때 기존 논리 순서가 배열의 앞쪽과 일치한다.
- 가득 찼을 때 `next_`는 가장 오래된 값의 위치이자 다음 교체 위치다.
- 성공한 push 뒤 `next_`는 정확히 한 칸 순환한다.
- `size_`는 N에 도달한 뒤 더 증가하지 않는다.
- `T` 복사가 실패하면 `values_`, `size_`, `next_`가 모두 호출 전과 같다.
- 슬롯 교체에 사용하는 swap은 예외를 던지지 않는다는 계약을 명시한다.

#### 경계 조건

- 첫 값 추가
- N-1개에서 N개가 되는 추가
- 처음으로 N개를 넘기는 추가
- 여러 바퀴 이상 순환하는 추가
- `at(0)`, `at(size()-1)`, `at(size())`
- 복사가 첫 추가에서 실패하는 경우
- 가득 찬 상태의 교체 복사가 실패하는 경우

#### 실패 조건

- 유효 범위를 벗어난 `at()`은 범위 오류를 보고한다.
- `T`의 복사 실패는 그대로 전파하며 상태를 보존한다.
- no-throw swap 계약을 만족하지 않는 `T`는 이 구현의 지원 범위 밖이다.

#### 필요한 제약

- C++98만 사용한다.
- 동적 메모리와 표준 sequence container를 사용하지 않는다.
- `T`는 기본 생성 가능하고, 복사 가능하며, 예외를 던지지 않는 `swap`을 제공한다고 가정한다.
- iterator 구현과 삭제 API는 범위에서 제외한다.

### 구현 후 자가 검증

- [ ] 빈 저장소에서 `size() == 0`이다.
- [ ] N개 이하에서는 삽입 순서와 논리 순서가 같다.
- [ ] N+1번째 추가 후 첫 값만 사라지고 나머지 순서가 유지된다.
- [ ] 여러 바퀴 순환한 뒤에도 가장 오래된 값과 최신 값이 정확하다.
- [ ] `at(size())`가 예외를 던지고 상태는 변하지 않는다.
- [ ] 복사 실패 전후 모든 논리 원소가 같다.
- [ ] 복사 실패 전후 `size_`와 다음 성공 push의 결과가 같다.
- [ ] 값 교체 성공 전에는 메타데이터를 변경하지 않는다.
- [ ] modulo 계산에 0으로 나누는 경우가 없다.
- [ ] push와 at의 시간 복잡도가 O(1), 저장 공간이 O(N)임을 설명할 수 있다.

### 구현 후 설명할 것

1. `next_`가 가득 차기 전과 후에 갖는 의미를 하나의 불변식으로 연결한 방식
   - 모범답변: `next_`는 항상 다음 성공 push가 기록할 물리 슬롯입니다. `size_ < N`이면 아직 사용하지 않은 다음 칸이고, `size_ == N`이면 가장 오래된 값을 가진 다음 교체 칸이므로 같은 갱신식 `(next_ + 1) % N`을 계속 사용할 수 있습니다.
2. 논리 인덱스를 물리 인덱스로 변환하는 식과 wrap-around 예시
   - 모범답변: `first = size_ == N ? next_ : 0`으로 두고 물리 인덱스는 `(first + logical_index) % N`입니다. 예를 들어 N=8, next_=3인 가득 찬 상태에서는 논리 0이 물리 3, 논리 4가 물리 7, 논리 5가 물리 0으로 감깁니다.
3. 슬롯 직접 대입 대신 지역 복사 후 swap을 사용한 실패 안전성
   - 모범답변: 직접 대입은 `T`가 일부 상태를 바꾼 뒤 던질 수 있습니다. 지역 복사 생성이 성공한 뒤 no-throw swap을 사용하면 복사 실패 때 슬롯이 그대로이고, 성공 때 옛 슬롯 값은 replacement가 소멸하며 정리합니다.
4. 메타데이터 갱신을 마지막 commit 단계로 둔 이유
   - 모범답변: `size_`와 `next_`는 배열 내용의 논리 순서를 해석하는 값입니다. 슬롯 교체 전에 갱신하면 실패 시 해석만 바뀌므로, no-throw swap 성공 후에만 전진시켜 내용과 메타데이터를 함께 commit합니다.
5. 고정 용량·예측 가능한 메모리와 동적 container의 유연성 사이 trade-off
   - 모범답변: 고정 배열은 추가 할당 없이 일정한 공간과 지연을 제공해 상한이 명확한 최근 기록에 적합합니다. 동적 container는 런타임 크기 변경과 일반 iterator 지원이 쉽지만 할당 실패·용량 정책·참조 무효화까지 고려해야 합니다.

### 원본 확인 위치

- Thread 01
- 커밋: `feat(contact): 고정 크기 연락처 저장 순서 보존`
- 커밋: `test(contact): 연락처 저장 용량과 논리 순서 검증`
- 커밋: `fix(contact): 할당 실패에도 저장 상태 보존`
- 커밋: `test(contact): 연락처 교체 실패 회귀 검증`
- 파일: `include/cppf/ContactBook.hpp`, `src/ContactBook.cpp`, `tests/test_contact_book.cpp`, `tests/failure/test_contact_failure.cpp`
- 클래스·함수: `cppf::ContactBook`, `add`, `at`, `size`, `contacts_`, `size_`, `next_`
- 관련 Thread: 12, 14

---

<a id="a-04"></a>
## [Thread 09 / `feat(template): 임의 접근 container batch 추상화 추가`, `test(template): iterator·정렬·복사 실패 계약 검증`] 대체 가능한 임의 접근 container 템플릿

### 면접 질문

`RandomAccessBatch<T, Container>`는 기본 `std::vector<T>`뿐 아니라 `std::deque<T>`도 사용할 수 있고, `at`, iterator, `sort`, 복사·대입을 제공합니다. 템플릿이 암묵적으로 요구하는 Container 계약을 설명하고, 왜 `std::list<T>`는 이 타입의 대체 container가 될 수 없는지 말해보십시오.

꼬리 질문:

- `typename Container::iterator`에서 `typename`이 필요한 이유는 무엇입니까?
  - 모범답변: `Container::iterator`는 template 인자에 의존하는 이름이라 파싱 시점에는 타입인지 정적 멤버인지 알 수 없습니다. `typename`으로 타입임을 명시해야 컴파일러가 typedef 선언으로 해석합니다.
- `at`을 `operator[]` 기반으로 직접 범위 검사하는 것과 `Container::at`에 위임하는 것의 차이는 무엇입니까?
  - 모범답변: 프로젝트 구현은 `size()`와 `operator[]`만 요구하고 자체 메시지의 `out_of_range`를 던져 container 계약을 명확히 합니다. `Container::at`에 맡기면 구현은 짧지만 대체 container에 `at`까지 요구하고 예외 메시지·세부 동작을 container에 위임합니다.
- `sort`가 `std::sort(begin(), end(), compare)`를 사용하면 필요한 iterator category는 무엇입니까?
  - 모범답변: random-access iterator가 필요합니다. `std::sort`는 임의 위치 이동과 iterator 차 계산을 사용하므로 bidirectional iterator만 제공하는 list 범위에는 적용할 수 없습니다.
- 복사 대입을 copy-and-swap으로 만들면 `T` 복사 실패 시 어떤 보장을 얻습니까?
  - 모범답변: 지역 batch의 `Container` 복사가 완전히 성공하기 전에는 대상 `values_`를 바꾸지 않습니다. 원소 복사나 할당이 실패하면 예외가 전파되고 대상의 크기와 값은 호출 전 그대로인 강한 보장을 얻습니다.
- 자기 대입에서 실제 container 복사를 생략하는 것이 왜 테스트 가능한 계약입니까?
  - 모범답변: `this != &other` 분기를 두면 자기 대입 때 `T` 복사 생성 횟수가 0이어야 합니다. 카운팅·실패 주입 원소를 사용하면 값 보존뿐 아니라 불필요한 복사가 없다는 선택까지 관찰할 수 있습니다.
- C++98에서 요구 조건을 명시적으로 concept로 표현할 수 없다면 어떻게 문서화·검증하겠습니까?
  - 모범답변: 필요한 typedef와 멤버·iterator category를 문서에 열거하고 vector/deque 긍정 컴파일·실행 테스트를 둡니다. list의 sort처럼 계약 밖 조합은 부정 컴파일 테스트로 실패 자체를 고정합니다.
- `equal_ranges`가 값뿐 아니라 길이까지 확인해야 하는 이유는 무엇입니까?
  - 모범답변: 공통 접두사의 모든 값이 같아도 한 범위에 원소가 더 남아 있으면 두 전체 범위는 같지 않습니다. 반복 종료 후 두 iterator가 각각 end에 도달했는지 함께 확인해야 합니다.

### 30초 모범 답변

이 템플릿은 Container에 기본·복사 생성, 대입 또는 swap, `push_back`, `size`, `empty`, `operator[]`, iterator와 const_iterator, 그리고 random-access iterator를 요구합니다. `vector`와 `deque`는 이를 만족하지만 `list`는 인덱싱이 없고 iterator가 bidirectional이라 `std::sort`를 사용할 수 없습니다. 대입은 완성된 container 복사본과 swap하면 원소 복사 실패 시 기존 batch를 보존합니다. C++98에서는 concept 문법이 없으므로 공개 문서, 긍정·부정 컴파일 테스트, 실제 vector/deque 인스턴스화로 계약을 고정할 수 있습니다.

### 답변 핵심 키워드

container requirements, substitutability, dependent type, `typename`, random-access iterator, `std::sort`, `operator[]`, copy-and-swap, compile-time negative test, generic range equality

### 백지 구현

#### 구현 목표

아래 API를 가진 C++98 template wrapper를 구현한다. `std::vector<T>`와 `std::deque<T>`에서 같은 동작을 하고, 지원하지 않는 container는 사용 시 컴파일 오류가 나도 된다.

#### 인터페이스 또는 함수 시그니처

```cpp
template <typename T, typename Container = std::vector<T> >
class RandomAccessBatch
{
public:
    typedef typename Container::iterator iterator;
    typedef typename Container::const_iterator const_iterator;

    RandomAccessBatch();
    RandomAccessBatch(const RandomAccessBatch &other);
    RandomAccessBatch &operator=(const RandomAccessBatch &other);

    void push_back(const T &value);
    std::size_t size() const;
    bool empty() const;
    T &at(std::size_t index);
    const T &at(std::size_t index) const;

    iterator begin();
    iterator end();
    const_iterator begin() const;
    const_iterator end() const;

    template <typename Compare>
    void sort(Compare compare);

    void swap(RandomAccessBatch &other);

private:
    Container values_;
};

template <typename FirstIterator, typename SecondIterator>
bool equal_ranges(FirstIterator first,
                  FirstIterator last,
                  SecondIterator second,
                  SecondIterator second_last);

template <typename T, typename Container>
RandomAccessBatch<T, Container>::RandomAccessBatch() : values_()
{
}

template <typename T, typename Container>
RandomAccessBatch<T, Container>::RandomAccessBatch(
    const RandomAccessBatch &other) : values_(other.values_)
{
}

template <typename T, typename Container>
RandomAccessBatch<T, Container> &RandomAccessBatch<T, Container>::operator=(
    const RandomAccessBatch &other)
{
    if (this != &other)
    {
        RandomAccessBatch copy(other);
        swap(copy); // container 복사가 성공한 뒤에만 commit한다.
    }
    return *this;
}

template <typename T, typename Container>
void RandomAccessBatch<T, Container>::push_back(const T &value)
{
    values_.push_back(value);
}

template <typename T, typename Container>
std::size_t RandomAccessBatch<T, Container>::size() const
{
    return static_cast<std::size_t>(values_.size());
}

template <typename T, typename Container>
bool RandomAccessBatch<T, Container>::empty() const
{
    return values_.empty();
}

template <typename T, typename Container>
T &RandomAccessBatch<T, Container>::at(std::size_t index)
{
    if (index >= values_.size())
        throw std::out_of_range("batch index");
    return values_[index];
}

template <typename T, typename Container>
const T &RandomAccessBatch<T, Container>::at(std::size_t index) const
{
    if (index >= values_.size())
        throw std::out_of_range("batch index");
    return values_[index];
}

template <typename T, typename Container>
typename RandomAccessBatch<T, Container>::iterator
RandomAccessBatch<T, Container>::begin()
{
    return values_.begin();
}

template <typename T, typename Container>
typename RandomAccessBatch<T, Container>::iterator
RandomAccessBatch<T, Container>::end()
{
    return values_.end();
}

template <typename T, typename Container>
typename RandomAccessBatch<T, Container>::const_iterator
RandomAccessBatch<T, Container>::begin() const
{
    return values_.begin();
}

template <typename T, typename Container>
typename RandomAccessBatch<T, Container>::const_iterator
RandomAccessBatch<T, Container>::end() const
{
    return values_.end();
}

template <typename T, typename Container>
template <typename Compare>
void RandomAccessBatch<T, Container>::sort(Compare compare)
{
    std::sort(values_.begin(), values_.end(), compare);
}

template <typename T, typename Container>
void RandomAccessBatch<T, Container>::swap(RandomAccessBatch &other)
{
    values_.swap(other.values_);
}

template <typename FirstIterator, typename SecondIterator>
bool equal_ranges(FirstIterator first,
                  FirstIterator last,
                  SecondIterator second,
                  SecondIterator second_last)
{
    while (first != last && second != second_last)
    {
        if (!(*first == *second))
            return false;
        ++first;
        ++second;
    }
    return first == last && second == second_last;
}
```

#### 입력과 출력

- `push_back`은 값을 뒤에 추가한다.
- `at`은 범위 검사된 참조를 반환한다.
- iterator는 내부 container의 전체 범위를 노출한다.
- `sort(compare)`는 comparator 순서로 내부 값을 정렬한다.
- `equal_ranges`는 두 범위의 길이와 각 위치의 동등성을 함께 비교한다.

#### 반드시 만족해야 할 조건

- 기본 container와 명시한 대체 container가 같은 공개 동작을 제공한다.
- const 객체에서는 const_iterator와 const 참조만 노출한다.
- `at(index)`는 `index >= size()`를 거부한다.
- 복사본의 원소는 원본과 독립적인 container 값이다.
- 대입 중 container 또는 원소 복사가 실패하면 대상 batch를 보존한다.
- 자기 대입은 값과 크기를 보존하며 불필요한 원소 복사를 하지 않아도 된다.
- `sort`는 random-access iterator를 전제로 한다.
- `equal_ranges`는 공통 접두사가 같아도 길이가 다르면 false다.

#### 경계 조건

- 빈 batch의 begin/end와 sort
- 한 원소 batch
- 첫·마지막 인덱스와 `at(size())`
- vector와 deque에 같은 입력과 comparator 적용
- 이미 정렬된 범위, 역순 범위, 같은 값이 여러 개인 범위
- 복사 생성 또는 대입 중 T 복사 실패
- 한 범위가 다른 범위의 접두사인 경우

#### 실패 조건

- 범위 밖 접근은 `std::out_of_range` 또는 동등한 범위 오류를 보고한다.
- 원소/container 복사 예외는 대상 값을 보존한 채 전파한다.
- 지원하지 않는 container는 template 인스턴스화 또는 `sort` 사용 시 컴파일 실패할 수 있다.

#### 필요한 제약

- C++98만 사용한다.
- 내부 container를 raw pointer로 별도 할당하지 않는다.
- `sort` 구현에 container 고유 멤버 함수가 아니라 iterator 기반 알고리즘을 사용한다.
- list 지원을 위한 별도 overload는 만들지 않는다.

### 구현 후 자가 검증

- [ ] vector 기본형과 deque 대체형에서 push/index/iterator 결과가 같다.
- [ ] const 객체가 가변 iterator나 가변 참조를 노출하지 않는다.
- [ ] 빈 범위의 `equal_ranges`가 true다.
- [ ] 길이가 다른 동일 접두사 범위가 false다.
- [ ] comparator를 사용한 정렬 결과가 두 container에서 같다.
- [ ] `std::distance`, `std::accumulate` 같은 iterator 알고리즘과 조합된다.
- [ ] `at(size())`가 범위 오류를 던진다.
- [ ] 복사 후 한쪽 원소 변경이 다른 쪽에 영향을 주지 않는다.
- [ ] 대입 중 원소 복사 실패가 대상 값을 보존한다.
- [ ] 자기 대입에서 복사 시도 횟수가 0이 되도록 선택했다면 테스트와 구현이 일치한다.
- [ ] `std::list` 기반 sort 사용이 지원 계약 밖임을 컴파일 계약으로 설명할 수 있다.

### 구현 후 설명할 것

1. 템플릿의 실제 Container 요구 조건을 멤버·iterator 연산으로 풀어낸 목록
   - 모범답변: `Container`에는 기본·복사 생성, `swap`, `push_back`, `size`, `empty`, `operator[]`, `begin/end`가 필요하고 `iterator`와 `const_iterator` 타입이 있어야 합니다. sort를 호출하는 경우 두 iterator는 `std::sort`가 요구하는 random-access 연산을 지원해야 합니다.
2. `vector`와 `deque`의 공통점, `list`가 빠지는 정확한 이유
   - 모범답변: vector와 deque는 뒤 삽입, 크기·빈 상태, 인덱싱, mutable/const random-access iterator를 모두 제공합니다. list는 `operator[]`가 없고 iterator가 bidirectional이라 현재 `at`과 `std::sort` 구현을 만족하지 않습니다.
3. C++98에서 concept 부재를 부정 컴파일 테스트로 보완하는 방식
   - 모범답변: 지원 container를 실제 인스턴스화하고 모든 공개 연산을 쓰는 긍정 테스트를 컴파일합니다. 반대로 `RandomAccessBatch<T, std::list<T> >::sort` 같은 계약 밖 코드는 컴파일 실패해야 한다는 별도 테스트로 iterator 요구를 확인합니다.
4. copy-and-swap이 generic container의 복사 실패까지 흡수하는 조건
   - 모범답변: `Container` 복사 생성이 실패하면 스스로 만든 부분 원소를 정리하고, `Container::swap`은 commit 단계에서 예외를 던지지 않아야 합니다. 이 조건이면 원소 `T`의 복사 예외도 지역 copy 안에 머물고 대상은 보존됩니다.
5. 범용성을 더 넓히기 위해 sort를 별도 free function으로 분리하는 대안과 trade-off
   - 모범답변: 정렬을 free function으로 분리하면 wrapper의 기본 저장·조회 계약은 bidirectional container까지 넓히고, random-access 범위에서만 `std::sort`를 사용할 수 있습니다. 대신 타입 하나가 항상 같은 정렬 API를 제공한다는 단순성이 줄고 container별 알고리즘 선택 책임이 호출자 쪽으로 이동합니다.

### 원본 확인 위치

- Thread 09
- 커밋: `feat(template): 임의 접근 container batch 추상화 추가`
- 커밋: `test(template): iterator·정렬·복사 실패 계약 검증`
- 파일: `include/cppf/RandomAccessBatch.hpp`, `tests/test_random_access_batch.cpp`
- 클래스·함수: `cppf::RandomAccessBatch<T, Container>`, `at`, `begin`, `end`, `sort`, `swap`, `equal_ranges`
- 관련 Thread: 11, 12, 13

---

<a id="s-05"></a>
## [Thread 11 / `feat(batch): 입력 문법과 원자 교체 구현`, `feat(batch): 결과 정렬과 직렬화 제공`] 배치 입력의 트랜잭션 평가와 결정적 결과

### 면접 질문

여러 줄의 `name | RPN-expression` 입력을 읽어 각 식을 계산하고 결과를 정렬한 뒤 기존 `BatchEngine` 결과를 교체합니다. 잘못된 한 줄, 중복 이름, RPN 문법 오류·overflow, stream 오류, 결과 저장 중 할당 실패가 발생해도 기존 결과와 기존 원소 참조를 보존하려면 처리 단계를 어떻게 나누겠습니까?

꼬리 질문:

- line마다 target 결과에 바로 push하면 어떤 실패 시나리오에서 부분 batch가 노출됩니까?
  - 모범답변: 앞줄이 성공한 뒤 중간 줄의 separator·중복·RPN 오류나 vector 재할당 실패가 나면 앞 결과만 target에 남습니다. 기존 batch도 이미 변경돼 호출 전 상태와 기존 원소 참조를 보존할 수 없습니다.
- `|`가 정확히 하나인지, 양쪽 field trim 후 name과 expression을 검증하는 순서는 왜 중요합니까?
  - 모범답변: 먼저 separator가 하나인지 확정해야 두 field의 경계가 모호하지 않습니다. 그다음 trim한 실제 의미 값으로 빈 field와 name 문법을 검사해야 공백뿐인 name·expression이나 추가 separator를 일관되게 거부할 수 있습니다.
- name 문법을 ASCII 기준으로 직접 제한하면 어떤 결정성을 얻습니까?
  - 모범답변: 현재 locale의 문자 분류와 signed char 처리에 영향을 받지 않고 같은 바이트가 모든 실행 환경에서 같은 유효성 결과를 냅니다. 허용 문자 집합과 사전식 tie-break의 입력 범위도 명확해집니다.
- 중복 이름을 계산 전에 찾는 것과 계산 후 찾는 것의 차이는 무엇입니까?
  - 모범답변: 계산 전에 찾으면 이미 무효인 줄의 RPN을 불필요하게 평가하지 않고 중복 오류를 우선 보고합니다. 계산 후 검사해도 candidate 방식이면 target 원자성은 유지되지만 overflow나 0 나누기 등이 중복보다 먼저 노출될 수 있습니다.
- 결과 comparator가 value가 같을 때 name으로 tie-break해야 하는 이유는 무엇입니까?
  - 모범답변: value만 비교하면 동등 그룹 내부 순서는 `std::sort`에서 보장되지 않습니다. name을 두 번째 키로 사용하면 strict weak ordering을 유지하면서 같은 입력 집합이 항상 같은 출력 순서를 갖습니다.
- 실패 시 기존 `results()[0]`의 주소까지 같다는 검증은 어떤 구현 속성을 확인합니까?
  - 모범답변: 단순히 같은 값을 다시 만들어 둔 것이 아니라 기존 `results_` container 자체를 commit 전혀 건드리지 않았음을 확인합니다. 실패 경로의 재할당·교체가 없어서 기존 원소 참조도 유효하다는 더 강한 관찰 속성입니다.
- 출력도 caller stream 설정과 무관하게 만들려면 어떻게 구성합니까?
  - 모범답변: classic locale를 imbue한 지역 `ostringstream`에서 정렬된 모든 줄을 완성하고 문자열을 얻은 뒤 `output.write`를 한 번 호출합니다. caller의 flags·precision·width·locale는 읽거나 변경하지 않습니다.
- vector와 deque를 둘 다 실행해 결과를 비교하는 방식은 어떤 결함을 찾고, 어떤 결함은 함께 공유할 수 있습니까?
  - 모범답변: 서로 다른 저장·iterator 구현에서 삽입·정렬 결과가 갈리면 container 가정이나 iterator 사용 결함을 찾을 수 있습니다. 하지만 같은 comparator와 같은 파싱·평가 로직을 공유하므로 comparator 자체의 잘못이나 공통 입력 해석 오류는 둘 다 같은 결과를 내며 놓칠 수 있습니다.

### 30초 모범 답변

입력 처리 전체를 지역 candidate에서 수행하고 마지막 no-throw swap만 commit으로 사용합니다. 각 줄은 separator 개수, trim된 field, ASCII name 문법, 중복 여부를 확인한 뒤 RPN을 평가해 candidate에 넣고, stream이 정상 EOF로 끝났으며 하나 이상 결과가 있는지 확인한 후 value와 name 순으로 정렬합니다. 이 과정의 예외는 candidate만 폐기하므로 기존 vector와 그 원소 주소가 실패 시 그대로 남습니다. 출력도 내부 classic-locale stream에서 전체 문자열을 만든 뒤 한 번 기록해 정렬 순서와 caller formatting에 무관한 바이트 결과를 제공합니다.

### 답변 핵심 키워드

transactional replace, local candidate, validation pipeline, exact separator, trim, ASCII identifier, duplicate set/map, exception propagation, deterministic comparator, stable observable state, reference preservation, render-then-write

### 백지 구현

#### 구현 목표

아래 축소된 `BatchProcessor`를 구현한다. 입력 전체가 유효할 때만 결과를 교체하고, 실패하면 이전 결과를 그대로 유지한다. RPN 평가는 앞 문제의 `evaluateRpn`을 호출한다고 가정한다.

#### 인터페이스 또는 함수 시그니처

```cpp
class JobResult
{
public:
    JobResult();
    JobResult(const std::string &name, long value);

    const std::string &name() const;
    long value() const;

private:
    std::string name_;
    long value_;
};

class BatchProcessor
{
public:
    void replace(std::istream &input);
    const std::vector<JobResult> &results() const;
    void write(std::ostream &output) const;

private:
    std::vector<JobResult> results_;
};

namespace
{

bool isBatchWhitespace(char value)
{
    return value == ' ' || value == '\t' || value == '\r' ||
           value == '\n' || value == '\v' || value == '\f';
}

std::string trimBatchField(const std::string &field)
{
    std::size_t first = 0;
    std::size_t last = field.size();

    while (first < last && isBatchWhitespace(field[first]))
        ++first;
    while (last > first && isBatchWhitespace(field[last - 1]))
        --last;
    return field.substr(first, last - first);
}

bool isBatchNameStart(char value)
{
    return (value >= 'A' && value <= 'Z') ||
           (value >= 'a' && value <= 'z');
}

bool isBatchNameRest(char value)
{
    return isBatchNameStart(value) ||
           (value >= '0' && value <= '9') || value == '_' || value == '-';
}

bool isValidBatchName(const std::string &name)
{
    if (name.empty() || !isBatchNameStart(name[0]))
        return false;
    for (std::size_t i = 1; i < name.size(); ++i)
    {
        if (!isBatchNameRest(name[i]))
            return false;
    }
    return true;
}

void parseBatchLine(const std::string &line,
                    std::string &name,
                    std::string &expression)
{
    const std::size_t separator = line.find('|');

    if (separator == std::string::npos ||
        line.find('|', separator + 1) != std::string::npos)
        throw std::invalid_argument("invalid batch input");
    name = trimBatchField(line.substr(0, separator));
    expression = trimBatchField(line.substr(separator + 1));
    if (!isValidBatchName(name) || expression.empty())
        throw std::invalid_argument("invalid batch input");
}

bool readBatchLine(std::istream &input, std::string &line)
{
    char value;

    line.clear();
    while (input.get(value))
    {
        if (value == '\n')
            return true;
        line.push_back(value);
    }
    if (!input.eof())
        throw std::invalid_argument("invalid batch input");
    return !line.empty();
}

bool batchResultLess(const JobResult &left, const JobResult &right)
{
    if (left.value() != right.value())
        return left.value() < right.value();
    return left.name() < right.name();
}

bool batchResultsEqual(
    const RandomAccessBatch<JobResult> &left,
    const RandomAccessBatch<JobResult, std::deque<JobResult> > &right)
{
    if (left.size() != right.size())
        return false;
    for (std::size_t i = 0; i < left.size(); ++i)
    {
        if (left.at(i).name() != right.at(i).name() ||
            left.at(i).value() != right.at(i).value())
            return false;
    }
    return true;
}

}

JobResult::JobResult() : name_(), value_(0)
{
}

JobResult::JobResult(const std::string &name, long value)
    : name_(name), value_(value)
{
}

const std::string &JobResult::name() const
{
    return name_;
}

long JobResult::value() const
{
    return value_;
}

void BatchProcessor::replace(std::istream &input)
{
    RandomAccessBatch<JobResult> vector_batch;
    RandomAccessBatch<JobResult, std::deque<JobResult> > deque_batch;
    std::map<std::string, long> seen;
    std::string line;

    while (readBatchLine(input, line))
    {
        std::string name;
        std::string expression;

        parseBatchLine(line, name, expression);
        if (seen.find(name) != seen.end())
            throw std::invalid_argument("invalid batch input");

        const long value = evaluateRpn(expression);
        const JobResult result(name, value);

        seen.insert(std::make_pair(name, value));
        vector_batch.push_back(result);
        deque_batch.push_back(result);
    }
    if (vector_batch.empty())
        throw std::invalid_argument("invalid batch input");

    vector_batch.sort(batchResultLess);
    deque_batch.sort(batchResultLess);
    if (!batchResultsEqual(vector_batch, deque_batch))
        throw std::logic_error("batch container disagreement");

    std::vector<JobResult> candidate(vector_batch.begin(),
                                     vector_batch.end());
    results_.swap(candidate); // 모든 실패 가능 작업 뒤의 유일한 commit이다.
}

const std::vector<JobResult> &BatchProcessor::results() const
{
    return results_;
}

void BatchProcessor::write(std::ostream &output) const
{
    std::ostringstream rendered;

    rendered.imbue(std::locale::classic());
    for (std::size_t i = 0; i < results_.size(); ++i)
        rendered << results_[i].value() << " | "
                 << results_[i].name() << '\n';
    const std::string text = rendered.str();

    output.write(text.data(), static_cast<std::streamsize>(text.size()));
}
```

#### 입력과 출력

입력 한 줄 형식:

```text
name | RPN-expression
```

- field 앞뒤의 ASCII 공백 문자는 trim한다.
- name 첫 글자는 `A-Z` 또는 `a-z`다.
- 이후 글자는 영문자, 숫자, `_`, `-`만 허용한다.
- 같은 name은 한 batch 안에서 한 번만 허용한다.
- 빈 batch와 빈 줄은 허용하지 않는다.
- 성공한 결과는 value 오름차순, 같은 value면 name 사전순으로 정렬한다.
- 출력은 각 결과를 `value | name\n` 형식으로 쓴다.

#### 반드시 만족해야 할 조건

- 줄마다 `|`가 정확히 하나 있어야 한다.
- trim 뒤 name과 expression이 모두 비어 있지 않아야 한다.
- name 문법은 locale와 무관한 ASCII 비교로 검사한다.
- 중복 이름을 거부한다.
- RPN 평가 오류와 overflow는 의미를 보존해 전파한다.
- input 반복이 정상 EOF로 끝나지 않으면 batch 입력 오류다.
- 모든 parse·evaluate·insert·sort가 끝나기 전까지 `results_`를 변경하지 않는다.
- 성공 commit은 예외를 던지지 않는 container swap으로 수행한다.
- 실패 시 기존 `results_`의 크기, 값, 원소 주소가 그대로다.
- 출력 순서와 숫자 형식은 caller locale·flags·precision과 무관하다.
- `results()`는 const 참조만 반환해 외부에서 결과를 변조할 수 없게 한다.

#### 경계 조건

- 마지막 줄에 newline이 있는 경우와 없는 경우
- CRLF 입력에서 expression 끝의 `\r`
- 공백이 많은 field
- 빈 줄, 중간 빈 줄, trailing blank line
- separator 누락과 두 개 이상 separator
- digit로 시작하는 name, 허용 punctuation `_`와 `-`, 금지 punctuation `.`
- embedded NUL과 비ASCII name
- 첫 줄과 마지막 줄의 중복 이름
- RPN 0 나누기와 overflow
- stream이 미리 bad 상태인 경우
- 같은 value를 가진 여러 name
- 결과 vector 할당·정렬 중 예외

#### 실패 조건

- 입력 문법·중복·비정상 stream·빈 batch는 batch 입력 오류다.
- RPN 문법 오류는 해당 오류 형식을 유지한다.
- RPN overflow는 overflow 오류를 유지한다.
- 메모리 할당 실패는 그대로 전파하되 기존 결과를 보존한다.

#### 필요한 제약

- C++98만 사용한다.
- 파일 전체를 하나의 문자열로 먼저 읽는 방식은 요구하지 않는다.
- target `results_`에 임시 marker나 부분 결과를 기록하지 않는다.
- 정렬 comparator는 strict weak ordering을 만족해야 한다.
- 출력 시 caller stream 설정을 변경하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 여러 줄 입력이 계산되고 value/name 순으로 정렬된다.
- [ ] 입력 stream의 lifetime이 끝난 뒤에도 결과 문자열을 소유한다.
- [ ] 마지막 newline 유무가 정상 입력 결과를 바꾸지 않는다.
- [ ] CRLF와 field trim이 의도대로 처리된다.
- [ ] separator 누락·추가, 빈 field, 잘못된 name을 모두 거부한다.
- [ ] exact duplicate를 거부하고 기존 결과와 첫 원소 주소를 보존한다.
- [ ] RPN 문법 오류와 overflow가 기존 결과를 보존한다.
- [ ] bad stream이 기존 결과를 보존한다.
- [ ] candidate push 또는 최종 vector 구성의 할당 실패가 기존 결과를 보존한다.
- [ ] 같은 value에서 name tie-break가 결정적이다.
- [ ] 같은 상태를 두 번 write한 바이트 결과가 동일하다.
- [ ] caller stream의 locale·flags·fill·width·precision이 호출 뒤 그대로다.
- [ ] 시간 복잡도를 입력 총 길이와 결과 수 m에 대해 O(total input + m log m)로 설명할 수 있다.

### 구현 후 설명할 것

1. parse·duplicate check·evaluate·sort를 모두 candidate 안에서 끝내는 트랜잭션 경계
   - 모범답변: 줄 읽기·구문 검증·중복 검사·RPN 평가·두 container 삽입·정렬·대조·최종 vector 구성까지 모두 지역 객체에서 수행합니다. 이 전 과정이 성공한 뒤 `results_.swap(candidate)` 한 문장만 target을 바꾸는 commit입니다.
2. 실패 시 값뿐 아니라 기존 원소 주소까지 보존되는 이유
   - 모범답변: 실패 가능한 동안 `results_`에 push·clear·대입을 전혀 하지 않으므로 내부 vector의 저장소도 재할당되지 않습니다. 예외 시 지역 후보만 소멸해 기존 크기·값뿐 아니라 기존 원소의 주소도 그대로입니다.
3. 이름 문법과 정렬 tie-break를 명시해 실행 환경별 차이를 줄인 방식
   - 모범답변: 이름은 locale 함수가 아니라 ASCII 범위를 직접 비교해 같은 바이트 집합만 허용합니다. 정렬은 value 뒤 name을 비교해 동점 순서를 완전히 정하므로 container와 `std::sort`의 동등 원소 배치 차이가 출력에 남지 않습니다.
4. 오류 형식을 통합하지 않고 RPN 문법·overflow를 전파한 진단 trade-off
   - 모범답변: batch 구조 오류만 `invalid_argument`로 만들고 RPN 평가의 문법·overflow 예외는 그대로 통과시켜 어느 계층에서 왜 실패했는지 보존합니다. 호출자는 여러 예외를 처리해야 하지만 재시도나 사용자 메시지를 더 정확히 선택할 수 있습니다.
5. vector/deque 이중 계산이 제공하는 런타임 오라클과 comparator 공유로 남는 공통 결함 위험
   - 모범답변: 동일 결과를 두 저장 구조에 넣고 독립 정렬한 뒤 비교하면 특정 container 동작을 잘못 가정한 결함을 찾는 상호 오라클이 됩니다. 다만 파서·평가기와 comparator를 공유하므로 공통 로직이 같은 방식으로 틀리면 두 결과가 일치해도 오류를 발견하지 못합니다.

### 원본 확인 위치

- Thread 11
- 커밋: `feat(batch): 입력 문법과 원자 교체 구현`
- 커밋: `feat(batch): 결과 정렬과 직렬화 제공`
- 커밋: `feat(batch): 두 container의 정렬 결과 대조`
- 관련 실패 검증 커밋: Thread 12의 `test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증`
- 파일: `include/cppf/BatchEngine.hpp`, `src/BatchEngine.cpp`, `tests/test_batch_engine.cpp`, `tests/failure/test_batch_failure.cpp`
- 클래스·함수: `cppf::JobResult`, `cppf::BatchEngine::replace`, `results`, `write`, `trimField`, `isValidName`, `parseLine`, `resultLess`
- 관련 Thread: 09, 10, 12, 14
