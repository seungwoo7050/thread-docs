# C++98 제네릭 선택과 반복자

C++98 환경에서 오버로드 선택과 반복자 표현을 설명하고 직접 축소 구현하는 문제다. 두 항목 모두 이후 `vector`와 `map` 공개 API의 기반이 된다.

<a id="gen-01"></a>
## GEN-01 — [Thread 01 / `feat(type-traits): CXX98 타입 선택 도구 구현`] C++98에서 개수 오버로드와 범위 오버로드 분리

### 면접 질문

`vector(count, value)`와 `vector(first, last)`처럼 인자 개수가 같은 오버로드를 C++98에서 어떻게 구분했습니까?  
정수 두 개가 범위 생성자로 해석되는 문제를 설명하고, `enable_if`와 `is_integral`이 어느 시점에 후보를 제거하는지 말해보세요.

꼬리 질문:
- `is_integral<const int>` 처리는 현재 프로젝트 범위에서 어떻게 제한되어 있습니까?
- SFINAE로 제거되는 오류와 함수 본문에서 발생하는 컴파일 오류는 무엇이 다릅니까?
- 범위 생성자뿐 아니라 `assign`과 `insert`에도 같은 제약이 필요한 이유는 무엇입니까?

### 30초 모범 답변

C++98에는 concepts나 표준 type traits가 없으므로 템플릿 인자 치환 단계에서 범위 오버로드를 후보에서 제거해야 합니다. 이 프로젝트는 `enable_if<!is_integral<InputIt>::value>`를 범위 생성자·`assign`·`insert`에 붙여 정수 인자는 개수 오버로드로만 가게 했습니다. 핵심은 오류를 함수 본문이 아니라 선언의 치환 문맥에서 만들고, 지원하는 정수 형식의 범위를 명시하는 것입니다. 현재 구현은 cv 한정을 자동 정규화하지 않는 제한도 공개 계약에 남겼습니다.

### 답변 핵심 키워드

- SFINAE
- substitution failure
- overload candidate 제거
- `enable_if`
- `is_integral`
- count-vs-range 모호성
- C++98 제약
- cv 한정 제한

### 백지 구현

- **구현 목표**: 정수 인자는 개수 오버로드로, 반복자 인자는 범위 오버로드로 선택되는 최소 C++98 API를 구현한다.
- **인터페이스 또는 함수 시그니처**: `enable_if`, `is_integral`, 그리고 두 개의 `assign` 오버로드를 제공한다.
- **입력과 출력**: 개수와 값 또는 `[first, last)` 범위를 받아 내부 시퀀스를 갱신한다.
- **반드시 만족해야 할 조건**: 정수 두 개로 호출할 때 범위 템플릿이 후보가 아니어야 하며, 포인터 반복자는 범위 오버로드를 선택해야 한다.
- **경계 조건**: 빈 범위, `count == 0`, 서로 다른 정수 형식, `const` 반복자.
- **실패 조건**: 지원하지 않는 인자 형식은 애매한 런타임 분기가 아니라 컴파일 단계에서 거부한다.
- **필요한 제약**: C++98만 사용하고 `<type_traits>`와 concepts를 사용하지 않는다.

```cpp
// 직접 구현
template <bool Condition, class T = void>
struct enable_if;

template <class T>
struct is_integral;

template <class T>
class range_buffer
{
public:
    typedef std::size_t size_type;

    void assign(size_type count, const T& value);

    template <class InputIt>
    void assign(InputIt first, InputIt last,
        typename enable_if<!is_integral<InputIt>::value>::type* = 0);

private:
    // 저장 방식은 자유롭게 선택
};
```

### 구현 후 자가 검증

- `assign(3, 7)`이 개수 오버로드를 선택한다.
- `int values[]`의 포인터 범위가 범위 오버로드를 선택한다.
- 빈 범위를 전달해도 상태가 유효하다.
- 지원하기로 한 모든 정수 형식에서 결과가 일관된다.
- 제약이 함수 본문이 아니라 선언의 치환 문맥에 걸려 있다.
- 불필요한 런타임 분기나 태그 값 저장이 없다.

### 구현 후 설명할 것

1. 왜 단순히 두 템플릿을 선언하면 정수 인자에서 오버로드가 잘못 선택되거나 모호해지는가.
2. SFINAE가 적용되는 즉시 문맥과 함수 본문 오류의 차이.
3. 정수 특수화를 어디까지 제공할지와 cv 한정 지원 범위의 trade-off.
4. 같은 제약을 생성자·`assign`·`insert`에 일관되게 적용해야 하는 이유.

### 원본 확인 위치

- **Thread**: 01, 04, 05
- **커밋 메시지**: `feat(type-traits): CXX98 타입 선택 도구 구현`; `feat(vector): 크기 변경과 값 범위 할당 구현`
- **파일**: `include/ft_type_traits.hpp`, `include/ft_vector.hpp`
- **함수·클래스·컴포넌트**: `ft::enable_if`, `ft::is_integral`, `ft::vector`의 범위 생성자·`assign`·`insert`
- **관련된 다른 Thread**: Thread 04의 시퀀스 API, Thread 05의 범위 변경·별칭 처리

