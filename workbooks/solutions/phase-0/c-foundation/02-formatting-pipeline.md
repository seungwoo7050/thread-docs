# 포맷팅 파이프라인 워크북

이 문서는 포맷 문자열을 문법으로 해석하고, 출력 길이를 선검증한 뒤, 텍스트와 숫자를 정해진 배치 규칙으로 렌더링하는 흐름을 다룬다.

---

<a id="f-01"></a>
## F-01. [Thread 9 / `feat(parser): 포맷 필드 모델과 해석기 추가`] 포맷 필드 파서와 문법 선검증

### 면접 질문

`%` 다음에 플래그, 너비, 선택적 정밀도, 변환 지정자가 이어지는 작은 포맷 문법을 파싱한다고 하겠습니다. 파서의 상태를 어떤 구조에 저장하고, 숫자 필드가 `INT_MAX`를 넘는지 문자열 전체를 큰 정수형으로 바꾸지 않고 어떻게 판정하겠습니까?

꼬리 질문:

- 플래그는 중복 입력을 허용하면서 결과 상태에서는 어떻게 표현할 수 있습니까?
  - 각 플래그를 한 비트로 두고 OR로 누적하면 같은 문자가 반복돼도 결과 비트는 한 번만 설정됩니다. 원본은 `FT_FLAG_LEFT`부터 `FT_FLAG_PLUS`까지의 비트셋을 사용합니다.
- `-`와 `0`, `+`와 공백 플래그가 동시에 있으면 어느 시점에 정규화하는 편이 좋습니까?
  - 필드를 모두 읽은 직후 파서에서 정규화하면 이후 측정기와 renderer가 같은 상태를 봅니다. 원본은 `-`가 있으면 `0`을, `+`가 있으면 공백을 제거합니다.
- `%.d`, `%.0d`, `%10.d`에서 정밀도 존재 여부와 값 0을 어떻게 구분합니까?
  - `.`을 본 순간 `has_precision = 1`로 두고 값은 0에서 시작합니다. 따라서 숫자가 없어도 명시적 0이고, 점 자체가 없을 때만 정밀도 부재입니다.
- 포맷 문자열이 `%`로 끝나거나 지원하지 않는 지정자가 나오면 파서는 어디까지 소비해야 합니까?
  - trailing `%`는 지정자가 없으므로 실패합니다. 미지원 문자는 필드의 지정자 위치에서 확인해 실패하며, 성공 cursor는 유효한 지정자 다음만 가리킵니다.
- 파싱 오류가 발생했을 때 이미 출력한 리터럴이 남지 않게 하려면 파서 바깥 구조가 어떻게 달라져야 합니까?
  - 렌더링 전에 전체 포맷을 같은 파서로 선측정해야 합니다. 원본의 2단계 구조는 이 단계가 실패하면 write 경로를 시작하지 않습니다.

### 30초 모범 답변

포맷 필드는 플래그 비트셋, 너비, 정밀도 값, 정밀도 존재 여부, 지정자로 분리합니다. 문법 순서대로 한 번만 전진하며 숫자는 새 digit을 더하기 전에 현재 값이 `(INT_MAX - digit) / 10`보다 큰지 확인합니다. 파싱 뒤에는 `-`가 있으면 `0`을 제거하고 `+`가 있으면 공백을 제거하는 식으로 의미를 정규화합니다. trailing `%`, 숫자 overflow, 지원하지 않는 지정자는 전체 포맷의 오류로 처리합니다.

### 답변 핵심 키워드

single pass, grammar order, bit flags, presence vs zero, pre-overflow check, normalization, unsupported specifier, trailing percent

### 백지 구현

#### 구현 목표

`%` 바로 다음 문자를 시작점으로 받아 한 개의 포맷 필드를 해석한다.

#### 인터페이스

```c
typedef struct s_format
{
	unsigned int	flags;
	int			width;
	int			precision;
	int			has_precision;
	char			specifier;
}   t_format;

const char	*parse_format_field(const char *cursor, t_format *out);
```

#### 입력과 출력

- `cursor`: `%` 다음 첫 문자
- 성공: 다음 미처리 문자를 가리키는 포인터
- 실패: `NULL`
- `out`: 성공한 필드의 정규화된 상태

#### 반드시 만족해야 할 조건

- 문법 순서는 플래그 0개 이상 → 너비 숫자 0개 이상 → 선택적 `.`과 정밀도 숫자 → 지정자다.
- 플래그 중복은 하나의 상태로 합친다.
- 너비와 정밀도 숫자 해석 중 `int` 범위를 넘으면 실패한다.
- `.`이 등장하면 뒤에 숫자가 없어도 `has_precision`은 참이고 값은 0이다.
- 지원하는 지정자 집합 밖의 문자는 실패로 처리한다.
- 성공한 결과에서는 충돌 플래그가 정규화되어 있다.

