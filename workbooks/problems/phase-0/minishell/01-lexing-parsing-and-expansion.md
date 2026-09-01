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
- 꼬리 질문: 제어 문자를 삽입하는 방식과 조각별 quote mode를 저장하는 방식의 trade-off는 무엇입니까?

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
    // 직접 구현
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
2. 제어 표식 인코딩의 불변식과 충돌 방지 규약
3. 빈 문자열 토큰과 토큰 없음의 차이
4. 오류 시 부분 토큰 목록까지 누가 정리해야 하는지
5. 조각 AST 표현으로 바꿀 때 얻는 명확성과 추가 비용

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
- 꼬리 질문: source order를 연결 리스트로 보존하는 설계와 배열 기반 설계의 trade-off는 무엇입니까?

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
    // 직접 구현
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
2. 생성 중 객체와 누적 결과의 소유권을 분리한 이유
3. 중앙 실패 경로가 유지보수성에 주는 이점과 주의점
4. 연결 리스트와 동적 배열 중 이 프로젝트에서 선택할 기준
5. 구문 오류와 할당 오류를 동일한 반환값 안에서 구분하는 계약

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
- 꼬리 질문: AST를 제자리에서 확장하는 방식과 실행용 복사본을 만드는 방식의 trade-off는 무엇입니까?

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
    // 직접 구현
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
2. 건너뛴 분기를 확장하지 않는 것이 의미론과 실패 격리에 주는 효과
3. heredoc만 별도의 선행 단계인 이유
4. 제자리 확장으로 인해 재실행이 어려워지는 점
5. scheduler를 process 실행과 분리해 테스트할 수 있는 방법

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
- 꼬리 질문: 문자열 오류 메시지로 오류 종류를 판별하는 설계의 한계와 대안은 무엇입니까?

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
    // 직접 구현
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
2. 출력 객체 선초기화가 호출자 코드를 단순하게 만드는 방식
3. 내부 임시 error slot의 수명
4. 오류 enum을 추가한다면 ABI와 호출부가 어떻게 달라지는지
5. hook 기반 실행 seam으로 parser만 독립 검증하는 방법

### 원본 확인 위치

- Thread 03
- 커밋 `fix(parser): 오류 출력 포인터 없이도 구문 실패 반환`
- 커밋 `test(parser): 공개 parser 오류 반환 검증`
- `src/parser.c`: `shell_parse_line`, `shell_sequence_init`, `shell_sequence_free`, `shell_execute_sequence`
- `tests/parser_api.c`: `check_line`, `main`
- 관련 Thread 09, Thread 11
