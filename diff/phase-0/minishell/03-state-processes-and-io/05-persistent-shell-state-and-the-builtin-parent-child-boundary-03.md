## `fix(io): builtin과 환경 출력 실패를 상태로 전파`

diff --git a/include/shell.h b/include/shell.h
index 828c13d..4cea593 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -116,7 +116,7 @@ const char  *env_get(t_env *env, const char *key);
 int         env_set(t_env **env, const char *key, const char *value, int exported);
 int         env_unset(t_env **env, const char *key);
 char        **env_to_environ(t_env *env);
-void        env_print(t_env *env, int declare_style);
+int         env_print(t_env *env, int declare_style);
 int         shell_env_init(t_env *env, char **envp);
 void        shell_env_free(t_env *env);
 const char  *shell_env_get(const t_env *env, const char *key);
diff --git a/src/builtin.c b/src/builtin.c
index f618604..8c11de1 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "shell.h"
+#include "runtime.h"
 
 #include <errno.h>
 #include <stdio.h>
@@ -11,14 +12,7 @@
 int builtin_is_known(const char *name)
 {
     static const char *builtins[] = {
-        "echo",
-        "pwd",
-        "cd",
-        "env",
-        "export",
-        "unset",
-        "exit",
-        NULL
+        "echo", "pwd", "cd", "env", "export", "unset", "exit", NULL
     };
     size_t i;
 
@@ -38,14 +32,14 @@ int builtin_is_parent(const char *name)
 
 static int builtin_echo(char **argv)
 {
-    size_t i;
-    int newline;
+    size_t  i;
+    int     newline;
 
     newline = 1;
     i = 1;
     while (argv[i] != NULL && argv[i][0] == '-' && argv[i][1] == 'n') {
-        size_t j;
-        int only_n;
+        size_t  j;
+        int     only_n;
 
         only_n = 1;
         for (j = 1; argv[i][j] != '\0'; j++) {
@@ -59,16 +53,17 @@ static int builtin_echo(char **argv)
         newline = 0;
         i++;
     }
-
     while (argv[i] != NULL) {
-        fputs(argv[i], stdout);
-        if (argv[i + 1] != NULL)
-            fputc(' ', stdout);
+        if (shell_write_text(STDOUT_FILENO, argv[i]) != 0)
+            return 1;
+        if (argv[i + 1] != NULL
+            && shell_write_text(STDOUT_FILENO, " ") != 0)
+            return 1;
         i++;
     }
-    if (newline)
-        fputc('\n', stdout);
-    return ferror(stdout) ? 1 : 0;
+    if (newline && shell_write_text(STDOUT_FILENO, "\n") != 0)
+        return 1;
+    return 0;
 }
 
 static int builtin_pwd(void)
@@ -80,9 +75,14 @@ static int builtin_pwd(void)
         fprintf(stderr, "small-shell: pwd: %s\n", strerror(errno));
         return 1;
     }
-    printf("%s\n", cwd);
+    if (shell_write_text(STDOUT_FILENO, cwd) != 0
+        || shell_write_text(STDOUT_FILENO, "\n") != 0) {
+        fprintf(stderr, "small-shell: pwd: %s\n", strerror(errno));
+        free(cwd);
+        return 1;
+    }
     free(cwd);
-    return ferror(stdout) ? 1 : 0;
+    return 0;
 }
 
 static size_t argv_count(char **argv)
