# Map 소유권 트랜잭션과 정책 객체

비교·할당·정책 객체가 모두 실패할 수 있을 때 트리 소유권을 언제 획득하고 언제 교환할지에 집중한다.

<a id="mtx-01"></a>
## MTX-01 — [Thread 09 / `fix(map): 삽입 위치를 노드 할당 전에 확정`] 비교 단계와 노드 소유권 획득을 분리한 삽입 트랜잭션

### 면접 질문

`map::insert`가 탐색 도중 노드를 먼저 할당한 뒤 마지막에 비교자를 다시 호출하면 어떤 예외 안전성 문제가 생깁니까?  
비교자가 던질 수 있다는 전제에서 중복 판정, 부모와 좌우 위치 결정, 노드 할당, 링크, 균형 보정의 순서를 설명해보세요.

꼬리 질문:
- 비교자 기반 동치는 어떤 두 호출로 판정합니까?
  - **모범답변:** `comp(new_key, current_key)`와 `comp(current_key, new_key)`를 호출합니다. 첫째가 참이면 왼쪽, 둘째가 참이면 오른쪽이며 둘 다 거짓이면 같은 동치류의 키입니다.
- 할당 전에 `parent`와 `insert_left`를 확정하면 소유권 경계가 어떻게 단순해집니까?
  - **모범답변:** 비교가 모두 끝날 때까지 새 자원이 없으므로 예외 시 정리할 것도 트리 변화도 없습니다. 이후 create 성공 노드는 저장한 parent와 방향에 바로 연결할 수 있어 할당 후 사용자 비교자를 다시 호출하지 않습니다.
- node 생성자가 던질 때와 비교자가 던질 때 각각 정리해야 할 자원은 무엇입니까?
  - **모범답변:** 비교 실패는 노드를 할당하기 전이라 새 자원이 없습니다. allocate 실패도 블록이 없고, construct 실패는 `create_node`가 방금 얻은 블록을 deallocate해야 하며 기존 트리는 어느 경우에도 그대로입니다.
- 링크가 끝난 뒤 레드-블랙 보정에서 비교자를 다시 호출할 필요가 없는 이유는 무엇입니까?
  - **모범답변:** 새 노드의 BST 위치는 이미 확정됐고 보정은 부모·조부모·삼촌의 링크와 색만 봅니다. 회전도 기존 중위 순서를 보존하므로 키 비교 없이 invariant를 복구할 수 있습니다.

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
    node* parent = NULL;
    node* current = root_;
    bool insert_left = false;

    // 던질 수 있는 비교는 ownership 획득 전에 모두 끝낸다.
    while (current)
    {
        parent = current;
        if (comp_(value.first, current->value.first))
        {
            insert_left = true;
            current = current->left;
        }
        else if (comp_(current->value.first, value.first))
        {
            insert_left = false;
            current = current->right;
        }
        else
        {
            insert_result duplicate = { current, false };
            return duplicate;
        }
    }

    node* created = create_node(value);
    link_and_fix(parent, created, insert_left);
    ++size_;
    insert_result success = { created, true };
    return success;
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
   - **모범답변:** 비교 단계는 트리를 읽기만 하므로 어느 호출에서 던져도 상태가 그대로입니다. 최종 위치가 확정된 뒤 노드를 생성하고 링크하는 단일 commit으로 만들면 예외가 소유권을 가진 미연결 노드를 남기는 구간을 최소화합니다.
2. 중복 판정에 `==`가 아니라 비교자 양방향을 사용하는 이유
   - **모범답변:** map의 key 유일성은 comparator가 만든 동치류를 따릅니다. 두 방향 비교가 모두 거짓인지를 사용해야 삽입과 find·bound가 같은 키 의미를 공유합니다.
3. 할당 전에 좌우 방향을 저장해 마지막 비교 호출을 제거한 판단
   - **모범답변:** 탐색 루프의 마지막 비교 결과를 `insert_left`에 보존하면 create 뒤 parent에 연결할 방향이 이미 정해집니다. 던질 수 있는 comparator를 ownership 획득 뒤 다시 호출하지 않아 생성된 노드의 정리 책임이 애매해지지 않습니다.
4. 노드 생성 helper가 할당·생성 rollback을 자체 책임지는 경계
   - **모범답변:** helper는 완성된 node를 반환하거나 아무 자원도 남기지 않는 계약을 가집니다. allocate 뒤 construct가 실패하면 같은 node allocator로 블록을 반환하므로 insert는 미완성 자원을 알 필요가 없습니다.
