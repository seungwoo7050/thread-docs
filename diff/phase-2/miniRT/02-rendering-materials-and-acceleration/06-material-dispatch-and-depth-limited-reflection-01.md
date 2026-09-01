# 재질 분기와 깊이 제한 반사

## `feat(material): diffuse 재질 값 모델 추가`

diff --git a/include/ray.hpp b/include/ray.hpp
index e1b9561..329e1e0 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -1,3 +1,4 @@
 #pragma once
 
+#include "ray/material.hpp"
 #include "ray/math.hpp"
diff --git a/include/ray/material.hpp b/include/ray/material.hpp
new file mode 100644
index 0000000..c0d0f2d
--- /dev/null
+++ b/include/ray/material.hpp
@@ -0,0 +1,14 @@
+#pragma once
+
+#include "ray/math.hpp"
+
+namespace ray {
+
+struct Material {
+    Color albedo;
+
+    Material();
+    explicit Material(const Color& color);
+};
+
+}  // namespace ray
diff --git a/src/scene.cpp b/src/scene.cpp
new file mode 100644
index 0000000..e87ade9
--- /dev/null
+++ b/src/scene.cpp
@@ -0,0 +1,9 @@
+#include "ray/material.hpp"
+
+namespace ray {
+
+Material::Material() : albedo(1.0, 1.0, 1.0) {}
+
+Material::Material(const Color& color) : albedo(color) {}
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


## `feat(material): metal 모델과 깊이 제한 반사 구현`

diff --git a/include/ray/material.hpp b/include/ray/material.hpp
index c0d0f2d..132c9a7 100644
--- a/include/ray/material.hpp
+++ b/include/ray/material.hpp
@@ -4,11 +4,19 @@
 
 namespace ray {
 
+enum class MaterialType {
+    Diffuse,
+    Metal
+};
+
 struct Material {
     Color albedo;
+    MaterialType type;
 
     Material();
-    explicit Material(const Color& color);
+    explicit Material(
+        const Color& color,
+        MaterialType type_value = MaterialType::Diffuse);
 };
 
 }  // namespace ray
diff --git a/src/scene.cpp b/src/scene.cpp
index c816dbf..e11e9a5 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -6,9 +6,12 @@
 
 namespace ray {
 
-Material::Material() : albedo(1.0, 1.0, 1.0) {}
+Material::Material()
+    : albedo(1.0, 1.0, 1.0),
+      type(MaterialType::Diffuse) {}
 
-Material::Material(const Color& color) : albedo(color) {}
+Material::Material(const Color& color, MaterialType type_value)
+    : albedo(color), type(type_value) {}
 
 Light::Light()
     : position(), brightness(1.0), color(1.0, 1.0, 1.0) {}
diff --git a/src/shading.cpp b/src/shading.cpp
index e64416a..2ccc063 100644
--- a/src/shading.cpp
+++ b/src/shading.cpp
@@ -88,6 +88,26 @@ Color traceRay(const Scene& scene,
                          stats)) {
         return scene.background;
     }
+    if (hit.material.type == MaterialType::Metal) {
+        if (max_depth <= 0) {
+            return Color();
+        }
+        const Vec3 reflected_direction =
+            ray.direction -
+            hit.normal * (2.0 * dot(ray.direction, hit.normal));
+        const Ray reflected_ray(
+            hit.point + hit.normal * kRayTMin,
+            reflected_direction);
+        if (stats) {
+            ++stats->secondaryRays;
+        }
+        return hit.material.albedo *
+               traceRay(scene,
+                        reflected_ray,
+                        max_depth - 1,
+                        mode,
+                        stats);
+    }
     return shadeHit(scene, hit, ray, mode, stats);
 }
 


## `feat(parser): 선택적 도형 재질 문법 추가`

