# 입력 정규화와 순위 압축

## `feat(parse): 개별 인자의 부호 있는 정수를 파싱`

diff --git a/Makefile b/Makefile
index e8d141f..cc01ea6 100644
--- a/Makefile
+++ b/Makefile
@@ -4,6 +4,7 @@ CPPFLAGS := -Iinclude
 OBJ_DIR := .build
 
 COMMON_SRCS := \
+	src/parser.c \
 	src/stack.c \
 	src/operations.c \
 	src/utils.c
diff --git a/include/push_swap.h b/include/push_swap.h
index 76f914c..41a1900 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -1,6 +1,7 @@
 #ifndef PUSH_SWAP_H
 # define PUSH_SWAP_H
 
+# include <limits.h>
 # include <stddef.h>
 # include <stdlib.h>
 # include <unistd.h>
@@ -36,6 +37,8 @@ void	op_rra(t_stack *a, int emit);
 void	op_rrb(t_stack *b, int emit);
 void	op_rrr(t_stack *a, t_stack *b, int emit);
 
+int		parse_input(int argc, char **argv, t_stack *a);
+
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
 void	ps_putstr_fd(int fd, const char *str);
diff --git a/src/parser.c b/src/parser.c
new file mode 100644
index 0000000..336211d
--- /dev/null
+++ b/src/parser.c
@@ -0,0 +1,64 @@
+#include "push_swap.h"
+
+static int	parse_token(const char *arg, int start, int end, int *out)
+{
+	long long	value;
+	long long	limit;
+	int			sign;
+	int			i;
+
+	value = 0;
+	sign = 1;
+	i = start;
+	if (arg[i] == '+' || arg[i] == '-')
+	{
+		if (arg[i] == '-')
+			sign = -1;
+		i++;
+	}
+	if (i == end)
+		return (0);
+	limit = INT_MAX;
+	if (sign < 0)
+		limit = (long long)INT_MAX + 1;
+	while (i < end)
+	{
+		if (arg[i] < '0' || arg[i] > '9')
+			return (0);
+		value = value * 10 + (arg[i] - '0');
+		if (value > limit)
+			return (0);
+		i++;
+	}
+	*out = (int)(value * sign);
+	return (1);
+}
+
+int	parse_input(int argc, char **argv, t_stack *a)
+{
+	int	count;
+	int	index;
+	int	value;
+
+	stack_init_empty(a);
+	if (argc == 1)
+		return (1);
+	count = argc - 1;
+	if (!stack_init(a, count))
+		return (0);
+	index = 0;
+	while (index < count)
+	{
+		if (!parse_token(argv[index + 1], 0,
+				(int)ps_strlen(argv[index + 1]), &value))
+		{
+			stack_free(a);
+			return (0);
+		}
+		a->values[index] = value;
+		a->ranks[index] = value;
+		index++;
+	}
+	a->size = count;
+	return (1);
+}


## `feat(parse): 공백으로 결합된 인자 토큰을 처리`

diff --git a/src/parser.c b/src/parser.c
index 336211d..4bd86de 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -1,5 +1,46 @@
 #include "push_swap.h"
 
+static int	is_space(char c)
+{
+	return (c == ' ' || c == '\t' || c == '\n'
+		|| c == '\r' || c == '\v' || c == '\f');
+}
+
+static int	count_tokens_in_arg(const char *arg)
+{
+	int	count;
+	int	i;
+
+	count = 0;
+	i = 0;
+	while (arg[i] != '\0')
+	{
+		while (is_space(arg[i]))
+			i++;
+		if (arg[i] == '\0')
+			break ;
+		count++;
+		while (arg[i] != '\0' && !is_space(arg[i]))
+			i++;
+	}
+	return (count);
+}
+
+static int	count_all_tokens(int argc, char **argv)
+{
+	int	count;
+	int	i;
+
+	count = 0;
+	i = 1;
+	while (i < argc)
+	{
+		count += count_tokens_in_arg(argv[i]);
+		i++;
+	}
+	return (count);
+}
+
 static int	parse_token(const char *arg, int start, int end, int *out)
 {
 	long long	value;
@@ -34,30 +75,55 @@ static int	parse_token(const char *arg, int start, int end, int *out)
 	return (1);
 }
 
