# Checker 명령 프레이밍·재생·판정

## `feat(checker): 표준 입력 명령 프레임을 읽음`

diff --git a/Makefile b/Makefile
index 188e2d8..d952264 100644
--- a/Makefile
+++ b/Makefile
@@ -12,11 +12,13 @@ COMMON_SRCS := \
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 PUSH_SRCS := src/push_swap.c src/sort.c
 PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+CHECKER_SRCS := src/checker_reader.c
+CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
 .PHONY: all clean fclean re test
 
-all: $(COMMON_OBJS) $(PUSH_OBJS) $(NAME)
+all: $(COMMON_OBJS) $(PUSH_OBJS) $(CHECKER_OBJS) $(NAME)
 
 $(NAME): $(COMMON_OBJS) $(PUSH_OBJS)
 	$(CC) $(CFLAGS) $(COMMON_OBJS) $(PUSH_OBJS) -o $@
diff --git a/include/push_swap.h b/include/push_swap.h
index 6b94c4e..4dad536 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -40,6 +40,8 @@ void	op_rrr(t_stack *a, t_stack *b, int emit);
 int		parse_input(int argc, char **argv, t_stack *a);
 void	sort_stack(t_stack *a, t_stack *b);
 
+int		read_next_line(int fd, char **line);
+
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
 void	ps_putstr_fd(int fd, const char *str);
diff --git a/src/checker_reader.c b/src/checker_reader.c
new file mode 100644
index 0000000..722a2c5
--- /dev/null
+++ b/src/checker_reader.c
@@ -0,0 +1,60 @@
+#include "push_swap.h"
+
+static int	grow_line(char **line, size_t *capacity, size_t needed)
+{
+	char	*next;
+	size_t	new_capacity;
+	size_t	i;
+
+	if (*capacity > needed)
+		return (1);
+	new_capacity = 32;
+	if (*capacity != 0)
+		new_capacity = *capacity;
+	while (new_capacity <= needed)
+		new_capacity *= 2;
+	next = (char *)malloc(new_capacity);
+	if (next == NULL)
+		return (0);
+	i = 0;
+	while (i < *capacity && *line != NULL)
+	{
+		next[i] = (*line)[i];
+		i++;
+	}
+	free(*line);
+	*line = next;
+	*capacity = new_capacity;
+	return (1);
+}
+
+int	read_next_line(int fd, char **line)
+{
+	char	c;
+	ssize_t	bytes;
+	size_t	len;
+	size_t	capacity;
+
+	*line = NULL;
+	len = 0;
+	capacity = 0;
+	while (1)
+	{
+		bytes = read(fd, &c, 1);
+		if (bytes < 0)
+			return (free(*line), -1);
+		if (bytes == 0)
+			break ;
+		if (c == '\n')
+			break ;
+		if (!grow_line(line, &capacity, len + 1))
+			return (free(*line), -1);
+		(*line)[len++] = c;
+	}
+	if (bytes == 0 && len == 0)
+		return (free(*line), 0);
+	if (!grow_line(line, &capacity, len + 1))
+		return (free(*line), -1);
+	(*line)[len] = '\0';
+	return (1);
+}


## `feat(checker): 스택 연산 명령을 해석`

diff --git a/Makefile b/Makefile
index d952264..3feda17 100644
--- a/Makefile
+++ b/Makefile
@@ -12,7 +12,7 @@ COMMON_SRCS := \
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 PUSH_SRCS := src/push_swap.c src/sort.c
 PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
-CHECKER_SRCS := src/checker_reader.c
+CHECKER_SRCS := src/checker.c src/checker_reader.c
 CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
diff --git a/include/push_swap.h b/include/push_swap.h
index 4dad536..1305e65 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -41,6 +41,7 @@ int		parse_input(int argc, char **argv, t_stack *a);
 void	sort_stack(t_stack *a, t_stack *b);
 
 int		read_next_line(int fd, char **line);
