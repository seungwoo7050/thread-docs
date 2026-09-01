# 자원 초기화와 작업자 수명주기

<a id="P03"></a>
## [Thread 02 / `feat(init): 뮤텍스 수명주기와 실패 롤백 구현`] 부분 초기화 롤백과 재시도 가능한 정리

### 면접 질문

`philo_table_init`는 동적 배열, 상태 뮤텍스, 시작 조건 변수, 출력 뮤텍스, 여러 개의 포크 뮤텍스를 순서대로 확보합니다. 중간의 어느 단계에서든 실패할 수 있을 때, 이미 확보한 자원만 정확히 한 번 정리하고 `philo_table_destroy`를 반복 호출해도 안전하도록 상태를 어떻게 표현하겠습니까?

꼬리 질문:

- 초기화 함수 안에서 일부를 직접 정리하고, 호출자가 다시 destructor를 부르면 어떤 문제가 생길 수 있습니까?
- 여러 개의 같은 종류 자원은 boolean flag 하나가 아니라 count로 추적해야 하는 이유가 무엇입니까?
- `pthread_mutex_destroy` 자체가 실패하면 소유권 상태를 언제 지워야 합니까?
- 정리를 역순으로 하는 이유는 무엇이며, 이 프로젝트에서 항상 필수입니까?
- 부분 초기화된 객체를 "사용 불가지만 정리 가능"한 상태로 만드는 방법을 설명해 보세요.

### 30초 모범 답변

객체를 먼저 정리 가능한 영 상태로 만들고, 각 자원 획득이 성공한 직후에만 ready flag나 count를 갱신합니다. 어느 단계에서 실패해도 하나의 `philo_table_destroy` 경로로 보내면 이미 성공한 자원만 정리할 수 있고 중복 파괴도 막을 수 있습니다. 여러 포크 뮤텍스는 성공 개수로 추적하고, 개별 destroy가 성공한 뒤에만 그 소유권 표시를 줄여야 재시도가 가능합니다. 포인터는 실제 메모리를 해제한 뒤 `NULL`로 바꿔 반복 호출을 안전하게 만듭니다.

### 답변 핵심 키워드

부분 초기화 · 영 상태 · acquire 후 표시 · 중앙집중식 cleanup · ready flag · 성공 개수 · 역순 해제 · 멱등성 · retryable state · exactly-once destroy

### 백지 구현

#### 구현 목표

동적 포크 배열과 세 종류의 동기화 자원을 가진 축소형 테이블의 초기화·정리 함수를 구현한다. 모든 초기화 단계에 실패 가능성이 있고, 정리 함수도 일부 자원 파괴에 실패할 수 있다고 가정한다.

#### 인터페이스 또는 함수 시그니처

```c
typedef struct s_resources
{
    pthread_mutex_t *forks;
    size_t          fork_count;
    pthread_mutex_t state_mutex;
    pthread_cond_t  start_cond;
    pthread_mutex_t print_mutex;
    int             state_ready;
    int             start_cond_ready;
    int             print_ready;
}   t_resources;

int resources_init(t_resources *resources, size_t fork_total)
{
    // 직접 구현
}

int resources_destroy(t_resources *resources)
{
    // 직접 구현
}
```

면접관이 `malloc`, `pthread_mutex_init`, `pthread_cond_init`, 각 destroy 함수의 실패를 원하는 호출 위치에 주입할 수 있다고 가정한다.

#### 입력과 출력

- `resources_init` 입력: 비어 있는 `resources`, 1 이상의 포크 개수
- `resources_init` 반환: 전체 성공 시 `PHILO_OK`, 중간 실패 후 가능한 롤백까지 수행하면 `PHILO_ERR`
- `resources_destroy` 입력: 완전 초기화 또는 부분 초기화된 객체
- `resources_destroy` 반환: 모든 소유 자원이 정리되면 `PHILO_OK`, 아직 재시도해야 할 자원이 남으면 `PHILO_ERR`

#### 반드시 만족해야 할 조건

