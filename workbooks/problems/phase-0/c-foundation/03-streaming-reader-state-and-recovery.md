# 스트리밍 reader 상태·수명·복구 워크북

이 문서는 줄 단위 reader를 단순 문자열 함수가 아니라 **지속 상태를 가진 스트리밍 컴포넌트**로 다룬다. 버퍼 cursor, EOF, FD 수명, 비차단 I/O 결과를 서로 섞지 않는 것이 핵심이다.

---

<a id="r-01"></a>
## R-01. [Thread 13 / `feat(reader): 명시적 결과 상태 API 추가`] 줄 추출 상태 머신과 결과 의미

### 면접 질문

줄 reader가 `char *` 하나만 반환하면 빈 줄, 마지막 줄, EOF, 할당 실패, 읽기 오류를 어떻게 구분해야 합니까? 별도의 결과 enum과 out parameter를 사용한다면 각 결과에서 `*line`과 내부 cursor는 어떤 상태여야 합니까?

꼬리 질문:

- 개행을 포함한 줄을 반환한 뒤 남은 입력을 어떻게 보존합니까?
- 파일이 개행 없이 끝났을 때 마지막 바이트들을 한 줄로 반환하고, 다음 호출에서 EOF를 반환하려면 어떤 상태가 필요합니까?
- EOF를 한 번 반환한 뒤 다시 호출해도 안정적으로 EOF를 반환하게 하려면 무엇을 기록해야 합니까?
- 줄 메모리 할당이 실패했을 때 이미 찾은 개행 위치를 소비해도 됩니까?
- `begin`, `scan`, `end`, `capacity` cursor 사이에 어떤 invariant를 둘 수 있습니까?

### 30초 모범 답변

반환 포인터만으로는 EOF와 오류를 구분할 수 없으므로 LINE, EOF, ERROR 같은 결과 enum을 두고 줄은 out parameter로 전달합니다. LINE일 때만 호출자가 해제할 새 문자열을 주고, 나머지 결과에서는 `*line`을 NULL로 둡니다. 버퍼는 소비 시작점, 다음 검색점, 유효 데이터 끝을 따로 유지해 이미 검사한 바이트를 다시 스캔하지 않습니다. 할당 실패에서는 소비 cursor를 전진시키지 않아 같은 입력으로 재시도할 수 있어야 합니다.

### 답변 핵심 키워드

explicit result enum, out parameter, LINE/EOF/ERROR, stable EOF, final unterminated line, cursor invariant, no consume before allocation succeeds

### 백지 구현

#### 구현 목표

동적 버퍼에 누적된 입력에서 한 줄씩 반환하는 상태 기반 reader를 작성한다. 이 문제에서는 blocking FD를 가정하고 `EINTR`·`EAGAIN` 처리는 R-04에서 별도로 구현한다.

#### 인터페이스

```c
typedef enum e_reader_result
{
	READER_ERROR = -1,
	READER_EOF = 0,
	READER_LINE = 1
}   t_reader_result;

typedef struct s_reader
{
	int		fd;
	char	*bytes;
	size_t	begin;
	size_t	scan;
	size_t	end;
	size_t	capacity;
	int		reached_eof;
}   t_reader;

t_reader_result	reader_next(t_reader *reader, char **line);
```

#### 입력과 출력

- LINE: `*line`은 새로 할당된 한 줄이며 호출자가 해제한다.
- EOF·ERROR: `*line == NULL`
- 반환한 줄은 개행이 입력에 있었다면 개행을 포함한다.

#### 반드시 만족해야 할 조건

- 호출 시작 시 가능한 경우 `*line`을 NULL로 초기화한다.
- 이미 버퍼에 완전한 줄이 있으면 추가 read 없이 반환한다.
- 개행 전까지 여러 read 결과를 누적할 수 있다.
- EOF 때 남은 바이트가 있으면 마지막 줄로 한 번 반환한다.
- 남은 바이트가 없는 EOF는 반복 호출해도 안정적인 EOF다.
- 줄 할당이 실패하면 해당 줄을 소비하지 않는다.
- 정상 상태에서 `begin <= scan <= end`이며 유효 저장소 범위를 벗어나지 않는다.

#### 경계 조건