5. 균형 보정이 키 비교 없이 링크와 색만 다루는 구조적 분리
   - **모범답변:** BST 순서는 탐색과 최초 링크가 확정하고 회전은 중위 순서를 보존합니다. fixup은 색과 친족 관계만 다뤄 comparator 예외 경계와 균형 invariant 복구를 서로 독립적으로 검증할 수 있습니다.

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
  - **모범답변:** map 객체 자체의 소멸자는 호출되지 않습니다. 이미 완성된 멤버의 소멸자는 처리되지만, raw node 링크처럼 본문에서 직접 얻은 자원은 생성자 catch가 명시적으로 정리해야 합니다.
- 부분 생성된 트리를 누가 `clear`해야 합니까?
  - **모범답변:** 범위·복사 생성자의 try/catch가 그 객체에 이미 연결된 노드를 `clear`한 뒤 예외를 다시 던집니다. 원본이 두 생성자 모두에서 이 패턴을 사용합니다.
- copy-and-swap와 달리 allocator를 임시 객체와 함께 교환하지 않는 이유는 무엇입니까?
  - **모범답변:** 복사 대입 결과의 노드는 target allocator가 소유하도록 임시 트리도 target allocator로 만듭니다. commit에서 트리·size·comparator만 교환하면 target의 allocator identity를 보존하고 이전 노드도 같은 allocator를 가진 임시 객체가 해제합니다.
- 임시 트리 구축과 최종 정책·소유권 교환 중 각각 어떤 연산이 던질 수 있습니까?
  - **모범답변:** 구축 중에는 comparator 호출, node allocate, value construct가 던질 수 있습니다. commit에서는 comparator 교환이 던질 수 있으며, 원본이 검증한 모델은 상태 변경 전 throw로 제한되고 그 뒤 포인터·size 교환은 비예외라고 가정합니다.

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
    : alloc_(alloc), node_alloc_(node_allocator(alloc)),
      header_(true), size_(0), comp_(comp)
{
    try
    {
        insert(first, last);
    }
    catch (...)
    {
        // 완성되지 않은 map의 소멸자는 호출되지 않으므로 직접 회수한다.
        clear();
        throw;
    }
}

