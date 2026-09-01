## `feat(frontend): redesign trip input flow`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 96ecd6d..690254e 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -173,10 +173,10 @@ WARNING_PUBLIC_SOURCE_LOCATOR: Final = (
 )
 ENTRY_FRESHNESS_MINUTES: Final = 36 * 60
 WARNING_FRESHNESS_MINUTES: Final = 8 * 60
-SITE_CSS_SHA256: Final = "0b4009a2837437f6af89241fa70a271e99ef2466ba27a38a38de4ed9de161f77"
-SITE_CSS_BYTES: Final = 8_534
-SITE_JS_SHA256: Final = "79754d4ab020672f48ea1d7311fd1583f40e19c50a6af41b4bdf2c1b438c97d4"
-SITE_JS_BYTES: Final = 1_544
+SITE_CSS_SHA256: Final = "08e4bbbcad4b0b451da4a3de39f8afb01adfcfcde7d49e3f3552925a60ab3797"
+SITE_CSS_BYTES: Final = 9_136
+SITE_JS_SHA256: Final = "8f689c169349770655b803b22b88578d7cc7ed2f1266cdc13625e918bc4888b6"
+SITE_JS_BYTES: Final = 1_557
 _SIGNAL_INTERRUPTED = False
 _SIGNAL_CLEANUP_DEPTH = 0
 
diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index 3c62766..c4355e2 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -49,14 +49,14 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        8_534,
-        "0b4009a2837437f6af89241fa70a271e99ef2466ba27a38a38de4ed9de161f77",
+        9_136,
+        "08e4bbbcad4b0b451da4a3de39f8afb01adfcfcde7d49e3f3552925a60ab3797",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
         "text/javascript",
-        1_544,
-        "79754d4ab020672f48ea1d7311fd1583f40e19c50a6af41b4bdf2c1b438c97d4",
+        1_557,
+        "8f689c169349770655b803b22b88578d7cc7ed2f1266cdc13625e918bc4888b6",
     ),
 }
 
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index f3b71e2..b7ecdd0 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -184,6 +184,47 @@ h2 {
   font-size: clamp(1rem, 2.5vw, 1.1875rem);
 }
 
