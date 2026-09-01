# 줄 추출 상태 머신과 결과 의미

## `feat(api): line reader 공개 계약 정의`

diff --git a/get_next_line.h b/get_next_line.h
new file mode 100644
index 0000000..e48c67f
--- /dev/null
+++ b/get_next_line.h
@@ -0,0 +1,14 @@
+#ifndef GET_NEXT_LINE_H
+# define GET_NEXT_LINE_H
+
+# ifndef BUFFER_SIZE
+#  define BUFFER_SIZE 42
+# endif
+
+# if BUFFER_SIZE <= 0
+#  error "BUFFER_SIZE must be greater than zero"
+# endif
+
+char	*get_next_line(int fd);
+
+#endif


## `feat(reader): 파일 끝의 마지막 줄 반환`

diff --git a/get_next_line.c b/get_next_line.c
new file mode 100644
index 0000000..7a9c123
--- /dev/null
+++ b/get_next_line.c
@@ -0,0 +1,130 @@
+#include "get_next_line.h"
+
+#include <stdlib.h>
+#include <unistd.h>
+
+typedef struct s_reader
+{
+	int			fd;
+	char		*bytes;
+	size_t		length;
+	size_t		capacity;
+}t_reader;
+
+static t_reader	g_reader = {-1, NULL, 0, 0};
+
+static void	copy_bytes(char *destination, const char *source, size_t length)
+{
+	size_t	index;
+
+	index = 0;
+	while (index < length)
+	{
+		destination[index] = source[index];
+		index++;
+	}
+}
+
+static void	reset_reader(void)
+{
+	free(g_reader.bytes);
+	g_reader.fd = -1;
+	g_reader.bytes = NULL;
+	g_reader.length = 0;
+	g_reader.capacity = 0;
+}
+
+static int	reserve_bytes(size_t required)
+{
+	char	*allocation;
+	size_t	capacity;
+
+	if (required <= g_reader.capacity)
+		return (1);
+	capacity = g_reader.capacity;
+	if (capacity == 0)
+		capacity = 1;
+	while (capacity < required)
+	{
+		if (capacity > (size_t)-1 / 2)
+			capacity = required;
+		else
+			capacity *= 2;
+		if (capacity < required && capacity == (size_t)-1)
+			return (0);
+	}
+	allocation = malloc(capacity);
+	if (allocation == NULL)
+		return (0);
+	copy_bytes(allocation, g_reader.bytes, g_reader.length);
+	free(g_reader.bytes);
+	g_reader.bytes = allocation;
+	g_reader.capacity = capacity;
+	return (1);
+}
+
+static int	append_bytes(const char *bytes, size_t length)
+{
+	size_t	required;
+
+	if (length > (size_t)-1 - g_reader.length - 1)
+		return (0);
+	required = g_reader.length + length + 1;
+	if (!reserve_bytes(required))
+		return (0);
+	copy_bytes(g_reader.bytes + g_reader.length, bytes, length);
+	g_reader.length += length;
+	g_reader.bytes[g_reader.length] = '\0';
+	return (1);
+}
+
+static char	*release_final_line(void)
+{
+	char	*line;
+
+	line = g_reader.bytes;
+	g_reader.fd = -1;
+	g_reader.bytes = NULL;
+	g_reader.length = 0;
+	g_reader.capacity = 0;
+	return (line);
+}
+
+char	*get_next_line(int fd)
+{
+	char	buffer[BUFFER_SIZE];
+	ssize_t	read_size;
+
+	if (fd < 0 || read(fd, buffer, 0) < 0)
+	{
+		if (fd == g_reader.fd)
+			reset_reader();
+		return (NULL);
+	}
+	if (g_reader.fd != fd)
+	{
+		reset_reader();
+		g_reader.fd = fd;
+	}
+	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
+	while (read_size > 0)
+	{
+		if (!append_bytes(buffer, (size_t)read_size))
+		{
+			reset_reader();
+			return (NULL);
+		}
+		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
+	}
+	if (read_size < 0)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	if (g_reader.length == 0)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	return (release_final_line());
+}


## `feat(reader): 줄을 분리하고 남은 입력 보존`

diff --git a/get_next_line.c b/get_next_line.c
index f951879..50faef8 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -8,11 +8,12 @@ typedef struct s_reader
 	int			fd;
 	char		*bytes;
 	size_t		begin;
+	size_t		scan;
 	size_t		end;
 	size_t		capacity;
 }t_reader;
 
-static t_reader	g_reader = {-1, NULL, 0, 0, 0};
+static t_reader	g_reader = {-1, NULL, 0, 0, 0, 0};
 
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
@@ -32,6 +33,7 @@ static void	reset_reader(void)
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
 	g_reader.begin = 0;
