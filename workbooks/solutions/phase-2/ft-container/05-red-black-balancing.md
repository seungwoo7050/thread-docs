# 레드-블랙 트리 균형화

삽입과 삭제 뒤 구조를 복구하는 핵심 알고리즘을 분리했다. 두 문제 모두 공개 결과만 맞추는 것이 아니라 부모 링크·색·검정 높이 invariant를 직접 다룬다.

<a id="rbt-01"></a>
## RBT-01 — [Thread 07 / `feat(map): 레드-블랙 삽입 회전과 색 보정 구현`] 회전과 삽입 보정으로 레드-블랙 invariant 복구

### 면접 질문

새 노드를 일반 BST 규칙으로 삽입한 직후 어떤 레드-블랙 invariant가 깨질 수 있습니까?  
부모와 삼촌의 색, 그리고 새 노드가 안쪽 자식인지 바깥쪽 자식인지에 따라 보정 절차가 어떻게 달라지는지 설명해보세요.

꼬리 질문:
- 왼쪽 회전과 오른쪽 회전이 BST 정렬 순서를 보존하는 이유는 무엇입니까?
  - **모범답변:** 왼쪽 회전에서 pivot과 오른쪽 자식 사이의 중간 서브트리만 pivot의 오른쪽으로 옮깁니다. 그 키들은 여전히 pivot보다 크고 새 부모보다 작으므로 중위 순서가 같으며 오른쪽 회전도 대칭입니다.
- 삼촌이 빨강일 때 회전하지 않고 재색칠한 뒤 문제를 위로 올리는 이유는 무엇입니까?
  - **모범답변:** 부모와 삼촌을 검정으로 바꾸면 두 하위 경로의 검정 높이가 함께 하나 늘고 현재 red-red 위반이 사라집니다. 조부모를 빨강으로 바꾸어 조부모 위 경로의 검정 높이는 유지하되, 새 위반 가능성만 조부모 위치로 올립니다.
- 회전 뒤 반드시 갱신해야 하는 부모·자식 링크와 루트 포인터는 무엇입니까?
  - **모범답변:** 이동한 중간 서브트리의 parent, 새 상위 노드의 기존 parent 연결, pivot의 parent, 새 상위 노드가 가리키는 pivot 링크를 모두 갱신합니다. pivot이 루트였다면 root 참조도 새 상위 노드로 바꿔야 합니다.
- 마지막에 루트를 검정으로 만드는 처리가 필요한 이유는 무엇입니까?
  - **모범답변:** 재색칠이 위로 전파되면 기존 루트가 빨강이 될 수 있고 첫 삽입 노드도 처음에는 빨강입니다. 루트를 검정으로 고정하면 루트 invariant를 회복하며 모든 root-to-leaf 경로에 동일하게 검정 하나를 더하므로 검정 높이 동등성은 유지됩니다.

### 30초 모범 답변

새 노드는 빨강으로 붙이면 기존 경로의 검정 높이는 유지되지만, 부모가 빨강이면 빨강-빨강 위반이 생깁니다. 삼촌도 빨강이면 부모와 삼촌을 검정, 조부모를 빨강으로 바꾸고 조부모에서 다시 검사합니다. 삼촌이 검정이면 안쪽 모양을 한 번 회전해 바깥쪽 모양으로 바꾼 뒤, 부모와 조부모를 재색칠하고 반대 방향으로 회전합니다. 회전은 중위 순서를 유지해야 하며, 끝에는 루트와 모든 부모 링크를 정상화하고 루트를 검정으로 둡니다.

### 답변 핵심 키워드

- BST 중위 순서
- 새 노드는 빨강
- red-red 위반
- 삼촌 재색칠
- 안쪽·바깥쪽 모양
- 좌·우 회전
- 부모 링크
- 검정 루트

### 백지 구현

