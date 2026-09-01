## `test(memory): 범위별 할당 실패 순회 검증`

diff --git a/Makefile b/Makefile
index 917eae7..ffb64c9 100644
--- a/Makefile
+++ b/Makefile
@@ -52,6 +52,7 @@ readline:
 test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET)
 	./tests/smoke.sh
 	./tests/faults.sh
+	./tests/allocation.sh
 	./$(PARSER_API_TARGET)
 
 clean:
diff --git a/src/exec.c b/src/exec.c
index 84e75cc..f6b2a3f 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -213,6 +213,7 @@ static int expand_one_pipeline(t_shell *shell, t_pipeline *pipeline)
 
     next = pipeline->next;
     pipeline->next = NULL;
+    shell_runtime_set_alloc_scope("expand");
     result = expand_pipeline(shell, pipeline);
     pipeline->next = next;
     return result;
@@ -228,6 +229,7 @@ static int execute_one_pipeline(t_shell *shell, t_pipeline *pipeline, const stru
         fprintf(stderr, "small-shell: allocation failure\n");
         return 1;
     }
+    shell_runtime_set_alloc_scope("execute");
     command = pipeline->commands;
     if (pipeline->command_count == 1
         && (command->argc == 0 || builtin_is_parent(command->argv[0])))
@@ -262,6 +264,7 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
 
     ctx.shell = shell;
     ctx.heredocs = NULL;
+    shell_runtime_set_alloc_scope("heredoc");
     return execute_pipeline_list_ctx(shell, pipeline, &ctx);
 }
 
@@ -282,7 +285,10 @@ int shell_process_line(t_shell *shell, const char *line)
     if (shell == NULL || line == NULL || line[0] == '\0')
         return shell != NULL ? shell->last_status : 1;
 
+    shell_runtime_begin_command();
+
     error = NULL;
+    shell_runtime_set_alloc_scope("token");
     errno = 0;
     tokens = tokenize_line(line, &error);
     if (error != NULL || (tokens == NULL && errno == ENOMEM)) {
@@ -294,6 +300,7 @@ int shell_process_line(t_shell *shell, const char *line)
         return shell->last_status;
     }
 
+    shell_runtime_set_alloc_scope("parser");
     errno = 0;
     pipelines = parse_tokens(tokens, &error);
     free_tokens(tokens);
@@ -310,6 +317,7 @@ int shell_process_line(t_shell *shell, const char *line)
 
     ctx.shell = shell;
     ctx.heredocs = NULL;
