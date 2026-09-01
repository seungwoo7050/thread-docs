# M01 — 최소 Local Item UI

## `feat(android): establish memory-only Item CRUD`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..c48dbf7
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,6 @@
+.gradle/
+**/build/
+local.properties
+.idea/
+*.iml
+.DS_Store
diff --git a/SPEC_REVISION b/SPEC_REVISION
new file mode 100644
index 0000000..81ee29e
--- /dev/null
+++ b/SPEC_REVISION
@@ -0,0 +1 @@
+ed7baa246ee947c6778e0f84751c9b91cec7abfe
diff --git a/TRACK.md b/TRACK.md
new file mode 100644
index 0000000..4714f61
--- /dev/null
+++ b/TRACK.md
@@ -0,0 +1,51 @@
+# Android Kotlin — Offline Item Tracker
+
+Independent orphan branch: `track/android-kotlin`.
+The fixed specification revision is recorded in `SPEC_REVISION` and every commit trailer.
+
+## M01 boundary
+
+One Compose screen creates Items, edits their titles, toggles completion, deletes Items,
+and renders the remaining list. `MemoryItemStore` contains only immutable list snapshots
+held in process memory. There is no disk storage, network access, synchronization,
+background task, or global state framework. `version` is always zero and carries no
+synchronization semantics. Activity recreation and process restart may clear everything;
+this Thread makes no persistence or restoration guarantee.
+
+The list uses the same `toRows()` mapping checked by the domain test. Controls have text
+labels/content descriptions; row test tags preserve Item identity through edits.
+
+## Pinned build
+
+- JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
+- Android Gradle Plugin 8.6.1, Kotlin 1.9.24, Compose compiler 1.5.14
+- Compose BOM 2024.06.00, Activity Compose 1.9.0
+- Compile SDK 35, target SDK 34, minimum SDK 26
+- Required verification device: API 34, Pixel 6, ARM64 Google APIs image, serial `emulator-5554`
+
+AGP/Gradle/JDK compatibility follows the [official AGP 8.6 release notes](https://developer.android.com/build/releases/agp-8-6-0-release-notes).
+The Kotlin/compiler pair follows the [official Compose compatibility table](https://developer.android.com/jetpack/androidx/releases/compose-kotlin).
+
+Set `JAVA_HOME` to a JDK 17 installation and `ANDROID_HOME` to the Android SDK. The SDK
+must have platform 35 and Build Tools 34.0.0; the shared verification emulator uses API 34.
+Keep machine paths in the environment or ignored `local.properties`, never in Git.
+
+```sh
+./gradlew --no-daemon :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest
+ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTest
+```
+
+The second command installs the app and instrumentation APK and runs the actual Compose
+UI sequence. It requires the shared emulator lease. The application package is
+`com.mobilesystemsevolution.kotlin`; the test package adds `.test`.
+
+## Frozen M01 sequence
+
+Start empty; create Alpha; create Beta; rename Alpha to Alpha edited; mark it completed,
+incomplete, then completed again; delete Beta. Exactly the first Item remains, with its
+original identity, title Alpha edited, and completed state true.
+
+The host test uses IDs `item-001`, `item-002` and a clock starting at 1700000000000 ms,
+advancing by 1000 ms for each subsequent edit. Android instrumentation uses actual
+screen interactions and captures the first row's identity tag before editing it.
+Invocation results and evidence are recorded in `verification/M01.md`.
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
new file mode 100644
index 0000000..e8aaf67
--- /dev/null
+++ b/app/build.gradle.kts
@@ -0,0 +1,41 @@
+plugins {
+    id("com.android.application")
+    id("org.jetbrains.kotlin.android")
+}
+
+android {
+    namespace = "com.mobilesystemsevolution.kotlin"
+    compileSdk = 35
+
+    defaultConfig {
+        applicationId = "com.mobilesystemsevolution.kotlin"
+        minSdk = 26
+        targetSdk = 34
+        versionCode = 1
+        versionName = "0.1"
+        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
+    }
+
+    buildFeatures { compose = true }
+    composeOptions { kotlinCompilerExtensionVersion = "1.5.14" }
+    compileOptions {
+        sourceCompatibility = JavaVersion.VERSION_17
+        targetCompatibility = JavaVersion.VERSION_17
+    }
+    kotlinOptions { jvmTarget = "17" }
+    packaging.resources.excludes += "/META-INF/{AL2.0,LGPL2.1}"
+}
+
+dependencies {
+    implementation(platform("androidx.compose:compose-bom:2024.06.00"))
+    implementation("androidx.activity:activity-compose:1.9.0")
+    implementation("androidx.compose.material3:material3")
+    implementation("androidx.compose.ui:ui")
+
+    testImplementation("junit:junit:4.13.2")
+    androidTestImplementation(platform("androidx.compose:compose-bom:2024.06.00"))
+    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
+    androidTestImplementation("androidx.test.ext:junit:1.2.1")
+    androidTestImplementation("androidx.test:runner:1.6.2")
+    debugImplementation("androidx.compose.ui:ui-test-manifest")
+}
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
new file mode 100644
index 0000000..8a6b52f
--- /dev/null
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/ItemUiTest.kt
@@ -0,0 +1,68 @@
+package com.mobilesystemsevolution.kotlin
+
+import androidx.compose.ui.semantics.SemanticsProperties
+import androidx.compose.ui.semantics.getOrNull
+import androidx.compose.ui.test.SemanticsMatcher
+import androidx.compose.ui.test.assertCountEquals
+import androidx.compose.ui.test.assertIsDisplayed
+import androidx.compose.ui.test.assertIsOff
+import androidx.compose.ui.test.assertIsOn
+import androidx.compose.ui.test.assertTextEquals
+import androidx.compose.ui.test.junit4.createAndroidComposeRule
+import androidx.compose.ui.test.onNodeWithContentDescription
+import androidx.compose.ui.test.onNodeWithTag
+import androidx.compose.ui.test.onNodeWithText
+import androidx.compose.ui.test.performClick
+import androidx.compose.ui.test.performTextInput
+import androidx.compose.ui.test.performTextReplacement
+import androidx.test.ext.junit.runners.AndroidJUnit4
+import org.junit.Rule
+import org.junit.Test
+import org.junit.runner.RunWith
+
+@RunWith(AndroidJUnit4::class)
+class ItemUiTest {
+    @get:Rule
+    val compose = createAndroidComposeRule<MainActivity>()
+
+    @Test
+    fun frozenM01SequenceThroughActualAndroidUi() {
+        val rowMatcher = SemanticsMatcher("Item row") {
+            it.config.getOrNull(SemanticsProperties.TestTag)?.startsWith("item-row-") == true
+        }
+        compose.onNodeWithTag("item-count").assertTextEquals("Items (0)")
+        compose.onNodeWithText("No items").assertIsDisplayed()
+
+        compose.onNodeWithTag("item-title-input").performTextInput("Alpha")
+        compose.onNodeWithText("Add").performClick()
+        compose.onNodeWithText("Alpha").assertIsDisplayed()
+        val firstRowTag = compose.onAllNodes(rowMatcher).fetchSemanticsNodes().single()
+            .config[SemanticsProperties.TestTag]
+
+        compose.onNodeWithTag("item-title-input").performTextInput("Beta")
+        compose.onNodeWithText("Add").performClick()
+        compose.onNodeWithText("Beta").assertIsDisplayed()
+        compose.onAllNodes(rowMatcher).assertCountEquals(2)
+
+        compose.onNodeWithContentDescription("Edit Alpha").performClick()
+        compose.onNodeWithTag("item-title-input").performTextReplacement("Alpha edited")
+        compose.onNodeWithText("Save").performClick()
+        compose.onNodeWithText("Alpha edited").assertIsDisplayed()
+        compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
+
+        val completed = compose.onNodeWithContentDescription("Completed: Alpha edited")
+        completed.assertIsOff().performClick().assertIsOn()
+        completed.performClick().assertIsOff()
+        completed.performClick().assertIsOn()
+
+        compose.onNodeWithContentDescription("Delete Beta").performClick()
+        compose.onNodeWithTag("item-count").assertTextEquals("Items (1)")
+        compose.onAllNodes(rowMatcher).assertCountEquals(1)
+        compose.onNodeWithTag(firstRowTag).assertIsDisplayed()
+        compose.onNodeWithText("Alpha edited").assertIsDisplayed()
+        compose.onNodeWithText("Completed").assertIsDisplayed()
+        completed.assertIsOn()
+        compose.onNodeWithText("Beta").assertDoesNotExist()
+        compose.onNodeWithText("Alpha").assertDoesNotExist()
+    }
+}
diff --git a/app/src/main/AndroidManifest.xml b/app/src/main/AndroidManifest.xml
new file mode 100644
index 0000000..3fa0c3c
--- /dev/null
+++ b/app/src/main/AndroidManifest.xml
@@ -0,0 +1,15 @@
+<?xml version="1.0" encoding="utf-8"?>
+<manifest xmlns:android="http://schemas.android.com/apk/res/android">
+    <application
+        android:allowBackup="false"
+        android:label="Offline Item Tracker"
+        android:supportsRtl="true"
+        android:theme="@android:style/Theme.Material.Light.NoActionBar">
+        <activity android:name=".MainActivity" android:exported="true">
+            <intent-filter>
+                <action android:name="android.intent.action.MAIN" />
+                <category android:name="android.intent.category.LAUNCHER" />
+            </intent-filter>
+        </activity>
+    </application>
+</manifest>
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
new file mode 100644
index 0000000..8c32f83
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -0,0 +1,125 @@
+package com.mobilesystemsevolution.kotlin
+
+import android.os.Bundle
+import androidx.activity.ComponentActivity
+import androidx.activity.compose.setContent
+import androidx.compose.foundation.layout.Arrangement
+import androidx.compose.foundation.layout.Column
+import androidx.compose.foundation.layout.Row
+import androidx.compose.foundation.layout.fillMaxSize
+import androidx.compose.foundation.layout.fillMaxWidth
+import androidx.compose.foundation.layout.padding
+import androidx.compose.foundation.rememberScrollState
+import androidx.compose.foundation.verticalScroll
+import androidx.compose.material3.Button
+import androidx.compose.material3.Checkbox
+import androidx.compose.material3.MaterialTheme
+import androidx.compose.material3.OutlinedTextField
+import androidx.compose.material3.Text
+import androidx.compose.material3.TextButton
+import androidx.compose.runtime.Composable
+import androidx.compose.runtime.getValue
+import androidx.compose.runtime.mutableStateOf
+import androidx.compose.runtime.remember
+import androidx.compose.runtime.setValue
+import androidx.compose.ui.Alignment
+import androidx.compose.ui.Modifier
+import androidx.compose.ui.platform.LocalFocusManager
+import androidx.compose.ui.platform.testTag
+import androidx.compose.ui.semantics.contentDescription
+import androidx.compose.ui.semantics.semantics
+import androidx.compose.ui.unit.dp
+
+class MainActivity : ComponentActivity() {
+    private val store = MemoryItemStore()
+
+    override fun onCreate(savedInstanceState: Bundle?) {
+        super.onCreate(savedInstanceState)
+        setContent { MaterialTheme { ItemScreen(store) } }
+    }
+}
+
+@Composable
+private fun ItemScreen(store: MemoryItemStore) {
+    var items by remember { mutableStateOf(store.items) }
+    var title by remember { mutableStateOf("") }
+    var editingId by remember { mutableStateOf<String?>(null) }
+    val focus = LocalFocusManager.current
+    val rows = items.toRows()
+
+    Column(
+        modifier = Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
+        verticalArrangement = Arrangement.spacedBy(8.dp),
+    ) {
+        Text("Offline Item Tracker", style = MaterialTheme.typography.headlineSmall)
+        OutlinedTextField(
+            value = title,
+            onValueChange = { title = it },
+            label = { Text("Item title") },
+            singleLine = true,
+            modifier = Modifier.fillMaxWidth().testTag("item-title-input"),
+        )
+        Row {
+            Button(
+                enabled = title.isNotBlank(),
+                onClick = {
+                    val id = editingId
+                    if (id == null) store.create(title) else store.rename(id, title)
+                    items = store.items
+                    title = ""
+                    editingId = null
+                    focus.clearFocus()
+                },
+            ) { Text(if (editingId == null) "Add" else "Save") }
+            if (editingId != null) {
+                TextButton(onClick = {
+                    title = ""
+                    editingId = null
+                    focus.clearFocus()
+                }) { Text("Cancel") }
+            }
+        }
+        Text("Items (${rows.size})", modifier = Modifier.testTag("item-count"))
+        if (rows.isEmpty()) Text("No items")
+        rows.forEach { row ->
+            Column(Modifier.fillMaxWidth().testTag(row.tag)) {
+                Text(row.title, style = MaterialTheme.typography.titleMedium)
+                Row(verticalAlignment = Alignment.CenterVertically) {
+                    Checkbox(
+                        checked = row.completed,
+                        onCheckedChange = {
+                            store.setCompleted(row.id, it)
+                            items = store.items
+                        },
+                        modifier = Modifier.semantics {
+                            contentDescription = "Completed: ${row.title}"
+                        },
+                    )
+                    Text(if (row.completed) "Completed" else "Incomplete")
+                    TextButton(
+                        onClick = {
+                            editingId = row.id
+                            title = row.title
+                        },
+                        modifier = Modifier.semantics {
+                            contentDescription = "Edit ${row.title}"
+                        },
+                    ) { Text("Edit") }
+                    TextButton(
+                        onClick = {
+                            store.delete(row.id)
+                            items = store.items
+                            if (editingId == row.id) {
+                                editingId = null
+                                title = ""
+                            }
+                        },
+                        modifier = Modifier.semantics {
+                            contentDescription = "Delete ${row.title}"
+                        },
+                    ) { Text("Delete") }
+                }
+            }
+        }
+    }
+}
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt
new file mode 100644
index 0000000..d686423
--- /dev/null
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MemoryItemStore.kt
@@ -0,0 +1,49 @@
+package com.mobilesystemsevolution.kotlin
+
+import java.util.UUID
+
+data class Item(
+    val id: String,
+    val title: String,
+    val completed: Boolean,
+    val version: Long,
+    val updatedAt: Long,
+)
+
+/** M01 owns only process memory. Version is a reserved field, always zero here. */
+class MemoryItemStore(
+    private val nextId: () -> String = { UUID.randomUUID().toString() },
+    private val now: () -> Long = System::currentTimeMillis,
+) {
+    var items: List<Item> = emptyList()
+        private set
+
+    fun create(title: String) {
+        val validTitle = title.trim().also { require(it.isNotEmpty()) }
+        items = items + Item(nextId(), validTitle, false, 0, now())
+    }
+
+    fun rename(id: String, title: String) {
+        val validTitle = title.trim().also { require(it.isNotEmpty()) }
+        items = items.map { item ->
+            if (item.id == id) item.copy(title = validTitle, updatedAt = now()) else item
+        }
+    }
+
+    fun setCompleted(id: String, completed: Boolean) {
+        items = items.map { item ->
+            if (item.id == id) item.copy(completed = completed, updatedAt = now()) else item
+        }
+    }
+
+    fun delete(id: String) {
+        items = items.filterNot { it.id == id }
+    }
+}
+
+/** The same plain mapping is consumed by the screen and checked in host tests. */
+data class ItemRow(val id: String, val title: String, val completed: Boolean) {
+    val tag: String get() = "item-row-$id"
+}
+
+fun List<Item>.toRows(): List<ItemRow> = map { ItemRow(it.id, it.title, it.completed) }
diff --git a/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt b/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt
new file mode 100644
index 0000000..926b803
--- /dev/null
+++ b/app/src/test/java/com/mobilesystemsevolution/kotlin/MemoryItemStoreTest.kt
@@ -0,0 +1,54 @@
+package com.mobilesystemsevolution.kotlin
+
+import org.junit.Assert.assertEquals
+import org.junit.Assert.assertFalse
+import org.junit.Assert.assertTrue
+import org.junit.Test
+
+class MemoryItemStoreTest {
+    @Test
+    fun frozenM01SequencePreservesIdentityAndRenderedRows() {
+        val ids = listOf("item-001", "item-002").iterator()
+        var time = 1700000000000L
+        val store = MemoryItemStore(nextId = { ids.next() }, now = { time.also { time += 1000 } })
+        assertTrue(store.items.isEmpty())
+        assertTrue(store.items.toRows().isEmpty())
+
+        store.create("Alpha")
+        assertEquals(Item("item-001", "Alpha", false, 0, 1700000000000L), store.items.single())
+        store.create("Beta")
+        assertEquals(
+            listOf(ItemRow("item-001", "Alpha", false), ItemRow("item-002", "Beta", false)),
+            store.items.toRows(),
+        )
+
+        store.rename("item-001", "Alpha edited")
+        assertEquals("item-001", store.items.first().id)
+        assertEquals("Alpha edited", store.items.toRows().first().title)
+        assertEquals(1700000002000L, store.items.first().updatedAt)
+
+        store.setCompleted("item-001", true)
+        assertTrue(store.items.toRows().first().completed)
+        store.setCompleted("item-001", false)
+        assertFalse(store.items.toRows().first().completed)
+        store.setCompleted("item-001", true)
+        assertTrue(store.items.toRows().first().completed)
+        store.delete("item-002")
+
+        assertEquals(
+            listOf(Item("item-001", "Alpha edited", true, 0, 1700000005000L)),
+            store.items,
+        )
+        assertEquals(listOf(ItemRow("item-001", "Alpha edited", true)), store.items.toRows())
+        assertEquals("item-row-item-001", store.items.toRows().single().tag)
+        assertFalse(ids.hasNext())
+    }
+
+    @Test
+    fun newStoreHasNoPersistence() {
+        val store = MemoryItemStore()
+        store.create("Memory only")
+        assertEquals(1, store.items.size)
+        assertTrue(MemoryItemStore().items.isEmpty())
+    }
+}
diff --git a/build.gradle.kts b/build.gradle.kts
new file mode 100644
index 0000000..bbd3daa
--- /dev/null
+++ b/build.gradle.kts
@@ -0,0 +1,4 @@
+plugins {
+    id("com.android.application") version "8.6.1" apply false
+    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
+}
diff --git a/gradle.properties b/gradle.properties
new file mode 100644
index 0000000..e172780
--- /dev/null
+++ b/gradle.properties
@@ -0,0 +1,4 @@
+android.useAndroidX=true
+org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
+org.gradle.workers.max=2
+kotlin.code.style=official
diff --git a/gradle/wrapper/gradle-wrapper.jar b/gradle/wrapper/gradle-wrapper.jar
new file mode 100644
index 0000000..a4b76b9
Binary files /dev/null and b/gradle/wrapper/gradle-wrapper.jar differ
diff --git a/gradle/wrapper/gradle-wrapper.properties b/gradle/wrapper/gradle-wrapper.properties
new file mode 100644
index 0000000..381baa9
--- /dev/null
+++ b/gradle/wrapper/gradle-wrapper.properties
@@ -0,0 +1,8 @@
+distributionBase=GRADLE_USER_HOME
+distributionPath=wrapper/dists
+distributionSha256Sum=544c35d6bd849ae8a5ed0bcea39ba677dc40f49df7d1835561582da2009b961d
+distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
+networkTimeout=10000
+validateDistributionUrl=true
+zipStoreBase=GRADLE_USER_HOME
+zipStorePath=wrapper/dists
diff --git a/gradlew b/gradlew
new file mode 100755
index 0000000..f5feea6
--- /dev/null
+++ b/gradlew
@@ -0,0 +1,252 @@
+#!/bin/sh
+
+#
+# Copyright © 2015-2021 the original authors.
+#
+# Licensed under the Apache License, Version 2.0 (the "License");
+# you may not use this file except in compliance with the License.
+# You may obtain a copy of the License at
+#
+#      https://www.apache.org/licenses/LICENSE-2.0
+#
+# Unless required by applicable law or agreed to in writing, software
+# distributed under the License is distributed on an "AS IS" BASIS,
+# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
+# See the License for the specific language governing permissions and
+# limitations under the License.
+#
+# SPDX-License-Identifier: Apache-2.0
+#
+
+##############################################################################
+#
+#   Gradle start up script for POSIX generated by Gradle.
+#
+#   Important for running:
+#
+#   (1) You need a POSIX-compliant shell to run this script. If your /bin/sh is
+#       noncompliant, but you have some other compliant shell such as ksh or
+#       bash, then to run this script, type that shell name before the whole
+#       command line, like:
+#
+#           ksh Gradle
+#
+#       Busybox and similar reduced shells will NOT work, because this script
+#       requires all of these POSIX shell features:
+#         * functions;
+#         * expansions «$var», «${var}», «${var:-default}», «${var+SET}»,
+#           «${var#prefix}», «${var%suffix}», and «$( cmd )»;
+#         * compound commands having a testable exit status, especially «case»;
+#         * various built-in commands including «command», «set», and «ulimit».
+#
+#   Important for patching:
+#
+#   (2) This script targets any POSIX shell, so it avoids extensions provided
+#       by Bash, Ksh, etc; in particular arrays are avoided.
+#
+#       The "traditional" practice of packing multiple parameters into a
+#       space-separated string is a well documented source of bugs and security
+#       problems, so this is (mostly) avoided, by progressively accumulating
+#       options in "$@", and eventually passing that to Java.
+#
+#       Where the inherited environment variables (DEFAULT_JVM_OPTS, JAVA_OPTS,
+#       and GRADLE_OPTS) rely on word-splitting, this is performed explicitly;
+#       see the in-line comments for details.
+#
+#       There are tweaks for specific operating systems such as AIX, CygWin,
+#       Darwin, MinGW, and NonStop.
+#
+#   (3) This script is generated from the Groovy template
+#       https://github.com/gradle/gradle/blob/HEAD/platforms/jvm/plugins-application/src/main/resources/org/gradle/api/internal/plugins/unixStartScript.txt
+#       within the Gradle project.
+#
+#       You can find Gradle at https://github.com/gradle/gradle/.
+#
+##############################################################################
+
+# Attempt to set APP_HOME
+
+# Resolve links: $0 may be a link
+app_path=$0
+
+# Need this for daisy-chained symlinks.
+while
+    APP_HOME=${app_path%"${app_path##*/}"}  # leaves a trailing /; empty if no leading path
+    [ -h "$app_path" ]
+do
+    ls=$( ls -ld "$app_path" )
+    link=${ls#*' -> '}
+    case $link in             #(
+      /*)   app_path=$link ;; #(
+      *)    app_path=$APP_HOME$link ;;
+    esac
+done
+
+# This is normally unused
+# shellcheck disable=SC2034
+APP_BASE_NAME=${0##*/}
+# Discard cd standard output in case $CDPATH is set (https://github.com/gradle/gradle/issues/25036)
+APP_HOME=$( cd -P "${APP_HOME:-./}" > /dev/null && printf '%s
+' "$PWD" ) || exit
+
+# Use the maximum available, or set MAX_FD != -1 to use that value.
+MAX_FD=maximum
+
+warn () {
+    echo "$*"
+} >&2
+
+die () {
+    echo
+    echo "$*"
+    echo
+    exit 1
+} >&2
+
+# OS specific support (must be 'true' or 'false').
+cygwin=false
+msys=false
+darwin=false
+nonstop=false
+case "$( uname )" in                #(
+  CYGWIN* )         cygwin=true  ;; #(
+  Darwin* )         darwin=true  ;; #(
+  MSYS* | MINGW* )  msys=true    ;; #(
+  NONSTOP* )        nonstop=true ;;
+esac
+
+CLASSPATH=$APP_HOME/gradle/wrapper/gradle-wrapper.jar
+
+
+# Determine the Java command to use to start the JVM.
+if [ -n "$JAVA_HOME" ] ; then
+    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
+        # IBM's JDK on AIX uses strange locations for the executables
+        JAVACMD=$JAVA_HOME/jre/sh/java
+    else
+        JAVACMD=$JAVA_HOME/bin/java
+    fi
+    if [ ! -x "$JAVACMD" ] ; then
+        die "ERROR: JAVA_HOME is set to an invalid directory: $JAVA_HOME
+
+Please set the JAVA_HOME variable in your environment to match the
+location of your Java installation."
+    fi
+else
+    JAVACMD=java
+    if ! command -v java >/dev/null 2>&1
+    then
+        die "ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH.
+
+Please set the JAVA_HOME variable in your environment to match the
+location of your Java installation."
+    fi
+fi
+
+# Increase the maximum file descriptors if we can.
+if ! "$cygwin" && ! "$darwin" && ! "$nonstop" ; then
+    case $MAX_FD in #(
+      max*)
+        # In POSIX sh, ulimit -H is undefined. That's why the result is checked to see if it worked.
+        # shellcheck disable=SC2039,SC3045
+        MAX_FD=$( ulimit -H -n ) ||
+            warn "Could not query maximum file descriptor limit"
+    esac
+    case $MAX_FD in  #(
+      '' | soft) :;; #(
+      *)
+        # In POSIX sh, ulimit -n is undefined. That's why the result is checked to see if it worked.
+        # shellcheck disable=SC2039,SC3045
+        ulimit -n "$MAX_FD" ||
+            warn "Could not set maximum file descriptor limit to $MAX_FD"
+    esac
+fi
+
+# Collect all arguments for the java command, stacking in reverse order:
+#   * args from the command line
+#   * the main class name
+#   * -classpath
+#   * -D...appname settings
+#   * --module-path (only if needed)
+#   * DEFAULT_JVM_OPTS, JAVA_OPTS, and GRADLE_OPTS environment variables.
+
+# For Cygwin or MSYS, switch paths to Windows format before running java
+if "$cygwin" || "$msys" ; then
+    APP_HOME=$( cygpath --path --mixed "$APP_HOME" )
+    CLASSPATH=$( cygpath --path --mixed "$CLASSPATH" )
+
+    JAVACMD=$( cygpath --unix "$JAVACMD" )
+
+    # Now convert the arguments - kludge to limit ourselves to /bin/sh
+    for arg do
+        if
+            case $arg in                                #(
+              -*)   false ;;                            # don't mess with options #(
+              /?*)  t=${arg#/} t=/${t%%/*}              # looks like a POSIX filepath
+                    [ -e "$t" ] ;;                      #(
+              *)    false ;;
+            esac
+        then
+            arg=$( cygpath --path --ignore --mixed "$arg" )
+        fi
+        # Roll the args list around exactly as many times as the number of
+        # args, so each arg winds up back in the position where it started, but
+        # possibly modified.
+        #
+        # NB: a `for` loop captures its iteration list before it begins, so
+        # changing the positional parameters here affects neither the number of
+        # iterations, nor the values presented in `arg`.
+        shift                   # remove old arg
+        set -- "$@" "$arg"      # push replacement arg
+    done
+fi
+
+
+# Add default JVM options here. You can also use JAVA_OPTS and GRADLE_OPTS to pass JVM options to this script.
+DEFAULT_JVM_OPTS='"-Xmx64m" "-Xms64m"'
+
+# Collect all arguments for the java command:
+#   * DEFAULT_JVM_OPTS, JAVA_OPTS, JAVA_OPTS, and optsEnvironmentVar are not allowed to contain shell fragments,
+#     and any embedded shellness will be escaped.
+#   * For example: A user cannot expect ${Hostname} to be expanded, as it is an environment variable and will be
+#     treated as '${Hostname}' itself on the command line.
+
+set -- \
+        "-Dorg.gradle.appname=$APP_BASE_NAME" \
+        -classpath "$CLASSPATH" \
+        org.gradle.wrapper.GradleWrapperMain \
+        "$@"
+
+# Stop when "xargs" is not available.
+if ! command -v xargs >/dev/null 2>&1
+then
+    die "xargs is not available"
+fi
+
+# Use "xargs" to parse quoted args.
+#
+# With -n1 it outputs one arg per line, with the quotes and backslashes removed.
+#
+# In Bash we could simply go:
+#
+#   readarray ARGS < <( xargs -n1 <<<"$var" ) &&
+#   set -- "${ARGS[@]}" "$@"
+#
+# but POSIX shell has neither arrays nor command substitution, so instead we
+# post-process each arg (as a line of input to sed) to backslash-escape any
+# character that might be a shell metacharacter, then use eval to reverse
+# that process (while maintaining the separation between arguments), and wrap
+# the whole thing up as a single "set" statement.
+#
+# This will of course break if any of these variables contains a newline or
+# an unmatched quote.
+#
+
+eval "set -- $(
+        printf '%s\n' "$DEFAULT_JVM_OPTS $JAVA_OPTS $GRADLE_OPTS" |
+        xargs -n1 |
+        sed ' s~[^-[:alnum:]+,./:=@_]~\\&~g; ' |
+        tr '\n' ' '
+    )" '"$@"'
+
+exec "$JAVACMD" "$@"
diff --git a/gradlew.bat b/gradlew.bat
new file mode 100644
index 0000000..9d21a21
--- /dev/null
+++ b/gradlew.bat
@@ -0,0 +1,94 @@
+@rem
+@rem Copyright 2015 the original author or authors.
+@rem
+@rem Licensed under the Apache License, Version 2.0 (the "License");
+@rem you may not use this file except in compliance with the License.
+@rem You may obtain a copy of the License at
+@rem
+@rem      https://www.apache.org/licenses/LICENSE-2.0
+@rem
+@rem Unless required by applicable law or agreed to in writing, software
+@rem distributed under the License is distributed on an "AS IS" BASIS,
+@rem WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
+@rem See the License for the specific language governing permissions and
+@rem limitations under the License.
+@rem
+@rem SPDX-License-Identifier: Apache-2.0
+@rem
+
+@if "%DEBUG%"=="" @echo off
+@rem ##########################################################################
+@rem
+@rem  Gradle startup script for Windows
+@rem
+@rem ##########################################################################
+
+@rem Set local scope for the variables with windows NT shell
+if "%OS%"=="Windows_NT" setlocal
+
+set DIRNAME=%~dp0
+if "%DIRNAME%"=="" set DIRNAME=.
+@rem This is normally unused
+set APP_BASE_NAME=%~n0
+set APP_HOME=%DIRNAME%
+
+@rem Resolve any "." and ".." in APP_HOME to make it shorter.
+for %%i in ("%APP_HOME%") do set APP_HOME=%%~fi
+
+@rem Add default JVM options here. You can also use JAVA_OPTS and GRADLE_OPTS to pass JVM options to this script.
+set DEFAULT_JVM_OPTS="-Xmx64m" "-Xms64m"
+
+@rem Find java.exe
+if defined JAVA_HOME goto findJavaFromJavaHome
+
+set JAVA_EXE=java.exe
+%JAVA_EXE% -version >NUL 2>&1
+if %ERRORLEVEL% equ 0 goto execute
+
+echo. 1>&2
+echo ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH. 1>&2
+echo. 1>&2
+echo Please set the JAVA_HOME variable in your environment to match the 1>&2
+echo location of your Java installation. 1>&2
+
+goto fail
+
+:findJavaFromJavaHome
+set JAVA_HOME=%JAVA_HOME:"=%
+set JAVA_EXE=%JAVA_HOME%/bin/java.exe
+
+if exist "%JAVA_EXE%" goto execute
+
+echo. 1>&2
+echo ERROR: JAVA_HOME is set to an invalid directory: %JAVA_HOME% 1>&2
+echo. 1>&2
+echo Please set the JAVA_HOME variable in your environment to match the 1>&2
+echo location of your Java installation. 1>&2
+
+goto fail
+
+:execute
+@rem Setup the command line
+
+set CLASSPATH=%APP_HOME%\gradle\wrapper\gradle-wrapper.jar
+
+
+@rem Execute Gradle
+"%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %*
+
+:end
+@rem End local scope for the variables with windows NT shell
+if %ERRORLEVEL% equ 0 goto mainEnd
+
+:fail
+rem Set variable GRADLE_EXIT_CONSOLE if you need the _script_ return code instead of
+rem the _cmd.exe /c_ return code!
+set EXIT_CODE=%ERRORLEVEL%
+if %EXIT_CODE% equ 0 set EXIT_CODE=1
+if not ""=="%GRADLE_EXIT_CONSOLE%" exit %EXIT_CODE%
+exit /b %EXIT_CODE%
+
+:mainEnd
+if "%OS%"=="Windows_NT" endlocal
+
+:omega
diff --git a/settings.gradle.kts b/settings.gradle.kts
new file mode 100644
index 0000000..3d6b916
--- /dev/null
+++ b/settings.gradle.kts
@@ -0,0 +1,18 @@
+pluginManagement {
+    repositories {
+        google()
+        mavenCentral()
+        gradlePluginPortal()
+    }
+}
+
+dependencyResolutionManagement {
+    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
+    repositories {
+        google()
+        mavenCentral()
+    }
+}
+
+rootProject.name = "OfflineItemTracker"
+include(":app")
diff --git a/verification/M01.md b/verification/M01.md
new file mode 100644
index 0000000..bffaa39
--- /dev/null
+++ b/verification/M01.md
@@ -0,0 +1,68 @@
+# M01 verification — attempt 1
+
+- Thread: M01; branch: `track/android-kotlin`; start: UNBORN
+- Spec revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`
+- Frozen shared scenario SHA-256: `435ace55ed211807526f9d11736820a88cce601f4ba158db74f108690ca0febc`
+- Flow: Establish → Verify → Commit → Report
+- Device: the shared API 34 / Pixel 6 / ARM64 Google APIs emulator, serial `emulator-5554`
+- Runtime: JDK 17; Gradle 8.7; SDK platform 35 and Build Tools 34.0.0
+- `JAVA_HOME`, `ANDROID_HOME`, and external `GRADLE_USER_HOME` are supplied by the caller.
+
+## Fixed input and result
+
+Both scenario tests start with zero Items, create Alpha, create Beta, rename Alpha to
+Alpha edited, complete it, mark it incomplete, complete it again, and delete Beta.
+The required final result is exactly one Item with the first identity, title Alpha edited,
+completed true; Beta is absent. Host IDs are item-001 and item-002; the clock begins at
+1700000000000 ms and advances 1000 ms for every subsequent edit. No fixture or acceptance
+was modified after observing any result.
+
+The host test checks stable identity and the exact `toRows()` list mapping used by the UI,
+including both completion states. The instrumented test uses only actual Compose controls,
+captures the first row's identity tag, and checks the final count, title, tag, checkbox,
+and absence of Beta.
+
+## Invocation ledger
+
+All commands below run from the branch root. This is not a performance/battery scenario;
+no benchmark, soak, device matrix, or polling-period exploration was performed.
+
+| Invocation | Command | Exit | Tests executed | Result |
+| --- | --- | --- | --- | --- |
+| Host/build 1 | `./gradlew --no-daemon --console=plain :app:testDebugUnitTest :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | 2 host; 0 device | 2 passed, 0 failed/errors/skipped; both APKs built (3m44s) |
+| Android UI 1 | `ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon --console=plain :app:connectedDebugAndroidTest` | 0 | 1 device; 0 host | 1 passed, 0 failed/errors/skipped; APK installation and actual fixed UI sequence succeeded (1m03s) |
+
+The second host test confirms a fresh memory store starts empty; it is not claimed as an
+Android restart test. The device lease was explicitly granted before the Android invocation
+and returned immediately after it completed. No device profile/network settings were changed.
+There were no failed test invocations and no repair attempts in this implementation task.
+
+Host report: `app/build/test-results/testDebugUnitTest/TEST-com.mobilesystemsevolution.kotlin.MemoryItemStoreTest.xml`.
+Android report: `app/build/outputs/androidTest-results/connected/debug/TEST-MSE_API34_Pixel6(AVD) - 14-_app-.xml`.
+Raw console output, XML copies, exact command/environment ledger, and instrumentation log are
+retained in the run's external evidence directory as `host-build-01.log`, `host-results-01.xml`,
+`android-ui-01.log`, `android-ui-results-01.xml`, `commands.txt`, and
+`android-instrumentation-01.log`; build outputs/caches are ignored.
+
+The Android test timestamp is 2026-08-27T23:23:59 UTC; the report records one successful
+`frozenM01SequenceThroughActualAndroidUi` test (24.097 seconds). This is actual platform
+verification, not a compiled-only test or a host substitute.
+
+## Setup audit (not acceptance execution)
+
+- Installed `gradle --version` initially exited 1 because the sandbox denied native-cache
+  initialization; the explicitly escalated rerun exited 0 (Gradle 8.11.1). No tests ran.
+- Generated the standard wrapper in an external empty bootstrap directory using Gradle
+  8.11.1: `--no-daemon wrapper --gradle-version 8.7 --distribution-type bin
+  --gradle-distribution-sha256-sum 544c35d6bd849ae8a5ed0bcea39ba677dc40f49df7d1835561582da2009b961d`.
+  Exit 0. The distribution hash was fetched from Gradle's official checksum endpoint.
+- Dependency downloads and Gradle execution used explicit sandbox escalation; no failed
+  setup command was counted as an acceptance pass.
+
+## Scope and history
+
+The app has one editing/list screen, no persistence/network permission/sync/background
+implementation, and no custom theme or state framework. Required Item fields exist;
+version remains zero without synchronization semantics. All M01 changes are committed
+only on the independent implementation branch, with the required Thread and Spec-Revision
+trailers. Progress tagging and independent verification belong to the main orchestrator.
