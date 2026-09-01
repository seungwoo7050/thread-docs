# Hit 계약과 해석적 도형 교차

## `feat(geometry): hit와 도형 교차 계약 정의`

diff --git a/include/ray.hpp b/include/ray.hpp
index 329e1e0..4182ae9 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -1,4 +1,5 @@
 #pragma once
 
+#include "ray/geometry.hpp"
 #include "ray/material.hpp"
 #include "ray/math.hpp"
diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
new file mode 100644
index 0000000..f428e7b
--- /dev/null
+++ b/include/ray/geometry.hpp
@@ -0,0 +1,39 @@
+#pragma once
+
+#include "ray/material.hpp"
+
+#include <string>
+
+namespace ray {
+
+class Shape;
+
+struct HitRecord {
+    double t;
+    Vec3 point;
+    Vec3 normal;
+    Material material;
+    const Shape* shape;
+    bool frontFace;
+
+    HitRecord();
+    void setFaceNormal(const Ray& ray, const Vec3& outward_normal);
+};
+
+class Shape {
+public:
+    explicit Shape(const Material& material_value = Material());
+    virtual ~Shape() = default;
+
+    const Material& material() const;
+    virtual bool intersect(const Ray& ray,
+                           double t_min,
+                           double t_max,
+                           HitRecord& hit) const = 0;
+    virtual std::string typeName() const = 0;
+
+protected:
+    Material material_;
+};
+
+}  // namespace ray
diff --git a/src/geometry.cpp b/src/geometry.cpp
new file mode 100644
index 0000000..ce73729
--- /dev/null
+++ b/src/geometry.cpp
@@ -0,0 +1,26 @@
+#include "ray/geometry.hpp"
+
+#include <utility>
+
+namespace ray {
+
+HitRecord::HitRecord()
+    : t(0.0),
+      point(),
+      normal(),
+      material(),
+      shape(nullptr),
+      frontFace(true) {}
+
+void HitRecord::setFaceNormal(const Ray& ray, const Vec3& outward_normal) {
+    frontFace = dot(ray.direction, outward_normal) < 0.0;
+    normal = frontFace ? outward_normal : -outward_normal;
+}
+
+Shape::Shape(const Material& material_value) : material_(material_value) {}
+
+const Material& Shape::material() const {
+    return material_;
+}
+
+}  // namespace ray


## `feat(geometry): 구 교차 계산 구현`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index f428e7b..86657eb 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -36,4 +36,20 @@ protected:
     Material material_;
 };
 
+class Sphere : public Shape {
+public:
+    Vec3 center;
+    double radius;
+
+    Sphere(const Vec3& center_value,
+           double radius_value,
+           const Material& material_value);
+
+    bool intersect(const Ray& ray,
+                   double t_min,
+                   double t_max,
+                   HitRecord& hit) const override;
+    std::string typeName() const override;
+};
+
 }  // namespace ray
diff --git a/src/geometry.cpp b/src/geometry.cpp
index ce73729..9e3dc2c 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -1,5 +1,6 @@
 #include "ray/geometry.hpp"
 
+#include <cmath>
 #include <utility>
 
 namespace ray {
@@ -23,4 +24,51 @@ const Material& Shape::material() const {
     return material_;
 }
 
+Sphere::Sphere(const Vec3& center_value,
+               double radius_value,
+               const Material& material_value)
+    : Shape(material_value), center(center_value), radius(radius_value) {}
+
+bool Sphere::intersect(const Ray& ray,
+                       double t_min,
+                       double t_max,
+                       HitRecord& hit) const {
+    if (radius <= kEpsilon) {
+        return false;
+    }
+
+    const Vec3 oc = ray.origin - center;
+    const double a = dot(ray.direction, ray.direction);
+    if (a <= kEpsilon) {
+        return false;
+    }
+
+    const double half_b = dot(oc, ray.direction);
+    const double c = dot(oc, oc) - radius * radius;
+    const double discriminant = half_b * half_b - a * c;
+    if (discriminant < 0.0) {
+        return false;
+    }
+
+    const double sqrt_discriminant = std::sqrt(discriminant);
+    double root = (-half_b - sqrt_discriminant) / a;
+    if (root < t_min || root > t_max) {
+        root = (-half_b + sqrt_discriminant) / a;
+        if (root < t_min || root > t_max) {
+            return false;
+        }
+    }
+
+    hit.t = root;
+    hit.point = ray.at(root);
+    hit.material = material_;
+    hit.shape = this;
+    hit.setFaceNormal(ray, (hit.point - center) / radius);
+    return true;
+}
+
+std::string Sphere::typeName() const {
+    return "sphere";
+}
+
 }  // namespace ray


