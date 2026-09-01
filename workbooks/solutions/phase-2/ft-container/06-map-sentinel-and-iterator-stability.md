# Map sentinel과 반복자 안정성

header sentinel을 단순한 종단 표식이 아니라 루트·양 끝·반복자 경계를 연결하는 표현 invariant로 다룬다.

<a id="sen-01"></a>
## SEN-01 — [Thread 08 / `fix(map): 값 없는 header로 끝 반복자 상태 안정화`] 값을 갖지 않는 header sentinel과 `end()` 표현

### 면접 질문

`map::end()`를 NULL로 표현하지 않고 컨테이너 내부의 header sentinel로 표현한 이유를 설명해보세요.  
header가 실제 `value_type`을 가지지 않도록 `node_base`와 `node`를 분리한 이유, 그리고 빈 트리와 비어 있지 않은 트리에서 header 링크가 만족해야 하는 invariant를 말해보세요.

꼬리 질문:
- header가 `value_type`을 직접 가지면 기본 생성할 수 없는 key에서 어떤 제약이 생깁니까?
  - **모범답변:** 종단 표식 하나를 만들기 위해 사용자가 key와 mapped type의 기본 생성자를 제공해야 합니다. 원본은 링크와 색만 가진 `node_base` header를 사용해 실제 값이 없는 end 표현이 value_type 생성 가능성에 의존하지 않게 했습니다.
- `header.parent`, `header.left`, `header.right`는 각각 무엇을 가리킵니까?
  - **모범답변:** 비어 있지 않을 때 parent는 root, left는 minimum, right는 maximum입니다. 빈 트리에서는 parent가 NULL이고 left와 right는 header 자신을 가리킵니다.
- `begin()`과 `end()`를 O(1)로 만들기 위해 어떤 메타데이터를 유지합니까?
  - **모범답변:** header.left에 최소 노드를 캐시해 begin이 바로 반환하고, end는 고정된 header 주소를 반환합니다. header.right도 최대 노드를 캐시해 `--end()`가 바로 최대값으로 갈 수 있습니다.
- 삽입·삭제·clear·swap 뒤 header를 갱신하지 않으면 어떤 반복자 버그가 생깁니까?
  - **모범답변:** begin이나 `--end()`가 삭제된 옛 극값을 가리키고, 루트의 parent가 옛 header를 가리켜 증감이 잘못된 컨테이너 종단에 도달할 수 있습니다. swap 뒤에는 기존 원소 반복자가 새 소유 컨테이너의 end와 연결되지 않습니다.

### 30초 모범 답변

NULL `end()`는 반복자가 루트 스냅샷을 따로 들고 있어야 해서 회전이나 루트 삭제 뒤 오래된 상태를 가리킬 수 있습니다. 이 구현은 값이 없는 `node_base` header를 컨테이너마다 두고 `end()`가 그 주소를 가리키게 했습니다. 비어 있으면 parent는 NULL이고 left와 right는 header 자신이며, 비어 있지 않으면 parent는 루트, left는 최솟값, right는 최댓값이고 루트의 parent는 header입니다. 값 없는 base 덕분에 key의 기본 생성도 요구하지 않으며, 구조 변경 뒤 이 링크를 다시 맞춰야 합니다.

### 답변 핵심 키워드

- header sentinel
- `node_base`와 `node` 분리
- 값 없는 `end()`
- root/min/max 캐시
- 빈 상태 self-reference
- 루트의 parent는 header
- non-default-constructible key
- `_refresh_header`

### 백지 구현

- **구현 목표**: 값을 소유하지 않는 header가 루트·최솟값·최댓값을 일관되게 가리키도록 초기화와 갱신 함수를 구현한다.
- **인터페이스 또는 함수 시그니처**: `init_header`, `refresh_header`, `begin_node`, `end_node`를 구현한다.
- **입력과 출력**: 현재 루트 포인터를 입력으로 받아 header 링크를 갱신하고, `begin`과 `end`가 사용할 노드 포인터를 반환한다.
- **반드시 만족해야 할 조건**: 빈 트리는 `parent == NULL`, `left == right == &header`; 비어 있지 않으면 `parent == root`, `left == minimum(root)`, `right == maximum(root)`, `root->parent == &header`다.
- **경계 조건**: 최초 빈 상태, 첫 삽입, 루트 교체, 최솟값·최댓값 삭제, 마지막 원소 삭제를 처리한다.
- **실패 조건**: 할당이나 값 생성은 하지 않으며 예외를 던지지 않는다. 깨진 자식 링크를 입력받는 경우는 사전 조건 위반이다.
- **필요한 제약**: 색 보정과 노드 할당은 제외하고 header 표현만 10~20분 안에 구현한다.

