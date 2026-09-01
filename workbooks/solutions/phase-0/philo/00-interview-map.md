# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 자료 범위와 선별 원칙

이 워크북은 현재 프로젝트에서 확인된 다음 7개 개발 Thread의 커밋 기록만 사용한다.

- Thread 01: 실행 인자 계약과 수치 안전성
- Thread 02: 부분 초기화와 작업 스레드 수명주기
- Thread 03: 포크 소유권과 작업 루틴
- Thread 04: 단조 시간과 동기화된 시작
- Thread 05: 식사 완료 커밋과 목표 집계
- Thread 06: 종료 상태와 로그 선형화
- Thread 07: 빌드와 동시성 검증 파이프라인

확인된 원문 문서는 `01-runtime-argument-contract-and-numeric-safety.md`, `02-partial-initialization-and-worker-lifecycle-01.md`, `02-partial-initialization-and-worker-lifecycle-02.md`, `03-fork-ownership-and-worker-routine.md`, `04-monotonic-time-and-synchronized-start-01.md`, `04-monotonic-time-and-synchronized-start-02.md`, `05-meal-completion-commit-and-goal-accounting.md`, `06-terminal-state-and-log-linearization.md`, `07-build-and-concurrency-verification-pipeline-01.md`, `07-build-and-concurrency-verification-pipeline-02.md`다.

같은 커밋이 여러 Thread 문서에 반복 수록된 경우 한 행만 남기고 대표 Thread와 연관 Thread를 구분했다. 관련 위치는 원문에서 확인된 파일·함수·구조체·테스트만 적었다.

### 우선순위 기준

