# 스택 모델·Checker·정렬·장애 주입 워크북

이 문서는 `push_swap` 계열 작업에서 확인된 공통 연산 모델, 입력 정규화, 정렬 전략, checker 경계, 결정적 실패 테스트를 묶는다. 명령 문자열을 외우는 것보다 상태 전이와 검증 계약을 우선한다.

---

<a id="s-01"></a>
## S-01. [Thread 23 / `test(sort): 생성 명령의 정렬 결과를 독립 검증`] 생성기–Checker 공유 모델과 독립 oracle

### 면접 질문

명령 생성기와 checker가 같은 `swap`, `push`, `rotate` 구현을 공유하면 어떤 장점이 있습니까? 반대로 공유 구현에 같은 버그가 있으면 잘못된 명령을 checker도 정답으로 인정할 수 있는데, 독립 Python replay 모델은 이 상관 오류를 어떻게 줄입니까?

꼬리 질문:

- 생성기와 checker가 공유해야 하는 것은 명령의 의미입니까, 파싱·출력까지 포함한 전체 코드입니까?
- operation 함수가 상태 변경과 명령 출력을 동시에 하면 checker에서 재사용할 때 어떤 문제가 생깁니까?
- 독립 oracle도 같은 테스트 데이터만 사용하면 놓칠 수 있는 오류는 무엇입니까?
- 최종 정렬 여부만 확인하는 테스트와 매 operation invariant를 확인하는 테스트는 어떤 버그를 각각 찾습니까?
- 이미 정렬된 입력에서 빈 명령 스트림을 기대하는 것은 기능 계약입니까, 성능 계약입니까?

### 30초 모범 답변

생성기와 checker가 같은 순수 operation 모델을 공유하면 명령 의미가 한곳에 정의되어 두 프로그램의 불일치를 줄일 수 있습니다. 다만 그 구현이 잘못되면 둘이 함께 틀릴 수 있으므로, 테스트에서는 별도 언어와 별도 자료구조로 명령 스트림을 다시 재생해 원래 값이 정렬되고 보조 스택이 비었는지 확인합니다. operation의 상태 변경과 명령 출력은 분리하거나 출력 플래그로 제어해 checker가 같은 전이를 조용히 사용할 수 있게 합니다.

### 답변 핵심 키워드

shared semantic model, single source of operation meaning, correlated bug, independent oracle, replay, state transition vs emission, final invariant

### 백지 구현

#### 구현 목표

명령 배열을 모델에 순서대로 재생하고 최종 상태를 검증하는 작은 oracle을 작성한다. primitive operation 구현은 제공된다고 가정한다.

#### 인터페이스

```c
typedef struct s_model t_model;

int	apply_model_command(t_model *model, const char *command);
int	replay_and_verify(t_model *model,
		const char *const *commands, size_t command_count);
```

#### 입력과 출력

- `apply_model_command`: 알려진 명령을 적용하면 1, 미지원 명령이면 0
- `replay_and_verify`: 모든 명령이 유효하고 최종 A가 오름차순이며 B가 비었으면 1, 아니면 0

#### 반드시 만족해야 할 조건

- 명령은 정확한 전체 문자열로 비교한다.
- 첫 invalid 명령에서 더 이상 상태를 변경하지 않는다.
- 각 명령은 정확히 한 번 적용한다.
- 최종 검증은 A의 정렬뿐 아니라 B가 비었는지도 확인한다.
- oracle은 generator의 정렬 함수나 checker의 최상위 루프를 호출하지 않는다.
- 가능하면 operation 구현도 제품 코드와 다른 단순 모델로 교차 검증한다.

#### 경계 조건

- 빈 명령 배열
- 이미 정렬된 입력
- 한 개 명령
- no-op이 되는 명령
- combined 명령 `ss`, `rr`, `rrr`
- 중간 invalid 명령
- A는 정렬됐지만 B에 값이 남은 상태
- 중복 값은 입력 파서에서 이미 거절됐다고 가정

#### 실패 조건과 제약

- NULL command, 미지원 command
- 모델 초기화 실패는 호출자가 처리했다고 가정한다.
- 명령 출력 자체의 형식 검증은 S-05에서 다룬다.

