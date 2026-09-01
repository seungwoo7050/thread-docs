# 자원 소유권과 동시 실행 스케줄링

<a id="P05"></a>
## [Thread 02 / `feat(init): 테이블 저장소와 철학자 관계 초기화`] 링 구조 자원 매핑과 차용 포인터

### 면접 질문

테이블이 N개의 포크 뮤텍스를 연속 배열로 소유하고 각 철학자가 자신의 왼쪽·오른쪽 포크를 포인터로 참조합니다. `assign_philos`에서 이 링 관계를 어떻게 만들고, 배열 소유권과 철학자 내부 포인터의 수명 계약을 어떻게 설명하겠습니까?

꼬리 질문:

- 철학자 ID는 1부터 시작하지만 배열 인덱스는 0부터 시작할 때 어떤 off-by-one 오류가 흔합니까?
  - 모범답변: `forks[id]`를 왼쪽 포크로 쓰면 1번 철학자가 두 번째 포크를 잡고 마지막 ID N은 배열 밖을 가리킵니다. 배열 관계는 index i로 계산하고 표시용 ID만 `i + 1`로 둬야 합니다.
- 마지막 철학자의 오른쪽 포크를 첫 번째 포크에 연결하는 식을 설명해 보세요.
  - 모범답변: i번째 오른쪽은 `&forks[(i + 1) % N]`입니다. i가 N-1이면 `(N-1+1)%N == 0`이 되어 첫 포크로 돌아갑니다.
- 철학자 구조체가 포크를 소유하지 않고 차용하도록 한 장점과 위험은 무엇입니까?
  - 모범답변: mutex를 복사하지 않고 인접 철학자가 같은 실제 자원을 공유해 소유권과 파괴 지점이 table 하나로 모입니다. 반면 table이 배열을 이동·해제하거나 worker보다 먼저 파괴하면 내부 포인터가 dangling이 됩니다.
- N이 1이면 왼쪽·오른쪽 포인터가 어떻게 되며, 이 사실이 작업 루틴에 어떤 영향을 줍니까?
  - 모범답변: 두 식 모두 `&forks[0]`을 가리킵니다. 일반 `lock_forks`는 non-recursive mutex를 같은 thread가 두 번 잠가 영구 대기하므로 한 번만 잠그고 사망 종료를 기다리는 전용 경로가 필요합니다.
- 포크 배열을 재할당하거나 먼저 해제하면 철학자 포인터에는 어떤 문제가 생깁니까?
  - 모범답변: realloc이 backing storage를 옮기면 모든 left/right 포인터가 이전 주소를 가리키고, free하면 즉시 dangling pointer가 됩니다. 그래서 할당 뒤 주소를 고정하고 모든 worker join 뒤 mutex 파괴와 배열 free를 수행합니다.

### 30초 모범 답변

테이블이 포크 배열과 철학자 배열을 소유하고, i번째 철학자는 ID `i + 1`, 왼쪽 포크는 i번째 원소, 오른쪽 포크는 다음 원소를 참조하도록 링을 구성합니다. 마지막 원소의 다음 위치는 모듈러 연산으로 첫 번째 포크에 연결합니다. 철학자 내부 포인터는 차용 참조이므로 테이블 배열보다 오래 살 수 없고, 실행 중 포크 배열의 주소가 바뀌어서도 안 됩니다. N이 1이면 두 포인터가 같은 뮤텍스를 가리키므로 일반적인 두 번 잠금 경로와 분리해야 합니다.

### 답변 핵심 키워드

링 배열 · modulo · 0 기반 인덱스/1 기반 ID · owner/borrower · 주소 안정성 · 수명 종속 · aliasing · N=1

### 백지 구현

#### 구현 목표

이미 할당된 포크 배열과 철학자 배열을 연결해 각 철학자의 ID, 초기 상태, 인접 포크 포인터, 테이블 역참조를 설정한다.

#### 인터페이스 또는 함수 시그니처

```c
void assign_philos(t_table *table)
{
    int i;

    i = 0;
    while (i < table->config.number)
    {
        table->philos[i].id = i + 1;
        table->philos[i].meals = 0;
        table->philos[i].last_meal_ms = 0;
        table->philos[i].left_fork = &table->forks[i];
        table->philos[i].right_fork
            = &table->forks[(i + 1) % table->config.number];
        table->philos[i].table = table;
        i++;
    }
}
```

