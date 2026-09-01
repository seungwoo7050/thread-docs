## `build: automate release assurance`

diff --git a/Makefile b/Makefile
index 53cb194..8551a13 100644
--- a/Makefile
+++ b/Makefile
@@ -1,7 +1,12 @@
 UV_RUN := env UV_CACHE_DIR=.cache/uv UV_TOOL_DIR=.cache/uv-tools UV_PYTHON_INSTALL_DIR=.cache/python uvx --from uv==0.12.6 uv
 PYTHON := .venv/bin/python
 
-.PHONY: check db-up format-check lint migrate migration-check serve sync test type
+# production-check requires explicit DJANGO_DEBUG=0, ADMIN_ENABLED=0,
+# DJANGO_SECRET_KEY, DJANGO_ALLOWED_HOSTS, DJANGO_CSRF_TRUSTED_ORIGINS,
+# DATABASE_URL, and the exact 40-character lowercase release DEPLOY_VERSION.
+# Its secret-check reads the ignored owner-only .env.local in-process; do not export
+# KAMIS_API_KEY into Make, a command argument, or a child-process environment.
+.PHONY: check db-up dependency-audit format-check license-inventory lint migrate migration-check production-check production-env-check secret-check serve sync test type
 
 sync:
 	$(UV_RUN) sync --frozen
@@ -22,7 +27,7 @@ lint:
 	.venv/bin/ruff check .
 
 type:
-	.venv/bin/mypy config grocery manage.py
+	.venv/bin/mypy config grocery scripts manage.py
 
 test:
 	.venv/bin/pytest
@@ -30,5 +35,24 @@ test:
 check: format-check lint type migration-check test
 	$(PYTHON) manage.py check
 
+secret-check:
+	$(PYTHON) -m scripts.secret_check
+
+dependency-audit:
+	.venv/bin/pip-audit --local --progress-spinner off
+
+license-inventory:
+	.venv/bin/pip-licenses --format=plain --with-urls
+
+production-env-check:
+	@case "$${DJANGO_DEBUG:-}" in 0|false|no|off) ;; *) echo "production_check=failed code=debug_must_be_disabled"; exit 2;; esac
+	@test "$${ADMIN_ENABLED:-}" = "0" || { echo "production_check=failed code=admin_must_be_disabled"; exit 2; }
+	@test -n "$${DATABASE_URL:-}" || { echo "production_check=failed code=database_url_required"; exit 2; }
+	@test "$${#DEPLOY_VERSION}" -eq 40 || { echo "production_check=failed code=release_sha_required"; exit 2; }
+	@case "$${DEPLOY_VERSION}" in *[!0-9a-f]*) echo "production_check=failed code=release_sha_required"; exit 2;; esac
+
+production-check: production-env-check check secret-check dependency-audit license-inventory
+	$(PYTHON) manage.py check --deploy
+
 serve:
 	.venv/bin/gunicorn config.wsgi:application --bind 127.0.0.1:8000 --workers 2 --threads 4
