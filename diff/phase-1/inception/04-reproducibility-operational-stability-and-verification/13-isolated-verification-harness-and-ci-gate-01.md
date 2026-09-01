# 격리형 검증 하네스와 CI 게이트

## `test(static): 스택 소스 계약 검사`

diff --git a/Makefile b/Makefile
index 11c11cf..6101c00 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,7 @@ ENV_FILE ?= .env
 
 COMPOSE_RUN := $(COMPOSE) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up down build logs ps clean fclean config
+.PHONY: up down build logs ps clean fclean test config
 
 up:
 	$(COMPOSE_RUN) up -d
@@ -28,3 +28,6 @@ fclean:
 
 config:
 	$(COMPOSE_RUN) config
+
+test:
+	python3 tests/validate_stack.py
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
new file mode 100755
index 0000000..4c73640
--- /dev/null
+++ b/tests/validate_stack.py
@@ -0,0 +1,168 @@
+#!/usr/bin/env python3
+from pathlib import Path
+import re
+import stat
+import sys
+
+
+ROOT = Path(__file__).resolve().parents[1]
+COMPOSE = ROOT / "srcs" / "docker-compose.yml"
+
+
+def fail(message: str) -> None:
+    print(f"FAIL: {message}", file=sys.stderr)
+    sys.exit(1)
+
+
+def require_file(path: str) -> Path:
+    full_path = ROOT / path
+    if not full_path.is_file():
+        fail(f"missing required file: {path}")
+    return full_path
+
+
+def require_text(path: str, patterns: list[str]) -> str:
+    text = require_file(path).read_text()
+    for pattern in patterns:
+        if not re.search(pattern, text, re.MULTILINE):
+            fail(f"{path} does not match {pattern!r}")
+    return text
+
+
+def require_executable(path: str) -> None:
+    mode = require_file(path).stat().st_mode
+    if not mode & stat.S_IXUSR:
+        fail(f"{path} must be executable")
+
+
+def validate_source_only() -> None:
+    forbidden = [
+        "docs",
+        "notes",
+        "evidence",
+        "PLAN.md",
+        "FAILURE_CASES.md",
+        "COMMIT_SCENARIO.md",
+        "TIMELINE.md",
+        "docker-compose.yml",
+        "conf",
+        "src",
+        "include",
+    ]
+    for item in forbidden:
+        if (ROOT / item).exists():
+            fail(f"forbidden final path exists: {item}")
+
+
+def validate_compose() -> None:
+    text = require_text(
+        "srcs/docker-compose.yml",
+        [
+            r"services:",
+            r"^\s+nginx:",
+            r"^\s+mariadb:",
+            r"^\s+wordpress:",
+            r"\"443:443\"",
+            r"condition: service_healthy",
+            r"healthcheck:",
+            r"secrets:",
+            r"mariadb_data:",
+            r"wordpress_data:",
+        ],
+    )
+    if re.search(r"(^|\s)-\s*[\"']?80:", text):
+        fail("nginx must not publish port 80")
+    if "mysqladmin ping -h127.0.0.1 -uroot" in text:
+        fail("mariadb healthcheck must not require TCP root login")
+    if not re.search(
+        r"mysqladmin\s+--socket=/run/mysqld/mysqld\.sock\s+-uroot\s+-p.+\s+ping\s+--silent",
+        text,
+    ):
+        fail("mariadb healthcheck must use the local socket as root")
+    if not re.search(
+        r"REQUEST_METHOD=GET\s+SCRIPT_NAME=/ping\s+SCRIPT_FILENAME=/ping\s+cgi-fcgi",
+        text,
+    ):
+        fail("wordpress healthcheck must call the FPM ping endpoint as a GET request")
+    for image in ("wordpress:", "mariadb:", "nginx:"):
+        if re.search(rf"image:\s*{image}", text):
+            fail(f"compose must not use the official {image.rstrip(':')} image directly")
+
+
+def validate_dockerfiles() -> None:
+    services = {
+        "nginx": [
+            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"apt-get install|apk add",
+            r"COPY conf/nginx\.conf",
+            r"EXPOSE 443",
+        ],
+        "mariadb": [
+            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"mariadb-server",
+            r"rm -rf /var/lib/mysql/\*",
+            r"COPY conf/50-server\.cnf",
+            r"ENTRYPOINT",
+        ],
+        "wordpress": [
+            r"FROM\s+debian:bookworm-slim|FROM\s+alpine:",
+            r"php8\.2-fpm|php-fpm",
+            r"wp-cli\.phar",
+            r"EXPOSE 9000",
+        ],
+    }
+    for service, patterns in services.items():
+        require_text(f"srcs/requirements/{service}/Dockerfile", patterns)
+        require_executable(f"srcs/requirements/{service}/tools/docker-entrypoint.sh")
+
+
+def validate_configs() -> None:
+    require_text(
+        "srcs/requirements/nginx/conf/nginx.conf",
+        [
+            r"listen 443 ssl http2",
+            r"fastcgi_pass wordpress:9000",
+            r"ssl_certificate",
+            r"location = /healthz",
+        ],
+    )
+    if "http2 on;" in require_file("srcs/requirements/nginx/conf/nginx.conf").read_text():
+        fail("nginx config must use Debian-compatible listen http2 syntax")
+    require_text(
+        "srcs/requirements/mariadb/conf/50-server.cnf",
+        [r"bind-address=0\.0\.0\.0", r"character-set-server=utf8mb4"],
+    )
+    require_text(
+        "srcs/requirements/wordpress/conf/www.conf",
+        [r"listen = 0\.0\.0\.0:9000", r"ping\.path = /ping"],
+    )
+
+
+def validate_env_policy() -> None:
+    env_text = require_file(".env.example").read_text()
+    for key in (
+        "DOMAIN_NAME",
+        "MYSQL_DATABASE",
+        "MYSQL_USER",
+        "DB_ROOT_PASSWORD_FILE",
+        "DB_PASSWORD_FILE",
+        "WP_ADMIN_PASSWORD_FILE",
+        "WP_USER_PASSWORD_FILE",
+    ):
+        if f"{key}=" not in env_text:
+            fail(f".env.example is missing {key}")
+    if re.search(r"PASSWORD=change-me", env_text):
+        fail(".env.example must point to secret files instead of embedding passwords")
+
+
+def main() -> None:
+    validate_source_only()
+    validate_compose()
+    validate_dockerfiles()
+    validate_configs()
+    validate_env_policy()
+    print("static stack validation passed")
+
+
+if __name__ == "__main__":
+    main()


