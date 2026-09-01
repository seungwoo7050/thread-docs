# 불변 국가·공항 정체성과 지원 범위 확장

## `docs: isolate the shared country identity`

diff --git a/docs/TECHNOLOGY-DECISIONS.md b/docs/TECHNOLOGY-DECISIONS.md
index 31090e9..1caf6cc 100644
--- a/docs/TECHNOLOGY-DECISIONS.md
+++ b/docs/TECHNOLOGY-DECISIONS.md
@@ -29,6 +29,7 @@ Kubernetes와 microservice를 추가하지 않습니다. 사용자 규모만으
 구현 시 단일 Django project 아래 다음 Python app 경계를 사용합니다.
 
 - `sources`: source configuration, rights, attempts, artifacts와 adapter protocol
+- `countries`: 승인된 내부 Country identity allowlist만 소유하며 source나 publication 상태는 소유하지 않음
 - `entry_requirements`: PassportPolicy, EntryFactRevision과 publication
 - `travel_warnings`: warning revision과 publication
 - `travel_windows`: source gate 통과 후 별도 migration으로 활성화
@@ -38,6 +39,8 @@ Kubernetes와 microservice를 추가하지 않습니다. 사용자 규모만으
 
 공통 source app은 transport lifecycle만 공유하고 domain field를 EAV나 JSON blob으로
 일반화하지 않습니다. adapter output은 typed dataclass/form/model boundary에서 검증합니다.
+Country를 entry app 안에 두어 warning이 entry publication schema에 의존하게 만들지 않고,
+공유 identity와 두 module의 ingestion/review/publication lifecycle을 분리합니다.
 
 ## UI와 background work
 


## `feat(domain): support travel destinations and airports`

