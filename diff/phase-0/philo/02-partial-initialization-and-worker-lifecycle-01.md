# 부분 초기화와 작업 스레드 수명주기

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


## `feat(thread): 철학자 작업 스레드 시작과 종료`

diff --git a/Makefile b/Makefile
index 621e8af..7220db3 100644
--- a/Makefile
+++ b/Makefile
@@ -12,6 +12,7 @@ SRCS := \
 	$(SRC_DIR)/monitor.c \
 	$(SRC_DIR)/parse.c \
 	$(SRC_DIR)/routine.c \
+	$(SRC_DIR)/run.c \
 	$(SRC_DIR)/state.c \
 	$(SRC_DIR)/time.c
 OBJS := $(SRCS:$(SRC_DIR)/%.c=$(OBJ_DIR)/%.o)
diff --git a/include/philo.h b/include/philo.h
index 88cc104..0a3c74b 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -49,6 +49,7 @@ struct s_table
 int	philo_parse_args(int argc, char **argv, t_config *config);
 int	philo_table_init(t_table *table, const t_config *config);
 void	philo_table_destroy(t_table *table);
+int	philo_run(t_table *table);
 void	philo_monitor(t_table *table);
 int	philo_has_ended(t_table *table);
 void	philo_finish(t_table *table);
diff --git a/src/run.c b/src/run.c
new file mode 100644
index 0000000..679b5e2
--- /dev/null
+++ b/src/run.c
@@ -0,0 +1,38 @@
+#include "philo.h"
+
+static void	join_started(t_table *table, int count)
+{
+	int	i;
+
+	i = 0;
+	while (i < count)
+	{
+		pthread_join(table->philos[i].thread, NULL);
+		i++;
+	}
+}
+
+int	philo_run(t_table *table)
+{
+	int	i;
+
+	table->start_ms = philo_now_ms();
+	i = 0;
+	while (i < table->config.number)
+	{
+		pthread_mutex_lock(&table->state_mutex);
+		table->philos[i].last_meal_ms = table->start_ms;
+		pthread_mutex_unlock(&table->state_mutex);
+		if (pthread_create(&table->philos[i].thread, NULL, philo_routine,
+				&table->philos[i]) != 0)
+		{
+			philo_finish(table);
+			join_started(table, i);
+			return (PHILO_ERR);
+		}
+		i++;
+	}
+	philo_monitor(table);
+	join_started(table, table->config.number);
+	return (PHILO_OK);
+}


## `feat(main): 입력부터 자원 정리까지 실행 흐름 연결`

diff --git a/src/main.c b/src/main.c
index 6bb7fdc..cd062a6 100644
--- a/src/main.c
+++ b/src/main.c
@@ -12,12 +12,24 @@ static void	print_usage(void)
 int	main(int argc, char **argv)
 {
 	t_config	config;
+	t_table		table;
 
 	if (philo_parse_args(argc, argv, &config) != PHILO_OK)
 	{
 		print_usage();
 		return (1);
 	}
-	(void)config;
+	if (philo_table_init(&table, &config) != PHILO_OK)
+	{
+		write(2, "Error: failed to initialize table\n", 34);
+		return (1);
+	}
+	if (philo_run(&table) != PHILO_OK)
+	{
+		philo_table_destroy(&table);
+		write(2, "Error: failed to run philosophers\n", 34);
+		return (1);
+	}
+	philo_table_destroy(&table);
 	return (0);
 }


## `fix(init): 포크 초기화 실패 시 중복 정리 방지`

diff --git a/src/init.c b/src/init.c
index 2e1671c..6082322 100644
--- a/src/init.c
+++ b/src/init.c
@@ -27,11 +27,7 @@ static int	init_forks(t_table *table, int count)
 	while (i < count)
 	{
 		if (pthread_mutex_init(&table->forks[i], NULL) != 0)
-		{
-			while (--i >= 0)
-				pthread_mutex_destroy(&table->forks[i]);
 			return (PHILO_ERR);
-		}
 		table->fork_count++;
 		i++;
 	}
