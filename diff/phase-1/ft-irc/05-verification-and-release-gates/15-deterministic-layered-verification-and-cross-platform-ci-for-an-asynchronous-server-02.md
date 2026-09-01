## `test(irc): 실행 조건과 오류 동작 계약 검증`

diff --git a/tests/irc_contract.py b/tests/irc_contract.py
new file mode 100644
index 0000000..48f946a
--- /dev/null
+++ b/tests/irc_contract.py
@@ -0,0 +1,547 @@
+#!/usr/bin/env python3
+"""Characterize the public CLI, IRC wire, and shutdown contracts."""
+
+import json
+import os
+from pathlib import Path
+import re
+import signal
+import subprocess
+import sys
+import time
+from typing import Dict, List, Optional
+
+
+ROOT = Path(__file__).resolve().parents[1]
+sys.path.insert(0, str(ROOT / "tools"))
+
+from irc_smoke_client import IrcPeer, metrics_pattern, register  # noqa: E402
+
+
+SERVER_NAME = "irc.relay.local"
+
+
+def fail(message: str) -> None:
+    raise AssertionError(message)
+
+
+def check_cli(
+    manifest: Dict[str, object],
+    binary: str,
+    label: str,
+    arguments: List[str],
+    expected_stderr: Optional[str] = None,
+    stderr_prefix: Optional[str] = None,
+) -> None:
+    completed = subprocess.run(
+        [binary] + arguments,
+        check=False,
+        stdout=subprocess.PIPE,
+        stderr=subprocess.PIPE,
+        text=True,
+    )
+    if completed.returncode != 1:
+        fail(f"CLI {label}: expected exit 1, got {completed.returncode}")
+    if completed.stdout != "":
+        fail(f"CLI {label}: expected empty stdout, got {completed.stdout!r}")
+    if expected_stderr is not None and completed.stderr != expected_stderr:
+        fail(
+            f"CLI {label}: stderr mismatch\n"
+            f"expected: {expected_stderr!r}\nactual:   {completed.stderr!r}"
+        )
+    if stderr_prefix is not None:
+        allowed_stderr = (
+            stderr_prefix + "\n",
+            stderr_prefix + ": Invalid argument\n",
+        )
+        if completed.stderr not in allowed_stderr:
+            fail(
+                f"CLI {label}: stderr was outside the same-platform contract; "
+                f"expected one of {allowed_stderr!r}, got {completed.stderr!r}"
+            )
+
+    cli_manifest = manifest["cli"]
+    if not isinstance(cli_manifest, dict):
+        fail("internal manifest error: cli entry is not a mapping")
+    cli_manifest[label] = {
+        "arguments": arguments,
+        "exit": completed.returncode,
+        "stdout": completed.stdout,
+        "stderr": completed.stderr,
+    }
+
+
+def check_cli_contract(manifest: Dict[str, object], binary: str) -> None:
+    usage = (
+        f"Usage: {binary} <port> <password> "
+        "[--idle-timeout=N] [--ping-timeout=N] [--registration-timeout=N] "
+        "[--rate-limit=COUNT:SECONDS] [--max-pending-bytes=N] "
+        "[--max-connections=N]\n"
+    )
+    check_cli(manifest, binary, "usage", [], expected_stderr=usage)
+    check_cli(
+        manifest,
+        binary,
+        "invalid_port",
+        ["0", "contract-secret"],
+        expected_stderr="irc-relay-server: port must be an integer from 1 to 65535\n",
+    )
+    check_cli(
+        manifest,
+        binary,
+        "zero_timeout",
+        ["6667", "contract-secret", "--idle-timeout=0"],
+        expected_stderr="irc-relay-server: idle timeout must be a positive integer\n",
+    )
+    check_cli(
+        manifest,
+        binary,
+        "rate_limit_shape",
+        ["6667", "contract-secret", "--rate-limit=24"],
+        expected_stderr="irc-relay-server: rate limit must use COUNT:SECONDS\n",
+    )
+    check_cli(
+        manifest,
+        binary,
+        "unknown_option",
+        ["6667", "contract-secret", "--unknown=1"],
+        expected_stderr="irc-relay-server: unknown option: --unknown=1\n",
+    )
+    check_cli(
+        manifest,
+        binary,
+        "platform_errno_suffix",
+        ["6667", "contract-secret", "--max-pending-bytes=abc"],
+        stderr_prefix="irc-relay-server: max pending bytes must be an unsigned integer",
+    )
+
+
+def record_exact(
+    manifest: Dict[str, object],
+    peer: IrcPeer,
+    label: str,
+    expected: str,
+    timeout: float = 2.0,
+    next_frame: bool = False,
+) -> str:
+    if next_frame:
+        actual = peer.expect_next_exact(expected, timeout)
+    else:
+        actual = peer.expect_exact(expected, timeout)
+    checks = manifest["wire_checks"]
+    if not isinstance(checks, list):
+        fail("internal manifest error: wire_checks entry is not a list")
+    normalized = re.sub(r"(@(?:\[[^]]+\]|[^ :]+):)\d+", r"\1<port>", expected)
+    checks.append({"label": label, "expected": normalized})
+    return actual
+
+
+def record_regex(
+    manifest: Dict[str, object],
+    peer: IrcPeer,
+    label: str,
+    pattern: str,
+    timeout: float = 2.0,
+) -> str:
+    actual = peer.expect_regex(pattern, timeout)
+    checks = manifest["wire_checks"]
+    if not isinstance(checks, list):
+        fail("internal manifest error: wire_checks entry is not a list")
+    checks.append({"label": label, "pattern": pattern})
+    return actual
+
+
+def register_contract_peer(
+    manifest: Dict[str, object],
+    host: str,
+    port: int,
+    password: str,
+    nick: str,
+) -> IrcPeer:
+    peer = register(host, port, password, nick, f"{nick} Contract")
+    checks = manifest["wire_checks"]
+    if not isinstance(checks, list):
+        fail("internal manifest error: wire_checks entry is not a list")
+    checks.extend(
+        [
+            {"label": f"{nick}_001", "expected": f":{SERVER_NAME} 001 {nick} :Welcome to irc-relay-server, {nick}"},
+            {"label": f"{nick}_002", "expected": f":{SERVER_NAME} 002 {nick} :Your host is {SERVER_NAME}"},
+            {"label": f"{nick}_003", "expected": f":{SERVER_NAME} 003 {nick} :This server is running a C++17 event backend"},
+        ]
+    )
+    return peer
+
+
+def close_peers(peers: List[IrcPeer]) -> None:
+    for peer in peers:
+        peer.close()
+
+
+def check_wire_contract(
+    manifest: Dict[str, object], host: str, port: int, password: str
+) -> IrcPeer:
+    peers: List[IrcPeer] = []
+    try:
+        pre_registration = IrcPeer(host, port, "contract-pre-registration")
+        peers.append(pre_registration)
+        pre_registration.send_line("PRIVMSG nobody :blocked")
+        record_exact(
+            manifest,
+            pre_registration,
+            "pre_registration_451",
+            f":{SERVER_NAME} 451 * :You have not registered",
+            next_frame=True,
+        )
+        pre_registration.send_line("NICK 1contract")
+        record_exact(
+            manifest,
+            pre_registration,
+            "invalid_nickname_432",
+            f":{SERVER_NAME} 432 * 1contract :Erroneous nickname",
+            next_frame=True,
+        )
+        pre_registration.close()
+
+        wrong_password = IrcPeer(host, port, "contract-wrong-password")
+        peers.append(wrong_password)
+        wrong_password.send_line("PASS wrong-password")
+        record_exact(
+            manifest,
+            wrong_password,
+            "wrong_password_464",
+            f":{SERVER_NAME} 464 * :Password incorrect",
+            next_frame=True,
+        )
+        wrong_password.close()
+
+        taken = register_contract_peer(manifest, host, port, password, "cttaken")
+        peers.append(taken)
+        collision = IrcPeer(host, port, "contract-collision")
+        peers.append(collision)
+        collision.send_line(f"PASS {password}")
+        collision.send_line("NICK CTTAKEN")
+        record_exact(
+            manifest,
+            collision,
+            "nickname_collision_433",
+            f":{SERVER_NAME} 433 * CTTAKEN :Nickname is already in use",
+            next_frame=True,
+        )
+        collision.close()
+        taken.close()
+
+        alpha = register_contract_peer(manifest, host, port, password, "ctalpha")
+        beta = register_contract_peer(manifest, host, port, password, "ctbeta")
+        gamma = register_contract_peer(manifest, host, port, password, "ctgamma")
+        peers.extend([alpha, beta, gamma])
+        alpha_prefix = alpha.hostmask("ctalpha", "ctalpha")
+        beta_prefix = beta.hostmask("ctbeta", "ctbeta")
+        gamma_prefix = gamma.hostmask("ctgamma", "ctgamma")
+
+        alpha.send_raw(b"PI")
+        time.sleep(0.05)
+        alpha.send_raw(b"NG :contract-token\r\n")
+        record_exact(
+            manifest,
+            alpha,
+            "split_ping_pong",
+            f":{SERVER_NAME} PONG {SERVER_NAME} contract-token",
+            next_frame=True,
+        )
+
+        alpha.send_line("JOIN #contract")
+        record_exact(manifest, alpha, "join", f":{alpha_prefix} JOIN #contract", next_frame=True)
+        record_exact(
+            manifest,
+            alpha,
+            "join_no_topic_331",
+            f":{SERVER_NAME} 331 ctalpha #contract :No topic is set",
+            next_frame=True,
+        )
+        record_exact(
+            manifest,
+            alpha,
+            "join_names_353",
+            f":{SERVER_NAME} 353 ctalpha = #contract @ctalpha",
+            next_frame=True,
+        )
+        record_exact(
+            manifest,
+            alpha,
+            "join_names_end_366",
+            f":{SERVER_NAME} 366 ctalpha #contract :End of /NAMES list",
+            next_frame=True,
+        )
+
+        alpha.send_line("LIST #contract")
+        record_exact(
+            manifest,
+            alpha,
+            "list_start_321",
+            f":{SERVER_NAME} 321 ctalpha Channel Users Name",
+            next_frame=True,
+        )
+        record_exact(
+            manifest,
+            alpha,
+            "list_item_322",
+            f":{SERVER_NAME} 322 ctalpha #contract 1 :open room",
+            next_frame=True,
+        )
+        record_exact(
+            manifest,
+            alpha,
+            "list_end_323",
+            f":{SERVER_NAME} 323 ctalpha :End of /LIST",
+            next_frame=True,
+        )
+        alpha.send_line("NAMES #contract")
+        record_exact(
+            manifest,
+            alpha,
+            "names_353",
+            f":{SERVER_NAME} 353 ctalpha = #contract @ctalpha",
+            next_frame=True,
+        )
+        record_exact(
+            manifest,
+            alpha,
+            "names_end_366",
+            f":{SERVER_NAME} 366 ctalpha #contract :End of /NAMES list",
+            next_frame=True,
+        )
+
+        beta.send_line("JOIN #contract")
+        record_exact(manifest, beta, "beta_join", f":{beta_prefix} JOIN #contract")
+        record_exact(manifest, alpha, "beta_join_broadcast", f":{beta_prefix} JOIN #contract")
+
+        alpha.send_line("TOPIC #contract :Contract topic")
+        record_exact(
+            manifest,
+            alpha,
+            "topic_broadcast",
+            f":{alpha_prefix} TOPIC #contract :Contract topic",
+        )
+        alpha.send_line("PRIVMSG #contract :channel contract")
+        record_exact(
+            manifest,
+            beta,
+            "channel_privmsg",
+            f":{alpha_prefix} PRIVMSG #contract :channel contract",
+        )
+
+        alpha.send_line("INVITE ctgamma #contract")
+        record_exact(
+            manifest,
+            alpha,
+            "invite_numeric_341",
+            f":{SERVER_NAME} 341 ctalpha ctgamma #contract",
+        )
+        record_exact(
+            manifest,
+            gamma,
+            "invite_delivery",
+            f":{alpha_prefix} INVITE ctgamma #contract",
+        )
+        gamma.send_line("JOIN #contract")
+        record_exact(manifest, gamma, "gamma_join", f":{gamma_prefix} JOIN #contract")
+
+        alpha.send_line("MODE #contract +i")
+        record_exact(manifest, alpha, "mode_invite_only", f":{alpha_prefix} MODE #contract +i")
+        alpha.send_line("MODE #contract +o ctbeta")
+        record_exact(
+            manifest,
+            beta,
+            "mode_operator",
+            f":{alpha_prefix} MODE #contract +o ctbeta",
+        )
+        beta.send_line("KICK #contract ctgamma :contract complete")
+        record_exact(
+            manifest,
+            gamma,
+            "kick",
+            f":{beta_prefix} KICK #contract ctgamma :contract complete",
+        )
+
+        alpha.send_line("MODE #contract")
+        record_exact(
+            manifest,
+            alpha,
+            "mode_query_324",
+            f":{SERVER_NAME} 324 ctalpha #contract +it",
+        )
+        alpha.send_line("PRIVMSG ctbeta :direct contract")
+        record_exact(
+            manifest,
+            beta,
+            "direct_privmsg",
+            f":{alpha_prefix} PRIVMSG ctbeta :direct contract",
+        )
+        alpha.send_line("METRICS")
+        record_regex(manifest, alpha, "metrics_key_order", metrics_pattern("ctalpha"))
+
+        flood = register_contract_peer(manifest, host, port, password, "ctflood")
+        peers.append(flood)
+        for index in range(25):
+            flood.send_line(f"PING :contract-burst-{index}")
+        record_exact(
+            manifest,
+            flood,
+            "rate_limit_439",
+            f":{SERVER_NAME} 439 ctflood :Command rate limit exceeded",
+        )
+        flood.close()
+
+        alpha.close()
+        beta.close()
+        gamma.close()
+        time.sleep(0.2)
+
+        heartbeat = register_contract_peer(manifest, host, port, password, "ctheartbeat")
+        peers.append(heartbeat)
+        record_regex(
+            manifest,
+            heartbeat,
+            "heartbeat_ping",
+            rf":{re.escape(SERVER_NAME)} PING heartbeat-\d+-\d+",
+            timeout=5.0,
+        )
+        heartbeat.send_line("METRICS")
+        record_regex(
+            manifest,
+            heartbeat,
+            "heartbeat_pong_then_metrics",
+            metrics_pattern("ctheartbeat"),
+        )
+        heartbeat.close()
+        time.sleep(0.2)
+
+        shutdown_peer = register_contract_peer(
+            manifest, host, port, password, "ctshutdown"
+        )
+        peers.append(shutdown_peer)
+        return shutdown_peer
+    except Exception:
+        close_peers(peers)
+        raise
+
+
+def validate_shutdown_log(
+    log_path: Path, port: int, timeout: float = 3.0
+) -> List[str]:
+    deadline = time.time() + timeout
+    last_text = ""
+    while time.time() < deadline:
+        try:
+            last_text = log_path.read_text(errors="replace")
+        except OSError:
+            time.sleep(0.02)
+            continue
+        lines = last_text.splitlines()
+
+        try:
+            started_index = lines.index(f"event=server_started port={port}")
+            registered_index = next(
+                index
+                for index, line in enumerate(lines)
+                if re.fullmatch(r"event=client_registered fd=\d+ nick=ctshutdown", line)
+            )
+            registered = lines[registered_index]
+            fd_match = re.search(r"fd=(\d+)", registered)
+            if fd_match is None:
+                raise ValueError("shutdown registration log has no fd")
+            fd = fd_match.group(1)
+            connected_index = max(
+                index
+                for index, line in enumerate(lines[:registered_index])
+                if line.startswith(f"event=client_connected fd={fd} peer=")
+            )
+            metrics_index = next(
+                index
+                for index, line in enumerate(lines[registered_index + 1 :], registered_index + 1)
+                if re.fullmatch(
+                    r"event=server_metrics accepted=\d+ closed=\d+ lines=\d+ "
+                    r"queue_drops=\d+ commands=\d+ messages=\d+ rooms=\d+ "
+                    r"rooms_created=\d+ rate_limited=\d+ idle_timeouts=\d+ "
+                    r"heartbeats=\d+",
+                    line,
+                )
+            )
+            disconnected_index = next(
+                index
+                for index, line in enumerate(lines[metrics_index + 1 :], metrics_index + 1)
+                if line == f"event=client_disconnected fd={fd} reason=Server_shutting_down"
+            )
+        except (StopIteration, ValueError):
+            time.sleep(0.02)
+            continue
+
+        indices = [
+            started_index,
+            connected_index,
+            registered_index,
+            metrics_index,
+            disconnected_index,
+        ]
+        if indices != sorted(indices) or len(set(indices)) != len(indices):
+            fail(f"shutdown log events are out of order: {indices}")
+        return [
+            "server_started",
+            "client_connected",
+            "client_registered",
+            "server_metrics",
+            "client_disconnected:Server_shutting_down",
+        ]
+
+    tail = "\n".join(last_text.splitlines()[-30:])
+    fail(f"shutdown log contract was not observed within {timeout}s; log tail:\n{tail}")
+    return []
+
+
+def write_manifest(manifest: Dict[str, object]) -> None:
+    target = os.environ.get("IRC_CONTRACT_MANIFEST")
+    if not target:
+        return
+    Path(target).write_text(json.dumps(manifest, indent=2, sort_keys=True) + "\n")
+
+
+def main() -> int:
+    if len(sys.argv) != 7:
+        print(
+            f"usage: {sys.argv[0]} <binary> <host> <port> <password> <server-pid> <log-file>",
+            file=sys.stderr,
+        )
+        return 2
+
+    binary = sys.argv[1]
+    host = sys.argv[2]
+    port = int(sys.argv[3])
+    password = sys.argv[4]
+    server_pid = int(sys.argv[5])
+    log_path = Path(sys.argv[6])
+    manifest: Dict[str, object] = {
+        "cli": {},
+        "wire_checks": [],
+        "log_order": [],
+    }
+
+    check_cli_contract(manifest, binary)
+    shutdown_peer = check_wire_contract(manifest, host, port, password)
+
+    os.kill(server_pid, signal.SIGTERM)
+    record_exact(
+        manifest,
+        shutdown_peer,
+        "shutdown_error",
+        "ERROR :Server shutting down",
+        timeout=3.0,
+    )
+    shutdown_peer.wait_closed(3.0)
+    manifest["log_order"] = validate_shutdown_log(log_path, port)
+    shutdown_peer.close()
+    write_manifest(manifest)
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/tests/irc_smoke.sh b/tests/irc_smoke.sh
index 1d7d470..7a25e5d 100755
--- a/tests/irc_smoke.sh
+++ b/tests/irc_smoke.sh
@@ -1,5 +1,6 @@
 #!/usr/bin/env bash
 set -euo pipefail
