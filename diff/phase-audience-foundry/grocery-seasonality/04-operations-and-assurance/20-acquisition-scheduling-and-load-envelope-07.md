## `fix(perf): validate supervised profile receipts`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index ae614b7..e3b9104 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -14,6 +14,7 @@ import json
 import math
 import os
 import re
+import selectors
 import signal
 import subprocess
 import sys
@@ -54,6 +55,7 @@ NOMINAL_REQUEST_INTERVAL_MS: Final = 1000.0 / REQUESTS_PER_SECOND
 # 90 ms after the prior actual submission.
 RECOVERY_FLOOR_INTERVAL_MS: Final = 90.0
 _CLOCK_COMPARISON_EPSILON_MS: Final = 0.001
+_RECEIPT_ROUNDING_HALF_UNIT: Final = 0.0005001
 
 _LOCAL_HOST: Final = "127.0.0.1"
 _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
@@ -71,9 +73,14 @@ _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
 _POSITIVE_INTEGER = re.compile(r"[1-9][0-9]*\Z")
 _REVISION_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
 _THREAD_STATE = threading.local()
-_CHILD_MODE: Final = "--internal-watchdog-child"
+_CHILD_BOOTSTRAP: Final = (
+    "import runpy,sys;"
+    "namespace=runpy.run_path(sys.argv[1]);"
+    "raise SystemExit(namespace['_child_main']())"
+)
 _MAX_CHILD_INPUT_CHARACTERS: Final = 4_096
-_MAX_CHILD_OUTPUT_CHARACTERS: Final = 32_768
+_MAX_CHILD_OUTPUT_BYTES: Final = 32_768
+_PROCESS_GROUP_POLL_INTERVAL_SECONDS: Final = 0.01
 
 type RequestKind = Literal["catalog", "detail"]
 type Requester = Callable[[str, int, float], "HttpObservation"]
