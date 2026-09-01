# 해석 파이프라인: lexer, parser, 확장

이 문서는 입력 문자열을 실행 가능한 명령 그래프로 바꾸는 경계에 집중한다. 인용 정보 보존, 소유권이 있는 parser, 실행 직전 확장, 공개 API 계약을 하나의 흐름으로 연습한다.

## 문서 내 면접 포인트

- [P01. 인용 의미를 보존하는 토큰화](#p01)
- [P02. 명령 그래프 파싱과 실패 원자성](#p02)
- [P03. 조건 실행과 지연 확장의 단계 순서](#p03)
- [P04. 공개 parser API의 결과·오류·소유권 계약](#p04)

---

<a id="p01"></a>
## P01. [Thread 02 / `feat(lexer): 인용 단어와 토큰 수명 관리`] 인용 의미를 보존하는 토큰화

### 면접 질문

- 토큰화 단계에서 따옴표를 단순히 제거하면 이후 확장 단계에서 어떤 정보를 잃게 됩니까?
- 연산자 문자가 인용 내부에 있을 때와 외부에 있을 때를 어떻게 구분했습니까?
- 빈 인용 문자열, 닫히지 않은 따옴표, 부분 인용 단어를 각각 어떤 토큰 상태로 표현해야 합니까?
- 꼬리 질문: heredoc 구분자의 인용 여부를 토큰에서 리다이렉션 노드까지 별도 필드로 전달한 이유는 무엇입니까?
  - 모범답변: 이 프로젝트에서는 구분자 문자열의 따옴표를 lexer에서 제거·인코딩하므로 실행 단계가 원문만 보고 인용 여부를 복원할 수 없습니다. `quoted`를 `heredoc_quoted`로 전달해야 구분자는 dequote만 하고, 인용된 구분자의 본문에서는 `$NAME`과 `$?` 확장을 끌 수 있습니다.
- 꼬리 질문: 제어 문자를 삽입하는 방식과 조각별 quote mode를 저장하는 방식의 trade-off는 무엇입니까?
  - 모범답변: 프로젝트의 `LITERAL_MARK` 방식은 문자열 하나로 작은따옴표의 literal 의미를 전달해 자료구조가 단순하지만 표식 바이트를 예약해야 하고 순회 코드가 인코딩 규약을 알아야 합니다. 조각별 mode는 의미가 명시적이고 충돌이 없지만 노드·할당·정리 비용이 늘어납니다.

### 30초 모범 답변

셸의 lexer는 문자열 경계만 자르는 단계가 아니라, 나중의 확장 가능 여부까지 보존해야 합니다. 이 구현은 인용 내부의 문자와 토큰 전체의 인용 여부를 보존해 `$`와 연산자가 이후 단계에서 올바르게 해석되도록 했습니다. 연산자는 인용 밖에서만 토큰이 되고, 닫히지 않은 따옴표는 부분 토큰을 넘기지 않고 오류와 함께 전부 정리합니다. 제어 표식은 구조가 단순한 대신 입력 문자와의 충돌 규약이 필요하고, 조각 AST는 명확하지만 자료구조와 메모리 관리가 더 복잡합니다.

### 답변 핵심 키워드

quote context · operator boundary · `LITERAL_MARK` · empty quoted word · `heredoc_quoted` · rollback

### 백지 구현

**구현 목표**

한 개의 셸 단어를 스캔해 인용 의미가 보존된 결과와 시작 위치를 만든다. 전체 lexer가 아니라 현재 커서에서 다음 단어 하나를 읽는 부분만 구현한다. 아래 타입과 함수는 면접용 축소 인터페이스다.

**인터페이스 또는 함수 시그니처**

```c
typedef struct s_scanned_word {
    char    *encoded;
    size_t  start;
    int     quoted;
} t_scanned_word;

int scan_shell_word(const char *line, size_t *cursor,
    t_scanned_word *out, char **error)
{
    t_string_builder    word;
    size_t              i;
    char                quote;

    if (line == NULL || cursor == NULL || out == NULL)
        return (1);
    out->encoded = NULL;
    out->start = *cursor;
    out->quoted = 0;
    if (error != NULL)
        *error = NULL;
    if (string_builder_init(&word) != 0)
        return (1);
    i = *cursor;
    while (line[i] != '\0'
        && !(line[i] == ' ' || line[i] == '\t' || line[i] == '\n'
            || line[i] == '\r' || line[i] == '\v' || line[i] == '\f')
        && !(line[i] == '|' || line[i] == '<' || line[i] == '>'
            || line[i] == '&' || line[i] == ';'))
    {
        if (line[i] == '\'' || line[i] == '"')
        {
            quote = line[i++];
            out->quoted = 1;
            while (line[i] != '\0' && line[i] != quote)
            {
                /* 작은따옴표 안의 바이트는 expander가 건드리지 않게 표시한다. */
                if ((quote == '\''
                        && (string_builder_append_char(&word, '\001') != 0
                            || string_builder_append_char(&word, line[i]) != 0))
                    || (quote == '"'
                        && string_builder_append_char(&word, line[i]) != 0))
                    return (string_builder_discard(&word), 1);
                i++;
            }
            if (line[i] == '\0')
            {
                string_builder_discard(&word);
                if (error != NULL)
                    *error = sh_strdup("syntax error: unclosed quote");
                return (1);
            }
            i++;
        }
        else if (string_builder_append_char(&word, line[i++]) != 0)
            return (string_builder_discard(&word), 1);
    }
    out->encoded = string_builder_take(&word);
    *cursor = i; /* 완성된 결과의 소유권과 커서를 함께 커밋한다. */
    return (0);
}
```

**입력과 출력**

- `line`: NUL 종료 입력 문자열
- `cursor`: 시작 인덱스를 받고, 성공하면 단어 다음 위치로 이동
- `out`: 소유권이 호출자에게 넘어가는 인코딩 문자열과 메타데이터
- 성공 시 `0`, 구문 또는 할당 실패 시 `1`

**반드시 만족해야 할 조건**

- 공백과 비인용 연산자 앞에서 단어 스캔을 멈춘다.
- 작은따옴표 내부 문자는 확장되지 않는 문자라는 의미를 잃지 않는다.
- 큰따옴표 내부의 연산자 문자는 연산자 토큰으로 분리하지 않는다.
- 빈 인용 문자열도 길이 0인 유효 단어로 표현한다.
- 부분 인용이 한 번이라도 있으면 `quoted`가 참이 된다.
- 성공 결과 문자열은 항상 NUL 종료된다.

**경계 조건**

- 입력 끝에서 시작하는 경우
- `''`, `""`, `a''b`, `E"OF"`
- 인용 안의 공백·파이프·리다이렉션 문자
- 표식 문자와 동일한 바이트를 입력으로 허용할지에 대한 명시적 규약

**실패 조건**

- 닫히지 않은 작은따옴표 또는 큰따옴표
- 결과 버퍼 확장 실패
- 오류 문자열 생성 실패가 발생해도 이미 확보한 결과 버퍼를 누수하지 않음

**제약**

- 전역 상태를 사용하지 않는다.
- 실패 시 `out`은 호출자가 해제할 유효 결과를 소유하지 않는다.
- 완전한 POSIX 셸 escape 규칙까지 구현하지 말고 프로젝트에서 확인된 인용 범위만 다룬다.

### 구현 후 자가 검증

- [ ] 정상 경로: `echo abc`, `echo 'a b'`, `echo "a|b"`가 예상한 단어 하나로 나온다.
- [ ] 경계값: `''`가 토큰 자체는 존재하지만 내용 길이는 0이다.
- [ ] 구문 실패: 닫히지 않은 따옴표에서 커서와 결과 소유권이 애매하게 남지 않는다.
- [ ] invariant: 성공한 `encoded`는 NUL 종료되고 `start`는 원래 입력 위치다.
- [ ] 누락 처리: 인용 밖의 연산자를 단어에 삼키지 않는다.
- [ ] resource cleanup: 모든 중간 버퍼가 성공 시 한 번만 이전되고 실패 시 한 번만 해제된다.
- [ ] 시간 복잡도: 결과 조립이 입력 길이에 대해 상각 O(n)인지 설명할 수 있다.

### 구현 후 설명할 것

1. 왜 quote 정보를 parser나 expander가 재추론하게 하지 않았는지
   - 모범답변: lexer가 원문 경계를 소비한 뒤에는 어떤 문자가 작은따옴표 안에 있었는지 재구성할 수 없습니다. 따라서 토큰 생성 시 literal 표식과 전체 `quoted` 상태를 남기고, parser와 expander는 그 계약만 해석합니다.
2. 제어 표식 인코딩의 불변식과 충돌 방지 규약
   - 모범답변: `LITERAL_MARK` 다음의 한 바이트는 확장하지 않고 그대로 출력한다는 불변식입니다. 현재 프로젝트는 이 바이트를 내부 예약 값으로 취급하므로, 임의 바이너리 입력까지 지원하려면 escape 규칙을 추가하거나 조각 표현으로 바꿔야 합니다.
3. 빈 문자열 토큰과 토큰 없음의 차이
   - 모범답변: `''`와 `""`는 명령 인자로 전달될 길이 0의 단어이므로 `encoded[0] == '\0'`인 토큰이 존재합니다. 공백이나 입력 끝만 만난 경우는 새 단어 토큰 자체가 없습니다.
4. 오류 시 부분 토큰 목록까지 누가 정리해야 하는지
   - 모범답변: 단어 스캐너는 자신이 만든 임시 builder를 정리하고, 전체 lexer는 이미 목록에 연결한 토큰들을 `free_tokens`로 회수합니다. 성공해 목록에 편입된 뒤의 소유권은 lexer의 head에 있습니다.
5. 조각 AST 표현으로 바꿀 때 얻는 명확성과 추가 비용
   - 모범답변: 각 조각에 unquoted·single·double mode를 두면 확장 규칙과 표식 충돌이 명시적으로 드러납니다. 대신 조각 노드 생성, 순회, 실패 롤백과 해제 경로가 늘어납니다.

### 원본 확인 위치

- Thread 02
- 커밋 `feat(lexer): 인용 단어와 토큰 수명 관리`
- 커밋 `feat(lexer): 셸 연산자를 토큰으로 구분`
- 커밋 `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존`
- `include/shell.h`: `t_token`, `t_redir.heredoc_quoted`
- `src/token.c`: `tokenize_line`, `read_word`, `new_token`, `push_token`, `free_tokens`
- `src/parser.c`: heredoc 구분자의 `quoted` 전달
- 관련 Thread 08, Thread 10

---

<a id="p02"></a>
## P02. [Thread 03 / `feat(parser): 명령 트리 소유권 모델 정의`] 명령 그래프 파싱과 실패 원자성

### 면접 질문

- `|`, `;`, `&&`, `||`를 어떤 계층의 자료구조로 나누었고, 왜 그렇게 나누었습니까?
- 파이프 앞뒤의 빈 명령, 연결자로 시작하거나 끝나는 입력을 어떤 parser 상태로 검출합니까?
- 파싱 중 네 번째 할당이 실패하면 이미 만든 명령·리다이렉션·pipeline의 소유권은 누가 회수합니까?
- 꼬리 질문: `command_count`와 연결 리스트 실제 길이가 어긋나지 않도록 어떤 invariant를 둡니까?
  - 모범답변: 새 command를 pipeline 목록에 실제로 연결하는 `append_command` 한 곳에서만 count를 증가시킵니다. 할당만 되었거나 실패 경로에서 해제되는 command는 세지 않으므로, 관찰 가능한 목록 길이와 count가 항상 함께 커밋됩니다.
- 꼬리 질문: source order를 연결 리스트로 보존하는 설계와 배열 기반 설계의 trade-off는 무엇입니까?
  - 모범답변: 연결 리스트는 파싱 중 뒤에 붙이기와 부분 실패 정리가 단순하지만 임의 접근과 cache locality가 약합니다. 배열은 순회가 빠르고 count가 자연스럽지만 성장 때 재할당 실패와 기존 원소 소유권 이전을 더 세밀하게 처리해야 합니다.

### 30초 모범 답변

명령은 리다이렉션과 argv를 소유하고, 여러 명령은 하나의 pipeline을, pipeline들은 `;`, `&&`, `||` 연결 관계를 이룹니다. parser는 `after_pipe`와 마지막 연결자 같은 상태로 빈 구간과 잘못된 종단을 즉시 거부합니다. 중요한 점은 생성 중인 객체와 이미 누적된 head를 모두 아는 단일 실패 경로에서 해제하는 것입니다. 성공 시에만 결과 소유권을 호출자에게 넘기고, 실패 시에는 결과가 관찰 가능한 반쪽 그래프로 남지 않게 합니다.

### 답변 핵심 키워드

ownership graph · source order · `after_pipe` · connector state · `parse_failure` · count invariant · failure atomicity

### 백지 구현

**구현 목표**

이미 만들어진 토큰 리스트를 `pipeline -> command -> redirection/argv` 그래프로 바꾸는 축소 parser를 구현한다. 토큰 생성과 실제 명령 실행은 범위에서 제외한다.

**인터페이스 또는 함수 시그니처**

```c
int parse_command_graph(const t_token *tokens,
    t_pipeline **out, char **error)
{
    t_pipeline  *parsed;
    char        *internal_error;
    char        **error_slot;

    if (out == NULL)
        return (1);
    *out = NULL;
    internal_error = NULL;
    error_slot = (error != NULL) ? error : &internal_error;
    *error_slot = NULL;
    /* 실제 parser는 토큰을 수정하지 않으며 생성 그래프만 새로 소유한다. */
    parsed = parse_tokens((t_token *)tokens, error_slot);
    if (*error_slot != NULL)
    {
        free_pipeline(parsed);
        free(internal_error);
        return (1);
    }
    *out = parsed;
    free(internal_error);
    return (0);
}
```

**입력과 출력**

- `tokens`: 호출자가 소유하며 함수가 수정하지 않는 토큰 리스트
- `out`: 성공 시 새 명령 그래프의 소유권을 받음
- `error`: 선택적 오류 문자열 출력 포인터
- 성공 시 `0`, 구문 또는 할당 실패 시 `1`

**반드시 만족해야 할 조건**

- `|`는 같은 pipeline 안의 명령 경계가 된다.
- `;`, `&&`, `||`는 pipeline 사이의 연결자가 된다.
- argv, redirection, command, pipeline의 원래 순서를 보존한다.
- 각 count 필드는 실제 연결 리스트 길이와 일치한다.
- 리다이렉션 연산자 다음에는 반드시 단어 토큰이 있어야 한다.
- 성공 전까지 생성 중인 모든 객체의 소유자가 명확해야 한다.

**경계 조건**

- 빈 토큰 리스트
- 선행·후행 pipe
- 연속 pipe와 연결자
- 연결자로 시작하거나 `&&`, `||`로 끝나는 입력
- 리다이렉션만 있는 명령
- 단일 명령·단일 pipeline

**실패 조건**

- 노드·argv 배열·문자열 복사 중 임의의 할당 실패
- 리다이렉션 대상 누락
- 지원하지 않는 연산자
- 오류 문자열 생성 자체가 실패해도 명령 그래프 누수 금지

**제약**

- 토큰 리스트는 해제하지 않는다.
- 실패 시 `*out == NULL`이어야 한다.
- 재귀 없이 연결 리스트 순회만 사용해도 된다.
- 면접 시간상 heredoc 본문 수집과 단어 확장은 구현하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 단일 명령, 다단 pipeline, 조건 연결 목록을 각각 만든다.
- [ ] 경계값: 리다이렉션만 있는 command를 빈 command와 혼동하지 않는다.
- [ ] 구문 실패: `| a`, `a |`, `a &&`, `a >`를 거부한다.
- [ ] invariant: 각 `command_count`와 `pipeline_count`가 실제 노드 수와 같다.
- [ ] resource cleanup: 모든 할당 지점 직후 실패를 가정해 누수를 추적한다.
- [ ] 중복·누락: 토큰 하나를 두 번 소비하거나 건너뛰지 않는다.
- [ ] 요구사항: 성공 결과의 원본 순서가 보존된다.

### 구현 후 설명할 것

1. 자료구조 계층이 셸 문법의 결합 구조를 어떻게 반영하는지
   - 모범답변: argv와 redirection은 command가 소유하고, `|`로 묶인 command 목록은 pipeline이 소유합니다. `;`, `&&`, `||`는 pipeline의 `next_op`로 다음 pipeline과 연결되어 파이프 결합이 조건 연결보다 안쪽 계층에 놓입니다.
2. 생성 중 객체와 누적 결과의 소유권을 분리한 이유
   - 모범답변: 아직 연결하지 않은 `cmd`와 `pipeline`은 지역 변수가 소유하고, 연결된 노드는 `head`가 소유합니다. 이 구분 덕분에 실패 시 현재 객체와 누적 그래프를 각각 한 번만 정리할 수 있습니다.
3. 중앙 실패 경로가 유지보수성에 주는 이점과 주의점
   - 모범답변: `parse_failure`가 오류 기록과 세 소유 영역의 해제를 한곳에서 처리해 새 할당 지점의 누락을 줄입니다. 다만 이미 head에 이전한 객체를 지역 포인터로 다시 해제하지 않도록 이전 직후 포인터 상태를 일관되게 관리해야 합니다.
4. 연결 리스트와 동적 배열 중 이 프로젝트에서 선택할 기준
   - 모범답변: 이 parser는 source order로 한 번 만들고 순차 실행하므로 연결 리스트의 단순한 증분 구축과 롤백이 적합합니다. 반복 임의 접근이나 노드 수를 미리 알 수 있다면 배열의 locality가 이점이 될 수 있습니다.
5. 구문 오류와 할당 오류를 동일한 반환값 안에서 구분하는 계약
   - 모범답변: 축소 API의 반환값은 성공·실패만 나타내고 `error`가 구체 메시지를 제공합니다. 실제 명령 처리에서는 `"allocation failure"`만 상태 1, 다른 parser 오류는 258로 매핑하지만, 일반 원칙으로는 문자열 비교 대신 오류 enum을 별도 반환하는 편이 안전합니다.

### 원본 확인 위치

- Thread 03
- 커밋 `feat(parser): 명령 트리 소유권 모델 정의`
- 커밋 `feat(parser): pipe로 명령을 pipeline에 결합`
- `include/shell.h`: `t_redir`, `t_command`, `t_connector`, `t_pipeline`, `t_sequence`
- `src/parser.c`: `parse_tokens`, `parse_failure`, `append_command`, `append_pipeline`, `free_pipeline`
- 관련 Thread 06, Thread 09

---

<a id="p03"></a>
## P03. [Thread 04 / `feat(exec): 조건 연결자와 지연 확장 실행`] 조건 실행과 지연 확장의 단계 순서

### 면접 질문

- 환경 변수와 `$?` 확장을 parse 직후가 아니라 실제 pipeline 실행 직전에 한 이유는 무엇입니까?
- `export X=new; echo $X`와 `false && echo $X`에서 eager expansion이 어떤 잘못을 만듭니까?
- 조건 연결자의 실행 여부와 `last_status` 갱신 순서를 설명해 보십시오.
- 꼬리 질문: heredoc은 조건 분기 전에 모두 수집하지만 일반 단어 확장은 선택된 pipeline에만 적용하는 이유는 무엇입니까?
  - 모범답변: 이 프로젝트는 한 입력 줄의 모든 heredoc을 source order로 먼저 읽어 후속 입력과의 경계를 보존합니다. 반면 argv와 일반 리다이렉션은 실행 시점 상태에 의존하므로 gate를 통과한 pipeline만 확장해야 앞선 builtin의 환경과 상태를 반영하고 건너뛴 분기의 실패도 격리됩니다.
- 꼬리 질문: AST를 제자리에서 확장하는 방식과 실행용 복사본을 만드는 방식의 trade-off는 무엇입니까?
  - 모범답변: 실제 구현의 제자리 교체는 추가 그래프 복사가 없어 단순하지만 일부 단어를 바꾼 뒤 할당 실패하면 원본 AST로 되돌리기 어렵고 재실행도 안전하지 않습니다. 실행용 복사본은 실패 원자성과 재사용성이 좋아지는 대신 전체 복사 비용과 소유권 계층이 늘어납니다.

### 30초 모범 답변

확장은 실행 시점의 환경과 직전 종료 상태를 읽으므로, parse 직후 미리 하면 앞선 builtin의 상태 변경을 반영할 수 없습니다. pipeline 목록은 왼쪽부터 보되 이전 연결자와 현재 `last_status`로 실행 여부를 먼저 결정하고, 실행할 pipeline만 확장합니다. 반면 heredoc 입력은 한 명령줄의 입력 스트림 경계를 유지하기 위해 분기 판단 전에 source order로 모두 소비합니다. 제자리 확장은 할당이 적지만 재실행과 실패 복구가 어렵고, 복사본은 안전한 대신 비용과 소유권이 늘어납니다.

### 답변 핵심 키워드

late binding · `last_status` · connector gating · selected pipeline only · heredoc pre-phase · mutation trade-off

### 백지 구현

**구현 목표**

파싱된 pipeline 목록을 왼쪽부터 실행하되 `&&`/`||` 조건을 적용하고, 실제로 실행할 pipeline에만 확장 callback을 호출하는 scheduler를 구현한다. fork와 FD 설정은 callback 뒤로 숨긴다.

**인터페이스 또는 함수 시그니처**

```c
typedef struct s_exec_hooks {
    int (*expand_one)(t_shell *shell, t_pipeline *pipeline);
    int (*run_one)(t_shell *shell, t_pipeline *pipeline, void *ctx);
} t_exec_hooks;

int execute_deferred(t_shell *shell, t_pipeline *head,
    const t_exec_hooks *hooks, void *ctx)
{
    t_connector previous;

    if (shell == NULL || hooks == NULL
        || hooks->expand_one == NULL || hooks->run_one == NULL)
    {
        if (shell != NULL)
            shell->last_status = 1;
        return (1);
    }
    previous = CONN_NONE;
    while (head != NULL && shell->running)
    {
        int should_run;

        should_run = 1;
        if (previous == CONN_AND && shell->last_status != 0)
            should_run = 0;
        else if (previous == CONN_OR && shell->last_status == 0)
            should_run = 0;
        if (should_run)
        {
            if (hooks->expand_one(shell, head) != 0)
                shell->last_status = 1;
            else
                shell->last_status = hooks->run_one(shell, head, ctx);
        }
        previous = head->next_op;
        head = head->next;
    }
    return (shell->last_status);
}
```

**입력과 출력**

- `shell`: `last_status`와 `running`을 보유
- `head`: source order의 pipeline 목록
- `hooks`: 선택된 pipeline 확장과 실행 함수
- 최종 `last_status` 반환

**반드시 만족해야 할 조건**

- 이전 pipeline의 `next_op`가 다음 pipeline의 gate로 작동한다.
- `&&` 뒤에서는 직전 실행 결과가 0일 때만 실행한다.
- `||` 뒤에서는 직전 실행 결과가 0이 아닐 때만 실행한다.
- 건너뛴 pipeline은 확장 callback도 호출하지 않는다.
- 실행한 pipeline의 결과만 `last_status`를 갱신한다.
- `shell->running == 0`이면 이후 pipeline을 실행하지 않는다.

**경계 조건**

- 빈 목록
- 첫 pipeline
- 연속된 조건 연결과 `;` 혼합
- 중간 builtin이 shell 실행 상태를 종료하는 경우
- 확장 결과가 빈 argv가 되는 경우는 callback의 책임으로 둔다.

**실패 조건**

- 확장 callback 실패
- 실행 callback 실패
- 필수 hook 누락
- 실패 후 상태 코드가 이전 성공 상태로 남지 않음

**제약**

- heredoc 수집은 이미 성공했다고 가정한다.
- 목록의 링크를 영구 변경하지 않는다.
- 건너뛴 분기의 문자열이나 리다이렉션을 관찰하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: `false || success`, `true && success`가 실행된다.
- [ ] 건너뛰기: `false && skipped`, `true || skipped`에서 두 callback이 모두 호출되지 않는다.
- [ ] 상태 변화: 각 실행 결과가 다음 gate에 즉시 반영된다.
- [ ] 경계값: 빈 목록은 기존 상태를 보존한다.
- [ ] 실패 경로: 확장 실패가 상태 1로 반영되고 다음 `||` 분기 판단에 사용된다.
- [ ] invariant: pipeline 순회 순서와 source order가 같다.
- [ ] 요구사항: `running`이 꺼지면 더 이상 callback을 호출하지 않는다.

### 구현 후 설명할 것

1. parse-time 정보와 run-time 정보의 경계를 어디에 그었는지
   - 모범답변: parser는 단어의 인코딩, command/pipeline 구조와 connector만 확정합니다. 현재 환경, `$?`, 실행 여부와 프로세스 상태는 실제 pipeline을 선택한 실행 단계에서 결정합니다.
2. 건너뛴 분기를 확장하지 않는 것이 의미론과 실패 격리에 주는 효과
   - 모범답변: `false && echo $X`처럼 실행되지 않을 명령은 현재 환경을 읽거나 할당할 필요가 없습니다. 따라서 앞선 상태 변경을 올바르게 반영하고, 실행되지 않을 분기의 확장 실패가 전체 명령 상태를 바꾸지 않습니다.
3. heredoc만 별도의 선행 단계인 이유
   - 모범답변: heredoc 본문은 같은 표준 입력 스트림에서 뒤따르는 줄을 소비하므로 조건 결과와 무관하게 문법에 나온 순서대로 경계를 먼저 확정해야 합니다. 본문 확장 여부는 저장된 구분자 인용 상태로 결정합니다.
4. 제자리 확장으로 인해 재실행이 어려워지는 점
   - 모범답변: 원래의 표식 포함 단어를 해제하고 확장 문자열로 교체하므로 두 번째 실행에서는 새 환경으로 다시 확장할 원본이 없습니다. 중간 실패 때도 일부 필드만 변한 상태가 남습니다.
5. scheduler를 process 실행과 분리해 테스트할 수 있는 방법
   - 모범답변: expand와 run을 callback으로 주입해 호출 순서, 건너뛴 pipeline, 전달된 상태와 반환 상태만 기록합니다. 그러면 fork나 FD 없이 connector gate와 `running` 중단을 결정적으로 검증할 수 있습니다.

### 원본 확인 위치

- Thread 04
- 커밋 `feat(expand): 환경과 종료 상태 단어 확장`
- 커밋 `feat(expand): argv와 리다이렉션 확장 연결`
- 커밋 `feat(exec): 조건 연결자와 지연 확장 실행`
- `src/expand.c`: `expand_word`, `expand_pipeline`, `shell_dequote_word`
- `src/exec.c`: `expand_one_pipeline`, `execute_one_pipeline`, `execute_pipeline_list_ctx`
- 관련 Thread 01, Thread 08

---

<a id="p04"></a>
## P04. [Thread 03 / `fix(parser): 오류 출력 포인터 없이도 구문 실패 반환`] 공개 parser API의 결과·오류·소유권 계약

### 면접 질문

- `shell_parse_line(line, &sequence, NULL)`에서 구문 오류가 나도 실패를 정확히 반환하려면 어떻게 해야 합니까?
- 빈 입력의 성공과 할당 실패의 `NULL` 결과를 어떻게 구분합니까?
- 성공·실패 각각에서 `sequence`와 오류 문자열의 소유권 계약을 설명해 보십시오.
- 꼬리 질문: executor hook을 둔 공개 sequence 실행 경로는 어떤 테스트 seam을 제공합니까?
  - 모범답변: `shell_execute_sequence`의 `run_pipeline` hook에 기록용 함수를 넣으면 실제 fork 없이 pipeline source order, connector별 실행·건너뛰기와 `last_status` 전파를 검증할 수 있습니다. `on_error`로 필수 hook 누락 경로도 관찰할 수 있습니다.
- 꼬리 질문: 문자열 오류 메시지로 오류 종류를 판별하는 설계의 한계와 대안은 무엇입니까?
  - 모범답변: 문구 변경, 번역이나 할당 실패로 문자열이 없을 때 분류가 깨지고 호출자가 내부 메시지에 결합됩니다. 성공 여부와 오류 enum을 반환하고, 사람이 읽는 문자열은 선택적 부가 정보로 분리하는 방식이 더 안정적입니다.

### 30초 모범 답변

오류 문자열은 선택적 출력일 뿐 성공 여부의 유일한 채널이 되어서는 안 됩니다. 호출자가 `error`를 주지 않으면 내부 임시 슬롯을 사용해 lexer와 parser 오류를 감지하고, 반환 전에 그 문자열을 해제합니다. 출력 sequence는 시작할 때 빈 상태로 초기화하고, 실패 시 부분 결과를 정리해 호출자가 일관된 상태만 보게 합니다. 더 큰 API라면 문자열 비교 대신 오류 enum과 선택적 메시지를 분리하는 편이 안정적입니다.

### 답변 핵심 키워드

optional error output · internal error slot · initialized output · ownership contract · error enum · test seam

### 백지 구현

**구현 목표**

기존 `tokenize_line`, `parse_tokens`, `free_tokens`, `free_pipeline`을 조합해 공개 parser wrapper를 구현한다. 하위 함수가 오류 문자열을 통해 실패를 알리는 현재 경계를 안전하게 감싼다.

**인터페이스 또는 함수 시그니처**

```c
int shell_parse_line(const char *line,
    t_sequence *sequence, char **error)
{
    t_token  *tokens;
    char     *internal_error;
    char     **error_slot;

    internal_error = NULL;
    error_slot = (error != NULL) ? error : &internal_error;
    if (sequence == NULL)
    {
        if (error_slot != NULL)
            *error_slot = sh_strdup("parse output is null");
        free(internal_error);
        return (1);
    }
    shell_sequence_init(sequence);
    tokens = tokenize_line(line, error_slot);
    if (*error_slot != NULL)
        return (free(internal_error), 1);
    sequence->pipelines = parse_tokens(tokens, error_slot);
    free_tokens(tokens);
    if (*error_slot != NULL)
    {
        shell_sequence_free(sequence);
        free(internal_error);
        return (1);
    }
    sequence->pipeline_count = count_pipelines(sequence->pipelines);
    free(internal_error);
    return (0);
}
```

**입력과 출력**

- `line`: 파싱할 한 줄
- `sequence`: 필수 출력 객체
- `error`: 선택적, 성공 시 NULL이고 실패 시 호출자가 해제할 문자열
- 성공 시 `0`, 실패 시 `1`

**반드시 만족해야 할 조건**

- `sequence == NULL`을 거부한다.
- 함수 시작 시 출력 sequence를 빈 상태로 초기화한다.
- `error == NULL`이어도 하위 구문 오류를 놓치지 않는다.
- 빈 입력은 유효한 빈 sequence로 성공한다.
- 실패하면 부분 pipeline을 모두 정리한다.
- 성공하면 pipeline 수를 실제 목록과 일치시킨다.

**경계 조건**

- 빈 문자열과 NULL line을 어떻게 취급할지 명시
- 오류 출력 포인터가 없는 호출
- 오류 출력 포인터가 가리키던 값이 NULL이 아닌 잘못된 호출 계약
- 유효한 단일 명령과 잘못된 후행 pipe

**실패 조건**

- 토큰화 구문 오류
- parser 구문 오류
- 하위 단계 할당 실패
- 출력 객체가 NULL인 API 오용

**제약**

- 하위 함수 구현은 수정하지 않는다.
- 내부 오류 문자열은 반환 전 반드시 해제한다.
- 정답을 문자열 메시지 내용과 단순 비교하는 테스트에 의존하지 않는다.

### 구현 후 자가 검증

- [ ] 정상 경로: 유효한 명령과 빈 입력이 성공한다.
- [ ] 선택 출력: `error == NULL`인 상태에서도 잘못된 pipe와 quote가 실패한다.
- [ ] 상태: 모든 실패 후 sequence가 빈 상태다.
- [ ] 소유권: 성공 결과는 호출자가 `shell_sequence_free`로 한 번 해제한다.
- [ ] 실패 경로: 내부 임시 오류 문자열이 누수되지 않는다.
- [ ] API 계약: NULL 출력 객체를 성공으로 받아들이지 않는다.
- [ ] 테스트 가능성: 실제 실행 없이 parser 결과와 connector 흐름을 검사할 수 있다.

### 구현 후 설명할 것

1. 오류 메시지 출력과 제어 흐름 반환을 분리해야 하는 이유
   - 모범답변: 메시지는 선택 출력이라 호출자가 NULL을 넘길 수 있고 메시지 할당도 실패할 수 있습니다. 따라서 반환값이 성공·실패를 결정하고 문자열은 진단 정보만 맡아야 합니다.
2. 출력 객체 선초기화가 호출자 코드를 단순하게 만드는 방식
   - 모범답변: 진입 즉시 빈 sequence로 만들고 실패 시에도 그 상태를 복구하면 호출자는 성공 여부와 무관하게 `shell_sequence_free`를 안전하게 호출할 수 있습니다.
3. 내부 임시 error slot의 수명
   - 모범답변: `error == NULL`일 때 지역 `internal_error`의 주소를 하위 함수에 전달하고, 오류 감지에 사용한 뒤 모든 반환 경로에서 문자열을 해제합니다. 외부 error가 있으면 그 소유권은 실패 시 호출자에게 갑니다.
4. 오류 enum을 추가한다면 ABI와 호출부가 어떻게 달라지는지
   - 모범답변: 반환형이나 별도 출력 인자에 lexical·syntax·allocation·invalid-argument enum을 추가하고 메시지는 선택적으로 유지할 수 있습니다. 호출부는 문자열 비교 없이 enum으로 상태 1과 258 같은 정책을 결정하게 됩니다.
5. hook 기반 실행 seam으로 parser만 독립 검증하는 방법
   - 모범답변: parse 결과를 `shell_execute_sequence`에 넘기고 가짜 `run_pipeline`이 방문한 command 수와 connector 순서를 기록하게 합니다. 실제 프로세스를 만들지 않고 그래프 구조와 조건 흐름을 함께 확인할 수 있습니다.

### 원본 확인 위치

- Thread 03
- 커밋 `fix(parser): 오류 출력 포인터 없이도 구문 실패 반환`
- 커밋 `test(parser): 공개 parser 오류 반환 검증`
- `src/parser.c`: `shell_parse_line`, `shell_sequence_init`, `shell_sequence_free`, `shell_execute_sequence`
- `tests/parser_api.c`: `check_line`, `main`
- 관련 Thread 09, Thread 11
