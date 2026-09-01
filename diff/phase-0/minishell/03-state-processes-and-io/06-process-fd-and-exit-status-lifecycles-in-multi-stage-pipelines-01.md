# 다단 파이프라인의 프로세스·FD 수명과 종료 상태

## `feat(parser): pipe로 명령을 pipeline에 결합`

diff --git a/src/parser.c b/src/parser.c
index 029fe70..89f1b60 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -66,6 +66,21 @@ static int command_empty(t_command *cmd) {
     return (!cmd || (cmd->argc == 0 && !cmd->redirs));
 }
 
+static void append_command(t_pipeline *pipeline, t_command *cmd) {
+    t_command *tail;
+
+    if (!pipeline->commands) {
+        pipeline->commands = cmd;
+        pipeline->command_count++;
+        return;
+    }
+    tail = pipeline->commands;
+    while (tail->next)
+        tail = tail->next;
+    tail->next = cmd;
+    pipeline->command_count++;
+}
+
 static int token_is_redir(t_token_type type) {
     return (type == TOK_REDIR_IN || type == TOK_REDIR_OUT
         || type == TOK_REDIR_APPEND);
@@ -83,16 +98,19 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
     t_pipeline  *pipeline;
     t_command   *cmd;
     t_token     *cur;
+    int         after_pipe;
 
     pipeline = new_pipeline();
     cmd = new_command();
     cur = tokens;
+    after_pipe = 0;
     if (error)
         *error = NULL;
     while (cur) {
-        if (cur->type == TOK_WORD)
+        if (cur->type == TOK_WORD) {
             add_arg(cmd, cur->text);
-        else if (token_is_redir(cur->type)) {
+            after_pipe = 0;
+        } else if (token_is_redir(cur->type)) {
             if (!cur->next || cur->next->type != TOK_WORD) {
                 set_error(error, "syntax error: redirection target missing");
                 free_commands(cmd);
@@ -101,21 +119,42 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
             }
             add_redir(cmd, redir_type(cur->type), cur->next->text);
             cur = cur->next;
+            after_pipe = 0;
+        } else if (cur->type == TOK_PIPE) {
+            if (command_empty(cmd)) {
+                set_error(error, "syntax error: empty command before pipe");
+                free_commands(cmd);
+                free_pipeline(pipeline);
+                return NULL;
+            }
+            append_command(pipeline, cmd);
+            cmd = new_command();
+            after_pipe = 1;
         } else {
-            set_error(error, "syntax error: unsupported operator");
+            if (after_pipe)
+                set_error(error, "syntax error: expected command after pipe");
+            else
+                set_error(error, "syntax error: unsupported operator");
             free_commands(cmd);
-            free(pipeline);
+            free_pipeline(pipeline);
             return NULL;
         }
         cur = cur->next;
     }
-    if (command_empty(cmd)) {
+    if (after_pipe) {
+        set_error(error, "syntax error: expected command after pipe");
         free_commands(cmd);
-        free(pipeline);
+        free_pipeline(pipeline);
         return NULL;
     }
-    pipeline->commands = cmd;
-    pipeline->command_count = 1;
+    if (command_empty(cmd)) {
+        free_commands(cmd);
+        if (!pipeline->commands) {
+            free(pipeline);
+            return NULL;
+        }
+    } else
+        append_command(pipeline, cmd);
     return pipeline;
 }
 


## `feat(exec): 단일 명령을 자식에서 실행`

diff --git a/Makefile b/Makefile
index cca9197..cecab3e 100644
--- a/Makefile
+++ b/Makefile
@@ -13,6 +13,7 @@ SRCS := \
 	src/expand.c \
 	src/env.c \
 	src/utils.c \
+	src/exec.c \
 	src/redirection.c \
 	src/builtin.c
 OBJS := $(SRCS:.c=.o)
diff --git a/include/shell.h b/include/shell.h
index d037ea8..19dc8d4 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -130,5 +130,6 @@ char    **shell_env_to_envp(t_env *env);
 int     builtin_is_parent(const char *name);
 int     builtin_is_known(const char *name);
 int     builtin_run(t_shell *shell, char **argv);
+int     execute_pipeline_list(t_shell *shell, t_pipeline *pipeline);
 
 #endif
diff --git a/src/exec.c b/src/exec.c
new file mode 100644
index 0000000..db9c1bf
--- /dev/null
+++ b/src/exec.c
@@ -0,0 +1,102 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "exec_internal.h"
+
+#include <errno.h>
+#include <stdio.h>
+#include <string.h>
+#include <sys/types.h>
+#include <sys/wait.h>
+#include <unistd.h>
+
+extern char **environ;
+
+static int status_from_wait(int wait_status)
+{
+    if (WIFEXITED(wait_status))
+        return WEXITSTATUS(wait_status);
+    if (WIFSIGNALED(wait_status))
+        return 128 + WTERMSIG(wait_status);
+    return 1;
+}
+
+static void run_child(t_shell *shell, const t_command *command,
+    const struct exec_context *ctx)
+{
+    if (exec_apply_redirections(command, ctx) != 0)
+        _exit(1);
+    if (command->argc == 0)
+        _exit(0);
+    if (builtin_is_known(command->argv[0])) {
+        int status;
+
+        status = builtin_run(shell, command->argv);
+        fflush(stdout);
+        fflush(stderr);
+        _exit(status & 0xff);
+    }
+    {
+        char **envp;
+
+        envp = env_to_environ(shell->env);
+        if (envp == NULL) {
+            fprintf(stderr, "small-shell: allocation failure\n");
+            _exit(1);
+        }
+        environ = envp;
+        execvp(command->argv[0], command->argv);
+        {
+            int err;
+
+            err = errno;
+            fprintf(stderr, "small-shell: %s: %s\n", command->argv[0],
+                strerror(err));
+            sh_free_words(envp);
+            if (err == ENOENT)
+                _exit(127);
+            _exit(126);
+        }
+    }
+}
+
+static int run_single_command(t_shell *shell, const t_command *command,
+    const struct exec_context *ctx)
+{
+    pid_t pid;
+    pid_t waited;
+    int wait_status;
+
+    pid = fork();
+    if (pid < 0) {
+        fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+        return 1;
+    }
+    if (pid == 0)
+        run_child(shell, command, ctx);
+    do {
+        waited = waitpid(pid, &wait_status, 0);
+    } while (waited < 0 && errno == EINTR);
+    if (waited != pid)
+        return 1;
+    return status_from_wait(wait_status);
+}
+
+int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
+{
+    const t_command *command;
+    struct exec_context ctx;
+
+    if (shell == NULL || pipeline == NULL || pipeline->command_count == 0)
+        return shell != NULL ? shell->last_status : 1;
+    if (pipeline->next != NULL || pipeline->command_count != 1)
+        return 1;
+    if (expand_pipeline(shell, pipeline) != 0)
+        return 1;
+    ctx.shell = shell;
+    command = pipeline->commands;
+    if (command->argc == 0 || builtin_is_parent(command->argv[0]))
+        shell->last_status = exec_run_parent_command(shell, command, &ctx);
+    else
+        shell->last_status = run_single_command(shell, command, &ctx);
+    return shell->last_status;
+}


## `feat(exec): pipeline 자식 상태를 순서대로 회수`

diff --git a/src/exec.c b/src/exec.c
index db9c1bf..0ece808 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -4,6 +4,7 @@
 
 #include <errno.h>
 #include <stdio.h>
+#include <stdlib.h>
 #include <string.h>
 #include <sys/types.h>
 #include <sys/wait.h>
@@ -59,26 +60,49 @@ static void run_child(t_shell *shell, const t_command *command,
     }
 }
 
-static int run_single_command(t_shell *shell, const t_command *command,
+static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
     const struct exec_context *ctx)
 {
-    pid_t pid;
-    pid_t waited;
-    int wait_status;
+    pid_t           *pids;
+    const t_command *command;
+    size_t          i;
+    size_t          spawned;
+    int             result;
 
-    pid = fork();
-    if (pid < 0) {
-        fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+    pids = (pid_t *)calloc(pipeline->command_count, sizeof(pid_t));
+    if (pids == NULL) {
+        fprintf(stderr, "small-shell: allocation failure\n");
         return 1;
     }
-    if (pid == 0)
-        run_child(shell, command, ctx);
-    do {
-        waited = waitpid(pid, &wait_status, 0);
-    } while (waited < 0 && errno == EINTR);
-    if (waited != pid)
-        return 1;
-    return status_from_wait(wait_status);
+    command = pipeline->commands;
+    spawned = 0;
+    result = 1;
+    for (i = 0; i < pipeline->command_count && command != NULL; i++) {
+        pid_t pid;
+
+        pid = fork();
+        if (pid < 0) {
+            fprintf(stderr, "small-shell: fork: %s\n", strerror(errno));
+            break;
+        }
+        if (pid == 0)
+            run_child(shell, command, ctx);
+        pids[i] = pid;
+        spawned++;
+        command = command->next;
+    }
+    for (i = 0; i < spawned; i++) {
+        int wait_status;
+        pid_t waited;
+
+        do {
+            waited = waitpid(pids[i], &wait_status, 0);
+        } while (waited < 0 && errno == EINTR);
+        if (waited == pids[i] && i + 1 == pipeline->command_count)
+            result = status_from_wait(wait_status);
+    }
+    free(pids);
+    return spawned == pipeline->command_count ? result : 1;
 }
 
 int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
@@ -97,6 +121,6 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
     if (command->argc == 0 || builtin_is_parent(command->argv[0]))
         shell->last_status = exec_run_parent_command(shell, command, &ctx);
     else
-        shell->last_status = run_single_command(shell, command, &ctx);
+        shell->last_status = run_forked_commands(shell, pipeline, &ctx);
     return shell->last_status;
 }


## `feat(exec): 다단 pipeline의 pipe FD 연결`

diff --git a/src/exec.c b/src/exec.c
index 0ece808..63aef88 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -12,6 +12,20 @@
 
 extern char **environ;
 
+static void close_pipes(int (*pipes)[2], size_t pipe_count)
+{
+    size_t i;
+
+    if (pipes == NULL)
+        return;
+    for (i = 0; i < pipe_count; i++) {
+        if (pipes[i][0] >= 0)
+            close(pipes[i][0]);
+        if (pipes[i][1] >= 0)
+            close(pipes[i][1]);
+    }
+}
+
 static int status_from_wait(int wait_status)
 {
     if (WIFEXITED(wait_status))
@@ -21,9 +35,21 @@ static int status_from_wait(int wait_status)
     return 1;
 }
 
-static void run_child(t_shell *shell, const t_command *command,
-    const struct exec_context *ctx)
+static void child_die(const char *what)
+{
+    fprintf(stderr, "small-shell: %s: %s\n", what, strerror(errno));
+    _exit(1);
+}
+
+static void run_child(t_shell *shell, const t_pipeline *pipeline, const t_command *command,
+    const struct exec_context *ctx, int (*pipes)[2], size_t pipe_count, size_t index)
 {
+    if (index > 0 && dup2(pipes[index - 1][0], STDIN_FILENO) < 0)
+        child_die("dup2");
+    if (index + 1 < pipeline->command_count && dup2(pipes[index][1], STDOUT_FILENO) < 0)
+        child_die("dup2");
+    close_pipes(pipes, pipe_count);
+
     if (exec_apply_redirections(command, ctx) != 0)
         _exit(1);
     if (command->argc == 0)
@@ -36,6 +62,7 @@ static void run_child(t_shell *shell, const t_command *command,
         fflush(stderr);
         _exit(status & 0xff);
     }
+
     {
         char **envp;
 
@@ -50,8 +77,7 @@ static void run_child(t_shell *shell, const t_command *command,
             int err;
 
             err = errno;
-            fprintf(stderr, "small-shell: %s: %s\n", command->argv[0],
-                strerror(err));
+            fprintf(stderr, "small-shell: %s: %s\n", command->argv[0], strerror(err));
             sh_free_words(envp);
             if (err == ENOENT)
                 _exit(127);
@@ -60,23 +86,45 @@ static void run_child(t_shell *shell, const t_command *command,
     }
 }
 
-static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
-    const struct exec_context *ctx)
+static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const struct exec_context *ctx)
 {
-    pid_t           *pids;
+    size_t pipe_count;
+    int (*pipes)[2];
+    pid_t *pids;
     const t_command *command;
-    size_t          i;
-    size_t          spawned;
-    int             result;
+    size_t i;
+    size_t spawned;
+    int result;
 
-    pids = (pid_t *)calloc(pipeline->command_count, sizeof(pid_t));
-    if (pids == NULL) {
-        fprintf(stderr, "small-shell: allocation failure\n");
-        return 1;
-    }
-    command = pipeline->commands;
+    pipe_count = pipeline->command_count - 1;
+    pipes = NULL;
+    pids = NULL;
     spawned = 0;
     result = 1;
+
+    if (pipe_count > 0) {
+        pipes = (int (*)[2])malloc(sizeof(int[2]) * pipe_count);
+        if (pipes == NULL)
+            goto alloc_error;
+        for (i = 0; i < pipe_count; i++) {
+            pipes[i][0] = -1;
+            pipes[i][1] = -1;
+        }
+        for (i = 0; i < pipe_count; i++) {
+            if (pipe(pipes[i]) < 0) {
+                fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
+                close_pipes(pipes, pipe_count);
+                free(pipes);
+                return 1;
+            }
+        }
+    }
+
+    pids = (pid_t *)calloc(pipeline->command_count, sizeof(pid_t));
+    if (pids == NULL)
+        goto alloc_error;
+
+    command = pipeline->commands;
     for (i = 0; i < pipeline->command_count && command != NULL; i++) {
         pid_t pid;
 
@@ -86,11 +134,13 @@ static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
             break;
         }
         if (pid == 0)
-            run_child(shell, command, ctx);
+            run_child(shell, pipeline, command, ctx, pipes, pipe_count, i);
         pids[i] = pid;
         spawned++;
         command = command->next;
     }
+
+    close_pipes(pipes, pipe_count);
     for (i = 0; i < spawned; i++) {
         int wait_status;
         pid_t waited;
@@ -101,8 +151,17 @@ static int run_forked_commands(t_shell *shell, const t_pipeline *pipeline,
         if (waited == pids[i] && i + 1 == pipeline->command_count)
             result = status_from_wait(wait_status);
     }
+
     free(pids);
+    free(pipes);
     return spawned == pipeline->command_count ? result : 1;
+
+alloc_error:
+    fprintf(stderr, "small-shell: allocation failure\n");
+    close_pipes(pipes, pipe_count);
+    free(pipes);
+    free(pids);
+    return 1;
 }
 
 int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
@@ -112,15 +171,16 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
 
     if (shell == NULL || pipeline == NULL || pipeline->command_count == 0)
         return shell != NULL ? shell->last_status : 1;
-    if (pipeline->next != NULL || pipeline->command_count != 1)
+    if (pipeline->next != NULL)
         return 1;
     if (expand_pipeline(shell, pipeline) != 0)
         return 1;
     ctx.shell = shell;
     command = pipeline->commands;
-    if (command->argc == 0 || builtin_is_parent(command->argv[0]))
+    if (pipeline->command_count == 1
+        && (command->argc == 0 || builtin_is_parent(command->argv[0])))
         shell->last_status = exec_run_parent_command(shell, command, &ctx);
     else
-        shell->last_status = run_forked_commands(shell, pipeline, &ctx);
+        shell->last_status = run_forked_pipeline(shell, pipeline, &ctx);
     return shell->last_status;
 }


