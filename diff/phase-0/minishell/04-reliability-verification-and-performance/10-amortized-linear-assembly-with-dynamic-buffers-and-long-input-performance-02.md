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


## `refactor(expand): 확장 결과를 가변 버퍼로 조립`

diff --git a/src/expand.c b/src/expand.c
index a11b455..39261c4 100644
--- a/src/expand.c
+++ b/src/expand.c
@@ -1,89 +1,90 @@
 #include "shell.h"
+#include "string_builder.h"
 
 #include <stdio.h>
 #include <stdlib.h>
 
 #define LITERAL_MARK '\001'
 
-static char *append_status(char *out, int status)
+static int append_status(t_string_builder *out, int status)
 {
     char buf[32];
 
     snprintf(buf, sizeof(buf), "%d", status);
-    return sh_strjoin_free(out, buf);
-}
-
-static char *append_char(char *out, char c)
-{
-    char buf[2];
-
-    buf[0] = c;
-    buf[1] = '\0';
-    return sh_strjoin_free(out, buf);
+    return string_builder_append_text(out, buf);
 }
 
 char *expand_word(t_shell *shell, const char *word)
 {
-    char    *out;
-    size_t  i;
-    size_t  start;
-    char    *key;
+    t_string_builder    out;
+    size_t              i;
 
-    out = sh_strdup("");
-    if (out == NULL)
+    if (string_builder_init(&out) != 0)
         return NULL;
     i = 0;
     while (word != NULL && word[i] != '\0') {
+        int failed;
+
+        failed = 0;
         if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
-            out = append_char(out, word[i + 1]);
+            failed = string_builder_append_char(&out, word[i + 1]);
             i += 2;
         } else if (word[i] == '$' && word[i + 1] == '?') {
-            out = append_status(out, shell->last_status);
+            failed = append_status(&out, shell->last_status);
             i += 2;
         } else if (word[i] == '$'
             && sh_is_name_start((unsigned char)word[i + 1])) {
+            size_t  start;
+            char    *key;
+
             start = i + 1;
             i = start + 1;
             while (sh_is_name_char((unsigned char)word[i]))
                 i++;
             key = sh_substr(word, start, i - start);
             if (key == NULL) {
-                free(out);
+                string_builder_discard(&out);
                 return NULL;
             }
-            out = sh_strjoin_free(out, env_get(shell->env, key));
+            failed = string_builder_append_text(&out,
+                    env_get(shell->env, key));
             free(key);
         } else {
-            out = append_char(out, word[i]);
+            failed = string_builder_append_char(&out, word[i]);
             i++;
         }
-        if (out == NULL)
+        if (failed != 0) {
+            string_builder_discard(&out);
             return NULL;
+        }
     }
-    return out;
+    return string_builder_take(&out);
 }
 
 static char *dequote_word(const char *word)
 {
-    char    *out;
-    size_t  i;
+    t_string_builder    out;
+    size_t              i;
 
-    out = sh_strdup("");
-    if (out == NULL)
+    if (string_builder_init(&out) != 0)
         return NULL;
     i = 0;
     while (word != NULL && word[i] != '\0') {
+        int failed;
+
         if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
-            out = append_char(out, word[i + 1]);
+            failed = string_builder_append_char(&out, word[i + 1]);
             i += 2;
         } else {
-            out = append_char(out, word[i]);
+            failed = string_builder_append_char(&out, word[i]);
             i++;
         }
-        if (out == NULL)
+        if (failed != 0) {
+            string_builder_discard(&out);
             return NULL;
+        }
     }
-    return out;
+    return string_builder_take(&out);
 }
 
 int shell_dequote_word(const char *word, char **out, char **error)


## `test(performance): 긴 입력 처리 시간 상한 검증`

diff --git a/Makefile b/Makefile
index 4070777..84dbfd7 100644
--- a/Makefile
+++ b/Makefile
@@ -60,6 +60,7 @@ test: $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET)
 	./tests/allocation.sh
 	./tests/lifecycle.sh
 	./$(PARSER_API_TARGET)
+	./tests/performance.sh
 
 clean:
 	rm -f $(TARGET) $(TEST_TARGET) $(PARSER_API_TARGET) $(TIMEOUT_TARGET) \
diff --git a/tests/performance.sh b/tests/performance.sh
new file mode 100755
index 0000000..af87606
--- /dev/null
+++ b/tests/performance.sh
@@ -0,0 +1,38 @@
+#!/bin/sh
+set -eu
+
+ROOT=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+BIN=${SMALL_SHELL_BIN:-"$ROOT/small-shell"}
+TIMEOUT=${SMALL_SHELL_TIMEOUT_BIN:-"$ROOT/tests/timeout-runner"}
+TMP=$(mktemp -d "${TMPDIR:-/tmp}/small-shell-performance.XXXXXX")
+PAYLOAD_SIZE=524288
+
+trap 'rm -rf "$TMP"' EXIT HUP INT TERM
+
+fail()
+{
+    echo "not ok - long input performance" >&2
+    if [ -f "$TMP/long.err" ]; then
+        sed 's/^/stderr: /' "$TMP/long.err" >&2
+    fi
+    exit 1
+}
+
+{
+    printf 'echo '
+    dd if=/dev/zero bs=1024 count=512 2>/dev/null | tr '\000' x
+    printf '\n'
+} >"$TMP/long.in"
+
+set +e
+"$TIMEOUT" 5 "$BIN" <"$TMP/long.in" \
+    >"$TMP/long.out" 2>"$TMP/long.err"
+status=$?
+set -e
+
+[ "$status" -eq 0 ] || fail
+[ ! -s "$TMP/long.err" ] || fail
+output_size=$(wc -c <"$TMP/long.out" | tr -d '[:space:]')
+[ "$output_size" -eq $((PAYLOAD_SIZE + 1)) ] || fail
+
+echo "ok - long input performance"
