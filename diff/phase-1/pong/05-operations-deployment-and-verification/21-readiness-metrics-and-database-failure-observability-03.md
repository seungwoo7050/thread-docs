## `test(db): 안전한 connection pool 오류 처리 검증`

diff --git a/packages/db/src/poolError.test.ts b/packages/db/src/poolError.test.ts
new file mode 100644
index 0000000..cbc1b16
--- /dev/null
+++ b/packages/db/src/poolError.test.ts
@@ -0,0 +1,50 @@
+import { Pool } from "pg";
+import { describe, expect, it, vi } from "vitest";
+import { installPostgresPoolErrorHandler } from "./poolError";
+
+describe("PostgreSQL pool error handling", () => {
+  it("observes idle client errors without exposing connection details", async () => {
+    const pool = new Pool();
+    const onPoolError = vi.fn();
+    installPostgresPoolErrorHandler(pool, onPoolError);
+    const error = Object.assign(
+      new Error("Connection terminated unexpectedly at postgresql://user:secret@database:5432/app"),
+      {
+        code: "57P01",
+        connectionString: "postgresql://user:secret@database:5432/app"
+      }
+    );
+
+    expect(pool.listenerCount("error")).toBe(1);
+    expect(() => pool.emit("error", error)).not.toThrow();
+    expect(onPoolError).toHaveBeenCalledWith({
+      kind: "idle_client_error",
+      errorName: "Error",
+      errorCode: "57P01"
+    });
+    expect(JSON.stringify(onPoolError.mock.calls)).not.toContain("secret");
+    expect(JSON.stringify(onPoolError.mock.calls)).not.toContain("Connection terminated");
+
+    await pool.end();
+  });
+
+  it("keeps the pool error boundary safe when no reporter is configured", async () => {
+    const pool = new Pool();
+    installPostgresPoolErrorHandler(pool);
+
+    expect(() => pool.emit("error", new Error("Connection terminated unexpectedly"))).not.toThrow();
+
+    await pool.end();
+  });
+
+  it("does not let a reporter failure become an uncaught pool error", async () => {
+    const pool = new Pool();
+    installPostgresPoolErrorHandler(pool, () => {
+      throw new Error("reporter failed");
+    });
+
+    expect(() => pool.emit("error", new Error("Connection terminated unexpectedly"))).not.toThrow();
+
+    await pool.end();
+  });
+});
