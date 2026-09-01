# Map 정렬·노드 수명·기본 삭제

비교자 기반 동치, 노드 allocator 소유권, 균형 보정 전 단계의 BST 삭제를 분리해 연습한다.

<a id="map-01"></a>
## MAP-01 — [Thread 09 / `fix(map): 값 allocator 상태로 노드 allocator 구성`] allocator rebind와 노드 소유권

### 면접 질문

`map`의 공개 allocator는 `value_type`용인데 실제 할당 단위는 내부 node입니다.  
`rebind`로 만든 node allocator가 원래 allocator의 상태를 이어받아야 하는 이유와, 노드 생성 중 값 복사가 실패할 때의 정리 순서를 설명해보세요.

꼬리 질문:
- 기본 생성한 node allocator를 쓰면 stateful allocator에서 어떤 버그가 생깁니까?
  - **모범답변:** value allocator가 가진 풀 식별자·소유자·계측 포인터가 node allocator로 전달되지 않습니다. 그러면 다른 풀에서 할당하거나, 할당한 owner와 해제한 owner가 달라지고 outstanding block 회계도 틀어질 수 있습니다.
- `_create_node`가 allocate와 construct를 분리해야 하는 이유는 무엇입니까?
  - **모범답변:** allocate 성공 뒤 construct가 값 복사 때문에 실패할 수 있어 '블록은 있지만 객체는 없는' 중간 상태가 생깁니다. 두 단계를 분리해야 catch에서 destroy 없이 해당 블록만 deallocate하는 정확한 rollback을 할 수 있습니다.
- `clear`와 소멸자에서 노드 수와 outstanding block 수가 어떻게 대응해야 합니까?
  - **모범답변:** 성공적으로 생성된 값 노드 하나마다 outstanding block 하나가 있고, clear는 각 노드를 정확히 한 번 destroy/deallocate해야 합니다. 전체 clear나 소멸 뒤에는 도달 노드 수와 size가 0이고 해당 allocator owner의 outstanding block도 0이어야 합니다.

### 30초 모범 답변

`map`은 `value_type` allocator를 node 형식으로 rebind해서 사용하지만 소유자·풀·계측 상태는 원래 allocator와 같아야 합니다. 그래서 node allocator를 기본 생성하지 않고 전달받은 value allocator로부터 구성해야 합니다. 노드 생성은 먼저 한 블록을 할당하고 값을 포함한 node를 생성하며, 생성이 던지면 아직 노드 객체는 없으므로 블록만 해제합니다. 생성에 성공한 노드는 이후 정확히 한 번 destroy와 deallocate를 거칩니다.

### 답변 핵심 키워드

- allocator rebind
- stateful allocator
- owner identity
- allocate vs construct
- construction rollback
- node ownership
- destroy/deallocate
- outstanding blocks

### 백지 구현

- **구현 목표**: value allocator의 상태를 보존하는 node allocator와 안전한 노드 생성·파괴 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: 생성자, `create_node`, `destroy_node`, `clear_subtree`.
- **입력과 출력**: 키-값 쌍을 받아 부모·자식 포인터가 초기화된 노드를 반환한다.
- **반드시 만족해야 할 조건**: node allocator는 전달된 allocator에서 구성되고, 모든 성공 노드는 같은 소유 상태로 해제된다.
- **경계 조건**: 첫 노드, 빈 트리 clear, 한쪽 자식만 있는 트리, 깊은 트리.
- **실패 조건**: node 할당 실패, node/value 생성 실패.
- **필요한 제약**: C++98 allocator `rebind`를 사용하고 현대 allocator traits에 의존하지 않는다.

