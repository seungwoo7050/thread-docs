# 빌드와 동시성 검증 파이프라인

## `build(philo): 실행 파일 빌드 규약 구성`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..e60752c
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,5 @@
+philo
+*.o
+*.d
+*.out
+.DS_Store
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..ec6fd3b
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,35 @@
+NAME := philo
+
+CC := cc
+CFLAGS := -Wall -Wextra -Werror -pthread -Iinclude
+
+SRC_DIR := src
+OBJ_DIR := .obj
+
+SRCS := \
+	$(SRC_DIR)/main.c \
+	$(SRC_DIR)/parse.c
+OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
+
+.PHONY: all bonus clean fclean re
+
+all: $(NAME)
+
+$(NAME): $(OBJS)
+	$(CC) $(CFLAGS) $(OBJS) -o $@
+
+$(OBJ_DIR)/%.o: $(SRC_DIR)/%.c include/philo.h
+	@mkdir -p $(dir $@)
+	$(CC) $(CFLAGS) -c $< -o $@
+
+bonus:
+	@printf 'bonus target is unavailable\n'
+	@exit 1
+
+clean:
+	rm -rf $(OBJ_DIR)
+
+fclean: clean
+	rm -f $(NAME)
+
+re: fclean all


## `test(smoke): 주요 입력과 종료 조건 검증`

diff --git a/Makefile b/Makefile
index 7220db3..06b7ab2 100644
--- a/Makefile
+++ b/Makefile
@@ -17,7 +17,7 @@ SRCS := \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
 
-.PHONY: all bonus clean fclean re
+.PHONY: all bonus clean fclean re test
 
 all: $(NAME)
 
@@ -39,3 +39,6 @@ fclean: clean
 	rm -f $(NAME)
 
 re: fclean all
+
+test: all
+	./tests/smoke.sh
diff --git a/tests/smoke.sh b/tests/smoke.sh
new file mode 100755
index 0000000..32b9ce8
--- /dev/null
+++ b/tests/smoke.sh
@@ -0,0 +1,75 @@
+#!/bin/sh
+
+set -eu
+
+ROOT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TMP_DIR=$(mktemp -d)
+
+cleanup()
+{
+	rm -rf "$TMP_DIR"
+}
+
+fail()
+{
+	printf 'smoke: %s\n' "$1" >&2
+	exit 1
+}
+
+run_timeout()
+{
+	limit=$1
+	outfile=$2
+	shift 2
+	"$@" >"$outfile" 2>&1 &
+	pid=$!
+	(
+		sleep "$limit"
+		kill -TERM "$pid" 2>/dev/null || true
+	) &
+	guard=$!
+	set +e
+	wait "$pid"
+	status=$?
+	set -e
+	kill "$guard" 2>/dev/null || true
+	wait "$guard" 2>/dev/null || true
+	return "$status"
+}
+
+trap cleanup EXIT INT TERM
+
+make -C "$ROOT_DIR" >/dev/null
+
+invalid_out="$TMP_DIR/invalid.out"
+if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
+	fail 'invalid philosopher count succeeded'
+fi
+grep -q 'Usage: ./philo' "$invalid_out" || fail 'invalid args did not print usage'
+
+overflow_out="$TMP_DIR/overflow.out"
+if "$ROOT_DIR/philo" 2 999999999999999999999 10 10 >"$overflow_out" 2>&1; then
+	fail 'overflow argument succeeded'
+fi
+
+single_out="$TMP_DIR/single.out"
+run_timeout 2 "$single_out" "$ROOT_DIR/philo" 1 80 40 40 \
+	|| fail 'single philosopher did not exit cleanly'
+grep -q '1 has taken a fork' "$single_out" || fail 'single philosopher missed fork log'
+grep -q '1 died' "$single_out" || fail 'single philosopher missed death log'
+
+finite_out="$TMP_DIR/finite.out"
+run_timeout 3 "$finite_out" "$ROOT_DIR/philo" 2 250 50 50 2 \
+	|| fail 'finite meal run did not exit cleanly'
+grep -q 'died' "$finite_out" && fail 'finite meal run had a death'
+eat_count=$(grep -c 'is eating' "$finite_out" || true)
+[ "$eat_count" -ge 4 ] || fail 'finite meal run did not eat enough'
+
+nodeath_out="$TMP_DIR/nodeath.out"
+run_timeout 5 "$nodeath_out" "$ROOT_DIR/philo" 5 800 100 100 3 \
+	|| fail 'no-death meal run did not exit cleanly'
+grep -q 'died' "$nodeath_out" && fail 'no-death meal run had a death'
+eat_count=$(grep -c 'is eating' "$nodeath_out" || true)
+[ "$eat_count" -ge 15 ] || fail 'no-death meal run did not reach meal count'
+
+printf 'smoke: ok\n'


