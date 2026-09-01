# 실패 주입·구조 검증·복잡도 회귀

구현 코드 자체보다 테스트 설계 판단을 묻는 항목이다. 예외 경로, 내부 구조, 성능 퇴화를 서로 다른 관측 수단으로 검증한다.

<a id="test-01"></a>
## TEST-01 — [Thread 05 / `test(vector): 삽입 복사·대입·할당 실패 sweep`] 결정적 실패 주입과 객체·할당 자원 회계

### 면접 질문

예외 안전성을 정상 입력 몇 개와 한 번의 실패만으로 검증하기 어려운 이유는 무엇입니까?  
복사·대입·할당·비교의 N번째 호출에 실패를 주입하고, 각 실패 지점을 sweep하면서 살아 있는 객체와 outstanding allocation을 어떻게 검증했는지 설명해보세요.

꼬리 질문:
- 단순 생성·소멸 횟수만 세는 것보다 살아 있는 주소 집합이 유용한 이유는 무엇입니까?
  - **모범답변:** 총횟수는 이중 소멸 하나와 누락 하나가 상쇄되어 같아질 수 있습니다. live 주소 집합은 복사 원본과 대입 대상이 실제로 생성된 객체인지, 소멸 주소가 정확히 한 번 등록 해제되는지를 개별 identity로 검사합니다.
- 실패 지점을 0부터 증가시키다가 성공 경로까지 확인해야 하는 이유는 무엇입니까?
  - **모범답변:** 몇 번의 자원 획득이 있는지 구현마다 달라 마지막 실패 지점을 미리 알기 어렵습니다. 최초 성공까지 진행해야 모든 실제 실패 경계를 통과했고, 주입을 벗어난 정상 commit 경로도 동작한다는 것을 함께 확인할 수 있습니다.
- 강한 보장과 기본 보장을 테스트 assertion에서 어떻게 구분합니까?
  - **모범답변:** 강한 보장 연산은 예외 뒤 원소열·size·capacity 또는 tree·정책이 baseline과 같은지 비교합니다. 기본 보장은 값 일부 변경은 허용하되 객체 수명, size와 실제 생성 수, 구조 invariant, 누수 없음, 재사용 가능성을 검사합니다.
- comparator 실패, allocator 실패, 값 복사 실패는 서로 어떤 다른 소유권 경계를 통과합니까?
  - **모범답변:** comparator는 보통 자원 획득 전 탐색·정책 단계, allocator는 원시 블록 획득 단계, 값 복사는 블록 안에서 객체 수명을 시작하거나 기존 객체를 변경하는 단계에서 실패합니다. 각각 정리할 자원이 없음·블록 rollback·생성 접두부 rollback이라는 다른 경계를 시험합니다.

### 30초 모범 답변

예외는 루프의 첫 복사, 중간 복사, 마지막 commit 직전처럼 지점마다 다른 정리 경로를 타므로 한 번만 던져서는 누락을 찾기 어렵습니다. 값 형식은 살아 있는 주소와 복사·대입 시도 수를 기록하고, allocator는 할당 시도와 outstanding block을 기록합니다. 실패 지점을 0부터 하나씩 옮겨 각 catch 뒤 계약된 상태와 자원 회계를 확인하고, 더 이상 예외가 나지 않는 성공 지점까지 진행합니다. 강한 보장인 연산은 원래 값을 비교하고, 기본 보장만 가능한 경로는 유효한 수명·invariant·누수 없음만 확인합니다.

### 답변 핵심 키워드

- deterministic failure injection
- fail-at-N sweep
- live address set
- copy/assignment counters
- outstanding allocations
- strong vs basic guarantee
- rollback verification
- eventual success

### 백지 구현

