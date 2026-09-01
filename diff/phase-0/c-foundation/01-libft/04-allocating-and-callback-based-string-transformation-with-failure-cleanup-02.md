## `test(string): callback 문자열 변환 검증`

diff --git a/tests/test.h b/tests/test.h
index 8c5abba..95b3c50 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -16,6 +16,7 @@ void	test_allocate(void);
 void	test_string_build(void);
 void	test_split(void);
 void	test_itoa(void);
+void	test_string_transform(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index da2bd0b..e5be8b8 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -41,5 +41,6 @@ int	main(void)
 	test_string_build();
 	test_split();
 	test_itoa();
+	test_string_transform();
 	return (test_finish());
 }
diff --git a/tests/test_string_transform.c b/tests/test_string_transform.c
new file mode 100644
index 0000000..1b49288
--- /dev/null
+++ b/tests/test_string_transform.c
@@ -0,0 +1,79 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+#include <string.h>
+
+static unsigned int	g_map_calls;
+static unsigned int	g_iter_calls;
+
+static char	map_character(unsigned int index, char character)
+{
+	CHECK(index == g_map_calls);
+	g_map_calls++;
+	return ((char)(character + (char)index));
+}
+
+static void	iterate_character(unsigned int index, char *character)
+{
+	CHECK(index == g_iter_calls);
+	g_iter_calls++;
+	*character = (char)(*character - (char)index);
+}
+
+static void	erase_character(unsigned int index, char *character)
+{
+	CHECK(index == g_iter_calls);
+	g_iter_calls++;
+	*character = '\0';
+}
+
+static void	check_map(const char *text, const char *expected)
+{
+	char	*mapped;
+
+	g_map_calls = 0;
+	mapped = ft_strmapi(text, map_character);
+	CHECK(mapped != NULL);
+	if (mapped != NULL)
+	{
+		CHECK(strcmp(mapped, expected) == 0);
+		CHECK(mapped != text);
+		CHECK(g_map_calls == strlen(text));
+		free(mapped);
+	}
+}
+
+static void	check_iteration(char *text, const char *expected)
+{
+	size_t	initial_length;
+
+	initial_length = strlen(text);
+	g_iter_calls = 0;
+	ft_striteri(text, iterate_character);
+	CHECK(strcmp(text, expected) == 0);
+	CHECK(g_iter_calls == initial_length);
+}
+
+void	test_string_transform(void)
+{
+	char	first[] = "aceg";
+	char	second[] = "bdfh";
+	char	erased[] = "fixed";
+
+	check_map("", "");
+	check_map("aaaa", "abcd");
+	check_map("0123", "0246");
+	check_iteration(first, "abcd");
+	check_iteration(second, "bcde");
+	g_iter_calls = 0;
+	ft_striteri(erased, erase_character);
+	CHECK(g_iter_calls == 5);
+	CHECK(erased[0] == '\0' && erased[1] == '\0' && erased[2] == '\0');
+	CHECK(erased[3] == '\0' && erased[4] == '\0' && erased[5] == '\0');
+	CHECK(ft_strmapi(NULL, map_character) == NULL);
+	CHECK(ft_strmapi("text", NULL) == NULL);
+	ft_striteri(NULL, iterate_character);
+	ft_striteri(first, NULL);
+}


## `fix(string): callback 순회의 진행 인덱스를 확장`

diff --git a/src/string/ft_string_transform.c b/src/string/ft_string_transform.c
index 3208108..c89fc40 100644
--- a/src/string/ft_string_transform.c
+++ b/src/string/ft_string_transform.c
@@ -7,7 +7,7 @@ char	*ft_strmapi(const char *text,
 {
 	char		*mapped;
 	size_t		length;
-	unsigned int	index;
+	size_t		index;
 
 	if (text == NULL || function == NULL)
 		return (NULL);
@@ -16,9 +16,9 @@ char	*ft_strmapi(const char *text,
 	if (mapped == NULL)
 		return (NULL);
 	index = 0;
-	while ((size_t)index < length)
+	while (index < length)
 	{
-		mapped[index] = function(index, text[index]);
+		mapped[index] = function((unsigned int)index, text[index]);
 		index++;
 	}
 	mapped[index] = '\0';
@@ -28,15 +28,15 @@ char	*ft_strmapi(const char *text,
 void	ft_striteri(char *text, void (*function)(unsigned int, char *))
 {
 	size_t		length;
-	unsigned int	index;
+	size_t		index;
 
 	if (text == NULL || function == NULL)
 		return ;
 	length = ft_strlen(text);
 	index = 0;
-	while ((size_t)index < length)
+	while (index < length)
 	{
-		function(index, text + index);
+		function((unsigned int)index, text + index);
 		index++;
 	}
 }
