## `test(qa): isolate fixture database gate`

diff --git a/grocery/tests/test_vnext_browser_fixture.py b/grocery/tests/test_vnext_browser_fixture.py
index d62e548..b158d72 100644
--- a/grocery/tests/test_vnext_browser_fixture.py
+++ b/grocery/tests/test_vnext_browser_fixture.py
@@ -1,7 +1,6 @@
 from unittest.mock import patch
 
 import pytest
-from django.db import connection
 from django.test import override_settings
 
 from grocery.historical_activation_models import HistoricalRetailPublicationChannel
@@ -17,9 +16,13 @@ def test_vnext_browser_fixture_uses_real_sealed_activation_without_source_calls(
     transactional_db: None,
 ) -> None:
     with (
-        patch.dict(
-            connection.settings_dict,
-            {"HOST": "127.0.0.1", "NAME": "grocery_vnext_browser_test"},
+        patch(
+            "grocery.tests.vnext_browser_fixture._database_configuration",
+            return_value={
+                "ENGINE": "django.db.backends.postgresql",
+                "HOST": "127.0.0.1",
+                "NAME": "grocery_vnext_browser_test",
+            },
         ),
         patch(
             "grocery.source.client.KamisHttpClient.fetch_recent_prices",
@@ -54,7 +57,14 @@ def test_vnext_browser_fixture_is_denied_outside_disposable_qa() -> None:
 @override_settings(DEBUG=True, ADMIN_ENABLED=False, QA_STATE_PREVIEWS_ENABLED=True)
 def test_vnext_browser_fixture_rejects_non_disposable_database_identity() -> None:
     with (
-        patch.dict(connection.settings_dict, {"HOST": "127.0.0.1", "NAME": "grocery"}),
+        patch(
+            "grocery.tests.vnext_browser_fixture._database_configuration",
+            return_value={
+                "ENGINE": "django.db.backends.postgresql",
+                "HOST": "127.0.0.1",
+                "NAME": "grocery",
+            },
+        ),
         pytest.raises(RuntimeError, match="database_denied"),
     ):
         build_vnext_browser_fixture()
@@ -65,9 +75,13 @@ def test_vnext_browser_fixture_rejects_non_disposable_database_identity() -> Non
 def test_vnext_browser_fixture_rejects_existing_source_or_domain_rows() -> None:
     create_source_configuration()
     with (
-        patch.dict(
-            connection.settings_dict,
-            {"HOST": "127.0.0.1", "NAME": "grocery_vnext_browser_test"},
+        patch(
+            "grocery.tests.vnext_browser_fixture._database_configuration",
+            return_value={
+                "ENGINE": "django.db.backends.postgresql",
+                "HOST": "127.0.0.1",
+                "NAME": "grocery_vnext_browser_test",
+            },
         ),
         pytest.raises(RuntimeError, match="database_not_empty"),
     ):
diff --git a/grocery/tests/vnext_browser_fixture.py b/grocery/tests/vnext_browser_fixture.py
index e0cd9e2..a2516ba 100644
--- a/grocery/tests/vnext_browser_fixture.py
+++ b/grocery/tests/vnext_browser_fixture.py
@@ -11,7 +11,7 @@ from decimal import Decimal
 from django.conf import settings
 from django.contrib.auth import get_user_model
 from django.contrib.auth.models import Permission
-from django.db import connection, transaction
+from django.db import transaction
 
 from grocery.historical_activation_models import (
     HistoricalRetailPublicationActivation,
@@ -58,6 +58,10 @@ class VnextBrowserFixture:
     monthly_fact_count: int
 
 
+def _database_configuration() -> dict[str, object]:
+    return dict(settings.DATABASES["default"])
+
+
 def _require_disposable_qa() -> None:
     if (
         settings.DEBUG is not True
@@ -65,7 +69,7 @@ def _require_disposable_qa() -> None:
         or getattr(settings, "QA_STATE_PREVIEWS_ENABLED", None) is not True
     ):
         raise RuntimeError("qa_fixture_environment_denied")
-    database = connection.settings_dict
+    database = _database_configuration()
     name = database.get("NAME")
     host = database.get("HOST")
     if (


