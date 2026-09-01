# 단조 시간과 동기화된 시작

## `feat(time): 밀리초 시각 계산 함수 추가`

diff --git a/Makefile b/Makefile
index 58114c6..71fc430 100644
--- a/Makefile
+++ b/Makefile
@@ -9,7 +9,8 @@ OBJ_DIR := .obj
 SRCS := \
 	$(SRC_DIR)/init.c \
 	$(SRC_DIR)/main.c \
-	$(SRC_DIR)/parse.c
+	$(SRC_DIR)/parse.c \
+	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
 
 .PHONY: all bonus clean fclean re
diff --git a/include/philo.h b/include/philo.h
index eeb37db..c228da2 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -3,6 +3,7 @@
 
 # include <pthread.h>
 # include <stddef.h>
+# include <sys/time.h>
 
 # define PHILO_OK 0
 # define PHILO_ERR 1
@@ -48,5 +49,7 @@ struct s_table
 int	philo_parse_args(int argc, char **argv, t_config *config);
 int	philo_table_init(t_table *table, const t_config *config);
 void	philo_table_destroy(t_table *table);
+long	philo_now_ms(void);
+void	philo_sleep_ms(t_table *table, long duration_ms);
 
 #endif
diff --git a/src/time.c b/src/time.c
new file mode 100644
index 0000000..a16f11d
--- /dev/null
+++ b/src/time.c
@@ -0,0 +1,28 @@
+#include "philo.h"
+
+#include <unistd.h>
+
+long	philo_now_ms(void)
+{
+	struct timeval	tv;
+
+	gettimeofday(&tv, NULL);
+	return ((tv.tv_sec * 1000L) + (tv.tv_usec / 1000L));
+}
+
+void	philo_sleep_ms(t_table *table, long duration_ms)
+{
+	long	deadline;
+	int		ended;
+
+	deadline = philo_now_ms() + duration_ms;
+	while (philo_now_ms() < deadline)
+	{
+		pthread_mutex_lock(&table->state_mutex);
+		ended = table->ended;
+		pthread_mutex_unlock(&table->state_mutex);
+		if (ended)
+			break ;
+		usleep(500);
+	}
+}


## `fix(time): 짧은 대기 시간의 초과 지연 완화`

diff --git a/src/time.c b/src/time.c
index a16f11d..50cec18 100644
--- a/src/time.c
+++ b/src/time.c
@@ -13,6 +13,7 @@ long	philo_now_ms(void)
 void	philo_sleep_ms(t_table *table, long duration_ms)
 {
 	long	deadline;
+	long	remaining;
 	int		ended;
 
 	deadline = philo_now_ms() + duration_ms;
@@ -23,6 +24,10 @@ void	philo_sleep_ms(t_table *table, long duration_ms)
 		pthread_mutex_unlock(&table->state_mutex);
 		if (ended)
 			break ;
-		usleep(500);
+		remaining = deadline - philo_now_ms();
+		if (remaining > 1)
+			usleep(500);
+		else
+			usleep(100);
 	}
 }


## `fix(time): 단조 시계로 경과 시간 계산`

diff --git a/include/philo.h b/include/philo.h
index 0a3c74b..a2ffa53 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -3,7 +3,8 @@
 
 # include <pthread.h>
 # include <stddef.h>
-# include <sys/time.h>
+# include <stdint.h>
+# include <time.h>
 
 # define PHILO_OK 0
 # define PHILO_ERR 1
@@ -13,9 +14,9 @@ typedef struct s_table	t_table;
 typedef struct s_config
 {
 	int	number;
-	long	time_to_die;
-	long	time_to_eat;
-	long	time_to_sleep;
+	int64_t	time_to_die;
+	int64_t	time_to_eat;
+	int64_t	time_to_sleep;
 	int	must_eat;
 	int	has_meal_limit;
 }	t_config;
