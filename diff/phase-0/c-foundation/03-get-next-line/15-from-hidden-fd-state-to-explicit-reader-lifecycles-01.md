# 숨은 FD 상태에서 명시적 컨텍스트 수명으로

## `refactor(state): reader 상태를 helper 인자로 전달`

diff --git a/get_next_line.c b/get_next_line.c
index 50faef8..311b83a 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -27,55 +27,55 @@ static void	copy_bytes(char *destination, const char *source, size_t length)
 	}
 }
 
-static void	reset_reader(void)
+static void	reset_reader(t_reader *reader)
 {
-	free(g_reader.bytes);
-	g_reader.fd = -1;
-	g_reader.bytes = NULL;
-	g_reader.begin = 0;
-	g_reader.scan = 0;
-	g_reader.end = 0;
-	g_reader.capacity = 0;
+	free(reader->bytes);
+	reader->fd = -1;
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
 }
 
-static size_t	unread_length(void)
+static size_t	unread_length(const t_reader *reader)
 {
-	return (g_reader.end - g_reader.begin);
+	return (reader->end - reader->begin);
 }
 
-static void	compact_bytes(void)
+static void	compact_bytes(t_reader *reader)
 {
 	size_t	length;
 
-	length = unread_length();
-	copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
-	g_reader.scan -= g_reader.begin;
-	g_reader.begin = 0;
-	g_reader.end = length;
-	g_reader.bytes[g_reader.end] = '\0';
+	length = unread_length(reader);
+	copy_bytes(reader->bytes, reader->bytes + reader->begin, length);
+	reader->scan -= reader->begin;
+	reader->begin = 0;
+	reader->end = length;
+	reader->bytes[reader->end] = '\0';
 }
 
-static int	reserve_bytes(size_t appended)
+static int	reserve_bytes(t_reader *reader, size_t appended)
 {
 	char	*allocation;
 	size_t	capacity;
 	size_t	required;
 	size_t	length;
 
-	length = unread_length();
+	length = unread_length(reader);
 	if (appended > (size_t)-1 - length - 1)
 		return (0);
 	required = length + appended + 1;
-	if (g_reader.capacity - g_reader.end >= appended + 1)
+	if (reader->capacity - reader->end >= appended + 1)
 		return (1);
-	if (g_reader.begin > 0 && required <= g_reader.capacity)
+	if (reader->begin > 0 && required <= reader->capacity)
 	{
-		compact_bytes();
+		compact_bytes(reader);
 		return (1);
 	}
-	if (required <= g_reader.capacity)
+	if (required <= reader->capacity)
 		return (1);
-	capacity = g_reader.capacity;
+	capacity = reader->capacity;
 	if (capacity == 0)
 		capacity = 1;
 	while (capacity < required)
@@ -91,85 +91,85 @@ static int	reserve_bytes(size_t appended)
 	if (allocation == NULL)
 		return (0);
 	if (length > 0)
-		copy_bytes(allocation, g_reader.bytes + g_reader.begin, length);
-	free(g_reader.bytes);
-	g_reader.bytes = allocation;
-	g_reader.begin = 0;
-	g_reader.scan = length;
-	g_reader.end = length;
-	g_reader.capacity = capacity;
-	g_reader.bytes[g_reader.end] = '\0';
+		copy_bytes(allocation, reader->bytes + reader->begin, length);
+	free(reader->bytes);
+	reader->bytes = allocation;
+	reader->begin = 0;
+	reader->scan = length;
+	reader->end = length;
+	reader->capacity = capacity;
+	reader->bytes[reader->end] = '\0';
 	return (1);
 }
 
-static int	append_bytes(const char *bytes, size_t length)
+static int	append_bytes(t_reader *reader, const char *bytes, size_t length)
 {
-	if (!reserve_bytes(length))
+	if (!reserve_bytes(reader, length))
 		return (0);
-	copy_bytes(g_reader.bytes + g_reader.end, bytes, length);
-	g_reader.end += length;
-	g_reader.bytes[g_reader.end] = '\0';
+	copy_bytes(reader->bytes + reader->end, bytes, length);
+	reader->end += length;
+	reader->bytes[reader->end] = '\0';
 	return (1);
 }
 
