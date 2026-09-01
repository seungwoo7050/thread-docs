# 구간 버퍼와 선형 스캔 비용

## `feat(reader): 파일 끝의 마지막 줄 반환`

diff --git a/get_next_line.c b/get_next_line.c
new file mode 100644
index 0000000..7a9c123
--- /dev/null
+++ b/get_next_line.c
@@ -0,0 +1,130 @@
+#include "get_next_line.h"
+
+#include <stdlib.h>
+#include <unistd.h>
+
+typedef struct s_reader
+{
+	int			fd;
+	char		*bytes;
+	size_t		length;
+	size_t		capacity;
+}t_reader;
+
+static t_reader	g_reader = {-1, NULL, 0, 0};
+
+static void	copy_bytes(char *destination, const char *source, size_t length)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < length)
+	{
+		destination[index] = source[index];
+		index++;
+	}
+}
+
+static void	reset_reader(void)
+{
+	free(g_reader.bytes);
+	g_reader.fd = -1;
+	g_reader.bytes = NULL;
+	g_reader.length = 0;
+	g_reader.capacity = 0;
+}
+
+static int	reserve_bytes(size_t required)
+{
+	char	*allocation;
+	size_t	capacity;
+
+	if (required <= g_reader.capacity)
+		return (1);
+	capacity = g_reader.capacity;
+	if (capacity == 0)
+		capacity = 1;
+	while (capacity < required)
+	{
+		if (capacity > (size_t)-1 / 2)
+			capacity = required;
+		else
+			capacity *= 2;
+		if (capacity < required && capacity == (size_t)-1)
+			return (0);
+	}
+	allocation = malloc(capacity);
+	if (allocation == NULL)
+		return (0);
+	copy_bytes(allocation, g_reader.bytes, g_reader.length);
+	free(g_reader.bytes);
+	g_reader.bytes = allocation;
+	g_reader.capacity = capacity;
+	return (1);
+}
+
+static int	append_bytes(const char *bytes, size_t length)
+{
+	size_t	required;
+
+	if (length > (size_t)-1 - g_reader.length - 1)
+		return (0);
+	required = g_reader.length + length + 1;
+	if (!reserve_bytes(required))
+		return (0);
+	copy_bytes(g_reader.bytes + g_reader.length, bytes, length);
+	g_reader.length += length;
+	g_reader.bytes[g_reader.length] = '\0';
+	return (1);
+}
+
+static char	*release_final_line(void)
+{
+	char	*line;
+
+	line = g_reader.bytes;
+	g_reader.fd = -1;
+	g_reader.bytes = NULL;
+	g_reader.length = 0;
+	g_reader.capacity = 0;
+	return (line);
+}
+
+char	*get_next_line(int fd)
+{
+	char	buffer[BUFFER_SIZE];
+	ssize_t	read_size;
+
+	if (fd < 0 || read(fd, buffer, 0) < 0)
+	{
+		if (fd == g_reader.fd)
+			reset_reader();
+		return (NULL);
+	}
+	if (g_reader.fd != fd)
+	{
+		reset_reader();
+		g_reader.fd = fd;
+	}
+	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
+	while (read_size > 0)
+	{
+		if (!append_bytes(buffer, (size_t)read_size))
+		{
+			reset_reader();
+			return (NULL);
+		}
+		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
+	}
+	if (read_size < 0)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	if (g_reader.length == 0)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	return (release_final_line());
+}


## `refactor(buffer): 읽지 않은 입력을 구간으로 표현`

diff --git a/get_next_line.c b/get_next_line.c
index 7a9c123..f951879 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -7,11 +7,12 @@ typedef struct s_reader
 {
 	int			fd;
 	char		*bytes;
-	size_t		length;
+	size_t		begin;
+	size_t		end;
 	size_t		capacity;
 }t_reader;
 
-static t_reader	g_reader = {-1, NULL, 0, 0};
+static t_reader	g_reader = {-1, NULL, 0, 0, 0};
 
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
@@ -30,15 +31,45 @@ static void	reset_reader(void)
 	free(g_reader.bytes);
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
-	g_reader.length = 0;
+	g_reader.begin = 0;
+	g_reader.end = 0;
 	g_reader.capacity = 0;
 }
 
