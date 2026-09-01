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


## `test(main): 결합 실패 시 안전하지 않은 정리 방지`

diff --git a/tests/main_unsafe.c b/tests/main_unsafe.c
new file mode 100644
index 0000000..9590631
--- /dev/null
+++ b/tests/main_unsafe.c
@@ -0,0 +1,41 @@
+#include "philo.h"
+
+#include <stdio.h>
+#include <stdlib.h>
+#include <unistd.h>
+
+static void	normal_exit_hook(void)
+{
+	write(1, "normal exit hook\n", 17);
+}
+
+int	test_parse_args(int argc, char **argv, t_config *config)
+{
+	(void)argc;
+	(void)argv;
+	(void)config;
+	if (atexit(normal_exit_hook) != 0)
+		return (PHILO_ERR);
+	return (PHILO_OK);
+}
+
+int	test_table_init(t_table *table, const t_config *config)
+{
+	(void)table;
+	(void)config;
+	return (PHILO_OK);
+}
+
+int	test_run(t_table *table)
+{
+	(void)table;
+	printf("buffered stdio marker\n");
+	return (PHILO_UNSAFE);
+}
+
+int	test_destroy(t_table *table)
+{
+	(void)table;
+	puts("unsafe destroy called");
+	return (PHILO_OK);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 8663da4..617028f 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -143,6 +143,28 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 run_timeout 8 "$TMP_DIR/lifecycle_failure.out" "$TMP_DIR/lifecycle_failure" \
 	|| fail 'thread lifecycle failure was not propagated safely'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dphilo_parse_args=test_parse_args \
+	-Dphilo_table_init=test_table_init \
+	-Dphilo_run=test_run \
+	-Dphilo_table_destroy=test_destroy \
+	-c "$ROOT_DIR/src/main.c" -o "$TMP_DIR/main_unsafe_main.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/main_unsafe.c" "$TMP_DIR/main_unsafe_main.o" \
+	-o "$TMP_DIR/main_unsafe"
+main_unsafe_out="$TMP_DIR/main_unsafe.out"
+if "$TMP_DIR/main_unsafe" 1 2 3 4 >"$main_unsafe_out" 2>&1; then
+	fail 'unsafe join failure returned success'
+fi
+grep -q 'worker thread could not be joined' "$main_unsafe_out" \
+	|| fail 'join failure did not reach main'
+if grep -q 'unsafe destroy called' "$main_unsafe_out"; then
+	fail 'main destroyed resources after join failure'
+fi
+if grep -q 'normal exit hook\|buffered stdio marker' "$main_unsafe_out"; then
+	fail 'unsafe join failure used the normal stdio exit path'
+fi
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'