#### 경계 조건

- 플래그·너비·정밀도가 모두 없는 필드
- 같은 플래그의 반복
- 모든 플래그가 섞인 입력
- 너비 0, `INT_MAX`, `INT_MAX + 1`
- 정밀도 0, `INT_MAX`, `INT_MAX + 1`
- `.`만 있는 정밀도
- 문자열 끝에서 지정자가 없는 경우
- 알 수 없는 지정자

#### 실패 조건과 제약

- `cursor == NULL` 또는 `out == NULL`
- 숫자 overflow
- 지정자 누락·미지원
- 동적 할당을 사용하지 않는다.
- 숫자 해석을 위해 `atoi`, `strtol`을 호출하지 않는다.

```c
const char	*parse_format_field(const char *cursor, t_format *out)
{
	unsigned int	flag;
	int				digit;

	if (cursor == NULL || out == NULL)
		return (NULL);
	out->flags = 0;
	out->width = 0;
	out->precision = 0;
	out->has_precision = 0;
	out->specifier = '\0';
	while (*cursor == '-' || *cursor == '0' || *cursor == '#'
		|| *cursor == ' ' || *cursor == '+')
	{
		flag = 0;
		if (*cursor == '-')
			flag = 1U;
		else if (*cursor == '0')
			flag = 2U;
		else if (*cursor == '#')
			flag = 4U;
		else if (*cursor == ' ')
			flag = 8U;
		else if (*cursor == '+')
			flag = 16U;
		out->flags |= flag;
		cursor++;
	}
	while (*cursor >= '0' && *cursor <= '9')
	{
		digit = *cursor - '0';
		if (out->width > (INT_MAX - digit) / 10)
			return (NULL);
		out->width = out->width * 10 + digit;
		cursor++;
	}
	if (*cursor == '.')
	{
		out->has_precision = 1;
		cursor++;
		while (*cursor >= '0' && *cursor <= '9')
		{
			digit = *cursor - '0';
			if (out->precision > (INT_MAX - digit) / 10)
				return (NULL);
			out->precision = out->precision * 10 + digit;
			cursor++;
		}
	}
	out->specifier = *cursor;
	if (*cursor != 'c' && *cursor != 's' && *cursor != 'd'
		&& *cursor != 'i' && *cursor != 'u' && *cursor != 'x'
		&& *cursor != 'X' && *cursor != 'p' && *cursor != '%')
		return (NULL);
	/* 정규화된 상태 하나를 측정과 렌더가 함께 사용한다. */
	if (out->flags & 1U)
		out->flags &= ~2U;
	if (out->flags & 16U)
		out->flags &= ~8U;
	return (cursor + 1);
}
```

### 구현 후 자가 검증

- [ ] 가장 짧은 정상 필드가 정확히 한 지정자 뒤를 반환한다.
- [ ] 중복 플래그가 결과 비트에 한 번만 반영된다.
- [ ] `-`와 `0`, `+`와 공백의 우선순위가 일관된다.
- [ ] 정밀도 부재와 명시적 정밀도 0이 구분된다.
- [ ] `INT_MAX`는 허용하고 그보다 큰 첫 값에서 실패한다.
- [ ] 실패 시 지정되지 않은 포인터를 반환하지 않는다.
- [ ] trailing `%`와 미지원 지정자가 정상 변환으로 넘어가지 않는다.
- [ ] 입력 포인터는 각 문자를 최대 한 번씩 지나간다.
- [ ] 시간 복잡도는 필드 길이에 O(n), 추가 공간은 O(1)이다.

### 구현 후 설명할 것

1. 파서와 의미 정규화를 한 함수에 둘지 분리할지 선택한 이유.
   - 원본처럼 parsing 뒤 작은 정규화 helper를 호출하는 구조를 선택했습니다. 문법 소비와 우선순위 규칙은 논리적으로 나뉘지만, 파서의 성공 결과는 항상 정규화된 상태라는 계약을 유지합니다.
2. 정밀도 존재 여부를 별도 필드로 둔 이유.
   - 정밀도 부재와 명시적 0은 숫자 변환에서 의미가 다릅니다. 특히 값 0은 `.0`에서 digits가 사라지지만 정밀도 부재에서는 `0`을 출력합니다.
