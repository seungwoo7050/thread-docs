# Signal·Unix Datagram 프로토콜 워크북

이 문서는 signal을 데이터 전달 수단으로 사용하면서 Unix datagram을 세션·응답 제어 채널로 결합한 작업을 다룬다. 핵심은 특정 API 암기가 아니라 **비동기 경계, 식별·상관관계, 파일시스템 endpoint, 외부 효과의 완료 순서**다.

---

<a id="p-01"></a>
## P-01. [Thread 17·18 / `feat(client): 메시지 바이트를 시그널로 전송` · `feat(protocol): NUL 바이트로 메시지 종료 표시`] signal 데이터 채널과 datagram 제어 채널

### 면접 질문

메시지 바이트는 두 종류의 signal로 비트 단위 전송하고, 세션 획득과 READY·ACK 응답은 Unix datagram으로 전달하는 구조입니다. 왜 하나의 채널로 모두 처리하지 않고 데이터 채널과 제어 채널을 분리했으며, 각 채널의 약점을 다른 채널이 어떻게 보완합니까?

꼬리 질문:

- 한 바이트를 MSB-first로 보내면 수신 상태는 어떤 순서로 갱신해야 합니까?
- standard signal은 payload와 buffering이 제한적인데 bit마다 ACK를 기다리는 방식은 어떤 문제를 줄입니까?
- NUL 바이트를 프레임 끝으로 쓰면 빈 문자열과 일반 문자열을 어떻게 구분할 수 있습니까?
- 메시지 내용에 NUL 자체를 포함해야 한다면 현재 framing은 어떻게 바뀌어야 합니까?
- 데이터 비트와 ACK에 sequence가 필요한 이유는 무엇입니까?
- 세션 예약 없이 여러 client의 bit가 섞이면 어떤 상태 오염이 발생합니까?

### 30초 모범 답변

signal은 두 값으로 bit를 전달하기 간단하지만 sender 식별, 구조화된 응답, buffering에 제약이 큽니다. 그래서 실제 byte 데이터는 signal로 유지하되 세션 획득과 READY·ACK는 source와 구조체를 검증할 수 있는 Unix datagram으로 분리했습니다. client는 한 bit를 보낸 뒤 해당 sequence의 ACK를 확인해 signal 유실·병합 위험과 송신 속도를 제어합니다. 수신기는 8비트를 MSB-first로 조립하고 NUL을 메시지 종료 프레임으로 처리합니다.

### 답변 핵심 키워드

hybrid channels, data vs control plane, MSB-first, per-bit acknowledgement, session ownership, sequence, NUL framing, empty message

### 백지 구현

#### 구현 목표

두 signal 값을 한 바이트로 조립하는 순수 상태 머신과, 한 바이트를 8개 bit 값으로 분해하는 함수를 작성한다. 프로세스·소켓 호출은 문제 범위 밖이다.

#### 인터페이스

```c
typedef struct s_bit_decoder
{
	unsigned char	current;
	unsigned int	received_bits;
}   t_bit_decoder;

typedef enum e_decode_result
{
	DECODE_INVALID = -1,
	DECODE_PENDING = 0,
	DECODE_BYTE = 1,
	DECODE_FRAME_END = 2
}   t_decode_result;

void			encode_byte_msb(unsigned char byte, int bits[8]);
t_decode_result	decode_bit(t_bit_decoder *decoder, int bit, unsigned char *byte);
```

#### 입력과 출력

- `encode_byte_msb`: `bits[0]`이 최상위 bit, `bits[7]`이 최하위 bit
- `decode_bit`: 0 또는 1을 한 개 소비
- 8비트 완성 값이 NUL이면 `DECODE_FRAME_END`, 아니면 `DECODE_BYTE`
- 완성 전에는 `DECODE_PENDING`

#### 반드시 만족해야 할 조건

- 한 바이트의 bit 순서는 MSB-first다.
- decoder는 정확히 8개 bit마다 한 번만 결과를 만든다.
- 완성 뒤 다음 바이트를 위해 `current`와 bit count를 초기화한다.
- frame end도 완성된 한 바이트이므로 동일한 8비트 경로를 거친다.
- invalid bit 입력에서 진행 중 상태를 보존할지 reset할지 정책을 명시한다.
- 결과가 없는 경우 output byte를 유효값처럼 사용하지 못하게 한다.

