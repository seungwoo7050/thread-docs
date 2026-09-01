# 수치 안전성과 시간 계약

<a id="P01"></a>
## [Thread 01 / `feat(parse): 철학자 실행 인자 검증`] 오버플로우 없는 실행 인자 파싱

### 면접 질문

`philo_parse_args`는 문자열 인자를 숫자로 바꾸면서 잘못된 문자, 0 이하의 값, 표현 범위 초과, 필드별 상한을 거부합니다. 이때 누적 계산 자체가 overflow를 일으키지 않도록 파서를 어떻게 설계하겠습니까?

꼬리 질문:

- `atoi`처럼 오류를 구분하기 어려운 변환 함수를 그대로 쓰면 어떤 문제가 생깁니까?
  - 모범답변: `atoi`는 변환 실패와 실제 0을 구분할 수 없고 overflow 동작도 안전한 오류 계약을 주지 않습니다. 잘못된 값을 truncate하거나 wrap된 유효값처럼 받아 실행 설정에 넣을 수 있습니다.
- "정수형에 들어가는가"와 "이 프로젝트가 허용하는 값인가"를 왜 분리해야 합니까?
  - 모범답변: 먼저 양의 십진수가 `int64_t`에 표현 가능한지 확인하는 것은 변환 안전성입니다. 그 다음 철학자 수 200, 시간과 식사 횟수 `INT_MAX` 상한을 적용하는 것은 이 프로젝트의 도메인 정책이라 독립적으로 바뀔 수 있습니다.
- 철학자 수, 시간 인자, 선택적 식사 횟수에 서로 다른 상한을 적용할 때 파서와 정책 검증의 책임을 어떻게 나누겠습니까?
  - 모범답변: 공용 digit parser는 선택적 `+`, 숫자 여부, 양수와 `INT64_MAX`만 검사합니다. `philo_parse_args`가 필드 위치에 따라 number는 200, 나머지는 `INT_MAX`를 확인한 뒤 안전하게 narrowing합니다.
- `long` 대신 고정 폭 정수형을 선택한 이유는 무엇입니까?
  - 모범답변: `long` 폭은 플랫폼에 따라 32비트 또는 64비트일 수 있습니다. 시간과 식사 누적의 표현 범위를 `int64_t`로 고정하면 parsing 상한과 overflow 검사가 플랫폼마다 달라지지 않습니다.

### 30초 모범 답변

문자열을 왼쪽부터 읽되 각 문자를 숫자로 확인하고, `현재값 × 10 + 다음 숫자`를 수행하기 전에 최대값을 넘는지 검사합니다. 변환 단계에서는 양의 10진수와 표현 가능 범위만 책임지고, 철학자 수 200 이하나 시간·식사 횟수의 `INT_MAX` 이하 같은 도메인 상한은 별도로 적용합니다. 이렇게 하면 signed overflow를 일으키지 않고, 파싱 규칙과 실행 정책도 분리할 수 있습니다. 빈 문자열, 숫자 없는 `+`, 0, 음수, 비숫자, 너무 긴 입력은 모두 실패해야 합니다.

### 답변 핵심 키워드

사전 overflow 검사 · `INT64_MAX` · 어휘 검증/도메인 검증 분리 · 양수 계약 · 선택 인자 · 고정 폭 정수 · 실패 시 부분 설정 방지

### 백지 구현

#### 구현 목표

`argc`와 `argv`를 받아 실행 설정을 채우는 `philo_parse_args`를 구현한다. 표준 변환 함수의 오류 동작에 의존하지 않고, 양의 10진수 파싱과 필드별 상한을 직접 검증한다.

#### 인터페이스 또는 함수 시그니처

