# 연결 리스트 구조와 소유권 수명

## `feat(list): 연결 리스트 노드 생성과 앞 삽입을 구현`

diff --git a/Makefile b/Makefile
index ec7dd83..a37e165 100644
--- a/Makefile
+++ b/Makefile
@@ -24,7 +24,8 @@ SRC := \
 	src/convert/ft_atoi.c \
 	src/convert/ft_itoa.c \
 	src/alloc/ft_allocate.c \
-	src/io/ft_fd_output.c
+	src/io/ft_fd_output.c \
+	src/list/ft_list_basic.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 318bb23..00acb50 100644
--- a/libft.h
+++ b/libft.h
@@ -3,13 +3,19 @@
 
 # include <stddef.h>
 
-int	ft_isalpha(int c);
-int	ft_isdigit(int c);
-int	ft_isalnum(int c);
-int	ft_isascii(int c);
-int	ft_isprint(int c);
-int	ft_toupper(int c);
-int	ft_tolower(int c);
+typedef struct s_list
+{
+	void			*content;
+	struct s_list	*next;
+} t_list;
+
+int		ft_isalpha(int c);
+int		ft_isdigit(int c);
+int		ft_isalnum(int c);
+int		ft_isascii(int c);
+int		ft_isprint(int c);
+int		ft_toupper(int c);
+int		ft_tolower(int c);
 
 void	*ft_memset(void *memory, int byte, size_t length);
 void	ft_bzero(void *memory, size_t length);
@@ -42,4 +48,7 @@ void	ft_putstr_fd(char *text, int fd);
 void	ft_putendl_fd(char *text, int fd);
 void	ft_putnbr_fd(int number, int fd);
 
+t_list	*ft_lstnew(void *content);
+void	ft_lstadd_front(t_list **list, t_list *node);
+
 #endif
diff --git a/src/list/ft_list_basic.c b/src/list/ft_list_basic.c
new file mode 100644
index 0000000..1b0acc8
--- /dev/null
+++ b/src/list/ft_list_basic.c
@@ -0,0 +1,23 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+t_list	*ft_lstnew(void *content)
+{
+	t_list	*node;
+
+	node = malloc(sizeof(t_list));
+	if (node == NULL)
+		return (NULL);
+	node->content = content;
+	node->next = NULL;
+	return (node);
+}
+
+void	ft_lstadd_front(t_list **list, t_list *node)
+{
+	if (list == NULL || node == NULL)
+		return ;
+	node->next = *list;
+	*list = node;
+}


## `feat(list): 연결 리스트 크기와 마지막 노드를 조회`

diff --git a/libft.h b/libft.h
index 00acb50..c0316e7 100644
--- a/libft.h
+++ b/libft.h
@@ -50,5 +50,7 @@ void	ft_putnbr_fd(int number, int fd);
 
 t_list	*ft_lstnew(void *content);
 void	ft_lstadd_front(t_list **list, t_list *node);
+int		ft_lstsize(t_list *list);
+t_list	*ft_lstlast(t_list *list);
 
 #endif
diff --git a/src/list/ft_list_basic.c b/src/list/ft_list_basic.c
index 1b0acc8..3205d69 100644
--- a/src/list/ft_list_basic.c
+++ b/src/list/ft_list_basic.c
@@ -21,3 +21,25 @@ void	ft_lstadd_front(t_list **list, t_list *node)
 	node->next = *list;
 	*list = node;
 }
+
+int	ft_lstsize(t_list *list)
+{
+	int	size;
+
+	size = 0;
+	while (list != NULL)
+	{
+		size++;
+		list = list->next;
+	}
+	return (size);
+}
+
+t_list	*ft_lstlast(t_list *list)
+{
+	if (list == NULL)
+		return (NULL);
+	while (list->next != NULL)
+		list = list->next;
+	return (list);
+}


## `feat(list): 연결 리스트 뒤 삽입을 구현`