#### 경계 조건

- `0x00`, `0x01`, `0x80`, `0xFF`
- 7비트까지만 입력된 상태
- 연속 두 바이트
- 첫 바이트가 NUL인 빈 메시지
- 일반 byte 바로 다음 NUL
- invalid bit가 중간에 들어온 경우

#### 실패 조건과 제약

- 포인터가 NULL이거나 bit가 0·1 이외인 경우
- 동적 할당과 시스템 호출을 사용하지 않는다.
- sender·sequence 검증은 P-03 문제에서 다룬다.

```c
void	encode_byte_msb(unsigned char byte, int bits[8])
{
	// 직접 구현
}

t_decode_result	decode_bit(t_bit_decoder *decoder, int bit,
		unsigned char *byte)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 모든 0~255 값에 대해 encode 후 decode가 원래 byte를 만든다.
- [ ] `0x80`과 `0x01`로 bit 방향이 뒤집히지 않았음을 확인한다.
- [ ] 7비트 상태에서는 완성 결과가 나오지 않는다.
- [ ] 8번째 bit에서 정확히 한 번 결과를 반환한다.
- [ ] 완성 뒤 다음 byte가 이전 상태의 영향을 받지 않는다.
- [ ] NUL은 데이터 byte가 아니라 frame end 결과로 분류된다.
- [ ] 빈 메시지는 첫 8비트 NUL로 종료된다.
- [ ] invalid 입력 정책이 상태 invariant와 일치한다.
- [ ] 시간·공간 복잡도는 byte당 O(1)이다.

### 구현 후 설명할 것

1. 데이터 채널과 제어 채널을 분리한 이유.
2. bit ACK와 sequence가 제공하는 순서·속도 제어.
3. NUL framing의 장점과 바이너리 payload 제한.
4. 세션 소유자가 아닌 sender의 signal을 버려야 하는 이유.
5. protocol state와 signal handler 실행 문맥을 분리해야 하는 이유.

### 원본 확인 위치

- Thread 17 — signal 데이터와 datagram 제어 채널 결합
- Thread 18 — 시그널 비트 인코딩과 NUL 프레이밍
- 커밋: `feat(client): 메시지 바이트를 시그널로 전송`, `feat(protocol): NUL 바이트로 메시지 종료 표시`
- 파일: `include/minitalk.h`, `src/client.c`, `src/server.c`, `src/parse_pid.c`, `tests/smoke.sh`
- 함수·컴포넌트: `send_bit`, `send_byte`, `flush_byte`, `mt_parse_pid`, `t_mt_request`, `t_mt_response`
- 관련 Thread: 20, 21, 22

---

<a id="p-02"></a>
## P-02. [Thread 19 / `feat(runtime): 안전한 응답 endpoint 경로 생성`] Unix datagram endpoint의 안전한 수명

### 면접 질문

프로세스별 Unix socket 파일을 `/tmp` 아래에 만들 때 단순히 기존 경로를 `unlink`하고 `bind`하면 어떤 보안·안전 문제가 생깁니까? runtime directory와 stale endpoint를 검증할 때 파일 종류, 소유자, 권한, 경로 길이를 각각 왜 확인해야 합니까?

꼬리 질문:

- `stat`보다 `lstat`이 필요한 이유는 무엇입니까?
- 기존 경로가 같은 사용자의 socket일 때만 stale로 보고 지워야 하는 이유는 무엇입니까?
- regular file이나 다른 사용자의 socket이 있으면 왜 덮어쓰지 않고 실패해야 합니까?
- runtime directory가 존재하더라도 group·other 권한이 열려 있으면 왜 거절해야 합니까?
- socket에 `O_NONBLOCK`과 `FD_CLOEXEC`를 설정한 이유는 각각 무엇입니까?
- `atexit` 정리만으로 모든 종료에서 파일 삭제를 보장할 수 있습니까?

### 30초 모범 답변

Unix socket 경로는 파일시스템 namespace이므로 다른 객체를 잘못 삭제하거나 다른 사용자가 만든 endpoint에 연결되는 문제가 있습니다. 전용 runtime directory를 0700으로 만들고 `lstat`으로 실제 디렉터리·소유자·권한을 검증합니다. endpoint가 이미 있으면 같은 사용자의 socket일 때만 stale로 보고 삭제하고, regular file·symlink·다른 소유자는 실패합니다. 경로는 `sun_path`와 자체 버퍼 크기를 모두 검사하고, 생성한 FD에는 nonblocking과 close-on-exec 정책을 적용합니다.

### 답변 핵심 키워드

filesystem namespace, `lstat`, owner UID, mode 0700, stale socket only, no destructive unlink, `sun_path`, nonblocking, close-on-exec, cleanup

### 백지 구현

#### 구현 목표

안전한 프로세스별 endpoint 경로를 만들고, bind 전 stale socket만 제거하는 두 함수를 작성한다.

#### 인터페이스

```c
int	build_endpoint_path(char *buffer, size_t size,
		const char *role, pid_t pid);