diff --git a/scripts/__init__.py b/scripts/__init__.py
new file mode 100644
index 0000000..1c41540
--- /dev/null
+++ b/scripts/__init__.py
@@ -0,0 +1 @@
+"""Local release-assurance commands."""
diff --git a/scripts/secret_check.py b/scripts/secret_check.py
new file mode 100644
index 0000000..ef9f5b6
--- /dev/null
+++ b/scripts/secret_check.py
@@ -0,0 +1,424 @@
+"""Fail-closed local KAMIS credential release assurance.
+
+This command deliberately emits only fixed receipt fields or fixed failure codes.
+Credential material stays in this Python process and is never sent to Git.
+"""
+
+from __future__ import annotations
+
+import os
+import re
+import shutil
+import stat
+import subprocess
+import sys
+from dataclasses import dataclass
+from pathlib import Path
+from typing import Final
+
+from grocery.source.secrets import SecretLoadError, load_kamis_api_key
+
+_SECRET_PATH: Final = Path(".env.local")
+_GIT_TIMEOUT_SECONDS: Final = 30
+_MAX_GIT_METADATA_BYTES: Final = 64 * 1024 * 1024
+_MAX_CURRENT_ENTRIES: Final = 100_000
+_MAX_HISTORY_BLOBS: Final = 250_000
+_MAX_SINGLE_BLOB_BYTES: Final = 64 * 1024 * 1024
+_MAX_TOTAL_SCAN_BYTES: Final = 2 * 1024 * 1024 * 1024
+_READ_CHUNK_BYTES: Final = 64 * 1024
+_OBJECT_ID = re.compile(rb"(?:[0-9a-f]{40}|[0-9a-f]{64})\Z")
+_REGULAR_INDEX_MODES: Final = {b"100644", b"100755"}
+_FIXED_GIT_ENV: Final = {
+    "GIT_CONFIG_NOSYSTEM": "1",
+    "GIT_CONFIG_GLOBAL": "/dev/null",
+    "GIT_CONFIG_SYSTEM": "/dev/null",
+    "GIT_OPTIONAL_LOCKS": "0",
+    "GIT_PAGER": "cat",
+    "GIT_TERMINAL_PROMPT": "0",
+    "LANG": "C",
+    "LC_ALL": "C",
+    "PAGER": "cat",
+    "PATH": os.defpath,
+}
+_LOADER_CODES: Final = frozenset(
+    {
+        "secret_file_changed",
+        "secret_file_invalid_encoding",
+        "secret_file_malformed",
+        "secret_file_missing",
+        "secret_file_not_regular",
+        "secret_file_nul",
+        "secret_file_permissions",
+        "secret_file_symlink",
+        "secret_file_too_large",
+        "secret_file_unreadable",
+        "secret_file_wrong_owner",
+        "secret_key_duplicate",
+        "secret_key_empty",
+        "secret_key_missing",
+        "secret_key_unsafe_syntax",
+    }
+)
+_CHECK_CODES: Final = frozenset(
+    {
+        *_LOADER_CODES,
+        "current_match",
+        "current_scan_budget_exceeded",
+        "current_scan_failed",
+        "git_command_failed",
+        "git_boundary_collision",
+        "git_metadata_invalid",
+        "git_unavailable",
+        "history_match",
+        "history_scan_budget_exceeded",
+        "history_scan_failed",
+        "internal_error",
+        "not_git_repository",
+        "secret_file_not_ignored",
+        "secret_file_tracked",
+        "usage_error",
+    }
+)
+
+
+class SecretCheckError(RuntimeError):
+    """A release-check failure represented by one fixed, non-sensitive code."""
+
+    def __init__(self, code: str) -> None:
+        selected_code = code if code in _CHECK_CODES else "internal_error"
+        self.code = selected_code
+        super().__init__(selected_code)
+
+
+@dataclass(frozen=True, slots=True)
+class SecretCheckReceipt:
+    """Successful fixed receipt; no credential-derived field is permitted."""
+
+    present: str = "yes"
+    ignored: str = "yes"
+    permissions: str = "ok"
+    current_match: str = "no"
+    history_match: str = "no"
+
+    def render(self) -> str:
+        return "\n".join(
+            (
+                f"present={self.present}",
+                f"ignored={self.ignored}",
+                f"permissions={self.permissions}",
+                f"current_match={self.current_match}",
+                f"history_match={self.history_match}",
+            )
+        )
+
+
+def run_secret_check(repository: Path) -> SecretCheckReceipt:
+    """Verify one repository without printing or delegating credential comparison."""
+
+    try:
+        root = repository.resolve(strict=True)
+    except OSError:
+        raise SecretCheckError("not_git_repository") from None
+    try:
+        secret = load_kamis_api_key(environment={}, path=root / _SECRET_PATH)
+    except SecretLoadError as error:
+        raise SecretCheckError(
+            error.code if error.code in _LOADER_CODES else "internal_error"
+        ) from None
+
+    credential = secret.reveal().encode("utf-8")
+    git_executable = _git_executable()
+    _require_repository_root(git_executable, root, credential)
+    _require_ignored_untracked_secret(git_executable, root, credential)
+    if _tracked_worktree_contains(git_executable, root, credential):
+        raise SecretCheckError("current_match")
+    if _object_database_contains(git_executable, root, credential):
+        raise SecretCheckError("history_match")
+    return SecretCheckReceipt()
+
+
+def main(arguments: list[str] | None = None) -> int:
+    selected_arguments = sys.argv[1:] if arguments is None else arguments
+    if selected_arguments:
+        _print_failure("usage_error")
+        return 2
+    try:
+        receipt = run_secret_check(Path.cwd())
+    except SecretCheckError as error:
+        _print_failure(error.code)
+        return 1
+    except Exception:  # noqa: BLE001 - CLI must convert every failure without rendering it.
+        _print_failure("internal_error")
+        return 1
+    print(receipt.render())
+    return 0
+
+
+def _print_failure(code: str) -> None:
+    safe_code = code if code in _CHECK_CODES else "internal_error"
+    print("status=failed")
+    print(f"code={safe_code}")
+
+
+def _git_executable() -> str:
+    executable = shutil.which("git")
+    if executable is None:
+        raise SecretCheckError("git_unavailable")
+    return executable
+
+
+def _run_git(
+    executable: str,
+    repository: Path,
+    arguments: tuple[str, ...],
+    *,
+    credential: bytes,
+    allowed_returncodes: frozenset[int] = frozenset({0}),
+    maximum_output_bytes: int = _MAX_GIT_METADATA_BYTES,
+) -> subprocess.CompletedProcess[bytes]:
+    boundary_values = (
+        os.fsencode(executable),
+        *(os.fsencode(argument) for argument in arguments),
+        *(os.fsencode(key) for key in _FIXED_GIT_ENV),
+        *(os.fsencode(value) for value in _FIXED_GIT_ENV.values()),
+    )
+    if any(credential in value for value in boundary_values):
+        raise SecretCheckError("git_boundary_collision")
+    try:
+        completed = subprocess.run(  # noqa: S603 - executable is resolved, arguments are fixed/OIDs.
+            (executable, *arguments),
+            cwd=repository,
+            env=_FIXED_GIT_ENV,
+            stdin=subprocess.DEVNULL,
+            capture_output=True,
+            check=False,
+            timeout=_GIT_TIMEOUT_SECONDS,
+        )
+    except OSError, subprocess.SubprocessError:
+        raise SecretCheckError("git_command_failed") from None
+    if completed.returncode not in allowed_returncodes:
+        raise SecretCheckError("git_command_failed")
+    if len(completed.stdout) > maximum_output_bytes:
+        raise SecretCheckError("git_metadata_invalid")
+    return completed
+
+
+def _require_repository_root(executable: str, repository: Path, credential: bytes) -> None:
+    completed = _run_git(
+        executable,
+        repository,
+        ("rev-parse", "--show-toplevel"),
+        credential=credential,
+    )
+    try:
+        discovered = Path(os.fsdecode(completed.stdout.rstrip(b"\n"))).resolve(strict=True)
+    except OSError, ValueError:
+        raise SecretCheckError("not_git_repository") from None
+    if discovered != repository:
+        raise SecretCheckError("not_git_repository")
+
+
+def _require_ignored_untracked_secret(
+    executable: str,
+    repository: Path,
+    credential: bytes,
+) -> None:
+    tracked = _run_git(
+        executable,
+        repository,
+        ("ls-files", "--error-unmatch", "--", str(_SECRET_PATH)),
+        credential=credential,
+        allowed_returncodes=frozenset({0, 1}),
+    )
+    if tracked.returncode == 0:
+        raise SecretCheckError("secret_file_tracked")
+    ignored = _run_git(
+        executable,
+        repository,
+        ("check-ignore", "--quiet", "--", str(_SECRET_PATH)),
+        credential=credential,
+        allowed_returncodes=frozenset({0, 1}),
+    )
+    if ignored.returncode != 0:
+        raise SecretCheckError("secret_file_not_ignored")
+
+
+def _tracked_worktree_contains(executable: str, repository: Path, credential: bytes) -> bool:
+    completed = _run_git(
+        executable,
+        repository,
+        ("ls-files", "--stage", "-z"),
+        credential=credential,
+    )
+    records = completed.stdout.split(b"\0")
+    if records and records[-1] == b"":
+        records.pop()
+    if len(records) > _MAX_CURRENT_ENTRIES:
+        raise SecretCheckError("current_scan_budget_exceeded")
+
+    total_bytes = 0
+    for record in records:
+        metadata, separator, path_bytes = record.partition(b"\t")
+        fields = metadata.split(b" ")
+        if separator != b"\t" or len(fields) != 3 or fields[2] != b"0" or not path_bytes:
+            raise SecretCheckError("git_metadata_invalid")
+        mode, object_id, _stage = fields
+        if not _OBJECT_ID.fullmatch(object_id):
+            raise SecretCheckError("git_metadata_invalid")
+        try:
+            relative_path = Path(os.fsdecode(path_bytes))
+        except UnicodeError, ValueError:
+            raise SecretCheckError("git_metadata_invalid") from None
+        if relative_path.is_absolute() or ".." in relative_path.parts:
+            raise SecretCheckError("git_metadata_invalid")
+
+        if mode in _REGULAR_INDEX_MODES:
+            match, scanned_bytes = _regular_file_contains(
+                repository,
+                relative_path,
+                credential,
+            )
+        elif mode == b"120000":
+            match, scanned_bytes = _symlink_target_contains(
+                repository,
+                relative_path,
+                credential,
+            )
+        elif mode == b"160000":
+            match, scanned_bytes = False, 0
+        else:
+            raise SecretCheckError("git_metadata_invalid")
+        total_bytes += scanned_bytes
+        if total_bytes > _MAX_TOTAL_SCAN_BYTES:
+            raise SecretCheckError("current_scan_budget_exceeded")
+        if match:
+            return True
+    return False
+
+
+def _regular_file_contains(
+    repository: Path,
+    relative_path: Path,
+    credential: bytes,
+) -> tuple[bool, int]:
+    path = repository / relative_path
+    flags = os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0)
+    try:
+        descriptor = os.open(path, flags)
+    except FileNotFoundError:
+        return False, 0
+    except OSError:
+        raise SecretCheckError("current_scan_failed") from None
+    try:
+        before = os.fstat(descriptor)
+        if not stat.S_ISREG(before.st_mode):
+            raise SecretCheckError("current_scan_failed")
+        if before.st_size > _MAX_SINGLE_BLOB_BYTES:
+            raise SecretCheckError("current_scan_budget_exceeded")
+        matched = _descriptor_contains(descriptor, credential)
+        after = os.fstat(descriptor)
+        if (
+            before.st_dev,
+            before.st_ino,
+            before.st_size,
+            before.st_mtime_ns,
+        ) != (
+            after.st_dev,
+            after.st_ino,
+            after.st_size,
+            after.st_mtime_ns,
+        ):
+            raise SecretCheckError("current_scan_failed")
+        return matched, before.st_size
+    except SecretCheckError:
+        raise
+    except OSError:
+        raise SecretCheckError("current_scan_failed") from None
+    finally:
+        os.close(descriptor)
+
+
+def _descriptor_contains(descriptor: int, credential: bytes) -> bool:
+    overlap = b""
+    overlap_size = max(len(credential) - 1, 0)
+    while True:
+        chunk = os.read(descriptor, _READ_CHUNK_BYTES)
+        if not chunk:
+            return False
+        combined = overlap + chunk
+        if credential in combined:
+            return True
+        overlap = combined[-overlap_size:] if overlap_size else b""
+
+
+def _symlink_target_contains(
+    repository: Path,
+    relative_path: Path,
+    credential: bytes,
+) -> tuple[bool, int]:
+    path = repository / relative_path
+    try:
+        metadata = path.lstat()
+    except FileNotFoundError:
+        return False, 0
+    except OSError:
+        raise SecretCheckError("current_scan_failed") from None
+    if not stat.S_ISLNK(metadata.st_mode):
+        raise SecretCheckError("current_scan_failed")
+    try:
+        target = os.fsencode(os.readlink(path))
+    except OSError:
+        raise SecretCheckError("current_scan_failed") from None
+    if len(target) > _MAX_SINGLE_BLOB_BYTES:
+        raise SecretCheckError("current_scan_budget_exceeded")
+    return credential in target, len(target)
+
+
+def _object_database_contains(executable: str, repository: Path, credential: bytes) -> bool:
+    completed = _run_git(
+        executable,
+        repository,
+        (
+            "cat-file",
+            "--batch-all-objects",
+            "--batch-check=%(objectname) %(objecttype) %(objectsize)",
+        ),
+        credential=credential,
+    )
+    blobs: list[tuple[str, int]] = []
+    total_bytes = 0
+    for line in completed.stdout.splitlines():
+        fields = line.split(b" ")
+        if len(fields) != 3 or not _OBJECT_ID.fullmatch(fields[0]):
+            raise SecretCheckError("git_metadata_invalid")
+        if fields[1] != b"blob":
+            continue
+        try:
+            object_size = int(fields[2])
+            object_id = fields[0].decode("ascii")
+        except UnicodeError, ValueError:
+            raise SecretCheckError("git_metadata_invalid") from None
+        if object_size < 0 or object_size > _MAX_SINGLE_BLOB_BYTES:
+            raise SecretCheckError("history_scan_budget_exceeded")
+        total_bytes += object_size
+        if len(blobs) >= _MAX_HISTORY_BLOBS or total_bytes > _MAX_TOTAL_SCAN_BYTES:
+            raise SecretCheckError("history_scan_budget_exceeded")
+        blobs.append((object_id, object_size))
+
+    for object_id, expected_size in blobs:
+        blob = _run_git(
+            executable,
+            repository,
+            ("cat-file", "blob", object_id),
+            credential=credential,
+            maximum_output_bytes=_MAX_SINGLE_BLOB_BYTES,
+        ).stdout
+        if len(blob) != expected_size:
+            raise SecretCheckError("history_scan_failed")
+        if credential in blob:
+            return True
+    return False
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/scripts/tests/__init__.py b/scripts/tests/__init__.py
new file mode 100644
index 0000000..a0c50ec
--- /dev/null
+++ b/scripts/tests/__init__.py
@@ -0,0 +1 @@
+"""Tests for local release-assurance commands."""
diff --git a/scripts/tests/test_secret_check.py b/scripts/tests/test_secret_check.py
new file mode 100644
index 0000000..92480d4
--- /dev/null
+++ b/scripts/tests/test_secret_check.py
@@ -0,0 +1,201 @@
+from __future__ import annotations
+
+import os
+import shutil
+import subprocess
+from collections.abc import Callable
+from pathlib import Path
+from typing import Any
+
+import pytest
+
+from scripts.secret_check import SecretCheckError, main, run_secret_check
+
+_TEST_GIT_ENV = {
+    "GIT_AUTHOR_EMAIL": "release-check@example.invalid",
+    "GIT_AUTHOR_NAME": "Release Check",
+    "GIT_COMMITTER_EMAIL": "release-check@example.invalid",
+    "GIT_COMMITTER_NAME": "Release Check",
+    "GIT_CONFIG_GLOBAL": "/dev/null",
+    "GIT_CONFIG_NOSYSTEM": "1",
+    "GIT_CONFIG_SYSTEM": "/dev/null",
+    "LANG": "C",
+    "LC_ALL": "C",
+    "PATH": os.defpath,
+}
+
+
+def git(repository: Path, *arguments: str) -> None:
+    executable = shutil.which("git")
+    assert executable is not None
+    subprocess.run(  # noqa: S603 - executable is resolved for a synthetic repository.
+        (executable, *arguments),
+        cwd=repository,
+        env=_TEST_GIT_ENV,
+        stdin=subprocess.DEVNULL,
+        capture_output=True,
+        check=True,
+    )
+
+
+def synthetic_secret() -> str:
+    return "synthetic-release-credential-" + "q" * 23
+
+
+def write_secret(repository: Path, value: str, *, mode: int = 0o600) -> Path:
+    secret_path = repository / ".env.local"
+    secret_path.write_text(f"KAMIS_API_KEY={value}\n", encoding="utf-8")
+    secret_path.chmod(mode)
+    return secret_path
+
+
+def repository(tmp_path: Path, secret: str) -> Path:
+    root = tmp_path / "synthetic-repository"
+    root.mkdir()
+    git(root, "init", "--quiet")
+    (root / ".gitignore").write_text(".env.local\n", encoding="utf-8")
+    (root / "tracked.txt").write_text("safe tracked content\n", encoding="utf-8")
+    git(root, "add", ".gitignore", "tracked.txt")
+    git(root, "commit", "--quiet", "-m", "initial")
+    write_secret(root, secret)
+    return root
+
+
+def test_success_emits_only_fixed_receipt_fields(tmp_path: Path) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+
+    receipt = run_secret_check(root)
+
+    assert receipt.render().splitlines() == [
+        "present=yes",
+        "ignored=yes",
+        "permissions=ok",
+        "current_match=no",
+        "history_match=no",
+    ]
+    assert secret not in receipt.render()
+    assert str(len(secret)) not in receipt.render()
+
+
+def test_uncommitted_tracked_leak_fails_without_filename_or_value(
+    tmp_path: Path,
+    monkeypatch: pytest.MonkeyPatch,
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+    leak_name = "tracked.txt"
+    (root / leak_name).write_text(f"prefix:{secret}:suffix", encoding="utf-8")
+
+    monkeypatch.chdir(root)
+    exit_code = main([])
+    output = capsys.readouterr().out
+
+    assert exit_code == 1
+    assert output == "status=failed\ncode=current_match\n"
+    assert secret not in output
+    assert str(len(secret)) not in output
+    assert leak_name not in output
+
+
+def test_historical_only_leak_is_detected_and_redacted(
+    tmp_path: Path,
+    monkeypatch: pytest.MonkeyPatch,
+    capsys: pytest.CaptureFixture[str],
+) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+    leak_name = "historical-only.txt"
+    leaked = root / leak_name
+    leaked.write_text(secret, encoding="utf-8")
+    git(root, "add", leak_name)
+    git(root, "commit", "--quiet", "-m", "historical leak")
+    leaked.write_text("replaced with safe bytes\n", encoding="utf-8")
+    git(root, "add", leak_name)
+    git(root, "commit", "--quiet", "-m", "remove leak from current tree")
+
+    monkeypatch.chdir(root)
+    exit_code = main([])
+    output = capsys.readouterr().out
+
+    assert exit_code == 1
+    assert output == "status=failed\ncode=history_match\n"
+    assert secret not in output
+    assert str(len(secret)) not in output
+    assert leak_name not in output
+
+
+def test_secret_is_never_sent_to_git_args_environment_or_stdin(
+    tmp_path: Path,
+    monkeypatch: pytest.MonkeyPatch,
+) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+    real_run = subprocess.run
+
+    def audited_run(*args: Any, **kwargs: Any) -> subprocess.CompletedProcess[bytes]:
+        command = args[0]
+        environment = kwargs.get("env", {})
+        stdin_value = kwargs.get("input")
+        assert secret not in repr(command)
+        assert all(secret not in str(value) for value in environment.values())
+        assert stdin_value is None or secret.encode() not in stdin_value
+        return real_run(*args, **kwargs)
+
+    monkeypatch.setattr("scripts.secret_check.subprocess.run", audited_run)
+
+    run_secret_check(root)
+
+
+def test_fixed_git_boundary_collision_fails_before_spawning_child(tmp_path: Path) -> None:
+    root = repository(tmp_path, "C")
+
+    with pytest.raises(SecretCheckError) as caught:
+        run_secret_check(root)
+
+    assert caught.value.code == "git_boundary_collision"
+
+
+@pytest.mark.parametrize(
+    ("arrange", "code"),
+    [
+        (
+            lambda root: (root / ".gitignore").write_text("", encoding="utf-8"),
+            "secret_file_not_ignored",
+        ),
+        (lambda root: (root / ".env.local").chmod(0o640), "secret_file_permissions"),
+    ],
+)
+def test_prerequisite_failure_uses_fixed_code(
+    tmp_path: Path,
+    arrange: Callable[[Path], object],
+    code: str,
+) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+    arrange(root)
+
+    with pytest.raises(SecretCheckError) as caught:
+        run_secret_check(root)
+
+    assert caught.value.code == code
+    assert str(caught.value) == code
+    assert secret not in repr(caught.value)
+
+
+def test_secret_file_symlink_is_rejected_without_following_target(tmp_path: Path) -> None:
+    secret = synthetic_secret()
+    root = repository(tmp_path, secret)
+    secret_path = root / ".env.local"
+    target = root / "outside-secret"
+    target.write_text(f"KAMIS_API_KEY={secret}\n", encoding="utf-8")
+    target.chmod(0o600)
+    secret_path.unlink()
+    secret_path.symlink_to(target)
+
+    with pytest.raises(SecretCheckError) as caught:
+        run_secret_check(root)
+
+    assert caught.value.code == "secret_file_symlink"
+    assert secret not in repr(caught.value)


