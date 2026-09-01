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
