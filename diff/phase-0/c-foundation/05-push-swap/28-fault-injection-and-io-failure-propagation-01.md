# 장애 주입과 I/O 실패 전파

## `refactor(runtime): 메모리와 입력 시스템 호출을 공통화`

diff --git a/Makefile b/Makefile
index 894111f..a9c2643 100644
--- a/Makefile
+++ b/Makefile
@@ -11,6 +11,7 @@ COMMON_SRCS := \
 	src/parser.c \
 	src/stack.c \
 	src/operations.c \
+	src/runtime.c \
 	src/utils.c
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 PUSH_SRCS := src/push_swap.c src/sort.c
@@ -33,7 +34,9 @@ $(CHECKER): $(COMMON_OBJS) $(CHECKER_OBJS)
 $(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
 
-$(OPERATION_TEST): tests/operation_invariants.c $(COMMON_OBJS)
+$(OPERATION_TEST): tests/operation_invariants.c \
+		$(OBJ_DIR)/stack.o $(OBJ_DIR)/operations.o $(OBJ_DIR)/runtime.o \
+		$(OBJ_DIR)/utils.o
 	$(CC) $(CFLAGS) $(CPPFLAGS) $^ -o $@
 
 $(OBJ_DIR):
diff --git a/include/push_swap.h b/include/push_swap.h
index 1305e65..ec8752e 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -43,6 +43,10 @@ void	sort_stack(t_stack *a, t_stack *b);
 int		read_next_line(int fd, char **line);
 int		apply_checker_command(const char *line, t_stack *a, t_stack *b);
 
+void	*ps_malloc(size_t size);
+void	ps_free(void *pointer);
+ssize_t	ps_read(int fd, void *buffer, size_t count);
+
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
 void	ps_putstr_fd(int fd, const char *str);
diff --git a/src/checker.c b/src/checker.c
index 687a44b..d9566ba 100644
--- a/src/checker.c
+++ b/src/checker.c
@@ -39,10 +39,10 @@ static int	read_and_apply(t_stack *a, t_stack *b)
 	{
 		if (!apply_checker_command(line, a, b))
 		{
-			free(line);
+			ps_free(line);
 			return (0);
 		}
-		free(line);
+		ps_free(line);
 		status = read_next_line(0, &line);
 	}
 	return (status == 0);
diff --git a/src/checker_reader.c b/src/checker_reader.c
index 722a2c5..d439cbc 100644
--- a/src/checker_reader.c
+++ b/src/checker_reader.c
@@ -13,7 +13,7 @@ static int	grow_line(char **line, size_t *capacity, size_t needed)
 		new_capacity = *capacity;
 	while (new_capacity <= needed)
 		new_capacity *= 2;
-	next = (char *)malloc(new_capacity);
+	next = (char *)ps_malloc(new_capacity);
 	if (next == NULL)
 		return (0);
 	i = 0;
@@ -22,7 +22,7 @@ static int	grow_line(char **line, size_t *capacity, size_t needed)
 		next[i] = (*line)[i];
 		i++;
 	}
-	free(*line);
+	ps_free(*line);
 	*line = next;
 	*capacity = new_capacity;
 	return (1);
@@ -40,21 +40,21 @@ int	read_next_line(int fd, char **line)
 	capacity = 0;
 	while (1)
 	{
-		bytes = read(fd, &c, 1);
+		bytes = ps_read(fd, &c, 1);
 		if (bytes < 0)
-			return (free(*line), -1);
+			return (ps_free(*line), -1);
 		if (bytes == 0)
 			break ;
 		if (c == '\n')
 			break ;
 		if (!grow_line(line, &capacity, len + 1))
-			return (free(*line), -1);
+			return (ps_free(*line), -1);
 		(*line)[len++] = c;
 	}
 	if (bytes == 0 && len == 0)
-		return (free(*line), 0);
+		return (ps_free(*line), 0);
 	if (!grow_line(line, &capacity, len + 1))
-		return (free(*line), -1);
+		return (ps_free(*line), -1);
 	(*line)[len] = '\0';
 	return (1);
 }
diff --git a/src/parser.c b/src/parser.c
index 1efed98..c3d2412 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -142,7 +142,7 @@ static int	assign_ranks(t_stack *a)
 	int	*sorted;
 	int	i;
 
-	sorted = (int *)malloc(sizeof(int) * (size_t)a->size);
+	sorted = (int *)ps_malloc(sizeof(int) * (size_t)a->size);
 	if (sorted == NULL)
 		return (0);
 	i = 0;