- 빈 파일
- 첫 바이트가 개행
- 연속 빈 줄
- 한 read 안에 여러 줄
- 한 줄이 여러 read에 걸침
- 마지막 줄에 개행이 없음
- 마지막 바이트가 개행
- 줄 길이 0, 1, 버퍼 경계 전후
- 줄 할당 실패 뒤 재호출

#### 실패 조건과 제약

- `reader == NULL` 또는 `line == NULL`
- read 오류
- 버퍼 확장 실패
- 결과 줄 할당 실패
- 한 번에 전체 파일을 별도 문자열로 읽지 않는다.

```c
t_reader_result	reader_next(t_reader *reader, char **line)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 빈 파일은 첫 호출부터 EOF이며 line은 NULL이다.
- [ ] `"\n\nx\n"`을 세 줄로 정확히 분리한다.
- [ ] 한 read에 여러 줄이 들어와도 다음 호출이 남은 줄을 반환한다.
- [ ] 여러 read에 걸친 긴 줄을 누락·중복 없이 반환한다.
- [ ] 개행 없는 마지막 줄을 한 번 반환한 뒤 EOF가 안정적이다.
- [ ] LINE 이외 결과에서 line이 이전 호출의 포인터를 유지하지 않는다.
- [ ] 줄 할당 실패 뒤 `begin`이 전진하지 않고 같은 줄을 다시 얻을 수 있다.
- [ ] 모든 상태 전이 뒤 `begin <= scan <= end <= capacity` 성질을 확인했다.
- [ ] 소비가 끝난 버퍼를 정리해도 아직 읽지 않은 바이트는 보존된다.
- [ ] 각 바이트의 검색 횟수가 불필요하게 반복되지 않는다.

### 구현 후 설명할 것

1. 포인터 반환 대신 enum과 out parameter를 선택한 이유.
2. `begin`, `scan`, `end`를 하나나 둘의 cursor로 합치지 않은 이유.
3. EOF 플래그와 read 결과 0을 구분해 저장하는 방법.
4. 할당 성공 전에는 상태를 commit하지 않는 이유.
5. legacy `get_next_line` 형태와 명시적 reader API의 표현력 차이.

### 원본 확인 위치

- Thread 13 — 줄 추출 상태 머신과 결과 의미
- 커밋: `feat(reader): 줄을 분리하고 남은 입력 보존`, `feat(reader): 명시적 결과 상태 API 추가`
- 파일: `get_next_line.c`, `get_next_line.h`, `tests/test_reader.c`
- 함수·구조: `t_blr_reader`, `t_blr_result`, `blr_reader_next`, `find_line_end`, `extract_line`, `unread_length`
- 관련 Thread: 14, 15, 16

---

<a id="r-02"></a>
## R-02. [Thread 14 / `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용`] 구간 버퍼·기하 증가·선형 스캔 비용

### 면접 질문

긴 줄을 읽을 때 매 read마다 기존 문자열 길이를 다시 찾고 정확한 새 크기로 재할당하면 왜 O(n²) 복사나 재스캔이 발생할 수 있습니까? `begin`, `scan`, `end`, `capacity` 구간과 기하 증가 전략으로 비용을 어떻게 선형에 가깝게 유지합니까?

꼬리 질문:

- 소비한 prefix를 매 줄마다 즉시 `memmove`하는 방식은 어떤 입력에서 비싸집니까?
- `scan` cursor를 `begin`으로 되돌려야 하는 경우와 유지해야 하는 경우는 언제입니까?
- capacity를 두 배로 키울 때 `size_t` overflow는 어떻게 판정합니까?
- 필요한 공간이 현재 capacity보다 클 때 정확한 크기로 한 번만 키우는 것과 기하 증가의 trade-off는 무엇입니까?
- 벽시계 시간보다 read·allocation·copy 횟수를 측정하는 테스트가 더 안정적인 이유는 무엇입니까?

### 30초 모범 답변

매번 정확한 크기로 새 버퍼를 만들면 지금까지 읽은 모든 바이트를 반복 복사해 총 O(n²)이 될 수 있습니다. capacity를 기하급수적으로 키우면 재할당 횟수가 로그 수준으로 줄고 전체 복사량은 누적 입력의 상수배가 됩니다. 검색 cursor는 이미 확인한 구간 뒤에서 계속 시작해 같은 바이트를 다시 스캔하지 않습니다. 소비한 prefix는 읽기 공간이 실제로 필요할 때만 압축하거나 새 할당 시 함께 정리합니다.

### 답변 핵심 키워드

amortized growth, geometric capacity, monotonic scan cursor, segment buffer, lazy compaction, overflow check, operation-count metrics

### 백지 구현

#### 구현 목표

누적 입력 버퍼의 공간 확보와 새 개행 검색을 담당하는 두 helper를 작성한다.

#### 인터페이스

```c
typedef struct s_buffer
{
	char	*bytes;
	size_t	begin;
	size_t	scan;
	size_t	end;
	size_t	capacity;
}   t_buffer;

