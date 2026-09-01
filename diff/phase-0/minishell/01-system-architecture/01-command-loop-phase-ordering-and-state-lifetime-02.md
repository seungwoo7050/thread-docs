## `test(smoke): 주요 셸 명령 흐름 검증`

diff --git a/Makefile b/Makefile
index cbd8550..92c8e3a 100644
--- a/Makefile
+++ b/Makefile
@@ -35,7 +35,10 @@ $(TARGET): $(OBJS)
 readline:
 	$(MAKE) USE_READLINE=1
 
+test: $(TARGET)
+	./tests/smoke.sh
+
 clean:
 	rm -f $(TARGET) $(OBJS)
 
-.PHONY: all readline clean
+.PHONY: all readline test clean
diff --git a/tests/smoke.sh b/tests/smoke.sh
new file mode 100755
index 0000000..88d6e3c
--- /dev/null
+++ b/tests/smoke.sh
@@ -0,0 +1,133 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN="$ROOT/small-shell"
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell.XXXXXX")
+TMP_PHYSICAL=$(CDPATH= cd -- "$TMP" && pwd -P)
+
+trap 'rm -rf "$TMP"' EXIT
+
+make -C "$ROOT" >/dev/null
+
+fail() {
+    echo "not ok - $1" >&2
+    if [ -f "$TMP/$1.out" ]; then
+        echo "stdout:" >&2
+        sed 's/^/  /' "$TMP/$1.out" >&2
+    fi
+    if [ -f "$TMP/$1.err" ]; then
+        echo "stderr:" >&2
+        sed 's/^/  /' "$TMP/$1.err" >&2
+    fi
+    exit 1
+}
+
+run_case() {
+    name=$1
+    input=$2
+    expected_stdout=$3
+    expected_status=$4
+
+    set +e
+    printf "%s" "$input" | "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err"
+    status=$?
+    set -e
+
+    printf "%s" "$expected_stdout" >"$TMP/$name.expected"
+    cmp -s "$TMP/$name.expected" "$TMP/$name.out" || fail "$name"
+    [ "$status" -eq "$expected_status" ] || fail "$name"
+}
+
+run_case builtin_cd_pwd \
+"cd $TMP
+pwd
+" \
+"$TMP_PHYSICAL
+" \
+0
+
+run_case export_env_unset \
+"export SMALLSH_SMOKE=ok
+env | grep '^SMALLSH_SMOKE=ok$'
+unset SMALLSH_SMOKE
+env | grep '^SMALLSH_SMOKE=ok$'
+echo \$?
+" \
+"SMALLSH_SMOKE=ok
+1
+" \
+0
+
+run_case quote_expansion \
+"export WHO=world
+echo \"hello \$WHO\"
+echo '\$WHO'
+" \
+"hello world
+\$WHO
+" \
+0
+
+run_case last_status \
+"missing-small-shell-command
+echo \$?
+" \
+"127
+" \
+0
+
+run_case pipeline \
+"echo hello | tr a-z A-Z
+" \
+"HELLO
+" \
+0
+
+run_case redirection \
+"echo first > $TMP/redir.txt
+echo second >> $TMP/redir.txt
+cat < $TMP/redir.txt
+" \
+"first
+second
+" \
+0
+
+run_case heredoc \
+"export HD=beta
+cat <<EOF
+alpha
+\$HD
+EOF
+" \
+"alpha
+beta
+" \
+0
+
+run_case syntax_error_status \
+"echo |
+echo \$?
+" \
+"258
+" \
+0
+
+run_case non_interactive_stdin \
+"echo one
+echo two
+" \
+"one
+two
+" \
+0
+
+run_case exit_builtin \
+"exit 7
+echo never
+" \
+"" \
+7
+
+echo "ok - smoke"


## `fix(input): EOF와 입력 실패를 구분`

