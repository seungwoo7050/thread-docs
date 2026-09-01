## `feat(renderer): 작업자 수 설정과 자동 선택 추가`

diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
index dcc8051..f446348 100644
--- a/benchmarks/render_benchmark.cpp
+++ b/benchmarks/render_benchmark.cpp
@@ -66,6 +66,7 @@ Sample render(const ray::Scene& scene, ray::AccelMode mode) {
     ray::RenderStats stats;
     ray::RenderSettings settings;
     settings.accelMode = mode;
+    settings.threadCount = 1;
     const ray::Image image = ray::renderScene(scene, settings, &stats);
     return Sample{stats.renderMilliseconds, stats, ray::checksumHex(image)};
 }
diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index 9089004..9cf6a87 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -13,6 +13,7 @@ struct RenderSettings {
     double tMin;
     double tMax;
     AccelMode accelMode;
+    unsigned int threadCount;
 
     RenderSettings();
 };
diff --git a/src/renderer.cpp b/src/renderer.cpp
index 9cc16e0..472076b 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -34,7 +34,8 @@ RenderSettings::RenderSettings()
       maxDepth(1),
       tMin(kRayTMin),
       tMax(std::numeric_limits<double>::infinity()),
-      accelMode(AccelMode::Bvh) {}
+      accelMode(AccelMode::Bvh),
+      threadCount(0) {}
 
 Image::Image() : width(0), height(0), pixels() {}
 