```cpp
struct node_base
{
    node_base* parent;
    node_base* left;
    node_base* right;
    bool is_header;
};

void init_header(node_base& header)
{
    header.parent = NULL;
    header.left = &header;
    header.right = &header;
    header.is_header = true;
}

void refresh_header(node_base& header, node_base* root)
{
    if (root == NULL)
    {
        init_header(header);
        return;
    }
    header.parent = root;
    root->parent = &header;

    node_base* minimum = root;
    while (minimum->left)
        minimum = minimum->left;
    node_base* maximum = root;
    while (maximum->right)
        maximum = maximum->right;
    header.left = minimum;
    header.right = maximum;
    header.is_header = true;
}

node_base* begin_node(node_base& header)
{
    return header.left;
}

node_base* end_node(node_base& header)
{
    return &header;
}
```

### 구현 후 자가 검증

- 빈 트리에서 `begin_node(header) == end_node(header)`인가?
- 빈 header가 `value_type`이나 key의 기본 생성을 전혀 요구하지 않는가?
- 첫 삽입 뒤 parent·left·right가 모두 새 노드를 가리키고 새 루트의 parent가 header인가?
- 내부 루트 회전 뒤 `header.parent`가 새 루트로 갱신되는가?
- 최솟값이나 최댓값 삭제 뒤 left·right가 남은 극값을 가리키는가?
- 마지막 원소 삭제와 `clear` 뒤 빈 상태 self-reference가 복구되는가?
- header가 실제 트리 자식 경로 안으로 들어가지 않는가?
- 갱신 비용이 트리 높이에 선형이며 균형 트리에서 O(log n)인가?

### 구현 후 설명할 것

1. NULL 종단과 컨테이너 고유 sentinel 표현의 차이
   - **모범답변:** NULL은 어느 컨테이너의 종단인지와 현재 root·maximum 정보를 담지 못합니다. 컨테이너 고유 header는 고정 identity와 최신 root/min/max 링크를 제공해 저장된 end도 구조 변경 뒤 현재 트리 경계를 볼 수 있습니다.
2. `node_base`를 사용해 header에서 `value_type` 생성을 제거한 이유
   - **모범답변:** header는 탐색 링크와 표식만 필요하고 key-value를 역참조할 대상이 아닙니다. base/derived 분리로 실제 node만 value를 생성하므로 기본 생성 불가능한 Key도 map에서 사용할 수 있습니다.
3. 최솟값·최댓값 캐시로 `begin()`과 `--end()`를 단순화한 판단
   - **모범답변:** begin은 header.left를, header에서 previous는 header.right를 반환하면 별도 루트 탐색이 없습니다. 반복자의 경계 로직이 단순해지고 두 연산이 O(1)이 됩니다.
4. 구조 변경마다 전체 극값을 다시 찾는 O(log n) 갱신과 더 세밀한 O(1) 갱신의 trade-off
   - **모범답변:** 원본 `_refresh_header`는 root에서 minimum과 maximum을 다시 찾아 구현과 검증이 단순하지만 균형 트리 높이만큼 비용이 듭니다. 삽입·삭제 위치에 따라 극값만 조건부 갱신하면 O(1)도 가능하지만 모든 구조 변경 경우의 정확성을 따로 관리해야 합니다.
5. header invariant를 삽입·삭제·clear·swap의 공통 후처리로 모으는 이유
   - **모범답변:** 여러 연산이 root와 극값을 바꾸지만 필요한 최종 표현은 같습니다. 공통 refresh/reset에 모으면 루트 parent 재연결과 빈 상태 self-reference 누락을 한 곳에서 방지할 수 있습니다.

### 원본 확인 위치

- **Thread**: 08
- **커밋 메시지**: `fix(map): 값 없는 header로 끝 반복자 상태 안정화`
- **파일**: `include/ft_map.hpp`, `tests/test_map_iterators.cpp`
- **함수·클래스·컴포넌트**: `node_base`, `node`, `_header`, `_root`, `_value`, `_refresh_header`, `begin`, `end`, `test_header_does_not_hold_a_value`
- **관련된 다른 Thread**: 07의 회전·삭제 보정, 09의 header 기반 소유권 교환

