## `feat(source): publish supported-country travel evidence`

diff --git a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
index 0e6b488..96485ca 100644
--- a/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
+++ b/docs/TRAVEL-OPPORTUNITY-CONTRACT.md
@@ -32,7 +32,9 @@
 - 첫 후보 도시는 도쿄, 오사카, 후쿠오카, 삿포로, 오키나와, 타이베이, 홍콩,
   마카오, 하노이, 다낭, 호찌민, 방콕입니다.
 - 직항이며 current flight publication, 검수된 양방향 비행시간, entry publication과
-  warning publication이 모두 있는 도시만 추천합니다.
+  warning publication이 모두 있고 그 국가에 승인된 정확한 source/parser contract인
+  도시만 추천합니다. 일본 기존 JP 전용 계약은 계속 인정하고, 일본 외 5개국은
+  2026-08-31 승인된 supported-6 sibling 계약만 인정합니다.
 - 코드셰어는 master flight를 기준으로 중복 제거합니다.
 - 예상 현지 도착은 ICN 출발 instant에 검수된 outbound duration을 더해 계산합니다.
 - 예상 귀국편 출발은 ICN 도착 instant에서 검수된 inbound duration을 빼 계산합니다.
@@ -55,14 +57,28 @@ public result context는 `search_window`, `flight_state`, `opportunities`를 제
 ## source와 권리
 
 - 인천국제공항공사 정기운항편 source는 시즌·유효일·요일·방향·ICN event time을
-  current schedule 사실로 제공합니다.
-- 인천국제공항공사 취항도시 source는 공항 code와 국가·도시 명칭을 제공합니다.
+  current schedule 사실로 제공합니다. 전체 응답을 strict 검증한 뒤 curated 13개
+  공항만 typed row로 투영하며, 실제 시즌에 포함된 각 공항은 출발·도착 방향이 모두
+  있어야 합니다.
+- `StatusOfSrvDestinations/getServiceDestinationInfo` 취항도시 전체 응답에서 curated
+  13개 IATA와 국문 국가·도시 명칭이 정확한 subset인지 검증합니다. extra 도시는
+  허용하지만 중복 IATA나 curated identity 불일치는 게시를 닫습니다.
+- `PaxFltSched/getPaxFltSchedArrivals` complete pages를 보조 증거로 수집하고, curated
+  inbound 노선의 시즌·요일·ICN 도착시각이 관광플랫폼 운항표와 일치하지 않으면
+  게시하지 않습니다.
 - 해양수산부 수출입 물류 플랫폼 항공 스케줄 파일의 비행시간은 검수된 duration
-  reference로만 사용하며 current 운항 사실로 표시하지 않습니다.
+  reference로만 사용하며 current 운항 사실로 표시하지 않습니다. 각 typed CSV row의
+  reference locator는 정확히 `https://www.data.go.kr/data/15151728/fileData.do`여야
+  합니다.
+- 공개 운항 source 링크는 인증키 없는 API endpoint가 아니라 사람이 읽을 수 있는
+  `https://www.data.go.kr/data/15143060/openapi.do` 문서 페이지를 사용합니다.
 - source rights, parser, typed revision, review, publication, atomic pointer, rollback과
   freshness는 기존 공통 lifecycle을 통과합니다.
 - `DATA_GO_KR_SERVICE_KEY`를 표준 secret reference로 사용하고 기존
   `MOFA_TRAVEL_ALARM_SERVICE_KEY`는 한시적 호환 fallback으로만 허용합니다.
+- 2026-08-31 제품 계약은 JP baseline evidence를 재작성하지 않고 계승하면서
+  JP/TW/HK/MO/VN/TH의 입국요건·여행경보 typed field scope를 sibling source로
+  append-only 승인합니다. 비JP publication에 JP 전용 scope를 재사용하지 않습니다.
 - live schema probe는 management command에서 endpoint별 최소 한 번만 수행하며 raw
   response를 fixture, file, log 또는 audit에 저장하지 않습니다.
 
