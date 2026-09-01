## `feat(redirection): heredoc을 stdin으로 연결`

diff --git a/include/shell.h b/include/shell.h
index 024e0c1..d127c43 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -2,6 +2,7 @@
 # define SHELL_H
 
 # include <stddef.h>
+# include <unistd.h>
 
 typedef enum e_token_type {
     TOK_WORD,
@@ -9,6 +10,7 @@ typedef enum e_token_type {
     TOK_REDIR_IN,
     TOK_REDIR_OUT,
     TOK_REDIR_APPEND,
+    TOK_HEREDOC,
     TOK_AND,
     TOK_OR,
     TOK_SEQ
@@ -87,52 +89,54 @@ typedef struct s_executor_hooks {
     t_shell_error_fn        on_error;
 }   t_executor_hooks;
 
-char    *shell_read_line(const char *prompt, int interactive);
-void    shell_loop(t_shell *shell);
-char    *sh_strdup(const char *s);
-char    *sh_substr(const char *s, size_t start, size_t len);
-char    *sh_strjoin_free(char *left, const char *right);
-void    *sh_xcalloc(size_t count, size_t size);
-void    sh_free_words(char **words);
-char    *shell_strndup(const char *s, size_t len);
-char    *shell_itoa_status(int status);
-void    shell_strv_free(char **words);
-int     sh_is_name_char(int c);
-int     sh_is_name_start(int c);
-t_token *tokenize_line(const char *line, char **error);
-void    free_tokens(t_token *tokens);
+char        *sh_strdup(const char *s);
+char        *sh_substr(const char *s, size_t start, size_t len);
+char        *sh_strjoin_free(char *left, const char *right);
+void        *sh_xcalloc(size_t count, size_t size);
+void        sh_free_words(char **words);
+char        *shell_strndup(const char *s, size_t len);
+char        *shell_itoa_status(int status);
+void        shell_strv_free(char **words);
+int         sh_is_name_char(int c);
+int         sh_is_name_start(int c);
+
+t_token     *tokenize_line(const char *line, char **error);
+void        free_tokens(t_token *tokens);
 t_pipeline  *parse_tokens(t_token *tokens, char **error);
-void    free_pipeline(t_pipeline *pipeline);
-void    shell_sequence_init(t_sequence *sequence);
-void    shell_sequence_free(t_sequence *sequence);
-int     shell_parse_line(const char *line, t_sequence *sequence, char **error);
-int     shell_execute_sequence(const t_sequence *sequence, t_env *env,
-            int *last_status, const t_executor_hooks *hooks, void *ctx);
-char    *expand_word(t_shell *shell, const char *word);
-int     expand_pipeline(t_shell *shell, t_pipeline *pipeline);
-int     shell_dequote_word(const char *word, char **out, char **error);
-int     shell_expand_sequence(t_sequence *sequence, const t_env *env,
-            int last_status, char **error);
-t_env   *env_from_environ(char **envp);
-void    env_free(t_env *env);
+void        free_pipeline(t_pipeline *pipeline);
+void        shell_sequence_init(t_sequence *sequence);
+void        shell_sequence_free(t_sequence *sequence);
+int         shell_parse_line(const char *line, t_sequence *sequence, char **error);
+
+t_env       *env_from_environ(char **envp);
+void        env_free(t_env *env);
 const char  *env_get(t_env *env, const char *key);
-int     env_set(t_env **env, const char *key, const char *value, int exported);
-int     env_unset(t_env **env, const char *key);
-char    **env_to_environ(t_env *env);
-void    env_print(t_env *env, int declare_style);
-int     shell_env_init(t_env *env, char **envp);
-void    shell_env_free(t_env *env);
+int         env_set(t_env **env, const char *key, const char *value, int exported);
+int         env_unset(t_env **env, const char *key);
+char        **env_to_environ(t_env *env);
+void        env_print(t_env *env, int declare_style);
+int         shell_env_init(t_env *env, char **envp);
+void        shell_env_free(t_env *env);
 const char  *shell_env_get(const t_env *env, const char *key);
-int     shell_env_set(t_env *env, const char *key, const char *value,
-            int exported);
-int     shell_env_unset(t_env *env, const char *key);
-int     shell_env_is_valid_name(const char *key);
-char    **shell_env_export_list(t_env *env);
-char    **shell_env_to_envp(t_env *env);
-int     builtin_is_parent(const char *name);
-int     builtin_is_known(const char *name);
-int     builtin_run(t_shell *shell, char **argv);
-int     execute_pipeline_list(t_shell *shell, t_pipeline *pipeline);
-int     shell_process_line(t_shell *shell, const char *line);
+int         shell_env_set(t_env *env, const char *key, const char *value, int exported);
+int         shell_env_unset(t_env *env, const char *key);
+int         shell_env_is_valid_name(const char *key);
+char        **shell_env_export_list(t_env *env);
+char        **shell_env_to_envp(t_env *env);
+
+char        *expand_word(t_shell *shell, const char *word);
+int         expand_pipeline(t_shell *shell, t_pipeline *pipeline);
+int         shell_dequote_word(const char *word, char **out, char **error);
+int         shell_expand_sequence(t_sequence *sequence, const t_env *env,
+                int last_status, char **error);
+
+int         execute_pipeline_list(t_shell *shell, t_pipeline *pipeline);
+int         shell_execute_sequence(const t_sequence *sequence, t_env *env,
+                int *last_status, const t_executor_hooks *hooks, void *ctx);
+int         builtin_is_parent(const char *name);
+int         builtin_is_known(const char *name);
+int         builtin_run(t_shell *shell, char **argv);
+int         shell_process_line(t_shell *shell, const char *line);
+void        shell_loop(t_shell *shell);
 
 #endif
diff --git a/src/exec.c b/src/exec.c
index 6a2a2b6..53e89c0 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -217,6 +217,7 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
     struct exec_context ctx;
 
     ctx.shell = shell;
+    ctx.heredocs = NULL;
     return execute_pipeline_list_ctx(shell, pipeline, &ctx);
 }
 
@@ -224,10 +225,12 @@ int shell_process_line(t_shell *shell, const char *line)
 {
     t_token *tokens;
     t_pipeline *pipelines;
+    struct exec_context ctx;
     char *error;
 
     if (shell == NULL || line == NULL || line[0] == '\0')
         return shell != NULL ? shell->last_status : 1;
+
     error = NULL;
     tokens = tokenize_line(line, &error);
     if (error != NULL) {
@@ -236,6 +239,7 @@ int shell_process_line(t_shell *shell, const char *line)
         shell->last_status = 258;
         return shell->last_status;
     }
+
     pipelines = parse_tokens(tokens, &error);
     free_tokens(tokens);
     if (error != NULL) {
@@ -246,7 +250,18 @@ int shell_process_line(t_shell *shell, const char *line)
     }
     if (pipelines == NULL)
         return shell->last_status;
-    (void)execute_pipeline_list(shell, pipelines);
+
+    ctx.shell = shell;
+    ctx.heredocs = NULL;
+    if (exec_prepare_heredocs(&ctx, pipelines) != 0) {
+        exec_heredoc_entries_free(ctx.heredocs);
+        free_pipeline(pipelines);
+        shell->last_status = 1;
+        return shell->last_status;
+    }
+
+    (void)execute_pipeline_list_ctx(shell, pipelines, &ctx);
+    exec_heredoc_entries_free(ctx.heredocs);
     free_pipeline(pipelines);
     return shell->last_status;
 }
diff --git a/src/exec_internal.h b/src/exec_internal.h
index 372ba1d..c57c48b 100644
--- a/src/exec_internal.h
+++ b/src/exec_internal.h
@@ -14,13 +14,14 @@ struct exec_context {
     struct heredoc_entry    *heredocs;
 };
 
-int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines);
-void exec_heredoc_entries_free(struct heredoc_entry *entry);
-const char *exec_find_heredoc_body(const struct exec_context *ctx,
-        const t_redir *redir);
-int exec_apply_redirections(const t_command *command,
-        const struct exec_context *ctx);
-int exec_run_parent_command(t_shell *shell, const t_command *command,
-        const struct exec_context *ctx);
+int         exec_prepare_heredocs(struct exec_context *ctx,
+                t_pipeline *pipelines);
+void        exec_heredoc_entries_free(struct heredoc_entry *entry);
+const char  *exec_find_heredoc_body(const struct exec_context *ctx,
+                const t_redir *redir);
+int         exec_apply_redirections(const t_command *command,
+                const struct exec_context *ctx);
+int         exec_run_parent_command(t_shell *shell, const t_command *command,
+                const struct exec_context *ctx);
 
 #endif
