# 용량 제한 문자열 탐색

## `feat(string): 문자열 길이 계산과 제한 복사·붙이기 추가`

diff --git a/Makefile b/Makefile
index da70ad4..db3acd9 100644
--- a/Makefile
+++ b/Makefile
@@ -15,7 +15,8 @@ SRC := \
 	src/memory/ft_memory_fill.c \
 	src/memory/ft_memory_copy.c \
 	src/memory/ft_memory_move.c \
-	src/memory/ft_memory_scan.c
+	src/memory/ft_memory_scan.c \
+	src/string/ft_string_bounds.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 8dad255..3bd89bf 100644
--- a/libft.h
+++ b/libft.h
@@ -18,4 +18,8 @@ void	*ft_memmove(void *destination, const void *source, size_t length);
 void	*ft_memchr(const void *memory, int byte, size_t length);
 int		ft_memcmp(const void *left, const void *right, size_t length);
 
+size_t	ft_strlen(const char *text);
+size_t	ft_strlcpy(char *destination, const char *source, size_t capacity);
+size_t	ft_strlcat(char *destination, const char *source, size_t capacity);
+
 #endif
diff --git a/src/string/ft_string_bounds.c b/src/string/ft_string_bounds.c
new file mode 100644
index 0000000..6d91475
--- /dev/null
+++ b/src/string/ft_string_bounds.c
@@ -0,0 +1,53 @@
+#include "libft.h"
+
+size_t	ft_strlen(const char *text)
+{
+	size_t	length;
+
+	length = 0;
+	while (text[length] != '\0')
+		length++;
+	return (length);
+}
+
+size_t	ft_strlcpy(char *destination, const char *source, size_t capacity)
+{
+	size_t	source_length;
+	size_t	index;
+
+	source_length = ft_strlen(source);
+	if (capacity == 0)
+		return (source_length);
+	index = 0;
+	while (source[index] != '\0' && index + 1 < capacity)
+	{
+		destination[index] = source[index];
+		index++;
+	}
+	destination[index] = '\0';
+	return (source_length);
+}
+
+size_t	ft_strlcat(char *destination, const char *source, size_t capacity)
+{
+	size_t	destination_length;
+	size_t	source_length;
+	size_t	index;
+
+	destination_length = 0;
+	while (destination_length < capacity
+		&& destination[destination_length] != '\0')
+		destination_length++;
+	source_length = ft_strlen(source);
+	if (destination_length == capacity)
+		return (capacity + source_length);
+	index = 0;
+	while (source[index] != '\0'
+		&& destination_length + index + 1 < capacity)
+	{
+		destination[destination_length + index] = source[index];
+		index++;
+	}
+	destination[destination_length + index] = '\0';
+	return (destination_length + source_length);
+}


## `test(string): 문자열 길이와 capacity 경계 검증`

diff --git a/tests/test.h b/tests/test.h
index a54a2b4..dffb95d 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -9,6 +9,7 @@ void	test_memory_fill(void);
 void	test_memory_copy(void);
 void	test_memory_move(void);
 void	test_memory_scan(void);
+void	test_string_bounds(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index c247c86..ea015ad 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -34,5 +34,6 @@ int	main(void)
 	test_memory_copy();
 	test_memory_move();
 	test_memory_scan();
+	test_string_bounds();
 	return (test_finish());
 }