## `feat(exec): 조건 연결자와 지연 확장 실행`

diff --git a/src/exec.c b/src/exec.c
index 63aef88..d0d9795 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -164,23 +164,58 @@ alloc_error:
     return 1;
 }
 
-int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
+static int expand_one_pipeline(t_shell *shell, t_pipeline *pipeline)
+{
+    t_pipeline *next;
+    int result;
+
+    next = pipeline->next;
+    pipeline->next = NULL;
+    result = expand_pipeline(shell, pipeline);
+    pipeline->next = next;
+    return result;
+}
+
+static int execute_one_pipeline(t_shell *shell, t_pipeline *pipeline, const struct exec_context *ctx)
 {
     const t_command *command;
-    struct exec_context ctx;
 
-    if (shell == NULL || pipeline == NULL || pipeline->command_count == 0)
-        return shell != NULL ? shell->last_status : 1;
-    if (pipeline->next != NULL)
-        return 1;
-    if (expand_pipeline(shell, pipeline) != 0)
+    if (pipeline == NULL || pipeline->command_count == 0)
+        return shell->last_status;
+    if (expand_one_pipeline(shell, pipeline) != 0)
         return 1;
-    ctx.shell = shell;
     command = pipeline->commands;
     if (pipeline->command_count == 1
         && (command->argc == 0 || builtin_is_parent(command->argv[0])))
-        shell->last_status = exec_run_parent_command(shell, command, &ctx);
-    else
-        shell->last_status = run_forked_pipeline(shell, pipeline, &ctx);
+        return exec_run_parent_command(shell, command, ctx);
+    return run_forked_pipeline(shell, pipeline, ctx);
+}
+
+static int execute_pipeline_list_ctx(t_shell *shell, t_pipeline *pipeline, const struct exec_context *ctx)
+{
+    t_connector previous;
+
+    previous = CONN_NONE;
+    while (pipeline != NULL && shell->running) {
+        int should_run;
+
+        should_run = 1;
+        if (previous == CONN_AND && shell->last_status != 0)
+            should_run = 0;
+        else if (previous == CONN_OR && shell->last_status == 0)
+            should_run = 0;
+        if (should_run)
+            shell->last_status = execute_one_pipeline(shell, pipeline, ctx);
+        previous = pipeline->next_op;
+        pipeline = pipeline->next;
+    }
     return shell->last_status;
 }