+.trip-layout {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: var(--space-12) var(--space-16);
+  align-items: flex-start;
+}
+
+.trip-intro {
+  min-width: 0;
+  flex: 5 1 24rem;
+}
+
+.trip-intro h1 {
+  max-width: 12ch;
+}
+
+.trip-intro .page-lead,
+.service-limit,
+.privacy-note {
+  max-width: 34rem;
+}
+
+.service-limit {
+  margin: var(--space-8) 0 0;
+  padding-top: var(--space-4);
+  border-top: 0.0625rem solid var(--line);
+  font-weight: 700;
+}
+
+.privacy-note {
+  margin: var(--space-4) 0 0;
+  color: var(--muted-ink);
+  font-size: 0.9375rem;
+}
+
+.trip-panel {
+  min-width: 0;
+  flex: 7 1 34rem;
+}
+
 .error-summary {
   max-width: 48rem;
   margin-bottom: 1.5rem;
@@ -218,7 +259,7 @@ h2 {
 }
 
 .trip-form {
-  max-width: 48rem;
+  max-width: none;
   padding: clamp(1rem, 5vw, 2rem);
   background: var(--surface);
   border: 0.0625rem solid var(--line);
diff --git a/public_web/static/public_web/site.js b/public_web/static/public_web/site.js
index d63417b..c0a341f 100644
--- a/public_web/static/public_web/site.js
+++ b/public_web/static/public_web/site.js
@@ -11,7 +11,7 @@
     button.removeAttribute("aria-disabled");
     const label = button.querySelector("[data-submit-label]");
     const status = form.querySelector("[data-submit-status]");
-    if (label) label.textContent = "게시 정보 확인";
+    if (label) label.textContent = "입국요건·여행경보 보기";
     if (status) status.textContent = "";
   };
 
diff --git a/public_web/templates/public_web/index.html b/public_web/templates/public_web/index.html
index 38d96ca..ff0d9dd 100644
--- a/public_web/templates/public_web/index.html
+++ b/public_web/templates/public_web/index.html
@@ -1,89 +1,98 @@
 {% extends "public_web/base.html" %}
 
-{% block title %}여행준비 — 일본 정보 확인{% endblock %}
+{% block title %}여행준비 — 일본 입국요건·여행경보{% endblock %}
 
 {% block nav_links %}
-  <a href="{% url 'public_web:index' %}" aria-current="page">여행준비</a>
-  <a href="{% url 'public_web:results' %}">게시 정보 보기</a>
+  <a href="{% url 'public_web:index' %}" aria-current="page">정보 입력</a>
+  <a href="{% url 'public_web:results' %}">게시본 보기</a>
 {% endblock %}
 
 {% block main %}
-  <div class="page-heading">
-    <p class="eyebrow">공식 source · 저장하지 않는 입력</p>
-    <h1>일본 여행 정보 확인</h1>
-    <p id="form-privacy-note" class="page-lead">
-      대한민국 일반여권 여행자를 위한 공식 source 사실을 확인합니다.
-      입력은 이 요청 안에서만 검증하며 저장하지 않습니다.
-    </p>
-  </div>
-
-  {% if form.errors %}
-    <section class="error-summary" role="alert" aria-live="assertive"
-             aria-labelledby="form-error-heading" tabindex="-1"
-             data-error-summary>
-      <h2 id="form-error-heading">입력 내용을 확인해 주세요</h2>
-      <p>오류가 있는 항목으로 이동해 내용을 수정할 수 있습니다.</p>
-      <ul>
-        {% for field in form %}
-          {% for error in field.errors %}
-            <li><a href="#{{ field.id_for_label }}">{{ field.label }}: {{ error }}</a></li>
-          {% endfor %}
-        {% endfor %}
-        {% for error in form.non_field_errors %}
-          <li><a href="#trip-form">여행 범위: {{ error }}</a></li>
-        {% endfor %}
-      </ul>
+  <div class="trip-layout">
+    <section class="trip-intro">
+      <p class="eyebrow">대한민국 일반여권 · 목적지 일본</p>
+      <h1>일본 여행 전 확인할 입국요건과 여행경보</h1>
+      <p class="page-lead">
+        공식 출처에서 수집해 검수·게시한 입국요건 사실과 여행경보를 보여드립니다.
+      </p>
+      <p class="service-limit">
+        이 서비스는 입국 여부, 법률 해석, 여행 목적·날짜별 적용성을 판정하지 않습니다.
+      </p>
+      <p id="form-privacy-note" class="privacy-note">
+        목적지와 날짜는 이 요청 안에서 형식과 순서만 확인하며 저장하거나 성공 결과에 표시하지 않습니다.
+      </p>
     </section>
-  {% endif %}
 
-  <form id="trip-form" class="trip-form" method="post"
-        action="{% url 'public_web:index' %}" novalidate
-        aria-describedby="form-privacy-note" data-trip-form>
-    {% csrf_token %}
-    <fieldset>
-      <legend>여행 범위</legend>
-
-      <div class="form-field{% if form.destination.errors %} form-field--error{% endif %}">
-        <label for="{{ form.destination.id_for_label }}">{{ form.destination.label }}</label>
-        {{ form.destination }}
-        <p class="field-hint" id="id_destination_hint">Phase 0 공개 범위는 일본으로 고정되어 있습니다.</p>
-        {% if form.destination.errors %}
-          <ul class="field-errors" id="id_destination_error">
-            {% for error in form.destination.errors %}<li>{{ error }}</li>{% endfor %}
+    <div class="trip-panel">
+      {% if form.errors %}
+        <section class="error-summary" role="alert" aria-live="assertive"
+                 aria-labelledby="form-error-heading" tabindex="-1"
+                 data-error-summary>
+          <h2 id="form-error-heading">입력 내용을 확인해 주세요</h2>
+          <p>오류가 있는 항목으로 이동해 내용을 수정할 수 있습니다.</p>
+          <ul>
+            {% for field in form %}
+              {% for error in field.errors %}
+                <li><a href="#{{ field.id_for_label }}">{{ field.label }}: {{ error }}</a></li>
+              {% endfor %}
+            {% endfor %}
+            {% for error in form.non_field_errors %}
+              <li><a href="#trip-form">여행 일정: {{ error }}</a></li>
+            {% endfor %}
           </ul>
-        {% endif %}
-      </div>
+        </section>
+      {% endif %}
 
-      <div class="form-field{% if form.departure_date.errors %} form-field--error{% endif %}">
-        <label for="{{ form.departure_date.id_for_label }}">{{ form.departure_date.label }}</label>
-        {{ form.departure_date }}
-        <p class="field-hint" id="id_departure_date_hint">날짜 적용성은 결과에서 계산하지 않습니다.</p>
-        {% if form.departure_date.errors %}
-          <ul class="field-errors" id="id_departure_date_error">
-            {% for error in form.departure_date.errors %}<li>{{ error }}</li>{% endfor %}
-          </ul>
-        {% endif %}
-      </div>
+      <form id="trip-form" class="trip-form" method="post"
+            action="{% url 'public_web:index' %}" novalidate
+            aria-describedby="form-privacy-note" data-trip-form>
+        {% csrf_token %}
+        <fieldset>
+          <legend>여행 일정 입력</legend>
 
-      <div class="form-field{% if form.return_date.errors %} form-field--error{% endif %}">
-        <label for="{{ form.return_date.id_for_label }}">{{ form.return_date.label }}</label>
-        {{ form.return_date }}
-        <p class="field-hint" id="id_return_date_hint">출국일과 같거나 이후 날짜를 입력해 주세요.</p>
-        {% if form.return_date.errors %}
-          <ul class="field-errors" id="id_return_date_error">
-            {% for error in form.return_date.errors %}<li>{{ error }}</li>{% endfor %}
-          </ul>
-        {% endif %}
-      </div>
-    </fieldset>
-    <button class="primary-button" type="submit" data-submit-button>
-      <span data-submit-label>게시 정보 확인</span>
-    </button>
-    <p class="submit-status" data-submit-status role="status"
-       aria-live="polite" aria-atomic="true"></p>
-  </form>
+          <div class="form-field{% if form.destination.errors %} form-field--error{% endif %}">
+            <label for="{{ form.destination.id_for_label }}">{{ form.destination.label }}</label>
+            {{ form.destination }}
+            <p class="field-hint" id="id_destination_hint">현재 공개 범위는 대한민국 일반여권 여행자의 일본 여행입니다.</p>
+            {% if form.destination.errors %}
+              <ul class="field-errors" id="id_destination_error">
+                {% for error in form.destination.errors %}<li>{{ error }}</li>{% endfor %}
+              </ul>
+            {% endif %}
+          </div>
+
+          <div class="form-field{% if form.departure_date.errors %} form-field--error{% endif %}">
+            <label for="{{ form.departure_date.id_for_label }}">{{ form.departure_date.label }}</label>
+            {{ form.departure_date }}
+            <p class="field-hint" id="id_departure_date_hint">날짜는 형식과 순서만 확인하며 게시 정보의 적용성을 계산하는 데 사용하지 않습니다.</p>
+            {% if form.departure_date.errors %}
+              <ul class="field-errors" id="id_departure_date_error">
+                {% for error in form.departure_date.errors %}<li>{{ error }}</li>{% endfor %}
+              </ul>
+            {% endif %}
+          </div>
+
+          <div class="form-field{% if form.return_date.errors %} form-field--error{% endif %}">
+            <label for="{{ form.return_date.id_for_label }}">{{ form.return_date.label }}</label>
+            {{ form.return_date }}
+            <p class="field-hint" id="id_return_date_hint">출국일과 같거나 이후인 날짜를 입력해 주세요.</p>
+            {% if form.return_date.errors %}
+              <ul class="field-errors" id="id_return_date_error">
+                {% for error in form.return_date.errors %}<li>{{ error }}</li>{% endfor %}
+              </ul>
+            {% endif %}
+          </div>
+        </fieldset>
+        <button class="primary-button" type="submit" data-submit-button>
+          <span data-submit-label>입국요건·여행경보 보기</span>
+        </button>
+        <p class="submit-status" data-submit-status role="status"
+           aria-live="polite" aria-atomic="true"></p>
+      </form>
+    </div>
+  </div>
 {% endblock %}
 
 {% block footer %}
-  <p>여행 목적과 날짜별 적용 여부는 공식 기관에서 다시 확인해 주세요.</p>
+  <p>여행 목적과 날짜에 맞는 최신 조건은 공식 기관에서 다시 확인해 주세요.</p>
 {% endblock %}
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
index 978c25c..49ea18a 100644
--- a/public_web/tests/test_accessibility_contract.py
+++ b/public_web/tests/test_accessibility_contract.py
@@ -37,6 +37,10 @@ class AccessiblePresentationContractTests(SimpleTestCase):
         self.assertContains(response, 'for="id_destination"')
         self.assertContains(response, 'for="id_departure_date"')
         self.assertContains(response, 'for="id_return_date"')
+        self.assertContains(response, "대한민국 일반여권 · 목적지 일본")
+        self.assertContains(response, "일본 여행 전 확인할 입국요건과 여행경보")
+        self.assertContains(response, "입국요건·여행경보 보기")
+        self.assertContains(response, 'aria-describedby="form-privacy-note"')
         self.assertLess(body.index("skip-link"), body.index("<header"))
         self.assertLess(body.index("<header"), body.index("<main"))
         self.assertLess(body.index("<main"), body.index("<footer"))
diff --git a/public_web/tests/test_privacy_form.py b/public_web/tests/test_privacy_form.py
index 8a21965..2c1d739 100644
--- a/public_web/tests/test_privacy_form.py
+++ b/public_web/tests/test_privacy_form.py
@@ -18,7 +18,21 @@ class TripFormPrivacyTests(SimpleTestCase):
     def test_get_is_unbound_and_query_input_is_removed(self):
         response = self.client.get(reverse("public_web:index"))
         self.assertEqual(response.status_code, 200)
-        self.assertContains(response, "일본 여행 정보 확인")
+        self.assertContains(response, "대한민국 일반여권 · 목적지 일본")
+        self.assertContains(response, "일본 여행 전 확인할 입국요건과 여행경보")
+        self.assertContains(
+            response,
+            "공식 출처에서 수집해 검수·게시한 입국요건 사실과 여행경보를 보여드립니다.",
+        )
+        self.assertContains(
+            response,
+            "이 서비스는 입국 여부, 법률 해석, 여행 목적·날짜별 적용성을 판정하지 않습니다.",
+        )
+        self.assertContains(
+            response,
+            "목적지와 날짜는 이 요청 안에서 형식과 순서만 확인하며 저장하거나 성공 결과에 표시하지 않습니다.",
+        )
+        self.assertContains(response, "입국요건·여행경보 보기")
         self.assertNotIn("sessionid", response.cookies)
 
         canonical = self.client.get(
@@ -99,6 +113,7 @@ class TripFormPrivacyTests(SimpleTestCase):
         )
 
         self.assertEqual(response.status_code, 200)
+        self.assertContains(response, "입력 내용을 확인해 주세요")
         self.assertContains(
             response,
             "귀국일은 출국일과 같거나 그 이후여야 합니다.",