-static size_t	find_line_end(void)
+static size_t	find_line_end(t_reader *reader)
 {
-	while (g_reader.scan < g_reader.end)
+	while (reader->scan < reader->end)
 	{
-		if (g_reader.bytes[g_reader.scan] == '\n')
+		if (reader->bytes[reader->scan] == '\n')
 		{
-			g_reader.scan++;
-			return (g_reader.scan);
+			reader->scan++;
+			return (reader->scan);
 		}
-		g_reader.scan++;
+		reader->scan++;
 	}
 	return (0);
 }
 
-static char	*extract_line(size_t line_end)
+static char	*extract_line(t_reader *reader, size_t line_end)
 {
 	char	*line;
 	size_t	length;
 
-	length = line_end - g_reader.begin;
+	length = line_end - reader->begin;
 	line = malloc(length + 1);
 	if (line == NULL)
 	{
-		reset_reader();
+		reset_reader(reader);
 		return (NULL);
 	}
-	copy_bytes(line, g_reader.bytes + g_reader.begin, length);
+	copy_bytes(line, reader->bytes + reader->begin, length);
 	line[length] = '\0';
-	g_reader.begin = line_end;
-	g_reader.scan = g_reader.begin;
-	if (g_reader.begin == g_reader.end)
+	reader->begin = line_end;
+	reader->scan = reader->begin;
+	if (reader->begin == reader->end)
 	{
-		free(g_reader.bytes);
-		g_reader.bytes = NULL;
-		g_reader.begin = 0;
-		g_reader.scan = 0;
-		g_reader.end = 0;
-		g_reader.capacity = 0;
+		free(reader->bytes);
+		reader->bytes = NULL;
+		reader->begin = 0;
+		reader->scan = 0;
+		reader->end = 0;
+		reader->capacity = 0;
 	}
 	return (line);
 }
 
-static char	*release_final_line(void)
+static char	*release_final_line(t_reader *reader)
 {
 	char	*line;
 	size_t	length;
 
-	length = unread_length();
-	if (g_reader.begin > 0)
-		copy_bytes(g_reader.bytes, g_reader.bytes + g_reader.begin, length);
-	g_reader.bytes[length] = '\0';
-	line = g_reader.bytes;
-	g_reader.fd = -1;
-	g_reader.bytes = NULL;
-	g_reader.begin = 0;
-	g_reader.scan = 0;
-	g_reader.end = 0;
-	g_reader.capacity = 0;
+	length = unread_length(reader);
+	if (reader->begin > 0)
+		copy_bytes(reader->bytes, reader->bytes + reader->begin, length);
+	reader->bytes[length] = '\0';
+	line = reader->bytes;
+	reader->fd = -1;
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
 	return (line);
 }
 
@@ -178,43 +178,45 @@ char	*get_next_line(int fd)
 	char	buffer[BUFFER_SIZE];
 	ssize_t	read_size;
 	size_t	line_end;
+	t_reader	*reader;
 
+	reader = &g_reader;
 	if (fd < 0 || read(fd, buffer, 0) < 0)
 	{
-		if (fd == g_reader.fd)
-			reset_reader();
+		if (fd == reader->fd)
+			reset_reader(reader);
 		return (NULL);
 	}
-	if (g_reader.fd != fd)
+	if (reader->fd != fd)
 	{
-		reset_reader();
-		g_reader.fd = fd;
+		reset_reader(reader);
+		reader->fd = fd;
 	}
-	line_end = find_line_end();
+	line_end = find_line_end(reader);
 	if (line_end != 0)
-		return (extract_line(line_end));
+		return (extract_line(reader, line_end));
 	read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	while (read_size > 0)
 	{
-		if (!append_bytes(buffer, (size_t)read_size))
+		if (!append_bytes(reader, buffer, (size_t)read_size))
 		{
-			reset_reader();
+			reset_reader(reader);
 			return (NULL);
 		}
-		line_end = find_line_end();
+		line_end = find_line_end(reader);
 		if (line_end != 0)
-			return (extract_line(line_end));
+			return (extract_line(reader, line_end));
 		read_size = read(fd, buffer, (size_t)BUFFER_SIZE);
 	}
 	if (read_size < 0)
 	{
-		reset_reader();
+		reset_reader(reader);
 		return (NULL);
 	}
-	if (unread_length() == 0)
+	if (unread_length(reader) == 0)
 	{
-		reset_reader();
+		reset_reader(reader);
 		return (NULL);
 	}
-	return (release_final_line());
+	return (release_final_line(reader));
 }


## `feat(state): 디스크립터별 읽기 상태 분리`

