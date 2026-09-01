## `ci(e24): run worker lifecycle checks after browser build`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index 30c748f..d1ef7cd 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -50,8 +50,6 @@ jobs:
       # capped fault/plan scenarios are not repeated by every push.
       - name: Outbound safety and resource limits
         run: npm run test:outbound
-      - name: Worker lifecycle and scheduler
-        run: npm run test:execution
       - name: Stop isolated PostgreSQL
         if: always()
         run: npm run db:down
@@ -73,6 +71,8 @@ jobs:
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
         run: npm run test:e2e
+      - name: Worker lifecycle and scheduler
+        run: npm run test:execution
       - name: Stop isolated PostgreSQL
         if: always()
         run: npm run db:down