+int		apply_checker_command(const char *line, t_stack *a, t_stack *b);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/checker.c b/src/checker.c
new file mode 100644
index 0000000..6a06235
--- /dev/null
+++ b/src/checker.c
@@ -0,0 +1,30 @@
+#include "push_swap.h"
+
+int	apply_checker_command(const char *line, t_stack *a, t_stack *b)
+{
+	if (ps_strcmp(line, "sa") == 0)
+		op_sa(a, 0);
+	else if (ps_strcmp(line, "sb") == 0)
+		op_sb(b, 0);
+	else if (ps_strcmp(line, "ss") == 0)
+		op_ss(a, b, 0);
+	else if (ps_strcmp(line, "pa") == 0)
+		op_pa(a, b, 0);
+	else if (ps_strcmp(line, "pb") == 0)
+		op_pb(a, b, 0);
+	else if (ps_strcmp(line, "ra") == 0)
+		op_ra(a, 0);
+	else if (ps_strcmp(line, "rb") == 0)
+		op_rb(b, 0);
+	else if (ps_strcmp(line, "rr") == 0)
+		op_rr(a, b, 0);
+	else if (ps_strcmp(line, "rra") == 0)
+		op_rra(a, 0);
+	else if (ps_strcmp(line, "rrb") == 0)
+		op_rrb(b, 0);
+	else if (ps_strcmp(line, "rrr") == 0)
+		op_rrr(a, b, 0);
+	else
+		return (0);
+	return (1);
+}


## `feat(checker): 명령 실행 결과를 판정`

diff --git a/Makefile b/Makefile
index 3feda17..f90afbd 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,5 @@
 NAME := push_swap
+CHECKER := checker
 CC := cc
 CFLAGS := -std=c99 -Wall -Wextra -Werror -Wpedantic
 CPPFLAGS := -Iinclude
@@ -18,11 +19,14 @@ OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
 .PHONY: all clean fclean re test
 
-all: $(COMMON_OBJS) $(PUSH_OBJS) $(CHECKER_OBJS) $(NAME)
+all: $(COMMON_OBJS) $(PUSH_OBJS) $(CHECKER_OBJS) $(NAME) $(CHECKER)
 
 $(NAME): $(COMMON_OBJS) $(PUSH_OBJS)
 	$(CC) $(CFLAGS) $(COMMON_OBJS) $(PUSH_OBJS) -o $@
 
+$(CHECKER): $(COMMON_OBJS) $(CHECKER_OBJS)
+	$(CC) $(CFLAGS) $(COMMON_OBJS) $(CHECKER_OBJS) -o $@
+
 $(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
 
diff --git a/src/checker.c b/src/checker.c
index 6a06235..687a44b 100644
--- a/src/checker.c
+++ b/src/checker.c
@@ -28,3 +28,51 @@ int	apply_checker_command(const char *line, t_stack *a, t_stack *b)
 		return (0);
 	return (1);
 }
+
+static int	read_and_apply(t_stack *a, t_stack *b)
+{
+	char	*line;
+	int		status;
+
+	status = read_next_line(0, &line);
+	while (status > 0)
+	{
+		if (!apply_checker_command(line, a, b))
+		{
+			free(line);
+			return (0);
+		}
+		free(line);
+		status = read_next_line(0, &line);
+	}
+	return (status == 0);
+}
+
+int	main(int argc, char **argv)
+{
+	t_stack	a;
+	t_stack	b;
+
+	if (argc == 1)
+		return (0);
+	if (!parse_input(argc, argv, &a))
+		return (write_error(), 1);
+	if (!stack_init(&b, a.capacity))
+	{
+		stack_free(&a);
+		return (write_error(), 1);
+	}
+	if (!read_and_apply(&a, &b))
+	{
+		stack_free(&a);
+		stack_free(&b);
+		return (write_error(), 1);
+	}
+	if (stack_is_complete_sorted(&a, &b))
+		ps_putstr_fd(1, "OK\n");
+	else
+		ps_putstr_fd(1, "KO\n");
+	stack_free(&a);
+	stack_free(&b);
+	return (0);
+}


## `test(checker): 명령 연산과 최종 판정을 검증`

diff --git a/tests/run_tests.py b/tests/run_tests.py
index 3087061..5b3549b 100644
--- a/tests/run_tests.py
+++ b/tests/run_tests.py
@@ -130,10 +130,45 @@ def test_checker_without_values_does_not_read_stdin():
         assert_equal(input_file.tell(), 0, "checker without values leaves stdin unread")
 
 