`t_table`은 `config.number`, `forks`, `philos`를 가지며, `t_philo`는 `id`, `meals`, `last_meal_ms`, `left_fork`, `right_fork`, `table`을 가진다고 가정한다.

#### 입력과 출력

- 입력: N개의 포크와 N개의 철학자 저장소가 할당된 `table`
- 출력: 반환값 없이 모든 철학자 원소의 관계와 초기 필드를 설정

#### 반드시 만족해야 할 조건

- 모든 철학자 ID는 1부터 N까지 중복 없이 설정된다.
- 모든 철학자의 초기 식사 횟수와 마지막 식사 시각은 초기값으로 설정된다.
- 각 포크는 정확히 두 인접 철학자에게 참조된다. N이 1인 경우 같은 철학자가 두 포인터로 동일 포크를 참조하는 별칭을 허용한다.
- 마지막 철학자와 첫 번째 철학자가 링으로 연결된다.
- 모든 철학자의 `table` 포인터는 입력 테이블을 가리킨다.
- 철학자 내부에는 포크 복사본을 만들지 않는다.
- 함수 실행 뒤 포크 배열 주소가 바뀌지 않는다는 수명 계약이 명확해야 한다.

#### 경계 조건

- N이 1인 경우
- N이 2인 경우
- 마지막 인덱스 N-1
- ID와 배열 인덱스 변환
- 매우 큰 허용 범위의 N

#### 실패 조건

- 마지막 철학자의 오른쪽 포크가 배열 밖을 가리키는 경우
- ID 0 또는 중복 ID가 생기는 경우
- 철학자가 자신의 인접 관계가 아닌 포크를 참조하는 경우
- 초기화 뒤 포크 배열을 이동해 내부 포인터가 dangling이 되는 경우
- N=1 별칭을 서로 다른 자원으로 가정하는 경우

#### 필요한 제약

- 시간 복잡도는 O(N), 추가 공간은 O(1)이어야 한다.
- 동적 할당은 이미 끝났다고 가정하며 이 함수에서는 새 메모리를 할당하지 않는다.
- 원본과 반대 방향으로 left/right를 정의해도 전체 코드에서 일관되고 링 불변식을 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: N=5에서 각 철학자의 두 포인터가 인접 포크를 가리킨다.
- 경계값: N=1과 N=2의 별칭·공유 관계를 직접 확인했다.
- 상태 변화: 모든 철학자 필드가 초기값으로 덮어써져 이전 쓰레기 값을 남기지 않는다.
- invariant: 모든 i에 대해 ID는 i+1이며 마지막과 첫 번째가 연결된다.
- resource cleanup: 철학자 구조체가 포크를 별도로 해제하지 않는 소유권 계약이 유지된다.
- 중복·누락 처리: N>1에서 각 포크가 두 인접 철학자에게 연결되는지 확인했다.
- 시간·공간 복잡도: 한 번의 순회와 상수 공간만 사용한다.

### 구현 후 설명할 것

1. 테이블을 owner, 철학자를 borrower로 둔 이유
   - 모범답변: 포크는 인접 철학자 둘이 공유하므로 개별 철학자 소유로 두면 중복 파괴 책임이 생깁니다. table이 배열의 유일 owner가 되고 철학자는 실행 동안만 포인터를 빌리면 allocation과 cleanup 경계가 하나가 됩니다.
2. 모듈러 연산이 마지막 원소의 연결을 단순화하는 방식
   - 모범답변: 모든 i에 같은 `(i + 1) % number` 식을 적용해 중간 원소는 다음 index, 마지막은 0을 얻습니다. 마지막 철학자만 따로 분기하는 off-by-one 경로가 사라집니다.
3. ID와 인덱스를 분리해 다룬 방법
   - 모범답변: 포인터 계산과 배열 순회는 0 기반 i만 사용하고, 출력·홀짝 스케줄 정책에 쓰는 ID는 `i + 1`로 저장합니다. 둘을 섞지 않아 배열 밖 접근을 피합니다.
4. N=1에서 동일 포인터 별칭이 생기는 이유와 후속 루틴에 미치는 영향
   - 모범답변: 다음 index도 modulo 1에서 0이라 left와 right가 동일합니다. 단일 철학자는 두 포크를 얻을 수 없으므로 한 번만 lock·log하고 `time_to_die + 1` 동안 종료 가능한 sleep을 한 뒤 unlock합니다.
