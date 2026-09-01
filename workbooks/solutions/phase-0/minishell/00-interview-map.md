# 개발자 기술면접 워크북 마스터 인덱스

## 사용 범위와 원칙

이 인덱스는 현재 GPT 프로젝트에서 확인된 DevThread 01~11의 분할 문서와 커밋 작업 기록만을 바탕으로 작성했다. 원격 저장소의 현재 상태, 커밋 해시, 기록에 나타나지 않은 파일·함수·세부 동작은 사용하지 않았다. 같은 역량을 반복하는 커밋은 대표 면접 포인트 P01~P17로 묶었으며, S/A 항목은 모두 독립 상세 항목 또는 명시적 통합 대상으로 연결했다.

프로젝트에서 확인되는 범위는 C99와 POSIX process·file descriptor를 사용하는 제한된 `small-shell`이다. 확인된 문법은 `|`, `;`, `&&`, `||`, `<`, `>`, `>>`, `<<`이며, 확인된 builtin은 `echo`, `pwd`, `cd`, `env`, `export`, `unset`, `exit`다. background 실행, job control, subshell, glob, 완전한 POSIX shell 호환은 기록상 범위 밖이다.

## 우선순위 기준

| 우선순위 | 의미 |
| --- | --- |
| S | 반드시 준비. 질문과 직접 구현 모두 가치가 높다. |
| A | 준비 가치가 높다. 질문 또는 핵심 구현 가능성이 높다. |
| B | 구현보다 설계·개념 설명 준비가 중요하다. |
| C | 별도 면접 준비 항목으로 만들 필요가 낮다. |

## 검토한 Thread 기록

| Thread | 확인한 프로젝트 기록 | 대표 상세 포인트 |
| --- | --- | --- |
| 01 | `01-command-loop-phase-ordering-and-state-lifetime-01.md`, `-02.md` | P03, P16에 통합 |
| 02 | `02-quote-preserving-shell-tokenization.md` | P01, P15 |
| 03 | `03-command-graph-parsing-ownership-and-public-error-contracts-01.md` ~ `-03.md` | P02, P04 |
| 04 | `04-quote-aware-word-expansion-and-deferred-evaluation-01.md` ~ `-03.md` | P03, P15 |
| 05 | `05-persistent-shell-state-and-the-builtin-parent-child-boundary-01.md` ~ `-03.md` | P05~P08 |
| 06 | `06-process-fd-and-exit-status-lifecycles-in-multi-stage-pipelines-01.md`, `-02.md` | P09, P10 |
| 07 | `07-redirection-ordering-and-parent-stream-transactions-01.md`, `-02.md` | P11, P16 |
| 08 | `08-heredoc-precollection-quoting-semantics-storage-and-recovery-01.md` ~ `-04.md` | P12, P13, P16 |
| 09 | `09-system-call-and-allocation-fault-injection-with-command-level-recovery-01.md` ~ `-06.md` | P14, P16 |
| 10 | `10-amortized-linear-assembly-with-dynamic-buffers-and-long-input-performance-01.md`, `-02.md` | P15 |
| 11 | `11-deterministic-verification-pipeline-for-regression-fault-and-lifecycle-testing-01.md` ~ `-03.md` | P17 및 각 포인트의 검증 근거 |

## 전체 Thread/커밋 선별 결과