```cpp
template <class Value, class Alloc>
class node_store
{
    struct node
    {
        Value value;
        node* parent;
        node* left;
        node* right;

        explicit node(const Value& v)
            : value(v), parent(NULL), left(NULL), right(NULL) {}
    };

    typedef typename Alloc::template rebind<node>::other node_allocator;

public:
    explicit node_store(const Alloc& alloc);

private:
    Alloc value_alloc_;
    node_allocator node_alloc_;

    node* create_node(const Value& value);
    void destroy_node(node* current);
    void clear_subtree(node* current);
};

template <class Value, class Alloc>
node_store<Value, Alloc>::node_store(const Alloc& alloc)
    : value_alloc_(alloc), node_alloc_(node_allocator(alloc))
{
}

template <class Value, class Alloc>
typename node_store<Value, Alloc>::node*
node_store<Value, Alloc>::create_node(const Value& value)
{
    node* created = node_alloc_.allocate(1);
    try
    {
        node_alloc_.construct(created, node(value));
    }
    catch (...)
    {
        // construct가 실패했으므로 객체는 없고 원시 블록만 반환한다.
        node_alloc_.deallocate(created, 1);
        throw;
    }
    return created;
}

template <class Value, class Alloc>
void node_store<Value, Alloc>::destroy_node(node* current)
{
    node_alloc_.destroy(current);
    node_alloc_.deallocate(current, 1);
}

template <class Value, class Alloc>
void node_store<Value, Alloc>::clear_subtree(node* current)
{
    if (current == NULL)
        return;
    clear_subtree(current->left);
    clear_subtree(current->right);
    destroy_node(current);
}
```

### 구현 후 자가 검증

- stateful allocator의 식별 상태가 node allocator에도 전달된다.
- 할당 실패 시 outstanding block이 늘지 않는다.
- node 생성 실패 시 방금 할당한 블록이 해제된다.
- 성공 노드의 부모·왼쪽·오른쪽 포인터가 정의된 초기 상태다.
- 빈 트리 clear는 아무 작업도 하지 않는다.
- 모든 노드를 clear한 뒤 할당 블록 수가 0이다.
- 서로 다른 allocator 소유자가 상대 노드를 해제하지 않는다.

### 구현 후 설명할 것

1. 공개 value allocator와 내부 node allocator가 형식은 달라도 소유 상태를 공유해야 하는 이유.
   - **모범답변:** rebind는 할당 대상 형식만 Value에서 node로 바꿉니다. 메모리 풀이나 owner identity 같은 런타임 상태는 같은 컨테이너 소유권의 일부이므로 원본처럼 `node_allocator(alloc)`로 이어받아야 같은 주체가 노드를 해제합니다.
2. 할당된 메모리와 생성된 객체를 별도 상태로 보는 관점.
   - **모범답변:** allocate 뒤 construct 전에는 node를 담을 공간만 있고 node 객체나 value의 수명은 시작되지 않았습니다. construct 성공 뒤에만 destroy 대상이 되며, 그 전 실패는 deallocate만 해야 합니다.
3. 노드 생성 함수에 rollback을 캡슐화했을 때 상위 삽입 코드가 단순해지는 이유.
   - **모범답변:** `create_node`가 성공하면 완성된 노드 ownership을 반환하고, 실패하면 자원을 남기지 않는 계약이 됩니다. 삽입 코드는 비교로 위치를 정한 뒤 반환 노드를 연결하는 일만 하면 되어 모든 호출부에 allocate/construct catch를 반복하지 않습니다.
4. 재귀 clear의 단순함과 트리 높이가 비정상적으로 커질 때의 호출 스택 trade-off.
   - **모범답변:** 후위 재귀는 자식을 먼저 지운다는 수명 순서를 코드로 직접 표현해 단순합니다. 다만 호출 깊이는 트리 높이이므로 균형 invariant가 깨져 선형 높이가 되면 스택 사용도 O(n)이 되며, 그런 환경에서는 명시적 스택이나 반복 삭제를 고려할 수 있습니다.

### 원본 확인 위치

- **Thread**: 06, 09
- **커밋 메시지**: `feat(map): 노드 소유권과 빈 tree 상태 구현`; `fix(map): 값 allocator 상태로 노드 allocator 구성`
- **파일**: `include/ft_map.hpp`
- **함수·클래스·컴포넌트**: `node`, `node_allocator`, `_create_node`, `_destroy_node`, `_clear`, `_alloc`, `_node_alloc`
- **관련된 다른 Thread**: Thread 07의 균형 트리 노드, Thread 09의 소유권·예외 주입

