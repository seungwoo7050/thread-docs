# C 메모리·소유권·I/O 워크북

이 문서는 저수준 C 코드에서 반복해서 드러난 다섯 역량을 묶는다. 각 문제는 원본 구현을 보지 않고 먼저 작성한 뒤, 마지막의 원본 확인 위치만 사용한다.

---

<a id="c-01"></a>
## C-01. [Thread 1 / `feat(memory): 겹치는 메모리의 안전한 이동 구현`] 겹치는 바이트 범위 이동

### 면접 질문

`memcpy`와 달리 `memmove`는 원본과 목적지 범위가 겹쳐도 원래 원본 바이트를 보존해야 합니다. 이 계약을 추가 메모리 없이 만족시키려면 어떤 경우에 앞에서 뒤로 복사하고, 어떤 경우에 뒤에서 앞으로 복사해야 합니까?

꼬리 질문:

- 목적지와 원본이 같은 주소인 경우는 어떻게 처리합니까?
- 두 범위가 맞닿기만 하고 겹치지 않는 경우는 어떻게 구분합니까?
- `length == 0`일 때 포인터가 `NULL`이어도 역참조가 발생하지 않게 하려면 무엇을 주의해야 합니까?
- 객체의 표현을 복사할 때 `char`보다 `unsigned char` 바이트 관점이 적절한 이유는 무엇입니까?

### 30초 모범 답변

겹치는 이동은 임시 배열에 원본을 복사한 뒤 목적지에 쓰는 것과 같은 결과를 내야 합니다. 목적지가 원본보다 앞에 있거나 두 범위가 겹치지 않으면 앞에서 복사해도 아직 읽지 않은 원본을 덮지 않습니다. 목적지가 원본 내부의 뒤쪽에서 시작하면 뒤에서 앞으로 복사해야 합니다. 길이가 0이면 주소 계산이나 역참조 없이 즉시 반환하고, 실제 이동은 바이트 단위로 처리합니다.

### 답변 핵심 키워드

`temporary-copy semantics`, overlap, copy direction, forward/backward, byte representation, zero length, no dereference

### 백지 구현

#### 구현 목표

추가 동적 할당 없이 겹치는 두 메모리 범위를 안전하게 이동하는 함수를 작성한다.

#### 인터페이스

```c
void	*move_bytes(void *destination, const void *source, size_t length);
```

#### 입력과 출력

- `destination`: 이동 결과를 쓸 첫 바이트
- `source`: 원본 첫 바이트
- `length`: 이동할 바이트 수
- 반환값: `destination`

#### 반드시 만족해야 할 조건

- 결과는 원본 `length`바이트를 별도 임시 저장소에 복사한 뒤 목적지에 쓴 것과 같아야 한다.
- 부분 겹침이 어느 방향으로 발생해도 아직 읽지 않은 원본 바이트가 손상되면 안 된다.
- `length == 0`이면 메모리를 읽거나 쓰지 않는다.
- `destination == source`여도 내용이 바뀌지 않는다.

#### 경계 조건

- 완전히 같은 범위
- 한 바이트 이동
- 서로 인접하지만 겹치지 않는 범위
- 목적지가 원본보다 한 바이트 앞에서 시작하는 겹침
- 목적지가 원본보다 한 바이트 뒤에서 시작하는 겹침
- 한 범위가 다른 범위를 완전히 포함하는 경우

#### 실패 조건과 제약

- 별도의 실패 반환은 없다.
- `length > 0`일 때 두 포인터는 각각 해당 범위에 접근 가능하다고 가정한다.
- `memcpy`, `memmove`, 동적 할당을 사용하지 않는다.