- **구현 목표**: 테스트 대상 연산의 각 복사 또는 할당 지점에서 결정적으로 실패시키고 자원 누수와 상태 계약을 검사하는 작은 sweep harness를 작성한다.
- **인터페이스 또는 함수 시그니처**: `tracked_value`, `tracking_allocator<T>`, `run_failure_sweep(Operation op, AssertAfterFailure check)`의 최소 기능을 구현한다.
- **입력과 출력**: 연산 callable과 실패 뒤 상태 검사 callable을 받아, 실패 지점을 순회하며 assertion 실패 또는 최초 성공 결과를 보고한다.
- **반드시 만족해야 할 조건**: 각 실행은 독립 상태에서 시작하고, catch 뒤 살아 있는 객체·할당 블록·컨테이너 invariant를 확인하며, 성공 경로도 한 번 이상 실행한다.
- **경계 조건**: 첫 번째 시도 실패, 마지막 자원 획득 직후 실패, 실패 없이 성공, 빈 입력 연산을 처리한다.
- **실패 조건**: 값 복사·대입 또는 allocator의 N번째 호출이 테스트 예외를 던진다. 예기치 않은 예외는 별도로 전파한다.
- **필요한 제약**: 한 종류의 값 실패와 한 종류의 allocator 실패만 구현하고 20~30분 안에 완성한다.

```cpp
struct failure_control
{
    int attempts;
    int fail_at;
};

class injected_failure : public std::exception
{
};

class tracked_value
{
public:
    tracked_value(int value = 0);
    tracked_value(const tracked_value& other);
    tracked_value& operator=(const tracked_value& other);
    ~tracked_value();

    static void reset();
    static bool resources_are_clean();
    static failure_control failures;
};

failure_control tracked_value::failures = { 0, -1 };

template <class Operation, class Check>
void run_failure_sweep(Operation operation, Check check)
{
    for (int fail_at = 0; ; ++fail_at)
    {
        tracked_value::reset();
        tracked_value::failures.attempts = 0;
        tracked_value::failures.fail_at = fail_at;
        bool injected = false;
        try
        {
            // operation은 매 호출마다 독립 컨테이너를 만들고 scope 안에서 정리한다.
            operation();
        }
        catch (const injected_failure&)
        {
            injected = true;
        }
        catch (...)
        {
            tracked_value::failures.fail_at = -1;
            throw; // 예상한 주입 예외만 sweep가 소비한다.
        }

        tracked_value::failures.fail_at = -1;
        check(injected, fail_at);
        if (!tracked_value::resources_are_clean())
            throw std::logic_error("failure sweep leaked a resource");
        if (!injected)
            break; // 모든 fail point 뒤의 성공 경로까지 확인했다.
    }
}
```

### 구현 후 자가 검증

- 각 sweep 반복 전에 카운터와 실패 지점, live set이 독립적으로 초기화되는가?
- 생성되지 않은 주소를 복사하거나 두 번 소멸하면 별도 오류로 감지하는가?
- 할당 성공 뒤 생성 실패가 나면 outstanding block이 원래 값으로 돌아오는가?
- 첫 번째·중간·마지막 복사 실패를 모두 통과하는가?
- 강한 보장 대상 연산은 실패 뒤 원소열·size·capacity 또는 tree 상태가 원래와 같은가?
- 기본 보장 대상 연산은 값이 일부 바뀔 수 있음을 허용하되 객체 수명과 구조 invariant를 확인하는가?
- 실패 뒤 테스트 대상 객체를 다시 사용할 수 있는가?
- 최종 scope 종료 뒤 live object와 outstanding block이 모두 0인가?
- 성공 가능한 연산에서 sweep가 무한히 실패 지점만 늘리지 않고 성공 경로를 확인하는가?
- 실패 진단에 연산 이름과 fail index가 포함되는가?

### 구현 후 설명할 것

1. 예외 지점 하나가 아니라 모든 자원 획득 경계를 sweep해야 하는 이유
   - **모범답변:** 첫 복사 실패는 정리할 객체가 없지만 중간 실패는 접두부가 있고, 마지막 실패는 commit 직전 상태를 만듭니다. 지점마다 rollback 범위가 달라 누락은 특정 N에서만 드러날 수 있습니다.
