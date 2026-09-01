# DevThread 개발자 기술면접 워크북 — 마스터 인덱스

## 사용 범위와 선별 기준

이 문서는 현재 GPT 프로젝트 메모리에서 확인된 29개 DevThread 학습 문서와 그 안의 커밋 기록만을 기준으로 작성했다. 원격 저장소의 현재 상태, 프로젝트 메모리에 나타나지 않은 파일·함수·구현은 전제하지 않는다.

우선순위는 Thread 전체의 난이도가 아니라 **그 Thread에서 면접 준비 항목으로 남길 대표 지점의 가치**를 뜻한다.

- **S**: 질문과 10~30분 직접 구현 모두 반드시 준비할 가치가 높다.
- **A**: 질문 또는 핵심 구현으로 준비 가치가 높다.
- **B**: 별도 구현 문제보다 설계·개념 설명으로 준비하는 편이 낫다.
- **C**: 반복 구현, 연결 작업, 단독으로는 변별력이 낮은 검증이므로 별도 항목을 만들지 않는다.

동일한 역량이 여러 Thread에 반복되면 대표 문제 하나로 통합했다. `질문형`과 `구현형`은 실제 면접에서 다뤄질 가능성과 변별력을 상대적으로 표시한 것이다.

## 전체 Thread·커밋 선별 결과

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---:|---|---|---|---|---|---|---|---|
| A | 1 | `feat(memory): 겹치는 메모리의 안전한 이동 구현` | `src/memory/ft_memory_move.c` / `ft_memmove`, `tests/test_memory_move.c` | 겹치는 바이트 범위와 복사 방향 | 포인터 범위, 겹침, 역방향 복사, 0길이 계약을 짧은 코드에서 함께 확인할 수 있다. | 높음 | 높음 | 2, 4, 6 | [C-01](01-c-memory-ownership-and-io.md#c-01) |
| A | 2 | `feat(string): 문자열 길이 계산과 제한 복사·붙이기 추가` | `src/string/ft_string_bounds.c` / `ft_strlcpy`, `ft_strlcat`; `tests/test_string_bounds.c` | capacity와 반환값 계약 | 쓰지 못한 경우에도 계산해야 하는 길이, capacity 0, 종단 문자가 없는 목적지 등 API 계약을 검증하기 좋다. | 높음 | 높음 | 1, 10 | [C-02](01-c-memory-ownership-and-io.md#c-02) |
| B | 3 | `feat(convert): 표현 가능한 10진수 정수 해석` / `feat(convert): 부호 있는 정수의 문자열 변환 구현` | `src/convert/ft_atoi.c` / `ft_atoi`; `src/convert/ft_itoa.c` / `ft_itoa` | `INT_MIN`, 오버플로 사전 판정 | 핵심 경계는 유효하지만 Thread 25의 입력 파서와 rank 전처리 문제가 더 넓고 대표성이 높다. | 높음 | 중간 | 25 | — |
| S | 4 | `test(alloc): 할당 실패와 rollback을 검증` | `src/alloc/ft_allocate.c` / `ft_calloc`; `src/string/ft_split.c` / `ft_split`, `free_fields`; `tests/failure/test_failure.c` | 부분 성공 뒤 소유권 회수 | N번째 할당 실패에서도 누수·이중 해제 없이 원자적으로 실패하는지 확인할 수 있어 C 자원 관리 면접 가치가 높다. | 높음 | 높음 | 5, 15, 28 | [C-03](01-c-memory-ownership-and-io.md#c-03) |
| A | 5 | `feat(list): 실패 시 정리되는 리스트 변환 구현` | `src/list/ft_list_map.c` / `ft_lstmap`; `src/list/ft_list_lifecycle.c` / `ft_lstclear` | 콜백 결과의 소유권과 실패 정리 | 새 노드와 변환된 content의 소유권, 중간 실패 정리를 설명하기 좋다. Thread 4의 rollback 문제에 통합한다. | 높음 | 중간 | 4 | [C-03에 통합](01-c-memory-ownership-and-io.md#c-03) |
| S | 6 | `fix(io): 파일 디스크립터 출력을 끝까지 재시도` | `src/io/ft_fd_output.c` / `write_all`; `tests/failure/test_fd_output_failure.c` | partial write, `EINTR`, 0바이트 쓰기 | 시스템 호출 한 번이 요청 전체를 처리한다는 잘못된 가정을 제거하는 대표적인 저수준 I/O 문제다. | 높음 | 높음 | 12, 22, 28 | [C-04](01-c-memory-ownership-and-io.md#c-04) |
| B | 7 | `test(release): archive와 consumer 경계를 검증` | `tests/check_archive.sh`; `tests/api-symbols.txt`, `tests/archive-members.txt`, `tests/allowed-undefined.txt`; `Makefile` | 정적 라이브러리 공개 경계와 재현 가능한 검증 | 빌드·릴리스 경계 설명에는 유용하지만 직접 구현 변별력은 낮아 별도 워크북 문제로 만들지 않는다. | 중간 | 낮음 | 6, 13 | — |
| S | 8 | `fix(format): 지원 문법과 전체 출력 크기 선검증` | `src/ft_printf.c` / `ft_printf`; `src/ft_measure.c` / `ft_printf_measure` | 측정 후 렌더링하는 2단계 파이프라인 | 잘못된 포맷이나 반환 길이 오버플로가 부분 출력으로 남지 않게 하는 설계 판단과 `va_list` 복제 문제를 함께 묻기 좋다. | 높음 | 높음 | 9, 12 | [F-02](02-formatting-pipeline.md#f-02) |
| S | 9 | `feat(parser): 포맷 필드 모델과 해석기 추가` / `fix(format): 지원 문법과 전체 출력 크기 선검증` | `src/ft_parse.c` / `ft_printf_parse`, `ft_parse_decimal`; `src/ft_measure.c` / `ft_is_supported_specifier` | 포맷 문법 파싱과 선검증 | 플래그·너비·정밀도 상태, 숫자 오버플로, 지원하지 않는 지정자, trailing `%`를 명확한 계약으로 만들 수 있다. | 높음 | 높음 | 8, 10, 11 | [F-01](02-formatting-pipeline.md#f-01), [F-02](02-formatting-pipeline.md#f-02) |
| A | 10 | `fix(text): 문자열 정밀도 범위까지만 읽기` | `src/ft_text.c` / `ft_local_strlen`, `ft_printf_print_string`; `tests/test_ft_printf.c` | 출력 한계와 메모리 읽기 한계의 일치 | 결과를 나중에 자르는 것과 애초에 범위 밖을 읽지 않는 것은 다르다는 점을 작은 문제로 검증한다. | 높음 | 높음 | 2, 9 | [F-03](02-formatting-pipeline.md#f-03) |
| A | 11 | `refactor(output): 숫자 출력 배치 로직 통합` | `src/ft_numeric_layout.c` / `ft_printf_write_numeric_layout`; `src/ft_number.c`; `src/ft_hex.c` | 접두사·정밀도 0·필드 패딩 배치 | 여러 플래그의 우선순위를 순서 불변식으로 정리해야 하므로 조건문 나열보다 설계력을 확인하기 좋다. | 높음 | 높음 | 9, 10, 12 | [F-04](02-formatting-pipeline.md#f-04) |
| S | 12 | `fix(output): 쓰기 결과를 집계하기 전에 범위 검증` / `test(output): 쓰기 실패 시퀀스와 채움 전략 검증` | `src/ft_output.c` / `ft_printf_write`, `ft_printf_putnchar`; `src/ft_printf.c` | sticky error와 출력 길이 경계 | partial write·`EINTR`·`INT_MAX` 집계·후속 출력 차단을 하나의 출력 컨텍스트 계약으로 설명할 수 있다. | 높음 | 높음 | 6, 8, 22 | [C-05](01-c-memory-ownership-and-io.md#c-05) |
| S | 13 | `feat(reader): 명시적 결과 상태 API 추가` | `get_next_line.c` / `blr_reader_next`, `find_line_end`; `get_next_line.h` / `t_blr_result` | 줄 추출 상태 머신과 결과 의미 | LINE·EOF·ERROR를 포인터 하나로 섞지 않고 버퍼 커서 invariant와 함께 설계하는 핵심 스트리밍 문제다. | 높음 | 높음 | 14, 15, 16 | [R-01](03-streaming-reader-state-and-recovery.md#r-01) |
| A | 14 | `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용` | `get_next_line.c` / `reserve_bytes`, `find_line_end`; `tests/metrics/metric_runtime.c`; `tests/manifests/metrics-4mib.txt` | 기하 증가, 스캔 커서, 복사 비용 | 기능 정답만이 아니라 읽기·할당·복사 횟수를 결정적으로 측정해 선형 비용을 검증한 점이 일반화된다. | 높음 | 중간 | 13, 15 | [R-02](03-streaming-reader-state-and-recovery.md#r-02) |
| S | 15 | `feat(context): 명시적 reader 수명 API 추가` | `get_next_line.h` / `blr_reader_create`, `blr_reader_reset`, `blr_reader_destroy`; `get_next_line.c` | 숨은 FD 상태 제거와 명시적 lifecycle | FD 소유권, reset 의미, 번호 재사용, `dup` alias, 독립 컨텍스트 병렬 사용을 API 계약으로 분리한다. | 높음 | 높음 | 13, 14, 16 | [R-03](03-streaming-reader-state-and-recovery.md#r-03) |
| S | 16 | `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존` | `get_next_line.c` / `read_retrying`, `blr_reader_next`; `get_next_line.h` / `BLR_AGAIN`; `tests/test_nonblocking.c` | `EINTR`, `EAGAIN`, 오류 뒤 상태 보존 | 비차단 입력에서 "아직 없음"과 EOF·영구 오류를 분리하고 이미 읽은 바이트를 잃지 않는지 묻는 고가치 문제다. | 높음 | 높음 | 13, 15 | [R-04](03-streaming-reader-state-and-recovery.md#r-04) |
| S | 17 | `feat(server): 획득 요청을 검증해 세션 소유권 예약` | `include/minitalk.h` / `t_mt_request`, `t_mt_response`; `src/server.c`; `src/client.c` | signal 데이터 채널과 datagram 제어 채널 분리 | signal의 작은 payload와 async 제약을 보완하기 위해 세션·응답을 별도 채널로 둔 경계 설계가 면접 가치가 높다. | 높음 | 중간 | 18, 20, 21 | [P-01](04-signal-datagram-protocol.md#p-01) |
| A | 18 | `feat(protocol): NUL 바이트로 메시지 종료 표시` | `src/client.c` / `send_bit`, `send_byte`; `src/server.c` / `flush_byte`; `tests/smoke.sh` | MSB-first 비트 조립과 프레이밍 | 바이트 순서, 8비트 상태, 빈 메시지, 프레임 종료를 확인할 수 있으나 채널 선택까지 포함한 Thread 17 문제에 통합한다. | 높음 | 높음 | 17 | [P-01에 통합](04-signal-datagram-protocol.md#p-01) |
| S | 19 | `feat(runtime): 안전한 응답 endpoint 경로 생성` / `feat(client): datagram 응답 endpoint 수명주기 관리` | `src/response_channel.c` / `mt_runtime_dir`, `mt_response_path`; `src/client.c` / `remove_stale_socket`, `bind_client_socket` | Unix socket 경로 검증과 수명 정리 | 파일시스템 namespace를 쓰는 IPC에서 권한·소유자·파일 종류·stale 경로·정리 순서를 모두 다룬다. | 높음 | 높음 | 20, 21 | [P-02](04-signal-datagram-protocol.md#p-02) |
| S | 20 | `feat(server): 획득 요청을 검증해 세션 소유권 예약` | `src/server.c` / `valid_request_source`, `read_session_request`, `handle_session_request`; `src/client.c` / `read_response`, `wait_for_response`; `tests/response_validation.sh` | 세션 소유권, 응답 상관관계, 죽은 소유자 회수 | 위조·지연·순서가 바뀐 응답을 무시하고 nonce·sequence·source를 검증하는 실제 프로토콜 질문으로 확장된다. | 높음 | 높음 | 17, 19, 22 | [P-03](04-signal-datagram-protocol.md#p-03) |
| S | 21 | `refactor(server): signal 처리를 self-pipe event loop로 제한` / `test(server): self-pipe 이벤트 손실 시 fail-stop 검증` | `src/server.c` / `t_bit_event`, `handle_bit`, `read_event`, `process_bit`, `run_event_loop` | async-signal-safe 경계와 self-pipe | 핸들러를 고정 크기 이벤트 기록으로 제한하고 실제 상태 전이를 일반 실행 문맥으로 옮기는 대표 동시성·OS 문제다. | 높음 | 높음 | 17, 20, 22 | [P-04](04-signal-datagram-protocol.md#p-04) |
| S | 22 | `fix(server): stdout 실패 뒤 ACK 전송 차단` / `test(server): 회수 줄바꿈 출력 실패 검증` | `src/write_utils.c` / `mt_write_all`; `src/server.c` / `flush_byte`, `process_bit`, `send_response`; `tests/output_failure.sh` | 출력 완료와 ACK의 커밋 경계 | 외부 효과가 완료되기 전에 성공 응답을 보내면 생기는 정합성 오류를 작은 분산 트랜잭션으로 설명할 수 있다. | 높음 | 높음 | 6, 12, 20 | [P-05](04-signal-datagram-protocol.md#p-05) |
| A | 23 | `test(sort): 생성 명령의 정렬 결과를 독립 검증` | `src/operations.c`; `src/push_swap.c`; `src/checker.c`; `tests/run_tests.py` | 생성기–checker 공유 모델과 독립 oracle | 실행 모델을 공유해 의미 불일치를 줄이면서도 Python 재생기로 상관 오류를 막는 테스트 경계가 좋다. | 높음 | 중간 | 24, 26, 27 | [S-01](05-stack-model-checker-and-sorting.md#s-01) |
| S | 24 | `test(operation): 정확한 상태 전이와 no-op을 검증` | `include/push_swap.h` / `t_stack`; `src/stack.c`; `src/operations.c`; `tests/operation_invariants.c` | 배열 스택 연산 invariant | `values`와 `ranks`의 쌍 보존, 크기 상한, 원소 보존, 부족한 입력의 no-op을 직접 구현으로 확인하기 좋다. | 높음 | 높음 | 23, 25, 26 | [S-02](05-stack-model-checker-and-sorting.md#s-02) |
| S | 25 | `feat(parse): 중복 입력을 거절하고 상대 순위를 계산` / `fix(parse): 토큰 수와 배열 크기 계산을 방어` | `src/parser.c` / `parse_token`, `count_all_tokens`, `assign_ranks`, `find_rank`, `parse_input` | 입력 정규화와 rank compression | 정수 경계, 토큰화, 중복, 크기 산술, 정렬된 복사본과 이진 탐색을 한 문제에서 확인한다. | 높음 | 높음 | 3, 24, 26 | [S-03](05-stack-model-checker-and-sorting.md#s-03) |
| S | 26 | `feat(sort): 네다섯 개의 스택을 정렬` / `feat(sort): 큰 입력을 기수 정렬로 처리` | `src/sort.c` / `sort_tiny`, `count_bits`, `radix_sort`, `sort_stack`; `tests/run_tests.py` | 입력 크기 적응 전략과 LSD radix | 작은 입력은 전용 전략, 큰 입력은 연속 rank 기반 기수 정렬로 나누는 선택과 복잡도를 설명·구현할 수 있다. | 높음 | 높음 | 24, 25, 29 | [S-04](05-stack-model-checker-and-sorting.md#s-04) |
| A | 27 | `fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도` / `test(checker): 읽기 실패와 명령 경계를 검증` | `src/checker_reader.c` / `read_next_line`; `src/checker.c` / `apply_checker_command`, `read_and_apply`; `include/push_swap.h` / `PS_COMMAND_MAX` | bounded framing, 재생, verdict | 짧은 명령 문법에 무제한 동적 버퍼를 쓰지 않고 NUL·과장 길이·EOF 프레임·`EINTR`를 구분한다. | 높음 | 높음 | 23, 24, 28 | [S-05](05-stack-model-checker-and-sorting.md#s-05) |
| A | 28 | `refactor(runtime): 메모리와 입력 시스템 호출을 공통화` / `test(memory): 할당 실패 뒤 자원 정리를 검증` | `src/runtime.c` / `ps_malloc`, `ps_free`, `ps_read`, `ps_write_all`, `ps_ignore_sigpipe`, `ps_test_finish`; `tests/fault_tests.py`; `Makefile` | fault injection과 실패 전파 | N번째 할당, 읽기·쓰기 오류, short write, 닫힌 pipe를 결정적으로 주입해 실패 경로를 제품 코드 밖에서 검증한다. | 높음 | 중간 | 4, 6, 27 | [S-06](05-stack-model-checker-and-sorting.md#s-06) |
| A | 29 | `test(sort): 큰 입력의 명령 수 상한을 검증` / `test(resource): 명령과 배열 이동 및 할당량을 기준화` | `tests/run_tests.py` / `test_move_counts`, `deterministic_values`; `tests/resource_tests.py`; `tests/resource_baseline.json` | 알고리즘 비용의 결정적 회귀 기준 | 벽시계 대신 명령 수·배열 이동·peak bytes를 측정해 성능 퇴행을 재현 가능하게 만든다. Thread 26 문제에 통합한다. | 높음 | 중간 | 26, 28 | [S-04에 통합](05-stack-model-checker-and-sorting.md#s-04) |
| C | 1 | `feat(memory): 메모리 채우기와 0 초기화 구현` | `src/memory/ft_memory_fill.c` / `ft_memset`, `ft_bzero` | 단순 바이트 반복 | 기본기는 필요하지만 단독 문제로는 분기·상태·실패 경계의 변별력이 낮다. | 낮음 | 낮음 | 1의 A 항목 | — |
| C | 8 | `feat(text): 문자·문자열·퍼센트 변환 추가` | `src/ft_dispatch.c`; `src/ft_text.c` | 변환기 연결 boilerplate | 핵심 설계는 파서·측정·배치·출력 복구에 있고 단순 dispatch는 별도 준비 가치가 낮다. | 낮음 | 낮음 | 9, 10, 11 | — |
| C | 18 | `test(smoke): 프로세스 간 메시지 전달 검증` | `tests/smoke.sh` | 정상 경로 smoke test | 필수 회귀 검증이지만 면접 변별력은 실패·경쟁·상관관계 테스트보다 낮다. | 낮음 | 낮음 | 17, 20, 21 | — |
| C | 23 | `feat(push_swap): 정렬 명령 생성 흐름을 연결` | `src/push_swap.c` | 상위 orchestration 연결 | 파싱·모델·정렬 전략·I/O 실패 전파를 호출하는 연결 코드 자체는 별도 문제로 만들 필요가 낮다. | 낮음 | 낮음 | 24, 25, 26, 28 | — |

## 대표 면접 포인트와 상세 문서

| ID | 대표 주제 | 포함 Thread | 문서 |
|---|---|---|---|
| C-01 | 겹치는 메모리 이동 | 1 | [01-c-memory-ownership-and-io.md](01-c-memory-ownership-and-io.md#c-01) |
| C-02 | capacity 제한 문자열 복사·붙이기 | 2 | [01-c-memory-ownership-and-io.md](01-c-memory-ownership-and-io.md#c-02) |
| C-03 | 할당 실패 rollback과 소유권 | 4, 5 | [01-c-memory-ownership-and-io.md](01-c-memory-ownership-and-io.md#c-03) |
| C-04 | 완전 쓰기 루프 | 6 | [01-c-memory-ownership-and-io.md](01-c-memory-ownership-and-io.md#c-04) |
| C-05 | 출력 컨텍스트와 sticky error | 12 | [01-c-memory-ownership-and-io.md](01-c-memory-ownership-and-io.md#c-05) |
| F-01 | 포맷 문법 파서 | 9 | [02-formatting-pipeline.md](02-formatting-pipeline.md#f-01) |
| F-02 | 2단계 측정·렌더링 | 8, 9 | [02-formatting-pipeline.md](02-formatting-pipeline.md#f-02) |
| F-03 | 정밀도 제한 문자열 읽기 | 10 | [02-formatting-pipeline.md](02-formatting-pipeline.md#f-03) |
| F-04 | 숫자 접두사와 패딩 배치 | 11 | [02-formatting-pipeline.md](02-formatting-pipeline.md#f-04) |
| R-01 | 줄 추출 상태 머신 | 13 | [03-streaming-reader-state-and-recovery.md](03-streaming-reader-state-and-recovery.md#r-01) |
| R-02 | 구간 버퍼와 선형 비용 | 14 | [03-streaming-reader-state-and-recovery.md](03-streaming-reader-state-and-recovery.md#r-02) |
| R-03 | 명시적 reader lifecycle | 15 | [03-streaming-reader-state-and-recovery.md](03-streaming-reader-state-and-recovery.md#r-03) |
| R-04 | `EINTR`·`EAGAIN` 뒤 상태 보존 | 16 | [03-streaming-reader-state-and-recovery.md](03-streaming-reader-state-and-recovery.md#r-04) |
| P-01 | signal 데이터와 datagram 제어 채널 | 17, 18 | [04-signal-datagram-protocol.md](04-signal-datagram-protocol.md#p-01) |
| P-02 | 안전한 Unix datagram endpoint | 19 | [04-signal-datagram-protocol.md](04-signal-datagram-protocol.md#p-02) |
| P-03 | 세션 예약·응답 상관관계 | 20 | [04-signal-datagram-protocol.md](04-signal-datagram-protocol.md#p-03) |
| P-04 | self-pipe 시그널 안전 경계 | 21 | [04-signal-datagram-protocol.md](04-signal-datagram-protocol.md#p-04) |
| P-05 | 출력 완료와 ACK 경계 | 22 | [04-signal-datagram-protocol.md](04-signal-datagram-protocol.md#p-05) |
| S-01 | 생성기–checker 공유 모델과 독립 oracle | 23 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-01) |
| S-02 | 배열 스택 연산 invariant | 24 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-02) |
| S-03 | 입력 정규화와 rank compression | 25 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-03) |
| S-04 | 크기 적응 정렬·LSD radix·비용 기준 | 26, 29 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-04) |
| S-05 | checker 명령 프레이밍과 재생 | 27 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-05) |
| S-06 | fault injection과 실패 전파 | 28 | [05-stack-model-checker-and-sorting.md](05-stack-model-checker-and-sorting.md#s-06) |

## S/A 완전성 검증

아래 표는 위 선별표의 모든 S/A Thread가 상세 문서에서 독립 항목 또는 명시적 통합 항목으로 처리되었는지 대조한 결과다.

| Thread | 우선순위 | 상태 | 상세 항목 |
|---:|---|---|---|
| 1 | A | 독립 항목 | C-01 |
| 2 | A | 독립 항목 | C-02 |
| 4 | S | 대표 항목 | C-03 |
| 5 | A | Thread 4에 명시적 통합 | C-03 |
| 6 | S | 독립 항목 | C-04 |
| 8 | S | Thread 9와 명시적 통합 | F-02 |
| 9 | S | 독립 항목 및 Thread 8과 통합 | F-01, F-02 |
| 10 | A | 독립 항목 | F-03 |
| 11 | A | 독립 항목 | F-04 |
| 12 | S | 독립 항목 | C-05 |
| 13 | S | 독립 항목 | R-01 |
| 14 | A | 독립 항목 | R-02 |
| 15 | S | 독립 항목 | R-03 |
| 16 | S | 독립 항목 | R-04 |
| 17 | S | 대표 항목 | P-01 |
| 18 | A | Thread 17에 명시적 통합 | P-01 |
| 19 | S | 독립 항목 | P-02 |
| 20 | S | 독립 항목 | P-03 |
| 21 | S | 독립 항목 | P-04 |
| 22 | S | 독립 항목 | P-05 |
| 23 | A | 독립 항목 | S-01 |
| 24 | S | 독립 항목 | S-02 |
| 25 | S | 독립 항목 | S-03 |
| 26 | S | 대표 항목 | S-04 |
| 27 | A | 독립 항목 | S-05 |
| 28 | A | 독립 항목 | S-06 |
| 29 | A | Thread 26에 명시적 통합 | S-04 |

**대조 결과:** S/A로 선별한 27개 Thread는 모두 상세 워크북에 독립적으로 작성되었거나 대표 항목에 명시적으로 통합되었다. 미배정 S/A 항목은 없다.

## 백지 구현 우선순위

### 1순위 — 반드시 손으로 다시 구현

1. **R-04** `EINTR`·`EAGAIN`·I/O 오류 뒤 스트리밍 상태 보존
2. **P-04** async-signal-safe 핸들러와 self-pipe 이벤트 경계
3. **P-03** 세션 예약과 응답 상관관계 검증
4. **C-03** 부분 할당 실패 rollback과 소유권 정리
5. **C-04** partial write·`EINTR`를 처리하는 완전 쓰기 루프
6. **S-02** 배열 스택 연산과 invariant
7. **S-03** 경계 안전 정수 파싱과 rank compression
8. **S-04** rank 기반 LSD radix 한 라운드와 전체 정렬
9. **F-01** 포맷 필드 파서와 오버플로 선검증
10. **R-01** 줄 추출 상태 머신

### 2순위 — 대표 구현을 한 번 이상 수행

- **C-01**, **C-02**, **C-05**
- **F-02**, **F-03**, **F-04**
- **R-02**, **R-03**
- **P-01**, **P-02**, **P-05**
- **S-01**, **S-05**, **S-06**

## 설명 연습 우선순위

1. **P-04** 왜 signal handler에서 상태 전이·소켓 I/O·출력을 하지 않았는가
2. **P-03** source·magic·PID·kind·token·deadline을 함께 검증해야 하는 이유
3. **R-03** reader가 FD를 빌릴 뿐 닫지 않는 계약, reset과 destroy의 차이
4. **R-04** EOF·AGAIN·ERROR를 분리하고 오류 뒤 입력을 보존하는 이유
5. **F-02** `va_copy`를 사용한 측정/렌더링 2회 순회와 부분 출력 방지 trade-off
6. **P-05** 출력 완료가 ACK보다 먼저여야 하는 커밋 경계
7. **C-03** 다단계 할당에서 각 시점의 소유자와 rollback 순서
8. **R-02** 기하 증가와 단조 scan cursor가 O(n²) 재스캔을 막는 방식
9. **S-01** 공유 연산 모델의 장점과 독립 oracle이 필요한 이유
10. **S-04** 작은 입력 전용 전략과 큰 입력 LSD radix를 나눈 이유
11. **P-02** Unix socket 경로가 보안·수명 관리 대상인 이유
12. **C-05** sticky error가 후속 쓰기와 반환 길이 의미를 단순화하는 방식
13. **F-04** 부호·접두사·정밀도 0·필드 패딩의 출력 순서 invariant
14. **S-06** fault injection을 제품 코드와 격리하면서도 실패 경로를 결정적으로 만드는 방법

## 한 문제로 통합한 Thread 묶음

- **Thread 4 + 5 → C-03**: 문자열/배열과 연결 리스트는 자료구조가 다르지만, 핵심 역량은 "부분 성공 시점의 소유권과 실패 rollback"으로 같다.
- **Thread 8 + 9 → F-02**: 포맷 파싱 결과를 먼저 측정하고 그 뒤 렌더링하는 전체 파이프라인 문제로 통합했다. Thread 9의 파서 자체는 F-01로 별도 유지했다.
- **Thread 17 + 18 → P-01**: signal 비트 인코딩·NUL 프레이밍을 데이터 채널과 datagram 제어 채널의 역할 분리 문제에 포함했다.
- **Thread 26 + 29 → S-04**: 정렬 알고리즘 선택과 명령 수·배열 이동·할당량 회귀 기준을 같은 성능 문제로 묶었다.
- **Thread 3 → S-03의 연관 위치**: 정수 경계 처리 역량은 입력 토큰화·중복 검증·rank compression을 포함하는 Thread 25 문제가 대표한다. Thread 3은 B로 남겨 별도 상세 문제를 만들지 않았다.
- **Thread 6, 12, 22, 28은 통합하지 않고 계층별로 분리**했다. 같은 쓰기 실패를 다루지만 각각 시스템 호출 완결성(C-04), 출력 컨텍스트 상태(C-05), 프로토콜 커밋 경계(P-05), 장애 주입 검증(S-06)을 묻는다.
