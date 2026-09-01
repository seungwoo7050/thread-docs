# 명령 수와 물리 이동 비용

## `test(sort): 큰 입력의 명령 수 상한을 검증`

diff --git a/tests/run_tests.py b/tests/run_tests.py
index e638f86..c324a5c 100644
--- a/tests/run_tests.py
+++ b/tests/run_tests.py
@@ -1,6 +1,7 @@
 #!/usr/bin/env python3
 from pathlib import Path
 import itertools
+import random
 import subprocess
 import sys
 import tempfile
@@ -259,12 +260,28 @@ def test_sort_programs():
             assert_sorted_by_program(list(values))
 
 
+def test_move_counts():
+    random.seed(4242)
+    limits = [
+        (100, 1500),
+        (500, 8000),
+    ]
+    for size, limit in limits:
+        values = random.sample(range(-10000, 10000), size)
+        moves = assert_sorted_by_program(values)
+        assert_ok(
+            len(moves) <= limit,
+            f"{size} value move count {len(moves)} exceeds {limit}",
+        )
+
+
 def main():
     test_parser_inputs()
     test_parser_boundaries()
     test_checker_without_values_does_not_read_stdin()
     test_checker_operations()
     test_sort_programs()
+    test_move_counts()
     print("tests passed")
 
 


## `test(sort): 결정적 다중 시드 동치 검사를 추가`

diff --git a/tests/run_tests.py b/tests/run_tests.py
index c324a5c..cad7743 100644
--- a/tests/run_tests.py
+++ b/tests/run_tests.py
@@ -1,15 +1,15 @@
 #!/usr/bin/env python3
 from pathlib import Path
 import itertools
-import random
+import os
 import subprocess
 import sys
 import tempfile
 
 
 ROOT = Path(__file__).resolve().parents[1]
-PUSH_SWAP = ROOT / "push_swap"
-CHECKER = ROOT / "checker"
+PUSH_SWAP = ROOT / os.environ.get("PS_PUSH_SWAP", "push_swap")
+CHECKER = ROOT / os.environ.get("PS_CHECKER", "checker")
 VALID_MOVES = {"sa", "sb", "ss", "pa", "pb", "ra", "rb", "rr", "rra", "rrb", "rrr"}
 CHILD_TIMEOUT_SECONDS = 5
 
@@ -240,6 +240,17 @@ def assert_sorted_by_program(values):
     return moves
 
 
+def deterministic_values(size, seed):
+    values = list(range(size))
+    state = seed & 0xFFFFFFFF
+    for index in range(size - 1, 0, -1):
+        state = (1664525 * state + 1013904223) & 0xFFFFFFFF
+        selected = state % (index + 1)
+        values[index], values[selected] = values[selected], values[index]
+    offset = size * 23
+    return [value * 37 - offset for value in values]
+
+
 def test_sort_programs():
     sorted_case = run([PUSH_SWAP, "1", "2", "3", "4", "5"])
     assert_equal(sorted_case.returncode, 0, "already sorted input exits cleanly")
@@ -261,18 +272,27 @@ def test_sort_programs():
 
 
 def test_move_counts():
-    random.seed(4242)
     limits = [
         (100, 1500),
         (500, 8000),
     ]
     for size, limit in limits:
-        values = random.sample(range(-10000, 10000), size)
-        moves = assert_sorted_by_program(values)
-        assert_ok(
-            len(moves) <= limit,
-            f"{size} value move count {len(moves)} exceeds {limit}",
-        )
+        for seed in (7, 4242, 9001):
+            values = deterministic_values(size, seed)
+            moves = assert_sorted_by_program(values)
+            assert_ok(
+                len(moves) <= limit,
+                f"{size} value move count {len(moves)} exceeds {limit}",
+            )
+
+
+def test_seeded_differential_properties():
+    for seed in (1, 7, 97, 4242, 9001):
+        for size in (2, 3, 5, 6, 17, 64):
+            values = deterministic_values(size, seed)
+            first = assert_sorted_by_program(values)
+            second = assert_sorted_by_program(values)
+            assert_equal(second, first, f"deterministic moves for seed {seed}, size {size}")
 
 
 def main():
@@ -281,6 +301,7 @@ def main():
     test_checker_without_values_does_not_read_stdin()
     test_checker_operations()
     test_sort_programs()
+    test_seeded_differential_properties()
     test_move_counts()
     print("tests passed")
 


## `test(resource): 명령과 배열 이동 및 할당량을 기준화`

