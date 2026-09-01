## `fix(test): isolate production gate runtime`

diff --git a/Makefile b/Makefile
index 5bd2906..aeaa091 100644
--- a/Makefile
+++ b/Makefile
@@ -33,7 +33,14 @@ type:
 	.venv/bin/mypy config grocery scripts manage.py
 
 test:
-	.venv/bin/pytest
+	# Keep the test runtime deterministic even when production-check is invoked with
+	# HTTPS redirect, HSTS, and Admin disabled. Production settings are validated by
+	# production-env-check and the final check --deploy gate below.
+	env DJANGO_DEBUG=1 ADMIN_ENABLED=1 QA_STATE_PREVIEWS_ENABLED=0 \
+		CONTROL_PLANE_OPERATIONS_ENABLED=0 DJANGO_SECURE_SSL_REDIRECT=0 \
+		DJANGO_SECURE_HSTS_SECONDS=0 DJANGO_SECURE_HSTS_INCLUDE_SUBDOMAINS=0 \
+		DJANGO_SECURE_HSTS_PRELOAD=0 DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1,testserver \
+		DEPLOY_VERSION=0000000 .venv/bin/pytest
 
 check: format-check lint type migration-check test
 	$(PYTHON) manage.py check


