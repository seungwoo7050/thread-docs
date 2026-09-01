# Map 소유권 트랜잭션과 정책 객체

비교·할당·정책 객체가 모두 실패할 수 있을 때 트리 소유권을 언제 획득하고 언제 교환할지에 집중한다.

<a id="mtx-01"></a>
## MTX-01 — [Thread 09 / `fix(map): 삽입 위치를 노드 할당 전에 확정`] 비교 단계와 노드 소유권 획득을 분리한 삽입 트랜잭션

### 면접 질문

`map::insert`가 탐색 도중 노드를 먼저 할당한 뒤 마지막에 비교자를 다시 호출하면 어떤 예외 안전성 문제가 생깁니까?  
비교자가 던질 수 있다는 전제에서 중복 판정, 부모와 좌우 위치 결정, 노드 할당, 링크, 균형 보정의 순서를 설명해보세요.

꼬리 질문:
- 비교자 기반 동치는 어떤 두 호출로 판정합니까?
- 할당 전에 `parent`와 `insert_left`를 확정하면 소유권 경계가 어떻게 단순해집니까?
- node 생성자가 던질 때와 비교자가 던질 때 각각 정리해야 할 자원은 무엇입니까?
- 링크가 끝난 뒤 레드-블랙 보정에서 비교자를 다시 호출할 필요가 없는 이유는 무엇입니까?

### 30초 모범 답변

탐색 중에는 비교자가 언제든 던질 수 있으므로 아직 자원을 획득하지 않는 편이 안전합니다. 먼저 루트에서 내려가며 중복 여부, 최종 부모, 왼쪽·오른쪽 연결 방향을 모두 확정합니다. 그다음 노드를 할당·생성하고, 성공한 노드만 한 번 링크한 뒤 색 보정을 수행합니다. 이렇게 하면 비교 실패 때는 트리가 전혀 바뀌지 않고 해제할 노드도 없으며, 생성 실패 때는 생성 함수가 자기 블록만 회수하면 됩니다. 할당 뒤에는 비교자를 다시 호출하지 않아 소유권이 공중에 뜨는 구간을 없앱니다.

### 답변 핵심 키워드

- compare before allocate
- duplicate detection
- parent and side
- ownership acquisition
- no comparator after allocation
- construction rollback
- single commit point
- tree unchanged on failure

### 백지 구현

- **구현 목표**: 던질 수 있는 비교자와 allocator를 사용하는 단순 BST의 unique insert를 강한 실패 경계로 구현한다.
- **인터페이스 또는 함수 시그니처**: `pair<node*, bool> insert_unique(const value_type& value)`를 구현하고 `create_node`는 제공된 것으로 가정한다.
- **입력과 출력**: 키-값을 받아 새 노드와 성공 여부를 반환하며, 동치 키가 있으면 기존 노드와 false를 반환한다.
- **반드시 만족해야 할 조건**: 모든 비교와 삽입 위치 결정은 노드 할당 전에 끝나고, 할당 뒤에는 비교자를 호출하지 않는다.
- **경계 조건**: 빈 트리, 루트보다 작은·큰 키, 깊은 경로, 동치 키를 처리한다.
- **실패 조건**: 비교 실패 시 트리·크기·할당 수가 그대로이며, 할당·생성 실패 시 새 노드가 링크되지 않고 누수도 없다.
- **필요한 제약**: 색 보정은 제공되며, unique BST 삽입의 탐색·commit 경계만 15~25분 안에 구현한다.

```cpp
struct insert_result
{
    node* position;
    bool inserted;
};

insert_result insert_unique(const value_type& value)
{
    // 직접 구현
}

node* create_node(const value_type& value);
void link_and_fix(node* parent, node* created, bool insert_left);
```

### 구현 후 자가 검증

- 빈 트리에 첫 노드를 넣을 때 크기가 정확히 하나 증가하는가?
- 동치 키 삽입은 기존 노드를 반환하고 할당·크기 변화를 만들지 않는가?
- 각 탐색 단계에서 비교자의 두 방향 호출로 동치를 판정하는가?
- 비교자가 어느 호출에서 던져도 도달 가능한 노드 집합과 중위 순서가 그대로인가?
- 비교 실패 시 allocator의 outstanding block 수가 증가하지 않는가?
- 노드 생성이 던지면 확보한 블록이 회수되고 트리에 링크되지 않는가?
- 노드 생성 성공 뒤 비교자가 한 번도 호출되지 않는가?
- 링크 후 부모·자식 관계와 `size`가 일치하는가?
- 탐색과 보정의 전체 시간 복잡도가 O(log n)인가?

