# Vector 저장소·용량·경계

`vector`의 가장 중요한 면접 지점인 원시 저장소, 객체 수명, 재할당 트랜잭션, 용량 산술, 빈 저장소 경계를 다룬다.

<a id="vec-01"></a>
## VEC-01 — [Thread 04 / `feat(vector): allocator 기반 저장소 수명 관리`] 연속 저장소와 객체 수명 invariant

### 면접 질문

`vector`의 메모리 블록과 그 안에 실제로 생성된 객체를 구분해서 설명해보세요.  
`data`, `size`, `capacity`가 유지해야 하는 invariant는 무엇이며, 소멸·`clear`·저장소 해제 순서는 왜 중요합니까?

꼬리 질문:
- 할당된 공간 전체를 이미 생성된 `T` 배열처럼 다루면 어떤 문제가 생깁니까?
- `allocate`, `construct`, `destroy`, `deallocate`의 책임을 각각 설명해보세요.
- 생성 도중 예외가 발생했을 때 `_size`를 언제 증가시키는 것이 안전합니까?

### 30초 모범 답변

`[data, data + size)`만 살아 있는 `T` 객체이고 `[data + size, data + capacity)`는 아직 객체가 없는 원시 저장 공간입니다. 따라서 항상 `size <= capacity`이고 생성된 객체 수가 정확히 `size`여야 합니다. 해제할 때는 살아 있는 객체를 먼저 `destroy`한 뒤 블록을 `deallocate`해야 하며, 생성 중 실패하면 성공한 접두 구간만 역순 정리해야 합니다. 포인터·크기·용량은 소유권 상태이므로 부분 갱신하지 않는 것도 중요합니다.

### 답변 핵심 키워드

- raw storage
- constructed range
- `size <= capacity`
- object lifetime
- `allocate/construct/destroy/deallocate`
- 성공한 접두 구간
- 소유권
- RAII

### 백지 구현

- **구현 목표**: allocator로 원시 저장소를 확보하고 정확한 수의 객체만 생성·소멸하는 작은 버퍼를 구현한다.
- **인터페이스 또는 함수 시그니처**: fill 생성자, 소멸자, `clear`, `size`, `capacity`를 제공한다.
- **입력과 출력**: `count`와 초기값을 받아 `count`개의 살아 있는 객체를 소유한다.
- **반드시 만족해야 할 조건**: 어떤 시점에도 생성된 객체 수와 `size`가 다르지 않아야 한다.
- **경계 조건**: `count == 0`, 첫 번째 복사에서 실패, 마지막 복사에서 실패.
- **실패 조건**: 할당 또는 원소 생성 실패 시 블록과 이미 생성된 객체가 남지 않아야 한다.
- **필요한 제약**: 배열 `new[]` 대신 전달된 allocator의 수명 API를 사용한다.

```cpp
template <class T, class Alloc>
class raw_vector_core
{
public:
    typedef typename Alloc::pointer pointer;
    typedef typename Alloc::size_type size_type;

    raw_vector_core(size_type count, const T& value,
        const Alloc& alloc = Alloc());
    ~raw_vector_core();

    void clear();
    size_type size() const;
    size_type capacity() const;

private:
    Alloc alloc_;
    pointer data_;
    size_type size_;
    size_type capacity_;

    void destroy_storage();
    // 직접 구현
};
```

### 구현 후 자가 검증

- `count == 0`에서 할당 없이 유효한 빈 상태가 된다.
- 정상 생성 뒤 살아 있는 객체 수가 `size`와 같다.
- 중간 복사 생성 실패 시 성공한 객체만 한 번씩 소멸한다.
- 실패한 생성자에서 메모리 블록이 남지 않는다.
- `clear`는 객체만 없애고 블록 소유 여부는 설계한 계약과 일치한다.
- 소멸자는 모든 객체를 소멸한 뒤 정확히 한 번 블록을 해제한다.
- double destroy, destroy-before-construct, deallocate-before-destroy가 없다.

### 구현 후 설명할 것

1. 원시 저장 공간과 객체 수명을 분리하지 않으면 비정상 소멸·복사 문제가 생기는 이유.
2. `size`를 생성 성공 뒤에만 증가시키는 이유.
3. 정리 루프의 범위를 별도 카운터로 추적하는 방법.
4. `clear`와 전체 저장소 해제를 분리했을 때 얻는 재사용성과 복잡도.

### 원본 확인 위치