+export PYTHONDONTWRITEBYTECODE=1
 
 ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
 PORT="${IRC_SMOKE_PORT:-$(python3 - <<'PY'
@@ -50,4 +51,9 @@ raise SystemExit("server did not accept connections")
 PY
 
 python3 "${ROOT}/tools/irc_smoke_client.py" 127.0.0.1 "${PORT}" "${PASSWORD}"
+python3 "${ROOT}/tests/irc_contract.py" \
+	"${ROOT}/irc-relay-server" 127.0.0.1 "${PORT}" "${PASSWORD}" \
+	"${SERVER_PID}" "${LOG_FILE}"
+wait "${SERVER_PID}"
+SERVER_PID=""
 echo "IRC smoke passed on port ${PORT}"
diff --git a/tools/irc_smoke_client.py b/tools/irc_smoke_client.py
index d15b34d..9c1b3bf 100755
--- a/tools/irc_smoke_client.py
+++ b/tools/irc_smoke_client.py
@@ -1,10 +1,11 @@
 #!/usr/bin/env python3
 """Small IRC smoke client for irc-relay-server."""
 
+import re
 import socket
 import sys
 import time
-from typing import List
+from typing import List, Tuple
 
 
 class IrcPeer:
@@ -15,6 +16,8 @@ class IrcPeer:
         self.sock.settimeout(0.2)
         self.buffer = b""
         self.lines: List[str] = []
