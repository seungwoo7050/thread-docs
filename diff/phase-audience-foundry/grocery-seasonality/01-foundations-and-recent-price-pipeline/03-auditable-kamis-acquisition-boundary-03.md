## `feat(source): load local key without disclosure`

diff --git a/grocery/source/secrets.py b/grocery/source/secrets.py
new file mode 100644
index 0000000..e6b12c1
--- /dev/null
+++ b/grocery/source/secrets.py
@@ -0,0 +1,152 @@
+"""Fail-closed loading for the local KAMIS development credential.
+
+The loader does not source a shell or retain file contents.  Callers must make an
+explicit ``reveal()`` at the narrow HTTP invocation boundary.
+"""
+
+from __future__ import annotations
+
+import os
+import stat
+from collections.abc import Mapping
+from pathlib import Path
+from typing import Final
+
+KAMIS_API_KEY: Final = "KAMIS_API_KEY"
+DEFAULT_LOCAL_SECRET_PATH: Final = Path(".env.local")
+_MAX_SECRET_FILE_BYTES: Final = 16 * 1024
+_REDACTED: Final = "<redacted>"
+
+
+class SecretLoadError(RuntimeError):
+    """A credential-loading failure whose message is a non-sensitive code."""
+
+    def __init__(self, code: str) -> None:
+        self.code = code
+        super().__init__(code)
+
+
+class SecretValue:
+    """A deliberately redacted wrapper around credential material."""
+
+    __slots__ = ("__value",)
+
+    def __init__(self, value: str) -> None:
+        self.__value = value
+
+    def __repr__(self) -> str:
+        return _REDACTED
+
+    def __str__(self) -> str:
+        return _REDACTED
+
+    def __format__(self, format_spec: str) -> str:
+        del format_spec
+        return _REDACTED
+
+    def reveal(self) -> str:
+        """Reveal the value only for immediate use by the KAMIS HTTP client."""
+
+        return self.__value
+
+
+def load_kamis_api_key(
+    *,
+    environment: Mapping[str, str] | None = None,
+    path: Path = DEFAULT_LOCAL_SECRET_PATH,
+) -> SecretValue:
+    """Load KAMIS_API_KEY from the process environment or a strict local file."""
+
+    selected_environment = os.environ if environment is None else environment
+    if KAMIS_API_KEY in selected_environment:
+        return SecretValue(
+            _validate_value(selected_environment[KAMIS_API_KEY], source="environment")
+        )
+
+    contents = _read_secret_file(path)
+    return SecretValue(_parse_secret_file(contents))
+
+
+def _read_secret_file(path: Path) -> str:
+    try:
+        before_open = path.lstat()
+    except FileNotFoundError:
+        raise SecretLoadError("secret_file_missing") from None
+    except OSError:
+        raise SecretLoadError("secret_file_unreadable") from None
+
+    if stat.S_ISLNK(before_open.st_mode):
+        raise SecretLoadError("secret_file_symlink")
+    _validate_file_metadata(before_open)
+
+    flags = os.O_RDONLY | getattr(os, "O_CLOEXEC", 0) | getattr(os, "O_NOFOLLOW", 0)
+    try:
+        descriptor = os.open(path, flags)
+    except OSError:
+        raise SecretLoadError("secret_file_unreadable") from None
+
+    try:
+        after_open = os.fstat(descriptor)
+        _validate_file_metadata(after_open)
+        if (before_open.st_dev, before_open.st_ino) != (after_open.st_dev, after_open.st_ino):
+            raise SecretLoadError("secret_file_changed")
+        raw = os.read(descriptor, _MAX_SECRET_FILE_BYTES + 1)
+    except SecretLoadError:
+        raise
+    except OSError:
+        raise SecretLoadError("secret_file_unreadable") from None
+    finally:
+        os.close(descriptor)
+
+    if len(raw) > _MAX_SECRET_FILE_BYTES:
+        raise SecretLoadError("secret_file_too_large")
+    try:
+        return raw.decode("utf-8", errors="strict")
+    except UnicodeDecodeError:
+        raise SecretLoadError("secret_file_invalid_encoding") from None
+
+
+def _validate_file_metadata(metadata: os.stat_result) -> None:
+    if not stat.S_ISREG(metadata.st_mode):
+        raise SecretLoadError("secret_file_not_regular")
+    if metadata.st_uid != os.geteuid():
+        raise SecretLoadError("secret_file_wrong_owner")
+    if stat.S_IMODE(metadata.st_mode) & 0o077:
+        raise SecretLoadError("secret_file_permissions")
+
+
+def _parse_secret_file(contents: str) -> str:
+    if "\x00" in contents:
+        raise SecretLoadError("secret_file_nul")
+
+    found: str | None = None
+    prefix = f"{KAMIS_API_KEY}="
+    for line in contents.splitlines():
+        if not line or line.startswith("#"):
+            continue
+        if not line.startswith(prefix):
+            raise SecretLoadError("secret_file_malformed")
+        if found is not None:
+            raise SecretLoadError("secret_key_duplicate")
+        found = _validate_value(line.removeprefix(prefix), source="file")
+
+    if found is None:
+        raise SecretLoadError("secret_key_missing")
+    return found
+
+
+def _validate_value(value: str, *, source: str) -> str:
+    if not value:
+        code = "environment_value_empty" if source == "environment" else "secret_key_empty"
+        raise SecretLoadError(code)
+    if "\x00" in value or any(character.isspace() for character in value):
+        code = (
+            "environment_value_invalid" if source == "environment" else "secret_key_unsafe_syntax"
+        )
+        raise SecretLoadError(code)
+    if any(character in value for character in ("$", "`", "'", '"', "\\")):
+        code = (
+            "environment_value_invalid" if source == "environment" else "secret_key_unsafe_syntax"
+        )
+        raise SecretLoadError(code)
+    return value
diff --git a/grocery/tests/test_source_secrets.py b/grocery/tests/test_source_secrets.py
new file mode 100644
index 0000000..68ca71f
--- /dev/null
+++ b/grocery/tests/test_source_secrets.py
@@ -0,0 +1,167 @@
+"""Synthetic tests for local credential loading; the real .env.local is never read."""
+
+from __future__ import annotations
+
+import os
+from pathlib import Path
+
+import pytest
+
+from grocery.source.secrets import SecretLoadError, load_kamis_api_key
+
+SYNTHETIC_SECRET = "synthetic+credential/segment="
+
+
+def _secret_file(tmp_path: Path, contents: str, *, mode: int = 0o600) -> Path:
+    path = tmp_path / "synthetic.env.local"
+    path.write_text(contents, encoding="utf-8")
+    path.chmod(mode)
+    return path
+
+
+def _assert_error_is_redacted(error: SecretLoadError) -> None:
+    rendered = f"{error!s} {error!r}"
+    assert SYNTHETIC_SECRET not in rendered
+    assert "synthetic.env.local" not in rendered
+
+
+def test_process_environment_wins_without_reading_the_file(tmp_path: Path) -> None:
+    insecure_path = _secret_file(tmp_path, "not a valid assignment", mode=0o644)
+
+    secret = load_kamis_api_key(
+        environment={"KAMIS_API_KEY": SYNTHETIC_SECRET},
+        path=insecure_path,
+    )
+
+    assert str(secret) == "<redacted>"
+    assert repr(secret) == "<redacted>"
+    assert f"{secret}" == "<redacted>"
+    assert secret.reveal() == SYNTHETIC_SECRET
+
+
+def test_owner_only_file_accepts_one_exact_assignment(tmp_path: Path) -> None:
+    path = _secret_file(
+        tmp_path,
+        f"# local-only credential\n\nKAMIS_API_KEY={SYNTHETIC_SECRET}\n",
+    )
+
+    secret = load_kamis_api_key(environment={}, path=path)
+
+    assert repr(secret) == "<redacted>"
+    assert secret.reveal() == SYNTHETIC_SECRET
+
+
+@pytest.mark.parametrize(
+    ("contents", "code"),
+    [
+        ("", "secret_key_missing"),
+        ("KAMIS_API_KEY=", "secret_key_empty"),
+        ("export KAMIS_API_KEY=anything", "secret_file_malformed"),
+        ("OTHER_KEY=anything", "secret_file_malformed"),
+        (" KAMIS_API_KEY=anything", "secret_file_malformed"),
+        ("KAMIS_API_KEY=$OTHER_KEY", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY=`command`", "secret_key_unsafe_syntax"),
+        ('KAMIS_API_KEY="quoted"', "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY='quoted'", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY=escaped\\value", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY= leading", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY=trailing ", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY=embedded\tspace", "secret_key_unsafe_syntax"),
+        ("KAMIS_API_KEY=first\nKAMIS_API_KEY=second", "secret_key_duplicate"),
+        ("KAMIS_API_KEY=first\ncontinued-secret", "secret_file_malformed"),
+        ("KAMIS_API_KEY=first\x00second", "secret_file_nul"),
+    ],
+)
+def test_file_syntax_fails_with_code_only(tmp_path: Path, contents: str, code: str) -> None:
+    path = _secret_file(tmp_path, contents)
+
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=path)
+
+    assert raised.value.code == code
+    assert str(raised.value) == code
+    _assert_error_is_redacted(raised.value)
+
+
+@pytest.mark.parametrize(
+    ("value", "code"),
+    [
+        ("", "environment_value_empty"),
+        (" ", "environment_value_invalid"),
+        ("$OTHER_KEY", "environment_value_invalid"),
+        ("line-one\nline-two", "environment_value_invalid"),
+    ],
+)
+def test_invalid_environment_value_fails_with_code_only(
+    tmp_path: Path, value: str, code: str
+) -> None:
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(
+            environment={"KAMIS_API_KEY": value},
+            path=tmp_path / "must-not-be-read",
+        )
+
+    assert raised.value.code == code
+    assert str(raised.value) == code
+    _assert_error_is_redacted(raised.value)
+
+
+def test_group_or_other_permissions_are_rejected(tmp_path: Path) -> None:
+    path = _secret_file(tmp_path, f"KAMIS_API_KEY={SYNTHETIC_SECRET}\n", mode=0o640)
+
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=path)
+
+    assert raised.value.code == "secret_file_permissions"
+    _assert_error_is_redacted(raised.value)
+
+
+def test_symlink_is_rejected_without_following_it(tmp_path: Path) -> None:
+    target = _secret_file(tmp_path, f"KAMIS_API_KEY={SYNTHETIC_SECRET}\n")
+    link = tmp_path / "secret-link"
+    link.symlink_to(target)
+
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=link)
+
+    assert raised.value.code == "secret_file_symlink"
+    _assert_error_is_redacted(raised.value)
+
+
+def test_non_regular_path_is_rejected(tmp_path: Path) -> None:
+    directory = tmp_path / "not-a-file"
+    directory.mkdir(mode=0o700)
+
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=directory)
+
+    assert raised.value.code == "secret_file_not_regular"
+
+
+def test_invalid_utf8_is_rejected_without_leaking_bytes(tmp_path: Path) -> None:
+    path = tmp_path / "synthetic.env.local"
+    path.write_bytes(b"KAMIS_API_KEY=synthetic\xffcredential")
+    path.chmod(0o600)
+
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=path)
+
+    assert raised.value.code == "secret_file_invalid_encoding"
+    assert "synthetic" not in repr(raised.value)
+
+
+def test_missing_file_error_contains_only_a_code(tmp_path: Path) -> None:
+    with pytest.raises(SecretLoadError) as raised:
+        load_kamis_api_key(environment={}, path=tmp_path / "synthetic.env.local")
+
+    assert str(raised.value) == "secret_file_missing"
+    _assert_error_is_redacted(raised.value)
+
+
+def test_owner_execute_bit_does_not_grant_group_or_other_access(tmp_path: Path) -> None:
+    path = _secret_file(tmp_path, f"KAMIS_API_KEY={SYNTHETIC_SECRET}\n", mode=0o700)
+
+    secret = load_kamis_api_key(environment={}, path=path)
+
+    assert secret.reveal() == SYNTHETIC_SECRET
+    assert os.stat(path).st_mode & 0o077 == 0