int		buffer_reserve_append(t_buffer *buffer, size_t additional);
size_t	buffer_find_line_end(t_buffer *buffer);
```

#### 입력과 출력

- `buffer_reserve_append`: `additional`바이트와 필요한 sentinel 공간을 확보하면 1, 실패하면 0
- `buffer_find_line_end`: 개행 다음 위치를 반환하고, 없으면 0

#### 반드시 만족해야 할 조건

- 읽지 않은 구간 `[begin, end)`의 바이트 순서를 보존한다.
- 새 read가 쓸 수 있는 연속 공간을 만든다.
- 기존 capacity가 충분하면 할당하지 않는다.
- 성장 산술에서 overflow가 나면 원래 버퍼를 그대로 둔 채 실패한다.
- 검색은 `scan`부터 시작하고, 확인한 위치만큼 `scan`을 단조 증가시킨다.
- 개행을 찾으면 반환 위치는 개행 바로 다음이다.

#### 경계 조건

- 비어 있는 초기 버퍼
- `additional == 0`
- 소비한 prefix가 크고 tail은 작은 경우
- tail과 추가 공간이 현재 capacity에 정확히 맞는 경우
- 한 번의 압축만으로 충분한 경우
- 압축 뒤에도 성장해야 하는 경우
- capacity 0에서 첫 성장
- 최대값 근처에서 두 배 증가가 overflow하는 경우
- 개행이 `scan`, `end - 1`, 또는 없음

#### 실패 조건과 제약

- 할당 실패와 크기 overflow
- 실패 시 기존 포인터·cursor·데이터가 유효해야 한다.
- 매 호출마다 무조건 새 버퍼를 만들지 않는다.
- 검색 함수는 메모리를 할당하지 않는다.

```c
int	buffer_reserve_append(t_buffer *buffer, size_t additional)
{
	// 직접 구현
}

size_t	buffer_find_line_end(t_buffer *buffer)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 초기 append, 반복 append, prefix 소비 뒤 append가 모두 데이터를 보존한다.
- [ ] 충분한 capacity에서 allocation count가 증가하지 않는다.
- [ ] 기하 증가 과정에서 필요한 크기보다 작은 capacity가 선택되지 않는다.
- [ ] overflow·할당 실패 뒤 기존 상태가 변하지 않는다.
- [ ] 개행이 없을 때 같은 바이트를 다음 호출에서 다시 스캔하지 않는다.
- [ ] 새 바이트가 추가된 뒤 검색이 이전 `end`부터 이어진다.
- [ ] 압축 뒤 `begin`, `scan`, `end`가 일관된 새 좌표를 갖는다.
- [ ] 큰 단일 줄에서 총 할당 횟수가 입력 바이트 수에 비례하지 않는다.
- [ ] 총 복사량과 총 스캔량이 입력 크기의 상수배인지 계수 기반으로 확인했다.

### 구현 후 설명할 것

1. exact growth와 geometric growth의 메모리·복사 trade-off.
2. prefix를 즉시 지우지 않고 구간으로 남기는 이유.
3. 압축 시점을 어떤 조건으로 선택했는가.
4. 성장 실패에서 strong state preservation을 어떻게 지켰는가.
5. 성능 회귀를 벽시계 대신 operation count로 검증하는 이유.

### 원본 확인 위치

- Thread 14 — 구간 버퍼와 선형 스캔 비용
- 커밋: `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용`
- 파일: `get_next_line.c`, `tests/metrics/metric_runtime.c`, `tests/metrics/metric_runtime.h`, `tests/metrics/test_metrics.c`, `tests/manifests/metrics-4mib.txt`
- 함수: `reserve_bytes`, `copy_bytes`, `find_line_end`
- 관련 Thread: 13, 15

---

<a id="r-03"></a>
## R-03. [Thread 15 / `feat(context): 명시적 reader 수명 API 추가`] 숨은 FD 상태를 명시적 lifecycle로 바꾸기