+static int	fill_values(int argc, char **argv, t_stack *a)
+{
+	int	i;
+	int	pos;
+	int	start;
+	int	end;
+	int	value;
+
+	i = 1;
+	pos = 0;
+	while (i < argc)
+	{
+		end = 0;
+		while (argv[i][end] != '\0')
+		{
+			while (is_space(argv[i][end]))
+				end++;
+			if (argv[i][end] == '\0')
+				break ;
+			start = end;
+			while (argv[i][end] != '\0' && !is_space(argv[i][end]))
+				end++;
+			if (!parse_token(argv[i], start, end, &value))
+				return (0);
+			a->values[pos] = value;
+			a->ranks[pos] = value;
+			pos++;
+		}
+		i++;
+	}
+	return (1);
+}
+
 int	parse_input(int argc, char **argv, t_stack *a)
 {
 	int	count;
-	int	index;
-	int	value;
 
 	stack_init_empty(a);
 	if (argc == 1)
 		return (1);
-	count = argc - 1;
+	count = count_all_tokens(argc, argv);
+	if (count == 0)
+		return (0);
 	if (!stack_init(a, count))
 		return (0);
-	index = 0;
-	while (index < count)
+	if (!fill_values(argc, argv, a))
 	{
-		if (!parse_token(argv[index + 1], 0,
-				(int)ps_strlen(argv[index + 1]), &value))
-		{
-			stack_free(a);
-			return (0);
-		}
-		a->values[index] = value;
-		a->ranks[index] = value;
-		index++;
+		stack_free(a);
+		return (0);
 	}
 	a->size = count;
 	return (1);


## `feat(parse): 중복 입력을 거절하고 상대 순위를 계산`

diff --git a/src/parser.c b/src/parser.c
index 4bd86de..1efed98 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -108,6 +108,70 @@ static int	fill_values(int argc, char **argv, t_stack *a)
 	return (1);
 }
 
+static int	compare_ints(const void *left, const void *right)
+{
+	int	a;
+	int	b;
+
+	a = *(const int *)left;
+	b = *(const int *)right;
+	return ((a > b) - (a < b));
+}
+
+static int	find_rank(const int *sorted, int size, int value)
+{
+	int	low;
+	int	high;
+	int	mid;
+
+	low = 0;
+	high = size;
+	while (low < high)
+	{
+		mid = low + (high - low) / 2;
+		if (sorted[mid] < value)
+			low = mid + 1;
+		else
+			high = mid;
+	}
+	return (low);
+}
+
+static int	assign_ranks(t_stack *a)
+{
+	int	*sorted;
+	int	i;
+
+	sorted = (int *)malloc(sizeof(int) * (size_t)a->size);
+	if (sorted == NULL)
+		return (0);
+	i = 0;
+	while (i < a->size)
+	{
+		sorted[i] = a->values[i];
+		i++;
+	}
+	qsort(sorted, (size_t)a->size, sizeof(int), compare_ints);
+	i = 1;
+	while (i < a->size)
+	{
+		if (sorted[i - 1] == sorted[i])
+		{
+			free(sorted);
+			return (0);
+		}
+		i++;
+	}
+	i = 0;
+	while (i < a->size)
+	{
+		a->ranks[i] = find_rank(sorted, a->size, a->values[i]);
+		i++;
+	}
+	free(sorted);
+	return (1);
+}
+
 int	parse_input(int argc, char **argv, t_stack *a)
 {
 	int	count;
@@ -126,5 +190,10 @@ int	parse_input(int argc, char **argv, t_stack *a)
 		return (0);
 	}
 	a->size = count;
+	if (!assign_ranks(a))
+	{
+		stack_free(a);
+		return (0);
+	}
 	return (1);
 }


## `test(parser): 정상 입력과 오류 입력을 검증`

diff --git a/Makefile b/Makefile
index f90afbd..c658e64 100644
--- a/Makefile
+++ b/Makefile
@@ -46,3 +46,4 @@ re: fclean all
 
 test: all $(OPERATION_TEST)
 	$(OPERATION_TEST)