<a id="sen-02"></a>
## SEN-02 — [Thread 08 / `test(map): 회전·삭제·교환 뒤 반복자 상태 검증`] 노드 주소와 header를 이용한 반복자 안정성

### 면접 질문

회전은 트리의 부모·자식 관계를 바꾸는데도 기존 원소 반복자가 왜 유효할 수 있습니까?  
저장된 `end()`가 이후 회전이나 루트 삭제를 반영하고, `swap` 전 얻은 원소 반복자가 교환 뒤 새 소유 컨테이너의 `end()`까지 진행하려면 반복자가 무엇을 저장해야 하는지 설명해보세요.

꼬리 질문:
- 반복자가 노드와 당시 루트를 함께 저장하는 설계는 어떤 실패를 만들었습니까?
  - **모범답변:** 회전이나 루트 삭제 뒤 저장한 root가 오래된 내부 노드 또는 삭제된 노드가 됩니다. 이후 증감이 그 스냅샷을 경계로 사용하면 현재 종단을 찾지 못하거나 잘못된 경로를 따릅니다.
- 회전과 삭제에서 유효성이 다른 반복자는 무엇입니까?
  - **모범답변:** 회전은 노드를 파괴하지 않아 모든 원소 반복자가 유효합니다. 삭제는 제거한 노드의 반복자만 무효화하고, 다른 노드 주소와 value는 유지됩니다.
- `swap` 뒤 원소 노드의 최상위 parent를 각 컨테이너 header에 다시 연결해야 하는 이유는 무엇입니까?
  - **모범답변:** 원소 반복자의 증감은 parent 체인을 따라 header를 종단으로 만납니다. 루트가 옛 header를 계속 가리키면 교환된 원소 반복자가 새 소유 map의 end가 아니라 이전 map의 end에 도달합니다.
- header 극값을 다시 계산하는 비용이 `swap` 복잡도에 어떤 영향을 줍니까?
  - **모범답변:** 포인터·size 교환 자체는 상수 시간이지만 양쪽 `_refresh_header`가 각 새 root에서 min/max를 찾습니다. 따라서 원본 public swap은 O(h₁+h₂), 균형 트리에서는 O(log n + log m)입니다.

### 30초 모범 답변

회전은 노드 객체를 옮겨 생성하지 않고 링크만 바꾸므로 원소 반복자가 노드 주소 하나만 저장하면 해당 원소는 유지됩니다. 증감은 현재 부모 링크와 header를 따라 계산하므로 최신 구조를 봅니다. `end()`는 컨테이너의 고정 header 주소라서 저장해둔 뒤에도 현재 최댓값을 찾을 수 있습니다. 삭제는 지운 노드의 반복자만 무효화하고, `swap`은 루트 소유권을 교환한 뒤 각 루트의 parent와 header의 root/min/max를 다시 연결해야 기존 원소 반복자가 새 컨테이너의 종단에 도달합니다.

### 답변 핵심 키워드

- iterator stores node identity
- rotation changes links, not nodes
- saved `end()`
- current parent chain
- erase invalidates erased node
- swap transfers ownership
- root-to-header relink
- iterator stability

### 백지 구현

- **구현 목표**: 노드 주소 하나만 보유하고 header-aware successor·predecessor를 사용하는 양방향 트리 반복자를 구현한다.
- **인터페이스 또는 함수 시그니처**: `next_node`, `previous_node`, 반복자의 전위 `++`·`--`를 구현한다.
- **입력과 출력**: 현재 노드 또는 header를 입력받아 중위 순회의 다음 또는 이전 노드를 반환한다.
- **반드시 만족해야 할 조건**: 최댓값의 다음은 header, header의 이전은 현재 최댓값, 일반 노드의 증감은 중위 순서를 따른다.
- **경계 조건**: 최솟값, 최댓값, header, 한 원소 트리, 회전 뒤 변경된 부모 링크를 처리한다.
- **실패 조건**: 빈 컨테이너의 `--end()`나 `end()` 역참조처럼 공개 계약 밖 연산의 결과는 요구하지 않는다.
- **필요한 제약**: 노드 생성·삭제·회전은 제공되며 반복자 탐색 로직만 15~25분 안에 구현한다.

