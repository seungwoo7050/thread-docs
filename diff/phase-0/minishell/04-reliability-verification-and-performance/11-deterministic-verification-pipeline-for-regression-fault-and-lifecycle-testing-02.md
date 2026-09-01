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


## `build(test): 테스트 시간 제한 하네스 추가`

diff --git a/.gitignore b/.gitignore
index b677ed8..fae90d7 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,5 +1,6 @@
 small-shell
 /small-shell-test
+/tests/timeout-runner
 /tests/parser-api
 *.o
 *.d
diff --git a/Makefile b/Makefile
index ffb64c9..1be74ae 100644
--- a/Makefile
+++ b/Makefile
@@ -22,6 +22,7 @@ OBJS := $(SRCS:.c=.o)
 TEST_OBJS := $(SRCS:.c=.test.o)
 TEST_TARGET := small-shell-test
 PARSER_API_TARGET := tests/parser-api
+TIMEOUT_TARGET := tests/timeout-runner
 
 ifeq ($(USE_READLINE),1)
 CPPFLAGS += -DUSE_READLINE
@@ -46,16 +47,20 @@ $(PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
 	$(CC) $(CPPFLAGS) $(CFLAGS) $(LDFLAGS) -o $@ \
 		tests/parser_api.c $(filter-out src/main.c,$(SRCS)) $(LDLIBS)
 
+$(TIMEOUT_TARGET): tests/timeout_runner.c
+	$(CC) $(CPPFLAGS) $(CFLAGS) -o $@ $<
+
 readline:
 	$(MAKE) USE_READLINE=1
 
-test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET)
+test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./tests/smoke.sh
 	./tests/faults.sh
 	./tests/allocation.sh
 	./$(PARSER_API_TARGET)
 
 clean:
-	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(OBJS) $(TEST_OBJS)
+	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
+		$(OBJS) $(TEST_OBJS)
 
 .PHONY: all readline test clean