### 면접 질문

FD 번호를 key로 한 전역 reader 목록은 여러 호출을 편하게 만들지만 어떤 수명·동시성 문제를 숨깁니까? `create`, `next`, `reset`, `destroy`를 가진 명시적 컨텍스트로 바꾸면 FD 소유권과 상태 재사용 계약을 어떻게 정의해야 합니까?

꼬리 질문:

- `destroy`가 내부 버퍼는 해제하지만 FD를 닫지 않는 설계의 장단점은 무엇입니까?
- 외부에서 `lseek`한 뒤 `reset`하지 않으면 왜 prefetched 데이터와 실제 FD offset이 어긋날 수 있습니까?
- 닫힌 FD 번호가 다른 파일에 재사용됐을 때 전역 map이 오래된 버퍼를 붙이는 문제를 어떻게 막습니까?
- `dup`으로 만든 두 FD가 같은 open file description을 공유한다는 사실이 reader 상태에 어떤 영향을 줍니까?
- 서로 다른 컨텍스트를 서로 다른 스레드에서 쓰는 것과 같은 컨텍스트를 동시에 쓰는 것은 왜 다른 문제입니까?

### 30초 모범 답변

전역 FD map은 번호 재사용, 외부 seek, 정리 시점, 동시 접근을 API 밖에 숨깁니다. 명시적 reader는 FD를 빌리고 내부 버퍼만 소유하도록 하며, destroy는 FD를 닫지 않고 reset은 누적 입력과 EOF 상태만 버립니다. 호출자는 외부 offset 변경 뒤 reset해야 하고, 같은 open file description을 공유하는 alias는 하나의 reader로 직렬화해야 합니다. 독립 컨텍스트끼리는 전역 가변 상태가 없어 병렬 사용이 가능합니다.

### 답변 핵심 키워드

explicit context, borrowed FD, owned buffer, reset vs destroy, descriptor reuse, open file description, `dup` alias, thread confinement

### 백지 구현

#### 구현 목표

FD를 소유하지 않는 reader 컨텍스트의 생성·reset·destroy 수명 함수를 작성한다. 줄 읽기 알고리즘은 제공된다고 가정한다.

#### 인터페이스

```c
typedef struct s_reader t_reader;

t_reader	*reader_create(int fd);
void		reader_reset(t_reader *reader);
void		reader_destroy(t_reader *reader);
```

#### 입력과 출력

- `reader_create`: 유효한 FD를 빌려 초기 상태를 만들고, 실패하면 `NULL`
- `reader_reset`: 내부 누적 데이터와 EOF 상태를 초기화
- `reader_destroy`: 내부 자원과 컨텍스트를 해제

#### 반드시 만족해야 할 조건

- create·reset·destroy 어느 함수도 FD를 닫지 않는다.
- create 성공 후 빈 버퍼, 초기 cursor, EOF 아님 상태다.
- reset 뒤 같은 컨텍스트를 다시 사용할 수 있다.
- destroy는 `NULL` 입력에도 안전하도록 할지 계약을 정한다.
- 부분 초기화 실패에서 할당된 자원을 모두 정리한다.
- 전역 reader 목록에 등록하지 않는다.

#### 경계 조건

- 음수 FD
- 유효성 확인용 0바이트 read가 `EINTR`인 경우를 지원할지 여부
- 첫 할당 실패
- 빈 상태 reset
- 여러 번 reset
- reset 뒤 재사용
- destroy 뒤 FD가 여전히 열려 있는지 확인
- FD 번호를 닫고 같은 번호로 다른 파일을 연 경우 새 컨텍스트 생성

#### 실패 조건과 제약

- invalid FD, 컨텍스트 할당 실패
- 같은 컨텍스트를 동시에 호출하는 동기화는 제공하지 않는다.
- FD alias 자동 탐지는 문제 범위 밖이며 사용 계약으로 제한한다.

