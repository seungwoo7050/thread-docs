# 이미지 무결성과 실패 안전 PPM 저장

## `feat(renderer): 직렬 이미지 렌더링 구현`

diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index c72be15..0b130d9 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -3,9 +3,28 @@
 #include "ray/camera.hpp"
 
 #include <limits>
+#include <vector>
 
 namespace ray {
 
+struct RenderSettings {
+    int samplesPerPixel;
+    int maxDepth;
+    double tMin;
+    double tMax;
+
+    RenderSettings();
+};
+
+struct Image {
+    int width;
+    int height;
+    std::vector<unsigned char> pixels;
+
+    Image();
+    Image(int width_value, int height_value);
+};
+
 bool findNearestHit(const Scene& scene,
                     const Ray& ray,
                     HitRecord& hit,
@@ -20,5 +39,7 @@ Color shadeHit(const Scene& scene,
 Color traceRay(const Scene& scene,
                const Ray& ray,
                int max_depth = 1);
+Image renderScene(const Scene& scene,
+                  const RenderSettings& settings = RenderSettings());
 
 }  // namespace ray
diff --git a/src/renderer.cpp b/src/renderer.cpp
new file mode 100644
index 0000000..de19a3d
--- /dev/null
+++ b/src/renderer.cpp
@@ -0,0 +1,43 @@
+#include "ray/renderer.hpp"
+
+#include <cmath>
+
+namespace ray {
+
+RenderSettings::RenderSettings()
+    : samplesPerPixel(1),
+      maxDepth(1),
+      tMin(kRayTMin),
+      tMax(std::numeric_limits<double>::infinity()) {}
+
+Image::Image() : width(0), height(0), pixels() {}
+
+Image::Image(int width_value, int height_value)
+    : width(width_value),
+      height(height_value),
+      pixels(static_cast<std::size_t>(width_value * height_value * 3), 0) {}
+
+Image renderScene(const Scene& scene, const RenderSettings& settings) {
+    Image image(scene.width, scene.height);
+    std::size_t offset = 0;
+    for (int y = 0; y < scene.height; ++y) {
+        for (int x = 0; x < scene.width; ++x) {
+            const Ray ray = makeCameraRay(scene.camera,
+                                          scene.width,
+                                          scene.height,
+                                          x + 0.5,
+                                          y + 0.5);
+            const Color color = traceRay(scene, ray, settings.maxDepth);
+            const Color clamped = clampColor(color);
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.x * 255.0));
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.y * 255.0));
+            image.pixels[offset++] = static_cast<unsigned char>(
+                std::lround(clamped.z * 255.0));
+        }
+    }
+    return image;
+}
+
+}  // namespace ray


## `feat(output): PPM 직렬화와 이미지 체크섬 구현`

diff --git a/include/ray.hpp b/include/ray.hpp
index 6a19f38..3432841 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -4,6 +4,7 @@
 #include "ray/geometry.hpp"
 #include "ray/material.hpp"
 #include "ray/math.hpp"
+#include "ray/output.hpp"
 #include "ray/parser.hpp"
 #include "ray/renderer.hpp"
 #include "ray/scene.hpp"
