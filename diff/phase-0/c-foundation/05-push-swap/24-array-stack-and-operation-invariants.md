# 배열 스택과 연산 불변식

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


## `feat(operation): 스택 간 이동 연산을 구현`

diff --git a/include/push_swap.h b/include/push_swap.h
index c30fa9d..c038438 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -20,10 +20,13 @@ int		stack_is_sorted(const t_stack *stack);
 int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
 
 void	stack_swap(t_stack *stack);
+void	stack_push(t_stack *dst, t_stack *src);
 
 void	op_sa(t_stack *a, int emit);
 void	op_sb(t_stack *b, int emit);
 void	op_ss(t_stack *a, t_stack *b, int emit);
+void	op_pa(t_stack *a, t_stack *b, int emit);
+void	op_pb(t_stack *a, t_stack *b, int emit);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/operations.c b/src/operations.c
index 0f19ed4..856912e 100644
--- a/src/operations.c
+++ b/src/operations.c
@@ -23,6 +23,30 @@ void	stack_swap(t_stack *stack)
 	stack->ranks[1] = rank;
 }
 
+void	stack_push(t_stack *dst, t_stack *src)
+{
+	if (src->size == 0)
+		return ;
+	if (dst->size > 0)
+	{
+		memmove(dst->values + 1, dst->values,
+			sizeof(int) * (size_t)dst->size);
+		memmove(dst->ranks + 1, dst->ranks,
+			sizeof(int) * (size_t)dst->size);
+	}
+	dst->values[0] = src->values[0];
+	dst->ranks[0] = src->ranks[0];
+	dst->size++;
+	src->size--;
+	if (src->size > 0)
+	{
+		memmove(src->values, src->values + 1,
+			sizeof(int) * (size_t)src->size);
+		memmove(src->ranks, src->ranks + 1,
+			sizeof(int) * (size_t)src->size);
+	}
+}
+
 void	op_sa(t_stack *a, int emit)
 {
 	stack_swap(a);
@@ -41,3 +65,15 @@ void	op_ss(t_stack *a, t_stack *b, int emit)
 	stack_swap(b);
 	emit_op("ss\n", emit);
 }
+
+void	op_pa(t_stack *a, t_stack *b, int emit)
+{
+	stack_push(a, b);
+	emit_op("pa\n", emit);
+}
+
+void	op_pb(t_stack *a, t_stack *b, int emit)
+{
+	stack_push(b, a);
+	emit_op("pb\n", emit);
+}


## `feat(operation): 스택 정방향 회전을 구현`

diff --git a/include/push_swap.h b/include/push_swap.h
index c038438..da59c0a 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -21,12 +21,16 @@ int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
 
 void	stack_swap(t_stack *stack);
 void	stack_push(t_stack *dst, t_stack *src);
+void	stack_rotate(t_stack *stack);
 
 void	op_sa(t_stack *a, int emit);
 void	op_sb(t_stack *b, int emit);
 void	op_ss(t_stack *a, t_stack *b, int emit);
 void	op_pa(t_stack *a, t_stack *b, int emit);
 void	op_pb(t_stack *a, t_stack *b, int emit);
+void	op_ra(t_stack *a, int emit);
+void	op_rb(t_stack *b, int emit);
+void	op_rr(t_stack *a, t_stack *b, int emit);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/operations.c b/src/operations.c
index 856912e..7cadd4b 100644
--- a/src/operations.c
+++ b/src/operations.c
@@ -47,6 +47,23 @@ void	stack_push(t_stack *dst, t_stack *src)
 	}
 }
 
+void	stack_rotate(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[0];
+	rank = stack->ranks[0];
+	memmove(stack->values, stack->values + 1,
+		sizeof(int) * (size_t)(stack->size - 1));
+	memmove(stack->ranks, stack->ranks + 1,
+		sizeof(int) * (size_t)(stack->size - 1));
+	stack->values[stack->size - 1] = value;
+	stack->ranks[stack->size - 1] = rank;
+}
+
 void	op_sa(t_stack *a, int emit)
 {
 	stack_swap(a);
@@ -77,3 +94,22 @@ void	op_pb(t_stack *a, t_stack *b, int emit)
 	stack_push(b, a);
 	emit_op("pb\n", emit);
 }
