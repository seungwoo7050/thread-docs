# C 메모리·소유권·I/O 워크북

이 문서는 저수준 C 코드에서 반복해서 드러난 다섯 역량을 묶는다. 각 문제는 원본 구현을 보지 않고 먼저 작성한 뒤, 마지막의 원본 확인 위치만 사용한다.

---

<a id="c-01"></a>
## C-01. [Thread 1 / `feat(memory): 겹치는 메모리의 안전한 이동 구현`] 겹치는 바이트 범위 이동

### 면접 질문

`memcpy`와 달리 `memmove`는 원본과 목적지 범위가 겹쳐도 원래 원본 바이트를 보존해야 합니다. 이 계약을 추가 메모리 없이 만족시키려면 어떤 경우에 앞에서 뒤로 복사하고, 어떤 경우에 뒤에서 앞으로 복사해야 합니까?

꼬리 질문:

- 목적지와 원본이 같은 주소인 경우는 어떻게 처리합니까?
  - 같은 주소이므로 복사가 필요 없습니다. 원본 구현처럼 즉시 `destination`을 반환하면 내용과 반환 계약을 모두 지킵니다.
- 두 범위가 맞닿기만 하고 겹치지 않는 경우는 어떻게 구분합니까?
  - 프로젝트 구현은 목적지가 `source + 1`부터 `source + length - 1` 사이와 같은지만 검사합니다. 따라서 `source + length`에서 시작하는 목적지는 겹침으로 보지 않고 순방향 복사합니다.
- `length == 0`일 때 포인터가 `NULL`이어도 역참조가 발생하지 않게 하려면 무엇을 주의해야 합니까?
  - 포인터 산술이나 바이트 접근보다 먼저 0길이를 검사해 반환해야 합니다. 일반 원칙상 0길이 호출은 메모리에 접근하지 않아야 합니다.
- 객체의 표현을 복사할 때 `char`보다 `unsigned char` 바이트 관점이 적절한 이유는 무엇입니까?
  - `unsigned char`는 객체 표현의 각 바이트를 값 손실이나 부호 해석 없이 다루는 타입입니다. 이 프로젝트도 두 포인터를 `unsigned char *` 계열로 바꿔 복사합니다.

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
	unsigned char		*destination_byte;
	const unsigned char	*source_byte;
	size_t				offset;

	destination_byte = destination;
	source_byte = source;
	if (length == 0 || destination_byte == source_byte)
		return (destination);
	/* 목적지가 아직 읽지 않은 원본 내부에서 시작할 때만 역방향이다. */
	offset = 1;
	while (offset < length && destination_byte != source_byte + offset)
		offset++;
	if (offset < length)
	{
		while (length > 0)
		{
			length--;
			destination_byte[length] = source_byte[length];
		}
		return (destination);
	}
	offset = 0;
	while (offset < length)
	{
		destination_byte[offset] = source_byte[offset];
		offset++;
	}
	return (destination);
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
   - 목적지가 원본의 두 번째 바이트부터 마지막 바이트 사이에서 시작하면 뒤에서 앞으로 복사하고, 그 밖에는 앞에서 뒤로 복사했습니다. 이는 원본 `ft_memmove`의 동일 주소 탐색 방식을 따른 것입니다.
2. 역방향 루프에서 unsigned 인덱스 underflow를 어떻게 피했는가.
   - `while (length > 0)`을 먼저 검사한 다음 `length--` 후 그 인덱스를 사용했습니다. 따라서 0에서 감소한 값을 배열 인덱스로 쓰지 않습니다.
3. 0길이 계약을 함수 초기에 분리한 이유는 무엇인가.
   - 0길이는 주소 계산과 역참조가 전혀 필요 없는 성공 경로입니다. 초기에 반환하면 `NULL`이 전달돼도 메모리에 접근하지 않는 계약이 분명해집니다.