```c
int	apply_model_command(t_model *model, const char *command)
{
	// 직접 구현
}

int	replay_and_verify(t_model *model,
		const char *const *commands, size_t command_count)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 빈 스트림이 이미 정렬된 모델에서는 성공한다.
- [ ] 각 primitive와 combined 명령이 의도한 상태를 만든다.
- [ ] prefix가 같은 잘못된 명령을 허용하지 않는다.
- [ ] invalid 명령 뒤의 명령은 적용되지 않는다.
- [ ] A만 정렬되고 B가 비지 않은 상태를 실패로 판정한다.
- [ ] generator와 checker가 같은 버그를 가졌다는 가정에서도 독립 모델이 차이를 찾을 수 있다.
- [ ] operation마다 원소 multiset과 전체 원소 수가 보존된다.
- [ ] 무작위 입력뿐 아니라 작은 크기의 전체 순열을 재생해 본다.

### 구현 후 설명할 것

1. 제품 코드에서 공유할 경계와 테스트에서 독립시킬 경계.
2. operation의 상태 전이와 명령 emission을 분리한 방법.
3. 최종 상태 검증만으로 부족한 invariant.
4. 독립 구현이 오히려 유지비를 늘리는 trade-off.
5. exhaustive small cases와 seeded large cases를 함께 쓰는 이유.

### 원본 확인 위치

- Thread 23 — 공유 연산 모델 위의 생성기–Checker 계약
- 커밋: `feat(model): 배열 기반 스택 상태를 구현`, `test(checker): 명령 연산과 최종 판정을 검증`, `test(sort): 생성 명령의 정렬 결과를 독립 검증`
- 파일: `include/push_swap.h`, `src/stack.c`, `src/operations.c`, `src/push_swap.c`, `src/checker.c`, `tests/run_tests.py`
- 함수·구조: `t_stack`, operation 함수들, checker 명령 적용, Python 독립 replay 모델
- 관련 Thread: 24, 26, 27

---

<a id="s-02"></a>
## S-02. [Thread 24 / `test(operation): 정확한 상태 전이와 no-op을 검증`] 배열 스택 연산 invariant

### 면접 질문

스택을 `values`와 `ranks` 두 병렬 배열, `size`, `capacity`로 표현할 때 swap·push·rotate·reverse rotate가 반드시 보존해야 하는 invariant는 무엇입니까? 요소가 부족한 경우를 오류가 아니라 no-op으로 정의하면 wrapper와 combined operation 구현이 어떻게 단순해집니까?

꼬리 질문:

- `values[i]`와 `ranks[i]`를 항상 함께 이동해야 하는 이유는 무엇입니까?
- push가 source top을 제거하고 destination top에 넣을 때 배열 이동 비용은 얼마입니까?
- combined operation에서 한쪽이 no-op이어도 다른 쪽은 실행해야 합니까?
- operation 성공 여부와 명령 출력 성공 여부를 같은 반환값으로 묶으면 어떤 전파 문제가 생길 수 있습니까?
- 배열 표현과 linked-list 표현의 시간·공간·캐시 trade-off는 무엇입니까?

### 30초 모범 답변

항상 `0 <= size <= capacity`, 같은 인덱스의 value-rank 쌍 일치, 두 스택 전체의 원소 multiset 보존이 필요합니다. swap은 크기 2 미만, rotate 계열은 크기 2 미만, push는 source가 비었으면 no-op으로 두면 combined 명령도 각 primitive를 그대로 호출할 수 있습니다. 배열 top이 index 0이면 push와 rotate에서 O(n) 이동이 들지만 메모리가 연속이고 구조가 단순합니다.

### 답변 핵심 키워드

parallel arrays, pair invariant, size bounds, element conservation, no-op semantics, combined operations, array movement cost

### 백지 구현

#### 구현 목표

병렬 배열 스택의 네 primitive 상태 전이를 작성한다. 명령 문자열 출력은 문제 범위 밖이다.

#### 인터페이스

```c
typedef struct s_stack
{
	int	*values;
	int	*ranks;
	int	size;
	int	capacity;
}   t_stack;

void	stack_swap(t_stack *stack);
void	stack_push(t_stack *destination, t_stack *source);
void	stack_rotate(t_stack *stack);
void	stack_reverse_rotate(t_stack *stack);
```

#### 입력과 출력

- 함수는 스택 상태만 변경한다.
- top은 index 0이다.
- 필요한 요소가 없으면 상태를 바꾸지 않는다.

#### 반드시 만족해야 할 조건

- 모든 이동에서 value와 rank를 하나의 쌍으로 취급한다.
- `size`는 0과 capacity 사이를 벗어나지 않는다.
- push 전후 두 스택 size 합은 같다.
- 어떤 operation도 값을 새로 만들거나 잃지 않는다.
- destination에 빈 capacity가 있다는 전제를 검사하거나 계약으로 명시한다.
- no-op에서도 유효 메모리 범위 밖을 읽지 않는다.

#### 경계 조건

- size 0, 1, 2
- destination empty/source one element push
- source empty push
- destination가 capacity까지 찬 상태
- rotate 두 번으로 원상복구되는 size 2
- rotate 후 reverse rotate
- 두 스택 사이 여러 번 왕복 push

#### 실패 조건과 제약

- invalid stack pointer, 깨진 size/capacity 상태는 계약 위반으로 본다.
- 동적 할당을 하지 않는다.
- 라이브러리 `memmove` 사용 여부는 선택하되 이동하는 pair 수를 설명한다.

```c
void	stack_swap(t_stack *stack)
{
	// 직접 구현
}

void	stack_push(t_stack *destination, t_stack *source)
{
	// 직접 구현
}

