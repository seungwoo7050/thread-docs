# 동시성 회귀 테스트와 실패 주입

<a id="P10"></a>
## [Thread 07 / `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증`] 비결정적 실행 trace의 불변식 검증

### 면접 질문

철학자 프로그램은 실행할 때마다 로그 순서가 달라집니다. 특정한 한 스케줄이나 정확한 전체 출력과 비교하지 않고도 교착, 진행 실패, 잘못된 로그 형식, 감소하는 시각, 중복 사망, terminal 로그 뒤의 추가 출력을 어떻게 자동 검증하겠습니까?

꼬리 질문:

- 동시성 테스트에서 정확한 줄 순서를 golden file로 비교하면 왜 취약합니까?
- timeout은 기능 검증입니까, 교착·무한 대기 탐지입니까? 오탐 가능성은 어떻게 줄입니까?
- "모든 철학자가 목표 횟수만큼 먹었다"는 조건과 "프로그램이 종료했다"는 조건을 왜 따로 검사해야 합니까?
- 사망 시나리오에서 정확히 한 번의 `died`와 그 뒤 로그 없음은 safety invariant입니까, liveness invariant입니까?
- 같은 시나리오를 여러 번 반복하는 이유와 반복 테스트만으로 race 부재를 증명할 수 없는 이유는 무엇입니까?
- ThreadSanitizer를 통과하는 것과 도메인 불변식 테스트를 통과하는 것이 각각 무엇을 보장합니까?

### 30초 모범 답변

비결정적인 내부 순서 대신 외부에서 반드시 성립해야 하는 불변식을 검증합니다. 각 줄을 파싱해 형식, ID 범위, timestamp 비감소를 검사하고, 유한 식사 시나리오에서는 철학자별 `is eating` 횟수가 목표 이상인지 확인합니다. 사망 시나리오는 `died`가 정확히 한 번이고 그 줄이 마지막인지 확인합니다. 각 실행에는 timeout을 둬 교착을 탐지하고 여러 작업자 수와 타이밍 조합을 반복합니다. 다만 반복 실행은 race의 증명이 아니므로 TSan 같은 동적 분석과 결정적 실패 주입을 함께 사용해야 합니다.

### 답변 핵심 키워드

observable invariant · schedule-independent oracle · trace parser · safety/liveness · per-worker progress · exactly-one terminal · timeout guard · repetition · TSan complementary

### 백지 구현

#### 구현 목표

이미 파싱된 이벤트 배열과 시나리오 명세를 받아 로그의 공통 불변식, 유한 목표 진행성, 사망 terminal 조건을 검증하는 순수 함수를 구현한다.

#### 인터페이스 또는 함수 시그니처

```c
typedef enum e_event_kind
{
    EVENT_FORK,
    EVENT_EAT,
    EVENT_SLEEP,
    EVENT_THINK,
    EVENT_DIED
}   t_event_kind;

typedef struct s_event
{
    int64_t      timestamp_ms;
    int          philo_id;
    t_event_kind kind;
}   t_event;

typedef enum e_scenario_kind
{
    SCENARIO_FINITE_MEALS,
    SCENARIO_EXPECT_DEATH
}   t_scenario_kind;

typedef struct s_trace_spec
{
    int             philo_count;
    int             meal_target;
    t_scenario_kind scenario;
}   t_trace_spec;

int validate_trace(const t_event *events, size_t count,
        const t_trace_spec *spec)
{
    // 직접 구현
}
```

프로세스 실행과 텍스트 파싱은 구현 범위 밖이며, 빈 로그를 허용하지 않는다고 가정한다.

#### 입력과 출력

- 입력: 시간순으로 관찰된 이벤트 배열, 이벤트 수, 철학자 수와 시나리오 명세
- 출력: 모든 요구 불변식을 만족하면 `PHILO_OK`, 하나라도 위반하면 `PHILO_ERR`

#### 반드시 만족해야 할 조건

- 모든 이벤트의 철학자 ID는 1부터 `philo_count` 사이여야 한다.
- timestamp는 앞선 이벤트보다 작아지면 안 된다. 같은 값은 허용한다.
- 이벤트 종류는 정의된 값 중 하나여야 한다.
- `SCENARIO_FINITE_MEALS`에서는 `EVENT_DIED`가 없어야 한다.
- 유한 식사 시나리오에서는 각 철학자의 `EVENT_EAT` 횟수가 `meal_target` 이상이어야 한다.
- `SCENARIO_EXPECT_DEATH`에서는 `EVENT_DIED`가 정확히 한 번이어야 한다.
- 사망 이벤트가 있다면 그 뒤에는 어떤 이벤트도 없어야 한다.
- 사망 시나리오에서 terminal 조건을 만족하지 않으면 단순히 프로세스가 종료했다는 이유로 성공하면 안 된다.
- 카운터 배열 접근 전에 ID 범위를 검증한다.
- 입력을 변경하지 않는다.

