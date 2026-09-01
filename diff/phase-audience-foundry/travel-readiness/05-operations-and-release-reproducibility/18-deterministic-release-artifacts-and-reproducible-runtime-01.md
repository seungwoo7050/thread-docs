# 결정적 릴리스 산출물과 재현 가능한 런타임

## `build: pin travel readiness runtime`

diff --git a/.python-version b/.python-version
new file mode 100644
index 0000000..a128d5c
--- /dev/null
+++ b/.python-version
@@ -0,0 +1 @@
+3.14.7
diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index f983c6b..6fd3de0 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -1,13 +1,49 @@
 # 제3자 고지
 
-이 정책 전용 기준선에는 runtime dependency, vendored third-party source, 생성된
-third-party artifact 또는 가져온 제품 구현이 없습니다.
+이 파일은 `uv.lock` revision 3이 해석하는 Python `3.14.7` 환경과 runtime 결정을
+기준으로 합니다. repository에는 wheel, source archive, Python 또는 PostgreSQL binary를
+vendoring하지 않습니다.
 
-dependency 또는 import·generated material을 도입하는 동일한 격리 commit에서 이
-파일을 갱신합니다. 정확한 version 또는 revision, upstream source, license, 사용
-목적, integrity mechanism, 재배포 의무와 수용한 advisory·compatibility risk를
-기록합니다.
+## toolchain과 server 결정
 
-동일한 material과 정확한 revision이 실제로 존재하지 않으면 다른 Audience Foundry
-제품의 고지를 복사하지 않습니다. vulnerability scan은 license·provenance 검토를
-대체하지 않습니다.
+| 구성요소 | revision | license | upstream와 목적 |
+| --- | --- | --- | --- |
+| CPython | 3.14.7 | PSF License Version 2 | `https://www.python.org/`; application interpreter |
+| uv | 0.12.6, commit `7938ca5d53dbb9c614a4a030df406e41ff101ab9` | MIT OR Apache-2.0 | `https://github.com/astral-sh/uv`; resolver와 frozen installer |
+| PostgreSQL | 18.6 | PostgreSQL License | `https://www.postgresql.org/`; canonical database version decision |
+
+uv macOS arm64 archive SHA-256은
+`14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d`이며 release
+attestation을 GitHub CLI와 public Sigstore Rekor로 검증했습니다. PostgreSQL package/image,
+production Linux target과 WSGI server는 아직 선택하거나 내려받지 않았습니다.
+
+## lock에 포함된 Python distribution
+
+| dependency | 관계 | license | upstream | 사용 목적 |
+| --- | --- | --- | --- | --- |
+| Django 5.2.17 | direct | BSD-3-Clause | `https://github.com/django/django` | server-rendered web, models, migrations, Admin |
+| psycopg 3.3.4 | direct | LGPL-3.0-only | `https://github.com/psycopg/psycopg` | Django PostgreSQL adapter |
+| psycopg-binary 3.3.4 | direct extra | LGPL-3.0-only | `https://github.com/psycopg/psycopg` | pinned CPython adapter binary for development verification |
+| asgiref 3.12.1 | Django transitive | BSD-3-Clause | `https://github.com/django/asgiref` | Django sync/async compatibility |
+| sqlparse 0.6.0 | Django transitive | BSD-3-Clause | `https://github.com/andialbrecht/sqlparse` | Django SQL formatting support |
+| tzdata 2026.3 | Windows-only transitive | Apache-2.0 | `https://github.com/python/tzdata` | platforms without system IANA timezone data |
+
+각 registry artifact URL, upload time, size와 SHA-256은 `uv.lock`에 있습니다. 대표적으로
+Django universal wheel은
+`f04fb3b36ee119e1af4fa1d397d5fd6cf12700f49321e84d4f4c642c5b1973db`, psycopg
+universal wheel은 `b6bbc25ccf05c8fad3b061d9db2ef0909a555171b84b07f29458a447253d679a`,
+psycopg-binary CPython 3.14 macOS arm64 wheel은
+`1fbaa292a3c8bb61b45df1ad3da1908ccee7cb889db9425e3557d9e34e2a4829`입니다.
+
+BSD와 Apache 재배포 시 해당 copyright/license/notice를 유지해야 합니다. LGPL 구성요소를
+배포하면 LGPL/GPL license 사본, 고지, 수정·재링크 조건을 충족해야 합니다. 설치된 macOS
+psycopg-binary wheel에는 libpq, OpenSSL 3, Kerberos와 LDAP support library가 함께 있음을
+검사했습니다. exact production artifact와 그 bundled license inventory는 미결정이므로 이
+development lock만으로 production 재배포를 승인하지 않습니다. 설치 distribution 안의
+license 파일과 Django vendored asset 고지를 제거하지 않습니다.
+
+2026-08-30 exact-version PyPI metadata의 vulnerability 목록은 비어 있었고 lock에서 yanked
+artifact는 없었습니다. 이는 지속적인 advisory 감시나 bundled native library 검사를
+대체하지 않습니다. dependency가 application telemetry를 새로 활성화하지는 않으며, uv의
+network access는 install/resolve 때만 사용합니다. 제거 시 `pyproject.toml`에서 direct
+dependency를 삭제하고 같은 exact uv로 lock을 다시 만든 뒤 이 inventory를 함께 갱신합니다.
diff --git a/pyproject.toml b/pyproject.toml
new file mode 100644
index 0000000..dfe238c
--- /dev/null
+++ b/pyproject.toml
@@ -0,0 +1,13 @@
+[project]
+name = "audience-foundry-travel-readiness"
+version = "0.1.0"
+description = "Evidence-backed travel-readiness publication service"
+requires-python = "==3.14.7"
+dependencies = [
+    "Django==5.2.17",
+    "psycopg[binary]==3.3.4",
+]
+
+[tool.uv]
+required-version = "==0.12.6"
+package = false
diff --git a/runtime/versions.toml b/runtime/versions.toml
new file mode 100644
index 0000000..2295e2e
--- /dev/null
+++ b/runtime/versions.toml
@@ -0,0 +1,18 @@
+[runtime]
+python = "3.14.7"
+django = "5.2.17"
+postgresql = "18.6"
+uv = "0.12.6"
+psycopg = "3.3.4"
+psycopg_distribution = "binary-wheel"
+
+[integrity]
+uv_release_commit = "7938ca5d53dbb9c614a4a030df406e41ff101ab9"
+uv_macos_arm64_archive_sha256 = "14b459d51ea2e71eeba28c45a268c922bdf8607fc6455e3f40b4e082895d160d"
+uv_attestation = "verified-with-github-cli-and-public-sigstore-rekor"
+
+[delivery]
+production_wsgi = "UNRESOLVED"
+production_postgresql_package_or_image = "UNRESOLVED"
+production_linux_target = "UNRESOLVED"
+status = "development-only"
diff --git a/uv.lock b/uv.lock
new file mode 100644
index 0000000..94ea44a
--- /dev/null
+++ b/uv.lock
@@ -0,0 +1,94 @@
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
+name = "audience-foundry-travel-readiness"
+version = "0.1.0"
+source = { virtual = "." }
+dependencies = [
+    { name = "django" },
+    { name = "psycopg", extra = ["binary"] },
+]
+
+[package.metadata]
+requires-dist = [
+    { name = "django", specifier = "==5.2.17" },
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