+        self.endings: List[bytes] = []
+        self.closed = False
 
     def send_line(self, line: str) -> None:
         self.sock.sendall(line.encode("utf-8") + b"\r\n")
@@ -30,14 +33,23 @@ class IrcPeer:
             except socket.timeout:
                 break
             except OSError:
+                self.closed = True
                 break
             if not chunk:
+                self.closed = True
                 break
             self.buffer += chunk
             while b"\n" in self.buffer:
                 raw, self.buffer = self.buffer.split(b"\n", 1)
-                line = raw.rstrip(b"\r").decode("utf-8", "replace")
+                if raw.endswith(b"\r"):
+                    line_bytes = raw[:-1]
+                    ending = b"\r\n"
+                else:
+                    line_bytes = raw
+                    ending = b"\n"
+                line = line_bytes.decode("utf-8", "replace")
                 self.lines.append(line)
+                self.endings.append(ending)
                 self._auto_reply_to_ping(line)
         return list(self.lines)
 
@@ -46,16 +58,86 @@ class IrcPeer:
         while time.time() < deadline:
             for index, line in enumerate(self.lines):
                 if needle in line:
-                    return self.lines.pop(index)
+                    return self._pop_line(index)[0]
             self.read_available(0.1)
         transcript = "\n".join(self.lines[-20:])
         raise AssertionError(f"{self.label}: expected {needle!r}; recent lines:\n{transcript}")
 
