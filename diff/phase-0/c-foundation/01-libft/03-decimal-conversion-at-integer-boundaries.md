# 정수 경계의 10진 변환

## `feat(convert): 표현 가능한 10진수 정수 해석`

diff --git a/Makefile b/Makefile
index 513f955..f127bc6 100644
--- a/Makefile
+++ b/Makefile
@@ -17,7 +17,8 @@ SRC := \
 	src/memory/ft_memory_move.c \
 	src/memory/ft_memory_scan.c \
 	src/string/ft_string_bounds.c \
-	src/string/ft_string_search.c
+	src/string/ft_string_search.c \
+	src/convert/ft_atoi.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 6b8f122..17af003 100644
--- a/libft.h
+++ b/libft.h
@@ -25,5 +25,6 @@ char	*ft_strchr(const char *text, int character);
 char	*ft_strrchr(const char *text, int character);
 int		ft_strncmp(const char *left, const char *right, size_t length);
 char	*ft_strnstr(const char *haystack, const char *needle, size_t length);
+int		ft_atoi(const char *text);
 
 #endif
diff --git a/src/convert/ft_atoi.c b/src/convert/ft_atoi.c
new file mode 100644
index 0000000..17c3ffb
--- /dev/null
+++ b/src/convert/ft_atoi.c
@@ -0,0 +1,41 @@
+#include "libft.h"
+
+#include <limits.h>
+
+static int	is_space(char character)
+{
+	return (character == ' ' || character == '\t' || character == '\n'
+		|| character == '\v' || character == '\f' || character == '\r');
+}
+
+int	ft_atoi(const char *text)
+{
+	unsigned int	value;
+	unsigned int	limit;
+	int			sign;
+
+	while (is_space(*text))
+		text++;
+	sign = 1;
+	if (*text == '+' || *text == '-')
+	{
+		if (*text == '-')
+			sign = -1;
+		text++;
+	}
+	limit = INT_MAX;
+	if (sign < 0)
+		limit = (unsigned int)INT_MAX + 1U;
+	value = 0;
+	while (*text >= '0' && *text <= '9')
+	{
+		if (value > (limit - (unsigned int)(*text - '0')) / 10U)
+			value = limit;
+		else
+			value = value * 10U + (unsigned int)(*text - '0');
+		text++;
+	}
+	if (sign < 0 && value == (unsigned int)INT_MAX + 1U)
+		return (INT_MIN);
+	return ((int)value * sign);
+}


## `test(convert): 정수 해석 경계 검증`

diff --git a/tests/test.h b/tests/test.h
index ac8f338..cf8a459 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -11,6 +11,7 @@ void	test_memory_move(void);
 void	test_memory_scan(void);
 void	test_string_bounds(void);
 void	test_string_search(void);
+void	test_atoi(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_atoi.c b/tests/test_atoi.c
new file mode 100644
index 0000000..f8ff7e9
--- /dev/null
+++ b/tests/test_atoi.c
@@ -0,0 +1,25 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+
+void	test_atoi(void)
+{
+	static const char	*cases[] = {
+		"", "+", "-", "0", "+0", "-0", "1", "-1", "42", "-42",
+		"2147483647", "-2147483648", "00000123", "-00000123",
+		"  17", "\t\n\v\f\r 23", " +98words", "-77tail",
+		"words42", "++1", "--1", "+-1", "-+1"
+	};
+	size_t			index;
+
+	index = 0;
+	while (index < sizeof(cases) / sizeof(cases[0]))
+	{
+		CHECK(ft_atoi(cases[index]) == atoi(cases[index]));
+		index++;
+	}
+	CHECK(ft_atoi("999999999999999999999999") == 2147483647);
+	CHECK(ft_atoi("-999999999999999999999999") == (-2147483647 - 1));
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index 817d6b6..866ad34 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -36,5 +36,6 @@ int	main(void)
 	test_memory_scan();
 	test_string_bounds();
 	test_string_search();
+	test_atoi();
 	return (test_finish());
 }


## `feat(convert): 부호 있는 정수의 문자열 변환 구현`

diff --git a/Makefile b/Makefile
index decd930..07a710b 100644
--- a/Makefile
+++ b/Makefile
@@ -21,6 +21,7 @@ SRC := \
 	src/string/ft_string_build.c \
 	src/string/ft_split.c \
 	src/convert/ft_atoi.c \
+	src/convert/ft_itoa.c \
 	src/alloc/ft_allocate.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
diff --git a/libft.h b/libft.h
index a805b00..f34ec5f 100644
--- a/libft.h
+++ b/libft.h
@@ -33,5 +33,6 @@ char	*ft_substr(const char *text, unsigned int start, size_t length);
 char	*ft_strjoin(const char *left, const char *right);
 char	*ft_strtrim(const char *text, const char *set);
 char	**ft_split(const char *text, char delimiter);
+char	*ft_itoa(int number);
 
 #endif
diff --git a/src/convert/ft_itoa.c b/src/convert/ft_itoa.c
new file mode 100644
index 0000000..3dc33df
--- /dev/null
+++ b/src/convert/ft_itoa.c
@@ -0,0 +1,42 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+static size_t	digit_count(unsigned int magnitude)
+{
+	size_t	count;
+
+	count = 1;
+	while (magnitude >= 10U)
+	{
+		magnitude /= 10U;
+		count++;
+	}
+	return (count);
+}
+
+char	*ft_itoa(int number)
+{
+	char		*text;
+	unsigned int	magnitude;
+	size_t		length;
+
+	if (number < 0)
+		magnitude = (unsigned int)(-(number + 1)) + 1U;
+	else
+		magnitude = (unsigned int)number;
+	length = digit_count(magnitude) + (number < 0);
+	text = malloc(length + 1);
+	if (text == NULL)
+		return (NULL);
+	text[length] = '\0';
+	while (length > (size_t)(number < 0))
+	{
+		length--;
+		text[length] = (char)('0' + magnitude % 10U);
+		magnitude /= 10U;
+	}
+	if (number < 0)
+		text[0] = '-';
+	return (text);
+}


## `test(convert): 정수 문자열 변환 검증`

diff --git a/tests/test.h b/tests/test.h
index 0676755..8c5abba 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -15,6 +15,7 @@ void	test_atoi(void);
 void	test_allocate(void);
 void	test_string_build(void);
 void	test_split(void);
+void	test_itoa(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_itoa.c b/tests/test_itoa.c
new file mode 100644
index 0000000..a21cf89
--- /dev/null
+++ b/tests/test_itoa.c
@@ -0,0 +1,36 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <limits.h>
+#include <stdlib.h>
+#include <string.h>
+
+static void	check_itoa(int number, const char *expected)
+{
+	char	*actual;
+
+	actual = ft_itoa(number);
+	CHECK(actual != NULL);
+	if (actual != NULL)
+	{
+		CHECK(strcmp(actual, expected) == 0);
+		free(actual);
+	}
+}
+
+void	test_itoa(void)
+{
+	check_itoa(INT_MIN, "-2147483648");
+	check_itoa(-100, "-100");
+	check_itoa(-10, "-10");
+	check_itoa(-9, "-9");
+	check_itoa(-1, "-1");
+	check_itoa(0, "0");
+	check_itoa(1, "1");
+	check_itoa(9, "9");
+	check_itoa(10, "10");
+	check_itoa(99, "99");
+	check_itoa(100, "100");
+	check_itoa(INT_MAX, "2147483647");
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index 791d931..da2bd0b 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -40,5 +40,6 @@ int	main(void)
 	test_allocate();
 	test_string_build();
 	test_split();
+	test_itoa();
 	return (test_finish());
 }