```c
int philo_parse_args(int argc, char **argv, t_config *config)
{
    int64_t values[5];
    int     arg_index;

    if (config == NULL || argv == NULL || (argc != 5 && argc != 6))
        return (PHILO_ERR);
    arg_index = 1;
    while (arg_index < argc)
    {
        const char  *text;
        int64_t     value;
        int         i;

        text = argv[arg_index];
        if (text == NULL || text[0] == '\0')
            return (PHILO_ERR);
        i = (text[0] == '+');
        if (text[i] == '\0')
            return (PHILO_ERR);
        value = 0;
        while (text[i] != '\0')
        {
            int digit;

            if (text[i] < '0' || text[i] > '9')
                return (PHILO_ERR);
            digit = text[i] - '0';
            /* value * 10 + digit를 계산하기 전에 표현 범위를 확인한다. */
            if (value > (INT64_MAX - digit) / 10)
                return (PHILO_ERR);
            value = value * 10 + digit;
            i++;
        }
        if (value <= 0)
            return (PHILO_ERR);
        values[arg_index - 1] = value;
        arg_index++;
    }
    if (values[0] > 200 || values[1] > INT_MAX
        || values[2] > INT_MAX || values[3] > INT_MAX
        || (argc == 6 && values[4] > INT_MAX))
        return (PHILO_ERR);
    /* 모든 검증이 끝난 뒤에만 실행 가능한 설정을 공개한다. */
    config->number = (int)values[0];
    config->time_to_die = values[1];
    config->time_to_eat = values[2];
    config->time_to_sleep = values[3];
    config->must_eat = (argc == 6) ? (int)values[4] : 0;
    config->has_meal_limit = (argc == 6);
    return (PHILO_OK);
}
```

필요하면 파일 내부 정적 helper를 추가해도 된다.

#### 입력과 출력

- 입력: 프로그램명을 포함한 `argc`, `argv`
- 출력: 성공 시 `config`의 `number`, `time_to_die`, `time_to_eat`, `time_to_sleep`, `must_eat`, `has_meal_limit`
- 반환: 모든 계약을 만족하면 `PHILO_OK`, 하나라도 어기면 `PHILO_ERR`

#### 반드시 만족해야 할 조건

- 인자 개수는 프로그램명 포함 5개 또는 6개다.
- 숫자 문자열은 선택적인 선행 `+` 뒤에 하나 이상의 10진 숫자가 와야 한다.
- 값은 0보다 커야 한다.
- 누적 중 signed overflow가 발생해서는 안 된다.
- 철학자 수는 200 이하여야 한다.
- 세 시간 인자와 선택적 식사 횟수는 `INT_MAX` 이하여야 한다.
- 선택 인자가 없으면 `has_meal_limit`는 거짓이고, 있으면 참이다.
- 실패한 입력을 성공한 값처럼 truncate하거나 wrap-around해서는 안 된다.

#### 경계 조건

- `"1"`, `"+1"`
- `"0"`, `"+0"`
- `"+"`, 빈 문자열
- `"001"`
- `"200"`, `"201"`
- `INT_MAX`와 `INT_MAX + 1`을 나타내는 문자열
- `INT64_MAX`와 그보다 큰 문자열
- 매우 긴 숫자 문자열

#### 실패 조건

- `NULL` 포인터, 잘못된 인자 개수
- 공백, `-`, 알파벳, 구두점이 섞인 입력
- 숫자가 없는 부호
- 0 또는 범위 초과
- 필드별 도메인 상한 초과

#### 필요한 제약

- 시간 복잡도는 전체 입력 문자 수에 대해 선형이어야 한다.
- 추가 공간은 상수 크기여야 한다.
- overflow가 난 뒤 감지하는 방식은 허용하지 않는다.
- 원본과 다른 helper 구성도 계약을 만족하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 필수 인자만 있는 경우와 선택 인자가 있는 경우를 모두 확인했다.
- 경계값: 철학자 수 200, 시간·식사 횟수 `INT_MAX`, 내부 파서의 `INT64_MAX`를 확인했다.
- 실패 경로: 빈 문자열, `+`만 있는 문자열, 0, 음수, 비숫자, 상한 초과가 모두 실패한다.
- 상태 변화: 실패 중간에 `config`가 실행 가능한 설정처럼 남지 않도록 처리했다.
- invariant: 성공한 모든 수치는 양수이고 각 필드 상한 안에 있다.
- 시간·공간 복잡도: 각 문자열을 한 번만 순회하며 동적 할당을 쓰지 않는다.
- 요구사항 충족 여부: 표현 범위 검사와 도메인 정책 검사가 코드상 구분되어 있다.

### 구현 후 설명할 것

1. 누적 전에 overflow를 검사해야 하는 이유와 검사식을 도출한 방법
   - 모범답변: signed overflow가 발생한 뒤에는 이미 C의 정의되지 않은 동작이므로 사후 비교로 복구할 수 없습니다. `value * 10 + digit <= INT64_MAX`를 이항해 `value <= (INT64_MAX - digit) / 10`을 계산 전에 검사합니다.
