# 출력 컨텍스트와 쓰기 복구

## `feat(output): 출력 컨텍스트와 쓰기 API 추가`

diff --git a/Makefile b/Makefile
index 90d8db3..092971e 100644
--- a/Makefile
+++ b/Makefile
@@ -7,15 +7,17 @@ AR := ar
 ARFLAGS := rcs
 RM := rm -f
 
-SRC := src/ft_printf.c
+SRC := src/ft_printf.c \
+	src/ft_output.c
 OBJ := $(SRC:.c=.o)
+HEADER := include/ft_printf.h src/ft_printf_internal.h
 
 all: $(NAME)
 
 $(NAME): $(OBJ)
 	$(AR) $(ARFLAGS) $@ $^
 
-%.o: %.c include/ft_printf.h
+%.o: %.c $(HEADER)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
 
 clean:
diff --git a/src/ft_output.c b/src/ft_output.c
new file mode 100644
index 0000000..de1d83a
--- /dev/null
+++ b/src/ft_output.c
@@ -0,0 +1,42 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+#include <unistd.h>
+
+void	ft_printf_init(t_printf *ctx, int fd)
+{
+	ctx->fd = fd;
+	ctx->count = 0;
+	ctx->error = 0;
+}
+
+int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
+{
+	ssize_t	written;
+
+	if (ctx->error)
+		return (-1);
+	while (length > 0)
+	{
+		written = write(ctx->fd, buffer, length);
+		if (written <= 0)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		if (ctx->count > INT_MAX - (int)written)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		ctx->count += (int)written;
+		buffer += written;
+		length -= (size_t)written;
+	}
+	return (0);
+}
+
+int	ft_printf_putchar(t_printf *ctx, char c)
+{
+	return (ft_printf_write(ctx, &c, 1));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
new file mode 100644
index 0000000..524c99a
--- /dev/null
+++ b/src/ft_printf_internal.h
@@ -0,0 +1,19 @@
+#ifndef FT_PRINTF_INTERNAL_H
+# define FT_PRINTF_INTERNAL_H
+
+# include "ft_printf.h"
+
+# include <stddef.h>
+
+typedef struct s_printf
+{
+	int	fd;
+	int	count;
+	int	error;
+}	t_printf;
+
+void		ft_printf_init(t_printf *ctx, int fd);
+int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
+int			ft_printf_putchar(t_printf *ctx, char c);
+
+#endif


## `refactor(core): 리터럴 출력을 컨텍스트 API로 이관`

diff --git a/src/ft_printf.c b/src/ft_printf.c
index 9f4e8a3..fd0e7bd 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -1,51 +1,34 @@
-#include "ft_printf.h"
+#include "ft_printf_internal.h"
 
-#include <limits.h>
 #include <stdarg.h>
-#include <unistd.h>
-
-static int	ft_write_count(const char *buffer, int length, int *count)
-{
-	int	written;
-
-	if (length <= 0)
-		return (0);
-	if (*count > INT_MAX - length)
-		return (-1);
-	written = (int)write(1, buffer, (size_t)length);
-	if (written < 0 || written != length)
-		return (-1);
-	*count += written;
-	return (0);
-}
 
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
-	int		count;
+	t_printf	ctx;
 
 	(void)args;
 	if (format == 0)
 		return (-1);
 	va_start(args, format);
-	count = 0;
+	ft_printf_init(&ctx, 1);
 	while (*format)
 	{
 		if (*format == '%' && *(format + 1) == '%')
 		{
-			if (ft_write_count("%", 1, &count) < 0)
-				count = -1;
+			if (ft_printf_putchar(&ctx, '%') < 0)
+				break ;
 			format += 2;
 		}
 		else
 		{
-			if (ft_write_count(format, 1, &count) < 0)
-				count = -1;
+			if (ft_printf_putchar(&ctx, *format) < 0)
+				break ;
 			format++;
 		}
-		if (count < 0)
-			break ;
 	}
 	va_end(args);
-	return (count);
+	if (ctx.error)
+		return (-1);
+	return (ctx.count);
 }


## `fix(output): 쓰기 결과를 집계하기 전에 범위 검증`

diff --git a/src/ft_output.c b/src/ft_output.c
index 1a214c3..a4d4061 100644
--- a/src/ft_output.c
+++ b/src/ft_output.c
@@ -24,7 +24,7 @@ int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
 			ctx->error = 1;
 			return (-1);
 		}