5. 포인터 대신 포크 인덱스를 저장하는 대안의 장단점
   - 모범답변: index는 배열이 이동해도 table의 새 base로 다시 계산할 수 있고 직렬화하기 쉽습니다. 대신 매 접근마다 table을 통해 주소를 계산해야 하며, 현재처럼 배열 주소가 수명 동안 고정이면 직접 포인터가 단순합니다.

### 원본 확인 위치

- Thread 02
- `feat(init): 테이블 저장소와 철학자 관계 초기화`
- `include/philo.h`: `t_philo`, `t_table`
- `src/init.c`: `assign_philos`, `philo_table_init`, `philo_table_destroy`
- 관련 Thread: 03의 `feat(routine): 철학자의 식사·수면·사고 흐름 구현`, `fix(single): 철학자가 한 명일 때 포크 재잠금 방지`

---

<a id="P06"></a>
## [Thread 03 / `feat(routine): 철학자의 식사·수면·사고 흐름 구현`] 두 자원 획득의 교착 회피와 해제 불변식

### 면접 질문

각 철학자가 식사하려면 왼쪽과 오른쪽 포크 뮤텍스를 모두 가져야 합니다. 모든 철학자가 같은 순서로 첫 번째 포크를 잡으면 교착이 생길 수 있는데, 이를 어떻게 회피하고 종료 경쟁이나 중간 실패가 있어도 이미 잡은 포크를 빠짐없이 해제하겠습니까?

꼬리 질문:

- 이 상황에서 Coffman의 교착 조건 네 가지가 어떻게 성립합니까?
  - 모범답변: fork mutex는 상호 배제되고, 철학자는 첫 포크를 보유한 채 두 번째를 기다려 hold-and-wait가 됩니다. mutex는 강제로 빼앗지 못하고, 모두 같은 방향이면 각자가 다음 철학자의 포크를 기다리는 원형 대기가 생깁니다.
- 홀수·짝수 철학자가 반대 순서로 포크를 잡으면 어떤 순환 대기를 깨뜨립니까?
  - 모범답변: 홀수는 left, 짝수는 right를 먼저 잡아 링 전체가 한 방향의 다음 자원만 기다리는 동일한 cycle을 만들지 못하게 합니다. 원본은 ID parity에 따라 이 순서를 일관되게 적용합니다.
- 교착이 없다는 것과 starvation이 없다는 것은 왜 다른 주장입니까?
  - 모범답변: 교착 부재는 전체가 영원히 서로 기다리는 cycle이 없다는 뜻입니다. 특정 철학자는 mutex 공정성과 scheduler 선택 때문에 다른 철학자들에게 계속 선점당할 수 있어 개인별 진행 보장은 별도입니다.
- 첫 번째 포크를 잡은 뒤 종료 상태가 설정되면 어떤 경로로 빠져나와야 합니까?
  - 모범답변: 두 번째 lock으로 들어가기 전에 종료 predicate를 확인하고 `FORK_STOPPED`와 현재 held mask를 반환합니다. 호출자는 mask에 기록된 첫 포크만 unlock합니다.
- N=1에서 일반 `lock_forks`를 호출하면 왜 자기 자신을 다시 잠그게 됩니까?
  - 모범답변: left와 right가 같은 `pthread_mutex_t *`이기 때문입니다. 기본 mutex는 재귀 잠금이 아니어서 첫 lock을 보유한 thread가 같은 mutex의 두 번째 lock에서 자신이 풀기를 기다립니다.
- 락 해제를 함수 끝에만 두면 조기 반환이 추가될 때 어떤 유지보수 위험이 있습니까?
  - 모범답변: 종료·sleep 중단 같은 새 return이 cleanup 이전에 놓이면 한 포크가 영구 보유됩니다. 획득 상태를 명시하고 단일 cleanup 경로나 mask 기반 release를 쓰면 제어 경로가 늘어도 실제 보유분만 해제할 수 있습니다.

### 30초 모범 답변

