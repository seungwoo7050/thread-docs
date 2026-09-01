## `fix(entry): require durable ingestion sessions`

diff --git a/entry_requirements/ingestion.py b/entry_requirements/ingestion.py
index fab359a..5d7b78e 100644
--- a/entry_requirements/ingestion.py
+++ b/entry_requirements/ingestion.py
@@ -136,7 +136,11 @@ class _ClosedPersistenceFailure(Exception):
 
 
 def _try_acquire_entry_ingestion_lock() -> bool:
-    if connection.vendor != "postgresql" or connection.in_atomic_block:
+    if (
+        connection.vendor != "postgresql"
+        or connection.in_atomic_block
+        or connection.get_autocommit() is not True
+    ):
         raise RuntimeError("entry ingestion requires an autocommit PostgreSQL session")
     with connection.cursor() as cursor:
         cursor.execute(
diff --git a/entry_requirements/tests/test_ingestion.py b/entry_requirements/tests/test_ingestion.py
index a17d7d0..5e80834 100644
--- a/entry_requirements/tests/test_ingestion.py
+++ b/entry_requirements/tests/test_ingestion.py
@@ -359,6 +359,22 @@ class EntryIngestionTests(TransactionTestCase):
             EntryIngestionOutcome(EntryIngestionCode.PERSISTENCE_FAILED, 0),
         )
 
+    def test_manual_transaction_is_rejected_before_receipt_or_transport(self):
+        connection.set_autocommit(False)
+        try:
+            outcome = ingest_entry_snapshot(
+                transport=lambda **kwargs: self.fail("transport must not run")
+            )
+        finally:
+            connection.rollback()
+            connection.set_autocommit(True)
+
+        self.assertEqual(
+            outcome,
+            EntryIngestionOutcome(EntryIngestionCode.PERSISTENCE_FAILED, 0),
+        )
+        self.assertFalse(FetchAttempt.objects.exists())
+
     def test_postgresql_session_lock_blocks_a_second_ingestion_session(self):
         contender = self.open_second_session()
         try:


