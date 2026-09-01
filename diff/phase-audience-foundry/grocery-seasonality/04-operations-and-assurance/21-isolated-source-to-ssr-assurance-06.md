## `docs(ops): document live source smoke boundary`

diff --git a/README.md b/README.md
index 2f97301..0985055 100644
--- a/README.md
+++ b/README.md
@@ -18,13 +18,13 @@ publication이 있을 때 선택 지역의 월별 기록, 지역별 조사 범
 - 원격 저장소: `origin`
   (`https://github.com/seungwoo7050/audience-foundry-grocery-seasonality.git`)
 - 공개 상태: GitHub 공개 저장소·production service 미배포
-- 구현 상태: **vNext local implementation candidate**; production 미배포·미활성
+- 구현 상태: **vNext implementation candidate**; production 미배포·미활성
 - 시작 기준선: `bb0b28038243c539db2eafcfebc05144d9d59d66`
 - source path: 최근 비교 `15156063`; historical `15156060`, `15156062`, `15156065`
 
-vNext 변경은 시작 기준선 이후 local에서만 만들었고 이 문서 갱신 시점에는 `origin`으로
-push하거나 production에 배포하지 않았습니다. exact 최종 local SHA와 검증 상태는 완료
-보고서에서 고정합니다.
+vNext 변경은 시작 기준선 이후 검토 가능한 선형 commit으로 만들었습니다. exact 최종 local·
+승인된 remote SHA와 검증 상태는 완료 응답에서 고정하며, source repository push를 production
+배포나 자료 공개 활성화로 해석하지 않습니다.
 
 ## 역사적 첫 범위
 
@@ -100,6 +100,28 @@ rollback은 reverse migration 없이 최신 schema를 유지하고 검증된 이
 되돌립니다. 실제 artifact packaging, bundled notice, vendor deploy·traffic·rollback CLI는
 platform을 선택한 뒤 사람이 별도로 승인하는 production checkpoint입니다.
 
+### 명시적 live source-to-SSR smoke
+
+소유자가 발급받은 local 개발 key로 네 공식 API부터 typed persistence와 공개 SSR까지의 최소
+연결을 다시 확인할 때만 다음 명령을 명시적으로 실행합니다.
+
+```sh
+make db-up
+make live-source-e2e-smoke
+```
+
+이 target은 ambient `KAMIS_API_KEY`를 거부하고 ignored owner-only `.env.local`을 process 안에서만
+읽습니다. 고정 loopback PostgreSQL의 exact 전용 DB를 새로 만든 뒤 최근·월별·지역별·시장별 API를
+각각 bounded 호출하고, test-only 자동 mapping·review·seal·activation을 거쳐 catalog·detail·history·
+regions·markets SSR을 확인합니다. 공개 SSR 중 source 재호출이 있으면 실패합니다. 모든 write는
+outer transaction에서 rollback하고 target이 새로 만든 DB만 삭제하며, 출력은 값·query·원응답이
+없는 고정 count receipt입니다.
+
+실제 provider 호출이므로 이 target은 `make check`와 `make production-check`에 포함하지 않습니다.
+성공은 현재 key·schema·최소 연결의 smoke evidence일 뿐 사람의 mapping·권리·전체 coverage 검수,
+production publication 승인이나 scheduler 검증을 대신하지 않습니다. 실패 역시 숨기지 않고 live
+evidence로 취급합니다. 세부 안전 경계는 [운영 런북](docs/OPERATIONS-RUNBOOK.md)에 있습니다.
+
 ## 계약·증거 문서
 
 - [도메인 개요](docs/DOMAIN-BRIEF.md)
diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
index 840fe12..0b2b29f 100644
--- a/docs/COMPLETION-REPORT.md
+++ b/docs/COMPLETION-REPORT.md
@@ -260,7 +260,8 @@ production readiness, live historical data 검증, 배포 또는 traffic 공개
 - 검증 대상 application·운영 문서 exact SHA는
   `cb0d4264ceee434fd66ff230cac0c29fe28308a2`, evidence commit은 `5119b3b`다. 이 부록을 포함하는
   최종 local `main` SHA는 문서가 자기 commit을 참조할 수 없으므로 완료 응답에서 고정한다.
-- `origin`은 vNext 시작 기준선에 그대로 남아 있고 이번 구현은 push·deploy하지 않았다.
+- 이 부록 최초 작성 시점에는 `origin`이 vNext 시작 기준선에 남아 있었다. 이후 source-to-SSR
+  follow-up과 승인된 remote 고정 결과는 아래 추가 증거와 세션 완료 응답에서 구분해 기록한다.
 
 ## 2. schema·fixture·필수 gate
 
@@ -316,3 +317,18 @@ production readiness, live historical data 검증, 배포 또는 traffic 공개
   recent-only다. historical 전용 health/authoritative inspection, canonical backup restore와
   이전 release의 migration `0028`·vNext route rollback 호환성을 별도로 증명하기 전에는
   historical production traffic을 열지 않는다.
