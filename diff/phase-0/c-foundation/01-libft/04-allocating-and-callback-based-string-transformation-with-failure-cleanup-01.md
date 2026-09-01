# 할당·콜백 기반 문자열 변환과 실패 정리

## `feat(alloc): 0 초기화 메모리와 문자열 복제 추가`

diff --git a/Makefile b/Makefile
index f127bc6..3d7fa04 100644
--- a/Makefile
+++ b/Makefile
@@ -18,7 +18,8 @@ SRC := \
 	src/memory/ft_memory_scan.c \
 	src/string/ft_string_bounds.c \
 	src/string/ft_string_search.c \
-	src/convert/ft_atoi.c
+	src/convert/ft_atoi.c \
+	src/alloc/ft_allocate.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 17af003..41e5ae7 100644
--- a/libft.h
+++ b/libft.h
@@ -27,4 +27,7 @@ int		ft_strncmp(const char *left, const char *right, size_t length);
 char	*ft_strnstr(const char *haystack, const char *needle, size_t length);
 int		ft_atoi(const char *text);
 
+void	*ft_calloc(size_t count, size_t size);
+char	*ft_strdup(const char *text);
+
 #endif
diff --git a/src/alloc/ft_allocate.c b/src/alloc/ft_allocate.c
new file mode 100644
index 0000000..25cbbb6
--- /dev/null
+++ b/src/alloc/ft_allocate.c
@@ -0,0 +1,37 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+void	*ft_calloc(size_t count, size_t size)
+{
+	size_t	allocation_size;
+	void	*allocation;
+
+	if (count != 0 && size > (size_t)-1 / count)
+		return (NULL);
+	allocation_size = count * size;
+	if (allocation_size == 0)
+		allocation_size = 1;
+	allocation = malloc(allocation_size);
+	if (allocation == NULL)
+		return (NULL);
+	ft_bzero(allocation, allocation_size);
+	return (allocation);
+}
+
+char	*ft_strdup(const char *text)
+{
+	char	*duplicate;
+	size_t	length;
+
+	if (text == NULL)
+		return (NULL);
+	length = ft_strlen(text);
+	if (length == (size_t)-1)
+		return (NULL);
+	duplicate = malloc(length + 1);
+	if (duplicate == NULL)
+		return (NULL);
+	ft_memcpy(duplicate, text, length + 1);
+	return (duplicate);
+}


## `feat(string): 부분 문자열 생성을 구현`

diff --git a/Makefile b/Makefile
index 3d7fa04..3e4b360 100644
--- a/Makefile
+++ b/Makefile
@@ -18,6 +18,7 @@ SRC := \
 	src/memory/ft_memory_scan.c \
 	src/string/ft_string_bounds.c \
 	src/string/ft_string_search.c \
+	src/string/ft_string_build.c \
 	src/convert/ft_atoi.c \
 	src/alloc/ft_allocate.c
 OBJ_DIR := build/obj
diff --git a/libft.h b/libft.h
index 41e5ae7..a57e748 100644
--- a/libft.h
+++ b/libft.h
@@ -29,5 +29,6 @@ int		ft_atoi(const char *text);
 
 void	*ft_calloc(size_t count, size_t size);
 char	*ft_strdup(const char *text);
+char	*ft_substr(const char *text, unsigned int start, size_t length);
 
 #endif
diff --git a/src/string/ft_string_build.c b/src/string/ft_string_build.c
new file mode 100644
index 0000000..13d8175
--- /dev/null
+++ b/src/string/ft_string_build.c
@@ -0,0 +1,23 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+char	*ft_substr(const char *text, unsigned int start, size_t length)
+{
+	char	*substring;
+	size_t	text_length;
+
+	if (text == NULL)
+		return (NULL);
+	text_length = ft_strlen(text);
+	if ((size_t)start >= text_length)
+		return (ft_strdup(""));
+	if (length > text_length - (size_t)start)
+		length = text_length - (size_t)start;
+	substring = malloc(length + 1);
+	if (substring == NULL)
+		return (NULL);
+	ft_memcpy(substring, text + start, length);
+	substring[length] = '\0';
+	return (substring);
+}


## `feat(string): 문자열 결합을 구현`

diff --git a/libft.h b/libft.h
index a57e748..269231f 100644
--- a/libft.h
+++ b/libft.h
@@ -30,5 +30,6 @@ int		ft_atoi(const char *text);
 void	*ft_calloc(size_t count, size_t size);
 char	*ft_strdup(const char *text);
 char	*ft_substr(const char *text, unsigned int start, size_t length);
