# heredoc의 사전 수집, 인용 의미, 저장·복구 수명

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


## `feat(heredoc): 수집 본문 저장소 수명 관리`

diff --git a/src/exec_internal.h b/src/exec_internal.h
index 28315df..1718d87 100644
--- a/src/exec_internal.h
+++ b/src/exec_internal.h
@@ -3,10 +3,20 @@
 
 # include "shell.h"
 
+struct heredoc_entry {
+    const t_redir           *redir;
+    char                    *body;
+    struct heredoc_entry    *next;
+};
+
 struct exec_context {
-    t_shell *shell;
+    t_shell                 *shell;
+    struct heredoc_entry    *heredocs;
 };
 
+void exec_heredoc_entries_free(struct heredoc_entry *entry);
+const char *exec_find_heredoc_body(const struct exec_context *ctx,
+        const t_redir *redir);
 int exec_apply_redirections(const t_command *command,
         const struct exec_context *ctx);
 int exec_run_parent_command(t_shell *shell, const t_command *command,
diff --git a/src/heredoc.c b/src/heredoc.c
index 902b931..58355e8 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -85,3 +85,44 @@ char *dequote_runtime_word(const char *word)
     }
     return out.data;
 }
+
+int add_heredoc_entry(struct exec_context *ctx, const t_redir *redir,
+    char *body)
+{
+    struct heredoc_entry *entry;
+
+    entry = (struct heredoc_entry *)malloc(sizeof(*entry));
+    if (entry == NULL)
+        return 1;
+    entry->redir = redir;
+    entry->body = body;
+    entry->next = ctx->heredocs;
+    ctx->heredocs = entry;
+    return 0;
+}
+
+void exec_heredoc_entries_free(struct heredoc_entry *entry)
+{
+    struct heredoc_entry *next;
+
+    while (entry != NULL) {
+        next = entry->next;
+        free(entry->body);
+        free(entry);
+        entry = next;
+    }
+}
+
+const char *exec_find_heredoc_body(const struct exec_context *ctx,
+    const t_redir *redir)
+{
+    struct heredoc_entry *entry;
+
+    entry = ctx->heredocs;
+    while (entry != NULL) {
+        if (entry->redir == redir)
+            return entry->body;
+        entry = entry->next;
+    }
+    return "";
+}


## `feat(heredoc): 구분자별 본문 순차 수집`

diff --git a/include/shell.h b/include/shell.h
index f21371d..024e0c1 100644
--- a/include/shell.h
+++ b/include/shell.h
@@ -24,12 +24,14 @@ typedef struct s_token {
 typedef enum e_redir_type {
     REDIR_IN,
     REDIR_OUT,
-    REDIR_APPEND
+    REDIR_APPEND,
+    REDIR_HEREDOC
 }   t_redir_type;
 
 # define SHELL_REDIR_IN REDIR_IN
 # define SHELL_REDIR_OUT REDIR_OUT
 # define SHELL_REDIR_APPEND REDIR_APPEND
+# define SHELL_REDIR_HEREDOC REDIR_HEREDOC
 
 typedef struct s_redir {
     t_redir_type    type;
diff --git a/src/exec_internal.h b/src/exec_internal.h
index 1718d87..372ba1d 100644
--- a/src/exec_internal.h
+++ b/src/exec_internal.h
@@ -14,6 +14,7 @@ struct exec_context {
     struct heredoc_entry    *heredocs;
 };
 
+int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines);
 void exec_heredoc_entries_free(struct heredoc_entry *entry);
 const char *exec_find_heredoc_body(const struct exec_context *ctx,
         const t_redir *redir);
diff --git a/src/heredoc.c b/src/heredoc.c
index 58355e8..310def9 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -60,7 +60,22 @@ static int sb_push(struct strbuf *buf, char ch)
     return 0;
 }
 
-char *dequote_runtime_word(const char *word)
+static int sb_append(struct strbuf *buf, const char *text)
+{
+    size_t len;
+
+    if (text == NULL)
+        return 0;
+    len = strlen(text);
+    if (sb_reserve(buf, len) != 0)
+        return 1;
+    memcpy(buf->data + buf->len, text, len);
+    buf->len += len;
+    buf->data[buf->len] = '\0';
+    return 0;
+}
+
+static char *dequote_runtime_word(const char *word)
 {
     struct strbuf   out;
     size_t          i;
@@ -86,7 +101,14 @@ char *dequote_runtime_word(const char *word)
     return out.data;
 }
 
