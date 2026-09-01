## `fix(ops): align restored travel probes`

diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
index bfbdbbc..86b7d67 100644
--- a/docs/OPERATIONS-RUNBOOK.md
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -68,23 +68,28 @@ deferred-constraint state. A future migration therefore fails closed until this
 contract and its tests are reviewed together.
 
 The archive contains the country and passport-policy canonical rows, source
-configuration and rights evidence, fetch/artifact/parse receipts, typed entry
-and travel-warning revisions, review decisions, immutable publications,
-current pointers, audit events, and the Django schema/authentication metadata
-needed to restore their foreign keys. It cannot include an environment file or
-an API key value because those are not database columns. Source artifacts hold
-only hashes and byte counts; raw response bodies are not a database field.
+configuration and rights evidence, fetch/artifact/parse receipts and ordered
+aviation parse inputs, typed entry and travel-warning revisions, airports,
+flight schedule and reviewed-duration evidence, candidate seals, review
+decisions, immutable publications, all three pointer families, audit events,
+and the Django schema/authentication metadata needed to restore their foreign
+keys. It cannot include an environment file or an API key value because those
+are not database columns. Source artifacts and reviewed-duration receipts hold
+only hashes, byte counts and typed review evidence; raw response bodies are not
+a database field.
 
 `django_session` and `django_admin_log` schemas are included so the migrated
 schema remains complete, but their rows are always excluded. Their sequence
 definitions and next-value state remain part of the schema archive so inserts
 after restore cannot collide; do not describe the archive as erasing historical
-sequence state. The approved schema has no trip destination, departure date,
-return date, purpose, raw response body, credential, or key-value column. The
-only names containing `raw` are the exact rights-policy metadata fields
-`raw_body_storage_allowed` and `raw_retention_seconds`; they do not contain a
-body. The preflight rejects any other matching storage shape. PostgreSQL large
-objects are also rejected so they cannot become an unreviewed side channel.
+sequence state. The approved schema has no user-input `departure_at` or
+`return_by` column, raw response body, credential, or key-value column.
+Destination-airport references are durable reviewed flight evidence, not user
+search input. The only names containing `raw` are the exact rights-policy
+metadata fields `raw_body_storage_allowed` and `raw_retention_seconds`; they do
+not contain a body. The preflight rejects any other matching storage shape.
+PostgreSQL large objects are also rejected so they cannot become an unreviewed
+side channel.
 
 Use libpq's non-interactive credential mechanism. For example, point
 `PGPASSFILE` at a mode-0600 file supplied outside the repository. Never put a
@@ -108,7 +113,8 @@ only `backup_result=ok`. The directory is mode 0700 and contains only:
 
 - `database.dump`: PostgreSQL custom-format archive, mode 0600.
 - `integrity.manifest`: archive SHA-256 plus deterministic counts, row-set
-  SHA-256 values, and entry/travel-warning pointer identity, mode 0600.
+  SHA-256 values, six country-scoped entry pointers, six country-scoped
+  travel-warning pointers and the flight-schedule pointer, mode 0600.
 
 The command calculates the database integrity snapshot both before and after
 `pg_dump`; the snapshot includes every protected row set, atomic pointers,
@@ -222,22 +228,28 @@ matched by an approved SHA-256 and intentionally omitted from the archive.
 
 After a successful database verification, start the local production command
 against only the restored disposable database. Do not enable request or SQL