- 초기화 시작 시 모든 포인터, count, ready flag가 정리 가능한 값으로 설정되어야 한다.
- 자원 획득이 성공하기 전에는 소유권 표시를 세우지 않는다.
- 어느 단계에서 실패해도 성공한 자원만 destroy 대상으로 삼는다.
- 포크 뮤텍스는 실제 초기화 성공 개수만큼만 파괴한다.
- 같은 자원을 두 번 destroy하지 않는다.
- destroy가 실패한 자원의 소유권 표시는 유지해 다음 호출에서 재시도할 수 있어야 한다.
- 배열 메모리는 내부 자원에 더 이상 접근할 필요가 없을 때만 해제한다.
- 완전히 정리된 뒤 포인터는 `NULL`, count와 ready flag는 0이어야 한다.
- 완전히 정리된 객체에 `resources_destroy`를 다시 호출해도 추가 파괴가 없어야 한다.

#### 경계 조건

- 첫 번째 할당 실패
- 첫 번째 공용 뮤텍스 초기화 실패
- 조건 변수 초기화 실패
- 첫 번째·중간·마지막 포크 뮤텍스 초기화 실패
- 초기화 실패 직후 destructor 재호출
- 포크 destroy의 첫 번째·중간·마지막 호출 실패
- 공용 동기화 자원 destroy 실패
- 모든 자원이 이미 정리된 객체

#### 실패 조건

- 실패 전에 획득하지 않은 자원을 destroy하는 경우
- 내부 롤백과 외부 destructor가 같은 자원을 각각 파괴하는 경우
- destroy 실패에도 ready flag나 count를 지워 재시도 정보를 잃는 경우
- 동기화 자원이 남아 있는데 배열 메모리를 먼저 해제하는 경우
- 정리 후 dangling pointer가 남는 경우

#### 필요한 제약

- 전역 변수 없이 객체 내부 상태만으로 소유권을 판단한다.
- 전체 초기화와 부분 초기화 모두 동일한 destructor를 사용한다.
- 정상 경로 시간 복잡도는 포크 수에 대해 선형이어야 한다.
- 별도 동적 rollback 목록은 사용하지 않는다.
- 원본과 다른 해제 순서를 선택해도 의존 관계와 재시도 가능성을 설명할 수 있으면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 모든 자원이 초기화되고 한 번의 destroy로 완전히 정리된다.
- 경계값: 각 획득 단계가 첫 번째·중간·마지막에서 실패하는 경우를 확인했다.
- 실패 경로: 초기화 실패가 호출자에 전달되고, 이미 확보한 자원만 정리된다.
- 상태 변화: ready flag와 count는 해당 시스템 호출 성공 뒤에만 변한다.
- invariant: `fork_count`는 현재 초기화되어 아직 파괴되지 않은 포크 수와 일치한다.
- resource cleanup: 각 자원은 최대 한 번 성공적으로 destroy되며, 성공한 destroy 뒤에만 소유권 상태가 지워진다.
- 중복·누락 처리: 두 번 destroy와 누락 destroy를 모두 검사했다.
- 요구사항 충족 여부: 실패 후 재호출로 남은 자원을 계속 정리할 수 있다.

### 구현 후 설명할 것

1. count와 개별 ready flag를 각각 선택한 기준
2. 초기화 실패 시 모든 정리를 하나의 destructor에 위임한 이유
3. destroy 실패가 있을 때 멱등성과 재시도 가능성을 동시에 유지하는 방법
4. 소유권 상태를 시스템 호출 전이 아니라 성공 후 갱신해야 하는 이유
5. 구조체를 먼저 영 상태로 만드는 방식과 별도 rollback 스택 방식의 trade-off

### 원본 확인 위치

- Thread 02
- `feat(init): 뮤텍스 수명주기와 실패 롤백 구현`
- `fix(init): 포크 초기화 실패 시 중복 정리 방지`
- `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파`
- `include/philo.h`: `t_table`, `fork_count`, `state_ready`, `start_cond_ready`, `print_ready`
- `src/init.c`: `init_forks`, `philo_table_init`, `philo_table_destroy`
- `tests/init_failure.c`
- 관련 Thread: 03의 포크 배열 소유 관계, 07의 `test(init): 부분 뮤텍스 초기화 롤백 검증`