+char	*ft_strjoin(const char *left, const char *right);
 
 #endif
diff --git a/src/string/ft_string_build.c b/src/string/ft_string_build.c
index 13d8175..e47fca3 100644
--- a/src/string/ft_string_build.c
+++ b/src/string/ft_string_build.c
@@ -21,3 +21,24 @@ char	*ft_substr(const char *text, unsigned int start, size_t length)
 	substring[length] = '\0';
 	return (substring);
 }
+
+char	*ft_strjoin(const char *left, const char *right)
+{
+	char	*joined;
+	size_t	left_length;
+	size_t	right_length;
+
+	if (left == NULL || right == NULL)
+		return (NULL);
+	left_length = ft_strlen(left);
+	right_length = ft_strlen(right);
+	if (right_length == (size_t)-1
+		|| left_length > (size_t)-2 - right_length)
+		return (NULL);
+	joined = malloc(left_length + right_length + 1);
+	if (joined == NULL)
+		return (NULL);
+	ft_memcpy(joined, left, left_length);
+	ft_memcpy(joined + left_length, right, right_length + 1);
+	return (joined);
+}


## `feat(string): 양끝 문자 집합 제거를 구현`

diff --git a/libft.h b/libft.h
index 269231f..74eb5e5 100644
--- a/libft.h
+++ b/libft.h
@@ -31,5 +31,6 @@ void	*ft_calloc(size_t count, size_t size);
 char	*ft_strdup(const char *text);
 char	*ft_substr(const char *text, unsigned int start, size_t length);
 char	*ft_strjoin(const char *left, const char *right);
+char	*ft_strtrim(const char *text, const char *set);
 
 #endif
diff --git a/src/string/ft_string_build.c b/src/string/ft_string_build.c
index e47fca3..c0ef865 100644
--- a/src/string/ft_string_build.c
+++ b/src/string/ft_string_build.c
@@ -42,3 +42,36 @@ char	*ft_strjoin(const char *left, const char *right)
 	ft_memcpy(joined + left_length, right, right_length + 1);
 	return (joined);
 }
+
+static int	is_in_set(char character, const char *set)
+{
+	while (*set != '\0')
+	{
+		if (*set == character)
+			return (1);
+		set++;
+	}
+	return (0);
+}
+
+char	*ft_strtrim(const char *text, const char *set)
+{
+	char	*trimmed;
+	size_t	start;
+	size_t	end;
+
+	if (text == NULL || set == NULL)
+		return (NULL);
+	start = 0;
+	while (text[start] != '\0' && is_in_set(text[start], set))
+		start++;
+	end = ft_strlen(text);
+	while (end > start && is_in_set(text[end - 1], set))
+		end--;
+	trimmed = malloc(end - start + 1);
+	if (trimmed == NULL)
+		return (NULL);
+	ft_memcpy(trimmed, text + start, end - start);
+	trimmed[end - start] = '\0';
+	return (trimmed);
+}


## `test(string): 문자열 생성과 소유권 검증`

diff --git a/tests/test.h b/tests/test.h
index b5ad6aa..94fe8f9 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -13,6 +13,7 @@ void	test_string_bounds(void);
 void	test_string_search(void);
 void	test_atoi(void);
 void	test_allocate(void);
+void	test_string_build(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index 61d84a3..40a8f08 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -38,5 +38,6 @@ int	main(void)
 	test_string_search();
 	test_atoi();
 	test_allocate();
+	test_string_build();
 	return (test_finish());
 }
