# 식사 완료 커밋과 목표 집계

## `feat(routine): 철학자의 식사·수면·사고 흐름 구현`

diff --git a/Makefile b/Makefile
index 3800886..32900ed 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,7 @@ SRCS := \
 	$(SRC_DIR)/init.c \
 	$(SRC_DIR)/main.c \
 	$(SRC_DIR)/parse.c \
+	$(SRC_DIR)/routine.c \
 	$(SRC_DIR)/state.c \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
diff --git a/include/philo.h b/include/philo.h
index 08fe420..7427301 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -53,6 +53,7 @@ int	philo_has_ended(t_table *table);
 void	philo_finish(t_table *table);
 void	philo_log(t_philo *philo, const char *message);
 void	philo_log_death(t_philo *philo);
+void	*philo_routine(void *arg);
 long	philo_now_ms(void);
 void	philo_sleep_ms(t_table *table, long duration_ms);
 
diff --git a/src/routine.c b/src/routine.c
new file mode 100644
index 0000000..22d0605
--- /dev/null
+++ b/src/routine.c
@@ -0,0 +1,71 @@
+#include "philo.h"
+
+static void	lock_forks(t_philo *philo)
+{
+	if (philo->id % 2 == 0)
+	{
+		pthread_mutex_lock(philo->right_fork);
+		philo_log(philo, "has taken a fork");
+		pthread_mutex_lock(philo->left_fork);
+		philo_log(philo, "has taken a fork");
+	}
+	else
+	{
+		pthread_mutex_lock(philo->left_fork);
+		philo_log(philo, "has taken a fork");
+		pthread_mutex_lock(philo->right_fork);
+		philo_log(philo, "has taken a fork");
+	}
+}
+
+static void	unlock_forks(t_philo *philo)
+{
+	pthread_mutex_unlock(philo->left_fork);
+	pthread_mutex_unlock(philo->right_fork);
+}
+
+static void	record_meal_start(t_philo *philo)
+{
+	pthread_mutex_lock(&philo->table->state_mutex);
+	philo->last_meal_ms = philo_now_ms();
+	pthread_mutex_unlock(&philo->table->state_mutex);
+}
+
+static void	record_meal_done(t_philo *philo)
+{
+	t_table	*table;
+
+	table = philo->table;
+	pthread_mutex_lock(&table->state_mutex);
+	philo->meals++;
+	if (table->config.has_meal_limit && philo->meals == table->config.must_eat)
+		table->full_count++;
+	pthread_mutex_unlock(&table->state_mutex);
+}
+
+static void	eat_once(t_philo *philo)
+{
+	lock_forks(philo);
+	record_meal_start(philo);
+	philo_log(philo, "is eating");
+	philo_sleep_ms(philo->table, philo->table->config.time_to_eat);
+	record_meal_done(philo);
+	unlock_forks(philo);
+}
+
+void	*philo_routine(void *arg)
+{
+	t_philo	*philo;
+
+	philo = (t_philo *)arg;
+	if (philo->id % 2 == 0)
+		philo_sleep_ms(philo->table, 1);
+	while (!philo_has_ended(philo->table))
+	{
+		eat_once(philo);
+		philo_log(philo, "is sleeping");
+		philo_sleep_ms(philo->table, philo->table->config.time_to_sleep);
+		philo_log(philo, "is thinking");
+	}
+	return (NULL);
+}


## `feat(monitor): 사망과 식사 완료 조건 감시`

diff --git a/Makefile b/Makefile
index 32900ed..621e8af 100644
--- a/Makefile
+++ b/Makefile
@@ -9,6 +9,7 @@ OBJ_DIR := .obj
 SRCS := \
 	$(SRC_DIR)/init.c \
 	$(SRC_DIR)/main.c \
+	$(SRC_DIR)/monitor.c \
 	$(SRC_DIR)/parse.c \
 	$(SRC_DIR)/routine.c \
 	$(SRC_DIR)/state.c \
diff --git a/include/philo.h b/include/philo.h
index 7427301..88cc104 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -49,6 +49,7 @@ struct s_table
 int	philo_parse_args(int argc, char **argv, t_config *config);
 int	philo_table_init(t_table *table, const t_config *config);
 void	philo_table_destroy(t_table *table);
