## `feat(history): ingest bounded source collections`

diff --git a/grocery/historical_ingestion_workflow.py b/grocery/historical_ingestion_workflow.py
new file mode 100644
index 0000000..88f1f80
--- /dev/null
+++ b/grocery/historical_ingestion_workflow.py
@@ -0,0 +1,156 @@
+"""Bounded source-to-candidate workflow shared by three operator commands."""
+
+from __future__ import annotations
+
+import uuid
+from dataclasses import dataclass
+from typing import Protocol
+
+from django.db.models import Max
+
+from grocery.historical_collection_models import HistoricalSourceCollection
+from grocery.historical_collection_plans import plan_historical_collection
+from grocery.historical_collections import complete_historical_collection
+from grocery.historical_daily_generation import persist_market_part, persist_regional_part
+from grocery.historical_generation import persist_monthly_part
+from grocery.historical_registry import load_historical_dimension_registry
+from grocery.models import FetchAttempt
+from grocery.source.client import DEFAULT_PAGE_SIZE, KamisFetchResult, KamisTransportError
+from grocery.source.historical_client import prepare_historical_request
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.source.historical_persistence import start_historical_fetch
+from grocery.source.kamis import KamisParseError
+from grocery.source.market_history import parse_market_price_rows
+from grocery.source.monthly_history import parse_monthly_price_rows
+from grocery.source.persistence import complete_kamis_fetch, fail_kamis_fetch
+from grocery.source.regional_history import parse_regional_price_rows
+
+
+class HistoricalSourceClient(Protocol):
+    def fetch_historical_prices(
+        self,
+        dataset: HistoricalDataset,
+        service_key: str,
+        *,
+        query: HistoricalPriceQuery,
+        page_size: int,
+    ) -> KamisFetchResult: ...
+
+
+class HistoricalIngestionError(RuntimeError):
+    SAFE_CODES = {
+        "FETCH_FAILED",
+        "FETCH_FINALIZATION_FAILED",
+        "FETCH_PERSISTENCE_FAILED",
+        "PARSE_FAILED",
+        "PART_PERSISTENCE_FAILED",
+        "COLLECTION_COMPLETION_FAILED",
+    }
+
+    def __init__(self, code: str) -> None:
+        safe_code = code if code in self.SAFE_CODES else "PART_PERSISTENCE_FAILED"
+        self.code = safe_code
+        super().__init__(safe_code)
+
+
+@dataclass(frozen=True, slots=True)
+class HistoricalIngestionOutcome:
+    collection: HistoricalSourceCollection
+    partition_count: int
+    accepted_row_count: int
+
+
+def ingest_historical_collection(
+    *,
+    collection_id: uuid.UUID,
+    source_configuration_id: uuid.UUID,
+    dataset: HistoricalDataset,
+    queries: tuple[HistoricalPriceQuery, ...],
+    code_manifest_sha256: str,
+    service_key: str,
+    client: HistoricalSourceClient,
+    page_size: int = DEFAULT_PAGE_SIZE,
+) -> HistoricalIngestionOutcome:
+    prepared_requests = tuple(prepare_historical_request(dataset, query) for query in queries)
+    collection = plan_historical_collection(
+        collection_id=collection_id,
+        source_configuration_id=source_configuration_id,
+        prepared_requests=prepared_requests,
+        code_manifest_sha256=code_manifest_sha256,
+    )
+    registry = load_historical_dimension_registry(code_manifest_sha256)
+    next_attempt = (
+        FetchAttempt.objects.filter(acquisition_run_id=collection.id).aggregate(
+            maximum=Max("attempt_ordinal")
+        )["maximum"]
+        or 0
+    ) + 1
+    accepted = 0
+    for partition_ordinal, (query, prepared) in enumerate(
+        zip(queries, prepared_requests, strict=True), start=1
+    ):
+        attempt = start_historical_fetch(
+            source_configuration_id,
+            prepared_request=prepared,
+            acquisition_run_id=collection.id,
+            attempt_ordinal=next_attempt + partition_ordinal - 1,
+        )
+        try:
+            fetched = client.fetch_historical_prices(
+                dataset,
+                service_key,
+                query=query,
+                page_size=page_size,
+            )
+        except KamisTransportError as error:
+            try:
+                fail_kamis_fetch(attempt.id, error)
+            except Exception:
+                raise HistoricalIngestionError("FETCH_FINALIZATION_FAILED") from None
+            raise HistoricalIngestionError("FETCH_FAILED") from None
+        try:
+            completed_fetch = complete_kamis_fetch(attempt.id, fetched)
+        except Exception:
+            raise HistoricalIngestionError("FETCH_PERSISTENCE_FAILED") from None
+        try:
+            if dataset == HistoricalDataset.MONTHLY:
+                parsed = parse_monthly_price_rows(fetched.rows, registry=registry)
+                persist_monthly_part(
+                    collection_id=collection.id,
+                    ordinal=partition_ordinal,
+                    artifact_id=completed_fetch.artifact.id,
+                    prepared_request=prepared,
+                    parsed=parsed,
+                    code_manifest_sha256=code_manifest_sha256,
+                )
+            elif dataset == HistoricalDataset.REGIONAL:
+                parsed = parse_regional_price_rows(fetched.rows, registry=registry)
+                persist_regional_part(
+                    collection_id=collection.id,
+                    ordinal=partition_ordinal,
+                    artifact_id=completed_fetch.artifact.id,
+                    prepared_request=prepared,
+                    parsed=parsed,
+                    code_manifest_sha256=code_manifest_sha256,
+                )
+            else:
+                parsed = parse_market_price_rows(fetched.rows, registry=registry)
+                persist_market_part(
+                    collection_id=collection.id,
+                    ordinal=partition_ordinal,
+                    artifact_id=completed_fetch.artifact.id,
+                    prepared_request=prepared,
+                    parsed=parsed,
+                    code_manifest_sha256=code_manifest_sha256,
+                )
+        except KamisParseError:
+            raise HistoricalIngestionError("PARSE_FAILED") from None
+        except Exception:
+            raise HistoricalIngestionError("PART_PERSISTENCE_FAILED") from None
+        accepted += len(parsed.rows)
+        del fetched, parsed
+    try:
+        completed = complete_historical_collection(collection.id)
+    except Exception:
+        raise HistoricalIngestionError("COLLECTION_COMPLETION_FAILED") from None
+    return HistoricalIngestionOutcome(completed, len(prepared_requests), accepted)
diff --git a/grocery/tests/test_historical_ingestion_workflow.py b/grocery/tests/test_historical_ingestion_workflow.py
new file mode 100644
index 0000000..585026a
--- /dev/null
+++ b/grocery/tests/test_historical_ingestion_workflow.py
@@ -0,0 +1,78 @@
+import hashlib
+import json
+import uuid
+
+from grocery.historical_ingestion_workflow import ingest_historical_collection
+from grocery.historical_publication_models import HistoricalRetailPublicationRevision
+from grocery.historical_review_models import HistoricalCollectionReviewDecision
+from grocery.models import SourceConfiguration
+from grocery.source.client import KamisFetchResult, PageReceipt
+from grocery.source.historical_client import prepare_historical_request
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+from grocery.tests.historical_bundle_factory import create_reviewed_historical_bundle
+from grocery.tests.historical_fixtures import monthly_row
+from grocery.tests.test_acquisition_models import create_source_configuration
+
+
+class _SyntheticClient:
+    def __init__(self, row: dict[str, str]) -> None:
+        self.row = row
+        self.calls = 0
+
+    def fetch_historical_prices(
+        self,
+        dataset: HistoricalDataset,
+        service_key: str,
+        *,
+        query: HistoricalPriceQuery,
+        page_size: int,
+    ) -> KamisFetchResult:
+        del service_key
+        self.calls += 1
+        prepared = prepare_historical_request(dataset, query)
+        body_hash = hashlib.sha256(b"synthetic-page").hexdigest()
+        manifest = hashlib.sha256(
+            json.dumps([body_hash], separators=(",", ":")).encode("ascii")
+        ).hexdigest()
+        return KamisFetchResult(
+            rows=(self.row,),
+            page_receipts=(
+                PageReceipt(1, 1, 1, page_size, 1, 1, 200, "0", 10, body_hash),
+            ),
+            ordered_manifest_sha256=manifest,
+            call_count=1,
+            request_scope_sha256=prepared.scope_sha256,
+        )
+
+
+def test_workflow_uses_synthetic_transport_and_stops_before_review(db: None) -> None:
+    bundle = create_reviewed_historical_bundle()
+    source = create_source_configuration(
+        dataset_id="15156060",
+        publication_mode=SourceConfiguration.PublicationMode.HISTORICAL_MONTHLY,
+    )
+    row = monthly_row()
+    row.update(exmn_ym="202512", sgg_cd=bundle.region.region_code, sgg_nm=bundle.region.region_name)
+    client = _SyntheticClient(row)
+    query = HistoricalPriceQuery(
+        start="202512", end="202512", category_code="200", item_code="212"
+    )
+    review_count = HistoricalCollectionReviewDecision.objects.count()
+
+    outcome = ingest_historical_collection(
+        collection_id=uuid.uuid4(),
+        source_configuration_id=source.id,
+        dataset=HistoricalDataset.MONTHLY,
+        queries=(query,),
+        code_manifest_sha256="a" * 64,
+        service_key="synthetic-only",
+        client=client,
+    )
+
+    assert (outcome.collection.state, outcome.accepted_row_count, client.calls) == (
+        "VALIDATED",
+        1,
+        1,
+    )
+    assert HistoricalCollectionReviewDecision.objects.count() == review_count
+    assert HistoricalRetailPublicationRevision.objects.count() == 0


