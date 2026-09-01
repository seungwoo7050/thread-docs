# 저장소 격리와 마이그레이션 무결성

## `build(postgres): bootstrap service databases`

diff --git a/compose.yaml b/compose.yaml
new file mode 100644
index 0000000..6e5a546
--- /dev/null
+++ b/compose.yaml
@@ -0,0 +1,25 @@
+name: sportsbook
+
+services:
+  postgres:
+    image: postgres:16-alpine
+    environment:
+      POSTGRES_USER: sportsbook
+      POSTGRES_DB: postgres
+      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-}
+    volumes:
+      - postgres-data:/var/lib/postgresql/data
+      - ./docker/postgres-init.sql:/docker-entrypoint-initdb.d/10-databases.sql:ro
+    healthcheck:
+      test: ["CMD-SHELL", "pg_isready -U sportsbook -d postgres"]
+      interval: 2s
+      timeout: 3s
+      retries: 30
+    networks: [backend]
+
+volumes:
+  postgres-data:
+
+networks:
+  backend:
+    internal: true
diff --git a/docker/postgres-init.sql b/docker/postgres-init.sql
new file mode 100644
index 0000000..0a6ce81
--- /dev/null
+++ b/docker/postgres-init.sql
@@ -0,0 +1,13 @@
+\set ON_ERROR_STOP on
+
+SELECT 'CREATE DATABASE wallet OWNER sportsbook'
+WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'wallet')\gexec
+
+SELECT 'CREATE DATABASE betting OWNER sportsbook'
+WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'betting')\gexec
+
+SELECT 'CREATE DATABASE settlement OWNER sportsbook'
+WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'settlement')\gexec
+
+SELECT 'CREATE DATABASE admin OWNER sportsbook'
+WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'admin')\gexec


## `test(postgres): verify four database bootstrap`