4. 바이트 표현용 포인터 타입을 선택한 이유는 무엇인가.
   - `unsigned char` 포인터는 임의 객체의 표현을 바이트 단위로 읽고 쓰기에 적합하고, 부호 확장 없이 0부터 255까지의 바이트 값을 보존합니다.
5. 주소 비교의 이식성 범위를 어떤 전제로 두었는가.
   - 서로 무관한 객체 포인터의 대소 비교는 피하고, 원본 범위 안에서 만든 `source + offset`과 목적지의 동일성만 검사합니다. 전제는 `length > 0`이면 원본과 목적지 범위가 유효하다는 것입니다.

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
  - `capacity == 0`이면 목적지를 전혀 쓰지 않으므로 NUL도 기록하지 않습니다. 그 밖에는 이 구현이 최대 `capacity - 1`바이트 뒤에 NUL을 씁니다.
- `strlcat`에서 목적지 길이를 `capacity`까지만 탐색해야 하는 이유는 무엇입니까?
  - `capacity`가 목적지 객체의 접근 한계이기 때문입니다. 그 안에서 NUL을 못 찾으면 더 읽지 않고 미종료 목적지로 처리해야 범위 밖 읽기를 막을 수 있습니다.
- 덧셈식 조건보다 "남은 공간" 관점의 조건이 오버플로 방어에 유리한 이유는 무엇입니까?
  - `destination_length + index + 1`처럼 계속 더하면 큰 값에서 wraparound할 수 있습니다. 이미 검증한 `destination_length < capacity`를 바탕으로 `capacity - destination_length - 1`을 계산해 인덱스 상한으로 쓰면 반복 조건의 덧셈을 없앨 수 있습니다.
- 반환값만으로 truncation을 어떻게 감지할 수 있습니까?
  - copy는 반환값이 `capacity` 이상이면 잘렸고, append도 반환값이 `capacity` 이상이면 완성하려던 문자열이 버퍼에 다 들어가지 않았습니다.

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
	size_t	source_length;
	size_t	index;

	source_length = 0;
	while (source[source_length] != '\0')
		source_length++;
	if (capacity == 0)
		return (source_length);
	index = 0;
	while (source[index] != '\0' && index < capacity - 1)
	{
		destination[index] = source[index];
		index++;
	}
	destination[index] = '\0';
	return (source_length);
}

