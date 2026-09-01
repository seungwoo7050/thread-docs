# 장면 문법·값 검증·객체 구성

## `feat(parser): 소스 위치 오류와 line tokenization 구성`

diff --git a/include/ray.hpp b/include/ray.hpp
index e95bfc1..934048f 100644
--- a/include/ray.hpp
+++ b/include/ray.hpp
@@ -3,4 +3,5 @@
 #include "ray/geometry.hpp"
 #include "ray/material.hpp"
 #include "ray/math.hpp"
+#include "ray/parser.hpp"
 #include "ray/scene.hpp"
diff --git a/include/ray/parser.hpp b/include/ray/parser.hpp
new file mode 100644
index 0000000..75099af
--- /dev/null
+++ b/include/ray/parser.hpp
@@ -0,0 +1,31 @@
+#pragma once
+
+#include "ray/scene.hpp"
+
+#include <cstddef>
+#include <iosfwd>
+#include <stdexcept>
+#include <string>
+
+namespace ray {
+
+class ParseError : public std::runtime_error {
+public:
+    ParseError(const std::string& source_name,
+               std::size_t line_number,
+               const std::string& message);
+
+    const std::string& source() const noexcept;
+    std::size_t line() const noexcept;
+
+private:
+    std::string source_;
+    std::size_t line_;
+};
+
+namespace parser {
+Scene parseScene(std::istream& input,
+                 const std::string& source_name = "<stream>");
+}  // namespace parser
+
+}  // namespace ray
diff --git a/src/parser.cpp b/src/parser.cpp
new file mode 100644
index 0000000..bb899a3
--- /dev/null
+++ b/src/parser.cpp
@@ -0,0 +1,67 @@
+#include "ray/parser.hpp"
+
+#include <cctype>
+#include <sstream>
+#include <string>
+#include <vector>
+
+namespace ray {
+
+namespace {
+
+std::string makeParseMessage(const std::string& source_name,
+                             std::size_t line_number,
+                             const std::string& message) {
+    std::ostringstream output;
+    output << source_name;
+    if (line_number > 0) {
+        output << ':' << line_number;
+    }
+    output << ": " << message;
+    return output.str();
+}
+
+[[maybe_unused]] std::string trim(const std::string& value) {
+    std::size_t first = 0;
+    while (first < value.size() &&
+           std::isspace(static_cast<unsigned char>(value[first]))) {
+        ++first;
+    }
+
+    std::size_t last = value.size();
+    while (last > first &&
+           std::isspace(static_cast<unsigned char>(value[last - 1]))) {
+        --last;
+    }
+    return value.substr(first, last - first);
+}
+
+[[maybe_unused]] std::vector<std::string> splitTokens(const std::string& line) {
+    std::vector<std::string> tokens;
+    std::istringstream input(line);
+    std::string token;
+    while (input >> token) {
+        tokens.push_back(token);
+    }
+    return tokens;
+}
+
+}  // namespace
+
+ParseError::ParseError(const std::string& source_name,
+                       std::size_t line_number,
+                       const std::string& message)
+    : std::runtime_error(
+          makeParseMessage(source_name, line_number, message)),
+      source_(source_name),
+      line_(line_number) {}
+
+const std::string& ParseError::source() const noexcept {
+    return source_;
+}
+
+std::size_t ParseError::line() const noexcept {
+    return line_;
+}
+
+}  // namespace ray


## `feat(parser): 유한 수와 범위 값 해석 구현`

diff --git a/src/parser.cpp b/src/parser.cpp
index bb899a3..3f2897a 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -1,6 +1,8 @@
 #include "ray/parser.hpp"
 
 #include <cctype>
+#include <cmath>
+#include <limits>
 #include <sstream>
 #include <string>
 #include <vector>
@@ -46,6 +48,85 @@ std::string makeParseMessage(const std::string& source_name,
     return tokens;
 }
 