2. live 주소 집합으로 미생성 객체 접근과 이중 소멸을 잡는 방법
   - **모범답변:** 생성 성공 시 this를 집합에 넣고 복사·대입 전에 원본과 대상을 조회합니다. 소멸 시 erase 결과가 1인지 확인하면 등록되지 않은 주소 소멸과 두 번째 소멸을 즉시 구분할 수 있습니다.
3. 연산별 강한 보장·기본 보장 차이를 assertion에 반영한 판단
   - **모범답변:** 임시 저장소를 쓰는 assign·재할당은 실패 뒤 전체 baseline 동등성을 요구합니다. 사용자 대입이 섞인 제자리 insert는 원본도 새 꼬리 정리와 size·수명 일치만 보장하므로 값 전체 동일성 대신 기본 invariant와 후속 사용을 검사합니다.
4. 값 실패·allocator 실패·comparator 실패를 분리해 원인을 좁히는 구조
   - **모범답변:** 원본 테스트는 복사·대입 카운터, allocate 카운터, comparator 호출 카운터를 각각 초기화하고 하나만 활성화합니다. 이렇게 하면 실패가 비교 전 소유권, 블록 소유권, 객체 수명 중 어느 경계를 깨뜨렸는지 진단할 수 있습니다.
5. 결정적 실패 주입이 sanitizer와 서로 보완하는 범위
   - **모범답변:** 실패 주입은 어떤 호출에서 던질지와 예외 뒤 논리 상태를 결정적으로 검증합니다. sanitizer는 실행된 경로의 use-after-free·잘못된 접근 같은 메모리 오류를 잡으므로, 주입으로 희귀 경로를 열고 sanitizer로 저수준 위반을 관찰하면 서로 보완됩니다.

### 원본 확인 위치

- **대표 Thread**: 05
- **대표 커밋 메시지**: `test(vector): 생성·대입·크기 변경 실패 주입`, `test(vector): 삽입 복사·대입·할당 실패 sweep`
- **파일**: `tests/test_vector_exceptions.cpp`
- **함수·클래스·컴포넌트**: `tracked_value`, `tracking_allocator`, 복사·대입·할당 실패 제어 상태
- **관련된 다른 Thread**: 09의 `tests/test_map_exceptions.cpp`, `tests/test_map_policy_exceptions.cpp`; 커밋 `test(map): 비교·할당 실패 시 노드 소유권 검증`, `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증`

<a id="test-02"></a>
## TEST-02 — [Thread 07 / `test(map): 무작위 연산마다 레드-블랙 불변식 검증`] 구조 invariant 검사기와 재현 가능한 무작위 차등 테스트

### 면접 질문

`ft::map`의 공개 결과를 `std::map`과 비교하는 것만으로 레드-블랙 트리 구현을 충분히 검증할 수 없는 이유는 무엇입니까?  
white-box inspector가 검사해야 할 header, BST, 부모 링크, 색, 검정 높이, 노드 수 조건과 고정 seed 무작위 테스트의 재현 전략을 설명해보세요.

꼬리 질문:
- NULL 잎의 검정 높이를 어떤 기준값으로 두면 재귀 계산이 단순해집니까?
  - **모범답변:** 원본 inspector는 NULL 잎의 black_height를 1로 둡니다. 내부 노드는 좌우 결과가 같은지 확인한 뒤 자신이 검정이면 1을 더해 일관된 재귀식을 사용합니다.
- 하위 트리의 키 범위를 단순 직전 키가 아니라 lower·upper bound로 전달하는 이유는 무엇입니까?
  - **모범답변:** 부모와 직접 자식만 비교하면 깊은 후손이 조상 범위를 넘어간 오류를 놓칠 수 있습니다. 왼쪽에는 기존 lower와 현재 key를 upper로, 오른쪽에는 현재 key를 lower와 기존 upper로 전달하면 모든 조상 제약을 누적합니다.