@@ -24,7 +25,7 @@ typedef struct s_philo
 {
 	int				id;
 	int				meals;
-	long			last_meal_ms;
+	int64_t		last_meal_ms;
 	pthread_t		thread;
 	pthread_mutex_t	*left_fork;
 	pthread_mutex_t	*right_fork;
@@ -34,7 +35,7 @@ typedef struct s_philo
 struct s_table
 {
 	t_config		config;
-	long			start_ms;
+	int64_t		start_ms;
 	int				ended;
 	int				full_count;
 	int				fork_count;
@@ -56,7 +57,7 @@ void	philo_finish(t_table *table);
 void	philo_log(t_philo *philo, const char *message);
 void	philo_log_death(t_philo *philo);
 void	*philo_routine(void *arg);
-long	philo_now_ms(void);
-void	philo_sleep_ms(t_table *table, long duration_ms);
+int64_t	philo_now_ms(void);
+void	philo_sleep_ms(t_table *table, int64_t duration_ms);
 
 #endif
diff --git a/src/monitor.c b/src/monitor.c
index 4d4eddd..e154b2a 100644
--- a/src/monitor.c
+++ b/src/monitor.c
@@ -8,7 +8,7 @@ static int	all_meals_done(t_table *table)
 		&& table->full_count >= table->config.number);
 }
 
