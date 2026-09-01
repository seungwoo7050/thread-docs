# 재현 가능한 브라우저 인수 검증

## `test(browser): add isolated acceptance harness`

diff --git a/e2e/__init__.py b/e2e/__init__.py
new file mode 100644
index 0000000..e9f6cb2
--- /dev/null
+++ b/e2e/__init__.py
@@ -0,0 +1 @@
+"""Local, privacy-preserving browser acceptance tooling."""
diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
new file mode 100644
index 0000000..2cfc4c5
--- /dev/null
+++ b/e2e/browser_acceptance.py
@@ -0,0 +1,2959 @@
+#!/usr/bin/env python3
+"""Privacy-safe real-Chromium acceptance for the Phase 0 release candidate.
+
+The runner accepts only seven separately-started loopback HTTPS origins. It
+has no application state switch and never installs packages. Every origin is
+bound to the requested release through its own ``/releasez`` response before
+the page is accepted. Persistent evidence is limited to PNG screenshots and a
+fixed-schema JSON receipt; command output, accessibility snapshots, raw HTML,
+cookies, form values, traces, HARs, videos, and browser profiles are transient.
+"""
+
+from __future__ import annotations
+
+import argparse
+import base64
+import binascii
+from dataclasses import dataclass
+from datetime import UTC, datetime
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import selectors
+import shutil
+import signal
+import socket
+import ssl
+import stat
+import struct
+import subprocess
+import sys
+import time
+from typing import Final, Mapping, Sequence
+from urllib.parse import urlsplit
+import zlib
+
+
+REPOSITORY_ROOT: Final = Path(__file__).resolve().parents[1]
+OUTPUT_ROOT: Final = REPOSITORY_ROOT / "output" / "playwright"
+PLAYWRIGHT_WRAPPER: Final = Path(
+    "/Users/woopinbell/.codex/skills/playwright/scripts/playwright_cli.sh"
+)
+WRAPPER_SHA256: Final = "aa3fdff5d0e4556177f4dfd5f04117e772aa54f94b6a2e34b6c0edf629c6b9b5"
+REVIEWED_CLI_VERSION: Final = "0.1.18"
+CACHED_CLI_ROOT: Final = Path("/private/tmp/npm-cache/_npx/31e32ef8478fbf80")
+CACHED_CLI_TREE_SHA256: Final = "7bdee27eb125919be7c20bb115794d933790540a364ef063b17624179db43c0e"
+CACHED_CLI_CONTENT_SHA256: Final = "6a26b64c65840331c735b35a2d562fb7387acb35cfe68484c2c40fb8cce8f368"
+CACHED_CLI_LOCK_SHA256: Final = "39b2c57962abddc563433510a9fbd02c001a5d9957b33dd6615a29839c32881d"
+CACHED_CLI_ENTRY_COUNT: Final = 242
+CACHED_CLI_FILE_BYTES: Final = 20_115_877
+PLAYWRIGHT_VERSION: Final = "1.63.0-alpha-2026-08-05"
+PACKAGE_INTEGRITIES: Final = {
+    "@playwright/cli": "sha512-ggNfYYH+GsZTGUiBEL8f6N5j0seYEUE52v+fIWqK/A36QG36cL0EJ79qWTXYO2uZMUU7vm+jk3x0fKCPL6UuIw==",
+    "playwright": "sha512-zbGZUK+JYkoDV3cUgfvh2czTBJL34Gmz5gHVI25xiIpvYSR17Q1M7TS8hnwECUe+IkKaeXbKrSyJTyogm2DVWw==",
+    "playwright-core": "sha512-YussvUybTfBtyYbGXWh43f+5kNP03wg98M6mu4DphYET7PSbNVajsdLGjWE1xrsjqOw32i2wFlRP7U5mcOpMZg==",
+    "fsevents": "sha512-xiqMQR4xAeHTuB9uWm+fFRcIOgKBMiOBP+eXiyT7jsgVCq1bkVygt00oASowB7EdtpOHaaPgKt812P9ab+DDKA==",
+}
+NODE_SOURCE: Final = Path("/opt/homebrew/Cellar/node/23.11.0/bin/node")
+NODE_VERSION: Final = "v23.11.0"
+NODE_SHA256: Final = "01ca46d5dbf4e6fd39e2cca6154b24e867fae029512b125b1b108ecfbcc1b462"
+NODE_LIBRARY_DIGESTS: Final = {
+    "/opt/homebrew/opt/libuv/lib/libuv.1.dylib": ("/opt/homebrew/Cellar/libuv/1.52.0/lib/libuv.1.0.0.dylib", "b1c36b2842a848d61078a2d125b3027e072f10670f35220907bdb18ba77eb2ff"),
+    "/opt/homebrew/opt/brotli/lib/libbrotlidec.1.dylib": ("/opt/homebrew/Cellar/brotli/1.2.0/lib/libbrotlidec.1.2.0.dylib", "d7ac1e69b6c443341fb4302de0169d187489d58d8a641c8eadd6db9ca2cd0cbf"),
+    "/opt/homebrew/opt/brotli/lib/libbrotlienc.1.dylib": ("/opt/homebrew/Cellar/brotli/1.2.0/lib/libbrotlienc.1.2.0.dylib", "32e38a8ab06c8770ea4bfaf11ea324d7bcb6fe5f07eaff9d6260f93984ee6729"),
+    "/opt/homebrew/Cellar/brotli/1.2.0/lib/libbrotlicommon.1.2.0.dylib": ("/opt/homebrew/Cellar/brotli/1.2.0/lib/libbrotlicommon.1.2.0.dylib", "3426742c78df5c3b523071df603f07f2bb8dea6bd8e65c366a2c6adf2bf0a3ad"),
+    "/opt/homebrew/opt/c-ares/lib/libcares.2.dylib": ("/opt/homebrew/Cellar/c-ares/1.34.6/lib/libcares.2.19.5.dylib", "e5595a0c640a2341e2df5a0fedaa63e7e9afe99b01c9880338064e0746414518"),
+    "/opt/homebrew/opt/libnghttp2/lib/libnghttp2.14.dylib": ("/opt/homebrew/Cellar/libnghttp2/1.68.0/lib/libnghttp2.14.dylib", "eb958b5285712d474a6542d59be092361057e8abc7a03f0dccf0282417faacbf"),
+    "/opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib": ("/opt/homebrew/Cellar/openssl@3/3.6.3/lib/libcrypto.3.dylib", "a12805a18cd5e4f733fa8727b91afa08b587f9da5a760517cd79cb508a3a3f71"),
+    "/opt/homebrew/opt/openssl@3/lib/libssl.3.dylib": ("/opt/homebrew/Cellar/openssl@3/3.6.3/lib/libssl.3.dylib", "ffd8ac6981000def0928367924b6cb1e7a98712efbc06e2a2f3f750138bd89ca"),
+    "/opt/homebrew/opt/icu4c@77/lib/libicui18n.77.dylib": ("/opt/homebrew/Cellar/icu4c@77/77.1/lib/libicui18n.77.1.dylib", "e7424bd49b3bfd8bb9be1b653530a2f4bc074b174254fda73d63bca6d215b18f"),
+    "/opt/homebrew/opt/icu4c@77/lib/libicuuc.77.dylib": ("/opt/homebrew/Cellar/icu4c@77/77.1/lib/libicuuc.77.1.dylib", "848c6be8f00da778505e2c9f7b73c8afec92b1d2b42a984b950cfd62e232674e"),
+    "/opt/homebrew/Cellar/icu4c@77/77.1/lib/libicudata.77.1.dylib": ("/opt/homebrew/Cellar/icu4c@77/77.1/lib/libicudata.77.1.dylib", "7aae18a54bd008f9543255ae964c4eecdc8de21e6894ef223272c4db1a21bde6"),
+}
+NODE_LIBRARY_SET_SHA256: Final = "dae98347d886178af38662ca91dbd025b631a709d5c1d30037da17e49f107703"
+BASH_SOURCE: Final = Path("/bin/bash")
+BASH_SHA256: Final = "16303dedab719ddb63efd9dd68092a0da37b397bfd4fcb856d5f1c86c5ee414b"
+SH_SOURCE: Final = Path("/bin/sh")
+SH_SHA256: Final = "c0eaf44f9242d5bbc2e14f4e8b7dccc1eff7f2976d64dc18914d5ef9f373e100"
+ENV_SOURCE: Final = Path("/usr/bin/env")
+ENV_SHA256: Final = "75690864f0e7397db05bcc0f4439915559ce24c2d834d530e4e619c14b938556"
+CHROME_BUNDLE: Final = Path(
+    "/Users/woopinbell/Library/Caches/ms-playwright/chromium-1237"
+)
+CHROME_EXECUTABLE: Final = (
+    CHROME_BUNDLE
+    / "chrome-mac-arm64"
+    / "Google Chrome for Testing.app"
+    / "Contents"
+    / "MacOS"
+    / "Google Chrome for Testing"
+)
+CHROME_REVISION: Final = "1237"
+CHROME_PRODUCT_NAME: Final = "Google Chrome for Testing"
+CHROME_VERSION: Final = "152.0.7977.8"
+CHROME_VERSION_OUTPUT: Final = f"{CHROME_PRODUCT_NAME} {CHROME_VERSION}"
+CHROME_BINARY_SHA256: Final = "72d65943199c16a93085b9d4b11fabb23362c44a2d09fc1d2912565911d3c191"
+CHROME_TREE_SHA256: Final = "35da223dd6f8d25fcd6a7cbc057a84aa7e7ecbdbf9b3911db3d8ce5bdf30ae48"
+CHROME_ENTRY_COUNT: Final = 652
+CHROME_FILE_BYTES: Final = 371_215_493
+
+VIEWPORTS: Final = ((360, 800), (390, 844), (768, 1024), (1440, 900))
+ORIGIN_NAMES: Final = (
+    "ready",
+    "loading",
+    "empty",
+    "unavailable",
+    "stale",
+    "server-error",
+    "long-korean",
+)
+STATE_NAMES: Final = (
+    "ready",
+    "empty",
+    "unavailable",
+    "stale",
+    "server-error",
+    "long-korean",
+)
+STATE_PAIRS: Final = {
+    "ready": ("ready", "ready"),
+    "empty": ("empty", "empty"),
+    "unavailable": ("ready", "unavailable"),
+    "stale": ("stale", "stale"),
+    "server-error": ("server-error", "server-error"),
+    "long-korean": ("ready", "ready"),
+}
+SCREENSHOT_NAMES: Final = (
+    "form-pristine",
+    "form-loading",
+    "form-validation",
+    "form-correction-ready",
+    "ready-forced-colors",
+    "empty",
+    "unavailable",
+    "stale",
+    "server-error",
+    "long-korean",
+    "long-korean-200-percent",
+    "form-200-percent",
+    "ready-200-percent",
+)
+MAX_CLI_OUTPUT_BYTES: Final = 1_048_576
+MAX_SNAPSHOT_BYTES: Final = 262_144
+MAX_SCREENSHOT_BYTES: Final = 25_000_000
+MAX_SCREENSHOT_HEIGHT: Final = 50_000
+CLI_TIMEOUT_SECONDS: Final = 75
+PNG_SIGNATURE: Final = b"\x89PNG\r\n\x1a\n"
+SAFE_SLUG = re.compile(r"\A[a-z0-9](?:[a-z0-9-]{0,62}[a-z0-9])?\Z")
+SAFE_RELEASE = re.compile(r"\A(?:[0-9a-f]{40}|[0-9a-f]{64})\Z")
+SAFE_MARKER = re.compile(r"\APW_ACCEPTANCE_OK:[a-z0-9-]+\Z")
+SAFE_CHECK = re.compile(r"\A[a-z0-9][a-z0-9-]{0,95}\Z")
+SYNTHETIC_DEPARTURE: Final = "2030-01-10"
+SYNTHETIC_INVALID_RETURN: Final = "2030-01-09"
+SYNTHETIC_VALID_RETURN: Final = "2030-01-10"
+ENTRY_PUBLIC_SOURCE_LOCATOR: Final = (
+    "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
+    "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N"
+)
+WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
+    "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+    "getTravelAlarmList2"
+)
+ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
+WARNING_FRESHNESS_MINUTES: Final = 8 * 60
+SITE_CSS_SHA256: Final = "7591bba210c39bdd69c1409b3b2bccb1c829b8f059c601220965884251cce968"
+SITE_CSS_BYTES: Final = 8_478
+SITE_JS_SHA256: Final = "79754d4ab020672f48ea1d7311fd1583f40e19c50a6af41b4bdf2c1b438c97d4"
+SITE_JS_BYTES: Final = 1_544
+_SIGNAL_INTERRUPTED = False
+_SIGNAL_CLEANUP_DEPTH = 0
+
+
+class AcceptanceFailure(RuntimeError):
+    """A failure whose fixed code is safe to publish."""
+
+    def __init__(self, check: str):
+        if not SAFE_CHECK.fullmatch(check):
+            check = "internal-error"
+        self.check = check
+        super().__init__(check)
+
+
+class SafeArgumentParser(argparse.ArgumentParser):
+    def error(self, message: str) -> None:
+        del message
+        raise AcceptanceFailure("invalid-arguments")
+
+
+@dataclass(frozen=True, slots=True)
+class Toolchain:
+    wrapper: Path
+    environment: Mapping[str, str]
+    daemon_root: Path
+    global_root: Path
+    sealed_root: Path
+    chrome_executable: Path
+    chrome_tree_sha256: str
+    sealed_tree_sha256: str
+    sealed_entry_count: int
+    sealed_file_bytes: int
+
+
+@dataclass(frozen=True, slots=True)
+class ProcessIdentity:
+    pid: int
+    uid: int
+    process_group: int
+    started: str
+    command: str
+
+
+def utc_now() -> str:
+    return datetime.now(UTC).isoformat(timespec="seconds").replace("+00:00", "Z")
+
+
+def _sha256(path: Path) -> str:
+    digest = hashlib.sha256()
+    try:
+        with path.open("rb") as handle:
+            while chunk := handle.read(1024 * 1024):
+                digest.update(chunk)
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-read-failed") from exc
+    return digest.hexdigest()
+
+
+def _tree_digest(root: Path, *, include_modes: bool) -> tuple[str, int, int]:
+    digest = hashlib.sha256()
+    count = 0
+    file_bytes = 0
+    try:
+        entries = sorted(
+            root.rglob("*"),
+            key=lambda item: item.relative_to(root).as_posix().encode("utf-8"),
+        )
+        for path in entries:
+            metadata = path.lstat()
+            relative = path.relative_to(root).as_posix()
+            mode = stat.S_IMODE(metadata.st_mode) if include_modes else 0
+            if stat.S_ISLNK(metadata.st_mode):
+                record = ["l", relative, mode, 0, os.readlink(path)]
+            elif stat.S_ISDIR(metadata.st_mode):
+                record = ["d", relative, mode, 0, ""]
+            elif stat.S_ISREG(metadata.st_mode):
+                file_digest = _sha256(path)
+                file_bytes += metadata.st_size
+                record = ["f", relative, mode, metadata.st_size, file_digest]
+            else:
+                raise AcceptanceFailure("toolchain-special-file")
+            digest.update(
+                json.dumps(record, ensure_ascii=False, separators=(",", ":")).encode(
+                    "utf-8"
+                )
+                + b"\n"
+            )
+            count += 1
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-tree-read-failed") from exc
+    return digest.hexdigest(), count, file_bytes
+
+
+def _assert_regular(
+    path: Path,
+    *,
+    digest: str,
+    executable: bool = False,
+    require_owner: bool = True,
+) -> Path:
+    try:
+        resolved = path.resolve(strict=True)
+        metadata = resolved.stat()
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-file-unavailable") from exc
+    if not stat.S_ISREG(metadata.st_mode) or _sha256(resolved) != digest:
+        raise AcceptanceFailure("toolchain-file-mismatch")
+    if require_owner and metadata.st_uid not in {0, os.getuid()}:
+        raise AcceptanceFailure("toolchain-file-owner")
+    if executable and not os.access(resolved, os.X_OK):
+        raise AcceptanceFailure("toolchain-file-not-executable")
+    return resolved
+
+
+def _validate_node_libraries() -> None:
+    encoded = json.dumps(
+        NODE_LIBRARY_DIGESTS, sort_keys=True, separators=(",", ":")
+    ).encode("utf-8")
+    if hashlib.sha256(encoded).hexdigest() != NODE_LIBRARY_SET_SHA256:
+        raise AcceptanceFailure("node-library-manifest")
+    for loader_name, (canonical_name, digest) in NODE_LIBRARY_DIGESTS.items():
+        loader = Path(loader_name)
+        canonical = Path(canonical_name)
+        try:
+            if loader.resolve(strict=True) != canonical or canonical.resolve(strict=True) != canonical:
+                raise AcceptanceFailure("node-library-path")
+        except OSError as exc:
+            raise AcceptanceFailure("node-library-unavailable") from exc
+        metadata = canonical.stat()
+        if metadata.st_uid != os.getuid() or metadata.st_mode & (
+            stat.S_IWUSR | stat.S_IWGRP | stat.S_IWOTH
+        ):
+            raise AcceptanceFailure("node-library-permissions")
+        _assert_regular(canonical, digest=digest)
+
+
+def validate_wrapper(path: Path = PLAYWRIGHT_WRAPPER) -> Path:
+    try:
+        resolved = path.resolve(strict=True)
+        expected = PLAYWRIGHT_WRAPPER.resolve(strict=True)
+        metadata = resolved.stat()
+    except OSError as exc:
+        raise AcceptanceFailure("playwright-wrapper-unavailable") from exc
+    if (
+        resolved != expected
+        or not stat.S_ISREG(metadata.st_mode)
+        or _sha256(resolved) != WRAPPER_SHA256
+    ):
+        raise AcceptanceFailure("playwright-wrapper-invalid")
+    if metadata.st_uid != os.getuid() or metadata.st_mode & (stat.S_IWGRP | stat.S_IWOTH):
+        raise AcceptanceFailure("playwright-wrapper-permissions")
+    if not os.access(resolved, os.X_OK):
+        raise AcceptanceFailure("playwright-wrapper-not-executable")
+    return resolved
+
+
+def _assert_private_directory(path: Path) -> None:
+    try:
+        metadata = path.lstat()
+    except OSError as exc:
+        raise AcceptanceFailure("unsafe-private-directory") from exc
+    if (
+        not stat.S_ISDIR(metadata.st_mode)
+        or stat.S_ISLNK(metadata.st_mode)
+        or metadata.st_uid != os.getuid()
+        or stat.S_IMODE(metadata.st_mode) != 0o700
+    ):
+        raise AcceptanceFailure("unsafe-private-directory")
+
+
+def _ensure_private_directory(path: Path, *, exist_ok: bool = False) -> None:
+    try:
+        os.mkdir(path, 0o700)
+    except FileExistsError:
+        if not exist_ok:
+            raise
+    except OSError as exc:
+        raise AcceptanceFailure("private-directory-create-failed") from exc
+    _assert_private_directory(path)
+
+
+def create_run_directory(label: str) -> tuple[Path, Path]:
+    if not SAFE_SLUG.fullmatch(label):
+        raise AcceptanceFailure("invalid-output-label")
+    output_parent = REPOSITORY_ROOT / "output"
+    for path in (output_parent, OUTPUT_ROOT):
+        try:
+            _ensure_private_directory(path, exist_ok=True)
+        except FileExistsError as exc:
+            raise AcceptanceFailure("unsafe-output-root") from exc
+    try:
+        if OUTPUT_ROOT.resolve(strict=True) != (
+            REPOSITORY_ROOT.resolve(strict=True) / "output" / "playwright"
+        ):
+            raise AcceptanceFailure("unsafe-output-root")
+        run_root = OUTPUT_ROOT / label
+        _ensure_private_directory(run_root)
+        work_root = run_root / ".work"
+        _ensure_private_directory(work_root)
+    except FileExistsError as exc:
+        raise AcceptanceFailure("output-label-exists") from exc
+    return run_root, work_root
+
+
+def _write_private(path: Path, payload: bytes, *, mode: int) -> None:
+    try:
+        descriptor = os.open(
+            path,
+            os.O_WRONLY | os.O_CREAT | os.O_EXCL | getattr(os, "O_NOFOLLOW", 0),
+            mode,
+        )
+        with os.fdopen(descriptor, "wb") as handle:
+            handle.write(payload)
+            handle.flush()
+            os.fsync(handle.fileno())
+        os.chmod(path, mode)
+    except OSError as exc:
+        try:
+            path.unlink(missing_ok=True)
+        except OSError:
+            pass
+        raise AcceptanceFailure("private-file-write-failed") from exc
+
+
+def atomic_json(path: Path, value: object) -> None:
+    try:
+        path.lstat()
+    except FileNotFoundError:
+        pass
+    except OSError as exc:
+        raise AcceptanceFailure("evidence-target-invalid") from exc
+    else:
+        raise AcceptanceFailure("evidence-target-exists")
+    temporary = path.with_name(f".{path.name}.tmp")
+    payload = (
+        json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(",", ":"))
+        + "\n"
+    ).encode("utf-8")
+    _write_private(temporary, payload, mode=0o600)
+    try:
+        os.replace(temporary, path)
+    except OSError as exc:
+        temporary.unlink(missing_ok=True)
+        raise AcceptanceFailure("evidence-write-failed") from exc
+
+
+def _bounded_process(
+    argv: Sequence[str],
+    *,
+    cwd: Path,
+    environment: Mapping[str, str],
+    timeout: float,
+    check: str,
+) -> tuple[int, bytes, bytes]:
+    """Drain both pipes concurrently with caps and a single absolute deadline."""
+
+    if not SAFE_CHECK.fullmatch(check):
+        raise AcceptanceFailure("invalid-check-name")
+    try:
+        process = subprocess.Popen(
+            list(argv),
+            cwd=cwd,
+            env=dict(environment),
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            start_new_session=True,
+        )
+    except OSError as exc:
+        raise AcceptanceFailure(f"cli-{check}-unavailable") from exc
+    if process.stdout is None or process.stderr is None:
+        try:
+            os.killpg(process.pid, signal.SIGKILL)
+        except OSError:
+            pass
+        try:
+            process.wait(timeout=2)
+        except subprocess.TimeoutExpired:
+            pass
+        raise AcceptanceFailure(f"cli-{check}-unavailable")
+
+    stdout_fd = process.stdout.fileno()
+    stderr_fd = process.stderr.fileno()
+    selector = selectors.DefaultSelector()
+    buffers = {stdout_fd: bytearray(), stderr_fd: bytearray()}
+    streams = {stdout_fd: process.stdout, stderr_fd: process.stderr}
+    for descriptor, stream in streams.items():
+        os.set_blocking(descriptor, False)
+        selector.register(stream, selectors.EVENT_READ, descriptor)
+    deadline = time.monotonic() + timeout
+    failure: str | None = None
+    interrupted: BaseException | None = None
+    try:
+        while selector.get_map() or process.poll() is None:
+            remaining = deadline - time.monotonic()
+            if remaining <= 0:
+                failure = f"cli-{check}-timeout"
+                break
+            events = selector.select(min(0.2, remaining))
+            for key, _ in events:
+                descriptor = key.data
+                try:
+                    chunk = os.read(descriptor, 65_536)
+                except BlockingIOError:
+                    continue
+                if not chunk:
+                    selector.unregister(key.fileobj)
+                    key.fileobj.close()
+                    continue
+                buffers[descriptor].extend(chunk)
+                if len(buffers[descriptor]) > MAX_CLI_OUTPUT_BYTES:
+                    failure = f"cli-{check}-output-limit"
+                    break
+            if failure is not None:
+                break
+        if failure is None:
+            try:
+                process.wait(timeout=max(0.01, deadline - time.monotonic()))
+            except subprocess.TimeoutExpired:
+                failure = f"cli-{check}-timeout"
+    except BaseException as exc:
+        interrupted = exc
+    finally:
+        selector.close()
+        for stream in streams.values():
+            try:
+                stream.close()
+            except OSError:
+                pass
+
+    if failure is not None or interrupted is not None:
+        try:
+            os.killpg(process.pid, signal.SIGTERM)
+        except ProcessLookupError:
+            pass
+        except OSError:
+            try:
+                process.terminate()
+            except OSError:
+                pass
+        try:
+            process.wait(timeout=2)
+        except subprocess.TimeoutExpired:
+            try:
+                os.killpg(process.pid, signal.SIGKILL)
+            except ProcessLookupError:
+                pass
+            except OSError:
+                try:
+                    process.kill()
+                except OSError:
+                    pass
+            try:
+                process.wait(timeout=2)
+            except subprocess.TimeoutExpired as exc:
+                raise AcceptanceFailure(f"cli-{check}-reap-failed") from exc
+        if interrupted is not None:
+            raise interrupted
+        assert failure is not None
+        raise AcceptanceFailure(failure)
+    return process.returncode, bytes(buffers[stdout_fd]), bytes(buffers[stderr_fd])
+
+
+def _copy_toolchain_tree(source: Path, destination: Path) -> None:
+    try:
+        shutil.copytree(source, destination, symlinks=True, copy_function=shutil.copyfile)
+        destination_root = destination.resolve(strict=True)
+        for path in sorted(destination.rglob("*"), reverse=True):
+            metadata = path.lstat()
+            if stat.S_ISLNK(metadata.st_mode):
+                resolved = path.resolve(strict=True)
+                if not resolved.is_relative_to(destination_root):
+                    raise AcceptanceFailure("toolchain-symlink-escape")
+            elif stat.S_ISDIR(metadata.st_mode):
+                os.chmod(path, 0o700)
+            elif stat.S_ISREG(metadata.st_mode):
+                source_mode = (source / path.relative_to(destination)).stat().st_mode
+                os.chmod(path, 0o700 if source_mode & 0o111 else 0o600)
+            else:
+                raise AcceptanceFailure("toolchain-special-file")
+        os.chmod(destination, 0o700)
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-copy-failed") from exc
+
+
+def _seal_toolchain_tree(root: Path) -> tuple[str, int, int]:
+    """Remove every write bit from the private execution snapshot."""
+
+    _assert_private_directory(root)
+    try:
+        for path in sorted(root.rglob("*"), key=lambda item: len(item.parts), reverse=True):
+            metadata = path.lstat()
+            if metadata.st_uid != os.getuid():
+                raise AcceptanceFailure("private-toolchain-owner")
+            if stat.S_ISLNK(metadata.st_mode):
+                if not path.resolve(strict=True).is_relative_to(root.resolve(strict=True)):
+                    raise AcceptanceFailure("private-toolchain-symlink")
+            elif stat.S_ISDIR(metadata.st_mode):
+                os.chmod(path, 0o500)
+            elif stat.S_ISREG(metadata.st_mode):
+                os.chmod(path, 0o500 if metadata.st_mode & 0o111 else 0o400)
+            else:
+                raise AcceptanceFailure("private-toolchain-special-file")
+        os.chmod(root, 0o500)
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("private-toolchain-seal-failed") from exc
+    return _verify_sealed_toolchain_tree(root)
+
+
+def _verify_sealed_toolchain_tree(root: Path) -> tuple[str, int, int]:
+    try:
+        root_metadata = root.lstat()
+        root_resolved = root.resolve(strict=True)
+    except OSError as exc:
+        raise AcceptanceFailure("private-toolchain-unavailable") from exc
+    if (
+        not stat.S_ISDIR(root_metadata.st_mode)
+        or stat.S_ISLNK(root_metadata.st_mode)
+        or root_metadata.st_uid != os.getuid()
+        or stat.S_IMODE(root_metadata.st_mode) != 0o500
+    ):
+        raise AcceptanceFailure("private-toolchain-permissions")
+    try:
+        for path in root.rglob("*"):
+            metadata = path.lstat()
+            if metadata.st_uid != os.getuid():
+                raise AcceptanceFailure("private-toolchain-permissions")
+            if stat.S_ISLNK(metadata.st_mode):
+                if not path.resolve(strict=True).is_relative_to(root_resolved):
+                    raise AcceptanceFailure("private-toolchain-symlink")
+            elif stat.S_ISDIR(metadata.st_mode) or stat.S_ISREG(metadata.st_mode):
+                if metadata.st_mode & (
+                    stat.S_IWUSR | stat.S_IWGRP | stat.S_IWOTH
+                ):
+                    raise AcceptanceFailure("private-toolchain-permissions")
+            else:
+                raise AcceptanceFailure("private-toolchain-special-file")
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("private-toolchain-read-failed") from exc
+    return _tree_digest(root, include_modes=True)
+
+
+def _validate_package_lock(root: Path) -> None:
+    lock = root / "package-lock.json"
+    if _sha256(lock) != CACHED_CLI_LOCK_SHA256:
+        raise AcceptanceFailure("cli-lock-mismatch")
+    try:
+        packages = json.loads(lock.read_text(encoding="utf-8"))["packages"]
+    except (OSError, UnicodeError, json.JSONDecodeError, KeyError, TypeError) as exc:
+        raise AcceptanceFailure("cli-lock-invalid") from exc
+    found: dict[str, dict[str, object]] = {}
+    for key, value in packages.items():
+        for name in PACKAGE_INTEGRITIES:
+            if key.endswith(f"/node_modules/{name}"):
+                found[name] = value
+    if set(found) != set(PACKAGE_INTEGRITIES):
+        raise AcceptanceFailure("cli-lock-package-set")
+    versions = {
+        "@playwright/cli": REVIEWED_CLI_VERSION,
+        "playwright": PLAYWRIGHT_VERSION,
+        "playwright-core": PLAYWRIGHT_VERSION,
+        "fsevents": "2.3.2",
+    }
+    for name, expected_integrity in PACKAGE_INTEGRITIES.items():
+        item = found[name]
+        if (
+            item.get("version") != versions[name]
+            or item.get("integrity") != expected_integrity
+            or not str(item.get("resolved", "")).startswith("https://registry.npmjs.org/")
+        ):
+            raise AcceptanceFailure("cli-lock-package-mismatch")
+
+
+def restricted_cli_environment(
+    *, bin_directory: Path, work_root: Path, daemon_root: Path, global_root: Path,
+    chrome_executable: Path,
+) -> dict[str, str]:
+    return {
+        "PATH": str(bin_directory),
+        "HOME": str(work_root / "home"),
+        "LANG": "C.UTF-8",
+        "LC_ALL": "C.UTF-8",
+        "TZ": "UTC",
+        "CI": "1",
+        "TMPDIR": str(work_root / "tmp"),
+        "NO_PROXY": "localhost,127.0.0.1,::1",
+        "no_proxy": "localhost,127.0.0.1,::1",
+        "NPM_CONFIG_OFFLINE": "true",
+        "NPM_CONFIG_AUDIT": "false",
+        "NPM_CONFIG_FUND": "false",
+        "NPM_CONFIG_UPDATE_NOTIFIER": "false",
+        "NO_UPDATE_NOTIFIER": "1",
+        "PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD": "1",
+        "PWTEST_CLI_HEADLESS": "1",
+        "PWTEST_CLI_EXECUTABLE_PATH": str(chrome_executable),
+        "PWTEST_DAEMON_SESSION_DIR": str(daemon_root),
+        "PWTEST_CLI_GLOBAL_CONFIG": str(global_root),
+    }
+
+
+def prepare_toolchain(work_root: Path, wrapper: Path = PLAYWRIGHT_WRAPPER) -> Toolchain:
+    """Verify reviewed host artifacts, then copy the CLI into private storage."""
+
+    _assert_private_directory(work_root)
+    validated_wrapper = validate_wrapper(wrapper)
+    for path in (
+        NODE_SOURCE, BASH_SOURCE, SH_SOURCE, ENV_SOURCE,
+        CHROME_BUNDLE, CHROME_EXECUTABLE,
+    ):
+        try:
+            if path.resolve(strict=True) != path:
+                raise AcceptanceFailure("toolchain-canonical-path")
+        except OSError as exc:
+            raise AcceptanceFailure("toolchain-file-unavailable") from exc
+    _assert_regular(NODE_SOURCE, digest=NODE_SHA256, executable=True)
+    _validate_node_libraries()
+    _assert_regular(BASH_SOURCE, digest=BASH_SHA256, executable=True)
+    _assert_regular(SH_SOURCE, digest=SH_SHA256, executable=True)
+    _assert_regular(ENV_SOURCE, digest=ENV_SHA256, executable=True)
+    _assert_regular(CHROME_EXECUTABLE, digest=CHROME_BINARY_SHA256, executable=True)
+    try:
+        if CACHED_CLI_ROOT.resolve(strict=True) != CACHED_CLI_ROOT:
+            raise AcceptanceFailure("cli-cache-canonical-path")
+    except OSError as exc:
+        raise AcceptanceFailure("cli-cache-unavailable") from exc
+    cli_tree = _tree_digest(CACHED_CLI_ROOT, include_modes=True)
+    if cli_tree != (
+        CACHED_CLI_TREE_SHA256,
+        CACHED_CLI_ENTRY_COUNT,
+        CACHED_CLI_FILE_BYTES,
+    ):
+        raise AcceptanceFailure("cli-cache-tree-mismatch")
+    for path in (CACHED_CLI_ROOT, *CACHED_CLI_ROOT.rglob("*")):
+        metadata = path.lstat()
+        if metadata.st_uid != os.getuid() or metadata.st_mode & (stat.S_IWGRP | stat.S_IWOTH):
+            raise AcceptanceFailure("cli-cache-permissions")
+    _validate_package_lock(CACHED_CLI_ROOT)
+    chrome_tree = _tree_digest(CHROME_BUNDLE, include_modes=True)
+    chrome_content = _tree_digest(CHROME_BUNDLE, include_modes=False)
+    if chrome_tree != (CHROME_TREE_SHA256, CHROME_ENTRY_COUNT, CHROME_FILE_BYTES):
+        raise AcceptanceFailure("chrome-bundle-mismatch")
+    for path in (CHROME_BUNDLE, *CHROME_BUNDLE.rglob("*")):
+        metadata = path.lstat()
+        if metadata.st_uid != os.getuid() or metadata.st_mode & (stat.S_IWGRP | stat.S_IWOTH):
+            raise AcceptanceFailure("chrome-bundle-permissions")
+    try:
+        browser_manifest = json.loads(
+            (CACHED_CLI_ROOT / "node_modules" / "playwright-core" / "browsers.json").read_text(
+                encoding="utf-8"
+            )
+        )
+        chromium = next(
+            item for item in browser_manifest["browsers"] if item.get("name") == "chromium"
+        )
+    except (OSError, UnicodeError, json.JSONDecodeError, KeyError, StopIteration, TypeError) as exc:
+        raise AcceptanceFailure("chrome-manifest-invalid") from exc
+    if (
+        chromium.get("revision") != CHROME_REVISION
+        or chromium.get("browserVersion") != CHROME_VERSION
+        or chromium.get("title") != "Chrome for Testing"
+    ):
+        raise AcceptanceFailure("chrome-manifest-mismatch")
+
+    base_environment = {"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"}
+    code, output, _ = _bounded_process(
+        [str(NODE_SOURCE), "--version"], cwd=work_root,
+        environment=base_environment, timeout=10, check="node-version",
+    )
+    if code != 0 or output.decode("ascii", "strict").strip() != NODE_VERSION:
+        raise AcceptanceFailure("node-version-mismatch")
+    code, output, _ = _bounded_process(
+        [str(CHROME_EXECUTABLE), "--version"], cwd=work_root,
+        environment=base_environment, timeout=15, check="chrome-version",
+    )
+    if code != 0 or output.decode("ascii", "strict").strip() != CHROME_VERSION_OUTPUT:
+        raise AcceptanceFailure("chrome-version-mismatch")
+
+    sealed_root = work_root / "toolchain"
+    bin_directory = sealed_root / "bin"
+    cli_destination = sealed_root / "cli"
+    chrome_destination = sealed_root / "chrome"
+    private_chrome_executable = chrome_destination / CHROME_EXECUTABLE.relative_to(
+        CHROME_BUNDLE
+    )
+    daemon_root = work_root / "daemon"
+    global_root = work_root / "global"
+    for directory in (
+        sealed_root,
+        bin_directory,
+        daemon_root,
+        global_root,
+        global_root / ".playwright",
+        work_root / "home",
+        work_root / "tmp",
+    ):
+        _ensure_private_directory(directory)
+    _copy_toolchain_tree(CACHED_CLI_ROOT, cli_destination)
+    _copy_toolchain_tree(CHROME_BUNDLE, chrome_destination)
+    copied_content = _tree_digest(cli_destination, include_modes=False)
+    if copied_content != (
+        CACHED_CLI_CONTENT_SHA256,
+        CACHED_CLI_ENTRY_COUNT,
+        CACHED_CLI_FILE_BYTES,
+    ):
+        raise AcceptanceFailure("private-cli-copy-mismatch")
+    _validate_package_lock(cli_destination)
+    if _tree_digest(chrome_destination, include_modes=False) != chrome_content:
+        raise AcceptanceFailure("private-chrome-copy-mismatch")
+    _assert_regular(
+        private_chrome_executable,
+        digest=CHROME_BINARY_SHA256,
+        executable=True,
+    )
+
+    private_node = bin_directory / "node"
+    private_wrapper = bin_directory / "playwright-wrapper"
+    try:
+        shutil.copyfile(NODE_SOURCE, private_node)
+        os.chmod(private_node, 0o700)
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-copy-failed") from exc
+    if _sha256(private_node) != NODE_SHA256:
+        raise AcceptanceFailure("private-toolchain-mismatch")
+    try:
+        shutil.copyfile(validated_wrapper, private_wrapper)
+        os.chmod(private_wrapper, 0o700)
+    except OSError as exc:
+        raise AcceptanceFailure("toolchain-copy-failed") from exc
+    if _sha256(private_wrapper) != WRAPPER_SHA256:
+        raise AcceptanceFailure("private-toolchain-mismatch")
+    cli_entry = cli_destination / "node_modules" / "@playwright" / "cli" / "playwright-cli.js"
+    adapter = bin_directory / "npx"
+    adapter_payload = f"""#!/usr/bin/env node
+'use strict';
+const args = process.argv.slice(2);
+if (args.length < 4 || args[0] !== '--yes' || args[1] !== '--package' || args[2] !== '@playwright/cli' || args[3] !== 'playwright-cli') process.exit(64);
+process.argv = [process.execPath, {json.dumps(str(cli_entry))}, ...args.slice(4)];
+require({json.dumps(str(cli_entry))});
+""".encode("utf-8")
+    _write_private(adapter, adapter_payload, mode=0o700)
+    atomic_json(global_root / ".playwright" / "cli.config.json", {})
+
+    sealed_tree = _seal_toolchain_tree(sealed_root)
+
+    environment = restricted_cli_environment(
+        bin_directory=bin_directory,
+        work_root=work_root,
+        daemon_root=daemon_root,
+        global_root=global_root,
+        chrome_executable=private_chrome_executable,
+    )
+    if _tree_digest(CACHED_CLI_ROOT, include_modes=True) != cli_tree:
+        raise AcceptanceFailure("cli-cache-changed")
+    if _tree_digest(CHROME_BUNDLE, include_modes=True) != chrome_tree:
+        raise AcceptanceFailure("chrome-bundle-changed")
+    _validate_node_libraries()
+    code, output, _ = _bounded_process(
+        [str(private_node), "--version"], cwd=work_root,
+        environment=environment, timeout=10, check="private-node-version",
+    )
+    if code != 0 or output.decode("ascii", "strict").strip() != NODE_VERSION:
+        raise AcceptanceFailure("private-node-version-mismatch")
+    code, output, _ = _bounded_process(
+        [str(private_chrome_executable), "--version"], cwd=work_root,
+        environment=environment, timeout=15, check="private-chrome-version",
+    )
+    if code != 0 or output.decode("ascii", "strict").strip() != CHROME_VERSION_OUTPUT:
+        raise AcceptanceFailure("private-chrome-version-mismatch")
+    if _verify_sealed_toolchain_tree(sealed_root) != sealed_tree:
+        raise AcceptanceFailure("private-toolchain-changed")
+    return Toolchain(
+        wrapper=private_wrapper,
+        environment=environment,
+        daemon_root=daemon_root,
+        global_root=global_root,
+        sealed_root=sealed_root,
+        chrome_executable=private_chrome_executable,
+        chrome_tree_sha256=chrome_tree[0],
+        sealed_tree_sha256=sealed_tree[0],
+        sealed_entry_count=sealed_tree[1],
+        sealed_file_bytes=sealed_tree[2],
+    )
+
+
+def build_cli_config(
+    *, width: int, height: int, origins: Sequence[str], spki: str,
+    output_dir: Path, chrome_executable: Path,
+) -> dict[str, object]:
+    if (width, height) not in VIEWPORTS:
+        raise AcceptanceFailure("unsupported-viewport")
+    if len(origins) != len(ORIGIN_NAMES) or len(origins) != len(set(origins)):
+        raise AcceptanceFailure("scenario-origins-not-distinct")
+    for index, origin in enumerate(origins):
+        validate_base_url(origin, field=f"origin-{index}")
+    validate_spki(spki)
+    _assert_private_directory(output_dir)
+    try:
+        chrome_metadata = chrome_executable.lstat()
+    except OSError as exc:
+        raise AcceptanceFailure("private-chrome-unavailable") from exc
+    if (
+        not stat.S_ISREG(chrome_metadata.st_mode)
+        or chrome_metadata.st_uid != os.getuid()
+        or chrome_metadata.st_mode & (stat.S_IWUSR | stat.S_IWGRP | stat.S_IWOTH)
+        or not os.access(chrome_executable, os.X_OK)
+    ):
+        raise AcceptanceFailure("private-chrome-permissions")
+    return {
+        "browser": {
+            "browserName": "chromium",
+            "isolated": True,
+            "launchOptions": {
+                "executablePath": str(chrome_executable),
+                "headless": True,
+                "args": [
+                    f"--ignore-certificate-errors-spki-list={spki}",
+                    "--disable-background-networking",
+                    "--disable-component-update",
+                    "--disable-default-apps",
+                    "--disable-domain-reliability",
+                    "--disable-features=OptimizationHints,MediaRouter",
+                    "--disable-sync",
+                    "--metrics-recording-only",
+                    "--no-first-run",
+                    "--safebrowsing-disable-auto-update",
+                ],
+                "timeout": 30_000,
+            },
+            "contextOptions": {
+                "acceptDownloads": False,
+                "colorScheme": "light",
+                "deviceScaleFactor": 1,
+                "forcedColors": "none",
+                "hasTouch": width <= 768,
+                "ignoreHTTPSErrors": False,
+                "isMobile": False,
+                "javaScriptEnabled": True,
+                "locale": "ko-KR",
+                "reducedMotion": "reduce",
+                "screen": {"width": width, "height": height},
+                "serviceWorkers": "block",
+                "timezoneId": "Asia/Seoul",
+                "viewport": {"width": width, "height": height},
+            },
+        },
+        "network": {"allowedOrigins": [f"{origin}/**" for origin in origins]},
+        "outputDir": str(output_dir),
+        "outputMode": "stdout",
+        "imageResponses": "omit",
+        "saveSession": False,
+        "saveTrace": False,
+        "saveVideo": False,
+        "timeouts": {"action": 10_000, "navigation": 30_000},
+    }
+
+
+def validate_base_url(value: str, *, field: str) -> str:
+    try:
+        parsed = urlsplit(value)
+        port = parsed.port
+    except (ValueError, TypeError) as exc:
+        raise AcceptanceFailure(f"invalid-{field}-url") from exc
+    if (
+        parsed.scheme != "https"
+        or parsed.hostname not in {"127.0.0.1", "localhost", "::1"}
+        or parsed.username is not None
+        or parsed.password is not None
+        or parsed.query
+        or parsed.fragment
+        or parsed.path not in {"", "/"}
+        or port is None
+        or not 1 <= port <= 65_535
+    ):
+        raise AcceptanceFailure(f"invalid-{field}-url")
+    return f"https://[::1]:{port}" if parsed.hostname == "::1" else f"https://{parsed.hostname}:{port}"
+
+
+def validate_spki(value: str) -> str:
+    try:
+        decoded = base64.b64decode(value, validate=True)
+    except (binascii.Error, ValueError) as exc:
+        raise AcceptanceFailure("invalid-certificate-spki") from exc
+    if len(decoded) != 32 or base64.b64encode(decoded).decode("ascii") != value:
+        raise AcceptanceFailure("invalid-certificate-spki")
+    return value
+
+
+def _der_tlv(document: bytes, offset: int) -> tuple[int, int, int]:
+    if offset + 2 > len(document):
+        raise AcceptanceFailure("tls-certificate-invalid")
+    tag = document[offset]
+    first = document[offset + 1]
+    cursor = offset + 2
+    if first & 0x80:
+        length_octets = first & 0x7F
+        if length_octets == 0 or length_octets > 4 or cursor + length_octets > len(document):
+            raise AcceptanceFailure("tls-certificate-invalid")
+        length = int.from_bytes(document[cursor : cursor + length_octets], "big")
+        if length < 128:
+            raise AcceptanceFailure("tls-certificate-invalid")
+        cursor += length_octets
+    else:
+        length = first
+    end = cursor + length
+    if end > len(document):
+        raise AcceptanceFailure("tls-certificate-invalid")
+    return tag, cursor, end
+
+
+def certificate_spki_sha256(certificate: bytes) -> str:
+    tag, certificate_body, certificate_end = _der_tlv(certificate, 0)
+    if tag != 0x30 or certificate_end != len(certificate):
+        raise AcceptanceFailure("tls-certificate-invalid")
+    tag, tbs_body, tbs_end = _der_tlv(certificate, certificate_body)
+    if tag != 0x30:
+        raise AcceptanceFailure("tls-certificate-invalid")
+    children: list[tuple[int, int, int]] = []
+    cursor = tbs_body
+    while cursor < tbs_end:
+        child_tag, _, child_end = _der_tlv(certificate, cursor)
+        children.append((child_tag, cursor, child_end))
+        cursor = child_end
+    spki_index = 6 if children and children[0][0] == 0xA0 else 5
+    if len(children) <= spki_index or children[spki_index][0] != 0x30:
+        raise AcceptanceFailure("tls-certificate-invalid")
+    _, start, end = children[spki_index]
+    return base64.b64encode(hashlib.sha256(certificate[start:end]).digest()).decode("ascii")
+
+
+def verify_tls_peer(origin: str, expected_spki: str) -> None:
+    parsed = urlsplit(validate_base_url(origin, field="tls-peer"))
+    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+    context.minimum_version = ssl.TLSVersion.TLSv1_2
+    context.check_hostname = False
+    context.verify_mode = ssl.CERT_NONE
+    try:
+        with socket.create_connection((parsed.hostname, parsed.port), timeout=5) as plain:
+            with context.wrap_socket(plain, server_hostname=parsed.hostname) as secured:
+                protocol = secured.version()
+                certificate = secured.getpeercert(binary_form=True)
+    except (OSError, ssl.SSLError) as exc:
+        raise AcceptanceFailure("tls-peer-unavailable") from exc
+    if protocol not in {"TLSv1.2", "TLSv1.3"} or not certificate:
+        raise AcceptanceFailure("tls-peer-protocol")
+    if certificate_spki_sha256(certificate) != validate_spki(expected_spki):
+        raise AcceptanceFailure("tls-peer-spki")
+
+
+def validate_release_document(
+    *, status: int, content_type: str, cache_control: str, body: bytes, expected_sha: str
+) -> None:
+    if status != 200 or len(body) > 1024:
+        raise AcceptanceFailure("release-endpoint-response")
+    if content_type.split(";", 1)[0].strip().lower() != "application/json":
+        raise AcceptanceFailure("release-endpoint-content-type")
+    if "no-store" not in {token.strip().lower() for token in cache_control.split(",")}:
+        raise AcceptanceFailure("release-endpoint-cache")
+    try:
+        document = json.loads(body.decode("utf-8"))
+    except (UnicodeError, json.JSONDecodeError) as exc:
+        raise AcceptanceFailure("release-endpoint-json") from exc
+    if document != {"release_sha": expected_sha} or not SAFE_RELEASE.fullmatch(expected_sha):
+        raise AcceptanceFailure("release-endpoint-mismatch")
+
+
+def _pid_alive(pid: int) -> bool:
+    try:
+        os.kill(pid, 0)
+        return True
+    except ProcessLookupError:
+        return False
+    except PermissionError:
+        return True
+
+
+def _wait_pid_gone(pid: int, seconds: float) -> bool:
+    deadline = time.monotonic() + seconds
+    while time.monotonic() < deadline:
+        if not _pid_alive(pid):
+            return True
+        time.sleep(0.05)
+    return not _pid_alive(pid)
+
+
+def _read_process_identity(
+    pid: int, *, working_directory: Path, check: str
+) -> ProcessIdentity | None:
+    if pid <= 0:
+        return None
+    code, output, _ = _bounded_process(
+        [
+            "/bin/ps", "-p", str(pid), "-o",
+            "uid=,pgid=,lstart=,command=",
+        ],
+        cwd=working_directory,
+        environment={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+        timeout=3,
+        check=check,
+    )
+    if code != 0:
+        return None
+    fields = output.decode("utf-8", "replace").strip().split(maxsplit=7)
+    if len(fields) != 8 or not fields[0].isdigit() or not fields[1].isdigit():
+        return None
+    return ProcessIdentity(
+        pid=pid,
+        uid=int(fields[0]),
+        process_group=int(fields[1]),
+        started=" ".join(fields[2:7]),
+        command=fields[7],
+    )
+
+
+def _private_browser_processes(
+    toolchain: Toolchain, working_directory: Path
+) -> dict[int, ProcessIdentity]:
+    private_root = str(Path(toolchain.environment["HOME"]).parent.resolve(strict=True))
+    code, output, _ = _bounded_process(
+        ["/usr/bin/pgrep", "-U", str(os.getuid()), "-f", re.escape(private_root)],
+        cwd=working_directory,
+        environment={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+        timeout=3,
+        check="browser-process-scan",
+    )
+    if code not in {0, 1}:
+        raise AcceptanceFailure("browser-process-scan-failed")
+    matches: dict[int, ProcessIdentity] = {}
+    for value in output.decode("ascii", "replace").splitlines():
+        if not value.isascii() or not value.isdigit():
+            continue
+        try:
+            pid = int(value)
+        except ValueError:
+            continue
+        identity = _read_process_identity(
+            pid, working_directory=working_directory,
+            check="browser-process-identity",
+        )
+        if identity is None:
+            continue
+        if (
+            identity.uid == os.getuid()
+            and pid != os.getpid()
+            and private_root in identity.command
+            and (
+                "/playwright-core/lib/entry/cliDaemon.js " in identity.command
+                or str(toolchain.chrome_executable) in identity.command
+            )
+        ):
+            matches[pid] = identity
+    return matches
+
+
+def _wait_private_processes_gone(
+    toolchain: Toolchain, working_directory: Path, seconds: float
+) -> dict[int, ProcessIdentity]:
+    deadline = time.monotonic() + seconds
+    while time.monotonic() < deadline:
+        matches = _private_browser_processes(toolchain, working_directory)
+        if not matches:
+            return {}
+        time.sleep(0.05)
+    return _private_browser_processes(toolchain, working_directory)
+
+
+def _signal_verified_process(
+    expected: ProcessIdentity,
+    *, working_directory: Path,
+    requested_signal: signal.Signals,
+) -> None:
+    current = _read_process_identity(
+        expected.pid,
+        working_directory=working_directory,
+        check="browser-process-revalidate",
+    )
+    if current is None:
+        return
+    if current != expected or current.uid != os.getuid():
+        raise AcceptanceFailure("browser-process-identity")
+    try:
+        if current.process_group == current.pid:
+            os.killpg(current.process_group, requested_signal)
+        else:
+            os.kill(current.pid, requested_signal)
+    except ProcessLookupError:
+        pass
+    except OSError as exc:
+        raise AcceptanceFailure("browser-process-signal") from exc
+
+
+def _remove_daemon_residue(root: Path, *, allow_err_only: bool) -> None:
+    try:
+        _assert_private_directory(root)
+        unexpected = False
+        for path in sorted(
+            root.rglob("*"), key=lambda item: len(item.parts), reverse=True
+        ):
+            metadata = path.lstat()
+            if metadata.st_uid != os.getuid() or stat.S_ISLNK(metadata.st_mode):
+                raise AcceptanceFailure("daemon-residue-unsafe")
+            if stat.S_ISDIR(metadata.st_mode):
+                path.rmdir()
+            elif stat.S_ISREG(metadata.st_mode):
+                if not (allow_err_only and path.name.endswith(".err")):
+                    unexpected = True
+                path.unlink()
+            elif stat.S_ISSOCK(metadata.st_mode):
+                unexpected = True
+                path.unlink()
+            else:
+                raise AcceptanceFailure("daemon-residue-unsafe")
+        if any(root.iterdir()):
+            raise AcceptanceFailure("daemon-residue-cleanup")
+        if unexpected:
+            raise AcceptanceFailure("daemon-unexpected-residue")
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("daemon-residue-cleanup") from exc
+
+
+def _enter_cleanup_signal_guard() -> bool:
+    global _SIGNAL_CLEANUP_DEPTH
+    interrupted_before = _SIGNAL_INTERRUPTED
+    _SIGNAL_CLEANUP_DEPTH += 1
+    return interrupted_before
+
+
+def _leave_cleanup_signal_guard(interrupted_before: bool) -> bool:
+    global _SIGNAL_CLEANUP_DEPTH
+    if _SIGNAL_CLEANUP_DEPTH <= 0:
+        raise AcceptanceFailure("cleanup-signal-state")
+    _SIGNAL_CLEANUP_DEPTH -= 1
+    return not interrupted_before and _SIGNAL_INTERRUPTED
+
+
+class CliSession:
+    """One isolated CLI daemon with bounded output and deterministic cleanup."""
+
+    def __init__(
+        self, *, session_name: str, config_path: Path,
+        working_directory: Path, toolchain: Toolchain,
+    ) -> None:
+        if not SAFE_SLUG.fullmatch(session_name):
+            raise AcceptanceFailure("invalid-cli-session")
+        self.session_name = session_name
+        self.config_path = config_path
+        self.working_directory = working_directory
+        self.toolchain = toolchain
+        self.opened = False
+        self.daemon_pid: int | None = None
+        self.daemon_identity: ProcessIdentity | None = None
+
+    def command(self, *arguments: str) -> list[str]:
+        return [
+            str(BASH_SOURCE), str(self.toolchain.wrapper),
+            "--session", self.session_name, *arguments,
+        ]
+
+    def invoke(
+        self, check: str, *arguments: str, marker: str | None = None,
+        timeout: int = CLI_TIMEOUT_SECONDS,
+    ) -> bytes:
+        if marker is not None and not SAFE_MARKER.fullmatch(marker):
+            raise AcceptanceFailure("invalid-cli-marker")
+        code, output, error = _bounded_process(
+            self.command(*arguments), cwd=self.working_directory,
+            environment=self.toolchain.environment, timeout=timeout, check=check,
+        )
+        if code != 0:
+            if check == "open":
+                # A failed launch can still have created a daemon. Recover only
+                # its bounded numeric PID; never retain or expose error output.
+                candidates = {
+                    int(match)
+                    for match in re.findall(rb"\bDaemon pid=([1-9][0-9]{0,10})\b", error)
+                }
+                if len(candidates) == 1:
+                    candidate = candidates.pop()
+                    if _pid_alive(candidate):
+                        self.daemon_pid = candidate
+            safe_codes = {
+                match.decode("ascii")
+                for match in re.findall(
+                    rb"acceptance:([a-z0-9][a-z0-9-]{0,63})",
+                    output + b"\n" + error,
+                )
+            }
+            if len(safe_codes) == 1:
+                detailed = f"cli-{check}-{safe_codes.pop()}"
+                if SAFE_CHECK.fullmatch(detailed):
+                    raise AcceptanceFailure(detailed)
+            combined = output + b"\n" + error
+            safe_diagnostics = {
+                token
+                for needle, token in (
+                    (b"Timeout", "timeout-detail"),
+                    (b"waitForURL", "wait-url-detail"),
+                    (b"Execution context was destroyed", "context-destroyed"),
+                    (b"strict mode violation", "strict-locator"),
+                    (b"route.continue", "route-continue"),
+                    (b"allHeaders", "request-headers"),
+                    (b"net::ERR", "network-detail"),
+                    (b"Target page, context or browser has been closed", "target-closed"),
+                )
+                if needle in combined
+            }
+            if len(safe_diagnostics) == 1:
+                raise AcceptanceFailure(
+                    f"cli-{check}-{safe_diagnostics.pop()}"
+                )
+            raise AcceptanceFailure(f"cli-{check}-failed")
+        if marker is not None:
+            try:
+                lines = output.decode("utf-8", "strict").splitlines()
+            except UnicodeError as exc:
+                raise AcceptanceFailure(f"cli-{check}-output-invalid") from exc
+            accepted = {marker, json.dumps(marker)}
+            if not any(line.strip() in accepted for line in lines):
+                raise AcceptanceFailure(f"cli-{check}-assertion-missing")
+        return output
+
+    def open(self, url: str) -> None:
+        output = self.invoke("open", "open", url, f"--config={self.config_path}")
+        try:
+            text = output.decode("utf-8", "strict")
+        except UnicodeError as exc:
+            raise AcceptanceFailure("cli-open-output-invalid") from exc
+        pattern = re.compile(
+            rf"\A### Browser `{re.escape(self.session_name)}` opened with pid ([1-9][0-9]*)\.\Z"
+        )
+        recovery = {
+            int(value)
+            for value in re.findall(r"\bopened with pid ([1-9][0-9]{0,10})\b", text)
+        }
+        if len(recovery) == 1:
+            candidate = recovery.pop()
+            if _pid_alive(candidate):
+                self.daemon_pid = candidate
+        matches = [pattern.fullmatch(line.strip()) for line in text.splitlines()]
+        pids = [int(match.group(1)) for match in matches if match is not None]
+        if len(pids) != 1 or not _pid_alive(pids[0]):
+            raise AcceptanceFailure("cli-open-pid-missing")
+        self.daemon_pid = pids[0]
+        self.daemon_identity = self._daemon_process_identity(pids[0])
+        if self.daemon_identity is None:
+            raise AcceptanceFailure("cli-open-process-identity")
+        self.opened = True
+
+    def goto(self, url: str, *, check: str) -> None:
+        self.invoke(check, "goto", url)
+
+    def snapshot(self, *, check: str, required_tokens: Sequence[str]) -> None:
+        snapshot = self.working_directory / f"transient-{check}.md"
+        try:
+            self.invoke(check, "snapshot", f"--filename={snapshot}", "--depth=8")
+            metadata = snapshot.lstat()
+            if (
+                not stat.S_ISREG(metadata.st_mode)
+                or metadata.st_uid != os.getuid()
+                or metadata.st_size > MAX_SNAPSHOT_BYTES
+            ):
+                raise AcceptanceFailure("snapshot-invalid")
+            with snapshot.open("rb") as handle:
+                payload = handle.read(MAX_SNAPSHOT_BYTES + 1)
+            if len(payload) > MAX_SNAPSHOT_BYTES:
+                raise AcceptanceFailure("snapshot-too-large")
+            document = payload.decode("utf-8", "strict")
+            if not required_tokens or any(token not in document for token in required_tokens):
+                raise AcceptanceFailure("snapshot-semantic-token")
+        except AcceptanceFailure:
+            raise
+        except (OSError, UnicodeError) as exc:
+            raise AcceptanceFailure("snapshot-read-failed") from exc
+        finally:
+            try:
+                snapshot.unlink(missing_ok=True)
+            except OSError as exc:
+                raise AcceptanceFailure("snapshot-cleanup-failed") from exc
+
+    def run_code(self, check: str, code: str) -> None:
+        marker = f"PW_ACCEPTANCE_OK:{check}"
+        self.invoke(check, "run-code", code, marker=marker)
+
+    def screenshot(self, relative_name: str, *, check: str) -> None:
+        self.invoke(check, "screenshot", f"--filename={relative_name}", "--full-page")
+
+    def close(self) -> None:
+        interrupted_before = _enter_cleanup_signal_guard()
+        failures: list[AcceptanceFailure] = []
+        deferred_interrupt = False
+        try:
+            if self.opened:
+                try:
+                    self.invoke("close", "close", timeout=30)
+                except AcceptanceFailure as exc:
+                    failures.append(exc)
+                self.opened = False
+            daemon_pid = self.daemon_pid
+            if daemon_pid is not None and not _wait_pid_gone(daemon_pid, 5):
+                try:
+                    if not self._daemon_identity_matches(daemon_pid):
+                        raise AcceptanceFailure("daemon-process-identity")
+                    identity = self._daemon_process_identity(daemon_pid)
+                    if identity is None or (
+                        self.daemon_identity is not None
+                        and identity != self.daemon_identity
+                    ):
+                        raise AcceptanceFailure("daemon-process-identity")
+                    _signal_verified_process(
+                        identity,
+                        working_directory=self.working_directory,
+                        requested_signal=signal.SIGTERM,
+                    )
+                    if not _wait_pid_gone(daemon_pid, 2):
+                        current = self._daemon_process_identity(daemon_pid)
+                        if current is None or (
+                            self.daemon_identity is not None
+                            and current != self.daemon_identity
+                        ):
+                            raise AcceptanceFailure("daemon-process-identity")
+                        _signal_verified_process(
+                            current,
+                            working_directory=self.working_directory,
+                            requested_signal=signal.SIGKILL,
+                        )
+                        if not _wait_pid_gone(daemon_pid, 2):
+                            raise AcceptanceFailure("daemon-process-reap-failed")
+                    failures.append(AcceptanceFailure("daemon-forced-cleanup"))
+                except AcceptanceFailure as exc:
+                    failures.append(exc)
+            self.daemon_pid = None
+            self.daemon_identity = None
+            try:
+                _remove_daemon_residue(
+                    self.toolchain.daemon_root, allow_err_only=True
+                )
+            except AcceptanceFailure as exc:
+                failures.append(exc)
+            try:
+                remaining = _wait_private_processes_gone(
+                    self.toolchain, self.working_directory, 3
+                )
+                if remaining:
+                    for identity in remaining.values():
+                        try:
+                            _signal_verified_process(
+                                identity,
+                                working_directory=self.working_directory,
+                                requested_signal=signal.SIGTERM,
+                            )
+                        except AcceptanceFailure as exc:
+                            failures.append(exc)
+                    remaining = _wait_private_processes_gone(
+                        self.toolchain, self.working_directory, 2
+                    )
+                    for identity in remaining.values():
+                        try:
+                            _signal_verified_process(
+                                identity,
+                                working_directory=self.working_directory,
+                                requested_signal=signal.SIGKILL,
+                            )
+                        except AcceptanceFailure as exc:
+                            failures.append(exc)
+                    if _wait_private_processes_gone(
+                        self.toolchain, self.working_directory, 2
+                    ):
+                        failures.append(
+                            AcceptanceFailure("browser-process-reap-failed")
+                        )
+                    else:
+                        failures.append(
+                            AcceptanceFailure("browser-forced-cleanup")
+                        )
+            except AcceptanceFailure as exc:
+                failures.append(exc)
+        finally:
+            deferred_interrupt = _leave_cleanup_signal_guard(interrupted_before)
+        if failures:
+            raise failures[0]
+        if deferred_interrupt:
+            raise AcceptanceFailure("interrupted")
+
+    def _daemon_process_identity(self, pid: int) -> ProcessIdentity | None:
+        identity = _read_process_identity(
+            pid,
+            working_directory=self.working_directory,
+            check="daemon-identity",
+        )
+        try:
+            expected_node = str(
+                (Path(self.toolchain.environment["PATH"]) / "node").resolve(strict=True)
+            )
+        except OSError:
+            return None
+        if identity is None:
+            return None
+        if (
+            identity.uid != os.getuid()
+            or not identity.command.startswith(expected_node + " ")
+            or "/playwright-core/lib/entry/cliDaemon.js " not in identity.command
+            or f" {self.session_name} " not in identity.command
+        ):
+            return None
+        return identity
+
+    def _daemon_identity_matches(self, pid: int) -> bool:
+        current = self._daemon_process_identity(pid)
+        if current is None:
+            return not _pid_alive(pid)
+        return self.daemon_identity is None or current == self.daemon_identity
+
+
+def _marker(check: str) -> str:
+    return json.dumps(f"PW_ACCEPTANCE_OK:{check}")
+
+
+def install_guards_javascript(*, origins: Sequence[str], check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  const context = page.context();
+  if ((await context.cookies()).length !== 0) fail('initial-cookie-state');
+  const allowed = {json.dumps(list(origins))};
+  const allowedGets = new Set(allowed.flatMap((origin) => [origin + '/', origin + '/results/', origin + '/releasez', origin + '/favicon.ico', origin + '/static/public_web/site.css', origin + '/static/public_web/site.js']));
+  const allowedPosts = new Set(allowed.map((origin) => origin + '/'));
+  context.__acceptanceGuard = {{ external: 0, unexpected: 0, posts: 0 }};
+  context.__acceptanceRequestListener = (request) => {{
+    const address = request.url(); const method = request.method();
+    const sameOrigin = allowed.some((origin) => address === origin || address.startsWith(`${{origin}}/`));
+    if (!sameOrigin) context.__acceptanceGuard.external += 1;
+    else if (method === 'GET' && !allowedGets.has(address)) context.__acceptanceGuard.unexpected += 1;
+    else if (method === 'POST') {{ context.__acceptanceGuard.posts += 1; if (!allowedPosts.has(address)) context.__acceptanceGuard.unexpected += 1; }}
+    else if (method !== 'GET') context.__acceptanceGuard.unexpected += 1;
+  }};
+  context.on('request', context.__acceptanceRequestListener);
+  return {_marker(check)};
+}}"""
+
+
+def _client_state_source() -> str:
+    return """const clientState = async () => {
+    const databases = typeof indexedDB.databases === 'function' ? await indexedDB.databases() : null;
+    const cacheKeys = 'caches' in self ? await caches.keys() : [];
+    const registrations = 'serviceWorker' in navigator ? await navigator.serviceWorker.getRegistrations() : [];
+    return { databases, cacheKeys, registrations, local: localStorage.length, session: sessionStorage.length };
+  };
+  const assertClientState = async () => {
+    const state = await page.evaluate(clientState);
+    if (state.databases === null || state.databases.length || state.cacheKeys.length || state.registrations.length || state.local || state.session) fail('client-storage');
+  };"""
+
+
+def _static_asset_integrity_source(origin: str) -> str:
+    return f"""const staticAssets = {json.dumps([
+      {"path": "/static/public_web/site.css", "sha256": SITE_CSS_SHA256, "bytes": SITE_CSS_BYTES, "types": ["text/css"]},
+      {"path": "/static/public_web/site.js", "sha256": SITE_JS_SHA256, "bytes": SITE_JS_BYTES, "types": ["text/javascript", "application/javascript"]},
+  ], separators=(",", ":"))};
+  for (const asset of staticAssets) {{
+    const assetUrl = {json.dumps(origin)} + asset.path;
+    const awaited = page.waitForResponse((response) => response.url() === assetUrl && response.request().method() === 'GET');
+    const observed = await page.evaluate(async (contract) => {{
+      const response = await fetch(contract.url, {{ credentials: 'omit', cache: 'no-store', redirect: 'error', referrerPolicy: 'no-referrer' }});
+      const bytes = new Uint8Array(await response.arrayBuffer());
+      const digest = [...new Uint8Array(await crypto.subtle.digest('SHA-256', bytes))].map((value) => value.toString(16).padStart(2, '0')).join('');
+      return {{ status: response.status, contentType: response.headers.get('content-type') || '', byteLength: bytes.byteLength, digest }};
+    }}, {{ url: assetUrl }});
+    const response = await awaited; const details = await response.securityDetails(); const responseHeaders = await response.allHeaders(); const requestHeaders = await response.request().allHeaders();
+    if (!details || !['TLS 1.2', 'TLS 1.3'].includes(details.protocol) || 'set-cookie' in responseHeaders || 'cookie' in requestHeaders || response.request().redirectedFrom()) fail('static-transport');
+    if (observed.status !== 200 || observed.byteLength !== asset.bytes || observed.digest !== asset.sha256 || !asset.types.includes(observed.contentType.split(';')[0].trim().toLowerCase())) fail('static-digest');
+  }}"""
+
+
+def _dom_privacy_source(*, origin: str, path: str) -> str:
+    if path not in {"/", "/results/"}:
+        raise AcceptanceFailure("invalid-dom-privacy-path")
+    form_page = path == "/"
+    expected_title = (
+        "여행준비 — 일본 정보 확인" if form_page else "여행준비 — 일본 게시 정보"
+    )
+    approved_hrefs = [
+        "#main-content", "/", "/results/", "/static/public_web/site.css",
+        "#id_destination", "#id_departure_date", "#id_return_date", "#trip-form",
+        ENTRY_PUBLIC_SOURCE_LOCATOR, WARNING_PUBLIC_SOURCE_LOCATOR,
+    ]
+    return f"""const privacyFailure = await page.evaluate((contract) => {{
+    const text = (node) => node.textContent.replace(/\\s+/g, ' ').trim();
+    const headChildren = [...document.head.children];
+    if (headChildren.map((node) => node.tagName).join('|') !== 'META|META|META|TITLE|LINK') return 'head';
+    if (headChildren[0].getAttributeNames().join(',') !== 'charset' || headChildren[0].getAttribute('charset').toLowerCase() !== 'utf-8') return 'charset';
+    if (headChildren[1].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[1].getAttribute('name') !== 'viewport' || headChildren[1].getAttribute('content') !== 'width=device-width, initial-scale=1') return 'viewport-meta';
+    if (headChildren[2].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[2].getAttribute('name') !== 'theme-color' || headChildren[2].getAttribute('content') !== '#123f46') return 'theme-meta';
+    if (headChildren[3].getAttributeNames().length || text(headChildren[3]) !== contract.title) return 'title';
+    if (headChildren[4].getAttributeNames().sort().join(',') !== 'href,rel' || headChildren[4].getAttribute('rel') !== 'stylesheet' || headChildren[4].getAttribute('href') !== '/static/public_web/site.css' || headChildren[4].href !== contract.origin + '/static/public_web/site.css') return 'style-link';
+    const scripts = [...document.scripts];
+    if (scripts.length !== 1 || scripts[0].getAttributeNames().sort().join(',') !== 'defer,src' || scripts[0].getAttribute('src') !== '/static/public_web/site.js' || scripts[0].src !== contract.origin + '/static/public_web/site.js' || scripts[0].textContent.trim()) return 'script';
+    const hidden = [...document.querySelectorAll('input[type="hidden"]')];
+    if (contract.form) {{
+      if (hidden.length !== 1 || hidden[0].getAttributeNames().sort().join(',') !== 'name,type,value' || hidden[0].getAttribute('name') !== 'csrfmiddlewaretoken' || !hidden[0].getAttribute('value')) return 'hidden-input';
+    }} else if (hidden.length) return 'hidden-input';
+    const allowedNames = new Set(['action','aria-atomic','aria-busy','aria-current','aria-describedby','aria-hidden','aria-invalid','aria-label','aria-labelledby','aria-live','aria-disabled','autocomplete','charset','class','content','data-error-summary','data-state','data-submit-button','data-submit-label','data-submit-status','data-submitting','data-trip-form','defer','for','href','id','lang','method','name','novalidate','rel','required','role','selected','src','style','tabindex','type','value']);
+    const approvedHrefs = new Set(contract.href); const approvedSources = new Set(['/static/public_web/site.js']);
+    const forbidden = /(?:MOFA_TRAVEL_ALARM_SERVICE_KEY|serviceKey|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
+    const opaque = /[a-z0-9+/_=-]{{32,}}/i; const encoded = /(?:%[0-9a-f]{{2}}){{4,}}/i;
+    const attributeLeak = [...document.querySelectorAll('*')].some((node) => [...node.attributes].some((attribute) => {{
+      if (!allowedNames.has(attribute.name)) return true;
+      if (attribute.name === 'href') return !approvedHrefs.has(attribute.value);
+      if (attribute.name === 'src') return !approvedSources.has(attribute.value);
+      if (attribute.name === 'action') return attribute.value !== '/';
+      if (attribute.name === 'style') return node !== document.documentElement || attribute.value !== 'font-size: 200%;';
+      if (attribute.name === 'value' && node === hidden[0]) return false;
+      const value = `${{attribute.name}}=${{attribute.value}}`;
+      return forbidden.test(value) || /[<>{{}}]/.test(attribute.value) || opaque.test(attribute.value) || encoded.test(attribute.value);
+    }}));
+    if (attributeLeak) return 'attribute';
+    const commentWalker = document.createTreeWalker(document, NodeFilter.SHOW_COMMENT);
+    while (commentWalker.nextNode()) if ((commentWalker.currentNode.nodeValue || '').trim()) return 'comment';
+    const bodyText = document.body.textContent || '';
+    if (bodyText.length > 100000 || forbidden.test(bodyText) || /[<>{{}}]/.test(bodyText) || encoded.test(bodyText) || opaque.test(bodyText)) return 'text';
+    return null;
+  }}, {{ origin: {json.dumps(origin)}, title: {json.dumps(expected_title)}, form: {json.dumps(form_page)}, href: {json.dumps(approved_hrefs)} }});
+  if (privacyFailure) fail(`dom-privacy-${{privacyFailure}}`);"""
+
+
+def _common_javascript(*, origin: str, path: str, check: str) -> str:
+    static_integrity = _static_asset_integrity_source(origin) if path == "/" else ""
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_client_state_source()}
+  await page.waitForLoadState('networkidle');
+  if (page.url() !== {json.dumps(origin + path)}) fail('canonical-location');
+  if (!page.context().__acceptanceGuard || page.context().__acceptanceGuard.external !== 0 || page.context().__acceptanceGuard.unexpected !== 0) fail('request-attempt');
+  await assertClientState();
+  const viewport = page.viewportSize();
+  const failure = await page.evaluate((expected) => {{
+    if (window.innerWidth !== expected.width || window.innerHeight !== expected.height) return 'viewport';
+    if (document.documentElement.lang !== 'ko') return 'language';
+    if (document.querySelectorAll('h1').length !== 1) return 'one-h1';
+    for (const selector of ['header', 'nav[aria-label]', 'main', 'footer']) if (!document.querySelector(selector)) return 'landmark';
+    const ids = [...document.querySelectorAll('[id]')].map((node) => node.id);
+    if (ids.length !== new Set(ids).size) return 'duplicate-id';
+    if (document.documentElement.scrollWidth > window.innerWidth + 1 || document.body.scrollWidth > window.innerWidth + 1) return 'horizontal-overflow';
+    const bodySize = Number.parseFloat(getComputedStyle(document.body).fontSize);
+    const bodyLine = Number.parseFloat(getComputedStyle(document.body).lineHeight);
+    const h1Size = Number.parseFloat(getComputedStyle(document.querySelector('h1')).fontSize);
+    if (!(bodySize >= 16 && bodyLine / bodySize >= 1.45 && h1Size > bodySize)) return 'typography';
+    const targets = [...document.querySelectorAll('a, button, input:not([type="hidden"]), select, [tabindex="0"]')].filter((node) => {{
+      const style = getComputedStyle(node); const rect = node.getBoundingClientRect();
+      return style.display !== 'none' && style.visibility !== 'hidden' && rect.width > 0 && rect.height > 0;
+    }});
+    if (!targets.length || targets.some((node) => {{ const rect = node.getBoundingClientRect(); return rect.width < 44 || rect.height < 44; }})) return 'touch-target';
+    return null;
+  }}, {{ width: viewport && viewport.width, height: viewport && viewport.height }});
+  if (!viewport || failure) fail(failure || 'viewport');
+  if (await page.getByRole('heading', {{ level: 1 }}).count() !== 1) fail('heading-role');
+  if (await page.getByRole('navigation', {{ name: '주요 메뉴', exact: true }}).count() !== 1) fail('navigation-name');
+  {_dom_privacy_source(origin=origin, path=path)}
+  {static_integrity}
+  return {_marker(check)};
+}}"""
+
+
+def _dynamic_layout_source() -> str:
+    return """const dynamicLayoutFailure = await page.evaluate(() => {
+    if (document.documentElement.scrollWidth > window.innerWidth + 1 || document.body.scrollWidth > window.innerWidth + 1) return 'horizontal-overflow';
+    const bodySize = Number.parseFloat(getComputedStyle(document.body).fontSize);
+    const bodyLine = Number.parseFloat(getComputedStyle(document.body).lineHeight);
+    if (!(bodySize >= 16 && bodyLine / bodySize >= 1.45)) return 'typography';
+    const targets = [...document.querySelectorAll('a, button, input:not([type="hidden"]), select, [tabindex="0"]')].filter((node) => {
+      const style = getComputedStyle(node); const rect = node.getBoundingClientRect();
+      return style.display !== 'none' && style.visibility !== 'hidden' && rect.width > 0 && rect.height > 0;
+    });
+    return !targets.length || targets.some((node) => { const rect = node.getBoundingClientRect(); return rect.width < 44 || rect.height < 44; }) ? 'touch-target' : null;
+  });
+  if (dynamicLayoutFailure) fail(dynamicLayoutFailure);"""
+
+
+def verify_release_javascript(*, origin: str, release_sha: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  const url = {json.dumps(origin + '/releasez')};
+  const awaited = page.waitForResponse((response) => response.url() === url && response.request().method() === 'GET');
+  const observed = await page.evaluate(async (input) => {{
+    const response = await fetch(input.url, {{ credentials: 'omit', cache: 'no-store', redirect: 'error', referrerPolicy: 'no-referrer' }});
+    const body = await response.text();
+    return {{ status: response.status, contentType: response.headers.get('content-type') || '', cacheControl: response.headers.get('cache-control') || '', body }};
+  }}, {{ url }});
+  const response = await awaited;
+  const details = await response.securityDetails();
+  const transportHeaders = await response.allHeaders();
+  const requestHeaders = await response.request().allHeaders();
+  if (!details || !['TLS 1.2', 'TLS 1.3'].includes(details.protocol)) fail('release-tls');
+  if ('set-cookie' in transportHeaders || 'cookie' in requestHeaders || response.request().redirectedFrom()) fail('release-transport');
+  if (observed.status !== 200 || observed.body.length > 1024 || observed.contentType.split(';')[0].trim().toLowerCase() !== 'application/json') fail('release-response');
+  if (!observed.cacheControl.toLowerCase().split(',').map((item) => item.trim()).includes('no-store')) fail('release-cache');
+  let document; try {{ document = JSON.parse(observed.body); }} catch {{ fail('release-json'); }}
+  if (!document || Object.keys(document).length !== 1 || document.release_sha !== {json.dumps(release_sha)}) fail('release-sha');
+  return {_marker(check)};
+}}"""
+
+
+def form_pristine_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await ({_common_javascript(origin=origin, path='/', check=check)})(page);
+  for (const name of ['목적지', '출국일', '귀국일']) {{
+    const locator = page.getByLabel(name, {{ exact: true }}); if (await locator.count() !== 1) fail('label-role');
+  }}
+  if (await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).count() !== 1) fail('submit-name');
+  const formFailure = await page.evaluate(() => {{
+    for (const id of ['id_destination', 'id_departure_date', 'id_return_date']) {{
+      const control = document.getElementById(id); const label = document.querySelector(`label[for="${{id}}"]`);
+      if (!control || !label || !label.textContent.trim()) return 'native-label';
+      for (const described of (control.getAttribute('aria-describedby') || '').split(/\\s+/).filter(Boolean)) if (!document.getElementById(described)) return 'description-link';
+    }}
+    if (document.getElementById('id_departure_date').value || document.getElementById('id_return_date').value) return 'pristine-date';
+    if (document.getElementById('id_destination').value !== 'JP') return 'fixed-destination';
+    return null;
+  }});
+  if (formFailure) fail(formFailure);
+  await page.evaluate(() => document.activeElement && document.activeElement.blur());
+  const order = ['.skip-link', '.site-nav a:nth-of-type(1)', '.site-nav a:nth-of-type(2)', '#id_destination', '#id_departure_date', '#id_return_date', '[data-submit-button]'];
+  for (const selector of order) {{
+    await page.keyboard.press('Tab');
+    const focusFailure = await page.evaluate((candidate) => {{
+      if (!document.activeElement || !document.activeElement.matches(candidate)) return 'tab-order';
+      const style = getComputedStyle(document.activeElement); return style.outlineStyle === 'none' || Number.parseFloat(style.outlineWidth) < 2 ? 'visible-focus' : null;
+    }}, selector);
+    if (focusFailure) fail(focusFailure);
+  }}
+  return {_marker(check)};
+}}"""
+
+
+def _submission_support(origin: str) -> str:
+    return f"""{_client_state_source()}
+  const validateSubmitCookie = async () => {{
+    await assertClientState();
+    const context = page.context();
+    const cookies = await context.cookies();
+    if (cookies.length !== 1 || cookies[0].name !== 'csrftoken' || !cookies[0].value || !cookies[0].secure || !cookies[0].httpOnly || cookies[0].sameSite !== 'Strict') fail('csrf-cookie-contract');
+  }};
+  const installSubmitObservers = async () => {{
+    const guard = page.context().__acceptanceGuard;
+    if (!guard || guard.external !== 0 || guard.unexpected !== 0) fail('request-attempt');
+    const state = {{ postCount: 0, cookiePostCount: 0, statuses: [], guardPostStart: guard.posts }};
+    state.route = async (route) => {{
+      const request = route.request();
+      if (request.method() === 'POST' && request.url() === {json.dumps(origin + '/')}) {{
+        state.postCount += 1;
+        const headers = await request.allHeaders();
+        if (Object.prototype.hasOwnProperty.call(headers, 'cookie')) state.cookiePostCount += 1;
+        await route.continue();
+      }} else await route.continue();
+    }};
+    state.response = (response) => {{
+      const request = response.request();
+      if (request.method() === 'POST' && response.url() === {json.dumps(origin + '/')}) state.statuses.push(response.status());
+    }};
+    await page.route({json.dumps(origin + '/')}, state.route);
+    page.on('response', state.response);
+    page.__acceptanceSubmit = state;
+    return state;
+  }};
+  const prepareSubmit = async () => {{
+    await validateSubmitCookie();
+    return installSubmitObservers();
+  }};
+  const finishSubmit = async (expectedStatus, clearCookie = true) => {{
+    const state = page.__acceptanceSubmit; if (!state) fail('submit-state');
+    page.off('response', state.response); await page.unroute({json.dumps(origin + '/')}, state.route); delete page.__acceptanceSubmit;
+    const guard = page.context().__acceptanceGuard;
+    if (!guard || guard.external !== 0 || guard.unexpected !== 0 || guard.posts - state.guardPostStart !== 1 || state.postCount !== 1 || state.cookiePostCount !== 1 || state.statuses.length !== 1 || state.statuses[0] !== expectedStatus) fail('single-post-status');
+    if (clearCookie) {{
+      await page.context().clearCookies();
+      if ((await page.context().cookies()).length !== 0) fail('post-submit-cookie-state');
+    }} else {{
+      await validateSubmitCookie();
+    }}
+    await assertClientState();
+  }};"""
+
+
+def loading_start_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await ({_common_javascript(origin=origin, path='/', check=check)})(page);
+  {_submission_support(origin)}
+  await prepareSubmit();
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
+  await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
+  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.keyboard.press('Enter');
+  await page.waitForFunction(() => document.querySelector('[data-trip-form]')?.getAttribute('aria-busy') === 'true');
+  const loading = await page.evaluate(() => {{
+    const form = document.querySelector('[data-trip-form]'); const button = document.querySelector('[data-submit-button]'); const status = document.querySelector('[data-submit-status]');
+    return form?.getAttribute('aria-busy') === 'true' && button?.getAttribute('aria-disabled') === 'true' && button.textContent.includes('제출 중') && status?.textContent.includes('불러오는 중') && status.getAttribute('role') === 'status' && status.getAttribute('aria-live') === 'polite';
+  }});
+  if (!loading) fail('loading-semantics');
+  {_dom_privacy_source(origin=origin, path='/')}
+  {_dynamic_layout_source()}
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_submit_javascript(*, origin: str, check: str) -> str:
+    """Exercise the production browser-owned CSRF/single-POST helper."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  if (page.url() !== {json.dumps(origin + '/')}) fail('csrf-form-location');
+  await prepareSubmit();
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
+  await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
+  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.keyboard.press('Enter');
+  await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }});
+  await finishSubmit(303);
+  if (page.url() !== {json.dumps(origin + '/results/')}) fail('csrf-submit-location');
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_prepare_javascript(*, origin: str, check: str) -> str:
+    """Install POST accounting while keeping the CSRF cookie browser-owned."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  if (page.url() !== {json.dumps(origin + '/')}) fail('csrf-form-location');
+  await validateSubmitCookie();
+  return {_marker(check)};
+}}"""
+
+
+def csrf_contract_probe_javascript(
+    *, origin: str, aspect: str, check: str
+) -> str:
+    """Assert one non-secret browser/CSRF property with a fixed stage name."""
+
+    validate_base_url(origin, field="csrf-regression")
+    expressions = {
+        "location": "page.url() === " + json.dumps(origin + "/"),
+        "client-state": None,
+        "count": "cookies.length === 1",
+        "name": "cookies.length === 1 && cookies[0].name === 'csrftoken'",
+        "value-present": "cookies.length === 1 && Boolean(cookies[0].value)",
+        "secure": "cookies.length === 1 && cookies[0].secure === true",
+        "http-only": "cookies.length === 1 && cookies[0].httpOnly === true",
+        "same-site": "cookies.length === 1 && cookies[0].sameSite === 'Strict'",
+    }
+    if aspect not in expressions:
+        raise AcceptanceFailure("invalid-csrf-probe")
+    if aspect == "client-state":
+        assertion = "await assertClientState();"
+    elif aspect == "location":
+        assertion = f"if (!({expressions[aspect]})) fail('csrf-contract');"
+    else:
+        assertion = (
+            "const cookies = await page.context().cookies(); "
+            f"if (!({expressions[aspect]})) fail('csrf-contract');"
+        )
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_client_state_source()}
+  {assertion}
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_observe_javascript(*, origin: str, check: str) -> str:
+    """Install exact POST/response accounting after cookie validation."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  if (page.__acceptanceSubmit) fail('submit-state');
+  await installSubmitObservers();
+  if (!page.__acceptanceSubmit) fail('submit-state');
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_fill_javascript(*, origin: str, check: str) -> str:
+    """Fill the real form and focus its submit button without mouse input."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  if (!page.__acceptanceSubmit || page.url() !== {json.dumps(origin + '/')}) fail('submit-state');
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
+  await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
+  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  const prepared = await page.evaluate((values) => {{
+    const departure = document.querySelector('[name="departure_date"]');
+    const returning = document.querySelector('[name="return_date"]');
+    return departure?.value === values.departure && returning?.value === values.returning && document.activeElement?.matches('button[type="submit"]');
+  }}, {{ departure: {json.dumps(SYNTHETIC_DEPARTURE)}, returning: {json.dumps(SYNTHETIC_VALID_RETURN)} }});
+  if (!prepared) fail('csrf-form-preparation');
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_press_javascript(*, origin: str, check: str) -> str:
+    """Submit the prepared form by keyboard and await the exact PRG target."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  if (!page.__acceptanceSubmit) fail('submit-state');
+  await page.keyboard.press('Enter');
+  await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }});
+  return {_marker(check)};
+}}"""
+
+
+def csrf_keyboard_finish_javascript(*, origin: str, check: str) -> str:
+    """Verify the exact POST/response counts and remove browser state."""
+
+    validate_base_url(origin, field="csrf-regression")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  await finishSubmit(303);
+  if (page.url() !== {json.dumps(origin + '/results/')}) fail('csrf-submit-location');
+  return {_marker(check)};
+}}"""
+
+
+def loading_finish_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  const state = page.__acceptanceSubmit;
+  if (!state || state.statuses.length !== 0 || page.url() !== {json.dumps(origin + '/')} || !await page.locator('[data-trip-form][aria-busy="true"]').count()) fail('loading-not-intermediate');
+  await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }});
+  await finishSubmit(303);
+  if (page.url() !== {json.dumps(origin + '/results/')}) fail('loading-prg');
+  return {_marker(check)};
+}}"""
+
+
+def validation_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await ({_common_javascript(origin=origin, path='/', check=check)})(page);
+  {_submission_support(origin)}
+  await prepareSubmit();
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
+  await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_INVALID_RETURN)});
+  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.keyboard.press('Enter');
+  const summary = page.getByRole('alert'); await summary.waitFor({{ state: 'visible' }});
+  await page.waitForFunction(() => document.activeElement?.matches('[data-error-summary]'));
+  await finishSubmit(200, false);
+  if (await summary.count() !== 1 || await page.getByRole('heading', {{ level: 2, name: '입력 내용을 확인해 주세요', exact: true }}).count() !== 1) fail('validation-semantics');
+  const field = page.getByLabel('귀국일', {{ exact: true }});
+  if (await field.getAttribute('aria-invalid') !== 'true' || !(await field.getAttribute('aria-describedby') || '').includes('id_return_date_error')) fail('validation-description');
+  if (page.url() !== {json.dumps(origin + '/')}) fail('validation-location');
+  {_dom_privacy_source(origin=origin, path='/')}
+  {_static_asset_integrity_source(origin)}
+  {_dynamic_layout_source()}
+  return {_marker(check)};
+}}"""
+
+
+def correction_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  if (!await page.evaluate(() => document.activeElement?.matches('[data-error-summary]'))) fail('error-focus-start');
+  await page.keyboard.press('Tab'); await page.keyboard.press('Enter');
+  if (!await page.evaluate(() => document.activeElement?.id === 'id_return_date')) fail('error-link-target');
+  await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
+  await page.keyboard.press('Tab');
+  if (!await page.evaluate(() => document.activeElement?.matches('[data-submit-button]'))) fail('correction-order');
+  await prepareSubmit();
+  await page.keyboard.press('Enter');
+  await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }});
+  await finishSubmit(303);
+  if (page.url() !== {json.dumps(origin + '/results/')}) fail('fixed-prg');
+  if (await page.evaluate((values) => values.some((value) => document.body.textContent.includes(value)), [{json.dumps(SYNTHETIC_DEPARTURE)}, {json.dumps(SYNTHETIC_VALID_RETURN)}])) fail('input-reflection');
+  return {_marker(check)};
+}}"""
+
+
+def results_javascript(*, origin: str, state: str, check: str) -> str:
+    if state not in STATE_NAMES:
+        raise AcceptanceFailure("invalid-scenario-state")
+    entry_state, warning_state = STATE_PAIRS[state]
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await ({_common_javascript(origin=origin, path='/results/', check=check)})(page);
+  if (await page.getByRole('heading', {{ level: 1, name: '일본 게시 정보', exact: true }}).count() !== 1) fail('results-heading');
+  const pageContract = await page.evaluate((expectedOrigin) => {{
+    const identity = (node) => `${{node.tagName}}:${{node.id}}:${{node.className}}`;
+    const text = (node) => node.textContent.replace(/\\s+/g, ' ').trim();
+    const headChildren = [...document.head.children];
+    if (headChildren.map((node) => node.tagName).join('|') !== 'META|META|META|TITLE|LINK') return 'head';
+    if (headChildren[0].getAttributeNames().join(',') !== 'charset' || headChildren[0].getAttribute('charset').toLowerCase() !== 'utf-8') return 'charset';
+    if (headChildren[1].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[1].getAttribute('name') !== 'viewport' || headChildren[1].getAttribute('content') !== 'width=device-width, initial-scale=1') return 'viewport-meta';
+    if (headChildren[2].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[2].getAttribute('name') !== 'theme-color' || headChildren[2].getAttribute('content') !== '#123f46') return 'theme-meta';
+    if (headChildren[3].getAttributeNames().length || text(headChildren[3]) !== '여행준비 — 일본 게시 정보') return 'title';
+    if (headChildren[4].getAttributeNames().sort().join(',') !== 'href,rel' || headChildren[4].getAttribute('rel') !== 'stylesheet' || headChildren[4].href !== expectedOrigin + '/static/public_web/site.css') return 'style-link';
+    if ([...document.body.children].map(identity).join('|') !== 'A::skip-link|HEADER::site-header|MAIN:main-content:page-shell page-main|FOOTER::site-footer|SCRIPT::') return 'body';
+    if (text(document.querySelector('.skip-link')) !== '본문으로 건너뛰기') return 'skip-link';
+    const main = document.querySelector('main'); if ([...main.children].map(identity).join('|') !== 'DIV::page-heading|SECTION::publication-grid') return 'main';
+    const headingChildren = [...main.querySelector('.page-heading').children];
+    if (headingChildren.map(identity).join('|') !== 'P::eyebrow|H1::|P::page-lead' || text(headingChildren[0]) !== '두 개의 독립 publication' || text(headingChildren[1]) !== '일본 게시 정보' || text(headingChildren[2]) !== '고정된 일본 publication만 표시합니다. 여행 목적과 날짜에 대한 적용 여부는 계산하지 않으므로 공식 기관 확인이 필요합니다.') return 'heading';
+    if ([...main.querySelector('.publication-grid').children].map(identity).join('|') !== 'ARTICLE:entry-card:publication-card|ARTICLE:warning-card:publication-card') return 'grid';
+    const header = document.querySelector('header'); const headerShell = document.querySelector('header > .page-shell'); const navigation = document.querySelector('header > .page-shell > nav.site-nav'); const navigationLinks = navigation ? [...navigation.children] : [];
+    if (header.children.length !== 1 || headerShell.children.length !== 1 || navigationLinks.length !== 2 || navigationLinks.some((item) => item.tagName !== 'A') || navigationLinks[0].href !== expectedOrigin + '/' || navigationLinks[1].href !== expectedOrigin + '/results/' || text(navigationLinks[0]) !== '다시 입력' || text(navigationLinks[1]) !== '게시 정보') return 'navigation';
+    const footer = document.querySelector('footer'); const footerShell = document.querySelector('footer > .page-shell'); const footerChildren = document.querySelectorAll('footer > .page-shell > p'); if (footer.children.length !== 1 || footerChildren.length !== 1 || footerShell.children.length !== 1 || text(footerChildren[0]) !== '두 카드는 서로 독립된 검수·게시 경계를 사용합니다.') return 'footer';
+    const scripts = [...document.scripts]; if (scripts.length !== 1 || scripts[0].src !== expectedOrigin + '/static/public_web/site.js' || scripts[0].textContent.trim()) return 'script';
+    const styles = [...document.querySelectorAll('link[rel="stylesheet"]')]; if (styles.length !== 1 || styles[0].href !== expectedOrigin + '/static/public_web/site.css') return 'style';
+    return null;
+  }}, {json.dumps(origin)});
+  if (pageContract) fail(`results-dom-${{pageContract}}`);
+  const staticAssets = {json.dumps([
+      {"path": "/static/public_web/site.css", "sha256": SITE_CSS_SHA256, "bytes": SITE_CSS_BYTES, "types": ["text/css"]},
+      {"path": "/static/public_web/site.js", "sha256": SITE_JS_SHA256, "bytes": SITE_JS_BYTES, "types": ["text/javascript", "application/javascript"]},
+  ], separators=(",", ":"))};
+  for (const asset of staticAssets) {{
+    const assetUrl = {json.dumps(origin)} + asset.path;
+    const awaited = page.waitForResponse((response) => response.url() === assetUrl && response.request().method() === 'GET');
+    const observed = await page.evaluate(async (contract) => {{
+      const response = await fetch(contract.url, {{ credentials: 'omit', cache: 'no-store', redirect: 'error', referrerPolicy: 'no-referrer' }});
+      const bytes = new Uint8Array(await response.arrayBuffer());
+      const digest = [...new Uint8Array(await crypto.subtle.digest('SHA-256', bytes))].map((value) => value.toString(16).padStart(2, '0')).join('');
+      return {{ status: response.status, contentType: response.headers.get('content-type') || '', byteLength: bytes.byteLength, digest }};
+    }}, {{ url: assetUrl }});
+    const response = await awaited; const details = await response.securityDetails(); const responseHeaders = await response.allHeaders(); const requestHeaders = await response.request().allHeaders();
+    if (!details || !['TLS 1.2', 'TLS 1.3'].includes(details.protocol) || 'set-cookie' in responseHeaders || 'cookie' in requestHeaders || response.request().redirectedFrom()) fail('static-transport');
+    if (observed.status !== 200 || observed.byteLength !== asset.bytes || observed.digest !== asset.sha256 || !asset.types.includes(observed.contentType.split(';')[0].trim().toLowerCase())) fail('static-digest');
+  }}
+  const expected = {{ entry: {json.dumps(entry_state)}, warning: {json.dumps(warning_state)} }};
+  const statusLabels = {{ ready: '게시된 source 사실', stale: '재확인 필요', empty: '게시 전', unavailable: '정보 확인 필요', 'server-error': '일시적 오류' }};
+  const unsafeSourceValue = (value, maximum) => {{
+    if (typeof value !== 'string' || !value.trim() || value.length > maximum) return true;
+    if ([...value].some((character) => {{ const code = character.charCodeAt(0); return code < 32 && ![9, 10, 13].includes(code); }})) return true;
+    return /[<>{{}}]/.test(value)
+      || /"(?:response|header|body|items?|resultCode|serviceKey)"\\s*:/i.test(value)
+      || /(?:%[0-9a-f]{{2}}){{4,}}/i.test(value)
+      || /[a-z0-9+/_=-]{{32,}}/i.test(value);
+  }};
+  const parseDateOnly = (value) => {{
+    const match = typeof value === 'string' ? value.match(/^(\\d{{4}})-(\\d{{2}})-(\\d{{2}})$/) : null;
+    if (!match) return NaN; const parts = match.slice(1).map(Number); const milliseconds = Date.UTC(parts[0], parts[1] - 1, parts[2]); const date = new Date(milliseconds);
+    return date.getUTCFullYear() === parts[0] && date.getUTCMonth() === parts[1] - 1 && date.getUTCDate() === parts[2] ? milliseconds : NaN;
+  }};
+  const parseUtcMinute = (value) => {{
+    const match = typeof value === 'string' ? value.match(/^(\\d{{4}})-(\\d{{2}})-(\\d{{2}}) (\\d{{2}}):(\\d{{2}}) UTC$/) : null;
+    if (!match) return NaN; const parts = match.slice(1).map(Number); const milliseconds = Date.UTC(parts[0], parts[1] - 1, parts[2], parts[3], parts[4]); const date = new Date(milliseconds);
+    return date.getUTCFullYear() === parts[0] && date.getUTCMonth() === parts[1] - 1 && date.getUTCDate() === parts[2] && date.getUTCHours() === parts[3] && date.getUTCMinutes() === parts[4] ? milliseconds : NaN;
+  }};
+  const contracts = {{
+    entry: {{ id: 'entry-card', heading: '입국요건 사실', link: '외교부 입국요건 source 열기', labels: ['국가','일반여권 source 표기','source 근거 문구','snapshot date','마지막 성공 확인시각','publication revision','게시시각','source revision','출처'], owner: '대한민국 외교부 정보화담당관실 · 외교부|공공데이터포털', source: {json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {ENTRY_FRESHNESS_MINUTES}, note: '확인 필요: 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.' }},
+    warning: {{ id: 'warning-card', heading: '여행경보', link: '외교부 여행경보 source 열기', labels: ['국가','source 경보 단계 코드','source 범위 유형','source 범위','source 작성일','마지막 성공 확인시각','publication revision','게시시각','source revision','출처'], owner: '대한민국 외교부 · 외교부|공공데이터포털', source: {json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {WARNING_FRESHNESS_MINUTES}, note: '확인 필요: 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.' }}
+  }};
+  if (await page.locator('pre, code, textarea, template, iframe, object, embed, script:not([src]), [data-raw-body], [data-secret]').count()) fail('raw-secret-node');
+  const hiddenLeak = await page.evaluate((approved) => {{
+    const allowedNames = new Set(['aria-current','aria-describedby','aria-hidden','aria-label','aria-labelledby','charset','class','content','data-state','defer','href','id','lang','name','rel','role','src','style','tabindex']);
+    const approvedHrefs = new Set(approved.href); const approvedSources = new Set(approved.src);
+    const forbidden = /(?:MOFA_TRAVEL_ALARM_SERVICE_KEY|serviceKey|api[_-]?key|raw[_-]?body|secret|authorization:)/i;
+    const opaque = /[a-z0-9+/_=-]{{24,}}/i; const encoded = /(?:%[0-9a-f]{{2}}){{4,}}/i;
+    const attributeLeak = [...document.querySelectorAll('*')].some((node) => [...node.attributes].some((attribute) => {{
+      if (!allowedNames.has(attribute.name)) return true;
+      if (attribute.name === 'href') return !approvedHrefs.has(attribute.value);
+      if (attribute.name === 'src') return !approvedSources.has(attribute.value);
+      if (attribute.name === 'style') return node !== document.documentElement || attribute.value !== 'font-size: 200%;';
+      return forbidden.test(`${{attribute.name}}=${{attribute.value}}`) || /[<>{{}}]/.test(attribute.value) || opaque.test(attribute.value) || encoded.test(attribute.value);
+    }}));
+    const commentWalker = document.createTreeWalker(document, NodeFilter.SHOW_COMMENT); let commentLeak = false;
+    while (commentWalker.nextNode()) if ((commentWalker.currentNode.nodeValue || '').trim()) commentLeak = true;
+    return attributeLeak || commentLeak;
+  }}, {{
+    href: ['#main-content','/','/results/','/static/public_web/site.css',{json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)},{json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}],
+    src: ['/static/public_web/site.js']
+  }});
+  if (hiddenLeak) fail('raw-secret-hidden');
+  let publicationCount = 0;
+  for (const module of ['entry', 'warning']) {{
+    const contract = contracts[module]; const card = page.locator(`#${{contract.id}}`);
+    if (await card.count() !== 1 || await card.getAttribute('data-state') !== expected[module]) fail(`${{module}}-state`);
+    if (await card.getAttribute('tabindex') !== '0' || await card.getAttribute('aria-labelledby') !== `${{module}}-heading` || await card.getAttribute('aria-describedby') !== `${{module}}-status ${{module}}-message`) fail(`${{module}}-aria-contract`);
+    if (await card.getByRole('heading', {{ level: 2, name: contract.heading, exact: true }}).count() !== 1) fail(`${{module}}-heading`);
+    if (await card.getByRole('status').count() !== 1 || (await card.getByRole('status').innerText()).trim() !== `상태: ${{statusLabels[expected[module]]}}` || await card.locator('.status-symbol[aria-hidden="true"]').count() !== 1) fail(`${{module}}-status`);
+    const stateValue = expected[module]; const published = stateValue === 'ready' || stateValue === 'stale';
+    const expectedMessage = stateValue === 'ready'
+      ? (module === 'entry' ? '공식 source의 검수·게시 사실입니다.' : '입국요건과 독립된 공식 source의 검수·게시 사실입니다.')
+      : stateValue === 'stale' ? '마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요.'
+      : stateValue === 'empty' ? '아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.'
+      : stateValue === 'unavailable' ? '게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.'
+      : '이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.';
+    if ((await card.locator(`#${{module}}-message`).innerText()).trim() !== expectedMessage) fail(`${{module}}-message`);
+    const factLists = card.locator('dl.fact-list'); const links = card.getByRole('link', {{ name: contract.link, exact: true }}); const notes = card.locator('.verification-note');
+    const directChildren = await card.evaluate((node) => [...node.children].map((child) => `${{child.tagName}}:${{child.id}}:${{child.className}}`));
+    const expectedChildren = [`H2:${{module}}-heading:`,`P:${{module}}-status:status-line`,`P:${{module}}-message:`];
+    if (published) expectedChildren.push('DL::fact-list','P::verification-note');
+    if (JSON.stringify(directChildren) !== JSON.stringify(expectedChildren)) fail(`${{module}}-dom-contract`);
+    const shellExact = await card.evaluate((node) => {{
+      const names = (item) => [...item.attributes].map((attribute) => attribute.name).sort().join(',');
+      const heading = node.querySelector('h2'); const status = node.querySelector('.status-line'); const symbol = node.querySelector('.status-symbol'); const strong = status.querySelector('strong'); const message = node.querySelector(`#${{node.id.replace('-card', '-message')}}`);
+      return names(node) === 'aria-describedby,aria-labelledby,class,data-state,id,tabindex' && names(heading) === 'id' && names(status) === 'class,id,role' && names(symbol) === 'aria-hidden,class' && names(strong) === '' && names(message) === 'id' && !heading.children.length && !symbol.children.length && !strong.children.length && !message.children.length;
+    }});
+    if (!shellExact) fail(`${{module}}-shell-contract`);
+    if (!published) {{
+      if (await factLists.count() || await links.count() || await notes.count()) fail(`${{module}}-unpublished-leak`);
+      continue;
+    }}
+    publicationCount += 1;
+    if (await factLists.count() !== 1 || await links.count() !== 1 || await notes.count() !== 1 || (await notes.innerText()).trim() !== contract.note) fail(`${{module}}-publication-contract`);
+    const metadata = await card.evaluate((node) => {{
+      const dts = [...node.querySelectorAll('.fact-list dt')];
+      const values = Object.fromEntries(dts.map((dt) => [dt.textContent.trim(), dt.nextElementSibling?.textContent.trim() || '']));
+      const bodySize = Number.parseFloat(getComputedStyle(document.body).fontSize);
+      const h2Size = Number.parseFloat(getComputedStyle(node.querySelector('h2')).fontSize);
+      const metadataReadable = [...node.querySelectorAll('.fact-list dt, .fact-list dd')].every((item) => Number.parseFloat(getComputedStyle(item).fontSize) >= 14 && Number.parseFloat(getComputedStyle(item).lineHeight) / Number.parseFloat(getComputedStyle(item).fontSize) >= 1.35);
+      const children = [...node.querySelector('dl.fact-list').children];
+      const exactStructure = children.length === dts.length * 2 && children.every((item, index) => item.tagName === (index % 2 ? 'DD' : 'DT'));
+      const descriptions = children.filter((item) => item.tagName === 'DD'); const sourceDescription = descriptions.at(-1);
+      const attributeNames = (item) => [...item.attributes].map((attribute) => attribute.name).sort().join(',');
+      const exactAttributes = attributeNames(node) === 'aria-describedby,aria-labelledby,class,data-state,id,tabindex'
+        && attributeNames(node.querySelector('h2')) === 'id'
+        && attributeNames(node.querySelector('.status-line')) === 'class,id,role'
+        && attributeNames(node.querySelector('.status-symbol')) === 'aria-hidden,class'
+        && attributeNames(node.querySelector('.status-line strong')) === ''
+        && attributeNames(node.querySelector(`#${{node.id.replace('-card', '-message')}}`)) === 'id'
+        && attributeNames(node.querySelector('dl.fact-list')) === 'class'
+        && dts.every((item) => attributeNames(item) === '')
+        && descriptions.every((item) => attributeNames(item) === '')
+        && attributeNames(sourceDescription.firstElementChild) === 'aria-label,class,href,rel'
+        && attributeNames(node.querySelector('.verification-note')) === 'class'
+        && attributeNames(node.querySelector('.verification-note strong')) === '';
+      const exactDescendants = node.querySelector('h2').children.length === 0
+        && [...node.querySelector('.status-line').children].map((item) => item.tagName).join(',') === 'SPAN,STRONG'
+        && [...node.querySelector('.status-line').children].every((item) => item.children.length === 0)
+        && node.querySelector(`#${{node.id.replace('-card', '-message')}}`).children.length === 0
+        && dts.every((item) => item.children.length === 0)
+        && descriptions.slice(0, -1).every((item) => item.children.length === 0)
+        && sourceDescription?.children.length === 1 && sourceDescription.firstElementChild?.tagName === 'A'
+        && sourceDescription.firstElementChild?.children.length === 0
+        && [...node.querySelector('.verification-note').children].map((item) => item.tagName).join(',') === 'STRONG'
+        && node.querySelector('.verification-note strong').children.length === 0;
+      return {{ labels: dts.map((dt) => dt.textContent.trim()), values, h2Size, bodySize, metadataReadable, exactStructure, exactDescendants, exactAttributes }};
+    }});
+    if (JSON.stringify(metadata.labels) !== JSON.stringify(contract.labels) || !metadata.exactStructure || !metadata.exactDescendants || !metadata.exactAttributes || !metadata.metadataReadable || !(metadata.h2Size > metadata.bodySize)) fail(`${{module}}-metadata`);
+    if (contract.labels.some((label) => {{ const value = metadata.values[label]; const trimmed = value?.trimStart() || ''; return !value || trimmed.startsWith('{{') || trimmed.startsWith('['); }})) fail(`${{module}}-empty-value`);
+    const nowMillis = Date.now(); const observedMillis = parseUtcMinute(metadata.values['마지막 성공 확인시각']);
+    if (metadata.values['국가'] !== '일본' || !Number.isFinite(observedMillis)) fail(`${{module}}-freshness`);
+    const ageMinutes = (nowMillis - observedMillis) / 60000;
+    if (!Number.isFinite(ageMinutes) || ageMinutes < -5 || (stateValue === 'ready' && ageMinutes > contract.freshnessMinutes + 2) || (stateValue === 'stale' && ageMinutes <= contract.freshnessMinutes)) fail(`${{module}}-freshness-age`);
+    const publishedMillis = parseUtcMinute(metadata.values['게시시각']);
+    if (!/^generation [1-9]\\d*$/.test(metadata.values['publication revision']) || !Number.isFinite(publishedMillis) || publishedMillis > nowMillis + 300000 || publishedMillis + 60000 < observedMillis || metadata.values['source revision'] !== 'rights-v1') fail(`${{module}}-revision-metadata`);
+    if (module === 'entry') {{ const snapshotMillis = parseDateOnly(metadata.values['snapshot date']); if (!Number.isFinite(snapshotMillis) || snapshotMillis > nowMillis + 86400000 || !/^[1-9]\\d{{0,2}}일$/.test(metadata.values['일반여권 source 표기']) || unsafeSourceValue(metadata.values['source 근거 문구'], 1000)) fail('entry-facts'); }}
+    if (module === 'warning') {{ const written = metadata.values['source 작성일']; const writtenMillis = written === 'source가 제공하지 않음' ? null : parseDateOnly(written); if (unsafeSourceValue(metadata.values['source 경보 단계 코드'], 32) || unsafeSourceValue(metadata.values['source 범위 유형'], 100) || unsafeSourceValue(metadata.values['source 범위'], 1000) || (writtenMillis !== null && (!Number.isFinite(writtenMillis) || writtenMillis > nowMillis + 86400000))) fail('warning-facts'); }}
+    if (metadata.values['출처'].replace(/\\s+/g, ' ') !== `${{contract.owner}} 공식 source`) fail(`${{module}}-source-name`);
+    const link = links.first(); const href = await link.getAttribute('href'); const rel = (await link.getAttribute('rel') || '').split(/\\s+/);
+    if (href !== contract.source || !rel.includes('noopener') || !rel.includes('noreferrer')) fail(`${{module}}-source-link`);
+  }}
+  const noteCount = await page.locator('.verification-note').count();
+  if (noteCount !== publicationCount || (publicationCount > 0) !== (noteCount > 0)) fail('verification-count');
+  if ({json.dumps(state)} === 'long-korean' && await page.locator('.fact-list dd').evaluateAll((nodes) => Math.max(...nodes.map((node) => node.textContent.trim().length), 0)) < 40) fail('long-content');
+  const bodyText = await page.locator('body').innerText(); if (unsafeSourceValue(bodyText, 100000)) fail('forbidden-content');
+  const foldedBody = bodyText.toLowerCase().replace(/\\s+/g, ' '); const compactBody = foldedBody.replace(/\\s+/g, '');
+  for (const marker of ['allow' + 'ed','deni' + 'ed','mofa_' + 'travel_alarm_service_key','service' + 'key','api_key','api key','secret key','authorization:']) if (foldedBody.includes(marker)) fail('forbidden-content');
+  for (const marker of ['입국가능','법적판단']) if (compactBody.includes(marker)) fail('forbidden-content');
+  return {_marker(check)};
+}}"""
+
+
+def results_keyboard_javascript(*, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await page.evaluate(() => document.activeElement && document.activeElement.blur());
+  for (const selector of ['.skip-link','.site-nav a:nth-of-type(1)','.site-nav a:nth-of-type(2)','#entry-card','#entry-card .source-link','#warning-card','#warning-card .source-link']) {{
+    await page.keyboard.press('Tab');
+    const visible = await page.evaluate((candidate) => {{ const node = document.activeElement; if (!node?.matches(candidate)) return false; const style = getComputedStyle(node); return style.outlineStyle !== 'none' && Number.parseFloat(style.outlineWidth) >= 2; }}, selector);
+    if (!visible) fail('result-focus');
+  }}
+  return {_marker(check)};
+}}"""
+
+
+def forced_colors_javascript(*, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await page.emulateMedia({{ forcedColors: 'active', reducedMotion: 'reduce' }});
+  const okay = await page.locator('.publication-card').evaluateAll((cards) => cards.length === 2 && cards.every((card) => {{ const style = getComputedStyle(card); return style.borderStyle !== 'none' && Number.parseFloat(style.borderWidth) >= 2 && card.querySelector('[role="status"]').textContent.includes('상태:'); }}));
+  if (!okay) fail('forced-colors-state'); return {_marker(check)};
+}}"""
+
+
+def reset_media_javascript(*, check: str) -> str:
+    return f"""async (page) => {{ await page.emulateMedia({{ forcedColors: 'none', reducedMotion: 'reduce' }}); return {_marker(check)}; }}"""
+
+
+def text_scale_javascript(*, origin: str, path: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  await page.emulateMedia({{ forcedColors: 'none', reducedMotion: 'reduce' }}); await page.evaluate(() => document.documentElement.style.fontSize = '200%');
+  await ({_common_javascript(origin=origin, path=path, check=check)})(page);
+  const scaled = await page.evaluate(() => ({{ root: Number.parseFloat(getComputedStyle(document.documentElement).fontSize), body: Number.parseFloat(getComputedStyle(document.body).fontSize) }}));
+  if (scaled.root < 31 || scaled.body < 31) fail('text-scale');
+  return {_marker(check)};
+}}"""
+
+
+def scaled_form_submit_javascript(*, origin: str, check: str) -> str:
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)} await prepareSubmit();
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)}); await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
+  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus(); await page.keyboard.press('Enter');
+  await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }}); await finishSubmit(303);
+  await page.evaluate(() => document.documentElement.style.fontSize = '200%');
+  if (await page.evaluate(() => document.documentElement.scrollWidth > window.innerWidth + 1)) fail('scaled-overflow');
+  return {_marker(check)};
+}}"""
+
+
+def final_guard_javascript(*, check: str, expected_posts: int) -> str:
+    if type(expected_posts) is not int or not 0 <= expected_posts <= 100:
+        raise AcceptanceFailure("invalid-expected-posts")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }}; {_client_state_source()}
+  if (!page.context().__acceptanceGuard || page.context().__acceptanceGuard.external !== 0 || page.context().__acceptanceGuard.unexpected !== 0 || page.context().__acceptanceGuard.posts !== {expected_posts}) fail('request-attempt');
+  await page.context().clearCookies(); if ((await page.context().cookies()).length !== 0) fail('final-cookie-state'); await assertClientState();
+  page.context().off('request', page.context().__acceptanceRequestListener); delete page.context().__acceptanceRequestListener;
+  return {_marker(check)};
+}}"""
+
+
+def png_metadata(
+    path: Path, *, expected_width: int, minimum_height: int,
+    expected_mode: int = 0o600,
+) -> tuple[str, int, int]:
+    try:
+        metadata = path.lstat()
+        if (
+            not stat.S_ISREG(metadata.st_mode)
+            or metadata.st_uid != os.getuid()
+            or stat.S_IMODE(metadata.st_mode) != expected_mode
+            or not 45 <= metadata.st_size <= MAX_SCREENSHOT_BYTES
+        ):
+            raise AcceptanceFailure("screenshot-file-contract")
+        payload = path.read_bytes()
+        if payload[:8] != PNG_SIGNATURE:
+            raise AcceptanceFailure("screenshot-invalid-png")
+        offset = len(PNG_SIGNATURE)
+        chunks: list[tuple[bytes, bytes]] = []
+        while offset < len(payload):
+            if len(payload) - offset < 12:
+                raise AcceptanceFailure("screenshot-invalid-png")
+            length = struct.unpack(">I", payload[offset : offset + 4])[0]
+            kind = payload[offset + 4 : offset + 8]
+            end = offset + 12 + length
+            if end > len(payload):
+                raise AcceptanceFailure("screenshot-invalid-png")
+            data = payload[offset + 8 : offset + 8 + length]
+            crc = struct.unpack(">I", payload[end - 4 : end])[0]
+            if zlib.crc32(kind + data) & 0xFFFFFFFF != crc:
+                raise AcceptanceFailure("screenshot-invalid-png")
+            chunks.append((kind, data))
+            offset = end
+            if kind == b"IEND":
+                break
+        if (
+            offset != len(payload)
+            or not chunks
+            or chunks[0][0] != b"IHDR"
+            or len(chunks[0][1]) != 13
+            or chunks[-1] != (b"IEND", b"")
+            or not any(kind == b"IDAT" for kind, _ in chunks)
+        ):
+            raise AcceptanceFailure("screenshot-invalid-png")
+        width, height = struct.unpack(">II", chunks[0][1][:8])
+        if width != expected_width or height < minimum_height or height > MAX_SCREENSHOT_HEIGHT:
+            raise AcceptanceFailure("screenshot-dimensions")
+        return hashlib.sha256(payload).hexdigest(), width, height
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("screenshot-read-failed") from exc
+
+
+def _screenshot(
+    session: CliSession, *, run_root: Path, viewport: str,
+    scenario: str, width: int, height: int,
+) -> dict[str, object]:
+    if scenario not in SCREENSHOT_NAMES:
+        raise AcceptanceFailure("invalid-screenshot-name")
+    directory = run_root / viewport
+    _ensure_private_directory(directory, exist_ok=True)
+    relative = Path(viewport) / f"{scenario}.png"
+    session.screenshot(relative.as_posix(), check=f"screenshot-{scenario}")
+    digest, actual_width, actual_height = png_metadata(
+        run_root / relative, expected_width=width, minimum_height=height
+    )
+    return {
+        "file": relative.as_posix(), "height": actual_height,
+        "scenario": scenario, "sha256": digest, "viewport": viewport,
+        "width": actual_width,
+    }
+
+
+def _write_config(path: Path, config: object) -> None:
+    atomic_json(path, config)
+
+
+def _safe_remove_tree(root: Path) -> None:
+    if not root.exists():
+        return
+    try:
+        root_metadata = root.lstat()
+    except OSError as exc:
+        raise AcceptanceFailure("transient-cleanup-root") from exc
+    if (
+        not stat.S_ISDIR(root_metadata.st_mode)
+        or stat.S_ISLNK(root_metadata.st_mode)
+        or root_metadata.st_uid != os.getuid()
+        or stat.S_IMODE(root_metadata.st_mode) not in {0o500, 0o700}
+    ):
+        raise AcceptanceFailure("transient-cleanup-root")
+    root_resolved = root.resolve(strict=True)
+    paths = sorted(root.rglob("*"), key=lambda item: len(item.parts), reverse=True)
+    for path in paths:
+        metadata = path.lstat()
+        if metadata.st_uid != os.getuid():
+            raise AcceptanceFailure("transient-cleanup-owner")
+        if stat.S_ISLNK(metadata.st_mode):
+            resolved = path.resolve(strict=False)
+            if not resolved.is_relative_to(root_resolved):
+                raise AcceptanceFailure("transient-cleanup-symlink")
+        elif not (
+            stat.S_ISDIR(metadata.st_mode)
+            or stat.S_ISREG(metadata.st_mode)
+            or stat.S_ISSOCK(metadata.st_mode)
+        ):
+            raise AcceptanceFailure("transient-cleanup-special")
+    try:
+        os.chmod(root, 0o700)
+        for path in reversed(paths):
+            if stat.S_ISDIR(path.lstat().st_mode):
+                os.chmod(path, 0o700)
+    except OSError as exc:
+        raise AcceptanceFailure("transient-cleanup-unseal") from exc
+    for path in paths:
+        metadata = path.lstat()
+        if stat.S_ISLNK(metadata.st_mode):
+            path.unlink()
+        elif stat.S_ISDIR(metadata.st_mode):
+            path.rmdir()
+        elif stat.S_ISREG(metadata.st_mode) or stat.S_ISSOCK(metadata.st_mode):
+            path.unlink()
+    root.rmdir()
+
+
+def _assert_persistent_artifacts(
+    run_root: Path, report_path: Path, receipt: Mapping[str, object], *,
+    sealed: bool,
+) -> None:
+    expected_dirs = {run_root / f"{width}x{height}" for width, height in VIEWPORTS}
+    expected_files = {report_path} | {
+        directory / f"{scenario}.png"
+        for directory in expected_dirs
+        for scenario in SCREENSHOT_NAMES
+    }
+    actual_dirs: set[Path] = set()
+    actual_files: set[Path] = set()
+    directory_mode = 0o500 if sealed else 0o700
+    file_mode = 0o400 if sealed else 0o600
+    root_metadata = run_root.lstat()
+    if (
+        root_metadata.st_uid != os.getuid()
+        or not stat.S_ISDIR(root_metadata.st_mode)
+        or stat.S_IMODE(root_metadata.st_mode) != directory_mode
+    ):
+        raise AcceptanceFailure("artifact-root-mode")
+    for path in run_root.rglob("*"):
+        metadata = path.lstat()
+        if metadata.st_uid != os.getuid() or stat.S_ISLNK(metadata.st_mode):
+            raise AcceptanceFailure("unexpected-artifact")
+        if stat.S_ISDIR(metadata.st_mode):
+            if stat.S_IMODE(metadata.st_mode) != directory_mode:
+                raise AcceptanceFailure("artifact-directory-mode")
+            actual_dirs.add(path)
+        elif stat.S_ISREG(metadata.st_mode):
+            if stat.S_IMODE(metadata.st_mode) != file_mode:
+                raise AcceptanceFailure("artifact-file-mode")
+            actual_files.add(path)
+        else:
+            raise AcceptanceFailure("unexpected-artifact")
+    if actual_dirs != expected_dirs or actual_files != expected_files:
+        raise AcceptanceFailure("unexpected-artifact")
+    try:
+        if report_path.stat().st_size > MAX_CLI_OUTPUT_BYTES:
+            raise AcceptanceFailure("evidence-report-size")
+        persisted = json.loads(report_path.read_text(encoding="utf-8"))
+    except AcceptanceFailure:
+        raise
+    except (OSError, UnicodeError, json.JSONDecodeError) as exc:
+        raise AcceptanceFailure("evidence-report-invalid") from exc
+    if persisted != receipt:
+        raise AcceptanceFailure("evidence-report-mismatch")
+    screenshots = receipt.get("screenshots")
+    if not isinstance(screenshots, list) or len(screenshots) != (
+        len(VIEWPORTS) * len(SCREENSHOT_NAMES)
+    ):
+        raise AcceptanceFailure("evidence-screenshot-count")
+    seen: set[str] = set()
+    viewport_dimensions = {f"{w}x{h}": (w, h) for w, h in VIEWPORTS}
+    for item in screenshots:
+        if not isinstance(item, dict) or set(item) != {
+            "file", "height", "scenario", "sha256", "viewport", "width"
+        }:
+            raise AcceptanceFailure("evidence-screenshot-schema")
+        relative = item.get("file")
+        viewport = item.get("viewport")
+        scenario = item.get("scenario")
+        if (
+            not isinstance(relative, str)
+            or relative in seen
+            or viewport not in viewport_dimensions
+            or scenario not in SCREENSHOT_NAMES
+            or relative != f"{viewport}/{scenario}.png"
+        ):
+            raise AcceptanceFailure("evidence-screenshot-identity")
+        seen.add(relative)
+        width, minimum_height = viewport_dimensions[viewport]
+        digest, actual_width, actual_height = png_metadata(
+            run_root / relative,
+            expected_width=width,
+            minimum_height=minimum_height,
+            expected_mode=file_mode,
+        )
+        if (
+            item.get("sha256") != digest
+            or item.get("width") != actual_width
+            or item.get("height") != actual_height
+        ):
+            raise AcceptanceFailure("evidence-screenshot-mismatch")
+
+
+def _seal_persistent_artifacts(
+    run_root: Path, report_path: Path, receipt: Mapping[str, object]
+) -> None:
+    _assert_persistent_artifacts(run_root, report_path, receipt, sealed=False)
+    try:
+        for path in run_root.rglob("*"):
+            metadata = path.lstat()
+            if stat.S_ISREG(metadata.st_mode):
+                os.chmod(path, 0o400)
+        for path in sorted(
+            (item for item in run_root.rglob("*") if item.is_dir()),
+            key=lambda item: len(item.parts),
+            reverse=True,
+        ):
+            os.chmod(path, 0o500)
+        os.chmod(run_root, 0o500)
+    except OSError as exc:
+        raise AcceptanceFailure("evidence-seal-failed") from exc
+    _assert_persistent_artifacts(run_root, report_path, receipt, sealed=True)
+
+
+def _seal_failure_report(run_root: Path, report_path: Path) -> None:
+    try:
+        items = tuple(run_root.iterdir())
+        metadata = report_path.lstat()
+        if (
+            items != (report_path,)
+            or not stat.S_ISREG(metadata.st_mode)
+            or metadata.st_uid != os.getuid()
+            or stat.S_IMODE(metadata.st_mode) != 0o600
+        ):
+            raise AcceptanceFailure("failure-report-contract")
+        os.chmod(report_path, 0o400)
+        os.chmod(run_root, 0o500)
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("failure-report-seal") from exc
+
+
+def _verify_all_tls(origins: Mapping[str, str], spki: str) -> None:
+    for origin in origins.values():
+        verify_tls_peer(origin, spki)
+
+
+def run_matrix(
+    *, run_root: Path, work_root: Path, origins: Mapping[str, str], spki: str,
+    expected_cli_version: str, release_sha: str, toolchain: Toolchain | None = None,
+) -> dict[str, object]:
+    if tuple(origins) != ORIGIN_NAMES or len(set(origins.values())) != len(ORIGIN_NAMES):
+        raise AcceptanceFailure("scenario-origins-not-distinct")
+    if expected_cli_version != REVIEWED_CLI_VERSION:
+        raise AcceptanceFailure("cli-version-not-reviewed")
+    if not SAFE_RELEASE.fullmatch(release_sha):
+        raise AcceptanceFailure("invalid-release-sha")
+    toolchain = toolchain or prepare_toolchain(work_root)
+    _verify_all_tls(origins, spki)
+    probe = CliSession(
+        session_name="travel-readiness-version", config_path=work_root / "unused.json",
+        working_directory=work_root, toolchain=toolchain,
+    )
+    version = probe.invoke("version", "--version", timeout=30)
+    try:
+        actual_version = version.decode("ascii", "strict").strip()
+    except UnicodeError as exc:
+        raise AcceptanceFailure("cli-version-invalid") from exc
+    if actual_version != REVIEWED_CLI_VERSION:
+        raise AcceptanceFailure("cli-version-mismatch")
+
+    receipt: dict[str, object] = {
+        "browser": {
+            "name": CHROME_PRODUCT_NAME,
+            "revision": CHROME_REVISION,
+            "version": CHROME_VERSION,
+            "binary_sha256": CHROME_BINARY_SHA256,
+            "bundle_sha256": toolchain.chrome_tree_sha256,
+        },
+        "checks": [],
+        "cli_version": actual_version,
+        "completed_at_utc": None,
+        "node": {"version": NODE_VERSION, "sha256": NODE_SHA256},
+        "release_sha": release_sha,
+        "schema_version": 2,
+        "screenshots": [],
+        "shells": {
+            "bash_sha256": BASH_SHA256,
+            "env_sha256": ENV_SHA256,
+            "sh_sha256": SH_SHA256,
+        },
+        "started_at_utc": utc_now(),
+        "status": "running",
+        "synthetic_inputs_only": True,
+        "viewports": [f"{w}x{h}" for w, h in VIEWPORTS],
+        "wrapper_sha256": WRAPPER_SHA256,
+        "toolchain": {
+            "adapter": "private-exact-argv-v1",
+            "cli_cache_tree_sha256": CACHED_CLI_TREE_SHA256,
+            "cli_lock_sha256": CACHED_CLI_LOCK_SHA256,
+            "execution_tree_sha256": toolchain.sealed_tree_sha256,
+            "execution_tree_entries": toolchain.sealed_entry_count,
+            "execution_tree_file_bytes": toolchain.sealed_file_bytes,
+            "playwright_version": PLAYWRIGHT_VERSION,
+            "node_library_set_sha256": NODE_LIBRARY_SET_SHA256,
+        },
+    }
+    screenshots = receipt["screenshots"]
+    checks = receipt["checks"]
+    assert isinstance(screenshots, list) and isinstance(checks, list)
+    origin_values = list(origins.values())
+    for width, height in VIEWPORTS:
+        viewport = f"{width}x{height}"
+        viewport_work = work_root / viewport
+        _ensure_private_directory(viewport_work)
+        config_path = viewport_work / "cli-config.json"
+        _write_config(
+            config_path,
+            build_cli_config(
+                width=width, height=height, origins=origin_values,
+                spki=spki, output_dir=run_root,
+                chrome_executable=toolchain.chrome_executable,
+            ),
+        )
+        session = CliSession(
+            session_name=f"acceptance-{width}-{height}", config_path=config_path,
+            working_directory=viewport_work, toolchain=toolchain,
+        )
+        try:
+            session.open("about:blank")
+            session.run_code(
+                "install-guards",
+                install_guards_javascript(origins=origin_values, check="install-guards"),
+            )
+            ready = origins["ready"]
+            session.goto(f"{ready}/", check="goto-pristine")
+            session.run_code(
+                "release-ready-form",
+                verify_release_javascript(
+                    origin=ready, release_sha=release_sha, check="release-ready-form"
+                ),
+            )
+            session.snapshot(
+                check="snapshot-pristine",
+                required_tokens=(
+                    "일본 여행 정보 확인", "목적지", "출국일", "귀국일", "게시 정보 확인"
+                ),
+            )
+            session.run_code(
+                "form-pristine",
+                form_pristine_javascript(origin=ready, check="form-pristine"),
+            )
+            checks.append({"check": "form-semantics-keyboard", "viewport": viewport})
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="form-pristine", width=width, height=height,
+                )
+            )
+
+            loading = origins["loading"]
+            session.goto(f"{loading}/", check="goto-loading")
+            session.run_code(
+                "release-loading-form",
+                verify_release_javascript(
+                    origin=loading, release_sha=release_sha, check="release-loading-form"
+                ),
+            )
+            session.run_code(
+                "form-loading-start",
+                loading_start_javascript(origin=loading, check="form-loading-start"),
+            )
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="form-loading", width=width, height=height,
+                )
+            )
+            session.run_code(
+                "form-loading-finish",
+                loading_finish_javascript(origin=loading, check="form-loading-finish"),
+            )
+            session.run_code(
+                "release-loading-result",
+                verify_release_javascript(
+                    origin=loading, release_sha=release_sha, check="release-loading-result"
+                ),
+            )
+            session.run_code(
+                "loading-ready-result",
+                results_javascript(
+                    origin=loading, state="ready", check="loading-ready-result"
+                ),
+            )
+            checks.append({"check": "loading-keyboard-single-post-303", "viewport": viewport})
+
+            session.goto(f"{ready}/", check="goto-validation")
+            session.run_code(
+                "release-validation-form",
+                verify_release_javascript(
+                    origin=ready, release_sha=release_sha, check="release-validation-form"
+                ),
+            )
+            session.run_code(
+                "form-validation",
+                validation_javascript(origin=ready, check="form-validation"),
+            )
+            checks.append({"check": "validation-single-post-focus", "viewport": viewport})
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="form-validation", width=width, height=height,
+                )
+            )
+            session.snapshot(
+                check="snapshot-validation",
+                required_tokens=(
+                    "입력 내용을 확인해 주세요",
+                    "귀국일은 출국일과 같거나 그 이후여야 합니다.",
+                ),
+            )
+            session.run_code(
+                "form-correction",
+                correction_javascript(origin=ready, check="form-correction"),
+            )
+            session.run_code(
+                "release-ready-result",
+                verify_release_javascript(
+                    origin=ready, release_sha=release_sha, check="release-ready-result"
+                ),
+            )
+            session.run_code(
+                "ready-results",
+                results_javascript(origin=ready, state="ready", check="ready-results"),
+            )
+            session.snapshot(
+                check="snapshot-ready",
+                required_tokens=(
+                    "일본 게시 정보", "입국요건 사실", "여행경보", "확인 필요", "공식 source"
+                ),
+            )
+            session.run_code(
+                "ready-keyboard", results_keyboard_javascript(check="ready-keyboard")
+            )
+            checks.append({"check": "correction-single-post-303-ready", "viewport": viewport})
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="form-correction-ready", width=width, height=height,
+                )
+            )
+            session.run_code(
+                "forced-colors", forced_colors_javascript(check="forced-colors")
+            )
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="ready-forced-colors", width=width, height=height,
+                )
+            )
+            session.run_code("reset-media", reset_media_javascript(check="reset-media"))
+
+            for state in ("empty", "unavailable", "stale", "server-error", "long-korean"):
+                origin = origins[state]
+                session.goto(f"{origin}/results/", check=f"goto-{state}")
+                session.run_code(
+                    f"release-{state}",
+                    verify_release_javascript(
+                        origin=origin, release_sha=release_sha, check=f"release-{state}"
+                    ),
+                )
+                session.run_code(
+                    state, results_javascript(origin=origin, state=state, check=state)
+                )
+                session.snapshot(
+                    check=f"snapshot-{state}",
+                    required_tokens=("일본 게시 정보", "입국요건 사실", "여행경보", "상태:"),
+                )
+                checks.append({"check": f"state-{state}-exact-pair", "viewport": viewport})
+                screenshots.append(
+                    _screenshot(
+                        session, run_root=run_root, viewport=viewport,
+                        scenario=state, width=width, height=height,
+                    )
+                )
+
+            long_origin = origins["long-korean"]
+            session.run_code(
+                "long-text-scale",
+                text_scale_javascript(
+                    origin=long_origin, path="/results/", check="long-text-scale"
+                ),
+            )
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="long-korean-200-percent", width=width, height=height,
+                )
+            )
+            checks.append({"check": "long-korean-200-percent", "viewport": viewport})
+            session.goto(f"{ready}/", check="goto-scaled-form")
+            session.run_code(
+                "release-scaled-form",
+                verify_release_javascript(
+                    origin=ready, release_sha=release_sha, check="release-scaled-form"
+                ),
+            )
+            session.run_code(
+                "form-text-scale",
+                text_scale_javascript(origin=ready, path="/", check="form-text-scale"),
+            )
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="form-200-percent", width=width, height=height,
+                )
+            )
+            session.run_code(
+                "scaled-form-submit",
+                scaled_form_submit_javascript(origin=ready, check="scaled-form-submit"),
+            )
+            session.run_code(
+                "release-scaled-result",
+                verify_release_javascript(
+                    origin=ready, release_sha=release_sha, check="release-scaled-result"
+                ),
+            )
+            session.run_code(
+                "scaled-ready-results",
+                results_javascript(
+                    origin=ready, state="ready", check="scaled-ready-results"
+                ),
+            )
+            screenshots.append(
+                _screenshot(
+                    session, run_root=run_root, viewport=viewport,
+                    scenario="ready-200-percent", width=width, height=height,
+                )
+            )
+            checks.append({"check": "form-completion-200-percent", "viewport": viewport})
+            session.run_code(
+                "final-guard",
+                final_guard_javascript(check="final-guard", expected_posts=4),
+            )
+        finally:
+            session.close()
+    _verify_all_tls(origins, spki)
+    validate_wrapper()
+    _assert_regular(NODE_SOURCE, digest=NODE_SHA256, executable=True)
+    _assert_regular(BASH_SOURCE, digest=BASH_SHA256, executable=True)
+    _assert_regular(SH_SOURCE, digest=SH_SHA256, executable=True)
+    _assert_regular(ENV_SOURCE, digest=ENV_SHA256, executable=True)
+    _assert_regular(CHROME_EXECUTABLE, digest=CHROME_BINARY_SHA256, executable=True)
+    if _tree_digest(CACHED_CLI_ROOT, include_modes=True) != (
+        CACHED_CLI_TREE_SHA256,
+        CACHED_CLI_ENTRY_COUNT,
+        CACHED_CLI_FILE_BYTES,
+    ):
+        raise AcceptanceFailure("cli-cache-changed")
+    _validate_package_lock(CACHED_CLI_ROOT)
+    if _tree_digest(CHROME_BUNDLE, include_modes=True) != (
+        CHROME_TREE_SHA256,
+        CHROME_ENTRY_COUNT,
+        CHROME_FILE_BYTES,
+    ) or toolchain.chrome_tree_sha256 != CHROME_TREE_SHA256:
+        raise AcceptanceFailure("chrome-bundle-changed")
+    _validate_node_libraries()
+    sealed_tree = _verify_sealed_toolchain_tree(toolchain.sealed_root)
+    if sealed_tree != (
+        toolchain.sealed_tree_sha256,
+        toolchain.sealed_entry_count,
+        toolchain.sealed_file_bytes,
+    ):
+        raise AcceptanceFailure("private-toolchain-changed")
+    receipt["status"] = "passed"
+    receipt["completed_at_utc"] = utc_now()
+    return receipt
+
+
+def parse_arguments(arguments: Sequence[str]) -> argparse.Namespace:
+    parser = SafeArgumentParser(
+        description="Run the isolated loopback HTTPS browser acceptance matrix."
+    )
+    for name in ORIGIN_NAMES:
+        parser.add_argument(
+            f"--{name}-base-url",
+            required=True,
+            help=(
+                "distinct slow-ready loopback HTTPS origin; its keyboard POST must remain pending through the loading screenshot"
+                if name == "loading"
+                else f"distinct loopback HTTPS origin serving the {name} scenario"
+            ),
+        )
+    parser.add_argument("--certificate-spki", required=True)
+    parser.add_argument("--expected-cli-version", required=True)
+    parser.add_argument("--release-sha", required=True)
+    parser.add_argument("--output-label", required=True)
+    return parser.parse_args(arguments)
+
+
+def _validated_origins(namespace: argparse.Namespace) -> dict[str, str]:
+    origins = {
+        name: validate_base_url(
+            getattr(namespace, f"{name.replace('-', '_')}_base_url"), field=name
+        )
+        for name in ORIGIN_NAMES
+    }
+    if len(set(origins.values())) != len(origins):
+        raise AcceptanceFailure("scenario-origins-not-distinct")
+    return origins
+
+
+def _purge_failed_artifacts(run_root: Path) -> None:
+    try:
+        metadata = run_root.lstat()
+        if (
+            not stat.S_ISDIR(metadata.st_mode)
+            or stat.S_ISLNK(metadata.st_mode)
+            or metadata.st_uid != os.getuid()
+            or stat.S_IMODE(metadata.st_mode) not in {0o500, 0o700}
+        ):
+            raise AcceptanceFailure("failure-root-contract")
+        os.chmod(run_root, 0o700)
+    except AcceptanceFailure:
+        raise
+    except OSError as exc:
+        raise AcceptanceFailure("failure-root-unseal") from exc
+    for child in tuple(run_root.iterdir()):
+        if child.is_dir() and not child.is_symlink():
+            _safe_remove_tree(child)
+        else:
+            child.unlink()
+
+
+def _install_interrupt_handlers() -> dict[signal.Signals, object]:
+    global _SIGNAL_CLEANUP_DEPTH, _SIGNAL_INTERRUPTED
+    _SIGNAL_INTERRUPTED = False
+    _SIGNAL_CLEANUP_DEPTH = 0
+
+    def interrupt(signum: int, frame: object) -> None:
+        del frame
+        global _SIGNAL_INTERRUPTED
+        if signum not in {signal.SIGINT, signal.SIGTERM} or _SIGNAL_INTERRUPTED:
+            return
+        _SIGNAL_INTERRUPTED = True
+        if _SIGNAL_CLEANUP_DEPTH:
+            return
+        raise AcceptanceFailure("interrupted")
+
+    previous: dict[signal.Signals, object] = {}
+    try:
+        for requested in (signal.SIGINT, signal.SIGTERM):
+            previous[requested] = signal.getsignal(requested)
+            signal.signal(requested, interrupt)
+    except ValueError:
+        return {}
+    return previous
+
+
+def _restore_interrupt_handlers(previous: Mapping[signal.Signals, object]) -> None:
+    for requested, handler in previous.items():
+        signal.signal(requested, handler)
+
+
+def main(arguments: Sequence[str] | None = None) -> int:
+    previous_handlers = _install_interrupt_handlers()
+    namespace: argparse.Namespace | None = None
+    run_root: Path | None = None
+    work_root: Path | None = None
+    receipt: dict[str, object] | None = None
+    failure: str | None = None
+    try:
+        namespace = parse_arguments(sys.argv[1:] if arguments is None else arguments)
+        origins = _validated_origins(namespace)
+        spki = validate_spki(namespace.certificate_spki)
+        if namespace.expected_cli_version != REVIEWED_CLI_VERSION:
+            raise AcceptanceFailure("cli-version-not-reviewed")
+        if not SAFE_RELEASE.fullmatch(namespace.release_sha):
+            raise AcceptanceFailure("invalid-release-sha")
+        run_root, work_root = create_run_directory(namespace.output_label)
+        report = run_root / "acceptance-report.json"
+        toolchain = prepare_toolchain(work_root)
+        receipt = run_matrix(
+            run_root=run_root, work_root=work_root, origins=origins, spki=spki,
+            expected_cli_version=namespace.expected_cli_version,
+            release_sha=namespace.release_sha, toolchain=toolchain,
+        )
+        _safe_remove_tree(work_root)
+        atomic_json(report, receipt)
+        _seal_persistent_artifacts(run_root, report, receipt)
+    except AcceptanceFailure as exc:
+        failure = exc.check
+    except Exception:
+        failure = "internal-error"
+    if failure is not None or receipt is None:
+        failure = failure or "internal-error"
+        if work_root is not None:
+            try:
+                _safe_remove_tree(work_root)
+            except AcceptanceFailure:
+                failure = "transient-cleanup-failed"
+        if run_root is not None:
+            try:
+                _purge_failed_artifacts(run_root)
+                atomic_json(
+                    run_root / "acceptance-report.json",
+                    {
+                        "completed_at_utc": utc_now(),
+                        "failure_check": failure,
+                        "release_sha": namespace.release_sha
+                        if namespace is not None
+                        and SAFE_RELEASE.fullmatch(namespace.release_sha)
+                        else "invalid",
+                        "schema_version": 2,
+                        "status": "failed",
+                    },
+                )
+                _seal_failure_report(
+                    run_root, run_root / "acceptance-report.json"
+                )
+            except AcceptanceFailure:
+                pass
+        _restore_interrupt_handlers(previous_handlers)
+        print(f"browser_acceptance=failed check={failure}", file=sys.stderr)
+        return 1
+    _restore_interrupt_handlers(previous_handlers)
+    print(
+        "browser_acceptance=passed viewports=4 scenarios=13 "
+        f"screenshots={len(receipt['screenshots'])} "
+        f"report=output/playwright/{namespace.output_label}/acceptance-report.json"
+    )
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
new file mode 100644
index 0000000..b0db2a3
--- /dev/null
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -0,0 +1,1233 @@
+import base64
+import hashlib
+import importlib
+import json
+import os
+from pathlib import Path
+import signal
+import socket
+import ssl
+import stat
+import struct
+import subprocess
+import sys
+import tempfile
+import time
+import unittest
+from unittest.mock import patch
+import zlib
+
+from e2e import browser_acceptance as acceptance
+
+
+DJANGO_CSRF_SERVER_CODE = r"""
+import html
+from pathlib import Path
+import ssl
+import sys
+import time
+import types
+from wsgiref.simple_server import WSGIRequestHandler, WSGIServer, make_server
+
+from django.conf import settings
+
+port = int(sys.argv[1])
+certificate = sys.argv[2]
+private_key = sys.argv[3]
+result_file = Path(sys.argv[4])
+url_module_name = "isolated_browser_csrf_urls"
+settings.configure(
+    ALLOWED_HOSTS=["127.0.0.1"],
+    CSRF_COOKIE_HTTPONLY=True,
+    CSRF_COOKIE_SAMESITE="Strict",
+    CSRF_COOKIE_SECURE=True,
+    DEBUG=False,
+    DEFAULT_CHARSET="utf-8",
+    INSTALLED_APPS=[],
+    LOGGING_CONFIG=None,
+    MIDDLEWARE=["django.middleware.csrf.CsrfViewMiddleware"],
+    ROOT_URLCONF=url_module_name,
+    SECRET_KEY="isolated-browser-csrf-regression-only",
+    USE_TZ=True,
+)
+import django
+django.setup(set_prefix=False)
+from django.http import HttpResponse
+from django.middleware.csrf import get_token
+from django.urls import path
+
+def form_view(request):
+    if request.method == "POST":
+        time.sleep(0.15)
+        result_file.write_text("1\n", encoding="ascii")
+        return HttpResponse(status=303, headers={"Location": "/results/"})
+    token = html.escape(get_token(request), quote=True)
+    body = f'''<!doctype html><html lang="ko"><head><meta charset="utf-8"><title>CSRF test</title></head><body>
+<form method="post" action="/"><input type="hidden" name="csrfmiddlewaretoken" value="{token}">
+<label for="departure">출국일</label><input id="departure" name="departure_date" type="date">
+<label for="return">귀국일</label><input id="return" name="return_date" type="date">
+<button type="submit">게시 정보 확인</button></form></body></html>'''
+    return HttpResponse(body, content_type="text/html; charset=utf-8")
+
+def results_view(request):
+    return HttpResponse("<!doctype html><html lang=\"ko\"><title>ok</title><body>ok</body></html>", content_type="text/html; charset=utf-8")
+
+urls = types.ModuleType(url_module_name)
+urls.urlpatterns = [path("", form_view), path("results/", results_view)]
+sys.modules[url_module_name] = urls
+
+class HTTPSRequestHandler(WSGIRequestHandler):
+    def get_environ(self):
+        environ = super().get_environ()
+        environ["wsgi.url_scheme"] = "https"
+        environ["HTTPS"] = "on"
+        return environ
+
+    def log_message(self, format, *args):
+        return
+
+server = make_server("127.0.0.1", port, __import__("django.core.wsgi", fromlist=["get_wsgi_application"]).get_wsgi_application(), server_class=WSGIServer, handler_class=HTTPSRequestHandler)
+context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
+context.minimum_version = ssl.TLSVersion.TLSv1_2
+context.load_cert_chain(certificate, private_key)
+server.socket = context.wrap_socket(server.socket, server_side=True)
+server.serve_forever()
+"""
+
+
+class BrowserAcceptanceHarnessTests(unittest.TestCase):
+    maxDiff = None
+
+    def setUp(self):
+        self.spki = base64.b64encode(b"s" * 32).decode("ascii")
+        self.origins = {
+            name: f"https://127.0.0.1:{9443 + index}"
+            for index, name in enumerate(acceptance.ORIGIN_NAMES)
+        }
+
+    @staticmethod
+    def private_directory(parent: Path, name: str) -> Path:
+        path = parent / name
+        path.mkdir(mode=0o700)
+        path.chmod(0o700)
+        return path
+
+    @staticmethod
+    def private_executable(parent: Path, name: str = "chrome") -> Path:
+        path = parent / name
+        path.write_bytes(b"test-only executable\n")
+        path.chmod(0o500)
+        return path
+
+    @staticmethod
+    def unused_port() -> int:
+        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as probe:
+            probe.bind(("127.0.0.1", 0))
+            return probe.getsockname()[1]
+
+    def make_certificate(self, root: Path) -> tuple[Path, Path, str]:
+        certificate = root / "csrf-cert.pem"
+        private_key = root / "csrf-private-key.pem"
+        configuration = root / "csrf-cert.cnf"
+        configuration.write_text(
+            "[req]\n"
+            "distinguished_name=dn\n"
+            "x509_extensions=v3\n"
+            "prompt=no\n"
+            "[dn]\n"
+            "CN=127.0.0.1\n"
+            "[v3]\n"
+            "subjectAltName=IP:127.0.0.1\n"
+            "basicConstraints=critical,CA:TRUE\n"
+            "keyUsage=critical,digitalSignature,keyEncipherment,keyCertSign\n"
+            "extendedKeyUsage=serverAuth\n",
+            encoding="ascii",
+        )
+        generated = subprocess.run(
+            [
+                "/usr/bin/openssl", "req", "-x509", "-newkey", "rsa:2048",
+                "-nodes", "-days", "1", "-config", str(configuration),
+                "-keyout", str(private_key), "-out", str(certificate),
+            ],
+            env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C"},
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+            timeout=15,
+        )
+        self.assertEqual(generated.returncode, 0)
+        private_key.chmod(0o600)
+        der = ssl.PEM_cert_to_DER_cert(certificate.read_text(encoding="ascii"))
+        return certificate, private_key, acceptance.certificate_spki_sha256(der)
+
+    def wait_for_https(self, port: int, process: subprocess.Popen[bytes]) -> None:
+        context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
+        context.check_hostname = False
+        context.verify_mode = ssl.CERT_NONE
+        deadline = time.monotonic() + 10
+        while time.monotonic() < deadline:
+            if process.poll() is not None:
+                self.fail("isolated Django CSRF server exited before readiness")
+            try:
+                with socket.create_connection(("127.0.0.1", port), timeout=0.25) as plain:
+                    with context.wrap_socket(plain, server_hostname="127.0.0.1"):
+                        return
+            except (OSError, ssl.SSLError):
+                time.sleep(0.05)
+        self.fail("isolated Django CSRF server did not become ready")
+
+    def fake_toolchain(self, root: Path) -> acceptance.Toolchain:
+        daemon = self.private_directory(root, "daemon")
+        global_root = self.private_directory(root, "global")
+        home = self.private_directory(root, "home")
+        sealed = self.private_directory(root, "sealed")
+        chrome = self.private_executable(sealed)
+        sealed.chmod(0o500)
+        return acceptance.Toolchain(
+            wrapper=Path("/test-only/playwright-wrapper"),
+            environment={"PATH": "/test-only/private-bin", "HOME": str(home)},
+            daemon_root=daemon,
+            global_root=global_root,
+            sealed_root=sealed,
+            chrome_executable=chrome,
+            chrome_tree_sha256=acceptance.CHROME_TREE_SHA256,
+            sealed_tree_sha256=acceptance.CHROME_TREE_SHA256,
+            sealed_entry_count=0,
+            sealed_file_bytes=0,
+        )
+
+    def test_matrix_state_pairs_and_loading_origin_are_fixed(self):
+        self.assertEqual(
+            acceptance.VIEWPORTS,
+            ((360, 800), (390, 844), (768, 1024), (1440, 900)),
+        )
+        self.assertEqual(
+            acceptance.ORIGIN_NAMES,
+            (
+                "ready",
+                "loading",
+                "empty",
+                "unavailable",
+                "stale",
+                "server-error",
+                "long-korean",
+            ),
+        )
+        self.assertEqual(
+            acceptance.STATE_PAIRS,
+            {
+                "ready": ("ready", "ready"),
+                "empty": ("empty", "empty"),
+                "unavailable": ("ready", "unavailable"),
+                "stale": ("stale", "stale"),
+                "server-error": ("server-error", "server-error"),
+                "long-korean": ("ready", "ready"),
+            },
+        )
+        self.assertEqual(len(acceptance.SCREENSHOT_NAMES), 13)
+
+    def test_urls_are_distinct_explicit_loopback_https_origins(self):
+        self.assertEqual(
+            acceptance.validate_base_url("https://localhost:9443/", field="ready"),
+            "https://localhost:9443",
+        )
+        self.assertEqual(
+            acceptance.validate_base_url("https://[::1]:9443", field="ready"),
+            "https://[::1]:9443",
+        )
+        invalid = (
+            "http://127.0.0.1:9443",
+            "https://example.invalid:9443",
+            "https://name:password@127.0.0.1:9443",
+            "https://127.0.0.1:9443/?state=stale",
+            "https://127.0.0.1:9443/results/",
+            "https://127.0.0.1:9443/#state",
+            "https://127.0.0.1",
+            "https://127.0.0.1:0",
+        )
+        for value in invalid:
+            with self.subTest(value=value):
+                with self.assertRaises(acceptance.AcceptanceFailure):
+                    acceptance.validate_base_url(value, field="scenario")
+
+    def test_cli_config_pins_chrome_and_blocks_persistent_browser_artifacts(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            output = Path(temporary)
+            output.chmod(0o700)
+            chrome = self.private_executable(output)
+            config = acceptance.build_cli_config(
+                width=390,
+                height=844,
+                origins=list(self.origins.values()),
+                spki=self.spki,
+                output_dir=output,
+                chrome_executable=chrome,
+            )
+        browser = config["browser"]
+        launch = browser["launchOptions"]
+        context = browser["contextOptions"]
+        self.assertEqual(browser["browserName"], "chromium")
+        self.assertTrue(browser["isolated"])
+        self.assertEqual(launch["executablePath"], str(chrome))
+        self.assertNotIn("channel", launch)
+        self.assertTrue(launch["headless"])
+        self.assertIn(
+            f"--ignore-certificate-errors-spki-list={self.spki}", launch["args"]
+        )
+        self.assertIn("--disable-background-networking", launch["args"])
+        self.assertFalse(context["ignoreHTTPSErrors"])
+        self.assertFalse(context["acceptDownloads"])
+        self.assertEqual(context["serviceWorkers"], "block")
+        self.assertEqual(context["viewport"], {"width": 390, "height": 844})
+        self.assertTrue(context["hasTouch"])
+        self.assertEqual(
+            config["network"]["allowedOrigins"],
+            [f"{origin}/**" for origin in self.origins.values()],
+        )
+        self.assertFalse(config["saveTrace"])
+        self.assertFalse(config["saveVideo"])
+        self.assertFalse(config["saveSession"])
+        self.assertNotIn("expect", config["timeouts"])
+        serialized = json.dumps(config)
+        self.assertNotIn("recordHar", serialized)
+        self.assertNotIn("proxy", serialized)
+
+    def test_desktop_config_does_not_claim_touch_emulation(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            output = Path(temporary)
+            output.chmod(0o700)
+            chrome = self.private_executable(output)
+            config = acceptance.build_cli_config(
+                width=1440,
+                height=900,
+                origins=list(self.origins.values()),
+                spki=self.spki,
+                output_dir=output,
+                chrome_executable=chrome,
+            )
+        self.assertFalse(config["browser"]["contextOptions"]["hasTouch"])
+
+    def test_restricted_environment_has_private_path_and_no_inherited_values(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            values = [self.private_directory(root, name) for name in ("bin", "work", "daemon", "global")]
+            self.private_directory(values[1], "home")
+            self.private_directory(values[1], "tmp")
+            environment = acceptance.restricted_cli_environment(
+                bin_directory=values[0],
+                work_root=values[1],
+                daemon_root=values[2],
+                global_root=values[3],
+                chrome_executable=Path("/test-only/chrome"),
+            )
+        self.assertEqual(environment["PATH"], str(values[0]))
+        self.assertEqual(environment["PWTEST_DAEMON_SESSION_DIR"], str(values[2]))
+        self.assertEqual(environment["PWTEST_CLI_GLOBAL_CONFIG"], str(values[3]))
+        self.assertEqual(environment["CI"], "1")
+        self.assertEqual(environment["NPM_CONFIG_OFFLINE"], "true")
+        self.assertEqual(environment["NO_UPDATE_NOTIFIER"], "1")
+        self.assertEqual(environment["PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD"], "1")
+        for name in (
+            "MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            "DATABASE_URL",
+            "HTTPS_PROXY",
+            "SSLKEYLOGFILE",
+            "NODE_OPTIONS",
+            "NPM_TOKEN",
+        ):
+            self.assertNotIn(name, environment)
+
+    def test_only_reviewed_cli_version_can_be_requested(self):
+        self.assertEqual(acceptance.REVIEWED_CLI_VERSION, "0.1.18")
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            run_root = self.private_directory(root, "run")
+            work_root = self.private_directory(run_root, ".work")
+            with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                acceptance.run_matrix(
+                    run_root=run_root,
+                    work_root=work_root,
+                    origins=self.origins,
+                    spki=self.spki,
+                    expected_cli_version="0.1.19",
+                    release_sha="a" * 40,
+                )
+        self.assertEqual(caught.exception.check, "cli-version-not-reviewed")
+
+    def test_release_document_requires_exact_sha_schema_tls_response_headers(self):
+        release = "a" * 40
+        acceptance.validate_release_document(
+            status=200,
+            content_type="application/json; charset=utf-8",
+            cache_control="private, no-store",
+            body=json.dumps({"release_sha": release}).encode(),
+            expected_sha=release,
+        )
+        adversarial = (
+            {"status": 201},
+            {"content_type": "text/plain"},
+            {"cache_control": "no-cache"},
+            {"body": b'{"release_sha":"' + b"b" * 40 + b'"}'},
+            {"body": json.dumps({"release_sha": release, "extra": 1}).encode()},
+            {"body": b"not-json"},
+        )
+        baseline = {
+            "status": 200,
+            "content_type": "application/json",
+            "cache_control": "no-store",
+            "body": json.dumps({"release_sha": release}).encode(),
+            "expected_sha": release,
+        }
+        for override in adversarial:
+            with self.subTest(override=override):
+                with self.assertRaises(acceptance.AcceptanceFailure):
+                    acceptance.validate_release_document(**(baseline | override))
+
+    def test_release_javascript_fetches_same_origin_and_checks_tls(self):
+        code = acceptance.verify_release_javascript(
+            origin=self.origins["ready"], release_sha="a" * 40, check="release"
+        )
+        self.assertIn(self.origins["ready"] + "/releasez", code)
+        self.assertIn("securityDetails", code)
+        self.assertIn("TLS 1.2", code)
+        self.assertIn("Object.keys(document).length !== 1", code)
+        self.assertNotIn("caller_release", code)
+
+    def test_bad_certificate_der_fails_closed(self):
+        for document in (b"", b"0\x00", b"0\x81\x01\x00"):
+            with self.subTest(document=document):
+                with self.assertRaises(acceptance.AcceptanceFailure):
+                    acceptance.certificate_spki_sha256(document)
+
+    def test_certificate_parser_hashes_exact_subject_public_key_info_tlv(self):
+        def tlv(tag, payload):
+            length = len(payload)
+            if length < 128:
+                encoded_length = bytes([length])
+            else:
+                octets = length.to_bytes((length.bit_length() + 7) // 8, "big")
+                encoded_length = bytes([0x80 | len(octets)]) + octets
+            return bytes([tag]) + encoded_length + payload
+
+        version = tlv(0xA0, tlv(0x02, b"\x02"))
+        serial = tlv(0x02, b"\x01")
+        algorithm = tlv(0x30, tlv(0x06, b"\x2a"))
+        issuer = tlv(0x30, b"")
+        validity = tlv(0x30, b"")
+        subject = tlv(0x30, b"")
+        spki = tlv(0x30, tlv(0x30, tlv(0x06, b"\x2a")) + tlv(0x03, b"\x00\x01"))
+        tbs = tlv(0x30, version + serial + algorithm + issuer + validity + subject + spki)
+        certificate = tlv(0x30, tbs + algorithm + tlv(0x03, b"\x00"))
+        expected = base64.b64encode(hashlib.sha256(spki).digest()).decode()
+        self.assertEqual(acceptance.certificate_spki_sha256(certificate), expected)
+
+    def test_generated_run_code_is_valid_javascript(self):
+        snippets = [
+            acceptance.install_guards_javascript(
+                origins=list(self.origins.values()), check="guards"
+            ),
+            acceptance.verify_release_javascript(
+                origin=self.origins["ready"], release_sha="a" * 40, check="release"
+            ),
+            acceptance.form_pristine_javascript(
+                origin=self.origins["ready"], check="pristine"
+            ),
+            acceptance.loading_start_javascript(
+                origin=self.origins["loading"], check="loading-start"
+            ),
+            acceptance.csrf_keyboard_submit_javascript(
+                origin=self.origins["ready"], check="csrf-submit"
+            ),
+            acceptance.csrf_keyboard_prepare_javascript(
+                origin=self.origins["ready"], check="csrf-prepare"
+            ),
+            acceptance.csrf_keyboard_observe_javascript(
+                origin=self.origins["ready"], check="csrf-observe"
+            ),
+            acceptance.csrf_keyboard_fill_javascript(
+                origin=self.origins["ready"], check="csrf-fill"
+            ),
+            acceptance.csrf_keyboard_press_javascript(
+                origin=self.origins["ready"], check="csrf-press"
+            ),
+            acceptance.csrf_keyboard_finish_javascript(
+                origin=self.origins["ready"], check="csrf-finish"
+            ),
+            *(
+                acceptance.csrf_contract_probe_javascript(
+                    origin=self.origins["ready"], aspect=aspect,
+                    check=f"csrf-probe-{aspect}",
+                )
+                for aspect in (
+                    "location", "client-state", "count", "name",
+                    "value-present", "secure", "http-only", "same-site",
+                )
+            ),
+            acceptance.loading_finish_javascript(
+                origin=self.origins["loading"], check="loading-finish"
+            ),
+            acceptance.validation_javascript(
+                origin=self.origins["ready"], check="validation"
+            ),
+            acceptance.correction_javascript(
+                origin=self.origins["ready"], check="correction"
+            ),
+            acceptance.results_keyboard_javascript(check="keyboard"),
+            acceptance.forced_colors_javascript(check="colors"),
+            acceptance.reset_media_javascript(check="media"),
+            acceptance.text_scale_javascript(
+                origin=self.origins["ready"], path="/", check="scale"
+            ),
+            acceptance.scaled_form_submit_javascript(
+                origin=self.origins["ready"], check="scaled-submit"
+            ),
+            acceptance.final_guard_javascript(check="final", expected_posts=4),
+        ]
+        snippets.extend(
+            acceptance.results_javascript(
+                origin=self.origins[state], state=state, check=state
+            )
+            for state in acceptance.STATE_NAMES
+        )
+        for index, snippet in enumerate(snippets):
+            with self.subTest(index=index):
+                completed = subprocess.run(
+                    [str(acceptance.NODE_SOURCE), "--check", "-"],
+                    input=f"({snippet});\n",
+                    text=True,
+                    stdout=subprocess.PIPE,
+                    stderr=subprocess.PIPE,
+                    check=False,
+                )
+                self.assertEqual(completed.returncode, 0, completed.stderr)
+
+    def test_submission_contract_is_keyboard_single_post_and_privacy_safe(self):
+        source = acceptance.loading_start_javascript(
+            origin=self.origins["loading"], check="loading"
+        ) + acceptance.loading_finish_javascript(
+            origin=self.origins["loading"], check="loading-finish"
+        )
+        self.assertIn("page.keyboard.press('Enter')", source)
+        self.assertIn("state.postCount !== 1", source)
+        self.assertIn("state.cookiePostCount !== 1", source)
+        self.assertIn("state.statuses.length !== 1", source)
+        self.assertIn("post-submit-cookie-state", source)
+        self.assertIn("finishSubmit(200, false)", acceptance.validation_javascript(
+            origin=self.origins["ready"], check="validation"
+        ))
+        self.assertIn("await route.continue();", source)
+        self.assertNotIn("route.continue({ headers", source)
+        self.assertNotIn("const csrf =", source)
+        self.assertIn("indexedDB.databases", source)
+        self.assertIn("caches.keys", source)
+        self.assertIn("getRegistrations", source)
+        self.assertNotIn("dispatchEvent(new Event('submit'", source)
+
+        form_source = acceptance.form_pristine_javascript(
+            origin=self.origins["ready"], check="form-privacy"
+        )
+        self.assertIn("csrfmiddlewaretoken", form_source)
+        self.assertIn("hidden.length !== 1", form_source)
+        self.assertIn("allowedNames = new Set", form_source)
+        self.assertIn("dom-privacy-", form_source)
+        self.assertIn("crypto.subtle.digest('SHA-256'", form_source)
+
+    def test_state_contract_has_exact_cards_metadata_and_nonvacuous_notes(self):
+        source = acceptance.results_javascript(
+            origin=self.origins["unavailable"], state="unavailable", check="state"
+        )
+        self.assertIn("expected = { entry: \"ready\", warning: \"unavailable\" }", source)
+        self.assertIn("외교부 입국요건 source 열기", source)
+        self.assertIn("외교부 여행경보 source 열기", source)
+        self.assertIn("마지막 성공 확인시각", source)
+        self.assertIn("source 작성일", source)
+        self.assertIn("noteCount !== publicationCount", source)
+        self.assertIn("metadata.h2Size > metadata.bodySize", source)
+        self.assertIn("freshnessMinutes", source)
+        self.assertIn(acceptance.ENTRY_PUBLIC_SOURCE_LOCATOR, source)
+        self.assertIn(acceptance.WARNING_PUBLIC_SOURCE_LOCATOR, source)
+        self.assertIn("raw-secret-node", source)
+        self.assertIn("document.head.children", source)
+        self.assertIn("allowedNames = new Set", source)
+        self.assertIn("crypto.subtle.digest('SHA-256'", source)
+        self.assertIn(acceptance.SITE_CSS_SHA256, source)
+        self.assertIn(acceptance.SITE_JS_SHA256, source)
+        self.assertIn("parseDateOnly", source)
+        self.assertIn("getUTCDate() === parts[2]", source)
+        self.assertIn("publishedMillis > nowMillis + 300000", source)
+
+    def test_pinned_static_assets_match_the_reviewed_bytes(self):
+        assets = (
+            (
+                acceptance.REPOSITORY_ROOT / "public_web/static/public_web/site.css",
+                acceptance.SITE_CSS_SHA256,
+                acceptance.SITE_CSS_BYTES,
+            ),
+            (
+                acceptance.REPOSITORY_ROOT / "public_web/static/public_web/site.js",
+                acceptance.SITE_JS_SHA256,
+                acceptance.SITE_JS_BYTES,
+            ),
+        )
+        for path, expected_digest, expected_size in assets:
+            with self.subTest(path=path.name):
+                payload = path.read_bytes()
+                self.assertEqual(len(payload), expected_size)
+                self.assertEqual(hashlib.sha256(payload).hexdigest(), expected_digest)
+
+    def test_marker_must_be_an_exact_output_line(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            toolchain = self.fake_toolchain(root)
+            session = acceptance.CliSession(
+                session_name="marker-test",
+                config_path=root / "config.json",
+                working_directory=root,
+                toolchain=toolchain,
+            )
+            marker = "PW_ACCEPTANCE_OK:marker"
+            with patch.object(
+                acceptance,
+                "_bounded_process",
+                return_value=(0, f'return "{marker}";\n'.encode(), b""),
+            ):
+                with self.assertRaises(acceptance.AcceptanceFailure):
+                    session.invoke("marker", "run-code", "source", marker=marker)
+            with patch.object(
+                acceptance,
+                "_bounded_process",
+                return_value=(0, (json.dumps(marker) + "\n").encode(), b""),
+            ):
+                self.assertEqual(
+                    session.invoke("marker", "run-code", "source", marker=marker),
+                    (json.dumps(marker) + "\n").encode(),
+                )
+
+    def test_failed_open_recovers_only_bounded_daemon_pid_for_cleanup(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            toolchain = self.fake_toolchain(root)
+            session = acceptance.CliSession(
+                session_name="failed-open",
+                config_path=root / "config.json",
+                working_directory=root,
+                toolchain=toolchain,
+            )
+            with (
+                patch.object(
+                    acceptance,
+                    "_bounded_process",
+                    return_value=(1, b"private-page", b"Daemon pid=43210: fixed error"),
+                ),
+                patch.object(acceptance, "_pid_alive", return_value=True),
+            ):
+                with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                    session.invoke("open", "open", "about:blank")
+            self.assertEqual(caught.exception.check, "cli-open-failed")
+            self.assertEqual(session.daemon_pid, 43210)
+            self.assertNotIn("private", str(caught.exception))
+
+    def test_bounded_process_drains_both_streams_concurrently(self):
+        payload_size = 300_000
+        code = (
+            "import os;"
+            f"os.write(1,b'o'*{payload_size});"
+            f"os.write(2,b'e'*{payload_size})"
+        )
+        returncode, stdout, stderr = acceptance._bounded_process(
+            [sys.executable, "-I", "-c", code],
+            cwd=acceptance.REPOSITORY_ROOT,
+            environment={"PATH": "/usr/bin:/bin"},
+            timeout=10,
+            check="bounded-output",
+        )
+        self.assertEqual(returncode, 0)
+        self.assertEqual(len(stdout), payload_size)
+        self.assertEqual(len(stderr), payload_size)
+
+    def test_bounded_process_output_overflow_is_fixed_error_and_reaped(self):
+        code = f"import os;os.write(1,b'x'*{acceptance.MAX_CLI_OUTPUT_BYTES + 1})"
+        with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+            acceptance._bounded_process(
+                [sys.executable, "-I", "-c", code],
+                cwd=acceptance.REPOSITORY_ROOT,
+                environment={"PATH": "/usr/bin:/bin"},
+                timeout=10,
+                check="overflow",
+            )
+        self.assertEqual(caught.exception.check, "cli-overflow-output-limit")
+        self.assertNotIn("xxx", str(caught.exception))
+
+    def test_sigterm_is_converted_to_fixed_cleanup_exception(self):
+        previous = acceptance._install_interrupt_handlers()
+        try:
+            with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                os.kill(os.getpid(), signal.SIGTERM)
+            self.assertEqual(caught.exception.check, "interrupted")
+        finally:
+            acceptance._restore_interrupt_handlers(previous)
+            acceptance._SIGNAL_INTERRUPTED = False
+            acceptance._SIGNAL_CLEANUP_DEPTH = 0
+
+    def test_snapshot_is_bounded_parsed_and_always_removed(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            toolchain = self.fake_toolchain(root)
+            session = acceptance.CliSession(
+                session_name="snapshot-test",
+                config_path=root / "config.json",
+                working_directory=root,
+                toolchain=toolchain,
+            )
+
+            def fake_invoke(check, *arguments, **kwargs):
+                del check, kwargs
+                filename = next(
+                    value.split("=", 1)[1]
+                    for value in arguments
+                    if value.startswith("--filename=")
+                )
+                Path(filename).write_text("일본 게시 정보\n입국요건 사실\n", encoding="utf-8")
+                return b""
+
+            with patch.object(session, "invoke", side_effect=fake_invoke):
+                session.snapshot(
+                    check="semantic", required_tokens=("일본 게시 정보", "입국요건 사실")
+                )
+            self.assertFalse((root / "transient-semantic.md").exists())
+
+    def test_fake_full_matrix_has_52_private_screenshots_and_no_values(self):
+        calls: list[tuple[str, str]] = []
+
+        def fake_invoke(session, check, *arguments, **kwargs):
+            del session, kwargs
+            if arguments == ("--version",):
+                return b"0.1.18\n"
+            calls.append((check, arguments[0] if arguments else ""))
+            return b""
+
+        def fake_open(session, url):
+            self.assertEqual(url, "about:blank")
+            session.opened = True
+
+        def fake_close(session):
+            session.opened = False
+
+        def fake_goto(session, url, *, check):
+            del session, url
+            calls.append((check, "goto"))
+
+        def fake_snapshot(session, *, check, required_tokens):
+            del session
+            self.assertTrue(required_tokens)
+            calls.append((check, "snapshot"))
+
+        def fake_run_code(session, check, code):
+            del session
+            self.assertNotIn("private-page-value", code)
+            calls.append((check, "run-code"))
+
+        def fake_screenshot(session, relative_name, *, check):
+            config = json.loads(session.config_path.read_text(encoding="utf-8"))
+            viewport = config["browser"]["contextOptions"]["viewport"]
+            destination = Path(config["outputDir"]) / relative_name
+            destination.parent.mkdir(mode=0o700, exist_ok=True)
+            destination.parent.chmod(0o700)
+            ihdr = struct.pack(
+                ">IIBBBBB", viewport["width"], viewport["height"] + 100, 8, 2, 0, 0, 0
+            )
+
+            def chunk(kind, data):
+                return (
+                    struct.pack(">I", len(data))
+                    + kind
+                    + data
+                    + struct.pack(">I", zlib.crc32(kind + data) & 0xFFFFFFFF)
+                )
+
+            destination.write_bytes(
+                acceptance.PNG_SIGNATURE
+                + chunk(b"IHDR", ihdr)
+                + chunk(b"IDAT", zlib.compress(b""))
+                + chunk(b"IEND", b"")
+            )
+            destination.chmod(0o600)
+            calls.append((check, "screenshot"))
+
+        def fake_tree_digest(root, *, include_modes):
+            del include_modes
+            if root == acceptance.CACHED_CLI_ROOT:
+                return (
+                    acceptance.CACHED_CLI_TREE_SHA256,
+                    acceptance.CACHED_CLI_ENTRY_COUNT,
+                    acceptance.CACHED_CLI_FILE_BYTES,
+                )
+            if root == acceptance.CHROME_BUNDLE:
+                return (
+                    acceptance.CHROME_TREE_SHA256,
+                    acceptance.CHROME_ENTRY_COUNT,
+                    acceptance.CHROME_FILE_BYTES,
+                )
+            return (acceptance.CHROME_TREE_SHA256, 0, 0)
+
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            run_root = self.private_directory(root, "run")
+            work_root = self.private_directory(run_root, ".work")
+            toolchain = self.fake_toolchain(work_root)
+            with (
+                patch.object(acceptance, "_verify_all_tls"),
+                patch.object(
+                    acceptance,
+                    "_tree_digest",
+                    side_effect=fake_tree_digest,
+                ),
+                patch.object(acceptance.CliSession, "invoke", fake_invoke),
+                patch.object(acceptance.CliSession, "open", fake_open),
+                patch.object(acceptance.CliSession, "close", fake_close),
+                patch.object(acceptance.CliSession, "goto", fake_goto),
+                patch.object(acceptance.CliSession, "snapshot", fake_snapshot),
+                patch.object(acceptance.CliSession, "run_code", fake_run_code),
+                patch.object(acceptance.CliSession, "screenshot", fake_screenshot),
+            ):
+                receipt = acceptance.run_matrix(
+                    run_root=run_root,
+                    work_root=work_root,
+                    origins=self.origins,
+                    spki=self.spki,
+                    expected_cli_version="0.1.18",
+                    release_sha="a" * 40,
+                    toolchain=toolchain,
+                )
+            acceptance._safe_remove_tree(work_root)
+            report = run_root / "acceptance-report.json"
+            acceptance.atomic_json(report, receipt)
+            acceptance._seal_persistent_artifacts(run_root, report, receipt)
+            self.assertEqual(stat.S_IMODE(run_root.stat().st_mode), 0o500)
+            self.assertEqual(stat.S_IMODE(report.stat().st_mode), 0o400)
+            for item in receipt["screenshots"]:
+                self.assertEqual(
+                    stat.S_IMODE((run_root / item["file"]).stat().st_mode), 0o400
+                )
+            serialized = json.dumps(receipt, ensure_ascii=False, sort_keys=True)
+            acceptance._safe_remove_tree(run_root)
+        self.assertEqual(receipt["status"], "passed")
+        self.assertEqual(len(receipt["screenshots"]), 52)
+        self.assertEqual(
+            {item["viewport"] for item in receipt["screenshots"]},
+            {"360x800", "390x844", "768x1024", "1440x900"},
+        )
+        for value in (*self.origins.values(), self.spki, acceptance.SYNTHETIC_DEPARTURE):
+            self.assertNotIn(value, serialized)
+        self.assertEqual(sum(command == "screenshot" for _, command in calls), 52)
+        self.assertEqual(sum(check == "form-loading-start" for check, _ in calls), 4)
+
+    def test_toolchain_wrapper_and_real_https_django_csrf_submit_leave_no_residue(self):
+        """Real Chrome and Django; run with local GUI/process permission."""
+
+        with tempfile.TemporaryDirectory() as temporary:
+            container = Path(temporary)
+            container.chmod(0o700)
+            root = self.private_directory(container, "work")
+            toolchain = acceptance.prepare_toolchain(root)
+            self.assertEqual(
+                Path(toolchain.environment["PATH"]).resolve(),
+                (root / "toolchain" / "bin").resolve(),
+            )
+            self.assertTrue(toolchain.chrome_executable.is_relative_to(toolchain.sealed_root))
+            self.assertEqual(
+                acceptance._verify_sealed_toolchain_tree(toolchain.sealed_root),
+                (
+                    toolchain.sealed_tree_sha256,
+                    toolchain.sealed_entry_count,
+                    toolchain.sealed_file_bytes,
+                ),
+            )
+            self.assertNotIn("/opt/homebrew/bin", toolchain.environment["PATH"])
+            global_config = toolchain.global_root / ".playwright" / "cli.config.json"
+            self.assertEqual(global_config.read_text(encoding="utf-8"), "{}\n")
+            self.assertEqual(stat.S_IMODE(global_config.stat().st_mode), 0o600)
+            origins = [f"https://127.0.0.1:{9600 + index}" for index in range(7)]
+            config_path = root / "probe-config.json"
+            acceptance.atomic_json(
+                config_path,
+                acceptance.build_cli_config(
+                    width=360,
+                    height=800,
+                    origins=origins,
+                    spki=self.spki,
+                    output_dir=root,
+                    chrome_executable=toolchain.chrome_executable,
+                ),
+            )
+            session = acceptance.CliSession(
+                session_name="cache-probe",
+                config_path=config_path,
+                working_directory=root,
+                toolchain=toolchain,
+            )
+            try:
+                self.assertEqual(session.invoke("version", "--version"), b"0.1.18\n")
+                session.open("about:blank")
+                self.assertTrue(session._daemon_identity_matches(session.daemon_pid))
+                session.run_code(
+                    "probe",
+                    "async (page) => { if (page.url() !== 'about:blank') throw new Error('probe'); return 'PW_ACCEPTANCE_OK:probe'; }",
+                )
+            finally:
+                session.close()
+
+            certificate, private_key, certificate_spki = self.make_certificate(root)
+            result_file = root / "csrf-result.txt"
+            port = self.unused_port()
+            origin = f"https://127.0.0.1:{port}"
+            csrf_process = subprocess.Popen(
+                [
+                    str(acceptance.REPOSITORY_ROOT / ".venv" / "bin" / "python"),
+                    "-I", "-B", "-c", DJANGO_CSRF_SERVER_CODE,
+                    str(port), str(certificate), str(private_key), str(result_file),
+                ],
+                cwd=root,
+                env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C", "TZ": "UTC"},
+                stdin=subprocess.DEVNULL,
+                stdout=subprocess.DEVNULL,
+                stderr=subprocess.DEVNULL,
+                start_new_session=True,
+            )
+            try:
+                self.wait_for_https(port, csrf_process)
+                acceptance.verify_tls_peer(origin, certificate_spki)
+                other_ports: list[int] = []
+                while len(other_ports) < 6:
+                    candidate = self.unused_port()
+                    if candidate != port and candidate not in other_ports:
+                        other_ports.append(candidate)
+                csrf_origins = [origin] + [
+                    f"https://127.0.0.1:{candidate}" for candidate in other_ports
+                ]
+                csrf_config = root / "csrf-config.json"
+                acceptance.atomic_json(
+                    csrf_config,
+                    acceptance.build_cli_config(
+                        width=390,
+                        height=844,
+                        origins=csrf_origins,
+                        spki=certificate_spki,
+                        output_dir=root,
+                        chrome_executable=toolchain.chrome_executable,
+                    ),
+                )
+                csrf_session = acceptance.CliSession(
+                    session_name="django-csrf-submit",
+                    config_path=csrf_config,
+                    working_directory=root,
+                    toolchain=toolchain,
+                )
+                try:
+                    csrf_session.open("about:blank")
+                    csrf_session.run_code(
+                        "csrf-install-guards",
+                        acceptance.install_guards_javascript(
+                            origins=csrf_origins, check="csrf-install-guards"
+                        ),
+                    )
+                    csrf_session.goto(origin + "/", check="csrf-goto-form")
+                    csrf_session.run_code(
+                        "csrf-location-exact",
+                        "async (page) => { "
+                        f"if (page.url() !== {json.dumps(origin + '/')}) throw new Error('acceptance:csrf-location'); "
+                        "return 'PW_ACCEPTANCE_OK:csrf-location-exact'; }",
+                    )
+                    for aspect in (
+                        "location", "client-state", "count", "name",
+                        "value-present", "secure", "http-only", "same-site",
+                    ):
+                        check = f"csrf-probe-{aspect}"
+                        csrf_session.run_code(
+                            check,
+                            acceptance.csrf_contract_probe_javascript(
+                                origin=origin, aspect=aspect, check=check
+                            ),
+                        )
+                    csrf_session.run_code(
+                        "csrf-keyboard-prepare",
+                        acceptance.csrf_keyboard_prepare_javascript(
+                            origin=origin, check="csrf-keyboard-prepare"
+                        ),
+                    )
+                    csrf_session.run_code(
+                        "csrf-keyboard-observe",
+                        acceptance.csrf_keyboard_observe_javascript(
+                            origin=origin, check="csrf-keyboard-observe"
+                        ),
+                    )
+                    csrf_session.run_code(
+                        "csrf-keyboard-fill",
+                        acceptance.csrf_keyboard_fill_javascript(
+                            origin=origin, check="csrf-keyboard-fill"
+                        ),
+                    )
+                    try:
+                        csrf_session.run_code(
+                            "csrf-keyboard-press",
+                            acceptance.csrf_keyboard_press_javascript(
+                                origin=origin, check="csrf-keyboard-press"
+                            ),
+                        )
+                    except acceptance.AcceptanceFailure as exc:
+                        if result_file.exists():
+                            raise acceptance.AcceptanceFailure(
+                                "csrf-regression-post-received"
+                            ) from exc
+                        if csrf_process.poll() is not None:
+                            raise acceptance.AcceptanceFailure(
+                                "csrf-regression-server-exited"
+                            ) from exc
+                        raise acceptance.AcceptanceFailure(
+                            "csrf-regression-post-not-received"
+                        ) from exc
+                    csrf_session.run_code(
+                        "csrf-keyboard-finish",
+                        acceptance.csrf_keyboard_finish_javascript(
+                            origin=origin, check="csrf-keyboard-finish"
+                        ),
+                    )
+                    csrf_session.run_code(
+                        "csrf-final-guard",
+                        acceptance.final_guard_javascript(
+                            check="csrf-final-guard", expected_posts=1
+                        ),
+                    )
+                finally:
+                    csrf_session.close()
+                self.assertEqual(result_file.read_text(encoding="ascii"), "1\n")
+            finally:
+                csrf_process.terminate()
+                try:
+                    csrf_process.wait(timeout=5)
+                except subprocess.TimeoutExpired:
+                    csrf_process.kill()
+                    csrf_process.wait(timeout=5)
+            self.assertEqual(
+                acceptance._verify_sealed_toolchain_tree(toolchain.sealed_root),
+                (
+                    toolchain.sealed_tree_sha256,
+                    toolchain.sealed_entry_count,
+                    toolchain.sealed_file_bytes,
+                ),
+            )
+            self.assertEqual(list(toolchain.daemon_root.rglob("*")), [])
+            self.assertFalse(
+                any(
+                    path.suffix in {".err", ".pid", ".socket", ".session"}
+                    for path in root.rglob("*")
+                )
+            )
+            adapter = root / "toolchain" / "bin" / "npx"
+            rejected = subprocess.run(
+                [str(adapter), "--yes", "--package", "unreviewed", "playwright-cli"],
+                cwd=root,
+                env=toolchain.environment,
+                stdout=subprocess.PIPE,
+                stderr=subprocess.PIPE,
+                check=False,
+            )
+            self.assertEqual(rejected.returncode, 64)
+            self.assertEqual(rejected.stdout, b"")
+            self.assertEqual(rejected.stderr, b"")
+            self.assertEqual(
+                acceptance._tree_digest(acceptance.CACHED_CLI_ROOT, include_modes=True),
+                (
+                    acceptance.CACHED_CLI_TREE_SHA256,
+                    acceptance.CACHED_CLI_ENTRY_COUNT,
+                    acceptance.CACHED_CLI_FILE_BYTES,
+                ),
+            )
+            acceptance._safe_remove_tree(root)
+            self.assertFalse(root.exists())
+
+    def test_daemon_cleanup_rejects_unknown_residue_after_removing_it(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            root.chmod(0o700)
+            unknown = root / "unexpected.session"
+            unknown.write_text("{}", encoding="utf-8")
+            unknown.chmod(0o600)
+            with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                acceptance._remove_daemon_residue(root, allow_err_only=True)
+            self.assertEqual(caught.exception.check, "daemon-unexpected-residue")
+            self.assertEqual(list(root.iterdir()), [])
+
+    def test_daemon_cleanup_oserror_is_a_fixed_failure(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            root.chmod(0o700)
+            with patch.object(
+                acceptance,
+                "_assert_private_directory",
+                side_effect=OSError("private diagnostic must not escape"),
+            ):
+                with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                    acceptance._remove_daemon_residue(root, allow_err_only=True)
+            self.assertEqual(caught.exception.check, "daemon-residue-cleanup")
+            self.assertNotIn("diagnostic", str(caught.exception))
+
+    def test_process_signal_oserror_is_a_fixed_failure(self):
+        identity = acceptance.ProcessIdentity(
+            pid=43210,
+            uid=os.getuid(),
+            process_group=43210,
+            started="Mon Aug 31 00:00:00 2026",
+            command="private reviewed command",
+        )
+        with (
+            patch.object(acceptance, "_read_process_identity", return_value=identity),
+            patch.object(os, "killpg", side_effect=PermissionError("private")),
+        ):
+            with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                acceptance._signal_verified_process(
+                    identity,
+                    working_directory=acceptance.REPOSITORY_ROOT,
+                    requested_signal=signal.SIGTERM,
+                )
+        self.assertEqual(caught.exception.check, "browser-process-signal")
+        self.assertNotIn("private", str(caught.exception))
+
+    def test_close_scans_private_processes_after_unexpected_residue(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            root.chmod(0o700)
+            toolchain = self.fake_toolchain(root)
+            residue = toolchain.daemon_root / "unexpected.session"
+            residue.write_text("{}", encoding="ascii")
+            residue.chmod(0o600)
+            session = acceptance.CliSession(
+                session_name="residue-cleanup", config_path=root / "config.json",
+                working_directory=root, toolchain=toolchain,
+            )
+            with patch.object(
+                acceptance, "_wait_private_processes_gone", return_value={}
+            ) as scan:
+                with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                    session.close()
+            self.assertEqual(caught.exception.check, "daemon-unexpected-residue")
+            self.assertTrue(scan.called)
+            self.assertEqual(list(toolchain.daemon_root.iterdir()), [])
+
+    def test_signal_during_close_is_deferred_until_process_scan_finishes(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            root.chmod(0o700)
+            toolchain = self.fake_toolchain(root)
+            session = acceptance.CliSession(
+                session_name="signal-cleanup", config_path=root / "config.json",
+                working_directory=root, toolchain=toolchain,
+            )
+            previous = acceptance._install_interrupt_handlers()
+
+            def interrupt_cleanup(*args, **kwargs):
+                del args, kwargs
+                os.kill(os.getpid(), signal.SIGTERM)
+
+            try:
+                with (
+                    patch.object(
+                        acceptance, "_remove_daemon_residue",
+                        side_effect=interrupt_cleanup,
+                    ),
+                    patch.object(
+                        acceptance, "_wait_private_processes_gone", return_value={}
+                    ) as scan,
+                ):
+                    with self.assertRaises(acceptance.AcceptanceFailure) as caught:
+                        session.close()
+                self.assertEqual(caught.exception.check, "interrupted")
+                self.assertTrue(scan.called)
+            finally:
+                acceptance._restore_interrupt_handlers(previous)
+                acceptance._SIGNAL_INTERRUPTED = False
+                acceptance._SIGNAL_CLEANUP_DEPTH = 0
+
+    def test_output_and_report_writers_are_private_and_nofollow(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            root.chmod(0o700)
+            path = root / "receipt.json"
+            acceptance.atomic_json(path, {"z": 2, "a": 1})
+            self.assertEqual(path.read_text(encoding="utf-8"), '{"a":1,"z":2}\n')
+            self.assertEqual(stat.S_IMODE(path.stat().st_mode), 0o600)
+            link = root / "linked.json"
+            link.symlink_to(path)
+            with self.assertRaises(acceptance.AcceptanceFailure):
+                acceptance.atomic_json(link, {"a": 2})
+            self.assertEqual(path.read_text(encoding="utf-8"), '{"a":1,"z":2}\n')
+
+    def test_source_has_no_communicate_shell_or_debug_artifact_commands(self):
+        source = Path(acceptance.__file__).read_text(encoding="utf-8")
+        self.assertNotIn(".communicate(", source)
+        self.assertNotIn("shell=True", source)
+        for forbidden in (
+            '"tracing-start"',
+            '"tracing-stop"',
+            '"state-save"',
+            '"state-load"',
+            '"video-start"',
+            '"pdf"',
+        ):
+            self.assertNotIn(forbidden, source)
+        self.assertIn('"snapshot"', source)
+        self.assertIn('"screenshot"', source)
+        self.assertIn('"run-code"', source)
+        self.assertNotIn("--departure-date", source)
+        self.assertNotIn("--return-date", source)
+
+    def test_shell_entrypoint_syntax_mode_and_poisoned_path(self):
+        entrypoint = acceptance.REPOSITORY_ROOT / "scripts" / "check-browser-acceptance"
+        syntax = subprocess.run(
+            ["/bin/sh", "-n", str(entrypoint)],
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+        )
+        self.assertEqual(syntax.returncode, 0, syntax.stderr.decode())
+        source = entrypoint.read_text(encoding="utf-8")
+        self.assertTrue(source.startswith("#!/bin/sh\nPATH=/usr/bin:/bin\nexport PATH\n"))
+        self.assertIn("script_directory=${0%/*}", source)
+        self.assertIn("exec /usr/bin/env -i PATH=/usr/bin:/bin", source)
+        self.assertIn('"${repository_directory}/.venv/bin/python" -I -S -B', source)
+        self.assertNotIn("dirname", source)
+        self.assertNotIn("set -x", source)
+        self.assertEqual(stat.S_IMODE(entrypoint.stat().st_mode), 0o755)
+        self.assertEqual(entrypoint.stat().st_uid, os.getuid())
+        poisoned = {"PATH": "/definitely/not/usable", "DATABASE_URL": "do-not-copy"}
+        help_result = subprocess.run(
+            [str(entrypoint), "--help"],
+            env=poisoned,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+        )
+        self.assertEqual(help_result.returncode, 0, help_result.stderr.decode())
+        self.assertIn(b"--loading-base-url", help_result.stdout)
+
+    def test_output_root_is_ignored(self):
+        completed = subprocess.run(
+            ["git", "check-ignore", "output/playwright/probe.png"],
+            cwd=acceptance.REPOSITORY_ROOT,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            check=False,
+        )
+        self.assertEqual(completed.returncode, 0, completed.stderr.decode())
+
+    def test_module_reload_has_no_filesystem_or_process_side_effect(self):
+        with patch.object(subprocess, "Popen") as popen:
+            importlib.reload(acceptance)
+        popen.assert_not_called()
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/check-browser-acceptance b/scripts/check-browser-acceptance
new file mode 100755
index 0000000..4ea4226
--- /dev/null
+++ b/scripts/check-browser-acceptance
@@ -0,0 +1,27 @@
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
+    printf '%s\n' 'browser_acceptance=failed check=absolute-entrypoint-required' >&2
+    exit 64
+    ;;
+esac
+script_directory=${0%/*}
+repository_directory=${script_directory%/*}
+
+exec /usr/bin/env -i PATH=/usr/bin:/bin LANG=C LC_ALL=C TZ=UTC \
+  "${repository_directory}/.venv/bin/python" -I -S -B \
+  "${repository_directory}/e2e/browser_acceptance.py" "$@"


