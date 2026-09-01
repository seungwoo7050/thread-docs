## `test(render): 작업자 수에 따른 함수 결과 동치 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index ce88edd..2922394 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -57,6 +57,10 @@ if(BUILD_TESTING)
     )
     add_test(NAME material_regression COMMAND ray-material-tests)
 
+    add_executable(ray-render-tests tests/render_tests.cpp)
+    target_link_libraries(ray-render-tests PRIVATE raycore)
+    add_test(NAME render_determinism COMMAND ray-render-tests)
+
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/render_tests.cpp b/tests/render_tests.cpp
new file mode 100644
index 0000000..1e74ea4
--- /dev/null
+++ b/tests/render_tests.cpp
@@ -0,0 +1,100 @@
+#include "ray.hpp"
+
+#include <iostream>
+#include <stdexcept>
+#include <string>
+
+namespace {
+
+void require(bool condition, const std::string& message) {
+    if (!condition) {
+        throw std::runtime_error(message);
+    }
+}
+
+ray::Scene makeScene() {
+    return ray::parser::parseSceneText(
+        "R 96 54\n"
+        "A 0.12 255,255,255\n"
+        "C 0,1,-4 0,-0.08,1 55\n"
+        "L -3,6,-1 0.9 255,244,220\n"
+        "L 4,3,5 0.4 180,210,255\n"
+        "sp -0.9,0,3 1.5 220,70,45 diffuse\n"
+        "sp 0.9,0,3.5 1.4 210,220,235 metal\n"
+        "pl 0,-0.8,0 0,1,0 150,165,180 diffuse\n"
+        "cy 2,-0.1,6 0.2,1,0.1 0.8 2 70,190,120 diffuse\n",
+        "determinism.rt");
+}
+
+struct Result {
+    ray::Image image;
+    ray::RenderStats stats;
+    std::string checksum;
+};
+
+Result render(const ray::Scene& scene,
+              ray::AccelMode mode,
+              unsigned int threads) {
+    ray::RenderSettings settings;
+    settings.accelMode = mode;
+    settings.threadCount = threads;
+    settings.maxDepth = 4;
+    Result result;
+    result.image = ray::renderScene(scene, settings, &result.stats);
+    result.checksum = ray::checksumHex(result.image);
+    return result;
+}
+
+void requireSameWork(const Result& left,
+                     const Result& right,
+                     const std::string& label) {
+    require(left.stats.primaryRays == right.stats.primaryRays,
+            label + " primary rays");
+    require(left.stats.secondaryRays == right.stats.secondaryRays,
+            label + " secondary rays");
+    require(left.stats.shadowRays == right.stats.shadowRays,
+            label + " shadow rays");
+    require(left.stats.primitiveTests == right.stats.primitiveTests,
+            label + " primitive tests");
+    require(left.stats.aabbTests == right.stats.aabbTests,
+            label + " AABB tests");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        const ray::Scene scene = makeScene();
+        const Result linear_one =
+            render(scene, ray::AccelMode::Linear, 1);
+        const Result linear_four =
+            render(scene, ray::AccelMode::Linear, 4);
+        const Result bvh_one =
+            render(scene, ray::AccelMode::Bvh, 1);
+        const Result bvh_four =
+            render(scene, ray::AccelMode::Bvh, 4);
+
+        require(linear_one.image.pixels == linear_four.image.pixels &&
+                    linear_one.image.pixels == bvh_one.image.pixels &&
+                    linear_one.image.pixels == bvh_four.image.pixels,
+                "all render modes produce identical pixels");
+        require(linear_one.checksum == linear_four.checksum &&
+                    linear_one.checksum == bvh_one.checksum &&
+                    linear_one.checksum == bvh_four.checksum,
+                "all render modes produce identical checksums");
+        requireSameWork(linear_one,
+                        linear_four,
+                        "linear thread count");
+        requireSameWork(bvh_one,
+                        bvh_four,
+                        "BVH thread count");
+        require(linear_one.stats.primaryRays == 96u * 54u,
+                "primary ray count");
+    } catch (const std::exception& error) {
+        std::cerr << "render determinism failed: "
+                  << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "render determinism checks passed\n";
+    return 0;
+}


## `test(cli): 렌더링 옵션과 오류 종료 계약 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 2922394..6644348 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -61,6 +61,14 @@ if(BUILD_TESTING)
     target_link_libraries(ray-render-tests PRIVATE raycore)
     add_test(NAME render_determinism COMMAND ray-render-tests)
 
+    add_test(
+        NAME cli_contract
+        COMMAND bash
+                ${CMAKE_CURRENT_SOURCE_DIR}/tests/cli_contract.sh
+                $<TARGET_FILE:ray-scene-tracer>
+                ${CMAKE_CURRENT_SOURCE_DIR}
+    )
+
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/cli_contract.sh b/tests/cli_contract.sh
new file mode 100644
index 0000000..bdcac40
--- /dev/null
+++ b/tests/cli_contract.sh
@@ -0,0 +1,71 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+BIN=$1
+ROOT=$2
+TMP=$(mktemp -d)
+trap 'rm -rf "$TMP"' EXIT
+
+expect_usage_error() {
+    set +e
+    "$BIN" "$@" >"$TMP/stdout" 2>"$TMP/stderr"
+    STATUS=$?
+    set -e
+    if [[ $STATUS -ne 2 ]]; then
+        echo "expected exit 2 for: $*" >&2
+        exit 1
+    fi
+    if ! grep -q '^usage: ray-scene-tracer ' "$TMP/stderr"; then
+        echo "expected usage for: $*" >&2
+        exit 1
+    fi
+}
+
+SCENE="$ROOT/scenes/basic.rt"
+OUT="$TMP/output.ppm"
+
+expect_usage_error
+expect_usage_error "$SCENE"
+expect_usage_error "$SCENE" "$OUT" --unknown
+expect_usage_error "$SCENE" "$OUT" --checksum --checksum
+expect_usage_error "$SCENE" "$OUT" --accel
+expect_usage_error "$SCENE" "$OUT" --accel tree
+expect_usage_error "$SCENE" "$OUT" --accel bvh --accel linear
+expect_usage_error "$SCENE" "$OUT" --threads
+expect_usage_error "$SCENE" "$OUT" --threads 0
+expect_usage_error "$SCENE" "$OUT" --threads -1
+expect_usage_error "$SCENE" "$OUT" --threads many
+expect_usage_error "$SCENE" "$OUT" --threads 18446744073709551616
+expect_usage_error "$SCENE" "$OUT" --threads 1 --threads auto
+expect_usage_error "$SCENE" "$OUT" --max-depth
+expect_usage_error "$SCENE" "$OUT" --max-depth -1
+expect_usage_error "$SCENE" "$OUT" --max-depth 33
+expect_usage_error "$SCENE" "$OUT" --max-depth 4x
+expect_usage_error "$SCENE" "$OUT" --max-depth 4 --max-depth 5
+
+CHECKSUM=$(
+    "$BIN" "$SCENE" "$OUT" \
+        --threads 1 \
+        --max-depth 0 \
+        --accel linear \
+        --checksum
+)
+if [[ ! "$CHECKSUM" =~ ^[0-9a-f]{16}$ || ! -s "$OUT" ]]; then
+    echo "valid options did not produce an image and checksum" >&2
+    exit 1
+fi
+
+AUTO_OUT="$TMP/auto.ppm"
+AUTO_CHECKSUM=$(
+    "$BIN" "$SCENE" "$AUTO_OUT" \
+        --threads auto \
+        --max-depth 32 \
+        --accel bvh \
+        --checksum
+)
+if [[ "$AUTO_CHECKSUM" != "$CHECKSUM" || ! -s "$AUTO_OUT" ]]; then
+    echo "boundary options changed the diffuse render" >&2
+    exit 1
+fi
+
+echo "CLI contract checks passed"


## `test(render): smoke 검사의 fixture와 실행 경로 정리`

diff --git a/tests/render_smoke.sh b/tests/render_smoke.sh
index c944071..888029e 100755
--- a/tests/render_smoke.sh
+++ b/tests/render_smoke.sh
@@ -10,22 +10,21 @@ if [[ $# -eq 0 ]]; then
     make -C "$ROOT" >/dev/null
 fi
 
-BAD_SCENE="$TMP/bad.rt"
+BAD_SCENE="$ROOT/scenes/invalid.rt"
 BAD_OUT="$TMP/bad.ppm"
-cat >"$BAD_SCENE" <<'SCENE'
-R 16 8
-C 0,0,3 0,0,-1 60
-not_a_shape 1 2 3
-SCENE
 
 if "$BIN" "$BAD_SCENE" "$BAD_OUT" >"$TMP/bad.stdout" 2>"$TMP/bad.stderr"; then
-    echo "expected parser failure for unknown directive" >&2
+    echo "expected parser failure for invalid ambient ratio" >&2
     exit 1
 fi
 if [[ -s "$BAD_OUT" ]]; then
     echo "parser failure should not leave a rendered image" >&2
     exit 1
 fi
+if ! grep -q 'invalid.rt:3:' "$TMP/bad.stderr"; then
+    echo "expected invalid.rt line 3 in parser error" >&2
+    exit 1
+fi
 
 SCENE_FILE="$TMP/smoke.rt"
 cat >"$SCENE_FILE" <<'SCENE'
@@ -44,17 +43,19 @@ PPM_TWO="$TMP/two.ppm"
 CHECK_ONE=$("$BIN" "$SCENE_FILE" "$PPM_ONE" --checksum)
 CHECK_TWO=$("$BIN" "$SCENE_FILE" "$PPM_TWO" --checksum)
 
-mapfile -t HEADER < <(head -n 3 "$PPM_ONE")
-if [[ "${HEADER[0]}" != "P3" ]]; then
-    echo "expected P3 PPM magic, got ${HEADER[0]}" >&2
+MAGIC=$(sed -n '1p' "$PPM_ONE")
+DIMENSIONS=$(sed -n '2p' "$PPM_ONE")
+MAX_CHANNEL=$(sed -n '3p' "$PPM_ONE")
+if [[ "$MAGIC" != "P3" ]]; then
+    echo "expected P3 PPM magic, got $MAGIC" >&2
     exit 1
 fi
-if [[ "${HEADER[1]}" != "64 32" ]]; then
-    echo "expected PPM dimensions 64 32, got ${HEADER[1]}" >&2
+if [[ "$DIMENSIONS" != "64 32" ]]; then
+    echo "expected PPM dimensions 64 32, got $DIMENSIONS" >&2
     exit 1
 fi
-if [[ "${HEADER[2]}" != "255" ]]; then
-    echo "expected PPM max channel 255, got ${HEADER[2]}" >&2
+if [[ "$MAX_CHANNEL" != "255" ]]; then
+    echo "expected PPM max channel 255, got $MAX_CHANNEL" >&2
     exit 1
 fi
 


## `test(render): 실행 모드별 PPM byte 결정성 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 6644348..68dc718 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -69,6 +69,13 @@ if(BUILD_TESTING)
                 ${CMAKE_CURRENT_SOURCE_DIR}
     )
 
+    add_test(
+        NAME render_output_determinism
+        COMMAND bash
+                ${CMAKE_CURRENT_SOURCE_DIR}/tests/render_determinism.sh
+                $<TARGET_FILE:ray-scene-tracer>
+    )
+
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/render_determinism.sh b/tests/render_determinism.sh
new file mode 100644
index 0000000..8546397
--- /dev/null
+++ b/tests/render_determinism.sh
@@ -0,0 +1,51 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+BIN=$1
+TMP=$(mktemp -d)
+trap 'rm -rf "$TMP"' EXIT
+
+SCENE="$TMP/determinism.rt"
+cat >"$SCENE" <<'SCENE'
+R 96 54
+A 0.12 255,255,255
+C 0,1,-4 0,-0.08,1 55
+L -3,6,-1 0.9 255,244,220
+L 4,3,5 0.4 180,210,255
+sp -0.9,0,3 1.5 220,70,45 diffuse
+sp 0.9,0,3.5 1.4 210,220,235 metal
+pl 0,-0.8,0 0,1,0 150,165,180 diffuse
+cy 2,-0.1,6 0.2,1,0.1 0.8 2 70,190,120 diffuse
+SCENE
+
+render() {
+    NAME=$1
+    ACCEL=$2
+    THREADS=$3
+    "$BIN" "$SCENE" "$TMP/$NAME.ppm" \
+        --accel "$ACCEL" \
+        --threads "$THREADS" \
+        --max-depth 4 \
+        --checksum
+}
+
+LINEAR_ONE=$(render linear-one linear 1)
+LINEAR_FOUR=$(render linear-four linear 4)
+BVH_ONE=$(render bvh-one bvh 1)
+BVH_FOUR=$(render bvh-four bvh 4)
+
+for CHECKSUM in "$LINEAR_FOUR" "$BVH_ONE" "$BVH_FOUR"; do
+    if [[ "$CHECKSUM" != "$LINEAR_ONE" ]]; then
+        echo "render modes produced different checksums" >&2
+        exit 1
+    fi
+done
+
+for IMAGE in linear-four bvh-one bvh-four; do
+    if ! cmp -s "$TMP/linear-one.ppm" "$TMP/$IMAGE.ppm"; then
+        echo "render modes produced different PPM bytes: $IMAGE" >&2
+        exit 1
+    fi
+done
+
+echo "render output determinism checks passed"


## `build(sanitizers): 메모리와 정의되지 않은 동작 검사 구성`

diff --git a/.gitignore b/.gitignore
index 62cc2c8..ba4054e 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,4 +1,4 @@
 ray-scene-tracer
 *.o
 *.d
-/build/
+/build*/
diff --git a/CMakeLists.txt b/CMakeLists.txt
index 68dc718..2194507 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -3,6 +3,7 @@ cmake_minimum_required(VERSION 3.16)
 project(ray_scene_tracer LANGUAGES CXX)
 
 find_package(Threads REQUIRED)
+option(RAY_ENABLE_SANITIZERS "Enable AddressSanitizer and UBSan" OFF)
 
 set(CMAKE_CXX_STANDARD 17)
 set(CMAKE_CXX_STANDARD_REQUIRED ON)
@@ -28,6 +29,21 @@ else()
     target_compile_options(raycore PRIVATE -Wall -Wextra -Wpedantic)
 endif()
 
+if(RAY_ENABLE_SANITIZERS)
+    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
+        target_compile_options(
+            raycore
+            PUBLIC -fsanitize=address,undefined -fno-omit-frame-pointer
+        )
+        target_link_options(
+            raycore
+            PUBLIC -fsanitize=address,undefined
+        )
+    else()
+        message(FATAL_ERROR "Sanitizers require Clang or GCC")
+    endif()
+endif()
+
 add_executable(ray-scene-tracer src/main.cpp)
 target_link_libraries(ray-scene-tracer PRIVATE raycore)
 


## `ci: 플랫폼별 빌드와 회귀 검사 자동화`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..b108d32
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,48 @@
+name: C++ 검증
+
+on:
+  push:
+  pull_request:
+
+jobs:
+  release:
+    name: Release · ${{ matrix.os }}
+    runs-on: ${{ matrix.os }}
+    strategy:
+      fail-fast: false
+      matrix:
+        os:
+          - ubuntu-latest
+          - macos-latest
+
+    steps:
+      - uses: actions/checkout@v4
+      - name: 구성
+        run: >
+          cmake -S . -B build
+          -DCMAKE_BUILD_TYPE=Release
+          -DBUILD_TESTING=ON
+      - name: 빌드
+        run: cmake --build build --parallel
+      - name: 회귀 검사
+        run: ctest --test-dir build --output-on-failure
+
+  sanitizers:
+    name: ASan · UBSan
+    runs-on: ubuntu-latest
+
+    steps:
+      - uses: actions/checkout@v4
+      - name: 구성
+        run: >
+          cmake -S . -B build
+          -DCMAKE_BUILD_TYPE=Debug
+          -DBUILD_TESTING=ON
+          -DRAY_ENABLE_SANITIZERS=ON
+      - name: 빌드
+        run: cmake --build build --parallel
+      - name: 메모리·정의되지 않은 동작 검사
+        env:
+          ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
+          UBSAN_OPTIONS: print_stacktrace=1:halt_on_error=1
+        run: ctest --test-dir build --output-on-failure


## `test(output): 출력 실패의 대상 보존과 정리 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 2194507..d48bcb4 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -77,6 +77,10 @@ if(BUILD_TESTING)
     target_link_libraries(ray-render-tests PRIVATE raycore)
     add_test(NAME render_determinism COMMAND ray-render-tests)
 
+    add_executable(ray-output-tests tests/output_tests.cpp)
+    target_link_libraries(ray-output-tests PRIVATE raycore)
+    add_test(NAME output_failure_regression COMMAND ray-output-tests)
+
     add_test(
         NAME cli_contract
         COMMAND bash
diff --git a/tests/output_tests.cpp b/tests/output_tests.cpp
new file mode 100644
index 0000000..1a7c7b7
--- /dev/null
+++ b/tests/output_tests.cpp
@@ -0,0 +1,155 @@
+#include "ray.hpp"
+
+#include <chrono>
+#include <filesystem>
+#include <fstream>
+#include <iostream>
+#include <ostream>
+#include <sstream>
+#include <stdexcept>
+#include <streambuf>
+#include <string>
+
+namespace {
+
+void require(bool condition, const std::string& message) {
+    if (!condition) {
+        throw std::runtime_error(message);
+    }
+}
+
+class FailingBuffer : public std::streambuf {
+protected:
+    std::streamsize xsputn(const char*, std::streamsize) override {
+        return 0;
+    }
+
+    int_type overflow(int_type) override {
+        return traits_type::eof();
+    }
+};
+
+class TestDirectory {
+public:
+    TestDirectory() {
+        const unsigned long long token =
+            static_cast<unsigned long long>(
+                std::chrono::steady_clock::now()
+                    .time_since_epoch()
+                    .count());
+        path_ = std::filesystem::temp_directory_path() /
+                ("ray-output-regression-" + std::to_string(token));
+        if (!std::filesystem::create_directory(path_)) {
+            throw std::runtime_error("cannot create output test directory");
+        }
+    }
+
+    ~TestDirectory() {
+        std::error_code ignored;
+        std::filesystem::remove_all(path_, ignored);
+    }
+
+    const std::filesystem::path& path() const {
+        return path_;
+    }
+
+private:
+    std::filesystem::path path_;
+};
+
+std::string readFile(const std::filesystem::path& path) {
+    std::ifstream input(path);
+    std::ostringstream contents;
+    contents << input.rdbuf();
+    return contents.str();
+}
+
+void writeFile(const std::filesystem::path& path,
+               const std::string& contents) {
+    std::ofstream output(path);
+    output << contents;
+    if (!output) {
+        throw std::runtime_error("cannot prepare output test file");
+    }
+}
+
+void requireNoTemporaryFiles(const std::filesystem::path& directory,
+                             const std::string& prefix) {
+    for (const std::filesystem::directory_entry& entry :
+         std::filesystem::directory_iterator(directory)) {
+        const std::string name = entry.path().filename().string();
+        require(name.rfind(prefix + ".tmp.", 0) != 0,
+                "temporary output file was not cleaned up");
+    }
+}
+
+ray::Image sampleImage() {
+    ray::Image image(2, 1);
+    image.pixels = {255, 0, 16, 0, 127, 255};
+    return image;
+}
+
+void testStreamFailure() {
+    FailingBuffer buffer;
+    std::ostream output(&buffer);
+    bool rejected = false;
+    try {
+        ray::writePpm(sampleImage(), output);
+    } catch (const std::runtime_error&) {
+        rejected = true;
+    }
+    require(rejected, "PPM serializer reports stream failure");
+}
+
+void testAtomicReplacement() {
+    TestDirectory directory;
+    const std::filesystem::path target = directory.path() / "image.ppm";
+    writeFile(target, "old image\n");
+
+    ray::writePpm(sampleImage(), target.string());
+
+    require(readFile(target) ==
+                "P3\n2 1\n255\n255 0 16\n0 127 255\n",
+            "PPM writer replaces an existing file");
+    requireNoTemporaryFiles(directory.path(), "image.ppm");
+}
+
+void testFailedReplacementPreservesDestination() {
+    TestDirectory directory;
+    const std::filesystem::path target =
+        directory.path() / "existing-destination";
+    std::filesystem::create_directory(target);
+    const std::filesystem::path sentinel = target / "keep.txt";
+    writeFile(sentinel, "preserve me\n");
+
+    bool rejected = false;
+    try {
+        ray::writePpm(sampleImage(), target.string());
+    } catch (const std::runtime_error&) {
+        rejected = true;
+    }
+
+    require(rejected, "PPM writer reports replacement failure");
+    require(std::filesystem::is_directory(target),
+            "failed replacement preserves destination type");
+    require(readFile(sentinel) == "preserve me\n",
+            "failed replacement preserves destination contents");
+    requireNoTemporaryFiles(directory.path(),
+                            "existing-destination");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        testStreamFailure();
+        testAtomicReplacement();
+        testFailedReplacementPreservesDestination();
+    } catch (const std::exception& error) {
+        std::cerr << "output failure regression failed: "
+                  << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "output failure regression checks passed\n";
+    return 0;
+}


## `build: expose deterministic verification targets`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index d48bcb4..0cbe348 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -102,4 +102,19 @@ if(BUILD_TESTING)
                 ${CMAKE_CURRENT_SOURCE_DIR}/tests/render_smoke.sh
                 $<TARGET_FILE:ray-scene-tracer>
     )
+
+    set_tests_properties(
+        core_regression
+        accel_regression
+        material_regression
+        render_determinism
+        output_failure_regression
+        cli_contract
+        PROPERTIES TIMEOUT 60
+    )
+    set_tests_properties(
+        render_output_determinism
+        render_smoke
+        PROPERTIES TIMEOUT 120
+    )
 endif()
diff --git a/Makefile b/Makefile
index 6fbc625..a13384b 100644
--- a/Makefile
+++ b/Makefile
@@ -1,17 +1,65 @@
 BUILD_DIR ?= build
+SANITIZER_BUILD_DIR ?= build/sanitize
 CMAKE ?= cmake
+CTEST ?= ctest
+CXX := c++
+BUILD_TYPE ?= Release
+RAY_ENABLE_SANITIZERS ?= OFF
+CMAKE_CONFIGURE_ARGS ?=
+CMAKE_BUILD_ARGS ?= --parallel
+CTEST_ARGS ?= --output-on-failure --no-tests=error --timeout 120
 
-.PHONY: all clean test
+.PHONY: all build check ci clean configure guard-build-dir help sanitize test
 
-all:
-	$(CMAKE) -S . -B $(BUILD_DIR) -DCMAKE_BUILD_TYPE=Release
-	$(CMAKE) --build $(BUILD_DIR)
-	$(CMAKE) -E create_symlink $(BUILD_DIR)/ray-scene-tracer ray-scene-tracer
+all: build
+	$(CMAKE) -E rm -f ray-scene-tracer
+	$(CMAKE) -E create_symlink "$(BUILD_DIR)/ray-scene-tracer" ray-scene-tracer
 
-test:
-	$(CMAKE) -S . -B $(BUILD_DIR) -DCMAKE_BUILD_TYPE=Release
-	$(CMAKE) --build $(BUILD_DIR)
-	ctest --test-dir $(BUILD_DIR) --output-on-failure
+configure:
+	$(CMAKE) -S . -B "$(BUILD_DIR)" \
+		"-DCMAKE_CXX_COMPILER=$(CXX)" \
+		"-DCMAKE_BUILD_TYPE=$(BUILD_TYPE)" \
+		-DBUILD_TESTING=ON \
+		"-DRAY_ENABLE_SANITIZERS=$(RAY_ENABLE_SANITIZERS)" \
+		$(CMAKE_CONFIGURE_ARGS)
 
-clean:
-	$(CMAKE) -E rm -rf $(BUILD_DIR) ray-scene-tracer
+build: configure
+	$(CMAKE) --build "$(BUILD_DIR)" $(CMAKE_BUILD_ARGS)
+
+test: build
+	$(CTEST) --test-dir "$(BUILD_DIR)" $(CTEST_ARGS)
+
+check: test
+
+ci:
+	$(MAKE) clean
+	$(MAKE) check
+
+sanitize:
+	$(MAKE) BUILD_DIR="$(SANITIZER_BUILD_DIR)" BUILD_TYPE=Debug \
+		RAY_ENABLE_SANITIZERS=ON ci
+
+help:
+	@printf '%s\n' \
+		'Targets:' \
+		'  all       Configure and build a release binary.' \
+		'  test      Build and run the complete CTest suite.' \
+		'  check     Run the complete functional test gate.' \
+		'  ci        Clean, rebuild, and run the functional gate.' \
+		'  sanitize  Run the gate with AddressSanitizer and UBSan.' \
+		'  clean     Remove generated outputs from BUILD_DIR.' \
+		'' \
+		'Overrides: CXX, CMAKE, CTEST, BUILD_DIR, BUILD_TYPE,' \
+		'           CMAKE_CONFIGURE_ARGS, CMAKE_BUILD_ARGS, CTEST_ARGS'
+
+guard-build-dir:
+	@build_dir="$(abspath $(BUILD_DIR))"; \
+	case "$$build_dir" in \
+		"$(CURDIR)"/*) ;; \
+		*) printf 'Refusing to remove unsafe BUILD_DIR: %s\n' \
+			"$$build_dir" >&2; exit 2 ;; \
+	esac
+
+clean: guard-build-dir
+	$(CMAKE) -E rm -rf "$(BUILD_DIR)"
+	$(CMAKE) -E rm -f ray-scene-tracer