-static int	reserve_bytes(size_t required)
+static size_t	unread_length(void)
+{
+	return (g_reader.end - g_reader.begin);
+}
+
+static void	compact_bytes(void)
+{
+	size_t	length;
+
+	length = unread_length();
+	copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
+	g_reader.begin = 0;
+	g_reader.end = length;
+	g_reader.bytes[g_reader.end] = '\0';
+}
+
+static int	reserve_bytes(size_t appended)
 {
 	char	*allocation;
 	size_t	capacity;
+	size_t	required;
+	size_t	length;
 
+	length = unread_length();
+	if (appended > (size_t)-1 - length - 1)
+		return (0);
+	required = length + appended + 1;
+	if (g_reader.capacity - g_reader.end >= appended + 1)
+		return (1);
+	if (g_reader.begin > 0 && required <= g_reader.capacity)
+	{
+		compact_bytes();
+		return (1);
+	}
 	if (required <= g_reader.capacity)
 		return (1);
 	capacity = g_reader.capacity;
@@ -56,36 +87,41 @@ static int	reserve_bytes(size_t required)
 	allocation = malloc(capacity);
 	if (allocation == NULL)
 		return (0);
-	copy_bytes(allocation, g_reader.bytes, g_reader.length);
+	if (length > 0)
+		copy_bytes(allocation, g_reader.bytes + g_reader.begin, length);
 	free(g_reader.bytes);
 	g_reader.bytes = allocation;
+	g_reader.begin = 0;
+	g_reader.end = length;
 	g_reader.capacity = capacity;
+	g_reader.bytes[g_reader.end] = '\0';
 	return (1);
 }
 
 static int	append_bytes(const char *bytes, size_t length)
 {
-	size_t	required;
-
-	if (length > (size_t)-1 - g_reader.length - 1)
-		return (0);
-	required = g_reader.length + length + 1;
-	if (!reserve_bytes(required))
+	if (!reserve_bytes(length))
 		return (0);
-	copy_bytes(g_reader.bytes + g_reader.length, bytes, length);
-	g_reader.length += length;
-	g_reader.bytes[g_reader.length] = '\0';
+	copy_bytes(g_reader.bytes + g_reader.end, bytes, length);
+	g_reader.end += length;
+	g_reader.bytes[g_reader.end] = '\0';
 	return (1);
 }
 
 static char	*release_final_line(void)
 {
 	char	*line;
+	size_t	length;
 
+	length = unread_length();
+	if (g_reader.begin > 0)
+		copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
+	g_reader.bytes[length] = '\0';
 	line = g_reader.bytes;
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
-	g_reader.length = 0;
+	g_reader.begin = 0;
+	g_reader.end = 0;
 	g_reader.capacity = 0;
 	return (line);
 }
@@ -121,7 +157,7 @@ char	*get_next_line(int fd)
 		reset_reader();
 		return (NULL);
 	}
-	if (g_reader.length == 0)
+	if (unread_length() == 0)
 	{
 		reset_reader();
 		return (NULL);


## `feat(reader): 줄을 분리하고 남은 입력 보존`

diff --git a/get_next_line.c b/get_next_line.c
index f951879..50faef8 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -8,11 +8,12 @@ typedef struct s_reader
 	int			fd;
 	char		*bytes;
 	size_t		begin;
+	size_t		scan;
 	size_t		end;
 	size_t		capacity;
 }t_reader;
 
-static t_reader	g_reader = {-1, NULL, 0, 0, 0};
+static t_reader	g_reader = {-1, NULL, 0, 0, 0, 0};
 
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
@@ -32,6 +33,7 @@ static void	reset_reader(void)
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
 	g_reader.begin = 0;
+	g_reader.scan = 0;
 	g_reader.end = 0;
 	g_reader.capacity = 0;
 }
@@ -47,6 +49,7 @@ static void	compact_bytes(void)
 
 	length = unread_length();
 	copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
+	g_reader.scan -= g_reader.begin;
 	g_reader.begin = 0;
 	g_reader.end = length;
 	g_reader.bytes[g_reader.end] = '\0';