두 자원을 모두 같은 전역 순서로 잠그거나, 이 구현처럼 철학자 그룹별로 첫 획득 방향을 달리해 원형 대기를 깨뜨릴 수 있습니다. 중요한 것은 교착 회피 규칙이 모든 작업자에게 일관되게 적용되고, 첫 포크 이후 어떤 종료·실패 경로에서도 획득한 자원만 정확히 해제하는 것입니다. N이 1이면 두 포인터가 같은 뮤텍스이므로 한 번만 잠근 채 종료를 기다리는 별도 경로가 필요합니다. 이 방식은 교착을 막지만 스케줄러 공정성까지 보장하지는 않으므로 starvation은 별도 문제입니다.

### 답변 핵심 키워드

mutual exclusion · hold-and-wait · circular wait · lock ordering · 비대칭 획득 · acquisition state · cleanup path · N=1 alias · deadlock≠starvation

### 백지 구현

#### 구현 목표

철학자의 두 포크를 교착 없이 획득하고, 성공·종료·실패 모든 경로에서 실제 획득한 포크만 해제하는 면접용 함수를 구현한다. 단일 철학자는 별도 결과로 처리한다.

#### 인터페이스 또는 함수 시그니처

```c
typedef enum e_fork_result
{
    FORK_ACQUIRED,
    FORK_STOPPED,
    FORK_SINGLE
}   t_fork_result;

t_fork_result acquire_forks(t_philo *philo, unsigned int *held_mask)
{
    pthread_mutex_t *first;
    pthread_mutex_t *second;
    unsigned int    first_bit;
    unsigned int    second_bit;

    *held_mask = 0;
    if (philo->left_fork == philo->right_fork)
    {
        pthread_mutex_lock(philo->left_fork);
        *held_mask = 1u;
        philo_log(philo, "has taken a fork");
        return (FORK_SINGLE);
    }
    if (philo->id % 2 == 0)
    {
        first = philo->right_fork;
        first_bit = 2u;
        second = philo->left_fork;
        second_bit = 1u;
    }
    else
    {
        first = philo->left_fork;
        first_bit = 1u;
        second = philo->right_fork;
        second_bit = 2u;
    }
    pthread_mutex_lock(first);
    *held_mask |= first_bit;
    philo_log(philo, "has taken a fork");
    if (philo_has_ended(philo->table))
        return (FORK_STOPPED);
    pthread_mutex_lock(second);
    *held_mask |= second_bit;
    philo_log(philo, "has taken a fork");
    if (philo_has_ended(philo->table))
        return (FORK_STOPPED);
    return (FORK_ACQUIRED);
}

void release_forks(t_philo *philo, unsigned int held_mask)
{
    if (held_mask & 1u)
        pthread_mutex_unlock(philo->left_fork);
    if (held_mask & 2u)
        pthread_mutex_unlock(philo->right_fork);
}
```

`held_mask`는 면접용 축소 장치이며, 어떤 포크를 실제로 획득했는지만 표현하면 된다. 종료 확인 함수와 포크 획득 로그 함수는 제공된다고 가정한다.

#### 입력과 출력

- 입력: 유효한 철학자와 두 포크 포인터, 비어 있는 획득 상태
- 출력: 두 포크 획득 성공, 종료로 인한 중단, 단일 포크 별칭 중 하나
- 부수 효과: 성공적으로 획득한 포크만 `held_mask`에 반영

#### 반드시 만족해야 할 조건

- N>1에서 모든 철학자가 동일 시점에 시작해도 원형 대기를 만들지 않는 일관된 획득 규칙이 있어야 한다.
- 두 포크를 모두 얻은 경우에만 `FORK_ACQUIRED`를 반환한다.
- 각 `pthread_mutex_lock` 성공 직후에만 해당 포크를 획득 상태에 기록한다.
- 첫 번째 포크 뒤 종료가 확인되면 두 번째 포크 획득을 진행하지 않고 첫 번째 포크를 해제할 수 있어야 한다.
- `release_forks`는 획득하지 않은 포크를 unlock하지 않는다.
- 같은 포크를 두 번 unlock하지 않는다.
- N=1 또는 두 포인터가 같은 경우 두 번 lock하지 않고 `FORK_SINGLE`로 구분한다.
- 로그는 실제 lock 성공 뒤에만 남긴다.

#### 경계 조건

- N=1에서 왼쪽·오른쪽 포인터 동일
- N=2에서 두 철학자가 같은 두 포크를 반대 관점으로 참조
- 첫 번째 포크 획득 직후 종료
- 두 번째 포크 획득 직후 종료
- 모든 철학자가 동시에 획득 시도
- 홀수·짝수 개수의 철학자