+	python3 tests/run_tests.py
diff --git a/tests/run_tests.py b/tests/run_tests.py
new file mode 100644
index 0000000..e9617f7
--- /dev/null
+++ b/tests/run_tests.py
@@ -0,0 +1,71 @@
+#!/usr/bin/env python3
+from pathlib import Path
+import subprocess
+import sys
+
+
+ROOT = Path(__file__).resolve().parents[1]
+PUSH_SWAP = ROOT / "push_swap"
+CHECKER = ROOT / "checker"
+
+
+def run(args, input_text=None):
+    return subprocess.run(
+        [str(arg) for arg in args],
+        input=input_text,
+        text=True,
+        capture_output=True,
+        cwd=ROOT,
+    )
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
+def assert_ok(condition, message):
+    if not condition:
+        fail(message)
+
+
+def test_parser_inputs():
+    no_args = run([PUSH_SWAP])
+    assert_equal(no_args.returncode, 0, "push_swap without args exits cleanly")
+    assert_equal(no_args.stdout, "", "push_swap without args has no stdout")
+    assert_equal(no_args.stderr, "", "push_swap without args has no stderr")
+
+    valid = run([PUSH_SWAP, "3 2", "1"])
+    assert_equal(valid.returncode, 0, "quoted and split argv are accepted")
+    checked = run([CHECKER, "3 2", "1"], valid.stdout)
+    assert_equal(checked.returncode, 0, "checker accepts generated moves")
+    assert_equal(checked.stdout, "OK\n", "generated moves sort quoted input")
+
+    invalid_cases = [
+        ["1", "2", "2"],
+        ["2147483648"],
+        ["-2147483649"],
+        ["12a"],
+        ["+"],
+        [""],
+        ["1", "2 1"],
+    ]
+    for case in invalid_cases:
+        result = run([PUSH_SWAP] + case)
+        assert_ok(result.returncode != 0, f"invalid input {case!r} fails")
+        assert_equal(result.stdout, "", f"invalid input {case!r} has no stdout")
+        assert_equal(result.stderr, "Error\n", f"invalid input {case!r} reports Error")
+
+
+def main():
+    test_parser_inputs()
+    print("tests passed")
+
+
+if __name__ == "__main__":
+    main()


## `test(cli): 입력 경계와 무인자 실행을 검증`

diff --git a/tests/run_tests.py b/tests/run_tests.py
index e9617f7..3087061 100644
--- a/tests/run_tests.py
+++ b/tests/run_tests.py
@@ -2,21 +2,28 @@
 from pathlib import Path
 import subprocess
 import sys
+import tempfile
 
 
 ROOT = Path(__file__).resolve().parents[1]
 PUSH_SWAP = ROOT / "push_swap"
 CHECKER = ROOT / "checker"
+CHILD_TIMEOUT_SECONDS = 5
 
 
 def run(args, input_text=None):
-    return subprocess.run(
-        [str(arg) for arg in args],
-        input=input_text,
-        text=True,
-        capture_output=True,
-        cwd=ROOT,
-    )
+    try:
+        return subprocess.run(
+            [str(arg) for arg in args],
+            input=input_text,
+            text=True,
+            capture_output=True,
+            cwd=ROOT,
+            timeout=CHILD_TIMEOUT_SECONDS,
+            check=False,
+        )
+    except subprocess.TimeoutExpired:
+        fail(f"child process exceeded {CHILD_TIMEOUT_SECONDS} seconds: {args!r}")
 
 
 def fail(message):
@@ -62,8 +69,71 @@ def test_parser_inputs():
         assert_equal(result.stderr, "Error\n", f"invalid input {case!r} reports Error")
 
 
+def assert_parser_accepts(args, label):
+    result = run([PUSH_SWAP] + args)
+    assert_equal(result.returncode, 0, f"{label} is accepted")
+    assert_equal(result.stderr, "", f"{label} has no stderr")
+    checked = run([CHECKER] + args, result.stdout)
+    assert_equal(checked.returncode, 0, f"checker accepts {label}")
+    assert_equal(checked.stdout, "OK\n", f"{label} sorts correctly")
+    assert_equal(checked.stderr, "", f"checker {label} has no stderr")
+
+
+def test_parser_boundaries():
+    valid_cases = [
+        (["+7"], "explicit plus sign"),
+        (["-0"], "negative zero"),
+        (["0003", "0002", "0001"], "leading zeroes"),
+        (["3", "", "2 1"], "empty argv mixed with values"),
+        (["3 2\t1\n0\r-1\v-2\f-3"], "all C whitespace separators"),
+        (["-2147483648", "0", "2147483647"], "exact int bounds"),
+    ]
+    for args, label in valid_cases:
+        assert_parser_accepts(args, label)
+
+    invalid_cases = [
+        ([" \t\n\r\v\f"], "whitespace-only argv"),
+        (["9" * 4096], "overlong integer"),
+        (["\u0661"], "non-ASCII digit"),
+        (["++1"], "repeated plus sign"),
+        (["--1"], "repeated minus sign"),
+        (["+-1"], "mixed signs"),
+        (["-0", "+0"], "signed duplicate zero"),
+    ]
+    for args, label in invalid_cases:
+        result = run([PUSH_SWAP] + args)
+        assert_equal(result.returncode, 1, f"{label} fails")
+        assert_equal(result.stdout, "", f"{label} has no stdout")
+        assert_equal(result.stderr, "Error\n", f"{label} reports Error")
+
+
+def test_checker_without_values_does_not_read_stdin():
+    with tempfile.TemporaryFile() as input_file:
+        input_file.write(b"sa\n")
+        input_file.flush()
+        input_file.seek(0)
+        try:
+            result = subprocess.run(
+                [str(CHECKER)],
+                stdin=input_file,
+                text=True,
+                capture_output=True,
+                cwd=ROOT,
+                timeout=CHILD_TIMEOUT_SECONDS,
+                check=False,
+            )
+        except subprocess.TimeoutExpired:
+            fail("checker without values timed out")
+        assert_equal(result.returncode, 0, "checker without values exits cleanly")
+        assert_equal(result.stdout, "", "checker without values has no stdout")
+        assert_equal(result.stderr, "", "checker without values has no stderr")
+        assert_equal(input_file.tell(), 0, "checker without values leaves stdin unread")
+
+
 def main():
     test_parser_inputs()
