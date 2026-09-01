# 바이트 범위 복사·이동·검색

## `feat(memory): 메모리 채우기와 0 초기화 구현`

diff --git a/Makefile b/Makefile
index 5ee4589..828533b 100644
--- a/Makefile
+++ b/Makefile
@@ -11,7 +11,8 @@ RMDIR := rm -rf
 MKDIR := mkdir -p
 
 SRC := \
-	src/char/ft_char.c
+	src/char/ft_char.c \
+	src/memory/ft_memory_fill.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 8a9193f..b660fac 100644
--- a/libft.h
+++ b/libft.h
@@ -1,6 +1,8 @@
 #ifndef LIBFT_H
 # define LIBFT_H
 
+# include <stddef.h>
+
 int	ft_isalpha(int c);
 int	ft_isdigit(int c);
 int	ft_isalnum(int c);
@@ -9,4 +11,7 @@ int	ft_isprint(int c);
 int	ft_toupper(int c);
 int	ft_tolower(int c);
 
+void	*ft_memset(void *memory, int byte, size_t length);
+void	ft_bzero(void *memory, size_t length);
+
 #endif
diff --git a/src/memory/ft_memory_fill.c b/src/memory/ft_memory_fill.c
new file mode 100644
index 0000000..c4b0428
--- /dev/null
+++ b/src/memory/ft_memory_fill.c
@@ -0,0 +1,20 @@
+#include "libft.h"
+
+void	*ft_memset(void *memory, int byte, size_t length)
+{
+	unsigned char	*byte_pointer;
+
+	byte_pointer = memory;
+	while (length > 0)
+	{
+		*byte_pointer = (unsigned char)byte;
+		byte_pointer++;
+		length--;
+	}
+	return (memory);
+}
+
+void	ft_bzero(void *memory, size_t length)
+{
+	ft_memset(memory, 0, length);
+}


## `feat(memory): 겹치지 않는 메모리 복사 구현`

diff --git a/Makefile b/Makefile
index 828533b..1d10251 100644
--- a/Makefile
+++ b/Makefile
@@ -12,7 +12,8 @@ MKDIR := mkdir -p
 
 SRC := \
 	src/char/ft_char.c \
-	src/memory/ft_memory_fill.c
+	src/memory/ft_memory_fill.c \
+	src/memory/ft_memory_copy.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index b660fac..058669d 100644
--- a/libft.h
+++ b/libft.h
@@ -13,5 +13,6 @@ int	ft_tolower(int c);
 
 void	*ft_memset(void *memory, int byte, size_t length);
 void	ft_bzero(void *memory, size_t length);
+void	*ft_memcpy(void *destination, const void *source, size_t length);
 
 #endif
diff --git a/src/memory/ft_memory_copy.c b/src/memory/ft_memory_copy.c
new file mode 100644
index 0000000..4ce303a
--- /dev/null
+++ b/src/memory/ft_memory_copy.c
@@ -0,0 +1,18 @@
+#include "libft.h"
+
+void	*ft_memcpy(void *destination, const void *source, size_t length)
+{
+	unsigned char		*destination_byte;
+	const unsigned char	*source_byte;
+
+	destination_byte = destination;
+	source_byte = source;
+	while (length > 0)
+	{
+		*destination_byte = *source_byte;
+		destination_byte++;
+		source_byte++;
+		length--;
+	}
+	return (destination);
+}


## `test(memory): 메모리 복사 경계 검증`

diff --git a/tests/test.h b/tests/test.h
index c6404f4..c7f9dca 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -6,6 +6,7 @@ void	test_check(int condition, const char *expression, const char *file,
 int		test_finish(void);
 void	test_char(void);
 void	test_memory_fill(void);
+void	test_memory_copy(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index ad520a8..a10e726 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -31,5 +31,6 @@ int	main(void)
 {
 	test_char();
 	test_memory_fill();
+	test_memory_copy();
 	return (test_finish());
 }
diff --git a/tests/test_memory_copy.c b/tests/test_memory_copy.c
new file mode 100644
index 0000000..129ffbf
--- /dev/null
+++ b/tests/test_memory_copy.c
@@ -0,0 +1,56 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stddef.h>
+#include <string.h>
+
+#define COPY_BUFFER_SIZE 128
+
+static void	seed_copy_buffers(unsigned char *destination,
+		unsigned char *source)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < COPY_BUFFER_SIZE)
+	{
+		destination[index] = (unsigned char)(index * 19U + 7U);
+		source[index] = (unsigned char)(index * 43U + 3U);
+		index++;
+	}
+}
+
+static void	check_copy(size_t destination_offset, size_t source_offset,
+		size_t length)
+{
+	unsigned char	actual[COPY_BUFFER_SIZE];
+	unsigned char	expected[COPY_BUFFER_SIZE];
+	unsigned char	source[COPY_BUFFER_SIZE];
+	unsigned char	source_before[COPY_BUFFER_SIZE];
+
+	seed_copy_buffers(actual, source);
+	memcpy(expected, actual, sizeof(actual));
+	memcpy(source_before, source, sizeof(source));
+	CHECK(ft_memcpy(actual + destination_offset, source + source_offset, length)
+		== actual + destination_offset);
+	memcpy(expected + destination_offset, source + source_offset, length);
+	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
+	CHECK(memcmp(source, source_before, sizeof(source)) == 0);
+}
+
+void	test_memory_copy(void)
+{
+	static const size_t	lengths[] = {0, 1, 2, 3, 7, 8, 15, 16, 31, 64};
+	size_t			index;
+
+	CHECK(ft_memcpy(NULL, NULL, 0) == NULL);
+	index = 0;
+	while (index < sizeof(lengths) / sizeof(lengths[0]))
+	{
+		check_copy(0, 0, lengths[index]);
+		check_copy(1, 3, lengths[index]);
+		check_copy(9, 5, lengths[index]);
+		index++;
+	}
+}


