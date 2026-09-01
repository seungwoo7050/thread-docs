## `fix(accel): 가속 구조의 도형 불변식 보호`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index a2171bd..443dc52 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -41,57 +41,71 @@ protected:
 
 class Sphere : public Shape {
 public:
-    Vec3 center;
-    double radius;
-
     Sphere(const Vec3& center_value,
            double radius_value,
            const Material& material_value);
 
+    const Vec3& center() const;
+    double radius() const;
+
     bool intersect(const Ray& ray,
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
     std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
+
+private:
+    Vec3 center_;
+    double radius_;
 };
 
 class Plane : public Shape {
 public:
-    Vec3 point;
-    Vec3 normal;
-
     Plane(const Vec3& point_value,
           const Vec3& normal_value,
           const Material& material_value);
 
+    const Vec3& point() const;
+    const Vec3& normal() const;
+
     bool intersect(const Ray& ray,
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
     std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
+
+private:
+    Vec3 point_;
+    Vec3 normal_;
 };
 
 class Cylinder : public Shape {
 public:
-    Vec3 center;
-    Vec3 axis;
-    double radius;
-    double height;
-
     Cylinder(const Vec3& center_value,
              const Vec3& axis_value,
              double radius_value,
              double height_value,
              const Material& material_value);
 
+    const Vec3& center() const;
+    const Vec3& axis() const;
+    double radius() const;
+    double height() const;
+
     bool intersect(const Ray& ray,
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
     std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
+
+private:
+    Vec3 center_;
+    Vec3 axis_;
+    double radius_;
+    double height_;
 };
 
 }  // namespace ray
diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index d9f996c..8eaa8b5 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -2,6 +2,7 @@
 
 #include "ray/geometry.hpp"
 
+#include <cstddef>
 #include <cstdint>
 #include <memory>
 #include <vector>
@@ -52,7 +53,6 @@ public:
     Color background;
     Camera camera;
     std::vector<Light> lights;
-    std::vector<std::unique_ptr<Shape>> shapes;
 
     Scene();
     Scene(const Scene&) = delete;
@@ -61,6 +61,8 @@ public:
     Scene& operator=(Scene&&) noexcept = default;
 
     void addShape(std::unique_ptr<Shape> shape);
+    std::size_t shapeCount() const;
+    const Shape& shapeAt(std::size_t index) const;
     void addLight(const Light& light);
     void buildAcceleration();
     bool accelerationReady() const;
@@ -72,6 +74,7 @@ public:
                    RenderStats* stats = nullptr) const;
 
 private:
+    std::vector<std::unique_ptr<Shape>> shapes_;
     Bvh bvh_;
     std::vector<std::uint32_t> unboundedIndices_;
     bool accelerationReady_;
diff --git a/src/geometry.cpp b/src/geometry.cpp
index d0feb84..2c37f7a 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -29,24 +29,34 @@ const Material& Shape::material() const {
 Sphere::Sphere(const Vec3& center_value,
                double radius_value,
                const Material& material_value)
-    : Shape(material_value), center(center_value), radius(radius_value) {}
+    : Shape(material_value),
+      center_(center_value),
+      radius_(radius_value) {}
+
+const Vec3& Sphere::center() const {
+    return center_;
+}
+
+double Sphere::radius() const {
+    return radius_;
+}
 
 bool Sphere::intersect(const Ray& ray,
                        double t_min,
                        double t_max,
                        HitRecord& hit) const {
-    if (radius <= kEpsilon) {
+    if (radius_ <= kEpsilon) {
         return false;
     }
 
-    const Vec3 oc = ray.origin - center;
+    const Vec3 oc = ray.origin - center_;
     const double a = dot(ray.direction, ray.direction);
     if (a <= kEpsilon) {
         return false;
     }
 
     const double half_b = dot(oc, ray.direction);
-    const double c = dot(oc, oc) - radius * radius;
+    const double c = dot(oc, oc) - radius_ * radius_;
     const double discriminant = half_b * half_b - a * c;
     if (discriminant < 0.0) {
         return false;
@@ -65,7 +75,7 @@ bool Sphere::intersect(const Ray& ray,
     hit.point = ray.at(root);
     hit.material = material_;
     hit.shape = this;
-    hit.setFaceNormal(ray, (hit.point - center) / radius);
+    hit.setFaceNormal(ray, (hit.point - center_) / radius_);
     return true;
 }
 
@@ -74,29 +84,39 @@ std::string Sphere::typeName() const {
 }
 
 std::optional<Aabb> Sphere::bounds() const {
-    const Vec3 extent(radius, radius, radius);
-    return Aabb(center - extent, center + extent);
+    const Vec3 extent(radius_, radius_, radius_);
+    return Aabb(center_ - extent, center_ + extent);
 }
 
 Plane::Plane(const Vec3& point_value,
              const Vec3& normal_value,
              const Material& material_value)
-    : Shape(material_value), point(point_value), normal(normalize(normal_value)) {}
+    : Shape(material_value),
+      point_(point_value),
+      normal_(normalize(normal_value)) {}
+
+const Vec3& Plane::point() const {
+    return point_;
+}
+
+const Vec3& Plane::normal() const {
+    return normal_;
+}
 
 bool Plane::intersect(const Ray& ray,
                       double t_min,
                       double t_max,
                       HitRecord& hit) const {
-    if (normal.isNearZero()) {
+    if (normal_.isNearZero()) {
         return false;
     }
 
-    const double denominator = dot(normal, ray.direction);
+    const double denominator = dot(normal_, ray.direction);
     if (std::fabs(denominator) <= kEpsilon) {
         return false;
     }
 
-    const double t = dot(point - ray.origin, normal) / denominator;
+    const double t = dot(point_ - ray.origin, normal_) / denominator;
     if (t < t_min || t > t_max) {
         return false;
     }
@@ -105,7 +125,7 @@ bool Plane::intersect(const Ray& ray,
     hit.point = ray.at(t);
     hit.material = material_;
     hit.shape = this;
-    hit.setFaceNormal(ray, normal);
+    hit.setFaceNormal(ray, normal_);
     return true;
 }
 
@@ -123,10 +143,26 @@ Cylinder::Cylinder(const Vec3& center_value,
                    double height_value,
                    const Material& material_value)
     : Shape(material_value),
-      center(center_value),
-      axis(normalize(axis_value)),
-      radius(radius_value),
-      height(height_value) {}
+      center_(center_value),
+      axis_(normalize(axis_value)),
+      radius_(radius_value),
+      height_(height_value) {}
+
+const Vec3& Cylinder::center() const {
+    return center_;
+}
+
+const Vec3& Cylinder::axis() const {
+    return axis_;
+}
+
+double Cylinder::radius() const {
+    return radius_;
+}
+
+double Cylinder::height() const {
+    return height_;
+}
 
 namespace {
 
@@ -190,23 +226,26 @@ bool Cylinder::intersect(const Ray& ray,
                          double t_min,
                          double t_max,
                          HitRecord& hit) const {
-    if (axis.isNearZero() || radius <= kEpsilon || height <= kEpsilon) {
+    if (axis_.isNearZero() ||
+        radius_ <= kEpsilon ||
+        height_ <= kEpsilon) {
         return false;
     }
 
     bool found = false;
     double closest = t_max;
-    const double half_height = height * 0.5;
-    const Vec3 oc = ray.origin - center;
-    const double direction_axis = dot(ray.direction, axis);
-    const double origin_axis = dot(oc, axis);
-    const Vec3 direction_perp = ray.direction - axis * direction_axis;
-    const Vec3 origin_perp = oc - axis * origin_axis;
+    const double half_height = height_ * 0.5;
+    const Vec3 oc = ray.origin - center_;
+    const double direction_axis = dot(ray.direction, axis_);
+    const double origin_axis = dot(oc, axis_);
+    const Vec3 direction_perp = ray.direction - axis_ * direction_axis;
+    const Vec3 origin_perp = oc - axis_ * origin_axis;
     const double a = dot(direction_perp, direction_perp);
 
     if (a > kEpsilon) {
         const double half_b = dot(direction_perp, origin_perp);
-        const double c = dot(origin_perp, origin_perp) - radius * radius;
+        const double c =
+            dot(origin_perp, origin_perp) - radius_ * radius_;
         const double discriminant = half_b * half_b - a * c;
         if (discriminant >= 0.0) {
             const double sqrt_discriminant = std::sqrt(discriminant);
@@ -220,13 +259,15 @@ bool Cylinder::intersect(const Ray& ray,
                     continue;
                 }
                 const Vec3 point = ray.at(root);
-                const double axial_distance = dot(point - center, axis);
+                const double axial_distance =
+                    dot(point - center_, axis_);
                 if (axial_distance < -half_height - kEpsilon ||
                     axial_distance > half_height + kEpsilon) {
                     continue;
                 }
                 const Vec3 outward_normal =
-                    normalize((point - center) - axis * axial_distance);
+                    normalize((point - center_) -
+                              axis_ * axial_distance);
                 if (outward_normal.isNearZero()) {
                     continue;
                 }
@@ -242,12 +283,12 @@ bool Cylinder::intersect(const Ray& ray,
         }
     }
 
-    const Vec3 top_center = center + axis * half_height;
-    const Vec3 bottom_center = center - axis * half_height;
+    const Vec3 top_center = center_ + axis_ * half_height;
+    const Vec3 bottom_center = center_ - axis_ * half_height;
     found = test_cylinder_cap(ray,
                               top_center,
-                              axis,
-                              radius,
+                              axis_,
+                              radius_,
                               material_,
                               this,
                               t_min,
@@ -255,8 +296,8 @@ bool Cylinder::intersect(const Ray& ray,
                               hit) || found;
     found = test_cylinder_cap(ray,
                               bottom_center,
-                              -axis,
-                              radius,
+                              -axis_,
+                              radius_,
                               material_,
                               this,
                               t_min,
@@ -271,24 +312,25 @@ std::string Cylinder::typeName() const {
 }
 
 std::optional<Aabb> Cylinder::bounds() const {
-    const double half_height = height * 0.5;
+    const double half_height = height_ * 0.5;
     const auto extent_for = [this, half_height](double axis_component) {
         const double absolute_axis = std::fabs(axis_component);
         const double radial =
             std::sqrt(std::max(0.0, 1.0 - axis_component * axis_component));
         const double side_extent =
-            absolute_axis * (half_height + kEpsilon) + radius * radial;
+            absolute_axis * (half_height + kEpsilon) +
+            radius_ * radial;
         const double cap_extent =
             absolute_axis * half_height +
-            std::sqrt(radius * radius + kEpsilon) * radial;
+            std::sqrt(radius_ * radius_ + kEpsilon) * radial;
         return std::max(side_extent, cap_extent);
     };
 
-    const Vec3 extent(extent_for(axis.x),
-                      extent_for(axis.y),
-                      extent_for(axis.z));
-    Vec3 minimum = center - extent;
-    Vec3 maximum = center + extent;
+    const Vec3 extent(extent_for(axis_.x),
+                      extent_for(axis_.y),
+                      extent_for(axis_.z));
+    Vec3 minimum = center_ - extent;
+    Vec3 maximum = center_ + extent;
     minimum.x = std::nextafter(
         minimum.x, -std::numeric_limits<double>::infinity());
     minimum.y = std::nextafter(
diff --git a/src/scene.cpp b/src/scene.cpp
index e11e9a5..98b8d94 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -48,20 +48,28 @@ Scene::Scene()
       background(0.02, 0.03, 0.05),
       camera(),
       lights(),
-      shapes(),
+      shapes_(),
       bvh_(),
       unboundedIndices_(),
       accelerationReady_(false) {}
 
 void Scene::addShape(std::unique_ptr<Shape> shape) {
     if (shape) {
-        shapes.push_back(std::move(shape));
+        shapes_.push_back(std::move(shape));
         bvh_.clear();
         unboundedIndices_.clear();
         accelerationReady_ = false;
     }
 }
 
+std::size_t Scene::shapeCount() const {
+    return shapes_.size();
+}
+
+const Shape& Scene::shapeAt(std::size_t index) const {
+    return *shapes_.at(index);
+}
+
 void Scene::addLight(const Light& light) {
     lights.push_back(light);
 }
@@ -69,14 +77,15 @@ void Scene::addLight(const Light& light) {
 void Scene::buildAcceleration() {
     std::vector<BvhPrimitive> bounded;
     unboundedIndices_.clear();
-    bounded.reserve(shapes.size());
-    unboundedIndices_.reserve(shapes.size());
+    bounded.reserve(shapes_.size());
+    unboundedIndices_.reserve(shapes_.size());
 
-    for (std::size_t index = 0; index < shapes.size(); ++index) {
-        if (!shapes[index]) {
+    for (std::size_t index = 0; index < shapes_.size(); ++index) {
+        if (!shapes_[index]) {
             continue;
         }
-        const std::optional<Aabb> shape_bounds = shapes[index]->bounds();
+        const std::optional<Aabb> shape_bounds =
+            shapes_[index]->bounds();
         if (shape_bounds && shape_bounds->isValid()) {
             bounded.push_back(BvhPrimitive{
                 static_cast<std::uint32_t>(index),
@@ -108,7 +117,7 @@ bool Scene::intersect(const Ray& ray,
 
     const auto test_shape =
         [&](std::uint32_t index) {
-            const std::unique_ptr<Shape>& shape = shapes[index];
+            const std::unique_ptr<Shape>& shape = shapes_[index];
             if (!shape) {
                 return;
             }
@@ -130,7 +139,7 @@ bool Scene::intersect(const Ray& ray,
         };
 
     if (mode == AccelMode::Linear || !accelerationReady_) {
-        for (std::size_t index = 0; index < shapes.size(); ++index) {
+        for (std::size_t index = 0; index < shapes_.size(); ++index) {
             test_shape(static_cast<std::uint32_t>(index));
         }
         return found;
diff --git a/tests/accel_tests.cpp b/tests/accel_tests.cpp
index a8d86d9..973a13b 100644
--- a/tests/accel_tests.cpp
+++ b/tests/accel_tests.cpp
@@ -111,7 +111,8 @@ void testEqualDistanceTie() {
         ray::Vec3(0.0, 0.0, 5.0),
         1.0,
         ray::Material(ray::Color(0.0, 1.0, 0.0))));
-    const ray::Shape* expected = scene.shapes.back().get();
+    const ray::Shape* expected =
+        &scene.shapeAt(scene.shapeCount() - 1);
     scene.buildAcceleration();
 
     ray::HitRecord linear_hit;
diff --git a/tests/material_tests.cpp b/tests/material_tests.cpp
index f514917..4920410 100644
--- a/tests/material_tests.cpp
+++ b/tests/material_tests.cpp
@@ -28,7 +28,7 @@ void testMaterialParsing() {
     const ray::Scene omitted = ray::parser::parseSceneText(
         scenePrefix() + "sp 0,0,5 2 255,0,0\n",
         "omitted.rt");
-    require(omitted.shapes[0]->material().type ==
+    require(omitted.shapeAt(0).material().type ==
                 ray::MaterialType::Diffuse,
             "omitted material defaults to diffuse");
 
@@ -38,11 +38,11 @@ void testMaterialParsing() {
             "pl 0,-1,0 0,1,0 255,255,255 diffuse\n"
             "cy 2,0,5 0,1,0 1 2 0,0,255 metal\n",
         "materials.rt");
-    require(explicit_types.shapes[0]->material().type ==
+    require(explicit_types.shapeAt(0).material().type ==
                 ray::MaterialType::Metal &&
-                explicit_types.shapes[1]->material().type ==
+                explicit_types.shapeAt(1).material().type ==
                     ray::MaterialType::Diffuse &&
-                explicit_types.shapes[2]->material().type ==
+                explicit_types.shapeAt(2).material().type ==
                     ray::MaterialType::Metal,
             "explicit material parsing");
 


## `test(accel): 장면 변경과 가속 상태 불변식 검증`

diff --git a/tests/accel_tests.cpp b/tests/accel_tests.cpp
index 973a13b..be73884 100644
--- a/tests/accel_tests.cpp
+++ b/tests/accel_tests.cpp
@@ -5,6 +5,8 @@
 #include <memory>
 #include <stdexcept>
 #include <string>
+#include <type_traits>
+#include <utility>
 
 namespace {
 
@@ -18,6 +20,38 @@ bool nearlyEqual(double left, double right, double epsilon = 1.0e-9) {
     return std::fabs(left - right) <= epsilon;
 }
 
+template <typename T, typename = void>
+struct ExposesShapeStorage : std::false_type {};
+
+template <typename T>
+struct ExposesShapeStorage<
+    T,
+    std::void_t<decltype(std::declval<T&>().shapes)>>
+    : std::true_type {};
+
+static_assert(!ExposesShapeStorage<ray::Scene>::value,
+              "Scene must not expose mutable shape storage");
+static_assert(
+    std::is_same<
+        decltype(std::declval<ray::Scene&>().shapeAt(0)),
+        const ray::Shape&>::value,
+    "Scene shape access must be read-only");
+static_assert(
+    !std::is_assignable<
+        decltype(std::declval<ray::Sphere&>().center()),
+        ray::Vec3>::value,
+    "sphere geometry must be read-only");
+static_assert(
+    !std::is_assignable<
+        decltype(std::declval<ray::Plane&>().normal()),
+        ray::Vec3>::value,
+    "plane geometry must be read-only");
+static_assert(
+    !std::is_assignable<
+        decltype(std::declval<ray::Cylinder&>().axis()),
+        ray::Vec3>::value,
+    "cylinder geometry must be read-only");
+
 void requireEquivalentHit(ray::Scene& scene,
                           const ray::Ray& ray_value,
                           const std::string& label) {
@@ -135,6 +169,48 @@ void testEqualDistanceTie() {
             "later primitive wins equal-distance tie");
 }
 
+void testShapeMutationBoundary() {
+    ray::Scene scene;
+    scene.addShape(std::make_unique<ray::Sphere>(
+        ray::Vec3(10.0, 0.0, 5.0),
+        1.0,
+        ray::Material(ray::Color(1.0, 0.0, 0.0))));
+    scene.buildAcceleration();
+    require(scene.accelerationReady(),
+            "initial acceleration state");
+
+    scene.addShape(std::make_unique<ray::Sphere>(
+        ray::Vec3(0.0, 0.0, 5.0),
+        1.0,
+        ray::Material(ray::Color(0.0, 1.0, 0.0))));
+    require(scene.shapeCount() == 2,
+            "shape mutation uses the Scene boundary");
+    require(!scene.accelerationReady(),
+            "shape addition invalidates acceleration");
+
+    const ray::Shape* expected = &scene.shapeAt(1);
+    const ray::Ray forward(
+        ray::Vec3(), ray::Vec3(0.0, 0.0, 1.0));
+    ray::HitRecord fallback_hit;
+    require(scene.intersect(forward,
+                            ray::kRayTMin,
+                            100.0,
+                            fallback_hit,
+                            ray::AccelMode::Bvh) &&
+                fallback_hit.shape == expected,
+            "invalidated BVH falls back to current geometry");
+
+    scene.buildAcceleration();
+    ray::HitRecord rebuilt_hit;
+    require(scene.intersect(forward,
+                            ray::kRayTMin,
+                            100.0,
+                            rebuilt_hit,
+                            ray::AccelMode::Bvh) &&
+                rebuilt_hit.shape == expected,
+            "rebuilt BVH indexes current geometry");
+}
+
 ray::Scene makeDenseScene() {
     ray::Scene scene;
     scene.width = 160;
@@ -213,6 +289,7 @@ int main() {
         testPlaneOnly();
         testCylinder();
         testEqualDistanceTie();
+        testShapeMutationBoundary();
         testDenseRender();
     } catch (const std::exception& error) {
         std::cerr << "acceleration regression failed: "
