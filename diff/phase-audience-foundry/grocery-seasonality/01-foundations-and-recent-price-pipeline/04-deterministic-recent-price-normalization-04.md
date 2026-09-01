## `feat(source): persist parsed retail generation`

diff --git a/grocery/source/generation.py b/grocery/source/generation.py
new file mode 100644
index 0000000..1fd80fd
--- /dev/null
+++ b/grocery/source/generation.py
@@ -0,0 +1,574 @@
+"""Persist one deterministic KAMIS parse generation without retaining raw rows."""
+
+from __future__ import annotations
+
+import hashlib
+import json
+import re
+import uuid
+from dataclasses import dataclass
+from datetime import date
+from decimal import Decimal
+from enum import StrEnum
+from typing import Final
+
+from django.db import transaction
+from django.utils import timezone
+
+from grocery.models import (
+    ParseRun,
+    PriceChangeFact,
+    PriceSeriesKey,
+    ReferencePrice,
+    RetailPriceSnapshot,
+    SourceArtifact,
+    persist_reference_price_facts,
+)
+from grocery.source.configuration import KAMIS_DATASET_ID, KAMIS_INTERFACE_REVISION
+from grocery.source.kamis import (
+    KAMIS_RETAIL_PRODUCT_CLASS_CODE,
+    IdentityObservation,
+    ParsedRecentPriceResult,
+    ParsedRetailPriceRow,
+    ReferenceDateStatus,
+    ReferencePeriod,
+    ValueStatus,
+)
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+
+KAMIS_PARSER_REVISION: Final = "kamis-recent-price-v1"
+KAMIS_SOURCE_CONTRACT_REVISION: Final = "data-go-15156063-recent-v1"
+KAMIS_ARTIFACT_SOURCE_IDENTITY: Final = ":".join(
+    (
+        KAMIS_DATASET_ID,
+        KAMIS_INTERFACE_REVISION,
+        "RECENT_COMPARISON",
+        INITIAL_RETAIL_IDENTITY_REGISTRY.coverage_identity,
+    )
+)
+
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
+_MAX_SCALE_ZERO_PRICE = Decimal("999999999999")
+_REFERENCE_PERIODS: Final = tuple(ReferencePeriod)
+
+
+def _parse_configuration_sha256() -> str:
+    registry = INITIAL_RETAIL_IDENTITY_REGISTRY
+    canonical_configuration = {
+        "coverage_identity": registry.coverage_identity,
+        "evidence": {
+            "codebook_sha256": registry.evidence.codebook_sha256,
+            "coverage_evidence_revision": registry.evidence.coverage_evidence_revision,
+            "unit_contract_sha256": registry.evidence.unit_contract_sha256,
+        },
+        "grade_names": [[*key, value] for key, value in sorted(registry.grade_names.items())],
+        "item_names": [[*key, value] for key, value in sorted(registry.item_names.items())],
+        "parser_revision": KAMIS_PARSER_REVISION,
+        "source_contract_revision": KAMIS_SOURCE_CONTRACT_REVISION,
+        "units": [
+            [*key, unit, unit_size] for key, (unit, unit_size) in sorted(registry.units.items())
+        ],
+        "variety_names": [[*key, value] for key, value in sorted(registry.variety_names.items())],
+    }
+    canonical = json.dumps(
+        canonical_configuration,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+KAMIS_PARSE_CONFIGURATION_SHA256: Final = _parse_configuration_sha256()
+
+
+class ParseGenerationFailureCode(StrEnum):
+    """Fixed, non-sensitive outcomes allowed on a failed parse run."""
+
+    SCHEMA_INVALID = "SCHEMA_INVALID"
+    IDENTITY_DRIFT = "IDENTITY_DRIFT"
+    RECONCILIATION_FAILED = "RECONCILIATION_FAILED"
+    PERSISTENCE_FAILED = "PERSISTENCE_FAILED"
+
+
+class ParseGenerationErrorCode(StrEnum):
+    """Fixed, non-sensitive errors exposed by this persistence boundary."""
+
+    ARTIFACT_CONTRACT_INVALID = "ARTIFACT_CONTRACT_INVALID"
+    ARTIFACT_NOT_FOUND = "ARTIFACT_NOT_FOUND"
+    CONFIGURATION_CONTRACT_DRIFT = "CONFIGURATION_CONTRACT_DRIFT"
+    FAILURE_CODE_INVALID = "FAILURE_CODE_INVALID"
+    NONDETERMINISTIC_REPLAY = "NONDETERMINISTIC_REPLAY"
+    PARSE_RUN_NOT_STARTED = "PARSE_RUN_NOT_STARTED"
+    PARSE_RUN_NOT_FOUND = "PARSE_RUN_NOT_FOUND"
+    PERSISTENCE_CONFLICT = "PERSISTENCE_CONFLICT"
+    RESULT_CONTRACT_INVALID = "RESULT_CONTRACT_INVALID"
+    RESULT_RECONCILIATION_FAILED = "RESULT_RECONCILIATION_FAILED"
+
+
+class ParseGenerationError(RuntimeError):
+    """A persistence failure whose string contains only a fixed safe code."""
+
+    def __init__(self, code: ParseGenerationErrorCode) -> None:
+        self.code = code
+        super().__init__(code.value)
+
+
+@dataclass(frozen=True, slots=True)
+class StartedParseGeneration:
+    parse_run: ParseRun
+    created: bool
+
+
+@dataclass(frozen=True, slots=True)
+class CompletedParseGeneration:
+    parse_run: ParseRun
+    snapshots: tuple[RetailPriceSnapshot, ...]
+    replayed: bool
+
+
+def start_or_get_kamis_parse_run(artifact_id: uuid.UUID) -> StartedParseGeneration:
+    """Start the sealed parser contract, or return its existing idempotency row."""
+
+    try:
+        with transaction.atomic():
+            artifact = SourceArtifact.objects.select_for_update().get(pk=artifact_id)
+            _validate_artifact(artifact)
+            _validate_configuration_constant()
+            parse_run, created = ParseRun.objects.get_or_create(
+                artifact=artifact,
+                parser_revision=KAMIS_PARSER_REVISION,
+                configuration_hash=KAMIS_PARSE_CONFIGURATION_SHA256,
+            )
+            return StartedParseGeneration(parse_run=parse_run, created=created)
+    except SourceArtifact.DoesNotExist:
+        raise ParseGenerationError(ParseGenerationErrorCode.ARTIFACT_NOT_FOUND) from None
+    except ParseGenerationError:
+        raise
+    except Exception:
+        raise ParseGenerationError(ParseGenerationErrorCode.PERSISTENCE_CONFLICT) from None
+
+
+def complete_kamis_parse_generation(
+    parse_run_id: uuid.UUID,
+    result: ParsedRecentPriceResult,
+) -> CompletedParseGeneration:
+    """Atomically validate and persist one complete ten-series parse generation."""
+
+    try:
+        with transaction.atomic():
+            parse_run = (
+                ParseRun.objects.select_for_update().select_related("artifact").get(pk=parse_run_id)
+            )
+            _validate_parse_run_contract(parse_run)
+            if parse_run.status == ParseRun.Status.VALIDATED:
+                if parse_run.result_hash != result.result_hash:
+                    raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+                missing_reference_count = _validate_result(result)
+                snapshots = _validate_persisted_replay(
+                    parse_run,
+                    result,
+                    missing_reference_count=missing_reference_count,
+                )
+                return CompletedParseGeneration(
+                    parse_run=parse_run,
+                    snapshots=snapshots,
+                    replayed=True,
+                )
+            if parse_run.status != ParseRun.Status.STARTED:
+                raise ParseGenerationError(ParseGenerationErrorCode.PARSE_RUN_NOT_STARTED)
+
+            missing_reference_count = _validate_result(result)
+            parse_run.status = ParseRun.Status.VALIDATED
+            parse_run.completed_at = timezone.now()
+            parse_run.result_hash = result.result_hash
+            parse_run.total_row_count = result.input_row_count
+            parse_run.accepted_row_count = result.accepted_row_count
+            parse_run.missing_reference_row_count = missing_reference_count
+            parse_run.out_of_scope_row_count = result.out_of_scope_row_count
+            parse_run.quarantined_row_count = 0
+            parse_run.failure_code = ""
+            parse_run.save()
+
+            for row in result.rows:
+                series = _get_or_validate_series(row)
+                snapshot = RetailPriceSnapshot.get_or_validate(
+                    parse_run_id=parse_run.id,
+                    series_id=series.id,
+                    source_effective_date=row.source_effective_date,
+                    source_recorded_at=None,
+                    current_price=row.current_price,
+                    source_row_sha256=row.source_row_hash,
+                    source_contract_revision=KAMIS_SOURCE_CONTRACT_REVISION,
+                )
+                persist_reference_price_facts(
+                    snapshot_id=snapshot.id,
+                    reference_values={
+                        reference.period.value: reference.value for reference in row.references
+                    },
+                )
+
+            snapshots = _validate_persisted_replay(
+                parse_run,
+                result,
+                missing_reference_count=missing_reference_count,
+            )
+            return CompletedParseGeneration(
+                parse_run=parse_run,
+                snapshots=snapshots,
+                replayed=False,
+            )
+    except ParseRun.DoesNotExist:
+        raise ParseGenerationError(ParseGenerationErrorCode.PARSE_RUN_NOT_FOUND) from None
+    except ParseGenerationError:
+        raise
+    except Exception:
+        raise ParseGenerationError(ParseGenerationErrorCode.PERSISTENCE_CONFLICT) from None
+
+
+def fail_kamis_parse_run(
+    parse_run_id: uuid.UUID,
+    failure_code: ParseGenerationFailureCode,
+) -> ParseRun:
+    """Finalize a still-started run with an allowlisted, fixed failure code."""
+
+    if not isinstance(failure_code, ParseGenerationFailureCode):
+        raise ParseGenerationError(ParseGenerationErrorCode.FAILURE_CODE_INVALID)
+    try:
+        with transaction.atomic():
+            parse_run = ParseRun.objects.select_for_update().get(pk=parse_run_id)
+            _validate_parse_run_contract(parse_run)
+            if parse_run.status != ParseRun.Status.STARTED:
+                raise ParseGenerationError(ParseGenerationErrorCode.PARSE_RUN_NOT_STARTED)
+            if parse_run.retail_price_snapshots.exists():
+                raise ParseGenerationError(ParseGenerationErrorCode.PERSISTENCE_CONFLICT)
+            parse_run.status = ParseRun.Status.FAILED
+            parse_run.completed_at = timezone.now()
+            parse_run.failure_code = failure_code.value
+            parse_run.save()
+            return parse_run
+    except ParseRun.DoesNotExist:
+        raise ParseGenerationError(ParseGenerationErrorCode.PARSE_RUN_NOT_FOUND) from None
+    except ParseGenerationError:
+        raise
+    except Exception:
+        raise ParseGenerationError(ParseGenerationErrorCode.PERSISTENCE_CONFLICT) from None
+
+
+def _validate_artifact(artifact: SourceArtifact) -> None:
+    if (
+        artifact.source_identity != KAMIS_ARTIFACT_SOURCE_IDENTITY
+        or artifact.media_type != SourceArtifact.MediaType.JSON
+        or artifact.encoding != SourceArtifact.Encoding.UTF_8
+        or artifact.retention_mode != SourceArtifact.RetentionMode.HASH_ONLY
+    ):
+        raise ParseGenerationError(ParseGenerationErrorCode.ARTIFACT_CONTRACT_INVALID)
+
+
+def _validate_configuration_constant() -> None:
+    if _parse_configuration_sha256() != KAMIS_PARSE_CONFIGURATION_SHA256:
+        raise ParseGenerationError(ParseGenerationErrorCode.CONFIGURATION_CONTRACT_DRIFT)
+
+
+def _validate_parse_run_contract(parse_run: ParseRun) -> None:
+    _validate_artifact(parse_run.artifact)
+    _validate_configuration_constant()
+    if (
+        parse_run.parser_revision != KAMIS_PARSER_REVISION
+        or parse_run.configuration_hash != KAMIS_PARSE_CONFIGURATION_SHA256
+    ):
+        raise ParseGenerationError(ParseGenerationErrorCode.CONFIGURATION_CONTRACT_DRIFT)
+
+
+def _canonical_sha256(value: object) -> str:
+    canonical = json.dumps(
+        value,
+        ensure_ascii=False,
+        sort_keys=True,
+        separators=(",", ":"),
+    ).encode("utf-8")
+    return hashlib.sha256(canonical).hexdigest()
+
+
+def _validate_result(result: ParsedRecentPriceResult) -> int:
+    if not isinstance(result, ParsedRecentPriceResult):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    if not isinstance(result.rows, tuple) or not isinstance(result.out_of_scope_row_hashes, tuple):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    integer_counts = (
+        result.input_row_count,
+        result.accepted_row_count,
+        result.out_of_scope_row_count,
+    )
+    if any(not isinstance(value, int) or isinstance(value, bool) for value in integer_counts):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED)
+    if any(value < 0 for value in integer_counts):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED)
+    if result.input_row_count != result.accepted_row_count + result.out_of_scope_row_count:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED)
+    if result.out_of_scope_row_count != len(result.out_of_scope_row_hashes):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED)
+    if tuple(sorted(result.out_of_scope_row_hashes)) != result.out_of_scope_row_hashes or any(
+        not isinstance(row_hash, str) or _SHA256.fullmatch(row_hash) is None
+        for row_hash in result.out_of_scope_row_hashes
+    ):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+
+    expected_identity_keys = tuple(
+        sorted(
+            (
+                KAMIS_RETAIL_PRODUCT_CLASS_CODE,
+                category_code,
+                item_code,
+                variety_code,
+                grade_code,
+                raw_unit,
+                raw_unit_size,
+                INITIAL_RETAIL_IDENTITY_REGISTRY.coverage_identity,
+            )
+            for (
+                category_code,
+                item_code,
+                variety_code,
+                grade_code,
+            ), (raw_unit, raw_unit_size) in INITIAL_RETAIL_IDENTITY_REGISTRY.units.items()
+        )
+    )
+    for row in result.rows:
+        _validate_row(row)
+    actual_identity_keys = tuple(row.semantic_identity_key for row in result.rows)
+    if actual_identity_keys != expected_identity_keys:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED)
+
+    missing_reference_count = 0
+    for row in result.rows:
+        if any(reference.value_status is ValueStatus.UNAVAILABLE for reference in row.references):
+            missing_reference_count += 1
+
+    expected_result_hash = _canonical_sha256(
+        {
+            "accepted_rows": [row.canonical_data() for row in result.rows],
+            "out_of_scope_row_hashes": result.out_of_scope_row_hashes,
+            "parser_contract": KAMIS_PARSER_REVISION,
+        }
+    )
+    if result.result_hash != expected_result_hash:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    return missing_reference_count
+
+
+def _validate_row(row: ParsedRetailPriceRow) -> None:
+    if not isinstance(row, ParsedRetailPriceRow):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    if type(row.source_effective_date) is not date:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    if not _valid_price(row.current_price):
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    if not isinstance(row.source_row_hash, str) or _SHA256.fullmatch(row.source_row_hash) is None:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+
+    observation = IdentityObservation(
+        product_class_code=row.product_class_code,
+        product_class_name=row.product_class_name,
+        category_code=row.category_code,
+        category_name=row.category_name,
+        item_code=row.item_code,
+        item_name=row.item_name,
+        variety_code=row.variety_code,
+        variety_name=row.variety_name,
+        grade_code=row.grade_code,
+        grade_name=row.grade_name,
+        raw_unit=row.raw_unit,
+        raw_unit_size=row.raw_unit_size,
+        coverage_identity=row.coverage_identity,
+    )
+    try:
+        INITIAL_RETAIL_IDENTITY_REGISTRY.validate(observation, row_index=0)
+    except Exception:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID) from None
+
+    if tuple(reference.period for reference in row.references) != _REFERENCE_PERIODS:
+        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+    for reference in row.references:
+        if (
+            reference.reference_date_status
+            is not ReferenceDateStatus.SOURCE_REFERENCE_DATE_UNAVAILABLE
+            or reference.source_reference_date is not None
+        ):
+            raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+        if reference.value_status is ValueStatus.AVAILABLE:
+            if not _valid_price(reference.value) or reference.unavailable_reason is not None:
+                raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+        elif reference.value_status is ValueStatus.UNAVAILABLE:
+            if (
+                reference.value is not None
+                or reference.unavailable_reason != "SOURCE_VALUE_MISSING"
+            ):
+                raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+        else:
+            raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
+
+
+def _valid_price(value: object) -> bool:
+    return (
+        isinstance(value, Decimal)
+        and value.is_finite()
+        and value > 0
+        and value <= _MAX_SCALE_ZERO_PRICE
+        and value.as_tuple().exponent == 0
+    )
+
+
+def _get_or_validate_series(row: ParsedRetailPriceRow) -> PriceSeriesKey:
+    return PriceSeriesKey.get_or_validate(
+        product_class_code=row.product_class_code,
+        product_class_name=row.product_class_name,
+        category_code=row.category_code,
+        category_name=row.category_name,
+        item_code=row.item_code,
+        item_name=row.item_name,
+        variety_code=row.variety_code,
+        variety_name=row.variety_name,
+        grade_code=row.grade_code,
+        grade_name=row.grade_name,
+        raw_unit=row.raw_unit,
+        raw_unit_size=row.raw_unit_size,
+        coverage_identity=row.coverage_identity,
+        identity_evidence_revision=(
+            INITIAL_RETAIL_IDENTITY_REGISTRY.evidence.coverage_evidence_revision
+        ),
+    )
+
+
+def _validate_persisted_replay(
+    parse_run: ParseRun,
+    result: ParsedRecentPriceResult,
+    *,
+    missing_reference_count: int,
+) -> tuple[RetailPriceSnapshot, ...]:
+    expected_run_fields: dict[str, object] = {
+        "result_hash": result.result_hash,
+        "total_row_count": result.input_row_count,
+        "accepted_row_count": result.accepted_row_count,
+        "missing_reference_row_count": missing_reference_count,
+        "out_of_scope_row_count": result.out_of_scope_row_count,
+        "quarantined_row_count": 0,
+        "failure_code": "",
+    }
+    if any(
+        getattr(parse_run, field_name) != expected_value
+        for field_name, expected_value in expected_run_fields.items()
+    ):
+        raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+
+    snapshots = tuple(
+        RetailPriceSnapshot.objects.select_for_update()
+        .select_related("series")
+        .filter(parse_run=parse_run)
+        .order_by(
+            "series__product_class_code",
+            "series__category_code",
+            "series__item_code",
+            "series__variety_code",
+            "series__grade_code",
+            "series__raw_unit",
+            "series__raw_unit_size",
+            "series__coverage_identity",
+        )
+    )
+    if len(snapshots) != len(result.rows):
+        raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+
+    references = tuple(
+        ReferencePrice.objects.select_for_update()
+        .filter(snapshot__parse_run=parse_run)
+        .order_by("snapshot_id", "period")
+    )
+    changes = tuple(
+        PriceChangeFact.objects.select_for_update().filter(
+            reference_price__snapshot__parse_run=parse_run
+        )
+    )
+    if len(references) != 3 * len(result.rows) or len(changes) != 3 * len(result.rows):
+        raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+    references_by_snapshot: dict[uuid.UUID, dict[str, ReferencePrice]] = {}
+    for reference in references:
+        references_by_snapshot.setdefault(reference.snapshot_id, {})[reference.period] = reference
+    changes_by_reference = {change.reference_price_id: change for change in changes}
+
+    for row, snapshot in zip(result.rows, snapshots, strict=True):
+        _validate_persisted_row(
+            row,
+            snapshot,
+            references_by_snapshot.get(snapshot.id, {}),
+            changes_by_reference,
+        )
+    return snapshots
+
+
+def _validate_persisted_row(
+    row: ParsedRetailPriceRow,
+    snapshot: RetailPriceSnapshot,
+    references: dict[str, ReferencePrice],
+    changes_by_reference: dict[uuid.UUID, PriceChangeFact],
+) -> None:
+    series = snapshot.series
+    expected_series: dict[str, object] = {
+        "product_class_code": row.product_class_code,
+        "product_class_name": row.product_class_name,
+        "category_code": row.category_code,
+        "category_name": row.category_name,
+        "item_code": row.item_code,
+        "item_name": row.item_name,
+        "variety_code": row.variety_code,
+        "variety_name": row.variety_name,
+        "grade_code": row.grade_code,
+        "grade_name": row.grade_name,
+        "raw_unit": row.raw_unit,
+        "raw_unit_size": row.raw_unit_size,
+        "coverage_identity": row.coverage_identity,
+        "identity_evidence_revision": (
+            INITIAL_RETAIL_IDENTITY_REGISTRY.evidence.coverage_evidence_revision
+        ),
+    }
+    expected_snapshot: dict[str, object] = {
+        "source_effective_date": row.source_effective_date,
+        "source_recorded_at": None,
+        "current_price": row.current_price,
+        "currency": RetailPriceSnapshot.Currency.KRW,
+        "source_row_sha256": row.source_row_hash,
+        "source_contract_revision": KAMIS_SOURCE_CONTRACT_REVISION,
+    }
+    if any(
+        getattr(series, field_name) != expected for field_name, expected in expected_series.items()
+    ) or any(
+        getattr(snapshot, field_name) != expected
+        for field_name, expected in expected_snapshot.items()
+    ):
+        raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+
+    if set(references) != {period.value for period in _REFERENCE_PERIODS}:
+        raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+    for parsed_reference in row.references:
+        stored_reference = references[parsed_reference.period.value]
+        expected_reference: dict[str, object] = {
+            "value_status": parsed_reference.value_status.value,
+            "value": parsed_reference.value,
+            "unavailable_reason": parsed_reference.unavailable_reason,
+            "reference_date_status": parsed_reference.reference_date_status.value,
+            "source_reference_date": parsed_reference.source_reference_date,
+        }
+        if any(
+            getattr(stored_reference, field_name) != expected
+            for field_name, expected in expected_reference.items()
+        ):
+            raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+        change = changes_by_reference.get(stored_reference.id)
+        if change is None:
+            raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
+        expected_change = PriceChangeFact._expected_fields(stored_reference)
+        if any(
+            getattr(change, field_name) != expected
+            for field_name, expected in expected_change.items()
+        ):
+            raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
diff --git a/grocery/tests/test_kamis_generation.py b/grocery/tests/test_kamis_generation.py
new file mode 100644
index 0000000..424af6d
--- /dev/null
+++ b/grocery/tests/test_kamis_generation.py
@@ -0,0 +1,323 @@
+"""PostgreSQL integration tests for deterministic KAMIS generation persistence."""
+
+from __future__ import annotations
+
+import uuid
+from collections.abc import Mapping
+from copy import deepcopy
+from dataclasses import replace
+from decimal import Decimal
+from unittest.mock import patch
+
+import pytest
+from django.core.exceptions import ValidationError
+
+from grocery.models import (
+    ParseRun,
+    PriceChangeFact,
+    PriceSeriesKey,
+    ReferencePrice,
+    RetailPriceSnapshot,
+    persist_reference_price_facts,
+)
+from grocery.source.generation import (
+    KAMIS_PARSE_CONFIGURATION_SHA256,
+    KAMIS_PARSER_REVISION,
+    KAMIS_SOURCE_CONTRACT_REVISION,
+    ParseGenerationError,
+    ParseGenerationErrorCode,
+    ParseGenerationFailureCode,
+    complete_kamis_parse_generation,
+    fail_kamis_parse_run,
+    start_or_get_kamis_parse_run,
+)
+from grocery.source.kamis import (
+    KAMIS_ALLOWED_CATEGORIES,
+    ParsedRecentPriceResult,
+    parse_recent_price_rows,
+)
+from grocery.source.registry import INITIAL_RETAIL_IDENTITY_REGISTRY
+from grocery.tests.test_artifact_parse_models import create_artifact
+
+pytestmark = pytest.mark.django_db
+
+
+def _synthetic_contract_row(
+    series_key: tuple[str, str, str, str],
+    *,
+    ordinal: int,
+    missing_period: str | None = None,
+) -> dict[str, object]:
+    category_code, item_code, variety_code, grade_code = series_key
+    registry = INITIAL_RETAIL_IDENTITY_REGISTRY
+    raw_unit, raw_unit_size = registry.units[series_key]
+    current = 8_000 + ordinal * 100
+    values: dict[str, object] = {
+        "ctgry_cd": category_code,
+        "ctgry_nm": KAMIS_ALLOWED_CATEGORIES[category_code],
+        "dd1_bfr_cnvs_prc": str(current - 50),
+        "dd1_bfr_prc": str(current - 50),
+        "exmn_dd_cnvs_prc": str(current),
+        "exmn_dd_prc": str(current),
+        "exmn_ymd": "20260829",
+        "grd_cd": grade_code,
+        "grd_nm": registry.grade_names[series_key],
+        "item_cd": item_code,
+        "item_nm": registry.item_names[(category_code, item_code)],
+        "mm1_bfr_cnvs_prc": str(current - 200),
+        "mm1_bfr_prc": str(current - 200),
+        "se_cd": "01",
+        "se_nm": "소매",
+        "unit": raw_unit,
+        "unit_sz": raw_unit_size,
+        "vrty_cd": variety_code,
+        "vrty_nm": registry.variety_names[(category_code, item_code, variety_code)],
+        "ww1_bfr_cnvs_prc": str(current - 100),
+        "ww1_bfr_prc": str(current - 100),
+        "yy1_bfr_cnvs_prc": str(current),
+        "yy1_bfr_prc": str(current),
+    }
+    if missing_period == "WEEK":
+        values["ww1_bfr_prc"] = None
+    elif missing_period == "MONTH":
+        values["mm1_bfr_prc"] = None
+    elif missing_period == "YEAR":
+        values["yy1_bfr_prc"] = None
+    return values
+
+
+def _parsed_result(*, missing_row_ordinal: int | None = None) -> ParsedRecentPriceResult:
+    rows = [
+        _synthetic_contract_row(
+            series_key,
+            ordinal=ordinal,
+            missing_period="MONTH" if ordinal == missing_row_ordinal else None,
+        )
+        for ordinal, series_key in enumerate(
+            sorted(INITIAL_RETAIL_IDENTITY_REGISTRY.units),
+            start=1,
+        )
+    ]
+    out_of_scope = deepcopy(rows[0])
+    out_of_scope["se_cd"] = "02"
+    out_of_scope["se_nm"] = "합성 중도매"
+    return parse_recent_price_rows(
+        [*reversed(rows), out_of_scope],
+        identity_registry=INITIAL_RETAIL_IDENTITY_REGISTRY,
+    )
+
+
+def _start_generation() -> ParseRun:
+    return start_or_get_kamis_parse_run(create_artifact().id).parse_run
+
+
+def test_positive_generation_persists_exact_typed_facts_and_replays_without_duplicates() -> None:
+    artifact = create_artifact()
+    first_start = start_or_get_kamis_parse_run(artifact.id)
+    repeated_start = start_or_get_kamis_parse_run(artifact.id)
+    result = _parsed_result()
+
+    completed = complete_kamis_parse_generation(first_start.parse_run.id, result)
+    replayed = complete_kamis_parse_generation(first_start.parse_run.id, result)
+
+    assert first_start.created is True
+    assert repeated_start.created is False
+    assert repeated_start.parse_run.id == first_start.parse_run.id
+    assert KAMIS_PARSER_REVISION == "kamis-recent-price-v1"
+    assert KAMIS_SOURCE_CONTRACT_REVISION == "data-go-15156063-recent-v1"
+    assert len(KAMIS_PARSE_CONFIGURATION_SHA256) == 64
+    assert completed.replayed is False
+    assert replayed.replayed is True
+    assert [snapshot.id for snapshot in replayed.snapshots] == [
+        snapshot.id for snapshot in completed.snapshots
+    ]
+
+    parse_run = ParseRun.objects.get(pk=first_start.parse_run.id)
+    assert parse_run.status == ParseRun.Status.VALIDATED
+    assert parse_run.parser_revision == KAMIS_PARSER_REVISION
+    assert parse_run.configuration_hash == KAMIS_PARSE_CONFIGURATION_SHA256
+    assert parse_run.result_hash == result.result_hash
+    assert parse_run.total_row_count == 11
+    assert parse_run.accepted_row_count == 10
+    assert parse_run.out_of_scope_row_count == 1
+    assert parse_run.missing_reference_row_count == 0
+    assert parse_run.quarantined_row_count == 0
+    assert PriceSeriesKey.objects.count() == 10
+    assert RetailPriceSnapshot.objects.count() == 10
+    assert ReferencePrice.objects.count() == 30
+    assert PriceChangeFact.objects.count() == 30
+    assert set(RetailPriceSnapshot.objects.values_list("source_contract_revision", flat=True)) == {
+        KAMIS_SOURCE_CONTRACT_REVISION
+    }
+    assert set(PriceSeriesKey.objects.values_list("identity_evidence_revision", flat=True)) == {
+        INITIAL_RETAIL_IDENTITY_REGISTRY.evidence.coverage_evidence_revision
+    }
+
+
+def test_missing_reference_counts_rows_once_and_persists_unavailable_fact() -> None:
+    parse_run = _start_generation()
+    result = _parsed_result(missing_row_ordinal=1)
+
+    complete_kamis_parse_generation(parse_run.id, result)
+
+    parse_run.refresh_from_db()
+    unavailable = ReferencePrice.objects.get(value_status=ReferencePrice.ValueStatus.UNAVAILABLE)
+    assert parse_run.missing_reference_row_count == 1
+    assert ReferencePrice.objects.count() == 30
+    assert unavailable.period == ReferencePrice.Period.MONTH
+    assert unavailable.value is None
+    assert unavailable.unavailable_reason == ReferencePrice.UnavailableReason.SOURCE_VALUE_MISSING
+    assert unavailable.reference_date_status == (
+        ReferencePrice.ReferenceDateStatus.SOURCE_REFERENCE_DATE_UNAVAILABLE
+    )
+    assert unavailable.source_reference_date is None
+    assert unavailable.change_fact.direction == PriceChangeFact.Direction.UNAVAILABLE
+    assert unavailable.change_fact.signed_difference is None
+    assert unavailable.change_fact.signed_percentage is None
+
+
+def test_count_drift_fails_before_any_domain_fact_and_can_be_safely_finalized() -> None:
+    parse_run = _start_generation()
+    valid = _parsed_result()
+    drifted = replace(valid, input_row_count=valid.input_row_count + 1)
+
+    with pytest.raises(ParseGenerationError) as caught:
+        complete_kamis_parse_generation(parse_run.id, drifted)
+
+    assert caught.value.code is ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED
+    assert str(caught.value) == "RESULT_RECONCILIATION_FAILED"
+    parse_run.refresh_from_db()
+    assert parse_run.status == ParseRun.Status.STARTED
+    assert not PriceSeriesKey.objects.exists()
+    assert not RetailPriceSnapshot.objects.exists()
+    assert not ReferencePrice.objects.exists()
+    assert not PriceChangeFact.objects.exists()
+
+    failed = fail_kamis_parse_run(
+        parse_run.id,
+        ParseGenerationFailureCode.RECONCILIATION_FAILED,
+    )
+    assert failed.status == ParseRun.Status.FAILED
+    assert failed.failure_code == "RECONCILIATION_FAILED"
+
+
+def test_reference_replay_conflict_fails_closed_without_partial_new_rows() -> None:
+    parse_run = _start_generation()
+    result = _parsed_result()
+    first_row = result.rows[0]
+    registry = INITIAL_RETAIL_IDENTITY_REGISTRY
+    series = PriceSeriesKey.get_or_validate(
+        product_class_code=first_row.product_class_code,
+        product_class_name=first_row.product_class_name,
+        category_code=first_row.category_code,
+        category_name=first_row.category_name,
+        item_code=first_row.item_code,
+        item_name=first_row.item_name,
+        variety_code=first_row.variety_code,
+        variety_name=first_row.variety_name,
+        grade_code=first_row.grade_code,
+        grade_name=first_row.grade_name,
+        raw_unit=first_row.raw_unit,
+        raw_unit_size=first_row.raw_unit_size,
+        coverage_identity=first_row.coverage_identity,
+        identity_evidence_revision=registry.evidence.coverage_evidence_revision,
+    )
+    snapshot = RetailPriceSnapshot(
+        parse_run=parse_run,
+        series=series,
+        source_effective_date=first_row.source_effective_date,
+        source_recorded_at=None,
+        current_price=first_row.current_price,
+        currency=RetailPriceSnapshot.Currency.KRW,
+        source_row_sha256=first_row.source_row_hash,
+        source_contract_revision=KAMIS_SOURCE_CONTRACT_REVISION,
+    )
+    RetailPriceSnapshot.objects.bulk_create([snapshot])
+    ReferencePrice.objects.create(
+        snapshot=snapshot,
+        period=ReferencePrice.Period.WEEK,
+        value_status=ReferencePrice.ValueStatus.AVAILABLE,
+        value=Decimal("1"),
+        unavailable_reason=None,
+    )
+
+    with pytest.raises(ParseGenerationError) as caught:
+        complete_kamis_parse_generation(parse_run.id, result)
+
+    assert caught.value.code is ParseGenerationErrorCode.PERSISTENCE_CONFLICT
+    parse_run.refresh_from_db()
+    assert parse_run.status == ParseRun.Status.STARTED
+    assert RetailPriceSnapshot.objects.count() == 1
+    assert ReferencePrice.objects.count() == 1
+    assert PriceChangeFact.objects.count() == 0
+    assert PriceSeriesKey.objects.count() == 1
+
+
+def test_mid_generation_failure_rolls_back_run_and_every_new_fact() -> None:
+    parse_run = _start_generation()
+    result = _parsed_result()
+    call_count = 0
+
+    def persist_then_fail(
+        *,
+        snapshot_id: uuid.UUID,
+        reference_values: Mapping[str, Decimal | None],
+    ) -> tuple[PriceChangeFact, ...]:
+        nonlocal call_count
+        facts = persist_reference_price_facts(
+            snapshot_id=snapshot_id,
+            reference_values=reference_values,
+        )
+        call_count += 1
+        if call_count == 1:
+            raise ValidationError("test-only injected failure")
+        return facts
+
+    with (
+        patch(
+            "grocery.source.generation.persist_reference_price_facts",
+            side_effect=persist_then_fail,
+        ),
+        pytest.raises(ParseGenerationError) as caught,
+    ):
+        complete_kamis_parse_generation(parse_run.id, result)
+
+    assert caught.value.code is ParseGenerationErrorCode.PERSISTENCE_CONFLICT
+    assert str(caught.value) == "PERSISTENCE_CONFLICT"
+    parse_run.refresh_from_db()
+    assert parse_run.status == ParseRun.Status.STARTED
+    assert parse_run.result_hash == ""
+    assert not PriceSeriesKey.objects.exists()
+    assert not RetailPriceSnapshot.objects.exists()
+    assert not ReferencePrice.objects.exists()
+    assert not PriceChangeFact.objects.exists()
+
+
+def test_same_artifact_changed_result_fails_as_nondeterministic_replay() -> None:
+    parse_run = _start_generation()
+    result = _parsed_result()
+    completed = complete_kamis_parse_generation(parse_run.id, result)
+    changed = replace(result, result_hash="f" * 64)
+
+    with pytest.raises(ParseGenerationError) as caught:
+        complete_kamis_parse_generation(parse_run.id, changed)
+
+    assert caught.value.code is ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY
+    assert str(caught.value) == "NONDETERMINISTIC_REPLAY"
+    assert RetailPriceSnapshot.objects.count() == len(completed.snapshots) == 10
+    assert ReferencePrice.objects.count() == 30
+    assert PriceChangeFact.objects.count() == 30
+
+
+def test_failure_boundary_rejects_arbitrary_codes_without_echoing_them() -> None:
+    parse_run = _start_generation()
+    unsafe = "NOT_ALLOWED_private-input"
+
+    with pytest.raises(ParseGenerationError) as caught:
+        fail_kamis_parse_run(parse_run.id, unsafe)  # type: ignore[arg-type]
+
+    assert caught.value.code is ParseGenerationErrorCode.FAILURE_CODE_INVALID
+    assert str(caught.value) == "FAILURE_CODE_INVALID"
+    assert unsafe not in str(caught.value)
+    parse_run.refresh_from_db()
+    assert parse_run.status == ParseRun.Status.STARTED


