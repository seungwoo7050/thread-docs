# 상태 커밋과 종료 선형화

<a id="P08"></a>
## [Thread 05 / `fix(routine): 중단된 식사를 완료 횟수에서 제외`] 완료 시점의 상태 커밋과 목표 집계

### 면접 질문

철학자가 "식사를 시작했다"는 사실과 "식사를 끝냈다"는 사실은 다릅니다. 식사 대기 중 전체 종료가 발생할 수 있을 때 `meals`, `full_count`, `ended`를 언제 어떻게 갱신해야 중단된 식사를 완료로 세지 않고 목표 달성을 정확히 한 번만 집계할 수 있습니까?

꼬리 질문:

- `last_meal_ms`는 왜 식사 완료가 아니라 식사 시작 시점에 갱신합니까?
- `philo_sleep_ms`가 반환값 없이 끝나면 호출자는 완료와 중단을 어떻게 구분합니까?
- `meals >= must_eat`일 때마다 `full_count`를 증가시키면 어떤 중복 집계가 생깁니까?
- 식사 대기가 성공했지만 상태 mutex를 잡기 직전에 `ended`가 설정되면 카운터를 올려야 합니까?
- 마지막 철학자의 완료가 `full_count == number`를 만들 때 종료 상태를 누가 commit하는 것이 좋습니까?
- 입력의 `must_eat`이 `int`여도 런타임 `meals`를 더 넓은 형으로 둔 이유는 무엇입니까?

### 30초 모범 답변

식사 시작 시에는 사망 판정 기준인 `last_meal_ms`만 갱신하고, 실제 식사 시간이 끝난 뒤에만 완료 카운터를 commit합니다. 대기 함수는 deadline 도달과 종료로 인한 중단을 구분해 반환해야 하며, 중단이면 `meals`와 `full_count`를 건드리지 않습니다. 완료 commit은 `state_mutex` 아래에서 종료 상태를 다시 확인하고 `meals`를 증가시킨 뒤, 값이 목표와 정확히 같아지는 전이에서만 `full_count`를 한 번 올립니다. 마지막 완료가 전체 목표를 만족시키면 같은 임계 구역에서 `ended`까지 설정해야 관찰자가 중간 상태를 보지 않습니다.

### 답변 핵심 키워드

start vs completion · commit point · interruptible operation · no partial effect · exact-threshold transition · `full_count` invariant · atomic state transition · `int64_t` accumulator

### 백지 구현

#### 구현 목표

식사 대기의 결과를 받아 완료된 식사만 상태에 반영하는 축소형 commit 함수를 구현한다. 전체 종료와 목표 집계는 같은 공유 상태 임계 구역에서 처리한다.

#### 인터페이스 또는 함수 시그니처

```c
int record_meal_result(t_philo *philo, int wait_status)
{
    // 직접 구현
}
```

`t_philo`에는 `meals`, `table`이 있고, `t_table`에는 `state_mutex`, `ended`, `full_count`, `config.number`, `config.must_eat`, `config.has_meal_limit`가 있다고 가정한다. `wait_status`는 식사 시간이 정상 완료되었으면 `PHILO_OK`, 종료 때문에 중단되었으면 `PHILO_ERR`다.

#### 입력과 출력

- 입력: 식사 중인 철학자와 대기 결과
- 반환: 완료를 commit했으면 `PHILO_OK`, 중단·이미 종료된 상태라 commit하지 않았으면 `PHILO_ERR`
- 상태 출력: 필요할 때만 `meals`, `full_count`, `ended` 갱신

#### 반드시 만족해야 할 조건

- `wait_status != PHILO_OK`이면 모든 완료 카운터가 그대로여야 한다.
- 공유 상태 갱신은 `state_mutex`로 보호한다.
- mutex를 획득한 뒤 `ended`를 다시 확인한다.
- 이미 종료된 실행에서는 완료 카운터를 변경하지 않는다.
- 정상 완료에서만 `meals`를 정확히 1 증가시킨다.
- 식사 제한이 있을 때 `meals`가 `must_eat`과 정확히 같아지는 순간에만 `full_count`를 1 증가시킨다.
- 목표를 초과한 추가 식사는 `full_count`를 다시 증가시키지 않는다.
- `full_count`가 철학자 수에 도달하면 같은 임계 구역에서 `ended`를 참으로 만든다.
- 모든 반환 경로에서 mutex를 해제한다.
- `meals` 누적형은 허용 가능한 장기 실행에서 overflow하지 않을 폭을 사용한다.

