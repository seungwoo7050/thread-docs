# 종료 상태와 로그 선형화

## `feat(log): 상태 로그의 동시 출력 보호`

diff --git a/Makefile b/Makefile
index 71fc430..3800886 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,7 @@ SRCS := \
 	$(SRC_DIR)/init.c \
 	$(SRC_DIR)/main.c \
 	$(SRC_DIR)/parse.c \
+	$(SRC_DIR)/state.c \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
 
diff --git a/include/philo.h b/include/philo.h
index c228da2..08fe420 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -49,6 +49,10 @@ struct s_table
 int	philo_parse_args(int argc, char **argv, t_config *config);
 int	philo_table_init(t_table *table, const t_config *config);
 void	philo_table_destroy(t_table *table);
+int	philo_has_ended(t_table *table);
+void	philo_finish(t_table *table);
+void	philo_log(t_philo *philo, const char *message);
+void	philo_log_death(t_philo *philo);
 long	philo_now_ms(void);
 void	philo_sleep_ms(t_table *table, long duration_ms);
 
diff --git a/src/state.c b/src/state.c
new file mode 100644
index 0000000..b32181c
--- /dev/null
+++ b/src/state.c
@@ -0,0 +1,55 @@
+#include "philo.h"
+
+#include <stdio.h>
+
+int	philo_has_ended(t_table *table)
+{
+	int	ended;
+
+	pthread_mutex_lock(&table->state_mutex);
+	ended = table->ended;
+	pthread_mutex_unlock(&table->state_mutex);
+	return (ended);
+}
+
+void	philo_finish(t_table *table)
+{
+	pthread_mutex_lock(&table->state_mutex);
+	table->ended = 1;
+	pthread_mutex_unlock(&table->state_mutex);
+}
+
+void	philo_log(t_philo *philo, const char *message)
+{
+	t_table	*table;
+	long	timestamp;
+
+	table = philo->table;
+	pthread_mutex_lock(&table->print_mutex);
+	if (!philo_has_ended(table))
+	{
+		timestamp = philo_now_ms() - table->start_ms;
+		printf("%ld %d %s\n", timestamp, philo->id, message);
+	}
+	pthread_mutex_unlock(&table->print_mutex);
+}
+
+void	philo_log_death(t_philo *philo)
+{
+	t_table	*table;
+	long	timestamp;
+	int		should_print;
+
+	table = philo->table;
+	pthread_mutex_lock(&table->state_mutex);
+	should_print = !table->ended;
+	table->ended = 1;
+	pthread_mutex_unlock(&table->state_mutex);
+	if (should_print)
+	{
+		pthread_mutex_lock(&table->print_mutex);
+		timestamp = philo_now_ms() - table->start_ms;
+		printf("%ld %d died\n", timestamp, philo->id);
+		pthread_mutex_unlock(&table->print_mutex);
+	}
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


## `test(format): 필수 상태 로그 형식 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index 32b9ce8..e2f418a 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -16,6 +16,15 @@ fail()
 	exit 1
 }
 
