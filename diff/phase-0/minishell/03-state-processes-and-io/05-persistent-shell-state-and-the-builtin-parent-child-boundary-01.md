# 지속되는 셸 상태와 builtin의 부모·자식 경계

## `feat(utils): 환경 식별자 문자 판정 제공`

diff --git a/include/shell.h b/include/shell.h
index 7a5314a..bdb3491 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -17,5 +17,7 @@ void    sh_free_words(char **words);
 char    *shell_strndup(const char *s, size_t len);
 char    *shell_itoa_status(int status);
 void    shell_strv_free(char **words);
+int     sh_is_name_char(int c);
+int     sh_is_name_start(int c);
 
 #endif
diff --git a/src/utils.c b/src/utils.c
index d11de78..c76e855 100644
--- a/src/utils.c
+++ b/src/utils.c
@@ -1,5 +1,6 @@
 #include "shell.h"
 
+#include <ctype.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
@@ -128,3 +129,11 @@ char *shell_itoa_status(int status) {
 void shell_strv_free(char **words) {
     sh_free_words(words);
 }
+
+int sh_is_name_start(int c) {
+    return (isalpha((unsigned char)c) || c == '_');
+}
+
+int sh_is_name_char(int c) {
+    return (isalnum((unsigned char)c) || c == '_');
+}


## `feat(env): 프로세스 환경 적재와 수명 관리`

diff --git a/Makefile b/Makefile
index dd2a8f7..14ac270 100644
--- a/Makefile
+++ b/Makefile
@@ -8,6 +8,7 @@ TARGET := small-shell
 SRCS := \
 	src/main.c \
 	src/input.c \
+	src/env.c \
 	src/utils.c
 OBJS := $(SRCS:.c=.o)
 
diff --git a/include/shell.h b/include/shell.h
index bdb3491..32cb099 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -3,8 +3,17 @@
 
 # include <stddef.h>
 
+typedef struct s_env {
+    char            *key;
+    char            *value;
+    int             exported;
+    struct s_env    *next;
+}   t_env;
+
 typedef struct s_shell {
-    int running;
+    t_env   *env;
+    int     last_status;
+    int     running;
 }   t_shell;
 
 char    *shell_read_line(const char *prompt, int interactive);
@@ -19,5 +28,7 @@ char    *shell_itoa_status(int status);
 void    shell_strv_free(char **words);
 int     sh_is_name_char(int c);
 int     sh_is_name_start(int c);
+t_env   *env_from_environ(char **envp);
+void    env_free(t_env *env);
 
 #endif
diff --git a/src/env.c b/src/env.c
new file mode 100644
index 0000000..69f236b
--- /dev/null
+++ b/src/env.c
@@ -0,0 +1,48 @@
+#include "shell.h"
+
+#include <stdlib.h>
+#include <string.h>
+
+static t_env *env_new(const char *key, const char *value, int exported) {
+    t_env *node = (t_env *)sh_xcalloc(1, sizeof(t_env));
+    node->key = sh_strdup(key);
+    node->value = sh_strdup(value ? value : "");
+    node->exported = exported;
+    return node;
+}
+
+t_env *env_from_environ(char **envp) {
+    t_env *head = NULL;
+    t_env *tail = NULL;
+    size_t i;
+
+    if (!envp)
+        return NULL;
+    for (i = 0; envp[i]; i++) {
+        const char *eq = strchr(envp[i], '=');
+        char *key;
+        t_env *node;
+
+        if (!eq)
+            continue;
+        key = sh_substr(envp[i], 0, (size_t)(eq - envp[i]));
+        node = env_new(key, eq + 1, 1);
+        free(key);
+        if (!head)
+            head = node;
+        else
+            tail->next = node;
+        tail = node;
+    }
+    return head;
+}
+
+void env_free(t_env *env) {
+    while (env) {
+        t_env *next = env->next;
+        free(env->key);
+        free(env->value);
+        free(env);
+        env = next;
+    }
+}
diff --git a/src/main.c b/src/main.c
index ec782e2..82023cc 100644
--- a/src/main.c
+++ b/src/main.c
@@ -2,13 +2,23 @@
 
 #include "shell.h"
 
-int main(int argc, char **argv)
+static int normalize_status(int status)
+{
+    return status & 0xff;
+}
+
+int main(int argc, char **argv, char **envp)
 {
     t_shell shell;
+    int result;
 
     (void)argc;
     (void)argv;
+    shell.env = env_from_environ(envp);
+    shell.last_status = 0;
     shell.running = 1;
     shell_loop(&shell);
-    return 0;
+    result = shell.last_status;
+    env_free(shell.env);
+    return normalize_status(result);
 }


## `feat(env): 환경 조회와 변경 연산 제공`

diff --git a/include/shell.h b/include/shell.h
index 32cb099..4f2b323 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -30,5 +30,9 @@ int     sh_is_name_char(int c);
 int     sh_is_name_start(int c);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
+const char  *env_get(t_env *env, const char *key);
+int     env_set(t_env **env, const char *key, const char *value, int exported);
+int     env_unset(t_env **env, const char *key);
+int     shell_env_is_valid_name(const char *key);
 
 #endif
diff --git a/src/env.c b/src/env.c
index 69f236b..6d3c86a 100644
--- a/src/env.c
+++ b/src/env.c
@@ -7,10 +7,33 @@ static t_env *env_new(const char *key, const char *value, int exported) {
     t_env *node = (t_env *)sh_xcalloc(1, sizeof(t_env));
     node->key = sh_strdup(key);
     node->value = sh_strdup(value ? value : "");
-    node->exported = exported;
+    node->exported = exported ? 1 : 0;
     return node;
 }
 
+static t_env *env_find(t_env *env, const char *key) {
+    while (env) {
+        if (env->key && key && strcmp(env->key, key) == 0)
+            return env;
+        env = env->next;
+    }
+    return NULL;
+}
+
+int shell_env_is_valid_name(const char *key) {
+    size_t i;
+
+    if (!key || !sh_is_name_start((unsigned char)key[0]))
+        return 0;
+    i = 1;
+    while (key[i]) {
+        if (!sh_is_name_char((unsigned char)key[i]))
+            return 0;
+        i++;
+    }
+    return 1;
+}
+
 t_env *env_from_environ(char **envp) {
     t_env *head = NULL;
     t_env *tail = NULL;
@@ -46,3 +69,64 @@ void env_free(t_env *env) {
         env = next;
     }
 }
+
+const char *env_get(t_env *env, const char *key) {
+    t_env *node;
+
+    node = env_find(env, key);
+    if (!node)
+        return "";
+    return node->value;
+}
+
+int env_set(t_env **env, const char *key, const char *value, int exported) {
+    t_env *node;
+    t_env *tail;
+
+    if (!env || !shell_env_is_valid_name(key))
+        return 1;
+    node = env_find(*env, key);
+    if (node) {
+        if (value) {
+            free(node->value);
+            node->value = sh_strdup(value);
+        }
+        if (exported)
+            node->exported = 1;
+        return 0;
+    }
+    node = env_new(key, value ? value : "", exported);
+    if (!*env) {
+        *env = node;
+        return 0;
+    }
+    tail = *env;
+    while (tail->next)
+        tail = tail->next;
+    tail->next = node;
+    return 0;
+}
+
+int env_unset(t_env **env, const char *key) {
+    t_env *cur;
+    t_env *prev;
+
+    if (!env || !key)
+        return 1;
+    cur = *env;
+    prev = NULL;
+    while (cur) {
+        if (cur->key && strcmp(cur->key, key) == 0) {
+            if (prev)
+                prev->next = cur->next;
+            else
+                *env = cur->next;
+            cur->next = NULL;
+            env_free(cur);
+            return 0;
+        }
+        prev = cur;
+        cur = cur->next;
+    }
+    return 0;
+}


## `feat(env): export 배열과 출력 뷰 생성`

diff --git a/include/shell.h b/include/shell.h
index 4f2b323..2ffd930 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -33,6 +33,8 @@ void    env_free(t_env *env);
 const char  *env_get(t_env *env, const char *key);
 int     env_set(t_env **env, const char *key, const char *value, int exported);
 int     env_unset(t_env **env, const char *key);
+char    **env_to_environ(t_env *env);
+void    env_print(t_env *env, int declare_style);
 int     shell_env_is_valid_name(const char *key);
 
 #endif
diff --git a/src/env.c b/src/env.c
index 6d3c86a..bf7e9de 100644
--- a/src/env.c
+++ b/src/env.c
@@ -1,5 +1,6 @@
 #include "shell.h"
 
+#include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
 
@@ -130,3 +131,43 @@ int env_unset(t_env **env, const char *key) {
     }
     return 0;
 }