+    shell_runtime_set_alloc_scope("heredoc");
     if (exec_prepare_heredocs(&ctx, pipelines) != 0) {
         exec_heredoc_entries_free(ctx.heredocs);
         free_pipeline(pipelines);
diff --git a/src/heredoc.c b/src/heredoc.c
index e6d4060..0b2a90c 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -194,6 +194,7 @@ static int discard_heredoc(const char *delimiter, int interactive)
         char    *line;
         int     failed;
 
+        shell_runtime_set_alloc_scope("input");
         line = shell_read_line("> ", interactive, &failed);
         if (line == NULL)
             return failed;
@@ -267,6 +268,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     int             input_failed;
 
     interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
+    shell_runtime_set_alloc_scope("heredoc");
     quoted = redir->heredoc_quoted;
     delimiter = dequote_runtime_word(redir->target);
     if (delimiter == NULL) {
@@ -282,6 +284,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     for (;;) {
         char *line;
 
+        shell_runtime_set_alloc_scope("input");
         line = shell_read_line("> ", interactive, &input_failed);
         if (line == NULL) {
             if (input_failed) {
@@ -301,6 +304,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
             free(line);
             break;
         }
+        shell_runtime_set_alloc_scope("heredoc");
         if (append_heredoc_body_line(ctx->shell, quoted, &body, line) != 0) {
             free(line);
             (void)discard_heredoc(redir->target, interactive);
@@ -309,6 +313,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
         }
         free(line);
     }
+    shell_runtime_set_alloc_scope("heredoc");
     if (add_heredoc_entry(ctx, redir, body.data) != 0) {
         sb_free(&body);
         return 1;
diff --git a/src/input.c b/src/input.c
index c52f5d8..a69245c 100644
--- a/src/input.c
+++ b/src/input.c
@@ -106,6 +106,7 @@ void shell_loop(t_shell *shell)
     while (shell->running) {
         int failed;
 
+        shell_runtime_set_alloc_scope("command-input");
         line = shell_read_line("small-shell$ ", interactive, &failed);
         if (line == NULL) {
             if (failed) {
diff --git a/src/main.c b/src/main.c
index 04884bd..26c21e8 100644
--- a/src/main.c
+++ b/src/main.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "shell.h"
+#include "runtime.h"
 
 #include <errno.h>
 #include <stdio.h>
@@ -19,6 +20,7 @@ int main(int argc, char **argv, char **envp)
     (void)argc;
     (void)argv;
     errno = 0;
+    shell_runtime_set_alloc_scope("startup");
     shell.env = env_from_environ(envp);
     if (shell.env == NULL && envp != NULL && envp[0] != NULL
         && errno == ENOMEM) {
diff --git a/src/runtime.c b/src/runtime.c
index 3280381..4f52b42 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -12,6 +12,11 @@
 
 #ifdef SMALL_SHELL_TESTING
 
+static const char       *g_alloc_scope;
+static unsigned long    g_alloc_calls;
+static int              g_alloc_failed;
+static unsigned long    g_command_number;
+
 static int fail_call(const char *name, unsigned long *calls)
 {
     const char      *text;
@@ -40,10 +45,69 @@ static int fail_call(const char *name, unsigned long *calls)
     return target == *calls;
 }
 
+static int fail_allocation(void)
+{
+    const char      *command_text;
+    const char      *scope;
+    const char      *text;
+    char            *end;
+    unsigned long   target;
+    unsigned long   target_command;
+    int             repeat;
+
+    if (g_alloc_failed)
+        return 0;
+    command_text = getenv("SMALL_SHELL_FAIL_ALLOC_COMMAND");
+    target_command = command_text != NULL ? strtoul(command_text, NULL, 10) : 1;
+    if (target_command == 0 || g_command_number != target_command)
+        return 0;
+    scope = getenv("SMALL_SHELL_FAIL_ALLOC_SCOPE");
+    if (scope == NULL || g_alloc_scope == NULL
+        || strcmp(scope, g_alloc_scope) != 0)
+        return 0;
+    g_alloc_calls++;
+    text = getenv("SMALL_SHELL_FAIL_ALLOC");
+    target = 1;
+    if (text != NULL && text[0] != '\0') {
+        target = strtoul(text, &end, 10);
+        if (end == text || *end != '\0' || target == 0)
+            return 0;
+    }
+    repeat = getenv("SMALL_SHELL_FAIL_ALLOC_REPEAT") != NULL;
+    if ((!repeat && g_alloc_calls != target)
+        || (repeat && g_alloc_calls < target))
+        return 0;
+    if (!repeat)
+        g_alloc_failed = 1;
+    errno = ENOMEM;
+    return 1;
+}
+
+#endif
+
+void shell_runtime_begin_command(void)
+{
+#ifdef SMALL_SHELL_TESTING
+    g_command_number++;
+    g_alloc_calls = 0;
+#endif
+}
+
+void shell_runtime_set_alloc_scope(const char *scope)
+{
+#ifdef SMALL_SHELL_TESTING
+    g_alloc_scope = scope;
+#else
+    (void)scope;
 #endif
+}
 
 void *shell_malloc(size_t size)
 {
+#ifdef SMALL_SHELL_TESTING
+    if (fail_allocation())
+        return NULL;
+#endif
     return malloc(size);
 }
 
@@ -53,11 +117,19 @@ void *shell_calloc(size_t count, size_t size)
         errno = ENOMEM;
         return NULL;
     }
+#ifdef SMALL_SHELL_TESTING
+    if (fail_allocation())
+        return NULL;
+#endif
     return calloc(count, size);
 }
 
 void *shell_realloc(void *ptr, size_t size)
 {
+#ifdef SMALL_SHELL_TESTING
+    if (fail_allocation())
+        return NULL;
+#endif
     return realloc(ptr, size);
 }
 
diff --git a/src/runtime.h b/src/runtime.h
index c5703dd..3cefa17 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -6,6 +6,8 @@
 # include <sys/types.h>
 # include <sys/stat.h>
 
+void    shell_runtime_begin_command(void);
+void    shell_runtime_set_alloc_scope(const char *scope);
 void    *shell_malloc(size_t size);
 void    *shell_calloc(size_t count, size_t size);
 void    *shell_realloc(void *ptr, size_t size);
diff --git a/tests/allocation.sh b/tests/allocation.sh
new file mode 100755
index 0000000..7ea3b4b
--- /dev/null
+++ b/tests/allocation.sh
@@ -0,0 +1,212 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN=${SMALL_SHELL_TEST_BIN:-"$ROOT/small-shell-test"}
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-allocation.XXXXXX")
+
+trap 'rm -rf "$TMP"' EXIT HUP INT TERM
+
+fail()
+{
+    echo "not ok - $1" >&2
+    if [ -f "$TMP/$1.out" ]; then
+        sed 's/^/stdout: /' "$TMP/$1.out" >&2
+    fi
+    if [ -f "$TMP/$1.err" ]; then
+        sed 's/^/stderr: /' "$TMP/$1.err" >&2
+    fi
+    exit 1
+}
+
+sweep()
+{
+    name=$1
+    scope=$2
+    maximum=$3
+    input=$4
+    failed_output=$5
+    successful_output=$6
+    call=1
+    failures=0
+    successes=0
+
+    printf '%s' "$failed_output" >"$TMP/$name.failed"
+    printf '%s' "$successful_output" >"$TMP/$name.success"
+    while [ "$call" -le "$maximum" ]; do
+        set +e
+        printf '%s' "$input" | env -i \
+            PATH="$PATH" \
+            ALLOC_SWEEP=old \
+            HEREDOC_VALUE=expanded \
+            SMALL_SHELL_FAIL_ALLOC_SCOPE="$scope" \
+            SMALL_SHELL_FAIL_ALLOC="$call" \
+            "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err"
+        status=$?
+        set -e
+        [ "$status" -eq 0 ] || fail "$name"
+        if cmp -s "$TMP/$name.failed" "$TMP/$name.out"; then
+            failures=$((failures + 1))
+        elif cmp -s "$TMP/$name.success" "$TMP/$name.out"; then
+            successes=$((successes + 1))
+        else
+            fail "$name"
+        fi
+        call=$((call + 1))
+    done
+    [ "$failures" -gt 0 ] || fail "$name"
+    [ "$successes" -gt 0 ] || fail "$name"
+    echo "ok - allocation $name: failures=$failures successes=$successes maximum=$maximum"
+}
+
+sweep token token 40 \
+    'echo marker
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'marker
+0
+after
+'
+
+sweep parser parser 20 \
+    'echo marker
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'marker
+0
+after
+'
+
+sweep expand expand 40 \
+    'echo marker
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'marker
+0
+after
+'
+
+sweep heredoc_input input 6 \
+    'cat <<EOF
+body
+EOF
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'body
+0
+after
+'
+
+sweep heredoc_quoted heredoc 8 \
+    'cat <<'"'"'EOF'"'"'
+$HEREDOC_VALUE
+EOF
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    '$HEREDOC_VALUE
+0
+after
+'
+
+sweep heredoc_multiple heredoc 14 \
+    'cat <<ONE <<TWO
+first
+ONE
+second
+TWO
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'second
+0
+after
+'
+
+sweep heredoc_unquoted heredoc 8 \
+    'cat <<EOF
+$HEREDOC_VALUE
+EOF
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    'expanded
+0
+after
+'
+
+sweep execute_builtin execute 10 \
+    'export ALLOC_SWEEP=new
+echo $?
+echo $ALLOC_SWEEP
+' \
+    '1
+old
+' \
+    '0
+new
+'
+
+sweep execute_external execute 30 \
+    'true
+echo $?
+echo after
+' \
+    '1
+after
+' \
+    '0
+after
+'
+
+set +e
+printf 'cat <<EOF\nbody\nEOF\necho never\n' | env -i \
+    PATH="$PATH" \
+    SMALL_SHELL_FAIL_ALLOC_SCOPE=input \
+    SMALL_SHELL_FAIL_ALLOC=1 \
+    SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
+    "$BIN" >"$TMP/persistent-input.out" 2>"$TMP/persistent-input.err"
+status=$?
+set -e
+[ "$status" -eq 1 ] || fail persistent-input
+[ ! -s "$TMP/persistent-input.out" ] || fail persistent-input
+
+set +e
+printf 'echo hidden\n' | env -i \
+    PATH="$PATH" \
+    SMALL_SHELL_FAIL_ALLOC_SCOPE=token \
+    SMALL_SHELL_FAIL_ALLOC=1 \
+    SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
+    "$BIN" >"$TMP/persistent.out" 2>"$TMP/persistent.err"
+status=$?
+set -e
+[ "$status" -eq 1 ] || fail persistent
+[ ! -s "$TMP/persistent.out" ] || fail persistent
+grep -q 'allocation failure' "$TMP/persistent.err" || fail persistent
+
+echo "ok - allocation failures"
diff --git a/tests/faults.sh b/tests/faults.sh
index ec887ee..bbdcdb2 100755
--- a/tests/faults.sh
+++ b/tests/faults.sh
@@ -38,6 +38,27 @@ EOF
     cmp -s "$TMP/$name.expected" "$TMP/$name.out" || fail "$name"
 }
 
+run_alloc_fault()
+{
+    name=$1
+    scope=$2
+    call=$3
+    input=$4
+    expected=$5
+
+    set +e
+    env SMALL_SHELL_FAIL_ALLOC_SCOPE="$scope" \
+        SMALL_SHELL_FAIL_ALLOC="$call" \
+        "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
+$input
+EOF
+    status=$?
+    set -e
+    [ "$status" -eq 0 ] || fail "$name"
+    printf '%s' "$expected" >"$TMP/$name.expected"
+    cmp -s "$TMP/$name.expected" "$TMP/$name.out" || fail "$name"
+}
+
 run_fault pipe_second SMALL_SHELL_FAIL_PIPE 2 \
     'printf alpha | cat | cat
 echo $?' \
@@ -124,6 +145,54 @@ echo after' \
 after
 '
 
+run_alloc_fault alloc_token token 1 \
+    'echo hidden
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_alloc_fault alloc_parser_node parser 1 \
+    'echo hidden
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_alloc_fault alloc_parser_argument parser 4 \
+    'echo hidden
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_alloc_fault alloc_expand expand 2 \
+    'echo hidden
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_alloc_fault alloc_parent_builtin execute 4 \
+    'export ALLOC_TEST=value
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_alloc_fault alloc_external_env execute 2 \
+    'true
+echo $?
+echo after' \
+    '1
+after
+'
+
 set +e
 env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
     "$BIN" >"$TMP/persistent-restore.out" \


## `test(io): read·write와 heredoc 입력 실패 검증`

diff --git a/src/runtime.c b/src/runtime.c
index 4f52b42..7279baf 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -244,11 +244,27 @@ int shell_fileno(FILE *stream)
 
 ssize_t shell_read(int fd, void *buffer, size_t size)
 {
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_READ", &calls)) {
+        errno = EIO;
+        return -1;
+    }
+#endif
     return read(fd, buffer, size);
 }
 
 ssize_t shell_write(int fd, const void *buffer, size_t size)
 {
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_WRITE", &calls)) {
+        errno = EIO;
+        return -1;
+    }
+#endif
     return write(fd, buffer, size);
 }
 
diff --git a/tests/faults.sh b/tests/faults.sh
index bbdcdb2..b77179d 100755
--- a/tests/faults.sh
+++ b/tests/faults.sh
@@ -59,6 +59,23 @@ EOF
     cmp -s "$TMP/$name.expected" "$TMP/$name.out" || fail "$name"
 }
 
+run_exit_fault()
+{
+    name=$1
+    variable=$2
+    call=$3
+    input=$4
+    expected_status=$5
+
+    set +e
+    printf '%s' "$input" | env "$variable=$call" "$BIN" \
+        >"$TMP/$name.out" 2>"$TMP/$name.err"
+    status=$?
+    set -e
+    [ "$status" -eq "$expected_status" ] || fail "$name"
+    [ ! -s "$TMP/$name.out" ] || fail "$name"
+}
+
 run_fault pipe_second SMALL_SHELL_FAIL_PIPE 2 \
     'printf alpha | cat | cat
 echo $?' \
@@ -193,6 +210,52 @@ echo after' \
 after
 '
 
+run_fault write_stdout SMALL_SHELL_FAIL_WRITE 1 \
+    'echo hidden
+echo $?' \
+    '1
+'
+
+run_exit_fault read_input SMALL_SHELL_FAIL_READ 1 \
+    'echo hidden
+' \
+    1
+
+run_fault heredoc_read_failure SMALL_SHELL_FAIL_READ 11 \
+    'cat <<EOF
+body
+EOF
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_fault multiple_heredoc_read_failure SMALL_SHELL_FAIL_READ 17 \
+    'cat <<ONE <<TWO
+first
+ONE
+second
+TWO
+echo $?
+echo after' \
+    '1
+after
+'
+
+set +e
+printf 'cat <<EOF
+body
+EOF
+echo never
+' | env \
+    SMALL_SHELL_FAIL_READ=11 SMALL_SHELL_FAIL_READ_REPEAT=1 \
+    "$BIN" >"$TMP/persistent-read.out" 2>"$TMP/persistent-read.err"
+status=$?
+set -e
+[ "$status" -eq 1 ] || fail persistent-read
+[ ! -s "$TMP/persistent-read.out" ] || fail persistent-read
+
 set +e
 env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
     "$BIN" >"$TMP/persistent-restore.out" \


## `fix(exec): pipe 생성 실패 시 PID 배열 해제`

diff --git a/src/exec.c b/src/exec.c
index a890b5c..2500275 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -157,6 +157,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
                 fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
                 close_pipes(pipes, pipe_count);
                 free(pipes);
+                free(pids);
                 return 1;
             }
         }