2. 공용 숫자 parser와 `philo` 전용 상한 검증을 분리한 기준
   - 모범답변: parser는 문자열이 양의 `int64_t`인지까지만 보장합니다. 호출부는 필드 의미를 아니까 철학자 수 200과 각 `INT_MAX` 상한을 적용해 lexical conversion과 실행 정책을 분리합니다.
3. `strtol`을 사용할 수 있는 환경이라면 직접 파서와 비교해 어떤 trade-off가 있는지
   - 모범답변: `strtol`은 표준 범위 검사와 end pointer를 제공해 코드가 짧지만 선행 공백·부호 등 기본 허용 문법을 별도로 제한해야 하고 `long` 폭도 고려해야 합니다. 직접 parser는 프로젝트 문법과 `int64_t` 상한이 명확하지만 검증 코드를 직접 책임집니다.
4. 실패 시 `config`를 단계적으로 갱신할지 임시 설정에 만든 뒤 commit할지
   - 모범답변: 원본은 필드별 검증 직후 설정하지만 실패 반환 뒤 config를 사용하지 않는 계약입니다. 면접용 구현처럼 임시 값에 모두 검증한 뒤 commit하면 실패 시 반쯤 유효한 설정이 관찰되지 않아 API가 더 강해집니다.
5. `int64_t`와 `int`를 각각 어디에 쓰고, narrowing 전에 무엇을 확인했는지
   - 모범답변: 시간과 철학자별 meals 누적은 `int64_t`, 철학자 수와 설정의 must_eat는 `int`입니다. `int` 필드로 cast하기 전에 number는 200, must_eat는 `INT_MAX` 이하임을 확인합니다.

### 원본 확인 위치

- Thread 01
- `feat(parse): 철학자 실행 인자 검증`
- `fix(parse): 밀리초 인자의 상한 적용`
- `fix(time): 단조 시계로 경과 시간 계산`
- `include/philo.h`: `t_config`, `PHILO_OK`, `PHILO_ERR`
- `src/parse.c`: `parse_positive_long`, 후속 `parse_positive_i64`, `philo_parse_args`
- 관련 Thread: 04의 시간 필드 `int64_t` 전환, 05의 `t_philo.meals` 범위 확장

---

<a id="P02"></a>
## [Thread 04 / `fix(time): 단조 시계로 경과 시간 계산`] 단조 시계 기반 중단 가능한 deadline 대기

### 면접 질문

이 프로그램은 사망 판정과 로그 시간을 "경과 시간"으로 다룹니다. 왜 wall clock이 아니라 monotonic clock을 써야 하며, 긴 대기 중에도 종료 상태에 빠르게 반응하는 `philo_sleep_ms`를 어떻게 설계하겠습니까?

꼬리 질문:

- 시작 시각에서 경과 시간을 재는 코드가 `gettimeofday`를 쓰면 어떤 오동작이 가능합니까?
  - 모범답변: NTP 보정이나 관리자의 시각 변경으로 wall clock이 뒤로 가면 경과 시간이 음수가 되거나 사망이 늦어질 수 있고, 앞으로 점프하면 즉시 사망으로 오판할 수 있습니다. monotonic clock은 이런 달력 시각 변경과 분리됩니다.
- 일정한 횟수만큼 `usleep`을 반복하는 방식과 절대 deadline을 비교해 보세요.
  - 모범답변: 고정 횟수는 각 sleep의 scheduler 초과 지연이 누적되어 목표 시간이 계속 밀립니다. 매번 `deadline - now`를 다시 계산하면 늦게 깨어난 시간을 다음 반복에서 보상하고 deadline 도달 즉시 끝납니다.
- 종료 반응성을 높이려고 polling 간격을 줄이면 어떤 비용이 생깁니까?
  - 모범답변: 종료 flag를 더 자주 확인해 중단 지연은 줄지만 clock 조회, mutex lock과 wakeup 횟수가 늘어 CPU 사용과 경합이 커집니다. 원본은 남은 시간이 1ms보다 크면 500us, 아니면 100us를 쉽니다.