-static t_philo	*find_dead_philo(t_table *table, long now)
+static t_philo	*find_dead_philo(t_table *table, int64_t now)
 {
 	int	i;
 
@@ -25,7 +25,7 @@ static t_philo	*find_dead_philo(t_table *table, long now)
 void	philo_monitor(t_table *table)
 {
 	t_philo	*dead;
-	long	now;
+	int64_t	now;
 
 	while (!philo_has_ended(table))
 	{
diff --git a/src/parse.c b/src/parse.c
index a2f35c5..3859186 100644
--- a/src/parse.c
+++ b/src/parse.c
@@ -2,9 +2,9 @@
 
 #include <limits.h>
 
-static int	parse_positive_long(const char *text, long *out)
+static int	parse_positive_i64(const char *text, int64_t *out)
 {
-	long	value;
+	int64_t	value;
 	int		i;
 
 	if (text == NULL || text[0] == '\0')
@@ -19,7 +19,7 @@ static int	parse_positive_long(const char *text, long *out)
 	{
 		if (text[i] < '0' || text[i] > '9')
 			return (PHILO_ERR);
-		if (value > (LONG_MAX - (text[i] - '0')) / 10)
+		if (value > (INT64_MAX - (text[i] - '0')) / 10)
 			return (PHILO_ERR);
 		value = value * 10 + (text[i] - '0');
 		i++;
@@ -32,27 +32,27 @@ static int	parse_positive_long(const char *text, long *out)
 
 int	philo_parse_args(int argc, char **argv, t_config *config)
 {
-	long	value;
+	int64_t	value;
 
 	if (argc != 5 && argc != 6)
 		return (PHILO_ERR);
-	if (parse_positive_long(argv[1], &value) != PHILO_OK || value > 200)
+	if (parse_positive_i64(argv[1], &value) != PHILO_OK || value > 200)
 		return (PHILO_ERR);
 	config->number = (int)value;
-	if (parse_positive_long(argv[2], &config->time_to_die) != PHILO_OK
+	if (parse_positive_i64(argv[2], &config->time_to_die) != PHILO_OK
 		|| config->time_to_die > INT_MAX)
 		return (PHILO_ERR);
-	if (parse_positive_long(argv[3], &config->time_to_eat) != PHILO_OK
+	if (parse_positive_i64(argv[3], &config->time_to_eat) != PHILO_OK
 		|| config->time_to_eat > INT_MAX)
 		return (PHILO_ERR);
-	if (parse_positive_long(argv[4], &config->time_to_sleep) != PHILO_OK
+	if (parse_positive_i64(argv[4], &config->time_to_sleep) != PHILO_OK
 		|| config->time_to_sleep > INT_MAX)
 		return (PHILO_ERR);
 	config->must_eat = 0;
 	config->has_meal_limit = (argc == 6);
 	if (argc == 6)
 	{
-		if (parse_positive_long(argv[5], &value) != PHILO_OK || value > INT_MAX)
+		if (parse_positive_i64(argv[5], &value) != PHILO_OK || value > INT_MAX)
 			return (PHILO_ERR);
 		config->must_eat = (int)value;
 	}
diff --git a/src/state.c b/src/state.c
index b32181c..078dae2 100644
--- a/src/state.c
+++ b/src/state.c
@@ -22,14 +22,14 @@ void	philo_finish(t_table *table)
 void	philo_log(t_philo *philo, const char *message)
 {
 	t_table	*table;
-	long	timestamp;
+	int64_t	timestamp;
 
 	table = philo->table;
 	pthread_mutex_lock(&table->print_mutex);
 	if (!philo_has_ended(table))
 	{
 		timestamp = philo_now_ms() - table->start_ms;
-		printf("%ld %d %s\n", timestamp, philo->id, message);
+		printf("%lld %d %s\n", (long long)timestamp, philo->id, message);
 	}
 	pthread_mutex_unlock(&table->print_mutex);
 }
@@ -37,7 +37,7 @@ void	philo_log(t_philo *philo, const char *message)
 void	philo_log_death(t_philo *philo)
 {
 	t_table	*table;
-	long	timestamp;
+	int64_t	timestamp;
 	int		should_print;
 
 	table = philo->table;
@@ -49,7 +49,7 @@ void	philo_log_death(t_philo *philo)
 	{
 		pthread_mutex_lock(&table->print_mutex);
 		timestamp = philo_now_ms() - table->start_ms;
-		printf("%ld %d died\n", timestamp, philo->id);
+		printf("%lld %d died\n", (long long)timestamp, philo->id);
 		pthread_mutex_unlock(&table->print_mutex);
 	}
 }
diff --git a/src/time.c b/src/time.c
index 50cec18..4c0d1df 100644
--- a/src/time.c
+++ b/src/time.c
@@ -2,18 +2,27 @@
 
 #include <unistd.h>
 
-long	philo_now_ms(void)
+static void	clock_failure(void)
 {
-	struct timeval	tv;
+	static const char	message[] = "Error: monotonic clock unavailable\n";
 
-	gettimeofday(&tv, NULL);
-	return ((tv.tv_sec * 1000L) + (tv.tv_usec / 1000L));
+	(void)write(2, message, sizeof(message) - 1);
+	_exit(PHILO_ERR);
 }
 
-void	philo_sleep_ms(t_table *table, long duration_ms)
+int64_t	philo_now_ms(void)
 {
-	long	deadline;
-	long	remaining;
+	struct timespec	now;
+
+	if (clock_gettime(CLOCK_MONOTONIC, &now) != 0)
+		clock_failure();
+	return (((int64_t)now.tv_sec * 1000) + (now.tv_nsec / 1000000));
+}
+
+void	philo_sleep_ms(t_table *table, int64_t duration_ms)
+{
+	int64_t	deadline;
+	int64_t	remaining;
 	int		ended;
 
 	deadline = philo_now_ms() + duration_ms;


## `test(time): 단조 시계와 시계 실패 경로 검증`

diff --git a/tests/monotonic_clock.c b/tests/monotonic_clock.c
new file mode 100644
index 0000000..2e96235
--- /dev/null
+++ b/tests/monotonic_clock.c
@@ -0,0 +1,49 @@
+#include "philo.h"
+
+#include <stdio.h>
+#include <sys/wait.h>
+#include <time.h>
+#include <unistd.h>
+
+static int	used_monotonic_clock;
+static int	fail_clock;
+
+int	test_clock_gettime(clockid_t clock_id, struct timespec *now)
+{
+	if (fail_clock)
+		return (-1);
+	if (clock_id == CLOCK_MONOTONIC)
+		used_monotonic_clock = 1;
+	now->tv_sec = 12;
+	now->tv_nsec = 345000000L;
+	return (0);
+}
+
+int	main(void)
+{
+	pid_t	pid;
+	int		status;
+
+	if (philo_now_ms() != 12345L || !used_monotonic_clock)
+	{
+		fprintf(stderr, "elapsed time did not use CLOCK_MONOTONIC\n");
+		return (1);
+	}
+	pid = fork();
+	if (pid < 0)
+		return (1);
+	if (pid == 0)
+	{
+		fail_clock = 1;
+		(void)philo_now_ms();
+		_exit(99);
+	}
+	if (waitpid(pid, &status, 0) != pid || !WIFEXITED(status)
+		|| WEXITSTATUS(status) != PHILO_ERR)
+	{
+		fprintf(stderr, "clock failure did not stop the process\n");
+		return (1);
+	}
+	puts("monotonic clock: ok");
+	return (0);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 6ecc31f..8b9ace9 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -59,6 +59,14 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 	-o "$TMP_DIR/init_failure"
 "$TMP_DIR/init_failure" || fail 'partial mutex initialization cleanup failed'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dclock_gettime=test_clock_gettime \
+	-c "$ROOT_DIR/src/time.c" -o "$TMP_DIR/monotonic_time.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/monotonic_clock.c" "$TMP_DIR/monotonic_time.o" \
+	-o "$TMP_DIR/monotonic_clock"
+"$TMP_DIR/monotonic_clock" || fail 'elapsed time did not use a monotonic clock'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'


## `fix(thread): 시작 장벽으로 기준 시각 통일`

diff --git a/include/philo.h b/include/philo.h
index a2ffa53..e01b8e8 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -40,9 +40,14 @@ struct s_table
 	int				full_count;
 	int				fork_count;
 	int				state_ready;
+	int				start_cond_ready;
 	int				print_ready;
+	int				start_released;
+	int				ready_count;
+	int				run_error;
 	pthread_mutex_t	*forks;
 	pthread_mutex_t	state_mutex;
+	pthread_cond_t	start_cond;
 	pthread_mutex_t	print_mutex;
 	t_philo			*philos;
 };
diff --git a/src/init.c b/src/init.c
index 6082322..1dae7ef 100644
--- a/src/init.c
+++ b/src/init.c
@@ -42,7 +42,11 @@ int	philo_table_init(t_table *table, const t_config *config)
 	table->full_count = 0;
 	table->fork_count = 0;
 	table->state_ready = 0;
+	table->start_cond_ready = 0;
 	table->print_ready = 0;
+	table->start_released = 0;
+	table->ready_count = 0;
+	table->run_error = 0;
 	table->forks = malloc(sizeof(*table->forks) * config->number);
 	table->philos = malloc(sizeof(*table->philos) * config->number);
 	if (table->forks == NULL || table->philos == NULL)
@@ -50,6 +54,9 @@ int	philo_table_init(t_table *table, const t_config *config)
 	if (pthread_mutex_init(&table->state_mutex, NULL) != 0)
 		return (philo_table_destroy(table), PHILO_ERR);
 	table->state_ready = 1;
+	if (pthread_cond_init(&table->start_cond, NULL) != 0)
+		return (philo_table_destroy(table), PHILO_ERR);
+	table->start_cond_ready = 1;
 	if (pthread_mutex_init(&table->print_mutex, NULL) != 0)
 		return (philo_table_destroy(table), PHILO_ERR);
 	table->print_ready = 1;
@@ -73,6 +80,11 @@ void	philo_table_destroy(t_table *table)
 		pthread_mutex_destroy(&table->print_mutex);
 		table->print_ready = 0;
 	}
+	if (table->start_cond_ready)
+	{
+		pthread_cond_destroy(&table->start_cond);
+		table->start_cond_ready = 0;
+	}
 	if (table->state_ready)
 	{
 		pthread_mutex_destroy(&table->state_mutex);
diff --git a/src/routine.c b/src/routine.c
index 8629398..b300d7e 100644
--- a/src/routine.c
+++ b/src/routine.c
@@ -1,5 +1,30 @@
 #include "philo.h"
 
+static int	wait_for_start(t_philo *philo)
+{
+	t_table	*table;
+	int		ended;
+
+	table = philo->table;
+	pthread_mutex_lock(&table->state_mutex);
+	table->ready_count++;
+	pthread_cond_broadcast(&table->start_cond);
+	while (!table->start_released)
+	{
+		if (pthread_cond_wait(&table->start_cond,
+				&table->state_mutex) != 0)
+		{
+			table->run_error = 1;
+			table->ended = 1;
+			table->start_released = 1;
+			pthread_cond_broadcast(&table->start_cond);
+		}
+	}
+	ended = table->ended;
+	pthread_mutex_unlock(&table->state_mutex);
+	return (ended);
+}
+
 static void	lock_forks(t_philo *philo)
 {
 	if (philo->id % 2 == 0)
@@ -74,6 +99,8 @@ void	*philo_routine(void *arg)
 	t_philo	*philo;
 
 	philo = (t_philo *)arg;
+	if (wait_for_start(philo))
+		return (NULL);
 	if (philo->table->config.number == 1)
 	{
 		wait_single_philo(philo);
diff --git a/src/run.c b/src/run.c
index 679b5e2..80f9dfa 100644
--- a/src/run.c
+++ b/src/run.c
@@ -12,26 +12,67 @@ static void	join_started(t_table *table, int count)
 	}
 }
 
+
+static int	release_start(t_table *table, int should_end)
+{
+	int		i;
+	int		status;
+	int64_t	start_ms;
+
+	status = PHILO_OK;
+	pthread_mutex_lock(&table->state_mutex);
+	while (!should_end && table->ready_count < table->config.number)
+	{
+		if (pthread_cond_wait(&table->start_cond,
+				&table->state_mutex) != 0)
+		{
+			table->run_error = 1;
+			should_end = 1;
+			status = PHILO_ERR;
+		}
+	}
+	if (table->run_error)
+	{
+		should_end = 1;
+		status = PHILO_ERR;
+	}
+	start_ms = philo_now_ms();
+	table->start_ms = start_ms;
+	i = 0;
+	while (i < table->config.number)
+	{
+		table->philos[i].last_meal_ms = start_ms;
+		i++;
+	}
+	if (should_end)
+		table->ended = 1;
+	table->start_released = 1;
+	pthread_cond_broadcast(&table->start_cond);
+	pthread_mutex_unlock(&table->state_mutex);
+	return (status);
+}
+
 int	philo_run(t_table *table)
 {
 	int	i;
 
-	table->start_ms = philo_now_ms();
 	i = 0;
 	while (i < table->config.number)
 	{
-		pthread_mutex_lock(&table->state_mutex);
-		table->philos[i].last_meal_ms = table->start_ms;
-		pthread_mutex_unlock(&table->state_mutex);
 		if (pthread_create(&table->philos[i].thread, NULL, philo_routine,
 				&table->philos[i]) != 0)
 		{
-			philo_finish(table);
+			release_start(table, 1);
 			join_started(table, i);
 			return (PHILO_ERR);
 		}
 		i++;
 	}
+	if (release_start(table, 0) != PHILO_OK)
+	{
+		join_started(table, table->config.number);
+		return (PHILO_ERR);
+	}
 	philo_monitor(table);
 	join_started(table, table->config.number);
 	return (PHILO_OK);


## `test(thread): 지연된 작업자의 공통 시작 시각 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 8b9ace9..eb4a2d5 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -67,6 +67,21 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 	-o "$TMP_DIR/monotonic_clock"
 "$TMP_DIR/monotonic_clock" || fail 'elapsed time did not use a monotonic clock'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_create=test_pthread_create \
+	-c "$ROOT_DIR/src/run.c" -o "$TMP_DIR/start_barrier_run.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/start_barrier.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/monitor.c" \
+	"$ROOT_DIR/src/routine.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/start_barrier_run.o" \
+	-o "$TMP_DIR/start_barrier"
+"$TMP_DIR/start_barrier" >"$TMP_DIR/start_barrier.out" \
+	|| fail 'workers did not share one release timestamp'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'
diff --git a/tests/start_barrier.c b/tests/start_barrier.c
new file mode 100644
index 0000000..5bf5202
--- /dev/null
+++ b/tests/start_barrier.c
@@ -0,0 +1,81 @@
+#include "philo.h"
+
+#include <pthread.h>
+#include <stdio.h>
+#include <unistd.h>
+
+typedef struct s_delayed_start
+{
+	void	*(*routine)(void *);
+	void	*arg;
+	useconds_t	delay_us;
+}	t_delayed_start;
+
+static pthread_mutex_t	g_gate_mutex = PTHREAD_MUTEX_INITIALIZER;
+static pthread_cond_t	g_gate_cond = PTHREAD_COND_INITIALIZER;
+static t_delayed_start	g_starts[8];
+static int				g_created;
+static int				g_released;
+
+static void	*delayed_start(void *arg)
+{
+	t_delayed_start	*start;
+
+	start = (t_delayed_start *)arg;
+	pthread_mutex_lock(&g_gate_mutex);
+	while (!g_released)
+		pthread_cond_wait(&g_gate_cond, &g_gate_mutex);
+	pthread_mutex_unlock(&g_gate_mutex);
+	usleep(start->delay_us);
+	return (start->routine(start->arg));
+}
+
+int	test_pthread_create(pthread_t *thread, const pthread_attr_t *attr,
+		void *(*routine)(void *), void *arg)
+{
+	int	index;
+	int	status;
+
+	index = g_created;
+	g_starts[index].routine = routine;
+	g_starts[index].arg = arg;
+	g_starts[index].delay_us = (index == 4) * 150000;
+	status = pthread_create(thread, attr, delayed_start, &g_starts[index]);
+	if (status != 0)
+		return (status);
+	g_created++;
+	usleep(30000);
+	if (g_created == 5)
+	{
+		pthread_mutex_lock(&g_gate_mutex);
+		g_released = 1;
+		pthread_cond_broadcast(&g_gate_cond);
+		pthread_mutex_unlock(&g_gate_mutex);
+	}
+	return (0);
+}
+
+int	main(void)
+{
+	t_config	config;
+	t_table		table;
+
+	config.number = 5;
+	config.time_to_die = 80;
+	config.time_to_eat = 5;
+	config.time_to_sleep = 5;
+	config.must_eat = 1;
+	config.has_meal_limit = 1;
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	if (philo_run(&table) != PHILO_OK || table.full_count != config.number
+		|| table.ready_count != config.number)
+	{
+		fprintf(stderr, "workers did not share one release timestamp\n");
+		philo_table_destroy(&table);
+		return (1);
+	}
+	philo_table_destroy(&table);
+	puts("start barrier: ok");
+	return (0);
+}


## `test(thread): 시작 대기 실패 전파 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index eb4a2d5..bd0fc20 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -82,6 +82,24 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 "$TMP_DIR/start_barrier" >"$TMP_DIR/start_barrier.out" \
 	|| fail 'workers did not share one release timestamp'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_cond_wait=test_pthread_cond_wait \
+	-c "$ROOT_DIR/src/routine.c" -o "$TMP_DIR/worker_wait_routine.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/worker_wait_failure.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/monitor.c" \
+	"$ROOT_DIR/src/run.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/worker_wait_routine.o" \
+	-o "$TMP_DIR/worker_wait_failure"
+run_timeout 5 "$TMP_DIR/worker_wait_failure.out" \
+	"$TMP_DIR/worker_wait_failure" \
+	|| fail 'worker condition wait failure was not propagated'
+grep -q 'worker wait failure: ok' "$TMP_DIR/worker_wait_failure.out" \
+	|| fail 'worker condition wait failure test did not finish'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'
diff --git a/tests/worker_wait_failure.c b/tests/worker_wait_failure.c
new file mode 100644
index 0000000..8655ace
--- /dev/null
+++ b/tests/worker_wait_failure.c
@@ -0,0 +1,53 @@
+#include "philo.h"
+
+#include <errno.h>
+#include <pthread.h>
+#include <stdio.h>
+
+static pthread_mutex_t	g_failure_mutex = PTHREAD_MUTEX_INITIALIZER;
+static int				g_failed;
+
+int	test_pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex)
+{
+	int	should_fail;
+
+	pthread_mutex_lock(&g_failure_mutex);
+	should_fail = !g_failed;
+	if (should_fail)
+		g_failed = 1;
+	pthread_mutex_unlock(&g_failure_mutex);
+	if (should_fail)
+		return (EINVAL);
+	return (pthread_cond_wait(cond, mutex));
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
+int	main(void)
+{
+	t_config	config;
+	t_table		table;
+	int			status;
+
+	set_config(&config);
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	g_failed = 0;
+	status = philo_run(&table);
+	if (!g_failed || status != PHILO_ERR || !table.run_error)
+	{
+		fprintf(stderr, "worker wait failure returned success\n");
+		return (1);
+	}
+	philo_table_destroy(&table);
+	puts("worker wait failure: ok");
+	return (0);
+}