3. 숫자 overflow를 곱셈·덧셈 전에 판정한 방식.
   - 다음 digit을 `d`라고 할 때 현재 값이 `(INT_MAX - d) / 10`보다 큰지 먼저 검사했습니다. 안전한 경우에만 `value * 10 + d`를 수행합니다.
4. 미지원 문법을 관대하게 출력하지 않고 실패시킨 이유.
   - 이 프로젝트의 측정 단계는 지원 지정자 집합을 명시적으로 검증합니다. 알 수 없는 문법을 리터럴처럼 처리하면 측정과 렌더 규칙이 갈리고 오류를 숨길 수 있습니다.
5. 파서가 반환하는 cursor 계약이 상위 루프를 어떻게 단순화하는가.
   - 성공하면 다음 미처리 문자를, 실패하면 NULL을 반환하므로 상위 루프는 별도 소비 길이를 다시 계산하지 않고 그대로 다음 필드나 리터럴 처리를 이어갈 수 있습니다.

### 원본 확인 위치

- Thread 9 — 포맷 문법 해석과 선검증
- 커밋: `feat(parser): 포맷 필드 모델과 해석기 추가`, `feat(flags): 숫자 플래그 우선순위 정규화`
- 파일: `src/ft_parse.c`, `src/ft_printf_internal.h`
- 함수·구조: `t_format`, `ft_parse_decimal`, `ft_printf_init_format`, `ft_printf_parse`
- 관련 Thread: 8, 10, 11

---

<a id="f-02"></a>
## F-02. [Thread 8·9 / `fix(format): 지원 문법과 전체 출력 크기 선검증`] 측정 후 렌더링하는 2단계 포맷 파이프라인

### 면접 질문

포맷 문자열의 뒤쪽에서 오류나 반환 길이 overflow를 발견하면 앞쪽 출력은 이미 외부에 기록될 수 있습니다. 렌더링 전에 전체 포맷을 측정하는 2단계 구조가 이 문제를 어떻게 줄이며, 가변 인자를 두 번 순회할 때 `va_list`를 어떻게 다뤄야 합니까?

꼬리 질문:

- 같은 `va_list`를 측정과 렌더링에 그대로 재사용하면 왜 안전하지 않습니까?
  - `va_arg`가 목록의 순회 상태를 전진시키기 때문입니다. 측정이 소비한 동일 목록을 다시 쓰면 renderer가 다음 위치나 무효 상태에서 인자를 읽게 됩니다.
- 측정 함수와 렌더 함수의 변환 규칙이 달라지면 어떤 버그가 생깁니까?
  - 측정은 통과했는데 렌더 길이가 달라져 반환 count가 어긋나거나, 반대로 안전하다고 계산한 접근 범위를 renderer가 넘을 수 있습니다. 문자열 정밀도와 숫자 prefix 규칙을 특히 동일하게 유지해야 합니다.
- 선측정이 막아 주는 실패와 막아 주지 못하는 실패는 각각 무엇입니까?
  - trailing `%`, 미지원 지정자, 너비·정밀도 및 총 길이 overflow 같은 결정적 오류는 출력 전에 막습니다. FD 오류, short write 뒤 영구 오류 같은 실행 중 I/O 실패는 막지 못합니다.
- 출력 길이가 유효하더라도 실제 `write`가 중간에 실패할 수 있는데 반환 계약은 어떻게 유지합니까?
  - renderer가 -1을 반환하고 최상위도 이를 그대로 전파합니다. 다만 이미 기록된 prefix는 rollback할 수 없다는 점은 출력 API 계약에 포함됩니다.
- 포맷을 두 번 순회하는 비용과 부분 출력 방지의 trade-off를 어떻게 평가합니까?
  - CPU는 포맷 parsing과 일부 변환 계산을 두 번 하지만 큰 중간 버퍼 없이 결정적 오류의 부분 출력을 막습니다. 원본은 이 예측 가능성을 한 번의 추가 순회보다 우선합니다.

### 30초 모범 답변

첫 단계는 포맷을 파싱하면서 각 인자의 출력 길이와 전체 합을 계산하고, 문법 오류나 `INT_MAX` 초과를 출력 전에 거절합니다. `va_list`는 소비되는 상태이므로 원본에서 `va_copy`한 별도 목록을 측정에 사용하고 각각 `va_end`해야 합니다. 검증이 끝난 뒤 원본 목록으로 같은 규칙을 렌더링합니다. 이 구조는 결정 가능한 포맷 오류의 부분 출력을 막지만 실제 I/O 실패까지 원자적으로 만들지는 못합니다.

### 답변 핵심 키워드

measure then render, no deterministic partial output, `va_copy`, `va_end`, semantic parity, total length overflow, I/O remains fallible