diff --git a/get_next_line.c b/get_next_line.c
index 311b83a..3a379ab 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -11,9 +11,10 @@ typedef struct s_reader
 	size_t		scan;
 	size_t		end;
 	size_t		capacity;
+	struct s_reader	*next;
 }t_reader;
 
-static t_reader	g_reader = {-1, NULL, 0, 0, 0, 0};
+static t_reader	*g_readers;
 
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
@@ -27,15 +28,46 @@ static void	copy_bytes(char *destination, const char *source, size_t length)
 	}
 }
 
-static void	reset_reader(t_reader *reader)
+static t_reader	*find_reader(int fd)
 {
-	free(reader->bytes);
-	reader->fd = -1;
+	t_reader	*reader;
+
+	reader = g_readers;
+	while (reader != NULL && reader->fd != fd)
+		reader = reader->next;
+	return (reader);
+}
+
+static t_reader	*create_reader(int fd)
+{
+	t_reader	*reader;
+
+	reader = malloc(sizeof(*reader));
+	if (reader == NULL)
+		return (NULL);
+	reader->fd = fd;
 	reader->bytes = NULL;
 	reader->begin = 0;
 	reader->scan = 0;
 	reader->end = 0;
 	reader->capacity = 0;
+	reader->next = g_readers;
+	g_readers = reader;
+	return (reader);
+}
+
+static void	discard_reader(t_reader *reader)
+{
+	t_reader	**link;
+
+	link = &g_readers;
+	while (*link != NULL && *link != reader)
+		link = &(*link)->next;
+	if (*link == NULL)
+		return ;
+	*link = reader->next;
+	free(reader->bytes);
+	free(reader);
 }
 
 static size_t	unread_length(const t_reader *reader)
@@ -135,7 +167,7 @@ static char	*extract_line(t_reader *reader, size_t line_end)
 	line = malloc(length + 1);
 	if (line == NULL)
 	{
-		reset_reader(reader);
+		discard_reader(reader);
 		return (NULL);
 	}
 	copy_bytes(line, reader->bytes + reader->begin, length);
@@ -143,14 +175,7 @@ static char	*extract_line(t_reader *reader, size_t line_end)
 	reader->begin = line_end;
 	reader->scan = reader->begin;
 	if (reader->begin == reader->end)
-	{
-		free(reader->bytes);
-		reader->bytes = NULL;
-		reader->begin = 0;
-		reader->scan = 0;
-		reader->end = 0;
-		reader->capacity = 0;
-	}
+		discard_reader(reader);
 	return (line);
 }
 
@@ -164,12 +189,8 @@ static char	*release_final_line(t_reader *reader)
 		copy_bytes(reader->bytes, reader->bytes + reader->begin, length);
 	reader->bytes[length] = '\0';
 	line = reader->bytes;
-	reader->fd = -1;
 	reader->bytes = NULL;
-	reader->begin = 0;
-	reader->scan = 0;
-	reader->end = 0;
-	reader->capacity = 0;
+	discard_reader(reader);
 	return (line);
 }
 
@@ -180,18 +201,18 @@ char	*get_next_line(int fd)
 	size_t	line_end;
 	t_reader	*reader;
 
-	reader = &g_reader;
 	if (fd < 0 || read(fd, buffer, 0) < 0)
 	{
-		if (fd == reader->fd)
-			reset_reader(reader);
+		reader = find_reader(fd);
+		if (reader != NULL)
+			discard_reader(reader);
 		return (NULL);
 	}
-	if (reader->fd != fd)
-	{
-		reset_reader(reader);
-		reader->fd = fd;
-	}
+	reader = find_reader(fd);
+	if (reader == NULL)
+		reader = create_reader(fd);
+	if (reader == NULL)
+		return (NULL);
 	line_end = find_line_end(reader);
 	if (line_end != 0)
 		return (extract_line(reader, line_end));
@@ -200,7 +221,7 @@ char	*get_next_line(int fd)
 	{
 		if (!append_bytes(reader, buffer, (size_t)read_size))
 		{
-			reset_reader(reader);
+			discard_reader(reader);
 			return (NULL);
 		}
 		line_end = find_line_end(reader);
@@ -210,12 +231,12 @@ char	*get_next_line(int fd)
 	}
 	if (read_size < 0)
 	{
-		reset_reader(reader);
+		discard_reader(reader);
 		return (NULL);
 	}
 	if (unread_length(reader) == 0)
 	{
-		reset_reader(reader);
+		discard_reader(reader);
 		return (NULL);
 	}
 	return (release_final_line(reader));