diff --git a/tests/test_string_build.c b/tests/test_string_build.c
new file mode 100644
index 0000000..e586275
--- /dev/null
+++ b/tests/test_string_build.c
@@ -0,0 +1,76 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+#include <string.h>
+
+static void	check_substr(const char *text, unsigned int start, size_t length,
+		const char *expected)
+{
+	char	*substring;
+
+	substring = ft_substr(text, start, length);
+	CHECK(substring != NULL);
+	if (substring != NULL)
+	{
+		CHECK(strcmp(substring, expected) == 0);
+		CHECK(substring != text);
+		free(substring);
+	}
+}
+
+static void	check_join(const char *left, const char *right,
+		const char *expected)
+{
+	char	*joined;
+
+	joined = ft_strjoin(left, right);
+	CHECK(joined != NULL);
+	if (joined != NULL)
+	{
+		CHECK(strcmp(joined, expected) == 0);
+		CHECK(joined != left && joined != right);
+		free(joined);
+	}
+}
+
+static void	check_trim(const char *text, const char *set,
+		const char *expected)
+{
+	char	*trimmed;
+
+	trimmed = ft_strtrim(text, set);
+	CHECK(trimmed != NULL);
+	if (trimmed != NULL)
+	{
+		CHECK(strcmp(trimmed, expected) == 0);
+		CHECK(trimmed != text);
+		free(trimmed);
+	}
+}
+
+void	test_string_build(void)
+{
+	check_substr("", 0, 0, "");
+	check_substr("foundation", 0, 4, "foun");
+	check_substr("foundation", 4, 99, "dation");
+	check_substr("foundation", 10, 4, "");
+	check_substr("foundation", 99, 4, "");
+	check_substr("foundation", 3, 0, "");
+	CHECK(ft_substr(NULL, 0, 1) == NULL);
+	check_join("", "", "");
+	check_join("left", "", "left");
+	check_join("", "right", "right");
+	check_join("left", "right", "leftright");
+	CHECK(ft_strjoin(NULL, "right") == NULL);
+	CHECK(ft_strjoin("left", NULL) == NULL);
+	check_trim("", "", "");
+	check_trim("hello", "", "hello");
+	check_trim("  hello  ", " ", "hello");
+	check_trim("abbahelloabba", "ab", "hello");
+	check_trim("aaaa", "a", "");
+	check_trim("abc", "xyz", "abc");
+	CHECK(ft_strtrim(NULL, "x") == NULL);
+	CHECK(ft_strtrim("x", NULL) == NULL);
+}


## `feat(string): 실패 시 정리되는 문자열 분리 구현`

diff --git a/Makefile b/Makefile
index 3e4b360..decd930 100644
--- a/Makefile
+++ b/Makefile
@@ -19,6 +19,7 @@ SRC := \
 	src/string/ft_string_bounds.c \
 	src/string/ft_string_search.c \
 	src/string/ft_string_build.c \
+	src/string/ft_split.c \
 	src/convert/ft_atoi.c \
 	src/alloc/ft_allocate.c
 OBJ_DIR := build/obj
diff --git a/libft.h b/libft.h
index 74eb5e5..a805b00 100644
--- a/libft.h
+++ b/libft.h
@@ -32,5 +32,6 @@ char	*ft_strdup(const char *text);
 char	*ft_substr(const char *text, unsigned int start, size_t length);
 char	*ft_strjoin(const char *left, const char *right);
 char	*ft_strtrim(const char *text, const char *set);
+char	**ft_split(const char *text, char delimiter);
 
 #endif
diff --git a/src/string/ft_split.c b/src/string/ft_split.c
new file mode 100644
index 0000000..9e57e18
--- /dev/null
+++ b/src/string/ft_split.c
@@ -0,0 +1,76 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+static size_t	count_fields(const char *text, char delimiter)
+{
+	size_t	count;
+	size_t	index;
+
+	count = 0;
+	index = 0;
+	while (text[index] != '\0')
+	{
+		while (text[index] == delimiter && text[index] != '\0')
+			index++;
+		if (text[index] != '\0')
+			count++;
+		while (text[index] != delimiter && text[index] != '\0')
+			index++;
+	}
+	return (count);
+}
+
+static char	*copy_field(const char *start, size_t length)
+{
+	char	*field;
+
+	field = malloc(length + 1);
+	if (field == NULL)
+		return (NULL);
+	ft_memcpy(field, start, length);
+	field[length] = '\0';
+	return (field);
+}
+
+static void	free_fields(char **fields, size_t count)
+{
+	while (count > 0)
+	{
+		count--;
+		free(fields[count]);
+	}
+	free(fields);
+}
+
+char	**ft_split(const char *text, char delimiter)
+{
+	char	**fields;
+	size_t	text_index;
+	size_t	field_index;
+	size_t	start;
+
+	if (text == NULL)
+		return (NULL);
+	fields = ft_calloc(count_fields(text, delimiter) + 1, sizeof(char *));
+	if (fields == NULL)
+		return (NULL);
+	text_index = 0;
+	field_index = 0;
+	while (text[text_index] != '\0')
+	{
+		while (text[text_index] == delimiter && text[text_index] != '\0')
+			text_index++;
+		start = text_index;
+		while (text[text_index] != delimiter && text[text_index] != '\0')
+			text_index++;
+		if (text_index > start)
+		{
+			fields[field_index] = copy_field(text + start, text_index - start);
+			if (fields[field_index] == NULL)
+				return (free_fields(fields, field_index), NULL);
+			field_index++;
+		}
+	}
+	return (fields);
+}