+
+char **env_to_environ(t_env *env) {
+    size_t  count;
+    size_t  i;
+    char    **out;
+    char    *pair;
+    t_env   *cur;
+
+    count = 0;
+    cur = env;
+    while (cur) {
+        if (cur->key && cur->exported)
+            count++;
+        cur = cur->next;
+    }
+    out = (char **)sh_xcalloc(count + 1, sizeof(char *));
+    i = 0;
+    cur = env;
+    while (cur) {
+        if (cur->key && cur->exported) {
+            pair = sh_strjoin_free(sh_strdup(cur->key), "=");
+            pair = sh_strjoin_free(pair, cur->value);
+            out[i++] = pair;
+        }
+        cur = cur->next;
+    }
+    return out;
+}
+
+void env_print(t_env *env, int declare_style) {
+    while (env) {
+        if (env->key && env->exported) {
+            if (declare_style)
+                printf("declare -x %s=\"%s\"\n", env->key, env->value);
+            else
+                printf("%s=%s\n", env->key, env->value);
+        }
+        env = env->next;
+    }
+}


## `feat(env): 공개 환경 저장소 어댑터 제공`

diff --git a/include/shell.h b/include/shell.h
index 2ffd930..205b1de 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -35,6 +35,14 @@ int     env_set(t_env **env, const char *key, const char *value, int exported);
 int     env_unset(t_env **env, const char *key);
 char    **env_to_environ(t_env *env);
 void    env_print(t_env *env, int declare_style);
