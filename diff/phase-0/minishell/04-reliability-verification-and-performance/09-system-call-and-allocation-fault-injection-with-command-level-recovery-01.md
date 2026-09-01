# 시스템 호출·할당 실패 주입과 명령 단위 복구

## `refactor(runtime): 프로세스 시스템 호출 경계 분리`

diff --git a/Makefile b/Makefile
index 304851f..5bfcaec 100644
--- a/Makefile
+++ b/Makefile
@@ -13,6 +13,7 @@ SRCS := \
 	src/expand.c \
 	src/env.c \
 	src/utils.c \
+	src/runtime.c \
 	src/exec.c \
 	src/heredoc.c \
 	src/redirection.c \
diff --git a/src/exec.c b/src/exec.c
index 53e89c0..750b65d 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "exec_internal.h"
+#include "runtime.h"
 
 #include <errno.h>
 #include <stdio.h>
@@ -111,7 +112,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
             pipes[i][1] = -1;
         }
         for (i = 0; i < pipe_count; i++) {
-            if (pipe(pipes[i]) < 0) {
+            if (shell_pipe(pipes[i]) < 0) {
                 fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
                 close_pipes(pipes, pipe_count);
                 free(pipes);
@@ -128,7 +129,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
     for (i = 0; i < pipeline->command_count && command != NULL; i++) {
         pid_t pid;
 
-        pid = fork();
+        pid = shell_fork();
         if (pid < 0) {
             fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
             break;
@@ -146,7 +147,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
         pid_t waited;
 
         do {
-            waited = waitpid(pids[i], &wait_status, 0);
+            waited = shell_waitpid(pids[i], &wait_status, 0);
         } while (waited < 0 && errno == EINTR);
         if (waited == pids[i] && i + 1 == pipeline->command_count)
             result = status_from_wait(wait_status);
diff --git a/src/runtime.c b/src/runtime.c
new file mode 100644
index 0000000..32fe72e
--- /dev/null
+++ b/src/runtime.c
@@ -0,0 +1,65 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "runtime.h"
+
+#include <errno.h>
+#include <stdlib.h>
+#include <sys/wait.h>
+#include <unistd.h>
+
+#ifdef SMALL_SHELL_TESTING
+
+static int fail_call(const char *name, unsigned long *calls)
+{
+    const char      *text;
+    char            *end;
+    unsigned long   target;
+
+    (*calls)++;
+    text = getenv(name);
+    if (text == NULL || text[0] == '\0')
+        return 0;
+    target = strtoul(text, &end, 10);
+    return (end != text && *end == '\0' && target != 0 && target == *calls);
+}
+
+#endif
+
+int shell_pipe(int fds[2])
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_PIPE", &calls)) {
+        errno = EMFILE;
+        return -1;
+    }
+#endif
+    return pipe(fds);
+}
+
+pid_t shell_fork(void)
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_FORK", &calls)) {
+        errno = EAGAIN;
+        return -1;
+    }
+#endif
+    return fork();
+}
+
+pid_t shell_waitpid(pid_t pid, int *status, int options)
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_WAITPID", &calls)) {
+        errno = EIO;
+        return -1;
+    }
+#endif
+    return waitpid(pid, status, options);
+}
diff --git a/src/runtime.h b/src/runtime.h
new file mode 100644
index 0000000..d9d7d5c
--- /dev/null
+++ b/src/runtime.h
@@ -0,0 +1,10 @@
+#ifndef RUNTIME_H
+# define RUNTIME_H
+
+# include <sys/types.h>
+
+int     shell_pipe(int fds[2]);
+pid_t   shell_fork(void);
+pid_t   shell_waitpid(pid_t pid, int *status, int options);
+
+#endif


## `fix(exec): 부분 생성 파이프라인의 자식과 FD 회수`

diff --git a/src/exec.c b/src/exec.c
index 750b65d..81a6545 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -4,6 +4,7 @@
 #include "runtime.h"
 
 #include <errno.h>
+#include <signal.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
@@ -87,6 +88,38 @@ static void run_child(t_shell *shell, const t_pipeline *pipeline, const t_comman
     }
 }
 