#### 경계 조건

- 첫 번째 식사 완료
- 목표 바로 전, 목표와 같은 식사, 목표를 넘긴 식사
- `must_eat == 1`
- 마지막 미완료 철학자가 전체 목표를 채우는 경우
- 식사 대기 중 종료
- 대기 성공 직후 commit 전에 종료
- `meals`가 `INT_MAX` 부근인 경우
- 식사 제한이 없는 실행

#### 실패 조건

- 식사를 시작하자마자 `meals`를 올리는 경우
- 중단된 대기를 성공으로 간주하는 경우
- `meals >= must_eat`마다 `full_count`를 증가시키는 경우
- `full_count` 갱신과 `ended` 갱신을 서로 다른 임계 구역에 두어 중간 상태를 노출하는 경우
- 종료 확인 없이 늦은 완료가 카운터를 변경하는 경우
- 오류 반환에서 mutex나 포크 해제 책임이 모호해지는 경우

#### 필요한 제약

- 이 함수는 포크 unlock을 직접 수행하지 않는다. 호출자가 성공·실패 모두에서 포크를 정리하는 계약으로 둔다.
- 공유 상태 임계 구역 안에서 sleep이나 로그 I/O를 하지 않는다.
- 시간 복잡도와 추가 공간은 O(1)이어야 한다.
- 원본과 다른 함수 분할도 "완료 전에는 효과 없음, 완료 commit은 원자적"이라는 계약을 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 완료된 식사 한 번마다 `meals`가 한 번 증가한다.
- 경계값: 목표 도달 시점에만 `full_count`가 증가하고 이후에는 유지된다.
- 실패 경로: 대기 중단과 이미 종료된 상태에서 모든 카운터가 그대로다.
- 상태 변화: `meals`, `full_count`, `ended`의 관련 전이가 하나의 임계 구역에서 일어난다.
- invariant: `0 <= full_count <= number`이며 각 철학자는 최대 한 번만 full 집계에 기여한다.
- 중복·누락 처리: 목표 초과 시 중복 집계가 없고 마지막 완료 시 종료 누락이 없다.
- 동시성·비동기 문제: 종료와 완료가 경쟁해도 둘 중 먼저 commit된 상태에 따라 결과가 일관된다.
- 시간·공간 복잡도: 상수 시간·상수 공간이다.
- 요구사항 충족 여부: 시작된 식사와 완료된 식사를 분리했다.

### 구현 후 설명할 것

1. 식사 시작 시각과 식사 완료 카운터의 갱신 시점을 다르게 둔 이유
2. `philo_sleep_ms`의 반환 계약이 상태 commit을 가능하게 한 방식
3. `== must_eat` 전이만 집계해 중복을 막는 invariant
4. 마지막 완료와 전체 종료를 같은 임계 구역에서 처리한 이유
5. `must_eat`과 런타임 누적 `meals`의 정수 폭을 다르게 볼 수 있는 이유

### 원본 확인 위치

- Thread 05
- `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단`
- `fix(routine): 중단된 식사를 완료 횟수에서 제외`
- `test(routine): 중단된 식사의 카운터 불변식 검증`
- `fix(state): 식사 완료 횟수의 정수 범위 확장`
- `src/routine.c`: `record_meal_start`, `record_meal_done`, `eat_once`, `philo_routine`
- `src/time.c`: `philo_sleep_ms`
- `include/philo.h`: `t_philo.meals`, `t_table.full_count`, `ended`
- `tests/interrupted_meal.c`, `tests/meal_counter_range.c`
- 관련 Thread: 03의 작업 루틴, 04의 중단 가능한 대기, 06의 완료 종료 상태

---

<a id="P09"></a>
## [Thread 06 / `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정`] 종료 상태와 terminal 로그의 선형화

### 면접 질문