int	remove_stale_socket(const char *path);
```

#### 입력과 출력

- 성공 시 0, 실패 시 -1과 적절한 `errno`
- 허용 role은 문제에서 지정한 고정 집합만 인정한다.
- 경로 형식은 전용 runtime directory와 role·PID로 결정한다.

#### 반드시 만족해야 할 조건

- runtime directory가 없으면 제한 권한으로 생성한다.
- 이미 존재하면 실제 디렉터리인지, 현재 UID 소유인지, group·other 접근 비트가 없는지 확인한다.
- path 생성 결과가 출력 버퍼와 Unix socket path 한계를 넘으면 실패한다.
- PID와 role의 유효성을 검사한다.
- stale 제거는 실제 socket이고 현재 UID 소유인 경우에만 수행한다.
- regular file, directory, symlink, 다른 UID 객체는 삭제하지 않는다.
- `ENOENT`는 제거할 것이 없는 성공 상태로 처리한다.

#### 경계 조건

- runtime directory 최초 생성과 재사용
- 잘못된 directory 권한
- directory 대신 regular file이 있는 경우
- 빈 role, 미지원 role
- 유효하지 않은 PID
- 경로가 정확히 들어가는 크기와 한 바이트 부족한 크기
- endpoint 없음
- 같은 UID의 stale socket
- regular file·symlink가 endpoint 자리에 있는 경우

#### 실패 조건과 제약

- `mkdir`, `lstat`, `unlink` 실패
- 경로 truncation
- 공격적인 "무조건 unlink 후 재시도"를 금지한다.
- TOCTOU를 완전히 제거하는 문제는 범위 밖이지만 남는 위험을 설명한다.

```c
int	build_endpoint_path(char *buffer, size_t size,
		const char *role, pid_t pid)
{
	// 직접 구현
}

int	remove_stale_socket(const char *path)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 최초 실행에서 runtime directory가 제한 권한으로 생성된다.
- [ ] 같은 디렉터리를 재사용할 때 소유자·권한 검증을 건너뛰지 않는다.
- [ ] regular file과 symlink를 stale socket으로 오인해 삭제하지 않는다.
- [ ] 같은 UID socket만 삭제한다.
- [ ] 없는 endpoint는 오류로 취급하지 않는다.
- [ ] role·PID가 경로 외부 문자열을 주입하지 못한다.
- [ ] 출력 버퍼와 `sun_path` 양쪽의 truncation을 감지한다.
- [ ] bind·설정 중간 실패에서 열린 FD와 생성 경로를 정리할 수 있다.
- [ ] 정상 종료 뒤 endpoint가 사라지고, 비정상 종료 뒤에는 다음 실행이 안전하게 stale 검증을 수행한다.
- [ ] `O_NONBLOCK`, `FD_CLOEXEC` 설정 실패가 무시되지 않는다.

### 구현 후 설명할 것

1. 경로가 단순 문자열이 아니라 보안 경계인 이유.
2. `lstat`·UID·mode 검증 각각이 막는 공격 또는 사고.
3. stale socket 자동 제거와 보수적 실패의 trade-off.
4. 생성·bind·등록·cleanup의 자원 획득 순서.
5. 비정상 종료까지 고려한 수명 관리 전략.

### 원본 확인 위치