diff --git a/Makefile b/Makefile
index def63e4..868a142 100644
--- a/Makefile
+++ b/Makefile
@@ -73,3 +73,4 @@ test: all $(OPERATION_TEST) $(FAULT_PUSH_SWAP) $(FAULT_CHECKER)
 	python3 tests/run_tests.py
 	PS_PUSH_SWAP=$(FAULT_PUSH_SWAP) PS_CHECKER=$(FAULT_CHECKER) \
 		python3 tests/fault_tests.py
+	PS_PUSH_SWAP=$(FAULT_PUSH_SWAP) python3 tests/resource_tests.py
diff --git a/include/push_swap.h b/include/push_swap.h
index bad20f3..a170918 100644
--- a/include/push_swap.h
+++ b/include/push_swap.h
@@ -51,6 +51,13 @@ ssize_t	ps_read(int fd, void *buffer, size_t count);
 int		ps_write_all(int fd, const void *buffer, size_t count);
 int		ps_ignore_sigpipe(void);
 int		ps_test_finish(int status);
+# ifdef PS_FAULT_INJECTION
+void	ps_record_operation(void);
+void	ps_record_movements(size_t count);
+# else
+#  define ps_record_operation() ((void)0)
+#  define ps_record_movements(count) ((void)0)
+# endif
 
 size_t	ps_strlen(const char *str);
 int		ps_strcmp(const char *a, const char *b);
diff --git a/src/operations.c b/src/operations.c
index 15f9963..54077e7 100644
--- a/src/operations.c
+++ b/src/operations.c
@@ -5,7 +5,11 @@
 static int	emit_op(const char *name, int emit)
 {
 	if (emit)
-		return (ps_putstr_fd(1, name));
+	{
+		if (!ps_putstr_fd(1, name))
+			return (0);
+		ps_record_operation();
+	}
 	return (1);
 }
 
@@ -22,6 +26,7 @@ void	stack_swap(t_stack *stack)
 	stack->ranks[0] = stack->ranks[1];
 	stack->values[1] = value;
 	stack->ranks[1] = rank;
+	ps_record_movements(2);
 }
 
 void	stack_push(t_stack *dst, t_stack *src)
@@ -46,6 +51,7 @@ void	stack_push(t_stack *dst, t_stack *src)
 		memmove(src->ranks, src->ranks + 1,
 			sizeof(int) * (size_t)src->size);
 	}
+	ps_record_movements((size_t)(dst->size + src->size));
 }
 
 void	stack_rotate(t_stack *stack)
@@ -63,6 +69,7 @@ void	stack_rotate(t_stack *stack)
 		sizeof(int) * (size_t)(stack->size - 1));
 	stack->values[stack->size - 1] = value;
 	stack->ranks[stack->size - 1] = rank;
+	ps_record_movements((size_t)stack->size);
 }
 
 void	stack_reverse_rotate(t_stack *stack)
@@ -80,6 +87,7 @@ void	stack_reverse_rotate(t_stack *stack)
 		sizeof(int) * (size_t)(stack->size - 1));
 	stack->values[0] = value;
 	stack->ranks[0] = rank;
+	ps_record_movements((size_t)stack->size);
 }
 
 int	op_sa(t_stack *a, int emit)
diff --git a/src/runtime.c b/src/runtime.c
index 4e0c971..9baf7cb 100644
--- a/src/runtime.c
+++ b/src/runtime.c
@@ -20,6 +20,10 @@ static unsigned long	g_malloc_calls;
 static unsigned long	g_read_calls;
 static unsigned long	g_write_calls;
 static unsigned long	g_live_allocations;
+static size_t			g_current_bytes;
+static size_t			g_peak_bytes;
+static size_t			g_operation_count;
+static size_t			g_array_movements;
 
 static unsigned long	read_index(const char *name)
 {
@@ -65,6 +69,9 @@ void	*ps_malloc(size_t size)
 	header->data.size = size;
 	header->data.magic = 0x50535354UL;
 	g_live_allocations++;
+	g_current_bytes += size;
+	if (g_current_bytes > g_peak_bytes)
+		g_peak_bytes = g_current_bytes;
 	return ((void *)(header + 1));
 #else
 	return (malloc(size));
@@ -83,6 +90,7 @@ void	ps_free(void *pointer)
 	{
 		header->data.magic = 0;
 		g_live_allocations--;
+		g_current_bytes -= header->data.size;
 	}
 	free(header);
 #else
@@ -142,6 +150,22 @@ int	ps_ignore_sigpipe(void)
 	return (signal(SIGPIPE, SIG_IGN) != SIG_ERR);
 }
 