- 공개 결과는 맞지만 내부 invariant가 깨진 상태가 왜 위험합니까?
  - **모범답변:** 현재 중위 순회는 우연히 맞아도 잘못된 parent가 다음 반복자 증감을 망가뜨리고, red-red나 검정 높이 위반은 이후 삭제 보정 실패나 선형 높이로 이어질 수 있습니다.
- 실패 시 seed, step, 연산 prefix를 모두 출력하는 이유는 무엇입니까?
  - **모범답변:** seed만으로 생성 순서를 재현하고 step으로 최초 위반 시점을 좁히며, prefix로 실제로 적용된 insert·erase·swap 등을 그대로 재실행할 수 있습니다. 무작위 실패를 결정적인 최소 디버깅 입력으로 바꾸는 정보입니다.

### 30초 모범 답변

공개 원소열이 우연히 맞아도 부모 링크나 색, 검정 높이가 깨져 다음 삭제에서 실패하거나 복잡도가 선형으로 악화될 수 있습니다. inspector는 빈 header 표현, 루트와 극값 링크, 루트 검정, 부모 링크, 비교자 기준 BST 범위, 빨강-빨강 금지, 좌우 검정 높이 일치, 도달 노드 수와 size 일치를 재귀적으로 검사합니다. 무작위 테스트는 고정 seed로 삽입·삭제·복사·대입·swap·clear를 섞고 매 단계 `std::map` 결과와 내부 invariant를 함께 확인하며, 실패 시 연산 prefix를 남겨 그대로 재현합니다.

### 답변 핵심 키워드

- white-box inspector
- header invariant
- BST lower/upper bounds
- parent consistency
- red-red prohibition
- equal black height
- reachable count equals size
- fixed-seed differential test

### 백지 구현

- **구현 목표**: 레드-블랙 트리 한 개를 재귀적으로 검사해 첫 위반 메시지와 node count·height·black height를 반환한다.
- **인터페이스 또는 함수 시그니처**: `validation validate(const map_type&)`와 내부 `validate_subtree(current, expected_parent, lower, upper)`를 구현한다.
- **입력과 출력**: 테스트용 friend inspector가 map 내부 노드를 읽고 `valid`, `message`, `node_count`, `height`, `black_height`를 반환한다.
- **반드시 만족해야 할 조건**: header 표현, 루트 검정, root parent, min/max, BST 범위, parent 링크, red-red, 동일 검정 높이, reachable count를 모두 확인한다.
- **경계 조건**: 빈 트리, 한 노드, NULL 자식, 한쪽으로 깊은 구조, header가 자식으로 잘못 연결된 구조를 처리한다.
- **실패 조건**: 첫 위반을 발견하면 구체적 메시지와 함께 실패 결과를 반환한다. 검사 중 구조를 변경하지 않는다.
- **필요한 제약**: 키·색·링크를 읽을 수 있는 inspector 접근이 제공되며 20~30분 안에 구현한다.