+void	philo_monitor(t_table *table);
 int	philo_has_ended(t_table *table);
 void	philo_finish(t_table *table);
 void	philo_log(t_philo *philo, const char *message);
diff --git a/src/monitor.c b/src/monitor.c
new file mode 100644
index 0000000..4d4eddd
--- /dev/null
+++ b/src/monitor.c
@@ -0,0 +1,49 @@
+#include "philo.h"
+
+#include <unistd.h>
+
+static int	all_meals_done(t_table *table)
+{
+	return (table->config.has_meal_limit
+		&& table->full_count >= table->config.number);
+}
+
+static t_philo	*find_dead_philo(t_table *table, long now)
+{
+	int	i;
+
+	i = 0;
+	while (i < table->config.number)
+	{
+		if (now - table->philos[i].last_meal_ms >= table->config.time_to_die)
+			return (&table->philos[i]);
+		i++;
+	}
+	return (NULL);
+}
+
+void	philo_monitor(t_table *table)
+{
+	t_philo	*dead;
+	long	now;
+
+	while (!philo_has_ended(table))
+	{
+		now = philo_now_ms();
+		pthread_mutex_lock(&table->state_mutex);
+		if (all_meals_done(table))
+		{
+			pthread_mutex_unlock(&table->state_mutex);
+			philo_finish(table);
+			return ;
+		}
+		dead = find_dead_philo(table, now);
+		pthread_mutex_unlock(&table->state_mutex);
+		if (dead != NULL)
+		{
+			philo_log_death(dead);
+			return ;
+		}
+		usleep(500);
+	}
+}


## `fix(meals): 식사 제한 도달 시 작업 루프 즉시 중단`

diff --git a/src/routine.c b/src/routine.c
index 38f0588..8629398 100644
--- a/src/routine.c
+++ b/src/routine.c
@@ -40,17 +40,25 @@ static void	record_meal_done(t_philo *philo)
 	philo->meals++;
 	if (table->config.has_meal_limit && philo->meals == table->config.must_eat)
 		table->full_count++;
+	if (table->config.has_meal_limit && table->full_count >= table->config.number)
+		table->ended = 1;
 	pthread_mutex_unlock(&table->state_mutex);
 }
 
-static void	eat_once(t_philo *philo)
+static int	eat_once(t_philo *philo)
 {
 	lock_forks(philo);
+	if (philo_has_ended(philo->table))
+	{
+		unlock_forks(philo);
+		return (PHILO_ERR);
+	}
 	record_meal_start(philo);
 	philo_log(philo, "is eating");
 	philo_sleep_ms(philo->table, philo->table->config.time_to_eat);
 	record_meal_done(philo);
 	unlock_forks(philo);
+	return (PHILO_OK);
 }
 
 static void	wait_single_philo(t_philo *philo)
@@ -75,7 +83,10 @@ void	*philo_routine(void *arg)
 		philo_sleep_ms(philo->table, 1);
 	while (!philo_has_ended(philo->table))
 	{
-		eat_once(philo);
+		if (eat_once(philo) != PHILO_OK)
+			break ;
+		if (philo_has_ended(philo->table))
+			break ;
 		philo_log(philo, "is sleeping");
 		philo_sleep_ms(philo->table, philo->table->config.time_to_sleep);
 		philo_log(philo, "is thinking");


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


## `fix(routine): 중단된 식사를 완료 횟수에서 제외`

diff --git a/include/philo.h b/include/philo.h
index 2d329f4..1a9629a 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -63,6 +63,6 @@ void	philo_log(t_philo *philo, const char *message);
 int	philo_try_log_death(t_philo *philo);
 void	*philo_routine(void *arg);
 int64_t	philo_now_ms(void);
-void	philo_sleep_ms(t_table *table, int64_t duration_ms);
+int	philo_sleep_ms(t_table *table, int64_t duration_ms);
 
 #endif
diff --git a/src/routine.c b/src/routine.c
index b300d7e..ab1383b 100644
--- a/src/routine.c
+++ b/src/routine.c
@@ -56,18 +56,24 @@ static void	record_meal_start(t_philo *philo)
 	pthread_mutex_unlock(&philo->table->state_mutex);
 }
 