표의 한 행은 커밋 하나가 아니라 하나의 면접 역량 단위다. 같은 역량을 보강하는 커밋은 같은 행 또는 대표 포인트에 통합했다.

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C | 01 | `docs(readme): 프로젝트 목적과 초기 규약 정의`<br>`build(shell): C99 실행 골격 구성` | `README.md`, `Makefile`, `src/main.c` | 프로젝트 범위·빌드 골격 | 설정·문서·초기 boilerplate 자체는 기본기 평가력이 낮다. | 낮음 | 낮음 | 전체 Thread | 상세 없음 |
| A | 01 | `feat(input): 표준 입력 반복과 EOF 처리 연결` | `src/input.c`: `shell_read_line`, `shell_loop`, `read_plain_line` | 명령 loop와 입력 수명 | EOF, 입력 실패, shell 수명은 일반화 가능한 I/O 상태 문제다. | 높음 | 중간 | 08, 09, 11 | [P16](04-failure-performance-and-verification.md#p16)에 통합 |
| S | 01 / 04 | `feat(exec): 조건 연결자와 지연 확장 실행` | `src/exec.c`: `execute_pipeline_list_ctx`, `execute_one_pipeline`, `expand_one_pipeline` | 단계 순서·지연 평가 | 실행 시점 상태와 분기 의미를 결정하는 핵심 orchestration이다. | 높음 | 높음 | 03, 08 | [P03](01-lexing-parsing-and-expansion.md#p03) |
| S | 02 | `feat(lexer): 인용 단어와 토큰 수명 관리`<br>`feat(lexer): 셸 연산자를 토큰으로 구분` | `include/shell.h`: `t_token`<br>`src/token.c`: `tokenize_line`, `read_word`, `free_tokens` | quote-aware lexer | 언어 처리의 의미 보존, 경계 조건, 메모리 정리가 모두 드러난다. | 높음 | 높음 | 08, 10 | [P01](01-lexing-parsing-and-expansion.md#p01) |
| S | 02 / 08 | `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존` | `t_token.quoted`, `t_redir.heredoc_quoted`<br>`src/parser.c`, `src/heredoc.c` | 단계 간 semantic metadata 전달 | 초기 단계 정보 손실이 후속 의미론 버그로 이어지는 대표 사례다. | 높음 | 중간 | 01, 04 | P01과 [P12](03-process-io-and-resource-lifecycle.md#p12)에 통합 |
| S | 03 | `feat(parser): 명령 트리 소유권 모델 정의`<br>`feat(parser): pipe로 명령을 pipeline에 결합` | `include/shell.h`: `t_command`, `t_pipeline`, `t_sequence`<br>`src/parser.c`: `parse_tokens`, `free_pipeline` | 명령 그래프·소유권 | 자료구조, 문법 계층, lifecycle을 동시에 묻기 좋다. | 높음 | 높음 | 02, 06, 09 | [P02](01-lexing-parsing-and-expansion.md#p02) |
| S | 03 / 09 | parser 할당 실패 전파·`parse_failure` 경로 | `src/parser.c`: `parse_failure`, fallible add/new 경로 | 부분 생성 rollback | C에서 예외 없이 실패 원자성과 누수 방지를 구현하는 핵심 지점이다. | 높음 | 높음 | 08, 11 | P02와 [P14](04-failure-performance-and-verification.md#p14)에 통합 |
| A | 03 | `fix(parser): 오류 출력 포인터 없이도 구문 실패 반환`<br>`test(parser): 공개 parser 오류 반환 검증` | `src/parser.c`: `shell_parse_line`, `shell_sequence_free`<br>`tests/parser_api.c` | 공개 API 오류 계약 | 선택 출력 포인터와 반환값·소유권을 분리하는 API 설계 문제다. | 높음 | 중간 | 09, 11 | [P04](01-lexing-parsing-and-expansion.md#p04) |
| A | 03 | `feat(parser): hook 기반 sequence 실행 seam 제공` | `src/parser.c`: `shell_execute_sequence`<br>`include/shell.h`: `t_executor_hooks` | 컴포넌트 경계·테스트 seam | parser 결과를 실제 process 실행 없이 검증할 수 있는 경계 설계다. | 중간 | 낮음 | 11 | P04에 통합 |
| S | 04 | `feat(expand): 환경과 종료 상태 단어 확장`<br>`feat(expand): argv와 리다이렉션 확장 연결` | `src/expand.c`: `expand_word`, `expand_pipeline`, `shell_dequote_word` | 확장 의미와 late binding | 환경·직전 상태·quote context가 결합되는 핵심 언어 런타임 문제다. | 높음 | 높음 | 01, 02, 05 | [P03](01-lexing-parsing-and-expansion.md#p03) |
| A | 05 | `feat(env): export 배열과 출력 뷰 생성` | `src/env.c`: `env_get`, `env_set`, `env_unset`, `env_to_environ` | 환경 invariant와 snapshot | mutable 내부 상태와 exec 경계용 배열의 소유권을 설명하기 좋다. | 높음 | 중간 | 04, 06 | [P06](02-state-and-builtin-boundaries.md#p06) |
| B | 05 | `feat(builtin): pwd 작업 디렉터리 출력`<br>`feat(builtin): env 환경 목록 출력` | `src/builtin.c`: `builtin_pwd`, `builtin_env` | 단순 builtin 출력 | I/O 실패 전파를 제외하면 개별 구현은 반복적이고 면접 차별성이 낮다. | 중간 | 낮음 | 07, 09 | P16 설명에 일부 통합 |
| A | 05 | `feat(builtin): cd 이동과 PWD 상태 동기화` | `src/builtin.c`: `builtin_cd`, `argv_count` | cwd 상태 전이 | 프로세스 전역 상태와 환경 메타데이터의 비원자 경계를 설명할 수 있다. | 높음 | 중간 | 01, 07 | [P07](02-state-and-builtin-boundaries.md#p07) |
| A | 05 | `feat(builtin): export 대입과 선언 출력`<br>`feat(builtin): unset 환경 이름 제거` | `src/builtin.c`: `split_assignment`, `builtin_export`, `builtin_unset` | 부분 성공·이름 검증·환경 변경 | 여러 인자 처리 중 계속 가능한 오류와 중단해야 할 오류를 구분한다. | 높음 | 중간 | 09 | P06에 통합 |
| A | 05 | `feat(builtin): exit 상태를 셸 수명에 연결` | `src/builtin.c`: `parse_exit_status`, `builtin_exit` | 숫자 파싱·상태 machine | 검사 순서가 shell 수명과 결과를 바꾸는 고밀도 경계 문제다. | 높음 | 높음 | 01, 06 | [P08](02-state-and-builtin-boundaries.md#p08) |
| S | 05 | `feat(exec): 단일 명령을 자식에서 실행`<br>`feat(exec): 부모 builtin의 표준 스트림 복원` | `src/exec.c`, `src/redirection.c`<br>`builtin_is_parent`, `exec_run_parent_command` | 부모·자식 실행 경계 | fork 격리와 지속 상태를 함께 이해해야 하는 셸 핵심 설계다. | 높음 | 높음 | 06, 07 | [P05](02-state-and-builtin-boundaries.md#p05) |
| S | 06 | `feat(exec): 다단 pipeline의 pipe FD 연결` | `src/exec.c`: `run_forked_pipeline`, `run_child`, `close_pipes` | process·FD 그래프 | 직접 구현 가치가 높고 EOF·deadlock·FD 수명을 종합 평가한다. | 높음 | 높음 | 07, 09, 11 | [P09](03-process-io-and-resource-lifecycle.md#p09) |
| A | 06 | `test(status): 실행 불가 파일과 신호 종료 상태 검증` | `src/exec.c`: `status_from_wait`, exec 실패 경로 | wait·exec 상태 매핑 | POSIX wait status와 shell status의 경계를 정확히 설명해야 한다. | 높음 | 높음 | 05, 11 | [P10](03-process-io-and-resource-lifecycle.md#p10) |
| S | 06 | `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수`<br>`fix(exec): pipe 생성 실패 시 PID 배열 해제` | `terminate_children`, `wait_for_child`<br>`tests/faults.sh`, `tests/lifecycle.sh` | 부분 성공 cleanup | 정상 코드보다 실패 수명 관리 능력을 잘 드러낸다. | 높음 | 높음 | 09, 11 | P09에 통합 |
| S | 07 | `feat(redirection): 파일 입출력 리다이렉션 적용`<br>`test(exec): 다단 파이프와 리다이렉션 순서 검증` | `src/redirection.c`: `exec_apply_redirections` | source-order FD 변환 | 순서와 side effect가 관찰 가능한 I/O 의미를 만든다. | 높음 | 높음 | 06, 08 | [P11](03-process-io-and-resource-lifecycle.md#p11) |
| S | 07 | `fix(redirection): 부모 표준 입출력 복원 실패 전파`<br>`test(redirection): 저장·적용·복원 실패 회귀 검증` | `save_stdio`, `restore_one`, `restore_stdio`, `exec_run_parent_command` | 부모 stream transaction | 복원 실패가 shell invariant를 파괴하는 트랜잭션 문제다. | 높음 | 높음 | 05, 09 | P11에 통합 |
| A | 07 / 09 | `fix(io): builtin과 환경 출력 실패를 상태로 전파`<br>`test(io): read·write와 heredoc 입력 실패 검증` | `shell_write_all`, `shell_write_text`<br>`src/builtin.c`, `src/env.c` | partial I/O와 오류 전파 | POSIX I/O의 부분 성공을 command 상태까지 연결한다. | 높음 | 높음 | 08, 11 | [P16](04-failure-performance-and-verification.md#p16) |
| S | 08 | `feat(heredoc): 수집 본문 저장소 수명 관리`<br>`feat(heredoc): 구분자별 본문 순차 수집` | `src/heredoc.c`: `exec_prepare_heredocs`, `read_heredoc`<br>`struct heredoc_entry` | 사전 수집·identity·수명 | 입력 소비 순서와 실행 순서가 다른 고난도 상태 문제다. | 높음 | 높음 | 01, 02, 09 | [P12](03-process-io-and-resource-lifecycle.md#p12) |
| S | 08 | `test(heredoc): 인용 구분자와 본문 확장 검증`<br>`test(heredoc): 이중·부분 인용 구분자 회귀 검증` | `heredoc_quoted`, `append_heredoc_body_line`, `expand_heredoc_body_line` | quote semantics | lexer metadata가 실제 런타임 의미로 이어지는 대표 통합 지점이다. | 높음 | 중간 | 02, 04 | P12에 통합 |
| A | 08 / 09 | `refactor(runtime): heredoc 임시 파일 I/O 경계 분리`<br>`fix(heredoc): 임시 파일 저장 오류를 전파` | `src/redirection.c`: `heredoc_stream_error`<br>`shell_fflush`, `shell_fseek`, `shell_fileno` | 임시 저장 정합성 | buffered I/O의 숨은 실패와 데이터 절단을 다룬다. | 높음 | 중간 | 07 | [P13](03-process-io-and-resource-lifecycle.md#p13) |
| A | 08 | `fix(input): EOF와 입력 실패를 구분`<br>`fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구` | `src/input.c`, `src/heredoc.c`: `discard_heredoc`, `delimiter_matches` | 입력 정렬 복구 | 명령 단위 복구가 가능한지 판단하는 stream invariant다. | 높음 | 중간 | 09, 11 | P12와 P16에 통합 |
| S | 09 | `refactor(runtime): 프로세스 시스템 호출 경계 분리`<br>`refactor(runtime): FD 시스템 호출 경계 분리` | `src/runtime.c`, `src/runtime.h` | fault injection seam | 희귀 실패를 호출 위치별로 재현하고 실제 cleanup 경로를 검증한다. | 높음 | 높음 | 06, 07, 11 | [P14](04-failure-performance-and-verification.md#p14) |
| S | 09 | `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합`<br>`test(memory): 범위별 할당 실패 순회 검증` | allocation wrappers와 scope 설정<br>`tests/allocation.sh` | 할당 실패·명령 복구 | C 소유권과 부분 결과 rollback을 전체 단계에서 검증한다. | 높음 | 높음 | 03, 08 | P14와 P02에 통합 |
| S | 10 | `refactor(buffer): 가변 문자열 빌더 모듈 추가`<br>`refactor(lexer): 단어 조립을 가변 버퍼로 전환`<br>`refactor(expand): 확장 결과를 가변 버퍼로 조립` | `src/string_builder.c`, `.h`<br>`src/token.c`, `src/expand.c` | 상각 복잡도·overflow·ownership | 자료구조와 성능을 직접 구현하기 좋은 독립 문제다. | 높음 | 높음 | 02, 04 | [P15](04-failure-performance-and-verification.md#p15) |
| A | 10 / 11 | `test(performance): 긴 입력 처리 시간 상한 검증` | `tests/performance.sh`, `tests/timeout_runner.c` | 성능 회귀 oracle | 복잡도 개선을 524288바이트 입력과 시간 상한으로 검증한다. | 중간 | 낮음 | 09 | P15와 P17에 통합 |
| A | 11 | `build(test): 테스트 시간 제한 하네스 추가`<br>`test(lifecycle): FD와 자식 프로세스 누수 검증` | `tests/timeout_runner.c`, `tests/lifecycle.sh`<br>`shell_children_reaped` | 결정적 수명 검증 | hang·FD 누수·zombie를 관찰 가능한 oracle로 만든다. | 높음 | 중간 | 06, 09 | [P17](04-failure-performance-and-verification.md#p17) |
| A | 11 | `build(test): ASan·UBSan 검증 경로 추가`<br>`build: expose deterministic verification targets`<br>`ci: add cross-platform C validation` | `Makefile`, `tests/container_sanitizers.sh`, `.github/workflows/c-minishell-ci.yml` | 검증 계층·CI 경계 | 도구별 결함 범위와 portability trade-off를 설명하기 좋다. | 높음 | 낮음 | 09, 10 | P17에 통합 |

## 대표 Thread와 연관 Thread 관계

| 포인트 | 우선순위 | 대표 Thread | 명시적으로 통합한 Thread | 상세 문서 |
| --- | --- | --- | --- | --- |
| P01 | S | 02 | 08의 heredoc quote metadata, 10의 lexer buffer 전환 | [01](01-lexing-parsing-and-expansion.md#p01) |
| P02 | S | 03 | 06의 pipeline 문법, 09의 parser 할당 실패 rollback | [01](01-lexing-parsing-and-expansion.md#p02) |
| P03 | S | 04 | 01의 command loop 단계 순서, 08의 heredoc 선행 단계 | [01](01-lexing-parsing-and-expansion.md#p03) |
| P04 | A | 03 | 11의 parser API 회귀, hook 기반 test seam | [01](01-lexing-parsing-and-expansion.md#p04) |
| P05 | S | 05 | 06의 자식 실행, 07의 부모 stream wrapper | [02](02-state-and-builtin-boundaries.md#p05) |
| P06 | A | 05 | 04의 환경 확장, 06의 exec envp snapshot | [02](02-state-and-builtin-boundaries.md#p06) |
| P07 | A | 05 | 01의 shell state lifetime, 07의 출력·리다이렉션 | [02](02-state-and-builtin-boundaries.md#p07) |
| P08 | A | 05 | 01의 loop 종료, 06의 status convention | [02](02-state-and-builtin-boundaries.md#p08) |
| P09 | S | 06 | 07의 redirection ordering, 09의 syscall failure, 11의 lifecycle | [03](03-process-io-and-resource-lifecycle.md#p09) |
| P10 | A | 06 | 05의 `$?`·exit, 11의 signal 회귀 | [03](03-process-io-and-resource-lifecycle.md#p10) |
| P11 | S | 07 | 05의 parent builtin, 09의 dup·open failure | [03](03-process-io-and-resource-lifecycle.md#p11) |
| P12 | S | 08 | 01의 phase order, 02의 quote metadata, 09의 recovery | [03](03-process-io-and-resource-lifecycle.md#p12) |
| P13 | A | 08 | 07의 FD 적용, 09의 stdio fault seam | [03](03-process-io-and-resource-lifecycle.md#p13) |
| P14 | S | 09 | 03의 allocation rollback, 06·07·08 실패 경로, 11 harness | [04](04-failure-performance-and-verification.md#p14) |
| P15 | S | 10 | 02 lexer, 04 expander, 11 performance oracle | [04](04-failure-performance-and-verification.md#p15) |
| P16 | A | 09 | 07 출력 전파, 08 EOF·read 구분, 11 I/O 회귀 | [04](04-failure-performance-and-verification.md#p16) |
| P17 | A | 11 | 06 lifecycle, 09 fault sweep, 10 performance | [04](04-failure-performance-and-verification.md#p17) |

## 상세 워크북 문서

| 파일 | 역할 | 포함 포인트 |
| --- | --- | --- |
| [`01-lexing-parsing-and-expansion.md`](01-lexing-parsing-and-expansion.md) | quote-aware lexer, 명령 그래프 parser, 지연 확장, 공개 API 계약 | P01~P04 |
| [`02-state-and-builtin-boundaries.md`](02-state-and-builtin-boundaries.md) | 부모·자식 실행 경계, 환경, cwd, exit 상태 | P05~P08 |
| [`03-process-io-and-resource-lifecycle.md`](03-process-io-and-resource-lifecycle.md) | pipeline, wait status, 리다이렉션, heredoc, 임시 저장 | P09~P13 |
| [`04-failure-performance-and-verification.md`](04-failure-performance-and-verification.md) | fault injection, string builder, I/O 계약, 검증 pipeline | P14~P17 |

## S/A 완전성 대조

| 포인트 | 우선순위 | 상태 | 대조 결과 |
| --- | --- | --- | --- |
| P01 | S | 독립 상세 항목 | 작성됨: `01-lexing-parsing-and-expansion.md` |
| P02 | S | 독립 상세 항목 | 작성됨: `01-lexing-parsing-and-expansion.md` |
| P03 | S | 독립 상세 항목 | 작성됨: `01-lexing-parsing-and-expansion.md` |
| P04 | A | 독립 상세 항목 | 작성됨: `01-lexing-parsing-and-expansion.md` |
| P05 | S | 독립 상세 항목 | 작성됨: `02-state-and-builtin-boundaries.md` |
| P06 | A | 독립 상세 항목 | 작성됨: `02-state-and-builtin-boundaries.md` |
| P07 | A | 독립 상세 항목 | 작성됨: `02-state-and-builtin-boundaries.md` |
| P08 | A | 독립 상세 항목 | 작성됨: `02-state-and-builtin-boundaries.md` |
| P09 | S | 독립 상세 항목 | 작성됨: `03-process-io-and-resource-lifecycle.md` |
| P10 | A | 독립 상세 항목 | 작성됨: `03-process-io-and-resource-lifecycle.md` |
| P11 | S | 독립 상세 항목 | 작성됨: `03-process-io-and-resource-lifecycle.md` |
| P12 | S | 독립 상세 항목 | 작성됨: `03-process-io-and-resource-lifecycle.md` |
| P13 | A | 독립 상세 항목 | 작성됨: `03-process-io-and-resource-lifecycle.md` |
| P14 | S | 독립 상세 항목 | 작성됨: `04-failure-performance-and-verification.md` |
| P15 | S | 독립 상세 항목 | 작성됨: `04-failure-performance-and-verification.md` |
| P16 | A | 독립 상세 항목 | 작성됨: `04-failure-performance-and-verification.md` |
| P17 | A | 독립 상세 항목 | 작성됨: `04-failure-performance-and-verification.md` |

## 백지 구현 우선순위

1. [P09] 다단 pipeline의 프로세스·FD 수명
2. [P02] 명령 그래프 파싱과 실패 원자성
3. [P11] source-order 리다이렉션과 부모 stdio 트랜잭션
4. [P15] 상각 O(n) 문자열 builder
5. [P01] 인용 의미 보존 tokenizer
6. [P12] heredoc 사전 수집과 입력 경계 복구
7. [P14] call-index·scope 기반 failure injection
8. [P03] 조건 gate와 지연 확장 scheduler
9. [P05] parent·child builtin route
10. [P16] partial write·EINTR 처리
11. [P08] exit 숫자 파싱과 상태 전이
12. [P10] wait·exec 상태 매핑
13. [P06] 환경 update와 envp snapshot
14. [P13] heredoc 임시 저장
15. [P07] cd 상태 전이
16. [P04] 공개 parser wrapper
17. [P17] 결정적 테스트 harness

## 설명 연습 우선순위

1. [P05] 왜 상태 변경 builtin은 standalone일 때 부모에서 실행해야 하는가
2. [P09] 각 process의 pipe FD 소유권과 EOF 조건
3. [P12] heredoc 입력 소비 순서가 조건 실행보다 앞서는 이유
4. [P03] parse-time과 run-time 확장의 경계
5. [P11] 부모 stdio 복원 실패를 fatal invariant 손상으로 보는 이유
6. [P14] 복구 가능한 command 실패와 shell 중단 경계
7. [P02] parser 부분 결과의 ownership과 중앙 rollback
8. [P15] 반복 join의 O(n²)와 기하급수 성장의 상각 분석
9. [P16] EOF·read error·partial write 계약
10. [P06] mutable 환경과 exec용 snapshot의 소유권
11. [P10] wait status, 126·127, 128+signal
12. [P17] fault·lifecycle·sanitizer·CI가 잡는 결함 범위
13. [P07] chdir 이후 메타데이터 갱신의 비원자성
14. [P01] quote metadata를 lexer에서 보존하는 이유
15. [P08] exit 인자 검사 순서
16. [P13] tmpfile의 flush·seek·dup2 정합성
17. [P04] 오류 출력 포인터와 제어 흐름 반환의 분리

## 한 문제로 통합한 Thread 묶음

- **P01**: Thread 02 lexer + Thread 08 heredoc 인용 플래그 + Thread 10 lexer buffer 전환
- **P02**: Thread 03 parser·ownership + Thread 06 pipeline 문법 + Thread 09 parser 할당 실패 rollback
- **P03**: Thread 04 확장·조건 실행 + Thread 01 command loop 단계 순서 + Thread 08 heredoc 선행 단계
- **P04**: Thread 03 공개 parser API·hook seam + Thread 11 parser API 회귀
- **P05**: Thread 05 builtin 경계 + Thread 06 child 실행 + Thread 07 parent stream wrapper
- **P06**: Thread 05 환경 자료구조·export·unset + Thread 04 환경 확장 + Thread 06 exec snapshot
- **P07**: Thread 05 cd + Thread 01 상태 수명 + Thread 07 출력 실패
- **P08**: Thread 05 exit + Thread 01 loop 종료 + Thread 06 상태 관례
- **P09**: Thread 06 pipeline·부분 실패 + Thread 07 redirection precedence + Thread 09 syscall seam + Thread 11 lifecycle
- **P10**: Thread 06 wait·exec status + Thread 05 `$?`·exit + Thread 11 signal 회귀
- **P11**: Thread 07 리다이렉션·복원 + Thread 05 parent builtin + Thread 09 dup·open 실패
- **P12**: Thread 08 heredoc 수집·복구 + Thread 02 quote metadata + Thread 01 phase order + Thread 09 command recovery
- **P13**: Thread 08 임시 파일 저장 + Thread 07 FD 적용 + Thread 09 stdio failure seam
- **P14**: Thread 09 fault injection + Thread 03 allocation rollback + Thread 06·07·08 실패 cleanup + Thread 11 harness
- **P15**: Thread 10 builder·performance + Thread 02 lexer + Thread 04 expander + Thread 11 performance test
- **P16**: Thread 09 read·write failure + Thread 08 EOF 구분 + Thread 07 output propagation + Thread 11 I/O 회귀
- **P17**: Thread 11 검증 pipeline + Thread 06 lifecycle + Thread 09 fault sweep + Thread 10 성능 oracle
