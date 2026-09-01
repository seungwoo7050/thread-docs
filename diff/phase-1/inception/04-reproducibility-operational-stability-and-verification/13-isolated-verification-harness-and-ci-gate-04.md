## `ci(stack): 커밋 범위 공백 검사 도구 추가`

diff --git a/tools/check_commit_range.py b/tools/check_commit_range.py
new file mode 100755
index 0000000..eb0aaa1
--- /dev/null
+++ b/tools/check_commit_range.py
@@ -0,0 +1,69 @@
+#!/usr/bin/env python3
+"""현재 실행을 유발한 커밋 범위의 공백 오류를 검사합니다."""
+
+from __future__ import annotations
+
+import argparse
+import re
+import subprocess
+import sys
+
+
+OBJECT_ID = re.compile(r"^(?:[0-9a-f]{40}|[0-9a-f]{64})$")
+
+
+def git(*arguments: str, capture: bool = False) -> subprocess.CompletedProcess[str]:
+    return subprocess.run(
+        ["git", *arguments],
+        check=True,
+        text=True,
+        capture_output=capture,
+    )
+
+
+def fallback_base() -> str:
+    return git("rev-parse", "HEAD^", capture=True).stdout.strip()
+
+
+def select_base(requested: str) -> str:
+    if not requested or set(requested) == {"0"}:
+        return fallback_base()
+    if not OBJECT_ID.fullmatch(requested):
+        raise ValueError("기준 커밋 해시 형식이 올바르지 않습니다")
+
+    available = subprocess.run(
+        ["git", "cat-file", "-e", f"{requested}^{{commit}}"],
+        text=True,
+        capture_output=True,
+    )
+    if available.returncode == 0:
+        return requested
+
+    print(
+        f"요청한 기준 커밋을 찾을 수 없어 HEAD^를 검사합니다: {requested}",
+        file=sys.stderr,
+    )
+    return fallback_base()
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(description="CI 커밋 범위 공백 검사")
+    parser.add_argument("--base", default="")
+    return parser.parse_args()
+
+
+def main() -> int:
+    args = parse_arguments()
+    try:
+        base = select_base(args.base)
+        head = git("rev-parse", "HEAD", capture=True).stdout.strip()
+        git("diff", "--check", base, head)
+    except (OSError, ValueError, subprocess.CalledProcessError) as error:
+        print(f"커밋 범위 검사 실패: {error}", file=sys.stderr)
+        return 1
+    print(f"커밋 범위 공백 검사 통과: {base}..{head}")
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `ci(stack): 정적·런타임·복구 검증 자동화`

diff --git a/.github/workflows/container-stack.yml b/.github/workflows/container-stack.yml
new file mode 100644
index 0000000..8fcdb39
--- /dev/null
+++ b/.github/workflows/container-stack.yml
@@ -0,0 +1,83 @@
+name: container-stack
+
+on:
+  push:
+    branches: [main]
+  pull_request:
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+concurrency:
+  group: container-stack-${{ github.workflow }}-${{ github.ref }}
+  cancel-in-progress: true
+
+jobs:
+  verify:
+    runs-on: ubuntu-24.04
+    timeout-minutes: 210
+    env:
+      COMPOSE_ANSI: never
+      DOCKER_BUILDKIT: "1"
+    steps:
+      - name: 소스 받기
+        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
+        with:
+          persist-credentials: false
+          fetch-depth: 0
+
+      - name: 진단 경로 준비
+        run: install -d -m 0700 artifacts artifacts/projects
+
+      - name: 정적 구성 검사
+        run: |
+          set -eu
+          git diff --check
+          python3 tools/check_commit_range.py --base "${{ github.event.pull_request.base.sha || github.event.before }}"
+          make test
+          make config-strict ENV_FILE=.env.example
+
+      - name: HTTPS 종단 검증
+        timeout-minutes: 25
+        run: python3 tests/runtime_stack.py e2e --diagnostics-dir artifacts/e2e --project-record-dir artifacts/projects
+
+      - name: 초기화 강제 종료 복구 검증
+        timeout-minutes: 40
+        run: python3 tests/runtime_stack.py bootstrap --diagnostics-dir artifacts/bootstrap --project-record-dir artifacts/projects
+
+      - name: 영속성 검증
+        timeout-minutes: 25
+        run: python3 tests/runtime_stack.py persistence --diagnostics-dir artifacts/persistence --project-record-dir artifacts/projects
+
+      - name: 백업과 복원 검증
+        timeout-minutes: 30
+        run: python3 tests/runtime_stack.py backup-restore --diagnostics-dir artifacts/backup-restore --project-record-dir artifacts/projects
+
+      - name: 자격증명 회전 검증
+        timeout-minutes: 30
+        run: python3 tests/runtime_stack.py rotation --diagnostics-dir artifacts/rotation --project-record-dir artifacts/projects
+
+      - name: 운영 설정 검증
+        timeout-minutes: 25
+        run: python3 tests/runtime_stack.py operations --diagnostics-dir artifacts/operations --project-record-dir artifacts/projects
+
+      - name: 격리 자원 정리 확인
+        if: ${{ always() }}
+        run: python3 tools/cleanup_test_resources.py --project-record-dir artifacts/projects --report artifacts/cleanup.txt
+
+      - name: 실패 진단 업로드
+        if: ${{ failure() }}
+        uses: actions/upload-artifact@65c4c4a1ddee5b72f698fdd19549f0f0fb45cf08 # v4.6.0
+        with:
+          name: container-stack-diagnostics-${{ github.run_id }}-${{ github.run_attempt }}
+          path: |
+            artifacts/**/versions.txt
+            artifacts/**/compose-ps.txt
+            artifacts/**/compose-logs.txt
+            artifacts/**/compose-model.txt
+            artifacts/**/container-state.txt
+            artifacts/cleanup.txt
+          if-no-files-found: ignore
+          retention-days: 7
+          include-hidden-files: false