### 구현 후 설명할 것

1. 비교를 순수한 준비 단계, 링크를 commit 단계로 분리한 이유
2. 중복 판정에 `==`가 아니라 비교자 양방향을 사용하는 이유
3. 할당 전에 좌우 방향을 저장해 마지막 비교 호출을 제거한 판단
4. 노드 생성 helper가 할당·생성 rollback을 자체 책임지는 경계
5. 균형 보정이 키 비교 없이 링크와 색만 다루는 구조적 분리

### 원본 확인 위치

- **Thread**: 09
- **커밋 메시지**: `fix(map): 삽입 위치를 노드 할당 전에 확정`
- **파일**: `include/ft_map.hpp`, `tests/test_map_exceptions.cpp`
- **함수·컴포넌트**: `insert`, `_create_node`, `_insert_fixup`, `insert_left`, `test_insert_does_not_compare_after_allocation`
- **관련된 다른 Thread**: 06의 비교자 기반 삽입, 07의 삽입 보정, 09의 노드 allocator 상태

<a id="mtx-02"></a>
## MTX-02 — [Thread 09 / `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리`] 생성 rollback과 복사 대입의 임시 트리 commit

### 면접 질문

범위 생성자나 복사 생성자에서 여러 노드를 순차 삽입하다 중간에 비교·할당 예외가 나면 왜 소멸자가 자동으로 호출되지 않는다고 봐야 합니까?  
복사 대입에서 기존 target을 먼저 지우지 않고 target allocator로 임시 트리를 만든 뒤 commit하는 이유를 설명해보세요.

꼬리 질문:
- 생성자 본문에서 예외가 나면 완성되지 않은 객체의 소멸자는 호출됩니까?
- 부분 생성된 트리를 누가 `clear`해야 합니까?
- copy-and-swap와 달리 allocator를 임시 객체와 함께 교환하지 않는 이유는 무엇입니까?
- 임시 트리 구축과 최종 정책·소유권 교환 중 각각 어떤 연산이 던질 수 있습니까?

### 30초 모범 답변

생성자 본문이 실패하면 완성되지 않은 `map`의 소멸자는 호출되지 않으므로, 본문에서 이미 연결한 노드를 catch해 직접 `clear`한 뒤 다시 던져야 합니다. 복사 대입은 기존 target을 먼저 지우면 이후 비교나 할당 실패 때 원래 값을 잃습니다. 그래서 target의 allocator와 source의 비교자를 사용해 임시 트리를 완성한 뒤에만 트리·크기·비교자 상태를 교환합니다. 임시 구축이 실패하면 임시 객체만 정리되고 target은 그대로이며, allocator 소유권은 target 쪽에 남습니다. 최종 비교자 교환까지 같은 보장을 주려면 MTX-03에서 다루는 정책 객체의 예외 계약이 추가로 필요합니다.

### 답변 핵심 키워드

- constructor body failure
- destructor not called for incomplete object
- partial tree cleanup
- temporary tree
- temporary-build strong guarantee
- target allocator
- commit after full construction
- allocator ownership retained

### 백지 구현

- **구현 목표**: 부분 생성 노드를 회수하는 범위 생성자와, 기존 값을 보존하는 복사 대입 연산자를 구현한다.
- **인터페이스 또는 함수 시그니처**: 범위 생성자와 `map& operator=(const map& other)`를 구현하며 `insert`, `clear`, `swap_tree_and_compare`는 사용할 수 있다.
- **입력과 출력**: 입력 범위 또는 source map의 모든 원소를 복사해 새 객체나 target 결과를 만든다.
- **반드시 만족해야 할 조건**: 성공 시 source와 같은 순서·값·비교 정책을 가진다. 임시 트리 구축 중 실패하면 새로 얻은 모든 노드를 해제하고 기존 target을 보존한다. 최종 비교자 교환 실패의 보장은 '상태 변경 전 throw' 정책으로 제한한다.
- **경계 조건**: 빈 범위, 자기 대입, source와 target의 allocator 상태가 다른 경우를 처리한다.
- **실패 조건**: 비교자 호출, node allocation, node value construction에서 실패를 주입한다. 최종 비교자 교환 실패는 MTX-03의 제한된 정책 모델을 적용한다.
- **필요한 제약**: 노드별 deep copy 최적화는 제외하고 범위 삽입을 이용하며 20~30분 안에 구현한다.

