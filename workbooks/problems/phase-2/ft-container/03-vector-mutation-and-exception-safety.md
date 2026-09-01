# Vector 변경·별칭·예외 안전성

변경 연산이 입력을 무효화하거나 객체 수명을 깨뜨리는 경로를 중심으로, 면접에서 다시 구현할 수 있는 크기로 축소했다.

<a id="vec-05"></a>
## VEC-05 — [Thread 05 / `fix(vector): 자기 범위 assign과 insert 입력 보존`] 자기 참조 범위와 값 별칭을 snapshot으로 분리

### 면접 질문

`values.assign(values.begin() + 1, values.end())`나 `values.insert(pos, values.begin(), values.end())`를 순진하게 앞에서부터 처리하면 왜 입력이 깨질 수 있습니까?  
또 `push_back(values[0])`가 재할당을 일으킬 때 참조 인자가 왜 먼저 복사되어야 합니까?

꼬리 질문:
- 입력 범위를 임시 컨테이너로 복사하는 해법의 시간·공간 비용은 무엇입니까?
- 겹치는 구간을 방향에 따라 직접 이동하는 해법과 비교해보세요.
- 자기 별칭 가능성을 포인터 범위 검사로만 판단하기 어려운 이유는 무엇입니까?

### 30초 모범 답변

수정 대상과 입력이 같은 저장소를 가리키면 삭제·이동·재할당 과정이 아직 읽지 않은 입력을 무효화할 수 있습니다. 그래서 범위 값이나 단일 별칭 값을 변경 전에 snapshot으로 분리한 뒤 그 독립된 입력으로 작업했습니다. 구현은 단순하고 예외 경계를 명확히 만들지만 범위 길이만큼 추가 공간과 복사가 듭니다. 직접 겹침을 분석하는 방식은 공간은 줄일 수 있어도 반복자 종류와 이동 방향, 예외 중간 상태가 훨씬 복잡해집니다.

### 답변 핵심 키워드

- self-aliasing
- input invalidation
- snapshot
- aliased reference
- reallocation
- overlapping range
- O(m) extra space
- 단순성-vs-최적화

### 백지 구현

- **구현 목표**: 입력 범위가 자기 컨테이너를 가리켜도 올바른 결과를 내는 범위 `assign` 또는 `insert`를 구현한다.
- **인터페이스 또는 함수 시그니처**: `template<class InputIt> void assign(InputIt first, InputIt last)` 또는 같은 형태의 범위 `insert`.
- **입력과 출력**: 외부 범위 또는 자기 범위 `[first, last)`를 받아 컨테이너 값을 갱신한다.
- **반드시 만족해야 할 조건**: 입력 원소를 읽기 전에 원본 저장소를 지우거나 재배치하지 않는다.
- **경계 조건**: 전체 자기 범위, 부분 자기 범위, 빈 범위, 삽입 위치가 입력 범위 안에 있는 경우.
- **실패 조건**: snapshot 구성 실패 시 기존 컨테이너가 바뀌지 않아야 한다.
- **필요한 제약**: snapshot에 O(m) 추가 공간을 사용하는 해법을 허용한다.

```cpp
template <class T, class Alloc>
class alias_safe_sequence
{
public:
    typedef T* iterator;

    template <class InputIt>
    void assign(InputIt first, InputIt last);

    template <class InputIt>
    void insert(iterator position, InputIt first, InputIt last);

private:
    // 저장소 표현은 제공되었다고 가정
    // 직접 구현
};
```

### 구현 후 자가 검증

- 전체 자기 범위 `assign(begin(), end())`가 값을 보존한다.
- 부분 자기 범위 assign 결과가 변경 전 입력 snapshot과 같다.
- 삽입 위치가 원본 입력 범위 앞·안·뒤에 있어도 기대 순서가 같다.
- 빈 범위는 아무 변화도 만들지 않는다.
- snapshot 복사 중 실패하면 원본 상태가 유지된다.
- `push_back(back())`가 재할당을 일으켜도 값이 보존된다.
- 추가 공간과 복사 횟수의 상한을 설명할 수 있다.

### 구현 후 설명할 것

1. 수정 순서가 입력 반복자의 유효성을 깨뜨리는 구체적인 경로.
2. 단일 값 별칭과 범위 별칭을 같은 '입력 보존' 문제로 볼 수 있는 이유.
3. snapshot 방식의 O(m) 비용을 받아들인 설계 이유.
4. 직접 겹침 처리로 최적화할 때 추가로 증명해야 할 invariant.

### 원본 확인 위치

- **Thread**: 05
- **커밋 메시지**: `fix(vector): 자기 범위 assign과 insert 입력 보존`; `test(vector): 자기 범위 변경 결과 검증`
- **파일**: `include/ft_vector.hpp`, `tests/test_containers.cpp`
- **함수·클래스·컴포넌트**: `assign(InputIt, InputIt)`, `insert(iterator, InputIt, InputIt)`, `push_back`, 임시 `vector` snapshot
- **관련된 다른 Thread**: Thread 04의 범위 API와 용량 확장, Thread 05의 트랜잭션·삽입 수명 보정