### 백지 구현

#### 구현 목표

제공된 측정 함수와 렌더 함수를 조합해, 측정 실패 시 한 바이트도 출력하지 않는 최상위 가변 인자 함수를 작성한다.

#### 제공 인터페이스

```c
int	measure_format(const char *format, va_list *arguments);
int	render_format(int fd, const char *format, va_list *arguments);
```

- `measure_format`: 성공 시 전체 길이, 실패 시 -1
- `render_format`: 성공 시 실제 출력 길이, 실패 시 -1

#### 구현할 인터페이스

```c
int	formatted_write(int fd, const char *format, ...);
```

#### 반드시 만족해야 할 조건

- `format == NULL`이면 실패한다.
- 측정 단계에는 원본 가변 인자 상태와 독립된 복사본을 사용한다.
- 측정이 실패하면 `render_format`을 호출하지 않는다.
- 생성한 모든 `va_list` 수명은 정상·실패 경로에서 끝낸다.
- 렌더 실패는 -1로 전파한다.
- 성공 시 반환값은 출력한 바이트 수다.
- 측정 길이와 렌더 길이가 다르면 내부 계약 위반으로 취급할지 정책을 명시한다.

#### 경계 조건

- 빈 포맷
- 인자가 없는 리터럴
- 여러 종류의 인자를 소비하는 포맷
- trailing `%`
- 미지원 지정자
- 너비·정밀도 overflow
- 전체 길이가 `INT_MAX`와 정확히 같은 경우와 초과하는 경우
- 측정 성공 뒤 첫 쓰기 실패와 중간 쓰기 실패

#### 실패 조건과 제약

- 문법·길이 오류는 출력 전에 발견해야 한다.
- 실제 출력의 운영체제 오류는 선측정으로 제거할 수 없다.
- 포맷을 임의의 큰 중간 문자열로 조립하지 않는다.

```c
int	formatted_write(int fd, const char *format, ...)
{
	va_list	arguments;
	va_list	measure_arguments;
	int		measured;
	int		rendered;

	if (format == NULL)
		return (-1);
	va_start(arguments, format);
	va_copy(measure_arguments, arguments);
	measured = measure_format(format, &measure_arguments);
	va_end(measure_arguments);
	if (measured < 0)
	{
		va_end(arguments);
		return (-1);
	}
	rendered = render_format(fd, format, &arguments);
	va_end(arguments);
	if (rendered < 0 || rendered != measured)
		return (-1);
	return (rendered);
}
```

### 구현 후 자가 검증

- [ ] 정상 포맷에서 측정과 렌더가 같은 인자 값을 본다.
- [ ] 측정용 목록 소비가 렌더용 목록의 위치를 바꾸지 않는다.
- [ ] 모든 조기 반환에서 필요한 `va_end`가 실행된다.
- [ ] 잘못된 뒤쪽 필드가 있어도 앞쪽 리터럴이 출력되지 않는다.
- [ ] 전체 길이 경계에서 signed overflow가 발생하지 않는다.
- [ ] 렌더 중 I/O 실패는 성공 길이로 오인되지 않는다.
- [ ] 측정·렌더 규칙이 문자열 정밀도, 숫자 prefix, zero suppression에서 일치한다.
- [ ] 두 단계의 시간 복잡도가 각각 포맷과 출력 길이에 선형임을 설명할 수 있다.

### 구현 후 설명할 것

1. `va_copy`가 필요한 이유와 각 목록의 소유 수명.
   - 측정용 목록을 원본 `arguments`에서 복사해 독립적으로 소비합니다. 복사본은 측정 직후, 원본은 렌더 또는 조기 실패 직전에 각각 정확히 한 번 `va_end`합니다.
2. 측정 단계가 보장하는 무출력 실패 범위.
   - 포맷 문법, 지원 지정자, 숫자 필드 범위, 변환별 길이와 전체 `INT_MAX` 범위 오류는 renderer 호출 전에 실패하므로 출력이 없습니다.
3. 측정 코드와 렌더 코드의 의미 중복을 줄이는 방법.
   - 동일한 field parser와 정규화된 `t_format`을 공유하고, 문자열 bounded length 및 숫자 layout의 길이 규칙을 공통 helper나 동일한 식으로 유지합니다.
4. 포맷 2회 순회와 전체 문자열 사전 생성의 메모리·I/O trade-off.
   - 2회 순회는 계산을 반복하지만 출력 크기만큼의 heap이 필요 없습니다. 전체 문자열 생성은 write 호출을 줄이고 출력 전 완성을 확인할 수 있지만 큰 메모리와 할당 실패 경로가 추가됩니다.