@@ -157,7 +157,7 @@ static int	assign_ranks(t_stack *a)
 	{
 		if (sorted[i - 1] == sorted[i])
 		{
-			free(sorted);
+			ps_free(sorted);
 			return (0);
 		}
 		i++;
@@ -168,7 +168,7 @@ static int	assign_ranks(t_stack *a)
 		a->ranks[i] = find_rank(sorted, a->size, a->values[i]);
 		i++;
 	}
-	free(sorted);
+	ps_free(sorted);
 	return (1);
 }
 
diff --git a/src/runtime.c b/src/runtime.c
new file mode 100644
index 0000000..1eb2459
--- /dev/null
+++ b/src/runtime.c
@@ -0,0 +1,16 @@
+#include "push_swap.h"
+
+void	*ps_malloc(size_t size)
+{
+	return (malloc(size));
+}
+
+void	ps_free(void *pointer)
+{
+	free(pointer);
+}
+
+ssize_t	ps_read(int fd, void *buffer, size_t count)
+{
+	return (read(fd, buffer, count));
+}
diff --git a/src/stack.c b/src/stack.c
index 6be2997..f76604b 100644
--- a/src/stack.c
+++ b/src/stack.c
@@ -13,8 +13,8 @@ int	stack_init(t_stack *stack, int capacity)
 	stack_init_empty(stack);
 	if (capacity <= 0)
 		return (1);
-	stack->values = (int *)malloc(sizeof(int) * (size_t)capacity);
-	stack->ranks = (int *)malloc(sizeof(int) * (size_t)capacity);
+	stack->values = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
+	stack->ranks = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
 	if (stack->values == NULL || stack->ranks == NULL)
 	{
 		stack_free(stack);
@@ -26,8 +26,8 @@ int	stack_init(t_stack *stack, int capacity)
 
 void	stack_free(t_stack *stack)
 {
-	free(stack->values);
-	free(stack->ranks);
+	ps_free(stack->values);
+	ps_free(stack->ranks);
 	stack_init_empty(stack);
 }
 


## `test(memory): 할당 실패 뒤 자원 정리를 검증`

diff --git a/Makefile b/Makefile
index a9c2643..def63e4 100644
--- a/Makefile
+++ b/Makefile
@@ -19,6 +19,12 @@ PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 CHECKER_SRCS := src/checker.c src/checker_reader.c
 CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
+FAULT_DIR := $(OBJ_DIR)/fault
+FAULT_COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(FAULT_DIR)/%.o)
+FAULT_PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(FAULT_DIR)/%.o)
+FAULT_CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(FAULT_DIR)/%.o)
+FAULT_PUSH_SWAP := $(FAULT_DIR)/push_swap
+FAULT_CHECKER := $(FAULT_DIR)/checker
 
 .DELETE_ON_ERROR:
 .PHONY: all clean fclean re test
@@ -39,9 +45,21 @@ $(OPERATION_TEST): tests/operation_invariants.c \
 		$(OBJ_DIR)/utils.o
 	$(CC) $(CFLAGS) $(CPPFLAGS) $^ -o $@
 
+$(FAULT_PUSH_SWAP): $(FAULT_COMMON_OBJS) $(FAULT_PUSH_OBJS)
+	$(CC) $(CFLAGS) $^ -o $@
+
+$(FAULT_CHECKER): $(FAULT_COMMON_OBJS) $(FAULT_CHECKER_OBJS)
+	$(CC) $(CFLAGS) $^ -o $@
+
+$(FAULT_DIR)/%.o: src/%.c include/push_swap.h | $(FAULT_DIR)
+	$(CC) $(CFLAGS) $(CPPFLAGS) -DPS_FAULT_INJECTION -c $< -o $@
+
 $(OBJ_DIR):
 	mkdir -p $(OBJ_DIR)
 
+$(FAULT_DIR): | $(OBJ_DIR)
+	mkdir -p $(FAULT_DIR)
+
 clean:
 	$(RMDIR) $(OBJ_DIR) tests/__pycache__ .pytest_cache
 
@@ -50,6 +68,8 @@ fclean: clean
 
 re: fclean all
 
-test: all $(OPERATION_TEST)
+test: all $(OPERATION_TEST) $(FAULT_PUSH_SWAP) $(FAULT_CHECKER)
 	$(OPERATION_TEST)
 	python3 tests/run_tests.py
+	PS_PUSH_SWAP=$(FAULT_PUSH_SWAP) PS_CHECKER=$(FAULT_CHECKER) \
+		python3 tests/fault_tests.py
diff --git a/include/push_swap.h b/include/push_swap.h
index ec8752e..751f76a 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -46,6 +46,7 @@ int		apply_checker_command(const char *line, t_stack *a, t_stack *b);
 void	*ps_malloc(size_t size);
 void	ps_free(void *pointer);
 ssize_t	ps_read(int fd, void *buffer, size_t count);
