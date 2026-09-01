## `feat(publication): require operator review for flights`

diff --git a/travel_windows/admin.py b/travel_windows/admin.py
new file mode 100644
index 0000000..d648300
--- /dev/null
+++ b/travel_windows/admin.py
@@ -0,0 +1,225 @@
+from django import forms
+from django.contrib import admin, messages
+from django.contrib.admin.helpers import ActionForm
+
+from reviews.admin import operator_admin_site
+
+from .models import (
+    FlightAuditEvent,
+    FlightPublication,
+    FlightReviewDecision,
+    FlightScheduleRevision,
+)
+from .publication import (
+    FlightPublicationCode,
+    approve_flight_candidate,
+    reject_flight_candidate,
+    rollback_flight_publication,
+)
+
+
+class _FlightActionForm(ActionForm):
+    expected_pointer_version = forms.IntegerField(
+        required=False,
+        min_value=0,
+        label="확인한 현재 포인터 버전",
+    )
+
+
+class _ReadOnlyAdmin(admin.ModelAdmin):
+    actions = ()
+    list_per_page = 50
+
+    def has_add_permission(self, request):
+        return False
+
+    def has_change_permission(self, request, obj=None):
+        return False
+
+    def has_delete_permission(self, request, obj=None):
+        return False
+
+
+def _authorized(request, permission: str) -> bool:
+    try:
+        user = request.user
+        return bool(
+            user.is_active and user.is_staff and user.has_perm(permission)
+        )
+    except Exception:
+        return False
+
+
+def _single_id(queryset):
+    values = list(queryset.values_list("pk", flat=True)[:2])
+    return values[0] if len(values) == 1 else None
+
+
+def _expected_version(request):
+    value = request.POST.get("expected_pointer_version")
+    try:
+        parsed = int(value)
+    except (TypeError, ValueError):
+        return None
+    return parsed if parsed >= 0 and str(parsed) == value else None
+
+
+_MESSAGES = {
+    FlightPublicationCode.PUBLISHED: (messages.SUCCESS, "항공 게시본을 게시했습니다."),
+    FlightPublicationCode.REJECTED: (messages.SUCCESS, "검수 거절을 기록했습니다."),
+    FlightPublicationCode.ROLLED_BACK: (messages.SUCCESS, "과거 항공 게시본을 복원했습니다."),
+    FlightPublicationCode.NOT_AUTHORIZED: (messages.ERROR, "이 작업을 수행할 권한이 없습니다."),
+    FlightPublicationCode.STALE_POINTER: (messages.WARNING, "게시 상태가 바뀌었습니다. 새 포인터 버전을 확인하십시오."),
+    FlightPublicationCode.INVALID_TARGET: (messages.WARNING, "선택 항목 또는 포인터 버전을 확인하십시오."),
+    FlightPublicationCode.INVALID_EVIDENCE: (messages.WARNING, "후보 데이터가 게시 조건을 충족하지 않습니다."),
+    FlightPublicationCode.SOURCE_GATE_FAILED: (messages.WARNING, "출처 검증 조건을 충족하지 않습니다."),
+    FlightPublicationCode.TRANSACTION_ABORTED: (messages.ERROR, "작업을 완료하지 못했습니다."),
+}
+
+
+def _message(model_admin, request, code):
+    level, text = _MESSAGES.get(
+        code,
+        (messages.ERROR, "작업을 완료하지 못했습니다."),
+    )
+    model_admin.message_user(request, text, level=level)
+
+
+@admin.register(FlightScheduleRevision, site=operator_admin_site)
+class FlightScheduleRevisionAdmin(_ReadOnlyAdmin):
+    action_form = _FlightActionForm
+    fields = (
+        "id",
+        "source_date",
+        "season",
+        "coverage_code",
+        "state",
+        "source_checked_at",
+        "created_at",
+    )
+    readonly_fields = fields
+    list_display = fields
+    ordering = ("-created_at", "-id")
+    actions = ("approve_candidate", "reject_candidate")
+
+    def has_publish_flight_schedule_permission(self, request):
+        return _authorized(
+            request, "travel_windows.publish_flight_schedule"
+        )
+
+    def has_reject_flight_schedule_permission(self, request):
+        return _authorized(
+            request, "travel_windows.reject_flight_schedule"
+        )
+
+    @admin.action(
+        description="선택한 항공 후보 검수 승인 및 게시",
+        permissions=("publish_flight_schedule",),
+    )
+    def approve_candidate(self, request, queryset):
+        revision_id = _single_id(queryset)
+        expected = _expected_version(request)
+        if revision_id is None or expected is None:
+            _message(self, request, FlightPublicationCode.INVALID_TARGET)
+            return
+        outcome = approve_flight_candidate(
+            schedule_revision_id=revision_id,
+            actor=request.user,
+            expected_pointer_version=expected,
+        )
+        _message(self, request, outcome.code)
+
+    @admin.action(
+        description="선택한 항공 후보 검수 거절",
+        permissions=("reject_flight_schedule",),
+    )
+    def reject_candidate(self, request, queryset):
+        revision_id = _single_id(queryset)
+        if revision_id is None:
+            _message(self, request, FlightPublicationCode.INVALID_TARGET)
+            return
+        outcome = reject_flight_candidate(
+            schedule_revision_id=revision_id,
+            actor=request.user,
+        )
+        _message(self, request, outcome.code)
+
+
+@admin.register(FlightPublication, site=operator_admin_site)
+class FlightPublicationAdmin(_ReadOnlyAdmin):
+    action_form = _FlightActionForm
+    fields = (
+        "id",
+        "generation",
+        "schedule_revision",
+        "review_decision",
+        "supersedes",
+        "rollback_target",
+        "published_by",
+        "source_checked_at",
+        "published_at",
+    )
+    readonly_fields = fields
+    list_display = fields
+    ordering = ("-generation", "-published_at")
+    actions = ("rollback_publication",)
+
+    def has_rollback_flight_schedule_permission(self, request):
+        return _authorized(
+            request, "travel_windows.rollback_flight_schedule"
+        )
+
+    @admin.action(
+        description="선택한 과거 항공 게시본으로 복원",
+        permissions=("rollback_flight_schedule",),
+    )
+    def rollback_publication(self, request, queryset):
+        publication_id = _single_id(queryset)
+        expected = _expected_version(request)
+        if publication_id is None or expected is None or expected < 1:
+            _message(self, request, FlightPublicationCode.INVALID_TARGET)
+            return
+        outcome = rollback_flight_publication(
+            target_publication_id=publication_id,
+            actor=request.user,
+            expected_pointer_version=expected,
+        )
+        _message(self, request, outcome.code)
+
+
+@admin.register(FlightReviewDecision, site=operator_admin_site)
+class FlightReviewDecisionAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "schedule_revision",
+        "decision",
+        "decision_sequence",
+        "reason_code",
+        "rollback_target_publication",
+        "actor_principal",
+        "decided_at",
+    )
+    readonly_fields = fields
+    list_display = fields
+    ordering = ("-decided_at", "-id")
+
+
+@admin.register(FlightAuditEvent, site=operator_admin_site)
+class FlightAuditEventAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "action",
+        "schedule_revision",
+        "review_decision",
+        "publication",
+        "prior_publication",
+        "rollback_target_publication",
+        "actor_principal",
+        "occurred_at",
+        "outcome",
+        "redaction_state",
+    )
+    readonly_fields = fields
+    list_display = fields
+    ordering = ("-occurred_at", "-id")
+
diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index d6cb5f8..9ca77aa 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -52,11 +52,11 @@ from .parser import (
     parse_legacy_arrival_pages,
     parse_route_durations,
 )