+
+void	op_ra(t_stack *a, int emit)
+{
+	stack_rotate(a);
+	emit_op("ra\n", emit);
+}
+
+void	op_rb(t_stack *b, int emit)
+{
+	stack_rotate(b);
+	emit_op("rb\n", emit);
+}
+
+void	op_rr(t_stack *a, t_stack *b, int emit)
+{
+	stack_rotate(a);
+	stack_rotate(b);
+	emit_op("rr\n", emit);
+}


## `feat(operation): 스택 역방향 회전을 구현`

diff --git a/include/push_swap.h b/include/push_swap.h
index da59c0a..76f914c 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -22,6 +22,7 @@ int		stack_is_complete_sorted(const t_stack *a, const t_stack *b);
 void	stack_swap(t_stack *stack);
 void	stack_push(t_stack *dst, t_stack *src);
 void	stack_rotate(t_stack *stack);
+void	stack_reverse_rotate(t_stack *stack);
 
 void	op_sa(t_stack *a, int emit);
 void	op_sb(t_stack *b, int emit);
@@ -31,6 +32,9 @@ void	op_pb(t_stack *a, t_stack *b, int emit);
 void	op_ra(t_stack *a, int emit);
 void	op_rb(t_stack *b, int emit);
 void	op_rr(t_stack *a, t_stack *b, int emit);
+void	op_rra(t_stack *a, int emit);
+void	op_rrb(t_stack *b, int emit);
+void	op_rrr(t_stack *a, t_stack *b, int emit);
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/operations.c b/src/operations.c
index 7cadd4b..bc799f5 100644
--- a/src/operations.c
+++ b/src/operations.c
@@ -64,6 +64,23 @@ void	stack_rotate(t_stack *stack)
 	stack->ranks[stack->size - 1] = rank;
 }
 
+void	stack_reverse_rotate(t_stack *stack)
+{
+	int	value;
+	int	rank;
+
+	if (stack->size < 2)
+		return ;
+	value = stack->values[stack->size - 1];
+	rank = stack->ranks[stack->size - 1];
+	memmove(stack->values + 1, stack->values,
+		sizeof(int) * (size_t)(stack->size - 1));
+	memmove(stack->ranks + 1, stack->ranks,
+		sizeof(int) * (size_t)(stack->size - 1));
+	stack->values[0] = value;
+	stack->ranks[0] = rank;
+}
+
 void	op_sa(t_stack *a, int emit)
 {
 	stack_swap(a);
@@ -113,3 +130,22 @@ void	op_rr(t_stack *a, t_stack *b, int emit)
 	stack_rotate(b);
 	emit_op("rr\n", emit);
 }
+
+void	op_rra(t_stack *a, int emit)
+{
+	stack_reverse_rotate(a);
+	emit_op("rra\n", emit);
+}
+
+void	op_rrb(t_stack *b, int emit)
+{
+	stack_reverse_rotate(b);
+	emit_op("rrb\n", emit);
+}
+
+void	op_rrr(t_stack *a, t_stack *b, int emit)
+{
+	stack_reverse_rotate(a);
+	stack_reverse_rotate(b);
+	emit_op("rrr\n", emit);
+}


## `test(operation): 값과 순위의 보존 불변식을 검증`

diff --git a/Makefile b/Makefile
index 6b3f5f2..e8d141f 100644
--- a/Makefile
+++ b/Makefile
@@ -8,14 +8,18 @@ COMMON_SRCS := \
 	src/operations.c \
 	src/utils.c
 COMMON_OBJS := $(COMMON_SRCS:src/%.c=$(OBJ_DIR)/%.o)
+OPERATION_TEST := $(OBJ_DIR)/operation_invariants
 
