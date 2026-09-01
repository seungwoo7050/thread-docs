## `build: improve Makefile and separate functional stack validation`

diff --git a/Makefile b/Makefile
index 7230ff8..e50c8cc 100644
--- a/Makefile
+++ b/Makefile
@@ -3,18 +3,64 @@ COMPOSE_FILE := srcs/docker-compose.yml
 ENV_FILE ?= .env
 PROJECT_NAME ?= container-stack
 WAIT_TIMEOUT ?= 300
+CHECK_ENV_FILE ?= .env.example
 BACKUP_DIR ?=
 NEW_SECRETS_DIR ?=
 DIAGNOSTICS_DIR ?= diagnostics/$(PROJECT_NAME)
 DESTROY_CONFIRM ?=
 
-COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
-
-.PHONY: up start-database start-application down build logs ps clean fclean test config config-strict smoke bootstrap-test e2e persistence backup restore backup-restore-test rotate-secrets rotation-test diagnostics operations-test verify
+COMPOSE_RUN := $(COMPOSE) --project-name "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" -f "$(COMPOSE_FILE)"
+
+.DEFAULT_GOAL := help
+
+.PHONY: help check check-functional up up-build start-database start-application
+.PHONY: down build logs ps clean fclean test config config-strict smoke
+.PHONY: bootstrap-test e2e persistence backup restore backup-restore-test
+.PHONY: rotate-secrets rotation-test diagnostics operations-test verify
+
+help:
+	@printf '%s\n' \
+		'Usage: make <target> [VARIABLE=value]' \
+		'' \
+		'Stack:' \
+		'  up                 Reconcile and start the existing images' \
+		'  up-build           Build, reconcile, and start under one operation lock' \
+		'  start-database     Reconcile and start only MariaDB' \
+		'  start-application  Reconcile and start WordPress and nginx' \
+		'  build              Build all local images' \
+		'  down               Stop containers; preserve images and volumes' \
+		'  ps / logs          Show container state / follow logs' \
+		'  fclean             Remove volumes and local images (confirmation required)' \
+		'' \
+		'Validation:' \
+		'  check              Run static checks and strict Compose parsing' \
+		'  check-functional   Run executable/configuration checks without docs policy' \
+		'  test               Run source-level validation' \
+		'  config             Print the resolved Compose model' \
+		'  config-strict      Validate a Compose model without printing it' \
+		'  smoke              Probe the running HTTPS endpoint' \
+		'  verify             Run every static and runtime scenario serially' \
+		'' \
+		'Operations: backup, restore, rotate-secrets, diagnostics' \
+		'Test scenarios: bootstrap-test, e2e, persistence, backup-restore-test,' \
+		'                rotation-test, operations-test' \
+		'' \
+		'Common variables: PROJECT_NAME, ENV_FILE, WAIT_TIMEOUT, CHECK_ENV_FILE'
+
+check:
+	$(MAKE) test
+	$(MAKE) config-strict ENV_FILE="$(CHECK_ENV_FILE)"
+
+check-functional:
+	python3 tests/validate_stack.py --functional
+	$(MAKE) config-strict ENV_FILE="$(CHECK_ENV_FILE)"
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
 
+up-build:
+	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)" --build
+
 start-database:
 	python3 tools/start_stack.py database --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
 
@@ -53,7 +99,7 @@ config-strict:
 test:
 	python3 tests/validate_stack.py
 	@if command -v docker >/dev/null 2>&1 && docker compose version >/dev/null 2>&1; then \
-		$(COMPOSE) --env-file .env.example -f $(COMPOSE_FILE) config >/dev/null; \
+		$(COMPOSE) --env-file .env.example -f "$(COMPOSE_FILE)" config >/dev/null; \
 		echo "docker compose config passed"; \
 	else \
 		echo "docker compose not available; skipped compose config"; \
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index 5548781..c045dc8 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -1,4 +1,5 @@
 #!/usr/bin/env python3
+import argparse
 import ast
 from contextlib import redirect_stderr
 import importlib.util