```c
void	*move_bytes(void *destination, const void *source, size_t length)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 비겹침 복사가 원본과 동일한 바이트열을 만든다.
- [ ] `buffer + 1`로 오른쪽 이동할 때 반복 패턴으로 오염되지 않는다.
- [ ] `buffer`로 왼쪽 이동할 때 원본 뒤쪽이 빠지거나 중복되지 않는다.
- [ ] 같은 주소, 0바이트, 1바이트가 모두 안전하다.
- [ ] 이동 범위 밖의 앞뒤 sentinel 바이트가 바뀌지 않는다.
- [ ] 반환 포인터가 정확히 `destination`이다.
- [ ] 시간 복잡도는 O(length), 추가 공간은 O(1)이다.

### 구현 후 설명할 것

1. 겹침 판정과 복사 방향을 어떤 조건으로 나눴는가.
2. 역방향 루프에서 unsigned 인덱스 underflow를 어떻게 피했는가.
3. 0길이 계약을 함수 초기에 분리한 이유는 무엇인가.
4. 바이트 표현용 포인터 타입을 선택한 이유는 무엇인가.
5. 주소 비교의 이식성 범위를 어떤 전제로 두었는가.

### 원본 확인 위치

- Thread 1 — 바이트 범위 복사·이동·검색
- 커밋: `feat(memory): 겹치는 메모리의 안전한 이동 구현`
- 파일: `src/memory/ft_memory_move.c`, `tests/test_memory_move.c`
- 함수: `ft_memmove`
- 관련 Thread: 2, 4, 6

---

<a id="c-02"></a>
## C-02. [Thread 2 / `feat(string): 문자열 길이 계산과 제한 복사·붙이기 추가`] capacity 제한 문자열 복사·붙이기

### 면접 질문

`strlcpy`와 `strlcat` 계열 함수는 실제로 쓴 바이트 수가 아니라 호출자가 필요했을 전체 길이를 반환합니다. 이 반환 계약이 왜 유용하며, `capacity == 0`이거나 목적지 안에 NUL이 없는 경우에는 각각 어떤 값을 반환해야 합니까?

꼬리 질문:

- `strlcpy`가 항상 NUL을 기록한다고 말할 수 없는 경우는 언제입니까?
- `strlcat`에서 목적지 길이를 `capacity`까지만 탐색해야 하는 이유는 무엇입니까?
- 덧셈식 조건보다 "남은 공간" 관점의 조건이 오버플로 방어에 유리한 이유는 무엇입니까?
- 반환값만으로 truncation을 어떻게 감지할 수 있습니까?

### 30초 모범 답변

제한 복사 함수는 버퍼 안에 실제로 들어간 길이와 별개로 완성하려던 문자열 길이를 반환하면 호출자가 재할당 크기와 잘림 여부를 판단할 수 있습니다. 복사는 capacity가 0이면 쓰지 않고 원본 길이만 반환하며, 양수면 최대 capacity-1바이트 뒤 NUL을 둡니다. 붙이기는 목적지에서 NUL을 capacity까지만 찾고, 그 안에 없으면 목적지 크기를 capacity로 간주해 capacity와 원본 길이의 합을 반환합니다.

### 답변 핵심 키워드

capacity, attempted length, truncation detection, NUL termination, bounded scan, unterminated destination, overflow-safe arithmetic

### 백지 구현

#### 구현 목표

버퍼 용량 계약을 지키는 문자열 복사와 붙이기 함수를 작성한다.

#### 인터페이스

```c
size_t	bounded_copy(char *destination, const char *source, size_t capacity);
size_t	bounded_append(char *destination, const char *source, size_t capacity);
```

#### 입력과 출력

- 입력 문자열은 NUL 종료 문자열이다.
- `capacity`는 목적지 객체의 전체 크기다.
- `bounded_copy`는 원본 길이를 반환한다.
- `bounded_append`는 원래 목적지 길이와 원본 길이의 합에 해당하는 계약값을 반환한다.

#### 반드시 만족해야 할 조건

- 목적지에는 최대 `capacity`바이트만 접근한다.
- `capacity > 0`이고 정상적으로 목적지 NUL을 찾은 경우 결과는 NUL 종료된다.
- 반환값은 실제 기록량이 아니라 잘리지 않았을 때 필요했던 길이를 표현한다.
- 붙이기는 목적지의 NUL을 `capacity` 바깥에서 찾지 않는다.

#### 경계 조건

- `capacity == 0`
- 빈 원본
- 빈 목적지
- capacity가 정확히 1
- 정확히 맞는 크기와 한 바이트 부족한 크기
- 목적지에 `capacity` 범위 안의 NUL이 없는 경우
- 매우 큰 길이에서 길이 합 산술이 표현 범위를 넘을 가능성

#### 실패 조건과 제약

- 별도의 오류 코드는 없다.
- 입력 포인터가 유효하다는 전제에서 동작한다.
- 표준 문자열 복사·붙이기 함수를 호출하지 않는다.
- 목적지와 원본의 겹침은 지원 계약에 포함하지 않는다.

```c
size_t	bounded_copy(char *destination, const char *source, size_t capacity)
{
	// 직접 구현
}

