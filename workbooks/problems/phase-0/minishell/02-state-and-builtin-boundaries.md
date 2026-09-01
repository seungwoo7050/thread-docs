# 지속 상태와 builtin 실행 경계

이 문서는 프로세스에 남아야 하는 셸 상태와 fork로 격리해야 하는 실행을 구분한다. 환경 snapshot, cwd 전이, exit 상태를 포함해 어디에서 실행해야 변화가 지속되는가를 핵심 invariant로 삼는다.

## 문서 내 면접 포인트

- [P05. 상태를 보존하는 builtin의 부모·자식 실행 경계](#p05)
- [P06. 환경 상태의 불변식과 exec용 snapshot](#p06)
- [P07. 프로세스 cwd와 환경 메타데이터의 상태 전이](#p07)
- [P08. 숫자 파싱, 종료 상태와 셸 수명](#p08)

---

<a id="p05"></a>
## P05. [Thread 05 / `feat(exec): 단일 명령을 자식에서 실행`] 상태를 보존하는 builtin의 부모·자식 실행 경계

### 면접 질문

- `cd`, `export`, `unset`, `exit`를 항상 fork한 자식에서 실행하면 왜 셸 동작이 깨집니까?
- 반대로 모든 builtin을 부모에서 실행하면 pipeline과 리다이렉션에서 어떤 문제가 생깁니까?
- standalone builtin, pipeline 안의 builtin, redirection-only command를 각각 어느 프로세스에서 실행해야 합니까?
- 꼬리 질문: builtin 분류를 이름 목록으로 관리하는 방식과 capability 기반 분류의 trade-off는 무엇입니까?
- 꼬리 질문: parent 실행 경계와 표준 스트림 복원 책임은 왜 별도 계층이어야 합니까?

### 30초 모범 답변

프로세스의 cwd와 환경, 셸 종료 여부는 부모 셸의 상태이므로 이를 바꾸는 standalone builtin은 부모에서 실행해야 변화가 지속됩니다. 하지만 pipeline의 각 단계는 독립 프로세스와 FD 그래프를 가져야 하므로 그 안의 builtin은 자식에서 실행하고 상태 변경은 복사본에만 남겨야 합니다. 이 구현은 단일 command이면서 parent builtin이거나 redirection-only인 경우만 부모 경로를 선택합니다. 부모 경로에서는 리다이렉션 적용과 원상 복구를 반드시 트랜잭션처럼 감싸야 합니다.

### 답변 핵심 키워드

process isolation · persistent state · standalone builtin · pipeline child · redirection-only · dispatch boundary

### 백지 구현

**구현 목표**

pipeline 구조와 첫 command의 성격을 보고 부모 실행 또는 forked 실행을 선택하는 작은 dispatch 함수를 구현한다. 실제 builtin과 pipeline 실행 함수는 callback으로 제공된다.

**인터페이스 또는 함수 시그니처**

```c
typedef enum e_exec_route {
    EXEC_IN_PARENT,
    EXEC_FORKED
} t_exec_route;

t_exec_route choose_exec_route(const t_pipeline *pipeline,
    int (*is_parent_builtin)(const char *name))
{
    // 직접 구현
}
```

**입력과 출력**

- `pipeline`: command 수와 command 목록을 가진 실행 단위
- `is_parent_builtin`: 이름이 부모 상태를 변경하는 builtin인지 판정
- 부모 실행이면 `EXEC_IN_PARENT`, 그 외 `EXEC_FORKED`

**반드시 만족해야 할 조건**

- command가 정확히 하나일 때만 부모 경로를 고려한다.
- argv가 비어 있고 리다이렉션만 있는 command는 부모 경로다.
- 상태 지속이 필요한 builtin은 standalone일 때 부모 경로다.
- pipeline 안의 모든 builtin은 forked 경로다.
- 외부 명령은 단일이어도 forked 경로다.

**경계 조건**

- 빈 pipeline 또는 command 수 0
- `argv[0] == NULL`인 redirection-only command
- 알려진 stateless builtin과 stateful builtin
- `command_count`와 실제 목록 길이가 불일치하는 손상된 입력

**실패 조건**

- 필수 판정 callback 누락
- 손상된 그래프를 무조건 역참조하지 않음
- 부모·자식 경계를 잘못 선택해 상태가 사라지거나 부모 FD가 오염되는 경우

**제약**

- builtin 실행 자체와 fork 구현은 범위 밖이다.
- 이 함수는 shell 상태를 변경하지 않는다.
- 이름 문자열을 새로 할당하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: standalone `export`, `cd`, `exit`가 부모 경로다.
- [ ] 격리: `export X=1 | cat`은 forked 경로다.
- [ ] 외부 명령: 단일 `ls`도 forked 경로다.
- [ ] 경계값: redirection-only command는 부모 경로다.
- [ ] invariant: command 수 2 이상이면 이름과 무관하게 forked다.
- [ ] 상태 변화: dispatch 함수 자체는 shell을 수정하지 않는다.
- [ ] 요구사항: 분류표 추가만으로 새 parent builtin을 확장할 수 있다.

### 구현 후 설명할 것

1. fork 이후 주소 공간 복사 때문에 자식의 환경 변경이 부모에 남지 않는 이유
2. pipeline builtin을 부모에서 돌릴 수 없는 FD·동시 실행상의 이유
3. stateful/stateless 분류와 실제 현재 구현의 parent 판정 정책
4. redirection-only command가 부모에서 처리되는 이유
5. dispatch와 parent stream transaction을 분리한 설계

### 원본 확인 위치

- Thread 05
- 커밋 `feat(exec): 단일 명령을 자식에서 실행`
- 커밋 `feat(exec): 부모 builtin의 표준 스트림 복원`
- `src/builtin.c`: `builtin_is_parent`, `builtin_is_known`, `builtin_run`
- `src/exec.c`: `execute_one_pipeline`, 자식 실행 경로
- `src/redirection.c`: `exec_run_parent_command`
- 관련 Thread 06, Thread 07

---

<a id="p06"></a>
## P06. [Thread 05 / `feat(env): export 배열과 출력 뷰 생성`] 환경 상태의 불변식과 exec용 snapshot

### 면접 질문

- 환경 변수를 연결 리스트로 관리할 때 반드시 지켜야 할 invariant는 무엇입니까?
- `export NAME`과 `export NAME=value`의 상태 변화가 어떻게 다릅니까?
- 왜 내부 환경 구조를 `execvp`에 직접 넘기지 않고 `char **envp` snapshot으로 변환합니까?
- 꼬리 질문: update 중 문자열 할당이 실패하면 기존 값 보존을 어떻게 보장할 수 있습니까?
- 꼬리 질문: 연결 리스트 대신 hash map을 썼을 때 조회·출력 순서·snapshot 비용이 어떻게 달라집니까?

### 30초 모범 답변

내부 환경은 키가 유일하고 각 노드가 값과 exported 플래그를 소유한다는 invariant가 핵심입니다. `export NAME`은 기존 값을 유지한 채 exported만 올릴 수 있고, `NAME=value`는 값까지 갱신합니다. 외부 프로세스 경계에서는 exported 항목만 `KEY=VALUE` 형태의 독립 snapshot으로 만들어 전달하고, 자식 경로가 그 배열을 해제합니다. 갱신 실패에 강하려면 새 문자열을 먼저 확보한 뒤 기존 값을 교체해야 합니다.

### 답변 핵심 키워드

unique key · exported flag · snapshot · `KEY=VALUE` · ownership transfer · failure-atomic update

### 백지 구현

**구현 목표**

기존 환경 노드를 갱신하거나 새 노드를 추가하는 `env_set`과 exported 노드만 복사하는 `env_to_environ`의 핵심을 구현한다. 정렬 출력은 제외한다.

**인터페이스 또는 함수 시그니처**

```c
int env_set(t_env **env, const char *key,
    const char *value, int exported);

char **env_to_environ(t_env *env);
```

**입력과 출력**

- `env_set`: 리스트 헤드 주소, 키, 선택적 값, export 요청을 받고 성공 시 0
- `env_to_environ`: exported 노드만 담은 NULL 종료 배열을 반환
- snapshot의 각 문자열과 배열은 호출자가 소유

**반드시 만족해야 할 조건**

- 같은 키의 노드를 중복 생성하지 않는다.
- `value == NULL`인 기존 노드 갱신은 기존 값을 유지한다.
- export 요청은 플래그를 참으로 만들며 기존 참 값을 거짓으로 내리지 않는다.
- 새 노드는 키와 값을 독립 복사한다.
- snapshot은 exported 항목만 포함하고 NULL로 끝난다.
- 실패한 갱신이 기존 노드의 유효 문자열을 잃게 하지 않는다.

**경계 조건**

- 빈 환경 리스트
- 빈 값과 값 미지정의 차이
- 이미 존재하는 키
- exported 노드가 하나도 없는 snapshot
- 유효하지 않은 변수 이름

**실패 조건**

- 노드, key, value, snapshot 배열, `KEY=VALUE` 문자열 할당 실패
- 중간 snapshot 생성 실패 시 이미 만든 문자열 모두 회수
- 무효한 key 입력

**제약**

- 환경 이름 검증 helper는 제공된다고 가정한다.
- 리스트 순서는 기존 순서를 유지한다.
- snapshot은 내부 노드 문자열을 빌려 쓰지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 새 변수 추가, 기존 값 교체, export 승격이 동작한다.
- [ ] 경계값: `NAME=`과 `export NAME`을 구분한다.
- [ ] invariant: 동일 key 노드가 둘 이상 존재하지 않는다.
- [ ] 실패 경로: 모든 할당 지점 실패 후 원래 리스트가 여전히 순회 가능하다.
- [ ] resource cleanup: snapshot 중간 실패 시 부분 배열이 전부 해제된다.
- [ ] 누락 처리: 미export 노드가 envp에 포함되지 않는다.
- [ ] 복잡도: 리스트 조회 O(n), snapshot O(n + 총 문자열 길이)를 설명한다.

### 구현 후 설명할 것

1. 내부 mutable state와 exec 경계용 immutable snapshot의 차이
2. 값 미지정과 빈 값의 의미 차이
3. 새 값 선할당 후 교체가 필요한 이유
4. 연결 리스트 선택의 단순성과 조회 비용
5. snapshot 소유권을 부모와 자식 중 어디에 둘지

### 원본 확인 위치

- Thread 05
- 커밋 `feat(env): export 배열과 출력 뷰 생성`
- 커밋 `feat(builtin): export 대입과 선언 출력`
- 커밋 `feat(builtin): unset 환경 이름 제거`
- `src/env.c`: `env_find`, `env_get`, `env_set`, `env_unset`, `env_to_environ`, `env_print`, `shell_env_is_valid_name`
- `src/builtin.c`: `split_assignment`, `builtin_export`, `builtin_unset`
- 관련 Thread 04, Thread 06

---

<a id="p07"></a>
## P07. [Thread 05 / `feat(builtin): cd 이동과 PWD 상태 동기화`] 프로세스 cwd와 환경 메타데이터의 상태 전이

### 면접 질문

- `cd`를 구현할 때 실제 프로세스 cwd와 `PWD`/`OLDPWD`의 갱신 순서를 어떻게 잡았습니까?
- `cd`, `cd -`, 일반 경로의 target 결정 실패를 각각 어떻게 처리합니까?
- `chdir`은 성공했지만 이후 `getcwd`나 환경 문자열 할당이 실패하면 완전한 rollback이 가능한가요?
- 꼬리 질문: 심볼릭 링크가 포함된 논리 경로와 `getcwd`가 돌려주는 물리 경로 중 무엇을 저장하는 설계입니까?
- 꼬리 질문: `cd -`가 새 경로를 출력하는 시점과 출력 실패 상태는 어떻게 연결합니까?

### 30초 모범 답변

현재 작업 디렉터리는 프로세스 전역 상태이므로 먼저 이전 cwd를 확보하고, target을 결정한 뒤 `chdir`에 성공했을 때만 환경 메타데이터 갱신을 시도합니다. 인자 없음은 `HOME`, `-`는 `OLDPWD`를 사용하며 누락되면 상태를 바꾸지 않고 실패합니다. `chdir` 뒤의 할당 실패는 일반 메모리 트랜잭션처럼 완전히 되돌리기 어렵기 때문에 실제 cwd를 진실의 원천으로 두고 메타데이터 동기화의 한계를 명시해야 합니다. 확인된 구현은 `getcwd`가 제공한 값을 이용해 `PWD`와 `OLDPWD`를 갱신합니다.

### 답변 핵심 키워드

process-wide cwd · `HOME` · `OLDPWD` · chdir boundary · best-effort metadata · non-atomic transition

### 백지 구현

**구현 목표**

프로젝트에서 확인된 범위의 `cd` builtin을 구현한다. target 결정, 인자 검증, cwd 변경, `PWD`/`OLDPWD` 동기화까지만 다룬다.

**인터페이스 또는 함수 시그니처**

```c
static int builtin_cd(t_shell *shell, char **argv)
{
    // 직접 구현
}
```

**입력과 출력**

- `shell->env`: `HOME`, `OLDPWD`, `PWD` 상태
- `argv`: `cd`, 선택적 단일 인자
- 성공 시 0, target 결정·인자·`chdir` 실패 시 비0

**반드시 만족해야 할 조건**

- 인자가 둘 이상이면 cwd를 바꾸지 않는다.
- 인자 없음은 비어 있지 않은 `HOME`을 요구한다.
- `cd -`는 비어 있지 않은 `OLDPWD`를 요구한다.
- `chdir` 실패 시 `PWD`와 `OLDPWD`를 갱신하지 않는다.
- 성공 전 cwd와 성공 후 cwd를 별도로 얻어 환경 갱신에 사용한다.
- `cd -`는 새 cwd를 출력한다.

**경계 조건**

- 현재 디렉터리가 삭제돼 이전 `getcwd`가 실패하는 경우
- `chdir`은 성공했지만 새 `getcwd`가 실패하는 경우
- HOME/OLDPWD 미설정 또는 빈 문자열
- 경로가 너무 길거나 접근 권한이 없는 경우

**실패 조건**

- `getcwd` 할당 실패
- `chdir`의 `ENOENT`, `EACCES` 등
- 환경 노드 갱신 실패
- `cd -` 출력 실패

**제약**

- 논리적 `PWD` 경로 보존이나 `CDPATH`는 구현하지 않는다.
- 이미 성공한 `chdir`을 억지로 rollback했다고 가정하지 않는다.
- 환경 갱신의 원자성 한계를 코드 설명에 포함한다.

### 구현 후 자가 검증

- [ ] 정상 경로: 일반 경로와 HOME 이동 후 실제 cwd가 바뀐다.
- [ ] 상태 변화: 성공 전 값이 OLDPWD, 성공 후 값이 PWD가 된다.
- [ ] 실패 경로: 잘못된 경로와 인자 과다에서 cwd가 그대로다.
- [ ] 경계값: HOME/OLDPWD가 없을 때 명확히 실패한다.
- [ ] invariant: `chdir` 실패 전에는 환경 상태를 먼저 바꾸지 않는다.
- [ ] resource cleanup: `getcwd(NULL, 0)` 결과를 모든 경로에서 해제한다.
- [ ] 출력: `cd -` 외에는 불필요한 표준 출력을 만들지 않는다.

### 구현 후 설명할 것

1. cwd는 왜 환경 변수와 다른 종류의 프로세스 상태인지
2. `chdir` 전후에 어떤 값을 먼저 확보해야 하는지
3. 완전한 rollback이 불가능한 실패 경계
4. 물리 경로 기반 PWD 동기화의 의미
5. 환경 갱신 실패를 반환 상태에 반영할지에 대한 trade-off

### 원본 확인 위치

- Thread 05
- 커밋 `feat(builtin): cd 이동과 PWD 상태 동기화`
- `src/builtin.c`: `argv_count`, `builtin_cd`
- `src/env.c`: `env_get`, `env_set`
- 관련 Thread 01, Thread 07

---

<a id="p08"></a>
## P08. [Thread 05 / `feat(builtin): exit 상태를 셸 수명에 연결`] 숫자 파싱, 종료 상태와 셸 수명

### 면접 질문

- `exit`의 인자 없음, 유효 숫자, 숫자 아님, 인자 과다를 어떤 순서로 판정합니까?
- 왜 유효한 큰 정수나 음수도 최종 상태를 8비트로 정규화합니까?
- 숫자가 아닌 첫 인자와 추가 인자가 함께 있을 때도 셸이 종료되어야 합니까?
- 꼬리 질문: `strtol` 사용 시 `errno`, end pointer, 범위 초과를 모두 확인해야 하는 이유는 무엇입니까?
- 꼬리 질문: parser 구문 상태 258과 프로세스 종료 코드 8비트 범위는 어떤 층에서 구분됩니까?

### 30초 모범 답변

인자가 없으면 직전 상태로 종료하고, 첫 인자가 숫자가 아니면 상태 2를 설정한 뒤 셸을 종료합니다. 첫 인자가 유효한 숫자인 경우에만 추가 인자를 검사하며, 인자가 너무 많으면 상태 1을 반환하지만 셸은 계속 실행합니다. 유효 숫자는 프로세스 종료 상태 범위에 맞게 unsigned char로 정규화합니다. `strtol`은 변환 시작 여부, 문자열 전체 소비, `ERANGE`를 함께 확인해야 잘못된 숫자를 놓치지 않습니다.

### 답변 핵심 키워드

`strtol` · end pointer · `ERANGE` · unsigned char · running flag · too many arguments · status 2

### 백지 구현

**구현 목표**

숫자 파서와 `exit` 상태 전이를 구현한다. 실제 프로세스 `exit()` 호출은 하지 않고 `shell->running`과 반환값만 변경한다.

**인터페이스 또는 함수 시그니처**

```c
static int parse_exit_status(const char *text, int *status);

static int builtin_exit(t_shell *shell, char **argv)
{
    // 직접 구현
}
```

**입력과 출력**

- `argv[0]`은 `exit`, 이후 선택적 인자
- `shell->last_status`, `shell->running`을 갱신
- builtin 결과 상태 반환

**반드시 만족해야 할 조건**

- 인자 없음: 기존 `last_status`를 반환하고 `running = 0`
- 숫자 아님 또는 범위 오류: 상태 2, `running = 0`
- 유효 숫자와 추가 인자: 상태 1, `running` 유지
- 유효 숫자 하나: 8비트 정규화한 상태, `running = 0`
- 문자열 일부만 숫자인 입력을 허용하지 않는다.

**경계 조건**

- `0`, `255`, `256`, `-1`
- 앞뒤 공백을 `strtol` 기본 동작대로 허용할지 명시
- 빈 문자열, 부호만 있는 문자열
- `LONG_MIN`, `LONG_MAX`, 범위 초과 문자열

**실패 조건**

- NULL shell 또는 손상된 argv
- 변환 불가 숫자
- 인자 과다
- 상태는 바뀌었지만 `running` 변경을 누락하는 버그

**제약**

- 동적 할당을 하지 않는다.
- 실제 `_exit`나 `exit`를 호출하지 않는다.
- 진단 출력 형식보다 상태 전이 정확성을 우선 검증한다.

### 구현 후 자가 검증

- [ ] 정상 경로: `exit`, `exit 0`, `exit 7`의 상태와 running이 맞다.
- [ ] 경계값: `256 -> 0`, `-1 -> 255`가 된다.
- [ ] 실패 경로: `exit abc`는 상태 2로 종료한다.
- [ ] 인자 과다: `exit 7 8`은 상태 1이지만 계속 실행한다.
- [ ] invariant: 종료 결정과 반환 상태가 서로 모순되지 않는다.
- [ ] 누락 처리: `12x`를 유효 숫자로 받아들이지 않는다.
- [ ] 복잡도: 입력 문자열 길이에 대해 O(n), 추가 공간 O(1)이다.

### 구현 후 설명할 것

1. 검사 순서가 동작 차이를 만드는 이유
2. C 정수 파싱과 shell status 정규화의 경계
3. 상태 반환과 실제 loop 종료를 분리한 이유
4. parser의 내부 상태값과 OS 종료 코드의 층 분리
5. 더 엄격한 숫자 문법을 직접 스캔할지 `strtol`을 쓸지

### 원본 확인 위치

- Thread 05
- 커밋 `feat(builtin): exit 상태를 셸 수명에 연결`
- `src/builtin.c`: `parse_exit_status`, `builtin_exit`, `builtin_run`
- `src/input.c`: `shell_loop`
- 관련 Thread 01, Thread 06
