## `fix(io): 출력 실패를 호출 경로 끝까지 전파`

diff --git a/include/push_swap.h b/include/push_swap.h
index 2322d88..bad20f3 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -27,20 +27,20 @@ void	stack_push(t_stack *dst, t_stack *src);
 void	stack_rotate(t_stack *stack);
 void	stack_reverse_rotate(t_stack *stack);
 
-void	op_sa(t_stack *a, int emit);
-void	op_sb(t_stack *b, int emit);
-void	op_ss(t_stack *a, t_stack *b, int emit);
-void	op_pa(t_stack *a, t_stack *b, int emit);
-void	op_pb(t_stack *a, t_stack *b, int emit);
-void	op_ra(t_stack *a, int emit);
-void	op_rb(t_stack *b, int emit);
-void	op_rr(t_stack *a, t_stack *b, int emit);
-void	op_rra(t_stack *a, int emit);
-void	op_rrb(t_stack *b, int emit);
-void	op_rrr(t_stack *a, t_stack *b, int emit);
+int		op_sa(t_stack *a, int emit);
+int		op_sb(t_stack *b, int emit);
+int		op_ss(t_stack *a, t_stack *b, int emit);
+int		op_pa(t_stack *a, t_stack *b, int emit);
+int		op_pb(t_stack *a, t_stack *b, int emit);
+int		op_ra(t_stack *a, int emit);
+int		op_rb(t_stack *b, int emit);
+int		op_rr(t_stack *a, t_stack *b, int emit);
+int		op_rra(t_stack *a, int emit);
+int		op_rrb(t_stack *b, int emit);
+int		op_rrr(t_stack *a, t_stack *b, int emit);
 
 int		parse_input(int argc, char **argv, t_stack *a);
-void	sort_stack(t_stack *a, t_stack *b);
+int		sort_stack(t_stack *a, t_stack *b);
 
 int		read_next_line(int fd, char **line);
 int		apply_checker_command(const char *line, t_stack *a, t_stack *b);
@@ -48,11 +48,13 @@ int		apply_checker_command(const char *line, t_stack *a, t_stack *b);
 void	*ps_malloc(size_t size);
 void	ps_free(void *pointer);
 ssize_t	ps_read(int fd, void *buffer, size_t count);
+int		ps_write_all(int fd, const void *buffer, size_t count);
+int		ps_ignore_sigpipe(void);
 int		ps_test_finish(int status);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
-void	ps_putstr_fd(int fd, const char *str);
-void	write_error(void);
+int		ps_putstr_fd(int fd, const char *str);
+int		write_error(void);
 
 #endif
diff --git a/src/checker.c b/src/checker.c
index 9016a34..3f2bd2d 100644
--- a/src/checker.c
+++ b/src/checker.c
@@ -3,30 +3,28 @@
 int	apply_checker_command(const char *line, t_stack *a, t_stack *b)
 {
 	if (ps_strcmp(line, "sa") == 0)
-		op_sa(a, 0);
+		return (op_sa(a, 0));
 	else if (ps_strcmp(line, "sb") == 0)
-		op_sb(b, 0);
+		return (op_sb(b, 0));
 	else if (ps_strcmp(line, "ss") == 0)
-		op_ss(a, b, 0);
+		return (op_ss(a, b, 0));
 	else if (ps_strcmp(line, "pa") == 0)
-		op_pa(a, b, 0);
+		return (op_pa(a, b, 0));
 	else if (ps_strcmp(line, "pb") == 0)
-		op_pb(a, b, 0);
+		return (op_pb(a, b, 0));
 	else if (ps_strcmp(line, "ra") == 0)
-		op_ra(a, 0);
+		return (op_ra(a, 0));
 	else if (ps_strcmp(line, "rb") == 0)
-		op_rb(b, 0);
+		return (op_rb(b, 0));
 	else if (ps_strcmp(line, "rr") == 0)
-		op_rr(a, b, 0);
+		return (op_rr(a, b, 0));
 	else if (ps_strcmp(line, "rra") == 0)
-		op_rra(a, 0);
+		return (op_rra(a, 0));
 	else if (ps_strcmp(line, "rrb") == 0)
-		op_rrb(b, 0);
+		return (op_rrb(b, 0));
 	else if (ps_strcmp(line, "rrr") == 0)
-		op_rrr(a, b, 0);
-	else
-		return (0);
-	return (1);
+		return (op_rrr(a, b, 0));
+	return (0);
 }
 
 static int	read_and_apply(t_stack *a, t_stack *b)