void	stack_rotate(t_stack *stack)
{
	// 직접 구현
}

void	stack_reverse_rotate(t_stack *stack)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 각 operation의 size 0·1 no-op이 안전하다.
- [ ] swap이 top 두 pair만 바꾼다.
- [ ] push가 source top pair를 destination top으로 정확히 옮긴다.
- [ ] rotate와 reverse rotate가 서로 역연산이다.
- [ ] operation 전후 value-rank 대응이 유지된다.
- [ ] 두 스택 전체 원소 multiset과 원소 수가 보존된다.
- [ ] 배열 경계 바깥을 읽거나 쓰지 않는다.
- [ ] combined 명령은 각 primitive의 no-op 의미를 보존한다.
- [ ] 각 operation의 pair 이동 횟수를 셀 수 있다.
- [ ] 시간 복잡도를 배열 top 위치와 연결해 설명할 수 있다.

### 구현 후 설명할 것

1. top을 index 0에 둔 선택이 각 operation 비용에 미치는 영향.
2. values와 ranks를 구조체 배열 대신 병렬 배열로 둔 trade-off.
3. no-op 의미가 checker와 combined operation을 단순화하는 방식.
4. 상태 전이와 명령 출력 실패를 분리할지 여부.
5. invariant 테스트를 최종 정렬 테스트와 별도로 둔 이유.

### 원본 확인 위치

- Thread 24 — 배열 스택과 연산 불변식
- 커밋: `feat(operation): 스택 교환 연산을 구현`, `feat(operation): 스택 간 이동 연산을 구현`, `feat(operation): 스택 회전을 구현`, `feat(operation): 스택 역방향 회전을 구현`, `test(operation): 정확한 상태 전이와 no-op을 검증`
- 파일: `include/push_swap.h`, `src/stack.c`, `src/operations.c`, `tests/operation_invariants.c`
- 함수·구조: `t_stack`, `stack_swap`, `stack_push`, `stack_rotate`, `stack_reverse_rotate`, `op_sa`부터 `op_rrr`까지의 wrapper
- 관련 Thread: 23, 25, 26

---

<a id="s-03"></a>
## S-03. [Thread 25 / `feat(parse): 중복 입력을 거절하고 상대 순위를 계산`] 경계 안전 정수 파싱과 rank compression

### 면접 질문

여러 argv 안에 공백으로 결합된 정수 토큰이 들어올 수 있고, 중복을 거절한 뒤 값의 상대 순위 0..n-1을 만들어야 합니다. 정수 파싱, 토큰 수 계산, 배열 크기 산술, 중복 검증, rank 계산을 어떤 단계로 나누겠습니까?

꼬리 질문:

- `INT_MIN`의 절댓값이 `INT_MAX`보다 1 큰 점을 파서에서 어떻게 다룹니까?
- 값을 누적한 뒤 범위를 검사하는 것보다 digit 추가 전에 검사하는 편이 안전한 이유는 무엇입니까?
- `-0`과 `+0`은 서로 다른 문자열이지만 왜 중복 값입니까?
- C whitespace는 어떤 문자 집합으로 한정하며 locale 의존 분류 함수를 쓸지 직접 판정할지 어떻게 결정합니까?
- 정렬된 복사본에서 중복을 찾고 각 원본 값을 binary search하는 방식의 복잡도는 얼마입니까?
- token count가 `INT_MAX`를 넘거나 `count * sizeof(int)`가 overflow하면 언제 실패해야 합니까?

### 30초 모범 답변

먼저 모든 argv를 C whitespace 기준으로 스캔해 token 수를 overflow 없이 계산하고, 각 token을 부호와 ASCII digit만 허용해 범위 안의 int로 변환합니다. 값 배열을 채운 뒤 복사본을 정렬해 인접 중복을 거절합니다. 각 원본 값은 정렬 복사본에서 위치를 찾아 0..n-1 rank로 저장합니다. 이 정규화 덕분에 정렬 단계는 실제 값 범위와 음수 여부를 신경 쓰지 않습니다.

### 답변 핵심 키워드

ASCII tokenization, signed limit, pre-overflow check, token-count overflow, duplicate values, sorted copy, binary search, coordinate compression

### 백지 구현

#### 구현 목표

하나의 token을 `int`로 파싱하고, 중복 없는 값 배열을 연속 rank로 압축하는 두 함수를 작성한다.

#### 인터페이스

```c
int	parse_int_token(const char *start, size_t length, int *out);
int	compress_ranks(const int *values, size_t count, int *ranks);
```

#### 입력과 출력

- `parse_int_token`: 정확히 `length`바이트가 하나의 정수 문법이면 1, 아니면 0
- `compress_ranks`: 중복이 없고 성공하면 1, 중복·할당 실패·크기 오류면 0
- 최소값의 rank는 0, 최대값은 `count - 1`

#### 반드시 만족해야 할 조건

