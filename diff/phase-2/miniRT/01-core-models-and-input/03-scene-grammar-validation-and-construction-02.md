## `feat(parser): 구와 평면 지시어 지원`

diff --git a/src/parser.cpp b/src/parser.cpp
index 7d50452..1704f1c 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -3,6 +3,7 @@
 #include <cctype>
 #include <cmath>
 #include <limits>
+#include <memory>
 #include <sstream>
 #include <string>
 #include <vector>
@@ -128,7 +129,7 @@ double parseRatio(const std::string& token,
     return value;
 }
 
-[[maybe_unused]] double parsePositiveDouble(const std::string& token,
+double parsePositiveDouble(const std::string& token,
                            const std::string& source_name,
                            std::size_t line_number,
                            const std::string& field_name) {
@@ -369,6 +370,58 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            line_number,
                            "light color");
             scene.addLight(Light(position, brightness, color));
+        } else if (id == "sp") {
+            expectCount(tokens,
+                        4,
+                        source_name,
+                        line_number,
+                        "sp center diameter r,g,b");
+            const Vec3 center =
+                parseVec3(tokens[1],
+                          source_name,
+                          line_number,
+                          "sphere center");
+            const double diameter =
+                parsePositiveDouble(tokens[2],
+                                    source_name,
+                                    line_number,
+                                    "sphere diameter");
+            const Material material(
+                parseColor(tokens[3],
+                           source_name,
+                           line_number,
+                           "sphere color"));
+            scene.addShape(std::make_shared<Sphere>(
+                center,
+                diameter * 0.5,
+                material));
+        } else if (id == "pl") {
+            expectCount(tokens,
+                        4,
+                        source_name,
+                        line_number,
+                        "pl point normal r,g,b");
+            const Vec3 point =
+                parseVec3(tokens[1],
+                          source_name,
+                          line_number,
+                          "plane point");
+            const Vec3 normal =
+                parseVec3(tokens[2],
+                          source_name,
+                          line_number,
+                          "plane normal");
+            requireNonzeroVector(normal,
+                                 source_name,
+                                 line_number,
+                                 "plane normal");
+            const Material material(
+                parseColor(tokens[3],
+                           source_name,
+                           line_number,
+                           "plane color"));
+            scene.addShape(
+                std::make_shared<Plane>(point, normal, material));
         } else {
             throw ParseError(source_name,
                              line_number,


## `feat(parser): 원기둥 지시어 지원`

diff --git a/src/parser.cpp b/src/parser.cpp
index 1704f1c..e69d64c 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -422,6 +422,47 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            "plane color"));
             scene.addShape(
                 std::make_shared<Plane>(point, normal, material));