5. 실제 write 실패가 발생했을 때 이미 출력된 prefix의 의미.
   - 반환값은 -1이지만 prefix는 외부 부작용으로 남을 수 있습니다. 선측정은 포맷의 결정적 오류만 무출력으로 만들며 I/O를 트랜잭션으로 만들지는 않습니다.

### 원본 확인 위치

- Thread 8 — 2단계 측정·렌더링 포맷 파이프라인
- Thread 9 — 포맷 문법 해석과 선검증
- 커밋: `fix(format): 지원 문법과 전체 출력 크기 선검증`
- 파일: `src/ft_printf.c`, `src/ft_measure.c`, `src/ft_parse.c`, `src/ft_printf_internal.h`
- 함수: `ft_printf`, `ft_printf_measure`, `ft_printf_parse`, `ft_is_supported_specifier`
- 관련 Thread: 10, 11, 12

---

<a id="f-03"></a>
## F-03. [Thread 10 / `fix(text): 문자열 정밀도 범위까지만 읽기`] 출력 정밀도만큼만 문자열 읽기

### 면접 질문

`%.3s`로 출력할 입력이 정확히 3바이트짜리 배열이고 그 안에 NUL이 없다고 하겠습니다. 먼저 `strlen`으로 전체 길이를 구한 뒤 3으로 자르는 구현이 왜 잘못이며, 출력 길이 계산과 실제 메모리 접근 범위를 어떻게 일치시켜야 합니까?

꼬리 질문:

- 정밀도가 없는 `%s`와 정밀도가 있는 `%s`는 탐색 계약이 어떻게 다릅니까?
  - 정밀도가 없으면 NUL까지 읽을 수 있는 정상 C 문자열이어야 합니다. 정밀도가 있으면 NUL 또는 precision 중 먼저 만나는 경계까지만 읽을 수 있으면 됩니다.
- precision이 0이면 입력 첫 바이트를 읽을 필요가 있습니까?
  - 없습니다. 길이는 즉시 0이고 출력 바이트도 없으므로 포인터를 역참조할 이유가 없습니다.
- 너비 계산을 위해 길이를 구하는 측정 단계에서도 같은 bounded scan이 필요한 이유는 무엇입니까?
  - 측정도 실제 메모리를 읽는 단계입니다. renderer만 제한하면 측정의 선행 `strlen`이 이미 범위 밖을 읽을 수 있고 두 단계 길이도 달라집니다.
- 정밀도 제한이 결과 길이만 제한하는지, 메모리 안전 계약까지 제한하는지 어떻게 문서화합니까?
  - 이 구현에서는 둘 다 제한한다고 명시합니다. precision이 있으면 최대 그 바이트까지만 관찰하고, 반환 길이와 실제 쓰기 길이도 같은 값으로 사용합니다.

### 30초 모범 답변

정밀도가 있는 문자열 변환에서는 최대 precision바이트만 필요하므로 그 범위를 넘어서 NUL을 찾으면 안 됩니다. `strlen` 뒤 길이를 자르면 출력은 3바이트여도 측정 과정에서 배열 밖을 읽을 수 있습니다. 따라서 NUL을 만나거나 precision에 도달할 때까지만 세고, 측정과 렌더가 같은 길이를 사용해야 합니다. precision 0이면 입력 내용을 읽지 않고 길이 0으로 처리할 수 있습니다.

### 답변 핵심 키워드

bounded scan, read bound equals output bound, non-NUL buffer, precision zero, measurement/render parity, no speculative read

### 백지 구현

#### 구현 목표

최대 바이트 수 안에서만 NUL을 찾는 길이 함수를 작성한다.

#### 인터페이스

```c
size_t	bounded_text_length(const char *text, size_t limit);
```

#### 입력과 출력

- `text`는 NUL 종료일 수도 있고, 최소 `limit`바이트만 접근 가능한 배열일 수도 있다.
- 반환값은 첫 NUL의 인덱스와 `limit` 중 작은 값이다.

#### 반드시 만족해야 할 조건

- `limit` 바깥의 바이트를 읽지 않는다.
- `limit == 0`이면 `text`를 역참조하지 않는다.
- NUL이 범위 안에 있으면 그 앞 길이를 반환한다.
- NUL이 없으면 정확히 `limit`를 반환한다.

#### 경계 조건

- limit 0
- 첫 바이트가 NUL
- 마지막 허용 바이트 직전에 NUL
- limit 범위에 NUL이 없음
- 실제 문자열 길이보다 작은 limit, 같은 limit, 큰 limit

#### 실패 조건과 제약

