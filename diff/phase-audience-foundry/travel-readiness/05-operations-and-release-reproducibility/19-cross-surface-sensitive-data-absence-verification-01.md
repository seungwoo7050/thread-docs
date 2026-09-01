# Git·DB·산출물·런타임을 가로지르는 민감정보 부재 검증

## `ops: add silent sensitive-absence release gate`

diff --git a/operations/release_manifest_validation.py b/operations/release_manifest_validation.py
new file mode 100644
index 0000000..07a7500
--- /dev/null
+++ b/operations/release_manifest_validation.py
@@ -0,0 +1,46 @@
+#!/usr/bin/env python3
+"""Silent bridge to the authoritative release-manifest validator."""
+
+from __future__ import annotations
+
+import sys
+
+
+def main() -> int:
+    if not (
+        sys.flags.isolated
+        and sys.flags.no_site
+        and sys.flags.dont_write_bytecode
+    ):
+        return 1
+    try:
+        from importlib.machinery import SourceFileLoader
+        from importlib.util import module_from_spec, spec_from_loader
+        from pathlib import Path
+
+        repository = Path(__file__).resolve().parents[1]
+        verifier_path = repository / "scripts" / "verify-release-runtime"
+        status = verifier_path.lstat()
+        if not verifier_path.is_file() or verifier_path.is_symlink():
+            return 1
+        if status.st_size > 4 * 1024 * 1024:
+            return 1
+        data = sys.stdin.buffer.read(16 * 1024 * 1024 + 1)
+        if not data or len(data) > 16 * 1024 * 1024:
+            return 1
+        loader = SourceFileLoader(
+            "_sensitive_absence_release_verifier", str(verifier_path)
+        )
+        specification = spec_from_loader(loader.name, loader)
+        if specification is None:
+            return 1
+        module = module_from_spec(specification)
+        loader.exec_module(module)
+        module.validate_manifest(data)
+        return 0
+    except BaseException:
+        return 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/operations/sensitive_absence.py b/operations/sensitive_absence.py
new file mode 100644
index 0000000..ad747d7
--- /dev/null
+++ b/operations/sensitive_absence.py
@@ -0,0 +1,1662 @@
+"""Bounded, silent absence scans for release-candidate evidence.
+
+The target values are supplied by the CLI after it has removed them from the
+process environment.  This module never reads dotenv files and never exposes a
+path, value, match, fragment, SQL result, or exception in a receipt.
+"""
+
+from __future__ import annotations
+
+from dataclasses import dataclass
+import hashlib
+import json
+import os
+from pathlib import Path, PurePosixPath
+import re
+import stat
+import subprocess
+import tempfile
+from typing import Callable, Final, Mapping, Sequence
+from urllib.parse import quote, quote_plus, unquote, unquote_plus
+
+
+REPOSITORY_ROOT: Final = Path(__file__).resolve().parents[1]
+GIT: Final = Path("/usr/bin/git")
+PYTHON: Final = REPOSITORY_ROOT / ".venv" / "bin" / "python"
+MANIFEST_VALIDATOR: Final = (
+    REPOSITORY_ROOT / "operations" / "release_manifest_validation.py"
+)
+CHUNK_BYTES: Final = 64 * 1024
+MAX_PATTERN_BYTES: Final = 16 * 1024
+FRAGMENT_BYTES: Final = 24
+MAX_FRAGMENT_PATTERNS: Final = 500_000
+MAX_FILE_BYTES: Final = 256 * 1024 * 1024
+MAX_TREE_BYTES: Final = 4 * 1024 * 1024 * 1024
+MAX_TREE_ENTRIES: Final = 100_000
+MAX_GIT_BLOB_BYTES: Final = 512 * 1024 * 1024
+MAX_GIT_METADATA_BYTES: Final = 16 * 1024 * 1024
+MAX_GIT_BLOBS: Final = 1_000_000
+MAX_GIT_COMMITS: Final = 100_000
+MAX_DB_COLUMNS: Final = 10_000
+MAX_DB_VALUE_BYTES: Final = 32 * 1024 * 1024
+MAX_MANIFEST_BYTES: Final = 16 * 1024 * 1024
+SAFE_SHA = re.compile(r"\A[0-9a-f]{40}\Z")
+SAFE_SHA256 = re.compile(r"\A[0-9a-f]{64}\Z")
+SAFE_IDENTIFIER = re.compile(r"\A[a-z_][a-z0-9_]{0,62}\Z")
+SAFE_RESTORE_DATABASE = re.compile(
+    r"\Atravel_readiness_restore_[a-z0-9]{6,32}\Z"
+)
+RECEIPT_ORDER: Final = ("git", "db", "artifact", "runtime")
+FORBIDDEN_ENV_BASENAMES: Final = {".env", ".env.local"}
+BACKUP_INTEGRITY_KEYS: Final = frozenset(
+    {
+        "postgresql.version_num",
+        "schema.columns.sha256",
+        "schema.constraints.sha256",
+        "schema.indexes.sha256",
+        "schema.trigger_functions.sha256",
+        "schema.triggers.sha256",
+        "database.locale_profile.sha256",
+        "schema.sequences.sha256",
+        "schema.sequence_state.sha256",
+        "table.auth_group.count",
+        "table.auth_group.sha256",
+        "table.auth_group_permissions.count",
+        "table.auth_group_permissions.sha256",
+        "table.auth_permission.count",
+        "table.auth_permission.sha256",
+        "table.auth_user.count",
+        "table.auth_user.sha256",
+        "table.auth_user_groups.count",
+        "table.auth_user_groups.sha256",
+        "table.auth_user_user_permissions.count",
+        "table.auth_user_user_permissions.sha256",
+        "table.django_content_type.count",
+        "table.django_content_type.sha256",
+        "table.django_migrations.count",
+        "table.django_migrations.sha256",
+        "table.countries_country.count",
+        "table.countries_country.sha256",
+        "table.sources_sourceconfiguration.count",
+        "table.sources_sourceconfiguration.sha256",
+        "table.sources_sourcerightsdecision.count",
+        "table.sources_sourcerightsdecision.sha256",
+        "table.sources_fetchattempt.count",
+        "table.sources_fetchattempt.sha256",
+        "table.sources_sourceartifact.count",
+        "table.sources_sourceartifact.sha256",
+        "table.sources_parserun.count",
+        "table.sources_parserun.sha256",
+        "table.entry_requirements_passportpolicy.count",
+        "table.entry_requirements_passportpolicy.sha256",
+        "table.entry_requirements_entryfactrevision.count",
+        "table.entry_requirements_entryfactrevision.sha256",
+        "table.travel_warnings_travelwarningrevision.count",
+        "table.travel_warnings_travelwarningrevision.sha256",
+        "table.reviews_reviewdecision.count",
+        "table.reviews_reviewdecision.sha256",
+        "table.reviews_publicationrevision.count",
+        "table.reviews_publicationrevision.sha256",
+        "table.reviews_publishedentryfacts.count",
+        "table.reviews_publishedentryfacts.sha256",
+        "table.reviews_publishedtravelwarning.count",
+        "table.reviews_publishedtravelwarning.sha256",
+        "table.reviews_auditevent.count",
+        "table.reviews_auditevent.sha256",
+        "pointer.entry",
+        "pointer.travel_warning",
+    }
+)
+
+
+class ScanFailure(RuntimeError):
+    """Intentionally carries no source exception or sensitive detail."""
+
+
+class ArgumentFailure(ScanFailure):
+    pass
+
+
+@dataclass(frozen=True, slots=True)
+class ScanConfiguration:
+    repository_root: Path
+    release_sha: str
+    git_safety_token: str
+    artifact_root: Path
+    artifact_safety_token: str
+    runtime_roots: tuple[Path, ...]
+    runtime_files: tuple[Path, ...]
+    backup_directory: Path
+    runtime_safety_token: str
+    db_host: str
+    db_port: int
+    main_db_user: str
+    main_db_password_env: str
+    restored_db_user: str
+    restored_db_password_env: str
+    main_database: str
+    restored_database: str
+    db_safety_token: str
+
+
+def fixed_receipt(statuses: Mapping[str, bool]) -> str:
+    if set(statuses) != set(RECEIPT_ORDER):
+        statuses = {name: False for name in RECEIPT_ORDER}
+    return " ".join(
+        f"{name}={'ok' if statuses[name] is True else 'failed'}"
+        for name in RECEIPT_ORDER
+    )
+
+
+def _scope_digest(items: Sequence[tuple[str, str]]) -> str:
+    encoded = json.dumps(
+        list(items),
+        ensure_ascii=False,
+        allow_nan=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(encoded).hexdigest()
+
+
+def expected_artifact_safety_token(release_sha: str, root: Path) -> str:
+    digest = _scope_digest((("artifact", str(root)),))
+    return f"SENSITIVE_ABSENCE_ARTIFACT_READ_ONLY:{release_sha}:{digest}"
+
+
+def expected_runtime_safety_token(
+    release_sha: str,
+    roots: Sequence[Path],
+    files: Sequence[Path],
+    backup_directory: Path,
+) -> str:
+    items = [
+        *(("runtime-root", str(path)) for path in roots),
+        *(("runtime-file", str(path)) for path in files),
+        ("backup-directory", str(backup_directory)),
+    ]
+    digest = _scope_digest(items)
+    return f"SENSITIVE_ABSENCE_RUNTIME_READ_ONLY:{release_sha}:{digest}"
+
+
+def expected_db_safety_token(
+    configuration: ScanConfiguration, archive_sha256: str
+) -> str:
+    if SAFE_SHA256.fullmatch(archive_sha256) is None:
+        raise ScanFailure()
+    digest = _scope_digest(
+        (
+            ("host", configuration.db_host),
+            ("port", str(configuration.db_port)),
+            ("main-user", configuration.main_db_user),
+            ("main-database", configuration.main_database),
+            ("restored-user", configuration.restored_db_user),
+            ("restored-database", configuration.restored_database),
+            ("archive-sha256", archive_sha256),
+        )
+    )
+    return (
+        "SENSITIVE_ABSENCE_DB_READ_ONLY:"
+        f"{configuration.release_sha}:{digest}"
+    )
+
+
+def _safe_text(value: object, *, minimum: int, maximum: int) -> str:
+    if not isinstance(value, str) or not minimum <= len(value) <= maximum:
+        raise ScanFailure()
+    if any(ord(character) < 32 or ord(character) == 127 for character in value):
+        raise ScanFailure()
+    return value
+
+
+def parse_marker_array(raw: object) -> tuple[str, ...]:
+    raw_text = _safe_text(raw, minimum=4, maximum=65_536)
+    try:
+        value = json.loads(raw_text)
+    except (TypeError, ValueError) as exc:
+        raise ScanFailure() from exc
+    if (
+        not isinstance(value, list)
+        or not 1 <= len(value) <= 32
+        or any(not isinstance(item, str) for item in value)
+    ):
+        raise ScanFailure()
+    markers = tuple(
+        _safe_text(item, minimum=24, maximum=4_096) for item in value
+    )
+    if len(set(markers)) != len(markers):
+        raise ScanFailure()
+    return markers
+
+
+def _encoded_variants(value: str) -> set[bytes]:
+    decoded = {value, unquote(value), unquote_plus(value)}
+    text_variants: set[str] = set(decoded)
+    for item in tuple(decoded):
+        text_variants.add(quote(item, safe=""))
+        text_variants.add(quote_plus(item, safe=""))
+        for quoted in (quote(item, safe=""), quote_plus(item, safe="")):
+            text_variants.add(
+                re.sub(
+                    r"%[0-9A-F]{2}",
+                    lambda match: match.group(0).lower(),
+                    quoted,
+                )
+            )
+        for ensure_ascii in (False, True):
+            encoded = json.dumps(item, ensure_ascii=ensure_ascii)
+            text_variants.add(encoded)
+            text_variants.add(encoded[1:-1])
+    variants: set[bytes] = set()
+    for item in text_variants:
+        payload = item.encode("utf-8")
+        if payload:
+            variants.add(payload)
+    return variants
+
+
+class SensitivePatterns:
+    """Categorized byte patterns with cross-chunk matching."""
+
+    def __init__(
+        self,
+        secret: str,
+        trip_markers: Sequence[str],
+        raw_markers: Sequence[str],
+    ) -> None:
+        secret = _safe_text(secret, minimum=32, maximum=4_096)
+        categories = {
+            "secret": (secret,),
+            "trip": tuple(trip_markers),
+            "raw": tuple(raw_markers),
+        }
+        built: dict[str, set[bytes]] = {}
+        fragments: dict[bytes, set[str]] = {}
+        for category, values in categories.items():
+            if not values:
+                raise ScanFailure()
+            variants: set[bytes] = set()
+            for value in values:
+                _safe_text(
+                    value,
+                    minimum=24 if category != "secret" else 32,
+                    maximum=4_096,
+                )
+                for encoded in _encoded_variants(value):
+                    if len(encoded) > MAX_PATTERN_BYTES:
+                        raise ScanFailure()
+                    variants.add(encoded)
+                    for index in range(
+                        max(0, len(encoded) - FRAGMENT_BYTES + 1)
+                    ):
+                        fragment = encoded[index : index + FRAGMENT_BYTES]
+                        fragments.setdefault(fragment, set()).add(category)
+                        if len(fragments) > MAX_FRAGMENT_PATTERNS:
+                            raise ScanFailure()
+            if not variants or any(len(item) < 16 for item in variants):
+                raise ScanFailure()
+            built[category] = variants
+        all_patterns: dict[bytes, set[str]] = {}
+        for category, patterns in built.items():
+            for pattern in patterns:
+                all_patterns.setdefault(pattern, set()).add(category)
+        self.by_category = {
+            category: tuple(sorted(patterns, key=lambda item: (len(item), item)))
+            for category, patterns in built.items()
+        }
+        self.patterns = tuple(
+            sorted(all_patterns, key=lambda item: (len(item), item))
+        )
+        self.categories = {
+            pattern: frozenset(all_patterns[pattern]) for pattern in self.patterns
+        }
+        self.fragment_categories = {
+            fragment: frozenset(categories)
+            for fragment, categories in fragments.items()
+        }
+        self.overlap = max(
+            max(len(pattern) for pattern in self.patterns) - 1,
+            FRAGMENT_BYTES - 1,
+        )
+
+    def matches(self, payload: bytes) -> frozenset[str]:
+        categories: set[str] = set()
+        normalized = re.sub(
+            rb"%[0-9a-fA-F]{2}",
+            lambda match: match.group(0).upper(),
+            payload,
+        )
+        candidates = (payload,) if normalized == payload else (payload, normalized)
+        for candidate in candidates:
+            for pattern in self.patterns:
+                if pattern in candidate:
+                    categories.update(self.categories[pattern])
+            for index in range(
+                max(0, len(candidate) - FRAGMENT_BYTES + 1)
+            ):
+                fragment_categories = self.fragment_categories.get(
+                    candidate[index : index + FRAGMENT_BYTES]
+                )
+                if fragment_categories:
+                    categories.update(fragment_categories)
+        return frozenset(categories)
+
+    def stream_matches(
+        self, reader: Callable[[int], bytes], total_bytes: int
+    ) -> frozenset[str]:
+        if not 0 <= total_bytes <= MAX_FILE_BYTES:
+            raise ScanFailure()
+        remaining = total_bytes
+        tail = b""
+        found: set[str] = set()
+        while remaining:
+            chunk = reader(min(CHUNK_BYTES, remaining))
+            if not chunk or len(chunk) > remaining:
+                raise ScanFailure()
+            window = tail + chunk
+            found.update(self.matches(window))
+            if found:
+                return frozenset(found)
+            tail = window[-self.overlap :] if self.overlap else b""
+            remaining -= len(chunk)
+        if reader(1):
+            raise ScanFailure()
+        return frozenset()
+
+
+def _is_forbidden_env_name(name: str) -> bool:
+    return name in FORBIDDEN_ENV_BASENAMES or (
+        name.startswith(".env.") and name != ".env.example"
+    )
+
+
+def _safe_posix_path(value: object) -> str:
+    if not isinstance(value, str) or not value or len(value) > 1_024:
+        raise ScanFailure()
+    path = PurePosixPath(value)
+    if (
+        path.is_absolute()
+        or not path.parts
+        or value != path.as_posix()
+        or any(
+            part in {"", ".", ".."} or len(part) > 255
+            for part in path.parts
+        )
+        or path.parts[0] == ".venv"
+    ):
+        raise ScanFailure()
+    if any(_is_forbidden_env_name(part) for part in path.parts):
+        raise ScanFailure()
+    return path.as_posix()
+
+
+def _fixed_git_environment() -> dict[str, str]:
+    return {
+        "PATH": "/usr/bin:/bin",
+        "LANG": "C",
+        "LC_ALL": "C",
+        "TZ": "UTC",
+        "GIT_CONFIG_NOSYSTEM": "1",
+        "GIT_CONFIG_GLOBAL": "/dev/null",
+        "GIT_CONFIG_SYSTEM": "/dev/null",
+        "GIT_NO_LAZY_FETCH": "1",
+        "GIT_OPTIONAL_LOCKS": "0",
+        "GIT_TERMINAL_PROMPT": "0",
+    }
+
+
+def _authoritative_manifest_valid(data: bytes) -> bool:
+    try:
+        process = subprocess.Popen(
+            [
+                str(PYTHON),
+                "-I",
+                "-S",
+                "-B",
+                str(MANIFEST_VALIDATOR),
+            ],
+            cwd=REPOSITORY_ROOT,
+            env=_fixed_git_environment(),
+            stdin=subprocess.PIPE,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+        )
+        process.communicate(input=data, timeout=15)
+        return process.returncode == 0
+    except subprocess.TimeoutExpired:
+        process.kill()
+        process.communicate(timeout=5)
+        return False
+    except (OSError, ValueError):
+        return False
+
+
+def _git_process(arguments: Sequence[str], root: Path) -> subprocess.Popen[bytes]:
+    try:
+        return subprocess.Popen(
+            [str(GIT), *arguments],
+            cwd=root,
+            env=_fixed_git_environment(),
+            stdin=(
+                subprocess.PIPE
+                if arguments[:2] == ["cat-file", "--batch"]
+                else subprocess.DEVNULL
+            ),
+            stdout=subprocess.PIPE,
+            stderr=subprocess.DEVNULL,
+        )
+    except OSError as exc:
+        raise ScanFailure() from exc
+
+
+def _finish_process(process: subprocess.Popen[bytes], *, expected: int = 0) -> None:
+    try:
+        status = process.wait(timeout=10)
+    except subprocess.TimeoutExpired:
+        process.kill()
+        process.wait(timeout=5)
+        raise ScanFailure()
+    if status != expected:
+        raise ScanFailure()
+
+
+def _close_process_pipes(process: subprocess.Popen[bytes]) -> None:
+    for stream in (process.stdin, process.stdout):
+        if stream is not None and not stream.closed:
+            try:
+                stream.close()
+            except OSError:
+                pass
+
+
+def _git_lines(
+    root: Path, arguments: Sequence[str], *, maximum: int
+) -> list[bytes]:
+    process = _git_process(arguments, root)
+    if process.stdout is None:
+        raise ScanFailure()
+    records: list[bytes] = []
+    try:
+        while True:
+            line = process.stdout.readline(512)
+            if not line:
+                break
+            if len(line) >= 512 or not line.endswith(b"\n"):
+                raise ScanFailure()
+            value = line[:-1]
+            if value:
+                records.append(value)
+                if len(records) > maximum:
+                    raise ScanFailure()
+        _finish_process(process)
+    except BaseException:
+        if process.poll() is None:
+            process.kill()
+            process.wait(timeout=5)
+        raise
+    finally:
+        _close_process_pipes(process)
+    return records
+
+
+def _git_has_forbidden_environment_path(root: Path, commits: Sequence[bytes]) -> bool:
+    for commit in commits:
+        if not re.fullmatch(rb"[0-9a-f]{40}", commit):
+            raise ScanFailure()
+        process = _git_process(
+            ["ls-tree", "-r", "-z", "--name-only", commit.decode("ascii")],
+            root,
+        )
+        if process.stdout is None:
+            raise ScanFailure()
+        buffered = b""
+        entry_count = 0
+        try:
+            while True:
+                chunk = process.stdout.read(CHUNK_BYTES)
+                if not chunk:
+                    break
+                buffered += chunk
+                if len(buffered) > 2 * CHUNK_BYTES:
+                    raise ScanFailure()
+                parts = buffered.split(b"\0")
+                buffered = parts.pop()
+                for raw_path in parts:
+                    entry_count += 1
+                    if entry_count > MAX_TREE_ENTRIES:
+                        raise ScanFailure()
+                    try:
+                        path = raw_path.decode("utf-8", "strict")
+                    except UnicodeError as exc:
+                        raise ScanFailure() from exc
+                    if any(
+                        _is_forbidden_env_name(part)
+                        for part in PurePosixPath(path).parts
+                    ):
+                        process.kill()
+                        process.wait(timeout=5)
+                        return True
+            if buffered:
+                raise ScanFailure()
+            _finish_process(process)
+        except BaseException:
+            if process.poll() is None:
+                process.kill()
+                process.wait(timeout=5)
+            raise
+        finally:
+            _close_process_pipes(process)
+    return False
+
+
+def _git_head(root: Path) -> str:
+    values = _git_lines(
+        root, ["rev-parse", "--verify", "HEAD^{commit}"], maximum=1
+    )
+    if len(values) != 1 or not re.fullmatch(rb"[0-9a-f]{40}", values[0]):
+        raise ScanFailure()
+    return values[0].decode("ascii")
+
+
+def _git_worktree_clean(root: Path) -> bool:
+    process = _git_process(
+        ["status", "--porcelain=v1", "--untracked-files=all", "-z"], root
+    )
+    if process.stdout is None:
+        raise ScanFailure()
+    try:
+        first = process.stdout.read(1)
+        if first:
+            process.kill()
+            process.wait(timeout=5)
+            return False
+        _finish_process(process)
+        return True
+    except BaseException:
+        if process.poll() is None:
+            process.kill()
+            process.wait(timeout=5)
+        raise
+    finally:
+        _close_process_pipes(process)
+
+
+def _git_reference_snapshot(root: Path) -> tuple[list[bytes], list[bytes]]:
+    references = _git_lines(
+        root, ["for-each-ref", "--format=%(refname)"], maximum=MAX_GIT_COMMITS
+    )
+    reflog_records = _git_lines(
+        root,
+        ["log", "-g", "--all", "--format=%gD%x00%gs"],
+        maximum=MAX_GIT_COMMITS,
+    )
+    return references, reflog_records
+
+
+def scan_git_reachable(
+    root: Path,
+    release_sha: str,
+    safety_token: str,
+    patterns: SensitivePatterns,
+) -> bool:
+    if (
+        _git_head(root) != release_sha
+        or safety_token != f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}"
+    ):
+        return False
+    if not _git_worktree_clean(root) or _git_lines(
+        root, ["remote"], maximum=1
+    ):
+        return False
+    references, reflog_records = _git_reference_snapshot(root)
+    if not references or any(
+        patterns.matches(value) for value in (*references, *reflog_records)
+    ):
+        return False
+    commits = _git_lines(
+        root, ["rev-list", "--all", "--reflog"], maximum=MAX_GIT_COMMITS
+    )
+    if not commits or _git_has_forbidden_environment_path(root, commits):
+        return False
+    object_ids = _git_lines(
+        root,
+        [
+            "rev-list",
+            "--objects",
+            "--all",
+            "--reflog",
+            "--no-object-names",
+        ],
+        maximum=MAX_GIT_BLOBS,
+    )
+    if not object_ids:
+        return False
+    process = _git_process(["cat-file", "--batch"], root)
+    if process.stdin is None or process.stdout is None:
+        return False
+    seen: set[bytes] = set()
+    blob_count = 0
+    try:
+        for object_id in object_ids:
+            if object_id in seen:
+                continue
+            seen.add(object_id)
+            if not re.fullmatch(rb"[0-9a-f]{40}", object_id):
+                raise ScanFailure()
+            process.stdin.write(object_id + b"\n")
+            process.stdin.flush()
+            header = process.stdout.readline(256)
+            match = re.fullmatch(
+                rb"([0-9a-f]{40}) (blob|commit|tree|tag) ([0-9]{1,12})\n",
+                header,
+            )
+            if match is None or match.group(1) != object_id:
+                raise ScanFailure()
+            object_type = match.group(2)
+            size = int(match.group(3))
+            maximum = (
+                MAX_GIT_BLOB_BYTES
+                if object_type == b"blob"
+                else MAX_GIT_METADATA_BYTES
+            )
+            if size > maximum:
+                raise ScanFailure()
+            if object_type == b"blob":
+                blob_count += 1
+            # Git adds one framing newline after the exact blob body.
+            remaining = size
+            tail = b""
+            while remaining:
+                chunk = process.stdout.read(min(CHUNK_BYTES, remaining))
+                if not chunk:
+                    raise ScanFailure()
+                window = tail + chunk
+                if patterns.matches(window):
+                    process.kill()
+                    process.wait(timeout=5)
+                    return False
+                tail = window[-patterns.overlap :] if patterns.overlap else b""
+                remaining -= len(chunk)
+            if process.stdout.read(1) != b"\n":
+                raise ScanFailure()
+        process.stdin.close()
+        _finish_process(process)
+    except BaseException:
+        if process.poll() is None:
+            process.kill()
+            process.wait(timeout=5)
+        raise
+    finally:
+        _close_process_pipes(process)
+    if blob_count == 0 or _git_head(root) != release_sha:
+        return False
+    final_commits = _git_lines(
+        root, ["rev-list", "--all", "--reflog"], maximum=MAX_GIT_COMMITS
+    )
+    final_objects = _git_lines(
+        root,
+        [
+            "rev-list",
+            "--objects",
+            "--all",
+            "--reflog",
+            "--no-object-names",
+        ],
+        maximum=MAX_GIT_BLOBS,
+    )
+    final_references, final_reflog_records = _git_reference_snapshot(root)
+    return (
+        final_commits == commits
+        and final_objects == object_ids
+        and final_references == references
+        and final_reflog_records == reflog_records
+        and _git_worktree_clean(root)
+        and not _git_lines(root, ["remote"], maximum=1)
+    )
+
+
+def _safe_open_file(path: Path) -> tuple[object, os.stat_result]:
+    try:
+        before = path.lstat()
+        descriptor = os.open(path, os.O_RDONLY | getattr(os, "O_NOFOLLOW", 0))
+        observed = os.fstat(descriptor)
+    except OSError as exc:
+        raise ScanFailure() from exc
+    if (
+        not stat.S_ISREG(before.st_mode)
+        or stat.S_ISLNK(before.st_mode)
+        or not stat.S_ISREG(observed.st_mode)
+        or (before.st_dev, before.st_ino, before.st_size)
+        != (observed.st_dev, observed.st_ino, observed.st_size)
+        or before.st_size > MAX_FILE_BYTES
+    ):
+        os.close(descriptor)
+        raise ScanFailure()
+    return os.fdopen(descriptor, "rb"), before
+
+
+def scan_regular_file(
+    path: Path,
+    patterns: SensitivePatterns,
+    *,
+    expected_sha256: str | None = None,
+) -> tuple[bool, str]:
+    handle, before = _safe_open_file(path)
+    digest = hashlib.sha256()
+    tail = b""
+    found = False
+    try:
+        remaining = before.st_size
+        while remaining:
+            chunk = handle.read(min(CHUNK_BYTES, remaining))
+            if not chunk:
+                raise ScanFailure()
+            digest.update(chunk)
+            if not found:
+                window = tail + chunk
+                found = bool(patterns.matches(window))
+                tail = window[-patterns.overlap :] if patterns.overlap else b""
+            remaining -= len(chunk)
+        if handle.read(1):
+            raise ScanFailure()
+        after = os.fstat(handle.fileno())
+    finally:
+        handle.close()
+    if (
+        (before.st_dev, before.st_ino, before.st_size, before.st_mtime_ns)
+        != (after.st_dev, after.st_ino, after.st_size, after.st_mtime_ns)
+    ):
+        raise ScanFailure()
+    hexdigest = digest.hexdigest()
+    if expected_sha256 is not None and hexdigest != expected_sha256:
+        return False, hexdigest
+    return not found, hexdigest
+
+
+def _manifest_entry_map(
+    manifest: object, release_sha: str
+) -> dict[str, tuple[str, str]]:
+    if not isinstance(manifest, dict) or set(manifest) != {
+        "archive",
+        "dependencies",
+        "git_object_format",
+        "migrations",
+        "release_sha",
+        "runtime",
+        "schema_version",
+        "source",
+        "static",
+    }:
+        raise ScanFailure()
+    if (
+        manifest.get("schema_version") != 1
+        or manifest.get("release_sha") != release_sha
+        or manifest.get("git_object_format") != "sha1"
+    ):
+        raise ScanFailure()
+    source = manifest.get("source")
+    static_section = manifest.get("static")
+    if (
+        not isinstance(source, dict)
+        or set(source) != {"tracked", "tracked_set_sha256"}
+        or not isinstance(source.get("tracked"), list)
+        or not isinstance(static_section, dict)
+        or set(static_section)
+        != {
+            "collected",
+            "collected_set_sha256",
+            "origins",
+            "tracked",
+            "tracked_set_sha256",
+        }
+        or not isinstance(static_section.get("collected"), list)
+    ):
+        raise ScanFailure()
+    expected: dict[str, tuple[str, str]] = {
+        "manifest.json": ("100644", "")
+    }
+    for entry in source["tracked"]:
+        if not isinstance(entry, dict) or set(entry) != {
+            "git_mode",
+            "path",
+            "sha256",
+        }:
+            raise ScanFailure()
+        path = _safe_posix_path(entry["path"])
+        mode = entry["git_mode"]
+        digest = entry["sha256"]
+        if mode not in {"100644", "100755"} or not isinstance(
+            digest, str
+        ) or SAFE_SHA256.fullmatch(digest) is None:
+            raise ScanFailure()
+        artifact_path = f"source/{path}"
+        if artifact_path in expected:
+            raise ScanFailure()
+        expected[artifact_path] = (mode, digest)
+    for entry in static_section["collected"]:
+        if not isinstance(entry, dict) or set(entry) != {"path", "sha256"}:
+            raise ScanFailure()
+        path = _safe_posix_path(entry["path"])
+        digest = entry["sha256"]
+        if not isinstance(digest, str) or SAFE_SHA256.fullmatch(digest) is None:
+            raise ScanFailure()
+        artifact_path = f"static/{path}"
+        if artifact_path in expected:
+            raise ScanFailure()
+        expected[artifact_path] = ("100644", digest)
+    if len(expected) > MAX_TREE_ENTRIES:
+        raise ScanFailure()
+    return expected
+
+
+def scan_extracted_artifact(
+    root: Path,
+    release_sha: str,
+    safety_token: str,
+    patterns: SensitivePatterns,
+) -> bool:
+    if (
+        safety_token != expected_artifact_safety_token(release_sha, root)
+        or root.name != "release"
+    ):
+        return False
+    try:
+        root_status = root.lstat()
+    except OSError:
+        return False
+    if (
+        not stat.S_ISDIR(root_status.st_mode)
+        or stat.S_ISLNK(root_status.st_mode)
+        or stat.S_IMODE(root_status.st_mode) != 0o755
+    ):
+        return False
+    manifest_path = root / "manifest.json"
+    try:
+        metadata = manifest_path.lstat()
+    except OSError:
+        return False
+    if (
+        not stat.S_ISREG(metadata.st_mode)
+        or stat.S_ISLNK(metadata.st_mode)
+        or metadata.st_size > MAX_MANIFEST_BYTES
+    ):
+        return False
+    manifest_ok, _ = scan_regular_file(manifest_path, patterns)
+    if not manifest_ok:
+        return False
+    try:
+        handle, observed = _safe_open_file(manifest_path)
+        try:
+            manifest_data = handle.read(observed.st_size + 1)
+        finally:
+            handle.close()
+        if len(manifest_data) != observed.st_size:
+            raise ScanFailure()
+        if patterns.matches(manifest_data):
+            raise ScanFailure()
+        if not _authoritative_manifest_valid(manifest_data):
+            raise ScanFailure()
+        manifest = json.loads(manifest_data.decode("utf-8"))
+        expected = _manifest_entry_map(manifest, release_sha)
+    except (OSError, UnicodeError, ValueError, ScanFailure):
+        return False
+    actual: dict[str, Path] = {}
+    try:
+        for path in root.rglob("*"):
+            status = path.lstat()
+            if stat.S_ISLNK(status.st_mode) or not (
+                stat.S_ISDIR(status.st_mode) or stat.S_ISREG(status.st_mode)
+            ):
+                return False
+            if (
+                stat.S_ISDIR(status.st_mode)
+                and stat.S_IMODE(status.st_mode) != 0o755
+            ):
+                return False
+            if stat.S_ISREG(status.st_mode):
+                relative = path.relative_to(root).as_posix()
+                if relative in actual:
+                    return False
+                actual[relative] = path
+    except OSError:
+        return False
+    if set(actual) != set(expected):
+        return False
+    total = 0
+    for relative, path in actual.items():
+        status = path.lstat()
+        total += status.st_size
+        if total > MAX_TREE_BYTES:
+            return False
+        mode, digest = expected[relative]
+        expected_mode = 0o755 if mode == "100755" else 0o644
+        if stat.S_IMODE(status.st_mode) != expected_mode:
+            return False
+        if relative == "manifest.json":
+            continue
+        clean, _ = scan_regular_file(path, patterns, expected_sha256=digest)
+        if not clean:
+            return False
+    return True
+
+
+def _tree_files(root: Path, patterns: SensitivePatterns) -> list[Path]:
+    files: list[Path] = []
+    total = 0
+    for path in root.rglob("*"):
+        status = path.lstat()
+        if stat.S_ISLNK(status.st_mode) or not (
+            stat.S_ISDIR(status.st_mode) or stat.S_ISREG(status.st_mode)
+        ):
+            raise ScanFailure()
+        try:
+            relative_bytes = path.relative_to(root).as_posix().encode("utf-8")
+        except (UnicodeError, ValueError) as exc:
+            raise ScanFailure() from exc
+        if patterns.matches(relative_bytes):
+            raise ScanFailure()
+        if stat.S_ISREG(status.st_mode):
+            if _is_forbidden_env_name(path.name):
+                raise ScanFailure()
+            files.append(path)
+            total += status.st_size
+            if len(files) > MAX_TREE_ENTRIES or total > MAX_TREE_BYTES:
+                raise ScanFailure()
+    return files
+
+
+def _hash_without_patterns(path: Path) -> str:
+    handle, before = _safe_open_file(path)
+    digest = hashlib.sha256()
+    try:
+        remaining = before.st_size
+        while remaining:
+            chunk = handle.read(min(CHUNK_BYTES, remaining))
+            if not chunk:
+                raise ScanFailure()
+            digest.update(chunk)
+            remaining -= len(chunk)
+        if handle.read(1):
+            raise ScanFailure()
+        after = os.fstat(handle.fileno())
+    finally:
+        handle.close()
+    if (
+        (before.st_dev, before.st_ino, before.st_size, before.st_mtime_ns)
+        != (after.st_dev, after.st_ino, after.st_size, after.st_mtime_ns)
+    ):
+        raise ScanFailure()
+    return digest.hexdigest()
+
+
+def _validated_backup_values(directory: Path) -> dict[str, str] | None:
+    """Validate dump scope and integrity without applying target patterns."""
+
+    try:
+        entries = {path.name: path for path in directory.iterdir()}
+        status = directory.lstat()
+    except OSError:
+        return None
+    if (
+        set(entries) != {"database.dump", "integrity.manifest"}
+        or not stat.S_ISDIR(status.st_mode)
+        or stat.S_ISLNK(status.st_mode)
+        or stat.S_IMODE(status.st_mode) != 0o700
+    ):
+        return None
+    for path in entries.values():
+        item = path.lstat()
+        if (
+            not stat.S_ISREG(item.st_mode)
+            or stat.S_ISLNK(item.st_mode)
+            or stat.S_IMODE(item.st_mode) != 0o600
+        ):
+            return None
+    try:
+        handle, observed = _safe_open_file(entries["integrity.manifest"])
+        try:
+            if observed.st_size > MAX_MANIFEST_BYTES:
+                return None
+            manifest_data = handle.read(observed.st_size + 1)
+        finally:
+            handle.close()
+        if len(manifest_data) != observed.st_size:
+            return None
+        text = manifest_data.decode("utf-8")
+    except (OSError, UnicodeError, ScanFailure):
+        return None
+    if not text.endswith("\n") or "\r" in text:
+        return None
+    lines = text.splitlines()
+    if len(lines) != len(BACKUP_INTEGRITY_KEYS) + 3:
+        return None
+    values: dict[str, str] = {}
+    for line in lines:
+        key, separator, value = line.partition("=")
+        if not separator or key in values or not value:
+            return None
+        values[key] = value
+    if (
+        set(values)
+        != BACKUP_INTEGRITY_KEYS
+        | {"manifest.format", "archive.scope", "archive.sha256"}
+        or lines[0]
+        != "manifest.format=travel-readiness-postgresql-backup-v1"
+        or not lines[1].startswith("archive.sha256=")
+        or lines[2]
+        != "archive.scope=approved-public-schema-without-ephemeral-data"
+        or values.get("manifest.format")
+        != "travel-readiness-postgresql-backup-v1"
+        or values.get("archive.scope")
+        != "approved-public-schema-without-ephemeral-data"
+        or SAFE_SHA256.fullmatch(values.get("archive.sha256", "")) is None
+        or values.get("postgresql.version_num") != "180006"
+    ):
+        return None
+    for key, value in values.items():
+        if key in {
+            "manifest.format",
+            "archive.scope",
+            "archive.sha256",
+            "postgresql.version_num",
+        }:
+            continue
+        if re.fullmatch(
+            r"(?:schema\.[a-z0-9_.]+|database\.locale_profile)\.sha256",
+            key,
+        ):
+            if SAFE_SHA256.fullmatch(value) is None:
+                return None
+        elif re.fullmatch(r"table\.[a-z0-9_]+\.sha256", key):
+            if SAFE_SHA256.fullmatch(value) is None:
+                return None
+        elif re.fullmatch(r"table\.[a-z0-9_]+\.count", key):
+            if re.fullmatch(r"0|[1-9][0-9]{0,19}", value) is None:
+                return None
+        elif key in {"pointer.entry", "pointer.travel_warning"}:
+            if re.fullmatch(
+                r"(?:NONE|[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}):(0|[1-9][0-9]{0,19})",
+                value,
+            ) is None:
+                return None
+        else:
+            return None
+    try:
+        if _hash_without_patterns(entries["database.dump"]) != values[
+            "archive.sha256"
+        ]:
+            return None
+    except ScanFailure:
+        return None
+    return values
+
+
+def validate_backup_boundary(directory: Path) -> bool:
+    """Validate dump scope and integrity without applying target patterns."""
+
+    return _validated_backup_values(directory) is not None
+
+
+def scan_runtime_evidence(
+    roots: Sequence[Path],
+    files: Sequence[Path],
+    backup_directory: Path,
+    release_sha: str,
+    safety_token: str,
+    patterns: SensitivePatterns,
+) -> bool:
+    if safety_token != expected_runtime_safety_token(
+        release_sha, roots, files, backup_directory
+    ):
+        return False
+    try:
+        candidates: list[Path] = []
+        for root in roots:
+            if patterns.matches(root.name.encode("utf-8")):
+                return False
+            candidates.extend(_tree_files(root, patterns))
+        if any(patterns.matches(path.name.encode("utf-8")) for path in files):
+            return False
+        if patterns.matches(backup_directory.name.encode("utf-8")):
+            return False
+        candidates.extend(files)
+        if not candidates:
+            return False
+        if len(set(candidates)) != len(candidates):
+            return False
+        total = 0
+        for path in candidates:
+            if _is_forbidden_env_name(path.name):
+                return False
+            status = path.lstat()
+            total += status.st_size
+            if total > MAX_TREE_BYTES:
+                return False
+            clean, _ = scan_regular_file(path, patterns)
+            if not clean:
+                return False
+        return validate_backup_boundary(backup_directory)
+    except (OSError, ScanFailure):
+        return False
+
+
+def _database_value_bytes(value: object) -> bytes:
+    if isinstance(value, str):
+        payload = value.encode("utf-8")
+    elif isinstance(value, (bytes, bytearray, memoryview)):
+        payload = bytes(value)
+    elif isinstance(value, (dict, list, tuple)):
+        payload = json.dumps(
+            value,
+            ensure_ascii=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        ).encode("utf-8")
+    else:
+        payload = str(value).encode("utf-8")
+    if len(payload) > MAX_DB_VALUE_BYTES:
+        raise ScanFailure()
+    return payload
+
+
+def _artifact_integrity_query(artifact_root: Path) -> str:
+    path = artifact_root / "source" / "scripts" / "postgresql-integrity.sql"
+    handle, observed = _safe_open_file(path)
+    try:
+        if observed.st_size > MAX_MANIFEST_BYTES:
+            raise ScanFailure()
+        data = handle.read(observed.st_size + 1)
+    finally:
+        handle.close()
+    if len(data) != observed.st_size:
+        raise ScanFailure()
+    try:
+        text = data.decode("utf-8")
+    except UnicodeError as exc:
+        raise ScanFailure() from exc
+    prefix = (
+        "SET TIME ZONE 'UTC';\n"
+        "SET DateStyle = 'ISO, YMD';\n"
+        "SET search_path = pg_catalog, public;\n\n"
+    )
+    if not text.startswith(prefix) or not text.endswith(";\n") or "\x00" in text:
+        raise ScanFailure()
+    query = text[len(prefix) :]
+    if not query.startswith("WITH integrity_lines(sort_key, line) AS ("):
+        raise ScanFailure()
+    return query
+
+
+def _integrity_values_from_rows(rows: object) -> dict[str, str]:
+    if not isinstance(rows, (list, tuple)) or len(rows) != len(
+        BACKUP_INTEGRITY_KEYS
+    ):
+        raise ScanFailure()
+    values: dict[str, str] = {}
+    for row in rows:
+        if not isinstance(row, tuple) or len(row) != 1 or not isinstance(row[0], str):
+            raise ScanFailure()
+        key, separator, value = row[0].partition("=")
+        if not separator or not value or key in values:
+            raise ScanFailure()
+        values[key] = value
+    if set(values) != BACKUP_INTEGRITY_KEYS:
+        raise ScanFailure()
+    return values
+
+
+def _scan_one_database(
+    *,
+    connector: Callable[..., object],
+    host: str,
+    port: int,
+    user: str,
+    password: str,
+    database: str,
+    patterns: SensitivePatterns,
+    integrity_query: str,
+) -> tuple[
+    bool,
+    tuple[tuple[str, str, str, str], ...],
+    dict[str, str],
+]:
+    try:
+        from psycopg import sql
+    except ImportError as exc:
+        raise ScanFailure() from exc
+    connection = None
+    try:
+        connection = connector(
+            host=host,
+            port=port,
+            user=user,
+            password=password,
+            dbname=database,
+            connect_timeout=5,
+            options=(
+                "-c default_transaction_read_only=on "
+                "-c statement_timeout=15000 "
+                "-c lock_timeout=5000 "
+                "-c idle_in_transaction_session_timeout=15000 "
+                "-c timezone=UTC "
+                "-c DateStyle=ISO,YMD "
+                "-c search_path=pg_catalog,public"
+            ),
+            autocommit=False,
+        )
+        with connection.cursor() as cursor:
+            cursor.execute(
+                "BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY"
+            )
+            cursor.execute("SHOW transaction_read_only")
+            row = cursor.fetchone()
+            if row != ("on",):
+                raise ScanFailure()
+            cursor.execute("SHOW server_version_num")
+            if cursor.fetchone() != ("180006",):
+                raise ScanFailure()
+            cursor.execute("SELECT current_database() = %s", (database,))
+            if cursor.fetchone() != (True,):
+                raise ScanFailure()
+            cursor.execute("SELECT current_user = %s", (user,))
+            if cursor.fetchone() != (True,):
+                raise ScanFailure()
+            cursor.execute(
+                """
+                SELECT NOT (
+                    rolsuper OR rolcreatedb OR rolcreaterole OR rolinherit
+                    OR rolreplication OR rolbypassrls
+                )
+                FROM pg_catalog.pg_roles
+                WHERE rolname = current_user
+                """
+            )
+            if cursor.fetchone() != (True,):
+                raise ScanFailure()
+            cursor.execute(
+                "SELECT count(*) = 0 FROM pg_catalog.pg_largeobject_metadata"
+            )
+            if cursor.fetchone() != (True,):
+                raise ScanFailure()
+            cursor.execute(
+                """
+                SELECT count(*) = 0
+                FROM pg_catalog.pg_class AS c
+                JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+                WHERE n.nspname <> %s AND n.nspname !~ %s
+                  AND (
+                    c.relkind = 'f'
+                    OR (
+                      c.relkind IN ('r', 'p', 'm')
+                      AND (c.relrowsecurity OR c.relforcerowsecurity)
+                    )
+                  )
+                """,
+                ("information_schema", "^pg_"),
+            )
+            if cursor.fetchone() != (True,):
+                raise ScanFailure()
+            cursor.execute(
+                """
+                SELECT n.nspname, c.relname, a.attname, t.typname,
+                       c.relrowsecurity, c.relforcerowsecurity
+                FROM pg_catalog.pg_attribute AS a
+                JOIN pg_catalog.pg_class AS c ON c.oid = a.attrelid
+                JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
+                JOIN pg_catalog.pg_type AS t ON t.oid = a.atttypid
+                WHERE a.attnum > 0 AND NOT a.attisdropped
+                  AND c.relkind IN ('r', 'p', 'm')
+                  AND n.nspname <> %s
+                  AND n.nspname !~ %s
+                ORDER BY n.nspname COLLATE "C", c.relname COLLATE "C", a.attnum
+                """,
+                ("information_schema", "^pg_"),
+            )
+            columns = tuple(cursor.fetchall())
+        if not columns or len(columns) > MAX_DB_COLUMNS:
+            raise ScanFailure()
+        normalized: list[tuple[str, str, str, str]] = []
+        for index, column in enumerate(columns):
+            if (
+                not isinstance(column, tuple)
+                or len(column) != 6
+                or any(
+                    not isinstance(item, str) or not item for item in column[:4]
+                )
+                or column[4:] != (False, False)
+            ):
+                raise ScanFailure()
+            schema, table, name, type_name = column[:4]
+            if any("\x00" in item or len(item) > 63 for item in column[:3]):
+                raise ScanFailure()
+            if patterns.matches("\0".join(column[:4]).encode("utf-8")):
+                connection.rollback()
+                return False, tuple(normalized), {}
+            normalized.append((schema, table, name, type_name))
+            query = sql.SQL("SELECT {} FROM {}.{} WHERE {} IS NOT NULL").format(
+                sql.Identifier(name),
+                sql.Identifier(schema),
+                sql.Identifier(table),
+                sql.Identifier(name),
+            )
+            with connection.cursor(name=f"sensitive_absence_{index}") as stream:
+                stream.itersize = 128
+                stream.execute(query)
+                for row in stream:
+                    if not isinstance(row, tuple) or len(row) != 1:
+                        raise ScanFailure()
+                    if patterns.matches(_database_value_bytes(row[0])):
+                        connection.rollback()
+                        return False, tuple(normalized), {}
+        with connection.cursor() as cursor:
+            cursor.execute(integrity_query)
+            integrity_values = _integrity_values_from_rows(cursor.fetchall())
+        connection.rollback()
+        return True, tuple(normalized), integrity_values
+    except ScanFailure:
+        if connection is not None:
+            try:
+                connection.rollback()
+            except Exception:
+                pass
+        raise
+    except Exception as exc:
+        if connection is not None:
+            try:
+                connection.rollback()
+            except Exception:
+                pass
+        raise ScanFailure() from exc
+    finally:
+        if connection is not None:
+            try:
+                connection.close()
+            except Exception:
+                pass
+
+
+def scan_databases(
+    configuration: ScanConfiguration,
+    patterns: SensitivePatterns,
+    main_password: str,
+    restored_password: str,
+    *,
+    connector: Callable[..., object] | None = None,
+    integrity_query: str | None = None,
+) -> bool:
+    try:
+        backup_values = _validated_backup_values(configuration.backup_directory)
+        if backup_values is None or configuration.db_safety_token != (
+            expected_db_safety_token(
+                configuration, backup_values["archive.sha256"]
+            )
+        ):
+            return False
+        expected_integrity = {
+            key: backup_values[key] for key in BACKUP_INTEGRITY_KEYS
+        }
+        if integrity_query is None:
+            integrity_query = _artifact_integrity_query(
+                configuration.artifact_root
+            )
+        if connector is None:
+            import psycopg
+
+            connector = psycopg.connect
+        main_ok, main_columns, main_integrity = _scan_one_database(
+            connector=connector,
+            host=configuration.db_host,
+            port=configuration.db_port,
+            user=configuration.main_db_user,
+            password=main_password,
+            database=configuration.main_database,
+            patterns=patterns,
+            integrity_query=integrity_query,
+        )
+        restored_ok, restored_columns, restored_integrity = _scan_one_database(
+            connector=connector,
+            host=configuration.db_host,
+            port=configuration.db_port,
+            user=configuration.restored_db_user,
+            password=restored_password,
+            database=configuration.restored_database,
+            patterns=patterns,
+            integrity_query=integrity_query,
+        )
+        return (
+            main_ok
+            and restored_ok
+            and main_columns == restored_columns
+            and main_integrity == expected_integrity
+            and restored_integrity == expected_integrity
+        )
+    except ScanFailure:
+        return False
+
+
+def _canonical_existing_path(value: str, *, directory: bool) -> Path:
+    path = Path(value)
+    if not path.is_absolute() or os.path.abspath(path) != str(path):
+        raise ArgumentFailure()
+    try:
+        status = path.lstat()
+        resolved = path.resolve(strict=True)
+    except OSError as exc:
+        raise ArgumentFailure() from exc
+    if resolved != path or stat.S_ISLNK(status.st_mode):
+        raise ArgumentFailure()
+    if directory and not stat.S_ISDIR(status.st_mode):
+        raise ArgumentFailure()
+    if not directory and not stat.S_ISREG(status.st_mode):
+        raise ArgumentFailure()
+    return path
+
+
+def _is_descendant(path: Path, ancestor: Path) -> bool:
+    try:
+        relative = path.relative_to(ancestor)
+    except ValueError:
+        return False
+    return bool(relative.parts)
+
+
+def _allowed_evidence_path(path: Path) -> bool:
+    anchors: list[Path] = []
+    output = REPOSITORY_ROOT / "output"
+    if output.exists():
+        anchors.append(output.resolve(strict=True))
+    for candidate in (Path(tempfile.gettempdir()), Path("/private/tmp")):
+        try:
+            resolved = candidate.resolve(strict=True)
+        except OSError:
+            continue
+        if resolved not in anchors:
+            anchors.append(resolved)
+    return any(_is_descendant(path, anchor) for anchor in anchors)
+
+
+def _overlaps(left: Path, right: Path) -> bool:
+    return left == right or _is_descendant(left, right) or _is_descendant(right, left)
+
+
+def parse_configuration(arguments: Sequence[str]) -> ScanConfiguration:
+    scalar_options = {
+        "--repository-root",
+        "--release-sha",
+        "--git-safety-token",
+        "--artifact-root",
+        "--artifact-safety-token",
+        "--backup-directory",
+        "--runtime-safety-token",
+        "--db-host",
+        "--db-port",
+        "--main-db-user",
+        "--main-db-password-env",
+        "--restored-db-user",
+        "--restored-db-password-env",
+        "--main-database",
+        "--restored-database",
+        "--db-safety-token",
+    }
+    repeated_options = {"--runtime-root", "--runtime-file"}
+    scalar: dict[str, str] = {}
+    repeated: dict[str, list[str]] = {option: [] for option in repeated_options}
+    index = 0
+    if not arguments or len(arguments) > 96:
+        raise ArgumentFailure()
+    while index < len(arguments):
+        option = arguments[index]
+        if (
+            option not in scalar_options | repeated_options
+            or index + 1 >= len(arguments)
+        ):
+            raise ArgumentFailure()
+        value = arguments[index + 1]
+        if (
+            not isinstance(value, str)
+            or not value
+            or len(value) > 4_096
+            or any(ord(character) < 32 for character in value)
+        ):
+            raise ArgumentFailure()
+        if option in scalar_options:
+            if option in scalar:
+                raise ArgumentFailure()
+            scalar[option] = value
+        else:
+            repeated[option].append(value)
+        index += 2
+    if (
+        set(scalar) != scalar_options
+        or not 1 <= len(repeated["--runtime-root"]) <= 16
+        or len(repeated["--runtime-file"]) > 32
+    ):
+        raise ArgumentFailure()
+
+    repository = _canonical_existing_path(
+        scalar["--repository-root"], directory=True
+    )
+    if repository != REPOSITORY_ROOT.resolve(strict=True):
+        raise ArgumentFailure()
+    release_sha = scalar["--release-sha"]
+    if SAFE_SHA.fullmatch(release_sha) is None:
+        raise ArgumentFailure()
+    artifact = _canonical_existing_path(scalar["--artifact-root"], directory=True)
+    backup = _canonical_existing_path(
+        scalar["--backup-directory"], directory=True
+    )
+    roots = tuple(
+        _canonical_existing_path(value, directory=True)
+        for value in repeated["--runtime-root"]
+    )
+    files = tuple(
+        _canonical_existing_path(value, directory=False)
+        for value in repeated["--runtime-file"]
+    )
+    evidence_paths = (artifact, backup, *roots, *files)
+    if any(
+        not _allowed_evidence_path(path) or _is_forbidden_env_name(path.name)
+        for path in evidence_paths
+    ):
+        raise ArgumentFailure()
+    if artifact.name != "release" or any(
+        _overlaps(left, right)
+        for position, left in enumerate((artifact, backup, *roots))
+        for right in (artifact, backup, *roots)[position + 1 :]
+    ):
+        raise ArgumentFailure()
+    if any(
+        any(_is_descendant(path, root) or path == root for root in (artifact, backup, *roots))
+        for path in files
+    ) or len(set(evidence_paths)) != len(evidence_paths):
+        raise ArgumentFailure()
+
+    restored = scalar["--restored-database"]
+    if (
+        scalar["--db-host"] != "127.0.0.1"
+        or not scalar["--db-port"].isdigit()
+        or not 1 <= int(scalar["--db-port"]) <= 65_535
+        or SAFE_IDENTIFIER.fullmatch(scalar["--main-db-user"]) is None
+        or SAFE_IDENTIFIER.fullmatch(scalar["--restored-db-user"]) is None
+        or scalar["--main-db-password-env"]
+        != "TRAVEL_READINESS_SENSITIVE_SCAN_MAIN_DB_PASSWORD"
+        or scalar["--restored-db-password-env"]
+        != "TRAVEL_READINESS_SENSITIVE_SCAN_RESTORED_DB_PASSWORD"
+        or scalar["--main-database"] != "travel_readiness"
+        or SAFE_RESTORE_DATABASE.fullmatch(restored) is None
+        or any(
+            marker in restored
+            for marker in ("prod", "live", "stag", "main", "master", "release")
+        )
+    ):
+        raise ArgumentFailure()
+    return ScanConfiguration(
+        repository_root=repository,
+        release_sha=release_sha,
+        git_safety_token=scalar["--git-safety-token"],
+        artifact_root=artifact,
+        artifact_safety_token=scalar["--artifact-safety-token"],
+        runtime_roots=roots,
+        runtime_files=files,
+        backup_directory=backup,
+        runtime_safety_token=scalar["--runtime-safety-token"],
+        db_host=scalar["--db-host"],
+        db_port=int(scalar["--db-port"]),
+        main_db_user=scalar["--main-db-user"],
+        main_db_password_env=scalar["--main-db-password-env"],
+        restored_db_user=scalar["--restored-db-user"],
+        restored_db_password_env=scalar["--restored-db-password-env"],
+        main_database=scalar["--main-database"],
+        restored_database=restored,
+        db_safety_token=scalar["--db-safety-token"],
+    )
+
+
+def run_boundaries(
+    configuration: ScanConfiguration,
+    patterns: SensitivePatterns,
+    main_database_password: str,
+    restored_database_password: str,
+    *,
+    connector: Callable[..., object] | None = None,
+) -> dict[str, bool]:
+    statuses = {name: False for name in RECEIPT_ORDER}
+    try:
+        statuses["git"] = scan_git_reachable(
+            configuration.repository_root,
+            configuration.release_sha,
+            configuration.git_safety_token,
+            patterns,
+        )
+    except BaseException:
+        statuses["git"] = False
+    try:
+        statuses["artifact"] = scan_extracted_artifact(
+            configuration.artifact_root,
+            configuration.release_sha,
+            configuration.artifact_safety_token,
+            patterns,
+        )
+    except BaseException:
+        statuses["artifact"] = False
+    try:
+        statuses["runtime"] = scan_runtime_evidence(
+            configuration.runtime_roots,
+            configuration.runtime_files,
+            configuration.backup_directory,
+            configuration.release_sha,
+            configuration.runtime_safety_token,
+            patterns,
+        )
+    except BaseException:
+        statuses["runtime"] = False
+    try:
+        statuses["db"] = scan_databases(
+            configuration,
+            patterns,
+            main_database_password,
+            restored_database_password,
+            connector=connector,
+        )
+    except BaseException:
+        statuses["db"] = False
+    return statuses
diff --git a/operations/sensitive_absence_cli.py b/operations/sensitive_absence_cli.py
new file mode 100644
index 0000000..91bcd12
--- /dev/null
+++ b/operations/sensitive_absence_cli.py
@@ -0,0 +1,91 @@
+#!/usr/bin/env python3
+"""Secret-first entry point for the fixed-shape absence scanner."""
+
+import os
+
+
+# These removals precede argument parsing, path access, Django/psycopg imports,
+# subprocess creation and every possible diagnostic path.
+_TARGET_SECRET = os.environ.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)
+_TRIP_MARKERS = os.environ.pop(
+    "TRAVEL_READINESS_SENSITIVE_SCAN_TRIP_MARKERS", None
+)
+_RAW_MARKERS = os.environ.pop(
+    "TRAVEL_READINESS_SENSITIVE_SCAN_RAW_MARKERS", None
+)
+_MAIN_DATABASE_PASSWORD = os.environ.pop(
+    "TRAVEL_READINESS_SENSITIVE_SCAN_MAIN_DB_PASSWORD", None
+)
+_RESTORED_DATABASE_PASSWORD = os.environ.pop(
+    "TRAVEL_READINESS_SENSITIVE_SCAN_RESTORED_DB_PASSWORD", None
+)
+os.environ.pop("PGPASSWORD", None)
+os.environ.pop("DATABASE_URL", None)
+
+import sys
+from pathlib import Path
+
+
+FAILED_RECEIPT = "git=failed db=failed artifact=failed runtime=failed"
+
+
+def _bootstrap() -> None:
+    repository = Path(__file__).resolve().parents[1]
+    prefix = Path(sys.prefix).resolve(strict=True)
+    if prefix != (repository / ".venv").resolve(strict=True):
+        raise RuntimeError()
+    site_packages = (
+        prefix
+        / "lib"
+        / f"python{sys.version_info.major}.{sys.version_info.minor}"
+        / "site-packages"
+    ).resolve(strict=True)
+    if not site_packages.is_dir():
+        raise RuntimeError()
+    sys.path[:0] = [str(repository), str(site_packages)]
+
+
+def main() -> int:
+    try:
+        _bootstrap()
+        from operations.sensitive_absence import (
+            SensitivePatterns,
+            fixed_receipt,
+            parse_configuration,
+            parse_marker_array,
+            run_boundaries,
+        )
+
+        if not all(
+            isinstance(value, str) and value
+            for value in (
+                _TARGET_SECRET,
+                _TRIP_MARKERS,
+                _RAW_MARKERS,
+                _MAIN_DATABASE_PASSWORD,
+                _RESTORED_DATABASE_PASSWORD,
+            )
+        ):
+            raise RuntimeError()
+        patterns = SensitivePatterns(
+            _TARGET_SECRET,
+            parse_marker_array(_TRIP_MARKERS),
+            parse_marker_array(_RAW_MARKERS),
+        )
+        configuration = parse_configuration(sys.argv[1:])
+        statuses = run_boundaries(
+            configuration,
+            patterns,
+            _MAIN_DATABASE_PASSWORD,
+            _RESTORED_DATABASE_PASSWORD,
+        )
+        receipt = fixed_receipt(statuses)
+        print(receipt, flush=True)
+        return 0 if all(statuses.values()) else 1
+    except BaseException:
+        print(FAILED_RECEIPT, flush=True)
+        return 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/operations/tests/test_sensitive_absence.py b/operations/tests/test_sensitive_absence.py
new file mode 100644
index 0000000..52f66d2
--- /dev/null
+++ b/operations/tests/test_sensitive_absence.py
@@ -0,0 +1,1295 @@
+from __future__ import annotations
+
+from dataclasses import replace
+import hashlib
+import io
+import json
+import os
+from pathlib import Path
+import re
+import socket
+import stat
+import subprocess
+import tempfile
+import unittest
+from urllib.parse import quote, quote_plus
+
+from operations.sensitive_absence import (
+    ArgumentFailure,
+    BACKUP_INTEGRITY_KEYS,
+    REPOSITORY_ROOT,
+    ScanFailure,
+    ScanConfiguration,
+    SensitivePatterns,
+    expected_artifact_safety_token,
+    expected_db_safety_token,
+    expected_runtime_safety_token,
+    fixed_receipt,
+    parse_configuration,
+    scan_databases,
+    scan_extracted_artifact,
+    scan_git_reachable,
+    scan_runtime_evidence,
+    validate_backup_boundary,
+    _artifact_integrity_query,
+    _scan_one_database,
+)
+
+
+SYNTHETIC_SECRET = (
+    "only-for-test/Q7v+M3p%2Fz=K9d_R4x-A8nY6uW2cT5jH0bN1sE"
+)
+SYNTHETIC_TRIP = (
+    "fixture-trip-marker-7F3B9D1A6C8E2K4M5P0R"
+)
+SYNTHETIC_RAW = (
+    "합성원문표식-가나다라마바사아자차카타파하-9041736258"
+)
+MAIN_DATABASE_PASSWORD = "synthetic-main-database-password-not-a-provider-key"
+RESTORED_DATABASE_PASSWORD = (
+    "synthetic-restored-database-password-not-a-provider-key"
+)
+FAILED_RECEIPT = "git=failed db=failed artifact=failed runtime=failed"
+SCRIPT = REPOSITORY_ROOT / "scripts" / "check-sensitive-absence"
+CLI = REPOSITORY_ROOT / "operations" / "sensitive_absence_cli.py"
+INTEGRITY_SQL = REPOSITORY_ROOT / "scripts" / "postgresql-integrity.sql"
+POSTGRESQL_BIN = Path("/opt/homebrew/Cellar/postgresql@18/18.6/bin")
+
+
+def patterns() -> SensitivePatterns:
+    return SensitivePatterns(
+        SYNTHETIC_SECRET,
+        (SYNTHETIC_TRIP,),
+        (SYNTHETIC_RAW,),
+    )
+
+
+def run_git(repository: Path, *arguments: str) -> str:
+    result = subprocess.run(
+        ["/usr/bin/git", *arguments],
+        cwd=repository,
+        env={
+            "PATH": "/usr/bin:/bin",
+            "LANG": "C",
+            "LC_ALL": "C",
+            "GIT_CONFIG_NOSYSTEM": "1",
+            "GIT_CONFIG_GLOBAL": "/dev/null",
+            "GIT_CONFIG_SYSTEM": "/dev/null",
+        },
+        stdin=subprocess.DEVNULL,
+        stdout=subprocess.PIPE,
+        stderr=subprocess.PIPE,
+        text=True,
+        check=True,
+    )
+    return result.stdout.strip()
+
+
+def commit_file(repository: Path, path: str, body: str, message: str) -> str:
+    target = repository / path
+    target.parent.mkdir(parents=True, exist_ok=True)
+    target.write_text(body, encoding="utf-8")
+    run_git(repository, "add", "--", path)
+    run_git(repository, "commit", "-q", "-m", message)
+    return run_git(repository, "rev-parse", "HEAD")
+
+
+def initialize_git_repository(root: Path) -> Path:
+    repository = root / "repository"
+    repository.mkdir()
+    run_git(repository, "init", "-q", "-b", "main")
+    run_git(repository, "config", "user.name", "Synthetic Test")
+    run_git(repository, "config", "user.email", "synthetic@example.invalid")
+    return repository
+
+
+def canonical_json(value: object) -> bytes:
+    return (
+        json.dumps(
+            value,
+            ensure_ascii=False,
+            allow_nan=False,
+            sort_keys=True,
+            separators=(",", ":"),
+        )
+        + "\n"
+    ).encode("utf-8")
+
+
+def canonical_digest(value: object) -> str:
+    return hashlib.sha256(canonical_json(value)).hexdigest()
+
+
+def write_artifact(root: Path, release_sha: str, source_body: bytes) -> Path:
+    release = root / "release"
+    static_file = release / "static" / "site.css"
+    static_file.parent.mkdir(parents=True)
+    static_file.write_bytes(b"body { color: #123456; }\n")
+    static_file.chmod(0o644)
+    release.chmod(0o755)
+    static_file.parent.chmod(0o755)
+    tracked_content = {
+        "app.txt": source_body,
+        "runtime/versions.toml": b"python = '3.14.7'\n",
+        "scripts/postgresql-integrity.sql": INTEGRITY_SQL.read_bytes(),
+        "scripts/verify-release-runtime": b"synthetic verifier fixture\n",
+        "uv.lock": b"version = 1\n",
+    }
+    tracked = []
+    for relative, content in sorted(tracked_content.items()):
+        path = release / "source" / relative
+        path.parent.mkdir(parents=True, exist_ok=True)
+        path.parent.chmod(0o755)
+        path.write_bytes(content)
+        path.chmod(0o644)
+        tracked.append(
+            {
+                "git_mode": "100644",
+                "path": relative,
+                "sha256": hashlib.sha256(content).hexdigest(),
+            }
+        )
+    static_entries = [
+        {
+            "path": "site.css",
+            "sha256": hashlib.sha256(static_file.read_bytes()).hexdigest(),
+        }
+    ]
+    packages = [{"name": "Django", "version": "5.2.17"}]
+    leaves = [
+        {
+            "app": "admin",
+            "module": "django.contrib.admin.migrations.0003_logentry_add_action_flag_choices",
+            "name": "0003_logentry_add_action_flag_choices",
+            "origin": "django",
+            "path": "django/contrib/admin/migrations/0003_logentry_add_action_flag_choices.py",
+            "sha256": "4" * 64,
+        }
+    ]
+    origins = [
+        {
+            "collected_path": "site.css",
+            "origin": "django",
+            "path": "django/contrib/admin/static/admin/css/base.css",
+            "sha256": static_entries[0]["sha256"],
+        }
+    ]
+    manifest = {
+        "archive": {
+            "compression": "none",
+            "format": "ustar",
+            "gid": 0,
+            "group": "root",
+            "mtime": 0,
+            "regular_modes": ["0644", "0755"],
+            "uid": 0,
+            "user": "root",
+        },
+        "dependencies": {
+            "lock_path": "uv.lock",
+            "lock_sha256": hashlib.sha256(tracked_content["uv.lock"]).hexdigest(),
+            "package_set_sha256": canonical_digest(packages),
+            "packages": packages,
+        },
+        "git_object_format": "sha1",
+        "migrations": {
+            "leaf_set_sha256": canonical_digest(leaves),
+            "leaves": leaves,
+        },
+        "release_sha": release_sha,
+        "runtime": {
+            "path": "runtime/versions.toml",
+            "sha256": hashlib.sha256(
+                tracked_content["runtime/versions.toml"]
+            ).hexdigest(),
+            "versions": {
+                "django": "5.2.17",
+                "gunicorn": "26.2.0",
+                "postgresql": "18.6",
+                "psycopg": "3.3.4",
+                "psycopg_distribution": "binary-wheel",
+                "python": "3.14.7",
+                "uv": "0.12.6",
+            },
+        },
+        "schema_version": 1,
+        "source": {
+            "tracked": tracked,
+            "tracked_set_sha256": canonical_digest(tracked),
+        },
+        "static": {
+            "collected": static_entries,
+            "collected_set_sha256": canonical_digest(static_entries),
+            "origins": origins,
+            "tracked": [],
+            "tracked_set_sha256": canonical_digest([]),
+        },
+    }
+    manifest_path = release / "manifest.json"
+    manifest_path.write_bytes(canonical_json(manifest))
+    manifest_path.chmod(0o644)
+    return release
+
+
+def synthetic_integrity_values() -> dict[str, str]:
+    values: dict[str, str] = {}
+    for key in sorted(BACKUP_INTEGRITY_KEYS):
+        if key == "postgresql.version_num":
+            value = "180006"
+        elif key in {"pointer.entry", "pointer.travel_warning"}:
+            value = "NONE:0"
+        elif key.endswith(".count"):
+            value = "0"
+        else:
+            value = "3" * 64
+        values[key] = value
+    return values
+
+
+def write_backup(directory: Path, archive: bytes = b"synthetic-pg-dump") -> None:
+    directory.mkdir()
+    directory.chmod(0o700)
+    dump = directory / "database.dump"
+    dump.write_bytes(archive)
+    dump.chmod(0o600)
+    digest = hashlib.sha256(archive).hexdigest()
+    lines = [
+        "manifest.format=travel-readiness-postgresql-backup-v1",
+        f"archive.sha256={digest}",
+        "archive.scope=approved-public-schema-without-ephemeral-data",
+    ]
+    for key, value in synthetic_integrity_values().items():
+        lines.append(f"{key}={value}")
+    manifest = directory / "integrity.manifest"
+    manifest.write_text("\n".join(lines) + "\n", encoding="utf-8")
+    manifest.chmod(0o600)
+
+
+class PatternTests(unittest.TestCase):
+    def test_literal_transform_fragment_and_cross_chunk_matches_are_categorized(self):
+        subject = patterns()
+        self.assertEqual(subject.matches(SYNTHETIC_SECRET.encode()), {"secret"})
+        self.assertIn(
+            "secret",
+            subject.matches(quote(SYNTHETIC_SECRET, safe="").encode()),
+        )
+        self.assertIn(
+            "secret",
+            subject.matches(quote_plus(SYNTHETIC_SECRET, safe="").encode()),
+        )
+        lower_escapes = re.sub(
+            r"%[0-9A-F]{2}",
+            lambda match: match.group(0).lower(),
+            quote(SYNTHETIC_SECRET, safe=""),
+        )
+        self.assertIn("secret", subject.matches(lower_escapes.encode()))
+        escape_index = 0
+
+        def alternating_escape_case(match):
+            nonlocal escape_index
+            escape_index += 1
+            return (
+                match.group(0).lower()
+                if escape_index % 2
+                else match.group(0)
+            )
+
+        mixed_escapes = re.sub(
+            r"%[0-9A-F]{2}",
+            alternating_escape_case,
+            quote(SYNTHETIC_SECRET, safe=""),
+        )
+        self.assertIn("secret", subject.matches(mixed_escapes.encode()))
+        decoded_once = SYNTHETIC_SECRET.replace("%2F", "/")
+        self.assertIn("secret", subject.matches(decoded_once.encode()))
+        json_escaped = json.dumps(SYNTHETIC_RAW, ensure_ascii=True).encode()
+        self.assertIn("raw", subject.matches(json_escaped))
+
+        secret_bytes = SYNTHETIC_SECRET.encode()
+        fragment_size = min(32, max(24, len(secret_bytes) // 3))
+        middle = (len(secret_bytes) - fragment_size) // 2
+        for fragment in (
+            secret_bytes[:fragment_size],
+            secret_bytes[middle : middle + fragment_size],
+            secret_bytes[-fragment_size:],
+            secret_bytes[7:31],
+        ):
+            with self.subTest(fragment_position=fragment[:1]):
+                self.assertIn("secret", subject.matches(fragment))
+
+        payload = b"x" * (64 * 1024 - 8) + secret_bytes + b"tail"
+        stream = io.BytesIO(payload)
+        self.assertIn(
+            "secret", subject.stream_matches(stream.read, len(payload))
+        )
+        self.assertFalse(subject.matches(b"ordinary-publication-evidence"))
+
+    def test_fixed_receipt_never_adds_details(self):
+        self.assertEqual(
+            fixed_receipt(
+                {"git": True, "db": False, "artifact": True, "runtime": False}
+            ),
+            "git=ok db=failed artifact=ok runtime=failed",
+        )
+        self.assertEqual(fixed_receipt({"git": True}), FAILED_RECEIPT)
+
+
+class GitBoundaryTests(unittest.TestCase):
+    def test_clean_reachable_history_passes_and_historical_blob_fails(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            repository = initialize_git_repository(Path(temporary))
+            release_sha = commit_file(
+                repository, "public.txt", "safe public fact\n", "initial"
+            )
+            token = f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}"
+            self.assertTrue(
+                scan_git_reachable(repository, release_sha, token, patterns())
+            )
+
+            commit_file(
+                repository,
+                "public.txt",
+                f"historical {SYNTHETIC_SECRET}\n",
+                "synthetic leak",
+            )
+            release_sha = commit_file(
+                repository, "public.txt", "clean current tree\n", "remove leak"
+            )
+            token = f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}"
+            self.assertFalse(
+                scan_git_reachable(repository, release_sha, token, patterns())
+            )
+
+    def test_historical_environment_path_fails_without_target_content(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            repository = initialize_git_repository(Path(temporary))
+            commit_file(repository, ".env.local", "fixture=true\n", "fixture")
+            (repository / ".env.local").unlink()
+            run_git(repository, "add", "--all")
+            run_git(repository, "commit", "-q", "-m", "remove fixture")
+            release_sha = run_git(repository, "rev-parse", "HEAD")
+            self.assertFalse(
+                scan_git_reachable(
+                    repository,
+                    release_sha,
+                    f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}",
+                    patterns(),
+                )
+            )
+
+    def test_reflog_only_synthetic_blob_fails(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            repository = initialize_git_repository(Path(temporary))
+            clean_sha = commit_file(
+                repository, "public.txt", "clean current tree\n", "initial"
+            )
+            commit_file(
+                repository,
+                "public.txt",
+                f"reflog only {SYNTHETIC_SECRET}\n",
+                "synthetic reflog leak",
+            )
+            run_git(repository, "reset", "--hard", clean_sha)
+            self.assertFalse(
+                scan_git_reachable(
+                    repository,
+                    clean_sha,
+                    f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{clean_sha}",
+                    patterns(),
+                )
+            )
+
+    def test_dirty_remote_and_sensitive_ref_names_fail_closed(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            repository = initialize_git_repository(Path(temporary))
+            release_sha = commit_file(
+                repository, "public.txt", "safe\n", "initial"
+            )
+            token = f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}"
+            (repository / "untracked.txt").write_text("safe\n", encoding="utf-8")
+            self.assertFalse(
+                scan_git_reachable(repository, release_sha, token, patterns())
+            )
+            (repository / "untracked.txt").unlink()
+            run_git(
+                repository,
+                "remote",
+                "add",
+                "origin",
+                "https://example.invalid/repository",
+            )
+            self.assertFalse(
+                scan_git_reachable(repository, release_sha, token, patterns())
+            )
+            run_git(repository, "remote", "remove", "origin")
+            run_git(repository, "branch", SYNTHETIC_TRIP)
+            self.assertFalse(
+                scan_git_reachable(repository, release_sha, token, patterns())
+            )
+
+
+class ArtifactBoundaryTests(unittest.TestCase):
+    def test_exact_extracted_tree_passes_and_synthetic_value_fails(self):
+        release_sha = "a" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(
+                Path(temporary), release_sha, b"safe release source\n"
+            )
+            token = expected_artifact_safety_token(release_sha, release)
+            self.assertTrue(
+                scan_extracted_artifact(release, release_sha, token, patterns())
+            )
+
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(
+                Path(temporary), release_sha, SYNTHETIC_SECRET.encode()
+            )
+            token = expected_artifact_safety_token(release_sha, release)
+            self.assertFalse(
+                scan_extracted_artifact(release, release_sha, token, patterns())
+            )
+
+    def test_unmanifested_and_environment_paths_fail_closed(self):
+        release_sha = "b" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(Path(temporary), release_sha, b"safe\n")
+            token = expected_artifact_safety_token(release_sha, release)
+            extra = release / "unmanifested.txt"
+            extra.write_text("safe", encoding="utf-8")
+            extra.chmod(0o644)
+            self.assertFalse(
+                scan_extracted_artifact(release, release_sha, token, patterns())
+            )
+
+    def test_authoritative_manifest_semantics_and_canonical_bytes_are_required(self):
+        release_sha = "8" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(Path(temporary), release_sha, b"safe\n")
+            token = expected_artifact_safety_token(release_sha, release)
+            manifest_path = release / "manifest.json"
+            manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
+            manifest["archive"] = {}
+            manifest_path.write_bytes(canonical_json(manifest))
+            self.assertFalse(
+                scan_extracted_artifact(release, release_sha, token, patterns())
+            )
+
+    def test_artifact_contains_the_authoritative_integrity_query(self):
+        release_sha = "6" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(Path(temporary), release_sha, b"safe\n")
+            query = _artifact_integrity_query(release)
+            self.assertTrue(query.startswith("WITH integrity_lines"))
+            self.assertTrue(query.endswith(";\n"))
+
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            release = write_artifact(Path(temporary), release_sha, b"safe\n")
+            token = expected_artifact_safety_token(release_sha, release)
+            manifest_path = release / "manifest.json"
+            manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
+            manifest["source"]["tracked"][0]["path"] = ".env.local"
+            manifest_path.write_text(json.dumps(manifest), encoding="utf-8")
+            self.assertFalse(
+                scan_extracted_artifact(release, release_sha, token, patterns())
+            )
+
+
+class RuntimeAndBackupBoundaryTests(unittest.TestCase):
+    def test_manifest_allowlist_matches_the_authoritative_integrity_query(self):
+        source = INTEGRITY_SQL.read_text(encoding="utf-8")
+        observed = set(
+            re.findall(
+                r"'((?:postgresql|schema|database|table|pointer)"
+                r"\.[a-z0-9_.]+)='",
+                source,
+            )
+        )
+        self.assertEqual(observed, set(BACKUP_INTEGRITY_KEYS))
+
+    def test_runtime_is_scanned_while_dump_uses_manifest_and_db_evidence(self):
+        release_sha = "c" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            root = Path(temporary)
+            evidence = root / "evidence"
+            evidence.mkdir()
+            (evidence / "health.receipt").write_text(
+                "health=ok\n", encoding="utf-8"
+            )
+            runtime_file = root / "runtime.receipt"
+            runtime_file.write_text("runtime=ok\n", encoding="utf-8")
+            backup = root / "backup"
+            write_backup(backup)
+            token = expected_runtime_safety_token(
+                release_sha, (evidence,), (runtime_file,), backup
+            )
+
+            self.assertTrue(validate_backup_boundary(backup))
+            self.assertTrue(
+                scan_runtime_evidence(
+                    (evidence,),
+                    (runtime_file,),
+                    backup,
+                    release_sha,
+                    token,
+                    patterns(),
+                )
+            )
+            runtime_file.write_text(SYNTHETIC_TRIP, encoding="utf-8")
+            self.assertFalse(
+                scan_runtime_evidence(
+                    (evidence,),
+                    (runtime_file,),
+                    backup,
+                    release_sha,
+                    token,
+                    patterns(),
+                )
+            )
+
+    def test_manifest_key_drift_and_runtime_environment_file_fail(self):
+        release_sha = "d" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            root = Path(temporary)
+            evidence = root / "evidence"
+            evidence.mkdir()
+            forbidden = evidence / ".env.local"
+            forbidden.write_text("fixture=true\n", encoding="utf-8")
+            backup = root / "backup"
+            write_backup(backup)
+            token = expected_runtime_safety_token(
+                release_sha, (evidence,), (), backup
+            )
+            self.assertFalse(
+                scan_runtime_evidence(
+                    (evidence,), (), backup, release_sha, token, patterns()
+                )
+            )
+            forbidden.unlink()
+            manifest = backup / "integrity.manifest"
+            manifest.write_text(
+                manifest.read_text(encoding="utf-8")
+                + f"table.unapproved.count=0\n",
+                encoding="utf-8",
+            )
+            self.assertFalse(validate_backup_boundary(backup))
+
+    def test_runtime_filename_marker_fails_without_output(self):
+        release_sha = "9" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            root = Path(temporary)
+            evidence = root / "evidence"
+            evidence.mkdir()
+            marker_file = evidence / SYNTHETIC_TRIP
+            marker_file.write_text("safe body\n", encoding="utf-8")
+            backup = root / "backup"
+            write_backup(backup)
+            token = expected_runtime_safety_token(
+                release_sha, (evidence,), (), backup
+            )
+            self.assertFalse(
+                scan_runtime_evidence(
+                    (evidence,), (), backup, release_sha, token, patterns()
+                )
+            )
+
+    def test_empty_runtime_and_wrong_path_bound_token_fail(self):
+        release_sha = "7" * 40
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            root = Path(temporary)
+            empty = root / "empty"
+            empty.mkdir()
+            backup = root / "backup"
+            write_backup(backup)
+            empty_token = expected_runtime_safety_token(
+                release_sha, (empty,), (), backup
+            )
+            self.assertFalse(
+                scan_runtime_evidence(
+                    (empty,), (), backup, release_sha, empty_token, patterns()
+                )
+            )
+            evidence = root / "evidence"
+            evidence.mkdir()
+            (evidence / "receipt").write_text("ok\n", encoding="utf-8")
+            self.assertFalse(
+                scan_runtime_evidence(
+                    (evidence,), (), backup, release_sha, empty_token, patterns()
+                )
+            )
+
+
+class FakeControlCursor:
+    def __init__(self, connection):
+        self.connection = connection
+        self.last = ""
+
+    def __enter__(self):
+        return self
+
+    def __exit__(self, *_):
+        return False
+
+    def execute(self, query, params=None):
+        self.last = query if isinstance(query, str) else repr(query)
+        self.connection.executions.append((self.last, params))
+
+    def fetchone(self):
+        if self.last == "SHOW transaction_read_only":
+            return ("on",)
+        if self.last == "SHOW server_version_num":
+            return ("180006",)
+        if self.last == "SELECT current_database() = %s":
+            return (True,)
+        if self.last == "SELECT current_user = %s":
+            return (True,)
+        if "FROM pg_catalog.pg_roles" in self.last:
+            return (True,)
+        if "pg_catalog.pg_largeobject_metadata" in self.last:
+            return (True,)
+        if "FROM pg_catalog.pg_class AS c" in self.last:
+            return (True,)
+        raise AssertionError("unexpected scalar query")
+
+    def fetchall(self):
+        if "pg_catalog.pg_attribute" in self.last:
+            return list(self.connection.columns)
+        if self.last.startswith("WITH integrity_lines"):
+            return [
+                (f"{key}={value}",)
+                for key, value in self.connection.integrity_values.items()
+            ]
+        raise AssertionError("unexpected rows query")
+
+
+class FakeStreamCursor:
+    def __init__(self, connection, index):
+        self.connection = connection
+        self.index = index
+        self.itersize = None
+
+    def __enter__(self):
+        return self
+
+    def __exit__(self, *_):
+        return False
+
+    def execute(self, query, params=None):
+        self.connection.executions.append((repr(query), params))
+
+    def __iter__(self):
+        return iter(self.connection.rows[self.index])
+
+
+class FakeConnection:
+    def __init__(self, columns, rows, integrity_values):
+        self.columns = columns
+        self.rows = rows
+        self.integrity_values = integrity_values
+        self.executions = []
+        self.streams = []
+        self.rolled_back = False
+        self.closed = False
+
+    def cursor(self, name=None):
+        if name is None:
+            return FakeControlCursor(self)
+        stream = FakeStreamCursor(self, int(name.rsplit("_", 1)[1]))
+        self.streams.append(stream)
+        return stream
+
+    def rollback(self):
+        self.rolled_back = True
+
+    def close(self):
+        self.closed = True
+
+
+class FakeConnector:
+    def __init__(self, values):
+        self.values = values
+        self.calls = []
+        self.connections = []
+
+    def __call__(self, **kwargs):
+        self.calls.append(dict(kwargs))
+        columns, rows, integrity_values = self.values[kwargs["dbname"]]
+        connection = FakeConnection(columns, rows, integrity_values)
+        self.connections.append(connection)
+        return connection
+
+
+def database_configuration(
+    backup_directory: Path,
+    *,
+    release_sha: str = "e" * 40,
+) -> ScanConfiguration:
+    configuration = ScanConfiguration(
+        repository_root=REPOSITORY_ROOT,
+        release_sha=release_sha,
+        git_safety_token="unused",
+        artifact_root=REPOSITORY_ROOT,
+        artifact_safety_token="unused",
+        runtime_roots=(),
+        runtime_files=(),
+        backup_directory=backup_directory,
+        runtime_safety_token="unused",
+        db_host="127.0.0.1",
+        db_port=5432,
+        main_db_user="travel_readiness_backup",
+        main_db_password_env=(
+            "TRAVEL_READINESS_SENSITIVE_SCAN_MAIN_DB_PASSWORD"
+        ),
+        restored_db_user="travel_readiness_restorecheck_abcdef_restore",
+        restored_db_password_env=(
+            "TRAVEL_READINESS_SENSITIVE_SCAN_RESTORED_DB_PASSWORD"
+        ),
+        main_database="travel_readiness",
+        restored_database="travel_readiness_restore_abcdef",
+        db_safety_token="pending",
+    )
+    return replace(
+        configuration,
+        db_safety_token=expected_db_safety_token(
+            configuration,
+            hashlib.sha256(b"synthetic-pg-dump").hexdigest(),
+        ),
+    )
+
+
+class DatabaseBoundaryTests(unittest.TestCase):
+    def values(
+        self,
+        *,
+        restored_blob: bytes = b"safe restored bytes",
+        restored_integrity_mismatch: bool = False,
+        restored_rls: bool = False,
+    ):
+        columns = (
+            ("public", "publication", "body", "text", False, False),
+            ("public", "publication", "raw", "bytea", False, False),
+            ("public", "publication", "metadata", "jsonb", False, False),
+        )
+        integrity = synthetic_integrity_values()
+        restored_integrity = dict(integrity)
+        if restored_integrity_mismatch:
+            restored_integrity["table.auth_group.count"] = "1"
+        restored_columns = columns
+        if restored_rls:
+            restored_columns = (
+                ("public", "publication", "body", "text", True, False),
+                *columns[1:],
+            )
+        return {
+            "travel_readiness": (
+                columns,
+                [
+                    [("safe main text",)],
+                    [(b"safe main bytes",)],
+                    [({"state": "published"},)],
+                ],
+                integrity,
+            ),
+            "travel_readiness_restore_abcdef": (
+                restored_columns,
+                [
+                    [("safe restored text",)],
+                    [(restored_blob,)],
+                    [({"state": "published"},)],
+                ],
+                restored_integrity,
+            ),
+        }
+
+    def test_main_and_restored_text_bytea_are_streamed_read_only(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            backup = Path(temporary) / "backup"
+            write_backup(backup)
+            connector = FakeConnector(self.values())
+            self.assertTrue(
+                scan_databases(
+                    database_configuration(backup),
+                    patterns(),
+                    MAIN_DATABASE_PASSWORD,
+                    RESTORED_DATABASE_PASSWORD,
+                    connector=connector,
+                    integrity_query="WITH integrity_lines AS (SELECT 1) SELECT 1",
+                )
+            )
+        self.assertEqual(len(connector.connections), 2)
+        self.assertEqual(
+            [call["user"] for call in connector.calls],
+            [
+                "travel_readiness_backup",
+                "travel_readiness_restorecheck_abcdef_restore",
+            ],
+        )
+        self.assertEqual(
+            [call["password"] for call in connector.calls],
+            [MAIN_DATABASE_PASSWORD, RESTORED_DATABASE_PASSWORD],
+        )
+        for call in connector.calls:
+            self.assertIn("default_transaction_read_only=on", call["options"])
+            self.assertNotIn(SYNTHETIC_SECRET, repr(call))
+        for connection in connector.connections:
+            self.assertTrue(connection.rolled_back)
+            self.assertTrue(connection.closed)
+            self.assertNotIn(SYNTHETIC_SECRET, repr(connection.executions))
+            self.assertTrue(
+                all(cursor.itersize == 128 for cursor in connection.streams)
+            )
+            self.assertEqual(len(connection.streams), 3)
+
+    def test_synthetic_restored_database_value_fails_without_sql_literal(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            backup = Path(temporary) / "backup"
+            write_backup(backup)
+            connector = FakeConnector(
+                self.values(restored_blob=SYNTHETIC_RAW.encode("utf-8"))
+            )
+            self.assertFalse(
+                scan_databases(
+                    database_configuration(backup),
+                    patterns(),
+                    MAIN_DATABASE_PASSWORD,
+                    RESTORED_DATABASE_PASSWORD,
+                    connector=connector,
+                    integrity_query="WITH integrity_lines AS (SELECT 1) SELECT 1",
+                )
+            )
+        for connection in connector.connections:
+            self.assertNotIn(SYNTHETIC_RAW, repr(connection.executions))
+            self.assertTrue(connection.rolled_back)
+            self.assertTrue(connection.closed)
+
+    def test_manifest_snapshot_mismatch_and_rls_fail_closed(self):
+        for values in (
+            self.values(restored_integrity_mismatch=True),
+            self.values(restored_rls=True),
+        ):
+            with self.subTest(case=list(values)[-1]):
+                with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+                    backup = Path(temporary) / "backup"
+                    write_backup(backup)
+                    connector = FakeConnector(values)
+                    self.assertFalse(
+                        scan_databases(
+                            database_configuration(backup),
+                            patterns(),
+                            MAIN_DATABASE_PASSWORD,
+                            RESTORED_DATABASE_PASSWORD,
+                            connector=connector,
+                            integrity_query=(
+                                "WITH integrity_lines AS (SELECT 1) SELECT 1"
+                            ),
+                        )
+                    )
+
+    def test_archive_bound_safety_token_rejects_another_dump(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            backup = Path(temporary) / "backup"
+            write_backup(backup, b"another synthetic dump")
+            connector = FakeConnector(self.values())
+            self.assertFalse(
+                scan_databases(
+                    database_configuration(backup),
+                    patterns(),
+                    MAIN_DATABASE_PASSWORD,
+                    RESTORED_DATABASE_PASSWORD,
+                    connector=connector,
+                    integrity_query="WITH integrity_lines AS (SELECT 1) SELECT 1",
+                )
+            )
+            self.assertEqual(connector.calls, [])
+
+
+class ActualPostgreSQLCatalogTests(unittest.TestCase):
+    def run_postgresql(self, *arguments: str, timeout: int = 30) -> None:
+        subprocess.run(
+            [str(POSTGRESQL_BIN / arguments[0]), *arguments[1:]],
+            env={"PATH": f"{POSTGRESQL_BIN}:/usr/bin:/bin", "LC_ALL": "C"},
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.DEVNULL,
+            stderr=subprocess.DEVNULL,
+            timeout=timeout,
+            check=True,
+        )
+
+    def integrity_query(self) -> str:
+        rows = [
+            f"('{key}={value}')"
+            for key, value in synthetic_integrity_values().items()
+        ]
+        return "SELECT line FROM (VALUES " + ",".join(rows) + ") AS v(line)"
+
+    def test_actual_catalog_visibility_values_rls_and_role_posture(self):
+        required = ("initdb", "pg_ctl", "postgres")
+        if any(not (POSTGRESQL_BIN / name).is_file() for name in required):
+            self.skipTest("PostgreSQL 18.6 binaries unavailable")
+        import psycopg
+
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            root = Path(temporary)
+            data = root / "data"
+            socket_directory = root / "socket"
+            socket_directory.mkdir()
+            with socket.socket() as reservation:
+                reservation.bind(("127.0.0.1", 0))
+                port = reservation.getsockname()[1]
+            self.run_postgresql(
+                "initdb",
+                "-D",
+                str(data),
+                "--no-locale",
+                "--encoding=UTF8",
+                "--auth-local=trust",
+                "--auth-host=trust",
+                "-U",
+                "postgres",
+            )
+            options = (
+                f"-h 127.0.0.1 -p {port} -k {socket_directory} "
+                "-c fsync=off -c synchronous_commit=off "
+                "-c full_page_writes=off"
+            )
+            self.run_postgresql(
+                "pg_ctl",
+                "-D",
+                str(data),
+                "-l",
+                str(root / "postgres.log"),
+                "-o",
+                options,
+                "-w",
+                "start",
+            )
+            try:
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        "CREATE ROLE sensitive_scan_reader LOGIN NOINHERIT"
+                    )
+                    administrator.execute(
+                        "CREATE ROLE sensitive_scan_inheriting LOGIN INHERIT"
+                    )
+                    administrator.execute(
+                        "CREATE DOMAIN public.sensitive_bytes AS bytea"
+                    )
+                    administrator.execute(
+                        """
+                        CREATE TABLE public.actual_catalog_evidence (
+                            id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
+                            body text NOT NULL,
+                            payload bytea NOT NULL,
+                            domain_payload public.sensitive_bytes NOT NULL,
+                            metadata jsonb NOT NULL
+                        )
+                        """
+                    )
+                    administrator.execute(
+                        """
+                        INSERT INTO public.actual_catalog_evidence
+                            (body, payload, domain_payload, metadata)
+                        VALUES (%s, %s, %s, %s)
+                        """,
+                        (
+                            "safe",
+                            b"safe",
+                            b"safe domain",
+                            json.dumps({"state": "safe"}),
+                        ),
+                    )
+                    administrator.execute(
+                        "GRANT USAGE ON SCHEMA public TO sensitive_scan_reader, sensitive_scan_inheriting"
+                    )
+                    administrator.execute(
+                        "GRANT SELECT ON public.actual_catalog_evidence TO sensitive_scan_reader, sensitive_scan_inheriting"
+                    )
+
+                clean, columns, integrity = _scan_one_database(
+                    connector=psycopg.connect,
+                    host="127.0.0.1",
+                    port=port,
+                    user="sensitive_scan_reader",
+                    password="synthetic-unused-password",
+                    database="postgres",
+                    patterns=patterns(),
+                    integrity_query=self.integrity_query(),
+                )
+                self.assertTrue(clean)
+                self.assertTrue(columns)
+                self.assertTrue(
+                    any(column[2] == "domain_payload" for column in columns)
+                )
+                self.assertEqual(integrity, synthetic_integrity_values())
+
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        """
+                        UPDATE public.actual_catalog_evidence
+                        SET domain_payload = %s
+                        """,
+                        (SYNTHETIC_SECRET.encode("utf-8"),),
+                    )
+                clean, _, _ = _scan_one_database(
+                    connector=psycopg.connect,
+                    host="127.0.0.1",
+                    port=port,
+                    user="sensitive_scan_reader",
+                    password="synthetic-unused-password",
+                    database="postgres",
+                    patterns=patterns(),
+                    integrity_query=self.integrity_query(),
+                )
+                self.assertFalse(clean)
+
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        """
+                        UPDATE public.actual_catalog_evidence
+                        SET domain_payload = %s, body = %s
+                        """,
+                        (b"safe domain", SYNTHETIC_SECRET),
+                    )
+                clean, _, _ = _scan_one_database(
+                    connector=psycopg.connect,
+                    host="127.0.0.1",
+                    port=port,
+                    user="sensitive_scan_reader",
+                    password="synthetic-unused-password",
+                    database="postgres",
+                    patterns=patterns(),
+                    integrity_query=self.integrity_query(),
+                )
+                self.assertFalse(clean)
+
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        "UPDATE public.actual_catalog_evidence SET body = 'safe'"
+                    )
+                    administrator.execute(
+                        "ALTER TABLE public.actual_catalog_evidence ENABLE ROW LEVEL SECURITY"
+                    )
+                with self.assertRaises(ScanFailure):
+                    _scan_one_database(
+                        connector=psycopg.connect,
+                        host="127.0.0.1",
+                        port=port,
+                        user="sensitive_scan_reader",
+                        password="synthetic-unused-password",
+                        database="postgres",
+                        patterns=patterns(),
+                        integrity_query=self.integrity_query(),
+                    )
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        "ALTER TABLE public.actual_catalog_evidence DISABLE ROW LEVEL SECURITY"
+                    )
+                    large_object_oid = administrator.execute(
+                        "SELECT pg_catalog.lo_create(0)"
+                    ).fetchone()[0]
+                with self.assertRaises(ScanFailure):
+                    _scan_one_database(
+                        connector=psycopg.connect,
+                        host="127.0.0.1",
+                        port=port,
+                        user="sensitive_scan_reader",
+                        password="synthetic-unused-password",
+                        database="postgres",
+                        patterns=patterns(),
+                        integrity_query=self.integrity_query(),
+                    )
+                with psycopg.connect(
+                    host="127.0.0.1",
+                    port=port,
+                    dbname="postgres",
+                    user="postgres",
+                    autocommit=True,
+                ) as administrator:
+                    administrator.execute(
+                        "SELECT pg_catalog.lo_unlink(%s)", (large_object_oid,)
+                    )
+                with self.assertRaises(ScanFailure):
+                    _scan_one_database(
+                        connector=psycopg.connect,
+                        host="127.0.0.1",
+                        port=port,
+                        user="sensitive_scan_inheriting",
+                        password="synthetic-unused-password",
+                        database="postgres",
+                        patterns=patterns(),
+                        integrity_query=self.integrity_query(),
+                    )
+            finally:
+                try:
+                    self.run_postgresql(
+                        "pg_ctl",
+                        "-D",
+                        str(data),
+                        "-m",
+                        "fast",
+                        "-w",
+                        "stop",
+                    )
+                except (OSError, subprocess.SubprocessError):
+                    self.run_postgresql(
+                        "pg_ctl",
+                        "-D",
+                        str(data),
+                        "-m",
+                        "immediate",
+                        "stop",
+                    )
+
+
+class ConfigurationAndCliTests(unittest.TestCase):
+    def valid_arguments(self, root: Path) -> list[str]:
+        artifact = root / "release"
+        backup = root / "backup"
+        runtime = root / "runtime"
+        runtime_file = root / "single.receipt"
+        for directory in (artifact, runtime):
+            directory.mkdir()
+        write_backup(backup)
+        runtime_file.write_text("receipt=ok\n", encoding="utf-8")
+        release_sha = "f" * 40
+        artifact_token = expected_artifact_safety_token(release_sha, artifact)
+        runtime_token = expected_runtime_safety_token(
+            release_sha, (runtime,), (runtime_file,), backup
+        )
+        database_token = expected_db_safety_token(
+            database_configuration(backup, release_sha=release_sha),
+            hashlib.sha256(b"synthetic-pg-dump").hexdigest(),
+        )
+        return [
+            "--repository-root",
+            str(REPOSITORY_ROOT),
+            "--release-sha",
+            release_sha,
+            "--git-safety-token",
+            f"SENSITIVE_ABSENCE_GIT_READ_ONLY:{release_sha}",
+            "--artifact-root",
+            str(artifact),
+            "--artifact-safety-token",
+            artifact_token,
+            "--runtime-root",
+            str(runtime),
+            "--runtime-file",
+            str(runtime_file),
+            "--backup-directory",
+            str(backup),
+            "--runtime-safety-token",
+            runtime_token,
+            "--db-host",
+            "127.0.0.1",
+            "--db-port",
+            "5432",
+            "--main-db-user",
+            "travel_readiness_backup",
+            "--main-db-password-env",
+            "TRAVEL_READINESS_SENSITIVE_SCAN_MAIN_DB_PASSWORD",
+            "--restored-db-user",
+            "travel_readiness_restorecheck_abcdef_restore",
+            "--restored-db-password-env",
+            "TRAVEL_READINESS_SENSITIVE_SCAN_RESTORED_DB_PASSWORD",
+            "--main-database",
+            "travel_readiness",
+            "--restored-database",
+            "travel_readiness_restore_abcdef",
+            "--db-safety-token",
+            database_token,
+        ]
+
+    def test_exact_allowlisted_arguments_parse_and_prod_like_database_fails(self):
+        with tempfile.TemporaryDirectory(dir="/private/tmp") as temporary:
+            arguments = self.valid_arguments(Path(temporary))
+            configuration = parse_configuration(arguments)
+            self.assertEqual(configuration.repository_root, REPOSITORY_ROOT)
+            index = arguments.index("--restored-database") + 1
+            arguments[index] = "travel_readiness_restore_prod01"
+            with self.assertRaises(ArgumentFailure):
+                parse_configuration(arguments)
+
+    def test_cli_pops_only_env_inputs_and_emits_one_fixed_failure_receipt(self):
+        environment = {
+            "PATH": "/usr/bin:/bin",
+            "MOFA_TRAVEL_ALARM_SERVICE_KEY": SYNTHETIC_SECRET,
+            "TRAVEL_READINESS_SENSITIVE_SCAN_TRIP_MARKERS": json.dumps(
+                [SYNTHETIC_TRIP], ensure_ascii=False
+            ),
+            "TRAVEL_READINESS_SENSITIVE_SCAN_RAW_MARKERS": json.dumps(
+                [SYNTHETIC_RAW], ensure_ascii=False
+            ),
+            "TRAVEL_READINESS_SENSITIVE_SCAN_MAIN_DB_PASSWORD": (
+                MAIN_DATABASE_PASSWORD
+            ),
+            "TRAVEL_READINESS_SENSITIVE_SCAN_RESTORED_DB_PASSWORD": (
+                RESTORED_DATABASE_PASSWORD
+            ),
+            "PGPASSWORD": SYNTHETIC_SECRET,
+            "DATABASE_URL": SYNTHETIC_SECRET,
+        }
+        result = subprocess.run(
+            [str(SCRIPT)],
+            cwd="/private/tmp",
+            env=environment,
+            stdin=subprocess.DEVNULL,
+            stdout=subprocess.PIPE,
+            stderr=subprocess.PIPE,
+            text=True,
+            check=False,
+        )
+        self.assertNotEqual(result.returncode, 0)
+        self.assertEqual(result.stdout, FAILED_RECEIPT + "\n")
+        self.assertEqual(result.stderr, "")
+        for marker in (SYNTHETIC_SECRET, SYNTHETIC_TRIP, SYNTHETIC_RAW):
+            self.assertNotIn(marker, result.stdout + result.stderr)
+
+    def test_secret_pop_precedes_path_and_application_imports(self):
+        source = CLI.read_text(encoding="utf-8")
+        secret_pop = source.index(
+            'os.environ.pop("MOFA_TRAVEL_ALARM_SERVICE_KEY", None)'
+        )
+        self.assertLess(secret_pop, source.index("import sys"))
+        self.assertLess(secret_pop, source.index("from pathlib import Path"))
+        self.assertLess(secret_pop, source.index("def _bootstrap"))
+        self.assertNotIn("dotenv", source.lower())
+        self.assertNotIn(".env.local", source)
+
+    def test_launcher_mode_and_failure_stream_are_fixed(self):
+        self.assertEqual(stat.S_IMODE(SCRIPT.stat().st_mode), 0o755)
+        source = SCRIPT.read_text(encoding="utf-8")
+        self.assertIn('"$entrypoint" "$@" 2>/dev/null', source)
+        self.assertTrue(source.endswith("exit 70\n"))
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/check-sensitive-absence b/scripts/check-sensitive-absence
new file mode 100755
index 0000000..1b7e702
--- /dev/null
+++ b/scripts/check-sensitive-absence
@@ -0,0 +1,34 @@
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
+unset PYTHONHOME PYTHONPATH PYTHONSTARTUP SSLKEYLOGFILE
+IFS=' '
+
+case "$0" in
+  /*) ;;
+  *)
+    printf '%s\n' 'git=failed db=failed artifact=failed runtime=failed'
+    exit 64
+    ;;
+esac
+script_directory=${0%/*}
+repository_directory=${script_directory%/*}
+python_executable=${repository_directory}/.venv/bin/python
+entrypoint=${repository_directory}/operations/sensitive_absence_cli.py
+
+[ -x "$python_executable" ] && [ -f "$entrypoint" ] && [ ! -L "$entrypoint" ] || {
+  printf '%s\n' 'git=failed db=failed artifact=failed runtime=failed'
+  exit 64
+}
+
+set +e
+exec "$python_executable" -I -S -B "$entrypoint" "$@" 2>/dev/null
+printf '%s\n' 'git=failed db=failed artifact=failed runtime=failed'
+exit 70