@@ -65,20 +61,23 @@ int	philo_table_init(t_table *table, const t_config *config)
 
 void	philo_table_destroy(t_table *table)
 {
-	int	i;
-
 	if (table == NULL)
 		return ;
-	i = 0;
 	if (table->forks != NULL)
 	{
-		while (i < table->fork_count)
-			pthread_mutex_destroy(&table->forks[i++]);
+		while (table->fork_count > 0)
+			pthread_mutex_destroy(&table->forks[--table->fork_count]);
 	}
 	if (table->print_ready)
+	{
 		pthread_mutex_destroy(&table->print_mutex);
+		table->print_ready = 0;
+	}
 	if (table->state_ready)
+	{
 		pthread_mutex_destroy(&table->state_mutex);
+		table->state_ready = 0;
+	}
 	free(table->forks);
 	free(table->philos);
 	table->forks = NULL;


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


## `fix(lifecycle): 부분 시작과 정리 오류를 호출자에 전파`

diff --git a/include/philo.h b/include/philo.h
index 1a9629a..3959b99 100644
--- a/include/philo.h
+++ b/include/philo.h
@@ -8,6 +8,7 @@
 
 # define PHILO_OK 0
 # define PHILO_ERR 1
+# define PHILO_UNSAFE 2
 
 typedef struct s_table	t_table;
 
@@ -45,6 +46,9 @@ struct s_table
 	int				start_released;
 	int				ready_count;
 	int				run_error;
+	int				threads_started;
+	int				threads_joined;
+	int				destroy_safe;
 	pthread_mutex_t	*forks;
 	pthread_mutex_t	state_mutex;
 	pthread_cond_t	start_cond;
@@ -54,7 +58,7 @@ struct s_table
 
 int	philo_parse_args(int argc, char **argv, t_config *config);
 int	philo_table_init(t_table *table, const t_config *config);
-void	philo_table_destroy(t_table *table);
+int	philo_table_destroy(t_table *table);
 int	philo_run(t_table *table);
 void	philo_monitor(t_table *table);
 int	philo_has_ended(t_table *table);
diff --git a/src/init.c b/src/init.c
index 1dae7ef..2de07da 100644
--- a/src/init.c
+++ b/src/init.c
@@ -47,6 +47,9 @@ int	philo_table_init(t_table *table, const t_config *config)
 	table->start_released = 0;
 	table->ready_count = 0;
 	table->run_error = 0;
+	table->threads_started = 0;
+	table->threads_joined = 0;
+	table->destroy_safe = 1;
 	table->forks = malloc(sizeof(*table->forks) * config->number);
 	table->philos = malloc(sizeof(*table->philos) * config->number);
 	if (table->forks == NULL || table->philos == NULL)
@@ -66,32 +69,43 @@ int	philo_table_init(t_table *table, const t_config *config)
 	return (PHILO_OK);
 }
 
-void	philo_table_destroy(t_table *table)
+int	philo_table_destroy(t_table *table)
 {
 	if (table == NULL)
-		return ;
+		return (PHILO_ERR);
+	if (!table->destroy_safe || table->threads_joined < table->threads_started)
+		return (PHILO_UNSAFE);
 	if (table->forks != NULL)
 	{
 		while (table->fork_count > 0)
-			pthread_mutex_destroy(&table->forks[--table->fork_count]);
+		{
+			if (pthread_mutex_destroy(
+					&table->forks[table->fork_count - 1]) != 0)
+				return (PHILO_ERR);
+			table->fork_count--;
+		}
 	}
 	if (table->print_ready)
 	{
-		pthread_mutex_destroy(&table->print_mutex);
+		if (pthread_mutex_destroy(&table->print_mutex) != 0)
+			return (PHILO_ERR);
 		table->print_ready = 0;
 	}
 	if (table->start_cond_ready)
 	{
-		pthread_cond_destroy(&table->start_cond);
+		if (pthread_cond_destroy(&table->start_cond) != 0)
+			return (PHILO_ERR);
 		table->start_cond_ready = 0;
 	}
 	if (table->state_ready)
 	{
-		pthread_mutex_destroy(&table->state_mutex);
+		if (pthread_mutex_destroy(&table->state_mutex) != 0)
+			return (PHILO_ERR);
 		table->state_ready = 0;
 	}
 	free(table->forks);
 	free(table->philos);
 	table->forks = NULL;
 	table->philos = NULL;
+	return (PHILO_OK);
 }