size_t	bounded_append(char *destination, const char *source, size_t capacity)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] capacity 0에서 목적지 sentinel이 전혀 바뀌지 않는다.
- [ ] capacity 1에서 결과가 빈 문자열이며 반환 길이는 원본 길이다.
- [ ] 정확히 맞는 경우와 잘리는 경우의 반환값이 동일한 계약을 따른다.
- [ ] 목적지 NUL이 capacity 안에 없을 때 범위 밖을 읽거나 쓰지 않는다.
- [ ] 목적지 끝의 NUL 위치가 정확하다.
- [ ] truncation 판정식이 copy와 append 각각 올바르다.
- [ ] 길이 합과 인덱스 조건에 unsigned wraparound 가능성이 없는지 검토했다.
- [ ] 시간 복잡도는 각 입력 길이에 선형이고 추가 공간은 O(1)이다.

### 구현 후 설명할 것

1. 반환값을 실제 쓴 길이가 아닌 "필요했던 길이"로 둔 이유.
2. capacity 0과 1을 일반 루프에 섞지 않고 다루는 방법.
3. NUL 없는 목적지에서 반환값과 쓰기 동작을 어떻게 정의했는가.
4. 오버플로 가능성이 있는 덧셈 조건을 어떻게 피했는가.
5. 이 API가 겹치는 입력을 지원하지 않는다는 계약을 어디에 둘 것인가.

### 원본 확인 위치

- Thread 2 — 용량 제한 문자열 탐색
- 커밋: `feat(string): 문자열 길이 계산과 제한 복사·붙이기 추가`
- 파일: `src/string/ft_string_bounds.c`, `tests/test_string_bounds.c`
- 함수: `ft_strlen`, `ft_strlcpy`, `ft_strlcat`
- 관련 Thread: 1, 10

---

<a id="c-03"></a>
## C-03. [Thread 4·5 / `test(alloc): 할당 실패와 rollback을 검증` · `feat(list): 실패 시 정리되는 리스트 변환 구현`] 부분 할당 실패와 소유권 rollback

### 면접 질문

문자열 분할 함수가 결과 포인터 배열과 여러 개의 필드 문자열을 순차적으로 할당한다고 가정하겠습니다. 중간 필드 할당이 실패했을 때 누수 없이 실패하려면 각 시점에 누가 무엇을 소유해야 합니까? 같은 원칙을 새 content와 새 노드를 만드는 연결 리스트 map에 적용하면 어떤 자원을 어떤 순서로 정리해야 합니까?

꼬리 질문:

- 아직 결과 배열에 연결하지 않은 새 객체와 이미 연결한 객체의 소유자는 어떻게 다릅니까?
- cleanup 함수에 "성공한 개수"를 넘기는 방식의 장점은 무엇입니까?
- content 변환은 성공했지만 노드 할당이 실패한 경우 누가 content를 해제해야 합니까?
- 콜백이 반환한 포인터가 원본 content와 같을 수도 있다면 소유권 계약을 어떻게 문서화해야 합니까?
- N번째 할당 실패 테스트가 정상 경로 테스트보다 잘 찾는 버그는 무엇입니까?

### 30초 모범 답변

다단계 할당에서는 자원이 만들어지는 즉시 단 하나의 명확한 소유자가 있어야 합니다. 결과 컨테이너에 연결되기 전에는 현재 함수가 소유하고, 연결된 뒤에는 컨테이너 cleanup이 소유합니다. 어느 단계에서 실패하든 아직 연결하지 않은 임시 자원을 먼저 해제하고, 이미 연결된 모든 자원을 역으로 정리한 뒤 실패를 반환해야 합니다. 리스트 map도 변환된 content와 노드 할당을 별도 단계로 보고 둘 사이 실패를 처리해야 합니다.