- 선택적 `+` 또는 `-` 뒤에 ASCII digit이 하나 이상 있어야 한다.
- token 전체를 소비해야 하며 공백·NUL·다른 문자는 token 안에서 허용하지 않는다.
- `INT_MIN`과 `INT_MAX`를 허용하고 그 밖의 값은 거절한다.
- signed overflow가 발생하기 전에 범위를 검사한다.
- rank는 0..count-1의 순열이다.
- 같은 정수 값이 둘 이상이면 실패한다.
- 실패 시 `ranks`를 유효 결과처럼 사용하지 못하도록 계약을 정한다.
- 임시 정렬 배열 크기의 곱셈 overflow를 검사한다.

#### 경계 조건

- `0`, `-0`, `+0`
- `INT_MIN`, `INT_MAX`
- 경계보다 1 큰 양수·작은 음수
- 부호만 있는 token
- leading zero
- 비ASCII 숫자 바이트
- count 0, 1
- 이미 정렬된 값, 역순 값
- duplicate zero와 일반 duplicate
- 큰 count의 할당 크기

#### 실패 조건과 제약

- NULL 인자, 빈 token, 잘못된 문자, 범위 초과
- duplicate, 임시 배열 할당 실패, 크기 overflow
- 원본 `values`는 수정하지 않는다.
- 정렬 알고리즘은 표준 `qsort` 또는 직접 구현 중 선택할 수 있다.

