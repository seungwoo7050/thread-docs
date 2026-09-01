## `feat(builtin): export 대입과 선언 출력`

diff --git a/src/builtin.c b/src/builtin.c
index 0d8893d..9c33b8f 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -15,6 +15,7 @@ int builtin_is_known(const char *name)
         "pwd",
         "cd",
         "env",
+        "export",
         NULL
     };
     size_t i;
@@ -147,6 +148,71 @@ static int builtin_env(t_shell *shell, char **argv)
     return ferror(stdout) ? 1 : 0;
 }
 
+static int split_assignment(const char *arg, char **key, const char **value)
+{
+    size_t len;
+
+    len = 0;
+    while (arg[len] != '\0' && arg[len] != '=')
+        len++;
+    *key = shell_strndup(arg, len);
+    if (*key == NULL)
+        return 1;
+    *value = arg[len] == '=' ? arg + len + 1 : NULL;
+    return 0;
+}
+
+static int builtin_export(t_shell *shell, char **argv)
+{
+    size_t i;
+    int status;
+
+    if (argv[1] == NULL) {
+        env_print(shell->env, 1);
+        return ferror(stdout) ? 1 : 0;
+    }
+    status = 0;
+    for (i = 1; argv[i] != NULL; i++) {
+        char *key;
+        const char *value;
+
+        key = NULL;
+        value = NULL;
+        if (split_assignment(argv[i], &key, &value) != 0) {
+            fprintf(stderr, "small-shell: export: allocation failure\n");
+            return 1;
+        }
+        if (!sh_is_name_start((unsigned char)key[0])) {
+            fprintf(stderr, "small-shell: export: `%s': not a valid identifier\n", argv[i]);
+            free(key);
+            status = 1;
+            continue;
+        }
+        {
+            size_t j;
+
+            for (j = 1; key[j] != '\0'; j++) {
+                if (!sh_is_name_char((unsigned char)key[j])) {
+                    fprintf(stderr, "small-shell: export: `%s': not a valid identifier\n", argv[i]);
+                    free(key);
+                    key = NULL;
+                    status = 1;
+                    break;
+                }
+            }
+        }
+        if (key == NULL)
+            continue;
+        if (env_set(&shell->env, key, value, 1) != 0) {
+            fprintf(stderr, "small-shell: export: allocation failure\n");
+            free(key);
+            return 1;
+        }
+        free(key);
+    }
+    return status;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
@@ -159,5 +225,7 @@ int builtin_run(t_shell *shell, char **argv)
         return builtin_cd(shell, argv);
     if (strcmp(argv[0], "env") == 0)
         return builtin_env(shell, argv);
+    if (strcmp(argv[0], "export") == 0)
+        return builtin_export(shell, argv);
     return 127;
 }


## `feat(builtin): unset 환경 이름 제거`

diff --git a/src/builtin.c b/src/builtin.c
index 9c33b8f..6e62c23 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -16,6 +16,7 @@ int builtin_is_known(const char *name)
         "cd",
         "env",
         "export",
+        "unset",
         NULL
     };
     size_t i;
@@ -213,6 +214,15 @@ static int builtin_export(t_shell *shell, char **argv)
     return status;
 }
 
+static int builtin_unset(t_shell *shell, char **argv)
+{
+    size_t i;
+
+    for (i = 1; argv[i] != NULL; i++)
+        (void)env_unset(&shell->env, argv[i]);
+    return 0;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
@@ -227,5 +237,7 @@ int builtin_run(t_shell *shell, char **argv)
         return builtin_env(shell, argv);
     if (strcmp(argv[0], "export") == 0)
         return builtin_export(shell, argv);
+    if (strcmp(argv[0], "unset") == 0)
+        return builtin_unset(shell, argv);
     return 127;
 }


## `feat(builtin): exit 상태를 셸 수명에 연결`

diff --git a/src/builtin.c b/src/builtin.c
index 6e62c23..f618604 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -17,6 +17,7 @@ int builtin_is_known(const char *name)
         "env",
         "export",
         "unset",
+        "exit",
         NULL
     };
     size_t i;
@@ -58,6 +59,7 @@ static int builtin_echo(char **argv)
         newline = 0;
         i++;
     }
+
     while (argv[i] != NULL) {
         fputs(argv[i], stdout);
         if (argv[i + 1] != NULL)
@@ -104,6 +106,7 @@ static int builtin_cd(t_shell *shell, char **argv)
         fprintf(stderr, "small-shell: cd: too many arguments\n");
         return 1;
     }
+
     print_target = 0;
     if (argv[1] == NULL) {
         target = env_get(shell->env, "HOME");
@@ -121,6 +124,7 @@ static int builtin_cd(t_shell *shell, char **argv)
     } else {
         target = argv[1];
     }
+
     old_pwd = getcwd(NULL, 0);
     if (chdir(target) != 0) {
         fprintf(stderr, "small-shell: cd: %s: %s\n", target, strerror(errno));
@@ -172,6 +176,7 @@ static int builtin_export(t_shell *shell, char **argv)
         env_print(shell->env, 1);
         return ferror(stdout) ? 1 : 0;
     }
+
     status = 0;
     for (i = 1; argv[i] != NULL; i++) {
         char *key;
@@ -223,6 +228,42 @@ static int builtin_unset(t_shell *shell, char **argv)
     return 0;
 }
 
+static int parse_exit_status(const char *s, int *status)
+{
+    char *end;
+    long value;
+
+    errno = 0;
+    value = strtol(s, &end, 10);
+    if (s == end || *end != '\0' || errno == ERANGE)
+        return 0;
+    *status = (unsigned char)value;
+    return 1;
+}
+
+static int builtin_exit(t_shell *shell, char **argv)
+{
+    int status;
+
+    if (argv[1] == NULL) {
+        shell->running = 0;
+        return shell->last_status;
+    }
+    if (!parse_exit_status(argv[1], &status)) {
+        fprintf(stderr, "small-shell: exit: %s: numeric argument required\n", argv[1]);
+        shell->last_status = 2;
+        shell->running = 0;
+        return 2;
+    }
+    if (argv[2] != NULL) {
+        fprintf(stderr, "small-shell: exit: too many arguments\n");
+        return 1;
+    }
+    shell->last_status = status;
+    shell->running = 0;
+    return status;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
@@ -239,5 +280,7 @@ int builtin_run(t_shell *shell, char **argv)
         return builtin_export(shell, argv);
     if (strcmp(argv[0], "unset") == 0)
         return builtin_unset(shell, argv);
+    if (strcmp(argv[0], "exit") == 0)
+        return builtin_exit(shell, argv);
     return 127;
 }


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


