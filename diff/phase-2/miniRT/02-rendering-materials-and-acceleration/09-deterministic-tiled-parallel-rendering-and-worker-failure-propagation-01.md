# 결정적 타일 병렬 렌더링과 작업자 실패 전파

## `feat(renderer): 직렬 이미지 렌더링 구현`

diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index c72be15..0b130d9 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -3,9 +3,28 @@
 #include "ray/camera.hpp"
 
 #include <limits>
+#include <vector>
 
 namespace ray {
 
+struct RenderSettings {
+    int samplesPerPixel;
+    int maxDepth;
+    double tMin;
+    double tMax;
+
+    RenderSettings();
+};
+
+struct Image {
+    int width;
+    int height;
+    std::vector<unsigned char> pixels;
+
+    Image();
+    Image(int width_value, int height_value);
+};
+
 bool findNearestHit(const Scene& scene,
                     const Ray& ray,
                     HitRecord& hit,
@@ -20,5 +39,7 @@ Color shadeHit(const Scene& scene,
 Color traceRay(const Scene& scene,
                const Ray& ray,
                int max_depth = 1);
+Image renderScene(const Scene& scene,
+                  const RenderSettings& settings = RenderSettings());
 
 }  // namespace ray
diff --git a/src/renderer.cpp b/src/renderer.cpp
new file mode 100644
index 0000000..de19a3d
--- /dev/null
+++ b/src/renderer.cpp
@@ -0,0 +1,43 @@
+#include "ray/renderer.hpp"
+
+#include <cmath>
+
+namespace ray {
+
+RenderSettings::RenderSettings()
+    : samplesPerPixel(1),
+      maxDepth(1),
+      tMin(kRayTMin),
+      tMax(std::numeric_limits<double>::infinity()) {}
+
+Image::Image() : width(0), height(0), pixels() {}
+
+Image::Image(int width_value, int height_value)
+    : width(width_value),
+      height(height_value),
+      pixels(static_cast<std::size_t>(width_value * height_value * 3), 0) {}
+
+Image renderScene(const Scene& scene, const RenderSettings& settings) {
+    Image image(scene.width, scene.height);
+    std::size_t offset = 0;
+    for (int y = 0; y < scene.height; ++y) {
+        for (int x = 0; x < scene.width; ++x) {
+            const Ray ray = makeCameraRay(scene.camera,
+                                          scene.width,
+                                          scene.height,
+                                          x + 0.5,
+                                          y + 0.5);
+            const Color color = traceRay(scene, ray, settings.maxDepth);
+            const Color clamped = clampColor(color);
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.x * 255.0));
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.y * 255.0));
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.z * 255.0));
+        }
+    }
+    return image;
+}
+
+}  // namespace ray


## `perf(render): 광선과 교차 작업량 계측 추가`

diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index 0b130d9..9c66ab5 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -29,17 +29,22 @@ bool findNearestHit(const Scene& scene,
                     const Ray& ray,
                     HitRecord& hit,
                     double t_min = kRayTMin,
-                    double t_max = std::numeric_limits<double>::infinity());
+                    double t_max = std::numeric_limits<double>::infinity(),
+                    RenderStats* stats = nullptr);
 bool isOccluded(const Scene& scene,
                 const Ray& shadow_ray,
-                double max_distance);
+                double max_distance,
+                RenderStats* stats = nullptr);
 Color shadeHit(const Scene& scene,
                const HitRecord& hit,
-               const Ray& view_ray);
+               const Ray& view_ray,
+               RenderStats* stats = nullptr);
 Color traceRay(const Scene& scene,
                const Ray& ray,
-               int max_depth = 1);
+               int max_depth = 1,
+               RenderStats* stats = nullptr);
 Image renderScene(const Scene& scene,
-                  const RenderSettings& settings = RenderSettings());
+                  const RenderSettings& settings = RenderSettings(),
+                  RenderStats* stats = nullptr);
 
 }  // namespace ray
diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index bbbcc4d..002c887 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -2,6 +2,7 @@
 
 #include "ray/geometry.hpp"
 