+[[maybe_unused]] void expectCount(const std::vector<std::string>& tokens,
+                 std::size_t expected,
+                 const std::string& source_name,
+                 std::size_t line_number,
+                 const std::string& form) {
+    if (tokens.size() != expected) {
+        std::ostringstream message;
+        message << "expected " << form << " with " << expected - 1
+                << " argument(s), got " << tokens.size() - 1;
+        throw ParseError(source_name, line_number, message.str());
+    }
+}
+
+double parseDoubleToken(const std::string& token,
+                        const std::string& source_name,
+                        std::size_t line_number,
+                        const std::string& field_name) {
+    try {
+        std::size_t parsed = 0;
+        const double value = std::stod(token, &parsed);
+        if (parsed != token.size() || !std::isfinite(value)) {
+            throw std::invalid_argument("not a finite number");
+        }
+        return value;
+    } catch (const std::exception&) {
+        throw ParseError(source_name,
+                         line_number,
+                         "invalid " + field_name + " value '" + token + "'");
+    }
+}
+
+[[maybe_unused]] int parseIntToken(const std::string& token,
+                  const std::string& source_name,
+                  std::size_t line_number,
+                  const std::string& field_name) {
+    try {
+        std::size_t parsed = 0;
+        const long long value = std::stoll(token, &parsed);
+        if (parsed != token.size() ||
+            value < 1 ||
+            value > std::numeric_limits<int>::max()) {
+            throw std::invalid_argument("not a positive int");
+        }
+        return static_cast<int>(value);
+    } catch (const std::exception&) {
+        throw ParseError(source_name,
+                         line_number,
+                         "invalid " + field_name + " value '" + token + "'");
+    }
+}
+
+[[maybe_unused]] double parseRatio(const std::string& token,
+                  const std::string& source_name,
+                  std::size_t line_number,
+                  const std::string& field_name) {
+    const double value =
+        parseDoubleToken(token, source_name, line_number, field_name);
+    if (value < 0.0 || value > 1.0) {
+        throw ParseError(source_name,
+                         line_number,
+                         field_name + " must be between 0.0 and 1.0");
+    }
+    return value;
+}
+
+[[maybe_unused]] double parsePositiveDouble(const std::string& token,
+                           const std::string& source_name,
+                           std::size_t line_number,
+                           const std::string& field_name) {
+    const double value =
+        parseDoubleToken(token, source_name, line_number, field_name);
+    if (value <= 0.0) {
+        throw ParseError(source_name,
+                         line_number,
+                         field_name + " must be positive");
+    }
+    return value;
+}
+
 }  // namespace
 
 ParseError::ParseError(const std::string& source_name,


## `feat(parser): 벡터와 색상 token 해석 구현`

diff --git a/src/parser.cpp b/src/parser.cpp
index 3f2897a..60a8c1c 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -48,6 +48,21 @@ std::string makeParseMessage(const std::string& source_name,
     return tokens;
 }
 
+std::vector<std::string> splitCommas(const std::string& token) {
+    std::vector<std::string> parts;
+    std::size_t start = 0;
+    while (start <= token.size()) {
+        const std::size_t comma = token.find(',', start);
+        if (comma == std::string::npos) {
+            parts.push_back(token.substr(start));
+            break;
+        }
+        parts.push_back(token.substr(start, comma - start));
+        start = comma + 1;
+    }
+    return parts;
+}
+
 [[maybe_unused]] void expectCount(const std::vector<std::string>& tokens,
                  std::size_t expected,
                  const std::string& source_name,
@@ -127,6 +142,71 @@ double parseDoubleToken(const std::string& token,
     return value;
 }
 
+[[maybe_unused]] Vec3 parseVec3(const std::string& token,
+               const std::string& source_name,
+               std::size_t line_number,
+               const std::string& field_name) {
+    const std::vector<std::string> parts = splitCommas(token);
+    if (parts.size() != 3 ||
+        parts[0].empty() ||
+        parts[1].empty() ||
+        parts[2].empty()) {
+        throw ParseError(source_name,
+                         line_number,
+                         field_name + " must use x,y,z format");
+    }
+    return Vec3(
+        parseDoubleToken(parts[0],
+                         source_name,
+                         line_number,
+                         field_name + ".x"),
+        parseDoubleToken(parts[1],
+                         source_name,
+                         line_number,
+                         field_name + ".y"),
+        parseDoubleToken(parts[2],
+                         source_name,
+                         line_number,
+                         field_name + ".z"));
+}
+
+[[maybe_unused]] Color parseColor(const std::string& token,
+                 const std::string& source_name,
+                 std::size_t line_number,
+                 const std::string& field_name) {
+    const std::vector<std::string> parts = splitCommas(token);
+    if (parts.size() != 3 ||
+        parts[0].empty() ||
+        parts[1].empty() ||
+        parts[2].empty()) {
+        throw ParseError(source_name,
+                         line_number,
+                         field_name + " must use r,g,b format");
+    }
+
+    int channels[3] = {};
+    for (std::size_t index = 0; index < 3; ++index) {
+        try {
+            std::size_t parsed = 0;
+            const long long value = std::stoll(parts[index], &parsed);
+            if (parsed != parts[index].size() ||
+                value < 0 ||
+                value > 255) {
+                throw std::invalid_argument("not a byte");
+            }
+            channels[index] = static_cast<int>(value);
+        } catch (const std::exception&) {
+            throw ParseError(source_name,
+                             line_number,
+                             "invalid " + field_name + " channel '" +
+                                 parts[index] + "'");
+        }
+    }
+    return Color(static_cast<double>(channels[0]) / 255.0,
+                 static_cast<double>(channels[1]) / 255.0,
+                 static_cast<double>(channels[2]) / 255.0);
+}
+
 }  // namespace
 
 ParseError::ParseError(const std::string& source_name,


## `feat(parser): 줄 단위 지시어 dispatch 기반 구성`

diff --git a/src/parser.cpp b/src/parser.cpp
index 60a8c1c..a92f0bd 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -23,7 +23,7 @@ std::string makeParseMessage(const std::string& source_name,
     return output.str();
 }
 
-[[maybe_unused]] std::string trim(const std::string& value) {
+std::string trim(const std::string& value) {
     std::size_t first = 0;
     while (first < value.size() &&
            std::isspace(static_cast<unsigned char>(value[first]))) {
@@ -38,7 +38,7 @@ std::string makeParseMessage(const std::string& source_name,
     return value.substr(first, last - first);
 }
 
-[[maybe_unused]] std::vector<std::string> splitTokens(const std::string& line) {
+std::vector<std::string> splitTokens(const std::string& line) {
     std::vector<std::string> tokens;
     std::istringstream input(line);
     std::string token;
@@ -207,6 +207,28 @@ double parseDoubleToken(const std::string& token,
                  static_cast<double>(channels[2]) / 255.0);
 }
 
+[[maybe_unused]] void rejectDuplicate(bool already_seen,
+                     const std::string& source_name,
+                     std::size_t line_number,
+                     const std::string& directive) {
+    if (already_seen) {
+        throw ParseError(source_name,
+                         line_number,
+                         "duplicate " + directive + " directive");
+    }
+}
+
+[[maybe_unused]] void requireNonzeroVector(const Vec3& value,
+                          const std::string& source_name,
+                          std::size_t line_number,
+                          const std::string& field_name) {
+    if (value.isNearZero()) {
+        throw ParseError(source_name,
+                         line_number,
+                         field_name + " must not be the zero vector");
+    }
+}
+
 }  // namespace
 
 ParseError::ParseError(const std::string& source_name,
@@ -225,4 +247,35 @@ std::size_t ParseError::line() const noexcept {
     return line_;
 }
 
+namespace parser {
+
+Scene parseScene(std::istream& input, const std::string& source_name) {
+    Scene scene;
+    std::string line;
+    std::size_t line_number = 0;
+
+    while (std::getline(input, line)) {
+        ++line_number;
+        const std::size_t comment = line.find('#');
+        if (comment != std::string::npos) {
+            line.erase(comment);
+        }
+        line = trim(line);
+        if (line.empty()) {
+            continue;
+        }
+
+        const std::vector<std::string> tokens = splitTokens(line);
+        const std::string& id = tokens[0];
+
+            throw ParseError(source_name,
+                             line_number,
+                             "unknown directive '" + id + "'");
+    }
+
+    return scene;
+}
+
+}  // namespace parser
+
 }  // namespace ray


## `feat(parser): 해상도와 환경광 지시어 지원`

diff --git a/src/parser.cpp b/src/parser.cpp
index a92f0bd..c865e2a 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -63,7 +63,7 @@ std::vector<std::string> splitCommas(const std::string& token) {
     return parts;
 }
 
-[[maybe_unused]] void expectCount(const std::vector<std::string>& tokens,
+void expectCount(const std::vector<std::string>& tokens,
                  std::size_t expected,
                  const std::string& source_name,
                  std::size_t line_number,
@@ -94,7 +94,7 @@ double parseDoubleToken(const std::string& token,
     }
 }
 
-[[maybe_unused]] int parseIntToken(const std::string& token,
+int parseIntToken(const std::string& token,
                   const std::string& source_name,
                   std::size_t line_number,
                   const std::string& field_name) {
@@ -114,7 +114,7 @@ double parseDoubleToken(const std::string& token,
     }
 }
 
-[[maybe_unused]] double parseRatio(const std::string& token,
+double parseRatio(const std::string& token,
                   const std::string& source_name,
                   std::size_t line_number,
                   const std::string& field_name) {
@@ -170,7 +170,7 @@ double parseDoubleToken(const std::string& token,
                          field_name + ".z"));
 }
 
-[[maybe_unused]] Color parseColor(const std::string& token,
+Color parseColor(const std::string& token,
                  const std::string& source_name,
                  std::size_t line_number,
                  const std::string& field_name) {
@@ -207,7 +207,7 @@ double parseDoubleToken(const std::string& token,
                  static_cast<double>(channels[2]) / 255.0);
 }
 
-[[maybe_unused]] void rejectDuplicate(bool already_seen,
+void rejectDuplicate(bool already_seen,
                      const std::string& source_name,
                      std::size_t line_number,
                      const std::string& directive) {
@@ -268,9 +268,53 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
         const std::vector<std::string> tokens = splitTokens(line);
         const std::string& id = tokens[0];
 
+        if (id == "R") {
+            expectCount(tokens,
+                        3,
+                        source_name,
+                        line_number,
+                        "R width height");
+            rejectDuplicate(scene.hasResolution,
+                            source_name,
+                            line_number,
+                            "R");
+            scene.width =
+                parseIntToken(tokens[1],
+                              source_name,
+                              line_number,
+                              "width");
+            scene.height =
+                parseIntToken(tokens[2],
+                              source_name,
+                              line_number,
+                              "height");
+            scene.hasResolution = true;
+        } else if (id == "A") {
+            expectCount(tokens,
+                        3,
+                        source_name,
+                        line_number,
+                        "A ratio r,g,b");
+            rejectDuplicate(scene.hasAmbient,
+                            source_name,
+                            line_number,
+                            "A");
+            scene.ambientRatio =
+                parseRatio(tokens[1],
+                           source_name,
+                           line_number,
+                           "ambient ratio");
+            scene.ambientColor =
+                parseColor(tokens[2],
+                           source_name,
+                           line_number,
+                           "ambient color");
+            scene.hasAmbient = true;
+        } else {
             throw ParseError(source_name,
                              line_number,
                              "unknown directive '" + id + "'");
+        }
     }
 
     return scene;


## `feat(parser): 카메라와 광원 지시어 지원`

diff --git a/src/parser.cpp b/src/parser.cpp
index c865e2a..7d50452 100644
--- a/src/parser.cpp
+++ b/src/parser.cpp
@@ -142,7 +142,7 @@ double parseRatio(const std::string& token,
     return value;
 }
 
-[[maybe_unused]] Vec3 parseVec3(const std::string& token,
+Vec3 parseVec3(const std::string& token,
                const std::string& source_name,
                std::size_t line_number,
                const std::string& field_name) {
@@ -218,7 +218,7 @@ void rejectDuplicate(bool already_seen,
     }
 }
 
-[[maybe_unused]] void requireNonzeroVector(const Vec3& value,
+void requireNonzeroVector(const Vec3& value,
                           const std::string& source_name,
                           std::size_t line_number,
                           const std::string& field_name) {
@@ -310,6 +310,65 @@ Scene parseScene(std::istream& input, const std::string& source_name) {
                            line_number,
                            "ambient color");
             scene.hasAmbient = true;
+        } else if (id == "C") {
+            expectCount(tokens,
+                        4,
+                        source_name,
+                        line_number,
+                        "C pos dir fov");
+            rejectDuplicate(scene.hasCamera,
+                            source_name,
+                            line_number,
+                            "C");
+            const Vec3 position =
+                parseVec3(tokens[1],
+                          source_name,
+                          line_number,
+                          "camera position");
+            const Vec3 direction =
+                parseVec3(tokens[2],
+                          source_name,
+                          line_number,
+                          "camera direction");
+            requireNonzeroVector(direction,
+                                 source_name,
+                                 line_number,
+                                 "camera direction");
+            const double fov =
+                parseDoubleToken(tokens[3],
+                                 source_name,
+                                 line_number,
+                                 "camera fov");
+            if (fov <= 0.0 || fov >= 180.0) {
+                throw ParseError(
+                    source_name,
+                    line_number,
+                    "camera fov must be greater than 0 and less than 180");
+            }
+            scene.camera = Camera(position, normalize(direction), fov);
+            scene.hasCamera = true;
+        } else if (id == "L") {
+            expectCount(tokens,
+                        4,
+                        source_name,
+                        line_number,
+                        "L pos brightness r,g,b");
+            const Vec3 position =
+                parseVec3(tokens[1],
+                          source_name,
+                          line_number,
+                          "light position");
+            const double brightness =
+                parseRatio(tokens[2],
+                           source_name,
+                           line_number,
+                           "light brightness");
+            const Color color =
+                parseColor(tokens[3],
+                           source_name,
+                           line_number,
+                           "light color");
+            scene.addLight(Light(position, brightness, color));
         } else {
             throw ParseError(source_name,
                              line_number,


