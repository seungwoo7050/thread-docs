# 작업량 계측과 재현 가능한 성능 근거

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


## `perf(benchmark): 조밀 장면 기준 workload 추가`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index a1415ea..6b28795 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -27,6 +27,9 @@ endif()
 add_executable(ray-scene-tracer src/main.cpp)
 target_link_libraries(ray-scene-tracer PRIVATE raycore)
 
+add_executable(ray-benchmark benchmarks/render_benchmark.cpp)
+target_link_libraries(ray-benchmark PRIVATE raycore)
+
 include(CTest)
 if(BUILD_TESTING)
     add_executable(ray-core-tests tests/core_tests.cpp)
diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
new file mode 100644
index 0000000..c48b10b
--- /dev/null
+++ b/benchmarks/render_benchmark.cpp
@@ -0,0 +1,73 @@
+#include "ray.hpp"
+
+#include <chrono>
+#include <iostream>
+#include <memory>
+#include <string>
+
+namespace {
+
+ray::Scene makeDenseScene() {
+    ray::Scene scene;
+    scene.width = 640;
+    scene.height = 360;
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
+
+    const ray::Material ground(ray::Color(0.35, 0.38, 0.42));
+    scene.addShape(std::make_shared<ray::Plane>(
+        ray::Vec3(0.0, -1.0, 0.0),
+        ray::Vec3(0.0, 1.0, 0.0),
+        ground));
+
+    for (int row = 0; row < 20; ++row) {
+        for (int column = 0; column < 20; ++column) {
+            const double x = (column - 9.5) * 1.05;
+            const double z = 2.0 + row * 1.05;
+            const double y = -0.45 + 0.18 * ((row + column) % 4);
+            const ray::Color color(
+                0.2 + 0.6 * static_cast<double>(column % 5) / 4.0,
+                0.2 + 0.6 * static_cast<double>(row % 5) / 4.0,
+                0.25 + 0.5 * static_cast<double>((row + column) % 5) / 4.0);
+            scene.addShape(std::make_shared<ray::Sphere>(
+                ray::Vec3(x, y, z),
+                0.42,
+                ray::Material(color)));
+        }
+    }
+    return scene;
+}
+
+struct Sample {
+    double milliseconds;
+    ray::RenderStats stats;
+    std::string checksum;
+};
+
+Sample render(const ray::Scene& scene) {
+    ray::RenderStats stats;
+    const ray::Image image =
+        ray::renderScene(scene, ray::RenderSettings(), &stats);
+    return Sample{stats.renderMilliseconds, stats, ray::checksumHex(image)};
+}
+
+}  // namespace
+
+int main() {
+    const Sample sample = render(makeDenseScene());
+    std::cout << sample.checksum << '\n';
+    return 0;
+}


## `perf(benchmark): 반복 측정과 결정성 보고 구성`

diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
index c48b10b..f9bbf7c 100644
--- a/benchmarks/render_benchmark.cpp
+++ b/benchmarks/render_benchmark.cpp
@@ -1,9 +1,13 @@
 #include "ray.hpp"
 
+#include <algorithm>
 #include <chrono>
+#include <iomanip>
 #include <iostream>
 #include <memory>
+#include <stdexcept>
 #include <string>
+#include <vector>
 
 namespace {
 
@@ -67,7 +71,39 @@ Sample render(const ray::Scene& scene) {
 }  // namespace
 
 int main() {
-    const Sample sample = render(makeDenseScene());
-    std::cout << sample.checksum << '\n';
+    const ray::Scene scene = makeDenseScene();
+    (void)render(scene);
+
+    std::vector<Sample> samples;
+    for (int iteration = 0; iteration < 5; ++iteration) {
+        samples.push_back(render(scene));
+    }
+    std::sort(samples.begin(),
+              samples.end(),
+              [](const Sample& left, const Sample& right) {
+                  return left.milliseconds < right.milliseconds;
+              });
+    const Sample& median = samples[samples.size() / 2];
+    for (const Sample& sample : samples) {
+        if (sample.checksum != median.checksum ||
+            sample.stats.primitiveTests != median.stats.primitiveTests) {
+            throw std::runtime_error(
+                "benchmark runs produced different results");
+        }
+    }
+
+    std::cout << std::fixed << std::setprecision(3)
+              << "{\n"
+              << "  \"scene\": \"dense-20x20\",\n"
+              << "  \"width\": 640,\n"
+              << "  \"height\": 360,\n"
+              << "  \"warmupRuns\": 1,\n"
+              << "  \"measuredRuns\": 5,\n"
+              << "  \"medianMilliseconds\": " << median.milliseconds << ",\n"
+              << "  \"primaryRays\": " << median.stats.primaryRays << ",\n"
+              << "  \"shadowRays\": " << median.stats.shadowRays << ",\n"
+              << "  \"primitiveTests\": " << median.stats.primitiveTests << ",\n"
+              << "  \"checksum\": \"" << median.checksum << "\"\n"
+              << "}\n";
     return 0;
 }


## `perf(benchmark): 선형 탐색과 BVH 작업량 비교`

diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
index e19d154..dcc8051 100644
--- a/benchmarks/render_benchmark.cpp
+++ b/benchmarks/render_benchmark.cpp
@@ -52,6 +52,7 @@ ray::Scene makeDenseScene() {
                 ray::Material(color)));
         }
     }