```cpp
struct node_base
{
    node_base* parent;
    node_base* left;
    node_base* right;
    bool is_header;
};

node_base* next_node(node_base* current)
{
    if (current == NULL)
        return NULL;
    if (current->is_header)
        return current;
    if (current->right)
    {
        current = current->right;
        while (current->left)
            current = current->left;
        return current;
    }
    node_base* parent = current->parent;
    while (!parent->is_header && current == parent->right)
    {
        current = parent;
        parent = parent->parent;
    }
    return parent;
}

node_base* previous_node(node_base* current)
{
    if (current == NULL)
        return NULL;
    if (current->is_header)
        return current->right;
    if (current->left)
    {
        current = current->left;
        while (current->right)
            current = current->right;
        return current;
    }
    node_base* parent = current->parent;
    while (!parent->is_header && current == parent->left)
    {
        current = parent;
        parent = parent->parent;
    }
    return parent;
}

class tree_iterator
{
public:
    tree_iterator& operator++();
    tree_iterator& operator--();

private:
    node_base* current_;
};

tree_iterator& tree_iterator::operator++()
{
    current_ = next_node(current_);
    return *this;
}

tree_iterator& tree_iterator::operator--()
{
    current_ = previous_node(current_);
    return *this;
}
```

### 구현 후 자가 검증

- 중위 순회가 최솟값부터 최댓값까지 정확히 한 번씩 방문하는가?
- 최댓값에서 `++` 하면 해당 컨테이너 header가 되는가?
- 비어 있지 않은 트리의 header에서 `--` 하면 현재 최댓값이 되는가?
- 반복자를 저장한 뒤 회전해도 같은 원소를 역참조하고 다음 원소로 진행하는가?
- 루트를 삭제한 뒤 저장된 `end()`에서 감소하면 현재 최댓값을 얻는가?
- 한 노드를 삭제했을 때 다른 노드 반복자는 계속 순회할 수 있는가?
- 두 컨테이너를 교환한 뒤 기존 원소 반복자가 새 소유 컨테이너의 `end()`에 도달하는가?
- 잘못된 parent 링크로 무한 루프가 생기지 않는가?
- 한 번의 증감 비용이 최악 O(height), 전체 순회가 O(n)인가?

### 구현 후 설명할 것

1. 노드 주소 안정성과 트리 위치 안정성을 구분한 설계
   - **모범답변:** 회전과 swap은 노드의 부모·자식 위치는 바꾸지만 노드 객체의 주소와 value 수명은 바꾸지 않습니다. 반복자가 노드 identity만 저장하면 위치 변화와 무관하게 같은 원소를 계속 가리킬 수 있습니다.
2. 루트 스냅샷을 반복자에 저장하지 않고 header를 탐색 경계로 사용한 이유
   - **모범답변:** 저장한 root는 구조 변경 뒤 stale해질 수 있지만 root의 parent 체인은 공통 후처리로 현재 header에 연결됩니다. 반복자가 현재 링크를 따라 header를 만나는 방식이면 별도 스냅샷 갱신이 필요 없습니다.
3. 회전·삭제·swap 각각의 반복자 무효화 범위
   - **모범답변:** 원본에서 회전은 링크만 바꾸므로 원소 반복자를 무효화하지 않습니다. erase는 제거된 노드 반복자만 무효화하고, swap은 노드 ownership을 옮기므로 원소 반복자는 유효하지만 어느 컨테이너의 end에 도달하는지가 바뀝니다.
4. saved `end()`와 원소 반복자가 구조 변경 뒤 최신 메타데이터를 보는 경로
   - **모범답변:** saved end는 고정 header 주소 자체를 저장하고 `--`할 때 현재 header.right를 봅니다. 원소 반복자는 현재 노드의 parent 링크를 따라가며 refresh가 연결한 최신 header에 도달합니다.
5. `swap`에서 header 재연결 때문에 상수 시간이 아닌 비용을 받아들인 trade-off
   - **모범답변:** 양쪽 root의 parent와 header min/max를 확실히 재구축해 반복자·종단 invariant를 단순하게 보장하는 대신 O(h₁+h₂)를 지불합니다. 더 복잡한 header 소유 구조를 쓰면 상수 시간 swap을 노릴 수 있지만 원본은 명료한 재계산을 선택했습니다.

### 원본 확인 위치

- **Thread**: 08
- **커밋 메시지**: `test(map): 회전·삭제·교환 뒤 반복자 상태 검증`
- **파일**: `include/ft_map.hpp`, `tests/test_map_iterators.cpp`
- **함수·컴포넌트**: `iterator`, `const_iterator`, `_next`, `_previous`, `_minimum`, `_maximum`, `_refresh_header`, `swap`, `test_saved_end_after_rotation`, `test_saved_end_after_root_erasure`, `test_element_iterator_survives_rotation`, `test_iterators_across_swap`
- **관련된 다른 Thread**: 06의 최초 tree 반복자, 07의 회전·삭제, 09의 정책 객체를 포함한 swap