@@ -52,9 +50,12 @@ int	main(int argc, char **argv)
 {
 	t_stack	a;
 	t_stack	b;
+	int		status;
 
 	if (argc == 1)
 		return (ps_test_finish(0));
+	if (!ps_ignore_sigpipe())
+		return (write_error(), ps_test_finish(1));
 	if (!parse_input(argc, argv, &a))
 		return (write_error(), ps_test_finish(1));
 	if (!stack_init(&b, a.capacity))
@@ -68,11 +69,14 @@ int	main(int argc, char **argv)
 		stack_free(&b);
 		return (write_error(), ps_test_finish(1));
 	}
+	status = 0;
 	if (stack_is_complete_sorted(&a, &b))
-		ps_putstr_fd(1, "OK\n");
+		status = !ps_putstr_fd(1, "OK\n");
 	else
-		ps_putstr_fd(1, "KO\n");
+		status = !ps_putstr_fd(1, "KO\n");
 	stack_free(&a);
 	stack_free(&b);
-	return (ps_test_finish(0));
+	if (status != 0)
+		write_error();
+	return (ps_test_finish(status));
 }
diff --git a/src/operations.c b/src/operations.c
index bc799f5..15f9963 100644
--- a/src/operations.c
+++ b/src/operations.c
@@ -2,10 +2,11 @@
 
 #include <string.h>
 
-static void	emit_op(const char *name, int emit)
+static int	emit_op(const char *name, int emit)
 {
 	if (emit)
-		ps_putstr_fd(1, name);
+		return (ps_putstr_fd(1, name));
+	return (1);
 }
 
 void	stack_swap(t_stack *stack)
@@ -81,71 +82,71 @@ void	stack_reverse_rotate(t_stack *stack)
 	stack->ranks[0] = rank;
 }
 
-void	op_sa(t_stack *a, int emit)
+int	op_sa(t_stack *a, int emit)
 {
 	stack_swap(a);
-	emit_op("sa\n", emit);
+	return (emit_op("sa\n", emit));
 }
 
-void	op_sb(t_stack *b, int emit)
+int	op_sb(t_stack *b, int emit)
 {
 	stack_swap(b);
-	emit_op("sb\n", emit);
+	return (emit_op("sb\n", emit));
 }
 
-void	op_ss(t_stack *a, t_stack *b, int emit)
+int	op_ss(t_stack *a, t_stack *b, int emit)
 {
 	stack_swap(a);
 	stack_swap(b);
-	emit_op("ss\n", emit);
+	return (emit_op("ss\n", emit));
 }
 
-void	op_pa(t_stack *a, t_stack *b, int emit)
+int	op_pa(t_stack *a, t_stack *b, int emit)
 {
 	stack_push(a, b);
-	emit_op("pa\n", emit);
+	return (emit_op("pa\n", emit));
 }
 
-void	op_pb(t_stack *a, t_stack *b, int emit)
+int	op_pb(t_stack *a, t_stack *b, int emit)
 {
 	stack_push(b, a);
-	emit_op("pb\n", emit);
+	return (emit_op("pb\n", emit));
 }
 
-void	op_ra(t_stack *a, int emit)
+int	op_ra(t_stack *a, int emit)
 {
 	stack_rotate(a);
-	emit_op("ra\n", emit);
+	return (emit_op("ra\n", emit));
 }
 