- Thread 19 — Unix 데이터그램 엔드포인트의 안전한 수명
- 커밋: `feat(runtime): 안전한 응답 endpoint 경로 생성`, `feat(client): datagram 응답 endpoint 수명주기 관리`, `feat(server): datagram 응답 endpoint 수명주기 관리`
- 파일: `src/response_channel.c`, `src/client.c`, `src/server.c`, `include/minitalk.h`
- 함수: `validate_runtime_dir`, `mt_runtime_dir`, `mt_response_path`, `cleanup_response_socket`, `set_nonblocking_close_on_exec`, `remove_stale_socket`, `bind_client_socket`
- 관련 Thread: 20, 21

---

<a id="p-03"></a>
## P-03. [Thread 20 / `feat(server): 획득 요청을 검증해 세션 소유권 예약`] 세션 예약·응답 상관관계·죽은 소유자 회수

### 면접 질문

Unix datagram으로 받은 READY·ACK가 "기대하던 서버가 현재 요청에 보낸 응답"이라는 것을 무엇으로 검증해야 합니까? source path, magic, server PID, kind, token을 일부만 확인하면 각각 어떤 잘못된 응답을 받아들일 수 있습니까?

꼬리 질문:

- 세션 획득 요청의 payload PID와 datagram source path가 일치해야 하는 이유는 무엇입니까?
- 현재 소유자가 다른 PID일 때 `kill(pid, 0)`의 `ESRCH`만 회수 조건으로 쓰고 다른 오류는 BUSY로 남겨야 하는 이유는 무엇입니까?
- 새 소유자로 예약한 뒤 READY 전송이 실패하면 소유권을 rollback해야 하는 이유는 무엇입니까?
- 이전 요청의 지연 ACK나 위조 source에서 온 올바른 token을 어떻게 거절합니까?
- 잘못된 datagram을 하나 받았다고 즉시 실패하지 않고 deadline까지 무시하는 이유는 무엇입니까?
- 매 반복마다 timeout을 원래 값으로 초기화하면 전체 대기 시간이 늘어나는 문제를 어떻게 막습니까?

### 30초 모범 답변

응답은 payload만 믿지 않고 datagram source path, protocol magic, server PID, 응답 kind, 요청 nonce 또는 bit sequence를 모두 기대값과 비교해야 합니다. 서버도 요청 PID에서 계산한 client path와 실제 source를 대조하고 같은 사용자의 socket인지 확인합니다. 다른 소유자가 살아 있으면 BUSY, `ESRCH`로 죽음이 확인될 때만 세션을 회수합니다. 새 예약 뒤 READY 전송이 실패하면 예약을 되돌리고, client는 monotonic deadline 안에서 malformed·forged·stale 응답을 무시합니다.

### 답변 핵심 키워드

source authentication, correlation tuple, magic, PID, kind, nonce/token, sequence, owner liveness, `ESRCH`, rollback reservation, monotonic deadline

### 백지 구현

#### 구현 목표

네트워크 I/O와 분리된 세 가지 순수 로직을 작성한다.

1. 세션 획득 요청의 source와 payload 검증
2. 현재 소유권에 대한 예약 결정
3. 수신 응답이 현재 기대값과 일치하는지 판정

#### 인터페이스

```c
typedef struct s_request
{
	uint32_t	magic;
	uint32_t	kind;
	uint32_t	nonce;
	pid_t		client_pid;
}   t_request;

typedef struct s_response
{
	uint32_t	magic;
	uint32_t	kind;
	uint32_t	token;
	int32_t		status;
	pid_t		server_pid;
}   t_response;

typedef enum e_reservation
{
	RESERVATION_INVALID = -1,
	RESERVATION_BUSY = 0,
	RESERVATION_GRANTED = 1,
	RESERVATION_RECLAIMED = 2
}   t_reservation;

int	validate_acquire_request(const t_request *request,
		const char *source_path, const char *expected_client_path);
t_reservation	decide_reservation(pid_t current_owner,
		pid_t requester, int owner_probe_result, int owner_probe_errno);
int	response_matches(const t_response *response,
		const char *source_path, const char *expected_server_path,
		pid_t server_pid, uint32_t kind, uint32_t token);
```

#### 반드시 만족해야 할 조건

