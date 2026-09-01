## `feat(frontend): redesign travel window search`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 1a5a8b5..5309e1e 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -185,8 +185,8 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "25466db727b077fa07cd185a440db4686c4589c35763b52ef36a849f626e8e34"
-SITE_CSS_BYTES: Final = 18_909
+SITE_CSS_SHA256: Final = "ca317ac24714a9404cdaeb0a37a301ebb3781304c3aa210ba4adbe80d288f893"
+SITE_CSS_BYTES: Final = 21_361
 SITE_JS_SHA256: Final = "1a5a95b286d6e12f1765351d5a0b1528c833fb46af16d747b469b61323cc7f44"
 SITE_JS_BYTES: Final = 1_628
 MARU_BURI_SHA256: Final = "5c8b39b683595d0ddcf2554148f4d2fb14c55cb967e5bff2e282b2936034fc75"
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 49da61a..1238ea5 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -57,8 +57,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        18_909,
-        "25466db727b077fa07cd185a440db4686c4589c35763b52ef36a849f626e8e34",
+        21_361,
+        "ca317ac24714a9404cdaeb0a37a301ebb3781304c3aa210ba4adbe80d288f893",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 2b1e447..324eacf 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -125,7 +125,7 @@ body {
 .site-header {
   color: white;
   background: var(--header);
-  border-bottom: 0.0625rem solid rgb(255 255 255 / 32%);
+  border-bottom: 0.25rem solid #2C86C2;
 }
 
 .site-header-inner {
@@ -141,20 +141,29 @@ body {
   min-height: 44px;
   align-items: center;
   color: white;
-  font-size: 1.125rem;
-  font-weight: 850;
+  font-size: 1.1875rem;
+  font-weight: 700;
   letter-spacing: -0.035em;
   text-decoration: none;
 }
 
 .site-scope {
   margin: 0;
-  color: #DCE9E3;
+  color: #DCEAF4;
   font-size: 0.875rem;
   font-weight: 700;
   letter-spacing: 0.03em;
 }
 
+.site-scope span {
+  display: inline-block;
+  margin-right: var(--space-2);
+  padding-right: var(--space-2);
+  color: white;
+  border-right: 0.0625rem solid rgb(255 255 255 / 45%);
+  letter-spacing: 0.1em;
+}
+
 .page-main {
   padding-block: clamp(var(--space-8), 7vw, var(--space-16));
 }
@@ -173,6 +182,14 @@ body {
   letter-spacing: 0.08em;
 }
 
+.eyebrow-code {
+  display: inline-block;
+  margin-right: var(--space-2);
+  padding-right: var(--space-2);
+  color: var(--header);
+  border-right: 0.0625rem solid var(--line-strong);
+}
+
 h1,
 h2,
 h3,
@@ -184,8 +201,8 @@ h5 {
 
 h1 {
   margin: 0;
-  font-size: clamp(2.15rem, 6vw, 4.4rem);
-  letter-spacing: -0.055em;
+  font-size: clamp(2.5rem, 6vw, 5.15rem);
+  letter-spacing: -0.05em;
 }
 
 h2 {
@@ -210,7 +227,7 @@ h3 {
 .search-section {
   display: grid;
   min-width: 0;
-  gap: var(--space-12) var(--space-16);
+  gap: var(--space-12) clamp(var(--space-8), 5vw, var(--space-16));
   align-items: start;
   grid-template-columns: minmax(0, 5fr) minmax(21rem, 7fr);
 }
@@ -221,7 +238,7 @@ h3 {
 }
 
 .search-intro h1 {
-  max-width: 10.5ch;
+  max-width: 9.5ch;
 }
 
 .service-boundary {
@@ -230,15 +247,28 @@ h3 {
   border-top: 0.0625rem solid var(--line-strong);
 }
 
-.service-boundary p {
-  margin: 0;
+.service-boundary > div {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2) var(--space-4);
   padding-block: var(--space-4);
-  color: var(--muted-ink);
   border-bottom: 0.0625rem solid var(--line);
+  grid-template-columns: minmax(7rem, 0.7fr) minmax(0, 1.6fr);
+}
+
+.service-boundary dt,
+.service-boundary dd {
+  margin: 0;
 }
 
-.service-boundary strong {
+.service-boundary dt {
   color: var(--ink);
+  font-size: 0.8125rem;
+  font-weight: 700;
+}
+
+.service-boundary dd {
+  color: var(--muted-ink);
 }
 
 .search-panel {
@@ -283,9 +313,9 @@ h3 {
 
 .trip-form {
   min-width: 0;
-  padding: clamp(var(--space-6), 5vw, var(--space-8));
+  padding: clamp(var(--space-6), 4vw, var(--space-8));
   background: var(--surface);
-  border-top: 0.3rem solid var(--header);
+  border-top: 0.25rem solid var(--header);
   border-bottom: 0.0625rem solid var(--line-strong);
 }
 
@@ -296,28 +326,63 @@ h3 {
   border: 0;
 }
 
+.form-kicker {
+  display: block;
+  margin: 0 0 var(--space-1);
+  color: var(--link);
+  font-family: "NanumSquare Neo", system-ui, sans-serif;
+  font-size: 0.75rem;
+  font-weight: 700;
+  letter-spacing: 0.08em;
+}
+
 .trip-form legend {
-  margin-bottom: var(--space-2);
   padding: 0;
-  font-size: 1.4rem;
-  font-weight: 850;
-  letter-spacing: -0.025em;
+  font-family: "Maru Buri", "Batang", serif;
+  font-size: clamp(1.35rem, 3vw, 1.75rem);
+  font-weight: 600;
+  letter-spacing: -0.03em;
+  line-height: 1.35;
 }
 
 .form-intro {
-  margin: 0 0 var(--space-8);
+  margin: var(--space-2) 0 var(--space-8);
   color: var(--muted-ink);
   font-size: 0.9375rem;
 }
 
+.form-intro strong {
+  margin-right: var(--space-2);
+  color: var(--ink);
+  font-size: 0.75rem;
+  letter-spacing: 0.05em;
+}
+
+.journey-window {
+  min-width: 0;
+  padding-left: var(--space-6);
+  border-left: 0.125rem solid var(--current);
+}
+
 .form-field {
   min-width: 0;
 }
 
 .form-field label {
-  display: block;
+  display: flex;
+  min-width: 0;
+  gap: var(--space-3);
+  align-items: baseline;
+  justify-content: space-between;
   margin-bottom: var(--space-2);
-  font-weight: 850;
+  font-weight: 700;
+}
+
+.field-code {
+  flex: 0 0 auto;
+  color: var(--muted-ink);
+  font-size: 0.6875rem;
+  letter-spacing: 0.08em;
 }
 
 .form-field :where(input, select) {
@@ -327,8 +392,10 @@ h3 {
   min-height: 52px;
   padding: 0.75rem 0.875rem;
   color: var(--ink);
-  font: inherit;
+  font-family: "NanumSquare Neo", system-ui, sans-serif;
+  font-variant-numeric: tabular-nums;
   font-size: 1rem;
+  font-weight: 700;
   background: white;
   border: 0.125rem solid var(--line-strong);
   border-radius: var(--control-radius);
@@ -355,29 +422,66 @@ h3 {
 }
 
 .window-connector {
-  height: var(--space-8);
-  margin-left: var(--space-6);
-  border-left: 0.125rem solid var(--line);
+  display: grid;
+  height: var(--space-12);
+  min-width: 0;
+  gap: var(--space-2);
+  align-items: center;
+  grid-template-columns: minmax(var(--space-6), 1fr) auto auto;
+}
+
+.connector-line {
+  height: 0.0625rem;
+  background: var(--line);
+}
+
+.connector-copy,
+.connector-arrow {
+  color: var(--muted-ink);
+  font-size: 0.6875rem;
+  font-weight: 700;
+  letter-spacing: 0.05em;
+}
+
+.connector-arrow {
+  color: var(--current);
+  font-size: 1rem;
 }
 
 .search-explanation {
   margin-top: var(--space-8);
   padding-top: var(--space-4);
-  color: var(--muted-ink);
-  font-size: 0.875rem;
   border-top: 0.0625rem solid var(--line);
 }
 
-.search-explanation p {
-  margin: 0;
+.search-explanation > div {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2) var(--space-4);
+  grid-template-columns: 5rem minmax(0, 1fr);
 }
 
-.search-explanation p + p {
+.search-explanation > div + div {
   margin-top: var(--space-3);
 }
 
-.search-explanation strong {
+.search-explanation dt,
+.search-explanation dd {
+  margin: 0;
+  font-size: 0.8125rem;
+}
+
+.search-explanation dt {
   color: var(--ink);
+  font-weight: 700;
+}
+
+.search-explanation dd {
+  color: var(--muted-ink);
+}
+
+.form-action {
+  margin-top: var(--space-6);
 }
 
 .primary-button {
@@ -386,20 +490,20 @@ h3 {
   min-height: 52px;
   align-items: center;
   justify-content: center;
-  margin-top: var(--space-6);
+  margin-top: 0;
   padding: 0.8rem 1.25rem;
   color: white;
   font: inherit;
-  font-weight: 850;
-  background: var(--header);
-  border: 0.125rem solid var(--header);
+  font-weight: 700;
+  background: var(--link);
+  border: 0.125rem solid var(--link);
   border-radius: var(--control-radius);
   cursor: pointer;
 }
 
 .primary-button:hover {
-  background: var(--current);
-  border-color: var(--current);
+  background: #07558E;
+  border-color: #07558E;
 }
 
 .primary-button:active {
@@ -409,8 +513,15 @@ h3 {
 
 .primary-button[aria-disabled="true"] {
   cursor: wait;
-  background: var(--current);
-  border-color: var(--current);
+  background: var(--header);
+  border-color: var(--header);
+}
+
+.action-note {
+  max-width: 34rem;
+  margin: var(--space-3) 0 0;
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
 }
 
 .submit-status {
@@ -968,6 +1079,23 @@ h3 {
     padding-inline: var(--space-4);
   }
 
+  .journey-window {
+    padding-left: var(--space-4);
+  }
+
+  .service-boundary > div,
+  .search-explanation > div {
+    grid-template-columns: minmax(0, 1fr);
+  }
+
+  .service-boundary > div {
+    gap: var(--space-1);
+  }
+
+  .search-explanation > div {
+    gap: 0;
+  }
+
   input[type="datetime-local"] {
     font-size: 1rem;
   }
diff --git a/public_web/templates/public_web/base.html b/public_web/templates/public_web/base.html
index 0313e85..2f3447d 100644
--- a/public_web/templates/public_web/base.html
+++ b/public_web/templates/public_web/base.html
@@ -4,7 +4,7 @@
 <head>
   <meta charset="utf-8">
   <meta name="viewport" content="width=device-width, initial-scale=1">
-  <meta name="theme-color" content="#102F32">
+  <meta name="theme-color" content="#073B66">
   <title>{% block title %}어디 갈까 ??{% endblock %}</title>
   <link rel="stylesheet" href="{% static 'public_web/site.css' %}">
 </head>
@@ -13,7 +13,7 @@
   <header class="site-header">
     <div class="page-shell site-header-inner">
       <a class="site-brand" href="{% url 'public_web:index' %}" aria-label="어디 갈까 ?? 홈">어디 갈까 ??</a>
-      <p class="site-scope">인천 출발 · 직항 일정</p>
+      <p class="site-scope"><span>ICN</span> 인천 출발 · 직항 운항 예정편</p>
     </div>
   </header>
   <main id="main-content" class="page-shell page-main" tabindex="-1">
diff --git a/public_web/templates/public_web/index.html b/public_web/templates/public_web/index.html
index 02c3cea..85d16f1 100644
--- a/public_web/templates/public_web/index.html
+++ b/public_web/templates/public_web/index.html
@@ -5,17 +5,23 @@
 {% block main %}
   <section id="trip-search" class="search-section" aria-labelledby="search-heading">
     <div class="search-intro">
-      <p class="eyebrow">짧은 휴일을 길게 쓰는 방법</p>
+      <p class="eyebrow"><span class="eyebrow-code">ICN</span> 짧은 휴일을 길게 쓰는 방법</p>
       <h1 id="search-heading">이번 휴일, 어디까지 다녀올 수 있을까?</h1>
       <p class="page-lead">
-        비울 수 있는 시간을 알려주면 인천 직항 운항 예정편을 맞춰 보고,
+        비울 수 있는 시간만 알려주세요. 인천에서 오가는 직항 운항 예정편을 맞춰
         현지에서 40시간 이상 머물 수 있는 도시를 찾아드립니다.
       </p>
 
-      <div class="service-boundary" aria-label="서비스 범위">
-        <p><strong>찾아드리는 것</strong><br>최대 6개 도시의 운항 예정편과 예상 현지 체류시간</p>
-        <p><strong>포함하지 않는 것</strong><br>항공권 가격, 좌석, 예약 가능 여부와 실제 운항 보장</p>
-      </div>
+      <dl class="service-boundary" aria-label="서비스 범위">
+        <div>
+          <dt>찾아드리는 것</dt>
+          <dd>최대 6개 도시의 직항 일정과 예상 현지 체류시간</dd>
+        </div>
+        <div>
+          <dt>제공하지 않는 것</dt>
+          <dd>항공권 가격, 좌석, 예약 가능 여부와 실제 운항 보장</dd>
+        </div>
+      </dl>
     </div>
 
     <div class="search-panel">
@@ -44,49 +50,68 @@
             aria-describedby="search-method search-privacy" data-trip-form>
         {% csrf_token %}
         <fieldset>
-          <legend>여행 가능 시간</legend>
-          <p class="form-intro">한국 시각 기준으로 입력해 주세요. 두 시각 사이는 최대 7일입니다.</p>
+          <legend>
+            <span class="form-kicker">여행 시간 입력</span>
+            <span>언제 떠나서, 언제까지 돌아와야 하나요?</span>
+          </legend>
+          <p class="form-intro"><strong>KST · 최대 7일</strong> 두 시각 모두 한국 시각으로 입력해 주세요.</p>
 
-          <div class="form-field{% if form.departure_at.errors %} form-field--error{% endif %}">
-            <label for="{{ form.departure_at.id_for_label }}">{{ form.departure_at.label }}</label>
-            {{ form.departure_at }}
-            <p class="field-hint" id="id_departure_at_hint">인천공항에서 출발할 수 있는 가장 이른 시각</p>
-            {% if form.departure_at.errors %}
-              <ul class="field-errors" id="id_departure_at_error">
-                {% for error in form.departure_at.errors %}<li>{{ error }}</li>{% endfor %}
-              </ul>
-            {% endif %}
-          </div>
+          <div class="journey-window">
+            <div class="form-field{% if form.departure_at.errors %} form-field--error{% endif %}">
+              <label for="{{ form.departure_at.id_for_label }}">
+                <span>{{ form.departure_at.label }}</span>
+                <span class="field-code" aria-hidden="true">ICN · OUT</span>
+              </label>
+              {{ form.departure_at }}
+              <p class="field-hint" id="id_departure_at_hint">인천공항에서 출발할 수 있는 가장 이른 시각</p>
+              {% if form.departure_at.errors %}
+                <ul class="field-errors" id="id_departure_at_error">
+                  {% for error in form.departure_at.errors %}<li>{{ error }}</li>{% endfor %}
+                </ul>
+              {% endif %}
+            </div>
 
-          <div class="window-connector" aria-hidden="true"><span></span></div>
+            <div class="window-connector" aria-hidden="true">
+              <span class="connector-line"></span>
+              <span class="connector-copy">직항 운항 예정편</span>
+              <span class="connector-arrow">↓</span>
+            </div>
 
-          <div class="form-field{% if form.return_by.errors %} form-field--error{% endif %}">
-            <label for="{{ form.return_by.id_for_label }}">{{ form.return_by.label }}</label>
-            {{ form.return_by }}
-            <p class="field-hint" id="id_return_by_hint">귀국편이 인천공항에 도착해야 하는 가장 늦은 시각</p>
-            {% if form.return_by.errors %}
-              <ul class="field-errors" id="id_return_by_error">
-                {% for error in form.return_by.errors %}<li>{{ error }}</li>{% endfor %}
-              </ul>
-            {% endif %}
+            <div class="form-field{% if form.return_by.errors %} form-field--error{% endif %}">
+              <label for="{{ form.return_by.id_for_label }}">
+                <span>{{ form.return_by.label }}</span>
+                <span class="field-code" aria-hidden="true">ICN · RETURN</span>
+              </label>
+              {{ form.return_by }}
+              <p class="field-hint" id="id_return_by_hint">귀국편이 인천공항에 도착해야 하는 가장 늦은 시각</p>
+              {% if form.return_by.errors %}
+                <ul class="field-errors" id="id_return_by_error">
+                  {% for error in form.return_by.errors %}<li>{{ error }}</li>{% endfor %}
+                </ul>
+              {% endif %}
+            </div>
           </div>
         </fieldset>
 
-        <div class="search-explanation">
-          <p id="search-method">
-            <strong>40시간 기준</strong>은 현지 공항 도착 예상시각부터 귀국편 출발
-            예상시각까지입니다. 공항 이동, 입출국 수속과 시내 이동 시간은 차감하지 않습니다.
-          </p>
-          <p id="search-privacy">
-            입력한 시각과 계산 결과는 이 화면을 만드는 동안에만 사용하며 저장하지 않습니다.
-          </p>
-        </div>
+        <dl class="search-explanation">
+          <div>
+            <dt>40시간 기준</dt>
+            <dd id="search-method">현지 공항 도착 예상시각부터 귀국편 출발 예상시각까지입니다. 공항 이동, 입출국 수속과 시내 이동 시간은 차감하지 않습니다.</dd>
+          </div>
+          <div>
+            <dt>입력 정보</dt>
+            <dd id="search-privacy">입력한 시각과 계산 결과는 이 화면을 만드는 동안에만 사용하며 저장하지 않습니다.</dd>
+          </div>
+        </dl>
 
-        <button class="primary-button" type="submit" data-submit-button>
-          <span data-submit-label>갈 수 있는 곳 찾기</span>
-        </button>
-        <p class="submit-status" data-submit-status role="status"
-           aria-live="polite" aria-atomic="true"></p>
+        <div class="form-action">
+          <button class="primary-button" type="submit" data-submit-button>
+            <span data-submit-label>갈 수 있는 곳 찾기</span>
+          </button>
+          <p class="action-note">예약 가능한 항공편이 아닌, 검수·게시된 운항 예정편을 조회합니다.</p>
+          <p class="submit-status" data-submit-status role="status"
+             aria-live="polite" aria-atomic="true"></p>
+        </div>
       </form>
     </div>
   </section>