diff --git a/include/ray/output.hpp b/include/ray/output.hpp
new file mode 100644
index 0000000..f66d09d
--- /dev/null
+++ b/include/ray/output.hpp
@@ -0,0 +1,12 @@
+#pragma once
+
+#include "ray/renderer.hpp"
+
+#include <string>
+
+namespace ray {
+
+void writePpm(const Image& image, const std::string& path);
+std::string checksumHex(const Image& image);
+
+}  // namespace ray
diff --git a/src/output.cpp b/src/output.cpp
new file mode 100644
index 0000000..07f02d2
--- /dev/null
+++ b/src/output.cpp
@@ -0,0 +1,46 @@
+#include "ray/output.hpp"
+
+#include <cstdint>
+#include <fstream>
+#include <iomanip>
+#include <sstream>
+#include <stdexcept>
+
+namespace ray {
+
+void writePpm(const Image& image, const std::string& path) {
+    std::ofstream output(path);
+    if (!output) {
+        throw std::runtime_error("cannot open output file: " + path);
+    }
+    output << "P3\n" << image.width << ' ' << image.height << "\n255\n";
+    for (int y = 0; y < image.height; ++y) {
+        for (int x = 0; x < image.width; ++x) {
+            const std::size_t base =
+                static_cast<std::size_t>((y * image.width + x) * 3);
+            output << static_cast<int>(image.pixels[base]) << ' '
+                   << static_cast<int>(image.pixels[base + 1]) << ' '
+                   << static_cast<int>(image.pixels[base + 2]) << '\n';
+        }
+    }
+}
+
+std::string checksumHex(const Image& image) {
+    std::uint64_t hash = 1469598103934665603ULL;
+    const auto mix = [&hash](unsigned char value) {
+        hash ^= static_cast<std::uint64_t>(value);
+        hash *= 1099511628211ULL;
+    };
+    mix(static_cast<unsigned char>(image.width & 0xff));
+    mix(static_cast<unsigned char>((image.width >> 8) & 0xff));
+    mix(static_cast<unsigned char>(image.height & 0xff));
+    mix(static_cast<unsigned char>((image.height >> 8) & 0xff));
+    for (unsigned char value : image.pixels) {
+        mix(value);
+    }
+    std::ostringstream out;
+    out << std::hex << std::setfill('0') << std::setw(16) << hash;
+    return out.str();
+}
+
+}  // namespace ray


## `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`

