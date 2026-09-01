## `refactor(reader): legacy API를 context reader에 연결`

diff --git a/get_next_line.c b/get_next_line.c
index ec1f030..01a53c6 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -193,30 +193,13 @@ static char	*extract_line(t_blr_reader *reader, size_t line_end)
 	line = malloc(length + 1);
 	if (line == NULL)
 	{
-		discard_legacy_reader(reader);
+		reader->scan = reader->begin;
 		return (NULL);
 	}
 	copy_bytes(line, reader->bytes + reader->begin, length);
 	line[length] = '\0';
 	reader->begin = line_end;
 	reader->scan = reader->begin;
-	if (reader->begin == reader->end)
-		discard_legacy_reader(reader);
-	return (line);
-}
-
-static char	*release_final_line(t_blr_reader *reader)
-{
-	char	*line;
-	size_t	length;
-
-	length = unread_length(reader);
-	if (reader->begin > 0)
-		copy_bytes(reader->bytes, reader->bytes + reader->begin, length);
-	reader->bytes[length] = '\0';
-	line = reader->bytes;
-	reader->bytes = NULL;
-	discard_legacy_reader(reader);
 	return (line);
 }
 
@@ -225,7 +208,6 @@ t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
 	char		probe;
 	ssize_t	read_size;
 	size_t	line_end;
-	size_t	length;
 
 	if (line != NULL)
 		*line = NULL;
@@ -234,96 +216,66 @@ t_blr_result	blr_reader_next(t_blr_reader *reader, char **line)
 	if (read(reader->fd, &probe, 0) < 0)
 		return (BLR_ERROR);
 	line_end = find_line_end(reader);
-	if (line_end == 0 && reader->reached_eof)
+	if (line_end != 0)
+	{
+		*line = extract_line(reader, line_end);
+		if (*line == NULL)
+			return (BLR_ERROR);
+		return (BLR_LINE);
+	}
+	if (reader->reached_eof)
 	{
 		if (unread_length(reader) == 0)
 			return (BLR_EOF);
-		line_end = reader->end;
+		*line = extract_line(reader, reader->end);
+		if (*line == NULL)
+			return (BLR_ERROR);
+		return (BLR_LINE);
 	}
-	while (line_end == 0 && !reader->reached_eof)
+	while (1)
 	{
 		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
 			return (BLR_ERROR);
 		read_size = read(reader->fd, reader->bytes + reader->end,
 				(size_t)BUFFER_SIZE);
-		if (read_size < 0)
-			return (BLR_ERROR);
-		if (read_size == 0)
-		{
-			reader->reached_eof = 1;
-			if (unread_length(reader) == 0)
-				return (BLR_EOF);
-			line_end = reader->end;
-		}
-		else
+		if (read_size <= 0)
+			break ;
+		reader->end += (size_t)read_size;
+		reader->bytes[reader->end] = '\0';
+		line_end = find_line_end(reader);
+		if (line_end != 0)
 		{
-			reader->end += (size_t)read_size;
-			reader->bytes[reader->end] = '\0';
-			line_end = find_line_end(reader);
+			*line = extract_line(reader, line_end);
+			if (*line == NULL)
+				return (BLR_ERROR);
+			return (BLR_LINE);
 		}
 	}
-	length = line_end - reader->begin;
-	*line = malloc(length + 1);
+	if (read_size < 0)
+		return (BLR_ERROR);
+	reader->reached_eof = 1;
+	if (unread_length(reader) == 0)
+		return (BLR_EOF);
+	*line = extract_line(reader, reader->end);
 	if (*line == NULL)
