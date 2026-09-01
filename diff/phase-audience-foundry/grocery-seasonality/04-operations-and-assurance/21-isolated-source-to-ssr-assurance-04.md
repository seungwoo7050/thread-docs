## `fix(qa): make vnext fixture directly executable`

diff --git a/scripts/build_vnext_browser_fixture.py b/scripts/build_vnext_browser_fixture.py
index 16479f0..31b6d88 100644
--- a/scripts/build_vnext_browser_fixture.py
+++ b/scripts/build_vnext_browser_fixture.py
@@ -4,6 +4,12 @@
 from __future__ import annotations
 
 import os
+import sys
+from pathlib import Path
+
+REPOSITORY_ROOT = Path(__file__).resolve().parents[1]
+if str(REPOSITORY_ROOT) not in sys.path:
+    sys.path.insert(0, str(REPOSITORY_ROOT))
 
 os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
 


## `docs: record vnext local candidate completion`

diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
index 069c1d2..840fe12 100644
--- a/docs/COMPLETION-REPORT.md
+++ b/docs/COMPLETION-REPORT.md
@@ -4,10 +4,10 @@
 production 배포, traffic 공개나 `Phase 0 완료`를 주장하지 않는다.
 
 이 보고서는 tracked commit `d682908`까지의 Phase 0 증거를 보존합니다. 이후 frontend
-redesign commit이 추가되어 현재 vNext 시작 기준선은
-`bb0b28038243c539db2eafcfebc05144d9d59d66`이며, `main`은 GitHub 공개 remote
-`origin`과 동기화되어 있습니다. 아래의 `remote 없음`과 당시 SHA·검증값은 수정된 현재
-상태가 아니라 해당 시점의 역사적 사실로 읽습니다.
+redesign commit이 추가되어 vNext 시작 기준선은
+`bb0b28038243c539db2eafcfebc05144d9d59d66`입니다. vNext를 시작할 당시 `main`은 GitHub 공개
+remote `origin`과 동기화되어 있었습니다. 아래의 `remote 없음`과 당시 SHA·검증값은 수정된
+현재 상태가 아니라 해당 시점의 역사적 사실로 읽습니다.
 
 현재 상태는 **Phase 0 배포 직전 완료**다. 아래 결과는 local production candidate에만 적용되며
 production platform 선택, 실제 배포와 traffic 공개는 포함하지 않는다.
@@ -236,3 +236,83 @@ application rollback의 exact vendor CLI·account·application scope를 별도
 backup/PITR·alert 검증, vendor deploy/traffic/rollback 명령 확정과 실제 배포다.
 추가 API key·로그인·약관·결제, 고정 제품 결정 변경과 destructive migration이 필요해져도
 자동 진행하지 않고 별도 사람 승인에서 멈춘다.