-void	op_rb(t_stack *b, int emit)
+int	op_rb(t_stack *b, int emit)
 {
 	stack_rotate(b);
-	emit_op("rb\n", emit);
+	return (emit_op("rb\n", emit));
 }
 
-void	op_rr(t_stack *a, t_stack *b, int emit)
+int	op_rr(t_stack *a, t_stack *b, int emit)
 {
 	stack_rotate(a);
 	stack_rotate(b);
-	emit_op("rr\n", emit);
+	return (emit_op("rr\n", emit));
 }
 
-void	op_rra(t_stack *a, int emit)
+int	op_rra(t_stack *a, int emit)
 {
 	stack_reverse_rotate(a);
-	emit_op("rra\n", emit);
+	return (emit_op("rra\n", emit));
 }
 
-void	op_rrb(t_stack *b, int emit)
+int	op_rrb(t_stack *b, int emit)
 {
 	stack_reverse_rotate(b);
-	emit_op("rrb\n", emit);
+	return (emit_op("rrb\n", emit));
 }
 
-void	op_rrr(t_stack *a, t_stack *b, int emit)
+int	op_rrr(t_stack *a, t_stack *b, int emit)
 {
 	stack_reverse_rotate(a);
 	stack_reverse_rotate(b);
-	emit_op("rrr\n", emit);
+	return (emit_op("rrr\n", emit));
 }
diff --git a/src/push_swap.c b/src/push_swap.c
index 8f04725..2ce08e7 100644
--- a/src/push_swap.c
+++ b/src/push_swap.c
@@ -4,20 +4,23 @@ int	main(int argc, char **argv)
 {
 	t_stack	a;
 	t_stack	b;
+	int		status;
 
+	if (!ps_ignore_sigpipe())
+		return (write_error(), ps_test_finish(1));
 	if (!parse_input(argc, argv, &a))
-	{
-		write_error();
-		return (ps_test_finish(1));
-	}
+		return (write_error(), ps_test_finish(1));
 	if (!stack_init(&b, a.capacity))
 	{
 		stack_free(&a);
-		write_error();
-		return (ps_test_finish(1));
+		return (write_error(), ps_test_finish(1));
 	}
-	sort_stack(&a, &b);
+	status = 0;
+	if (!sort_stack(&a, &b))
+		status = 1;
 	stack_free(&a);
 	stack_free(&b);
-	return (ps_test_finish(0));
+	if (status != 0)
+		write_error();
+	return (ps_test_finish(status));
 }
diff --git a/src/runtime.c b/src/runtime.c
index a4e88b9..c10062e 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -1,6 +1,7 @@
 #include "push_swap.h"
 
 #include <errno.h>
+#include <signal.h>
 
 #ifdef PS_FAULT_INJECTION
 
@@ -100,6 +101,35 @@ ssize_t	ps_read(int fd, void *buffer, size_t count)
 	return (read(fd, buffer, count));
 }
 
+static ssize_t	ps_write_once(int fd, const void *buffer, size_t count)
+{
+	return (write(fd, buffer, count));
+}
+
+int	ps_write_all(int fd, const void *buffer, size_t count)
+{
+	const unsigned char	*cursor;
+	ssize_t				written;
+
+	cursor = (const unsigned char *)buffer;
+	while (count > 0)
+	{
+		written = ps_write_once(fd, cursor, count);
+		if (written < 0 && errno == EINTR)
+			continue ;
+		if (written <= 0)
+			return (0);
+		cursor += (size_t)written;
+		count -= (size_t)written;
+	}
+	return (1);
+}
+
+int	ps_ignore_sigpipe(void)
+{
+	return (signal(SIGPIPE, SIG_IGN) != SIG_ERR);
+}
+
 #ifdef PS_FAULT_INJECTION
 static void	raw_report(const char *message)
 {
diff --git a/src/sort.c b/src/sort.c
index 06fb848..8516658 100644
--- a/src/sort.c
+++ b/src/sort.c
@@ -14,77 +14,90 @@ static int	find_rank_index(const t_stack *stack, int rank)
 	return (-1);
 }
 