+    def expect_exact(self, expected: str, timeout: float = 2.0) -> str:
+        deadline = time.time() + timeout
+        while time.time() < deadline:
+            for index, line in enumerate(self.lines):
+                if line == expected:
+                    matched, ending = self._pop_line(index)
+                    self._require_crlf(matched, ending)
+                    return matched
+            self.read_available(0.1)
+        transcript = "\n".join(self.lines[-20:])
+        raise AssertionError(
+            f"{self.label}: expected exact frame {expected!r}; recent lines:\n{transcript}"
+        )
+
+    def expect_next_exact(self, expected: str, timeout: float = 2.0) -> str:
+        deadline = time.time() + timeout
+        while time.time() < deadline and not self.lines:
+            self.read_available(0.1)
+        if not self.lines:
+            raise AssertionError(f"{self.label}: expected next frame {expected!r}; no frame received")
+        actual, ending = self._pop_line(0)
+        self._require_crlf(actual, ending)
+        if actual != expected:
+            transcript = "\n".join(self.lines[-20:])
+            raise AssertionError(
+                f"{self.label}: expected next frame {expected!r}, got {actual!r}; "
+                f"remaining lines:\n{transcript}"
+            )
+        return actual
+
+    def expect_regex(self, pattern: str, timeout: float = 2.0) -> str:
+        deadline = time.time() + timeout
+        while time.time() < deadline:
+            for index, line in enumerate(self.lines):
+                if re.fullmatch(pattern, line):
+                    matched, ending = self._pop_line(index)
+                    self._require_crlf(matched, ending)
+                    return matched
+            self.read_available(0.1)
+        transcript = "\n".join(self.lines[-20:])
+        raise AssertionError(
+            f"{self.label}: expected frame matching {pattern!r}; recent lines:\n{transcript}"
+        )
+
+    def wait_closed(self, timeout: float = 3.0) -> None:
+        deadline = time.time() + timeout
+        while time.time() < deadline and not self.closed:
+            self.read_available(0.1)
+        if not self.closed:
+            raise AssertionError(f"{self.label}: connection did not close within {timeout}s")
+
+    def hostmask(self, nick: str, user: str) -> str:
+        address = self.sock.getsockname()
+        host = str(address[0])
+        port = int(address[1])
+        peer = f"[{host}]:{port}" if ":" in host else f"{host}:{port}"
+        return f"{nick}!{user}@{peer}"
+
     def close(self) -> None:
         try:
             self.sock.close()
         except OSError:
             pass