@@ -97,16 +97,16 @@ static size_t argv_count(char **argv)
 
 static int builtin_cd(t_shell *shell, char **argv)
 {
-    const char *target;
-    char *old_pwd;
-    char *new_pwd;
-    int print_target;
+    const char  *target;
+    char        *old_pwd;
+    char        *new_pwd;
+    int         print_target;
+    int         status;
 
     if (argv_count(argv) > 2) {
         fprintf(stderr, "small-shell: cd: too many arguments\n");
         return 1;
     }
-
     print_target = 0;
     if (argv[1] == NULL) {
         target = env_get(shell->env, "HOME");
@@ -124,7 +124,6 @@ static int builtin_cd(t_shell *shell, char **argv)
     } else {
         target = argv[1];
     }
-
     old_pwd = getcwd(NULL, 0);
     if (chdir(target) != 0) {
         fprintf(stderr, "small-shell: cd: %s: %s\n", target, strerror(errno));
@@ -132,15 +131,18 @@ static int builtin_cd(t_shell *shell, char **argv)
         return 1;
     }
     new_pwd = getcwd(NULL, 0);
-    if (old_pwd != NULL)
-        (void)env_set(&shell->env, "OLDPWD", old_pwd, 1);
-    if (new_pwd != NULL)
-        (void)env_set(&shell->env, "PWD", new_pwd, 1);
-    if (print_target && new_pwd != NULL)
-        printf("%s\n", new_pwd);
+    status = 0;
+    if (old_pwd != NULL && env_set(&shell->env, "OLDPWD", old_pwd, 1) != 0)
+        status = 1;
+    if (new_pwd != NULL && env_set(&shell->env, "PWD", new_pwd, 1) != 0)
+        status = 1;
+    if (print_target && new_pwd != NULL
+        && (shell_write_text(STDOUT_FILENO, new_pwd) != 0
+            || shell_write_text(STDOUT_FILENO, "\n") != 0))
+        status = 1;
     free(old_pwd);
     free(new_pwd);
-    return ferror(stdout) ? 1 : 0;
+    return status;
 }
 
 static int builtin_env(t_shell *shell, char **argv)
@@ -149,8 +151,7 @@ static int builtin_env(t_shell *shell, char **argv)
         fprintf(stderr, "small-shell: env: arguments are not supported\n");
         return 1;
     }
-    env_print(shell->env, 0);
-    return ferror(stdout) ? 1 : 0;
+    return env_print(shell->env, 0);
 }
 
 static int split_assignment(const char *arg, char **key, const char **value)
@@ -167,20 +168,32 @@ static int split_assignment(const char *arg, char **key, const char **value)
     return 0;
 }
 