```cpp
struct validation
{
    bool valid;
    std::string message;
    std::size_t node_count;
    std::size_t height;
    std::size_t black_height;
};

// 축소 skeleton의 validate_subtree 시그니처를 유지하기 위한 테스트 전용 문맥이다.
// 실제 원본 inspector는 map 참조를 재귀 helper의 인자로 직접 전달한다.
const map_type* validation_target = NULL;

validation fail_validation(const std::string& message)
{
    validation result = { false, message, 0, 0, 0 };
    return result;
}

validation validate(const map_type& values)
{
    const node_base* header = &values._header;
    if (!header->is_header)
        return fail_validation("header flag is not set");
    if (header->red)
        return fail_validation("header must be black");
    if (values._size == 0)
    {
        if (header->parent != NULL)
            return fail_validation("empty header has a root");
        if (header->left != header || header->right != header)
            return fail_validation("empty extrema do not self-reference");
        validation empty = { true, "", 0, 0, 1 };
        return empty;
    }

    const node_base* root = header->parent;
    if (root == NULL || root->parent != header)
        return fail_validation("invalid root/header link");
    if (root->red)
        return fail_validation("root must be black");
    const node_base* minimum = root;
    while (minimum->left)
        minimum = minimum->left;
    const node_base* maximum = root;
    while (maximum->right)
        maximum = maximum->right;
    if (minimum != header->left || maximum != header->right)
        return fail_validation("header extrema are stale");

    validation_target = &values;
    validation result = validate_subtree(root, header, NULL, NULL);
    validation_target = NULL;
    if (result.valid && result.node_count != values._size)
        return fail_validation("reachable node count differs from size");
    return result;
}

validation validate_subtree(
    const node_base* current,
    const node_base* expected_parent,
    const key_type* lower,
    const key_type* upper)
{
    if (current == NULL)
    {
        validation leaf = { true, "", 0, 0, 1 };
        return leaf;
    }
    if (current->is_header)
        return fail_validation("header is reachable as a child");
    if (current->parent != expected_parent)
        return fail_validation("child and parent links disagree");

    const key_type& key = map_type::_value(current).first;
    if (lower && !validation_target->_comp(*lower, key))
        return fail_validation("key is not greater than lower bound");
    if (upper && !validation_target->_comp(key, *upper))
        return fail_validation("key is not less than upper bound");
    if (current->red
        && ((current->left && current->left->red)
            || (current->right && current->right->red)))
        return fail_validation("red node has a red child");

    validation left = validate_subtree(
        current->left, current, lower, &key);
    if (!left.valid)
        return left;
    validation right = validate_subtree(
        current->right, current, &key, upper);
    if (!right.valid)
        return right;
    if (left.black_height != right.black_height)
        return fail_validation("black heights differ");

    validation result = { true, "",
        left.node_count + right.node_count + 1,
        std::max(left.height, right.height) + 1,
        left.black_height + (current->red ? 0 : 1) };
    return result;
}
```

### 구현 후 자가 검증

- 빈 map의 header parent·left·right와 검정 높이 기준이 올바른가?
- 비어 있지 않은 map에서 root parent가 header이고 root가 검정인지 확인하는가?
- header left·right가 실제 minimum·maximum과 일치하는가?
- 각 자식의 parent가 호출자가 기대한 노드와 일치하는가?
- 모든 키가 comparator 기준 lower보다 크고 upper보다 작은가?
- 빨강 노드의 자식 중 빨강이 하나라도 있으면 실패하는가?
- NULL 잎까지 포함한 좌우 검정 높이가 같은가?
- 도달 가능한 값 노드 수가 `size()`와 같은가?
- cycle이나 header-as-child처럼 재귀가 끝나지 않을 구조를 진단할 수 있는가?
- 고정 seed 테스트가 실패한 seed·step·연산 prefix를 다시 실행 가능한 형태로 남기는가?

### 구현 후 설명할 것

1. 공개 결과 차등 비교와 내부 invariant 검사가 잡는 버그 범위의 차이
   - **모범답변:** std::map과 원소열·조회 결과를 비교하면 외부 의미 오류를 잡지만, 아직 결과에 드러나지 않은 색·parent·header 손상은 놓칠 수 있습니다. white-box 검사는 다음 연산과 복잡도 보장을 깨뜨릴 잠복 구조 오류를 직접 찾습니다.
2. 재귀 반환값에 node count·height·black height를 함께 누적한 이유
   - **모범답변:** 한 번의 후위 순회에서 도달 노드 수와 size 일치, 실제 높이, 좌우 검정 높이 동등성을 함께 계산할 수 있습니다. 같은 트리를 조건마다 반복 순회하지 않고 실패가 발생한 하위 트리에서 즉시 전파합니다.