+
+int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
+{
+    struct exec_context ctx;
+
+    ctx.shell = shell;
+    return execute_pipeline_list_ctx(shell, pipeline, &ctx);
+}


## `test(exec): 다단 파이프와 리다이렉션 순서 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 30f152a..1f310d0 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -2,7 +2,7 @@
 set -eu
 
 ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
-BIN="$ROOT/small-shell"
+BIN=${SMALL_SHELL_BIN:-"$ROOT/small-shell"}
 TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell.XXXXXX")
 TMP_PHYSICAL=$(CDPATH= cd -- "$TMP" && pwd -P)
 
@@ -178,4 +178,22 @@ echo never
 "" \
 7
 
+run_case multi_stage_pipeline \
+"printf abc | tr a-z A-Z | sed 's/B/X/' | cat
+" \
+"AXC" \
+0
+
+run_case redirection_order \
+"echo first > $TMP/first.txt > $TMP/second.txt
+cat $TMP/first.txt
+cat $TMP/second.txt
+echo pipe > $TMP/pipe.txt | cat
+cat $TMP/pipe.txt
+" \
+"first
+pipe
+" \
+0
+
 echo "ok - smoke"


## `test(status): 실행 불가 파일과 신호 종료 상태 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 1f310d0..ad5cb18 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -196,4 +196,22 @@ pipe
 " \
 0
 
+printf '#!/bin/sh\necho should-not-run\n' >"$TMP/not-executable"
+chmod 0644 "$TMP/not-executable"
+run_case cannot_execute_status \
+"$TMP/not-executable
+echo \$?
+" \
+"126
+" \
+0
+
+run_case signal_exit_status \
+"/bin/sh -c 'kill -TERM \$\$'
+echo \$?
+" \
+"143
+" \
+0
+
 echo "ok - smoke"


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