## `test(state): 교차 디스크립터 상태 격리 검증`

diff --git a/tests/test_reader.c b/tests/test_reader.c
index 01882dc..0b3e572 100644
--- a/tests/test_reader.c
+++ b/tests/test_reader.c
@@ -174,6 +174,73 @@ static void	test_empty_lines(void)
 	close(fd);
 }
 
+static void	test_alternating_descriptors(void)
+{
+	const char	*left_expected[2];
+	const char	*right_expected[3];
+	char			*line;
+	int				left;
+	int				right;
+	size_t			index;
+
+	left_expected[0] = "left one\n";
+	left_expected[1] = "left two";
+	right_expected[0] = "right one\n";
+	right_expected[1] = "right two\n";
+	right_expected[2] = "right three";
+	left = reader_for("left one\nleft two", 17);
+	right = reader_for("right one\nright two\nright three", 31);
+	CHECK(left >= 0 && right >= 0);
+	if (left < 0 || right < 0)
+		return ;
+	index = 0;
+	while (index < 3)
+	{
+		if (index < 2)
+		{
+			line = get_next_line(left);
+			CHECK(line != NULL);
+			if (line != NULL)
+			{
+				CHECK(strcmp(line, left_expected[index]) == 0);
+				free(line);
+			}
+		}
+		line = get_next_line(right);
+		CHECK(line != NULL);
+		if (line != NULL)
+		{
+			CHECK(strcmp(line, right_expected[index]) == 0);
+			free(line);
+		}
+		index++;
+	}
+	CHECK(get_next_line(left) == NULL);
+	CHECK(get_next_line(right) == NULL);
+	close(left);
+	close(right);
+}
+
+static void	test_invalid_fd_preserves_other_state(void)
+{
+	char	*line;
+	int		fd;
+
+	fd = reader_for("first\nsecond", 12);
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "first\n") == 0);
+	free(line);
+	CHECK(get_next_line(-1) == NULL);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "second") == 0);
+	free(line);
+	CHECK(get_next_line(fd) == NULL);
+	close(fd);
+}
+
 void	test_reader(void)
 {
 	test_empty_input();
@@ -182,4 +249,6 @@ void	test_reader(void)
 	test_invalid_descriptors();
 	test_newline_and_remainder();
 	test_empty_lines();
+	test_alternating_descriptors();
+	test_invalid_fd_preserves_other_state();
 }


## `test(error): 오류 발생 시 디스크립터 상태 정리 검증`

diff --git a/tests/test_reader.c b/tests/test_reader.c
index 0b3e572..407b018 100644
--- a/tests/test_reader.c
+++ b/tests/test_reader.c
@@ -241,6 +241,47 @@ static void	test_invalid_fd_preserves_other_state(void)
 	close(fd);
 }
 
+static void	test_write_only_descriptor(void)
+{
+	int	fds[2];
+
+	if (pipe(fds) != 0)
+	{
+		CHECK(0);
+		return ;
+	}
+	CHECK(get_next_line(fds[1]) == NULL);
+	CHECK(write(fds[1], "x", 1) == 1);
+	close(fds[0]);
+	close(fds[1]);
+}
+
+static void	test_closed_reader_cleanup_is_local(void)
+{
+	char	*line;
+	int		closed;
+	int		kept;
+
+	closed = reader_for("first\ndiscarded", 15);
+	kept = reader_for("keep\nsurvive", 12);
+	CHECK(closed >= 0 && kept >= 0);
+	if (closed < 0 || kept < 0)
+		return ;
+	line = get_next_line(closed);
+	CHECK(line != NULL && strcmp(line, "first\n") == 0);
+	free(line);
+	line = get_next_line(kept);
+	CHECK(line != NULL && strcmp(line, "keep\n") == 0);
+	free(line);
+	close(closed);
+	CHECK(get_next_line(closed) == NULL);
+	line = get_next_line(kept);
+	CHECK(line != NULL && strcmp(line, "survive") == 0);
+	free(line);
+	CHECK(get_next_line(kept) == NULL);
+	close(kept);
+}
+
 void	test_reader(void)
 {
 	test_empty_input();
@@ -251,4 +292,6 @@ void	test_reader(void)
 	test_empty_lines();
 	test_alternating_descriptors();
 	test_invalid_fd_preserves_other_state();
+	test_write_only_descriptor();
+	test_closed_reader_cleanup_is_local();
 }


