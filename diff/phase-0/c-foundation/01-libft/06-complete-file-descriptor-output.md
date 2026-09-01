# 파일 디스크립터 출력 완결성

## `feat(io): 파일 디스크립터 출력 함수 추가`

diff --git a/Makefile b/Makefile
index 35d5892..ec7dd83 100644
--- a/Makefile
+++ b/Makefile
@@ -23,7 +23,8 @@ SRC := \
 	src/string/ft_string_transform.c \
 	src/convert/ft_atoi.c \
 	src/convert/ft_itoa.c \
-	src/alloc/ft_allocate.c
+	src/alloc/ft_allocate.c \
+	src/io/ft_fd_output.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index a73b9d8..318bb23 100644
--- a/libft.h
+++ b/libft.h
@@ -37,4 +37,9 @@ char	*ft_itoa(int number);
 char	*ft_strmapi(const char *text, char (*function)(unsigned int, char));
 void	ft_striteri(char *text, void (*function)(unsigned int, char *));
 
+void	ft_putchar_fd(char character, int fd);
+void	ft_putstr_fd(char *text, int fd);
+void	ft_putendl_fd(char *text, int fd);
+void	ft_putnbr_fd(int number, int fd);
+
 #endif
diff --git a/src/io/ft_fd_output.c b/src/io/ft_fd_output.c
new file mode 100644
index 0000000..150a79f
--- /dev/null
+++ b/src/io/ft_fd_output.c
@@ -0,0 +1,46 @@
+#include "libft.h"
+
+#include <unistd.h>
+
+void	ft_putchar_fd(char character, int fd)
+{
+	(void)write(fd, &character, 1);
+}
+
+void	ft_putstr_fd(char *text, int fd)
+{
+	(void)write(fd, text, ft_strlen(text));
+}
+
+void	ft_putendl_fd(char *text, int fd)
+{
+	ft_putstr_fd(text, fd);
+	ft_putchar_fd('\n', fd);
+}
+
+void	ft_putnbr_fd(int number, int fd)
+{
+	char		buffer[11];
+	unsigned int	magnitude;
+	size_t		index;
+
+	if (number < 0)
+		magnitude = (unsigned int)(-(number + 1)) + 1U;
+	else
+		magnitude = (unsigned int)number;
+	index = sizeof(buffer);
+	while (magnitude >= 10U)
+	{
+		index--;
+		buffer[index] = (char)('0' + magnitude % 10U);
+		magnitude /= 10U;
+	}
+	index--;
+	buffer[index] = (char)('0' + magnitude);
+	if (number < 0)
+	{
+		index--;
+		buffer[index] = '-';
+	}
+	(void)write(fd, buffer + index, sizeof(buffer) - index);
+}


## `test(io): 파일 디스크립터 출력 검증`

diff --git a/tests/test.h b/tests/test.h
index 95b3c50..1fff33b 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -17,6 +17,7 @@ void	test_string_build(void);
 void	test_split(void);
 void	test_itoa(void);
 void	test_string_transform(void);