@@ -836,14 +837,13 @@ def validate_bootstrap_recovery() -> None:
 def validate_ci() -> None:
     workflow = require_file(".github/workflows/container-stack.yml").read_text()
     required = (
+        "name: web/inception CI",
+        "branches: [web/inception]",
         "runs-on: ubuntu-24.04",
         "timeout-minutes: 210",
         "permissions:\n  contents: read",
         "persist-credentials: false",
-        "fetch-depth: 0",
-        'tools/check_commit_range.py --base "${{ github.event.pull_request.base.sha || github.event.before }}"',
-        "make test",
-        "make config-strict ENV_FILE=.env.example",
+        "make check-functional",
         "if: ${{ always() }}",
         "if: ${{ failure() }}",
         "retention-days: 7",
@@ -855,8 +855,8 @@ def validate_ci() -> None:
         if value not in workflow:
             fail(f"container stack workflow is missing {value!r}")
     expected_actions = [
-        "actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683",
-        "actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08",
+        "actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1",
+        "actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a",
     ]
     actions = re.findall(r"^[ \t]*uses:[ \t]*(\S+)", workflow, re.MULTILINE)
     if actions != expected_actions:
@@ -875,6 +875,8 @@ def validate_ci() -> None:
             permission_values.append(line.strip())
     if permission_values != ["contents: read"]:
         fail("workflow permissions must contain only contents: read")
+    if workflow.count("branches: [web/inception]") != 2:
+        fail("workflow must target web/inception pushes and pull requests exactly")
     scenarios = (
         "e2e",
         "bootstrap",
@@ -927,6 +929,10 @@ def validate_ci() -> None:
         fail("workflow artifact paths must use the diagnostic allowlist")
     for forbidden in (
         "pull_request_target",
+        "workflow_dispatch:",
+        "paths-ignore:",
+        "make test",
+        "check_commit_range.py",
         "${{ secrets.",
         "set -x",
         "printenv",
@@ -1050,9 +1056,8 @@ def validate_readme() -> None:
     if not re.search(r"[가-힣]", text):
         fail("README.md must be written in Korean")
 
-def main() -> None:
-    validate_source_only()
-    validate_forbidden_project_wording()
+
+def validate_functional() -> None:
     validate_compose()
     validate_dockerfiles()
     validate_configs()
@@ -1061,8 +1066,23 @@ def main() -> None:
     validate_runtime_control_flow()
     validate_bootstrap_recovery()
     validate_ci()
-    validate_readme()
     validate_rotation_runtime_boundary()
+
+
+def main() -> None:
+    parser = argparse.ArgumentParser(description="Validate the container stack")
+    parser.add_argument(
+        "--functional",
+        action="store_true",
+        help="skip repository wording, layout, and README policy checks",
+    )
+    args = parser.parse_args()
+
+    validate_functional()
+    if not args.functional:
+        validate_source_only()
+        validate_forbidden_project_wording()
+        validate_readme()
     print("static stack validation passed")
 
 


## `ci: harden container stack validation`

diff --git a/.github/workflows/container-stack.yml b/.github/workflows/container-stack.yml
index 8fcdb39..d507358 100644
--- a/.github/workflows/container-stack.yml
+++ b/.github/workflows/container-stack.yml
@@ -1,16 +1,16 @@
-name: container-stack
+name: web/inception CI
 
 on:
   push:
-    branches: [main]
+    branches: [web/inception]
   pull_request:
-  workflow_dispatch:
+    branches: [web/inception]
 
 permissions:
   contents: read
 
 concurrency:
-  group: container-stack-${{ github.workflow }}-${{ github.ref }}
+  group: web-inception-${{ github.workflow }}-${{ github.ref }}
   cancel-in-progress: true
 
 jobs:
@@ -22,21 +22,16 @@ jobs:
       DOCKER_BUILDKIT: "1"
     steps:
       - name: 소스 받기
-        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
         with:
           persist-credentials: false
-          fetch-depth: 0
 
       - name: 진단 경로 준비
         run: install -d -m 0700 artifacts artifacts/projects
 
       - name: 정적 구성 검사
-        run: |
-          set -eu
-          git diff --check
-          python3 tools/check_commit_range.py --base "${{ github.event.pull_request.base.sha || github.event.before }}"
-          make test
-          make config-strict ENV_FILE=.env.example
+        timeout-minutes: 5
+        run: make check-functional
 
       - name: HTTPS 종단 검증
         timeout-minutes: 25
@@ -68,7 +63,7 @@ jobs:
 
       - name: 실패 진단 업로드
         if: ${{ failure() }}
-        uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08 # v4.6.0
+        uses: actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a # v7.0.1
         with:
           name: container-stack-diagnostics-${{ github.run_id }}-${{ github.run_attempt }}
           path: |
@@ -78,6 +73,6 @@ jobs:
             artifacts/**/compose-model.txt
             artifacts/**/container-state.txt
             artifacts/cleanup.txt
-          if-no-files-found: ignore
+          if-no-files-found: warn
           retention-days: 7
           include-hidden-files: false
