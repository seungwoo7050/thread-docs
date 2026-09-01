# 실패 주입, I/O 계약, 성능과 검증

이 문서는 희귀 실패를 결정적으로 재현하는 runtime seam, 상각 선형 문자열 조립, POSIX I/O의 부분 성공, 그리고 이를 자동화하는 검증 파이프라인을 다룬다.

## 문서 내 면접 포인트

- [P14. 결정적 실패 주입과 명령 단위 복구](#p14)
- [P15. 상각 O(n) 문자열 조립과 overflow-safe 성장](#p15)
- [P16. EOF·read 오류·partial write를 구분하는 I/O 계약](#p16)
- [P17. 결정적 회귀·장애·수명 검증 파이프라인](#p17)

---

<a id="p14"></a>
## P14. [Thread 09 / `refactor(runtime): 프로세스 시스템 호출 경계 분리`] 결정적 실패 주입과 명령 단위 복구

### 면접 질문

- pipe·fork·waitpid·dup2·malloc 같은 실패를 실제 자원 고갈 없이 어떻게 재현 가능하게 만들었습니까?
- 두 번째 fork만 실패시키는 경우와 두 번째부터 계속 실패시키는 경우를 어떻게 구분해 주입합니까?
- 할당 실패에 command 번호와 scope를 함께 둔 이유는 무엇입니까?
- 일반 명령 실패 뒤에는 다음 명령을 계속 읽지만, 지속적 입력 실패나 stdio 복원 실패에서는 왜 loop를 중단합니까?
- 꼬리 질문: production 코드에 wrapper를 넣는 비용과 테스트 가능성의 trade-off는 무엇입니까?
  - 모범답변: 얇은 함수 호출과 API 표면이 늘고 모든 호출부가 wrapper를 일관되게 사용해야 합니다. 대신 희귀한 syscall·할당 실패를 동일한 production 정리 경로에 결정적으로 넣을 수 있으며, 현재 구현은 주입 로직을 `SMALL_SHELL_TESTING` 빌드에만 포함해 일반 실행 비용을 제한합니다.

### 30초 모범 답변

외부 호출을 runtime wrapper로 모으고 테스트 빌드에서 호출 횟수, 명령 번호, 할당 scope를 기준으로 원하는 지점에 errno와 실패를 반환하게 했습니다. 정확히 한 번 실패하는 모드와 target 이후 반복 실패 모드를 분리하면 일시 오류와 영구 손상을 모두 검증할 수 있습니다. 각 command 시작 시 카운터를 재설정해 테스트가 다른 명령의 호출 수에 덜 의존하도록 합니다. 그래프나 일회성 syscall 실패는 정리 후 상태 1로 복구하지만, 입력 경계나 부모 stdio를 신뢰할 수 없으면 추가 실행을 멈춥니다.

### 답변 핵심 키워드

runtime wrapper · call index · repeat failure · command scope · allocation scope · errno injection · recoverability boundary

### 백지 구현

**구현 목표**

호출 번호·scope·repeat 정책으로 실패 여부를 결정하는 작은 fault engine과 하나의 `open` wrapper를 구현한다. 환경 변수 파싱은 미리 끝난 설정 구조체로 제공된다.

**인터페이스 또는 함수 시그니처**

```c
typedef struct s_fault_plan {
    const char      *scope;
    unsigned long   target_call;
    int             repeat;
} t_fault_plan;

typedef struct s_fault_state {
    const char      *current_scope;
    unsigned long   calls;
} t_fault_state;

int fault_should_fail(t_fault_state *state,
    const t_fault_plan *plan);

int wrapped_open(t_fault_state *state,
    const t_fault_plan *plan,
    const char *path, int flags, mode_t mode)
{
    if (fault_should_fail(state, plan))
    {
        /* 실제 open 실패와 같은 호출자 경로를 타도록 대표 errno를 설정한다. */
        errno = EACCES;
        return (-1);
    }
    return (open(path, flags, mode));
}
```

**입력과 출력**

- 현재 scope와 누적 호출 수
- target 호출 번호와 repeat 정책
- 실패 시 `-1`과 지정 errno, 아니면 실제 `open` 결과

**반드시 만족해야 할 조건**

- 호출 횟수는 wrapper 호출마다 정확히 증가한다.
- scope가 다르면 target 호출이어도 실패시키지 않는다.
- repeat가 거짓이면 정확히 target에서만 실패한다.
- repeat가 참이면 target 이상에서 계속 실패한다.
- 주입 실패 시 실제 syscall을 호출하지 않는다.
- production 설정이 없으면 원래 syscall과 같은 동작을 한다.

**경계 조건**

- target 0 또는 계획 없음
- 첫 호출 실패
- scope 전환 후 카운터 재설정 정책
- `unsigned long` 최대 근처의 호출 수

**실패 조건**

- 주입 설정 자체가 잘못된 경우
- scope 포인터 NULL
- 실제 open 실패와 주입 open 실패
- 잘못된 errno 선택으로 호출부 분기가 달라지는 경우

**제약**

- thread safety는 현재 단일-thread 셸 범위 밖이지만 전역 카운터의 한계를 설명한다.
- wrapper 내부에서 동적 할당하지 않는다.
- 테스트 빌드 외에는 불필요한 환경 조회를 반복하지 않는 설계를 제안한다.

### 구현 후 자가 검증

- [ ] 정상 경로: plan이 없으면 실제 함수 결과를 그대로 전달한다.
- [ ] 정확한 실패: target 전후 호출은 성공하고 target만 실패한다.
- [ ] 반복 실패: target 이후 모든 호출이 실패한다.
- [ ] scope: 다른 단계 호출이 카운터나 실패를 오염시키지 않는다.
- [ ] 상태 변화: command 시작 시 필요한 카운터가 초기화된다.
- [ ] 실패 경로: 주입 실패에서 실제 자원 생성이 일어나지 않는다.
- [ ] 요구사항: 호출부가 실제 실패와 동일한 정리 경로를 탄다.

### 구현 후 설명할 것

1. 링크 대체나 mocking 대신 얇은 wrapper를 선택한 이유
   - 모범답변: 호출부가 실제 POSIX API와 거의 같은 시그니처를 유지하면서 테스트 빌드에서만 실패 정책을 삽입할 수 있습니다. 별도 linker 설정 없이 제품의 정리 분기를 그대로 실행한다는 점도 단순합니다.
2. 호출 index 테스트가 구현 변경에 취약해지는 지점과 scope로 완화한 방법
   - 모범답변: 앞에 syscall이나 할당 하나가 추가되면 전역 n번째 호출의 의미가 바뀝니다. 명령 시작마다 카운터를 재설정하고 token·parser·expand·heredoc 같은 scope를 지정해 다른 단계의 호출 변화가 target을 밀지 않게 합니다.
3. 일시 실패와 반복 실패를 둘 다 검증해야 하는 이유
   - 모범답변: 한 번만 실패하면 재시도 또는 다음 command에서 회복하는 경로를 검증할 수 있습니다. target 이후 계속 실패시키면 제한 없는 retry, 입력 drain 불능, stdio 영구 복원 실패처럼 loop를 중단해야 하는 경계를 드러냅니다.
4. 복구 가능한 command 실패와 shell invariant 파괴를 구분하는 기준
   - 모범답변: 부분 그래프·자식·FD를 모두 회수하고 입력과 부모 stdio가 정상이라면 상태 1로 현재 command만 실패시킬 수 있습니다. 다음 command의 시작 위치나 stdin/stdout 대상을 신뢰할 수 없으면 shell invariant가 깨졌으므로 loop를 멈춥니다.
5. 전역 test state를 멀티스레드 환경으로 확장할 때 필요한 변경
   - 모범답변: 현재 단일-thread shell의 정적 카운터는 데이터 경합과 호출 순서 비결정성이 생깁니다. thread-local 또는 mutex/atomic 상태와 thread별 scope·호출 ID가 필요하고, scheduler 순서 대신 명시적 이벤트 조건으로 실패 대상을 정해야 합니다.

### 원본 확인 위치

- Thread 09
- 커밋 `refactor(runtime): 프로세스 시스템 호출 경계 분리`
- 커밋 `refactor(runtime): FD 시스템 호출 경계 분리`
- 커밋 `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합`
- 커밋 `test(memory): 범위별 할당 실패 순회 검증`
- `src/runtime.c`, `src/runtime.h`: syscall·I/O·allocation wrapper, `shell_runtime_begin_command`, `shell_runtime_set_alloc_scope`
- `src/exec.c`, `src/heredoc.c`, `src/input.c`: scope 설정과 복구 경계
- `tests/faults.sh`, `tests/allocation.sh`
- 관련 Thread 03, Thread 06, Thread 08, Thread 11

---

<a id="p15"></a>
## P15. [Thread 10 / `refactor(buffer): 가변 문자열 빌더 모듈 추가`] 상각 O(n) 문자열 조립과 overflow-safe 성장

### 면접 질문

- 문자를 하나 붙일 때마다 새 문자열을 할당·복사하면 긴 입력에서 왜 O(n²)이 됩니까?
- 가변 버퍼가 항상 유지해야 하는 length·capacity·NUL 종료 invariant를 설명해 보십시오.
- `length + extra + 1` 계산 전에 overflow를 검사해야 하는 이유는 무엇입니까?
- `realloc` 실패 시 기존 포인터와 데이터는 어떻게 보존해야 합니까?
- 꼬리 질문: capacity를 2배로 늘리는 방식의 상각 복잡도와 최악 메모리 낭비를 비교해 보십시오.
  - 모범답변: capacity가 기하급수적으로 커지면 각 바이트가 성장 복사에 참여하는 총 횟수가 상수배라 append 전체가 상각 O(n)입니다. 막 성장한 직후에는 필요한 크기의 거의 2배까지 확보할 수 있어 사용하지 않는 공간이 capacity의 절반 가까이 될 수 있습니다.

### 30초 모범 답변

매 append마다 지금까지의 전체 문자열을 복사하면 누적 복사량이 1부터 n까지 합쳐져 O(n²)이 됩니다. builder는 `length + 1 <= capacity`, `data[length] == '\0'`를 유지하고, 부족할 때만 capacity를 기하급수적으로 늘려 전체 조립을 상각 O(n)으로 만듭니다. 필요 크기 계산은 `SIZE_MAX` overflow를 먼저 막고, `realloc` 결과는 임시 포인터에 받아 실패해도 기존 버퍼를 잃지 않습니다. `take`는 소유권을 호출자에게 넘긴 뒤 builder를 빈 상태로 초기화합니다.

### 답변 핵심 키워드

amortized O(n) · capacity doubling · `SIZE_MAX` · NUL invariant · realloc temporary · ownership take

### 백지 구현

**구현 목표**

프로젝트의 문자열 builder 핵심 API 중 init, reserve, append char/text, take, discard를 구현한다.

**인터페이스 또는 함수 시그니처**

```c
typedef struct s_string_builder {
    char    *data;
    size_t  length;
    size_t  capacity;
} t_string_builder;

int string_builder_init(t_string_builder *builder);
void string_builder_discard(t_string_builder *builder);
int string_builder_append_char(t_string_builder *builder, char value);
int string_builder_append_text(t_string_builder *builder,
    const char *text);
char *string_builder_take(t_string_builder *builder);
```

**입력과 출력**

- builder 상태와 추가할 문자 또는 문자열
- append 성공 시 0, 실패 시 1
- `take` 성공 시 호출자가 소유할 NUL 종료 문자열

**반드시 만족해야 할 조건**

- 초기화 직후 빈 NUL 종료 문자열이다.
- 항상 `length < capacity`이고 `data[length] == '\0'`이다.
- 필요 크기 계산 전에 `size_t` overflow를 검사한다.
- 성장 시 기하급수 증가하되 overflow 근처에서는 필요한 크기로 제한한다.
- `realloc` 실패 시 기존 data와 메타데이터가 유효하다.
- `take` 뒤 builder는 discard 가능한 빈 상태다.

**경계 조건**

- 빈 문자열 append
- NULL text를 no-op으로 볼지 오류로 볼지 명시
- capacity 경계에서 정확히 한 문자 추가
- `SIZE_MAX` 근처 length·extra
- `take` 후 재사용

**실패 조건**

- 초기 malloc 실패
- reserve의 overflow
- realloc 실패
- 초기화되지 않은 builder 사용

**제약**

- 문자 append마다 문자열 전체 길이를 다시 계산하지 않는다.
- append 전체가 실패하면 기존 내용은 유지한다.
- 숨은 전역 버퍼를 사용하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 문자와 여러 문자열 append 결과가 정확하다.
- [ ] 경계값: `capacity - 1` 길이에서 추가 시 올바르게 성장한다.
- [ ] invariant: 모든 성공·실패 뒤 NUL 종료와 length/capacity 관계가 유지된다.
- [ ] overflow: 계산이 wrap되기 전에 실패한다.
- [ ] resource cleanup: discard를 여러 상태에서 안전하게 호출한다.
- [ ] 소유권: take 반환값과 builder가 같은 버퍼를 이중 해제하지 않는다.
- [ ] 성능: 512 KiB 단어 조립이 반복 전체 복사 없이 완료된다.

### 구현 후 설명할 것

1. 기존 반복 join의 누적 복사 비용
   - 모범답변: 길이 k인 결과에 한 문자를 붙일 때마다 새 k+1 버퍼로 전체를 복사하면 1+2+...+n 바이트를 옮겨 O(n²)이 됩니다. lexer와 expander의 긴 단어에서 이 비용이 직접 나타납니다.
2. 2배 성장의 상각 분석
   - 모범답변: capacity가 64, 128, 256처럼 늘면 n바이트까지의 성장 복사량은 기하급수 합으로 2n 미만입니다. 각 append의 평균 비용은 상수이고 전체 조립은 O(n)입니다.
3. overflow 검사식을 연산 전에 배치한 이유
   - 모범답변: `length + extra + 1`을 먼저 계산해 wrap되면 작은 capacity를 충분하다고 오판하고 buffer overflow가 납니다. 실제 구현은 `extra > SIZE_MAX - length - 1`을 먼저 검사합니다.
4. realloc 임시 포인터와 실패 원자성
   - 모범답변: `realloc` 반환값을 `grown`에 받고 성공한 뒤에만 `builder->data`와 capacity를 갱신합니다. 실패하면 기존 포인터, 내용, length와 NUL invariant가 그대로 남습니다.
5. take/discard API가 소유권을 명시하는 방식
   - 모범답변: `take`는 data 포인터를 반환하고 builder의 포인터·길이·capacity를 0으로 만들어 호출자에게 소유권을 이전합니다. `discard`는 builder가 아직 소유한 경우만 해제하므로 이중 해제를 피합니다.

### 원본 확인 위치

- Thread 10
- 커밋 `refactor(buffer): 가변 문자열 빌더 모듈 추가`
- 커밋 `refactor(lexer): 단어 조립을 가변 버퍼로 전환`
- 커밋 `refactor(expand): 확장 결과를 가변 버퍼로 조립`
- 커밋 `test(performance): 긴 입력 처리 시간 상한 검증`
- `src/string_builder.c`, `src/string_builder.h`: `t_string_builder`와 공개 API
- `src/token.c`, `src/expand.c`: builder 사용 경로
- `tests/performance.sh`: 524288바이트 입력과 timeout 검증
- 관련 Thread 02, Thread 04, Thread 11

---

<a id="p16"></a>
## P16. [Thread 09 / `test(io): read·write와 heredoc 입력 실패 검증`] EOF·read 오류·partial write를 구분하는 I/O 계약

### 면접 질문

- `read`가 0을 반환한 EOF와 -1을 반환한 오류를 같은 NULL 결과로 합치면 shell loop가 어떤 결정을 잘못합니까?
- `write(fd, buf, n)`이 n보다 작은 양수를 반환할 수 있다는 사실을 builtin 출력에 어떻게 반영했습니까?
- `EINTR`, 0바이트 write, 영구 오류를 각각 어떻게 처리해야 합니까?
- 꼬리 질문: line reader가 문자열 포인터와 별도 `failed` 출력을 함께 반환하는 계약의 장단점은 무엇입니까?
  - 모범답변: NULL 하나로 정상 EOF와 오류를 표현하는 C API의 모호함을 작은 변경으로 해소해 호출자가 loop 종료와 상태 1을 구분할 수 있습니다. 다만 두 출력의 조합을 지켜야 하며 `failed` 초기화 누락 위험이 있어 enum 결과와 별도 line 출력 구조가 더 명시적일 수 있습니다.
- 꼬리 질문: stdio 출력과 직접 write wrapper를 혼용하지 않도록 경계를 정한 이유는 무엇입니까?
  - 모범답변: 서로 다른 buffering과 오류 전파 규칙을 섞으면 FD redirection 전후 출력 순서와 실패 상태가 달라질 수 있습니다. 현재 builtin·환경 출력은 `shell_write_text`로 모아 partial write를 처리하고 command status에 반영합니다.

### 30초 모범 답변

EOF는 정상적인 입력 종료지만 read 오류는 상태 1과 복구 판단이 필요한 실패이므로 별도 채널로 구분해야 합니다. write는 일부만 쓸 수 있어 남은 구간을 반복하고, `EINTR`만 재시도하며 0바이트 write는 진행 불가로 보고 EIO 처리합니다. builtin과 환경 출력이 이 helper를 사용하면 출력 실패가 정확히 command status로 올라옵니다. 입력 오류가 지속돼 command 경계를 보장할 수 없으면 정상 EOF처럼 조용히 끝내지 않고 shell loop를 중단합니다.

### 답변 핵심 키워드

EOF vs error · partial write · `EINTR` · zero write · write-all · failed out-parameter · stream integrity

### 백지 구현

**구현 목표**

POSIX `write`를 감싸 전체 버퍼를 쓰거나 명확히 실패하는 `shell_write_all`을 구현한다. read 쪽 계약은 구현 후 구두로 확장 설명한다.

**인터페이스 또는 함수 시그니처**

```c
ssize_t shell_write(int fd, const void *buffer, size_t size);

int shell_write_all(int fd, const void *buffer, size_t size)
{
    const unsigned char *cursor;

    cursor = (const unsigned char *)buffer;
    while (size > 0)
    {
        ssize_t written;

        written = shell_write(fd, cursor, size);
        if (written > 0)
        {
            cursor += (size_t)written;
            size -= (size_t)written;
        }
        else if (written < 0 && errno == EINTR)
            continue ;
        else
        {
            if (written == 0)
                errno = EIO; /* 진행 없는 성공을 무한 재시도로 바꾸지 않는다. */
            return (1);
        }
    }
    return (0);
}
```

**입력과 출력**

- fd, 시작 버퍼, 전체 바이트 수
- 모든 바이트를 쓰면 0
- 영구 오류 또는 진행 불가이면 1과 유효 errno

**반드시 만족해야 할 조건**

- 양수 partial write만큼 cursor와 남은 길이를 갱신한다.
- `errno == EINTR`인 음수 결과만 재시도한다.
- 0바이트 write는 무한 loop 대신 실패 처리한다.
- 크기 0은 syscall 없이 성공할 수 있다.
- 호출자 버퍼를 수정하지 않는다.

**경계 조건**

- size 0
- 1바이트씩 반복되는 partial write
- 마지막 한 바이트에서 EINTR
- NULL buffer와 size 0·양수의 계약

**실패 조건**

- EPIPE, EIO 등 영구 write 오류
- 0바이트 반환
- 잘못된 fd
- cursor 산술 오류

**제약**

- busy loop를 만들지 않는다.
- 동적 할당을 하지 않는다.
- signal 정책 자체는 변경하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 한 번에 전체 write와 여러 partial write가 같은 결과다.
- [ ] EINTR: 여러 번 방해받아도 남은 데이터부터 계속한다.
- [ ] 영구 오류: 추가 write를 시도하지 않고 실패한다.
- [ ] 0 write: 무한 loop 없이 EIO로 실패한다.
- [ ] 경계값: size 0은 성공한다.
- [ ] 데이터 정합성: 바이트 중복이나 누락이 없다.
- [ ] 복잡도: 성공한 write 호출 수에 비례하고 추가 공간 O(1)이다.

### 구현 후 설명할 것

1. EOF와 I/O 오류가 shell 수명에 미치는 차이
   - 모범답변: command 입력의 EOF는 정상 종료라 loop를 조용히 끝낼 수 있습니다. read 오류는 상태 1로 기록해야 하고, heredoc처럼 입력 경계를 복구할 수 없으면 이후 명령을 신뢰할 수 없어 loop를 중단합니다.
2. POSIX write가 partial할 수 있는 이유와 cursor 관리
   - 모범답변: pipe·terminal·signal과 자원 상태에 따라 요청보다 적은 양수 바이트만 쓸 수 있습니다. 실제로 쓴 만큼 byte cursor를 앞으로 옮기고 남은 size만 다시 전달해야 중복과 누락이 없습니다.
3. `EINTR`만 재시도하고 다른 오류는 전파하는 기준
   - 모범답변: EINTR는 signal 때문에 호출이 완료되지 않은 일시 상태라 같은 남은 구간을 재시도합니다. EPIPE, EBADF, EIO 등은 동일 호출을 반복해도 회복 근거가 없으므로 즉시 호출자에 실패를 전파합니다.
4. 0바이트 write를 진행 불가로 처리한 이유
   - 모범답변: size가 양수인데 0이면 cursor와 남은 길이가 변하지 않아 반복문이 영원히 돕니다. 현재 구현은 errno를 EIO로 만들어 명시적으로 실패시킵니다.
5. line reader에서 포인터 결과와 실패 flag를 분리한 API trade-off
   - 모범답변: 기존 포인터 반환을 유지하면서 `NULL, failed=0`은 EOF, `NULL, failed=1`은 오류로 구분할 수 있어 변경이 작습니다. 대신 호출자가 flag를 반드시 제공·검사해야 하며 tagged enum 결과보다 잘못된 조합을 만들기 쉽습니다.

### 원본 확인 위치

- Thread 09
- 커밋 `test(io): read·write와 heredoc 입력 실패 검증`
- Thread 08 커밋 `fix(input): EOF와 입력 실패를 구분`
- Thread 07 커밋 `fix(io): builtin과 환경 출력 실패를 상태로 전파`
- `src/runtime.c`: `shell_read`, `shell_write`, `shell_write_all`, `shell_write_text`
- `src/input.c`: `read_plain_line`, `shell_read_line`, `shell_loop`
- `src/builtin.c`, `src/env.c`: 출력 실패 전파 경로
- 관련 Thread 07, Thread 08, Thread 11

---

<a id="p17"></a>
## P17. [Thread 11 / `build(test): 테스트 시간 제한 하네스 추가`] 결정적 회귀·장애·수명 검증 파이프라인

### 면접 질문

- 정상 출력 테스트만으로 pipe FD 누수, zombie, 영구 read 실패, 복원 실패를 잡기 어려운 이유는 무엇입니까?
- timeout runner가 단순 `sleep && kill` 스크립트보다 어떤 결정성을 제공합니까?
- `ulimit -n` 압박과 test-only child reap 검사를 함께 쓴 이유는 무엇입니까?
- ASan, UBSan, fault injection, lifecycle test가 각각 잡는 결함 범위를 구분해 보십시오.
- 꼬리 질문: Linux·macOS와 GCC·Clang 조합을 CI에 둔 것이 의미론 테스트와 별개로 주는 가치가 무엇입니까?
  - 모범답변: 같은 기능 테스트라도 libc·syscall 세부 동작, compiler 경고와 sanitizer 지원이 달라 비표준 가정이나 UB 징후를 드러낼 수 있습니다. 이는 모든 동시성 schedule을 증명하지는 않지만 한 플랫폼에만 맞춘 코드를 줄입니다.

### 30초 모범 답변

리소스 누수와 hang은 최종 stdout만 비교해서는 드러나지 않으므로 시간, FD 한도, 남은 자식, 실패 위치를 관찰 가능한 test oracle로 만들어야 합니다. call-index fault injection은 희귀 실패를 재현하고, timeout runner는 전체 process group을 정리해 테스트 자체가 자식을 남기지 않게 합니다. ASan은 주소·누수 계열, UBSan은 정의되지 않은 연산, lifecycle test는 OS 자원 수명, 기능 회귀는 셸 의미를 담당합니다. 서로 다른 compiler와 OS를 통과해야 비표준 가정과 경고 차이도 조기에 발견할 수 있습니다.

### 답변 핵심 키워드

deterministic oracle · timeout · process group · `ulimit` · child reap · fault sweep · ASan · UBSan · CI matrix

### 백지 구현

**구현 목표**

제품 코드 대신 20~30분짜리 테스트 harness를 작성한다. 한 입력 파일을 timeout 안에 실행하고 stdout·stderr·status를 검증하며, 종료 시 임시 파일과 남은 runner를 정리한다.

**인터페이스 또는 함수 시그니처**

```sh
run_case()
{
    # name, input, expected_stdout, expected_status를 받는다.
    name=$1
    input=$2
    expected_stdout=$3
    expected_status=$4
    input_file="$TMP_DIR/$name.in"
    stdout_file="$TMP_DIR/$name.out"
    stderr_file="$TMP_DIR/$name.err"
    expected_file="$TMP_DIR/$name.expected"

    printf '%s' "$input" >"$input_file"
    printf '%s' "$expected_stdout" >"$expected_file"
    set +e
    "$TIMEOUT_RUNNER" 5 "$TEST_BIN" <"$input_file" \
        >"$stdout_file" 2>"$stderr_file"
    status=$?
    set -e
    if [ "$status" -ne "$expected_status" ] \
        || ! cmp -s "$expected_file" "$stdout_file"
    then
        printf '%s: status=%s expected=%s\n' \
            "$name" "$status" "$expected_status" >&2
        printf '%s\n' '--- stdout ---' >&2
        cat "$stdout_file" >&2
        printf '%s\n' '--- stderr ---' >&2
        cat "$stderr_file" >&2
        return 1
    fi
}

run_fault_case()
{
    # name, 환경 변수, 실패 호출 번호, input, expected를 받는다.
    name=$1
    fault_variable=$2
    fail_at=$3
    input=$4
    expected_stdout=$5
    expected_status=$6
    input_file="$TMP_DIR/$name.in"
    stdout_file="$TMP_DIR/$name.out"
    stderr_file="$TMP_DIR/$name.err"
    expected_file="$TMP_DIR/$name.expected"

    printf '%s' "$input" >"$input_file"
    printf '%s' "$expected_stdout" >"$expected_file"
    set +e
    env "$fault_variable=$fail_at" \
        "$TIMEOUT_RUNNER" 5 "$TEST_BIN" <"$input_file" \
        >"$stdout_file" 2>"$stderr_file"
    status=$?
    set -e
    if [ "$status" -ne "$expected_status" ] \
        || ! cmp -s "$expected_file" "$stdout_file"
    then
        printf '%s: injected %s=%s, status=%s expected=%s\n' \
            "$name" "$fault_variable" "$fail_at" \
            "$status" "$expected_status" >&2
        cat "$stdout_file" >&2
        cat "$stderr_file" >&2
        return 1
    fi
}
```

**입력과 출력**

- 테스트 이름, 입력 문자열, 기대 stdout, 기대 상태
- fault 환경 변수와 target 호출 번호
- 성공 시 조용히 반환, 불일치 시 진단 후 비0 종료

**반드시 만족해야 할 조건**

- 입력은 파일로 고정해 producer pipe의 추가 동시성을 제거한다.
- stdout과 stderr를 별도 파일에 캡처한다.
- timeout wrapper를 통해 hang을 유한 시간 안에 실패시킨다.
- status와 stdout을 바이트 단위로 비교한다.
- trap에서 임시 디렉터리와 남은 process를 정리한다.
- fault case는 테스트 빌드와 주입 환경을 명시한다.

**경계 조건**

- 빈 stdout
- 프로그램이 signal로 종료되는 경우
- timeout 자체의 특수 상태
- 테스트가 중간에 INT·TERM을 받는 경우
- stderr가 예상상 비어 있어야 하는 경우

**실패 조건**

- 테스트 대상 hang
- runner 또는 child process 잔존
- 임시 파일 생성 실패
- 예상 status는 맞지만 출력이 잘린 경우
- 환경 변수가 다른 case로 새는 경우

**제약**

- 점수화나 PASS/FAIL 등급 체계를 만들지 않는다.
- 실제 구현과 동일한 로직을 테스트 코드에 복제하지 않는다.
- 외부 네트워크와 비결정적 sleep에 의존하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 성공 case와 의도된 command 실패 case를 구분한다.
- [ ] 시간: hang case가 timeout 상태로 끝난다.
- [ ] resource cleanup: runner와 child process가 테스트 후 남지 않는다.
- [ ] 격리: case마다 입력·출력·환경 파일이 독립적이다.
- [ ] 오류 진단: 실패 시 stdout·stderr·status를 재현 가능하게 보여 준다.
- [ ] 누수 검증: 낮은 FD 한도 반복 실행에서 마지막 marker까지 도달한다.
- [ ] 요구사항: sanitizer와 fault test가 같은 결함을 중복 증명한다고 가정하지 않는다.

### 구현 후 설명할 것

1. 어떤 실패마다 어떤 oracle을 선택했는지
   - 모범답변: 의미 회귀는 stdout·stderr·status, hang은 timeout, FD 누수는 낮은 descriptor 한도에서의 반복 marker, 자식 누수는 test-only reap 검사, 메모리·UB는 sanitizer, 희귀 시스템 오류는 call-index fault 결과를 oracle로 사용합니다.
2. 입력을 pipe가 아니라 파일로 공급해 결정성을 높인 이유
   - 모범답변: producer process의 scheduling, 조기 종료와 SIGPIPE를 테스트 변수에서 제거하고 같은 바이트와 EOF 위치를 매번 제공합니다. 테스트 대상의 pipeline 동시성만 남겨 실패 원인을 좁힐 수 있습니다.
3. timeout runner가 process group을 정리해야 하는 이유
   - 모범답변: shell만 kill하면 그 shell이 만든 외부 command가 고아로 남아 다음 테스트를 오염시킬 수 있습니다. runner는 별도 process group을 만들고 timeout·signal 때 group 전체를 종료·회수합니다.
4. fault injection·sanitizer·lifecycle test의 보완 관계
   - 모범답변: fault injection은 특정 실패 분기 도달을 보장하고, sanitizer는 실행된 경로의 메모리·UB를 관찰합니다. lifecycle test는 FD·자식과 hang처럼 sanitizer가 직접 증명하지 않는 OS 자원 invariant를 반복·압박 조건에서 검사합니다.
5. CI matrix를 늘릴 때 실행 시간과 신뢰도 사이의 trade-off
   - 모범답변: 플랫폼·compiler·sanitizer 조합을 늘리면 환경 의존 결함 탐지 범위가 커지지만 전체 피드백 시간과 flaky 가능성도 늘어납니다. 빠른 의미 회귀는 모든 조합에, 느린 fault sweep과 container sanitizer는 대표 환경에 배치하는 식으로 층화할 수 있습니다.

### 원본 확인 위치

- Thread 11
- 커밋 `test(parser): 공개 parser 오류 반환 검증`
- 커밋 `test(exec): pipe·fork·wait 실패 회귀 검증`
- 커밋 `build(test): 테스트 시간 제한 하네스 추가`
- 커밋 `test(lifecycle): FD와 자식 프로세스 누수 검증`
- 커밋 `test(performance): 긴 입력 처리 시간 상한 검증`
- 커밋 `build(test): ASan·UBSan 검증 경로 추가`
- 커밋 `build: expose deterministic verification targets`
- 커밋 `ci: add cross-platform C validation`
- `tests/smoke.sh`, `tests/faults.sh`, `tests/allocation.sh`, `tests/lifecycle.sh`, `tests/performance.sh`, `tests/parser_api.c`, `tests/timeout_runner.c`, `tests/container_sanitizers.sh`
- `Makefile`, `.github/workflows/c-minishell-ci.yml`
- 관련 Thread 06, Thread 09, Thread 10
