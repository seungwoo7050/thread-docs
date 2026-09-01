# 실행 인자 계약과 수치 안전성

## `feat(parse): 철학자 실행 인자 검증`

diff --git a/include/philo.h b/include/philo.h
new file mode 100644
index 0000000..17cd0be
--- /dev/null
+++ b/include/philo.h
@@ -0,0 +1,21 @@
+#ifndef PHILO_H
+# define PHILO_H
+
+# include <stddef.h>
+
+# define PHILO_OK 0
+# define PHILO_ERR 1
+
+typedef struct s_config
+{
+	int	number;
+	long	time_to_die;
+	long	time_to_eat;
+	long	time_to_sleep;
+	int	must_eat;
+	int	has_meal_limit;
+}	t_config;
+
+int	philo_parse_args(int argc, char **argv, t_config *config);
+
+#endif
diff --git a/src/parse.c b/src/parse.c
new file mode 100644
index 0000000..f984897
--- /dev/null
+++ b/src/parse.c
@@ -0,0 +1,57 @@
+#include "philo.h"
+
+#include <limits.h>
+
+static int	parse_positive_long(const char *text, long *out)
+{
+	long	value;
+	int		i;
+
+	if (text == NULL || text[0] == '\0')
+		return (PHILO_ERR);
+	value = 0;
+	i = 0;
+	if (text[i] == '+')
+		i++;
+	if (text[i] == '\0')
+		return (PHILO_ERR);
+	while (text[i] != '\0')
+	{
+		if (text[i] < '0' || text[i] > '9')
+			return (PHILO_ERR);
+		if (value > (LONG_MAX - (text[i] - '0')) / 10)
+			return (PHILO_ERR);
+		value = value * 10 + (text[i] - '0');
+		i++;
+	}
+	if (value <= 0)
+		return (PHILO_ERR);
+	*out = value;
+	return (PHILO_OK);
+}
+
+int	philo_parse_args(int argc, char **argv, t_config *config)
+{
+	long	value;
+
+	if (argc != 5 && argc != 6)
+		return (PHILO_ERR);
+	if (parse_positive_long(argv[1], &value) != PHILO_OK || value > 200)
+		return (PHILO_ERR);
+	config->number = (int)value;
+	if (parse_positive_long(argv[2], &config->time_to_die) != PHILO_OK)
+		return (PHILO_ERR);
+	if (parse_positive_long(argv[3], &config->time_to_eat) != PHILO_OK)
+		return (PHILO_ERR);
+	if (parse_positive_long(argv[4], &config->time_to_sleep) != PHILO_OK)
+		return (PHILO_ERR);
+	config->must_eat = 0;
+	config->has_meal_limit = (argc == 6);
+	if (argc == 6)
+	{
+		if (parse_positive_long(argv[5], &value) != PHILO_OK || value > INT_MAX)
+			return (PHILO_ERR);
+		config->must_eat = (int)value;
+	}
+	return (PHILO_OK);
+}


## `feat(cli): 입력 계약을 실행 진입점에 연결`

diff --git a/src/main.c b/src/main.c
new file mode 100644
index 0000000..6bb7fdc
--- /dev/null
+++ b/src/main.c
@@ -0,0 +1,23 @@
+#include "philo.h"
+
+#include <unistd.h>
+
+static void	print_usage(void)
+{
+	write(2, "Usage: ./philo number_of_philosophers time_to_die ", 50);
+	write(2, "time_to_eat time_to_sleep ", 26);
+	write(2, "[number_of_times_each_philosopher_must_eat]\n", 45);
+}
+
+int	main(int argc, char **argv)
+{
+	t_config	config;
+
+	if (philo_parse_args(argc, argv, &config) != PHILO_OK)
+	{
+		print_usage();
+		return (1);
+	}
+	(void)config;
+	return (0);
+}


## `fix(parse): 밀리초 인자의 상한 적용`

diff --git a/src/parse.c b/src/parse.c
index f984897..a2f35c5 100644
--- a/src/parse.c
+++ b/src/parse.c
@@ -39,11 +39,14 @@ int	philo_parse_args(int argc, char **argv, t_config *config)
 	if (parse_positive_long(argv[1], &value) != PHILO_OK || value > 200)
 		return (PHILO_ERR);
 	config->number = (int)value;
-	if (parse_positive_long(argv[2], &config->time_to_die) != PHILO_OK)
+	if (parse_positive_long(argv[2], &config->time_to_die) != PHILO_OK
+		|| config->time_to_die > INT_MAX)
 		return (PHILO_ERR);
-	if (parse_positive_long(argv[3], &config->time_to_eat) != PHILO_OK)
+	if (parse_positive_long(argv[3], &config->time_to_eat) != PHILO_OK
+		|| config->time_to_eat > INT_MAX)
 		return (PHILO_ERR);
-	if (parse_positive_long(argv[4], &config->time_to_sleep) != PHILO_OK)
+	if (parse_positive_long(argv[4], &config->time_to_sleep) != PHILO_OK
+		|| config->time_to_sleep > INT_MAX)
 		return (PHILO_ERR);
 	config->must_eat = 0;
 	config->has_meal_limit = (argc == 6);


## `fix(cli): 명령행 오류 출력 길이 계산`

diff --git a/src/main.c b/src/main.c
index cd062a6..b8545dc 100644
--- a/src/main.c
+++ b/src/main.c
@@ -1,12 +1,21 @@
 #include "philo.h"
 
+#include <stddef.h>
 #include <unistd.h>
 
-static void	print_usage(void)
+static size_t	ft_strlen(const char *text)
 {
-	write(2, "Usage: ./philo number_of_philosophers time_to_die ", 50);
-	write(2, "time_to_eat time_to_sleep ", 26);
-	write(2, "[number_of_times_each_philosopher_must_eat]\n", 45);
+	size_t	len;
+
+	len = 0;
+	while (text[len] != '\0')
+		len++;
+	return (len);
+}
+
+static void	put_error(const char *message)
+{
+	write(2, message, ft_strlen(message));
 }
 
 int	main(int argc, char **argv)
@@ -16,18 +25,20 @@ int	main(int argc, char **argv)
 
 	if (philo_parse_args(argc, argv, &config) != PHILO_OK)
 	{
-		print_usage();
+		put_error("Usage: ./philo number_of_philosophers time_to_die ");
+		put_error("time_to_eat time_to_sleep ");
+		put_error("[number_of_times_each_philosopher_must_eat]\n");
 		return (1);
 	}
 	if (philo_table_init(&table, &config) != PHILO_OK)
 	{
-		write(2, "Error: failed to initialize table\n", 34);
+		put_error("Error: failed to initialize table\n");
 		return (1);
 	}
 	if (philo_run(&table) != PHILO_OK)
 	{
 		philo_table_destroy(&table);
-		write(2, "Error: failed to run philosophers\n", 34);
+		put_error("Error: failed to run philosophers\n");
 		return (1);
 	}
 	philo_table_destroy(&table);


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
