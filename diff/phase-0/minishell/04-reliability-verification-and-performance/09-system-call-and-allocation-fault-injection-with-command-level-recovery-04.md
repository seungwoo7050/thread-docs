## `fix(memory): 실행 자원 할당 실패를 pipeline 오류로 전파`

diff --git a/src/exec.c b/src/exec.c
index 85f7a3b..84e75cc 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -139,13 +139,19 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
     wait_error = 0;
 
     if (pipe_count > 0) {
-        pipes = (int (*)[2])shell_malloc(sizeof(int[2]) * pipe_count);
+        pipes = (int (*)[2])shell_calloc(pipe_count, sizeof(int[2]));
         if (pipes == NULL)
             goto alloc_error;
         for (i = 0; i < pipe_count; i++) {
             pipes[i][0] = -1;
             pipes[i][1] = -1;
         }
+    }
+    pids = (pid_t *)shell_calloc(pipeline->command_count, sizeof(pid_t));
+    if (pids == NULL)
+        goto alloc_error;
+
+    if (pipe_count > 0) {
         for (i = 0; i < pipe_count; i++) {
             if (shell_pipe(pipes[i]) < 0) {
                 fprintf(stderr, "small-shell: pipe: %s\n", strerror(errno));
@@ -156,10 +162,6 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
         }
     }
 
-    pids = (pid_t *)shell_calloc(pipeline->command_count, sizeof(pid_t));
-    if (pids == NULL)
-        goto alloc_error;
-
     command = pipeline->commands;
     for (i = 0; i < pipeline->command_count && command != NULL; i++) {
         pid_t pid;


## `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구`

diff --git a/src/heredoc.c b/src/heredoc.c
index 9e752e4..8dab1bd 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -3,6 +3,7 @@
 #include "exec_internal.h"
 #include "runtime.h"
 
+#include <stdint.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
@@ -45,8 +46,11 @@ static int sb_reserve(struct strbuf *buf, size_t extra)
     needed = buf->len + extra + 1;
     if (needed <= buf->cap)
         return 0;
-    while (buf->cap < needed)
+    while (buf->cap < needed) {
+        if (buf->cap > SIZE_MAX / 2)
+            return 1;
         buf->cap *= 2;
+    }
     next = (char *)shell_realloc(buf->data, buf->cap);
     if (next == NULL)
         return 1;
@@ -165,6 +169,40 @@ static char *dequote_runtime_word(const char *word)
     return out.data;
 }
 
+static int delimiter_matches(const char *line, const char *encoded)
+{
+    size_t i;
+    size_t j;
+
+    i = 0;
+    j = 0;
+    while (encoded != NULL && encoded[i] != '\0') {
+        if (encoded[i] == LITERAL_MARK && encoded[i + 1] != '\0')
+            i++;
+        if (line[j] != encoded[i])
+            return 0;
+        i++;
+        j++;
+    }
+    return line[j] == '\0';
+}
+
+static void discard_heredoc(const char *delimiter, int interactive)
+{
+    for (;;) {
+        char *line;
+
+        line = shell_read_line("> ", interactive);
+        if (line == NULL)
+            return;
+        if (delimiter_matches(line, delimiter)) {
+            free(line);
+            return;
+        }
+        free(line);
+    }
+}
+
 static int append_heredoc_body_line(t_shell *shell, int quoted,
     struct strbuf *body, const char *line)
 {
@@ -225,15 +263,19 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     int             quoted;
     int             interactive;
 
+    interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     quoted = redir->heredoc_quoted;
     delimiter = dequote_runtime_word(redir->target);
-    if (delimiter == NULL)
+    if (delimiter == NULL) {
+        discard_heredoc(redir->target, interactive);
         return 1;
+    }
     free(redir->target);
     redir->target = delimiter;
-    if (sb_init(&body) != 0)
+    if (sb_init(&body) != 0) {
+        discard_heredoc(redir->target, interactive);
         return 1;
-    interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
+    }
     for (;;) {
         char *line;
 
@@ -250,6 +292,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
         }
         if (append_heredoc_body_line(ctx->shell, quoted, &body, line) != 0) {
             free(line);
+            discard_heredoc(redir->target, interactive);
             sb_free(&body);
             return 1;
         }
@@ -264,8 +307,12 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
 
 int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
 {
-    t_pipeline *pipeline;
+    t_pipeline  *pipeline;
+    int         failed;
+    int         interactive;
 
+    failed = 0;
+    interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     pipeline = pipelines;
     while (pipeline != NULL) {
         t_command *command;
@@ -276,11 +323,11 @@ int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
 
             redir = command->redirs;
             while (redir != NULL) {
-                if (redir->type == REDIR_HEREDOC
-                    && read_heredoc(ctx, redir) != 0) {
-                    fprintf(stderr,
-                        "small-shell: heredoc: allocation failure\n");
-                    return 1;
+                if (redir->type == REDIR_HEREDOC) {
+                    if (!failed && read_heredoc(ctx, redir) != 0)
+                        failed = 1;
+                    else if (failed)
+                        discard_heredoc(redir->target, interactive);
                 }
                 redir = redir->next;
             }
@@ -288,5 +335,7 @@ int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
         }
         pipeline = pipeline->next;
     }
-    return 0;
+    if (failed)
+        fprintf(stderr, "small-shell: heredoc: preparation failure\n");
+    return failed;
 }


## `fix(input): EOF와 입력 실패를 구분`

diff --git a/src/heredoc.c b/src/heredoc.c
index 8dab1bd..e6d4060 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -3,6 +3,7 @@
 #include "exec_internal.h"
 #include "runtime.h"
 
+#include <errno.h>
 #include <stdint.h>
 #include <stdio.h>
 #include <stdlib.h>
@@ -17,7 +18,7 @@ struct strbuf {
     size_t  cap;
 };
 
-char *shell_read_line(const char *prompt, int interactive);
+char *shell_read_line(const char *prompt, int interactive, int *failed);
 
 static int sb_init(struct strbuf *buf)
 {
@@ -187,17 +188,18 @@ static int delimiter_matches(const char *line, const char *encoded)
     return line[j] == '\0';
 }
 
-static void discard_heredoc(const char *delimiter, int interactive)
+static int discard_heredoc(const char *delimiter, int interactive)
 {
     for (;;) {
-        char *line;
+        char    *line;
+        int     failed;
 
-        line = shell_read_line("> ", interactive);
+        line = shell_read_line("> ", interactive, &failed);
         if (line == NULL)
-            return;
+            return failed;
         if (delimiter_matches(line, delimiter)) {
             free(line);
-            return;
+            return 0;
         }
         free(line);
     }
@@ -262,25 +264,34 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
     char            *delimiter;
     int             quoted;
     int             interactive;
+    int             input_failed;
 
     interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     quoted = redir->heredoc_quoted;
     delimiter = dequote_runtime_word(redir->target);
     if (delimiter == NULL) {
-        discard_heredoc(redir->target, interactive);
+        (void)discard_heredoc(redir->target, interactive);
         return 1;
     }
     free(redir->target);
     redir->target = delimiter;
     if (sb_init(&body) != 0) {
-        discard_heredoc(redir->target, interactive);
+        (void)discard_heredoc(redir->target, interactive);
         return 1;
     }
     for (;;) {
         char *line;
 
-        line = shell_read_line("> ", interactive);
+        line = shell_read_line("> ", interactive, &input_failed);
         if (line == NULL) {
+            if (input_failed) {
+                fprintf(stderr, "small-shell: heredoc input: %s\n",
+                    strerror(errno));
+                if (discard_heredoc(redir->target, interactive) != 0)
+                    ctx->shell->running = 0;
+                sb_free(&body);
+                return 1;
+            }
             fprintf(stderr,
                 "small-shell: warning: here-document delimited by end-of-file (wanted `%s')\n",
                 redir->target);
@@ -292,7 +303,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
         }
         if (append_heredoc_body_line(ctx->shell, quoted, &body, line) != 0) {
             free(line);
-            discard_heredoc(redir->target, interactive);
+            (void)discard_heredoc(redir->target, interactive);
             sb_free(&body);
             return 1;
         }
@@ -326,8 +337,9 @@ int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
                 if (redir->type == REDIR_HEREDOC) {
                     if (!failed && read_heredoc(ctx, redir) != 0)
                         failed = 1;
-                    else if (failed)
-                        discard_heredoc(redir->target, interactive);
+                    else if (failed
+                        && discard_heredoc(redir->target, interactive) != 0)
+                        failed = 1;
                 }
                 redir = redir->next;
             }
diff --git a/src/input.c b/src/input.c
index 597e9e1..9c226ea 100644
--- a/src/input.c
+++ b/src/input.c
@@ -3,8 +3,11 @@
 #include "shell.h"
 #include "runtime.h"
 
+#include <errno.h>
+#include <stdint.h>
 #include <stdio.h>
 #include <stdlib.h>
+#include <string.h>
 #include <unistd.h>
 
 #ifdef USE_READLINE
@@ -12,77 +15,104 @@
 #include <readline/readline.h>
 #endif
 
-static char *read_plain_line(const char *prompt, int interactive)
+static char *read_plain_line(const char *prompt, int interactive, int *failed)
 {
-    size_t cap;
-    size_t len;
-    char *line;
-    int ch;
+    size_t  cap;
+    size_t  len;
+    char    *line;
 
+    *failed = 0;
     if (interactive && prompt != NULL) {
         fputs(prompt, stderr);
         fflush(stderr);
     }
-
     cap = 128;
     len = 0;
-    line = shell_malloc(cap);
-    if (line == NULL)
+    line = (char *)shell_malloc(cap);
+    if (line == NULL) {
+        *failed = 1;
         return NULL;
+    }
+    for (;;) {
+        unsigned char   ch;
+        ssize_t         count;
 
-    while ((ch = fgetc(stdin)) != EOF) {
-        char *grown;
-
+        count = shell_read(STDIN_FILENO, &ch, 1);
+        if (count < 0 && errno == EINTR)
+            continue;
+        if (count < 0) {
+            free(line);
+            *failed = 1;
+            return NULL;
+        }
+        if (count == 0) {
+            if (len == 0) {
+                free(line);
+                return NULL;
+            }
+            break;
+        }
         if (ch == '\n')
             break;
         if (len + 1 >= cap) {
+            char *grown;
+
+            if (cap > SIZE_MAX / 2) {
+                free(line);
+                errno = ENOMEM;
+                *failed = 1;
+                return NULL;
+            }
             cap *= 2;
-            grown = shell_realloc(line, cap);
+            grown = (char *)shell_realloc(line, cap);
             if (grown == NULL) {
                 free(line);
+                *failed = 1;
                 return NULL;
             }
             line = grown;
         }
         line[len++] = (char)ch;
     }
-
-    if (ch == EOF && len == 0) {
-        free(line);
-        return NULL;
-    }
-
     line[len] = '\0';
     return line;
 }
 
-char *shell_read_line(const char *prompt, int interactive)
+char *shell_read_line(const char *prompt, int interactive, int *failed)
 {
 #ifdef USE_READLINE
     if (interactive) {
         char *line;
 
+        *failed = 0;
         line = readline(prompt != NULL ? prompt : "");
         if (line != NULL && line[0] != '\0')
             add_history(line);
         return line;
     }
 #endif
-    return read_plain_line(prompt, interactive);
+    return read_plain_line(prompt, interactive, failed);
 }
 
 void shell_loop(t_shell *shell)
 {
-    int interactive;
-    char *line;
+    int     interactive;
+    char    *line;
 
     if (shell == NULL)
         return;
     interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
     while (shell->running) {
-        line = shell_read_line("small-shell$ ", interactive);
-        if (line == NULL)
+        int failed;
+
+        line = shell_read_line("small-shell$ ", interactive, &failed);
+        if (line == NULL) {
+            if (failed) {
+                fprintf(stderr, "small-shell: input: %s\n", strerror(errno));
+                shell->last_status = 1;
+            }
             break;
+        }
         (void)shell_process_line(shell, line);
         free(line);
     }
diff --git a/src/runtime.c b/src/runtime.c
index 67405df..47d18f8 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -169,3 +169,8 @@ int shell_fileno(FILE *stream)
 {
     return fileno(stream);
 }
+
+ssize_t shell_read(int fd, void *buffer, size_t size)
+{
+    return read(fd, buffer, size);
+}
diff --git a/src/runtime.h b/src/runtime.h
index 285edc1..a78440d 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -18,5 +18,6 @@ int     shell_open(const char *path, int flags, mode_t mode);
 int     shell_fflush(FILE *stream);
 int     shell_fseek(FILE *stream, long offset, int whence);
 int     shell_fileno(FILE *stream);
+ssize_t shell_read(int fd, void *buffer, size_t size);
 
 #endif


