# 크기별 정렬과 LSD 기수 전략

## `feat(sort): 세 개 이하의 스택을 정렬`

diff --git a/Makefile b/Makefile
index cc01ea6..8550f9e 100644
--- a/Makefile
+++ b/Makefile
@@ -9,11 +9,13 @@ COMMON_SRCS := \
 	src/operations.c \
 	src/utils.c
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+PUSH_SRCS := src/sort.c
+PUSH_OBJS := $(PUSH_SRCS:src/%.c=$(OBJ_DIR)/%.o)
 OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
 .PHONY: all clean fclean re test
 
-all: $(COMMON_OBJS)
+all: $(COMMON_OBJS) $(PUSH_OBJS)
 
 $(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
diff --git a/include/push_swap.h b/include/push_swap.h
index 41a1900..6b94c4e 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -38,6 +38,7 @@ void	op_rrb(t_stack *b, int emit);
 void	op_rrr(t_stack *a, t_stack *b, int emit);
 
 int		parse_input(int argc, char **argv, t_stack *a);
+void	sort_stack(t_stack *a, t_stack *b);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/sort.c b/src/sort.c
new file mode 100644
index 0000000..db7c519
--- /dev/null
+++ b/src/sort.c
@@ -0,0 +1,47 @@
+#include "push_swap.h"
+
+static void	sort_two(t_stack *a)
+{
+	if (a->ranks[0] > a->ranks[1])
+		op_sa(a, 1);
+}
+
+static void	sort_three(t_stack *a)
+{
+	int	first;
+	int	second;
+	int	third;
+
+	if (stack_is_sorted(a))
+		return ;
+	first = a->ranks[0];
+	second = a->ranks[1];
+	third = a->ranks[2];
+	if (first > second && second < third && first < third)
+		op_sa(a, 1);
+	else if (first > second && second > third)
+	{
+		op_sa(a, 1);
+		op_rra(a, 1);
+	}
+	else if (first > second && second < third && first > third)
+		op_ra(a, 1);
+	else if (first < second && second > third && first < third)
+	{
+		op_sa(a, 1);
+		op_ra(a, 1);
+	}
+	else if (first < second && second > third && first > third)
+		op_rra(a, 1);
+}
+
+void	sort_stack(t_stack *a, t_stack *b)
+{
+	(void)b;
+	if (a->size < 2 || stack_is_sorted(a))
+		return ;
+	if (a->size == 2)
+		sort_two(a);
+	else if (a->size == 3)
+		sort_three(a);
+}


## `feat(sort): 네다섯 개의 스택을 정렬`

diff --git a/src/sort.c b/src/sort.c
index db7c519..c130518 100644
--- a/src/sort.c
+++ b/src/sort.c
@@ -1,5 +1,34 @@
 #include "push_swap.h"
 
+static int	find_rank_index(const t_stack *stack, int rank)
+{
+	int	i;
+
+	i = 0;
+	while (i < stack->size)
+	{
+		if (stack->ranks[i] == rank)
+			return (i);
+		i++;
+	}
+	return (-1);
+}
+
+static void	move_index_to_top(t_stack *a, int index)
+{
+	if (index <= a->size / 2)
+	{
+		while (index-- > 0)
+			op_ra(a, 1);
+	}
+	else
+	{
+		index = a->size - index;
+		while (index-- > 0)
+			op_rra(a, 1);
+	}
+}
+
 static void	sort_two(t_stack *a)
 {
 	if (a->ranks[0] > a->ranks[1])
@@ -35,13 +64,33 @@ static void	sort_three(t_stack *a)
 		op_rra(a, 1);
 }
 
+static void	sort_tiny(t_stack *a, t_stack *b)
+{
+	int	target_rank;
+	int	index;
+
+	if (a->size == 2)
+	{
+		sort_two(a);
+		return ;
+	}
+	target_rank = 0;
+	while (a->size > 3)
+	{
+		index = find_rank_index(a, target_rank);
+		move_index_to_top(a, index);
+		op_pb(a, b, 1);
+		target_rank++;
+	}
+	sort_three(a);
+	while (b->size > 0)
+		op_pa(a, b, 1);
+}
+
 void	sort_stack(t_stack *a, t_stack *b)
 {
-	(void)b;
 	if (a->size < 2 || stack_is_sorted(a))
 		return ;
-	if (a->size == 2)
-		sort_two(a);
-	else if (a->size == 3)
-		sort_three(a);
+	if (a->size <= 5)
+		sort_tiny(a, b);
 }


## `feat(sort): 큰 입력을 기수 정렬로 처리`

diff --git a/src/sort.c b/src/sort.c
index c130518..06fb848 100644
--- a/src/sort.c
+++ b/src/sort.c
@@ -87,10 +87,51 @@ static void	sort_tiny(t_stack *a, t_stack *b)
 		op_pa(a, b, 1);
 }
 
+static int	count_bits(int size)
+{
+	int	bits;
+	int	max_rank;
+
+	bits = 0;
+	max_rank = size - 1;
+	while ((max_rank >> bits) != 0)
+		bits++;
+	return (bits);
+}
+
+static void	radix_sort(t_stack *a, t_stack *b)
+{
+	int	bits;
+	int	bit;
+	int	i;
+	int	round_size;
+
+	bits = count_bits(a->size);
+	bit = 0;
+	while (bit < bits)
+	{
+		round_size = a->size;
+		i = 0;
+		while (i < round_size)
+		{
+			if (((a->ranks[0] >> bit) & 1) == 1)
+				op_ra(a, 1);
+			else
+				op_pb(a, b, 1);
+			i++;
+		}
+		while (b->size > 0)
+			op_pa(a, b, 1);
+		bit++;
+	}
+}
+
 void	sort_stack(t_stack *a, t_stack *b)
 {
 	if (a->size < 2 || stack_is_sorted(a))
 		return ;
 	if (a->size <= 5)
 		sort_tiny(a, b);
+	else
+		radix_sort(a, b);
 }


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
 
 