+        self.closed = True
+
+    def _pop_line(self, index: int) -> Tuple[str, bytes]:
+        line = self.lines.pop(index)
+        ending = self.endings.pop(index)
+        return line, ending
+
+    def _require_crlf(self, line: str, ending: bytes) -> None:
+        if ending != b"\r\n":
+            raise AssertionError(
+                f"{self.label}: frame {line!r} ended with {ending!r}, expected b'\\r\\n'"
+            )
 
     def _auto_reply_to_ping(self, line: str) -> None:
         if not self.auto_pong:
@@ -76,10 +158,24 @@ def register(host: str, port: int, password: str, nick: str, realname: str) -> I
     peer.send_line(f"PASS {password}")
     peer.send_line(f"NICK {nick}")
     peer.send_line(f"USER {nick} 0 * :{realname}")
-    peer.expect(" 001 ")
+    peer.expect_next_exact(
+        f":irc.relay.local 001 {nick} :Welcome to irc-relay-server, {nick}"
+    )
+    peer.expect_next_exact(f":irc.relay.local 002 {nick} :Your host is irc.relay.local")
+    peer.expect_next_exact(
+        f":irc.relay.local 003 {nick} :This server is running a C++17 event backend"
+    )
     return peer
 
 
+def metrics_pattern(nick: str) -> str:
+    return (
+        rf":irc\.relay\.local NOTICE {re.escape(nick)} :"
+        r"connections=\d+ accepted=\d+ closed=\d+ rooms=\d+ commands=\d+ "
+        r"messages=\d+ queue_drops=\d+ rate_limited=\d+"
+    )
+
+
 def main() -> int:
     if len(sys.argv) != 4:
         print(f"usage: {sys.argv[0]} <host> <port> <password>", file=sys.stderr)