## `feat(memory): 겹치는 메모리의 안전한 이동 구현`

diff --git a/Makefile b/Makefile
index 1d10251..5cce25f 100644
--- a/Makefile
+++ b/Makefile
@@ -13,7 +13,8 @@ MKDIR := mkdir -p
 SRC := \
 	src/char/ft_char.c \
 	src/memory/ft_memory_fill.c \
-	src/memory/ft_memory_copy.c
+	src/memory/ft_memory_copy.c \
+	src/memory/ft_memory_move.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index 058669d..a57a979 100644
--- a/libft.h
+++ b/libft.h
@@ -14,5 +14,6 @@ int	ft_tolower(int c);
 void	*ft_memset(void *memory, int byte, size_t length);
 void	ft_bzero(void *memory, size_t length);
 void	*ft_memcpy(void *destination, const void *source, size_t length);
+void	*ft_memmove(void *destination, const void *source, size_t length);
 
 #endif
diff --git a/src/memory/ft_memory_move.c b/src/memory/ft_memory_move.c
new file mode 100644
index 0000000..082bbb1
--- /dev/null
+++ b/src/memory/ft_memory_move.c
@@ -0,0 +1,29 @@
+#include "libft.h"
+
+void	*ft_memmove(void *destination, const void *source, size_t length)
+{
+	unsigned char		*destination_byte;
+	const unsigned char	*source_byte;
+	size_t			offset;
+
+	destination_byte = destination;
+	source_byte = source;
+	if (destination_byte == source_byte || length == 0)
+		return (destination);
+	offset = 1;
+	while (offset < length)
+	{
+		if (destination_byte == source_byte + offset)
+		{
+			while (length > 0)
+			{
+				length--;
+				destination_byte[length] = source_byte[length];
+			}
+			return (destination);
+		}
+		offset++;
+	}
+	ft_memcpy(destination_byte, source_byte, length);
+	return (destination);
+}


## `test(memory): 겹치는 메모리 이동 검증`

diff --git a/tests/test.h b/tests/test.h
index c7f9dca..c997876 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -7,6 +7,7 @@ int		test_finish(void);
 void	test_char(void);
 void	test_memory_fill(void);
 void	test_memory_copy(void);
+void	test_memory_move(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index a10e726..e947657 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -32,5 +32,6 @@ int	main(void)
 	test_char();
 	test_memory_fill();
 	test_memory_copy();
+	test_memory_move();
 	return (test_finish());
 }
diff --git a/tests/test_memory_move.c b/tests/test_memory_move.c
new file mode 100644
index 0000000..01f8be4
--- /dev/null
+++ b/tests/test_memory_move.c
@@ -0,0 +1,52 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stddef.h>
+#include <string.h>
+
+#define MOVE_BUFFER_SIZE 128
+
+static void	seed_move_buffer(unsigned char *buffer)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < MOVE_BUFFER_SIZE)
+	{
+		buffer[index] = (unsigned char)(index * 29U + 17U);
+		index++;
+	}
+}
+
+static void	check_move(size_t destination_offset, size_t source_offset,
+		size_t length)
+{
+	unsigned char	actual[MOVE_BUFFER_SIZE];
+	unsigned char	expected[MOVE_BUFFER_SIZE];
+
+	seed_move_buffer(actual);
+	memcpy(expected, actual, sizeof(actual));
+	CHECK(ft_memmove(actual + destination_offset, actual + source_offset, length)
+		== actual + destination_offset);
+	memmove(expected + destination_offset, expected + source_offset, length);
+	CHECK(memcmp(actual, expected, sizeof(actual)) == 0);
+}
+
+void	test_memory_move(void)
+{
+	static const size_t	lengths[] = {0, 1, 2, 3, 7, 8, 15, 16, 31, 63};
+	size_t			index;
+
+	CHECK(ft_memmove(NULL, NULL, 0) == NULL);
+	index = 0;
+	while (index < sizeof(lengths) / sizeof(lengths[0]))
+	{
+		check_move(0, 0, lengths[index]);
+		check_move(0, 1, lengths[index]);
+		check_move(1, 0, lengths[index]);
+		check_move(7, 19, lengths[index]);
+		check_move(23, 5, lengths[index]);
+		index++;
+	}
+}