-.PHONY: all clean fclean re
+.PHONY: all clean fclean re test
 
 all: $(COMMON_OBJS)
 
 $(OBJ_DIR)/%.o: src/%.c include/push_swap.h | $(OBJ_DIR)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
 
+$(OPERATION_TEST): tests/operation_invariants.c $(COMMON_OBJS)
+	$(CC) $(CFLAGS) $(CPPFLAGS) $^ -o $@
+
 $(OBJ_DIR):
 	mkdir -p $(OBJ_DIR)
 
@@ -26,3 +30,6 @@ fclean: clean
 	rm -f push_swap checker
 
 re: fclean all
+
+test: all $(OPERATION_TEST)
+	$(OPERATION_TEST)
diff --git a/tests/operation_invariants.c b/tests/operation_invariants.c
new file mode 100644
index 0000000..24cf2f7
--- /dev/null
+++ b/tests/operation_invariants.c
@@ -0,0 +1,190 @@
+#include "push_swap.h"
+
+#include <stdio.h>
+
+typedef struct s_fixture
+{
+	t_stack	a;
+	t_stack	b;
+}	t_fixture;
+
+static int	expected_rank(int value)
+{
+	if (value == 40)
+		return (4);
+	if (value == 10)
+		return (1);
+	if (value == 30)
+		return (3);
+	if (value == 20)
+		return (2);
+	if (value == 0)
+		return (0);
+	return (-1);
+}
+
+static int	stack_pairs_are_valid(const t_stack *stack)
+{
+	int	i;
+
+	if (stack->size < 0 || stack->size > stack->capacity)
+		return (0);
+	i = 0;
+	while (i < stack->size)
+	{
+		if (expected_rank(stack->values[i]) != stack->ranks[i])
+			return (0);
+		i++;
+	}
+	return (1);
+}
+
+static int	all_pairs_are_present(const t_fixture *fixture)
+{
+	int	seen[5];
+	int	i;
+	int	rank;
+
+	i = 0;
+	while (i < 5)
+		seen[i++] = 0;
+	i = 0;
+	while (i < fixture->a.size)
+	{
+		rank = fixture->a.ranks[i++];
+		if (rank < 0 || rank >= 5 || seen[rank])
+			return (0);
+		seen[rank] = 1;
+	}
+	i = 0;
+	while (i < fixture->b.size)
+	{
+		rank = fixture->b.ranks[i++];
+		if (rank < 0 || rank >= 5 || seen[rank])
+			return (0);
+		seen[rank] = 1;
+	}
+	i = 0;
+	while (i < 5)
+	{
+		if (!seen[i++])
+			return (0);
+	}
+	return (1);
+}
+
+static int	fixture_init(t_fixture *fixture)
+{
+	if (!stack_init(&fixture->a, 5))
+		return (0);
+	if (!stack_init(&fixture->b, 5))
+	{
+		stack_free(&fixture->a);
+		return (0);
+	}
+	fixture->a.values[0] = 40;
+	fixture->a.ranks[0] = 4;
+	fixture->a.values[1] = 10;
+	fixture->a.ranks[1] = 1;
+	fixture->a.values[2] = 30;
+	fixture->a.ranks[2] = 3;
+	fixture->a.size = 3;
+	fixture->b.values[0] = 20;
+	fixture->b.ranks[0] = 2;
+	fixture->b.values[1] = 0;
+	fixture->b.ranks[1] = 0;
+	fixture->b.size = 2;
+	return (1);
+}
+
+static void	fixture_free(t_fixture *fixture)
+{
+	stack_free(&fixture->a);
+	stack_free(&fixture->b);
+}
+
+static int	fixture_is_valid(const char *operation, const t_fixture *fixture)
+{
+	if (!stack_pairs_are_valid(&fixture->a)
+		|| !stack_pairs_are_valid(&fixture->b)
+		|| fixture->a.size + fixture->b.size != 5
+		|| !all_pairs_are_present(fixture))
+	{
+		fprintf(stderr, "%s broke the value/rank pairing invariant\n",
+			operation);
+		return (0);
+	}
+	return (1);
+}
+
+static int	check_fixture(const char *operation, t_fixture *fixture)
+{
+	int	valid;
+
+	valid = fixture_is_valid(operation, fixture);
+	fixture_free(fixture);
+	return (valid);
+}
+
+static int	test_operation_sequence(void)
+{
+	t_fixture	fixture;
+
+	if (!fixture_init(&fixture))
+		return (0);
+#define APPLY_AND_CHECK(name, call) \
+	do { \
+		call; \
+		if (!fixture_is_valid(name " in sequence", &fixture)) \
+			return (fixture_free(&fixture), 0); \
+	} while (0)
+	APPLY_AND_CHECK("sa", op_sa(&fixture.a, 0));
+	APPLY_AND_CHECK("sb", op_sb(&fixture.b, 0));
+	APPLY_AND_CHECK("ss", op_ss(&fixture.a, &fixture.b, 0));
+	APPLY_AND_CHECK("pa", op_pa(&fixture.a, &fixture.b, 0));
+	APPLY_AND_CHECK("pb", op_pb(&fixture.a, &fixture.b, 0));
+	APPLY_AND_CHECK("ra", op_ra(&fixture.a, 0));
+	APPLY_AND_CHECK("rb", op_rb(&fixture.b, 0));
+	APPLY_AND_CHECK("rr", op_rr(&fixture.a, &fixture.b, 0));
+	APPLY_AND_CHECK("rra", op_rra(&fixture.a, 0));
+	APPLY_AND_CHECK("rrb", op_rrb(&fixture.b, 0));
+	APPLY_AND_CHECK("rrr", op_rrr(&fixture.a, &fixture.b, 0));
+#undef APPLY_AND_CHECK
+	fixture_free(&fixture);
+	return (1);
+}
+
+static int	test_operations(void)
+{
+	t_fixture	fixture;
+
+#define RUN_OPERATION(name, call) \
+	do { \
+		if (!fixture_init(&fixture)) \
+			return (0); \
+		call; \
+		if (!check_fixture(name, &fixture)) \
+			return (0); \
+	} while (0)
+	RUN_OPERATION("sa", op_sa(&fixture.a, 0));
+	RUN_OPERATION("sb", op_sb(&fixture.b, 0));
+	RUN_OPERATION("ss", op_ss(&fixture.a, &fixture.b, 0));
+	RUN_OPERATION("pa", op_pa(&fixture.a, &fixture.b, 0));
+	RUN_OPERATION("pb", op_pb(&fixture.a, &fixture.b, 0));
+	RUN_OPERATION("ra", op_ra(&fixture.a, 0));
+	RUN_OPERATION("rb", op_rb(&fixture.b, 0));
+	RUN_OPERATION("rr", op_rr(&fixture.a, &fixture.b, 0));
+	RUN_OPERATION("rra", op_rra(&fixture.a, 0));
+	RUN_OPERATION("rrb", op_rrb(&fixture.b, 0));
+	RUN_OPERATION("rrr", op_rrr(&fixture.a, &fixture.b, 0));
+#undef RUN_OPERATION
+	return (1);
+}
+
+int	main(void)
+{
+	if (!test_operations() || !test_operation_sequence())
+		return (1);
+	printf("operation pairing invariants passed\n");
+	return (0);
+}