+        } else if (id == "cy") {
+            expectCount(tokens,
+                        6,
+                        source_name,
+                        line_number,
+                        "cy center axis diameter height r,g,b");
+            const Vec3 center =
+                parseVec3(tokens[1],
+                          source_name,
+                          line_number,
+                          "cylinder center");
+            const Vec3 axis =
+                parseVec3(tokens[2],
+                          source_name,
+                          line_number,
+                          "cylinder axis");
+            requireNonzeroVector(axis,
+                                 source_name,
+                                 line_number,
+                                 "cylinder axis");
+            const double diameter =
+                parsePositiveDouble(tokens[3],
+                                    source_name,
+                                    line_number,
+                                    "cylinder diameter");
+            const double height =
+                parsePositiveDouble(tokens[4],
+                                    source_name,
+                                    line_number,
+                                    "cylinder height");
+            const Material material(
+                parseColor(tokens[5],
+                           source_name,
+                           line_number,
+                           "cylinder color"));
+            scene.addShape(std::make_shared<Cylinder>(
+                center,
+                axis,
+                diameter * 0.5,
+                height,
+                material));
         } else {
             throw ParseError(source_name,
                              line_number,


## `feat(parser): 필수 지시어 검증과 입력 loader 완성`

diff --git a/include/ray/parser.hpp b/include/ray/parser.hpp
index 75099af..ce1a1e0 100644
--- a/include/ray/parser.hpp
+++ b/include/ray/parser.hpp
@@ -26,6 +26,11 @@ private:
 namespace parser {
 Scene parseScene(std::istream& input,
                  const std::string& source_name = "<stream>");
+Scene parseSceneText(const std::string& text,
+                     const std::string& source_name = "<text>");
+Scene parseSceneFile(const std::string& path);
 }  // namespace parser
 
+Scene loadScene(const std::string& path);
+
 }  // namespace ray
diff --git a/scenes/basic.rt b/scenes/basic.rt
new file mode 100644
index 0000000..fb4de21
--- /dev/null
+++ b/scenes/basic.rt
@@ -0,0 +1,8 @@
+# Minimal valid educational miniRT-style scene.
+R 640 360
+A 0.20 255,255,255
+C 0,0,0 0,0,1 60
+L 4,5,-3 0.85 255,255,255
+sp 0,0,5 2.0 220,70,60
+pl 0,-1.2,0 0,1,0 80,140,210
+cy 2,0,6 0,1,0 1.0 2.4 70,210,120
diff --git a/scenes/invalid.rt b/scenes/invalid.rt
new file mode 100644
index 0000000..3f942b1
--- /dev/null
+++ b/scenes/invalid.rt
@@ -0,0 +1,5 @@
+# Invalid on purpose: ambient ratio must be in the inclusive range [0.0, 1.0].
+R 320 200
+A 1.50 255,255,255
+C 0,0,0 0,0,1 60
+sp 0,0,5 2 255,0,0
diff --git a/src/parser.cpp b/src/parser.cpp
index e69d64c..02dc6f8 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -2,6 +2,7 @@
 
 #include <cctype>
 #include <cmath>
+#include <fstream>
 #include <limits>
 #include <memory>
 #include <sstream>
@@ -470,9 +471,39 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
         }
     }
 
+    if (!scene.hasResolution) {
+        throw ParseError(
+            source_name, 0, "missing R width height directive");
+    }
+    if (!scene.hasAmbient) {
+        throw ParseError(
+            source_name, 0, "missing A ratio r,g,b directive");
+    }
+    if (!scene.hasCamera) {
+        throw ParseError(
+            source_name, 0, "missing C pos dir fov directive");
+    }
     return scene;
 }
 
+Scene parseSceneText(const std::string& text,
+                     const std::string& source_name) {
+    std::istringstream input(text);
+    return parseScene(input, source_name);
+}
+
+Scene parseSceneFile(const std::string& path) {
+    std::ifstream input(path);
+    if (!input) {
+        throw ParseError(path, 0, "unable to open scene file");
+    }
+    return parseScene(input, path);
+}
+
 }  // namespace parser
 
+Scene loadScene(const std::string& path) {
+    return parser::parseSceneFile(path);
+}
+
 }  // namespace ray


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


## `fix(parser): 임계값 이하 방향 벡터 거부`