## `refactor(string): 결합 문자열의 할당 한계를 명시`

diff --git a/src/string/ft_string_build.c b/src/string/ft_string_build.c
index c0ef865..8807d5f 100644
--- a/src/string/ft_string_build.c
+++ b/src/string/ft_string_build.c
@@ -33,7 +33,7 @@ char	*ft_strjoin(const char *left, const char *right)
 	left_length = ft_strlen(left);
 	right_length = ft_strlen(right);
 	if (right_length == (size_t)-1
-		|| left_length > (size_t)-2 - right_length)
+		|| left_length > (size_t)-1 - right_length - 1)
 		return (NULL);
 	joined = malloc(left_length + right_length + 1);
 	if (joined == NULL)


## `test(alloc): 할당 실패와 rollback을 검증`

diff --git a/Makefile b/Makefile
index 85b5697..0728d56 100644
--- a/Makefile
+++ b/Makefile
@@ -37,6 +37,13 @@ DEP := $(OBJ:.o=.d)
 TEST_BIN := tests/bin/test_libft
 TEST_SRC := $(wildcard tests/test_*.c)
 
+FAIL_OBJ_DIR := build/failure
+FAIL_OBJ := $(SRC:%.c=$(FAIL_OBJ_DIR)/%.o)
+FAIL_DEP := $(FAIL_OBJ:.o=.d)
+FAIL_BIN := tests/bin/test_failure
+FAIL_TEST_SRC := tests/failure/test_failure.c tests/support/fail_alloc.c
+FAIL_DEFINES := -Dmalloc=test_malloc -Dfree=test_free
+
 WRITE_OBJ_DIR := build/write-failure
 WRITE_OUTPUT_OBJ := $(WRITE_OBJ_DIR)/ft_fd_output.o
 WRITE_DEP := $(WRITE_OUTPUT_OBJ:.o=.d)
@@ -45,7 +52,7 @@ WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
 	tests/support/fail_write.c
 WRITE_DEFINES := -Dwrite=test_write
 
-.PHONY: all bonus clean fclean re test write-failure-test
+.PHONY: all bonus clean fclean re test failure-test write-failure-test
 
 all: $(NAME)
 
@@ -67,6 +74,18 @@ $(TEST_BIN): $(NAME) $(TEST_SRC) tests/test.h
 test: $(TEST_BIN)
 	./$(TEST_BIN)
 
+$(FAIL_OBJ_DIR)/%.o: %.c libft.h tests/support/fail_alloc.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(FAIL_DEFINES) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(FAIL_BIN): $(FAIL_OBJ) $(FAIL_TEST_SRC) tests/support/fail_alloc.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(FAIL_TEST_SRC) $(FAIL_OBJ) -o $@
+
+failure-test: $(FAIL_BIN)
+	./$(FAIL_BIN)
+
 $(WRITE_OUTPUT_OBJ): src/io/ft_fd_output.c libft.h \
 		tests/support/fail_write.h
 	@$(MKDIR) $(dir $@)
@@ -90,4 +109,4 @@ fclean: clean
 
 re: fclean all
 
