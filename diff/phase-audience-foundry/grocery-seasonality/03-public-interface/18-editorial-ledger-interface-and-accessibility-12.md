## `test(evidence): refresh typography acceptance`

diff --git a/output/playwright/vnext-redesign-v2/1440x900-catalog.png b/output/playwright/vnext-redesign-v2/1440x900-catalog.png
index d288d2a..76560f0 100644
Binary files a/output/playwright/vnext-redesign-v2/1440x900-catalog.png and b/output/playwright/vnext-redesign-v2/1440x900-catalog.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-detail.png b/output/playwright/vnext-redesign-v2/1440x900-detail.png
index 5f1f872..b5d0093 100644
Binary files a/output/playwright/vnext-redesign-v2/1440x900-detail.png and b/output/playwright/vnext-redesign-v2/1440x900-detail.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-history.png b/output/playwright/vnext-redesign-v2/1440x900-history.png
index c3132dd..d721b1d 100644
Binary files a/output/playwright/vnext-redesign-v2/1440x900-history.png and b/output/playwright/vnext-redesign-v2/1440x900-history.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-selection.png b/output/playwright/vnext-redesign-v2/1440x900-selection.png
index ff4d560..d0ac37f 100644
Binary files a/output/playwright/vnext-redesign-v2/1440x900-selection.png and b/output/playwright/vnext-redesign-v2/1440x900-selection.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-catalog.png b/output/playwright/vnext-redesign-v2/390x844-catalog.png
index 29a7baf..41351f4 100644
Binary files a/output/playwright/vnext-redesign-v2/390x844-catalog.png and b/output/playwright/vnext-redesign-v2/390x844-catalog.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-detail.png b/output/playwright/vnext-redesign-v2/390x844-detail.png
index 4a5910d..99b906c 100644
Binary files a/output/playwright/vnext-redesign-v2/390x844-detail.png and b/output/playwright/vnext-redesign-v2/390x844-detail.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-history.png b/output/playwright/vnext-redesign-v2/390x844-history.png
index 8c549c4..b971c52 100644
Binary files a/output/playwright/vnext-redesign-v2/390x844-history.png and b/output/playwright/vnext-redesign-v2/390x844-history.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-selection.png b/output/playwright/vnext-redesign-v2/390x844-selection.png
index 59ea6fc..6b3b944 100644
Binary files a/output/playwright/vnext-redesign-v2/390x844-selection.png and b/output/playwright/vnext-redesign-v2/390x844-selection.png differ
diff --git a/output/playwright/vnext-redesign-v2/README.md b/output/playwright/vnext-redesign-v2/README.md
index 84c7bc0..058b5c5 100644
--- a/output/playwright/vnext-redesign-v2/README.md
+++ b/output/playwright/vnext-redesign-v2/README.md
@@ -1,7 +1,7 @@
 # Frontend redesign v2 local browser evidence
 
-생성 시각은 `2026-08-31T07:50:19Z`이며 검증 대상 frontend candidate는
-`d97888885e8a2e5b8db88005ddf0bf3a336dcdc6`다. 기존 Phase 0와 vNext evidence는 수정하지
+생성 시각은 `2026-08-31T08:12:03Z`이며 검증 대상 frontend candidate는
+`6a0118272675a92ef28db7a76f870646f0fbbb56`다. 기존 Phase 0와 vNext evidence는 수정하지
 않았다.
 
 이 matrix의 ready 화면은 synthetic fixture가 아니다. 승인된 로컬 실행에서 KAMIS 공공 API의
@@ -51,26 +51,29 @@ fixture의 mapping과 approval은 browser acceptance용으로 자동 생성한 t
   source·series·응답 용어를 노출하지 않는다.
 - Brand/UI: warm paper, forest/harvest/data-blue와 굵은 장부선을 사용하되 photo, gradient,
   glass, dashboard card grid를 사용하지 않는다. 기능 제목은 system sans, 브랜드와 큰 editorial
-  제목만 Gowun Batang이다.
+  제목은 Hahmlet Bold, `초록장부` wordmark는 나눔손글씨 야채장수 백금례 outline이다.
 - Engineering: Django SSR, no-JS, active-publication-only read, bounded query, template 산술 금지,
   fail-closed validation과 보안 header를 유지한다.
 
 첫 실제 render 리뷰에서 dark masthead focus/hover 대비, historical 503의 빈 제목과 잘못된 복구
 label, detail의 market 탐색 단서, desktop catalog 열 배치, SVG 끝 월 label clipping을 수정했다.
 axe가 찾은 generic `div`의 unsupported `aria-label`도 제거한 뒤 전체 matrix를 다시 생성했다.
+후속 typography review에서는 둥근 Gowun Batang을 Hahmlet Bold로 교체하고, 네이버 공식
+나눔손글씨 야채장수 백금례의 `초록장부` outline만 wordmark SVG로 사용했다. font·license
+provenance와 11KB local SVG를 고정한 뒤 같은 browser·axe matrix를 다시 통과했다.
 
 ## 파일 무결성
 
 | 파일 | SHA-256 |
 |---|---|