<a id="gen-02"></a>
## GEN-02 — [Thread 01 / `feat(iterator): 역방향 반복자의 양방향 동작 구현`] `reverse_iterator::base()`와 one-past 표현

### 면접 질문

역방향 반복자가 현재 원소 자체가 아니라 그 다음 위치를 `base()`로 저장하는 이유를 설명해보세요.  
`*rbegin()`, `rend()`, `++reverse_iterator`, 두 역방향 반복자의 거리와 대소 비교가 정방향 반복자와 어떻게 연결됩니까?

꼬리 질문:
- 빈 범위에서 `rbegin() == rend()`가 자연스럽게 성립하는 이유는 무엇입니까?
- 역방향 비교 연산의 방향을 뒤집지 않으면 어떤 오류가 생깁니까?
- 가변 반복자에서 상수 역방향 반복자로 변환할 때 필요한 조건은 무엇입니까?

### 30초 모범 답변

역방향 반복자는 반열린 범위 계약을 유지하려고 정방향의 one-past 위치를 `base()`로 저장합니다. 따라서 역참조는 `base()`의 직전 원소를 가리키고, 역방향 증가에는 정방향 감소가 대응합니다. 거리와 순서도 방향이 반대이므로 피연산자 순서를 뒤집어야 합니다. 이 표현 덕분에 `reverse_iterator(end())`가 `rbegin()`이 되고, 빈 범위에서는 `begin() == end()`만으로 `rbegin() == rend()`가 됩니다.

### 답변 핵심 키워드

- one-past
- `base()`
- 반열린 범위
- 역참조는 직전 원소
- 증가·감소 반전
- 비교 순서 반전
- 거리 부호
- 빈 범위

### 백지 구현

- **구현 목표**: 포인터 또는 임의 접근 반복자를 감싸는 최소 역방향 반복자를 구현한다.
- **인터페이스 또는 함수 시그니처**: 생성자, `base`, `operator*`, 전위 `++/--`, `+`, `-`, 차이, 비교 연산을 제공한다.
- **입력과 출력**: 기저 반복자를 받아 역방향 순회와 위치 연산을 제공한다.
- **반드시 만족해야 할 조건**: `reverse_iterator(last)`의 첫 역참조 결과는 `*(last - 1)`과 같아야 한다.
- **경계 조건**: 빈 범위, 한 원소 범위, `n == 0`, 서로 다른 const 자격 반복자 비교.
- **실패 조건**: `rend()` 역참조처럼 원래 반복자 계약에서 금지된 연산은 지원 대상으로 삼지 않는다.
- **필요한 제약**: 기저 반복자가 제공하지 않는 연산은 역방향 반복자에서도 요구하지 않는다.

```cpp
template <class Iterator>
class reverse_cursor
{
public:
    typedef Iterator iterator_type;

    reverse_cursor();
    explicit reverse_cursor(iterator_type current);

    iterator_type base() const;

    typename iterator_traits<Iterator>::reference operator*() const;
    reverse_cursor& operator++();
    reverse_cursor& operator--();
    reverse_cursor operator+(typename iterator_traits<Iterator>::difference_type n) const;

private:
    iterator_type current_;
};

// 직접 구현: 필요한 비멤버 비교·거리 연산도 선언
```

### 구현 후 자가 검증

- 빈 배열 범위에서 `rbegin == rend`이다.
- 한 원소 범위에서 한 번 역참조한 뒤 증가하면 `rend`가 된다.
- `++rit`가 기저 반복자를 한 칸 감소시킨다.
- `rit + n`과 `rit[n]`의 의미가 일치한다.
- 정방향에서 앞선 위치가 역방향 비교에서는 뒤로 평가된다.
- 두 역방향 반복자의 차이가 순회 방향과 일치한다.
- `base()` 자체는 현재 역참조 원소가 아니라 one-past 위치를 반환한다.

### 구현 후 설명할 것

1. 현재 원소 포인터를 직접 저장하는 방식보다 one-past 표현이 반열린 범위와 잘 맞는 이유.
2. 비교와 거리 연산에서 피연산자 순서를 뒤집는 이유.
3. 양방향 반복자와 임의 접근 반복자에서 제공 가능한 연산 차이.
4. 가변→상수 변환은 허용하고 상수→가변 변환은 막아야 하는 이유.

### 원본 확인 위치

- **Thread**: 01, 04, 06, 08
- **커밋 메시지**: `feat(iterator): 역방향 반복자의 양방향 동작 구현`; `feat(iterator): 역방향 반복자의 임의 접근 연산 완성`
- **파일**: `include/ft_iterator.hpp`
- **함수·클래스·컴포넌트**: `ft::reverse_iterator`, `base`, 역참조·증감·산술·비교 연산
- **관련된 다른 Thread**: Thread 04의 `vector` 역방향 순회, Thread 06·08의 `map` 역방향 반복자
