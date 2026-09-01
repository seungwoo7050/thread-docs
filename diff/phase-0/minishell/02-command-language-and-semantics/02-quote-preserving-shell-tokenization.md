# 인용 상태를 보존하는 셸 토큰화

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


## `feat(lexer): 셸 연산자를 토큰으로 구분`

diff --git a/include/shell.h b/include/shell.h
index 23884ae..e94a8cb 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -4,7 +4,14 @@
 # include <stddef.h>
 
 typedef enum e_token_type {
-    TOK_WORD
+    TOK_WORD,
+    TOK_PIPE,
+    TOK_REDIR_IN,
+    TOK_REDIR_OUT,
+    TOK_REDIR_APPEND,
+    TOK_AND,
+    TOK_OR,
+    TOK_SEQ
 }   t_token_type;
 
 typedef struct s_token {
diff --git a/src/token.c b/src/token.c
index 7756646..3f1ef2a 100644
--- a/src/token.c
+++ b/src/token.c
@@ -1,6 +1,7 @@
 #include "shell.h"
 
 #include <stdlib.h>
+#include <string.h>
 
 #define LITERAL_MARK '\001'
 
@@ -90,19 +91,51 @@ t_token *tokenize_line(const char *line, char **error) {
             i++;
         if (!line[i])
             break;
-        if (is_operator_char(line[i])) {
+        if (line[i] == '|') {
+            if (line[i + 1] == '|') {
+                push_token(&head, &tail, new_token(TOK_OR, sh_strdup("||"), i));
+                i += 2;
+            } else {
+                push_token(&head, &tail, new_token(TOK_PIPE, sh_strdup("|"), i));
+                i++;
+            }
+        } else if (line[i] == '&' && line[i + 1] == '&') {
+            push_token(&head, &tail, new_token(TOK_AND, sh_strdup("&&"), i));
+            i += 2;
+        } else if (line[i] == '&') {
             if (error)
-                *error = sh_strdup("syntax error: unsupported operator");
-            free_tokens(head);
-            return NULL;
-        }
-        start = i;
-        word = read_word(line, &i, error);
-        if (!word) {
+                *error = sh_strdup("syntax error: unsupported operator '&'");
             free_tokens(head);
             return NULL;
+        } else if (line[i] == ';') {
+            push_token(&head, &tail, new_token(TOK_SEQ, sh_strdup(";"), i));
+            i++;
+        } else if (line[i] == '<') {
+            if (line[i + 1] == '<') {
+                if (error)
+                    *error = sh_strdup("syntax error: unsupported operator '<<'");
+                free_tokens(head);
+                return NULL;
+            }
+            push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i));
+            i++;
+        } else if (line[i] == '>') {
+            if (line[i + 1] == '>') {
+                push_token(&head, &tail, new_token(TOK_REDIR_APPEND, sh_strdup(">>"), i));
+                i += 2;
+            } else {
+                push_token(&head, &tail, new_token(TOK_REDIR_OUT, sh_strdup(">"), i));
+                i++;
+            }
+        } else {
+            start = i;
+            word = read_word(line, &i, error);
+            if (!word) {
+                free_tokens(head);
+                return NULL;
+            }
+            push_token(&head, &tail, new_token(TOK_WORD, word, start));
         }
-        push_token(&head, &tail, new_token(TOK_WORD, word, start));
     }
     return head;
 }


## `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존`

diff --git a/include/shell.h b/include/shell.h
index d127c43..e3c1919 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -20,6 +20,7 @@ typedef struct s_token {
     t_token_type    type;
     char            *text;
     size_t          start;
+    int             quoted;
     struct s_token  *next;
 }   t_token;
 
