# 명령 반복의 단계 순서와 상태 수명

## `docs(readme): 프로젝트 목적과 초기 규약 정의`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..e6fe06d
--- /dev/null
+++ b/README.md
@@ -0,0 +1,40 @@
+# small-shell
+
+`small-shell`은 명령줄을 읽고 해석해 실행하는 과정을 C로 구현하며
+셸의 데이터 구조와 프로세스 수명을 학습하기 위한 프로젝트다.
+
+## 목표
+
+- C99와 POSIX 인터페이스를 사용한다.
+- tokenizer, parser, expansion 단계를 명확히 분리한다.
+- builtin과 외부 명령의 실행 경계를 구분한다.
+- pipeline, redirection, heredoc의 자원 소유권을 추적한다.
+- 오류를 호출자에게 전달하고 부분 결과를 정리한다.
+
+완성된 프로그램은 표준 입력에서 한 줄씩 읽어 제한된 셸 문법을 실행하는
+`small-shell` 실행 파일로 제공할 예정이다.
+
+## 개발 규약
+
+- `-std=c99 -Wall -Wextra -Wpedantic` 경고 없이 빌드한다.
+- 한 변경은 하나의 명확한 책임만 다룬다.
+- 새 추상화는 실제 사용 경로와 함께 검증한다.
+- 파일 디스크립터, 동적 메모리와 자식 프로세스의 소유자를 명시한다.
+- 실패 경로도 정상 경로와 같은 수준으로 정리한다.
+- 동작을 추가한 뒤 독립적으로 재현 가능한 검증을 남긴다.
+
+## 예정 범위
+
+- 인용 문자열과 셸 연산자의 tokenization
+- pipeline과 조건 연결자를 포함한 parsing
+- 환경 변수와 종료 상태 expansion
+- 기본 builtin과 외부 명령 실행
+- 파일 redirection과 heredoc
+
+background 실행, job control, subshell, glob과 완전한 POSIX shell 호환은
+이 프로젝트의 범위에 포함하지 않는다.
+
+## 검증 원칙
+
+각 단계는 그 시점에 존재하는 코드만으로 빌드할 수 있어야 한다. 기능이 연결된
+뒤에는 정상 동작뿐 아니라 입력·할당·시스템 호출 실패와 자원 수명도 검증한다.


## `build(shell): C99 실행 골격 구성`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..3ebfbaa
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,4 @@
+small-shell
+*.o
+*.d
+evidence/tmp/
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..1fb5aab
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,22 @@
+CC ?= cc
+CFLAGS ?= -std=c99 -Wall -Wextra -Wpedantic
+CPPFLAGS ?= -Iinclude
+LDFLAGS ?=
+LDLIBS ?=
+
+TARGET := small-shell
+SRCS := src/main.c
+OBJS := $(SRCS:.c=.o)
+
+all: $(TARGET)
+
+$(TARGET): $(OBJS)
+	$(CC) $(LDFLAGS) -o $@ $(OBJS) $(LDLIBS)
+
+%.o: %.c
+	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<
+
+clean:
+	rm -f $(TARGET) $(OBJS)
+
+.PHONY: all clean
diff --git a/src/main.c b/src/main.c
new file mode 100644
index 0000000..53e6e6c
--- /dev/null
+++ b/src/main.c
@@ -0,0 +1,8 @@
+#define _POSIX_C_SOURCE 200809L
+
+int main(int argc, char **argv)
+{
+    (void)argc;
+    (void)argv;
+    return 0;
+}


## `feat(input): 표준 입력 반복과 EOF 처리 연결`

diff --git a/Makefile b/Makefile
index 1fb5aab..d71f0a9 100644
--- a/Makefile
+++ b/Makefile
@@ -5,7 +5,9 @@ LDFLAGS ?=
 LDLIBS ?=
 
 TARGET := small-shell
-SRCS := src/main.c
+SRCS := \
+	src/main.c \
+	src/input.c
 OBJS := $(SRCS:.c=.o)
 
 all: $(TARGET)
diff --git a/include/shell.h b/include/shell.h
new file mode 100644
index 0000000..5735a7e
--- /dev/null
+++ b/include/shell.h
@@ -0,0 +1,11 @@
+#ifndef SHELL_H
+# define SHELL_H
+
+typedef struct s_shell {
+    int running;
+}   t_shell;
+
+char    *shell_read_line(const char *prompt, int interactive);
+void    shell_loop(t_shell *shell);
+
+#endif
diff --git a/src/input.c b/src/input.c
new file mode 100644
index 0000000..227a485
--- /dev/null
+++ b/src/input.c
@@ -0,0 +1,70 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "shell.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+#include <unistd.h>
+
+static char *read_plain_line(const char *prompt, int interactive)
+{
+    size_t  cap;
+    size_t  len;
+    char    *line;
+    int     ch;
+
+    if (interactive && prompt != NULL) {
+        fputs(prompt, stderr);
+        fflush(stderr);
+    }
+    cap = 128;
+    len = 0;
+    line = malloc(cap);
+    if (line == NULL)
+        return NULL;
+    while ((ch = fgetc(stdin)) != EOF) {
+        char *grown;
+
+        if (ch == '\n')
+            break;
+        if (len + 1 >= cap) {
+            cap *= 2;
+            grown = realloc(line, cap);
+            if (grown == NULL) {
+                free(line);
+                return NULL;
+            }
+            line = grown;
+        }
+        line[len++] = (char)ch;
+    }
+    if (ch == EOF && len == 0) {
+        free(line);
+        return NULL;
+    }
+    line[len] = '\0';
+    return line;
+}
+
+char *shell_read_line(const char *prompt, int interactive)
+{
+    return read_plain_line(prompt, interactive);
+}
+
+void shell_loop(t_shell *shell)
+{
+    int     interactive;
+    char    *line;
+
+    if (shell == NULL)
+        return;
+    interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
+    while (shell->running) {
+        line = shell_read_line("small-shell$ ", interactive);
+        if (line == NULL)
+            break;
+        free(line);
+    }
+    if (interactive && shell->running)
+        fputc('\n', stderr);
+}
diff --git a/src/main.c b/src/main.c
index 53e6e6c..ec782e2 100644
--- a/src/main.c
+++ b/src/main.c
@@ -1,8 +1,14 @@
 #define _POSIX_C_SOURCE 200809L
 
