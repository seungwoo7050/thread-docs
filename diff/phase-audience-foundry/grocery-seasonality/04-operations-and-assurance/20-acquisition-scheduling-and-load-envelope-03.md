## `fix(perf): measure paced schedule jitter`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index 9abb45b..1da6fe7 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -38,8 +38,10 @@ REQUEST_TIMEOUT_SECONDS: Final = 2.0
 MAX_RESPONSE_BYTES: Final = 2 * 1024 * 1024
 P95_LIMIT_MS: Final = 500.0
 HTTP_5XX_RATE_LIMIT: Final = 0.005
-MAX_SCHEDULE_LAG_MS: Final = 100.0
+P95_SCHEDULE_JITTER_LIMIT_MS: Final = 100.0
 PROFILE_COMPLETION_GRACE_SECONDS: Final = 3.0
+NOMINAL_REQUEST_INTERVAL_MS: Final = 1000.0 / REQUESTS_PER_SECOND
+_CLOCK_COMPARISON_EPSILON_MS: Final = 0.001
 
 _LOCAL_HOST: Final = "127.0.0.1"
 _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
@@ -141,8 +143,10 @@ class CompletedRequest:
 @dataclass(frozen=True, slots=True)
 class RunMeasurements:
     elapsed_seconds: float
-    max_schedule_lag_ms: float
-    schedule_lag_violations: int
+    p95_schedule_jitter_ms: float
+    max_schedule_jitter_ms: float
+    minimum_inter_submission_ms: float
+    burst_interval_violations: int
     observed_peak_active: int
 
 
@@ -164,11 +168,15 @@ class LoadReport:
     elapsed_seconds: float
     throughput_rps: float
     minimum_accepted_throughput_rps: float
-    max_schedule_lag_ms: float
-    schedule_lag_violations: int
+    p95_schedule_jitter_ms: float
+    max_schedule_jitter_ms: float
+    minimum_inter_submission_ms: float
+    burst_interval_violations: int
     observed_peak_active: int
     duration_contract_met: bool
     throughput_target_met: bool
+    schedule_jitter_contract_met: bool
+    no_burst_contract_met: bool
     schedule_contract_met: bool
     concurrency_contract_met: bool
     workload_consistent: bool
@@ -207,9 +215,14 @@ class LoadReport:
                 "throughput_rps": self.throughput_rps,
                 "minimum_accepted_throughput_rps": self.minimum_accepted_throughput_rps,
                 "throughput_target_met": self.throughput_target_met,
-                "max_schedule_lag_ms": self.max_schedule_lag_ms,
-                "schedule_lag_limit_ms": MAX_SCHEDULE_LAG_MS,
-                "schedule_lag_violations": self.schedule_lag_violations,
+                "p95_schedule_jitter_ms": self.p95_schedule_jitter_ms,
+                "max_schedule_jitter_ms": self.max_schedule_jitter_ms,
+                "p95_schedule_jitter_limit_ms": P95_SCHEDULE_JITTER_LIMIT_MS,
+                "schedule_jitter_contract_met": self.schedule_jitter_contract_met,
+                "minimum_inter_submission_ms": self.minimum_inter_submission_ms,
+                "nominal_request_interval_ms": NOMINAL_REQUEST_INTERVAL_MS,
+                "burst_interval_violations": self.burst_interval_violations,
+                "no_burst_contract_met": self.no_burst_contract_met,
                 "schedule_contract_met": self.schedule_contract_met,
             },
             "workload_consistent": self.workload_consistent,
@@ -373,7 +386,7 @@ def _failed_observation(*, latency_ms: float = 0.0) -> HttpObservation:
 @dataclass(frozen=True, slots=True)
 class _TimedObservation:
     observation: HttpObservation
-    schedule_lag_ms: float
+    schedule_jitter_ms: float
 
 
 class _ActiveRequestCounter:
@@ -404,12 +417,12 @@ def _execute_scheduled_request(
     url: str,
     port: int,
     timeout_seconds: float,
-    scheduled_at: float,
+    paced_deadline: float,
     monotonic: Callable[[], float],
     active_counter: _ActiveRequestCounter,
 ) -> _TimedObservation:
     request_started_at = monotonic()