@@ -38,6 +39,7 @@ typedef enum e_redir_type {
 typedef struct s_redir {
     t_redir_type    type;
     char            *target;
+    int             heredoc_quoted;
     struct s_redir  *next;
 }   t_redir;
 
diff --git a/src/heredoc.c b/src/heredoc.c
index d5b3d0b..2502529 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -138,19 +138,6 @@ static int expand_heredoc_body_line(t_shell *shell, const char *line,
     return 0;
 }
 
-static int word_has_literal_mark(const char *word)
-{
-    size_t i;
-
-    i = 0;
-    while (word != NULL && word[i] != '\0') {
-        if (word[i] == LITERAL_MARK)
-            return 1;
-        i++;
-    }
-    return 0;
-}
-
 static char *dequote_runtime_word(const char *word)
 {
     struct strbuf   out;
@@ -237,7 +224,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     int             quoted;
     int             interactive;
 
-    quoted = word_has_literal_mark(redir->target);
+    quoted = redir->heredoc_quoted;
     delimiter = dequote_runtime_word(redir->target);
     if (delimiter == NULL)
         return 1;
diff --git a/src/parser.c b/src/parser.c
index 3e73b51..81b6bad 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -45,13 +45,15 @@ static void add_arg(t_command *cmd, const char *text) {
     cmd->argc = n + 1;
 }
 
-static void add_redir(t_command *cmd, t_redir_type type, const char *target) {
+static void add_redir(t_command *cmd, t_redir_type type, const char *target,
+        int target_quoted) {
     t_redir *node;
     t_redir *tail;
 
     node = (t_redir *)sh_xcalloc(1, sizeof(t_redir));
     node->type = type;
     node->target = sh_strdup(target);
+    node->heredoc_quoted = (type == REDIR_HEREDOC && target_quoted);
     if (!cmd->redirs) {
         cmd->redirs = node;
         return;
@@ -142,7 +144,8 @@ t_pipeline *parse_tokens(t_token *tokens, char **error) {
                 free_pipeline(head);
                 return NULL;
             }
-            add_redir(cmd, redir_type(cur->type), cur->next->text);
+            add_redir(cmd, redir_type(cur->type), cur->next->text,
+                cur->next->quoted);
             cur = cur->next;
             after_pipe = 0;
         } else if (cur->type == TOK_PIPE) {
diff --git a/src/token.c b/src/token.c
index ccae0c2..5b657d1 100644
--- a/src/token.c
+++ b/src/token.c
@@ -5,11 +5,13 @@
 
 #define LITERAL_MARK '\001'
 
-static t_token *new_token(t_token_type type, char *text, size_t start) {
+static t_token *new_token(t_token_type type, char *text, size_t start,
+        int quoted) {
     t_token *token = (t_token *)sh_xcalloc(1, sizeof(t_token));
     token->type = type;
     token->text = text ? text : sh_strdup("");
     token->start = start;
+    token->quoted = quoted;
     return token;
 }
 
@@ -43,15 +45,18 @@ static char *append_literal(char *word, char c) {
     return append_char(word, c);
 }
 
-static char *read_word(const char *line, size_t *i, char **error) {
+static char *read_word(const char *line, size_t *i, char **error,
+        int *quoted) {
     char    *word;
     char    quote;
 
     word = sh_strdup("");
+    *quoted = 0;
     while (line[*i] && !is_shell_space(line[*i])
         && !is_operator_char(line[*i])) {
         if (line[*i] == '\'' || line[*i] == '"') {
             quote = line[*i];
+            *quoted = 1;
             (*i)++;
             while (line[*i] && line[*i] != quote) {
                 if (quote == '\'')
@@ -80,6 +85,7 @@ t_token *tokenize_line(const char *line, char **error) {
     size_t  i;
     size_t  start;
     char    *word;
+    int     quoted;
 
     head = NULL;
     tail = NULL;
@@ -93,14 +99,14 @@ t_token *tokenize_line(const char *line, char **error) {
             break;
         if (line[i] == '|') {
             if (line[i + 1] == '|') {
-                push_token(&head, &tail, new_token(TOK_OR, sh_strdup("||"), i));
+                push_token(&head, &tail, new_token(TOK_OR, sh_strdup("||"), i, 0));
                 i += 2;
             } else {
-                push_token(&head, &tail, new_token(TOK_PIPE, sh_strdup("|"), i));
+                push_token(&head, &tail, new_token(TOK_PIPE, sh_strdup("|"), i, 0));
                 i++;
             }
         } else if (line[i] == '&' && line[i + 1] == '&') {
-            push_token(&head, &tail, new_token(TOK_AND, sh_strdup("&&"), i));
+            push_token(&head, &tail, new_token(TOK_AND, sh_strdup("&&"), i, 0));
             i += 2;
         } else if (line[i] == '&') {
             if (error)
@@ -108,32 +114,32 @@ t_token *tokenize_line(const char *line, char **error) {
             free_tokens(head);
             return NULL;
         } else if (line[i] == ';') {
-            push_token(&head, &tail, new_token(TOK_SEQ, sh_strdup(";"), i));
+            push_token(&head, &tail, new_token(TOK_SEQ, sh_strdup(";"), i, 0));
             i++;
         } else if (line[i] == '<') {
             if (line[i + 1] == '<') {
-                push_token(&head, &tail, new_token(TOK_HEREDOC, sh_strdup("<<"), i));
+                push_token(&head, &tail, new_token(TOK_HEREDOC, sh_strdup("<<"), i, 0));
                 i += 2;
             } else {
-                push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i));
+                push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i, 0));
                 i++;
             }
         } else if (line[i] == '>') {
             if (line[i + 1] == '>') {
-                push_token(&head, &tail, new_token(TOK_REDIR_APPEND, sh_strdup(">>"), i));
+                push_token(&head, &tail, new_token(TOK_REDIR_APPEND, sh_strdup(">>"), i, 0));
                 i += 2;
             } else {
-                push_token(&head, &tail, new_token(TOK_REDIR_OUT, sh_strdup(">"), i));
+                push_token(&head, &tail, new_token(TOK_REDIR_OUT, sh_strdup(">"), i, 0));
                 i++;
             }
         } else {
             start = i;
-            word = read_word(line, &i, error);
+            word = read_word(line, &i, error, &quoted);
             if (!word) {
                 free_tokens(head);
                 return NULL;
             }
-            push_token(&head, &tail, new_token(TOK_WORD, word, start));
+            push_token(&head, &tail, new_token(TOK_WORD, word, start, quoted));
         }
     }
     return head;


## `test(parser): 조건 연결자와 잘못된 연산자 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index ad5cb18..39ef5cb 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -214,4 +214,33 @@ echo \$?
 " \
 0
 
+run_case conditional_connectors \
+"false && echo skipped
+false || echo recovered
+true && echo continued
+" \
+"recovered
+continued
+" \
+0
+
+run_case malformed_conditionals \
+"&& echo never
+echo \$?
+echo never ||
+echo \$?
+" \
+"258
+258
+" \
+0
+
+run_case unsupported_operator \
+"echo never & echo never
+echo \$?
+" \
+"258
+" \
+0
+
 echo "ok - smoke"


## `refactor(buffer): 가변 문자열 빌더 모듈 추가`

diff --git a/Makefile b/Makefile
index 19efba4..4070777 100644
--- a/Makefile
+++ b/Makefile
@@ -13,6 +13,7 @@ SRCS := \
 	src/expand.c \
 	src/env.c \
 	src/utils.c \
+	src/string_builder.c \
 	src/runtime.c \
 	src/exec.c \
 	src/heredoc.c \
diff --git a/src/string_builder.c b/src/string_builder.c
new file mode 100644
index 0000000..c533c38
--- /dev/null
+++ b/src/string_builder.c
@@ -0,0 +1,109 @@
+#include "string_builder.h"
+#include "runtime.h"
+
+#include <errno.h>
+#include <stdint.h>
+#include <stdlib.h>
+#include <string.h>
+
+#define STRING_BUILDER_INITIAL_CAPACITY 64
+
+int string_builder_init(t_string_builder *builder)
+{
+    if (builder == NULL) {
+        errno = EINVAL;
+        return 1;
+    }
+    builder->data = (char *)shell_malloc(STRING_BUILDER_INITIAL_CAPACITY);
+    builder->length = 0;
+    builder->capacity = 0;
+    if (builder->data == NULL)
+        return 1;
+    builder->capacity = STRING_BUILDER_INITIAL_CAPACITY;
+    builder->data[0] = '\0';
+    return 0;
+}
+
+void string_builder_discard(t_string_builder *builder)
+{
+    if (builder == NULL)
+        return;
+    free(builder->data);
+    builder->data = NULL;
+    builder->length = 0;
+    builder->capacity = 0;
+}
+
+static int string_builder_reserve(t_string_builder *builder, size_t extra)
+{
+    size_t  needed;
+    size_t  capacity;
+    char    *grown;
+
+    if (extra > SIZE_MAX - builder->length - 1) {
+        errno = ENOMEM;
+        return 1;
+    }
+    needed = builder->length + extra + 1;
+    if (needed <= builder->capacity)
+        return 0;
+    capacity = builder->capacity;
+    while (capacity < needed) {
+        if (capacity > SIZE_MAX / 2) {
+            capacity = needed;
+            break;
+        }
+        capacity *= 2;
+    }
+    grown = (char *)shell_realloc(builder->data, capacity);
+    if (grown == NULL)
+        return 1;
+    builder->data = grown;
+    builder->capacity = capacity;
+    return 0;
+}
+
+int string_builder_append_char(t_string_builder *builder, char value)
+{
+    if (builder == NULL || builder->data == NULL) {
+        errno = EINVAL;
+        return 1;
+    }
+    if (string_builder_reserve(builder, 1) != 0)
+        return 1;
+    builder->data[builder->length++] = value;
+    builder->data[builder->length] = '\0';
+    return 0;
+}
+
+int string_builder_append_text(t_string_builder *builder, const char *text)
+{
+    size_t length;
+
+    if (builder == NULL || builder->data == NULL) {
+        errno = EINVAL;
+        return 1;
+    }
+    if (text == NULL)
+        return 0;
+    length = strlen(text);
+    if (string_builder_reserve(builder, length) != 0)
+        return 1;
+    memcpy(builder->data + builder->length, text, length);
+    builder->length += length;
+    builder->data[builder->length] = '\0';
+    return 0;
+}
+
+char *string_builder_take(t_string_builder *builder)
+{
+    char *data;
+
+    if (builder == NULL)
+        return NULL;
+    data = builder->data;
+    builder->data = NULL;
+    builder->length = 0;
+    builder->capacity = 0;
+    return data;
+}
diff --git a/src/string_builder.h b/src/string_builder.h
new file mode 100644
index 0000000..597df22
--- /dev/null
+++ b/src/string_builder.h
@@ -0,0 +1,19 @@
+#ifndef STRING_BUILDER_H
+# define STRING_BUILDER_H
+
+# include <stddef.h>
+
+typedef struct s_string_builder {
+    char    *data;
+    size_t  length;
+    size_t  capacity;
+}   t_string_builder;
+
+int     string_builder_init(t_string_builder *builder);
+void    string_builder_discard(t_string_builder *builder);
+int     string_builder_append_char(t_string_builder *builder, char value);
+int     string_builder_append_text(t_string_builder *builder,
+            const char *text);
+char    *string_builder_take(t_string_builder *builder);
+
+#endif


## `refactor(lexer): 단어 조립을 가변 버퍼로 전환`

diff --git a/src/token.c b/src/token.c
index 5465b58..ae73803 100644
--- a/src/token.c
+++ b/src/token.c
@@ -1,7 +1,7 @@
 #include "shell.h"
+#include "string_builder.h"
 
 #include <stdlib.h>
-#include <string.h>
 
 #define LITERAL_MARK '\001'
 
@@ -62,31 +62,19 @@ static int is_shell_space(char c)
         || c == '\r' || c == '\v' || c == '\f');
 }
 
-static char *append_char(char *word, char c)
+static int append_literal(t_string_builder *word, char c)
 {
-    char buf[2];
-
-    buf[0] = c;
-    buf[1] = '\0';
-    return sh_strjoin_free(word, buf);
-}
-
-static char *append_literal(char *word, char c)
-{
-    word = append_char(word, LITERAL_MARK);
-    if (word == NULL)
-        return NULL;
-    return append_char(word, c);
+    return (string_builder_append_char(word, LITERAL_MARK) != 0
+        || string_builder_append_char(word, c) != 0);
 }
 
 static char *read_word(const char *line, size_t *i, char **error,
         int *quoted)
 {
-    char    *word;
-    char    quote;
+    t_string_builder    word;
+    char                quote;
 
-    word = sh_strdup("");
-    if (word == NULL)
+    if (string_builder_init(&word) != 0)
         return NULL;
     *quoted = 0;
     while (line[*i] != '\0' && !is_shell_space(line[*i])
@@ -96,28 +84,33 @@ static char *read_word(const char *line, size_t *i, char **error,
             *quoted = 1;
             (*i)++;
             while (line[*i] != '\0' && line[*i] != quote) {
+                int failed;
+
                 if (quote == '\'')
-                    word = append_literal(word, line[*i]);
+                    failed = append_literal(&word, line[*i]);
                 else
-                    word = append_char(word, line[*i]);
-                if (word == NULL)
+                    failed = string_builder_append_char(&word, line[*i]);
+                if (failed != 0) {
+                    string_builder_discard(&word);
                     return NULL;
+                }
                 (*i)++;
             }
             if (line[*i] == '\0') {
-                free(word);
+                string_builder_discard(&word);
                 set_error(error, "syntax error: unclosed quote");
                 return NULL;
             }
             (*i)++;
         } else {
-            word = append_char(word, line[*i]);
-            if (word == NULL)
+            if (string_builder_append_char(&word, line[*i]) != 0) {
+                string_builder_discard(&word);
                 return NULL;
+            }
             (*i)++;
         }
     }
-    return word;
+    return string_builder_take(&word);
 }
 
 static int push_word(const char *line, size_t *i, char **error,