+    scene.buildAcceleration();
     return scene;
 }
 
@@ -61,36 +62,67 @@ struct Sample {
     std::string checksum;
 };
 
-Sample render(const ray::Scene& scene) {
+Sample render(const ray::Scene& scene, ray::AccelMode mode) {
     ray::RenderStats stats;
-    const ray::Image image =
-        ray::renderScene(scene, ray::RenderSettings(), &stats);
+    ray::RenderSettings settings;
+    settings.accelMode = mode;
+    const ray::Image image = ray::renderScene(scene, settings, &stats);
     return Sample{stats.renderMilliseconds, stats, ray::checksumHex(image)};
 }
 
-}  // namespace
-
-int main() {
-    const ray::Scene scene = makeDenseScene();
-    (void)render(scene);
+Sample measure(const ray::Scene& scene, ray::AccelMode mode) {
+    (void)render(scene, mode);
 
     std::vector<Sample> samples;
     for (int iteration = 0; iteration < 5; ++iteration) {
-        samples.push_back(render(scene));
+        samples.push_back(render(scene, mode));
     }
     std::sort(samples.begin(),
               samples.end(),
               [](const Sample& left, const Sample& right) {
                   return left.milliseconds < right.milliseconds;
               });
-    const Sample& median = samples[samples.size() / 2];
+    const Sample median = samples[samples.size() / 2];
     for (const Sample& sample : samples) {
         if (sample.checksum != median.checksum ||
-            sample.stats.primitiveTests != median.stats.primitiveTests) {
+            sample.stats.primitiveTests != median.stats.primitiveTests ||
+            sample.stats.aabbTests != median.stats.aabbTests) {
             throw std::runtime_error(
                 "benchmark runs produced different results");
         }
     }
+    return median;
+}
+
+void printResult(const std::string& name,
+                 const Sample& sample,
+                 bool trailing_comma) {
+    std::cout << "    \"" << name << "\": {\n"
+              << "      \"medianMilliseconds\": "
+              << sample.milliseconds << ",\n"
+              << "      \"primaryRays\": "
+              << sample.stats.primaryRays << ",\n"
+              << "      \"shadowRays\": "
+              << sample.stats.shadowRays << ",\n"
+              << "      \"aabbTests\": "
+              << sample.stats.aabbTests << ",\n"
+              << "      \"primitiveTests\": "
+              << sample.stats.primitiveTests << ",\n"
+              << "      \"checksum\": \""
+              << sample.checksum << "\"\n"
+              << "    }" << (trailing_comma ? "," : "") << "\n";
+}
+
+}  // namespace
+
+int main() {
+    const ray::Scene scene = makeDenseScene();
+    const Sample linear = measure(scene, ray::AccelMode::Linear);
+    const Sample bvh = measure(scene, ray::AccelMode::Bvh);
+    if (linear.checksum != bvh.checksum) {
+        throw std::runtime_error(
+            "linear and BVH renders produced different images");
+    }
 
     std::cout << std::fixed << std::setprecision(3)
               << "{\n"
@@ -99,11 +131,10 @@ int main() {
               << "  \"height\": 360,\n"
               << "  \"warmupRuns\": 1,\n"
               << "  \"measuredRuns\": 5,\n"
-              << "  \"medianMilliseconds\": " << median.milliseconds << ",\n"
-              << "  \"primaryRays\": " << median.stats.primaryRays << ",\n"
-              << "  \"shadowRays\": " << median.stats.shadowRays << ",\n"
-              << "  \"primitiveTests\": " << median.stats.primitiveTests << ",\n"
-              << "  \"checksum\": \"" << median.checksum << "\"\n"
+              << "  \"results\": {\n";
+    printResult("linear", linear, true);
+    printResult("bvh", bvh, false);
+    std::cout << "  }\n"
               << "}\n";
     return 0;
 }


## `perf(benchmark): 측정 schema와 가속 기준 검증 고정`

diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
index f446348..da3e834 100644
--- a/benchmarks/render_benchmark.cpp
+++ b/benchmarks/render_benchmark.cpp
@@ -86,6 +86,9 @@ Sample measure(const ray::Scene& scene, ray::AccelMode mode) {
     const Sample median = samples[samples.size() / 2];
     for (const Sample& sample : samples) {
         if (sample.checksum != median.checksum ||
+            sample.stats.primaryRays != median.stats.primaryRays ||
+            sample.stats.secondaryRays != median.stats.secondaryRays ||
+            sample.stats.shadowRays != median.stats.shadowRays ||
             sample.stats.primitiveTests != median.stats.primitiveTests ||
             sample.stats.aabbTests != median.stats.aabbTests) {
             throw std::runtime_error(
@@ -105,6 +108,8 @@ void printResult(const std::string& name,
               << sample.stats.primaryRays << ",\n"
               << "      \"shadowRays\": "
               << sample.stats.shadowRays << ",\n"
+              << "      \"secondaryRays\": "
+              << sample.stats.secondaryRays << ",\n"
               << "      \"aabbTests\": "
               << sample.stats.aabbTests << ",\n"
               << "      \"primitiveTests\": "
@@ -124,18 +129,41 @@ int main() {
         throw std::runtime_error(
             "linear and BVH renders produced different images");
     }
+    if (bvh.stats.primitiveTests * 4 >=
+        linear.stats.primitiveTests) {
+        throw std::runtime_error(
+            "BVH did not reduce primitive tests below 25 percent");
+    }
+    const double primitive_ratio =
+        static_cast<double>(bvh.stats.primitiveTests) /
+        static_cast<double>(linear.stats.primitiveTests);
+    const double speedup =
+        linear.milliseconds / bvh.milliseconds;
 
     std::cout << std::fixed << std::setprecision(3)
               << "{\n"
-              << "  \"scene\": \"dense-20x20\",\n"
-              << "  \"width\": 640,\n"
-              << "  \"height\": 360,\n"
-              << "  \"warmupRuns\": 1,\n"
-              << "  \"measuredRuns\": 5,\n"
+              << "  \"schemaVersion\": 1,\n"
+              << "  \"configuration\": {\n"
+              << "    \"scene\": \"dense-20x20\",\n"
+              << "    \"width\": 640,\n"
+              << "    \"height\": 360,\n"
+              << "    \"spheres\": 400,\n"
+              << "    \"planes\": 1,\n"
+              << "    \"lights\": 2,\n"
+              << "    \"threads\": 1,\n"
+              << "    \"maxDepth\": 4,\n"
+              << "    \"tileSize\": 16,\n"
+              << "    \"warmupRuns\": 1,\n"
+              << "    \"measuredRuns\": 5\n"
+              << "  },\n"
               << "  \"results\": {\n";
     printResult("linear", linear, true);
-    printResult("bvh", bvh, false);
-    std::cout << "  }\n"
+    printResult("bvh", bvh, true);
+    std::cout << "    \"primitiveTestRatio\": "
+              << primitive_ratio << ",\n"
+              << "    \"medianSpeedup\": "
+              << speedup << "\n"
+              << "  }\n"
               << "}\n";
     return 0;
 }


## `perf(benchmark): 참조 측정값 기록`

diff --git a/benchmarks/reference.json b/benchmarks/reference.json
new file mode 100644
index 0000000..6525ad7
--- /dev/null
+++ b/benchmarks/reference.json
@@ -0,0 +1,44 @@
+{
+  "schemaVersion": 1,
+  "configuration": {
+    "scene": "dense-20x20",
+    "width": 640,
+    "height": 360,
+    "spheres": 400,
+    "planes": 1,
+    "lights": 2,
+    "threads": 1,
+    "maxDepth": 4,
+    "tileSize": 16,
+    "warmupRuns": 1,
+    "measuredRuns": 5
+  },
+  "environment": {
+    "buildType": "Release",
+    "compiler": "AppleClang 17.0.0",
+    "architecture": "arm64",
+    "logicalThreads": 8
+  },
+  "results": {
+    "linear": {
+      "medianMilliseconds": 2326.770,
+      "primaryRays": 230400,
+      "shadowRays": 283078,
+      "secondaryRays": 0,
+      "aabbTests": 0,
+      "primitiveTests": 205904678,
+      "checksum": "f3f4cf26aca94dc1"
+    },
+    "bvh": {
+      "medianMilliseconds": 87.001,
+      "primaryRays": 230400,
+      "shadowRays": 283078,
+      "secondaryRays": 0,
+      "aabbTests": 1696156,
+      "primitiveTests": 904630,
+      "checksum": "f3f4cf26aca94dc1"
+    },
+    "primitiveTestRatio": 0.004,
+    "medianSpeedup": 26.744
+  }
+}