- 별도 실패 반환은 없다.
- `limit > 0`이면 `text`는 최소 limit바이트 또는 그 전에 NUL까지 읽을 수 있어야 한다.
- `strlen`, `strnlen`을 호출하지 않는다.

```c
size_t	bounded_text_length(const char *text, size_t limit)
{
	size_t	length;

	length = 0;
	while (length < limit && text[length] != '\0')
		length++;
	return (length);
}
```

### 구현 후 자가 검증

- [ ] NUL 종료 문자열에서 기대 길이를 반환한다.
- [ ] NUL 없는 고정 배열에서 limit까지만 읽는다.
- [ ] limit 0에서 잘못된 포인터를 전달해도 역참조하지 않는다.
- [ ] NUL이 limit 바로 바깥에 있어도 결과와 접근 범위가 달라지지 않는다.
- [ ] 측정 단계와 렌더 단계가 같은 길이를 공유할 수 있다.
- [ ] 시간 복잡도는 O(min(문자열 길이, limit)), 공간은 O(1)이다.

### 구현 후 설명할 것

1. 결과를 자르는 것과 읽기 자체를 제한하는 것의 차이.
   - `strlen` 결과를 나중에 자르면 이미 NUL을 찾느라 limit 밖을 읽었습니다. 조건식에서 `length < limit`를 먼저 검사해야 관찰 범위 자체가 제한됩니다.
2. precision 0을 특별히 빠르게 처리할 수 있는 이유.
   - loop의 첫 조건이 거짓이라 입력을 읽지 않고 0을 반환합니다. 출력 의미와 접근 의미가 모두 빈 범위입니다.
3. 측정·렌더 양쪽에서 동일 helper를 써야 하는 이유.
   - 같은 바이트 수와 같은 접근 상한을 보장해야 선측정 값이 실제 출력과 일치합니다. 원본의 측정기와 text renderer도 동일한 bounded 조건을 사용합니다.
4. 바이트 정밀도와 문자 단위 정밀도가 다른 인코딩에서 생길 수 있는 trade-off.
   - 이 printf 계열 계약은 바이트 기준이라 멀티바이트 문자를 중간에서 자를 수 있습니다. 문자 단위 안전성이 필요하면 decoding과 유효성 검사가 추가되지만 API 의미와 비용이 달라집니다.

### 원본 확인 위치

- Thread 10 — 문자 변환의 너비와 정밀도
- 커밋: `fix(text): 문자열 정밀도 범위까지만 읽기`, `test(text): NUL 없는 제한 문자열 회귀 검증`
- 파일: `src/ft_text.c`, `tests/test_ft_printf.c`
- 함수: `ft_local_strlen`, `ft_printf_print_string`
- 관련 Thread: 2, 8, 9

---

<a id="f-04"></a>
## F-04. [Thread 11 / `refactor(output): 숫자 출력 배치 로직 통합`] 숫자 접두사·정밀도·필드 패딩 배치

### 면접 질문

부호 또는 `0x` 접두사가 있는 숫자에 너비, 정밀도, 왼쪽 정렬, 0 채움을 적용한다고 하겠습니다. 공백, 접두사, 필드용 0, 정밀도용 0, 숫자, 오른쪽 공백은 어떤 순서로 출력되어야 하며, 이 순서를 하나의 공통 layout 함수로 묶는 장점은 무엇입니까?

꼬리 질문:

- 정밀도가 지정되면 `0` 플래그를 무시해야 하는 이유는 무엇입니까?
  - 정밀도가 숫자 자릿수의 최소 폭과 leading zero를 이미 정의하기 때문입니다. field 폭의 나머지는 공백으로 채워 두 종류의 0 의미가 충돌하지 않게 합니다.
- 값이 0이고 정밀도가 0일 때 숫자 자릿수를 생략하더라도 prefix는 항상 남습니까?
  - 항상은 아닙니다. prefix 결정은 변환기가 먼저 합니다. 원본에서 `%#x`의 0은 prefix가 없지만 포인터는 항상 `0x` prefix를 전달하므로 digits가 사라져도 prefix는 남습니다.
- 음수 부호가 field zero보다 먼저 나와야 하는 이유를 예로 설명해 보세요.
  - `%05d`에 -42라면 `-0042`여야 합니다. `00-42`처럼 부호 전에 0을 쓰면 숫자의 접두사와 크기 표현이 분리됩니다.
- `-`가 있으면 `0`을, `+`가 있으면 공백 부호를 제거하는 정규화는 parser와 renderer 중 어디에 두겠습니까?
  - 원본처럼 parser 직후에 둡니다. 그러면 측정과 모든 renderer가 충돌이 제거된 같은 field 상태를 사용합니다.
