# 검증 가능한 커밋 히스토리와 릴리스 경계

## `docs(project): establish admin API ownership`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..3d0bc0c
--- /dev/null
+++ b/README.md
@@ -0,0 +1,4 @@
+# Sportsbook Admin API
+
+Operator-facing control-plane API for the sportsbook archive project.
+


## `build(history): enforce archive commit policy`

diff --git a/.github/scripts/verify-history.sh b/.github/scripts/verify-history.sh
new file mode 100644
index 0000000..a1fe612
--- /dev/null
+++ b/.github/scripts/verify-history.sh
@@ -0,0 +1,97 @@
+#!/usr/bin/env bash
+set -euo pipefail
+cd "$(git rev-parse --show-toplevel)"
+head_commit=$(git rev-parse HEAD)
+root_commit=$(git rev-list --max-parents=0 HEAD)
+subject_pattern='^(feat|fix|test|refactor|perf|build|docs|chore|style|ci)\([a-z0-9-]+\): .+'
+forbidden_subject='fixup|squash|devlog|changelog|provenance|reconstruct'
+forbidden_path='(^|/)(devlog|changelog|provenance|reconstruction)(/|\.|$)|(^|/)load-test/results/'
+failed=0
+
+report() {
+  printf '%s\n' "$1" >&2
+  failed=1
+}
+
+only_paths() {
+  local pattern=$1 path
+  while read -r path; do
+    [[ $path =~ $pattern ]] || return 1
+  done
+}
+
+while read -r commit; do
+  subject=$(git show -s --format=%s "$commit")
+  body=$(git show -s --format=%b "$commit")
+  paths=$(git diff-tree --root --no-commit-id --name-only -r "$commit")
+  short=${commit:0:12}
+  has_main=false
+  has_test=false
+  main_files=0
+
+  while read -r path; do
+    if [[ $path == src/main/* ]]; then has_main=true; ((main_files += 1)); fi
+    [[ $path == src/test/* ]] && has_test=true
+  done <<<"$paths"
+
+  [[ $subject =~ $subject_pattern ]] || report "$short has a non-conventional subject: $subject"
+  [[ -z $body ]] || report "$short has a non-empty commit body"
+  if grep -Eiq "$forbidden_subject" <<<"$subject" || grep -Eiq "$forbidden_path" <<<"$paths"; then
+    report "$short contains forbidden reconstruction material"
+  fi
+  [[ $has_main == false || $has_test == false ]] || report "$short mixes production and test files"
+  if [[ $subject == test\(* && $has_main == true ]]; then
+    report "$short labels production code as a test"
+  elif [[ $subject != test\(* && $has_test == true ]]; then
+    report "$short includes tests in a non-test commit"
+  fi
+  ((main_files <= 2)) || report "$short changes more than two production files"
+
+  churn=$(git show --numstat --format= "$commit" |
+    awk '$1 ~ /^[0-9]+$/ && $2 ~ /^[0-9]+$/ {n += $1 + $2} END {print n + 0}')
+  if ((churn > 100)); then
+    exception=false
+    if [[ $subject == "build(maven): establish Java 17 baseline" && $paths == pom.xml ]]; then
+      exception=true
+    elif [[ $subject =~ ^build\(wrapper\): ]] && only_paths '^(mvnw|mvnw.cmd|\.mvn/wrapper/)' <<<"$paths"; then
+      exception=true
+    elif [[ $subject =~ ^build\(flyway\): ]] && [[ $(wc -l <<<"$paths") -eq 1 ]] && [[ $paths == *.sql ]]; then
+      exception=true
+    elif [[ $subject =~ ^feat\(audit\): ]] && [[ $(wc -l <<<"$paths") -eq 1 ]] && [[ $paths == *.avsc ]]; then
+      exception=true
+    elif [[ $commit == "$head_commit" && $subject == "docs(project): document admin API contracts" && $paths == README.md ]]; then
+      exception=true
+    fi
+    [[ $exception == true ]] || report "$short exceeds the 100-line review gate: $churn lines"
+  fi
+
+  if [[ $commit == "$root_commit" ]]; then
+    [[ $subject == "docs(project): establish admin API ownership" && $paths == README.md ]] ||
+      report "$short is not the required README-only root"
+  elif [[ $subject == docs\(* ]]; then
+    [[ $commit == "$head_commit" && $subject == "docs(project): document admin API contracts" ]] ||
+      report "$short is an intermediate documentation commit"
+  fi
+
+  if grep -Eq '^(src/main/|pom\.xml$|config/|mvnw$|mvnw\.cmd$|\.mvn/wrapper/|\.github/(scripts|workflows)/)' <<<"$paths" \
+    && [[ $subject != test\(* && $subject != docs\(* && $subject != "chore(release): release admin API 1.0.0" ]]; then
+    next=$(git rev-list --ancestry-path "$commit..HEAD" --reverse | head -1)
+    next_subject=$([[ -n $next ]] && git show -s --format=%s "$next" || true)
+    [[ $next_subject == test\(* ]] || report "$short is not followed by its test commit"
+  fi
+done < <(git rev-list --reverse HEAD)
+if [[ -n $(git rev-list --min-parents=2 --max-count=1 HEAD) ]]; then
+  report "archive history contains a merge commit"
+fi
+if git ls-tree -r --name-only HEAD | grep -Eiq "$forbidden_path"; then
+  report "final tree contains forbidden reconstruction material"
+fi
+head_subject=$(git show -s --format=%s HEAD)
+if [[ $head_subject == "docs(project): document admin API contracts" ]]; then
+  [[ $(git show -s --format=%s HEAD^) == "chore(release): release admin API 1.0.0" ]] ||
+    report "release commit is not immediately before final documentation"
+elif git log --format=%s | grep -q '^chore(release):'; then
+  report "release commit exists without final documentation"
+fi
+
+exit "$failed"


## `test(history): provide guard repository fixture`

diff --git a/src/test/java/com/sportsbook/admin/ops/HistoryGuardFixture.java b/src/test/java/com/sportsbook/admin/ops/HistoryGuardFixture.java
new file mode 100644
index 0000000..f77b392
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/HistoryGuardFixture.java
@@ -0,0 +1,56 @@
+package com.sportsbook.admin.ops;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.io.IOException;
+import java.nio.charset.StandardCharsets;
+import java.nio.file.Files;
+import java.nio.file.Path;
+import java.util.ArrayList;
+import java.util.Arrays;
+import java.util.List;
+import org.junit.jupiter.api.io.TempDir;
+
+abstract class HistoryGuardFixture {
+
+  private static final Path GUARD = Path.of(".github/scripts/verify-history.sh").toAbsolutePath();
+
+  @TempDir Path repository;
+
+  void initialize() throws Exception {
+    assertThat(run("git", "init", "-q").code()).isZero();
+    assertThat(run("git", "config", "user.name", "Admin CI").code()).isZero();
+    assertThat(run("git", "config", "user.email", "admin@example.invalid").code()).isZero();
+    write("README.md", "# Admin API\n");
+    commit("docs(project): establish admin API ownership", "README.md");
+  }
+
+  void write(String path, String content) throws IOException {
+    Path target = repository.resolve(path);
+    Files.createDirectories(target.getParent());
+    Files.writeString(target, content);
+  }
+
+  void commit(String subject, String... paths) throws Exception {
+    List<String> add = new ArrayList<>(List.of("git", "add"));
+    add.addAll(Arrays.asList(paths));
+    assertThat(run(add.toArray(String[]::new)).code()).isZero();
+    assertThat(run("git", "commit", "-q", "-m", subject).code()).isZero();
+  }
+
+  Result guard() throws Exception {
+    return run("bash", GUARD.toString());
+  }
+
+  Result run(String... command) throws IOException, InterruptedException {
+    Process process =
+        new ProcessBuilder(command)
+            .directory(repository.toFile())
+            .redirectErrorStream(true)
+            .start();
+    String output = new String(process.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
+    return new Result(process.waitFor(), output);
+  }
+
+  record Result(int code, String output) {}
+}


## `test(history): reject archive policy violations`

diff --git a/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java b/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java
new file mode 100644
index 0000000..94ea967
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java
@@ -0,0 +1,80 @@
+package com.sportsbook.admin.ops;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class AdminHistoryGuardPolicyTest extends HistoryGuardFixture {
+
+  @Test
+  void acceptsTheRequiredReadmeOnlyRoot() throws Exception {
+    initialize();
+
+    assertThat(guard().code()).isZero();
+  }
+
+  @Test
+  void rejectsNonConventionalSubjectsAndBodies() throws Exception {
+    initialize();
+    write("marker.txt", "marker\n");
+    assertThat(run("git", "add", "marker.txt").code()).isZero();
+    assertThat(run("git", "commit", "-q", "-m", "misc change", "-m", "body").code()).isZero();
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("non-conventional subject", "non-empty commit body");
+  }
+
+  @Test
+  void rejectsReconstructionMaterial() throws Exception {
+    initialize();
+    write("devlog.txt", "reconstruction provenance\n");
+    commit("docs(project): add reconstruction notes", "devlog.txt");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("forbidden reconstruction material");
+  }
+
+  @Test
+  void rejectsProductionAndTestMixing() throws Exception {
+    initialize();
+    write("src/main/java/Feature.java", "class Feature {}\n");
+    write("src/test/java/FeatureTest.java", "class FeatureTest {}\n");
+    commit(
+        "feat(core): mix feature and test",
+        "src/main/java/Feature.java",
+        "src/test/java/FeatureTest.java");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("mixes production and test files");
+  }
+
+  @Test
+  void rejectsOversizedUnapprovedCommits() throws Exception {
+    initialize();
+    write("marker.txt", "large\n".repeat(101));
+    commit("chore(project): add oversized marker", "marker.txt");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("100-line review gate");
+  }
+
+  @Test
+  void requiresAnAdjacentTestAfterProduction() throws Exception {
+    initialize();
+    write("src/main/java/Feature.java", "class Feature {}\n");
+    commit("feat(core): add feature", "src/main/java/Feature.java");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("not followed by its test commit");
+  }
+}


## `build(history): constrain release documentation`

diff --git a/.github/scripts/verify-history.sh b/.github/scripts/verify-history.sh
index a1fe612..ce7f60e 100644
--- a/.github/scripts/verify-history.sh
+++ b/.github/scripts/verify-history.sh
@@ -3,6 +3,11 @@ set -euo pipefail
 cd "$(git rev-parse --show-toplevel)"
 head_commit=$(git rev-parse HEAD)
 root_commit=$(git rev-list --max-parents=0 HEAD)
+head_subject=$(git show -s --format=%s HEAD)
+expected_release=""
+if [[ $head_subject == "docs(project): document admin API contracts" ]]; then
+  expected_release=$(git rev-parse HEAD^)
+fi
 subject_pattern='^(feat|fix|test|refactor|perf|build|docs|chore|style|ci)\([a-z0-9-]+\): .+'
 forbidden_subject='fixup|squash|devlog|changelog|provenance|reconstruct'
 forbidden_path='(^|/)(devlog|changelog|provenance|reconstruction)(/|\.|$)|(^|/)load-test/results/'
@@ -69,9 +74,13 @@ while read -r commit; do
     [[ $subject == "docs(project): establish admin API ownership" && $paths == README.md ]] ||
       report "$short is not the required README-only root"
   elif [[ $subject == docs\(* ]]; then
-    [[ $commit == "$head_commit" && $subject == "docs(project): document admin API contracts" ]] ||
+    [[ $commit == "$head_commit" && $subject == "docs(project): document admin API contracts" && $paths == README.md ]] ||
       report "$short is an intermediate documentation commit"
   fi
+  if [[ $subject == chore\(release\):* ]]; then
+    [[ $commit == "$expected_release" && $subject == "chore(release): release admin API 1.0.0" && $paths == pom.xml ]] ||
+      report "$short is not the single penultimate release commit"
+  fi
 
   if grep -Eq '^(src/main/|pom\.xml$|config/|mvnw$|mvnw\.cmd$|\.mvn/wrapper/|\.github/(scripts|workflows)/)' <<<"$paths" \
     && [[ $subject != test\(* && $subject != docs\(* && $subject != "chore(release): release admin API 1.0.0" ]]; then
@@ -86,7 +95,6 @@ fi
 if git ls-tree -r --name-only HEAD | grep -Eiq "$forbidden_path"; then
   report "final tree contains forbidden reconstruction material"
 fi
-head_subject=$(git show -s --format=%s HEAD)
 if [[ $head_subject == "docs(project): document admin API contracts" ]]; then
   [[ $(git show -s --format=%s HEAD^) == "chore(release): release admin API 1.0.0" ]] ||
     report "release commit is not immediately before final documentation"


## `test(history): enforce final documentation boundary`

diff --git a/src/test/java/com/sportsbook/admin/ops/AdminDocumentationHistoryTest.java b/src/test/java/com/sportsbook/admin/ops/AdminDocumentationHistoryTest.java
new file mode 100644
index 0000000..91eb64a
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/AdminDocumentationHistoryTest.java
@@ -0,0 +1,72 @@
+package com.sportsbook.admin.ops;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import org.junit.jupiter.api.Test;
+
+class AdminDocumentationHistoryTest extends HistoryGuardFixture {
+
+  @Test
+  void permitsOneLargeFinalReadmeAfterRelease() throws Exception {
+    initialize();
+    release();
+    write("README.md", "admin API contract\n".repeat(120));
+    commit("docs(project): document admin API contracts", "README.md");
+
+    assertThat(guard().code()).isZero();
+  }
+
+  @Test
+  void rejectsFinalDocumentationWithoutRelease() throws Exception {
+    initialize();
+    write("README.md", "admin API contract\n".repeat(120));
+    commit("docs(project): document admin API contracts", "README.md");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("release commit is not immediately before");
+  }
+
+  @Test
+  void rejectsAdditionalFinalDocumentationPaths() throws Exception {
+    initialize();
+    release();
+    write("README.md", "admin API contract\n".repeat(120));
+    write("notes.md", "extra documentation\n");
+    commit("docs(project): document admin API contracts", "README.md", "notes.md");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("intermediate documentation commit");
+  }
+
+  @Test
+  void rejectsReleaseWithoutFinalDocumentation() throws Exception {
+    initialize();
+    release();
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("release commit exists without final documentation");
+  }
+
+  @Test
+  void rejectsIntermediateDocumentation() throws Exception {
+    initialize();
+    write("notes.md", "intermediate\n");
+    commit("docs(project): describe progress", "notes.md");
+
+    Result result = guard();
+
+    assertThat(result.code()).isNotZero();
+    assertThat(result.output()).contains("intermediate documentation commit");
+  }
+
+  private void release() throws Exception {
+    write("pom.xml", "<project><version>1.0.0</version></project>\n");
+    commit("chore(release): release admin API 1.0.0", "pom.xml");
+  }
+}


## `ci(archive): verify fixed admin release inputs`

diff --git a/.github/workflows/admin-ci.yml b/.github/workflows/admin-ci.yml
new file mode 100644
index 0000000..aff4fbb
--- /dev/null
+++ b/.github/workflows/admin-ci.yml
@@ -0,0 +1,47 @@
+name: admin API archive verify
+
+on:
+  push:
+    branches: [admin-api]
+  pull_request:
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+jobs:
+  verify:
+    runs-on: ubuntu-latest
+    timeout-minutes: 40
+    steps:
+      - name: Checkout exact admin revision
+        uses: actions/checkout@v4
+        with:
+          ref: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.sha || github.sha }}
+          path: admin-api
+          fetch-depth: 0
+      - name: Checkout fixed shared protocol
+        uses: actions/checkout@v4
+        with:
+          ref: f9de6bc1e533761ab4bb1454d8d4ab8175cdf001
+          path: shared-protocol
+      - name: Use Java 17
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: "17"
+      - name: Install fixed shared protocol
+        working-directory: shared-protocol
+        run: >-
+          ./mvnw -B
+          -Dmaven.repo.local=${{ runner.temp }}/admin-m2
+          -DskipTests install
+      - name: Verify archive history
+        working-directory: admin-api
+        run: bash .github/scripts/verify-history.sh
+      - name: Verify admin API once
+        working-directory: admin-api
+        run: >-
+          ./mvnw -B
+          -Dmaven.repo.local=${{ runner.temp }}/admin-m2
+          clean verify


## `test(ci): verify archive workflow semantics`

diff --git a/src/test/java/com/sportsbook/admin/ops/AdminCiWorkflowTest.java b/src/test/java/com/sportsbook/admin/ops/AdminCiWorkflowTest.java
new file mode 100644
index 0000000..b1f8b47
--- /dev/null
+++ b/src/test/java/com/sportsbook/admin/ops/AdminCiWorkflowTest.java
@@ -0,0 +1,41 @@
+package com.sportsbook.admin.ops;
+
+import static org.assertj.core.api.Assertions.assertThat;
+
+import java.nio.file.Files;
+import java.nio.file.Path;
+import org.junit.jupiter.api.Test;
+
+class AdminCiWorkflowTest {
+
+  @Test
+  void pinsSharedAndRunsOneFreshJava17Verification() throws Exception {
+    String workflow = Files.readString(Path.of(".github/workflows/admin-ci.yml"));
+
+    assertThat(workflow)
+        .contains(
+            "fetch-depth: 0",
+            "github.event.pull_request.head.sha",
+            "distribution: temurin",
+            "java-version: \"17\"",
+            "ref: f9de6bc1e533761ab4bb1454d8d4ab8175cdf001",
+            "working-directory: shared-protocol",
+            "working-directory: admin-api",
+            "-Dmaven.repo.local=${{ runner.temp }}/admin-m2",
+            "-DskipTests install",
+            "bash .github/scripts/verify-history.sh",
+            "clean verify")
+        .containsOnlyOnce("clean verify")
+        .doesNotContain(
+            "ref: shared-protocol",
+            "sportsbook-shared-protocol",
+            "sportsbook-v2.0.0",
+            "java-version: \"21\"",
+            "clean package",
+            "mvn test");
+    assertThat(workflow.indexOf("-DskipTests install"))
+        .isLessThan(workflow.indexOf("clean verify"));
+    assertThat(workflow.indexOf("bash .github/scripts/verify-history.sh"))
+        .isLessThan(workflow.indexOf("clean verify"));
+  }
+}


## `build(history): traverse the complete commit chain`

diff --git a/.github/scripts/verify-history.sh b/.github/scripts/verify-history.sh
index ce7f60e..1a8b3eb 100644
--- a/.github/scripts/verify-history.sh
+++ b/.github/scripts/verify-history.sh
@@ -84,7 +84,7 @@ while read -r commit; do
 
   if grep -Eq '^(src/main/|pom\.xml$|config/|mvnw$|mvnw\.cmd$|\.mvn/wrapper/|\.github/(scripts|workflows)/)' <<<"$paths" \
     && [[ $subject != test\(* && $subject != docs\(* && $subject != "chore(release): release admin API 1.0.0" ]]; then
-    next=$(git rev-list --ancestry-path "$commit..HEAD" --reverse | head -1)
+    next=$(git rev-list --ancestry-path "$commit..HEAD" | tail -1)
     next_subject=$([[ -n $next ]] && git show -s --format=%s "$next" || true)
     [[ $next_subject == test\(* ]] || report "$short is not followed by its test commit"
   fi


## `test(history): verify long history traversal`

diff --git a/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java b/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java
index 94ea967..d11671e 100644
--- a/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java
+++ b/src/test/java/com/sportsbook/admin/ops/AdminHistoryGuardPolicyTest.java
@@ -6,6 +6,8 @@ import org.junit.jupiter.api.Test;
 
 class AdminHistoryGuardPolicyTest extends HistoryGuardFixture {
 
+  private static final int LONG_HISTORY_LENGTH = 240;
+
   @Test
   void acceptsTheRequiredReadmeOnlyRoot() throws Exception {
     initialize();
@@ -77,4 +79,19 @@ class AdminHistoryGuardPolicyTest extends HistoryGuardFixture {
     assertThat(result.code()).isNotZero();
     assertThat(result.output()).contains("not followed by its test commit");
   }
+
+  @Test
+  void traversesLongHistoriesWithoutPipeTermination() throws Exception {
+    initialize();
+    write("src/main/java/Feature.java", "class Feature {}\n");
+    commit("feat(core): add feature", "src/main/java/Feature.java");
+    write("src/test/java/FeatureTest.java", "class FeatureTest {}\n");
+    commit("test(core): verify feature", "src/test/java/FeatureTest.java");
+    for (int index = 0; index < LONG_HISTORY_LENGTH; index++) {
+      write("marker.txt", index + "\n");
+      commit("chore(project): extend history " + index, "marker.txt");
+    }
+
+    assertThat(guard().code()).isZero();
+  }
 }