+#ifdef PS_FAULT_INJECTION
+void	ps_record_operation(void)
+{
+	if (g_operation_count != (size_t)-1)
+		g_operation_count++;
+}
+
+void	ps_record_movements(size_t count)
+{
+	if (count > (size_t)-1 - g_array_movements)
+		g_array_movements = (size_t)-1;
+	else
+		g_array_movements += count;
+}
+#endif
+
 #ifdef PS_FAULT_INJECTION
 static void	raw_report(const char *message)
 {
@@ -162,6 +186,35 @@ static void	raw_report(const char *message)
 		length -= (size_t)written;
 	}
 }
+
+static void	raw_report_number(const char *label, size_t value)
+{
+	char	digits[3 * sizeof(size_t) + 1];
+	char	temporary;
+	size_t	length;
+	size_t	index;
+
+	raw_report(label);
+	length = 0;
+	if (value == 0)
+		digits[length++] = '0';
+	while (value > 0)
+	{
+		digits[length++] = (char)('0' + value % 10);
+		value /= 10;
+	}
+	index = 0;
+	while (index < length / 2)
+	{
+		temporary = digits[index];
+		digits[index] = digits[length - index - 1];
+		digits[length - index - 1] = temporary;
+		index++;
+	}
+	digits[length++] = '\n';
+	digits[length] = '\0';
+	raw_report(digits);
+}
 #endif
 
 int	ps_test_finish(int status)
@@ -177,6 +230,12 @@ int	ps_test_finish(int status)
 			return (99);
 		}
 	}
+	if (getenv("PS_REPORT_METRICS") != NULL)
+	{
+		raw_report_number("PS_OPERATIONS=", g_operation_count);
+		raw_report_number("PS_ARRAY_MOVEMENTS=", g_array_movements);
+		raw_report_number("PS_PEAK_BYTES=", g_peak_bytes);
+	}
 #endif
 	return (status);
 }
diff --git a/tests/fault_tests.py b/tests/fault_tests.py
index 164d87a..e477cd9 100644
--- a/tests/fault_tests.py
+++ b/tests/fault_tests.py
@@ -27,7 +27,9 @@ def assert_true(condition, message):
 
 
 def run(binary, args, input_bytes=b"", faults=None):
-    environment = os.environ.copy()
+    environment = {
+        name: value for name, value in os.environ.items() if not name.startswith("PS_")
+    }
     environment["PS_REPORT_ALLOCATIONS"] = "1"
     if faults:
         environment.update({key: str(value) for key, value in faults.items()})
@@ -187,7 +189,9 @@ def test_write_failures():
 
     read_fd, write_fd = os.pipe()
     os.close(read_fd)
