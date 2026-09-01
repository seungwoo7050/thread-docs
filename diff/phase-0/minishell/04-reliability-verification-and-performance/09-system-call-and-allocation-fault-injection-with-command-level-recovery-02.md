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