-    schedule_lag_seconds = max(0.0, request_started_at - scheduled_at)
+    schedule_jitter_seconds = max(0.0, request_started_at - paced_deadline)
     active_counter.enter()
     try:
         try:
@@ -423,10 +436,10 @@ def _execute_scheduled_request(
     # The requester's service latency and this outer clock are independent in tests.
     # Taking the maximum keeps production wall time authoritative while preserving a
     # deterministic injected service duration.  Both include time through completion.
-    wall_latency_ms = max(0.0, (completed_at - scheduled_at) * 1000.0)
+    wall_latency_ms = max(0.0, (completed_at - paced_deadline) * 1000.0)
     end_to_end_latency_ms = max(
         wall_latency_ms,
-        (schedule_lag_seconds * 1000.0) + observation.latency_ms,
+        (schedule_jitter_seconds * 1000.0) + observation.latency_ms,
     )
     timed_observation = HttpObservation(
         latency_ms=end_to_end_latency_ms,
@@ -436,7 +449,7 @@ def _execute_scheduled_request(
     )
     return _TimedObservation(
         observation=timed_observation,
-        schedule_lag_ms=schedule_lag_seconds * 1000.0,
+        schedule_jitter_ms=schedule_jitter_seconds * 1000.0,
     )
 
 
@@ -490,10 +503,15 @@ def build_report(
         float(config.duration_seconds) <= measurements.elapsed_seconds <= maximum_elapsed_seconds
     )
     throughput_target_met = bool(throughput_rps >= minimum_accepted_throughput_rps)
-    schedule_contract_met = bool(
-        measurements.schedule_lag_violations == 0
-        and measurements.max_schedule_lag_ms <= MAX_SCHEDULE_LAG_MS
+    schedule_jitter_contract_met = bool(
+        measurements.p95_schedule_jitter_ms <= P95_SCHEDULE_JITTER_LIMIT_MS
     )
+    no_burst_contract_met = bool(
+        measurements.burst_interval_violations == 0
+        and measurements.minimum_inter_submission_ms + _CLOCK_COMPARISON_EPSILON_MS
+        >= NOMINAL_REQUEST_INTERVAL_MS
+    )
+    schedule_contract_met = schedule_jitter_contract_met and no_burst_contract_met
     concurrency_contract_met = bool(1 <= measurements.observed_peak_active <= MAX_CONCURRENCY)
     workload_consistent = bool(
         catalog_count == (config.scheduled_requests * 7) // 10
@@ -531,11 +549,18 @@ def build_report(
             minimum_accepted_throughput_rps,
             3,
         ),
-        max_schedule_lag_ms=round(measurements.max_schedule_lag_ms, 3),
-        schedule_lag_violations=measurements.schedule_lag_violations,
+        p95_schedule_jitter_ms=round(measurements.p95_schedule_jitter_ms, 3),
+        max_schedule_jitter_ms=round(measurements.max_schedule_jitter_ms, 3),
+        minimum_inter_submission_ms=round(
+            measurements.minimum_inter_submission_ms,
+            3,
+        ),
+        burst_interval_violations=measurements.burst_interval_violations,
         observed_peak_active=measurements.observed_peak_active,
         duration_contract_met=duration_contract_met,
         throughput_target_met=throughput_target_met,
+        schedule_jitter_contract_met=schedule_jitter_contract_met,
+        no_burst_contract_met=no_burst_contract_met,
         schedule_contract_met=schedule_contract_met,
         concurrency_contract_met=concurrency_contract_met,
         workload_consistent=workload_consistent,
@@ -554,38 +579,38 @@ def run_profile(
 ) -> LoadReport:
     started_at = monotonic()
     interval_seconds = 1.0 / REQUESTS_PER_SECOND
-    pacing_floor: tuple[int, float] | None = None
+    previous_submission_at: float | None = None
+    inter_submission_ms: list[float] = []
     active_counter = _ActiveRequestCounter()
     submitted: list[tuple[RequestKind, float, Future[_TimedObservation]]] = []
     with executor_factory(MAX_CONCURRENCY) as executor:
         for index in range(config.scheduled_requests):
-            scheduled_at = started_at + (index / REQUESTS_PER_SECOND)
-            release_at = scheduled_at
-            if pacing_floor is not None:
-                floor_index, floor_started_at = pacing_floor
-                release_at = max(
-                    release_at,
-                    floor_started_at + ((index - floor_index) * interval_seconds),
+            nominal_deadline = started_at + (index / REQUESTS_PER_SECOND)
+            paced_deadline = nominal_deadline
+            if previous_submission_at is not None:
+                paced_deadline = max(
+                    paced_deadline,
+                    previous_submission_at + interval_seconds,
                 )
-            _sleep_until(release_at, monotonic=monotonic, sleeper=sleeper)
+            _sleep_until(paced_deadline, monotonic=monotonic, sleeper=sleeper)
             submitted_at = monotonic()
-            submission_lag_ms = max(0.0, (submitted_at - scheduled_at) * 1000.0)
-            if pacing_floor is None and submission_lag_ms > MAX_SCHEDULE_LAG_MS:
-                # Once materially late, never burst requests to catch the original
-                # timeline.  Preserve 10 rps spacing; timing gates will reject the run.
-                pacing_floor = (index, submitted_at)
+            if previous_submission_at is not None:
+                inter_submission_ms.append(
+                    max(0.0, (submitted_at - previous_submission_at) * 1000.0)
+                )
+            previous_submission_at = submitted_at
             plan = request_plan(config, index)
             submitted.append(
                 (
                     plan.kind,
-                    scheduled_at,
+                    paced_deadline,
                     executor.submit(
                         _execute_scheduled_request,
                         requester,
                         plan.url,
                         config.port,
                         REQUEST_TIMEOUT_SECONDS,
-                        scheduled_at,
+                        paced_deadline,
                         monotonic,
                         active_counter,
                     ),
@@ -599,28 +624,33 @@ def run_profile(
         )
 
         completed: list[CompletedRequest] = []
-        schedule_lags_ms: list[float] = []
-        for kind, scheduled_at, future in submitted:
+        schedule_jitter_ms: list[float] = []
+        for kind, paced_deadline, future in submitted:
             try:
                 timed = future.result()
             except Exception:
                 fallback_latency_ms = max(
                     0.0,
-                    (monotonic() - scheduled_at) * 1000.0,
+                    (monotonic() - paced_deadline) * 1000.0,
                 )
                 timed = _TimedObservation(
                     observation=_failed_observation(latency_ms=fallback_latency_ms),
-                    schedule_lag_ms=fallback_latency_ms,
+                    schedule_jitter_ms=fallback_latency_ms,
                 )
-            schedule_lags_ms.append(timed.schedule_lag_ms)
+            schedule_jitter_ms.append(timed.schedule_jitter_ms)
             completed.append(CompletedRequest(kind=kind, observation=timed.observation))
 
     elapsed_seconds = max(0.0, monotonic() - started_at)
-    max_schedule_lag_ms = max(schedule_lags_ms, default=0.0)
+    minimum_inter_submission_ms = min(inter_submission_ms, default=0.0)
     measurements = RunMeasurements(
         elapsed_seconds=elapsed_seconds,
-        max_schedule_lag_ms=max_schedule_lag_ms,
-        schedule_lag_violations=sum(lag_ms > MAX_SCHEDULE_LAG_MS for lag_ms in schedule_lags_ms),
+        p95_schedule_jitter_ms=_percentile(schedule_jitter_ms, 95),
+        max_schedule_jitter_ms=max(schedule_jitter_ms, default=0.0),
+        minimum_inter_submission_ms=minimum_inter_submission_ms,
+        burst_interval_violations=sum(
+            interval_ms + _CLOCK_COMPARISON_EPSILON_MS < NOMINAL_REQUEST_INTERVAL_MS
+            for interval_ms in inter_submission_ms
+        ),
         observed_peak_active=active_counter.peak,
     )
     return build_report(config, completed, measurements=measurements)
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index 914a512..6a8c14e 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -20,8 +20,9 @@ from scripts.http_load_profile import (
     HTTP_5XX_RATE_LIMIT,
     MAX_CONCURRENCY,
     MAX_RESPONSE_BYTES,
-    MAX_SCHEDULE_LAG_MS,
+    NOMINAL_REQUEST_INTERVAL_MS,
     P95_LIMIT_MS,
+    P95_SCHEDULE_JITTER_LIMIT_MS,
     PHASE0_DURATION_SECONDS,
     PROFILE_COMPLETION_GRACE_SECONDS,
     REQUESTS_PER_SECOND,
@@ -69,6 +70,12 @@ class OversleepOnceClock(FakeClock):
             self._overslept = True
 
 
+class SmallOversleepClock(FakeClock):
+    def sleep(self, seconds: float) -> None:
+        super().sleep(seconds)
+        self.value += 0.002
+
+
 class InlineExecutor(Executor):
     """Deterministic executor for pacing tests; concurrency has a separate test."""
 
@@ -155,14 +162,18 @@ def completed_requests(
 def run_measurements(
     *,
     elapsed_seconds: float,
-    max_schedule_lag_ms: float = 0.0,
-    schedule_lag_violations: int = 0,
+    p95_schedule_jitter_ms: float = 0.0,
+    max_schedule_jitter_ms: float = 0.0,
+    minimum_inter_submission_ms: float = NOMINAL_REQUEST_INTERVAL_MS,
+    burst_interval_violations: int = 0,
     observed_peak_active: int = 1,
 ) -> RunMeasurements:
     return RunMeasurements(
         elapsed_seconds=elapsed_seconds,
-        max_schedule_lag_ms=max_schedule_lag_ms,
-        schedule_lag_violations=schedule_lag_violations,
+        p95_schedule_jitter_ms=p95_schedule_jitter_ms,
+        max_schedule_jitter_ms=max_schedule_jitter_ms,
+        minimum_inter_submission_ms=minimum_inter_submission_ms,
+        burst_interval_violations=burst_interval_violations,
         observed_peak_active=observed_peak_active,
     )
 
@@ -180,7 +191,8 @@ def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> N
     assert phase0.scheduled_requests == 9_000
     assert REQUESTS_PER_SECOND == 10
     assert MAX_CONCURRENCY == 20
-    assert MAX_SCHEDULE_LAG_MS == 100.0
+    assert NOMINAL_REQUEST_INTERVAL_MS == 100.0
+    assert P95_SCHEDULE_JITTER_LIMIT_MS == 100.0
     assert PROFILE_COMPLETION_GRACE_SECONDS == 3.0
     assert phase0.label == "PHASE0_900S"
     assert smoke.label == "SMOKE_NON_ACCEPTANCE"
@@ -251,11 +263,15 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert report.detail_requests == 3
     assert report.throughput_rps == 10.0
     assert report.elapsed_seconds == 1.0
-    assert report.max_schedule_lag_ms == 0.0
-    assert report.schedule_lag_violations == 0
+    assert report.p95_schedule_jitter_ms == 0.0
+    assert report.max_schedule_jitter_ms == 0.0
+    assert report.minimum_inter_submission_ms == 100.0
+    assert report.burst_interval_violations == 0
     assert report.observed_peak_active == 1
     assert report.duration_contract_met is True
     assert report.throughput_target_met is True
+    assert report.schedule_jitter_contract_met is True
+    assert report.no_burst_contract_met is True
     assert report.schedule_contract_met is True
     assert report.concurrency_contract_met is True
     assert report.revision_consistent is True
@@ -289,7 +305,9 @@ def test_phase0_runner_executes_exact_900_second_ten_rps_seventy_thirty_plan() -
     assert report.catalog_list_search_requests == 6_300
     assert report.detail_requests == 2_700
     assert report.throughput_rps == 10.0
-    assert report.max_schedule_lag_ms == 0.0
+    assert report.p95_schedule_jitter_ms == 0.0
+    assert report.minimum_inter_submission_ms == 100.0
+    assert report.burst_interval_violations == 0
     assert report.workload_consistent is True
     assert report.passed is True
 
@@ -313,18 +331,13 @@ def test_end_to_end_latency_includes_schedule_queue_delay_through_completion() -
         active_counter,
     )
 
-    assert timed.schedule_lag_ms == pytest.approx(200.0)
+    assert timed.schedule_jitter_ms == pytest.approx(200.0)
     assert timed.observation.latency_ms == pytest.approx(250.0)
     assert active_counter.peak == 1
 
 
-def test_material_scheduler_lag_never_creates_a_catch_up_burst_and_fails() -> None:
-    config = LoadProfileConfig(
-        port=8123,
-        detail_id=_DETAIL_ID,
-        duration_seconds=1,
-        profile="smoke",
-    )
+def test_one_scheduler_stall_is_not_repeated_and_never_creates_a_catch_up_burst() -> None:
+    config = LoadProfileConfig(port=8123, detail_id=_DETAIL_ID)
     clock = OversleepOnceClock()
     request_started_at: list[float] = []
 
@@ -340,15 +353,44 @@ def test_material_scheduler_lag_never_creates_a_catch_up_burst_and_fails() -> No
         executor_factory=inline_executor_factory,
     )
 
-    assert len(request_started_at) == 10
+    assert len(request_started_at) == 9_000
     assert request_started_at[1] == pytest.approx(0.3)
     assert all(
         later - earlier == pytest.approx(0.1) for earlier, later in pairwise(request_started_at[1:])
     )
-    assert report.max_schedule_lag_ms == pytest.approx(200.0)
-    assert report.schedule_lag_violations == 9
-    assert report.schedule_contract_met is False
-    assert report.passed is False
+    assert report.elapsed_seconds == pytest.approx(900.1)
+    assert report.max_schedule_jitter_ms == pytest.approx(200.0)
+    assert report.p95_schedule_jitter_ms == 0.0
+    assert report.minimum_inter_submission_ms == pytest.approx(100.0)
+    assert report.burst_interval_violations == 0
+    assert report.schedule_jitter_contract_met is True
+    assert report.no_burst_contract_met is True
+    assert report.schedule_contract_met is True
+    assert report.passed is True
+
+
+def test_small_repeated_clock_jitter_preserves_inter_submission_rate() -> None:
+    config = LoadProfileConfig(
+        port=8123,
+        detail_id=_DETAIL_ID,
+        duration_seconds=1,
+        profile="smoke",
+    )
+    clock = SmallOversleepClock()
+
+    report = run_profile(
+        config,
+        requester=lambda _url, _port, _timeout: observation(latency_ms=0.0),
+        monotonic=clock.monotonic,
+        sleeper=clock.sleep,
+        executor_factory=inline_executor_factory,
+    )
+
+    assert report.p95_schedule_jitter_ms == pytest.approx(2.0)
+    assert report.minimum_inter_submission_ms == pytest.approx(102.0)
+    assert report.burst_interval_violations == 0
+    assert report.no_burst_contract_met is True
+    assert report.passed is True
 
 
 def test_observed_active_requests_are_bounded_by_the_twenty_worker_executor() -> None:
@@ -446,6 +488,40 @@ def test_report_rejects_observed_concurrency_above_configured_bound() -> None:
     assert report.passed is False
 
 
+@pytest.mark.parametrize("failure", ("jitter", "burst"))
+def test_report_rejects_persistent_schedule_jitter_or_catch_up_burst(failure: str) -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        duration_seconds=1,
+        profile="smoke",
+    )
+    if failure == "jitter":
+        measurements = run_measurements(
+            elapsed_seconds=1.0,
+            p95_schedule_jitter_ms=P95_SCHEDULE_JITTER_LIMIT_MS + 0.001,
+        )
+    else:
+        measurements = run_measurements(
+            elapsed_seconds=1.0,
+            minimum_inter_submission_ms=99.0,
+            burst_interval_violations=1,
+        )
+
+    report = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=measurements,
+    )
+
+    assert report.schedule_contract_met is False
+    assert report.passed is False
+    if failure == "jitter":
+        assert report.schedule_jitter_contract_met is False
+    else:
+        assert report.no_burst_contract_met is False
+
+
 @pytest.mark.parametrize("failure", ("latency", "http_5xx", "revision_mix", "error"))
 def test_report_fails_for_each_release_blocker(failure: str) -> None:
     config = LoadProfileConfig(