3. lower·upper key 경계로 전체 BST 순서를 검증하는 방식
   - **모범답변:** 현재 key는 comparator 기준 lower보다 크고 upper보다 작아야 합니다. 왼쪽 재귀에는 현재 key를 새 upper로, 오른쪽에는 새 lower로 넘겨 모든 후손이 전체 조상 범위 안에 있는지 확인합니다.
4. friend inspector라는 테스트 전용 캡슐화 예외의 trade-off
   - **모범답변:** 공개 API로 볼 수 없는 색·링크·header를 검사해 강한 구조 검증을 얻는 대신 내부 표현과 테스트가 결합됩니다. 원본은 일반 사용자에게 노출하지 않고 특정 `detail::map_inspector`만 friend로 두어 예외 범위를 제한합니다.
5. 고정 seed와 연산 로그로 무작위 테스트를 결정적으로 재현하는 방법
   - **모범답변:** 같은 seed의 의사난수 생성기로 항상 같은 연산·키 순서를 만들고 각 step의 연산을 로그에 누적합니다. 실패 메시지의 seed와 prefix를 다시 실행하면 동일한 첫 invariant 위반을 재현할 수 있습니다.

### 원본 확인 위치

- **Thread**: 07
- **커밋 메시지**: `test(map): 무작위 연산마다 레드-블랙 불변식 검증`
- **파일**: `tests/support/map_inspector.hpp`, `tests/test_map_randomized.cpp`, `include/ft_map.hpp`
- **함수·클래스·컴포넌트**: `map_validation`, `map_inspector::validate`, `_validate_subtree`, `validate_map`, `test_fixed_erasure_sequences`, `test_repeated_root_erasure`, `test_randomized_differential`, `random_seed`, `operation_log`
- **관련된 다른 Thread**: 07의 삽입·삭제 보정, 08의 header sentinel, 09의 copy·swap 소유권

<a id="test-03"></a>
## TEST-03 — [Thread 07 / `perf(map): 높이와 비교 횟수 회귀 상한 추가`] 구조 높이와 비교자 호출 수로 복잡도 회귀 감지

### 면접 질문

기능 테스트가 모두 통과해도 `map`이 사실상 연결 리스트처럼 퇴화할 수 있습니다.  
오름차순·내림차순·고정 난수 입력에서 트리 높이와 비교자 호출 수를 계측해 O(log n) 기대를 회귀 테스트로 만드는 방법을 설명해보세요.

꼬리 질문:
- 레드-블랙 트리의 높이 상한과 `ceil_log2`를 어떤 관계로 사용합니까?
  - **모범답변:** 원본은 n개 노드에 `logarithm = ceil_log2(n + 1)`을 계산하고 height limit을 `2 * logarithm`으로 둡니다. 이는 레드-블랙 높이가 로그 크기의 두 배 이내라는 이론을 회귀 상한으로 옮긴 것입니다.
- wall-clock 시간보다 비교자 호출 수를 세는 것이 안정적인 이유는 무엇입니까?
  - **모범답변:** 실행 시간은 CPU·최적화·시스템 부하에 따라 흔들립니다. 비교 호출 수는 탐색 경로와 알고리즘 선택에 직접 연결된 결정적 지표라 같은 입력에서 재현 가능하고 선형 탐색 회귀를 안정적으로 드러냅니다.
- 삽입 전체 비교 횟수와 단일 조회 비교 횟수는 각각 어떤 차원의 상한을 가져야 합니까?
  - **모범답변:** n번 삽입 전체는 각 O(log n)이 누적되어 O(n log n) 예산이어야 합니다. 한 번의 find는 한 경로에서 키를 양방향 비교하므로 O(height), 즉 O(log n) 상한을 가져야 합니다.
- 상한을 정확한 구현 상수에 너무 가깝게 두면 어떤 문제가 생깁니까?
  - **모범답변:** 올바른 다른 회전 모양, 비교식 배치, 작은 n의 경계 차이도 실패해 테스트가 구현 세부에 과적합됩니다. 퇴화는 확실히 잡되 합리적인 상수 차이는 허용하는 넉넉한 예산이 회귀 테스트에 적합합니다.