-from .publication import FlightPublicationCode, publish_flight_evidence
+from .publication import FlightPublicationCode, stage_flight_evidence
 
 
 class FlightIngestionCode:
-    PUBLISHED = "PUBLISHED"
+    STAGED = "STAGED"
     PARSE_QUARANTINED = "PARSE_QUARANTINED"
     SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
     PERSISTENCE_FAILED = "PERSISTENCE_FAILED"
@@ -65,12 +65,11 @@ class FlightIngestionCode:
 @dataclass(frozen=True, slots=True)
 class FlightIngestionOutcome:
     code: str
-    publication_id: str | None = None
-    generation: int | None = None
+    schedule_revision_id: str | None = None
 
     @property
     def succeeded(self) -> bool:
-        return self.code == FlightIngestionCode.PUBLISHED
+        return self.code == FlightIngestionCode.STAGED
 
 
 class _IngestionRejected(Exception):
@@ -208,7 +207,7 @@ def _record_validated_parse(
         return parse_run
 
 
-def ingest_and_publish_flight_evidence(
+def ingest_flight_evidence_candidate(
     *,
     departure_pages: tuple[bytes, ...],
     arrival_pages: tuple[bytes, ...],
@@ -216,10 +215,9 @@ def ingest_and_publish_flight_evidence(
     legacy_arrival_pages: tuple[bytes, ...],
     duration_csv: bytes,
     source_date: date,
-    published_by: str,
     source_checked_at: datetime | None = None,
 ) -> FlightIngestionOutcome:
-    """Validate complete pages, persist redacted evidence, and publish atomically."""
+    """Validate source pages and persist a review-required typed candidate."""
 
     checked_at = source_checked_at or timezone.now()
     schedule = adapt_data_go_schedule_pages(
@@ -308,7 +306,7 @@ def ingest_and_publish_flight_evidence(
             schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
             completed_at=checked_at,
         )
-        outcome = publish_flight_evidence(
+        outcome = stage_flight_evidence(
             schedule_run=schedule_run,
             schedule=schedule,
             city_reference_run=city_reference_run,
@@ -317,7 +315,6 @@ def ingest_and_publish_flight_evidence(
             legacy_arrivals=legacy_arrivals,
             duration_run=duration_run,
             durations=durations,
-            published_by=published_by,
             source_checked_at=checked_at,
         )
     except _IngestionRejected as exc:
@@ -328,10 +325,9 @@ def ingest_and_publish_flight_evidence(
         return FlightIngestionOutcome(FlightIngestionCode.SOURCE_GATE_FAILED)
     if outcome.code == FlightPublicationCode.INVALID_EVIDENCE:
         return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
-    if not outcome.succeeded:
+    if outcome.code != FlightPublicationCode.STAGED:
         return FlightIngestionOutcome(FlightIngestionCode.PERSISTENCE_FAILED)
     return FlightIngestionOutcome(
-        FlightIngestionCode.PUBLISHED,
-        outcome.publication_id,
-        outcome.generation,
+        FlightIngestionCode.STAGED,
+        outcome.schedule_revision_id,
     )
diff --git a/travel_windows/management/commands/publish_scheduled_flights.py b/travel_windows/management/commands/publish_scheduled_flights.py
index 899c8dd..df8cecb 100644
--- a/travel_windows/management/commands/publish_scheduled_flights.py
+++ b/travel_windows/management/commands/publish_scheduled_flights.py
@@ -15,13 +15,13 @@ from travel_windows.contracts import (
     LEGACY_SCHEDULE_SOURCE_CODE,
     SCHEDULE_SOURCE_CODE,
 )
-from travel_windows.ingestion import ingest_and_publish_flight_evidence
+from travel_windows.ingestion import ingest_flight_evidence_candidate
 
 
 class Command(BaseCommand):
     help = (
         "Validate complete documented schedule pages and a reviewed duration "
-        "CSV, then advance the flight publication pointer."
+        "CSV, then stage an immutable candidate for operator review."
     )
     requires_migrations_checks = True
 
@@ -29,7 +29,6 @@ class Command(BaseCommand):
         parser.add_argument("--duration-csv", required=True)
         parser.add_argument("--source-date", required=True)
         parser.add_argument("--season", required=True)
-        parser.add_argument("--published-by", required=True)
 
     def handle(self, *args, **options):
         try:
@@ -87,7 +86,7 @@ class Command(BaseCommand):
                 f"code={reference_evidence.failure_code}"
             )
 
-        outcome = ingest_and_publish_flight_evidence(
+        outcome = ingest_flight_evidence_candidate(
             departure_pages=fetched.departure_pages,
             arrival_pages=fetched.arrival_pages,
             city_payload=reference_evidence.city_payload,
@@ -96,8 +95,10 @@ class Command(BaseCommand):
             ),
             duration_csv=duration_csv,
             source_date=source_date,
-            published_by=options["published_by"],
         )
         if not outcome.succeeded:
             raise CommandError(f"result=blocked code={outcome.code}")
-        self.stdout.write(f"result=published generation={outcome.generation}")
+        self.stdout.write(
+            "result=staged "
+            f"schedule_revision={outcome.schedule_revision_id}"
+        )
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index d86b728..d4bb9ab 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -9,10 +9,14 @@ from __future__ import annotations
 
 import hashlib
 import json
+import uuid
 from dataclasses import dataclass
 from datetime import datetime
+from typing import Any
 
+from django.contrib.auth import get_user_model
 from django.db import connection, transaction
+from django.db.models import F, Max
 from django.utils import timezone
 
 from sources.models import ParseRun, SourceConfiguration, SourceRightsDecision
@@ -41,8 +45,11 @@ from .models import (
     FLIGHT_POINTER_ID,
     Airport,
     CURATED_AIRPORT_ROWS,
+    FlightAuditEvent,
+    FlightCandidateDuration,
     FlightPublication,
     FlightPublicationDuration,
+    FlightReviewDecision,
     FlightSchedule,
     FlightScheduleRevision,
     PublishedFlightSchedule,
@@ -62,20 +69,34 @@ PUBLICATION_LOCK_KEY = 20_260_831
 
 
 class FlightPublicationCode:
+    STAGED = "STAGED"
     PUBLISHED = "PUBLISHED"
+    REJECTED = "REJECTED"
+    ROLLED_BACK = "ROLLED_BACK"
     INVALID_EVIDENCE = "INVALID_EVIDENCE"
+    INVALID_TARGET = "INVALID_TARGET"
     SOURCE_GATE_FAILED = "SOURCE_GATE_FAILED"
+    NOT_AUTHORIZED = "NOT_AUTHORIZED"
+    STALE_POINTER = "STALE_POINTER"
+    TRANSACTION_ABORTED = "TRANSACTION_ABORTED"
 
 
 @dataclass(frozen=True, slots=True)
 class FlightPublicationOutcome:
     code: str
+    schedule_revision_id: str | None = None
     publication_id: str | None = None
     generation: int | None = None
+    pointer_version: int | None = None
 
     @property
     def succeeded(self) -> bool:
-        return self.code == FlightPublicationCode.PUBLISHED
+        return self.code in {
+            FlightPublicationCode.STAGED,
+            FlightPublicationCode.PUBLISHED,
+            FlightPublicationCode.REJECTED,
+            FlightPublicationCode.ROLLED_BACK,
+        }
 
 
 class _PublicationRejected(Exception):
@@ -238,7 +259,7 @@ def _curated_schedule_projection(
     )
 
 
-def publish_flight_evidence(
+def stage_flight_evidence(
     *,
     schedule_run: ParseRun,
     schedule: ScheduleParseSuccess,
@@ -248,13 +269,12 @@ def publish_flight_evidence(
     legacy_arrivals: LegacyArrivalParseSuccess,
     duration_run: ParseRun,
     durations: DurationParseSuccess,
-    published_by: str,
     source_checked_at: datetime,
 ) -> FlightPublicationOutcome:
-    """Publish one complete schedule and its reviewed route durations.
+    """Persist one immutable, typed candidate without moving the pointer.
 
-    The function is fail-closed and never retains raw response bytes.  Invalid
-    input rolls the transaction back and returns only a closed outcome code.
+    A worker may call this service after parsing source responses. It cannot
+    publish because no actor, review, audit, or pointer mutation exists here.
     """
 
     if (
@@ -262,15 +282,11 @@ def publish_flight_evidence(
         or not isinstance(city_reference, CityReferenceParseSuccess)
         or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
         or not isinstance(durations, DurationParseSuccess)
-        or not isinstance(published_by, str)
-        or not published_by.strip()
-        or len(published_by) > 100
         or not _aware(source_checked_at)
     ):
         return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
     try:
         with transaction.atomic(durable=True):
-            _lock_registry()
             locked_schedule_run = _approved_run(
                 schedule_run,
                 source_code=SCHEDULE_SOURCE_CODE,
@@ -307,6 +323,13 @@ def publish_flight_evidence(
                 contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
             )
+            effective_checked_at = max(
+                source_checked_at,
+                locked_schedule_run.completed_at,
+                locked_city_reference_run.completed_at,
+                locked_legacy_arrivals_run.completed_at,
+                locked_duration_run.completed_at,
+            )
             if (
                 schedule.observed_schema_fingerprint_sha256
                 != SCHEDULE_SCHEMA_FINGERPRINT_SHA256
@@ -353,6 +376,7 @@ def publish_flight_evidence(
                 completeness=FlightScheduleRevision.Completeness.COMPLETE,
                 state=FlightScheduleRevision.State.VALIDATED,
                 first_observed_at=locked_schedule_run.completed_at,
+                source_checked_at=effective_checked_at,
                 typed_fingerprint_sha256=_fingerprint(projected_schedule),
             )
             FlightSchedule.objects.bulk_create(
@@ -386,51 +410,509 @@ def publish_flight_evidence(
                         reference_date=row.reference_date,
                         reference_locator=row.reference_locator,
                         state=RouteDurationRevision.State.VALIDATED,
-                        reviewed_by=published_by.strip(),
-                        reviewed_at=source_checked_at,
+                        reviewed_by="AVIATION_TYPED_VALIDATOR_V1",
+                        reviewed_at=effective_checked_at,
                         typed_fingerprint_sha256=_fingerprint(row),
                     )
                 )
+            FlightCandidateDuration.objects.bulk_create(
+                [
+                    FlightCandidateDuration(
+                        schedule_revision=revision,
+                        duration_revision=duration,
+                        destination_airport=duration.destination_airport,
+                    )
+                    for duration in duration_revisions
+                ]
+            )
+        return FlightPublicationOutcome(
+            FlightPublicationCode.STAGED,
+            schedule_revision_id=str(revision.pk),
+        )
+    except _PublicationRejected as exc:
+        return FlightPublicationOutcome(exc.code)
+    except Exception:
+        return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
 
-            pointer, _ = (
-                PublishedFlightSchedule.objects.select_for_update().get_or_create(
-                    pk=FLIGHT_POINTER_ID,
-                    defaults={"version": 0},
-                )
+
+def _principal(actor: Any) -> str:
+    try:
+        value = str(actor.get_username())
+    except Exception:
+        return "AUTHENTICATED_OPERATOR"
+    if not value or any(ord(character) < 32 for character in value):
+        return "AUTHENTICATED_OPERATOR"
+    return value[:100]
+
+
+def _lock_authorized_actor(actor: Any, permission: str):
+    try:
+        if not getattr(actor, "is_authenticated", False) or actor.pk is None:
+            return None
+        user_model = get_user_model()
+        locked = user_model.objects.select_for_update().get(pk=actor.pk)
+        if not locked.is_active or not locked.is_staff:
+            return None
+        if locked.is_superuser or locked.has_perm(permission):
+            return locked
+    except get_user_model().DoesNotExist:
+        return None
+    except Exception:
+        raise _PublicationRejected(
+            FlightPublicationCode.TRANSACTION_ABORTED
+        ) from None
+    return None
+
+
+def _next_review_sequence(revision: FlightScheduleRevision) -> int:
+    current = revision.review_decisions.aggregate(
+        current=Max("decision_sequence")
+    )["current"]
+    return (current or 0) + 1
+
+
+def _load_valid_candidate(
+    schedule_revision_id: object,
+) -> tuple[FlightScheduleRevision, list[FlightCandidateDuration]]:
+    try:
+        revision = (
+            FlightScheduleRevision.objects.select_for_update(of=("self",))
+            .select_related(
+                "parse_run",
+                "city_reference_parse_run",
+                "legacy_arrivals_parse_run",
             )
-            generation = pointer.version + 1
+            .get(pk=schedule_revision_id)
+        )
+    except (FlightScheduleRevision.DoesNotExist, TypeError, ValueError):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET) from None
+    if (
+        revision.state != FlightScheduleRevision.State.VALIDATED
+        or revision.completeness
+        != FlightScheduleRevision.Completeness.COMPLETE
+        or revision.coverage_code != "ICN_DIRECT_PUBLIC_V1"
+        or not _aware(revision.source_checked_at)
+        or revision.city_reference_parse_run_id is None
+        or revision.legacy_arrivals_parse_run_id is None
+    ):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+
+    _approved_run(
+        revision.parse_run,
+        source_code=SCHEDULE_SOURCE_CODE,
+        source_revision=SCHEDULE_SOURCE_REVISION,
+        parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
+        contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    )
+    _approved_run(
+        revision.city_reference_parse_run,
+        source_code=CITY_SOURCE_CODE,
+        source_revision=CITY_SOURCE_REVISION,
+        parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+        contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
+        schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+    )
+    _approved_run(
+        revision.legacy_arrivals_parse_run,
+        source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+        source_revision=LEGACY_SCHEDULE_SOURCE_REVISION,
+        parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+        contract_fingerprint=LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+        schema_fingerprint=LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    )
+
+    schedules = list(
+        FlightSchedule.objects.select_for_update().filter(revision=revision)
+    )
+    directions: dict[object, set[str]] = {}
+    for row in schedules:
+        directions.setdefault(row.destination_airport_id, set()).add(
+            row.direction
+        )
+    if not directions or any(
+        values != {FlightSchedule.Direction.OUTBOUND, FlightSchedule.Direction.INBOUND}
+        for values in directions.values()
+    ):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+
+    links = list(
+        FlightCandidateDuration.objects.select_for_update()
+        .select_related("duration_revision__parse_run")
+        .filter(schedule_revision=revision)
+    )
+    if {link.destination_airport_id for link in links} != set(directions):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+    validated_runs: set[object] = set()
+    for link in links:
+        duration = link.duration_revision
+        if (
+            duration.destination_airport_id != link.destination_airport_id
+            or duration.state != RouteDurationRevision.State.VALIDATED
+            or not 1 <= duration.outbound_minutes <= 1440
+            or not 1 <= duration.inbound_minutes <= 1440
+        ):
+            raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+        if duration.parse_run_id not in validated_runs:
+            _approved_run(
+                duration.parse_run,
+                source_code=DURATION_SOURCE_CODE,
+                source_revision=DURATION_SOURCE_REVISION,
+                parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
+                contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
+                schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
+            )
+            validated_runs.add(duration.parse_run_id)
+    return revision, links
+
+
+def _audit_hash(
+    *,
+    action: str,
+    schedule_revision_id: object,
+    review_id: object,
+    publication_id: object | None,
+    prior_id: object | None,
+    rollback_target_id: object | None,
+) -> str:
+    value = "\n".join(
+        str(part) if part is not None else ""
+        for part in (
+            action,
+            schedule_revision_id,
+            review_id,
+            publication_id,
+            prior_id,
+            rollback_target_id,
+            "SUCCEEDED",
+        )
+    )
+    return hashlib.sha256(value.encode("utf-8")).hexdigest()
+
+
+def _create_audit(
+    *,
+    action: str,
+    actor,
+    principal: str,
+    revision: FlightScheduleRevision,
+    review: FlightReviewDecision,
+    publication: FlightPublication | None = None,
+    prior: FlightPublication | None = None,
+    rollback_target: FlightPublication | None = None,
+) -> FlightAuditEvent:
+    return FlightAuditEvent.objects.create(
+        actor=actor,
+        actor_principal=principal,
+        action=action,
+        schedule_revision=revision,
+        review_decision=review,
+        publication=publication,
+        prior_publication=prior,
+        rollback_target_publication=rollback_target,
+        logical_transaction_id=uuid.uuid4(),
+        input_identity_sha256=_audit_hash(
+            action=action,
+            schedule_revision_id=revision.pk,
+            review_id=review.pk,
+            publication_id=publication.pk if publication else None,
+            prior_id=prior.pk if prior else None,
+            rollback_target_id=(
+                rollback_target.pk if rollback_target else None
+            ),
+        ),
+    )
+
+
+def _locked_pointer() -> PublishedFlightSchedule:
+    try:
+        pointer, _created = (
+            PublishedFlightSchedule.objects.select_for_update().get_or_create(
+                pk=FLIGHT_POINTER_ID,
+                defaults={"version": 0},
+            )
+        )
+        return pointer
+    except Exception:
+        raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET) from None
+
+
+def _advance_pointer(
+    *,
+    pointer: PublishedFlightSchedule,
+    publication: FlightPublication,
+    expected_pointer_version: int,
+) -> None:
+    updated = PublishedFlightSchedule.objects.filter(
+        pk=pointer.pk,
+        version=expected_pointer_version,
+    ).update(
+        current_publication=publication,
+        version=F("version") + 1,
+        updated_at=timezone.now(),
+    )
+    if updated != 1:
+        raise _PublicationRejected(FlightPublicationCode.STALE_POINTER)
+
+
+def approve_flight_candidate(
+    *,
+    schedule_revision_id: object,
+    actor: Any,
+    expected_pointer_version: int,
+) -> FlightPublicationOutcome:
+    """Approve and publish a candidate as one atomic operator action."""
+
+    if type(expected_pointer_version) is not int or expected_pointer_version < 0:
+        return FlightPublicationOutcome(FlightPublicationCode.INVALID_TARGET)
+    try:
+        with transaction.atomic(durable=True):
+            _lock_registry()
+            pointer = _locked_pointer()
+            if pointer.version != expected_pointer_version:
+                raise _PublicationRejected(FlightPublicationCode.STALE_POINTER)
+            revision, links = _load_valid_candidate(schedule_revision_id)
+            if revision.publications.exists() or revision.review_decisions.exists():
+                raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET)
+            locked_actor = _lock_authorized_actor(
+                actor, "travel_windows.publish_flight_schedule"
+            )
+            if locked_actor is None:
+                raise _PublicationRejected(FlightPublicationCode.NOT_AUTHORIZED)
+            principal = _principal(locked_actor)
+            review = FlightReviewDecision.objects.create(
+                schedule_revision=revision,
+                decision=FlightReviewDecision.Decision.APPROVED,
+                decision_sequence=1,
+                reason_code=FlightReviewDecision.ReasonCode.APPROVE,
+                actor=locked_actor,
+                actor_principal=principal,
+                correlation_id=uuid.uuid4(),
+            )
+            published_at = max(timezone.now(), revision.source_checked_at)
             publication = FlightPublication.objects.create(
                 schedule_revision=revision,
-                generation=generation,
+                generation=pointer.version + 1,
                 source_revision=SCHEDULE_SOURCE_REVISION,
                 source_locator=SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
                 source_attribution=SCHEDULE_ATTRIBUTION,
-                source_checked_at=source_checked_at,
-                published_by=published_by.strip(),
+                source_checked_at=revision.source_checked_at,
+                published_by=principal,
+                published_at=published_at,
+                review_decision=review,
                 supersedes=pointer.current_publication,
             )
             FlightPublicationDuration.objects.bulk_create(
                 [
                     FlightPublicationDuration(
                         publication=publication,
-                        duration_revision=duration,
-                        destination_airport=duration.destination_airport,
+                        duration_revision=link.duration_revision,
+                        destination_airport=link.destination_airport,
                     )
-                    for duration in duration_revisions
+                    for link in links
                 ]
             )
-            pointer.current_publication = publication
-            pointer.version = generation
-            pointer.updated_at = timezone.now()
-            pointer.save(
-                update_fields=("current_publication", "version", "updated_at")
+            _create_audit(
+                action=FlightAuditEvent.Action.PUBLISH,
+                actor=locked_actor,
+                principal=principal,
+                revision=revision,
+                review=review,
+                publication=publication,
+                prior=pointer.current_publication,
+            )
+            _advance_pointer(
+                pointer=pointer,
+                publication=publication,
+                expected_pointer_version=expected_pointer_version,
             )
         return FlightPublicationOutcome(
             FlightPublicationCode.PUBLISHED,
-            str(publication.pk),
-            generation,
+            schedule_revision_id=str(revision.pk),
+            publication_id=str(publication.pk),
+            generation=publication.generation,
+            pointer_version=expected_pointer_version + 1,
         )
     except _PublicationRejected as exc:
         return FlightPublicationOutcome(exc.code)
     except Exception:
-        return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
+        return FlightPublicationOutcome(FlightPublicationCode.TRANSACTION_ABORTED)
+
+
+def reject_flight_candidate(
+    *,
+    schedule_revision_id: object,
+    actor: Any,
+) -> FlightPublicationOutcome:
+    """Append a rejection review and audit without publishing anything."""
+
+    try:
+        with transaction.atomic(durable=True):
+            _lock_registry()
+            revision, _links = _load_valid_candidate(schedule_revision_id)
+            if revision.publications.exists() or revision.review_decisions.exists():
+                raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET)
+            locked_actor = _lock_authorized_actor(
+                actor, "travel_windows.reject_flight_schedule"
+            )
+            if locked_actor is None:
+                raise _PublicationRejected(FlightPublicationCode.NOT_AUTHORIZED)
+            principal = _principal(locked_actor)
+            review = FlightReviewDecision.objects.create(
+                schedule_revision=revision,
+                decision=FlightReviewDecision.Decision.REJECTED,
+                decision_sequence=1,
+                reason_code=FlightReviewDecision.ReasonCode.REJECT,
+                actor=locked_actor,
+                actor_principal=principal,
+                correlation_id=uuid.uuid4(),
+            )
+            _create_audit(
+                action=FlightAuditEvent.Action.REVIEW_REJECT,
+                actor=locked_actor,
+                principal=principal,
+                revision=revision,
+                review=review,
+            )
+        return FlightPublicationOutcome(
+            FlightPublicationCode.REJECTED,
+            schedule_revision_id=str(revision.pk),
+        )
+    except _PublicationRejected as exc:
+        return FlightPublicationOutcome(exc.code)
+    except Exception:
+        return FlightPublicationOutcome(FlightPublicationCode.TRANSACTION_ABORTED)
+
+
+def _target_is_ancestor(
+    *, current: FlightPublication, target: FlightPublication
+) -> bool:
+    predecessor_id = current.supersedes_id
+    visited: set[object] = set()
+    while predecessor_id is not None and predecessor_id not in visited:
+        if predecessor_id == target.pk:
+            return True
+        visited.add(predecessor_id)
+        predecessor = FlightPublication.objects.select_for_update().filter(
+            pk=predecessor_id
+        ).first()
+        if predecessor is None:
+            return False
+        predecessor_id = predecessor.supersedes_id
+    return False
+
+
+def rollback_flight_publication(
+    *,
+    target_publication_id: object,
+    actor: Any,
+    expected_pointer_version: int,
+) -> FlightPublicationOutcome:
+    """Append a generation that restores one historical evidence set."""
+
+    if type(expected_pointer_version) is not int or expected_pointer_version < 1:
+        return FlightPublicationOutcome(FlightPublicationCode.INVALID_TARGET)
+    try:
+        with transaction.atomic(durable=True):
+            _lock_registry()
+            pointer = _locked_pointer()
+            if pointer.version != expected_pointer_version:
+                raise _PublicationRejected(FlightPublicationCode.STALE_POINTER)
+            current = pointer.current_publication
+            if current is None:
+                raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET)
+            try:
+                target = FlightPublication.objects.select_for_update().get(
+                    pk=target_publication_id
+                )
+            except (FlightPublication.DoesNotExist, TypeError, ValueError):
+                raise _PublicationRejected(
+                    FlightPublicationCode.INVALID_TARGET
+                ) from None
+            if target.pk == current.pk or not _target_is_ancestor(
+                current=current, target=target
+            ):
+                raise _PublicationRejected(FlightPublicationCode.INVALID_TARGET)
+            revision, candidate_links = _load_valid_candidate(
+                target.schedule_revision_id
+            )
+            target_links = list(
+                FlightPublicationDuration.objects.select_for_update()
+                .select_related("duration_revision", "destination_airport")
+                .filter(publication=target)
+            )
+            candidate_identity = {
+                (link.duration_revision_id, link.destination_airport_id)
+                for link in candidate_links
+            }
+            target_identity = {
+                (link.duration_revision_id, link.destination_airport_id)
+                for link in target_links
+            }
+            if not target_identity or target_identity != candidate_identity:
+                raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+            locked_actor = _lock_authorized_actor(
+                actor, "travel_windows.rollback_flight_schedule"
+            )
+            if locked_actor is None:
+                raise _PublicationRejected(FlightPublicationCode.NOT_AUTHORIZED)
+            principal = _principal(locked_actor)
+            review = FlightReviewDecision.objects.create(
+                schedule_revision=revision,
+                decision=FlightReviewDecision.Decision.APPROVED,
+                decision_sequence=_next_review_sequence(revision),
+                reason_code=FlightReviewDecision.ReasonCode.ROLLBACK,
+                rollback_target_publication=target,
+                actor=locked_actor,
+                actor_principal=principal,
+                correlation_id=uuid.uuid4(),
+            )
+            publication = FlightPublication.objects.create(
+                schedule_revision=revision,
+                generation=pointer.version + 1,
+                source_revision=target.source_revision,
+                source_locator=target.source_locator,
+                source_attribution=target.source_attribution,
+                source_checked_at=target.source_checked_at,
+                published_by=principal,
+                published_at=max(timezone.now(), target.source_checked_at),
+                review_decision=review,
+                supersedes=current,
+                rollback_target=target,
+            )
+            FlightPublicationDuration.objects.bulk_create(
+                [
+                    FlightPublicationDuration(
+                        publication=publication,
+                        duration_revision=link.duration_revision,
+                        destination_airport=link.destination_airport,
+                    )
+                    for link in target_links
+                ]
+            )
+            _create_audit(
+                action=FlightAuditEvent.Action.ROLLBACK,
+                actor=locked_actor,
+                principal=principal,
+                revision=revision,
+                review=review,
+                publication=publication,
+                prior=current,
+                rollback_target=target,
+            )
+            _advance_pointer(
+                pointer=pointer,
+                publication=publication,
+                expected_pointer_version=expected_pointer_version,
+            )
+        return FlightPublicationOutcome(
+            FlightPublicationCode.ROLLED_BACK,
+            schedule_revision_id=str(revision.pk),
+            publication_id=str(publication.pk),
+            generation=publication.generation,
+            pointer_version=expected_pointer_version + 1,
+        )
+    except _PublicationRejected as exc:
+        return FlightPublicationOutcome(exc.code)
+    except Exception:
+        return FlightPublicationOutcome(FlightPublicationCode.TRANSACTION_ABORTED)