#### 실패 조건

- 전체 작업자가 동일한 방향으로 첫 포크를 잡아 순환 대기가 생기는 경우
- 포크를 잡기 전에 획득 로그를 출력하는 경우
- 조기 반환에서 첫 포크를 놓치거나, 획득하지 않은 두 번째 포크를 unlock하는 경우
- N=1에서 동일 뮤텍스를 두 번 잠그는 경우
- 서로 다른 함수가 상충하는 락 순서를 사용하는 경우

#### 필요한 제약

- 전역 중앙 잠금 하나로 모든 식사를 직렬화하는 해법은 사용하지 않는다.
- 유효하게 초기화된 뮤텍스의 `pthread_mutex_lock`은 성공한다고 가정하며, lock API 자체의 오류 주입은 구현 범위 밖이다.
- trylock 기반 재시도, 전역 자원 순서, 홀짝 비대칭 중 하나를 선택할 수 있으나 종료 반응성과 starvation trade-off를 설명해야 한다.
- 추가 상태는 상수 공간이어야 한다.
- 원본과 다른 교착 회피 전략도 교착 불가능성을 논리적으로 설명하고 해제 불변식을 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 두 포크 획득 뒤 둘 다 정확히 한 번 해제한다.
- 경계값: N=1과 N=2에서 자기 재잠금이나 순환 대기가 없다.
- 실패 경로: 첫 포크 뒤 종료해도 보유 자원이 남지 않는다.
- 상태 변화: `held_mask`는 실제 lock 성공과 unlock에 대응한다.
- invariant: 함수 반환 시 호출자가 보유한 포크 집합을 상태로 정확히 알 수 있다.
- resource cleanup: 모든 조기 반환 경로에서 획득한 자원 수와 해제한 자원 수가 일치한다.
- 동시성·비동기 문제: 선택한 순서가 전 작업자 사이의 원형 대기를 깨뜨린다.
- 요구사항 충족 여부: 교착 회피와 starvation 보장을 혼동하지 않았다.

### 구현 후 설명할 것

1. 선택한 교착 회피 규칙이 순환 대기를 없애는 논리
   - 모범답변: 원본과 같이 홀수 ID는 left, 짝수 ID는 right를 먼저 잡습니다. 인접 작업자가 모두 같은 방향으로 다음 fork를 보유·대기하는 링을 만들 수 없게 해 Coffman의 circular-wait 조건을 깹니다.
2. 획득 상태를 별도로 추적해 cleanup을 안전하게 만든 방법
   - 모범답변: left 성공 뒤 bit 0, right 성공 뒤 bit 1을 설정합니다. release는 mask만 보고 unlock하므로 첫 lock 뒤 종료, 두 lock 뒤 종료와 single alias 모두 획득하지 않은 mutex를 건드리지 않습니다.
3. 종료 확인을 어느 지점에 두고 왜 그 지점을 선택했는지
   - 모범답변: 각 lock 성공과 로그 뒤 다음 단계 전에 확인합니다. 첫 포크만 가진 상태에서 불필요한 두 번째 대기를 피하고, 두 포크를 얻은 직후 종료 경쟁도 식사 시작으로 넘기지 않습니다.
4. N=1을 일반 경로와 분리한 이유
   - 모범답변: 두 포인터가 같은 mutex라 일반 두 lock은 self-deadlock입니다. 한 번만 획득했다고 표시해 caller가 사망을 기다린 후 한 번만 해제하도록 별도 결과를 반환합니다.
5. 홀짝 비대칭, 전역 포크 순서, trylock·backoff 방식의 trade-off
   - 모범답변: 홀짝 방식은 원본 mapping과 ID만으로 단순하지만 공정성을 보장하지 않습니다. 주소/번호의 전역 순서는 교착 증명이 명확하고, trylock·backoff는 종료 반응성이 좋을 수 있으나 재시도 비용과 livelock·starvation 분석이 늘어납니다.

### 원본 확인 위치