모니터가 잠금 아래에서 한 철학자를 사망 후보로 찾은 직후, 해당 철학자가 식사를 시작하거나 전체 식사 목표가 완료될 수 있습니다. 동시에 여러 스레드가 일반 상태 로그를 출력하려 할 때, 오래된 사망 판정을 버리고 "사망 로그는 정확히 한 번, 마지막 줄로만 출력된다"는 불변식을 어떻게 보장하겠습니까?

꼬리 질문:

- 모니터가 찾은 포인터를 곧바로 신뢰하면 어떤 check-then-act 경쟁이 생깁니까?
- 사망 여부를 출력 직전에 다시 검사해야 하는 이유는 무엇입니까?
- `ended = 1`과 사망 로그 출력 사이에 일반 로그가 끼어들 수 없게 하려면 무엇을 함께 직렬화해야 합니까?
- `state_mutex`와 `print_mutex`를 모두 쓰는 코드에서 전역 락 순서가 필요한 이유는 무엇입니까?
- 완료 종료와 사망 종료가 동시에 경쟁할 때 어느 하나만 terminal 전이를 가져가게 하는 선형화 지점은 어디입니까?
- 모든 상태를 하나의 mutex로 보호하는 단순한 설계와 두 mutex 설계의 trade-off는 무엇입니까?

### 30초 모범 답변

모니터가 잠금 아래에서 찾은 사망 후보는 힌트일 뿐이므로, 실제 terminal 전이는 출력 직전에 다시 검증해야 합니다. 일반 로그와 사망 로그가 같은 출력 직렬화 경계를 사용하고, 사망 경로는 정해진 전역 순서로 출력 락과 상태 락을 잡은 뒤 `ended`와 최신 `last_meal_ms`를 재확인합니다. 조건이 여전히 참인 한 스레드만 `ended`를 설정하고 timestamp를 확정한 뒤 출력 락을 유지한 채 사망 로그를 남깁니다. 그러면 다른 terminal 후보는 실패하고, 일반 로그도 terminal 출력 뒤에 나타날 수 없습니다.

### 답변 핵심 키워드

linearization point · stale snapshot · revalidation · check-then-act · terminal CAS 의미 · lock ordering · output serialization · exactly-once · no log after terminal

### 백지 구현

#### 구현 목표

일반 상태 로그와 사망 로그가 경쟁하는 축소형 로깅 함수를 구현한다. 사망 조건은 commit 직전에 최신 상태로 재검증하며, 사망 로그가 출력되면 그 뒤에 어떤 일반 로그도 출력되지 않아야 한다.

#### 인터페이스 또는 함수 시그니처

```c
void philo_log(t_philo *philo, const char *message)
{
    // 직접 구현
}

int philo_try_log_death(t_philo *philo)
{
    // 직접 구현
}
```

`t_table`에는 `state_mutex`, `print_mutex`, `ended`, `start_ms`, `config.time_to_die`가 있고, `t_philo`에는 `last_meal_ms`, `id`, `table`이 있다고 가정한다. `philo_now_ms`와 실제 한 줄 출력 함수는 제공된다.

#### 입력과 출력

- `philo_log`: 실행 중일 때만 일반 상태 한 줄을 출력
- `philo_try_log_death`: 최신 상태로 사망을 확정하고 출력했으면 1, 이미 종료되었거나 더 이상 사망 조건이 아니면 0
- 상태 출력: 성공한 사망 경로만 `ended`를 0에서 1로 바꿈

#### 반드시 만족해야 할 조건

- 일반 로그 한 줄은 다른 로그와 바이트 단위로 섞이지 않는다.
- 일반 로그는 출력 직전 terminal 상태를 확인하고, 종료 뒤에는 출력하지 않는다.
- 사망 후보를 발견한 과거 스냅샷만으로 terminal 상태를 확정하지 않는다.
- 사망 경로는 최신 `now`와 `last_meal_ms`로 조건을 다시 계산한다.
- 이미 `ended`가 참이면 사망 로그를 출력하지 않는다.
- 조건이 여전히 참인 한 호출만 `ended`를 설정하고 사망 로그를 출력한다.
- timestamp와 terminal 상태는 같은 판정 시점에서 확정되어야 한다.
- 사망 로그를 출력한 뒤 일반 로그가 추가될 수 없는 락 범위를 설계한다.
- 두 mutex를 함께 잡는 모든 경로는 하나의 전역 순서를 따라야 한다.
- 반환 전에 획득한 모든 mutex를 해제한다.