## `feat(memory): 범위를 제한한 메모리 검색과 비교 추가`

diff --git a/Makefile b/Makefile
index 5cce25f..da70ad4 100644
--- a/Makefile
+++ b/Makefile
@@ -14,7 +14,8 @@ SRC := \
 	src/char/ft_char.c \
 	src/memory/ft_memory_fill.c \
 	src/memory/ft_memory_copy.c \
-	src/memory/ft_memory_move.c
+	src/memory/ft_memory_move.c \
+	src/memory/ft_memory_scan.c
 OBJ_DIR := build/obj
 OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
diff --git a/libft.h b/libft.h
index a57a979..8dad255 100644
--- a/libft.h
+++ b/libft.h
@@ -15,5 +15,7 @@ void	*ft_memset(void *memory, int byte, size_t length);
 void	ft_bzero(void *memory, size_t length);
 void	*ft_memcpy(void *destination, const void *source, size_t length);
 void	*ft_memmove(void *destination, const void *source, size_t length);
+void	*ft_memchr(const void *memory, int byte, size_t length);
+int		ft_memcmp(const void *left, const void *right, size_t length);
 
 #endif
diff --git a/src/memory/ft_memory_scan.c b/src/memory/ft_memory_scan.c
new file mode 100644
index 0000000..356c4d1
--- /dev/null
+++ b/src/memory/ft_memory_scan.c
@@ -0,0 +1,35 @@
+#include "libft.h"
+
+void	*ft_memchr(const void *memory, int byte, size_t length)
+{
+	const unsigned char	*memory_byte;
+	size_t			index;
+
+	memory_byte = memory;
+	index = 0;
+	while (index < length)
+	{
+		if (memory_byte[index] == (unsigned char)byte)
+			return ((void *)(memory_byte + index));
+		index++;
+	}
+	return (NULL);
+}
+
+int	ft_memcmp(const void *left, const void *right, size_t length)
+{
+	const unsigned char	*left_byte;
+	const unsigned char	*right_byte;
+	size_t			index;
+
+	left_byte = left;
+	right_byte = right;
+	index = 0;
+	while (index < length)
+	{
+		if (left_byte[index] != right_byte[index])
+			return ((int)left_byte[index] - (int)right_byte[index]);
+		index++;
+	}
+	return (0);
+}


## `test(memory): 메모리 검색과 비교 검증`

diff --git a/tests/test.h b/tests/test.h
index c997876..a54a2b4 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -8,6 +8,7 @@ void	test_char(void);
 void	test_memory_fill(void);
 void	test_memory_copy(void);
 void	test_memory_move(void);
+void	test_memory_scan(void);
 
 # define CHECK(expression) \
 	test_check((expression), #expression, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index e947657..c247c86 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -33,5 +33,6 @@ int	main(void)
 	test_memory_fill();
 	test_memory_copy();
 	test_memory_move();
+	test_memory_scan();
 	return (test_finish());
 }
diff --git a/tests/test_memory_scan.c b/tests/test_memory_scan.c
new file mode 100644
index 0000000..cf7a900
--- /dev/null
+++ b/tests/test_memory_scan.c
@@ -0,0 +1,51 @@
+#include "test.h"
+
+#include "libft.h"
+
+#include <stddef.h>
+#include <string.h>
+
+static int	sign_of(int value)
+{
+	if (value < 0)
+		return (-1);
+	if (value > 0)
+		return (1);
+	return (0);
+}
+
+static void	check_memchr(const unsigned char *bytes, size_t length, int byte)
+{
+	CHECK(ft_memchr(bytes, byte, length) == memchr(bytes, byte, length));
+}
+
+static void	check_memcmp(const unsigned char *left,
+		const unsigned char *right, size_t length)
+{
+	CHECK(sign_of(ft_memcmp(left, right, length))
+		== sign_of(memcmp(left, right, length)));
+}
+
+void	test_memory_scan(void)
+{
+	unsigned char	left[] = {0, 1, 2, 0, 127, 128, 254, 255, 2, 1};
+	unsigned char	right[] = {0, 1, 2, 0, 127, 129, 254, 255, 2, 1};
+	size_t		length;
+
+	CHECK(ft_memchr(NULL, 0, 0) == NULL);
+	CHECK(ft_memcmp(NULL, NULL, 0) == 0);
+	length = 0;
+	while (length <= sizeof(left))
+	{
+		check_memchr(left, length, 0);
+		check_memchr(left, length, 2);
+		check_memchr(left, length, 255);
+		check_memchr(left, length, 256);
+		check_memchr(left, length, -1);
+		check_memchr(left, length, 42);
+		check_memcmp(left, left, length);
+		check_memcmp(left, right, length);
+		check_memcmp(right, left, length);
+		length++;
+	}
+}