- 요청은 정확한 magic·acquire kind·유효 PID를 가져야 한다.
- source path는 요청 PID로 계산한 expected client path와 정확히 같아야 한다.
- 소유자가 없거나 같은 PID면 허용한다.
- 다른 소유자 probe가 `-1/ESRCH`일 때만 회수한다.
- probe 성공 또는 `ESRCH` 이외 오류에서는 다른 requester를 BUSY로 처리한다.
- 응답은 source path, magic, server PID, kind, token을 모두 일치시켜야 한다.
- status의 허용 값도 별도로 검증한다.
- pure validator는 전역 상태를 직접 변경하지 않는다.

#### 경계 조건

- 현재 소유자 없음
- 같은 client의 재요청
- 다른 살아 있는 소유자
- 죽은 소유자
- probe 권한 오류
- 잘못된 magic·kind·PID
- payload PID는 맞지만 source path가 다른 경우
- source는 맞지만 stale token·sequence인 경우
- 모든 필드는 맞지만 status가 미지원 값인 경우

#### 실패 조건과 제약

- path는 NUL 종료 및 길이 검증을 끝낸 값이라고 가정하거나 함수에서 방어한다.
- 실제 `recvfrom`, `kill`, clock 대기는 문제 범위 밖이다.
- 예약을 실제 상태에 commit하는 시점과 READY 전송 실패 rollback은 호출자가 구현한다.

```c
int	validate_acquire_request(const t_request *request,
		const char *source_path, const char *expected_client_path)
{
	// 직접 구현
}

t_reservation	decide_reservation(pid_t current_owner,
		pid_t requester, int owner_probe_result, int owner_probe_errno)
{
	// 직접 구현
}

int	response_matches(const t_response *response,
		const char *source_path, const char *expected_server_path,
		pid_t server_pid, uint32_t kind, uint32_t token)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 검증 tuple의 각 필드를 하나씩 틀리게 했을 때 모두 거절된다.
- [ ] source path가 다르면 payload가 완벽해도 거절된다.
- [ ] token이 이전 요청 값이면 거절된다.
- [ ] 같은 소유자의 재요청과 다른 소유자의 요청을 구분한다.
- [ ] `ESRCH`만 회수 조건이고 다른 probe 오류는 회수하지 않는다.
- [ ] 새 소유권을 commit한 뒤 READY 전송 실패 시 호출자가 rollback할 수 있는 반환 정보를 갖는다.
- [ ] malformed datagram이 이후의 유효 응답을 기다릴 기회를 없애지 않는다.
- [ ] deadline 계산이 monotonic clock 기준이며 invalid 응답마다 연장되지 않는다.
- [ ] 세션 reset 시 bit 조립 상태와 sequence도 함께 초기화된다.
- [ ] validation 함수가 전역 세션 상태를 부분 변경하지 않는다.

### 구현 후 설명할 것

1. 상관관계 tuple의 각 필드가 막는 오류·위조 유형.
2. PID 생존 확인의 한계와 `ESRCH`만 회수 조건으로 둔 이유.
3. 예약 상태 commit과 READY 전송의 rollback 경계.
4. malformed 응답 무시와 즉시 실패의 trade-off.
5. relative timeout을 반복 초기화하지 않고 absolute deadline을 쓰는 이유.

### 원본 확인 위치

- Thread 20 — 세션 예약·응답 상관관계·소유자 회수
- 커밋: `feat(server): 획득 요청을 검증해 세션 소유권 예약`
- 파일: `include/minitalk.h`, `src/server.c`, `src/client.c`, `tests/session_ownership.sh`, `tests/response_validation.sh`, `tests/response_server.c`
- 함수·구조: `t_mt_request`, `t_mt_response`, `valid_client_socket`, `valid_request_source`, `read_session_request`, `handle_session_request`, `valid_source`, `read_response`, `wait_for_response`, `request_session`
- 관련 Thread: 17, 19, 21, 22

---

<a id="p-04"></a>
## P-04. [Thread 21 / `refactor(server): signal 처리를 self-pipe event loop로 제한`] self-pipe 기반 async-signal-safe 경계

### 면접 질문

signal handler에서 bit 조립, stdout 출력, Unix datagram ACK, 세션 reset을 모두 수행하면 왜 위험합니까? handler가 고정 크기 이벤트를 nonblocking pipe에 기록하고 event loop가 실제 처리를 맡는 self-pipe 구조는 어떤 안전 경계를 만듭니까?

꼬리 질문:

- handler에서 호출 가능한 함수가 제한되는 이유는 무엇입니까?
- handler가 진입 전 `errno`를 저장하고 복원해야 하는 이유는 무엇입니까?
- 이벤트 크기가 `PIPE_BUF` 이하인지 확인하는 이유는 무엇입니까?
- pipe write end를 blocking으로 두면 handler에서 어떤 deadlock이 생길 수 있습니까?
- nonblocking write가 `EAGAIN`으로 실패했을 때 이벤트를 조용히 버리지 않고 fail-stop하는 이유는 무엇입니까?
- 종료 signal도 같은 pipe를 통해 event loop cleanup 경로로 보내면 어떤 장점이 있습니까?
- 상속된 signal mask가 차단된 채 실행되면 handler 설치만으로 충분하지 않은 이유는 무엇입니까?

### 30초 모범 답변

signal handler는 비동기적으로 일반 코드 사이에 끼어들기 때문에 malloc, stdio, 복잡한 전역 상태 전이는 재진입 안전하지 않습니다. handler에서는 `errno`를 보존하고 sender PID와 signal 번호를 고정 크기 이벤트로 만들어 nonblocking pipe에 한 번 쓰는 작업만 합니다. event loop가 pipe를 읽어 bit 상태, 출력, socket 응답을 순차 처리합니다. pipe가 가득 차 이벤트를 기록하지 못하면 protocol 순서가 깨졌으므로 overflow flag를 세우고 서버를 fail-stop합니다.

### 답변 핵심 키워드

async-signal-safe, self-pipe, fixed-size event, `PIPE_BUF`, nonblocking handler, saved `errno`, event loop serialization, overflow fail-stop, cleanup path

### 백지 구현

#### 구현 목표

signal handler는 이벤트 enqueue만 하고, 일반 함수가 이벤트 한 개를 완전히 읽도록 경계를 구현한다.

#### 인터페이스

```c
typedef struct s_signal_event
{
	pid_t	sender;
	int		signal_number;
}   t_signal_event;