diff --git a/entry_requirements/ingestion.py b/entry_requirements/ingestion.py
index 5d7b78e..7d2ed24 100644
--- a/entry_requirements/ingestion.py
+++ b/entry_requirements/ingestion.py
@@ -16,7 +16,7 @@ from typing import Callable
 from django.db import connection, transaction
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import SUPPORTED_COUNTRY_IDS, Country
 from sources.models import (
     FetchAttempt,
     ParseRun,
@@ -50,6 +50,7 @@ from .models import (
     PassportPolicy,
 )
 from .parser import (
+    ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256,
     ENTRY_SCHEMA_FINGERPRINT_SHA256,
     EntryParseResult,
     EntryProjection,
@@ -57,17 +58,29 @@ from .parser import (
 )
 
 
-ENTRY_SOURCE_CODE = "MOFA_ENTRY_CSV"
-ENTRY_SOURCE_REVISION = "rights-v1"
+LEGACY_ENTRY_SOURCE_CODE = "MOFA_ENTRY_CSV"
+LEGACY_ENTRY_SOURCE_REVISION = "rights-v1"
+LEGACY_ENTRY_FIELD_SCOPE = "JP_ORDINARY_TEXT_V1"
+LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256 = (
+    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+)
+MULTI_COUNTRY_ENTRY_SOURCE_CODE = "MOFA_ENTRY_CSV_TRAVEL6"
+MULTI_COUNTRY_ENTRY_SOURCE_REVISION = "travel6-v1"
+MULTI_COUNTRY_ENTRY_FIELD_SCOPE = "SUPPORTED_6_ORDINARY_TEXT_V1"
+MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256 = (
+    ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256
+)
+ENTRY_SOURCE_CODE = MULTI_COUNTRY_ENTRY_SOURCE_CODE
+ENTRY_SOURCE_REVISION = MULTI_COUNTRY_ENTRY_SOURCE_REVISION
 ENTRY_SOURCE_MODULE = SourceConfiguration.Module.ENTRY
 ENTRY_SOURCE_OWNER = "대한민국 외교부 정보화담당관실"
-ENTRY_FIELD_SCOPE = "JP_ORDINARY_TEXT_V1"
+ENTRY_FIELD_SCOPE = MULTI_COUNTRY_ENTRY_FIELD_SCOPE
 ENTRY_ATTRIBUTION = "외교부|공공데이터포털"
 ENTRY_CONTRACT_FINGERPRINT_SHA256 = (
-    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+    MULTI_COUNTRY_ENTRY_CONTRACT_FINGERPRINT_SHA256
 )
 ENTRY_DECIDED_BY = "PROJECT_OWNER_REQUEST"
-ENTRY_DECISION_BASIS = "USER_DIRECTIVE_20260830"
+ENTRY_DECISION_BASIS = "USER_TRAVEL6_SCOPE_20260831"
 ENTRY_PARSER_NAME = ParseRun.ParserName.MOFA_ENTRY_CSV
 ENTRY_PARSER_VERSION = ParseRun.ParserVersion.V1
 ENTRY_INGESTION_LOCK_NAMESPACE = 1_414_678_614
@@ -452,9 +465,14 @@ def _close_attempt(
 def _safe_parse(
     body: bytes,
     parser: Callable[[bytes], EntryParseResult],
+    country_iso2: str,
 ) -> EntryParseResult:
     try:
-        result = parser(body)
+        result = (
+            parser(body, country_iso2=country_iso2)
+            if parser is parse_entry_csv
+            else parser(body)
+        )
     except Exception:
         result = None
     if not isinstance(result, EntryParseResult):
@@ -471,7 +489,7 @@ def _safe_parse(
             and result.observed_schema_fingerprint_sha256
             == ENTRY_SCHEMA_FINGERPRINT_SHA256
             and isinstance(projection, EntryProjection)
-            and projection.country_iso2 == "JP"
+            and projection.country_iso2 == country_iso2
             and projection.passport_policy_code == PASSPORT_POLICY_CODE
             and projection.passport_policy_revision == PASSPORT_POLICY_REVISION
         ):
@@ -503,12 +521,15 @@ def _safe_parse(
     )
 
 
-def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
+def _consistent_replay_run(
+    artifact: SourceArtifact,
+    byte_count: int,
+) -> ParseRun | None:
     if (
         artifact.byte_count != byte_count
         or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
     ):
-        return False
+        return None
     runs = list(
         ParseRun.objects.select_for_update().filter(
             artifact=artifact,
@@ -517,9 +538,9 @@ def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
         )
     )
     if len(runs) != 1:
-        return False
+        return None
     run = runs[0]
-    return bool(
+    consistent = bool(
         run.outcome == ParseRun.Outcome.VALIDATED
         and run.parser_contract_fingerprint_sha256
         == ENTRY_CONTRACT_FINGERPRINT_SHA256
@@ -527,7 +548,39 @@ def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
         == ENTRY_SCHEMA_FINGERPRINT_SHA256
         and run.observed_schema_fingerprint_sha256
         == ENTRY_SCHEMA_FINGERPRINT_SHA256
-        and EntryFactRevision.objects.filter(parse_run=run).count() == 1
+        and EntryFactRevision.objects.filter(parse_run=run).exists()
+    )
+    return run if consistent else None
+
+
+def _create_entry_revision(
+    *,
+    source: SourceConfiguration,
+    rights: SourceRightsDecision,
+    parse_run: ParseRun,
+    first_observed_at,
+    projection: EntryProjection,
+    country_iso2: str,
+) -> None:
+    current_source, current_rights = _locked_approved_source()
+    if current_source.id != source.id or current_rights.id != rights.id:
+        raise _ClosedPersistenceFailure(EntryIngestionCode.SOURCE_GATE_FAILED)
+    country = Country.objects.get(
+        pk=SUPPORTED_COUNTRY_IDS[country_iso2],
+        iso_alpha2=country_iso2,
+    )
+    policy = PassportPolicy.objects.get(pk=PASSPORT_POLICY_ID)
+    EntryFactRevision.objects.create(
+        country=country,
+        passport_policy=policy,
+        parse_run=parse_run,
+        ordinary_passport_period_text=(
+            projection.ordinary_passport_period_text
+        ),
+        basis_text=projection.basis_text,
+        snapshot_date=projection.snapshot_date,
+        first_observed_at=first_observed_at,
+        typed_fingerprint_sha256=projection.typed_fingerprint_sha256,
     )
 
 
@@ -535,6 +588,7 @@ def _persist_success(
     attempt: FetchAttempt,
     verified: _VerifiedTransportResult,
     parser: Callable[[bytes], EntryParseResult],
+    country_iso2: str,
 ) -> str:
     try:
         with transaction.atomic(durable=True):
@@ -570,11 +624,49 @@ def _persist_success(
                 .first()
             )
             if existing is not None:
-                if not _replay_is_consistent(existing, verified.byte_count):
+                replay_run = _consistent_replay_run(
+                    existing,
+                    verified.byte_count,
+                )
+                if replay_run is None:
+                    raise _ClosedPersistenceFailure(
+                        EntryIngestionCode.EVIDENCE_CONFLICT
+                    )
+                if EntryFactRevision.objects.filter(
+                    parse_run=replay_run,
+                    country__iso_alpha2=country_iso2,
+                    passport_policy_id=PASSPORT_POLICY_ID,
+                ).exists():
+                    return EntryIngestionCode.REPLAY_OBSERVED
+                parse_result = _safe_parse(
+                    verified.body,
+                    parser,
+                    country_iso2,
+                )
+                projection = parse_result.projection
+                if (
+                    parse_result.outcome != ParseRun.Outcome.VALIDATED
+                    or projection is None
+                ):
+                    raise _ClosedPersistenceFailure(
+                        EntryIngestionCode.EVIDENCE_CONFLICT
+                    )
+                anchor = FetchAttempt.objects.select_for_update().get(
+                    pk=existing.first_successful_attempt_id
+                )
+                if anchor.completed_at is None:
                     raise _ClosedPersistenceFailure(
                         EntryIngestionCode.EVIDENCE_CONFLICT
                     )
-                return EntryIngestionCode.REPLAY_OBSERVED
+                _create_entry_revision(
+                    source=source,
+                    rights=rights,
+                    parse_run=replay_run,
+                    first_observed_at=anchor.completed_at,
+                    projection=projection,
+                    country_iso2=country_iso2,
+                )
+                return EntryIngestionCode.REVIEW_REQUIRED
 
             anchor = (
                 FetchAttempt.objects.select_for_update()
@@ -607,7 +699,11 @@ def _persist_success(
                     ENTRY_SCHEMA_FINGERPRINT_SHA256
                 ),
             )
-            parse_result = _safe_parse(verified.body, parser)
+            parse_result = _safe_parse(
+                verified.body,
+                parser,
+                country_iso2,
+            )
             completed_at = timezone.now()
             updated = ParseRun.objects.filter(
                 pk=parse_run.id,
@@ -652,28 +748,13 @@ def _persist_success(
                     EntryIngestionCode.PERSISTENCE_FAILED
                 )
 
-            # This second exact/latest check sits immediately before the typed
-            # insert.  Holding the source row lock also serializes rejection.
-            current_source, current_rights = _locked_approved_source()
-            if current_source.id != source.id or current_rights.id != rights.id:
-                raise _ClosedPersistenceFailure(
-                    EntryIngestionCode.SOURCE_GATE_FAILED
-                )
-            country = Country.objects.get(pk=JP_COUNTRY_ID)
-            policy = PassportPolicy.objects.get(pk=PASSPORT_POLICY_ID)
-            EntryFactRevision.objects.create(
-                country=country,
-                passport_policy=policy,
+            _create_entry_revision(
+                source=source,
+                rights=rights,
                 parse_run=parse_run,
-                ordinary_passport_period_text=(
-                    projection.ordinary_passport_period_text
-                ),
-                basis_text=projection.basis_text,
-                snapshot_date=projection.snapshot_date,
                 first_observed_at=anchor.completed_at,
-                typed_fingerprint_sha256=(
-                    projection.typed_fingerprint_sha256
-                ),
+                projection=projection,
+                country_iso2=country_iso2,
             )
             return EntryIngestionCode.REVIEW_REQUIRED
     except _ClosedPersistenceFailure:
@@ -686,6 +767,7 @@ def _persist_success(
 
 def _run_entry_snapshot_ingestion(
     *,
+    country_iso2: str,
     transport: Callable[..., SingleAttemptResult],
     parser: Callable[[bytes], EntryParseResult],
     retry_wait: Callable[[int], None],
@@ -781,7 +863,12 @@ def _run_entry_snapshot_ingestion(
 
             if verified.attempt_result == ATTEMPT_SUCCEEDED:
                 try:
-                    code = _persist_success(closed_attempt, verified, parser)
+                    code = _persist_success(
+                        closed_attempt,
+                        verified,
+                        parser,
+                        country_iso2,
+                    )
                 except _ClosedPersistenceFailure as failure:
                     code = failure.code
                 return EntryIngestionOutcome(code, attempt_count)
@@ -820,13 +907,21 @@ def _run_entry_snapshot_ingestion(
 
 def ingest_entry_snapshot(
     *,
+    country_iso2: str = "JP",
     transport: Callable[..., SingleAttemptResult] = fetch_entry_attachment,
     parser: Callable[[bytes], EntryParseResult] = parse_entry_csv,
     retry_wait: Callable[[int], None] = _bounded_retry_wait,
 ) -> EntryIngestionOutcome:
     """Run one redacted operation, including only its bounded permitted retries."""
 
+    if (
+        type(country_iso2) is not str
+        or country_iso2 not in SUPPORTED_COUNTRY_IDS
+    ):
+        return EntryIngestionOutcome(EntryIngestionCode.SOURCE_GATE_FAILED, 0)
+
     result = _run_entry_snapshot_ingestion(
+        country_iso2=country_iso2,
         transport=transport,
         parser=parser,
         retry_wait=retry_wait,
diff --git a/entry_requirements/migrations/0002_country_scoped_entry_facts.py b/entry_requirements/migrations/0002_country_scoped_entry_facts.py
new file mode 100644
index 0000000..67ca406
--- /dev/null
+++ b/entry_requirements/migrations/0002_country_scoped_entry_facts.py
@@ -0,0 +1,178 @@
+import uuid
+
+import django.db.models.deletion
+from django.db import migrations, models
+
+
+SUPPORTED_COUNTRY_IDS = (
+    uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9"),
+    uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"),
+    uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"),
+    uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"),
+    uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"),
+    uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"),
+)
+
+
+FUNCTION_REPLACEMENTS = (
+    (
+        """    IF NEW.country_id <> '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+       OR NEW.passport_policy_id <> 'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid THEN""",
+        """    IF NEW.country_id NOT IN (
+           '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid,
+           '3d374024-be31-5be3-99b3-fc28626b076a'::uuid,
+           '008d7e8f-412e-53ca-a5c6-d06a9fbafda8'::uuid,
+           '55d20bb0-9d97-5a53-9600-e8f102f38fe9'::uuid,
+           '17e47e71-07e3-57e6-8c72-e1f8b47e34df'::uuid,
+           '5438e3c3-df2b-593a-8f04-7e64e66219e7'::uuid
+       )
+       OR NEW.passport_policy_id <> 'f461f7a7-18f7-5b0d-9831-bfbd47b695e5'::uuid THEN""",
+    ),
+    (
+        """        ',"country_iso2":"JP"' ||""",
+        """        ',"country_iso2":' ||
+            to_jsonb((
+                SELECT iso_alpha2
+                  FROM countries_country
+                 WHERE id = NEW.country_id
+            ))::text ||""",
+    ),
+    (
+        """       OR parser_contract <>
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'""",
+        """       OR parser_contract <>
+          '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f'""",
+    ),
+    (
+        """       OR source_code <> 'MOFA_ENTRY_CSV'
+       OR source_revision <> 'rights-v1'""",
+        """       OR source_code <> 'MOFA_ENTRY_CSV_TRAVEL6'
+       OR source_revision <> 'travel6-v1'""",
+    ),
+    (
+        """       OR rights_field_scope <> 'JP_ORDINARY_TEXT_V1'""",
+        """       OR rights_field_scope <> 'SUPPORTED_6_ORDINARY_TEXT_V1'""",
+    ),
+    (
+        """       OR rights_contract <>
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021' THEN""",
+        """       OR rights_contract <>
+          '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f' THEN""",
+    ),
+)
+
+
+def _rewrite_guard(schema_editor, *, reverse=False):
+    function_name = "entry_requirements_guard_fact_revision"
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [function_name],
+        )
+        rows = cursor.fetchall()
+        if len(rows) != 1:
+            raise RuntimeError(f"expected one trigger function named {function_name}")
+        body = rows[0][0]
+        for forward_old, forward_new in FUNCTION_REPLACEMENTS:
+            old, new = (
+                (forward_new, forward_old)
+                if reverse
+                else (forward_old, forward_new)
+            )
+            if body.count(old) != 1:
+                raise RuntimeError(f"unexpected {function_name} trigger definition")
+            body = body.replace(old, new)
+        quoted_name = schema_editor.quote_name(function_name)
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $entry_country_scope$
+            {body}
+            $entry_country_scope$
+            """
+        )
+
+
+def generalize_entry_guard(apps, schema_editor):
+    _rewrite_guard(schema_editor)
+
+
+def restore_entry_guard(apps, schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            LOCK TABLE entry_requirements_entryfactrevision
+                IN ACCESS EXCLUSIVE MODE;
+            DO $$
+            BEGIN
+                IF EXISTS (
+                    SELECT 1
+                      FROM entry_requirements_entryfactrevision AS fact
+                      JOIN sources_parserun AS parse_run
+                        ON parse_run.id = fact.parse_run_id
+                      JOIN sources_sourceartifact AS artifact
+                        ON artifact.id = parse_run.artifact_id
+                      JOIN sources_sourceconfiguration AS source
+                        ON source.id = artifact.source_id
+                     WHERE fact.country_id <>
+                           '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+                        OR source.code = 'MOFA_ENTRY_CSV_TRAVEL6'
+                        OR parse_run.parser_contract_fingerprint_sha256 =
+                           '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f'
+                ) THEN
+                    RAISE EXCEPTION
+                        'entry country-scope rollback requires legacy JP revisions'
+                        USING ERRCODE = 'check_violation';
+                END IF;
+            END;
+            $$;
+            """
+        )
+    _rewrite_guard(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("countries", "0003_supported_country_allowlist"),
+        ("entry_requirements", "0001_initial"),
+        ("sources", "0011_multi_country_parser_contracts"),
+    ]
+
+    operations = [
+        migrations.AlterField(
+            model_name="entryfactrevision",
+            name="parse_run",
+            field=models.ForeignKey(
+                on_delete=django.db.models.deletion.PROTECT,
+                related_name="entry_fact_revisions",
+                to="sources.parserun",
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="entryfactrevision",
+            name="entry_fact_country_jp",
+        ),
+        migrations.AddConstraint(
+            model_name="entryfactrevision",
+            constraint=models.CheckConstraint(
+                condition=models.Q(country_id__in=SUPPORTED_COUNTRY_IDS),
+                name="entry_fact_country_supported",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="entryfactrevision",
+            constraint=models.UniqueConstraint(
+                fields=("parse_run", "country", "passport_policy"),
+                name="entry_fact_parse_country_policy_unique",
+            ),
+        ),
+        migrations.RunPython(generalize_entry_guard, restore_entry_guard),
+    ]
diff --git a/entry_requirements/models.py b/entry_requirements/models.py
index 2ea2da7..5df03b6 100644
--- a/entry_requirements/models.py
+++ b/entry_requirements/models.py
@@ -4,7 +4,7 @@ from datetime import date
 from django.db import models
 from django.db.models import F, Q
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import SUPPORTED_COUNTRY_IDS, Country
 from sources.models import ParseRun
 
 
@@ -53,10 +53,10 @@ class EntryFactRevision(models.Model):
         on_delete=models.PROTECT,
         related_name="entry_fact_revisions",
     )
-    parse_run = models.OneToOneField(
+    parse_run = models.ForeignKey(
         ParseRun,
         on_delete=models.PROTECT,
-        related_name="entry_fact_revision",
+        related_name="entry_fact_revisions",
     )
     ordinary_passport_period_text = models.CharField(
         max_length=ENTRY_PERIOD_TEXT_MAX_LENGTH
@@ -70,8 +70,8 @@ class EntryFactRevision(models.Model):
     class Meta:
         constraints = [
             models.CheckConstraint(
-                condition=Q(country_id=JP_COUNTRY_ID),
-                name="entry_fact_country_jp",
+                condition=Q(country_id__in=tuple(SUPPORTED_COUNTRY_IDS.values())),
+                name="entry_fact_country_supported",
             ),
             models.CheckConstraint(
                 condition=Q(passport_policy_id=PASSPORT_POLICY_ID),
@@ -101,4 +101,8 @@ class EntryFactRevision(models.Model):
                 condition=Q(created_at__gte=F("first_observed_at")),
                 name="entry_fact_created_after_observed",
             ),
+            models.UniqueConstraint(
+                fields=("parse_run", "country", "passport_policy"),
+                name="entry_fact_parse_country_policy_unique",
+            ),
         ]
diff --git a/entry_requirements/parser.py b/entry_requirements/parser.py
index eb4663d..75df41b 100644
--- a/entry_requirements/parser.py
+++ b/entry_requirements/parser.py
@@ -6,6 +6,7 @@ import re
 from dataclasses import dataclass
 from datetime import date
 
+from countries.models import SUPPORTED_COUNTRY_ROWS
 from sources.models import ParseRun
 
 from .models import (
@@ -18,6 +19,20 @@ from .models import (
 
 
 MAX_ENTRY_CSV_BYTES = 262_144
+LEGACY_ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256 = (
+    "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+)
+MULTI_COUNTRY_ENTRY_PARSER_CONTRACT_TEXT = (
+    "MOFA entry CSV V1|cp949|exact schema|supported countries "
+    "JP,TW,HK,MO,VN,TH|KOR ordinary short tourism|typed period and basis|"
+    "raw body retention zero"
+)
+MULTI_COUNTRY_ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256 = hashlib.sha256(
+    MULTI_COUNTRY_ENTRY_PARSER_CONTRACT_TEXT.encode("utf-8")
+).hexdigest()
+ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256 = (
+    MULTI_COUNTRY_ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256
+)
 ENTRY_SCHEMA_FINGERPRINT_SHA256 = (
     "46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b"
 )
@@ -34,7 +49,11 @@ ENTRY_HEADERS = (
     "비고",
 )
 
-_COUNTRY_NAME = "일본"
+_SUPPORTED_COUNTRY_NAMES = {
+    iso_alpha2: name_ko
+    for _country_id, iso_alpha2, name_ko, _name_en, _is_public
+    in SUPPORTED_COUNTRY_ROWS
+}
 _COUNTRY_INDEX = 0
 _VALIDATION_MARKER_INDEX = 2
 _PERIOD_INDEX = 3
@@ -80,11 +99,12 @@ def schema_fingerprint(headers: tuple[str, ...] | list[str]) -> str:
 def typed_fingerprint_sha256(
     ordinary_passport_period_text: str,
     basis_text: str,
+    country_iso2: str = "JP",
 ) -> str:
     canonical = json.dumps(
         {
             "basis_text": basis_text,
-            "country_iso2": "JP",
+            "country_iso2": country_iso2,
             "ordinary_passport_period_text": ordinary_passport_period_text,
             "passport_policy_code": PASSPORT_POLICY_CODE,
             "passport_policy_revision": PASSPORT_POLICY_REVISION,
@@ -115,7 +135,10 @@ def _quarantined(failure_code: str, observed: str = "") -> EntryParseResult:
     return _result(ParseRun.Outcome.QUARANTINED, failure_code, observed)
 
 
-def _parse_entry_csv(payload: bytes) -> EntryParseResult:
+def _parse_entry_csv(payload: bytes, country_iso2: str) -> EntryParseResult:
+    country_name = _SUPPORTED_COUNTRY_NAMES.get(country_iso2)
+    if country_name is None:
+        return _quarantined(ParseRun.FailureCode.IDENTITY_MISMATCH)
     if len(payload) > MAX_ENTRY_CSV_BYTES:
         return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
 
@@ -142,7 +165,7 @@ def _parse_entry_csv(payload: bytes) -> EntryParseResult:
         for row in reader:
             if len(row) != len(ENTRY_HEADERS):
                 return _quarantined(ParseRun.FailureCode.SYNTAX_ERROR)
-            if row[_COUNTRY_INDEX] == _COUNTRY_NAME:
+            if row[_COUNTRY_INDEX] == country_name:
                 country_rows.append(
                     (
                         row[_VALIDATION_MARKER_INDEX],
@@ -186,9 +209,13 @@ def _parse_entry_csv(payload: bytes) -> EntryParseResult:
             ENTRY_SCHEMA_FINGERPRINT_SHA256,
         )
 
-    typed_hash = typed_fingerprint_sha256(period_text, basis_text)
+    typed_hash = typed_fingerprint_sha256(
+        period_text,
+        basis_text,
+        country_iso2,
+    )
     projection = EntryProjection(
-        country_iso2="JP",
+        country_iso2=country_iso2,
         passport_policy_code=PASSPORT_POLICY_CODE,
         passport_policy_revision=PASSPORT_POLICY_REVISION,
         ordinary_passport_period_text=period_text,
@@ -203,14 +230,18 @@ def _parse_entry_csv(payload: bytes) -> EntryParseResult:
     )
 
 
-def parse_entry_csv(payload: bytes) -> EntryParseResult:
-    if not isinstance(payload, bytes):
+def parse_entry_csv(
+    payload: bytes,
+    *,
+    country_iso2: str = "JP",
+) -> EntryParseResult:
+    if not isinstance(payload, bytes) or type(country_iso2) is not str:
         return _result(
             ParseRun.Outcome.FAILED,
             ParseRun.FailureCode.INTERNAL_PARSER_ERROR,
         )
     try:
-        return _parse_entry_csv(payload)
+        return _parse_entry_csv(payload, country_iso2)
     except Exception:
         return _result(
             ParseRun.Outcome.FAILED,
diff --git a/entry_requirements/tests/test_ingestion.py b/entry_requirements/tests/test_ingestion.py
index 03067a9..92759dd 100644
--- a/entry_requirements/tests/test_ingestion.py
+++ b/entry_requirements/tests/test_ingestion.py
@@ -12,11 +12,12 @@ from django.core.management.base import CommandError
 from django.db import connection, models
 from django.test import SimpleTestCase, TransactionTestCase
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS, Country
 import entry_requirements.ingestion as entry_ingestion
 from entry_requirements.ingestion import (
     ENTRY_INGESTION_LOCK_KEY,
     ENTRY_INGESTION_LOCK_NAMESPACE,
+    ENTRY_SOURCE_CODE,
     EntryIngestionCode,
     EntryIngestionOutcome,
     _verify_transport_result,
@@ -95,16 +96,25 @@ def validation_marker() -> str:
     raise AssertionError("approved validation marker could not be reconstructed")
 
 
-def entry_payload(*, headers=ENTRY_HEADERS, period=PERIOD_TEXT) -> bytes:
-    row = [""] * len(ENTRY_HEADERS)
-    row[0] = "일본"
-    row[2] = validation_marker()
-    row[3] = period
-    row[8] = BASIS_TEXT
+def entry_payload(
+    *,
+    headers=ENTRY_HEADERS,
+    period=PERIOD_TEXT,
+    country="일본",
+    additional_countries=(),
+) -> bytes:
+    rows = []
+    for country_name in (country, *additional_countries):
+        row = [""] * len(ENTRY_HEADERS)
+        row[0] = country_name
+        row[2] = validation_marker()
+        row[3] = period
+        row[8] = BASIS_TEXT
+        rows.append(row)
     output = io.StringIO(newline="")
     writer = csv.writer(output, lineterminator="\r\n")
     writer.writerow(headers)
-    writer.writerow(row)
+    writer.writerows(rows)
     return output.getvalue().encode("cp949", errors="strict")
 
 
@@ -178,7 +188,7 @@ class EntryIngestionTests(TransactionTestCase):
         register_approved_sources(apply=True)
 
     def reject_entry_rights(self):
-        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
+        source = SourceConfiguration.objects.get(code=ENTRY_SOURCE_CODE)
         approval = source.rights_decisions.get(decision_sequence=1)
         return SourceRightsDecision.objects.create(
             source=source,
@@ -245,6 +255,38 @@ class EntryIngestionTests(TransactionTestCase):
         self.assertEqual(fact.ordinary_passport_period_text, PERIOD_TEXT)
         self.assertEqual(fact.basis_text, BASIS_TEXT)
 
+    def test_same_snapshot_adds_tw_to_one_parse_run_then_replays(self):
+        payload = entry_payload(additional_countries=("대만",))
+
+        jp = ingest_entry_snapshot(
+            country_iso2="JP",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+        tw = ingest_entry_snapshot(
+            country_iso2="TW",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+        tw_replay = ingest_entry_snapshot(
+            country_iso2="TW",
+            transport=SequenceTransport(successful_result(payload)),
+        )
+
+        self.assertEqual(jp.code, EntryIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(tw.code, EntryIngestionCode.REVIEW_REQUIRED)
+        self.assertEqual(tw_replay.code, EntryIngestionCode.REPLAY_OBSERVED)
+        self.assertEqual(SourceArtifact.objects.count(), 1)
+        self.assertEqual(ParseRun.objects.count(), 1)
+        self.assertEqual(EntryFactRevision.objects.count(), 2)
+        revisions = EntryFactRevision.objects.order_by("country__iso_alpha2")
+        self.assertEqual(
+            set(revisions.values_list("country_id", flat=True)),
+            {JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS["TW"]},
+        )
+        self.assertEqual(
+            len(set(revisions.values_list("parse_run_id", flat=True))),
+            1,
+        )
+
     def test_same_body_replay_adds_only_fresh_attempt_evidence(self):
         payload = entry_payload()
         first = ingest_entry_snapshot(
@@ -443,7 +485,7 @@ class EntryIngestionTests(TransactionTestCase):
             contender.close()
 
     def test_prior_started_receipt_is_reconciled_before_new_operation(self):
-        source = SourceConfiguration.objects.get(code="MOFA_ENTRY_CSV")
+        source = SourceConfiguration.objects.get(code=ENTRY_SOURCE_CODE)
         rights = source.rights_decisions.get(decision_sequence=1)
         abandoned = FetchAttempt.objects.create(
             source=source,
@@ -718,7 +760,10 @@ class EntryIngestionTests(TransactionTestCase):
         self.assertNotIn(BASIS_TEXT, repr(verified))
         self.assertNotIn(payload.hex(), repr(verified))
         parameters = set(inspect.signature(ingest_entry_snapshot).parameters)
-        self.assertEqual(parameters, {"transport", "parser", "retry_wait"})
+        self.assertEqual(
+            parameters,
+            {"country_iso2", "transport", "parser", "retry_wait"},
+        )
         banned = {
             "raw",
             "secret",
@@ -750,6 +795,15 @@ class EntryIngestionTests(TransactionTestCase):
 
 
 class EntryIngestionCommandTests(SimpleTestCase):
+    def test_command_country_option_is_exactly_the_supported_allowlist(self):
+        with patch(
+            "operations.management.commands.ingest_entry_snapshot."
+            "ingest_entry_snapshot"
+        ) as ingestion:
+            with self.assertRaises(CommandError):
+                call_command("ingest_entry_snapshot", "--country", "US")
+        ingestion.assert_not_called()
+
     def test_command_output_is_a_fixed_redacted_code(self):
         output = io.StringIO()
         with patch(
@@ -759,12 +813,13 @@ class EntryIngestionCommandTests(SimpleTestCase):
                 EntryIngestionCode.REVIEW_REQUIRED,
                 1,
             ),
-        ):
-            call_command("ingest_entry_snapshot", stdout=output)
+        ) as ingestion:
+            call_command("ingest_entry_snapshot", "--country", "TW", stdout=output)
         self.assertEqual(
             output.getvalue(),
-            "entry_ingestion_result=REVIEW_REQUIRED attempts=1\n",
+            "entry_ingestion_result=REVIEW_REQUIRED attempts=1 country=TW\n",
         )
+        ingestion.assert_called_once_with(country_iso2="TW")
 
     def test_command_never_exposes_internal_exception_text(self):
         marker = "sensitive-secret-and-raw-marker"
@@ -777,7 +832,10 @@ class EntryIngestionCommandTests(SimpleTestCase):
                 call_command("ingest_entry_snapshot")
         self.assertEqual(
             str(raised.exception),
-            "entry_ingestion_result=PERSISTENCE_FAILED",
+            (
+                "entry_ingestion_result=PERSISTENCE_FAILED "
+                "attempts=0 country=JP"
+            ),
         )
         self.assertNotIn(marker, str(raised.exception))
         assert_exception_boundary_is_redacted(self, raised.exception, marker)
@@ -793,7 +851,10 @@ class EntryIngestionCommandTests(SimpleTestCase):
                 call_command("ingest_entry_snapshot")
         self.assertEqual(
             str(raised.exception),
-            "entry_ingestion_result=PERSISTENCE_FAILED",
+            (
+                "entry_ingestion_result=PERSISTENCE_FAILED "
+                "attempts=0 country=JP"
+            ),
         )
         self.assertNotIn(marker, str(raised.exception))
         assert_exception_boundary_is_redacted(self, raised.exception, marker)
@@ -814,6 +875,9 @@ class EntryIngestionCommandTests(SimpleTestCase):
                         call_command("ingest_entry_snapshot")
                 self.assertEqual(
                     str(raised.exception),
-                    "entry_ingestion_result=PERSISTENCE_FAILED",
+                    (
+                        "entry_ingestion_result=PERSISTENCE_FAILED "
+                        "attempts=0 country=JP"
+                    ),
                 )
                 self.assertNotIn(marker, str(raised.exception))
diff --git a/entry_requirements/tests/test_models.py b/entry_requirements/tests/test_models.py
index 8dbd3b8..3f47a51 100644
--- a/entry_requirements/tests/test_models.py
+++ b/entry_requirements/tests/test_models.py
@@ -35,7 +35,7 @@ from sources.management.commands.register_approved_sources import (
 )
 
 
-ENTRY_CONTRACT = "622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021"
+ENTRY_CONTRACT = "2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f"
 ENTRY_SCHEMA = "46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b"
 WARNING_CONTRACT = "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
 WARNING_SCHEMA = "64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b"
@@ -51,7 +51,7 @@ class EntryFactFixtureMixin:
     def make_parse_run(
         self,
         *,
-        source_code="MOFA_ENTRY_CSV",
+        source_code="MOFA_ENTRY_CSV_TRAVEL6",
         terminal=True,
         validate_artifact=True,
         byte_count=100,
@@ -98,7 +98,9 @@ class EntryFactFixtureMixin:
             ),
             parser_version=ParseRun.ParserVersion.V1,
             parser_contract_fingerprint_sha256=(
-                ENTRY_CONTRACT if is_entry else WARNING_CONTRACT
+                ENTRY_CONTRACT
+                if source_code == "MOFA_ENTRY_CSV_TRAVEL6"
+                else WARNING_CONTRACT
             ),
             expected_schema_fingerprint_sha256=(
                 ENTRY_SCHEMA if is_entry else WARNING_SCHEMA
@@ -420,7 +422,11 @@ class EntryFactRevisionTests(EntryFactFixtureMixin, TestCase):
 
 class EntryRightsCapabilityTests(EntryFactFixtureMixin, TestCase):
     def activate_entry_source(self, **rights_overrides):
-        spec = APPROVED_SOURCE_SPECS[0]
+        spec = next(
+            item
+            for item in APPROVED_SOURCE_SPECS
+            if item.code == "MOFA_ENTRY_CSV_TRAVEL6"
+        )
         source = SourceConfiguration.objects.create(
             **spec.configuration_values(),
             state=SourceConfiguration.State.DRAFT,
@@ -499,10 +505,7 @@ class EntryMigrationBoundaryTests(EntryFactFixtureMixin, TransactionTestCase):
         run, _, attempt = self.make_parse_run()
         EntryFactRevision.objects.create(**self.fact_values(run, attempt))
 
-        with self.assertRaisesMessage(
-            RuntimeError,
-            "entry boundary rollback requires no fact revisions",
-        ):
+        with self.assertRaises(IntegrityError):
             MigrationExecutor(connection).migrate([("entry_requirements", None)])
 
         self.assertIn(
diff --git a/entry_requirements/tests/test_parser.py b/entry_requirements/tests/test_parser.py
index 63ae879..9a66df5 100644
--- a/entry_requirements/tests/test_parser.py
+++ b/entry_requirements/tests/test_parser.py
@@ -117,6 +117,22 @@ class EntryCsvParserTests(SimpleTestCase):
             typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT),
         )
 
+    def test_supported_non_jp_country_has_country_scoped_typed_hash(self):
+        payload = encode_csv([make_row(country="일본"), make_row(country="대만")])
+
+        result = parse_entry_csv(payload, country_iso2="TW")
+
+        self.assertEqual(result.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(result.projection.country_iso2, "TW")
+        self.assertEqual(
+            result.projection.typed_fingerprint_sha256,
+            typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT, "TW"),
+        )
+        self.assertNotEqual(
+            result.projection.typed_fingerprint_sha256,
+            typed_fingerprint_sha256(PERIOD_TEXT, BASIS_TEXT),
+        )
+
     def test_quoted_commas_and_newlines_are_parsed_without_changing_typed_hash(self):
         quoted_basis = BASIS_TEXT + ", synthetic detail\nsecond line"
         crlf = parse_entry_csv(encode_csv([make_row(basis=quoted_basis)]))
diff --git a/operations/management/commands/ingest_entry_snapshot.py b/operations/management/commands/ingest_entry_snapshot.py
index de08324..7818469 100644
--- a/operations/management/commands/ingest_entry_snapshot.py
+++ b/operations/management/commands/ingest_entry_snapshot.py
@@ -1,5 +1,6 @@
 from django.core.management.base import BaseCommand, CommandError
 
+from countries.models import SUPPORTED_COUNTRY_IDS
 from entry_requirements.ingestion import (
     SUCCESS_CODES,
     EntryIngestionCode,
@@ -24,18 +25,30 @@ KNOWN_CODES = frozenset(
 )
 
 
-def _invoke_ingestion_without_exception_context():
+SUPPORTED_COUNTRY_CODES = tuple(SUPPORTED_COUNTRY_IDS)
+
+
+def _invoke_ingestion_without_exception_context(country_iso2):
     try:
-        return ingest_entry_snapshot()
+        return ingest_entry_snapshot(country_iso2=country_iso2)
     except BaseException:
         return None
 
 
 class Command(BaseCommand):
-    help = "Ingest the exact approved MOFA entry snapshot into review."
+    help = "Ingest one supported-country row from the approved MOFA entry snapshot."
+
+    def add_arguments(self, parser):
+        parser.add_argument(
+            "--country",
+            choices=SUPPORTED_COUNTRY_CODES,
+            default="JP",
+            dest="country_iso2",
+        )
 
     def handle(self, *args, **options):
-        outcome = _invoke_ingestion_without_exception_context()
+        country_iso2 = options["country_iso2"]
+        outcome = _invoke_ingestion_without_exception_context(country_iso2)
         if (
             not isinstance(outcome, EntryIngestionOutcome)
             or outcome.code not in KNOWN_CODES
@@ -43,11 +56,12 @@ class Command(BaseCommand):
             or not 0 <= outcome.attempt_count <= 3
         ):
             raise CommandError(
-                "entry_ingestion_result=PERSISTENCE_FAILED"
+                "entry_ingestion_result=PERSISTENCE_FAILED "
+                f"attempts=0 country={country_iso2}"
             ) from None
         message = (
             f"entry_ingestion_result={outcome.code} "
-            f"attempts={outcome.attempt_count}"
+            f"attempts={outcome.attempt_count} country={country_iso2}"
         )
         if outcome.code not in SUCCESS_CODES:
             raise CommandError(message)
diff --git a/operations/management/commands/ingest_travel_warning.py b/operations/management/commands/ingest_travel_warning.py
index 820a6a9..6e73fd2 100644
--- a/operations/management/commands/ingest_travel_warning.py
+++ b/operations/management/commands/ingest_travel_warning.py
@@ -1,5 +1,6 @@
 from django.core.management.base import BaseCommand, CommandError
 
+from countries.models import SUPPORTED_COUNTRY_IDS
 from travel_warnings.ingestion import (
     SUCCESS_CODES,
     TravelWarningIngestionCode,
@@ -24,18 +25,30 @@ KNOWN_CODES = frozenset(
 )
 
 
-def _invoke_ingestion_without_exception_context():
+SUPPORTED_COUNTRY_CODES = tuple(SUPPORTED_COUNTRY_IDS)
+
+
+def _invoke_ingestion_without_exception_context(country_iso2):
     try:
-        return ingest_travel_warning()
+        return ingest_travel_warning(country_iso2=country_iso2)
     except BaseException:
         return None
 
 
 class Command(BaseCommand):
-    help = "Ingest the exact approved MOFA JP travel warning into review."
+    help = "Ingest one supported-country warning from the approved MOFA API."
+
+    def add_arguments(self, parser):
+        parser.add_argument(
+            "--country",
+            choices=SUPPORTED_COUNTRY_CODES,
+            default="JP",
+            dest="country_iso2",
+        )
 
     def handle(self, *args, **options):
-        outcome = _invoke_ingestion_without_exception_context()
+        country_iso2 = options["country_iso2"]
+        outcome = _invoke_ingestion_without_exception_context(country_iso2)
         if (
             not isinstance(outcome, TravelWarningIngestionOutcome)
             or outcome.code not in KNOWN_CODES
@@ -43,11 +56,12 @@ class Command(BaseCommand):
             or not 0 <= outcome.attempt_count <= 3
         ):
             raise CommandError(
-                "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0"
+                "travel_warning_ingestion_result=PERSISTENCE_FAILED "
+                f"attempts=0 country={country_iso2}"
             ) from None
         message = (
             f"travel_warning_ingestion_result={outcome.code} "
-            f"attempts={outcome.attempt_count}"
+            f"attempts={outcome.attempt_count} country={country_iso2}"
         )
         if outcome.code not in SUCCESS_CODES:
             raise CommandError(message)
diff --git a/reviews/migrations/0003_country_conditional_source_contracts.py b/reviews/migrations/0003_country_conditional_source_contracts.py
new file mode 100644
index 0000000..8f958d1
--- /dev/null
+++ b/reviews/migrations/0003_country_conditional_source_contracts.py
@@ -0,0 +1,234 @@
+import uuid
+
+from django.db import migrations, models
+
+
+FUNCTION_NAME = "reviews_guard_publication_revision"
+REPLACEMENTS = (
+    (
+        """       OR NEW.scope_country_id IS DISTINCT FROM typed_country_id
+       OR NEW.scope_passport_policy_id IS DISTINCT FROM typed_policy_id
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'""",
+        """       OR NEW.scope_country_id IS DISTINCT FROM typed_country_id
+       OR NEW.scope_passport_policy_id IS DISTINCT FROM typed_policy_id
+       OR (
+            typed_country_id <>
+              '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+            AND source_code IN (
+                'MOFA_ENTRY_CSV',
+                'MOFA_TRAVEL_ALARM_API_JP'
+            )
+       )
+       OR parse_outcome IS DISTINCT FROM 'VALIDATED'""",
+    ),
+    (
+        """        OR rights_contract IS DISTINCT FROM
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+    ) THEN""",
+        """        OR rights_contract IS DISTINCT FROM
+          '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'
+    ) AND NOT (
+        source_code IS NOT DISTINCT FROM 'MOFA_ENTRY_CSV_TRAVEL6'
+        AND chain_source_revision IS NOT DISTINCT FROM 'travel6-v1'
+        AND source_module IS NOT DISTINCT FROM 'ENTRY'
+        AND source_owner IS NOT DISTINCT FROM '대한민국 외교부 정보화담당관실'
+        AND source_locator IS NOT DISTINCT FROM
+          'https://www.data.go.kr/cmm/cmm/fileDownload.do?atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N'
+        AND source_secret_reference IS NOT DISTINCT FROM ''
+        AND artifact_byte_count <= 262144
+        AND rights_decided_by IS NOT DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+        AND rights_decision_basis IS NOT DISTINCT FROM
+          'USER_TRAVEL6_SCOPE_20260831'
+        AND parse_name IS NOT DISTINCT FROM 'MOFA_ENTRY_CSV'
+        AND parse_version IS NOT DISTINCT FROM 'V1'
+        AND parse_contract IS NOT DISTINCT FROM
+          '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f'
+        AND parse_schema IS NOT DISTINCT FROM
+          '46bb5428a08a94810e6d381541cb7a3aed3f2f5110a529c2ae32e101cd5e1e0b'
+        AND attempt_provider_code IS NOT DISTINCT FROM ''
+        AND rights_access_mode IS NOT DISTINCT FROM 'ANONYMOUS_PUBLIC'
+        AND rights_scope IS NOT DISTINCT FROM
+          'SUPPORTED_6_ORDINARY_TEXT_V1'
+        AND rights_attribution IS NOT DISTINCT FROM '외교부|공공데이터포털'
+        AND rights_contract IS NOT DISTINCT FROM
+          '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f'
+    ) THEN""",
+    ),
+    (
+        """        OR rights_contract IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+    ) THEN""",
+        """        OR rights_contract IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'
+    ) AND NOT (
+        source_code IS NOT DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_TRAVEL6'
+        AND chain_source_revision IS NOT DISTINCT FROM 'travel6-v1'
+        AND source_module IS NOT DISTINCT FROM 'TRAVEL_WARNING'
+        AND source_owner IS NOT DISTINCT FROM '대한민국 외교부'
+        AND source_locator IS NOT DISTINCT FROM
+          'https://apis.data.go.kr/1262000/TravelAlarmService2/getTravelAlarmList2'
+        AND source_secret_reference IS NOT DISTINCT FROM
+          'DATA_GO_KR_SERVICE_KEY'
+        AND artifact_byte_count <= 4096
+        AND rights_decided_by IS NOT DISTINCT FROM 'PROJECT_OWNER_REQUEST'
+        AND rights_decision_basis IS NOT DISTINCT FROM
+          'USER_TRAVEL6_SCOPE_20260831'
+        AND parse_name IS NOT DISTINCT FROM 'MOFA_TRAVEL_ALARM_JSON'
+        AND parse_version IS NOT DISTINCT FROM 'V1'
+        AND parse_contract IS NOT DISTINCT FROM
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+        AND parse_schema IS NOT DISTINCT FROM
+          '64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b'
+        AND attempt_provider_code IS NOT DISTINCT FROM 'MOFA_SUCCESS_0'
+        AND rights_access_mode IS NOT DISTINCT FROM 'CREDENTIAL_REFERENCE'
+        AND rights_scope IS NOT DISTINCT FROM 'SUPPORTED_6_WARNING_V1'
+        AND rights_attribution IS NOT DISTINCT FROM '외교부|공공데이터포털'
+        AND rights_contract IS NOT DISTINCT FROM
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+    ) THEN""",
+    ),
+)
+
+
+def _rewrite_guard(schema_editor, *, reverse=False):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [FUNCTION_NAME],
+        )
+        rows = cursor.fetchall()
+        if len(rows) != 1:
+            raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+        body = rows[0][0]
+        for forward_old, forward_new in REPLACEMENTS:
+            old, new = (
+                (forward_new, forward_old)
+                if reverse
+                else (forward_old, forward_new)
+            )
+            if body.count(old) != 1:
+                raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+            body = body.replace(old, new)
+        quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $reviews_country_contracts$
+            {body}
+            $reviews_country_contracts$
+            """
+        )
+
+
+def allow_country_contracts(apps, schema_editor):
+    _rewrite_guard(schema_editor)
+
+
+def restore_legacy_contracts(apps, schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            DO $$
+            BEGIN
+                IF EXISTS (
+                    SELECT 1 FROM reviews_publicationrevision
+                     WHERE source_code_snapshot IN (
+                        'MOFA_ENTRY_CSV_TRAVEL6',
+                        'MOFA_TRAVEL_ALARM_API_TRAVEL6'
+                     )
+                ) THEN
+                    RAISE EXCEPTION
+                        'multi-country publication rollback requires no new history'
+                        USING ERRCODE = 'check_violation';
+                END IF;
+            END;
+            $$;
+            """
+        )
+    _rewrite_guard(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("entry_requirements", "0002_country_scoped_entry_facts"),
+        ("reviews", "0002_country_scoped_publications"),
+        ("travel_warnings", "0003_country_scoped_warning_revisions"),
+    ]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="publicationrevision",
+            name="publication_exact_typed_scope",
+        ),
+        migrations.AddConstraint(
+            model_name="publicationrevision",
+            constraint=models.CheckConstraint(
+                condition=(
+                    (
+                        models.Q(
+                            module="ENTRY",
+                            scope_passport_policy_id=uuid.UUID(
+                                "f461f7a7-18f7-5b0d-9831-bfbd47b695e5"
+                            ),
+                            entry_fact_revision__isnull=False,
+                            travel_warning_revision__isnull=True,
+                            limitation_code=(
+                                "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN"
+                            ),
+                            parser_name="MOFA_ENTRY_CSV",
+                            parser_version="V1",
+                        )
+                        & (
+                            models.Q(
+                                source_code_snapshot="MOFA_ENTRY_CSV_TRAVEL6"
+                            )
+                            | models.Q(
+                                scope_country_id=uuid.UUID(
+                                    "575fa8b9-14f9-526e-9464-ebd1dea76da9"
+                                ),
+                                source_code_snapshot="MOFA_ENTRY_CSV",
+                            )
+                        )
+                    )
+                    | (
+                        models.Q(
+                            module="TRAVEL_WARNING",
+                            scope_passport_policy__isnull=True,
+                            entry_fact_revision__isnull=True,
+                            travel_warning_revision__isnull=False,
+                            limitation_code=(
+                                "WARNING_EFFECTIVE_PERIOD_UNPROVEN"
+                            ),
+                            parser_name="MOFA_TRAVEL_ALARM_JSON",
+                            parser_version="V1",
+                        )
+                        & (
+                            models.Q(
+                                source_code_snapshot=(
+                                    "MOFA_TRAVEL_ALARM_API_TRAVEL6"
+                                )
+                            )
+                            | models.Q(
+                                scope_country_id=uuid.UUID(
+                                    "575fa8b9-14f9-526e-9464-ebd1dea76da9"
+                                ),
+                                source_code_snapshot=(
+                                    "MOFA_TRAVEL_ALARM_API_JP"
+                                ),
+                            )
+                        )
+                    )
+                ),
+                name="publication_exact_typed_scope",
+            ),
+        ),
+        migrations.RunPython(allow_country_contracts, restore_legacy_contracts),
+    ]
diff --git a/reviews/models.py b/reviews/models.py
index c25c8d9..9c68fe3 100644
--- a/reviews/models.py
+++ b/reviews/models.py
@@ -5,6 +5,7 @@ from django.db import models
 from django.db.models import F, Q
 from django.utils import timezone
 
+from countries.models import JP_COUNTRY_ID
 from entry_requirements.models import PASSPORT_POLICY_ID
 
 
@@ -233,27 +234,51 @@ class PublicationRevision(models.Model):
         constraints = [
             models.CheckConstraint(
                 condition=(
-                    Q(
-                        module=PublicationModule.ENTRY,
-                        scope_passport_policy_id=PASSPORT_POLICY_ID,
-                        entry_fact_revision__isnull=False,
-                        travel_warning_revision__isnull=True,
-                        limitation_code=(
-                            "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN"
-                        ),
-                        source_code_snapshot="MOFA_ENTRY_CSV",
-                        parser_name="MOFA_ENTRY_CSV",
-                        parser_version="V1",
+                    (
+                        Q(
+                            module=PublicationModule.ENTRY,
+                            scope_passport_policy_id=PASSPORT_POLICY_ID,
+                            entry_fact_revision__isnull=False,
+                            travel_warning_revision__isnull=True,
+                            limitation_code=(
+                                "ENTRY_PURPOSE_DATE_APPLICABILITY_UNPROVEN"
+                            ),
+                            parser_name="MOFA_ENTRY_CSV",
+                            parser_version="V1",
+                        )
+                        & (
+                            Q(source_code_snapshot="MOFA_ENTRY_CSV_TRAVEL6")
+                            | Q(
+                                scope_country_id=JP_COUNTRY_ID,
+                                source_code_snapshot="MOFA_ENTRY_CSV",
+                            )
+                        )
                     )
-                    | Q(
-                        module=PublicationModule.TRAVEL_WARNING,
-                        scope_passport_policy__isnull=True,
-                        entry_fact_revision__isnull=True,
-                        travel_warning_revision__isnull=False,
-                        limitation_code="WARNING_EFFECTIVE_PERIOD_UNPROVEN",
-                        source_code_snapshot="MOFA_TRAVEL_ALARM_API_JP",
-                        parser_name="MOFA_TRAVEL_ALARM_JSON",
-                        parser_version="V1",
+                    | (
+                        Q(
+                            module=PublicationModule.TRAVEL_WARNING,
+                            scope_passport_policy__isnull=True,
+                            entry_fact_revision__isnull=True,
+                            travel_warning_revision__isnull=False,
+                            limitation_code=(
+                                "WARNING_EFFECTIVE_PERIOD_UNPROVEN"
+                            ),
+                            parser_name="MOFA_TRAVEL_ALARM_JSON",
+                            parser_version="V1",
+                        )
+                        & (
+                            Q(
+                                source_code_snapshot=(
+                                    "MOFA_TRAVEL_ALARM_API_TRAVEL6"
+                                )
+                            )
+                            | Q(
+                                scope_country_id=JP_COUNTRY_ID,
+                                source_code_snapshot=(
+                                    "MOFA_TRAVEL_ALARM_API_JP"
+                                ),
+                            )
+                        )
                     )
                 ),
                 name="publication_exact_typed_scope",
diff --git a/reviews/publication.py b/reviews/publication.py
index 11508ce..853a7aa 100644
--- a/reviews/publication.py
+++ b/reviews/publication.py
@@ -4,7 +4,7 @@ from __future__ import annotations
 
 import hashlib
 import uuid
-from dataclasses import dataclass
+from dataclasses import dataclass, replace
 from enum import Enum
 from typing import Any, NoReturn
 
@@ -13,6 +13,7 @@ from django.db import connection, transaction
 from django.db.models import F
 from django.utils import timezone
 
+from countries.models import JP_COUNTRY_ID
 from entry_requirements.ingestion import (
     ENTRY_ATTRIBUTION,
     ENTRY_CONTRACT_FINGERPRINT_SHA256,
@@ -25,6 +26,10 @@ from entry_requirements.ingestion import (
     ENTRY_SOURCE_MODULE,
     ENTRY_SOURCE_OWNER,
     ENTRY_SOURCE_REVISION,
+    LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_ENTRY_FIELD_SCOPE,
+    LEGACY_ENTRY_SOURCE_CODE,
+    LEGACY_ENTRY_SOURCE_REVISION,
 )
 from entry_requirements.models import EntryFactRevision
 from entry_requirements.parser import ENTRY_SCHEMA_FINGERPRINT_SHA256
@@ -34,7 +39,11 @@ from sources.models import (
     SourceArtifact,
     SourceRightsDecision,
 )
-from sources.transport import ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR
+from sources.transport import (
+    ENTRY_SOURCE_LOCATOR,
+    WARNING_SECRET_REFERENCE,
+    WARNING_SOURCE_LOCATOR,
+)
 from travel_warnings.ingestion import (
     WARNING_ATTRIBUTION,
     WARNING_DECIDED_BY,
@@ -46,6 +55,10 @@ from travel_warnings.ingestion import (
     WARNING_SOURCE_MODULE,
     WARNING_SOURCE_OWNER,
     WARNING_SOURCE_REVISION,
+    LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_WARNING_FIELD_SCOPE,
+    LEGACY_WARNING_SOURCE_CODE,
+    LEGACY_WARNING_SOURCE_REVISION,
 )
 from travel_warnings.models import TravelWarningRevision
 from travel_warnings.parser import (
@@ -185,11 +198,31 @@ _SPECS = {
         parser_version=WARNING_PARSER_VERSION,
         schema_fingerprint=EXPECTED_SCHEMA_FINGERPRINT_SHA256,
         access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
-        secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+        secret_reference_name=WARNING_SECRET_REFERENCE,
         provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
     ),
 }
 
+_LEGACY_SPECS = {
+    PublicationModule.ENTRY: replace(
+        _SPECS[PublicationModule.ENTRY],
+        source_code=LEGACY_ENTRY_SOURCE_CODE,
+        source_revision=LEGACY_ENTRY_SOURCE_REVISION,
+        field_scope=LEGACY_ENTRY_FIELD_SCOPE,
+        contract_fingerprint=LEGACY_ENTRY_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis="USER_DIRECTIVE_20260830",
+    ),
+    PublicationModule.TRAVEL_WARNING: replace(
+        _SPECS[PublicationModule.TRAVEL_WARNING],
+        source_code=LEGACY_WARNING_SOURCE_CODE,
+        source_revision=LEGACY_WARNING_SOURCE_REVISION,
+        field_scope=LEGACY_WARNING_FIELD_SCOPE,
+        contract_fingerprint=LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256,
+        decision_basis="USER_FOLLOWUP_20260830",
+        secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+    ),
+}
+
 
 class _ClosedFailure(Exception):
     def __init__(self, code: str):
@@ -357,6 +390,12 @@ def _lock_subject_source(spec: _ModuleSpec, subject) -> None:
 
 
 def _validate_source_chain(spec: _ModuleSpec, subject) -> dict[str, object]:
+    source_code = subject.parse_run.artifact.source.code
+    if (
+        subject.country_id == JP_COUNTRY_ID
+        and source_code == _LEGACY_SPECS[spec.module].source_code
+    ):
+        spec = _LEGACY_SPECS[spec.module]
     parse_run = subject.parse_run
     artifact = parse_run.artifact
     source = artifact.source
diff --git a/reviews/tests/test_publication.py b/reviews/tests/test_publication.py
index 0cc25e9..d7f471d 100644
--- a/reviews/tests/test_publication.py
+++ b/reviews/tests/test_publication.py
@@ -1,5 +1,7 @@
+import hashlib
 import uuid
 from concurrent.futures import ThreadPoolExecutor
+from datetime import timedelta
 from importlib import import_module
 from threading import Barrier
 from types import SimpleNamespace
@@ -20,18 +22,25 @@ from django.db.migrations.executor import MigrationExecutor
 from django.test import SimpleTestCase, TransactionTestCase
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS, Country
 from entry_requirements.ingestion import (
+    ENTRY_SOURCE_CODE,
     EntryIngestionCode,
     ingest_entry_snapshot,
 )
 from entry_requirements.models import (
+    ENTRY_SNAPSHOT_DATE,
     PASSPORT_POLICY_CODE,
     PASSPORT_POLICY_ID,
     PASSPORT_POLICY_REVISION,
     EntryFactRevision,
     PassportPolicy,
 )
+from entry_requirements.parser import (
+    ENTRY_SCHEMA_FINGERPRINT_SHA256,
+    LEGACY_ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    typed_fingerprint_sha256,
+)
 from entry_requirements.tests.test_ingestion import (
     entry_payload,
     successful_result as successful_entry_result,
@@ -40,14 +49,28 @@ from operations.tests.migration_helpers import (
     restore_canonical_migration_graph,
 )
 from sources.management.commands.register_approved_sources import (
+    APPROVED_SOURCE_SPECS,
     register_approved_sources,
 )
-from sources.models import SourceConfiguration, SourceRightsDecision
+from sources.models import (
+    FetchAttempt,
+    ParseRun,
+    SourceArtifact,
+    SourceConfiguration,
+    SourceRightsDecision,
+)
 from travel_warnings.ingestion import (
+    WARNING_SOURCE_CODE,
     TravelWarningIngestionCode,
     ingest_travel_warning,
 )
 from travel_warnings.models import TravelWarningRevision
+from travel_warnings.parser import (
+    EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+    LEGACY_PARSER_CONTRACT_FINGERPRINT_SHA256,
+    TravelWarningParseSuccess,
+    parse_travel_alarm_jp,
+)
 from travel_warnings.tests.test_ingestion import (
     successful_result as successful_warning_result,
     warning_item,
@@ -112,19 +135,44 @@ class PublicationFixtureMixin:
         )
         register_approved_sources(apply=True)
 
-    def make_entry(self, *, period="90일"):
-        payload = entry_payload(period=period)
+    def make_entry(
+        self,
+        *,
+        period="90일",
+        country_iso2="JP",
+        country_name="일본",
+    ):
+        payload = entry_payload(period=period, country=country_name)
         outcome = ingest_entry_snapshot(
+            country_iso2=country_iso2,
             transport=lambda **_kwargs: successful_entry_result(payload),
             retry_wait=lambda _attempt: None,
         )
         self.assertEqual(outcome.code, EntryIngestionCode.REVIEW_REQUIRED)
         return EntryFactRevision.objects.latest("created_at")
 
-    def make_warning(self, *, scope_text="합성 검증 범위"):
-        payload = warning_payload(item=warning_item(remark=scope_text))
+    def make_warning(
+        self,
+        *,
+        scope_text="합성 검증 범위",
+        country_iso2="JP",
+        country_name_ko="일본",
+        country_name_en="Japan",
+    ):
+        payload = warning_payload(
+            item=warning_item(
+                remark=scope_text,
+                country_iso_alp2=country_iso2,
+                country_nm=country_name_ko,
+                country_eng_nm=country_name_en,
+            )
+        )
         outcome = ingest_travel_warning(
-            transport=lambda **_kwargs: successful_warning_result(payload),
+            country_iso2=country_iso2,
+            transport=lambda **_kwargs: successful_warning_result(
+                payload,
+                country_iso2=country_iso2,
+            ),
             secret_loader=lambda _reference: "synthetic-test-credential",
             retry_wait=lambda _attempt: None,
         )
@@ -313,6 +361,284 @@ class PublicationFixtureMixin:
         return actor
 
 
+class LegacyJpPublicationCompatibilityTests(TransactionTestCase):
+    reset_sequences = True
+
+    def _activate_legacy_source(self, code):
+        spec = next(item for item in APPROVED_SOURCE_SPECS if item.code == code)
+        source = SourceConfiguration.objects.create(
+            **spec.configuration_values(),
+            state=SourceConfiguration.State.DRAFT,
+            enabled=False,
+        )
+        rights = SourceRightsDecision.objects.create(
+            source=source,
+            **spec.rights_values(),
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.RIGHTS_APPROVED
+        )
+        SourceConfiguration.objects.filter(pk=source.pk).update(
+            state=SourceConfiguration.State.ACTIVE,
+            enabled=True,
+        )
+        source.refresh_from_db()
+        return source, rights
+
+    def _legacy_parse_run(
+        self,
+        *,
+        source,
+        rights,
+        suffix,
+        parser_name,
+        parser_contract,
+        schema_fingerprint,
+        provider_code="",
+    ):
+        body_hash = hashlib.sha256(suffix.encode("ascii")).hexdigest()
+        started_at = timezone.now() - timedelta(minutes=2)
+        completed_at = started_at + timedelta(seconds=1)
+        attempt = FetchAttempt.objects.create(
+            source=source,
+            source_revision=source.revision,
+            rights_decision=rights,
+            operation_id=uuid.uuid4(),
+            attempt_number=1,
+            request_fingerprint_revision="LEGACY_JP_FORWARD_TEST_V1",
+            normalized_request_sha256=hashlib.sha256(
+                f"request:{suffix}".encode("ascii")
+            ).hexdigest(),
+            started_at=started_at,
+        )
+        FetchAttempt.objects.filter(pk=attempt.pk).update(
+            completed_at=completed_at,
+            result=FetchAttempt.Result.SUCCEEDED,
+            http_status=200,
+            provider_code=provider_code,
+            response_sha256=body_hash,
+        )
+        attempt.refresh_from_db()
+        artifact = SourceArtifact.objects.create(
+            source=source,
+            body_sha256=body_hash,
+            byte_count=128,
+            first_successful_attempt=attempt,
+        )
+        parse_run = ParseRun.objects.create(
+            artifact=artifact,
+            parser_name=parser_name,
+            parser_version=ParseRun.ParserVersion.V1,
+            parser_contract_fingerprint_sha256=parser_contract,
+            expected_schema_fingerprint_sha256=schema_fingerprint,
+            started_at=completed_at,
+        )
+        ParseRun.objects.filter(pk=parse_run.pk).update(
+            completed_at=completed_at + timedelta(seconds=1),
+            outcome=ParseRun.Outcome.VALIDATED,
+            observed_schema_fingerprint_sha256=schema_fingerprint,
+        )
+        SourceArtifact.objects.filter(pk=artifact.pk).update(
+            state=SourceArtifact.State.REVIEW_REQUIRED
+        )
+        parse_run.refresh_from_db()
+        return parse_run, attempt
+
+    def _legacy_entry(self, source, rights, *, suffix, period):
+        parse_run, attempt = self._legacy_parse_run(
+            source=source,
+            rights=rights,
+            suffix=suffix,
+            parser_name=ParseRun.ParserName.MOFA_ENTRY_CSV,
+            parser_contract=LEGACY_ENTRY_PARSER_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=ENTRY_SCHEMA_FINGERPRINT_SHA256,
+        )
+        basis = f"일반여권 소지자 : legacy {suffix}"
+        return EntryFactRevision.objects.create(
+            country_id=JP_COUNTRY_ID,
+            passport_policy_id=PASSPORT_POLICY_ID,
+            parse_run=parse_run,
+            ordinary_passport_period_text=period,
+            basis_text=basis,
+            snapshot_date=ENTRY_SNAPSHOT_DATE,
+            first_observed_at=attempt.completed_at,
+            typed_fingerprint_sha256=typed_fingerprint_sha256(period, basis),
+        )
+
+    def _legacy_warning(self, source, rights, *, suffix, scope_text):
+        parse_run, attempt = self._legacy_parse_run(
+            source=source,
+            rights=rights,
+            suffix=suffix,
+            parser_name=ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON,
+            parser_contract=LEGACY_PARSER_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=EXPECTED_SCHEMA_FINGERPRINT_SHA256,
+            provider_code=FetchAttempt.ProviderCode.MOFA_SUCCESS_0,
+        )
+        parsed = parse_travel_alarm_jp(
+            warning_payload(item=warning_item(remark=scope_text))
+        )
+        self.assertIsInstance(parsed, TravelWarningParseSuccess)
+        warning = parsed.warning
+        return TravelWarningRevision.objects.create(
+            country_id=JP_COUNTRY_ID,
+            parse_run=parse_run,
+            source_alarm_level_code=warning.source_alarm_level_code,
+            source_scope_type=warning.source_scope_type,
+            source_scope_text=warning.source_scope_text,
+            source_written_date=warning.source_written_date,
+            first_observed_at=attempt.completed_at,
+            typed_fingerprint_sha256=warning.typed_fingerprint_sha256,
+        )
+
+    def test_legacy_jp_history_survives_forward_migrations_and_can_rollback(self):
+        self.addCleanup(lambda: restore_canonical_migration_graph(connection))
+        MigrationExecutor(connection).migrate(
+            [
+                ("entry_requirements", "0001_initial"),
+                ("travel_warnings", "0002_revision_source_rights_guard"),
+                ("reviews", "0002_country_scoped_publications"),
+            ]
+        )
+        Country.objects.get_or_create(
+            id=JP_COUNTRY_ID,
+            defaults={
+                "iso_alpha2": "JP",
+                "name_ko": "일본",
+                "name_en": "Japan",
+                "is_public": True,
+            },
+        )
+        PassportPolicy.objects.get_or_create(
+            id=PASSPORT_POLICY_ID,
+            defaults={
+                "code": PASSPORT_POLICY_CODE,
+                "revision": PASSPORT_POLICY_REVISION,
+            },
+        )
+        entry_source, entry_rights = self._activate_legacy_source(
+            "MOFA_ENTRY_CSV"
+        )
+        warning_source, warning_rights = self._activate_legacy_source(
+            "MOFA_TRAVEL_ALARM_API_JP"
+        )
+        entry_first = self._legacy_entry(
+            entry_source,
+            entry_rights,
+            suffix="ENTRY_FIRST",
+            period="90일",
+        )
+        entry_second = self._legacy_entry(
+            entry_source,
+            entry_rights,
+            suffix="ENTRY_SECOND",
+            period="30일",
+        )
+        warning_first = self._legacy_warning(
+            warning_source,
+            warning_rights,
+            suffix="WARNING_FIRST",
+            scope_text="합성 기존 경보 첫 번째",
+        )
+        warning_second = self._legacy_warning(
+            warning_source,
+            warning_rights,
+            suffix="WARNING_SECOND",
+            scope_text="합성 기존 경보 두 번째",
+        )
+        legacy_ids = {
+            entry_first.pk,
+            entry_second.pk,
+            warning_first.pk,
+            warning_second.pk,
+        }
+        actor = get_user_model().objects.create_superuser(
+            username="legacy-jp-publication-operator",
+            password=None,
+        )
+        first_publications = {}
+        for module, first in (
+            (PublicationModule.ENTRY, entry_first),
+            (PublicationModule.TRAVEL_WARNING, warning_first),
+        ):
+            self.assertEqual(
+                publish_candidate(
+                    module=module,
+                    typed_revision_id=first.pk,
+                    actor=actor,
+                    expected_pointer_version=0,
+                ).code,
+                PublicationCode.PUBLISHED,
+            )
+            first_publications[module] = PublicationRevision.objects.get(
+                module=module,
+                generation=1,
+            ).pk
+
+        restore_canonical_migration_graph(connection)
+
+        self.assertEqual(
+            legacy_ids,
+            {
+                EntryFactRevision.objects.get(pk=entry_first.pk).pk,
+                EntryFactRevision.objects.get(pk=entry_second.pk).pk,
+                TravelWarningRevision.objects.get(pk=warning_first.pk).pk,
+                TravelWarningRevision.objects.get(pk=warning_second.pk).pk,
+            },
+        )
+        self.assertEqual(
+            PublishedEntryFacts.objects.get(
+                country_id=JP_COUNTRY_ID,
+                passport_policy_id=PASSPORT_POLICY_ID,
+            ).current_publication_id,
+            first_publications[PublicationModule.ENTRY],
+        )
+        self.assertEqual(
+            PublishedTravelWarning.objects.get(
+                country_id=JP_COUNTRY_ID,
+            ).current_publication_id,
+            first_publications[PublicationModule.TRAVEL_WARNING],
+        )
+        for module, second in (
+            (PublicationModule.ENTRY, entry_second),
+            (PublicationModule.TRAVEL_WARNING, warning_second),
+        ):
+            first_publication = PublicationRevision.objects.get(
+                pk=first_publications[module]
+            )
+            self.assertEqual(
+                publish_candidate(
+                    module=module,
+                    typed_revision_id=second.pk,
+                    actor=actor,
+                    expected_pointer_version=1,
+                ).code,
+                PublicationCode.PUBLISHED,
+            )
+            self.assertEqual(
+                rollback_publication(
+                    module=module,
+                    target_publication_id=first_publication.pk,
+                    actor=actor,
+                    expected_pointer_version=2,
+                ).code,
+                PublicationCode.ROLLED_BACK,
+            )
+            restored = PublicationRevision.objects.get(
+                module=module,
+                generation=3,
+            )
+            self.assertEqual(restored.rollback_target_id, first_publication.pk)
+            self.assertEqual(
+                restored.source_code_snapshot,
+                (
+                    "MOFA_ENTRY_CSV"
+                    if module == PublicationModule.ENTRY
+                    else "MOFA_TRAVEL_ALARM_API_JP"
+                ),
+            )
+
+
 class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
     reset_sequences = True
 
@@ -366,6 +692,48 @@ class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
         )
         self.assertEqual(PublicationRevision.objects.count(), 2)
 
+    def test_tw_entry_and_warning_publish_to_country_scoped_pointers(self):
+        entry = self.make_entry(
+            country_iso2="TW",
+            country_name="대만",
+        )
+        warning = self.make_warning(
+            country_iso2="TW",
+            country_name_ko="대만",
+            country_name_en="Taiwan",
+        )
+
+        entry_outcome = publish_candidate(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=entry.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+        warning_outcome = publish_candidate(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=warning.id,
+            actor=self.actor,
+            expected_pointer_version=0,
+        )
+
+        self.assertEqual(entry_outcome.code, PublicationCode.PUBLISHED)
+        self.assertEqual(warning_outcome.code, PublicationCode.PUBLISHED)
+        entry_pointer = PublishedEntryFacts.objects.get(
+            country_id=SUPPORTED_COUNTRY_IDS["TW"],
+            passport_policy_id=PASSPORT_POLICY_ID,
+        )
+        warning_pointer = PublishedTravelWarning.objects.get(
+            country_id=SUPPORTED_COUNTRY_IDS["TW"]
+        )
+        self.assertEqual(
+            entry_pointer.current_publication.entry_fact_revision_id,
+            entry.id,
+        )
+        self.assertEqual(
+            warning_pointer.current_publication.travel_warning_revision_id,
+            warning.id,
+        )
+
     def test_reject_is_append_only_review_evidence_without_publication(self):
         entry = self.make_entry()
         warning = self.make_warning()
@@ -463,7 +831,7 @@ class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
         )
         publication = PublicationRevision.objects.get()
         self.assertEqual(publication.review_decision_id, reviews[-1].id)
-        self.assertEqual(publication.source_code_snapshot, "MOFA_ENTRY_CSV")
+        self.assertEqual(publication.source_code_snapshot, ENTRY_SOURCE_CODE)
         self.assertTrue(publication.source_locator_snapshot.startswith("https://"))
         self.assertEqual(
             publication.attribution_text_snapshot,
@@ -515,7 +883,7 @@ class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
 
     def test_inactive_source_and_latest_rights_rejection_fail_closed(self):
         entry = self.make_entry()
-        self.reject_current_rights("MOFA_ENTRY_CSV")
+        self.reject_current_rights(ENTRY_SOURCE_CODE)
 
         rights_outcome = publish_candidate(
             module=PublicationModule.ENTRY,
@@ -533,7 +901,7 @@ class PublicationServiceTests(PublicationFixtureMixin, TransactionTestCase):
 
         warning = self.make_warning()
         warning_source = SourceConfiguration.objects.get(
-            code="MOFA_TRAVEL_ALARM_API_JP"
+            code=WARNING_SOURCE_CODE
         )
         warning_source.state = SourceConfiguration.State.PAUSED
         warning_source.enabled = False
diff --git a/sources/management/commands/register_approved_sources.py b/sources/management/commands/register_approved_sources.py
index 54a32cc..bbd4541 100644
--- a/sources/management/commands/register_approved_sources.py
+++ b/sources/management/commands/register_approved_sources.py
@@ -20,6 +20,13 @@ from travel_windows.contracts import (
     DURATION_SOURCE_LOCATOR,
     DURATION_SOURCE_OWNER,
     DURATION_SOURCE_REVISION,
+    LEGACY_SCHEDULE_ATTRIBUTION,
+    LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_FIELD_SCOPE,
+    LEGACY_SCHEDULE_SOURCE_CODE,
+    LEGACY_SCHEDULE_SOURCE_LOCATOR,
+    LEGACY_SCHEDULE_SOURCE_OWNER,
+    LEGACY_SCHEDULE_SOURCE_REVISION,
     SCHEDULE_ATTRIBUTION,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_FIELD_SCOPE,
@@ -123,6 +130,42 @@ APPROVED_SOURCE_SPECS = (
         decision_basis_code="USER_FOLLOWUP_20260830",
         attribution_text="외교부|공공데이터포털",
     ),
+    ApprovedSourceSpec(
+        code="MOFA_ENTRY_CSV_TRAVEL6",
+        revision="travel6-v1",
+        module=SourceConfiguration.Module.ENTRY,
+        owner="대한민국 외교부 정보화담당관실",
+        official_locator=(
+            "https://www.data.go.kr/cmm/cmm/fileDownload.do?"
+            "atchFileId=FILE_000000003090472&fileDetailSn=1&insertDataPrcus=N"
+        ),
+        secret_reference_name="",
+        access_mode=SourceRightsDecision.AccessMode.ANONYMOUS_PUBLIC,
+        field_scope_code="SUPPORTED_6_ORDINARY_TEXT_V1",
+        contract_fingerprint_sha256=(
+            "2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f"
+        ),
+        decision_basis_code="USER_TRAVEL6_SCOPE_20260831",
+        attribution_text="외교부|공공데이터포털",
+    ),
+    ApprovedSourceSpec(
+        code="MOFA_TRAVEL_ALARM_API_TRAVEL6",
+        revision="travel6-v1",
+        module=SourceConfiguration.Module.TRAVEL_WARNING,
+        owner="대한민국 외교부",
+        official_locator=(
+            "https://apis.data.go.kr/1262000/TravelAlarmService2/"
+            "getTravelAlarmList2"
+        ),
+        secret_reference_name=DATA_GO_SECRET_REFERENCE,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code="SUPPORTED_6_WARNING_V1",
+        contract_fingerprint_sha256=(
+            "5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7"
+        ),
+        decision_basis_code="USER_TRAVEL6_SCOPE_20260831",
+        attribution_text="외교부|공공데이터포털",
+    ),
     ApprovedSourceSpec(
         code=SCHEDULE_SOURCE_CODE,
         revision=SCHEDULE_SOURCE_REVISION,
@@ -149,6 +192,21 @@ APPROVED_SOURCE_SPECS = (
         decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
         attribution_text=CITY_ATTRIBUTION,
     ),
+    ApprovedSourceSpec(
+        code=LEGACY_SCHEDULE_SOURCE_CODE,
+        revision=LEGACY_SCHEDULE_SOURCE_REVISION,
+        module=SourceConfiguration.Module.AVIATION,
+        owner=LEGACY_SCHEDULE_SOURCE_OWNER,
+        official_locator=LEGACY_SCHEDULE_SOURCE_LOCATOR,
+        secret_reference_name=DATA_GO_SECRET_REFERENCE,
+        access_mode=SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
+        field_scope_code=LEGACY_SCHEDULE_FIELD_SCOPE,
+        contract_fingerprint_sha256=(
+            LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+        ),
+        decision_basis_code="TRAVEL_OPPORTUNITY_CONTRACT_20260831",
+        attribution_text=LEGACY_SCHEDULE_ATTRIBUTION,
+    ),
     ApprovedSourceSpec(
         code=DURATION_SOURCE_CODE,
         revision=DURATION_SOURCE_REVISION,
@@ -297,10 +355,15 @@ def _validate_rights_history(
 
 
 def _inspect_spec(spec: ApprovedSourceSpec) -> RegistrationPlan:
+    approved_locator_siblings = {
+        candidate.code
+        for candidate in APPROVED_SOURCE_SPECS
+        if candidate.official_locator == spec.official_locator
+    }
     locator_conflict = (
         SourceConfiguration.objects.select_for_update()
         .filter(official_locator=spec.official_locator)
-        .exclude(code=spec.code)
+        .exclude(code__in=approved_locator_siblings)
         .exists()
     )
     if locator_conflict:
diff --git a/sources/migrations/0011_multi_country_parser_contracts.py b/sources/migrations/0011_multi_country_parser_contracts.py
new file mode 100644
index 0000000..0123486
--- /dev/null
+++ b/sources/migrations/0011_multi_country_parser_contracts.py
@@ -0,0 +1,100 @@
+from django.db import migrations
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+REPLACEMENTS = (
+    (
+        """             AND NEW.parser_contract_fingerprint_sha256 =
+                 '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021'""",
+        """             AND NEW.parser_contract_fingerprint_sha256 IN (
+                 '622a399317ab730f9e9780f51b3ac837073cd99939f07f248323636e97676021',
+                 '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f'
+             )""",
+    ),
+    (
+        """                AND NEW.parser_contract_fingerprint_sha256 =
+                    'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'""",
+        """                AND NEW.parser_contract_fingerprint_sha256 IN (
+                    'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6',
+                    '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+                )""",
+    ),
+)
+
+
+def _rewrite_guard(schema_editor, *, reverse=False):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [FUNCTION_NAME],
+        )
+        rows = cursor.fetchall()
+        if len(rows) != 1:
+            raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+        body = rows[0][0]
+        for forward_old, forward_new in REPLACEMENTS:
+            old, new = (
+                (forward_new, forward_old)
+                if reverse
+                else (forward_old, forward_new)
+            )
+            if body.count(old) != 1:
+                raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+            body = body.replace(old, new)
+        quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $multi_country_parser_contracts$
+            {body}
+            $multi_country_parser_contracts$
+            """
+        )
+
+
+def allow_multi_country_contracts(apps, schema_editor):
+    _rewrite_guard(schema_editor)
+
+
+def restore_legacy_contracts(apps, schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            DO $$
+            BEGIN
+                IF EXISTS (
+                    SELECT 1
+                      FROM sources_parserun
+                     WHERE parser_contract_fingerprint_sha256 IN (
+                        '2fb96548971450bf5538a6d67f27c0305b566514bfc17a9adee594f645e3c75f',
+                        '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+                     )
+                ) THEN
+                    RAISE EXCEPTION
+                        'multi-country parser rollback requires no new parse runs'
+                        USING ERRCODE = 'check_violation';
+                END IF;
+            END;
+            $$;
+            """
+        )
+    _rewrite_guard(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0010_aviation_approved_sources")]
+
+    operations = [
+        migrations.RunPython(
+            allow_multi_country_contracts,
+            restore_legacy_contracts,
+        ),
+    ]
diff --git a/sources/migrations/0012_aviation_reference_contracts.py b/sources/migrations/0012_aviation_reference_contracts.py
new file mode 100644
index 0000000..510a800
--- /dev/null
+++ b/sources/migrations/0012_aviation_reference_contracts.py
@@ -0,0 +1,191 @@
+from django.db import migrations, models
+from django.db.models import Q
+
+
+FUNCTION_NAME = "sources_guard_parse_run_change"
+OLD_ROUTE_CLAUSE = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ROUTE_DURATION_CSV'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '3018365ff3d3549765d5a428a1413ea730071209434def786ab6734f89f8c2ba'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307')"""
+NEW_REFERENCE_CLAUSES = """            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ROUTE_DURATION_CSV'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 IN (
+                    '3018365ff3d3549765d5a428a1413ea730071209434def786ab6734f89f8c2ba',
+                    '430903be2d56367dfc1ca9617e69e6317ff76dadb54912fd02ca1e34d88fee01'
+                )
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '0fe301b62df9abd8b449aeeb1e6ea62cbca80ab7c61e35b6cee552aa58278307')
+            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_DESTINATION_CITY_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    '18a1baa88e4077dbb64a4108620d046b24882dad72e6ec0d56629396391d669f'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    '907670feb0293b0a1a40e6d7065cae2c803e4ba458828b22631dbee002017c18')
+            OR (source_module = 'AVIATION'
+                AND NEW.parser_name = 'ICN_LEGACY_ARRIVALS_JSON'
+                AND NEW.parser_version = 'V1'
+                AND NEW.parser_contract_fingerprint_sha256 =
+                    'a88a91166bb7ea567f59637159f07f2285981f2ae0588e921ec2882e1c73bbfb'
+                AND NEW.expected_schema_fingerprint_sha256 =
+                    'e32b4246a27c5266b52aa0ea24923018eb7d474d09cab1cbcc36ebb85f32ebbd')"""
+
+
+def _function_body(schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            SELECT procedure.prosrc
+              FROM pg_proc AS procedure
+              JOIN pg_namespace AS namespace
+                ON namespace.oid = procedure.pronamespace
+             WHERE namespace.nspname = current_schema()
+               AND procedure.proname = %s
+               AND procedure.pronargs = 0
+            """,
+            [FUNCTION_NAME],
+        )
+        rows = cursor.fetchall()
+    if len(rows) != 1:
+        raise RuntimeError(f"expected one trigger function named {FUNCTION_NAME}")
+    return rows[0][0]
+
+
+def _replace_guard(schema_editor, old, new):
+    body = _function_body(schema_editor)
+    if body.count(old) != 1:
+        raise RuntimeError(f"unexpected {FUNCTION_NAME} trigger definition")
+    body = body.replace(old, new)
+    quoted_name = schema_editor.quote_name(FUNCTION_NAME)
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            f"""
+            CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+            LANGUAGE plpgsql AS $aviation_reference_contracts$
+            {body}
+            $aviation_reference_contracts$
+            """
+        )
+
+
+def allow_aviation_reference_contracts(apps, schema_editor):
+    _replace_guard(schema_editor, OLD_ROUTE_CLAUSE, NEW_REFERENCE_CLAUSES)
+
+
+def restore_prior_aviation_contracts(apps, schema_editor):
+    ParseRun = apps.get_model("sources", "ParseRun")
+    SourceConfiguration = apps.get_model("sources", "SourceConfiguration")
+    if ParseRun.objects.filter(
+        models.Q(
+            parser_name__in=(
+                "ICN_DESTINATION_CITY_JSON",
+                "ICN_LEGACY_ARRIVALS_JSON",
+            )
+        )
+        | models.Q(
+            parser_contract_fingerprint_sha256=(
+                "430903be2d56367dfc1ca9617e69e6317ff76dadb54912fd02ca1e34d88fee01"
+            )
+        )
+    ).exists() or SourceConfiguration.objects.filter(
+        code__in=(
+            "ICN_DESTINATION_CITY_API_15095067",
+            "ICN_LEGACY_ARRIVALS_API",
+            "PORT_LOGISTICS_ROUTE_DURATION_15151728",
+        )
+    ).exists():
+        raise RuntimeError(
+            "aviation reference rollback requires no new source evidence"
+        )
+    _replace_guard(schema_editor, NEW_REFERENCE_CLAUSES, OLD_ROUTE_CLAUSE)
+
+
+class Migration(migrations.Migration):
+    dependencies = [("sources", "0011_multi_country_parser_contracts")]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="sourceconfiguration",
+            name="source_aviation_approved_activation",
+        ),
+        migrations.AddConstraint(
+            model_name="sourceconfiguration",
+            constraint=models.CheckConstraint(
+                condition=(
+                    ~Q(module="AVIATION")
+                    | Q(state="DRAFT", enabled=False)
+                    | Q(
+                        code__in=(
+                            "ICN_SCHEDULE_API",
+                            "ICN_DESTINATION_CITY_API",
+                            "ICN_DESTINATION_CITY_API_15095067",
+                            "ICN_LEGACY_ARRIVALS_API",
+                        ),
+                        revision="travel-v1",
+                        secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+                    )
+                    | Q(
+                        code__in=(
+                            "PORT_LOGISTICS_ROUTE_DURATION",
+                            "PORT_LOGISTICS_ROUTE_DURATION_15151728",
+                        ),
+                        revision="travel-v1",
+                        secret_reference_name="",
+                    )
+                ),
+                name="source_aviation_approved_activation",
+            ),
+        ),
+        migrations.RemoveConstraint(
+            model_name="parserun",
+            name="parse_name_allowlist",
+        ),
+        migrations.AlterField(
+            model_name="parserun",
+            name="parser_name",
+            field=models.CharField(
+                choices=[
+                    ("MOFA_ENTRY_CSV", "MOFA entry CSV"),
+                    ("MOFA_TRAVEL_ALARM_JSON", "MOFA travel alarm JSON"),
+                    (
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ICN scheduled flights JSON",
+                    ),
+                    (
+                        "ICN_DESTINATION_CITY_JSON",
+                        "ICN destination city JSON",
+                    ),
+                    (
+                        "ICN_LEGACY_ARRIVALS_JSON",
+                        "ICN legacy arrivals JSON",
+                    ),
+                    ("ROUTE_DURATION_CSV", "Route duration CSV"),
+                ],
+                max_length=64,
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="parserun",
+            constraint=models.CheckConstraint(
+                condition=Q(
+                    parser_name__in=(
+                        "MOFA_ENTRY_CSV",
+                        "MOFA_TRAVEL_ALARM_JSON",
+                        "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ICN_DESTINATION_CITY_JSON",
+                        "ICN_LEGACY_ARRIVALS_JSON",
+                        "ROUTE_DURATION_CSV",
+                    )
+                ),
+                name="parse_name_allowlist",
+            ),
+        ),
+        migrations.RunPython(
+            allow_aviation_reference_contracts,
+            restore_prior_aviation_contracts,
+        ),
+    ]
diff --git a/sources/models.py b/sources/models.py
index 423631c..ec4538d 100644
--- a/sources/models.py
+++ b/sources/models.py
@@ -42,12 +42,20 @@ class SourceConfiguration(models.Model):
                     ~Q(module="AVIATION")
                     | Q(state="DRAFT", enabled=False)
                     | Q(
-                        code__in=["ICN_SCHEDULE_API", "ICN_DESTINATION_CITY_API"],
+                        code__in=[
+                            "ICN_SCHEDULE_API",
+                            "ICN_DESTINATION_CITY_API",
+                            "ICN_DESTINATION_CITY_API_15095067",
+                            "ICN_LEGACY_ARRIVALS_API",
+                        ],
                         revision="travel-v1",
                         secret_reference_name="DATA_GO_KR_SERVICE_KEY",
                     )
                     | Q(
-                        code="PORT_LOGISTICS_ROUTE_DURATION",
+                        code__in=[
+                            "PORT_LOGISTICS_ROUTE_DURATION",
+                            "PORT_LOGISTICS_ROUTE_DURATION_15151728",
+                        ],
                         revision="travel-v1",
                         secret_reference_name="",
                     )
@@ -238,6 +246,14 @@ class ParseRun(models.Model):
             "ICN_FLIGHT_SCHEDULE_JSON",
             "ICN scheduled flights JSON",
         )
+        ICN_DESTINATION_CITY_JSON = (
+            "ICN_DESTINATION_CITY_JSON",
+            "ICN destination city JSON",
+        )
+        ICN_LEGACY_ARRIVALS_JSON = (
+            "ICN_LEGACY_ARRIVALS_JSON",
+            "ICN legacy arrivals JSON",
+        )
         ROUTE_DURATION_CSV = "ROUTE_DURATION_CSV", "Route duration CSV"
 
     class ParserVersion(models.TextChoices):
@@ -281,6 +297,8 @@ class ParseRun(models.Model):
                         "MOFA_ENTRY_CSV",
                         "MOFA_TRAVEL_ALARM_JSON",
                         "ICN_FLIGHT_SCHEDULE_JSON",
+                        "ICN_DESTINATION_CITY_JSON",
+                        "ICN_LEGACY_ARRIVALS_JSON",
                         "ROUTE_DURATION_CSV",
                     ]
                 ),
diff --git a/sources/tests/test_register_approved_sources.py b/sources/tests/test_register_approved_sources.py
index 27a54de..b16beb9 100644
--- a/sources/tests/test_register_approved_sources.py
+++ b/sources/tests/test_register_approved_sources.py
@@ -15,6 +15,7 @@ from sources.models import SourceConfiguration, SourceRightsDecision
 
 
 SECRET_REFERENCE_NAME = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+CANONICAL_SECRET_REFERENCE_NAME = "DATA_GO_KR_SERVICE_KEY"
 SECRET_SENTINEL = "never-disclose-source-secret-sentinel-20260830"
 
 
@@ -49,6 +50,7 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
     def assert_safe_output(self, output):
         self.assertNotIn("https://", output)
         self.assertNotIn(SECRET_REFERENCE_NAME, output)
+        self.assertNotIn(CANONICAL_SECRET_REFERENCE_NAME, output)
         self.assertNotIn(SECRET_SENTINEL, output)
 
     def make_exact_source(self, spec):
@@ -192,7 +194,113 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             SourceConfiguration.objects.filter(
                 module=SourceConfiguration.Module.AVIATION
             ).count(),
-            3,
+            sum(
+                spec.module == SourceConfiguration.Module.AVIATION
+                for spec in registration.APPROVED_SOURCE_SPECS
+            ),
+        )
+
+    def test_multi_country_siblings_are_append_only_exact_contracts(self):
+        specs = {spec.code: spec for spec in registration.APPROVED_SOURCE_SPECS}
+        legacy_entry = specs["MOFA_ENTRY_CSV"]
+        legacy_warning = specs["MOFA_TRAVEL_ALARM_API_JP"]
+        multi_entry = specs["MOFA_ENTRY_CSV_TRAVEL6"]
+        multi_warning = specs["MOFA_TRAVEL_ALARM_API_TRAVEL6"]
+
+        self.assertEqual(legacy_entry.revision, "rights-v1")
+        self.assertEqual(legacy_entry.field_scope_code, "JP_ORDINARY_TEXT_V1")
+        self.assertEqual(legacy_warning.revision, "rights-v1")
+        self.assertEqual(legacy_warning.field_scope_code, "JP_WARNING_V1")
+        self.assertEqual(
+            legacy_warning.secret_reference_name,
+            SECRET_REFERENCE_NAME,
+        )
+        self.assertEqual(multi_entry.revision, "travel6-v1")
+        self.assertEqual(
+            multi_entry.field_scope_code,
+            "SUPPORTED_6_ORDINARY_TEXT_V1",
+        )
+        self.assertEqual(multi_warning.revision, "travel6-v1")
+        self.assertEqual(
+            multi_warning.field_scope_code,
+            "SUPPORTED_6_WARNING_V1",
+        )
+        self.assertEqual(
+            multi_warning.secret_reference_name,
+            CANONICAL_SECRET_REFERENCE_NAME,
+        )
+        self.assertEqual(legacy_entry.official_locator, multi_entry.official_locator)
+        self.assertEqual(
+            legacy_warning.official_locator,
+            multi_warning.official_locator,
+        )
+
+        legacy_entry_row = self.make_exact_source(legacy_entry)
+        legacy_entry_rights = self.make_exact_approval(
+            legacy_entry_row,
+            legacy_entry,
+        )
+        legacy_warning_row = self.make_exact_source(legacy_warning)
+        legacy_warning_rights = self.make_exact_approval(
+            legacy_warning_row,
+            legacy_warning,
+        )
+        legacy_identity = {
+            legacy_entry.code: (
+                legacy_entry_row.pk,
+                legacy_entry_row.revision,
+                legacy_entry_rights.pk,
+                legacy_entry_rights.field_scope_code,
+            ),
+            legacy_warning.code: (
+                legacy_warning_row.pk,
+                legacy_warning_row.revision,
+                legacy_warning_rights.pk,
+                legacy_warning_rights.field_scope_code,
+            ),
+        }
+
+        self.call_registration("--apply")
+
+        for code, expected in legacy_identity.items():
+            source = SourceConfiguration.objects.get(code=code)
+            rights = source.rights_decisions.get()
+            self.assertEqual(
+                (source.pk, source.revision, rights.pk, rights.field_scope_code),
+                expected,
+            )
+        self.assertTrue(
+            SourceConfiguration.objects.filter(
+                code="MOFA_ENTRY_CSV_TRAVEL6",
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+            ).exists()
+        )
+        self.assertTrue(
+            SourceConfiguration.objects.filter(
+                code="MOFA_TRAVEL_ALARM_API_TRAVEL6",
+                state=SourceConfiguration.State.ACTIVE,
+                enabled=True,
+            ).exists()
+        )
+
+    def test_unregistered_same_locator_source_still_fails_closed(self):
+        entry_spec = registration.APPROVED_SOURCE_SPECS[0]
+        SourceConfiguration.objects.create(
+            code="UNAPPROVED_ENTRY_SIBLING",
+            revision="draft-v1",
+            module=SourceConfiguration.Module.ENTRY,
+            owner="Synthetic owner",
+            official_locator=entry_spec.official_locator,
+        )
+
+        error, _, _ = self.call_registration_failure("--apply")
+
+        self.assertIn("code=SOURCE_LOCATOR_CONFLICT", error)
+        self.assertFalse(
+            SourceConfiguration.objects.filter(
+                code="MOFA_ENTRY_CSV_TRAVEL6"
+            ).exists()
         )
 
     def test_apply_rerun_is_a_noop_and_preserves_immutable_times(self):
@@ -428,7 +536,11 @@ class ApprovedSourceRegistrationCommandTests(TransactionTestCase):
             SourceConfiguration.objects.filter(
                 module=SourceConfiguration.Module.AVIATION
             ).count(),
-            4,
+            1
+            + sum(
+                spec.module == SourceConfiguration.Module.AVIATION
+                for spec in registration.APPROVED_SOURCE_SPECS
+            ),
         )
 
     def test_secret_environment_is_not_read_or_disclosed(self):
diff --git a/sources/tests/test_transport.py b/sources/tests/test_transport.py
index 824bd59..566f321 100644
--- a/sources/tests/test_transport.py
+++ b/sources/tests/test_transport.py
@@ -28,6 +28,7 @@ from sources.transport import (
     WARNING_SOURCE_LOCATOR,
     fetch_entry_attachment,
     fetch_travel_alarm_jp,
+    warning_request_fingerprint,
 )
 
 
@@ -133,10 +134,18 @@ class TransportTestCase(unittest.TestCase):
             connection_factory=FakeFactory(connection),
         )
 
-    def warning_fetch(self, connection, *, secret=None, factory=None):
+    def warning_fetch(
+        self,
+        connection,
+        *,
+        secret=None,
+        factory=None,
+        country_iso2="JP",
+    ):
         if secret is None:
             secret = "%2B".join(("Q7vR4mP2", "zN8kL6xT")) + "%3D"
         return fetch_travel_alarm_jp(
+            country_iso2=country_iso2,
             official_locator=WARNING_SOURCE_LOCATOR,
             secret_reference_name=WARNING_SECRET_REFERENCE,
             secret_value=secret,
@@ -332,6 +341,28 @@ class TransportTestCase(unittest.TestCase):
         self.assertNotIn("%252B", urlsplit(target).query)
         self.assert_sensitive_absent(result, secret)
 
+    def test_warning_country_filter_and_fingerprint_are_country_scoped(self):
+        connection = FakeConnection(FakeResponse(200, warning_body()))
+
+        result = self.warning_fetch(connection, country_iso2="TW")
+
+        parameters = parse_qs(
+            urlsplit(connection.requests[0][1]).query,
+            keep_blank_values=True,
+        )
+        self.assertEqual(
+            parameters["cond[country_iso_alp2::EQ]"],
+            ["TW"],
+        )
+        self.assertEqual(
+            result.request_fingerprint,
+            warning_request_fingerprint("TW"),
+        )
+        self.assertNotEqual(
+            result.request_fingerprint,
+            WARNING_REQUEST_FINGERPRINT,
+        )
+
     def test_warning_rejects_redirect_auth_rate_limit_and_server_error(self):
         cases = (
             (204, ATTEMPT_TERMINAL_FAILED, FAILURE_HTTP_CLIENT),
diff --git a/sources/transport.py b/sources/transport.py
index ed7c9c6..bee9576 100644
--- a/sources/transport.py
+++ b/sources/transport.py
@@ -20,7 +20,9 @@ from urllib.parse import parse_qsl, quote, quote_plus, unquote, urlencode, urlsp
 from xml.etree import ElementTree
 
 from travel_windows.contracts import (
+    CITY_SOURCE_LOCATOR,
     DATA_GO_SECRET_REFERENCE,
+    LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
     SCHEDULE_ARRIVALS_LOCATOR,
     SCHEDULE_DEPARTURES_LOCATOR,
     load_data_go_service_key,
@@ -35,7 +37,8 @@ WARNING_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/1262000/TravelAlarmService2/"
     "getTravelAlarmList2"
 )
-WARNING_SECRET_REFERENCE = "MOFA_TRAVEL_ALARM_SERVICE_KEY"
+WARNING_SECRET_REFERENCE = DATA_GO_SECRET_REFERENCE
+SUPPORTED_WARNING_COUNTRY_ISO2 = frozenset(("JP", "TW", "HK", "MO", "VN", "TH"))
 
 ENTRY_MAX_RESPONSE_BYTES = 262_144
 WARNING_MAX_RESPONSE_BYTES = 4_096
@@ -80,6 +83,17 @@ class SchedulePageFetchResult:
     failure_code: str = ""
 
 
+@dataclass(frozen=True, slots=True)
+class AviationReferenceFetchResult:
+    succeeded: bool
+    city_payload: bytes = field(default=b"", repr=False)
+    legacy_arrival_pages: tuple[bytes, ...] = field(
+        default=(),
+        repr=False,
+    )
+    failure_code: str = ""
+
+
 @dataclass(frozen=True, slots=True)
 class RequestFingerprint:
     revision: str
@@ -160,21 +174,38 @@ ENTRY_REQUEST_FINGERPRINT = RequestFingerprint(
     ),
 )
 
-_WARNING_FIXED_QUERY = (
-    ("returnType", "JSON"),
-    ("numOfRows", "1"),
-    ("pageNo", "1"),
-    ("cond[country_iso_alp2::EQ]", "JP"),
-)
 _warning_parts = urlsplit(WARNING_SOURCE_LOCATOR)
-WARNING_REQUEST_FINGERPRINT = RequestFingerprint(
-    revision="MOFA_WARNING_V1",
-    normalized_request_sha256=_request_hash(
-        _warning_parts.hostname or "",
-        _warning_parts.path,
-        [("ServiceKey", "<redacted>"), *_WARNING_FIXED_QUERY],
-    ),
-)
+
+
+def _warning_fixed_query(country_iso2: str) -> tuple[tuple[str, str], ...]:
+    return (
+        ("returnType", "JSON"),
+        ("numOfRows", "1"),
+        ("pageNo", "1"),
+        ("cond[country_iso_alp2::EQ]", country_iso2),
+    )
+
+
+def warning_request_fingerprint(country_iso2: str) -> RequestFingerprint:
+    if (
+        type(country_iso2) is not str
+        or country_iso2 not in SUPPORTED_WARNING_COUNTRY_ISO2
+    ):
+        raise ValueError("unsupported warning country identity")
+    return RequestFingerprint(
+        revision="MOFA_WARNING_V1",
+        normalized_request_sha256=_request_hash(
+            _warning_parts.hostname or "",
+            _warning_parts.path,
+            [
+                ("ServiceKey", "<redacted>"),
+                *_warning_fixed_query(country_iso2),
+            ],
+        ),
+    )
+
+
+WARNING_REQUEST_FINGERPRINT = warning_request_fingerprint("JP")
 
 _COMMON_REQUEST_HEADERS = {
     "Connection": "close",
@@ -494,6 +525,7 @@ def fetch_entry_attachment(
 
 def fetch_travel_alarm_jp(
     *,
+    country_iso2: str = "JP",
     official_locator: str,
     secret_reference_name: str,
     secret_value: str,
@@ -501,9 +533,12 @@ def fetch_travel_alarm_jp(
     read_timeout_seconds: int | float,
     connection_factory: ConnectionFactory = _default_connection_factory,
 ) -> SingleAttemptResult:
-    """Fetch the approved JP/page-one TravelAlarmService2 request once."""
+    """Fetch one approved country/page-one TravelAlarmService2 request."""
 
-    fingerprint = WARNING_REQUEST_FINGERPRINT
+    try:
+        fingerprint = warning_request_fingerprint(country_iso2)
+    except (TypeError, ValueError):
+        return _failed(WARNING_REQUEST_FINGERPRINT, FAILURE_HTTP_CLIENT)
     if (
         official_locator != WARNING_SOURCE_LOCATOR
         or not _valid_timeout(connect_timeout_seconds)
@@ -521,7 +556,11 @@ def fetch_travel_alarm_jp(
     try:
         decoded_secret = unquote(secret_value, encoding="utf-8", errors="strict")
         query = urlencode(
-            [("ServiceKey", decoded_secret), *_WARNING_FIXED_QUERY], doseq=False
+            [
+                ("ServiceKey", decoded_secret),
+                *_warning_fixed_query(country_iso2),
+            ],
+            doseq=False,
         )
     except (UnicodeError, TypeError, ValueError):
         return _failed(fingerprint, FAILURE_AUTHENTICATION)
@@ -727,3 +766,181 @@ def fetch_data_go_schedule_pages(
         departure_pages=departures,
         arrival_pages=arrivals,
     )
+
+
+def _data_go_wire_failure_code(
+    wire_result: _WireResponse | _WireFailure,
+) -> str:
+    if isinstance(wire_result, _WireFailure):
+        return wire_result.failure_code
+    if wire_result.status in {401, 403}:
+        return FAILURE_AUTHENTICATION
+    if wire_result.status == 429:
+        return FAILURE_RATE_LIMITED
+    if 500 <= wire_result.status <= 599:
+        return FAILURE_UPSTREAM_5XX
+    if wire_result.status != 200:
+        return FAILURE_HTTP_CLIENT
+    return ""
+
+
+def _fetch_destination_city_reference(
+    *,
+    original_secret: str,
+    decoded_secret: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory,
+) -> tuple[bytes, str]:
+    parts = urlsplit(CITY_SOURCE_LOCATOR)
+    query = urlencode(
+        (("serviceKey", decoded_secret), ("type", "json")),
+        doseq=False,
+    )
+    wire_result = _read_once(
+        host=parts.hostname or "",
+        request_target=f"{parts.path}?{query}",
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
+        request_headers={
+            **_COMMON_REQUEST_HEADERS,
+            "Accept": "application/json",
+        },
+        connection_factory=connection_factory,
+    )
+    failure = _data_go_wire_failure_code(wire_result)
+    if failure:
+        return b"", failure
+    if not isinstance(wire_result, _WireResponse):
+        return b"", FAILURE_TRANSPORT
+    if _contains_secret_reflection(
+        wire_result.body,
+        original_secret,
+        decoded_secret,
+    ):
+        return b"", FAILURE_SECRET_REFLECTION
+    if _provider_result_code(wire_result.body) not in {"0", "00"}:
+        return b"", FAILURE_PROVIDER_ERROR
+    return wire_result.body, ""
+
+
+def _fetch_legacy_arrival_pages(
+    *,
+    original_secret: str,
+    decoded_secret: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory,
+) -> tuple[tuple[bytes, ...], str]:
+    parts = urlsplit(LEGACY_SCHEDULE_ARRIVALS_LOCATOR)
+    pages: list[bytes] = []
+    expected_pages: int | None = None
+    for page_number in range(1, SCHEDULE_MAX_PAGES_PER_DIRECTION + 1):
+        query = urlencode(
+            (
+                ("serviceKey", decoded_secret),
+                ("numOfRows", str(SCHEDULE_PAGE_SIZE)),
+                ("pageNo", str(page_number)),
+                ("lang", "K"),
+                ("type", "json"),
+            ),
+            doseq=False,
+        )
+        wire_result = _read_once(
+            host=parts.hostname or "",
+            request_target=f"{parts.path}?{query}",
+            connect_timeout_seconds=connect_timeout_seconds,
+            read_timeout_seconds=read_timeout_seconds,
+            max_response_bytes=SCHEDULE_PAGE_MAX_RESPONSE_BYTES,
+            request_headers={
+                **_COMMON_REQUEST_HEADERS,
+                "Accept": "application/json",
+            },
+            connection_factory=connection_factory,
+        )
+        failure = _data_go_wire_failure_code(wire_result)
+        if failure:
+            return (), failure
+        if not isinstance(wire_result, _WireResponse):
+            return (), FAILURE_TRANSPORT
+        if _contains_secret_reflection(
+            wire_result.body,
+            original_secret,
+            decoded_secret,
+        ):
+            return (), FAILURE_SECRET_REFLECTION
+        if _provider_result_code(wire_result.body) not in {"0", "00"}:
+            return (), FAILURE_PROVIDER_ERROR
+        page_count = _schedule_page_count(wire_result.body)
+        if (
+            page_count is None
+            or page_count > SCHEDULE_MAX_PAGES_PER_DIRECTION
+        ):
+            return (), FAILURE_PROVIDER_ERROR
+        if expected_pages is None:
+            expected_pages = page_count
+        elif expected_pages != page_count:
+            return (), FAILURE_PROVIDER_ERROR
+        pages.append(wire_result.body)
+        if page_number == expected_pages:
+            return tuple(pages), ""
+    return (), FAILURE_RESPONSE_TOO_LARGE
+
+
+def fetch_data_go_reference_evidence(
+    *,
+    secret_reference_name: str,
+    secret_value: str,
+    connect_timeout_seconds: int | float,
+    read_timeout_seconds: int | float,
+    connection_factory: ConnectionFactory = _default_connection_factory,
+) -> AviationReferenceFetchResult:
+    """Fetch 15095067 cities and complete 15095059 arrival pages in memory."""
+
+    if (
+        secret_reference_name != DATA_GO_SECRET_REFERENCE
+        or not isinstance(secret_value, str)
+        or not secret_value
+        or len(secret_value) > _MAX_SECRET_CHARACTERS
+        or not _valid_timeout(connect_timeout_seconds)
+        or not _valid_timeout(read_timeout_seconds)
+    ):
+        return AviationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_AUTHENTICATION,
+        )
+    try:
+        decoded_secret = unquote(
+            secret_value,
+            encoding="utf-8",
+            errors="strict",
+        )
+    except (UnicodeError, TypeError, ValueError):
+        return AviationReferenceFetchResult(
+            False,
+            failure_code=FAILURE_AUTHENTICATION,
+        )
+    city_payload, failure = _fetch_destination_city_reference(
+        original_secret=secret_value,
+        decoded_secret=decoded_secret,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+    )
+    if failure:
+        return AviationReferenceFetchResult(False, failure_code=failure)
+    legacy_pages, failure = _fetch_legacy_arrival_pages(
+        original_secret=secret_value,
+        decoded_secret=decoded_secret,
+        connect_timeout_seconds=connect_timeout_seconds,
+        read_timeout_seconds=read_timeout_seconds,
+        connection_factory=connection_factory,
+    )
+    if failure:
+        return AviationReferenceFetchResult(False, failure_code=failure)
+    return AviationReferenceFetchResult(
+        True,
+        city_payload=city_payload,
+        legacy_arrival_pages=legacy_pages,
+    )
diff --git a/travel_warnings/ingestion.py b/travel_warnings/ingestion.py
index 1d876b9..3e74c67 100644
--- a/travel_warnings/ingestion.py
+++ b/travel_warnings/ingestion.py
@@ -1,4 +1,4 @@
-"""Closed ingestion boundary for the approved JP travel-warning source.
+"""Closed ingestion boundary for the approved travel-warning source.
 
 Every network attempt is surrounded by a durable redacted receipt. Successful
 bytes remain in memory only until their hash, schema, and typed projection have
@@ -19,7 +19,7 @@ from typing import Callable
 from django.db import connection, transaction
 from django.utils import timezone
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import SUPPORTED_COUNTRY_IDS, SUPPORTED_COUNTRY_ROWS, Country
 from sources.models import (
     FetchAttempt,
     ParseRun,
@@ -49,7 +49,9 @@ from sources.transport import (
     WARNING_SOURCE_LOCATOR,
     SingleAttemptResult,
     fetch_travel_alarm_jp,
+    warning_request_fingerprint,
 )
+from travel_windows.contracts import load_data_go_service_key
 
 from .models import (
     SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
@@ -68,18 +70,35 @@ from .parser import (
 )
 
 
-WARNING_SOURCE_CODE = "MOFA_TRAVEL_ALARM_API_JP"
-WARNING_SOURCE_REVISION = "rights-v1"
+LEGACY_WARNING_SOURCE_CODE = "MOFA_TRAVEL_ALARM_API_JP"
+LEGACY_WARNING_SOURCE_REVISION = "rights-v1"
+LEGACY_WARNING_FIELD_SCOPE = "JP_WARNING_V1"
+LEGACY_WARNING_CONTRACT_FINGERPRINT_SHA256 = (
+    "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
+)
+MULTI_COUNTRY_WARNING_SOURCE_CODE = "MOFA_TRAVEL_ALARM_API_TRAVEL6"
+MULTI_COUNTRY_WARNING_SOURCE_REVISION = "travel6-v1"
+MULTI_COUNTRY_WARNING_FIELD_SCOPE = "SUPPORTED_6_WARNING_V1"
+MULTI_COUNTRY_WARNING_CONTRACT_FINGERPRINT_SHA256 = (
+    PARSER_CONTRACT_FINGERPRINT_SHA256
+)
+WARNING_SOURCE_CODE = MULTI_COUNTRY_WARNING_SOURCE_CODE
+WARNING_SOURCE_REVISION = MULTI_COUNTRY_WARNING_SOURCE_REVISION
 WARNING_SOURCE_MODULE = SourceConfiguration.Module.TRAVEL_WARNING
 WARNING_SOURCE_OWNER = "대한민국 외교부"
-WARNING_FIELD_SCOPE = "JP_WARNING_V1"
+WARNING_FIELD_SCOPE = MULTI_COUNTRY_WARNING_FIELD_SCOPE
 WARNING_ATTRIBUTION = "외교부|공공데이터포털"
 WARNING_DECIDED_BY = "PROJECT_OWNER_REQUEST"
-WARNING_DECISION_BASIS = "USER_FOLLOWUP_20260830"
+WARNING_DECISION_BASIS = "USER_TRAVEL6_SCOPE_20260831"
 WARNING_PARSER_NAME = ParseRun.ParserName.MOFA_TRAVEL_ALARM_JSON
 WARNING_PARSER_VERSION = ParseRun.ParserVersion.V1
 WARNING_INGESTION_LOCK_NAMESPACE = 1_414_678_614
 WARNING_INGESTION_LOCK_KEY = 1_006_002
+_SUPPORTED_COUNTRY_IDENTITIES = {
+    iso_alpha2: (name_ko, name_en)
+    for _country_id, iso_alpha2, name_ko, name_en, _is_public
+    in SUPPORTED_COUNTRY_ROWS
+}
 
 
 class TravelWarningIngestionCode:
@@ -178,9 +197,11 @@ def _raise_sanitized_base_exception(kind: str) -> None:
 
 
 def _environment_secret_loader(reference_name: str) -> object:
-    """Read only the named process environment variable; never a dotenv file."""
+    """Resolve the canonical process key with its documented legacy fallback."""
 
-    return os.environ.get(reference_name)
+    if reference_name != WARNING_SECRET_REFERENCE:
+        return None
+    return load_data_go_service_key(os.environ)
 
 
 def _load_secret(
@@ -349,18 +370,20 @@ def _locked_approved_source() -> tuple[SourceConfiguration, SourceRightsDecision
 def _create_started_attempt(
     operation_id: uuid.UUID,
     attempt_number: int,
+    country_iso2: str,
 ) -> tuple[FetchAttempt, _FetchConfiguration]:
     with transaction.atomic(durable=True):
         source, rights = _locked_approved_source()
+        request_fingerprint = warning_request_fingerprint(country_iso2)
         attempt = FetchAttempt.objects.create(
             source=source,
             source_revision=source.revision,
             rights_decision=rights,
             operation_id=operation_id,
             attempt_number=attempt_number,
-            request_fingerprint_revision=WARNING_REQUEST_FINGERPRINT.revision,
+            request_fingerprint_revision=request_fingerprint.revision,
             normalized_request_sha256=(
-                WARNING_REQUEST_FINGERPRINT.normalized_request_sha256
+                request_fingerprint.normalized_request_sha256
             ),
         )
         configuration = _FetchConfiguration(
@@ -386,23 +409,26 @@ def _worker_interrupted_result() -> _VerifiedTransportResult:
     )
 
 
-def _missing_secret_result() -> SingleAttemptResult:
+def _missing_secret_result(country_iso2: str) -> SingleAttemptResult:
     return SingleAttemptResult(
-        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        request_fingerprint=warning_request_fingerprint(country_iso2),
         attempt_result=ATTEMPT_TERMINAL_FAILED,
         failure_code=FAILURE_AUTHENTICATION,
     )
 
 
-def _verify_transport_result(result: object) -> _VerifiedTransportResult:
+def _verify_transport_result(
+    result: object,
+    expected_fingerprint=WARNING_REQUEST_FINGERPRINT,
+) -> _VerifiedTransportResult:
     if type(result) is not SingleAttemptResult:
         return _worker_interrupted_result()
     if (
-        type(result.request_fingerprint) is not type(WARNING_REQUEST_FINGERPRINT)
+        type(result.request_fingerprint) is not type(expected_fingerprint)
         or result.request_fingerprint.revision
-        != WARNING_REQUEST_FINGERPRINT.revision
+        != expected_fingerprint.revision
         or result.request_fingerprint.normalized_request_sha256
-        != WARNING_REQUEST_FINGERPRINT.normalized_request_sha256
+        != expected_fingerprint.normalized_request_sha256
         or type(result.attempt_result) is not str
         or (
             result.http_status is not None
@@ -587,21 +613,29 @@ def _is_sha256(value: object) -> bool:
 def _safe_parse(
     body: bytes,
     parser: Callable[[bytes], TravelWarningParseResult],
+    country_iso2: str,
 ) -> _VerifiedParseResult:
     try:
-        result = parser(body)
+        result = (
+            parser(body, country_iso2=country_iso2)
+            if parser is parse_travel_alarm_jp
+            else parser(body)
+        )
         if type(result) is TravelWarningParseSuccess:
             warning = result.warning
+            expected_name_ko, expected_name_en = (
+                _SUPPORTED_COUNTRY_IDENTITIES[country_iso2]
+            )
             if (
                 type(warning) is ParsedTravelWarning
                 and result.observed_schema_fingerprint_sha256
                 == EXPECTED_SCHEMA_FINGERPRINT_SHA256
                 and type(warning.country_iso2) is str
-                and warning.country_iso2 == "JP"
+                and warning.country_iso2 == country_iso2
                 and type(warning.country_name_ko) is str
-                and warning.country_name_ko == "일본"
+                and warning.country_name_ko == expected_name_ko
                 and type(warning.country_name_en) is str
-                and warning.country_name_en == "Japan"
+                and warning.country_name_en == expected_name_en
                 and type(warning.source_alarm_level_code) is str
                 and bool(warning.source_alarm_level_code.strip())
                 and len(warning.source_alarm_level_code)
@@ -690,7 +724,11 @@ def _safe_parse(
     return _internal_parse_failure()
 
 
-def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
+def _replay_is_consistent(
+    artifact: SourceArtifact,
+    byte_count: int,
+    country_iso2: str,
+) -> bool:
     if (
         artifact.byte_count != byte_count
         or artifact.state != SourceArtifact.State.REVIEW_REQUIRED
@@ -714,7 +752,11 @@ def _replay_is_consistent(artifact: SourceArtifact, byte_count: int) -> bool:
         == EXPECTED_SCHEMA_FINGERPRINT_SHA256
         and run.observed_schema_fingerprint_sha256
         == EXPECTED_SCHEMA_FINGERPRINT_SHA256
-        and TravelWarningRevision.objects.filter(parse_run=run).count() == 1
+        and TravelWarningRevision.objects.filter(
+            parse_run=run,
+            country__iso_alpha2=country_iso2,
+        ).count()
+        == 1
     )
 
 
@@ -722,6 +764,7 @@ def _persist_success(
     attempt: FetchAttempt,
     verified: _VerifiedTransportResult,
     parser: Callable[[bytes], TravelWarningParseResult],
+    country_iso2: str,
 ) -> str:
     try:
         with transaction.atomic(durable=True):
@@ -761,7 +804,11 @@ def _persist_success(
                 .first()
             )
             if existing is not None:
-                if not _replay_is_consistent(existing, verified.byte_count):
+                if not _replay_is_consistent(
+                    existing,
+                    verified.byte_count,
+                    country_iso2,
+                ):
                     raise _ClosedPersistenceFailure(
                         TravelWarningIngestionCode.EVIDENCE_CONFLICT
                     )
@@ -808,7 +855,11 @@ def _persist_success(
                 ),
                 started_at=max(timezone.now(), anchor.completed_at),
             )
-            parse_result = _safe_parse(verified.body, parser)
+            parse_result = _safe_parse(
+                verified.body,
+                parser,
+                country_iso2,
+            )
             completed_at = max(timezone.now(), parse_run.started_at)
             updated = ParseRun.objects.filter(
                 pk=parse_run.id,
@@ -858,7 +909,10 @@ def _persist_success(
                 raise _ClosedPersistenceFailure(
                     TravelWarningIngestionCode.SOURCE_GATE_FAILED
                 )
-            country = Country.objects.get(pk=JP_COUNTRY_ID)
+            country = Country.objects.get(
+                pk=SUPPORTED_COUNTRY_IDS[country_iso2],
+                iso_alpha2=country_iso2,
+            )
             TravelWarningRevision.objects.create(
                 country=country,
                 parse_run=parse_run,
@@ -880,6 +934,7 @@ def _persist_success(
 
 def _run_travel_warning_ingestion(
     *,
+    country_iso2: str,
     transport: Callable[..., SingleAttemptResult],
     parser: Callable[[bytes], TravelWarningParseResult],
     retry_wait: Callable[[int], None],
@@ -923,6 +978,7 @@ def _run_travel_warning_ingestion(
                 attempt, configuration = _create_started_attempt(
                     operation_id,
                     attempt_count,
+                    country_iso2,
                 )
             except _ClosedPersistenceFailure as failure:
                 return TravelWarningIngestionOutcome(
@@ -948,9 +1004,10 @@ def _run_travel_warning_ingestion(
                     secret_loader,
                 )
                 if loaded_secret is None:
-                    raw_result: object = _missing_secret_result()
+                    raw_result: object = _missing_secret_result(country_iso2)
                 else:
                     raw_result = transport(
+                        country_iso2=country_iso2,
                         official_locator=WARNING_SOURCE_LOCATOR,
                         secret_reference_name=(
                             configuration.secret_reference_name
@@ -973,7 +1030,10 @@ def _run_travel_warning_ingestion(
                 loaded_secret = None
 
             try:
-                verified = _verify_transport_result(raw_result)
+                verified = _verify_transport_result(
+                    raw_result,
+                    warning_request_fingerprint(country_iso2),
+                )
             except Exception:
                 verified = _worker_interrupted_result()
             except BaseException as exception:
@@ -1016,7 +1076,12 @@ def _run_travel_warning_ingestion(
 
             if verified.attempt_result == ATTEMPT_SUCCEEDED:
                 try:
-                    code = _persist_success(closed_attempt, verified, parser)
+                    code = _persist_success(
+                        closed_attempt,
+                        verified,
+                        parser,
+                        country_iso2,
+                    )
                 except _ClosedPersistenceFailure as failure:
                     code = failure.code
                 return TravelWarningIngestionOutcome(code, attempt_count)
@@ -1057,6 +1122,7 @@ def _run_travel_warning_ingestion(
 
 def ingest_travel_warning(
     *,
+    country_iso2: str = "JP",
     transport: Callable[..., SingleAttemptResult] = fetch_travel_alarm_jp,
     parser: Callable[[bytes], TravelWarningParseResult] = parse_travel_alarm_jp,
     retry_wait: Callable[[int], None] = _bounded_retry_wait,
@@ -1064,7 +1130,17 @@ def ingest_travel_warning(
 ) -> TravelWarningIngestionOutcome:
     """Run one redacted operation, including only its permitted retries."""
 
+    if (
+        type(country_iso2) is not str
+        or country_iso2 not in SUPPORTED_COUNTRY_IDS
+    ):
+        return TravelWarningIngestionOutcome(
+            TravelWarningIngestionCode.SOURCE_GATE_FAILED,
+            0,
+        )
+
     result = _run_travel_warning_ingestion(
+        country_iso2=country_iso2,
         transport=transport,
         parser=parser,
         retry_wait=retry_wait,
diff --git a/travel_warnings/migrations/0003_country_scoped_warning_revisions.py b/travel_warnings/migrations/0003_country_scoped_warning_revisions.py
new file mode 100644
index 0000000..f49bcb0
--- /dev/null
+++ b/travel_warnings/migrations/0003_country_scoped_warning_revisions.py
@@ -0,0 +1,200 @@
+import uuid
+
+from django.db import migrations, models
+
+
+SUPPORTED_COUNTRY_IDS = (
+    uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9"),
+    uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"),
+    uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"),
+    uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"),
+    uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"),
+    uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"),
+)
+
+SUPPORTED_SQL = """(
+           '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid,
+           '3d374024-be31-5be3-99b3-fc28626b076a'::uuid,
+           '008d7e8f-412e-53ca-a5c6-d06a9fbafda8'::uuid,
+           '55d20bb0-9d97-5a53-9600-e8f102f38fe9'::uuid,
+           '17e47e71-07e3-57e6-8c72-e1f8b47e34df'::uuid,
+           '5438e3c3-df2b-593a-8f04-7e64e66219e7'::uuid
+       )"""
+
+
+FUNCTION_REPLACEMENTS = {
+    "travel_warnings_guard_revision_change": (
+        (
+            """    IF NEW.country_id <> '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid THEN
+        RAISE EXCEPTION 'a travel warning revision requires the approved JP identity'""",
+            f"""    IF NEW.country_id NOT IN {SUPPORTED_SQL} THEN
+        RAISE EXCEPTION 'a travel warning revision requires a supported identity'""",
+        ),
+        (
+            """        '{"country_iso2":' || to_jsonb('JP'::text)::text ||
+        ',"country_name_en":' || to_jsonb('Japan'::text)::text ||
+        ',"country_name_ko":' || to_jsonb('일본'::text)::text ||""",
+            """        '{"country_iso2":' || to_jsonb((
+            SELECT iso_alpha2 FROM countries_country
+             WHERE id = NEW.country_id
+        ))::text ||
+        ',"country_name_en":' || to_jsonb((
+            SELECT name_en FROM countries_country
+             WHERE id = NEW.country_id
+        ))::text ||
+        ',"country_name_ko":' || to_jsonb((
+            SELECT name_ko FROM countries_country
+             WHERE id = NEW.country_id
+        ))::text ||""",
+        ),
+        (
+            """       OR source_code <> 'MOFA_TRAVEL_ALARM_API_JP' THEN""",
+            """       OR source_code <> 'MOFA_TRAVEL_ALARM_API_TRAVEL6' THEN""",
+        ),
+        (
+            """       OR parser_contract <>
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6'""",
+            """       OR parser_contract <>
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'""",
+        ),
+    ),
+    "travel_warnings_validate_revision_source_rights": (
+        (
+            """BEGIN
+    SELECT artifact.source_id, artifact.body_sha256,""",
+            f"""BEGIN
+    IF NEW.country_id NOT IN {SUPPORTED_SQL} THEN
+        RAISE EXCEPTION 'travel warning rights require a supported identity'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT artifact.source_id, artifact.body_sha256,""",
+        ),
+        (
+            """       OR source_secret_reference IS DISTINCT FROM
+          'MOFA_TRAVEL_ALARM_SERVICE_KEY' THEN""",
+            """       OR source_secret_reference IS DISTINCT FROM
+          'DATA_GO_KR_SERVICE_KEY' THEN""",
+        ),
+        (
+            """       OR source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_JP'
+       OR active_source_revision IS DISTINCT FROM 'rights-v1'""",
+            """       OR source_code IS DISTINCT FROM 'MOFA_TRAVEL_ALARM_API_TRAVEL6'
+       OR active_source_revision IS DISTINCT FROM 'travel6-v1'""",
+        ),
+        (
+            """       OR latest_rights_revision IS DISTINCT FROM 'rights-v1'""",
+            """       OR latest_rights_revision IS DISTINCT FROM 'travel6-v1'""",
+        ),
+        (
+            """       OR latest_field_scope IS DISTINCT FROM 'JP_WARNING_V1'""",
+            """       OR latest_field_scope IS DISTINCT FROM 'SUPPORTED_6_WARNING_V1'""",
+        ),
+        (
+            """       OR latest_contract_fingerprint IS DISTINCT FROM
+          'c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6' THEN""",
+            """       OR latest_contract_fingerprint IS DISTINCT FROM
+          '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7' THEN""",
+        ),
+    ),
+}
+
+
+def _rewrite_functions(schema_editor, *, reverse=False):
+    with schema_editor.connection.cursor() as cursor:
+        for function_name, replacements in FUNCTION_REPLACEMENTS.items():
+            cursor.execute(
+                """
+                SELECT procedure.prosrc
+                  FROM pg_proc AS procedure
+                  JOIN pg_namespace AS namespace
+                    ON namespace.oid = procedure.pronamespace
+                 WHERE namespace.nspname = current_schema()
+                   AND procedure.proname = %s
+                   AND procedure.pronargs = 0
+                """,
+                [function_name],
+            )
+            rows = cursor.fetchall()
+            if len(rows) != 1:
+                raise RuntimeError(
+                    f"expected one trigger function named {function_name}"
+                )
+            body = rows[0][0]
+            for forward_old, forward_new in replacements:
+                old, new = (
+                    (forward_new, forward_old)
+                    if reverse
+                    else (forward_old, forward_new)
+                )
+                if body.count(old) != 1:
+                    raise RuntimeError(
+                        f"unexpected {function_name} trigger definition"
+                    )
+                body = body.replace(old, new)
+            quoted_name = schema_editor.quote_name(function_name)
+            cursor.execute(
+                f"""
+                CREATE OR REPLACE FUNCTION {quoted_name}() RETURNS trigger
+                LANGUAGE plpgsql AS $warning_country_scope$
+                {body}
+                $warning_country_scope$
+                """
+            )
+
+
+def generalize_warning_guards(apps, schema_editor):
+    _rewrite_functions(schema_editor)
+
+
+def restore_warning_guards(apps, schema_editor):
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            """
+            LOCK TABLE travel_warnings_travelwarningrevision
+                IN ACCESS EXCLUSIVE MODE;
+            DO $$
+            BEGIN
+                IF EXISTS (
+                    SELECT 1
+                      FROM travel_warnings_travelwarningrevision AS warning
+                      JOIN sources_parserun AS parse_run
+                        ON parse_run.id = warning.parse_run_id
+                      JOIN sources_sourceartifact AS artifact
+                        ON artifact.id = parse_run.artifact_id
+                      JOIN sources_sourceconfiguration AS source
+                        ON source.id = artifact.source_id
+                     WHERE warning.country_id <>
+                           '575fa8b9-14f9-526e-9464-ebd1dea76da9'::uuid
+                        OR source.code = 'MOFA_TRAVEL_ALARM_API_TRAVEL6'
+                        OR parse_run.parser_contract_fingerprint_sha256 =
+                           '5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7'
+                ) THEN
+                    RAISE EXCEPTION
+                        'warning country-scope rollback requires legacy JP revisions'
+                        USING ERRCODE = 'check_violation';
+                END IF;
+            END;
+            $$;
+            """
+        )
+    _rewrite_functions(schema_editor, reverse=True)
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("countries", "0003_supported_country_allowlist"),
+        ("sources", "0011_multi_country_parser_contracts"),
+        ("travel_warnings", "0002_revision_source_rights_guard"),
+    ]
+
+    operations = [
+        migrations.AddConstraint(
+            model_name="travelwarningrevision",
+            constraint=models.CheckConstraint(
+                condition=models.Q(country_id__in=SUPPORTED_COUNTRY_IDS),
+                name="warning_country_supported",
+            ),
+        ),
+        migrations.RunPython(generalize_warning_guards, restore_warning_guards),
+    ]
diff --git a/travel_warnings/models.py b/travel_warnings/models.py
index 28b63d2..582828f 100644
--- a/travel_warnings/models.py
+++ b/travel_warnings/models.py
@@ -3,6 +3,8 @@ import uuid
 from django.db import models
 from django.db.models import F, Q
 
+from countries.models import SUPPORTED_COUNTRY_IDS
+
 
 SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH = 32
 SOURCE_SCOPE_TYPE_MAX_LENGTH = 100
@@ -33,6 +35,12 @@ class TravelWarningRevision(models.Model):
 
     class Meta:
         constraints = [
+            models.CheckConstraint(
+                condition=Q(
+                    country_id__in=tuple(SUPPORTED_COUNTRY_IDS.values())
+                ),
+                name="warning_country_supported",
+            ),
             models.CheckConstraint(
                 condition=~Q(source_alarm_level_code=""),
                 name="warning_level_code_present",
diff --git a/travel_warnings/parser.py b/travel_warnings/parser.py
index e3c013e..717bfa4 100644
--- a/travel_warnings/parser.py
+++ b/travel_warnings/parser.py
@@ -7,6 +7,7 @@ from dataclasses import dataclass
 from datetime import date
 from typing import Any
 
+from countries.models import SUPPORTED_COUNTRY_ROWS
 from sources.models import ParseRun
 from travel_warnings.models import (
     SOURCE_ALARM_LEVEL_CODE_MAX_LENGTH,
@@ -16,9 +17,20 @@ from travel_warnings.models import (
 
 
 MAX_RESPONSE_BYTES = 4096
-PARSER_CONTRACT_FINGERPRINT_SHA256 = (
+LEGACY_PARSER_CONTRACT_FINGERPRINT_SHA256 = (
     "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
 )
+MULTI_COUNTRY_PARSER_CONTRACT_TEXT = (
+    "MOFA TravelAlarm JSON V1|exact schema|explicit country filter|"
+    "supported countries JP,TW,HK,MO,VN,TH|typed alarm level, scope, "
+    "written date|raw body retention zero"
+)
+MULTI_COUNTRY_PARSER_CONTRACT_FINGERPRINT_SHA256 = hashlib.sha256(
+    MULTI_COUNTRY_PARSER_CONTRACT_TEXT.encode("utf-8")
+).hexdigest()
+PARSER_CONTRACT_FINGERPRINT_SHA256 = (
+    MULTI_COUNTRY_PARSER_CONTRACT_FINGERPRINT_SHA256
+)
 EXPECTED_SCHEMA_FINGERPRINT_SHA256 = (
     "64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b"
 )
@@ -49,6 +61,11 @@ _WRITTEN_DATE_PATH = (
     "[]",
     "written_dt",
 )
+_SUPPORTED_COUNTRY_IDENTITIES = {
+    iso_alpha2: (name_ko, name_en)
+    for _country_id, iso_alpha2, name_ko, name_en, _is_public
+    in SUPPORTED_COUNTRY_ROWS
+}
 
 
 class _DuplicateKey(ValueError):
@@ -169,9 +186,16 @@ def _typed_fingerprint(item: dict[str, Any], written_date: date | None) -> str:
     )
 
 
-def parse_travel_alarm_jp(payload: bytes) -> TravelWarningParseResult:
-    if not isinstance(payload, bytes):
+def parse_travel_alarm_jp(
+    payload: bytes,
+    *,
+    country_iso2: str = "JP",
+) -> TravelWarningParseResult:
+    if not isinstance(payload, bytes) or type(country_iso2) is not str:
         return _failure(ParseRun.FailureCode.INTERNAL_PARSER_ERROR)
+    expected_identity = _SUPPORTED_COUNTRY_IDENTITIES.get(country_iso2)
+    if expected_identity is None:
+        return _failure(ParseRun.FailureCode.IDENTITY_MISMATCH)
     if len(payload) > MAX_RESPONSE_BYTES:
         return _failure(ParseRun.FailureCode.SYNTAX_ERROR)
     try:
@@ -210,10 +234,11 @@ def parse_travel_alarm_jp(payload: bytes) -> TravelWarningParseResult:
     item = items[0]
     if set(item) != ITEM_KEYS:
         return _failure(ParseRun.FailureCode.SCHEMA_MISMATCH, observed_schema)
+    expected_name_ko, expected_name_en = expected_identity
     if (
-        item["country_iso_alp2"] != "JP"
-        or item["country_nm"] != "일본"
-        or item["country_eng_nm"] != "Japan"
+        item["country_iso_alp2"] != country_iso2
+        or item["country_nm"] != expected_name_ko
+        or item["country_eng_nm"] != expected_name_en
     ):
         return _failure(ParseRun.FailureCode.IDENTITY_MISMATCH, observed_schema)
     if any(
diff --git a/travel_warnings/tests/test_ingestion.py b/travel_warnings/tests/test_ingestion.py
index d2f927a..397065e 100644
--- a/travel_warnings/tests/test_ingestion.py
+++ b/travel_warnings/tests/test_ingestion.py
@@ -11,7 +11,7 @@ from django.core.management.base import CommandError
 from django.db import connection, models
 from django.test import SimpleTestCase, TransactionTestCase
 
-from countries.models import JP_COUNTRY_ID, Country
+from countries.models import JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS, Country
 from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
@@ -38,11 +38,13 @@ from sources.transport import (
     WARNING_SECRET_REFERENCE,
     WARNING_SOURCE_LOCATOR,
     SingleAttemptResult,
+    warning_request_fingerprint,
 )
 import travel_warnings.ingestion as warning_ingestion
 from travel_warnings.ingestion import (
     WARNING_INGESTION_LOCK_KEY,
     WARNING_INGESTION_LOCK_NAMESPACE,
+    WARNING_SOURCE_CODE,
     TravelWarningIngestionCode,
     TravelWarningIngestionOutcome,
     _LoadedSecret,
@@ -125,9 +127,10 @@ def successful_result(
     payload: bytes,
     *,
     provider_code: str = PROVIDER_SUCCESS_0,
+    country_iso2: str = "JP",
 ) -> SingleAttemptResult:
     return SingleAttemptResult(
-        request_fingerprint=WARNING_REQUEST_FINGERPRINT,
+        request_fingerprint=warning_request_fingerprint(country_iso2),
         attempt_result=ATTEMPT_SUCCEEDED,
         http_status=200,
         provider_code=provider_code,
@@ -253,7 +256,7 @@ class TravelWarningIngestionTests(TransactionTestCase):
 
     def reject_warning_rights(self):
         source = SourceConfiguration.objects.get(
-            code="MOFA_TRAVEL_ALARM_API_JP"
+            code=WARNING_SOURCE_CODE
         )
         approval = source.rights_decisions.get(decision_sequence=1)
         return SourceRightsDecision.objects.create(
@@ -303,6 +306,7 @@ class TravelWarningIngestionTests(TransactionTestCase):
         self.assertEqual(
             names,
             {
+                "country_iso2",
                 "official_locator",
                 "secret_reference_name",
                 "secret_value",
@@ -336,6 +340,38 @@ class TravelWarningIngestionTests(TransactionTestCase):
         self.assertEqual(revision.source_scope_text, SCOPE_TEXT)
         self.assert_warning_lock_available()
 
+    def test_tw_transport_parse_and_typed_revision_are_country_scoped(self):
+        payload = warning_payload(
+            item=warning_item(
+                country_iso_alp2="TW",
+                country_nm="대만",
+                country_eng_nm="Taiwan",
+            )
+        )
+
+        outcome = ingest_travel_warning(
+            country_iso2="TW",
+            transport=SequenceTransport(
+                successful_result(payload, country_iso2="TW")
+            ),
+            secret_loader=self.loader(),
+        )
+
+        self.assertEqual(
+            outcome.code,
+            TravelWarningIngestionCode.REVIEW_REQUIRED,
+        )
+        revision = TravelWarningRevision.objects.get()
+        self.assertEqual(revision.country_id, SUPPORTED_COUNTRY_IDS["TW"])
+        attempt = FetchAttempt.objects.get()
+        self.assertEqual(
+            attempt.normalized_request_sha256,
+            warning_request_fingerprint("TW").normalized_request_sha256,
+        )
+        self.assertEqual(revision.parse_run.outcome, ParseRun.Outcome.VALIDATED)
+        self.assertEqual(revision.first_observed_at, attempt.completed_at)
+        self.assert_warning_lock_available()
+
     def test_same_body_replay_adds_only_fresh_attempt_evidence(self):
         payload = warning_payload()
         first = ingest_travel_warning(
@@ -576,7 +612,7 @@ class TravelWarningIngestionTests(TransactionTestCase):
         self.assertFalse(ParseRun.objects.exists())
         self.assertFalse(TravelWarningRevision.objects.exists())
         source = SourceConfiguration.objects.get(
-            code="MOFA_TRAVEL_ALARM_API_JP"
+            code=WARNING_SOURCE_CODE
         )
         self.assertEqual(source.state, SourceConfiguration.State.ACTIVE)
         self.assertEqual(source.rights_decisions.count(), 1)
@@ -602,7 +638,7 @@ class TravelWarningIngestionTests(TransactionTestCase):
 
     def test_abandoned_receipt_is_closed_before_a_new_operation(self):
         source = SourceConfiguration.objects.get(
-            code="MOFA_TRAVEL_ALARM_API_JP"
+            code=WARNING_SOURCE_CODE
         )
         rights = source.rights_decisions.get(decision_sequence=1)
         abandoned = FetchAttempt.objects.create(
@@ -1050,7 +1086,13 @@ class TravelWarningIngestionTests(TransactionTestCase):
         self.assertNotIn(raw_marker.decode("ascii"), repr(raw_verified))
         self.assertEqual(
             set(inspect.signature(ingest_travel_warning).parameters),
-            {"transport", "parser", "retry_wait", "secret_loader"},
+            {
+                "country_iso2",
+                "transport",
+                "parser",
+                "retry_wait",
+                "secret_loader",
+            },
         )
         banned = {
             "raw",
@@ -1108,24 +1150,37 @@ class TravelWarningIngestionTests(TransactionTestCase):
 
 
 class TravelWarningSecretLoaderTests(SimpleTestCase):
-    def test_default_loader_reads_only_the_requested_environment_name(self):
-        class EnvironmentBoundary:
-            def __init__(self):
-                self.requested = []
-
-            def get(self, name):
-                self.requested.append(name)
-                return SYNTHETIC_CREDENTIAL
-
-        boundary = EnvironmentBoundary()
-        with patch("travel_warnings.ingestion.os.environ", boundary):
-            value = _environment_secret_loader(WARNING_SECRET_REFERENCE)
+    def test_default_loader_prefers_canonical_and_supports_legacy_fallback(self):
+        with patch.dict(
+            "travel_warnings.ingestion.os.environ",
+            {
+                "DATA_GO_KR_SERVICE_KEY": "synthetic-canonical",
+                "MOFA_TRAVEL_ALARM_SERVICE_KEY": "synthetic-legacy",
+            },
+            clear=True,
+        ):
+            canonical = _environment_secret_loader(WARNING_SECRET_REFERENCE)
+        with patch.dict(
+            "travel_warnings.ingestion.os.environ",
+            {"MOFA_TRAVEL_ALARM_SERVICE_KEY": "synthetic-legacy"},
+            clear=True,
+        ):
+            fallback = _environment_secret_loader(WARNING_SECRET_REFERENCE)
 
-        self.assertEqual(value, SYNTHETIC_CREDENTIAL)
-        self.assertEqual(boundary.requested, [WARNING_SECRET_REFERENCE])
+        self.assertEqual(canonical, "synthetic-canonical")
+        self.assertEqual(fallback, "synthetic-legacy")
 
 
 class TravelWarningIngestionCommandTests(SimpleTestCase):
+    def test_command_country_option_is_exactly_the_supported_allowlist(self):
+        with patch(
+            "operations.management.commands.ingest_travel_warning."
+            "ingest_travel_warning"
+        ) as ingestion:
+            with self.assertRaises(CommandError):
+                call_command("ingest_travel_warning", "--country", "US")
+        ingestion.assert_not_called()
+
     def test_command_output_contains_only_fixed_code_and_attempt_count(self):
         output = io.StringIO()
         with patch(
@@ -1135,12 +1190,16 @@ class TravelWarningIngestionCommandTests(SimpleTestCase):
                 TravelWarningIngestionCode.REVIEW_REQUIRED,
                 1,
             ),
-        ):
-            call_command("ingest_travel_warning", stdout=output)
+        ) as ingestion:
+            call_command("ingest_travel_warning", "--country", "TW", stdout=output)
         self.assertEqual(
             output.getvalue(),
-            "travel_warning_ingestion_result=REVIEW_REQUIRED attempts=1\n",
+            (
+                "travel_warning_ingestion_result=REVIEW_REQUIRED "
+                "attempts=1 country=TW\n"
+            ),
         )
+        ingestion.assert_called_once_with(country_iso2="TW")
 
     def test_command_failure_contains_only_fixed_code_and_attempt_count(self):
         with patch(
@@ -1155,7 +1214,10 @@ class TravelWarningIngestionCommandTests(SimpleTestCase):
                 call_command("ingest_travel_warning")
         self.assertEqual(
             str(raised.exception),
-            "travel_warning_ingestion_result=FETCH_TERMINAL_FAILED attempts=1",
+            (
+                "travel_warning_ingestion_result=FETCH_TERMINAL_FAILED "
+                "attempts=1 country=JP"
+            ),
         )
 
     def test_command_never_exposes_exception_text(self):
@@ -1169,7 +1231,10 @@ class TravelWarningIngestionCommandTests(SimpleTestCase):
                 call_command("ingest_travel_warning")
         self.assertEqual(
             str(raised.exception),
-            "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0",
+            (
+                "travel_warning_ingestion_result=PERSISTENCE_FAILED "
+                "attempts=0 country=JP"
+            ),
         )
         self.assertNotIn(marker, str(raised.exception))
         assert_exception_boundary_is_redacted(self, raised.exception, marker)
@@ -1185,7 +1250,10 @@ class TravelWarningIngestionCommandTests(SimpleTestCase):
                 call_command("ingest_travel_warning")
         self.assertEqual(
             str(raised.exception),
-            "travel_warning_ingestion_result=PERSISTENCE_FAILED attempts=0",
+            (
+                "travel_warning_ingestion_result=PERSISTENCE_FAILED "
+                "attempts=0 country=JP"
+            ),
         )
         self.assertNotIn(marker, str(raised.exception))
         assert_exception_boundary_is_redacted(self, raised.exception, marker)
@@ -1208,7 +1276,7 @@ class TravelWarningIngestionCommandTests(SimpleTestCase):
                     str(raised.exception),
                     (
                         "travel_warning_ingestion_result="
-                        "PERSISTENCE_FAILED attempts=0"
+                        "PERSISTENCE_FAILED attempts=0 country=JP"
                     ),
                 )
                 self.assertNotIn(marker, str(raised.exception))
diff --git a/travel_warnings/tests/test_models.py b/travel_warnings/tests/test_models.py
index 9e3ef62..306b874 100644
--- a/travel_warnings/tests/test_models.py
+++ b/travel_warnings/tests/test_models.py
@@ -29,7 +29,7 @@ from travel_warnings.models import TravelWarningRevision
 
 class TravelWarningFixtureMixin:
     WARNING_CONTRACT = (
-        "c43c5f7e7e6f37f13ba7e5d9e2448b29b24d5524012b41fc5e851abb907c55f6"
+        "5a6c879394b8b25c56ad53eaaf3f294f87ae71956c5a9397d830972b426f85a7"
     )
     WARNING_SCHEMA = (
         "64a98b89c39d6c04e0daf2bbac9353de053320b835a62076a37f620cc07c4f0b"
@@ -46,19 +46,19 @@ class TravelWarningFixtureMixin:
 
     def create_warning_source(self, **rights_overrides):
         source = SourceConfiguration.objects.create(
-            code="MOFA_TRAVEL_ALARM_API_JP",
-            revision="rights-v1",
+            code="MOFA_TRAVEL_ALARM_API_TRAVEL6",
+            revision="travel6-v1",
             module=SourceConfiguration.Module.TRAVEL_WARNING,
             owner="대한민국 외교부",
             official_locator=(
                 "https://apis.data.go.kr/1262000/TravelAlarmService2/"
                 "getTravelAlarmList2"
             ),
-            secret_reference_name="MOFA_TRAVEL_ALARM_SERVICE_KEY",
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
         )
         rights_values = {
             "source": source,
-            "source_revision": "rights-v1",
+            "source_revision": "travel6-v1",
             "decision_sequence": 1,
             "decision": SourceRightsDecision.Decision.APPROVED,
             "access_mode": SourceRightsDecision.AccessMode.CREDENTIAL_REFERENCE,
@@ -70,11 +70,11 @@ class TravelWarningFixtureMixin:
             "raw_retention_seconds": 0,
             "typed_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
             "evidence_retention": SourceRightsDecision.Retention.PRODUCT_HISTORY,
-            "field_scope_code": "JP_WARNING_V1",
+            "field_scope_code": "SUPPORTED_6_WARNING_V1",
             "attribution_text": "외교부|공공데이터포털",
             "contract_fingerprint_sha256": self.WARNING_CONTRACT,
             "decided_by": "PROJECT_OWNER_REQUEST",
-            "decision_basis_code": "USER_FOLLOWUP_20260830",
+            "decision_basis_code": "USER_TRAVEL6_SCOPE_20260831",
         }
         rights_values.update(rights_overrides)
         rights = SourceRightsDecision.objects.create(**rights_values)
@@ -91,7 +91,7 @@ class TravelWarningFixtureMixin:
     def make_parse(
         self,
         suffix,
-        source_code="MOFA_TRAVEL_ALARM_API_JP",
+        source_code="MOFA_TRAVEL_ALARM_API_TRAVEL6",
         *,
         close=True,
         review_artifact=True,
@@ -417,7 +417,7 @@ class TravelWarningRevisionTests(TravelWarningFixtureMixin, TestCase):
 
     def test_candidate_requires_an_active_source_at_insert_time(self):
         SourceConfiguration.objects.filter(
-            code="MOFA_TRAVEL_ALARM_API_JP"
+            code="MOFA_TRAVEL_ALARM_API_TRAVEL6"
         ).update(state=SourceConfiguration.State.PAUSED, enabled=False)
 
         self.assert_integrity_error(
@@ -427,7 +427,9 @@ class TravelWarningRevisionTests(TravelWarningFixtureMixin, TestCase):
         )
 
     def test_later_rights_rejection_closes_candidate_creation(self):
-        source = SourceConfiguration.objects.get(code="MOFA_TRAVEL_ALARM_API_JP")
+        source = SourceConfiguration.objects.get(
+            code="MOFA_TRAVEL_ALARM_API_TRAVEL6"
+        )
         SourceRightsDecision.objects.create(
             source=source,
             source_revision=source.revision,
@@ -541,7 +543,7 @@ class TravelWarningRightsGuardTests(TravelWarningFixtureMixin, TestCase):
 
 
 class TravelWarningMigrationTests(TravelWarningFixtureMixin, TransactionTestCase):
-    latest = [("travel_warnings", "0002_revision_source_rights_guard")]
+    latest = [("travel_warnings", "0003_country_scoped_warning_revisions")]
     previous = [("travel_warnings", "0001_initial")]
 
     def restore_latest(self):
diff --git a/travel_warnings/tests/test_parser.py b/travel_warnings/tests/test_parser.py
index b76f66d..a05540a 100644
--- a/travel_warnings/tests/test_parser.py
+++ b/travel_warnings/tests/test_parser.py
@@ -135,6 +135,24 @@ class TravelAlarmParserTests(SimpleTestCase):
             hashlib.sha256(canonical).hexdigest(),
         )
 
+    def test_supported_tw_identity_is_parsed_with_country_scoped_hash(self):
+        payload = self.encode(
+            self.document(
+                item=self.item(
+                    country_iso_alp2="TW",
+                    country_nm="대만",
+                    country_eng_nm="Taiwan",
+                )
+            )
+        )
+
+        result = parse_travel_alarm_jp(payload, country_iso2="TW")
+
+        self.assertIsInstance(result, TravelWarningParseSuccess)
+        self.assertEqual(result.warning.country_iso2, "TW")
+        self.assertEqual(result.warning.country_name_ko, "대만")
+        self.assertEqual(result.warning.country_name_en, "Taiwan")
+
     def test_provider_error_and_unknown_envelope_fail_closed(self):
         expected = EXPECTED_SCHEMA_FINGERPRINT_SHA256
         self.assert_failure(
diff --git a/travel_windows/contracts.py b/travel_windows/contracts.py
index a6ebd88..503ba52 100644
--- a/travel_windows/contracts.py
+++ b/travel_windows/contracts.py
@@ -10,6 +10,9 @@ SCHEDULE_SOURCE_OWNER = "인천국제공항공사"
 SCHEDULE_SOURCE_LOCATOR = (
     "https://apis.data.go.kr/B551177/statusOfSPaxFlt4TripPlatform"
 )
+SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR = (
+    "https://www.data.go.kr/data/15143060/openapi.do"
+)
 SCHEDULE_DEPARTURES_LOCATOR = (
     f"{SCHEDULE_SOURCE_LOCATOR}/getSPaxFlt4TripPlatformDepartures"
 )
@@ -19,18 +22,35 @@ SCHEDULE_ARRIVALS_LOCATOR = (
 SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 SCHEDULE_FIELD_SCOPE = "ICN_SCHEDULE_V1"
 
-CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API"
+CITY_SOURCE_CODE = "ICN_DESTINATION_CITY_API_15095067"
 CITY_SOURCE_REVISION = "travel-v1"
 CITY_SOURCE_OWNER = "인천국제공항공사"
-CITY_SOURCE_LOCATOR = "https://www.data.go.kr/"
+CITY_SOURCE_LOCATOR = (
+    "https://apis.data.go.kr/B551177/StatusOfSrvDestinations/"
+    "getServiceDestinationInfo"
+)
 CITY_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
 CITY_FIELD_SCOPE = "ICN_DESTINATION_CITY_V1"
 
-DURATION_SOURCE_CODE = "PORT_LOGISTICS_ROUTE_DURATION"
+LEGACY_SCHEDULE_SOURCE_CODE = "ICN_LEGACY_ARRIVALS_API"
+LEGACY_SCHEDULE_SOURCE_REVISION = "travel-v1"
+LEGACY_SCHEDULE_SOURCE_OWNER = "인천국제공항공사"
+LEGACY_SCHEDULE_SOURCE_LOCATOR = (
+    "https://apis.data.go.kr/B551177/PaxFltSched"
+)
+LEGACY_SCHEDULE_ARRIVALS_LOCATOR = (
+    f"{LEGACY_SCHEDULE_SOURCE_LOCATOR}/getPaxFltSchedArrivals"
+)
+LEGACY_SCHEDULE_ATTRIBUTION = "인천국제공항공사|공공데이터포털"
+LEGACY_SCHEDULE_FIELD_SCOPE = "ICN_LEGACY_ARRIVALS_CROSSCHECK_V1"
+
+DURATION_SOURCE_CODE = "PORT_LOGISTICS_ROUTE_DURATION_15151728"
 DURATION_SOURCE_REVISION = "travel-v1"
 DURATION_SOURCE_OWNER = "대한민국 해양수산부"
-DURATION_SOURCE_LOCATOR = "https://www.ulip.go.kr/"
-DURATION_ATTRIBUTION = "해양수산부 수출입 물류 플랫폼"
+DURATION_SOURCE_LOCATOR = (
+    "https://www.data.go.kr/data/15151728/fileData.do"
+)
+DURATION_ATTRIBUTION = "해양수산부 수출입 물류 플랫폼|공공데이터포털"
 DURATION_FIELD_SCOPE = "ROUTE_DURATION_V1"
 
 SCHEDULE_CONTRACT_TEXT = (
@@ -49,7 +69,7 @@ SCHEDULE_SCHEMA_TEXT = (
 )
 DURATION_CONTRACT_TEXT = (
     "route duration v1|destination_iata,outbound_minutes,inbound_minutes,"
-    "reference_date,reference_locator"
+    "reference_date,reference_locator|exact-dataset-15151728"
 )
 DURATION_SCHEMA_TEXT = (
     "csv(destination_iata,outbound_minutes,inbound_minutes,reference_date,"
@@ -65,8 +85,35 @@ SCHEDULE_CONTRACT_FINGERPRINT_SHA256 = _sha256(SCHEDULE_CONTRACT_TEXT)
 SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(SCHEDULE_SCHEMA_TEXT)
 DURATION_CONTRACT_FINGERPRINT_SHA256 = _sha256(DURATION_CONTRACT_TEXT)
 DURATION_SCHEMA_FINGERPRINT_SHA256 = _sha256(DURATION_SCHEMA_TEXT)
-CITY_CONTRACT_FINGERPRINT_SHA256 = _sha256(
-    "ICN destination city v1|iata|country|city|timezone evidence"
+CITY_CONTRACT_TEXT = (
+    "ICN destination city v1|response(header(resultCode,resultMsg),body("
+    "items(item(countryName,airportName,airportCode))))|exact-curated-13|"
+    "Korean-country-city-names"
+)
+CITY_SCHEMA_TEXT = (
+    "documented-data-go-json(response.header+response.body.items.item+"
+    "countryName+airportName+airportCode)"
+)
+LEGACY_SCHEDULE_CONTRACT_TEXT = (
+    "ICN legacy arrivals v1|response(header(resultCode,resultMsg),body("
+    "items(item(airline,airport,airportcode,firstdate,flightid,friday,"
+    "lastdate,monday,saturday,season,st,sunday,thursday,tuesday,wednesday)),"
+    "pageNo,numOfRows,totalCount))|complete-pages|inbound-route-day-time-"
+    "crosscheck|fail-closed"
+)
+LEGACY_SCHEDULE_SCHEMA_TEXT = (
+    "documented-data-go-json(response.header+response.body.items.item+"
+    "pageNo+numOfRows+totalCount); typed-arrival(iata,flight,time,season,"
+    "valid-range,weekdays)"
+)
+
+CITY_CONTRACT_FINGERPRINT_SHA256 = _sha256(CITY_CONTRACT_TEXT)
+CITY_SCHEMA_FINGERPRINT_SHA256 = _sha256(CITY_SCHEMA_TEXT)
+LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256 = _sha256(
+    LEGACY_SCHEDULE_CONTRACT_TEXT
+)
+LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256 = _sha256(
+    LEGACY_SCHEDULE_SCHEMA_TEXT
 )
 
 
diff --git a/travel_windows/ingestion.py b/travel_windows/ingestion.py
index d64733a..d6cb5f8 100644
--- a/travel_windows/ingestion.py
+++ b/travel_windows/ingestion.py
@@ -24,9 +24,17 @@ from sources.models import (
 )
 
 from .contracts import (
+    CITY_CONTRACT_FINGERPRINT_SHA256,
+    CITY_SCHEMA_FINGERPRINT_SHA256,
+    CITY_SOURCE_CODE,
+    CITY_SOURCE_LOCATOR,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
+    LEGACY_SCHEDULE_ARRIVALS_LOCATOR,
+    LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_SOURCE_CODE,
     SCHEDULE_ARRIVALS_LOCATOR,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
     SCHEDULE_DEPARTURES_LOCATOR,
@@ -35,9 +43,13 @@ from .contracts import (
 )
 from .parser import (
     AviationParseFailure,
+    CityReferenceParseSuccess,
     DurationParseSuccess,
+    LegacyArrivalParseSuccess,
     ScheduleParseSuccess,
     adapt_data_go_schedule_pages,
+    parse_destination_city_reference,
+    parse_legacy_arrival_pages,
     parse_route_durations,
 )
 from .publication import FlightPublicationCode, publish_flight_evidence
@@ -200,6 +212,8 @@ def ingest_and_publish_flight_evidence(
     *,
     departure_pages: tuple[bytes, ...],
     arrival_pages: tuple[bytes, ...],
+    city_payload: bytes,
+    legacy_arrival_pages: tuple[bytes, ...],
     duration_csv: bytes,
     source_date: date,
     published_by: str,
@@ -213,13 +227,24 @@ def ingest_and_publish_flight_evidence(
         arrival_pages=arrival_pages,
         source_date=source_date,
     )
+    city_reference = parse_destination_city_reference(city_payload)
+    legacy_arrivals = parse_legacy_arrival_pages(legacy_arrival_pages)
     durations = parse_route_durations(duration_csv)
-    if isinstance(schedule, AviationParseFailure) or isinstance(
-        durations, AviationParseFailure
+    if any(
+        isinstance(result, AviationParseFailure)
+        for result in (
+            schedule,
+            city_reference,
+            legacy_arrivals,
+            durations,
+        )
     ):
         return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
-    if not isinstance(schedule, ScheduleParseSuccess) or not isinstance(
-        durations, DurationParseSuccess
+    if (
+        not isinstance(schedule, ScheduleParseSuccess)
+        or not isinstance(city_reference, CityReferenceParseSuccess)
+        or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
+        or not isinstance(durations, DurationParseSuccess)
     ):
         return FlightIngestionOutcome(FlightIngestionCode.PARSE_QUARANTINED)
     try:
@@ -229,7 +254,8 @@ def ingest_and_publish_flight_evidence(
             request_fingerprint_revision="ICN_SCHEDULE_V1",
             normalized_request_sha256=_request_hash(
                 f"{SCHEDULE_DEPARTURES_LOCATOR}\n{SCHEDULE_ARRIVALS_LOCATOR}\n"
-                "serviceKey=<redacted>&pageNo=1..N&type=json"
+                "serviceKey=<redacted>&pageNo=1..N&type=json&"
+                f"season={schedule.season}"
             ),
             provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
             parser_name=ParseRun.ParserName.ICN_FLIGHT_SCHEDULE_JSON,
@@ -237,6 +263,38 @@ def ingest_and_publish_flight_evidence(
             schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
             completed_at=checked_at,
         )
+        city_reference_run = _record_validated_parse(
+            source_code=CITY_SOURCE_CODE,
+            payload_parts=(city_payload,),
+            request_fingerprint_revision="ICN_DESTINATION_CITY_V1",
+            normalized_request_sha256=_request_hash(
+                f"{CITY_SOURCE_LOCATOR}\nserviceKey=<redacted>&type=json"
+            ),
+            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+            parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+            contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
+            schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+            completed_at=checked_at,
+        )
+        legacy_arrivals_run = _record_validated_parse(
+            source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+            payload_parts=legacy_arrival_pages,
+            request_fingerprint_revision="ICN_LEGACY_ARRIVALS_V1",
+            normalized_request_sha256=_request_hash(
+                f"{LEGACY_SCHEDULE_ARRIVALS_LOCATOR}\n"
+                "serviceKey=<redacted>&pageNo=1..N&numOfRows=100&"
+                "lang=K&type=json"
+            ),
+            provider_code=FetchAttempt.ProviderCode.DATA_GO_SUCCESS,
+            parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+            contract_fingerprint=(
+                LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+            ),
+            schema_fingerprint=(
+                LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+            ),
+            completed_at=checked_at,
+        )
         duration_run = _record_validated_parse(
             source_code=DURATION_SOURCE_CODE,
             payload_parts=(duration_csv,),
@@ -253,6 +311,10 @@ def ingest_and_publish_flight_evidence(
         outcome = publish_flight_evidence(
             schedule_run=schedule_run,
             schedule=schedule,
+            city_reference_run=city_reference_run,
+            city_reference=city_reference,
+            legacy_arrivals_run=legacy_arrivals_run,
+            legacy_arrivals=legacy_arrivals,
             duration_run=duration_run,
             durations=durations,
             published_by=published_by,
diff --git a/travel_windows/management/commands/publish_scheduled_flights.py b/travel_windows/management/commands/publish_scheduled_flights.py
index 97346a9..899c8dd 100644
--- a/travel_windows/management/commands/publish_scheduled_flights.py
+++ b/travel_windows/management/commands/publish_scheduled_flights.py
@@ -5,10 +5,16 @@ from django.core.management.base import BaseCommand, CommandError
 
 from sources.models import SourceConfiguration
 from sources.transport import (
+    fetch_data_go_reference_evidence,
     fetch_data_go_schedule_pages,
     load_aviation_secret_reference,
 )
-from travel_windows.contracts import DATA_GO_SECRET_REFERENCE, SCHEDULE_SOURCE_CODE
+from travel_windows.contracts import (
+    CITY_SOURCE_CODE,
+    DATA_GO_SECRET_REFERENCE,
+    LEGACY_SCHEDULE_SOURCE_CODE,
+    SCHEDULE_SOURCE_CODE,
+)
 from travel_windows.ingestion import ingest_and_publish_flight_evidence
 
 
@@ -35,6 +41,19 @@ class Command(BaseCommand):
                 enabled=True,
                 secret_reference_name=DATA_GO_SECRET_REFERENCE,
             )
+            reference_sources = list(
+                SourceConfiguration.objects.filter(
+                    code__in=(
+                        CITY_SOURCE_CODE,
+                        LEGACY_SCHEDULE_SOURCE_CODE,
+                    ),
+                    state=SourceConfiguration.State.ACTIVE,
+                    enabled=True,
+                    secret_reference_name=DATA_GO_SECRET_REFERENCE,
+                )
+            )
+            if len(reference_sources) != 2:
+                raise SourceConfiguration.DoesNotExist
         except (OSError, TypeError, ValueError):
             raise CommandError("result=blocked code=INVALID_INPUT") from None
         except SourceConfiguration.DoesNotExist:
@@ -52,10 +71,29 @@ class Command(BaseCommand):
         )
         if not fetched.succeeded:
             raise CommandError(f"result=blocked code={fetched.failure_code}")
+        reference_evidence = fetch_data_go_reference_evidence(
+            secret_reference_name=DATA_GO_SECRET_REFERENCE,
+            secret_value=secret_value,
+            connect_timeout_seconds=min(
+                row.connect_timeout_seconds for row in reference_sources
+            ),
+            read_timeout_seconds=min(
+                row.read_timeout_seconds for row in reference_sources
+            ),
+        )
+        if not reference_evidence.succeeded:
+            raise CommandError(
+                "result=blocked "
+                f"code={reference_evidence.failure_code}"
+            )
 
         outcome = ingest_and_publish_flight_evidence(
             departure_pages=fetched.departure_pages,
             arrival_pages=fetched.arrival_pages,
+            city_payload=reference_evidence.city_payload,
+            legacy_arrival_pages=(
+                reference_evidence.legacy_arrival_pages
+            ),
             duration_csv=duration_csv,
             source_date=source_date,
             published_by=options["published_by"],
diff --git a/travel_windows/migrations/0005_aviation_reference_evidence.py b/travel_windows/migrations/0005_aviation_reference_evidence.py
new file mode 100644
index 0000000..02f7def
--- /dev/null
+++ b/travel_windows/migrations/0005_aviation_reference_evidence.py
@@ -0,0 +1,34 @@
+import django.db.models.deletion
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("sources", "0012_aviation_reference_contracts"),
+        ("travel_windows", "0004_curated_airport_constraint"),
+    ]
+
+    operations = [
+        migrations.AddField(
+            model_name="flightschedulerevision",
+            name="city_reference_parse_run",
+            field=models.OneToOneField(
+                blank=True,
+                null=True,
+                on_delete=django.db.models.deletion.PROTECT,
+                related_name="flight_city_reference_revision",
+                to="sources.parserun",
+            ),
+        ),
+        migrations.AddField(
+            model_name="flightschedulerevision",
+            name="legacy_arrivals_parse_run",
+            field=models.OneToOneField(
+                blank=True,
+                null=True,
+                on_delete=django.db.models.deletion.PROTECT,
+                related_name="flight_legacy_arrivals_revision",
+                to="sources.parserun",
+            ),
+        ),
+    ]
diff --git a/travel_windows/models.py b/travel_windows/models.py
index cf82211..44fd3fc 100644
--- a/travel_windows/models.py
+++ b/travel_windows/models.py
@@ -112,6 +112,20 @@ class FlightScheduleRevision(models.Model):
         on_delete=models.PROTECT,
         related_name="flight_schedule_revision",
     )
+    city_reference_parse_run = models.OneToOneField(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="flight_city_reference_revision",
+        null=True,
+        blank=True,
+    )
+    legacy_arrivals_parse_run = models.OneToOneField(
+        "sources.ParseRun",
+        on_delete=models.PROTECT,
+        related_name="flight_legacy_arrivals_revision",
+        null=True,
+        blank=True,
+    )
     source_date = models.DateField()
     season = models.CharField(max_length=32)
     coverage_code = models.CharField(max_length=64)
diff --git a/travel_windows/parser.py b/travel_windows/parser.py
index b52c533..d2a5b2a 100644
--- a/travel_windows/parser.py
+++ b/travel_windows/parser.py
@@ -9,7 +9,10 @@ from datetime import date, time
 from typing import Any
 
 from .contracts import (
+    CITY_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
+    DURATION_SOURCE_LOCATOR,
+    LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
 )
 
@@ -85,6 +88,35 @@ _OFFICIAL_REQUIRED_ITEM_KEYS = {
     "airportCode",
 }
 _WEEKDAY_KEYS = ("ynMon", "ynTue", "ynWed", "ynThu", "ynFri", "ynSat", "ynSun")
+_CITY_BODY_KEYS = {"items"}
+_CITY_ITEM_KEYS = {"countryName", "airportName", "airportCode"}
+_LEGACY_BODY_KEYS = {"items", "pageNo", "numOfRows", "totalCount"}
+_LEGACY_ITEM_KEYS = {
+    "airline",
+    "airport",
+    "airportcode",
+    "firstdate",
+    "flightid",
+    "friday",
+    "lastdate",
+    "monday",
+    "saturday",
+    "season",
+    "st",
+    "sunday",
+    "thursday",
+    "tuesday",
+    "wednesday",
+}
+_LEGACY_WEEKDAY_KEYS = (
+    "monday",
+    "tuesday",
+    "wednesday",
+    "thursday",
+    "friday",
+    "saturday",
+    "sunday",
+)
 
 
 class _DuplicateKey(ValueError):
@@ -122,6 +154,40 @@ class ScheduleParseSuccess:
     observed_schema_fingerprint_sha256: str = SCHEDULE_SCHEMA_FINGERPRINT_SHA256
 
 
+@dataclass(frozen=True, slots=True)
+class ParsedDestinationCity:
+    airport_code: str
+    country_name_ko: str
+    city_name_ko: str
+
+
+@dataclass(frozen=True, slots=True)
+class CityReferenceParseSuccess:
+    destinations: tuple[ParsedDestinationCity, ...]
+    observed_schema_fingerprint_sha256: str = CITY_SCHEMA_FINGERPRINT_SHA256
+
+
+@dataclass(frozen=True, slots=True)
+class ParsedLegacyArrival:
+    carrier_name: str
+    airport_name: str
+    destination_iata: str
+    flight_number: str
+    icn_event_time: time
+    season: str
+    valid_from: date
+    valid_until: date
+    weekday_mask: str
+
+
+@dataclass(frozen=True, slots=True)
+class LegacyArrivalParseSuccess:
+    arrivals: tuple[ParsedLegacyArrival, ...]
+    observed_schema_fingerprint_sha256: str = (
+        LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+    )
+
+
 @dataclass(frozen=True, slots=True)
 class ParsedRouteDuration:
     destination_iata: str
@@ -254,7 +320,7 @@ def parse_route_durations(payload: bytes) -> DurationParseSuccess | AviationPars
                 or iata in seen
                 or not 1 <= outbound <= 1_440
                 or not 1 <= inbound <= 1_440
-                or not locator.startswith("https://")
+                or locator != DURATION_SOURCE_LOCATOR
             ):
                 raise ValueError
             seen.add(iata)
@@ -314,6 +380,221 @@ def _flight_id(value: object) -> str:
     return normalized
 
 
+def _documented_response(payload: bytes) -> tuple[dict, dict]:
+    if not isinstance(payload, bytes) or not 0 < len(payload) <= MAX_SCHEDULE_BYTES:
+        raise ValueError
+    document = json.loads(
+        payload.decode("utf-8"),
+        object_pairs_hook=_unique_object,
+    )
+    if type(document) is not dict or set(document) != _OFFICIAL_ROOT_KEYS:
+        raise ValueError
+    response = document["response"]
+    if type(response) is not dict or set(response) != _OFFICIAL_RESPONSE_KEYS:
+        raise ValueError
+    header = response["header"]
+    body = response["body"]
+    if (
+        type(header) is not dict
+        or set(header) != _OFFICIAL_HEADER_KEYS
+        or header["resultCode"] not in {"00", "0"}
+        or type(header["resultMsg"]) is not str
+        or not header["resultMsg"]
+        or type(body) is not dict
+    ):
+        raise ValueError
+    return header, body
+
+
+def _item_rows(items: object, *, exact_keys: set[str]) -> list[dict]:
+    if type(items) is not dict or set(items) != _OFFICIAL_ITEMS_KEYS:
+        raise ValueError
+    raw_items = items["item"]
+    if type(raw_items) is dict:
+        rows = [raw_items]
+    elif type(raw_items) is list:
+        rows = raw_items
+    else:
+        raise ValueError
+    if not rows or any(
+        type(row) is not dict or set(row) != exact_keys for row in rows
+    ):
+        raise ValueError
+    return rows
+
+
+def parse_destination_city_reference(
+    payload: bytes,
+) -> CityReferenceParseSuccess | AviationParseFailure:
+    """Parse the documented 15095067 Korean destination-city response."""
+
+    try:
+        _, body = _documented_response(payload)
+        if set(body) != _CITY_BODY_KEYS:
+            raise ValueError
+        rows = _item_rows(body["items"], exact_keys=_CITY_ITEM_KEYS)
+        destinations: list[ParsedDestinationCity] = []
+        seen: set[str] = set()
+        for row in rows:
+            airport_code = row["airportCode"]
+            country_name = row["countryName"]
+            city_name = row["airportName"]
+            if (
+                type(airport_code) is not str
+                or not _IATA.fullmatch(airport_code)
+                or airport_code in seen
+                or type(country_name) is not str
+                or not country_name.strip()
+                or len(country_name) > 100
+                or type(city_name) is not str
+                or not city_name.strip()
+                or len(city_name) > 100
+            ):
+                raise ValueError
+            seen.add(airport_code)
+            destinations.append(
+                ParsedDestinationCity(
+                    airport_code=airport_code,
+                    country_name_ko=country_name.strip(),
+                    city_name_ko=city_name.strip(),
+                )
+            )
+        return CityReferenceParseSuccess(
+            destinations=tuple(
+                sorted(destinations, key=lambda row: row.airport_code)
+            )
+        )
+    except (
+        UnicodeDecodeError,
+        json.JSONDecodeError,
+        _DuplicateKey,
+        TypeError,
+        ValueError,
+    ):
+        return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+
+
+def _decode_legacy_arrival_page(
+    payload: bytes,
+) -> tuple[int, int, int, list[dict]]:
+    _, body = _documented_response(payload)
+    if set(body) != _LEGACY_BODY_KEYS:
+        raise ValueError
+    rows = _item_rows(body["items"], exact_keys=_LEGACY_ITEM_KEYS)
+    return (
+        _integer(body["pageNo"]),
+        _integer(body["numOfRows"]),
+        _integer(body["totalCount"]),
+        rows,
+    )
+
+
+def parse_legacy_arrival_pages(
+    pages: tuple[bytes, ...],
+) -> LegacyArrivalParseSuccess | AviationParseFailure:
+    """Parse complete 15095059 arrivals pages for fail-closed cross-checking."""
+
+    try:
+        if not pages:
+            raise ValueError
+        decoded = [_decode_legacy_arrival_page(payload) for payload in pages]
+        first_page, page_size, total, _ = decoded[0]
+        if first_page != 1 or page_size < 1 or total < 1:
+            raise ValueError
+        expected_pages = (total + page_size - 1) // page_size
+        if len(decoded) != expected_pages:
+            raise ValueError
+        raw_rows: list[dict] = []
+        for expected_page, page in enumerate(decoded, 1):
+            page_number, observed_size, observed_total, rows = page
+            if (
+                page_number != expected_page
+                or observed_size != page_size
+                or observed_total != total
+                or len(rows) > page_size
+            ):
+                raise ValueError
+            raw_rows.extend(rows)
+        if len(raw_rows) != total:
+            raise ValueError
+
+        arrivals: list[ParsedLegacyArrival] = []
+        identities: set[tuple[object, ...]] = set()
+        for row in raw_rows:
+            weekdays = []
+            for index, key in enumerate(_LEGACY_WEEKDAY_KEYS):
+                if row[key] not in {"Y", "N"}:
+                    raise ValueError
+                if row[key] == "Y":
+                    weekdays.append(index)
+            valid_from = date.fromisoformat(_official_date(row["firstdate"]))
+            valid_until = date.fromisoformat(_official_date(row["lastdate"]))
+            if valid_until < valid_from or not weekdays:
+                raise ValueError
+            airport_code = row["airportcode"]
+            carrier_name = row["airline"]
+            airport_name = row["airport"]
+            season = row["season"]
+            if (
+                type(airport_code) is not str
+                or not _IATA.fullmatch(airport_code)
+                or type(carrier_name) is not str
+                or not carrier_name.strip()
+                or type(airport_name) is not str
+                or not airport_name.strip()
+                or type(season) is not str
+                or not season
+                or len(season) > 32
+            ):
+                raise ValueError
+            arrival = ParsedLegacyArrival(
+                carrier_name=carrier_name.strip(),
+                airport_name=airport_name.strip(),
+                destination_iata=airport_code,
+                flight_number=_flight_id(row["flightid"]),
+                icn_event_time=time.fromisoformat(_official_time(row["st"])),
+                season=season,
+                valid_from=valid_from,
+                valid_until=valid_until,
+                weekday_mask="".join(
+                    "1" if day in weekdays else "0" for day in range(7)
+                ),
+            )
+            identity = (
+                arrival.destination_iata,
+                arrival.flight_number,
+                arrival.icn_event_time,
+                arrival.season,
+                arrival.valid_from,
+                arrival.valid_until,
+                arrival.weekday_mask,
+            )
+            if identity in identities:
+                raise ValueError
+            identities.add(identity)
+            arrivals.append(arrival)
+        return LegacyArrivalParseSuccess(
+            arrivals=tuple(
+                sorted(
+                    arrivals,
+                    key=lambda row: (
+                        row.destination_iata,
+                        row.icn_event_time,
+                        row.flight_number,
+                    ),
+                )
+            )
+        )
+    except (
+        UnicodeDecodeError,
+        json.JSONDecodeError,
+        _DuplicateKey,
+        TypeError,
+        ValueError,
+    ):
+        return AviationParseFailure("SCHEMA_MISMATCH", "f" * 64)
+
+
 def _decode_official_page(payload: bytes) -> tuple[int, int, int, list[dict[str, object]]]:
     if not isinstance(payload, bytes) or not 0 < len(payload) <= MAX_SCHEDULE_BYTES:
         raise ValueError
diff --git a/travel_windows/publication.py b/travel_windows/publication.py
index 18465cf..d86b728 100644
--- a/travel_windows/publication.py
+++ b/travel_windows/publication.py
@@ -18,20 +18,29 @@ from django.utils import timezone
 from sources.models import ParseRun, SourceConfiguration, SourceRightsDecision
 
 from .contracts import (
+    CITY_CONTRACT_FINGERPRINT_SHA256,
+    CITY_SCHEMA_FINGERPRINT_SHA256,
+    CITY_SOURCE_CODE,
+    CITY_SOURCE_REVISION,
     DURATION_CONTRACT_FINGERPRINT_SHA256,
     DURATION_SCHEMA_FINGERPRINT_SHA256,
     DURATION_SOURCE_CODE,
     DURATION_SOURCE_REVISION,
+    LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
+    LEGACY_SCHEDULE_SOURCE_CODE,
+    LEGACY_SCHEDULE_SOURCE_REVISION,
     SCHEDULE_ATTRIBUTION,
     SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
+    SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
     SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
     SCHEDULE_SOURCE_CODE,
-    SCHEDULE_SOURCE_LOCATOR,
     SCHEDULE_SOURCE_REVISION,
 )
 from .models import (
     FLIGHT_POINTER_ID,
     Airport,
+    CURATED_AIRPORT_ROWS,
     FlightPublication,
     FlightPublicationDuration,
     FlightSchedule,
@@ -39,7 +48,13 @@ from .models import (
     PublishedFlightSchedule,
     RouteDurationRevision,
 )
-from .parser import DurationParseSuccess, ScheduleParseSuccess
+from .parser import (
+    CityReferenceParseSuccess,
+    DurationParseSuccess,
+    LegacyArrivalParseSuccess,
+    ScheduleParseSuccess,
+)
+from countries.models import SUPPORTED_COUNTRY_ROWS
 
 
 PUBLICATION_LOCK_NAMESPACE = 1_414_678_614
@@ -141,10 +156,96 @@ def _aware(value: datetime) -> bool:
     return isinstance(value, datetime) and timezone.is_aware(value)
 
 
+def _reference_evidence_is_exact(
+    *,
+    schedule: ScheduleParseSuccess,
+    city_reference: CityReferenceParseSuccess,
+    legacy_arrivals: LegacyArrivalParseSuccess,
+) -> bool:
+    country_names = {
+        iso_alpha2: name_ko
+        for _country_id, iso_alpha2, name_ko, _name_en, _public in (
+            SUPPORTED_COUNTRY_ROWS
+        )
+    }
+    expected_cities = {
+        iata: (country_names[country_iso2], city_name)
+        for (
+            _airport_id,
+            iata,
+            country_iso2,
+            _city_code,
+            city_name,
+            _airport_name,
+            _timezone,
+        ) in CURATED_AIRPORT_ROWS
+    }
+    observed_cities = {
+        row.airport_code: (row.country_name_ko, row.city_name_ko)
+        for row in city_reference.destinations
+    }
+    if any(
+        observed_cities.get(airport_code) != identity
+        for airport_code, identity in expected_cities.items()
+    ):
+        return False
+    required_inbound = {
+        (
+            row.destination_iata,
+            row.icn_event_time,
+            row.weekday_mask,
+        )
+        for row in schedule.flights
+        if row.direction == "INBOUND"
+    }
+    observed_inbound = {
+        (
+            row.destination_iata,
+            row.icn_event_time,
+            row.weekday_mask,
+        )
+        for row in legacy_arrivals.arrivals
+        if row.season == schedule.season
+    }
+    return bool(required_inbound) and required_inbound.issubset(
+        observed_inbound
+    )
+
+
+def _curated_schedule_projection(
+    schedule: ScheduleParseSuccess,
+) -> ScheduleParseSuccess | None:
+    curated_codes = {row[1] for row in CURATED_AIRPORT_ROWS}
+    flights = tuple(
+        row
+        for row in schedule.flights
+        if row.destination_iata in curated_codes
+    )
+    directions_by_airport: dict[str, set[str]] = {}
+    for row in flights:
+        directions_by_airport.setdefault(row.destination_iata, set()).add(
+            row.direction
+        )
+    if not directions_by_airport or any(
+        directions != {"OUTBOUND", "INBOUND"}
+        for directions in directions_by_airport.values()
+    ):
+        return None
+    return ScheduleParseSuccess(
+        source_date=schedule.source_date,
+        season=schedule.season,
+        flights=flights,
+    )
+
+
 def publish_flight_evidence(
     *,
     schedule_run: ParseRun,
     schedule: ScheduleParseSuccess,
+    city_reference_run: ParseRun,
+    city_reference: CityReferenceParseSuccess,
+    legacy_arrivals_run: ParseRun,
+    legacy_arrivals: LegacyArrivalParseSuccess,
     duration_run: ParseRun,
     durations: DurationParseSuccess,
     published_by: str,
@@ -158,6 +259,8 @@ def publish_flight_evidence(
 
     if (
         not isinstance(schedule, ScheduleParseSuccess)
+        or not isinstance(city_reference, CityReferenceParseSuccess)
+        or not isinstance(legacy_arrivals, LegacyArrivalParseSuccess)
         or not isinstance(durations, DurationParseSuccess)
         or not isinstance(published_by, str)
         or not published_by.strip()
@@ -176,6 +279,26 @@ def publish_flight_evidence(
                 contract_fingerprint=SCHEDULE_CONTRACT_FINGERPRINT_SHA256,
                 schema_fingerprint=SCHEDULE_SCHEMA_FINGERPRINT_SHA256,
             )
+            locked_city_reference_run = _approved_run(
+                city_reference_run,
+                source_code=CITY_SOURCE_CODE,
+                source_revision=CITY_SOURCE_REVISION,
+                parser_name=ParseRun.ParserName.ICN_DESTINATION_CITY_JSON,
+                contract_fingerprint=CITY_CONTRACT_FINGERPRINT_SHA256,
+                schema_fingerprint=CITY_SCHEMA_FINGERPRINT_SHA256,
+            )
+            locked_legacy_arrivals_run = _approved_run(
+                legacy_arrivals_run,
+                source_code=LEGACY_SCHEDULE_SOURCE_CODE,
+                source_revision=LEGACY_SCHEDULE_SOURCE_REVISION,
+                parser_name=ParseRun.ParserName.ICN_LEGACY_ARRIVALS_JSON,
+                contract_fingerprint=(
+                    LEGACY_SCHEDULE_CONTRACT_FINGERPRINT_SHA256
+                ),
+                schema_fingerprint=(
+                    LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+                ),
+            )
             locked_duration_run = _approved_run(
                 duration_run,
                 source_code=DURATION_SOURCE_CODE,
@@ -187,14 +310,28 @@ def publish_flight_evidence(
             if (
                 schedule.observed_schema_fingerprint_sha256
                 != SCHEDULE_SCHEMA_FINGERPRINT_SHA256
+                or city_reference.observed_schema_fingerprint_sha256
+                != CITY_SCHEMA_FINGERPRINT_SHA256
+                or legacy_arrivals.observed_schema_fingerprint_sha256
+                != LEGACY_SCHEDULE_SCHEMA_FINGERPRINT_SHA256
                 or durations.observed_schema_fingerprint_sha256
                 != DURATION_SCHEMA_FINGERPRINT_SHA256
             ):
                 raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
 
-            iata_codes = {row.destination_iata for row in schedule.flights}
+            projected_schedule = _curated_schedule_projection(schedule)
+            if projected_schedule is None or not _reference_evidence_is_exact(
+                schedule=projected_schedule,
+                city_reference=city_reference,
+                legacy_arrivals=legacy_arrivals,
+            ):
+                raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
+
+            iata_codes = {
+                row.destination_iata for row in projected_schedule.flights
+            }
             duration_codes = {row.destination_iata for row in durations.routes}
-            if not iata_codes or iata_codes != duration_codes:
+            if not iata_codes or not iata_codes.issubset(duration_codes):
                 raise _PublicationRejected(FlightPublicationCode.INVALID_EVIDENCE)
             airports = {
                 airport.iata_code: airport
@@ -208,13 +345,15 @@ def publish_flight_evidence(
 
             revision = FlightScheduleRevision.objects.create(
                 parse_run=locked_schedule_run,
+                city_reference_parse_run=locked_city_reference_run,
+                legacy_arrivals_parse_run=locked_legacy_arrivals_run,
                 source_date=schedule.source_date,
                 season=schedule.season,
                 coverage_code="ICN_DIRECT_PUBLIC_V1",
                 completeness=FlightScheduleRevision.Completeness.COMPLETE,
                 state=FlightScheduleRevision.State.VALIDATED,
                 first_observed_at=locked_schedule_run.completed_at,
-                typed_fingerprint_sha256=_fingerprint(schedule),
+                typed_fingerprint_sha256=_fingerprint(projected_schedule),
             )
             FlightSchedule.objects.bulk_create(
                 [
@@ -231,11 +370,13 @@ def publish_flight_evidence(
                         valid_until=row.valid_until,
                         weekday_mask=row.weekday_mask,
                     )
-                    for row in schedule.flights
+                    for row in projected_schedule.flights
                 ]
             )
             duration_revisions = []
             for row in durations.routes:
+                if row.destination_iata not in iata_codes:
+                    continue
                 duration_revisions.append(
                     RouteDurationRevision.objects.create(
                         parse_run=locked_duration_run,
@@ -262,7 +403,7 @@ def publish_flight_evidence(
                 schedule_revision=revision,
                 generation=generation,
                 source_revision=SCHEDULE_SOURCE_REVISION,
-                source_locator=SCHEDULE_SOURCE_LOCATOR,
+                source_locator=SCHEDULE_PUBLIC_DOCUMENTATION_LOCATOR,
                 source_attribution=SCHEDULE_ATTRIBUTION,
                 source_checked_at=source_checked_at,
                 published_by=published_by.strip(),
diff --git a/travel_windows/tests/test_source_publication.py b/travel_windows/tests/test_source_publication.py
index 5bfec28..46b6c23 100644
--- a/travel_windows/tests/test_source_publication.py
+++ b/travel_windows/tests/test_source_publication.py
@@ -1,16 +1,22 @@
+import hashlib
+import io
 import json
 from datetime import date
+from types import SimpleNamespace
+from unittest.mock import patch
 
+from django.core.management import call_command
 from django.test import SimpleTestCase, TransactionTestCase
 from django.utils import timezone
 
-from countries.models import Country
+from countries.models import Country, SUPPORTED_COUNTRY_ROWS
 from sources.management.commands.register_approved_sources import (
     register_approved_sources,
 )
 from sources.models import FetchAttempt, ParseRun, SourceArtifact
 from sources.tests.test_transport import FakeConnection, FakeResponse
 from sources.transport import (
+    fetch_data_go_reference_evidence,
     fetch_data_go_schedule_pages,
     load_aviation_secret_reference,
 )
@@ -18,6 +24,12 @@ from travel_windows.ingestion import (
     FlightIngestionCode,
     ingest_and_publish_flight_evidence,
 )
+from travel_windows.contracts import (
+    DURATION_SOURCE_LOCATOR,
+    SCHEDULE_ARRIVALS_LOCATOR,
+    SCHEDULE_DEPARTURES_LOCATOR,
+    SCHEDULE_SOURCE_CODE,
+)
 from travel_windows.models import (
     Airport,
     CURATED_AIRPORT_ROWS,
@@ -28,6 +40,9 @@ from travel_windows.models import (
 from travel_windows.parser import (
     AviationParseFailure,
     adapt_data_go_schedule_pages,
+    parse_destination_city_reference,
+    parse_legacy_arrival_pages,
+    parse_route_durations,
 )
 
 
@@ -79,6 +94,92 @@ def official_page(rows, *, page=1, page_size=None, total=None):
     ).encode("utf-8")
 
 
+def city_reference_payload(*, overrides=None, extra_rows=()):
+    country_names = {
+        iso_alpha2: name_ko
+        for _country_id, iso_alpha2, name_ko, _name_en, _public in (
+            SUPPORTED_COUNTRY_ROWS
+        )
+    }
+    overrides = overrides or {}
+    rows = [
+        {
+            "countryName": country_names[country_iso2],
+            "airportName": city_name,
+            "airportCode": iata,
+            **overrides.get(iata, {}),
+        }
+        for (
+            _airport_id,
+            iata,
+            country_iso2,
+            _city_code,
+            city_name,
+            _airport_name,
+            _timezone,
+        ) in CURATED_AIRPORT_ROWS
+    ]
+    rows.extend(extra_rows)
+    return json.dumps(
+        {
+            "response": {
+                "header": {
+                    "resultCode": "00",
+                    "resultMsg": "NORMAL SERVICE.",
+                },
+                "body": {"items": {"item": rows}},
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
+def legacy_arrival_row(**overrides):
+    row = {
+        "airline": "대한항공",
+        "airport": "도쿄/나리타",
+        "airportcode": "NRT",
+        "firstdate": "20260329",
+        "flightid": "KE704",
+        "friday": "Y",
+        "lastdate": "20261024",
+        "monday": "Y",
+        "saturday": "Y",
+        "season": "S26",
+        "st": "2015",
+        "sunday": "Y",
+        "thursday": "Y",
+        "tuesday": "Y",
+        "wednesday": "Y",
+    }
+    row.update(overrides)
+    return row
+
+
+def legacy_arrival_page(rows, *, page=1, page_size=None, total=None):
+    page_size = len(rows) if page_size is None else page_size
+    total = len(rows) if total is None else total
+    return json.dumps(
+        {
+            "response": {
+                "header": {
+                    "resultCode": "00",
+                    "resultMsg": "NORMAL SERVICE.",
+                },
+                "body": {
+                    "items": {"item": rows},
+                    "pageNo": str(page),
+                    "numOfRows": str(page_size),
+                    "totalCount": str(total),
+                },
+            }
+        },
+        ensure_ascii=False,
+        separators=(",", ":"),
+    ).encode("utf-8")
+
+
 class DataGoScheduleAdapterTests(SimpleTestCase):
     def test_complete_pages_normalize_and_dedupe_codeshare_by_master(self):
         departure = official_page(
@@ -175,16 +276,187 @@ class DataGoScheduleAdapterTests(SimpleTestCase):
         self.assertTrue(departures.closed)
         self.assertTrue(arrivals.closed)
 
+    def test_city_and_legacy_documented_shapes_are_strictly_typed(self):
+        city = parse_destination_city_reference(
+            city_reference_payload(
+                extra_rows=(
+                    {
+                        "countryName": "네덜란드",
+                        "airportName": "암스테르담",
+                        "airportCode": "AMS",
+                    },
+                )
+            )
+        )
+        legacy = parse_legacy_arrival_pages(
+            (legacy_arrival_page([legacy_arrival_row()]),)
+        )
+
+        self.assertNotIsInstance(city, AviationParseFailure)
+        self.assertEqual(len(city.destinations), 14)
+        self.assertNotIsInstance(legacy, AviationParseFailure)
+        self.assertEqual(legacy.arrivals[0].destination_iata, "NRT")
+        self.assertEqual(legacy.arrivals[0].weekday_mask, "1111111")
+
+        drift = json.loads(city_reference_payload())
+        drift["response"]["body"]["items"]["item"][0]["unknown"] = "x"
+        self.assertIsInstance(
+            parse_destination_city_reference(json.dumps(drift).encode()),
+            AviationParseFailure,
+        )
+        self.assertIsInstance(
+            parse_legacy_arrival_pages(
+                (
+                    legacy_arrival_page(
+                        [legacy_arrival_row()],
+                        page_size=1,
+                        total=2,
+                    ),
+                )
+            ),
+            AviationParseFailure,
+        )
+
+    def test_duration_rows_require_the_exact_15151728_dataset_locator(self):
+        result = parse_route_durations(
+            b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+            b"reference_locator\r\nNRT,150,165,2026-08-01,"
+            b"https://example.invalid/not-the-approved-dataset\r\n"
+        )
+
+        self.assertIsInstance(result, AviationParseFailure)
+
+    def test_reference_transport_fetches_city_and_complete_legacy_pages(self):
+        city_connection = FakeConnection(
+            FakeResponse(200, city_reference_payload())
+        )
+        legacy_connection = FakeConnection(
+            FakeResponse(
+                200,
+                legacy_arrival_page([legacy_arrival_row()]),
+            )
+        )
+        connections = [city_connection, legacy_connection]
+
+        def factory(_host, _port, _timeout):
+            return connections.pop(0)
+
+        secret = "synthetic-reference-sensitive-key"
+        result = fetch_data_go_reference_evidence(
+            secret_reference_name="DATA_GO_KR_SERVICE_KEY",
+            secret_value=secret,
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+            connection_factory=factory,
+        )
+
+        self.assertTrue(result.succeeded)
+        self.assertEqual(len(result.legacy_arrival_pages), 1)
+        self.assertNotIn(secret, repr(result))
+        self.assertTrue(city_connection.closed)
+        self.assertTrue(legacy_connection.closed)
+
+
+class FlightPublicationCommandTests(SimpleTestCase):
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "Command.requires_migrations_checks",
+        False,
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "ingest_and_publish_flight_evidence"
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
+        "Path.read_bytes",
+        return_value=b"reviewed-duration-csv",
+    )
+    @patch(
+        "travel_windows.management.commands.publish_scheduled_flights."
+        "SourceConfiguration.objects"
+    )
+    def test_command_fetches_reference_evidence_before_publication(
+        self,
+        source_objects,
+        _read_bytes,
+        _secret,
+        schedule_fetch,
+        reference_fetch,
+        ingest,
+    ):
+        source = SimpleNamespace(
+            connect_timeout_seconds=3,
+            read_timeout_seconds=8,
+        )
+        source_objects.get.return_value = source
+        source_objects.filter.return_value = [source, source]
+        schedule_fetch.return_value = SimpleNamespace(
+            succeeded=True,
+            departure_pages=(b"departure",),
+            arrival_pages=(b"arrival",),
+        )
+        reference_fetch.return_value = SimpleNamespace(
+            succeeded=True,
+            city_payload=b"city",
+            legacy_arrival_pages=(b"legacy",),
+        )
+        ingest.return_value = SimpleNamespace(succeeded=True, generation=9)
+        output = io.StringIO()
+
+        call_command(
+            "publish_scheduled_flights",
+            duration_csv="reviewed.csv",
+            source_date="2026-08-31",
+            season="S26",
+            published_by="reviewer",
+            stdout=output,
+        )
+
+        reference_fetch.assert_called_once()
+        kwargs = ingest.call_args.kwargs
+        self.assertEqual(kwargs["city_payload"], b"city")
+        self.assertEqual(kwargs["legacy_arrival_pages"], (b"legacy",))
+        self.assertEqual(output.getvalue(), "result=published generation=9\n")
+        self.assertNotIn("synthetic-secret", output.getvalue())
+
 
 class FlightEvidencePublicationTests(TransactionTestCase):
     def setUp(self):
         register_approved_sources(apply=True)
         row = CURATED_AIRPORT_ROWS[0]
+        country_row = next(
+            country
+            for country in SUPPORTED_COUNTRY_ROWS
+            if country[1] == row[2]
+        )
+        country, _ = Country.objects.get_or_create(
+            id=country_row[0],
+            defaults={
+                "iso_alpha2": country_row[1],
+                "name_ko": country_row[2],
+                "name_en": country_row[3],
+                "is_public": country_row[4],
+            },
+        )
         Airport.objects.get_or_create(
             id=row[0],
             defaults={
                 "iata_code": row[1],
-                "country": Country.objects.get(iso_alpha2=row[2]),
+                "country": country,
                 "city_code": row[3],
                 "city_name_ko": row[4],
                 "name_ko": row[5],
@@ -202,11 +474,16 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         duration = (
             b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
             b"reference_locator\r\nNRT,150,165,2026-08-01,"
-            b"https://www.ulip.go.kr/\r\n"
+            + DURATION_SOURCE_LOCATOR.encode("ascii")
+            + b"\r\n"
         )
         outcome = ingest_and_publish_flight_evidence(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
+            city_payload=city_reference_payload(),
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
             duration_csv=duration,
             source_date=date(2026, 8, 31),
             published_by="synthetic-reviewer",
@@ -219,9 +496,26 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         self.assertEqual(pointer.version, 1)
         self.assertEqual(pointer.current_publication.generation, 1)
         self.assertEqual(FlightSchedule.objects.count(), 2)
-        self.assertEqual(FetchAttempt.objects.count(), 2)
-        self.assertEqual(SourceArtifact.objects.count(), 2)
-        self.assertEqual(ParseRun.objects.count(), 2)
+        self.assertEqual(FetchAttempt.objects.count(), 4)
+        self.assertEqual(SourceArtifact.objects.count(), 4)
+        self.assertEqual(ParseRun.objects.count(), 4)
+        schedule_attempt = FetchAttempt.objects.get(
+            source__code=SCHEDULE_SOURCE_CODE
+        )
+        self.assertEqual(
+            schedule_attempt.normalized_request_sha256,
+            hashlib.sha256(
+                (
+                    f"{SCHEDULE_DEPARTURES_LOCATOR}\n"
+                    f"{SCHEDULE_ARRIVALS_LOCATOR}\n"
+                    "serviceKey=<redacted>&pageNo=1..N&type=json&"
+                    "season=S26"
+                ).encode("utf-8")
+            ).hexdigest(),
+        )
+        revision = pointer.current_publication.schedule_revision
+        self.assertIsNotNone(revision.city_reference_parse_run_id)
+        self.assertIsNotNone(revision.legacy_arrivals_parse_run_id)
         model_fields = {
             field.name
             for model in (FetchAttempt, SourceArtifact, ParseRun)
@@ -247,11 +541,23 @@ class FlightEvidencePublicationTests(TransactionTestCase):
         duration = (
             b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
             b"reference_locator\r\nAAA,150,165,2026-08-01,"
-            b"https://www.ulip.go.kr/\r\n"
+            + DURATION_SOURCE_LOCATOR.encode("ascii")
+            + b"\r\n"
         )
         outcome = ingest_and_publish_flight_evidence(
             departure_pages=(departure,),
             arrival_pages=(arrival,),
+            city_payload=city_reference_payload(),
+            legacy_arrival_pages=(
+                legacy_arrival_page(
+                    [
+                        legacy_arrival_row(
+                            airport="알 수 없는 공항",
+                            airportcode="AAA",
+                        )
+                    ]
+                ),
+            ),
             duration_csv=duration,
             source_date=date(2026, 8, 31),
             published_by="synthetic-reviewer",
@@ -263,3 +569,147 @@ class FlightEvidencePublicationTests(TransactionTestCase):
                 current_publication__isnull=False
             ).exists()
         )
+
+    def test_city_identity_or_legacy_inbound_mismatch_fails_closed(self):
+        departure = official_page([official_row()])
+        arrival = official_page(
+            [official_row(flightId="KE704", masterFlightId="KE704", st="2015")]
+        )
+        duration = (
+            b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+            b"reference_locator\r\nNRT,150,165,2026-08-01,"
+            + DURATION_SOURCE_LOCATOR.encode("ascii")
+            + b"\r\n"
+        )
+        cases = (
+            (
+                city_reference_payload(
+                    overrides={"NRT": {"airportName": "잘못된 도시"}}
+                ),
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            (
+                city_reference_payload(),
+                legacy_arrival_page([legacy_arrival_row(st="2016")]),
+            ),
+        )
+        for city_payload, legacy_page in cases:
+            with self.subTest(city_sha=len(city_payload), legacy_sha=len(legacy_page)):
+                outcome = ingest_and_publish_flight_evidence(
+                    departure_pages=(departure,),
+                    arrival_pages=(arrival,),
+                    city_payload=city_payload,
+                    legacy_arrival_pages=(legacy_page,),
+                    duration_csv=duration,
+                    source_date=date(2026, 8, 31),
+                    published_by="synthetic-reviewer",
+                    source_checked_at=timezone.now(),
+                )
+                self.assertEqual(
+                    outcome.code,
+                    FlightIngestionCode.PARSE_QUARANTINED,
+                )
+        self.assertFalse(
+            PublishedFlightSchedule.objects.filter(
+                current_publication__isnull=False
+            ).exists()
+        )
+
+    def test_full_official_rows_project_to_curated_bidirectional_routes(self):
+        departure = official_page(
+            [
+                official_row(),
+                official_row(
+                    airportCode="AMS",
+                    airport="암스테르담",
+                    flightId="KE925",
+                    masterFlightId="KE925",
+                ),
+            ]
+        )
+        arrival = official_page(
+            [
+                official_row(
+                    flightId="KE704",
+                    masterFlightId="KE704",
+                    st="2015",
+                ),
+                official_row(
+                    airportCode="AMS",
+                    airport="암스테르담",
+                    flightId="KE926",
+                    masterFlightId="KE926",
+                    st="1630",
+                ),
+            ]
+        )
+        duration = (
+            b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+            b"reference_locator\r\nNRT,150,165,2026-08-01,"
+            + DURATION_SOURCE_LOCATOR.encode("ascii")
+            + b"\r\nKIX,100,110,2026-08-01,"
+            + DURATION_SOURCE_LOCATOR.encode("ascii")
+            + b"\r\n"
+        )
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=(departure,),
+            arrival_pages=(arrival,),
+            city_payload=city_reference_payload(
+                extra_rows=(
+                    {
+                        "countryName": "네덜란드",
+                        "airportName": "암스테르담",
+                        "airportCode": "AMS",
+                    },
+                )
+            ),
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            duration_csv=duration,
+            source_date=date(2026, 8, 31),
+            published_by="synthetic-reviewer",
+            source_checked_at=timezone.now(),
+        )
+
+        self.assertEqual(outcome.code, FlightIngestionCode.PUBLISHED)
+        self.assertEqual(
+            set(FlightSchedule.objects.values_list("destination_airport__iata_code", flat=True)),
+            {"NRT"},
+        )
+        self.assertEqual(FlightSchedule.objects.count(), 2)
+
+    def test_curated_route_without_both_directions_is_quarantined(self):
+        outcome = ingest_and_publish_flight_evidence(
+            departure_pages=(official_page([official_row()]),),
+            arrival_pages=(
+                official_page(
+                    [
+                        official_row(
+                            airportCode="AMS",
+                            airport="암스테르담",
+                            flightId="KE926",
+                            masterFlightId="KE926",
+                        )
+                    ]
+                ),
+            ),
+            city_payload=city_reference_payload(),
+            legacy_arrival_pages=(
+                legacy_arrival_page([legacy_arrival_row()]),
+            ),
+            duration_csv=(
+                b"destination_iata,outbound_minutes,inbound_minutes,reference_date,"
+                b"reference_locator\r\nNRT,150,165,2026-08-01,"
+                + DURATION_SOURCE_LOCATOR.encode("ascii")
+                + b"\r\n"
+            ),
+            source_date=date(2026, 8, 31),
+            published_by="synthetic-reviewer",
+            source_checked_at=timezone.now(),
+        )
+
+        self.assertEqual(
+            outcome.code,
+            FlightIngestionCode.PARSE_QUARANTINED,
+        )