-logging and do not retain the response body. Make a credential-free `GET` of
-the queryless `/results/` route and verify all of the following:
-
-1. HTTP status is 200 and the response has `Cache-Control: no-store`.
-2. The document contains `id="entry-card"` and `id="warning-card"` exactly once.
-3. Each card has a non-colour `data-state` marker. A restored publication is
-   `ready` or `stale`; it must not silently become `empty` or `unavailable`.
-4. Each published card renders its `publication revision` generation and its
-   source attribution, while no verdict such as `ALLOWED` or `DENIED` appears.
-5. The displayed generations correspond to the restored entry and warning
-   pointer rows recorded by the integrity manifest.
-
-This check proves that the restored pointers cross the server-rendered boundary;
-the database hash comparison alone does not prove that the web process can read
-them. Record only fixed pass/fail markers, never the HTML, a source URL, a
-database credential, or trip form values.
+logging and do not retain response bodies. The probe derives a bounded future
+seven-day search from the restored, already-published schedule; it performs no
+provider request. Verify all of the following through real HTTPS:
+
+1. `GET /` returns 200 with `Cache-Control: no-store`, both labelled datetime
+   inputs and only the hardened CSRF cookie.
+2. Queryless `GET /results/` returns 303 with `Location: /` and no cookie.
+3. A same-origin, CSRF-protected `POST /` with `departure_at` and `return_by`
+   returns the result in the same 200 response without a redirect or query.
+4. The result contains one `id="recommendations"` root and a ranked destination
+   whose flight, entry and travel-warning modules are `ready` or `stale`.
+5. The flight revision and source attribution match the restored flight pointer;
+   the selected country's entry and travel-warning generations and attribution
+   match their two restored country-scoped pointers. No verdict such as
+   `ALLOWED` or `DENIED` appears.
+6. Neither input reaches a URL, cookie, session row or runtime log. The current
+   HTML response may display the submitted search window as designed.
+
+This check proves that all three restored pointer families cross the
+server-rendered search boundary; the database hash comparison alone does not
+prove that the web process can read them. Record only fixed pass/fail markers,
+never the HTML, a source URL, a database credential, or trip form values.
 
 ## Rehearsal receipt
 
@@ -269,8 +281,9 @@ The wrapper first requires a clean worktree whose `HEAD` exactly matches
 `--release-sha`. After restore it starts the pinned `.venv` Gunicorn process
 with `runtime/gunicorn.conf.py`, a private pre-bound loopback socket and a
 single-use locally generated TLS certificate. It validates the certificate and
-hostname, then probes `/healthz`, `/readyz`, `/releasez` and `/results/` through
-real HTTPS before terminating and reaping the complete runtime process group.
+hostname, then probes `/healthz`, `/readyz`, `/releasez`, `GET /`, the
+`GET /results/` redirect and the CSRF-protected `POST /` result through real
+HTTPS before terminating and reaping the complete runtime process group.
 `WRITERS_QUIESCED` is an operator assertion, not an automatic lock; give it only
 after all source, reviewer and Admin writers are actually paused. It generates
 a single-use restore credential in memory, keeps libpq material in a mode-0600
@@ -294,9 +307,11 @@ For each release SHA, record these non-sensitive fields in the release evidence:
 - `backup_result=ok`, `restore_result=ok`, and the UTC rehearsal timestamp;
 - `writers_quiesced=confirmed` for the full accepted backup window;
 - `session_rows=0`, `admin_log_rows=0`, `integrity_manifest=match`, and
-  `publication_pointers=match`;
-- `ssr_results_status=200`, `entry_marker=match`,
-  `travel_warning_marker=match`, `source_attribution=match`, and
+  `flight_pointer=match`, `entry_pointer=match`, and
+  `travel_warning_pointer=match`;
+- `ssr_search_status=200`, `results_redirect_status=303`,
+  `flight_marker=match`, `entry_marker=match`, `travel_warning_marker=match`,
+  `source_attribution=match`, and
   `restored_wsgi_tls=match`, `release_identity=match`,
   `runtime_cleanup=match`, and `cleanup=match`;
 - an external exact-name check showing zero remaining disposable database,
