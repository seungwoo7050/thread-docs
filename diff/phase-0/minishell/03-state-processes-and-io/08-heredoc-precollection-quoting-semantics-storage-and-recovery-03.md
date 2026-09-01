## `test(heredoc): 이중·부분 인용 구분자 회귀 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 74fb956..30f152a 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -130,6 +130,30 @@ EOF
 " \
 0
 
+run_case double_quoted_heredoc \
+"export HD=beta
+cat <<\"EOF\"
+alpha
+\$HD
+EOF
+" \
+"alpha
+\$HD
+" \
+0
+
+run_case partially_quoted_heredoc \
+"export HD=beta
+cat <<E\"OF\"
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


## `refactor(runtime): heredoc 임시 파일 I/O 경계 분리`

diff --git a/src/redirection.c b/src/redirection.c
index f1cec1c..8c9a9b4 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -69,9 +69,9 @@ int exec_apply_redirections(const t_command *command,
                 fclose(tmp);
                 return 1;
             }
-            fflush(tmp);
-            rewind(tmp);
-            if (shell_dup2(fileno(tmp), STDIN_FILENO) < 0) {
+            (void)shell_fflush(tmp);
+            (void)shell_fseek(tmp, 0L, SEEK_SET);
+            if (shell_dup2(shell_fileno(tmp), STDIN_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 fclose(tmp);
                 return 1;
diff --git a/src/runtime.c b/src/runtime.c
index c764b16..7066875 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -118,3 +118,18 @@ int shell_open(const char *path, int flags, mode_t mode)
 #endif
     return open(path, flags, mode);
 }
+
+int shell_fflush(FILE *stream)
+{
+    return fflush(stream);
+}
+
+int shell_fseek(FILE *stream, long offset, int whence)
+{
+    return fseek(stream, offset, whence);
+}
+
+int shell_fileno(FILE *stream)
+{
+    return fileno(stream);
+}
diff --git a/src/runtime.h b/src/runtime.h
index 7558d03..c6f092f 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -1,6 +1,7 @@
 #ifndef RUNTIME_H
 # define RUNTIME_H
 
+# include <stdio.h>
 # include <sys/types.h>
 # include <sys/stat.h>
 
@@ -10,5 +11,8 @@ pid_t   shell_waitpid(pid_t pid, int *status, int options);
 int     shell_dup(int fd);
 int     shell_dup2(int oldfd, int newfd);
 int     shell_open(const char *path, int flags, mode_t mode);
+int     shell_fflush(FILE *stream);
+int     shell_fseek(FILE *stream, long offset, int whence);
+int     shell_fileno(FILE *stream);
 
 #endif


## `fix(heredoc): 임시 파일 저장 오류를 전파`

diff --git a/src/redirection.c b/src/redirection.c
index 8c9a9b4..2e9c99e 100644
--- a/src/redirection.c
+++ b/src/redirection.c
@@ -9,6 +9,20 @@
 #include <string.h>
 #include <unistd.h>
 
+static int heredoc_stream_error(FILE *stream, const char *operation)
+{
+    int saved_errno;
+
+    saved_errno = errno;
+    if (saved_errno == 0)
+        saved_errno = EIO;
+    fprintf(stderr, "small-shell: heredoc %s: %s\n", operation,
+        strerror(saved_errno));
+    fclose(stream);
+    errno = saved_errno;
+    return 1;
+}
+
 int exec_apply_redirections(const t_command *command,
     const struct exec_context *ctx)
 {
@@ -63,15 +77,16 @@ int exec_apply_redirections(const t_command *command,
                 return 1;
             }
             body = exec_find_heredoc_body(ctx, redir);
-            if (body != NULL && fputs(body, tmp) == EOF) {
-                fprintf(stderr, "small-shell: heredoc: %s\n",
-                    strerror(errno));
-                fclose(tmp);
-                return 1;
-            }
-            (void)shell_fflush(tmp);
-            (void)shell_fseek(tmp, 0L, SEEK_SET);
-            if (shell_dup2(shell_fileno(tmp), STDIN_FILENO) < 0) {
+            if (body != NULL && fputs(body, tmp) == EOF)
+                return heredoc_stream_error(tmp, "write");
+            if (shell_fflush(tmp) != 0)
+                return heredoc_stream_error(tmp, "flush");
+            if (shell_fseek(tmp, 0L, SEEK_SET) != 0)
+                return heredoc_stream_error(tmp, "seek");
+            fd = shell_fileno(tmp);
+            if (fd < 0)
+                return heredoc_stream_error(tmp, "descriptor");
+            if (shell_dup2(fd, STDIN_FILENO) < 0) {
                 fprintf(stderr, "small-shell: dup2: %s\n", strerror(errno));
                 fclose(tmp);
                 return 1;


## `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증`

diff --git a/src/runtime.c b/src/runtime.c
index 7066875..4e2661b 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -121,11 +121,27 @@ int shell_open(const char *path, int flags, mode_t mode)
 
 int shell_fflush(FILE *stream)
 {
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_FFLUSH", &calls)) {
+        errno = ENOSPC;
+        return EOF;
+    }
+#endif
     return fflush(stream);
 }
 
 int shell_fseek(FILE *stream, long offset, int whence)
 {
+#ifdef SMALL_SHELL_TESTING
+    static unsigned long calls;
+
+    if (fail_call("SMALL_SHELL_FAIL_FSEEK", &calls)) {
+        errno = EIO;
+        return -1;
+    }
+#endif
     return fseek(stream, offset, whence);
 }
 
diff --git a/tests/faults.sh b/tests/faults.sh
index 705de88..ec887ee 100755
--- a/tests/faults.sh
+++ b/tests/faults.sh
@@ -104,6 +104,26 @@ echo after" \
 after
 '
 
+run_fault heredoc_flush SMALL_SHELL_FAIL_FFLUSH 1 \
+    'cat <<EOF
+body
+EOF
+echo $?
+echo after' \
+    '1
+after
+'
+
+run_fault heredoc_seek SMALL_SHELL_FAIL_FSEEK 1 \
+    'cat <<EOF
+body
+EOF
+echo $?
+echo after' \
+    '1
+after
+'
+
 set +e
 env SMALL_SHELL_FAIL_DUP2=2 SMALL_SHELL_FAIL_DUP2_REPEAT=1 \
     "$BIN" >"$TMP/persistent-restore.out" \


## `refactor(runtime): 실행 경로의 동적 할당 래퍼 통합`

diff --git a/src/exec.c b/src/exec.c
index da74412..7b623aa 100644
--- a/src/exec.c
+++ b/src/exec.c
@@ -139,7 +139,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
     wait_error = 0;
 
     if (pipe_count > 0) {
-        pipes = (int (*)[2])malloc(sizeof(int[2]) * pipe_count);
+        pipes = (int (*)[2])shell_malloc(sizeof(int[2]) * pipe_count);
         if (pipes == NULL)
             goto alloc_error;
         for (i = 0; i < pipe_count; i++) {
@@ -156,7 +156,7 @@ static int run_forked_pipeline(t_shell *shell, const t_pipeline *pipeline, const
         }
     }
 
-    pids = (pid_t *)calloc(pipeline->command_count, sizeof(pid_t));
+    pids = (pid_t *)shell_calloc(pipeline->command_count, sizeof(pid_t));
     if (pids == NULL)
         goto alloc_error;
 
diff --git a/src/heredoc.c b/src/heredoc.c
index 2502529..9e752e4 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "exec_internal.h"
+#include "runtime.h"
 
 #include <stdio.h>
 #include <stdlib.h>
@@ -21,7 +22,7 @@ static int sb_init(struct strbuf *buf)
 {
     buf->cap = 64;
     buf->len = 0;
-    buf->data = (char *)malloc(buf->cap);
+    buf->data = (char *)shell_malloc(buf->cap);
     if (buf->data == NULL)
         return 1;
     buf->data[0] = '\0';
@@ -46,7 +47,7 @@ static int sb_reserve(struct strbuf *buf, size_t extra)
         return 0;
     while (buf->cap < needed)
         buf->cap *= 2;
-    next = (char *)realloc(buf->data, buf->cap);
+    next = (char *)shell_realloc(buf->data, buf->cap);
     if (next == NULL)
         return 1;
     buf->data = next;
@@ -181,7 +182,7 @@ static int add_heredoc_entry(struct exec_context *ctx, const t_redir *redir,
 {
     struct heredoc_entry *entry;
 
-    entry = (struct heredoc_entry *)malloc(sizeof(*entry));
+    entry = (struct heredoc_entry *)shell_malloc(sizeof(*entry));
     if (entry == NULL)
         return 1;
     entry->redir = redir;
diff --git a/src/input.c b/src/input.c
index 034dd28..597e9e1 100644
--- a/src/input.c
+++ b/src/input.c
@@ -1,6 +1,7 @@
 #define _POSIX_C_SOURCE 200809L
 
 #include "shell.h"
+#include "runtime.h"
 
 #include <stdio.h>
 #include <stdlib.h>
@@ -25,7 +26,7 @@ static char *read_plain_line(const char *prompt, int interactive)
 
     cap = 128;
     len = 0;
-    line = malloc(cap);
+    line = shell_malloc(cap);
     if (line == NULL)
         return NULL;
 
@@ -36,7 +37,7 @@ static char *read_plain_line(const char *prompt, int interactive)
             break;
         if (len + 1 >= cap) {
             cap *= 2;
-            grown = realloc(line, cap);
+            grown = shell_realloc(line, cap);
             if (grown == NULL) {
                 free(line);
                 return NULL;
diff --git a/src/runtime.c b/src/runtime.c
index 4e2661b..67405df 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -4,6 +4,7 @@
 
 #include <errno.h>
 #include <fcntl.h>
+#include <stdint.h>
 #include <stdlib.h>
 #include <string.h>
 #include <sys/wait.h>
@@ -41,6 +42,25 @@ static int fail_call(const char *name, unsigned long *calls)
 
 #endif
 
+void *shell_malloc(size_t size)
+{
+    return malloc(size);
+}
+
+void *shell_calloc(size_t count, size_t size)
+{
+    if (size != 0 && count > SIZE_MAX / size) {
+        errno = ENOMEM;
+        return NULL;
+    }
+    return calloc(count, size);
+}
+
+void *shell_realloc(void *ptr, size_t size)
+{
+    return realloc(ptr, size);
+}
+
 int shell_pipe(int fds[2])
 {
 #ifdef SMALL_SHELL_TESTING
diff --git a/src/runtime.h b/src/runtime.h
index c6f092f..285edc1 100644
--- a/src/runtime.h
+++ b/src/runtime.h
@@ -2,9 +2,13 @@
 # define RUNTIME_H
 
 # include <stdio.h>
+# include <stddef.h>
 # include <sys/types.h>
 # include <sys/stat.h>
 
+void    *shell_malloc(size_t size);
+void    *shell_calloc(size_t count, size_t size);
+void    *shell_realloc(void *ptr, size_t size);
 int     shell_pipe(int fds[2]);
 pid_t   shell_fork(void);
 pid_t   shell_waitpid(pid_t pid, int *status, int options);
diff --git a/src/utils.c b/src/utils.c
index c76e855..384eb15 100644
--- a/src/utils.c
+++ b/src/utils.c
@@ -1,4 +1,5 @@
 #include "shell.h"
+#include "runtime.h"
 
 #include <ctype.h>
 #include <stdio.h>
@@ -6,7 +7,7 @@
 #include <string.h>
 
 void *sh_xcalloc(size_t count, size_t size) {
-    void *ptr = calloc(count, size);
+    void *ptr = shell_calloc(count, size);
     if (!ptr) {
         perror("small-shell: calloc");
         exit(1);
@@ -21,7 +22,7 @@ char *sh_strdup(const char *s) {
     if (!s)
         s = "";
     len = strlen(s);
-    copy = (char *)malloc(len + 1);
+    copy = (char *)shell_malloc(len + 1);
     if (!copy) {
         perror("small-shell: malloc");
         exit(1);
@@ -119,7 +120,7 @@ char *shell_itoa_status(int status) {
             buf[len++] = digits[--count];
     }
     buf[len] = '\0';
-    out = (char *)malloc(len + 1);
+    out = (char *)shell_malloc(len + 1);
     if (!out)
         return NULL;
     memcpy(out, buf, len + 1);


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