- **Thread**: 04, 05
- **커밋 메시지**: `feat(vector): allocator 기반 저장소 수명 관리`
- **파일**: `include/ft_vector.hpp`
- **함수·클래스·컴포넌트**: `ft::vector`, `_alloc`, `_data`, `_size`, `_capacity`, `_destroy_storage`
- **관련된 다른 Thread**: Thread 05의 생성·삽입 예외 안전성과 객체 수명 보정

<a id="vec-02"></a>
## VEC-02 — [Thread 04 / `feat(vector): 용량 확장과 원소 재배치 구현`] 재할당을 새 블록에서 완성한 뒤 커밋하기

### 면접 질문

`reserve`나 성장 과정에서 기존 블록을 먼저 지우면 안 되는 이유는 무엇입니까?  
새 블록 할당, 원소 복사 생성, 실패 정리, 기존 상태 폐기, 새 상태 반영의 경계를 설명해보세요.

꼬리 질문:
- 몇 번째 원소 복사에서 예외가 나도 기존 `vector`가 보존되려면 무엇을 추적해야 합니까?
- 재할당 성공 시 기존 반복자와 참조가 무효화되는 이유는 무엇입니까?
- 복사 생성만 있는 C++98 타입에서 이동 최적화가 없는 비용은 얼마입니까?

### 30초 모범 답변

재할당은 새 블록을 임시 소유권으로 다루는 트랜잭션입니다. 새 블록에서 기존 원소 전체의 구성이 끝나기 전에는 `data`, `size`, `capacity`와 기존 객체를 건드리지 않습니다. 복사 중 실패하면 새 블록에 성공한 원소만 소멸하고 블록을 해제하면 원래 상태가 그대로 남습니다. 모두 성공한 뒤에만 기존 객체와 블록을 정리하고 새 포인터·용량으로 커밋합니다.

### 답변 핵심 키워드

- commit point
- temporary ownership
- strong exception guarantee
- constructed count
- rollback
- iterator invalidation
- O(n) copy
- 상태 일괄 교체

### 백지 구현

- **구현 목표**: 기존 내용을 보존하면서 더 큰 allocator 블록으로 재할당하는 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: `void reserve(size_type new_capacity)`와 내부 `reallocate`를 구현한다.
- **입력과 출력**: 요청 용량이 현재 용량보다 크면 새 블록으로 옮기고, 아니면 아무 변화도 없다.
- **반드시 만족해야 할 조건**: 원소 복사 실패 시 포인터·크기·용량·기존 값이 변하지 않아야 한다.
- **경계 조건**: 빈 벡터, 같은 용량, `new_capacity == size`, 복사 실패 위치 0과 `size - 1`.
- **실패 조건**: 할당 실패 또는 원소 복사 생성 실패.
- **필요한 제약**: 성공 전에는 기존 저장소를 파괴하거나 상태 필드를 갱신하지 않는다.

```cpp
template <class T, class Alloc>
class vector_core
{
public:
    typedef typename Alloc::size_type size_type;

    void reserve(size_type new_capacity);

private:
    typedef typename Alloc::pointer pointer;

    Alloc alloc_;
    pointer data_;
    size_type size_;
    size_type capacity_;

    void reallocate(size_type new_capacity);
    // 직접 구현
};
```

### 구현 후 자가 검증

- 요청 용량이 현재 용량 이하이면 할당·복사가 발생하지 않는다.
- 빈 상태 재할당이 유효한 첫 블록을 만든다.
- 각 실패 위치에서 기존 포인터·크기·용량·원소 값이 보존된다.
- 새 블록에서 성공한 객체만 정리되고 미생성 슬롯은 소멸하지 않는다.
- 성공 경로에서 기존 객체를 모두 소멸한 후 기존 블록을 해제한다.
- 새 상태 필드가 한 커밋 구간에서 일관되게 반영된다.
- 시간 복잡도 O(n), 추가 공간 O(new_capacity)를 설명할 수 있다.

### 구현 후 설명할 것

1. 재할당을 두 소유권 상태 사이의 트랜잭션으로 보는 이유.
2. 성공한 생성 개수를 별도로 추적해야 하는 이유.
3. 기존 상태를 먼저 변경하는 구현이 기본 보장조차 깨뜨릴 수 있는 경로.
4. 강한 예외 보장과 재할당 시 반복자 무효화가 동시에 성립하는 이유.

### 원본 확인 위치

- **Thread**: 04, 05
- **커밋 메시지**: `feat(vector): 용량 확장과 원소 재배치 구현`; `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리`
- **파일**: `include/ft_vector.hpp`
- **함수·클래스·컴포넌트**: `reserve`, `_next_capacity`, `_reallocate`, `_swap_storage`, `_replace_storage`
- **관련된 다른 Thread**: Thread 05의 생성·대입·삽입 rollback