```c
int	parse_int_token(const char *start, size_t length, int *out)
{
	// 직접 구현
}

int	compress_ranks(const int *values, size_t count, int *ranks)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 정확한 int 양끝값을 성공시키고 바로 바깥값을 실패시킨다.
- [ ] 부호만 있는 입력과 반복·혼합 부호를 거절한다.
- [ ] ASCII digit 외 문자를 거절한다.
- [ ] `-0`과 `+0`이 동일 값으로 중복 판정된다.
- [ ] 원본 값 순서에 맞는 rank가 생성된다.
- [ ] rank가 빠짐없는 0..n-1 순열이다.
- [ ] 중복 발견 뒤 임시 배열이 해제된다.
- [ ] 임시 배열 크기 overflow를 allocation 전에 거절한다.
- [ ] count 0·1 정책이 호출자 계약과 일치한다.
- [ ] 전체 복잡도는 정렬 기준 O(n log n), 추가 공간 O(n)이다.
- [ ] argv 전체 토큰화 단계에서도 token count 합산 overflow를 별도로 점검할 수 있다.

### 구현 후 설명할 것

1. 양수·음수의 허용 magnitude가 다른 문제를 해결한 방식.
2. token count, 배열 크기, 숫자 누적의 서로 다른 overflow 경계.
3. hash set 대신 정렬 복사본으로 duplicate와 rank를 함께 해결한 trade-off.
4. rank compression이 후속 radix 정렬을 단순화하는 이유.
5. locale 독립 ASCII 문법을 선택한 이유.

### 원본 확인 위치

- Thread 25 — 입력 정규화와 rank compression
- 커밋: `feat(parse): 공백으로 결합된 인자 토큰을 처리`, `feat(parse): 중복 입력을 거절하고 상대 순위를 계산`, `test(parser): 정상 입력과 오류 입력을 검증`, `fix(parse): 토큰 수와 배열 크기 계산을 방어`
- 파일: `src/parser.c`, `tests/run_tests.py`
- 함수: `count_tokens_in_arg`, `count_all_tokens`, `parse_token`, `fill_values`, `compare_ints`, `find_rank`, `assign_ranks`, `parse_input`
- 관련 Thread: 3, 24, 26

---

<a id="s-04"></a>
## S-04. [Thread 26·29 / `feat(sort): 큰 입력을 기수 정렬로 처리` · `test(sort): 큰 입력의 명령 수 상한을 검증`] 크기 적응 정렬·LSD radix·결정적 비용 기준

### 면접 질문

입력 rank가 0..n-1로 압축되어 있을 때 두 스택 연산만으로 LSD radix sort를 구성할 수 있는 이유는 무엇입니까? 작은 입력에는 전용 정렬을 쓰고 큰 입력에는 radix를 쓰는 기준을 어떤 비용으로 설명하겠습니까?

꼬리 질문:

- 각 bit pass가 끝났을 때 유지되어야 하는 정렬 invariant는 무엇입니까?
- pass에서 처음 A에 있던 원소 수만큼만 처리해야 하는 이유는 무엇입니까?
- 필요한 bit 수를 최대 rank에서 계산할 수 있는 이유는 무엇입니까?
- 이미 정렬된 입력에서 아무 명령도 내지 않는 early exit는 어떻게 검증합니까?
- 명령 수만 세면 배열 기반 구현의 실제 이동 비용을 놓칠 수 있는 이유는 무엇입니까?
- wall-clock 대신 명령 수, array movements, peak bytes를 baseline으로 둔 장점은 무엇입니까?
- 100개·500개 상한 테스트와 다중 고정 seed 동치 검사는 서로 무엇을 보완합니까?

### 30초 모범 답변

rank가 0..n-1이면 음수와 큰 값 범위를 제거하고 필요한 bit 수가 작아집니다. LSD 방식은 낮은 bit부터 stable하게 두 그룹으로 분리해 이전 bit 순서를 보존하며, 모든 bit를 처리하면 rank 순서가 완성됩니다. 2~5개는 전용 분기로 명령 수를 줄이고 큰 입력은 O(n log n)의 예측 가능한 radix를 사용합니다. 성능 회귀는 wall-clock보다 명령 수와 배열 pair 이동량, peak allocation을 고정 입력과 seed로 측정하는 편이 재현성이 높습니다.

### 답변 핵심 키워드

rank domain 0..n-1, LSD stability, bit passes, two-stack partition, adaptive strategy, O(n log n), deterministic seeds, command budget, physical movements, peak bytes

### 백지 구현

#### 구현 목표

연속 rank를 가진 A를 제공된 operation만 사용해 오름차순으로 만들고 B를 비우는 큰 입력용 정렬 함수를 작성한다.

#### 제공 인터페이스

```c
int	op_pa(t_stack *a, t_stack *b, int emit);
int	op_pb(t_stack *a, t_stack *b, int emit);
int	op_ra(t_stack *a, int emit);
int	stack_is_sorted(const t_stack *stack);
```

#### 구현할 인터페이스

```c
int	radix_sort(t_stack *a, t_stack *b);
```

#### 입력과 출력

- 입력 A의 rank는 중복 없는 0..n-1 순열이다.
- 성공 시 A는 rank 오름차순, B는 empty, 반환 1
- operation 출력 실패 등으로 중단하면 0

#### 반드시 만족해야 할 조건

- 이미 정렬되었거나 크기 0·1이면 명령을 내지 않는다.
- 필요한 bit 수보다 불필요한 pass를 수행하지 않는다.
- 각 pass가 시작할 때 처리 대상 원소 수를 고정해 새로 회전·이동된 원소를 중복 처리하지 않는다.
- 각 operation 실패를 즉시 전파한다.
- 모든 성공 경로에서 B가 비어 있다.
- 정렬 결과는 실제 values가 아니라 ranks 기준이지만 value-rank pair는 함께 이동한다.
- 최악 명령 수는 입력 크기와 bit 수의 곱에 비례해야 한다.

#### 경계 조건

- n = 0, 1, 2
- 이미 정렬된 입력
- 역순 입력
- rank 최대값이 2의 거듭제곱 전후인 크기
- 한 bit가 모든 원소에서 0 또는 1인 pass
- operation 출력이 pass 중간에 실패
- B에 값이 남는 잘못된 종료

#### 실패 조건과 제약

- 허용 operation 외의 직접 배열 정렬을 사용하지 않는다.
- 추가 rank 배열을 만들지 않는다.
- 작은 입력 전용 정렬은 별도 함수가 담당한다고 가정한다.
- 실패 뒤 이미 출력한 명령은 rollback할 수 없다.

```c
int	radix_sort(t_stack *a, t_stack *b)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] n 0·1과 이미 정렬된 입력의 명령 수가 0이다.
- [ ] 작은 고정 입력과 역순 입력이 정렬된다.
- [ ] 성공 후 A의 ranks가 0..n-1 순서이고 B가 비었다.
- [ ] value-rank pair가 분리되지 않는다.
- [ ] 각 pass가 시작 시점의 n개 원소만 분류한다.
- [ ] bit 수 경계에서 마지막 필요한 pass를 빠뜨리거나 하나 더 수행하지 않는다.
- [ ] 모든 operation 반환값을 확인한다.
- [ ] 크기 2~5 전체 순열은 전용 정렬과 독립 replay로 검증한다.
- [ ] 100·500 입력의 명령 수 상한을 고정 seed로 검증한다.
- [ ] 동일 seed에서 명령 수가 결정적이다.
- [ ] 명령 수와 별도로 배열 pair 이동량·peak bytes 회귀를 확인한다.
- [ ] 시간 복잡도 O(n log n), 추가 알고리즘 공간 O(1)을 설명할 수 있다.

### 구현 후 설명할 것

1. rank compression이 radix의 bit 수와 구현을 단순화한 방식.
2. LSD pass의 안정성이 필요한 이유.
3. 작은 입력 전용 전략과 큰 입력 일반 전략의 전환 기준.
4. 명령 수와 배열 물리 이동량이 다른 비용 지표인 이유.
5. deterministic seed·exhaustive small permutations·resource baseline의 역할 분담.

### 원본 확인 위치