#### 경계 조건

- 철학자 수 1
- 이벤트 수 1인 사망 trace
- 여러 이벤트가 같은 timestamp를 가진 경우
- 첫 줄이 사망인 경우
- 마지막 직전 줄이 사망이고 뒤에 일반 로그가 하나 있는 경우
- 한 철학자만 목표 횟수에 1회 부족한 경우
- 사망 로그가 0개, 1개, 2개인 경우
- 매우 긴 trace

#### 실패 조건

- 정확한 스레드 인터리빙을 요구해 정상 실행을 실패시키는 경우
- 전체 `EVENT_EAT` 합만 확인해 특정 철학자의 진행 누락을 놓치는 경우
- 첫 사망 뒤 나머지 이벤트를 검사하지 않는 경우
- timestamp가 감소하는 로그를 허용하는 경우
- timeout 종료를 정상 종료로 오인하는 경우
- 빈 출력이나 파싱 실패를 성공으로 처리하는 경우

#### 필요한 제약

- 시간 복잡도는 O(event count + philo count) 이하여야 한다.
- 추가 공간은 철학자 수에 비례하는 카운터 정도만 허용한다.
- 이벤트의 구체적인 정상 순서를 강제하지 않는다.
- timeout과 프로세스 종료 상태 검사는 호출자가 별도로 수행한다고 가정한다.
- 원본 shell·AWK 구현과 다른 언어를 사용해도 같은 관찰 불변식을 검증하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 여러 정상 인터리빙이 모두 통과하고 정확한 순서 차이에는 영향받지 않는다.
- 경계값: 같은 timestamp, 단일 사망 줄, 철학자 수 1을 확인했다.
- 실패 경로: ID 범위 오류, 감소 시각, 목표 미달, 중복 사망, 사망 뒤 로그가 모두 실패한다.
- 상태 변화: 입력 trace를 수정하지 않고 별도 집계 상태만 사용한다.
- invariant: 유한 시나리오의 개별 진행성과 사망 시나리오의 terminal 단일성이 분리되어 있다.
- 중복·누락 처리: 전체 합이 충분해도 한 철학자 목표 미달을 잡는다.
- 동시성·비동기 문제: 특정 스케줄이 아니라 모든 정상 스케줄에 공통인 성질만 요구한다.
- 시간·공간 복잡도: 한 번의 trace 순회와 철학자별 카운터로 끝난다.
- 요구사항 충족 여부: timeout, 반복 실행, 동적 분석이 이 순수 validator와 별도 책임임을 구분했다.

### 구현 후 설명할 것

1. 정확한 출력 순서 대신 불변식을 oracle로 선택한 이유
2. safety 속성과 liveness 속성을 각각 어떻게 관찰했는지
3. 전체 식사 합이 아니라 철학자별 진행을 검사한 이유
4. timeout 값이 너무 짧거나 길 때 생기는 trade-off
5. 반복 회귀 테스트와 ThreadSanitizer가 서로 대체 관계가 아닌 이유

### 원본 확인 위치

- Thread 07
- `test(smoke): 주요 입력과 종료 조건 검증`
- `test(format): 필수 상태 로그 형식 검증`
- `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증`
- `test(tsan): ThreadSanitizer 검증 경로 추가`
- `tests/concurrency.sh`: `run_timeout`, `check_log`, `check_progress`, `check_terminal_line`
- `tests/log_terminal_race.c`
- `tests/tsan.sh`
- 관련 Thread: 03의 포크 획득 진행성, 05의 식사 목표, 06의 terminal 로그 선형화

---

<a id="P11"></a>
## [Thread 07 / `test(init): 부분 뮤텍스 초기화 롤백 검증`] 시스템 경계의 결정적 실패 주입

### 면접 질문

`malloc`, mutex 초기화·파괴, 스레드 생성·join, 조건 변수 대기, 시계 조회처럼 평소에는 거의 실패하지 않는 호출의 오류 경로를 어떻게 결정적으로 테스트하겠습니까? 특히 "k번째 호출만 실패"하게 만들고, 호출 횟수 자체보다 최종 자원 소유권과 상태 불변식을 어떻게 검증하겠습니까?

