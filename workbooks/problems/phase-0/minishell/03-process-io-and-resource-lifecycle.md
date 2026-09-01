# 프로세스, FD, 리다이렉션, heredoc 수명

이 문서는 POSIX process와 file descriptor의 소유권을 직접 설명하고 구현하는 문제를 모은다. 정상 경로보다 부분 생성·복원 실패·입력 경계 훼손에서 어떤 자원을 누가 회수하는지가 핵심이다.

## 문서 내 면접 포인트

- [P09. 다단 pipeline의 프로세스·FD 수명](#p09)
- [P10. wait status와 실행 오류의 셸 상태 매핑](#p10)
- [P11. source-order 리다이렉션과 부모 stdio 트랜잭션](#p11)
- [P12. heredoc 사전 수집, 인용 의미와 입력 경계 복구](#p12)
- [P13. heredoc 본문의 임시 저장과 I/O 오류 전파](#p13)

---

<a id="p09"></a>
## P09. [Thread 06 / `feat(exec): 다단 pipeline의 pipe FD 연결`] 다단 pipeline의 프로세스·FD 수명

### 면접 질문

- 명령 N개를 연결할 때 pipe가 왜 N-1개 필요하며, i번째 자식은 어떤 끝을 stdin/stdout에 연결합니까?
- 각 자식이 `dup2` 뒤 사용하지 않는 모든 pipe FD를 닫아야 하는 이유는 무엇입니까?
- 부모가 pipe 끝을 오래 들고 있으면 어떤 명령이 EOF를 받지 못합니까?
- 두 번째 fork가 실패한 부분 생성 pipeline에서 이미 생성한 자식과 FD를 어떻게 회수합니까?
- 꼬리 질문: pipeline 연결 후 command redirection을 적용하는 순서가 결과에 어떤 영향을 줍니까?

### 30초 모범 답변

N개 명령에는 인접 단계마다 하나씩 N-1개의 pipe가 필요합니다. 자식 i는 이전 pipe의 read end를 stdin에, 다음 pipe의 write end를 stdout에 복제한 뒤 상속받은 모든 원본 pipe FD를 닫습니다. 부모도 spawn이 끝나면 모든 pipe 끝을 닫아야 reader가 writer 종료를 정확히 관찰합니다. 중간 fork 실패 시 이미 만든 자식을 종료·wait하고 모든 FD와 PID 배열을 회수해야 zombie와 hang을 남기지 않습니다.

### 답변 핵심 키워드

N-1 pipes · `dup2` · close inherited ends · EOF propagation · partial spawn · kill and reap · last-command status

### 백지 구현

**구현 목표**

argv 배열 목록으로 2~4단계 pipeline을 실행하는 축소 함수를 구현한다. builtin, heredoc, 일반 redirection은 제외하지만 부분 생성 실패 정리는 포함한다.

**인터페이스 또는 함수 시그니처**

```c
int run_pipeline(char ***argvv, size_t command_count)
{
    // 직접 구현
}
```

**입력과 출력**

- `argvv[i]`: NULL 종료 argv를 가진 외부 명령
- `command_count`: 1 이상
- 성공적으로 전부 생성·대기한 경우 마지막 명령의 상태
- 생성·대기 실패 시 1

**반드시 만족해야 할 조건**

- 정확히 `command_count - 1`개의 pipe를 만든다.
- 각 자식의 stdin/stdout을 인덱스에 맞게 연결한다.
- 자식은 연결 후 모든 pipe FD를 닫고 `execvp`한다.
- 부모는 spawn 시도 후 모든 pipe FD를 닫는다.
- 모든 생성된 자식을 반드시 wait한다.
- 부분 fork 실패 시 생성된 자식을 종료하고 reap한다.

**경계 조건**

- 단일 명령
- 첫·마지막 자식의 한쪽 연결 없음
- 빈 `argv[0]`
- pipe 생성 도중 실패
- 첫 fork 또는 마지막 fork 실패

**실패 조건**

- pipe, fork, `dup2`, `execvp`, `waitpid` 실패
- `waitpid`의 `EINTR`
- PID·pipe 배열 할당 실패
- 정리 과정의 kill에서 이미 종료된 자식

**제약**

- background 실행과 job control은 구현하지 않는다.
- wait 결과는 마지막 명령만 pipeline 상태로 사용하되 모든 자식을 reap한다.
- FD를 닫는 helper를 사용해도 되지만 소유권은 설명해야 한다.

### 구현 후 자가 검증

- [ ] 정상 경로: 1단계, 2단계, 4단계 pipeline 출력이 맞다.
- [ ] EOF: writer가 끝난 뒤 마지막 reader가 hang하지 않는다.
- [ ] resource cleanup: 부모와 각 자식이 소유하지 않는 FD를 모두 닫는다.
- [ ] 부분 실패: 두 번째 fork 실패 후 첫 자식이 남지 않는다.
- [ ] wait: `EINTR`를 재시도하고 모든 PID를 한 번씩 reap한다.
- [ ] 상태: 마지막 명령 상태만 정상 pipeline 결과가 된다.
- [ ] 시간·공간 복잡도: 프로세스와 FD 수가 O(N)이다.

### 구현 후 설명할 것

1. 각 프로세스별 FD 테이블을 그려서 소유권을 설명
2. 불필요한 write end 하나가 EOF를 지연시키는 이유
3. pipe wiring과 command redirection의 적용 순서
4. 부분 성공을 정상 성공보다 더 어렵게 만드는 정리 책임
5. 병렬 실행을 유지하면서 wait 순서와 결과 상태를 분리하는 방법

### 원본 확인 위치

- Thread 06
- 커밋 `feat(exec): 다단 pipeline의 pipe FD 연결`
- 커밋 `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수`
- 커밋 `fix(exec): pipe 생성 실패 시 PID 배열 해제`
- `src/exec.c`: `run_forked_pipeline`, `run_child`, `close_pipes`, `terminate_children`, `wait_for_child`
- `src/runtime.c`: `shell_pipe`, `shell_fork`, `shell_waitpid`
- `tests/lifecycle.sh`, `tests/faults.sh`
- 관련 Thread 07, Thread 09, Thread 11

---

<a id="p10"></a>
## P10. [Thread 06 / `test(status): 실행 불가 파일과 신호 종료 상태 검증`] wait status와 실행 오류의 셸 상태 매핑

### 면접 질문

- `waitpid`가 채운 정수에서 정상 종료와 signal 종료를 어떻게 구분합니까?
- signal로 종료된 자식 상태를 `128 + signal`로 만드는 이유와 한계는 무엇입니까?
- `execvp`가 `ENOENT`일 때 127, 그 외 실행 불가일 때 126을 반환하도록 어디에서 결정합니까?
- 꼬리 질문: pipeline 중간 명령이 signal 종료되고 마지막 명령은 성공했다면 pipeline 상태는 무엇입니까?
- 꼬리 질문: `waitpid` 자체가 실패한 경우 마지막 명령 상태를 신뢰할 수 있습니까?

### 30초 모범 답변

wait status는 일반 정수가 아니라 매크로로 해석해야 하며, 정상 종료는 `WEXITSTATUS`, signal 종료는 `128 + WTERMSIG`로 셸 상태에 투영합니다. `execvp`가 돌아왔다는 것은 실행 실패이므로 자식 안에서 errno를 저장한 뒤 `ENOENT`는 127, 그 밖의 실행 불가는 126으로 `_exit`합니다. 정상 pipeline은 마지막 명령의 상태를 대표값으로 쓰지만 모든 자식은 반드시 reap합니다. wait 자체가 실패하면 관찰이 불완전하므로 일반 실패 1로 덮는 편이 안전합니다.

### 답변 핵심 키워드

`WIFEXITED` · `WEXITSTATUS` · `WIFSIGNALED` · `WTERMSIG` · 126 · 127 · 128+signal

### 백지 구현

**구현 목표**

wait status와 exec 실패 errno를 셸 상태로 변환하는 두 개의 작은 순수 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```c
int status_from_wait(int wait_status)
{
    // 직접 구현
}

int status_from_exec_errno(int error_number)
{
    // 직접 구현
}
```

**입력과 출력**

- `wait_status`: `waitpid`가 성공적으로 채운 값
- `error_number`: 실패 직후 저장한 errno
- 셸이 사용할 정수 상태

**반드시 만족해야 할 조건**

- 정상 종료는 자식 exit status를 반환한다.
- signal 종료는 `128 + signal number`를 반환한다.
- 다른 wait 상태는 1로 처리한다.
- `ENOENT`는 127, 그 외 exec 실패는 126이다.
- errno는 진단 출력 전 지역 변수에 보존한다.

**경계 조건**

- exit 0, exit 255
- SIGTERM, SIGKILL
- stopped·continued 상태가 전달될 가능성
- ENOENT와 권한 부족

**실패 조건**

- `waitpid` 호출 자체 실패는 이 순수 함수 밖에서 1로 처리
- 매크로 판정 없이 비트 연산을 추측하는 구현
- 진단 함수 호출로 errno가 바뀐 뒤 상태를 계산하는 버그

**제약**

- POSIX wait 매크로만 사용한다.
- 동적 할당과 전역 상태를 사용하지 않는다.
- pipeline 결과 선택은 구현 범위 밖이다.

### 구현 후 자가 검증

- [ ] 정상 경로: exit 0, 7, 255 매핑이 맞다.
- [ ] signal 경로: SIGTERM이 143이 된다.
- [ ] exec 경로: 없는 명령은 127, 실행 권한 없음은 126이다.
- [ ] 경계값: 알 수 없는 wait 상태는 1이다.
- [ ] invariant: 변환 함수가 외부 상태를 변경하지 않는다.
- [ ] 요구사항: raw wait status 자체를 반환하지 않는다.

### 구현 후 설명할 것

1. wait status가 왜 일반 exit code가 아닌지
2. 126과 127을 자식 exec 경계에서 결정하는 이유
3. 마지막 command 정책과 전체 자식 회수의 독립성
4. signal 번호에 128을 더하는 관례의 이식성 한계
5. wait 실패를 성공 상태와 섞지 않는 방법

### 원본 확인 위치

- Thread 06
- 커밋 `test(status): 실행 불가 파일과 신호 종료 상태 검증`
- `src/exec.c`: `status_from_wait`, 자식 `execvp` 실패 경로
- `tests/smoke.sh`: 실행 불가 파일·signal 종료 회귀
- 관련 Thread 05, Thread 11

---

<a id="p11"></a>
## P11. [Thread 07 / `fix(redirection): 부모 표준 입출력 복원 실패 전파`] source-order 리다이렉션과 부모 stdio 트랜잭션

### 면접 질문

- `echo x > first > second`에서 first 파일도 생성되지만 최종 출력은 second로 가는 이유를 설명해 보십시오.
- 부모에서 builtin을 실행할 때 stdin/stdout을 저장·적용·실행·복원하는 순서를 왜 트랜잭션으로 봐야 합니까?
- 리다이렉션 적용 도중 실패해도 이미 적용된 FD를 원상 복구해야 하는 이유는 무엇입니까?
- 꼬리 질문: 복원 `dup2`가 반복해서 실패하면 왜 셸 loop를 중단하는 선택이 합리적입니까?
- 꼬리 질문: stdio 버퍼와 직접 `write`를 섞을 때 flush 시점이 어떤 문제를 만듭니까?

### 30초 모범 답변

리다이렉션은 source order로 적용하므로 앞선 open의 생성·truncate 부작용은 남고, 뒤의 `dup2`가 최종 stdout 대상을 덮어씁니다. 부모 builtin은 셸 프로세스의 FD를 직접 바꾸기 때문에 실행 전 원본을 복제하고, 적용이나 builtin이 실패해도 반드시 복원해야 합니다. 복원 오류는 다음 명령이 잘못된 입력이나 출력으로 실행될 수 있는 상태 오염입니다. 재시도 후에도 복구하지 못하면 상태 1을 반환하고 셸 실행을 멈춰 더 큰 데이터 손상을 막는 것이 안전합니다.

### 답변 핵심 키워드

source order · open side effect · `dup`/`dup2` · save-apply-run-restore · state contamination · fatal restore failure

### 백지 구현

**구현 목표**

제공된 `exec_apply_redirections`와 `builtin_run`을 이용해 부모 command 실행 wrapper를 구현한다. 핵심은 FD 저장, 항상 복원, 복원 실패 전파다.

**인터페이스 또는 함수 시그니처**

```c
int exec_run_parent_command(t_shell *shell,
    const t_command *command,
    const struct exec_context *ctx)
{
    // 직접 구현
}
```

**입력과 출력**

- `shell`: 상태와 `running` flag
- `command`: argv와 source-order redirection 목록
- `ctx`: heredoc 본문 등 실행 문맥
- builtin 또는 리다이렉션 결과 상태

**반드시 만족해야 할 조건**

- stdin과 stdout을 모두 적용 전에 저장한다.
- 한쪽 저장 실패 시 성공한 다른 saved FD를 닫는다.
- redirection 적용 실패 시에도 저장한 두 스트림을 복원한다.
- redirection-only command는 builtin 호출 없이 성공할 수 있다.
- 복원은 두 스트림 모두 시도한 뒤 saved FD를 닫는다.
- 영구 복원 실패 시 `shell->running = 0`으로 둔다.

**경계 조건**

- 리다이렉션 없음
- 입력만 또는 출력만 리다이렉션
- 여러 출력 리다이렉션
- argv가 없는 command
- stdin 복원은 성공하고 stdout 복원은 실패하는 경우

**실패 조건**

- 첫 번째 또는 두 번째 `dup` 실패
- open·`dup2` 등 redirection 적용 실패
- builtin 출력 실패
- 복원 `dup2`의 `EINTR`와 반복 실패
- close 오류를 상태에 포함할지에 대한 명시적 정책

**제약**

- saved FD는 함수 밖으로 노출하지 않는다.
- 복원 실패를 무시하지 않는다.
- 완전한 stderr 리다이렉션은 현재 프로젝트 범위 밖이다.

### 구현 후 자가 검증

- [ ] 정상 경로: 부모 builtin 후 다음 명령의 stdin/stdout이 원래 스트림이다.
- [ ] source order: 두 출력 리다이렉션의 파일 부작용과 최종 대상이 맞다.
- [ ] 적용 실패: 앞서 바뀐 stdout이 복원된다.
- [ ] 복원 실패: 상태 1이며 영구 실패에서는 loop가 중단된다.
- [ ] resource cleanup: saved FD를 모든 경로에서 정확히 한 번 닫는다.
- [ ] invariant: 함수 종료 후 성공 경로에서 부모 표준 스트림이 원래 대상이다.
- [ ] 중복 처리: 한 스트림 실패 때문에 다른 스트림 복원을 건너뛰지 않는다.

### 구현 후 설명할 것

1. 리다이렉션을 source order로 적용해야 하는 관찰 가능한 부작용
2. 부모 FD 변경을 트랜잭션으로 모델링한 이유
3. restore에서 `EINTR`와 영구 오류를 다르게 다루는 기준
4. 복원 불가 상태에서 계속 실행하는 위험
5. stdio 버퍼 대신 write-all 경계를 둔 이유

### 원본 확인 위치

- Thread 07
- 커밋 `feat(redirection): 파일 입출력 리다이렉션 적용`
- 커밋 `feat(exec): 부모 builtin의 표준 스트림 복원`
- 커밋 `fix(redirection): 부모 표준 입출력 복원 실패 전파`
- 커밋 `test(redirection): 저장·적용·복원 실패 회귀 검증`
- `src/redirection.c`: `exec_apply_redirections`, `save_stdio`, `restore_one`, `restore_stdio`, `exec_run_parent_command`
- `src/runtime.c`: `shell_open`, `shell_dup`, `shell_dup2`
- 관련 Thread 05, Thread 09

---

<a id="p12"></a>
## P12. [Thread 08 / `feat(heredoc): 구분자별 본문 순차 수집`] heredoc 사전 수집, 인용 의미와 입력 경계 복구

### 면접 질문

- `false && cat <<EOF ...`처럼 실행되지 않을 분기의 heredoc도 왜 먼저 읽어야 합니까?
- 구분자가 작은따옴표·큰따옴표·부분 인용된 경우 본문 확장 여부를 어떻게 결정합니까?
- 본문 저장소를 문자열 delimiter가 아니라 정확한 `t_redir *`에 연결한 이유는 무엇입니까?
- 첫 heredoc 준비 중 할당 실패가 났는데 같은 명령줄에 heredoc이 더 있다면 입력 스트림을 어떻게 복구합니까?
- 꼬리 질문: EOF와 실제 read 오류를 같은 NULL로 취급하면 어떤 잘못된 경고나 무한 대기가 생길 수 있습니까?

### 30초 모범 답변

heredoc은 실행 분기보다 입력 소비 규칙이 먼저이므로, 한 명령줄에 등장한 모든 구분자를 source order로 사전 수집해야 다음 command 경계가 보존됩니다. 구분자에 인용이 하나라도 있으면 delimiter 자체는 dequote하지만 본문의 `$NAME`과 `$?` 확장은 끕니다. 동일한 delimiter 문자열이 여러 번 나올 수 있어 본문은 정확한 redirection 노드 identity에 연결합니다. 준비 실패 뒤에도 남은 heredoc delimiter까지 입력을 drain하고, 실제 read 오류로 정렬을 보장할 수 없으면 셸을 중단합니다.

### 답변 핵심 키워드

precollection · input alignment · quoted delimiter · dequote not expand · redir identity · drain on failure · EOF vs error

### 백지 구현

**구현 목표**

파싱된 전체 명령 그래프를 source order로 순회해 heredoc 본문을 수집하고 실행 context에 연결한다. 라인 읽기·본문 확장·노드 생성 helper는 제공되며, 순서와 실패 복구를 구현한다.

**인터페이스 또는 함수 시그니처**

```c
int exec_prepare_heredocs(struct exec_context *ctx,
    t_pipeline *pipelines)
{
    // 직접 구현
}
```

**입력과 출력**

- `ctx->shell`: 환경과 상태
- `pipelines`: 전체 명령줄 그래프
- `ctx->heredocs`: 성공한 본문 entry 목록
- 모두 준비하면 0, 하나라도 실패하면 1

**반드시 만족해야 할 조건**

- pipeline, command, redirection의 source order로 모든 heredoc을 방문한다.
- 조건 분기 실행 여부를 여기서 판단하지 않는다.
- 본문 entry는 해당 `t_redir *`와 연결한다.
- `heredoc_quoted`가 참이면 본문 확장을 하지 않는다.
- 첫 준비 실패 뒤의 heredoc도 delimiter까지 drain한다.
- 호출자는 성공·실패 모두에서 누적 entry를 해제할 수 있어야 한다.

**경계 조건**

- heredoc이 없는 명령줄
- 같은 delimiter를 가진 여러 heredoc
- 한 command의 연속 heredoc
- 부분 인용 delimiter
- delimiter 전에 EOF

**실패 조건**

- delimiter dequote 또는 본문 버퍼 할당 실패
- line read 실패와 정상 EOF
- 본문 확장 실패
- entry 노드 할당 실패
- drain 과정 자체의 read 실패

**제약**

- 실제 tmpfile·`dup2` 적용은 이 함수 범위 밖이다.
- delimiter 문자열 내용만으로 entry를 조회하지 않는다.
- 실패한 명령줄은 실행하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 단일·복수 heredoc을 source order로 수집한다.
- [ ] 조건 분기: 실행되지 않을 branch의 heredoc도 입력에서 소비된다.
- [ ] 인용: unquoted는 제한된 확장, quoted·partially quoted는 literal 본문이다.
- [ ] identity: 같은 delimiter 두 개가 서로 다른 본문을 찾는다.
- [ ] 실패 복구: 첫 준비 실패 뒤 다음 delimiter까지 drain해 다음 command line이 어긋나지 않는다.
- [ ] EOF와 오류: 정상 EOF 경고와 read 실패 상태를 구분한다.
- [ ] resource cleanup: 부분 entry와 body가 모두 해제 가능하다.

### 구현 후 설명할 것

1. 실행 순서와 입력 소비 순서가 다른 이유
2. quote metadata를 lexer에서 heredoc 실행까지 보존한 경로
3. pointer identity 기반 저장소의 장점과 AST 수명 의존성
4. 실패 후 drain이 command-level recovery의 전제인 이유
5. read 오류가 지속될 때 loop 중단이 필요한 기준

### 원본 확인 위치

- Thread 08
- 커밋 `feat(heredoc): 수집 본문 저장소 수명 관리`
- 커밋 `feat(heredoc): 구분자별 본문 순차 수집`
- 커밋 `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존`
- 커밋 `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`
- `src/heredoc.c`: `exec_prepare_heredocs`, `read_heredoc`, `discard_heredoc`, `delimiter_matches`, `add_heredoc_entry`, `exec_find_heredoc_body`
- `src/exec_internal.h`: `struct heredoc_entry`, `struct exec_context`
- 관련 Thread 01, Thread 02, Thread 09

---

<a id="p13"></a>
## P13. [Thread 08 / `fix(heredoc): 임시 파일 저장 오류를 전파`] heredoc 본문의 임시 저장과 I/O 오류 전파

### 면접 질문

- 수집한 heredoc 문자열을 command stdin으로 연결할 때 임시 파일을 사용한 이유는 무엇입니까?
- `fputs`만 성공하면 저장이 끝났다고 볼 수 없는 이유는 무엇입니까?
- `fflush`, `fseek`, `fileno`, `dup2` 각각의 실패를 무시하면 어떤 데이터 손상이 생깁니까?
- 꼬리 질문: 작은 heredoc에 pipe를 쓰는 방식과 tmpfile 방식의 deadlock·용량·수명 trade-off는 무엇입니까?
- 꼬리 질문: 오류를 출력하는 동안 errno가 바뀌지 않게 하는 방법은 무엇입니까?

### 30초 모범 답변

임시 파일은 수집을 실행 전에 끝내고 나중에 stdin으로 재사용하기 쉬우며 pipe buffer 크기에 따른 선행 write block을 피합니다. stdio write는 사용자 버퍼에만 남을 수 있으므로 flush, 시작 위치로 seek, 유효 descriptor 획득, `dup2`까지 모두 성공해야 완전한 입력으로 사용할 수 있습니다. 중간 실패를 무시하면 잘린 heredoc을 정상 데이터처럼 실행할 수 있습니다. 그래서 최초 errno를 보존해 진단하고 stream을 닫은 뒤 전체 리다이렉션 적용을 실패시킵니다.

### 답변 핵심 키워드

`tmpfile` · buffered I/O · `fflush` · `fseek` · `fileno` · `dup2` · truncation prevention · errno preservation

### 백지 구현

**구현 목표**

이미 수집된 heredoc body를 임시 파일에 완전히 저장한 뒤 현재 stdin으로 연결하는 축소 helper를 구현한다.

**인터페이스 또는 함수 시그니처**

```c
int heredoc_body_to_stdin(const char *body)
{
    // 직접 구현
}
```

**입력과 출력**

- `body`: NUL 종료 본문, NULL은 빈 본문으로 취급 가능
- 성공 시 현재 프로세스 stdin이 본문 처음을 가리키고 0
- 어느 단계든 실패하면 1

**반드시 만족해야 할 조건**

- `tmpfile` 성공 여부를 확인한다.
- 본문 write, flush, 시작 위치 seek를 모두 확인한다.
- 유효한 descriptor를 얻은 뒤 `dup2`한다.
- `dup2` 뒤에는 임시 FILE을 닫아도 stdin 복제본은 유효해야 한다.
- 실패 시 임시 stream을 닫고 부분 데이터를 실행에 사용하지 않는다.
- 오류 진단 전 errno를 보존한다.

**경계 조건**

- 빈 body
- 매우 큰 body
- 본문 끝의 newline 유무
- `fileno`가 실패하는 비정상 stream

**실패 조건**

- tmpfile 생성 실패
- write, flush, seek, fileno, `dup2` 실패
- `fclose` 실패를 어떻게 취급할지 명시
- 오류 처리 중 errno 덮어쓰기

**제약**

- 본문 내용은 수정하지 않는다.
- pipe 기반 대안은 구현하지 않는다.
- 성공 후 임시 FILE 객체를 누수하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 빈·여러 줄·큰 body를 stdin에서 정확히 읽는다.
- [ ] 데이터 정합성: write 성공 뒤 flush 실패 시 실행을 진행하지 않는다.
- [ ] 경계값: seek 실패와 fileno 실패를 별도 경로로 검증한다.
- [ ] resource cleanup: 모든 실패 단계에서 FILE이 닫힌다.
- [ ] 상태: 성공한 `dup2` 뒤 `fclose`가 stdin을 닫지 않는다.
- [ ] 오류: 최초 errno에 맞는 진단을 유지한다.
- [ ] 공간 복잡도: 이미 있는 body 외에 파일 저장을 사용하므로 추가 heap 복사가 필요 없다.

### 구현 후 설명할 것

1. pipe와 tmpfile 중 선행 수집에 tmpfile이 유리한 이유
2. stdio 계층과 FD 계층 사이의 경계
3. flush·seek를 생략했을 때 나타나는 데이터 절단
4. `dup2` 이후 FILE 수명과 복제된 FD 수명
5. 각 I/O 호출을 runtime wrapper로 분리한 테스트 이점

### 원본 확인 위치

- Thread 08
- 커밋 `refactor(runtime): heredoc 임시 파일 I/O 경계 분리`
- 커밋 `fix(heredoc): 임시 파일 저장 오류를 전파`
- 커밋 `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증`
- `src/redirection.c`: heredoc 분기, `heredoc_stream_error`
- `src/runtime.c`: `shell_fflush`, `shell_fseek`, `shell_fileno`, `shell_dup2`
- `tests/faults.sh`
- 관련 Thread 07, Thread 09