- **구현 목표**: 이미 BST 위치에 연결된 빨강 노드 하나를 받아 회전과 재색칠로 레드-블랙 invariant를 복구한다.
- **인터페이스 또는 함수 시그니처**: `rotate_left`, `rotate_right`, `insert_fixup` 세 함수를 구현한다.
- **입력과 출력**: 부모·왼쪽·오른쪽·색 필드를 가진 노드와 루트 참조를 입력받고, 같은 노드 집합을 올바르게 다시 연결한다.
- **반드시 만족해야 할 조건**: 중위 순서, 부모·자식 링크의 대칭성, 검정 루트, 빨강 노드의 검정 자식, 모든 경로의 동일한 검정 높이를 유지한다.
- **경계 조건**: 새 노드가 루트인 경우, 부모가 루트인 경우, 삼촌이 없는 경우, 네 가지 좌우 대칭 모양을 처리한다.
- **실패 조건**: 이 문제에서는 비교·할당·사용자 코드 호출이 없으므로 예외를 발생시키지 않는다. 잘못된 링크 입력은 사전 조건 위반으로 본다.
- **필요한 제약**: 노드 생성과 BST 탐색은 이미 끝났다고 가정하고, 보정 코드만 20~30분 안에 작성한다.

```cpp
struct node
{
    node* parent;
    node* left;
    node* right;
    bool red;
};

void rotate_left(node*& root, node* pivot)
{
    node* upper = pivot->right;
    pivot->right = upper->left;
    if (upper->left)
        upper->left->parent = pivot;
    upper->parent = pivot->parent;
    if (pivot->parent == NULL)
        root = upper;
    else if (pivot == pivot->parent->left)
        pivot->parent->left = upper;
    else
        pivot->parent->right = upper;
    upper->left = pivot;
    pivot->parent = upper;
}

void rotate_right(node*& root, node* pivot)
{
    node* upper = pivot->left;
    pivot->left = upper->right;
    if (upper->right)
        upper->right->parent = pivot;
    upper->parent = pivot->parent;
    if (pivot->parent == NULL)
        root = upper;
    else if (pivot == pivot->parent->right)
        pivot->parent->right = upper;
    else
        pivot->parent->left = upper;
    upper->right = pivot;
    pivot->parent = upper;
}

void insert_fixup(node*& root, node* inserted)
{
    while (inserted != root && inserted->parent->red)
    {
        node* parent = inserted->parent;
        node* grandparent = parent->parent;
        if (parent == grandparent->left)
        {
            node* uncle = grandparent->right;
            if (uncle && uncle->red)
            {
                parent->red = false;
                uncle->red = false;
                grandparent->red = true;
                inserted = grandparent; // 위쪽에서 새 red-red 가능성을 검사한다.
            }
            else
            {
                if (inserted == parent->right)
                {
                    inserted = parent;
                    rotate_left(root, inserted);
                    parent = inserted->parent;
                    grandparent = parent->parent;
                }
                parent->red = false;
                grandparent->red = true;
                rotate_right(root, grandparent);
            }
        }
        else
        {
            node* uncle = grandparent->left;
            if (uncle && uncle->red)
            {
                parent->red = false;
                uncle->red = false;
                grandparent->red = true;
                inserted = grandparent;
            }
            else
            {
                if (inserted == parent->left)
                {
                    inserted = parent;
                    rotate_right(root, inserted);
                    parent = inserted->parent;
                    grandparent = parent->parent;
                }
                parent->red = false;
                grandparent->red = true;
                rotate_left(root, grandparent);
            }
        }
    }
    if (root)
        root->red = false;
}
```

### 구현 후 자가 검증

- 오름차순, 내림차순, 지그재그 삽입 순서에서도 중위 순회 결과가 정렬되어 있는가?
- 회전 후 모든 자식의 `parent`가 실제 부모를 가리키고, 루트의 부모 표현이 계약과 일치하는가?
- 루트가 항상 검정인가?
- 빨강 노드 아래에 빨강 자식이 남지 않는가?
- 모든 NULL 잎까지의 검정 높이가 같은가?
- 회전이나 재색칠 때문에 노드가 중복되거나 도달 불가능해지지 않았는가?
- 삽입 한 번당 노드 수가 정확히 하나만 증가하는가?
- 보정의 시간 복잡도가 트리 높이에 선형, 즉 균형 트리에서 O(log n)인가?

### 구현 후 설명할 것

1. 새 노드를 빨강으로 시작해 검정 높이 변화를 피하는 이유
   - **모범답변:** 빨강 노드는 어느 root-to-NULL 경로의 검정 노드 수에도 포함되지 않습니다. 따라서 BST 위치에 붙였을 때 검정 높이는 그대로이고, 부모가 빨강인 경우의 국소 red-red 위반만 처리하면 됩니다.
