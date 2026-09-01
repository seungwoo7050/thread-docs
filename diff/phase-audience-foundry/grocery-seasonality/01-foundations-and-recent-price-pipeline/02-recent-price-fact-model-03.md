## `feat(price): identify exact retail series`

diff --git a/grocery/migrations/0003_price_series_key.py b/grocery/migrations/0003_price_series_key.py
new file mode 100644
index 0000000..27efb61
--- /dev/null
+++ b/grocery/migrations/0003_price_series_key.py
@@ -0,0 +1,154 @@
+import uuid
+
+import django.core.validators
+from django.db import migrations, models
+
+IMMUTABLE_TRIGGER_SQL = """
+CREATE FUNCTION grocery_priceserieskey_reject_mutation()
+RETURNS trigger
+LANGUAGE plpgsql
+AS $function$
+BEGIN
+    RAISE EXCEPTION 'grocery_priceserieskey rows are immutable'
+        USING ERRCODE = '55000';
+END;
+$function$;
+
+CREATE TRIGGER grocery_priceserieskey_immutable
+BEFORE UPDATE OR DELETE ON grocery_priceserieskey
+FOR EACH ROW
+EXECUTE FUNCTION grocery_priceserieskey_reject_mutation();
+"""
+
+DROP_IMMUTABLE_TRIGGER_SQL = """
+DROP TRIGGER IF EXISTS grocery_priceserieskey_immutable ON grocery_priceserieskey;
+DROP FUNCTION IF EXISTS grocery_priceserieskey_reject_mutation();
+"""
+
+
+class Migration(migrations.Migration):
+    dependencies = [("grocery", "0002_artifact_parse_runs")]
+
+    operations = [
+        migrations.CreateModel(
+            name="PriceSeriesKey",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4,
+                        editable=False,
+                        primary_key=True,
+                        serialize=False,
+                    ),
+                ),
+                (
+                    "product_class_code",
+                    models.CharField(
+                        choices=[("01", "Retail")],
+                        default="01",
+                        max_length=2,
+                    ),
+                ),
+                ("product_class_name", models.CharField(max_length=100)),
+                (
+                    "category_code",
+                    models.CharField(
+                        choices=[("200", "Vegetables"), ("400", "Fruit")],
+                        max_length=3,
+                    ),
+                ),
+                ("category_name", models.CharField(max_length=100)),
+                (
+                    "item_code",
+                    models.CharField(
+                        max_length=32,
+                        validators=[django.core.validators.RegexValidator("^[0-9]+$")],
+                    ),
+                ),
+                ("item_name", models.CharField(max_length=200)),
+                (
+                    "variety_code",
+                    models.CharField(
+                        max_length=32,
+                        validators=[django.core.validators.RegexValidator("^[0-9]+$")],
+                    ),
+                ),
+                ("variety_name", models.CharField(max_length=200)),
+                (
+                    "grade_code",
+                    models.CharField(
+                        max_length=32,
+                        validators=[django.core.validators.RegexValidator("^[0-9]+$")],
+                    ),
+                ),
+                ("grade_name", models.CharField(max_length=200)),
+                ("raw_unit", models.CharField(max_length=64)),
+                ("raw_unit_size", models.CharField(max_length=64)),
+                ("coverage_identity", models.CharField(max_length=128)),
+                ("identity_evidence_revision", models.CharField(max_length=128)),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=(
+                            "product_class_code",
+                            "category_code",
+                            "item_code",
+                            "variety_code",
+                            "grade_code",
+                            "raw_unit",
+                            "raw_unit_size",
+                            "coverage_identity",
+                        ),
+                        name="grocery_series_semantic_identity_uniq",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("product_class_code", "01")),
+                        name="grocery_series_product_class_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("category_code__in", ("200", "400"))),
+                        name="grocery_series_category_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("item_code__regex", "^[0-9]+$"),
+                            ("variety_code__regex", "^[0-9]+$"),
+                            ("grade_code__regex", "^[0-9]+$"),
+                        ),
+                        name="grocery_series_codes_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("product_class_name", ""), _negated=True),
+                            models.Q(("category_name", ""), _negated=True),
+                            models.Q(("item_name", ""), _negated=True),
+                            models.Q(("variety_name", ""), _negated=True),
+                            models.Q(("grade_name", ""), _negated=True),
+                        ),
+                        name="grocery_series_names_nonempty",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("raw_unit", ""), _negated=True),
+                            models.Q(("raw_unit_size", ""), _negated=True),
+                        ),
+                        name="grocery_series_raw_unit_nonempty",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("coverage_identity", ""), _negated=True),
+                            models.Q(("identity_evidence_revision", ""), _negated=True),
+                        ),
+                        name="grocery_series_evidence_nonempty",
+                    ),
+                ]
+            },
+        ),
+        migrations.RunSQL(
+            sql=IMMUTABLE_TRIGGER_SQL,
+            reverse_sql=DROP_IMMUTABLE_TRIGGER_SQL,
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index 87773bf..b0dfa7b 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -12,6 +12,7 @@ from django.db.models import F, Q
 from django.utils import timezone
 
 SHA256_PATTERN = r"^[0-9a-f]{64}$"
+DIGIT_CODE_PATTERN = r"^[0-9]+$"
 sha256_validator = RegexValidator(
     regex=SHA256_PATTERN,
     message="Enter a lowercase 64-character SHA-256 digest.",
@@ -825,6 +826,167 @@ class ParseRun(models.Model):
         super().save(*args, **kwargs)
 
 
+class PriceSeriesKey(models.Model):
+    """Immutable semantic identity for one reviewed KAMIS retail price series."""
+
+    class ProductClass(models.TextChoices):
+        RETAIL = "01", "Retail"
+
+    class Category(models.TextChoices):
+        VEGETABLE = "200", "Vegetables"
+        FRUIT = "400", "Fruit"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    product_class_code = models.CharField(
+        max_length=2,
+        choices=ProductClass.choices,
+        default=ProductClass.RETAIL,
+    )
+    product_class_name = models.CharField(max_length=100)
+    category_code = models.CharField(max_length=3, choices=Category.choices)
+    category_name = models.CharField(max_length=100)
+    item_code = models.CharField(
+        max_length=32,
+        validators=[RegexValidator(DIGIT_CODE_PATTERN)],
+    )
+    item_name = models.CharField(max_length=200)
+    variety_code = models.CharField(
+        max_length=32,
+        validators=[RegexValidator(DIGIT_CODE_PATTERN)],
+    )
+    variety_name = models.CharField(max_length=200)
+    grade_code = models.CharField(
+        max_length=32,
+        validators=[RegexValidator(DIGIT_CODE_PATTERN)],
+    )
+    grade_name = models.CharField(max_length=200)
+    raw_unit = models.CharField(max_length=64)
+    raw_unit_size = models.CharField(max_length=64)
+    coverage_identity = models.CharField(max_length=128)
+    identity_evidence_revision = models.CharField(max_length=128)
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=(
+                    "product_class_code",
+                    "category_code",
+                    "item_code",
+                    "variety_code",
+                    "grade_code",
+                    "raw_unit",
+                    "raw_unit_size",
+                    "coverage_identity",
+                ),
+                name="grocery_series_semantic_identity_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(product_class_code="01"),
+                name="grocery_series_product_class_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(category_code__in=("200", "400")),
+                name="grocery_series_category_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(item_code__regex=DIGIT_CODE_PATTERN)
+                    & Q(variety_code__regex=DIGIT_CODE_PATTERN)
+                    & Q(grade_code__regex=DIGIT_CODE_PATTERN)
+                ),
+                name="grocery_series_codes_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    ~Q(product_class_name="")
+                    & ~Q(category_name="")
+                    & ~Q(item_name="")
+                    & ~Q(variety_name="")
+                    & ~Q(grade_name="")
+                ),
+                name="grocery_series_names_nonempty",
+            ),
+            models.CheckConstraint(
+                condition=~Q(raw_unit="") & ~Q(raw_unit_size=""),
+                name="grocery_series_raw_unit_nonempty",
+            ),
+            models.CheckConstraint(
+                condition=~Q(coverage_identity="") & ~Q(identity_evidence_revision=""),
+                name="grocery_series_evidence_nonempty",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return ":".join(
+            (
+                self.product_class_code,
+                self.category_code,
+                self.item_code,
+                self.variety_code,
+                self.grade_code,
+                self.raw_unit,
+                self.raw_unit_size,
+                self.coverage_identity,
+            )
+        )
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Price series keys are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def delete(self, *args: Any, **kwargs: Any) -> tuple[int, dict[str, int]]:
+        raise ValidationError("Price series keys are immutable.")
+
+    @classmethod
+    @transaction.atomic
+    def get_or_validate(
+        cls,
+        *,
+        product_class_code: str,
+        product_class_name: str,
+        category_code: str,
+        category_name: str,
+        item_code: str,
+        item_name: str,
+        variety_code: str,
+        variety_name: str,
+        grade_code: str,
+        grade_name: str,
+        raw_unit: str,
+        raw_unit_size: str,
+        coverage_identity: str,
+        identity_evidence_revision: str,
+    ) -> PriceSeriesKey:
+        identity = {
+            "product_class_code": product_class_code,
+            "category_code": category_code,
+            "item_code": item_code,
+            "variety_code": variety_code,
+            "grade_code": grade_code,
+            "raw_unit": raw_unit,
+            "raw_unit_size": raw_unit_size,
+            "coverage_identity": coverage_identity,
+        }
+        reviewed_fields = {
+            "product_class_name": product_class_name,
+            "category_name": category_name,
+            "item_name": item_name,
+            "variety_name": variety_name,
+            "grade_name": grade_name,
+            "identity_evidence_revision": identity_evidence_revision,
+        }
+        series, _ = cls.objects.get_or_create(**identity, defaults=reviewed_fields)
+        if any(getattr(series, field) != value for field, value in reviewed_fields.items()):
+            raise ValidationError(
+                "Price series display identity or evidence revision drifted for an existing "
+                "semantic identity."
+            )
+        return series
+
+
 def ordered_page_manifest_sha256(receipts: Sequence[PageReceipt]) -> str:
     manifest = [receipt.body_sha256 for receipt in receipts]
     canonical = json.dumps(manifest, ensure_ascii=True, separators=(",", ":")).encode("ascii")
diff --git a/grocery/tests/test_price_series_key_models.py b/grocery/tests/test_price_series_key_models.py
new file mode 100644
index 0000000..2e95518
--- /dev/null
+++ b/grocery/tests/test_price_series_key_models.py
@@ -0,0 +1,145 @@
+import uuid
+
+from django.core.exceptions import ValidationError
+from django.db import DatabaseError, IntegrityError, connection, transaction
+from django.test import TestCase
+
+from grocery.models import PriceSeriesKey
+
+
+def series_fields(**overrides: str) -> dict[str, str]:
+    fields = {
+        "product_class_code": "01",
+        "product_class_name": "소매",
+        "category_code": "200",
+        "category_name": "채소류",
+        "item_code": "212",
+        "item_name": "배추",
+        "variety_code": "00",
+        "variety_name": "월동",
+        "grade_code": "04",
+        "grade_name": "상품",
+        "raw_unit": "포기",
+        "raw_unit_size": "1",
+        "coverage_identity": "KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1",
+        "identity_evidence_revision": "kamis-codebook-and-coverage-v1",
+    }
+    fields.update(overrides)
+    return fields
+
+
+def create_series(**overrides: str) -> PriceSeriesKey:
+    return PriceSeriesKey.objects.create(**series_fields(**overrides))
+
+
+def get_or_validate_series(**overrides: str) -> PriceSeriesKey:
+    fields = series_fields(**overrides)
+    return PriceSeriesKey.get_or_validate(
+        product_class_code=fields["product_class_code"],
+        product_class_name=fields["product_class_name"],
+        category_code=fields["category_code"],
+        category_name=fields["category_name"],
+        item_code=fields["item_code"],
+        item_name=fields["item_name"],
+        variety_code=fields["variety_code"],
+        variety_name=fields["variety_name"],
+        grade_code=fields["grade_code"],
+        grade_name=fields["grade_name"],
+        raw_unit=fields["raw_unit"],
+        raw_unit_size=fields["raw_unit_size"],
+        coverage_identity=fields["coverage_identity"],
+        identity_evidence_revision=fields["identity_evidence_revision"],
+    )
+
+
+class PriceSeriesKeyTests(TestCase):
+    def test_valid_series_preserves_leading_zero_codes_and_semantic_identity(self) -> None:
+        series = create_series(item_code="006", variety_code="01", grade_code="04")
+
+        self.assertIsInstance(series.id, uuid.UUID)
+        self.assertEqual(series.product_class_code, "01")
+        self.assertEqual(series.item_code, "006")
+        self.assertEqual(series.variety_code, "01")
+        self.assertEqual(series.grade_code, "04")
+        self.assertIn(series.coverage_identity, str(series))
+
+    def test_get_or_validate_is_idempotent_and_fails_closed_on_reviewed_drift(self) -> None:
+        original = get_or_validate_series()
+        repeated = get_or_validate_series()
+
+        self.assertEqual(repeated.id, original.id)
+        self.assertEqual(PriceSeriesKey.objects.count(), 1)
+
+        for changed_field in (
+            "product_class_name",
+            "category_name",
+            "item_name",
+            "variety_name",
+            "grade_name",
+            "identity_evidence_revision",
+        ):
+            with self.subTest(changed_field=changed_field):
+                with self.assertRaisesMessage(ValidationError, "drifted"):
+                    get_or_validate_series(**{changed_field: "CHANGED"})
+
+        self.assertEqual(PriceSeriesKey.objects.count(), 1)
+
+    def test_semantic_unique_constraint_excludes_names_but_rejects_name_drift(self) -> None:
+        original = create_series()
+        duplicate = PriceSeriesKey(**series_fields(item_name="바뀐 이름"))
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PriceSeriesKey.objects.bulk_create([duplicate])
+
+        self.assertEqual(PriceSeriesKey.objects.get().id, original.id)
+
+    def test_database_checks_reject_invalid_enum_code_name_raw_and_evidence_fields(self) -> None:
+        invalid_variants = (
+            {"product_class_code": "02"},
+            {"category_code": "300"},
+            {"item_code": "A12"},
+            {"variety_code": ""},
+            {"grade_code": "04-A"},
+            {"item_name": ""},
+            {"raw_unit": ""},
+            {"raw_unit_size": ""},
+            {"coverage_identity": ""},
+            {"identity_evidence_revision": ""},
+        )
+        for overrides in invalid_variants:
+            with self.subTest(overrides=overrides):
+                invalid = PriceSeriesKey(**series_fields(**overrides))
+                with self.assertRaises(IntegrityError), transaction.atomic():
+                    PriceSeriesKey.objects.bulk_create([invalid])
+
+        self.assertFalse(PriceSeriesKey.objects.exists())
+
+    def test_model_validation_rejects_non_digit_codes_before_insert(self) -> None:
+        with self.assertRaises(ValidationError):
+            create_series(item_code="12A")
+
+    def test_orm_and_database_both_block_update_and_delete(self) -> None:
+        series = create_series()
+        series.item_name = "수정 시도"
+
+        with self.assertRaisesMessage(ValidationError, "immutable"):
+            series.save()
+        with self.assertRaisesMessage(ValidationError, "immutable"):
+            series.delete()
+
+        with self.assertRaisesMessage(DatabaseError, "immutable"), transaction.atomic():
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    "UPDATE grocery_priceserieskey SET item_name = %s WHERE id = %s",
+                    ["직접 수정 시도", series.id],
+                )
+
+        with self.assertRaisesMessage(DatabaseError, "immutable"), transaction.atomic():
+            with connection.cursor() as cursor:
+                cursor.execute(
+                    "DELETE FROM grocery_priceserieskey WHERE id = %s",
+                    [series.id],
+                )
+
+        series.refresh_from_db()
+        self.assertEqual(series.item_name, "배추")