<a id="vec-06"></a>
## VEC-06 — [Thread 05 / `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리`] 생성·대입·크기 증가의 예외 보장

### 면접 질문

fill 생성자, `assign`, 복사 대입, `resize` 증가, 재할당이 필요한 `push_back`에서 예외 안전성을 각각 어떻게 확보했습니까?  
모든 연산에 같은 보장을 억지로 적용하지 말고, 어떤 연산은 임시 저장소 교체가 적합하고 어떤 연산은 생성한 접미부만 되돌리면 되는지 설명해보세요.

꼬리 질문:
- copy-and-swap과 '저장소만 교환'은 allocator 상태 때문에 어떻게 다를 수 있습니까?
- `_size`를 생성 전에 증가시키면 rollback이 왜 어려워집니까?
- 사용자 대입 연산이 던지는 제자리 변경에는 왜 강한 보장을 약속하기 어려울 수 있습니까?

### 30초 모범 답변

전체 내용을 바꾸는 생성·`assign`·복사 대입은 새 상태를 임시 객체나 새 블록에서 완성한 뒤 저장소를 교체해 강한 보장을 얻습니다. 기존 용량 안에서 `resize`로 접미부를 생성할 때는 이전 크기를 기록하고 성공한 새 원소만 역순 소멸하면 됩니다. 재할당 `push_back`은 인자가 기존 원소일 수 있으므로 값을 먼저 복사한 뒤 저장소를 늘립니다. 반면 기존 객체에 대한 사용자 대입이 중간에 실패하는 제자리 이동은 값 일부가 바뀔 수 있어 보장 수준을 명확히 제한해야 합니다.

### 답변 핵심 키워드

- strong guarantee
- temporary object
- storage-only swap
- old_size
- suffix rollback
- aliased `push_back`
- commit boundary
- basic guarantee

### 백지 구현

- **구현 목표**: 기존 값을 보존하는 강한 예외 보장의 fill `assign`을 구현한다.
- **인터페이스 또는 함수 시그니처**: `void assign(size_type count, const T& value)`.
- **입력과 출력**: 현재 내용을 `count`개의 `value`로 교체한다.
- **반드시 만족해야 할 조건**: 할당 또는 복사 생성 실패 시 원래 크기·용량·값이 모두 보존된다.
- **경계 조건**: `count == 0`, 기존 용량보다 작은·같은·큰 count, `value`가 현재 원소를 참조하는 경우.
- **실패 조건**: 임시 블록 할당 실패, 임의 위치의 복사 생성 실패.
- **필요한 제약**: allocator 객체 자체의 소유 정책은 바꾸지 않고 저장소 상태만 교체한다.

```cpp
template <class T, class Alloc>
class transactional_vector
{
public:
    typedef typename Alloc::size_type size_type;

    void assign(size_type count, const T& value);

private:
    Alloc alloc_;
    T* data_;
    size_type size_;
    size_type capacity_;

    void swap_storage(transactional_vector& other);
    // 직접 구현
};
```

### 구현 후 자가 검증

- 정상 경로에서 정확히 `count`개의 값이 생성된다.
- `count == 0`이 유효한 빈 결과를 만든다.
- 각 복사 실패 위치에서 기존 값과 메타데이터가 그대로다.
- 별칭 값 `assign(n, values[i])`의 입력을 변경 전에 보존한다.
- 임시 상태의 생성된 객체와 블록이 모두 정리된다.
- allocator 상태를 의도치 않게 교환하지 않는다.
- 성공 시 이전 저장소는 임시 객체의 소멸을 통해 한 번만 정리된다.

### 구현 후 설명할 것

1. 연산 전체를 임시 상태에서 완성하는 전략과 접미부 rollback 전략의 적용 범위.
2. allocator를 포함한 전체 `swap` 대신 저장소만 교환한 이유.
3. `size` 갱신을 생성 성공 뒤에 하는 규칙.
4. 강한 보장을 제공하지 않는 제자리 대입 경로를 문서화해야 하는 이유.

### 원본 확인 위치

- **Thread**: 05
- **커밋 메시지**: `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리`; `test(vector): 생성·대입·크기 변경 실패 주입`
- **파일**: `include/ft_vector.hpp`, `tests/test_vector_exceptions.cpp`
- **함수·클래스·컴포넌트**: `_initialize_fill`, `_swap_storage`, `assign`, 복사 대입, `resize`, `push_back`
- **관련된 다른 Thread**: Thread 04의 재할당, Thread 05의 별칭·삽입 객체 수명

<a id="vec-07"></a>
## VEC-07 — [Thread 05 / `fix(vector): fill·range 삽입의 객체 수명 보존`] 제자리 삽입에서 생성 영역과 대입 영역 분리