-static void	record_meal_done(t_philo *philo)
+static int	record_meal_done(t_philo *philo)
 {
 	t_table	*table;
 
 	table = philo->table;
 	pthread_mutex_lock(&table->state_mutex);
+	if (table->ended)
+	{
+		pthread_mutex_unlock(&table->state_mutex);
+		return (PHILO_ERR);
+	}
 	philo->meals++;
 	if (table->config.has_meal_limit && philo->meals == table->config.must_eat)
 		table->full_count++;
 	if (table->config.has_meal_limit && table->full_count >= table->config.number)
 		table->ended = 1;
 	pthread_mutex_unlock(&table->state_mutex);
+	return (PHILO_OK);
 }
 
 static int	eat_once(t_philo *philo)
@@ -80,8 +86,12 @@ static int	eat_once(t_philo *philo)
 	}
 	record_meal_start(philo);
 	philo_log(philo, "is eating");
-	philo_sleep_ms(philo->table, philo->table->config.time_to_eat);
-	record_meal_done(philo);
+	if (philo_sleep_ms(philo->table, philo->table->config.time_to_eat)
+		!= PHILO_OK || record_meal_done(philo) != PHILO_OK)
+	{
+		unlock_forks(philo);
+		return (PHILO_ERR);
+	}
 	unlock_forks(philo);
 	return (PHILO_OK);
 }
diff --git a/src/time.c b/src/time.c
index 4c0d1df..f33b458 100644
--- a/src/time.c
+++ b/src/time.c
@@ -19,21 +19,25 @@ int64_t	philo_now_ms(void)
 	return (((int64_t)now.tv_sec * 1000) + (now.tv_nsec / 1000000));
 }
 