<a id="sen-03"></a>
## SEN-03 — [Thread 08 / `feat(map): 가변·상수 반복자 상호 비교 지원`] 가변·상수 반복자 변환과 대칭 비교

### 면접 질문

`iterator`에서 `const_iterator`로는 변환할 수 있지만 반대 방향은 허용하면 안 되는 이유를 설명해보세요.  
두 반복자 형식 사이의 `==`와 `!=`를 양쪽 피연산자 순서에서 모두 지원하면서 const-correctness와 중복 구현을 어떻게 관리할 수 있습니까?

꼬리 질문:
- 비교의 기준이 노드 주소 하나여야 하는 이유는 무엇입니까?
  - **모범답변:** 가변·상수 반복자는 접근 권한만 다르고 같은 순회 위치를 표현합니다. value 비교는 서로 다른 노드의 같은 값도 같다고 만들 수 있고 end는 역참조할 수도 없으므로 내부 node identity를 비교해야 합니다.
- 멤버 연산자만 정의했을 때 `iterator == const_iterator`와 반대 순서의 후보 집합은 어떻게 달라질 수 있습니까?
  - **모범답변:** 왼쪽 피연산자의 형식에 따라 검색되는 멤버 함수가 달라지고, 허용된 단방향 변환이 어느 피연산자에 적용되는지도 달라집니다. 양 클래스에 같은 혼합 템플릿 비교를 제공하거나 대칭 비멤버를 두어 두 순서를 명시적으로 지원해야 합니다.
- `reverse_iterator<const_iterator>`는 별도 순회 알고리즘 없이 어떻게 구성됩니까?
  - **모범답변:** const_iterator를 기저 반복자로 reverse adapter에 넘깁니다. adapter는 역참조 시 기저를 하나 감소시키고 증감 방향만 반전하므로 tree의 next/previous와 const 접근 권한을 그대로 재사용합니다.
- 기본 생성 반복자 비교와 서로 다른 컨테이너 반복자 비교에서 어떤 연산이 계약 밖입니까?
  - **모범답변:** 기본 생성 반복자끼리 같은 null identity를 비교하는 것은 구현상 가능하지만 역참조·증감은 계약 밖입니다. 서로 다른 컨테이너 반복자는 일반적으로 순서·거리·증감 관계가 없고, 이 문제도 비종단 반복자의 교차 컨테이너 비교 의미를 보장하지 않습니다.

### 30초 모범 답변

가변 반복자를 상수 반복자로 바꾸는 것은 쓰기 권한을 줄이는 변환이라 안전하지만, 반대는 const를 제거하므로 허용하지 않습니다. 두 형식은 같은 내부 노드 식별자를 공유하고 비교는 그 식별자만 사용합니다. 한쪽 형식에서 다른 쪽을 볼 수 있게 friend 관계나 공통 비교 템플릿을 두면 양쪽 피연산자 순서의 `==`와 `!=`를 대칭으로 제공할 수 있습니다. 역방향 반복자는 이 상수·가변 기본 반복자를 어댑터에 넣어 재사용합니다.

### 답변 핵심 키워드

- const-correctness
- one-way conversion
- shared node identity
- mixed-type equality
- symmetric overload
- friend access
- comparison reuse
- reverse adapter composition

### 백지 구현

- **구현 목표**: 같은 노드 포인터를 보유하는 `iterator`와 `const_iterator` 사이의 단방향 변환과 대칭 비교를 구현한다.
- **인터페이스 또는 함수 시그니처**: 두 반복자 클래스, `const_iterator(const iterator&)`, 혼합 `operator==`·`operator!=`를 제공한다.
- **입력과 출력**: 같거나 다른 노드를 가리키는 두 반복자를 비교해 bool을 반환한다.
- **반드시 만족해야 할 조건**: `iterator`에서 `const_iterator` 변환은 가능하고 반대는 불가능하며, 두 비교 순서의 결과가 동일하다.
- **경계 조건**: 기본 생성된 두 반복자, 같은 원소의 가변·상수 반복자, 서로 다른 원소를 처리한다.
- **실패 조건**: 서로 다른 컨테이너의 비종단 반복자를 비교한 의미는 보장 대상으로 삼지 않는다.
- **필요한 제약**: 증감과 역참조는 제공된 것으로 가정하고 형식 변환·비교만 10~15분 안에 구현한다.