## `test(compose): 렌더링된 Compose 설정 검사`

diff --git a/Makefile b/Makefile
index 6101c00..4a1ecf0 100644
--- a/Makefile
+++ b/Makefile
@@ -31,3 +31,9 @@ config:
 
 test:
 	python3 tests/validate_stack.py
+	@if command -v docker >/dev/null 2>&1 && docker compose version >/dev/null 2>&1; then \
+		$(COMPOSE) --env-file .env.example -f $(COMPOSE_FILE) config >/dev/null; \
+		echo "docker compose config passed"; \
+	else \
+		echo "docker compose not available; skipped compose config"; \
+	fi


## `test(bootstrap): 격리된 런타임 하네스 추가`

diff --git a/Makefile b/Makefile
index d6aded2..d211751 100644
--- a/Makefile
+++ b/Makefile
@@ -6,7 +6,7 @@ WAIT_TIMEOUT ?= 300
 
 COMPOSE_RUN := $(COMPOSE) --project-name $(PROJECT_NAME) --env-file $(ENV_FILE) -f $(COMPOSE_FILE)
 
-.PHONY: up start-database start-application down build logs ps clean fclean test config smoke
+.PHONY: up start-database start-application down build logs ps clean fclean test config smoke bootstrap-test
 
 up:
 	python3 tools/start_stack.py start --project "$(PROJECT_NAME)" --env-file "$(ENV_FILE)" --wait-timeout "$(WAIT_TIMEOUT)"
@@ -48,3 +48,6 @@ test:
 
 smoke:
 	tools/smoke_https.sh