extern int				g_event_pipe[2];
extern volatile sig_atomic_t	g_event_overflow;

void	signal_event_handler(int signal_number, siginfo_t *info, void *context);
int	read_signal_event(int fd, t_signal_event *event);
```

#### 반드시 만족해야 할 조건

- handler는 `errno`를 저장하고 반환 전에 복원한다.
- handler는 동적 할당, stdio, socket send, 복잡한 상태 전이를 하지 않는다.
- `info == NULL`인 경우 sender를 안전한 기본값으로 기록한다.
- 이벤트 전체를 한 번의 write 요청으로 enqueue한다.
- write 결과가 이벤트 크기와 다르면 `g_event_overflow`를 설정한다.
- write end는 초기화 단계에서 nonblocking이어야 한다.
- event 구조 크기가 atomic pipe write 한계를 넘지 않는다는 전제를 검증한다.
- reader는 `EINTR`를 재시도하고 이벤트 구조 전체를 채운다.
- event loop는 overflow flag를 발견하면 정상 처리를 계속하지 않는다.

#### 경계 조건

- `info == NULL`
- 정상 한 이벤트
- 여러 signal의 연속 enqueue
- handler write가 `EAGAIN`
- handler write가 partial로 관측되는 fault injection
- reader가 `EINTR`
- reader가 구조 일부만 반환하는 fault injection
- 종료 signal 이벤트
- invalid signal 번호 이벤트

#### 실패 조건과 제약

- handler에서는 오류 메시지 출력이나 프로세스 정리를 수행하지 않는다.
- overflow flag는 `sig_atomic_t`로 표현 가능한 단순 상태여야 한다.
- 이벤트 처리 순서가 깨진 뒤 복구를 시도하지 않고 상위 event loop에 실패를 알린다.

```c
void	signal_event_handler(int signal_number, siginfo_t *info, void *context)
{
	// 직접 구현
}