## `feat(source): seal approved source configuration`

diff --git a/grocery/source/configuration.py b/grocery/source/configuration.py
new file mode 100644
index 0000000..4f02c60
--- /dev/null
+++ b/grocery/source/configuration.py
@@ -0,0 +1,122 @@
+"""Deterministic database bootstrap for the approved KAMIS A-path source.
+
+This module contains only public configuration metadata and the logical name of the
+credential.  It deliberately does not import or invoke the credential loader.
+"""
+
+from __future__ import annotations
+
+import uuid
+from datetime import datetime
+from types import MappingProxyType
+from typing import Final
+from zoneinfo import ZoneInfo
+
+from django.db import transaction
+
+from grocery.models import SourceConfiguration
+from grocery.source.client import (
+    CONNECT_READ_TIMEOUT_SECONDS,
+    KAMIS_ENDPOINT,
+    MAX_ATTEMPTS_PER_PAGE,
+    MAX_CALLS,
+    MAX_PAGE_BYTES,
+    MAX_PAGES,
+)
+from grocery.source.kamis import KAMIS_RETAIL_COVERAGE_IDENTITY
+from grocery.source.registry import IDENTITY_EVIDENCE_REVISION, OFFICIAL_DOCS_ZIP_SHA256
+
+KAMIS_DATASET_ID: Final = "15156063"
+KAMIS_CONFIGURATION_REVISION: Final = "kamis-15156063-recent-comparison-v1"
+KAMIS_INTERFACE_REVISION: Final = "recent-price-v1"
+KAMIS_ENDPOINT_HOST: Final = "apis.data.go.kr"
+KAMIS_ENDPOINT_PATH: Final = "/B552845/recent/price"
+KAMIS_LOGICAL_SECRET_NAME: Final = "KAMIS_API_KEY"  # noqa: S105 - logical reference only
+KAMIS_RIGHTS_EVIDENCE_LOCATOR: Final = "https://www.data.go.kr/data/15156063/openapi.do"
+KAMIS_GATE_CONFIRMED_AT: Final = datetime(
+    2026,
+    8,
+    30,
+    tzinfo=ZoneInfo("Asia/Seoul"),
+)
+KAMIS_SOURCE_CONFIGURATION_ID: Final = uuid.uuid5(
+    uuid.NAMESPACE_URL,
+    f"{KAMIS_RIGHTS_EVIDENCE_LOCATOR}#{KAMIS_CONFIGURATION_REVISION}",
+)
+
+_EXPECTED_CONTRACT_FIELDS: Final = MappingProxyType(
+    {
+        "source_owner_name": "한국농수산식품유통공사",
+        "dataset_id": KAMIS_DATASET_ID,
+        "configuration_revision": KAMIS_CONFIGURATION_REVISION,
+        "interface_revision": KAMIS_INTERFACE_REVISION,
+        "state": SourceConfiguration.State.ACTIVE,
+        "state_changed_at": KAMIS_GATE_CONFIRMED_AT,
+        "publication_mode": SourceConfiguration.PublicationMode.RECENT_COMPARISON,
+        "coverage_identity": KAMIS_RETAIL_COVERAGE_IDENTITY,
+        "coverage_evidence_revision": IDENTITY_EVIDENCE_REVISION,
+        "endpoint_scheme": SourceConfiguration.EndpointScheme.HTTPS,
+        "endpoint_host": KAMIS_ENDPOINT_HOST,
+        "endpoint_path": KAMIS_ENDPOINT_PATH,
+        "endpoint_method": SourceConfiguration.EndpointMethod.GET,
+        "authentication_mode": (SourceConfiguration.AuthenticationMode.DATA_GO_KR_SERVICE_KEY),
+        "logical_secret_name": KAMIS_LOGICAL_SECRET_NAME,
+        "provider_quota_limit": 10_000,
+        "provider_quota_period": SourceConfiguration.QuotaPeriod.UNSPECIFIED,
+        "request_timeout_seconds": int(CONNECT_READ_TIMEOUT_SECONDS),
+        "retry_policy": SourceConfiguration.RetryPolicy.BOUNDED_TRANSIENT_ONLY,
+        "max_retries": MAX_ATTEMPTS_PER_PAGE - 1,
+        "max_requests_per_attempt": MAX_CALLS,
+        "max_pages_per_attempt": MAX_PAGES,
+        "max_page_bytes": MAX_PAGE_BYTES,
+        "rights_evidence_locator": KAMIS_RIGHTS_EVIDENCE_LOCATOR,
+        "rights_evidence_sha256": OFFICIAL_DOCS_ZIP_SHA256,
+        "rights_confirmed_at": KAMIS_GATE_CONFIRMED_AT,
+        "raw_retention": SourceConfiguration.RawRetention.HASH_ONLY,
+    }
+)
+
+
+class SourceConfigurationDriftError(RuntimeError):
+    """A same-revision row differs from the sealed, public-only contract."""
+
+    def __init__(self, field_names: tuple[str, ...]) -> None:
+        self.field_names = field_names
+        super().__init__(f"source_configuration_drift fields={','.join(field_names)}")
+
+
+def bootstrap_kamis_source_configuration() -> SourceConfiguration:
+    """Create the approved A-path configuration once, or validate its exact revision.
+
+    The comparison reports field names only.  No credential is loaded, compared, or
+    included in an exception.
+    """
+
+    expected_endpoint = (
+        f"{SourceConfiguration.EndpointScheme.HTTPS}://{KAMIS_ENDPOINT_HOST}{KAMIS_ENDPOINT_PATH}"
+    )
+    if KAMIS_ENDPOINT != expected_endpoint:
+        raise SourceConfigurationDriftError(("transport_endpoint",))
+
+    defaults = dict(_EXPECTED_CONTRACT_FIELDS)
+    defaults.pop("dataset_id")
+    defaults.pop("configuration_revision")
+    defaults["id"] = KAMIS_SOURCE_CONFIGURATION_ID
+
+    with transaction.atomic():
+        source, _created = SourceConfiguration.objects.select_for_update().get_or_create(
+            dataset_id=KAMIS_DATASET_ID,
+            configuration_revision=KAMIS_CONFIGURATION_REVISION,
+            defaults=defaults,
+        )
+        drifted_fields = tuple(
+            sorted(
+                field_name
+                for field_name, expected_value in _EXPECTED_CONTRACT_FIELDS.items()
+                if getattr(source, field_name) != expected_value
+            )
+        )
+        if drifted_fields:
+            raise SourceConfigurationDriftError(drifted_fields)
+
+    return source
diff --git a/grocery/tests/test_source_configuration.py b/grocery/tests/test_source_configuration.py
new file mode 100644
index 0000000..397abab
--- /dev/null
+++ b/grocery/tests/test_source_configuration.py
@@ -0,0 +1,90 @@
+from unittest.mock import patch
+
+import pytest
+
+from grocery.models import SourceConfiguration
+from grocery.source.client import MAX_CALLS, MAX_PAGE_BYTES, MAX_PAGES
+from grocery.source.configuration import (
+    KAMIS_CONFIGURATION_REVISION,
+    KAMIS_DATASET_ID,
+    KAMIS_ENDPOINT_HOST,
+    KAMIS_ENDPOINT_PATH,
+    KAMIS_GATE_CONFIRMED_AT,
+    KAMIS_LOGICAL_SECRET_NAME,
+    KAMIS_RIGHTS_EVIDENCE_LOCATOR,
+    KAMIS_SOURCE_CONFIGURATION_ID,
+    SourceConfigurationDriftError,
+    bootstrap_kamis_source_configuration,
+)
+from grocery.source.kamis import KAMIS_RETAIL_COVERAGE_IDENTITY
+from grocery.source.registry import IDENTITY_EVIDENCE_REVISION, OFFICIAL_DOCS_ZIP_SHA256
+
+pytestmark = pytest.mark.django_db
+
+
+def test_bootstrap_seals_the_approved_a_path_contract_without_loading_a_secret() -> None:
+    with patch("grocery.source.secrets.load_kamis_api_key") as secret_loader:
+        source = bootstrap_kamis_source_configuration()
+
+    secret_loader.assert_not_called()
+    assert source.pk == KAMIS_SOURCE_CONFIGURATION_ID
+    assert source.source_owner_name == "한국농수산식품유통공사"
+    assert source.dataset_id == KAMIS_DATASET_ID
+    assert source.configuration_revision == KAMIS_CONFIGURATION_REVISION
+    assert source.interface_revision == "recent-price-v1"
+    assert source.state == SourceConfiguration.State.ACTIVE
+    assert source.state_changed_at == KAMIS_GATE_CONFIRMED_AT
+    assert source.publication_mode == SourceConfiguration.PublicationMode.RECENT_COMPARISON
+    assert source.coverage_identity == KAMIS_RETAIL_COVERAGE_IDENTITY
+    assert source.coverage_evidence_revision == IDENTITY_EVIDENCE_REVISION
+    assert source.endpoint_scheme == SourceConfiguration.EndpointScheme.HTTPS
+    assert source.endpoint_host == KAMIS_ENDPOINT_HOST
+    assert source.endpoint_path == KAMIS_ENDPOINT_PATH
+    assert source.endpoint_method == SourceConfiguration.EndpointMethod.GET
+    assert (
+        source.authentication_mode == SourceConfiguration.AuthenticationMode.DATA_GO_KR_SERVICE_KEY
+    )
+    assert source.logical_secret_name == KAMIS_LOGICAL_SECRET_NAME
+    assert source.provider_quota_limit == 10_000
+    assert source.provider_quota_period == SourceConfiguration.QuotaPeriod.UNSPECIFIED
+    assert source.request_timeout_seconds == 10
+    assert source.retry_policy == SourceConfiguration.RetryPolicy.BOUNDED_TRANSIENT_ONLY
+    assert source.max_retries == 2
+    assert source.max_requests_per_attempt == MAX_CALLS == 12
+    assert source.max_pages_per_attempt == MAX_PAGES == 12
+    assert source.max_page_bytes == MAX_PAGE_BYTES == 4 * 1024 * 1024
+    assert source.rights_evidence_locator == KAMIS_RIGHTS_EVIDENCE_LOCATOR
+    assert source.rights_evidence_sha256 == OFFICIAL_DOCS_ZIP_SHA256
+    assert source.rights_confirmed_at == KAMIS_GATE_CONFIRMED_AT
+    assert source.raw_retention == SourceConfiguration.RawRetention.HASH_ONLY
+
+
+def test_bootstrap_is_idempotent_for_the_same_sealed_revision() -> None:
+    first = bootstrap_kamis_source_configuration()
+    second = bootstrap_kamis_source_configuration()
+
+    assert second.pk == first.pk
+    assert SourceConfiguration.objects.count() == 1
+
+
+def test_bootstrap_fails_closed_on_same_revision_field_drift_without_values() -> None:
+    source = bootstrap_kamis_source_configuration()
+    SourceConfiguration.objects.filter(pk=source.pk).update(endpoint_path="/changed/public/path")
+
+    with pytest.raises(SourceConfigurationDriftError) as caught:
+        bootstrap_kamis_source_configuration()
+
+    assert caught.value.field_names == ("endpoint_path",)
+    assert str(caught.value) == "source_configuration_drift fields=endpoint_path"
+    assert "/changed/public/path" not in str(caught.value)
+
+
+def test_bootstrap_does_not_modify_an_existing_drifted_revision() -> None:
+    source = bootstrap_kamis_source_configuration()
+    SourceConfiguration.objects.filter(pk=source.pk).update(state=SourceConfiguration.State.PAUSED)
+
+    with pytest.raises(SourceConfigurationDriftError):
+        bootstrap_kamis_source_configuration()
+
+    source.refresh_from_db()
+    assert source.state == SourceConfiguration.State.PAUSED