size_t	bounded_append(char *destination, const char *source, size_t capacity)
{
	size_t	destination_length;
	size_t	source_length;
	size_t	index;

	destination_length = 0;
	while (destination_length < capacity
		&& destination[destination_length] != '\0')
		destination_length++;
	source_length = 0;
	while (source[source_length] != '\0')
		source_length++;
	if (destination_length == capacity)
		return (capacity + source_length);
	index = 0;
	/* 목적지 뒤에 남은 payload 칸만 사용하고 마지막 칸은 NUL용이다. */
	while (source[index] != '\0'
		&& index < capacity - destination_length - 1)
	{
		destination[destination_length + index] = source[index];
		index++;
	}
	destination[destination_length + index] = '\0';
	return (destination_length + source_length);
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
   - 호출자가 반환값과 capacity를 비교해 잘림을 검출하고, 필요하면 정확한 크기로 다시 할당할 수 있기 때문입니다. 원본도 copy는 원본 길이, append는 기존 목적지 길이와 원본 길이의 합을 반환합니다.
2. capacity 0과 1을 일반 루프에 섞지 않고 다루는 방법.
   - copy는 capacity 0에서 즉시 길이만 반환합니다. capacity 1은 `capacity - 1`이 0이라 복사 루프를 건너뛰고 첫 칸에 NUL만 기록합니다.
3. NUL 없는 목적지에서 반환값과 쓰기 동작을 어떻게 정의했는가.
   - capacity 안에 NUL이 없으면 목적지 길이를 capacity로 간주하고 아무것도 쓰지 않으며 `capacity + source_length`를 반환합니다. 이는 원본 `ft_strlcat`의 계약입니다.
4. 오버플로 가능성이 있는 덧셈 조건을 어떻게 피했는가.
   - 쓰기 조건은 검증된 범위에서 뺄셈으로 남은 payload 칸을 구했습니다. 다만 반환 길이의 합 자체는 이 오류 코드 없는 API로 표현할 수 있어야 한다는 호출자 전제가 필요하며, 원본 역시 같은 전제를 둡니다.
5. 이 API가 겹치는 입력을 지원하지 않는다는 계약을 어디에 둘 것인가.
   - 공개 헤더나 API 문서의 사전조건에 둡니다. 이 프로젝트의 제한 복사·붙이기 구현도 `memmove` 의미를 제공하지 않으므로, 겹침이 필요하면 별도 이동 API를 사용해야 합니다.

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
  - 연결 전 임시 객체는 현재 함수가 직접 정리해야 하고, 배열 슬롯이나 리스트 노드에 연결된 뒤에는 컨테이너 cleanup이 소유합니다.
- cleanup 함수에 "성공한 개수"를 넘기는 방식의 장점은 무엇입니까?
  - 실패 지점까지 초기화된 슬롯만 정확히 순회하므로 미초기화 포인터를 읽지 않습니다. 원본 `free_fields`도 성공한 필드 수를 신뢰합니다.
- content 변환은 성공했지만 노드 할당이 실패한 경우 누가 content를 해제해야 합니까?
  - 아직 노드가 소유하지 못했으므로 map 함수가 `del(mapped_content)`를 호출한 뒤, 이미 연결된 리스트는 `ft_lstclear`로 정리해야 합니다.
- 콜백이 반환한 포인터가 원본 content와 같을 수도 있다면 소유권 계약을 어떻게 문서화해야 합니까?
  - 성공·실패 시 `del`이 반환 포인터를 해제할 수 있다는 점과, 반환 포인터가 독립적으로 소유 가능한 객체여야 하는지를 명시해야 합니다. 원본과 같은 borrowed 포인터를 반환한다면 `del` 계약과 충돌할 수 있습니다.
- N번째 할당 실패 테스트가 정상 경로 테스트보다 잘 찾는 버그는 무엇입니까?
  - 배열만 만든 상태, 일부 필드만 연결된 상태, 변환 content만 생긴 상태처럼 정상 경로가 지나가기만 하는 부분 상태의 누수·double free·invalid free를 결정적으로 찾습니다.

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
	while (initialized_count > 0)
	{
		initialized_count--;
		free(fields[initialized_count]);
	}
	free(fields);
}

