## `build(evidence): verify Flyway release history`

diff --git a/scripts/cold_gate/migration_evidence.py b/scripts/cold_gate/migration_evidence.py
new file mode 100644
index 0000000..2bd094a
--- /dev/null
+++ b/scripts/cold_gate/migration_evidence.py
@@ -0,0 +1,97 @@
+from __future__ import annotations
+
+import re
+import zlib
+from pathlib import Path
+
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.database import PostgresClient
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import MIGRATION_VERSIONS
+from scripts.cold_gate.owned_path import require_directory, require_regular_file
+
+
+MIGRATION_NAME = re.compile(r"^V([1-9][0-9]*)__([A-Za-z0-9_]+)\.sql$")
+HISTORY_QUERY = (
+    "SELECT installed_rank::text AS installed_rank, version, script, "
+    "checksum::text AS checksum, success::text AS success "
+    "FROM flyway_schema_history ORDER BY installed_rank"
+)
+
+
+def flyway_checksum(path: Path) -> int:
+    require_regular_file(path)
+    try:
+        text = path.read_text(encoding="utf-8-sig")
+    except UnicodeDecodeError as error:
+        raise RuntimeError(f"{path.name} is not UTF-8") from error
+    checksum = 0
+    for line in text.splitlines():
+        checksum = zlib.crc32(line.encode("utf-8"), checksum)
+    return checksum - (1 << 32) if checksum >= (1 << 31) else checksum
+
+
+class MigrationEvidence:
+    def __init__(
+        self,
+        artifacts: ReleaseArtifacts,
+        database: PostgresClient,
+        store: EvidenceStore,
+    ) -> None:
+        if artifacts.sources != store.context.runtime / "sources":
+            raise RuntimeError("migration evidence ownership mismatch")
+        self.artifacts = artifacts
+        self.database = database
+        self.store = store
+
+    def capture(self) -> None:
+        self.store.context.require_owned()
+        evidence = ["database\tinstalled_rank\tversion\tscript\tchecksum\tsuccess"]
+        for database, versions in MIGRATION_VERSIONS.items():
+            expected = self._source_history(database, versions)
+            observed = self.database.query(database, HISTORY_QUERY)
+            if observed != expected:
+                raise RuntimeError(f"{database} Flyway history drifted")
+            for row in observed:
+                evidence.append(
+                    "\t".join(
+                        [
+                            database,
+                            row["installed_rank"],
+                            row["version"],
+                            row["script"],
+                            row["checksum"],
+                            row["success"],
+                        ]
+                    )
+                )
+        if len(evidence) != 26:
+            raise RuntimeError("release migration inventory is not exactly 25 rows")
+        self.store.write("migrations.tsv", "\n".join(evidence) + "\n")
+
+    def _source_history(
+        self, database: str, versions: tuple[str, ...]
+    ) -> list[dict[str, str]]:
+        directory = (
+            self.artifacts.sources / database / "src/main/resources/db/migration"
+        )
+        require_directory(directory)
+        files: dict[str, Path] = {}
+        for path in directory.iterdir():
+            require_regular_file(path)
+            match = MIGRATION_NAME.fullmatch(path.name)
+            if match is None or match.group(1) in files:
+                raise RuntimeError(f"{database} migration source inventory is invalid")
+            files[match.group(1)] = path
+        if set(files) != set(versions):
+            raise RuntimeError(f"{database} migration source versions drifted")
+        return [
+            {
+                "installed_rank": str(rank),
+                "version": version,
+                "script": files[version].name,
+                "checksum": str(flyway_checksum(files[version])),
+                "success": "true",
+            }
+            for rank, version in enumerate(versions, 1)
+        ]


## `test(evidence): reject migration history drift`