-| `390x844-catalog.png` | `193f09f97826858b2ce6da28768805b73ccbdb9501160b3c4e615c9e4b9a339b` |
-| `390x844-detail.png` | `ffce845e16d3ebbb5727ed4b16f0c2f4194612c7b30a8e2c7a90a1138cf6712d` |
-| `390x844-history.png` | `8eb3d0be2d66b73c00e10897fedfb229c528f94566737cc36cf3069f34555dec` |
-| `390x844-selection.png` | `f820f2a2a22b20fe20756d282c4adc6e65bb826871f972747d4b0f3e249eb55e` |
-| `1440x900-catalog.png` | `b0143c982d439fdb6aec923bc52761de7b8ba5d87910bb4b29e9fc827eefe1d0` |
-| `1440x900-detail.png` | `8f876daaa966000e0fb083f5c0446893184f95beb182c03409a7209cd953f2d4` |
-| `1440x900-history.png` | `53cab0b995f030355add993cdfd01a08f6c77a85edc8b7708fa4fccdc8c740ea` |
-| `1440x900-selection.png` | `f572b203dd0b18ac9373023deba61bb0359753243c01566292a7309cfc1c12bf` |
+| `390x844-catalog.png` | `778a9820da4eefaba3d792d61271fd624278b053ce9252b6cf938683a0f07c3e` |
+| `390x844-detail.png` | `65e2dd8ce241468f001ac59626d3a2acc5843048d808dce7f7732b039813e584` |
+| `390x844-history.png` | `18d698d4ed1b92359d435f4e95ef5fc544a10924ee59b342e409cd6e10f5516b` |
+| `390x844-selection.png` | `c1841dc220adeb20d9cc6072b0315a12159a32f020633045f12c170e158325b2` |
+| `1440x900-catalog.png` | `328d70224d5dcb153f18237760c05569adc8943a60ff8701afc14804fd39a5bb` |
+| `1440x900-detail.png` | `876a9a4932db55277ee0fdf30d8988dcda98529311304fed7c2042245b27d702` |
+| `1440x900-history.png` | `8a54376df6f53658d47ecfdc231ce82d7a0610d2cd72a66475ae796ad8df4b98` |
+| `1440x900-selection.png` | `4bb04b3c028b6a6baffaf6e6f592196339ca502060f2026c0bf9c7afd59c2855` |
 | `browser-results.json` | `e9de4eba8ef4690731536ce284adabc392598c9533ac5ebb51d5402ef878c0f1` |
 | `axe-results.json` | `c8be5cc052b0a3c6ce544c279bf2ca9882118df1b6aab604396e31c77763a7e7` |
 | `live-source-results.json` | `8ec68cea621e6a77cf6dd588093f67320f0bc604b4d3b7029e2140831ff4aa21` |


## `docs: record final typography acceptance`

diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
index 13dc0c5..91cae7a 100644
--- a/docs/COMPLETION-REPORT.md
+++ b/docs/COMPLETION-REPORT.md
@@ -336,9 +336,9 @@ scheduler·traffic 검증을 대신하지 않는다.
 ## 6. Frontend redesign v2 실제 자료 browser evidence
 
 Frontend redesign v2는 `27bb0cc3e9c65309c567fb6b4e08ad8b989907a6`에서 시작했다. browser
-evidence 대상 frontend commit은 `d97888885e8a2e5b8db88005ddf0bf3a336dcdc6`, evidence commit은
-`db906b4`다. 이 절을 포함한 최종 local `main` SHA는 문서가 자기 commit을 참조할 수 없으므로
-완료 응답에서 고정한다.
+evidence 대상 frontend commit은 `6a0118272675a92ef28db7a76f870646f0fbbb56`, evidence commit은
+`81e05296cd74da9dd6a4ec2b2bc4ecac8edb073d`다. 이 절을 포함한 최종 local `main` SHA는 문서가
+자기 commit을 참조할 수 없으므로 완료 응답에서 고정한다.
 
 명시적 opt-in 실행에서 실제 네 KAMIS 공공 API 응답을 normal adapter와 typed persistence에
 통과시키고, disposable PostgreSQL에서 test-only mapping·review·seal·activation으로 local active
@@ -352,6 +352,9 @@ raw response 보존 없음으로 통과했다. browser ready 화면의 가격값
 `360×800`과 `768×1024`의 overflow 0, mobile 첫 catalog record의 fold 안 노출, client script와
 외부 request 0을 고정한다. local axe-core 4.13.0은 ready 6면, validation 400, catalog 503,
 generic 404의 9면에서 WCAG 2/2.1/2.2 A·AA violation 0과 unexpected incomplete 0을 확인했다.
+대형 제목은 self-hosted Hahmlet Bold를 사용하고, `초록장부` wordmark는 Clova Nanum Handwriting
+`야채장수 백금례`의 글자 윤곽만 담은 local SVG다. 원 TTF는 runtime에 포함하지 않으며 외부 font
+요청도 없다.
 
 따라서 application code, local 실제 자료 흐름과 browser 인수 범위에서는 배포 직전 candidate다.
 그러나 자동 mapping·approval과 representative historical scope는 첫 catalog 품목과 한 지역에
