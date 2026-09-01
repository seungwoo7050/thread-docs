# 포크 소유권과 작업 루틴

## `feat(init): 테이블 저장소와 철학자 관계 초기화`

diff --git a/Makefile b/Makefile
index ec6fd3b..58114c6 100644
--- a/Makefile
+++ b/Makefile
@@ -7,6 +7,7 @@ SRC_DIR := src
 OBJ_DIR := .obj
 
 SRCS := \
+	$(SRC_DIR)/init.c \
 	$(SRC_DIR)/main.c \
 	$(SRC_DIR)/parse.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
diff --git a/include/philo.h b/include/philo.h
index 17cd0be..eeb37db 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -1,11 +1,14 @@
 #ifndef PHILO_H
 # define PHILO_H
 
+# include <pthread.h>
 # include <stddef.h>
 
 # define PHILO_OK 0
 # define PHILO_ERR 1
 
+typedef struct s_table	t_table;
+
 typedef struct s_config
 {
 	int	number;
@@ -16,6 +19,34 @@ typedef struct s_config
 	int	has_meal_limit;
 }	t_config;
 
+typedef struct s_philo
+{
+	int				id;
+	int				meals;
+	long			last_meal_ms;
+	pthread_t		thread;
+	pthread_mutex_t	*left_fork;
+	pthread_mutex_t	*right_fork;
+	t_table			*table;
+}	t_philo;
+
+struct s_table
+{
+	t_config		config;
+	long			start_ms;
+	int				ended;
+	int				full_count;
+	int				fork_count;
+	int				state_ready;
+	int				print_ready;
+	pthread_mutex_t	*forks;
+	pthread_mutex_t	state_mutex;
+	pthread_mutex_t	print_mutex;
+	t_philo			*philos;
+};
+
 int	philo_parse_args(int argc, char **argv, t_config *config);
+int	philo_table_init(t_table *table, const t_config *config);
+void	philo_table_destroy(t_table *table);
 
 #endif
diff --git a/src/init.c b/src/init.c
new file mode 100644
index 0000000..ffb50af
--- /dev/null
+++ b/src/init.c
@@ -0,0 +1,47 @@
+#include "philo.h"
+
+#include <stdlib.h>
+
+static void	assign_philos(t_table *table)
+{
+	int	i;
+
+	i = 0;
+	while (i < table->config.number)
+	{
+		table->philos[i].id = i + 1;
+		table->philos[i].meals = 0;
+		table->philos[i].last_meal_ms = 0;
+		table->philos[i].left_fork = &table->forks[i];
+		table->philos[i].right_fork = &table->forks[(i + 1) % table->config.number];
+		table->philos[i].table = table;
+		i++;
+	}
+}
+
+int	philo_table_init(t_table *table, const t_config *config)
+{
+	table->config = *config;
+	table->start_ms = 0;
+	table->ended = 0;
+	table->full_count = 0;
+	table->fork_count = 0;
+	table->state_ready = 0;
+	table->print_ready = 0;
+	table->forks = malloc(sizeof(*table->forks) * config->number);
+	table->philos = malloc(sizeof(*table->philos) * config->number);
+	if (table->forks == NULL || table->philos == NULL)
+		return (philo_table_destroy(table), PHILO_ERR);
+	assign_philos(table);
+	return (PHILO_OK);
+}
+
+void	philo_table_destroy(t_table *table)
+{
+	if (table == NULL)
+		return ;
+	free(table->forks);
+	free(table->philos);
+	table->forks = NULL;
+	table->philos = NULL;
+}


## `feat(init): 뮤텍스 수명주기와 실패 롤백 구현`

diff --git a/src/init.c b/src/init.c
index ffb50af..2e1671c 100644
--- a/src/init.c
+++ b/src/init.c
@@ -19,6 +19,25 @@ static void	assign_philos(t_table *table)
 	}
 }
 
+static int	init_forks(t_table *table, int count)
+{
+	int	i;
+
+	i = 0;
+	while (i < count)
+	{
+		if (pthread_mutex_init(&table->forks[i], NULL) != 0)
+		{
+			while (--i >= 0)
+				pthread_mutex_destroy(&table->forks[i]);
+			return (PHILO_ERR);
+		}
+		table->fork_count++;
+		i++;
+	}
+	return (PHILO_OK);
+}
+
 int	philo_table_init(t_table *table, const t_config *config)
 {
 	table->config = *config;
@@ -32,14 +51,34 @@ int	philo_table_init(t_table *table, const t_config *config)
 	table->philos = malloc(sizeof(*table->philos) * config->number);
 	if (table->forks == NULL || table->philos == NULL)
 		return (philo_table_destroy(table), PHILO_ERR);
+	if (pthread_mutex_init(&table->state_mutex, NULL) != 0)
+		return (philo_table_destroy(table), PHILO_ERR);
+	table->state_ready = 1;
+	if (pthread_mutex_init(&table->print_mutex, NULL) != 0)
+		return (philo_table_destroy(table), PHILO_ERR);
+	table->print_ready = 1;
+	if (init_forks(table, config->number) != PHILO_OK)
+		return (philo_table_destroy(table), PHILO_ERR);
 	assign_philos(table);
 	return (PHILO_OK);
 }
 
 void	philo_table_destroy(t_table *table)
 {
+	int	i;
+
 	if (table == NULL)
 		return ;
+	i = 0;
+	if (table->forks != NULL)
+	{
+		while (i < table->fork_count)
+			pthread_mutex_destroy(&table->forks[i++]);
+	}
+	if (table->print_ready)
+		pthread_mutex_destroy(&table->print_mutex);
+	if (table->state_ready)
+		pthread_mutex_destroy(&table->state_mutex);
 	free(table->forks);
 	free(table->philos);
 	table->forks = NULL;


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


## `fix(single): 철학자가 한 명일 때 포크 재잠금 방지`

diff --git a/src/routine.c b/src/routine.c
index 22d0605..38f0588 100644
--- a/src/routine.c
+++ b/src/routine.c
@@ -53,11 +53,24 @@ static void	eat_once(t_philo *philo)
 	unlock_forks(philo);
 }
 
+static void	wait_single_philo(t_philo *philo)
+{
+	pthread_mutex_lock(philo->left_fork);
+	philo_log(philo, "has taken a fork");
+	philo_sleep_ms(philo->table, philo->table->config.time_to_die + 1);
+	pthread_mutex_unlock(philo->left_fork);
+}
+
 void	*philo_routine(void *arg)
 {
 	t_philo	*philo;
 
 	philo = (t_philo *)arg;
+	if (philo->table->config.number == 1)
+	{
+		wait_single_philo(philo);
+		return (NULL);
+	}
 	if (philo->id % 2 == 0)
 		philo_sleep_ms(philo->table, 1);
 	while (!philo_has_ended(philo->table))


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