diff --git a/tests/allocation.sh b/tests/allocation.sh
index 7ea3b4b..ad61087 100755
--- a/tests/allocation.sh
+++ b/tests/allocation.sh
@@ -3,6 +3,7 @@ set -eu
 
 ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
 BIN=${SMALL_SHELL_TEST_BIN:-"$ROOT/small-shell-test"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
 TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-allocation.XXXXXX")
 
 trap 'rm -rf "$TMP"' EXIT HUP INT TERM
@@ -33,15 +34,17 @@ sweep()
 
     printf '%s' "$failed_output" >"$TMP/$name.failed"
     printf '%s' "$successful_output" >"$TMP/$name.success"
+    printf '%s' "$input" >"$TMP/$name.in"
     while [ "$call" -le "$maximum" ]; do
         set +e
-        printf '%s' "$input" | env -i \
+        env -i \
             PATH="$PATH" \
             ALLOC_SWEEP=old \
             HEREDOC_VALUE=expanded \
             SMALL_SHELL_FAIL_ALLOC_SCOPE="$scope" \
             SMALL_SHELL_FAIL_ALLOC="$call" \
-            "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err"
+            "$TIMEOUT" 5 "$BIN" <"$TMP/$name.in" \
+            >"$TMP/$name.out" 2>"$TMP/$name.err"
         status=$?
         set -e
         [ "$status" -eq 0 ] || fail "$name"
@@ -185,24 +188,29 @@ after
 '
 
 set +e
-printf 'cat <<EOF\nbody\nEOF\necho never\n' | env -i \
+printf 'cat <<EOF\nbody\nEOF\necho never\n' >"$TMP/persistent-input.in"
+env -i \
     PATH="$PATH" \
     SMALL_SHELL_FAIL_ALLOC_SCOPE=input \
     SMALL_SHELL_FAIL_ALLOC=1 \
     SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
-    "$BIN" >"$TMP/persistent-input.out" 2>"$TMP/persistent-input.err"
+    "$TIMEOUT" 5 "$BIN" <"$TMP/persistent-input.in" \
+    >"$TMP/persistent-input.out" 2>"$TMP/persistent-input.err"
 status=$?
 set -e
 [ "$status" -eq 1 ] || fail persistent-input
 [ ! -s "$TMP/persistent-input.out" ] || fail persistent-input
 
+
 set +e
-printf 'echo hidden\n' | env -i \
+printf 'echo hidden\n' >"$TMP/persistent.in"
+env -i \
     PATH="$PATH" \
     SMALL_SHELL_FAIL_ALLOC_SCOPE=token \
     SMALL_SHELL_FAIL_ALLOC=1 \
     SMALL_SHELL_FAIL_ALLOC_REPEAT=1 \
-    "$BIN" >"$TMP/persistent.out" 2>"$TMP/persistent.err"
+    "$TIMEOUT" 5 "$BIN" <"$TMP/persistent.in" \
+    >"$TMP/persistent.out" 2>"$TMP/persistent.err"
 status=$?
 set -e
 [ "$status" -eq 1 ] || fail persistent
diff --git a/tests/faults.sh b/tests/faults.sh
index b77179d..ca174a8 100755
--- a/tests/faults.sh
+++ b/tests/faults.sh
@@ -3,6 +3,7 @@ set -eu
 
 ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
 BIN=${SMALL_SHELL_TEST_BIN:-"$ROOT/small-shell-test"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
 TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-faults.XXXXXX")
 
 trap 'rm -rf "$TMP"' EXIT HUP INT TERM
@@ -28,7 +29,8 @@ run_fault()
     expected=$5
 
     set +e
-    env "$variable=$call" "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
+    env "$variable=$call" "$TIMEOUT" 5 "$BIN" \
+        >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
 $input
 EOF
     status=$?
@@ -49,7 +51,7 @@ run_alloc_fault()
     set +e
     env SMALL_SHELL_FAIL_ALLOC_SCOPE="$scope" \
         SMALL_SHELL_FAIL_ALLOC="$call" \
-        "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
+        "$TIMEOUT" 5 "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err" <<EOF
 $input
 EOF
     status=$?
@@ -68,8 +70,9 @@ run_exit_fault()
     expected_status=$5
 
     set +e
-    printf '%s' "$input" | env "$variable=$call" "$BIN" \
-        >"$TMP/$name.out" 2>"$TMP/$name.err"
+    printf '%s' "$input" >"$TMP/$name.in"
+    env "$variable=$call" "$TIMEOUT" 5 "$BIN" \
+        <"$TMP/$name.in" >"$TMP/$name.out" 2>"$TMP/$name.err"
     status=$?
     set -e
     [ "$status" -eq "$expected_status" ] || fail "$name"
@@ -248,17 +251,20 @@ printf 'cat <<EOF
 body
 EOF
 echo never
-' | env \
+' >"$TMP/persistent-read.in"
+env \
     SMALL_SHELL_FAIL_READ=11 SMALL_SHELL_FAIL_READ_REPEAT=1 \
-    "$BIN" >"$TMP/persistent-read.out" 2>"$TMP/persistent-read.err"
+    "$TIMEOUT" 5 "$BIN" <"$TMP/persistent-read.in" \
+    >"$TMP/persistent-read.out" 2>"$TMP/persistent-read.err"
 status=$?
 set -e
 [ "$status" -eq 1 ] || fail persistent-read
 [ ! -s "$TMP/persistent-read.out" ] || fail persistent-read
 
+
 set +e
 env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
-    "$BIN" >"$TMP/persistent-restore.out" \
+    "$TIMEOUT" 5 "$BIN" >"$TMP/persistent-restore.out" \
     2>"$TMP/persistent-restore.err" <<EOF
 echo hidden > $TMP/persistent-restore-file
 echo never
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 39ef5cb..5d96cb4 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -3,6 +3,7 @@ set -eu
 
 ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
 BIN=${SMALL_SHELL_BIN:-"$ROOT/small-shell"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
 TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell.XXXXXX")
 TMP_PHYSICAL=$(CDPATH= cd -- "$TMP" && pwd -P)
 
@@ -29,8 +30,10 @@ run_case() {
     expected_stdout=$3
     expected_status=$4
 
+    printf "%s" "$input" >"$TMP/$name.in"
     set +e
-    printf "%s" "$input" | "$BIN" >"$TMP/$name.out" 2>"$TMP/$name.err"
+    "$TIMEOUT" 5 "$BIN" <"$TMP/$name.in" \
+        >"$TMP/$name.out" 2>"$TMP/$name.err"
     status=$?
     set -e
 
diff --git a/tests/timeout_runner.c b/tests/timeout_runner.c
new file mode 100644
index 0000000..e26c8bd
--- /dev/null
+++ b/tests/timeout_runner.c
@@ -0,0 +1,176 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include <errno.h>
+#include <signal.h>
+#include <stdio.h>
+#include <stdlib.h>
+#include <sys/types.h>
+#include <sys/wait.h>
+#include <time.h>
+#include <unistd.h>
+
+static volatile sig_atomic_t pending_signal;
+
+static void record_signal(int signal_number)
+{
+    if (pending_signal == 0)
+        pending_signal = signal_number;
+}
+
+static int prepare_signals(sigset_t *blocked, sigset_t *previous)
+{
+    struct sigaction action;
+
+    sigemptyset(blocked);
+    sigaddset(blocked, SIGHUP);
+    sigaddset(blocked, SIGINT);
+    sigaddset(blocked, SIGTERM);
+    if (sigprocmask(SIG_BLOCK, blocked, previous) != 0)
+        return 1;
+    sigemptyset(&action.sa_mask);
+    action.sa_handler = record_signal;
+    action.sa_flags = 0;
+    if (sigaction(SIGHUP, &action, NULL) != 0
+        || sigaction(SIGINT, &action, NULL) != 0
+        || sigaction(SIGTERM, &action, NULL) != 0) {
+        (void)sigprocmask(SIG_SETMASK, previous, NULL);
+        return 1;
+    }
+    return 0;
+}
+
+static int prepare_child(const sigset_t *previous)
+{
+    struct sigaction action;
+
+    sigemptyset(&action.sa_mask);
+    action.sa_handler = SIG_DFL;
+    action.sa_flags = 0;
+    if (sigaction(SIGHUP, &action, NULL) != 0
+        || sigaction(SIGINT, &action, NULL) != 0
+        || sigaction(SIGTERM, &action, NULL) != 0
+        || sigprocmask(SIG_SETMASK, previous, NULL) != 0)
+        return 1;
+    return 0;
+}
+
+static long long monotonic_milliseconds(void)
+{
+    struct timespec now;
+
+    if (clock_gettime(CLOCK_MONOTONIC, &now) != 0)
+        return -1;
+    return (long long)now.tv_sec * 1000LL + now.tv_nsec / 1000000LL;
+}
+
+static int child_status(int status)
+{
+    if (WIFEXITED(status))
+        return WEXITSTATUS(status);
+    if (WIFSIGNALED(status))
+        return 128 + WTERMSIG(status);
+    return 1;
+}
+
+static int parse_seconds(const char *text, long *seconds)
+{
+    char *end;
+    long value;
+
+    errno = 0;
+    value = strtol(text, &end, 10);
+    if (text == end || *end != '\0' || errno == ERANGE
+        || value <= 0 || value > 3600)
+        return 1;
+    *seconds = value;
+    return 0;
+}
+
+static void terminate_child_group(pid_t pid)
+{
+    pid_t waited;
+
+    (void)kill(-pid, SIGKILL);
+    (void)kill(pid, SIGKILL);
+    do {
+        waited = waitpid(pid, NULL, 0);
+    } while (waited < 0 && errno == EINTR);
+}
+
+int main(int argc, char **argv)
+{
+    long        seconds;
+    long long   deadline;
+    pid_t       pid;
+    sigset_t    blocked;
+    sigset_t    previous;
+
+    if (argc < 3 || parse_seconds(argv[1], &seconds) != 0) {
+        fprintf(stderr, "usage: timeout-runner SECONDS COMMAND [ARG...]\n");
+        return 2;
+    }
+    deadline = monotonic_milliseconds();
+    if (deadline < 0)
+        return 2;
+    deadline += (long long)seconds * 1000LL;
+    if (prepare_signals(&blocked, &previous) != 0)
+        return 2;
+    pid = fork();
+    if (pid < 0) {
+        (void)sigprocmask(SIG_SETMASK, &previous, NULL);
+        return 2;
+    }
+    if (pid == 0) {
+        (void)setpgid(0, 0);
+        if (prepare_child(&previous) != 0)
+            _exit(126);
+        execvp(argv[2], &argv[2]);
+        _exit(errno == ENOENT ? 127 : 126);
+    }
+    if (setpgid(pid, pid) != 0 && errno != EACCES && errno != ESRCH) {
+        terminate_child_group(pid);
+        (void)sigprocmask(SIG_SETMASK, &previous, NULL);
+        return 2;
+    }
+    if (sigprocmask(SIG_SETMASK, &previous, NULL) != 0) {
+        terminate_child_group(pid);
+        return 2;
+    }
+    for (;;) {
+        int     status;
+        int     signal_number;
+        long long now;
+        pid_t   waited;
+
+        signal_number = pending_signal;
+        if (signal_number != 0) {
+            terminate_child_group(pid);
+            return 128 + signal_number;
+        }
+        waited = waitpid(pid, &status, WNOHANG);
+        if (waited == pid)
+            return child_status(status);
+        if (waited < 0 && errno != EINTR) {
+            terminate_child_group(pid);
+            return 2;
+        }
+        now = monotonic_milliseconds();
+        if (now < 0) {
+            terminate_child_group(pid);
+            return 2;
+        }
+        if (now >= deadline) {
+            terminate_child_group(pid);
+            return 124;
+        }
+        {
+            struct timespec pause_time;
+
+            pause_time.tv_sec = 0;
+            pause_time.tv_nsec = 10000000L;
+            while (nanosleep(&pause_time, &pause_time) != 0
+                && errno == EINTR) {
+            }
+        }
+    }
+}


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


## `test(performance): 긴 입력 처리 시간 상한 검증`

diff --git a/Makefile b/Makefile
index 4070777..84dbfd7 100644
--- a/Makefile
+++ b/Makefile
@@ -60,6 +60,7 @@ test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./tests/allocation.sh
 	./tests/lifecycle.sh
 	./$(PARSER_API_TARGET)
+	./tests/performance.sh
 
 clean:
 	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
diff --git a/tests/performance.sh b/tests/performance.sh
new file mode 100755
index 0000000..af87606
--- /dev/null
+++ b/tests/performance.sh
@@ -0,0 +1,38 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN=${SMALL_SHELL_BIN:-"$ROOT/small-shell"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-performance.XXXXXX")
+PAYLOAD_SIZE=524288
+
+trap 'rm -rf "$TMP"' EXIT HUP INT TERM
+
+fail()
+{
+    echo "not ok - long input performance" >&2
+    if [ -f "$TMP/long.err" ]; then
+        sed 's/^/stderr: /' "$TMP/long.err" >&2
+    fi
+    exit 1
+}
+
+{
+    printf 'echo '
+    dd if=/dev/zero bs=1024 count=512 2>/dev/null | tr '\000' x
+    printf '\n'
+} >"$TMP/long.in"
+
+set +e
+"$TIMEOUT" 5 "$BIN" <"$TMP/long.in" \
+    >"$TMP/long.out" 2>"$TMP/long.err"
+status=$?
+set -e
+
+[ "$status" -eq 0 ] || fail
+[ ! -s "$TMP/long.err" ] || fail
+output_size=$(wc -c <"$TMP/long.out" | tr -d '[:space:]')
+[ "$output_size" -eq $((PAYLOAD_SIZE + 1)) ] || fail
+
+echo "ok - long input performance"