```c
struct s_reader
{
	// FD와 내부 버퍼 상태를 직접 설계
};

t_reader	*reader_create(int fd)
{
	// 직접 구현
}

void	reader_reset(t_reader *reader)
{
	// 직접 구현
}

void	reader_destroy(t_reader *reader)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 생성 직후 모든 cursor와 EOF 상태가 초기값이다.
- [ ] 생성 실패 경로에 누수가 없다.
- [ ] reset가 버퍼를 해제하고 컨텍스트 자체와 FD는 유지한다.
- [ ] destroy 뒤에도 `fcntl(fd, F_GETFD)`가 성공한다.
- [ ] 외부 seek 후 reset하면 새 위치의 데이터를 읽을 수 있다.
- [ ] FD 번호 재사용 시 새 컨텍스트가 이전 버퍼를 보지 않는다.
- [ ] 서로 다른 네 컨텍스트를 병렬로 사용해도 전역 상태 충돌이 없다.
- [ ] 같은 컨텍스트의 동시 호출을 지원하지 않는다는 계약이 명확하다.
- [ ] `dup` alias는 별도 컨텍스트 두 개보다 하나의 컨텍스트 공유가 필요함을 설명할 수 있다.

### 구현 후 설명할 것

1. FD를 소유하지 않는 정책을 선택한 이유.
2. reset과 destroy의 상태·자원 차이.
3. 전역 map을 없애면서 legacy API를 유지한다면 adapter를 어디에 둘 것인가.
4. FD 번호와 open file description을 구분해야 하는 이유.
5. thread-safe와 reentrant를 이 API 문맥에서 어떻게 구분하는가.

### 원본 확인 위치

- Thread 15 — 숨은 FD 상태에서 명시적 컨텍스트 수명으로
- 커밋: `refactor(state): reader 상태를 helper 인자로 전달`, `feat(state): 디스크립터별 읽기 상태 분리`, `feat(context): 명시적 reader 수명 API 추가`, `test(thread): 독립 컨텍스트의 병렬 사용 검증`
- 파일: `get_next_line.c`, `get_next_line.h`, `tests/test_context.c`, `tests/failure/test_failure.c`, `tests/support/fault_runtime.c`, `tests/support/fault_runtime.h`
- 함수: `blr_reader_create`, `blr_reader_reset`, `blr_reader_destroy`, `blr_reader_next`
- 관련 Thread: 13, 14, 16

---

<a id="r-04"></a>
## R-04. [Thread 16 / `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존`] `EINTR`·`EAGAIN`·I/O 오류 뒤 상태 보존

### 면접 질문

비차단 FD에서 줄 일부를 읽은 뒤 `EAGAIN`이 발생했습니다. 이 결과를 EOF나 일반 오류로 처리하면 어떤 데이터 손실이 생기며, LINE·EOF·AGAIN·ERROR 네 상태에서 버퍼와 out parameter를 어떻게 유지해야 합니까?

꼬리 질문:

- `EINTR`와 `EAGAIN`은 둘 다 `read == -1`인데 왜 처리 방식이 다릅니까?
- 부분 입력을 읽은 뒤 `EIO`가 발생해 ERROR를 반환하더라도 다음 호출에서 복구를 허용할 수 있습니까?
- AGAIN 뒤 legacy `get_next_line` adapter가 컨텍스트를 폐기하면 어떤 문제가 생깁니까?
- read가 0을 반환한 EOF와 현재 읽을 데이터가 없는 AGAIN을 event loop에서 어떻게 다르게 사용합니까?
- 할당 실패와 I/O 오류에서 상태 보존 정책을 같게 할지 다르게 할지 어떻게 결정합니까?

### 30초 모범 답변

`EINTR`는 호출 자체가 중단된 것이므로 같은 요청을 재시도하고, `EAGAIN`은 현재 데이터가 준비되지 않았다는 정상적인 비차단 상태이므로 AGAIN을 반환합니다. 두 경우 모두 이미 누적한 바이트를 버리면 안 됩니다. EOF는 앞으로 데이터가 없다는 영구 상태이고, ERROR는 호출 실패를 알리되 이 구현에서는 누적 입력을 보존해 같은 컨텍스트로 재시도할 수 있습니다. LINE 이외 결과에서는 out line을 NULL로 둡니다.

### 답변 핵심 키워드

`EINTR` retry, `EAGAIN`/`EWOULDBLOCK`, nonblocking readiness, partial-state preservation, EOF permanence, retriable context, no destructive adapter cleanup

### 백지 구현

#### 구현 목표

비차단 FD를 가진 상태형 reader에서 한 줄을 얻거나 명시적 상태를 반환하는 함수를 작성한다.

#### 인터페이스

```c
typedef enum e_reader_result
{
	READER_ERROR = -1,
	READER_EOF = 0,
	READER_LINE = 1,
	READER_AGAIN = 2
}   t_reader_result;