+static void terminate_children(const pid_t *pids, size_t count)
+{
+    size_t i;
+
+    for (i = 0; i < count; i++) {
+        if (pids[i] > 0 && kill(pids[i], SIGKILL) < 0 && errno != ESRCH)
+            fprintf(stderr, "small-shell: kill: %s\n", strerror(errno));
+    }
+}
+
+static int wait_for_child(pid_t pid, int *wait_status)
+{
+    int attempts;
+    int had_error;
+
+    attempts = 0;
+    had_error = 0;
+    while (attempts < 2) {
+        pid_t waited;
+
+        waited = shell_waitpid(pid, wait_status, 0);
+        if (waited == pid)
+            return had_error;
+        if (waited < 0 && errno == EINTR)
+            continue;
+        fprintf(stderr, "small-shell: waitpid: %s\n", strerror(errno));
+        had_error = 1;
+        attempts++;
+    }
+    return -1;
+}
+
 static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const struct exec_context *ctx)
 {
     size_t pipe_count;
@@ -96,12 +129,14 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
     size_t i;
     size_t spawned;
     int result;
+    int wait_error;
 
     pipe_count = pipeline->command_count - 1;
     pipes = NULL;
     pids = NULL;
     spawned = 0;
     result = 1;
+    wait_error = 0;
 
     if (pipe_count > 0) {
         pipes = (int (*)[2])malloc(sizeof(int[2]) * pipe_count);
@@ -142,17 +177,21 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
     }
 
     close_pipes(pipes, pipe_count);
+    if (spawned != pipeline->command_count)
+        terminate_children(pids, spawned);
     for (i = 0; i < spawned; i++) {
         int wait_status;
-        pid_t waited;
+        int wait_result;
 
-        do {
-            waited = shell_waitpid(pids[i], &wait_status, 0);
-        } while (waited < 0 && errno == EINTR);
-        if (waited == pids[i] && i + 1 == pipeline->command_count)
+        wait_result = wait_for_child(pids[i], &wait_status);
+        if (wait_result == 0 && i + 1 == pipeline->command_count)
             result = status_from_wait(wait_status);
+        if (wait_result != 0)
+            wait_error = 1;
     }
 
+    if (wait_error)
+        result = 1;
     free(pids);
     free(pipes);
     return spawned == pipeline->command_count ? result : 1;


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


## `refactor(runtime): FD 시스템 호출 경계 분리`

diff --git a/src/exec.c b/src/exec.c
index 81a6545..da74412 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -46,9 +46,9 @@ static void child_die(const char *what)
 static void run_child(t_shell *shell, const t_pipeline *pipeline, const t_command *command,
     const struct exec_context *ctx, int (*pipes)[2], size_t pipe_count, size_t index)
 {
-    if (index > 0 && dup2(pipes[index - 1][0], STDIN_FILENO) < 0)
+    if (index > 0 && shell_dup2(pipes[index - 1][0], STDIN_FILENO) < 0)
         child_die("dup2");
-    if (index + 1 < pipeline->command_count && dup2(pipes[index][1], STDOUT_FILENO) < 0)
+    if (index + 1 < pipeline->command_count && shell_dup2(pipes[index][1], STDOUT_FILENO) < 0)
         child_die("dup2");
     close_pipes(pipes, pipe_count);
 
diff --git a/src/redirection.c b/src/redirection.c
index a8a5b95..86a42aa 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "exec_internal.h"
+#include "runtime.h"
 
 #include <errno.h>
 #include <fcntl.h>
@@ -18,13 +19,13 @@ int exec_apply_redirections(const t_command *command,
         int fd;
 
         if (redir->type == REDIR_IN) {
-            fd = open(redir->target, O_RDONLY);
+            fd = shell_open(redir->target, O_RDONLY, 0);
             if (fd < 0) {
                 fprintf(stderr, "small-shell: %s: %s\n", redir->target,
                     strerror(errno));
                 return 1;
             }
-            if (dup2(fd, STDIN_FILENO) < 0) {
+            if (shell_dup2(fd, STDIN_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 close(fd);
                 return 1;
@@ -39,13 +40,13 @@ int exec_apply_redirections(const t_command *command,
                 flags |= O_TRUNC;
             else
                 flags |= O_APPEND;
-            fd = open(redir->target, flags, 0644);
+            fd = shell_open(redir->target, flags, 0644);
             if (fd < 0) {
                 fprintf(stderr, "small-shell: %s: %s\n", redir->target,
                     strerror(errno));
                 return 1;
             }
-            if (dup2(fd, STDOUT_FILENO) < 0) {
+            if (shell_dup2(fd, STDOUT_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 close(fd);
                 return 1;
@@ -70,7 +71,7 @@ int exec_apply_redirections(const t_command *command,
             }
             fflush(tmp);
             rewind(tmp);
-            if (dup2(fileno(tmp), STDIN_FILENO) < 0) {
+            if (shell_dup2(fileno(tmp), STDIN_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 fclose(tmp);
                 return 1;
@@ -84,8 +85,8 @@ int exec_apply_redirections(const t_command *command,
 
 static int save_stdio(int saved[2])
 {
-    saved[0] = dup(STDIN_FILENO);
-    saved[1] = dup(STDOUT_FILENO);
+    saved[0] = shell_dup(STDIN_FILENO);
+    saved[1] = shell_dup(STDOUT_FILENO);
     if (saved[0] < 0 || saved[1] < 0) {
         if (saved[0] >= 0)
             close(saved[0]);
@@ -100,11 +101,11 @@ static int save_stdio(int saved[2])
 static void restore_stdio(int saved[2])
 {
     if (saved[0] >= 0) {
-        (void)dup2(saved[0], STDIN_FILENO);
+        (void)shell_dup2(saved[0], STDIN_FILENO);
         close(saved[0]);
     }
     if (saved[1] >= 0) {
-        (void)dup2(saved[1], STDOUT_FILENO);
+        (void)shell_dup2(saved[1], STDOUT_FILENO);
         close(saved[1]);
     }
 }
diff --git a/src/runtime.c b/src/runtime.c
index 32fe72e..c764b16 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -3,7 +3,9 @@
 #include "runtime.h"
 
 #include <errno.h>
+#include <fcntl.h>
 #include <stdlib.h>
+#include <string.h>
 #include <sys/wait.h>
 #include <unistd.h>
 
@@ -20,7 +22,21 @@ static int fail_call(const char *name, unsigned long *calls)
     if (text == NULL || text[0] == '\0')
         return 0;
     target = strtoul(text, &end, 10);
-    return (end != text && *end == '\0' && target != 0 && target == *calls);
+    if (end == text || *end != '\0' || target == 0)
+        return 0;
+    {
+        char repeat_name[64];
+        size_t length;
+
+        length = strlen(name);
+        if (length + sizeof("_REPEAT") <= sizeof(repeat_name)) {
+            memcpy(repeat_name, name, length);
+            memcpy(repeat_name + length, "_REPEAT", sizeof("_REPEAT"));
+            if (getenv(repeat_name) != NULL)
+                return *calls >= target;
+        }
+    }
+    return target == *calls;
 }
 
 #endif
@@ -63,3 +79,42 @@ pid_t shell_waitpid(pid_t pid, int *status, int options)
 #endif
     return waitpid(pid, status, options);
 }
+
+int shell_dup(int fd)
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_DUP", &calls)) {
+        errno = EMFILE;
+        return -1;
+    }
+#endif
+    return dup(fd);
+}
+
+int shell_dup2(int oldfd, int newfd)
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_DUP2", &calls)) {
+        errno = EIO;
+        return -1;
+    }
+#endif
+    return dup2(oldfd, newfd);
+}
+
+int shell_open(const char *path, int flags, mode_t mode)
+{
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_OPEN", &calls)) {
+        errno = EACCES;
+        return -1;
+    }
+#endif
+    return open(path, flags, mode);
+}
diff --git a/src/runtime.h b/src/runtime.h
index d9d7d5c..7558d03 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -2,9 +2,13 @@
 # define RUNTIME_H
 
 # include <sys/types.h>
+# include <sys/stat.h>
 
 int     shell_pipe(int fds[2]);
 pid_t   shell_fork(void);
 pid_t   shell_waitpid(pid_t pid, int *status, int options);
+int     shell_dup(int fd);
+int     shell_dup2(int oldfd, int newfd);
+int     shell_open(const char *path, int flags, mode_t mode);
 
 #endif


## `fix(redirection): 부모 표준 입출력 복원 실패 전파`

diff --git a/src/redirection.c b/src/redirection.c
index 86a42aa..f1cec1c 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -98,16 +98,37 @@ static int save_stdio(int saved[2])
     return 0;
 }
 
-static void restore_stdio(int saved[2])
+static int restore_one(int saved, int target)
 {
-    if (saved[0] >= 0) {
-        (void)shell_dup2(saved[0], STDIN_FILENO);
-        close(saved[0]);
-    }
-    if (saved[1] >= 0) {
-        (void)shell_dup2(saved[1], STDOUT_FILENO);
-        close(saved[1]);
+    int attempts;
+    int had_error;
+
+    attempts = 0;
+    had_error = 0;
+    while (attempts < 2) {
+        if (shell_dup2(saved, target) >= 0)
+            return had_error;
+        if (errno == EINTR)
+            continue;
+        fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
+        had_error = 1;
+        attempts++;
     }
+    return -1;
+}
+
+static int restore_stdio(int saved[2])
+{
+    int input_result;
+    int output_result;
+
+    input_result = restore_one(saved[0], STDIN_FILENO);
+    output_result = restore_one(saved[1], STDOUT_FILENO);
+    close(saved[0]);
+    close(saved[1]);
+    if (input_result < 0 || output_result < 0)
+        return -1;
+    return (input_result != 0 || output_result != 0);
 }
 
 int exec_run_parent_command(t_shell *shell, const t_command *command,
@@ -119,14 +140,24 @@ int exec_run_parent_command(t_shell *shell, const t_command *command,
     if (save_stdio(saved) != 0)
         return 1;
     if (exec_apply_redirections(command, ctx) != 0) {
-        restore_stdio(saved);
+        if (restore_stdio(saved) < 0)
+            shell->running = 0;
         return 1;
     }
     if (command->argc == 0)
         status = 0;
     else
         status = builtin_run(shell, command->argv);
-    fflush(stdout);
-    restore_stdio(saved);
+    if (fflush(stdout) == EOF)
+        status = 1;
+    {
+        int restore_result;
+
+        restore_result = restore_stdio(saved);
+        if (restore_result != 0)
+            status = 1;
+        if (restore_result < 0)
+            shell->running = 0;
+    }
     return status;
 }


## `test(redirection): 저장·적용·복원 실패 회귀 검증`

diff --git a/tests/faults.sh b/tests/faults.sh
index 9c994f0..705de88 100755
--- a/tests/faults.sh
+++ b/tests/faults.sh
@@ -56,4 +56,65 @@ echo $?' \
     '1
 '
 
+run_fault save_stdin SMALL_SHELL_FAIL_DUP 1 \
+    "echo hidden > $TMP/save-stdin
+echo \$?
+echo after" \
+    '1
+after
+'
+
+run_fault save_stdout SMALL_SHELL_FAIL_DUP 2 \
+    "echo hidden > $TMP/save-stdout
+echo \$?
+echo after" \
+    '1
+after
+'
+
+run_fault apply_stdout SMALL_SHELL_FAIL_DUP2 1 \
+    "echo hidden > $TMP/apply-stdout
+echo \$?
+echo after" \
+    '1
+after
+'
+
+run_fault restore_stdin SMALL_SHELL_FAIL_DUP2 2 \
+    "echo hidden > $TMP/restore-stdin
+echo \$?
+echo after" \
+    '1
+after
+'
+
+run_fault restore_stdout SMALL_SHELL_FAIL_DUP2 3 \
+    "echo hidden > $TMP/restore-stdout
+echo \$?
+echo after" \
+    '1
+after
+'
+
+run_fault open_output SMALL_SHELL_FAIL_OPEN 1 \
+    "echo hidden > $TMP/open-output
+echo \$?
+echo after" \
+    '1
+after
+'
+
+set +e
+env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
+    "$BIN" >"$TMP/persistent-restore.out" \
+    2>"$TMP/persistent-restore.err" <<EOF
+echo hidden > $TMP/persistent-restore-file
+echo never
+EOF
+status=$?
+set -e
+[ "$status" -eq 1 ] || fail persistent-restore
+[ ! -s "$TMP/persistent-restore.out" ] || fail persistent-restore
+grep -q 'dup2' "$TMP/persistent-restore.err" || fail persistent-restore
+
 echo "ok - pipeline faults"


## `refactor(runtime): heredoc 임시 파일 I/O 경계 분리`

diff --git a/src/redirection.c b/src/redirection.c
index f1cec1c..8c9a9b4 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -69,9 +69,9 @@ int exec_apply_redirections(const t_command *command,
                 fclose(tmp);
                 return 1;
             }
-            fflush(tmp);
-            rewind(tmp);
-            if (shell_dup2(fileno(tmp), STDIN_FILENO) < 0) {
+            (void)shell_fflush(tmp);
+            (void)shell_fseek(tmp, 0L, SEEK_SET);
+            if (shell_dup2(shell_fileno(tmp), STDIN_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 fclose(tmp);
                 return 1;
diff --git a/src/runtime.c b/src/runtime.c
index c764b16..7066875 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -118,3 +118,18 @@ int shell_open(const char *path, int flags, mode_t mode)
 #endif
     return open(path, flags, mode);
 }
+
+int shell_fflush(FILE *stream)
+{
+    return fflush(stream);
+}
+
+int shell_fseek(FILE *stream, long offset, int whence)
+{
+    return fseek(stream, offset, whence);
+}
+
+int shell_fileno(FILE *stream)
+{
+    return fileno(stream);
+}
diff --git a/src/runtime.h b/src/runtime.h
index 7558d03..c6f092f 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -1,6 +1,7 @@
 #ifndef RUNTIME_H
 # define RUNTIME_H
 
+# include <stdio.h>
 # include <sys/types.h>
 # include <sys/stat.h>
 
@@ -10,5 +11,8 @@ pid_t   shell_waitpid(pid_t pid, int *status, int options);
 int     shell_dup(int fd);
 int     shell_dup2(int oldfd, int newfd);
 int     shell_open(const char *path, int flags, mode_t mode);
+int     shell_fflush(FILE *stream);
+int     shell_fseek(FILE *stream, long offset, int whence);
+int     shell_fileno(FILE *stream);
 
 #endif