diff --git a/countries/migrations/0002_generalize_country_identity.py b/countries/migrations/0002_generalize_country_identity.py
new file mode 100644
index 0000000..4474ca5
--- /dev/null
+++ b/countries/migrations/0002_generalize_country_identity.py
@@ -0,0 +1,98 @@
+import uuid
+
+from django.db import migrations, models
+from django.db.models import Q
+
+
+JP_COUNTRY_ID = uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9")
+SUPPORTED_COUNTRIES = (
+    (JP_COUNTRY_ID, "JP", "일본", "Japan", True),
+    (uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"), "TW", "대만", "Taiwan", True),
+    (uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"), "HK", "홍콩", "Hong Kong", True),
+    (uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"), "MO", "마카오", "Macau", True),
+    (uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"), "VN", "베트남", "Vietnam", True),
+    (uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"), "TH", "태국", "Thailand", True),
+)
+
+
+def seed_supported_countries(apps, schema_editor):
+    Country = apps.get_model("countries", "Country")
+    alias = schema_editor.connection.alias
+    for country_id, iso_alpha2, name_ko, name_en, is_public in SUPPORTED_COUNTRIES:
+        country, _ = Country.objects.using(alias).get_or_create(
+            id=country_id,
+            defaults={
+                "iso_alpha2": iso_alpha2,
+                "name_ko": name_ko,
+                "name_en": name_en,
+                "is_public": is_public,
+            },
+        )
+        if (
+            country.iso_alpha2,
+            country.name_ko,
+            country.name_en,
+            country.is_public,
+        ) != (iso_alpha2, name_ko, name_en, is_public):
+            raise RuntimeError(f"country identity conflicts with {iso_alpha2}")
+
+
+def require_jp_only(apps, schema_editor):
+    Country = apps.get_model("countries", "Country")
+    alias = schema_editor.connection.alias
+    rows = list(Country.objects.using(alias).exclude(id=JP_COUNTRY_ID))
+    supported_extra_ids = {country[0] for country in SUPPORTED_COUNTRIES[1:]}
+    if {country.id for country in rows} != supported_extra_ids:
+        raise RuntimeError("country generalization rollback requires the original JP row only")
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            "ALTER TABLE countries_country DISABLE TRIGGER countries_country_immutable_guard"
+        )
+    Country.objects.using(alias).filter(id__in=supported_extra_ids).delete()
+    with schema_editor.connection.cursor() as cursor:
+        cursor.execute(
+            "ALTER TABLE countries_country ENABLE TRIGGER countries_country_immutable_guard"
+        )
+
+
+class Migration(migrations.Migration):
+    dependencies = [("countries", "0001_initial")]
+
+    operations = [
+        migrations.RemoveConstraint(
+            model_name="country",
+            name="country_jp_exact_allowlist",
+        ),
+        migrations.AlterField(
+            model_name="country",
+            name="id",
+            field=models.UUIDField(
+                default=uuid.uuid4,
+                editable=False,
+                primary_key=True,
+                serialize=False,
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="country",
+            constraint=models.CheckConstraint(
+                condition=Q(iso_alpha2__regex=r"^[A-Z]{2}$"),
+                name="country_iso_alpha2_format",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="country",
+            constraint=models.CheckConstraint(
+                condition=~Q(name_ko=""),
+                name="country_name_ko_present",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="country",
+            constraint=models.CheckConstraint(
+                condition=~Q(name_en=""),
+                name="country_name_en_present",
+            ),
+        ),
+        migrations.RunPython(seed_supported_countries, require_jp_only),
+    ]
diff --git a/countries/models.py b/countries/models.py
index e80049e..37facac 100644
--- a/countries/models.py
+++ b/countries/models.py
@@ -5,10 +5,18 @@ from django.db.models import Q
 
 
 JP_COUNTRY_ID = uuid.UUID("575fa8b9-14f9-526e-9464-ebd1dea76da9")
+SUPPORTED_COUNTRY_IDS = {
+    "JP": JP_COUNTRY_ID,
+    "TW": uuid.UUID("3d374024-be31-5be3-99b3-fc28626b076a"),
+    "HK": uuid.UUID("008d7e8f-412e-53ca-a5c6-d06a9fbafda8"),
+    "MO": uuid.UUID("55d20bb0-9d97-5a53-9600-e8f102f38fe9"),
+    "VN": uuid.UUID("17e47e71-07e3-57e6-8c72-e1f8b47e34df"),
+    "TH": uuid.UUID("5438e3c3-df2b-593a-8f04-7e64e66219e7"),
+}
 
 
 class Country(models.Model):
-    id = models.UUIDField(primary_key=True, default=JP_COUNTRY_ID, editable=False)
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
     iso_alpha2 = models.CharField(max_length=2, unique=True)
     name_ko = models.CharField(max_length=100)
     name_en = models.CharField(max_length=100)
@@ -17,15 +25,11 @@ class Country(models.Model):
     class Meta:
         constraints = [
             models.CheckConstraint(
-                condition=Q(
-                    id=JP_COUNTRY_ID,
-                    iso_alpha2="JP",
-                    name_ko="일본",
-                    name_en="Japan",
-                    is_public=True,
-                ),
-                name="country_jp_exact_allowlist",
-            )
+                condition=Q(iso_alpha2__regex=r"^[A-Z]{2}$"),
+                name="country_iso_alpha2_format",
+            ),
+            models.CheckConstraint(condition=~Q(name_ko=""), name="country_name_ko_present"),
+            models.CheckConstraint(condition=~Q(name_en=""), name="country_name_en_present"),
         ]
 
     def __str__(self) -> str:
diff --git a/countries/tests/test_country.py b/countries/tests/test_country.py
index 6e665f4..ffea9a6 100644
--- a/countries/tests/test_country.py
+++ b/countries/tests/test_country.py
@@ -11,16 +11,20 @@ from operations.tests.migration_helpers import (
     restore_canonical_migration_graph,
 )
 
-from countries.models import Country, JP_COUNTRY_ID
+from countries.models import Country, JP_COUNTRY_ID, SUPPORTED_COUNTRY_IDS
+
+
+generalization = importlib.import_module(
+    "countries.migrations.0002_generalize_country_identity"
+)
 
 
 migration = importlib.import_module("countries.migrations.0001_initial")
 
 
 class CountryTests(TestCase):
-    def test_seed_is_the_exact_public_jp_allowlist(self):
-        self.assertEqual(Country.objects.count(), 1)
-        country = Country.objects.get()
+    def test_seed_preserves_the_exact_public_jp_identity(self):
+        country = Country.objects.get(pk=JP_COUNTRY_ID)
         self.assertEqual(country.id, JP_COUNTRY_ID)
         self.assertIsInstance(country.id, uuid.UUID)
         self.assertEqual(country.iso_alpha2, "JP")
@@ -32,20 +36,30 @@ class CountryTests(TestCase):
         schema_editor = SimpleNamespace(connection=connection)
         migration.seed_jp(apps, schema_editor)
         migration.seed_jp(apps, schema_editor)
-        self.assertEqual(Country.objects.count(), 1)
+        generalization.seed_supported_countries(apps, schema_editor)
+        generalization.seed_supported_countries(apps, schema_editor)
+        self.assertEqual(
+            set(Country.objects.values_list("iso_alpha2", flat=True)),
+            set(SUPPORTED_COUNTRY_IDS),
+        )
 
     def test_country_identity_cannot_be_updated_or_deleted(self):
-        country = Country.objects.get()
+        country = Country.objects.get(pk=JP_COUNTRY_ID)
         with self.assertRaises(IntegrityError), transaction.atomic():
             Country.objects.filter(pk=country.pk).update(name_en="Changed")
         with self.assertRaises(IntegrityError), transaction.atomic():
             Country.objects.filter(pk=country.pk).delete()
 
-    def test_unapproved_country_or_jp_alias_is_rejected(self):
+    def test_additional_countries_use_distinct_valid_iso_identities(self):
+        country = Country.objects.get(iso_alpha2="TW")
+        self.assertNotEqual(country.id, JP_COUNTRY_ID)
+        self.assertEqual(country.iso_alpha2, "TW")
+
+    def test_invalid_country_or_duplicate_jp_alias_is_rejected(self):
         with self.assertRaises(IntegrityError), transaction.atomic():
             Country.objects.create(
                 id=uuid.uuid4(),
-                iso_alpha2="KR",
+                iso_alpha2="kr",
                 name_ko="대한민국",
                 name_en="South Korea",
                 is_public=True,
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index 491d2a1..22df215 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -155,6 +155,7 @@ INSTALLED_APPS = [
     "sources",
     "entry_requirements",
     "travel_warnings",
+    "travel_windows",
     "reviews",
     "public_web",
     "reviews.admin_config.DormantAdminConfig",
diff --git a/travel_windows/__init__.py b/travel_windows/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/travel_windows/apps.py b/travel_windows/apps.py
new file mode 100644
index 0000000..fda25b4
--- /dev/null
+++ b/travel_windows/apps.py
@@ -0,0 +1,6 @@
+from django.apps import AppConfig
+
+
+class TravelWindowsConfig(AppConfig):
+    default_auto_field = "django.db.models.BigAutoField"
+    name = "travel_windows"
diff --git a/travel_windows/migrations/0001_initial.py b/travel_windows/migrations/0001_initial.py
new file mode 100644
index 0000000..781876b
--- /dev/null
+++ b/travel_windows/migrations/0001_initial.py
@@ -0,0 +1,43 @@
+import django.db.models.deletion
+import uuid
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    initial = True
+    dependencies = [("countries", "0002_generalize_country_identity")]
+
+    operations = [
+        migrations.CreateModel(
+            name="Airport",
+            fields=[
+                ("id", models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True, serialize=False)),
+                ("iata_code", models.CharField(max_length=3, unique=True)),
+                ("city_code", models.CharField(max_length=3)),
+                ("city_name_ko", models.CharField(max_length=100)),
+                ("name_ko", models.CharField(max_length=120)),
+                ("iana_timezone", models.CharField(max_length=64)),
+                ("timezone_evidence_locator", models.URLField(max_length=500)),
+                ("is_public", models.BooleanField(default=True)),
+                (
+                    "country",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="airports",
+                        to="countries.country",
+                    ),
+                ),
+            ],
+            options={
+                "ordering": ("city_code", "iata_code"),
+                "constraints": [
+                    models.CheckConstraint(condition=models.Q(("iata_code__regex", "^[A-Z]{3}$")), name="airport_iata_format"),
+                    models.CheckConstraint(condition=models.Q(("city_code__regex", "^[A-Z]{3}$")), name="airport_city_code_format"),
+                    models.CheckConstraint(condition=models.Q(("city_name_ko", ""), _negated=True), name="airport_city_name_present"),
+                    models.CheckConstraint(condition=models.Q(("name_ko", ""), _negated=True), name="airport_name_present"),
+                    models.CheckConstraint(condition=models.Q(("iana_timezone__regex", "^[A-Za-z]+/[A-Za-z0-9_+\\-]+$")), name="airport_timezone_format"),
+                    models.CheckConstraint(condition=models.Q(("timezone_evidence_locator__startswith", "https://")), name="airport_timezone_evidence_https"),
+                ],
+            },
+        )
+    ]
diff --git a/travel_windows/migrations/__init__.py b/travel_windows/migrations/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/travel_windows/models.py b/travel_windows/models.py
new file mode 100644
index 0000000..381a4f2
--- /dev/null
+++ b/travel_windows/models.py
@@ -0,0 +1,46 @@
+import uuid
+
+from django.db import models
+from django.db.models import Q
+
+
+class Airport(models.Model):
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    iata_code = models.CharField(max_length=3, unique=True)
+    country = models.ForeignKey(
+        "countries.Country",
+        on_delete=models.PROTECT,
+        related_name="airports",
+    )
+    city_code = models.CharField(max_length=3)
+    city_name_ko = models.CharField(max_length=100)
+    name_ko = models.CharField(max_length=120)
+    iana_timezone = models.CharField(max_length=64)
+    timezone_evidence_locator = models.URLField(max_length=500)
+    is_public = models.BooleanField(default=True)
+
+    class Meta:
+        ordering = ("city_code", "iata_code")
+        constraints = [
+            models.CheckConstraint(
+                condition=Q(iata_code__regex=r"^[A-Z]{3}$"),
+                name="airport_iata_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(city_code__regex=r"^[A-Z]{3}$"),
+                name="airport_city_code_format",
+            ),
+            models.CheckConstraint(condition=~Q(city_name_ko=""), name="airport_city_name_present"),
+            models.CheckConstraint(condition=~Q(name_ko=""), name="airport_name_present"),
+            models.CheckConstraint(
+                condition=Q(iana_timezone__regex=r"^[A-Za-z]+/[A-Za-z0-9_+\-]+$"),
+                name="airport_timezone_format",
+            ),
+            models.CheckConstraint(
+                condition=Q(timezone_evidence_locator__startswith="https://"),
+                name="airport_timezone_evidence_https",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.iata_code} — {self.city_name_ko}"
diff --git a/travel_windows/tests/__init__.py b/travel_windows/tests/__init__.py
new file mode 100644
index 0000000..e69de29
diff --git a/travel_windows/tests/test_domain.py b/travel_windows/tests/test_domain.py
new file mode 100644
index 0000000..7df1441
--- /dev/null
+++ b/travel_windows/tests/test_domain.py
@@ -0,0 +1,45 @@
+from django.db import IntegrityError, transaction
+from django.test import TestCase
+
+from countries.models import Country, JP_COUNTRY_ID
+from travel_windows.models import Airport
+
+
+class TravelDestinationDomainTests(TestCase):
+    def test_existing_jp_identity_is_preserved_and_new_countries_are_supported(self):
+        self.assertTrue(Country.objects.filter(pk=JP_COUNTRY_ID, iso_alpha2="JP").exists())
+        taiwan = Country.objects.get(iso_alpha2="TW")
+        self.assertNotEqual(taiwan.id, JP_COUNTRY_ID)
+
+    def test_country_format_is_fail_closed(self):
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            Country.objects.create(iso_alpha2="tw", name_ko="대만", name_en="Taiwan")
+
+    def test_airport_keeps_country_city_timezone_and_evidence(self):
+        airport = Airport.objects.create(
+            iata_code="NRT",
+            country=Country.objects.get(pk=JP_COUNTRY_ID),
+            city_code="TYO",
+            city_name_ko="도쿄",
+            name_ko="나리타 국제공항",
+            iana_timezone="Asia/Tokyo",
+            timezone_evidence_locator="https://example.invalid/timezones/NRT",
+        )
+        self.assertEqual(airport.country.iso_alpha2, "JP")
+        self.assertEqual(airport.iana_timezone, "Asia/Tokyo")
+
+    def test_airport_identity_and_timezone_shape_are_bounded(self):
+        country = Country.objects.get(pk=JP_COUNTRY_ID)
+        for values in (
+            {"iata_code": "nrt", "city_code": "TYO", "iana_timezone": "Asia/Tokyo"},
+            {"iata_code": "NRT", "city_code": "tyo", "iana_timezone": "Asia/Tokyo"},
+            {"iata_code": "NRT", "city_code": "TYO", "iana_timezone": "UTC"},
+        ):
+            with self.assertRaises(IntegrityError), transaction.atomic():
+                Airport.objects.create(
+                    country=country,
+                    city_name_ko="도쿄",
+                    name_ko="나리타 국제공항",
+                    timezone_evidence_locator="https://example.invalid/timezones/NRT",
+                    **values,
+                )