int	read_signal_event(int fd, t_signal_event *event)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] handler가 `errno` 값을 바꾸지 않는다.
- [ ] handler 안에서 허용하지 않은 함수가 호출되지 않는다.
- [ ] 정상 signal에서 sender와 signal 번호가 정확히 전달된다.
- [ ] pipe가 가득 찼을 때 handler가 block하지 않는다.
- [ ] enqueue 실패가 overflow flag로 남는다.
- [ ] overflow 뒤 event loop가 오류 종료와 endpoint cleanup 경로로 간다.
- [ ] 여러 이벤트가 바이트 단위로 섞이지 않는다.
- [ ] reader의 `EINTR`·partial read에서 이벤트 경계가 유지된다.
- [ ] 종료 signal이 직접 cleanup하지 않고 event loop가 종료 상태를 반환하게 한다.
- [ ] 초기 signal mask가 차단된 실행 환경에서도 필요한 signal을 명시적으로 해제한다.

### 구현 후 설명할 것

1. handler와 event loop의 책임을 어디서 나눴는가.
2. fixed-size event와 `PIPE_BUF`가 주는 원자성 전제.
3. nonblocking pipe와 overflow fail-stop의 trade-off.
4. `volatile sig_atomic_t`가 보장하는 것과 보장하지 않는 것.
5. 종료 signal을 일반 이벤트처럼 처리해 cleanup을 일원화한 이유.

### 원본 확인 위치

- Thread 21 — Self-Pipe 기반 시그널 안전 경계
- 커밋: `refactor(server): 비트 상태 전이 로직 추출`, `refactor(server): signal 처리를 self-pipe event loop로 제한`, `fix(server): 종료 시그널을 이벤트 루프 정리 경로로 처리`, `test(server): self-pipe 이벤트 손실 시 fail-stop 검증`, `fix(server): 상속된 이벤트 시그널 마스크 해제`
- 파일: `src/server.c`, `tests/protocol_regressions.sh`, `tests/write_fault.c`, `tests/write_fault.h`
- 함수·구조: `t_bit_event`, `handle_bit`, `read_event`, `process_bit`, `run_event_loop`, `unblock_event_signals`
- 관련 Thread: 17, 20, 22

---

<a id="p-05"></a>
## P-05. [Thread 22 / `fix(server): stdout 실패 뒤 ACK 전송 차단`] 출력 완료와 ACK의 커밋 경계

### 면접 질문

서버가 한 byte를 stdout에 기록한 뒤 client에 ACK를 보냅니다. stdout 쓰기가 실패했는데도 ACK를 보내면 protocol 관점에서 어떤 정합성 문제가 생깁니까? 일반 byte, NUL 종료 newline, 이전 client의 미완성 줄을 닫는 recovery newline에서 성공 응답의 commit 지점을 어디에 둬야 합니까?

꼬리 질문:

- stdout에 일부 prefix만 기록된 뒤 실패하면 ACK를 보내지 않는 것만으로 충분합니까?
- client가 ACK timeout으로 실패했지만 서버 stdout에는 일부 데이터가 남는 경우를 어떻게 설명합니까?
- NUL byte 자체보다 그에 대응하는 newline 출력 완료가 먼저여야 하는 이유는 무엇입니까?
- 새 세션을 받기 전에 이전 partial line을 닫는 newline이 실패하면 새 소유자에게 READY를 보내도 됩니까?
- 출력과 ACK를 진짜 원자적 트랜잭션으로 만들 수 없는 환경에서 최소한 어떤 순서를 보장해야 합니까?

### 30초 모범 답변

ACK는 서버가 해당 bit의 외부 효과를 성공적으로 반영했다는 의미여야 합니다. 따라서 일반 byte는 stdout 전체 쓰기가 완료된 뒤, NUL은 종료 newline이 완료된 뒤에만 ACK를 보냅니다. 이전 partial line을 정리하는 newline이 실패하면 새 세션도 성공으로 예약하면 안 됩니다. 출력과 datagram을 원자적으로 묶을 수는 없지만 최소한 "효과 완료 후 ACK" 순서를 지켜 거짓 성공을 막고, 실패 시 event loop를 중단해 더 이상 진행하지 않습니다.

### 답변 핵심 키워드

commit point, effect before acknowledgement, no false success, partial external effect, fail-stop, terminating newline, recovery newline, at-least-once ambiguity

### 백지 구현

#### 구현 목표

완성된 byte를 외부 출력에 반영하고, 성공한 경우에만 ACK callback을 호출하는 처리 함수를 작성한다.

