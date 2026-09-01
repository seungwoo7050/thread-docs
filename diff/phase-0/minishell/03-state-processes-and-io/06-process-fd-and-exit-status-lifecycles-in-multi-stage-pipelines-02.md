## `test(exec): pipe·fork·wait 실패 회귀 검증`

diff --git a/.gitignore b/.gitignore
index 690532f..b677ed8 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,4 +1,5 @@
 small-shell
+/small-shell-test
 /tests/parser-api
 *.o
 *.d
diff --git a/Makefile b/Makefile
index 5bfcaec..917eae7 100644
--- a/Makefile
+++ b/Makefile
@@ -19,6 +19,8 @@ SRCS := \
 	src/redirection.c \
 	src/builtin.c
 OBJS := $(SRCS:.c=.o)
+TEST_OBJS := $(SRCS:.c=.test.o)
+TEST_TARGET := small-shell-test
 PARSER_API_TARGET := tests/parser-api
 
 ifeq ($(USE_READLINE),1)
@@ -34,6 +36,12 @@ $(TARGET): $(OBJS)
 %.o: %.c
 	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<
 
+%.test.o: %.c
+	$(CC) $(CPPFLAGS) -DSMALL_SHELL_TESTING $(CFLAGS) -c -o $@ $<
+
+$(TEST_TARGET): $(TEST_OBJS)
+	$(CC) $(LDFLAGS) -o $@ $(TEST_OBJS) $(LDLIBS)
+
 $(PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
 	$(CC) $(CPPFLAGS) $(CFLAGS) $(LDFLAGS) -o $@ \
 		tests/parser_api.c $(filter-out src/main.c,$(SRCS)) $(LDLIBS)
@@ -41,11 +49,12 @@ $(PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
 readline:
 	$(MAKE) USE_READLINE=1
 
-test: $(TARGET) $(PARSER_API_TARGET)
+test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET)
 	./tests/smoke.sh
+	./tests/faults.sh
 	./$(PARSER_API_TARGET)
 
 clean:
-	rm -f $(TARGET) $(PARSER_API_TARGET) $(OBJS)
+	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(OBJS) $(TEST_OBJS)
 
 .PHONY: all readline test clean
diff --git a/tests/faults.sh b/tests/faults.sh
new file mode 100755
index 0000000..9c994f0
--- /dev/null
+++ b/tests/faults.sh
@@ -0,0 +1,59 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN=${SMALL_SHELL_TEST_BIN:-"$ROOT/small-shell-test"}
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-faults.XXXXXX")
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
+run_fault()
+{
+    name=$1
+    variable=$2
+    call=$3
+    input=$4
+    expected=$5
+
+    set +e
+    env "$variable=$call" "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
+$input
+EOF
+    status=$?
+    set -e
+    [ "$status" -eq 0 ] || fail "$name"
+    printf '%s' "$expected" >"$TMP/$name.expected"
+    cmp -s "$TMP/$name.expected" "$TMP/$name.out" || fail "$name"
+}
+
+run_fault pipe_second SMALL_SHELL_FAIL_PIPE 2 \
+    'printf alpha | cat | cat
+echo $?' \
+    '1
+'
+
+run_fault fork_second SMALL_SHELL_FAIL_FORK 2 \
+    'sleep 30 | cat | cat
+echo $?' \
+    '1
+'
+
+run_fault waitpid_first SMALL_SHELL_FAIL_WAITPID 1 \
+    'printf alpha | cat > /dev/null
+echo $?' \
+    '1
+'
+
+echo "ok - pipeline faults"


## `test(lifecycle): FD와 자식 프로세스 누수 검증`

diff --git a/Makefile b/Makefile
index 1be74ae..19efba4 100644
--- a/Makefile
+++ b/Makefile
@@ -57,6 +57,7 @@ test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./tests/smoke.sh
 	./tests/faults.sh
 	./tests/allocation.sh
+	./tests/lifecycle.sh
 	./$(PARSER_API_TARGET)
 
 clean:
diff --git a/src/exec.c b/src/exec.c
index f6b2a3f..a890b5c 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -194,6 +194,13 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
 
     if (wait_error)
         result = 1;
+#ifdef SMALL_SHELL_TESTING
+    if (getenv("SMALL_SHELL_CHECK_CHILDREN") != NULL
+        && !shell_children_reaped()) {
+        fprintf(stderr, "small-shell: unreaped child process\n");
+        result = 1;
+    }
+#endif
     free(pids);
     free(pipes);
     return spawned == pipeline->command_count ? result : 1;
diff --git a/src/runtime.c b/src/runtime.c
index 7279baf..c40e9b1 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -172,6 +172,31 @@ pid_t shell_waitpid(pid_t pid, int *status, int options)
     return waitpid(pid, status, options);
 }
 