t_reader_result	reader_next_nonblocking(t_reader *reader, char **line);
```

#### 입력과 출력

- LINE: 새 줄을 반환하고 해당 구간을 소비한다.
- AGAIN: 현재 완전한 줄이 없고 read가 대기 상태이며 `*line == NULL`
- EOF: 남은 최종 줄까지 모두 반환한 뒤의 영구 종료 상태
- ERROR: I/O 또는 할당 실패이며 `*line == NULL`

#### 반드시 만족해야 할 조건

- `EINTR`는 사용자에게 노출하지 않고 read를 재시도한다.
- `EAGAIN`과 `EWOULDBLOCK`은 AGAIN으로 변환한다.
- AGAIN·ERROR에서 이미 누적한 바이트와 검색 위치를 잃지 않는다.
- 완전한 줄이 이미 버퍼에 있으면 read 결과와 무관하게 먼저 반환할 수 있다.
- read 0은 EOF 상태를 기록한다.
- EOF 전에 남은 개행 없는 tail은 LINE으로 한 번 반환한다.
- LINE 이외 결과에서 line은 NULL이다.
- 줄 결과 할당 성공 전에 소비 cursor를 commit하지 않는다.

#### 경계 조건

- 첫 read부터 `EAGAIN`
- 일부 바이트 → `EAGAIN` → 나머지 바이트와 개행
- 연속 `EINTR` 뒤 성공
- 일부 바이트 → `EIO` → 다음 호출 성공
- 개행까지 읽은 직후 다음 read가 `EAGAIN`
- tail 뒤 EOF
- EOF 뒤 반복 호출
- line allocation 실패 뒤 재호출

#### 실패 조건과 제약

- 영구 read 오류, 버퍼 확장 실패, 줄 할당 실패
- ERROR를 반환하더라도 컨텍스트를 자동 destroy하지 않는다.
- readiness 대기는 함수 밖 event loop가 담당한다.
- busy loop를 만들지 않도록 AGAIN에서 즉시 제어를 돌려준다.

```c
t_reader_result	reader_next_nonblocking(t_reader *reader, char **line)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 첫 EAGAIN에서 AGAIN과 NULL line을 반환하며 상태가 비어 있다.
- [ ] partial → EAGAIN 뒤 다음 호출이 앞의 partial과 새 입력을 합쳐 한 줄을 만든다.
- [ ] `EINTR` 횟수만큼 재시도해도 중복 read 상태 갱신이 없다.
- [ ] partial → EIO에서 ERROR를 반환하되 partial이 보존된다.
- [ ] 오류가 제거된 다음 호출에서 보존된 입력으로 LINE을 완성한다.
- [ ] EOF와 AGAIN이 서로 바뀌지 않는다.
- [ ] EOF tail을 한 번만 반환하고 이후 EOF가 안정적이다.
- [ ] AGAIN 뒤 legacy adapter가 reader를 삭제하지 않도록 호출 경계를 확인했다.
- [ ] 모든 실패 경로에서 cursor invariant와 live allocation을 확인했다.
- [ ] event loop가 AGAIN일 때 FD readiness를 기다린 뒤 재호출할 수 있다.

### 구현 후 설명할 것

1. `EINTR`, `EAGAIN`, EOF, 영구 오류를 나눈 기준.
2. ERROR 뒤에도 누적 상태를 보존하는 정책의 장단점.
3. state mutation을 read 성공량과 line allocation 성공 시점에 어떻게 commit했는가.
4. AGAIN을 반환하는 API와 blocking wrapper를 함께 제공하는 방법.
5. 오류를 숨기지 않으면서 호출자가 reset·destroy·retry를 선택하게 한 이유.

### 원본 확인 위치

- Thread 16 — 중단·비차단 읽기의 재시도와 상태 보존
- 커밋: `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존`, `test(reader): 비차단 부분 입력 보존 검증`, `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증`
- 파일: `get_next_line.c`, `get_next_line.h`, `tests/test_nonblocking.c`, `tests/failure/test_failure.c`, `tests/support/fault_runtime.c`, `tests/support/fault_runtime.h`
- 함수·상태: `read_retrying`, `blr_reader_next`, `BLR_AGAIN`
- 관련 Thread: 13, 15
