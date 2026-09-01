# PostgreSQL 정확 백업·복구 리허설

## `test(database): rehearse separated runtime roles`

diff --git a/operations/tests/test_database_role_rehearsal.py b/operations/tests/test_database_role_rehearsal.py
new file mode 100644
index 0000000..7576d00
--- /dev/null
+++ b/operations/tests/test_database_role_rehearsal.py
@@ -0,0 +1,281 @@
+from __future__ import annotations
+
+import os
+from pathlib import Path
+import stat
+import subprocess
+import tempfile
+import unittest
+
+
+class DatabaseRoleRehearsalContractTests(unittest.TestCase):
+    @classmethod
+    def setUpClass(cls):
+        super().setUpClass()
+        cls.root = Path(__file__).resolve().parents[2]
+        cls.script = cls.root / "scripts" / "check-database-roles"
+
+    def run_script(self, *arguments: str, env=None):
+        return subprocess.run(
+            ["/bin/sh", str(self.script), *arguments],
+            cwd=self.root,
+            env=env,
+            capture_output=True,
+            text=True,
+            check=False,
+            timeout=10,
+        )
+
+    def complete_arguments(self, **overrides):
+        values = {
+            "host": "127.0.0.1",
+            "port": "5432",
+            "admin_role": "postgres",
+            "admin_password_env": (
+                "TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD"
+            ),
+            "database_prefix": "travel_readiness_rolecheck_unit",
+            "safety_token": (
+                "ROLE_REHEARSAL_DISPOSABLE:"
+                "travel_readiness_rolecheck_unit"
+            ),
+        }
+        values.update(overrides)
+        return [
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
+    def test_script_is_executable_and_help_is_fixed(self):
+        self.assertEqual(stat.S_IMODE(self.script.stat().st_mode), 0o755)
+        result = self.run_script("--help")
+        self.assertEqual(result.returncode, 0)
+        self.assertEqual(result.stderr, "")
+        self.assertEqual(
+            result.stdout,
+            "usage: check-database-roles --host LOOPBACK --port PORT "
+            "--admin-role ROLE --admin-password-env "
+            "TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD "
+            "--database-prefix travel_readiness_rolecheck_NAME "
+            "--safety-token ROLE_REHEARSAL_DISPOSABLE:"
+            "travel_readiness_rolecheck_NAME\n",
+        )
+
+    def test_missing_unknown_and_duplicate_arguments_are_fixed(self):
+        for arguments in (
+            (),
+            ("--unknown",),
+            ("--host", "127.0.0.1", "--host", "localhost"),
+        ):
+            with self.subTest(arguments=arguments):
+                result = self.run_script(*arguments)
+                self.assertEqual(result.returncode, 64)
+                self.assertEqual(result.stdout, "")
+                self.assertEqual(
+                    result.stderr,
+                    "database_role_check=INVALID_ARGUMENTS\n",
+                )
+
+    def test_non_loopback_and_production_like_targets_are_refused_first(self):
+        remote = self.run_script(
+            *self.complete_arguments(host="database.invalid")
+        )
+        self.assertEqual(remote.returncode, 65)
+        self.assertEqual(
+            remote.stderr,
+            "database_role_check=NON_LOOPBACK_REFUSED\n",
+        )
+
+        for suffix in ("prod", "production", "live", "stage", "main", "release"):
+            prefix = f"travel_readiness_rolecheck_{suffix}"
+            result = self.run_script(
+                *self.complete_arguments(
+                    database_prefix=prefix,
+                    safety_token=f"ROLE_REHEARSAL_DISPOSABLE:{prefix}",
+                )
+            )
+            self.assertEqual(result.returncode, 65)
+            self.assertEqual(
+                result.stderr,
+                "database_role_check=PRODUCTION_LIKE_PREFIX_REFUSED\n",
+            )
+
+    def test_prefix_and_safety_token_must_name_one_disposable_target(self):
+        for prefix in (
+            "travel_readiness",
+            "rolecheck_unit",
+            "travel_readiness_rolecheck_",
+            "travel_readiness_rolecheck_BAD",
+            "travel_readiness_rolecheck_unit-with-dash",
+        ):
+            with self.subTest(prefix=prefix):
+                result = self.run_script(
+                    *self.complete_arguments(
+                        database_prefix=prefix,
+                        safety_token=f"ROLE_REHEARSAL_DISPOSABLE:{prefix}",
+                    )
+                )
+                self.assertEqual(result.returncode, 65)
+                self.assertEqual(
+                    result.stderr,
+                    "database_role_check=UNSAFE_PREFIX\n",
+                )
+
+        mismatch = self.run_script(
+            *self.complete_arguments(
+                safety_token=(
+                    "ROLE_REHEARSAL_DISPOSABLE:"
+                    "travel_readiness_rolecheck_other"
+                )
+            )
+        )
+        self.assertEqual(mismatch.returncode, 65)
+        self.assertEqual(
+            mismatch.stderr,
+            "database_role_check=SAFETY_TOKEN_MISMATCH\n",
+        )
+
+    def test_only_dedicated_admin_password_reference_is_accepted(self):
+        result = self.run_script(
+            *self.complete_arguments(admin_password_env="PGPASSWORD")
+        )
+        self.assertEqual(result.returncode, 65)
+        self.assertEqual(
+            result.stderr,
+            "database_role_check=UNSAFE_PASSWORD_REFERENCE\n",
+        )
+
+        environment = os.environ.copy()
+        environment.pop("TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD", None)
+        missing = self.run_script(
+            *self.complete_arguments(),
+            env=environment,
+        )
+        self.assertEqual(missing.returncode, 66)
+        self.assertEqual(
+            missing.stderr,
+            "database_role_check=ADMIN_PASSWORD_MISSING\n",
+        )
+
+    def test_wrong_psql_version_fails_without_connecting_or_leaking(self):
+        marker = "synthetic-admin-password-marker"
+        with tempfile.TemporaryDirectory() as temporary:
+            temporary_path = Path(temporary)
+            fake_bin = temporary_path / "bin"
+            fake_bin.mkdir()
+            fake_psql = fake_bin / "psql"
+            fake_psql.write_text(
+                "#!/bin/sh\n"
+                "if [ \"${1:-}\" = \"--version\" ]; then\n"
+                "  printf '%s\\n' 'psql (PostgreSQL) 17.0'\n"
+                "  exit 0\n"
+                "fi\n"
+                "printf '%s\\n' 'unexpected connection' >&2\n"
+                "exit 99\n",
+                encoding="utf-8",
+            )
+            fake_psql.chmod(fake_psql.stat().st_mode | stat.S_IXUSR)
+            environment = os.environ.copy()
+            environment["PATH"] = (
+                f"{fake_bin}:{environment.get('PATH', '')}"
+            )
+            environment[
+                "TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD"
+            ] = marker
+            result = self.run_script(
+                *self.complete_arguments(),
+                env=environment,
+            )
+
+        self.assertEqual(result.returncode, 69)
+        self.assertEqual(result.stdout, "")
+        self.assertEqual(
+            result.stderr,
+            "database_role_check=POSTGRESQL_18_6_REQUIRED\n",
+        )
+        self.assertNotIn(marker, result.stdout + result.stderr)
+        self.assertNotIn("unexpected connection", result.stderr)
+
+    def test_script_encodes_least_privilege_and_cleanup_contract(self):
+        script = self.script.read_text(encoding="utf-8")
+        lower = script.lower()
+
+        for required in (
+            "CREATE ROLE",
+            'POSTGRESQL_REQUIRED_VERSION_NUM="180006"',
+            "NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT",
+            "CREATE DATABASE",
+            "ALTER SCHEMA public OWNER TO",
+            "REVOKE ALL ON SCHEMA public FROM PUBLIC",
+            "GRANT USAGE ON SCHEMA public",
+            "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES",
+            "GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES",
+            "REVOKE INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER "
+            "ON TABLE public.django_migrations",
+            "ALTER DEFAULT PRIVILEGES FOR ROLE",
+            "REVOKE EXECUTE ON ALL FUNCTIONS",
+            "CREATE TABLE public.rolecheck_forbidden_create",
+            "ALTER TABLE public.auth_group ADD COLUMN",
+            "migrate reviews zero --fake",
+            "pg_get_userbyid(datdba)",
+            "has_schema_privilege",
+            "DROP DATABASE",
+            "DROP ROLE",
+            'PREVIOUS_ARTIFACT_COMMIT="eb10145b3865c1dfa18c614639e895da2d8b8999"',
+            "PREVIOUS_ARTIFACT_INCOMPATIBLE",
+            "has_function_privilege",
+        ):
+            self.assertIn(required, script)
+
+        self.assertIn('case "$host" in', script)
+        self.assertIn("127.0.0.1|localhost|::1", script)
+        self.assertIn("TARGET_ALREADY_EXISTS", script)
+        self.assertIn("current_setting('server_version_num')", script)
+        self.assertEqual(
+            script.count("DJANGO_SETTINGS_MODULE=travel_readiness.settings"),
+            3,
+        )
+        self.assertIn("set +x", script)
+        self.assertIn("umask 077", script)
+        self.assertIn("--no-password", script)
+        self.assertIn(">/dev/null 2>&1", script)
+        self.assertNotIn(".env.local", lower)
+        self.assertNotIn("mofa_travel_alarm_service_key", lower)
+        self.assertNotIn("database_url", lower)
+        self.assertNotIn("set -x", lower)
+        self.assertNotIn("eval ", lower)
+        self.assertNotIn("printenv \"$admin_password_env\"\n", script)
+        self.assertNotIn(
+            "GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public",
+            script,
+        )
+
+    def test_runtime_probe_covers_public_ssr_readiness_and_disposable_dml(self):
+        script = self.script.read_text(encoding="utf-8")
+        for required in (
+            'client.get("/healthz", secure=True)',
+            'client.get("/readyz", secure=True)',
+            'client.get("/results/", secure=True)',
+            'b"entry-card"',
+            'b"warning-card"',
+            'git -C "$project_dir" archive',
+            'run_previous_python "$previous_probe"',
+            "Group.objects.create",
+            "Group.objects.filter(pk=group.pk).update",
+            "group.delete()",
+        ):
+            self.assertIn(required, script)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/scripts/check-database-roles b/scripts/check-database-roles
new file mode 100755
index 0000000..fa183aa
--- /dev/null
+++ b/scripts/check-database-roles
@@ -0,0 +1,456 @@
+#!/bin/sh
+
+set +x
+set -eu
+umask 077
+LC_ALL=C
+export LC_ALL
+unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD
+
+POSTGRESQL_REQUIRED_VERSION="18.6"
+POSTGRESQL_REQUIRED_VERSION_NUM="180006"
+PREVIOUS_ARTIFACT_COMMIT="eb10145b3865c1dfa18c614639e895da2d8b8999"
+
+usage() {
+    printf '%s\n' 'usage: check-database-roles --host LOOPBACK --port PORT --admin-role ROLE --admin-password-env TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD --database-prefix travel_readiness_rolecheck_NAME --safety-token ROLE_REHEARSAL_DISPOSABLE:travel_readiness_rolecheck_NAME'
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
+    case "$value" in
+        [a-z_]* ) ;;
+        * ) return 1 ;;
+    esac
+    case "$value" in
+        *[!a-z0-9_]* ) return 1 ;;
+    esac
+}
+
+is_port() {
+    value=$1
+    case "$value" in
+        ''|*[!0-9]* ) return 1 ;;
+    esac
+    [ "${#value}" -le 5 ] || return 1
+    [ "$value" -ge 1 ] 2>/dev/null || return 1
+    [ "$value" -le 65535 ] 2>/dev/null || return 1
+}
+
+admin_psql() {
+    connection_database=$1
+    shift
+    PGPASSWORD="$admin_password" \
+    PGAPPNAME=travel-readiness-role-check-admin \
+    PGCONNECT_TIMEOUT=5 \
+        psql \
+        --no-password \
+        --host="$host" \
+        --port="$port" \
+        --dbname="$connection_database" \
+        --username="$admin_role" \
+        --no-psqlrc \
+        --set=ON_ERROR_STOP=1 \
+        "$@"
+}
+
+admin_scalar() {
+    query=$1
+    admin_psql postgres \
+        --quiet --tuples-only --no-align --command="$query" 2>/dev/null
+}
+
+role_psql() {
+    connection_role=$1
+    connection_password=$2
+    shift 2
+    PGPASSWORD="$connection_password" \
+    PGAPPNAME=travel-readiness-role-check \
+    PGCONNECT_TIMEOUT=5 \
+        psql \
+        --no-password \
+        --host="$host" \
+        --port="$port" \
+        --dbname="$database" \
+        --username="$connection_role" \
+        --no-psqlrc \
+        --set=ON_ERROR_STOP=1 \
+        "$@"
+}
+
+run_manage() {
+    connection_role=$1
+    connection_password=$2
+    shift 2
+    TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+    TRAVEL_READINESS_DB_NAME="$database" \
+    TRAVEL_READINESS_DB_USER="$connection_role" \
+    TRAVEL_READINESS_DB_PASSWORD="$connection_password" \
+    TRAVEL_READINESS_DB_HOST="$host" \
+    TRAVEL_READINESS_DB_PORT="$port" \
+    TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+    TRAVEL_READINESS_BUILD=0 \
+    TRAVEL_READINESS_DEBUG=0 \
+    TRAVEL_READINESS_HTTPS=0 \
+    DJANGO_SETTINGS_MODULE=travel_readiness.settings \
+    PYTHONPATH="$project_dir" \
+        "$python_bin" -s "$project_dir/manage.py" "$@"
+}
+
+run_runtime_python() {
+    program=$1
+    TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+    TRAVEL_READINESS_DB_NAME="$database" \
+    TRAVEL_READINESS_DB_USER="$runtime_role" \
+    TRAVEL_READINESS_DB_PASSWORD="$runtime_password" \
+    TRAVEL_READINESS_DB_HOST="$host" \
+    TRAVEL_READINESS_DB_PORT="$port" \
+    TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+    TRAVEL_READINESS_BUILD=0 \
+    TRAVEL_READINESS_DEBUG=0 \
+    TRAVEL_READINESS_HTTPS=0 \
+    DJANGO_SETTINGS_MODULE=travel_readiness.settings \
+    PYTHONPATH="$project_dir" \
+        "$python_bin" -s -c "$program"
+}
+
+run_previous_python() {
+    program=$1
+    (
+        cd "$previous_artifact_dir" || exit 1
+        TRAVEL_READINESS_SECRET_KEY="$django_secret" \
+        TRAVEL_READINESS_DB_NAME="$database" \
+        TRAVEL_READINESS_DB_USER="$runtime_role" \
+        TRAVEL_READINESS_DB_PASSWORD="$runtime_password" \
+        TRAVEL_READINESS_DB_HOST="$host" \
+        TRAVEL_READINESS_DB_PORT="$port" \
+        TRAVEL_READINESS_ALLOWED_HOSTS='testserver,localhost' \
+        TRAVEL_READINESS_BUILD=0 \
+        TRAVEL_READINESS_DEBUG=0 \
+        TRAVEL_READINESS_HTTPS=0 \
+        DJANGO_SETTINGS_MODULE=travel_readiness.settings \
+        PYTHONDONTWRITEBYTECODE=1 \
+        PYTHONPATH="$previous_artifact_dir" \
+            "$python_bin" -s -c "$program"
+    )
+}
+
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
+            [ "$#" -eq 1 ] || fail 'database_role_check=INVALID_ARGUMENTS' 64
+            usage
+            exit 0
+            ;;
+        --host|--port|--admin-role|--admin-password-env|--database-prefix|--safety-token)
+            [ "$#" -ge 2 ] || fail 'database_role_check=INVALID_ARGUMENTS' 64
+            option=$1
+            value=$2
+            shift 2
+            case "$option" in
+                --host) [ -z "$host" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; host=$value ;;
+                --port) [ -z "$port" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; port=$value ;;
+                --admin-role) [ -z "$admin_role" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; admin_role=$value ;;
+                --admin-password-env) [ -z "$admin_password_env" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; admin_password_env=$value ;;
+                --database-prefix) [ -z "$database_prefix" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; database_prefix=$value ;;
+                --safety-token) [ -z "$safety_token" ] || fail 'database_role_check=INVALID_ARGUMENTS' 64; safety_token=$value ;;
+            esac
+            ;;
+        *) fail 'database_role_check=INVALID_ARGUMENTS' 64 ;;
+    esac
+done
+
+[ -n "$host" ] \
+    && [ -n "$port" ] \
+    && [ -n "$admin_role" ] \
+    && [ -n "$admin_password_env" ] \
+    && [ -n "$database_prefix" ] \
+    && [ -n "$safety_token" ] \
+    || fail 'database_role_check=INVALID_ARGUMENTS' 64
+case "$host" in
+    127.0.0.1|localhost|::1) ;;
+    *) fail 'database_role_check=NON_LOOPBACK_REFUSED' 65 ;;
+esac
+is_port "$port" || fail 'database_role_check=INVALID_ARGUMENTS' 64
+is_identifier "$admin_role" || fail 'database_role_check=INVALID_ARGUMENTS' 64
+is_identifier "$database_prefix" || fail 'database_role_check=UNSAFE_PREFIX' 65
+[ "${#database_prefix}" -le 50 ] || fail 'database_role_check=UNSAFE_PREFIX' 65
+case "$database_prefix" in
+    travel_readiness_rolecheck_[a-z0-9]*) ;;
+    *) fail 'database_role_check=UNSAFE_PREFIX' 65 ;;
+esac
+case "$database_prefix" in
+    *prod*|*live*|*stag*|*main*|*master*|*release*)
+        fail 'database_role_check=PRODUCTION_LIKE_PREFIX_REFUSED' 65
+        ;;
+esac
+[ "$safety_token" = "ROLE_REHEARSAL_DISPOSABLE:$database_prefix" ] \
+    || fail 'database_role_check=SAFETY_TOKEN_MISMATCH' 65
+[ "$admin_password_env" = 'TRAVEL_READINESS_ROLE_CHECK_ADMIN_PASSWORD' ] \
+    || fail 'database_role_check=UNSAFE_PASSWORD_REFERENCE' 65
+command -v printenv >/dev/null 2>&1 \
+    || fail 'database_role_check=REQUIRED_TOOL_MISSING' 69
+admin_password=$(printenv "$admin_password_env" 2>/dev/null) \
+    || fail 'database_role_check=ADMIN_PASSWORD_MISSING' 66
+[ -n "$admin_password" ] \
+    || fail 'database_role_check=ADMIN_PASSWORD_MISSING' 66
+[ "${#admin_password}" -le 1024 ] \
+    || fail 'database_role_check=ADMIN_PASSWORD_INVALID' 66
+case "$admin_password" in
+    *'
+'*) fail 'database_role_check=ADMIN_PASSWORD_INVALID' 66 ;;
+esac
+
+database="${database_prefix}_db"
+migration_role="${database_prefix}_migration"
+runtime_role="${database_prefix}_runtime"
+is_identifier "$database" || fail 'database_role_check=UNSAFE_PREFIX' 65
+is_identifier "$migration_role" || fail 'database_role_check=UNSAFE_PREFIX' 65
+is_identifier "$runtime_role" || fail 'database_role_check=UNSAFE_PREFIX' 65
+[ "$admin_role" != "$migration_role" ] \
+    || fail 'database_role_check=ADMIN_ROLE_COLLISION' 65
+[ "$admin_role" != "$runtime_role" ] \
+    || fail 'database_role_check=ADMIN_ROLE_COLLISION' 65
+
+script_dir=$(CDPATH='' cd "$(dirname "$0")" && pwd -P)
+project_dir=$(CDPATH='' cd "$script_dir/.." && pwd -P)
+python_bin="$project_dir/.venv/bin/python"
+
+command -v psql >/dev/null 2>&1 \
+    || fail 'database_role_check=POSTGRESQL_18_6_REQUIRED' 69
+reported_version=$(psql --version 2>/dev/null) \
+    || fail 'database_role_check=POSTGRESQL_18_6_REQUIRED' 69
+case "$reported_version" in
+    "psql (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION"|"psql (PostgreSQL) $POSTGRESQL_REQUIRED_VERSION "*) ;;
+    *) fail 'database_role_check=POSTGRESQL_18_6_REQUIRED' 69 ;;
+esac
+[ -x "$python_bin" ] || fail 'database_role_check=PINNED_PYTHON_REQUIRED' 69
+command -v openssl >/dev/null 2>&1 \
+    || fail 'database_role_check=REQUIRED_TOOL_MISSING' 69
+command -v git >/dev/null 2>&1 \
+    || fail 'database_role_check=REQUIRED_TOOL_MISSING' 69
+command -v tar >/dev/null 2>&1 \
+    || fail 'database_role_check=REQUIRED_TOOL_MISSING' 69
+command -v mktemp >/dev/null 2>&1 \
+    || fail 'database_role_check=REQUIRED_TOOL_MISSING' 69
+
+server_version=$(admin_scalar "SELECT current_setting('server_version_num')") \
+    || fail 'database_role_check=ADMIN_CONNECTION_FAILED' 70
+[ "$server_version" = "$POSTGRESQL_REQUIRED_VERSION_NUM" ] \
+    || fail 'database_role_check=DATABASE_VERSION_MISMATCH' 70
+admin_is_superuser=$(admin_scalar "SELECT rolsuper FROM pg_catalog.pg_roles WHERE rolname = current_user") \
+    || fail 'database_role_check=ADMIN_CONNECTION_FAILED' 70
+[ "$admin_is_superuser" = 't' ] \
+    || fail 'database_role_check=ADMIN_CAPABILITY_REQUIRED' 70
+
+existing_targets=$(admin_scalar "SELECT (SELECT count(*) FROM pg_catalog.pg_database WHERE datname = '$database') + (SELECT count(*) FROM pg_catalog.pg_roles WHERE rolname IN ('$migration_role', '$runtime_role'))") \
+    || fail 'database_role_check=TARGET_PREFLIGHT_FAILED' 70
+[ "$existing_targets" = '0' ] \
+    || fail 'database_role_check=TARGET_ALREADY_EXISTS' 71
+
+migration_password=$(openssl rand -hex 24 2>/dev/null) \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+runtime_password=$(openssl rand -hex 24 2>/dev/null) \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+django_secret=$(openssl rand -hex 32 2>/dev/null) \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+case "$migration_password$runtime_password$django_secret" in
+    *[!a-f0-9]*) fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69 ;;
+esac
+[ "${#migration_password}" -eq 48 ] \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+[ "${#runtime_password}" -eq 48 ] \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+[ "${#django_secret}" -eq 64 ] \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+[ "$migration_password" != "$runtime_password" ] \
+    || fail 'database_role_check=PASSWORD_GENERATION_FAILED' 69
+
+roles_created=0
+database_created=0
+previous_artifact_dir=''
+
+cleanup() {
+    cleanup_result=0
+    if [ "$database_created" = '1' ]; then
+        admin_psql postgres --quiet --command="REVOKE CONNECT ON DATABASE \"$database\" FROM PUBLIC, \"$migration_role\", \"$runtime_role\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet --command="SELECT pg_catalog.pg_terminate_backend(pid) FROM pg_catalog.pg_stat_activity WHERE datname = '$database' AND pid <> pg_catalog.pg_backend_pid()" \
+            >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet --command="DROP DATABASE \"$database\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+    fi
+    if [ "$roles_created" = '1' ]; then
+        admin_psql postgres --quiet --command="DROP ROLE \"$runtime_role\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+        admin_psql postgres --quiet --command="DROP ROLE \"$migration_role\"" \
+            >/dev/null 2>&1 || cleanup_result=1
+    fi
+    if [ -n "$previous_artifact_dir" ]; then
+        case "$previous_artifact_dir" in
+            /tmp/travel-readiness-previous.*)
+                rm -rf -- "$previous_artifact_dir" 2>/dev/null \
+                    || cleanup_result=1
+                ;;
+            *) cleanup_result=1 ;;
+        esac
+    fi
+    return "$cleanup_result"
+}
+
+cleanup_on_exit() {
+    original_status=$?
+    trap - EXIT HUP INT TERM
+    if ! cleanup; then
+        printf '%s\n' 'database_role_check_cleanup=FAILED' >&2
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
+roles_sql=$(printf '%s\n%s\n' \
+    "CREATE ROLE \"$migration_role\" LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT NOREPLICATION NOBYPASSRLS PASSWORD '$migration_password';" \
+    "CREATE ROLE \"$runtime_role\" LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT NOREPLICATION NOBYPASSRLS PASSWORD '$runtime_password';")
+printf '%s\n' "$roles_sql" | admin_psql postgres --quiet --single-transaction \
+    >/dev/null 2>&1 || fail 'database_role_check=ROLE_CREATE_FAILED' 72
+roles_created=1
+
+printf '%s\n' "CREATE DATABASE \"$database\" OWNER \"$migration_role\" TEMPLATE template0;" \
+    | admin_psql postgres --quiet >/dev/null 2>&1 \
+    || fail 'database_role_check=DATABASE_CREATE_FAILED' 72
+database_created=1
+
+bootstrap_sql=$(printf '%s\n' \
+    "ALTER SCHEMA public OWNER TO \"$migration_role\";" \
+    "REVOKE ALL ON DATABASE \"$database\" FROM PUBLIC;" \
+    "GRANT CONNECT, TEMPORARY ON DATABASE \"$database\" TO \"$migration_role\";" \
+    "GRANT CONNECT ON DATABASE \"$database\" TO \"$runtime_role\";" \
+    'REVOKE ALL ON SCHEMA public FROM PUBLIC;' \
+    "GRANT USAGE, CREATE ON SCHEMA public TO \"$migration_role\";" \
+    "GRANT USAGE ON SCHEMA public TO \"$runtime_role\";")
+printf '%s\n' "$bootstrap_sql" \
+    | role_psql "$migration_role" "$migration_password" --quiet --single-transaction \
+        >/dev/null 2>&1 \
+    || fail 'database_role_check=DATABASE_BOOTSTRAP_FAILED' 72
+
+run_manage "$migration_role" "$migration_password" migrate --plan --noinput --verbosity 0 \
+    >/dev/null 2>&1 || fail 'database_role_check=MIGRATION_PLAN_FAILED' 73
+run_manage "$migration_role" "$migration_password" migrate --noinput --verbosity 0 \
+    >/dev/null 2>&1 || fail 'database_role_check=MIGRATION_FAILED' 73
+run_manage "$migration_role" "$migration_password" makemigrations --check --dry-run --verbosity 0 \
+    >/dev/null 2>&1 || fail 'database_role_check=MIGRATION_DRIFT' 73
+
+grants_sql=$(printf '%s\n' \
+    "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"$runtime_role\";" \
+    "GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA public TO \"$runtime_role\";" \
+    "REVOKE INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER ON TABLE public.django_migrations FROM \"$runtime_role\";" \
+    'REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA public FROM PUBLIC;' \
+    "REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA public FROM \"$runtime_role\";" \
+    "ALTER DEFAULT PRIVILEGES FOR ROLE \"$migration_role\" IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO \"$runtime_role\";" \
+    "ALTER DEFAULT PRIVILEGES FOR ROLE \"$migration_role\" IN SCHEMA public GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO \"$runtime_role\";" \
+    "ALTER DEFAULT PRIVILEGES FOR ROLE \"$migration_role\" IN SCHEMA public REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;")
+printf '%s\n' "$grants_sql" \
+    | role_psql "$migration_role" "$migration_password" --quiet --single-transaction \
+        >/dev/null 2>&1 \
+    || fail 'database_role_check=RUNTIME_GRANT_FAILED' 74
+
+runtime_probe='import django
+django.setup()
+from django.contrib.auth.models import Group
+from django.test import Client
+client = Client()
+assert client.get("/healthz", secure=True).status_code == 200
+assert client.get("/readyz", secure=True).status_code == 200
+results = client.get("/results/", secure=True)
+assert results.status_code == 200
+assert b"entry-card" in results.content
+assert b"warning-card" in results.content
+group = Group.objects.create(name="rolecheck_synthetic_group")
+assert Group.objects.filter(pk=group.pk).exists()
+Group.objects.filter(pk=group.pk).update(name="rolecheck_synthetic_group_updated")
+assert Group.objects.get(pk=group.pk).name == "rolecheck_synthetic_group_updated"
+group.delete()
+assert not Group.objects.filter(pk=group.pk).exists()'
+run_runtime_python "$runtime_probe" >/dev/null 2>&1 \
+    || fail 'database_role_check=RUNTIME_APPLICATION_PROBE_FAILED' 75
+
+resolved_previous_commit=$(git -C "$project_dir" rev-parse \
+    --verify "$PREVIOUS_ARTIFACT_COMMIT^{commit}" 2>/dev/null) \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_MISSING' 75
+[ "$resolved_previous_commit" = "$PREVIOUS_ARTIFACT_COMMIT" ] \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_MISMATCH' 75
+previous_artifact_dir=$(mktemp -d /tmp/travel-readiness-previous.XXXXXX \
+    2>/dev/null) || fail 'database_role_check=PREVIOUS_ARTIFACT_TEMP_FAILED' 75
+git -C "$project_dir" archive --format=tar \
+    --output="$previous_artifact_dir/artifact.tar" \
+    "$PREVIOUS_ARTIFACT_COMMIT" >/dev/null 2>&1 \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_EXPORT_FAILED' 75
+tar -xf "$previous_artifact_dir/artifact.tar" \
+    -C "$previous_artifact_dir" >/dev/null 2>&1 \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_EXPORT_FAILED' 75
+rm -f "$previous_artifact_dir/artifact.tar" 2>/dev/null \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_EXPORT_FAILED' 75
+
+previous_probe='import django
+django.setup()
+from django.test import Client
+client = Client()
+assert client.get("/healthz", secure=True).status_code == 200
+assert client.get("/", secure=True).status_code == 200
+results = client.get("/results/", secure=True)
+assert results.status_code == 200
+assert b"entry-card" in results.content
+assert b"warning-card" in results.content'
+run_previous_python "$previous_probe" >/dev/null 2>&1 \
+    || fail 'database_role_check=PREVIOUS_ARTIFACT_INCOMPATIBLE' 75
+
+if printf '%s\n' 'CREATE TABLE public.rolecheck_forbidden_create (id integer);' \
+    | role_psql "$runtime_role" "$runtime_password" --quiet --single-transaction \
+        >/dev/null 2>&1; then
+    fail 'database_role_check=RUNTIME_CREATE_NOT_DENIED' 76
+fi
+if printf '%s\n' 'ALTER TABLE public.auth_group ADD COLUMN rolecheck_forbidden_alter integer;' \
+    | role_psql "$runtime_role" "$runtime_password" --quiet --single-transaction \
+        >/dev/null 2>&1; then
+    fail 'database_role_check=RUNTIME_ALTER_NOT_DENIED' 76
+fi
+if run_manage "$runtime_role" "$runtime_password" migrate reviews zero --fake --noinput --verbosity 0 \
+    >/dev/null 2>&1; then
+    fail 'database_role_check=RUNTIME_MIGRATE_NOT_DENIED' 76
+fi
+
+ownership_check=$(admin_psql "$database" --quiet --tuples-only --no-align --command="SELECT (SELECT pg_catalog.pg_get_userbyid(datdba) = '$migration_role' FROM pg_catalog.pg_database WHERE datname = '$database') AND (SELECT pg_catalog.pg_get_userbyid(nspowner) = '$migration_role' FROM pg_catalog.pg_namespace WHERE nspname = 'public') AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind IN ('r', 'i', 'S') AND pg_catalog.pg_get_userbyid(c.relowner) <> '$migration_role') AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace WHERE n.nspname = 'public' AND pg_catalog.pg_get_userbyid(p.proowner) <> '$migration_role') AND NOT EXISTS (SELECT 1 FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND pg_catalog.pg_get_userbyid(c.relowner) = '$runtime_role')") \
+    2>/dev/null || fail 'database_role_check=OWNERSHIP_CHECK_FAILED' 76
+[ "$ownership_check" = 't' ] \
+    || fail 'database_role_check=OWNER_SEPARATION_FAILED' 76
+
+privilege_check=$(admin_psql "$database" --quiet --tuples-only --no-align --command="SELECT (SELECT rolcanlogin AND NOT rolsuper AND NOT rolcreatedb AND NOT rolcreaterole AND NOT rolreplication AND NOT rolbypassrls FROM pg_catalog.pg_roles WHERE rolname = '$migration_role') AND (SELECT rolcanlogin AND NOT rolsuper AND NOT rolcreatedb AND NOT rolcreaterole AND NOT rolreplication AND NOT rolbypassrls FROM pg_catalog.pg_roles WHERE rolname = '$runtime_role') AND NOT pg_catalog.pg_has_role('$runtime_role', '$migration_role', 'MEMBER') AND has_database_privilege('$runtime_role', '$database', 'CONNECT') AND NOT has_database_privilege('$runtime_role', '$database', 'TEMPORARY') AND has_schema_privilege('$runtime_role', 'public', 'USAGE') AND NOT has_schema_privilege('$runtime_role', 'public', 'CREATE') AND (SELECT bool_and(has_table_privilege('$runtime_role', c.oid, 'SELECT') AND has_table_privilege('$runtime_role', c.oid, 'INSERT') AND has_table_privilege('$runtime_role', c.oid, 'UPDATE') AND has_table_privilege('$runtime_role', c.oid, 'DELETE')) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind = 'r' AND c.relname <> 'django_migrations') AND has_table_privilege('$runtime_role', 'public.django_migrations', 'SELECT') AND NOT has_table_privilege('$runtime_role', 'public.django_migrations', 'INSERT') AND NOT has_table_privilege('$runtime_role', 'public.django_migrations', 'UPDATE') AND NOT has_table_privilege('$runtime_role', 'public.django_migrations', 'DELETE') AND (SELECT bool_and(has_sequence_privilege('$runtime_role', c.oid, 'USAGE') AND has_sequence_privilege('$runtime_role', c.oid, 'SELECT') AND has_sequence_privilege('$runtime_role', c.oid, 'UPDATE')) FROM pg_catalog.pg_class AS c JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace WHERE n.nspname = 'public' AND c.relkind = 'S') AND (SELECT bool_and(NOT has_function_privilege('$runtime_role', p.oid, 'EXECUTE')) FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_namespace AS n ON n.oid = p.pronamespace WHERE n.nspname = 'public')") \
+    2>/dev/null || fail 'database_role_check=PRIVILEGE_CHECK_FAILED' 76
+[ "$privilege_check" = 't' ] \
+    || fail 'database_role_check=LEAST_PRIVILEGE_FAILED' 76
+
+if ! cleanup; then
+    fail 'database_role_check=CLEANUP_FAILED' 77
+fi
+database_created=0
+roles_created=0
+trap - EXIT HUP INT TERM
+printf '%s\n' 'database_role_check=ok'


