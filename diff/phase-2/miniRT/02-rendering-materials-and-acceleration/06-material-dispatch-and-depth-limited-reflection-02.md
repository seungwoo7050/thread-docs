## `feat(cli): 반사 깊이 option과 기본값 추가`

diff --git a/src/main.cpp b/src/main.cpp
index 2c67e73..40ff1eb 100644
--- a/src/main.cpp
+++ b/src/main.cpp
@@ -21,7 +21,8 @@ void printUsage() {
         << "usage: ray-scene-tracer <scene.rt> <output.ppm>"
         << " [--checksum]"
         << " [--accel linear|bvh]"
-        << " [--threads N|auto]\n";
+        << " [--threads N|auto]"
+        << " [--max-depth 0..32]\n";
 }
 
 bool parseUnsigned(const std::string& token,
@@ -60,6 +61,7 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
     bool seen_checksum = false;
     bool seen_accel = false;
     bool seen_threads = false;
+    bool seen_max_depth = false;
 
     for (int index = 3; index < argc; ++index) {
         const std::string option = argv[index];
@@ -112,6 +114,20 @@ bool parseCli(int argc, char** argv, CliOptions& options) {
                 static_cast<unsigned int>(parsed);
             continue;
         }
+
+        if (option == "--max-depth") {
+            if (seen_max_depth || index + 1 >= argc) {
+                return false;
+            }
+            seen_max_depth = true;
+            unsigned long long parsed = 0;
+            if (!parseUnsigned(argv[++index], 32, parsed)) {
+                return false;
+            }
+            options.renderSettings.maxDepth =
+                static_cast<int>(parsed);
+            continue;
+        }
         return false;
     }
     return true;
diff --git a/src/renderer.cpp b/src/renderer.cpp
index 472076b..9669736 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -31,7 +31,7 @@ std::size_t pixelStorageSize(int width, int height) {
 
 RenderSettings::RenderSettings()
     : samplesPerPixel(1),
-      maxDepth(1),
+      maxDepth(4),
       tMin(kRayTMin),
       tMax(std::numeric_limits<double>::infinity()),
       accelMode(AccelMode::Bvh),