## `feat(geometry): 평면 교차 계산 구현`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index 86657eb..a03e7ba 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -52,4 +52,20 @@ public:
     std::string typeName() const override;
 };
 
+class Plane : public Shape {
+public:
+    Vec3 point;
+    Vec3 normal;
+
+    Plane(const Vec3& point_value,
+          const Vec3& normal_value,
+          const Material& material_value);
+
+    bool intersect(const Ray& ray,
+                   double t_min,
+                   double t_max,
+                   HitRecord& hit) const override;
+    std::string typeName() const override;
+};
+
 }  // namespace ray
diff --git a/src/geometry.cpp b/src/geometry.cpp
index 9e3dc2c..74588aa 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -71,4 +71,39 @@ std::string Sphere::typeName() const {
     return "sphere";
 }
 
+Plane::Plane(const Vec3& point_value,
+             const Vec3& normal_value,
+             const Material& material_value)
+    : Shape(material_value), point(point_value), normal(normalize(normal_value)) {}
+
+bool Plane::intersect(const Ray& ray,
+                      double t_min,
+                      double t_max,
+                      HitRecord& hit) const {
+    if (normal.isNearZero()) {
+        return false;
+    }
+
+    const double denominator = dot(normal, ray.direction);
+    if (std::fabs(denominator) <= kEpsilon) {
+        return false;
+    }
+
+    const double t = dot(point - ray.origin, normal) / denominator;
+    if (t < t_min || t > t_max) {
+        return false;
+    }
+
+    hit.t = t;
+    hit.point = ray.at(t);
+    hit.material = material_;
+    hit.shape = this;
+    hit.setFaceNormal(ray, normal);
+    return true;
+}
+
+std::string Plane::typeName() const {
+    return "plane";
+}
+
 }  // namespace ray


## `feat(geometry): 유한 원기둥 옆면 교차 구현`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index a03e7ba..2daf96b 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -68,4 +68,24 @@ public:
     std::string typeName() const override;
 };
 
+class Cylinder : public Shape {
+public:
+    Vec3 center;
+    Vec3 axis;
+    double radius;
+    double height;
+
+    Cylinder(const Vec3& center_value,
+             const Vec3& axis_value,
+             double radius_value,
+             double height_value,
+             const Material& material_value);
+
+    bool intersect(const Ray& ray,
+                   double t_min,
+                   double t_max,
+                   HitRecord& hit) const override;
+    std::string typeName() const override;
+};
+
 }  // namespace ray
diff --git a/src/geometry.cpp b/src/geometry.cpp
index 74588aa..2050a7e 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -106,4 +106,76 @@ std::string Plane::typeName() const {
     return "plane";
 }
 
+Cylinder::Cylinder(const Vec3& center_value,
+                   const Vec3& axis_value,
+                   double radius_value,
+                   double height_value,
+                   const Material& material_value)
+    : Shape(material_value),
+      center(center_value),
+      axis(normalize(axis_value)),
+      radius(radius_value),
+      height(height_value) {}
+
+bool Cylinder::intersect(const Ray& ray,
+                         double t_min,
+                         double t_max,
+                         HitRecord& hit) const {
+    if (axis.isNearZero() || radius <= kEpsilon || height <= kEpsilon) {
+        return false;
+    }
+
+    bool found = false;
+    double closest = t_max;
+    const double half_height = height * 0.5;
+    const Vec3 oc = ray.origin - center;
+    const double direction_axis = dot(ray.direction, axis);
+    const double origin_axis = dot(oc, axis);
+    const Vec3 direction_perp = ray.direction - axis * direction_axis;
+    const Vec3 origin_perp = oc - axis * origin_axis;
+    const double a = dot(direction_perp, direction_perp);
+
+    if (a > kEpsilon) {
+        const double half_b = dot(direction_perp, origin_perp);
+        const double c = dot(origin_perp, origin_perp) - radius * radius;
+        const double discriminant = half_b * half_b - a * c;
+        if (discriminant >= 0.0) {
+            const double sqrt_discriminant = std::sqrt(discriminant);
+            const double roots[2] = {
+                (-half_b - sqrt_discriminant) / a,
+                (-half_b + sqrt_discriminant) / a
+            };
+
+            for (double root : roots) {
+                if (root < t_min || root > closest) {
+                    continue;
+                }
+                const Vec3 point = ray.at(root);
+                const double axial_distance = dot(point - center, axis);
+                if (axial_distance < -half_height - kEpsilon ||
+                    axial_distance > half_height + kEpsilon) {
+                    continue;
+                }
+                const Vec3 outward_normal =
+                    normalize((point - center) - axis * axial_distance);
+                if (outward_normal.isNearZero()) {
+                    continue;
+                }
+                hit.t = root;
+                hit.point = point;
+                hit.material = material_;
+                hit.shape = this;
+                hit.setFaceNormal(ray, outward_normal);
+                closest = root;
+                found = true;
+            }
+        }
+    }
+    return found;
+}
+
+std::string Cylinder::typeName() const {
+    return "cylinder";
+}
+
 }  // namespace ray