-static void	move_index_to_top(t_stack *a, int index)
+static int	move_index_to_top(t_stack *a, int index)
 {
 	if (index <= a->size / 2)
 	{
 		while (index-- > 0)
-			op_ra(a, 1);
+		{
+			if (!op_ra(a, 1))
+				return (0);
+		}
 	}
 	else
 	{
 		index = a->size - index;
 		while (index-- > 0)
-			op_rra(a, 1);
+		{
+			if (!op_rra(a, 1))
+				return (0);
+		}
 	}
+	return (1);
 }
 
-static void	sort_two(t_stack *a)
+static int	sort_two(t_stack *a)
 {
 	if (a->ranks[0] > a->ranks[1])
-		op_sa(a, 1);
+		return (op_sa(a, 1));
+	return (1);
 }
 
-static void	sort_three(t_stack *a)
+static int	sort_three(t_stack *a)
 {
 	int	first;
 	int	second;
 	int	third;
 
 	if (stack_is_sorted(a))
-		return ;
+		return (1);
 	first = a->ranks[0];
 	second = a->ranks[1];
 	third = a->ranks[2];
 	if (first > second && second < third && first < third)
-		op_sa(a, 1);
+		return (op_sa(a, 1));
 	else if (first > second && second > third)
 	{
-		op_sa(a, 1);
-		op_rra(a, 1);
+		if (!op_sa(a, 1))
+			return (0);
+		return (op_rra(a, 1));
 	}
 	else if (first > second && second < third && first > third)
-		op_ra(a, 1);
+		return (op_ra(a, 1));
 	else if (first < second && second > third && first < third)
 	{
-		op_sa(a, 1);
-		op_ra(a, 1);
+		if (!op_sa(a, 1))
+			return (0);
+		return (op_ra(a, 1));
 	}
 	else if (first < second && second > third && first > third)
-		op_rra(a, 1);
+		return (op_rra(a, 1));
+	return (1);
 }
 
-static void	sort_tiny(t_stack *a, t_stack *b)
+static int	sort_tiny(t_stack *a, t_stack *b)
 {
 	int	target_rank;
 	int	index;
 
 	if (a->size == 2)
-	{
-		sort_two(a);
-		return ;
-	}
+		return (sort_two(a));
 	target_rank = 0;
 	while (a->size > 3)
 	{
 		index = find_rank_index(a, target_rank);
-		move_index_to_top(a, index);
-		op_pb(a, b, 1);
+		if (!move_index_to_top(a, index) || !op_pb(a, b, 1))
+			return (0);
 		target_rank++;
 	}
-	sort_three(a);
+	if (!sort_three(a))
+		return (0);
 	while (b->size > 0)
-		op_pa(a, b, 1);
+	{
+		if (!op_pa(a, b, 1))
+			return (0);
+	}
+	return (1);
 }
 
 static int	count_bits(int size)
@@ -99,7 +112,7 @@ static int	count_bits(int size)
 	return (bits);
 }
 
-static void	radix_sort(t_stack *a, t_stack *b)
+static int	radix_sort(t_stack *a, t_stack *b)
 {
 	int	bits;
 	int	bit;
@@ -115,23 +128,29 @@ static void	radix_sort(t_stack *a, t_stack *b)
 		while (i < round_size)
 		{
 			if (((a->ranks[0] >> bit) & 1) == 1)
-				op_ra(a, 1);
-			else
-				op_pb(a, b, 1);
+			{
+				if (!op_ra(a, 1))
+					return (0);
+			}
+			else if (!op_pb(a, b, 1))
+				return (0);
 			i++;
 		}
 		while (b->size > 0)
-			op_pa(a, b, 1);
+		{
+			if (!op_pa(a, b, 1))
+				return (0);
+		}
 		bit++;
 	}
+	return (1);
 }
 