+int		ps_test_finish(int status);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/checker.c b/src/checker.c
index d9566ba..9016a34 100644
--- a/src/checker.c
+++ b/src/checker.c
@@ -54,19 +54,19 @@ int	main(int argc, char **argv)
 	t_stack	b;
 
 	if (argc == 1)
-		return (0);
+		return (ps_test_finish(0));
 	if (!parse_input(argc, argv, &a))
-		return (write_error(), 1);
+		return (write_error(), ps_test_finish(1));
 	if (!stack_init(&b, a.capacity))
 	{
 		stack_free(&a);
-		return (write_error(), 1);
+		return (write_error(), ps_test_finish(1));
 	}
 	if (!read_and_apply(&a, &b))
 	{
 		stack_free(&a);
 		stack_free(&b);
-		return (write_error(), 1);
+		return (write_error(), ps_test_finish(1));
 	}
 	if (stack_is_complete_sorted(&a, &b))
 		ps_putstr_fd(1, "OK\n");
@@ -74,5 +74,5 @@ int	main(int argc, char **argv)
 		ps_putstr_fd(1, "KO\n");
 	stack_free(&a);
 	stack_free(&b);
-	return (0);
+	return (ps_test_finish(0));
 }
diff --git a/src/push_swap.c b/src/push_swap.c
index c1ec597..8f04725 100644
--- a/src/push_swap.c
+++ b/src/push_swap.c
@@ -8,16 +8,16 @@ int	main(int argc, char **argv)
 	if (!parse_input(argc, argv, &a))
 	{
 		write_error();
-		return (1);
+		return (ps_test_finish(1));
 	}
 	if (!stack_init(&b, a.capacity))
 	{
 		stack_free(&a);
 		write_error();
-		return (1);
+		return (ps_test_finish(1));
 	}
 	sort_stack(&a, &b);
 	stack_free(&a);
 	stack_free(&b);
-	return (0);
+	return (ps_test_finish(0));
 }
diff --git a/src/runtime.c b/src/runtime.c
index 1eb2459..dc1dc32 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -1,16 +1,132 @@
 #include "push_swap.h"
 
+#include <errno.h>
+
+#ifdef PS_FAULT_INJECTION
+
+typedef union u_allocation_header
+{
+	struct
+	{
+		size_t			size;
+		unsigned long	magic;
+	}	data;
+	long double	align_long_double;
+	void		*align_pointer;
+}	t_allocation_header;
+
+static unsigned long	g_malloc_calls;
+static unsigned long	g_live_allocations;
+
+static unsigned long	read_index(const char *name)
+{
+	const char		*value;
+	unsigned long	index;
+
+	value = getenv(name);
+	if (value == NULL || *value == '\0')
+		return (0);
+	index = 0;
+	while (*value >= '0' && *value <= '9')
+	{
+		if (index > (ULONG_MAX - (unsigned long)(*value - '0')) / 10)
+			return (0);
+		index = index * 10 + (unsigned long)(*value - '0');
+		value++;
+	}
+	if (*value != '\0')
+		return (0);
+	return (index);
+}
+
+static int	at_index(const char *name, unsigned long call)
+{
+	return (read_index(name) == call && call != 0);
+}
+
+#endif
+
 void	*ps_malloc(size_t size)
 {
+#ifdef PS_FAULT_INJECTION
+	t_allocation_header	*header;
+
+	g_malloc_calls++;
+	if (at_index("PS_FAIL_MALLOC_AT", g_malloc_calls))
+		return (NULL);
+	if (size > (size_t)-1 - sizeof(*header))
+		return (NULL);
+	header = (t_allocation_header *)malloc(sizeof(*header) + size);
+	if (header == NULL)
+		return (NULL);
+	header->data.size = size;
+	header->data.magic = 0x50535354UL;
+	g_live_allocations++;
+	return ((void *)(header + 1));
+#else
 	return (malloc(size));
+#endif
 }
 
 void	ps_free(void *pointer)
 {
+#ifdef PS_FAULT_INJECTION
+	t_allocation_header	*header;
+
+	if (pointer == NULL)
+		return ;
+	header = ((t_allocation_header *)pointer) - 1;
+	if (header->data.magic == 0x50535354UL)
+	{
+		header->data.magic = 0;
+		g_live_allocations--;
+	}
+	free(header);
+#else
 	free(pointer);
+#endif
 }
 
 ssize_t	ps_read(int fd, void *buffer, size_t count)
 {
 	return (read(fd, buffer, count));
 }
