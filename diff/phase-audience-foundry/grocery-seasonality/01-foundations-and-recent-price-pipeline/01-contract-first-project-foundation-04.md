## `build: pin django runtime`

diff --git a/.env.example b/.env.example
new file mode 100644
index 0000000..5ec6723
--- /dev/null
+++ b/.env.example
@@ -0,0 +1,7 @@
+# 로컬 예시값뿐이다. production 값이나 KAMIS key를 commit하지 않는다.
+DJANGO_DEBUG=0
+DJANGO_SECRET_KEY=replace-in-secret-store
+DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
+DATABASE_URL=postgresql://grocery:local-grocery-only@127.0.0.1:55432/grocery
+KAMIS_API_KEY=
+DEPLOY_VERSION=dev
diff --git a/.gitignore b/.gitignore
index 7629239..0df0fa3 100644
--- a/.gitignore
+++ b/.gitignore
@@ -17,4 +17,6 @@ tmp/
 .mypy_cache/
 .pytest_cache/
 .ruff_cache/
-
+artifacts/
+playwright-report/
+test-results/
diff --git a/.python-version b/.python-version
new file mode 100644
index 0000000..a128d5c
--- /dev/null
+++ b/.python-version
@@ -0,0 +1 @@
+3.14.7
diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index f983c6b..755b7e5 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -1,13 +1,28 @@
 # 제3자 고지
 
-이 정책 전용 기준선에는 runtime dependency, vendored third-party source, 생성된
-third-party artifact 또는 가져온 제품 구현이 없습니다.
+이 프로젝트는 아래 구성 요소를 직접 또는 전이 의존성으로 사용한다. Python package의
+artifact URL과 SHA-256은 `uv.lock`에 고정한다. container는 tag와 multi-platform index
+digest를 함께 고정한다. 이 고지는 각 upstream license 원문을 대체하지 않는다.
 
-dependency 또는 import·generated material을 도입하는 동일한 격리 commit에서 이
-파일을 갱신합니다. 정확한 version 또는 revision, upstream source, license, 사용
-목적, integrity mechanism, 재배포 의무와 수용한 advisory·compatibility risk를
-기록합니다.
+| 구성 요소 | 고정 버전 | 사용 목적 | upstream | license |
+|---|---:|---|---|---|
+| Python | `3.14.7` | application runtime | `python.org` | PSF-2.0 |
+| Django | `5.2.17` | SSR, Forms, Auth, ORM, migration | `djangoproject.com` | BSD-3-Clause |
+| Gunicorn | `23.0.0` | 고정 production WSGI process | `gunicorn.org` | MIT |
+| Psycopg | `3.3.4` | PostgreSQL driver | `psycopg.org` | LGPL-3.0-only |
+| psycopg-binary | `3.3.4` | local candidate의 self-contained libpq runtime | `psycopg.org` | LGPL-3.0-only 및 wheel 내 고지 |
+| asgiref | `3.12.1` | Django 전이 runtime | `github.com/django/asgiref` | BSD-3-Clause |
+| packaging | `26.3` | Gunicorn 전이 runtime | `github.com/pypa/packaging` | Apache-2.0 OR BSD-2-Clause |
+| sqlparse | `0.6.0` | Django 전이 runtime | `github.com/andialbrecht/sqlparse` | BSD-3-Clause |
+| tzdata | `2026.3` | Windows 조건부 timezone data | `github.com/python/tzdata` | Apache-2.0 |
+| PostgreSQL official image | `18.6` | local DB·migration·restore rehearsal | `docker.io/library/postgres` | PostgreSQL License 및 image 내 고지 |
+| uv | `0.12.6` | Python·dependency·lock 실행 도구 | `github.com/astral-sh/uv` | Apache-2.0 OR MIT |
 