## `test(init): 부분 뮤텍스 초기화 롤백 검증`

diff --git a/tests/init_failure.c b/tests/init_failure.c
new file mode 100644
index 0000000..51c9908
--- /dev/null
+++ b/tests/init_failure.c
@@ -0,0 +1,74 @@
+#include "philo.h"
+
+#include <pthread.h>
+#include <stddef.h>
+#include <stdio.h>
+
+static int		init_calls;
+static void		*destroyed[32];
+static size_t	destroy_count;
+static int		duplicate_destroy;
+
+int	test_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr)
+{
+	(void)mutex;
+	(void)attr;
+	init_calls++;
+	if (init_calls == 4)
+		return (1);
+	return (0);
+}
+
+int	test_mutex_destroy(pthread_mutex_t *mutex)
+{
+	size_t	i;
+
+	i = 0;
+	while (i < destroy_count)
+	{
+		if (destroyed[i] == (void *)mutex)
+		{
+			duplicate_destroy = 1;
+			return (1);
+		}
+		i++;
+	}
+	destroyed[destroy_count++] = (void *)mutex;
+	return (0);
+}
+
+int	main(void)
+{
+	t_config	config;
+	t_table		table;
+
+	config.number = 4;
+	config.time_to_die = 100;
+	config.time_to_eat = 20;
+	config.time_to_sleep = 20;
+	config.must_eat = 0;
+	config.has_meal_limit = 0;
+	if (philo_table_init(&table, &config) != PHILO_ERR)
+	{
+		fprintf(stderr, "injected init failure was not propagated\n");
+		return (1);
+	}
+	if (duplicate_destroy || destroy_count != 3)
+	{
+		fprintf(stderr, "mutex rollback did not destroy each resource once\n");
+		return (1);
+	}
+	if (table.forks != NULL || table.philos != NULL)
+	{
+		fprintf(stderr, "failed init retained allocations\n");
+		return (1);
+	}
+	philo_table_destroy(&table);
+	if (duplicate_destroy || destroy_count != 3)
+	{
+		fprintf(stderr, "cleanup was not idempotent after failed init\n");
+		return (1);
+	}
+	puts("init failure rollback: ok");
+	return (0);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index e2f418a..6ecc31f 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -50,6 +50,15 @@ trap cleanup EXIT INT TERM
 
 make -C "$ROOT_DIR" >/dev/null
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_mutex_init=test_mutex_init \
+	-Dpthread_mutex_destroy=test_mutex_destroy \
+	-c "$ROOT_DIR/src/init.c" -o "$TMP_DIR/init_failure_init.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/init_failure.c" "$TMP_DIR/init_failure_init.o" \
+	-o "$TMP_DIR/init_failure"
+"$TMP_DIR/init_failure" || fail 'partial mutex initialization cleanup failed'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'


## `test(lifecycle): 생성·결합·정리 실패 경로 검증`

diff --git a/tests/lifecycle_failure.c b/tests/lifecycle_failure.c
new file mode 100644
index 0000000..86a786e
--- /dev/null
+++ b/tests/lifecycle_failure.c
@@ -0,0 +1,156 @@
+#include "philo.h"
+
+#include <errno.h>
+#include <pthread.h>
+#include <stdio.h>
+
+static int	g_create_calls;
+static int	g_fail_create_at;
+static int	g_join_calls;
+static int	g_fail_join_at;
+static int	g_destroy_calls;
+static int	g_fail_destroy_at;
+
+int	test_pthread_create(pthread_t *thread, const pthread_attr_t *attr,
+		void *(*routine)(void *), void *arg)
+{
+	int	call;
+
+	call = g_create_calls++;
+	if (call == g_fail_create_at)
+		return (EAGAIN);
+	return (pthread_create(thread, attr, routine, arg));
+}
+
+int	test_pthread_join(pthread_t thread, void **result)
+{
+	int	call;
+	int	status;
+
+	call = g_join_calls++;
+	if (call == g_fail_join_at)
+		return (EINVAL);
+	status = pthread_join(thread, result);
+	if (status != 0)
+		return (status);
+	return (0);
+}
+
+int	test_mutex_destroy(pthread_mutex_t *mutex)
+{
+	int	call;
+
+	call = g_destroy_calls++;
+	if (call == g_fail_destroy_at)
+		return (EBUSY);
+	return (pthread_mutex_destroy(mutex));
+}
+
+static void	set_config(t_config *config)
+{
+	config->number = 3;
+	config->time_to_die = 1000;
+	config->time_to_eat = 2;
+	config->time_to_sleep = 2;
+	config->must_eat = 1;
+	config->has_meal_limit = 1;
+}
+
+static int	create_failure_case(int fail_at)
+{
+	t_config	config;
+	t_table		table;
+	int			status;
+
+	set_config(&config);
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	g_create_calls = 0;
+	g_fail_create_at = fail_at;
+	g_join_calls = 0;
+	g_fail_join_at = -1;
+	g_fail_destroy_at = -1;
+	status = philo_run(&table);
+	if (status != PHILO_ERR || table.threads_started != fail_at
+		|| table.threads_joined != fail_at || g_join_calls != fail_at)
+	{
+		fprintf(stderr, "create failure at %d was not rolled back\n", fail_at);
+		return (1);
+	}
+	if (philo_table_destroy(&table) != PHILO_OK)
+		return (1);
+	return (0);
+}
+
+static int	join_failure_case(int fail_at)
+{
+	t_config		config;
+	t_table			table;
+	pthread_mutex_t	*forks;
+	int				before_destroy;
+
+	set_config(&config);
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	g_create_calls = 0;
+	g_fail_create_at = -1;
+	g_join_calls = 0;
+	g_fail_join_at = fail_at;
+	g_fail_destroy_at = -1;
+	if (philo_run(&table) != PHILO_UNSAFE || g_join_calls != config.number
+		|| table.threads_joined != config.number - 1)
+		return (1);
+	forks = table.forks;
+	before_destroy = g_destroy_calls;
+	if (philo_table_destroy(&table) != PHILO_UNSAFE || table.forks != forks
+		|| table.fork_count != config.number
+		|| g_destroy_calls != before_destroy)
+	{
+		fprintf(stderr, "unsafe table resources were released after join failure\n");
+		return (1);
+	}
+	if (pthread_join(table.philos[fail_at].thread, NULL) != 0)
+	{
+		fprintf(stderr, "failed join did not leave a joinable worker\n");
+		return (1);
+	}
+	table.destroy_safe = 1;
+	table.threads_joined++;
+	if (philo_table_destroy(&table) != PHILO_OK)
+		return (1);
+	return (0);
+}
+
+static int	destroy_failure_case(int fail_at, int remaining_forks)
+{
+	t_config	config;
+	t_table		table;
+
+	set_config(&config);
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	g_destroy_calls = 0;
+	g_fail_destroy_at = fail_at;
+	if (philo_table_destroy(&table) != PHILO_ERR
+		|| table.forks == NULL || table.fork_count != remaining_forks)
+	{
+		fprintf(stderr, "destroy failure at %d lost retryable state\n", fail_at);
+		return (1);
+	}
+	g_fail_destroy_at = -1;
+	if (philo_table_destroy(&table) != PHILO_OK || table.forks != NULL)
+		return (1);
+	return (0);
+}
+
+int	main(void)
+{
+	if (create_failure_case(0) || create_failure_case(1)
+		|| create_failure_case(2) || join_failure_case(0)
+		|| join_failure_case(1) || destroy_failure_case(0, 3)
+		|| destroy_failure_case(1, 2) || destroy_failure_case(3, 0)
+		|| destroy_failure_case(4, 0))
+		return (1);
+	puts("lifecycle failure: ok");
+	return (0);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index c81ed70..8663da4 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -125,6 +125,24 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 "$TMP_DIR/interrupted_meal" >"$TMP_DIR/interrupted_meal.out" \
 	|| fail 'interrupted meal changed completion counters'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_create=test_pthread_create \
+	-Dpthread_join=test_pthread_join \
+	-c "$ROOT_DIR/src/run.c" -o "$TMP_DIR/lifecycle_run.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_mutex_destroy=test_mutex_destroy \
+	-c "$ROOT_DIR/src/init.c" -o "$TMP_DIR/lifecycle_init.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/lifecycle_failure.c" \
+	"$ROOT_DIR/src/monitor.c" \
+	"$ROOT_DIR/src/routine.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/lifecycle_run.o" \
+	"$TMP_DIR/lifecycle_init.o" -o "$TMP_DIR/lifecycle_failure"
+run_timeout 8 "$TMP_DIR/lifecycle_failure.out" "$TMP_DIR/lifecycle_failure" \
+	|| fail 'thread lifecycle failure was not propagated safely'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'


## `test(concurrency): 철학자별 진행과 종료 로그 불변식 검증`

diff --git a/Makefile b/Makefile
index 06b7ab2..fc01ead 100644
--- a/Makefile
+++ b/Makefile
@@ -42,3 +42,4 @@ re: fclean all
 
 test: all
 	./tests/smoke.sh
+	./tests/concurrency.sh
diff --git a/tests/concurrency.sh b/tests/concurrency.sh
new file mode 100755
index 0000000..c04bd78
--- /dev/null
+++ b/tests/concurrency.sh
@@ -0,0 +1,124 @@
+#!/bin/sh
+
+set -eu
+
+ROOT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TMP_DIR=$(mktemp -d)
+
+cleanup()
+{
+	rm -rf "$TMP_DIR"
+}
+
+fail()
+{
+	printf 'concurrency: %s\n' "$1" >&2
+	exit 1
+}
+
+run_timeout()
+{
+	limit=$1
+	outfile=$2
+	shift 2
+	"$@" >"$outfile" 2>&1 &
+	pid=$!
+	(
+		sleep "$limit"
+		kill -TERM "$pid" 2>/dev/null || true
+	) &
+	guard=$!
+	set +e
+	wait "$pid"
+	status=$?
+	set -e
+	kill "$guard" 2>/dev/null || true
+	wait "$guard" 2>/dev/null || true
+	return "$status"
+}
+
+check_log()
+{
+	awk '
+		/^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$/ {
+			if (seen && $1 < previous) bad = 1
+			previous = $1
+			seen = 1
+			next
+		}
+		{ bad = 1 }
+		END { exit bad }
+	' "$1" || fail "invalid or decreasing log in $1"
+}
+
+check_progress()
+{
+	file=$1
+	count=$2
+	target=$3
+	awk -v count="$count" -v target="$target" '
+		$3 == "is" && $4 == "eating" { meals[$2]++ }
+		END {
+			for (id = 1; id <= count; id++)
+				if (meals[id] < target) exit 1
+		}
+	' "$file" || fail "a philosopher did not reach the meal target"
+}
+
+check_terminal_line()
+{
+	awk '
+		{
+			if (terminal) after = 1
+			if ($3 == "died") { deaths++; terminal = 1 }
+		}
+		END { exit after || deaths != 1 }
+	' "$1" || fail "death was not the only terminal log"
+}
+
+trap cleanup EXIT INT TERM
+
+make -C "$ROOT_DIR" >/dev/null
+
+for count in 2 5 17; do
+	output="$TMP_DIR/finite-$count.out"
+	run_timeout 6 "$output" "$ROOT_DIR/philo" "$count" 2000 5 5 4 \
+		|| fail "finite run for $count philosophers did not finish"
+	check_log "$output"
+	grep -q 'died' "$output" && fail "finite run reported a death"
+	check_progress "$output" "$count" 4
+done
+
+iteration=0
+while [ "$iteration" -lt 8 ]; do
+	output="$TMP_DIR/repeat-$iteration.out"
+	run_timeout 4 "$output" "$ROOT_DIR/philo" 7 1000 4 4 3 \
+		|| fail "repeated schedule $iteration did not finish"
+	check_log "$output"
+	grep -q 'died' "$output" && fail "repeated schedule reported a death"
+	check_progress "$output" 7 3
+	iteration=$((iteration + 1))
+done
+
+iteration=0
+while [ "$iteration" -lt 10 ]; do
+	output="$TMP_DIR/death-$iteration.out"
+	run_timeout 3 "$output" "$ROOT_DIR/philo" 5 60 80 10 \
+		|| fail "death schedule $iteration did not finish"
+	check_log "$output"
+	check_terminal_line "$output"
+	iteration=$((iteration + 1))
+done
+
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/log_terminal_race.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" -o "$TMP_DIR/log_terminal_race"
+race_out="$TMP_DIR/log-terminal-race.out"
+run_timeout 5 "$race_out" "$TMP_DIR/log_terminal_race" \
+	|| fail 'terminal logging race did not finish'
+check_log "$race_out"
+check_terminal_line "$race_out"
+
+printf 'concurrency: ok\n'
diff --git a/tests/log_terminal_race.c b/tests/log_terminal_race.c
new file mode 100644
index 0000000..da2d39b
--- /dev/null
+++ b/tests/log_terminal_race.c
@@ -0,0 +1,86 @@
+#include "philo.h"
+
+#include <pthread.h>
+#include <stdio.h>
+
+#define LOGGER_COUNT 12
+#define LOGS_PER_LOGGER 200
+
+static t_table			g_table;
+static pthread_mutex_t	g_gate_mutex = PTHREAD_MUTEX_INITIALIZER;
+static pthread_cond_t	g_gate_cond = PTHREAD_COND_INITIALIZER;
+static int				g_ready;
+static int				g_go;
+
+static void	*log_states(void *arg)
+{
+	t_philo	*philo;
+	int		i;
+
+	philo = (t_philo *)arg;
+	pthread_mutex_lock(&g_gate_mutex);
+	g_ready++;
+	pthread_cond_broadcast(&g_gate_cond);
+	while (!g_go)
+		pthread_cond_wait(&g_gate_cond, &g_gate_mutex);
+	pthread_mutex_unlock(&g_gate_mutex);
+	i = 0;
+	while (i < LOGS_PER_LOGGER)
+	{
+		philo_log(philo, "is thinking");
+		i++;
+	}
+	return (NULL);
+}
+
+static void	set_config(t_config *config)
+{
+	config->number = 1;
+	config->time_to_die = 1;
+	config->time_to_eat = 1;
+	config->time_to_sleep = 1;
+	config->must_eat = 0;
+	config->has_meal_limit = 0;
+}
+
+int	main(void)
+{
+	pthread_t	threads[LOGGER_COUNT];
+	t_config	config;
+	int			started;
+	int			i;
+
+	set_config(&config);
+	if (philo_table_init(&g_table, &config) != PHILO_OK)
+		return (1);
+	g_table.start_ms = philo_now_ms() - 100;
+	g_table.philos[0].last_meal_ms = g_table.start_ms;
+	started = 0;
+	while (started < LOGGER_COUNT)
+	{
+		if (pthread_create(&threads[started], NULL, log_states,
+				&g_table.philos[0]) != 0)
+			break ;
+		started++;
+	}
+	pthread_mutex_lock(&g_gate_mutex);
+	while (g_ready < started)
+		pthread_cond_wait(&g_gate_cond, &g_gate_mutex);
+	g_go = 1;
+	pthread_cond_broadcast(&g_gate_cond);
+	pthread_mutex_unlock(&g_gate_mutex);
+	if (started != LOGGER_COUNT || !philo_try_log_death(&g_table.philos[0]))
+	{
+		fprintf(stderr, "terminal log race could not be started\n");
+		philo_finish(&g_table);
+	}
+	i = 0;
+	while (i < started)
+	{
+		pthread_join(threads[i], NULL);
+		i++;
+	}
+	if (started != LOGGER_COUNT || philo_table_destroy(&g_table) != PHILO_OK)
+		return (1);
+	return (0);
+}


## `test(tsan): ThreadSanitizer 검증 경로 추가`

diff --git a/Makefile b/Makefile
index fc01ead..22eed0b 100644
--- a/Makefile
+++ b/Makefile
@@ -2,6 +2,8 @@ NAME := philo
 
 CC := cc
 CFLAGS := -Wall -Wextra -Werror -pthread -Iinclude
+TSAN_CC ?= $(CC)
+TSAN_REQUIRED ?= 0
 
 SRC_DIR := src
 OBJ_DIR := .obj
@@ -17,7 +19,7 @@ SRCS := \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
 
-.PHONY: all bonus clean fclean re test
+.PHONY: all bonus clean fclean re test test-tsan
 
 all: $(NAME)
 
@@ -43,3 +45,6 @@ re: fclean all
 test: all
 	./tests/smoke.sh
 	./tests/concurrency.sh
+
+test-tsan:
+	TSAN_CC="$(TSAN_CC)" TSAN_REQUIRED="$(TSAN_REQUIRED)" ./tests/tsan.sh
diff --git a/tests/tsan.sh b/tests/tsan.sh
new file mode 100755
index 0000000..3297265
--- /dev/null
+++ b/tests/tsan.sh
@@ -0,0 +1,208 @@
+#!/bin/sh
+
+set -u
+
+ROOT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
+TMP_DIR=$(mktemp -d)
+TSAN_CC=${TSAN_CC:-cc}
+TSAN_REQUIRED=${TSAN_REQUIRED:-0}
+
+cleanup()
+{
+	rm -rf "$TMP_DIR"
+}
+
+fail()
+{
+	printf 'tsan: %s\n' "$1" >&2
+	exit 1
+}
+
+skip()
+{
+	printf 'tsan: skipped (%s)\n' "$1" >&2
+	if [ "$TSAN_REQUIRED" -eq 1 ]; then
+		exit 1
+	fi
+	exit 77
+}
+
+run_timeout()
+{
+	limit=$1
+	stdout_file=$2
+	stderr_file=$3
+	shift 3
+	"$@" >"$stdout_file" 2>"$stderr_file" &
+	pid=$!
+	(
+		sleep "$limit"
+		kill -TERM "$pid" 2>/dev/null || true
+	) &
+	guard=$!
+	wait "$pid"
+	status=$?
+	kill "$guard" 2>/dev/null || true
+	wait "$guard" 2>/dev/null || true
+	return "$status"
+}
+
+check_tsan_stderr()
+{
+	if grep -q 'ThreadSanitizer' "$1"; then
+		cat "$1" >&2
+		return 1
+	fi
+	return 0
+}
+
+run_case()
+{
+	name=$1
+	shift
+	stdout_file="$TMP_DIR/$name.out"
+	stderr_file="$TMP_DIR/$name.err"
+	TSAN_OPTIONS='halt_on_error=1:exitcode=66' \
+		run_timeout 20 "$stdout_file" "$stderr_file" "$TMP_DIR/philo-tsan" "$@"
+	status=$?
+	if [ "$status" -ne 0 ]; then
+		cat "$stderr_file" >&2
+		printf 'tsan: %s workload exited with status %d\n' \
+			"$name" "$status" >&2
+		return 1
+	fi
+	check_tsan_stderr "$stderr_file"
+}
+
+check_log()
+{
+	awk '
+		/^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$/ {
+			if (seen && $1 < previous) bad = 1
+			previous = $1
+			seen = 1
+			next
+		}
+		{ bad = 1 }
+		END { exit bad || !seen }
+	' "$1" || fail "$2 workload produced an invalid log"
+}
+
+check_progress()
+{
+	awk -v count="$2" -v target="$3" '
+		$3 == "is" && $4 == "eating" { meals[$2]++ }
+		END {
+			for (id = 1; id <= count; id++)
+				if (meals[id] < target) exit 1
+		}
+	' "$1" || fail "$4 workload did not reach its meal target"
+}
+
+check_death()
+{
+	awk '
+		{
+			if (terminal) after = 1
+			if ($3 == "died") { deaths++; terminal = 1 }
+		}
+		END { exit after || deaths != 1 }
+	' "$1" || fail 'death workload did not end with exactly one death'
+}
+
+trap cleanup EXIT INT TERM
+
+case "$TSAN_REQUIRED" in
+	0|1)
+		;;
+	*)
+		fail 'TSAN_REQUIRED must be 0 or 1'
+		;;
+esac
+
+cat >"$TMP_DIR/probe.c" <<'PROBE'
+#include <pthread.h>
+
+static int	g_value;
+
+static void	*set_value(void *arg)
+{
+	(void)arg;
+	g_value = 1;
+	return (0);
+}
+
+int	main(void)
+{
+	pthread_t	thread;
+
+	if (pthread_create(&thread, 0, set_value, 0) != 0)
+		return (1);
+	if (pthread_join(thread, 0) != 0)
+		return (1);
+	return (g_value != 1);
+}
+PROBE
+
+if ! "$TSAN_CC" -Wall -Wextra -Werror -pthread -fsanitize=thread -g \
+	"$TMP_DIR/probe.c" -o "$TMP_DIR/tsan-probe" \
+	>"$TMP_DIR/probe-build.out" 2>"$TMP_DIR/probe-build.err"; then
+	cat "$TMP_DIR/probe-build.err" >&2
+	skip "$TSAN_CC cannot build a ThreadSanitizer probe"
+fi
+if [ ! -x "$TMP_DIR/tsan-probe" ]; then
+	fail "$TSAN_CC reported probe success without producing an executable"
+fi
+
+TSAN_OPTIONS='halt_on_error=1:exitcode=66' \
+	run_timeout 10 "$TMP_DIR/probe.out" "$TMP_DIR/probe.err" \
+	"$TMP_DIR/tsan-probe"
+probe_status=$?
+if [ "$probe_status" -ne 0 ]; then
+	cat "$TMP_DIR/probe.err" >&2
+	skip "ThreadSanitizer probe exited with status $probe_status"
+fi
+if ! check_tsan_stderr "$TMP_DIR/probe.err"; then
+	skip 'ThreadSanitizer probe reported a runtime error'
+fi
+
+if ! "$TSAN_CC" -Wall -Wextra -Werror -pthread -fsanitize=thread -g \
+	-I"$ROOT_DIR/include" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/main.c" \
+	"$ROOT_DIR/src/monitor.c" \
+	"$ROOT_DIR/src/parse.c" \
+	"$ROOT_DIR/src/routine.c" \
+	"$ROOT_DIR/src/run.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" -o "$TMP_DIR/philo-tsan" \
+	>"$TMP_DIR/build.out" 2>"$TMP_DIR/build.err"; then
+	cat "$TMP_DIR/build.err" >&2
+	fail 'project build failed after the ThreadSanitizer probe passed'
+fi
+if [ ! -x "$TMP_DIR/philo-tsan" ]; then
+	fail "$TSAN_CC reported project build success without producing an executable"
+fi
+
+run_case finite 7 1000 5 5 4 \
+	|| fail 'finite schedule reported a race or runtime error'
+check_log "$TMP_DIR/finite.out" finite
+if grep -q 'died' "$TMP_DIR/finite.out"; then
+	fail 'finite workload reported a death'
+fi
+check_progress "$TMP_DIR/finite.out" 7 4 finite
+
+run_case death 5 60 80 10 \
+	|| fail 'terminal schedule reported a race or runtime error'
+check_log "$TMP_DIR/death.out" death
+check_death "$TMP_DIR/death.out"
+
+run_case contention 17 2000 5 5 3 \
+	|| fail 'contention schedule reported a race or runtime error'
+check_log "$TMP_DIR/contention.out" contention
+if grep -q 'died' "$TMP_DIR/contention.out"; then
+	fail 'contention workload reported a death'
+fi
+check_progress "$TMP_DIR/contention.out" 17 3 contention
+
+printf 'tsan: ok\n'