+	g_reader.scan = 0;
 	g_reader.end = 0;
 	g_reader.capacity = 0;
 }
@@ -47,6 +49,7 @@ static void	compact_bytes(void)
 
 	length = unread_length();
 	copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
+	g_reader.scan -= g_reader.begin;
 	g_reader.begin = 0;
 	g_reader.end = length;
 	g_reader.bytes[g_reader.end] = '\0';
@@ -92,6 +95,7 @@ static int	reserve_bytes(size_t appended)
 	free(g_reader.bytes);
 	g_reader.bytes = allocation;
 	g_reader.begin = 0;
+	g_reader.scan = length;
 	g_reader.end = length;
 	g_reader.capacity = capacity;
 	g_reader.bytes[g_reader.end] = '\0';
@@ -108,6 +112,48 @@ static int	append_bytes(const char *bytes, size_t length)
 	return (1);
 }
 
+static size_t	find_line_end(void)
+{
+	while (g_reader.scan < g_reader.end)
+	{
+		if (g_reader.bytes[g_reader.scan] == '\n')
+		{
+			g_reader.scan++;
+			return (g_reader.scan);
+		}
+		g_reader.scan++;
+	}
+	return (0);
+}
+
+static char	*extract_line(size_t line_end)
+{
+	char	*line;
+	size_t	length;
+
+	length = line_end - g_reader.begin;
+	line = malloc(length + 1);
+	if (line == NULL)
+	{
+		reset_reader();
+		return (NULL);
+	}
+	copy_bytes(line, g_reader.bytes + g_reader.begin, length);
+	line[length] = '\0';
+	g_reader.begin = line_end;
+	g_reader.scan = g_reader.begin;
+	if (g_reader.begin == g_reader.end)
+	{
+		free(g_reader.bytes);
+		g_reader.bytes = NULL;
+		g_reader.begin = 0;
+		g_reader.scan = 0;
+		g_reader.end = 0;
+		g_reader.capacity = 0;
+	}
+	return (line);
+}
+
 static char	*release_final_line(void)
 {
 	char	*line;
@@ -121,6 +167,7 @@ static char	*release_final_line(void)
 	g_reader.fd = -1;
 	g_reader.bytes = NULL;
 	g_reader.begin = 0;
+	g_reader.scan = 0;
 	g_reader.end = 0;
 	g_reader.capacity = 0;
 	return (line);
@@ -130,6 +177,7 @@ char	*get_next_line(int fd)
 {
 	char	buffer[BUFFER_SIZE];
 	ssize_t	read_size;
+	size_t	line_end;
 
 	if (fd < 0 || read(fd, buffer, 0) < 0)
 	{
@@ -142,6 +190,9 @@ char	*get_next_line(int fd)
 		reset_reader();
 		g_reader.fd = fd;
 	}
+	line_end = find_line_end();
+	if (line_end != 0)
+		return (extract_line(line_end));
 	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	while (read_size > 0)
 	{
@@ -150,6 +201,9 @@ char	*get_next_line(int fd)
 			reset_reader();
 			return (NULL);
 		}
+		line_end = find_line_end();
+		if (line_end != 0)
+			return (extract_line(line_end));
 		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	}
 	if (read_size < 0)


## `test(reader): 개행과 남은 입력 처리 검증`

diff --git a/tests/test_reader.c b/tests/test_reader.c
index 71b4a0d..01882dc 100644
--- a/tests/test_reader.c
+++ b/tests/test_reader.c
@@ -114,10 +114,72 @@ static void	test_invalid_descriptors(void)
 	CHECK(get_next_line(fds[0]) == NULL);
 }
 
+static void	test_newline_and_remainder(void)
+{
+	const char	*expected[3];
+	char			*line;
+	int				fd;
+	size_t			index;
+
+	expected[0] = "one\n";
+	expected[1] = "second\n";
+	expected[2] = "third";
+	fd = reader_for("one\nsecond\nthird", 16);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	index = 0;
+	while (index < 3)
+	{
+		line = get_next_line(fd);
+		CHECK(line != NULL);
+		if (line != NULL)
+		{
+			CHECK(strcmp(line, expected[index]) == 0);
+			free(line);
+		}
+		index++;
+	}
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
+static void	test_empty_lines(void)
+{
+	const char	*expected[3];
+	char			*line;
+	int				fd;
+	size_t			index;
+
+	expected[0] = "\n";
+	expected[1] = "\n";
+	expected[2] = "x\n";
+	fd = reader_for("\n\nx\n", 4);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	index = 0;
+	while (index < 3)
+	{
+		line = get_next_line(fd);
+		CHECK(line != NULL);
+		if (line != NULL)
+		{
+			CHECK(strcmp(line, expected[index]) == 0);
+			free(line);
+		}
+		index++;
+	}
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
 void	test_reader(void)
 {
 	test_empty_input();
 	test_final_line();
 	test_multiple_chunks();
 	test_invalid_descriptors();
+	test_newline_and_remainder();
+	test_empty_lines();
 }


## `feat(reader): 명시적 결과 상태 API 추가`

diff --git a/get_next_line.c b/get_next_line.c
index 7c0c3ef..ec1f030 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -11,6 +11,7 @@ struct s_blr_reader
 	size_t		scan;
 	size_t		end;
 	size_t		capacity;
+	int			reached_eof;
 	t_blr_reader	*next;
 };
 
@@ -54,6 +55,7 @@ t_blr_reader	*blr_reader_create(int fd)
 	reader->scan = 0;
 	reader->end = 0;
 	reader->capacity = 0;
+	reader->reached_eof = 0;
 	reader->next = NULL;
 	return (reader);
 }
