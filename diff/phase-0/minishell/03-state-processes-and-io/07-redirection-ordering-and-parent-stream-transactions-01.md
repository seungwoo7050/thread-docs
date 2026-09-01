# 리다이렉션 순서와 부모 표준 스트림 트랜잭션

## `feat(parser): 인자와 리다이렉션 구문 구성`

diff --git a/include/shell.h b/include/shell.h
index 5965a58..ded9d2b 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -85,6 +85,7 @@ int     sh_is_name_char(int c);
 int     sh_is_name_start(int c);
 t_token *tokenize_line(const char *line, char **error);
 void    free_tokens(t_token *tokens);
+t_pipeline  *parse_tokens(t_token *tokens, char **error);
 void    free_pipeline(t_pipeline *pipeline);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
diff --git a/src/parser.c b/src/parser.c
index 411a46d..029fe70 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -1,6 +1,123 @@
 #include "shell.h"
 
 #include <stdlib.h>
+#include <string.h>
+
+static void free_commands(t_command *cmd);
+
+static void set_error(char **error, const char *message) {
+    if (error)
+        *error = sh_strdup(message);
+}
+
+static t_command *new_command(void) {
+    return (t_command *)sh_xcalloc(1, sizeof(t_command));
+}
+
+static t_pipeline *new_pipeline(void) {
+    t_pipeline *pipeline = (t_pipeline *)sh_xcalloc(1, sizeof(t_pipeline));
+    pipeline->next_op = CONN_NONE;
+    return pipeline;
+}
+
+static size_t word_count(char **argv) {
+    size_t n = 0;
+    while (argv && argv[n])
+        n++;
+    return n;
+}
+
+static void add_arg(t_command *cmd, const char *text) {
+    size_t  n;
+    char    **next;
+    size_t  i;
+
+    n = word_count(cmd->argv);
+    next = (char **)sh_xcalloc(n + 2, sizeof(char *));
+    i = 0;
+    while (i < n) {
+        next[i] = cmd->argv[i];
+        i++;
+    }
+    next[n] = sh_strdup(text);
+    free(cmd->argv);
+    cmd->argv = next;
+    cmd->argc = n + 1;
+}
+
+static void add_redir(t_command *cmd, t_redir_type type, const char *target) {
+    t_redir *node;
+    t_redir *tail;
+
+    node = (t_redir *)sh_xcalloc(1, sizeof(t_redir));
+    node->type = type;
+    node->target = sh_strdup(target);
+    if (!cmd->redirs) {
+        cmd->redirs = node;
+        return;
+    }
+    tail = cmd->redirs;
+    while (tail->next)
+        tail = tail->next;
+    tail->next = node;
+}
+
+static int command_empty(t_command *cmd) {
+    return (!cmd || (cmd->argc == 0 && !cmd->redirs));
+}
+
+static int token_is_redir(t_token_type type) {
+    return (type == TOK_REDIR_IN || type == TOK_REDIR_OUT
+        || type == TOK_REDIR_APPEND);
+}
+
+static t_redir_type redir_type(t_token_type type) {
+    if (type == TOK_REDIR_OUT)
+        return REDIR_OUT;
+    if (type == TOK_REDIR_APPEND)
+        return REDIR_APPEND;
+    return REDIR_IN;
+}
+
+t_pipeline *parse_tokens(t_token *tokens, char **error) {
+    t_pipeline  *pipeline;
+    t_command   *cmd;
+    t_token     *cur;
+
+    pipeline = new_pipeline();
+    cmd = new_command();
+    cur = tokens;
+    if (error)
+        *error = NULL;
+    while (cur) {
+        if (cur->type == TOK_WORD)
+            add_arg(cmd, cur->text);
+        else if (token_is_redir(cur->type)) {
+            if (!cur->next || cur->next->type != TOK_WORD) {
+                set_error(error, "syntax error: redirection target missing");
+                free_commands(cmd);
+                free(pipeline);
+                return NULL;
+            }
+            add_redir(cmd, redir_type(cur->type), cur->next->text);
+            cur = cur->next;
+        } else {
+            set_error(error, "syntax error: unsupported operator");
+            free_commands(cmd);
+            free(pipeline);
+            return NULL;
+        }
+        cur = cur->next;
+    }
+    if (command_empty(cmd)) {
+        free_commands(cmd);
+        free(pipeline);
+        return NULL;
+    }
+    pipeline->commands = cmd;
+    pipeline->command_count = 1;
+    return pipeline;
+}
 
 static void free_redirs(t_redir *redir) {
     t_redir *next;


## `feat(redirection): 파일 입출력 리다이렉션 적용`

diff --git a/Makefile b/Makefile
index 33658ca..cca9197 100644
--- a/Makefile
+++ b/Makefile
@@ -13,6 +13,7 @@ SRCS := \
 	src/expand.c \
 	src/env.c \
 	src/utils.c \
+	src/redirection.c \
 	src/builtin.c
 OBJS := $(SRCS:.c=.o)
 
diff --git a/src/exec_internal.h b/src/exec_internal.h
new file mode 100644
index 0000000..230d956
--- /dev/null
+++ b/src/exec_internal.h
@@ -0,0 +1,13 @@
+#ifndef EXEC_INTERNAL_H
+# define EXEC_INTERNAL_H
+
+# include "shell.h"
+
+struct exec_context {
+    t_shell *shell;
+};
+
+int exec_apply_redirections(const t_command *command,
+        const struct exec_context *ctx);
+
+#endif
diff --git a/src/redirection.c b/src/redirection.c
new file mode 100644
index 0000000..0cd1762
--- /dev/null
+++ b/src/redirection.c
@@ -0,0 +1,59 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "exec_internal.h"
+
+#include <errno.h>
+#include <fcntl.h>
+#include <stdio.h>
+#include <string.h>
+#include <unistd.h>
+
+int exec_apply_redirections(const t_command *command,
+    const struct exec_context *ctx)
+{
+    const t_redir *redir;
+
+    (void)ctx;
+    redir = command->redirs;
+    while (redir != NULL) {
+        int fd;
+
+        if (redir->type == REDIR_IN) {
+            fd = open(redir->target, O_RDONLY);
+            if (fd < 0) {
+                fprintf(stderr, "small-shell: %s: %s\n", redir->target,
+                    strerror(errno));
+                return 1;
+            }
+            if (dup2(fd, STDIN_FILENO) < 0) {
+                fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
+                close(fd);
+                return 1;
+            }
+            close(fd);
+        } else if (redir->type == REDIR_OUT
+            || redir->type == REDIR_APPEND) {
+            int flags;
+
+            flags = O_WRONLY | O_CREAT;
+            if (redir->type == REDIR_OUT)
+                flags |= O_TRUNC;
+            else
+                flags |= O_APPEND;
+            fd = open(redir->target, flags, 0644);
+            if (fd < 0) {
+                fprintf(stderr, "small-shell: %s: %s\n", redir->target,
+                    strerror(errno));
+                return 1;
+            }
+            if (dup2(fd, STDOUT_FILENO) < 0) {
+                fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
+                close(fd);
+                return 1;
+            }
+            close(fd);
+        }
+        redir = redir->next;
+    }
+    return 0;
+}


## `feat(exec): 부모 builtin의 표준 스트림 복원`

diff --git a/src/exec_internal.h b/src/exec_internal.h
index 230d956..28315df 100644
--- a/src/exec_internal.h
+++ b/src/exec_internal.h
@@ -9,5 +9,7 @@ struct exec_context {
 
 int exec_apply_redirections(const t_command *command,
         const struct exec_context *ctx);
+int exec_run_parent_command(t_shell *shell, const t_command *command,
+        const struct exec_context *ctx);
 
 #endif
diff --git a/src/redirection.c b/src/redirection.c
index 0cd1762..290bc7e 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -57,3 +57,51 @@ int exec_apply_redirections(const t_command *command,
     }
     return 0;
 }
+
+static int save_stdio(int saved[2])
+{
+    saved[0] = dup(STDIN_FILENO);
+    saved[1] = dup(STDOUT_FILENO);
+    if (saved[0] < 0 || saved[1] < 0) {
+        if (saved[0] >= 0)
+            close(saved[0]);
+        if (saved[1] >= 0)
+            close(saved[1]);
+        fprintf(stderr, "small-shell: dup: %s\n", strerror(errno));
+        return 1;
+    }
+    return 0;
+}
+
+static void restore_stdio(int saved[2])
+{
+    if (saved[0] >= 0) {
+        (void)dup2(saved[0], STDIN_FILENO);
+        close(saved[0]);
+    }
+    if (saved[1] >= 0) {
+        (void)dup2(saved[1], STDOUT_FILENO);
+        close(saved[1]);
+    }
+}
+
+int exec_run_parent_command(t_shell *shell, const t_command *command,
+    const struct exec_context *ctx)
+{
+    int saved[2];
+    int status;
+
+    if (save_stdio(saved) != 0)
+        return 1;
+    if (exec_apply_redirections(command, ctx) != 0) {
+        restore_stdio(saved);
+        return 1;
+    }
+    if (command->argc == 0)
+        status = 0;
+    else
+        status = builtin_run(shell, command->argv);
+    fflush(stdout);
+    restore_stdio(saved);
+    return status;
+}


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


## `test(redirection): 부모 명령의 표준 입출력 복원 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 88d6e3c..7ca7fb8 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -94,6 +94,18 @@ second
 " \
 0
 
+run_case parent_redirection_restore \
+"echo file-data > $TMP/parent-in.txt
+export SMALLSH_PARENT=kept < $TMP/parent-in.txt > $TMP/parent-out.txt
+echo after
+env | grep '^SMALLSH_PARENT=kept$'
+cat $TMP/parent-out.txt
+" \
+"after
+SMALLSH_PARENT=kept
+" \
+0
+
 run_case heredoc \
 "export HD=beta
 cat <<EOF


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