+def checker_ok(args, program, label):
+    result = run([CHECKER] + args, program)
+    assert_equal(result.returncode, 0, f"{label} checker exit")
+    assert_equal(result.stdout, "OK\n", f"{label} checker stdout")
+    assert_equal(result.stderr, "", f"{label} checker stderr")
+
+
+def test_checker_operations():
+    cases = [
+        ("sa", ["2", "1"], "sa\n"),
+        ("sb", ["2", "1", "3"], "pb\npb\nsb\npa\npa\n"),
+        ("ss", ["2", "1", "4", "3"], "pb\npb\nss\npa\npa\n"),
+        ("pa-pb", ["1", "2"], "pb\npa\n"),
+        ("ra", ["3", "1", "2"], "ra\n"),
+        ("rb", ["2", "1", "3"], "pb\npb\nrb\npa\npa\n"),
+        ("rr", ["2", "1", "4", "3"], "pb\npb\nrr\npa\npa\n"),
+        ("rra", ["2", "3", "1"], "rra\n"),
+        ("rrb", ["2", "1", "3"], "pb\npb\nrrb\npa\npa\n"),
+        ("rrr", ["2", "1", "4", "3"], "pb\npb\nrrr\npa\npa\n"),
+    ]
+    for label, args, program in cases:
+        checker_ok(args, program, label)
+
+    ko = run([CHECKER, "2", "1"], "")
+    assert_equal(ko.returncode, 0, "unsorted checker input exits cleanly")
+    assert_equal(ko.stdout, "KO\n", "unsorted checker input reports KO")
+    assert_equal(ko.stderr, "", "unsorted checker input has no stderr")
+
+    invalid = run([CHECKER, "1", "2"], "ra\nwat\n")
+    assert_ok(invalid.returncode != 0, "invalid checker command fails")
+    assert_equal(invalid.stdout, "", "invalid checker command has no stdout")
+    assert_equal(invalid.stderr, "Error\n", "invalid checker command reports Error")
+
+
 def main():
     test_parser_inputs()
     test_parser_boundaries()
     test_checker_without_values_does_not_read_stdin()
+    test_checker_operations()
     print("tests passed")
 
 


## `fix(checker): 명령 길이를 제한하고 중단된 읽기를 재시도`

diff --git a/include/push_swap.h b/include/push_swap.h
index 751f76a..2322d88 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -6,6 +6,8 @@
 # include <stdlib.h>
 # include <unistd.h>
 
+# define PS_COMMAND_MAX 3
+
 typedef struct s_stack
 {
 	int	*values;
diff --git a/src/checker_reader.c b/src/checker_reader.c
index d439cbc..c2fe03d 100644
--- a/src/checker_reader.c
+++ b/src/checker_reader.c
@@ -1,60 +1,34 @@
 #include "push_swap.h"
 
-static int	grow_line(char **line, size_t *capacity, size_t needed)
-{
-	char	*next;
-	size_t	new_capacity;
-	size_t	i;
-
-	if (*capacity > needed)
-		return (1);
-	new_capacity = 32;
-	if (*capacity != 0)
-		new_capacity = *capacity;
-	while (new_capacity <= needed)
-		new_capacity *= 2;
-	next = (char *)ps_malloc(new_capacity);
-	if (next == NULL)
-		return (0);
-	i = 0;
-	while (i < *capacity && *line != NULL)
-	{
-		next[i] = (*line)[i];
-		i++;
-	}
-	ps_free(*line);
-	*line = next;
-	*capacity = new_capacity;
-	return (1);
-}
+#include <errno.h>
 
 int	read_next_line(int fd, char **line)
 {
 	char	c;
 	ssize_t	bytes;
 	size_t	len;
-	size_t	capacity;
 
-	*line = NULL;
+	*line = (char *)ps_malloc(PS_COMMAND_MAX + 1);
+	if (*line == NULL)
+		return (-1);
 	len = 0;
-	capacity = 0;
 	while (1)
 	{
 		bytes = ps_read(fd, &c, 1);
+		if (bytes < 0 && errno == EINTR)
+			continue ;
 		if (bytes < 0)
-			return (ps_free(*line), -1);
+			return (ps_free(*line), *line = NULL, -1);
 		if (bytes == 0)
 			break ;
 		if (c == '\n')
 			break ;
-		if (!grow_line(line, &capacity, len + 1))
-			return (ps_free(*line), -1);
+		if (c == '\0' || len >= PS_COMMAND_MAX)
+			return (ps_free(*line), *line = NULL, -1);
 		(*line)[len++] = c;
 	}
 	if (bytes == 0 && len == 0)
-		return (ps_free(*line), 0);
-	if (!grow_line(line, &capacity, len + 1))
-		return (ps_free(*line), -1);
+		return (ps_free(*line), *line = NULL, 0);
 	(*line)[len] = '\0';
 	return (1);
 }
diff --git a/tests/fault_tests.py b/tests/fault_tests.py
index 71b96d3..4bc38ec 100644
--- a/tests/fault_tests.py
+++ b/tests/fault_tests.py
@@ -71,7 +71,7 @@ def test_nth_allocation_failures():
     )
     assert_equal(success.returncode, 0, "push_swap past-last malloc succeeds")
 
