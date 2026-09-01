## `test(browser): serve isolated acceptance states`

diff --git a/e2e/browser_scenario_launcher.py b/e2e/browser_scenario_launcher.py
new file mode 100644
index 0000000..bc1852e
--- /dev/null
+++ b/e2e/browser_scenario_launcher.py
@@ -0,0 +1,535 @@
+#!/usr/bin/env python3
+"""Start and clean seven private loopback HTTPS browser scenarios."""
+
+from __future__ import annotations
+
+import base64
+import hashlib
+import ipaddress
+import json
+import os
+from pathlib import Path
+import re
+import selectors
+import signal
+import socket
+import ssl
+import stat
+import subprocess
+import sys
+import tempfile
+import threading
+import time
+from typing import Final, Mapping, Sequence
+
+
+REPOSITORY_ROOT: Final = Path(__file__).resolve().parents[1]
+SERVER: Final = REPOSITORY_ROOT / "e2e" / "browser_scenario_server.py"
+PYTHON: Final = REPOSITORY_ROOT / ".venv" / "bin" / "python"
+OPENSSL: Final = Path("/usr/bin/openssl")
+E2E_MARKER: Final = "browser-scenarios-v1"
+SCENARIOS: Final = (
+    "ready",
+    "loading",
+    "empty",
+    "unavailable",
+    "stale",
+    "server-error",
+    "long-korean",
+)
+SAFE_RELEASE: Final = re.compile(r"\A[0-9a-f]{40}\Z")
+SAFE_TEXT: Final = re.compile(r"\A[^\x00-\x1f\x7f]{1,1024}\Z")
+STARTUP_SECONDS: Final = 20.0
+_STOP = threading.Event()
+
+
+class LauncherFailure(RuntimeError):
+    def __init__(self, check: str):
+        if not re.fullmatch(r"[a-z0-9][a-z0-9-]{0,63}", check):
+            check = "internal-error"
+        self.check = check
+        super().__init__(check)
+
+
+def _is_loopback(value: str) -> bool:
+    try:
+        return ipaddress.ip_address(value).is_loopback
+    except ValueError:
+        return False
+
+
+def _required_environment() -> dict[str, str]:
+    if (
+        os.environ.get("TRAVEL_READINESS_E2E_SCENARIO_MARKER") != E2E_MARKER
+        or os.environ.get("MOFA_TRAVEL_ALARM_SERVICE_KEY") is not None
+    ):
+        raise LauncherFailure("environment-gate")
+    names = (
+        "TRAVEL_READINESS_RELEASE_SHA",
+        "TRAVEL_READINESS_SECRET_KEY",
+        "TRAVEL_READINESS_DB_PASSWORD",
+        "TRAVEL_READINESS_DB_HOST",
+        "TRAVEL_READINESS_DB_PORT",
+        "TRAVEL_READINESS_DB_NAME",
+        "TRAVEL_READINESS_DB_USER",
+    )
+    values = {name: os.environ.get(name, "") for name in names}
+    if (
+        not SAFE_RELEASE.fullmatch(values["TRAVEL_READINESS_RELEASE_SHA"])
+        or len(values["TRAVEL_READINESS_SECRET_KEY"]) < 50
+        or not SAFE_TEXT.fullmatch(values["TRAVEL_READINESS_SECRET_KEY"])
+        or not SAFE_TEXT.fullmatch(values["TRAVEL_READINESS_DB_PASSWORD"])
+        or not _is_loopback(values["TRAVEL_READINESS_DB_HOST"])
+        or not values["TRAVEL_READINESS_DB_PORT"].isdigit()
+        or not 1 <= int(values["TRAVEL_READINESS_DB_PORT"]) <= 65_535
+        or not re.fullmatch(r"[A-Za-z0-9_.-]{1,63}", values["TRAVEL_READINESS_DB_NAME"])
+        or not re.fullmatch(r"[A-Za-z0-9_.-]{1,63}", values["TRAVEL_READINESS_DB_USER"])
+    ):
+        raise LauncherFailure("environment-gate")
+    return values
+
+
+def _write_exclusive(path: Path, payload: bytes, mode: int) -> None:
+    descriptor = os.open(
+        path,
+        os.O_WRONLY | os.O_CREAT | os.O_EXCL | getattr(os, "O_NOFOLLOW", 0),
+        mode,
+    )
+    with os.fdopen(descriptor, "wb") as handle:
+        handle.write(payload)
+        handle.flush()
+        os.fsync(handle.fileno())
+    os.chmod(path, mode)
+
+
+def _bounded_openssl(
+    arguments: Sequence[str], *, input_bytes: bytes | None = None
+) -> bytes:
+    try:
+        completed = subprocess.run(
+            [str(OPENSSL), *arguments],
+            env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+            input=input_bytes,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+            timeout=15,
+        )
+    except (OSError, subprocess.TimeoutExpired) as exc:
+        raise LauncherFailure("certificate-tool") from exc
+    if (
+        completed.returncode != 0
+        or len(completed.stdout) > 64 * 1024
+        or len(completed.stderr) > 64 * 1024
+    ):
+        raise LauncherFailure("certificate-tool")
+    return completed.stdout
+
+
+def generate_certificate(root: Path) -> tuple[Path, Path, str]:
+    configuration = root / "certificate.cnf"
+    certificate = root / "certificate.pem"
+    private_key = root / "private-key.pem"
+    _write_exclusive(
+        configuration,
+        (
+            b"[req]\n"
+            b"distinguished_name=dn\n"
+            b"x509_extensions=v3\n"
+            b"prompt=no\n"
+            b"[dn]\n"
+            b"CN=127.0.0.1\n"
+            b"[v3]\n"
+            b"subjectAltName=IP:127.0.0.1\n"
+            b"basicConstraints=critical,CA:FALSE\n"
+            b"keyUsage=critical,digitalSignature,keyEncipherment\n"
+            b"extendedKeyUsage=serverAuth\n"
+        ),
+        0o600,
+    )
+    _bounded_openssl(
+        (
+            "req",
+            "-x509",
+            "-newkey",
+            "rsa:2048",
+            "-nodes",
+            "-days",
+            "1",
+            "-config",
+            str(configuration),
+            "-keyout",
+            str(private_key),
+            "-out",
+            str(certificate),
+        )
+    )
+    try:
+        os.chmod(private_key, 0o600)
+        os.chmod(certificate, 0o600)
+        public_pem = _bounded_openssl(
+            ("x509", "-in", str(certificate), "-pubkey", "-noout")
+        )
+        public_der = _bounded_openssl(
+            ("pkey", "-pubin", "-outform", "DER"), input_bytes=public_pem
+        )
+        spki = base64.b64encode(hashlib.sha256(public_der).digest()).decode("ascii")
+    except OSError as exc:
+        raise LauncherFailure("certificate-permissions") from exc
+    if not re.fullmatch(r"[A-Za-z0-9+/]{43}=", spki):
+        raise LauncherFailure("certificate-spki")
+    return certificate, private_key, spki
+
+
+def _private_temp_directory() -> Path:
+    try:
+        root = Path(tempfile.mkdtemp(prefix="travel-readiness-browser-scenarios-"))
+        os.chmod(root, 0o700)
+        metadata = root.lstat()
+    except OSError as exc:
+        raise LauncherFailure("temporary-directory") from exc
+    if (
+        not stat.S_ISDIR(metadata.st_mode)
+        or stat.S_ISLNK(metadata.st_mode)
+        or metadata.st_uid != os.getuid()
+        or stat.S_IMODE(metadata.st_mode) != 0o700
+    ):
+        raise LauncherFailure("temporary-directory")
+    return root
+
+
+def _remove_private_tree(root: Path) -> None:
+    metadata = root.lstat()
+    if (
+        not stat.S_ISDIR(metadata.st_mode)
+        or stat.S_ISLNK(metadata.st_mode)
+        or metadata.st_uid != os.getuid()
+        or not root.name.startswith("travel-readiness-browser-scenarios-")
+    ):
+        raise LauncherFailure("cleanup-root")
+    for path in root.iterdir():
+        item = path.lstat()
+        if (
+            not stat.S_ISREG(item.st_mode)
+            or stat.S_ISLNK(item.st_mode)
+            or item.st_uid != os.getuid()
+        ):
+            raise LauncherFailure("cleanup-entry")
+        path.unlink()
+    root.rmdir()
+
+
+def _listeners() -> dict[str, socket.socket]:
+    listeners: dict[str, socket.socket] = {}
+    try:
+        for scenario in SCENARIOS:
+            listener = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
+            listener.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 0)
+            listener.bind(("127.0.0.1", 0))
+            listeners[scenario] = listener
+    except OSError as exc:
+        for listener in listeners.values():
+            listener.close()
+        raise LauncherFailure("loopback-bind") from exc
+    return listeners
+
+
+def _child_environment(
+    required: Mapping[str, str], scenario: str
+) -> dict[str, str]:
+    return {
+        "PATH": "/usr/bin:/bin",
+        "LANG": "C",
+        "LC_ALL": "C",
+        "TZ": "UTC",
+        **required,
+        "TRAVEL_READINESS_ALLOWED_HOSTS": "127.0.0.1",
+        "TRAVEL_READINESS_DEBUG": "0",
+        "TRAVEL_READINESS_HTTPS": "1",
+        "TRAVEL_READINESS_E2E_SCENARIO_MARKER": E2E_MARKER,
+        "TRAVEL_READINESS_E2E_LOOPBACK": "1",
+        "TRAVEL_READINESS_E2E_SCENARIO": scenario,
+    }
+
+
+def _spawn_children(
+    *,
+    listeners: Mapping[str, socket.socket],
+    certificate: Path,
+    private_key: Path,
+    required: Mapping[str, str],
+) -> tuple[dict[str, subprocess.Popen[bytes]], dict[str, int]]:
+    children: dict[str, subprocess.Popen[bytes]] = {}
+    readers: dict[str, int] = {}
+    try:
+        for scenario in SCENARIOS:
+            listener = listeners[scenario]
+            reader, writer = os.pipe()
+            os.set_blocking(reader, False)
+            try:
+                child = subprocess.Popen(
+                    [
+                        str(PYTHON),
+                        "-I",
+                        "-S",
+                        "-B",
+                        str(SERVER),
+                        "--scenario",
+                        scenario,
+                        "--socket-fd",
+                        str(listener.fileno()),
+                        "--ready-fd",
+                        str(writer),
+                        "--certificate",
+                        str(certificate),
+                        "--private-key",
+                        str(private_key),
+                    ],
+                    cwd=REPOSITORY_ROOT,
+                    env=_child_environment(required, scenario),
+                    stdin=subprocess.DEVNULL,
+                    stdout=subprocess.DEVNULL,
+                    stderr=subprocess.DEVNULL,
+                    pass_fds=(listener.fileno(), writer),
+                    start_new_session=True,
+                )
+            except OSError:
+                os.close(reader)
+                os.close(writer)
+                raise
+            finally:
+                listener.close()
+            os.close(writer)
+            children[scenario] = child
+            readers[scenario] = reader
+    except OSError as exc:
+        for reader in readers.values():
+            os.close(reader)
+        cleanup_processes(children)
+        raise LauncherFailure("scenario-spawn") from exc
+    return children, readers
+
+
+def _wait_ready(
+    children: Mapping[str, subprocess.Popen[bytes]], readers: Mapping[str, int]
+) -> None:
+    selector = selectors.DefaultSelector()
+    pending = set(SCENARIOS)
+    try:
+        for scenario, reader in readers.items():
+            selector.register(reader, selectors.EVENT_READ, scenario)
+        deadline = time.monotonic() + STARTUP_SECONDS
+        while pending:
+            if time.monotonic() >= deadline:
+                raise LauncherFailure("scenario-startup-timeout")
+            for scenario in pending:
+                if children[scenario].poll() is not None:
+                    raise LauncherFailure("scenario-startup-exit")
+            for key, _ in selector.select(0.1):
+                scenario = key.data
+                payload = os.read(key.fd, 2)
+                selector.unregister(key.fd)
+                os.close(key.fd)
+                if payload != b"R":
+                    raise LauncherFailure("scenario-startup-failed")
+                pending.remove(scenario)
+    finally:
+        selector.close()
+        for scenario in tuple(pending):
+            try:
+                os.close(readers[scenario])
+            except OSError:
+                pass
+
+
+def _probe_release(url: str, release: str) -> None:
+    match = re.fullmatch(r"https://127\.0\.0\.1:([1-9][0-9]{0,4})", url)
+    if match is None:
+        raise LauncherFailure("probe-url")
+    port = int(match.group(1))
+    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+    context.check_hostname = False
+    context.verify_mode = ssl.CERT_NONE
+    request = (
+        b"GET /releasez HTTP/1.1\r\nHost: 127.0.0.1\r\n"
+        b"Connection: close\r\n\r\n"
+    )
+    try:
+        with socket.create_connection(("127.0.0.1", port), timeout=2) as plain:
+            with context.wrap_socket(plain, server_hostname="127.0.0.1") as tls:
+                tls.sendall(request)
+                response = bytearray()
+                while chunk := tls.recv(4096):
+                    response.extend(chunk)
+                    if len(response) > 16_384:
+                        raise LauncherFailure("probe-response")
+    except (OSError, ssl.SSLError) as exc:
+        raise LauncherFailure("probe-connect") from exc
+    head, separator, body = bytes(response).partition(b"\r\n\r\n")
+    if (
+        separator != b"\r\n\r\n"
+        or not head.startswith(b"HTTP/1.0 200 ")
+        and not head.startswith(b"HTTP/1.1 200 ")
+        or b"content-type: application/json" not in head.lower()
+        or b"cache-control: no-store" not in head.lower()
+        or body != json.dumps(
+            {"release_sha": release}, separators=(",", ":")
+        ).encode("ascii")
+    ):
+        raise LauncherFailure("probe-response")
+
+
+def cleanup_processes(children: Mapping[str, subprocess.Popen[bytes]]) -> None:
+    """Terminate only verified child-led process groups, then reap them."""
+
+    cleanup_failed = False
+    for child in children.values():
+        if child.poll() is not None:
+            continue
+        try:
+            if os.getpgid(child.pid) != child.pid:
+                cleanup_failed = True
+                child.terminate()
+            else:
+                os.killpg(child.pid, signal.SIGTERM)
+        except ProcessLookupError:
+            pass
+        except OSError:
+            cleanup_failed = True
+    deadline = time.monotonic() + 3
+    for child in children.values():
+        remaining = max(0.01, deadline - time.monotonic())
+        try:
+            child.wait(timeout=remaining)
+        except subprocess.TimeoutExpired:
+            try:
+                if os.getpgid(child.pid) != child.pid:
+                    cleanup_failed = True
+                    child.kill()
+                else:
+                    os.killpg(child.pid, signal.SIGKILL)
+            except ProcessLookupError:
+                pass
+            except OSError:
+                cleanup_failed = True
+    for child in children.values():
+        try:
+            child.wait(timeout=2)
+        except subprocess.TimeoutExpired:
+            cleanup_failed = True
+    if cleanup_failed:
+        raise LauncherFailure("cleanup-processes")
+
+
+def _install_signal_handlers() -> dict[signal.Signals, object]:
+    _STOP.clear()
+    previous: dict[signal.Signals, object] = {}
+
+    def stop(signum: int, frame: object) -> None:
+        del frame
+        if signum in {signal.SIGINT, signal.SIGTERM, signal.SIGHUP}:
+            _STOP.set()
+
+    for requested in (signal.SIGINT, signal.SIGTERM, signal.SIGHUP):
+        previous[requested] = signal.getsignal(requested)
+        signal.signal(requested, stop)
+    return previous
+
+
+def _restore_signal_handlers(previous: Mapping[signal.Signals, object]) -> None:
+    for requested, handler in previous.items():
+        signal.signal(requested, handler)
+
+
+def _ready_receipt(origins: Mapping[str, str], spki: str) -> str:
+    payload: dict[str, str] = {
+        "browser_scenarios": "ready",
+        "certificate_spki": spki,
+    }
+    for scenario in SCENARIOS:
+        payload[f"{scenario.replace('-', '_')}_base_url"] = origins[scenario]
+    return json.dumps(payload, sort_keys=True, separators=(",", ":"))
+
+
+def run() -> int:
+    required = _required_environment()
+    for path in (PYTHON, SERVER, OPENSSL):
+        try:
+            metadata = path.resolve(strict=True).stat()
+        except OSError as exc:
+            raise LauncherFailure("runtime-file") from exc
+        if not stat.S_ISREG(metadata.st_mode) or (
+            path != SERVER and not os.access(path, os.X_OK)
+        ):
+            raise LauncherFailure("runtime-file")
+
+    temporary: Path | None = None
+    listeners: dict[str, socket.socket] = {}
+    children: dict[str, subprocess.Popen[bytes]] = {}
+    prior = _install_signal_handlers()
+    ready = False
+    try:
+        temporary = _private_temp_directory()
+        certificate, private_key, spki = generate_certificate(temporary)
+        listeners = _listeners()
+        origins = {
+            scenario: f"https://127.0.0.1:{listeners[scenario].getsockname()[1]}"
+            for scenario in SCENARIOS
+        }
+        children, readers = _spawn_children(
+            listeners=listeners,
+            certificate=certificate,
+            private_key=private_key,
+            required=required,
+        )
+        listeners = {}
+        _wait_ready(children, readers)
+        for scenario in SCENARIOS:
+            _probe_release(origins[scenario], required["TRAVEL_READINESS_RELEASE_SHA"])
+        print(_ready_receipt(origins, spki), flush=True)
+        ready = True
+        while not _STOP.wait(0.2):
+            if any(child.poll() is not None for child in children.values()):
+                raise LauncherFailure("scenario-exited")
+    finally:
+        for listener in listeners.values():
+            listener.close()
+        signal.signal(signal.SIGINT, signal.SIG_IGN)
+        signal.signal(signal.SIGTERM, signal.SIG_IGN)
+        signal.signal(signal.SIGHUP, signal.SIG_IGN)
+        cleanup_failure: LauncherFailure | None = None
+        try:
+            cleanup_processes(children)
+        except LauncherFailure as exc:
+            cleanup_failure = exc
+        if temporary is not None:
+            try:
+                _remove_private_tree(temporary)
+            except LauncherFailure as exc:
+                cleanup_failure = cleanup_failure or exc
+        _restore_signal_handlers(prior)
+        if cleanup_failure is not None:
+            raise cleanup_failure
+    if ready:
+        print('{"browser_scenarios":"stopped"}', flush=True)
+    return 0
+
+
+def main(arguments: Sequence[str] | None = None) -> int:
+    if arguments is None:
+        arguments = sys.argv[1:]
+    if arguments:
+        print("browser_scenarios=failed check=invalid-arguments", file=sys.stderr)
+        return 64
+    try:
+        return run()
+    except LauncherFailure as exc:
+        print(f"browser_scenarios=failed check={exc.check}", file=sys.stderr)
+        return 70
+    except BaseException:
+        print("browser_scenarios=failed check=internal-error", file=sys.stderr)
+        return 70
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
new file mode 100644
index 0000000..d11e1a1
--- /dev/null
+++ b/e2e/browser_scenario_server.py
@@ -0,0 +1,574 @@
+#!/usr/bin/env python3
+"""One fail-closed loopback HTTPS origin for browser acceptance.
+
+This module is E2E-only.  The acceptance launcher starts one process for each
+fixed scenario and gives it an already-bound loopback socket.  Published cards
+are either read from the real read-only PostgreSQL boundary or derived in
+memory from those typed cards; this process never fetches a source or writes a
+database row.
+"""
+
+from __future__ import annotations
+
+import argparse
+import copy
+from datetime import UTC, datetime, timedelta
+import hashlib
+import ipaddress
+import json
+import logging
+import os
+from pathlib import Path
+import re
+import socket
+import ssl
+import stat
+import sys
+import types
+from typing import Callable, Final, Mapping, Sequence
+
+
+REPOSITORY_ROOT: Final = Path(__file__).resolve().parents[1]
+E2E_MARKER: Final = "browser-scenarios-v1"
+SCENARIOS: Final = (
+    "ready",
+    "loading",
+    "empty",
+    "unavailable",
+    "stale",
+    "server-error",
+    "long-korean",
+)
+SAFE_RELEASE: Final = re.compile(r"\A[0-9a-f]{40}\Z")
+SAFE_FAILURE: Final = re.compile(r"\A[a-z0-9][a-z0-9-]{0,63}\Z")
+LOADING_SECONDS: Final = 8.0
+SITE_ASSETS: Final = {
+    "/static/public_web/site.css": (
+        REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
+        "text/css",
+        8_478,
+        "7591bba210c39bdd69c1409b3b2bccb1c829b8f059c601220965884251cce968",
+    ),
+    "/static/public_web/site.js": (
+        REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
+        "text/javascript",
+        1_544,
+        "79754d4ab020672f48ea1d7311fd1583f40e19c50a6af41b4bdf2c1b438c97d4",
+    ),
+}
+
+
+class ScenarioFailure(RuntimeError):
+    """A bounded failure code; raw exceptions never cross the process."""
+
+    def __init__(self, check: str):
+        self.check = check if SAFE_FAILURE.fullmatch(check) else "internal-error"
+        super().__init__(self.check)
+
+
+class SafeArgumentParser(argparse.ArgumentParser):
+    def error(self, message: str) -> None:
+        del message
+        raise ScenarioFailure("invalid-arguments")
+
+
+def _sha256(payload: bytes) -> str:
+    return hashlib.sha256(payload).hexdigest()
+
+
+def _bootstrap_imports() -> None:
+    """Restore only this repository and its sealed venv under ``-I -S``."""
+
+    try:
+        repository = REPOSITORY_ROOT.resolve(strict=True)
+        prefix = Path(sys.prefix).resolve(strict=True)
+        expected_prefix = (repository / ".venv").resolve(strict=True)
+        version = f"python{sys.version_info.major}.{sys.version_info.minor}"
+        site_packages = (prefix / "lib" / version / "site-packages").resolve(
+            strict=True
+        )
+        if prefix != expected_prefix or not site_packages.is_dir():
+            raise ScenarioFailure("python-environment")
+    except OSError as exc:
+        raise ScenarioFailure("python-environment") from exc
+    sys.path[:0] = [str(repository), str(site_packages)]
+
+
+def _is_loopback(value: str) -> bool:
+    try:
+        return ipaddress.ip_address(value).is_loopback
+    except ValueError:
+        return False
+
+
+def _validate_environment(scenario: str) -> str:
+    release = os.environ.get("TRAVEL_READINESS_RELEASE_SHA", "")
+    if (
+        os.environ.get("TRAVEL_READINESS_E2E_SCENARIO_MARKER") != E2E_MARKER
+        or os.environ.get("TRAVEL_READINESS_E2E_LOOPBACK") != "1"
+        or os.environ.get("TRAVEL_READINESS_E2E_SCENARIO") != scenario
+        or scenario not in SCENARIOS
+        or not SAFE_RELEASE.fullmatch(release)
+        or not _is_loopback(os.environ.get("TRAVEL_READINESS_DB_HOST", ""))
+        or os.environ.get("MOFA_TRAVEL_ALARM_SERVICE_KEY") is not None
+    ):
+        raise ScenarioFailure("environment-gate")
+    return release
+
+
+def _load_static_assets() -> dict[str, tuple[bytes, str]]:
+    loaded: dict[str, tuple[bytes, str]] = {}
+    for public_path, (source, content_type, size, digest) in SITE_ASSETS.items():
+        try:
+            metadata = source.lstat()
+            resolved = source.resolve(strict=True)
+            payload = resolved.read_bytes()
+        except OSError as exc:
+            raise ScenarioFailure("static-read") from exc
+        if (
+            resolved != source
+            or not stat.S_ISREG(metadata.st_mode)
+            or stat.S_ISLNK(metadata.st_mode)
+            or len(payload) != size
+            or _sha256(payload) != digest
+        ):
+            raise ScenarioFailure("static-integrity")
+        loaded[public_path] = (payload, content_type)
+    return loaded
+
+
+def _hangul_count(value: str) -> int:
+    return sum("\uac00" <= character <= "\ud7a3" for character in value)
+
+
+def _stale_checked_at(module: str, current_time: datetime) -> str:
+    hours = 37 if module == "entry" else 9
+    return (current_time.astimezone(UTC) - timedelta(hours=hours)).strftime(
+        "%Y-%m-%d %H:%M UTC"
+    )
+
+
+def _validate_actual_cards(cards: object) -> dict[str, dict[str, object]]:
+    """Require exact typed publication identities before making a clone."""
+
+    from entry_requirements.ingestion import (
+        ENTRY_ATTRIBUTION,
+        ENTRY_SOURCE_OWNER,
+    )
+    from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+    from travel_warnings.ingestion import (
+        WARNING_ATTRIBUTION,
+        WARNING_SOURCE_OWNER,
+    )
+
+    if not isinstance(cards, dict) or set(cards) != {"entry", "warning"}:
+        raise ScenarioFailure("publication-shape")
+    expected = {
+        "entry": (ENTRY_SOURCE_OWNER, ENTRY_SOURCE_LOCATOR, ENTRY_ATTRIBUTION),
+        "warning": (
+            WARNING_SOURCE_OWNER,
+            WARNING_SOURCE_LOCATOR,
+            WARNING_ATTRIBUTION,
+        ),
+    }
+    required = {
+        "entry": {
+            "period_text",
+            "basis_text",
+            "snapshot_date",
+            "checked_at",
+            "published_at",
+            "generation",
+        },
+        "warning": {
+            "alarm_level_code",
+            "scope_type",
+            "scope_text",
+            "written_date",
+            "checked_at",
+            "published_at",
+            "generation",
+        },
+    }
+    validated: dict[str, dict[str, object]] = {}
+    for module in ("entry", "warning"):
+        card = cards.get(module)
+        if not isinstance(card, dict):
+            raise ScenarioFailure("publication-shape")
+        owner, locator, attribution = expected[module]
+        if (
+            card.get("module") != module
+            or card.get("state") != "ready"
+            or card.get("has_publication") is not True
+            or card.get("country_name") != "일본"
+            or card.get("source_owner") != owner
+            or card.get("source_locator") != locator
+            or card.get("attribution") != attribution
+            or card.get("source_revision") != "rights-v1"
+            or any(key not in card for key in required[module])
+        ):
+            raise ScenarioFailure("publication-not-ready")
+        validated[module] = copy.deepcopy(card)
+    return validated
+
+
+def build_scenario_cards(
+    scenario: str,
+    loader: Callable[[], object],
+    *,
+    current_time: datetime | None = None,
+) -> dict[str, dict[str, object]]:
+    """Build one scenario without changing a database-backed object."""
+
+    from public_web.results import _state_card
+
+    if scenario not in SCENARIOS:
+        raise ScenarioFailure("unknown-scenario")
+    if scenario == "empty":
+        return {
+            "entry": _state_card("entry", "empty"),
+            "warning": _state_card("warning", "empty"),
+        }
+    if scenario == "server-error":
+        return {
+            "entry": _state_card("entry", "server-error"),
+            "warning": _state_card("warning", "server-error"),
+        }
+
+    cards = _validate_actual_cards(loader())
+    if scenario in {"ready", "loading"}:
+        return cards
+    if scenario == "unavailable":
+        cards["warning"] = _state_card("warning", "unavailable")
+        return cards
+    if scenario == "stale":
+        observed = current_time or datetime.now(UTC)
+        for module in ("entry", "warning"):
+            cards[module].update(
+                {
+                    "state": "stale",
+                    "status_label": "재확인 필요",
+                    "message": (
+                        "마지막 검수·게시 사실입니다. 더 최근 조회 또는 "
+                        "source 상태를 재확인해 주세요."
+                    ),
+                    "checked_at": _stale_checked_at(module, observed),
+                }
+            )
+        return cards
+    if scenario == "long-korean":
+        # Repeat an existing typed source statement.  The stress clone adds no
+        # new travel fact, while guaranteeing a genuinely long Korean value.
+        basis = cards["entry"].get("basis_text")
+        if not isinstance(basis, str) or not basis or _hangul_count(basis) < 1:
+            raise ScenarioFailure("long-source-fact")
+        repeated = basis
+        while _hangul_count(repeated) < 40 and len(repeated) <= 500:
+            repeated = f"{repeated} · {basis}"
+        if _hangul_count(repeated) < 40 or len(repeated) > 1_000:
+            raise ScenarioFailure("long-source-fact")
+        cards["entry"]["basis_text"] = repeated
+        return cards
+    raise ScenarioFailure("unknown-scenario")
+
+
+def _configure_django() -> None:
+    """Use production settings with an E2E-only, read-only runtime envelope."""
+
+    os.environ["DJANGO_SETTINGS_MODULE"] = "travel_readiness.settings"
+    from django.conf import settings
+
+    # This process deliberately has no request/access telemetry and no session
+    # middleware.  It retains the real security, CSRF and privacy boundaries.
+    settings.LOGGING_CONFIG = None
+    settings.ROOT_URLCONF = "travel_readiness_e2e_scenario_urls"
+    settings.WSGI_APPLICATION = "e2e.browser_scenario_server.application"
+    settings.MIDDLEWARE = [
+        "public_web.middleware.PublicPrivacyMiddleware",
+        "django.middleware.security.SecurityMiddleware",
+        "django.middleware.common.CommonMiddleware",
+        "django.middleware.csrf.CsrfViewMiddleware",
+        "django.middleware.clickjacking.XFrameOptionsMiddleware",
+    ]
+    templates = copy.deepcopy(settings.TEMPLATES)
+    templates[0]["OPTIONS"]["context_processors"] = [
+        "django.template.context_processors.request"
+    ]
+    settings.TEMPLATES = templates
+    settings.ALLOWED_HOSTS = ["127.0.0.1"]
+    settings.CSRF_TRUSTED_ORIGINS = []
+    database = copy.deepcopy(settings.DATABASES["default"])
+    options = copy.deepcopy(database.get("OPTIONS", {}))
+    prior = options.get("options", "")
+    options["options"] = f"{prior} -c default_transaction_read_only=on".strip()
+    database["OPTIONS"] = options
+    database["CONN_MAX_AGE"] = 0
+    settings.DATABASES = {"default": database}
+    logging.disable(logging.CRITICAL)
+
+
+def create_views(
+    *,
+    scenario: str,
+    release_sha: str,
+    assets: Mapping[str, tuple[bytes, str]],
+    card_loader: Callable[[], object],
+    delay: Callable[[], None],
+) -> dict[str, Callable]:
+    """Create real-template views with injectable read/sleep boundaries."""
+
+    from django.http import HttpRequest, HttpResponse
+    from django.shortcuts import render
+    from django.urls import reverse
+    from django.views.decorators.debug import sensitive_post_parameters
+    from django.views.decorators.http import require_GET, require_http_methods
+    from public_web.forms import TripForm
+    from public_web.views import index as normal_index
+
+    def fixed_redirect(route_name: str) -> HttpResponse:
+        return HttpResponse(
+            status=303,
+            headers={
+                "Location": reverse(route_name),
+                "Cache-Control": "no-store",
+            },
+        )
+
+    @sensitive_post_parameters("destination", "departure_date", "return_date")
+    @require_http_methods(("GET", "POST"))
+    def loading_index(request: HttpRequest) -> HttpResponse:
+        if request.method != "POST":
+            return normal_index(request)
+        form = TripForm(request.POST)
+        if form.is_valid():
+            try:
+                build_scenario_cards("loading", card_loader)
+                delay()
+            except Exception:
+                return HttpResponse(
+                    "일시적으로 요청을 처리할 수 없습니다.\n",
+                    status=503,
+                    content_type="text/plain; charset=utf-8",
+                )
+            return fixed_redirect("public_web:results")
+        form.mark_errors_for_accessibility()
+        response = render(request, "public_web/index.html", {"form": form})
+        response.headers["Cache-Control"] = "no-store"
+        return response
+
+    @require_GET
+    def results(request: HttpRequest) -> HttpResponse:
+        if request.GET:
+            return fixed_redirect("public_web:results")
+        try:
+            cards = build_scenario_cards(scenario, card_loader)
+        except Exception:
+            return HttpResponse(
+                "일시적으로 요청을 처리할 수 없습니다.\n",
+                status=503,
+                content_type="text/plain; charset=utf-8",
+            )
+        response = render(
+            request,
+            "public_web/results.html",
+            {
+                "entry_card": cards["entry"],
+                "warning_card": cards["warning"],
+            },
+        )
+        response.headers["Cache-Control"] = "no-store"
+        return response
+
+    @require_GET
+    def release(request: HttpRequest) -> HttpResponse:
+        del request
+        return HttpResponse(
+            json.dumps(
+                {"release_sha": release_sha}, separators=(",", ":")
+            ),
+            content_type="application/json; charset=utf-8",
+            headers={"Cache-Control": "no-store"},
+        )
+
+    @require_GET
+    def static_asset(request: HttpRequest, asset: str) -> HttpResponse:
+        del request
+        public_path = f"/static/public_web/{asset}"
+        if public_path not in assets:
+            return HttpResponse(status=404)
+        payload, content_type = assets[public_path]
+        return HttpResponse(
+            payload,
+            content_type=content_type,
+            headers={"Cache-Control": "no-store"},
+        )
+
+    return {
+        "index": loading_index if scenario == "loading" else normal_index,
+        "results": results,
+        "release": release,
+        "static": static_asset,
+    }
+
+
+def _install_urlconf(views: Mapping[str, Callable]) -> None:
+    from django.urls import include, path
+    from operations import error_handlers
+
+    public_name = "travel_readiness_e2e_public_urls"
+    root_name = "travel_readiness_e2e_scenario_urls"
+    public = types.ModuleType(public_name)
+    public.app_name = "public_web"
+    public.urlpatterns = [
+        path("", views["index"], name="index"),
+        path("results/", views["results"], name="results"),
+    ]
+    root = types.ModuleType(root_name)
+    root.urlpatterns = [
+        path(
+            "",
+            include(
+                (public.urlpatterns, public.app_name), namespace="public_web"
+            ),
+        ),
+        path("releasez", views["release"], name="releasez"),
+        path("static/public_web/<str:asset>", views["static"], name="static"),
+    ]
+    root.handler400 = error_handlers.bad_request
+    root.handler403 = error_handlers.permission_denied
+    root.handler404 = error_handlers.page_not_found
+    root.handler500 = error_handlers.server_error
+    sys.modules[public_name] = public
+    sys.modules[root_name] = root
+
+
+class QuietHTTPSRequestHandler:
+    """Mixin supplied to WSGIRequestHandler below after Django is imported."""
+
+    def address_string(self) -> str:
+        return "loopback"
+
+    def log_message(self, format: str, *args: object) -> None:
+        del format, args
+
+    def get_environ(self) -> dict[str, object]:
+        environ = super().get_environ()  # type: ignore[misc]
+        environ["wsgi.url_scheme"] = "https"
+        environ["HTTPS"] = "on"
+        return environ
+
+
+def _serve(
+    *,
+    scenario: str,
+    socket_fd: int,
+    ready_fd: int,
+    certificate: Path,
+    private_key: Path,
+) -> None:
+    release = _validate_environment(scenario)
+    assets = _load_static_assets()
+    try:
+        listener = socket.socket(fileno=socket_fd)
+        host, port = listener.getsockname()[:2]
+    except (OSError, TypeError, ValueError) as exc:
+        raise ScenarioFailure("listener-invalid") from exc
+    if host != "127.0.0.1" or not 1 <= int(port) <= 65_535:
+        raise ScenarioFailure("listener-not-loopback")
+
+    _configure_django()
+    import django
+
+    django.setup(set_prefix=False)
+    from django.core.wsgi import get_wsgi_application
+    from django.db import connections
+    from public_web.results import load_publication_cards
+    from wsgiref.simple_server import WSGIRequestHandler, WSGIServer
+
+    views = create_views(
+        scenario=scenario,
+        release_sha=release,
+        assets=assets,
+        card_loader=load_publication_cards,
+        delay=lambda: __import__("time").sleep(LOADING_SECONDS),
+    )
+    _install_urlconf(views)
+    application = get_wsgi_application()
+
+    class Handler(QuietHTTPSRequestHandler, WSGIRequestHandler):
+        pass
+
+    class Server(WSGIServer):
+        request_queue_size = 16
+
+        def handle_error(self, request, client_address) -> None:
+            del request, client_address
+
+    server = Server((host, port), Handler, bind_and_activate=False)
+    server.socket.close()
+    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    context.maximum_version = ssl.TLSVersion.MAXIMUM_SUPPORTED
+    try:
+        context.load_cert_chain(str(certificate), str(private_key))
+        server.socket = context.wrap_socket(listener, server_side=True)
+        server.server_address = (host, port)
+        server.server_name = host
+        server.server_port = port
+        server.setup_environ()
+        server.server_activate()
+        server.set_app(application)
+        os.write(ready_fd, b"R")
+    except (OSError, ssl.SSLError) as exc:
+        raise ScenarioFailure("https-startup") from exc
+    finally:
+        try:
+            os.close(ready_fd)
+        except OSError:
+            pass
+
+    try:
+        server.serve_forever(poll_interval=0.2)
+    finally:
+        server.server_close()
+        connections.close_all()
+
+
+def parse_arguments(arguments: Sequence[str]) -> argparse.Namespace:
+    parser = SafeArgumentParser(add_help=False)
+    parser.add_argument("--scenario", choices=SCENARIOS, required=True)
+    parser.add_argument("--socket-fd", type=int, required=True)
+    parser.add_argument("--ready-fd", type=int, required=True)
+    parser.add_argument("--certificate", type=Path, required=True)
+    parser.add_argument("--private-key", type=Path, required=True)
+    namespace = parser.parse_args(arguments)
+    if namespace.socket_fd < 3 or namespace.ready_fd < 3:
+        raise ScenarioFailure("invalid-descriptor")
+    return namespace
+
+
+def main(arguments: Sequence[str] | None = None) -> int:
+    ready_fd: int | None = None
+    try:
+        namespace = parse_arguments(sys.argv[1:] if arguments is None else arguments)
+        ready_fd = namespace.ready_fd
+        _bootstrap_imports()
+        _serve(
+            scenario=namespace.scenario,
+            socket_fd=namespace.socket_fd,
+            ready_fd=namespace.ready_fd,
+            certificate=namespace.certificate,
+            private_key=namespace.private_key,
+        )
+        return 0
+    except BaseException:
+        if ready_fd is not None:
+            try:
+                os.write(ready_fd, b"F")
+            except OSError:
+                pass
+        return 70
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
new file mode 100644
index 0000000..e94ff14
--- /dev/null
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -0,0 +1,551 @@
+import copy
+from datetime import UTC, datetime
+from http.cookies import SimpleCookie
+import html
+import http.client
+import json
+import os
+from pathlib import Path
+import re
+import selectors
+import signal
+import socket
+import ssl
+import subprocess
+import tempfile
+import unittest
+from unittest.mock import patch
+from urllib.parse import urlencode, urlsplit
+
+from django.test import RequestFactory, SimpleTestCase
+from django.urls import clear_url_caches
+
+from e2e import browser_scenario_launcher as launcher
+from e2e import browser_acceptance
+from e2e import browser_scenario_server as scenario_server
+from entry_requirements.ingestion import ENTRY_ATTRIBUTION, ENTRY_SOURCE_OWNER
+from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+from travel_warnings.ingestion import WARNING_ATTRIBUTION, WARNING_SOURCE_OWNER
+
+
+REPOSITORY_ROOT = Path(__file__).resolve().parents[2]
+LAUNCHER = REPOSITORY_ROOT / "scripts" / "run-browser-scenarios"
+RELEASE_SHA = "a" * 40
+
+
+def ready_cards() -> dict[str, dict[str, object]]:
+    return {
+        "entry": {
+            "module": "entry",
+            "heading": "입국요건 사실",
+            "state": "ready",
+            "status_label": "게시된 source 사실",
+            "message": "공식 source의 검수·게시 사실입니다.",
+            "has_publication": True,
+            "country_name": "일본",
+            "generation": 1,
+            "published_at": "2026-08-30 00:00 UTC",
+            "source_revision": "rights-v1",
+            "source_owner": ENTRY_SOURCE_OWNER,
+            "source_locator": ENTRY_SOURCE_LOCATOR,
+            "attribution": ENTRY_ATTRIBUTION,
+            "checked_at": "2026-08-30 00:00 UTC",
+            "period_text": "90일",
+            "basis_text": "일반여권 소지자: 실제 게시 source 근거",
+            "snapshot_date": "2025-07-11",
+        },
+        "warning": {
+            "module": "warning",
+            "heading": "여행경보",
+            "state": "ready",
+            "status_label": "게시된 source 사실",
+            "message": (
+                "입국요건과 독립된 공식 source의 검수·게시 사실입니다."
+            ),
+            "has_publication": True,
+            "country_name": "일본",
+            "generation": 1,
+            "published_at": "2026-08-30 00:00 UTC",
+            "source_revision": "rights-v1",
+            "source_owner": WARNING_SOURCE_OWNER,
+            "source_locator": WARNING_SOURCE_LOCATOR,
+            "attribution": WARNING_ATTRIBUTION,
+            "checked_at": "2026-08-30 00:00 UTC",
+            "alarm_level_code": "1",
+            "scope_type": "일부",
+            "scope_text": "실제 게시 source 범위",
+            "written_date": "2025-07-11",
+        },
+    }
+
+
+class BrowserScenarioCardTests(SimpleTestCase):
+    maxDiff = None
+
+    def test_fixed_state_transformations_are_in_memory_and_exact(self):
+        original = ready_cards()
+        snapshot = copy.deepcopy(original)
+        calls = 0
+
+        def load():
+            nonlocal calls
+            calls += 1
+            return original
+
+        ready = scenario_server.build_scenario_cards("ready", load)
+        self.assertEqual(
+            (ready["entry"]["state"], ready["warning"]["state"]),
+            ("ready", "ready"),
+        )
+        unavailable = scenario_server.build_scenario_cards("unavailable", load)
+        self.assertEqual(
+            (unavailable["entry"]["state"], unavailable["warning"]["state"]),
+            ("ready", "unavailable"),
+        )
+        stale = scenario_server.build_scenario_cards(
+            "stale", load, current_time=datetime(2026, 8, 31, 12, 0, tzinfo=UTC)
+        )
+        self.assertEqual(
+            (stale["entry"]["state"], stale["warning"]["state"]),
+            ("stale", "stale"),
+        )
+        self.assertEqual(stale["entry"]["checked_at"], "2026-08-29 23:00 UTC")
+        self.assertEqual(stale["warning"]["checked_at"], "2026-08-31 03:00 UTC")
+        long_cards = scenario_server.build_scenario_cards("long-korean", load)
+        self.assertGreaterEqual(
+            scenario_server._hangul_count(long_cards["entry"]["basis_text"]),
+            40,
+        )
+        for module in ("entry", "warning"):
+            self.assertEqual(long_cards[module]["country_name"], "일본")
+            self.assertEqual(
+                long_cards[module]["source_owner"], original[module]["source_owner"]
+            )
+            self.assertEqual(
+                long_cards[module]["source_locator"],
+                original[module]["source_locator"],
+            )
+        self.assertEqual(original, snapshot)
+        self.assertEqual(calls, 4)
+
+    def test_empty_and_server_error_do_not_touch_postgresql_loader(self):
+        def forbidden_loader():
+            self.fail("controlled state attempted a database read")
+
+        empty = scenario_server.build_scenario_cards("empty", forbidden_loader)
+        error = scenario_server.build_scenario_cards(
+            "server-error", forbidden_loader
+        )
+        self.assertEqual(
+            (empty["entry"]["state"], empty["warning"]["state"]),
+            ("empty", "empty"),
+        )
+        self.assertEqual(
+            (error["entry"]["state"], error["warning"]["state"]),
+            ("server-error", "server-error"),
+        )
+
+    def test_identity_or_freshness_drift_fails_closed(self):
+        for field, value in (
+            ("country_name", "다른 국가"),
+            ("source_owner", "다른 소유자"),
+            ("source_locator", "https://example.invalid/source"),
+            ("state", "stale"),
+        ):
+            with self.subTest(field=field):
+                cards = ready_cards()
+                cards["entry"][field] = value
+                with self.assertRaises(scenario_server.ScenarioFailure):
+                    scenario_server.build_scenario_cards("ready", lambda: cards)
+
+    def test_e2e_marker_and_loopback_database_are_mandatory(self):
+        valid = {
+            "TRAVEL_READINESS_E2E_SCENARIO_MARKER": launcher.E2E_MARKER,
+            "TRAVEL_READINESS_E2E_LOOPBACK": "1",
+            "TRAVEL_READINESS_E2E_SCENARIO": "ready",
+            "TRAVEL_READINESS_RELEASE_SHA": RELEASE_SHA,
+            "TRAVEL_READINESS_SECRET_KEY": (
+                "browser-scenario-test-secret-material-not-production-0001"
+            ),
+            "TRAVEL_READINESS_DB_PASSWORD": "browser-scenario-test-db-value",
+            "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+            "TRAVEL_READINESS_DB_PORT": "5432",
+            "TRAVEL_READINESS_DB_NAME": "scenario_test",
+            "TRAVEL_READINESS_DB_USER": "scenario_test",
+        }
+        with patch.dict(os.environ, valid, clear=True):
+            self.assertEqual(scenario_server._validate_environment("ready"), RELEASE_SHA)
+            self.assertEqual(
+                launcher._required_environment()["TRAVEL_READINESS_RELEASE_SHA"],
+                RELEASE_SHA,
+            )
+        for changed in (
+            {"TRAVEL_READINESS_E2E_SCENARIO_MARKER": ""},
+            {"TRAVEL_READINESS_DB_HOST": "192.0.2.1"},
+            {"MOFA_TRAVEL_ALARM_SERVICE_KEY": "synthetic-test-only"},
+        ):
+            environment = {**valid, **changed}
+            with self.subTest(changed=tuple(changed)):
+                with patch.dict(os.environ, environment, clear=True):
+                    with self.assertRaises(scenario_server.ScenarioFailure):
+                        scenario_server._validate_environment("ready")
+                    with self.assertRaises(launcher.LauncherFailure):
+                        launcher._required_environment()
+
+    def test_real_templates_release_static_validation_and_loading_contract(self):
+        assets = scenario_server._load_static_assets()
+        calls = []
+        delays = []
+
+        def loader():
+            calls.append("read")
+            return ready_cards()
+
+        views = scenario_server.create_views(
+            scenario="loading",
+            release_sha=RELEASE_SHA,
+            assets=assets,
+            card_loader=loader,
+            delay=lambda: delays.append("delay"),
+        )
+        scenario_server._install_urlconf(views)
+        clear_url_caches()
+        factory = RequestFactory()
+
+        result = views["results"](factory.get("/results/"))
+        self.assertEqual(result.status_code, 200)
+        self.assertIn(b'data-state="ready"', result.content)
+        self.assertIn(
+            html.escape(ENTRY_SOURCE_LOCATOR, quote=True).encode("ascii"),
+            result.content,
+        )
+        self.assertIn(WARNING_SOURCE_LOCATOR.encode("ascii"), result.content)
+
+        release = views["release"](factory.get("/releasez"))
+        self.assertEqual(release.status_code, 200)
+        self.assertEqual(
+            release.content,
+            json.dumps(
+                {"release_sha": RELEASE_SHA}, separators=(",", ":")
+            ).encode("ascii"),
+        )
+        self.assertEqual(release.headers["Cache-Control"], "no-store")
+
+        for public_path, (payload, content_type) in assets.items():
+            response = views["static"](
+                factory.get(public_path), asset=public_path.rsplit("/", 1)[1]
+            )
+            self.assertEqual(response.status_code, 200)
+            self.assertEqual(response.content, payload)
+            self.assertTrue(response.headers["Content-Type"].startswith(content_type))
+
+        invalid = views["index"](
+            factory.post(
+                "/",
+                {
+                    "destination": "JP",
+                    "departure_date": "2030-01-10",
+                    "return_date": "2030-01-09",
+                },
+            )
+        )
+        self.assertEqual(invalid.status_code, 200)
+        self.assertIn("입력 내용을 확인해 주세요".encode(), invalid.content)
+        self.assertEqual(delays, [])
+
+        valid = views["index"](
+            factory.post(
+                "/",
+                {
+                    "destination": "JP",
+                    "departure_date": "2030-01-10",
+                    "return_date": "2030-01-10",
+                },
+            )
+        )
+        self.assertEqual(valid.status_code, 303)
+        self.assertEqual(valid.headers["Location"], "/results/")
+        self.assertEqual(delays, ["delay"])
+        self.assertEqual(calls, ["read", "read"])
+
+    def test_each_controlled_state_renders_through_the_real_template(self):
+        assets = scenario_server._load_static_assets()
+        expected = {
+            "ready": ("ready", "ready", 2),
+            "empty": ("empty", "empty", 0),
+            "unavailable": ("ready", "unavailable", 1),
+            "stale": ("stale", "stale", 2),
+            "server-error": ("server-error", "server-error", 0),
+            "long-korean": ("ready", "ready", 2),
+        }
+        factory = RequestFactory()
+        for name, (entry_state, warning_state, link_count) in expected.items():
+            with self.subTest(name=name):
+                views = scenario_server.create_views(
+                    scenario=name,
+                    release_sha=RELEASE_SHA,
+                    assets=assets,
+                    card_loader=ready_cards,
+                    delay=lambda: self.fail("non-loading scenario delayed"),
+                )
+                scenario_server._install_urlconf(views)
+                clear_url_caches()
+                response = views["results"](factory.get("/results/"))
+                self.assertEqual(response.status_code, 200)
+                self.assertIn(
+                    f'id="entry-card" data-state="{entry_state}"'.encode(),
+                    response.content,
+                )
+                self.assertIn(
+                    f'id="warning-card" data-state="{warning_state}"'.encode(),
+                    response.content,
+                )
+                self.assertEqual(
+                    response.content.count(b'class="source-link"'), link_count
+                )
+                if name == "long-korean":
+                    long_cards = scenario_server.build_scenario_cards(
+                        name, ready_cards
+                    )
+                    self.assertIn(
+                        long_cards["entry"]["basis_text"].encode(),
+                        response.content,
+                    )
+
+
+class BrowserScenarioLauncherIntegrationTests(SimpleTestCase):
+    """Exercise real TLS, process isolation and cleanup without reading a DB."""
+
+    @staticmethod
+    def _read_line(
+        process: subprocess.Popen[bytes], timeout: float
+    ) -> bytes:
+        assert process.stdout is not None
+        selector = selectors.DefaultSelector()
+        selector.register(process.stdout, selectors.EVENT_READ)
+        try:
+            if not selector.select(timeout):
+                raise AssertionError("scenario launcher readiness timed out")
+            line = process.stdout.readline()
+        finally:
+            selector.close()
+        if not line:
+            raise AssertionError("scenario launcher exited before readiness")
+        return line
+
+    @staticmethod
+    def _request(
+        origin: str,
+        method: str,
+        path: str,
+        *,
+        body: bytes | None = None,
+        headers: dict[str, str] | None = None,
+    ) -> tuple[int, list[tuple[str, str]], bytes]:
+        parsed = urlsplit(origin)
+        context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+        context.check_hostname = False
+        context.verify_mode = ssl.CERT_NONE
+        connection = http.client.HTTPSConnection(
+            parsed.hostname,
+            parsed.port,
+            context=context,
+            timeout=5,
+        )
+        try:
+            connection.request(method, path, body=body, headers=headers or {})
+            response = connection.getresponse()
+            payload = response.read(128 * 1024)
+            if response.read(1):
+                raise AssertionError("scenario response exceeded bound")
+            return response.status, response.getheaders(), payload
+        finally:
+            connection.close()
+
+    def test_seven_tls_origins_csrf_privacy_and_cleanup(self):
+        if not LAUNCHER.exists():
+            self.skipTest("scenario launcher is unavailable")
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            temporary.chmod(0o700)
+            environment = {
+                "PATH": "/usr/bin:/bin",
+                "LANG": "C",
+                "LC_ALL": "C",
+                "TZ": "UTC",
+                "TMPDIR": str(temporary),
+                "TRAVEL_READINESS_E2E_SCENARIO_MARKER": launcher.E2E_MARKER,
+                "TRAVEL_READINESS_RELEASE_SHA": RELEASE_SHA,
+                "TRAVEL_READINESS_SECRET_KEY": (
+                    "browser-scenario-test-secret-material-not-production-0001"
+                ),
+                "TRAVEL_READINESS_DB_PASSWORD": "browser-scenario-test-db-value",
+                "TRAVEL_READINESS_DB_HOST": "127.0.0.1",
+                "TRAVEL_READINESS_DB_PORT": "1",
+                "TRAVEL_READINESS_DB_NAME": "scenario_test",
+                "TRAVEL_READINESS_DB_USER": "scenario_test",
+            }
+            process = subprocess.Popen(
+                [str(LAUNCHER)],
+                cwd=REPOSITORY_ROOT,
+                env=environment,
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                start_new_session=True,
+            )
+            receipt = None
+            ports: list[int] = []
+            try:
+                line = self._read_line(process, 25)
+                self.assertLessEqual(len(line), 4096)
+                receipt = json.loads(line)
+                self.assertEqual(receipt["browser_scenarios"], "ready")
+                self.assertEqual(
+                    set(receipt),
+                    {
+                        "browser_scenarios",
+                        "certificate_spki",
+                        *(f"{name.replace('-', '_')}_base_url" for name in launcher.SCENARIOS),
+                    },
+                )
+                origins = {
+                    name: receipt[f"{name.replace('-', '_')}_base_url"]
+                    for name in launcher.SCENARIOS
+                }
+                self.assertEqual(len(set(origins.values())), 7)
+                self.assertRegex(receipt["certificate_spki"], r"\A[A-Za-z0-9+/]{43}=\Z")
+
+                peer_certificates = set()
+
+                for origin in origins.values():
+                    parsed = urlsplit(origin)
+                    self.assertEqual(
+                        (parsed.scheme, parsed.hostname),
+                        ("https", "127.0.0.1"),
+                    )
+                    ports.append(parsed.port)
+                    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+                    context.check_hostname = False
+                    context.verify_mode = ssl.CERT_NONE
+                    with socket.create_connection(
+                        ("127.0.0.1", parsed.port), timeout=2
+                    ) as plain:
+                        with context.wrap_socket(
+                            plain, server_hostname="127.0.0.1"
+                        ) as tls:
+                            certificate = tls.getpeercert(binary_form=True)
+                    self.assertTrue(certificate, "peer certificate missing")
+                    peer_certificates.add(__import__("hashlib").sha256(certificate).hexdigest())
+                    self.assertEqual(
+                        browser_acceptance.certificate_spki_sha256(certificate),
+                        receipt["certificate_spki"],
+                    )
+                    status, headers, body = self._request(origin, "GET", "/releasez")
+                    self.assertEqual(status, 200)
+                    self.assertEqual(
+                        body,
+                        json.dumps(
+                            {"release_sha": RELEASE_SHA}, separators=(",", ":")
+                        ).encode("ascii"),
+                    )
+                    names = {name.lower(): value for name, value in headers}
+                    self.assertTrue(
+                        names["content-type"].startswith("application/json")
+                    )
+                    self.assertIn("no-store", names["cache-control"].lower())
+                    self.assertNotIn("set-cookie", names)
+                self.assertEqual(len(peer_certificates), 1)
+
+                ready = origins["ready"]
+                for path, (
+                    source,
+                    content_type,
+                    _,
+                    digest,
+                ) in scenario_server.SITE_ASSETS.items():
+                    status, headers, body = self._request(ready, "GET", path)
+                    self.assertEqual(status, 200)
+                    self.assertEqual(body, source.read_bytes())
+                    self.assertEqual(
+                        __import__("hashlib").sha256(body).hexdigest(), digest
+                    )
+                    names = {name.lower(): value for name, value in headers}
+                    self.assertTrue(names["content-type"].startswith(content_type))
+                    self.assertNotIn("set-cookie", names)
+
+                status, headers, form = self._request(ready, "GET", "/")
+                self.assertEqual(status, 200)
+                set_cookie_values = [
+                    value for name, value in headers if name.lower() == "set-cookie"
+                ]
+                self.assertEqual(len(set_cookie_values), 1)
+                cookie = SimpleCookie()
+                cookie.load(set_cookie_values[0])
+                self.assertEqual(set(cookie), {"csrftoken"})
+                morsel = cookie["csrftoken"]
+                self.assertTrue(bool(morsel.value), "csrf cookie value missing")
+                self.assertTrue(bool(morsel["secure"]), "csrf cookie is not Secure")
+                self.assertTrue(bool(morsel["httponly"]), "csrf cookie is not HttpOnly")
+                self.assertEqual(morsel["samesite"], "Strict")
+                token_match = re.search(
+                    rb'name="csrfmiddlewaretoken" value="([^"]+)"', form
+                )
+                self.assertTrue(token_match is not None, "csrf form token missing")
+                assert token_match is not None
+                post_body = urlencode(
+                    {
+                        "csrfmiddlewaretoken": token_match.group(1).decode("ascii"),
+                        "destination": "JP",
+                        "departure_date": "2030-01-10",
+                        "return_date": "2030-01-09",
+                    }
+                ).encode("ascii")
+                status, _, validation = self._request(
+                    ready,
+                    "POST",
+                    "/",
+                    body=post_body,
+                    headers={
+                        "Content-Type": "application/x-www-form-urlencoded",
+                        "Cookie": f"csrftoken={morsel.value}",
+                        "Origin": ready,
+                    },
+                )
+                self.assertEqual(status, 200)
+                self.assertIn(
+                    "입력 내용을 확인해 주세요".encode(), validation
+                )
+                status, _, pristine = self._request(ready, "GET", "/")
+                self.assertEqual(status, 200)
+                self.assertNotIn(b"2030-01-10", pristine)
+                self.assertNotIn(b"2030-01-09", pristine)
+
+                status, headers, body = self._request(
+                    ready, "GET", "/releasez?destination=JP"
+                )
+                self.assertEqual(status, 303)
+                self.assertEqual(body, b"")
+                names = {name.lower(): value for name, value in headers}
+                self.assertEqual(names["location"], "/releasez")
+            finally:
+                if process.poll() is None:
+                    os.kill(process.pid, signal.SIGTERM)
+                try:
+                    remaining_stdout, stderr = process.communicate(timeout=10)
+                except subprocess.TimeoutExpired:
+                    os.killpg(process.pid, signal.SIGKILL)
+                    process.wait(timeout=5)
+                    self.fail("scenario launcher cleanup timed out")
+
+            self.assertEqual(process.returncode, 0)
+            self.assertEqual(remaining_stdout, b'{"browser_scenarios":"stopped"}\n')
+            self.assertEqual(stderr, b"")
+            self.assertEqual(list(temporary.iterdir()), [])
+            for port in ports:
+                with self.assertRaises(OSError):
+                    socket.create_connection(("127.0.0.1", port), timeout=0.1)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/run-browser-scenarios b/scripts/run-browser-scenarios
new file mode 100755
index 0000000..7f77218
--- /dev/null
+++ b/scripts/run-browser-scenarios
@@ -0,0 +1,26 @@
+#!/bin/sh
+PATH=/usr/bin:/bin
+export PATH
+set +x
+set -eu
+IFS=' '
+umask 077
+
+unset BASH_ENV CDPATH ENV IFS
+unset LD_LIBRARY_PATH DYLD_LIBRARY_PATH DYLD_INSERT_LIBRARIES
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY PGPASSWORD PGOPTIONS
+unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
+IFS=' '
+
+case "$0" in
+  /*) ;;
+  *)
+    printf '%s\n' 'browser_scenarios=failed check=absolute-entrypoint-required' >&2
+    exit 64
+    ;;
+esac
+script_directory=${0%/*}
+repository_directory=${script_directory%/*}
+
+exec "${repository_directory}/.venv/bin/python" -I -S -B \
+  "${repository_directory}/e2e/browser_scenario_launcher.py" "$@"