-	{
-		reader->scan = reader->begin;
 		return (BLR_ERROR);
-	}
-	copy_bytes(*line, reader->bytes + reader->begin, length);
-	(*line)[length] = '\0';
-	reader->begin = line_end;
-	reader->scan = reader->begin;
 	return (BLR_LINE);
 }
 
 char	*get_next_line(int fd)
 {
-	char		probe;
-	ssize_t	read_size;
-	size_t	line_end;
+	char			*line;
 	t_blr_reader	*reader;
+	t_blr_result	result;
 
-	if (fd < 0 || read(fd, &probe, 0) < 0)
-	{
-		reader = find_reader(fd);
-		if (reader != NULL)
-			discard_legacy_reader(reader);
-		return (NULL);
-	}
 	reader = find_reader(fd);
 	if (reader == NULL)
 		reader = create_legacy_reader(fd);
 	if (reader == NULL)
 		return (NULL);
-	line_end = find_line_end(reader);
-	if (line_end != 0)
-		return (extract_line(reader, line_end));
-	while (1)
-	{
-		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
-		{
-			discard_legacy_reader(reader);
-			return (NULL);
-		}
-		read_size = read(fd, reader->bytes + reader->end,
-				(size_t)BUFFER_SIZE);
-		if (read_size <= 0)
-			break ;
-		reader->end += (size_t)read_size;
-		reader->bytes[reader->end] = '\0';
-		line_end = find_line_end(reader);
-		if (line_end != 0)
-			return (extract_line(reader, line_end));
-	}
-	if (read_size < 0)
-	{
-		discard_legacy_reader(reader);
-		return (NULL);
-	}
-	if (unread_length(reader) == 0)
-	{
-		discard_legacy_reader(reader);
-		return (NULL);
-	}
-	return (release_final_line(reader));
+	result = blr_reader_next(reader, &line);
+	if (result == BLR_LINE)
+		return (line);
+	discard_legacy_reader(reader);
+	return (NULL);
 }
diff --git a/get_next_line.h b/get_next_line.h
index 5ba9023..e92ce17 100644
--- a/get_next_line.h
+++ b/get_next_line.h
@@ -18,6 +18,19 @@ typedef enum e_blr_result
 	BLR_LINE = 1
 }	t_blr_result;
 
+/*
+ * 컨텍스트는 fd를 빌릴 뿐 닫지 않습니다. 같은 open file description을
+ * 공유하는 dup 계열 fd에는 컨텍스트를 하나만 두어야 하며, 외부에서
+ * 오프셋을 바꾼 뒤에는 blr_reader_reset을 호출해야 합니다. 서로 독립된
+ * 컨텍스트는 다른 스레드에서 사용할 수 있지만 한 컨텍스트를 동시에
+ * 호출하는 동작은 지원하지 않습니다. reset은 누적 상태만 버리고 fd나
+ * 현재 오프셋은 바꾸지 않습니다. 컨텍스트가 살아 있는 동안 fd를 닫아
+ * 같은 번호로 다시 열었다면 기존 컨텍스트를 재사용하지 않아야 합니다.
+ *
+ * BLR_LINE이면 *line은 호출자가 free해야 합니다. 나머지 결과에서는
+ * *line을 NULL로 설정합니다. BLR_ERROR 뒤에도 reset하거나 destroy할 수
+ * 있습니다.
+ */
 t_blr_reader	*blr_reader_create(int fd);
 t_blr_result	blr_reader_next(t_blr_reader *reader, char **line);
 void			blr_reader_reset(t_blr_reader *reader);


## `test(context): 결과 상태와 컨텍스트 수명 검증`

diff --git a/Makefile b/Makefile
index e546851..ad3b63c 100644
--- a/Makefile
+++ b/Makefile
@@ -18,7 +18,8 @@ OBJ := $(SRC:%.c=$(OBJ_DIR)/%.o)
 DEP := $(OBJ:.o=.d)
 
 TEST_BIN := tests/bin/test_reader_$(BUFFER_SIZE)
-TEST_SRC := tests/test_main.c tests/test_reader.c tests/test_boundaries.c
+TEST_SRC := tests/test_main.c tests/test_reader.c tests/test_boundaries.c \
+	tests/test_context.c
 MATRIX_SIZES := 1 2 42 1024
 
 FAULT_OBJ_DIR := build/fault/$(BUFFER_SIZE)
