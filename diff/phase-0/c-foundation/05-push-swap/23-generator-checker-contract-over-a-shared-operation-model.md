# 공유 연산 모델 위의 생성기–Checker 계약

## `feat(model): 배열 기반 스택 상태를 구현`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..050653c
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,8 @@
+push_swap
+checker
+.build/
+*.o
+*.d
+tests/__pycache__/
+.pytest_cache/
+.DS_Store
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..659372b
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,25 @@
+CC := cc
+CFLAGS := -std=c99 -Wall -Wextra -Werror -Wpedantic
+CPPFLAGS := -Iinclude
+OBJ_DIR := .build
+
+MODEL_SRCS := src/stack.c
+MODEL_OBJS := $(MODEL_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+
+.PHONY: all clean fclean re
+
+all: $(MODEL_OBJS)
+
+$(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
+	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
+
+$(OBJ_DIR):
+	mkdir -p $(OBJ_DIR)
+
+clean:
+	rm -rf $(OBJ_DIR)
+
+fclean: clean
+	rm -f push_swap checker
+
+re: fclean all
diff --git a/include/push_swap.h b/include/push_swap.h
new file mode 100644
index 0000000..adfe1de
--- /dev/null
+++ b/include/push_swap.h
@@ -0,0 +1,21 @@
+#ifndef PUSH_SWAP_H
+# define PUSH_SWAP_H
+
+# include <stddef.h>
+# include <stdlib.h>
+
+typedef struct s_stack
+{
+	int	*values;
+	int	*ranks;
+	int	size;
+	int	capacity;
+}	t_stack;
+
+void	stack_init_empty(t_stack *stack);
+int		stack_init(t_stack *stack, int capacity);
+void	stack_free(t_stack *stack);
+int		stack_is_sorted(const t_stack *stack);
+int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
+
+#endif
diff --git a/src/stack.c b/src/stack.c
new file mode 100644
index 0000000..6be2997
--- /dev/null
+++ b/src/stack.c
@@ -0,0 +1,51 @@
+#include "push_swap.h"
+
+void	stack_init_empty(t_stack *stack)
+{
+	stack->values = NULL;
+	stack->ranks = NULL;
+	stack->size = 0;
+	stack->capacity = 0;
+}
+
+int	stack_init(t_stack *stack, int capacity)
+{
+	stack_init_empty(stack);
+	if (capacity <= 0)
+		return (1);
+	stack->values = (int *)malloc(sizeof(int) * (size_t)capacity);
+	stack->ranks = (int *)malloc(sizeof(int) * (size_t)capacity);
+	if (stack->values == NULL || stack->ranks == NULL)
+	{
+		stack_free(stack);
+		return (0);
+	}
+	stack->capacity = capacity;
+	return (1);
+}
+
+void	stack_free(t_stack *stack)
+{
+	free(stack->values);
+	free(stack->ranks);
+	stack_init_empty(stack);
+}
+
+int	stack_is_sorted(const t_stack *stack)
+{
+	int	i;
+
+	i = 1;
+	while (i < stack->size)
+	{
+		if (stack->ranks[i - 1] > stack->ranks[i])
+			return (0);
+		i++;
+	}
+	return (1);
+}
+
+int	stack_is_complete_sorted(const t_stack *a, const t_stack *b)
+{
+	return (b->size == 0 && stack_is_sorted(a));
+}


## `feat(operation): 스택 교환 연산을 구현`

diff --git a/Makefile b/Makefile
index 0116cd0..6b3f5f2 100644
--- a/Makefile
+++ b/Makefile
@@ -5,6 +5,7 @@ OBJ_DIR := .build
 
 COMMON_SRCS := \
 	src/stack.c \
+	src/operations.c \
 	src/utils.c
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 
diff --git a/include/push_swap.h b/include/push_swap.h
index 0f2a48c..c30fa9d 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -19,6 +19,12 @@ void	stack_free(t_stack *stack);
 int		stack_is_sorted(const t_stack *stack);
 int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
 
+void	stack_swap(t_stack *stack);
+
+void	op_sa(t_stack *a, int emit);
+void	op_sb(t_stack *b, int emit);
+void	op_ss(t_stack *a, t_stack *b, int emit);
+
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
 void	ps_putstr_fd(int fd, const char *str);
diff --git a/src/operations.c b/src/operations.c
new file mode 100644
index 0000000..0f19ed4
--- /dev/null
+++ b/src/operations.c
@@ -0,0 +1,43 @@
+#include "push_swap.h"
+
+#include <string.h>
+
+static void	emit_op(const char *name, int emit)
+{
+	if (emit)
+		ps_putstr_fd(1, name);
+}
+
+void	stack_swap(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[0];
+	rank = stack->ranks[0];
+	stack->values[0] = stack->values[1];
+	stack->ranks[0] = stack->ranks[1];
+	stack->values[1] = value;
+	stack->ranks[1] = rank;
+}
+
+void	op_sa(t_stack *a, int emit)
+{
+	stack_swap(a);
+	emit_op("sa\n", emit);
+}
+
+void	op_sb(t_stack *b, int emit)
+{
+	stack_swap(b);
+	emit_op("sb\n", emit);
+}
+
+void	op_ss(t_stack *a, t_stack *b, int emit)
+{
+	stack_swap(a);
+	stack_swap(b);
+	emit_op("ss\n", emit);
+}


## `feat(push_swap): 정렬 명령 생성 흐름을 연결`

diff --git a/Makefile b/Makefile
index 8550f9e..188e2d8 100644
--- a/Makefile
+++ b/Makefile
@@ -1,3 +1,4 @@
+NAME := push_swap
 CC := cc
 CFLAGS := -std=c99 -Wall -Wextra -Werror -Wpedantic
 CPPFLAGS := -Iinclude
@@ -9,13 +10,16 @@ COMMON_SRCS := \
 	src/operations.c \
 	src/utils.c
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
-PUSH_SRCS := src/sort.c
+PUSH_SRCS := src/push_swap.c src/sort.c
 PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
 .PHONY: all clean fclean re test
 
-all: $(COMMON_OBJS) $(PUSH_OBJS)
+all: $(COMMON_OBJS) $(PUSH_OBJS) $(NAME)
+
+$(NAME): $(COMMON_OBJS) $(PUSH_OBJS)
+	$(CC) $(CFLAGS) $(COMMON_OBJS) $(PUSH_OBJS) -o $@
 
 $(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
diff --git a/src/push_swap.c b/src/push_swap.c
new file mode 100644
index 0000000..c1ec597
--- /dev/null
+++ b/src/push_swap.c
@@ -0,0 +1,23 @@
+#include "push_swap.h"
+
+int	main(int argc, char **argv)
+{
+	t_stack	a;
+	t_stack	b;
+
+	if (!parse_input(argc, argv, &a))
+	{
+		write_error();
+		return (1);
+	}
+	if (!stack_init(&b, a.capacity))
+	{
+		stack_free(&a);
+		write_error();
+		return (1);
+	}
+	sort_stack(&a, &b);
+	stack_free(&a);
+	stack_free(&b);
+	return (0);
+}


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
 
 


## `test(sort): 생성 명령의 정렬 결과를 독립 검증`

diff --git a/tests/run_tests.py b/tests/run_tests.py
index 5b3549b..e638f86 100644
--- a/tests/run_tests.py
+++ b/tests/run_tests.py
@@ -1,5 +1,6 @@
 #!/usr/bin/env python3
 from pathlib import Path
+import itertools
 import subprocess
 import sys
 import tempfile
@@ -8,6 +9,7 @@ import tempfile
 ROOT = Path(__file__).resolve().parents[1]
 PUSH_SWAP = ROOT / "push_swap"
 CHECKER = ROOT / "checker"
+VALID_MOVES = {"sa", "sb", "ss", "pa", "pb", "ra", "rb", "rr", "rra", "rrb", "rrr"}
 CHILD_TIMEOUT_SECONDS = 5
 
 
@@ -164,11 +166,105 @@ def test_checker_operations():
     assert_equal(invalid.stderr, "Error\n", "invalid checker command reports Error")
 
 
+def split_moves(output):
+    if output == "":
+        return []
+    moves = output.splitlines()
+    for move in moves:
+        assert_ok(move in VALID_MOVES, f"unknown move emitted: {move!r}")
+    return moves
+
+
+def apply_moves(values, moves):
+    a = list(values)
+    b = []
+
+    def swap(stack):
+        if len(stack) > 1:
+            stack[0], stack[1] = stack[1], stack[0]
+
+    def push(dst, src):
+        if src:
+            dst.insert(0, src.pop(0))
+
+    def rotate(stack):
+        if len(stack) > 1:
+            stack.append(stack.pop(0))
+
+    def reverse_rotate(stack):
+        if len(stack) > 1:
+            stack.insert(0, stack.pop())
+
+    for move in moves:
+        if move == "sa":
+            swap(a)
+        elif move == "sb":
+            swap(b)
+        elif move == "ss":
+            swap(a)
+            swap(b)
+        elif move == "pa":
+            push(a, b)
+        elif move == "pb":
+            push(b, a)
+        elif move == "ra":
+            rotate(a)
+        elif move == "rb":
+            rotate(b)
+        elif move == "rr":
+            rotate(a)
+            rotate(b)
+        elif move == "rra":
+            reverse_rotate(a)
+        elif move == "rrb":
+            reverse_rotate(b)
+        elif move == "rrr":
+            reverse_rotate(a)
+            reverse_rotate(b)
+    return a, b
+
+
+def assert_sorted_by_program(values):
+    args = [str(value) for value in values]
+    result = run([PUSH_SWAP] + args)
+    assert_equal(result.returncode, 0, f"push_swap sorts {values!r}")
+    assert_equal(result.stderr, "", f"push_swap sort stderr for {values!r}")
+    moves = split_moves(result.stdout)
+    a, b = apply_moves(values, moves)
+    assert_equal(a, sorted(values), f"move stream sorts stack {values!r}")
+    assert_equal(b, [], f"move stream leaves stack b empty for {values!r}")
+    checked = run([CHECKER] + args, result.stdout)
+    assert_equal(checked.returncode, 0, f"checker validates stream for {values!r}")
+    assert_equal(checked.stdout, "OK\n", f"checker OK for {values!r}")
+    return moves
+
+
+def test_sort_programs():
+    sorted_case = run([PUSH_SWAP, "1", "2", "3", "4", "5"])
+    assert_equal(sorted_case.returncode, 0, "already sorted input exits cleanly")
+    assert_equal(sorted_case.stdout, "", "already sorted input emits no moves")
+    assert_equal(sorted_case.stderr, "", "already sorted input has no stderr")
+
+    fixed_cases = [
+        [3, 2, 1],
+        [5, -1, 4, 0, 2],
+        [42, -2147483648, 0, 2147483647, -7, 9],
+        [8, 3, 7, 1, 9, 0, 2, 6, 5, 4],
+    ]
+    for values in fixed_cases:
+        assert_sorted_by_program(values)
+
+    for size in range(2, 6):
+        for values in itertools.permutations(range(size)):
+            assert_sorted_by_program(list(values))
+
+
 def main():
     test_parser_inputs()
     test_parser_boundaries()
     test_checker_without_values_does_not_read_stdin()
     test_checker_operations()
+    test_sort_programs()
     print("tests passed")
 
 