<a id="map-02"></a>
## MAP-02 — [Thread 06 / `feat(map): 검색과 경계 query 구현`] 비교자 동치와 `find`·`lower_bound`·`upper_bound`

### 면접 질문

`map`에서 키의 같음을 `operator==`로 판단하지 않고 비교자 두 번으로 판단하는 이유는 무엇입니까?  
`find`, `lower_bound`, `upper_bound`를 각각 한 번의 루트-리프 탐색으로 구현할 때 후보 노드를 어떻게 유지합니까?

꼬리 질문:
- `lower_bound(k)`와 `upper_bound(k)`의 비교식 차이를 말해보세요.
  - **모범답변:** lower_bound는 현재 키가 k보다 작지 않을 때, 즉 `!comp(current, k)`이면 후보로 잡습니다. upper_bound는 k가 현재 키보다 엄격히 작을 때, 즉 `comp(k, current)`일 때만 후보가 됩니다.
- 사용자 비교자가 역순 정렬이어도 알고리즘이 동작하려면 무엇을 가정해야 합니까?
  - **모범답변:** 왼쪽·오른쪽 링크가 그 비교자의 순서로 구성되고 비교자가 strict weak ordering을 만족해야 합니다. 알고리즘이 `operator<`가 아니라 동일한 comp만 사용하면 오름차순인지 내림차순인지와 무관하게 동작합니다.
- 비교자가 strict weak ordering을 깨면 어떤 API가 서로 모순될 수 있습니까?
  - **모범답변:** 삽입이 판단한 중복 여부와 `find`·`count`의 동치 판정이 달라질 수 있고, 중위 순회 순서와 `lower_bound`·`upper_bound`·`equal_range` 결과도 서로 일치하지 않을 수 있습니다.

### 30초 모범 답변

정렬 연관 컨테이너의 동치는 `!comp(a,b) && !comp(b,a)`로 정의되므로 키의 `operator==`와 독립적입니다. `find`는 두 방향 비교가 모두 거짓일 때 찾고, `lower_bound`는 현재 키가 검색 키보다 작지 않으면 현재를 후보로 저장하고 왼쪽으로 갑니다. `upper_bound`는 검색 키가 현재 키보다 작을 때만 후보로 저장합니다. 이 논리는 비교자가 strict weak ordering을 제공한다는 계약 위에서 정렬 방향과 무관하게 동작합니다.

### 답변 핵심 키워드

- comparator equivalence
- strict weak ordering
- `find`
- `lower_bound`
- `upper_bound`
- candidate node
- single tree walk
- O(height)

### 백지 구현

- **구현 목표**: 부모 포인터가 있는 이진 탐색 트리에서 세 가지 조회 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: `find_node`, `lower_bound_node`, `upper_bound_node`.
- **입력과 출력**: 키를 받아 해당 노드 또는 조건을 만족하는 첫 후보 노드를 반환한다.
- **반드시 만족해야 할 조건**: 키의 동치는 비교자만으로 판단하고, 각 함수는 루트에서 한 경로만 탐색한다.
- **경계 조건**: 빈 트리, 최소보다 작은 키, 최대보다 큰 키, 존재하는 키, 비교자 기준 동치 키.
- **실패 조건**: 비교자가 예외를 던지면 트리 상태를 변경하지 않고 전파한다.
- **필요한 제약**: 비교자는 strict weak ordering을 만족한다고 가정한다.