@@ -59,9 +60,12 @@ Image renderScene(const Scene& scene,
         (static_cast<std::size_t>(scene.height) + kTileSize - 1) /
         kTileSize;
     const std::size_t tile_count = tiles_x * tiles_y;
-    unsigned int worker_count = std::thread::hardware_concurrency();
+    unsigned int worker_count = settings.threadCount;
     if (worker_count == 0) {
-        worker_count = 1;
+        worker_count = std::thread::hardware_concurrency();
+        if (worker_count == 0) {
+            worker_count = 1;
+        }
     }
     worker_count = static_cast<unsigned int>(
         std::min<std::size_t>(worker_count, tile_count));


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


## `fix(renderer): 작업자 예외를 호출자에게 전달`

diff --git a/src/renderer.cpp b/src/renderer.cpp
index 4e3f41b..2dd8428 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -4,6 +4,7 @@
 #include <atomic>
 #include <chrono>
 #include <cmath>
+#include <exception>
 #include <limits>
 #include <stdexcept>
 #include <thread>
@@ -84,6 +85,7 @@ Image renderScene(const Scene& scene,
     std::vector<WorkerStats> worker_stats(worker_count);
     std::atomic<std::size_t> next_tile{0};
     std::vector<std::thread> workers;
+    std::vector<std::exception_ptr> worker_errors(worker_count);
     workers.reserve(worker_count);
 
     struct ThreadJoiner {
@@ -108,61 +110,71 @@ Image renderScene(const Scene& scene,
 
     for (unsigned int worker = 0; worker < worker_count; ++worker) {
         workers.emplace_back([&, worker]() {
-            RenderStats& local = worker_stats[worker].values;
-            for (;;) {
-                const std::size_t tile =
-                    next_tile.fetch_add(1, std::memory_order_relaxed);
-                if (tile >= tile_count) {
-                    break;
-                }
-                const int start_x =
-                    static_cast<int>((tile % tiles_x) * kTileSize);
-                const int start_y =
-                    static_cast<int>((tile / tiles_x) * kTileSize);
-                const int end_x =
-                    std::min(start_x + kTileSize, scene.width);
-                const int end_y =
-                    std::min(start_y + kTileSize, scene.height);
-
-                for (int y = start_y; y < end_y; ++y) {
-                    for (int x = start_x; x < end_x; ++x) {
-                        const Ray ray =
-                            makeCameraRay(scene.camera,
-                                          camera_frame,
-                                          scene.width,
-                                          scene.height,
-                                          x + 0.5,
-                                          y + 0.5);
-                        ++local.primaryRays;
-                        const Color color =
-                            traceRay(scene,
-                                     ray,
-                                     settings.maxDepth,
-                                     settings.accelMode,
-                                     &local);
-                        const Color clamped = clampColor(color);
-                        const std::size_t offset =
-                            (static_cast<std::size_t>(y) *
-                                 static_cast<std::size_t>(scene.width) +
-                             static_cast<std::size_t>(x)) *
-                            3;
-                        image.pixels[offset] =
-                            static_cast<unsigned char>(
-                                std::lround(clamped.x * 255.0));
-                        image.pixels[offset + 1] =
-                            static_cast<unsigned char>(
-                                std::lround(clamped.y * 255.0));
-                        image.pixels[offset + 2] =
-                            static_cast<unsigned char>(
-                                std::lround(clamped.z * 255.0));
+            try {
+                RenderStats& local = worker_stats[worker].values;
+                for (;;) {
+                    const std::size_t tile =
+                        next_tile.fetch_add(1, std::memory_order_relaxed);
+                    if (tile >= tile_count) {
+                        break;
+                    }
+                    const int start_x =
+                        static_cast<int>((tile % tiles_x) * kTileSize);
+                    const int start_y =
+                        static_cast<int>((tile / tiles_x) * kTileSize);
+                    const int end_x =
+                        std::min(start_x + kTileSize, scene.width);
+                    const int end_y =
+                        std::min(start_y + kTileSize, scene.height);
+
+                    for (int y = start_y; y < end_y; ++y) {
+                        for (int x = start_x; x < end_x; ++x) {
+                            const Ray ray =
+                                makeCameraRay(scene.camera,
+                                              camera_frame,
+                                              scene.width,
+                                              scene.height,
+                                              x + 0.5,
+                                              y + 0.5);
+                            ++local.primaryRays;
+                            const Color color =
+                                traceRay(scene,
+                                         ray,
+                                         settings.maxDepth,
+                                         settings.accelMode,
+                                         &local);
+                            const Color clamped = clampColor(color);
+                            const std::size_t offset =
+                                (static_cast<std::size_t>(y) *
+                                     static_cast<std::size_t>(scene.width) +
+                                 static_cast<std::size_t>(x)) *
+                                3;
+                            image.pixels[offset] =
+                                static_cast<unsigned char>(
+                                    std::lround(clamped.x * 255.0));
+                            image.pixels[offset + 1] =
+                                static_cast<unsigned char>(
+                                    std::lround(clamped.y * 255.0));
+                            image.pixels[offset + 2] =
+                                static_cast<unsigned char>(
+                                    std::lround(clamped.z * 255.0));
+                        }
                     }
                 }
+            } catch (...) {
+                worker_errors[worker] = std::current_exception();
+                next_tile.store(tile_count, std::memory_order_relaxed);
             }
         });
     }
     for (std::thread& worker : workers) {
         worker.join();
     }
+    for (const std::exception_ptr& error : worker_errors) {
+        if (error) {
+            std::rethrow_exception(error);
+        }
+    }
 
     if (stats) {
         *stats = RenderStats();


## `test(renderer): 작업자 실패 전파와 회수 검증`

diff --git a/tests/render_tests.cpp b/tests/render_tests.cpp
index 1e74ea4..b355966 100644
--- a/tests/render_tests.cpp
+++ b/tests/render_tests.cpp
@@ -1,6 +1,8 @@
 #include "ray.hpp"
 
 #include <iostream>
+#include <memory>
+#include <optional>
 #include <stdexcept>
 #include <string>
 
@@ -60,6 +62,46 @@ void requireSameWork(const Result& left,
             label + " AABB tests");
 }
 
+class ThrowingShape : public ray::Shape {
+public:
+    bool intersect(const ray::Ray&,
+                   double,
+                   double,
+                   ray::HitRecord&) const override {
+        throw std::runtime_error("worker exception sentinel");
+    }
+
+    std::optional<ray::Aabb> bounds() const override {
+        return std::nullopt;
+    }
+
+    std::string typeName() const override {
+        return "throwing";
+    }
+};
+
+void testWorkerExceptionPropagation() {
+    ray::Scene scene;
+    scene.width = 32;
+    scene.height = 16;
+    scene.camera = ray::Camera(
+        ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0), 60.0);
+    scene.addShape(std::make_unique<ThrowingShape>());
+    scene.buildAcceleration();
+
+    ray::RenderSettings settings;
+    settings.threadCount = 4;
+    bool propagated = false;
+    try {
+        (void)ray::renderScene(scene, settings);
+    } catch (const std::runtime_error& error) {
+        propagated =
+            std::string(error.what()) == "worker exception sentinel";
+    }
+    require(propagated,
+            "renderScene propagates an exception from a worker");
+}
+
 }  // namespace
 
 int main() {
@@ -90,6 +132,7 @@ int main() {
                         "BVH thread count");
         require(linear_one.stats.primaryRays == 96u * 54u,
                 "primary ray count");
+        testWorkerExceptionPropagation();
     } catch (const std::exception& error) {
         std::cerr << "render determinism failed: "
                   << error.what() << '\n';