map& operator=(const map& other)
{
    if (this != &other)
    {
        // source 정책과 target allocator로 임시 트리를 완성한다.
        map temporary(other.begin(), other.end(), other.comp_, alloc_);
        swap_tree_and_compare(temporary);
    }
    return *this;
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
   - **모범답변:** 생성자 본문이 던지면 map의 lifetime이 시작되지 않아 `~map()`이 호출되지 않습니다. 본문에서 insert로 연결한 raw node는 멤버 객체의 자동 소멸만으로 없어지지 않으므로 catch에서 clear해야 합니다.
2. 기존 target 파괴를 미루어 임시 구축 단계에 강한 보장을 주는 방식
   - **모범답변:** source의 모든 원소를 임시 map에 넣는 동안 target은 읽지도 수정하지도 않습니다. 비교·할당·복사 중 실패하면 임시만 소멸하고, 전체 성공 뒤에만 storage ownership을 교환해 target을 새 상태로 만듭니다.
3. 임시 트리를 source allocator가 아니라 target allocator로 만드는 소유권 판단
   - **모범답변:** 원본 복사 대입은 allocator를 전파하지 않고 target의 기존 allocator 정책을 유지합니다. 따라서 새 노드는 처음부터 `_alloc`로 만들고, commit 뒤 target이 같은 owner로 해제할 수 있게 합니다.
4. allocator를 교환하지 않고 트리·크기·비교자만 commit하는 이유
   - **모범답변:** 새 트리와 이전 트리 모두 target allocator 계열로 만들어져 있으므로 allocator 교환이 필요 없습니다. `_swap_tree_and_compare`는 논리 정렬 상태인 comparator와 tree·size만 바꾸고 양 header를 재연결합니다.
5. 원소별 삽입을 재사용하는 단순성과 O(k log k) 비용의 trade-off
   - **모범답변:** 기존 insert의 중복 판정, 노드 rollback, 균형 보정을 그대로 재사용해 복사 경로가 단순합니다. 반면 정렬된 입력을 선형 시간에 복제하는 전용 tree clone보다 k개 원소에 O(k log k) 비교가 듭니다.

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
  - **모범답변:** 각 노드의 왼쪽·오른쪽 배치는 삽입 당시 comparator의 순서로 결정됩니다. tree만 옮기거나 comparator만 바꾸면 find와 insert가 실제 링크 구조와 다른 방향으로 내려갑니다.
- 비교자 교환을 먼저 시도하면 allocator·루트·size는 실패 시 어떻게 남아야 합니까?
  - **모범답변:** 프로젝트의 상태 변경 전 throw 모델에서는 비교자 단계가 실패하면 이후 교환이 실행되지 않아야 합니다. 양쪽 allocator와 node allocator, root 포인터, size, header 링크, outstanding node 수가 모두 호출 전 그대로 남습니다.
- 이 순서만으로 임의의 부분 변경 후 예외를 던지는 comparator까지 강한 보장을 할 수 있습니까?
  - **모범답변:** 할 수 없습니다. `std::swap` 내부 첫 대입이 성공하고 두 번째가 실패하거나 comparator가 자기 상태 일부를 바꾼 뒤 던지면 정책만 부분 변경될 수 있습니다. 원본 테스트가 보장한 더 좁은 실패 모델을 명시해야 합니다.
- 비교자 교환 성공 뒤 수행하는 포인터·size·allocator 교환은 어떤 예외 가정이 필요합니까?
  - **모범답변:** allocator와 node allocator의 swap, root 포인터와 size 교환이 던지지 않는다고 가정해야 합니다. 그래야 comparator 성공 이후 ownership commit이 중간 실패 없이 끝납니다.

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
        // 검증 범위에서는 comparator가 상태 변경 전에 던진다.
        std::swap(comp_, other.comp_);

        // 정책 교환이 끝난 뒤에만 실제 node ownership을 이동한다.
        std::swap(alloc_, other.alloc_);
        std::swap(node_alloc_, other.node_alloc_);
        std::swap(header_.parent, other.header_.parent);
        std::swap(size_, other.size_);
        refresh_header();
        other.refresh_header();
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
   - **모범답변:** tree 링크는 comparator가 정의한 순서의 물리적 표현입니다. 둘이 불일치하면 공개 순회는 한 순서인데 검색과 삽입은 다른 순서를 가정하므로 컨테이너가 더 이상 자신의 key ordering 계약을 만족하지 못합니다.
2. 비교자 교환을 소유권 이동보다 먼저 둔 commit 순서
   - **모범답변:** 프로젝트에서 실패를 주입한 comparator 대입이 유일한 throw 지점이므로 이를 먼저 끝냅니다. 여기서 실패하면 tree·allocator는 그대로이고, 성공 뒤 비예외 ownership 교환을 연속 수행해 논리 상태를 완성합니다.
3. 실패 시 allocator owner와 outstanding node 수까지 보존해야 하는 이유
   - **모범답변:** 공개 키 순서만 같아도 node를 다른 stateful allocator가 해제하면 ownership 오류입니다. 실패한 swap은 각 root뿐 아니라 그 노드를 해제할 allocator identity와 owner별 outstanding block 회계도 함께 보존해야 합니다.
4. 프로젝트가 검증한 '상태 변경 전 throw' 모델과 일반적인 부분 변경 throw의 차이
   - **모범답변:** 원본 `throwing_compare`는 대입 카운터를 올린 뒤 실제 `_control`·`_reverse`를 바꾸기 전에 던집니다. 일반 comparator는 일부 필드를 수정한 뒤 던질 수 있어 같은 순서만으로 비교 정책의 강한 보장을 얻지 못합니다.
5. header 재연결 비용을 지불하면서 반복자·종단 invariant를 복구한 trade-off
   - **모범답변:** swap한 root의 parent를 새 header로 잇고 min/max를 다시 계산하면 기존 원소 반복자가 새 소유 map의 end까지 올바르게 순회합니다. 대신 refresh 두 번 때문에 비용이 O(h₁+h₂)가 됩니다.

### 원본 확인 위치

- **Thread**: 09
- **커밋 메시지**: `fix(map): 비교자 교환 실패 전에 tree 소유권 유지`
- **파일**: `include/ft_map.hpp`, `tests/test_map_policy_exceptions.cpp`
- **함수·컴포넌트**: `swap`, `_swap_tree_and_compare`, `_refresh_header`, `throwing_compare`, `test_copy_assignment_keeps_target_ownership`, `test_public_swap_keeps_both_owners`
- **관련된 다른 Thread**: 08의 header·swap 반복자 안정성, 09의 임시 트리 복사 대입