diff --git a/src/heredoc.c b/src/heredoc.c
index 8dab1bd..e6d4060 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -3,6 +3,7 @@
 #include "exec_internal.h"
 #include "runtime.h"
 
+#include <errno.h>
 #include <stdint.h>
 #include <stdio.h>
 #include <stdlib.h>
@@ -17,7 +18,7 @@ struct strbuf {
     size_t  cap;
 };
 
-char *shell_read_line(const char *prompt, int interactive);
+char *shell_read_line(const char *prompt, int interactive, int *failed);
 
 static int sb_init(struct strbuf *buf)
 {
@@ -187,17 +188,18 @@ static int delimiter_matches(const char *line, const char *encoded)
     return line[j] == '\0';
 }
 
-static void discard_heredoc(const char *delimiter, int interactive)
+static int discard_heredoc(const char *delimiter, int interactive)
 {
     for (;;) {
-        char *line;
+        char    *line;
+        int     failed;
 
-        line = shell_read_line("> ", interactive);
+        line = shell_read_line("> ", interactive, &failed);
         if (line == NULL)
-            return;
+            return failed;
         if (delimiter_matches(line, delimiter)) {
             free(line);
-            return;
+            return 0;
         }
         free(line);
     }
@@ -262,25 +264,34 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     char            *delimiter;
     int             quoted;
     int             interactive;
+    int             input_failed;
 
     interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     quoted = redir->heredoc_quoted;
     delimiter = dequote_runtime_word(redir->target);
     if (delimiter == NULL) {
-        discard_heredoc(redir->target, interactive);
+        (void)discard_heredoc(redir->target, interactive);
         return 1;
     }
     free(redir->target);
     redir->target = delimiter;
     if (sb_init(&body) != 0) {
-        discard_heredoc(redir->target, interactive);
+        (void)discard_heredoc(redir->target, interactive);
         return 1;
     }
     for (;;) {
         char *line;
 
-        line = shell_read_line("> ", interactive);
+        line = shell_read_line("> ", interactive, &input_failed);
         if (line == NULL) {
+            if (input_failed) {
+                fprintf(stderr, "small-shell: heredoc input: %s\n",
+                    strerror(errno));
+                if (discard_heredoc(redir->target, interactive) != 0)
+                    ctx->shell->running = 0;
+                sb_free(&body);
+                return 1;
+            }
             fprintf(stderr,
                 "small-shell: warning: here-document delimited by end-of-file (wanted `%s')\n",
                 redir->target);
@@ -292,7 +303,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
         }
         if (append_heredoc_body_line(ctx->shell, quoted, &body, line) != 0) {
             free(line);
-            discard_heredoc(redir->target, interactive);
+            (void)discard_heredoc(redir->target, interactive);
             sb_free(&body);
             return 1;
         }
@@ -326,8 +337,9 @@ int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
                 if (redir->type == REDIR_HEREDOC) {
                     if (!failed && read_heredoc(ctx, redir) != 0)
                         failed = 1;
-                    else if (failed)
-                        discard_heredoc(redir->target, interactive);
+                    else if (failed
+                        && discard_heredoc(redir->target, interactive) != 0)
+                        failed = 1;
                 }
                 redir = redir->next;
             }
diff --git a/src/input.c b/src/input.c
index 597e9e1..9c226ea 100644
--- a/src/input.c
+++ b/src/input.c
@@ -3,8 +3,11 @@
 #include "shell.h"
 #include "runtime.h"
 
+#include <errno.h>
+#include <stdint.h>
 #include <stdio.h>
 #include <stdlib.h>
+#include <string.h>
 #include <unistd.h>
 
 #ifdef USE_READLINE
@@ -12,77 +15,104 @@
 #include <readline/readline.h>
 #endif
 
-static char *read_plain_line(const char *prompt, int interactive)
+static char *read_plain_line(const char *prompt, int interactive, int *failed)
 {
-    size_t cap;
-    size_t len;
-    char *line;
-    int ch;
+    size_t  cap;
+    size_t  len;
+    char    *line;
 
+    *failed = 0;
     if (interactive && prompt != NULL) {
         fputs(prompt, stderr);
         fflush(stderr);
     }
-
     cap = 128;
     len = 0;
-    line = shell_malloc(cap);
-    if (line == NULL)
+    line = (char *)shell_malloc(cap);
+    if (line == NULL) {
+        *failed = 1;
         return NULL;
+    }
+    for (;;) {
+        unsigned char   ch;
+        ssize_t         count;
 
-    while ((ch = fgetc(stdin)) != EOF) {
-        char *grown;
-
+        count = shell_read(STDIN_FILENO, &ch, 1);
+        if (count < 0 && errno == EINTR)
+            continue;
+        if (count < 0) {
+            free(line);
+            *failed = 1;
+            return NULL;
+        }
+        if (count == 0) {
+            if (len == 0) {
+                free(line);
+                return NULL;
+            }
+            break;
+        }
         if (ch == '\n')
             break;
         if (len + 1 >= cap) {
+            char *grown;
+
+            if (cap > SIZE_MAX / 2) {
+                free(line);
+                errno = ENOMEM;
+                *failed = 1;
+                return NULL;
+            }
             cap *= 2;
-            grown = shell_realloc(line, cap);
+            grown = (char *)shell_realloc(line, cap);
             if (grown == NULL) {
                 free(line);
+                *failed = 1;
                 return NULL;
             }
             line = grown;
         }
         line[len++] = (char)ch;
     }
-
-    if (ch == EOF && len == 0) {
-        free(line);
-        return NULL;
-    }
-
     line[len] = '\0';
     return line;
 }
 
-char *shell_read_line(const char *prompt, int interactive)
+char *shell_read_line(const char *prompt, int interactive, int *failed)
 {
 #ifdef USE_READLINE
     if (interactive) {
         char *line;
 
+        *failed = 0;
         line = readline(prompt != NULL ? prompt : "");
         if (line != NULL && line[0] != '\0')
             add_history(line);
         return line;
     }
 #endif
-    return read_plain_line(prompt, interactive);
+    return read_plain_line(prompt, interactive, failed);
 }
 
 void shell_loop(t_shell *shell)
 {
-    int interactive;
-    char *line;
+    int     interactive;
+    char    *line;
 
     if (shell == NULL)
         return;
     interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     while (shell->running) {
-        line = shell_read_line("small-shell$ ", interactive);
-        if (line == NULL)
+        int failed;
+
+        line = shell_read_line("small-shell$ ", interactive, &failed);
+        if (line == NULL) {
+            if (failed) {
+                fprintf(stderr, "small-shell: input: %s\n", strerror(errno));
+                shell->last_status = 1;
+            }
             break;
+        }
         (void)shell_process_line(shell, line);
         free(line);
     }
diff --git a/src/runtime.c b/src/runtime.c
index 67405df..47d18f8 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -169,3 +169,8 @@ int shell_fileno(FILE *stream)
 {
     return fileno(stream);
 }
+
+ssize_t shell_read(int fd, void *buffer, size_t size)
+{
+    return read(fd, buffer, size);
+}
diff --git a/src/runtime.h b/src/runtime.h
index 285edc1..a78440d 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -18,5 +18,6 @@ int     shell_open(const char *path, int flags, mode_t mode);
 int     shell_fflush(FILE *stream);
 int     shell_fseek(FILE *stream, long offset, int whence);
 int     shell_fileno(FILE *stream);
+ssize_t shell_read(int fd, void *buffer, size_t size);
 
 #endif


## `docs: improve README with project visuals`

diff --git a/README.md b/README.md
index e6fe06d..748abf5 100644
--- a/README.md
+++ b/README.md
@@ -1,40 +1,182 @@
-# small-shell
+# Small Shell
 
-`small-shell`은 명령줄을 읽고 해석해 실행하는 과정을 C로 구현하며
-셸의 데이터 구조와 프로세스 수명을 학습하기 위한 프로젝트다.
+표준 입력에서 명령을 읽어 제한된 셸 문법을 실행하는 C99/POSIX 프로그램입니다. 토큰화와 파싱, 환경 변수 확장, 파이프라인, 조건 연결, heredoc, 리다이렉션, 내장 명령과 외부 프로세스 실행을 하나의 명령 반복 안에서 처리합니다.
 
-## 목표
+완전한 POSIX 셸을 복제하는 프로젝트는 아닙니다. 현재 구현이 실제로 지원하는 문법, 프로세스·파일 디스크립터 수명, 종료 상태와 signal 처리 범위를 명시적으로 구분합니다.
 
-- C99와 POSIX 인터페이스를 사용한다.
-- tokenizer, parser, expansion 단계를 명확히 분리한다.
-- builtin과 외부 명령의 실행 경계를 구분한다.
-- pipeline, redirection, heredoc의 자원 소유권을 추적한다.
-- 오류를 호출자에게 전달하고 부분 결과를 정리한다.
+![입력 한 줄이 parser와 파일 디스크립터 계획을 거쳐 부모 또는 자식에서 실행되는 과정](docs/images/parse-to-execution.svg)
 
-완성된 프로그램은 표준 입력에서 한 줄씩 읽어 제한된 셸 문법을 실행하는
-`small-shell` 실행 파일로 제공할 예정이다.
+## 한눈에 보기
 
-## 개발 규약
+| 항목 | 내용 |
+| --- | --- |
+| 실행 파일 | `small-shell` |
+| 언어·환경 | C99, POSIX 프로세스·파일 디스크립터 |
+| 입력 | 표준 입력의 명령 줄 |
+| 연결 연산자 | `|`, `;`, `&&`, `||` |
+| 리다이렉션 | `<`, `>`, `>>`, `<<` |
+| 내장 명령 | `echo`, `pwd`, `cd`, `env`, `export`, `unset`, `exit` |
+| 외부 명령 | `execvp`를 통한 `PATH` 검색 |
+| 선택 기능 | `USE_READLINE=1` 빌드 |
 
-- `-std=c99 -Wall -Wextra -Wpedantic` 경고 없이 빌드한다.
-- 한 변경은 하나의 명확한 책임만 다룬다.
-- 새 추상화는 실제 사용 경로와 함께 검증한다.
-- 파일 디스크립터, 동적 메모리와 자식 프로세스의 소유자를 명시한다.
-- 실패 경로도 정상 경로와 같은 수준으로 정리한다.
-- 동작을 추가한 뒤 독립적으로 재현 가능한 검증을 남긴다.
+## 빠른 시작
 
-## 예정 범위
+```sh
+make
+./small-shell
+```
 
-- 인용 문자열과 셸 연산자의 tokenization
-- pipeline과 조건 연결자를 포함한 parsing
-- 환경 변수와 종료 상태 expansion
-- 기본 builtin과 외부 명령 실행
-- 파일 redirection과 heredoc
+파이프로 명령을 전달할 수도 있습니다.
 
-background 실행, job control, subshell, glob과 완전한 POSIX shell 호환은
-이 프로젝트의 범위에 포함하지 않는다.
+```sh
+printf 'echo hello | tr a-z A-Z\n' | ./small-shell
+```
 
-## 검증 원칙
+대화형 입력에 Readline을 사용하려면 기존 객체 파일을 지운 뒤 다시 빌드합니다.
 
-각 단계는 그 시점에 존재하는 코드만으로 빌드할 수 있어야 한다. 기능이 연결된
-뒤에는 정상 동작뿐 아니라 입력·할당·시스템 호출 실패와 자원 수명도 검증한다.
+```sh
+make clean
+make USE_READLINE=1
+```
+
+지원되는 실행 인터페이스는 `./small-shell` 하나입니다. `-c`와 스크립트 파일 인자는 해석하지 않습니다.
+
+## 실행 예제
+
+```text
+$ ./small-shell
+small-shell$ export NAME=world
+small-shell$ echo "hello $NAME" | tr a-z A-Z
+HELLO WORLD
+small-shell$ false || echo recovered
+recovered
+small-shell$ printf 'alpha\nbeta\n' > sample.txt
+small-shell$ cat < sample.txt
+alpha
+beta
+```
+
+## 지원 문법
+
+| 영역 | 현재 동작 |
+| --- | --- |
+| 공백 | space, tab, newline, carriage return, vertical tab, form feed |
+| 단어 | 인용하지 않은 조각, 작은따옴표, 큰따옴표와 이들의 연속 결합 |
+| 확장 | `$NAME`, `$?`; 작은따옴표 안은 그대로 두고 큰따옴표 안은 확장 |
+| 파이프라인 | `|`로 연결한 명령 묶음 |
+| 조건 연결 | `;`, `&&`, `||`를 같은 우선순위로 왼쪽부터 처리 |
+| 리다이렉션 | `<`, `>`, `>>`, `<<` |
+| heredoc | 구분자 인용 여부에 따라 본문 변수 확장 결정 |
+
+끝의 `;`는 허용하지만 끝의 `&&`와 `||`, 빈 파이프라인, 대상이 없는 리다이렉션은 구문 오류입니다.
+
+일반 인자와 `<`, `>`, `>>` 대상은 해당 파이프라인을 실행하기 직전의 환경과 직전 종료 상태로 확장합니다. 따옴표 제거 뒤 필드 분리와 glob은 수행하지 않습니다.
+
+모든 heredoc은 조건 분기 결과와 관계없이 입력에 나타난 순서대로 먼저 수집합니다. 구분자가 조금이라도 인용되었다면 heredoc 본문의 `$NAME`과 `$?` 확장을 끕니다. 구분자 자체는 변수 확장하지 않습니다.
+
+## 명령 처리 과정
+
+```text
+입력 한 줄
+  -> 토큰화와 따옴표 상태 확인
+  -> 명령·파이프라인·조건 연결 파싱
+  -> 모든 heredoc 입력 수집
+  -> 실행할 조건 분기 선택
+  -> 인자와 리다이렉션 대상 확장
+  -> 파이프·파일 디스크립터 준비
+  -> 부모 내장 명령 또는 자식 프로세스 실행
+  -> 자식 회수와 파일 디스크립터 정리
+  -> 마지막 상태를 $?에 보존
+```
+
+`|`가 한 파이프라인을 먼저 묶습니다. `;`, `&&`, `||`는 같은 우선순위로 왼쪽부터 결합합니다.
+
+## 부모와 자식에서 실행되는 명령
+
+단독 파이프라인의 내장 명령과 리다이렉션만 있는 명령은 부모 프로세스에서 실행합니다. 그래야 `cd`, `export`, `unset`, `exit`가 현재 셸 상태를 바꿀 수 있습니다.
+
+파이프라인 안의 내장 명령과 모든 외부 명령은 자식 프로세스에서 실행합니다. 이 경우 자식 안의 디렉터리·환경 변경은 부모 셸에 남지 않습니다.
+
+```text
+단독 builtin
+  -> 부모에서 리다이렉션 적용
+  -> builtin 실행
+  -> 원래 파일 디스크립터 복원
+
+pipeline
+  -> 명령별 pipe 연결
+  -> 각 명령을 자식에서 실행
+  -> 부모가 pipe를 닫고 모든 자식을 회수
+```
+
+## 종료 상태
+
+| 상태 | 대표 의미 |
+| ---: | --- |
+| `0` | 성공 |
+| `1` | 준비·할당·시스템 호출 오류 또는 일반 내장 명령 실패 |
+| `2` | 잘못된 `exit` 숫자, 또는 내부 구문 상태 258의 프로세스 종료값 |
+| `126` | `execvp`가 `ENOENT` 이외의 오류로 실패 |
+| `127` | `execvp`가 `ENOENT`로 실패 |
+| `128 + n` | 마지막으로 기다린 자식이 signal `n`으로 종료 |
+| `0..255` | 정상 종료한 마지막 자식 또는 범위를 줄인 `exit N` |
+| `258` | 명령 반복 안에서 보존하는 lexer/parser 구문 오류 |
+
+같은 숫자를 외부 명령이 직접 반환할 수도 있으므로 숫자만 보고 원인을 유일하게 역추론할 수는 없습니다. 구문 오류 뒤 다음 명령은 `$? == 258`을 볼 수 있으며, 그 상태에서 EOF에 도달하면 운영체제에는 하위 8비트인 2가 반환됩니다.
+
+## signal 범위
+
+stdin과 stderr가 모두 터미널일 때만 프롬프트와 선택적 Readline 경로를 사용합니다. 이 판정은 입력 방식만 바꾸며 signal 정책을 바꾸지 않습니다.
+
+현재 제품에는 다음 기능이 없습니다.
+
+- `sigaction`을 이용한 프롬프트 복구
+- 별도 프로세스 그룹과 foreground terminal 제어
+- job control
+- signal에 중단된 heredoc을 복구해 새 프롬프트를 내는 동작
+
+`128 + signal`은 부모 셸이 signal을 복구했다는 뜻이 아니라 `waitpid`가 관찰한 자식 종료 상태의 변환입니다. 테스트 하네스가 사용하는 signal handler와 프로세스 그룹 정리는 제품 기능에 포함되지 않습니다.
+
+## 검증
+
+```sh
+make test
+make test-asan
+make test-ubsan
+make test-sanitizers-container
+```
+
+검증 범위는 다음을 포함합니다.
+
+- lexer와 parser의 정상·오류 입력
+- 따옴표와 변수 확장
+- `;`, `&&`, `||`와 파이프라인 결합
+- 부모·자식 내장 명령의 상태 차이
+- 리다이렉션과 heredoc 파일 디스크립터 수명
+- 외부 명령 종료와 signal 상태 변환
+- 할당·시스템 호출 실패 주입
+- 자식과 파일 디스크립터 누수 검사
+- 512 KiB 긴 입력의 실행 상한
+
+`make test-asan`과 `make test-ubsan`은 제품과 parser 검사용 실행 파일을 각각의 sanitizer로 빌드해 같은 핵심 시나리오를 실행합니다. 컨테이너 대상은 `gcc:13-bookworm`에서 소스를 읽기 전용으로 연결하고 임시 파일 시스템에서 두 sanitizer 경로를 실행합니다.
+
+## 문서
+
+- [해석과 실행](architecture/parsing-and-execution.md)
+- [프로세스와 파일 디스크립터 수명](architecture/process-and-fd-lifecycle.md)
+- [의미와 오류 범위](architecture/semantics-and-error-boundaries.md)
+
+## 제한 사항
+
+다음 기능은 지원하지 않습니다.
+
+- background 실행과 job control
+- 역슬래시 보호, `${VAR}`, 필드 분리, glob과 주석
+- 명령 치환, 산술 치환, subshell과 grouping
+- 숫자 파일 디스크립터와 stderr 리다이렉션
+- here-string
+- `-c`, 스크립트 파일 인자와 완전한 POSIX 셸 호환
+
+## 프로젝트 배경
+
+이 저장소는 42의 `minishell`에서 출발했습니다. 과제의 셸 구현 범위를 바탕으로 조건 연결, 명시적인 상태 계약, parser API 검사, 장애 주입, sanitizer와 파일 디스크립터·자식 수명 검증을 추가했습니다.
diff --git a/docs/images/parse-to-execution.svg b/docs/images/parse-to-execution.svg
new file mode 100644
index 0000000..8404152
--- /dev/null
+++ b/docs/images/parse-to-execution.svg
@@ -0,0 +1,20 @@
+<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 650" role="img" aria-labelledby="title desc">
+  <title id="title">Small shell parse and execution flow</title>
+  <desc id="desc">An input line is tokenized and parsed, all heredocs are collected, conditions choose a pipeline, and builtins run in the parent only when their state must persist.</desc>
+  <defs><marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#66d4ff"/></marker>
+  <style>.bg{fill:#0b1020}.box{fill:#151d31;stroke:#344463;stroke-width:2}.accent{fill:#112d3c;stroke:#66d4ff;stroke-width:2}.parent{fill:#173528;stroke:#6ee7a8;stroke-width:2}.child{fill:#3a2c17;stroke:#f6c665;stroke-width:2}.line{stroke:#66d4ff;stroke-width:3;fill:none;marker-end:url(#arrow)}.t{fill:#f4f7ff;font:600 21px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.s{fill:#abb9d2;font:16px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.m{fill:#dce8ff;font:16px ui-monospace,SFMono-Regular,Menlo,monospace}.label{fill:#66d4ff;font:600 15px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}</style></defs>
+  <rect class="bg" width="1200" height="650" rx="24"/><text class="t" x="55" y="58">From one command line to owned processes and file descriptors</text>
+  <rect class="box" x="55" y="110" width="180" height="105" rx="15"/><text class="t" x="91" y="153">input line</text><text class="m" x="87" y="187">quotes · $VAR</text>
+  <rect class="box" x="300" y="110" width="180" height="105" rx="15"/><text class="t" x="344" y="153">lexer</text><text class="s" x="334" y="187">token boundaries</text>
+  <rect class="box" x="545" y="110" width="180" height="105" rx="15"/><text class="t" x="588" y="153">parser</text><text class="m" x="576" y="187">| ; &amp;&amp; || redir</text>
+  <rect class="accent" x="790" y="110" width="180" height="105" rx="15"/><text class="t" x="823" y="153">heredocs</text><text class="s" x="816" y="187">collect all first</text>
+  <path class="line" d="M235 162 H290"/><path class="line" d="M480 162 H535"/><path class="line" d="M725 162 H780"/>
+  <rect class="box" x="790" y="285" width="180" height="105" rx="15"/><text class="t" x="814" y="328">condition</text><text class="s" x="818" y="362">select pipeline</text>
+  <rect class="box" x="545" y="285" width="180" height="105" rx="15"/><text class="t" x="575" y="328">expansion</text><text class="s" x="578" y="362">argv · redir path</text>
+  <rect class="accent" x="300" y="285" width="180" height="105" rx="15"/><text class="t" x="342" y="328">FD plan</text><text class="m" x="330" y="362">pipe · open · dup</text>
+  <path class="line" d="M880 215 V275"/><path class="line" d="M790 337 H735"/><path class="line" d="M545 337 H490"/>
+  <rect class="parent" x="120" y="480" width="400" height="115" rx="18"/><text class="t" x="164" y="523">parent execution</text><text class="m" x="164" y="556">cd · export · unset · exit</text><text class="s" x="164" y="582">apply redirection, run, restore FDs</text>
+  <rect class="child" x="680" y="480" width="400" height="115" rx="18"/><text class="t" x="724" y="523">child pipeline</text><text class="m" x="724" y="556">builtin | execvp | builtin</text><text class="s" x="724" y="582">close unused FDs, wait for every child</text>
+  <path class="line" d="M390 390 V470"/><path class="line" d="M390 390 C390 430 880 430 880 470"/>
+  <text class="label" x="242" y="447">single stateful builtin</text><text class="label" x="762" y="447">pipeline / external command</text>
+</svg>
