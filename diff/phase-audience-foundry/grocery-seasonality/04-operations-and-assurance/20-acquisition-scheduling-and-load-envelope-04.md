## `fix(perf): recover paced schedule without bursts`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index 1da6fe7..e6930e3 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -41,6 +41,9 @@ HTTP_5XX_RATE_LIMIT: Final = 0.005
 P95_SCHEDULE_JITTER_LIMIT_MS: Final = 100.0
 PROFILE_COMPLETION_GRACE_SECONDS: Final = 3.0
 NOMINAL_REQUEST_INTERVAL_MS: Final = 1000.0 / REQUESTS_PER_SECOND
+# Recover scheduler stalls by at most 10 ms per request; never submit less than
+# 90 ms after the prior actual submission.
+RECOVERY_FLOOR_INTERVAL_MS: Final = 90.0
 _CLOCK_COMPARISON_EPSILON_MS: Final = 0.001
 
 _LOCAL_HOST: Final = "127.0.0.1"
@@ -221,6 +224,7 @@ class LoadReport:
                 "schedule_jitter_contract_met": self.schedule_jitter_contract_met,
                 "minimum_inter_submission_ms": self.minimum_inter_submission_ms,
                 "nominal_request_interval_ms": NOMINAL_REQUEST_INTERVAL_MS,
+                "recovery_floor_interval_ms": RECOVERY_FLOOR_INTERVAL_MS,
                 "burst_interval_violations": self.burst_interval_violations,
                 "no_burst_contract_met": self.no_burst_contract_met,
                 "schedule_contract_met": self.schedule_contract_met,
