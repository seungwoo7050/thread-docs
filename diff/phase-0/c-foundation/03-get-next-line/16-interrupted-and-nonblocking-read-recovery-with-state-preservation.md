# 중단·비차단 읽기의 재시도와 상태 보존

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


## `test(reader): 비차단 부분 입력 보존 검증`

diff --git a/Makefile b/Makefile
index ad3b63c..730130a 100644
--- a/Makefile
+++ b/Makefile
@@ -19,7 +19,7 @@ DEP := $(OBJ:.o=.d)
 
 TEST_BIN := tests/bin/test_reader_$(BUFFER_SIZE)
 TEST_SRC := tests/test_main.c tests/test_reader.c tests/test_boundaries.c \
-	tests/test_context.c
+	tests/test_context.c tests/test_nonblocking.c
 MATRIX_SIZES := 1 2 42 1024
 
 FAULT_OBJ_DIR := build/fault/$(BUFFER_SIZE)
diff --git a/tests/test.h b/tests/test.h
index e55ae28..68a66e3 100644
--- a/tests/test.h
+++ b/tests/test.h
@@ -9,6 +9,7 @@ int		test_finish(void);
 void	test_reader(void);
 void	test_boundaries(void);
 void	test_context(void);
+void	test_nonblocking(void);
 
 # define CHECK(condition) \
 	test_check((condition), #condition, __FILE__, __LINE__)
diff --git a/tests/test_main.c b/tests/test_main.c
index 635c4cf..26c5833 100644
--- a/tests/test_main.c
+++ b/tests/test_main.c
@@ -33,5 +33,6 @@ int	main(void)
 	test_reader();
 	test_boundaries();
 	test_context();
+	test_nonblocking();
 	return (test_finish());
 }
diff --git a/tests/test_nonblocking.c b/tests/test_nonblocking.c
new file mode 100644
index 0000000..5abd3e5
--- /dev/null
+++ b/tests/test_nonblocking.c
@@ -0,0 +1,90 @@
+#include "test.h"
+
+#include "get_next_line.h"
+
+#include <fcntl.h>
+#include <stdlib.h>
+#include <string.h>
+#include <unistd.h>
+
+static int	make_nonblocking_pipe(int fds[2])
+{
+	int	flags;
+
+	if (pipe(fds) != 0)
+		return (0);
+	flags = fcntl(fds[0], F_GETFL);
+	if (flags < 0 || fcntl(fds[0], F_SETFL, flags | O_NONBLOCK) < 0)
+	{
+		close(fds[0]);
+		close(fds[1]);
+		return (0);
+	}
+	return (1);
+}
+
+static void	test_context_preserves_partial_nonblocking_input(void)
+{
+	char			*line;
+	int				fds[2];
+	int				opened;
+	t_blr_reader	*reader;
+
+	opened = make_nonblocking_pipe(fds);
+	CHECK(opened);
+	if (!opened)
+		return ;
+	reader = blr_reader_create(fds[0]);
+	CHECK(reader != NULL);
+	if (reader == NULL)
+	{
+		close(fds[0]);
+		close(fds[1]);
+		return ;
+	}
+	CHECK(write(fds[1], "part", 4) == 4);
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	CHECK(write(fds[1], "ial\nnext", 8) == 8);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "partial\n") == 0);
+	free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	close(fds[1]);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "next") == 0);
+	free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
+	blr_reader_destroy(reader);
+	close(fds[0]);
+}
+
+static void	test_compatibility_wrapper_keeps_waiting_state(void)
+{
+	char	*line;
+	int		fds[2];
+	int		opened;
+
+	opened = make_nonblocking_pipe(fds);
+	CHECK(opened);
+	if (!opened)
+		return ;
+	CHECK(write(fds[1], "leg", 3) == 3);
+	CHECK(get_next_line(fds[0]) == NULL);
+	CHECK(write(fds[1], "acy\n", 4) == 4);
+	line = get_next_line(fds[0]);
+	CHECK(line != NULL && strcmp(line, "legacy\n") == 0);
+	free(line);
+	close(fds[1]);
+	CHECK(get_next_line(fds[0]) == NULL);
+	close(fds[0]);
+}
+
+void	test_nonblocking(void)
+{
+	test_context_preserves_partial_nonblocking_input();
+	test_compatibility_wrapper_keeps_waiting_state();
+}


## `test(failure): EINTR·EAGAIN·I/O 오류 순서 검증`

diff --git a/tests/failure/test_failure.c b/tests/failure/test_failure.c
index b440d68..06518c3 100644
--- a/tests/failure/test_failure.c
+++ b/tests/failure/test_failure.c
@@ -1,6 +1,7 @@
 #include "fault_runtime.h"
 #include "get_next_line.h"
 
+#include <errno.h>
 #include <stdio.h>
 #include <string.h>
 #include <unistd.h>
@@ -304,6 +305,77 @@ static void	test_context_retries_final_line_allocation(void)
 	}
 }
 