+#ifdef SMALL_SHELL_TESTING
+int shell_children_reaped(void)
+{
+    int     status;
+    int     found;
+    pid_t   pid;
+
+    found = 0;
+    for (;;) {
+        pid = waitpid(-1, &status, WNOHANG);
+        if (pid > 0) {
+            found = 1;
+            continue;
+        }
+        if (pid == 0)
+            return 0;
+        if (errno == EINTR)
+            continue;
+        if (errno == ECHILD)
+            return !found;
+        return 0;
+    }
+}
+#endif
+
 int shell_dup(int fd)
 {
 #ifdef SMALL_SHELL_TESTING
diff --git a/src/runtime.h b/src/runtime.h
index 3cefa17..94b83f3 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -14,6 +14,9 @@ void    *shell_realloc(void *ptr, size_t size);
 int     shell_pipe(int fds[2]);
 pid_t   shell_fork(void);
 pid_t   shell_waitpid(pid_t pid, int *status, int options);
+#ifdef SMALL_SHELL_TESTING
+int     shell_children_reaped(void);
+#endif
 int     shell_dup(int fd);
 int     shell_dup2(int oldfd, int newfd);
 int     shell_open(const char *path, int flags, mode_t mode);
diff --git a/tests/lifecycle.sh b/tests/lifecycle.sh
new file mode 100755
index 0000000..3847d72
--- /dev/null
+++ b/tests/lifecycle.sh
@@ -0,0 +1,97 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN=${SMALL_SHELL_TEST_BIN:-"$ROOT/small-shell-test"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-lifecycle.XXXXXX")
+runner_pid=
+child_pid=
+
+cleanup()
+{
+    if [ -n "$runner_pid" ]; then
+        kill -KILL "$runner_pid" 2>/dev/null || :
+        wait "$runner_pid" 2>/dev/null || :
+    fi
+    if [ -n "$child_pid" ]; then
+        kill -KILL "$child_pid" 2>/dev/null || :
+    fi
+    rm -rf "$TMP"
+}
+
+trap cleanup EXIT HUP INT TERM
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
+: >"$TMP/fd-pressure.in"
+i=0
+while [ "$i" -lt 60 ]; do
+    printf 'echo parent > %s/parent\n' "$TMP" >>"$TMP/fd-pressure.in"
+    printf 'printf child | cat | cat > %s/pipeline\n' "$TMP" \
+        >>"$TMP/fd-pressure.in"
+    printf 'cat < %s/pipeline > %s/copy\n' "$TMP" "$TMP" \
+        >>"$TMP/fd-pressure.in"
+    i=$((i + 1))
+done
+printf 'echo fd-ok\n' >>"$TMP/fd-pressure.in"
+
+set +e
+(
+    ulimit -n 48
+    SMALL_SHELL_CHECK_CHILDREN=1 "$TIMEOUT" 20 "$BIN" \
+        <"$TMP/fd-pressure.in" >"$TMP/fd-pressure.out" \
+        2>"$TMP/fd-pressure.err"
+)
+status=$?
+set -e
+[ "$status" -eq 0 ] || fail fd-pressure
+printf 'fd-ok\n' >"$TMP/fd-pressure.expected"
+cmp -s "$TMP/fd-pressure.expected" "$TMP/fd-pressure.out" \
+    || fail fd-pressure
+[ ! -s "$TMP/fd-pressure.err" ] || fail fd-pressure
+
+printf 'sleep 30 | cat | cat\n' >"$TMP/timeout.in"
+set +e
+"$TIMEOUT" 1 "$BIN" <"$TMP/timeout.in" >"$TMP/timeout.out" \
+    2>"$TMP/timeout.err"
+status=$?
+set -e
+[ "$status" -eq 124 ] || fail timeout
+
+"$TIMEOUT" 20 /bin/sh -c \
+    'printf "%s\n" "$$" >"$1"; exec sleep 30' \
+    timeout-child "$TMP/child.pid" &
+runner_pid=$!
+i=0
+while [ ! -s "$TMP/child.pid" ] && [ "$i" -lt 100 ]; do
+    sleep 0.01
+    i=$((i + 1))
+done
+[ -s "$TMP/child.pid" ] || fail external-signal
+child_pid=$(sed -n '1p' "$TMP/child.pid")
+kill -TERM "$runner_pid"
+set +e
+wait "$runner_pid"
+status=$?
+set -e
+runner_pid=
+[ "$status" -eq 143 ] || fail external-signal
+set +e
+kill -0 "$child_pid" 2>/dev/null
+alive=$?
+set -e
+[ "$alive" -ne 0 ] || fail external-signal
+child_pid=
+
+echo "ok - lifecycle"


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