+int     shell_env_init(t_env *env, char **envp);
+void    shell_env_free(t_env *env);
+const char  *shell_env_get(const t_env *env, const char *key);
+int     shell_env_set(t_env *env, const char *key, const char *value,
+            int exported);
+int     shell_env_unset(t_env *env, const char *key);
 int     shell_env_is_valid_name(const char *key);
+char    **shell_env_export_list(t_env *env);
+char    **shell_env_to_envp(t_env *env);
 
 #endif
diff --git a/src/env.c b/src/env.c
index bf7e9de..97cd087 100644
--- a/src/env.c
+++ b/src/env.c
@@ -5,7 +5,9 @@
 #include <string.h>
 
 static t_env *env_new(const char *key, const char *value, int exported) {
-    t_env *node = (t_env *)sh_xcalloc(1, sizeof(t_env));
+    t_env *node;
+
+    node = (t_env *)sh_xcalloc(1, sizeof(t_env));
     node->key = sh_strdup(key);
     node->value = sh_strdup(value ? value : "");
     node->exported = exported ? 1 : 0;
@@ -36,34 +38,38 @@ int shell_env_is_valid_name(const char *key) {
 }
 
 t_env *env_from_environ(char **envp) {
-    t_env *head = NULL;
-    t_env *tail = NULL;
-    size_t i;
+    t_env   *head;
+    t_env   *tail;
+    size_t  i;
+    char    *eq;
+    char    *key;
+    t_env   *node;
 
-    if (!envp)
-        return NULL;
-    for (i = 0; envp[i]; i++) {
-        const char *eq = strchr(envp[i], '=');
-        char *key;
-        t_env *node;
-
-        if (!eq)
-            continue;
-        key = sh_substr(envp[i], 0, (size_t)(eq - envp[i]));
-        node = env_new(key, eq + 1, 1);
-        free(key);
-        if (!head)
-            head = node;
-        else
-            tail->next = node;
-        tail = node;
+    head = NULL;
+    tail = NULL;
+    i = 0;
+    while (envp && envp[i]) {
+        eq = strchr(envp[i], '=');
+        if (eq) {
+            key = sh_substr(envp[i], 0, (size_t)(eq - envp[i]));
+            node = env_new(key, eq + 1, 1);
+            free(key);
+            if (!head)
+                head = node;
+            else
+                tail->next = node;
+            tail = node;
+        }
+        i++;
     }
     return head;
 }
 
 void env_free(t_env *env) {
+    t_env *next;
+
     while (env) {
-        t_env *next = env->next;
+        next = env->next;
         free(env->key);
         free(env->value);
         free(env);
@@ -171,3 +177,82 @@ void env_print(t_env *env, int declare_style) {
         env = env->next;
     }
 }
+
+static int is_sentinel(t_env *env) {
+    return (env && !env->key && !env->value && env->exported == 0);
+}
+
+static t_env *env_head(t_env *env) {
+    if (is_sentinel(env))
+        return env->next;
+    return env;
+}
+
+static const t_env *env_head_const(const t_env *env) {
+    if (env && !env->key && !env->value && env->exported == 0)
+        return env->next;
+    return env;
+}
+
+int shell_env_init(t_env *env, char **envp) {
+    if (!env)
+        return 1;
+    env->key = NULL;
+    env->value = NULL;
+    env->exported = 0;
+    env->next = env_from_environ(envp);
+    return 0;
+}
+
+void shell_env_free(t_env *env) {
+    if (!env)
+        return;
+    if (is_sentinel(env)) {
+        env_free(env->next);
+        env->next = NULL;
+        return;
+    }
+    env_free(env);
+}
+
+const char *shell_env_get(const t_env *env, const char *key) {
+    const t_env *node;
+
+    node = env_head_const(env);
+    while (node) {
+        if (node->key && key && strcmp(node->key, key) == 0)
+            return node->value;
+        node = node->next;
+    }
+    return NULL;
+}
+
+int shell_env_set(t_env *env, const char *key, const char *value, int exported) {
+    t_env *head;
+
+    if (!env)
+        return 1;
+    if (is_sentinel(env))
+        return env_set(&env->next, key, value, exported);
+    head = env;
+    return env_set(&head, key, value, exported);
+}
+
+int shell_env_unset(t_env *env, const char *key) {
+    t_env *head;
+
+    if (!env)
+        return 1;
+    if (is_sentinel(env))
+        return env_unset(&env->next, key);
+    head = env;
+    return env_unset(&head, key);
+}
+
+char **shell_env_export_list(t_env *env) {
+    return env_to_environ(env_head(env));
+}
+
+char **shell_env_to_envp(t_env *env) {
+    return env_to_environ(env_head(env));
+}
diff --git a/src/main.c b/src/main.c
index 82023cc..ab57407 100644
--- a/src/main.c
+++ b/src/main.c
@@ -17,6 +17,7 @@ int main(int argc, char **argv, char **envp)
     shell.env = env_from_environ(envp);
     shell.last_status = 0;
     shell.running = 1;
+
     shell_loop(&shell);
     result = shell.last_status;
     env_free(shell.env);


## `feat(builtin): echo 출력 명령 제공`

diff --git a/Makefile b/Makefile
index 49fbba9..33658ca 100644
--- a/Makefile
+++ b/Makefile
@@ -12,7 +12,8 @@ SRCS := \
 	src/parser.c \
 	src/expand.c \
 	src/env.c \
-	src/utils.c
+	src/utils.c \
+	src/builtin.c
 OBJS := $(SRCS:.c=.o)
 
 ifeq ($(USE_READLINE),1)
diff --git a/include/shell.h b/include/shell.h
index 23c26da..d037ea8 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -127,5 +127,8 @@ int     shell_env_unset(t_env *env, const char *key);
 int     shell_env_is_valid_name(const char *key);
 char    **shell_env_export_list(t_env *env);
 char    **shell_env_to_envp(t_env *env);
+int     builtin_is_parent(const char *name);
+int     builtin_is_known(const char *name);
+int     builtin_run(t_shell *shell, char **argv);
 
 #endif
diff --git a/src/builtin.c b/src/builtin.c
new file mode 100644
index 0000000..387bef9
--- /dev/null
+++ b/src/builtin.c
@@ -0,0 +1,74 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "shell.h"
+
+#include <errno.h>
+#include <stdio.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+int builtin_is_known(const char *name)
+{
+    static const char *builtins[] = {
+        "echo",
+        NULL
+    };
+    size_t i;
+
+    if (name == NULL)
+        return 0;
+    for (i = 0; builtins[i] != NULL; i++) {
+        if (strcmp(name, builtins[i]) == 0)
+            return 1;
+    }
+    return 0;
+}
+
+int builtin_is_parent(const char *name)
+{
+    return builtin_is_known(name);
+}
+
+static int builtin_echo(char **argv)
+{
+    size_t i;
+    int newline;
+
+    newline = 1;
+    i = 1;
+    while (argv[i] != NULL && argv[i][0] == '-' && argv[i][1] == 'n') {
+        size_t j;
+        int only_n;
+
+        only_n = 1;
+        for (j = 1; argv[i][j] != '\0'; j++) {
+            if (argv[i][j] != 'n') {
+                only_n = 0;
+                break;
+            }
+        }
+        if (!only_n)
+            break;
+        newline = 0;
+        i++;
+    }
+    while (argv[i] != NULL) {
+        fputs(argv[i], stdout);
+        if (argv[i + 1] != NULL)
+            fputc(' ', stdout);
+        i++;
+    }
+    if (newline)
+        fputc('\n', stdout);
+    return ferror(stdout) ? 1 : 0;
+}
+
+int builtin_run(t_shell *shell, char **argv)
+{
+    if (shell == NULL || argv == NULL || argv[0] == NULL)
+        return 0;
+    if (strcmp(argv[0], "echo") == 0)
+        return builtin_echo(argv);
+    return 127;
+}


## `feat(builtin): pwd 작업 디렉터리 출력`

diff --git a/src/builtin.c b/src/builtin.c
index 387bef9..8ab7eaa 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -12,6 +12,7 @@ int builtin_is_known(const char *name)
 {
     static const char *builtins[] = {
         "echo",
+        "pwd",
         NULL
     };
     size_t i;
@@ -64,11 +65,27 @@ static int builtin_echo(char **argv)
     return ferror(stdout) ? 1 : 0;
 }
 
+static int builtin_pwd(void)
+{
+    char *cwd;
+
+    cwd = getcwd(NULL, 0);
+    if (cwd == NULL) {
+        fprintf(stderr, "small-shell: pwd: %s\n", strerror(errno));
+        return 1;
+    }
+    printf("%s\n", cwd);
+    free(cwd);
+    return ferror(stdout) ? 1 : 0;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
         return 0;
     if (strcmp(argv[0], "echo") == 0)
         return builtin_echo(argv);
+    if (strcmp(argv[0], "pwd") == 0)
+        return builtin_pwd();
     return 127;
 }


## `feat(builtin): cd 이동과 PWD 상태 동기화`

diff --git a/src/builtin.c b/src/builtin.c
index 8ab7eaa..18e319b 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -13,6 +13,7 @@ int builtin_is_known(const char *name)
     static const char *builtins[] = {
         "echo",
         "pwd",
+        "cd",
         NULL
     };
     size_t i;
@@ -79,6 +80,62 @@ static int builtin_pwd(void)
     return ferror(stdout) ? 1 : 0;
 }
 
+static size_t argv_count(char **argv)
+{
+    size_t count;
+
+    count = 0;
+    while (argv != NULL && argv[count] != NULL)
+        count++;
+    return count;
+}
+
+static int builtin_cd(t_shell *shell, char **argv)
+{
+    const char *target;
+    char *old_pwd;
+    char *new_pwd;
+    int print_target;
+
+    if (argv_count(argv) > 2) {
+        fprintf(stderr, "small-shell: cd: too many arguments\n");
+        return 1;
+    }
+    print_target = 0;
+    if (argv[1] == NULL) {
+        target = env_get(shell->env, "HOME");
+        if (target == NULL || target[0] == '\0') {
+            fprintf(stderr, "small-shell: cd: HOME not set\n");
+            return 1;
+        }
+    } else if (strcmp(argv[1], "-") == 0) {
+        target = env_get(shell->env, "OLDPWD");
+        if (target == NULL || target[0] == '\0') {
+            fprintf(stderr, "small-shell: cd: OLDPWD not set\n");
+            return 1;
+        }
+        print_target = 1;
+    } else {
+        target = argv[1];
+    }
+    old_pwd = getcwd(NULL, 0);
+    if (chdir(target) != 0) {
+        fprintf(stderr, "small-shell: cd: %s: %s\n", target, strerror(errno));
+        free(old_pwd);
+        return 1;
+    }
+    new_pwd = getcwd(NULL, 0);
+    if (old_pwd != NULL)
+        (void)env_set(&shell->env, "OLDPWD", old_pwd, 1);
+    if (new_pwd != NULL)
+        (void)env_set(&shell->env, "PWD", new_pwd, 1);
+    if (print_target && new_pwd != NULL)
+        printf("%s\n", new_pwd);
+    free(old_pwd);
+    free(new_pwd);
+    return ferror(stdout) ? 1 : 0;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
@@ -87,5 +144,7 @@ int builtin_run(t_shell *shell, char **argv)
         return builtin_echo(argv);
     if (strcmp(argv[0], "pwd") == 0)
         return builtin_pwd();
+    if (strcmp(argv[0], "cd") == 0)
+        return builtin_cd(shell, argv);
     return 127;
 }


## `feat(builtin): env 환경 목록 출력`

diff --git a/src/builtin.c b/src/builtin.c
index 18e319b..0d8893d 100644
--- a/src/builtin.c
+++ b/src/builtin.c
@@ -14,6 +14,7 @@ int builtin_is_known(const char *name)
         "echo",
         "pwd",
         "cd",
+        "env",
         NULL
     };
     size_t i;
@@ -136,6 +137,16 @@ static int builtin_cd(t_shell *shell, char **argv)
     return ferror(stdout) ? 1 : 0;
 }
 
+static int builtin_env(t_shell *shell, char **argv)
+{
+    if (argv[1] != NULL) {
+        fprintf(stderr, "small-shell: env: arguments are not supported\n");
+        return 1;
+    }
+    env_print(shell->env, 0);
+    return ferror(stdout) ? 1 : 0;
+}
+
 int builtin_run(t_shell *shell, char **argv)
 {
     if (shell == NULL || argv == NULL || argv[0] == NULL)
@@ -146,5 +157,7 @@ int builtin_run(t_shell *shell, char **argv)
         return builtin_pwd();
     if (strcmp(argv[0], "cd") == 0)
         return builtin_cd(shell, argv);
+    if (strcmp(argv[0], "env") == 0)
+        return builtin_env(shell, argv);
     return 127;
 }