diff --git a/src/parser.cpp b/src/parser.cpp
index 02dc6f8..a2aba11 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -224,7 +224,7 @@ void requireNonzeroVector(const Vec3& value,
                           const std::string& source_name,
                           std::size_t line_number,
                           const std::string& field_name) {
-    if (value.isNearZero()) {
+    if (value.length() <= kEpsilon) {
         throw ParseError(source_name,
                          line_number,
                          field_name + " must not be the zero vector");


## `test(parser): 퇴화한 카메라와 원기둥 방향 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index 08b1f8e..a96d8d0 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -79,6 +79,36 @@ void testInvalidFixture() {
     require(rejected, "invalid scene fixture");
 }
 
+void testNearZeroDirections() {
+    const std::string prefix =
+        "R 8 8\n"
+        "A 0.1 255,255,255\n";
+    bool camera_rejected = false;
+    try {
+        (void)ray::parser::parseSceneText(
+            prefix +
+                "C 0,0,0 0.000001,0,0 60\n",
+            "small-camera-direction.rt");
+    } catch (const ray::ParseError&) {
+        camera_rejected = true;
+    }
+    require(camera_rejected,
+            "near-zero camera direction rejection");
+
+    bool cylinder_rejected = false;
+    try {
+        (void)ray::parser::parseSceneText(
+            prefix +
+                "C 0,0,0 0,0,1 60\n"
+                "cy 0,0,5 0.000001,0,0 1 2 255,0,0\n",
+            "small-cylinder-axis.rt");
+    } catch (const ray::ParseError&) {
+        cylinder_rejected = true;
+    }
+    require(cylinder_rejected,
+            "near-zero cylinder axis rejection");
+}
+
 void testOutput() {
     ray::Image image(2, 1);
     image.pixels = {255, 0, 16, 0, 127, 255};
@@ -105,6 +135,7 @@ int main() {
         testMath();
         testGeometry();
         testInvalidFixture();
+        testNearZeroDirections();
         testOutput();
         testRenderGolden();
     } catch (const std::exception& error) {


## `feat(scene): 가속 구조 소유권과 rebuild 경계 구성`

diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index a26cb1c..d9f996c 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -62,12 +62,19 @@ public:
 
     void addShape(std::unique_ptr<Shape> shape);
     void addLight(const Light& light);
+    void buildAcceleration();
+    bool accelerationReady() const;
     bool intersect(const Ray& ray,
                    double t_min,
                    double t_max,
                    HitRecord& hit,
                    AccelMode mode = AccelMode::Bvh,
                    RenderStats* stats = nullptr) const;
+
+private:
+    Bvh bvh_;
+    std::vector<std::uint32_t> unboundedIndices_;
+    bool accelerationReady_;
 };
 
 }  // namespace ray
diff --git a/src/parser.cpp b/src/parser.cpp
index 090fca8..2ab87cd 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -483,6 +483,7 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
         throw ParseError(
             source_name, 0, "missing C pos dir fov directive");
     }
+    scene.buildAcceleration();
     return scene;
 }
 
diff --git a/src/scene.cpp b/src/scene.cpp
index b7c5240..649037b 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -1,5 +1,8 @@
 #include "ray/scene.hpp"
 
+#include <optional>
+#include <utility>
+
 namespace ray {
 
 Material::Material() : albedo(1.0, 1.0, 1.0) {}
@@ -41,11 +44,17 @@ Scene::Scene()
       background(0.02, 0.03, 0.05),
       camera(),
       lights(),
-      shapes() {}
+      shapes(),
+      bvh_(),
+      unboundedIndices_(),
+      accelerationReady_(false) {}
 
 void Scene::addShape(std::unique_ptr<Shape> shape) {
     if (shape) {
         shapes.push_back(std::move(shape));
+        bvh_.clear();
+        unboundedIndices_.clear();
+        accelerationReady_ = false;
     }
 }
 
@@ -53,6 +62,35 @@ void Scene::addLight(const Light& light) {
     lights.push_back(light);
 }
 
+void Scene::buildAcceleration() {
+    std::vector<BvhPrimitive> bounded;
+    unboundedIndices_.clear();
+    bounded.reserve(shapes.size());
+    unboundedIndices_.reserve(shapes.size());
+
+    for (std::size_t index = 0; index < shapes.size(); ++index) {
+        if (!shapes[index]) {
+            continue;
+        }
+        const std::optional<Aabb> shape_bounds = shapes[index]->bounds();
+        if (shape_bounds && shape_bounds->isValid()) {
+            bounded.push_back(BvhPrimitive{
+                static_cast<std::uint32_t>(index),
+                *shape_bounds
+            });
+        } else {
+            unboundedIndices_.push_back(
+                static_cast<std::uint32_t>(index));
+        }
+    }
+    bvh_.build(std::move(bounded));
+    accelerationReady_ = true;
+}
+
+bool Scene::accelerationReady() const {
+    return accelerationReady_;
+}
+
 bool Scene::intersect(const Ray& ray,
                       double t_min,
                       double t_max,


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