### 답변 핵심 키워드

single owner, ownership transfer, partial construction, rollback, cleanup helper, Nth-allocation failure, no leak, no double free

### 백지 구현

#### 구현 목표

구분자로 나눈 문자열 배열을 만들고, 어느 할당에서 실패해도 이미 만든 모든 결과를 해제한 뒤 `NULL`을 반환한다.

#### 인터페이스

```c
char	**split_fields(const char *text, char delimiter);
```

#### 입력과 출력

- `text`: NUL 종료 문자열
- `delimiter`: 한 바이트 구분자
- 성공: 각 필드를 새로 할당한 NUL 종료 포인터 배열
- 마지막 배열 원소는 `NULL`
- 실패: `NULL`

#### 반드시 만족해야 할 조건

- 연속 구분자는 빈 필드를 만들지 않는다.
- 선행·후행 구분자를 건너뛴다.
- 결과 배열과 각 필드는 호출자가 소유한다.
- 어떤 필드 할당이 실패해도 그 전에 성공한 필드와 배열을 모두 해제한다.
- 성공 결과의 각 문자열은 입력 저장소와 독립적이다.

#### 경계 조건

- 빈 문자열
- 구분자만 있는 문자열
- 구분자가 없는 문자열
- 한 글자 필드가 연속되는 문자열
- 매우 긴 단일 필드
- 필드 개수 계산 및 `count + 1`, `length + 1` 산술 경계

#### 실패 조건과 제약

- 입력이 `NULL`이면 실패한다.
- 결과 배열 할당 실패
- 첫 필드, 중간 필드, 마지막 필드 할당 실패
- 동적 배열 확장 대신 필요한 필드 수를 먼저 세는 2단계 접근을 사용할 수 있다.
- 정답 코드나 특정 cleanup 순서는 주어지지 않는다.

