## `test(failure): 메모리 할당과 읽기 실패 처리 검증`

diff --git a/Makefile b/Makefile
index ec95782..10ed922 100644
--- a/Makefile
+++ b/Makefile
@@ -19,8 +19,19 @@ DEP := $(OBJ:.o=.d)
 
 TEST_BIN := tests/bin/test_reader_$(BUFFER_SIZE)
 TEST_SRC := tests/test_main.c tests/test_reader.c
+MATRIX_SIZES := 1 2 42 1024
 
-.PHONY: all bonus clean fclean re test check
+FAULT_OBJ_DIR := build/fault/$(BUFFER_SIZE)
+FAULT_READER_OBJ := $(FAULT_OBJ_DIR)/get_next_line.o
+FAULT_RUNTIME_OBJ := $(FAULT_OBJ_DIR)/fault_runtime.o
+FAULT_TEST_OBJ := $(FAULT_OBJ_DIR)/test_failure.o
+FAULT_OBJ := $(FAULT_READER_OBJ) $(FAULT_RUNTIME_OBJ) $(FAULT_TEST_OBJ)
+FAULT_DEP := $(FAULT_OBJ:.o=.d)
+FAULT_BIN := tests/bin/test_failure_$(BUFFER_SIZE)
+FAULT_CPPFLAGS := $(CPPFLAGS) -Itests/support
+FAULT_DEFINES := -Dmalloc=test_malloc -Dfree=test_free -Dread=test_read
+
+.PHONY: all bonus clean fclean re test failure-run failure-test check
 
 all: $(NAME)
 
@@ -40,11 +51,39 @@ $(TEST_BIN): $(NAME) $(TEST_SRC) tests/test.h
 test: $(TEST_BIN)
 	./$(TEST_BIN)
 
+$(FAULT_READER_OBJ): get_next_line.c get_next_line.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(FAULT_CPPFLAGS) $(FAULT_DEFINES) $(CFLAGS) $(DEPFLAGS) \
+		-c $< -o $@
+
+$(FAULT_RUNTIME_OBJ): tests/support/fault_runtime.c \
+		tests/support/fault_runtime.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(FAULT_CPPFLAGS) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(FAULT_TEST_OBJ): tests/failure/test_failure.c get_next_line.h \
+		tests/support/fault_runtime.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(FAULT_CPPFLAGS) $(CFLAGS) $(DEPFLAGS) -c $< -o $@
+
+$(FAULT_BIN): $(FAULT_OBJ)
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CFLAGS) $(FAULT_OBJ) -o $@
+
+failure-run: $(FAULT_BIN)
+	./$(FAULT_BIN)
+
+failure-test:
+	@set -e; for size in $(MATRIX_SIZES); do \
+		$(MAKE) --no-print-directory failure-run BUFFER_SIZE=$$size; \
+	done
+
 check:
 	git diff --check
 	$(MAKE) fclean
 	$(MAKE) all
 	$(MAKE) test
+	$(MAKE) failure-test
 	$(MAKE) -q all
 
 clean:
@@ -55,4 +94,4 @@ fclean: clean
 
 re: fclean all
 
--include $(DEP)
+-include $(DEP) $(FAULT_DEP)
diff --git a/tests/failure/test_failure.c b/tests/failure/test_failure.c
new file mode 100644
index 0000000..2fabe36
--- /dev/null
+++ b/tests/failure/test_failure.c
@@ -0,0 +1,229 @@
+#include "fault_runtime.h"
+#include "get_next_line.h"
+
+#include <stdio.h>
+#include <string.h>
+#include <unistd.h>
+
+static int	g_checks;
+static int	g_failures;
+
+static void	check(int condition, const char *expression, int line)
+{
+	g_checks++;
+	if (!condition)
+	{
+		g_failures++;
+		fprintf(stderr, "%s:%d: check failed: %s\n", __FILE__, line,
+			expression);
+	}
+}
+
+#define CHECK(condition) check((condition), #condition, __LINE__)
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
+static void	check_clean_runtime(void)
+{
+	CHECK(fault_live_allocations() == 0);
+	CHECK(fault_invalid_frees() == 0);
+	CHECK(fault_double_frees() == 0);
+}
+
+static size_t	consume_all(size_t fail_at, int *failure_observed)
+{
+	char	*line;
+	int		fd;
+	size_t	attempts;
+
+	fault_runtime_reset();
+	fault_allocation_fail_at(fail_at);
+	fd = reader_for("alpha\nbeta\nlast");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return (0);
+	line = get_next_line(fd);
+	while (line != NULL)
+	{
+		test_free(line);
+		line = get_next_line(fd);
+	}
+	attempts = fault_allocation_attempts();
+	*failure_observed = fault_allocation_failed();
+	close(fd);
+	check_clean_runtime();
+	return (attempts);
+}
+
+static void	test_allocation_failures(void)
+{
+	int		failed;
+	size_t	attempts;
+	size_t	baseline;
+	size_t	index;
+
+	baseline = consume_all(0, &failed);
+	CHECK(!failed);
+	CHECK(baseline > 0);
+	index = 1;
+	while (index <= baseline)
+	{
+		attempts = consume_all(index, &failed);
+		CHECK(failed);
+		CHECK(attempts == index);
+		index++;
+	}
+	consume_all(baseline + 1, &failed);
+	CHECK(!failed);
+}
+
+static void	test_short_reads(void)
+{
+	char	*line;
+	int		fd;
+
+	fault_runtime_reset();
+	fd = reader_for("short reads still work\nlast");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	fault_read_fail_on(fd, 0);
+	fault_read_limit(3);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "short reads still work\n") == 0);
+	test_free(line);
+	line = get_next_line(fd);
+	CHECK(line != NULL && strcmp(line, "last") == 0);
+	test_free(line);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(fault_read_calls() > 2);
+	CHECK(!fault_read_failed());
+	close(fd);
+	check_clean_runtime();
+}
+
+static void	test_first_read_error(void)
+{
+	int	fd;
+
+	fault_runtime_reset();
+	fd = reader_for("unread input");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	fault_read_fail_on(fd, 1);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(fault_read_calls() == 1);
+	CHECK(fault_read_failed());
+	close(fd);
+	check_clean_runtime();
+}
+
+static void	test_middle_read_error(void)
+{
+	int	fd;
+
+	fault_runtime_reset();
+	fd = reader_for("partial bytes must be discarded");
+	CHECK(fd >= 0);
+	if (fd < 0)
+		return ;
+	fault_read_limit(4);
+	fault_read_fail_on(fd, 3);
+	CHECK(get_next_line(fd) == NULL);
+	CHECK(fault_read_calls() == 3);
+	CHECK(fault_read_failed());
+	close(fd);
+	check_clean_runtime();
+}
+
+static void	test_one_fd_failure_preserves_another(void)
+{
+	char	*line;
+	int		left;
+	int		right;
+
+	fault_runtime_reset();
+	left = reader_for("left one\nleft two");
+	right = reader_for("right stream fails");
+	CHECK(left >= 0 && right >= 0);
+	if (left < 0 || right < 0)
+		return ;
+	line = get_next_line(left);
+	CHECK(line != NULL && strcmp(line, "left one\n") == 0);
+	test_free(line);
+	fault_read_fail_on(right, 1);
+	CHECK(get_next_line(right) == NULL);
+	CHECK(fault_read_failed());
+	line = get_next_line(left);
+	CHECK(line != NULL && strcmp(line, "left two") == 0);
+	test_free(line);
+	CHECK(get_next_line(left) == NULL);
+	close(left);
+	close(right);
+	check_clean_runtime();
+}
+
+static void	test_free_guards(void)
+{
+	char	local;
+	void	*allocation;
+
+	fault_runtime_reset();
+	allocation = test_malloc(1);
+	CHECK(allocation != NULL);
+	test_free(allocation);
+	test_free(allocation);
+	test_free(&local);
+	CHECK(fault_live_allocations() == 0);
+	CHECK(fault_double_frees() == 1);
+	CHECK(fault_invalid_frees() == 1);
+	fault_runtime_reset();
+	check_clean_runtime();
+}
+
+int	main(void)
+{
+	test_allocation_failures();
+	test_short_reads();
+	test_first_read_error();
+	test_middle_read_error();
+	test_one_fd_failure_preserves_another();
+	test_free_guards();
+	if (g_failures != 0)
+	{
+		fprintf(stderr, "%d of %d checks failed\n", g_failures, g_checks);
+		return (1);
+	}
+	printf("%d failure checks passed\n", g_checks);
+	return (0);
+}
diff --git a/tests/support/fault_runtime.c b/tests/support/fault_runtime.c
new file mode 100644
index 0000000..7617c23
--- /dev/null
+++ b/tests/support/fault_runtime.c
@@ -0,0 +1,187 @@
+#ifdef malloc
+# undef malloc
+#endif
+#ifdef free
+# undef free
+#endif
+#ifdef read
+# undef read
+#endif
+
+#include "fault_runtime.h"
+
+#include <errno.h>
+#include <stdlib.h>
+#include <unistd.h>
+
+#define MAX_TRACKED_ALLOCATIONS 4096
+
+typedef struct s_allocation
+{
+	void	*pointer;
+	int		live;
+}t_allocation;
+
+static t_allocation	g_allocations[MAX_TRACKED_ALLOCATIONS];
+static size_t		g_allocation_count;
+static size_t		g_allocation_attempts;
+static size_t		g_allocation_fail_at;
+static size_t		g_live_allocations;
+static size_t		g_invalid_frees;
+static size_t		g_double_frees;
+static int			g_allocation_failed;
+static int			g_read_fd;
+static size_t		g_read_calls;
+static size_t		g_read_fail_at;
+static size_t		g_read_limit;
+static int			g_read_failed;
+
+void	fault_runtime_reset(void)
+{
+	if (g_live_allocations == 0)
+		g_allocation_count = 0;
+	g_allocation_attempts = 0;
+	g_allocation_fail_at = 0;
+	g_invalid_frees = 0;
+	g_double_frees = 0;
+	g_allocation_failed = 0;
+	g_read_fd = -1;
+	g_read_calls = 0;
+	g_read_fail_at = 0;
+	g_read_limit = 0;
+	g_read_failed = 0;
+}
+
+void	fault_allocation_fail_at(size_t attempt)
+{
+	g_allocation_fail_at = attempt;
+}
+
+size_t	fault_allocation_attempts(void)
+{
+	return (g_allocation_attempts);
+}
+
+int	fault_allocation_failed(void)
+{
+	return (g_allocation_failed);
+}
+
+size_t	fault_live_allocations(void)
+{
+	return (g_live_allocations);
+}
+
+size_t	fault_invalid_frees(void)
+{
+	return (g_invalid_frees);
+}
+
+size_t	fault_double_frees(void)
+{
+	return (g_double_frees);
+}
+
+void	fault_read_fail_on(int fd, size_t call_index)
+{
+	g_read_fd = fd;
+	g_read_calls = 0;
+	g_read_fail_at = call_index;
+	g_read_failed = 0;
+}
+
+void	fault_read_limit(size_t limit)
+{
+	g_read_limit = limit;
+}
+
+size_t	fault_read_calls(void)
+{
+	return (g_read_calls);
+}
+
+int	fault_read_failed(void)
+{
+	return (g_read_failed);
+}
+
+void	*test_malloc(size_t size)
+{
+	void	*pointer;
+
+	g_allocation_attempts++;
+	if (g_allocation_fail_at != 0
+		&& g_allocation_attempts == g_allocation_fail_at)
+	{
+		g_allocation_failed = 1;
+		return (NULL);
+	}
+	pointer = malloc(size);
+	if (pointer == NULL)
+	{
+		g_allocation_failed = 1;
+		return (NULL);
+	}
+	if (g_allocation_count == MAX_TRACKED_ALLOCATIONS)
+	{
+		free(pointer);
+		g_allocation_failed = 1;
+		return (NULL);
+	}
+	g_allocations[g_allocation_count].pointer = pointer;
+	g_allocations[g_allocation_count].live = 1;
+	g_allocation_count++;
+	g_live_allocations++;
+	return (pointer);
+}
+
+void	test_free(void *pointer)
+{
+	size_t	index;
+
+	if (pointer == NULL)
+		return ;
+	index = g_allocation_count;
+	while (index > 0)
+	{
+		index--;
+		if (g_allocations[index].pointer == pointer
+			&& g_allocations[index].live)
+		{
+			g_allocations[index].live = 0;
+			g_live_allocations--;
+			free(pointer);
+			return ;
+		}
+	}
+	index = g_allocation_count;
+	while (index > 0)
+	{
+		index--;
+		if (g_allocations[index].pointer == pointer)
+		{
+			g_double_frees++;
+			return ;
+		}
+	}
+	g_invalid_frees++;
+}
+
+ssize_t	test_read(int fd, void *buffer, size_t count)
+{
+	if (count == 0)
+		return (read(fd, buffer, count));
+	if (fd == g_read_fd)
+	{
+		g_read_calls++;
+		if (g_read_fail_at != 0 && g_read_calls == g_read_fail_at)
+		{
+			g_read_failed = 1;
+			errno = EIO;
+			return (-1);
+		}
+	}
+	if (g_read_limit != 0 && count > g_read_limit)
+		count = g_read_limit;
+	return (read(fd, buffer, count));
+}
diff --git a/tests/support/fault_runtime.h b/tests/support/fault_runtime.h
new file mode 100644
index 0000000..078aedd
--- /dev/null
+++ b/tests/support/fault_runtime.h
@@ -0,0 +1,25 @@
+#ifndef FAULT_RUNTIME_H
+# define FAULT_RUNTIME_H
+
+# include <stddef.h>
+# include <sys/types.h>
+
+void	fault_runtime_reset(void);
+
+void	fault_allocation_fail_at(size_t attempt);
+size_t	fault_allocation_attempts(void);
+int		fault_allocation_failed(void);
+size_t	fault_live_allocations(void);
+size_t	fault_invalid_frees(void);
+size_t	fault_double_frees(void);
+
+void	fault_read_fail_on(int fd, size_t call_index);
+void	fault_read_limit(size_t limit);
+size_t	fault_read_calls(void);
+int		fault_read_failed(void);
+
+void	*test_malloc(size_t size);
+void	test_free(void *pointer);
+ssize_t	test_read(int fd, void *buffer, size_t count);
+
+#endif


## `feat(context): 명시적 reader 수명 API 추가`

diff --git a/get_next_line.c b/get_next_line.c
index 7f80c08..7c0c3ef 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -3,7 +3,7 @@
 #include <stdlib.h>
 #include <unistd.h>
 
-typedef struct s_reader
+struct s_blr_reader
 {
 	int			fd;
 	char		*bytes;
@@ -11,10 +11,10 @@ typedef struct s_reader
 	size_t		scan;
 	size_t		end;
 	size_t		capacity;
-	struct s_reader	*next;
-}t_reader;
+	t_blr_reader	*next;
+};
 
-static t_reader	*g_readers;
+static t_blr_reader	*g_readers;
 
 static void	copy_bytes(char *destination, const char *source, size_t length)
 {
@@ -28,9 +28,9 @@ static void	copy_bytes(char *destination, const char *source, size_t length)
 	}
 }
 
-static t_reader	*find_reader(int fd)
+static t_blr_reader	*find_reader(int fd)
 {
-	t_reader	*reader;
+	t_blr_reader	*reader;
 
 	reader = g_readers;
 	while (reader != NULL && reader->fd != fd)
@@ -38,10 +38,13 @@ static t_reader	*find_reader(int fd)
 	return (reader);
 }
 
-static t_reader	*create_reader(int fd)
+t_blr_reader	*blr_reader_create(int fd)
 {
-	t_reader	*reader;
+	char			probe;
+	t_blr_reader	*reader;
 
+	if (fd < 0 || read(fd, &probe, 0) < 0)
+		return (NULL);
 	reader = malloc(sizeof(*reader));
 	if (reader == NULL)
 		return (NULL);
@@ -51,14 +54,33 @@ static t_reader	*create_reader(int fd)
 	reader->scan = 0;
 	reader->end = 0;
 	reader->capacity = 0;
-	reader->next = g_readers;
-	g_readers = reader;
+	reader->next = NULL;
 	return (reader);
 }
 
-static void	discard_reader(t_reader *reader)
+void	blr_reader_reset(t_blr_reader *reader)
 {
-	t_reader	**link;
+	if (reader == NULL)
+		return ;
+	free(reader->bytes);
+	reader->bytes = NULL;
+	reader->begin = 0;
+	reader->scan = 0;
+	reader->end = 0;
+	reader->capacity = 0;
+}
+
+void	blr_reader_destroy(t_blr_reader *reader)
+{
+	if (reader == NULL)
+		return ;
+	free(reader->bytes);
+	free(reader);
+}
+
+static void	discard_legacy_reader(t_blr_reader *reader)
+{
+	t_blr_reader	**link;
 
 	link = &g_readers;
 	while (*link != NULL && *link != reader)
@@ -66,16 +88,27 @@ static void	discard_reader(t_reader *reader)
 	if (*link == NULL)
 		return ;
 	*link = reader->next;
-	free(reader->bytes);
-	free(reader);
+	blr_reader_destroy(reader);
+}
+
+static t_blr_reader	*create_legacy_reader(int fd)
+{
+	t_blr_reader	*reader;
+
+	reader = blr_reader_create(fd);
+	if (reader == NULL)
+		return (NULL);
+	reader->next = g_readers;
+	g_readers = reader;
+	return (reader);
 }
 
-static size_t	unread_length(const t_reader *reader)
+static size_t	unread_length(const t_blr_reader *reader)
 {
 	return (reader->end - reader->begin);
 }
 
-static void	compact_bytes(t_reader *reader)
+static void	compact_bytes(t_blr_reader *reader)
 {
 	size_t	length;
 
@@ -87,7 +120,7 @@ static void	compact_bytes(t_reader *reader)
 	reader->bytes[reader->end] = '\0';
 }
 
-static int	reserve_bytes(t_reader *reader, size_t appended)
+static int	reserve_bytes(t_blr_reader *reader, size_t appended)
 {
 	char	*allocation;
 	size_t	capacity;
@@ -134,7 +167,7 @@ static int	reserve_bytes(t_reader *reader, size_t appended)
 	return (1);
 }
 
-static size_t	find_line_end(t_reader *reader)
+static size_t	find_line_end(t_blr_reader *reader)
 {
 	while (reader->scan < reader->end)
 	{
@@ -148,7 +181,7 @@ static size_t	find_line_end(t_reader *reader)
 	return (0);
 }
 
-static char	*extract_line(t_reader *reader, size_t line_end)
+static char	*extract_line(t_blr_reader *reader, size_t line_end)
 {
 	char	*line;
 	size_t	length;
@@ -157,7 +190,7 @@ static char	*extract_line(t_reader *reader, size_t line_end)
 	line = malloc(length + 1);
 	if (line == NULL)
 	{
-		discard_reader(reader);
+		discard_legacy_reader(reader);
 		return (NULL);
 	}
 	copy_bytes(line, reader->bytes + reader->begin, length);
@@ -165,11 +198,11 @@ static char	*extract_line(t_reader *reader, size_t line_end)
 	reader->begin = line_end;
 	reader->scan = reader->begin;
 	if (reader->begin == reader->end)
-		discard_reader(reader);
+		discard_legacy_reader(reader);
 	return (line);
 }
 
-static char	*release_final_line(t_reader *reader)
+static char	*release_final_line(t_blr_reader *reader)
 {
 	char	*line;
 	size_t	length;
@@ -180,7 +213,7 @@ static char	*release_final_line(t_reader *reader)
 	reader->bytes[length] = '\0';
 	line = reader->bytes;
 	reader->bytes = NULL;
-	discard_reader(reader);
+	discard_legacy_reader(reader);
 	return (line);
 }
 
@@ -189,18 +222,18 @@ char	*get_next_line(int fd)
 	char		probe;
 	ssize_t	read_size;
 	size_t	line_end;
-	t_reader	*reader;
+	t_blr_reader	*reader;
 
 	if (fd < 0 || read(fd, &probe, 0) < 0)
 	{
 		reader = find_reader(fd);
 		if (reader != NULL)
-			discard_reader(reader);
+			discard_legacy_reader(reader);
 		return (NULL);
 	}
 	reader = find_reader(fd);
 	if (reader == NULL)
-		reader = create_reader(fd);
+		reader = create_legacy_reader(fd);
 	if (reader == NULL)
 		return (NULL);
 	line_end = find_line_end(reader);
@@ -210,7 +243,7 @@ char	*get_next_line(int fd)
 	{
 		if (!reserve_bytes(reader, (size_t)BUFFER_SIZE))
 		{
-			discard_reader(reader);
+			discard_legacy_reader(reader);
 			return (NULL);
 		}
 		read_size = read(fd, reader->bytes + reader->end,
@@ -225,12 +258,12 @@ char	*get_next_line(int fd)
 	}
 	if (read_size < 0)
 	{
-		discard_reader(reader);
+		discard_legacy_reader(reader);
 		return (NULL);
 	}
 	if (unread_length(reader) == 0)
 	{
-		discard_reader(reader);
+		discard_legacy_reader(reader);
 		return (NULL);
 	}
 	return (release_final_line(reader));
diff --git a/get_next_line.h b/get_next_line.h
index e48c67f..c49f1a3 100644
--- a/get_next_line.h
+++ b/get_next_line.h
@@ -9,6 +9,12 @@
 #  error "BUFFER_SIZE must be greater than zero"
 # endif
 
+typedef struct s_blr_reader	t_blr_reader;
+
+t_blr_reader	*blr_reader_create(int fd);
+void			blr_reader_reset(t_blr_reader *reader);
+void			blr_reader_destroy(t_blr_reader *reader);
+
 char	*get_next_line(int fd);
 
 #endif
diff --git a/tests/manifests/api-symbols.txt b/tests/manifests/api-symbols.txt
index 80de17c..8890e35 100644
--- a/tests/manifests/api-symbols.txt
+++ b/tests/manifests/api-symbols.txt
@@ -1 +1,4 @@
+blr_reader_create
+blr_reader_destroy
+blr_reader_reset
 get_next_line


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


