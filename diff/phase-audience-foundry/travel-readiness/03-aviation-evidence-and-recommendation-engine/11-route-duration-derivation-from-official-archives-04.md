## `feat(flights): bind official duration evidence`

diff --git a/travel_windows/admin.py b/travel_windows/admin.py
index 9c545b6..ea09088 100644
--- a/travel_windows/admin.py
+++ b/travel_windows/admin.py
@@ -1,6 +1,7 @@
 from django import forms
 from django.contrib import admin, messages
 from django.contrib.admin.helpers import ActionForm
+from django.utils.html import format_html
 
 from reviews.admin import operator_admin_site
 
@@ -11,6 +12,7 @@ from .models import (
     FlightReviewDecision,
     FlightSchedule,
     FlightScheduleRevision,
+    RouteDurationDerivedRoute,
 )
 from .publication import (
     FlightPublicationCode,
@@ -74,9 +76,64 @@ class FlightScheduleInline(_ReadOnlyInline):
 class FlightCandidateDurationInline(_ReadOnlyInline):
     model = FlightCandidateDuration
     fk_name = "schedule_revision"
-    fields = ("destination_airport", "duration_revision")
+    fields = (
+        "destination_airport",
+        "outbound_minutes",
+        "inbound_minutes",
+        "reference_date",
+        "reference_locator",
+        "duration_evidence",
+        "derived_sample_summary",
+    )
     readonly_fields = fields
 
+    @admin.display(description="출국 예상 비행시간(분)")
+    def outbound_minutes(self, obj):
+        return obj.duration_revision.outbound_minutes
+
+    @admin.display(description="귀국 예상 비행시간(분)")
+    def inbound_minutes(self, obj):
+        return obj.duration_revision.inbound_minutes
+
+    @admin.display(description="공식 자료 기준일")
+    def reference_date(self, obj):
+        return obj.duration_revision.reference_date
+
+    @admin.display(description="공식 자료")
+    def reference_locator(self, obj):
+        return format_html(
+            '<a href="{}" target="_blank" rel="noopener noreferrer">'
+            "공식 자료 열기</a>",
+            obj.duration_revision.reference_locator,
+        )
+
+    @admin.display(description="비행시간 근거")
+    def duration_evidence(self, obj):
+        duration = obj.duration_revision
+        if duration.parse_run_id is None:
+            return f"기존 사람 검수 파일 · {duration.reviewed_by}"
+        return f"공식 파일 자동 산출 · {duration.parse_run_id}"
+
+    @admin.display(description="산출 표본")
+    def derived_sample_summary(self, obj):
+        duration = obj.duration_revision
+        if duration.parse_run_id is None:
+            return "—"
+        try:
+            row = duration.parse_run.derived_duration_routes.get(
+                destination_airport=duration.destination_airport
+            )
+        except (
+            RouteDurationDerivedRoute.DoesNotExist,
+            RouteDurationDerivedRoute.MultipleObjectsReturned,
+            AttributeError,
+        ):
+            return "근거 행 확인 필요"
+        return (
+            f"출국 {row.outbound_retained}/{row.outbound_occurrences} · "
+            f"귀국 {row.inbound_retained}/{row.inbound_occurrences}"
+        )
+
 
 def _authorized(request, permission: str) -> bool:
     try:
diff --git a/travel_windows/management/commands/publish_scheduled_flights.py b/travel_windows/management/commands/publish_scheduled_flights.py
index bc97efe..875a64c 100644
--- a/travel_windows/management/commands/publish_scheduled_flights.py
+++ b/travel_windows/management/commands/publish_scheduled_flights.py
@@ -1,4 +1,5 @@
 import re
+import uuid
 from datetime import date
 from pathlib import Path
 
@@ -14,48 +15,67 @@ from travel_windows.contracts import (
     CITY_SOURCE_CODE,
     DATA_GO_SECRET_REFERENCE,
     DURATION_SOURCE_CODE,
+    DURATION_SOURCE_REVISION,
     LEGACY_SCHEDULE_SOURCE_CODE,
     SCHEDULE_SOURCE_CODE,
 )
 from travel_windows.ingestion import (
     DurableAviationAttemptRecorder,
     ingest_flight_evidence_candidate,
+    ingest_flight_evidence_candidate_with_derivation,
 )
 
 
 class Command(BaseCommand):
     help = (
-        "Validate complete documented schedule pages and a reviewed duration "
-        "CSV, then stage an immutable candidate for operator review."
+        "Validate complete documented schedule pages and an exact official "
+        "duration derivation, then stage a candidate for operator review."
     )
     requires_migrations_checks = True
 
     def add_arguments(self, parser):
-        parser.add_argument("--duration-csv", required=True)
+        duration = parser.add_mutually_exclusive_group(required=True)
+        duration.add_argument("--duration-derivation-id")
+        duration.add_argument("--duration-csv")
         parser.add_argument("--source-date", required=True)
         parser.add_argument("--season", required=True)
-        parser.add_argument("--duration-reviewed-by", required=True)
-        parser.add_argument("--duration-review-basis", required=True)
+        parser.add_argument("--duration-reviewed-by")
+        parser.add_argument("--duration-review-basis")
 
     def handle(self, *args, **options):
         try:
             source_date = date.fromisoformat(options["source_date"])
+            duration_derivation_id = options["duration_derivation_id"]
+            duration_csv_path = options["duration_csv"]
             duration_reviewed_by = options["duration_reviewed_by"]
             duration_review_basis = options["duration_review_basis"]
-            if (
-                re.fullmatch(
-                    r"[A-Z][A-Z0-9_.:-]{2,99}",
-                    duration_reviewed_by,
-                )
-                is None
-                or re.fullmatch(
-                    r"[A-Z][A-Z0-9_]{2,99}",
-                    duration_review_basis,
-                )
-                is None
-            ):
-                raise ValueError
-            duration_csv = Path(options["duration_csv"]).read_bytes()
+            if duration_derivation_id is not None:
+                if (
+                    duration_reviewed_by is not None
+                    or duration_review_basis is not None
+                ):
+                    raise ValueError
+                parsed_derivation_id = uuid.UUID(duration_derivation_id)
+                if str(parsed_derivation_id) != duration_derivation_id.lower():
+                    raise ValueError
+                duration_csv = None
+                expected_duration_revision = DURATION_SOURCE_REVISION
+            else:
+                if (
+                    re.fullmatch(
+                        r"[A-Z][A-Z0-9_.:-]{2,99}",
+                        duration_reviewed_by or "",
+                    )
+                    is None
+                    or re.fullmatch(
+                        r"[A-Z][A-Z0-9_]{2,99}",
+                        duration_review_basis or "",
+                    )
+                    is None
+                ):
+                    raise ValueError
+                duration_csv = Path(duration_csv_path).read_bytes()
+                expected_duration_revision = DURATION_SOURCE_REVISION
             source = SourceConfiguration.objects.get(
                 code=SCHEDULE_SOURCE_CODE,
                 state=SourceConfiguration.State.ACTIVE,
@@ -77,6 +97,7 @@ class Command(BaseCommand):
                 raise SourceConfiguration.DoesNotExist
             SourceConfiguration.objects.get(
                 code=DURATION_SOURCE_CODE,
+                revision=expected_duration_revision,
                 state=SourceConfiguration.State.ACTIVE,
                 enabled=True,
                 secret_reference_name="",
@@ -117,15 +138,24 @@ class Command(BaseCommand):
                 f"code={reference_evidence.failure_code}"
             )
 
-        outcome = ingest_flight_evidence_candidate(
-            schedule_fetch=fetched,
-            reference_fetch=reference_evidence,
-            attempt_recorder=attempt_recorder,
-            duration_csv=duration_csv,
-            source_date=source_date,
-            duration_reviewed_by=duration_reviewed_by,
-            duration_reviewer_basis_code=duration_review_basis,
-        )
+        common = {
+            "schedule_fetch": fetched,
+            "reference_fetch": reference_evidence,
+            "attempt_recorder": attempt_recorder,
+            "source_date": source_date,
+        }
+        if duration_derivation_id is not None:
+            outcome = ingest_flight_evidence_candidate_with_derivation(
+                **common,
+                duration_derivation_id=parsed_derivation_id,
+            )
+        else:
+            outcome = ingest_flight_evidence_candidate(
+                **common,
+                duration_csv=duration_csv,
+                duration_reviewed_by=duration_reviewed_by,
+                duration_reviewer_basis_code=duration_review_basis,
+            )
         if not outcome.succeeded:
             raise CommandError(f"result=blocked code={outcome.code}")
         self.stdout.write(
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index ec4c4c9..2de88d3 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -39,10 +39,15 @@ from .contracts import (
     CITY_V2_CONTRACT_FINGERPRINT_SHA256,
     CITY_V2_SCHEMA_FINGERPRINT_SHA256,
     CITY_V2_SOURCE_REVISION,
-    DURATION_CONTRACT_FINGERPRINT_SHA256,
-    DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_DERIVATION_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
+    DURATION_SOURCE_LOCATOR,
     DURATION_SOURCE_REVISION,
+    DURATION_V1_CONTRACT_FINGERPRINT_SHA256,
+    DURATION_V1_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_V1_SOURCE_REVISION,
     LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     LEGACY_SCHEDULE_SOURCE_CODE,
@@ -72,6 +77,8 @@ from .models import (
     FlightScheduleRevision,
     PublishedFlightSchedule,
     ReviewedDurationReceipt,
+    RouteDurationDerivation,
+    RouteDurationDerivedRoute,
     RouteDurationRevision,
 )
 from .parser import (
@@ -194,16 +201,6 @@ def _schedule_payload_from_models(
     }
 
 
-def _duration_payload_from_parsed(row: object) -> dict[str, object]:
-    return {
-        "destination_iata": row.destination_iata,
-        "outbound_minutes": row.outbound_minutes,
-        "inbound_minutes": row.inbound_minutes,
-        "reference_date": row.reference_date.isoformat(),
-        "reference_locator": row.reference_locator,
-    }
-
-
 def _duration_payload_from_model(
     duration: RouteDurationRevision,
 ) -> dict[str, object]:
@@ -216,6 +213,54 @@ def _duration_payload_from_model(
     }
 
 
+def _derived_route_payload_from_model(
+    row: RouteDurationDerivedRoute,
+) -> dict[str, object]:
+    return {
+        "destination_iata": row.destination_airport.iata_code,
+        "outbound_minutes": row.outbound_minutes,
+        "inbound_minutes": row.inbound_minutes,
+        "outbound": {
+            "source_rows": row.outbound_source_rows,
+            "invalid_rows": row.outbound_invalid_rows,
+            "cancelled_rows": row.outbound_cancelled_rows,
+            "occurrences": row.outbound_occurrences,
+            "outliers": row.outbound_outliers,
+            "retained": row.outbound_retained,
+        },
+        "inbound": {
+            "source_rows": row.inbound_source_rows,
+            "invalid_rows": row.inbound_invalid_rows,
+            "cancelled_rows": row.inbound_cancelled_rows,
+            "occurrences": row.inbound_occurrences,
+            "outliers": row.inbound_outliers,
+            "retained": row.inbound_retained,
+        },
+    }
+
+
+def _duration_derivation_payload_from_models(
+    derivation: RouteDurationDerivation,
+    rows: list[RouteDurationDerivedRoute],
+) -> dict[str, object]:
+    return {
+        "reference_date": derivation.reference_date.isoformat(),
+        "reference_locator": derivation.reference_locator,
+        "reference_version_sha256": derivation.reference_version_sha256,
+        "source_row_count": derivation.source_row_count,
+        "derivation_contract": (
+            derivation.derivation_contract_fingerprint_sha256
+        ),
+        "routes": [
+            _derived_route_payload_from_model(row)
+            for row in sorted(
+                rows,
+                key=lambda value: value.destination_airport.iata_code,
+            )
+        ],
+    }
+
+
 def _closure_fingerprint(
     *, schedule_fingerprint: str, duration_identities: list[tuple[str, str]]
 ) -> str:
@@ -301,6 +346,7 @@ def _approved_run(
         ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
         ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
         ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+        ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE,
     }
     if ordered_parser:
         inputs = list(
@@ -335,6 +381,8 @@ def _approved_run(
             expected_roles = ("SCHEDULE_DEPARTURE", "SCHEDULE_ARRIVAL")
         elif parser_name == ParseRun.ParserName.ICN_DESTINATION_CITY_JSON:
             expected_roles = ("DESTINATION_CITY",)
+        elif parser_name == ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE:
+            expected_roles = ("DURATION_ARCHIVE",)
         else:
             expected_roles = ("LEGACY_ARRIVAL",)
         observed_roles = tuple(dict.fromkeys(row.role for row in inputs))
@@ -375,43 +423,113 @@ def _approved_duration_receipt(
     rights = locked.rights_decision
     latest_rights = (
         SourceRightsDecision.objects.select_for_update()
-        .filter(source=source, source_revision=source.revision)
+        .filter(
+            source=source,
+            source_revision=DURATION_V1_SOURCE_REVISION,
+        )
         .order_by("-decision_sequence", "-id")
         .first()
     )
     if (
         locked.outcome != ReviewedDurationReceipt.Outcome.VALIDATED
         or locked.failure_code
-        or locked.source_revision != DURATION_SOURCE_REVISION
+        or locked.source_revision != DURATION_V1_SOURCE_REVISION
         or locked.parser_contract_fingerprint_sha256
-        != DURATION_CONTRACT_FINGERPRINT_SHA256
+        != DURATION_V1_CONTRACT_FINGERPRINT_SHA256
         or locked.expected_schema_fingerprint_sha256
-        != DURATION_SCHEMA_FINGERPRINT_SHA256
+        != DURATION_V1_SCHEMA_FINGERPRINT_SHA256
         or locked.observed_schema_fingerprint_sha256
-        != DURATION_SCHEMA_FINGERPRINT_SHA256
+        != DURATION_V1_SCHEMA_FINGERPRINT_SHA256
         or locked.byte_count < 1
         or not _aware(locked.reviewed_at)
         or source.code != DURATION_SOURCE_CODE
-        or source.revision != DURATION_SOURCE_REVISION
+        or source.official_locator != DURATION_SOURCE_LOCATOR
+        or source.secret_reference_name != ""
         or source.module != SourceConfiguration.Module.AVIATION
         or source.state != SourceConfiguration.State.ACTIVE
         or not source.enabled
         or latest_rights is None
         or latest_rights.pk != rights.pk
         or rights.source_id != source.pk
-        or rights.source_revision != source.revision
+        or rights.source_revision != DURATION_V1_SOURCE_REVISION
         or rights.decision != SourceRightsDecision.Decision.APPROVED
         or not rights.access_allowed
         or not rights.typed_field_storage_allowed
         or not rights.typed_republication_allowed
         or rights.raw_body_storage_allowed
         or rights.contract_fingerprint_sha256
-        != DURATION_CONTRACT_FINGERPRINT_SHA256
+        != DURATION_V1_CONTRACT_FINGERPRINT_SHA256
     ):
         raise _PublicationRejected(FlightPublicationCode.SOURCE_GATE_FAILED)
     return locked
 
 
+def _approved_duration_derivation(
+    derivation: RouteDurationDerivation,
+) -> tuple[RouteDurationDerivation, list[RouteDurationDerivedRoute]]:
+    try:
+        locked = (
+            RouteDurationDerivation.objects.select_for_update()
+            .select_related("parse_run")
+            .get(pk=derivation.pk)
+        )
+    except (RouteDurationDerivation.DoesNotExist, AttributeError, TypeError):
+        raise _PublicationRejected(
+            FlightPublicationCode.INVALID_EVIDENCE
+        ) from None
+    run = _approved_run(
+        locked.parse_run,
+        source_code=DURATION_SOURCE_CODE,
+        source_revision=DURATION_SOURCE_REVISION,
+        parser_name=ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE,
+        parser_version=ParseRun.ParserVersion.V2,
+        contract_fingerprint=(
+            DURATION_REFERENCE_CONTRACT_FINGERPRINT_SHA256
+        ),
+        schema_fingerprint=DURATION_REFERENCE_SCHEMA_FINGERPRINT_SHA256,
+    )
+    rows = list(
+        RouteDurationDerivedRoute.objects.select_for_update(of=("self",))
+        .select_related("destination_airport")
+        .filter(parse_run=run)
+        .order_by("destination_airport__iata_code")
+    )
+    expected_codes = {row[1] for row in CURATED_AIRPORT_ROWS}
+    observed_codes = {row.destination_airport.iata_code for row in rows}
+    if (
+        run.artifact.source.official_locator != DURATION_SOURCE_LOCATOR
+        or run.artifact.source.secret_reference_name != ""
+        or locked.reference_locator != DURATION_SOURCE_LOCATOR
+        or locked.derivation_contract_fingerprint_sha256
+        != DURATION_DERIVATION_FINGERPRINT_SHA256
+        or len(locked.reference_version_sha256) != 64
+        or any(
+            character not in "0123456789abcdef"
+            for character in locked.reference_version_sha256
+        )
+        or locked.route_count != len(expected_codes)
+        or len(rows) != len(expected_codes)
+        or observed_codes != expected_codes
+        or locked.source_row_count < 1
+        or locked.created_at < run.completed_at
+    ):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+    for row in rows:
+        if (
+            row.parse_run_id != run.pk
+            or not row.destination_airport.is_public
+            or row.typed_fingerprint_sha256
+            != _fingerprint(_derived_route_payload_from_model(row))
+            or row.created_at < run.completed_at
+        ):
+            raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+    if locked.typed_fingerprint_sha256 != _fingerprint(
+        _duration_derivation_payload_from_models(locked, rows)
+    ):
+        raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+    return locked, rows
+
+
 def _aware(value: datetime) -> bool:
     return isinstance(value, datetime) and timezone.is_aware(value)
 
@@ -525,9 +643,10 @@ def stage_flight_evidence(
     city_reference: CityReferenceParseSuccess,
     legacy_arrivals_run: ParseRun,
     legacy_arrivals: LegacyArrivalParseSuccess,
-    duration_receipt: ReviewedDurationReceipt,
-    durations: DurationParseSuccess,
     source_checked_at: datetime,
+    duration_receipt: ReviewedDurationReceipt | None = None,
+    durations: DurationParseSuccess | None = None,
+    duration_derivation: RouteDurationDerivation | None = None,
 ) -> FlightPublicationOutcome:
     """Persist one immutable, typed candidate without moving the pointer.
 
@@ -535,12 +654,22 @@ def stage_flight_evidence(
     publish because no actor, review, audit, or pointer mutation exists here.
     """
 
+    v1_duration_mode = (
+        isinstance(duration_receipt, ReviewedDurationReceipt)
+        and isinstance(durations, DurationParseSuccess)
+        and duration_derivation is None
+    )
+    v2_duration_mode = (
+        isinstance(duration_derivation, RouteDurationDerivation)
+        and duration_receipt is None
+        and durations is None
+    )
     if (
         not isinstance(schedule, ScheduleParseSuccess)
         or not isinstance(city_reference, CityReferenceParseSuccess)
         or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
-        or not isinstance(durations, DurationParseSuccess)
         or not _aware(source_checked_at)
+        or v1_duration_mode == v2_duration_mode
     ):
         return FlightPublicationOutcome(FlightPublicationCode.INVALID_EVIDENCE)
     try:
@@ -576,15 +705,28 @@ def stage_flight_evidence(
                     LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
                 ),
             )
-            locked_duration_receipt = _approved_duration_receipt(
-                duration_receipt
-            )
+            locked_duration_receipt = None
+            locked_duration_derivation = None
+            derived_duration_rows: list[RouteDurationDerivedRoute] = []
+            if duration_derivation is not None:
+                (
+                    locked_duration_derivation,
+                    derived_duration_rows,
+                ) = _approved_duration_derivation(duration_derivation)
+                duration_checked_at = (
+                    locked_duration_derivation.parse_run.completed_at
+                )
+            else:
+                locked_duration_receipt = _approved_duration_receipt(
+                    duration_receipt
+                )
+                duration_checked_at = locked_duration_receipt.reviewed_at
             effective_checked_at = max(
                 source_checked_at,
                 locked_schedule_run.completed_at,
                 locked_city_reference_run.completed_at,
                 locked_legacy_arrivals_run.completed_at,
-                locked_duration_receipt.reviewed_at,
+                duration_checked_at,
             )
             if (
                 schedule.observed_schema_fingerprint_sha256
@@ -593,8 +735,11 @@ def stage_flight_evidence(
                 != CITY_SCHEMA_FINGERPRINT_SHA256
                 or legacy_arrivals.observed_schema_fingerprint_sha256
                 != LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
-                or durations.observed_schema_fingerprint_sha256
-                != DURATION_SCHEMA_FINGERPRINT_SHA256
+                or (
+                    durations is not None
+                    and durations.observed_schema_fingerprint_sha256
+                    != DURATION_V1_SCHEMA_FINGERPRINT_SHA256
+                )
             ):
                 raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
 
@@ -609,7 +754,18 @@ def stage_flight_evidence(
             iata_codes = {
                 row.destination_iata for row in projected_schedule.flights
             }
-            duration_codes = {row.destination_iata for row in durations.routes}
+            if locked_duration_derivation is not None:
+                duration_rows = derived_duration_rows
+            else:
+                duration_rows = list(durations.routes)
+            duration_codes = {
+                (
+                    row.destination_airport.iata_code
+                    if isinstance(row, RouteDurationDerivedRoute)
+                    else row.destination_iata
+                )
+                for row in duration_rows
+            }
             if not iata_codes or not iata_codes.issubset(duration_codes):
                 raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
             airports = {
@@ -656,28 +812,62 @@ def stage_flight_evidence(
                 ]
             )
             duration_revisions = []
-            for row in durations.routes:
-                if row.destination_iata not in iata_codes:
+            for row in duration_rows:
+                destination_iata = (
+                    row.destination_airport.iata_code
+                    if isinstance(row, RouteDurationDerivedRoute)
+                    else row.destination_iata
+                )
+                if destination_iata not in iata_codes:
                     continue
+                if locked_duration_derivation is not None:
+                    reference_date = locked_duration_derivation.reference_date
+                    reference_locator = (
+                        locked_duration_derivation.reference_locator
+                    )
+                    reviewed_by = "ROUTE_DURATION_DERIVATION_V1"
+                    reviewed_at = (
+                        locked_duration_derivation.parse_run.completed_at
+                    )
+                    created_at = max(
+                        timezone.now(),
+                        locked_duration_derivation.created_at,
+                    )
+                    parse_run = locked_duration_derivation.parse_run
+                    receipt = None
+                else:
+                    reference_date = row.reference_date
+                    reference_locator = row.reference_locator
+                    reviewed_by = locked_duration_receipt.reviewed_by
+                    reviewed_at = locked_duration_receipt.reviewed_at
+                    created_at = max(
+                        timezone.now(),
+                        locked_duration_receipt.created_at,
+                    )
+                    parse_run = None
+                    receipt = locked_duration_receipt
                 duration_revisions.append(
                     RouteDurationRevision.objects.create(
-                        parse_run=None,
-                        reviewed_duration_receipt=locked_duration_receipt,
-                        destination_airport=airports[row.destination_iata],
+                        parse_run=parse_run,
+                        reviewed_duration_receipt=receipt,
+                        destination_airport=airports[destination_iata],
                         outbound_minutes=row.outbound_minutes,
                         inbound_minutes=row.inbound_minutes,
-                        reference_date=row.reference_date,
-                        reference_locator=row.reference_locator,
+                        reference_date=reference_date,
+                        reference_locator=reference_locator,
                         state=RouteDurationRevision.State.VALIDATED,
-                        reviewed_by=locked_duration_receipt.reviewed_by,
-                        reviewed_at=locked_duration_receipt.reviewed_at,
+                        reviewed_by=reviewed_by,
+                        reviewed_at=reviewed_at,
                         typed_fingerprint_sha256=_fingerprint(
-                            _duration_payload_from_parsed(row)
-                        ),
-                        created_at=max(
-                            timezone.now(),
-                            locked_duration_receipt.created_at,
+                            {
+                                "destination_iata": destination_iata,
+                                "outbound_minutes": row.outbound_minutes,
+                                "inbound_minutes": row.inbound_minutes,
+                                "reference_date": reference_date.isoformat(),
+                                "reference_locator": reference_locator,
+                            }
                         ),
+                        created_at=created_at,
                     )
                 )
             FlightCandidateDuration.objects.bulk_create(
@@ -930,6 +1120,13 @@ def _load_valid_candidate(
         raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
     validated_runs: set[object] = set()
     validated_receipts: set[object] = set()
+    validated_derivations: dict[
+        object,
+        tuple[
+            RouteDurationDerivation,
+            dict[str, RouteDurationDerivedRoute],
+        ],
+    ] = {}
     duration_identities: list[tuple[str, str]] = []
     duration_fingerprints: list[tuple[RouteDurationRevision, str]] = []
     for link in links:
@@ -969,17 +1166,62 @@ def _load_valid_candidate(
                         FlightPublicationCode.INVALID_EVIDENCE
                     )
                 validated_receipts.add(receipt.pk)
+        elif (
+            duration.parse_run.parser_name
+            == ParseRun.ParserName.ROUTE_DURATION_REFERENCE_FILE
+            and duration.parse_run.parser_version
+            == ParseRun.ParserVersion.V2
+        ):
+            if duration.parse_run_id not in validated_derivations:
+                try:
+                    derivation = duration.parse_run.duration_derivation
+                except RouteDurationDerivation.DoesNotExist:
+                    raise _PublicationRejected(
+                        FlightPublicationCode.INVALID_EVIDENCE
+                    ) from None
+                locked_derivation, derived_rows = (
+                    _approved_duration_derivation(derivation)
+                )
+                validated_derivations[duration.parse_run_id] = (
+                    locked_derivation,
+                    {
+                        row.destination_airport.iata_code: row
+                        for row in derived_rows
+                    },
+                )
+            locked_derivation, derived_rows_by_code = validated_derivations[
+                duration.parse_run_id
+            ]
+            derived = derived_rows_by_code.get(
+                duration.destination_airport.iata_code
+            )
+            if (
+                derived is None
+                or duration.outbound_minutes != derived.outbound_minutes
+                or duration.inbound_minutes != derived.inbound_minutes
+                or duration.reference_date != locked_derivation.reference_date
+                or duration.reference_locator
+                != locked_derivation.reference_locator
+                or duration.reviewed_by != "ROUTE_DURATION_DERIVATION_V1"
+                or duration.reviewed_at
+                != locked_derivation.parse_run.completed_at
+                or duration.created_at < locked_derivation.created_at
+            ):
+                raise _PublicationRejected(
+                    FlightPublicationCode.INVALID_EVIDENCE
+                )
         elif not allow_legacy_lineage:
             raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
         elif duration.parse_run_id not in validated_runs:
             _approved_run(
                 duration.parse_run,
                 source_code=DURATION_SOURCE_CODE,
-                source_revision=DURATION_SOURCE_REVISION,
+                source_revision=DURATION_V1_SOURCE_REVISION,
                 parser_name=ParseRun.ParserName.ROUTE_DURATION_CSV,
                 parser_version=ParseRun.ParserVersion.V1,
-                contract_fingerprint=DURATION_CONTRACT_FINGERPRINT_SHA256,
-                schema_fingerprint=DURATION_SCHEMA_FINGERPRINT_SHA256,
+                contract_fingerprint=DURATION_V1_CONTRACT_FINGERPRINT_SHA256,
+                schema_fingerprint=DURATION_V1_SCHEMA_FINGERPRINT_SHA256,
+                allow_historical_source_revision=True,
             )
             validated_runs.add(duration.parse_run_id)
     try:
diff --git a/travel_windows/tests/test_publication_sealing.py b/travel_windows/tests/test_publication_sealing.py
index 1c77ab9..8fa39db 100644
--- a/travel_windows/tests/test_publication_sealing.py
+++ b/travel_windows/tests/test_publication_sealing.py
@@ -6,9 +6,6 @@ from django.test import TransactionTestCase
 from django.utils import timezone
 
 from operations.tests import migration_helpers as _migration_helpers  # noqa: F401
-from sources.management.commands.register_approved_sources import (
-    register_approved_sources,
-)
 from travel_windows.contracts import DURATION_SOURCE_LOCATOR
 from travel_windows.ingestion import FlightIngestionCode
 from travel_windows.models import (
@@ -29,12 +26,13 @@ from travel_windows.tests.test_source_publication import (
     legacy_arrival_row,
     official_page,
     official_row,
+    register_sources_with_duration_v1_history,
 )
 
 
 class PublishedFlightEvidenceSealingTests(TransactionTestCase):
     def setUp(self):
-        register_approved_sources(apply=True)
+        register_sources_with_duration_v1_history()
         outcome = collect_and_stage_fixture(
             departure_pages=(official_page([official_row()]),),
             arrival_pages=(
diff --git a/travel_windows/tests/test_review_lifecycle.py b/travel_windows/tests/test_review_lifecycle.py
index abb1a8a..5f1f41b 100644
--- a/travel_windows/tests/test_review_lifecycle.py
+++ b/travel_windows/tests/test_review_lifecycle.py
@@ -10,9 +10,6 @@ from django.test import SimpleTestCase, TransactionTestCase
 from django.utils import timezone
 
 from countries.models import Country, SUPPORTED_COUNTRY_ROWS
-from sources.management.commands.register_approved_sources import (
-    register_approved_sources,
-)
 from travel_windows.contracts import (
     DURATION_SOURCE_LOCATOR,
     SCHEDULE_ATTRIBUTION,
@@ -48,6 +45,7 @@ from travel_windows.tests.test_source_publication import (
     legacy_arrival_row,
     official_page,
     official_row,
+    register_sources_with_duration_v1_history,
 )
 
 
@@ -72,7 +70,7 @@ class FlightAdminBoundaryTests(SimpleTestCase):
 
 class FlightReviewLifecycleTests(TransactionTestCase):
     def setUp(self):
-        register_approved_sources(apply=True)
+        register_sources_with_duration_v1_history()
         airport_row = CURATED_AIRPORT_ROWS[0]
         country_row = next(
             row for row in SUPPORTED_COUNTRY_ROWS if row[1] == airport_row[2]
diff --git a/travel_windows/tests/test_search.py b/travel_windows/tests/test_search.py
index 0de9b8b..38b9878 100644
--- a/travel_windows/tests/test_search.py
+++ b/travel_windows/tests/test_search.py
@@ -21,9 +21,6 @@ from entry_requirements.ingestion import (
     MULTI_COUNTRY_ENTRY_SOURCE_REVISION,
 )
 from operations.tests import migration_helpers as _migration_helpers  # noqa: F401
-from sources.management.commands.register_approved_sources import (
-    register_approved_sources,
-)
 from travel_windows.contracts import DURATION_SOURCE_LOCATOR
 from travel_windows.ingestion import FlightIngestionCode
 from travel_windows.models import (
@@ -58,6 +55,7 @@ from travel_windows.tests.test_source_publication import (
     collect_and_stage_fixture,
     legacy_arrival_page,
     legacy_arrival_row,
+    register_sources_with_duration_v1_history,
 )
 from travel_warnings.ingestion import (
     LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
@@ -747,7 +745,7 @@ def _official_page(rows):
 
 class CurrentFlightPublicationSearchTests(TransactionTestCase):
     def setUp(self):
-        register_approved_sources(apply=True)
+        register_sources_with_duration_v1_history()
         row = next(row for row in CURATED_AIRPORT_ROWS if row[1] == "NRT")
         country_row = next(
             country
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 5c3b5dc..c3b5121 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -12,6 +12,7 @@ from django.utils import timezone
 
 from countries.models import Country, SUPPORTED_COUNTRY_ROWS
 from sources.management.commands.register_approved_sources import (
+    APPROVED_SOURCE_SPECS,
     register_approved_sources,
 )
 from sources.models import (
@@ -25,6 +26,15 @@ from sources.models import (
 )
 from sources.tests.test_transport import FakeConnection, FakeResponse
 from sources.transport import (
+    ATTEMPT_SUCCEEDED,
+    ROLE_DURATION_ARCHIVE,
+    ROLE_DURATION_LIMIT,
+    ROLE_DURATION_METADATA,
+    ROLE_DURATION_PAGE,
+    AviationResponseLineage,
+    RequestFingerprint,
+    RouteDurationReferenceFetchResult,
+    SingleAttemptResult,
     fetch_data_go_reference_evidence,
     fetch_data_go_schedule_pages,
     load_aviation_secret_reference,
@@ -33,6 +43,8 @@ from travel_windows.ingestion import (
     DurableAviationAttemptRecorder,
     FlightIngestionCode,
     ingest_flight_evidence_candidate,
+    ingest_flight_evidence_candidate_with_derivation,
+    ingest_route_duration_reference,
 )
 from travel_windows.management.commands.publish_scheduled_flights import Command
 from travel_windows.contracts import (
@@ -48,8 +60,11 @@ from travel_windows.models import (
     CURATED_AIRPORT_ROWS,
     FlightSchedule,
     FlightScheduleRevision,
+    FlightPublication,
+    FlightReviewDecision,
     PublishedFlightSchedule,
     ReviewedDurationReceipt,
+    RouteDurationDerivation,
     RouteDurationRevision,
     TIMEZONE_EVIDENCE_LOCATOR,
 )
@@ -60,6 +75,11 @@ from travel_windows.parser import (
     parse_legacy_arrival_pages,
     parse_route_durations,
 )
+from travel_windows.publication import (
+    FlightPublicationCode,
+    approve_flight_candidate,
+)
+from travel_windows.tests.test_duration_reference import _archive, _row
 
 
 def official_row(**overrides):
@@ -279,6 +299,132 @@ def collect_and_stage_fixture(
     )
 
 
+def register_sources_with_duration_v1_history():
+    """Model an existing v1 deployment before applying the v2 registry."""
+
+    current = next(
+        spec
+        for spec in APPROVED_SOURCE_SPECS
+        if spec.code == DURATION_SOURCE_CODE
+    )
+    (prior_v1,) = current.prior_contracts
+    source = SourceConfiguration.objects.create(
+        **prior_v1.configuration_values(),
+        state=SourceConfiguration.State.DRAFT,
+        enabled=False,
+    )
+    SourceRightsDecision.objects.create(
+        source=source,
+        **prior_v1.rights_values(),
+    )
+    SourceConfiguration.objects.filter(pk=source.pk).update(
+        state=SourceConfiguration.State.RIGHTS_APPROVED
+    )
+    SourceConfiguration.objects.filter(pk=source.pk).update(
+        state=SourceConfiguration.State.ACTIVE,
+        enabled=True,
+    )
+    register_approved_sources(apply=True)
+
+
+def collect_duration_derivation_fixture(*, source_checked_at):
+    countries = {}
+    for values in SUPPORTED_COUNTRY_ROWS:
+        country, _created = Country.objects.get_or_create(
+            id=values[0],
+            defaults={
+                "iso_alpha2": values[1],
+                "name_ko": values[2],
+                "name_en": values[3],
+                "is_public": values[4],
+            },
+        )
+        countries[values[1]] = country
+    for values in CURATED_AIRPORT_ROWS:
+        Airport.objects.get_or_create(
+            id=values[0],
+            defaults={
+                "iata_code": values[1],
+                "country": countries[values[2]],
+                "city_code": values[3],
+                "city_name_ko": values[4],
+                "name_ko": values[5],
+                "iana_timezone": values[6],
+                "timezone_evidence_locator": TIMEZONE_EVIDENCE_LOCATOR,
+                "is_public": True,
+            },
+        )
+    rows = []
+    for route_number, airport_row in enumerate(CURATED_AIRPORT_ROWS):
+        iata = airport_row[1]
+        for day in range(5):
+            outbound = _row(
+                direction="OUTBOUND",
+                day=route_number * 5 + day,
+                duration="120",
+            )
+            inbound = _row(
+                direction="INBOUND",
+                day=route_number * 5 + day,
+                duration="130",
+            )
+            outbound[4] = iata
+            inbound[3] = iata
+            rows.extend((outbound, inbound))
+    archive = _archive(rows)
+    recorder = DurableAviationAttemptRecorder()
+    lineages = []
+    bodies = (
+        (ROLE_DURATION_PAGE, b"synthetic-page"),
+        (ROLE_DURATION_METADATA, b"synthetic-metadata"),
+        (ROLE_DURATION_LIMIT, b'{"needCaptcha":false}'),
+        (ROLE_DURATION_ARCHIVE, archive),
+    )
+    for sequence, (role, body) in enumerate(bodies, start=1):
+        fingerprint = RequestFingerprint(
+            revision="DURATION_TEST_V1",
+            normalized_request_sha256=hashlib.sha256(
+                f"duration-test-{sequence}".encode("ascii")
+            ).hexdigest(),
+        )
+        attempted = SingleAttemptResult(
+            request_fingerprint=fingerprint,
+            attempt_result=ATTEMPT_SUCCEEDED,
+            http_status=200,
+            response_sha256=hashlib.sha256(body).hexdigest(),
+            byte_count=len(body),
+            body=body,
+        )
+        recorded = recorder(
+            source_code=DURATION_SOURCE_CODE,
+            request_fingerprint=fingerprint,
+            call_once=lambda attempted=attempted: attempted,
+        )
+        lineages.append(
+            AviationResponseLineage(
+                source_code=DURATION_SOURCE_CODE,
+                role=role,
+                role_sequence=1,
+                attempt=recorded,
+            )
+        )
+    fetch = RouteDurationReferenceFetchResult(
+        succeeded=True,
+        reference_date="2024-12-12",
+        reference_version_sha256=hashlib.sha256(
+            b"synthetic-attachment-v1"
+        ).hexdigest(),
+        archive=archive,
+        lineage=tuple(lineages),
+    )
+    outcome = ingest_route_duration_reference(
+        fetch=fetch,
+        attempt_recorder=recorder,
+        source_checked_at=source_checked_at,
+    )
+    return outcome
+
+
 class DataGoScheduleAdapterTests(SimpleTestCase):
     def test_direct_list_schedule_matches_the_documented_item_wrapper(self):
         departure_rows = [official_row()]
@@ -600,6 +746,88 @@ class DataGoScheduleAdapterTests(SimpleTestCase):
 
 
 class FlightPublicationCommandTests(SimpleTestCase):
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "Command.requires_migrations_checks",
+        False,
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "ingest_flight_evidence_candidate_with_derivation"
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "ingest_flight_evidence_candidate"
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "fetch_data_go_reference_evidence"
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "fetch_data_go_schedule_pages"
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "load_aviation_secret_reference",
+        return_value="synthetic-secret",
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "Path.read_bytes"
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "SourceConfiguration.objects"
+    )
+    def test_command_uses_exact_official_duration_derivation(
+        self,
+        source_objects,
+        read_bytes,
+        _secret,
+        schedule_fetch,
+        reference_fetch,
+        legacy_ingest,
+        derived_ingest,
+    ):
+        source = SimpleNamespace(
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+        )
+        source_objects.get.return_value = source
+        source_objects.filter.return_value = [source, source]
+        schedule_fetch.return_value = SimpleNamespace(succeeded=True)
+        reference_fetch.return_value = SimpleNamespace(succeeded=True)
+        derived_ingest.return_value = SimpleNamespace(
+            succeeded=True,
+            schedule_revision_id="00000000-0000-0000-0000-000000000009",
+        )
+        output = io.StringIO()
+        derivation_id = "00000000-0000-0000-0000-000000000010"
+
+        call_command(
+            "publish_scheduled_flights",
+            duration_derivation_id=derivation_id,
+            source_date="2026-08-31",
+            season="S26",
+            stdout=output,
+        )
+
+        read_bytes.assert_not_called()
+        legacy_ingest.assert_not_called()
+        kwargs = derived_ingest.call_args.kwargs
+        self.assertEqual(str(kwargs["duration_derivation_id"]), derivation_id)
+        self.assertIs(
+            kwargs["attempt_recorder"],
+            schedule_fetch.call_args.kwargs["attempt_executor"],
+        )
+        self.assertEqual(
+            output.getvalue(),
+            "result=staged "
+            "schedule_revision=00000000-0000-0000-0000-000000000009\n",
+        )
+        self.assertNotIn("synthetic-secret", output.getvalue())
+
     @patch(
         "travel_windows.management.commands.publish_scheduled_flights."
         "Command.requires_migrations_checks",
@@ -703,7 +931,7 @@ class FlightPublicationCommandTests(SimpleTestCase):
 
 class FlightEvidencePublicationTests(TransactionTestCase):
     def setUp(self):
-        register_approved_sources(apply=True)
+        register_sources_with_duration_v1_history()
         row = CURATED_AIRPORT_ROWS[0]
         country_row = next(
             country
@@ -733,6 +961,92 @@ class FlightEvidencePublicationTests(TransactionTestCase):
             },
         )
 
+    def test_official_derivation_stages_parse_lineage_and_still_requires_operator(self):
+        checked_at = timezone.now()
+        duration_outcome = collect_duration_derivation_fixture(
+            source_checked_at=checked_at
+        )
+        self.assertEqual(duration_outcome.code, FlightIngestionCode.STAGED)
+        derivation = RouteDurationDerivation.objects.get(
+            pk=duration_outcome.derivation_id
+        )
+        self.assertEqual(derivation.route_count, len(CURATED_AIRPORT_ROWS))
+
+        departure = official_page([official_row()], direct_items=True)
+        arrival = official_page(
+            [
+                official_row(
+                    flightId="KE704",
+                    masterFlightId="KE704",
+                    st="2015",
+                )
+            ],
+            direct_items=True,
+        )
+        city_payload = city_reference_payload(direct_items=True)
+        legacy_payload = legacy_arrival_page(
+            [legacy_arrival_row()],
+            direct_items=True,
+        )
+        recorder = DurableAviationAttemptRecorder()
+        schedule_connections = [
+            FakeConnection(FakeResponse(200, departure)),
+            FakeConnection(FakeResponse(200, arrival)),
+        ]
+        schedule_fetch = fetch_data_go_schedule_pages(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-collection-key",
+            season="S26",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=lambda _host, _port, _timeout: (
+                schedule_connections.pop(0)
+            ),
+            attempt_executor=recorder,
+        )
+        reference_connections = [
+            FakeConnection(FakeResponse(200, city_payload)),
+            FakeConnection(FakeResponse(200, legacy_payload)),
+        ]
+        reference_fetch = fetch_data_go_reference_evidence(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value="synthetic-collection-key",
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=lambda _host, _port, _timeout: (
+                reference_connections.pop(0)
+            ),
+            attempt_executor=recorder,
+        )
+
+        staged = ingest_flight_evidence_candidate_with_derivation(
+            schedule_fetch=schedule_fetch,
+            reference_fetch=reference_fetch,
+            attempt_recorder=recorder,
+            duration_derivation_id=derivation.pk,
+            source_date=date(2026, 8, 31),
+            source_checked_at=checked_at,
+        )
+
+        self.assertEqual(staged.code, FlightIngestionCode.STAGED)
+        duration = RouteDurationRevision.objects.get()
+        self.assertEqual(duration.parse_run_id, derivation.parse_run_id)
+        self.assertIsNone(duration.reviewed_duration_receipt_id)
+        self.assertEqual(
+            duration.reviewed_by,
+            "ROUTE_DURATION_DERIVATION_V1",
+        )
+        self.assertFalse(FlightReviewDecision.objects.exists())
+        self.assertFalse(FlightPublication.objects.exists())
+        blocked = approve_flight_candidate(
+            schedule_revision_id=staged.schedule_revision_id,
+            actor=SimpleNamespace(is_authenticated=False, pk=None),
+            expected_pointer_version=0,
+        )
+        self.assertEqual(blocked.code, FlightPublicationCode.NOT_AUTHORIZED)
+        self.assertFalse(FlightReviewDecision.objects.exists())
+        self.assertFalse(FlightPublication.objects.exists())
+
     def test_receipts_and_typed_candidate_persist_without_raw_body_or_publication(self):
         departure = official_page([official_row()], direct_items=True)
         arrival = official_page(