diff --git a/tests/test.h b/tests/test.h
index 7b442e2..e55ae28 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -8,6 +8,7 @@ void	test_check(int condition, const char *expression, const char *file,
 int		test_finish(void);
 void	test_reader(void);
 void	test_boundaries(void);
+void	test_context(void);
 
 # define CHECK(condition) \
 	test_check((condition), #condition, __FILE__, __LINE__)
diff --git a/tests/test_context.c b/tests/test_context.c
new file mode 100644
index 0000000..d45c3d6
--- /dev/null
+++ b/tests/test_context.c
@@ -0,0 +1,248 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "test.h"
+
+#include "get_next_line.h"
+
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+static int	write_all(int fd, const char *bytes, size_t length)
+{
+	ssize_t	written;
+
+	while (length > 0)
+	{
+		written = write(fd, bytes, length);
+		if (written <= 0)
+			return (0);
+		bytes += written;
+		length -= (size_t)written;
+	}
+	return (1);
+}
+
+static int	file_from(const char *bytes)
+{
+	char	path[] = "/tmp/buffered-line-context-XXXXXX";
+	int		fd;
+
+	fd = mkstemp(path);
+	if (fd < 0)
+		return (-1);
+	unlink(path);
+	if (!write_all(fd, bytes, strlen(bytes)) || lseek(fd, 0, SEEK_SET) < 0)
+	{
+		close(fd);
+		return (-1);
+	}
+	return (fd);
+}
+
+static void	check_line(t_blr_reader *reader, const char *expected)
+{
+	char			*line;
+	t_blr_result	result;
+
+	line = (char *)reader;
+	result = blr_reader_next(reader, &line);
+	CHECK(result == BLR_LINE);
+	CHECK(line != NULL);
+	if (line != NULL)
+	{
+		CHECK(strcmp(line, expected) == 0);
+		free(line);
+	}
+}
+
+static void	test_result_states(void)
+{
+	char			*line;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fd = file_from("first\nlast");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	reader = blr_reader_create(fd);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "first\n");
+		check_line(reader, "last");
+		line = (char *)reader;
+		CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+		CHECK(line == NULL);
+		CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+		CHECK(line == NULL);
+		blr_reader_destroy(reader);
+	}
+	close(fd);
+}
+
+static void	test_empty_and_error_are_distinct(void)
+{
+	char			*line;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fd = file_from("");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	reader = blr_reader_create(fd);
+	CHECK(reader != NULL);
+	if (reader == NULL)
+	{
+		close(fd);
+		return ;
+	}
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
+	blr_reader_reset(reader);
+	close(fd);
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	blr_reader_destroy(reader);
+}
+
+static void	test_reset_after_external_seek(void)
+{
+	int				fd;
+	t_blr_reader	*reader;
+
+	fd = file_from("repeat\nignored");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	reader = blr_reader_create(fd);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "repeat\n");
+		CHECK(lseek(fd, 0, SEEK_SET) == 0);
+		blr_reader_reset(reader);
+		check_line(reader, "repeat\n");
+		blr_reader_destroy(reader);
+	}
+	close(fd);
+}
+
+static void	test_destroy_cancels_without_closing(void)
+{
+	int				fd;
+	t_blr_reader	*reader;
+
+	fd = file_from("first\nprefetched");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	reader = blr_reader_create(fd);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "first\n");
+		blr_reader_destroy(reader);
+		CHECK(fcntl(fd, F_GETFD) >= 0);
+		CHECK(lseek(fd, 0, SEEK_SET) == 0);
+		reader = blr_reader_create(fd);
+		CHECK(reader != NULL);
+		if (reader != NULL)
+		{
+			check_line(reader, "first\n");
+			blr_reader_destroy(reader);
+		}
+	}
+	close(fd);
+}
+
+static void	test_reused_descriptor_gets_new_context(void)
+{
+	int				first;
+	int				replacement;
+	t_blr_reader	*reader;
+
+	first = file_from("old\nbuffered");
+	replacement = file_from("new\ncontent");
+	CHECK(first >= 0 && replacement >= 0);
+	if (first < 0 || replacement < 0)
+	{
+		if (first >= 0)
+			close(first);
+		if (replacement >= 0)
+			close(replacement);
+		return ;
+	}
+	reader = blr_reader_create(first);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "old\n");
+		blr_reader_destroy(reader);
+	}
+	close(first);
+	CHECK(dup2(replacement, first) == first);
+	close(replacement);
+	reader = blr_reader_create(first);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "new\n");
+		blr_reader_destroy(reader);
+	}
+	close(first);
+}
+
+static void	test_single_context_on_dup_alias(void)
+{
+	int				alias;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fd = file_from("alias one\nalias two");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	alias = dup(fd);
+	CHECK(alias >= 0);
+	close(fd);
+	if (alias < 0)
+		return ;
+	reader = blr_reader_create(alias);
+	CHECK(reader != NULL);
+	if (reader != NULL)
+	{
+		check_line(reader, "alias one\n");
+		check_line(reader, "alias two");
+		blr_reader_destroy(reader);
+	}
+	close(alias);
+}
+
+static void	test_invalid_arguments(void)
+{
+	char	*line;
+
+	line = (char *)1;
+	CHECK(blr_reader_create(-1) == NULL);
+	CHECK(blr_reader_next(NULL, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	CHECK(blr_reader_next(NULL, NULL) == BLR_ERROR);
+	blr_reader_reset(NULL);
+	blr_reader_destroy(NULL);
+}
+
+void	test_context(void)
+{
+	test_result_states();
+	test_empty_and_error_are_distinct();
+	test_reset_after_external_seek();
+	test_destroy_cancels_without_closing();
+	test_reused_descriptor_gets_new_context();
+	test_single_context_on_dup_alias();
+	test_invalid_arguments();
+}
diff --git a/tests/test_main.c b/tests/test_main.c
index 42b1a3a..635c4cf 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -32,5 +32,6 @@ int	main(void)
 {
 	test_reader();
 	test_boundaries();
+	test_context();
 	return (test_finish());
 }


## `test(failure): 컨텍스트의 line 할당 재시도 검증`

diff --git a/tests/failure/test_failure.c b/tests/failure/test_failure.c
index 2fabe36..b440d68 100644
--- a/tests/failure/test_failure.c
+++ b/tests/failure/test_failure.c
@@ -211,6 +211,99 @@ static void	test_free_guards(void)
 	check_clean_runtime();
 }
 
+static void	test_context_retries_line_allocation(void)
+{
+	char			*line;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fault_runtime_reset();
+	fd = reader_for("\n");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	reader = blr_reader_create(fd);
+	CHECK(reader != NULL);
+	if (reader == NULL)
+	{
+		close(fd);
+		return ;
+	}
+	fault_allocation_fail_at(3);
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	CHECK(fault_allocation_failed());
+	fault_allocation_fail_at(0);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "\n") == 0);
+	test_free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
+	blr_reader_destroy(reader);
+	close(fd);
+	check_clean_runtime();
+}
+
+static size_t	consume_context_tail(size_t fail_at, int *failed)
+{
+	char			*line;
+	int				fd;
+	int				seen;
+	size_t			attempts;
+	t_blr_reader	*reader;
+	t_blr_result	result;
+
+	fault_runtime_reset();
+	fault_allocation_fail_at(fail_at);
+	fd = reader_for("tail");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return (0);
+	reader = blr_reader_create(fd);
+	seen = 0;
+	result = BLR_ERROR;
+	while (reader != NULL && result != BLR_EOF)
+	{
+		result = blr_reader_next(reader, &line);
+		if (result == BLR_ERROR && fault_allocation_failed())
+			fault_allocation_fail_at(0);
+		else if (result == BLR_LINE)
+		{
+			CHECK(!seen);
+			CHECK(line != NULL && strcmp(line, "tail") == 0);
+			seen = 1;
+			test_free(line);
+		}
+	}
+	CHECK(reader != NULL);
+	CHECK(seen);
+	attempts = fault_allocation_attempts();
+	*failed = fault_allocation_failed();
+	blr_reader_destroy(reader);
+	close(fd);
+	check_clean_runtime();
+	return (attempts);
+}
+
+static void	test_context_retries_final_line_allocation(void)
+{
+	int		failed;
+	size_t	baseline;
+	size_t	index;
+
+	baseline = consume_context_tail(0, &failed);
+	CHECK(!failed);
+	CHECK(baseline > 0);
+	index = 2;
+	while (index <= baseline)
+	{
+		consume_context_tail(index, &failed);
+		CHECK(failed);
+		index++;
+	}
+}
+
 int	main(void)
 {
 	test_allocation_failures();
@@ -218,6 +311,8 @@ int	main(void)
 	test_first_read_error();
 	test_middle_read_error();
 	test_one_fd_failure_preserves_another();
+	test_context_retries_line_allocation();
+	test_context_retries_final_line_allocation();
 	test_free_guards();
 	if (g_failures != 0)
 	{


## `test(thread): 독립 컨텍스트의 병렬 사용 검증`

diff --git a/Makefile b/Makefile
index 730130a..f7acf2d 100644
--- a/Makefile
+++ b/Makefile
@@ -3,6 +3,7 @@ NAME := libbuffered_line_reader.a
 CC := cc
 override CFLAGS := -Wall -Wextra -Werror -Wpedantic -std=c99 \
 	-fno-builtin
+THREAD_FLAGS := -pthread
 BUFFER_SIZE ?= 42
 override CPPFLAGS := -I. -DBUFFER_SIZE=$(BUFFER_SIZE)
 DEPFLAGS := -MMD -MP
@@ -19,7 +20,7 @@ DEP := $(OBJ:.o=.d)
 
 TEST_BIN := tests/bin/test_reader_$(BUFFER_SIZE)
 TEST_SRC := tests/test_main.c tests/test_reader.c tests/test_boundaries.c \
-	tests/test_context.c tests/test_nonblocking.c
+	tests/test_context.c tests/test_nonblocking.c tests/test_threads.c
 MATRIX_SIZES := 1 2 42 1024
 
 FAULT_OBJ_DIR := build/fault/$(BUFFER_SIZE)
@@ -52,7 +53,7 @@ $(OBJ_DIR)/%.o: %.c get_next_line.h
 
 $(TEST_BIN): $(OBJ) $(TEST_SRC) tests/test.h
 	@$(MKDIR) $(dir $@)
-	$(CC) $(CPPFLAGS) $(CFLAGS) $(TEST_SRC) $(OBJ) -o $@
+	$(CC) $(CPPFLAGS) $(CFLAGS) $(THREAD_FLAGS) $(TEST_SRC) $(OBJ) -o $@
 
 test-run: $(TEST_BIN)
 	./$(TEST_BIN)
@@ -91,7 +92,8 @@ failure-test:
 
 $(UBSAN_BIN): $(SRC) $(TEST_SRC) get_next_line.h tests/test.h
 	@$(MKDIR) $(dir $@)
-	$(CC) $(CPPFLAGS) $(CFLAGS) $(UBSAN_FLAGS) $(SRC) $(TEST_SRC) -o $@
+	$(CC) $(CPPFLAGS) $(CFLAGS) $(THREAD_FLAGS) $(UBSAN_FLAGS) $(SRC) \
+		$(TEST_SRC) -o $@
 
 ubsan-run: $(UBSAN_BIN)
 	./$(UBSAN_BIN)
diff --git a/tests/test.h b/tests/test.h
index 68a66e3..a7d27af 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -10,6 +10,7 @@ void	test_reader(void);
 void	test_boundaries(void);
 void	test_context(void);
 void	test_nonblocking(void);
+void	test_threads(void);
 
 # define CHECK(condition) \
 	test_check((condition), #condition, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index 26c5833..2a31426 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -34,5 +34,6 @@ int	main(void)
 	test_boundaries();
 	test_context();
 	test_nonblocking();
+	test_threads();
 	return (test_finish());
 }
diff --git a/tests/test_threads.c b/tests/test_threads.c
new file mode 100644
index 0000000..be5c338
--- /dev/null
+++ b/tests/test_threads.c
@@ -0,0 +1,134 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "test.h"
+
+#include "get_next_line.h"
+
+#include <pthread.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+#define THREAD_COUNT 4
+
+typedef struct s_thread_case
+{
+	int			fd;
+	const char	*first;
+	const char	*second;
+	int			passed;
+}	t_thread_case;
+
+static int	write_all(int fd, const char *bytes, size_t length)
+{
+	ssize_t	written;
+
+	while (length > 0)
+	{
+		written = write(fd, bytes, length);
+		if (written <= 0)
+			return (0);
+		bytes += written;
+		length -= (size_t)written;
+	}
+	return (1);
+}
+
+static int	reader_for(const char *bytes)
+{
+	int	fds[2];
+
+	if (pipe(fds) != 0)
+		return (-1);
+	if (!write_all(fds[1], bytes, strlen(bytes)))
+	{
+		close(fds[0]);
+		close(fds[1]);
+		return (-1);
+	}
+	close(fds[1]);
+	return (fds[0]);
+}
+
+static int	matches_next(t_blr_reader *reader, const char *expected)
+{
+	char	*line;
+	int		matches;
+
+	if (blr_reader_next(reader, &line) != BLR_LINE || line == NULL)
+		return (0);
+	matches = strcmp(line, expected) == 0;
+	free(line);
+	return (matches);
+}
+
+static void	*consume_context(void *argument)
+{
+	char			*line;
+	t_thread_case	*test_case;
+	t_blr_reader	*reader;
+
+	test_case = argument;
+	reader = blr_reader_create(test_case->fd);
+	if (reader == NULL)
+		return (NULL);
+	test_case->passed = matches_next(reader, test_case->first)
+		&& matches_next(reader, test_case->second)
+		&& blr_reader_next(reader, &line) == BLR_EOF
+		&& line == NULL;
+	blr_reader_destroy(reader);
+	return (NULL);
+}
+
+void	test_threads(void)
+{
+	const char		*inputs[THREAD_COUNT];
+	const char		*first[THREAD_COUNT];
+	const char		*second[THREAD_COUNT];
+	int				created[THREAD_COUNT];
+	size_t			index;
+	pthread_t		threads[THREAD_COUNT];
+	t_thread_case	cases[THREAD_COUNT];
+
+	inputs[0] = "alpha one\nalpha two";
+	inputs[1] = "beta one\nbeta two";
+	inputs[2] = "gamma one\ngamma two";
+	inputs[3] = "delta one\ndelta two";
+	first[0] = "alpha one\n";
+	first[1] = "beta one\n";
+	first[2] = "gamma one\n";
+	first[3] = "delta one\n";
+	second[0] = "alpha two";
+	second[1] = "beta two";
+	second[2] = "gamma two";
+	second[3] = "delta two";
+	index = 0;
+	while (index < THREAD_COUNT)
+	{
+		cases[index].fd = reader_for(inputs[index]);
+		cases[index].first = first[index];
+		cases[index].second = second[index];
+		cases[index].passed = 0;
+		created[index] = 0;
+		CHECK(cases[index].fd >= 0);
+		if (cases[index].fd >= 0)
+		{
+			created[index] = pthread_create(&threads[index], NULL,
+					consume_context, &cases[index]) == 0;
+			CHECK(created[index]);
+		}
+		index++;
+	}
+	index = 0;
+	while (index < THREAD_COUNT)
+	{
+		if (created[index])
+		{
+			CHECK(pthread_join(threads[index], NULL) == 0);
+			CHECK(cases[index].passed);
+		}
+		if (cases[index].fd >= 0)
+			close(cases[index].fd);
+		index++;
+	}
+}