diff --git a/tests/test_migration_evidence.py b/tests/test_migration_evidence.py
new file mode 100644
index 0000000..447b53e
--- /dev/null
+++ b/tests/test_migration_evidence.py
@@ -0,0 +1,98 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.build import ReleaseArtifacts
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.inventory import MIGRATION_VERSIONS
+from scripts.cold_gate.migration_evidence import HISTORY_QUERY, MigrationEvidence, flyway_checksum
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+class FakeDatabase:
+    def __init__(self, histories):
+        self.histories = histories
+        self.calls = []
+
+    def query(self, database, statement):
+        self.calls.append((database, statement))
+        return [row.copy() for row in self.histories[database]]
+
+
+class MigrationEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path):
+        context = ColdGateContext.create(root, SHA, "00000001")
+        sources = context.runtime / "sources"
+        sources.mkdir()
+        histories = {}
+        for database, versions in MIGRATION_VERSIONS.items():
+            migration_dir = sources / database / "src/main/resources/db/migration"
+            migration_dir.mkdir(parents=True)
+            histories[database] = []
+            for rank, version in enumerate(versions, 1):
+                path = migration_dir / f"V{version}__{database}_{version}.sql"
+                path.write_text(f"CREATE TABLE {database}_{version}(id INT);\n")
+                histories[database].append(
+                    {
+                        "installed_rank": str(rank),
+                        "version": version,
+                        "script": path.name,
+                        "checksum": str(flyway_checksum(path)),
+                        "success": "true",
+                    }
+                )
+        artifacts = ReleaseArtifacts(sources, context.runtime / "m2", context.runtime / "jars",
+                                     context.runtime / "fixture.jar")
+        store = EvidenceStore(context, EvidenceRedactor(["migration-secret-value"]))
+        return context, artifacts, store, histories
+
+    def test_records_exact_source_backed_flyway_history(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, artifacts, store, histories = self.fixture(
+                pathlib.Path(temporary).resolve()
+            )
+            database = FakeDatabase(histories)
+
+            MigrationEvidence(artifacts, database, store).capture()
+
+            lines = (context.evidence / "migrations.tsv").read_text().splitlines()
+            self.assertEqual(
+                lines[0], "database\tinstalled_rank\tversion\tscript\tchecksum\tsuccess"
+            )
+            self.assertEqual(len(lines), 26)
+            self.assertEqual([call[0] for call in database.calls], list(MIGRATION_VERSIONS))
+            self.assertTrue(all(call[1] == HISTORY_QUERY for call in database.calls))
+            self.assertEqual(sum(line.endswith("\ttrue") for line in lines[1:]), 25)
+
+    def test_rejects_database_or_source_inventory_drift_without_evidence(self) -> None:
+        for drift in ("checksum", "source"):
+            with self.subTest(drift=drift), tempfile.TemporaryDirectory() as temporary:
+                context, artifacts, store, histories = self.fixture(
+                    pathlib.Path(temporary).resolve()
+                )
+                if drift == "checksum":
+                    histories["settlement"][0]["checksum"] = "0"
+                else:
+                    directory = artifacts.sources / "wallet/src/main/resources/db/migration"
+                    (directory / "V5__unexpected.sql").write_text("SELECT 1;\n")
+
+                with self.assertRaisesRegex(RuntimeError, "drifted|versions"):
+                    MigrationEvidence(artifacts, FakeDatabase(histories), store).capture()
+                self.assertEqual(list(context.evidence.iterdir()), [])
+
+    def test_checksum_matches_flyway_line_and_signed_integer_rules(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            root = pathlib.Path(temporary).resolve()
+            unix = root / "unix.sql"
+            windows = root / "windows.sql"
+            unix.write_bytes(b"a\n\n")
+            windows.write_bytes(b"\xef\xbb\xbfa\r\n\r\n")
+
+            self.assertEqual(flyway_checksum(unix), -390611389)
+            self.assertEqual(flyway_checksum(windows), -390611389)
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): verify readiness and integrity`

diff --git a/scripts/cold_gate/readiness_evidence.py b/scripts/cold_gate/readiness_evidence.py
new file mode 100644
index 0000000..d6a3f5c
--- /dev/null
+++ b/scripts/cold_gate/readiness_evidence.py
@@ -0,0 +1,93 @@
+from __future__ import annotations
+
+import decimal
+import re
+from collections.abc import Callable
+
+from scripts.cold_gate.compose import ComposeProject
+from scripts.cold_gate.container_http import ContainerHttpClient
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.polling import poll_until
+
+
+READINESS_ENDPOINTS = (
+    ("wallet", "/actuator/health"),
+    ("risk", "/actuator/health/readiness"),
+    ("odds", "/actuator/health/readiness"),
+    ("betting", "/actuator/health"),
+    ("gateway", "/actuator/health/readiness"),
+    ("settlement", "/actuator/health/readiness"),
+    ("admin", "/actuator/health/readiness"),
+)
+INTEGRITY_METRICS = (
+    "wallet_integrity_total_drift",
+    "wallet_integrity_scan_failed",
+    "wallet_integrity_last_checked_epoch_seconds",
+)
+NUMBER = r"[-+]?(?:[0-9]+(?:\.[0-9]+)?|\.[0-9]+)(?:[eE][-+]?[0-9]+)?"
+ClientFactory = Callable[[ComposeProject, str], ContainerHttpClient]
+
+
+class ReadinessEvidence:
+    def __init__(
+        self,
+        compose: ComposeProject,
+        store: EvidenceStore,
+        client_factory: ClientFactory = ContainerHttpClient,
+    ) -> None:
+        if compose.context is not store.context:
+            raise RuntimeError("readiness evidence ownership mismatch")
+        self.compose = compose
+        self.store = store
+        self.client_factory = client_factory
+
+    def capture(self) -> None:
+        rows = ["kind\tservice\tcheck\tvalue"]
+        for service, endpoint in READINESS_ENDPOINTS:
+            client = self.client_factory(self.compose, service)
+            response = client.request("GET", endpoint).require_status(200)
+            payload = response.json()
+            if not isinstance(payload, dict) or payload.get("status") != "UP":
+                raise RuntimeError(f"{service} readiness is not exactly UP")
+            rows.append(f"readiness\t{service}\t{endpoint}\tUP")
+
+        wallet = self.client_factory(self.compose, "wallet")
+        metrics = poll_until(
+            "Wallet integrity first scan",
+            lambda: self._metrics(wallet),
+            self._clean_first_scan,
+            timeout=60,
+            interval=0.25,
+        )
+        rows.extend(
+            (
+                f"integrity\twallet\t{INTEGRITY_METRICS[0]}\t0",
+                f"integrity\twallet\t{INTEGRITY_METRICS[1]}\t0",
+                f"integrity\twallet\t{INTEGRITY_METRICS[2]}\t{int(metrics[2])}",
+            )
+        )
+        self.store.write("readiness.tsv", "\n".join(rows) + "\n")
+
+    @staticmethod
+    def _metrics(client: ContainerHttpClient) -> tuple[decimal.Decimal, ...]:
+        response = client.request("GET", "/actuator/prometheus").require_status(200)
+        try:
+            body = response.body.decode("utf-8")
+        except UnicodeDecodeError as error:
+            raise RuntimeError("Wallet metrics are not UTF-8") from error
+        values = []
+        for name in INTEGRITY_METRICS:
+            matches = re.findall(rf"^{name}\s+({NUMBER})$", body, re.MULTILINE)
+            if len(matches) != 1:
+                raise RuntimeError(f"expected one unlabelled {name} metric")
+            values.append(decimal.Decimal(matches[0]))
+        return tuple(values)
+
+    @staticmethod
+    def _clean_first_scan(values: tuple[decimal.Decimal, ...]) -> bool:
+        drift, failed, checked = values
+        if drift != 0 or failed != 0:
+            raise RuntimeError("Wallet integrity first scan is not clean")
+        if checked != checked.to_integral_value():
+            raise RuntimeError("Wallet integrity timestamp is not an epoch second")
+        return checked > 0


## `test(evidence): reject unready release state`

diff --git a/tests/test_readiness_evidence.py b/tests/test_readiness_evidence.py
new file mode 100644
index 0000000..ceb73ba
--- /dev/null
+++ b/tests/test_readiness_evidence.py
@@ -0,0 +1,85 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.http import HttpResponse
+from scripts.cold_gate.readiness_evidence import READINESS_ENDPOINTS, ReadinessEvidence
+from scripts.cold_gate.redaction import EvidenceRedactor
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+METRICS = (
+    b"wallet_integrity_total_drift 0.0\n"
+    b"wallet_integrity_scan_failed 0.0\n"
+    b"wallet_integrity_last_checked_epoch_seconds 1.777e9\n"
+)
+
+
+class FakeClient:
+    def __init__(self, service: str, statuses: dict[str, str], metrics: bytes) -> None:
+        self.service = service
+        self.statuses = statuses
+        self.metrics = metrics
+        self.calls = []
+
+    def request(self, method: str, path: str) -> HttpResponse:
+        self.calls.append((method, path))
+        if path == "/actuator/prometheus":
+            return HttpResponse(200, (), self.metrics)
+        body = ('{"status":"%s"}' % self.statuses[self.service]).encode()
+        return HttpResponse(200, (), body)
+
+
+class ReadinessEvidenceTest(unittest.TestCase):
+    def fixture(self, root: pathlib.Path, statuses=None, metrics=METRICS):
+        context = ColdGateContext.create(root, SHA, "00000001")
+        store = EvidenceStore(context, EvidenceRedactor(["redaction-secret-value"]))
+        values = statuses or {service: "UP" for service, _path in READINESS_ENDPOINTS}
+        clients = []
+
+        def factory(_compose, service):
+            client = FakeClient(service, values, metrics)
+            clients.append(client)
+            return client
+
+        compose = type("FakeCompose", (), {"context": context})()
+        return context, store, clients, factory, compose
+
+    def test_records_exact_readiness_and_clean_first_scan(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            context, store, clients, factory, compose = self.fixture(pathlib.Path(temporary))
+
+            ReadinessEvidence(compose, store, factory).capture()
+
+            lines = (context.evidence / "readiness.tsv").read_text().splitlines()
+            self.assertEqual(lines[0], "kind\tservice\tcheck\tvalue")
+            self.assertEqual([line.split("\t")[1] for line in lines[1:8]],
+                             [service for service, _path in READINESS_ENDPOINTS])
+            self.assertEqual(lines[-3:], [
+                "integrity\twallet\twallet_integrity_total_drift\t0",
+                "integrity\twallet\twallet_integrity_scan_failed\t0",
+                "integrity\twallet\twallet_integrity_last_checked_epoch_seconds\t1777000000",
+            ])
+            self.assertEqual(clients[-1].calls, [("GET", "/actuator/prometheus")])
+
+    def test_rejects_non_up_service_or_integrity_drift_before_writing(self) -> None:
+        cases = (
+            ({"admin": "DOWN"}, METRICS, "admin readiness"),
+            ({}, METRICS.replace(b"total_drift 0.0", b"total_drift 1.0"), "not clean"),
+        )
+        for overrides, metrics, message in cases:
+            with self.subTest(message=message), tempfile.TemporaryDirectory() as temporary:
+                statuses = {service: "UP" for service, _path in READINESS_ENDPOINTS}
+                statuses.update(overrides)
+                context, store, _clients, factory, compose = self.fixture(
+                    pathlib.Path(temporary), statuses, metrics
+                )
+                with self.assertRaisesRegex(RuntimeError, message):
+                    ReadinessEvidence(compose, store, factory).capture()
+                self.assertEqual(list(context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


## `build(evidence): record fixed E2E outcomes`

diff --git a/scripts/cold_gate/scenario_evidence.py b/scripts/cold_gate/scenario_evidence.py
new file mode 100644
index 0000000..34c58d3
--- /dev/null
+++ b/scripts/cold_gate/scenario_evidence.py
@@ -0,0 +1,40 @@
+from __future__ import annotations
+
+from collections.abc import Iterable
+
+from e2e.scenarios import SCENARIOS
+from scripts.cold_gate.evidence import EvidenceStore
+
+
+EXPECTED_SCENARIOS = (
+    "authenticated-placement-and-settlement",
+    "risk-outage-pending-recovery",
+    "wallet-lost-response-exactly-once",
+    "lifecycle-before-placement-refund",
+    "result-before-placement-settlement",
+    "payout-increase-correction",
+    "payout-decrease-blocked-recovery",
+    "admin-candidate-approve-reject",
+    "admin-revision-retry",
+    "revision-ordering-projection",
+    "replay-invariance",
+    "partition-two-poison-dlt",
+    "admin-audit-downstream-correlation",
+)
+
+
+class ScenarioEvidence:
+    def __init__(self, store: EvidenceStore) -> None:
+        self.store = store
+
+    def capture(self, passed: Iterable[str]) -> None:
+        actual = tuple(passed)
+        if (
+            tuple(module.NAME for module in SCENARIOS) != EXPECTED_SCENARIOS
+            or len(set(EXPECTED_SCENARIOS)) != 13
+            or actual != EXPECTED_SCENARIOS
+        ):
+            raise RuntimeError("E2E PASS inventory is incomplete or out of order")
+        rows = ["scenario\tresult"]
+        rows.extend(f"{name}\tPASS" for name in actual)
+        self.store.write("scenarios.tsv", "\n".join(rows) + "\n")


## `test(evidence): enforce complete E2E outcomes`

diff --git a/tests/test_scenario_evidence.py b/tests/test_scenario_evidence.py
new file mode 100644
index 0000000..7b14bdb
--- /dev/null
+++ b/tests/test_scenario_evidence.py
@@ -0,0 +1,47 @@
+import pathlib
+import tempfile
+import unittest
+
+from scripts.cold_gate.context import ColdGateContext
+from scripts.cold_gate.evidence import EvidenceStore
+from scripts.cold_gate.redaction import EvidenceRedactor
+from scripts.cold_gate.scenario_evidence import EXPECTED_SCENARIOS, ScenarioEvidence
+
+
+SHA = "0123456789abcdef0123456789abcdef01234567"
+
+
+class ScenarioEvidenceTest(unittest.TestCase):
+    def store(self, root: pathlib.Path) -> EvidenceStore:
+        context = ColdGateContext.create(root, SHA, "00000001")
+        return EvidenceStore(context, EvidenceRedactor(["redaction-secret-value"]))
+
+    def test_records_the_fixed_thirteen_passes_in_order(self) -> None:
+        with tempfile.TemporaryDirectory() as temporary:
+            store = self.store(pathlib.Path(temporary))
+
+            ScenarioEvidence(store).capture(EXPECTED_SCENARIOS)
+
+            lines = (store.context.evidence / "scenarios.tsv").read_text().splitlines()
+            self.assertEqual(lines[0], "scenario\tresult")
+            self.assertEqual(len(lines), 14)
+            self.assertEqual(
+                lines[1:], [f"{name}\tPASS" for name in EXPECTED_SCENARIOS]
+            )
+
+    def test_rejects_missing_duplicate_or_out_of_order_passes(self) -> None:
+        invalid = (
+            EXPECTED_SCENARIOS[:-1],
+            EXPECTED_SCENARIOS[:-1] + (EXPECTED_SCENARIOS[0],),
+            tuple(reversed(EXPECTED_SCENARIOS)),
+        )
+        for passed in invalid:
+            with self.subTest(passed=passed), tempfile.TemporaryDirectory() as temporary:
+                store = self.store(pathlib.Path(temporary))
+                with self.assertRaisesRegex(RuntimeError, "incomplete or out of order"):
+                    ScenarioEvidence(store).capture(passed)
+                self.assertEqual(list(store.context.evidence.iterdir()), [])
+
+
+if __name__ == "__main__":
+    unittest.main()


