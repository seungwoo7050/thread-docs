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


## `feat(accel): 결정적 BVH 최근접 순회 구현`

diff --git a/src/scene.cpp b/src/scene.cpp
index 649037b..c816dbf 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -1,5 +1,6 @@
 #include "ray/scene.hpp"
 
+#include <limits>
 #include <optional>
 #include <utility>
 
@@ -97,7 +98,6 @@ bool Scene::intersect(const Ray& ray,
                       HitRecord& hit,
                       AccelMode mode,
                       RenderStats* stats) const {
-    (void)mode;
     bool found = false;
     double closest = t_max;
     std::uint32_t best_index = 0;
@@ -126,8 +126,83 @@ bool Scene::intersect(const Ray& ray,
             }
         };
 
-    for (std::size_t index = 0; index < shapes.size(); ++index) {
-        test_shape(static_cast<std::uint32_t>(index));
+    if (mode == AccelMode::Linear || !accelerationReady_) {
+        for (std::size_t index = 0; index < shapes.size(); ++index) {
+            test_shape(static_cast<std::uint32_t>(index));
+        }
+        return found;
+    }
+
+    struct StackEntry {
+        std::uint32_t node;
+        double entry;
+    };
+    std::vector<StackEntry> stack;
+    const std::vector<BvhNode>& nodes = bvh_.nodes();
+    const std::vector<std::uint32_t>& indices =
+        bvh_.primitiveIndices();
+    if (!nodes.empty()) {
+        double root_entry = t_min;
+        if (stats) {
+            ++stats->aabbTests;
+        }
+        if (nodes[0].bounds.intersect(
+                ray, t_min, closest, &root_entry)) {
+            stack.push_back(StackEntry{0, root_entry});
+        }
+    }
+
+    while (!stack.empty()) {
+        const StackEntry current = stack.back();
+        stack.pop_back();
+        if (current.entry > closest) {
+            continue;
+        }
+
+        const BvhNode& node = nodes[current.node];
+        if (node.isLeaf()) {
+            for (std::uint32_t offset = 0;
+                 offset < node.count;
+                 ++offset) {
+                test_shape(indices[node.first + offset]);
+            }
+            continue;
+        }
+
+        double left_entry = t_min;
+        double right_entry = t_min;
+        if (stats) {
+            stats->aabbTests += 2;
+        }
+        const bool hit_left = nodes[node.left].bounds.intersect(
+            ray, t_min, closest, &left_entry);
+        const bool hit_right = nodes[node.right].bounds.intersect(
+            ray, t_min, closest, &right_entry);
+
+        if (hit_left && hit_right) {
+            const bool left_first =
+                left_entry < right_entry ||
+                (left_entry == right_entry &&
+                 node.left < node.right);
+            const StackEntry near_entry =
+                left_first
+                    ? StackEntry{node.left, left_entry}
+                    : StackEntry{node.right, right_entry};
+            const StackEntry far_entry =
+                left_first
+                    ? StackEntry{node.right, right_entry}
+                    : StackEntry{node.left, left_entry};
+            stack.push_back(far_entry);
+            stack.push_back(near_entry);
+        } else if (hit_left) {
+            stack.push_back(StackEntry{node.left, left_entry});
+        } else if (hit_right) {
+            stack.push_back(StackEntry{node.right, right_entry});
+        }
+    }
+
+    for (std::uint32_t index : unboundedIndices_) {
+        test_shape(index);
     }
     return found;
 }


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


