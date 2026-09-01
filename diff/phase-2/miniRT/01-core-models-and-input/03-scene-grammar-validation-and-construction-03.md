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
