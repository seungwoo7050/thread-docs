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