+void	test_fd_output(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_fd_output.c b/tests/test_fd_output.c
new file mode 100644
index 0000000..a5530d5
--- /dev/null
+++ b/tests/test_fd_output.c
@@ -0,0 +1,67 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <errno.h>
+#include <limits.h>
+#include <signal.h>
+#include <stddef.h>
+#include <string.h>
+#include <unistd.h>
+
+static void	check_output(void)
+{
+	static const char	expected[] =
+		"Afoundation\n0|-1|2147483647|-2147483648";
+	char			actual[sizeof(expected) + 8];
+	int			pipe_fd[2];
+	ssize_t			bytes_read;
+
+	CHECK(pipe(pipe_fd) == 0);
+	ft_putchar_fd('A', pipe_fd[1]);
+	ft_putendl_fd("foundation", pipe_fd[1]);
+	ft_putnbr_fd(0, pipe_fd[1]);
+	ft_putchar_fd('|', pipe_fd[1]);
+	ft_putnbr_fd(-1, pipe_fd[1]);
+	ft_putchar_fd('|', pipe_fd[1]);
+	ft_putnbr_fd(INT_MAX, pipe_fd[1]);
+	ft_putchar_fd('|', pipe_fd[1]);
+	ft_putnbr_fd(INT_MIN, pipe_fd[1]);
+	CHECK(close(pipe_fd[1]) == 0);
+	bytes_read = read(pipe_fd[0], actual, sizeof(actual));
+	CHECK(bytes_read == (ssize_t)(sizeof(expected) - 1));
+	if (bytes_read == (ssize_t)(sizeof(expected) - 1))
+		CHECK(memcmp(actual, expected, sizeof(expected) - 1) == 0);
+	CHECK(close(pipe_fd[0]) == 0);
+}
+
+static void	check_broken_pipe_policy(void)
+{
+	void	(*previous_handler)(int);
+	int		pipe_fd[2];
+
+	previous_handler = signal(SIGPIPE, SIG_IGN);
+	CHECK(previous_handler != SIG_ERR);
+	CHECK(pipe(pipe_fd) == 0);
+	CHECK(close(pipe_fd[0]) == 0);
+	errno = 0;
+	ft_putstr_fd("closed", pipe_fd[1]);
+	CHECK(errno == EPIPE);
+	CHECK(close(pipe_fd[1]) == 0);
+	CHECK(signal(SIGPIPE, previous_handler) == SIG_IGN);
+}
+
+void	test_fd_output(void)
+{
+	int	pipe_fd[2];
+
+	check_output();
+	check_broken_pipe_policy();
+	CHECK(pipe(pipe_fd) == 0);
+	CHECK(close(pipe_fd[0]) == 0);
+	CHECK(close(pipe_fd[1]) == 0);
+	ft_putchar_fd('x', pipe_fd[1]);
+	ft_putstr_fd("", pipe_fd[1]);
+	ft_putendl_fd("", pipe_fd[1]);
+	ft_putnbr_fd(42, pipe_fd[1]);
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index e5be8b8..f49b011 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -42,5 +42,6 @@ int	main(void)
 	test_split();
 	test_itoa();
 	test_string_transform();
+	test_fd_output();
 	return (test_finish());
 }


## `refactor(io): 숫자 출력을 자릿수 helper로 분리`

diff --git a/src/io/ft_fd_output.c b/src/io/ft_fd_output.c
index 150a79f..9087f23 100644
--- a/src/io/ft_fd_output.c
+++ b/src/io/ft_fd_output.c
@@ -18,29 +18,23 @@ void	ft_putendl_fd(char *text, int fd)
 	ft_putchar_fd('\n', fd);
 }
 
+static void	put_unsigned(unsigned int magnitude, int fd)
+{
+	if (magnitude >= 10U)
+		put_unsigned(magnitude / 10U, fd);
+	ft_putchar_fd((char)('0' + magnitude % 10U), fd);
+}
+
 void	ft_putnbr_fd(int number, int fd)
 {
-	char		buffer[11];
 	unsigned int	magnitude;
-	size_t		index;
 
 	if (number < 0)
+	{
+		ft_putchar_fd('-', fd);
 		magnitude = (unsigned int)(-(number + 1)) + 1U;
+	}
 	else
 		magnitude = (unsigned int)number;
-	index = sizeof(buffer);
-	while (magnitude >= 10U)
-	{
-		index--;
-		buffer[index] = (char)('0' + magnitude % 10U);
-		magnitude /= 10U;
-	}
-	index--;
-	buffer[index] = (char)('0' + magnitude);
-	if (number < 0)
-	{
-		index--;
-		buffer[index] = '-';
-	}
-	(void)write(fd, buffer + index, sizeof(buffer) - index);
+	put_unsigned(magnitude, fd);
 }


## `fix(io): 파일 디스크립터 출력을 끝까지 재시도`

diff --git a/src/io/ft_fd_output.c b/src/io/ft_fd_output.c
index 9087f23..0595eaa 100644
--- a/src/io/ft_fd_output.c
+++ b/src/io/ft_fd_output.c
@@ -1,28 +1,62 @@
+#define _POSIX_C_SOURCE 200809L
+
 #include "libft.h"
 
+#include <errno.h>
+#include <limits.h>
 #include <unistd.h>
 
