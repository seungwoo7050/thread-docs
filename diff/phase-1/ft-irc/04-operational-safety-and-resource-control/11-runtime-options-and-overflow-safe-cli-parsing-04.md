## `fix(config): 서버 크기 옵션을 오버플로 없이 해석`

diff --git a/src/RuntimeConfig.cpp b/src/RuntimeConfig.cpp
index 982e41a..7daa84a 100644
--- a/src/RuntimeConfig.cpp
+++ b/src/RuntimeConfig.cpp
@@ -1,9 +1,37 @@
 #include "RuntimeConfig.hpp"
 
-#include <cstdlib>
 #include <iostream>
+#include <limits>
 #include <stdexcept>
 
+namespace {
+
+template <typename Unsigned>
+Unsigned parseUnsignedDecimal(const std::string& value,
+                              Unsigned maximum,
+                              const std::string& errorMessage) {
+    if (value.empty()) {
+        throw std::runtime_error(errorMessage);
+    }
+
+    Unsigned parsed = 0;
+    for (std::string::size_type i = 0; i < value.size(); ++i) {
+        if (value[i] < '0' || value[i] > '9') {
+            throw std::runtime_error(errorMessage);
+        }
+
+        const Unsigned digit = static_cast<Unsigned>(value[i] - '0');
+        if (parsed > maximum / 10
+            || (parsed == maximum / 10 && digit > maximum % 10)) {
+            throw std::runtime_error(errorMessage);
+        }
+        parsed = static_cast<Unsigned>(parsed * 10 + digit);
+    }
+    return parsed;
+}
+
+} // namespace
+
 RuntimeConfig::RuntimeConfig()
     : rateLimitCount(24),
       rateLimitWindowSeconds(3),
@@ -20,9 +48,12 @@ void RuntimeConfig::printUsage(const char* programName) {
 }
 
 int RuntimeConfig::parsePort(const char* value) {
-    char* end = NULL;
-    const long port = std::strtol(value, &end, 10);
-    if (!value[0] || *end != '\0' || port <= 0 || port > 65535) {
+    const std::string text(value == NULL ? "" : value);
+    const unsigned short port = parseUnsignedDecimal<unsigned short>(
+        text,
+        std::numeric_limits<unsigned short>::max(),
+        "port must be an integer from 1 to 65535");
+    if (port == 0) {
         throw std::runtime_error("port must be an integer from 1 to 65535");
     }
     return static_cast<int>(port);
@@ -59,18 +90,18 @@ RuntimeConfig RuntimeConfig::parseOptions(int argc, char** argv, Server::Config&
 }
 
 std::size_t RuntimeConfig::parseSize(const std::string& value, const std::string& name) {
-    char* end = NULL;
-    const unsigned long parsed = std::strtoul(value.c_str(), &end, 10);
-    if (value.empty() || *end != '\0') {
-        throw std::runtime_error(name + " must be an unsigned integer");
-    }
-    return static_cast<std::size_t>(parsed);
+    return parseUnsignedDecimal<std::size_t>(
+        value,
+        std::numeric_limits<std::size_t>::max(),
+        name + " must be an unsigned integer");
 }
 
 int RuntimeConfig::parsePositiveInt(const std::string& value, const std::string& name) {
-    char* end = NULL;
-    const long parsed = std::strtol(value.c_str(), &end, 10);
-    if (value.empty() || *end != '\0' || parsed <= 0 || parsed > 86400) {
+    const unsigned int parsed = parseUnsignedDecimal<unsigned int>(
+        value,
+        86400U,
+        name + " must be a positive integer");
+    if (parsed == 0) {
         throw std::runtime_error(name + " must be a positive integer");
     }
     return static_cast<int>(parsed);


## `test(config): 크기 옵션 경계와 오류 입력 검증`

diff --git a/tests/irc_contract.py b/tests/irc_contract.py
index 48f946a..0343d04 100644
--- a/tests/irc_contract.py
+++ b/tests/irc_contract.py
@@ -6,6 +6,7 @@ import os
 from pathlib import Path
 import re
 import signal
+import struct
 import subprocess
 import sys
 import time
@@ -86,6 +87,20 @@ def check_cli_contract(manifest: Dict[str, object], binary: str) -> None:
         ["0", "contract-secret"],
         expected_stderr="irc-relay-server: port must be an integer from 1 to 65535\n",
     )
+    for label, port in (
+        ("signed_port", "+6667"),
+        ("negative_port", "-6667"),
+        ("leading_space_port", " 6667"),
+        ("trailing_space_port", "6667 "),
+        ("port_narrowing_overflow", "65536"),
+    ):
+        check_cli(
+            manifest,
+            binary,
+            label,
+            [port, "contract-secret"],
+            expected_stderr="irc-relay-server: port must be an integer from 1 to 65535\n",
+        )
     check_cli(
         manifest,
         binary,
@@ -93,6 +108,20 @@ def check_cli_contract(manifest: Dict[str, object], binary: str) -> None:
         ["6667", "contract-secret", "--idle-timeout=0"],
         expected_stderr="irc-relay-server: idle timeout must be a positive integer\n",
     )
+    for label, value in (
+        ("signed_timeout", "+1"),
+        ("negative_timeout", "-1"),
+        ("leading_space_timeout", " 1"),
+        ("trailing_space_timeout", "1 "),
+        ("timeout_overflow", "999999999999999999999999999999999999999"),
+    ):
+        check_cli(
+            manifest,
+            binary,
+            label,
+            ["6667", "contract-secret", f"--idle-timeout={value}"],
+            expected_stderr="irc-relay-server: idle timeout must be a positive integer\n",
+        )
     check_cli(
         manifest,
         binary,
@@ -114,6 +143,32 @@ def check_cli_contract(manifest: Dict[str, object], binary: str) -> None:
         ["6667", "contract-secret", "--max-pending-bytes=abc"],
         stderr_prefix="irc-relay-server: max pending bytes must be an unsigned integer",
     )
+    size_t_overflow = str(1 << (struct.calcsize("P") * 8))
+    for label, option, expected_name in (
+        ("signed_size", "--max-connections=+1", "max connections"),
+        ("negative_size", "--max-connections=-1", "max connections"),
+        ("leading_space_size", "--max-connections= 1", "max connections"),
+        ("trailing_space_size", "--max-connections=1 ", "max connections"),
+        (
+            "size_t_narrowing_overflow",
+            f"--max-pending-bytes={size_t_overflow}",
+            "max pending bytes",
+        ),
+        (
+            "rate_count_overflow",
+            f"--rate-limit={size_t_overflow}:1",
+            "rate limit count",
+        ),
+    ):
+        check_cli(
+            manifest,
+            binary,
+            label,
+            ["6667", "contract-secret", option],
+            expected_stderr=(
+                f"irc-relay-server: {expected_name} must be an unsigned integer\n"
+            ),
+        )
 
 
 def record_exact(