- Thread 26 — 입력 크기 적응 정렬과 LSD radix
  - 커밋: `feat(sort): 네다섯 개의 스택을 정렬`, `feat(sort): 큰 입력을 기수 정렬로 처리`, `test(sort): 생성 명령의 정렬 결과를 독립 검증`
  - 파일: `src/sort.c`, `tests/run_tests.py`
  - 함수: `sort_two`, `sort_three`, `find_rank_index`, `move_index_to_top`, `sort_tiny`, `count_bits`, `radix_sort`, `sort_stack`
- Thread 29 — 명령 수와 물리 이동 비용
  - 커밋: `test(sort): 큰 입력의 명령 수 상한을 검증`, `test(sort): 결정적 다중 시드 동치 검사를 추가`, `test(resource): 명령과 배열 이동 및 할당량을 기준화`
  - 파일: `tests/run_tests.py`, `tests/resource_tests.py`, `tests/resource_baseline.json`, `src/runtime.c`
  - 함수·지표: `deterministic_values`, `test_move_counts`, `PS_OPERATIONS`, `PS_ARRAY_MOVEMENTS`, `PS_PEAK_BYTES`
- 관련 Thread: 23, 24, 25, 28

---

<a id="s-05"></a>
## S-05. [Thread 27 / `fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도`] bounded 명령 프레이밍·재생·판정

### 면접 질문

checker가 stdin에서 `sa\n`, `rrr\n` 같은 짧은 고정 vocabulary만 읽는데 무제한 동적 line reader를 쓰는 것은 왜 불필요합니까? 최대 명령 길이를 3으로 제한할 때 개행, EOF 종료 명령, NUL, 빈 줄, 과장 길이, `EINTR`를 어떻게 구분해야 합니까?

꼬리 질문:

- `sa` 뒤 EOF는 유효한 마지막 명령인데 빈 입력 뒤 EOF는 왜 EOF입니까?
- `sa\0pb\n`을 C 문자열 `"sa"`로 오인하면 어떤 명령 smuggling이 가능합니까?
- 길이 4를 발견한 즉시 나머지 줄을 버리고 계속할지 전체 입력을 오류로 끝낼지 어떻게 정합니까?
- 명령 파싱 오류, 입력 I/O 오류, 최종 `KO`는 서로 다른 결과인데 stdout·stderr·exit status를 어떻게 나눕니까?
- checker가 입력 값이 하나도 없을 때 stdin을 읽지 않고 끝내는 계약은 왜 유용합니까?

### 30초 모범 답변

명령 집합의 최대 길이가 3이면 `PS_COMMAND_MAX + 1` 고정 버퍼로 충분해 메모리 폭증을 막을 수 있습니다. read가 `EINTR`이면 재시도하고, NUL이나 최대 길이 초과는 즉시 문법 오류로 처리합니다. 개행 전의 비어 있는 프레임은 invalid이고, EOF 전에 읽은 유효한 명령은 마지막 프레임으로 적용합니다. I/O·문법 오류는 stderr와 실패 status, 정상 재생 결과는 stdout의 OK 또는 KO로 분리합니다.

### 답변 핵심 키워드

bounded frame, fixed vocabulary, `PS_COMMAND_MAX`, `EINTR`, embedded NUL, overlong command, EOF-terminated frame, error channel vs verdict

### 백지 구현

#### 구현 목표

최대 3문자 명령을 한 프레임씩 읽고 정확한 command enum으로 변환하는 함수를 작성한다.

#### 인터페이스

```c
typedef enum e_read_command
{
	COMMAND_READ_ERROR = -1,
	COMMAND_READ_EOF = 0,
	COMMAND_READ_OK = 1
}   t_read_command;

typedef enum e_command
{
	CMD_SA, CMD_SB, CMD_SS,
	CMD_PA, CMD_PB,
	CMD_RA, CMD_RB, CMD_RR,
	CMD_RRA, CMD_RRB, CMD_RRR
}   t_command;

t_read_command	read_command(int fd, t_command *command);
```

#### 입력과 출력

- 한 줄 또는 EOF로 끝난 마지막 프레임을 읽는다.
- 유효 명령이면 enum을 채우고 `COMMAND_READ_OK`
- 입력이 완전히 끝났으면 `COMMAND_READ_EOF`
- I/O·문법·길이 오류는 `COMMAND_READ_ERROR`

#### 반드시 만족해야 할 조건

- 각 read는 `EINTR`에서 재시도한다.
- NUL byte는 명령 내부에서 허용하지 않는다.
- 3자를 넘는 순간 오류다.
- 빈 줄은 오류다.
- EOF 전까지 1~3자를 읽었으면 하나의 마지막 프레임으로 판정한다.
- 명령 문자열은 prefix가 아니라 정확히 전체가 일치해야 한다.
- EOF·ERROR에서 output enum을 이전 값처럼 사용하지 못하게 한다.
- 한 프레임에 동적 성장 버퍼를 사용하지 않는다.

#### 경계 조건

- 정상 2문자 명령
- 정상 3문자 명령
- 마지막 개행 있음·없음
- 입력 시작 즉시 EOF
- 빈 줄
- 1문자·4문자 명령
- `sa\0pb\n`
- 여러 정상 명령
- 첫·중간·마지막 byte read의 `EINTR`
- 영구 read 오류