+static void	test_interrupted_read_and_again_sequence(void)
+{
+	const int		errors[] = {EINTR, 0, EAGAIN};
+	char			*line;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fault_runtime_reset();
+	fd = reader_for("partial\nlast");
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
+	fault_read_limit(3);
+	fault_read_script(fd, errors, sizeof(errors) / sizeof(errors[0]));
+	line = (char *)reader;
+	CHECK(blr_reader_next(reader, &line) == BLR_AGAIN);
+	CHECK(line == NULL);
+	CHECK(fault_read_calls() == 3);
+	CHECK(fault_read_failed());
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "partial\n") == 0);
+	test_free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "last") == 0);
+	test_free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	CHECK(line == NULL);
+	blr_reader_destroy(reader);
+	close(fd);
+	check_clean_runtime();
+}
+
+static void	test_io_error_preserves_partial_input(void)
+{
+	const int		errors[] = {0, EIO};
+	char			*line;
+	int				fd;
+	t_blr_reader	*reader;
+
+	fault_runtime_reset();
+	fd = reader_for("recoverable\n");
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
+	fault_read_limit(3);
+	fault_read_script(fd, errors, sizeof(errors) / sizeof(errors[0]));
+	CHECK(blr_reader_next(reader, &line) == BLR_ERROR);
+	CHECK(line == NULL);
+	CHECK(blr_reader_next(reader, &line) == BLR_LINE);
+	CHECK(line != NULL && strcmp(line, "recoverable\n") == 0);
+	test_free(line);
+	CHECK(blr_reader_next(reader, &line) == BLR_EOF);
+	blr_reader_destroy(reader);
+	close(fd);
+	check_clean_runtime();
+}
+
 int	main(void)
 {
 	test_allocation_failures();
@@ -313,6 +385,8 @@ int	main(void)
 	test_one_fd_failure_preserves_another();
 	test_context_retries_line_allocation();
 	test_context_retries_final_line_allocation();
+	test_interrupted_read_and_again_sequence();
+	test_io_error_preserves_partial_input();
 	test_free_guards();
 	if (g_failures != 0)
 	{
diff --git a/tests/support/fault_runtime.c b/tests/support/fault_runtime.c
index 7617c23..e1c6f7b 100644
--- a/tests/support/fault_runtime.c
+++ b/tests/support/fault_runtime.c
@@ -15,6 +15,7 @@
 #include <unistd.h>
 
 #define MAX_TRACKED_ALLOCATIONS 4096
+#define MAX_READ_SCRIPT 64
 
 typedef struct s_allocation
 {
@@ -35,6 +36,8 @@ static size_t		g_read_calls;
 static size_t		g_read_fail_at;
 static size_t		g_read_limit;
 static int			g_read_failed;
+static int			g_read_script[MAX_READ_SCRIPT];
+static size_t		g_read_script_length;
 
 void	fault_runtime_reset(void)
 {
@@ -50,6 +53,7 @@ void	fault_runtime_reset(void)
 	g_read_fail_at = 0;
 	g_read_limit = 0;
 	g_read_failed = 0;
+	g_read_script_length = 0;
 }
 
 void	fault_allocation_fail_at(size_t attempt)
@@ -88,6 +92,7 @@ void	fault_read_fail_on(int fd, size_t call_index)
 	g_read_calls = 0;
 	g_read_fail_at = call_index;
 	g_read_failed = 0;
+	g_read_script_length = 0;
 }
 
 void	fault_read_limit(size_t limit)
@@ -95,6 +100,25 @@ void	fault_read_limit(size_t limit)
 	g_read_limit = limit;
 }
 
+void	fault_read_script(int fd, const int *errors, size_t length)
+{
+	size_t	index;
+
+	if (length > MAX_READ_SCRIPT)
+		length = MAX_READ_SCRIPT;
+	g_read_fd = fd;
+	g_read_calls = 0;
+	g_read_fail_at = 0;
+	g_read_failed = 0;
+	g_read_script_length = length;
+	index = 0;
+	while (index < length)
+	{
+		g_read_script[index] = errors[index];
+		index++;
+	}
+}
+
 size_t	fault_read_calls(void)
 {
 	return (g_read_calls);
@@ -174,6 +198,13 @@ ssize_t	test_read(int fd, void *buffer, size_t count)
 	if (fd == g_read_fd)
 	{
 		g_read_calls++;
+		if (g_read_calls <= g_read_script_length
+			&& g_read_script[g_read_calls - 1] != 0)
+		{
+			g_read_failed = 1;
+			errno = g_read_script[g_read_calls - 1];
+			return (-1);
+		}
 		if (g_read_fail_at != 0 && g_read_calls == g_read_fail_at)
 		{
 			g_read_failed = 1;
diff --git a/tests/support/fault_runtime.h b/tests/support/fault_runtime.h
index 078aedd..701ca71 100644
--- a/tests/support/fault_runtime.h
+++ b/tests/support/fault_runtime.h
@@ -15,6 +15,7 @@ size_t	fault_double_frees(void);
 
 void	fault_read_fail_on(int fd, size_t call_index);
 void	fault_read_limit(size_t limit);
+void	fault_read_script(int fd, const int *errors, size_t length);
 size_t	fault_read_calls(void);
 int		fault_read_failed(void);
 