```c
static void	free_fields(char **fields, size_t initialized_count)
{
	// 직접 구현
}

char	**split_fields(const char *text, char delimiter)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 정상 입력에서 필드 순서와 내용이 정확하다.
- [ ] 빈 입력과 구분자만 있는 입력이 유효한 빈 결과를 만든다.
- [ ] 결과 배열이 항상 `NULL`로 끝난다.
- [ ] 배열 할당 실패에서 해제할 수 없는 포인터를 해제하지 않는다.
- [ ] 각 필드 할당 위치를 하나씩 실패시켜도 live allocation이 0이 된다.
- [ ] 같은 포인터를 두 번 해제하거나 초기화되지 않은 슬롯을 읽지 않는다.
- [ ] 성공 후 호출자가 필드와 배열을 정해진 방법으로 모두 해제할 수 있다.
- [ ] `count + 1`, 포인터 배열 크기 곱셈, 필드 `length + 1`의 오버플로를 검토했다.
- [ ] 리스트 map으로 바꾸어 생각했을 때 "변환 content 성공·노드 실패" 경로의 소유자도 설명할 수 있다.

### 구현 후 설명할 것

1. 자원이 생성되고 컨테이너에 연결되는 정확한 소유권 이전 시점.
2. cleanup helper가 개수 또는 NULL sentinel 중 무엇을 신뢰하는가.
3. 1-pass 동적 확장과 2-pass 사전 계수의 trade-off.
4. 콜백 기반 리스트 변환에서 content 해제 콜백이 필요한 이유.
5. N번째 실패 sweep을 어떻게 구성해야 모든 부분 상태를 통과하는가.

### 원본 확인 위치

- Thread 4 — 할당·콜백 기반 문자열 변환과 실패 정리
  - 커밋: `test(alloc): 할당 실패와 rollback을 검증`
  - 파일: `src/alloc/ft_allocate.c`, `src/string/ft_split.c`, `tests/failure/test_failure.c`, `tests/support/fail_alloc.c`
  - 함수: `ft_calloc`, `ft_split`, `count_fields`, `copy_field`, `free_fields`
- Thread 5 — 연결 리스트 구조와 소유권 수명주기
  - 커밋: `feat(list): 실패 시 정리되는 리스트 변환 구현`
  - 파일: `src/list/ft_list_map.c`, `src/list/ft_list_lifecycle.c`
  - 함수: `ft_lstmap`, `ft_lstclear`, `ft_lstdelone`
- 관련 Thread: 15, 28

---

<a id="c-04"></a>
## C-04. [Thread 6 / `fix(io): 파일 디스크립터 출력을 끝까지 재시도`] 완전 쓰기 루프

### 면접 질문

`write(fd, buffer, length)`가 성공하면 항상 `length`바이트가 기록된다고 가정하면 왜 잘못입니까? 요청한 전체 바이트를 기록하는 helper는 partial write, `EINTR`, 0바이트 반환, `SSIZE_MAX`보다 큰 길이를 각각 어떻게 다뤄야 합니까?

꼬리 질문:

- `EINTR`에서 같은 버퍼 전체가 아니라 남은 범위를 다시 요청해야 하는 경우는 언제입니까?
- 양수 partial write 뒤 영구 오류가 발생하면 이미 기록된 prefix를 되돌릴 수 있습니까?
- 쓰기 결과 0을 무한 재시도하면 어떤 문제가 생깁니까?
- `SIGPIPE`와 `EPIPE`의 관계는 무엇이며 helper 바깥에서 어떤 정책이 필요할 수 있습니까?
- `errno`를 helper의 공개 계약에 포함할지 어떻게 결정합니까?

### 30초 모범 답변

`write`는 signal, pipe·socket 상태, 커널 버퍼 여유 때문에 요청보다 적은 양만 기록할 수 있습니다. 양수 반환이면 커서와 남은 길이만 갱신하고 계속하며, `EINTR`는 진행량 없이 재시도합니다. 0은 진전이 없는 상태이므로 무한 루프 대신 실패로 처리하고, 다른 음수 오류는 즉시 전파합니다. 한 번의 요청 크기는 `ssize_t`가 표현할 수 있는 범위로 제한해야 합니다.

### 답변 핵심 키워드

partial write, progress cursor, remaining length, `EINTR`, zero progress, permanent error, `SSIZE_MAX`, `SIGPIPE`/`EPIPE`

### 백지 구현

#### 구현 목표

지정한 바이트 범위를 전부 기록하거나 실패하는 함수를 작성한다.

#### 인터페이스

```c
int	write_all(int fd, const void *buffer, size_t length);
```

#### 입력과 출력

- 성공 시 1
- 전체 기록 전에 실패하면 0
- 입력 버퍼의 소유권은 호출자에게 남는다.

#### 반드시 만족해야 할 조건

- 양수 partial write 뒤에는 이미 기록한 prefix를 다시 쓰지 않는다.
- `errno == EINTR`인 음수 반환은 재시도한다.
- 0바이트 반환은 진전 없는 영구 실패로 취급한다.
- 각 `write` 요청 크기는 반환형이 표현 가능한 범위를 넘지 않는다.
- 길이 0이면 시스템 호출 없이 성공한다.

#### 경계 조건

- 빈 출력
- 한 바이트 출력
- 여러 번의 1바이트 partial write
- partial write 다음 `EINTR`
- 여러 번의 `EINTR` 뒤 성공
- partial write 다음 `EPIPE` 또는 `EIO`
- 첫 호출에서 0바이트 반환
- 매우 큰 `size_t` 길이

#### 실패 조건과 제약

- 잘못된 FD, 닫힌 pipe, 영구 I/O 오류
- 이미 기록된 prefix를 rollback하지 않는다.
- `write` 외의 고수준 출력 함수를 사용하지 않는다.
- 0바이트 반환 시 필요하면 `errno`를 `EIO`로 설정하는 정책을 명시한다.

```c
int	write_all(int fd, const void *buffer, size_t length)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 한 번에 전부 쓰는 정상 경로가 성공한다.
- [ ] 여러 partial write 뒤 최종 출력이 입력과 정확히 같다.
- [ ] `EINTR` 뒤 동일한 남은 범위부터 이어 쓴다.
- [ ] partial prefix 뒤 영구 오류에서 prefix가 중복되지 않는다.
- [ ] 0바이트 반환에서 무한 루프가 발생하지 않는다.
- [ ] 길이 0에서 `write` 호출 수가 0이다.
- [ ] 버퍼 커서 증가와 남은 길이 감소에 overflow·underflow가 없다.
- [ ] 닫힌 pipe에서 프로세스 종료 정책과 함수의 오류 반환 정책을 구분했다.
- [ ] 시간 복잡도는 기록한 바이트 수와 시스템 호출 횟수에 비례하고 추가 공간은 O(1)이다.