--include $(DEP) $(WRITE_DEP)
+-include $(DEP) $(FAIL_DEP) $(WRITE_DEP)
diff --git a/tests/failure/test_failure.c b/tests/failure/test_failure.c
new file mode 100644
index 0000000..7313d9e
--- /dev/null
+++ b/tests/failure/test_failure.c
@@ -0,0 +1,168 @@
+#include "libft.h"
+#include "tests/support/fail_alloc.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+
+static int	g_checks;
+static int	g_failures;
+static int	g_content_deletes;
+
+#define VERIFY(expression) verify((expression), #expression, __LINE__)
+
+static void	verify(int condition, const char *expression, int line)
+{
+	g_checks++;
+	if (!condition)
+	{
+		g_failures++;
+		fprintf(stderr, "failure test line %d: %s\n", line, expression);
+	}
+}
+
+static char	identity_character(unsigned int index, char character)
+{
+	(void)index;
+	return (character);
+}
+
+static void	check_single_allocation_failures(void)
+{
+	t_list	*node;
+
+	test_allocator_reset(1);
+	VERIFY(ft_calloc(4, 8) == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_strdup("owned") == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_substr("owned", 1, 3) == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_strjoin("left", "right") == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_strtrim("  text  ", " ") == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_itoa(-42) == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	VERIFY(ft_strmapi("map", identity_character) == NULL);
+	VERIFY(test_allocator_live() == 0);
+	test_allocator_reset(1);
+	node = ft_lstnew((void *)"content");
+	VERIFY(node == NULL);
+	VERIFY(test_allocator_live() == 0);
+}
+
+static void	free_failure_split(char **fields)
+{
+	size_t	index;
+
+	index = 0;
+	while (fields[index] != NULL)
+	{
+		test_free(fields[index]);
+		index++;
+	}
+	test_free(fields);
+}
+
+static void	check_split_failures(void)
+{
+	char	**fields;
+	size_t	allocation_count;
+	size_t	failure_index;
+
+	test_allocator_reset(0);
+	fields = ft_split("alpha,beta,gamma,delta", ',');
+	VERIFY(fields != NULL);
+	allocation_count = test_allocator_attempts();
+	if (fields != NULL)
+		free_failure_split(fields);
+	VERIFY(allocation_count == 5);
+	VERIFY(test_allocator_live() == 0);
+	failure_index = 1;
+	while (failure_index <= allocation_count)
+	{
+		test_allocator_reset(failure_index);
+		fields = ft_split("alpha,beta,gamma,delta", ',');
+		VERIFY(fields == NULL);
+		VERIFY(test_allocator_live() == 0);
+		VERIFY(test_allocator_invalid_frees() == 0);
+		failure_index++;
+	}
+}
+
+static void	*map_integer(void *content)
+{
+	int	*mapped;
+
+	mapped = malloc(sizeof(int));
+	if (mapped != NULL)
+		*mapped = *(int *)content + 10;
+	return (mapped);
+}
+
+static void	delete_map_content(void *content)
+{
+	g_content_deletes++;
+	free(content);
+}
+
+static void	check_list_map_failures(void)
+{
+	int		values[3];
+	t_list		source[3];
+	t_list		*mapped;
+	size_t		failure_index;
+
+	values[0] = 1;
+	values[1] = 2;
+	values[2] = 3;
+	source[0].content = &values[0];
+	source[0].next = &source[1];
+	source[1].content = &values[1];
+	source[1].next = &source[2];
+	source[2].content = &values[2];
+	source[2].next = NULL;
+	test_allocator_reset(0);
+	g_content_deletes = 0;
+	mapped = ft_lstmap(source, map_integer, delete_map_content);
+	VERIFY(mapped != NULL);
+	VERIFY(test_allocator_attempts() == 3);
+	if (mapped != NULL)
+		ft_lstclear(&mapped, delete_map_content);
+	VERIFY(g_content_deletes == 3);
+	VERIFY(test_allocator_live() == 0);
+	failure_index = 1;
+	while (failure_index <= 3)
+	{
+		test_allocator_reset(failure_index);
+		g_content_deletes = 0;
+		mapped = ft_lstmap(source, map_integer, delete_map_content);
+		VERIFY(mapped == NULL);
+		VERIFY(g_content_deletes == (int)failure_index);
+		VERIFY(test_allocator_live() == 0);
+		VERIFY(test_allocator_invalid_frees() == 0);
+		VERIFY(values[0] == 1 && values[1] == 2 && values[2] == 3);
+		failure_index++;
+	}
+}
+
+int	main(void)
+{
+	check_single_allocation_failures();
+	check_split_failures();
+	check_list_map_failures();
+	if (g_failures != 0)
+	{
+		fprintf(stderr, "%d of %d failure checks failed\n",
+			g_failures, g_checks);
+		return (1);
+	}
+	printf("%d allocation failure checks passed\n", g_checks);
+	return (0);
+}
diff --git a/tests/support/fail_alloc.c b/tests/support/fail_alloc.c
new file mode 100644
index 0000000..7555344
--- /dev/null
+++ b/tests/support/fail_alloc.c
@@ -0,0 +1,85 @@
+#include "tests/support/fail_alloc.h"
+
+#include <stdlib.h>
+
+#define TRACKED_LIMIT 4096
+
+static void	*g_allocations[TRACKED_LIMIT];
+static size_t	g_attempts;
+static size_t	g_failure_index;
+static size_t	g_live;
+static size_t	g_invalid_frees;
+
+void	test_allocator_reset(size_t failure_index)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < TRACKED_LIMIT)
+	{
+		g_allocations[index] = NULL;
+		index++;
+	}
+	g_attempts = 0;
+	g_failure_index = failure_index;
+	g_live = 0;
+	g_invalid_frees = 0;
+}
+
+void	*test_malloc(size_t size)
+{
+	void	*allocation;
+	size_t	index;
+
+	g_attempts++;
+	if (g_failure_index != 0 && g_attempts == g_failure_index)
+		return (NULL);
+	allocation = malloc(size);
+	if (allocation == NULL)
+		return (NULL);
+	index = 0;
+	while (index < TRACKED_LIMIT && g_allocations[index] != NULL)
+		index++;
+	if (index == TRACKED_LIMIT)
+	{
+		free(allocation);
+		return (NULL);
+	}
+	g_allocations[index] = allocation;
+	g_live++;
+	return (allocation);
+}
+
+void	test_free(void *allocation)
+{
+	size_t	index;
+
+	if (allocation == NULL)
+		return ;
+	index = 0;
+	while (index < TRACKED_LIMIT && g_allocations[index] != allocation)
+		index++;
+	if (index == TRACKED_LIMIT)
+	{
+		g_invalid_frees++;
+		return ;
+	}
+	g_allocations[index] = NULL;
+	g_live--;
+	free(allocation);
+}
+
+size_t	test_allocator_attempts(void)
+{
+	return (g_attempts);
+}
+
+size_t	test_allocator_live(void)
+{
+	return (g_live);
+}
+
+size_t	test_allocator_invalid_frees(void)
+{
+	return (g_invalid_frees);
+}
diff --git a/tests/support/fail_alloc.h b/tests/support/fail_alloc.h
new file mode 100644
index 0000000..e8022f3
--- /dev/null
+++ b/tests/support/fail_alloc.h
@@ -0,0 +1,13 @@
+#ifndef FAIL_ALLOC_H
+# define FAIL_ALLOC_H
+
+# include <stddef.h>
+
+void	*test_malloc(size_t size);
+void	test_free(void *allocation);
+void	test_allocator_reset(size_t failure_index);
+size_t	test_allocator_attempts(void);
+size_t	test_allocator_live(void);
+size_t	test_allocator_invalid_frees(void);
+
+#endif


