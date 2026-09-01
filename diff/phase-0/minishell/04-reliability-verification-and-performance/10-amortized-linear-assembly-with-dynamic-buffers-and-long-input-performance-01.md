# 가변 버퍼의 상각 선형 조립과 긴 입력 성능

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


## `feat(heredoc): 구분자 정규화 버퍼 구현`

diff --git a/Makefile b/Makefile
index cecab3e..cbd8550 100644
--- a/Makefile
+++ b/Makefile
@@ -14,6 +14,7 @@ SRCS := \
 	src/env.c \
 	src/utils.c \
 	src/exec.c \
+	src/heredoc.c \
 	src/redirection.c \
 	src/builtin.c
 OBJS := $(SRCS:.c=.o)
diff --git a/src/heredoc.c b/src/heredoc.c
new file mode 100644
index 0000000..902b931
--- /dev/null
+++ b/src/heredoc.c
@@ -0,0 +1,87 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "exec_internal.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+#define LITERAL_MARK '\001'
+
+struct strbuf {
+    char    *data;
+    size_t  len;
+    size_t  cap;
+};
+
+static int sb_init(struct strbuf *buf)
+{
+    buf->cap = 64;
+    buf->len = 0;
+    buf->data = (char *)malloc(buf->cap);
+    if (buf->data == NULL)
+        return 1;
+    buf->data[0] = '\0';
+    return 0;
+}
+
+static void sb_free(struct strbuf *buf)
+{
+    free(buf->data);
+    buf->data = NULL;
+    buf->len = 0;
+    buf->cap = 0;
+}
+
+static int sb_reserve(struct strbuf *buf, size_t extra)
+{
+    size_t  needed;
+    char    *next;
+
+    needed = buf->len + extra + 1;
+    if (needed <= buf->cap)
+        return 0;
+    while (buf->cap < needed)
+        buf->cap *= 2;
+    next = (char *)realloc(buf->data, buf->cap);
+    if (next == NULL)
+        return 1;
+    buf->data = next;
+    return 0;
+}
+
+static int sb_push(struct strbuf *buf, char ch)
+{
+    if (sb_reserve(buf, 1) != 0)
+        return 1;
+    buf->data[buf->len++] = ch;
+    buf->data[buf->len] = '\0';
+    return 0;
+}
+
+char *dequote_runtime_word(const char *word)
+{
+    struct strbuf   out;
+    size_t          i;
+
+    if (sb_init(&out) != 0)
+        return NULL;
+    i = 0;
+    while (word != NULL && word[i] != '\0') {
+        if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
+            if (sb_push(&out, word[i + 1]) != 0) {
+                sb_free(&out);
+                return NULL;
+            }
+            i += 2;
+        } else {
+            if (sb_push(&out, word[i]) != 0) {
+                sb_free(&out);
+                return NULL;
+            }
+            i++;
+        }
+    }
+    return out.data;
+}


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