- 시계 조회 실패를 복구 가능한 오류로 둘지 프로세스 중단으로 둘지 어떻게 판단하겠습니까?
  - 모범답변: 이 프로젝트의 사망 판정과 모든 로그가 같은 시계에 의존해 대체 기준 없이 계속하면 안전한 의미를 보장할 수 없습니다. 그래서 고정 오류를 쓰고 `_exit(PHILO_ERR)`하며, 라이브러리라면 상위 정책이 결정하도록 오류를 전파할 수 있습니다.
- `philo_sleep_ms`가 `void`가 아니라 성공·중단을 반환해야 식사 카운터와 어떤 계약을 맺을 수 있습니까?
  - 모범답변: `PHILO_OK`일 때만 식사 duration을 끝까지 채웠다는 뜻이므로 그 뒤 `record_meal_done`을 호출할 수 있습니다. 종료 때문에 `PHILO_ERR`이면 포크만 해제하고 meals와 full_count를 커밋하지 않습니다.

### 30초 모범 답변

경과 시간은 시스템 시각 보정에 따라 뒤로 가거나 크게 점프하면 안 되므로 monotonic clock을 사용합니다. 대기는 반복 횟수가 아니라 `시작 시각 + duration`의 절대 deadline을 계산하고 현재 시각과 비교해야 누적 drift를 줄일 수 있습니다. 루프마다 공유 종료 상태를 잠금 아래 확인해 종료되면 중단 상태를 반환하고, deadline에 도달하면 완료를 반환합니다. 멀리 남았을 때는 비교적 긴 sleep, 끝에 가까울 때는 짧은 sleep을 써서 CPU 사용량과 초과 지연을 절충합니다.

### 답변 핵심 키워드

`CLOCK_MONOTONIC` · 경과 시간 · absolute deadline · drift 방지 · interruptible wait · 종료 predicate · coarse/fine polling · `int64_t` · 실패 계약

### 백지 구현

#### 구현 목표

단조 시계의 현재 밀리초를 구하는 함수와, deadline까지 기다리되 테이블의 종료 상태가 설정되면 즉시 중단하는 함수를 구현한다.

#### 인터페이스 또는 함수 시그니처

```c
int64_t philo_now_ms(void)
{
    struct timespec now;

    if (clock_gettime(CLOCK_MONOTONIC, &now) != 0)
        clock_fatal();
    return ((int64_t)now.tv_sec * 1000 + now.tv_nsec / 1000000);
}

int philo_sleep_ms(t_table *table, int64_t duration_ms)
{
    int64_t deadline;

    deadline = philo_now_ms() + duration_ms;
    while (1)
    {
        int64_t now;
        int64_t remaining;
        int     ended;

        now = philo_now_ms();
        if (now >= deadline)
            return (PHILO_OK);
        pthread_mutex_lock(&table->state_mutex);
        ended = table->ended;
        pthread_mutex_unlock(&table->state_mutex);
        if (ended)
            return (PHILO_ERR);
        remaining = deadline - now;
        /* mutex 밖에서 쉬어 종료를 설정할 다른 스레드를 막지 않는다. */
        if (remaining > 1)
            usleep(500);
        else
            usleep(100);
    }
}
```

면접용 축소 문제에서는 시계 조회가 실패하면 호출 가능한 `clock_fatal()`이 제공된다고 가정한다. 그 함수의 내부는 구현 대상이 아니다.

#### 입력과 출력

- `philo_now_ms`: 입력 없음, monotonic clock 기준 밀리초 반환
- `philo_sleep_ms`: 공유 종료 상태를 가진 `table`과 0 이상의 대기 시간
- 반환: deadline까지 도달하면 `PHILO_OK`, 종료 상태 때문에 중단되면 `PHILO_ERR`

#### 반드시 만족해야 할 조건

- wall clock이 아니라 monotonic clock을 사용한다.
- 초와 나노초를 밀리초로 바꾸는 계산은 충분한 정수 폭에서 수행한다.
- 대기는 절대 deadline을 기준으로 한다.
- 매 반복에서 종료 상태를 `state_mutex`로 보호해 읽는다.
- 종료가 확인되면 완료로 오인하지 않고 중단을 반환한다.
- deadline을 넘긴 경우에는 더 자지 않고 성공한다.
- polling 중 mutex를 잡은 채 sleep하지 않는다.

#### 경계 조건