- Thread 03
- `feat(routine): 철학자의 식사·수면·사고 흐름 구현`
- `fix(single): 철학자가 한 명일 때 포크 재잠금 방지`
- `src/routine.c`: `lock_forks`, `unlock_forks`, `eat_once`, `wait_single_philo`, `philo_routine`
- `src/init.c`: `assign_philos`
- 관련 Thread: 05의 중단된 식사 정리, 06의 종료 상태 확인, 07의 진행성 회귀 테스트

---

<a id="P07"></a>
## [Thread 04 / `fix(thread): 시작 장벽으로 기준 시각 통일`] 조건 변수 기반 공통 시작 장벽

### 면접 질문

스레드를 순서대로 생성하면 먼저 만들어진 철학자가 마지막 철학자보다 훨씬 먼저 실행될 수 있습니다. 모든 작업자가 준비된 뒤 하나의 `start_ms`와 `last_meal_ms`를 공유하며 동시에 출발하도록 조건 변수 장벽을 어떻게 설계하겠습니까?

꼬리 질문:

- 작업자는 왜 조건 변수를 기다리기 전에 `ready_count`를 갱신해야 합니까?
  - 모범답변: 조정자는 count가 N이 되어야 시작 시각을 commit합니다. 작업자가 대기부터 하면 자신의 준비를 알릴 수 없어 조정자와 서로 기다리므로 같은 mutex 아래 count 증가와 broadcast를 먼저 수행합니다.
- `pthread_cond_wait`를 `if`가 아니라 `while`로 감싸야 하는 이유는 무엇입니까?
  - 모범답변: condition wait는 spurious wake-up이 가능하고 다른 상태 변경 broadcast에도 깰 수 있습니다. mutex를 다시 얻은 뒤 `start_released`를 재검사하는 while만 predicate가 참일 때 진행을 보장합니다.
- signal이 먼저 발생해도 predicate를 사용하면 lost wake-up을 피할 수 있는 이유를 설명해 보세요.
  - 모범답변: 알림 자체를 기억하는 것이 아니라 `start_released`가 mutex 아래 지속 상태로 남습니다. 작업자가 늦게 wait 구간에 들어와도 먼저 predicate를 검사해 이미 참이면 잠들지 않습니다.
- 스레드 생성이 중간에 실패하면 이미 장벽에서 기다리는 작업자를 어떻게 깨웁니까?
  - 모범답변: 조정자가 `ended = 1`, `start_released = 1`을 같은 mutex 아래 설정하고 broadcast합니다. 실제 worker 수가 N보다 적어도 count 대기를 계속하지 않고 모두 오류로 빠집니다.
- 작업자 하나의 `pthread_cond_wait`가 실패했을 때 다른 작업자가 영원히 기다리지 않게 하려면 어떤 공유 상태를 바꿔야 합니까?
  - 모범답변: `run_error`, `ended`, `start_released`를 함께 세우고 condition을 broadcast합니다. 조정자와 다른 worker 모두 같은 predicate를 보고 정상 시작이 아닌 전체 실패로 종료합니다.
- 공통 시작 시각을 각 작업자가 깨어난 뒤 개별 기록하면 어떤 편향이 생깁니까?
  - 모범답변: broadcast 뒤 scheduler가 worker마다 다른 시점에 실행하므로 늦게 깨어난 철학자의 사망 deadline이 더 늦어집니다. 조정자가 한 시각을 모든 `last_meal_ms`에 미리 써야 동일한 생존 예산으로 출발합니다.

### 30초 모범 답변

조건 변수 자체가 이벤트를 저장하는 것이 아니라 공유 predicate의 변화 알림이라는 점이 핵심입니다. 각 작업자는 `state_mutex` 아래에서 `ready_count`를 늘리고, `start_released`가 참이 될 때까지 while 루프로 기다립니다. 조정자는 모든 작업자의 준비를 같은 mutex 아래 확인한 뒤 단 한 번 현재 시각을 읽어 `start_ms`와 모든 `last_meal_ms`에 기록하고, release predicate를 세운 뒤 broadcast합니다. 생성이나 wait 실패 시에는 `ended`와 `run_error`, `start_released`를 함께 갱신하고 broadcast해 대기자를 모두 탈출시켜야 합니다.

### 답변 핵심 키워드

condition predicate · mutex 보호 · `while` wait · spurious wake-up · ready count · single timestamp commit · broadcast · abort path · failure propagation

### 백지 구현

#### 구현 목표