### 30초 모범 답변

실행 시간은 머신과 부하에 흔들리므로 구조 높이와 비교자 호출처럼 알고리즘에 직접 연결된 지표를 셉니다. 오름차순·내림차순·고정 난수 순서로 같은 수의 키를 넣고 inspector로 높이를 측정하며, `counting_less`로 삽입과 조회의 비교 횟수를 기록합니다. 상한은 레드-블랙 높이가 로그 크기에 비례한다는 이론에서 넉넉하게 잡아 구현 세부 상수에는 덜 민감하게 합니다. 이 테스트는 복잡도 증명은 아니지만 회전 누락이나 선형 탐색 회귀를 안정적으로 잡습니다.

### 답변 핵심 키워드

- structural metric
- counting comparator
- height bound
- `ceil_log2`
- ascending/descending/fixed-random
- comparison budget
- generous threshold
- regression, not proof

### 백지 구현

- **구현 목표**: 서로 다른 입력 순서로 트리를 만들고 높이와 비교자 호출 수가 로그 기반 예산 안에 있는지 검사한다.
- **인터페이스 또는 함수 시그니처**: `counting_less`, `ceil_log2`, `check_scenario(label, keys)`를 구현한다.
- **입력과 출력**: 서로 다른 순서의 unique key 배열을 받아 삽입하고, 높이·전체 삽입 비교·선택 조회 비교를 계측해 assertion한다.
- **반드시 만족해야 할 조건**: 비교 카운터를 측정 구간마다 초기화하고, 높이와 비교 횟수 상한을 입력 크기의 로그 함수로 표현한다.
- **경계 조건**: 0개, 1개, 2개 키와 오름차순·내림차순·고정 seed shuffle을 처리한다.
- **실패 조건**: 높이나 비교 횟수가 정한 예산을 넘으면 label·n·실측값·상한을 출력한다.
- **필요한 제약**: 트리 높이는 제공된 inspector로 읽고, 상한 상수는 특정 회전 모양에 종속되지 않게 넉넉히 선택한다. 15~25분 범위다.

```cpp
class counting_less
{
public:
    explicit counting_less(std::size_t* comparisons);
    bool operator()(int lhs, int rhs) const;

private:
    std::size_t* comparisons_;
};

counting_less::counting_less(std::size_t* comparisons)
    : comparisons_(comparisons)
{
}

bool counting_less::operator()(int lhs, int rhs) const
{
    if (comparisons_)
        ++(*comparisons_);
    return lhs < rhs;
}

std::size_t ceil_log2(std::size_t value)
{
    std::size_t exponent = 0;
    std::size_t power = 1;
    while (power < value)
    {
        power *= 2;
        ++exponent;
    }
    return exponent;
}

void check_scenario(
    const std::string& label,
    const std::vector<int>& keys)
{
    std::size_t comparisons = 0;
    measured_map values((counting_less(&comparisons)));
    for (std::size_t i = 0; i < keys.size(); ++i)
        values.insert(ft::make_pair(keys[i], keys[i] * 3));
    const std::size_t insertion_comparisons = comparisons;

    map_validation state = map_inspector<measured_map>::validate(values);
    require(state.valid, label + " invariant: " + state.message);
    const std::size_t logarithm = ceil_log2(values.size() + 1);
    const std::size_t height_limit = 2 * logarithm;
    require(state.height <= height_limit, label + " height limit");

    const std::size_t insertion_limit =
        values.size() * (4 * logarithm + 4);
    require(insertion_comparisons <= insertion_limit,
        label + " insertion comparison limit");

    for (std::size_t i = 0; i < keys.size(); ++i)
    {
        comparisons = 0; // 단일 조회마다 별도 최악값을 측정한다.
        require(values.find(keys[i]) != values.end(), label + " existing key");
        require(comparisons <= 2 * state.height + 2,
            label + " find comparison limit");
    }
    const int missing[] = { -1, static_cast<int>(keys.size()) };
    for (std::size_t i = 0; i < 2; ++i)
    {
        comparisons = 0;
        require(values.find(missing[i]) == values.end(), label + " missing key");
        require(comparisons <= 2 * state.height + 2,
            label + " missing comparison limit");
    }
}
```