-int add_heredoc_entry(struct exec_context *ctx, const t_redir *redir,
+static int append_literal_body_line(struct strbuf *body, const char *line)
+{
+    if (sb_append(body, line) != 0)
+        return 1;
+    return sb_push(body, '\n');
+}
+
+static int add_heredoc_entry(struct exec_context *ctx, const t_redir *redir,
     char *body)
 {
     struct heredoc_entry *entry;
@@ -126,3 +148,74 @@ const char *exec_find_heredoc_body(const struct exec_context *ctx,
     }
     return "";
 }
+
+static int read_heredoc(struct exec_context *ctx, t_redir *redir)
+{
+    struct strbuf   body;
+    char            *delimiter;
+    int             interactive;
+
+    delimiter = dequote_runtime_word(redir->target);
+    if (delimiter == NULL)
+        return 1;
+    free(redir->target);
+    redir->target = delimiter;
+    if (sb_init(&body) != 0)
+        return 1;
+    interactive = isatty(STDIN_FILENO) && isatty(STDERR_FILENO);
+    for (;;) {
+        char *line;
+
+        line = shell_read_line("> ", interactive);
+        if (line == NULL) {
+            fprintf(stderr,
+                "small-shell: warning: here-document delimited by end-of-file (wanted `%s')\n",
+                redir->target);
+            break;
+        }
+        if (strcmp(line, redir->target) == 0) {
+            free(line);
+            break;
+        }
+        if (append_literal_body_line(&body, line) != 0) {
+            free(line);
+            sb_free(&body);
+            return 1;
+        }
+        free(line);
+    }
+    if (add_heredoc_entry(ctx, redir, body.data) != 0) {
+        sb_free(&body);
+        return 1;
+    }
+    return 0;
+}
+
+int exec_prepare_heredocs(struct exec_context *ctx, t_pipeline *pipelines)
+{
+    t_pipeline *pipeline;
+
+    pipeline = pipelines;
+    while (pipeline != NULL) {
+        t_command *command;
+
+        command = pipeline->commands;
+        while (command != NULL) {
+            t_redir *redir;
+
+            redir = command->redirs;
+            while (redir != NULL) {
+                if (redir->type == REDIR_HEREDOC
+                    && read_heredoc(ctx, redir) != 0) {
+                    fprintf(stderr,
+                        "small-shell: heredoc: allocation failure\n");
+                    return 1;
+                }
+                redir = redir->next;
+            }
+            command = command->next;
+        }
+        pipeline = pipeline->next;
+    }
+    return 0;
+}


## `feat(heredoc): 인용 여부에 따라 본문 확장`

diff --git a/src/heredoc.c b/src/heredoc.c
index 310def9..d5b3d0b 100644
--- a/src/heredoc.c
+++ b/src/heredoc.c
@@ -15,6 +15,8 @@ struct strbuf {
     size_t  cap;
 };
 
+char *shell_read_line(const char *prompt, int interactive);
+
 static int sb_init(struct strbuf *buf)
 {
     buf->cap = 64;
@@ -75,6 +77,80 @@ static int sb_append(struct strbuf *buf, const char *text)
     return 0;
 }
 
+static int append_status(struct strbuf *buf, int status)
+{
+    char tmp[32];
+
+    snprintf(tmp, sizeof(tmp), "%d", status);
+    return sb_append(buf, tmp);
+}
+
+static int expand_dollar_at(t_shell *shell, const char *line, size_t *i,
+    struct strbuf *out)
+{
+    size_t pos;
+
+    pos = *i;
+    if (line[pos + 1] == '?') {
+        *i = pos + 2;
+        return append_status(out, shell->last_status);
+    }
+    if (sh_is_name_start((unsigned char)line[pos + 1])) {
+        size_t      start;
+        size_t      end;
+        char        *name;
+        const char  *value;
+        int         result;
+
+        start = pos + 1;
+        end = start + 1;
+        while (sh_is_name_char((unsigned char)line[end]))
+            end++;
+        name = shell_strndup(line + start, end - start);
+        if (name == NULL)
+            return 1;
+        value = env_get(shell->env, name);
+        result = sb_append(out, value);
+        free(name);
+        *i = end;
+        return result;
+    }
+    *i = pos + 1;
+    return sb_push(out, '$');
+}
+
+static int expand_heredoc_body_line(t_shell *shell, const char *line,
+    struct strbuf *out)
+{
+    size_t i;
+
+    i = 0;
+    while (line[i] != '\0') {
+        if (line[i] == '$') {
+            if (expand_dollar_at(shell, line, &i, out) != 0)
+                return 1;
+        } else {
+            if (sb_push(out, line[i]) != 0)
+                return 1;
+            i++;
+        }
+    }
+    return 0;
+}
+
+static int word_has_literal_mark(const char *word)
+{
+    size_t i;
+
+    i = 0;
+    while (word != NULL && word[i] != '\0') {
+        if (word[i] == LITERAL_MARK)
+            return 1;
+        i++;
+    }
+    return 0;
+}
+
 static char *dequote_runtime_word(const char *word)
 {
     struct strbuf   out;
@@ -101,10 +177,15 @@ static char *dequote_runtime_word(const char *word)
     return out.data;
 }
 
-static int append_literal_body_line(struct strbuf *body, const char *line)
+static int append_heredoc_body_line(t_shell *shell, int quoted,
+    struct strbuf *body, const char *line)
 {
-    if (sb_append(body, line) != 0)
+    if (quoted) {
+        if (sb_append(body, line) != 0)
+            return 1;
+    } else if (expand_heredoc_body_line(shell, line, body) != 0) {
         return 1;
+    }
     return sb_push(body, '\n');
 }
 
@@ -153,8 +234,10 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
 {
     struct strbuf   body;
     char            *delimiter;
+    int             quoted;
     int             interactive;
 
+    quoted = word_has_literal_mark(redir->target);
     delimiter = dequote_runtime_word(redir->target);
     if (delimiter == NULL)
         return 1;
@@ -177,7 +260,7 @@ static int read_heredoc(struct exec_context *ctx, t_redir *redir)
             free(line);
             break;
         }
-        if (append_literal_body_line(&body, line) != 0) {
+        if (append_heredoc_body_line(ctx->shell, quoted, &body, line) != 0) {
             free(line);
             sb_free(&body);
             return 1;