-void	philo_sleep_ms(t_table *table, int64_t duration_ms)
+int	philo_sleep_ms(t_table *table, int64_t duration_ms)
 {
 	int64_t	deadline;
+	int64_t	now;
 	int64_t	remaining;
 	int		ended;
 
 	deadline = philo_now_ms() + duration_ms;
-	while (philo_now_ms() < deadline)
+	while (1)
 	{
+		now = philo_now_ms();
+		if (now >= deadline)
+			return (PHILO_OK);
 		pthread_mutex_lock(&table->state_mutex);
 		ended = table->ended;
 		pthread_mutex_unlock(&table->state_mutex);
 		if (ended)
-			break ;
-		remaining = deadline - philo_now_ms();
+			return (PHILO_ERR);
+		remaining = deadline - now;
 		if (remaining > 1)
 			usleep(500);
 		else


## `test(routine): 중단된 식사의 카운터 불변식 검증`

diff --git a/tests/interrupted_meal.c b/tests/interrupted_meal.c
new file mode 100644
index 0000000..861bcec
--- /dev/null
+++ b/tests/interrupted_meal.c
@@ -0,0 +1,43 @@
+#include "philo.h"
+
+#include <stdio.h>
+
+static int	interrupted;
+
+int	test_philo_sleep_ms(t_table *table, int64_t duration_ms)
+{
+	(void)duration_ms;
+	pthread_mutex_lock(&table->state_mutex);
+	table->ended = 1;
+	pthread_mutex_unlock(&table->state_mutex);
+	interrupted = 1;
+	return (PHILO_ERR);
+}
+
+int	main(void)
+{
+	t_config	config;
+	t_table		table;
+
+	config.number = 2;
+	config.time_to_die = 100;
+	config.time_to_eat = 50;
+	config.time_to_sleep = 10;
+	config.must_eat = 1;
+	config.has_meal_limit = 1;
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	table.start_ms = philo_now_ms();
+	table.start_released = 1;
+	table.philos[0].last_meal_ms = table.start_ms;
+	philo_routine(&table.philos[0]);
+	if (!interrupted || table.philos[0].meals != 0 || table.full_count != 0)
+	{
+		fprintf(stderr, "interrupted meal changed completion counters\n");
+		philo_table_destroy(&table);
+		return (1);
+	}
+	philo_table_destroy(&table);
+	puts("interrupted meal: ok");
+	return (0);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 882f2d8..c81ed70 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -113,6 +113,18 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 	|| fail 'terminal state was not committed atomically'
 grep -q 'died' "$TMP_DIR/terminal_state.out" && fail 'stale death was printed'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dphilo_sleep_ms=test_philo_sleep_ms \
+	-c "$ROOT_DIR/src/routine.c" -o "$TMP_DIR/interrupted_routine.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/interrupted_meal.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/interrupted_routine.o" -o "$TMP_DIR/interrupted_meal"
+"$TMP_DIR/interrupted_meal" >"$TMP_DIR/interrupted_meal.out" \
+	|| fail 'interrupted meal changed completion counters'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'


## `fix(state): 식사 완료 횟수의 정수 범위 확장`

diff --git a/include/philo.h b/include/philo.h
index 3959b99..80d423a 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -25,7 +25,7 @@ typedef struct s_config
 typedef struct s_philo
 {
 	int				id;
-	int				meals;
+	int64_t			meals;
 	int64_t		last_meal_ms;
 	pthread_t		thread;
 	pthread_mutex_t	*left_fork;


## `test(routine): 최대 목표 이후 식사 카운터 검증`

diff --git a/tests/meal_counter_range.c b/tests/meal_counter_range.c
new file mode 100644
index 0000000..cfb9c9e
--- /dev/null
+++ b/tests/meal_counter_range.c
@@ -0,0 +1,57 @@
+#include "philo.h"
+
+#include <limits.h>
+#include <stdint.h>
+#include <stdio.h>
+
+static int	sleep_calls;
+
+int	test_philo_sleep_ms(t_table *table, int64_t duration_ms)
+{
+	(void)duration_ms;
+	sleep_calls++;
+	if (sleep_calls == 1)
+		return (PHILO_OK);
+	pthread_mutex_lock(&table->state_mutex);
+	table->ended = 1;
+	pthread_mutex_unlock(&table->state_mutex);
+	return (PHILO_ERR);
+}
+
+int	main(void)
+{
+	t_config	config;
+	t_table		table;
+	int64_t		expected_meals;
+	int			status;
+
+	config.number = 2;
+	config.time_to_die = 100;
+	config.time_to_eat = 50;
+	config.time_to_sleep = 10;
+	config.must_eat = INT_MAX;
+	config.has_meal_limit = 1;
+	if (philo_table_init(&table, &config) != PHILO_OK)
+		return (1);
+	table.start_ms = philo_now_ms();
+	table.start_released = 1;
+	table.full_count = 1;
+	table.philos[0].meals = INT_MAX;
+	table.philos[0].last_meal_ms = table.start_ms;
+	philo_routine(&table.philos[0]);
+	expected_meals = INT_MAX;
+	expected_meals++;
+	status = 0;
+	if (sleep_calls != 2
+		|| table.philos[0].meals != expected_meals
+		|| table.full_count != 1)
+	{
+		fprintf(stderr, "meal counter did not advance beyond INT_MAX\n");
+		status = 1;
+	}
+	if (philo_table_destroy(&table) != PHILO_OK)
+		status = 1;
+	if (status == 0)
+		puts("meal counter range: ok");
+	return (status);
+}
diff --git a/tests/smoke.sh b/tests/smoke.sh
index 617028f..3e5f6ff 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -125,6 +125,19 @@ cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 "$TMP_DIR/interrupted_meal" >"$TMP_DIR/interrupted_meal.out" \
 	|| fail 'interrupted meal changed completion counters'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dphilo_sleep_ms=test_philo_sleep_ms \
+	-c "$ROOT_DIR/src/routine.c" -o "$TMP_DIR/meal_counter_range_routine.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/meal_counter_range.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/meal_counter_range_routine.o" \
+	-o "$TMP_DIR/meal_counter_range"
+"$TMP_DIR/meal_counter_range" >"$TMP_DIR/meal_counter_range.out" \
+	|| fail 'meal counter did not advance beyond INT_MAX'
+
 cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
 	-Dpthread_create=test_pthread_create \
 	-Dpthread_join=test_pthread_join \
