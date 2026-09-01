# 결정적 검증 게이트와 크로스 플랫폼 CI

## `docs(readme): 프로젝트 목표와 초기 개발 규약 정의`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..fbc4194
--- /dev/null
+++ b/README.md
@@ -0,0 +1,22 @@
+# ray-scene-tracer
+
+`ray-scene-tracer`는 `.rt` 장면을 읽어 P3 PPM 이미지를 만드는 C++17 CPU ray tracer를 단계적으로 구현하는 프로젝트다.
+
+## 초기 개발 규약
+
+- C++17과 표준 라이브러리를 기준으로 한다.
+- 직접 빌드는 `-Wall -Wextra -Wpedantic` 경고를 활성화한다.
+- 소스, 공개 선언, 장면 fixture, 검사를 `src/`, `include/`, `scenes/`, `tests/`에 분리한다.
+- 성능 근거가 생기면 측정 코드와 결과를 `benchmarks/`에 둔다.
+- 실행 파일, object, build directory, 생성 PPM은 버전 관리하지 않는다.
+- 구현 커밋은 해당 시점에 가능한 build와 결정적 검증을 통과시킨다.
+- 실패 계약과 자원 소유권은 공개 경계와 함께 관리한다.
+
+## 예정 범위
+
+- 해상도, 환경광, 카메라, 점광원 장면 입력
+- 구, 평면, 유한 원기둥 교차 계산
+- 픽셀 중심 광선, 직접광, 그림자, P3 PPM 출력
+- 선형 기준 구현을 보존하는 가속과 실행 모드 간 결과 결정성
+
+GUI, GPU 렌더링, mesh, texture, 장면 편집기는 초기 범위에 포함하지 않는다.


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


## `test(render): 장면 렌더링 smoke 검사 추가`

diff --git a/Makefile b/Makefile
index 263b96c..04a91fd 100644
--- a/Makefile
+++ b/Makefile
@@ -13,7 +13,7 @@ $(TARGET): $(SRC)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -o $@ $(SRC)
 
 test: $(TARGET)
-	./$(TARGET) scenes/basic.rt /tmp/ray-scene-tracer-check.ppm --checksum >/dev/null
+	tests/render_smoke.sh
 
 clean:
 	rm -f $(TARGET)
diff --git a/tests/render_smoke.sh b/tests/render_smoke.sh
new file mode 100755
index 0000000..d937cbe
--- /dev/null
+++ b/tests/render_smoke.sh
@@ -0,0 +1,72 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT=$(cd "$(dirname "$0")/.." && pwd)
+BIN="$ROOT/ray-scene-tracer"
+TMP=$(mktemp -d)
+trap 'rm -rf "$TMP"' EXIT
+
+make -C "$ROOT" >/dev/null
+
+BAD_SCENE="$TMP/bad.rt"
+BAD_OUT="$TMP/bad.ppm"
+cat >"$BAD_SCENE" <<'SCENE'
+R 16 8
+C 0,0,3 0,0,-1 60
+not_a_shape 1 2 3
+SCENE
+
+if "$BIN" "$BAD_SCENE" "$BAD_OUT" >"$TMP/bad.stdout" 2>"$TMP/bad.stderr"; then
+    echo "expected parser failure for unknown directive" >&2
+    exit 1
+fi
+if [[ -s "$BAD_OUT" ]]; then
+    echo "parser failure should not leave a rendered image" >&2
+    exit 1
+fi
+
+SCENE_FILE="$TMP/smoke.rt"
+cat >"$SCENE_FILE" <<'SCENE'
+# Educational miniRT-style scene. RGB uses 0..255.
+R 64 32
+A 0.12 255,255,255
+C 0,0.8,3.2 0,0.25,-1 55
+L -3,5,2.5 0.9 255,244,220
+sp 0,0,-1.4 1.10 220,70,45
+sp 0.9,-0.1,-2.2 0.90 65,120,220
+pl 0,-0.65,0 0,1,0 185,190,178
+SCENE
+
+PPM_ONE="$TMP/one.ppm"
+PPM_TWO="$TMP/two.ppm"
+CHECK_ONE=$("$BIN" "$SCENE_FILE" "$PPM_ONE" --checksum)
+CHECK_TWO=$("$BIN" "$SCENE_FILE" "$PPM_TWO" --checksum)
+
+mapfile -t HEADER < <(head -n 3 "$PPM_ONE")
+if [[ "${HEADER[0]}" != "P3" ]]; then
+    echo "expected P3 PPM magic, got ${HEADER[0]}" >&2
+    exit 1
+fi
+if [[ "${HEADER[1]}" != "64 32" ]]; then
+    echo "expected PPM dimensions 64 32, got ${HEADER[1]}" >&2
+    exit 1
+fi
+if [[ "${HEADER[2]}" != "255" ]]; then
+    echo "expected PPM max channel 255, got ${HEADER[2]}" >&2
+    exit 1
+fi
+
+if [[ ! "$CHECK_ONE" =~ ^[0-9a-f]{16}$ ]]; then
+    echo "expected checksum hex output, got $CHECK_ONE" >&2
+    exit 1
+fi
+if [[ "$CHECK_ONE" != "$CHECK_TWO" ]]; then
+    echo "checksum output is not deterministic" >&2
+    exit 1
+fi
+if ! cmp -s "$PPM_ONE" "$PPM_TWO"; then
+    echo "PPM output is not deterministic" >&2
+    exit 1
+fi
+
+echo "render smoke checks passed"