```cpp
template <class Key, class Compare>
struct bst_lookup
{
    struct node
    {
        Key key;
        node* left;
        node* right;
    };

    static node* find_node(node* root, const Key& key, Compare comp);
    static node* lower_bound_node(node* root, const Key& key, Compare comp);
    static node* upper_bound_node(node* root, const Key& key, Compare comp);
};

template <class Key, class Compare>
typename bst_lookup<Key, Compare>::node*
bst_lookup<Key, Compare>::find_node(node* current,
    const Key& key, Compare comp)
{
    while (current)
    {
        if (comp(key, current->key))
            current = current->left;
        else if (comp(current->key, key))
            current = current->right;
        else
            return current; // 두 방향이 모두 거짓이면 비교자 기준 동치다.
    }
    return NULL;
}

template <class Key, class Compare>
typename bst_lookup<Key, Compare>::node*
bst_lookup<Key, Compare>::lower_bound_node(node* current,
    const Key& key, Compare comp)
{
    node* candidate = NULL;
    while (current)
    {
        if (!comp(current->key, key))
        {
            candidate = current;
            current = current->left;
        }
        else
            current = current->right;
    }
    return candidate;
}

template <class Key, class Compare>
typename bst_lookup<Key, Compare>::node*
bst_lookup<Key, Compare>::upper_bound_node(node* current,
    const Key& key, Compare comp)
{
    node* candidate = NULL;
    while (current)
    {
        if (comp(key, current->key))
        {
            candidate = current;
            current = current->left;
        }
        else
            current = current->right;
    }
    return candidate;
}
```

### 구현 후 자가 검증

- 빈 트리에서 세 함수가 null을 반환한다.
- 존재하는 키를 비교자 동치로 찾는다.
- 없는 키의 `lower_bound`가 첫 '작지 않은' 키를 반환한다.
- 없는 키의 `upper_bound`가 첫 '큰' 키를 반환한다.
- 최솟값보다 작은 입력과 최댓값보다 큰 입력을 처리한다.
- 역순 비교자를 사용해도 비교자 기준 결과가 맞다.
- 방문 노드 수가 트리 높이에 비례한다.
- 조회 중 노드·메타데이터 변경이 없다.

### 구현 후 설명할 것

1. `operator==` 대신 비교자 동치를 사용해야 하는 컨테이너 계약.
   - **모범답변:** 정렬 연관 컨테이너의 키 유일성과 조회는 `!comp(a,b) && !comp(b,a)`인 동치류를 기준으로 합니다. 따라서 `operator==`가 없거나 그 의미가 달라도 comparator가 정의한 같은 키를 일관되게 다뤄야 합니다.
2. 경계 조회에서 마지막 후보를 저장하며 반대쪽으로 좁히는 이유.
   - **모범답변:** 현재 노드가 조건을 만족해도 왼쪽 서브트리에 더 이른 답이 있을 수 있습니다. 현재를 candidate로 보존하고 왼쪽으로 좁히며, 조건을 만족하지 않으면 오른쪽만 탐색하면 한 경로에서 최선 후보를 얻습니다.
3. 평균·최악 복잡도가 트리 높이에 의존하는 점.
   - **모범답변:** 각 단계에서 자식 하나만 선택하므로 방문 수는 O(h)입니다. 일반 BST는 평균 O(log n), 퇴화하면 O(n)이지만 원본 map은 레드-블랙 균형으로 최악 높이도 O(log n)을 유지합니다.
4. 비교자 계약 위반을 컨테이너가 내부에서 복구할 수 없는 이유.
   - **모범답변:** 트리의 배치·동치·경계 조회가 모두 같은 비교 결과를 전제로 합니다. 비추이적이거나 호출마다 바뀌는 비교자는 일관된 트리 순서 자체를 정의하지 못하므로 컨테이너가 추가 비교만으로 모순을 복구할 수 없습니다.

### 원본 확인 위치

- **Thread**: 06
- **커밋 메시지**: `feat(map): 검색과 경계 query 구현`; `test(map): 역방향 순회와 경계 query 검증`
- **파일**: `include/ft_map.hpp`, `tests/test_containers.cpp`
- **함수·클래스·컴포넌트**: `find`, `count`, `lower_bound`, `upper_bound`, `equal_range`, `_find_node`, `_lower_bound_node`, `_upper_bound_node`, `_comp`
- **관련된 다른 Thread**: Thread 07의 균형화·복잡도, Thread 09의 비교자 예외 경계