diff --git a/tests/test_postgres_bootstrap.py b/tests/test_postgres_bootstrap.py
new file mode 100644
index 0000000..7b6381f
--- /dev/null
+++ b/tests/test_postgres_bootstrap.py
@@ -0,0 +1,43 @@
+from tests.compose_fixture import ComposeFixture
+
+
+class PostgresBootstrapTest(ComposeFixture):
+    def test_bootstraps_exactly_the_four_service_databases(self) -> None:
+        started = self.compose("up", "--detach", "--wait", "postgres")
+        self.assertEqual(started.returncode, 0, started.stderr)
+
+        query = self.compose(
+            "exec",
+            "--no-TTY",
+            "postgres",
+            "psql",
+            "--username",
+            "sportsbook",
+            "--dbname",
+            "postgres",
+            "--tuples-only",
+            "--no-align",
+            "--field-separator",
+            "|",
+            "--command",
+            "SELECT datname, pg_get_userbyid(datdba) FROM pg_database "
+            "WHERE datallowconn AND NOT datistemplate AND datname <> 'postgres' "
+            "ORDER BY datname",
+        )
+
+        self.assertEqual(query.returncode, 0, query.stderr)
+        self.assertEqual(
+            query.stdout.splitlines(),
+            [
+                "admin|sportsbook",
+                "betting|sportsbook",
+                "settlement|sportsbook",
+                "wallet|sportsbook",
+            ],
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(redis): isolate risk persistence`

diff --git a/compose.yaml b/compose.yaml
index 8295b66..3a34705 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -1,5 +1,22 @@
 name: sportsbook
 
+x-redis-service: &redis-service
+  image: redis:7.4-alpine
+  command:
+    - redis-server
+    - --appendonly
+    - "yes"
+    - --appendfsync
+    - everysec
+    - --maxmemory-policy
+    - noeviction
+  healthcheck:
+    test: ["CMD", "redis-cli", "ping"]
+    interval: 2s
+    timeout: 3s
+    retries: 30
+  networks: [backend]
+
 services:
   postgres:
     image: postgres:16-alpine
@@ -59,9 +76,15 @@ services:
         condition: service_healthy
     networks: [backend]
 
+  redis-risk:
+    <<: *redis-service
+    volumes:
+      - redis-risk-data:/data
+
 volumes:
   postgres-data:
   kafka-data:
+  redis-risk-data:
 
 networks:
   backend:


## `test(redis): verify risk storage contract`

diff --git a/tests/redis_fixture.py b/tests/redis_fixture.py
new file mode 100644
index 0000000..f8737aa
--- /dev/null
+++ b/tests/redis_fixture.py
@@ -0,0 +1,54 @@
+import json
+
+from tests.compose_fixture import ComposeFixture
+
+
+class RedisFixture(ComposeFixture):
+    def start_redis(self, *services: str) -> None:
+        started = self.compose("up", "--detach", "--wait", *services)
+        self.assertEqual(started.returncode, 0, started.stderr)
+
+    def redis(self, service: str, *arguments: str):
+        return self.compose("exec", "--no-TTY", service, "redis-cli", *arguments)
+
+    def assert_redis_contract(self, service: str, volume: str) -> None:
+        self.start_redis(service)
+        for option, expected in {
+            "appendonly": "yes",
+            "appendfsync": "everysec",
+            "maxmemory-policy": "noeviction",
+        }.items():
+            with self.subTest(service=service, option=option):
+                config = self.redis(service, "CONFIG", "GET", option)
+                self.assertEqual(config.returncode, 0, config.stderr)
+                self.assertEqual(config.stdout.splitlines(), [option, expected])
+
+        rendered = self.compose("config", "--format", "json")
+        self.assertEqual(rendered.returncode, 0, rendered.stderr)
+        service_config = json.loads(rendered.stdout)["services"][service]
+        data_mounts = [
+            mount
+            for mount in service_config["volumes"]
+            if mount["target"] == "/data"
+        ]
+        self.assertEqual(
+            data_mounts,
+            [
+                {
+                    "type": "volume",
+                    "source": volume,
+                    "target": "/data",
+                    "volume": {},
+                }
+            ],
+        )
+
+    def assert_isolated_values(self, *services: str) -> None:
+        self.start_redis(*services)
+        for index, service in enumerate(services):
+            stored = self.redis(service, "SET", "contract:isolation", str(index))
+            self.assertEqual(stored.returncode, 0, stored.stderr)
+        for index, service in enumerate(services):
+            loaded = self.redis(service, "GET", "contract:isolation")
+            self.assertEqual(loaded.returncode, 0, loaded.stderr)
+            self.assertEqual(loaded.stdout.strip(), str(index))
diff --git a/tests/test_redis_risk.py b/tests/test_redis_risk.py
new file mode 100644
index 0000000..6d53a89
--- /dev/null
+++ b/tests/test_redis_risk.py
@@ -0,0 +1,12 @@
+from tests.redis_fixture import RedisFixture
+
+
+class RiskRedisTest(RedisFixture):
+    def test_uses_dedicated_aof_noeviction_storage(self) -> None:
+        self.assert_redis_contract("redis-risk", "redis-risk-data")
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(redis): isolate gateway persistence`

diff --git a/compose.yaml b/compose.yaml
index ee32088..6d8a4e8 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -91,12 +91,18 @@ services:
     volumes:
       - redis-wallet-data:/data
 
+  redis-gateway:
+    <<: *redis-service
+    volumes:
+      - redis-gateway-data:/data
+
 volumes:
   postgres-data:
   kafka-data:
   redis-risk-data:
   redis-odds-data:
   redis-wallet-data:
+  redis-gateway-data:
 
 networks:
   backend:


## `build(redis): isolate wallet persistence`

diff --git a/compose.yaml b/compose.yaml
index 1118958..ee32088 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -86,11 +86,17 @@ services:
     volumes:
       - redis-odds-data:/data
 
+  redis-wallet:
+    <<: *redis-service
+    volumes:
+      - redis-wallet-data:/data
+
 volumes:
   postgres-data:
   kafka-data:
   redis-risk-data:
   redis-odds-data:
+  redis-wallet-data:
 
 networks:
   backend:


## `test(redis): verify wallet storage isolation`

diff --git a/tests/test_redis_wallet.py b/tests/test_redis_wallet.py
new file mode 100644
index 0000000..50b923e
--- /dev/null
+++ b/tests/test_redis_wallet.py
@@ -0,0 +1,13 @@
+from tests.redis_fixture import RedisFixture
+
+
+class WalletRedisTest(RedisFixture):
+    def test_isolates_durable_wallet_storage_from_other_domains(self) -> None:
+        self.assert_redis_contract("redis-wallet", "redis-wallet-data")
+        self.assert_isolated_values("redis-risk", "redis-odds", "redis-wallet")
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(redis): isolate odds projections`

diff --git a/compose.yaml b/compose.yaml
index 3a34705..1118958 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -81,10 +81,16 @@ services:
     volumes:
       - redis-risk-data:/data
 
+  redis-odds:
+    <<: *redis-service
+    volumes:
+      - redis-odds-data:/data
+
 volumes:
   postgres-data:
   kafka-data:
   redis-risk-data:
+  redis-odds-data:
 
 networks:
   backend:


## `test(redis): verify odds storage isolation`

diff --git a/tests/test_redis_odds.py b/tests/test_redis_odds.py
new file mode 100644
index 0000000..2d87029
--- /dev/null
+++ b/tests/test_redis_odds.py
@@ -0,0 +1,13 @@
+from tests.redis_fixture import RedisFixture
+
+
+class OddsRedisTest(RedisFixture):
+    def test_isolates_durable_projection_storage_from_risk(self) -> None:
+        self.assert_redis_contract("redis-odds", "redis-odds-data")
+        self.assert_isolated_values("redis-risk", "redis-odds")
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `test(redis): verify four instance isolation`

diff --git a/tests/test_redis_gateway.py b/tests/test_redis_gateway.py
new file mode 100644
index 0000000..3d73d03
--- /dev/null
+++ b/tests/test_redis_gateway.py
@@ -0,0 +1,37 @@
+import json
+
+from tests.redis_fixture import RedisFixture
+
+
+REDIS_SERVICES = {"redis-risk", "redis-odds", "redis-wallet", "redis-gateway"}
+
+
+class GatewayRedisTest(RedisFixture):
+    def test_completes_the_exact_four_instance_isolation_boundary(self) -> None:
+        self.assert_redis_contract("redis-gateway", "redis-gateway-data")
+        self.assert_isolated_values(*sorted(REDIS_SERVICES))
+
+        rendered = self.compose("config", "--format", "json")
+        self.assertEqual(rendered.returncode, 0, rendered.stderr)
+        services = json.loads(rendered.stdout)["services"]
+        self.assertEqual(
+            {name for name in services if name.startswith("redis-")}, REDIS_SERVICES
+        )
+        volumes = {
+            services[name]["volumes"][0]["source"] for name in REDIS_SERVICES
+        }
+        self.assertEqual(
+            volumes,
+            {
+                "redis-risk-data",
+                "redis-odds-data",
+                "redis-wallet-data",
+                "redis-gateway-data",
+            },
+        )
+
+
+if __name__ == "__main__":
+    import unittest
+
+    unittest.main()


## `build(gate): query owned release databases`

diff --git a/scripts/cold_gate/database.py b/scripts/cold_gate/database.py
new file mode 100644
index 0000000..f11831f
--- /dev/null
+++ b/scripts/cold_gate/database.py
@@ -0,0 +1,72 @@
+from __future__ import annotations
+
+import csv
+import io
+import re
+import subprocess
+import uuid
+
+from scripts.cold_gate.compose import ComposeProject
+
+
+DATABASES = frozenset({"wallet", "betting", "settlement", "admin"})
+
+
+class PostgresClient:
+    def __init__(self, compose: ComposeProject) -> None:
+        self.compose = compose
+
+    def query(self, database: str, statement: str) -> list[dict[str, str]]:
+        if database not in DATABASES:
+            raise ValueError("database is outside the release inventory")
+        normalized = statement.strip()
+        if (
+            not re.match(r"^(SELECT|WITH|UPDATE)\b", normalized, re.IGNORECASE)
+            or "\0" in normalized
+            or ";" in normalized
+            or len(normalized.encode()) > 16_384
+        ):
+            raise ValueError("database statement is outside the gate contract")
+        try:
+            result = self.compose.run(
+                "exec",
+                "-T",
+                "postgres",
+                "psql",
+                "--no-psqlrc",
+                "--set",
+                "ON_ERROR_STOP=1",
+                "--username",
+                "sportsbook",
+                "--dbname",
+                database,
+                "--csv",
+                "--command",
+                normalized,
+                capture_output=True,
+            )
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError(f"PostgreSQL query failed for {database}") from error
+        reader = csv.DictReader(io.StringIO(result.stdout))
+        if reader.fieldnames is None or len(reader.fieldnames) != len(set(reader.fieldnames)):
+            raise RuntimeError("PostgreSQL result columns are invalid")
+        return [dict(row) for row in reader]
+
+    def one(self, database: str, statement: str) -> dict[str, str]:
+        rows = self.query(database, statement)
+        if len(rows) != 1:
+            raise RuntimeError(f"expected one PostgreSQL row, observed {len(rows)}")
+        return rows[0]
+
+    def scalar(self, database: str, statement: str) -> str:
+        row = self.one(database, statement)
+        if len(row) != 1:
+            raise RuntimeError("expected one PostgreSQL column")
+        return next(iter(row.values()))
+
+
+def uuid_literal(value: str) -> str:
+    parsed = uuid.UUID(value)
+    if str(parsed) != value:
+        raise ValueError("SQL UUID must be canonical")
+    return f"'{value}'::uuid"


## `build(gate): query isolated Redis state`

diff --git a/scripts/cold_gate/redis.py b/scripts/cold_gate/redis.py
new file mode 100644
index 0000000..63e0acc
--- /dev/null
+++ b/scripts/cold_gate/redis.py
@@ -0,0 +1,44 @@
+from __future__ import annotations
+
+import subprocess
+
+from scripts.cold_gate.compose import ComposeProject
+
+
+REDIS_SERVICES = frozenset({"redis-risk", "redis-odds", "redis-wallet", "redis-gateway"})
+
+
+class RedisClient:
+    def __init__(self, compose: ComposeProject, service: str) -> None:
+        if service not in REDIS_SERVICES:
+            raise ValueError("Redis service is outside the release inventory")
+        self.compose = compose
+        self.service = service
+
+    def command(self, name: str, *arguments: str) -> tuple[str, ...]:
+        values = (name.upper(), *arguments)
+        if (
+            name.upper() not in {"GET", "SET", "HGET", "EXISTS", "TTL"}
+            or any(not value or "\0" in value or "\r" in value or "\n" in value for value in values)
+            or sum(len(value.encode()) for value in values) > 4096
+        ):
+            raise ValueError("Redis command is outside the gate contract")
+        try:
+            result = self.compose.run(
+                "exec",
+                "-T",
+                self.service,
+                "redis-cli",
+                "--raw",
+                *values,
+                capture_output=True,
+            )
+        except subprocess.CalledProcessError as error:
+            raise RuntimeError(f"Redis command failed for {self.service}") from error
+        return tuple(result.stdout.rstrip("\n").splitlines()) if result.stdout else ()
+
+    def scalar(self, name: str, *arguments: str) -> str:
+        lines = self.command(name, *arguments)
+        if len(lines) != 1:
+            raise RuntimeError("expected one Redis response line")
+        return lines[0]


## `test(gate): reject cross-instance Redis access`

diff --git a/tests/test_cold_gate_redis.py b/tests/test_cold_gate_redis.py
new file mode 100644
index 0000000..3f6bbe3
--- /dev/null
+++ b/tests/test_cold_gate_redis.py
@@ -0,0 +1,57 @@
+import subprocess
+import unittest
+
+from scripts.cold_gate.redis import RedisClient
+
+
+class FakeCompose:
+    def __init__(self, output: str = "OK\n", failure: bool = False) -> None:
+        self.output = output
+        self.failure = failure
+        self.calls = []
+
+    def run(self, *arguments, **options):
+        self.calls.append((arguments, options))
+        if self.failure:
+            raise subprocess.CalledProcessError(1, arguments, stderr="secret")
+        return subprocess.CompletedProcess(arguments, 0, stdout=self.output)
+
+
+class ColdGateRedisTest(unittest.TestCase):
+    def test_executes_one_allowlisted_command_in_the_selected_instance(self) -> None:
+        compose = FakeCompose()
+
+        result = RedisClient(compose, "redis-odds").scalar(
+            "set", "market:event:market", "OPEN", "EX", "3600"
+        )
+
+        arguments, options = compose.calls[0]
+        self.assertEqual(
+            arguments,
+            (
+                "exec", "-T", "redis-odds", "redis-cli", "--raw",
+                "SET", "market:event:market", "OPEN", "EX", "3600",
+            ),
+        )
+        self.assertEqual(options, {"capture_output": True})
+        self.assertEqual(result, "OK")
+
+    def test_rejects_unowned_instances_or_mutating_commands(self) -> None:
+        with self.assertRaisesRegex(ValueError, "outside the release"):
+            RedisClient(FakeCompose(), "redis")
+        client = RedisClient(FakeCompose(), "redis-risk")
+        for command in (("FLUSHALL",), ("GET", "bad\nkey"), ("SET", "", "value")):
+            with self.subTest(command=command):
+                with self.assertRaisesRegex(ValueError, "outside the gate"):
+                    client.command(*command)
+
+    def test_requires_a_scalar_and_hides_transport_output(self) -> None:
+        with self.assertRaisesRegex(RuntimeError, "one Redis"):
+            RedisClient(FakeCompose("one\ntwo\n"), "redis-wallet").scalar("GET", "key")
+        with self.assertRaisesRegex(RuntimeError, "redis-wallet") as captured:
+            RedisClient(FakeCompose(failure=True), "redis-wallet").command("GET", "key")
+        self.assertNotIn("secret", str(captured.exception))
+
+
+if __name__ == "__main__":
+    unittest.main()


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