-		if (ctx->count > INT_MAX - (int)written)
+		if (written > INT_MAX || ctx->count > INT_MAX - (int)written)
 		{
 			ctx->error = 1;
 			return (-1);


## `test(output): 표준 출력 실패 전파 검증`

diff --git a/tests/test_ft_printf.c b/tests/test_ft_printf.c
index 46847b1..bb08d3f 100644
--- a/tests/test_ft_printf.c
+++ b/tests/test_ft_printf.c
@@ -173,11 +173,31 @@ static void	run_parser_boundary_cases(void)
 	expect_field_error(__LINE__, "%.2147483648d");
 }
 
+static void	run_error_cases(void)
+{
+	int	saved_stdout;
+	int	result;
+
+	fflush(stdout);
+	saved_stdout = dup(STDOUT_FILENO);
+	if (saved_stdout < 0)
+		fail_test(__LINE__, "dup for write-error test failed");
+	if (close(STDOUT_FILENO) < 0)
+		fail_test(__LINE__, "close stdout failed");
+	result = ft_printf("closed stdout");
+	if (dup2(saved_stdout, STDOUT_FILENO) < 0)
+		fail_test(__LINE__, "stdout restore failed");
+	close(saved_stdout);
+	if (result != -1)
+		fail_test(__LINE__, "ft_printf did not report write failure");
+}
+
 int	main(void)
 {
 	run_core_cases();
 	run_bonus_cases();
 	run_parser_boundary_cases();
+	run_error_cases();
 	dprintf(STDERR_FILENO, "ft_printf tests passed\n");
 	return (0);
 }


## `fix(output): 중단된 쓰기 재시도와 요청 크기 제한`

diff --git a/src/ft_output.c b/src/ft_output.c
index a4d4061..99987ed 100644
--- a/src/ft_output.c
+++ b/src/ft_output.c
@@ -1,8 +1,16 @@
 #include "ft_printf_internal.h"
 
+#include <errno.h>
 #include <limits.h>
 #include <unistd.h>
 