diff --git a/tests/test_string_bounds.c b/tests/test_string_bounds.c
new file mode 100644
index 0000000..9a1eb19
--- /dev/null
+++ b/tests/test_string_bounds.c
@@ -0,0 +1,121 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stddef.h>
+#include <string.h>
+
+#define STRING_BUFFER_SIZE 40
+
+static size_t	reference_strlcpy(char *destination, const char *source,
+		size_t capacity)
+{
+	size_t	length;
+	size_t	index;
+
+	length = strlen(source);
+	if (capacity == 0)
+		return (length);
+	index = 0;
+	while (index + 1 < capacity && source[index] != '\0')
+	{
+		destination[index] = source[index];
+		index++;
+	}
+	destination[index] = '\0';
+	return (length);
+}
+
+static size_t	reference_strlcat(char *destination, const char *source,
+		size_t capacity)
+{
+	size_t	destination_length;
+	size_t	source_length;
+	size_t	index;
+
+	destination_length = 0;
+	while (destination_length < capacity
+		&& destination[destination_length] != '\0')
+		destination_length++;
+	source_length = strlen(source);
+	if (destination_length == capacity)
+		return (capacity + source_length);
+	index = 0;
+	while (source[index] != '\0'
+		&& destination_length + index + 1 < capacity)
+	{
+		destination[destination_length + index] = source[index];
+		index++;
+	}
+	destination[destination_length + index] = '\0';
+	return (destination_length + source_length);
+}
+
+static void	seed_string_buffer(char *buffer)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < STRING_BUFFER_SIZE)
+	{
+		buffer[index] = (char)('A' + index % 26);
+		index++;
+	}
+}
+
+static void	check_strlcpy(const char *source, size_t capacity)
+{
+	char	actual[STRING_BUFFER_SIZE];
+	char	expected[STRING_BUFFER_SIZE];
+	size_t	actual_length;
+	size_t	expected_length;
+
+	seed_string_buffer(actual);
+	memcpy(expected, actual, sizeof(actual));
+	actual_length = ft_strlcpy(actual, source, capacity);
+	expected_length = reference_strlcpy(expected, source, capacity);
+	CHECK(actual_length == expected_length);
+	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
+}
+
+static void	check_strlcat(const char *initial, const char *source,
+		size_t capacity)
+{
+	char	actual[STRING_BUFFER_SIZE];
+	char	expected[STRING_BUFFER_SIZE];
+	size_t	initial_length;
+	size_t	actual_length;
+	size_t	expected_length;
+
+	seed_string_buffer(actual);
+	initial_length = strlen(initial);
+	memcpy(actual, initial, initial_length + 1);
+	memcpy(expected, actual, sizeof(actual));
+	actual_length = ft_strlcat(actual, source, capacity);
+	expected_length = reference_strlcat(expected, source, capacity);
+	CHECK(actual_length == expected_length);
+	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
+}
+
+void	test_string_bounds(void)
+{
+	static const char	*texts[] = {"", "a", "hello", "embedded", "0123456789"};
+	size_t			text_index;
+	size_t			capacity;
+
+	text_index = 0;
+	while (text_index < sizeof(texts) / sizeof(texts[0]))
+	{
+		CHECK(ft_strlen(texts[text_index]) == strlen(texts[text_index]));
+		capacity = 0;
+		while (capacity <= 16)
+		{
+			check_strlcpy(texts[text_index], capacity);
+			check_strlcat("", texts[text_index], capacity);
+			check_strlcat("abc", texts[text_index], capacity);
+			check_strlcat("abcdefgh", texts[text_index], capacity);
+			capacity++;
+		}
+		text_index++;
+	}
+}


## `feat(string): 문자의 첫·마지막 위치 검색을 추가`

diff --git a/Makefile b/Makefile
index db3acd9..513f955 100644
--- a/Makefile
+++ b/Makefile
@@ -16,7 +16,8 @@ SRC := \
 	src/memory/ft_memory_copy.c \
 	src/memory/ft_memory_move.c \
 	src/memory/ft_memory_scan.c \
-	src/string/ft_string_bounds.c
+	src/string/ft_string_bounds.c \
+	src/string/ft_string_search.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 3bd89bf..00516e6 100644
--- a/libft.h
+++ b/libft.h
@@ -21,5 +21,7 @@ int		ft_memcmp(const void *left, const void *right, size_t length);
 size_t	ft_strlen(const char *text);
 size_t	ft_strlcpy(char *destination, const char *source, size_t capacity);
 size_t	ft_strlcat(char *destination, const char *source, size_t capacity);
+char	*ft_strchr(const char *text, int character);
+char	*ft_strrchr(const char *text, int character);
 
 #endif
diff --git a/src/string/ft_string_search.c b/src/string/ft_string_search.c
new file mode 100644
index 0000000..1f247b8
--- /dev/null
+++ b/src/string/ft_string_search.c
@@ -0,0 +1,35 @@
+#include "libft.h"
+
+char	*ft_strchr(const char *text, int character)
+{
+	unsigned char	target;
+	size_t		index;
+
+	target = (unsigned char)character;
+	index = 0;
+	while (1)
+	{
+		if ((unsigned char)text[index] == target)
+			return ((char *)(text + index));
+		if (text[index] == '\0')
+			return (NULL);
+		index++;
+	}
+}
+
+char	*ft_strrchr(const char *text, int character)
+{
+	unsigned char	target;
+	char		*match;
+
+	target = (unsigned char)character;
+	match = NULL;
+	while (1)
+	{
+		if ((unsigned char)*text == target)
+			match = (char *)text;
+		if (*text == '\0')
+			return (match);
+		text++;
+	}
+}


## `feat(string): 범위 비교와 부분 문자열 검색을 추가`

diff --git a/libft.h b/libft.h
index 00516e6..6b8f122 100644
--- a/libft.h
+++ b/libft.h
@@ -23,5 +23,7 @@ size_t	ft_strlcpy(char *destination, const char *source, size_t capacity);
 size_t	ft_strlcat(char *destination, const char *source, size_t capacity);
 char	*ft_strchr(const char *text, int character);
 char	*ft_strrchr(const char *text, int character);
+int		ft_strncmp(const char *left, const char *right, size_t length);
+char	*ft_strnstr(const char *haystack, const char *needle, size_t length);
 
 #endif