- **S**: 질문과 직접 구현 모두 반드시 준비할 가치가 높다.
- **A**: 질문 가치가 높고, 핵심 일부를 직접 구현할 가능성이 있다.
- **B**: 구현보다 설계 판단·개념 설명 준비가 중요하다.
- **C**: 별도 면접 문제로 만들 필요가 낮다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
|---|---:|---|---|---|---|---|---|---|
| C | 07 | `build(philo): 실행 파일 빌드 규약 구성` | `Makefile`, `.gitignore` | 기본 C 빌드 규약 | 프로젝트 실행에 필요하지만 독립 면접 문제로는 보일러플레이트 비중이 높다. | 낮음 | 낮음 | — |
| S | 01 | `feat(parse): 철학자 실행 인자 검증` | `include/philo.h`의 `t_config`; `src/parse.c`의 `parse_positive_long`, `philo_parse_args` | 수치 파싱, 입력 계약, 오버플로우 사전 방지 | 라이브러리 호출 암기보다 정수 표현 범위와 실패 계약을 직접 확인하기 좋다. **P01**의 대표 커밋이다. | 높음 | 높음 | 04, 05 |
| C | 01 | `feat(cli): 입력 계약을 실행 진입점에 연결` | `src/main.c`의 `print_usage`, `main` | 진입점 오류 처리 | 파서 결과를 진입점에 연결하는 일반적인 제어 흐름이라 별도 문제 가치가 낮다. | 낮음 | 낮음 | 02 |
| S | 01 | `fix(parse): 밀리초 인자의 상한 적용` | `src/parse.c`의 `philo_parse_args` | 파싱 범위와 도메인 상한 분리 | 문자열이 표현 가능한 값과 애플리케이션이 허용하는 값이 다르다는 점을 묻기 좋다. **P01**에 통합한다. | 높음 | 높음 | 04 |
| A | 02 | `feat(init): 테이블 저장소와 철학자 관계 초기화` | `include/philo.h`의 `t_philo`, `t_table`; `src/init.c`의 `assign_philos`, `philo_table_init`, `philo_table_destroy` | 링 구조 매핑, 소유권과 차용 포인터 | 인접 자원 매핑, 0/1 기반 인덱스, 공유 배열 수명이라는 일반화 가능한 지점이 있다. **P05**의 대표 커밋이다. | 높음 | 중간 | 03 |
| S | 02 | `feat(init): 뮤텍스 수명주기와 실패 롤백 구현` | `src/init.c`의 `init_forks`, `philo_table_init`, `philo_table_destroy`; `t_table.fork_count`, `state_ready`, `print_ready` | 부분 초기화, 롤백, 자원 상태 추적 | 중간 단계 실패 후 이미 확보한 자원만 정확히 정리하는 능력을 직접 확인할 수 있다. **P03**의 대표 커밋이다. | 높음 | 높음 | 03, 07 |
| B | 04 | `feat(time): 밀리초 시각 계산 함수 추가` | `src/time.c`의 `philo_now_ms`, `philo_sleep_ms` | 시간 추상화의 시작점 | 이후 단조 시계와 중단 가능한 대기로 개선되므로 단독 문제보다 설계 변화 설명에 적합하다. | 중간 | 낮음 | 01 |
| B | 04 | `fix(time): 짧은 대기 시간의 초과 지연 완화` | `src/time.c`의 `philo_sleep_ms` | 폴링 간격과 지연·CPU 사용량 절충 | 정밀도와 wake-up 빈도의 trade-off는 설명 가치가 있지만 정답이 하나인 구현 문제는 아니다. | 높음 | 낮음 | 05 |
| A | 06 | `feat(log): 상태 로그의 동시 출력 보호` | `src/state.c`의 `philo_has_ended`, `philo_finish`, `philo_log`, `philo_log_death`; `t_table.state_mutex`, `print_mutex` | 공유 상태와 출력 직렬화 | 두 종류의 공유 자원을 어떤 순서와 범위로 잠글지 묻는 기반이 된다. 최종 문제는 **P09**에 통합한다. | 높음 | 중간 | 05, 07 |
| S | 03 | `feat(routine): 철학자의 식사·수면·사고 흐름 구현` | `src/routine.c`의 `lock_forks`, `unlock_forks`, `record_meal_start`, `record_meal_done`, `eat_once`, `philo_routine` | 다중 자원 획득, 작업 루프, 정리 경로 | 교착 조건, 락 획득 순서, 조기 종료 시 해제 누락을 한 번에 확인할 수 있다. **P06**의 대표 커밋이며 식사 완료 부분은 **P08**과 연결된다. | 높음 | 높음 | 05, 06 |
| A | 06 | `feat(monitor): 사망과 식사 완료 조건 감시` | `src/monitor.c`의 `all_meals_done`, `find_dead_philo`, `philo_monitor` | 감시자 패턴과 상태 스냅샷 | 감시 루프의 역할 분리와 잠금 범위를 묻기 좋지만, 최종적인 원자성은 후속 수정에서 완성된다. **P09**에 통합한다. | 높음 | 중간 | 05 |
| S | 05 | `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단` | `src/routine.c`의 `record_meal_done`, `eat_once`, `philo_routine`; `t_table.full_count`, `ended` | 목표 도달 전이와 즉시 종료 | 목표 달성 상태를 누가 언제 확정하는지, 종료 뒤 추가 작업을 어떻게 막는지 확인할 수 있다. **P08**에 통합한다. | 높음 | 높음 | 06 |
| A | 02 | `feat(thread): 철학자 작업 스레드 시작과 종료` | `src/run.c`의 `join_started`, `philo_run` | 생성 성공 범위와 join 책임 | 부분 생성 실패 시 어떤 스레드까지 소유하는지 묻는 수명주기 문제의 기반이다. 최종 문제는 **P04**에 통합한다. | 높음 | 중간 | 04 |
| C | 02 | `feat(main): 입력부터 자원 정리까지 실행 흐름 연결` | `src/main.c`의 `main` | 애플리케이션 오케스트레이션 | 정상 경로의 연결 코드 비중이 높아 별도 구현 문제로 만들 필요가 낮다. | 낮음 | 낮음 | 01 |
| S | 02 | `fix(init): 포크 초기화 실패 시 중복 정리 방지` | `src/init.c`의 `init_forks`, `philo_table_destroy`; `t_table.fork_count` | 중앙집중식 정리와 중복 파괴 방지 | 내부 롤백과 외부 destructor가 동시에 같은 자원을 정리하면 생기는 이중 파괴를 정확히 짚는다. **P03**에 통합한다. | 높음 | 높음 | 07 |
| S | 03 | `fix(single): 철학자가 한 명일 때 포크 재잠금 방지` | `src/routine.c`의 `wait_single_philo`, `philo_routine` | 동일 뮤텍스 별칭과 단일 작업자 경계값 | 링 매핑에서 두 포인터가 같은 뮤텍스를 가리키는 특수 사례를 놓치지 않는지 확인하기 좋다. **P06**에 통합한다. | 높음 | 높음 | 02 |
| C | 01 | `fix(cli): 명령행 오류 출력 길이 계산` | `src/main.c`의 `ft_strlen`, `put_error`, `main` | 저수준 출력 길이 안전성 | 하드코딩된 길이 오류를 제거했지만 면접 핵심 역량을 대표하기에는 범위가 작다. | 낮음 | 낮음 | — |
| B | 07 | `test(smoke): 주요 입력과 종료 조건 검증` | `Makefile`; `tests/smoke.sh`의 `run_timeout`, 입력·단일 작업자·유한 식사 시나리오 | 종단 간 스모크 테스트 | 필수 회귀망이지만 개별 핵심 역량은 후속 실패 주입·동시성 테스트가 더 잘 대표한다. | 중간 | 낮음 | 01, 03, 05 |
| A | 04 | `fix(time): 단조 시계로 경과 시간 계산` | `include/philo.h`의 `int64_t` 시간 필드; `src/time.c`의 `clock_failure`, `philo_now_ms`, `philo_sleep_ms` | 단조 시계, 경과 시간, 정수 폭 | wall clock 변화와 플랫폼별 `long` 폭을 피하는 이유를 설명하고 작은 구현도 요구할 수 있다. **P02**의 대표 커밋이다. | 높음 | 중간 | 01, 05 |
| A | 07 | `test(init): 부분 뮤텍스 초기화 롤백 검증` | `tests/init_failure.c`; `tests/smoke.sh`의 심볼 치환 빌드 | 실패 주입과 자원 불변식 검증 | 드문 초기화 실패를 재현하고 중복 파괴·잔존 할당·반복 정리를 검증한다. **P03**, **P11**에 통합한다. | 높음 | 중간 | 02 |
| A | 04 | `test(time): 단조 시계와 시계 실패 경로 검증` | `tests/monotonic_clock.c`; `clock_gettime` 치환; `fork`, `waitpid` | 시간 의존성 주입과 fatal 경로 격리 | 외부 시계와 프로세스 종료를 결정적으로 검증하는 테스트 설계 가치가 높다. **P02**, **P11**에 통합한다. | 높음 | 중간 | 07 |
| S | 04 | `fix(thread): 시작 장벽으로 기준 시각 통일` | `include/philo.h`의 `start_cond`, `start_released`, `ready_count`, `run_error`; `src/routine.c`의 `wait_for_start`; `src/run.c`의 `release_start`, `philo_run` | 조건 변수 장벽, 공통 기준 시각, 실패 시 깨우기 | lost wake-up, 조건식, 공정한 시작 기준, 오류 전파를 함께 묻는 대표 동시성 문제다. **P07**의 대표 커밋이다. | 높음 | 높음 | 02 |
| A | 04 | `test(thread): 지연된 작업자의 공통 시작 시각 검증` | `tests/start_barrier.c`; `pthread_create` 치환 | 시작 지연을 통제한 장벽 검증 | 느린 작업자가 있어도 모든 작업자가 같은 기준 시각을 받는지 확인한다. **P07**, **P11**에 통합한다. | 높음 | 중간 | 02, 07 |
| A | 04 | `test(thread): 시작 대기 실패 전파 검증` | `tests/worker_wait_failure.c`; `pthread_cond_wait` 치환; `t_table.run_error` | 조건 변수 실패 전파와 전체 깨우기 | 한 작업자의 대기 실패가 다른 작업자를 영구 대기시키지 않는지 검증한다. **P07**, **P11**에 통합한다. | 높음 | 중간 | 02, 07 |
| B | 06 | `test(format): 필수 상태 로그 형식 검증` | `tests/smoke.sh`의 `check_log_format` | 출력 프로토콜 검증 | 외부 계약 검증은 중요하지만 정규식·파싱 자체는 핵심 구현보다 설명 보조에 가깝다. | 중간 | 낮음 | 07 |
| S | 06 | `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정` | `src/monitor.c`의 `philo_monitor`; `src/state.c`의 `philo_try_log_death`; `state_mutex`, `print_mutex` | 선형화 지점, check-then-act 재검증, 락 순서 | 하나의 사망 로그만 마지막에 남겨야 하는 강한 불변식을 실제 락 설계로 해결한다. **P09**의 대표 커밋이다. | 높음 | 높음 | 05, 07 |
| A | 06 | `test(monitor): 완료 상태와 오래된 사망 판정 검증` | `tests/terminal_state.c`; `pthread_mutex_unlock` 치환 | 상태 커밋 시점과 오래된 판정 재검증 | 잠금 해제 경계에 경쟁을 주입해 선형화 지점이 맞는지 검증한다. **P09**, **P11**에 통합한다. | 높음 | 중간 | 05, 07 |
| S | 05 | `fix(routine): 중단된 식사를 완료 횟수에서 제외` | `include/philo.h`의 `philo_sleep_ms` 반환 계약; `src/routine.c`의 `record_meal_done`, `eat_once`; `src/time.c`의 `philo_sleep_ms` | 완료 커밋과 중단 가능한 작업 | 시작된 작업과 완료된 작업을 구분하고, 종료 경쟁 중 카운터가 오염되지 않게 한다. **P08**의 대표 커밋이다. | 높음 | 높음 | 04, 06 |
| A | 05 | `test(routine): 중단된 식사의 카운터 불변식 검증` | `tests/interrupted_meal.c`; `philo_sleep_ms` 치환 | 실패 후 상태 미변경 검증 | 중단 시 `meals`와 `full_count`가 바뀌지 않는다는 트랜잭션 불변식을 직접 검증한다. **P08**, **P11**에 통합한다. | 높음 | 중간 | 04, 07 |
| S | 02 | `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파` | `include/philo.h`의 `PHILO_UNSAFE`, `threads_started`, `threads_joined`, `destroy_safe`; `src/run.c`; `src/init.c` | 생성·결합 실패와 안전한 파괴 경계 | join 실패 후 살아 있을 수 있는 작업자가 공유 메모리를 참조한다는 소유권 문제를 명시한다. **P04**의 대표 커밋이다. | 높음 | 높음 | 07 |
| A | 07 | `test(lifecycle): 생성·결합·정리 실패 경로 검증` | `tests/lifecycle_failure.c`; `pthread_create`, `pthread_join`, `pthread_mutex_destroy` 치환 | 실패 위치 전수 검사와 재시도 가능한 상태 | 생성·join·destroy의 여러 실패 인덱스를 순회하며 소유권 상태를 검증한다. **P04**, **P11**에 통합한다. | 높음 | 중간 | 02 |
| B | 02 | `test(main): 결합 실패 시 안전하지 않은 정리 방지` | `tests/main_unsafe.c`; `src/main.c`; `atexit`, `_exit` 관찰 | 비정상 종료와 사용자 공간 정리 회피 | join 실패 뒤 정상 종료 경로가 위험한 이유를 설명하기 좋지만 직접 구현보다는 설계 판단이 중요하다. | 높음 | 낮음 | 07 |
| A | 05 | `fix(state): 식사 완료 횟수의 정수 범위 확장` | `include/philo.h`의 `t_philo.meals` | 장기 실행 카운터의 정수 폭 | 입력 상한과 런타임 누적 상한이 다름을 보여 주며 카운터 overflow를 예방한다. **P08**에 통합하고 **P01**과 연결한다. | 높음 | 중간 | 01, 04 |
| B | 05 | `test(routine): 최대 목표 이후 식사 카운터 검증` | `tests/meal_counter_range.c`; `philo_sleep_ms` 치환 | 임계값 초과와 중복 집계 방지 | `INT_MAX` 경계를 넘는 누적과 `full_count`의 단일 전이를 검증하지만 독립 문제보다는 P08 보조 사례다. | 높음 | 낮음 | 07 |
| A | 07 | `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증` | `tests/concurrency.sh`의 `check_log`, `check_progress`, `check_terminal_line`; `tests/log_terminal_race.c` | 비결정적 실행의 관찰 가능 불변식 테스트 | 특정 스케줄을 고정하지 않고 진행성·종료성·로그 선형성을 검증하는 일반화 가치가 높다. **P10**의 대표 커밋이다. | 높음 | 중간 | 03, 06 |
| B | 07 | `test(tsan): ThreadSanitizer 검증 경로 추가` | `Makefile`의 `test-tsan`; `tests/tsan.sh`; `TSAN_REQUIRED` | 동적 race 검출 도구의 신뢰성 확인 | 도구 사용법보다 지원 여부 probe와 선택적·필수 정책 설명이 중요하다. | 높음 | 낮음 | 06 |
| B | 07 | `build: expose deterministic verification targets` | `Makefile`의 `test`, `check`, `test-tsan`, `ci`; `CC`, `CPPFLAGS`, `CFLAGS`, `LDFLAGS`, `LDLIBS` | 재현 가능한 검증 진입점과 도구 주입 | 빌드 경계 설계는 의미 있지만 직접 구현 문제보다 CI·테스트 운영 설명에 적합하다. | 높음 | 낮음 | 01–06 |
| B | 07 | `ci: add cross-platform C validation` | `.github/workflows/c-philo-ci.yml` | 다중 컴파일러·플랫폼 검증, CI 최소 권한 | GCC·Clang·macOS TSan을 분리한 판단은 설명 가치가 있으나 프로젝트 핵심 알고리즘은 아니다. | 중간 | 낮음 | 01–06 |