+#include "shell.h"
+
 int main(int argc, char **argv)
 {
+    t_shell shell;
+
     (void)argc;
     (void)argv;
+    shell.running = 1;
+    shell_loop(&shell);
     return 0;
 }


## `build(input): 선택적 readline 입력 경로 제공`

diff --git a/Makefile b/Makefile
index d71f0a9..83ca787 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,11 @@ SRCS := \
 	src/input.c
 OBJS := $(SRCS:.c=.o)
 
+ifeq ($(USE_READLINE),1)
+CPPFLAGS += -DUSE_READLINE
+LDLIBS += -lreadline
+endif
+
 all: $(TARGET)
 
 $(TARGET): $(OBJS)
@@ -18,7 +23,10 @@ $(TARGET): $(OBJS)
 %.o: %.c
 	$(CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<
 
+readline:
+	$(MAKE) USE_READLINE=1
+
 clean:
 	rm -f $(TARGET) $(OBJS)
 
-.PHONY: all clean
+.PHONY: all readline clean
diff --git a/src/input.c b/src/input.c
index 227a485..08db1d0 100644
--- a/src/input.c
+++ b/src/input.c
@@ -6,6 +6,11 @@
 #include <stdlib.h>
 #include <unistd.h>
 
+#ifdef USE_READLINE
+#include <readline/history.h>
+#include <readline/readline.h>
+#endif
+
 static char *read_plain_line(const char *prompt, int interactive)
 {
     size_t  cap;
@@ -48,6 +53,16 @@ static char *read_plain_line(const char *prompt, int interactive)
 
 char *shell_read_line(const char *prompt, int interactive)
 {
+#ifdef USE_READLINE
+    if (interactive) {
+        char *line;
+
+        line = readline(prompt != NULL ? prompt : "");
+        if (line != NULL && line[0] != '\0')
+            add_history(line);
+        return line;
+    }
+#endif
     return read_plain_line(prompt, interactive);
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


## `feat(shell): 한 줄 해석과 실행 수명 연결`

diff --git a/include/shell.h b/include/shell.h
index 19dc8d4..f21371d 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -131,5 +131,6 @@ int     builtin_is_parent(const char *name);
 int     builtin_is_known(const char *name);
 int     builtin_run(t_shell *shell, char **argv);
 int     execute_pipeline_list(t_shell *shell, t_pipeline *pipeline);
+int     shell_process_line(t_shell *shell, const char *line);
 
 #endif
diff --git a/src/exec.c b/src/exec.c
index d0d9795..6a2a2b6 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -219,3 +219,34 @@ int execute_pipeline_list(t_shell *shell, t_pipeline *pipeline)
     ctx.shell = shell;
     return execute_pipeline_list_ctx(shell, pipeline, &ctx);
 }
+
+int shell_process_line(t_shell *shell, const char *line)
+{
+    t_token *tokens;
+    t_pipeline *pipelines;
+    char *error;
+
+    if (shell == NULL || line == NULL || line[0] == '\0')
+        return shell != NULL ? shell->last_status : 1;
+    error = NULL;
+    tokens = tokenize_line(line, &error);
+    if (error != NULL) {
+        fprintf(stderr, "small-shell: %s\n", error);
+        free(error);
+        shell->last_status = 258;
+        return shell->last_status;
+    }
+    pipelines = parse_tokens(tokens, &error);
+    free_tokens(tokens);
+    if (error != NULL) {
+        fprintf(stderr, "small-shell: %s\n", error);
+        free(error);
+        shell->last_status = 258;
+        return shell->last_status;
+    }
+    if (pipelines == NULL)
+        return shell->last_status;
+    (void)execute_pipeline_list(shell, pipelines);
+    free_pipeline(pipelines);
+    return shell->last_status;
+}
diff --git a/src/input.c b/src/input.c
index 08db1d0..f581496 100644
--- a/src/input.c
+++ b/src/input.c
@@ -78,6 +78,7 @@ void shell_loop(t_shell *shell)
         line = shell_read_line("small-shell$ ", interactive);
         if (line == NULL)
             break;
+        (void)shell_process_line(shell, line);
         free(line);
     }
     if (interactive && shell->running)


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