-동일한 material과 정확한 revision이 실제로 존재하지 않으면 다른 Audience Foundry
-제품의 고지를 복사하지 않습니다. vulnerability scan은 license·provenance 검토를
-대체하지 않습니다.
+PostgreSQL image index digest는
+`sha256:4ef4dbc939d61acea57712655ddb4b4ab27419c913f94cca0cd57cb3ea3c2280`다.
+`psycopg-binary`는 local candidate의 재현성을 위해 사용하며 bundle의 libpq·OpenSSL 등
+고지는 설치 wheel의 `licenses/`와 함께 배포한다. production platform이 system libpq를
+관리할 수 있으면 별도 호환성·license 검토 후 `psycopg[c]`로 전환하는 것은 새 기술 결정이다.
+
+프로젝트는 이 package source를 vendoring하거나 수정하지 않는다. vulnerability scan은
+license·provenance 검토를 대체하지 않으며, release gate에서 lock 전체를 다시 검사한다.
diff --git a/compose.yaml b/compose.yaml
new file mode 100644
index 0000000..f04d8ac
--- /dev/null
+++ b/compose.yaml
@@ -0,0 +1,21 @@
+name: audience-foundry-grocery-seasonality
+
+services:
+  db:
+    image: postgres:18.6@sha256:4ef4dbc939d61acea57712655ddb4b4ab27419c913f94cca0cd57cb3ea3c2280
+    environment:
+      POSTGRES_DB: grocery
+      POSTGRES_PASSWORD: local-grocery-only
+      POSTGRES_USER: grocery
+    healthcheck:
+      test: ["CMD-SHELL", "pg_isready -U grocery -d grocery"]
+      interval: 2s
+      timeout: 3s
+      retries: 15
+    ports:
+      - "127.0.0.1:55432:5432"
+    volumes:
+      - grocery-postgres:/var/lib/postgresql
+
+volumes:
+  grocery-postgres:
diff --git a/pyproject.toml b/pyproject.toml
new file mode 100644
index 0000000..0c34510
--- /dev/null
+++ b/pyproject.toml
@@ -0,0 +1,13 @@
+[project]
+name = "audience-foundry-grocery-seasonality"
+version = "0.1.0"
+requires-python = "==3.14.7"
+dependencies = [
+    "django==5.2.17",
+    "gunicorn==23.0.0",
+    "psycopg[binary]==3.3.4",
+]
+
+[tool.uv]
+package = false
+required-version = "==0.12.6"
diff --git a/uv.lock b/uv.lock
new file mode 100644
index 0000000..5382a49
--- /dev/null
+++ b/uv.lock
@@ -0,0 +1,117 @@
+version = 1
+revision = 3
+requires-python = "==3.14.7"
+
+[[package]]
+name = "asgiref"
+version = "3.12.1"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/e6/26/3b59f2bdae5f640389becb1f673cded775287f5fc4f816309d9ca9a3f93d/asgiref-3.12.1.tar.gz", hash = "sha256:59dcb51c272ad209d59bed5708a64a333083e86017d7fcdd67498eeab7784340", size = 42378, upload-time = "2026-07-14T09:56:18.087Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/c0/1b/54f4ad77cd8a584fa70746c47df988e002cf1ee1eba43364d46f87803647/asgiref-3.12.1-py3-none-any.whl", hash = "sha256:fe386d1c2bff7259ea95929266d12a8cf9a8b5a1c2598402967d8792e7a7c094", size = 25478, upload-time = "2026-07-14T09:56:16.926Z" },
+]
+
+[[package]]
+name = "audience-foundry-grocery-seasonality"
+version = "0.1.0"
+source = { virtual = "." }
+dependencies = [
+    { name = "django" },
+    { name = "gunicorn" },
+    { name = "psycopg", extra = ["binary"] },
+]
+
+[package.metadata]
+requires-dist = [
+    { name = "django", specifier = "==5.2.17" },
+    { name = "gunicorn", specifier = "==23.0.0" },
+    { name = "psycopg", extras = ["binary"], specifier = "==3.3.4" },
+]
+
+[[package]]
+name = "django"
+version = "5.2.17"
+source = { registry = "https://pypi.org/simple" }
+dependencies = [
+    { name = "asgiref" },
+    { name = "sqlparse" },
+    { name = "tzdata", marker = "sys_platform == 'win32'" },
+]
+sdist = { url = "https://files.pythonhosted.org/packages/d5/d8/43e9d000519adceb189620b6869ff88031e046df91c2e9da72f8f6918399/django-5.2.17.tar.gz", hash = "sha256:9d4d93be539a18ab80d058eb515900e10951e04c537c5a6b394fc49528d3251f", size = 10889740, upload-time = "2026-08-04T15:04:03.173Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/df/f8/ce120525ca78f12b07daf65786679c5d0b54a75285a8958d3ae55e39da35/django-5.2.17-py3-none-any.whl", hash = "sha256:f04fb3b36ee119e1af4fa1d397d5fd6cf12700f49321e84d4f4c642c5b1973db", size = 8315563, upload-time = "2026-08-04T15:03:59.1Z" },
+]
+
+[[package]]
+name = "gunicorn"
+version = "23.0.0"
+source = { registry = "https://pypi.org/simple" }
+dependencies = [
+    { name = "packaging" },
+]
+sdist = { url = "https://files.pythonhosted.org/packages/34/72/9614c465dc206155d93eff0ca20d42e1e35afc533971379482de953521a4/gunicorn-23.0.0.tar.gz", hash = "sha256:f014447a0101dc57e294f6c18ca6b40227a4c90e9bdb586042628030cba004ec", size = 375031, upload-time = "2024-08-10T20:25:27.378Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/cb/7d/6dac2a6e1eba33ee43f318edbed4ff29151a49b5d37f080aad1e6469bca4/gunicorn-23.0.0-py3-none-any.whl", hash = "sha256:ec400d38950de4dfd418cff8328b2c8faed0edb0d517d3394e457c317908ca4d", size = 85029, upload-time = "2024-08-10T20:25:24.996Z" },
+]
+
+[[package]]
+name = "packaging"
+version = "26.3"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/7d/fa/3944b40b07da9ce895c0e6303a5ab7d53da063554f534556b134a54d6093/packaging-26.3.tar.gz", hash = "sha256:94edc256424af38762eb31306eed28beb9f0efc50a8837492c9d6fd6004aed79", size = 313412, upload-time = "2026-08-04T18:15:28.737Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/63/34/ba1c580383c9eada3711951fef0795c80b829a078d72188184bcab9dd527/packaging-26.3-py3-none-any.whl", hash = "sha256:d7193f7c8e4e93f444fde0262bf90af30e16fa0ad0ad44cb553c87339b23cd1c", size = 129956, upload-time = "2026-08-04T18:15:27.159Z" },
+]
+
+[[package]]
+name = "psycopg"
+version = "3.3.4"
+source = { registry = "https://pypi.org/simple" }
+dependencies = [
+    { name = "tzdata", marker = "sys_platform == 'win32'" },
+]
+sdist = { url = "https://files.pythonhosted.org/packages/db/2f/cb91e5502ec9de1de6f1b76cfbf69531932725361168bb06963620c77e2e/psycopg-3.3.4.tar.gz", hash = "sha256:e21207764952cff81b6b8bdacad9a3939f2793367fdac2987b3aac36a651b5bc", size = 165799, upload-time = "2026-05-01T23:31:55.179Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/5c/e0/7b3dee031daae7743609ce3c746565d4a3ed7c2c186479eb48e34e838c64/psycopg-3.3.4-py3-none-any.whl", hash = "sha256:b6bbc25ccf05c8fad3b061d9db2ef0909a555171b84b07f29458a447253d679a", size = 213001, upload-time = "2026-05-01T23:20:50.816Z" },
+]
+
+[package.optional-dependencies]
+binary = [
+    { name = "psycopg-binary", marker = "implementation_name != 'pypy'" },
+]
+
+[[package]]
+name = "psycopg-binary"
+version = "3.3.4"
+source = { registry = "https://pypi.org/simple" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/48/a6/828c9185701dab71b234c2a76c38a08b098ebfec5020716b4e93807492b5/psycopg_binary-3.3.4-cp314-cp314-macosx_10_15_x86_64.whl", hash = "sha256:28b7398fdd19db3232c884fb24550bdfe951221f510e195e233299e4c9b78f97", size = 4607292, upload-time = "2026-05-01T23:30:38.962Z" },
+    { url = "https://files.pythonhosted.org/packages/92/58/5b40dbc9d839045c9dae956960e4fb6d20bcabe6c59a2aa34fc3a371913f/psycopg_binary-3.3.4-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:1fbaa292a3c8bb61b45df1ad3da1908ccee7cb889db9425e3557d9e34e2a4829", size = 4687023, upload-time = "2026-05-01T23:30:47.227Z" },
+    { url = "https://files.pythonhosted.org/packages/85/a9/793f0ac107a9003b48441d0d1f9f616d96e0f37458dd8dc12528ceff55fb/psycopg_binary-3.3.4-cp314-cp314-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:94596f9e7633ee3f6440711d43bb70aa31cc0a46a900ab8b4201a366ace5c9e7", size = 5486985, upload-time = "2026-05-01T23:30:55.517Z" },
+    { url = "https://files.pythonhosted.org/packages/8f/26/42e8533497e2592334f68ec529cf5f840f7fa4e99575a4bb61aa184dbfbf/psycopg_binary-3.3.4-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:8c0056529e68dbe9184cd4019a1f3d8f3a4ead2f6fc7a5afcf27d3314edd1277", size = 5168745, upload-time = "2026-05-01T23:31:01.904Z" },
+    { url = "https://files.pythonhosted.org/packages/15/af/b7151776cc08d5935d45c833ec818a9beb417cf7c08239af1aafbdae78ee/psycopg_binary-3.3.4-cp314-cp314-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:2c09aad7051326e7603c14e50636db9c01f78272dc54b3accff03d46370461e6", size = 6761486, upload-time = "2026-05-01T23:31:14.511Z" },
+    { url = "https://files.pythonhosted.org/packages/d0/ed/c92533b9124712d592cbf1cd6c76da933a2e0acea81dfe1fbe7e735f0cff/psycopg_binary-3.3.4-cp314-cp314-manylinux_2_38_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:514404ed543efd620c85602b747df2a23cf1241b4067199e1a66f2d2757aaa41", size = 4997427, upload-time = "2026-05-01T23:31:20.901Z" },
+    { url = "https://files.pythonhosted.org/packages/a2/23/ccadfd0de416aa188356daa199453af24087b042e296088706d190ae0295/psycopg_binary-3.3.4-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:46893c26858be12cc49ca4226ed6a60b4bfccadd946b3bebb783a60b38788228", size = 4533549, upload-time = "2026-05-01T23:31:26.204Z" },
+    { url = "https://files.pythonhosted.org/packages/fd/a0/c8f43cee36386f7bc891ab41a9d31ea07cf9826038e732da79f26b1e5f34/psycopg_binary-3.3.4-cp314-cp314-musllinux_1_2_ppc64le.whl", hash = "sha256:df1d567fc430f6df15c9fcf67d87685fc49bdb325adc0db5af1adfb2f44eb5c9", size = 4210256, upload-time = "2026-05-01T23:31:33.884Z" },
+    { url = "https://files.pythonhosted.org/packages/4e/2c/c1547871be3790676e8868b38655496422f94f0978dfb66b74bdba2f1676/psycopg_binary-3.3.4-cp314-cp314-musllinux_1_2_riscv64.whl", hash = "sha256:6b9016b1714da4dd5ecaaa75b82098aa5a0b87854ce9b092e21c27c4ae23e014", size = 3946204, upload-time = "2026-05-01T23:31:39.626Z" },
+    { url = "https://files.pythonhosted.org/packages/c4/b1/f6670f00fa7ea601584623f6c11602ab92117d83eaff885e0210f6de7418/psycopg_binary-3.3.4-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:47c656a8a7ba6eb0cff1801a4caaa9c8bdc12d03080e273aff1c8ac39971a77e", size = 4255811, upload-time = "2026-05-01T23:31:44.986Z" },
+    { url = "https://files.pythonhosted.org/packages/eb/e6/5fff07a70d1f945ed90ae131c3bd76cab32beff7c58c6db15ad5820b6d1f/psycopg_binary-3.3.4-cp314-cp314-win_amd64.whl", hash = "sha256:c37e024c07308cd06cf3ec51bfd0e7f6157585a4d84d1bce4a7f5f7913719bf8", size = 3666849, upload-time = "2026-05-01T23:31:51.165Z" },
+]
+
+[[package]]
+name = "sqlparse"
+version = "0.6.0"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/5f/d3/3f06a1006f2261d1342aefb3c71eed02f5d4ca5bdbecd86ebc12ad38306e/sqlparse-0.6.0.tar.gz", hash = "sha256:113c35c75365ab9cc9c7231d68c6428fb11c085fc8e9eb1ad659b7ddbf6cd2b9", size = 178477, upload-time = "2026-08-13T19:16:06.396Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/d9/50/f00935da0ec7cbf325f8dc4f772ae46fbc7b672dd62876e73f0a94adda57/sqlparse-0.6.0-py3-none-any.whl", hash = "sha256:b861c0288ce2fa56209a9a6412d2e066ac664b3873b89c26c9d8415e8e32996f", size = 50070, upload-time = "2026-08-13T19:16:04.062Z" },
+]
+
+[[package]]
+name = "tzdata"
+version = "2026.3"
+source = { registry = "https://pypi.org/simple" }
+sdist = { url = "https://files.pythonhosted.org/packages/92/ff/5a28bdfd8c3ebec42564ac7d0e54ca3db65044a9314a97f9564fa7a1e926/tzdata-2026.3.tar.gz", hash = "sha256:4a1518b8993086a7982523e071643f3c0e5f213e75b21318e78bcabfff9d1415", size = 198674, upload-time = "2026-07-10T08:50:37.887Z" }
+wheels = [
+    { url = "https://files.pythonhosted.org/packages/e5/6d/b53b99a9f2766d095985947a5782f1702cabb129a34f7a802d7197af832f/tzdata-2026.3-py2.py3-none-any.whl", hash = "sha256:dc096730c87af6cab1b171c9d532be840741ff5d459015e7f6947bd7d7e54931", size = 348168, upload-time = "2026-07-10T08:50:36.46Z" },
+]