diff --git a/src/output.cpp b/src/output.cpp
index 07f02d2..8c4b756 100644
--- a/src/output.cpp
+++ b/src/output.cpp
@@ -17,7 +17,10 @@ void writePpm(const Image& image, const std::string& path) {
     for (int y = 0; y < image.height; ++y) {
         for (int x = 0; x < image.width; ++x) {
             const std::size_t base =
-                static_cast<std::size_t>((y * image.width + x) * 3);
+                (static_cast<std::size_t>(y) *
+                     static_cast<std::size_t>(image.width) +
+                 static_cast<std::size_t>(x)) *
+                3;
             output << static_cast<int>(image.pixels[base]) << ' '
                    << static_cast<int>(image.pixels[base + 1]) << ' '
                    << static_cast<int>(image.pixels[base + 2]) << '\n';
diff --git a/src/renderer.cpp b/src/renderer.cpp
index fd54281..c306f91 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -2,9 +2,29 @@
 
 #include <chrono>
 #include <cmath>
+#include <limits>
+#include <stdexcept>
 
 namespace ray {
 
+namespace {
+
+std::size_t pixelStorageSize(int width, int height) {
+    if (width <= 0 || height <= 0) {
+        throw std::invalid_argument("image dimensions must be positive");
+    }
+    const std::size_t safe_width = static_cast<std::size_t>(width);
+    const std::size_t safe_height = static_cast<std::size_t>(height);
+    const std::size_t limit = std::numeric_limits<std::size_t>::max();
+    if (safe_width > limit / safe_height ||
+        safe_width * safe_height > limit / 3) {
+        throw std::overflow_error("image dimensions are too large");
+    }
+    return safe_width * safe_height * 3;
+}
+
+}  // namespace
+
 RenderSettings::RenderSettings()
     : samplesPerPixel(1),
       maxDepth(1),
@@ -16,7 +36,7 @@ Image::Image() : width(0), height(0), pixels() {}
 Image::Image(int width_value, int height_value)
     : width(width_value),
       height(height_value),
-      pixels(static_cast<std::size_t>(width_value * height_value * 3), 0) {}
+      pixels(pixelStorageSize(width_value, height_value), 0) {}
 
 Image renderScene(const Scene& scene,
                   const RenderSettings& settings,


## `test(image): 잘못된 차원과 저장 크기 계산 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index a96d8d0..d5d1772 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -121,6 +121,32 @@ void testOutput() {
     require(ppm == "P3\n2 1\n255\n255 0 16\n0 127 255\n", "PPM encoding");
 }
 
+void testImageDimensions() {
+    const ray::Image image(2, 3);
+    require(image.width == 2 &&
+                image.height == 3 &&
+                image.pixels.size() == 18,
+            "image storage size");
+
+    bool zero_rejected = false;
+    try {
+        (void)ray::Image(0, 1);
+    } catch (const std::invalid_argument&) {
+        zero_rejected = true;
+    }
+    require(zero_rejected,
+            "zero image dimension rejection");
+
+    bool negative_rejected = false;
+    try {
+        (void)ray::Image(-1, 1);
+    } catch (const std::invalid_argument&) {
+        negative_rejected = true;
+    }
+    require(negative_rejected,
+            "negative image dimension rejection");
+}
+
 void testRenderGolden() {
     const ray::Scene scene = ray::loadScene(
         std::string(RAY_SOURCE_DIR) + "/scenes/basic.rt");
@@ -136,6 +162,7 @@ int main() {
         testGeometry();
         testInvalidFixture();
         testNearZeroDirections();
+        testImageDimensions();
         testOutput();
         testRenderGolden();
     } catch (const std::exception& error) {


## `fix(output): 표준 FNV-1a 기준값 적용`

diff --git a/src/output.cpp b/src/output.cpp
index 8c4b756..41c60e5 100644
--- a/src/output.cpp
+++ b/src/output.cpp
@@ -29,7 +29,7 @@ void writePpm(const Image& image, const std::string& path) {
 }
 
 std::string checksumHex(const Image& image) {
-    std::uint64_t hash = 1469598103934665603ULL;
+    std::uint64_t hash = 14695981039346656037ULL;
     const auto mix = [&hash](unsigned char value) {
         hash ^= static_cast<std::uint64_t>(value);
         hash *= 1099511628211ULL;


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


## `fix(output): 불일치한 이미지 저장소 거부`

diff --git a/include/ray/renderer.hpp b/include/ray/renderer.hpp
index 9cf6a87..6fdbaec 100644
--- a/include/ray/renderer.hpp
+++ b/include/ray/renderer.hpp
@@ -25,6 +25,8 @@ struct Image {
 
     Image();
     Image(int width_value, int height_value);
+
+    void validate() const;
 };
 
 bool findNearestHit(const Scene& scene,
diff --git a/src/output.cpp b/src/output.cpp
index 41c60e5..c8b0d96 100644
--- a/src/output.cpp
+++ b/src/output.cpp
@@ -9,6 +9,7 @@
 namespace ray {
 
 void writePpm(const Image& image, const std::string& path) {
+    image.validate();
     std::ofstream output(path);
     if (!output) {
         throw std::runtime_error("cannot open output file: " + path);
@@ -29,6 +30,7 @@ void writePpm(const Image& image, const std::string& path) {
 }
 
 std::string checksumHex(const Image& image) {
+    image.validate();
     std::uint64_t hash = 14695981039346656037ULL;
     const auto mix = [&hash](unsigned char value) {
         hash ^= static_cast<std::uint64_t>(value);
diff --git a/src/renderer.cpp b/src/renderer.cpp
index 9669736..4e3f41b 100644
--- a/src/renderer.cpp
+++ b/src/renderer.cpp
@@ -44,6 +44,14 @@ Image::Image(int width_value, int height_value)
       height(height_value),
       pixels(pixelStorageSize(width_value, height_value), 0) {}
 
+void Image::validate() const {
+    const std::size_t expected = pixelStorageSize(width, height);
+    if (pixels.size() != expected) {
+        throw std::invalid_argument(
+            "image pixel storage does not match its dimensions");
+    }
+}
+
 Image renderScene(const Scene& scene,
                   const RenderSettings& settings,
                   RenderStats* stats) {


## `test(output): 잘못된 이미지 저장소 처리 검증`

diff --git a/tests/core_tests.cpp b/tests/core_tests.cpp
index 2872ac4..f740266 100644
--- a/tests/core_tests.cpp
+++ b/tests/core_tests.cpp
@@ -178,6 +178,46 @@ void testOutput() {
     require(ray::checksumHex(image) == "0fde7b4d509f1daf", "checksum golden");
 }
 
+void testInvalidImageStorage() {
+    ray::Image image(2, 1);
+    image.pixels.pop_back();
+
+    bool checksum_rejected = false;
+    try {
+        (void)ray::checksumHex(image);
+    } catch (const std::invalid_argument&) {
+        checksum_rejected = true;
+    }
+    require(checksum_rejected, "checksum rejects short pixel storage");
+
+    const std::string path = "ray-core-test-preserved-output.ppm";
+    {
+        std::ofstream output(path);
+        output << "preserve me\n";
+    }
+    bool output_rejected = false;
+    try {
+        ray::writePpm(image, path);
+    } catch (const std::invalid_argument&) {
+        output_rejected = true;
+    }
+    const std::string preserved = readFile(path);
+    std::remove(path.c_str());
+    require(output_rejected, "PPM writer rejects short pixel storage");
+    require(preserved == "preserve me\n",
+            "invalid image does not truncate existing output");
+
+    image.pixels.push_back(0);
+    image.pixels.push_back(0);
+    bool oversized_rejected = false;
+    try {
+        image.validate();
+    } catch (const std::invalid_argument&) {
+        oversized_rejected = true;
+    }
+    require(oversized_rejected, "image rejects excess pixel storage");
+}
+
 void testCameraFrameReuse() {
     const ray::Camera camera(ray::Vec3(1.0, 2.0, -3.0),
                              ray::Vec3(-0.1, 0.25, 1.0),
@@ -238,6 +278,7 @@ int main() {
         testImageDimensions();
         testCameraFrameReuse();
         testOutput();
+        testInvalidImageStorage();
         testRenderGolden();
     } catch (const std::exception& error) {
         std::cerr << "core regression failed: " << error.what() << '\n';


## `fix(output): PPM 출력 실패 시 기존 파일 보존`

diff --git a/include/ray/output.hpp b/include/ray/output.hpp
index f66d09d..af31110 100644
--- a/include/ray/output.hpp
+++ b/include/ray/output.hpp
@@ -2,10 +2,12 @@
 
 #include "ray/renderer.hpp"
 
+#include <iosfwd>
 #include <string>
 
 namespace ray {
 
+void writePpm(const Image& image, std::ostream& output);
 void writePpm(const Image& image, const std::string& path);
 std::string checksumHex(const Image& image);
 
diff --git a/src/output.cpp b/src/output.cpp
index c8b0d96..6330590 100644
--- a/src/output.cpp
+++ b/src/output.cpp
@@ -1,19 +1,87 @@
 #include "ray/output.hpp"
 
+#include <atomic>
+#include <cerrno>
+#include <chrono>
+#include <cstdio>
 #include <cstdint>
+#include <cstring>
 #include <fstream>
 #include <iomanip>
+#include <ostream>
 #include <sstream>
 #include <stdexcept>
+#include <utility>
+
+#ifdef _WIN32
+#include <windows.h>
+#endif
 
 namespace ray {
 
-void writePpm(const Image& image, const std::string& path) {
-    image.validate();
-    std::ofstream output(path);
-    if (!output) {
-        throw std::runtime_error("cannot open output file: " + path);
+namespace {
+
+class TemporaryOutput {
+public:
+    explicit TemporaryOutput(std::string path)
+        : path_(std::move(path)), committed_(false) {}
+
+    ~TemporaryOutput() {
+        if (!committed_) {
+            (void)std::remove(path_.c_str());
+        }
+    }
+
+    const std::string& path() const {
+        return path_;
+    }
+
+    void commit() {
+        committed_ = true;
+    }
+
+private:
+    std::string path_;
+    bool committed_;
+};
+
+std::string temporaryPathFor(const std::string& path) {
+    static std::atomic<unsigned long long> sequence{0};
+    const unsigned long long stamp =
+        static_cast<unsigned long long>(
+            std::chrono::steady_clock::now()
+                .time_since_epoch()
+                .count());
+    const unsigned long long number =
+        sequence.fetch_add(1, std::memory_order_relaxed);
+    return path + ".tmp." + std::to_string(stamp) + "." +
+           std::to_string(number);
+}
+
+bool replaceFile(const std::string& source,
+                 const std::string& destination,
+                 std::string& reason) {
+#ifdef _WIN32
+    if (MoveFileExA(source.c_str(),
+                    destination.c_str(),
+                    MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)) {
+        return true;
     }
+    reason = "system error " + std::to_string(GetLastError());
+    return false;
+#else
+    if (std::rename(source.c_str(), destination.c_str()) == 0) {
+        return true;
+    }
+    reason = std::strerror(errno);
+    return false;
+#endif
+}
+
+}  // namespace
+
+void writePpm(const Image& image, std::ostream& output) {
+    image.validate();
     output << "P3\n" << image.width << ' ' << image.height << "\n255\n";
     for (int y = 0; y < image.height; ++y) {
         for (int x = 0; x < image.width; ++x) {
@@ -27,6 +95,33 @@ void writePpm(const Image& image, const std::string& path) {
                    << static_cast<int>(image.pixels[base + 2]) << '\n';
         }
     }
+    if (!output) {
+        throw std::runtime_error("cannot write PPM output stream");
+    }
+}
+
+void writePpm(const Image& image, const std::string& path) {
+    image.validate();
+    TemporaryOutput temporary(temporaryPathFor(path));
+    {
+        std::ofstream output(temporary.path(),
+                             std::ios::out | std::ios::trunc);
+        if (!output) {
+            throw std::runtime_error(
+                "cannot open temporary output file for: " + path);
+        }
+        output.exceptions(std::ios::badbit | std::ios::failbit);
+        writePpm(image, output);
+        output.flush();
+        output.close();
+    }
+
+    std::string reason;
+    if (!replaceFile(temporary.path(), path, reason)) {
+        throw std::runtime_error(
+            "cannot replace output file " + path + ": " + reason);
+    }
+    temporary.commit();
 }
 
 std::string checksumHex(const Image& image) {


## `test(output): 출력 실패의 대상 보존과 정리 검증`

diff --git a/CMakeLists.txt b/CMakeLists.txt
index 2194507..d48bcb4 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -77,6 +77,10 @@ if(BUILD_TESTING)
     target_link_libraries(ray-render-tests PRIVATE raycore)
     add_test(NAME render_determinism COMMAND ray-render-tests)
 
+    add_executable(ray-output-tests tests/output_tests.cpp)
+    target_link_libraries(ray-output-tests PRIVATE raycore)
+    add_test(NAME output_failure_regression COMMAND ray-output-tests)
+
     add_test(
         NAME cli_contract
         COMMAND bash
diff --git a/tests/output_tests.cpp b/tests/output_tests.cpp
new file mode 100644
index 0000000..1a7c7b7
--- /dev/null
+++ b/tests/output_tests.cpp
@@ -0,0 +1,155 @@
+#include "ray.hpp"
+
+#include <chrono>
+#include <filesystem>
+#include <fstream>
+#include <iostream>
+#include <ostream>
+#include <sstream>
+#include <stdexcept>
+#include <streambuf>
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
+class FailingBuffer : public std::streambuf {
+protected:
+    std::streamsize xsputn(const char*, std::streamsize) override {
+        return 0;
+    }
+
+    int_type overflow(int_type) override {
+        return traits_type::eof();
+    }
+};
+
+class TestDirectory {
+public:
+    TestDirectory() {
+        const unsigned long long token =
+            static_cast<unsigned long long>(
+                std::chrono::steady_clock::now()
+                    .time_since_epoch()
+                    .count());
+        path_ = std::filesystem::temp_directory_path() /
+                ("ray-output-regression-" + std::to_string(token));
+        if (!std::filesystem::create_directory(path_)) {
+            throw std::runtime_error("cannot create output test directory");
+        }
+    }
+
+    ~TestDirectory() {
+        std::error_code ignored;
+        std::filesystem::remove_all(path_, ignored);
+    }
+
+    const std::filesystem::path& path() const {
+        return path_;
+    }
+
+private:
+    std::filesystem::path path_;
+};
+
+std::string readFile(const std::filesystem::path& path) {
+    std::ifstream input(path);
+    std::ostringstream contents;
+    contents << input.rdbuf();
+    return contents.str();
+}
+
+void writeFile(const std::filesystem::path& path,
+               const std::string& contents) {
+    std::ofstream output(path);
+    output << contents;
+    if (!output) {
+        throw std::runtime_error("cannot prepare output test file");
+    }
+}
+
+void requireNoTemporaryFiles(const std::filesystem::path& directory,
+                             const std::string& prefix) {
+    for (const std::filesystem::directory_entry& entry :
+         std::filesystem::directory_iterator(directory)) {
+        const std::string name = entry.path().filename().string();
+        require(name.rfind(prefix + ".tmp.", 0) != 0,
+                "temporary output file was not cleaned up");
+    }
+}
+
+ray::Image sampleImage() {
+    ray::Image image(2, 1);
+    image.pixels = {255, 0, 16, 0, 127, 255};
+    return image;
+}
+
+void testStreamFailure() {
+    FailingBuffer buffer;
+    std::ostream output(&buffer);
+    bool rejected = false;
+    try {
+        ray::writePpm(sampleImage(), output);
+    } catch (const std::runtime_error&) {
+        rejected = true;
+    }
+    require(rejected, "PPM serializer reports stream failure");
+}
+
+void testAtomicReplacement() {
+    TestDirectory directory;
+    const std::filesystem::path target = directory.path() / "image.ppm";
+    writeFile(target, "old image\n");
+
+    ray::writePpm(sampleImage(), target.string());
+
+    require(readFile(target) ==
+                "P3\n2 1\n255\n255 0 16\n0 127 255\n",
+            "PPM writer replaces an existing file");
+    requireNoTemporaryFiles(directory.path(), "image.ppm");
+}
+
+void testFailedReplacementPreservesDestination() {
+    TestDirectory directory;
+    const std::filesystem::path target =
+        directory.path() / "existing-destination";
+    std::filesystem::create_directory(target);
+    const std::filesystem::path sentinel = target / "keep.txt";
+    writeFile(sentinel, "preserve me\n");
+
+    bool rejected = false;
+    try {
+        ray::writePpm(sampleImage(), target.string());
+    } catch (const std::runtime_error&) {
+        rejected = true;
+    }
+
+    require(rejected, "PPM writer reports replacement failure");
+    require(std::filesystem::is_directory(target),
+            "failed replacement preserves destination type");
+    require(readFile(sentinel) == "preserve me\n",
+            "failed replacement preserves destination contents");
+    requireNoTemporaryFiles(directory.path(),
+                            "existing-destination");
+}
+
+}  // namespace
+
+int main() {
+    try {
+        testStreamFailure();
+        testAtomicReplacement();
+        testFailedReplacementPreservesDestination();
+    } catch (const std::exception& error) {
+        std::cerr << "output failure regression failed: "
+                  << error.what() << '\n';
+        return 1;
+    }
+    std::cout << "output failure regression checks passed\n";
+    return 0;
+}