#### 경계 조건

- 사망 임계값 바로 전과 정확히 같은 시각
- 모니터 판정 뒤 철학자가 `last_meal_ms`를 갱신한 경우
- 목표 완료가 사망 commit보다 먼저 `ended`를 설정한 경우
- 두 모니터성 호출이 동시에 같은 철학자의 사망을 확정하려는 경우
- 여러 일반 로거와 사망 로거가 동시에 시작하는 경우
- 사망 로그 출력 직전 일반 로그가 출력 중인 경우
- timestamp가 0인 매우 이른 종료

#### 실패 조건

- `ended`를 먼저 확인하고 락을 놓은 뒤 나중에 출력하는 경우
- 사망 조건을 한 번만 검사해 오래된 후보를 출력하는 경우
- `ended` 설정과 사망 출력 사이에 일반 로그가 끼어드는 경우
- 서로 다른 함수가 `state_mutex -> print_mutex`, `print_mutex -> state_mutex`처럼 반대 순서를 사용하는 경우
- 여러 사망 로그가 출력되거나 terminal 로그 뒤 일반 로그가 나오는 경우
- 출력 락을 잡은 채 필요한 상태를 갱신하지 않아 다른 경로가 terminal 전이를 먼저 가져가는 경우

#### 필요한 제약

- atomic 정수나 단일 통합 mutex로 다시 설계해도 되지만, 선택한 선형화 지점과 출력 순서 보장을 설명해야 한다.
- `printf` 자체의 내부 잠금만으로 프로그램 수준 불변식을 보장한다고 가정하지 않는다.
- 사망 조건 계산은 충분한 정수 폭의 단조 시각을 사용한다.
- 임계 구역은 정확성을 우선하되 불필요한 긴 작업은 피한다.
- 원본과 다른 락 구성도 정확히 한 terminal 전이와 이후 로그 금지를 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 실행 중 일반 로그는 직렬화되어 출력된다.
- 경계값: 사망 임계값 직전에는 실패하고, 임계값 이상에서만 성공한다.
- 실패 경로: 후보 판정 뒤 식사를 시작한 경우 오래된 사망 로그를 버린다.
- 상태 변화: 성공한 단 하나의 호출만 `ended`를 0에서 1로 바꾼다.
- invariant: 사망 로그는 최대 한 줄이고, 존재한다면 전체 trace의 마지막 줄이다.
- 중복·누락 처리: 동시 사망 시도에서 중복 출력이 없고 실제 사망 조건에서는 terminal 전이가 누락되지 않는다.
- 동시성·비동기 문제: 모든 복수 락 경로가 동일 순서를 사용하며 lock-order cycle이 없다.
- 요구사항 충족 여부: 상태 확정과 관찰 가능한 로그가 하나의 선형화된 사건처럼 보인다.

### 구현 후 설명할 것

1. 모니터의 최초 사망 탐색을 "후보 스냅샷"으로만 취급한 이유
2. terminal 전이의 선형화 지점을 어디로 잡았는지
3. 상태 락과 출력 락의 순서가 중복 로그와 교착을 동시에 막는 방식
4. timestamp를 언제 계산하고 왜 그 시점의 값을 출력하는지
5. 하나의 mutex, 두 mutex, atomic 상태를 사용하는 대안의 trade-off

### 원본 확인 위치

- Thread 06
- `feat(log): 상태 로그의 동시 출력 보호`
- `feat(monitor): 사망과 식사 완료 조건 감시`
- `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정`
- `test(monitor): 완료 상태와 오래된 사망 판정 검증`
- `src/state.c`: `philo_has_ended`, `philo_finish`, `philo_log`, `philo_try_log_death`
- `src/monitor.c`: `all_meals_done`, `find_dead_philo`, `philo_monitor`
- `tests/terminal_state.c`, `tests/log_terminal_race.c`, `tests/concurrency.sh`
- 관련 Thread: 05의 목표 완료 commit, 07의 terminal trace 불변식 검증