## `build(cmake): 코어 라이브러리와 검증 타깃 구성`

diff --git a/.gitignore b/.gitignore
index ba4054e..62cc2c8 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1,4 +1,4 @@
 ray-scene-tracer
 *.o
 *.d
-/build*/
+/build/
diff --git a/CMakeLists.txt b/CMakeLists.txt
new file mode 100644
index 0000000..b952b6d
--- /dev/null
+++ b/CMakeLists.txt
@@ -0,0 +1,38 @@
+cmake_minimum_required(VERSION 3.16)
+
+project(ray_scene_tracer LANGUAGES CXX)
+
+set(CMAKE_CXX_STANDARD 17)
+set(CMAKE_CXX_STANDARD_REQUIRED ON)
+set(CMAKE_CXX_EXTENSIONS OFF)
+
+add_library(raycore
+    src/camera.cpp
+    src/geometry.cpp
+    src/math.cpp
+    src/output.cpp
+    src/parser.cpp
+    src/renderer.cpp
+    src/scene.cpp
+    src/shading.cpp
+)
+target_include_directories(raycore PUBLIC include)
+
+if(MSVC)
+    target_compile_options(raycore PRIVATE /W4)
+else()
+    target_compile_options(raycore PRIVATE -Wall -Wextra -Wpedantic)
+endif()
+
+add_executable(ray-scene-tracer src/main.cpp)
+target_link_libraries(ray-scene-tracer PRIVATE raycore)
+
+include(CTest)
+if(BUILD_TESTING)
+    add_test(
+        NAME render_smoke
+        COMMAND bash
+                ${CMAKE_CURRENT_SOURCE_DIR}/tests/render_smoke.sh
+                $<TARGET_FILE:ray-scene-tracer>
+    )
+endif()
diff --git a/Makefile b/Makefile
index 04a91fd..6fbc625 100644
--- a/Makefile
+++ b/Makefile
@@ -1,19 +1,17 @@
-CXX ?= c++
-CPPFLAGS ?= -Iinclude
-CXXFLAGS ?= -std=c++17 -Wall -Wextra -Wpedantic -O2
-
-TARGET := ray-scene-tracer
-SRC := $(sort $(wildcard src/*.cpp))
+BUILD_DIR ?= build
+CMAKE ?= cmake
 
 .PHONY: all clean test
 
-all: $(TARGET)
-
-$(TARGET): $(SRC)
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -o $@ $(SRC)
+all:
+	$(CMAKE) -S . -B $(BUILD_DIR) -DCMAKE_BUILD_TYPE=Release
+	$(CMAKE) --build $(BUILD_DIR)
+	$(CMAKE) -E create_symlink $(BUILD_DIR)/ray-scene-tracer ray-scene-tracer
 
-test: $(TARGET)
-	tests/render_smoke.sh
+test:
+	$(CMAKE) -S . -B $(BUILD_DIR) -DCMAKE_BUILD_TYPE=Release
+	$(CMAKE) --build $(BUILD_DIR)
+	ctest --test-dir $(BUILD_DIR) --output-on-failure
 
 clean:
-	rm -f $(TARGET)
+	$(CMAKE) -E rm -rf $(BUILD_DIR) ray-scene-tracer
diff --git a/tests/render_smoke.sh b/tests/render_smoke.sh
index d937cbe..c944071 100755
--- a/tests/render_smoke.sh
+++ b/tests/render_smoke.sh
@@ -2,11 +2,13 @@
 set -euo pipefail
 
 ROOT=$(cd "$(dirname "$0")/.." && pwd)
-BIN="$ROOT/ray-scene-tracer"
+BIN=${1:-"$ROOT/ray-scene-tracer"}
 TMP=$(mktemp -d)
 trap 'rm -rf "$TMP"' EXIT
 
-make -C "$ROOT" >/dev/null
+if [[ $# -eq 0 ]]; then
+    make -C "$ROOT" >/dev/null
+fi
 
 BAD_SCENE="$TMP/bad.rt"
 BAD_OUT="$TMP/bad.ppm"


## `test(core): 수학·기하·파서·출력 회귀 기준 추가`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index b952b6d..a1415ea 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -29,6 +29,14 @@ target_link_libraries(ray-scene-tracer PRIVATE raycore)
 
 include(CTest)
 if(BUILD_TESTING)
+    add_executable(ray-core-tests tests/core_tests.cpp)
+    target_link_libraries(ray-core-tests PRIVATE raycore)
+    target_compile_definitions(
+        ray-core-tests
+        PRIVATE RAY_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}"
+    )
+
+    add_test(NAME core_regression COMMAND ray-core-tests)
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
new file mode 100644
index 0000000..a2b33bd
--- /dev/null
+++ b/tests/core_tests.cpp
@@ -0,0 +1,113 @@
+#include "ray.hpp"
+
+#include <cmath>
+#include <cstdio>
+#include <fstream>
+#include <iostream>
+#include <sstream>
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
+bool nearlyEqual(double left, double right, double epsilon = 1.0e-9) {
+    return std::fabs(left - right) <= epsilon;
+}
+
+std::string readFile(const std::string& path) {
+    std::ifstream input(path);
+    std::ostringstream contents;
+    contents << input.rdbuf();
+    return contents.str();
+}
+
+void testMath() {
+    const ray::Vec3 a(1.0, 2.0, 3.0);
+    const ray::Vec3 b(-2.0, 0.5, 4.0);
+    require(a + b == ray::Vec3(-1.0, 2.5, 7.0), "vector addition");
+    require(nearlyEqual(ray::dot(a, b), 11.0), "dot product");
+    require(ray::cross(ray::Vec3(1.0, 0.0, 0.0),
+                       ray::Vec3(0.0, 1.0, 0.0)) ==
+                ray::Vec3(0.0, 0.0, 1.0),
+            "cross product");
+    require(nearlyEqual(ray::normalize(ray::Vec3(0.0, 3.0, 4.0)).length(), 1.0),
+            "normalization");
+}
+
+void testGeometry() {
+    const ray::Material white(ray::Color(1.0, 1.0, 1.0));
+    const ray::Ray forward(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0));
+    ray::HitRecord hit;
+
+    const ray::Sphere sphere(ray::Vec3(0.0, 0.0, 5.0), 1.0, white);
+    require(sphere.intersect(forward, ray::kRayTMin, 100.0, hit), "sphere hit");
+    require(nearlyEqual(hit.t, 4.0), "sphere distance");
+
+    const ray::Plane plane(ray::Vec3(0.0, 0.0, 2.0),
+                           ray::Vec3(0.0, 0.0, -1.0),
+                           white);
+    require(plane.intersect(forward, ray::kRayTMin, 100.0, hit), "plane hit");
+    require(nearlyEqual(hit.t, 2.0), "plane distance");
+
+    const ray::Cylinder cylinder(ray::Vec3(0.0, 0.0, 5.0),
+                                 ray::Vec3(0.0, 1.0, 0.0),
+                                 1.0,
+                                 2.0,
+                                 white);
+    require(cylinder.intersect(forward, ray::kRayTMin, 100.0, hit),
+            "cylinder hit");
+    require(nearlyEqual(hit.t, 4.0), "cylinder distance");
+}
+
+void testInvalidFixture() {
+    bool rejected = false;
+    try {
+        (void)ray::parser::parseSceneFile(
+            std::string(RAY_SOURCE_DIR) + "/scenes/invalid.rt");
+    } catch (const ray::ParseError& error) {
+        rejected = error.line() == 3;
+    }
+    require(rejected, "invalid scene fixture");
+}
+
+void testOutput() {
+    ray::Image image(2, 1);
+    image.pixels = {255, 0, 16, 0, 127, 255};
+
+    const std::string path = "ray-core-test-output.ppm";
+    ray::writePpm(image, path);
+    const std::string ppm = readFile(path);
+    std::remove(path.c_str());
+
+    require(ppm == "P3\n2 1\n255\n255 0 16\n0 127 255\n", "PPM encoding");
+}
+
+void testRenderGolden() {
+    const ray::Scene scene = ray::loadScene(
+        std::string(RAY_SOURCE_DIR) + "/scenes/basic.rt");
+    const ray::Image image = ray::renderScene(scene);
+    require(image.width == 640 && image.height == 360, "render dimensions");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        testMath();
+        testGeometry();
+        testInvalidFixture();
+        testOutput();
+        testRenderGolden();
+    } catch (const std::exception& error) {
+        std::cerr << "core regression failed: " << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "core regression checks passed\n";
+    return 0;
+}


## `test(accel): 선형 탐색과 BVH 결과 동치 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 273d7ba..97618dc 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -41,6 +41,11 @@ if(BUILD_TESTING)
     )
 
     add_test(NAME core_regression COMMAND ray-core-tests)
+
+    add_executable(ray-accel-tests tests/accel_tests.cpp)
+    target_link_libraries(ray-accel-tests PRIVATE raycore)
+    add_test(NAME accel_regression COMMAND ray-accel-tests)
+
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/accel_tests.cpp b/tests/accel_tests.cpp
new file mode 100644
index 0000000..a8d86d9
--- /dev/null
+++ b/tests/accel_tests.cpp
@@ -0,0 +1,223 @@
+#include "ray.hpp"
+
+#include <cmath>
+#include <iostream>
+#include <memory>
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
+bool nearlyEqual(double left, double right, double epsilon = 1.0e-9) {
+    return std::fabs(left - right) <= epsilon;
+}
+
+void requireEquivalentHit(ray::Scene& scene,
+                          const ray::Ray& ray_value,
+                          const std::string& label) {
+    ray::HitRecord linear_hit;
+    ray::HitRecord bvh_hit;
+    const bool linear_found =
+        scene.intersect(ray_value,
+                        ray::kRayTMin,
+                        1000.0,
+                        linear_hit,
+                        ray::AccelMode::Linear);
+    const bool bvh_found =
+        scene.intersect(ray_value,
+                        ray::kRayTMin,
+                        1000.0,
+                        bvh_hit,
+                        ray::AccelMode::Bvh);
+    require(linear_found == bvh_found, label + " hit state");
+    if (!linear_found) {
+        return;
+    }
+    require(nearlyEqual(linear_hit.t, bvh_hit.t),
+            label + " distance");
+    require(linear_hit.point == bvh_hit.point,
+            label + " point");
+    require(linear_hit.normal == bvh_hit.normal,
+            label + " normal");
+    require(linear_hit.material.albedo == bvh_hit.material.albedo,
+            label + " material");
+    require(linear_hit.shape == bvh_hit.shape,
+            label + " primitive");
+}
+
+void testEmptyScene() {
+    ray::Scene scene;
+    scene.buildAcceleration();
+    requireEquivalentHit(
+        scene,
+        ray::Ray(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0)),
+        "empty scene");
+}
+
+void testSingleSphere() {
+    ray::Scene scene;
+    scene.addShape(std::make_unique<ray::Sphere>(
+        ray::Vec3(0.0, 0.0, 5.0),
+        1.0,
+        ray::Material(ray::Color(0.8, 0.2, 0.1))));
+    scene.buildAcceleration();
+    requireEquivalentHit(
+        scene,
+        ray::Ray(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0)),
+        "single sphere");
+}
+
+void testPlaneOnly() {
+    ray::Scene scene;
+    scene.addShape(std::make_unique<ray::Plane>(
+        ray::Vec3(0.0, 0.0, 3.0),
+        ray::Vec3(0.0, 0.0, -1.0),
+        ray::Material(ray::Color(0.2, 0.8, 0.1))));
+    scene.buildAcceleration();
+    requireEquivalentHit(
+        scene,
+        ray::Ray(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0)),
+        "plane-only scene");
+}
+
+void testCylinder() {
+    ray::Scene scene;
+    scene.addShape(std::make_unique<ray::Cylinder>(
+        ray::Vec3(0.0, 0.0, 5.0),
+        ray::Vec3(1.0, 1.0, 0.0),
+        1.0,
+        3.0,
+        ray::Material(ray::Color(0.2, 0.4, 0.9))));
+    scene.buildAcceleration();
+    requireEquivalentHit(
+        scene,
+        ray::Ray(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0)),
+        "arbitrary-axis cylinder");
+}
+
+void testEqualDistanceTie() {
+    ray::Scene scene;
+    scene.addShape(std::make_unique<ray::Sphere>(
+        ray::Vec3(0.0, 0.0, 5.0),
+        1.0,
+        ray::Material(ray::Color(1.0, 0.0, 0.0))));
+    scene.addShape(std::make_unique<ray::Sphere>(
+        ray::Vec3(0.0, 0.0, 5.0),
+        1.0,
+        ray::Material(ray::Color(0.0, 1.0, 0.0))));
+    const ray::Shape* expected = scene.shapes.back().get();
+    scene.buildAcceleration();
+
+    ray::HitRecord linear_hit;
+    ray::HitRecord bvh_hit;
+    const ray::Ray ray_value(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0));
+    require(scene.intersect(ray_value,
+                            ray::kRayTMin,
+                            100.0,
+                            linear_hit,
+                            ray::AccelMode::Linear),
+            "linear equal-distance hit");
+    require(scene.intersect(ray_value,
+                            ray::kRayTMin,
+                            100.0,
+                            bvh_hit,
+                            ray::AccelMode::Bvh),
+            "BVH equal-distance hit");
+    require(linear_hit.shape == expected &&
+                bvh_hit.shape == expected,
+            "later primitive wins equal-distance tie");
+}
+
+ray::Scene makeDenseScene() {
+    ray::Scene scene;
+    scene.width = 160;
+    scene.height = 90;
+    scene.hasResolution = true;
+    scene.hasAmbient = true;
+    scene.hasCamera = true;
+    scene.ambientRatio = 0.08;
+    scene.ambientColor = ray::Color(1.0, 1.0, 1.0);
+    scene.background = ray::Color(0.02, 0.03, 0.05);
+    scene.camera = ray::Camera(ray::Vec3(0.0, 5.0, -18.0),
+                               ray::Vec3(0.0, -0.12, 1.0),
+                               48.0);
+    scene.addLight(ray::Light(ray::Vec3(-12.0, 16.0, -8.0),
+                              0.85,
+                              ray::Color(1.0, 0.96, 0.88)));
+    scene.addLight(ray::Light(ray::Vec3(14.0, 10.0, 12.0),
+                              0.55,
+                              ray::Color(0.75, 0.85, 1.0)));
+    scene.addShape(std::make_unique<ray::Plane>(
+        ray::Vec3(0.0, -1.0, 0.0),
+        ray::Vec3(0.0, 1.0, 0.0),
+        ray::Material(ray::Color(0.35, 0.38, 0.42))));
+
+    for (int row = 0; row < 20; ++row) {
+        for (int column = 0; column < 20; ++column) {
+            const double x = (column - 9.5) * 1.05;
+            const double z = 2.0 + row * 1.05;
+            const double y = -0.45 + 0.18 * ((row + column) % 4);
+            const ray::Color color(
+                0.2 + 0.6 * static_cast<double>(column % 5) / 4.0,
+                0.2 + 0.6 * static_cast<double>(row % 5) / 4.0,
+                0.25 +
+                    0.5 *
+                        static_cast<double>((row + column) % 5) /
+                        4.0);
+            scene.addShape(std::make_unique<ray::Sphere>(
+                ray::Vec3(x, y, z),
+                0.42,
+                ray::Material(color)));
+        }
+    }
+    scene.buildAcceleration();
+    return scene;
+}
+
+void testDenseRender() {
+    const ray::Scene scene = makeDenseScene();
+    ray::RenderSettings linear_settings;
+    linear_settings.accelMode = ray::AccelMode::Linear;
+    ray::RenderSettings bvh_settings;
+    bvh_settings.accelMode = ray::AccelMode::Bvh;
+    ray::RenderStats linear_stats;
+    ray::RenderStats bvh_stats;
+
+    const ray::Image linear =
+        ray::renderScene(scene, linear_settings, &linear_stats);
+    const ray::Image bvh =
+        ray::renderScene(scene, bvh_settings, &bvh_stats);
+
+    require(linear.pixels == bvh.pixels,
+            "linear and BVH pixels");
+    require(ray::checksumHex(linear) == ray::checksumHex(bvh),
+            "linear and BVH checksum");
+    require(bvh_stats.primitiveTests * 4 <
+                linear_stats.primitiveTests,
+            "BVH primitive test reduction");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        testEmptyScene();
+        testSingleSphere();
+        testPlaneOnly();
+        testCylinder();
+        testEqualDistanceTie();
+        testDenseRender();
+    } catch (const std::exception& error) {
+        std::cerr << "acceleration regression failed: "
+                  << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "acceleration regression checks passed\n";
+    return 0;
+}


## `test(material): 재질 파싱과 반사 깊이 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 97618dc..948f7b5 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -46,6 +46,14 @@ if(BUILD_TESTING)
     target_link_libraries(ray-accel-tests PRIVATE raycore)
     add_test(NAME accel_regression COMMAND ray-accel-tests)
 
+    add_executable(ray-material-tests tests/material_tests.cpp)
+    target_link_libraries(ray-material-tests PRIVATE raycore)
+    target_compile_definitions(
+        ray-material-tests
+        PRIVATE RAY_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}"
+    )
+    add_test(NAME material_regression COMMAND ray-material-tests)
+
     add_test(
         NAME render_smoke
         COMMAND bash
diff --git a/tests/material_tests.cpp b/tests/material_tests.cpp
new file mode 100644
index 0000000..f514917
--- /dev/null
+++ b/tests/material_tests.cpp
@@ -0,0 +1,124 @@
+#include "ray.hpp"
+
+#include <cmath>
+#include <iostream>
+#include <memory>
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
+bool nearlyEqual(double left, double right, double epsilon = 1.0e-9) {
+    return std::fabs(left - right) <= epsilon;
+}
+
+std::string scenePrefix() {
+    return "R 8 8\n"
+           "A 0.1 255,255,255\n"
+           "C 0,0,0 0,0,1 60\n";
+}
+
+void testMaterialParsing() {
+    const ray::Scene omitted = ray::parser::parseSceneText(
+        scenePrefix() + "sp 0,0,5 2 255,0,0\n",
+        "omitted.rt");
+    require(omitted.shapes[0]->material().type ==
+                ray::MaterialType::Diffuse,
+            "omitted material defaults to diffuse");
+
+    const ray::Scene explicit_types = ray::parser::parseSceneText(
+        scenePrefix() +
+            "sp 0,0,5 2 255,0,0 metal\n"
+            "pl 0,-1,0 0,1,0 255,255,255 diffuse\n"
+            "cy 2,0,5 0,1,0 1 2 0,0,255 metal\n",
+        "materials.rt");
+    require(explicit_types.shapes[0]->material().type ==
+                ray::MaterialType::Metal &&
+                explicit_types.shapes[1]->material().type ==
+                    ray::MaterialType::Diffuse &&
+                explicit_types.shapes[2]->material().type ==
+                    ray::MaterialType::Metal,
+            "explicit material parsing");
+
+    bool rejected = false;
+    try {
+        (void)ray::parser::parseSceneText(
+            scenePrefix() + "sp 0,0,5 2 255,0,0 glass\n",
+            "unknown-material.rt");
+    } catch (const ray::ParseError&) {
+        rejected = true;
+    }
+    require(rejected, "unknown material rejection");
+}
+
+ray::Scene makeMirrorScene() {
+    ray::Scene scene;
+    scene.width = 1;
+    scene.height = 1;
+    scene.hasResolution = true;
+    scene.hasAmbient = true;
+    scene.hasCamera = true;
+    scene.background = ray::Color(0.25, 0.5, 0.75);
+    scene.addShape(std::make_unique<ray::Plane>(
+        ray::Vec3(0.0, 0.0, 1.0),
+        ray::Vec3(0.0, 0.0, -1.0),
+        ray::Material(ray::Color(0.8, 0.5, 0.25),
+                      ray::MaterialType::Metal)));
+    scene.buildAcceleration();
+    return scene;
+}
+
+void testMetalDepthAndReflection() {
+    const ray::Scene scene = makeMirrorScene();
+    const ray::Ray primary(ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0));
+    const ray::Color exhausted =
+        ray::traceRay(scene, primary, 0, ray::AccelMode::Bvh);
+    require(exhausted == ray::Color(), "exhausted metal path is black");
+
+    ray::RenderStats first_stats;
+    const ray::Color first =
+        ray::traceRay(scene,
+                      primary,
+                      1,
+                      ray::AccelMode::Bvh,
+                      &first_stats);
+    const ray::Color second =
+        ray::traceRay(scene, primary, 1, ray::AccelMode::Bvh);
+    require(nearlyEqual(first.x, 0.2) &&
+                nearlyEqual(first.y, 0.25) &&
+                nearlyEqual(first.z, 0.1875),
+            "perfect metal reflection");
+    require(first == second, "metal reflection determinism");
+    require(first_stats.secondaryRays == 1,
+            "secondary ray accounting");
+}
+
+void testDiffuseGolden() {
+    const ray::Scene scene = ray::loadScene(
+        std::string(RAY_SOURCE_DIR) + "/scenes/basic.rt");
+    const ray::Image image = ray::renderScene(scene);
+    require(ray::checksumHex(image) == "456dc8d87ebf194f",
+            "existing diffuse render");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        testMaterialParsing();
+        testMetalDepthAndReflection();
+        testDiffuseGolden();
+    } catch (const std::exception& error) {
+        std::cerr << "material regression failed: "
+                  << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "material regression checks passed\n";
+    return 0;
+}