---

<a id="P04"></a>
## [Thread 02 / `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파`] 생성·결합 실패를 포함한 작업자 수명주기

### 면접 질문

`philo_run`이 여러 작업 스레드를 순서대로 만들다가 `pthread_create`에 실패하거나, 실행 종료 후 `pthread_join` 하나가 실패했다고 가정하겠습니다. 어떤 스레드를 현재 호출자가 소유하고 있는지 추적하고, 언제 테이블의 공유 자원을 파괴해도 안전한지 어떻게 판단하겠습니까?

꼬리 질문:

- 세 번째 스레드 생성이 실패하면 앞서 성공한 두 스레드에는 어떤 조치가 필요합니까?
- join 실패를 일반 실행 오류와 다른 상태로 구분한 이유는 무엇입니까?
- 한 join이 실패하더라도 나머지 join을 계속 시도해야 하는 이유는 무엇입니까?
- 살아 있을 가능성이 있는 작업자가 공유 테이블을 참조하는데 메인 스레드가 메모리를 해제하면 어떤 종류의 버그가 발생합니까?
- `main`이 안전하지 않은 상태에서 일반 `exit` 경로와 buffered stdio를 피하려 한 설계를 어떻게 평가하겠습니까?

### 30초 모범 답변

스레드 생성이 성공할 때마다 `threads_started`를 늘리고, 실패하면 전체 종료를 알린 뒤 실제로 시작된 스레드만 join합니다. join 성공 횟수도 별도로 추적합니다. create 실패는 시작된 스레드를 모두 회수했다면 일반 오류로 정리할 수 있지만, join 실패는 해당 작업자가 아직 공유 메모리를 사용할 수 있으므로 테이블 파괴가 안전하다고 증명되지 않습니다. 그래서 일반 오류와 파괴 불가 상태를 구분하고, 한 join이 실패해도 나머지 작업자는 계속 회수한 뒤 호출자에게 상태를 전달해야 합니다.

### 답변 핵심 키워드

started/joined ownership · 부분 생성 · cooperative stop · join 책임 · use-after-free · `PHILO_UNSAFE` · destroy gate · 오류 집계 · 호출자 전파 · 비정상 종료 경계

### 백지 구현

#### 구현 목표

N개의 작업자를 시작하고 종료 신호 뒤 모두 결합하는 축소형 실행 함수를 구현한다. 생성 실패와 join 실패를 구분하고, 공유 자원 파괴 가능 여부를 객체 상태에 남긴다.

#### 인터페이스 또는 함수 시그니처

```c
#define PHILO_OK      0
#define PHILO_ERR     1
#define PHILO_UNSAFE  2

int philo_run(t_table *table)
{
    // 직접 구현
}
```

면접용 환경에서는 `philo_routine`, 전체 작업자를 깨우는 `release_or_abort_start`, 종료를 기록하는 `philo_finish`가 제공된다고 가정한다. `t_table`에는 `config.number`, `philos`, `threads_started`, `threads_joined`, `destroy_safe`가 있다.

#### 입력과 출력

- 입력: 완전히 초기화되었고 아직 작업 스레드가 없는 `table`
- 반환:
  - `PHILO_OK`: 모든 작업자가 시작되고 정상 실행 뒤 모두 join됨
  - `PHILO_ERR`: 실행 실패가 있었지만 시작된 모든 작업자를 회수해 정리가 안전함
  - `PHILO_UNSAFE`: 하나 이상 join되지 않아 공유 자원 파괴 안전성을 보장할 수 없음
- 상태 출력: 시작·join 성공 개수와 `destroy_safe`

#### 반드시 만족해야 할 조건