@@ -92,6 +95,7 @@ static int	reserve_bytes(size_t appended)
 	free(g_reader.bytes);
 	g_reader.bytes = allocation;
 	g_reader.begin = 0;
+	g_reader.scan = length;
 	g_reader.end = length;
 	g_reader.capacity = capacity;
 	g_reader.bytes[g_reader.end] = '\0';
@@ -108,6 +112,48 @@ static int	append_bytes(const char *bytes, size_t length)
 	return (1);
 }
 
+static size_t	find_line_end(void)
+{
+	while (g_reader.scan < g_reader.end)
+	{
+		if (g_reader.bytes[g_reader.scan] == '\n')
+		{
+			g_reader.scan++;
+			return (g_reader.scan);
+		}
+		g_reader.scan++;
+	}
+	return (0);
+}
+
+static char	*extract_line(size_t line_end)
+{
+	char	*line;
+	size_t	length;
+
+	length = line_end - g_reader.begin;
+	line = malloc(length + 1);
+	if (line == NULL)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	copy_bytes(line, g_reader.bytes + g_reader.begin, length);
+	line[length] = '\0';
+	g_reader.begin = line_end;
+	g_reader.scan = g_reader.begin;
+	if (g_reader.begin == g_reader.end)
+	{
+		free(g_reader.bytes);
+		g_reader.bytes = NULL;
+		g_reader.begin = 0;
+		g_reader.scan = 0;
+		g_reader.end = 0;
+		g_reader.capacity = 0;
+	}
+	return (line);
+}
+
 static char	*release_final_line(void)
 {
 	char	*line;
@@ -121,6 +167,7 @@ static char	*release_final_line(void)
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
 	g_reader.begin = 0;
+	g_reader.scan = 0;
 	g_reader.end = 0;
 	g_reader.capacity = 0;
 	return (line);
@@ -130,6 +177,7 @@ char	*get_next_line(int fd)
 {
 	char	buffer[BUFFER_SIZE];
 	ssize_t	read_size;
+	size_t	line_end;
 
 	if (fd < 0 || read(fd, buffer, 0) < 0)
 	{
@@ -142,6 +190,9 @@ char	*get_next_line(int fd)
 		reset_reader();
 		g_reader.fd = fd;
 	}
+	line_end = find_line_end();
+	if (line_end != 0)
+		return (extract_line(line_end));
 	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	while (read_size > 0)
 	{
@@ -150,6 +201,9 @@ char	*get_next_line(int fd)
 			reset_reader();
 			return (NULL);
 		}
+		line_end = find_line_end();
+		if (line_end != 0)
+			return (extract_line(line_end));
 		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	}
 	if (read_size < 0)


## `test(reader): 개행과 남은 입력 처리 검증`

diff --git a/tests/test_reader.c b/tests/test_reader.c
index 71b4a0d..01882dc 100644
--- a/tests/test_reader.c
+++ b/tests/test_reader.c
@@ -114,10 +114,72 @@ static void	test_invalid_descriptors(void)
 	CHECK(get_next_line(fds[0]) == NULL);
 }
 
