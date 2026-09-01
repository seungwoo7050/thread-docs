## `fix(perf): measure scheduled end-to-end load`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index 199aad6..9abb45b 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -1,4 +1,11 @@
-"""Bounded Phase 0 HTTP read-profile runner for the local Django candidate."""
+"""Bounded Phase 0 HTTP read-profile runner for the local Django candidate.
+
+The runner deliberately reaches only a trusted loopback process.  A standard-library
+socket timeout cannot forcibly stop a peer that keeps slowly producing bytes, so the
+worker pool may take longer than the nominal profile to return in that pathological
+case.  Such a run cannot pass: queue time is included in end-to-end latency and total
+elapsed time must remain inside the fixed completion grace.
+"""
 
 from __future__ import annotations
 
@@ -11,7 +18,7 @@ import threading
 import time
 import uuid
 from collections.abc import Callable
-from concurrent.futures import Future, ThreadPoolExecutor
+from concurrent.futures import Executor, Future, ThreadPoolExecutor
 from dataclasses import dataclass, field
 from typing import Any, Final, Literal, Never
 from urllib.error import HTTPError
@@ -31,6 +38,8 @@ REQUEST_TIMEOUT_SECONDS: Final = 2.0
 MAX_RESPONSE_BYTES: Final = 2 * 1024 * 1024
 P95_LIMIT_MS: Final = 500.0
 HTTP_5XX_RATE_LIMIT: Final = 0.005
+MAX_SCHEDULE_LAG_MS: Final = 100.0
+PROFILE_COMPLETION_GRACE_SECONDS: Final = 3.0
 
 _LOCAL_HOST: Final = "127.0.0.1"
 _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
@@ -46,6 +55,7 @@ _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
     "detail",
 )
 _POSITIVE_INTEGER = re.compile(r"[1-9][0-9]*\Z")
+_REVISION_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
 _THREAD_STATE = threading.local()
 
 type RequestKind = Literal["catalog", "detail"]
@@ -128,6 +138,14 @@ class CompletedRequest:
     observation: HttpObservation
 
 
+@dataclass(frozen=True, slots=True)
+class RunMeasurements:
+    elapsed_seconds: float
+    max_schedule_lag_ms: float
+    schedule_lag_violations: int
+    observed_peak_active: int
+
+
 @dataclass(frozen=True, slots=True)
 class LoadReport:
     profile: str
@@ -143,7 +161,17 @@ class LoadReport:
     p95_ms: float
     max_ms: float
     http_5xx_rate: float
+    elapsed_seconds: float
     throughput_rps: float
+    minimum_accepted_throughput_rps: float
+    max_schedule_lag_ms: float
+    schedule_lag_violations: int
+    observed_peak_active: int
+    duration_contract_met: bool
+    throughput_target_met: bool
+    schedule_contract_met: bool
+    concurrency_contract_met: bool
+    workload_consistent: bool
     revision_consistent: bool
     passed: bool
 
@@ -152,7 +180,11 @@ class LoadReport:
             "profile": self.profile,
             "duration_seconds": self.duration_seconds,
             "target_requests_per_second": REQUESTS_PER_SECOND,
-            "concurrency": MAX_CONCURRENCY,
+            "concurrency": {
+                "limit": MAX_CONCURRENCY,
+                "observed_peak_active": self.observed_peak_active,
+                "within_limit": self.concurrency_contract_met,
+            },
             "counts": {
                 "scheduled": self.scheduled_requests,
                 "completed": self.completed_requests,
@@ -168,7 +200,19 @@ class LoadReport:
                 "max": self.max_ms,
             },
             "http_5xx_rate": self.http_5xx_rate,
-            "throughput_rps": self.throughput_rps,
+            "timing": {
+                "elapsed_seconds": self.elapsed_seconds,
+                "completion_grace_seconds": PROFILE_COMPLETION_GRACE_SECONDS,
+                "duration_contract_met": self.duration_contract_met,
+                "throughput_rps": self.throughput_rps,
+                "minimum_accepted_throughput_rps": self.minimum_accepted_throughput_rps,
+                "throughput_target_met": self.throughput_target_met,
+                "max_schedule_lag_ms": self.max_schedule_lag_ms,
+                "schedule_lag_limit_ms": MAX_SCHEDULE_LAG_MS,
+                "schedule_lag_violations": self.schedule_lag_violations,
+                "schedule_contract_met": self.schedule_contract_met,
+            },
+            "workload_consistent": self.workload_consistent,
             "revision_consistent": self.revision_consistent,
             "passed": self.passed,
         }