- `pthread_create`가 성공한 스레드만 `threads_started`에 포함한다.
- 생성 도중 실패하면 대기 중인 작업자도 빠져나올 수 있도록 종료·시작 predicate를 풀어야 한다.
- 생성 실패 전 성공한 모든 작업자에 대해 join을 시도한다.
- join은 성공 여부와 무관하게 시작된 모든 스레드에 한 번씩 시도한다.
- join 성공 시에만 `threads_joined`를 늘린다.
- `threads_joined == threads_started`인 경우에만 공유 자원 파괴를 안전하다고 표시한다.
- 하나라도 join 실패가 있으면 `PHILO_UNSAFE`를 반환한다.
- 첫 join 실패 뒤에도 나머지 join을 생략하지 않는다.
- 호출자가 `PHILO_UNSAFE`일 때 일반 destructor를 실행하지 않도록 구분 가능한 계약을 제공한다.

#### 경계 조건

- 첫 번째 스레드 생성 실패
- 일부 스레드 생성 뒤 실패
- 마지막 스레드 생성 실패
- 작업자 수 1
- 첫 번째·중간·마지막 join 실패
- 둘 이상의 join 실패
- 생성 실패 롤백 중 join 실패
- 모든 스레드가 이미 자연 종료한 뒤 join

#### 실패 조건

- 생성되지 않은 `pthread_t`에 join하는 경우
- 생성 실패 후 대기 중인 작업자가 시작 장벽에서 영구 대기하는 경우
- 한 join 실패만 보고 나머지 작업자의 회수를 중단하는 경우
- join되지 않은 작업자가 있는데 `destroy_safe`를 참으로 만드는 경우
- `PHILO_UNSAFE`를 일반 오류처럼 처리해 공유 메모리를 해제하는 경우

#### 필요한 제약

- 취소 API에 의존하지 않고 공유 종료 상태와 join으로 수명주기를 닫는다.
- 시간 복잡도는 작업자 수에 대해 선형이어야 한다.
- 오류가 여러 개여도 반환 코드는 가장 위험한 상태를 보존해야 한다.
- 원본과 다른 상태 표현을 써도 "어떤 작업자를 누가 아직 소유하는가"를 판정할 수 있으면 허용한다.

### 구현 후 자가 검증

- 정상 경로: N개 생성 후 N개 join되고 `destroy_safe`가 참이다.
- 경계값: 생성 실패 위치 0, 중간, 마지막과 join 실패 위치 0, 중간, 마지막을 확인했다.
- 실패 경로: 생성 실패와 join 실패가 서로 다른 반환 상태로 전달된다.
- 상태 변화: `threads_started`와 `threads_joined`는 해당 POSIX 호출 성공 횟수와 일치한다.
- invariant: `0 <= threads_joined <= threads_started <= number`가 항상 유지된다.
- resource cleanup: join되지 않은 작업자가 있으면 테이블 자원 정리를 허용하지 않는다.
- 동시성·비동기 문제: 생성 실패 시 시작 장벽에 있던 작업자도 종료 predicate를 보고 깨어난다.
- 요구사항 충족 여부: 첫 오류 뒤에도 회수 가능한 모든 작업자를 끝까지 회수한다.

### 구현 후 설명할 것

1. create 실패와 join 실패가 자원 소유권에 미치는 차이
2. `PHILO_ERR`와 `PHILO_UNSAFE`를 별도 상태로 둔 이유
3. join 실패 후 나머지 join을 계속 시도하는 best-effort cleanup 전략
4. 공유 메모리를 해제할 수 있는 충분조건을 어떻게 정의했는지
5. 안전하지 않은 상태에서 프로세스를 즉시 끝내는 방식과 복구·재시도 방식의 trade-off

### 원본 확인 위치

- Thread 02
- `feat(thread): 철학자 작업 스레드 시작과 종료`
- `fix(thread): 시작 장벽으로 기준 시각 통일`
- `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파`
- `test(main): 결합 실패 시 안전하지 않은 정리 방지`
- `include/philo.h`: `PHILO_UNSAFE`, `threads_started`, `threads_joined`, `destroy_safe`
- `src/run.c`: `join_started`, `release_start`, `philo_run`
- `src/init.c`: `philo_table_destroy`
- `src/main.c`: `main`
- `tests/lifecycle_failure.c`, `tests/main_unsafe.c`
- 관련 Thread: 04의 시작 장벽, 07의 `test(lifecycle): 생성·결합·정리 실패 경로 검증`