+
+---
+
+# vNext local implementation candidate 완료 부록
+
+검증일은 2026-08-31(KST)다. 이 부록은 시작 기준선
+`bb0b28038243c539db2eafcfebc05144d9d59d66` 이후의 local vNext 구현과 합성 검증을 기록한다.
+production readiness, live historical data 검증, 배포 또는 traffic 공개 완료를 주장하지 않는다.
+
+## 1. 구현과 Git 경계
+
+- 제품·source 계약을 먼저 고정한 뒤 source adapter·typed historical model·collection review·
+  independent publication·public-read·frontend 순서의 선형 commit으로 통합했다. backend commit
+  뒤에 public-read와 frontend commit이 이어지며 merge commit이나 history rewrite는 없다.
+- 최근 공개본과 historical 공개본은 독립 active pointer, freshness와 fact-set을 유지한다.
+  recent detail은 historical mapping이나 publication 오류에도 계속 제공하고 확장 링크만 숨긴다.
+- catalog는 기간·방향·정렬과 최대 100개 bounded read를 제공하고, history는 선택 지역의 월별
+  기록, regions는 실제 공통 조사일의 지역 범위, markets는 해당 지역·조사일 시장 관측,
+  selection은 URL에 최대 5개 series를 담는 no-account 비교를 제공한다.
+- public GET은 canonical state만 허용하고 source API를 호출하지 않는다. current publication의
+  전체 shape를 먼저 검증하며 malformed hidden fact도 503으로 실패 폐쇄한다.
+- 검증 대상 application·운영 문서 exact SHA는
+  `cb0d4264ceee434fd66ff230cac0c29fe28308a2`, evidence commit은 `5119b3b`다. 이 부록을 포함하는
+  최종 local `main` SHA는 문서가 자기 commit을 참조할 수 없으므로 완료 응답에서 고정한다.
+- `origin`은 vNext 시작 기준선에 그대로 남아 있고 이번 구현은 push·deploy하지 않았다.
+
+## 2. schema·fixture·필수 gate
+
+- 빈 loopback PostgreSQL의 exact disposable DB
+  `grocery_vnext_verify_20260831_01`에 Django와 grocery migration `0001`~`0028`을 순방향 적용했다.
+- browser fixture는 production command가 아니라 DEBUG+QA, Admin off, loopback, 빈 DB와
+  `grocery_vnext_` 이름을 모두 확인한 뒤 정상 service로 review·seal·activate했다. 합성 범위는
+  recent series 5개, region 2개, market 31개, 36개월 monthly fact 360개다.
+- final `make check`는 Ruff format `218 files`, lint, mypy `201 source files`, migration drift,
+  Django system check와 pytest `859 passed`를 통과했다. synthetic production 설정의
+  `check --deploy --fail-level WARNING`도 warning 없이 통과했다.
+- 선행 gate 실패는 모두 실패로 처리하고 수정했다. 첫 통합 gate는 37개 format drift, 다음은
+  test fixture typing 44건, 그다음은 source schedule constraint와 catalog validation recovery
+  3건을 발견했다. schedule을 monthly `1..168`, 나머지 `1..24`로 DB에 고정하고 구체 오류 복구를
+  복원한 뒤 전체 gate를 통과했다.
+- 문서대로 fixture script를 직접 실행했을 때 repository root import가 누락된 결함도 발견해
+  수정했고, 그 뒤 같은 command로 fixture를 성공 생성했다. 테스트를 삭제하거나 약화하지 않았다.
+
+## 3. browser·접근성·짧은 load smoke
+
+[vNext browser evidence](../output/playwright/vnext/README.md)는 exact tested SHA, 도구 버전,
+8개 PNG와 JSON receipt의 SHA-256을 고정한다.
+
+- `390×844`와 `1440×900`에서 catalog → detail → history region GET → regions → markets →
+  detail → 2개 품목 selection GET 흐름을 통과했다. `360×800`, `768×1024`에서는 같은 6개
+  surface의 horizontal overflow가 없다.
+- 모든 검사 화면은 client script·inline event·외부 request 0, 44px target, 한 개의 main·h1,
+  keyboard 순서와 skip-link focus, no-store·no-referrer·CSP·fact-set header 분리를 통과했다.
+- 첫 render에서 390px 첫 record가 fold 아래에서 시작하고 detail `392/390`, regions `364/360`
+  overflow가 있음을 발견했다. secondary search disclosure, compact publication summary와 mobile
+  stacked facts로 수정했다. 최종 첫 record는 844px 안에 완전히 보인다.
+- local axe-core 4.13.0은 390px의 ready 6개 surface와 validation 400에서 WCAG A·AA violation 0,
+  unexpected incomplete 0이다. 자동 판정 불가인 aria-hidden 장식 기호와 SVG label만 수동검토로
+  분리했고 실제 palette 4.5:1 단위 gate로 보강했다. 불필요한 generic div의
+  `aria-labelledby`는 제거했다.
+- 최종 60초 recent read smoke는 600/600 성공, error·5xx 0, p95 74.8ms, 9.999rps와 single
+  revision을 확인했다. 첫 smoke는 active pointer가 아닌 과거 revision series를 잘못 선택해 detail
+  180건이 의도대로 503을 반환했으며, active channel의 exact series를 다시 읽어 재실행했다.
+- 이 60초 결과는 `SMOKE_NON_ACCEPTANCE`다. 역사적 Phase 0 900초 profile을 재실행하거나
+  history·regions·markets·selection의 capacity를 검증했다고 해석하지 않는다.
+
+## 4. 폐기·미접근·사람 checkpoint
+
+- 검증 후 임시 Gunicorn과 이 작업이 연 Playwright session만 종료했다. exact disposable DB
+  `grocery_vnext_verify_20260831_01`을 삭제했고 PostgreSQL catalog에서 잔존 개수 `0`을 확인했다.
+  기존 container·volume과 다른 database는 변경하지 않았다.
+- KAMIS source API, `.env.local`, API key의 값·길이·일부·encoding, production database,
+  production credential, platform, domain·DNS에는 접근하지 않았다. raw source data, credential,
+  검색어 또는 사용자 입력을 evidence artifact로 만들지 않았다.
+- 실제 full collection과 cross-source mapping 검수, production actor/IAM/MFA, `ko-v4` review·seal·
+  activation, migration과 traffic switch·rollback은 사람 checkpoint다.
+- repository health·freshness와 `inspect_recent_publication`, 기존 backup canonical 검증은
+  recent-only다. historical 전용 health/authoritative inspection, canonical backup restore와
+  이전 release의 migration `0028`·vNext route rollback 호환성을 별도로 증명하기 전에는
+  historical production traffic을 열지 않는다.