N개의 작업자가 준비를 보고한 뒤 조정자가 공통 기준 시각을 한 번 기록하고 모두를 해제하는 재사용하지 않는 one-shot 시작 장벽을 구현한다. 준비 중 오류가 나면 전체를 중단하고 모든 대기자를 깨운다.

#### 인터페이스 또는 함수 시그니처

```c
int worker_wait_for_start(t_philo *philo)
{
    t_table *table;
    int     failed;

    table = philo->table;
    failed = 0;
    pthread_mutex_lock(&table->state_mutex);
    table->ready_count++;
    pthread_cond_broadcast(&table->start_cond);
    while (!table->start_released)
    {
        if (pthread_cond_wait(&table->start_cond,
                &table->state_mutex) != 0)
        {
            table->run_error = 1;
            table->ended = 1;
            table->start_released = 1;
            pthread_cond_broadcast(&table->start_cond);
            failed = 1;
        }
    }
    if (table->ended || table->run_error)
        failed = 1;
    pthread_mutex_unlock(&table->state_mutex);
    return (failed ? PHILO_ERR : PHILO_OK);
}

int coordinator_release_start(t_table *table, int abort_run)
{
    int     i;
    int     status;
    int64_t start_ms;

    status = PHILO_OK;
    pthread_mutex_lock(&table->state_mutex);
    while (!abort_run && table->ready_count < table->config.number)
    {
        if (pthread_cond_wait(&table->start_cond,
                &table->state_mutex) != 0)
        {
            table->run_error = 1;
            abort_run = 1;
            status = PHILO_ERR;
        }
    }
    if (table->run_error)
    {
        abort_run = 1;
        status = PHILO_ERR;
    }
    start_ms = philo_now_ms();
    table->start_ms = start_ms;
    i = 0;
    while (i < table->config.number)
        table->philos[i++].last_meal_ms = start_ms;
    if (abort_run)
    {
        table->ended = 1;
        status = PHILO_ERR;
    }
    /* 기준 시각과 오류 상태를 모두 commit한 뒤 release를 공개한다. */
    table->start_released = 1;
    pthread_cond_broadcast(&table->start_cond);
    pthread_mutex_unlock(&table->state_mutex);
    return (status);
}
```

`t_table`에는 `state_mutex`, `start_cond`, `ready_count`, `start_released`, `run_error`, `ended`, `start_ms`, `config.number`, `philos`가 있다고 가정한다.

#### 입력과 출력

- 작업자 함수 입력: 자신의 철학자 객체
- 작업자 함수 반환: 정상 해제면 `PHILO_OK`, 전체 중단 또는 조건 변수 실패면 `PHILO_ERR`
- 조정자 함수 입력: 테이블과 이미 알려진 abort 여부
- 조정자 함수 반환: 정상 시작 해제면 `PHILO_OK`, 실패 상태로 전체를 깨웠으면 `PHILO_ERR`

#### 반드시 만족해야 할 조건

- `ready_count`, `start_released`, `run_error`, `ended`, 시작 시각 필드는 같은 mutex로 보호한다.
- 작업자는 준비되었다는 사실을 기록한 뒤 조정자에게 알린다.
- 모든 condition wait는 predicate를 검사하는 `while` 안에서 수행한다.
- 조정자는 정상 경로에서 `ready_count == number`가 될 때까지 기다린다.
- 공통 시각은 조정자가 한 번만 읽고 테이블과 모든 철학자에 같은 값으로 기록한다.
- 시작 관련 상태를 완전히 기록한 뒤 `start_released`를 참으로 만들고 broadcast한다.
- 생성 실패나 wait 실패가 있으면 종료·오류 predicate를 설정하고 모든 대기자를 broadcast로 깨운다.
- 깨어난 작업자는 release만 보지 말고 종료·오류 상태도 확인한다.
- 어떤 실패 경로에서도 조건 변수를 기다리는 스레드가 남아서는 안 된다.

#### 경계 조건

- 작업자 수 1
- 마지막 작업자가 늦게 준비되는 경우
- 작업자 준비 알림이 조정자의 wait보다 먼저 발생한 경우
- spurious wake-up
- 생성 실패로 실제 작업자 수가 목표보다 적은 경우
- 첫 번째 작업자의 condition wait 실패
- broadcast 직전 또는 직후 오류 상태 변경

#### 실패 조건

