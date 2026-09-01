# 카메라 투영과 프레임 재사용

## `feat(camera): 화면 좌표를 카메라 광선으로 변환`

diff --git a/include/ray.hpp b/include/ray.hpp
index 934048f..aa39b79 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -1,5 +1,6 @@
 #pragma once
 
+#include "ray/camera.hpp"
 #include "ray/geometry.hpp"
 #include "ray/material.hpp"
 #include "ray/math.hpp"
diff --git a/include/ray/camera.hpp b/include/ray/camera.hpp
new file mode 100644
index 0000000..7ebad74
--- /dev/null
+++ b/include/ray/camera.hpp
@@ -0,0 +1,22 @@
+#pragma once
+
+#include "ray/scene.hpp"
+
+namespace ray {
+
+struct CameraFrame {
+    Vec3 forward;
+    Vec3 right;
+    Vec3 up;
+    double viewportWidth;
+    double viewportHeight;
+};
+
+CameraFrame buildCameraFrame(const Camera& camera, int width, int height);
+Ray makeCameraRay(const Camera& camera,
+                  int width,
+                  int height,
+                  double pixel_x,
+                  double pixel_y);
+
+}  // namespace ray
diff --git a/src/camera.cpp b/src/camera.cpp
new file mode 100644
index 0000000..1fd3a3e
--- /dev/null
+++ b/src/camera.cpp
@@ -0,0 +1,56 @@
+#include "ray/camera.hpp"
+
+#include <algorithm>
+#include <cmath>
+
+namespace ray {
+
+CameraFrame buildCameraFrame(const Camera& camera, int width, int height) {
+    Vec3 forward = normalize(camera.direction);
+    if (forward.isNearZero()) {
+        forward = Vec3(0.0, 0.0, 1.0);
+    }
+
+    Vec3 up_seed = normalize(camera.up);
+    if (up_seed.isNearZero() || std::fabs(dot(up_seed, forward)) > 0.999) {
+        up_seed = std::fabs(forward.y) < 0.999
+                      ? Vec3(0.0, 1.0, 0.0)
+                      : Vec3(1.0, 0.0, 0.0);
+    }
+
+    const Vec3 right = normalize(cross(up_seed, forward));
+    const Vec3 true_up = normalize(cross(forward, right));
+    const double safe_width = std::max(1, width);
+    const double safe_height = std::max(1, height);
+    const double aspect = safe_width / safe_height;
+    const double fov_radians =
+        camera.fovDegrees * 3.14159265358979323846 / 180.0;
+    const double viewport_height = 2.0 * std::tan(fov_radians * 0.5);
+
+    CameraFrame frame;
+    frame.forward = forward;
+    frame.right = right;
+    frame.up = true_up;
+    frame.viewportHeight = viewport_height;
+    frame.viewportWidth = viewport_height * aspect;
+    return frame;
+}
+
+Ray makeCameraRay(const Camera& camera,
+                  int width,
+                  int height,
+                  double pixel_x,
+                  double pixel_y) {
+    const CameraFrame frame = buildCameraFrame(camera, width, height);
+    const double safe_width = static_cast<double>(std::max(1, width));
+    const double safe_height = static_cast<double>(std::max(1, height));
+    const double u =
+        (pixel_x / safe_width - 0.5) * frame.viewportWidth;
+    const double v =
+        (0.5 - pixel_y / safe_height) * frame.viewportHeight;
+    const Vec3 direction =
+        normalize(frame.forward + frame.right * u + frame.up * v);
+    return Ray(camera.position, direction);
+}
+
+}  // namespace ray


## `perf(camera): 픽셀별 카메라 프레임 재계산 제거`

diff --git a/include/ray/camera.hpp b/include/ray/camera.hpp
index 7ebad74..682c8bb 100644
--- a/include/ray/camera.hpp
+++ b/include/ray/camera.hpp
@@ -18,5 +18,11 @@ Ray makeCameraRay(const Camera& camera,
                   int height,
                   double pixel_x,
                   double pixel_y);
+Ray makeCameraRay(const Camera& camera,
+                  const CameraFrame& frame,
+                  int width,
+                  int height,
+                  double pixel_x,
+                  double pixel_y);
 
 }  // namespace ray
diff --git a/src/camera.cpp b/src/camera.cpp
index 1fd3a3e..23133d8 100644
--- a/src/camera.cpp
+++ b/src/camera.cpp
@@ -42,6 +42,20 @@ Ray makeCameraRay(const Camera& camera,
                   double pixel_x,
                   double pixel_y) {
     const CameraFrame frame = buildCameraFrame(camera, width, height);
+    return makeCameraRay(camera,
+                         frame,
+                         width,
+                         height,
+                         pixel_x,
+                         pixel_y);
+}
+
+Ray makeCameraRay(const Camera& camera,
+                  const CameraFrame& frame,
+                  int width,
+                  int height,
+                  double pixel_x,
+                  double pixel_y) {
     const double safe_width = static_cast<double>(std::max(1, width));
     const double safe_height = static_cast<double>(std::max(1, height));
     const double u =
diff --git a/src/renderer.cpp b/src/renderer.cpp
index c306f91..ee8f10f 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -43,10 +43,13 @@ Image renderScene(const Scene& scene,
                   RenderStats* stats) {
     const auto started = std::chrono::steady_clock::now();
     Image image(scene.width, scene.height);
+    const CameraFrame camera_frame =
+        buildCameraFrame(scene.camera, scene.width, scene.height);
     std::size_t offset = 0;
     for (int y = 0; y < scene.height; ++y) {
         for (int x = 0; x < scene.width; ++x) {
             const Ray ray = makeCameraRay(scene.camera,
+                                          camera_frame,
                                           scene.width,
                                           scene.height,
                                           x + 0.5,


## `test(camera): 재사용한 카메라 프레임의 동치 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index a2e432c..0eb578c 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -122,6 +122,20 @@ void testOutput() {
     require(ray::checksumHex(image) == "0fde7b4d509f1daf", "checksum golden");
 }
 
+void testCameraFrameReuse() {
+    const ray::Camera camera(ray::Vec3(1.0, 2.0, -3.0),
+                             ray::Vec3(-0.1, 0.25, 1.0),
+                             57.0);
+    const ray::CameraFrame frame = ray::buildCameraFrame(camera, 640, 360);
+    const ray::Ray rebuilt =
+        ray::makeCameraRay(camera, 640, 360, 217.5, 103.5);
+    const ray::Ray reused =
+        ray::makeCameraRay(camera, frame, 640, 360, 217.5, 103.5);
+    require(rebuilt.origin == reused.origin &&
+                rebuilt.direction == reused.direction,
+            "cached camera frame");
+}
+
 void testImageDimensions() {
     const ray::Image image(2, 3);
     require(image.width == 2 &&
@@ -165,6 +179,7 @@ int main() {
         testInvalidFixture();
         testNearZeroDirections();
         testImageDimensions();
+        testCameraFrameReuse();
         testOutput();
         testRenderGolden();
     } catch (const std::exception& error) {