#### 실패 조건과 제약

- overlong 프레임을 만난 뒤 checker 전체를 실패시킨다.
- 입력을 한 바이트씩 읽는 단순 구현을 허용하되 시스템 호출 비용을 설명한다.
- read buffering 최적화는 문제 범위 밖이다.

```c
t_read_command	read_command(int fd, t_command *command)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 모든 11개 명령을 개행 있음·없음으로 각각 읽는다.
- [ ] prefix·suffix가 추가된 문자열을 거절한다.
- [ ] 빈 줄과 1문자 명령을 거절한다.
- [ ] 4문자와 매우 긴 입력을 고정 메모리로 거절한다.
- [ ] embedded NUL을 문자열 끝으로 오인하지 않는다.
- [ ] 입력 시작 EOF와 마지막 명령 뒤 EOF를 구분한다.
- [ ] 각 read 위치에 `EINTR`를 주입해도 결과가 같다.
- [ ] 영구 read 오류에서 할당 누수와 잘못된 verdict 출력이 없다.
- [ ] invalid command는 stdout에 OK·KO를 쓰지 않는다.
- [ ] 정상 명령 스트림 뒤 A 정렬·B empty에 따라 OK·KO가 결정된다.

### 구현 후 설명할 것

1. 최대 문법 길이를 메모리 상한으로 직접 사용한 이유.
2. EOF가 frame delimiter이기도 하고 stream 종료이기도 한 이중 의미 처리.
3. NUL과 overlong 입력을 fail-fast한 이유.
4. 문법 오류와 정상 `KO`를 분리한 API·CLI 계약.
5. 한 바이트 read의 단순성과 buffering 최적화 trade-off.

### 원본 확인 위치

- Thread 27 — Checker 명령 프레이밍·재생·판정
- 커밋: `feat(checker): 표준 입력 명령 프레임을 읽음`, `feat(checker): 스택 연산 명령을 해석`, `feat(checker): 명령 실행 결과를 판정`, `fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도`, `test(checker): 읽기 실패와 명령 경계를 검증`
- 파일: `src/checker_reader.c`, `src/checker.c`, `include/push_swap.h`, `tests/fault_tests.py`, `tests/run_tests.py`
- 함수·상수: `read_next_line`, `apply_checker_command`, `read_and_apply`, `PS_COMMAND_MAX`
- 관련 Thread: 23, 24, 28

---

<a id="s-06"></a>
## S-06. [Thread 28 / `refactor(runtime): 메모리와 입력 시스템 호출을 공통화`] 결정적 fault injection과 실패 전파

### 면접 질문

정상 테스트만으로는 "세 번째 malloc 실패", "partial write 다음 영구 오류", "첫 read의 EINTR", "닫힌 stdout의 SIGPIPE" 경로를 안정적으로 재현하기 어렵습니다. 시스템 호출 wrapper와 fault build를 어떻게 구성하면 제품 코드의 정상 경로는 유지하면서 실패 위치를 결정적으로 제어할 수 있습니까?

꼬리 질문:

- `malloc`을 전역 macro로 치환하는 방식과 프로젝트 wrapper를 두는 방식의 장단점은 무엇입니까?
- N번째 할당 실패 sweep에서 마지막 실제 할당보다 한 번 뒤의 성공 case도 확인해야 하는 이유는 무엇입니까?
- live allocation count만 0이면 invalid free·double free까지 없다고 말할 수 있습니까?
- write fault에서 partial prefix가 재출력되지 않았는지 어떻게 검증합니까?
- 오류 메시지 자체의 write도 실패할 수 있을 때 원래 실패 status를 어떻게 보존합니까?
- 닫힌 pipe에서 SIGPIPE로 죽지 않고 `EPIPE`를 처리하려면 어느 계층에 정책을 둡니까?
- sanitizer와 fault injection은 각각 어떤 종류의 결함을 찾습니까?

### 30초 모범 답변

메모리·read·write를 프로젝트 runtime wrapper로 모으고, fault 전용 빌드에서 호출 횟수와 지정된 실패 지점을 주입합니다. 정상 빌드는 그대로 시스템 호출을 통과시킵니다. 테스트는 1번째부터 마지막 할당까지 모두 실패시켜 종료 status, stdout·stderr, live allocation을 확인하고, 마지막 다음 index에서는 정상 성공을 확인합니다. write는 EINTR·short·0·영구 오류와 닫힌 pipe를 각각 주입하고, sanitizer는 주입과 별개로 실제 메모리·UB 문제를 탐지합니다.

### 답변 핵심 키워드

runtime seam, fault build, Nth-call injection, deterministic failure, live allocations, invalid/double free, partial prefix, SIGPIPE policy, sanitizer complement

### 백지 구현

#### 구현 목표

테스트가 명시한 N번째 호출에서 실패할 수 있는 최소 runtime wrapper를 작성한다. 이 문제는 명시적으로 전달받은 fault state를 사용해 환경 변수 parsing은 제외한다.

#### 인터페이스

```c
typedef struct s_fault_state
{
	size_t	malloc_calls;
	size_t	fail_malloc_at;
	size_t	live_allocations;
	size_t	read_calls;
	size_t	eintr_read_at;
	size_t	fail_read_at;
}   t_fault_state;