## `feat(geometry): 원기둥 cap과 최근접 hit 선택 완성`

diff --git a/src/geometry.cpp b/src/geometry.cpp
index 2050a7e..19a4f3c 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -117,6 +117,64 @@ Cylinder::Cylinder(const Vec3& center_value,
       radius(radius_value),
       height(height_value) {}
 
+namespace {
+
+bool update_hit_if_closer(const Ray& ray,
+                          const Material& material,
+                          const Shape* shape,
+                          double t,
+                          const Vec3& outward_normal,
+                          double t_min,
+                          double& closest,
+                          HitRecord& hit) {
+    if (t < t_min || t > closest) {
+        return false;
+    }
+    hit.t = t;
+    hit.point = ray.at(t);
+    hit.material = material;
+    hit.shape = shape;
+    hit.setFaceNormal(ray, outward_normal);
+    closest = t;
+    return true;
+}
+
+bool test_cylinder_cap(const Ray& ray,
+                       const Vec3& cap_center,
+                       const Vec3& outward_normal,
+                       double radius,
+                       const Material& material,
+                       const Shape* shape,
+                       double t_min,
+                       double& closest,
+                       HitRecord& hit) {
+    const double denominator = dot(outward_normal, ray.direction);
+    if (std::fabs(denominator) <= kEpsilon) {
+        return false;
+    }
+
+    const double t = dot(cap_center - ray.origin, outward_normal) / denominator;
+    if (t < t_min || t > closest) {
+        return false;
+    }
+
+    const Vec3 point = ray.at(t);
+    if ((point - cap_center).lengthSquared() > radius * radius + kEpsilon) {
+        return false;
+    }
+
+    return update_hit_if_closer(ray,
+                                material,
+                                shape,
+                                t,
+                                outward_normal,
+                                t_min,
+                                closest,
+                                hit);
+}
+
+}  // namespace
+
 bool Cylinder::intersect(const Ray& ray,
                          double t_min,
                          double t_max,
@@ -161,16 +219,39 @@ bool Cylinder::intersect(const Ray& ray,
                 if (outward_normal.isNearZero()) {
                     continue;
                 }
-                hit.t = root;
-                hit.point = point;
-                hit.material = material_;
-                hit.shape = this;
-                hit.setFaceNormal(ray, outward_normal);
-                closest = root;
-                found = true;
+                found = update_hit_if_closer(ray,
+                                             material_,
+                                             this,
+                                             root,
+                                             outward_normal,
+                                             t_min,
+                                             closest,
+                                             hit) || found;
             }
         }
     }
+
+    const Vec3 top_center = center + axis * half_height;
+    const Vec3 bottom_center = center - axis * half_height;
+    found = test_cylinder_cap(ray,
+                              top_center,
+                              axis,
+                              radius,
+                              material_,
+                              this,
+                              t_min,
+                              closest,
+                              hit) || found;
+    found = test_cylinder_cap(ray,
+                              bottom_center,
+                              -axis,
+                              radius,
+                              material_,
+                              this,
+                              t_min,
+                              closest,
+                              hit) || found;
+
     return found;
 }
 


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