@@ -269,12 +313,9 @@ def _cache_control_is_no_store(value: object) -> bool:
 def _revision_token(value: object) -> str | None:
     if type(value) is not str:
         return None
-    token = value.strip()
-    if not 1 <= len(token) <= 256:
-        return None
-    if any(ord(character) < 33 or ord(character) > 126 for character in token):
+    if _REVISION_SHA256.fullmatch(value) is None:
         return None
-    return token
+    return value
 
 
 def http_request(url: str, expected_port: int, timeout_seconds: float) -> HttpObservation:
@@ -320,15 +361,85 @@ def http_request(url: str, expected_port: int, timeout_seconds: float) -> HttpOb
     )
 
 
-def _failed_observation() -> HttpObservation:
+def _failed_observation(*, latency_ms: float = 0.0) -> HttpObservation:
     return HttpObservation(
-        latency_ms=REQUEST_TIMEOUT_SECONDS * 1000.0,
+        latency_ms=max(0.0, latency_ms),
         status_code=None,
         valid=False,
         revision_token=None,
     )
 
 
+@dataclass(frozen=True, slots=True)
+class _TimedObservation:
+    observation: HttpObservation
+    schedule_lag_ms: float
+
+
+class _ActiveRequestCounter:
+    """Thread-safe observed concurrency without retaining request data."""
+
+    def __init__(self) -> None:
+        self._lock = threading.Lock()
+        self._active = 0
+        self._peak = 0
+
+    def enter(self) -> None:
+        with self._lock:
+            self._active += 1
+            self._peak = max(self._peak, self._active)
+
+    def leave(self) -> None:
+        with self._lock:
+            self._active -= 1
+
+    @property
+    def peak(self) -> int:
+        with self._lock:
+            return self._peak
+
+
+def _execute_scheduled_request(
+    requester: Requester,
+    url: str,
+    port: int,
+    timeout_seconds: float,
+    scheduled_at: float,
+    monotonic: Callable[[], float],
+    active_counter: _ActiveRequestCounter,
+) -> _TimedObservation:
+    request_started_at = monotonic()
+    schedule_lag_seconds = max(0.0, request_started_at - scheduled_at)
+    active_counter.enter()
+    try:
+        try:
+            observation = requester(url, port, timeout_seconds)
+        except Exception:
+            observation = _failed_observation()
+        completed_at = monotonic()
+    finally:
+        active_counter.leave()
+
+    # The requester's service latency and this outer clock are independent in tests.
+    # Taking the maximum keeps production wall time authoritative while preserving a
+    # deterministic injected service duration.  Both include time through completion.
+    wall_latency_ms = max(0.0, (completed_at - scheduled_at) * 1000.0)
+    end_to_end_latency_ms = max(
+        wall_latency_ms,
+        (schedule_lag_seconds * 1000.0) + observation.latency_ms,
+    )
+    timed_observation = HttpObservation(
+        latency_ms=end_to_end_latency_ms,
+        status_code=observation.status_code,
+        valid=observation.valid,
+        revision_token=observation.revision_token,
+    )
+    return _TimedObservation(
+        observation=timed_observation,
+        schedule_lag_ms=schedule_lag_seconds * 1000.0,
+    )
+
+
 def _percentile(values: list[float], percentile: int) -> float:
     if not values:
         return 0.0
