# 인용 의미를 보존하는 단어 확장과 지연 평가

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


## `feat(lexer): 인용 단어와 토큰 수명 관리`

diff --git a/Makefile b/Makefile
index 14ac270..774cceb 100644
--- a/Makefile
+++ b/Makefile
@@ -8,6 +8,7 @@ TARGET := small-shell
 SRCS := \
 	src/main.c \
 	src/input.c \
+	src/token.c \
 	src/env.c \
 	src/utils.c
 OBJS := $(SRCS:.c=.o)
diff --git a/include/shell.h b/include/shell.h
index 205b1de..23884ae 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -3,6 +3,17 @@
 
 # include <stddef.h>
 
+typedef enum e_token_type {
+    TOK_WORD
+}   t_token_type;
+
+typedef struct s_token {
+    t_token_type    type;
+    char            *text;
+    size_t          start;
+    struct s_token  *next;
+}   t_token;
+
 typedef struct s_env {
     char            *key;
     char            *value;
@@ -28,6 +39,8 @@ char    *shell_itoa_status(int status);
 void    shell_strv_free(char **words);
 int     sh_is_name_char(int c);
 int     sh_is_name_start(int c);
+t_token *tokenize_line(const char *line, char **error);
+void    free_tokens(t_token *tokens);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
 const char  *env_get(t_env *env, const char *key);
diff --git a/src/token.c b/src/token.c
new file mode 100644
index 0000000..7756646
--- /dev/null
+++ b/src/token.c
@@ -0,0 +1,119 @@
+#include "shell.h"
+
+#include <stdlib.h>
+
+#define LITERAL_MARK '\001'
+
+static t_token *new_token(t_token_type type, char *text, size_t start) {
+    t_token *token = (t_token *)sh_xcalloc(1, sizeof(t_token));
+    token->type = type;
+    token->text = text ? text : sh_strdup("");
+    token->start = start;
+    return token;
+}
+
+static void push_token(t_token **head, t_token **tail, t_token *node) {
+    if (!*head)
+        *head = node;
+    else
+        (*tail)->next = node;
+    *tail = node;
+}
+
+static int is_operator_char(char c) {
+    return (c == '|' || c == '<' || c == '>' || c == '&' || c == ';');
+}
+
+static int is_shell_space(char c) {
+    return (c == ' ' || c == '\t' || c == '\n'
+        || c == '\r' || c == '\v' || c == '\f');
+}
+
+static char *append_char(char *word, char c) {
+    char buf[2];
+
+    buf[0] = c;
+    buf[1] = '\0';
+    return sh_strjoin_free(word, buf);
+}
+
+static char *append_literal(char *word, char c) {
+    word = append_char(word, LITERAL_MARK);
+    return append_char(word, c);
+}
+
+static char *read_word(const char *line, size_t *i, char **error) {
+    char    *word;
+    char    quote;
+
+    word = sh_strdup("");
+    while (line[*i] && !is_shell_space(line[*i])
+        && !is_operator_char(line[*i])) {
+        if (line[*i] == '\'' || line[*i] == '"') {
+            quote = line[*i];
+            (*i)++;
+            while (line[*i] && line[*i] != quote) {
+                if (quote == '\'')
+                    word = append_literal(word, line[*i]);
+                else
+                    word = append_char(word, line[*i]);
+                (*i)++;
+            }
+            if (!line[*i]) {
+                free(word);
+                *error = sh_strdup("syntax error: unclosed quote");
+                return NULL;
+            }
+            (*i)++;
+        } else {
+            word = append_char(word, line[*i]);
+            (*i)++;
+        }
+    }
+    return word;
+}
+
+t_token *tokenize_line(const char *line, char **error) {
+    t_token *head;
+    t_token *tail;
+    size_t  i;
+    size_t  start;
+    char    *word;
+
+    head = NULL;
+    tail = NULL;
+    i = 0;
+    if (error)
+        *error = NULL;
+    while (line && line[i]) {
+        while (is_shell_space(line[i]))
+            i++;
+        if (!line[i])
+            break;
+        if (is_operator_char(line[i])) {
+            if (error)
+                *error = sh_strdup("syntax error: unsupported operator");
+            free_tokens(head);
+            return NULL;
+        }
+        start = i;
+        word = read_word(line, &i, error);
+        if (!word) {
+            free_tokens(head);
+            return NULL;
+        }
+        push_token(&head, &tail, new_token(TOK_WORD, word, start));
+    }
+    return head;
+}
+
+void free_tokens(t_token *tokens) {
+    t_token *next;
+
+    while (tokens) {
+        next = tokens->next;
+        free(tokens->text);
+        free(tokens);
+        tokens = next;
+    }
+}


## `feat(expand): 인용 표식 제거 경로 제공`

diff --git a/Makefile b/Makefile
index fd1ded5..49fbba9 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,7 @@ SRCS := \
 	src/input.c \
 	src/token.c \
 	src/parser.c \
+	src/expand.c \
 	src/env.c \
 	src/utils.c
 OBJS := $(SRCS:.c=.o)
diff --git a/include/shell.h b/include/shell.h
index 3dc162a..5fdf19e 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -106,6 +106,7 @@ void    shell_sequence_free(t_sequence *sequence);
 int     shell_parse_line(const char *line, t_sequence *sequence, char **error);
 int     shell_execute_sequence(const t_sequence *sequence, t_env *env,
             int *last_status, const t_executor_hooks *hooks, void *ctx);
+int     shell_dequote_word(const char *word, char **out, char **error);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
 const char  *env_get(t_env *env, const char *key);
diff --git a/src/expand.c b/src/expand.c
new file mode 100644
index 0000000..f94f55a
--- /dev/null
+++ b/src/expand.c
@@ -0,0 +1,44 @@
+#include "shell.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+#include <string.h>
+
+#define LITERAL_MARK '\001'
+
+static char *append_char(char *out, char c) {
+    char buf[2];
+    buf[0] = c;
+    buf[1] = '\0';
+    return sh_strjoin_free(out, buf);
+}
+
+static char *dequote_word(const char *word) {
+    char    *out;
+    size_t  i;
+
+    out = sh_strdup("");
+    i = 0;
+    while (word && word[i]) {
+        if (word[i] == LITERAL_MARK && word[i + 1]) {
+            out = append_char(out, word[i + 1]);
+            i += 2;
+        } else {
+            out = append_char(out, word[i]);
+            i++;
+        }
+    }
+    return out;
+}
+
+int shell_dequote_word(const char *word, char **out, char **error) {
+    if (error)
+        *error = NULL;
+    if (!out) {
+        if (error)
+            *error = sh_strdup("dequote output is null");
+        return 1;
+    }
+    *out = dequote_word(word);
+    return 0;
+}


## `feat(expand): 환경과 종료 상태 단어 확장`

diff --git a/include/shell.h b/include/shell.h
index 5fdf19e..9a1641c 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -106,6 +106,7 @@ void    shell_sequence_free(t_sequence *sequence);
 int     shell_parse_line(const char *line, t_sequence *sequence, char **error);
 int     shell_execute_sequence(const t_sequence *sequence, t_env *env,
             int *last_status, const t_executor_hooks *hooks, void *ctx);
+char    *expand_word(t_shell *shell, const char *word);
 int     shell_dequote_word(const char *word, char **out, char **error);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
diff --git a/src/expand.c b/src/expand.c
index f94f55a..01e1f9d 100644
--- a/src/expand.c
+++ b/src/expand.c
@@ -6,6 +6,12 @@
 
 #define LITERAL_MARK '\001'
 
+static char *append_status(char *out, int status) {
+    char buf[32];
+    snprintf(buf, sizeof(buf), "%d", status);
+    return sh_strjoin_free(out, buf);
+}
+
 static char *append_char(char *out, char c) {
     char buf[2];
     buf[0] = c;
@@ -13,6 +19,37 @@ static char *append_char(char *out, char c) {
     return sh_strjoin_free(out, buf);
 }
 
+char *expand_word(t_shell *shell, const char *word) {
+    char    *out;
+    size_t  i;
+    size_t  start;
+    char    *key;
+
+    out = sh_strdup("");
+    i = 0;
+    while (word && word[i]) {
+        if (word[i] == LITERAL_MARK && word[i + 1]) {
+            out = append_char(out, word[i + 1]);
+            i += 2;
+        } else if (word[i] == '$' && word[i + 1] == '?') {
+            out = append_status(out, shell->last_status);
+            i += 2;
+        } else if (word[i] == '$' && sh_is_name_start((unsigned char)word[i + 1])) {
+            start = i + 1;
+            i = start + 1;
+            while (sh_is_name_char((unsigned char)word[i]))
+                i++;
+            key = sh_substr(word, start, i - start);
+            out = sh_strjoin_free(out, env_get(shell->env, key));
+            free(key);
+        } else {
+            out = append_char(out, word[i]);
+            i++;
+        }
+    }
+    return out;
+}
+
 static char *dequote_word(const char *word) {
     char    *out;
     size_t  i;


## `feat(expand): argv와 리다이렉션 확장 연결`

diff --git a/include/shell.h b/include/shell.h
index 9a1641c..23c26da 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -107,7 +107,10 @@ int     shell_parse_line(const char *line, t_sequence *sequence, char **error);
 int     shell_execute_sequence(const t_sequence *sequence, t_env *env,
             int *last_status, const t_executor_hooks *hooks, void *ctx);
 char    *expand_word(t_shell *shell, const char *word);
+int     expand_pipeline(t_shell *shell, t_pipeline *pipeline);
 int     shell_dequote_word(const char *word, char **out, char **error);
+int     shell_expand_sequence(t_sequence *sequence, const t_env *env,
+            int last_status, char **error);
 t_env   *env_from_environ(char **envp);
 void    env_free(t_env *env);
 const char  *env_get(t_env *env, const char *key);
diff --git a/src/expand.c b/src/expand.c
index 01e1f9d..d20b330 100644
--- a/src/expand.c
+++ b/src/expand.c
@@ -79,3 +79,54 @@ int shell_dequote_word(const char *word, char **out, char **error) {
     *out = dequote_word(word);
     return 0;
 }
+
+static int expand_words(t_shell *shell, char ***words) {
+    size_t  i;
+    char    *expanded;
+
+    i = 0;
+    while (*words && (*words)[i]) {
+        expanded = expand_word(shell, (*words)[i]);
+        free((*words)[i]);
+        (*words)[i] = expanded;
+        i++;
+    }
+    return 0;
+}
+
+int expand_pipeline(t_shell *shell, t_pipeline *pipeline) {
+    t_command   *cmd;
+    t_redir     *redir;
+    char        *expanded;
+
+    while (pipeline) {
+        cmd = pipeline->commands;
+        while (cmd) {
+            expand_words(shell, &cmd->argv);
+            redir = cmd->redirs;
+            while (redir) {
+                expanded = expand_word(shell, redir->target);
+                free(redir->target);
+                redir->target = expanded;
+                redir = redir->next;
+            }
+            cmd = cmd->next;
+        }
+        pipeline = pipeline->next;
+    }
+    return 0;
+}
+
+int shell_expand_sequence(t_sequence *sequence, const t_env *env,
+        int last_status, char **error) {
+    t_shell shell;
+
+    if (error)
+        *error = NULL;
+    if (!sequence)
+        return 0;
+    shell.env = (t_env *)env;
+    shell.last_status = last_status;
+    shell.running = 1;
+    return expand_pipeline(&shell, sequence->pipelines);
+}


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