+static int	write_all(int fd, const char *buffer, size_t length)
+{
+	ssize_t	written;
+	size_t	offset;
+	size_t	request;
+
+	offset = 0;
+	while (offset < length)
+	{
+		request = length - offset;
+		if (request > (size_t)SSIZE_MAX)
+			request = (size_t)SSIZE_MAX;
+		written = write(fd, buffer + offset, request);
+		if (written > 0)
+			offset += (size_t)written;
+		else if (written < 0 && errno == EINTR)
+			continue ;
+		else
+		{
+			if (written == 0)
+				errno = EIO;
+			return (0);
+		}
+	}
+	return (1);
+}
+
 void	ft_putchar_fd(char character, int fd)
 {
-	(void)write(fd, &character, 1);
+	(void)write_all(fd, &character, 1);
 }
 
 void	ft_putstr_fd(char *text, int fd)
 {
-	(void)write(fd, text, ft_strlen(text));
+	(void)write_all(fd, text, ft_strlen(text));
 }
 
 void	ft_putendl_fd(char *text, int fd)
 {
-	ft_putstr_fd(text, fd);
-	ft_putchar_fd('\n', fd);
+	if (write_all(fd, text, ft_strlen(text)))
+		(void)write_all(fd, "\n", 1);
 }
 
-static void	put_unsigned(unsigned int magnitude, int fd)
+static int	put_unsigned(unsigned int magnitude, int fd)
 {
-	if (magnitude >= 10U)
-		put_unsigned(magnitude / 10U, fd);
-	ft_putchar_fd((char)('0' + magnitude % 10U), fd);
+	char	digit;
+
+	if (magnitude >= 10U && !put_unsigned(magnitude / 10U, fd))
+		return (0);
+	digit = (char)('0' + magnitude % 10U);
+	return (write_all(fd, &digit, 1));
 }
 
 void	ft_putnbr_fd(int number, int fd)
@@ -31,10 +65,11 @@ void	ft_putnbr_fd(int number, int fd)
 
 	if (number < 0)
 	{
-		ft_putchar_fd('-', fd);
+		if (!write_all(fd, "-", 1))
+			return ;
 		magnitude = (unsigned int)(-(number + 1)) + 1U;
 	}
 	else
 		magnitude = (unsigned int)number;
-	put_unsigned(magnitude, fd);
+	(void)put_unsigned(magnitude, fd);
 }


## `test(io): 부분 쓰기와 EINTR 이후 진행을 검증`

diff --git a/Makefile b/Makefile
index fcc65a4..fba7298 100644
--- a/Makefile
+++ b/Makefile
@@ -34,7 +34,15 @@ DEP := $(OBJ:.o=.d)
 TEST_BIN := tests/bin/test_libft
 TEST_SRC := $(wildcard tests/test_*.c)
 
-.PHONY: all clean fclean re test
+WRITE_OBJ_DIR := build/write-failure
+WRITE_OUTPUT_OBJ := $(WRITE_OBJ_DIR)/ft_fd_output.o
+WRITE_DEP := $(WRITE_OUTPUT_OBJ:.o=.d)
+WRITE_BIN := tests/bin/test_write_failure
+WRITE_TEST_SRC := tests/failure/test_fd_output_failure.c \
+	tests/support/fail_write.c
+WRITE_DEFINES := -Dwrite=test_write
+
+.PHONY: all clean fclean re test write-failure-test
 
 all: $(NAME)
 
@@ -52,6 +60,21 @@ $(TEST_BIN): $(NAME) $(TEST_SRC) tests/test.h
 test: $(TEST_BIN)
 	./$(TEST_BIN)
 
+$(WRITE_OUTPUT_OBJ): src/io/ft_fd_output.c libft.h \
+		tests/support/fail_write.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(WRITE_DEFINES) $(CFLAGS) \
+		$(DEPFLAGS) -c $< -o $@
+
+$(WRITE_BIN): $(NAME) $(WRITE_OUTPUT_OBJ) $(WRITE_TEST_SRC) \
+		tests/support/fail_write.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CPPFLAGS) $(CFLAGS) \
+		$(WRITE_TEST_SRC) $(WRITE_OUTPUT_OBJ) $(NAME) -o $@
+
+write-failure-test: $(WRITE_BIN)
+	./$(WRITE_BIN)
+
 clean:
 	$(RMDIR) build tests/bin
 
@@ -60,4 +83,4 @@ fclean: clean
 
 re: fclean all
 
