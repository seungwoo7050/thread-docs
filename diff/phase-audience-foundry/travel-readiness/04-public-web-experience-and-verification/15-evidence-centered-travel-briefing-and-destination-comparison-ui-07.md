## `feat(frontend): integrate entry and warning briefs`

diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index dac416b..1a5a1ff 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -49,8 +49,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        15_265,
-        "e041050e5ccba01e218582072ae29c687e5cc2ad5432861e77870ffc45cec34a",
+        17_870,
+        "30310db2f9c4070fc754972c8196ace469a7dc5ae2a65e0dcc04d274739ddd46",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 18753fb..07bebf9 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -146,7 +146,9 @@ body {
 
 h1,
 h2,
-h3 {
+h3,
+h4,
+h5 {
   line-height: 1.22;
   text-wrap: balance;
 }
@@ -729,6 +731,130 @@ h3 {
   font-size: 0.8125rem;
 }
 
+.travel-briefs {
+  margin-top: var(--space-12);
+  padding-top: var(--space-6);
+  border-top: 0.0625rem solid var(--line-strong);
+}
+
+.travel-briefs-intro {
+  max-width: 40rem;
+  margin: 0;
+  color: var(--muted-ink);
+  font-size: 0.9rem;
+}
+
+.publication-brief {
+  min-width: 0;
+  margin-top: var(--space-6);
+  padding: var(--space-4) 0 var(--space-4) var(--space-6);
+  border-left: 0.3rem solid var(--current);
+}
+
+.publication-brief + .publication-brief {
+  padding-top: var(--space-6);
+  border-top: 0.0625rem solid var(--line);
+}
+
+.publication-brief[data-state="empty"] {
+  border-left-color: var(--muted-ink);
+}
+
+.publication-brief[data-state="unavailable"],
+.publication-brief[data-state="stale"] {
+  border-left-color: var(--stale);
+}
+
+.publication-brief[data-state="server-error"] {
+  border-left-color: var(--error);
+}
+
+.publication-brief-header {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: var(--space-2) var(--space-6);
+  align-items: baseline;
+  justify-content: space-between;
+}
+
+.publication-brief h5 {
+  margin: 0;
+  font-size: 1.2rem;
+}
+
+.publication-status,
+.publication-message {
+  margin: 0;
+}
+
+.publication-status {
+  font-size: 0.875rem;
+}
+
+.publication-message {
+  margin-top: var(--space-2);
+  color: var(--muted-ink);
+}
+
+.brief-facts,
+.source-history dl {
+  min-width: 0;
+  margin: var(--space-4) 0 0;
+  border-top: 0.0625rem solid var(--line);
+}
+
+.brief-facts > div,
+.source-history dl > div {
+  display: grid;
+  min-width: 0;
+  padding-block: var(--space-2);
+  border-bottom: 0.0625rem solid var(--line);
+  grid-template-columns: minmax(9rem, 0.7fr) minmax(0, 1.3fr);
+}
+
+.brief-facts dt,
+.source-history dt {
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
+  font-weight: 800;
+}
+
+.brief-facts dd,
+.source-history dd {
+  min-width: 0;
+  margin: 0;
+}
+
+.source-history {
+  margin-top: var(--space-3);
+}
+
+.source-history summary {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  color: var(--link);
+  font-weight: 800;
+  cursor: pointer;
+}
+
+.verification-note {
+  margin: var(--space-4) 0 0;
+  padding: var(--space-3) var(--space-4);
+  background: var(--surface-muted);
+  border-left: 0.25rem solid var(--stale);
+}
+
+.source-link {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  margin-top: var(--space-2);
+  color: var(--link);
+  font-weight: 800;
+}
+
 .site-footer {
   padding-block: var(--space-6);
   color: white;
@@ -825,6 +951,15 @@ h3 {
   .alternative-schedule div {
     grid-template-columns: minmax(0, 1fr);
   }
+
+  .publication-brief {
+    padding-left: var(--space-4);
+  }
+
+  .brief-facts > div,
+  .source-history dl > div {
+    grid-template-columns: minmax(0, 1fr);
+  }
 }
 
 @media (pointer: coarse) {
@@ -832,7 +967,9 @@ h3 {
   .primary-button,
   .error-summary a,
   .flight-source a,
-  .alternatives summary {
+  .alternatives summary,
+  .source-history summary,
+  .source-link {
     min-height: 48px;
   }
 }
@@ -863,7 +1000,10 @@ h3 {
   .itinerary-legs,
   .flight-leg,
   .calculation-note,
-  .alternatives {
+  .alternatives,
+  .travel-briefs,
+  .publication-brief,
+  .verification-note {
     border-color: CanvasText;
   }
 
diff --git a/public_web/templates/public_web/partials/entry_card.html b/public_web/templates/public_web/partials/entry_card.html
index ccfd893..dfb2167 100644
--- a/public_web/templates/public_web/partials/entry_card.html
+++ b/public_web/templates/public_web/partials/entry_card.html
@@ -1,32 +1,28 @@
-<article id="entry-card" data-state="{{ entry_card.state }}" class="publication-card"
-         tabindex="0" aria-labelledby="entry-heading"
-         aria-describedby="entry-status entry-message">
-  <h2 id="entry-heading">{{ entry_card.heading }}</h2>
-  <p class="status-line" id="entry-status" role="status">
-    <strong>상태: {{ entry_card.status_label }}</strong>
-  </p>
-  <p id="entry-message">{{ entry_card.message }}</p>
+<article id="{% if opportunity %}destination-{{ opportunity.rank }}-entry{% else %}entry-card{% endif %}"
+         data-module="entry" data-state="{{ entry_card.state }}" class="publication-brief"
+         aria-labelledby="{% if opportunity %}destination-{{ opportunity.rank }}-entry-heading{% else %}entry-heading{% endif %}">
+  <header class="publication-brief-header">
+    <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-entry-heading{% else %}entry-heading{% endif %}">{{ entry_card.heading|default:"입국요건" }}</h5>
+    <p class="publication-status" role="status"><strong>상태: {{ entry_card.status_label }}</strong></p>
+  </header>
+  <p class="publication-message">{{ entry_card.message }}</p>
   {% if entry_card.has_publication %}
-    <section class="publication-section">
-      <h3>출처에서 확인된 사실</h3>
-      <dl class="fact-list">
-        <dt>대상 국가</dt><dd>{{ entry_card.country_name }}</dd>
-        <dt>일반여권 관련 출처 표기</dt><dd>{{ entry_card.period_text }}</dd>
-        <dt>출처 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
-        <dt>출처 자료 날짜</dt><dd>{{ entry_card.snapshot_date|default:"출처가 제공하지 않음" }}</dd>
+    <dl class="brief-facts">
+      <div><dt>일반여권 관련 출처 표기</dt><dd>{{ entry_card.period_text }}</dd></div>
+      <div><dt>출처 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd></div>
+      <div><dt>출처 자료 날짜</dt><dd>{{ entry_card.snapshot_date|default:"출처가 제공하지 않음" }}</dd></div>
+    </dl>
+    <details class="source-history">
+      <summary>출처와 확인 이력 보기</summary>
+      <dl>
+        <div><dt>대상 국가</dt><dd>{{ entry_card.country_name }}</dd></div>
+        <div><dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd></div>
+        <div><dt>게시 자료</dt><dd>generation {{ entry_card.generation }} · {{ entry_card.published_at }}</dd></div>
+        <div><dt>출처 리비전</dt><dd>{{ entry_card.source_revision }}</dd></div>
+        <div><dt>출처</dt><dd>{{ entry_card.source_owner }} · {{ entry_card.attribution }}</dd></div>
       </dl>
-    </section>
-    <section class="publication-section">
-      <h3>출처와 게시 이력</h3>
-      <dl class="fact-list">
-        <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
-        <dt>게시 리비전</dt><dd>generation {{ entry_card.generation }}</dd>
-        <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
-        <dt>출처 리비전</dt><dd>{{ entry_card.source_revision }}</dd>
-        <dt>출처</dt><dd>{{ entry_card.source_owner }} · {{ entry_card.attribution }}</dd>
-      </dl>
-    </section>
-    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜에 따른 적용성과 최신 조건은 공식 출처에서 다시 확인해 주세요.</p>
+    </details>
+    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜에 따른 적용성과 최신 조건</p>
     <a class="source-link" href="{{ entry_card.source_locator }}"
        rel="noopener noreferrer">입국요건 공식 출처 열기</a>
   {% endif %}
diff --git a/public_web/templates/public_web/partials/opportunity.html b/public_web/templates/public_web/partials/opportunity.html
index 0946460..e93049f 100644
--- a/public_web/templates/public_web/partials/opportunity.html
+++ b/public_web/templates/public_web/partials/opportunity.html
@@ -82,5 +82,13 @@
     <p>공식 운항 일정 {{ opportunity.calculation_basis.schedule_source_date|default:"날짜 없음" }} · 비행시간 참고자료 {{ opportunity.calculation_basis.duration_reference_date|default:"날짜 없음" }}</p>
   </aside>
 
-  <div class="travel-briefs" data-travel-briefs></div>
+  <section class="travel-briefs" aria-labelledby="destination-{{ opportunity.rank }}-briefs">
+    <h4 id="destination-{{ opportunity.rank }}-briefs">여행 전 확인</h4>
+    <p class="travel-briefs-intro">
+      아래 정보는 추천 계산과 별도로 검수·게시됩니다. 실제 입국 가능 여부와
+      여행일 적용성은 공식 기관에서 다시 확인해 주세요.
+    </p>
+    {% include "public_web/partials/entry_card.html" with entry_card=opportunity.entry_card %}
+    {% include "public_web/partials/warning_card.html" with warning_card=opportunity.warning_card %}
+  </section>
 </article>
diff --git a/public_web/templates/public_web/partials/warning_card.html b/public_web/templates/public_web/partials/warning_card.html
index 824d4b7..a08e7ac 100644
--- a/public_web/templates/public_web/partials/warning_card.html
+++ b/public_web/templates/public_web/partials/warning_card.html
@@ -1,33 +1,29 @@
-<article id="warning-card" data-state="{{ warning_card.state }}" class="publication-card"
-         tabindex="0" aria-labelledby="warning-heading"
-         aria-describedby="warning-status warning-message">
-  <h2 id="warning-heading">{{ warning_card.heading }}</h2>
-  <p class="status-line" id="warning-status" role="status">
-    <strong>상태: {{ warning_card.status_label }}</strong>
-  </p>
-  <p id="warning-message">{{ warning_card.message }}</p>
+<article id="{% if opportunity %}destination-{{ opportunity.rank }}-warning{% else %}warning-card{% endif %}"
+         data-module="warning" data-state="{{ warning_card.state }}" class="publication-brief"
+         aria-labelledby="{% if opportunity %}destination-{{ opportunity.rank }}-warning-heading{% else %}warning-heading{% endif %}">
+  <header class="publication-brief-header">
+    <h5 id="{% if opportunity %}destination-{{ opportunity.rank }}-warning-heading{% else %}warning-heading{% endif %}">{{ warning_card.heading|default:"여행경보" }}</h5>
+    <p class="publication-status" role="status"><strong>상태: {{ warning_card.status_label }}</strong></p>
+  </header>
+  <p class="publication-message">{{ warning_card.message }}</p>
   {% if warning_card.has_publication %}
-    <section class="publication-section">
-      <h3>출처에서 확인된 사실</h3>
-      <dl class="fact-list">
-        <dt>대상 국가</dt><dd>{{ warning_card.country_name }}</dd>
-        <dt>출처 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
-        <dt>출처 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
-        <dt>출처 범위</dt><dd>{{ warning_card.scope_text }}</dd>
-        <dt>출처 작성일</dt><dd>{{ warning_card.written_date|default:"출처가 제공하지 않음" }}</dd>
+    <dl class="brief-facts">
+      <div><dt>출처 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd></div>
+      <div><dt>출처 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd></div>
+      <div><dt>출처 범위</dt><dd>{{ warning_card.scope_text }}</dd></div>
+      <div><dt>출처 작성일</dt><dd>{{ warning_card.written_date|default:"출처가 제공하지 않음" }}</dd></div>
+    </dl>
+    <details class="source-history">
+      <summary>출처와 확인 이력 보기</summary>
+      <dl>
+        <div><dt>대상 국가</dt><dd>{{ warning_card.country_name }}</dd></div>
+        <div><dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd></div>
+        <div><dt>게시 자료</dt><dd>generation {{ warning_card.generation }} · {{ warning_card.published_at }}</dd></div>
+        <div><dt>출처 리비전</dt><dd>{{ warning_card.source_revision }}</dd></div>
+        <div><dt>출처</dt><dd>{{ warning_card.source_owner }} · {{ warning_card.attribution }}</dd></div>
       </dl>
-    </section>
-    <section class="publication-section">
-      <h3>출처와 게시 이력</h3>
-      <dl class="fact-list">
-        <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
-        <dt>게시 리비전</dt><dd>generation {{ warning_card.generation }}</dd>
-        <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
-        <dt>출처 리비전</dt><dd>{{ warning_card.source_revision }}</dd>
-        <dt>출처</dt><dd>{{ warning_card.source_owner }} · {{ warning_card.attribution }}</dd>
-      </dl>
-    </section>
-    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 공식 출처에서 다시 확인해 주세요.</p>
+    </details>
+    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성</p>
     <a class="source-link" href="{{ warning_card.source_locator }}"
        rel="noopener noreferrer">여행경보 공식 출처 열기</a>
   {% endif %}


