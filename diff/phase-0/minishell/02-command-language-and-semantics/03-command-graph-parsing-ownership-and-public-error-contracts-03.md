## `fix(memory): 구조화 단계의 할당 실패를 명령 오류로 전파`

diff --git a/include/shell.h b/include/shell.h
index e3c1919..828c13d 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -94,7 +94,7 @@ typedef struct s_executor_hooks {
 char        *sh_strdup(const char *s);
 char        *sh_substr(const char *s, size_t start, size_t len);
 char        *sh_strjoin_free(char *left, const char *right);
-void        *sh_xcalloc(size_t count, size_t size);
+void        *sh_calloc(size_t count, size_t size);
 void        sh_free_words(char **words);
 char        *shell_strndup(const char *s, size_t len);
 char        *shell_itoa_status(int status);
diff --git a/src/env.c b/src/env.c
index 97cd087..650dbba 100644
--- a/src/env.c
+++ b/src/env.c
@@ -1,35 +1,46 @@
 #include "shell.h"
-
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
 
-static t_env *env_new(const char *key, const char *value, int exported) {
+static t_env *env_new(const char *key, const char *value, int exported)
+{
     t_env *node;
 
-    node = (t_env *)sh_xcalloc(1, sizeof(t_env));
+    node = (t_env *)sh_calloc(1, sizeof(t_env));
+    if (node == NULL)
+        return NULL;
     node->key = sh_strdup(key);
-    node->value = sh_strdup(value ? value : "");
+    node->value = sh_strdup(value != NULL ? value : "");
+    if (node->key == NULL || node->value == NULL) {
+        free(node->key);
+        free(node->value);
+        free(node);
+        return NULL;
+    }
     node->exported = exported ? 1 : 0;
     return node;
 }
 
-static t_env *env_find(t_env *env, const char *key) {
-    while (env) {
-        if (env->key && key && strcmp(env->key, key) == 0)
+static t_env *env_find(t_env *env, const char *key)
+{
+    while (env != NULL) {
+        if (env->key != NULL && key != NULL
+            && strcmp(env->key, key) == 0)
             return env;
         env = env->next;
     }
     return NULL;
 }
 
-int shell_env_is_valid_name(const char *key) {
+int shell_env_is_valid_name(const char *key)
+{
     size_t i;
 
-    if (!key || !sh_is_name_start((unsigned char)key[0]))
+    if (key == NULL || !sh_is_name_start((unsigned char)key[0]))
         return 0;
     i = 1;
-    while (key[i]) {
+    while (key[i] != '\0') {
         if (!sh_is_name_char((unsigned char)key[i]))
             return 0;
         i++;
@@ -37,24 +48,34 @@ int shell_env_is_valid_name(const char *key) {
     return 1;
 }
 
-t_env *env_from_environ(char **envp) {
+t_env *env_from_environ(char **envp)
+{
     t_env   *head;
     t_env   *tail;
     size_t  i;
-    char    *eq;
-    char    *key;
-    t_env   *node;
 
     head = NULL;
     tail = NULL;
     i = 0;
-    while (envp && envp[i]) {
+    while (envp != NULL && envp[i] != NULL) {
+        char    *eq;
+        char    *key;
+        t_env   *node;
+
         eq = strchr(envp[i], '=');
-        if (eq) {
+        if (eq != NULL) {
             key = sh_substr(envp[i], 0, (size_t)(eq - envp[i]));
+            if (key == NULL) {
+                env_free(head);
+                return NULL;
+            }
             node = env_new(key, eq + 1, 1);
             free(key);
-            if (!head)
+            if (node == NULL) {
+                env_free(head);
+                return NULL;
+            }
+            if (head == NULL)
                 head = node;
             else
                 tail->next = node;
@@ -65,10 +86,11 @@ t_env *env_from_environ(char **envp) {
     return head;
 }
 
-void env_free(t_env *env) {
+void env_free(t_env *env)
+{
     t_env *next;
 
-    while (env) {
+    while (env != NULL) {
         next = env->next;
         free(env->key);
         free(env->value);
@@ -77,54 +99,64 @@ void env_free(t_env *env) {
     }
 }
 
-const char *env_get(t_env *env, const char *key) {
+const char *env_get(t_env *env, const char *key)
+{
     t_env *node;
 
     node = env_find(env, key);
-    if (!node)
+    if (node == NULL)
         return "";
     return node->value;
 }
 
-int env_set(t_env **env, const char *key, const char *value, int exported) {
+int env_set(t_env **env, const char *key, const char *value, int exported)
+{
     t_env *node;
     t_env *tail;
 
-    if (!env || !shell_env_is_valid_name(key))
+    if (env == NULL || !shell_env_is_valid_name(key))
         return 1;
     node = env_find(*env, key);
-    if (node) {
-        if (value) {
+    if (node != NULL) {
+        if (value != NULL) {
+            char *copy;
+
+            copy = sh_strdup(value);
+            if (copy == NULL)
+                return 1;
             free(node->value);
-            node->value = sh_strdup(value);
+            node->value = copy;
         }
         if (exported)
             node->exported = 1;
         return 0;
     }
-    node = env_new(key, value ? value : "", exported);
-    if (!*env) {
+    node = env_new(key, value != NULL ? value : "", exported);
+    if (node == NULL)
+        return 1;
+    if (*env == NULL) {
         *env = node;
         return 0;
     }
     tail = *env;
-    while (tail->next)
+    while (tail->next != NULL)
         tail = tail->next;
     tail->next = node;
     return 0;
 }
 
-int env_unset(t_env **env, const char *key) {
+int env_unset(t_env **env, const char *key)
+{
     t_env *cur;
     t_env *prev;
 
-    if (!env || !key)
+    if (env == NULL || key == NULL)
         return 1;
     cur = *env;
     prev = NULL;
-    while (cur) {
-        if (cur->key && strcmp(cur->key, key) == 0) {
-            if (prev)
+    while (cur != NULL) {
+        if (cur->key != NULL && strcmp(cur->key, key) == 0) {
+            if (prev != NULL)
                 prev->next = cur->next;
             else
                 *env = cur->next;
@@ -138,27 +170,35 @@ int env_unset(t_env **env, const char *key) {
     return 0;
 }
 
-char **env_to_environ(t_env *env) {
+char **env_to_environ(t_env *env)
+{
     size_t  count;
     size_t  i;
     char    **out;
-    char    *pair;
     t_env   *cur;
 
     count = 0;
     cur = env;
-    while (cur) {
-        if (cur->key && cur->exported)
+    while (cur != NULL) {
+        if (cur->key != NULL && cur->exported)
             count++;
         cur = cur->next;
     }
-    out = (char **)sh_xcalloc(count + 1, sizeof(char *));
+    out = (char **)sh_calloc(count + 1, sizeof(char *));
+    if (out == NULL)
+        return NULL;
     i = 0;
     cur = env;
-    while (cur) {
-        if (cur->key && cur->exported) {
+    while (cur != NULL) {
+        if (cur->key != NULL && cur->exported) {
+            char *pair;
+
             pair = sh_strjoin_free(sh_strdup(cur->key), "=");
             pair = sh_strjoin_free(pair, cur->value);
+            if (pair == NULL) {
+                sh_free_words(out);
+                return NULL;
+            }
             out[i++] = pair;
         }
         cur = cur->next;
@@ -166,9 +206,10 @@ char **env_to_environ(t_env *env) {
     return out;
 }
 
-void env_print(t_env *env, int declare_style) {
-    while (env) {
-        if (env->key && env->exported) {
+void env_print(t_env *env, int declare_style)
+{
+    while (env != NULL) {
+        if (env->key != NULL && env->exported) {
             if (declare_style)
                 printf("declare -x %s=\"%s\"\n", env->key, env->value);
             else
@@ -178,34 +219,41 @@ void env_print(t_env *env, int declare_style) {
     }
 }
 
-static int is_sentinel(t_env *env) {
-    return (env && !env->key && !env->value && env->exported == 0);
+static int is_sentinel(t_env *env)
+{
+    return (env != NULL && env->key == NULL && env->value == NULL
+        && env->exported == 0);
 }
 
-static t_env *env_head(t_env *env) {
+static t_env *env_head(t_env *env)
+{
     if (is_sentinel(env))
         return env->next;
     return env;
 }
 
-static const t_env *env_head_const(const t_env *env) {
-    if (env && !env->key && !env->value && env->exported == 0)
+static const t_env *env_head_const(const t_env *env)
+{
+    if (env != NULL && env->key == NULL && env->value == NULL
+        && env->exported == 0)
         return env->next;
     return env;
 }
 
-int shell_env_init(t_env *env, char **envp) {
-    if (!env)
+int shell_env_init(t_env *env, char **envp)
+{
+    if (env == NULL)
         return 1;
     env->key = NULL;
     env->value = NULL;
     env->exported = 0;
     env->next = env_from_environ(envp);
-    return 0;
+    return (envp != NULL && envp[0] != NULL && env->next == NULL);
 }
 
-void shell_env_free(t_env *env) {
-    if (!env)
+void shell_env_free(t_env *env)
+{
+    if (env == NULL)
         return;
     if (is_sentinel(env)) {
         env_free(env->next);
@@ -215,22 +263,25 @@ void shell_env_free(t_env *env) {
     env_free(env);
 }
 
-const char *shell_env_get(const t_env *env, const char *key) {
+const char *shell_env_get(const t_env *env, const char *key)
+{
     const t_env *node;
 
     node = env_head_const(env);
-    while (node) {
-        if (node->key && key && strcmp(node->key, key) == 0)
+    while (node != NULL) {
+        if (node->key != NULL && key != NULL
+            && strcmp(node->key, key) == 0)
             return node->value;
         node = node->next;
     }
     return NULL;
 }
 
-int shell_env_set(t_env *env, const char *key, const char *value, int exported) {
+int shell_env_set(t_env *env, const char *key, const char *value, int exported)
+{
     t_env *head;
 
-    if (!env)
+    if (env == NULL)
         return 1;
     if (is_sentinel(env))
         return env_set(&env->next, key, value, exported);
@@ -238,10 +289,11 @@ int shell_env_set(t_env *env, const char *key, const char *value, int exported)
     return env_set(&head, key, value, exported);
 }
 
-int shell_env_unset(t_env *env, const char *key) {
+int shell_env_unset(t_env *env, const char *key)
+{
     t_env *head;
 
-    if (!env)
+    if (env == NULL)
         return 1;
     if (is_sentinel(env))
         return env_unset(&env->next, key);
@@ -249,10 +301,12 @@ int shell_env_unset(t_env *env, const char *key) {
     return env_unset(&head, key);
 }
 
-char **shell_env_export_list(t_env *env) {
+char **shell_env_export_list(t_env *env)
+{
     return env_to_environ(env_head(env));
 }
 
-char **shell_env_to_envp(t_env *env) {
+char **shell_env_to_envp(t_env *env)
+{
     return env_to_environ(env_head(env));
 }
diff --git a/src/exec.c b/src/exec.c
index 7b623aa..85f7a3b 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -222,8 +222,10 @@ static int execute_one_pipeline(t_shell *shell, t_pipeline *pipeline, const stru
 
     if (pipeline == NULL || pipeline->command_count == 0)
         return shell->last_status;
-    if (expand_one_pipeline(shell, pipeline) != 0)
+    if (expand_one_pipeline(shell, pipeline) != 0) {
+        fprintf(stderr, "small-shell: allocation failure\n");
         return 1;
+    }
     command = pipeline->commands;
     if (pipeline->command_count == 1
         && (command->argc == 0 || builtin_is_parent(command->argv[0])))
@@ -261,6 +263,13 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
     return execute_pipeline_list_ctx(shell, pipeline, &ctx);
 }
 
+static int process_error_status(const char *error)
+{
+    if (error != NULL && strcmp(error, "allocation failure") == 0)
+        return 1;
+    return 258;
+}
+
 int shell_process_line(t_shell *shell, const char *line)
 {
     t_token *tokens;
@@ -272,20 +281,26 @@ int shell_process_line(t_shell *shell, const char *line)
         return shell != NULL ? shell->last_status : 1;
 
     error = NULL;
+    errno = 0;
     tokens = tokenize_line(line, &error);
-    if (error != NULL) {
-        fprintf(stderr, "small-shell: %s\n", error);
+    if (error != NULL || (tokens == NULL && errno == ENOMEM)) {
+        shell->last_status = error != NULL
+            ? process_error_status(error) : 1;
+        fprintf(stderr, "small-shell: %s\n",
+            error != NULL ? error : "allocation failure");
         free(error);
-        shell->last_status = 258;
         return shell->last_status;
     }
 
+    errno = 0;
     pipelines = parse_tokens(tokens, &error);
     free_tokens(tokens);
-    if (error != NULL) {
-        fprintf(stderr, "small-shell: %s\n", error);
+    if (error != NULL || (pipelines == NULL && errno == ENOMEM)) {
+        shell->last_status = error != NULL
+            ? process_error_status(error) : 1;
+        fprintf(stderr, "small-shell: %s\n",
+            error != NULL ? error : "allocation failure");
         free(error);
-        shell->last_status = 258;
         return shell->last_status;
     }
     if (pipelines == NULL)
diff --git a/src/expand.c b/src/expand.c
index 3411bf5..a11b455 100644
--- a/src/expand.c
+++ b/src/expand.c
@@ -2,91 +2,118 @@
 
 #include <stdio.h>
 #include <stdlib.h>
-#include <string.h>
 
 #define LITERAL_MARK '\001'
 
-static char *append_status(char *out, int status) {
+static char *append_status(char *out, int status)
+{
     char buf[32];
+
     snprintf(buf, sizeof(buf), "%d", status);
     return sh_strjoin_free(out, buf);
 }
 
-static char *append_char(char *out, char c) {
+static char *append_char(char *out, char c)
+{
     char buf[2];
+
     buf[0] = c;
     buf[1] = '\0';
     return sh_strjoin_free(out, buf);
 }
 
-char *expand_word(t_shell *shell, const char *word) {
+char *expand_word(t_shell *shell, const char *word)
+{
     char    *out;
     size_t  i;
     size_t  start;
     char    *key;
 
     out = sh_strdup("");
+    if (out == NULL)
+        return NULL;
     i = 0;
-    while (word && word[i]) {
-        if (word[i] == LITERAL_MARK && word[i + 1]) {
+    while (word != NULL && word[i] != '\0') {
+        if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
             out = append_char(out, word[i + 1]);
             i += 2;
         } else if (word[i] == '$' && word[i + 1] == '?') {
             out = append_status(out, shell->last_status);
             i += 2;
-        } else if (word[i] == '$' && sh_is_name_start((unsigned char)word[i + 1])) {
+        } else if (word[i] == '$'
+            && sh_is_name_start((unsigned char)word[i + 1])) {
             start = i + 1;
             i = start + 1;
             while (sh_is_name_char((unsigned char)word[i]))
                 i++;
             key = sh_substr(word, start, i - start);
+            if (key == NULL) {
+                free(out);
+                return NULL;
+            }
             out = sh_strjoin_free(out, env_get(shell->env, key));
             free(key);
         } else {
             out = append_char(out, word[i]);
             i++;
         }
+        if (out == NULL)
+            return NULL;
     }
     return out;
 }
 
-static char *dequote_word(const char *word) {
+static char *dequote_word(const char *word)
+{
     char    *out;
     size_t  i;
 
     out = sh_strdup("");
+    if (out == NULL)
+        return NULL;
     i = 0;
-    while (word && word[i]) {
-        if (word[i] == LITERAL_MARK && word[i + 1]) {
+    while (word != NULL && word[i] != '\0') {
+        if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
             out = append_char(out, word[i + 1]);
             i += 2;
         } else {
             out = append_char(out, word[i]);
             i++;
         }
+        if (out == NULL)
+            return NULL;
     }
     return out;
 }
 
-int shell_dequote_word(const char *word, char **out, char **error) {
-    if (error)
+int shell_dequote_word(const char *word, char **out, char **error)
+{
+    if (error != NULL)
         *error = NULL;
-    if (!out) {
-        if (error)
+    if (out == NULL) {
+        if (error != NULL)
             *error = sh_strdup("dequote output is null");
         return 1;
     }
     *out = dequote_word(word);
+    if (*out == NULL) {
+        if (error != NULL)
+            *error = sh_strdup("allocation failure");
+        return 1;
+    }
     return 0;
 }
 
-static int expand_words(t_shell *shell, char ***words) {
+static int expand_words(t_shell *shell, char ***words)
+{
     size_t  i;
     char    *expanded;
 
     i = 0;
-    while (*words && (*words)[i]) {
+    while (*words != NULL && (*words)[i] != NULL) {
         expanded = expand_word(shell, (*words)[i]);
+        if (expanded == NULL)
+            return 1;
         free((*words)[i]);
         (*words)[i] = expanded;
         i++;
@@ -94,21 +121,25 @@ static int expand_words(t_shell *shell, char ***words) {
     return 0;
 }
 
-int expand_pipeline(t_shell *shell, t_pipeline *pipeline) {
+int expand_pipeline(t_shell *shell, t_pipeline *pipeline)
+{
     t_command   *cmd;
     t_redir     *redir;
     char        *expanded;
 
-    while (pipeline) {
+    while (pipeline != NULL) {
         cmd = pipeline->commands;
-        while (cmd) {
-            expand_words(shell, &cmd->argv);
+        while (cmd != NULL) {
+            if (expand_words(shell, &cmd->argv) != 0)
+                return 1;
             redir = cmd->redirs;
-            while (redir) {
+            while (redir != NULL) {
                 if (redir->type == REDIR_HEREDOC)
                     expanded = dequote_word(redir->target);
                 else
                     expanded = expand_word(shell, redir->target);
+                if (expanded == NULL)
+                    return 1;
                 free(redir->target);
                 redir->target = expanded;
                 redir = redir->next;
@@ -121,15 +152,20 @@ int expand_pipeline(t_shell *shell, t_pipeline *pipeline) {
 }
 
 int shell_expand_sequence(t_sequence *sequence, const t_env *env,
-        int last_status, char **error) {
+        int last_status, char **error)
+{
     t_shell shell;
+    int     result;
 
-    if (error)
+    if (error != NULL)
         *error = NULL;
-    if (!sequence)
+    if (sequence == NULL)
         return 0;
     shell.env = (t_env *)env;
     shell.last_status = last_status;
     shell.running = 1;
-    return expand_pipeline(&shell, sequence->pipelines);
+    result = expand_pipeline(&shell, sequence->pipelines);
+    if (result != 0 && error != NULL)
+        *error = sh_strdup("allocation failure");
+    return result;
 }
diff --git a/src/main.c b/src/main.c
index ab57407..04884bd 100644
--- a/src/main.c
+++ b/src/main.c
@@ -2,6 +2,10 @@
 
 #include "shell.h"
 
+#include <errno.h>
+#include <stdio.h>
+#include <string.h>
+
 static int normalize_status(int status)
 {
     return status & 0xff;
@@ -14,7 +18,13 @@ int main(int argc, char **argv, char **envp)
 
     (void)argc;
     (void)argv;
+    errno = 0;
     shell.env = env_from_environ(envp);
+    if (shell.env == NULL && envp != NULL && envp[0] != NULL
+        && errno == ENOMEM) {
+        fprintf(stderr, "small-shell: startup: %s\n", strerror(errno));
+        return 1;
+    }
     shell.last_status = 0;
     shell.running = 1;
 
diff --git a/src/parser.c b/src/parser.c
index 00efc78..fcf858f 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -1,102 +1,134 @@
 #include "shell.h"
 
 #include <stdlib.h>
-#include <string.h>
 
 static void free_commands(t_command *cmd);
 
-static void set_error(char **error, const char *message) {
-    if (error)
+static void set_error(char **error, const char *message)
+{
+    if (error != NULL && *error == NULL)
         *error = sh_strdup(message);
 }
 
-static t_command *new_command(void) {
-    return (t_command *)sh_xcalloc(1, sizeof(t_command));
+static t_command *new_command(void)
+{
+    return (t_command *)sh_calloc(1, sizeof(t_command));
 }
 
-static t_pipeline *new_pipeline(void) {
-    t_pipeline *pipeline = (t_pipeline *)sh_xcalloc(1, sizeof(t_pipeline));
-    pipeline->next_op = CONN_NONE;
+static t_pipeline *new_pipeline(void)
+{
+    t_pipeline *pipeline;
+
+    pipeline = (t_pipeline *)sh_calloc(1, sizeof(t_pipeline));
+    if (pipeline != NULL)
+        pipeline->next_op = CONN_NONE;
     return pipeline;
 }
 
-static size_t word_count(char **argv) {
-    size_t n = 0;
-    while (argv && argv[n])
+static size_t word_count(char **argv)
+{
+    size_t n;
+
+    n = 0;
+    while (argv != NULL && argv[n] != NULL)
         n++;
     return n;
 }
 
-static void add_arg(t_command *cmd, const char *text) {
+static int add_arg(t_command *cmd, const char *text)
+{
     size_t  n;
     char    **next;
+    char    *copy;
     size_t  i;
 
     n = word_count(cmd->argv);
-    next = (char **)sh_xcalloc(n + 2, sizeof(char *));
+    next = (char **)sh_calloc(n + 2, sizeof(char *));
+    if (next == NULL)
+        return 1;
+    copy = sh_strdup(text);
+    if (copy == NULL) {
+        free(next);
+        return 1;
+    }
     i = 0;
     while (i < n) {
         next[i] = cmd->argv[i];
         i++;
     }
-    next[n] = sh_strdup(text);
+    next[n] = copy;
     free(cmd->argv);
     cmd->argv = next;
     cmd->argc = n + 1;
+    return 0;
 }
 
-static void add_redir(t_command *cmd, t_redir_type type, const char *target,
-        int target_quoted) {
+static int add_redir(t_command *cmd, t_redir_type type, const char *target,
+        int target_quoted)
+{
     t_redir *node;
     t_redir *tail;
 
-    node = (t_redir *)sh_xcalloc(1, sizeof(t_redir));
-    node->type = type;
+    node = (t_redir *)sh_calloc(1, sizeof(t_redir));
+    if (node == NULL)
+        return 1;
     node->target = sh_strdup(target);
+    if (node->target == NULL) {
+        free(node);
+        return 1;
+    }
+    node->type = type;
     node->heredoc_quoted = (type == REDIR_HEREDOC && target_quoted);
-    if (!cmd->redirs) {
+    if (cmd->redirs == NULL) {
         cmd->redirs = node;
-        return;
+        return 0;
     }
     tail = cmd->redirs;
-    while (tail->next)
+    while (tail->next != NULL)
         tail = tail->next;
     tail->next = node;
+    return 0;
 }
 
-static int command_empty(t_command *cmd) {
-    return (!cmd || (cmd->argc == 0 && !cmd->redirs));
+static int command_empty(t_command *cmd)
+{
+    return (cmd == NULL || (cmd->argc == 0 && cmd->redirs == NULL));
 }
 
-static void append_command(t_pipeline *pipeline, t_command *cmd) {
+static void append_command(t_pipeline *pipeline, t_command *cmd)
+{
     t_command *tail;
 
-    if (!pipeline->commands) {
+    if (pipeline->commands == NULL) {
         pipeline->commands = cmd;
         pipeline->command_count++;
         return;
     }
     tail = pipeline->commands;
-    while (tail->next)
+    while (tail->next != NULL)
         tail = tail->next;
     tail->next = cmd;
     pipeline->command_count++;
 }
 
-static void append_pipeline(t_pipeline **head, t_pipeline **tail, t_pipeline *node) {
-    if (!*head)
+static void append_pipeline(t_pipeline **head, t_pipeline **tail,
+        t_pipeline *node)
+{
+    if (*head == NULL)
         *head = node;
     else
         (*tail)->next = node;
     *tail = node;
 }
 
-static int token_is_redir(t_token_type type) {
+static int token_is_redir(t_token_type type)
+{
     return (type == TOK_REDIR_IN || type == TOK_REDIR_OUT
         || type == TOK_REDIR_APPEND || type == TOK_HEREDOC);
 }
 
-static t_redir_type redir_type(t_token_type type) {
+static t_redir_type redir_type(t_token_type type)
+{
     if (type == TOK_REDIR_OUT)
         return REDIR_OUT;
     if (type == TOK_REDIR_APPEND)
@@ -106,7 +138,8 @@ static t_redir_type redir_type(t_token_type type) {
     return REDIR_IN;
 }
 
-static t_connector connector_type(t_token_type type) {
+static t_connector connector_type(t_token_type type)
+{
     if (type == TOK_AND)
         return CONN_AND;
     if (type == TOK_OR)
@@ -114,7 +147,18 @@ static t_connector connector_type(t_token_type type) {
     return CONN_SEQ;
 }
 
-t_pipeline *parse_tokens(t_token *tokens, char **error) {
+static t_pipeline *parse_failure(t_pipeline *head, t_pipeline *pipeline,
+        t_command *cmd, char **error, const char *message)
+{
+    set_error(error, message);
+    free_commands(cmd);
+    free_pipeline(pipeline);
+    free_pipeline(head);
+    return NULL;
+}
+
+t_pipeline *parse_tokens(t_token *tokens, char **error)
+{
     t_pipeline  *head;
     t_pipeline  *tail;
     t_pipeline  *pipeline;
@@ -130,75 +174,70 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
     cur = tokens;
     last_connector = TOK_WORD;
     after_pipe = 0;
-    if (error)
+    if (error != NULL)
         *error = NULL;
-    while (cur) {
+    if (pipeline == NULL || cmd == NULL)
+        return parse_failure(head, pipeline, cmd, error,
+            "allocation failure");
+    while (cur != NULL) {
         if (cur->type == TOK_WORD) {
-            add_arg(cmd, cur->text);
+            if (add_arg(cmd, cur->text) != 0)
+                return parse_failure(head, pipeline, cmd, error,
+                    "allocation failure");
             after_pipe = 0;
         } else if (token_is_redir(cur->type)) {
-            if (!cur->next || cur->next->type != TOK_WORD) {
-                set_error(error, "syntax error: redirection target missing");
-                free_commands(cmd);
-                free_pipeline(pipeline);
-                free_pipeline(head);
-                return NULL;
-            }
-            add_redir(cmd, redir_type(cur->type), cur->next->text,
-                cur->next->quoted);
+            if (cur->next == NULL || cur->next->type != TOK_WORD)
+                return parse_failure(head, pipeline, cmd, error,
+                    "syntax error: redirection target missing");
+            if (add_redir(cmd, redir_type(cur->type), cur->next->text,
+                    cur->next->quoted) != 0)
+                return parse_failure(head, pipeline, cmd, error,
+                    "allocation failure");
             cur = cur->next;
             after_pipe = 0;
         } else if (cur->type == TOK_PIPE) {
-            if (command_empty(cmd)) {
-                set_error(error, "syntax error: empty command before pipe");
-                free_commands(cmd);
-                free_pipeline(pipeline);
-                free_pipeline(head);
-                return NULL;
-            }
+            if (command_empty(cmd))
+                return parse_failure(head, pipeline, cmd, error,
+                    "syntax error: empty command before pipe");
             append_command(pipeline, cmd);
             cmd = new_command();
+            if (cmd == NULL)
+                return parse_failure(head, pipeline, cmd, error,
+                    "allocation failure");
             after_pipe = 1;
         } else {
-            if (after_pipe) {
-                set_error(error, "syntax error: expected command after pipe");
-                free_commands(cmd);
-                free_pipeline(pipeline);
-                free_pipeline(head);
-                return NULL;
-            }
-            if (command_empty(cmd) && !pipeline->commands) {
-                set_error(error, "syntax error: empty command before connector");
-                free_commands(cmd);
-                free_pipeline(pipeline);
-                free_pipeline(head);
-                return NULL;
-            }
+            if (after_pipe)
+                return parse_failure(head, pipeline, cmd, error,
+                    "syntax error: expected command after pipe");
+            if (command_empty(cmd) && pipeline->commands == NULL)
+                return parse_failure(head, pipeline, cmd, error,
+                    "syntax error: empty command before connector");
             if (!command_empty(cmd))
                 append_command(pipeline, cmd);
             else
                 free_commands(cmd);
-            cmd = new_command();
+            cmd = NULL;
             pipeline->next_op = connector_type(cur->type);
             append_pipeline(&head, &tail, pipeline);
             last_connector = cur->type;
             pipeline = new_pipeline();
+            cmd = new_command();
+            if (pipeline == NULL || cmd == NULL)
+                return parse_failure(head, pipeline, cmd, error,
+                    "allocation failure");
             after_pipe = 0;
         }
         cur = cur->next;
     }
-    if (after_pipe) {
-        set_error(error, "syntax error: expected command after pipe");
-        free_commands(cmd);
-        free_pipeline(pipeline);
-        free_pipeline(head);
-        return NULL;
-    }
+    if (after_pipe)
+        return parse_failure(head, pipeline, cmd, error,
+            "syntax error: expected command after pipe");
     if (!command_empty(cmd))
         append_command(pipeline, cmd);
     else
         free_commands(cmd);
-    if (!pipeline->commands) {
+    cmd = NULL;
+    if (pipeline->commands == NULL) {
         free(pipeline);
         if (last_connector == TOK_AND || last_connector == TOK_OR) {
             set_error(error,
@@ -206,7 +245,7 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
             free_pipeline(head);
             return NULL;
         }
-        if (last_connector == TOK_SEQ && tail)
+        if (last_connector == TOK_SEQ && tail != NULL)
             tail->next_op = CONN_NONE;
         return head;
     }
@@ -214,10 +253,11 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
     return head;
 }
 
-static void free_redirs(t_redir *redir) {
+static void free_redirs(t_redir *redir)
+{
     t_redir *next;
 
-    while (redir) {
+    while (redir != NULL) {
         next = redir->next;
         free(redir->target);
         free(redir);
@@ -225,10 +265,11 @@ static void free_redirs(t_redir *redir) {
     }
 }
 
-static void free_commands(t_command *cmd) {
+static void free_commands(t_command *cmd)
+{
     t_command *next;
 
-    while (cmd) {
+    while (cmd != NULL) {
         next = cmd->next;
         sh_free_words(cmd->argv);
         free_redirs(cmd->redirs);
@@ -237,10 +278,11 @@ static void free_commands(t_command *cmd) {
     }
 }
 
-void free_pipeline(t_pipeline *pipeline) {
+void free_pipeline(t_pipeline *pipeline)
+{
     t_pipeline *next;
 
-    while (pipeline) {
+    while (pipeline != NULL) {
         next = pipeline->next;
         free_commands(pipeline->commands);
         free(pipeline);
@@ -248,53 +290,57 @@ void free_pipeline(t_pipeline *pipeline) {
     }
 }
 
-static size_t count_pipelines(t_pipeline *pipeline) {
+static size_t count_pipelines(t_pipeline *pipeline)
+{
     size_t count;
 
     count = 0;
-    while (pipeline) {
+    while (pipeline != NULL) {
         count++;
         pipeline = pipeline->next;
     }
     return count;
 }
 
-void shell_sequence_init(t_sequence *sequence) {
-    if (!sequence)
+void shell_sequence_init(t_sequence *sequence)
+{
+    if (sequence == NULL)
         return;
     sequence->pipelines = NULL;
     sequence->pipeline_count = 0;
 }
 
-void shell_sequence_free(t_sequence *sequence) {
-    if (!sequence)
+void shell_sequence_free(t_sequence *sequence)
+{
+    if (sequence == NULL)
         return;
     free_pipeline(sequence->pipelines);
     sequence->pipelines = NULL;
     sequence->pipeline_count = 0;
 }
 
-int shell_parse_line(const char *line, t_sequence *sequence, char **error) {
+int shell_parse_line(const char *line, t_sequence *sequence, char **error)
+{
     t_token  *tokens;
     char     *internal_error;
     char     **error_slot;
 
     internal_error = NULL;
-    error_slot = error ? error : &internal_error;
-    if (!sequence) {
-        *error_slot = sh_strdup("parse output is null");
+    error_slot = error != NULL ? error : &internal_error;
+    if (sequence == NULL) {
+        set_error(error_slot, "parse output is null");
         free(internal_error);
         return 1;
     }
     shell_sequence_init(sequence);
     tokens = tokenize_line(line, error_slot);
-    if (*error_slot) {
+    if (*error_slot != NULL) {
         free(internal_error);
         return 1;
     }
     sequence->pipelines = parse_tokens(tokens, error_slot);
     free_tokens(tokens);
-    if (*error_slot) {
+    if (*error_slot != NULL) {
         shell_sequence_free(sequence);
         free(internal_error);
         return 1;
@@ -305,23 +351,24 @@ int shell_parse_line(const char *line, t_sequence *sequence, char **error) {
 }
 
 int shell_execute_sequence(const t_sequence *sequence, t_env *env,
-        int *last_status, const t_executor_hooks *hooks, void *ctx) {
+        int *last_status, const t_executor_hooks *hooks, void *ctx)
+{
     const t_pipeline *pipeline;
-    t_connector gate;
-    int status;
-    int should_run;
+    t_connector      gate;
+    int              status;
+    int              should_run;
 
-    if (!hooks || !hooks->run_pipeline) {
-        if (hooks && hooks->on_error)
+    if (hooks == NULL || hooks->run_pipeline == NULL) {
+        if (hooks != NULL && hooks->on_error != NULL)
             hooks->on_error("missing executor pipeline hook", ctx);
-        if (last_status)
+        if (last_status != NULL)
             *last_status = 1;
         return 1;
     }
-    status = last_status ? *last_status : 0;
+    status = last_status != NULL ? *last_status : 0;
     gate = CONN_NONE;
-    pipeline = sequence ? sequence->pipelines : NULL;
-    while (pipeline) {
+    pipeline = sequence != NULL ? sequence->pipelines : NULL;
+    while (pipeline != NULL) {
         should_run = 1;
         if (gate == CONN_AND && status != 0)
             should_run = 0;
@@ -332,7 +379,7 @@ int shell_execute_sequence(const t_sequence *sequence, t_env *env,
         gate = pipeline->next_op;
         pipeline = pipeline->next;
     }
-    if (last_status)
+    if (last_status != NULL)
         *last_status = status;
     return status;
 }
diff --git a/src/token.c b/src/token.c
index 5b657d1..5465b58 100644
--- a/src/token.c
+++ b/src/token.c
@@ -5,34 +5,65 @@
 
 #define LITERAL_MARK '\001'
 
+static void set_error(char **error, const char *message)
+{
+    if (error != NULL && *error == NULL)
+        *error = sh_strdup(message);
+}
+
 static t_token *new_token(t_token_type type, char *text, size_t start,
-        int quoted) {
-    t_token *token = (t_token *)sh_xcalloc(1, sizeof(t_token));
+        int quoted)
+{
+    t_token *token;
+
+    token = (t_token *)sh_calloc(1, sizeof(t_token));
+    if (token == NULL) {
+        free(text);
+        return NULL;
+    }
     token->type = type;
-    token->text = text ? text : sh_strdup("");
+    token->text = text;
     token->start = start;
     token->quoted = quoted;
     return token;
 }
 
-static void push_token(t_token **head, t_token **tail, t_token *node) {
-    if (!*head)
+static int push_token(t_token **head, t_token **tail, t_token *node)
+{
+    if (node == NULL)
+        return 1;
+    if (*head == NULL)
         *head = node;
     else
         (*tail)->next = node;
     *tail = node;
+    return 0;
 }
 
-static int is_operator_char(char c) {
+static int push_operator(t_token **head, t_token **tail, t_token_type type,
+        const char *text, size_t start)
+{
+    char *copy;
+
+    copy = sh_strdup(text);
+    if (copy == NULL)
+        return 1;
+    return push_token(head, tail, new_token(type, copy, start, 0));
+}
+
+static int is_operator_char(char c)
+{
     return (c == '|' || c == '<' || c == '>' || c == '&' || c == ';');
 }
 
-static int is_shell_space(char c) {
+static int is_shell_space(char c)
+{
     return (c == ' ' || c == '\t' || c == '\n'
         || c == '\r' || c == '\v' || c == '\f');
 }
 
-static char *append_char(char *word, char c) {
+static char *append_char(char *word, char c)
+{
     char buf[2];
 
     buf[0] = c;
@@ -40,115 +71,133 @@ static char *append_char(char *word, char c) {
     return sh_strjoin_free(word, buf);
 }
 
-static char *append_literal(char *word, char c) {
+static char *append_literal(char *word, char c)
+{
     word = append_char(word, LITERAL_MARK);
+    if (word == NULL)
+        return NULL;
     return append_char(word, c);
 }
 
 static char *read_word(const char *line, size_t *i, char **error,
-        int *quoted) {
+        int *quoted)
+{
     char    *word;
     char    quote;
 
     word = sh_strdup("");
+    if (word == NULL)
+        return NULL;
     *quoted = 0;
-    while (line[*i] && !is_shell_space(line[*i])
+    while (line[*i] != '\0' && !is_shell_space(line[*i])
         && !is_operator_char(line[*i])) {
         if (line[*i] == '\'' || line[*i] == '"') {
             quote = line[*i];
             *quoted = 1;
             (*i)++;
-            while (line[*i] && line[*i] != quote) {
+            while (line[*i] != '\0' && line[*i] != quote) {
                 if (quote == '\'')
                     word = append_literal(word, line[*i]);
                 else
                     word = append_char(word, line[*i]);
+                if (word == NULL)
+                    return NULL;
                 (*i)++;
             }
-            if (!line[*i]) {
+            if (line[*i] == '\0') {
                 free(word);
-                *error = sh_strdup("syntax error: unclosed quote");
+                set_error(error, "syntax error: unclosed quote");
                 return NULL;
             }
             (*i)++;
         } else {
             word = append_char(word, line[*i]);
+            if (word == NULL)
+                return NULL;
             (*i)++;
         }
     }
     return word;
 }
 
-t_token *tokenize_line(const char *line, char **error) {
-    t_token *head;
-    t_token *tail;
-    size_t  i;
+static int push_word(const char *line, size_t *i, char **error,
+        t_token **head, t_token **tail)
+{
     size_t  start;
     char    *word;
     int     quoted;
 
+    start = *i;
+    word = read_word(line, i, error, &quoted);
+    if (word == NULL)
+        return 1;
+    return push_token(head, tail,
+        new_token(TOK_WORD, word, start, quoted));
+}
+
+t_token *tokenize_line(const char *line, char **error)
+{
+    t_token *head;
+    t_token *tail;
+    size_t  i;
+    int     failed;
+
     head = NULL;
     tail = NULL;
     i = 0;
-    if (error)
+    failed = 0;
+    if (error != NULL)
         *error = NULL;
-    while (line && line[i]) {
+    while (line != NULL && line[i] != '\0' && !failed) {
         while (is_shell_space(line[i]))
             i++;
-        if (!line[i])
+        if (line[i] == '\0')
             break;
-        if (line[i] == '|') {
-            if (line[i + 1] == '|') {
-                push_token(&head, &tail, new_token(TOK_OR, sh_strdup("||"), i, 0));
-                i += 2;
-            } else {
-                push_token(&head, &tail, new_token(TOK_PIPE, sh_strdup("|"), i, 0));
-                i++;
-            }
+        if (line[i] == '|' && line[i + 1] == '|') {
+            failed = push_operator(&head, &tail, TOK_OR, "||", i);
+            i += 2;
+        } else if (line[i] == '|') {
+            failed = push_operator(&head, &tail, TOK_PIPE, "|", i);
+            i++;
         } else if (line[i] == '&' && line[i + 1] == '&') {
-            push_token(&head, &tail, new_token(TOK_AND, sh_strdup("&&"), i, 0));
+            failed = push_operator(&head, &tail, TOK_AND, "&&", i);
             i += 2;
         } else if (line[i] == '&') {
-            if (error)
-                *error = sh_strdup("syntax error: unsupported operator '&'");
+            set_error(error, "syntax error: unsupported operator '&'");
             free_tokens(head);
             return NULL;
         } else if (line[i] == ';') {
-            push_token(&head, &tail, new_token(TOK_SEQ, sh_strdup(";"), i, 0));
+            failed = push_operator(&head, &tail, TOK_SEQ, ";", i);
             i++;
+        } else if (line[i] == '<' && line[i + 1] == '<') {
+            failed = push_operator(&head, &tail, TOK_HEREDOC, "<<", i);
+            i += 2;
         } else if (line[i] == '<') {
-            if (line[i + 1] == '<') {
-                push_token(&head, &tail, new_token(TOK_HEREDOC, sh_strdup("<<"), i, 0));
-                i += 2;
-            } else {
-                push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i, 0));
-                i++;
-            }
+            failed = push_operator(&head, &tail, TOK_REDIR_IN, "<", i);
+            i++;
+        } else if (line[i] == '>' && line[i + 1] == '>') {
+            failed = push_operator(&head, &tail, TOK_REDIR_APPEND, ">>", i);
+            i += 2;
         } else if (line[i] == '>') {
-            if (line[i + 1] == '>') {
-                push_token(&head, &tail, new_token(TOK_REDIR_APPEND, sh_strdup(">>"), i, 0));
-                i += 2;
-            } else {
-                push_token(&head, &tail, new_token(TOK_REDIR_OUT, sh_strdup(">"), i, 0));
-                i++;
-            }
+            failed = push_operator(&head, &tail, TOK_REDIR_OUT, ">", i);
+            i++;
         } else {
-            start = i;
-            word = read_word(line, &i, error, &quoted);
-            if (!word) {
-                free_tokens(head);
-                return NULL;
-            }
-            push_token(&head, &tail, new_token(TOK_WORD, word, start, quoted));
+            failed = push_word(line, &i, error, &head, &tail);
         }
     }
+    if (failed) {
+        set_error(error, "allocation failure");
+        free_tokens(head);
+        return NULL;
+    }
     return head;
 }
 
-void free_tokens(t_token *tokens) {
+void free_tokens(t_token *tokens)
+{
     t_token *next;
 
-    while (tokens) {
+    while (tokens != NULL) {
         next = tokens->next;
         free(tokens->text);
         free(tokens);
diff --git a/src/utils.c b/src/utils.c
index 384eb15..ff65463 100644
--- a/src/utils.c
+++ b/src/utils.c
@@ -2,98 +2,111 @@
 #include "runtime.h"
 
 #include <ctype.h>
-#include <stdio.h>
+#include <stdint.h>
 #include <stdlib.h>
 #include <string.h>
 
-void *sh_xcalloc(size_t count, size_t size) {
-    void *ptr = shell_calloc(count, size);
-    if (!ptr) {
-        perror("small-shell: calloc");
-        exit(1);
-    }
-    return ptr;
+void *sh_calloc(size_t count, size_t size)
+{
+    return shell_calloc(count, size);
 }
 
-char *sh_strdup(const char *s) {
-    char *copy;
-    size_t len;
+char *sh_strdup(const char *s)
+{
+    char    *copy;
+    size_t  len;
 
-    if (!s)
+    if (s == NULL)
         s = "";
     len = strlen(s);
+    if (len == SIZE_MAX)
+        return NULL;
     copy = (char *)shell_malloc(len + 1);
-    if (!copy) {
-        perror("small-shell: malloc");
-        exit(1);
-    }
-    memcpy(copy, s, len);
-    copy[len] = '\0';
+    if (copy == NULL)
+        return NULL;
+    memcpy(copy, s, len + 1);
     return copy;
 }
 
-char *sh_substr(const char *s, size_t start, size_t len) {
-    char *out;
-    size_t n;
+char *sh_substr(const char *s, size_t start, size_t len)
+{
+    char    *out;
+    size_t  n;
 
-    out = (char *)sh_xcalloc(len + 1, sizeof(char));
+    if (s == NULL || len == SIZE_MAX)
+        return NULL;
+    out = (char *)shell_calloc(len + 1, sizeof(char));
+    if (out == NULL)
+        return NULL;
     n = 0;
-    while (n < len && s[start + n]) {
+    while (n < len && s[start + n] != '\0') {
         out[n] = s[start + n];
         n++;
     }
-    out[n] = '\0';
     return out;
 }
 
-char *sh_strjoin_free(char *left, const char *right) {
+char *sh_strjoin_free(char *left, const char *right)
+{
     size_t  a;
     size_t  b;
     char    *out;
 
-    if (!left)
-        left = sh_strdup("");
-    if (!right)
+    if (left == NULL)
+        return NULL;
+    if (right == NULL)
         right = "";
     a = strlen(left);
     b = strlen(right);
-    out = (char *)sh_xcalloc(a + b + 1, sizeof(char));
+    if (b > SIZE_MAX - a - 1) {
+        free(left);
+        return NULL;
+    }
+    out = (char *)shell_malloc(a + b + 1);
+    if (out == NULL) {
+        free(left);
+        return NULL;
+    }
     memcpy(out, left, a);
-    memcpy(out + a, right, b);
+    memcpy(out + a, right, b + 1);
     free(left);
     return out;
 }
 
-void sh_free_words(char **words) {
+void sh_free_words(char **words)
+{
     size_t i;
 
-    if (!words)
+    if (words == NULL)
         return;
     i = 0;
-    while (words[i]) {
+    while (words[i] != NULL) {
         free(words[i]);
         i++;
     }
     free(words);
 }
 
-char *shell_strndup(const char *s, size_t len) {
-    char *out;
-    size_t i;
+char *shell_strndup(const char *s, size_t len)
+{
+    char    *out;
+    size_t  i;
 
-    if (!s)
-        s = "";
-    out = (char *)sh_xcalloc(len + 1, sizeof(char));
+    if (s == NULL || len == SIZE_MAX)
+        return NULL;
+    out = (char *)shell_calloc(len + 1, sizeof(char));
+    if (out == NULL)
+        return NULL;
     i = 0;
-    while (i < len && s[i]) {
+    while (i < len && s[i] != '\0') {
         out[i] = s[i];
         i++;
     }
-    out[i] = '\0';
     return out;
 }
 
-char *shell_itoa_status(int status) {
+char *shell_itoa_status(int status)
+{
     char    buf[32];
     long    value;
     size_t  len;
@@ -108,8 +121,8 @@ char *shell_itoa_status(int status) {
     if (value == 0)
         buf[len++] = '0';
     else {
-        char digits[24];
-        size_t count;
+        char    digits[24];
+        size_t  count;
 
         count = 0;
         while (value > 0) {
@@ -119,22 +132,25 @@ char *shell_itoa_status(int status) {
         while (count > 0)
             buf[len++] = digits[--count];
     }
-    buf[len] = '\0';
     out = (char *)shell_malloc(len + 1);
-    if (!out)
+    if (out == NULL)
         return NULL;
-    memcpy(out, buf, len + 1);
+    memcpy(out, buf, len);
+    out[len] = '\0';
     return out;
 }
 
-void shell_strv_free(char **words) {
+void shell_strv_free(char **words)
+{
     sh_free_words(words);
 }
 
-int sh_is_name_start(int c) {
+int sh_is_name_start(int c)
+{
     return (isalpha((unsigned char)c) || c == '_');
 }
 
-int sh_is_name_char(int c) {
+int sh_is_name_char(int c)
+{
     return (isalnum((unsigned char)c) || c == '_');
 }
