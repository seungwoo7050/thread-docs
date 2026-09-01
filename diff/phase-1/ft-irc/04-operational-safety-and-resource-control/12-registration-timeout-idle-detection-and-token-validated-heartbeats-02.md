## `test(heartbeat): PONG 토큰과 시간 경계 검증`

diff --git a/tests/irc_contract.py b/tests/irc_contract.py
index 0343d04..805cd2c 100644
--- a/tests/irc_contract.py
+++ b/tests/irc_contract.py
@@ -453,22 +453,55 @@ def check_wire_contract(
         time.sleep(0.2)
 
         heartbeat = register_contract_peer(manifest, host, port, password, "ctheartbeat")
+        heartbeat.auto_pong = False
         peers.append(heartbeat)
-        record_regex(
+        heartbeat_ping = record_regex(
             manifest,
             heartbeat,
             "heartbeat_ping",
             rf":{re.escape(SERVER_NAME)} PING heartbeat-\d+-\d+",
             timeout=5.0,
         )
+        heartbeat_token = heartbeat_ping.rsplit(" ", 1)[1].lstrip(":")
+        heartbeat.send_line(f"PONG :{heartbeat_token}")
+        time.sleep(1.1)
+        heartbeat.send_line("METRICS")
+        record_regex(
+            manifest,
+            heartbeat,
+            "matching_pong_keeps_connection",
+            metrics_pattern("ctheartbeat"),
+        )
+        time.sleep(1.1)
         heartbeat.send_line("METRICS")
         record_regex(
             manifest,
             heartbeat,
-            "heartbeat_pong_then_metrics",
+            "matching_pong_clears_deadline",
             metrics_pattern("ctheartbeat"),
         )
         heartbeat.close()
+
+        forged = register_contract_peer(manifest, host, port, password, "ctforgedpong")
+        forged.auto_pong = False
+        peers.append(forged)
+        record_regex(
+            manifest,
+            forged,
+            "forged_pong_ping",
+            rf":{re.escape(SERVER_NAME)} PING heartbeat-\d+-\d+",
+            timeout=5.0,
+        )
+        forged.send_line("PONG :forged-heartbeat-token")
+        record_exact(
+            manifest,
+            forged,
+            "forged_pong_timeout",
+            "ERROR :Ping timeout",
+            timeout=4.0,
+        )
+        forged.wait_closed(2.0)
+        forged.close()
         time.sleep(0.2)
 
         shutdown_peer = register_contract_peer(