@@ -509,7 +513,7 @@ def build_report(
     no_burst_contract_met = bool(
         measurements.burst_interval_violations == 0
         and measurements.minimum_inter_submission_ms + _CLOCK_COMPARISON_EPSILON_MS
-        >= NOMINAL_REQUEST_INTERVAL_MS
+        >= RECOVERY_FLOOR_INTERVAL_MS
     )
     schedule_contract_met = schedule_jitter_contract_met and no_burst_contract_met
     concurrency_contract_met = bool(1 <= measurements.observed_peak_active <= MAX_CONCURRENCY)
@@ -578,7 +582,7 @@ def run_profile(
     executor_factory: Callable[[int], Executor] = ThreadPoolExecutor,
 ) -> LoadReport:
     started_at = monotonic()
-    interval_seconds = 1.0 / REQUESTS_PER_SECOND
+    recovery_floor_seconds = RECOVERY_FLOOR_INTERVAL_MS / 1000.0
     previous_submission_at: float | None = None
     inter_submission_ms: list[float] = []
     active_counter = _ActiveRequestCounter()
@@ -590,7 +594,7 @@ def run_profile(
             if previous_submission_at is not None:
                 paced_deadline = max(
                     paced_deadline,
-                    previous_submission_at + interval_seconds,
+                    previous_submission_at + recovery_floor_seconds,
                 )
             _sleep_until(paced_deadline, monotonic=monotonic, sleeper=sleeper)
             submitted_at = monotonic()
@@ -648,7 +652,7 @@ def run_profile(
         max_schedule_jitter_ms=max(schedule_jitter_ms, default=0.0),
         minimum_inter_submission_ms=minimum_inter_submission_ms,
         burst_interval_violations=sum(
-            interval_ms + _CLOCK_COMPARISON_EPSILON_MS < NOMINAL_REQUEST_INTERVAL_MS
+            interval_ms + _CLOCK_COMPARISON_EPSILON_MS < RECOVERY_FLOOR_INTERVAL_MS
             for interval_ms in inter_submission_ms
         ),
         observed_peak_active=active_counter.peak,
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index 6a8c14e..791043c 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -25,6 +25,7 @@ from scripts.http_load_profile import (
     P95_SCHEDULE_JITTER_LIMIT_MS,
     PHASE0_DURATION_SECONDS,
     PROFILE_COMPLETION_GRACE_SECONDS,
+    RECOVERY_FLOOR_INTERVAL_MS,
     REQUESTS_PER_SECOND,
     CompletedRequest,
     HttpObservation,
@@ -192,6 +193,7 @@ def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> N
     assert REQUESTS_PER_SECOND == 10
     assert MAX_CONCURRENCY == 20
     assert NOMINAL_REQUEST_INTERVAL_MS == 100.0
+    assert RECOVERY_FLOOR_INTERVAL_MS == 90.0
     assert P95_SCHEDULE_JITTER_LIMIT_MS == 100.0
     assert PROFILE_COMPLETION_GRACE_SECONDS == 3.0
     assert phase0.label == "PHASE0_900S"
@@ -277,6 +279,10 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert report.revision_consistent is True
     assert report.passed is True
     assert clock.value == 1.0
+    timing = report.data()["timing"]
+    assert isinstance(timing, dict)
+    assert timing["nominal_request_interval_ms"] == 100.0
+    assert timing["recovery_floor_interval_ms"] == 90.0
     receipt = report.render()
     assert str(_DETAIL_ID) not in receipt
     assert _REVISION_TOKEN not in receipt
@@ -312,6 +318,32 @@ def test_phase0_runner_executes_exact_900_second_ten_rps_seventy_thirty_plan() -
     assert report.passed is True
 
 
+def test_phase0_elapsed_and_throughput_boundary_is_exactly_nine_hundred_to_nine_oh_three() -> None:
+    config = LoadProfileConfig(port=8123, detail_id=_DETAIL_ID)
+    requests = completed_requests([observation() for _index in range(9_000)])
+
+    at_boundary = build_report(
+        config,
+        requests,
+        measurements=run_measurements(elapsed_seconds=903.0),
+    )
+    beyond_boundary = build_report(
+        config,
+        requests,
+        measurements=run_measurements(elapsed_seconds=903.001),
+    )
+
+    assert at_boundary.elapsed_seconds == 903.0
+    assert at_boundary.throughput_rps == round(9_000 / 903, 3)
+    assert at_boundary.minimum_accepted_throughput_rps == round(9_000 / 903, 3)
+    assert at_boundary.duration_contract_met is True
+    assert at_boundary.throughput_target_met is True
+    assert at_boundary.passed is True
+    assert beyond_boundary.duration_contract_met is False
+    assert beyond_boundary.throughput_target_met is False
+    assert beyond_boundary.passed is False
+
+
 def test_end_to_end_latency_includes_schedule_queue_delay_through_completion() -> None:
     clock = FakeClock()
     clock.value = 0.3
@@ -336,7 +368,7 @@ def test_end_to_end_latency_includes_schedule_queue_delay_through_completion() -
     assert active_counter.peak == 1
 
 
-def test_one_scheduler_stall_is_not_repeated_and_never_creates_a_catch_up_burst() -> None:
+def test_one_scheduler_stall_recovers_gradually_without_a_catch_up_burst() -> None:
     config = LoadProfileConfig(port=8123, detail_id=_DETAIL_ID)
     clock = OversleepOnceClock()
     request_started_at: list[float] = []
@@ -356,12 +388,17 @@ def test_one_scheduler_stall_is_not_repeated_and_never_creates_a_catch_up_burst(
     assert len(request_started_at) == 9_000
     assert request_started_at[1] == pytest.approx(0.3)
     assert all(
-        later - earlier == pytest.approx(0.1) for earlier, later in pairwise(request_started_at[1:])
+        later - earlier == pytest.approx(0.09)
+        for earlier, later in pairwise(request_started_at[1:22])
+    )
+    assert all(
+        later - earlier == pytest.approx(0.1)
+        for earlier, later in pairwise(request_started_at[21:])
     )
-    assert report.elapsed_seconds == pytest.approx(900.1)
+    assert report.elapsed_seconds == pytest.approx(900.0)
     assert report.max_schedule_jitter_ms == pytest.approx(200.0)
     assert report.p95_schedule_jitter_ms == 0.0
-    assert report.minimum_inter_submission_ms == pytest.approx(100.0)
+    assert report.minimum_inter_submission_ms == pytest.approx(90.0)
     assert report.burst_interval_violations == 0
     assert report.schedule_jitter_contract_met is True
     assert report.no_burst_contract_met is True
@@ -387,7 +424,7 @@ def test_small_repeated_clock_jitter_preserves_inter_submission_rate() -> None:
     )
 
     assert report.p95_schedule_jitter_ms == pytest.approx(2.0)
-    assert report.minimum_inter_submission_ms == pytest.approx(102.0)
+    assert report.minimum_inter_submission_ms == pytest.approx(100.0)
     assert report.burst_interval_violations == 0
     assert report.no_burst_contract_met is True
     assert report.passed is True
@@ -504,7 +541,7 @@ def test_report_rejects_persistent_schedule_jitter_or_catch_up_burst(failure: st
     else:
         measurements = run_measurements(
             elapsed_seconds=1.0,
-            minimum_inter_submission_ms=99.0,
+            minimum_inter_submission_ms=RECOVERY_FLOOR_INTERVAL_MS - 0.001,
             burst_interval_violations=1,
         )
 


## `fix(perf): model twenty logical users`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index e6930e3..06d1c4a 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -19,6 +19,7 @@ import time
 import uuid
 from collections.abc import Callable
 from concurrent.futures import Executor, Future, ThreadPoolExecutor
+from contextlib import ExitStack
 from dataclasses import dataclass, field
 from typing import Any, Final, Literal, Never
 from urllib.error import HTTPError
@@ -33,7 +34,8 @@ from urllib.request import (
 
 PHASE0_DURATION_SECONDS: Final = 900
 REQUESTS_PER_SECOND: Final = 10
-MAX_CONCURRENCY: Final = 20
+LOGICAL_VIRTUAL_USERS: Final = 20
+MAX_CONCURRENCY: Final = LOGICAL_VIRTUAL_USERS
 REQUEST_TIMEOUT_SECONDS: Final = 2.0
 MAX_RESPONSE_BYTES: Final = 2 * 1024 * 1024
 P95_LIMIT_MS: Final = 500.0
@@ -140,6 +142,7 @@ class HttpObservation:
 @dataclass(frozen=True, slots=True)
 class CompletedRequest:
     kind: RequestKind
+    virtual_user_id: int
     observation: HttpObservation
 
 
@@ -175,12 +178,15 @@ class LoadReport:
     max_schedule_jitter_ms: float
     minimum_inter_submission_ms: float
     burst_interval_violations: int
+    logical_users_configured: int
+    logical_users_participated: int
     observed_peak_active: int
     duration_contract_met: bool
     throughput_target_met: bool
     schedule_jitter_contract_met: bool
     no_burst_contract_met: bool
     schedule_contract_met: bool
+    logical_users_contract_met: bool
     concurrency_contract_met: bool
     workload_consistent: bool
     revision_consistent: bool
@@ -191,10 +197,15 @@ class LoadReport:
             "profile": self.profile,
             "duration_seconds": self.duration_seconds,
             "target_requests_per_second": REQUESTS_PER_SECOND,
+            "logical_users": {
+                "configured": self.logical_users_configured,
+                "participated": self.logical_users_participated,
+                "round_robin_contract_met": self.logical_users_contract_met,
+            },
             "concurrency": {
-                "limit": MAX_CONCURRENCY,
-                "observed_peak_active": self.observed_peak_active,
-                "within_limit": self.concurrency_contract_met,
+                "in_flight_limit": MAX_CONCURRENCY,
+                "observed_in_flight_peak": self.observed_peak_active,
+                "in_flight_within_limit": self.concurrency_contract_met,
             },
             "counts": {
                 "scheduled": self.scheduled_requests,
@@ -417,6 +428,7 @@ class _ActiveRequestCounter:
 
 
 def _execute_scheduled_request(
+    virtual_user_id: int,
     requester: Requester,
     url: str,
     port: int,
@@ -425,6 +437,8 @@ def _execute_scheduled_request(
     monotonic: Callable[[], float],
     active_counter: _ActiveRequestCounter,
 ) -> _TimedObservation:
+    if type(virtual_user_id) is not int or not 0 <= virtual_user_id < LOGICAL_VIRTUAL_USERS:
+        raise LoadProfileError("argument_invalid")
     request_started_at = monotonic()
     schedule_jitter_seconds = max(0.0, request_started_at - paced_deadline)
     active_counter.enter()
@@ -516,6 +530,18 @@ def build_report(
         >= RECOVERY_FLOOR_INTERVAL_MS
     )
     schedule_contract_met = schedule_jitter_contract_met and no_burst_contract_met
+    expected_virtual_user_ids = [
+        index % LOGICAL_VIRTUAL_USERS for index in range(config.scheduled_requests)
+    ]
+    completed_virtual_user_ids = [request.virtual_user_id for request in completed]
+    logical_users_participated = len(
+        {
+            virtual_user_id
+            for virtual_user_id in completed_virtual_user_ids
+            if type(virtual_user_id) is int and 0 <= virtual_user_id < LOGICAL_VIRTUAL_USERS
+        }
+    )
+    logical_users_contract_met = bool(completed_virtual_user_ids == expected_virtual_user_ids)
     concurrency_contract_met = bool(1 <= measurements.observed_peak_active <= MAX_CONCURRENCY)
     workload_consistent = bool(
         catalog_count == (config.scheduled_requests * 7) // 10
@@ -531,6 +557,7 @@ def build_report(
         and duration_contract_met
         and throughput_target_met
         and schedule_contract_met
+        and logical_users_contract_met
         and concurrency_contract_met
     )
     return LoadReport(
@@ -560,12 +587,15 @@ def build_report(
             3,
         ),
         burst_interval_violations=measurements.burst_interval_violations,
+        logical_users_configured=LOGICAL_VIRTUAL_USERS,
+        logical_users_participated=logical_users_participated,
         observed_peak_active=measurements.observed_peak_active,
         duration_contract_met=duration_contract_met,
         throughput_target_met=throughput_target_met,
         schedule_jitter_contract_met=schedule_jitter_contract_met,
         no_burst_contract_met=no_burst_contract_met,
         schedule_contract_met=schedule_contract_met,
+        logical_users_contract_met=logical_users_contract_met,
         concurrency_contract_met=concurrency_contract_met,
         workload_consistent=workload_consistent,
         revision_consistent=revision_consistent,
@@ -586,8 +616,12 @@ def run_profile(
     previous_submission_at: float | None = None
     inter_submission_ms: list[float] = []
     active_counter = _ActiveRequestCounter()
-    submitted: list[tuple[RequestKind, float, Future[_TimedObservation]]] = []
-    with executor_factory(MAX_CONCURRENCY) as executor:
+    submitted: list[tuple[RequestKind, int, float, Future[_TimedObservation]]] = []
+    with ExitStack() as executor_stack:
+        virtual_user_executors = tuple(
+            executor_stack.enter_context(executor_factory(1))
+            for _virtual_user_id in range(LOGICAL_VIRTUAL_USERS)
+        )
         for index in range(config.scheduled_requests):
             nominal_deadline = started_at + (index / REQUESTS_PER_SECOND)
             paced_deadline = nominal_deadline
@@ -604,12 +638,15 @@ def run_profile(
                 )
             previous_submission_at = submitted_at
             plan = request_plan(config, index)
+            virtual_user_id = index % LOGICAL_VIRTUAL_USERS
             submitted.append(
                 (
                     plan.kind,
+                    virtual_user_id,
                     paced_deadline,
-                    executor.submit(
+                    virtual_user_executors[virtual_user_id].submit(
                         _execute_scheduled_request,
+                        virtual_user_id,
                         requester,
                         plan.url,
                         config.port,
@@ -629,7 +666,7 @@ def run_profile(
 
         completed: list[CompletedRequest] = []
         schedule_jitter_ms: list[float] = []
-        for kind, paced_deadline, future in submitted:
+        for kind, virtual_user_id, paced_deadline, future in submitted:
             try:
                 timed = future.result()
             except Exception:
@@ -642,7 +679,13 @@ def run_profile(
                     schedule_jitter_ms=fallback_latency_ms,
                 )
             schedule_jitter_ms.append(timed.schedule_jitter_ms)
-            completed.append(CompletedRequest(kind=kind, observation=timed.observation))
+            completed.append(
+                CompletedRequest(
+                    kind=kind,
+                    virtual_user_id=virtual_user_id,
+                    observation=timed.observation,
+                )
+            )
 
     elapsed_seconds = max(0.0, monotonic() - started_at)
     minimum_inter_submission_ms = min(inter_submission_ms, default=0.0)
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index 791043c..8fbce1e 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -18,6 +18,7 @@ import pytest
 
 from scripts.http_load_profile import (
     HTTP_5XX_RATE_LIMIT,
+    LOGICAL_VIRTUAL_USERS,
     MAX_CONCURRENCY,
     MAX_RESPONSE_BYTES,
     NOMINAL_REQUEST_INTERVAL_MS,
@@ -96,10 +97,34 @@ class InlineExecutor(Executor):
 
 
 def inline_executor_factory(max_workers: int) -> Executor:
-    assert max_workers == MAX_CONCURRENCY
+    assert max_workers == 1
     return InlineExecutor()
 
 
+class RecordingSessionExecutor(InlineExecutor):
+    def __init__(
+        self,
+        virtual_user_id: int,
+        submission_order: list[int],
+    ) -> None:
+        self.virtual_user_id = virtual_user_id
+        self.submission_order = submission_order
+        self.submitted_ids: list[int] = []
+
+    def submit[T](
+        self,
+        fn: Callable[..., T],
+        /,
+        *args: Any,
+        **kwargs: Any,
+    ) -> Future[T]:
+        submitted_id = args[0]
+        assert submitted_id == self.virtual_user_id
+        self.submitted_ids.append(submitted_id)
+        self.submission_order.append(submitted_id)
+        return super().submit(fn, *args, **kwargs)
+
+
 class FakeResponse:
     def __init__(
         self,
@@ -155,7 +180,11 @@ def completed_requests(
     observations: list[HttpObservation],
 ) -> list[CompletedRequest]:
     return [
-        CompletedRequest(kind="catalog" if index % 10 < 7 else "detail", observation=value)
+        CompletedRequest(
+            kind="catalog" if index % 10 < 7 else "detail",
+            virtual_user_id=index % LOGICAL_VIRTUAL_USERS,
+            observation=value,
+        )
         for index, value in enumerate(observations)
     ]
 
@@ -191,7 +220,7 @@ def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> N
     assert phase0.duration_seconds == PHASE0_DURATION_SECONDS == 900
     assert phase0.scheduled_requests == 9_000
     assert REQUESTS_PER_SECOND == 10
-    assert MAX_CONCURRENCY == 20
+    assert LOGICAL_VIRTUAL_USERS == MAX_CONCURRENCY == 20
     assert NOMINAL_REQUEST_INTERVAL_MS == 100.0
     assert RECOVERY_FLOOR_INTERVAL_MS == 90.0
     assert P95_SCHEDULE_JITTER_LIMIT_MS == 100.0
@@ -269,6 +298,9 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert report.max_schedule_jitter_ms == 0.0
     assert report.minimum_inter_submission_ms == 100.0
     assert report.burst_interval_violations == 0
+    assert report.logical_users_configured == 20
+    assert report.logical_users_participated == 10
+    assert report.logical_users_contract_met is True
     assert report.observed_peak_active == 1
     assert report.duration_contract_met is True
     assert report.throughput_target_met is True
@@ -283,12 +315,25 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert isinstance(timing, dict)
     assert timing["nominal_request_interval_ms"] == 100.0
     assert timing["recovery_floor_interval_ms"] == 90.0
+    logical_users = report.data()["logical_users"]
+    assert logical_users == {
+        "configured": 20,
+        "participated": 10,
+        "round_robin_contract_met": True,
+    }
+    concurrency = report.data()["concurrency"]
+    assert concurrency == {
+        "in_flight_limit": 20,
+        "observed_in_flight_peak": 1,
+        "in_flight_within_limit": True,
+    }
     receipt = report.render()
     assert str(_DETAIL_ID) not in receipt
     assert _REVISION_TOKEN not in receipt
     assert "series/" not in receipt
     assert "category=" not in receipt
     assert "%" not in receipt
+    assert "virtual_user_id" not in receipt
 
 
 def test_phase0_runner_executes_exact_900_second_ten_rps_seventy_thirty_plan() -> None:
@@ -314,10 +359,50 @@ def test_phase0_runner_executes_exact_900_second_ten_rps_seventy_thirty_plan() -
     assert report.p95_schedule_jitter_ms == 0.0
     assert report.minimum_inter_submission_ms == 100.0
     assert report.burst_interval_violations == 0
+    assert report.logical_users_configured == 20
+    assert report.logical_users_participated == 20
+    assert report.logical_users_contract_met is True
     assert report.workload_consistent is True
     assert report.passed is True
 
 
+def test_twenty_fixed_logical_user_sessions_receive_requests_round_robin() -> None:
+    config = LoadProfileConfig(
+        port=8123,
+        detail_id=_DETAIL_ID,
+        duration_seconds=4,
+        profile="smoke",
+    )
+    clock = FakeClock()
+    sessions: list[RecordingSessionExecutor] = []
+    submission_order: list[int] = []
+
+    def session_executor_factory(max_workers: int) -> Executor:
+        assert max_workers == 1
+        session = RecordingSessionExecutor(len(sessions), submission_order)
+        sessions.append(session)
+        return session
+
+    report = run_profile(
+        config,
+        requester=lambda _url, _port, _timeout: observation(),
+        monotonic=clock.monotonic,
+        sleeper=clock.sleep,
+        executor_factory=session_executor_factory,
+    )
+
+    assert len(sessions) == LOGICAL_VIRTUAL_USERS
+    assert submission_order == list(range(LOGICAL_VIRTUAL_USERS)) * 2
+    assert [session.submitted_ids for session in sessions] == [
+        [virtual_user_id, virtual_user_id] for virtual_user_id in range(LOGICAL_VIRTUAL_USERS)
+    ]
+    assert report.logical_users_configured == 20
+    assert report.logical_users_participated == 20
+    assert report.logical_users_contract_met is True
+    assert report.observed_peak_active == 1
+    assert report.passed is True
+
+
 def test_phase0_elapsed_and_throughput_boundary_is_exactly_nine_hundred_to_nine_oh_three() -> None:
     config = LoadProfileConfig(port=8123, detail_id=_DETAIL_ID)
     requests = completed_requests([observation() for _index in range(9_000)])
@@ -354,6 +439,7 @@ def test_end_to_end_latency_includes_schedule_queue_delay_through_completion() -
         return observation(latency_ms=50.0)
 
     timed = _execute_scheduled_request(
+        0,
         requester,
         "http://127.0.0.1:8000/",
         8000,
@@ -430,7 +516,7 @@ def test_small_repeated_clock_jitter_preserves_inter_submission_rate() -> None:
     assert report.passed is True
 
 
-def test_observed_active_requests_are_bounded_by_the_twenty_worker_executor() -> None:
+def test_observed_in_flight_peak_is_measured_separately_and_bounded_at_twenty() -> None:
     active_counter = _ActiveRequestCounter()
     all_active = threading.Barrier(MAX_CONCURRENCY)
 
@@ -443,6 +529,7 @@ def test_observed_active_requests_are_bounded_by_the_twenty_worker_executor() ->
         futures = [
             executor.submit(
                 _execute_scheduled_request,
+                virtual_user_id,
                 requester,
                 "http://127.0.0.1:8000/",
                 8000,
@@ -451,7 +538,7 @@ def test_observed_active_requests_are_bounded_by_the_twenty_worker_executor() ->
                 time.monotonic,
                 active_counter,
             )
-            for _index in range(MAX_CONCURRENCY)
+            for virtual_user_id in range(MAX_CONCURRENCY)
         ]
         assert all(future.result().observation.valid for future in futures)
 
@@ -525,6 +612,39 @@ def test_report_rejects_observed_concurrency_above_configured_bound() -> None:
     assert report.passed is False
 
 
+def test_report_rejects_non_round_robin_logical_user_sequence() -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        duration_seconds=2,
+        profile="smoke",
+    )
+    requests = completed_requests([observation() for _index in range(20)])
+    first = requests[0]
+    second = requests[1]
+    requests[0] = CompletedRequest(
+        kind=first.kind,
+        virtual_user_id=second.virtual_user_id,
+        observation=first.observation,
+    )
+    requests[1] = CompletedRequest(
+        kind=second.kind,
+        virtual_user_id=first.virtual_user_id,
+        observation=second.observation,
+    )
+
+    report = build_report(
+        config,
+        requests,
+        measurements=run_measurements(elapsed_seconds=2.0),
+    )
+
+    assert report.logical_users_configured == 20
+    assert report.logical_users_participated == 20
+    assert report.logical_users_contract_met is False
+    assert report.passed is False
+
+
 @pytest.mark.parametrize("failure", ("jitter", "burst"))
 def test_report_rejects_persistent_schedule_jitter_or_catch_up_burst(failure: str) -> None:
     config = LoadProfileConfig(


