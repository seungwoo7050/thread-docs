# 장면 소유권과 정확성 보존 BVH 수명 주기

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


## `refactor(scene): 장면 도형의 단독 소유권 적용`

diff --git a/benchmarks/render_benchmark.cpp b/benchmarks/render_benchmark.cpp
index f9bbf7c..e19d154 100644
--- a/benchmarks/render_benchmark.cpp
+++ b/benchmarks/render_benchmark.cpp
@@ -32,7 +32,7 @@ ray::Scene makeDenseScene() {
                               ray::Color(0.75, 0.85, 1.0)));
 
     const ray::Material ground(ray::Color(0.35, 0.38, 0.42));
-    scene.addShape(std::make_shared<ray::Plane>(
+    scene.addShape(std::make_unique<ray::Plane>(
         ray::Vec3(0.0, -1.0, 0.0),
         ray::Vec3(0.0, 1.0, 0.0),
         ground));
@@ -46,7 +46,7 @@ ray::Scene makeDenseScene() {
                 0.2 + 0.6 * static_cast<double>(column % 5) / 4.0,
                 0.2 + 0.6 * static_cast<double>(row % 5) / 4.0,
                 0.25 + 0.5 * static_cast<double>((row + column) % 5) / 4.0);
-            scene.addShape(std::make_shared<ray::Sphere>(
+            scene.addShape(std::make_unique<ray::Sphere>(
                 ray::Vec3(x, y, z),
                 0.42,
                 ray::Material(color)));
diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index 002c887..68d9e25 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -52,11 +52,15 @@ public:
     Color background;
     Camera camera;
     std::vector<Light> lights;
-    std::vector<std::shared_ptr<Shape>> shapes;
+    std::vector<std::unique_ptr<Shape>> shapes;
 
     Scene();
+    Scene(const Scene&) = delete;
+    Scene& operator=(const Scene&) = delete;
+    Scene(Scene&&) noexcept = default;
+    Scene& operator=(Scene&&) noexcept = default;
 
-    void addShape(const std::shared_ptr<Shape>& shape);
+    void addShape(std::unique_ptr<Shape> shape);
     void addLight(const Light& light);
     bool intersect(const Ray& ray,
                    double t_min,
diff --git a/src/parser.cpp b/src/parser.cpp
index a2aba11..090fca8 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -392,7 +392,7 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            source_name,
                            line_number,
                            "sphere color"));
-            scene.addShape(std::make_shared<Sphere>(
+            scene.addShape(std::make_unique<Sphere>(
                 center,
                 diameter * 0.5,
                 material));
@@ -422,7 +422,7 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            line_number,
                            "plane color"));
             scene.addShape(
-                std::make_shared<Plane>(point, normal, material));
+                std::make_unique<Plane>(point, normal, material));
         } else if (id == "cy") {
             expectCount(tokens,
                         6,
@@ -458,7 +458,7 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            source_name,
                            line_number,
                            "cylinder color"));
-            scene.addShape(std::make_shared<Cylinder>(
+            scene.addShape(std::make_unique<Cylinder>(
                 center,
                 axis,
                 diameter * 0.5,
diff --git a/src/scene.cpp b/src/scene.cpp
index ecaf08f..0e450e6 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -43,9 +43,9 @@ Scene::Scene()
       lights(),
       shapes() {}
 
-void Scene::addShape(const std::shared_ptr<Shape>& shape) {
+void Scene::addShape(std::unique_ptr<Shape> shape) {
     if (shape) {
-        shapes.push_back(shape);
+        shapes.push_back(std::move(shape));
     }
 }
 
@@ -62,7 +62,7 @@ bool Scene::intersect(const Ray& ray,
     double closest = t_max;
     HitRecord candidate;
 
-    for (const std::shared_ptr<Shape>& shape : shapes) {
+    for (const std::unique_ptr<Shape>& shape : shapes) {
         if (!shape) {
             continue;
         }


## `feat(accel): BVH node와 연속 저장소 구성`

diff --git a/include/ray/accel.hpp b/include/ray/accel.hpp
index dee14a7..8de5c8f 100644
--- a/include/ray/accel.hpp
+++ b/include/ray/accel.hpp
@@ -2,6 +2,9 @@
 
 #include "ray/math.hpp"
 
+#include <cstdint>
+#include <vector>
+
 namespace ray {
 
 struct Aabb {
@@ -21,4 +24,32 @@ struct Aabb {
 
 Aabb surroundingBox(const Aabb& left, const Aabb& right);
 
+struct BvhPrimitive {
+    std::uint32_t shapeIndex;
+    Aabb bounds;
+};
+
+struct BvhNode {
+    Aabb bounds;
+    std::uint32_t left = 0;
+    std::uint32_t right = 0;
+    std::uint32_t first = 0;
+    std::uint32_t count = 0;
+
+    bool isLeaf() const;
+};
+
+class Bvh {
+public:
+    void clear();
+    bool empty() const;
+
+    const std::vector<BvhNode>& nodes() const;
+    const std::vector<std::uint32_t>& primitiveIndices() const;
+
+private:
+    std::vector<BvhNode> nodes_;
+    std::vector<std::uint32_t> primitiveIndices_;
+};
+
 }  // namespace ray
diff --git a/src/accel.cpp b/src/accel.cpp
index 9b44e41..58a265a 100644
--- a/src/accel.cpp
+++ b/src/accel.cpp
@@ -92,4 +92,25 @@ Aabb surroundingBox(const Aabb& left, const Aabb& right) {
              std::max(left.maximum.z, right.maximum.z)));
 }
 
+bool BvhNode::isLeaf() const {
+    return count > 0;
+}
+
+void Bvh::clear() {
+    nodes_.clear();
+    primitiveIndices_.clear();
+}
+
+bool Bvh::empty() const {
+    return nodes_.empty();
+}
+
+const std::vector<BvhNode>& Bvh::nodes() const {
+    return nodes_;
+}
+
+const std::vector<std::uint32_t>& Bvh::primitiveIndices() const {
+    return primitiveIndices_;
+}
+
 }  // namespace ray


## `feat(accel): 결정적 중앙 분할 BVH 구축 구현`

diff --git a/include/ray/accel.hpp b/include/ray/accel.hpp
index 8de5c8f..f4292e0 100644
--- a/include/ray/accel.hpp
+++ b/include/ray/accel.hpp
@@ -41,6 +41,7 @@ struct BvhNode {
 
 class Bvh {
 public:
+    void build(std::vector<BvhPrimitive> primitives);
     void clear();
     bool empty() const;
 
@@ -48,6 +49,10 @@ public:
     const std::vector<std::uint32_t>& primitiveIndices() const;
 
 private:
+    std::uint32_t buildNode(std::vector<BvhPrimitive>& primitives,
+                            std::uint32_t first,
+                            std::uint32_t last);
+
     std::vector<BvhNode> nodes_;
     std::vector<std::uint32_t> primitiveIndices_;
 };
diff --git a/src/accel.cpp b/src/accel.cpp
index 58a265a..3971bc3 100644
--- a/src/accel.cpp
+++ b/src/accel.cpp
@@ -96,6 +96,21 @@ bool BvhNode::isLeaf() const {
     return count > 0;
 }
 
+void Bvh::build(std::vector<BvhPrimitive> primitives) {
+    clear();
+    if (primitives.empty()) {
+        return;
+    }
+    nodes_.reserve(primitives.size() * 2);
+    (void)buildNode(primitives,
+                    0,
+                    static_cast<std::uint32_t>(primitives.size()));
+    primitiveIndices_.reserve(primitives.size());
+    for (const BvhPrimitive& primitive : primitives) {
+        primitiveIndices_.push_back(primitive.shapeIndex);
+    }
+}
+
 void Bvh::clear() {
     nodes_.clear();
     primitiveIndices_.clear();
@@ -113,4 +128,61 @@ const std::vector<std::uint32_t>& Bvh::primitiveIndices() const {
     return primitiveIndices_;
 }
 
+std::uint32_t Bvh::buildNode(std::vector<BvhPrimitive>& primitives,
+                             std::uint32_t first,
+                             std::uint32_t last) {
+    Aabb node_bounds = primitives[first].bounds;
+    Vec3 centroid_min = primitives[first].bounds.centroid();
+    Vec3 centroid_max = centroid_min;
+    for (std::uint32_t index = first + 1; index < last; ++index) {
+        node_bounds = surroundingBox(node_bounds, primitives[index].bounds);
+        const Vec3 centroid = primitives[index].bounds.centroid();
+        centroid_min.x = std::min(centroid_min.x, centroid.x);
+        centroid_min.y = std::min(centroid_min.y, centroid.y);
+        centroid_min.z = std::min(centroid_min.z, centroid.z);
+        centroid_max.x = std::max(centroid_max.x, centroid.x);
+        centroid_max.y = std::max(centroid_max.y, centroid.y);
+        centroid_max.z = std::max(centroid_max.z, centroid.z);
+    }
+
+    const std::uint32_t node_index =
+        static_cast<std::uint32_t>(nodes_.size());
+    nodes_.push_back(BvhNode());
+    nodes_[node_index].bounds = node_bounds;
+
+    const std::uint32_t count = last - first;
+    if (count <= 4) {
+        nodes_[node_index].first = first;
+        nodes_[node_index].count = count;
+        return node_index;
+    }
+
+    const Vec3 extent = centroid_max - centroid_min;
+    int axis = 0;
+    if (extent.y > extent.x) {
+        axis = 1;
+    }
+    if (component(extent, 2) > component(extent, axis)) {
+        axis = 2;
+    }
+    std::stable_sort(
+        primitives.begin() + first,
+        primitives.begin() + last,
+        [axis](const BvhPrimitive& left, const BvhPrimitive& right) {
+            const double left_value = component(left.bounds.centroid(), axis);
+            const double right_value = component(right.bounds.centroid(), axis);
+            if (left_value != right_value) {
+                return left_value < right_value;
+            }
+            return left.shapeIndex < right.shapeIndex;
+        });
+
+    const std::uint32_t middle = first + count / 2;
+    const std::uint32_t left = buildNode(primitives, first, middle);
+    const std::uint32_t right = buildNode(primitives, middle, last);
+    nodes_[node_index].left = left;
+    nodes_[node_index].right = right;
+    return node_index;
+}
+
 }  // namespace ray


## `feat(accel): 선형·BVH 탐색 모드 계약 연결`

diff --git a/include/ray/accel.hpp b/include/ray/accel.hpp
index f4292e0..847c6b2 100644
--- a/include/ray/accel.hpp
+++ b/include/ray/accel.hpp
@@ -7,6 +7,11 @@
 
 namespace ray {
 
+enum class AccelMode {
+    Linear,
+    Bvh
+};
+
 struct Aabb {
     Vec3 minimum;
     Vec3 maximum;
diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index 9c66ab5..9089004 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -12,6 +12,7 @@ struct RenderSettings {
     int maxDepth;
     double tMin;
     double tMax;
+    AccelMode accelMode;
 
     RenderSettings();
 };
@@ -30,18 +31,22 @@ bool findNearestHit(const Scene& scene,
                     HitRecord& hit,
                     double t_min = kRayTMin,
                     double t_max = std::numeric_limits<double>::infinity(),
+                    AccelMode mode = AccelMode::Bvh,
                     RenderStats* stats = nullptr);
 bool isOccluded(const Scene& scene,
                 const Ray& shadow_ray,
                 double max_distance,
+                AccelMode mode = AccelMode::Bvh,
                 RenderStats* stats = nullptr);
 Color shadeHit(const Scene& scene,
                const HitRecord& hit,
                const Ray& view_ray,
+               AccelMode mode = AccelMode::Bvh,
                RenderStats* stats = nullptr);
 Color traceRay(const Scene& scene,
                const Ray& ray,
                int max_depth = 1,
+               AccelMode mode = AccelMode::Bvh,
                RenderStats* stats = nullptr);
 Image renderScene(const Scene& scene,
                   const RenderSettings& settings = RenderSettings(),
diff --git a/include/ray/scene.hpp b/include/ray/scene.hpp
index 68d9e25..a26cb1c 100644
--- a/include/ray/scene.hpp
+++ b/include/ray/scene.hpp
@@ -66,6 +66,7 @@ public:
                    double t_min,
                    double t_max,
                    HitRecord& hit,
+                   AccelMode mode = AccelMode::Bvh,
                    RenderStats* stats = nullptr) const;
 };
 
diff --git a/src/renderer.cpp b/src/renderer.cpp
index ee8f10f..7c3bab7 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -29,7 +29,8 @@ RenderSettings::RenderSettings()
     : samplesPerPixel(1),
       maxDepth(1),
       tMin(kRayTMin),
-      tMax(std::numeric_limits<double>::infinity()) {}
+      tMax(std::numeric_limits<double>::infinity()),
+      accelMode(AccelMode::Bvh) {}
 
 Image::Image() : width(0), height(0), pixels() {}
 
@@ -60,6 +61,7 @@ Image renderScene(const Scene& scene,
             const Color color = traceRay(scene,
                                          ray,
                                          settings.maxDepth,
+                                         settings.accelMode,
                                          stats);
             const Color clamped = clampColor(color);
             image.pixels[offset++] = static_cast<unsigned char>(
diff --git a/src/scene.cpp b/src/scene.cpp
index 0e450e6..b7c5240 100644
--- a/src/scene.cpp
+++ b/src/scene.cpp
@@ -57,23 +57,39 @@ bool Scene::intersect(const Ray& ray,
                       double t_min,
                       double t_max,
                       HitRecord& hit,
+                      AccelMode mode,
                       RenderStats* stats) const {
+    (void)mode;
     bool found = false;
     double closest = t_max;
+    std::uint32_t best_index = 0;
     HitRecord candidate;
 
-    for (const std::unique_ptr<Shape>& shape : shapes) {
-        if (!shape) {
-            continue;
-        }
-        if (stats) {
-            ++stats->primitiveTests;
-        }
-        if (shape->intersect(ray, t_min, closest, candidate)) {
-            found = true;
-            closest = candidate.t;
-            hit = candidate;
-        }
+    const auto test_shape =
+        [&](std::uint32_t index) {
+            const std::unique_ptr<Shape>& shape = shapes[index];
+            if (!shape) {
+                return;
+            }
+            if (stats) {
+                ++stats->primitiveTests;
+            }
+            if (!shape->intersect(
+                    ray, t_min, closest, candidate)) {
+                return;
+            }
+            if (!found ||
+                candidate.t < closest ||
+                (candidate.t == closest && index > best_index)) {
+                found = true;
+                closest = candidate.t;
+                best_index = index;
+                hit = candidate;
+            }
+        };
+
+    for (std::size_t index = 0; index < shapes.size(); ++index) {
+        test_shape(static_cast<std::uint32_t>(index));
     }
     return found;
 }
diff --git a/src/shading.cpp b/src/shading.cpp
index 80d5f74..e64416a 100644
--- a/src/shading.cpp
+++ b/src/shading.cpp
@@ -10,25 +10,29 @@ bool findNearestHit(const Scene& scene,
                     HitRecord& hit,
                     double t_min,
                     double t_max,
+                    AccelMode mode,
                     RenderStats* stats) {
-    return scene.intersect(ray, t_min, t_max, hit, stats);
+    return scene.intersect(ray, t_min, t_max, hit, mode, stats);
 }
 
 bool isOccluded(const Scene& scene,
                 const Ray& shadow_ray,
                 double max_distance,
+                AccelMode mode,
                 RenderStats* stats) {
     HitRecord ignored;
     return scene.intersect(shadow_ray,
                            kRayTMin,
                            std::max(kRayTMin, max_distance - kRayTMin),
                            ignored,
+                           mode,
                            stats);
 }
 
 Color shadeHit(const Scene& scene,
                const HitRecord& hit,
                const Ray& view_ray,
+               AccelMode mode,
                RenderStats* stats) {
     (void)view_ray;
 
@@ -56,6 +60,7 @@ Color shadeHit(const Scene& scene,
         if (isOccluded(scene,
                        Ray(shadow_origin, light_direction),
                        distance_to_light,
+                       mode,
                        stats)) {
             continue;
         }
@@ -70,6 +75,7 @@ Color shadeHit(const Scene& scene,
 Color traceRay(const Scene& scene,
                const Ray& ray,
                int max_depth,
+               AccelMode mode,
                RenderStats* stats) {
     (void)max_depth;
 
@@ -78,10 +84,11 @@ Color traceRay(const Scene& scene,
                          kRayTMin,
                          std::numeric_limits<double>::infinity(),
                          hit,
+                         mode,
                          stats)) {
         return scene.background;
     }
-    return shadeHit(scene, hit, ray, stats);
+    return shadeHit(scene, hit, ray, mode, stats);
 }
 
 }  // namespace ray