@@ -93,7 +189,7 @@ def main() -> int:
     try:
         wrong = IrcPeer(host, port, "wrong-password")
         wrong.send_line("PASS wrong-password")
-        wrong.expect(" 464 ")
+        wrong.expect_next_exact(":irc.relay.local 464 * :Password incorrect")
         wrong.close()
 
         alice = register(host, port, password, "alice", "Alice Learner")
@@ -102,95 +198,101 @@ def main() -> int:
         alice.send_raw(b"PI")
         time.sleep(0.05)
         alice.send_raw(b"NG :half-frame\r\n")
-        pong = alice.expect("PONG")
-        if "half-frame" not in pong:
-            raise AssertionError(f"alice: split PING response did not echo token: {pong}")
+        alice.expect_next_exact(":irc.relay.local PONG irc.relay.local half-frame")
 
         dup = register(host, port, password, "dupe", "Dupe One")
         peers.append(dup)
         collision = IrcPeer(host, port, "collision")
         collision.send_line(f"PASS {password}")
         collision.send_line("NICK dupe")
-        collision.expect(" 433 ")
+        collision.expect_next_exact(":irc.relay.local 433 * dupe :Nickname is already in use")
         collision.close()
 
+        alice_prefix = alice.hostmask("alice", "alice")
         alice.send_line("JOIN #edu")