+
+bootstrap-test:
+	python3 tests/runtime_stack.py bootstrap
diff --git a/tests/runtime_stack.py b/tests/runtime_stack.py
new file mode 100644
index 0000000..a88c538
--- /dev/null
+++ b/tests/runtime_stack.py
@@ -0,0 +1,430 @@
+#!/usr/bin/env python3
+"""격리된 Compose 프로젝트에서 컨테이너 스택의 실제 동작을 검사합니다."""
+
+from __future__ import annotations
+
+import argparse
+import json
+import os
+from pathlib import Path
+import secrets
+import shutil
+import socket
+import stat
+import subprocess
+import sys
+import tempfile
+
+
+ROOT = Path(__file__).resolve().parents[1]
+COMPOSE_FILE = ROOT / "srcs" / "docker-compose.yml"
+CONTROL_TIMEOUT_SECONDS = 330
+BUILD_TIMEOUT_SECONDS = 1200
+PROCESS_TIMEOUT_SECONDS = 60
+PORT_RETRY_LIMIT = 3
+PORT_CONFLICT_MARKERS = (
+    "address already in use",
+    "bind for 127.0.0.1",
+    "port is already allocated",
+)
+
+
+class StackError(RuntimeError):
+    pass
+
+
+def require_command(name: str) -> None:
+    if shutil.which(name) is None:
+        raise StackError(f"필요한 명령을 찾을 수 없습니다: {name}")
+
+
+def reserve_port() -> int:
+    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
+        listener.bind(("127.0.0.1", 0))
+        return int(listener.getsockname()[1])
+
+
+def write_private(path: Path, value: str) -> None:
+    descriptor = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
+    with os.fdopen(descriptor, "w", encoding="utf-8") as stream:
+        stream.write(value)
+        stream.write("\n")
+    if stat.S_IMODE(path.stat().st_mode) != 0o600:
+        raise StackError(f"비밀 파일 권한이 0600이 아닙니다: {path}")
+
+
+def replace_private(path: Path, value: str) -> None:
+    temporary = path.with_name(f".{path.name}.{secrets.token_hex(4)}")
+    write_private(temporary, value.rstrip("\n"))
+    os.replace(temporary, path)
+
+
+class RuntimeStack:
+    def __init__(self, *, keep: bool, diagnostics_dir: Path | None) -> None:
+        self.keep = keep
+        self.diagnostics_dir = diagnostics_dir
+        self.temp = Path(tempfile.mkdtemp(prefix="container-stack-e2e-"))
+        self.temp.chmod(0o700)
+        self.project = f"container-stack-{os.getpid()}-{secrets.token_hex(3)}"
+        self.domain = "stack.test"
+        self.port = reserve_port()
+        self.env_file = self.temp / ".env"
+        self.started = False
+        self.credential_values = {
+            "db_root_password.txt": f"root#-{secrets.token_urlsafe(24)}",
+            "db_password.txt": f"db#-{secrets.token_urlsafe(24)}",
+            "wp_admin_password.txt": f"admin-{secrets.token_urlsafe(24)}",
+            "wp_user_password.txt": f"user-{secrets.token_urlsafe(24)}",
+        }
+        try:
+            self._prepare_environment()
+        except Exception:
+            shutil.rmtree(self.temp, ignore_errors=True)
+            raise
+
+    def _prepare_environment(self) -> None:
+        for filename, value in self.credential_values.items():
+            write_private(self.temp / filename, value)
+
+        self.environment_values = {
+            "DOMAIN_NAME": self.domain,
+            "WORDPRESS_URL": f"https://{self.domain}:{self.port}",
+            "HTTPS_BIND_ADDRESS": "127.0.0.1",
+            "HTTPS_PORT": str(self.port),
+            "STACK_IMAGE_PREFIX": f"{self.project}-image",
+            "STACK_IMAGE_TAG": "local",
+            "MYSQL_DATABASE": "wordpress",
+            "MYSQL_USER": "wpuser",
+            "WORDPRESS_TITLE": "Container Stack E2E",
+            "WORDPRESS_ADMIN_USER": "administrator",
+            "WORDPRESS_ADMIN_EMAIL": "administrator@example.test",
+            "WORDPRESS_USER": "author",
+            "WORDPRESS_USER_EMAIL": "author@example.test",
+            "DB_ROOT_PASSWORD_FILE": str(self.temp / "db_root_password.txt"),
+            "DB_PASSWORD_FILE": str(self.temp / "db_password.txt"),
+            "WP_ADMIN_PASSWORD_FILE": str(self.temp / "wp_admin_password.txt"),
+            "WP_USER_PASSWORD_FILE": str(self.temp / "wp_user_password.txt"),
+        }
+        self._write_environment(create=True)
+
+    def _write_environment(self, *, create: bool = False) -> None:
+        content = "".join(
+            f"{key}={value}\n" for key, value in self.environment_values.items()
+        )
+        if create:
+            write_private(self.env_file, content.rstrip("\n"))
+        else:
+            replace_private(self.env_file, content)
+
+    def _select_new_port(self) -> None:
+        self.port = reserve_port()
+        self.environment_values["HTTPS_PORT"] = str(self.port)
+        self.environment_values["WORDPRESS_URL"] = (
+            f"https://{self.domain}:{self.port}"
+        )
+        self._write_environment()
+
+    def compose_command(self, *arguments: str) -> list[str]:
+        return [
+            "docker",
+            "compose",
+            "--project-name",
+            self.project,
+            "--env-file",
+            str(self.env_file),
+            "--file",
+            str(COMPOSE_FILE),
+            *arguments,
+        ]
+
+    def run_compose(
+        self,
+        *arguments: str,
+        capture: bool = False,
+        check: bool = True,
+        timeout: int = CONTROL_TIMEOUT_SECONDS,
+    ) -> subprocess.CompletedProcess[str]:
+        try:
+            return subprocess.run(
+                self.compose_command(*arguments),
+                cwd=ROOT,
+                check=check,
+                text=True,
+                capture_output=capture,
+                timeout=timeout,
+            )
+        except subprocess.TimeoutExpired as error:
+            raise StackError(
+                f"Compose 명령이 {timeout}초 안에 끝나지 않았습니다"
+            ) from error
+
+    def _run_start(
+        self,
+        action: str,
+        *,
+        build: bool = False,
+        check: bool = True,
+    ) -> subprocess.CompletedProcess[str]:
+        command = [
+            sys.executable,
+            str(ROOT / "tools" / "start_stack.py"),
+            action,
+            "--project",
+            self.project,
+            "--env-file",
+            str(self.env_file),
+            "--wait-timeout",
+            "300",
+        ]
+        if build:
+            command.append("--build")
+        try:
+            result = subprocess.run(
+                command,
+                cwd=ROOT,
+                text=True,
+                capture_output=True,
+                timeout=BUILD_TIMEOUT_SECONDS,
+            )
+        except subprocess.TimeoutExpired as error:
+            raise StackError("스택 기동이 제한 시간 안에 끝나지 않았습니다") from error
+        if check and result.returncode != 0:
+            raise StackError(
+                f"스택 기동이 실패했습니다: "
+                f"{result.stderr.strip() or result.stdout.strip()}"
+            )
+        return result
+
+    def start(self) -> None:
+        self.started = True
+        for attempt in range(PORT_RETRY_LIMIT):
+            result = self._run_start("start", build=True, check=False)
+            if result.returncode == 0:
+                return
+            output = (result.stdout + result.stderr).lower()
+            if (
+                attempt + 1 >= PORT_RETRY_LIMIT
+                or not any(marker in output for marker in PORT_CONFLICT_MARKERS)
+            ):
+                raise StackError(
+                    f"스택 기동이 실패했습니다: "
+                    f"{result.stderr.strip() or result.stdout.strip()}"
+                )
+            self.run_compose(
+                "down",
+                "--volumes",
+                "--remove-orphans",
+                "--timeout",
+                "20",
+                check=False,
+            )
+            self._select_new_port()
+
+    def wordpress(self, *arguments: str, capture: bool = False) -> str:
+        result = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "wp",
+            "--allow-root",
+            "--path=/var/www/html",
+            *arguments,
+            capture=capture,
+        )
+        return result.stdout.strip() if capture else ""
+
+    def inspect_service(self, service: str) -> dict[str, object]:
+        container_id = self.run_compose(
+            "ps", "--quiet", service, capture=True
+        ).stdout.strip()
+        if not container_id or "\n" in container_id:
+            raise StackError(f"{service} 컨테이너를 하나로 식별하지 못했습니다")
+        result = subprocess.run(
+            ["docker", "inspect", container_id],
+            check=True,
+            text=True,
+            capture_output=True,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        inspected = json.loads(result.stdout)
+        if not isinstance(inspected, list) or len(inspected) != 1:
+            raise StackError(f"{service} 컨테이너 검사 결과가 예상과 다릅니다")
+        return inspected[0]
+
+    def assert_runtime_secret_boundary(self) -> None:
+        forbidden_names = (
+            "MYSQL_ROOT_PASSWORD",
+            "MYSQL_PASSWORD",
+            "WORDPRESS_DB_PASSWORD",
+            "WORDPRESS_ADMIN_PASSWORD",
+            "WORDPRESS_USER_PASSWORD",
+        )
+        observed = ""
+        for service in ("mariadb", "wordpress", "nginx"):
+            inspected = self.inspect_service(service)
+            mounts = inspected.get("Mounts")
+            if not isinstance(mounts, list):
+                raise StackError(f"{service} 마운트 정보를 읽지 못했습니다")
+            destinations = {
+                str(mount.get("Destination", ""))
+                for mount in mounts
+                if isinstance(mount, dict)
+            }
+            if any(path.startswith("/run/secrets") for path in destinations):
+                raise StackError(f"{service} 런타임에 비밀 파일이 마운트되었습니다")
+            if service == "wordpress" and "/var/www/config" not in destinations:
+                raise StackError("WordPress 설정 전용 볼륨이 마운트되지 않았습니다")
+            if service == "nginx":
+                if "/var/www/config" in destinations:
+                    raise StackError("nginx가 WordPress 설정 전용 볼륨을 볼 수 있습니다")
+                hidden = self.run_compose(
+                    "exec",
+                    "--no-TTY",
+                    "nginx",
+                    "sh",
+                    "-ceu",
+                    "test -L /var/www/html/wp-config.php; "
+                    "test ! -e /var/www/html/wp-config.php; "
+                    "test ! -e /var/www/config/wp-config.php",
+                    capture=True,
+                    check=False,
+                )
+                if hidden.returncode != 0:
+                    raise StackError("nginx에서 WordPress 설정 파일이 격리되지 않았습니다")
+            config = inspected.get("Config")
+            if not isinstance(config, dict):
+                raise StackError(f"{service} 실행 환경을 읽지 못했습니다")
+            environment = config.get("Env") or []
+            config_text = "\n".join(str(item) for item in environment)
+            if any(name in config_text for name in forbidden_names):
+                raise StackError(f"{service} 런타임 환경에 비밀번호 변수가 남았습니다")
+            observed += config_text
+            container_id = str(inspected.get("Id", ""))
+            observed += subprocess.run(
+                ["docker", "top", container_id, "-eo", "pid,args"],
+                check=True,
+                text=True,
+                capture_output=True,
+                timeout=PROCESS_TIMEOUT_SECONDS,
+            ).stdout
+        for value in self.credential_values.values():
+            if value in observed:
+                raise StackError("런타임 환경이나 프로세스 인자에 비밀값이 남았습니다")
+        config = self.run_compose(
+            "exec",
+            "--no-TTY",
+            "wordpress",
+            "cat",
+            "/var/www/config/wp-config.php",
+            capture=True,
+        ).stdout
+        if self.credential_values["db_password.txt"] not in config:
+            raise StackError("wp-config.php에 DB 자격증명이 없습니다")
+        for filename in (
+            "db_root_password.txt",
+            "wp_admin_password.txt",
+            "wp_user_password.txt",
+        ):
+            if self.credential_values[filename] in config:
+                raise StackError(f"wp-config.php에 불필요한 비밀값이 남았습니다: {filename}")
+
+
+    def verify_bootstrap(self) -> None:
+        self.start()
+        self.assert_runtime_secret_boundary()
+        for service, marker in (
+            ("mariadb", "/var/lib/mysql-volume/data/.container-stack-initialized"),
+            ("wordpress", "/var/www/html/.container-stack-initialized"),
+        ):
+            result = self.run_compose(
+                "exec",
+                "--no-TTY",
+                service,
+                "test",
+                "-f",
+                marker,
+                capture=True,
+                check=False,
+            )
+            if result.returncode != 0:
+                raise StackError(f"{service} 초기화 완료 표식이 없습니다")
+        print("bootstrap completion and secret boundary passed")
+
+
+    def collect_diagnostics(self) -> Path:
+        destination = self.diagnostics_dir
+        if destination is None:
+            destination = Path(tempfile.mkdtemp(prefix="container-stack-diagnostics-"))
+            destination.chmod(0o700)
+        else:
+            destination.mkdir(parents=True, exist_ok=True)
+        commands = {
+            "compose-ps.txt": ("ps", "--all"),
+            "compose-logs.txt": ("logs", "--no-color", "--timestamps"),
+            "compose-config.txt": ("config", "--no-interpolate"),
+        }
+        for filename, arguments in commands.items():
+            result = self.run_compose(*arguments, capture=True, check=False)
+            (destination / filename).write_text(
+                result.stdout + result.stderr, encoding="utf-8"
+            )
+        print(f"진단 자료: {destination}", file=sys.stderr)
+        return destination
+
+    def close(self, *, failed: bool) -> None:
+        if failed:
+            try:
+                self.collect_diagnostics()
+            except OSError as error:
+                print(f"진단 자료를 저장하지 못했습니다: {error}", file=sys.stderr)
+        if self.started and not self.keep:
+            self.run_compose(
+                "down", "--volumes", "--remove-orphans", "--timeout", "20", check=False
+            )
+        if self.keep:
+            print(
+                f"검사용 프로젝트를 유지합니다: {self.project} ({self.temp})",
+                file=sys.stderr,
+            )
+        else:
+            shutil.rmtree(self.temp, ignore_errors=True)
+
+
+def parse_arguments() -> argparse.Namespace:
+    parser = argparse.ArgumentParser(description="격리된 컨테이너 스택 검증")
+    parser.add_argument("scenario", choices=("bootstrap",))
+    parser.add_argument("--keep", action="store_true", help="검사 뒤 프로젝트를 유지합니다")
+    parser.add_argument("--diagnostics-dir", type=Path)
+    return parser.parse_args()
+
+
+def main() -> int:
+    args = parse_arguments()
+    try:
+        require_command("docker")
+        require_command("curl")
+        subprocess.run(
+            ["docker", "compose", "version"],
+            check=True,
+            stdout=subprocess.DEVNULL,
+            timeout=PROCESS_TIMEOUT_SECONDS,
+        )
+        stack = RuntimeStack(keep=args.keep, diagnostics_dir=args.diagnostics_dir)
+    except (OSError, StackError, subprocess.SubprocessError) as error:
+        print(f"검증 환경을 준비하지 못했습니다: {error}", file=sys.stderr)
+        return 2
+
+    failed = True
+    try:
+        stack.verify_bootstrap()
+        failed = False
+        return 0
+    except (OSError, StackError, subprocess.SubprocessError) as error:
+        print(f"{args.scenario} 검증 실패: {error}", file=sys.stderr)
+        return 1
+    finally:
+        stack.close(failed=failed)
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/tests/validate_stack.py b/tests/validate_stack.py
index cbbfc33..e345099 100755
--- a/tests/validate_stack.py
+++ b/tests/validate_stack.py
@@ -178,6 +178,15 @@ def validate_tools() -> None:
             r"^start-application:",
             r"^smoke:",
             r"tools/smoke_https\.sh",
+            r"^bootstrap-test:",
+            r"runtime_stack\.py bootstrap",
+        ],
+    )
+    require_text(
+        "tests/runtime_stack.py",
+        [
+            r"--project-name",
+            r"tools.+start_stack\.py",
         ],
     )
 