- `duration_ms == 0`
- 1ms처럼 매우 짧은 대기
- 종료 상태가 호출 전에 이미 설정된 경우
- 마지막 polling 직전에 종료가 설정되는 경우
- sleep 호출이 요청보다 늦게 깨어나는 경우
- 초 경계와 나노초 경계를 넘는 시각 값

#### 실패 조건

- monotonic clock 조회 실패
- `table == NULL`을 허용하지 않는 계약 위반
- 음수 duration이 들어오는 계약 위반
- 종료 상태가 켜졌는데도 성공을 반환하는 경우

#### 필요한 제약

- mutex를 소유한 채 외부 sleep 함수에 들어가지 않는다.
- tight busy loop는 허용하지 않는다.
- polling 간격은 정확한 상수 자체보다 "멀리 있을 때와 가까울 때의 비용 절충"이 드러나야 한다.
- 원본과 다른 간격 정책도 종료 반응성과 CPU 사용량을 합리적으로 설명하면 허용한다.

### 구현 후 자가 검증

- 정상 경로: 0ms, 짧은 대기, 일반 대기에서 deadline 이후 성공한다.
- 경계값: 초·나노초를 밀리초로 변환할 때 overflow나 단위 오류가 없다.
- 실패 경로: 시계 실패가 정해진 fatal 계약으로 전달된다.
- 상태 변화: 대기 함수는 `ended`를 임의로 설정하지 않고 읽기만 한다.
- invariant: `PHILO_OK`는 deadline 도달을, `PHILO_ERR`는 종료로 인한 중단을 뜻한다.
- 동시성·비동기 문제: 종료 확인 뒤 mutex를 놓고 sleep하며 다른 스레드의 종료 갱신을 막지 않는다.
- 시간·공간 복잡도: 대기 횟수는 duration과 polling 정책에 비례하고 추가 공간은 상수다.
- 요구사항 충족 여부: wall clock 변경과 무관한 경과 시간을 제공한다.

### 구현 후 설명할 것

1. monotonic clock과 realtime clock의 의미 차이
   - 모범답변: realtime은 달력 시각이라 외부 보정으로 점프할 수 있습니다. monotonic은 임의 기준점부터 전진하는 경과 시간이라 duration, timeout과 사망 deadline 비교에 맞습니다.
2. 절대 deadline이 반복 sleep의 누적 오차를 줄이는 이유
   - 모범답변: 각 sleep이 늦게 깨어나도 다음 반복은 최초 deadline과 현재 시각을 비교합니다. 고정 sleep 횟수처럼 이전 초과 지연을 다음 요청 길이에 다시 더하지 않습니다.
3. polling 간격을 줄였을 때 반응성과 CPU 사용량이 어떻게 바뀌는지
   - 모범답변: 간격을 줄이면 ended를 관찰하는 최악 지연이 줄지만 더 많은 wakeup과 mutex 접근이 발생합니다. 원본의 두 단계 간격은 deadline 근처 정밀도와 평상시 비용을 절충합니다.
4. 중단과 완료를 반환값으로 구분해야 `record_meal_done`이 안전해지는 이유
   - 모범답변: sleep 중 종료된 식사는 시간 구간을 완주하지 않았으므로 완료 횟수가 아닙니다. 반환값을 확인해 중단이면 `record_meal_done`을 건너뛰어 meals와 full_count invariant를 지킵니다.
5. 시계 실패를 즉시 종료로 둔 원래 설계와 오류 반환 방식의 trade-off
   - 모범답변: 즉시 `_exit`하면 신뢰할 시간 없이 여러 thread가 계속되는 복잡한 복구를 피하지만 cleanup을 수행하지 못합니다. 오류 반환은 정리가 가능하지만 모든 호출 계층과 worker에 실패 전파·합의 경로를 추가해야 합니다.

### 원본 확인 위치

- Thread 04
- `feat(time): 밀리초 시각 계산 함수 추가`
- `fix(time): 짧은 대기 시간의 초과 지연 완화`
- `fix(time): 단조 시계로 경과 시간 계산`
- `test(time): 단조 시계와 시계 실패 경로 검증`
- `src/time.c`: `clock_failure`, `philo_now_ms`, `philo_sleep_ms`
- `include/philo.h`: `int64_t` 시간 필드와 함수 선언
- `tests/monotonic_clock.c`
- 관련 Thread: 01의 수치 폭, 05의 `fix(routine): 중단된 식사를 완료 횟수에서 제외`