@@ -341,7 +452,7 @@ def build_report(
     config: LoadProfileConfig,
     completed: list[CompletedRequest],
     *,
-    elapsed_seconds: float,
+    measurements: RunMeasurements,
 ) -> LoadReport:
     latencies = [request.observation.latency_ms for request in completed]
     successful = sum(
@@ -368,23 +479,45 @@ def build_report(
     p95_ms = _percentile(latencies, 95)
     max_ms = max(latencies, default=0.0)
     completed_count = len(completed)
+    catalog_count = sum(request.kind == "catalog" for request in completed)
+    detail_count = sum(request.kind == "detail" for request in completed)
     http_5xx_rate = http_5xx / completed_count if completed_count else 1.0
-    measurement_seconds = max(float(config.duration_seconds), elapsed_seconds, 0.001)
+    measurement_seconds = max(measurements.elapsed_seconds, 0.001)
     throughput_rps = completed_count / measurement_seconds
+    maximum_elapsed_seconds = float(config.duration_seconds) + PROFILE_COMPLETION_GRACE_SECONDS
+    minimum_accepted_throughput_rps = config.scheduled_requests / maximum_elapsed_seconds
+    duration_contract_met = bool(
+        float(config.duration_seconds) <= measurements.elapsed_seconds <= maximum_elapsed_seconds
+    )
+    throughput_target_met = bool(throughput_rps >= minimum_accepted_throughput_rps)
+    schedule_contract_met = bool(
+        measurements.schedule_lag_violations == 0
+        and measurements.max_schedule_lag_ms <= MAX_SCHEDULE_LAG_MS
+    )
+    concurrency_contract_met = bool(1 <= measurements.observed_peak_active <= MAX_CONCURRENCY)
+    workload_consistent = bool(
+        catalog_count == (config.scheduled_requests * 7) // 10
+        and detail_count == (config.scheduled_requests * 3) // 10
+    )
     report_passed = bool(
         completed_count == config.scheduled_requests
+        and workload_consistent
         and errors == 0
         and revision_consistent
         and p95_ms <= P95_LIMIT_MS
         and http_5xx_rate < HTTP_5XX_RATE_LIMIT
+        and duration_contract_met
+        and throughput_target_met
+        and schedule_contract_met
+        and concurrency_contract_met
     )
     return LoadReport(
         profile=config.label,
         duration_seconds=config.duration_seconds,
         scheduled_requests=config.scheduled_requests,
         completed_requests=completed_count,
-        catalog_list_search_requests=sum(request.kind == "catalog" for request in completed),
-        detail_requests=sum(request.kind == "detail" for request in completed),
+        catalog_list_search_requests=catalog_count,
+        detail_requests=detail_count,
         successful_requests=successful,
         error_count=errors,
         http_5xx_count=http_5xx,
@@ -392,7 +525,20 @@ def build_report(
         p95_ms=round(p95_ms, 3),
         max_ms=round(max_ms, 3),
         http_5xx_rate=round(http_5xx_rate, 6),
+        elapsed_seconds=round(measurements.elapsed_seconds, 3),
         throughput_rps=round(throughput_rps, 3),
+        minimum_accepted_throughput_rps=round(
+            minimum_accepted_throughput_rps,
+            3,
+        ),
+        max_schedule_lag_ms=round(measurements.max_schedule_lag_ms, 3),
+        schedule_lag_violations=measurements.schedule_lag_violations,
+        observed_peak_active=measurements.observed_peak_active,
+        duration_contract_met=duration_contract_met,
+        throughput_target_met=throughput_target_met,
+        schedule_contract_met=schedule_contract_met,
+        concurrency_contract_met=concurrency_contract_met,
+        workload_consistent=workload_consistent,
         revision_consistent=revision_consistent,
         passed=report_passed,
     )
@@ -404,42 +550,93 @@ def run_profile(
     requester: Requester = http_request,
     monotonic: Callable[[], float] = time.monotonic,
     sleeper: Callable[[float], None] = time.sleep,
+    executor_factory: Callable[[int], Executor] = ThreadPoolExecutor,
 ) -> LoadReport:
     started_at = monotonic()
-    submitted: list[tuple[RequestKind, Future[HttpObservation]]] = []
-    with ThreadPoolExecutor(max_workers=MAX_CONCURRENCY) as executor:
+    interval_seconds = 1.0 / REQUESTS_PER_SECOND
+    pacing_floor: tuple[int, float] | None = None
+    active_counter = _ActiveRequestCounter()
+    submitted: list[tuple[RequestKind, float, Future[_TimedObservation]]] = []
+    with executor_factory(MAX_CONCURRENCY) as executor:
         for index in range(config.scheduled_requests):
             scheduled_at = started_at + (index / REQUESTS_PER_SECOND)
-            remaining = scheduled_at - monotonic()
-            if remaining > 0:
-                sleeper(remaining)
+            release_at = scheduled_at
+            if pacing_floor is not None:
+                floor_index, floor_started_at = pacing_floor
+                release_at = max(
+                    release_at,
+                    floor_started_at + ((index - floor_index) * interval_seconds),
+                )
+            _sleep_until(release_at, monotonic=monotonic, sleeper=sleeper)
+            submitted_at = monotonic()
+            submission_lag_ms = max(0.0, (submitted_at - scheduled_at) * 1000.0)
+            if pacing_floor is None and submission_lag_ms > MAX_SCHEDULE_LAG_MS:
+                # Once materially late, never burst requests to catch the original
+                # timeline.  Preserve 10 rps spacing; timing gates will reject the run.
+                pacing_floor = (index, submitted_at)
             plan = request_plan(config, index)
             submitted.append(
                 (
                     plan.kind,
+                    scheduled_at,
                     executor.submit(
+                        _execute_scheduled_request,
                         requester,
                         plan.url,
                         config.port,
                         REQUEST_TIMEOUT_SECONDS,
+                        scheduled_at,
+                        monotonic,
+                        active_counter,
                     ),
                 )
             )
 
-        remaining_profile_time = started_at + config.duration_seconds - monotonic()
-        if remaining_profile_time > 0:
-            sleeper(remaining_profile_time)
+        _sleep_until(
+            started_at + config.duration_seconds,
+            monotonic=monotonic,
+            sleeper=sleeper,
+        )
 
         completed: list[CompletedRequest] = []
-        for kind, future in submitted:
+        schedule_lags_ms: list[float] = []
+        for kind, scheduled_at, future in submitted:
             try:
-                observation = future.result()
+                timed = future.result()
             except Exception:
-                observation = _failed_observation()
-            completed.append(CompletedRequest(kind=kind, observation=observation))
+                fallback_latency_ms = max(
+                    0.0,
+                    (monotonic() - scheduled_at) * 1000.0,
+                )
+                timed = _TimedObservation(
+                    observation=_failed_observation(latency_ms=fallback_latency_ms),
+                    schedule_lag_ms=fallback_latency_ms,
+                )
+            schedule_lags_ms.append(timed.schedule_lag_ms)
+            completed.append(CompletedRequest(kind=kind, observation=timed.observation))
 
     elapsed_seconds = max(0.0, monotonic() - started_at)
-    return build_report(config, completed, elapsed_seconds=elapsed_seconds)
+    max_schedule_lag_ms = max(schedule_lags_ms, default=0.0)
+    measurements = RunMeasurements(
+        elapsed_seconds=elapsed_seconds,
+        max_schedule_lag_ms=max_schedule_lag_ms,
+        schedule_lag_violations=sum(lag_ms > MAX_SCHEDULE_LAG_MS for lag_ms in schedule_lags_ms),
+        observed_peak_active=active_counter.peak,
+    )
+    return build_report(config, completed, measurements=measurements)
+
+
+def _sleep_until(
+    deadline: float,
+    *,
+    monotonic: Callable[[], float],
+    sleeper: Callable[[float], None],
+) -> None:
+    while True:
+        remaining = deadline - monotonic()
+        if remaining <= 0:
+            return
+        sleeper(remaining)
 
 
 def _positive_integer(value: str) -> int:
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index a1fe5aa..914a512 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -3,8 +3,13 @@ from __future__ import annotations
 import io
 import json
 import threading
+import time
 import uuid
+from collections.abc import Callable
+from concurrent.futures import Executor, Future, ThreadPoolExecutor
 from email.message import Message
+from itertools import pairwise
+from typing import Any
 from unittest.mock import MagicMock, patch
 from urllib.error import HTTPError
 from urllib.parse import urlsplit
@@ -15,13 +20,18 @@ from scripts.http_load_profile import (
     HTTP_5XX_RATE_LIMIT,
     MAX_CONCURRENCY,
     MAX_RESPONSE_BYTES,
+    MAX_SCHEDULE_LAG_MS,
     P95_LIMIT_MS,
     PHASE0_DURATION_SECONDS,
+    PROFILE_COMPLETION_GRACE_SECONDS,
     REQUESTS_PER_SECOND,
     CompletedRequest,
     HttpObservation,
     LoadProfileConfig,
     LoadProfileError,
+    RunMeasurements,
+    _ActiveRequestCounter,
+    _execute_scheduled_request,
     _NoRedirectHandler,
     _validate_local_url,
     build_report,
@@ -47,6 +57,41 @@ class FakeClock:
         self.value += seconds
 
 
+class OversleepOnceClock(FakeClock):
+    def __init__(self) -> None:
+        super().__init__()
+        self._overslept = False
+
+    def sleep(self, seconds: float) -> None:
+        super().sleep(seconds)
+        if not self._overslept:
+            self.value += 0.2
+            self._overslept = True
+
+
+class InlineExecutor(Executor):
+    """Deterministic executor for pacing tests; concurrency has a separate test."""
+
+    def submit[T](
+        self,
+        fn: Callable[..., T],
+        /,
+        *args: Any,
+        **kwargs: Any,
+    ) -> Future[T]:
+        future: Future[T] = Future()
+        try:
+            future.set_result(fn(*args, **kwargs))
+        except BaseException as error:
+            future.set_exception(error)
+        return future
+
+
+def inline_executor_factory(max_workers: int) -> Executor:
+    assert max_workers == MAX_CONCURRENCY
+    return InlineExecutor()
+
+
 class FakeResponse:
     def __init__(
         self,
@@ -107,6 +152,21 @@ def completed_requests(
     ]
 
 
+def run_measurements(
+    *,
+    elapsed_seconds: float,
+    max_schedule_lag_ms: float = 0.0,
+    schedule_lag_violations: int = 0,
+    observed_peak_active: int = 1,
+) -> RunMeasurements:
+    return RunMeasurements(
+        elapsed_seconds=elapsed_seconds,
+        max_schedule_lag_ms=max_schedule_lag_ms,
+        schedule_lag_violations=schedule_lag_violations,
+        observed_peak_active=observed_peak_active,
+    )
+
+
 def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> None:
     phase0 = LoadProfileConfig(port=8000, detail_id=_DETAIL_ID)
     smoke = LoadProfileConfig(
@@ -120,6 +180,8 @@ def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> N
     assert phase0.scheduled_requests == 9_000
     assert REQUESTS_PER_SECOND == 10
     assert MAX_CONCURRENCY == 20
+    assert MAX_SCHEDULE_LAG_MS == 100.0
+    assert PROFILE_COMPLETION_GRACE_SECONDS == 3.0
     assert phase0.label == "PHASE0_900S"
     assert smoke.label == "SMOKE_NON_ACCEPTANCE"
     with pytest.raises(LoadProfileError):
@@ -180,6 +242,7 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
         requester=requester,
         monotonic=clock.monotonic,
         sleeper=clock.sleep,
+        executor_factory=inline_executor_factory,
     )
 
     assert len(seen_urls) == 10
@@ -187,6 +250,14 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert report.catalog_list_search_requests == 7
     assert report.detail_requests == 3
     assert report.throughput_rps == 10.0
+    assert report.elapsed_seconds == 1.0
+    assert report.max_schedule_lag_ms == 0.0
+    assert report.schedule_lag_violations == 0
+    assert report.observed_peak_active == 1
+    assert report.duration_contract_met is True
+    assert report.throughput_target_met is True
+    assert report.schedule_contract_met is True
+    assert report.concurrency_contract_met is True
     assert report.revision_consistent is True
     assert report.passed is True
     assert clock.value == 1.0
@@ -198,6 +269,116 @@ def test_smoke_runner_paces_ten_requests_and_emits_no_request_or_revision_values
     assert "%" not in receipt
 
 
+def test_phase0_runner_executes_exact_900_second_ten_rps_seventy_thirty_plan() -> None:
+    config = LoadProfileConfig(port=8123, detail_id=_DETAIL_ID)
+    clock = FakeClock()
+
+    def requester(_url: str, _port: int, _timeout: float) -> HttpObservation:
+        return observation()
+
+    report = run_profile(
+        config,
+        requester=requester,
+        monotonic=clock.monotonic,
+        sleeper=clock.sleep,
+        executor_factory=inline_executor_factory,
+    )
+
+    assert report.elapsed_seconds == 900.0
+    assert report.completed_requests == 9_000
+    assert report.catalog_list_search_requests == 6_300
+    assert report.detail_requests == 2_700
+    assert report.throughput_rps == 10.0
+    assert report.max_schedule_lag_ms == 0.0
+    assert report.workload_consistent is True
+    assert report.passed is True
+
+
+def test_end_to_end_latency_includes_schedule_queue_delay_through_completion() -> None:
+    clock = FakeClock()
+    clock.value = 0.3
+    active_counter = _ActiveRequestCounter()
+
+    def requester(_url: str, _port: int, _timeout: float) -> HttpObservation:
+        clock.value += 0.05
+        return observation(latency_ms=50.0)
+
+    timed = _execute_scheduled_request(
+        requester,
+        "http://127.0.0.1:8000/",
+        8000,
+        2.0,
+        0.1,
+        clock.monotonic,
+        active_counter,
+    )
+
+    assert timed.schedule_lag_ms == pytest.approx(200.0)
+    assert timed.observation.latency_ms == pytest.approx(250.0)
+    assert active_counter.peak == 1
+
+
+def test_material_scheduler_lag_never_creates_a_catch_up_burst_and_fails() -> None:
+    config = LoadProfileConfig(
+        port=8123,
+        detail_id=_DETAIL_ID,
+        duration_seconds=1,
+        profile="smoke",
+    )
+    clock = OversleepOnceClock()
+    request_started_at: list[float] = []
+
+    def requester(_url: str, _port: int, _timeout: float) -> HttpObservation:
+        request_started_at.append(clock.monotonic())
+        return observation(latency_ms=0.0)
+
+    report = run_profile(
+        config,
+        requester=requester,
+        monotonic=clock.monotonic,
+        sleeper=clock.sleep,
+        executor_factory=inline_executor_factory,
+    )
+
+    assert len(request_started_at) == 10
+    assert request_started_at[1] == pytest.approx(0.3)
+    assert all(
+        later - earlier == pytest.approx(0.1) for earlier, later in pairwise(request_started_at[1:])
+    )
+    assert report.max_schedule_lag_ms == pytest.approx(200.0)
+    assert report.schedule_lag_violations == 9
+    assert report.schedule_contract_met is False
+    assert report.passed is False
+
+
+def test_observed_active_requests_are_bounded_by_the_twenty_worker_executor() -> None:
+    active_counter = _ActiveRequestCounter()
+    all_active = threading.Barrier(MAX_CONCURRENCY)
+
+    def requester(_url: str, _port: int, _timeout: float) -> HttpObservation:
+        all_active.wait(timeout=5.0)
+        return observation(latency_ms=1.0)
+
+    scheduled_at = time.monotonic()
+    with ThreadPoolExecutor(max_workers=MAX_CONCURRENCY) as executor:
+        futures = [
+            executor.submit(
+                _execute_scheduled_request,
+                requester,
+                "http://127.0.0.1:8000/",
+                8000,
+                2.0,
+                scheduled_at,
+                time.monotonic,
+                active_counter,
+            )
+            for _index in range(MAX_CONCURRENCY)
+        ]
+        assert all(future.result().observation.valid for future in futures)
+
+    assert active_counter.peak == MAX_CONCURRENCY == 20
+
+
 def test_report_uses_nearest_rank_metrics_and_exact_release_thresholds() -> None:
     config = LoadProfileConfig(
         port=8000,
@@ -206,7 +387,11 @@ def test_report_uses_nearest_rank_metrics_and_exact_release_thresholds() -> None
         profile="smoke",
     )
     values = [observation(latency_ms=float(index)) for index in range(1, 101)]
-    report = build_report(config, completed_requests(values), elapsed_seconds=10.0)
+    report = build_report(
+        config,
+        completed_requests(values),
+        measurements=run_measurements(elapsed_seconds=10.0),
+    )
 
     assert report.p50_ms == 50.0
     assert report.p95_ms == 95.0
@@ -218,6 +403,49 @@ def test_report_uses_nearest_rank_metrics_and_exact_release_thresholds() -> None
     assert HTTP_5XX_RATE_LIMIT == 0.005
 
 
+def test_slow_drain_cannot_pass_elapsed_or_throughput_contract() -> None:
+    config = LoadProfileConfig(port=8000, detail_id=_DETAIL_ID)
+    values = [observation(latency_ms=100.0) for _index in range(8_551)] + [
+        observation(latency_ms=100_000.0) for _index in range(449)
+    ]
+    report = build_report(
+        config,
+        completed_requests(values),
+        measurements=run_measurements(
+            elapsed_seconds=3_600.0,
+            observed_peak_active=MAX_CONCURRENCY,
+        ),
+    )
+
+    # A socket-level timeout cannot forcibly stop a trusted loopback slow-drip peer.
+    # Once it returns, however, the bounded elapsed gate makes false acceptance impossible.
+    assert report.p95_ms == 100.0
+    assert report.throughput_rps == 2.5
+    assert report.duration_contract_met is False
+    assert report.throughput_target_met is False
+    assert report.passed is False
+
+
+def test_report_rejects_observed_concurrency_above_configured_bound() -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        duration_seconds=1,
+        profile="smoke",
+    )
+    report = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(
+            elapsed_seconds=1.0,
+            observed_peak_active=MAX_CONCURRENCY + 1,
+        ),
+    )
+
+    assert report.concurrency_contract_met is False
+    assert report.passed is False
+
+
 @pytest.mark.parametrize("failure", ("latency", "http_5xx", "revision_mix", "error"))
 def test_report_fails_for_each_release_blocker(failure: str) -> None:
     config = LoadProfileConfig(
@@ -240,7 +468,11 @@ def test_report_fails_for_each_release_blocker(failure: str) -> None:
     else:
         values[-1] = observation(valid=False)
 
-    report = build_report(config, completed_requests(values), elapsed_seconds=1.0)
+    report = build_report(
+        config,
+        completed_requests(values),
+        measurements=run_measurements(elapsed_seconds=1.0),
+    )
 
     assert report.passed is False
     if failure == "http_5xx":
@@ -265,8 +497,16 @@ def test_report_applies_strict_five_xx_rate_without_treating_it_as_runner_error(
         observation(status_code=500, valid=False, revision_token=None) for _index in range(5)
     ]
 
-    passing = build_report(config, completed_requests(within_limit), elapsed_seconds=100.0)
-    failing = build_report(config, completed_requests(at_limit), elapsed_seconds=100.0)
+    passing = build_report(
+        config,
+        completed_requests(within_limit),
+        measurements=run_measurements(elapsed_seconds=100.0),
+    )
+    failing = build_report(
+        config,
+        completed_requests(at_limit),
+        measurements=run_measurements(elapsed_seconds=100.0),
+    )
 
     assert passing.http_5xx_rate == 0.004
     assert passing.error_count == 0
@@ -299,6 +539,11 @@ def test_http_request_bounds_body_and_requires_status_cache_and_revision_header(
         FakeResponse(url=url, status=204),
         FakeResponse(url=url, cache_control="public, max-age=60"),
         FakeResponse(url=url, revision_token=None),
+        FakeResponse(url=url, revision_token="c" * 63),
+        FakeResponse(url=url, revision_token="C" * 64),
+        FakeResponse(url=url, revision_token="g" * 64),
+        FakeResponse(url=url, revision_token=f" {_REVISION_TOKEN}"),
+        FakeResponse(url=url, revision_token=f"{_REVISION_TOKEN} "),
         FakeResponse(url="http://127.0.0.1:8000/changed"),
         FakeResponse(url=url, body=b"x" * (MAX_RESPONSE_BYTES + 1)),
     )
@@ -370,7 +615,7 @@ def test_main_prints_only_safe_json_and_uses_report_exit_status(
             duration_seconds=1,
         ),
         completed_requests([observation() for _index in range(10)]),
-        elapsed_seconds=1.0,
+        measurements=run_measurements(elapsed_seconds=1.0),
     )
     with patch("scripts.http_load_profile.run_profile", return_value=report):
         exit_code = main(config_args)