diff --git a/src/main.c b/src/main.c
index b8545dc..395b789 100644
--- a/src/main.c
+++ b/src/main.c
@@ -22,6 +22,8 @@ int	main(int argc, char **argv)
 {
 	t_config	config;
 	t_table		table;
+	int			run_status;
+	int			cleanup_status;
 
 	if (philo_parse_args(argc, argv, &config) != PHILO_OK)
 	{
@@ -35,12 +37,24 @@ int	main(int argc, char **argv)
 		put_error("Error: failed to initialize table\n");
 		return (1);
 	}
-	if (philo_run(&table) != PHILO_OK)
+	run_status = philo_run(&table);
+	if (run_status == PHILO_UNSAFE)
 	{
-		philo_table_destroy(&table);
+		put_error("Error: worker thread could not be joined\n");
+		_exit(1);
+	}
+	cleanup_status = philo_table_destroy(&table);
+	if (run_status != PHILO_OK)
+	{
+		if (cleanup_status != PHILO_OK)
+			put_error("Error: failed to release table resources\n");
 		put_error("Error: failed to run philosophers\n");
 		return (1);
 	}
-	philo_table_destroy(&table);
+	if (cleanup_status != PHILO_OK)
+	{
+		put_error("Error: failed to release table resources\n");
+		return (1);
+	}
 	return (0);
 }
diff --git a/src/run.c b/src/run.c
index 80f9dfa..781253c 100644
--- a/src/run.c
+++ b/src/run.c
@@ -1,15 +1,24 @@
 #include "philo.h"
 
-static void	join_started(t_table *table, int count)
+static int	join_started(t_table *table, int count)
 {
 	int	i;
+	int	status;
 
 	i = 0;
+	status = PHILO_OK;
 	while (i < count)
 	{
-		pthread_join(table->philos[i].thread, NULL);
+		if (pthread_join(table->philos[i].thread, NULL) == 0)
+			table->threads_joined++;
+		else
+		{
+			table->destroy_safe = 0;
+			status = PHILO_UNSAFE;
+		}
 		i++;
 	}
+	return (status);
 }
 
 
@@ -55,6 +64,7 @@ static int	release_start(t_table *table, int should_end)
 int	philo_run(t_table *table)
 {
 	int	i;
+	int	join_status;
 
 	i = 0;
 	while (i < table->config.number)
@@ -63,17 +73,24 @@ int	philo_run(t_table *table)
 				&table->philos[i]) != 0)
 		{
 			release_start(table, 1);
-			join_started(table, i);
+			join_status = join_started(table, table->threads_started);
+			if (join_status != PHILO_OK)
+				return (join_status);
 			return (PHILO_ERR);
 		}
+		table->threads_started++;
 		i++;
 	}
 	if (release_start(table, 0) != PHILO_OK)
 	{
-		join_started(table, table->config.number);
+		join_status = join_started(table, table->threads_started);
+		if (join_status != PHILO_OK)
+			return (join_status);
 		return (PHILO_ERR);
 	}
 	philo_monitor(table);
-	join_started(table, table->config.number);
+	join_status = join_started(table, table->threads_started);
+	if (join_status != PHILO_OK)
+		return (join_status);
 	return (PHILO_OK);
 }