## `feat(history): expose three bounded ingest commands`

diff --git a/grocery/management/commands/ingest_kamis_market_daily.py b/grocery/management/commands/ingest_kamis_market_daily.py
new file mode 100644
index 0000000..223bafe
--- /dev/null
+++ b/grocery/management/commands/ingest_kamis_market_daily.py
@@ -0,0 +1,16 @@
+from grocery.management.historical_ingestion import (
+    HistoricalIngestionCommand,
+    region_scopes,
+)
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+
+
+class Command(HistoricalIngestionCommand):
+    help = "Fetch one reviewed-scope KAMIS market daily candidate collection."
+    dataset = HistoricalDataset.MARKET
+
+    def build_queries(self, options: dict[str, object]) -> tuple[HistoricalPriceQuery, ...]:
+        return tuple(
+            self.query(options, region_code=region)
+            for region in region_scopes(options, required=False)
+        )
diff --git a/grocery/management/commands/ingest_kamis_monthly.py b/grocery/management/commands/ingest_kamis_monthly.py
new file mode 100644
index 0000000..cd5521b
--- /dev/null
+++ b/grocery/management/commands/ingest_kamis_monthly.py
@@ -0,0 +1,16 @@
+from grocery.management.historical_ingestion import (
+    HistoricalIngestionCommand,
+    region_scopes,
+)
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+
+
+class Command(HistoricalIngestionCommand):
+    help = "Fetch one reviewed-scope KAMIS monthly historical candidate collection."
+    dataset = HistoricalDataset.MONTHLY
+
+    def build_queries(self, options: dict[str, object]) -> tuple[HistoricalPriceQuery, ...]:
+        return tuple(
+            self.query(options, region_code=region)
+            for region in region_scopes(options, required=False)
+        )
diff --git a/grocery/management/commands/ingest_kamis_regional_daily.py b/grocery/management/commands/ingest_kamis_regional_daily.py
new file mode 100644
index 0000000..77c22e3
--- /dev/null
+++ b/grocery/management/commands/ingest_kamis_regional_daily.py
@@ -0,0 +1,16 @@
+from grocery.management.historical_ingestion import (
+    HistoricalIngestionCommand,
+    region_scopes,
+)
+from grocery.source.historical_contract import HistoricalDataset, HistoricalPriceQuery
+
+
+class Command(HistoricalIngestionCommand):
+    help = "Fetch one reviewed-scope KAMIS regional daily candidate collection."
+    dataset = HistoricalDataset.REGIONAL
+
+    def build_queries(self, options: dict[str, object]) -> tuple[HistoricalPriceQuery, ...]:
+        return tuple(
+            self.query(options, region_code=region)
+            for region in region_scopes(options, required=True)
+        )
diff --git a/grocery/management/historical_ingestion.py b/grocery/management/historical_ingestion.py
new file mode 100644
index 0000000..c76762f
--- /dev/null
+++ b/grocery/management/historical_ingestion.py
@@ -0,0 +1,148 @@
+"""Shared fail-closed command shell for three historical source families."""
+
+from __future__ import annotations
+
+import re
+import uuid
+from abc import abstractmethod
+
+from django.core.exceptions import ValidationError
+from django.core.management.base import BaseCommand, CommandError, CommandParser
+
+from grocery.historical_ingestion_workflow import (
+    HistoricalIngestionError,
+    ingest_historical_collection,
+)
+from grocery.historical_registry import load_historical_dimension_registry
+from grocery.source.client import DEFAULT_PAGE_SIZE, MAX_PAGE_SIZE, KamisHttpClient
+from grocery.source.historical_client import prepare_historical_request
+from grocery.source.historical_contract import (
+    HistoricalContractError,
+    HistoricalDataset,
+    HistoricalPriceQuery,
+)
+from grocery.source.secrets import SecretLoadError, load_kamis_api_key
+
+_SHA256 = re.compile(r"[0-9a-f]{64}\Z")
+
+
+def _uuid(value: object) -> uuid.UUID:
+    try:
+        return uuid.UUID(str(value))
+    except (TypeError, ValueError, AttributeError):  # fmt: skip
+        raise CommandError("code=HISTORICAL_INGEST_UUID_INVALID") from None
+
+
+def _sha256(value: object) -> str:
+    if not isinstance(value, str) or _SHA256.fullmatch(value) is None:
+        raise CommandError("code=HISTORICAL_INGEST_MANIFEST_INVALID")
+    return value
+
+
+def _page_size(value: object) -> int:
+    try:
+        parsed = int(str(value))
+    except (TypeError, ValueError):  # fmt: skip
+        raise CommandError("code=HISTORICAL_INGEST_PAGE_SIZE_INVALID") from None
+    if parsed < 1 or parsed > MAX_PAGE_SIZE:
+        raise CommandError("code=HISTORICAL_INGEST_PAGE_SIZE_INVALID")
+    return parsed
+
+
+def _optional_text(value: object) -> str | None:
+    return value if isinstance(value, str) and value else None
+
+
+class HistoricalIngestionCommand(BaseCommand):
+    dataset: HistoricalDataset
+
+    def add_arguments(self, parser: CommandParser) -> None:
+        parser.add_argument("--collection-id", required=True)
+        parser.add_argument("--source-configuration-id", required=True)
+        parser.add_argument("--code-manifest-sha256", required=True)
+        parser.add_argument("--start", required=True)
+        parser.add_argument("--end", required=True)
+        parser.add_argument("--category-code", required=True)
+        parser.add_argument("--item-code")
+        parser.add_argument("--variety-code")
+        parser.add_argument("--grade-code")
+        parser.add_argument("--region-code", action="append")
+        parser.add_argument("--page-size", default=DEFAULT_PAGE_SIZE)
+
+    @abstractmethod
+    def build_queries(self, options: dict[str, object]) -> tuple[HistoricalPriceQuery, ...]:
+        raise NotImplementedError
+
+    @staticmethod
+    def query(options: dict[str, object], *, region_code: str | None) -> HistoricalPriceQuery:
+        return HistoricalPriceQuery(
+            start=str(options["start"]),
+            end=str(options["end"]),
+            category_code=str(options["category_code"]),
+            item_code=_optional_text(options.get("item_code")),
+            variety_code=_optional_text(options.get("variety_code")),
+            grade_code=_optional_text(options.get("grade_code")),
+            region_code=region_code,
+        )
+
+    def handle(self, *args: object, **options: object) -> None:
+        del args
+        collection_id = _uuid(options.get("collection_id"))
+        source_id = _uuid(options.get("source_configuration_id"))
+        manifest = _sha256(options.get("code_manifest_sha256"))
+        page_size = _page_size(options.get("page_size"))
+        try:
+            queries = self.build_queries(options)
+            if not queries or len(queries) > 100:
+                raise HistoricalContractError("invalid_historical_partition_count")
+            for query in queries:
+                prepare_historical_request(self.dataset, query)
+            load_historical_dimension_registry(manifest)
+        except (HistoricalContractError, ValidationError):  # fmt: skip
+            raise CommandError("code=HISTORICAL_INGEST_CONTRACT_INVALID") from None
+
+        try:
+            secret = load_kamis_api_key()
+        except SecretLoadError:
+            raise CommandError("code=HISTORICAL_INGEST_SECRET_UNAVAILABLE") from None
+        except Exception:
+            raise CommandError("code=HISTORICAL_INGEST_SECRET_UNAVAILABLE") from None
+        try:
+            outcome = ingest_historical_collection(
+                collection_id=collection_id,
+                source_configuration_id=source_id,
+                dataset=self.dataset,
+                queries=queries,
+                code_manifest_sha256=manifest,
+                service_key=secret.reveal(),
+                client=KamisHttpClient(),
+                page_size=page_size,
+            )
+        except HistoricalIngestionError as error:
+            raise CommandError(f"code=HISTORICAL_INGEST_{error.code}") from None
+        except Exception:
+            raise CommandError("code=HISTORICAL_INGEST_FAILED") from None
+        finally:
+            del secret
+        self.stdout.write(
+            " ".join(
+                (
+                    "status=VALIDATED",
+                    f"collection_id={outcome.collection.id}",
+                    f"partitions={outcome.partition_count}",
+                    f"rows={outcome.accepted_row_count}",
+                )
+            )
+        )
+
+
+def region_scopes(options: dict[str, object], *, required: bool) -> tuple[str | None, ...]:
+    raw = options.get("region_code")
+    regions = (
+        tuple(value for value in raw if isinstance(value, str))
+        if isinstance(raw, list)
+        else ()
+    )
+    if required and not regions:
+        raise HistoricalContractError("missing_historical_region")
+    return regions or (None,)
diff --git a/grocery/tests/test_historical_ingestion_commands.py b/grocery/tests/test_historical_ingestion_commands.py
new file mode 100644
index 0000000..054fd8f
--- /dev/null
+++ b/grocery/tests/test_historical_ingestion_commands.py
@@ -0,0 +1,84 @@
+import uuid
+from io import StringIO
+from types import SimpleNamespace
+from unittest.mock import Mock, patch
+
+import pytest
+from django.core.management import call_command
+
+from grocery.source.historical_contract import HistoricalDataset
+
+
+@pytest.mark.parametrize(
+    ("command_name", "dataset", "start", "end", "regions", "partition_count"),
+    (
+        (
+            "ingest_kamis_monthly",
+            HistoricalDataset.MONTHLY,
+            "202501",
+            "202512",
+            ["1101", "2100"],
+            2,
+        ),
+        (
+            "ingest_kamis_regional_daily",
+            HistoricalDataset.REGIONAL,
+            "20250801",
+            "20250831",
+            ["1101"],
+            1,
+        ),
+        (
+            "ingest_kamis_market_daily",
+            HistoricalDataset.MARKET,
+            "20250801",
+            "20250831",
+            None,
+            1,
+        ),
+    ),
+)
+def test_historical_commands_delegate_only_bounded_validated_queries(
+    command_name: str,
+    dataset: HistoricalDataset,
+    start: str,
+    end: str,
+    regions: list[str] | None,
+    partition_count: int,
+) -> None:
+    collection_id = uuid.uuid4()
+    outcome = SimpleNamespace(
+        collection=SimpleNamespace(id=collection_id),
+        partition_count=partition_count,
+        accepted_row_count=7,
+    )
+    secret = Mock()
+    secret.reveal.return_value = "synthetic-command-key"
+    stdout = StringIO()
+    with (
+        patch("grocery.management.historical_ingestion.load_historical_dimension_registry"),
+        patch(
+            "grocery.management.historical_ingestion.load_kamis_api_key",
+            return_value=secret,
+        ),
+        patch(
+            "grocery.management.historical_ingestion.ingest_historical_collection",
+            return_value=outcome,
+        ) as ingest,
+    ):
+        call_command(
+            command_name,
+            collection_id=str(collection_id),
+            source_configuration_id=str(uuid.uuid4()),
+            code_manifest_sha256="a" * 64,
+            start=start,
+            end=end,
+            category_code="200",
+            region_code=regions,
+            stdout=stdout,
+        )
+
+    delegated = ingest.call_args.kwargs
+    assert delegated["dataset"] == dataset
+    assert len(delegated["queries"]) == partition_count
+    assert stdout.getvalue().startswith("status=VALIDATED collection_id=")
