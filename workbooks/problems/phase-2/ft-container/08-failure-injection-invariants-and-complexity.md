# 실패 주입·구조 검증·복잡도 회귀

구현 코드 자체보다 테스트 설계 판단을 묻는 항목이다. 예외 경로, 내부 구조, 성능 퇴화를 서로 다른 관측 수단으로 검증한다.

<a id="test-01"></a>
## TEST-01 — [Thread 05 / `test(vector): 삽입 복사·대입·할당 실패 sweep`] 결정적 실패 주입과 객체·할당 자원 회계

### 면접 질문

예외 안전성을 정상 입력 몇 개와 한 번의 실패만으로 검증하기 어려운 이유는 무엇입니까?  
복사·대입·할당·비교의 N번째 호출에 실패를 주입하고, 각 실패 지점을 sweep하면서 살아 있는 객체와 outstanding allocation을 어떻게 검증했는지 설명해보세요.

꼬리 질문:
- 단순 생성·소멸 횟수만 세는 것보다 살아 있는 주소 집합이 유용한 이유는 무엇입니까?
- 실패 지점을 0부터 증가시키다가 성공 경로까지 확인해야 하는 이유는 무엇입니까?
- 강한 보장과 기본 보장을 테스트 assertion에서 어떻게 구분합니까?
- comparator 실패, allocator 실패, 값 복사 실패는 서로 어떤 다른 소유권 경계를 통과합니까?

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

class tracked_value
{
public:
    tracked_value(int value = 0);
    tracked_value(const tracked_value& other);
    tracked_value& operator=(const tracked_value& other);
    ~tracked_value();

    static void reset();
    static bool resources_are_clean();
};

template <class Operation, class Check>
void run_failure_sweep(Operation operation, Check check)
{
    // 직접 구현
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
2. live 주소 집합으로 미생성 객체 접근과 이중 소멸을 잡는 방법
3. 연산별 강한 보장·기본 보장 차이를 assertion에 반영한 판단
4. 값 실패·allocator 실패·comparator 실패를 분리해 원인을 좁히는 구조
5. 결정적 실패 주입이 sanitizer와 서로 보완하는 범위

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
- 하위 트리의 키 범위를 단순 직전 키가 아니라 lower·upper bound로 전달하는 이유는 무엇입니까?
- 공개 결과는 맞지만 내부 invariant가 깨진 상태가 왜 위험합니까?
- 실패 시 seed, step, 연산 prefix를 모두 출력하는 이유는 무엇입니까?

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

validation validate(const map_type& values)
{
    // 직접 구현
}

validation validate_subtree(
    const node_base* current,
    const node_base* expected_parent,
    const key_type* lower,
    const key_type* upper)
{
    // 직접 구현
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
2. 재귀 반환값에 node count·height·black height를 함께 누적한 이유
3. lower·upper key 경계로 전체 BST 순서를 검증하는 방식
4. friend inspector라는 테스트 전용 캡슐화 예외의 trade-off
5. 고정 seed와 연산 로그로 무작위 테스트를 결정적으로 재현하는 방법

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
- wall-clock 시간보다 비교자 호출 수를 세는 것이 안정적인 이유는 무엇입니까?
- 삽입 전체 비교 횟수와 단일 조회 비교 횟수는 각각 어떤 차원의 상한을 가져야 합니까?
- 상한을 정확한 구현 상수에 너무 가깝게 두면 어떤 문제가 생깁니까?

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

std::size_t ceil_log2(std::size_t value)
{
    // 직접 구현
}

void check_scenario(
    const std::string& label,
    const std::vector<int>& keys)
{
    // 직접 구현
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
2. 레드-블랙 높이 이론을 회귀 상한으로 옮기는 방식
3. 정렬 입력 두 방향과 고정 난수 입력을 모두 사용하는 이유
4. 전체 삽입 예산과 단일 조회 예산을 분리한 판단
5. 복잡도 회귀 테스트가 수학적 증명을 대체하지 않는 한계

### 원본 확인 위치

- **Thread**: 07
- **커밋 메시지**: `perf(map): 높이와 비교 횟수 회귀 상한 추가`
- **파일**: `tests/test_complexity.cpp`, `tests/support/map_inspector.hpp`
- **함수·클래스·컴포넌트**: `counting_less`, `ceil_log2`, `make_ascending`, `make_descending`, `make_fixed_random`, `check_scenario`, `map_inspector`
- **관련된 다른 Thread**: 07의 레드-블랙 균형화와 invariant 검사