diff --git a/operations/tests/test_postgresql_backup_restore.py b/operations/tests/test_postgresql_backup_restore.py
index 0cc61a7..aeabf5c 100644
--- a/operations/tests/test_postgresql_backup_restore.py
+++ b/operations/tests/test_postgresql_backup_restore.py
@@ -393,8 +393,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         for required in (
             '"$script_dir/backup-postgresql"',
             '"$script_dir/restore-postgresql"',
-            "PublishedEntryFacts",
-            "PublishedTravelWarning",
+            "PublishedFlightSchedule",
+            "FlightSchedule",
             "subprocess.Popen(",
             'str(project_dir / "runtime" / "gunicorn.conf.py")',
             '"travel_readiness.wsgi:application"',
@@ -404,15 +404,23 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
             '"/healthz"',
             '"/readyz"',
             '"/releasez"',
+            'context, port, "GET", "/"',
             '"/results/"',
-            "document.count('id=\"entry-card\"') == 1",
-            "document.count('id=\"warning-card\"') == 1",
+            '"POST",',
+            "document.count('id=\"recommendations\"') == 1",
+            "document.count('id=\"destination-1-entry\"') == 1",
+            "document.count('id=\"destination-1-warning\"') == 1",
             "os.killpg(process_group, signal.SIGTERM)",
             "os.killpg(process_group, signal.SIGKILL)",
             "DROP DATABASE",
             "writers_quiesced=confirmed",
             "integrity_manifest=match",
-            "publication_pointers=match",
+            "flight_pointer=match",
+            "entry_pointer=match",
+            "travel_warning_pointer=match",
+            "ssr_search_status=200",
+            "results_redirect_status=303",
+            "flight_marker=match",
             "entry_marker=match",
             "travel_warning_marker=match",
             "source_attribution=match",
@@ -457,6 +465,8 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         self.assertNotIn("Client()", script)
         self.assertNotIn("run-production", script)
         self.assertNotIn(".env.local", lower)
+        self.assertEqual(lower.count("data_go_kr_service_key"), 2)
+        self.assertIn("unset data_go_kr_service_key", lower)
         self.assertEqual(lower.count("mofa_travel_alarm_service_key"), 2)
         self.assertIn("unset mofa_travel_alarm_service_key", lower)
         self.assertNotIn("set -x", lower)
@@ -517,6 +527,73 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
         self.assertNotIn("HOME", runtime)
         self.assertNotIn("PYTHONPATH", runtime)
 
+    def test_restored_wsgi_probe_accepts_only_the_hardened_csrf_cookie(self):
+        namespace = self.restored_wsgi_probe_namespace()
+        cookie_value = "a" * 32
+        header = namespace["_csrf_cookie_header"](
+            (
+                "csrftoken="
+                + cookie_value
+                + "; HttpOnly; Path=/; SameSite=Strict; Secure",
+            )
+        )
+        self.assertEqual(header, f"csrftoken={cookie_value}")
+        for rejected in (
+            ("sessionid=" + cookie_value + "; Path=/; Secure",),
+            ("csrftoken=" + cookie_value + "; Path=/; SameSite=Lax; Secure",),
+            (
+                "csrftoken="
+                + cookie_value
+                + "; Path=/; SameSite=Strict; Secure",
+            ),
+        ):
+            with self.assertRaises(RuntimeError):
+                namespace["_csrf_cookie_header"](rejected)
+
+    def test_restored_wsgi_probe_matches_ranked_three_pointer_result(self):
+        namespace = self.restored_wsgi_probe_namespace()
+        expectations = {
+            "flight": {
+                "generation": 7,
+                "attribution": "flight attribution",
+                "locator": "https://example.invalid/flight",
+            },
+            "destination": {"city_name": "도쿄", "airport_code": "NRT"},
+            "entry": {
+                "generation": 8,
+                "owner": "entry owner",
+                "attribution": "entry attribution",
+                "locator": "https://example.invalid/entry",
+            },
+            "warning": {
+                "generation": 9,
+                "owner": "warning owner",
+                "attribution": "warning attribution",
+                "locator": "https://example.invalid/warning",
+            },
+        }
+        document = """
+        <section id="recommendations" data-state="ready">
+          <p>운항 자료 리비전 7</p>
+          <p>flight attribution</p>
+          <a href="https://example.invalid/flight">flight</a>
+          <article id="destination-1">도쿄 NRT</article>
+          <article id="destination-1-entry" data-module="entry" data-state="ready">
+            8 entry owner entry attribution
+            <a href="https://example.invalid/entry">entry</a>
+          </article>
+          <article id="destination-1-warning" data-module="warning" data-state="stale">
+            9 warning owner warning attribution
+            <a href="https://example.invalid/warning">warning</a>
+          </article>
+        </section>
+        """.encode()
+        namespace["_validate_results"](
+            document,
+            expectations,
+            ("database-secret-not-present",),
+        )
+
     def test_restored_wsgi_runtime_cleanup_kills_stubborn_process_group(self):
         namespace = self.restored_wsgi_probe_namespace()
         grandchild = (
@@ -1215,9 +1292,11 @@ class PostgreSQLBackupRestoreContractTests(unittest.TestCase):
     def test_runbook_requires_real_restore_and_ssr_rehearsal(self):
         runbook = self.runbook.read_text(encoding="utf-8")
         self.assertIn("do not count as a restore\nrehearsal", runbook)
-        self.assertIn('id="entry-card"', runbook)
-        self.assertIn('id="warning-card"', runbook)
-        self.assertIn("`/results/`", runbook)
+        self.assertIn('id="recommendations"', runbook)
+        self.assertIn("`GET /results/` returns 303", runbook)
+        self.assertIn("CSRF-protected `POST /`", runbook)
+        self.assertIn("flight pointer", runbook)
+        self.assertIn("country-scoped pointers", runbook)
         self.assertIn("session_rows=0", runbook)
         self.assertIn("RPO 24 hours", runbook)
         self.assertIn("RTO 4 hours", runbook)
diff --git a/scripts/check-backup-restore b/scripts/check-backup-restore
index cccb2a5..d3399e6 100755
--- a/scripts/check-backup-restore
+++ b/scripts/check-backup-restore
@@ -6,7 +6,8 @@ umask 077
 LC_ALL=C
 export LC_ALL
 unset PGDATABASE PGUSER PGHOST PGPORT PGSERVICE PGSERVICEFILE PGOPTIONS PGPASSWORD PGPASSFILE
-unset DATA_GO_KR_SERVICE_KEY MOFA_TRAVEL_ALARM_SERVICE_KEY
+unset DATA_GO_KR_SERVICE_KEY
+unset MOFA_TRAVEL_ALARM_SERVICE_KEY
 unset TRAVEL_READINESS_SECRET_KEY TRAVEL_READINESS_DB_PASSWORD
 
 usage() {
@@ -311,8 +312,10 @@ TRAVEL_READINESS_RELEASE_SHA="$release_sha" \
     "$python_bin" -I -B - "$project_dir" >/dev/null 2>&1 <<'PY'
 # RESTORED_WSGI_PROBE_START
 import errno
+from datetime import datetime, timedelta
 import html
 import http.client
+from http.cookies import SimpleCookie
 import os
 from pathlib import Path
 import re
@@ -323,6 +326,8 @@ import subprocess
 import sys
 import threading
 import time
+from urllib.parse import urlencode
+from zoneinfo import ZoneInfo
 
 
 _LOG_LIMIT = 64 * 1024
@@ -456,7 +461,7 @@ def _terminate_process_group(process, term_timeout=5.0, kill_timeout=5.0):
     return clean and process.returncode is not None and not _process_group_exists(process_group)
 
 
-def _https_get(context, port, path):
+def _https_request(context, port, method, path, *, headers=None, body=None):
     connection = http.client.HTTPSConnection(
         "127.0.0.1",
         port,
@@ -464,25 +469,37 @@ def _https_get(context, port, path):
         context=context,
     )
     try:
+        request_headers = {
+            "Accept": "text/html,application/json,text/plain",
+            "Connection": "close",
+            "User-Agent": "travel-readiness-restore-probe",
+        }
+        if headers:
+            request_headers.update(headers)
         connection.request(
-            "GET",
+            method,
             path,
-            headers={
-                "Accept": "text/html,application/json,text/plain",
-                "Connection": "close",
-                "User-Agent": "travel-readiness-restore-probe",
-            },
+            body=body,
+            headers=request_headers,
         )
         response = connection.getresponse()
         body = response.read(_BODY_LIMIT + 1)
         _require(len(body) <= _BODY_LIMIT)
         headers = {name.lower(): value for name, value in response.getheaders()}
-        _require(not response.headers.get_all("Set-Cookie", []))
-        return response.status, headers, body
+        cookies = tuple(response.headers.get_all("Set-Cookie", []))
+        return response.status, headers, body, cookies
     finally:
         connection.close()
 
 
+def _https_get(context, port, path):
+    status, headers, body, cookies = _https_request(
+        context, port, "GET", path
+    )
+    _require(not cookies)
+    return status, headers, body
+
+
 def _exact_response(context, port, path, expected_status, expected_type, expected_body):
     status, headers, body = _https_get(context, port, path)
     _require(status == expected_status)
@@ -491,6 +508,20 @@ def _exact_response(context, port, path, expected_status, expected_type, expecte
     _require(body == expected_body)
 
 
+def _csrf_cookie_header(raw_cookies):
+    _require(len(raw_cookies) == 1)
+    cookie = SimpleCookie()
+    cookie.load(raw_cookies[0])
+    _require(set(cookie) == {"csrftoken"})
+    morsel = cookie["csrftoken"]
+    _require(re.fullmatch(r"[A-Za-z0-9]{32}", morsel.value) is not None)
+    _require(morsel["path"] == "/")
+    _require(morsel["samesite"] == "Strict")
+    _require(bool(morsel["secure"]) and bool(morsel["httponly"]))
+    _require(not morsel["domain"])
+    return f"csrftoken={morsel.value}"
+
+
 def _interrupt(signum, _frame):
     global _INTERRUPTED_SIGNAL
     if _INTERRUPTED_SIGNAL == 0:
@@ -541,41 +572,91 @@ def _load_publication_expectations(project_dir, runtime):
 
     django.setup()
     from django.db import connections
-    from reviews.models import PublishedEntryFacts, PublishedTravelWarning
+    from public_web.views import search_travel_opportunities
+    from travel_windows.models import FlightSchedule, PublishedFlightSchedule
 
-    entry_pointer = PublishedEntryFacts.objects.select_related(
-        "current_publication__entry_fact_revision__country"
+    pointer = PublishedFlightSchedule.objects.select_related(
+        "current_publication__schedule_revision"
     ).get()
-    warning_pointer = PublishedTravelWarning.objects.select_related(
-        "current_publication__travel_warning_revision__country"
-    ).get()
-    entry = entry_pointer.current_publication
-    warning = warning_pointer.current_publication
-    _require(entry is not None and warning is not None)
-    _require(entry_pointer.version == entry.generation)
-    _require(warning_pointer.version == warning.generation)
-    _require(entry.entry_fact_revision.ordinary_passport_period_text == "90일")
-    _require(entry.entry_fact_revision.country.iso_alpha2 == "JP")
-    _require(warning.travel_warning_revision.source_alarm_level_code == "3")
-    _require(warning.travel_warning_revision.country.iso_alpha2 == "JP")
+    publication = pointer.current_publication
+    _require(publication is not None)
+    _require(pointer.version == publication.generation)
+    schedules = tuple(
+        FlightSchedule.objects.filter(
+            revision=publication.schedule_revision,
+            direction=FlightSchedule.Direction.OUTBOUND,
+        ).values_list("valid_from", "valid_until", "weekday_mask")
+    )
+    _require(schedules)
+    seoul = ZoneInfo("Asia/Seoul")
+    tomorrow = datetime.now(tz=seoul).date() + timedelta(days=1)
+    search_dates = set()
+    for valid_from, valid_until, weekday_mask in schedules:
+        first = max(valid_from, tomorrow)
+        last = min(valid_until, tomorrow + timedelta(days=366))
+        current = first
+        while current <= last:
+            if weekday_mask[current.weekday()] == "1":
+                search_dates.add(current)
+            current += timedelta(days=1)
+
+    selected = None
+    for search_date in sorted(search_dates):
+        departure_at = datetime.combine(
+            search_date, datetime.min.time(), tzinfo=seoul
+        )
+        return_by = departure_at + timedelta(days=7)
+        context = search_travel_opportunities(
+            departure_at=departure_at,
+            return_by=return_by,
+        )
+        if context.get("opportunities"):
+            selected = (departure_at, return_by, context)
+            break
+    _require(selected is not None)
+    departure_at, return_by, context = selected
+    opportunity = context["opportunities"][0]
+    entry = opportunity["entry_card"]
+    warning = opportunity["warning_card"]
+    _require(context["flight_publication_revision"] == str(pointer.version))
+    for card in (entry, warning):
+        _require(card.get("state") in {"ready", "stale"})
+        _require(card.get("has_publication") is True)
+        _require(type(card.get("generation")) is int and card["generation"] > 0)
+        _require(
+            all(
+                type(card.get(name)) is str and card[name]
+                for name in ("source_owner", "attribution", "source_locator")
+            )
+        )
+        _require(card["source_locator"].startswith("https://"))
     expectations = {
+        "search": {
+            "departure_at": departure_at.strftime("%Y-%m-%dT%H:%M"),
+            "return_by": return_by.strftime("%Y-%m-%dT%H:%M"),
+        },
+        "flight": {
+            "generation": pointer.version,
+            "attribution": publication.source_attribution,
+            "locator": publication.source_locator,
+        },
+        "destination": {
+            "city_name": opportunity["destination"]["city_name"],
+            "airport_code": opportunity["destination"]["airport_code"],
+        },
         "entry": {
-            "generation": entry.generation,
-            "owner": entry.source_owner_snapshot,
-            "attribution": entry.attribution_text_snapshot,
-            "locator": entry.source_locator_snapshot,
+            "generation": entry["generation"],
+            "owner": entry["source_owner"],
+            "attribution": entry["attribution"],
+            "locator": entry["source_locator"],
         },
         "warning": {
-            "generation": warning.generation,
-            "owner": warning.source_owner_snapshot,
-            "attribution": warning.attribution_text_snapshot,
-            "locator": warning.source_locator_snapshot,
+            "generation": warning["generation"],
+            "owner": warning["source_owner"],
+            "attribution": warning["attribution"],
+            "locator": warning["source_locator"],
         },
     }
-    for expectation in expectations.values():
-        _require(type(expectation["generation"]) is int and expectation["generation"] > 0)
-        _require(all(type(expectation[name]) is str and expectation[name] for name in ("owner", "attribution", "locator")))
-        _require(expectation["locator"].startswith("https://"))
     connections.close_all()
     return expectations
 
@@ -584,10 +665,29 @@ def _validate_results(body, expectations, sensitive_values):
     for sensitive in sensitive_values:
         _require(sensitive.encode("utf-8") not in body)
     document = body.decode("utf-8", "strict")
-    _require(document.count('id="entry-card"') == 1)
-    _require(document.count('id="warning-card"') == 1)
-    entry_start = document.index('id="entry-card"')
-    warning_start = document.index('id="warning-card"')
+    _require(document.count('id="recommendations"') == 1)
+    recommendations = document.index('id="recommendations"')
+    recommendations_tag = document[recommendations:].split(">", 1)[0]
+    _require(
+        re.search(r'data-state="(?:ready|stale)"', recommendations_tag)
+        is not None
+    )
+    _require(document.count('id="destination-1"') == 1)
+    _require(document.count('id="destination-1-entry"') == 1)
+    _require(document.count('id="destination-1-warning"') == 1)
+    _require(
+        html.escape(expectations["destination"]["city_name"], quote=True)
+        in document
+    )
+    _require(expectations["destination"]["airport_code"] in document)
+    flight = expectations["flight"]
+    _require(f'운항 자료 리비전 {flight["generation"]}' in document)
+    _require(html.escape(flight["attribution"], quote=True) in document)
+    _require(
+        f'href="{html.escape(flight["locator"], quote=True)}"' in document
+    )
+    entry_start = document.index('id="destination-1-entry"')
+    warning_start = document.index('id="destination-1-warning"')
     _require(entry_start < warning_start)
     cards = {
         "entry": document[entry_start:warning_start],
@@ -596,8 +696,9 @@ def _validate_results(body, expectations, sensitive_values):
     for name, card in cards.items():
         opening_tag = card.split(">", 1)[0]
         _require(re.search(r'data-state="(?:ready|stale)"', opening_tag) is not None)
+        _require(f'data-module="{name}"' in opening_tag)
         expectation = expectations[name]
-        _require(f'generation {expectation["generation"]}' in card)
+        _require(str(expectation["generation"]) in card)
         _require(html.escape(expectation["owner"], quote=True) in card)
         _require(html.escape(expectation["attribution"], quote=True) in card)
         escaped_locator = html.escape(expectation["locator"], quote=True)
@@ -734,10 +835,62 @@ def main():
             ('{"release_sha":"' + required["TRAVEL_READINESS_RELEASE_SHA"] + '"}').encode("ascii"),
         )
         _raise_if_interrupted()
-        status, headers, body = _https_get(context, port, "/results/")
+        status, headers, body, cookies = _https_request(
+            context, port, "GET", "/"
+        )
+        _require(status == 200)
+        _require(headers.get("cache-control") == "no-store")
+        _require(headers.get("content-type") == "text/html; charset=utf-8")
+        landing = body.decode("utf-8", "strict")
+        _require('name="departure_at"' in landing)
+        _require('name="return_by"' in landing)
+        csrf_match = re.search(
+            r'name="csrfmiddlewaretoken" value="([A-Za-z0-9]{64})"',
+            landing,
+        )
+        _require(csrf_match is not None)
+        cookie_header = _csrf_cookie_header(cookies)
+        for value in (*sensitive_values, *expectations["search"].values()):
+            _require(value not in cookie_header)
+        del body, landing
+        _raise_if_interrupted()
+        status, headers, body, cookies = _https_request(
+            context, port, "GET", "/results/"
+        )
+        _require(status == 303)
+        _require(headers.get("location") == "/")
+        _require(headers.get("cache-control") == "no-store")
+        _require(not cookies)
+        _require(not body)
+        _raise_if_interrupted()
+        form_values = {
+            **expectations["search"],
+            "csrfmiddlewaretoken": csrf_match.group(1),
+        }
+        encoded_form = urlencode(form_values).encode("ascii")
+        origin = f"https://127.0.0.1:{port}"
+        status, headers, body, cookies = _https_request(
+            context,
+            port,
+            "POST",
+            "/",
+            headers={
+                "Content-Type": "application/x-www-form-urlencoded",
+                "Cookie": cookie_header,
+                "Origin": origin,
+                "Referer": origin + "/",
+            },
+            body=encoded_form,
+        )
         _require(status == 200)
         _require(headers.get("cache-control") == "no-store")
         _require(headers.get("content-type") == "text/html; charset=utf-8")
+        _require("location" not in headers)
+        if cookies:
+            renewed_cookie = _csrf_cookie_header(cookies)
+            _require(renewed_cookie == cookie_header)
+        for value in (*sensitive_values, *expectations["search"].values()):
+            _require(all(value not in cookie for cookie in cookies))
         _validate_results(body, expectations, sensitive_values)
         del body
         _raise_if_interrupted()
@@ -760,7 +913,7 @@ def main():
             signal.signal(signum, handler)
         combined_log = bytes(stdout_log) + bytes(stderr_log)
         log_clean = not stream_state["overflow"] and not stream_state["read_failed"]
-        for sensitive in sensitive_values:
+        for sensitive in (*sensitive_values, *expectations["search"].values()):
             if sensitive.encode("utf-8") in combined_log:
                 log_clean = False
         if not runtime_clean or not threads_clean or not log_clean:
@@ -794,6 +947,6 @@ rehearsal_finished=$(date +%s)
 unset admin_password backup_password backup_pgpass_password
 unset restore_password restore_pgpass_password django_secret
 
-printf 'backup_restore_check=ok writers_quiesced=confirmed backup_seconds=%s restore_seconds=%s rehearsal_seconds=%s backup_bytes=%s session_rows=0 admin_log_rows=0 integrity_manifest=match publication_pointers=match ssr_results_status=200 entry_marker=match travel_warning_marker=match source_attribution=match restored_wsgi_tls=match release_identity=match runtime_cleanup=match cleanup=match\n' \
+printf 'backup_restore_check=ok writers_quiesced=confirmed backup_seconds=%s restore_seconds=%s rehearsal_seconds=%s backup_bytes=%s session_rows=0 admin_log_rows=0 integrity_manifest=match flight_pointer=match entry_pointer=match travel_warning_pointer=match ssr_search_status=200 results_redirect_status=303 flight_marker=match entry_marker=match travel_warning_marker=match source_attribution=match restored_wsgi_tls=match release_identity=match runtime_cleanup=match cleanup=match\n' \
     "$((backup_finished - backup_started))" "$((restore_finished - restore_started))" \
     "$((rehearsal_finished - rehearsal_started))" "$backup_bytes"


