# 직접 조명과 하드 섀도

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


## `feat(scene): 카메라·조명과 장면 aggregate 구성`

diff --git a/include/ray.hpp b/include/ray.hpp
index 4182ae9..e95bfc1 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -3,3 +3,4 @@
 #include "ray/geometry.hpp"
 #include "ray/material.hpp"
 #include "ray/math.hpp"
+#include "ray/scene.hpp"
diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
new file mode 100644
index 0000000..229453b
--- /dev/null
+++ b/include/ray/scene.hpp
@@ -0,0 +1,52 @@
+#pragma once
+
+#include "ray/geometry.hpp"
+
+#include <memory>
+#include <vector>
+
+namespace ray {
+
+struct Light {
+    Vec3 position;
+    double brightness;
+    Color color;
+
+    Light();
+    Light(const Vec3& position_value,
+          double brightness_value,
+          const Color& color_value);
+};
+
+struct Camera {
+    Vec3 position;
+    Vec3 direction;
+    double fovDegrees;
+    Vec3 up;
+
+    Camera();
+    Camera(const Vec3& position_value,
+           const Vec3& direction_value,
+           double fov_degrees_value);
+};
+
+class Scene {
+public:
+    int width;
+    int height;
+    bool hasResolution;
+    bool hasAmbient;
+    bool hasCamera;
+    double ambientRatio;
+    Color ambientColor;
+    Color background;
+    Camera camera;
+    std::vector<Light> lights;
+    std::vector<std::shared_ptr<Shape>> shapes;
+
+    Scene();
+
+    void addLight(const Light& light);
+};
+
+}  // namespace ray
diff --git a/src/scene.cpp b/src/scene.cpp
index e87ade9..ac2c925 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -1,4 +1,4 @@
-#include "ray/material.hpp"
+#include "ray/scene.hpp"
 
 namespace ray {
 
@@ -6,4 +6,45 @@ Material::Material() : albedo(1.0, 1.0, 1.0) {}
 
 Material::Material(const Color& color) : albedo(color) {}
 
+Light::Light()
+    : position(), brightness(1.0), color(1.0, 1.0, 1.0) {}
+
+Light::Light(const Vec3& position_value,
+             double brightness_value,
+             const Color& color_value)
+    : position(position_value),
+      brightness(brightness_value),
+      color(color_value) {}
+
+Camera::Camera()
+    : position(),
+      direction(0.0, 0.0, 1.0),
+      fovDegrees(60.0),
+      up(0.0, 1.0, 0.0) {}
+
+Camera::Camera(const Vec3& position_value,
+               const Vec3& direction_value,
+               double fov_degrees_value)
+    : position(position_value),
+      direction(direction_value),
+      fovDegrees(fov_degrees_value),
+      up(0.0, 1.0, 0.0) {}
+
+Scene::Scene()
+    : width(0),
+      height(0),
+      hasResolution(false),
+      hasAmbient(false),
+      hasCamera(false),
+      ambientRatio(0.0),
+      ambientColor(1.0, 1.0, 1.0),
+      background(0.02, 0.03, 0.05),
+      camera(),
+      lights(),
+      shapes() {}
+
+void Scene::addLight(const Light& light) {
+    lights.push_back(light);
+}
+
 }  // namespace ray


## `feat(scene): 선형 최근접 교차 탐색 구현`

diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index 229453b..bbbcc4d 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -46,7 +46,12 @@ public:
 
     Scene();
 
+    void addShape(const std::shared_ptr<Shape>& shape);
     void addLight(const Light& light);
+    bool intersect(const Ray& ray,
+                   double t_min,
+                   double t_max,
+                   HitRecord& hit) const;
 };
 
 }  // namespace ray
diff --git a/src/scene.cpp b/src/scene.cpp
index ac2c925..7354196 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -43,8 +43,35 @@ Scene::Scene()
       lights(),
       shapes() {}
 
+void Scene::addShape(const std::shared_ptr<Shape>& shape) {
+    if (shape) {
+        shapes.push_back(shape);
+    }
+}
+
 void Scene::addLight(const Light& light) {
     lights.push_back(light);
 }
 