-    for index in range(1, 7):
+    for index in range(1, 8):
         result = run(
             CHECKER,
             ["2", "1"],
@@ -84,7 +84,7 @@ def test_nth_allocation_failures():
         CHECKER,
         ["2", "1"],
         b"sa\n",
-        faults={"PS_FAIL_MALLOC_AT": 7},
+        faults={"PS_FAIL_MALLOC_AT": 8},
     )
     assert_equal(success.returncode, 0, "checker past-last malloc succeeds")
     assert_equal(success.stdout, b"OK\n", "checker succeeds after allocation sweep")


## `test(checker): 읽기 실패와 명령 경계를 검증`

diff --git a/src/runtime.c b/src/runtime.c
index dc1dc32..a4e88b9 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -16,6 +16,7 @@ typedef union u_allocation_header
 }	t_allocation_header;
 
 static unsigned long	g_malloc_calls;
+static unsigned long	g_read_calls;
 static unsigned long	g_live_allocations;
 
 static unsigned long	read_index(const char *name)
@@ -89,6 +90,13 @@ void	ps_free(void *pointer)
 
 ssize_t	ps_read(int fd, void *buffer, size_t count)
 {
+#ifdef PS_FAULT_INJECTION
+	g_read_calls++;
+	if (at_index("PS_EINTR_READ_AT", g_read_calls))
+		return (errno = EINTR, -1);
+	if (at_index("PS_FAIL_READ_AT", g_read_calls))
+		return (errno = EIO, -1);
+#endif
 	return (read(fd, buffer, count));
 }
 
diff --git a/tests/fault_tests.py b/tests/fault_tests.py
index 4bc38ec..1a104d4 100644
--- a/tests/fault_tests.py
+++ b/tests/fault_tests.py
@@ -90,8 +90,48 @@ def test_nth_allocation_failures():
     assert_equal(success.stdout, b"OK\n", "checker succeeds after allocation sweep")
 
 
+def test_read_failures_and_command_bounds():
+    for index in range(1, 5):
+        result = run(
+            CHECKER,
+            ["2", "1"],
+            b"sa\n",
+            faults={"PS_FAIL_READ_AT": index},
+        )
+        assert_true(result.returncode != 0, f"checker read #{index} fails")
+        assert_equal(result.stderr, b"Error\n", "checker read error")
+    for index in (1, 4):
+        result = run(
+            CHECKER,
+            ["2", "1"],
+            b"sa\n",
+            faults={"PS_EINTR_READ_AT": index},
+        )
+        assert_equal(result.returncode, 0, f"checker retries read EINTR #{index}")
+        assert_equal(result.stdout, b"OK\n", "checker result after read EINTR")
+
+    invalid_streams = [
+        b"sa\x00pb\n",
+        b"rrrr\n",
+        b"rrrr",
+        b"\x00\n",
+        b"\n",
+        b"r" * 65536 + b"\n",
+    ]
+    for stream in invalid_streams:
+        result = run(CHECKER, ["2", "1"], stream)
+        assert_true(result.returncode != 0, f"checker rejects {stream!r}")
+        assert_equal(result.stdout, b"", "invalid command has no stdout")
+        assert_equal(result.stderr, b"Error\n", "invalid command reports Error")
+
+    unterminated = run(CHECKER, ["2", "1"], b"sa")
+    assert_equal(unterminated.returncode, 0, "unterminated valid command succeeds")
+    assert_equal(unterminated.stdout, b"OK\n", "unterminated command is applied")
+
+
 def main():
     test_nth_allocation_failures()
+    test_read_failures_and_command_bounds()
     print("fault injection tests passed")
 
 