-        alice.expect(" JOIN #edu")
-        alice.expect(" 353 ")
-        alice.expect(" 366 ")
+        alice.expect_next_exact(f":{alice_prefix} JOIN #edu")
+        alice.expect_next_exact(":irc.relay.local 331 alice #edu :No topic is set")
+        alice.expect_next_exact(":irc.relay.local 353 alice = #edu @alice")
+        alice.expect_next_exact(":irc.relay.local 366 alice #edu :End of /NAMES list")
         alice.send_line("LIST #edu")
-        alice.expect(" 322 ")
-        alice.expect(" 323 ")
+        alice.expect_next_exact(":irc.relay.local 321 alice Channel Users Name")
+        alice.expect_next_exact(":irc.relay.local 322 alice #edu 1 :open room")
+        alice.expect_next_exact(":irc.relay.local 323 alice :End of /LIST")
         alice.send_line("NAMES #edu")
-        alice.expect(" 353 ")
-        alice.expect(" 366 ")
+        alice.expect_next_exact(":irc.relay.local 353 alice = #edu @alice")
+        alice.expect_next_exact(":irc.relay.local 366 alice #edu :End of /NAMES list")
 
         bob = register(host, port, password, "bob", "Bob Learner")
         carol = register(host, port, password, "carol", "Carol Learner")
         peers.extend([bob, carol])
 
+        bob_prefix = bob.hostmask("bob", "bob")
         bob.send_line("JOIN #edu")
-        bob.expect(" JOIN #edu")
-        alice.expect(":bob!")
+        bob.expect_exact(f":{bob_prefix} JOIN #edu")
+        alice.expect_exact(f":{bob_prefix} JOIN #edu")
 
         alice.send_line("TOPIC #edu :Protocol lab")