diff --git a/src/parser.cpp b/src/parser.cpp
index 2ab87cd..b2a0bbd 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -78,6 +78,21 @@ void expectCount(const std::vector<std::string>& tokens,
     }
 }
 
+void expectCountEither(const std::vector<std::string>& tokens,
+                       std::size_t first,
+                       std::size_t second,
+                       const std::string& source_name,
+                       std::size_t line_number,
+                       const std::string& form) {
+    if (tokens.size() != first && tokens.size() != second) {
+        std::ostringstream message;
+        message << "expected " << form << " with "
+                << first - 1 << " or " << second - 1
+                << " argument(s), got " << tokens.size() - 1;
+        throw ParseError(source_name, line_number, message.str());
+    }
+}
+
 double parseDoubleToken(const std::string& token,
                         const std::string& source_name,
                         std::size_t line_number,
@@ -209,6 +224,25 @@ Color parseColor(const std::string& token,
                  static_cast<double>(channels[2]) / 255.0);
 }
 
+MaterialType parseMaterialType(const std::vector<std::string>& tokens,
+                               std::size_t base_count,
+                               const std::string& source_name,
+                               std::size_t line_number) {
+    if (tokens.size() == base_count) {
+        return MaterialType::Diffuse;
+    }
+    const std::string& token = tokens.back();
+    if (token == "diffuse") {
+        return MaterialType::Diffuse;
+    }
+    if (token == "metal") {
+        return MaterialType::Metal;
+    }
+    throw ParseError(source_name,
+                     line_number,
+                     "unknown material '" + token + "'");
+}
+
 void rejectDuplicate(bool already_seen,
                      const std::string& source_name,
                      std::size_t line_number,
@@ -372,11 +406,12 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            "light color");
             scene.addLight(Light(position, brightness, color));
         } else if (id == "sp") {
-            expectCount(tokens,
-                        4,
-                        source_name,
-                        line_number,
-                        "sp center diameter r,g,b");
+            expectCountEither(tokens,
+                              4,
+                              5,
+                              source_name,
+                              line_number,
+                              "sp center diameter r,g,b [material]");
             const Vec3 center =
                 parseVec3(tokens[1],
                           source_name,
@@ -391,17 +426,20 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                 parseColor(tokens[3],
                            source_name,
                            line_number,
-                           "sphere color"));
+                           "sphere color"),
+                parseMaterialType(
+                    tokens, 4, source_name, line_number));
             scene.addShape(std::make_unique<Sphere>(
                 center,
                 diameter * 0.5,
                 material));
         } else if (id == "pl") {
-            expectCount(tokens,
-                        4,
-                        source_name,
-                        line_number,
-                        "pl point normal r,g,b");
+            expectCountEither(tokens,
+                              4,
+                              5,
+                              source_name,
+                              line_number,
+                              "pl point normal r,g,b [material]");
             const Vec3 point =
                 parseVec3(tokens[1],
                           source_name,
@@ -420,15 +458,19 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                 parseColor(tokens[3],
                            source_name,
                            line_number,
-                           "plane color"));
+                           "plane color"),
+                parseMaterialType(
+                    tokens, 4, source_name, line_number));
             scene.addShape(
                 std::make_unique<Plane>(point, normal, material));
         } else if (id == "cy") {
-            expectCount(tokens,
-                        6,
-                        source_name,
-                        line_number,
-                        "cy center axis diameter height r,g,b");
+            expectCountEither(
+                tokens,
+                6,
+                7,
+                source_name,
+                line_number,
+                "cy center axis diameter height r,g,b [material]");
             const Vec3 center =
                 parseVec3(tokens[1],
                           source_name,
@@ -457,7 +499,9 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                 parseColor(tokens[5],
                            source_name,
                            line_number,
-                           "cylinder color"));
+                           "cylinder color"),
+                parseMaterialType(
+                    tokens, 6, source_name, line_number));
             scene.addShape(std::make_unique<Cylinder>(
                 center,
                 axis,


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