## 대표 면접 포인트와 상세 워크북 위치

| ID | 우선순위 | 대표 Thread / 커밋 | 통합한 역량 | 상세 문서 |
|---|---:|---|---|---|
| P01 | S | Thread 01 / `feat(parse): 철학자 실행 인자 검증` | 오버플로우 없는 양의 정수 파싱, 필드별 상한, 정수 폭 | [01-numeric-and-time.md#P01](01-numeric-and-time.md#P01) |
| P02 | A | Thread 04 / `fix(time): 단조 시계로 경과 시간 계산` | 단조 시계, 절대 deadline, 중단 가능한 대기, polling trade-off | [01-numeric-and-time.md#P02](01-numeric-and-time.md#P02) |
| P03 | S | Thread 02 / `feat(init): 뮤텍스 수명주기와 실패 롤백 구현` | 부분 초기화, 중앙 정리, 반복 호출 안전성, destroy 실패 재시도 | [02-resource-and-lifecycle.md#P03](02-resource-and-lifecycle.md#P03) |
| P04 | S | Thread 02 / `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파` | create/join 소유권, 안전하지 않은 teardown 구분, 호출자 전파 | [02-resource-and-lifecycle.md#P04](02-resource-and-lifecycle.md#P04) |
| P05 | A | Thread 02 / `feat(init): 테이블 저장소와 철학자 관계 초기화` | 링 자원 매핑, 배열 소유권, 차용 포인터, N=1 별칭 | [03-resource-ownership-and-scheduling.md#P05](03-resource-ownership-and-scheduling.md#P05) |
| P06 | S | Thread 03 / `feat(routine): 철학자의 식사·수면·사고 흐름 구현` | 교착 회피, 두 뮤텍스 획득, 모든 탈출 경로의 해제, 단일 철학자 | [03-resource-ownership-and-scheduling.md#P06](03-resource-ownership-and-scheduling.md#P06) |
| P07 | S | Thread 04 / `fix(thread): 시작 장벽으로 기준 시각 통일` | 조건 변수 predicate, spurious wake-up, 공통 시작 시각, abort broadcast | [03-resource-ownership-and-scheduling.md#P07](03-resource-ownership-and-scheduling.md#P07) |
| P08 | S | Thread 05 / `fix(routine): 중단된 식사를 완료 횟수에서 제외` | 완료 시점의 트랜잭션 커밋, 목표 달성 단일 전이, 중단 시 무변경 | [04-state-commit-and-linearization.md#P08](04-state-commit-and-linearization.md#P08) |
| P09 | S | Thread 06 / `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정` | check-then-act 재검증, 선형화 지점, 락 순서, 단일 terminal log | [04-state-commit-and-linearization.md#P09](04-state-commit-and-linearization.md#P09) |
| P10 | A | Thread 07 / `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증` | 정확한 스케줄 대신 외부 불변식 검증, timeout, 반복 시나리오 | [05-concurrency-testing.md#P10](05-concurrency-testing.md#P10) |
| P11 | A | Thread 07 / `test(init): 부분 뮤텍스 초기화 롤백 검증` | 시스템 호출 실패 주입, k번째 호출 실패, 프로세스 격리, 상태 중심 단언 | [05-concurrency-testing.md#P11](05-concurrency-testing.md#P11) |

## S/A 완전성 검증

아래 표의 모든 S/A 커밋은 독립 항목의 대표가 되었거나 다른 대표 항목에 명시적으로 통합되었다.

| 우선순위 | 커밋 메시지 | 상태 | 워크북 위치 |
|---|---|---|---|
| S | `feat(parse): 철학자 실행 인자 검증` | P01 독립 항목 | `01-numeric-and-time.md#P01` |
| S | `fix(parse): 밀리초 인자의 상한 적용` | P01에 통합 | `01-numeric-and-time.md#P01` |
| A | `feat(init): 테이블 저장소와 철학자 관계 초기화` | P05 독립 항목 | `03-resource-ownership-and-scheduling.md#P05` |
| S | `feat(init): 뮤텍스 수명주기와 실패 롤백 구현` | P03 독립 항목 | `02-resource-and-lifecycle.md#P03` |
| A | `feat(log): 상태 로그의 동시 출력 보호` | P09에 통합 | `04-state-commit-and-linearization.md#P09` |
| S | `feat(routine): 철학자의 식사·수면·사고 흐름 구현` | P06 독립 항목, 식사 커밋은 P08에 통합 | `03-resource-ownership-and-scheduling.md#P06`, `04-state-commit-and-linearization.md#P08` |
| A | `feat(monitor): 사망과 식사 완료 조건 감시` | P09에 통합 | `04-state-commit-and-linearization.md#P09` |
| S | `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단` | P08에 통합 | `04-state-commit-and-linearization.md#P08` |
| A | `feat(thread): 철학자 작업 스레드 시작과 종료` | P04에 통합 | `02-resource-and-lifecycle.md#P04` |
| S | `fix(init): 포크 초기화 실패 시 중복 정리 방지` | P03에 통합 | `02-resource-and-lifecycle.md#P03` |
| S | `fix(single): 철학자가 한 명일 때 포크 재잠금 방지` | P06에 통합 | `03-resource-ownership-and-scheduling.md#P06` |
| A | `fix(time): 단조 시계로 경과 시간 계산` | P02 독립 항목 | `01-numeric-and-time.md#P02` |
| A | `test(init): 부분 뮤텍스 초기화 롤백 검증` | P03의 검증, P11의 실패 주입 사례로 통합 | `02-resource-and-lifecycle.md#P03`, `05-concurrency-testing.md#P11` |
| A | `test(time): 단조 시계와 시계 실패 경로 검증` | P02의 검증, P11의 프로세스 격리 사례로 통합 | `01-numeric-and-time.md#P02`, `05-concurrency-testing.md#P11` |
| S | `fix(thread): 시작 장벽으로 기준 시각 통일` | P07 독립 항목 | `03-resource-ownership-and-scheduling.md#P07` |
| A | `test(thread): 지연된 작업자의 공통 시작 시각 검증` | P07의 검증, P11의 스케줄 지연 주입 사례로 통합 | `03-resource-ownership-and-scheduling.md#P07`, `05-concurrency-testing.md#P11` |
| A | `test(thread): 시작 대기 실패 전파 검증` | P07의 검증, P11의 API 실패 주입 사례로 통합 | `03-resource-ownership-and-scheduling.md#P07`, `05-concurrency-testing.md#P11` |
| S | `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정` | P09 독립 항목 | `04-state-commit-and-linearization.md#P09` |
| A | `test(monitor): 완료 상태와 오래된 사망 판정 검증` | P09의 검증, P11의 경쟁 경계 주입 사례로 통합 | `04-state-commit-and-linearization.md#P09`, `05-concurrency-testing.md#P11` |
| S | `fix(routine): 중단된 식사를 완료 횟수에서 제외` | P08 독립 항목, 대기 계약은 P02와 연결 | `04-state-commit-and-linearization.md#P08`, `01-numeric-and-time.md#P02` |
| A | `test(routine): 중단된 식사의 카운터 불변식 검증` | P08의 검증, P11의 함수 치환 사례로 통합 | `04-state-commit-and-linearization.md#P08`, `05-concurrency-testing.md#P11` |
| S | `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파` | P04 독립 항목 | `02-resource-and-lifecycle.md#P04` |
| A | `test(lifecycle): 생성·결합·정리 실패 경로 검증` | P04의 검증, P11의 실패 위치 순회 사례로 통합 | `02-resource-and-lifecycle.md#P04`, `05-concurrency-testing.md#P11` |
| A | `fix(state): 식사 완료 횟수의 정수 범위 확장` | P08에 통합, P01의 정수 폭 판단과 연결 | `04-state-commit-and-linearization.md#P08`, `01-numeric-and-time.md#P01` |
| A | `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증` | P10 독립 항목 | `05-concurrency-testing.md#P10` |

## 백지 구현 우선순위

1. **P09** 종료 상태와 terminal 로그 선형화
2. **P07** 조건 변수 기반 공통 시작 장벽
3. **P04** 부분 생성·join 실패를 포함한 작업자 수명주기
4. **P03** 부분 초기화 롤백과 반복·재시도 가능한 정리
5. **P08** 중단 가능한 작업의 완료 커밋과 목표 집계
6. **P06** 두 자원 획득의 교착 회피와 N=1 경계
7. **P01** 오버플로우 없는 양의 정수 파서
8. **P10** 비결정적 실행 trace의 불변식 검증기
9. **P02** 단조 시계 기반 중단 가능한 deadline 대기
10. **P11** 실패 위치를 순회하는 fault-injection 테스트
11. **P05** 링 구조 자원 관계 초기화

## 설명 연습 우선순위

1. **P04** join 실패가 일반 오류가 아니라 "파괴 불가" 상태인 이유
2. **P09** 사망 판정·종료 상태·출력을 하나의 선형화된 전이로 만드는 방법
3. **P07** 조건 변수는 이벤트가 아니라 predicate를 기다리는 도구라는 점
4. **P08** 시작된 식사와 완료된 식사를 분리하는 커밋 지점
5. **P06** 교착 회피와 starvation 방지가 서로 다른 보장이라는 점
6. **P03** 부분 초기화 상태를 count와 ready flag로 표현하는 이유
7. **P10** 정확한 실행 순서 대신 관찰 가능한 불변식을 테스트하는 이유
8. **P02** wall clock 대신 monotonic clock을 써야 하는 이유와 polling trade-off
9. **P01** 표현 범위 검사와 도메인 상한 검사를 분리하는 이유
10. **P11** 실패 주입 테스트가 호출 횟수보다 최종 소유권 상태를 단언해야 하는 이유
11. **P05** 테이블 소유 배열과 철학자의 차용 포인터 수명 관계

## 한 문제로 통합한 Thread 묶음

- **P01 수치 안전성**: Thread 01의 파싱·상한 + Thread 04의 `int64_t` 시간 폭 + Thread 05의 장기 누적 카운터
- **P02 시간 계약**: Thread 04의 단조 시계·deadline 대기 + Thread 05의 중단 가능한 식사 대기
- **P03 자원 롤백**: Thread 02의 부분 초기화·정리 + Thread 03의 뮤텍스 소유 관계 + Thread 07의 초기화 실패 주입
- **P04 작업자 수명주기**: Thread 02의 생성·join·파괴 안전성 + Thread 07의 lifecycle 실패 위치 순회
- **P05 링 매핑**: Thread 02의 저장소 초기화 + Thread 03의 인접 포크 소유 관계
- **P06 교착 회피**: Thread 03의 포크 획득·단일 철학자 + Thread 05·06의 종료 중 조기 탈출 정리
- **P07 시작 장벽**: Thread 02의 작업자 실행 수명주기 + Thread 04의 조건 변수·공통 시작 시각
- **P08 완료 커밋**: Thread 03의 식사 루틴 + Thread 05의 완료 카운터·정수 폭 + Thread 06의 목표 종료 상태
- **P09 terminal 선형화**: Thread 06의 상태·모니터·로그 + Thread 07의 terminal race 회귀 테스트
- **P10 동시성 불변식 테스트**: Thread 01·03·05·06의 관찰 계약 + Thread 07의 반복 실행·timeout·trace 검증
- **P11 실패 주입**: Thread 02의 init/lifecycle + Thread 04의 clock/condvar + Thread 05의 sleep 중단 + Thread 07의 테스트 빌드 경계