-    environment = os.environ.copy()
+    environment = {
+        name: value for name, value in os.environ.items() if not name.startswith("PS_")
+    }
     environment["PS_REPORT_ALLOCATIONS"] = "1"
     process = subprocess.Popen(
         [str(PUSH_SWAP), "3", "2", "1"],
diff --git a/tests/resource_baseline.json b/tests/resource_baseline.json
new file mode 100644
index 0000000..6566a51
--- /dev/null
+++ b/tests/resource_baseline.json
@@ -0,0 +1,17 @@
+{
+  "measurement": {
+    "array_movements": "이동하거나 다시 쓴 (value, rank) 쌍의 수",
+    "peak_bytes": "계측 헤더를 제외하고 동시에 살아 있는 프로젝트 할당 요청의 합"
+  },
+  "cases": [
+    {"size": 10, "seed": 7, "commands": 65, "max_array_movements": 650, "max_peak_bytes": 160},
+    {"size": 10, "seed": 4242, "commands": 65, "max_array_movements": 650, "max_peak_bytes": 160},
+    {"size": 10, "seed": 9001, "commands": 65, "max_array_movements": 650, "max_peak_bytes": 160},
+    {"size": 100, "seed": 7, "commands": 1084, "max_array_movements": 105000, "max_peak_bytes": 1600},
+    {"size": 100, "seed": 4242, "commands": 1084, "max_array_movements": 105000, "max_peak_bytes": 1600},
+    {"size": 100, "seed": 9001, "commands": 1084, "max_array_movements": 105000, "max_peak_bytes": 1600},
+    {"size": 500, "seed": 7, "commands": 6784, "max_array_movements": 3200000, "max_peak_bytes": 8000},
+    {"size": 500, "seed": 4242, "commands": 6784, "max_array_movements": 3200000, "max_peak_bytes": 8000},
+    {"size": 500, "seed": 9001, "commands": 6784, "max_array_movements": 3200000, "max_peak_bytes": 8000}
+  ]
+}
diff --git a/tests/resource_tests.py b/tests/resource_tests.py
new file mode 100644
index 0000000..797abc1
--- /dev/null
+++ b/tests/resource_tests.py
@@ -0,0 +1,99 @@
+#!/usr/bin/env python3
+import json
+import os
+from pathlib import Path
+import subprocess
+import sys
+import time
+
+
+ROOT = Path(__file__).resolve().parents[1]
+PUSH_SWAP = ROOT / os.environ.get("PS_PUSH_SWAP", ".build/fault/push_swap")
+BASELINE = ROOT / "tests" / "resource_baseline.json"
+TIMEOUT_SECONDS = 5
+
+
+def fail(message):
+    print(message, file=sys.stderr)
+    raise SystemExit(1)
+
+
+def deterministic_values(size, seed):
+    values = list(range(size))
+    state = seed & 0xFFFFFFFF
+    for index in range(size - 1, 0, -1):
+        state = (1664525 * state + 1013904223) & 0xFFFFFFFF
+        selected = state % (index + 1)
+        values[index], values[selected] = values[selected], values[index]
+    offset = size * 23
+    return [value * 37 - offset for value in values]
+
+
+def parse_metrics(stderr):
+    metrics = {}
+    for line in stderr.decode("ascii").splitlines():
+        if line.startswith("PS_") and "=" in line:
+            name, value = line.split("=", 1)
+            metrics[name] = int(value)
+    return metrics
+
+
+def check_case(case):
+    values = deterministic_values(case["size"], case["seed"])
+    environment = {
+        name: value for name, value in os.environ.items() if not name.startswith("PS_")
+    }
+    environment["PS_REPORT_ALLOCATIONS"] = "1"
+    environment["PS_REPORT_METRICS"] = "1"
+    started = time.perf_counter()
+    try:
+        result = subprocess.run(
+            [str(PUSH_SWAP), *[str(value) for value in values]],
+            capture_output=True,
+            cwd=ROOT,
+            env=environment,
+            timeout=TIMEOUT_SECONDS,
+            check=False,
+        )
+    except subprocess.TimeoutExpired:
+        fail(f"resource case timed out: {case!r}")
+    elapsed_ms = (time.perf_counter() - started) * 1000
+    if result.returncode != 0:
+        fail(f"resource case failed: {case!r}, stderr={result.stderr!r}")
+    metrics = parse_metrics(result.stderr)
+    expected_keys = {
+        "PS_LIVE_ALLOCATIONS",
+        "PS_OPERATIONS",
+        "PS_ARRAY_MOVEMENTS",
+        "PS_PEAK_BYTES",
+    }
+    if set(metrics) != expected_keys:
+        fail(f"incomplete resource metrics for {case!r}: {metrics!r}")
+    commands = len(result.stdout.splitlines())
+    if metrics["PS_LIVE_ALLOCATIONS"] != 0:
+        fail(f"resource case leaked allocations: {case!r}")
+    if commands != metrics["PS_OPERATIONS"]:
+        fail(f"emitted and recorded command counts differ: {case!r}")
+    if commands != case["commands"]:
+        fail(f"command count changed: {case!r}, actual={commands}")
+    if metrics["PS_ARRAY_MOVEMENTS"] > case["max_array_movements"]:
+        fail(f"array movement budget exceeded: {case!r}, metrics={metrics!r}")
+    if metrics["PS_PEAK_BYTES"] > case["max_peak_bytes"]:
+        fail(f"peak allocation budget exceeded: {case!r}, metrics={metrics!r}")
+    print(
+        f"resource size={case['size']} seed={case['seed']} "
+        f"commands={commands} movements={metrics['PS_ARRAY_MOVEMENTS']} "
+        f"peak_bytes={metrics['PS_PEAK_BYTES']} elapsed_ms={elapsed_ms:.2f}"
+    )
+
+
+def main():
+    with BASELINE.open(encoding="utf-8") as baseline_file:
+        baseline = json.load(baseline_file)
+    for case in baseline["cases"]:
+        check_case(case)
+    print("resource regression tests passed; elapsed time is informational")
+
+
+if __name__ == "__main__":
+    main()
