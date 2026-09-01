## `ci: scope verification triggers to the Spring track`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 5cae676..03acbe5 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -1,5 +1,9 @@
 name: Phase-1 verification
-on: [push, pull_request]
+on:
+  push:
+    branches: [track/industry-spring]
+  pull_request:
+    branches: [track/industry-spring]
 permissions:
   contents: read
 jobs:


## `ci: match the pinned Temurin build metadata`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
index 03acbe5..8847970 100644
--- a/.github/workflows/ci.yml
+++ b/.github/workflows/ci.yml
@@ -22,7 +22,7 @@ jobs:
       - uses: actions/setup-java@v4.7.1
         with:
           distribution: temurin
-          java-version: '21.0.7+6'
+          java-version: '21.0.7+6.0.LTS'
       - uses: actions/setup-node@v4.4.0
         with:
           node-version-file: .node-version