2. 삼촌이 빨강인 재색칠 경우와 삼촌이 검정인 회전 경우의 구분
   - **모범답변:** 빨강 삼촌이면 부모·삼촌을 함께 검정으로 바꿔 양쪽 검정 높이를 맞춘 채 위반을 조부모로 올릴 수 있습니다. 검정 삼촌이면 재색칠만으로 두 하위 경로를 맞출 수 없어 회전으로 구조와 색 관계를 함께 바꿉니다.
3. 안쪽 모양을 바깥쪽 모양으로 정규화하는 두 단계 회전
   - **모범답변:** 왼쪽 부모의 오른쪽 자식 같은 안쪽 모양은 먼저 부모를 왼쪽 회전해 삽입 노드를 바깥쪽 부모 위치로 바꿉니다. 그 뒤 부모·조부모를 재색칠하고 조부모를 반대 방향으로 회전하며, 오른쪽 경우는 대칭입니다.
4. 회전이 키 순서를 바꾸지 않으면서 높이와 색 관계만 바꾸는 이유
   - **모범답변:** 회전은 pivot, 자식, 그 사이 키 범위의 서브트리 위치만 재연결하며 중위 순서를 보존합니다. 키나 value는 수정하지 않고 부모-자식 깊이만 바꾸므로 색 보정에 필요한 구조 변화만 얻습니다.
5. 보정 루프가 조부모 방향으로 진행하므로 O(log n)인 근거
   - **모범답변:** 재색칠 경우 현재 위치가 한 번에 조부모로 올라가고 회전 경우에는 국소 위반을 해결하고 끝납니다. 따라서 이동 횟수는 트리 높이 O(h)이고 레드-블랙 트리에서는 h가 O(log n)입니다.

### 원본 확인 위치

- **Thread**: 07
- **커밋 메시지**: `feat(map): 레드-블랙 삽입 회전과 색 보정 구현`
- **파일**: `include/ft_map.hpp`
- **함수·컴포넌트**: `_is_red`, `_is_black`, `_rotate_left`, `_rotate_right`, `_insert_fixup`
- **관련된 다른 Thread**: 06의 BST 삽입·탐색, 08의 header sentinel 표현, 09의 삽입 예외 경계

<a id="rbt-02"></a>
## RBT-02 — [Thread 07 / `feat(map): 레드-블랙 삭제 보정 구현`] 검정 노드 삭제 뒤 검정 높이 결손 복구

### 면접 질문

레드-블랙 트리에서 노드를 구조적으로 제거한 뒤 언제 색 보정이 필요합니까?  
대체되어 실제 위치에서 빠져나간 노드의 원래 색을 왜 추적해야 하며, 보정 대상 `x`가 NULL일 수 있을 때 부모를 별도로 넘겨야 하는 이유를 설명해보세요.

꼬리 질문:
- NULL 자식을 검정으로 취급하면 형제의 색과 두 자식의 색에 따라 어떤 경우들이 생깁니까?
  - **모범답변:** 형제가 빨강인 경우, 검정 형제의 두 자식이 모두 검정인 경우, 검정 형제의 가까운 조카만 빨강인 경우, 먼 조카가 빨강인 최종 경우로 나뉩니다. NULL 형제와 NULL 조카도 검정으로 판정합니다.
- 빨강 형제를 먼저 회전해 검정 형제 경우로 바꾸는 이유는 무엇입니까?
  - **모범답변:** 빨강 형제가 있으면 parent는 검정이고 형제의 자식은 검정입니다. 형제를 검정, parent를 빨강으로 재색칠하고 parent를 회전하면 결손 위치는 그대로 두면서 새 형제가 검정인 표준 경우가 됩니다.
- 가까운 조카와 먼 조카를 구분하는 기준은 무엇입니까?
  - **모범답변:** 결손 x가 parent의 왼쪽이면 형제의 왼쪽 자식이 가까운 조카, 오른쪽이 먼 조카입니다. x가 오른쪽이면 방향이 반대입니다.
- 삭제 대상의 키-값을 다른 노드에 대입하지 않고 노드 자체를 이식하는 설계의 장점은 무엇입니까?
  - **모범답변:** `pair<const Key,T>`의 const key를 수정하지 않으며, successor를 가리키던 반복자가 같은 노드 주소와 값 객체를 계속 가리킬 수 있습니다. 링크와 색만 이동하므로 삭제 대상 외 노드의 identity를 보존합니다.

### 30초 모범 답변

실제로 제거된 위치의 노드가 빨강이면 검정 높이는 변하지 않지만, 검정이면 한 경로에 검정 하나가 부족해져 보정이 필요합니다. `x`가 NULL이면 그 객체에서 부모를 찾을 수 없으므로 부모를 별도로 추적해야 합니다. 빨강 형제는 재색칠과 회전으로 검정 형제 형태로 바꾸고, 형제의 두 자식이 검정이면 결손을 부모로 올립니다. 그렇지 않으면 가까운 조카를 이용해 먼 조카가 빨강인 형태로 정규화한 뒤 마지막 회전과 재색칠로 결손을 제거합니다. 좌우 경우는 완전히 대칭이어야 합니다.

### 답변 핵심 키워드

- removed original color
- black-height deficit
- NULL은 검정
- `x`와 별도 `parent`
- 빨강 형제 정규화
- 가까운·먼 조카
- 좌우 대칭
- 루트 검정

### 백지 구현

- **구현 목표**: BST 구조 삭제가 끝난 뒤 전달된 `x`와 그 부모에서 시작해 검정 높이 결손을 복구한다.
- **인터페이스 또는 함수 시그니처**: `erase_fixup(node*& root, node* x, node* parent)`를 구현하며 회전·색 판정 helper는 제공된 것으로 가정한다.
- **입력과 출력**: `x`는 실제 노드 또는 NULL일 수 있고, `parent`는 그 위치의 부모다. 함수는 같은 노드 집합의 색과 링크를 조정한다.
- **반드시 만족해야 할 조건**: BST 순서, 부모 링크, 검정 루트, 빨강-빨강 금지, 동일 검정 높이를 복구한다.
- **경계 조건**: 빈 트리, 루트만 남은 경우, NULL `x`, NULL 형제, 빨강 형제, 두 조카가 검정인 경우, 가까운 조카만 빨강인 경우, 먼 조카가 빨강인 경우를 좌우 대칭으로 처리한다.
- **실패 조건**: 할당이나 사용자 코드는 호출하지 않는다. 구조 삭제가 잘못된 입력은 사전 조건 위반이다.
- **필요한 제약**: 노드 검색·이식·파괴는 제외하고 보정 함수만 20~30분 안에 구현한다.

```cpp
struct node
{
    node* parent;
    node* left;
    node* right;
    bool red;
};

bool is_red(const node* current);
bool is_black(const node* current);
void rotate_left(node*& root, node* pivot);
void rotate_right(node*& root, node* pivot);

void erase_fixup(node*& root, node* x, node* parent)
{
    while (x != root && is_black(x))
    {
        if (x == (parent ? parent->left : NULL))
        {
            node* sibling = parent ? parent->right : NULL;
            if (is_red(sibling))
            {
                sibling->red = false;
                parent->red = true;
                rotate_left(root, parent);
                sibling = parent->right;
            }
            if (is_black(sibling ? sibling->left : NULL)
                && is_black(sibling ? sibling->right : NULL))
            {
                if (sibling)
                    sibling->red = true;
                x = parent; // 검정 결손을 한 단계 위로 올린다.
                parent = x ? x->parent : NULL;
            }
            else
            {
                if (is_black(sibling ? sibling->right : NULL))
                {
                    if (sibling && sibling->left)
                        sibling->left->red = false;
                    if (sibling)
                    {
                        sibling->red = true;
                        rotate_right(root, sibling);
                    }
                    sibling = parent ? parent->right : NULL;
                }
                if (sibling)
                    sibling->red = parent ? parent->red : false;
                if (parent)
                    parent->red = false;
                if (sibling && sibling->right)
                    sibling->right->red = false;
                if (parent)
                    rotate_left(root, parent);
                x = root;
                parent = NULL;
            }
        }
        else
        {
            node* sibling = parent ? parent->left : NULL;
            if (is_red(sibling))
            {
                sibling->red = false;
                parent->red = true;
                rotate_right(root, parent);
                sibling = parent->left;
            }
            if (is_black(sibling ? sibling->right : NULL)
                && is_black(sibling ? sibling->left : NULL))
            {
                if (sibling)
                    sibling->red = true;
                x = parent;
                parent = x ? x->parent : NULL;
            }
            else
            {
                if (is_black(sibling ? sibling->left : NULL))
                {
                    if (sibling && sibling->right)
                        sibling->right->red = false;
                    if (sibling)
                    {
                        sibling->red = true;
                        rotate_left(root, sibling);
                    }
                    sibling = parent ? parent->left : NULL;
                }
                if (sibling)
                    sibling->red = parent ? parent->red : false;
                if (parent)
                    parent->red = false;
                if (sibling && sibling->left)
                    sibling->left->red = false;
                if (parent)
                    rotate_right(root, parent);
                x = root;
                parent = NULL;
            }
        }
    }
    if (x)
        x->red = false;
}
```