<a id="map-03"></a>
## MAP-03 — [Thread 06 / `feat(map): 삭제와 clear 및 swap 구현`] BST 삭제·`transplant`·범위 삭제 진행자 보존

### 면접 질문

자식이 0개, 1개, 2개인 BST 노드를 삭제하는 구조적 처리를 설명해보세요.  
두 자식이 있는 경우 successor를 옮길 때 부모 링크를 어디까지 갱신해야 하며, `erase(first, last)`에서 왜 현재 노드를 지우기 전에 다음 반복자를 저장해야 합니까?

꼬리 질문:
- successor가 바로 오른쪽 자식인 경우와 더 아래에 있는 경우의 차이는 무엇입니까?
  - **모범답변:** 바로 오른쪽 자식이면 successor를 원래 자리에서 먼저 떼어낼 필요가 없고 그 오른쪽 자식의 부모만 successor로 유지하면 됩니다. 더 아래라면 successor를 자신의 오른쪽 자식으로 transplant한 뒤 대상의 오른쪽 서브트리를 successor에 다시 연결해야 합니다.
- 삭제 대상의 값만 복사하는 방식은 `pair<const Key, T>`에서 어떤 제약이 있습니까?
  - **모범답변:** `value_type`의 key가 const이므로 다른 노드의 pair를 대상 pair에 대입해 키를 바꿀 수 없습니다. 노드 자체를 구조적으로 이식하면 const key를 수정하지 않고 주소도 유지할 수 있습니다.
- 레드-블랙 트리에서는 이 구조 삭제 뒤 어떤 추가 정보가 필요합니까?
  - **모범답변:** 실제로 제거된 위치의 노드 원래 색, 그 자리를 대체한 `x`, 그리고 x가 NULL일 때 사용할 부모가 필요합니다. 제거 색이 검정이면 이 정보로 검정 높이 결손 fixup을 시작합니다.

### 30초 모범 답변

구조 삭제는 `transplant`로 부모가 가리키는 자식을 교체하는 문제로 정리할 수 있습니다. 자식이 하나 이하면 그 자식으로 대체하고, 두 자식이면 오른쪽 서브트리의 최소 노드인 successor를 분리해 대상 위치로 옮긴 뒤 왼쪽·오른쪽·부모 링크를 모두 고칩니다. 키가 const인 값 객체를 덮어쓰기보다 노드 자체를 이동하는 편이 맞습니다. 범위 삭제는 현재 반복자가 곧 무효화되므로 삭제 전에 successor 반복자를 구해 두어야 합니다.

### 답변 핵심 키워드

- BST deletion
- `transplant`
- successor
- parent link
- `pair<const Key,T>`
- iterator invalidation
- save-next-before-erase
- structural removal

### 백지 구현

- **구현 목표**: 균형 보정이 없는 부모 포인터 BST의 노드 삭제를 구현한다.
- **인터페이스 또는 함수 시그니처**: `transplant(node*, node*)`, `erase_node(node*)`, `erase_range(iterator, iterator)`.
- **입력과 출력**: 삭제할 노드를 제거하고 새 루트와 부모·자식 링크가 일관된 트리를 남긴다.
- **반드시 만족해야 할 조건**: 중위 순회 순서가 유지되고, 삭제된 노드만 소멸·해제된다.
- **경계 조건**: 루트 삭제, 잎 삭제, 한 자식, 두 자식, successor가 바로 오른쪽 자식, 전체 범위 삭제.
- **실패 조건**: 이 축소 문제에서는 비교·할당을 하지 않으며 삭제 자체는 예외를 던지지 않는다고 가정한다.
- **필요한 제약**: 값 복사로 키를 교체하지 않고 노드 링크를 변경한다.

