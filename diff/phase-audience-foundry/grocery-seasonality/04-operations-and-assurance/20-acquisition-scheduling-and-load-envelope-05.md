## `fix(perf): bound profile process lifetime`

diff --git a/scripts/http_load_profile.py b/scripts/http_load_profile.py
index 06d1c4a..ae614b7 100644
--- a/scripts/http_load_profile.py
+++ b/scripts/http_load_profile.py
@@ -2,9 +2,9 @@
 
 The runner deliberately reaches only a trusted loopback process.  A standard-library
 socket timeout cannot forcibly stop a peer that keeps slowly producing bytes, so the
-worker pool may take longer than the nominal profile to return in that pathological
-case.  Such a run cannot pass: queue time is included in end-to-end latency and total
-elapsed time must remain inside the fixed completion grace.
+external CLI supervises the worker in a separate process and terminates its process
+group at a fixed deadline.  Queue time remains part of end-to-end latency and the
+child report retains the stricter elapsed-time acceptance gate.
 """
 
 from __future__ import annotations
@@ -12,7 +12,10 @@ from __future__ import annotations
 import argparse
 import json
 import math
+import os
 import re
+import signal
+import subprocess
 import sys
 import threading
 import time
@@ -21,7 +24,7 @@ from collections.abc import Callable
 from concurrent.futures import Executor, Future, ThreadPoolExecutor
 from contextlib import ExitStack
 from dataclasses import dataclass, field
-from typing import Any, Final, Literal, Never
+from typing import Any, Final, Literal, Never, cast
 from urllib.error import HTTPError
 from urllib.parse import urlencode, urlsplit
 from urllib.request import (
@@ -42,6 +45,10 @@ P95_LIMIT_MS: Final = 500.0
 HTTP_5XX_RATE_LIMIT: Final = 0.005
 P95_SCHEDULE_JITTER_LIMIT_MS: Final = 100.0
 PROFILE_COMPLETION_GRACE_SECONDS: Final = 3.0
+# The child report still has the stricter three-second completion gate above.  The
+# process watchdog adds only enough fixed headroom for interpreter startup and IPC.
+CLI_WATCHDOG_GRACE_SECONDS: Final = 5.0
+CLI_TERMINATION_GRACE_SECONDS: Final = 1.0
 NOMINAL_REQUEST_INTERVAL_MS: Final = 1000.0 / REQUESTS_PER_SECOND
 # Recover scheduler stalls by at most 10 ms per request; never submit less than
 # 90 ms after the prior actual submission.
@@ -64,6 +71,9 @@ _WORKLOAD_SLOTS: Final[tuple[str, ...]] = (
 _POSITIVE_INTEGER = re.compile(r"[1-9][0-9]*\Z")
 _REVISION_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
 _THREAD_STATE = threading.local()
+_CHILD_MODE: Final = "--internal-watchdog-child"
+_MAX_CHILD_INPUT_CHARACTERS: Final = 4_096
+_MAX_CHILD_OUTPUT_CHARACTERS: Final = 32_768
 
 type RequestKind = Literal["catalog", "detail"]
 type Requester = Callable[[str, int, float], "HttpObservation"]
@@ -772,16 +782,20 @@ def _safe_failure(*, profile: str, code: str) -> str:
     )
 
 
+def _config_from_arguments(arguments: list[str]) -> LoadProfileConfig:
+    options = _parser().parse_args(arguments)
+    return LoadProfileConfig(
+        port=options.port,
+        detail_id=options.detail_id,
+        duration_seconds=options.duration_seconds,
+        profile=options.profile,
+    )
+
+
 def main(arguments: list[str] | None = None) -> int:
     selected_arguments = sys.argv[1:] if arguments is None else arguments
     try:
-        options = _parser().parse_args(selected_arguments)
-        config = LoadProfileConfig(
-            port=options.port,
-            detail_id=options.detail_id,
-            duration_seconds=options.duration_seconds,
-            profile=options.profile,
-        )
+        config = _config_from_arguments(selected_arguments)
     except LoadProfileError:
         print(_safe_failure(profile="UNAVAILABLE", code="CONFIG_INVALID"))
         return 2
@@ -794,5 +808,351 @@ def main(arguments: list[str] | None = None) -> int:
     return 0 if report.passed else 1
 
 
+def _exact_typed_object(
+    value: object,
+    expected_types: dict[str, type[Any]],
+) -> dict[str, object] | None:
+    if type(value) is not dict:
+        return None
+    selected = cast(dict[str, object], value)
+    if set(selected) != set(expected_types):
+        return None
+    if any(type(selected[name]) is not expected for name, expected in expected_types.items()):
+        return None
+    return selected
+
+
+def _nonnegative_finite_float(value: object) -> bool:
+    return bool(type(value) is float and math.isfinite(value) and 0.0 <= value <= 1_000_000_000.0)
+
+
+def _bounded_nonnegative_integer(value: object, *, maximum: int) -> bool:
+    return bool(type(value) is int and 0 <= value <= maximum)
+
+
+def _validated_report_payload(
+    value: object,
+    *,
+    config: LoadProfileConfig,
+) -> dict[str, object] | None:
+    report = _exact_typed_object(
+        value,
+        {
+            "profile": str,
+            "duration_seconds": int,
+            "target_requests_per_second": int,
+            "logical_users": dict,
+            "concurrency": dict,
+            "counts": dict,
+            "latency_ms": dict,
+            "http_5xx_rate": float,
+            "timing": dict,
+            "workload_consistent": bool,
+            "revision_consistent": bool,
+            "passed": bool,
+        },
+    )
+    if report is None:
+        return None
+    logical_users = _exact_typed_object(
+        report["logical_users"],
+        {
+            "configured": int,
+            "participated": int,
+            "round_robin_contract_met": bool,
+        },
+    )
+    concurrency = _exact_typed_object(
+        report["concurrency"],
+        {
+            "in_flight_limit": int,
+            "observed_in_flight_peak": int,
+            "in_flight_within_limit": bool,
+        },
+    )
+    counts = _exact_typed_object(
+        report["counts"],
+        {
+            "scheduled": int,
+            "completed": int,
+            "catalog_list_search": int,
+            "detail": int,
+            "successful": int,
+            "errors": int,
+            "http_5xx": int,
+        },
+    )
+    latency_ms = _exact_typed_object(
+        report["latency_ms"],
+        {"p50": float, "p95": float, "max": float},
+    )
+    timing = _exact_typed_object(
+        report["timing"],
+        {
+            "elapsed_seconds": float,
+            "completion_grace_seconds": float,
+            "duration_contract_met": bool,
+            "throughput_rps": float,
+            "minimum_accepted_throughput_rps": float,
+            "throughput_target_met": bool,
+            "p95_schedule_jitter_ms": float,
+            "max_schedule_jitter_ms": float,
+            "p95_schedule_jitter_limit_ms": float,
+            "schedule_jitter_contract_met": bool,
+            "minimum_inter_submission_ms": float,
+            "nominal_request_interval_ms": float,
+            "recovery_floor_interval_ms": float,
+            "burst_interval_violations": int,
+            "no_burst_contract_met": bool,
+            "schedule_contract_met": bool,
+        },
+    )
+    if logical_users is None:
+        return None
+    if concurrency is None:
+        return None
+    if counts is None:
+        return None
+    if latency_ms is None:
+        return None
+    if timing is None:
+        return None
+
+    if (
+        report["profile"] != config.label
+        or report["duration_seconds"] != config.duration_seconds
+        or report["target_requests_per_second"] != REQUESTS_PER_SECOND
+        or logical_users["configured"] != LOGICAL_VIRTUAL_USERS
+        or concurrency["in_flight_limit"] != MAX_CONCURRENCY
+        or counts["scheduled"] != config.scheduled_requests
+        or timing["completion_grace_seconds"] != PROFILE_COMPLETION_GRACE_SECONDS
+        or timing["p95_schedule_jitter_limit_ms"] != P95_SCHEDULE_JITTER_LIMIT_MS
+        or timing["nominal_request_interval_ms"] != NOMINAL_REQUEST_INTERVAL_MS
+        or timing["recovery_floor_interval_ms"] != RECOVERY_FLOOR_INTERVAL_MS
+    ):
+        return None
+
+    integer_values = [
+        logical_users["participated"],
+        concurrency["observed_in_flight_peak"],
+        counts["completed"],
+        counts["catalog_list_search"],
+        counts["detail"],
+        counts["successful"],
+        counts["errors"],
+        counts["http_5xx"],
+        timing["burst_interval_violations"],
+    ]
+    if not all(
+        _bounded_nonnegative_integer(value, maximum=config.scheduled_requests)
+        for value in integer_values
+    ):
+        return None
+    if not _bounded_nonnegative_integer(
+        logical_users["participated"],
+        maximum=LOGICAL_VIRTUAL_USERS,
+    ) or not _bounded_nonnegative_integer(
+        concurrency["observed_in_flight_peak"],
+        maximum=MAX_CONCURRENCY,
+    ):
+        return None
+
+    float_values = [
+        report["http_5xx_rate"],
+        latency_ms["p50"],
+        latency_ms["p95"],
+        latency_ms["max"],
+        timing["elapsed_seconds"],
+        timing["completion_grace_seconds"],
+        timing["throughput_rps"],
+        timing["minimum_accepted_throughput_rps"],
+        timing["p95_schedule_jitter_ms"],
+        timing["max_schedule_jitter_ms"],
+        timing["p95_schedule_jitter_limit_ms"],
+        timing["minimum_inter_submission_ms"],
+        timing["nominal_request_interval_ms"],
+        timing["recovery_floor_interval_ms"],
+    ]
+    if not all(_nonnegative_finite_float(value) for value in float_values):
+        return None
+
+    completed_count = cast(int, counts["completed"])
+    participated = cast(int, logical_users["participated"])
+    observed_peak = cast(int, concurrency["observed_in_flight_peak"])
+    if (
+        cast(int, counts["catalog_list_search"]) + cast(int, counts["detail"]) != completed_count
+        or cast(int, counts["successful"])
+        + cast(int, counts["errors"])
+        + cast(int, counts["http_5xx"])
+        != completed_count
+        or cast(float, report["http_5xx_rate"]) > 1.0
+        or cast(bool, concurrency["in_flight_within_limit"])
+        != (1 <= observed_peak <= MAX_CONCURRENCY)
+        or (
+            cast(bool, logical_users["round_robin_contract_met"])
+            and participated != min(LOGICAL_VIRTUAL_USERS, config.scheduled_requests)
+        )
+    ):
+        return None
+    return report
+
+
+def _validated_child_output(
+    output: str,
+    *,
+    return_code: int | None,
+    config: LoadProfileConfig,
+) -> str | None:
+    if (
+        len(output) > _MAX_CHILD_OUTPUT_CHARACTERS
+        or not output.endswith("\n")
+        or output.count("\n") != 1
+    ):
+        return None
+    try:
+        output.encode("ascii")
+        decoded: object = json.loads(output[:-1])
+    except UnicodeEncodeError, json.JSONDecodeError:
+        return None
+
+    report = _validated_report_payload(decoded, config=config)
+    if report is not None:
+        if return_code not in {0, 1} or cast(bool, report["passed"]) != (return_code == 0):
+            return None
+        return json.dumps(
+            report,
+            ensure_ascii=True,
+            separators=(",", ":"),
+            sort_keys=True,
+        )
+
+    failure = _exact_typed_object(
+        decoded,
+        {"error": str, "passed": bool, "profile": str},
+    )
+    if (
+        failure is None
+        or failure["error"] != "RUNNER_FAILED"
+        or failure["passed"] is not False
+        or failure["profile"] != config.label
+        or return_code != 2
+    ):
+        return None
+    return _safe_failure(profile=config.label, code="RUNNER_FAILED")
+
+
+def _terminate_child(process: subprocess.Popen[str]) -> None:
+    if process.poll() is not None:
+        return
+    try:
+        os.killpg(process.pid, signal.SIGTERM)
+    except OSError, ProcessLookupError:
+        try:
+            process.terminate()
+        except OSError:
+            return
+    try:
+        process.communicate(timeout=CLI_TERMINATION_GRACE_SECONDS)
+        return
+    except subprocess.TimeoutExpired:
+        pass
+    except Exception:
+        return
+    try:
+        os.killpg(process.pid, signal.SIGKILL)
+    except OSError, ProcessLookupError:
+        try:
+            process.kill()
+        except OSError:
+            return
+    try:
+        process.communicate(timeout=CLI_TERMINATION_GRACE_SECONDS)
+    except Exception:
+        return
+
+
+def supervised_main(arguments: list[str] | None = None) -> int:
+    selected_arguments = list(sys.argv[1:] if arguments is None else arguments)
+    try:
+        config = _config_from_arguments(selected_arguments)
+    except LoadProfileError:
+        print(_safe_failure(profile="UNAVAILABLE", code="CONFIG_INVALID"))
+        return 2
+
+    process: subprocess.Popen[str] | None = None
+    try:
+        process = subprocess.Popen(  # noqa: S603 - fixed interpreter/script command.
+            [sys.executable, os.path.abspath(__file__), _CHILD_MODE],
+            stdin=subprocess.PIPE,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.DEVNULL,
+            text=True,
+            encoding="utf-8",
+            errors="strict",
+            close_fds=True,
+            start_new_session=True,
+            env={
+                "PYTHONIOENCODING": "utf-8",
+                "PYTHONUNBUFFERED": "1",
+                "PYTHONUTF8": "1",
+            },
+        )
+        output, _discarded_stderr = process.communicate(
+            input=json.dumps(
+                selected_arguments,
+                ensure_ascii=True,
+                separators=(",", ":"),
+            ),
+            timeout=float(config.duration_seconds) + CLI_WATCHDOG_GRACE_SECONDS,
+        )
+    except subprocess.TimeoutExpired:
+        if process is not None:
+            _terminate_child(process)
+        print(_safe_failure(profile=config.label, code="WATCHDOG_TIMEOUT"))
+        return 2
+    except KeyboardInterrupt:
+        if process is not None:
+            _terminate_child(process)
+        print(_safe_failure(profile=config.label, code="WATCHDOG_INTERRUPTED"))
+        return 130
+    except Exception:
+        if process is not None:
+            _terminate_child(process)
+        print(_safe_failure(profile=config.label, code="SUPERVISOR_FAILED"))
+        return 2
+
+    safe_output = _validated_child_output(
+        output,
+        return_code=process.returncode,
+        config=config,
+    )
+    if safe_output is None:
+        print(_safe_failure(profile=config.label, code="CHILD_RESULT_INVALID"))
+        return 2
+    print(safe_output)
+    return cast(int, process.returncode)
+
+
+def _child_main() -> int:
+    try:
+        encoded_arguments = sys.stdin.read(_MAX_CHILD_INPUT_CHARACTERS + 1)
+        if len(encoded_arguments) > _MAX_CHILD_INPUT_CHARACTERS:
+            raise ValueError
+        decoded_arguments: object = json.loads(encoded_arguments)
+        if (
+            type(decoded_arguments) is not list
+            or len(decoded_arguments) > 16
+            or any(type(value) is not str or len(value) > 512 for value in decoded_arguments)
+        ):
+            raise ValueError
+        arguments = cast(list[str], decoded_arguments)
+    except Exception:
+        print(_safe_failure(profile="UNAVAILABLE", code="CONFIG_INVALID"))
+        return 2
+    return main(arguments)
+
+
 if __name__ == "__main__":
-    raise SystemExit(main())
+    if sys.argv[1:] == [_CHILD_MODE]:
+        raise SystemExit(_child_main())
+    raise SystemExit(supervised_main())
diff --git a/scripts/tests/test_http_load_profile.py b/scripts/tests/test_http_load_profile.py
index 8fbce1e..f204cdc 100644
--- a/scripts/tests/test_http_load_profile.py
+++ b/scripts/tests/test_http_load_profile.py
@@ -2,6 +2,8 @@ from __future__ import annotations
 
 import io
 import json
+import signal
+import subprocess
 import threading
 import time
 import uuid
@@ -17,6 +19,8 @@ from urllib.parse import urlsplit
 import pytest
 
 from scripts.http_load_profile import (
+    CLI_TERMINATION_GRACE_SECONDS,
+    CLI_WATCHDOG_GRACE_SECONDS,
     HTTP_5XX_RATE_LIMIT,
     LOGICAL_VIRTUAL_USERS,
     MAX_CONCURRENCY,
@@ -42,6 +46,7 @@ from scripts.http_load_profile import (
     main,
     request_plan,
     run_profile,
+    supervised_main,
 )
 
 _DETAIL_ID = uuid.UUID("018f47d2-f9b2-7cc4-8ddf-fce39c000001")
@@ -225,6 +230,8 @@ def test_phase0_defaults_are_exact_and_smoke_is_explicitly_non_acceptance() -> N
     assert RECOVERY_FLOOR_INTERVAL_MS == 90.0
     assert P95_SCHEDULE_JITTER_LIMIT_MS == 100.0
     assert PROFILE_COMPLETION_GRACE_SECONDS == 3.0
+    assert CLI_WATCHDOG_GRACE_SECONDS == 5.0
+    assert CLI_TERMINATION_GRACE_SECONDS == 1.0
     assert phase0.label == "PHASE0_900S"
     assert smoke.label == "SMOKE_NON_ACCEPTANCE"
     with pytest.raises(LoadProfileError):
@@ -870,3 +877,181 @@ def test_main_prints_only_safe_json_and_uses_report_exit_status(
         "profile": "UNAVAILABLE",
     }
     assert private_argument not in invalid_output
+
+
+def test_supervised_cli_reprints_only_a_validated_child_receipt(
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    config = LoadProfileConfig(
+        port=8000,
+        detail_id=_DETAIL_ID,
+        profile="smoke",
+        duration_seconds=1,
+    )
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
+    report = build_report(
+        config,
+        completed_requests([observation() for _index in range(10)]),
+        measurements=run_measurements(elapsed_seconds=1.0),
+    )
+    child = MagicMock()
+    child.communicate.return_value = (f"{report.render()}\n", None)
+    child.returncode = 0
+
+    with patch("scripts.http_load_profile.subprocess.Popen", return_value=child) as popen:
+        exit_code = supervised_main(arguments)
+
+    output = capsys.readouterr().out
+    assert exit_code == 0
+    assert output.count("\n") == 1
+    assert json.loads(output) == report.data()
+    assert str(_DETAIL_ID) not in output
+    assert _REVISION_TOKEN not in output
+    command = popen.call_args.args[0]
+    assert str(_DETAIL_ID) not in " ".join(command)
+    assert popen.call_args.kwargs["stderr"] is subprocess.DEVNULL
+    assert popen.call_args.kwargs["start_new_session"] is True
+    assert set(popen.call_args.kwargs["env"]) == {
+        "PYTHONIOENCODING",
+        "PYTHONUNBUFFERED",
+        "PYTHONUTF8",
+    }
+    assert child.communicate.call_args.kwargs["timeout"] == pytest.approx(
+        1.0 + CLI_WATCHDOG_GRACE_SECONDS
+    )
+
+
+def test_supervised_cli_kills_stubborn_child_and_redacts_timeout_details(
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
+    private_output = f"private={_DETAIL_ID};revision={_REVISION_TOKEN}"
+    child = MagicMock()
+    child.pid = 4_321
+    child.returncode = None
+    child.poll.return_value = None
+    child.communicate.side_effect = (
+        subprocess.TimeoutExpired(
+            cmd=("private-command", str(_DETAIL_ID)),
+            timeout=1.0 + CLI_WATCHDOG_GRACE_SECONDS,
+            output=private_output,
+        ),
+        subprocess.TimeoutExpired(
+            cmd="private-command",
+            timeout=CLI_TERMINATION_GRACE_SECONDS,
+            output=private_output,
+        ),
+        (private_output, None),
+    )
+
+    with (
+        patch("scripts.http_load_profile.subprocess.Popen", return_value=child),
+        patch("scripts.http_load_profile.os.killpg") as kill_process_group,
+    ):
+        exit_code = supervised_main(arguments)
+
+    output = capsys.readouterr().out
+    assert exit_code == 2
+    assert json.loads(output) == {
+        "error": "WATCHDOG_TIMEOUT",
+        "passed": False,
+        "profile": "SMOKE_NON_ACCEPTANCE",
+    }
+    assert output.count("\n") == 1
+    assert str(_DETAIL_ID) not in output
+    assert _REVISION_TOKEN not in output
+    assert "private-command" not in output
+    assert kill_process_group.call_args_list == [
+        ((4_321, signal.SIGTERM),),
+        ((4_321, signal.SIGKILL),),
+    ]
+
+
+def test_supervised_cli_redacts_process_creation_failure(
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
+    private_error = OSError(f"failed argument {str(_DETAIL_ID)} {_REVISION_TOKEN}")
+
+    with patch("scripts.http_load_profile.subprocess.Popen", side_effect=private_error):
+        exit_code = supervised_main(arguments)
+
+    output = capsys.readouterr().out
+    assert exit_code == 2
+    assert json.loads(output) == {
+        "error": "SUPERVISOR_FAILED",
+        "passed": False,
+        "profile": "SMOKE_NON_ACCEPTANCE",
+    }
+    assert output.count("\n") == 1
+    assert str(_DETAIL_ID) not in output
+    assert _REVISION_TOKEN not in output
+
+
+@pytest.mark.parametrize(
+    "child_output",
+    (
+        '{"passed":true,"private":"http://127.0.0.1:8000/?token=value"}\n',
+        '{}\n{"detail_id":"018f47d2-f9b2-7cc4-8ddf-fce39c000001"}\n',
+        "민감한 오류\n",
+    ),
+)
+def test_supervised_cli_rejects_unvalidated_child_output_without_reflection(
+    child_output: str,
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
+    child = MagicMock()
+    child.communicate.return_value = (child_output, None)
+    child.returncode = 0
+
+    with patch("scripts.http_load_profile.subprocess.Popen", return_value=child):
+        exit_code = supervised_main(arguments)
+
+    output = capsys.readouterr().out
+    assert exit_code == 2
+    assert json.loads(output) == {
+        "error": "CHILD_RESULT_INVALID",
+        "passed": False,
+        "profile": "SMOKE_NON_ACCEPTANCE",
+    }
+    assert output.count("\n") == 1
+    assert "token=value" not in output
+    assert str(_DETAIL_ID) not in output
+    assert "민감한 오류" not in output


