# AABB 교차와 보수적 도형 경계

## `feat(accel): AABB 값과 결합 연산 구현`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 6b28795..273d7ba 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -7,6 +7,7 @@ set(CMAKE_CXX_STANDARD_REQUIRED ON)
 set(CMAKE_CXX_EXTENSIONS OFF)
 
 add_library(raycore
+    src/accel.cpp
     src/camera.cpp
     src/geometry.cpp
     src/math.cpp
diff --git a/include/ray.hpp b/include/ray.hpp
index 3432841..b6cc828 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -1,5 +1,6 @@
 #pragma once
 
+#include "ray/accel.hpp"
 #include "ray/camera.hpp"
 #include "ray/geometry.hpp"
 #include "ray/material.hpp"
diff --git a/include/ray/accel.hpp b/include/ray/accel.hpp
new file mode 100644
index 0000000..cd5c479
--- /dev/null
+++ b/include/ray/accel.hpp
@@ -0,0 +1,20 @@
+#pragma once
+
+#include "ray/math.hpp"
+
+namespace ray {
+
+struct Aabb {
+    Vec3 minimum;
+    Vec3 maximum;
+
+    Aabb();
+    Aabb(const Vec3& minimum_value, const Vec3& maximum_value);
+
+    bool isValid() const;
+    Vec3 centroid() const;
+};
+
+Aabb surroundingBox(const Aabb& left, const Aabb& right);
+
+}  // namespace ray
diff --git a/src/accel.cpp b/src/accel.cpp
new file mode 100644
index 0000000..e87e8e1
--- /dev/null
+++ b/src/accel.cpp
@@ -0,0 +1,39 @@
+#include "ray/accel.hpp"
+
+#include <algorithm>
+#include <limits>
+
+namespace ray {
+
+Aabb::Aabb()
+    : minimum(std::numeric_limits<double>::infinity(),
+              std::numeric_limits<double>::infinity(),
+              std::numeric_limits<double>::infinity()),
+      maximum(-std::numeric_limits<double>::infinity(),
+              -std::numeric_limits<double>::infinity(),
+              -std::numeric_limits<double>::infinity()) {}
+
+Aabb::Aabb(const Vec3& minimum_value, const Vec3& maximum_value)
+    : minimum(minimum_value), maximum(maximum_value) {}
+
+bool Aabb::isValid() const {
+    return minimum.x <= maximum.x &&
+           minimum.y <= maximum.y &&
+           minimum.z <= maximum.z;
+}
+
+Vec3 Aabb::centroid() const {
+    return (minimum + maximum) * 0.5;
+}
+
+Aabb surroundingBox(const Aabb& left, const Aabb& right) {
+    return Aabb(
+        Vec3(std::min(left.minimum.x, right.minimum.x),
+             std::min(left.minimum.y, right.minimum.y),
+             std::min(left.minimum.z, right.minimum.z)),
+        Vec3(std::max(left.maximum.x, right.maximum.x),
+             std::max(left.maximum.y, right.maximum.y),
+             std::max(left.maximum.z, right.maximum.z)));
+}
+
+}  // namespace ray


## `feat(accel): ray-box slab 교차 구현`

diff --git a/include/ray/accel.hpp b/include/ray/accel.hpp
index cd5c479..dee14a7 100644
--- a/include/ray/accel.hpp
+++ b/include/ray/accel.hpp
@@ -13,6 +13,10 @@ struct Aabb {
 
     bool isValid() const;
     Vec3 centroid() const;
+    bool intersect(const Ray& ray,
+                   double t_min,
+                   double t_max,
+                   double* entry = nullptr) const;
 };
 
 Aabb surroundingBox(const Aabb& left, const Aabb& right);
diff --git a/src/accel.cpp b/src/accel.cpp
index e87e8e1..9b44e41 100644
--- a/src/accel.cpp
+++ b/src/accel.cpp
@@ -1,10 +1,25 @@
 #include "ray/accel.hpp"
 
 #include <algorithm>
+#include <cmath>
 #include <limits>
 
 namespace ray {
 
+namespace {
+
+double component(const Vec3& value, int axis) {
+    if (axis == 0) {
+        return value.x;
+    }
+    if (axis == 1) {
+        return value.y;
+    }
+    return value.z;
+}
+
+}  // namespace
+
 Aabb::Aabb()
     : minimum(std::numeric_limits<double>::infinity(),
               std::numeric_limits<double>::infinity(),
@@ -26,6 +41,47 @@ Vec3 Aabb::centroid() const {
     return (minimum + maximum) * 0.5;
 }
 
+bool Aabb::intersect(const Ray& ray,
+                     double t_min,
+                     double t_max,
+                     double* entry) const {
+    if (!isValid()) {
+        return false;
+    }
+
+    double near_value = t_min;
+    double far_value = t_max;
+    for (int axis = 0; axis < 3; ++axis) {
+        const double origin = component(ray.origin, axis);
+        const double direction = component(ray.direction, axis);
+        const double slab_min = component(minimum, axis);
+        const double slab_max = component(maximum, axis);
+
+        if (direction == 0.0) {
+            if (origin < slab_min || origin > slab_max) {
+                return false;
+            }
+            continue;
+        }
+
+        double first = (slab_min - origin) / direction;
+        double second = (slab_max - origin) / direction;
+        if (first > second) {
+            std::swap(first, second);
+        }
+        near_value = std::max(near_value, first);
+        far_value = std::min(far_value, second);
+        if (far_value < near_value) {
+            return false;
+        }
+    }
+
+    if (entry) {
+        *entry = near_value;
+    }
+    return true;
+}
+
 Aabb surroundingBox(const Aabb& left, const Aabb& right) {
     return Aabb(
         Vec3(std::min(left.minimum.x, right.minimum.x),


## `feat(accel): 도형 경계 계약과 구·평면 bounds 추가`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index 2daf96b..ce8f85b 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -1,7 +1,9 @@
 #pragma once
 
+#include "ray/accel.hpp"
 #include "ray/material.hpp"
 
+#include <optional>
 #include <string>
 
 namespace ray {
@@ -30,6 +32,9 @@ public:
                            double t_min,
                            double t_max,
                            HitRecord& hit) const = 0;
+    virtual std::optional<Aabb> bounds() const {
+        return std::nullopt;
+    }
     virtual std::string typeName() const = 0;
 
 protected:
@@ -49,6 +54,7 @@ public:
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
+    std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
 };
 
@@ -65,6 +71,7 @@ public:
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
+    std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
 };
 
diff --git a/src/geometry.cpp b/src/geometry.cpp
index 19a4f3c..7085907 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -71,6 +71,11 @@ std::string Sphere::typeName() const {
     return "sphere";
 }
 
+std::optional<Aabb> Sphere::bounds() const {
+    const Vec3 extent(radius, radius, radius);
+    return Aabb(center - extent, center + extent);
+}
+
 Plane::Plane(const Vec3& point_value,
              const Vec3& normal_value,
              const Material& material_value)
@@ -106,6 +111,10 @@ std::string Plane::typeName() const {
     return "plane";
 }
 
+std::optional<Aabb> Plane::bounds() const {
+    return std::nullopt;
+}
+
 Cylinder::Cylinder(const Vec3& center_value,
                    const Vec3& axis_value,
                    double radius_value,


## `feat(accel): 원기둥의 보수적 bounds 계산 추가`

diff --git a/include/ray/geometry.hpp b/include/ray/geometry.hpp
index ce8f85b..a2171bd 100644
--- a/include/ray/geometry.hpp
+++ b/include/ray/geometry.hpp
@@ -32,9 +32,7 @@ public:
                            double t_min,
                            double t_max,
                            HitRecord& hit) const = 0;
-    virtual std::optional<Aabb> bounds() const {
-        return std::nullopt;
-    }
+    virtual std::optional<Aabb> bounds() const = 0;
     virtual std::string typeName() const = 0;
 
 protected:
@@ -92,6 +90,7 @@ public:
                    double t_min,
                    double t_max,
                    HitRecord& hit) const override;
+    std::optional<Aabb> bounds() const override;
     std::string typeName() const override;
 };
 
diff --git a/src/geometry.cpp b/src/geometry.cpp
index 7085907..d0feb84 100644
--- a/src/geometry.cpp
+++ b/src/geometry.cpp
@@ -1,6 +1,8 @@
 #include "ray/geometry.hpp"
 
+#include <algorithm>
 #include <cmath>
+#include <limits>
 #include <utility>
 
 namespace ray {
@@ -268,4 +270,38 @@ std::string Cylinder::typeName() const {
     return "cylinder";
 }
 
+std::optional<Aabb> Cylinder::bounds() const {
+    const double half_height = height * 0.5;
+    const auto extent_for = [this, half_height](double axis_component) {
+        const double absolute_axis = std::fabs(axis_component);
+        const double radial =
+            std::sqrt(std::max(0.0, 1.0 - axis_component * axis_component));
+        const double side_extent =
+            absolute_axis * (half_height + kEpsilon) + radius * radial;
+        const double cap_extent =
+            absolute_axis * half_height +
+            std::sqrt(radius * radius + kEpsilon) * radial;
+        return std::max(side_extent, cap_extent);
+    };
+
+    const Vec3 extent(extent_for(axis.x),
+                      extent_for(axis.y),
+                      extent_for(axis.z));
+    Vec3 minimum = center - extent;
+    Vec3 maximum = center + extent;
+    minimum.x = std::nextafter(
+        minimum.x, -std::numeric_limits<double>::infinity());
+    minimum.y = std::nextafter(
+        minimum.y, -std::numeric_limits<double>::infinity());
+    minimum.z = std::nextafter(
+        minimum.z, -std::numeric_limits<double>::infinity());
+    maximum.x = std::nextafter(
+        maximum.x, std::numeric_limits<double>::infinity());
+    maximum.y = std::nextafter(
+        maximum.y, std::numeric_limits<double>::infinity());
+    maximum.z = std::nextafter(
+        maximum.z, std::numeric_limits<double>::infinity());
+    return Aabb(minimum, maximum);
+}
+
 }  // namespace ray


## `test(accel): AABB와 도형 경계 계산 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index 0eb578c..2872ac4 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -79,6 +79,62 @@ void testInvalidFixture() {
     require(rejected, "invalid scene fixture");
 }
 
+void testBounds() {
+    const ray::Aabb box(ray::Vec3(-1.0, -1.0, -1.0),
+                        ray::Vec3(1.0, 1.0, 1.0));
+    double entry = 0.0;
+    require(box.intersect(ray::Ray(ray::Vec3(0.0, 0.0, -3.0),
+                                   ray::Vec3(0.0, 0.0, 1.0)),
+                          0.0,
+                          100.0,
+                          &entry) &&
+                nearlyEqual(entry, 2.0),
+            "AABB forward hit");
+    require(box.intersect(ray::Ray(ray::Vec3(0.0, 0.0, 3.0),
+                                   ray::Vec3(0.0, 0.0, -1.0)),
+                          0.0,
+                          100.0),
+            "AABB negative direction");
+    require(box.intersect(ray::Ray(ray::Vec3(1.0, 0.0, -3.0),
+                                   ray::Vec3(0.0, 0.0, 1.0)),
+                          0.0,
+                          100.0),
+            "AABB boundary hit");
+    require(!box.intersect(ray::Ray(ray::Vec3(2.0, 0.0, -3.0),
+                                    ray::Vec3(0.0, 0.0, 1.0)),
+                           0.0,
+                           100.0),
+            "AABB parallel miss");
+
+    const ray::Material white(ray::Color(1.0, 1.0, 1.0));
+    const ray::Sphere sphere(ray::Vec3(1.0, 2.0, 3.0), 2.0, white);
+    const std::optional<ray::Aabb> sphere_bounds = sphere.bounds();
+    require(sphere_bounds &&
+                sphere_bounds->minimum == ray::Vec3(-1.0, 0.0, 1.0) &&
+                sphere_bounds->maximum == ray::Vec3(3.0, 4.0, 5.0),
+            "sphere bounds");
+
+    const ray::Plane plane(ray::Vec3(),
+                           ray::Vec3(0.0, 1.0, 0.0),
+                           white);
+    require(!plane.bounds(), "plane remains unbounded");
+
+    const ray::Cylinder cylinder(ray::Vec3(),
+                                 ray::Vec3(1.0, 1.0, 0.0),
+                                 1.0,
+                                 4.0,
+                                 white);
+    const std::optional<ray::Aabb> cylinder_bounds = cylinder.bounds();
+    const double exact_xy = 3.0 / std::sqrt(2.0);
+    require(cylinder_bounds &&
+                cylinder_bounds->maximum.x >= exact_xy &&
+                cylinder_bounds->maximum.y >= exact_xy &&
+                cylinder_bounds->maximum.x < exact_xy + 1.0e-3 &&
+                cylinder_bounds->maximum.z >= 1.0 &&
+                cylinder_bounds->maximum.z < 1.0 + 1.0e-3,
+            "arbitrary-axis cylinder bounds");
+}
+
 void testNearZeroDirections() {
     const std::string prefix =
         "R 8 8\n"
@@ -176,6 +232,7 @@ int main() {
     try {
         testMath();
         testGeometry();
+        testBounds();
         testInvalidFixture();
         testNearZeroDirections();
         testImageDimensions();