꼬리 질문:

- 실제 시스템이 우연히 실패하기를 기다리는 테스트가 왜 의미가 없습니까?
- C 코드에서 전처리기 심볼 치환과 함수 포인터 의존성 주입은 각각 어떤 장단점이 있습니까?
- 모든 실패 위치를 순회할 때 종료 조건을 어떻게 정합니까?
- fatal 경로가 `_exit`를 호출한다면 같은 프로세스 안에서 테스트하면 왜 안 됩니까?
- join 실패 테스트에서 단순 반환 코드 외에 어떤 공유 자원 상태를 확인해야 합니까?
- destroy 실패 뒤 재시도 가능한 상태를 어떻게 단언합니까?
- TSan 실행 가능 여부를 probe하고 "지원 안 됨"과 "race 발견"을 다른 종료 코드로 구분하는 이유는 무엇입니까?

### 30초 모범 답변

외부 시스템 호출을 얇은 경계로 만들고 테스트에서는 호출 횟수를 세어 지정한 k번째 호출만 실패시키면 드문 경로를 결정적으로 재현할 수 있습니다. 각 k를 순회하면서 단순히 오류 반환만 보지 않고, 성공한 자원만 한 번 정리됐는지, 포인터와 count가 일치하는지, join되지 않은 작업자가 있으면 파괴가 막히는지처럼 최종 소유권 상태를 단언합니다. `_exit` 같은 fatal 경로는 자식 프로세스에서 실행해 부모가 종료 상태와 출력만 확인합니다. 동적 분석 도구는 먼저 지원 여부를 probe하고 unsupported와 실제 검출 실패를 구분해야 CI 정책을 명확히 할 수 있습니다.

### 답변 핵심 키워드

fault injection · kth-call failure · seam/wrapper · symbol substitution · state-based assertion · exhaustive failure index · process isolation · retryable cleanup · capability probe

### 백지 구현

#### 구현 목표

포크 뮤텍스 여러 개를 초기화하는 함수에 대해 각 초기화 위치의 실패를 주입하고, 성공한 자원이 정확히 한 번만 정리되며 반복 destructor가 안전한지 검증하는 테스트 harness를 작성한다.

#### 인터페이스 또는 함수 시그니처

```c
typedef struct s_fault_plan
{
    size_t init_calls;
    size_t destroy_calls;
    size_t fail_init_at;
    int    duplicate_destroy;
}   t_fault_plan;

int fake_mutex_init(pthread_mutex_t *mutex,
        const pthread_mutexattr_t *attr)
{
    // 직접 구현
}

int fake_mutex_destroy(pthread_mutex_t *mutex)
{
    // 직접 구현
}

int run_init_failure_case(size_t fail_at, size_t fork_total)
{
    // 직접 구현
}

int main(void)
{
    // 모든 실패 위치를 순회하도록 직접 구현
}
```

면접용 빌드에서는 대상 소스의 `pthread_mutex_init`과 `pthread_mutex_destroy`가 위 fake 함수로 연결된다고 가정한다. 실제 주소별 destroy 여부를 기록할 고정 크기 저장소는 제공해도 된다.

#### 입력과 출력

- 입력: 실패시킬 초기화 호출 인덱스와 포크 개수
- 출력: 각 실패 위치에서 대상 초기화가 오류를 반환하고 모든 자원 상태 불변식을 만족하면 테스트 성공
- `main` 반환: 모든 case 성공 시 0, 하나라도 위반하면 0이 아닌 값

#### 반드시 만족해야 할 조건

- 지정한 호출 하나만 실패하고 나머지 fake 호출은 실제 계약과 같은 성공·실패 형태를 반환한다.
- 실패 인덱스는 명확한 0 기반 또는 1 기반 규칙 하나로 일관된다.
- 대상 함수가 실패를 호출자에 전파했는지 확인한다.
- 실패 전에 성공한 각 뮤텍스가 정확히 한 번 destroy 대상이 되었는지 확인한다.
- 실패한 뮤텍스와 아직 시도하지 않은 뮤텍스는 destroy되지 않아야 한다.
- 같은 주소에 destroy가 두 번 호출되면 실패한다.
- 초기화 실패 뒤 동적 배열 포인터와 소유 count·ready flag가 정리 계약에 맞는지 확인한다.
- 실패 후 destructor를 한 번 더 호출해 추가 destroy가 없는지 확인한다.
- 테스트 자체의 전역 상태는 case마다 완전히 초기화한다.
- 첫 번째·중간·마지막 실패 위치를 모두 포함한다.