-        alice.expect(" TOPIC #edu :Protocol lab")
+        alice.expect_exact(f":{alice_prefix} TOPIC #edu :Protocol lab")
 
         alice.send_line("PRIVMSG #edu :hello channel")
-        bob.expect(" PRIVMSG #edu :hello channel")
+        bob.expect_exact(f":{alice_prefix} PRIVMSG #edu :hello channel")
 
         alice.send_line("INVITE carol #edu")
-        alice.expect(" 341 ")
-        carol.expect(" INVITE carol #edu")
+        alice.expect_exact(":irc.relay.local 341 alice carol #edu")
+        carol.expect_exact(f":{alice_prefix} INVITE carol #edu")
         carol.send_line("JOIN #edu")
-        carol.expect(" JOIN #edu")
+        carol.expect_exact(f":{carol.hostmask('carol', 'carol')} JOIN #edu")
 
         alice.send_line("MODE #edu +i")
-        alice.expect(" MODE #edu +i")
+        alice.expect_exact(f":{alice_prefix} MODE #edu +i")
         alice.send_line("MODE #edu +o bob")
-        bob.expect(" MODE #edu +o bob")
+        bob.expect_exact(f":{alice_prefix} MODE #edu +o bob")
 
         bob.send_line("TOPIC #edu :Bob can set topics")
-        bob.expect(" TOPIC #edu :Bob can set topics")
+        bob.expect_exact(f":{bob_prefix} TOPIC #edu :Bob can set topics")
 
         bob.send_line("KICK #edu carol :practice complete")
-        carol.expect(" KICK #edu carol :practice complete")
+        carol.expect_exact(f":{bob_prefix} KICK #edu carol :practice complete")
 
         bob.send_line("PART #edu :done")
-        bob.expect(" PART #edu done")
+        bob.expect_exact(f":{bob_prefix} PART #edu done")
 
         alice.send_line("MODE #edu")
-        alice.expect(" 324 ")
+        alice.expect_exact(":irc.relay.local 324 alice #edu +it")
 
         alice.send_line("PRIVMSG bob :direct hello")
-        bob.expect(" PRIVMSG bob :direct hello")
-
-        idle = register(host, port, password, "idle", "Idle Tester")
-        peers.append(idle)
-        idle.expect(" PING ", timeout=5.0)
-        idle.send_line("PING :still-alive")
-        idle.expect(" PONG ")
-
-        flood = register(host, port, password, "flood", "Flood Tester")
-        for index in range(25):
-            flood.send_line(f"PING :burst-{index}")
-        flood.expect(" 439 ")
-        flood.close()
+        bob.expect_exact(f":{alice_prefix} PRIVMSG bob :direct hello")
 
         alice.send_line("METRICS")
-        alice.expect(" NOTICE alice :connections=")
+        alice.expect_regex(metrics_pattern("alice"))
 
         bots: List[IrcPeer] = []
         for index in range(6):
             bot = register(host, port, password, f"bot{index}", f"Bot {index}")
             bot.send_line("JOIN #load")
-            bot.expect(" JOIN #load")
+            bot.expect_exact(f":{bot.hostmask(f'bot{index}', f'bot{index}')} JOIN #load")
             bots.append(bot)
         peers.extend(bots)
         bots[0].send_line("PRIVMSG #load :load hello")
-        bots[1].expect(" PRIVMSG #load :load hello")
+        bots[1].expect_exact(
+            f":{bots[0].hostmask('bot0', 'bot0')} PRIVMSG #load :load hello"
+        )
+
+        flood = register(host, port, password, "flood", "Flood Tester")
+        for index in range(25):
+            flood.send_line(f"PING :burst-{index}")
+        flood.expect_exact(":irc.relay.local 439 flood :Command rate limit exceeded")
+        flood.close()
 
         alice.send_line("QUIT :smoke complete")
+        time.sleep(0.05)
+
+        idle = register(host, port, password, "idle", "Idle Tester")
+        peers.append(idle)
+        idle.expect_regex(r":irc\.relay\.local PING heartbeat-\d+-\d+", timeout=5.0)
+        idle.send_line("METRICS")
+        idle.expect_regex(metrics_pattern("idle"))
+
         time.sleep(0.05)
         return 0
     finally:


