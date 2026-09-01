## `test(perf): 4 MiB 입력의 작업량 기준 고정`

diff --git a/Makefile b/Makefile
index 5128e14..5a5ffa2 100644
--- a/Makefile
+++ b/Makefile
@@ -37,9 +37,21 @@ ASAN_BIN := tests/bin/test_asan_$(BUFFER_SIZE)
 UBSAN_FLAGS := -fsanitize=undefined -fno-sanitize-recover=all
 UBSAN_BIN := tests/bin/test_ubsan_$(BUFFER_SIZE)
 
+METRIC_BUFFER_SIZE := 4096
+METRIC_OBJ_DIR := build/metrics
+METRIC_READER_OBJ := $(METRIC_OBJ_DIR)/get_next_line.o
+METRIC_RUNTIME_OBJ := $(METRIC_OBJ_DIR)/metric_runtime.o
+METRIC_TEST_OBJ := $(METRIC_OBJ_DIR)/test_metrics.o
+METRIC_OBJ := $(METRIC_READER_OBJ) $(METRIC_RUNTIME_OBJ) $(METRIC_TEST_OBJ)
+METRIC_BIN := tests/bin/test_metrics
+METRIC_ACTUAL := build/metrics-4mib.txt
+METRIC_CPPFLAGS := -I. -Itests/metrics -DBUFFER_SIZE=$(METRIC_BUFFER_SIZE)
+METRIC_DEFINES := -Dmalloc=metric_malloc -Dfree=metric_free \
+	-Dread=metric_read -DBLR_COPY_OBSERVER=metric_copy_observer
+
 .PHONY: all bonus clean fclean re test-run test failure-run failure-test \
-	asan-run asan ubsan-run ubsan sanitize leak-run leak check-archive \
-	check-consumer check-buffer-size check
+	asan-run asan ubsan-run ubsan sanitize metrics leak-run leak \
+	check-archive check-consumer check-buffer-size check
 
 all: $(NAME)
 
@@ -123,6 +135,29 @@ ubsan:
 
 sanitize: asan ubsan
 
+$(METRIC_READER_OBJ): get_next_line.c get_next_line.h \
+		tests/metrics/metric_runtime.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(METRIC_CPPFLAGS) $(METRIC_DEFINES) $(CFLAGS) -c $< -o $@
+
+$(METRIC_RUNTIME_OBJ): tests/metrics/metric_runtime.c \
+		tests/metrics/metric_runtime.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(METRIC_CPPFLAGS) $(CFLAGS) -c $< -o $@
+
+$(METRIC_TEST_OBJ): tests/metrics/test_metrics.c get_next_line.h \
+		tests/metrics/metric_runtime.h
+	@$(MKDIR) $(dir $@)
+	$(CC) $(METRIC_CPPFLAGS) $(CFLAGS) -c $< -o $@
+
+$(METRIC_BIN): $(METRIC_OBJ)
+	@$(MKDIR) $(dir $@)
+	$(CC) $(CFLAGS) $(METRIC_OBJ) -o $@
+
+metrics: $(METRIC_BIN)
+	./$(METRIC_BIN) >$(METRIC_ACTUAL)
+	diff -u tests/manifests/metrics-4mib.txt $(METRIC_ACTUAL)
+
 leak-run: $(TEST_BIN)
 	@if [ "$$(uname -s)" = Darwin ] && [ "$(RUN_LEAKS)" != 1 ]; then \
 		echo "leaks execution skipped on Darwin (set RUN_LEAKS=1 to force)"; \
@@ -165,6 +200,7 @@ check:
 	$(MAKE) test
 	$(MAKE) failure-test
 	$(MAKE) sanitize
+	$(MAKE) metrics
 	$(MAKE) leak
 	$(MAKE) -q all
 
diff --git a/get_next_line.c b/get_next_line.c
index 4c2212c..8cec88b 100644
--- a/get_next_line.c
+++ b/get_next_line.c
@@ -4,6 +4,10 @@
 #include <stdlib.h>
 #include <unistd.h>
 
+#ifdef BLR_COPY_OBSERVER
+void	BLR_COPY_OBSERVER(size_t length);
+#endif
+
 struct s_blr_reader
 {
 	int			fd;
@@ -32,6 +36,10 @@ static void	copy_bytes(char *destination, const char *source, size_t length)
 {
 	size_t	index;
 
+#ifdef BLR_COPY_OBSERVER
+	if (length > 0)
+		BLR_COPY_OBSERVER(length);
+#endif
 	index = 0;
 	while (index < length)
 	{
diff --git a/tests/manifests/metrics-4mib.txt b/tests/manifests/metrics-4mib.txt
new file mode 100644
index 0000000..18d843f
--- /dev/null
+++ b/tests/manifests/metrics-4mib.txt
@@ -0,0 +1,9 @@
+input_bytes=4194304
+line_bytes=4194304
+checksum=790796585941148453
+read_calls=1025
+read_bytes=4194304
+allocation_calls=13
+allocation_bytes=20963393
+copy_calls=11
+copy_bytes=12533760
diff --git a/tests/metrics/metric_runtime.c b/tests/metrics/metric_runtime.c
new file mode 100644
index 0000000..b1a0007
--- /dev/null
+++ b/tests/metrics/metric_runtime.c
@@ -0,0 +1,81 @@
+#include "metric_runtime.h"
+
+#include <stdlib.h>
+#include <unistd.h>
+
+static size_t	g_allocation_calls;
+static size_t	g_allocation_bytes;
+static size_t	g_copy_calls;
+static size_t	g_copy_bytes;
+static size_t	g_read_calls;
+static size_t	g_read_bytes;
+
+void	metric_reset(void)
+{
+	g_allocation_calls = 0;
+	g_allocation_bytes = 0;
+	g_copy_calls = 0;
+	g_copy_bytes = 0;
+	g_read_calls = 0;
+	g_read_bytes = 0;
+}
+
+size_t	metric_allocation_calls(void)
+{
+	return (g_allocation_calls);
+}
+
+size_t	metric_allocation_bytes(void)
+{
+	return (g_allocation_bytes);
+}
+
+size_t	metric_copy_calls(void)
+{
+	return (g_copy_calls);
+}
+
+size_t	metric_copy_bytes(void)
+{
+	return (g_copy_bytes);
+}
+
+size_t	metric_read_calls(void)
+{
+	return (g_read_calls);
+}
+
+size_t	metric_read_bytes(void)
+{
+	return (g_read_bytes);
+}
+
+void	*metric_malloc(size_t size)
+{
+	g_allocation_calls++;
+	g_allocation_bytes += size;
+	return (malloc(size));
+}
+
+void	metric_free(void *pointer)
+{
+	free(pointer);
+}
+
+ssize_t	metric_read(int fd, void *buffer, size_t count)
+{
+	ssize_t	read_size;
+
+	if (count > 0)
+		g_read_calls++;
+	read_size = read(fd, buffer, count);
+	if (count > 0 && read_size > 0)
+		g_read_bytes += (size_t)read_size;
+	return (read_size);
+}
+
+void	metric_copy_observer(size_t length)
+{
+	g_copy_calls++;
+	g_copy_bytes += length;
+}
diff --git a/tests/metrics/metric_runtime.h b/tests/metrics/metric_runtime.h
new file mode 100644
index 0000000..f8b3bce
--- /dev/null
+++ b/tests/metrics/metric_runtime.h
@@ -0,0 +1,20 @@
+#ifndef METRIC_RUNTIME_H
+# define METRIC_RUNTIME_H
+
+# include <stddef.h>
+# include <sys/types.h>
+
+void	metric_reset(void);
+size_t	metric_allocation_calls(void);
+size_t	metric_allocation_bytes(void);
+size_t	metric_copy_calls(void);
+size_t	metric_copy_bytes(void);
+size_t	metric_read_calls(void);
+size_t	metric_read_bytes(void);
+
+void	*metric_malloc(size_t size);
+void	metric_free(void *pointer);
+ssize_t	metric_read(int fd, void *buffer, size_t count);
+void	metric_copy_observer(size_t length);
+
+#endif
diff --git a/tests/metrics/test_metrics.c b/tests/metrics/test_metrics.c
new file mode 100644
index 0000000..e8e6669
--- /dev/null
+++ b/tests/metrics/test_metrics.c
@@ -0,0 +1,128 @@
+#define _POSIX_C_SOURCE 200809L
+
+#include "metric_runtime.h"
+#include "get_next_line.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+#include <time.h>
+#include <unistd.h>
+
+#define INPUT_SIZE ((size_t)4 * 1024 * 1024)
+#define BLOCK_SIZE ((size_t)8192)
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
+static int	make_input(void)
+{
+	char	block[BLOCK_SIZE];
+	char	path[] = "/tmp/buffered-line-metrics-XXXXXX";
+	int		fd;
+	size_t	index;
+	size_t	written;
+
+	index = 0;
+	while (index < BLOCK_SIZE)
+	{
+		block[index] = (char)('a' + index % 26);
+		index++;
+	}
+	fd = mkstemp(path);
+	if (fd < 0)
+		return (-1);
+	unlink(path);
+	written = 0;
+	while (written < INPUT_SIZE)
+	{
+		if (!write_all(fd, block, BLOCK_SIZE))
+		{
+			close(fd);
+			return (-1);
+		}
+		written += BLOCK_SIZE;
+	}
+	if (lseek(fd, 0, SEEK_SET) < 0)
+	{
+		close(fd);
+		return (-1);
+	}
+	return (fd);
+}
+
+static unsigned long long	checksum(const char *bytes, size_t length)
+{
+	unsigned long long	hash;
+	size_t				index;
+
+	hash = 14695981039346656037ULL;
+	index = 0;
+	while (index < length)
+	{
+		hash ^= (unsigned char)bytes[index];
+		hash *= 1099511628211ULL;
+		index++;
+	}
+	return (hash);
+}
+
+static unsigned long long	elapsed_ns(struct timespec start,
+		struct timespec end)
+{
+	return ((unsigned long long)(end.tv_sec - start.tv_sec) * 1000000000ULL
+		+ (unsigned long long)(end.tv_nsec - start.tv_nsec));
+}
+
+int	main(void)
+{
+	char			*line;
+	int				fd;
+	size_t			length;
+	struct timespec	finished;
+	struct timespec	started;
+	t_blr_reader	*reader;
+
+	fd = make_input();
+	if (fd < 0)
+		return (1);
+	metric_reset();
+	clock_gettime(CLOCK_MONOTONIC, &started);
+	reader = blr_reader_create(fd);
+	if (reader == NULL || blr_reader_next(reader, &line) != BLR_LINE)
+	{
+		blr_reader_destroy(reader);
+		close(fd);
+		return (1);
+	}
+	clock_gettime(CLOCK_MONOTONIC, &finished);
+	length = 0;
+	while (line[length] != '\0')
+		length++;
+	printf("input_bytes=%zu\n", INPUT_SIZE);
+	printf("line_bytes=%zu\n", length);
+	printf("checksum=%llu\n", checksum(line, length));
+	printf("read_calls=%zu\n", metric_read_calls());
+	printf("read_bytes=%zu\n", metric_read_bytes());
+	printf("allocation_calls=%zu\n", metric_allocation_calls());
+	printf("allocation_bytes=%zu\n", metric_allocation_bytes());
+	printf("copy_calls=%zu\n", metric_copy_calls());
+	printf("copy_bytes=%zu\n", metric_copy_bytes());
+	fprintf(stderr, "wall_ns=%llu (informational)\n",
+		elapsed_ns(started, finished));
+	metric_free(line);
+	blr_reader_destroy(reader);
+	close(fd);
+	return (length != INPUT_SIZE);
+}
