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


## `test(heredoc): 인용 구분자와 본문 확장 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 7ca7fb8..74fb956 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -118,6 +118,18 @@ beta
 " \
 0
 
+run_case quoted_heredoc \
+"export HD=beta
+cat <<'EOF'
+alpha
+\$HD
+EOF
+" \
+"alpha
+\$HD
+" \
+0
+
 run_case syntax_error_status \
 "echo |
 echo \$?


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