diff --git a/src/expand.c b/src/expand.c
index d20b330..3411bf5 100644
--- a/src/expand.c
+++ b/src/expand.c
@@ -105,7 +105,10 @@ int expand_pipeline(t_shell *shell, t_pipeline *pipeline) {
             expand_words(shell, &cmd->argv);
             redir = cmd->redirs;
             while (redir) {
-                expanded = expand_word(shell, redir->target);
+                if (redir->type == REDIR_HEREDOC)
+                    expanded = dequote_word(redir->target);
+                else
+                    expanded = expand_word(shell, redir->target);
                 free(redir->target);
                 redir->target = expanded;
                 redir = redir->next;
diff --git a/src/input.c b/src/input.c
index f581496..034dd28 100644
--- a/src/input.c
+++ b/src/input.c
@@ -13,20 +13,22 @@
 
 static char *read_plain_line(const char *prompt, int interactive)
 {
-    size_t  cap;
-    size_t  len;
-    char    *line;
-    int     ch;
+    size_t cap;
+    size_t len;
+    char *line;
+    int ch;
 
     if (interactive && prompt != NULL) {
         fputs(prompt, stderr);
         fflush(stderr);
     }
+
     cap = 128;
     len = 0;
     line = malloc(cap);
     if (line == NULL)
         return NULL;
+
     while ((ch = fgetc(stdin)) != EOF) {
         char *grown;
 
@@ -43,10 +45,12 @@ static char *read_plain_line(const char *prompt, int interactive)
         }
         line[len++] = (char)ch;
     }
+
     if (ch == EOF && len == 0) {
         free(line);
         return NULL;
     }
+
     line[len] = '\0';
     return line;
 }
@@ -68,8 +72,8 @@ char *shell_read_line(const char *prompt, int interactive)
 
 void shell_loop(t_shell *shell)
 {
-    int     interactive;
-    char    *line;
+    int interactive;
+    char *line;
 
     if (shell == NULL)
         return;
diff --git a/src/parser.c b/src/parser.c
index 92f35e8..3e73b51 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -91,7 +91,7 @@ static void append_pipeline(t_pipeline **head, t_pipeline **tail, t_pipeline *no
 
 static int token_is_redir(t_token_type type) {
     return (type == TOK_REDIR_IN || type == TOK_REDIR_OUT
-        || type == TOK_REDIR_APPEND);
+        || type == TOK_REDIR_APPEND || type == TOK_HEREDOC);
 }
 
 static t_redir_type redir_type(t_token_type type) {
@@ -99,6 +99,8 @@ static t_redir_type redir_type(t_token_type type) {
         return REDIR_OUT;
     if (type == TOK_REDIR_APPEND)
         return REDIR_APPEND;
+    if (type == TOK_HEREDOC)
+        return REDIR_HEREDOC;
     return REDIR_IN;
 }
 
@@ -265,25 +267,28 @@ void shell_sequence_free(t_sequence *sequence) {
     if (!sequence)
         return;
     free_pipeline(sequence->pipelines);
-    shell_sequence_init(sequence);
+    sequence->pipelines = NULL;
+    sequence->pipeline_count = 0;
 }
 
 int shell_parse_line(const char *line, t_sequence *sequence, char **error) {
     t_token *tokens;
 
-    if (!sequence)
+    if (!sequence) {
+        if (error)
+            *error = sh_strdup("parse output is null");
         return 1;
+    }
     shell_sequence_init(sequence);
     tokens = tokenize_line(line, error);
-    if (!tokens) {
-        if (error && *error)
-            return 1;
-        return 0;
-    }
+    if (error && *error)
+        return 1;
     sequence->pipelines = parse_tokens(tokens, error);
     free_tokens(tokens);
-    if (!sequence->pipelines && error && *error)
+    if (error && *error) {
+        shell_sequence_free(sequence);
         return 1;
+    }
     sequence->pipeline_count = count_pipelines(sequence->pipelines);
     return 0;
 }
diff --git a/src/redirection.c b/src/redirection.c
index 290bc7e..a8a5b95 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -13,7 +13,6 @@ int exec_apply_redirections(const t_command *command,
 {
     const t_redir *redir;
 
-    (void)ctx;
     redir = command->redirs;
     while (redir != NULL) {
         int fd;
@@ -52,6 +51,31 @@ int exec_apply_redirections(const t_command *command,
                 return 1;
             }
             close(fd);
+        } else if (redir->type == REDIR_HEREDOC) {
+            FILE        *tmp;
+            const char  *body;
+
+            tmp = tmpfile();
+            if (tmp == NULL) {
+                fprintf(stderr, "small-shell: heredoc: %s\n",
+                    strerror(errno));
+                return 1;
+            }
+            body = exec_find_heredoc_body(ctx, redir);
+            if (body != NULL && fputs(body, tmp) == EOF) {
+                fprintf(stderr, "small-shell: heredoc: %s\n",
+                    strerror(errno));
+                fclose(tmp);
+                return 1;
+            }
+            fflush(tmp);
+            rewind(tmp);
+            if (dup2(fileno(tmp), STDIN_FILENO) < 0) {
+                fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
+                fclose(tmp);
+                return 1;
+            }
+            fclose(tmp);
         }
         redir = redir->next;
     }
diff --git a/src/token.c b/src/token.c
index 3f1ef2a..ccae0c2 100644
--- a/src/token.c
+++ b/src/token.c
@@ -112,13 +112,12 @@ t_token *tokenize_line(const char *line, char **error) {
             i++;
         } else if (line[i] == '<') {
             if (line[i + 1] == '<') {
-                if (error)
-                    *error = sh_strdup("syntax error: unsupported operator '<<'");
-                free_tokens(head);
-                return NULL;
+                push_token(&head, &tail, new_token(TOK_HEREDOC, sh_strdup("<<"), i));
+                i += 2;
+            } else {
+                push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i));
+                i++;
             }
-            push_token(&head, &tail, new_token(TOK_REDIR_IN, sh_strdup("<"), i));
-            i++;
         } else if (line[i] == '>') {
             if (line[i + 1] == '>') {
                 push_token(&head, &tail, new_token(TOK_REDIR_APPEND, sh_strdup(">>"), i));


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


## `fix(parser): 오류 출력 포인터 없이도 구문 실패 반환`

diff --git a/src/parser.c b/src/parser.c
index 81b6bad..00efc78 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -275,24 +275,32 @@ void shell_sequence_free(t_sequence *sequence) {
 }
 
 int shell_parse_line(const char *line, t_sequence *sequence, char **error) {
-    t_token *tokens;
+    t_token  *tokens;
+    char     *internal_error;
+    char     **error_slot;
 
+    internal_error = NULL;
+    error_slot = error ? error : &internal_error;
     if (!sequence) {
-        if (error)
-            *error = sh_strdup("parse output is null");
+        *error_slot = sh_strdup("parse output is null");
+        free(internal_error);
         return 1;
     }
     shell_sequence_init(sequence);
-    tokens = tokenize_line(line, error);
-    if (error && *error)
+    tokens = tokenize_line(line, error_slot);
+    if (*error_slot) {
+        free(internal_error);
         return 1;
-    sequence->pipelines = parse_tokens(tokens, error);
+    }
+    sequence->pipelines = parse_tokens(tokens, error_slot);
     free_tokens(tokens);
-    if (error && *error) {
+    if (*error_slot) {
         shell_sequence_free(sequence);
+        free(internal_error);
         return 1;
     }
     sequence->pipeline_count = count_pipelines(sequence->pipelines);
+    free(internal_error);
     return 0;
 }
 


## `test(parser): 공개 parser 오류 반환 검증`

diff --git a/.gitignore b/.gitignore
index 3ebfbaa..690532f 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,4 +1,5 @@
 small-shell
+/tests/parser-api
 *.o
 *.d
 evidence/tmp/
diff --git a/Makefile b/Makefile
index 92c8e3a..304851f 100644
--- a/Makefile
+++ b/Makefile
@@ -18,6 +18,7 @@ SRCS := \
 	src/redirection.c \
 	src/builtin.c
 OBJS := $(SRCS:.c=.o)
+PARSER_API_TARGET := tests/parser-api
 
 ifeq ($(USE_READLINE),1)
 CPPFLAGS += -DUSE_READLINE
@@ -32,13 +33,18 @@ $(TARGET): $(OBJS)
 %.o: %.c
 	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<
 
+$(PARSER_API_TARGET): tests/parser_api.c $(filter-out src/main.c,$(SRCS))
+	$(CC) $(CPPFLAGS) $(CFLAGS) $(LDFLAGS) -o $@ \
+		tests/parser_api.c $(filter-out src/main.c,$(SRCS)) $(LDLIBS)
+
 readline:
 	$(MAKE) USE_READLINE=1
 
-test: $(TARGET)
+test: $(TARGET) $(PARSER_API_TARGET)
 	./tests/smoke.sh
+	./$(PARSER_API_TARGET)
 
 clean:
-	rm -f $(TARGET) $(OBJS)
+	rm -f $(TARGET) $(PARSER_API_TARGET) $(OBJS)
 
 .PHONY: all readline test clean
diff --git a/tests/parser_api.c b/tests/parser_api.c
new file mode 100644
index 0000000..04d0970
--- /dev/null
+++ b/tests/parser_api.c
@@ -0,0 +1,54 @@
+#include "shell.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+
+static int check_line(const char *name, const char *line, int expect_failure,
+        int request_error)
+{
+    t_sequence  sequence;
+    char        *error;
+    int         result;
+
+    error = NULL;
+    result = shell_parse_line(line, &sequence,
+            request_error ? &error : NULL);
+    if ((result != 0) != expect_failure) {
+        fprintf(stderr, "not ok - %s: result=%d\n", name, result);
+        free(error);
+        if (result == 0)
+            shell_sequence_free(&sequence);
+        return 1;
+    }
+    if (request_error && ((error != NULL) != expect_failure)) {
+        fprintf(stderr, "not ok - %s: unexpected error state\n", name);
+        free(error);
+        if (result == 0)
+            shell_sequence_free(&sequence);
+        return 1;
+    }
+    free(error);
+    if (result == 0)
+        shell_sequence_free(&sequence);
+    return 0;
+}
+
+int main(void)
+{
+    int failed;
+
+    failed = 0;
+    failed |= check_line("valid command", "echo ok", 0, 1);
+    failed |= check_line("empty input", "", 0, 0);
+    failed |= check_line("pipe without error output", "echo |", 1, 0);
+    failed |= check_line("quote without error output", "echo 'open", 1, 0);
+    failed |= check_line("operator with error output", "echo &", 1, 1);
+    if (shell_parse_line("echo ok", NULL, NULL) == 0) {
+        fprintf(stderr, "not ok - null parse output accepted\n");
+        failed = 1;
+    }
+    if (failed)
+        return 1;
+    puts("ok - parser api");
+    return 0;
+}