+check_log_format()
+{
+	awk '
+		/^[0-9]+ [1-9][0-9]* (has taken a fork|is eating|is sleeping|is thinking|died)$/ { next }
+		{ bad = 1 }
+		END { exit bad }
+	' "$1" || fail "bad log format in $1"
+}
+
 run_timeout()
 {
 	limit=$1
@@ -55,12 +64,14 @@ fi
 single_out="$TMP_DIR/single.out"
 run_timeout 2 "$single_out" "$ROOT_DIR/philo" 1 80 40 40 \
 	|| fail 'single philosopher did not exit cleanly'
+check_log_format "$single_out"
 grep -q '1 has taken a fork' "$single_out" || fail 'single philosopher missed fork log'
 grep -q '1 died' "$single_out" || fail 'single philosopher missed death log'
 
 finite_out="$TMP_DIR/finite.out"
 run_timeout 3 "$finite_out" "$ROOT_DIR/philo" 2 250 50 50 2 \
 	|| fail 'finite meal run did not exit cleanly'
+check_log_format "$finite_out"
 grep -q 'died' "$finite_out" && fail 'finite meal run had a death'
 eat_count=$(grep -c 'is eating' "$finite_out" || true)
 [ "$eat_count" -ge 4 ] || fail 'finite meal run did not eat enough'
@@ -68,6 +79,7 @@ eat_count=$(grep -c 'is eating' "$finite_out" || true)
 nodeath_out="$TMP_DIR/nodeath.out"
 run_timeout 5 "$nodeath_out" "$ROOT_DIR/philo" 5 800 100 100 3 \
 	|| fail 'no-death meal run did not exit cleanly'
+check_log_format "$nodeath_out"
 grep -q 'died' "$nodeath_out" && fail 'no-death meal run had a death'
 eat_count=$(grep -c 'is eating' "$nodeath_out" || true)
 [ "$eat_count" -ge 15 ] || fail 'no-death meal run did not reach meal count'


## `fix(monitor): 종료 상태와 사망 로그를 원자적으로 확정`

diff --git a/include/philo.h b/include/philo.h
index e01b8e8..2d329f4 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -60,7 +60,7 @@ void	philo_monitor(t_table *table);
 int	philo_has_ended(t_table *table);
 void	philo_finish(t_table *table);
 void	philo_log(t_philo *philo, const char *message);
-void	philo_log_death(t_philo *philo);
+int	philo_try_log_death(t_philo *philo);
 void	*philo_routine(void *arg);
 int64_t	philo_now_ms(void);
 void	philo_sleep_ms(t_table *table, int64_t duration_ms);
diff --git a/src/monitor.c b/src/monitor.c
index e154b2a..4c5d67d 100644
--- a/src/monitor.c
+++ b/src/monitor.c
@@ -27,23 +27,25 @@ void	philo_monitor(t_table *table)
 	t_philo	*dead;
 	int64_t	now;
 
-	while (!philo_has_ended(table))
+	while (1)
 	{
 		now = philo_now_ms();
 		pthread_mutex_lock(&table->state_mutex);
+		if (table->ended)
+		{
+			pthread_mutex_unlock(&table->state_mutex);
+			return ;
+		}
 		if (all_meals_done(table))
 		{
+			table->ended = 1;
 			pthread_mutex_unlock(&table->state_mutex);
-			philo_finish(table);
 			return ;
 		}
 		dead = find_dead_philo(table, now);
 		pthread_mutex_unlock(&table->state_mutex);
-		if (dead != NULL)
-		{
-			philo_log_death(dead);
+		if (dead != NULL && philo_try_log_death(dead))
 			return ;
-		}
 		usleep(500);
 	}
 }
diff --git a/src/state.c b/src/state.c
index 078dae2..176adba 100644
--- a/src/state.c
+++ b/src/state.c
@@ -34,22 +34,29 @@ void	philo_log(t_philo *philo, const char *message)
 	pthread_mutex_unlock(&table->print_mutex);
 }
 
-void	philo_log_death(t_philo *philo)
+int	philo_try_log_death(t_philo *philo)
 {
 	t_table	*table;
+	int64_t	now;
 	int64_t	timestamp;
 	int		should_print;
 
 	table = philo->table;
+	should_print = 0;
+	timestamp = 0;
+	pthread_mutex_lock(&table->print_mutex);
 	pthread_mutex_lock(&table->state_mutex);
-	should_print = !table->ended;
-	table->ended = 1;
+	now = philo_now_ms();
+	if (!table->ended
+		&& now - philo->last_meal_ms >= table->config.time_to_die)
+	{
+		table->ended = 1;
+		timestamp = now - table->start_ms;
+		should_print = 1;
+	}
 	pthread_mutex_unlock(&table->state_mutex);
 	if (should_print)
-	{
-		pthread_mutex_lock(&table->print_mutex);
-		timestamp = philo_now_ms() - table->start_ms;
 		printf("%lld %d died\n", (long long)timestamp, philo->id);
-		pthread_mutex_unlock(&table->print_mutex);
-	}
+	pthread_mutex_unlock(&table->print_mutex);
+	return (should_print);
 }


## `test(monitor): 완료 상태와 오래된 사망 판정 검증`