### 면접 질문

여유 용량이 있는 `vector::insert`에서 왜 모든 목적지 슬롯에 placement construction을 하면 안 됩니까?  
기존 객체가 있는 구간과 아직 객체가 없는 꼬리 구간을 어떻게 나눠 이동·생성해야 하며, 복사 생성과 대입 실패 뒤 무엇을 정리해야 합니까?

꼬리 질문:
- 삽입 개수가 뒤쪽 원소 수보다 큰 경우와 작은 경우의 영역 구성이 어떻게 달라집니까?
- 재할당 경로가 오히려 강한 보장을 만들기 쉬운 이유는 무엇입니까?
- 대입 실패 뒤 일부 기존 값이 바뀌었더라도 어떤 invariant는 반드시 유지해야 합니까?

### 30초 모범 답변

제자리 삽입의 목적지에는 이미 살아 있는 객체와 미생성 저장 공간이 섞여 있습니다. 미생성 꼬리에는 `construct`, 기존 객체 자리에는 대입을 써야 하며 같은 주소를 두 번 생성하거나 소멸하면 안 됩니다. 새로 생성한 꼬리 수를 추적해 복사 생성 실패 때만 그 부분을 되돌리고, `size`는 구조가 완성된 뒤 갱신합니다. 사용자 대입 실패는 일부 값 변경을 되돌리기 어려울 수 있지만 살아 있는 객체 수, 소유 블록, 후속 사용 가능성은 유지해야 합니다.

### 답변 핵심 키워드

- constructed vs uninitialized
- placement construction
- assignment
- tail construction count
- in-place insert
- basic guarantee
- double construction
- size commit

### 백지 구현

- **구현 목표**: 재할당 없이 fill 삽입하는 축소 함수를 구현하고 객체 수명 invariant를 유지한다.
- **인터페이스 또는 함수 시그니처**: `void insert_in_place(size_type index, size_type count, const T& value)`.
- **입력과 출력**: 여유 용량이 충분한 버퍼의 `index` 위치에 `count`개 값을 삽입한다.
- **반드시 만족해야 할 조건**: 기존 객체 주소에는 대입만, 미생성 꼬리 주소에는 생성만 수행한다.
- **경계 조건**: 맨 앞·맨 뒤 삽입, `count == 0`, `count`가 tail 길이보다 작은·같은·큰 경우, 별칭 값.
- **실패 조건**: 꼬리 복사 생성 실패, 기존 객체 대입 실패.
- **필요한 제약**: 복사 생성 실패에는 새 꼬리를 정리하고 기존 크기를 유지한다. 대입 실패에는 최소 기본 보장을 유지한다.

```cpp
template <class T, class Alloc>
class spare_capacity_buffer
{
public:
    typedef typename Alloc::size_type size_type;

    void insert_in_place(size_type index, size_type count, const T& value);

private:
    Alloc alloc_;
    T* data_;
    size_type size_;
    size_type capacity_;

    void destroy_constructed_tail(size_type start, size_type count);
    // 직접 구현
};
```

### 구현 후 자가 검증

- `count == 0`에서 어떤 생성·대입·크기 변경도 없다.
- 맨 뒤 삽입은 기존 원소 대입 없이 새 객체만 생성한다.
- 중간 삽입 뒤 값 순서와 크기가 정확하다.
- 어떤 주소에도 살아 있는 객체 위로 다시 placement construction하지 않는다.
- 복사 생성 실패 시 그 시도에서 새로 생성한 꼬리만 소멸한다.
- 실패 뒤 `size`와 실제 살아 있는 객체 수가 같다.
- 대입 실패 뒤에도 소멸·추가 삽입이 가능하고 리소스 누수가 없다.
- 별칭 `value`는 이동 전에 snapshot으로 보존된다.

### 구현 후 설명할 것

1. 삽입 대상 구간을 '이미 생성됨'과 '미생성'으로 나누는 기준.
2. `count`와 tail 길이의 관계에 따라 필요한 생성·대입 수가 달라지는 이유.
3. 재할당 경로와 제자리 경로의 예외 보장 차이.
4. 사용자 대입 실패에서 강한 보장 대신 기본 보장을 선택한 trade-off.

### 원본 확인 위치

- **Thread**: 05
- **커밋 메시지**: `fix(vector): fill·range 삽입의 객체 수명 보존`; `test(vector): 삽입 복사·대입·할당 실패 sweep`
- **파일**: `include/ft_vector.hpp`, `tests/test_vector_exceptions.cpp`
- **함수·클래스·컴포넌트**: `_insert_fill_reallocate`, `_insert_fill_in_place`, `_insert_range_reallocate`, `_insert_range_in_place`, `_destroy_constructed_tail`, `_replace_storage`
- **관련된 다른 Thread**: Thread 04의 중간 변경 연산, Thread 05의 snapshot·트랜잭션