- 10진수와 16진수에 같은 layout 함수를 쓰려면 변환별 책임과 공통 책임을 어떻게 나눠야 합니까?
  - 변환기는 진법에 맞는 digits, 대소문자, 부호나 `0x` prefix를 정합니다. 공통 layout은 suppression, precision zero, field padding과 출력 순서만 담당합니다.

### 30초 모범 답변

숫자 변환은 먼저 실제 자릿수와 zero-suppression을 정하고, 정밀도 0 개수와 prefix 길이를 계산한 뒤 전체 payload를 기준으로 필드 패딩을 계산합니다. 출력 순서는 왼쪽 공백, prefix, 필드용 0, 정밀도용 0, digits, 오른쪽 공백입니다. 정밀도가 있거나 왼쪽 정렬이면 field zero는 쓰지 않습니다. 진법별 함수는 digits와 prefix만 결정하고 공통 layout이 배치를 담당하면 규칙 중복을 줄일 수 있습니다.

### 답변 핵심 키워드

layout invariant, prefix before zero padding, precision zeros, field zeros, zero suppression, left/right spaces, normalized flags, shared renderer

### 백지 구현

#### 구현 목표

이미 변환된 숫자 자릿수와 prefix를 받아 필드 규칙에 맞게 출력하는 함수를 작성한다.

#### 인터페이스

```c
typedef struct s_number_format
{
	int	width;
	int	precision;
	int	has_precision;
	int	left_aligned;
	int	zero_padded;
}   t_number_format;

int	render_numeric_layout(
	int fd,
	const t_number_format *format,
	const char *prefix,
	const char *digits,
	size_t digit_length,
	int value_is_zero
);
```

#### 입력과 출력

- `prefix`: `""`, `"-"`, `"+"`, `" "`, `"0x"`, `"0X"` 중 변환기가 결정한 문자열
- `digits`: 부호와 접두사를 제외한 자릿수
- 성공 시 0, 쓰기 실패 시 -1

#### 반드시 만족해야 할 조건

- `has_precision && precision == 0 && value_is_zero`이면 자릿수 길이를 0으로 취급한다.
- 정밀도용 0은 지정한 정밀도에서 실제 자릿수를 뺀 만큼이다.
- 왼쪽 정렬 또는 명시적 정밀도가 있으면 field zero를 사용하지 않는다.
- field zero를 사용하는 경우 prefix가 먼저 출력된다.
- 전체 결과 길이는 width보다 작지 않으며, payload가 더 길면 잘리지 않는다.
- 모든 쓰기 실패를 즉시 전파한다.

#### 경계 조건

- 값 0, 정밀도 부재
- 값 0, 정밀도 0
- 음수와 큰 width의 zero padding
- `0x` prefix와 precision
- width가 payload와 같은 경우
- width가 payload보다 작은 경우
- 왼쪽 정렬과 zero flag가 함께 있는 경우
- precision이 실제 자릿수보다 작은 경우

#### 실패 조건과 제약

- 길이 합이 공개 반환 범위를 넘는 경우를 상위 측정 단계에서 거절했다고 가정하거나 이 함수에서 별도 방어한다.
- 자릿수 문자열 생성은 문제 범위 밖이다.
- 모든 실제 쓰기는 완전 쓰기 helper를 사용한다고 가정한다.