### 구현 후 자가 검증

- 빨강 잎 삭제처럼 보정이 필요 없는 경로를 불필요하게 바꾸지 않는가?
- 검정 잎을 삭제해 `x == NULL`인 경우에도 NULL 역참조 없이 진행되는가?
- 형제가 NULL인 경우를 검정 자식 둘인 경우와 일관되게 처리하는가?
- 왼쪽 자식 결손과 오른쪽 자식 결손이 대칭적으로 동작하는가?
- 각 회전 뒤 `x`, `parent`, `sibling`의 의미를 다시 계산했는가?
- 완료 후 루트가 검정이고 빨강-빨강 간선이 없는가?
- 모든 root-to-NULL 경로의 검정 높이가 같은가?
- 삭제되지 않은 노드의 주소와 키-값이 유지되는가?
- 보정이 루트 방향으로만 진행해 O(log n)에 종료하는가?

### 구현 후 설명할 것

1. 삭제 대상의 색이 아니라 실제 제거된 위치의 원래 색을 추적하는 이유
   - **모범답변:** 두 자식 삭제에서는 successor가 대상 위치로 이동하고 실제로 비게 되는 곳은 successor의 옛 위치입니다. 검정 높이 변화는 그 위치에서 빠진 노드의 원래 색에 의해 결정되므로 원본도 `moved_was_red`를 따로 저장합니다.
2. NULL을 검정 잎으로 모델링하면서 부모를 별도 인자로 유지한 판단
   - **모범답변:** NULL을 검정으로 보면 색 판정 helper와 표준 경우 분류가 단순해집니다. 다만 NULL 객체에는 parent 필드가 없으므로 어느 쪽 자식 자리의 결손인지 알기 위해 구조 삭제 시 계산한 parent를 별도로 전달합니다.
3. 빨강 형제와 가까운 조카 경우를 표준 형태로 정규화하는 순서
   - **모범답변:** 빨강 형제는 회전해 검정 형제로 바꾸고, 먼 조카가 검정인데 가까운 조카가 빨강이면 형제를 반대 방향으로 회전해 먼 조카가 빨강인 형태로 만듭니다. 마지막 parent 회전과 재색칠이 결손을 제거합니다.
4. 키-값 대입보다 노드 이식이 반복자와 `const Key` 계약에 유리한 이유
   - **모범답변:** 이식은 const key에 대입하지 않고 삭제 대상 외 value 객체의 주소를 유지합니다. 따라서 살아남은 노드를 가리키는 반복자의 identity가 보존되고 사용자 값 대입의 예외도 삭제 구조 처리에 들어오지 않습니다.
5. 좌우 대칭 코드를 중복할지 helper로 추상화할지의 가독성 trade-off
   - **모범답변:** 원본처럼 두 분기를 명시하면 각 회전·조카 방향을 표준 알고리즘과 직접 대조하기 쉽지만 코드가 깁니다. 방향 helper로 줄일 수 있으나 인덱스나 방향 추상화가 오히려 디버깅 시 링크 의미를 숨길 수 있습니다.

### 원본 확인 위치

- **Thread**: 07
- **커밋 메시지**: `feat(map): 레드-블랙 삭제 보정 구현`
- **파일**: `include/ft_map.hpp`
- **함수·컴포넌트**: `_transplant`, `_erase_node`, `_erase_fixup`, `_is_red`, `_is_black`, `_rotate_left`, `_rotate_right`
- **관련된 다른 Thread**: 06의 기본 BST 삭제, 08의 header 갱신, 07의 무작위 invariant 검사