### 구현 후 설명할 것

1. 양수, 0, `-1/EINTR`, `-1/기타` 네 결과를 나눈 이유.
2. 이미 기록된 prefix를 재시도하지 않도록 상태를 어떻게 표현했는가.
3. `SSIZE_MAX`보다 큰 남은 길이를 어떻게 나누는가.
4. 0바이트 반환의 오류 정책을 선택한 이유.
5. `SIGPIPE` 무시 정책이 helper 내부가 아니라 프로세스 초기화에 있을 수 있는 이유.

### 원본 확인 위치

- Thread 6 — 파일 디스크립터 출력 완결성
- 커밋: `fix(io): 파일 디스크립터 출력을 끝까지 재시도`
- 파일: `src/io/ft_fd_output.c`, `tests/failure/test_fd_output_failure.c`, `tests/support/fail_write.c`
- 함수: `write_all`, `ft_putchar_fd`, `ft_putstr_fd`, `ft_putendl_fd`, `ft_putnbr_fd`
- 관련 Thread: 12, 22, 28

---

<a id="c-05"></a>
## C-05. [Thread 12 / `fix(output): 쓰기 결과를 집계하기 전에 범위 검증`] 출력 컨텍스트와 sticky error

### 면접 질문

여러 변환 함수가 같은 출력 대상으로 연속해서 쓰는 포맷터에서 FD, 누적 출력 길이, 실패 상태를 하나의 컨텍스트에 두면 무엇이 좋아집니까? 한 번 실패한 뒤 후속 쓰기를 막는 sticky error와 `INT_MAX` 반환 길이 제한은 어떤 순서로 갱신해야 합니까?

꼬리 질문:

- 시스템 호출이 양수 값을 반환했더라도 누적 count에 더할 수 없는 경우가 있습니까?
- count overflow를 쓰기 전에 검사할 수 없는 이유와, 쓰기 뒤 검사할 때 생기는 외부 효과는 무엇입니까?
- 실패 뒤 formatter가 계속 변환 로직을 호출하더라도 추가 출력이 생기지 않게 하려면 어느 계층에서 차단해야 합니까?
- 한 글자씩 채우기와 고정 크기 chunk 채우기의 시스템 호출 비용 차이는 무엇입니까?
- 출력 스트림의 원자성을 보장하지 못하는 상황에서 반환값 계약은 무엇까지 말할 수 있습니까?

### 30초 모범 답변

출력 컨텍스트는 모든 변환기가 같은 FD, 누적 길이, 오류 상태를 공유하게 해 실패 전파를 단순화합니다. write helper가 partial write를 끝까지 처리하고, 각 양수 반환을 count에 더하기 전에 `written`과 기존 count의 `INT_MAX` 경계를 검사합니다. 한 번 오류가 나면 상태를 고정해 후속 함수가 즉시 실패하도록 하고, 최상위 함수는 count 대신 `-1`을 반환합니다. 반복 패딩은 작은 고정 버퍼로 묶어 호출 수를 줄일 수 있습니다.

### 답변 핵심 키워드

output context, sticky error, centralized write path, partial write, count overflow, `INT_MAX`, no further side effect, chunked padding

### 백지 구현

#### 구현 목표

공유 출력 컨텍스트를 초기화하고, 전체 바이트 쓰기와 누적 길이 집계를 수행하는 API를 작성한다.