## `feat(string): 인덱스를 사용하는 문자열 변환 추가`

diff --git a/Makefile b/Makefile
index 07a710b..35d5892 100644
--- a/Makefile
+++ b/Makefile
@@ -20,6 +20,7 @@ SRC := \
 	src/string/ft_string_search.c \
 	src/string/ft_string_build.c \
 	src/string/ft_split.c \
+	src/string/ft_string_transform.c \
 	src/convert/ft_atoi.c \
 	src/convert/ft_itoa.c \
 	src/alloc/ft_allocate.c
diff --git a/libft.h b/libft.h
index f34ec5f..a73b9d8 100644
--- a/libft.h
+++ b/libft.h
@@ -34,5 +34,7 @@ char	*ft_strjoin(const char *left, const char *right);
 char	*ft_strtrim(const char *text, const char *set);
 char	**ft_split(const char *text, char delimiter);
 char	*ft_itoa(int number);
+char	*ft_strmapi(const char *text, char (*function)(unsigned int, char));
+void	ft_striteri(char *text, void (*function)(unsigned int, char *));
 
 #endif
diff --git a/src/string/ft_string_transform.c b/src/string/ft_string_transform.c
new file mode 100644
index 0000000..3208108
--- /dev/null
+++ b/src/string/ft_string_transform.c
@@ -0,0 +1,42 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+char	*ft_strmapi(const char *text,
+		char (*function)(unsigned int, char))
+{
+	char		*mapped;
+	size_t		length;
+	unsigned int	index;
+
+	if (text == NULL || function == NULL)
+		return (NULL);
+	length = ft_strlen(text);
+	mapped = malloc(length + 1);
+	if (mapped == NULL)
+		return (NULL);
+	index = 0;
+	while ((size_t)index < length)
+	{
+		mapped[index] = function(index, text[index]);
+		index++;
+	}
+	mapped[index] = '\0';
+	return (mapped);
+}
+
+void	ft_striteri(char *text, void (*function)(unsigned int, char *))
+{
+	size_t		length;
+	unsigned int	index;
+
+	if (text == NULL || function == NULL)
+		return ;
+	length = ft_strlen(text);
+	index = 0;
+	while ((size_t)index < length)
+	{
+		function(index, text + index);
+		index++;
+	}
+}