## `test(operation): 정확한 상태 전이와 no-op을 검증`

diff --git a/tests/operation_invariants.c b/tests/operation_invariants.c
index 24cf2f7..642513e 100644
--- a/tests/operation_invariants.c
+++ b/tests/operation_invariants.c
@@ -8,6 +8,32 @@ typedef struct s_fixture
 	t_stack	b;
 }	t_fixture;
 
+typedef enum e_test_operation
+{
+	TEST_SA,
+	TEST_SB,
+	TEST_SS,
+	TEST_PA,
+	TEST_PB,
+	TEST_RA,
+	TEST_RB,
+	TEST_RR,
+	TEST_RRA,
+	TEST_RRB,
+	TEST_RRR,
+	TEST_OPERATION_COUNT
+}	t_test_operation;
+
+typedef struct s_expected_fixture
+{
+	int	a_values[5];
+	int	a_ranks[5];
+	int	a_size;
+	int	b_values[5];
+	int	b_ranks[5];
+	int	b_size;
+}	t_expected_fixture;
+
 static int	expected_rank(int value)
 {
 	if (value == 40)
@@ -103,6 +129,179 @@ static void	fixture_free(t_fixture *fixture)
 	stack_free(&fixture->b);
 }
 
+static const char	*test_operation_name(t_test_operation operation)
+{
+	static const char *const	names[TEST_OPERATION_COUNT] = {
+		"sa", "sb", "ss", "pa", "pb", "ra",
+		"rb", "rr", "rra", "rrb", "rrr"
+	};
+
+	return (names[operation]);
+}
+
+static void	apply_test_operation(t_test_operation operation,
+		t_fixture *fixture)
+{
+	if (operation == TEST_SA)
+		op_sa(&fixture->a, 0);
+	else if (operation == TEST_SB)
+		op_sb(&fixture->b, 0);
+	else if (operation == TEST_SS)
+		op_ss(&fixture->a, &fixture->b, 0);
+	else if (operation == TEST_PA)
+		op_pa(&fixture->a, &fixture->b, 0);
+	else if (operation == TEST_PB)
+		op_pb(&fixture->a, &fixture->b, 0);
+	else if (operation == TEST_RA)
+		op_ra(&fixture->a, 0);
+	else if (operation == TEST_RB)
+		op_rb(&fixture->b, 0);
+	else if (operation == TEST_RR)
+		op_rr(&fixture->a, &fixture->b, 0);
+	else if (operation == TEST_RRA)
+		op_rra(&fixture->a, 0);
+	else if (operation == TEST_RRB)
+		op_rrb(&fixture->b, 0);
+	else
+		op_rrr(&fixture->a, &fixture->b, 0);
+}
+
+static int	stack_has_exact_state(const t_stack *stack, const int *values,
+		const int *ranks, int size)
+{
+	int	i;
+
+	if (stack->size != size || stack->capacity != 5)
+		return (0);
+	i = 0;
+	while (i < size)
+	{
+		if (stack->values[i] != values[i] || stack->ranks[i] != ranks[i])
+			return (0);
+		i++;
+	}
+	return (1);
+}
+
+static int	fixture_has_exact_state(const t_fixture *fixture,
+		const t_expected_fixture *expected)
+{
+	return (stack_has_exact_state(&fixture->a, expected->a_values,
+			expected->a_ranks, expected->a_size)
+		&& stack_has_exact_state(&fixture->b, expected->b_values,
+			expected->b_ranks, expected->b_size));
+}
+
+static int	test_exact_operation_states(void)
+{
+	static const t_expected_fixture	expected[TEST_OPERATION_COUNT] = {
+	{{10, 40, 30}, {1, 4, 3}, 3, {20, 0}, {2, 0}, 2},
+	{{40, 10, 30}, {4, 1, 3}, 3, {0, 20}, {0, 2}, 2},
+	{{10, 40, 30}, {1, 4, 3}, 3, {0, 20}, {0, 2}, 2},
+	{{20, 40, 10, 30}, {2, 4, 1, 3}, 4, {0}, {0}, 1},
+	{{10, 30}, {1, 3}, 2, {40, 20, 0}, {4, 2, 0}, 3},
+	{{10, 30, 40}, {1, 3, 4}, 3, {20, 0}, {2, 0}, 2},
+	{{40, 10, 30}, {4, 1, 3}, 3, {0, 20}, {0, 2}, 2},
+	{{10, 30, 40}, {1, 3, 4}, 3, {0, 20}, {0, 2}, 2},
+	{{30, 40, 10}, {3, 4, 1}, 3, {20, 0}, {2, 0}, 2},
+	{{40, 10, 30}, {4, 1, 3}, 3, {0, 20}, {0, 2}, 2},
+	{{30, 40, 10}, {3, 4, 1}, 3, {0, 20}, {0, 2}, 2}
+	};
+	t_fixture	fixture;
+	int			index;
+
+	index = 0;
+	while (index < TEST_OPERATION_COUNT)
+	{
+		if (!fixture_init(&fixture))
+			return (0);
+		apply_test_operation((t_test_operation)index, &fixture);
+		if (!fixture_has_exact_state(&fixture, &expected[index]))
+		{
+			fprintf(stderr, "%s produced an unexpected stack state\n",
+				test_operation_name((t_test_operation)index));
+			fixture_free(&fixture);
+			return (0);
+		}
+		fixture_free(&fixture);
+		index++;
+	}
+	return (1);
+}
+
+static int	small_fixture_init(t_fixture *fixture, int a_size, int b_size)
+{
+	if (!stack_init(&fixture->a, 2))
+		return (0);
+	if (!stack_init(&fixture->b, 2))
+		return (stack_free(&fixture->a), 0);
+	fixture->a.values[0] = 101;
+	fixture->a.values[1] = 102;
+	fixture->a.ranks[0] = 11;
+	fixture->a.ranks[1] = 12;
+	fixture->b.values[0] = 201;
+	fixture->b.values[1] = 202;
+	fixture->b.ranks[0] = 21;
+	fixture->b.ranks[1] = 22;
+	fixture->a.size = a_size;
+	fixture->b.size = b_size;
+	return (1);
+}
+
+static int	small_stack_is_unchanged(const t_stack *stack,
+		int first_value, int second_value, int first_rank, int second_rank,
+		int size)
+{
+	return (stack->capacity == 2 && stack->size == size
+		&& stack->values[0] == first_value
+		&& stack->values[1] == second_value
+		&& stack->ranks[0] == first_rank
+		&& stack->ranks[1] == second_rank);
+}
+
+static int	run_noop_case(t_test_operation operation, int a_size, int b_size)
+{
+	t_fixture	fixture;
+	int			valid;
+
+	if (!small_fixture_init(&fixture, a_size, b_size))
+		return (0);
+	apply_test_operation(operation, &fixture);
+	valid = small_stack_is_unchanged(&fixture.a, 101, 102, 11, 12, a_size)
+		&& small_stack_is_unchanged(&fixture.b, 201, 202, 21, 22, b_size);
+	if (!valid)
+		fprintf(stderr, "%s changed an insufficient stack\n",
+			test_operation_name(operation));
+	fixture_free(&fixture);
+	return (valid);
+}
+
+static int	test_operation_noops(void)
+{
+	int	index;
+
+	index = 0;
+	while (index < TEST_OPERATION_COUNT)
+	{
+		if (!run_noop_case((t_test_operation)index, 0, 0))
+			return (0);
+		if (index == TEST_PA)
+		{
+			if (!run_noop_case(TEST_PA, 1, 0))
+				return (0);
+		}
+		else if (index == TEST_PB)
+		{
+			if (!run_noop_case(TEST_PB, 0, 1))
+				return (0);
+		}
+		else if (!run_noop_case((t_test_operation)index, 1, 1))
+			return (0);
+		index++;
+	}
+	return (1);
+}
+
 static int	fixture_is_valid(const char *operation, const t_fixture *fixture)
 {
 	if (!stack_pairs_are_valid(&fixture->a)
@@ -183,8 +382,9 @@ static int	test_operations(void)
 
 int	main(void)
 {
-	if (!test_operations() || !test_operation_sequence())
+	if (!test_operations() || !test_operation_sequence()
+		|| !test_exact_operation_states() || !test_operation_noops())
 		return (1);
-	printf("operation pairing invariants passed\n");
+	printf("operation invariants passed\n");
 	return (0);
 }