@@ -68,6 +70,7 @@ void	blr_reader_reset(t_blr_reader *reader)
 	reader->scan = 0;
 	reader->end = 0;
 	reader->capacity = 0;
+	reader->reached_eof = 0;
 }
 
 void	blr_reader_destroy(t_blr_reader *reader)
@@ -217,6 +220,62 @@ static char	*release_final_line(t_blr_reader *reader)
 	return (line);
 }
 
+t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
+{
+	char		probe;
+	ssize_t	read_size;
+	size_t	line_end;
+	size_t	length;
+
+	if (line != NULL)
+		*line = NULL;
+	if (reader == NULL || line == NULL)
+		return (BLR_ERROR);
+	if (read(reader->fd, &probe, 0) < 0)
+		return (BLR_ERROR);
+	line_end = find_line_end(reader);
+	if (line_end == 0 && reader->reached_eof)
+	{
+		if (unread_length(reader) == 0)
+			return (BLR_EOF);
+		line_end = reader->end;
+	}
+	while (line_end == 0 && !reader->reached_eof)
+	{
+		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
+			return (BLR_ERROR);
+		read_size = read(reader->fd, reader->bytes + reader->end,
+				(size_t)BUFFER_SIZE);
+		if (read_size < 0)
+			return (BLR_ERROR);
+		if (read_size == 0)
+		{
+			reader->reached_eof = 1;
+			if (unread_length(reader) == 0)
+				return (BLR_EOF);
+			line_end = reader->end;
+		}
+		else
+		{
+			reader->end += (size_t)read_size;
+			reader->bytes[reader->end] = '\0';
+			line_end = find_line_end(reader);
+		}
+	}
+	length = line_end - reader->begin;
+	*line = malloc(length + 1);
+	if (*line == NULL)
+	{
+		reader->scan = reader->begin;
+		return (BLR_ERROR);
+	}
+	copy_bytes(*line, reader->bytes + reader->begin, length);
+	(*line)[length] = '\0';
+	reader->begin = line_end;
+	reader->scan = reader->begin;
+	return (BLR_LINE);
+}
+
 char	*get_next_line(int fd)
 {
 	char		probe;
diff --git a/get_next_line.h b/get_next_line.h
index c49f1a3..5ba9023 100644
--- a/get_next_line.h
+++ b/get_next_line.h
@@ -11,7 +11,15 @@
 
 typedef struct s_blr_reader	t_blr_reader;
 
+typedef enum e_blr_result
+{
+	BLR_ERROR = -1,
+	BLR_EOF = 0,
+	BLR_LINE = 1
+}	t_blr_result;
+
 t_blr_reader	*blr_reader_create(int fd);
+t_blr_result	blr_reader_next(t_blr_reader *reader, char **line);
 void			blr_reader_reset(t_blr_reader *reader);
 void			blr_reader_destroy(t_blr_reader *reader);
 
diff --git a/tests/manifests/api-symbols.txt b/tests/manifests/api-symbols.txt
index 8890e35..bc7ac5a 100644
--- a/tests/manifests/api-symbols.txt
+++ b/tests/manifests/api-symbols.txt
@@ -1,4 +1,5 @@
 blr_reader_create
 blr_reader_destroy
+blr_reader_next
 blr_reader_reset
 get_next_line


## `fix(reader): 중단된 읽기를 재시도하고 대기 상태를 보존`

diff --git a/get_next_line.c b/get_next_line.c
index 01a53c6..4c2212c 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -1,5 +1,6 @@
 #include "get_next_line.h"
 
+#include <errno.h>
 #include <stdlib.h>
 #include <unistd.h>
 
@@ -17,6 +18,16 @@ struct s_blr_reader
 
 static t_blr_reader	*g_readers;
 
+static ssize_t	read_retrying(int fd, void *buffer, size_t count)
+{
+	ssize_t	read_size;
+
+	read_size = read(fd, buffer, count);
+	while (read_size < 0 && errno == EINTR)
+		read_size = read(fd, buffer, count);
+	return (read_size);
+}
+
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
 	size_t	index;
@@ -44,7 +55,7 @@ t_blr_reader	*blr_reader_create(int fd)
 	char			probe;
 	t_blr_reader	*reader;
 
-	if (fd < 0 || read(fd, &probe, 0) < 0)
+	if (fd < 0 || read_retrying(fd, &probe, 0) < 0)
 		return (NULL);
 	reader = malloc(sizeof(*reader));
 	if (reader == NULL)
@@ -213,7 +224,7 @@ t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
 		*line = NULL;
 	if (reader == NULL || line == NULL)
 		return (BLR_ERROR);
-	if (read(reader->fd, &probe, 0) < 0)
+	if (read_retrying(reader->fd, &probe, 0) < 0)
 		return (BLR_ERROR);
 	line_end = find_line_end(reader);
 	if (line_end != 0)
@@ -236,7 +247,7 @@ t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
 	{
 		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
 			return (BLR_ERROR);
-		read_size = read(reader->fd, reader->bytes + reader->end,
+		read_size = read_retrying(reader->fd, reader->bytes + reader->end,
 				(size_t)BUFFER_SIZE);
 		if (read_size <= 0)
 			break ;
@@ -252,7 +263,11 @@ t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
 		}
 	}
 	if (read_size < 0)
+	{
+		if (errno == EAGAIN || errno == EWOULDBLOCK)
+			return (BLR_AGAIN);
 		return (BLR_ERROR);
+	}
 	reader->reached_eof = 1;
 	if (unread_length(reader) == 0)
 		return (BLR_EOF);
@@ -276,6 +291,7 @@ char	*get_next_line(int fd)
 	result = blr_reader_next(reader, &line);
 	if (result == BLR_LINE)
 		return (line);
-	discard_legacy_reader(reader);
+	if (result != BLR_AGAIN)
+		discard_legacy_reader(reader);
 	return (NULL);
 }