+bool Scene::intersect(const Ray& ray,
+                      double t_min,
+                      double t_max,
+                      HitRecord& hit) const {
+    bool found = false;
+    double closest = t_max;
+    HitRecord candidate;
+
+    for (const std::shared_ptr<Shape>& shape : shapes) {
+        if (!shape) {
+            continue;
+        }
+        if (shape->intersect(ray, t_min, closest, candidate)) {
+            found = true;
+            closest = candidate.t;
+            hit = candidate;
+        }
+    }
+    return found;
+}
+
 }  // namespace ray


## `feat(render): 직접광과 그림자 추적 구현`

diff --git a/include/ray.hpp b/include/ray.hpp
index aa39b79..6a19f38 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -5,4 +5,5 @@
 #include "ray/material.hpp"
 #include "ray/math.hpp"
 #include "ray/parser.hpp"
+#include "ray/renderer.hpp"
 #include "ray/scene.hpp"
diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
new file mode 100644
index 0000000..c72be15
--- /dev/null
+++ b/include/ray/renderer.hpp
@@ -0,0 +1,24 @@
+#pragma once
+
+#include "ray/camera.hpp"
+
+#include <limits>
+
+namespace ray {
+
+bool findNearestHit(const Scene& scene,
+                    const Ray& ray,
+                    HitRecord& hit,
+                    double t_min = kRayTMin,
+                    double t_max = std::numeric_limits<double>::infinity());
+bool isOccluded(const Scene& scene,
+                const Ray& shadow_ray,
+                double max_distance);
+Color shadeHit(const Scene& scene,
+               const HitRecord& hit,
+               const Ray& view_ray);
+Color traceRay(const Scene& scene,
+               const Ray& ray,
+               int max_depth = 1);
+
+}  // namespace ray
diff --git a/src/shading.cpp b/src/shading.cpp
new file mode 100644
index 0000000..7827064
--- /dev/null
+++ b/src/shading.cpp
@@ -0,0 +1,75 @@
+#include "ray/renderer.hpp"
+
+#include <algorithm>
+#include <limits>
+
+namespace ray {
+
+bool findNearestHit(const Scene& scene,
+                    const Ray& ray,
+                    HitRecord& hit,
+                    double t_min,
+                    double t_max) {
+    return scene.intersect(ray, t_min, t_max, hit);
+}
+
+bool isOccluded(const Scene& scene,
+                const Ray& shadow_ray,
+                double max_distance) {
+    HitRecord ignored;
+    return scene.intersect(shadow_ray,
+                           kRayTMin,
+                           std::max(kRayTMin, max_distance - kRayTMin),
+                           ignored);
+}
+
+Color shadeHit(const Scene& scene,
+               const HitRecord& hit,
+               const Ray& view_ray) {
+    (void)view_ray;
+
+    Color result =
+        hit.material.albedo * scene.ambientColor * scene.ambientRatio;
+    const Vec3 shadow_origin = hit.point + hit.normal * kRayTMin;
+
+    for (const Light& light : scene.lights) {
+        const Vec3 to_light = light.position - hit.point;
+        const double distance_to_light = to_light.length();
+        if (distance_to_light <= kEpsilon) {
+            continue;
+        }
+
+        const Vec3 light_direction = to_light / distance_to_light;
+        const double diffuse =
+            std::max(0.0, dot(hit.normal, light_direction));
+        if (diffuse <= 0.0) {
+            continue;
+        }
+
+        if (isOccluded(scene,
+                       Ray(shadow_origin, light_direction),
+                       distance_to_light)) {
+            continue;
+        }
+
+        result += hit.material.albedo * light.color *
+                  (light.brightness * diffuse);
+    }
+
+    return clampColor(result);
+}
+
+Color traceRay(const Scene& scene, const Ray& ray, int max_depth) {
+    (void)max_depth;
+
+    HitRecord hit;
+    if (!scene.intersect(ray,
+                         kRayTMin,
+                         std::numeric_limits<double>::infinity(),
+                         hit)) {
+        return scene.background;
+    }
+    return shadeHit(scene, hit, ray);
+}
+
+}  // namespace ray


## `test(render): 장면 렌더링 smoke 검사 추가`

diff --git a/Makefile b/Makefile
index 263b96c..04a91fd 100644
--- a/Makefile
+++ b/Makefile
@@ -13,7 +13,7 @@ $(TARGET): $(SRC)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -o $@ $(SRC)
 
 test: $(TARGET)