diff --git a/libft.h b/libft.h
index c0316e7..16006f6 100644
--- a/libft.h
+++ b/libft.h
@@ -52,5 +52,6 @@ t_list	*ft_lstnew(void *content);
 void	ft_lstadd_front(t_list **list, t_list *node);
 int		ft_lstsize(t_list *list);
 t_list	*ft_lstlast(t_list *list);
+void	ft_lstadd_back(t_list **list, t_list *node);
 
 #endif
diff --git a/src/list/ft_list_basic.c b/src/list/ft_list_basic.c
index 3205d69..ffa104c 100644
--- a/src/list/ft_list_basic.c
+++ b/src/list/ft_list_basic.c
@@ -43,3 +43,18 @@ t_list	*ft_lstlast(t_list *list)
 		list = list->next;
 	return (list);
 }
+
+void	ft_lstadd_back(t_list **list, t_list *node)
+{
+	t_list	*last;
+
+	if (list == NULL || node == NULL)
+		return ;
+	if (*list == NULL)
+	{
+		*list = node;
+		return ;
+	}
+	last = ft_lstlast(*list);
+	last->next = node;
+}


## `test(list): 노드 생성과 삽입 불변식 검증`

diff --git a/tests/test.h b/tests/test.h
index 1fff33b..bc49bed 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -18,6 +18,7 @@ void	test_split(void);
 void	test_itoa(void);
 void	test_string_transform(void);
 void	test_fd_output(void);
