# 레이 트레이싱 수학 기반과 수치 안정성

## `feat(math): 벡터 값과 산술 연산 구현`

diff --git a/Makefile b/Makefile
index 8ad26ec..a040b15 100644
--- a/Makefile
+++ b/Makefile
@@ -1,4 +1,5 @@
 CXX ?= c++
+CPPFLAGS ?= -Iinclude
 CXXFLAGS ?= -std=c++17 -Wall -Wextra -Wpedantic -O2
 
 TARGET := ray-scene-tracer
@@ -9,7 +10,7 @@ SRC := $(sort $(wildcard src/*.cpp))
 all: $(TARGET)
 
 $(TARGET): $(SRC)
-	$(CXX) $(CXXFLAGS) -o $@ $(SRC)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -o $@ $(SRC)
 
 test: $(TARGET)
 	./$(TARGET) --help >/dev/null
diff --git a/include/ray.hpp b/include/ray.hpp
new file mode 100644
index 0000000..e1b9561
--- /dev/null
+++ b/include/ray.hpp
@@ -0,0 +1,3 @@
+#pragma once
+
+#include "ray/math.hpp"
diff --git a/include/ray/math.hpp b/include/ray/math.hpp
new file mode 100644
index 0000000..cd9465b
--- /dev/null
+++ b/include/ray/math.hpp
@@ -0,0 +1,26 @@
+#pragma once
+
+namespace ray {
+
+constexpr double kEpsilon = 1.0e-6;
+constexpr double kRayTMin = 1.0e-4;
+
+struct Vec3 {
+    double x;
+    double y;
+    double z;
+
+    Vec3();
+    Vec3(double x_value, double y_value, double z_value);
+};
+
+using Color = Vec3;
+
+Vec3 operator+(const Vec3& left, const Vec3& right);
+Vec3 operator-(const Vec3& left, const Vec3& right);
+Vec3 operator-(const Vec3& value);
+Vec3 operator*(const Vec3& value, double scalar);
+Vec3 operator*(double scalar, const Vec3& value);
+Vec3 operator/(const Vec3& value, double scalar);
+
+}  // namespace ray
diff --git a/src/math.cpp b/src/math.cpp
new file mode 100644
index 0000000..68da7d0
--- /dev/null
+++ b/src/math.cpp
@@ -0,0 +1,34 @@
+#include "ray/math.hpp"
+
+namespace ray {
+
+Vec3::Vec3() : x(0.0), y(0.0), z(0.0) {}
+
+Vec3::Vec3(double x_value, double y_value, double z_value)
+    : x(x_value), y(y_value), z(z_value) {}
+
+Vec3 operator+(const Vec3& left, const Vec3& right) {
+    return Vec3(left.x + right.x, left.y + right.y, left.z + right.z);
+}
+
+Vec3 operator-(const Vec3& left, const Vec3& right) {
+    return Vec3(left.x - right.x, left.y - right.y, left.z - right.z);
+}
+
+Vec3 operator-(const Vec3& value) {
+    return Vec3(-value.x, -value.y, -value.z);
+}
+
+Vec3 operator*(const Vec3& value, double scalar) {
+    return Vec3(value.x * scalar, value.y * scalar, value.z * scalar);
+}
+
+Vec3 operator*(double scalar, const Vec3& value) {
+    return value * scalar;
+}
+
+Vec3 operator/(const Vec3& value, double scalar) {
+    return Vec3(value.x / scalar, value.y / scalar, value.z / scalar);
+}
+
+}  // namespace ray


## `feat(math): 벡터 길이와 기하 연산 구현`

diff --git a/include/ray/math.hpp b/include/ray/math.hpp
index cd9465b..e721f55 100644
--- a/include/ray/math.hpp
+++ b/include/ray/math.hpp
@@ -12,6 +12,11 @@ struct Vec3 {
 
     Vec3();
     Vec3(double x_value, double y_value, double z_value);
+
+    double lengthSquared() const;
+    double length() const;
+    Vec3 normalized() const;
+    bool isNearZero(double epsilon = kEpsilon) const;
 };
 
 using Color = Vec3;
@@ -23,4 +28,9 @@ Vec3 operator*(const Vec3& value, double scalar);
 Vec3 operator*(double scalar, const Vec3& value);
 Vec3 operator/(const Vec3& value, double scalar);
 
+double dot(const Vec3& left, const Vec3& right);
+Vec3 cross(const Vec3& left, const Vec3& right);
+double length(const Vec3& value);
+Vec3 normalize(const Vec3& value);
+
 }  // namespace ray
diff --git a/src/math.cpp b/src/math.cpp
index 68da7d0..2d68dc0 100644
--- a/src/math.cpp
+++ b/src/math.cpp
@@ -1,5 +1,7 @@
 #include "ray/math.hpp"
 
+#include <cmath>
+
 namespace ray {
 
 Vec3::Vec3() : x(0.0), y(0.0), z(0.0) {}
@@ -7,6 +9,24 @@ Vec3::Vec3() : x(0.0), y(0.0), z(0.0) {}
 Vec3::Vec3(double x_value, double y_value, double z_value)
     : x(x_value), y(y_value), z(z_value) {}
 
+double Vec3::lengthSquared() const {
+    return x * x + y * y + z * z;
+}
+
+double Vec3::length() const {
+    return std::sqrt(lengthSquared());
+}
+
+Vec3 Vec3::normalized() const {
+    return normalize(*this);
+}
+
+bool Vec3::isNearZero(double epsilon) const {
+    return std::fabs(x) < epsilon &&
+           std::fabs(y) < epsilon &&
+           std::fabs(z) < epsilon;
+}
+
 Vec3 operator+(const Vec3& left, const Vec3& right) {
     return Vec3(left.x + right.x, left.y + right.y, left.z + right.z);
 }
@@ -31,4 +51,26 @@ Vec3 operator/(const Vec3& value, double scalar) {
     return Vec3(value.x / scalar, value.y / scalar, value.z / scalar);
 }
 
+double dot(const Vec3& left, const Vec3& right) {
+    return left.x * right.x + left.y * right.y + left.z * right.z;
+}
+
+Vec3 cross(const Vec3& left, const Vec3& right) {
+    return Vec3(left.y * right.z - left.z * right.y,
+                left.z * right.x - left.x * right.z,
+                left.x * right.y - left.y * right.x);
+}
+
+double length(const Vec3& value) {
+    return value.length();
+}
+
+Vec3 normalize(const Vec3& value) {
+    const double len = value.length();
+    if (len <= kEpsilon) {
+        return Vec3();
+    }
+    return value / len;
+}
+
 }  // namespace ray


## `feat(math): 벡터 비교와 색상 범위 연산 추가`

diff --git a/include/ray/math.hpp b/include/ray/math.hpp
index e721f55..974e9d0 100644
--- a/include/ray/math.hpp
+++ b/include/ray/math.hpp
@@ -1,5 +1,7 @@
 #pragma once
 
+#include <iosfwd>
+
 namespace ray {
 
 constexpr double kEpsilon = 1.0e-6;
@@ -24,13 +26,25 @@ using Color = Vec3;
 Vec3 operator+(const Vec3& left, const Vec3& right);
 Vec3 operator-(const Vec3& left, const Vec3& right);
 Vec3 operator-(const Vec3& value);
+Vec3 operator*(const Vec3& left, const Vec3& right);
 Vec3 operator*(const Vec3& value, double scalar);
 Vec3 operator*(double scalar, const Vec3& value);
 Vec3 operator/(const Vec3& value, double scalar);
+Vec3& operator+=(Vec3& left, const Vec3& right);
+Vec3& operator-=(Vec3& left, const Vec3& right);
+Vec3& operator*=(Vec3& value, double scalar);
+Vec3& operator/=(Vec3& value, double scalar);
+bool operator==(const Vec3& left, const Vec3& right);
+bool operator!=(const Vec3& left, const Vec3& right);
+std::ostream& operator<<(std::ostream& stream, const Vec3& value);
 
 double dot(const Vec3& left, const Vec3& right);
 Vec3 cross(const Vec3& left, const Vec3& right);
 double length(const Vec3& value);
 Vec3 normalize(const Vec3& value);
+double clamp(double value, double min_value, double max_value);
+Color clampColor(const Color& value,
+                 double min_value = 0.0,
+                 double max_value = 1.0);
 
 }  // namespace ray
diff --git a/src/math.cpp b/src/math.cpp
index 2d68dc0..e5d1f7e 100644
--- a/src/math.cpp
+++ b/src/math.cpp
@@ -1,6 +1,8 @@
 #include "ray/math.hpp"
 
+#include <algorithm>
 #include <cmath>
+#include <ostream>
 
 namespace ray {
 
@@ -39,6 +41,10 @@ Vec3 operator-(const Vec3& value) {
     return Vec3(-value.x, -value.y, -value.z);
 }
 
+Vec3 operator*(const Vec3& left, const Vec3& right) {
+    return Vec3(left.x * right.x, left.y * right.y, left.z * right.z);
+}
+
 Vec3 operator*(const Vec3& value, double scalar) {
     return Vec3(value.x * scalar, value.y * scalar, value.z * scalar);
 }
@@ -51,6 +57,47 @@ Vec3 operator/(const Vec3& value, double scalar) {
     return Vec3(value.x / scalar, value.y / scalar, value.z / scalar);
 }
 
+Vec3& operator+=(Vec3& left, const Vec3& right) {
+    left.x += right.x;
+    left.y += right.y;
+    left.z += right.z;
+    return left;
+}
+
+Vec3& operator-=(Vec3& left, const Vec3& right) {
+    left.x -= right.x;
+    left.y -= right.y;
+    left.z -= right.z;
+    return left;
+}
+
+Vec3& operator*=(Vec3& value, double scalar) {
+    value.x *= scalar;
+    value.y *= scalar;
+    value.z *= scalar;
+    return value;
+}
+
+Vec3& operator/=(Vec3& value, double scalar) {
+    value.x /= scalar;
+    value.y /= scalar;
+    value.z /= scalar;
+    return value;
+}
+
+bool operator==(const Vec3& left, const Vec3& right) {
+    return left.x == right.x && left.y == right.y && left.z == right.z;
+}
+
+bool operator!=(const Vec3& left, const Vec3& right) {
+    return !(left == right);
+}
+
+std::ostream& operator<<(std::ostream& stream, const Vec3& value) {
+    stream << value.x << ',' << value.y << ',' << value.z;
+    return stream;
+}
+
 double dot(const Vec3& left, const Vec3& right) {
     return left.x * right.x + left.y * right.y + left.z * right.z;
 }
@@ -73,4 +120,14 @@ Vec3 normalize(const Vec3& value) {
     return value / len;
 }
 
+double clamp(double value, double min_value, double max_value) {
+    return std::max(min_value, std::min(value, max_value));
+}
+
+Color clampColor(const Color& value, double min_value, double max_value) {
+    return Color(clamp(value.x, min_value, max_value),
+                 clamp(value.y, min_value, max_value),
+                 clamp(value.z, min_value, max_value));
+}
+
 }  // namespace ray


## `feat(ray): 광선 위치 계산 모델 추가`

diff --git a/include/ray/math.hpp b/include/ray/math.hpp
index 974e9d0..9b61743 100644
--- a/include/ray/math.hpp
+++ b/include/ray/math.hpp
@@ -47,4 +47,14 @@ Color clampColor(const Color& value,
                  double min_value = 0.0,
                  double max_value = 1.0);
 
+struct Ray {
+    Vec3 origin;
+    Vec3 direction;
+
+    Ray();
+    Ray(const Vec3& origin_value, const Vec3& direction_value);
+
+    Vec3 at(double t) const;
+};
+
 }  // namespace ray
diff --git a/src/math.cpp b/src/math.cpp
index e5d1f7e..7a5154b 100644
--- a/src/math.cpp
+++ b/src/math.cpp
@@ -130,4 +130,13 @@ Color clampColor(const Color& value, double min_value, double max_value) {
                  clamp(value.z, min_value, max_value));
 }
 
+Ray::Ray() : origin(), direction(0.0, 0.0, 1.0) {}
+
+Ray::Ray(const Vec3& origin_value, const Vec3& direction_value)
+    : origin(origin_value), direction(direction_value) {}
+
+Vec3 Ray::at(double t) const {
+    return origin + direction * t;
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


## `fix(math): 큰 유한 벡터를 안정적으로 정규화`

diff --git a/src/math.cpp b/src/math.cpp
index 7a5154b..69193a0 100644
--- a/src/math.cpp
+++ b/src/math.cpp
@@ -16,7 +16,7 @@ double Vec3::lengthSquared() const {
 }
 
 double Vec3::length() const {
-    return std::sqrt(lengthSquared());
+    return std::hypot(x, y, z);
 }
 
 Vec3 Vec3::normalized() const {


## `test(math): 큰 유한 벡터 정규화 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index a2b33bd..08b1f8e 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -38,6 +38,9 @@ void testMath() {
             "cross product");
     require(nearlyEqual(ray::normalize(ray::Vec3(0.0, 3.0, 4.0)).length(), 1.0),
             "normalization");
+    require(ray::normalize(ray::Vec3(1.0e308, 0.0, 0.0)) ==
+                ray::Vec3(1.0, 0.0, 0.0),
+            "large finite vector normalization");
 }
 
 void testGeometry() {
