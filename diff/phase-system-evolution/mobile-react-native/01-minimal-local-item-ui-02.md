## `test: verify M01 sequence on actual Android controls`

diff --git a/android/app/src/androidTest/java/com/mse/reactnative/M01ItemUiTest.java b/android/app/src/androidTest/java/com/mse/reactnative/M01ItemUiTest.java
new file mode 100644
index 0000000..bab8242
--- /dev/null
+++ b/android/app/src/androidTest/java/com/mse/reactnative/M01ItemUiTest.java
@@ -0,0 +1,62 @@
+package com.mse.reactnative;
+
+import androidx.test.core.app.ActivityScenario;
+import androidx.test.ext.junit.runners.AndroidJUnit4;
+import androidx.test.platform.app.InstrumentationRegistry;
+import androidx.test.uiautomator.By;
+import androidx.test.uiautomator.UiDevice;
+import androidx.test.uiautomator.UiObject2;
+import androidx.test.uiautomator.Until;
+import org.junit.Test;
+import org.junit.runner.RunWith;
+import static org.junit.Assert.*;
+
+@RunWith(AndroidJUnit4.class)
+public class M01ItemUiTest {
+    private final UiDevice device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
+
+    private UiObject2 control(String label) {
+        UiObject2 object = device.wait(Until.findObject(By.desc(label)), 15000);
+        assertNotNull("Missing accessible control: " + label, object);
+        return object;
+    }
+
+    private void enterTitle(String label, String value) {
+        UiObject2 input = control(label);
+        input.click();
+        input.setText(value);
+        device.pressBack();
+    }
+
+    @Test
+    public void fixedM01SequenceOnRealReactNativeUi() {
+        try (ActivityScenario<MainActivity> ignored = ActivityScenario.launch(MainActivity.class)) {
+            control("Item count: 0");
+            enterTitle("New item title", "Alpha");
+            control("Add item").click();
+            control("Item count: 1");
+            enterTitle("New item title", "Beta");
+            control("Add item").click();
+            control("Item count: 2");
+
+            control("Edit Alpha").click();
+            enterTitle("Edit item title", "Alpha edited");
+            control("Save title").click();
+            assertTrue(device.wait(Until.hasObject(By.text("Alpha edited")), 15000));
+
+            control("Mark Alpha edited complete").click();
+            assertTrue(control("Mark Alpha edited incomplete").isChecked());
+            control("Mark Alpha edited incomplete").click();
+            assertFalse(control("Mark Alpha edited complete").isChecked());
+            control("Mark Alpha edited complete").click();
+            assertTrue(control("Mark Alpha edited incomplete").isChecked());
+            control("Delete Beta").click();
+
+            control("Item count: 1");
+            assertTrue(device.hasObject(By.text("Alpha edited")));
+            assertTrue(control("Mark Alpha edited incomplete").isChecked());
+            assertFalse(device.hasObject(By.text("Beta")));
+            assertFalse(device.hasObject(By.desc("Delete Beta")));
+        }
+    }
+}
diff --git a/verification/M01.md b/verification/M01.md
new file mode 100644
index 0000000..601982e
--- /dev/null
+++ b/verification/M01.md
@@ -0,0 +1,74 @@
+# M01 verification — attempt 1
+
+- Thread: M01
+- Branch: track/react-native
+- Spec-Revision: ed7baa246ee947c6778e0f84751c9b91cec7abfe
+- Start: UNBORN
+- Android environment: API 34, Pixel 6, ARM64, emulator-5554 (the shared fixed device).
+- No other device/API, benchmark, battery soak, or scenario tuning was used.
+
+## Frozen sequence
+
+Start with zero Items. Create Alpha, create Beta, rename Alpha to Alpha edited,
+complete the first Item, uncomplete it, complete it again, then delete Beta.
+The final list must contain exactly the first Item with its original identity,
+title Alpha edited, and completed=true. Domain IDs are item-001 and item-002.
+Host clock starts at 1700000000000 and advances 1000 ms per edit. Version is only
+a local edit counter in M01. The expected surviving Item has version 5 and
+updatedAt 1700000005000. No network/failure fixture applies in this Thread.
+
+## Commands and results
+
+All commands use Node 22.22.0; Android commands also use JDK 17 and SDK platform
+35/build-tools 35.0.0. `JAVA_HOME`, `ANDROID_HOME`, and `GRADLE_USER_HOME` are local
+environment variables, not checked-in machine paths. Android build commands run
+from `android/`; npm commands run from the branch root. Raw output is retained in
+the orchestration evidence directory `react-native/M01/` using the filenames below.
+
+| Invocation | Command | Exit | Executed tests | Raw log |
+|---|---|---:|---|---|
+| Host 1 | `npm run typecheck` | 0 | TypeScript compile check; 0 runtime tests | typecheck-01.log |
+| Host 2 | `npm test -- --runInBand --watchman=false` | 0 | 2 suites, 3 tests passed | jest-01.log |
+| Build 1 | `./gradlew --no-daemon :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 0 runtime tests; 80 build tasks executed, both APKs compiled | build-01.log |
+| Android 1 | `./gradlew --no-daemon :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=com.mse.reactnative.M01ItemUiTest` | 0 | 1 test passed; 0 failures, 0 skipped, on the fixed device | android-01.log |
+
+Dependency setup: `npm install --ignore-scripts --no-audit --no-fund` completed
+with exit 0 (750 packages). The Gradle 8.10.2 wrapper checks the published
+distribution SHA-256. Dependency deprecation warnings are recorded in
+`npm-install.log`; no unrelated dependency modernization was attempted.
+The APK build emitted upstream Hermes-global warnings and packaged prebuilt native
+libraries without stripping symbols because no NDK is installed; neither warning
+prevented compilation. No app-specific native code or CMake/NDK toolchain was added.
+
+The Android device lease was explicitly granted before the test and released after
+completion. Exactly one actual Android scenario ran in this implementation attempt;
+there were no test or build failures. The Gradle run completed 81 tasks (5 executed,
+76 up-to-date). The full Android result directory, including JUnit XML and test
+logcat, is preserved as `android-results-01/` in the external evidence directory.
+
+The tested app APK contains `assets/index.android.bundle` (808908 bytes). SHA-256:
+
+- app-debug.apk: `7b93afdcd3e45104d13451549c04dfbb13db8779a76fa302325b0d0b3166b284`
+- app-debug-androidTest.apk: `7259ba83ec046d34a3e489ebec46ddcf9ce89dd136a934784c0c89669795b7c1`
+
+## Evidence boundaries
+
+The domain test exercises every fixed mutation and verifies stable identity,
+both toggle directions, required Item fields, and final state. A separate React
+Native component test repeats the sequence through accessible controls and checks
+the rendered row retains item-001 while item-002 disappears. The small blank-title
+test covers rejected create/rename input.
+
+Android instrumentation uses AndroidX UI Automator against actual React Native
+controls, including checked state and the final count/title/deletion. It does not
+replace Android verification with a Jest renderer. Every debug APK embeds the JS
+bundle and Hermes bytecode and disables developer support, removing Metro timing
+or public-network dependencies from the scenario.
+
+No durability or process-recreation guarantee is claimed by M01.
+
+## Handoff
+
+All M01 acceptance checks above passed in attempt 1. No unresolved M01 behavior
+remains. The main orchestrator must independently verify commits and rerun tests
+before creating a progress tag; this implementation subagent creates no tag.