```cpp
class const_tree_iterator;

class tree_iterator
{
    friend class const_tree_iterator;
public:
    tree_iterator() : current_(NULL) {}
    explicit tree_iterator(node_base* current) : current_(current) {}

    template <class OtherIterator>
    bool operator==(const OtherIterator& other) const
    {
        return current_ == other.current_;
    }

    template <class OtherIterator>
    bool operator!=(const OtherIterator& other) const
    {
        return !(*this == other);
    }
private:
    node_base* current_;
};

class const_tree_iterator
{
    friend class tree_iterator;
public:
    const_tree_iterator() : current_(NULL) {}
    const_tree_iterator(const tree_iterator& other)
        : current_(other.current_) {}

    template <class OtherIterator>
    bool operator==(const OtherIterator& other) const
    {
        return current_ == other.current_;
    }

    template <class OtherIterator>
    bool operator!=(const OtherIterator& other) const
    {
        return !(*this == other);
    }
private:
    node_base* current_;
};
```

### 구현 후 자가 검증

- `const_iterator c = iterator_value;`가 컴파일되는가?
- `iterator i = const_iterator_value;`는 컴파일되지 않는가?
- 같은 노드를 가리킬 때 `i == c`와 `c == i`가 모두 참인가?
- `i != c`와 `c != i`가 각각 `==`의 정확한 부정인가?
- 다른 노드를 가리킬 때 네 혼합 비교가 일관된가?
- 비교 구현이 값을 역참조하지 않아 `end()`끼리도 동작하는가?
- 상수 반복자 역참조를 통해 mapped value를 수정할 수 없는가?
- 혼합 비교 추가로 모호한 오버로드나 무한 재귀가 생기지 않는가?

### 구현 후 설명할 것

1. 읽기 권한만 줄이는 단방향 변환 규칙
   - **모범답변:** iterator가 가리키는 mutable value를 const view로 노출하는 것은 안전하므로 const_iterator가 iterator를 받는 생성자를 둡니다. 반대 생성자를 두지 않으면 const_iterator에서 쓰기 권한을 되살리는 변환은 컴파일되지 않습니다.
2. 노드 식별자 비교와 값 비교를 분리한 이유
   - **모범답변:** 반복자 동등성은 같은 순회 위치인지의 문제이지 key-value가 같은지의 문제가 아닙니다. node 주소 비교는 end도 역참조 없이 처리하고 const 자격이 다른 두 view를 같은 identity로 판정합니다.
3. 멤버·비멤버·템플릿 비교 연산자 중 선택한 방식과 모호성 trade-off
   - **모범답변:** 원본은 양 클래스에 OtherIterator를 받는 멤버 템플릿을 두고 friend로 node를 공유해 두 피연산자 순서를 지원합니다. 구현은 짧지만 너무 넓은 형식이 후보가 될 수 있어, 더 큰 라이브러리라면 반복자 형식을 제한한 비멤버 오버로드가 진단과 모호성 면에서 나을 수 있습니다.
4. `!=`를 `==`의 부정으로 정의해 의미를 한 곳에 둔 판단
   - **모범답변:** 두 연산에 별도 비교 로직을 쓰면 const 혼합 조합마다 불일치할 수 있습니다. node identity 판정을 `==` 한 곳에 두고 `!=`를 부정으로 만들면 항상 보수 관계가 유지됩니다.
5. 기본 반복자 위에 reverse iterator를 조합해 const 동작을 재사용한 구조
   - **모범답변:** reverse adapter는 기저 반복자의 타입과 연산만 사용합니다. iterator와 const_iterator가 같은 순회·혼합 비교 계약을 제공하므로 각각을 감싸 가변·상수 역방향 반복자를 만들고 별도 tree 역순 알고리즘을 구현하지 않습니다.

### 원본 확인 위치

- **Thread**: 08
- **커밋 메시지**: `feat(map): 가변·상수 반복자 상호 비교 지원`
- **파일**: `include/ft_map.hpp`, `tests/test_map_iterators.cpp`
- **함수·클래스·컴포넌트**: `iterator`, `const_iterator`, `reverse_iterator`, `const_reverse_iterator`, `test_mixed_iterator_comparisons`
- **관련된 다른 Thread**: 01의 `reverse_iterator`, 06의 상수·역방향 tree 반복자