void	*fault_malloc(t_fault_state *state, size_t size);
void	fault_free(t_fault_state *state, void *pointer);
ssize_t	fault_read(t_fault_state *state, int fd, void *buffer, size_t count);
```

#### 입력과 출력

- `fail_*_at == 0`이면 해당 fault를 비활성화한다.
- 지정한 호출 번호에서만 fault를 발생시킨다.
- allocation 성공·해제에 따라 live count를 갱신한다.
- read fault는 `errno`를 실제 시나리오와 맞게 설정한다.

#### 반드시 만족해야 할 조건

- 호출 count는 실제 시스템 호출 시도 전에 일관되게 증가한다.
- 지정한 malloc 호출은 시스템 malloc을 호출하지 않고 NULL을 반환한다.
- 성공한 allocation만 live count를 증가시킨다.
- NULL free는 live count를 바꾸지 않는다.
- 지정한 read EINTR는 `-1`과 `errno == EINTR`를 반환한다.
- 지정한 영구 read fault는 `-1`과 선택한 영구 오류를 반환한다.
- fault가 아닌 호출은 실제 시스템 호출의 결과를 그대로 전달한다.
- 테스트 종료 시 live count를 관찰할 수 있다.

#### 경계 조건

- fault index 0
- 첫 호출 실패
- 마지막 실제 할당 실패
- 마지막보다 한 번 뒤 index
- malloc이 자체적으로 실패하는 경우
- NULL free
- 첫·중간 read의 EINTR
- EINTR 다음 정상 read
- 영구 read 오류
- 여러 test case 사이 state reset

#### 실패 조건과 제약

- 이 최소 문제에서는 외부 pointer의 free와 double free 탐지는 구현 범위 밖이다. 확장 시 allocation metadata와 정렬 보존을 설명한다.
- wrapper 자체의 보고 함수가 같은 fault 경로를 재사용해 재귀하지 않도록 해야 한다.
- 멀티스레드 안전성은 문제 범위 밖이다.

```c
void	*fault_malloc(t_fault_state *state, size_t size)
{
	// 직접 구현
}

void	fault_free(t_fault_state *state, void *pointer)
{
	// 직접 구현
}

ssize_t	fault_read(t_fault_state *state, int fd, void *buffer, size_t count)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] fault 비활성 상태가 시스템 호출과 동일하게 동작한다.
- [ ] 지정한 정확한 malloc index에서만 실패한다.
- [ ] 각 부분 생성 단계 실패 뒤 live allocation이 0으로 돌아온다.
- [ ] 마지막보다 한 번 뒤 index에서는 정상 실행이 성공한다.
- [ ] read EINTR가 호출자 재시도로 복구된다.
- [ ] 영구 read 오류가 최상위 실패 status까지 전파된다.
- [ ] test case마다 호출 count와 live count가 reset된다.
- [ ] write 확장 테스트에서 short prefix가 중복되지 않는다.
- [ ] 0바이트 write가 무한 루프 없이 실패한다.
- [ ] 닫힌 stdout에서 SIGPIPE로 비정상 종료하지 않고 실패·cleanup을 관찰한다.
- [ ] 오류 출력 자체가 실패해도 원래 실패 status가 성공으로 바뀌지 않는다.
- [ ] ASan·UBSan 실행이 fault test와 별도로 통과한다.

### 구현 후 설명할 것

1. wrapper seam을 제품 코드의 어디에 두었는가.
2. 호출 index 의미를 "시도 전" 또는 "시도 후" 중 어떻게 정의했는가.
3. fault harness의 관찰 정보가 실제 프로그램 의미를 바꾸지 않게 한 방법.
4. live count, invalid free, double free, peak bytes를 단계적으로 확장하는 방법.
5. fault injection과 sanitizer가 상호 대체가 아니라 보완 관계인 이유.

### 원본 확인 위치

- Thread 28 — 장애 주입과 I/O 실패 전파
- 커밋: `refactor(runtime): 메모리와 입력 시스템 호출을 공통화`, `test(memory): 할당 실패 뒤 자원 정리를 검증`
- 파일: `src/runtime.c`, `src/stack.c`, `src/checker_reader.c`, `src/checker.c`, `tests/fault_tests.py`, `tests/resource_tests.py`, `Makefile`
- 함수: `ps_malloc`, `ps_free`, `ps_read`, `ps_write_all`, `ps_ignore_sigpipe`, `ps_test_finish`
- 관련 Thread: 4, 6, 27, 29