### 구현 후 자가 검증

- `ceil_log2`가 0·1·2와 2의 거듭제곱 전후에서 정의한 계약대로 동작하는가?
- 오름차순·내림차순 입력에서도 높이가 선형으로 증가하지 않는가?
- 고정 난수 순서가 실행마다 동일하게 재현되는가?
- 삽입 비교 카운터에 테스트용 검증 비교가 섞이지 않도록 구간을 분리했는가?
- 조회마다 카운터를 초기화해 단일 연산의 최악값을 볼 수 있는가?
- 존재하는 키와 없는 키의 조회를 모두 계측하는가?
- n이 작은 경우 로그 상한이 0이 되어 정상 연산을 잘못 실패시키지 않는가?
- 실패 메시지에 시나리오·n·height·comparisons·bound가 포함되는가?
- 상한이 구현의 정확한 회전 횟수에 고정되지 않고 퇴화만 확실히 잡는가?
- 테스트가 O(n log n) 또는 허용 가능한 검사 비용 안에서 끝나는가?

### 구현 후 설명할 것

1. wall-clock 대신 높이·비교 횟수를 선택한 안정성 근거
   - **모범답변:** 두 지표는 머신 속도와 순간 부하보다 자료구조와 탐색 알고리즘에 직접 종속됩니다. 고정 입력에서 결정적으로 재현되어 성능 회귀 원인을 구조 높이와 탐색 비교로 나눠 볼 수 있습니다.
2. 레드-블랙 높이 이론을 회귀 상한으로 옮기는 방식
   - **모범답변:** 원본은 `2 * ceil_log2(n + 1)`을 높이 상한으로 사용하고, 삽입 비교는 `n * (4 * log + 4)`처럼 넉넉한 O(n log n) 예산으로 둡니다. 정확한 회전 모양이 아니라 로그 차수 위반을 잡는 기준입니다.
3. 정렬 입력 두 방향과 고정 난수 입력을 모두 사용하는 이유
   - **모범답변:** 오름차순과 내림차순은 단순 BST를 각각 한쪽으로 퇴화시키는 적대 입력이며 좌우 대칭 보정 누락도 드러냅니다. 고정 난수는 다양한 회전·분기 조합을 재현 가능하게 넓게 탐색합니다.
4. 전체 삽입 예산과 단일 조회 예산을 분리한 판단
   - **모범답변:** 삽입 counter는 n번 연산의 누적이므로 O(n log n)을 봐야 하고, find는 매 호출 전 counter를 0으로 만들어 O(height)를 확인해야 합니다. 섞으면 한 번의 선형 조회가 총합에 가려질 수 있습니다.
5. 복잡도 회귀 테스트가 수학적 증명을 대체하지 않는 한계
   - **모범답변:** 선택한 크기와 입력에서 상한을 넘지 않았다는 경험적 확인일 뿐 모든 n과 모든 연산열을 증명하지 않습니다. 레드-블랙 invariant와 알고리즘 분석이 최악 복잡도의 근거이고, 계측 테스트는 구현 회귀를 조기에 찾는 보조 수단입니다.

### 원본 확인 위치

- **Thread**: 07
- **커밋 메시지**: `perf(map): 높이와 비교 횟수 회귀 상한 추가`
- **파일**: `tests/test_complexity.cpp`, `tests/support/map_inspector.hpp`
- **함수·클래스·컴포넌트**: `counting_less`, `ceil_log2`, `make_ascending`, `make_descending`, `make_fixed_random`, `check_scenario`, `map_inspector`
- **관련된 다른 Thread**: 07의 레드-블랙 균형화와 invariant 검사