```cpp
template <class InputIt>
map(InputIt first, InputIt last,
    const key_compare& comp,
    const allocator_type& alloc)
{
    // 직접 구현
}

map& operator=(const map& other)
{
    // 직접 구현
}

void clear();
void swap_tree_and_compare(map& other);
```

### 구현 후 자가 검증

- 빈 범위 생성자가 빈 header·size 상태를 만드는가?
- 첫 번째, 중간, 마지막 원소 삽입에서 각각 실패를 주입해도 outstanding node가 0으로 돌아오는가?
- 복사 생성 실패 뒤 source의 키 순서와 노드 수가 그대로인가?
- 복사 대입의 임시 구축 실패 뒤 target의 기존 키·비교 정책·allocator가 그대로인가?
- 최종 비교자 교환 실패를 시험한다면 comparator가 상태 변경 전에 던진다는 제약을 명시했는가?
- 자기 대입이 값과 자원을 바꾸지 않는가?
- 성공한 대입 뒤 target allocator가 원래 target allocator인지 확인했는가?
- 임시 객체가 target의 이전 노드를 넘겨받은 뒤 정상 소멸하는가?
- header root/min/max와 `size`가 commit 뒤 새 트리에 맞는가?
- 범위 복사의 시간 복잡도를 원소 수 k에 대해 O(k log k)로 설명할 수 있는가?

### 구현 후 설명할 것

1. 완성되지 않은 객체의 소멸자에 의존할 수 없어 생성자 catch가 필요한 이유
2. 기존 target 파괴를 미루어 임시 구축 단계에 강한 보장을 주는 방식
3. 임시 트리를 source allocator가 아니라 target allocator로 만드는 소유권 판단
4. allocator를 교환하지 않고 트리·크기·비교자만 commit하는 이유
5. 원소별 삽입을 재사용하는 단순성과 O(k log k) 비용의 trade-off

### 원본 확인 위치

- **Thread**: 09
- **커밋 메시지**: `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리`
- **파일**: `include/ft_map.hpp`, `tests/test_map_exceptions.cpp`, `tests/test_map_policy_exceptions.cpp`
- **함수·컴포넌트**: 범위 생성자, 복사 생성자, `operator=`, `clear`, `_swap_tree_and_compare`, `test_range_constructor_rollback`, `test_copy_constructor_rollback`, `test_assignment_preserves_original`, `test_copy_assignment_keeps_target_ownership`
- **관련된 다른 Thread**: 09의 삽입 트랜잭션·비교자 교환 순서, 08의 header 갱신

<a id="mtx-03"></a>
## MTX-03 — [Thread 09 / `fix(map): 비교자 교환 실패 전에 tree 소유권 유지`] 정책 객체 예외와 트리 소유권 교환 순서

### 면접 질문

stateful comparator를 가진 두 `map`의 트리 소유권을 먼저 바꾼 뒤 비교자를 교환하면 어떤 불일치가 생길 수 있습니까?  
비교자 대입이 상태를 바꾸기 전에 예외를 던지는 프로젝트의 실패 모델에서, 공개 `swap`과 복사 대입 commit의 순서를 어떻게 잡아야 합니까?

꼬리 질문:
- 트리와 비교자는 왜 하나의 논리 상태 쌍입니까?
- 비교자 교환을 먼저 시도하면 allocator·루트·size는 실패 시 어떻게 남아야 합니까?
- 이 순서만으로 임의의 부분 변경 후 예외를 던지는 comparator까지 강한 보장을 할 수 있습니까?
- 비교자 교환 성공 뒤 수행하는 포인터·size·allocator 교환은 어떤 예외 가정이 필요합니까?

### 30초 모범 답변