+
+## 5. 실제 source-to-SSR follow-up evidence
+
+2026-08-31(KST), test commit `0eb9d62`의 명시적 opt-in smoke를 exact disposable loopback
+PostgreSQL에서 실제 실행했다. 네 공식 API를 normal adapter로 호출해 최근 10행, 월별 36행,
+지역별 1행, 시장별 9행을 typed model에 통과시킨 뒤 test-only mapping·review·seal·activation으로
+catalog·detail·history·regions·markets 5개 SSR route를 검증했다. SSR 처리 중 source call은 0이고
+각 화면에 저장된 제공값이 존재함을 확인했다.
+
+모든 write는 outer transaction으로 rollback했고 root table이 다시 비었는지 확인한 뒤 exact 전용
+DB를 삭제했다. source row·response body·URL query·credential·사용자 입력은 log, fixture, receipt나
+artifact에 남기지 않았고 고정 count receipt만 출력했다. 이 결과는 key·provider schema·adapter·
+persistence·SSR 연결의 실제 smoke evidence다. 동적으로 파생한 test mapping과 자동 approval은
+사람의 cross-source identity·권리·전체 coverage 검수, production publication·activation,
+scheduler·traffic 검증을 대신하지 않는다.
diff --git a/docs/OPERATIONS-RUNBOOK.md b/docs/OPERATIONS-RUNBOOK.md
index 1aa2098..7915f83 100644
--- a/docs/OPERATIONS-RUNBOOK.md
+++ b/docs/OPERATIONS-RUNBOOK.md
@@ -449,6 +449,45 @@ historical traffic 공개를 중단한다.
 production scheduler 성공을 뜻하지 않는다. 경계 전에 새 실패 attempt가 있거나 경계를 넘으면
 freshness contract에 따라 다시 확인하고 alert를 판정한다.
 
+## 명시적 local live source-to-SSR smoke
+
+개발 key가 실제 네 source adapter, typed persistence와 공개 SSR까지 이어지는지만 다시 확인할 때
+repository root에서 다음 opt-in target을 실행한다.
+
+```sh
+make db-up
+make live-source-e2e-smoke
+```
+
+target은 다음 조건을 모두 실패 폐쇄한다.
+
+- ambient `KAMIS_API_KEY` 부재와 `LIVE_SOURCE_E2E_SMOKE=1`
+- `DEBUG=True`, Admin·QA preview·production control plane 비활성
+- `127.0.0.1:55434`의 exact `grocery_vnext_live_api_smoke` PostgreSQL database
+- 실행 전 publication·source·audit root table이 모두 비어 있음
+
+Make recipe가 exact DB를 새로 만들지 못하면 기존 DB를 사용하거나 삭제하지 않는다. guard를 통과한
+뒤 ignored owner-only `.env.local`은 loader가 process 안에서만 읽고 값·길이·일부·encoding을
+출력하지 않는다. 최근, 월별, 지역별, 시장별 source를 각각 bounded 호출해 normal parser와 typed
+model로 저장한다. live response에서 series·region을 동적으로 고르고 local test-only mapping과
+review·seal·activation을 만든 뒤 catalog·detail·history·regions·markets를 읽는다. SSR 구간에서는
+source client를 호출하면 즉시 실패하도록 막는다.
+
+전체 data flow는 outer transaction 안에서 실행하고 성공·실패와 무관하게 rollback한다. 성공 뒤
+root table이 다시 비었는지 확인하고 recipe가 만든 exact DB만 trap에서 삭제한다. stdout은 status와
+row·route count, SSR source-call count, raw-response-retention 여부로 제한한 고정 receipt다. URL,
+query, 사용자 입력, source row, response body, credential 또는 traceback을 artifact로 남기지 않는다.
+
+2026-08-31(KST) 실제 실행은 최근 10행, 월별 36행, 지역별 1행, 시장별 9행과 공개 SSR 5 route를
+통과했고 SSR source call은 0이었다. 종료 후 root data 부재와 전용 DB 삭제를 별도로 확인했다.
+provider 응답은 바뀔 수 있으므로 이 count는 fixture 계약이나 장기 coverage 보장이 아니다.
+
+이 smoke의 자동 mapping·review·seal·activation은 disposable test database에만 존재하며 사람의
+cross-source identity·mapping·rights·coverage 검수, production actor/MFA, scheduler, 첫 production
+publication·activation을 대신하지 않는다. 따라서 일반 `make check`, `make production-check`, CI,
+배포 또는 production worker에 자동 연결하지 않는다. 실패는 key·provider·schema·adapter 경계의
+live evidence로 기록하고 fixture 성공으로 덮지 않는다.
+
 ## 수집부터 공개까지
 
 ### 역할 분리와 자동화 한계