@@ -979,19 +986,145 @@ def _validated_report_payload(
     completed_count = cast(int, counts["completed"])
     participated = cast(int, logical_users["participated"])
     observed_peak = cast(int, concurrency["observed_in_flight_peak"])
+    catalog_count = cast(int, counts["catalog_list_search"])
+    detail_count = cast(int, counts["detail"])
+    successful_count = cast(int, counts["successful"])
+    error_count = cast(int, counts["errors"])
+    http_5xx_count = cast(int, counts["http_5xx"])
+    elapsed_seconds = cast(float, timing["elapsed_seconds"])
+    throughput_rps = cast(float, timing["throughput_rps"])
+    minimum_accepted_throughput_rps = cast(float, timing["minimum_accepted_throughput_rps"])
+    p95_schedule_jitter_ms = cast(float, timing["p95_schedule_jitter_ms"])
+    minimum_inter_submission_ms = cast(float, timing["minimum_inter_submission_ms"])
+    burst_interval_violations = cast(int, timing["burst_interval_violations"])
+    elapsed_lower = max(0.0, elapsed_seconds - _RECEIPT_ROUNDING_HALF_UNIT)
+    elapsed_upper = elapsed_seconds + _RECEIPT_ROUNDING_HALF_UNIT
+    throughput_lower = max(0.0, throughput_rps - _RECEIPT_ROUNDING_HALF_UNIT)
+    throughput_upper = throughput_rps + _RECEIPT_ROUNDING_HALF_UNIT
+    if completed_count > 0:
+        if throughput_upper <= 0.0:
+            return None
+        throughput_elapsed_lower = completed_count / throughput_upper
+        throughput_elapsed_upper = (
+            completed_count / throughput_lower if throughput_lower > 0.0 else math.inf
+        )
+        feasible_elapsed_lower = max(elapsed_lower, throughput_elapsed_lower)
+        feasible_elapsed_upper = min(elapsed_upper, throughput_elapsed_upper)
+    else:
+        if not throughput_lower <= 0.0 <= throughput_upper:
+            return None
+        feasible_elapsed_lower = elapsed_lower
+        feasible_elapsed_upper = elapsed_upper
+    if feasible_elapsed_lower > feasible_elapsed_upper:
+        return None
+
+    expected_http_5xx_rate = round(
+        http_5xx_count / completed_count if completed_count else 1.0,
+        6,
+    )
+    raw_minimum_throughput = config.scheduled_requests / (
+        float(config.duration_seconds) + PROFILE_COMPLETION_GRACE_SECONDS
+    )
+    minimum_throughput_lower = max(
+        0.0,
+        minimum_accepted_throughput_rps - _RECEIPT_ROUNDING_HALF_UNIT,
+    )
+    minimum_throughput_upper = minimum_accepted_throughput_rps + _RECEIPT_ROUNDING_HALF_UNIT
+    if not minimum_throughput_lower <= raw_minimum_throughput <= minimum_throughput_upper:
+        return None
+
+    maximum_elapsed = float(config.duration_seconds) + PROFILE_COMPLETION_GRACE_SECONDS
+    duration_true_possible = bool(
+        feasible_elapsed_upper >= float(config.duration_seconds)
+        and feasible_elapsed_lower <= maximum_elapsed
+    )
+    duration_false_possible = bool(
+        feasible_elapsed_lower < float(config.duration_seconds)
+        or feasible_elapsed_upper > maximum_elapsed
+    )
+    throughput_true_possible = bool(throughput_upper >= raw_minimum_throughput)
+    throughput_false_possible = bool(throughput_lower < raw_minimum_throughput)
+    jitter_lower = max(
+        0.0,
+        p95_schedule_jitter_ms - _RECEIPT_ROUNDING_HALF_UNIT,
+    )
+    jitter_upper = p95_schedule_jitter_ms + _RECEIPT_ROUNDING_HALF_UNIT
+    jitter_true_possible = bool(jitter_lower <= P95_SCHEDULE_JITTER_LIMIT_MS)
+    jitter_false_possible = bool(jitter_upper > P95_SCHEDULE_JITTER_LIMIT_MS)
+    minimum_interval_lower = max(
+        0.0,
+        minimum_inter_submission_ms - _RECEIPT_ROUNDING_HALF_UNIT,
+    )
+    minimum_interval_upper = minimum_inter_submission_ms + _RECEIPT_ROUNDING_HALF_UNIT
+    no_burst_true_possible = bool(
+        burst_interval_violations == 0
+        and minimum_interval_upper + _CLOCK_COMPARISON_EPSILON_MS >= RECOVERY_FLOOR_INTERVAL_MS
+    )
+    no_burst_false_possible = bool(
+        burst_interval_violations != 0
+        or minimum_interval_lower + _CLOCK_COMPARISON_EPSILON_MS < RECOVERY_FLOOR_INTERVAL_MS
+    )
+    expected_concurrency_contract = bool(1 <= observed_peak <= MAX_CONCURRENCY)
+    expected_workload_contract = bool(
+        catalog_count == (config.scheduled_requests * 7) // 10
+        and detail_count == (config.scheduled_requests * 3) // 10
+    )
+    reported_duration_contract = cast(bool, timing["duration_contract_met"])
+    reported_throughput_contract = cast(bool, timing["throughput_target_met"])
+    reported_jitter_contract = cast(bool, timing["schedule_jitter_contract_met"])
+    reported_no_burst_contract = cast(bool, timing["no_burst_contract_met"])
+    reported_schedule_contract = cast(bool, timing["schedule_contract_met"])
+    reported_round_robin_contract = cast(
+        bool,
+        logical_users["round_robin_contract_met"],
+    )
+    base_pass_without_latency = bool(
+        completed_count == config.scheduled_requests
+        and expected_workload_contract
+        and error_count == 0
+        and cast(bool, report["revision_consistent"])
+        and expected_http_5xx_rate < HTTP_5XX_RATE_LIMIT
+        and reported_duration_contract
+        and reported_throughput_contract
+        and reported_schedule_contract
+        and reported_round_robin_contract
+        and expected_concurrency_contract
+    )
+    latency_p95_lower = max(
+        0.0,
+        cast(float, latency_ms["p95"]) - _RECEIPT_ROUNDING_HALF_UNIT,
+    )
+    latency_p95_upper = cast(float, latency_ms["p95"]) + _RECEIPT_ROUNDING_HALF_UNIT
+    pass_true_possible = bool(base_pass_without_latency and latency_p95_lower <= P95_LIMIT_MS)
+    pass_false_possible = bool(not base_pass_without_latency or latency_p95_upper > P95_LIMIT_MS)
+    reported_passed = cast(bool, report["passed"])
     if (
-        cast(int, counts["catalog_list_search"]) + cast(int, counts["detail"]) != completed_count
-        or cast(int, counts["successful"])
-        + cast(int, counts["errors"])
-        + cast(int, counts["http_5xx"])
-        != completed_count
-        or cast(float, report["http_5xx_rate"]) > 1.0
-        or cast(bool, concurrency["in_flight_within_limit"])
-        != (1 <= observed_peak <= MAX_CONCURRENCY)
+        catalog_count + detail_count != completed_count
+        or successful_count + error_count + http_5xx_count != completed_count
+        or cast(float, report["http_5xx_rate"]) != expected_http_5xx_rate
+        or not (
+            cast(float, latency_ms["p50"])
+            <= cast(float, latency_ms["p95"])
+            <= cast(float, latency_ms["max"])
+        )
+        or p95_schedule_jitter_ms > cast(float, timing["max_schedule_jitter_ms"])
+        or (reported_duration_contract and not duration_true_possible)
+        or (not reported_duration_contract and not duration_false_possible)
+        or (reported_throughput_contract and not throughput_true_possible)
+        or (not reported_throughput_contract and not throughput_false_possible)
+        or (reported_jitter_contract and not jitter_true_possible)
+        or (not reported_jitter_contract and not jitter_false_possible)
+        or (reported_no_burst_contract and not no_burst_true_possible)
+        or (not reported_no_burst_contract and not no_burst_false_possible)
+        or reported_schedule_contract != (reported_jitter_contract and reported_no_burst_contract)
+        or cast(bool, concurrency["in_flight_within_limit"]) != expected_concurrency_contract
+        or cast(bool, report["workload_consistent"]) != expected_workload_contract
         or (
-            cast(bool, logical_users["round_robin_contract_met"])
+            reported_round_robin_contract
             and participated != min(LOGICAL_VIRTUAL_USERS, config.scheduled_requests)
         )
+        or (reported_passed and not pass_true_possible)
+        or (not reported_passed and not pass_false_possible)
     ):
         return None
     return report
@@ -1004,7 +1137,7 @@ def _validated_child_output(
     config: LoadProfileConfig,
 ) -> str | None:
     if (
-        len(output) > _MAX_CHILD_OUTPUT_CHARACTERS
+        len(output) > _MAX_CHILD_OUTPUT_BYTES
         or not output.endswith("\n")
         or output.count("\n") != 1
     ):
@@ -1041,32 +1174,128 @@ def _validated_child_output(
     return _safe_failure(profile=config.label, code="RUNNER_FAILED")
 
 
-def _terminate_child(process: subprocess.Popen[str]) -> None:
-    if process.poll() is not None:
-        return
+class _ChildOutputLimitError(RuntimeError):
+    pass
+
+
+def _collect_child_output(
+    process: subprocess.Popen[bytes],
+    *,
+    encoded_arguments: bytes,
+    timeout_seconds: float,
+    monotonic: Callable[[], float] = time.monotonic,
+) -> tuple[str, int]:
+    if process.stdin is None or process.stdout is None:
+        raise RuntimeError
+    try:
+        process.stdin.write(encoded_arguments)
+    except BrokenPipeError:
+        pass
+    finally:
+        process.stdin.close()
+
+    deadline = monotonic() + timeout_seconds
+    output = bytearray()
     try:
-        os.killpg(process.pid, signal.SIGTERM)
-    except OSError, ProcessLookupError:
+        with selectors.DefaultSelector() as selector:
+            selector.register(process.stdout, selectors.EVENT_READ)
+            while True:
+                remaining = deadline - monotonic()
+                if remaining <= 0:
+                    raise subprocess.TimeoutExpired("load-profile-child", timeout_seconds)
+                if not selector.select(timeout=remaining):
+                    raise subprocess.TimeoutExpired("load-profile-child", timeout_seconds)
+                read_limit = min(
+                    8_192,
+                    (_MAX_CHILD_OUTPUT_BYTES + 1) - len(output),
+                )
+                chunk = os.read(process.stdout.fileno(), read_limit)
+                if not chunk:
+                    break
+                output.extend(chunk)
+                if len(output) > _MAX_CHILD_OUTPUT_BYTES:
+                    raise _ChildOutputLimitError
+    finally:
         try:
-            process.terminate()
+            process.stdout.close()
         except OSError:
-            return
+            pass
+
+    remaining = deadline - monotonic()
+    if remaining <= 0:
+        raise subprocess.TimeoutExpired("load-profile-child", timeout_seconds)
     try:
-        process.communicate(timeout=CLI_TERMINATION_GRACE_SECONDS)
-        return
+        return_code = process.wait(timeout=remaining)
     except subprocess.TimeoutExpired:
-        pass
-    except Exception:
-        return
+        raise subprocess.TimeoutExpired("load-profile-child", timeout_seconds) from None
+    try:
+        decoded_output = bytes(output).decode("ascii", errors="strict")
+    except UnicodeDecodeError:
+        raise _ChildOutputLimitError from None
+    return decoded_output, return_code
+
+
+def _process_group_exists(process_group_id: int) -> bool:
+    try:
+        os.killpg(process_group_id, 0)
+    except ProcessLookupError:
+        return False
+    except PermissionError:
+        return True
+    except OSError:
+        return True
+    return True
+
+
+def _signal_process_group(
+    process: subprocess.Popen[bytes],
+    selected_signal: signal.Signals,
+) -> None:
     try:
-        os.killpg(process.pid, signal.SIGKILL)
-    except OSError, ProcessLookupError:
+        os.killpg(process.pid, selected_signal)
+    except ProcessLookupError:
+        return
+    except OSError:
         try:
-            process.kill()
+            if selected_signal == signal.SIGTERM:
+                process.terminate()
+            else:
+                process.kill()
         except OSError:
             return
+
+
+def _terminate_child(
+    process: subprocess.Popen[bytes],
+    *,
+    monotonic: Callable[[], float] = time.monotonic,
+    sleeper: Callable[[float], None] = time.sleep,
+) -> None:
+    _signal_process_group(process, signal.SIGTERM)
+    deadline = monotonic() + CLI_TERMINATION_GRACE_SECONDS
+    process.poll()
+    while _process_group_exists(process.pid):
+        remaining = deadline - monotonic()
+        if remaining <= 0:
+            _signal_process_group(process, signal.SIGKILL)
+            break
+        sleeper(min(_PROCESS_GROUP_POLL_INTERVAL_SECONDS, remaining))
+        process.poll()
     try:
-        process.communicate(timeout=CLI_TERMINATION_GRACE_SECONDS)
+        process.wait(timeout=CLI_TERMINATION_GRACE_SECONDS)
+    except OSError, subprocess.TimeoutExpired:
+        pass
+    for pipe in (process.stdin, process.stdout):
+        if pipe is not None and not pipe.closed:
+            try:
+                pipe.close()
+            except OSError:
+                pass
+
+
+def _terminate_child_safely(process: subprocess.Popen[bytes]) -> None:
+    try:
+        _terminate_child(process)
     except Exception:
         return
 
@@ -1079,16 +1308,28 @@ def supervised_main(arguments: list[str] | None = None) -> int:
         print(_safe_failure(profile="UNAVAILABLE", code="CONFIG_INVALID"))
         return 2
 
-    process: subprocess.Popen[str] | None = None
+    encoded_arguments = json.dumps(
+        selected_arguments,
+        ensure_ascii=True,
+        separators=(",", ":"),
+    ).encode("ascii")
+    if len(encoded_arguments) > _MAX_CHILD_INPUT_CHARACTERS:
+        print(_safe_failure(profile="UNAVAILABLE", code="CONFIG_INVALID"))
+        return 2
+
+    process: subprocess.Popen[bytes] | None = None
     try:
         process = subprocess.Popen(  # noqa: S603 - fixed interpreter/script command.
-            [sys.executable, os.path.abspath(__file__), _CHILD_MODE],
+            [
+                sys.executable,
+                "-c",
+                _CHILD_BOOTSTRAP,
+                os.path.abspath(__file__),
+            ],
             stdin=subprocess.PIPE,
             stdout=subprocess.PIPE,
             stderr=subprocess.DEVNULL,
-            text=True,
-            encoding="utf-8",
-            errors="strict",
+            bufsize=0,
             close_fds=True,
             start_new_session=True,
             env={
@@ -1097,40 +1338,45 @@ def supervised_main(arguments: list[str] | None = None) -> int:
                 "PYTHONUTF8": "1",
             },
         )
-        output, _discarded_stderr = process.communicate(
-            input=json.dumps(
-                selected_arguments,
-                ensure_ascii=True,
-                separators=(",", ":"),
-            ),
-            timeout=float(config.duration_seconds) + CLI_WATCHDOG_GRACE_SECONDS,
+        output, return_code = _collect_child_output(
+            process,
+            encoded_arguments=encoded_arguments,
+            timeout_seconds=float(config.duration_seconds) + CLI_WATCHDOG_GRACE_SECONDS,
         )
     except subprocess.TimeoutExpired:
         if process is not None:
-            _terminate_child(process)
+            _terminate_child_safely(process)
         print(_safe_failure(profile=config.label, code="WATCHDOG_TIMEOUT"))
         return 2
     except KeyboardInterrupt:
         if process is not None:
-            _terminate_child(process)
+            _terminate_child_safely(process)
         print(_safe_failure(profile=config.label, code="WATCHDOG_INTERRUPTED"))
         return 130
+    except _ChildOutputLimitError:
+        if process is not None:
+            _terminate_child_safely(process)
+        print(_safe_failure(profile=config.label, code="CHILD_RESULT_INVALID"))
+        return 2
     except Exception:
         if process is not None:
-            _terminate_child(process)
+            _terminate_child_safely(process)
         print(_safe_failure(profile=config.label, code="SUPERVISOR_FAILED"))
         return 2
 
-    safe_output = _validated_child_output(
-        output,
-        return_code=process.returncode,
-        config=config,
-    )
+    try:
+        safe_output = _validated_child_output(
+            output,
+            return_code=return_code,
+            config=config,
+        )
+    except Exception:
+        safe_output = None
     if safe_output is None:
         print(_safe_failure(profile=config.label, code="CHILD_RESULT_INVALID"))
         return 2
     print(safe_output)
-    return cast(int, process.returncode)
+    return return_code
 
 
 def _child_main() -> int:
@@ -1152,7 +1398,10 @@ def _child_main() -> int:
     return main(arguments)
 
 
+def _entrypoint(arguments: list[str] | None = None) -> int:
+    selected_arguments = list(sys.argv[1:] if arguments is None else arguments)
+    return supervised_main(selected_arguments)
+
+
 if __name__ == "__main__":
-    if sys.argv[1:] == [_CHILD_MODE]:
-        raise SystemExit(_child_main())
-    raise SystemExit(supervised_main())
+    raise SystemExit(_entrypoint())
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index f204cdc..89d69e1 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -2,8 +2,10 @@ from __future__ import annotations
 
 import io
 import json
+import runpy
 import signal
 import subprocess
+import sys
 import threading
 import time
 import uuid
@@ -19,6 +21,7 @@ from urllib.parse import urlsplit
 import pytest
 
 from scripts.http_load_profile import (
+    _CHILD_BOOTSTRAP,
     CLI_TERMINATION_GRACE_SECONDS,
     CLI_WATCHDOG_GRACE_SECONDS,
     HTTP_5XX_RATE_LIMIT,
@@ -38,9 +41,14 @@ from scripts.http_load_profile import (
     LoadProfileError,
     RunMeasurements,
     _ActiveRequestCounter,
+    _ChildOutputLimitError,
+    _collect_child_output,
+    _entrypoint,
     _execute_scheduled_request,
     _NoRedirectHandler,
+    _terminate_child,
     _validate_local_url,
+    _validated_child_output,
     build_report,
     http_request,
     main,
@@ -904,10 +912,14 @@ def test_supervised_cli_reprints_only_a_validated_child_receipt(
         measurements=run_measurements(elapsed_seconds=1.0),
     )
     child = MagicMock()
-    child.communicate.return_value = (f"{report.render()}\n", None)
-    child.returncode = 0
 
-    with patch("scripts.http_load_profile.subprocess.Popen", return_value=child) as popen:
+    with (
+        patch("scripts.http_load_profile.subprocess.Popen", return_value=child) as popen,
+        patch(
+            "scripts.http_load_profile._collect_child_output",
+            return_value=(f"{report.render()}\n", 0),
+        ) as collect_child_output,
+    ):
         exit_code = supervised_main(arguments)
 
     output = capsys.readouterr().out
@@ -925,7 +937,7 @@ def test_supervised_cli_reprints_only_a_validated_child_receipt(
         "PYTHONUNBUFFERED",
         "PYTHONUTF8",
     }
-    assert child.communicate.call_args.kwargs["timeout"] == pytest.approx(
+    assert collect_child_output.call_args.kwargs["timeout_seconds"] == pytest.approx(
         1.0 + CLI_WATCHDOG_GRACE_SECONDS
     )
 
@@ -946,25 +958,16 @@ def test_supervised_cli_kills_stubborn_child_and_redacts_timeout_details(
     private_output = f"private={_DETAIL_ID};revision={_REVISION_TOKEN}"
     child = MagicMock()
     child.pid = 4_321
-    child.returncode = None
-    child.poll.return_value = None
-    child.communicate.side_effect = (
-        subprocess.TimeoutExpired(
-            cmd=("private-command", str(_DETAIL_ID)),
-            timeout=1.0 + CLI_WATCHDOG_GRACE_SECONDS,
-            output=private_output,
-        ),
-        subprocess.TimeoutExpired(
-            cmd="private-command",
-            timeout=CLI_TERMINATION_GRACE_SECONDS,
-            output=private_output,
-        ),
-        (private_output, None),
+    timeout = subprocess.TimeoutExpired(
+        cmd=("private-command", str(_DETAIL_ID)),
+        timeout=1.0 + CLI_WATCHDOG_GRACE_SECONDS,
+        output=private_output,
     )
 
     with (
         patch("scripts.http_load_profile.subprocess.Popen", return_value=child),
-        patch("scripts.http_load_profile.os.killpg") as kill_process_group,
+        patch("scripts.http_load_profile._collect_child_output", side_effect=timeout),
+        patch("scripts.http_load_profile._terminate_child") as terminate_child,
     ):
         exit_code = supervised_main(arguments)
 
@@ -979,10 +982,7 @@ def test_supervised_cli_kills_stubborn_child_and_redacts_timeout_details(
     assert str(_DETAIL_ID) not in output
     assert _REVISION_TOKEN not in output
     assert "private-command" not in output
-    assert kill_process_group.call_args_list == [
-        ((4_321, signal.SIGTERM),),
-        ((4_321, signal.SIGKILL),),
-    ]
+    terminate_child.assert_called_once_with(child)
 
 
 def test_supervised_cli_redacts_process_creation_failure(
@@ -1015,6 +1015,45 @@ def test_supervised_cli_redacts_process_creation_failure(
     assert _REVISION_TOKEN not in output
 
 
+def test_supervised_cli_redacts_unexpected_validator_failure(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    arguments = [
+        "--port",
+        "8000",
+        "--detail-id",
+        str(_DETAIL_ID),
+        "--profile",
+        "smoke",
+        "--duration-seconds",
+        "1",
+    ]
+    private_error = RuntimeError(f"validator failed {_DETAIL_ID} {_REVISION_TOKEN}")
+
+    with (
+        patch("scripts.http_load_profile.subprocess.Popen", return_value=MagicMock()),
+        patch(
+            "scripts.http_load_profile._collect_child_output",
+            return_value=("{}\n", 0),
+        ),
+        patch(
+            "scripts.http_load_profile._validated_child_output",
+            side_effect=private_error,
+        ),
+    ):
+        exit_code = supervised_main(arguments)
+
+    output = capsys.readouterr().out
+    assert exit_code == 2
+    assert json.loads(output) == {
+        "error": "CHILD_RESULT_INVALID",
+        "passed": False,
+        "profile": "SMOKE_NON_ACCEPTANCE",
+    }
+    assert str(_DETAIL_ID) not in output
+    assert _REVISION_TOKEN not in output
+
+
 @pytest.mark.parametrize(
     "child_output",
     (
@@ -1038,10 +1077,14 @@ def test_supervised_cli_rejects_unvalidated_child_output_without_reflection(
         "1",
     ]
     child = MagicMock()
-    child.communicate.return_value = (child_output, None)
-    child.returncode = 0
 
-    with patch("scripts.http_load_profile.subprocess.Popen", return_value=child):
+    with (
+        patch("scripts.http_load_profile.subprocess.Popen", return_value=child),
+        patch(
+            "scripts.http_load_profile._collect_child_output",
+            return_value=(child_output, 0),
+        ),
+    ):
         exit_code = supervised_main(arguments)
 
     output = capsys.readouterr().out
@@ -1055,3 +1098,255 @@ def test_supervised_cli_rejects_unvalidated_child_output_without_reflection(
     assert "token=value" not in output
     assert str(_DETAIL_ID) not in output
     assert "민감한 오류" not in output
+
+
+def test_parent_recomputes_derived_gates_before_accepting_child_receipt() -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        profile="smoke",
+        duration_seconds=1,
+    )
+    report = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(elapsed_seconds=1.0),
+    )
+    variants: list[dict[str, object]] = []
+    for mutation in ("throughput", "counts", "latency"):
+        payload = json.loads(report.render())
+        if mutation == "throughput":
+            payload["timing"]["throughput_rps"] = 0.0
+        elif mutation == "counts":
+            payload["counts"].update(
+                {
+                    "completed": 0,
+                    "catalog_list_search": 0,
+                    "detail": 0,
+                    "successful": 0,
+                }
+            )
+            payload["timing"]["throughput_rps"] = 0.0
+        else:
+            payload["latency_ms"]["p95"] = P95_LIMIT_MS + 1.0
+            payload["latency_ms"]["max"] = P95_LIMIT_MS + 1.0
+        variants.append(payload)
+
+    for payload in variants:
+        encoded = json.dumps(
+            payload,
+            ensure_ascii=True,
+            separators=(",", ":"),
+            sort_keys=True,
+        )
+        assert (
+            _validated_child_output(
+                f"{encoded}\n",
+                return_code=0,
+                config=config,
+            )
+            is None
+        )
+
+
+def test_parent_accepts_genuine_receipts_across_three_decimal_rounding_boundaries() -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        profile="smoke",
+        duration_seconds=1,
+    )
+    passing_after_small_overrun = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(elapsed_seconds=1.0002),
+    )
+    passing_with_distinct_rounded_throughput = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(elapsed_seconds=1.0038),
+    )
+    failing_after_completion_boundary = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(elapsed_seconds=4.0004),
+    )
+    failing_after_jitter_boundary = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(
+            elapsed_seconds=1.0,
+            p95_schedule_jitter_ms=P95_SCHEDULE_JITTER_LIMIT_MS + 0.0004,
+            max_schedule_jitter_ms=P95_SCHEDULE_JITTER_LIMIT_MS + 0.0004,
+        ),
+    )
+    failing_after_latency_boundary = build_report(
+        config,
+        completed_requests(
+            [observation() for _index in range(9)] + [observation(latency_ms=P95_LIMIT_MS + 0.0004)]
+        ),
+        measurements=run_measurements(elapsed_seconds=1.0),
+    )
+
+    assert passing_after_small_overrun.elapsed_seconds == 1.0
+    assert passing_after_small_overrun.throughput_rps == 9.998
+    assert passing_after_small_overrun.passed is True
+    assert passing_with_distinct_rounded_throughput.elapsed_seconds == 1.004
+    assert passing_with_distinct_rounded_throughput.throughput_rps == 9.962
+    assert passing_with_distinct_rounded_throughput.passed is True
+    cases = (
+        (passing_after_small_overrun, 0),
+        (passing_with_distinct_rounded_throughput, 0),
+        (failing_after_completion_boundary, 1),
+        (failing_after_jitter_boundary, 1),
+        (failing_after_latency_boundary, 1),
+    )
+    for report, return_code in cases:
+        assert (
+            _validated_child_output(
+                f"{report.render()}\n",
+                return_code=return_code,
+                config=config,
+            )
+            == report.render()
+        )
+
+
+def test_streaming_child_output_enforces_hard_byte_cap_before_buffering_all() -> None:
+    process = subprocess.Popen(  # noqa: S603 - fixed local test interpreter command.
+        [
+            sys.executable,
+            "-c",
+            "import sys; sys.stdout.buffer.write(b'x' * 40000)",
+        ],
+        stdin=subprocess.PIPE,
+        stdout=subprocess.PIPE,
+        stderr=subprocess.DEVNULL,
+        bufsize=0,
+        start_new_session=True,
+    )
+    try:
+        with pytest.raises(_ChildOutputLimitError):
+            _collect_child_output(
+                process,
+                encoded_arguments=b"[]",
+                timeout_seconds=2.0,
+            )
+    finally:
+        _terminate_child(process)
+
+
+def test_streaming_child_output_enforces_deadline_without_waiting_for_eof() -> None:
+    process = MagicMock()
+    process.stdin = MagicMock()
+    process.stdout = MagicMock()
+    clock = FakeClock()
+    selector = MagicMock()
+    selector.__enter__.return_value = selector
+
+    def expire_selection(timeout: float) -> list[object]:
+        clock.sleep(timeout)
+        return []
+
+    selector.select.side_effect = expire_selection
+    with patch("scripts.http_load_profile.selectors.DefaultSelector", return_value=selector):
+        with pytest.raises(subprocess.TimeoutExpired):
+            _collect_child_output(
+                process,
+                encoded_arguments=b"[]",
+                timeout_seconds=0.05,
+                monotonic=clock.monotonic,
+            )
+
+    assert clock.value == pytest.approx(0.05)
+    process.stdin.write.assert_called_once_with(b"[]")
+    process.stdin.close.assert_called_once_with()
+    process.stdout.close.assert_called_once_with()
+
+
+def test_termination_kills_surviving_process_group_even_when_leader_exited() -> None:
+    process = MagicMock()
+    process.pid = 4_321
+    process.stdin = None
+    process.stdout = None
+    process.poll.return_value = 0
+    process.wait.return_value = 0
+    clock = FakeClock()
+
+    with (
+        patch("scripts.http_load_profile._process_group_exists", return_value=True),
+        patch("scripts.http_load_profile._signal_process_group") as signal_group,
+    ):
+        _terminate_child(
+            process,
+            monotonic=clock.monotonic,
+            sleeper=clock.sleep,
+        )
+
+    assert signal_group.call_count == 2
+    signal_group.assert_any_call(process, signal.SIGTERM)
+    signal_group.assert_any_call(process, signal.SIGKILL)
+    assert clock.value == pytest.approx(CLI_TERMINATION_GRACE_SECONDS)
+
+
+def test_entrypoint_routes_external_cli_to_supervisor() -> None:
+    external_arguments = ["--port", "invalid-private-value"]
+    with patch("scripts.http_load_profile.supervised_main", return_value=7) as supervisor:
+        assert _entrypoint(external_arguments) == 7
+        supervisor.assert_called_once_with(external_arguments)
+
+
+def test_real_child_bootstrap_reads_stdin_without_traceback_or_argument_reflection() -> None:
+    script_path = str(__file__).replace("tests/test_http_load_profile.py", "http_load_profile.py")
+    result = subprocess.run(  # noqa: S603 - fixed local test interpreter/script command.
+        [sys.executable, "-c", _CHILD_BOOTSTRAP, script_path],
+        input=b"[]",
+        capture_output=True,
+        timeout=2.0,
+        check=False,
+    )
+
+    assert result.returncode == 2
+    assert json.loads(result.stdout) == {
+        "error": "CONFIG_INVALID",
+        "passed": False,
+        "profile": "UNAVAILABLE",
+    }
+    assert result.stdout.count(b"\n") == 1
+    assert result.stderr == b""
+
+
+def test_real_main_module_dispatches_external_cli_through_supervisor(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    script_path = str(__file__).replace("tests/test_http_load_profile.py", "http_load_profile.py")
+    arguments = [
+        script_path,
+        "--port",
+        "8000",
+        "--detail-id",
+        str(_DETAIL_ID),
+        "--profile",
+        "smoke",
+        "--duration-seconds",
+        "1",
+    ]
+    private_error = OSError(f"process failed {_DETAIL_ID} {_REVISION_TOKEN}")
+
+    with (
+        patch.object(sys, "argv", arguments),
+        patch("subprocess.Popen", side_effect=private_error),
+        pytest.raises(SystemExit) as caught,
+    ):
+        runpy.run_path(script_path, run_name="__main__")
+
+    output = capsys.readouterr().out
+    assert caught.value.code == 2
+    assert json.loads(output) == {
+        "error": "SUPERVISOR_FAILED",
+        "passed": False,
+        "profile": "SMOKE_NON_ACCEPTANCE",
+    }
+    assert output.count("\n") == 1
+    assert str(_DETAIL_ID) not in output
+    assert _REVISION_TOKEN not in output