+#ifdef FT_PRINTF_TEST_WRITE
+ssize_t	ft_printf_test_write(int fd, const void *buffer, size_t length);
+# define FT_PRINTF_SYSTEM_WRITE ft_printf_test_write
+#else
+# define FT_PRINTF_SYSTEM_WRITE write
+#endif
+
 void	ft_printf_init(t_printf *ctx, int fd)
 {
 	ctx->fd = fd;
@@ -13,12 +21,18 @@ void	ft_printf_init(t_printf *ctx, int fd)
 int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
 {
 	ssize_t	written;
+	size_t	request;
 
 	if (ctx->error)
 		return (-1);
 	while (length > 0)
 	{
-		written = write(ctx->fd, buffer, length);
+		request = length;
+		if (request > (size_t)SSIZE_MAX)
+			request = (size_t)SSIZE_MAX;
+		written = FT_PRINTF_SYSTEM_WRITE(ctx->fd, buffer, request);
+		if (written < 0 && errno == EINTR)
+			continue ;
 		if (written <= 0)
 		{
 			ctx->error = 1;


## `perf(output): 반복 채움을 묶어서 출력`

diff --git a/src/ft_output.c b/src/ft_output.c
index 99987ed..6785b7b 100644
--- a/src/ft_output.c
+++ b/src/ft_output.c
@@ -57,11 +57,21 @@ int	ft_printf_putchar(t_printf *ctx, char c)
 
 int	ft_printf_putnchar(t_printf *ctx, char c, int length)
 {
+	char	buffer[64];
+	int		index;
+	int		chunk;
+
+	index = 0;
+	while (index < (int)sizeof(buffer))
+		buffer[index++] = c;
 	while (length > 0)
 	{
-		if (ft_printf_putchar(ctx, c) < 0)
+		chunk = length;
+		if (chunk > (int)sizeof(buffer))
+			chunk = (int)sizeof(buffer);
+		if (ft_printf_write(ctx, buffer, (size_t)chunk) < 0)
 			return (-1);
-		length--;
+		length -= chunk;
 	}
 	return (0);
 }


## `test(output): 쓰기 실패 시퀀스와 채움 전략 검증`

diff --git a/Makefile b/Makefile
index 8a69095..730b260 100644
--- a/Makefile
+++ b/Makefile
@@ -18,6 +18,7 @@ SRC := src/ft_printf.c \
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 TEST_BIN := tests/bin/test_ft_printf
+FAULT_TEST_BIN := tests/bin/test_output_faults
 
 all: $(NAME)
 
@@ -31,13 +32,16 @@ test: $(NAME)
 	mkdir -p tests/bin
 	$(CC) $(CFLAGS) $(CPPFLAGS) tests/test_ft_printf.c $(NAME) -o $(TEST_BIN)
 	./$(TEST_BIN)
+	$(CC) $(CFLAGS) $(CPPFLAGS) -DFT_PRINTF_TEST_WRITE \
+		tests/test_output_faults.c $(SRC) -o $(FAULT_TEST_BIN)
+	./$(FAULT_TEST_BIN)
 
 clean:
 	$(RM) $(OBJ)
 	rm -rf tests/bin
 
 fclean: clean
-	$(RM) $(NAME) $(TEST_BIN)
+	$(RM) $(NAME) $(TEST_BIN) $(FAULT_TEST_BIN)
 
 re: fclean all
 
diff --git a/tests/test_ft_printf.c b/tests/test_ft_printf.c
index 77e7bcb..ae03124 100644
--- a/tests/test_ft_printf.c
+++ b/tests/test_ft_printf.c
@@ -2,6 +2,7 @@
 
 #include <limits.h>
 #include <stdint.h>
+#include <signal.h>
 #include <stdio.h>
 #include <stdlib.h>
 #include <string.h>
@@ -205,6 +206,50 @@ static void	run_error_cases(void)
 		fail_test(__LINE__, "ft_printf did not report write failure");
 }
 
+static volatile sig_atomic_t	g_sigpipe_count;
+
+static void	count_sigpipe(int signal_number)
+{
+	(void)signal_number;
+	g_sigpipe_count++;
+}
+
+static void	run_sigpipe_policy_case(void)
+{
+	struct sigaction	before;
+	struct sigaction	after;
+	struct sigaction	action;
+	int				pipe_fd[2];
+	int				saved_stdout;
+	int				result;
+
+	memset(&action, 0, sizeof(action));
+	action.sa_handler = count_sigpipe;
+	sigemptyset(&action.sa_mask);
+	if (sigaction(SIGPIPE, &action, &before) < 0)
+		fail_test(__LINE__, "sigaction setup failed");
+	if (pipe(pipe_fd) < 0)
+		fail_test(__LINE__, "pipe for SIGPIPE test failed");
+	close(pipe_fd[0]);
+	saved_stdout = dup(STDOUT_FILENO);
+	if (saved_stdout < 0 || dup2(pipe_fd[1], STDOUT_FILENO) < 0)
+		fail_test(__LINE__, "stdout setup for SIGPIPE test failed");
+	close(pipe_fd[1]);
+	g_sigpipe_count = 0;
+	result = ft_printf("broken pipe");
+	if (dup2(saved_stdout, STDOUT_FILENO) < 0)
+		fail_test(__LINE__, "stdout restore after SIGPIPE failed");
+	close(saved_stdout);
+	if (sigaction(SIGPIPE, 0, &after) < 0)
+		fail_test(__LINE__, "sigaction readback failed");
+	if (sigaction(SIGPIPE, &before, 0) < 0)
+		fail_test(__LINE__, "sigaction restore failed");
+	if (result != -1 || g_sigpipe_count != 1)
+		fail_test(__LINE__, "SIGPIPE write failure was not reported");
+	if (after.sa_handler != count_sigpipe)
+		fail_test(__LINE__, "library changed the SIGPIPE policy");
+}
+
 int	main(void)
 {
 	run_core_cases();
@@ -212,6 +257,7 @@ int	main(void)
 	run_parser_boundary_cases();
 	run_numeric_layout_cases();
 	run_error_cases();
+	run_sigpipe_policy_case();
 	dprintf(STDERR_FILENO, "ft_printf tests passed\n");
 	return (0);
 }
diff --git a/tests/test_output_faults.c b/tests/test_output_faults.c
new file mode 100644
index 0000000..e8c528e
--- /dev/null
+++ b/tests/test_output_faults.c
@@ -0,0 +1,177 @@
+#include "ft_printf.h"
+
+#include <errno.h>
+#include <stdio.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+typedef enum e_write_action
+{
+	WRITE_ALL,
+	WRITE_PART,
+	WRITE_EINTR,
+	WRITE_EPIPE,
+	WRITE_ZERO
+}	t_write_action;
+
+typedef struct s_write_step
+{
+	t_write_action	action;
+	size_t			amount;
+}	t_write_step;
+
+static t_write_step	g_steps[16];
+static int			g_step_count;
+static int			g_step_index;
+static int			g_write_calls;
+static size_t		g_largest_write;
+static char			g_output[4096];
+static size_t		g_output_length;
+
+static void	fail_test(int line, const char *message)
+{
+	dprintf(STDERR_FILENO, "line %d: %s\n", line, message);
+	exit(1);
+}
+
+static void	reset_writer(void)
+{
+	g_step_count = 0;
+	g_step_index = 0;
+	g_write_calls = 0;
+	g_largest_write = 0;
+	g_output_length = 0;
+}
+
+static void	add_step(t_write_action action, size_t amount)
+{
+	if (g_step_count >= (int)(sizeof(g_steps) / sizeof(g_steps[0])))
+		fail_test(__LINE__, "too many write steps");
+	g_steps[g_step_count].action = action;
+	g_steps[g_step_count].amount = amount;
+	g_step_count++;
+}
+
+static ssize_t	record_bytes(const void *buffer, size_t length)
+{
+	if (g_output_length + length > sizeof(g_output))
+		fail_test(__LINE__, "mock output buffer is full");
+	memcpy(g_output + g_output_length, buffer, length);
+	g_output_length += length;
+	return ((ssize_t)length);
+}
+
+ssize_t	ft_printf_test_write(int fd, const void *buffer, size_t length)
+{
+	t_write_step	step;
+	size_t			amount;
+
+	(void)fd;
+	g_write_calls++;
+	if (length > g_largest_write)
+		g_largest_write = length;
+	step.action = WRITE_ALL;
+	step.amount = 0;
+	if (g_step_index < g_step_count)
+		step = g_steps[g_step_index++];
+	if (step.action == WRITE_EINTR)
+	{
+		errno = EINTR;
+		return (-1);
+	}
+	if (step.action == WRITE_EPIPE)
+	{
+		errno = EPIPE;
+		return (-1);
+	}
+	if (step.action == WRITE_ZERO)
+		return (0);
+	if (step.action == WRITE_PART)
+	{
+		amount = step.amount;
+		if (amount > length)
+			amount = length;
+		return (record_bytes(buffer, amount));
+	}
+	return (record_bytes(buffer, length));
+}
+
+static void	expect_success(const char *expected, int expected_calls,
+		size_t expected_largest_write)
+{
+	int	result;
+
+	result = ft_printf("%s", expected);
+	if (result != (int)strlen(expected))
+		fail_test(__LINE__, "successful output returned the wrong length");
+	if (g_output_length != strlen(expected)
+		|| memcmp(g_output, expected, g_output_length) != 0)
+		fail_test(__LINE__, "successful output bytes do not match");
+	if (g_write_calls != expected_calls)
+		fail_test(__LINE__, "unexpected write call count");
+	if (g_largest_write != expected_largest_write)
+		fail_test(__LINE__, "unexpected largest write size");
+}
+
+static void	run_retry_cases(void)
+{
+	reset_writer();
+	add_step(WRITE_PART, 2);
+	add_step(WRITE_ALL, 0);
+	expect_success("partial", 2, 7);
+	reset_writer();
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_PART, 3);
+	add_step(WRITE_EINTR, 0);
+	add_step(WRITE_ALL, 0);
+	expect_success("interrupt", 4, 9);
+}
+
+static void	run_failure_cases(void)
+{
+	int	result;
+
+	reset_writer();
+	add_step(WRITE_EPIPE, 0);
+	result = ft_printf("broken");
+	if (result != -1 || g_output_length != 0)
+		fail_test(__LINE__, "EPIPE was not reported");
+	reset_writer();
+	add_step(WRITE_PART, 3);
+	add_step(WRITE_EPIPE, 0);
+	result = ft_printf("%s", "partial failure");
+	if (result != -1 || g_output_length != 3
+		|| memcmp(g_output, "par", 3) != 0)
+		fail_test(__LINE__, "failure after a partial write was not preserved");
+	reset_writer();
+	add_step(WRITE_ZERO, 0);
+	result = ft_printf("zero");
+	if (result != -1 || g_output_length != 0)
+		fail_test(__LINE__, "zero-byte write was not reported");
+}
+
+static void	run_padding_case(void)
+{
+	int	result;
+
+	reset_writer();
+	result = ft_printf("%1000d", 7);
+	if (result != 1000 || g_output_length != 1000)
+		fail_test(__LINE__, "padded output length mismatch");
+	if (g_write_calls != 17)
+		fail_test(__LINE__, "padding was not emitted in bounded chunks");
+	if (g_largest_write != 64)
+		fail_test(__LINE__, "padding chunk size changed");
+	if (g_output[998] != ' ' || g_output[999] != '7')
+		fail_test(__LINE__, "padded output bytes do not match");
+}
+
+int	main(void)
+{
+	run_retry_cases();
+	run_failure_cases();
+	run_padding_case();
+	dprintf(STDERR_FILENO, "ft_printf output fault tests passed\n");
+	return (0);
+}