-static int builtin_export(t_shell *shell, char **argv)
+static int valid_assignment_name(const char *key)
 {
     size_t i;
-    int status;
 
-    if (argv[1] == NULL) {
-        env_print(shell->env, 1);
-        return ferror(stdout) ? 1 : 0;
+    if (!sh_is_name_start((unsigned char)key[0]))
+        return 0;
+    i = 1;
+    while (key[i] != '\0') {
+        if (!sh_is_name_char((unsigned char)key[i]))
+            return 0;
+        i++;
     }
+    return 1;
+}
 
+static int builtin_export(t_shell *shell, char **argv)
+{
+    size_t  i;
+    int     status;
+
+    if (argv[1] == NULL)
+        return env_print(shell->env, 1);
     status = 0;
     for (i = 1; argv[i] != NULL; i++) {
-        char *key;
-        const char *value;
+        char        *key;
+        const char  *value;
 
         key = NULL;
         value = NULL;
@@ -188,27 +201,14 @@ static int builtin_export(t_shell *shell, char **argv)
             fprintf(stderr, "small-shell: export: allocation failure\n");
             return 1;
         }
-        if (!sh_is_name_start((unsigned char)key[0])) {
-            fprintf(stderr, "small-shell: export: `%s': not a valid identifier\n", argv[i]);
+        if (!valid_assignment_name(key)) {
+            fprintf(stderr,
+                "small-shell: export: `%s': not a valid identifier\n",
+                argv[i]);
             free(key);
             status = 1;
             continue;
         }
-        {
-            size_t j;
-
-            for (j = 1; key[j] != '\0'; j++) {
-                if (!sh_is_name_char((unsigned char)key[j])) {
-                    fprintf(stderr, "small-shell: export: `%s': not a valid identifier\n", argv[i]);
-                    free(key);
-                    key = NULL;
-                    status = 1;
-                    break;
-                }
-            }
-        }
-        if (key == NULL)
-            continue;
         if (env_set(&shell->env, key, value, 1) != 0) {
             fprintf(stderr, "small-shell: export: allocation failure\n");
             free(key);
@@ -230,8 +230,8 @@ static int builtin_unset(t_shell *shell, char **argv)
 
 static int parse_exit_status(const char *s, int *status)
 {
-    char *end;
-    long value;
+    char    *end;
+    long    value;
 
     errno = 0;
     value = strtol(s, &end, 10);
@@ -250,7 +250,8 @@ static int builtin_exit(t_shell *shell, char **argv)
         return shell->last_status;
     }
     if (!parse_exit_status(argv[1], &status)) {
-        fprintf(stderr, "small-shell: exit: %s: numeric argument required\n", argv[1]);
+        fprintf(stderr, "small-shell: exit: %s: numeric argument required\n",
+            argv[1]);
         shell->last_status = 2;
         shell->running = 0;
         return 2;
diff --git a/src/env.c b/src/env.c
index 650dbba..29df3bb 100644
--- a/src/env.c
+++ b/src/env.c
@@ -1,7 +1,9 @@
 #include "shell.h"
-#include <stdio.h>
+#include "runtime.h"
+
 #include <stdlib.h>
 #include <string.h>
+#include <unistd.h>
 
 static t_env *env_new(const char *key, const char *value, int exported)
 {
@@ -206,17 +208,27 @@ char **env_to_environ(t_env *env)
     return out;
 }
 
-void env_print(t_env *env, int declare_style)
+int env_print(t_env *env, int declare_style)
 {
     while (env != NULL) {
         if (env->key != NULL && env->exported) {
-            if (declare_style)
-                printf("declare -x %s=\"%s\"\n", env->key, env->value);
-            else
-                printf("%s=%s\n", env->key, env->value);
+            if (declare_style
+                && (shell_write_text(STDOUT_FILENO, "declare -x ") != 0
+                    || shell_write_text(STDOUT_FILENO, env->key) != 0
+                    || shell_write_text(STDOUT_FILENO, "=\"") != 0
+                    || shell_write_text(STDOUT_FILENO, env->value) != 0
+                    || shell_write_text(STDOUT_FILENO, "\"\n") != 0))
+                return 1;
+            if (!declare_style
+                && (shell_write_text(STDOUT_FILENO, env->key) != 0
+                    || shell_write_text(STDOUT_FILENO, "=") != 0
+                    || shell_write_text(STDOUT_FILENO, env->value) != 0
+                    || shell_write_text(STDOUT_FILENO, "\n") != 0))
+                return 1;
         }
         env = env->next;
     }
+    return 0;
 }
 
 static int is_sentinel(t_env *env)
diff --git a/src/input.c b/src/input.c
index 9c226ea..c52f5d8 100644
--- a/src/input.c
+++ b/src/input.c
@@ -22,9 +22,10 @@ static char *read_plain_line(const char *prompt, int interactive, int *failed)
     char    *line;
 
     *failed = 0;
-    if (interactive && prompt != NULL) {
-        fputs(prompt, stderr);
-        fflush(stderr);
+    if (interactive && prompt != NULL
+        && shell_write_text(STDERR_FILENO, prompt) != 0) {
+        *failed = 1;
+        return NULL;
     }
     cap = 128;
     len = 0;
@@ -117,5 +118,5 @@ void shell_loop(t_shell *shell)
         free(line);
     }
     if (interactive && shell->running)
-        fputc('\n', stderr);
+        (void)shell_write_text(STDERR_FILENO, "\n");
 }
diff --git a/src/redirection.c b/src/redirection.c
index 2e9c99e..5c9362e 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -163,8 +163,6 @@ int exec_run_parent_command(t_shell *shell, const t_command *command,
         status = 0;
     else
         status = builtin_run(shell, command->argv);
-    if (fflush(stdout) == EOF)
-        status = 1;
     {
         int restore_result;
 
diff --git a/src/runtime.c b/src/runtime.c
index 47d18f8..3280381 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -174,3 +174,38 @@ ssize_t shell_read(int fd, void *buffer, size_t size)
 {
     return read(fd, buffer, size);
 }
+
+ssize_t shell_write(int fd, const void *buffer, size_t size)
+{
+    return write(fd, buffer, size);
+}
+
+int shell_write_all(int fd, const void *buffer, size_t size)
+{
+    const unsigned char *cursor;
+
+    cursor = (const unsigned char *)buffer;
+    while (size > 0) {
+        ssize_t written;
+
+        written = shell_write(fd, cursor, size);
+        if (written > 0) {
+            cursor += (size_t)written;
+            size -= (size_t)written;
+        } else if (written < 0 && errno == EINTR) {
+            continue;
+        } else {
+            if (written == 0)
+                errno = EIO;
+            return 1;
+        }
+    }
+    return 0;
+}
+
+int shell_write_text(int fd, const char *text)
+{
+    if (text == NULL)
+        return 0;
+    return shell_write_all(fd, text, strlen(text));
+}
diff --git a/src/runtime.h b/src/runtime.h
index a78440d..c5703dd 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -19,5 +19,8 @@ int     shell_fflush(FILE *stream);
 int     shell_fseek(FILE *stream, long offset, int whence);
 int     shell_fileno(FILE *stream);
 ssize_t shell_read(int fd, void *buffer, size_t size);
+ssize_t shell_write(int fd, const void *buffer, size_t size);
+int     shell_write_all(int fd, const void *buffer, size_t size);
+int     shell_write_text(int fd, const char *text);
 
 #endif


## `refactor(env): 사용하지 않는 환경 저장소 래퍼 제거`

diff --git a/include/shell.h b/include/shell.h
index 4cea593..3bfd1cb 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -117,14 +117,7 @@ int         env_set(t_env **env, const char *key, const char *value, int exporte
 int         env_unset(t_env **env, const char *key);
 char        **env_to_environ(t_env *env);
 int         env_print(t_env *env, int declare_style);
-int         shell_env_init(t_env *env, char **envp);
-void        shell_env_free(t_env *env);
-const char  *shell_env_get(const t_env *env, const char *key);
-int         shell_env_set(t_env *env, const char *key, const char *value, int exported);
-int         shell_env_unset(t_env *env, const char *key);
 int         shell_env_is_valid_name(const char *key);
-char        **shell_env_export_list(t_env *env);
-char        **shell_env_to_envp(t_env *env);
 
 char        *expand_word(t_shell *shell, const char *word);
 int         expand_pipeline(t_shell *shell, t_pipeline *pipeline);
diff --git a/src/env.c b/src/env.c
index 29df3bb..5b2ab95 100644
--- a/src/env.c
+++ b/src/env.c
@@ -230,95 +230,3 @@ int env_print(t_env *env, int declare_style)
     }
     return 0;
 }
-
-static int is_sentinel(t_env *env)
-{
-    return (env != NULL && env->key == NULL && env->value == NULL
-        && env->exported == 0);
-}
-
-static t_env *env_head(t_env *env)
-{
-    if (is_sentinel(env))
-        return env->next;
-    return env;
-}
-
-static const t_env *env_head_const(const t_env *env)
-{
-    if (env != NULL && env->key == NULL && env->value == NULL
-        && env->exported == 0)
-        return env->next;
-    return env;
-}
-
-int shell_env_init(t_env *env, char **envp)
-{
-    if (env == NULL)
-        return 1;
-    env->key = NULL;
-    env->value = NULL;
-    env->exported = 0;
-    env->next = env_from_environ(envp);
-    return (envp != NULL && envp[0] != NULL && env->next == NULL);
-}
-
-void shell_env_free(t_env *env)
-{
-    if (env == NULL)
-        return;
-    if (is_sentinel(env)) {
-        env_free(env->next);
-        env->next = NULL;
-        return;
-    }
-    env_free(env);
-}
-
-const char *shell_env_get(const t_env *env, const char *key)
-{
-    const t_env *node;
-
-    node = env_head_const(env);
-    while (node != NULL) {
-        if (node->key != NULL && key != NULL
-            && strcmp(node->key, key) == 0)
-            return node->value;
-        node = node->next;
-    }
-    return NULL;
-}
-
-int shell_env_set(t_env *env, const char *key, const char *value, int exported)
-{
-    t_env *head;
-
-    if (env == NULL)
-        return 1;
-    if (is_sentinel(env))
-        return env_set(&env->next, key, value, exported);
-    head = env;
-    return env_set(&head, key, value, exported);
-}
-
-int shell_env_unset(t_env *env, const char *key)
-{
-    t_env *head;
-
-    if (env == NULL)
-        return 1;
-    if (is_sentinel(env))
-        return env_unset(&env->next, key);
-    head = env;
-    return env_unset(&head, key);
-}
-
-char **shell_env_export_list(t_env *env)
-{
-    return env_to_environ(env_head(env));
-}
-
-char **shell_env_to_envp(t_env *env)
-{
-    return env_to_environ(env_head(env));
-}