diff --git a/get_next_line.h b/get_next_line.h
index e92ce17..50dce96 100644
--- a/get_next_line.h
+++ b/get_next_line.h
@@ -15,7 +15,8 @@ typedef enum e_blr_result
 {
 	BLR_ERROR = -1,
 	BLR_EOF = 0,
-	BLR_LINE = 1
+	BLR_LINE = 1,
+	BLR_AGAIN = 2
 }	t_blr_result;
 
 /*
@@ -28,8 +29,9 @@ typedef enum e_blr_result
  * 같은 번호로 다시 열었다면 기존 컨텍스트를 재사용하지 않아야 합니다.
  *
  * BLR_LINE이면 *line은 호출자가 free해야 합니다. 나머지 결과에서는
- * *line을 NULL로 설정합니다. BLR_ERROR 뒤에도 reset하거나 destroy할 수
- * 있습니다.
+ * *line을 NULL로 설정합니다. 읽기 오류나 할당 실패가 나도 누적 입력은
+ * 유지되므로 BLR_ERROR 뒤 같은 컨텍스트로 재시도하거나 reset 또는
+ * destroy할 수 있습니다. BLR_AGAIN은 입력이 준비된 뒤 다시 호출합니다.
  */
 t_blr_reader	*blr_reader_create(int fd);
 t_blr_result	blr_reader_next(t_blr_reader *reader, char **line);
diff --git a/tests/check_archive.sh b/tests/check_archive.sh
index c7bc3cc..9230329 100644
--- a/tests/check_archive.sh
+++ b/tests/check_archive.sh
@@ -18,7 +18,9 @@ nm -g "$archive" |
 
 nm -g "$archive" |
 	awk 'NF == 2 && $1 == "U" {print $2}' |
-	sed 's/^_//' |
+	sed -e 's/^___error$/errno_accessor/' \
+		-e 's/^__errno_location$/errno_accessor/' \
+		-e 's/^_//' |
 	LC_ALL=C sort -u >"$temporary_dir/allowed-undefined.txt"
 
 diff -u "$manifest_dir/archive-members.txt" \
diff --git a/tests/manifests/allowed-undefined.txt b/tests/manifests/allowed-undefined.txt
index d042561..b0c65e7 100644
--- a/tests/manifests/allowed-undefined.txt
+++ b/tests/manifests/allowed-undefined.txt
@@ -1,3 +1,4 @@
+errno_accessor
 free
 malloc
 read