-void	sort_stack(t_stack *a, t_stack *b)
+int	sort_stack(t_stack *a, t_stack *b)
 {
 	if (a->size < 2 || stack_is_sorted(a))
-		return ;
+		return (1);
 	if (a->size <= 5)
-		sort_tiny(a, b);
-	else
-		radix_sort(a, b);
+		return (sort_tiny(a, b));
+	return (radix_sort(a, b));
 }
diff --git a/src/utils.c b/src/utils.c
index a996ac1..056a661 100644
--- a/src/utils.c
+++ b/src/utils.c
@@ -20,13 +20,14 @@ int	ps_strcmp(const char *a, const char *b)
 	return ((unsigned char)a[i] - (unsigned char)b[i]);
 }
 
-void	ps_putstr_fd(int fd, const char *str)
+int	ps_putstr_fd(int fd, const char *str)
 {
-	if (str != NULL)
-		(void)write(fd, str, ps_strlen(str));
+	if (str == NULL)
+		return (1);
+	return (ps_write_all(fd, str, ps_strlen(str)));
 }
 
-void	write_error(void)
+int	write_error(void)
 {
-	ps_putstr_fd(2, "Error\n");
+	return (ps_putstr_fd(2, "Error\n"));
 }


## `test(io): 부분 출력과 영구 쓰기 실패를 검증`

diff --git a/src/runtime.c b/src/runtime.c
index c10062e..4e0c971 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -18,6 +18,7 @@ typedef union u_allocation_header
 
 static unsigned long	g_malloc_calls;
 static unsigned long	g_read_calls;
+static unsigned long	g_write_calls;
 static unsigned long	g_live_allocations;
 
 static unsigned long	read_index(const char *name)
@@ -103,6 +104,17 @@ ssize_t	ps_read(int fd, void *buffer, size_t count)
 
 static ssize_t	ps_write_once(int fd, const void *buffer, size_t count)
 {
+#ifdef PS_FAULT_INJECTION
+	g_write_calls++;
+	if (at_index("PS_EINTR_WRITE_AT", g_write_calls))
+		return (errno = EINTR, -1);
+	if (at_index("PS_FAIL_WRITE_AT", g_write_calls))
+		return (errno = EPIPE, -1);
+	if (at_index("PS_ZERO_WRITE_AT", g_write_calls))
+		return (0);
+	if (at_index("PS_SHORT_WRITE_AT", g_write_calls) && count > 1)
+		count = 1;
+#endif
 	return (write(fd, buffer, count));
 }
 
diff --git a/tests/fault_tests.py b/tests/fault_tests.py
index 1a104d4..164d87a 100644
--- a/tests/fault_tests.py
+++ b/tests/fault_tests.py
@@ -129,9 +129,94 @@ def test_read_failures_and_command_bounds():
     assert_equal(unterminated.stdout, b"OK\n", "unterminated command is applied")
 
 