#### 경계 조건

- 첫 초기화 호출 실패
- 첫 포크 뮤텍스 실패
- 중간 포크 뮤텍스 실패
- 마지막 포크 뮤텍스 실패
- 실패하지 않는 정상 case
- destructor 재호출
- 같은 주소를 우연히 중복 기록하는 harness 오류
- 실패 인덱스가 호출 수 밖인 경우

#### 실패 조건

- fake 함수가 대상 코드와 다른 의미의 반환값을 사용하는 경우
- 테스트 case 사이에 호출 횟수나 파괴 기록이 누적되는 경우
- 특정 구현의 우연한 호출 횟수만 검사하고 최종 자원 상태를 보지 않는 경우
- 실패한 자원까지 destroy된 것을 놓치는 경우
- 중복 destroy를 집계 수만으로 확인해 같은 주소의 중복을 놓치는 경우
- fatal 경로를 같은 테스트 프로세스에서 호출해 전체 suite를 종료시키는 경우

#### 필요한 제약

- sleep이나 확률적 race에 의존하지 않는 결정적 테스트여야 한다.
- 테스트 한 case의 시간·공간 비용은 자원 수에 대해 선형 이하여야 한다.
- 외부 mocking 프레임워크는 사용하지 않는다.
- 전처리기 치환, 링크 seam, 함수 포인터 중 하나를 선택할 수 있다.
- 축소 구현 뒤에는 같은 패턴을 `pthread_create`, `pthread_join`, `pthread_cond_wait`, `clock_gettime`, `philo_sleep_ms` 실패에도 어떻게 확장할지 설명해야 한다.

### 구현 후 자가 검증

- 정상 경로: 실패를 주입하지 않은 case에서 전체 초기화·정리가 성공한다.
- 경계값: 첫 번째·중간·마지막 호출 실패를 모두 순회한다.
- 실패 경로: 대상 오류 반환과 최종 소유권 상태를 함께 확인한다.
- 상태 변화: 각 case 시작 전에 fault plan과 주소 기록을 초기화한다.
- invariant: 성공한 획득 수와 성공적으로 정리된 고유 자원 수가 일치한다.
- resource cleanup: 실패 후 destructor 재호출에도 중복 파괴가 없다.
- 중복·누락 처리: 주소 단위로 중복 destroy와 누락 destroy를 구분한다.
- 동시성·비동기 문제: lifecycle 확장 시 join되지 않은 작업자가 남으면 공유 자원 파괴를 실패로 본다.
- 요구사항 충족 여부: 테스트가 특정 정상 스케줄이나 실제 OS 실패 확률에 의존하지 않는다.

### 구현 후 설명할 것

1. 시스템 호출 경계를 테스트 seam으로 만든 방법과 production 코드 침투 정도
2. k번째 실패 위치를 순회해 rollback 경로를 체계적으로 덮는 방식
3. 호출 횟수보다 최종 소유권 상태를 핵심 assertion으로 둔 이유
4. `_exit`·시계 fatal 경로를 자식 프로세스로 격리해야 하는 이유
5. fault injection, 반복 동시성 테스트, TSan이 각각 찾는 결함의 차이

### 원본 확인 위치

- Thread 07
- `test(init): 부분 뮤텍스 초기화 롤백 검증`
- `test(time): 단조 시계와 시계 실패 경로 검증`
- `test(thread): 지연된 작업자의 공통 시작 시각 검증`
- `test(thread): 시작 대기 실패 전파 검증`
- `test(monitor): 완료 상태와 오래된 사망 판정 검증`
- `test(routine): 중단된 식사의 카운터 불변식 검증`
- `test(lifecycle): 생성·결합·정리 실패 경로 검증`
- `test(main): 결합 실패 시 안전하지 않은 정리 방지`
- `tests/init_failure.c`, `tests/monotonic_clock.c`, `tests/start_barrier.c`, `tests/worker_wait_failure.c`
- `tests/terminal_state.c`, `tests/interrupted_meal.c`, `tests/lifecycle_failure.c`, `tests/main_unsafe.c`
- `tests/smoke.sh`, `tests/tsan.sh`
- 관련 Thread: 02의 초기화·작업자 lifecycle, 04의 시계·조건 변수, 05의 중단 commit, 06의 terminal 경쟁