#### 인터페이스

```c
typedef struct s_output
{
	int	fd;
	int	count;
	int	failed;
}   t_output;

void	output_init(t_output *output, int fd);
int		output_write(t_output *output, const void *buffer, size_t length);
int		output_repeat(t_output *output, unsigned char byte, int count);
```

#### 입력과 출력

- 개별 함수는 성공 시 0, 실패 시 -1을 반환한다.
- `output.count`는 실제로 성공 처리된 총 바이트 수다.
- `output.failed`가 설정되면 이후 호출은 시스템 호출 없이 실패한다.

#### 반드시 만족해야 할 조건

- 모든 실제 쓰기는 하나의 중앙 경로를 통과한다.
- partial write와 `EINTR`를 처리한다.
- 누적 길이가 `INT_MAX`를 넘을 수 있으면 실패 상태가 된다.
- 실패 뒤 count는 더 이상 증가하지 않는다.
- 반복 채움은 한 바이트마다 시스템 호출하지 않는다.

#### 경계 조건

- 길이 0
- 첫 쓰기 실패
- 일부 바이트 성공 뒤 영구 실패
- count가 `INT_MAX`에 정확히 도달하는 경우
- 다음 양수 반환을 더하면 overflow가 되는 경우
- 이미 failed인 컨텍스트에 다시 쓰는 경우
- repeat count가 0, 1, 내부 chunk 크기 경계 전후인 경우

#### 실패 조건과 제약

- 잘못된 인자, 시스템 호출 오류, 0바이트 쓰기, count 표현 범위 초과
- 이미 외부에 기록된 바이트는 rollback할 수 없다.
- 내부 반복 버퍼는 고정 크기여야 한다.

```c
void	output_init(t_output *output, int fd)
{
	// 직접 구현
}

int	output_write(t_output *output, const void *buffer, size_t length)
{
	// 직접 구현
}

int	output_repeat(t_output *output, unsigned char byte, int count)
{
	// 직접 구현
}
```

### 구현 후 자가 검증

- [ ] 정상 쓰기들의 count 합이 실제 출력 길이와 같다.
- [ ] partial write·`EINTR` 시퀀스에서도 바이트 누락·중복이 없다.
- [ ] 첫 영구 오류 뒤 추가 시스템 호출이 없다.
- [ ] count 경계 바로 아래, 정확히 경계, 경계 초과를 구분한다.
- [ ] `written` 값을 좁은 정수형으로 변환하기 전에 범위를 검증한다.
- [ ] repeat가 요청한 정확한 개수만 기록한다.
- [ ] repeat의 시스템 호출 수가 문자 수와 일대일로 증가하지 않는다.
- [ ] 실패 반환과 컨텍스트 상태가 서로 모순되지 않는다.
- [ ] 외부에 일부 출력된 뒤 `-1`을 반환할 수 있다는 계약을 설명할 수 있다.

### 구현 후 설명할 것

1. 오류 상태를 컨텍스트에 고정하는 방식이 호출자 코드를 어떻게 단순화하는가.
2. 시스템 호출 결과형과 공개 반환형의 범위가 다른 문제를 어떻게 처리했는가.
3. count 갱신과 실패 상태 갱신의 순서.
4. 반복 패딩 chunk 크기의 시간·스택 공간 trade-off.
5. 2단계 선측정과 결합할 때 count overflow 위험이 어떻게 줄어드는가.

### 원본 확인 위치

- Thread 12 — 출력 컨텍스트와 쓰기 복구
- 커밋: `fix(output): 쓰기 결과를 집계하기 전에 범위 검증`, `perf(output): 반복 채움을 묶어서 출력`, `test(output): 쓰기 실패 시퀀스와 채움 전략 검증`
- 파일: `src/ft_output.c`, `src/ft_printf.c`, `src/ft_printf_internal.h`, `tests/test_output_faults.c`
- 함수·구조: `t_printf`, `ft_printf_init`, `ft_printf_write`, `ft_printf_putchar`, `ft_printf_putnchar`
- 관련 Thread: 6, 8, 22