diff --git a/tests/smoke.sh b/tests/smoke.sh
index bd0fc20..882f2d8 100755
--- a/tests/smoke.sh
+++ b/tests/smoke.sh
@@ -100,6 +100,19 @@ run_timeout 5 "$TMP_DIR/worker_wait_failure.out" \
 grep -q 'worker wait failure: ok' "$TMP_DIR/worker_wait_failure.out" \
 	|| fail 'worker condition wait failure test did not finish'
 
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	-Dpthread_mutex_unlock=test_mutex_unlock \
+	-c "$ROOT_DIR/src/monitor.c" -o "$TMP_DIR/terminal_monitor.o"
+cc -Wall -Wextra -Werror -pthread -I"$ROOT_DIR/include" \
+	"$ROOT_DIR/tests/terminal_state.c" \
+	"$ROOT_DIR/src/init.c" \
+	"$ROOT_DIR/src/state.c" \
+	"$ROOT_DIR/src/time.c" \
+	"$TMP_DIR/terminal_monitor.o" -o "$TMP_DIR/terminal_state"
+"$TMP_DIR/terminal_state" >"$TMP_DIR/terminal_state.out" \
+	|| fail 'terminal state was not committed atomically'
+grep -q 'died' "$TMP_DIR/terminal_state.out" && fail 'stale death was printed'
+
 invalid_out="$TMP_DIR/invalid.out"
 if "$ROOT_DIR/philo" 0 100 10 10 >"$invalid_out" 2>&1; then
 	fail 'invalid philosopher count succeeded'
diff --git a/tests/terminal_state.c b/tests/terminal_state.c
new file mode 100644
index 0000000..de08dcf
--- /dev/null
+++ b/tests/terminal_state.c
@@ -0,0 +1,110 @@
+#include "philo.h"
+
+#include <pthread.h>
+#include <stdio.h>
+
+#define MODE_COMPLETION 1
+#define MODE_STALE_DEATH 2
+
+static t_table	*g_table;
+static int		g_mode;
+static int		g_injected;
+static int		g_ended_at_unlock;
+
+int	test_mutex_unlock(pthread_mutex_t *mutex)
+{
+	int	status;
+
+	status = pthread_mutex_unlock(mutex);
+	if (status != 0)
+		return (status);
+	if (!g_injected && mutex == &g_table->state_mutex)
+	{
+		g_injected = 1;
+		g_ended_at_unlock = g_table->ended;
+		if (g_mode == MODE_STALE_DEATH)
+		{
+			pthread_mutex_lock(&g_table->state_mutex);
+			g_table->philos[0].last_meal_ms = philo_now_ms();
+			g_table->full_count = 1;
+			pthread_mutex_unlock(&g_table->state_mutex);
+		}
+	}
+	return (0);
+}
+
+static void	set_config(t_config *config)
+{
+	config->number = 1;
+	config->time_to_die = 100;
+	config->time_to_eat = 10;
+	config->time_to_sleep = 10;
+	config->must_eat = 1;
+	config->has_meal_limit = 1;
+}
+
+static int	init_case(t_table *table, t_config *config)
+{
+	if (philo_table_init(table, config) != PHILO_OK)
+		return (1);
+	table->start_ms = philo_now_ms();
+	table->philos[0].last_meal_ms = table->start_ms;
+	return (0);
+}
+
+static int	completion_case(void)
+{
+	t_table		table;
+	t_config	config;
+
+	set_config(&config);
+	if (init_case(&table, &config))
+		return (1);
+	table.full_count = 1;
+	g_table = &table;
+	g_mode = MODE_COMPLETION;
+	g_injected = 0;
+	g_ended_at_unlock = 0;
+	philo_monitor(&table);
+	if (!g_injected || !g_ended_at_unlock || !table.ended)
+	{
+		fprintf(stderr, "meal completion was not committed while locked\n");
+		philo_table_destroy(&table);
+		return (1);
+	}
+	philo_table_destroy(&table);
+	return (0);
+}
+
+static int	stale_death_case(void)
+{
+	t_table		table;
+	t_config	config;
+
+	set_config(&config);
+	config.time_to_die = 10;
+	if (init_case(&table, &config))
+		return (1);
+	table.start_ms = philo_now_ms() - 50;
+	table.philos[0].last_meal_ms = table.start_ms;
+	g_table = &table;
+	g_mode = MODE_STALE_DEATH;
+	g_injected = 0;
+	philo_monitor(&table);
+	if (!g_injected || !table.ended || table.full_count != 1)
+	{
+		fprintf(stderr, "stale death decision was not rechecked\n");
+		philo_table_destroy(&table);
+		return (1);
+	}
+	philo_table_destroy(&table);
+	return (0);
+}
+
+int	main(void)
+{
+	if (completion_case() || stale_death_case())
+		return (1);
+	puts("terminal state: ok");
+	return (0);
+}


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