char	**split_fields(const char *text, char delimiter)
{
	char	**fields;
	size_t	count;
	size_t	text_index;
	size_t	field_index;
	size_t	start;
	size_t	length;
	size_t	index;

	if (text == NULL)
		return (NULL);
	count = 0;
	text_index = 0;
	while (text[text_index] != '\0')
	{
		while (text[text_index] == delimiter && text[text_index] != '\0')
			text_index++;
		if (text[text_index] != '\0')
		{
			if (count == (size_t)-1)
				return (NULL);
			count++;
		}
		while (text[text_index] != delimiter && text[text_index] != '\0')
			text_index++;
	}
	if (count == (size_t)-1
		|| count + 1 > (size_t)-1 / sizeof(char *))
		return (NULL);
	/* calloc의 NULL 초기화가 성공 결과의 sentinel도 보장한다. */
	fields = calloc(count + 1, sizeof(char *));
	if (fields == NULL)
		return (NULL);
	text_index = 0;
	field_index = 0;
	while (text[text_index] != '\0')
	{
		while (text[text_index] == delimiter && text[text_index] != '\0')
			text_index++;
		start = text_index;
		while (text[text_index] != delimiter && text[text_index] != '\0')
			text_index++;
		length = text_index - start;
		if (length > 0)
		{
			if (length == (size_t)-1)
				return (free_fields(fields, field_index), NULL);
			fields[field_index] = malloc(length + 1);
			if (fields[field_index] == NULL)
				return (free_fields(fields, field_index), NULL);
			index = 0;
			while (index < length)
			{
				fields[field_index][index] = text[start + index];
				index++;
			}
			fields[field_index][length] = '\0';
			field_index++;
		}
	}
	return (fields);
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
   - 필드 포인터가 `fields[field_index]`에 저장되면 배열 cleanup의 소유가 되고, `field_index`를 증가시켜 초기화 완료를 기록합니다. 리스트에서는 새 노드를 `mapped` 체인에 연결하는 순간 같은 이전이 일어납니다.
2. cleanup helper가 개수 또는 NULL sentinel 중 무엇을 신뢰하는가.
   - 실패 rollback은 `initialized_count`를 신뢰합니다. 지금 할당하려던 슬롯은 실패 시 NULL이더라도, 개수를 기준으로 하면 calloc의 초기화 여부에 의존하지 않고 성공한 슬롯만 해제할 수 있습니다.
3. 1-pass 동적 확장과 2-pass 사전 계수의 trade-off.
   - 원본처럼 2-pass로 세면 입력을 한 번 더 읽지만 배열 재할당과 중간 용량 상태가 사라져 rollback이 단순합니다. 1-pass는 한 번만 스캔하지만 확장 비용과 더 복잡한 소유권 전이가 생깁니다.
4. 콜백 기반 리스트 변환에서 content 해제 콜백이 필요한 이유.
   - 리스트 노드는 content의 구체 타입과 해제 방법을 알 수 없습니다. 그래서 노드 할당 실패의 임시 content와 이미 만든 노드들의 content를 같은 계약으로 정리하려면 `del` 콜백이 필요합니다.
5. N번째 실패 sweep을 어떻게 구성해야 모든 부분 상태를 통과하는가.
   - 정상 실행의 총 할당 횟수를 먼저 측정하고 1부터 그 횟수까지 매번 정확히 N번째 할당을 실패시켜 NULL 반환, live allocation 0, invalid free 0을 확인합니다. 마지막 다음 번호에서는 정상 성공도 확인합니다.

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
  - 그 전에 양수 partial write가 있었다면 cursor가 이미 이동했으므로 `EINTR` 뒤에는 남은 범위만 다시 요청해야 합니다. 해당 호출 자체가 `EINTR`이면 그 호출에서는 진전이 없습니다.
- 양수 partial write 뒤 영구 오류가 발생하면 이미 기록된 prefix를 되돌릴 수 있습니까?
  - 일반적인 FD 출력은 rollback할 수 없습니다. helper는 실패를 반환하되 외부에 prefix가 이미 관찰될 수 있음을 계약에 포함해야 합니다.
- 쓰기 결과 0을 무한 재시도하면 어떤 문제가 생깁니까?
  - cursor와 남은 길이가 변하지 않아 CPU를 소모하는 무한 루프가 됩니다. 원본은 이를 `EIO`로 바꿔 영구 실패 처리합니다.
- `SIGPIPE`와 `EPIPE`의 관계는 무엇이며 helper 바깥에서 어떤 정책이 필요할 수 있습니까?
  - 닫힌 pipe에 쓰면 기본적으로 `SIGPIPE`가 프로세스를 종료시킬 수 있고, 신호를 무시하거나 처리하면 `write`가 `EPIPE`를 반환합니다. 이 정책은 보통 프로세스 초기화 계층에서 정합니다.
- `errno`를 helper의 공개 계약에 포함할지 어떻게 결정합니까?
  - 호출자가 오류 종류별 복구를 해야 하면 보존·설정 규칙을 공개 계약으로 둡니다. 단순 성공/실패만 필요하면 반환값이 주 계약이고, 이 프로젝트는 0바이트 결과에 한해 `EIO`를 설정합니다.

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
	const unsigned char	*bytes;
	ssize_t				written;
	size_t				offset;
	size_t				request;

	bytes = buffer;
	offset = 0;
	while (offset < length)
	{
		request = length - offset;
		if (request > (size_t)SSIZE_MAX)
			request = (size_t)SSIZE_MAX;
		written = write(fd, bytes + offset, request);
		if (written > 0)
			offset += (size_t)written;
		else if (written < 0 && errno == EINTR)
			continue ;
		else
		{
			if (written == 0)
				errno = EIO;
			return (0);
		}
	}
	return (1);
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
   - 양수는 진행량이고, 0은 진전 없는 비정상 상태이며, `EINTR`는 호출을 다시 시도할 수 있고, 다른 음수는 영구 오류입니다. 상태 전이가 서로 달라 반드시 분리해야 합니다.
2. 이미 기록된 prefix를 재시도하지 않도록 상태를 어떻게 표현했는가.
   - `offset`을 성공적으로 기록한 바이트 수로 유지하고 항상 `bytes + offset`, `length - offset`을 다음 호출에 전달합니다.
3. `SSIZE_MAX`보다 큰 남은 길이를 어떻게 나누는가.
   - 매 반복의 요청 크기를 `min(length - offset, SSIZE_MAX)`로 제한합니다. 따라서 반환값도 `ssize_t`로 안전하게 표현됩니다.
4. 0바이트 반환의 오류 정책을 선택한 이유.
   - 양의 길이를 요청했는데 0이면 상태가 진전하지 않아 재시도 근거가 없습니다. 원본과 같이 `errno = EIO`로 설정하고 실패해 무한 루프를 막습니다.
5. `SIGPIPE` 무시 정책이 helper 내부가 아니라 프로세스 초기화에 있을 수 있는 이유.
   - 신호 disposition은 프로세스 전체에 영향을 주는 정책입니다. 범용 write helper가 매 호출마다 바꾸면 다른 코드의 신호 계약까지 변경하므로 상위 초기화에서 한 번 정하는 편이 명확합니다.

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
  - 있습니다. `written` 자체가 `INT_MAX`보다 크거나 기존 `count + written`이 `INT_MAX`를 넘으면 공개 반환형인 int로 표현할 수 없습니다.
- count overflow를 쓰기 전에 검사할 수 없는 이유와, 쓰기 뒤 검사할 때 생기는 외부 효과는 무엇입니까?
  - 실제 partial write 크기는 호출 전에는 알 수 없습니다. 따라서 반환 후 검사해야 하며, 범위를 넘었다고 판단할 때는 그 바이트가 이미 외부 스트림에 기록됐을 수 있습니다.
- 실패 뒤 formatter가 계속 변환 로직을 호출하더라도 추가 출력이 생기지 않게 하려면 어느 계층에서 차단해야 합니까?
  - 모든 출력이 통과하는 `output_write`의 입구에서 sticky `failed`를 검사해야 합니다. 그러면 상위 변환기가 실수로 계속 호출돼도 시스템 호출이 발생하지 않습니다.
- 한 글자씩 채우기와 고정 크기 chunk 채우기의 시스템 호출 비용 차이는 무엇입니까?
  - 한 글자씩 쓰면 최대 count번 호출하지만, 64바이트 버퍼라면 대략 `ceil(count / 64)`번으로 줄어듭니다. 스택 공간은 고정 64바이트입니다.
- 출력 스트림의 원자성을 보장하지 못하는 상황에서 반환값 계약은 무엇까지 말할 수 있습니까?
  - 0은 전체 요청을 처리했고 누적 count가 유효하다는 뜻이고, -1은 완료하지 못했다는 뜻입니다. 실패 전에 이미 기록된 prefix의 취소나 전체 포맷의 원자성까지 보장하지는 않습니다.

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
	output->fd = fd;
	output->count = 0;
	output->failed = 0;
}

