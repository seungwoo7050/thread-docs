# 감사 가능한 KAMIS 수집 경계

## `feat(audit): record source fetch attempts`

diff --git a/grocery/migrations/0001_initial.py b/grocery/migrations/0001_initial.py
new file mode 100644
index 0000000..9bdd0b9
--- /dev/null
+++ b/grocery/migrations/0001_initial.py
@@ -0,0 +1,625 @@
+import uuid
+
+import django.core.validators
+import django.db.models.deletion
+import django.utils.timezone
+from django.db import migrations, models
+
+import grocery.models
+
+
+class Migration(migrations.Migration):
+    initial = True
+
+    dependencies = []
+
+    operations = [
+        migrations.CreateModel(
+            name="SourceConfiguration",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                ("source_owner_name", models.CharField(max_length=200)),
+                ("dataset_id", models.CharField(max_length=64)),
+                ("configuration_revision", models.CharField(max_length=64)),
+                ("interface_revision", models.CharField(max_length=64)),
+                (
+                    "state",
+                    models.CharField(
+                        choices=[
+                            ("DRAFT", "Draft"),
+                            ("RIGHTS_APPROVED", "Rights approved"),
+                            ("ACTIVE", "Active"),
+                            ("PAUSED", "Paused"),
+                            ("REJECTED", "Rejected"),
+                        ],
+                        default="DRAFT",
+                        max_length=32,
+                    ),
+                ),
+                ("state_changed_at", models.DateTimeField(default=django.utils.timezone.now)),
+                (
+                    "publication_mode",
+                    models.CharField(
+                        choices=[
+                            ("RECENT_COMPARISON", "Recent comparison"),
+                            ("CURRENT_ONLY", "Current only"),
+                            ("STATIC_MONTHLY_FILE", "Static monthly file"),
+                        ],
+                        max_length=32,
+                    ),
+                ),
+                ("coverage_identity", models.CharField(max_length=128)),
+                ("coverage_evidence_revision", models.CharField(max_length=64)),
+                (
+                    "endpoint_scheme",
+                    models.CharField(choices=[("https", "HTTPS")], default="https", max_length=8),
+                ),
+                (
+                    "endpoint_host",
+                    models.CharField(
+                        max_length=253, validators=[grocery.models.validate_endpoint_host]
+                    ),
+                ),
+                (
+                    "endpoint_path",
+                    models.CharField(
+                        max_length=512, validators=[grocery.models.validate_endpoint_path]
+                    ),
+                ),
+                (
+                    "endpoint_method",
+                    models.CharField(choices=[("GET", "GET")], default="GET", max_length=8),
+                ),
+                (
+                    "authentication_mode",
+                    models.CharField(
+                        choices=[
+                            ("NONE", "None"),
+                            ("DATA_GO_KR_SERVICE_KEY", "data.go.kr service key"),
+                        ],
+                        max_length=32,
+                    ),
+                ),
+                (
+                    "logical_secret_name",
+                    models.CharField(
+                        blank=True,
+                        max_length=128,
+                        validators=[django.core.validators.RegexValidator("^[A-Z][A-Z0-9_]*$")],
+                    ),
+                ),
+                ("provider_quota_limit", models.PositiveIntegerField()),
+                (
+                    "provider_quota_period",
+                    models.CharField(
+                        choices=[
+                            ("UNSPECIFIED", "Provider did not specify"),
+                            ("DAY", "Per day"),
+                            ("SECOND", "Per second"),
+                        ],
+                        max_length=16,
+                    ),
+                ),
+                ("request_timeout_seconds", models.PositiveSmallIntegerField()),
+                (
+                    "retry_policy",
+                    models.CharField(
+                        choices=[("BOUNDED_TRANSIENT_ONLY", "Bounded transient failures only")],
+                        max_length=32,
+                    ),
+                ),
+                ("max_retries", models.PositiveSmallIntegerField(default=0)),
+                ("max_requests_per_attempt", models.PositiveSmallIntegerField()),
+                ("max_pages_per_attempt", models.PositiveSmallIntegerField()),
+                ("max_page_bytes", models.PositiveIntegerField()),
+                ("rights_evidence_locator", models.URLField(blank=True, max_length=500)),
+                (
+                    "rights_evidence_sha256",
+                    models.CharField(
+                        blank=True,
+                        default="",
+                        max_length=64,
+                        validators=[
+                            django.core.validators.RegexValidator(
+                                message="Enter a lowercase 64-character SHA-256 digest.",
+                                regex="^[0-9a-f]{64}$",
+                            )
+                        ],
+                    ),
+                ),
+                ("rights_confirmed_at", models.DateTimeField(blank=True, null=True)),
+                (
+                    "raw_retention",
+                    models.CharField(
+                        choices=[("HASH_ONLY", "Hash only")], default="HASH_ONLY", max_length=16
+                    ),
+                ),
+                ("created_at", models.DateTimeField(auto_now_add=True)),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=("dataset_id", "configuration_revision"),
+                        name="grocery_source_dataset_revision_uniq",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            (
+                                "state__in",
+                                ("DRAFT", "RIGHTS_APPROVED", "ACTIVE", "PAUSED", "REJECTED"),
+                            )
+                        ),
+                        name="grocery_source_state_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            (
+                                "publication_mode__in",
+                                ("RECENT_COMPARISON", "CURRENT_ONLY", "STATIC_MONTHLY_FILE"),
+                            )
+                        ),
+                        name="grocery_source_publication_mode_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("endpoint_scheme", "https")),
+                        name="grocery_source_endpoint_scheme_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("endpoint_host__regex", "^[a-z0-9][a-z0-9.-]*[a-z0-9]$")
+                        ),
+                        name="grocery_source_endpoint_host_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("endpoint_path__startswith", "/"),
+                            models.Q(("endpoint_path__contains", "?"), _negated=True),
+                            models.Q(("endpoint_path__contains", "#"), _negated=True),
+                        ),
+                        name="grocery_source_endpoint_path_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("endpoint_method", "GET")),
+                        name="grocery_source_endpoint_method_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("authentication_mode__in", ("NONE", "DATA_GO_KR_SERVICE_KEY"))
+                        ),
+                        name="grocery_source_auth_mode_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(("authentication_mode", "NONE"), ("logical_secret_name", "")),
+                            models.Q(
+                                ("authentication_mode", "DATA_GO_KR_SERVICE_KEY"),
+                                ("logical_secret_name__regex", "^[A-Z][A-Z0-9_]*$"),
+                            ),
+                            _connector="OR",
+                        ),
+                        name="grocery_source_secret_reference_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("provider_quota_limit__gt", 0)),
+                        name="grocery_source_quota_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("provider_quota_period__in", ("UNSPECIFIED", "DAY", "SECOND"))
+                        ),
+                        name="grocery_source_quota_period_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("request_timeout_seconds__gt", 0)),
+                        name="grocery_source_timeout_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("retry_policy", "BOUNDED_TRANSIENT_ONLY")),
+                        name="grocery_source_retry_policy_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("max_retries__gte", 0)),
+                        name="grocery_source_retries_nonnegative",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("max_requests_per_attempt__gt", 0)),
+                        name="grocery_source_request_budget_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("max_pages_per_attempt__gt", 0)),
+                        name="grocery_source_page_budget_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("max_page_bytes__gt", 0)),
+                        name="grocery_source_byte_budget_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("rights_evidence_sha256", ""),
+                            ("rights_evidence_sha256__regex", "^[0-9a-f]{64}$"),
+                            _connector="OR",
+                        ),
+                        name="grocery_source_rights_hash_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(
+                                ("rights_confirmed_at__isnull", True),
+                                ("rights_evidence_locator", ""),
+                                ("rights_evidence_sha256", ""),
+                            ),
+                            models.Q(
+                                models.Q(("rights_evidence_locator", ""), _negated=True),
+                                models.Q(("rights_evidence_sha256", ""), _negated=True),
+                                ("rights_confirmed_at__isnull", False),
+                            ),
+                            _connector="OR",
+                        ),
+                        name="grocery_source_rights_evidence_complete",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(
+                                ("state__in", ("RIGHTS_APPROVED", "ACTIVE", "PAUSED")),
+                                _negated=True,
+                            ),
+                            ("rights_confirmed_at__isnull", False),
+                            _connector="OR",
+                        ),
+                        name="grocery_source_approved_rights_present",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("raw_retention", "HASH_ONLY")),
+                        name="grocery_source_retention_valid",
+                    ),
+                ],
+            },
+        ),
+        migrations.CreateModel(
+            name="FetchAttempt",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                ("acquisition_run_id", models.UUIDField()),
+                ("attempt_ordinal", models.PositiveSmallIntegerField()),
+                (
+                    "state",
+                    models.CharField(
+                        choices=[
+                            ("STARTED", "Started"),
+                            ("SUCCEEDED", "Succeeded"),
+                            ("RETRYABLE_FAILED", "Retryable failed"),
+                            ("TERMINAL_FAILED", "Terminal failed"),
+                        ],
+                        default="STARTED",
+                        max_length=24,
+                    ),
+                ),
+                ("started_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("completed_at", models.DateTimeField(blank=True, null=True)),
+                (
+                    "redacted_request_shape",
+                    models.CharField(
+                        max_length=512, validators=[grocery.models.validate_redacted_request_shape]
+                    ),
+                ),
+                ("received_page_count", models.PositiveIntegerField(default=0)),
+                ("received_row_count", models.PositiveIntegerField(default=0)),
+                ("received_byte_count", models.PositiveBigIntegerField(default=0)),
+                (
+                    "failure_class",
+                    models.CharField(
+                        blank=True,
+                        choices=[
+                            ("TIMEOUT", "Timeout"),
+                            ("NETWORK", "Network"),
+                            ("HTTP_429", "HTTP 429"),
+                            ("HTTP_5XX", "HTTP 5xx"),
+                            ("PROVIDER_TRANSIENT", "Provider transient"),
+                            ("AUTHENTICATION", "Authentication"),
+                            ("INVALID_REQUEST", "Invalid request"),
+                            ("RESPONSE_LIMIT", "Response limit"),
+                            ("SCHEMA", "Schema"),
+                            ("IDENTITY", "Identity"),
+                            ("RECONCILIATION", "Reconciliation"),
+                        ],
+                        default="",
+                        max_length=32,
+                    ),
+                ),
+                ("failure_code", models.CharField(blank=True, max_length=64)),
+                (
+                    "source_configuration",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="fetch_attempts",
+                        to="grocery.sourceconfiguration",
+                    ),
+                ),
+            ],
+        ),
+        migrations.CreateModel(
+            name="PageReceipt",
+            fields=[
+                (
+                    "id",
+                    models.UUIDField(
+                        default=uuid.uuid4, editable=False, primary_key=True, serialize=False
+                    ),
+                ),
+                ("request_ordinal", models.PositiveSmallIntegerField()),
+                ("page_number", models.PositiveIntegerField()),
+                ("received_at", models.DateTimeField(default=django.utils.timezone.now)),
+                ("http_status", models.PositiveSmallIntegerField(blank=True, null=True)),
+                ("provider_result_code", models.CharField(blank=True, max_length=32)),
+                ("declared_total_count", models.PositiveIntegerField(blank=True, null=True)),
+                ("received_row_count", models.PositiveIntegerField(default=0)),
+                (
+                    "body_state",
+                    models.CharField(
+                        choices=[("RECEIVED", "Received"), ("NOT_RECEIVED", "Not received")],
+                        max_length=16,
+                    ),
+                ),
+                ("body_byte_length", models.PositiveIntegerField(default=0)),
+                (
+                    "body_sha256",
+                    models.CharField(
+                        blank=True,
+                        default="",
+                        max_length=64,
+                        validators=[
+                            django.core.validators.RegexValidator(
+                                message="Enter a lowercase 64-character SHA-256 digest.",
+                                regex="^[0-9a-f]{64}$",
+                            )
+                        ],
+                    ),
+                ),
+                (
+                    "body_absence_reason",
+                    models.CharField(
+                        blank=True,
+                        choices=[
+                            ("TIMEOUT", "Timeout"),
+                            ("NETWORK", "Network"),
+                            ("REJECTED_BEFORE_BODY", "Rejected before body"),
+                        ],
+                        default="",
+                        max_length=32,
+                    ),
+                ),
+                (
+                    "media_type",
+                    models.CharField(
+                        blank=True,
+                        choices=[("application/json", "JSON"), ("application/xml", "XML")],
+                        default="",
+                        max_length=32,
+                    ),
+                ),
+                (
+                    "encoding",
+                    models.CharField(
+                        blank=True, choices=[("utf-8", "UTF-8")], default="", max_length=16
+                    ),
+                ),
+                (
+                    "fetch_attempt",
+                    models.ForeignKey(
+                        on_delete=django.db.models.deletion.PROTECT,
+                        related_name="page_receipts",
+                        to="grocery.fetchattempt",
+                    ),
+                ),
+            ],
+            options={
+                "constraints": [
+                    models.UniqueConstraint(
+                        fields=("fetch_attempt", "request_ordinal"),
+                        name="grocery_page_attempt_ordinal_uniq",
+                    ),
+                    models.UniqueConstraint(
+                        fields=("fetch_attempt", "page_number"),
+                        name="grocery_page_attempt_number_uniq",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("request_ordinal__gt", 0)),
+                        name="grocery_page_request_ordinal_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("page_number__gt", 0)),
+                        name="grocery_page_number_positive",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("http_status__isnull", True),
+                            models.Q(("http_status__gte", 100), ("http_status__lte", 599)),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_http_status_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("declared_total_count__isnull", True),
+                            ("declared_total_count__gte", 0),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_declared_total_nonnegative",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("received_row_count__gte", 0), ("body_byte_length__gte", 0)
+                        ),
+                        name="grocery_page_counts_nonnegative",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(("body_state__in", ("RECEIVED", "NOT_RECEIVED"))),
+                        name="grocery_page_body_state_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("body_absence_reason", ""),
+                            (
+                                "body_absence_reason__in",
+                                ("TIMEOUT", "NETWORK", "REJECTED_BEFORE_BODY"),
+                            ),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_absence_reason_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("media_type", ""),
+                            ("media_type__in", ("application/json", "application/xml")),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_media_type_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("encoding", ""), ("encoding", "utf-8"), _connector="OR"
+                        ),
+                        name="grocery_page_encoding_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            ("body_sha256", ""),
+                            ("body_sha256__regex", "^[0-9a-f]{64}$"),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_body_hash_valid",
+                    ),
+                    models.CheckConstraint(
+                        condition=models.Q(
+                            models.Q(
+                                ("body_absence_reason", ""),
+                                ("body_state", "RECEIVED"),
+                                ("http_status__isnull", False),
+                                models.Q(("body_sha256", ""), _negated=True),
+                                models.Q(("media_type", ""), _negated=True),
+                                models.Q(("encoding", ""), _negated=True),
+                            ),
+                            models.Q(
+                                ("body_byte_length", 0),
+                                ("body_sha256", ""),
+                                ("body_state", "NOT_RECEIVED"),
+                                ("encoding", ""),
+                                ("media_type", ""),
+                                ("received_row_count", 0),
+                                models.Q(("body_absence_reason", ""), _negated=True),
+                            ),
+                            _connector="OR",
+                        ),
+                        name="grocery_page_body_fields_valid",
+                    ),
+                ],
+            },
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.UniqueConstraint(
+                fields=("acquisition_run_id", "attempt_ordinal"),
+                name="grocery_fetch_run_attempt_uniq",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(("attempt_ordinal__gt", 0)),
+                name="grocery_fetch_attempt_ordinal_positive",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("state__in", ("STARTED", "SUCCEEDED", "RETRYABLE_FAILED", "TERMINAL_FAILED"))
+                ),
+                name="grocery_fetch_state_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("received_page_count__gte", 0),
+                    ("received_row_count__gte", 0),
+                    ("received_byte_count__gte", 0),
+                ),
+                name="grocery_fetch_counts_nonnegative",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("failure_class", ""),
+                    (
+                        "failure_class__in",
+                        (
+                            "TIMEOUT",
+                            "NETWORK",
+                            "HTTP_429",
+                            "HTTP_5XX",
+                            "PROVIDER_TRANSIENT",
+                            "AUTHENTICATION",
+                            "INVALID_REQUEST",
+                            "RESPONSE_LIMIT",
+                            "SCHEMA",
+                            "IDENTITY",
+                            "RECONCILIATION",
+                        ),
+                    ),
+                    _connector="OR",
+                ),
+                name="grocery_fetch_failure_class_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    models.Q(
+                        ("completed_at__isnull", True),
+                        ("failure_class", ""),
+                        ("failure_code", ""),
+                        ("state", "STARTED"),
+                    ),
+                    models.Q(
+                        ("completed_at__isnull", False),
+                        ("failure_class", ""),
+                        ("failure_code", ""),
+                        ("state", "SUCCEEDED"),
+                    ),
+                    models.Q(
+                        ("state__in", ("RETRYABLE_FAILED", "TERMINAL_FAILED")),
+                        ("completed_at__isnull", False),
+                        models.Q(("failure_class", ""), _negated=True),
+                    ),
+                    _connector="OR",
+                ),
+                name="grocery_fetch_state_outcome_valid",
+            ),
+        ),
+        migrations.AddConstraint(
+            model_name="fetchattempt",
+            constraint=models.CheckConstraint(
+                condition=models.Q(
+                    ("completed_at__isnull", True),
+                    ("completed_at__gte", models.F("started_at")),
+                    _connector="OR",
+                ),
+                name="grocery_fetch_time_order_valid",
+            ),
+        ),
+    ]
diff --git a/grocery/models.py b/grocery/models.py
index e8f8527..561c5a3 100644
--- a/grocery/models.py
+++ b/grocery/models.py
@@ -1 +1,556 @@
-# Models are introduced in additive, reversible migrations.
+import re
+import uuid
+from typing import Any
+
+from django.core.exceptions import ValidationError
+from django.core.validators import RegexValidator
+from django.db import models
+from django.db.models import F, Q
+from django.utils import timezone
+
+SHA256_PATTERN = r"^[0-9a-f]{64}$"
+sha256_validator = RegexValidator(
+    regex=SHA256_PATTERN,
+    message="Enter a lowercase 64-character SHA-256 digest.",
+)
+
+
+def validate_endpoint_host(value: str) -> None:
+    if not re.fullmatch(r"[a-z0-9](?:[a-z0-9.-]*[a-z0-9])?", value):
+        raise ValidationError("Endpoint host must be a lowercase DNS name without a port.")
+
+
+def validate_endpoint_path(value: str) -> None:
+    if not value.startswith("/") or any(character in value for character in "?#\r\n"):
+        raise ValidationError("Endpoint path must be an absolute path without a query or fragment.")
+
+
+def validate_redacted_request_shape(value: str) -> None:
+    if "://" in value or "?" in value or "\r" in value or "\n" in value:
+        raise ValidationError("Request shape must not contain a URL, query string, or newline.")
+
+
+class SourceConfiguration(models.Model):
+    class State(models.TextChoices):
+        DRAFT = "DRAFT", "Draft"
+        RIGHTS_APPROVED = "RIGHTS_APPROVED", "Rights approved"
+        ACTIVE = "ACTIVE", "Active"
+        PAUSED = "PAUSED", "Paused"
+        REJECTED = "REJECTED", "Rejected"
+
+    class PublicationMode(models.TextChoices):
+        RECENT_COMPARISON = "RECENT_COMPARISON", "Recent comparison"
+        CURRENT_ONLY = "CURRENT_ONLY", "Current only"
+        STATIC_MONTHLY_FILE = "STATIC_MONTHLY_FILE", "Static monthly file"
+
+    class EndpointScheme(models.TextChoices):
+        HTTPS = "https", "HTTPS"
+
+    class EndpointMethod(models.TextChoices):
+        GET = "GET", "GET"
+
+    class AuthenticationMode(models.TextChoices):
+        NONE = "NONE", "None"
+        DATA_GO_KR_SERVICE_KEY = "DATA_GO_KR_SERVICE_KEY", "data.go.kr service key"
+
+    class RawRetention(models.TextChoices):
+        HASH_ONLY = "HASH_ONLY", "Hash only"
+
+    class QuotaPeriod(models.TextChoices):
+        UNSPECIFIED = "UNSPECIFIED", "Provider did not specify"
+        DAY = "DAY", "Per day"
+        SECOND = "SECOND", "Per second"
+
+    class RetryPolicy(models.TextChoices):
+        BOUNDED_TRANSIENT_ONLY = "BOUNDED_TRANSIENT_ONLY", "Bounded transient failures only"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    source_owner_name = models.CharField(max_length=200)
+    dataset_id = models.CharField(max_length=64)
+    configuration_revision = models.CharField(max_length=64)
+    interface_revision = models.CharField(max_length=64)
+    state = models.CharField(max_length=32, choices=State.choices, default=State.DRAFT)
+    state_changed_at = models.DateTimeField(default=timezone.now)
+    publication_mode = models.CharField(max_length=32, choices=PublicationMode.choices)
+    coverage_identity = models.CharField(max_length=128)
+    coverage_evidence_revision = models.CharField(max_length=64)
+
+    endpoint_scheme = models.CharField(
+        max_length=8,
+        choices=EndpointScheme.choices,
+        default=EndpointScheme.HTTPS,
+    )
+    endpoint_host = models.CharField(max_length=253, validators=[validate_endpoint_host])
+    endpoint_path = models.CharField(max_length=512, validators=[validate_endpoint_path])
+    endpoint_method = models.CharField(
+        max_length=8,
+        choices=EndpointMethod.choices,
+        default=EndpointMethod.GET,
+    )
+    authentication_mode = models.CharField(max_length=32, choices=AuthenticationMode.choices)
+    logical_secret_name = models.CharField(
+        max_length=128,
+        blank=True,
+        validators=[RegexValidator(r"^[A-Z][A-Z0-9_]*$")],
+    )
+
+    provider_quota_limit = models.PositiveIntegerField()
+    provider_quota_period = models.CharField(max_length=16, choices=QuotaPeriod.choices)
+    request_timeout_seconds = models.PositiveSmallIntegerField()
+    retry_policy = models.CharField(max_length=32, choices=RetryPolicy.choices)
+    max_retries = models.PositiveSmallIntegerField(default=0)
+    max_requests_per_attempt = models.PositiveSmallIntegerField()
+    max_pages_per_attempt = models.PositiveSmallIntegerField()
+    max_page_bytes = models.PositiveIntegerField()
+
+    rights_evidence_locator = models.URLField(max_length=500, blank=True)
+    rights_evidence_sha256 = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[sha256_validator],
+    )
+    rights_confirmed_at = models.DateTimeField(null=True, blank=True)
+    raw_retention = models.CharField(
+        max_length=16,
+        choices=RawRetention.choices,
+        default=RawRetention.HASH_ONLY,
+    )
+    created_at = models.DateTimeField(auto_now_add=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("dataset_id", "configuration_revision"),
+                name="grocery_source_dataset_revision_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(state__in=("DRAFT", "RIGHTS_APPROVED", "ACTIVE", "PAUSED", "REJECTED")),
+                name="grocery_source_state_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    publication_mode__in=(
+                        "RECENT_COMPARISON",
+                        "CURRENT_ONLY",
+                        "STATIC_MONTHLY_FILE",
+                    )
+                ),
+                name="grocery_source_publication_mode_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(endpoint_scheme="https"),
+                name="grocery_source_endpoint_scheme_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(endpoint_host__regex=r"^[a-z0-9][a-z0-9.-]*[a-z0-9]$"),
+                name="grocery_source_endpoint_host_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(endpoint_path__startswith="/")
+                & ~Q(endpoint_path__contains="?")
+                & ~Q(endpoint_path__contains="#"),
+                name="grocery_source_endpoint_path_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(endpoint_method="GET"),
+                name="grocery_source_endpoint_method_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(authentication_mode__in=("NONE", "DATA_GO_KR_SERVICE_KEY")),
+                name="grocery_source_auth_mode_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(authentication_mode="NONE", logical_secret_name="")
+                    | (
+                        Q(authentication_mode="DATA_GO_KR_SERVICE_KEY")
+                        & Q(logical_secret_name__regex=r"^[A-Z][A-Z0-9_]*$")  # noqa: S106
+                    )
+                ),
+                name="grocery_source_secret_reference_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(provider_quota_limit__gt=0),
+                name="grocery_source_quota_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(provider_quota_period__in=("UNSPECIFIED", "DAY", "SECOND")),
+                name="grocery_source_quota_period_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(request_timeout_seconds__gt=0),
+                name="grocery_source_timeout_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(retry_policy="BOUNDED_TRANSIENT_ONLY"),
+                name="grocery_source_retry_policy_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(max_retries__gte=0),
+                name="grocery_source_retries_nonnegative",
+            ),
+            models.CheckConstraint(
+                condition=Q(max_requests_per_attempt__gt=0),
+                name="grocery_source_request_budget_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(max_pages_per_attempt__gt=0),
+                name="grocery_source_page_budget_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(max_page_bytes__gt=0),
+                name="grocery_source_byte_budget_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(rights_evidence_sha256="")
+                | Q(rights_evidence_sha256__regex=SHA256_PATTERN),
+                name="grocery_source_rights_hash_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        rights_evidence_locator="",
+                        rights_evidence_sha256="",
+                        rights_confirmed_at__isnull=True,
+                    )
+                    | (
+                        ~Q(rights_evidence_locator="")
+                        & ~Q(rights_evidence_sha256="")
+                        & Q(rights_confirmed_at__isnull=False)
+                    )
+                ),
+                name="grocery_source_rights_evidence_complete",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    ~Q(state__in=("RIGHTS_APPROVED", "ACTIVE", "PAUSED"))
+                    | Q(rights_confirmed_at__isnull=False)
+                ),
+                name="grocery_source_approved_rights_present",
+            ),
+            models.CheckConstraint(
+                condition=Q(raw_retention="HASH_ONLY"),
+                name="grocery_source_retention_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.dataset_id}@{self.configuration_revision}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding and self.pk:
+            persisted = type(self).objects.filter(pk=self.pk).first()
+            if persisted is not None and persisted.fetch_attempts.exists():
+                changed = any(
+                    getattr(self, field.attname) != getattr(persisted, field.attname)
+                    for field in self._meta.concrete_fields
+                    if not field.primary_key
+                )
+                if changed:
+                    raise ValidationError(
+                        "Source configuration revisions are immutable after the first fetch "
+                        "attempt."
+                    )
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def clean(self) -> None:
+        super().clean()
+        if self.authentication_mode == self.AuthenticationMode.NONE:
+            if self.logical_secret_name:
+                raise ValidationError(
+                    {"logical_secret_name": "Unauthenticated sources cannot reference a secret."}
+                )
+        elif not self.logical_secret_name:
+            raise ValidationError(
+                {"logical_secret_name": "Authenticated sources require a logical secret name."}
+            )
+
+        evidence_values = (
+            bool(self.rights_evidence_locator),
+            bool(self.rights_evidence_sha256),
+            self.rights_confirmed_at is not None,
+        )
+        if len(set(evidence_values)) != 1:
+            raise ValidationError(
+                "Rights evidence locator, hash, and confirmation time are atomic."
+            )
+        if self.state in {
+            self.State.RIGHTS_APPROVED,
+            self.State.ACTIVE,
+            self.State.PAUSED,
+        } and not all(evidence_values):
+            raise ValidationError("The selected state requires complete rights evidence.")
+
+
+class FetchAttempt(models.Model):
+    class State(models.TextChoices):
+        STARTED = "STARTED", "Started"
+        SUCCEEDED = "SUCCEEDED", "Succeeded"
+        RETRYABLE_FAILED = "RETRYABLE_FAILED", "Retryable failed"
+        TERMINAL_FAILED = "TERMINAL_FAILED", "Terminal failed"
+
+    class FailureClass(models.TextChoices):
+        TIMEOUT = "TIMEOUT", "Timeout"
+        NETWORK = "NETWORK", "Network"
+        HTTP_429 = "HTTP_429", "HTTP 429"
+        HTTP_5XX = "HTTP_5XX", "HTTP 5xx"
+        PROVIDER_TRANSIENT = "PROVIDER_TRANSIENT", "Provider transient"
+        AUTHENTICATION = "AUTHENTICATION", "Authentication"
+        INVALID_REQUEST = "INVALID_REQUEST", "Invalid request"
+        RESPONSE_LIMIT = "RESPONSE_LIMIT", "Response limit"
+        SCHEMA = "SCHEMA", "Schema"
+        IDENTITY = "IDENTITY", "Identity"
+        RECONCILIATION = "RECONCILIATION", "Reconciliation"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    source_configuration = models.ForeignKey(
+        SourceConfiguration,
+        on_delete=models.PROTECT,
+        related_name="fetch_attempts",
+    )
+    acquisition_run_id = models.UUIDField()
+    attempt_ordinal = models.PositiveSmallIntegerField()
+    state = models.CharField(max_length=24, choices=State.choices, default=State.STARTED)
+    started_at = models.DateTimeField(default=timezone.now)
+    completed_at = models.DateTimeField(null=True, blank=True)
+    redacted_request_shape = models.CharField(
+        max_length=512,
+        validators=[validate_redacted_request_shape],
+    )
+    received_page_count = models.PositiveIntegerField(default=0)
+    received_row_count = models.PositiveIntegerField(default=0)
+    received_byte_count = models.PositiveBigIntegerField(default=0)
+    failure_class = models.CharField(
+        max_length=32,
+        choices=FailureClass.choices,
+        blank=True,
+        default="",
+    )
+    failure_code = models.CharField(max_length=64, blank=True)
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("acquisition_run_id", "attempt_ordinal"),
+                name="grocery_fetch_run_attempt_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(attempt_ordinal__gt=0),
+                name="grocery_fetch_attempt_ordinal_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(
+                    state__in=("STARTED", "SUCCEEDED", "RETRYABLE_FAILED", "TERMINAL_FAILED")
+                ),
+                name="grocery_fetch_state_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(received_page_count__gte=0)
+                & Q(received_row_count__gte=0)
+                & Q(received_byte_count__gte=0),
+                name="grocery_fetch_counts_nonnegative",
+            ),
+            models.CheckConstraint(
+                condition=Q(failure_class="")
+                | Q(
+                    failure_class__in=(
+                        "TIMEOUT",
+                        "NETWORK",
+                        "HTTP_429",
+                        "HTTP_5XX",
+                        "PROVIDER_TRANSIENT",
+                        "AUTHENTICATION",
+                        "INVALID_REQUEST",
+                        "RESPONSE_LIMIT",
+                        "SCHEMA",
+                        "IDENTITY",
+                        "RECONCILIATION",
+                    )
+                ),
+                name="grocery_fetch_failure_class_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        state="STARTED",
+                        completed_at__isnull=True,
+                        failure_class="",
+                        failure_code="",
+                    )
+                    | Q(
+                        state="SUCCEEDED",
+                        completed_at__isnull=False,
+                        failure_class="",
+                        failure_code="",
+                    )
+                    | (
+                        Q(state__in=("RETRYABLE_FAILED", "TERMINAL_FAILED"))
+                        & Q(completed_at__isnull=False)
+                        & ~Q(failure_class="")
+                    )
+                ),
+                name="grocery_fetch_state_outcome_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(completed_at__isnull=True) | Q(completed_at__gte=F("started_at")),
+                name="grocery_fetch_time_order_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.acquisition_run_id}:{self.attempt_ordinal}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+
+class PageReceipt(models.Model):
+    class BodyState(models.TextChoices):
+        RECEIVED = "RECEIVED", "Received"
+        NOT_RECEIVED = "NOT_RECEIVED", "Not received"
+
+    class BodyAbsenceReason(models.TextChoices):
+        TIMEOUT = "TIMEOUT", "Timeout"
+        NETWORK = "NETWORK", "Network"
+        REJECTED_BEFORE_BODY = "REJECTED_BEFORE_BODY", "Rejected before body"
+
+    class MediaType(models.TextChoices):
+        JSON = "application/json", "JSON"
+        XML = "application/xml", "XML"
+
+    class Encoding(models.TextChoices):
+        UTF_8 = "utf-8", "UTF-8"
+
+    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
+    fetch_attempt = models.ForeignKey(
+        FetchAttempt,
+        on_delete=models.PROTECT,
+        related_name="page_receipts",
+    )
+    request_ordinal = models.PositiveSmallIntegerField()
+    page_number = models.PositiveIntegerField()
+    received_at = models.DateTimeField(default=timezone.now)
+    http_status = models.PositiveSmallIntegerField(null=True, blank=True)
+    provider_result_code = models.CharField(max_length=32, blank=True)
+    declared_total_count = models.PositiveIntegerField(null=True, blank=True)
+    received_row_count = models.PositiveIntegerField(default=0)
+    body_state = models.CharField(max_length=16, choices=BodyState.choices)
+    body_byte_length = models.PositiveIntegerField(default=0)
+    body_sha256 = models.CharField(
+        max_length=64,
+        blank=True,
+        default="",
+        validators=[sha256_validator],
+    )
+    body_absence_reason = models.CharField(
+        max_length=32,
+        choices=BodyAbsenceReason.choices,
+        blank=True,
+        default="",
+    )
+    media_type = models.CharField(
+        max_length=32,
+        choices=MediaType.choices,
+        blank=True,
+        default="",
+    )
+    encoding = models.CharField(
+        max_length=16,
+        choices=Encoding.choices,
+        blank=True,
+        default="",
+    )
+
+    class Meta:
+        constraints = [
+            models.UniqueConstraint(
+                fields=("fetch_attempt", "request_ordinal"),
+                name="grocery_page_attempt_ordinal_uniq",
+            ),
+            models.UniqueConstraint(
+                fields=("fetch_attempt", "page_number"),
+                name="grocery_page_attempt_number_uniq",
+            ),
+            models.CheckConstraint(
+                condition=Q(request_ordinal__gt=0),
+                name="grocery_page_request_ordinal_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(page_number__gt=0),
+                name="grocery_page_number_positive",
+            ),
+            models.CheckConstraint(
+                condition=Q(http_status__isnull=True)
+                | (Q(http_status__gte=100) & Q(http_status__lte=599)),
+                name="grocery_page_http_status_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(declared_total_count__isnull=True) | Q(declared_total_count__gte=0),
+                name="grocery_page_declared_total_nonnegative",
+            ),
+            models.CheckConstraint(
+                condition=Q(received_row_count__gte=0) & Q(body_byte_length__gte=0),
+                name="grocery_page_counts_nonnegative",
+            ),
+            models.CheckConstraint(
+                condition=Q(body_state__in=("RECEIVED", "NOT_RECEIVED")),
+                name="grocery_page_body_state_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(body_absence_reason="")
+                | Q(body_absence_reason__in=("TIMEOUT", "NETWORK", "REJECTED_BEFORE_BODY")),
+                name="grocery_page_absence_reason_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(media_type="")
+                | Q(media_type__in=("application/json", "application/xml")),
+                name="grocery_page_media_type_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(encoding="") | Q(encoding="utf-8"),
+                name="grocery_page_encoding_valid",
+            ),
+            models.CheckConstraint(
+                condition=Q(body_sha256="") | Q(body_sha256__regex=SHA256_PATTERN),
+                name="grocery_page_body_hash_valid",
+            ),
+            models.CheckConstraint(
+                condition=(
+                    Q(
+                        body_state="RECEIVED",
+                        http_status__isnull=False,
+                        body_absence_reason="",
+                    )
+                    & ~Q(body_sha256="")
+                    & ~Q(media_type="")
+                    & ~Q(encoding="")
+                    | Q(
+                        body_state="NOT_RECEIVED",
+                        received_row_count=0,
+                        body_byte_length=0,
+                        body_sha256="",
+                        media_type="",
+                        encoding="",
+                    )
+                    & ~Q(body_absence_reason="")
+                ),
+                name="grocery_page_body_fields_valid",
+            ),
+        ]
+
+    def __str__(self) -> str:
+        return f"{self.fetch_attempt_id}:{self.request_ordinal}/{self.page_number}"
+
+    def save(self, *args: Any, **kwargs: Any) -> None:
+        if not self._state.adding:
+            raise ValidationError("Page receipts are immutable.")
+        self.full_clean()
+        super().save(*args, **kwargs)
+
+    def clean(self) -> None:
+        super().clean()
+        if self._state.adding and self.fetch_attempt.state != FetchAttempt.State.STARTED:
+            raise ValidationError("Page receipts can only be added to a started fetch attempt.")
diff --git a/grocery/tests/test_acquisition_models.py b/grocery/tests/test_acquisition_models.py
new file mode 100644
index 0000000..81e1c47
--- /dev/null
+++ b/grocery/tests/test_acquisition_models.py
@@ -0,0 +1,261 @@
+import uuid
+from datetime import timedelta
+from typing import Any
+
+from django.core.exceptions import ValidationError
+from django.db import IntegrityError, transaction
+from django.db.models.deletion import ProtectedError
+from django.test import TestCase
+from django.utils import timezone
+
+from grocery.models import FetchAttempt, PageReceipt, SourceConfiguration
+
+
+def create_source_configuration(**overrides: Any) -> SourceConfiguration:
+    values: dict[str, Any] = {
+        "source_owner_name": "한국농수산식품유통공사",
+        "dataset_id": "15156063",
+        "configuration_revision": str(uuid.uuid4()),
+        "interface_revision": "recent-price-v1",
+        "state": SourceConfiguration.State.ACTIVE,
+        "publication_mode": SourceConfiguration.PublicationMode.RECENT_COMPARISON,
+        "coverage_identity": "KAMIS_RETAIL_ALL_REGIONS_22_CITIES_V1",
+        "coverage_evidence_revision": "2026-08-30",
+        "endpoint_host": "apis.data.go.kr",
+        "endpoint_path": "/B552845/recent/price",
+        "authentication_mode": SourceConfiguration.AuthenticationMode.DATA_GO_KR_SERVICE_KEY,
+        "logical_secret_name": "KAMIS_API_KEY",
+        "provider_quota_limit": 10_000,
+        "provider_quota_period": SourceConfiguration.QuotaPeriod.UNSPECIFIED,
+        "request_timeout_seconds": 10,
+        "retry_policy": SourceConfiguration.RetryPolicy.BOUNDED_TRANSIENT_ONLY,
+        "max_retries": 2,
+        "max_requests_per_attempt": 12,
+        "max_pages_per_attempt": 10,
+        "max_page_bytes": 4 * 1024 * 1024,
+        "rights_evidence_locator": "https://www.data.go.kr/data/15156063/openapi.do",
+        "rights_evidence_sha256": "a" * 64,
+        "rights_confirmed_at": timezone.now(),
+    }
+    values.update(overrides)
+    return SourceConfiguration.objects.create(**values)
+
+
+def create_fetch_attempt(
+    source: SourceConfiguration,
+    **overrides: Any,
+) -> FetchAttempt:
+    values: dict[str, Any] = {
+        "source_configuration": source,
+        "acquisition_run_id": uuid.uuid4(),
+        "attempt_ordinal": 1,
+        "redacted_request_shape": (
+            "GET endpoint parameters=[pageNo,numOfRows,returnType,serviceKey:<redacted>]"
+        ),
+    }
+    values.update(overrides)
+    return FetchAttempt.objects.create(**values)
+
+
+def create_page_receipt(attempt: FetchAttempt, **overrides: Any) -> PageReceipt:
+    values: dict[str, Any] = {
+        "fetch_attempt": attempt,
+        "request_ordinal": 1,
+        "page_number": 1,
+        "http_status": 200,
+        "provider_result_code": "0",
+        "declared_total_count": 452,
+        "received_row_count": 100,
+        "body_state": PageReceipt.BodyState.RECEIVED,
+        "body_byte_length": 1024,
+        "body_sha256": "b" * 64,
+        "media_type": PageReceipt.MediaType.JSON,
+        "encoding": PageReceipt.Encoding.UTF_8,
+    }
+    values.update(overrides)
+    return PageReceipt.objects.create(**values)
+
+
+class SourceConfigurationTests(TestCase):
+    def test_valid_configuration_uses_uuid_and_only_logical_secret_reference(self) -> None:
+        source = create_source_configuration()
+
+        self.assertIsInstance(source.pk, uuid.UUID)
+        self.assertEqual(source.endpoint_scheme, "https")
+        self.assertEqual(source.endpoint_method, "GET")
+        self.assertEqual(source.raw_retention, "HASH_ONLY")
+        field_names = {field.name for field in source._meta.fields}
+        self.assertIn("logical_secret_name", field_names)
+        self.assertFalse({"secret", "credential", "api_key"} & field_names)
+
+    def test_endpoint_query_invalid_hash_and_zero_budget_are_rejected(self) -> None:
+        invalid = create_source_configuration()
+        invalid.endpoint_path = "/B552845/recent/price?serviceKey=forbidden"
+        with self.assertRaises(ValidationError):
+            invalid.save()
+
+        invalid.endpoint_path = "/B552845/recent/price"
+        invalid.rights_evidence_sha256 = "A" * 64
+        with self.assertRaises(ValidationError):
+            invalid.save()
+
+        invalid.rights_evidence_sha256 = "a" * 64
+        invalid.max_pages_per_attempt = 0
+        with self.assertRaises(ValidationError):
+            invalid.save()
+
+    def test_authenticated_configuration_requires_a_logical_secret_name(self) -> None:
+        with self.assertRaises(ValidationError):
+            create_source_configuration(logical_secret_name="")
+        with self.assertRaises(ValidationError):
+            create_source_configuration(logical_secret_name="not-a-logical-name")
+
+    def test_configuration_is_immutable_and_protected_after_first_fetch(self) -> None:
+        source = create_source_configuration()
+        create_fetch_attempt(source)
+        source.state = SourceConfiguration.State.PAUSED
+        source.state_changed_at = timezone.now()
+
+        with self.assertRaisesMessage(ValidationError, "immutable"):
+            source.save()
+        with self.assertRaises(ProtectedError):
+            source.delete()
+
+    def test_database_rejects_an_unlisted_state_when_model_validation_is_bypassed(self) -> None:
+        source = create_source_configuration()
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            SourceConfiguration.objects.filter(pk=source.pk).update(state="UNLISTED")
+
+
+class FetchAttemptTests(TestCase):
+    def test_attempt_ordinal_is_unique_per_acquisition_run(self) -> None:
+        source = create_source_configuration()
+        run_id = uuid.uuid4()
+        create_fetch_attempt(source, acquisition_run_id=run_id)
+
+        with self.assertRaises(ValidationError):
+            create_fetch_attempt(source, acquisition_run_id=run_id)
+        second = create_fetch_attempt(source, acquisition_run_id=run_id, attempt_ordinal=2)
+        self.assertEqual(second.attempt_ordinal, 2)
+
+    def test_terminal_states_require_completion_and_failure_fields(self) -> None:
+        source = create_source_configuration()
+        completed_at = timezone.now()
+
+        with self.assertRaises(ValidationError):
+            create_fetch_attempt(source, state=FetchAttempt.State.SUCCEEDED)
+        with self.assertRaises(ValidationError):
+            create_fetch_attempt(
+                source,
+                state=FetchAttempt.State.RETRYABLE_FAILED,
+                started_at=completed_at - timedelta(seconds=1),
+                completed_at=completed_at,
+            )
+
+        failed = create_fetch_attempt(
+            source,
+            state=FetchAttempt.State.RETRYABLE_FAILED,
+            started_at=completed_at - timedelta(seconds=1),
+            completed_at=completed_at,
+            failure_class=FetchAttempt.FailureClass.TIMEOUT,
+            failure_code="CLIENT_TIMEOUT",
+        )
+        self.assertEqual(failed.failure_class, FetchAttempt.FailureClass.TIMEOUT)
+
+    def test_completion_cannot_precede_start(self) -> None:
+        source = create_source_configuration()
+        started_at = timezone.now()
+
+        with self.assertRaises(ValidationError):
+            create_fetch_attempt(
+                source,
+                state=FetchAttempt.State.SUCCEEDED,
+                started_at=started_at,
+                completed_at=started_at - timedelta(seconds=1),
+            )
+
+    def test_request_shape_rejects_full_urls_and_query_strings(self) -> None:
+        source = create_source_configuration()
+        with self.assertRaises(ValidationError):
+            create_fetch_attempt(
+                source,
+                redacted_request_shape="GET https://apis.data.go.kr/path?serviceKey=<redacted>",
+            )
+
+
+class PageReceiptTests(TestCase):
+    def test_received_and_not_received_receipts_have_distinct_shapes(self) -> None:
+        source = create_source_configuration()
+        attempt = create_fetch_attempt(source)
+        received = create_page_receipt(attempt)
+        absent = create_page_receipt(
+            attempt,
+            request_ordinal=2,
+            page_number=2,
+            http_status=None,
+            provider_result_code="",
+            declared_total_count=None,
+            received_row_count=0,
+            body_state=PageReceipt.BodyState.NOT_RECEIVED,
+            body_byte_length=0,
+            body_sha256="",
+            body_absence_reason=PageReceipt.BodyAbsenceReason.TIMEOUT,
+            media_type="",
+            encoding="",
+        )
+
+        self.assertIsInstance(received.pk, uuid.UUID)
+        self.assertEqual(absent.body_absence_reason, PageReceipt.BodyAbsenceReason.TIMEOUT)
+
+    def test_page_and_request_ordinals_are_independently_unique_per_attempt(self) -> None:
+        source = create_source_configuration()
+        attempt = create_fetch_attempt(source)
+        create_page_receipt(attempt)
+
+        with self.assertRaises(ValidationError):
+            create_page_receipt(attempt, page_number=2)
+        with self.assertRaises(ValidationError):
+            create_page_receipt(attempt, request_ordinal=2)
+
+    def test_invalid_body_shape_and_uppercase_hash_are_rejected(self) -> None:
+        source = create_source_configuration()
+        attempt = create_fetch_attempt(source)
+
+        with self.assertRaises(ValidationError):
+            create_page_receipt(attempt, body_sha256="B" * 64)
+        with self.assertRaises(ValidationError):
+            create_page_receipt(
+                attempt,
+                body_state=PageReceipt.BodyState.NOT_RECEIVED,
+                body_absence_reason="",
+            )
+
+    def test_database_hash_constraint_and_receipt_immutability(self) -> None:
+        source = create_source_configuration()
+        attempt = create_fetch_attempt(source)
+        receipt = create_page_receipt(attempt)
+
+        with self.assertRaises(IntegrityError), transaction.atomic():
+            PageReceipt.objects.filter(pk=receipt.pk).update(body_sha256="B" * 64)
+        receipt.provider_result_code = "CHANGED"
+        with self.assertRaisesMessage(ValidationError, "immutable"):
+            receipt.save()
+
+    def test_receipt_requires_a_started_attempt_and_fk_is_protected(self) -> None:
+        source = create_source_configuration()
+        completed_at = timezone.now()
+        attempt = create_fetch_attempt(
+            source,
+            state=FetchAttempt.State.SUCCEEDED,
+            started_at=completed_at - timedelta(seconds=1),
+            completed_at=completed_at,
+        )
+
+        with self.assertRaises(ValidationError):
+            create_page_receipt(attempt)
+
+        started_attempt = create_fetch_attempt(source)
+        create_page_receipt(started_attempt)
+        with self.assertRaises(ProtectedError):
+            started_attempt.delete()