```c
int	render_numeric_layout(
	int fd,
	const t_number_format *format,
	const char *prefix,
	const char *digits,
	size_t digit_length,
	int value_is_zero)
{
	char	padding_buffer[64];
	size_t	prefix_length;
	size_t	precision_zeros;
	size_t	padding;
	size_t	chunk;
	size_t	index;
	int		field_zero;

	prefix_length = 0;
	while (prefix[prefix_length] != '\0')
		prefix_length++;
	if (format->has_precision && format->precision == 0 && value_is_zero)
		digit_length = 0;
	precision_zeros = 0;
	if (format->has_precision
		&& (size_t)format->precision > digit_length)
		precision_zeros = (size_t)format->precision - digit_length;
	padding = 0;
	if ((size_t)format->width
		> prefix_length + precision_zeros + digit_length)
		padding = (size_t)format->width
			- prefix_length - precision_zeros - digit_length;
	field_zero = format->zero_padded && !format->left_aligned
		&& !format->has_precision;
	index = 0;
	while (index < sizeof(padding_buffer))
		padding_buffer[index++] = ' ';
	while (!format->left_aligned && !field_zero && padding > 0)
	{
		chunk = padding;
		if (chunk > sizeof(padding_buffer))
			chunk = sizeof(padding_buffer);
		if (!write_all(fd, padding_buffer, chunk))
			return (-1);
		padding -= chunk;
	}
	if (!write_all(fd, prefix, prefix_length))
		return (-1);
	index = 0;
	while (index < sizeof(padding_buffer))
		padding_buffer[index++] = '0';
	while (field_zero && padding > 0)
	{
		chunk = padding;
		if (chunk > sizeof(padding_buffer))
			chunk = sizeof(padding_buffer);
		if (!write_all(fd, padding_buffer, chunk))
			return (-1);
		padding -= chunk;
	}
	while (precision_zeros > 0)
	{
		chunk = precision_zeros;
		if (chunk > sizeof(padding_buffer))
			chunk = sizeof(padding_buffer);
		if (!write_all(fd, padding_buffer, chunk))
			return (-1);
		precision_zeros -= chunk;
	}
	if (!write_all(fd, digits, digit_length))
		return (-1);
	index = 0;
	while (index < sizeof(padding_buffer))
		padding_buffer[index++] = ' ';
	while (format->left_aligned && padding > 0)
	{
		chunk = padding;
		if (chunk > sizeof(padding_buffer))
			chunk = sizeof(padding_buffer);
		if (!write_all(fd, padding_buffer, chunk))
			return (-1);
		padding -= chunk;
	}
	return (0);
}
```

### 구현 후 자가 검증

- [ ] 음수 zero padding에서 `-`가 0들보다 앞에 있다.
- [ ] `0x`/`0X` prefix가 field zero보다 앞에 있다.
- [ ] precision 0과 값 0 조합에서 digits suppression이 정확하다.
- [ ] precision이 있으면 zero flag가 결과에 영향을 주지 않는다.
- [ ] 왼쪽 정렬에서 오른쪽 공백만 추가된다.
- [ ] width가 작아도 payload가 잘리지 않는다.
- [ ] prefix, precision zero, digit 길이의 합이 width 계산에서 한 번씩만 반영된다.
- [ ] 각 출력 단계의 실패 뒤 후속 단계가 실행되지 않는다.
- [ ] 시간 복잡도는 출력 길이에 O(n), 추가 공간은 고정 크기다.

### 구현 후 설명할 것

1. 진법별 변환과 공통 layout의 책임 경계.
   - 10진·16진 변환은 magnitude를 digits로 만들고 `-`, `+`, 공백, `0x` 같은 prefix를 결정합니다. layout은 그 문자열들의 배치와 width·precision만 처리합니다.
2. 출력 순서를 invariant로 고정해 조건문 폭증을 줄인 방식.
   - 항상 왼쪽 공백, prefix, field zero, precision zero, digits, 오른쪽 공백 순서로 두고 각 구간 길이가 0일 수 있게 했습니다. 변환별 분기는 구간 길이 계산에만 남습니다.
3. 값 0·정밀도 0에서 digits와 prefix를 각각 어떻게 해석했는가.
   - layout은 digits 길이만 0으로 바꾸고 prefix는 호출자가 결정한 그대로 둡니다. 그래서 hex 0의 `#`처럼 prefix가 원래 비어 있으면 모두 사라지고, pointer prefix처럼 전달됐으면 유지됩니다.
4. field zero와 precision zero를 별도 수량으로 둔 이유.
   - field zero는 width를 맞추며 prefix 뒤에 오고 명시적 precision에서 비활성화됩니다. precision zero는 최소 자릿수를 맞추는 숫자 의미이므로 서로 다른 조건과 길이를 가집니다.
5. 길이 계산과 실제 출력이 어긋나지 않게 만드는 방법.
   - 측정기와 layout이 suppression, prefix, precision, width 공식을 동일하게 사용하고, 실제 출력도 계산한 각 구간을 정확히 한 번씩 기록합니다. 최상위에서 측정값과 렌더값 불일치도 실패로 볼 수 있습니다.

### 원본 확인 위치

- Thread 11 — 숫자 접두사와 패딩 배치
- 커밋: `feat(numeric): 숫자 정밀도와 0 채움 적용`, `refactor(output): 숫자 출력 배치 로직 통합`, `fix(decimal): INT_MIN 크기를 unsigned 범위에서 계산`
- 파일: `src/ft_numeric_layout.c`, `src/ft_number.c`, `src/ft_hex.c`, `src/ft_printf_internal.h`
- 함수: `ft_printf_write_numeric_layout`, `ft_printf_print_signed`, `ft_printf_print_unsigned`, `ft_printf_print_hex`, `ft_printf_print_pointer`
- 관련 Thread: 8, 9, 10, 12
