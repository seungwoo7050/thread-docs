# 렌더링 CLI 옵션과 종료 코드 계약

## `chore(project): CXX17 실행 골격과 직접 빌드 구성`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..ba4054e
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,4 @@
+ray-scene-tracer
+*.o
+*.d
+/build*/
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..8ad26ec
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,18 @@
+CXX ?= c++
+CXXFLAGS ?= -std=c++17 -Wall -Wextra -Wpedantic -O2
+
+TARGET := ray-scene-tracer
+SRC := $(sort $(wildcard src/*.cpp))
+
+.PHONY: all clean test
+
+all: $(TARGET)
+
+$(TARGET): $(SRC)
+	$(CXX) $(CXXFLAGS) -o $@ $(SRC)
+
+test: $(TARGET)
+	./$(TARGET) --help >/dev/null
+
+clean:
+	rm -f $(TARGET)
diff --git a/src/main.cpp b/src/main.cpp
new file mode 100644
index 0000000..5ec802b
--- /dev/null
+++ b/src/main.cpp
@@ -0,0 +1,19 @@
+#include <iostream>
+#include <string>
+
+namespace {
+
+void print_usage(std::ostream& output) {
+    output << "usage: ./ray-scene-tracer <scene.rt> <output.ppm> [--checksum]\n";
+}
+
+}  // namespace
+
+int main(int argc, char** argv) {
+    if (argc == 2 && std::string(argv[1]) == "--help") {
+        print_usage(std::cout);
+        return 0;
+    }
+    print_usage(std::cerr);
+    return 2;
+}


## `feat(cli): 장면 렌더링 명령 연결`

diff --git a/Makefile b/Makefile
index a040b15..263b96c 100644
--- a/Makefile
+++ b/Makefile
@@ -13,7 +13,7 @@ $(TARGET): $(SRC)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -o $@ $(SRC)
 
 test: $(TARGET)
-	./$(TARGET) --help >/dev/null
+	./$(TARGET) scenes/basic.rt /tmp/ray-scene-tracer-check.ppm --checksum >/dev/null
 
 clean:
 	rm -f $(TARGET)
diff --git a/src/main.cpp b/src/main.cpp
index 5ec802b..2881566 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -1,19 +1,39 @@
+#include "ray.hpp"
+
+#include <exception>
 #include <iostream>
 #include <string>
 
 namespace {
 
-void print_usage(std::ostream& output) {
-    output << "usage: ./ray-scene-tracer <scene.rt> <output.ppm> [--checksum]\n";
+void print_usage() {
+    std::cerr << "usage: ./ray-scene-tracer <scene.rt> <output.ppm> [--checksum]\n";
 }
 
 }  // namespace
 
 int main(int argc, char** argv) {
-    if (argc == 2 && std::string(argv[1]) == "--help") {
-        print_usage(std::cout);
-        return 0;
+    if (argc != 3 && argc != 4) {
+        print_usage();
+        return 2;
+    }
+    const std::string scene_path = argv[1];
+    const std::string output_path = argv[2];
+    const bool print_checksum = argc == 4 && std::string(argv[3]) == "--checksum";
+    if (argc == 4 && !print_checksum) {
+        print_usage();
+        return 2;
+    }
+    try {
+        const ray::Scene scene = ray::loadScene(scene_path);
+        const ray::Image image = ray::renderScene(scene);
+        ray::writePpm(image, output_path);
+        if (print_checksum) {
+            std::cout << ray::checksumHex(image) << '\n';
+        }
+    } catch (const std::exception& error) {
+        std::cerr << "ray-scene-tracer: " << error.what() << '\n';
+        return 1;
     }
-    print_usage(std::cerr);
-    return 2;
+    return 0;
 }


## `refactor(cli): 위치 인자와 checksum option 모델 구성`

diff --git a/src/main.cpp b/src/main.cpp
index 2881566..c86f4a1 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -6,33 +6,64 @@
 
 namespace {
 
-void print_usage() {
-    std::cerr << "usage: ./ray-scene-tracer <scene.rt> <output.ppm> [--checksum]\n";
+struct CliOptions {
+    std::string scenePath;
+    std::string outputPath;
+    bool printChecksum = false;
+    ray::RenderSettings renderSettings;
+};
+
+void printUsage() {
+    std::cerr
+        << "usage: ray-scene-tracer <scene.rt> <output.ppm>"
+        << " [--checksum]\n";
+}
+
+bool parseCli(int argc, char** argv, CliOptions& options) {
+    if (argc < 3) {
+        return false;
+    }
+    options.scenePath = argv[1];
+    options.outputPath = argv[2];
+
+    bool seen_checksum = false;
+
+    for (int index = 3; index < argc; ++index) {
+        const std::string option = argv[index];
+        if (option == "--checksum") {
+            if (seen_checksum) {
+                return false;
+            }
+            seen_checksum = true;
+            options.printChecksum = true;
+            continue;
+        }
+        return false;
+    }
+    return true;
 }
 
 }  // namespace
 
 int main(int argc, char** argv) {
-    if (argc != 3 && argc != 4) {
-        print_usage();
-        return 2;
-    }
-    const std::string scene_path = argv[1];
-    const std::string output_path = argv[2];
-    const bool print_checksum = argc == 4 && std::string(argv[3]) == "--checksum";
-    if (argc == 4 && !print_checksum) {
-        print_usage();
+    CliOptions options;
+    if (!parseCli(argc, argv, options)) {
+        printUsage();
         return 2;
     }
+
     try {
-        const ray::Scene scene = ray::loadScene(scene_path);
-        const ray::Image image = ray::renderScene(scene);
-        ray::writePpm(image, output_path);
-        if (print_checksum) {
+        const ray::Scene scene =
+            ray::loadScene(options.scenePath);
+        const ray::Image image =
+            ray::renderScene(scene, options.renderSettings);
+        ray::writePpm(image, options.outputPath);
+        if (options.printChecksum) {
             std::cout << ray::checksumHex(image) << '\n';
         }
     } catch (const std::exception& error) {
-        std::cerr << "ray-scene-tracer: " << error.what() << '\n';
+        std::cerr << "ray-scene-tracer: "
+                  << error.what() << '\n';
         return 1;
     }
     return 0;


## `feat(cli): 가속 방식 선택 option 추가`

diff --git a/src/main.cpp b/src/main.cpp
index c86f4a1..f066e9f 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -16,7 +16,8 @@ struct CliOptions {
 void printUsage() {
     std::cerr
         << "usage: ray-scene-tracer <scene.rt> <output.ppm>"
-        << " [--checksum]\n";
+        << " [--checksum]"
+        << " [--accel linear|bvh]\n";
 }
 
 bool parseCli(int argc, char** argv, CliOptions& options) {
@@ -27,6 +28,7 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
     options.outputPath = argv[2];
 
     bool seen_checksum = false;
+    bool seen_accel = false;
 
     for (int index = 3; index < argc; ++index) {
         const std::string option = argv[index];
@@ -38,6 +40,24 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
             options.printChecksum = true;
             continue;
         }
+
+        if (option == "--accel") {
+            if (seen_accel || index + 1 >= argc) {
+                return false;
+            }
+            seen_accel = true;
+            const std::string value = argv[++index];
+            if (value == "linear") {
+                options.renderSettings.accelMode =
+                    ray::AccelMode::Linear;
+            } else if (value == "bvh") {
+                options.renderSettings.accelMode =
+                    ray::AccelMode::Bvh;
+            } else {
+                return false;
+            }
+            continue;
+        }
         return false;
     }
     return true;


## `feat(cli): 작업자 수 option 추가`

diff --git a/src/main.cpp b/src/main.cpp
index f066e9f..2c67e73 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -1,7 +1,10 @@
 #include "ray.hpp"
 
+#include <algorithm>
+#include <cctype>
 #include <exception>
 #include <iostream>
+#include <limits>
 #include <string>
 
 namespace {
@@ -17,7 +20,34 @@ void printUsage() {
     std::cerr
         << "usage: ray-scene-tracer <scene.rt> <output.ppm>"
         << " [--checksum]"
-        << " [--accel linear|bvh]\n";
+        << " [--accel linear|bvh]"
+        << " [--threads N|auto]\n";
+}
+
+bool parseUnsigned(const std::string& token,
+                   unsigned long long maximum,
+                   unsigned long long& value) {
+    if (token.empty() ||
+        !std::all_of(
+            token.begin(),
+            token.end(),
+            [](unsigned char character) {
+                return std::isdigit(character) != 0;
+            })) {
+        return false;
+    }
+    try {
+        std::size_t parsed = 0;
+        const unsigned long long candidate =
+            std::stoull(token, &parsed);
+        if (parsed != token.size() || candidate > maximum) {
+            return false;
+        }
+        value = candidate;
+        return true;
+    } catch (const std::exception&) {
+        return false;
+    }
 }
 
 bool parseCli(int argc, char** argv, CliOptions& options) {
@@ -29,6 +59,7 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
 
     bool seen_checksum = false;
     bool seen_accel = false;
+    bool seen_threads = false;
 
     for (int index = 3; index < argc; ++index) {
         const std::string option = argv[index];
@@ -58,6 +89,29 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
             }
             continue;
         }
+
+        if (option == "--threads") {
+            if (seen_threads || index + 1 >= argc) {
+                return false;
+            }
+            seen_threads = true;
+            const std::string value = argv[++index];
+            if (value == "auto") {
+                options.renderSettings.threadCount = 0;
+                continue;
+            }
+            unsigned long long parsed = 0;
+            if (!parseUnsigned(
+                    value,
+                    std::numeric_limits<unsigned int>::max(),
+                    parsed) ||
+                parsed == 0) {
+                return false;
+            }
+            options.renderSettings.threadCount =
+                static_cast<unsigned int>(parsed);
+            continue;
+        }
         return false;
     }
     return true;


## `feat(cli): 반사 깊이 option과 기본값 추가`

diff --git a/src/main.cpp b/src/main.cpp
index 2c67e73..40ff1eb 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -21,7 +21,8 @@ void printUsage() {
         << "usage: ray-scene-tracer <scene.rt> <output.ppm>"
         << " [--checksum]"
         << " [--accel linear|bvh]"
-        << " [--threads N|auto]\n";
+        << " [--threads N|auto]"
+        << " [--max-depth 0..32]\n";
 }
 
 bool parseUnsigned(const std::string& token,
@@ -60,6 +61,7 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
     bool seen_checksum = false;
     bool seen_accel = false;
     bool seen_threads = false;
+    bool seen_max_depth = false;
 
     for (int index = 3; index < argc; ++index) {
         const std::string option = argv[index];
@@ -112,6 +114,20 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
                 static_cast<unsigned int>(parsed);
             continue;
         }
+
+        if (option == "--max-depth") {
+            if (seen_max_depth || index + 1 >= argc) {
+                return false;
+            }
+            seen_max_depth = true;
+            unsigned long long parsed = 0;
+            if (!parseUnsigned(argv[++index], 32, parsed)) {
+                return false;
+            }
+            options.renderSettings.maxDepth =
+                static_cast<int>(parsed);
+            continue;
+        }
         return false;
     }
     return true;
diff --git a/src/renderer.cpp b/src/renderer.cpp
index 472076b..9669736 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -31,7 +31,7 @@ std::size_t pixelStorageSize(int width, int height) {
 
 RenderSettings::RenderSettings()
     : samplesPerPixel(1),
-      maxDepth(1),
+      maxDepth(4),
       tMin(kRayTMin),
       tMax(std::numeric_limits<double>::infinity()),
       accelMode(AccelMode::Bvh),


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