+    test_parser_boundaries()
+    test_checker_without_values_does_not_read_stdin()
     print("tests passed")
 
 


## `fix(parse): 토큰 수와 배열 크기 계산을 방어`

diff --git a/src/parser.c b/src/parser.c
index c3d2412..ffa0934 100644
--- a/src/parser.c
+++ b/src/parser.c
@@ -6,10 +6,10 @@ static int	is_space(char c)
 		|| c == '\r' || c == '\v' || c == '\f');
 }
 
-static int	count_tokens_in_arg(const char *arg)
+static size_t	count_tokens_in_arg(const char *arg)
 {
-	int	count;
-	int	i;
+	size_t	count;
+	size_t	i;
 
 	count = 0;
 	i = 0;
@@ -19,6 +19,8 @@ static int	count_tokens_in_arg(const char *arg)
 			i++;
 		if (arg[i] == '\0')
 			break ;
+		if (count == (size_t)INT_MAX)
+			return ((size_t)INT_MAX + 1);
 		count++;
 		while (arg[i] != '\0' && !is_space(arg[i]))
 			i++;
@@ -26,27 +28,32 @@ static int	count_tokens_in_arg(const char *arg)
 	return (count);
 }
 
-static int	count_all_tokens(int argc, char **argv)
+static int	count_all_tokens(int argc, char **argv, int *result)
 {
-	int	count;
+	size_t	count;
 	int	i;
+	size_t	argument_count;
 
 	count = 0;
 	i = 1;
 	while (i < argc)
 	{
-		count += count_tokens_in_arg(argv[i]);
+		argument_count = count_tokens_in_arg(argv[i]);
+		if (argument_count > (size_t)INT_MAX - count)
+			return (0);
+		count += argument_count;
 		i++;
 	}
-	return (count);
+	*result = (int)count;
+	return (1);
 }
 
-static int	parse_token(const char *arg, int start, int end, int *out)
+static int	parse_token(const char *arg, size_t start, size_t end, int *out)
 {
 	long long	value;
 	long long	limit;
 	int			sign;
-	int			i;
+	size_t		i;
 
 	value = 0;
 	sign = 1;
@@ -79,8 +86,8 @@ static int	fill_values(int argc, char **argv, t_stack *a)
 {
 	int	i;
 	int	pos;
-	int	start;
-	int	end;
+	size_t	start;
+	size_t	end;
 	int	value;
 
 	i = 1;
@@ -179,7 +186,8 @@ int	parse_input(int argc, char **argv, t_stack *a)
 	stack_init_empty(a);
 	if (argc == 1)
 		return (1);
-	count = count_all_tokens(argc, argv);
+	if (!count_all_tokens(argc, argv, &count))
+		return (0);
 	if (count == 0)
 		return (0);
 	if (!stack_init(a, count))
diff --git a/src/stack.c b/src/stack.c
index f76604b..b4e1fb2 100644
--- a/src/stack.c
+++ b/src/stack.c
@@ -13,6 +13,8 @@ int	stack_init(t_stack *stack, int capacity)
 	stack_init_empty(stack);
 	if (capacity <= 0)
 		return (1);
+	if ((size_t)capacity > (size_t)-1 / sizeof(int))
+		return (0);
 	stack->values = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
 	stack->ranks = (int *)ps_malloc(sizeof(int) * (size_t)capacity);
 	if (stack->values == NULL || stack->ranks == NULL)