+
+#ifdef PS_FAULT_INJECTION
+static void	raw_report(const char *message)
+{
+	size_t	length;
+	ssize_t	written;
+
+	length = 0;
+	while (message[length] != '\0')
+		length++;
+	while (length > 0)
+	{
+		written = write(2, message, length);
+		if (written < 0 && errno == EINTR)
+			continue ;
+		if (written <= 0)
+			return ;
+		message += written;
+		length -= (size_t)written;
+	}
+}
+#endif
+
+int	ps_test_finish(int status)
+{
+#ifdef PS_FAULT_INJECTION
+	if (getenv("PS_REPORT_ALLOCATIONS") != NULL)
+	{
+		if (g_live_allocations == 0)
+			raw_report("PS_LIVE_ALLOCATIONS=0\n");
+		else
+		{
+			raw_report("PS_LIVE_ALLOCATIONS=NONZERO\n");
+			return (99);
+		}
+	}
+#endif
+	return (status);
+}
diff --git a/tests/fault_tests.py b/tests/fault_tests.py
new file mode 100644
index 0000000..71b96d3
--- /dev/null
+++ b/tests/fault_tests.py
@@ -0,0 +1,99 @@
+#!/usr/bin/env python3
+import os
+from pathlib import Path
+import subprocess
+import sys
+
+
+ROOT = Path(__file__).resolve().parents[1]
+PUSH_SWAP = ROOT / os.environ.get("PS_PUSH_SWAP", ".build/fault/push_swap")
+CHECKER = ROOT / os.environ.get("PS_CHECKER", ".build/fault/checker")
+ALLOCATION_REPORT = b"PS_LIVE_ALLOCATIONS=0\n"
+
+
+def fail(message):
+    print(message, file=sys.stderr)
+    raise SystemExit(1)
+
+
+def assert_equal(actual, expected, message):
+    if actual != expected:
+        fail(f"{message}: expected {expected!r}, got {actual!r}")
+
+
+def assert_true(condition, message):
+    if not condition:
+        fail(message)
+
+
+def run(binary, args, input_bytes=b"", faults=None):
+    environment = os.environ.copy()
+    environment["PS_REPORT_ALLOCATIONS"] = "1"
+    if faults:
+        environment.update({key: str(value) for key, value in faults.items()})
+    try:
+        result = subprocess.run(
+            [str(binary), *[str(arg) for arg in args]],
+            input=input_bytes,
+            capture_output=True,
+            cwd=ROOT,
+            env=environment,
+            timeout=3,
+            check=False,
+        )
+    except subprocess.TimeoutExpired:
+        fail(f"{binary.name} timed out with faults {faults!r}")
+    assert_true(
+        b"PS_LIVE_ALLOCATIONS=NONZERO" not in result.stderr,
+        f"{binary.name} leaked allocations with faults {faults!r}",
+    )
+    assert_true(
+        ALLOCATION_REPORT in result.stderr,
+        f"{binary.name} did not report cleanup with faults {faults!r}",
+    )
+    result.stderr = result.stderr.replace(ALLOCATION_REPORT, b"")
+    return result
+
+
+def test_nth_allocation_failures():
+    for index in range(1, 6):
+        result = run(
+            PUSH_SWAP,
+            ["4", "3", "2", "1"],
+            faults={"PS_FAIL_MALLOC_AT": index},
+        )
+        assert_true(result.returncode != 0, f"push_swap malloc #{index} fails")
+        assert_equal(result.stderr, b"Error\n", "push_swap allocation error")
+    success = run(
+        PUSH_SWAP,
+        ["4", "3", "2", "1"],
+        faults={"PS_FAIL_MALLOC_AT": 6},
+    )
+    assert_equal(success.returncode, 0, "push_swap past-last malloc succeeds")
+
+    for index in range(1, 7):
+        result = run(
+            CHECKER,
+            ["2", "1"],
+            b"sa\n",
+            faults={"PS_FAIL_MALLOC_AT": index},
+        )
+        assert_true(result.returncode != 0, f"checker malloc #{index} fails")
+        assert_equal(result.stderr, b"Error\n", "checker allocation error")
+    success = run(
+        CHECKER,
+        ["2", "1"],
+        b"sa\n",
+        faults={"PS_FAIL_MALLOC_AT": 7},
+    )
+    assert_equal(success.returncode, 0, "checker past-last malloc succeeds")
+    assert_equal(success.stdout, b"OK\n", "checker succeeds after allocation sweep")
+
+
+def main():
+    test_nth_allocation_failures()
+    print("fault injection tests passed")
+
+
+if __name__ == "__main__":
+    main()


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
 
 


