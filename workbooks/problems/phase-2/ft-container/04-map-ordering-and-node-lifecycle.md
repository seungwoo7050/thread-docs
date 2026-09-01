# Map 정렬·노드 수명·기본 삭제

비교자 기반 동치, 노드 allocator 소유권, 균형 보정 전 단계의 BST 삭제를 분리해 연습한다.

<a id="map-01"></a>
## MAP-01 — [Thread 09 / `fix(map): 값 allocator 상태로 노드 allocator 구성`] allocator rebind와 노드 소유권

### 면접 질문

`map`의 공개 allocator는 `value_type`용인데 실제 할당 단위는 내부 node입니다.  
`rebind`로 만든 node allocator가 원래 allocator의 상태를 이어받아야 하는 이유와, 노드 생성 중 값 복사가 실패할 때의 정리 순서를 설명해보세요.

꼬리 질문:
- 기본 생성한 node allocator를 쓰면 stateful allocator에서 어떤 버그가 생깁니까?
- `_create_node`가 allocate와 construct를 분리해야 하는 이유는 무엇입니까?
- `clear`와 소멸자에서 노드 수와 outstanding block 수가 어떻게 대응해야 합니까?

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
    // 직접 구현
};
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
2. 할당된 메모리와 생성된 객체를 별도 상태로 보는 관점.
3. 노드 생성 함수에 rollback을 캡슐화했을 때 상위 삽입 코드가 단순해지는 이유.
4. 재귀 clear의 단순함과 트리 높이가 비정상적으로 커질 때의 호출 스택 trade-off.

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
- 사용자 비교자가 역순 정렬이어도 알고리즘이 동작하려면 무엇을 가정해야 합니까?
- 비교자가 strict weak ordering을 깨면 어떤 API가 서로 모순될 수 있습니까?

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

    // 직접 구현
};
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
2. 경계 조회에서 마지막 후보를 저장하며 반대쪽으로 좁히는 이유.
3. 평균·최악 복잡도가 트리 높이에 의존하는 점.
4. 비교자 계약 위반을 컨테이너가 내부에서 복구할 수 없는 이유.

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
- 삭제 대상의 값만 복사하는 방식은 `pair<const Key, T>`에서 어떤 제약이 있습니까?
- 레드-블랙 트리에서는 이 구조 삭제 뒤 어떤 추가 정보가 필요합니까?

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
    // 직접 구현
};
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
2. 두 자식 삭제에서 successor 노드를 이동하는 이유.
3. `const Key`가 값 대입 기반 삭제를 부적합하게 만드는 점.
4. 기본 BST 삭제와 Thread 07의 색·black-height 보정 책임을 분리하는 이유.

### 원본 확인 위치

- **Thread**: 06, 07
- **커밋 메시지**: `feat(map): 삭제와 clear 및 swap 구현`; `test(map): 범위 삭제 후 상태 검증`
- **파일**: `include/ft_map.hpp`, `tests/test_containers.cpp`
- **함수·클래스·컴포넌트**: `erase` 오버로드, `clear`, `_transplant`, `_erase_node`, `_minimum`
- **관련된 다른 Thread**: Thread 07의 `fixup`을 포함한 레드-블랙 삭제, Thread 08의 header 갱신