--include $(DEP)
+-include $(DEP) $(WRITE_DEP)
diff --git a/tests/failure/test_fd_output_failure.c b/tests/failure/test_fd_output_failure.c
new file mode 100644
index 0000000..a0ed32b
--- /dev/null
+++ b/tests/failure/test_fd_output_failure.c
@@ -0,0 +1,122 @@
+#include "libft.h"
+#include "tests/support/fail_write.h"
+
+#include <errno.h>
+#include <limits.h>
+#include <stdio.h>
+#include <string.h>
+
+static int	g_checks;
+static int	g_failures;
+
+#define VERIFY(expression) verify((expression), #expression, __LINE__)
+
+static void	verify(int condition, const char *expression, int line)
+{
+	g_checks++;
+	if (!condition)
+	{
+		g_failures++;
+		fprintf(stderr, "write failure test line %d: %s\n", line, expression);
+	}
+}
+
+static void	check_partial_and_interrupted_write(void)
+{
+	static const t_write_step	steps[] = {
+	{2, 0}, {-1, EINTR}, {1, 0}, {3, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	ft_putstr_fd("abcdef", 91);
+	VERIFY(test_writer_calls() == 4);
+	VERIFY(test_writer_fd(0) == 91 && test_writer_fd(3) == 91);
+	VERIFY(test_writer_request(0) == 6);
+	VERIFY(test_writer_request(1) == 4);
+	VERIFY(test_writer_request(2) == 4);
+	VERIFY(test_writer_request(3) == 3);
+	VERIFY(test_writer_output_size() == 6);
+	VERIFY(memcmp(test_writer_output(), "abcdef", 6) == 0);
+	VERIFY(test_writer_invalid() == 0);
+}
+
+static void	check_zero_write_stops(void)
+{
+	static const t_write_step	steps[] = {{2, 0}, {0, 0}, {4, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	errno = EDOM;
+	ft_putstr_fd("abcdef", 17);
+	VERIFY(test_writer_calls() == 2);
+	VERIFY(test_writer_request(0) == 6);
+	VERIFY(test_writer_request(1) == 4);
+	VERIFY(test_writer_output_size() == 2);
+	VERIFY(memcmp(test_writer_output(), "ab", 2) == 0);
+	VERIFY(test_writer_invalid() == 0);
+	VERIFY(errno == EIO);
+}
+
+static void	check_permanent_error_stops(void)
+{
+	static const t_write_step	steps[] = {{1, 0}, {-1, EIO}, {5, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	errno = 0;
+	ft_putendl_fd("error", 23);
+	VERIFY(test_writer_calls() == 2);
+	VERIFY(test_writer_request(0) == 5);
+	VERIFY(test_writer_request(1) == 4);
+	VERIFY(test_writer_output_size() == 1);
+	VERIFY(memcmp(test_writer_output(), "e", 1) == 0);
+	VERIFY(test_writer_invalid() == 0);
+	VERIFY(errno == EIO);
+}
+
+static void	check_number_retries_remaining_bytes(void)
+{
+	static const char			expected[] = "-2147483648";
+	static const t_write_step	steps[] = {
+	{1, 0}, {1, 0}, {-1, EINTR}, {1, 0}, {1, 0}, {1, 0},
+	{1, 0}, {1, 0}, {1, 0}, {1, 0}, {1, 0}, {1, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	ft_putnbr_fd(INT_MIN, 29);
+	VERIFY(test_writer_calls() == 12);
+	VERIFY(test_writer_request(0) == 1);
+	VERIFY(test_writer_request(2) == 1);
+	VERIFY(test_writer_request(11) == 1);
+	VERIFY(test_writer_output_size() == sizeof(expected) - 1);
+	VERIFY(memcmp(test_writer_output(), expected, sizeof(expected) - 1) == 0);
+	VERIFY(test_writer_invalid() == 0);
+}
+
+static void	check_number_stops_after_error(void)
+{
+	static const t_write_step	steps[] = {
+	{1, 0}, {1, 0}, {-1, EPIPE}, {1, 0}};
+
+	test_writer_reset(steps, sizeof(steps) / sizeof(steps[0]));
+	errno = 0;
+	ft_putnbr_fd(INT_MIN, 31);
+	VERIFY(test_writer_calls() == 3);
+	VERIFY(test_writer_output_size() == 2);
+	VERIFY(memcmp(test_writer_output(), "-2", 2) == 0);
+	VERIFY(test_writer_invalid() == 0);
+	VERIFY(errno == EPIPE);
+}
+
+int	main(void)
+{
+	check_partial_and_interrupted_write();
+	check_zero_write_stops();
+	check_permanent_error_stops();
+	check_number_retries_remaining_bytes();
+	check_number_stops_after_error();
+	if (g_failures != 0)
+	{
+		fprintf(stderr, "%d of %d write failure checks failed\n",
+			g_failures, g_checks);
+		return (1);
+	}
+	printf("%d write failure checks passed\n", g_checks);
+	return (0);
+}
diff --git a/tests/support/fail_write.c b/tests/support/fail_write.c
new file mode 100644
index 0000000..7e01538
--- /dev/null
+++ b/tests/support/fail_write.c
@@ -0,0 +1,84 @@
+#include "tests/support/fail_write.h"
+
+#include <errno.h>
+#include <string.h>
+
+#define TEST_WRITE_LIMIT 64
+#define TEST_OUTPUT_LIMIT 256
+
+static const t_write_step	*g_steps;
+static size_t				g_step_count;
+static size_t				g_calls;
+static int					g_fds[TEST_WRITE_LIMIT];
+static size_t				g_requests[TEST_WRITE_LIMIT];
+static char					g_output[TEST_OUTPUT_LIMIT];
+static size_t				g_output_size;
+static int					g_invalid;
+
+void	test_writer_reset(const t_write_step *steps, size_t step_count)
+{
+	g_steps = steps;
+	g_step_count = step_count;
+	g_calls = 0;
+	g_output_size = 0;
+	g_invalid = 0;
+}
+
+ssize_t	test_write(int fd, const void *buffer, size_t length)
+{
+	t_write_step	step;
+	size_t			current;
+
+	current = g_calls;
+	if (current >= TEST_WRITE_LIMIT || current >= g_step_count)
+		return (g_invalid = 1, errno = EIO, -1);
+	g_fds[current] = fd;
+	g_requests[current] = length;
+	g_calls++;
+	step = g_steps[current];
+	if (step.result < 0)
+		return (errno = step.error_number, -1);
+	if ((size_t)step.result > length
+		|| (size_t)step.result > TEST_OUTPUT_LIMIT - g_output_size)
+		return (g_invalid = 1, errno = EINVAL, -1);
+	if (step.result > 0)
+	{
+		memcpy(g_output + g_output_size, buffer, (size_t)step.result);
+		g_output_size += (size_t)step.result;
+	}
+	return (step.result);
+}
+
+size_t	test_writer_calls(void)
+{
+	return (g_calls);
+}
+
+int	test_writer_fd(size_t index)
+{
+	if (index >= g_calls)
+		return (-1);
+	return (g_fds[index]);
+}
+
+size_t	test_writer_request(size_t index)
+{
+	if (index >= g_calls)
+		return (0);
+	return (g_requests[index]);
+}
+
+const char	*test_writer_output(void)
+{
+	return (g_output);
+}
+
+size_t	test_writer_output_size(void)
+{
+	return (g_output_size);
+}
+
+int	test_writer_invalid(void)
+{
+	return (g_invalid);
+}
diff --git a/tests/support/fail_write.h b/tests/support/fail_write.h
new file mode 100644
index 0000000..2d01ded
--- /dev/null
+++ b/tests/support/fail_write.h
@@ -0,0 +1,22 @@
+#ifndef FAIL_WRITE_H
+# define FAIL_WRITE_H
+
+# include <stddef.h>
+# include <sys/types.h>
+
+typedef struct s_write_step
+{
+	ssize_t	result;
+	int		error_number;
+}	t_write_step;
+
+void		test_writer_reset(const t_write_step *steps, size_t step_count);
+ssize_t		test_write(int fd, const void *buffer, size_t length);
+size_t		test_writer_calls(void);
+int			test_writer_fd(size_t index);
+size_t		test_writer_request(size_t index);
+const char	*test_writer_output(void);
+size_t		test_writer_output_size(void);
+int			test_writer_invalid(void);
+
+#endif