diff --git a/src/string/ft_string_search.c b/src/string/ft_string_search.c
index 1f247b8..c040977 100644
--- a/src/string/ft_string_search.c
+++ b/src/string/ft_string_search.c
@@ -33,3 +33,43 @@ char	*ft_strrchr(const char *text, int character)
 		text++;
 	}
 }
+
+int	ft_strncmp(const char *left, const char *right, size_t length)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < length)
+	{
+		if ((unsigned char)left[index] != (unsigned char)right[index])
+			return ((int)(unsigned char)left[index]
+				- (int)(unsigned char)right[index]);
+		if (left[index] == '\0')
+			return (0);
+		index++;
+	}
+	return (0);
+}
+
+char	*ft_strnstr(const char *haystack, const char *needle, size_t length)
+{
+	size_t	haystack_index;
+	size_t	needle_index;
+
+	if (*needle == '\0')
+		return ((char *)haystack);
+	haystack_index = 0;
+	while (haystack_index < length && haystack[haystack_index] != '\0')
+	{
+		needle_index = 0;
+		while (needle[needle_index] != '\0'
+			&& needle_index < length - haystack_index
+			&& haystack[haystack_index + needle_index]
+				== needle[needle_index])
+			needle_index++;
+		if (needle[needle_index] == '\0')
+			return ((char *)(haystack + haystack_index));
+		haystack_index++;
+	}
+	return (NULL);
+}


## `test(string): 문자열 검색과 비교 경계 검증`

diff --git a/tests/test.h b/tests/test.h
index dffb95d..ac8f338 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -10,6 +10,7 @@ void	test_memory_copy(void);
 void	test_memory_move(void);
 void	test_memory_scan(void);
 void	test_string_bounds(void);
+void	test_string_search(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index ea015ad..817d6b6 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -35,5 +35,6 @@ int	main(void)
 	test_memory_move();
 	test_memory_scan();
 	test_string_bounds();
+	test_string_search();
 	return (test_finish());
 }
diff --git a/tests/test_string_search.c b/tests/test_string_search.c
new file mode 100644
index 0000000..3efeeee
--- /dev/null
+++ b/tests/test_string_search.c
@@ -0,0 +1,96 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stddef.h>
+#include <string.h>
+
+static int	search_sign(int value)
+{
+	if (value < 0)
+		return (-1);
+	if (value > 0)
+		return (1);
+	return (0);
+}
+
+static char	*reference_strnstr(const char *haystack, const char *needle,
+		size_t length)
+{
+	size_t	haystack_index;
+	size_t	needle_index;
+
+	if (*needle == '\0')
+		return ((char *)haystack);
+	haystack_index = 0;
+	while (haystack_index < length && haystack[haystack_index] != '\0')
+	{
+		needle_index = 0;
+		while (needle[needle_index] != '\0'
+			&& needle_index < length - haystack_index
+			&& haystack[haystack_index + needle_index]
+				== needle[needle_index])
+			needle_index++;
+		if (needle[needle_index] == '\0')
+			return ((char *)(haystack + haystack_index));
+		haystack_index++;
+	}
+	return (NULL);
+}
+
+static void	check_character_search(const char *text, int character)
+{
+	CHECK(ft_strchr(text, character) == strchr(text, character));
+	CHECK(ft_strrchr(text, character) == strrchr(text, character));
+}
+
+static void	check_bounded_compare(const char *left, const char *right)
+{
+	size_t	length;
+
+	length = 0;
+	while (length <= 12)
+	{
+		CHECK(search_sign(ft_strncmp(left, right, length))
+			== search_sign(strncmp(left, right, length)));
+		length++;
+	}
+}
+
+static void	check_substring(const char *haystack, const char *needle)
+{
+	size_t	length;
+
+	length = 0;
+	while (length <= strlen(haystack) + 2)
+	{
+		CHECK(ft_strnstr(haystack, needle, length)
+			== reference_strnstr(haystack, needle, length));
+		length++;
+	}
+}
+
+void	test_string_search(void)
+{
+	check_character_search("", '\0');
+	check_character_search("banana", 'a');
+	check_character_search("banana", 'b');
+	check_character_search("banana", 'n');
+	check_character_search("banana", 'z');
+	check_character_search("banana", '\0');
+	check_character_search("banana", 256);
+	check_bounded_compare("", "");
+	check_bounded_compare("abc", "abc");
+	check_bounded_compare("abc", "abd");
+	check_bounded_compare("abd", "abc");
+	check_bounded_compare("abc", "ab");
+	check_bounded_compare("ab", "abc");
+	check_substring("", "");
+	check_substring("abc", "");
+	check_substring("abc", "a");
+	check_substring("abc", "bc");
+	check_substring("abc", "abcd");
+	check_substring("aaaaab", "aaab");
+	check_substring("prefix middle suffix", "middle");
+	check_substring("prefix middle suffix", "missing");
+}