- `if` 한 번만 검사해 spurious wake-up 뒤 조기 진행하는 경우
- 준비 count와 release predicate를 서로 다른 잠금으로 보호하는 경우
- 오류 시 broadcast하지 않아 일부 작업자가 영구 대기하는 경우
- 작업자별로 서로 다른 시작 시각을 기록하는 경우
- mutex를 잡지 않고 predicate를 읽거나 쓰는 경우
- 시작 해제 전에 일부 작업자가 루틴 본문을 실행하는 경우

#### 필요한 제약

- busy waiting을 사용하지 않는다.
- 장벽은 한 번만 사용하며 세대 번호를 구현할 필요는 없다.
- 조건 변수 API의 반환 실패를 무시하지 않는다.
- 정상·중단 경로 모두 mutex를 정확히 한 번 해제하고 반환해야 한다.
- 원본과 다른 필드 이름이나 helper 분할도 predicate와 실패 전파 계약을 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 모든 작업자가 준비되기 전에는 누구도 루틴 본문으로 진행하지 않는다.
- 경계값: 작업자 수 1과 마지막 작업자의 큰 지연에서도 공통 시각이 같다.
- 실패 경로: 생성 실패와 condition wait 실패에서 모든 대기자가 빠져나온다.
- 상태 변화: 시작 시각과 각 `last_meal_ms`가 release predicate보다 먼저 commit된다.
- invariant: `0 <= ready_count <= number`이며 정상 해제 뒤 모든 작업자의 기준 시각이 동일하다.
- 동시성·비동기 문제: 조기 signal, spurious wake-up, 지연된 작업자에 의존하지 않는다.
- 중복·누락 처리: 각 작업자는 ready count를 정확히 한 번 증가시킨다.
- 요구사항 충족 여부: 오류가 한 스레드에만 머무르지 않고 전체 실행 상태로 전파된다.

### 구현 후 설명할 것

1. 조건 변수보다 predicate가 핵심인 이유
   - 모범답변: condition signal은 상태를 저장하지 않고 wait 중인 thread를 깨울 힌트일 뿐입니다. 실제 진행 가능 여부는 mutex로 보호한 `ready_count`와 `start_released`에 남아 조기·중복 알림과 무관하게 판정됩니다.
2. `while` 재검사로 spurious wake-up과 조기 알림을 처리하는 방식
   - 모범답변: wait가 반환하면 mutex를 다시 소유한 상태에서 predicate를 검사합니다. 아직 false면 이유와 무관하게 다시 기다리고, 알림이 먼저 왔어도 predicate가 true면 애초에 wait하지 않습니다.
3. 공통 시각과 모든 `last_meal_ms`를 한 임계 구역에서 기록한 이유
   - 모범답변: worker가 release를 보고 나가기 전에 모든 기준 값을 같은 `philo_now_ms()` 결과로 원자적으로 공개합니다. monitor와 worker가 일부만 갱신된 시각을 관찰하는 것을 막습니다.
4. 생성·대기 실패 시 release predicate까지 열어야 하는 이유
   - 모범답변: ended나 run_error만 세우고 `start_released`가 false면 worker의 while 조건은 계속 참이라 다시 기다릴 수 있습니다. release를 열고 broadcast해야 목표 count에 도달할 수 없는 장벽을 강제로 해제합니다.
5. one-shot barrier와 재사용 가능한 barrier의 상태 설계 차이
   - 모범답변: 현재는 `start_released`가 한 번 true가 되면 되돌아가지 않아 generation 구분이 필요 없습니다. 재사용 장벽은 다음 회차의 count 초기화와 이전 wake-up을 구분할 generation 번호, 참가자 이탈 정책이 필요합니다.

### 원본 확인 위치

- Thread 04
- `fix(thread): 시작 장벽으로 기준 시각 통일`
- `test(thread): 지연된 작업자의 공통 시작 시각 검증`
- `test(thread): 시작 대기 실패 전파 검증`
- `include/philo.h`: `start_cond`, `start_released`, `ready_count`, `run_error`
- `src/routine.c`: `wait_for_start`, `philo_routine`
- `src/run.c`: `release_start`, `philo_run`
- `tests/start_barrier.c`, `tests/worker_wait_failure.c`
- 관련 Thread: 02의 스레드 생성·join 수명주기, 07의 실패 주입 테스트