+#include <cstdint>
 #include <memory>
 #include <vector>
 
@@ -30,6 +31,15 @@ struct Camera {
            double fov_degrees_value);
 };
 
+struct RenderStats {
+    std::uint64_t primaryRays = 0;
+    std::uint64_t secondaryRays = 0;
+    std::uint64_t shadowRays = 0;
+    std::uint64_t primitiveTests = 0;
+    std::uint64_t aabbTests = 0;
+    double renderMilliseconds = 0.0;
+};
+
 class Scene {
 public:
     int width;
@@ -51,7 +61,8 @@ public:
     bool intersect(const Ray& ray,
                    double t_min,
                    double t_max,
-                   HitRecord& hit) const;
+                   HitRecord& hit,
+                   RenderStats* stats = nullptr) const;
 };
 
 }  // namespace ray
diff --git a/src/renderer.cpp b/src/renderer.cpp
index de19a3d..fd54281 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -1,5 +1,6 @@
 #include "ray/renderer.hpp"
 
+#include <chrono>
 #include <cmath>
 
 namespace ray {
@@ -17,7 +18,10 @@ Image::Image(int width_value, int height_value)
       height(height_value),
       pixels(static_cast<std::size_t>(width_value * height_value * 3), 0) {}
 
-Image renderScene(const Scene& scene, const RenderSettings& settings) {
+Image renderScene(const Scene& scene,
+                  const RenderSettings& settings,
+                  RenderStats* stats) {
+    const auto started = std::chrono::steady_clock::now();
     Image image(scene.width, scene.height);
     std::size_t offset = 0;
     for (int y = 0; y < scene.height; ++y) {
@@ -27,7 +31,13 @@ Image renderScene(const Scene& scene, const RenderSettings& settings) {
                                           scene.height,
                                           x + 0.5,
                                           y + 0.5);
-            const Color color = traceRay(scene, ray, settings.maxDepth);
+            if (stats) {
+                ++stats->primaryRays;
+            }
+            const Color color = traceRay(scene,
+                                         ray,
+                                         settings.maxDepth,
+                                         stats);
             const Color clamped = clampColor(color);
             image.pixels[offset++] = static_cast<unsigned char>(
                 std::lround(clamped.x * 255.0));
@@ -37,6 +47,11 @@ Image renderScene(const Scene& scene, const RenderSettings& settings) {
                 std::lround(clamped.z * 255.0));
         }
     }
+    if (stats) {
+        const auto finished = std::chrono::steady_clock::now();
+        stats->renderMilliseconds =
+            std::chrono::duration<double, std::milli>(finished - started).count();
+    }
     return image;
 }
 
diff --git a/src/scene.cpp b/src/scene.cpp
index 7354196..ecaf08f 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -56,7 +56,8 @@ void Scene::addLight(const Light& light) {
 bool Scene::intersect(const Ray& ray,
                       double t_min,
                       double t_max,
-                      HitRecord& hit) const {
+                      HitRecord& hit,
+                      RenderStats* stats) const {
     bool found = false;
     double closest = t_max;
     HitRecord candidate;
@@ -65,6 +66,9 @@ bool Scene::intersect(const Ray& ray,
         if (!shape) {
             continue;
         }
+        if (stats) {
+            ++stats->primitiveTests;
+        }
         if (shape->intersect(ray, t_min, closest, candidate)) {
             found = true;
             closest = candidate.t;
diff --git a/src/shading.cpp b/src/shading.cpp
index 7827064..80d5f74 100644
--- a/src/shading.cpp
+++ b/src/shading.cpp
@@ -9,23 +9,27 @@ bool findNearestHit(const Scene& scene,
                     const Ray& ray,
                     HitRecord& hit,
                     double t_min,
-                    double t_max) {
-    return scene.intersect(ray, t_min, t_max, hit);
+                    double t_max,
+                    RenderStats* stats) {
+    return scene.intersect(ray, t_min, t_max, hit, stats);
 }
 
 bool isOccluded(const Scene& scene,
                 const Ray& shadow_ray,
-                double max_distance) {
+                double max_distance,
+                RenderStats* stats) {
     HitRecord ignored;
     return scene.intersect(shadow_ray,
                            kRayTMin,
                            std::max(kRayTMin, max_distance - kRayTMin),
-                           ignored);
+                           ignored,
+                           stats);
 }
 
 Color shadeHit(const Scene& scene,
                const HitRecord& hit,
-               const Ray& view_ray) {
+               const Ray& view_ray,
+               RenderStats* stats) {
     (void)view_ray;
 
     Color result =
@@ -46,9 +50,13 @@ Color shadeHit(const Scene& scene,
             continue;
         }
 
+        if (stats) {
+            ++stats->shadowRays;
+        }
         if (isOccluded(scene,
                        Ray(shadow_origin, light_direction),
-                       distance_to_light)) {
+                       distance_to_light,
+                       stats)) {
             continue;
         }
 
@@ -59,17 +67,21 @@ Color shadeHit(const Scene& scene,
     return clampColor(result);
 }
 
-Color traceRay(const Scene& scene, const Ray& ray, int max_depth) {
+Color traceRay(const Scene& scene,
+               const Ray& ray,
+               int max_depth,
+               RenderStats* stats) {
     (void)max_depth;
 
     HitRecord hit;
     if (!scene.intersect(ray,
                          kRayTMin,
                          std::numeric_limits<double>::infinity(),
-                         hit)) {
+                         hit,
+                         stats)) {
         return scene.background;
     }
-    return shadeHit(scene, hit, ray);
+    return shadeHit(scene, hit, ray, stats);
 }
 
 }  // namespace ray


## `refactor(render): 직렬 렌더링을 고정 tile 순회로 전환`

diff --git a/src/renderer.cpp b/src/renderer.cpp
index 7c3bab7..3677219 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -1,5 +1,6 @@
 #include "ray/renderer.hpp"
 
+#include <algorithm>
 #include <chrono>
 #include <cmath>
 #include <limits>
@@ -42,34 +43,57 @@ Image::Image(int width_value, int height_value)
 Image renderScene(const Scene& scene,
                   const RenderSettings& settings,
                   RenderStats* stats) {
+    constexpr int kTileSize = 16;
     const auto started = std::chrono::steady_clock::now();
     Image image(scene.width, scene.height);
     const CameraFrame camera_frame =
         buildCameraFrame(scene.camera, scene.width, scene.height);
-    std::size_t offset = 0;
-    for (int y = 0; y < scene.height; ++y) {
-        for (int x = 0; x < scene.width; ++x) {
-            const Ray ray = makeCameraRay(scene.camera,
-                                          camera_frame,
-                                          scene.width,
-                                          scene.height,
-                                          x + 0.5,
-                                          y + 0.5);
-            if (stats) {
-                ++stats->primaryRays;
+
+    const std::size_t tiles_x =
+        (static_cast<std::size_t>(scene.width) + kTileSize - 1) /
+        kTileSize;
+    const std::size_t tiles_y =
+        (static_cast<std::size_t>(scene.height) + kTileSize - 1) /
+        kTileSize;
+    const std::size_t tile_count = tiles_x * tiles_y;
+
+    for (std::size_t tile = 0; tile < tile_count; ++tile) {
+        const int start_x =
+            static_cast<int>((tile % tiles_x) * kTileSize);
+        const int start_y =
+            static_cast<int>((tile / tiles_x) * kTileSize);
+        const int end_x = std::min(start_x + kTileSize, scene.width);
+        const int end_y = std::min(start_y + kTileSize, scene.height);
+
+        for (int y = start_y; y < end_y; ++y) {
+            for (int x = start_x; x < end_x; ++x) {
+                const Ray ray = makeCameraRay(scene.camera,
+                                              camera_frame,
+                                              scene.width,
+                                              scene.height,
+                                              x + 0.5,
+                                              y + 0.5);
+                if (stats) {
+                    ++stats->primaryRays;
+                }
+                const Color color = traceRay(scene,
+                                             ray,
+                                             settings.maxDepth,
+                                             settings.accelMode,
+                                             stats);
+                const Color clamped = clampColor(color);
+                const std::size_t offset =
+                    (static_cast<std::size_t>(y) *
+                         static_cast<std::size_t>(scene.width) +
+                     static_cast<std::size_t>(x)) *
+                    3;
+                image.pixels[offset] = static_cast<unsigned char>(
+                    std::lround(clamped.x * 255.0));
+                image.pixels[offset + 1] = static_cast<unsigned char>(
+                    std::lround(clamped.y * 255.0));
+                image.pixels[offset + 2] = static_cast<unsigned char>(
+                    std::lround(clamped.z * 255.0));
             }
-            const Color color = traceRay(scene,
-                                         ray,
-                                         settings.maxDepth,
-                                         settings.accelMode,
-                                         stats);
-            const Color clamped = clampColor(color);
-            image.pixels[offset++] = static_cast<unsigned char>(
-                std::lround(clamped.x * 255.0));
-            image.pixels[offset++] = static_cast<unsigned char>(
-                std::lround(clamped.y * 255.0));
-            image.pixels[offset++] = static_cast<unsigned char>(
-                std::lround(clamped.z * 255.0));
         }
     }
     if (stats) {


## `feat(render): 원자적 tile 분배와 작업자 통계 병합 구현`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 948f7b5..ce88edd 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -2,6 +2,8 @@ cmake_minimum_required(VERSION 3.16)
 
 project(ray_scene_tracer LANGUAGES CXX)
 
+find_package(Threads REQUIRED)
+
 set(CMAKE_CXX_STANDARD 17)
 set(CMAKE_CXX_STANDARD_REQUIRED ON)
 set(CMAKE_CXX_EXTENSIONS OFF)
@@ -18,6 +20,7 @@ add_library(raycore
     src/shading.cpp
 )
 target_include_directories(raycore PUBLIC include)
+target_link_libraries(raycore PUBLIC Threads::Threads)
 
 if(MSVC)
     target_compile_options(raycore PRIVATE /W4)
diff --git a/src/renderer.cpp b/src/renderer.cpp
index 3677219..9cc16e0 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -1,10 +1,13 @@
 #include "ray/renderer.hpp"
 
 #include <algorithm>
+#include <atomic>
 #include <chrono>
 #include <cmath>
 #include <limits>
 #include <stdexcept>
+#include <thread>
+#include <vector>
 
 namespace ray {
 
@@ -56,47 +59,108 @@ Image renderScene(const Scene& scene,
         (static_cast<std::size_t>(scene.height) + kTileSize - 1) /
         kTileSize;
     const std::size_t tile_count = tiles_x * tiles_y;
+    unsigned int worker_count = std::thread::hardware_concurrency();
+    if (worker_count == 0) {
+        worker_count = 1;
+    }
+    worker_count = static_cast<unsigned int>(
+        std::min<std::size_t>(worker_count, tile_count));
+
+    struct alignas(64) WorkerStats {
+        RenderStats values;
+    };
+    std::vector<WorkerStats> worker_stats(worker_count);
+    std::atomic<std::size_t> next_tile{0};
+    std::vector<std::thread> workers;
+    workers.reserve(worker_count);
+
+    struct ThreadJoiner {
+        std::vector<std::thread>& workers;
+        std::atomic<std::size_t>& nextTile;
+        std::size_t tileCount;
 
-    for (std::size_t tile = 0; tile < tile_count; ++tile) {
-        const int start_x =
-            static_cast<int>((tile % tiles_x) * kTileSize);
-        const int start_y =
-            static_cast<int>((tile / tiles_x) * kTileSize);
-        const int end_x = std::min(start_x + kTileSize, scene.width);
-        const int end_y = std::min(start_y + kTileSize, scene.height);
-
-        for (int y = start_y; y < end_y; ++y) {
-            for (int x = start_x; x < end_x; ++x) {
-                const Ray ray = makeCameraRay(scene.camera,
-                                              camera_frame,
-                                              scene.width,
-                                              scene.height,
-                                              x + 0.5,
-                                              y + 0.5);
-                if (stats) {
-                    ++stats->primaryRays;
+        ~ThreadJoiner() {
+            nextTile.store(tileCount, std::memory_order_relaxed);
+            for (std::thread& worker : workers) {
+                if (worker.joinable()) {
+                    worker.join();
                 }
-                const Color color = traceRay(scene,
-                                             ray,
-                                             settings.maxDepth,
-                                             settings.accelMode,
-                                             stats);
-                const Color clamped = clampColor(color);
-                const std::size_t offset =
-                    (static_cast<std::size_t>(y) *
-                         static_cast<std::size_t>(scene.width) +
-                     static_cast<std::size_t>(x)) *
-                    3;
-                image.pixels[offset] = static_cast<unsigned char>(
-                    std::lround(clamped.x * 255.0));
-                image.pixels[offset + 1] = static_cast<unsigned char>(
-                    std::lround(clamped.y * 255.0));
-                image.pixels[offset + 2] = static_cast<unsigned char>(
-                    std::lround(clamped.z * 255.0));
             }
         }
+    };
+    const ThreadJoiner thread_joiner{
+        workers,
+        next_tile,
+        tile_count
+    };
+
+    for (unsigned int worker = 0; worker < worker_count; ++worker) {
+        workers.emplace_back([&, worker]() {
+            RenderStats& local = worker_stats[worker].values;
+            for (;;) {
+                const std::size_t tile =
+                    next_tile.fetch_add(1, std::memory_order_relaxed);
+                if (tile >= tile_count) {
+                    break;
+                }
+                const int start_x =
+                    static_cast<int>((tile % tiles_x) * kTileSize);
+                const int start_y =
+                    static_cast<int>((tile / tiles_x) * kTileSize);
+                const int end_x =
+                    std::min(start_x + kTileSize, scene.width);
+                const int end_y =
+                    std::min(start_y + kTileSize, scene.height);
+
+                for (int y = start_y; y < end_y; ++y) {
+                    for (int x = start_x; x < end_x; ++x) {
+                        const Ray ray =
+                            makeCameraRay(scene.camera,
+                                          camera_frame,
+                                          scene.width,
+                                          scene.height,
+                                          x + 0.5,
+                                          y + 0.5);
+                        ++local.primaryRays;
+                        const Color color =
+                            traceRay(scene,
+                                     ray,
+                                     settings.maxDepth,
+                                     settings.accelMode,
+                                     &local);
+                        const Color clamped = clampColor(color);
+                        const std::size_t offset =
+                            (static_cast<std::size_t>(y) *
+                                 static_cast<std::size_t>(scene.width) +
+                             static_cast<std::size_t>(x)) *
+                            3;
+                        image.pixels[offset] =
+                            static_cast<unsigned char>(
+                                std::lround(clamped.x * 255.0));
+                        image.pixels[offset + 1] =
+                            static_cast<unsigned char>(
+                                std::lround(clamped.y * 255.0));
+                        image.pixels[offset + 2] =
+                            static_cast<unsigned char>(
+                                std::lround(clamped.z * 255.0));
+                    }
+                }
+            }
+        });
+    }
+    for (std::thread& worker : workers) {
+        worker.join();
     }
+
     if (stats) {
+        *stats = RenderStats();
+        for (const WorkerStats& worker : worker_stats) {
+            stats->primaryRays += worker.values.primaryRays;
+            stats->secondaryRays += worker.values.secondaryRays;
+            stats->shadowRays += worker.values.shadowRays;
+            stats->primitiveTests += worker.values.primitiveTests;
+            stats->aabbTests += worker.values.aabbTests;
+        }
         const auto finished = std::chrono::steady_clock::now();
         stats->renderMilliseconds =
             std::chrono::duration<double, std::milli>(finished - started).count();


