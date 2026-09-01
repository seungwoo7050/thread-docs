## `test(publication): close rollback readvance loops`

diff --git a/operations/management/commands/check_publication_rollback.py b/operations/management/commands/check_publication_rollback.py
new file mode 100644
index 0000000..1309335
--- /dev/null
+++ b/operations/management/commands/check_publication_rollback.py
@@ -0,0 +1,730 @@
+"""Prove both publication rollback chains in an exact disposable database."""
+
+from __future__ import annotations
+
+import csv
+import hashlib
+import io
+import json
+import os
+import re
+
+from django.contrib.admin.models import LogEntry
+from django.contrib.auth import get_user_model
+from django.contrib.auth.models import Permission
+from django.contrib.sessions.models import Session
+from django.core.management.base import BaseCommand, CommandError
+from django.db import connection
+from django.test import Client
+from psycopg import sql
+
+from entry_requirements.ingestion import (
+    EntryIngestionCode,
+    ingest_entry_snapshot,
+)
+from entry_requirements.models import EntryFactRevision
+from entry_requirements.parser import ENTRY_HEADERS
+from reviews.models import (
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+    ReviewDecision,
+)
+from reviews.publication import (
+    PublicationCode,
+    publish_candidate,
+    rollback_publication,
+)
+from sources.management.commands.register_approved_sources import (
+    register_approved_sources,
+)
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
+from sources.transport import (
+    ATTEMPT_SUCCEEDED,
+    ENTRY_REQUEST_FINGERPRINT,
+    PROVIDER_SUCCESS_0,
+    WARNING_REQUEST_FINGERPRINT,
+    WARNING_SECRET_REFERENCE,
+    SingleAttemptResult,
+)
+from travel_warnings.ingestion import (
+    TravelWarningIngestionCode,
+    ingest_travel_warning,
+)
+from travel_warnings.models import TravelWarningRevision
+
+
+SUCCESS_RECEIPT = (
+    "publication_rollback_result=VERIFIED modules=2 "
+    "entry_generation=4 warning_generation=4 pointers=match history=match "
+    "ssr=match isolation=match forbidden_verdict=absent sessions=0 "
+    "admin_log=0 trip_storage=absent cookies=trip_absent"
+)
+_SAFETY_ENV = "TRAVEL_READINESS_PUBLICATION_ROLLBACK_SAFETY_TOKEN"
+_SOURCE_SECRET_ENV = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+_DATABASE_RE = re.compile(
+    r"travel_readiness_rollbackcheck_[a-z0-9]{8,24}_db\Z"
+)
+_ENTRY_BASIS = "일반여권 소지자 : rollback synthetic evidence"
+_WARNING_LEVEL = "3"
+_WARNING_SCOPE_TYPE = "일부"
+_MEMORY_ONLY_TOKEN = "rollback-memory-only-token-7d84c2"
+_INPUT_MARKERS = (
+    "ROLLBACK-NONPERSIST-7D84C2",
+    "2088-03-04",
+    "2088-03-05",
+)
+
+
+class _RehearsalFailure(Exception):
+    pass
+
+
+def _require(condition: object) -> None:
+    if not condition:
+        raise _RehearsalFailure
+
+
+def _database_boundary(expected_database: str, safety_token: str) -> None:
+    _require(type(expected_database) is str)
+    _require(_DATABASE_RE.fullmatch(expected_database) is not None)
+    prefix = expected_database.removesuffix("_db")
+    expected_token = f"PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:{prefix}"
+    _require(safety_token == expected_token)
+    environment_token = os.environ.pop(_SAFETY_ENV, None)
+    _require(environment_token == expected_token)
+    _require(os.environ.pop(_SOURCE_SECRET_ENV, None) is None)
+    _require(connection.vendor == "postgresql")
+    _require(connection.get_autocommit() is True)
+    _require(not connection.in_atomic_block)
+    configured_name = connection.settings_dict.get("NAME")
+    _require(configured_name == expected_database)
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT current_database(), current_setting('server_version_num')"
+        )
+        row = cursor.fetchone()
+    _require(row == (expected_database, "180006"))
+
+
+def _trip_storage_columns() -> int:
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT count(*) FROM information_schema.columns "
+            "WHERE table_schema = 'public' AND column_name IN "
+            "('destination', 'departure_date', 'return_date')"
+        )
+        row = cursor.fetchone()
+    _require(row is not None and type(row[0]) is int)
+    return row[0]
+
+
+def _database_contains_fragments(fragments: tuple[str, ...]) -> bool:
+    with connection.cursor() as cursor:
+        cursor.execute(
+            "SELECT table_name, column_name FROM information_schema.columns "
+            "WHERE table_schema = 'public' AND data_type IN "
+            "('character varying', 'character', 'text', 'date', "
+            "'timestamp with time zone', 'timestamp without time zone', "
+            "'json', 'jsonb') ORDER BY table_name, ordinal_position"
+        )
+        columns = tuple(cursor.fetchall())
+        _require(
+            all(
+                type(table) is str and type(column) is str
+                for table, column in columns
+            )
+        )
+        for table, column in columns:
+            statement = sql.SQL(
+                "SELECT EXISTS (SELECT 1 FROM {} WHERE "
+                "position(%s IN {}::text) > 0)"
+            ).format(sql.Identifier(table), sql.Identifier(column))
+            for fragment in fragments:
+                cursor.execute(statement, [fragment])
+                row = cursor.fetchone()
+                _require(row is not None and type(row[0]) is bool)
+                if row[0]:
+                    return True
+    return False
+
+
+def _exercise_input_non_persistence() -> None:
+    client = Client()
+    valid = client.post(
+        "/",
+        {
+            "destination": "JP",
+            "departure_date": _INPUT_MARKERS[1],
+            "return_date": _INPUT_MARKERS[2],
+        },
+        secure=True,
+    )
+    _require(valid.status_code == 303)
+    _require(valid.headers.get("Location") == "/results/")
+    _require(valid.headers.get("Cache-Control") == "no-store")
+    _require(not valid.cookies and not client.cookies)
+
+    invalid = client.post(
+        "/",
+        {
+            "destination": _INPUT_MARKERS[0],
+            "departure_date": _INPUT_MARKERS[1],
+            "return_date": _INPUT_MARKERS[2],
+        },
+        secure=True,
+    )
+    _require(invalid.status_code == 200)
+    _require(invalid.headers.get("Cache-Control") == "no-store")
+    _require("sessionid" not in invalid.cookies)
+    _require("sessionid" not in client.cookies)
+    _require(set(invalid.cookies).issubset({"csrftoken"}))
+    _require(set(client.cookies).issubset({"csrftoken"}))
+    cookie_state = f"{invalid.cookies!r}{client.cookies!r}"
+    _require(
+        all(marker not in cookie_state for marker in _INPUT_MARKERS)
+    )
+    _require(Session.objects.count() == 0)
+    _require(not _database_contains_fragments(_INPUT_MARKERS))
+
+
+def _empty_migrated_database() -> None:
+    entry_pointer = PublishedEntryFacts.objects.get()
+    warning_pointer = PublishedTravelWarning.objects.get()
+    _require(entry_pointer.version == 0)
+    _require(entry_pointer.current_publication_id is None)
+    _require(warning_pointer.version == 0)
+    _require(warning_pointer.current_publication_id is None)
+    models = (
+        SourceConfiguration,
+        SourceRightsDecision,
+        FetchAttempt,
+        SourceArtifact,
+        ParseRun,
+        EntryFactRevision,
+        TravelWarningRevision,
+        ReviewDecision,
+        PublicationRevision,
+        AuditEvent,
+        get_user_model(),
+        Session,
+        LogEntry,
+    )
+    _require(all(model.objects.count() == 0 for model in models))
+    _require(_trip_storage_columns() == 0)
+
+
+def _entry_payload(period: str) -> bytes:
+    row = [""] * len(ENTRY_HEADERS)
+    row[0] = "일본"
+    row[2] = "Y"
+    row[3] = period
+    row[8] = _ENTRY_BASIS
+    output = io.StringIO(newline="")
+    writer = csv.writer(output, lineterminator="\r\n")
+    writer.writerow(ENTRY_HEADERS)
+    writer.writerow(row)
+    return output.getvalue().encode("cp949", errors="strict")
+
+
+def _warning_payload(scope_text: str) -> bytes:
+    document = {
+        "response": {
+            "header": {"resultCode": "0", "resultMsg": "SYNTHETIC"},
+            "body": {
+                "dataType": "JSON",
+                "items": {
+                    "item": [
+                        {
+                            "alarm_lvl": _WARNING_LEVEL,
+                            "continent_cd": "1000",
+                            "continent_eng_nm": "Asia",
+                            "continent_nm": "아시아",
+                            "country_eng_nm": "Japan",
+                            "country_iso_alp2": "JP",
+                            "country_nm": "일본",
+                            "dang_map_download_url": "",
+                            "flag_download_url": "",
+                            "map_download_url": "",
+                            "org_country_idx": "1",
+                            "region_ty": _WARNING_SCOPE_TYPE,
+                            "remark": scope_text,
+                            "written_dt": None,
+                        }
+                    ]
+                },
+                "numOfRows": 1,
+                "pageNo": 1,
+                "totalCount": 1,
+            },
+        }
+    }
+    return json.dumps(
+        document,
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+def _result(payload: bytes, *, warning: bool) -> SingleAttemptResult:
+    return SingleAttemptResult(
+        request_fingerprint=(
+            WARNING_REQUEST_FINGERPRINT if warning else ENTRY_REQUEST_FINGERPRINT
+        ),
+        attempt_result=ATTEMPT_SUCCEEDED,
+        http_status=200,
+        provider_code=PROVIDER_SUCCESS_0 if warning else "",
+        response_sha256=hashlib.sha256(payload).hexdigest(),
+        byte_count=len(payload),
+        body=payload,
+    )
+
+
+def _ingest_entry(period: str) -> EntryFactRevision:
+    payload = _entry_payload(period)
+    called = 0
+
+    def transport(**_kwargs):
+        nonlocal called
+        called += 1
+        return _result(payload, warning=False)
+
+    try:
+        outcome = ingest_entry_snapshot(
+            transport=transport,
+            retry_wait=lambda _attempt: _require(False),
+        )
+    finally:
+        payload = b""
+    _require(called == 1)
+    _require(outcome.code == EntryIngestionCode.REVIEW_REQUIRED)
+    return EntryFactRevision.objects.latest("created_at")
+
+
+def _ingest_warning(scope_text: str) -> TravelWarningRevision:
+    payload = _warning_payload(scope_text)
+    transport_calls = 0
+    secret_calls = 0
+
+    def secret_loader(reference_name: str):
+        nonlocal secret_calls
+        _require(reference_name == WARNING_SECRET_REFERENCE)
+        secret_calls += 1
+        return _MEMORY_ONLY_TOKEN
+
+    def transport(**kwargs):
+        nonlocal transport_calls
+        _require(kwargs.get("secret_reference_name") == WARNING_SECRET_REFERENCE)
+        _require(kwargs.get("secret_value") == _MEMORY_ONLY_TOKEN)
+        transport_calls += 1
+        return _result(payload, warning=True)
+
+    try:
+        outcome = ingest_travel_warning(
+            transport=transport,
+            secret_loader=secret_loader,
+            retry_wait=lambda _attempt: _require(False),
+        )
+    finally:
+        payload = b""
+    _require(secret_calls == 1 and transport_calls == 1)
+    _require(outcome.code == TravelWarningIngestionCode.REVIEW_REQUIRED)
+    return TravelWarningRevision.objects.latest("created_at")
+
+
+def _operator():
+    actor = get_user_model().objects.create_user(
+        username="publication-rollback-rehearsal",
+        password=None,
+        is_staff=True,
+    )
+    permissions = Permission.objects.filter(
+        content_type__app_label="reviews",
+        codename__in=(
+            "publish_entry",
+            "publish_warning",
+            "rollback_entry",
+            "rollback_warning",
+        ),
+    )
+    _require(permissions.count() == 4)
+    actor.user_permissions.add(*permissions)
+    return actor
+
+
+def _card(document: str, card_id: str) -> str:
+    start = document.index(f'<article id="{card_id}"')
+    end = document.index("</article>", start) + len("</article>")
+    return document[start:end]
+
+
+def _render_cards() -> tuple[str, str]:
+    client = Client(enforce_csrf_checks=True)
+    response = client.get("/results/", secure=True)
+    _require(response.status_code == 200)
+    _require(response.headers.get("Cache-Control") == "no-store")
+    _require(not response.cookies and not client.cookies)
+    document = response.content.decode("utf-8", errors="strict")
+    _require(document.count('id="entry-card"') == 1)
+    _require(document.count('id="warning-card"') == 1)
+    forbidden = (
+        "ALLOW" + "ED",
+        "DENI" + "ED",
+        "입국 " + "가능",
+        "법적 " + "판단",
+    )
+    _require(all(phrase not in document for phrase in forbidden))
+    _require(_MEMORY_ONLY_TOKEN not in document)
+    return _card(document, "entry-card"), _card(document, "warning-card")
+
+
+def _assert_entry_card(
+    card: str,
+    *,
+    generation: int,
+    period: str,
+    absent_period: str | None = None,
+    state: str = "ready",
+) -> None:
+    _require(f'data-state="{state}"' in card)
+    _require(f"generation {generation}" in card)
+    _require(period in card and _ENTRY_BASIS in card)
+    _require("외교부|공공데이터포털" in card)
+    if absent_period is not None:
+        _require(absent_period not in card)
+
+
+def _assert_warning_card(
+    card: str,
+    *,
+    generation: int,
+    scope_text: str,
+    absent_text: str | None = None,
+    state: str = "ready",
+) -> None:
+    _require(f'data-state="{state}"' in card)
+    _require(f"generation {generation}" in card)
+    _require(f"source 경보 단계 코드</dt><dd>{_WARNING_LEVEL}" in card)
+    _require(f"source 범위 유형</dt><dd>{_WARNING_SCOPE_TYPE}" in card)
+    _require(scope_text in card)
+    _require("외교부|공공데이터포털" in card)
+    if absent_text is not None:
+        _require(absent_text not in card)
+
+
+def _publication(
+    *,
+    module: str,
+    generation: int,
+    typed_id,
+    supersedes_id,
+    rollback_target_id,
+) -> PublicationRevision:
+    publication = PublicationRevision.objects.get(
+        module=module,
+        generation=generation,
+    )
+    typed_field = (
+        "entry_fact_revision_id"
+        if module == PublicationModule.ENTRY
+        else "travel_warning_revision_id"
+    )
+    _require(getattr(publication, typed_field) == typed_id)
+    _require(publication.supersedes_id == supersedes_id)
+    _require(publication.rollback_target_id == rollback_target_id)
+    pointer = (
+        PublishedEntryFacts.objects.get()
+        if module == PublicationModule.ENTRY
+        else PublishedTravelWarning.objects.get()
+    )
+    _require(pointer.version == generation)
+    _require(pointer.current_publication_id == publication.id)
+    return publication
+
+
+def _expect_outcome(outcome, *, code: str, generation: int) -> None:
+    _require(outcome.code == code)
+    _require(outcome.generation == generation)
+    _require(outcome.pointer_version == generation)
+
+
+def _publish(module: str, revision_id, actor, generation: int):
+    outcome = publish_candidate(
+        module=module,
+        typed_revision_id=revision_id,
+        actor=actor,
+        expected_pointer_version=generation - 1,
+    )
+    _expect_outcome(
+        outcome,
+        code=PublicationCode.PUBLISHED,
+        generation=generation,
+    )
+
+
+def _rollback(module: str, target_id, actor, generation: int):
+    outcome = rollback_publication(
+        module=module,
+        target_publication_id=target_id,
+        actor=actor,
+        expected_pointer_version=generation - 1,
+    )
+    _expect_outcome(
+        outcome,
+        code=PublicationCode.ROLLED_BACK,
+        generation=generation,
+    )
+
+
+def _boundary(pointer) -> tuple[object, int]:
+    pointer.refresh_from_db()
+    return pointer.current_publication_id, pointer.version
+
+
+def run_publication_rollback_rehearsal() -> str:
+    _empty_migrated_database()
+    register_approved_sources(apply=True)
+    actor = _operator()
+    _exercise_input_non_persistence()
+
+    entry_pointer = PublishedEntryFacts.objects.get()
+    warning_pointer = PublishedTravelWarning.objects.get()
+    entry_card_empty, warning_card_empty = _render_cards()
+    _require('data-state="empty"' in entry_card_empty)
+    _require('data-state="empty"' in warning_card_empty)
+
+    entry_first = _ingest_entry("90일")
+    _publish(PublicationModule.ENTRY, entry_first.id, actor, 1)
+    entry_publication_1 = _publication(
+        module=PublicationModule.ENTRY,
+        generation=1,
+        typed_id=entry_first.id,
+        supersedes_id=None,
+        rollback_target_id=None,
+    )
+    entry_card, warning_card_after = _render_cards()
+    _assert_entry_card(entry_card, generation=1, period="90일")
+    _require(warning_card_after != warning_card_empty)
+    _require('data-state="unavailable"' in warning_card_after)
+    _require("아직 검수·게시된 source 사실이 없습니다" not in warning_card_after)
+    _require(_boundary(warning_pointer) == (None, 0))
+    warning_card_unavailable = warning_card_after
+
+    entry_second = _ingest_entry("30일")
+    _publish(PublicationModule.ENTRY, entry_second.id, actor, 2)
+    entry_publication_2 = _publication(
+        module=PublicationModule.ENTRY,
+        generation=2,
+        typed_id=entry_second.id,
+        supersedes_id=entry_publication_1.id,
+        rollback_target_id=None,
+    )
+    entry_card, warning_card_after = _render_cards()
+    _assert_entry_card(
+        entry_card,
+        generation=2,
+        period="30일",
+        absent_period="90일",
+    )
+    _require(warning_card_after == warning_card_unavailable)
+    _require(_boundary(warning_pointer) == (None, 0))
+    entry_card_before = entry_card
+
+    warning_first = _ingest_warning("첫 경보 범위")
+    _publish(PublicationModule.TRAVEL_WARNING, warning_first.id, actor, 1)
+    warning_publication_1 = _publication(
+        module=PublicationModule.TRAVEL_WARNING,
+        generation=1,
+        typed_id=warning_first.id,
+        supersedes_id=None,
+        rollback_target_id=None,
+    )
+    entry_card_after, warning_card = _render_cards()
+    _assert_warning_card(
+        warning_card,
+        generation=1,
+        scope_text="첫 경보 범위",
+    )
+    _require(entry_card_after == entry_card_before)
+    _require(_boundary(entry_pointer) == (entry_publication_2.id, 2))
+
+    warning_second = _ingest_warning("둘째 경보 범위")
+    _publish(PublicationModule.TRAVEL_WARNING, warning_second.id, actor, 2)
+    warning_publication_2 = _publication(
+        module=PublicationModule.TRAVEL_WARNING,
+        generation=2,
+        typed_id=warning_second.id,
+        supersedes_id=warning_publication_1.id,
+        rollback_target_id=None,
+    )
+    entry_card_after, warning_card = _render_cards()
+    _assert_warning_card(
+        warning_card,
+        generation=2,
+        scope_text="둘째 경보 범위",
+        absent_text="첫 경보 범위",
+    )
+    _require(entry_card_after == entry_card_before)
+    warning_boundary = _boundary(warning_pointer)
+
+    _rollback(
+        PublicationModule.ENTRY,
+        entry_publication_1.id,
+        actor,
+        3,
+    )
+    entry_publication_3 = _publication(
+        module=PublicationModule.ENTRY,
+        generation=3,
+        typed_id=entry_first.id,
+        supersedes_id=entry_publication_2.id,
+        rollback_target_id=entry_publication_1.id,
+    )
+    entry_card, warning_card_after = _render_cards()
+    _assert_entry_card(
+        entry_card,
+        generation=3,
+        period="90일",
+        absent_period="30일",
+        state="stale",
+    )
+    _require(warning_card_after == warning_card)
+    _require(_boundary(warning_pointer) == warning_boundary)
+
+    _rollback(
+        PublicationModule.ENTRY,
+        entry_publication_2.id,
+        actor,
+        4,
+    )
+    entry_publication_4 = _publication(
+        module=PublicationModule.ENTRY,
+        generation=4,
+        typed_id=entry_second.id,
+        supersedes_id=entry_publication_3.id,
+        rollback_target_id=entry_publication_2.id,
+    )
+    entry_card, warning_card_after = _render_cards()
+    _assert_entry_card(
+        entry_card,
+        generation=4,
+        period="30일",
+        absent_period="90일",
+    )
+    _require(warning_card_after == warning_card)
+    _require(_boundary(warning_pointer) == warning_boundary)
+
+    entry_boundary = _boundary(entry_pointer)
+    entry_card_before = entry_card
+    _rollback(
+        PublicationModule.TRAVEL_WARNING,
+        warning_publication_1.id,
+        actor,
+        3,
+    )
+    warning_publication_3 = _publication(
+        module=PublicationModule.TRAVEL_WARNING,
+        generation=3,
+        typed_id=warning_first.id,
+        supersedes_id=warning_publication_2.id,
+        rollback_target_id=warning_publication_1.id,
+    )
+    entry_card_after, warning_card = _render_cards()
+    _assert_warning_card(
+        warning_card,
+        generation=3,
+        scope_text="첫 경보 범위",
+        absent_text="둘째 경보 범위",
+        state="stale",
+    )
+    _require(entry_card_after == entry_card_before)
+    _require(_boundary(entry_pointer) == entry_boundary)
+
+    _rollback(
+        PublicationModule.TRAVEL_WARNING,
+        warning_publication_2.id,
+        actor,
+        4,
+    )
+    _publication(
+        module=PublicationModule.TRAVEL_WARNING,
+        generation=4,
+        typed_id=warning_second.id,
+        supersedes_id=warning_publication_3.id,
+        rollback_target_id=warning_publication_2.id,
+    )
+    entry_card_after, warning_card = _render_cards()
+    _assert_warning_card(
+        warning_card,
+        generation=4,
+        scope_text="둘째 경보 범위",
+        absent_text="첫 경보 범위",
+    )
+    _require(entry_card_after == entry_card_before)
+    _require(_boundary(entry_pointer) == entry_boundary)
+
+    _require(PublicationRevision.objects.count() == 8)
+    _require(ReviewDecision.objects.count() == 8)
+    _require(
+        AuditEvent.objects.filter(
+            outcome=AuditEvent.Outcome.SUCCEEDED,
+            action=AuditEvent.Action.PUBLISH,
+        ).count()
+        == 4
+    )
+    _require(
+        AuditEvent.objects.filter(
+            outcome=AuditEvent.Outcome.SUCCEEDED,
+            action=AuditEvent.Action.ROLLBACK,
+        ).count()
+        == 4
+    )
+    _require(FetchAttempt.objects.count() == 4)
+    _require(SourceArtifact.objects.count() == 4)
+    _require(ParseRun.objects.count() == 4)
+    _require(EntryFactRevision.objects.count() == 2)
+    _require(TravelWarningRevision.objects.count() == 2)
+    _require(
+        not SourceRightsDecision.objects.filter(
+            raw_body_storage_allowed=True
+        ).exists()
+    )
+    _require(Session.objects.count() == 0)
+    _require(LogEntry.objects.count() == 0)
+    _require(_trip_storage_columns() == 0)
+    _require(
+        not _database_contains_fragments(
+            (*_INPUT_MARKERS, _MEMORY_ONLY_TOKEN)
+        )
+    )
+    return SUCCESS_RECEIPT
+
+
+class Command(BaseCommand):
+    help = "Verify entry and warning rollback chains in a disposable database."
+
+    def add_arguments(self, parser):
+        parser.add_argument("--expected-database", required=True)
+        parser.add_argument("--safety-token", required=True)
+
+    def handle(self, *args, **options):
+        try:
+            _database_boundary(
+                options["expected_database"],
+                options["safety_token"],
+            )
+            receipt = run_publication_rollback_rehearsal()
+            _require(receipt == SUCCESS_RECEIPT)
+        except Exception:
+            raise CommandError(
+                "publication_rollback_result=FAILED"
+            ) from None
+        self.stdout.write(SUCCESS_RECEIPT)
diff --git a/operations/tests/test_publication_rollback_rehearsal.py b/operations/tests/test_publication_rollback_rehearsal.py
new file mode 100644
index 0000000..42c7f1f
--- /dev/null
+++ b/operations/tests/test_publication_rollback_rehearsal.py
@@ -0,0 +1,649 @@
+from __future__ import annotations
+
+import os
+from pathlib import Path
+import shutil
+import stat
+import subprocess
+import tempfile
+import textwrap
+import unittest
+
+from django.test import TransactionTestCase
+
+from countries.models import Country, JP_COUNTRY_ID
+from entry_requirements.models import (
+    PASSPORT_POLICY_CODE,
+    PASSPORT_POLICY_ID,
+    PASSPORT_POLICY_REVISION,
+    PassportPolicy,
+)
+from operations.management.commands.check_publication_rollback import (
+    SUCCESS_RECEIPT,
+    run_publication_rollback_rehearsal,
+)
+from reviews.models import (
+    ENTRY_POINTER_ID,
+    WARNING_POINTER_ID,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+)
+
+
+class PublicationRollbackRunnerContractTests(unittest.TestCase):
+    @classmethod
+    def setUpClass(cls):
+        super().setUpClass()
+        cls.root = Path(__file__).resolve().parents[2]
+        cls.script = cls.root / "scripts" / "check-publication-rollback"
+        cls.prefix = "travel_readiness_rollbackcheck_unit12345"
+        cls.database = f"{cls.prefix}_db"
+        cls.safety_token = (
+            f"PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:{cls.prefix}"
+        )
+        cls.release_sha = "a" * 40
+        cls.outer_receipt = (
+            f"{SUCCESS_RECEIPT} release_identity=match "
+            "postgresql=18.6 cleanup=match\n"
+        )
+
+    def arguments(self, **overrides):
+        values = {
+            "release_sha": self.release_sha,
+            "host": "127.0.0.1",
+            "port": "5432",
+            "admin_role": "postgres",
+            "admin_password_env": (
+                "TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD"
+            ),
+            "database_prefix": self.prefix,
+            "safety_token": self.safety_token,
+        }
+        values.update(overrides)
+        return [
+            "--release-sha",
+            values["release_sha"],
+            "--host",
+            values["host"],
+            "--port",
+            values["port"],
+            "--admin-role",
+            values["admin_role"],
+            "--admin-password-env",
+            values["admin_password_env"],
+            "--database-prefix",
+            values["database_prefix"],
+            "--safety-token",
+            values["safety_token"],
+        ]
+
+    def run_script(self, script=None, *arguments, env=None, cwd=None):
+        return subprocess.run(
+            ["/bin/sh", str(script or self.script), *arguments],
+            cwd=cwd or self.root,
+            env=env,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=20,
+        )
+
+    def test_script_is_executable_and_help_is_fixed(self):
+        self.assertEqual(stat.S_IMODE(self.script.stat().st_mode), 0o755)
+        result = self.run_script(None, "--help")
+        self.assertEqual(result.returncode, 0)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(
+            result.stdout,
+            "usage: check-publication-rollback --release-sha SHA "
+            "--host LOOPBACK --port PORT --admin-role ROLE "
+            "--admin-password-env "
+            "TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD "
+            "--database-prefix travel_readiness_rollbackcheck_NAME "
+            "--safety-token PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:"
+            "travel_readiness_rollbackcheck_NAME\n",
+        )
+
+    def test_arguments_and_target_fail_closed_before_secret_access(self):
+        cases = (
+            ((), 64, "publication_rollback_check=INVALID_ARGUMENTS\n"),
+            (
+                ("--unknown",),
+                64,
+                "publication_rollback_check=INVALID_ARGUMENTS\n",
+            ),
+            (
+                tuple(self.arguments(host="remote.invalid")),
+                65,
+                "publication_rollback_check=NON_LOOPBACK_REFUSED\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_prefix="travel_readiness",
+                        safety_token=(
+                            "PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:"
+                            "travel_readiness"
+                        ),
+                    )
+                ),
+                65,
+                "publication_rollback_check=UNSAFE_DATABASE\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_prefix=(
+                            "travel_readiness_rollbackcheck_short"
+                        ),
+                        safety_token=(
+                            "PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:"
+                            "travel_readiness_rollbackcheck_short"
+                        ),
+                    )
+                ),
+                65,
+                "publication_rollback_check=UNSAFE_DATABASE\n",
+            ),
+            (
+                tuple(
+                    self.arguments(
+                        database_prefix=(
+                            "travel_readiness_rollbackcheck_prod12345"
+                        ),
+                        safety_token=(
+                            "PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:"
+                            "travel_readiness_rollbackcheck_prod12345"
+                        ),
+                    )
+                ),
+                65,
+                (
+                    "publication_rollback_check="
+                    "PRODUCTION_LIKE_DATABASE_REFUSED\n"
+                ),
+            ),
+            (
+                tuple(self.arguments(safety_token="wrong")),
+                65,
+                "publication_rollback_check=SAFETY_TOKEN_MISMATCH\n",
+            ),
+            (
+                tuple(self.arguments(release_sha="A" * 40)),
+                65,
+                "publication_rollback_check=INVALID_RELEASE_SHA\n",
+            ),
+        )
+        for arguments, code, stderr in cases:
+            with self.subTest(stderr=stderr):
+                result = self.run_script(None, *arguments, env={})
+                self.assertEqual(result.returncode, code)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, stderr)
+
+    def test_missing_admin_password_is_fixed_and_redacted(self):
+        environment = os.environ.copy()
+        environment.pop(
+            "TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD",
+            None,
+        )
+        result = self.run_script(
+            None,
+            *self.arguments(),
+            env=environment,
+        )
+        self.assertEqual(result.returncode, 66)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "publication_rollback_check=ADMIN_PASSWORD_MISSING\n",
+        )
+
+    def _fake_project(self, temporary: Path):
+        project = temporary / "project"
+        scripts = project / "scripts"
+        python_dir = project / ".venv" / "bin"
+        fake_bin = temporary / "bin"
+        scripts.mkdir(parents=True)
+        python_dir.mkdir(parents=True)
+        fake_bin.mkdir()
+        copied_script = scripts / "check-publication-rollback"
+        shutil.copy2(self.script, copied_script)
+        shutil.copy2(
+            self.root / "scripts" / "postgresql-common",
+            scripts / "postgresql-common",
+        )
+        (project / "manage.py").write_text(
+            "# test placeholder\n",
+            encoding="utf-8",
+        )
+
+        fake_python = python_dir / "python"
+        fake_python.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 91
+                case " $* " in
+                    *" check_publication_rollback "*)
+                        [ "${TRAVEL_READINESS_PUBLICATION_ROLLBACK_SAFETY_TOKEN:-}" = "${FAKE_SAFETY_TOKEN}" ] || exit 92
+                        [ "${TRAVEL_READINESS_DB_NAME:-}" = "${FAKE_DATABASE}" ] || exit 93
+                        [ "${TRAVEL_READINESS_RELEASE_SHA:-}" = "${FAKE_RELEASE_SHA}" ] || exit 94
+                        printf '%s\n' rollback >> "$FAKE_LOG"
+                        [ "${FAKE_REHEARSAL_FAIL:-0}" = 0 ] || exit 95
+                        printf '%s\n' "$FAKE_INNER_RECEIPT"
+                        ;;
+                    *)
+                        [ -z "${TRAVEL_READINESS_PUBLICATION_ROLLBACK_SAFETY_TOKEN:-}" ] || exit 96
+                        printf '%s\n' manage >> "$FAKE_LOG"
+                        [ "${FAKE_MANAGE_FAIL:-0}" = 0 ] || exit 97
+                        ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_python.chmod(0o755)
+
+        fake_psql = fake_bin / "psql"
+        fake_psql.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                if [ "${1:-}" = "--version" ]; then
+                    printf 'psql (PostgreSQL) %s\n' "${FAKE_CLIENT_VERSION:-18.6}"
+                    exit 0
+                fi
+                [ -z "${MOFA_TRAVEL_ALARM_SERVICE_KEY:-}" ] || exit 98
+                [ -z "${TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD:-}" ] || exit 99
+                input=$(cat)
+                combined="$* $input"
+                case "$combined" in
+                    *"current_setting('server_version_num')"*) printf '%s\n' "${FAKE_SERVER_VERSION:-180006}" ;;
+                    *"SELECT rolsuper"*) printf '%s\n' t ;;
+                    *"CREATE DATABASE"*)
+                        : > "$FAKE_PG_STATE"
+                        printf '%s\n' create >> "$FAKE_LOG"
+                        ;;
+                    *"DROP DATABASE"*)
+                        if [ "${FAKE_DROP_FAIL_ONCE:-0}" = 1 ] && [ ! -e "$FAKE_DROP_MARKER" ]; then
+                            : > "$FAKE_DROP_MARKER"
+                            printf '%s\n' drop-failed >> "$FAKE_LOG"
+                            exit 101
+                        fi
+                        rm -f "$FAKE_PG_STATE"
+                        printf '%s\n' drop >> "$FAKE_LOG"
+                        ;;
+                    *"SELECT count(*) FROM pg_catalog.pg_database"*)
+                        if [ -e "$FAKE_PG_STATE" ]; then
+                            printf '%s\n' 1
+                        else
+                            printf '%s\n' 0
+                        fi
+                        ;;
+                    *) : ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_psql.chmod(0o755)
+
+        fake_git = fake_bin / "git"
+        fake_git.write_text(
+            textwrap.dedent(
+                """\
+                #!/bin/sh
+                set -eu
+                case "$*" in
+                    *"rev-parse --verify HEAD"*) printf '%s\n' "$FAKE_RELEASE_SHA" ;;
+                    *"status --porcelain"*)
+                        [ "${FAKE_DIRTY:-0}" = 0 ] || printf '%s\n' ' M marker'
+                        ;;
+                    *) exit 100 ;;
+                esac
+                """
+            ),
+            encoding="utf-8",
+        )
+        fake_git.chmod(0o755)
+
+        fake_openssl = fake_bin / "openssl"
+        fake_openssl.write_text(
+            "#!/bin/sh\nprintf '%064d\\n' 0\n",
+            encoding="utf-8",
+        )
+        fake_openssl.chmod(0o755)
+        return project, copied_script, fake_bin
+
+    def _fake_environment(self, temporary: Path, fake_bin: Path):
+        environment = os.environ.copy()
+        environment.update(
+            {
+                "PATH": f"{fake_bin}:/usr/bin:/bin",
+                (
+                    "TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD"
+                ): "admin-private-marker",
+                "MOFA_TRAVEL_ALARM_SERVICE_KEY": "source-private-marker",
+                "FAKE_SAFETY_TOKEN": self.safety_token,
+                "FAKE_DATABASE": self.database,
+                "FAKE_RELEASE_SHA": self.release_sha,
+                "FAKE_INNER_RECEIPT": SUCCESS_RECEIPT,
+                "FAKE_PG_STATE": str(temporary / "database-created"),
+                "FAKE_DROP_MARKER": str(temporary / "drop-attempted"),
+                "FAKE_LOG": str(temporary / "events.log"),
+            }
+        )
+        return environment
+
+    def test_fake_rehearsal_is_ordered_fixed_and_always_cleans(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+            events = Path(environment["FAKE_LOG"]).read_text(
+                encoding="utf-8"
+            ).splitlines()
+
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(result.stdout, self.outer_receipt)
+        self.assertFalse(state_exists)
+        self.assertEqual(
+            events,
+            ["create", "manage", "manage", "manage", "rollback", "drop"],
+        )
+        rendered = result.stdout + result.stderr
+        self.assertNotIn("admin-private-marker", rendered)
+        self.assertNotIn("source-private-marker", rendered)
+        self.assertNotIn(self.safety_token, rendered)
+
+    def test_rehearsal_and_migration_failures_still_clean(self):
+        cases = (
+            (
+                "FAKE_REHEARSAL_FAIL",
+                "1",
+                75,
+                "publication_rollback_check=REHEARSAL_FAILED\n",
+                ["rollback", "drop"],
+            ),
+            (
+                "FAKE_MANAGE_FAIL",
+                "1",
+                73,
+                "publication_rollback_check=MIGRATION_PLAN_FAILED\n",
+                ["create", "manage", "drop"],
+            ),
+            (
+                "FAKE_INNER_RECEIPT",
+                "unexpected",
+                75,
+                "publication_rollback_check=RECEIPT_MISMATCH\n",
+                ["rollback", "drop"],
+            ),
+        )
+        for failure_flag, value, code, stderr, expected_events in cases:
+            with self.subTest(failure=failure_flag):
+                with tempfile.TemporaryDirectory() as temporary_name:
+                    temporary = Path(temporary_name)
+                    project, script, fake_bin = self._fake_project(temporary)
+                    environment = self._fake_environment(temporary, fake_bin)
+                    environment[failure_flag] = value
+                    result = self.run_script(
+                        script,
+                        *self.arguments(),
+                        env=environment,
+                        cwd=project,
+                    )
+                    state_exists = Path(
+                        environment["FAKE_PG_STATE"]
+                    ).exists()
+                    events = Path(environment["FAKE_LOG"]).read_text(
+                        encoding="utf-8"
+                    ).splitlines()
+
+                self.assertEqual(result.returncode, code)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, stderr)
+                self.assertFalse(state_exists)
+                if failure_flag != "FAKE_MANAGE_FAIL":
+                    self.assertEqual(events[-2:], expected_events)
+                else:
+                    self.assertEqual(events, expected_events)
+
+    def test_cleanup_trap_retries_drop_and_leaves_no_target(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            environment["FAKE_DROP_FAIL_ONCE"] = "1"
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = Path(environment["FAKE_PG_STATE"]).exists()
+            events = Path(environment["FAKE_LOG"]).read_text(
+                encoding="utf-8"
+            ).splitlines()
+
+        self.assertEqual(result.returncode, 77)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "publication_rollback_check_cleanup=FAILED\n",
+        )
+        self.assertFalse(state_exists)
+        self.assertEqual(events[-3:], ["rollback", "drop-failed", "drop"])
+
+    def test_release_identity_rejections_precede_database_creation(self):
+        cases = (
+            (
+                "FAKE_DIRTY",
+                "1",
+                "publication_rollback_check=WORKTREE_NOT_CLEAN\n",
+            ),
+            (
+                "FAKE_RELEASE_SHA",
+                "b" * 40,
+                "publication_rollback_check=RELEASE_IDENTITY_MISMATCH\n",
+            ),
+        )
+        for name, value, stderr in cases:
+            with self.subTest(name=name):
+                with tempfile.TemporaryDirectory() as temporary_name:
+                    temporary = Path(temporary_name)
+                    project, script, fake_bin = self._fake_project(temporary)
+                    environment = self._fake_environment(temporary, fake_bin)
+                    environment[name] = value
+                    result = self.run_script(
+                        script,
+                        *self.arguments(),
+                        env=environment,
+                        cwd=project,
+                    )
+                    state_exists = Path(
+                        environment["FAKE_PG_STATE"]
+                    ).exists()
+                    log_exists = Path(environment["FAKE_LOG"]).exists()
+
+                self.assertEqual(result.returncode, 70)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, stderr)
+                self.assertFalse(state_exists)
+                self.assertFalse(log_exists)
+
+    def test_postgresql_client_and_server_must_be_exactly_18_6(self):
+        cases = (
+            (
+                "FAKE_CLIENT_VERSION",
+                "18.5",
+                69,
+                "publication_rollback_check=POSTGRESQL_18_6_REQUIRED\n",
+            ),
+            (
+                "FAKE_SERVER_VERSION",
+                "180005",
+                70,
+                "publication_rollback_check=DATABASE_VERSION_MISMATCH\n",
+            ),
+        )
+        for name, value, code, stderr in cases:
+            with self.subTest(name=name):
+                with tempfile.TemporaryDirectory() as temporary_name:
+                    temporary = Path(temporary_name)
+                    project, script, fake_bin = self._fake_project(temporary)
+                    environment = self._fake_environment(temporary, fake_bin)
+                    environment[name] = value
+                    result = self.run_script(
+                        script,
+                        *self.arguments(),
+                        env=environment,
+                        cwd=project,
+                    )
+                    state_exists = Path(
+                        environment["FAKE_PG_STATE"]
+                    ).exists()
+                    log_exists = Path(environment["FAKE_LOG"]).exists()
+
+                self.assertEqual(result.returncode, code)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(result.stderr, stderr)
+                self.assertFalse(state_exists)
+                self.assertFalse(log_exists)
+
+    def test_preexisting_target_is_never_mutated_or_cleaned(self):
+        with tempfile.TemporaryDirectory() as temporary_name:
+            temporary = Path(temporary_name)
+            project, script, fake_bin = self._fake_project(temporary)
+            environment = self._fake_environment(temporary, fake_bin)
+            state = Path(environment["FAKE_PG_STATE"])
+            state.touch(mode=0o600)
+            result = self.run_script(
+                script,
+                *self.arguments(),
+                env=environment,
+                cwd=project,
+            )
+            state_exists = state.exists()
+            log_exists = Path(environment["FAKE_LOG"]).exists()
+
+        self.assertEqual(result.returncode, 71)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "publication_rollback_check=TARGET_ALREADY_EXISTS\n",
+        )
+        self.assertTrue(state_exists)
+        self.assertFalse(log_exists)
+
+    def test_sources_have_no_external_transport_or_dotenv_path(self):
+        wrapper = self.script.read_text(encoding="utf-8")
+        command = (
+            self.root
+            / "operations"
+            / "management"
+            / "commands"
+            / "check_publication_rollback.py"
+        ).read_text(encoding="utf-8")
+        lower = f"{wrapper}\n{command}".lower()
+        self.assertNotIn(".env.local", lower)
+        self.assertNotIn("curl", lower)
+        self.assertNotIn("requests.", lower)
+        self.assertNotIn("urllib", lower)
+        self.assertNotIn("fetch_entry_attachment", command)
+        self.assertNotIn("fetch_travel_alarm_jp", command)
+        self.assertNotIn("set -x", lower)
+        self.assertIn("cleanup_on_exit", wrapper)
+        self.assertIn("check_publication_rollback", wrapper)
+        self.assertIn("ingest_entry_snapshot", command)
+        self.assertIn("ingest_travel_warning", command)
+
+
+class PublicationRollbackFlowTests(TransactionTestCase):
+    reset_sequences = True
+
+    def setUp(self):
+        country, _ = Country.objects.get_or_create(
+            id=JP_COUNTRY_ID,
+            defaults={
+                "iso_alpha2": "JP",
+                "name_ko": "일본",
+                "name_en": "Japan",
+                "is_public": True,
+            },
+        )
+        self.assertEqual(
+            (
+                country.iso_alpha2,
+                country.name_ko,
+                country.name_en,
+                country.is_public,
+            ),
+            ("JP", "일본", "Japan", True),
+        )
+        policy, _ = PassportPolicy.objects.get_or_create(
+            id=PASSPORT_POLICY_ID,
+            defaults={
+                "code": PASSPORT_POLICY_CODE,
+                "revision": PASSPORT_POLICY_REVISION,
+            },
+        )
+        self.assertEqual(
+            (policy.code, policy.revision),
+            (PASSPORT_POLICY_CODE, PASSPORT_POLICY_REVISION),
+        )
+        entry_pointer, _ = PublishedEntryFacts.objects.get_or_create(
+            id=ENTRY_POINTER_ID,
+            defaults={
+                "country_id": JP_COUNTRY_ID,
+                "passport_policy_id": PASSPORT_POLICY_ID,
+                "current_publication_id": None,
+                "version": 0,
+            },
+        )
+        self.assertEqual(
+            (
+                entry_pointer.country_id,
+                entry_pointer.passport_policy_id,
+                entry_pointer.current_publication_id,
+                entry_pointer.version,
+            ),
+            (JP_COUNTRY_ID, PASSPORT_POLICY_ID, None, 0),
+        )
+        warning_pointer, _ = PublishedTravelWarning.objects.get_or_create(
+            id=WARNING_POINTER_ID,
+            defaults={
+                "country_id": JP_COUNTRY_ID,
+                "current_publication_id": None,
+                "version": 0,
+            },
+        )
+        self.assertEqual(
+            (
+                warning_pointer.country_id,
+                warning_pointer.current_publication_id,
+                warning_pointer.version,
+            ),
+            (JP_COUNTRY_ID, None, 0),
+        )
+
+    def test_both_modules_rollback_and_readvance_with_ssr_isolation(self):
+        self.assertEqual(
+            run_publication_rollback_rehearsal(),
+            SUCCESS_RECEIPT,
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/check-publication-rollback b/scripts/check-publication-rollback
new file mode 100755
index 0000000..2c72f3c
--- /dev/null
+++ b/scripts/check-publication-rollback
@@ -0,0 +1,272 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS
+unset PGPASSWORD PGPASSFILE MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
+
+INNER_RECEIPT='publication_rollback_result=VERIFIED modules=2 entry_generation=4 warning_generation=4 pointers=match history=match ssr=match isolation=match forbidden_verdict=absent sessions=0 admin_log=0 trip_storage=absent cookies=trip_absent'
+
+usage() {
+    printf '%s\n' 'usage: check-publication-rollback --release-sha SHA --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD --database-prefix travel_readiness_rollbackcheck_NAME --safety-token PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:travel_readiness_rollbackcheck_NAME'
+}
+
+fail() {
+    printf '%s\n' "$1" >&2
+    exit "$2"
+}
+
+is_identifier() {
+    value=$1
+    [ -n "$value" ] || return 1
+    [ "${#value}" -le 63 ] || return 1
+    case "$value" in [a-z_]*) ;; *) return 1 ;; esac
+    case "$value" in *[!a-z0-9_]*) return 1 ;; esac
+}
+
+is_port() {
+    value=$1
+    case "$value" in ''|*[!0-9]*) return 1 ;; esac
+    [ "${#value}" -le 5 ] || return 1
+    [ "$value" -ge 1 ] 2>/dev/null || return 1
+    [ "$value" -le 65535 ] 2>/dev/null || return 1
+}
+
+is_release_sha() {
+    value=$1
+    [ "${#value}" -eq 40 ] || return 1
+    case "$value" in *[!0-9a-f]*) return 1 ;; esac
+}
+
+release_sha=''
+host=''
+port=''
+admin_role=''
+admin_password_env=''
+database_prefix=''
+safety_token=''
+
+while [ "$#" -gt 0 ]; do
+    case "$1" in
+        --help)
+            [ "$#" -eq 1 ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --release-sha|--host|--port|--admin-role|--admin-password-env|--database-prefix|--safety-token)
+            [ "$#" -ge 2 ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64
+            option=$1
+            option_value=$2
+            shift 2
+            case "$option" in
+                --release-sha) [ -z "$release_sha" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; release_sha=$option_value ;;
+                --host) [ -z "$host" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; host=$option_value ;;
+                --port) [ -z "$port" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; port=$option_value ;;
+                --admin-role) [ -z "$admin_role" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; admin_role=$option_value ;;
+                --admin-password-env) [ -z "$admin_password_env" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; admin_password_env=$option_value ;;
+                --database-prefix) [ -z "$database_prefix" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; database_prefix=$option_value ;;
+                --safety-token) [ -z "$safety_token" ] || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64; safety_token=$option_value ;;
+            esac
+            ;;
+        *) fail 'publication_rollback_check=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+[ -n "$release_sha" ] && [ -n "$host" ] && [ -n "$port" ] \
+    && [ -n "$admin_role" ] && [ -n "$admin_password_env" ] \
+    && [ -n "$database_prefix" ] && [ -n "$safety_token" ] \
+    || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64
+is_release_sha "$release_sha" \
+    || fail 'publication_rollback_check=INVALID_RELEASE_SHA' 65
+case "$host" in 127.0.0.1|localhost) ;; *) fail 'publication_rollback_check=NON_LOOPBACK_REFUSED' 65 ;; esac
+is_port "$port" || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64
+is_identifier "$admin_role" || fail 'publication_rollback_check=INVALID_ARGUMENTS' 64
+is_identifier "$database_prefix" \
+    || fail 'publication_rollback_check=UNSAFE_DATABASE' 65
+case "$database_prefix" in
+    travel_readiness_rollbackcheck_*) ;;
+    *) fail 'publication_rollback_check=UNSAFE_DATABASE' 65 ;;
+esac
+database_suffix=${database_prefix#travel_readiness_rollbackcheck_}
+[ "${#database_suffix}" -ge 8 ] && [ "${#database_suffix}" -le 24 ] \
+    || fail 'publication_rollback_check=UNSAFE_DATABASE' 65
+case "$database_suffix" in *[!a-z0-9]*) fail 'publication_rollback_check=UNSAFE_DATABASE' 65 ;; esac
+case "$database_prefix" in
+    *prod*|*live*|*stag*|*main*|*master*|*release*)
+        fail 'publication_rollback_check=PRODUCTION_LIKE_DATABASE_REFUSED' 65
+        ;;
+esac
+[ "$safety_token" = "PUBLICATION_ROLLBACK_REHEARSAL_DISPOSABLE:$database_prefix" ] \
+    || fail 'publication_rollback_check=SAFETY_TOKEN_MISMATCH' 65
+[ "$admin_password_env" = 'TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD' ] \
+    || fail 'publication_rollback_check=UNSAFE_PASSWORD_REFERENCE' 65
+
+admin_password=${TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD-}
+unset TRAVEL_READINESS_PUBLICATION_ROLLBACK_ADMIN_PASSWORD
+[ -n "$admin_password" ] \
+    || fail 'publication_rollback_check=ADMIN_PASSWORD_MISSING' 66
+[ "${#admin_password}" -le 1024 ] \
+    || fail 'publication_rollback_check=ADMIN_PASSWORD_INVALID' 66
+case "$admin_password" in
+    *'
+'*) fail 'publication_rollback_check=ADMIN_PASSWORD_INVALID' 66 ;;
+esac
+
+database="${database_prefix}_db"
+is_identifier "$database" || fail 'publication_rollback_check=UNSAFE_DATABASE' 65
+
+script_dir=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+project_dir=$(CDPATH='' cd "$script_dir/.." && pwd -P)
+python_bin="$project_dir/.venv/bin/python"
+# shellcheck source-path=SCRIPTDIR
+# shellcheck source=postgresql-common
+. "$script_dir/postgresql-common"
+
+command -v git >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=REQUIRED_TOOL_MISSING' 69
+command -v openssl >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=REQUIRED_TOOL_MISSING' 69
+[ -x "$python_bin" ] \
+    || fail 'publication_rollback_check=PINNED_PYTHON_REQUIRED' 69
+
+resolved_sha=$(git -C "$project_dir" rev-parse --verify HEAD 2>/dev/null) \
+    || fail 'publication_rollback_check=RELEASE_IDENTITY_FAILED' 70
+[ "$resolved_sha" = "$release_sha" ] \
+    || fail 'publication_rollback_check=RELEASE_IDENTITY_MISMATCH' 70
+worktree_state=$(git -C "$project_dir" status --porcelain=v1 \
+    --untracked-files=normal 2>/dev/null) \
+    || fail 'publication_rollback_check=RELEASE_IDENTITY_FAILED' 70
+[ -z "$worktree_state" ] \
+    || fail 'publication_rollback_check=WORKTREE_NOT_CLEAN' 70
+
+require_pg_tool psql \
+    || fail 'publication_rollback_check=POSTGRESQL_18_6_REQUIRED' 69
+
+admin_psql() {
+    connection_database=$1
+    shift
+    PGPASSWORD="$admin_password" \
+    PGAPPNAME=travel-readiness-publication-rollback-admin \
+    PGCONNECT_TIMEOUT=5 \
+        psql --no-password --host="$host" --port="$port" \
+        --dbname="$connection_database" --username="$admin_role" \
+        --no-psqlrc --set=ON_ERROR_STOP=1 "$@"
+}
+
+admin_scalar() {
+    query=$1
+    admin_psql postgres --quiet --tuples-only --no-align \
+        --command="$query" 2>/dev/null
+}
+
+server_version=$(admin_scalar "SELECT current_setting('server_version_num')") \
+    || fail 'publication_rollback_check=ADMIN_CONNECTION_FAILED' 70
+[ "$server_version" = 180006 ] \
+    || fail 'publication_rollback_check=DATABASE_VERSION_MISMATCH' 70
+admin_superuser=$(admin_scalar \
+    "SELECT rolsuper FROM pg_catalog.pg_roles WHERE rolname = current_user") \
+    || fail 'publication_rollback_check=ADMIN_CONNECTION_FAILED' 70
+[ "$admin_superuser" = t ] \
+    || fail 'publication_rollback_check=ADMIN_CAPABILITY_REQUIRED' 70
+existing_target=$(admin_scalar \
+    "SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$database'") \
+    || fail 'publication_rollback_check=TARGET_PREFLIGHT_FAILED' 70
+[ "$existing_target" = 0 ] \
+    || fail 'publication_rollback_check=TARGET_ALREADY_EXISTS' 71
+
+django_secret=$(openssl rand -hex 32 2>/dev/null) \
+    || fail 'publication_rollback_check=SECRET_GENERATION_FAILED' 69
+[ "${#django_secret}" -eq 64 ] \
+    || fail 'publication_rollback_check=SECRET_GENERATION_FAILED' 69
+case "$django_secret" in *[!0-9a-f]*) fail 'publication_rollback_check=SECRET_GENERATION_FAILED' 69 ;; esac
+
+database_created=0
+
+cleanup() {
+    cleanup_result=0
+    if [ "$database_created" = 1 ]; then
+        admin_psql postgres --quiet \
+            --command="REVOKE CONNECT ON DATABASE \"$database\" FROM PUBLIC" \
+            >/dev/null 2>&1 || :
+        admin_psql postgres --quiet \
+            --command="SELECT pg_catalog.pg_terminate_backend(pid) FROM pg_catalog.pg_stat_activity WHERE datname = '$database' AND pid <> pg_catalog.pg_backend_pid()" \
+            >/dev/null 2>&1 || :
+        admin_psql postgres --quiet \
+            --command="DROP DATABASE \"$database\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+    fi
+    remaining=$(admin_scalar \
+        "SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$database'" \
+        2>/dev/null) || cleanup_result=1
+    if [ "${remaining:-1}" = 0 ]; then
+        database_created=0
+        cleanup_result=0
+    else
+        cleanup_result=1
+    fi
+    return "$cleanup_result"
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    if ! cleanup; then
+        printf '%s\n' 'publication_rollback_check_cleanup=FAILED' >&2
+        exit 77
+    fi
+    exit "$original_status"
+}
+
+trap cleanup_on_exit EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+printf '%s\n' \
+    "CREATE DATABASE \"$database\" OWNER \"$admin_role\" TEMPLATE template0" \
+    | admin_psql postgres --quiet >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=DATABASE_CREATE_FAILED' 72
+database_created=1
+
+run_manage() {
+    TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+    TRAVEL_READINESS_DB_NAME="$database" \
+    TRAVEL_READINESS_DB_USER="$admin_role" \
+    TRAVEL_READINESS_DB_PASSWORD="$admin_password" \
+    TRAVEL_READINESS_DB_HOST="$host" \
+    TRAVEL_READINESS_DB_PORT="$port" \
+    TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+    TRAVEL_READINESS_RELEASE_SHA="$release_sha" \
+    TRAVEL_READINESS_BUILD=0 \
+    TRAVEL_READINESS_DEBUG=0 \
+    TRAVEL_READINESS_HTTPS=0 \
+    DJANGO_SETTINGS_MODULE=travel_readiness.settings \
+    PYTHONPATH="$project_dir" \
+        "$python_bin" -s "$project_dir/manage.py" "$@"
+}
+
+run_manage migrate --plan --noinput --verbosity 0 >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=MIGRATION_PLAN_FAILED' 73
+run_manage migrate --noinput --verbosity 0 >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=MIGRATION_FAILED' 73
+run_manage makemigrations --check --dry-run --verbosity 0 >/dev/null 2>&1 \
+    || fail 'publication_rollback_check=MIGRATION_DRIFT' 73
+
+rehearsal_receipt=$( \
+    TRAVEL_READINESS_PUBLICATION_ROLLBACK_SAFETY_TOKEN="$safety_token" \
+    run_manage check_publication_rollback \
+        --expected-database "$database" \
+        --safety-token "$safety_token" \
+        --verbosity 0 2>/dev/null \
+) || fail 'publication_rollback_check=REHEARSAL_FAILED' 75
+[ "$rehearsal_receipt" = "$INNER_RECEIPT" ] \
+    || fail 'publication_rollback_check=RECEIPT_MISMATCH' 75
+
+cleanup || fail 'publication_rollback_check_cleanup=FAILED' 77
+trap - EXIT HUP INT TERM
+unset admin_password django_secret
+printf '%s\n' "$INNER_RECEIPT release_identity=match postgresql=18.6 cleanup=match"