#### 인터페이스

```c
typedef struct s_session_state
{
	pid_t		owner;
	uint32_t	sequence;
	int			line_started;
}   t_session_state;

int	process_completed_byte(
	t_session_state *session,
	unsigned char byte,
	int (*write_effect)(const void *buffer, size_t size),
	int (*send_ack)(pid_t client, uint32_t sequence)
);
```

#### 입력과 출력

- `write_effect`: 전체 출력 성공 시 0, 실패 시 -1
- `send_ack`: ACK 전송 성공 시 0, 실패 시 -1
- 처리 성공 시 0, 어느 단계든 실패하면 -1

#### 반드시 만족해야 할 조건

- 일반 byte는 해당 byte 출력 성공 뒤 ACK한다.
- NUL은 newline 출력 성공 뒤 ACK하고 세션 종료 상태를 commit한다.
- 출력 실패 시 ACK callback을 호출하지 않는다.
- ACK 실패 시 이미 완료된 출력은 되돌리지 못하며 함수는 실패한다.
- sequence는 어떤 시점에 증가하는지 명확한 commit 규칙을 가진다.
- 실패 뒤 상위 event loop가 계속 다음 bit를 처리하지 않도록 실패를 전파한다.
- 이전 partial line recovery는 별도 함수로 두더라도 같은 "newline 성공 후 상태 변경" 규칙을 적용한다.

#### 경계 조건

- 첫 일반 byte
- 여러 일반 byte
- 빈 메시지의 첫 NUL
- 일반 byte 뒤 NUL
- 일반 byte 출력 실패
- newline 출력 실패
- 출력 성공 뒤 ACK 실패
- recovery newline 실패
- 세션 owner가 없는 상태에서 호출
- sequence 최대값 정책

#### 실패 조건과 제약

- 출력과 ACK를 하나의 OS 원자 연산으로 묶을 수 없다고 가정한다.
- partial write 처리는 `write_effect` 내부에서 완료한다.
- retry·deduplication protocol은 문제 범위 밖이지만 남는 ambiguity를 설명한다.

```c
int	process_completed_byte(
	t_session_state *session,
	unsigned char byte,
	int (*write_effect)(const void *buffer, size_t size),
	int (*send_ack)(pid_t client, uint32_t sequence))
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 정상 byte에서 callback 호출 순서가 output → ACK다.
- [ ] 정상 NUL에서 newline → ACK → 세션 종료 상태 순서가 계약과 일치한다.
- [ ] byte 출력 실패 시 ACK 호출 횟수가 0이다.
- [ ] newline 실패 시 종료 ACK가 전송되지 않는다.
- [ ] recovery newline 실패 시 새 세션 READY로 진행하지 않는다.
- [ ] 출력 성공·ACK 실패에서 외부 출력은 남지만 함수와 event loop는 실패한다.
- [ ] sequence가 실패 경로에서 중복 증가하거나 건너뛰지 않는다.
- [ ] line_started와 owner reset이 출력 성공 전에 일어나지 않는다.
- [ ] fault injection으로 byte·newline·partial write 위치를 각각 실패시켰다.
- [ ] client가 거짓 성공을 관측하지 않는다는 protocol invariant를 확인했다.

### 구현 후 설명할 것

1. 성공 응답의 의미를 어떤 외부 효과에 연결했는가.
2. 출력과 ACK 사이에 남는 failure window.
3. ACK 전송 실패 뒤 재시도·중복 처리 가능성을 어떻게 볼 것인가.
4. fail-stop을 선택한 이유와 계속 처리할 때 생기는 위험.
5. partial line recovery도 같은 commit 규칙에 포함한 이유.

### 원본 확인 위치

- Thread 22 — 출력 완료와 ACK 실패 경계
- 커밋: `fix(server): stdout 실패 뒤 ACK 전송 차단`, `test(server): 회수 줄바꿈 출력 실패 검증`
- 파일: `src/write_utils.c`, `src/server.c`, `tests/output_failure.sh`, `tests/write_fault.c`, `tests/write_fault.h`
- 함수: `mt_write_all`, `reset_session`, `flush_byte`, `process_bit`, `send_response`
- 관련 Thread: 6, 12, 20, 21