+void	test_list_basic(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_list_basic.c b/tests/test_list_basic.c
new file mode 100644
index 0000000..7101c80
--- /dev/null
+++ b/tests/test_list_basic.c
@@ -0,0 +1,60 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+
+void	test_list_basic(void)
+{
+	int		first_content;
+	int		second_content;
+	int		third_content;
+	t_list		*list;
+	t_list		*first;
+	t_list		*second;
+	t_list		*third;
+
+	first_content = 11;
+	second_content = 22;
+	third_content = 33;
+	list = NULL;
+	CHECK(ft_lstsize(list) == 0);
+	CHECK(ft_lstlast(list) == NULL);
+	first = ft_lstnew(&first_content);
+	second = ft_lstnew(&second_content);
+	third = ft_lstnew(&third_content);
+	CHECK(first != NULL && second != NULL && third != NULL);
+	if (first == NULL || second == NULL || third == NULL)
+	{
+		free(first);
+		free(second);
+		free(third);
+		return ;
+	}
+	CHECK(first->content == &first_content && first->next == NULL);
+	CHECK(second->content == &second_content && second->next == NULL);
+	ft_lstadd_front(&list, second);
+	ft_lstadd_front(&list, first);
+	ft_lstadd_back(&list, third);
+	CHECK(list == first);
+	CHECK(list->next == second);
+	CHECK(second->next == third);
+	CHECK(third->next == NULL);
+	CHECK(ft_lstsize(list) == 3);
+	CHECK(ft_lstlast(list) == third);
+	ft_lstadd_front(NULL, NULL);
+	ft_lstadd_front(&list, NULL);
+	ft_lstadd_back(NULL, NULL);
+	ft_lstadd_back(&list, NULL);
+	CHECK(ft_lstsize(list) == 3);
+	free(first);
+	free(second);
+	free(third);
+	first = ft_lstnew(NULL);
+	CHECK(first != NULL);
+	if (first != NULL)
+	{
+		CHECK(first->content == NULL && first->next == NULL);
+		free(first);
+	}
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index f49b011..9ed5918 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -43,5 +43,6 @@ int	main(void)
 	test_itoa();
 	test_string_transform();
 	test_fd_output();
+	test_list_basic();
 	return (test_finish());
 }


## `feat(list): 연결 리스트 순회와 삭제 구현`

diff --git a/Makefile b/Makefile
index a37e165..c1d5a49 100644
--- a/Makefile
+++ b/Makefile
@@ -25,7 +25,8 @@ SRC := \
 	src/convert/ft_itoa.c \
 	src/alloc/ft_allocate.c \
 	src/io/ft_fd_output.c \
-	src/list/ft_list_basic.c
+	src/list/ft_list_basic.c \
+	src/list/ft_list_lifecycle.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 16006f6..2bdf4c1 100644
--- a/libft.h
+++ b/libft.h
@@ -53,5 +53,8 @@ void	ft_lstadd_front(t_list **list, t_list *node);
 int		ft_lstsize(t_list *list);
 t_list	*ft_lstlast(t_list *list);
 void	ft_lstadd_back(t_list **list, t_list *node);
+void	ft_lstdelone(t_list *node, void (*del)(void *));
+void	ft_lstclear(t_list **list, void (*del)(void *));
+void	ft_lstiter(t_list *list, void (*function)(void *));
 
 #endif
diff --git a/src/list/ft_list_lifecycle.c b/src/list/ft_list_lifecycle.c
new file mode 100644
index 0000000..0a6763a
--- /dev/null
+++ b/src/list/ft_list_lifecycle.c
@@ -0,0 +1,36 @@
+#include "libft.h"
+
+#include <stdlib.h>
+
+void	ft_lstdelone(t_list *node, void (*del)(void *))
+{
+	if (node == NULL || del == NULL)
+		return ;
+	del(node->content);
+	free(node);
+}
+
+void	ft_lstclear(t_list **list, void (*del)(void *))
+{
+	t_list	*next;
+
+	if (list == NULL || del == NULL)
+		return ;
+	while (*list != NULL)
+	{
+		next = (*list)->next;
+		ft_lstdelone(*list, del);
+		*list = next;
+	}
+}
+
+void	ft_lstiter(t_list *list, void (*function)(void *))
+{
+	if (function == NULL)
+		return ;
+	while (list != NULL)
+	{
+		function(list->content);
+		list = list->next;
+	}
+}


## `test(list): 순회와 삭제 수명 검증`

diff --git a/tests/test.h b/tests/test.h
index bc49bed..fde2e31 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -19,6 +19,7 @@ void	test_itoa(void);
 void	test_string_transform(void);
 void	test_fd_output(void);
 void	test_list_basic(void);
+void	test_list_lifecycle(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_list_lifecycle.c b/tests/test_list_lifecycle.c
new file mode 100644
index 0000000..f108bc8
--- /dev/null
+++ b/tests/test_list_lifecycle.c
@@ -0,0 +1,131 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+
+static int	g_delete_count;
+static int	g_delete_order[4];
+static int	g_iterate_count;
+
+static int	*new_integer(int value)
+{
+	int	*number;
+
+	number = malloc(sizeof(int));
+	if (number != NULL)
+		*number = value;
+	return (number);
+}
+
+static void	delete_integer(void *content)
+{
+	int	*number;
+
+	number = content;
+	if (number != NULL && g_delete_count < 4)
+		g_delete_order[g_delete_count] = *number;
+	g_delete_count++;
+	free(number);
+}
+
+static void	increment_integer(void *content)
+{
+	int	*number;
+
+	number = content;
+	CHECK(number != NULL);
+	if (number != NULL)
+		*number += g_iterate_count + 1;
+	g_iterate_count++;
+}
+
+static t_list	*new_integer_node(int value)
+{
+	int	*content;
+	t_list	*node;
+
+	content = new_integer(value);
+	if (content == NULL)
+		return (NULL);
+	node = ft_lstnew(content);
+	if (node == NULL)
+		free(content);
+	return (node);
+}
+
+void	test_list_lifecycle(void)
+{
+	t_list	*list;
+	t_list	*first;
+	t_list	*second;
+	t_list	*third;
+	int	stack_content;
+
+	first = new_integer_node(10);
+	second = new_integer_node(20);
+	third = new_integer_node(30);
+	CHECK(first != NULL && second != NULL && third != NULL);
+	if (first == NULL || second == NULL || third == NULL)
+	{
+		if (first != NULL)
+			ft_lstdelone(first, delete_integer);
+		if (second != NULL)
+			ft_lstdelone(second, delete_integer);
+		if (third != NULL)
+			ft_lstdelone(third, delete_integer);
+		return ;
+	}
+	list = first;
+	first->next = second;
+	second->next = third;
+	g_iterate_count = 0;
+	ft_lstiter(list, increment_integer);
+	CHECK(g_iterate_count == 3);
+	CHECK(*(int *)first->content == 11);
+	CHECK(*(int *)second->content == 22);
+	CHECK(*(int *)third->content == 33);
+	g_delete_count = 0;
+	ft_lstclear(&list, delete_integer);
+	CHECK(list == NULL);
+	CHECK(g_delete_count == 3);
+	CHECK(g_delete_order[0] == 11);
+	CHECK(g_delete_order[1] == 22);
+	CHECK(g_delete_order[2] == 33);
+	first = new_integer_node(44);
+	CHECK(first != NULL);
+	if (first != NULL)
+	{
+		g_delete_count = 0;
+		ft_lstdelone(first, delete_integer);
+		CHECK(g_delete_count == 1 && g_delete_order[0] == 44);
+	}
+	stack_content = 55;
+	first = ft_lstnew(&stack_content);
+	CHECK(first != NULL);
+	if (first != NULL)
+	{
+		ft_lstdelone(first, NULL);
+		CHECK(first->content == &stack_content && first->next == NULL);
+		free(first);
+	}
+	first = ft_lstnew(&stack_content);
+	second = ft_lstnew(&stack_content);
+	CHECK(first != NULL && second != NULL);
+	if (first != NULL && second != NULL)
+	{
+		first->next = second;
+		list = first;
+		ft_lstclear(&list, NULL);
+		CHECK(list == first && first->next == second);
+		free(second);
+		free(first);
+	}
+	else
+	{
+		free(first);
+		free(second);
+	}
+	ft_lstclear(NULL, delete_integer);
+	ft_lstiter(NULL, increment_integer);
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index 9ed5918..f0bd66d 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -44,5 +44,6 @@ int	main(void)
 	test_string_transform();
 	test_fd_output();
 	test_list_basic();
+	test_list_lifecycle();
 	return (test_finish());
 }


## `feat(list): 실패 시 정리되는 리스트 변환 구현`

diff --git a/Makefile b/Makefile
index c1d5a49..fcc65a4 100644
--- a/Makefile
+++ b/Makefile
@@ -26,7 +26,8 @@ SRC := \
 	src/alloc/ft_allocate.c \
 	src/io/ft_fd_output.c \
 	src/list/ft_list_basic.c \
-	src/list/ft_list_lifecycle.c
+	src/list/ft_list_lifecycle.c \
+	src/list/ft_list_map.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 2bdf4c1..9676ba4 100644
--- a/libft.h
+++ b/libft.h
@@ -56,5 +56,7 @@ void	ft_lstadd_back(t_list **list, t_list *node);
 void	ft_lstdelone(t_list *node, void (*del)(void *));
 void	ft_lstclear(t_list **list, void (*del)(void *));
 void	ft_lstiter(t_list *list, void (*function)(void *));
+t_list	*ft_lstmap(t_list *list, void *(*function)(void *),
+			void (*del)(void *));
 
 #endif
diff --git a/src/list/ft_list_map.c b/src/list/ft_list_map.c
new file mode 100644
index 0000000..0708b77
--- /dev/null
+++ b/src/list/ft_list_map.c
@@ -0,0 +1,33 @@
+#include "libft.h"
+
+t_list	*ft_lstmap(t_list *list, void *(*function)(void *),
+		void (*del)(void *))
+{
+	t_list	*mapped;
+	t_list	*tail;
+	t_list	*node;
+	void	*mapped_content;
+
+	if (function == NULL || del == NULL)
+		return (NULL);
+	mapped = NULL;
+	tail = NULL;
+	while (list != NULL)
+	{
+		mapped_content = function(list->content);
+		node = ft_lstnew(mapped_content);
+		if (node == NULL)
+		{
+			del(mapped_content);
+			ft_lstclear(&mapped, del);
+			return (NULL);
+		}
+		if (mapped == NULL)
+			mapped = node;
+		else
+			tail->next = node;
+		tail = node;
+		list = list->next;
+	}
+	return (mapped);
+}


## `test(list): 리스트 변환과 content 수명 검증`

diff --git a/tests/test.h b/tests/test.h
index fde2e31..ff5387c 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -20,6 +20,7 @@ void	test_string_transform(void);
 void	test_fd_output(void);
 void	test_list_basic(void);
 void	test_list_lifecycle(void);
+void	test_list_map(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_list_map.c b/tests/test_list_map.c
new file mode 100644
index 0000000..f96adcc
--- /dev/null
+++ b/tests/test_list_map.c
@@ -0,0 +1,96 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stdlib.h>
+
+static int	g_map_delete_count;
+
+static void	*double_integer(void *content)
+{
+	int	*mapped;
+
+	mapped = malloc(sizeof(int));
+	if (mapped != NULL)
+		*mapped = *(int *)content * 2;
+	return (mapped);
+}
+
+static void	*map_null_content(void *content)
+{
+	(void)content;
+	return (NULL);
+}
+
+static void	delete_mapped_integer(void *content)
+{
+	g_map_delete_count++;
+	free(content);
+}
+
+static void	keep_source_content(void *content)
+{
+	(void)content;
+}
+
+void	test_list_map(void)
+{
+	int		a;
+	int		b;
+	int		c;
+	t_list		*source;
+	t_list		*mapped;
+	t_list		*first;
+	t_list		*second;
+	t_list		*third;
+
+	a = 3;
+	b = 5;
+	c = 8;
+	first = ft_lstnew(&a);
+	second = ft_lstnew(&b);
+	third = ft_lstnew(&c);
+	CHECK(first != NULL && second != NULL && third != NULL);
+	if (first == NULL || second == NULL || third == NULL)
+	{
+		free(first);
+		free(second);
+		free(third);
+		return ;
+	}
+	source = first;
+	first->next = second;
+	second->next = third;
+	mapped = ft_lstmap(source, double_integer, delete_mapped_integer);
+	CHECK(mapped != NULL);
+	if (mapped != NULL)
+	{
+		CHECK(ft_lstsize(mapped) == 3);
+		CHECK(mapped != source && mapped->next != second);
+		CHECK(mapped->content != source->content);
+		CHECK(*(int *)mapped->content == 6);
+		CHECK(*(int *)mapped->next->content == 10);
+		CHECK(*(int *)mapped->next->next->content == 16);
+		CHECK(a == 3 && b == 5 && c == 8);
+		g_map_delete_count = 0;
+		ft_lstclear(&mapped, delete_mapped_integer);
+		CHECK(mapped == NULL && g_map_delete_count == 3);
+	}
+	g_map_delete_count = 0;
+	mapped = ft_lstmap(source, map_null_content, delete_mapped_integer);
+	CHECK(mapped != NULL);
+	if (mapped != NULL)
+	{
+		CHECK(ft_lstsize(mapped) == 3);
+		CHECK(mapped->content == NULL);
+		CHECK(mapped->next->content == NULL);
+		CHECK(mapped->next->next->content == NULL);
+		ft_lstclear(&mapped, delete_mapped_integer);
+		CHECK(mapped == NULL && g_map_delete_count == 3);
+	}
+	CHECK(ft_lstmap(source, NULL, delete_mapped_integer) == NULL);
+	CHECK(ft_lstmap(source, double_integer, NULL) == NULL);
+	CHECK(ft_lstmap(NULL, double_integer, delete_mapped_integer) == NULL);
+	ft_lstclear(&source, keep_source_content);
+	CHECK(source == NULL);
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index f0bd66d..87bfab6 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -45,5 +45,6 @@ int	main(void)
 	test_fd_output();
 	test_list_basic();
 	test_list_lifecycle();
+	test_list_map();
 	return (test_finish());
 }