```cpp
template <class Value>
class parent_bst
{
    struct node
    {
        Value value;
        node* parent;
        node* left;
        node* right;
    };

public:
    void erase_node(node* target);

private:
    node* root_;
    std::size_t size_;

    static node* minimum(node* current);
    void transplant(node* old_node, node* new_node);
    void destroy_node(node* current);
};

template <class Value>
typename parent_bst<Value>::node*
parent_bst<Value>::minimum(node* current)
{
    while (current && current->left)
        current = current->left;
    return current;
}

template <class Value>
void parent_bst<Value>::transplant(node* old_node, node* new_node)
{
    if (old_node->parent == NULL)
        root_ = new_node;
    else if (old_node == old_node->parent->left)
        old_node->parent->left = new_node;
    else
        old_node->parent->right = new_node;
    if (new_node)
        new_node->parent = old_node->parent;
}

template <class Value>
void parent_bst<Value>::erase_node(node* target)
{
    if (target == NULL)
        return;
    if (target->left == NULL)
        transplant(target, target->right);
    else if (target->right == NULL)
        transplant(target, target->left);
    else
    {
        node* successor = minimum(target->right);
        if (successor->parent != target)
        {
            transplant(successor, successor->right);
            successor->right = target->right;
            successor->right->parent = successor;
        }
        transplant(target, successor);
        successor->left = target->left;
        successor->left->parent = successor;
    }
    destroy_node(target);
    --size_;
}
```

### 구현 후 자가 검증

- 빈 트리 또는 유효하지 않은 `end` 삭제 정책이 명확하다.
- 잎·한 자식·두 자식·루트 삭제 뒤 중위 순서가 맞다.
- 모든 비루트 노드의 `parent`가 실제 부모를 가리킨다.
- 루트의 부모 표현이 설계 계약과 일치한다.
- successor를 원래 자리에서 분리할 때 오른쪽 자식의 부모를 갱신한다.
- 삭제된 노드만 한 번 destroy/deallocate한다.
- `size`는 성공한 삭제마다 정확히 1 감소한다.
- 범위 삭제는 다음 반복자를 먼저 저장해 누락·잘못된 역참조가 없다.

### 구현 후 설명할 것

1. `transplant`로 공통 링크 갱신을 분리한 이유.
   - **모범답변:** 루트인지 왼쪽 자식인지 오른쪽 자식인지에 따른 부모 링크 교체와 새 자식의 parent 갱신이 모든 삭제 경우에 반복됩니다. 이를 한 함수에 모으면 링크 대칭성 invariant를 한 곳에서 검증할 수 있습니다.
2. 두 자식 삭제에서 successor 노드를 이동하는 이유.
   - **모범답변:** 오른쪽 서브트리의 최소 노드는 대상보다 크면서 다음으로 작은 키이고 왼쪽 자식이 없습니다. 이 노드를 대상 위치로 옮기면 왼쪽·오른쪽 서브트리의 모든 키 순서를 그대로 유지할 수 있습니다.
3. `const Key`가 값 대입 기반 삭제를 부적합하게 만드는 점.
   - **모범답변:** map의 `value_type`은 `pair<const Key, T>`라 이미 생성된 노드의 key를 대입으로 교체할 수 없습니다. 구조 이식은 value를 수정하지 않고 링크만 바꾸므로 이 계약에 맞습니다.
4. 기본 BST 삭제와 Thread 07의 색·black-height 보정 책임을 분리하는 이유.
   - **모범답변:** 구조 삭제는 중위 순서와 부모 링크를 복구하는 책임이고, 색 보정은 실제 제거 색 때문에 생긴 검정 높이 결손을 복구하는 책임입니다. 원본도 transplant와 구조 처리를 끝낸 뒤 필요할 때 별도 erase fixup을 호출합니다.

### 원본 확인 위치

- **Thread**: 06, 07
- **커밋 메시지**: `feat(map): 삭제와 clear 및 swap 구현`; `test(map): 범위 삭제 후 상태 검증`
- **파일**: `include/ft_map.hpp`, `tests/test_containers.cpp`
- **함수·클래스·컴포넌트**: `erase` 오버로드, `clear`, `_transplant`, `_erase_node`, `_minimum`
- **관련된 다른 Thread**: Thread 07의 `fixup`을 포함한 레드-블랙 삭제, Thread 08의 header 갱신