-	./$(TARGET) scenes/basic.rt /tmp/ray-scene-tracer-check.ppm --checksum >/dev/null
+	tests/render_smoke.sh
 
 clean:
 	rm -f $(TARGET)
diff --git a/tests/render_smoke.sh b/tests/render_smoke.sh
new file mode 100755
index 0000000..d937cbe
--- /dev/null
+++ b/tests/render_smoke.sh
@@ -0,0 +1,72 @@
+#!/usr/bin/env bash
+set -euo pipefail
+
+ROOT=$(cd "$(dirname "$0")/.." && pwd)
+BIN="$ROOT/ray-scene-tracer"
+TMP=$(mktemp -d)
+trap 'rm -rf "$TMP"' EXIT
+
+make -C "$ROOT" >/dev/null
+
+BAD_SCENE="$TMP/bad.rt"
+BAD_OUT="$TMP/bad.ppm"
+cat >"$BAD_SCENE" <<'SCENE'
+R 16 8
+C 0,0,3 0,0,-1 60
+not_a_shape 1 2 3
+SCENE
+
+if "$BIN" "$BAD_SCENE" "$BAD_OUT" >"$TMP/bad.stdout" 2>"$TMP/bad.stderr"; then
+    echo "expected parser failure for unknown directive" >&2
+    exit 1
+fi
+if [[ -s "$BAD_OUT" ]]; then
+    echo "parser failure should not leave a rendered image" >&2
+    exit 1
+fi
+
+SCENE_FILE="$TMP/smoke.rt"
+cat >"$SCENE_FILE" <<'SCENE'
+# Educational miniRT-style scene. RGB uses 0..255.
+R 64 32
+A 0.12 255,255,255
+C 0,0.8,3.2 0,0.25,-1 55
+L -3,5,2.5 0.9 255,244,220
+sp 0,0,-1.4 1.10 220,70,45
+sp 0.9,-0.1,-2.2 0.90 65,120,220
+pl 0,-0.65,0 0,1,0 185,190,178
+SCENE
+
+PPM_ONE="$TMP/one.ppm"
+PPM_TWO="$TMP/two.ppm"
+CHECK_ONE=$("$BIN" "$SCENE_FILE" "$PPM_ONE" --checksum)
+CHECK_TWO=$("$BIN" "$SCENE_FILE" "$PPM_TWO" --checksum)
+
+mapfile -t HEADER < <(head -n 3 "$PPM_ONE")
+if [[ "${HEADER[0]}" != "P3" ]]; then
+    echo "expected P3 PPM magic, got ${HEADER[0]}" >&2
+    exit 1
+fi
+if [[ "${HEADER[1]}" != "64 32" ]]; then
+    echo "expected PPM dimensions 64 32, got ${HEADER[1]}" >&2
+    exit 1
+fi
+if [[ "${HEADER[2]}" != "255" ]]; then
+    echo "expected PPM max channel 255, got ${HEADER[2]}" >&2
+    exit 1
+fi
+
+if [[ ! "$CHECK_ONE" =~ ^[0-9a-f]{16}$ ]]; then
+    echo "expected checksum hex output, got $CHECK_ONE" >&2
+    exit 1
+fi
+if [[ "$CHECK_ONE" != "$CHECK_TWO" ]]; then
+    echo "checksum output is not deterministic" >&2
+    exit 1
+fi
+if ! cmp -s "$PPM_ONE" "$PPM_TWO"; then
+    echo "PPM output is not deterministic" >&2
+    exit 1
+fi
+
+echo "render smoke checks passed"


## `test(output): PPM과 렌더링 체크섬 기준 고정`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index d5d1772..a2e432c 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -119,6 +119,7 @@ void testOutput() {
     std::remove(path.c_str());
 
     require(ppm == "P3\n2 1\n255\n255 0 16\n0 127 255\n", "PPM encoding");
+    require(ray::checksumHex(image) == "0fde7b4d509f1daf", "checksum golden");
 }
 
 void testImageDimensions() {
@@ -152,6 +153,7 @@ void testRenderGolden() {
         std::string(RAY_SOURCE_DIR) + "/scenes/basic.rt");
     const ray::Image image = ray::renderScene(scene);
     require(image.width == 640 && image.height == 360, "render dimensions");
+    require(ray::checksumHex(image) == "456dc8d87ebf194f", "render checksum");
 }
 
 }  // namespace