<a id="vec-03"></a>
## VEC-03 — [Thread 04 / `fix(vector): 용량 계산을 allocator 상한에서 포화`] 용량 배수 성장의 overflow와 `max_size()` 경계

### 면접 질문

용량을 단순히 `capacity * 2`로 계산하면 어떤 경계에서 잘못됩니까?  
`minimum`, 현재 용량, allocator의 `max_size()`를 이용해 덧셈과 곱셈 overflow 없이 다음 용량을 정하는 원칙을 설명해보세요.

꼬리 질문:
- `_size + count > max_size()`보다 `count > max_size() - _size`가 안전한 이유는 무엇입니까?
- 두 배 성장 대신 정확한 최소 용량만 할당하는 전략의 trade-off는 무엇입니까?
- `reserve(max_size() + 1)`은 어느 단계에서 거부해야 합니까?

### 30초 모범 답변

부호 없는 크기형도 overflow하면 작은 값으로 순환하므로 계산 뒤 비교만 해서는 늦습니다. 필요한 원소 수는 `count > limit - size`처럼 뺄셈 기반으로 먼저 검증하고, 두 배 계산도 `capacity > limit - capacity`인지 확인해 상한으로 포화시킵니다. 그다음 최소 요구량과 비교해 더 큰 값을 선택하되 항상 `limit` 이하임을 보장합니다. 이 검사는 할당 전에 `length_error`로 실패 경계를 고정합니다.

### 답변 핵심 키워드

- unsigned wraparound
- precondition before arithmetic
- `limit - size`
- saturating growth
- `max_size()`
- `length_error`
- amortized O(1)
- allocator limit

### 백지 구현

- **구현 목표**: overflow 없이 다음 벡터 용량을 계산하는 순수 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: `size_type next_capacity(size_type current, size_type minimum, size_type limit)`.
- **입력과 출력**: 현재 용량·필요 최소 용량·상한을 받아 `[minimum, limit]` 범위의 다음 용량을 반환한다.
- **반드시 만족해야 할 조건**: 가능하면 배수 성장을 사용하고, 배수 계산이 상한을 넘으면 `limit`로 포화한다.
- **경계 조건**: `current == 0`, `minimum == current`, `minimum == limit`, `current > limit / 2`.
- **실패 조건**: `minimum > limit`이면 명시적으로 실패한다.
- **필요한 제약**: overflow가 일어난 뒤 결과를 검사하는 방식은 금지한다.

```cpp
template <class Size>
Size next_capacity(Size current, Size minimum, Size limit)
{
    // 직접 구현
}
```

### 구현 후 자가 검증

- `current == 0`, `minimum == 0`의 정책이 명확하다.
- 일반 구간에서 최소 요구량 이상을 반환한다.
- 반환값이 절대 `limit`를 넘지 않는다.
- `current`가 상한의 절반보다 큰 경우 곱셈 overflow가 없다.
- `minimum > limit`이 할당 시도 전에 실패한다.
- 큰 `count` 삽입 검사에서 `_size + count`를 먼저 계산하지 않는다.
- 성장 정책이 상각 복잡도에 미치는 영향을 설명할 수 있다.

### 구현 후 설명할 것

1. 부호 없는 정수 overflow가 정의된 wraparound라도 논리 오류인 이유.
2. 검사 순서를 계산 전으로 옮겨야 하는 이유.
3. 상한 포화와 정확한 최소 할당 사이의 메모리·성능 trade-off.
4. `length_error`와 실제 allocator의 `bad_alloc`을 구분하는 기준.

### 원본 확인 위치

- **Thread**: 04, 05
- **커밋 메시지**: `fix(vector): 용량 계산을 allocator 상한에서 포화`; `test(vector): 제한 allocator에서 용량 상한 검증`
- **파일**: `include/ft_vector.hpp`, `tests/test_vector_exceptions.cpp`
- **함수·클래스·컴포넌트**: `_next_capacity`, `reserve`, `resize`, `assign`, `insert`, `bounded_allocator`
- **관련된 다른 Thread**: Thread 05의 예외 주입 allocator

<a id="vec-04"></a>
## VEC-04 — [Thread 04 / `fix(vector): allocator 형식과 빈 반복자 연산 보정`] 빈 저장소에서 null 포인터 산술 피하기

### 면접 질문