+static void	test_newline_and_remainder(void)
+{
+	const char	*expected[3];
+	char			*line;
+	int				fd;
+	size_t			index;
+
+	expected[0] = "one\n";
+	expected[1] = "second\n";
+	expected[2] = "third";
+	fd = reader_for("one\nsecond\nthird", 16);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	index = 0;
+	while (index < 3)
+	{
+		line = get_next_line(fd);
+		CHECK(line != NULL);
+		if (line != NULL)
+		{
+			CHECK(strcmp(line, expected[index]) == 0);
+			free(line);
+		}
+		index++;
+	}
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
+static void	test_empty_lines(void)
+{
+	const char	*expected[3];
+	char			*line;
+	int				fd;
+	size_t			index;
+
+	expected[0] = "\n";
+	expected[1] = "\n";
+	expected[2] = "x\n";
+	fd = reader_for("\n\nx\n", 4);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	index = 0;
+	while (index < 3)
+	{
+		line = get_next_line(fd);
+		CHECK(line != NULL);
+		if (line != NULL)
+		{
+			CHECK(strcmp(line, expected[index]) == 0);
+			free(line);
+		}
+		index++;
+	}
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
 void	test_reader(void)
 {
 	test_empty_input();
 	test_final_line();
 	test_multiple_chunks();
 	test_invalid_descriptors();
+	test_newline_and_remainder();
+	test_empty_lines();
 }


## `test(reader): BUFFER_SIZE 경계값 검증`

diff --git a/Makefile b/Makefile
index 10ed922..7c66da0 100644
--- a/Makefile
+++ b/Makefile
@@ -18,7 +18,7 @@ OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
 
 TEST_BIN := tests/bin/test_reader_$(BUFFER_SIZE)
-TEST_SRC := tests/test_main.c tests/test_reader.c
+TEST_SRC := tests/test_main.c tests/test_reader.c tests/test_boundaries.c
 MATRIX_SIZES := 1 2 42 1024
 
 FAULT_OBJ_DIR := build/fault/$(BUFFER_SIZE)
@@ -31,7 +31,7 @@ FAULT_BIN := tests/bin/test_failure_$(BUFFER_SIZE)
 FAULT_CPPFLAGS := $(CPPFLAGS) -Itests/support
 FAULT_DEFINES := -Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
 
-.PHONY: all bonus clean fclean re test failure-run failure-test check
+.PHONY: all bonus clean fclean re test-run test failure-run failure-test check
 
 all: $(NAME)
 
@@ -44,13 +44,18 @@ $(OBJ_DIR)/%.o: %.c get_next_line.h
 	@$(MKDIR) $(dir $@)
 	$(CC) $(CPPFLAGS) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
 
-$(TEST_BIN): $(NAME) $(TEST_SRC) tests/test.h
+$(TEST_BIN): $(OBJ) $(TEST_SRC) tests/test.h
 	@$(MKDIR) $(dir $@)
-	$(CC) $(CPPFLAGS) $(CFLAGS) $(TEST_SRC) $(NAME) -o $@
+	$(CC) $(CPPFLAGS) $(CFLAGS) $(TEST_SRC) $(OBJ) -o $@
 
-test: $(TEST_BIN)
+test-run: $(TEST_BIN)
 	./$(TEST_BIN)
 
+test:
+	@set -e; for size in $(MATRIX_SIZES); do \
+		$(MAKE) --no-print-directory test-run BUFFER_SIZE=$$size; \
+	done
+
 $(FAULT_READER_OBJ): get_next_line.c get_next_line.h
 	@$(MKDIR) $(dir $@)
 	$(CC) $(FAULT_CPPFLAGS) $(FAULT_DEFINES) $(CFLAGS) $(DEPFLAGS) \
diff --git a/tests/test.h b/tests/test.h
index c427222..7b442e2 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -7,6 +7,7 @@ void	test_check(int condition, const char *expression, const char *file,
 			int line);
 int		test_finish(void);
 void	test_reader(void);
+void	test_boundaries(void);
 
 # define CHECK(condition) \
 	test_check((condition), #condition, __FILE__, __LINE__)
diff --git a/tests/test_boundaries.c b/tests/test_boundaries.c
new file mode 100644
index 0000000..e062c25
--- /dev/null
+++ b/tests/test_boundaries.c
@@ -0,0 +1,275 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "test.h"
+
+#include "get_next_line.h"
+
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+static int	write_all(int fd, const char *bytes, size_t length)
+{
+	ssize_t	written;
+
+	while (length > 0)
+	{
+		written = write(fd, bytes, length);
+		if (written <= 0)
+			return (0);
+		bytes += written;
+		length -= (size_t)written;
+	}
+	return (1);
+}
+
+static int	file_from(const char *bytes, size_t length)
+{
+	char	path[] = "/tmp/buffered-line-reader-XXXXXX";
+	int		fd;
+
+	fd = mkstemp(path);
+	if (fd < 0)
+		return (-1);
+	unlink(path);
+	if (!write_all(fd, bytes, length) || lseek(fd, 0, SEEK_SET) < 0)
+	{
+		close(fd);
+		return (-1);
+	}
+	return (fd);
+}
+
+static int	pipe_from(const char *bytes, size_t length)
+{
+	int	fds[2];
+
+	if (pipe(fds) != 0)
+		return (-1);
+	if (!write_all(fds[1], bytes, length))
+	{
+		close(fds[0]);
+		close(fds[1]);
+		return (-1);
+	}
+	close(fds[1]);
+	return (fds[0]);
+}
+
+static void	fill_bytes(char *bytes, size_t length, char offset)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < length)
+	{
+		bytes[index] = (char)('a' + (offset + (char)(index % 26)) % 26);
+		index++;
+	}
+}
+
+static int	matches(const char *actual, const char *expected, size_t length)
+{
+	return (actual != NULL && strlen(actual) == length
+		&& memcmp(actual, expected, length) == 0);
+}
+
+static void	check_single_line(size_t body_length, int with_newline)
+{
+	char	*expected;
+	char	*line;
+	int		fd;
+	size_t	length;
+
+	length = body_length + (size_t)with_newline;
+	expected = malloc(length + 1);
+	CHECK(expected != NULL);
+	if (expected == NULL)
+		return ;
+	fill_bytes(expected, body_length, 0);
+	if (with_newline)
+		expected[body_length] = '\n';
+	expected[length] = '\0';
+	fd = file_from(expected, length);
+	CHECK(fd >= 0);
+	if (fd >= 0)
+	{
+		line = get_next_line(fd);
+		CHECK(matches(line, expected, length));
+		free(line);
+		CHECK(get_next_line(fd) == NULL);
+		CHECK(get_next_line(fd) == NULL);
+		close(fd);
+	}
+	free(expected);
+}
+
+static void	test_chunk_boundaries(void)
+{
+	size_t	chunk;
+
+	chunk = (size_t)BUFFER_SIZE;
+	check_single_line(chunk - 1, 1);
+	check_single_line(chunk, 1);
+	check_single_line(chunk + 1, 1);
+	check_single_line(chunk * 3 + 7, 1);
+	check_single_line(chunk, 0);
+	check_single_line(chunk + 1, 0);
+	check_single_line(chunk * 2 + 1, 0);
+}
+
+static void	test_large_adjacent_lines(void)
+{
+	const size_t	first_length = 32769;
+	const size_t	second_length = 32771;
+	char			*bytes;
+	char			*line;
+	int				fd;
+
+	bytes = malloc(first_length + second_length + 1);
+	CHECK(bytes != NULL);
+	if (bytes == NULL)
+		return ;
+	fill_bytes(bytes, first_length - 1, 2);
+	bytes[first_length - 1] = '\n';
+	fill_bytes(bytes + first_length, second_length, 11);
+	bytes[first_length + second_length] = '\0';
+	fd = file_from(bytes, first_length + second_length);
+	CHECK(fd >= 0);
+	if (fd >= 0)
+	{
+		line = get_next_line(fd);
+		CHECK(matches(line, bytes, first_length));
+		free(line);
+		line = get_next_line(fd);
+		CHECK(matches(line, bytes + first_length, second_length));
+		free(line);
+		CHECK(get_next_line(fd) == NULL);
+		close(fd);
+	}
+	free(bytes);
+}
+
+static void	test_small_pipe(void)
+{
+	char	*line;
+	int		fd;
+
+	fd = pipe_from("pipe one\npipe two", 17);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "pipe one\n") == 0);
+	free(line);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "pipe two") == 0);
+	free(line);
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
+static void	test_return_storage_independence(void)
+{
+	char	*first;
+	char	*second;
+	int		fd;
+
+	fd = file_from("retained first\nsecond line\n", 27);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	first = get_next_line(fd);
+	CHECK(first != NULL && strcmp(first, "retained first\n") == 0);
+	second = get_next_line(fd);
+	CHECK(second != NULL && strcmp(second, "second line\n") == 0);
+	CHECK(first != NULL && strcmp(first, "retained first\n") == 0);
+	CHECK(first != second);
+	free(first);
+	free(second);
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
+static void	test_fd_ownership(void)
+{
+	char	*left_line;
+	char	*right_line;
+	int		left;
+	int		right;
+
+	left = file_from("left-a\nleft-b", 13);
+	right = file_from("right-a\nright-b\n", 16);
+	CHECK(left >= 0 && right >= 0);
+	if (left < 0 || right < 0)
+	{
+		if (left >= 0)
+			close(left);
+		if (right >= 0)
+			close(right);
+		return ;
+	}
+	left_line = get_next_line(left);
+	right_line = get_next_line(right);
+	CHECK(left_line != NULL && strcmp(left_line, "left-a\n") == 0);
+	CHECK(right_line != NULL && strcmp(right_line, "right-a\n") == 0);
+	free(left_line);
+	free(right_line);
+	right_line = get_next_line(right);
+	left_line = get_next_line(left);
+	CHECK(right_line != NULL && strcmp(right_line, "right-b\n") == 0);
+	CHECK(left_line != NULL && strcmp(left_line, "left-b") == 0);
+	free(right_line);
+	free(left_line);
+	CHECK(get_next_line(left) == NULL);
+	CHECK(get_next_line(right) == NULL);
+	close(left);
+	close(right);
+}
+
+static void	test_high_descriptor(void)
+{
+	char	*line;
+	int		fd;
+	int		high_fd;
+
+	fd = file_from("high descriptor\n", 16);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	high_fd = fcntl(fd, F_DUPFD, 128);
+	close(fd);
+	if (high_fd < 0)
+		return ;
+	line = get_next_line(high_fd);
+	CHECK(line != NULL && strcmp(line, "high descriptor\n") == 0);
+	free(line);
+	CHECK(get_next_line(high_fd) == NULL);
+	close(high_fd);
+}
+
+static void	test_empty_and_repeated_eof(void)
+{
+	int	fd;
+
+	fd = file_from("", 0);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
+void	test_boundaries(void)
+{
+	test_chunk_boundaries();
+	test_large_adjacent_lines();
+	test_small_pipe();
+	test_return_storage_independence();
+	test_fd_ownership();
+	test_high_descriptor();
+	test_empty_and_repeated_eof();
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index d14c319..42b1a3a 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -31,5 +31,6 @@ int	test_finish(void)
 int	main(void)
 {
 	test_reader();
+	test_boundaries();
 	return (test_finish());
 }


## `refactor(buffer): 남은 입력 버퍼를 읽기 공간으로 재사용`

diff --git a/get_next_line.c b/get_next_line.c
index 3a379ab..7f80c08 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -134,16 +134,6 @@ static int	reserve_bytes(t_reader *reader, size_t appended)
 	return (1);
 }
 
-static int	append_bytes(t_reader *reader, const char *bytes, size_t length)
-{
-	if (!reserve_bytes(reader, length))
-		return (0);
-	copy_bytes(reader->bytes + reader->end, bytes, length);
-	reader->end += length;
-	reader->bytes[reader->end] = '\0';
-	return (1);
-}
-
 static size_t	find_line_end(t_reader *reader)
 {
 	while (reader->scan < reader->end)
@@ -196,12 +186,12 @@ static char	*release_final_line(t_reader *reader)
 
 char	*get_next_line(int fd)
 {
-	char	buffer[BUFFER_SIZE];
+	char		probe;
 	ssize_t	read_size;
 	size_t	line_end;
 	t_reader	*reader;
 
-	if (fd < 0 || read(fd, buffer, 0) < 0)
+	if (fd < 0 || read(fd, &probe, 0) < 0)
 	{
 		reader = find_reader(fd);
 		if (reader != NULL)
@@ -216,18 +206,22 @@ char	*get_next_line(int fd)
 	line_end = find_line_end(reader);
 	if (line_end != 0)
 		return (extract_line(reader, line_end));
-	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
-	while (read_size > 0)
+	while (1)
 	{
-		if (!append_bytes(reader, buffer, (size_t)read_size))
+		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
 		{
 			discard_reader(reader);
 			return (NULL);
 		}
+		read_size = read(fd, reader->bytes + reader->end,
+				(size_t)BUFFER_SIZE);
+		if (read_size <= 0)
+			break ;
+		reader->end += (size_t)read_size;
+		reader->bytes[reader->end] = '\0';
 		line_end = find_line_end(reader);
 		if (line_end != 0)
 			return (extract_line(reader, line_end));
-		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	}
 	if (read_size < 0)
 	{