int	output_write(t_output *output, const void *buffer, size_t length)
{
	const unsigned char	*cursor;
	ssize_t				written;
	size_t				request;

	if (output == NULL)
		return (-1);
	if (output->failed)
		return (-1);
	if (buffer == NULL && length > 0)
		return (output->failed = 1, -1);
	cursor = buffer;
	while (length > 0)
	{
		request = length;
		if (request > (size_t)SSIZE_MAX)
			request = (size_t)SSIZE_MAX;
		written = write(output->fd, cursor, request);
		if (written < 0 && errno == EINTR)
			continue ;
		if (written <= 0)
			return (output->failed = 1, -1);
		/* 바이트는 이미 쓰였지만, 좁은 공개 count에는 검증 후 반영한다. */
		if ((size_t)written > (size_t)INT_MAX
			|| output->count > INT_MAX - (int)written)
			return (output->failed = 1, -1);
		output->count += (int)written;
		cursor += (size_t)written;
		length -= (size_t)written;
	}
	return (0);
}

int	output_repeat(t_output *output, unsigned char byte, int count)
{
	unsigned char	buffer[64];
	int				index;
	int				chunk;

	if (output == NULL)
		return (-1);
	if (count < 0)
		return (output->failed = 1, -1);
	index = 0;
	while (index < (int)sizeof(buffer))
		buffer[index++] = byte;
	while (count > 0)
	{
		chunk = count;
		if (chunk > (int)sizeof(buffer))
			chunk = (int)sizeof(buffer);
		if (output_write(output, buffer, (size_t)chunk) < 0)
			return (-1);
		count -= chunk;
	}
	return (0);
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
   - 한 번 실패하면 중앙 write 경로가 이후 출력을 모두 거절하고 최상위는 `failed`만 보고 -1을 반환할 수 있습니다. 각 변환기가 별도 rollback이나 오류 상태를 가질 필요가 없습니다.
2. 시스템 호출 결과형과 공개 반환형의 범위가 다른 문제를 어떻게 처리했는가.
   - `ssize_t written`을 곧바로 int로 좁히지 않고 먼저 `INT_MAX`와 비교한 뒤, 기존 int count에 더할 수 있는지도 검사했습니다.
3. count 갱신과 실패 상태 갱신의 순서.
   - 양수 쓰기 뒤 범위 검증을 먼저 하고 안전할 때만 count를 더합니다. 시스템 오류·0바이트·범위 초과면 count를 건드리지 않고 `failed`를 고정합니다.
4. 반복 패딩 chunk 크기의 시간·스택 공간 trade-off.
   - chunk가 클수록 시스템 호출 수는 줄지만 스택 사용량이 늘어납니다. 원본의 64바이트는 작은 고정 공간으로 호출 수를 크게 줄이는 선택입니다.
5. 2단계 선측정과 결합할 때 count overflow 위험이 어떻게 줄어드는가.
   - 원본 포맷터는 렌더링 전에 전체 길이가 `INT_MAX` 이내인지 검증하므로 정상 경로에서 count 초과가 예상되지 않습니다. write 단계의 검사는 시스템 호출 결과와 구현 불일치에 대한 마지막 방어선입니다.

### 원본 확인 위치

- Thread 12 — 출력 컨텍스트와 쓰기 복구
- 커밋: `fix(output): 쓰기 결과를 집계하기 전에 범위 검증`, `perf(output): 반복 채움을 묶어서 출력`, `test(output): 쓰기 실패 시퀀스와 채움 전략 검증`
- 파일: `src/ft_output.c`, `src/ft_printf.c`, `src/ft_printf_internal.h`, `tests/test_output_faults.c`
- 함수·구조: `t_printf`, `ft_printf_init`, `ft_printf_write`, `ft_printf_putchar`, `ft_printf_putnchar`
- 관련 Thread: 6, 8, 22