빈 `vector`가 `data == NULL`일 때 `end()`를 `data + size`로 계산하거나 두 반복자를 빼는 코드가 왜 위험합니까?  
빈 상태에서도 `begin() == end()`, 0개 삽입, 빈 범위 삭제가 동작하도록 위치 계산을 어떻게 경계 지을 수 있습니까?

꼬리 질문:
- null 포인터에 0을 더하는 표현을 굳이 피해야 하는 이유는 무엇입니까?
- allocator가 제공하는 `size_type`과 `difference_type`을 사용하는 이유는 무엇입니까?
- 빈 범위 `erase(first, last)`에서 `last - first`를 생략할 수 있는 조건은 무엇입니까?

### 30초 모범 답변

포인터 산술과 차이는 같은 실제 배열 객체 안의 포인터라는 전제가 필요하므로 null 저장소를 배열 시작처럼 취급하면 안 됩니다. 이 프로젝트는 `_iterator_at`과 `_index_of`에서 저장소 존재 여부를 먼저 확인해 빈 상태에서는 포인터를 그대로 반환하거나 인덱스 0을 사용했습니다. 빈 범위는 `first == last`를 먼저 처리해 차이 연산을 피합니다. 형식도 allocator의 `size_type`과 `difference_type`을 따라 공개 계약과 저장소 정책을 맞췄습니다.

### 답변 핵심 키워드

- null pointer arithmetic
- same-array precondition
- empty range fast path
- `_iterator_at`
- `_index_of`
- allocator typedef
- `begin == end`
- UB 경계

### 백지 구현

- **구현 목표**: 저장소가 없을 때 포인터 산술을 하지 않는 반복자·인덱스 보조 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: `iterator iterator_at(size_type)`, `size_type index_of(const_iterator)`, 빈 범위 `erase`.
- **입력과 출력**: 내부 포인터와 위치를 안전하게 상호 변환한다.
- **반드시 만족해야 할 조건**: 빈 상태의 `begin`, `end`, 0개 삽입, 빈 범위 삭제에서 산술·차이 연산이 발생하지 않는다.
- **경계 조건**: 기본 생성 직후, `clear` 뒤 남은 용량이 있는 상태, 완전히 저장소가 없는 상태.
- **실패 조건**: 다른 컨테이너의 반복자를 전달하는 경우는 사전 조건 위반으로 범위 밖이다.
- **필요한 제약**: 공개 반복자 형식은 allocator의 pointer 형식과 일치한다고 가정한다.

```cpp
template <class Alloc>
class empty_safe_positions
{
public:
    typedef typename Alloc::pointer iterator;
    typedef typename Alloc::const_pointer const_iterator;
    typedef typename Alloc::size_type size_type;

    iterator begin();
    iterator end();
    iterator erase(iterator first, iterator last);

private:
    iterator data_;
    size_type size_;

    iterator iterator_at(size_type index);
    size_type index_of(const_iterator position) const;
    // 직접 구현
};
```

### 구현 후 자가 검증

- 기본 생성 상태에서 `begin() == end()`이다.
- 빈 상태에서 `erase(begin(), end())`가 상태를 바꾸지 않는다.
- 0개 삽입의 위치 계산이 null 포인터 차이를 만들지 않는다.
- 저장소는 있지만 `size == 0`인 상태와 저장소가 없는 상태를 구분해도 계약은 동일하다.
- 비어 있지 않은 상태에서는 정상적인 인덱스 왕복이 된다.
- 반환 반복자가 삭제 뒤 새 논리 위치를 가리킨다.
- allocator의 크기·차이 형식을 사용한다.

### 구현 후 설명할 것

1. 주소값이 0이라는 사실과 유효한 배열 포인터라는 언어 규칙은 다르다는 점.
2. 빈 범위를 먼저 판별하면 불필요한 포인터 차이를 제거할 수 있는 이유.
3. 보조 함수로 경계를 한 곳에 모았을 때 얻는 검증 가능성.
4. raw pointer 반복자 설계의 단순함과 fancy pointer 미지원이라는 범위 제한.

### 원본 확인 위치

- **Thread**: 04, 05
- **커밋 메시지**: `fix(vector): allocator 형식과 빈 반복자 연산 보정`; `test(vector): 빈 저장소와 allocator 상태 검증`
- **파일**: `include/ft_vector.hpp`, `tests/test_vector_exceptions.cpp`
- **함수·클래스·컴포넌트**: `begin`, `end`, `insert`, `erase`, `_iterator_at`, `_index_of`, allocator의 `size_type`·`difference_type`
- **관련된 다른 Thread**: Thread 05의 빈 저장소 예외·수명 테스트