+def test_write_failures():
+    baseline = run(PUSH_SWAP, ["3", "2", "1"])
+    assert_equal(baseline.returncode, 0, "push_swap baseline succeeds")
+    write_count = len(baseline.stdout.splitlines())
+    assert_true(write_count >= 2, "write fault fixture emits multiple commands")
+    for index in range(1, write_count + 1):
+        result = run(
+            PUSH_SWAP,
+            ["3", "2", "1"],
+            faults={"PS_FAIL_WRITE_AT": index},
+        )
+        assert_true(result.returncode != 0, f"push_swap write #{index} fails")
+        assert_equal(result.stderr, b"Error\n", "push_swap write error")
+
+    for fault in ("PS_EINTR_WRITE_AT", "PS_SHORT_WRITE_AT"):
+        result = run(PUSH_SWAP, ["3", "2", "1"], faults={fault: 1})
+        assert_equal(result.returncode, 0, f"push_swap handles {fault}")
+        assert_equal(result.stdout, baseline.stdout, f"{fault} preserves command stream")
+
+    zero = run(PUSH_SWAP, ["3", "2", "1"], faults={"PS_ZERO_WRITE_AT": 1})
+    assert_true(zero.returncode != 0, "zero-byte write is a permanent failure")
+    assert_equal(zero.stderr, b"Error\n", "zero-byte write reports Error")
+
+    partial = run(
+        PUSH_SWAP,
+        ["3", "2", "1"],
+        faults={"PS_SHORT_WRITE_AT": 1, "PS_FAIL_WRITE_AT": 2},
+    )
+    assert_equal(partial.returncode, 1, "failure after a partial write reaches main")
+    assert_equal(partial.stdout, baseline.stdout[:1], "partial prefix is not repeated")
+    assert_equal(partial.stderr, b"Error\n", "partial-success failure reports Error")
+
+    checker = run(
+        CHECKER,
+        ["2", "1"],
+        b"sa\n",
+        faults={"PS_FAIL_WRITE_AT": 1},
+    )
+    assert_true(checker.returncode != 0, "checker propagates result write failure")
+    assert_equal(checker.stderr, b"Error\n", "checker output failure reports Error")
+
+    error_write_cases = [
+        ("push_swap parse error", PUSH_SWAP, ["1", "1"], b""),
+        ("checker command error", CHECKER, ["1", "2"], b"wat\n"),
+    ]
+    for label, binary, args, input_bytes in error_write_cases:
+        result = run(
+            binary,
+            args,
+            input_bytes,
+            faults={"PS_FAIL_WRITE_AT": 1},
+        )
+        assert_equal(result.returncode, 1, f"{label} keeps failure status")
+        assert_equal(result.stdout, b"", f"{label} has no stdout")
+        assert_equal(result.stderr, b"", f"{label} tolerates Error write failure")
+
+    read_fd, write_fd = os.pipe()
+    os.close(read_fd)
+    environment = os.environ.copy()
+    environment["PS_REPORT_ALLOCATIONS"] = "1"
+    process = subprocess.Popen(
+        [str(PUSH_SWAP), "3", "2", "1"],
+        stdin=subprocess.DEVNULL,
+        stdout=write_fd,
+        stderr=subprocess.PIPE,
+        cwd=ROOT,
+        env=environment,
+    )
+    os.close(write_fd)
+    try:
+        _, stderr = process.communicate(timeout=3)
+    except subprocess.TimeoutExpired:
+        process.kill()
+        process.communicate()
+        fail("push_swap timed out after its stdout pipe closed")
+    assert_equal(process.returncode, 1, "closed stdout is reported by main")
+    assert_equal(
+        stderr.replace(ALLOCATION_REPORT, b""),
+        b"Error\n",
+        "closed stdout reports Error instead of dying from SIGPIPE",
+    )
+    assert_true(ALLOCATION_REPORT in stderr, "closed stdout path releases allocations")
+
+
 def main():
     test_nth_allocation_failures()
     test_read_failures_and_command_bounds()
+    test_write_failures()
     print("fault injection tests passed")
 
 


## `build(sanitize): C99 sanitizer 검증 경로를 추가`

diff --git a/Makefile b/Makefile
index 868a142..1379366 100644
--- a/Makefile
+++ b/Makefile
@@ -13,23 +13,33 @@ COMMON_SRCS := \
 	src/operations.c \
 	src/runtime.c \
 	src/utils.c
-COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 PUSH_SRCS := src/push_swap.c src/sort.c
-PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 CHECKER_SRCS := src/checker.c src/checker_reader.c
-CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
+
+COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 FAULT_DIR := $(OBJ_DIR)/fault
 FAULT_COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(FAULT_DIR)/%.o)
 FAULT_PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(FAULT_DIR)/%.o)
 FAULT_CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(FAULT_DIR)/%.o)
 FAULT_PUSH_SWAP := $(FAULT_DIR)/push_swap
 FAULT_CHECKER := $(FAULT_DIR)/checker