트리의 정렬 구조는 그 트리를 만든 비교자의 의미와 결합되어 있으므로 둘 중 하나만 바뀌면 검색 계약이 깨집니다. 이 프로젝트는 비교자 대입이 자기 상태를 바꾸기 전에 던지는 실패 모델을 테스트했고, 비교자 교환을 가장 먼저 시도한 뒤에 allocator와 트리 소유권을 옮겼습니다. 그러면 비교자 단계에서 실패할 때 두 트리와 allocator 소유자는 그대로입니다. 다만 비교자가 일부 상태를 바꾼 뒤 던지거나 교환의 두 번째 대입에서 실패하는 임의의 타입까지 자동으로 강한 보장을 얻는 것은 아니며, 보장 범위를 정책 객체의 예외 계약과 함께 명시해야 합니다.

### 답변 핵심 키워드

- policy object
- tree-comparator consistency
- throw before mutation
- policy first
- ownership commit
- allocator owner
- narrow exception guarantee
- partial mutation caveat

### 백지 구현

- **구현 목표**: 비교자 교환이 상태 변경 전에 실패할 수 있는 모델에서 두 컨테이너의 트리·allocator 소유권을 보존하는 `swap` commit을 구현한다.
- **인터페이스 또는 함수 시그니처**: `void swap(map& other)`와 내부 `swap_tree_and_compare`의 핵심 순서를 구현한다.
- **입력과 출력**: 서로 다른 비교 방향과 allocator 소유자를 가진 두 컨테이너를 성공 시 완전히 교환한다.
- **반드시 만족해야 할 조건**: 비교자 교환이 시작 단계에서 실패하면 루트·size·allocator·노드 소유권이 양쪽 모두 그대로여야 한다.
- **경계 조건**: 한쪽 또는 양쪽이 빈 경우, 서로 다른 allocator 상태, 오름차순·내림차순 comparator를 처리한다.
- **실패 조건**: 비교자 교환만 실패를 주입한다. 정책이 부분 변경 후 던지는 더 강한 실패 모델은 요구 범위 밖이며 명시해야 한다.
- **필요한 제약**: 비교자 단계 성공 뒤 포인터·size·allocator 교환은 비예외 연산이라고 가정하고 10~20분 안에 작성한다.

```cpp
class map
{
public:
    void swap(map& other)
    {
        // 직접 구현
    }

private:
    key_compare comp_;
    allocator_type alloc_;
    node_allocator node_alloc_;
    node_base header_;
    size_type size_;

    void refresh_header();
};
```

### 구현 후 자가 검증

- 서로 다른 비교 방향을 가진 두 map의 정상 교환 뒤 각 트리가 새 비교자로 검색 가능한가?
- 비교자 대입 첫 단계에서 예외를 주입하면 양쪽 키 순서와 `size`가 그대로인가?
- 실패 뒤 각 map의 `get_allocator()`가 원래 소유자를 가리키는가?
- 실패 뒤 allocator별 outstanding block 수가 각각 원래 값인가?
- 실패한 컨테이너에 새 원소를 삽입·검색해 비교 정책과 트리가 계속 일치하는가?
- 성공 뒤 각 루트의 parent와 header의 root/min/max가 새 소유자에 맞게 갱신되는가?
- 빈 map과 비어 있지 않은 map의 성공·실패 경로가 모두 정상인가?
- 정책 객체가 일부 변경 후 던지는 경우를 보장한다고 과장하지 않았는가?
- `swap` 비용이 header 극값 재계산 때문에 O(log n + log m)임을 설명할 수 있는가?

### 구현 후 설명할 것

1. 트리 구조와 comparator를 분리할 수 없는 논리 상태로 본 이유
2. 비교자 교환을 소유권 이동보다 먼저 둔 commit 순서
3. 실패 시 allocator owner와 outstanding node 수까지 보존해야 하는 이유
4. 프로젝트가 검증한 '상태 변경 전 throw' 모델과 일반적인 부분 변경 throw의 차이
5. header 재연결 비용을 지불하면서 반복자·종단 invariant를 복구한 trade-off

### 원본 확인 위치

- **Thread**: 09
- **커밋 메시지**: `fix(map): 비교자 교환 실패 전에 tree 소유권 유지`
- **파일**: `include/ft_map.hpp`, `tests/test_map_policy_exceptions.cpp`
- **함수·컴포넌트**: `swap`, `_swap_tree_and_compare`, `_refresh_header`, `throwing_compare`, `test_copy_assignment_keeps_target_ownership`, `test_public_swap_keeps_both_owners`
- **관련된 다른 Thread**: 08의 header·swap 반복자 안정성, 09의 임시 트리 복사 대입