+SANITIZE_DIR := $(OBJ_DIR)/sanitize
+SANITIZE_CFLAGS := $(CFLAGS) -O1 -g -fno-omit-frame-pointer \
+	-fsanitize=address,undefined
+SANITIZE_COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_CHECKER_OBJS := $(CHECKER_SRCS:src/%.c=$(SANITIZE_DIR)/%.o)
+SANITIZE_PUSH_SWAP := $(SANITIZE_DIR)/push_swap
+SANITIZE_CHECKER := $(SANITIZE_DIR)/checker
+SANITIZE_OPERATION_TEST := $(SANITIZE_DIR)/operation_invariants
 
 .DELETE_ON_ERROR:
-.PHONY: all clean fclean re test
+.PHONY: all clean fclean re test sanitize
 
-all: $(COMMON_OBJS) $(PUSH_OBJS) $(CHECKER_OBJS) $(NAME) $(CHECKER)
+all: $(NAME) $(CHECKER)
 
 $(NAME): $(COMMON_OBJS) $(PUSH_OBJS)
 	$(CC) $(CFLAGS) $(COMMON_OBJS) $(PUSH_OBJS) -o $@
@@ -54,12 +64,29 @@ $(FAULT_CHECKER): $(FAULT_COMMON_OBJS) $(FAULT_CHECKER_OBJS)
 $(FAULT_DIR)/%.o: src/%.c include/push_swap.h | $(FAULT_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -DPS_FAULT_INJECTION -c $< -o $@
 
+$(SANITIZE_PUSH_SWAP): $(SANITIZE_COMMON_OBJS) $(SANITIZE_PUSH_OBJS)
+	$(CC) $(SANITIZE_CFLAGS) $^ -o $@
+
+$(SANITIZE_CHECKER): $(SANITIZE_COMMON_OBJS) $(SANITIZE_CHECKER_OBJS)
+	$(CC) $(SANITIZE_CFLAGS) $^ -o $@
+
+$(SANITIZE_OPERATION_TEST): tests/operation_invariants.c \
+		$(SANITIZE_DIR)/stack.o $(SANITIZE_DIR)/operations.o \
+		$(SANITIZE_DIR)/runtime.o $(SANITIZE_DIR)/utils.o
+	$(CC) $(SANITIZE_CFLAGS) $(CPPFLAGS) $^ -o $@
+
+$(SANITIZE_DIR)/%.o: src/%.c include/push_swap.h | $(SANITIZE_DIR)
+	$(CC) $(SANITIZE_CFLAGS) $(CPPFLAGS) -c $< -o $@
+
 $(OBJ_DIR):
 	mkdir -p $(OBJ_DIR)
 
 $(FAULT_DIR): | $(OBJ_DIR)
 	mkdir -p $(FAULT_DIR)
 
+$(SANITIZE_DIR): | $(OBJ_DIR)
+	mkdir -p $(SANITIZE_DIR)
+
 clean:
 	$(RMDIR) $(OBJ_DIR) tests/__pycache__ .pytest_cache
 
@@ -74,3 +101,13 @@ test: all $(OPERATION_TEST) $(FAULT_PUSH_SWAP) $(FAULT_CHECKER)
 	PS_PUSH_SWAP=$(FAULT_PUSH_SWAP) PS_CHECKER=$(FAULT_CHECKER) \
 		python3 tests/fault_tests.py
 	PS_PUSH_SWAP=$(FAULT_PUSH_SWAP) python3 tests/resource_tests.py
+
+sanitize: $(SANITIZE_PUSH_SWAP) $(SANITIZE_CHECKER) \
+		$(SANITIZE_OPERATION_TEST)
+	ASAN_OPTIONS=halt_on_error=1 \
+		UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		$(SANITIZE_OPERATION_TEST)
+	ASAN_OPTIONS=halt_on_error=1 \
+		UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		PS_PUSH_SWAP=$(SANITIZE_PUSH_SWAP) PS_CHECKER=$(SANITIZE_CHECKER) \
+		python3 tests/run_tests.py
