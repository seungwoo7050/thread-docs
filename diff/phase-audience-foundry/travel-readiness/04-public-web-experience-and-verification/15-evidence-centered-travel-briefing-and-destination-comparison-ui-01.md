# 공식 근거 중심 여행 브리핑과 목적지 비교 UI

## `feat(web): add the responsive accessible presentation`

diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
new file mode 100644
index 0000000..31821a5
--- /dev/null
+++ b/public_web/static/public_web/site.css
@@ -0,0 +1,465 @@
+:root {
+  color-scheme: light;
+  --ink: #142f35;
+  --muted-ink: #50656a;
+  --canvas: #f3f7f5;
+  --surface: #ffffff;
+  --surface-soft: #eaf2ef;
+  --line: #b9ccc7;
+  --brand: #123f46;
+  --brand-strong: #082d33;
+  --accent: #d7a441;
+  --focus: #006d8f;
+  --danger: #9f2f24;
+  --shadow: 0 0.75rem 2rem rgb(20 47 53 / 10%);
+  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif;
+  font-size: 100%;
+  line-height: 1.6;
+}
+
+* {
+  box-sizing: border-box;
+}
+
+html {
+  min-width: 0;
+  background: var(--canvas);
+}
+
+body {
+  min-width: 0;
+  min-height: 100vh;
+  margin: 0;
+  color: var(--ink);
+  background:
+    linear-gradient(180deg, rgb(215 164 65 / 10%), transparent 18rem),
+    var(--canvas);
+}
+
+:where(p, li, dt, dd, a, strong, label, legend) {
+  overflow-wrap: anywhere;
+  word-break: keep-all;
+}
+
+:where(a, button, input, select, [tabindex="0"]):focus-visible {
+  outline: 0.1875rem solid var(--focus);
+  outline-offset: 0.1875rem;
+}
+
+.skip-link:focus-visible,
+.site-header a:focus-visible {
+  outline-color: white;
+  box-shadow: 0 0 0 0.4375rem #000000;
+}
+
+[hidden] {
+  display: none !important;
+}
+
+.page-shell {
+  width: min(100% - 2rem, 72rem);
+  min-width: 0;
+  margin-inline: auto;
+}
+
+.skip-link {
+  position: fixed;
+  z-index: 10;
+  top: 0.5rem;
+  left: 0.5rem;
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  padding: 0.5rem 0.875rem;
+  color: white;
+  background: var(--brand-strong);
+  border-radius: 0.375rem;
+  transform: translateY(-150%);
+}
+
+.skip-link:focus {
+  transform: translateY(0);
+}
+
+.site-header {
+  color: white;
+  background: var(--brand);
+  border-bottom: 0.25rem solid var(--accent);
+}
+
+.site-nav {
+  display: flex;
+  flex-wrap: wrap;
+  gap: 0.5rem;
+  padding-block: 0.625rem;
+}
+
+.site-nav a {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  padding: 0.5rem 0.875rem;
+  color: white;
+  font-weight: 700;
+  text-decoration-thickness: 0.08em;
+  text-underline-offset: 0.2em;
+  border-radius: 0.4rem;
+}
+
+.site-nav a[aria-current="page"] {
+  color: var(--brand-strong);
+  background: white;
+  text-decoration: none;
+}
+
+.page-main {
+  padding-block: clamp(2rem, 7vw, 5rem);
+}
+
+.page-main > * {
+  min-width: 0;
+}
+
+.page-heading {
+  max-width: 48rem;
+  margin-bottom: clamp(1.5rem, 4vw, 2.5rem);
+}
+
+.eyebrow {
+  margin: 0 0 0.375rem;
+  color: var(--brand);
+  font-size: 0.875rem;
+  font-weight: 800;
+  letter-spacing: 0.05em;
+}
+
+h1,
+h2 {
+  line-height: 1.25;
+  text-wrap: balance;
+}
+
+h1 {
+  margin: 0;
+  font-size: clamp(2rem, 7vw, 3.75rem);
+  letter-spacing: -0.035em;
+}
+
+h2 {
+  margin: 0;
+  font-size: clamp(1.35rem, 4vw, 1.8rem);
+  letter-spacing: -0.02em;
+}
+
+.page-lead {
+  max-width: 42rem;
+  margin: 1rem 0 0;
+  color: var(--muted-ink);
+  font-size: clamp(1rem, 2.5vw, 1.1875rem);
+}
+
+.error-summary {
+  max-width: 48rem;
+  margin-bottom: 1.5rem;
+  padding: clamp(1rem, 4vw, 1.5rem);
+  background: #fff7f5;
+  border: 0.125rem solid var(--danger);
+  border-left-width: 0.5rem;
+  border-radius: 0.75rem;
+}
+
+.error-summary p {
+  margin-block: 0.5rem;
+}
+
+.error-summary ul,
+.field-errors {
+  margin: 0.5rem 0 0;
+  padding-left: 1.4rem;
+}
+
+.error-summary a {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  color: var(--danger);
+  font-weight: 700;
+}
+
+.trip-form {
+  max-width: 48rem;
+  padding: clamp(1rem, 5vw, 2rem);
+  background: var(--surface);
+  border: 0.0625rem solid var(--line);
+  border-radius: 1rem;
+  box-shadow: var(--shadow);
+}
+
+.trip-form fieldset {
+  min-width: 0;
+  margin: 0;
+  padding: 0;
+  border: 0;
+}
+
+.trip-form legend {
+  margin-bottom: 1.25rem;
+  padding: 0;
+  font-size: 1.35rem;
+  font-weight: 800;
+}
+
+.form-field + .form-field {
+  margin-top: 1.25rem;
+}
+
+.form-field {
+  min-width: 0;
+}
+
+.form-field label {
+  display: block;
+  margin-bottom: 0.375rem;
+  font-weight: 800;
+}
+
+.form-field :where(input, select) {
+  display: block;
+  width: min(100%, 28rem);
+  min-width: 0;
+  max-width: 100%;
+  min-height: 48px;
+  padding: 0.625rem 0.75rem;
+  color: var(--ink);
+  font: inherit;
+  font-size: 1rem;
+  background: white;
+  border: 0.125rem solid #718984;
+  border-radius: 0.5rem;
+}
+
+.form-field--error :where(input, select) {
+  border-color: var(--danger);
+  border-style: double;
+}
+
+.field-hint {
+  max-width: 38rem;
+  margin: 0.375rem 0 0;
+  color: var(--muted-ink);
+  font-size: 0.9375rem;
+}
+
+.field-errors {
+  color: var(--danger);
+  font-weight: 700;
+}
+
+.primary-button {
+  display: inline-flex;
+  width: min(100%, 20rem);
+  min-height: 48px;
+  align-items: center;
+  justify-content: center;
+  margin-top: 1.75rem;
+  padding: 0.75rem 1.25rem;
+  color: white;
+  font: inherit;
+  font-weight: 800;
+  background: var(--brand);
+  border: 0.125rem solid var(--brand);
+  border-radius: 0.6rem;
+  cursor: pointer;
+}
+
+.primary-button:hover {
+  background: var(--brand-strong);
+}
+
+.primary-button[aria-disabled="true"] {
+  cursor: wait;
+  border-style: dashed;
+}
+
+.submit-status {
+  min-height: 1.6em;
+  margin: 0.5rem 0 0;
+  color: var(--muted-ink);
+  font-weight: 700;
+}
+
+.publication-grid {
+  display: grid;
+  grid-template-columns: minmax(0, 1fr);
+  gap: 1.25rem;
+}
+
+.publication-card {
+  min-width: 0;
+  padding: clamp(1rem, 4vw, 1.75rem);
+  background: var(--surface);
+  border: 0.125rem solid var(--line);
+  border-top: 0.4rem solid var(--brand);
+  border-radius: 1rem;
+  box-shadow: var(--shadow);
+}
+
+.publication-card[data-state="empty"] {
+  border-style: dashed;
+}
+
+.publication-card[data-state="unavailable"],
+.publication-card[data-state="stale"] {
+  border-top-color: var(--accent);
+  border-top-style: double;
+}
+
+.publication-card[data-state="server-error"] {
+  border-top-color: var(--danger);
+  border-top-style: double;
+}
+
+.status-line {
+  display: flex;
+  flex-wrap: wrap;
+  gap: 0.5rem;
+  align-items: center;
+  margin-block: 1rem 0.5rem;
+}
+
+.status-symbol {
+  display: inline-grid;
+  width: 1.75rem;
+  height: 1.75rem;
+  flex: 0 0 auto;
+  place-items: center;
+  color: white;
+  background: var(--brand);
+  border-radius: 50%;
+  font-weight: 900;
+}
+
+[data-state="ready"] .status-symbol::before { content: "✓"; }
+[data-state="empty"] .status-symbol::before { content: "○"; }
+[data-state="unavailable"] .status-symbol::before { content: "?"; }
+[data-state="stale"] .status-symbol::before { content: "!"; }
+[data-state="server-error"] .status-symbol::before { content: "×"; }
+
+[data-state="empty"] .status-symbol,
+[data-state="unavailable"] .status-symbol,
+[data-state="stale"] .status-symbol {
+  color: var(--brand-strong);
+  background: var(--accent);
+}
+
+[data-state="server-error"] .status-symbol {
+  background: var(--danger);
+}
+
+.fact-list {
+  min-width: 0;
+  margin-block: 1.25rem;
+  border-top: 0.0625rem solid var(--line);
+}
+
+.fact-list dt,
+.fact-list dd {
+  min-width: 0;
+  margin: 0;
+  padding: 0.7rem 0;
+  border-bottom: 0.0625rem solid var(--line);
+}
+
+.fact-list dt {
+  color: var(--muted-ink);
+  font-size: 0.875rem;
+  font-weight: 800;
+}
+
+.fact-list dd {
+  padding-top: 0;
+}
+
+.source-link {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  margin-inline-start: 0.25rem;
+  padding-inline: 0.25rem;
+  color: var(--brand);
+  font-weight: 800;
+}
+
+.verification-note {
+  margin-bottom: 0;
+  padding: 1rem;
+  background: var(--surface-soft);
+  border-left: 0.35rem solid var(--accent);
+  border-radius: 0.4rem;
+}
+
+.site-footer {
+  padding-block: 1.5rem;
+  color: white;
+  background: var(--brand-strong);
+}
+
+.site-footer p {
+  max-width: 48rem;
+  margin: 0;
+}
+
+@media (min-width: 40rem) {
+  .fact-list {
+    display: grid;
+    grid-template-columns: minmax(8rem, 0.7fr) minmax(0, 1.6fr);
+  }
+
+  .fact-list dt,
+  .fact-list dd {
+    padding-block: 0.75rem;
+  }
+
+  .fact-list dd {
+    padding-left: 1rem;
+  }
+}
+
+@media (min-width: 64rem) {
+  .publication-grid {
+    grid-template-columns: repeat(2, minmax(0, 1fr));
+  }
+}
+
+@media (max-width: 30rem) {
+  .page-shell {
+    width: min(100% - 1.25rem, 72rem);
+  }
+
+  .site-nav a {
+    flex: 1 1 9rem;
+    justify-content: center;
+  }
+}
+
+@media (pointer: coarse) {
+  .site-nav a,
+  .source-link,
+  .error-summary a {
+    min-height: 48px;
+  }
+}
+
+@media (prefers-reduced-motion: no-preference) {
+  :where(a, button, input, select) {
+    transition: outline-offset 120ms ease, background-color 120ms ease;
+  }
+}
+
+@media (forced-colors: active) {
+  .publication-card,
+  .trip-form,
+  .verification-note {
+    border: 0.125rem solid CanvasText;
+  }
+}
diff --git a/public_web/static/public_web/site.js b/public_web/static/public_web/site.js
new file mode 100644
index 0000000..d63417b
--- /dev/null
+++ b/public_web/static/public_web/site.js
@@ -0,0 +1,45 @@
+(() => {
+  "use strict";
+
+  const forms = document.querySelectorAll("[data-trip-form]");
+
+  const resetSubmitState = (form) => {
+    form.dataset.submitting = "false";
+    form.removeAttribute("aria-busy");
+    const button = form.querySelector("[data-submit-button]");
+    if (!button) return;
+    button.removeAttribute("aria-disabled");
+    const label = button.querySelector("[data-submit-label]");
+    const status = form.querySelector("[data-submit-status]");
+    if (label) label.textContent = "게시 정보 확인";
+    if (status) status.textContent = "";
+  };
+
+  forms.forEach((form) => {
+    resetSubmitState(form);
+    form.addEventListener("submit", (event) => {
+      if (form.dataset.submitting === "true") {
+        event.preventDefault();
+        return;
+      }
+      form.dataset.submitting = "true";
+      form.setAttribute("aria-busy", "true");
+      const button = form.querySelector("[data-submit-button]");
+      if (!button) return;
+      button.setAttribute("aria-disabled", "true");
+      const label = button.querySelector("[data-submit-label]");
+      const status = form.querySelector("[data-submit-status]");
+      if (label) label.textContent = "제출 중…";
+      if (status) status.textContent = "게시 정보를 불러오는 중…";
+    });
+  });
+
+  window.addEventListener("pageshow", () => {
+    forms.forEach(resetSubmitState);
+  });
+
+  const errorSummary = document.querySelector("[data-error-summary]");
+  if (errorSummary) {
+    window.requestAnimationFrame(() => errorSummary.focus());
+  }
+})();
diff --git a/public_web/templates/public_web/base.html b/public_web/templates/public_web/base.html
new file mode 100644
index 0000000..b4a0ca2
--- /dev/null
+++ b/public_web/templates/public_web/base.html
@@ -0,0 +1,30 @@
+{% load static %}
+<!doctype html>
+<html lang="ko">
+<head>
+  <meta charset="utf-8">
+  <meta name="viewport" content="width=device-width, initial-scale=1">
+  <meta name="theme-color" content="#123f46">
+  <title>{% block title %}여행준비{% endblock %}</title>
+  <link rel="stylesheet" href="{% static 'public_web/site.css' %}">
+</head>
+<body>
+  <a class="skip-link" href="#main-content">본문으로 건너뛰기</a>
+  <header class="site-header">
+    <div class="page-shell">
+      <nav class="site-nav" aria-label="주요 메뉴">
+        {% block nav_links %}{% endblock %}
+      </nav>
+    </div>
+  </header>
+  <main id="main-content" class="page-shell page-main" tabindex="-1">
+    {% block main %}{% endblock %}
+  </main>
+  <footer class="site-footer">
+    <div class="page-shell">
+      {% block footer %}{% endblock %}
+    </div>
+  </footer>
+  <script src="{% static 'public_web/site.js' %}" defer></script>
+</body>
+</html>
diff --git a/public_web/templates/public_web/index.html b/public_web/templates/public_web/index.html
index b504f32..38d96ca 100644
--- a/public_web/templates/public_web/index.html
+++ b/public_web/templates/public_web/index.html
@@ -1,83 +1,89 @@
-<!doctype html>
-<html lang="ko">
-<head>
-  <meta charset="utf-8">
-  <meta name="viewport" content="width=device-width, initial-scale=1">
-  <title>여행준비 — 일본 정보 확인</title>
-</head>
-<body>
-  <header>
-    <nav aria-label="주요 메뉴">
-      <a href="{% url 'public_web:index' %}" aria-current="page">여행준비</a>
-      <a href="{% url 'public_web:results' %}">게시 정보 보기</a>
-    </nav>
-  </header>
-  <main id="main-content">
+{% extends "public_web/base.html" %}
+
+{% block title %}여행준비 — 일본 정보 확인{% endblock %}
+
+{% block nav_links %}
+  <a href="{% url 'public_web:index' %}" aria-current="page">여행준비</a>
+  <a href="{% url 'public_web:results' %}">게시 정보 보기</a>
+{% endblock %}
+
+{% block main %}
+  <div class="page-heading">
+    <p class="eyebrow">공식 source · 저장하지 않는 입력</p>
     <h1>일본 여행 정보 확인</h1>
-    <p>
+    <p id="form-privacy-note" class="page-lead">
       대한민국 일반여권 여행자를 위한 공식 source 사실을 확인합니다.
       입력은 이 요청 안에서만 검증하며 저장하지 않습니다.
     </p>
+  </div>
 
-    {% if form.errors %}
-      <section role="alert" aria-labelledby="form-error-heading" tabindex="-1">
-        <h2 id="form-error-heading">입력 내용을 확인해 주세요</h2>
-        <ul>
-          {% for field in form %}
-            {% for error in field.errors %}
-              <li><a href="#{{ field.id_for_label }}">{{ field.label }}: {{ error }}</a></li>
-            {% endfor %}
+  {% if form.errors %}
+    <section class="error-summary" role="alert" aria-live="assertive"
+             aria-labelledby="form-error-heading" tabindex="-1"
+             data-error-summary>
+      <h2 id="form-error-heading">입력 내용을 확인해 주세요</h2>
+      <p>오류가 있는 항목으로 이동해 내용을 수정할 수 있습니다.</p>
+      <ul>
+        {% for field in form %}
+          {% for error in field.errors %}
+            <li><a href="#{{ field.id_for_label }}">{{ field.label }}: {{ error }}</a></li>
           {% endfor %}
-          {% for error in form.non_field_errors %}
-            <li>{{ error }}</li>
-          {% endfor %}
-        </ul>
-      </section>
-    {% endif %}
+        {% endfor %}
+        {% for error in form.non_field_errors %}
+          <li><a href="#trip-form">여행 범위: {{ error }}</a></li>
+        {% endfor %}
+      </ul>
+    </section>
+  {% endif %}
+
+  <form id="trip-form" class="trip-form" method="post"
+        action="{% url 'public_web:index' %}" novalidate
+        aria-describedby="form-privacy-note" data-trip-form>
+    {% csrf_token %}
+    <fieldset>
+      <legend>여행 범위</legend>
 
-    <form method="post" action="{% url 'public_web:index' %}" novalidate>
-      {% csrf_token %}
-      <fieldset>
-        <legend>여행 범위</legend>
+      <div class="form-field{% if form.destination.errors %} form-field--error{% endif %}">
+        <label for="{{ form.destination.id_for_label }}">{{ form.destination.label }}</label>
+        {{ form.destination }}
+        <p class="field-hint" id="id_destination_hint">Phase 0 공개 범위는 일본으로 고정되어 있습니다.</p>
+        {% if form.destination.errors %}
+          <ul class="field-errors" id="id_destination_error">
+            {% for error in form.destination.errors %}<li>{{ error }}</li>{% endfor %}
+          </ul>
+        {% endif %}
+      </div>
 
-        <div>
-          <label for="{{ form.destination.id_for_label }}">{{ form.destination.label }}</label>
-          {{ form.destination }}
-          <p id="id_destination_hint">Phase 0 공개 범위는 일본으로 고정되어 있습니다.</p>
-          {% if form.destination.errors %}
-            <ul id="id_destination_error">
-              {% for error in form.destination.errors %}<li>{{ error }}</li>{% endfor %}
-            </ul>
-          {% endif %}
-        </div>
+      <div class="form-field{% if form.departure_date.errors %} form-field--error{% endif %}">
+        <label for="{{ form.departure_date.id_for_label }}">{{ form.departure_date.label }}</label>
+        {{ form.departure_date }}
+        <p class="field-hint" id="id_departure_date_hint">날짜 적용성은 결과에서 계산하지 않습니다.</p>
+        {% if form.departure_date.errors %}
+          <ul class="field-errors" id="id_departure_date_error">
+            {% for error in form.departure_date.errors %}<li>{{ error }}</li>{% endfor %}
+          </ul>
+        {% endif %}
+      </div>
 
-        <div>
-          <label for="{{ form.departure_date.id_for_label }}">{{ form.departure_date.label }}</label>
-          {{ form.departure_date }}
-          <p id="id_departure_date_hint">날짜 적용성은 결과에서 계산하지 않습니다.</p>
-          {% if form.departure_date.errors %}
-            <ul id="id_departure_date_error">
-              {% for error in form.departure_date.errors %}<li>{{ error }}</li>{% endfor %}
-            </ul>
-          {% endif %}
-        </div>
+      <div class="form-field{% if form.return_date.errors %} form-field--error{% endif %}">
+        <label for="{{ form.return_date.id_for_label }}">{{ form.return_date.label }}</label>
+        {{ form.return_date }}
+        <p class="field-hint" id="id_return_date_hint">출국일과 같거나 이후 날짜를 입력해 주세요.</p>
+        {% if form.return_date.errors %}
+          <ul class="field-errors" id="id_return_date_error">
+            {% for error in form.return_date.errors %}<li>{{ error }}</li>{% endfor %}
+          </ul>
+        {% endif %}
+      </div>
+    </fieldset>
+    <button class="primary-button" type="submit" data-submit-button>
+      <span data-submit-label>게시 정보 확인</span>
+    </button>
+    <p class="submit-status" data-submit-status role="status"
+       aria-live="polite" aria-atomic="true"></p>
+  </form>
+{% endblock %}
 
-        <div>
-          <label for="{{ form.return_date.id_for_label }}">{{ form.return_date.label }}</label>
-          {{ form.return_date }}
-          <p id="id_return_date_hint">출국일과 같거나 이후 날짜를 입력해 주세요.</p>
-          {% if form.return_date.errors %}
-            <ul id="id_return_date_error">
-              {% for error in form.return_date.errors %}<li>{{ error }}</li>{% endfor %}
-            </ul>
-          {% endif %}
-        </div>
-      </fieldset>
-      <button type="submit">게시 정보 확인</button>
-    </form>
-  </main>
-  <footer>
-    <p>여행 목적과 날짜별 적용 여부는 공식 기관에서 다시 확인해 주세요.</p>
-  </footer>
-</body>
-</html>
+{% block footer %}
+  <p>여행 목적과 날짜별 적용 여부는 공식 기관에서 다시 확인해 주세요.</p>
+{% endblock %}
diff --git a/public_web/templates/public_web/partials/entry_card.html b/public_web/templates/public_web/partials/entry_card.html
new file mode 100644
index 0000000..3ca93c0
--- /dev/null
+++ b/public_web/templates/public_web/partials/entry_card.html
@@ -0,0 +1,30 @@
+<article id="entry-card" data-state="{{ entry_card.state }}" class="publication-card"
+         tabindex="0" aria-labelledby="entry-heading"
+         aria-describedby="entry-status entry-message">
+  <h2 id="entry-heading">{{ entry_card.heading }}</h2>
+  <p class="status-line" id="entry-status" role="status">
+    <span class="status-symbol" aria-hidden="true"></span>
+    <strong>상태: {{ entry_card.status_label }}</strong>
+  </p>
+  <p id="entry-message">{{ entry_card.message }}</p>
+  {% if entry_card.has_publication %}
+    <dl class="fact-list">
+      <dt>국가</dt><dd>{{ entry_card.country_name }}</dd>
+      <dt>일반여권 source 표기</dt><dd>{{ entry_card.period_text }}</dd>
+      <dt>source 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
+      <dt>snapshot date</dt><dd>{{ entry_card.snapshot_date }}</dd>
+      <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
+      <dt>publication revision</dt><dd>generation {{ entry_card.generation }}</dd>
+      <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
+      <dt>source revision</dt><dd>{{ entry_card.source_revision }}</dd>
+      <dt>출처</dt>
+      <dd>
+        {{ entry_card.source_owner }} · {{ entry_card.attribution }}
+        <a class="source-link" href="{{ entry_card.source_locator }}"
+           rel="noopener noreferrer"
+           aria-label="외교부 입국요건 source 열기">공식 source</a>
+      </dd>
+    </dl>
+    <p class="verification-note"><strong>확인 필요:</strong> 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.</p>
+  {% endif %}
+</article>
diff --git a/public_web/templates/public_web/partials/warning_card.html b/public_web/templates/public_web/partials/warning_card.html
new file mode 100644
index 0000000..af60d27
--- /dev/null
+++ b/public_web/templates/public_web/partials/warning_card.html
@@ -0,0 +1,31 @@
+<article id="warning-card" data-state="{{ warning_card.state }}" class="publication-card"
+         tabindex="0" aria-labelledby="warning-heading"
+         aria-describedby="warning-status warning-message">
+  <h2 id="warning-heading">{{ warning_card.heading }}</h2>
+  <p class="status-line" id="warning-status" role="status">
+    <span class="status-symbol" aria-hidden="true"></span>
+    <strong>상태: {{ warning_card.status_label }}</strong>
+  </p>
+  <p id="warning-message">{{ warning_card.message }}</p>
+  {% if warning_card.has_publication %}
+    <dl class="fact-list">
+      <dt>국가</dt><dd>{{ warning_card.country_name }}</dd>
+      <dt>source 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
+      <dt>source 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
+      <dt>source 범위</dt><dd>{{ warning_card.scope_text }}</dd>
+      <dt>source 작성일</dt><dd>{{ warning_card.written_date|default:"source가 제공하지 않음" }}</dd>
+      <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
+      <dt>publication revision</dt><dd>generation {{ warning_card.generation }}</dd>
+      <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
+      <dt>source revision</dt><dd>{{ warning_card.source_revision }}</dd>
+      <dt>출처</dt>
+      <dd>
+        {{ warning_card.source_owner }} · {{ warning_card.attribution }}
+        <a class="source-link" href="{{ warning_card.source_locator }}"
+           rel="noopener noreferrer"
+           aria-label="외교부 여행경보 source 열기">공식 source</a>
+      </dd>
+    </dl>
+    <p class="verification-note"><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.</p>
+  {% endif %}
+</article>
diff --git a/public_web/templates/public_web/results.html b/public_web/templates/public_web/results.html
index 4a90ddd..a636e6f 100644
--- a/public_web/templates/public_web/results.html
+++ b/public_web/templates/public_web/results.html
@@ -1,79 +1,28 @@
-<!doctype html>
-<html lang="ko">
-<head>
-  <meta charset="utf-8">
-  <meta name="viewport" content="width=device-width, initial-scale=1">
-  <title>여행준비 — 일본 게시 정보</title>
-</head>
-<body>
-  <header>
-    <nav aria-label="주요 메뉴">
-      <a href="{% url 'public_web:index' %}">다시 입력</a>
-      <a href="{% url 'public_web:results' %}" aria-current="page">게시 정보</a>
-    </nav>
-  </header>
-  <main id="main-content">
+{% extends "public_web/base.html" %}
+
+{% block title %}여행준비 — 일본 게시 정보{% endblock %}
+
+{% block nav_links %}
+  <a href="{% url 'public_web:index' %}">다시 입력</a>
+  <a href="{% url 'public_web:results' %}" aria-current="page">게시 정보</a>
+{% endblock %}
+
+{% block main %}
+  <div class="page-heading">
+    <p class="eyebrow">두 개의 독립 publication</p>
     <h1>일본 게시 정보</h1>
-    <p>
+    <p class="page-lead">
       고정된 일본 publication만 표시합니다. 여행 목적과 날짜에 대한 적용 여부는
       계산하지 않으므로 공식 기관 확인이 필요합니다.
     </p>
+  </div>
 
-    <section aria-label="독립 publication 결과">
-      <article id="entry-card" data-state="{{ entry_card.state }}" aria-labelledby="entry-heading">
-        <h2 id="entry-heading">{{ entry_card.heading }}</h2>
-        <p role="status"><strong>상태: {{ entry_card.status_label }}</strong></p>
-        <p>{{ entry_card.message }}</p>
-        {% if entry_card.has_publication %}
-          <dl>
-            <dt>국가</dt><dd>{{ entry_card.country_name }}</dd>
-            <dt>일반여권 source 표기</dt><dd>{{ entry_card.period_text }}</dd>
-            <dt>source 근거 문구</dt><dd>{{ entry_card.basis_text }}</dd>
-            <dt>snapshot date</dt><dd>{{ entry_card.snapshot_date }}</dd>
-            <dt>마지막 성공 확인시각</dt><dd>{{ entry_card.checked_at|default:"확인 필요" }}</dd>
-            <dt>publication revision</dt><dd>generation {{ entry_card.generation }}</dd>
-            <dt>게시시각</dt><dd>{{ entry_card.published_at }}</dd>
-            <dt>source revision</dt><dd>{{ entry_card.source_revision }}</dd>
-            <dt>출처</dt>
-            <dd>
-              {{ entry_card.source_owner }} · {{ entry_card.attribution }}
-              <a href="{{ entry_card.source_locator }}" rel="noopener noreferrer"
-                 aria-label="외교부 입국요건 source 열기">공식 source</a>
-            </dd>
-          </dl>
-          <p><strong>확인 필요:</strong> 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.</p>
-        {% endif %}
-      </article>
+  <section class="publication-grid" aria-label="독립 publication 결과">
+    {% include "public_web/partials/entry_card.html" %}
+    {% include "public_web/partials/warning_card.html" %}
+  </section>
+{% endblock %}
 
-      <article id="warning-card" data-state="{{ warning_card.state }}" aria-labelledby="warning-heading">
-        <h2 id="warning-heading">{{ warning_card.heading }}</h2>
-        <p role="status"><strong>상태: {{ warning_card.status_label }}</strong></p>
-        <p>{{ warning_card.message }}</p>
-        {% if warning_card.has_publication %}
-          <dl>
-            <dt>국가</dt><dd>{{ warning_card.country_name }}</dd>
-            <dt>source 경보 단계 코드</dt><dd>{{ warning_card.alarm_level_code }}</dd>
-            <dt>source 범위 유형</dt><dd>{{ warning_card.scope_type }}</dd>
-            <dt>source 범위</dt><dd>{{ warning_card.scope_text }}</dd>
-            <dt>source 작성일</dt><dd>{{ warning_card.written_date|default:"source가 제공하지 않음" }}</dd>
-            <dt>마지막 성공 확인시각</dt><dd>{{ warning_card.checked_at|default:"확인 필요" }}</dd>
-            <dt>publication revision</dt><dd>generation {{ warning_card.generation }}</dd>
-            <dt>게시시각</dt><dd>{{ warning_card.published_at }}</dd>
-            <dt>source revision</dt><dd>{{ warning_card.source_revision }}</dd>
-            <dt>출처</dt>
-            <dd>
-              {{ warning_card.source_owner }} · {{ warning_card.attribution }}
-              <a href="{{ warning_card.source_locator }}" rel="noopener noreferrer"
-                 aria-label="외교부 여행경보 source 열기">공식 source</a>
-            </dd>
-          </dl>
-          <p><strong>확인 필요:</strong> 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.</p>
-        {% endif %}
-      </article>
-    </section>
-  </main>
-  <footer>
-    <p>두 카드는 서로 독립된 검수·게시 경계를 사용합니다.</p>
-  </footer>
-</body>
-</html>
+{% block footer %}
+  <p>두 카드는 서로 독립된 검수·게시 경계를 사용합니다.</p>
+{% endblock %}
diff --git a/public_web/tests/test_accessibility_contract.py b/public_web/tests/test_accessibility_contract.py
new file mode 100644
index 0000000..6c77433
--- /dev/null
+++ b/public_web/tests/test_accessibility_contract.py
@@ -0,0 +1,148 @@
+from pathlib import Path
+from unittest.mock import patch
+
+from django.conf import settings
+from django.test import SimpleTestCase, override_settings
+from django.urls import reverse
+
+
+@override_settings(SECURE_SSL_REDIRECT=False, ALLOWED_HOSTS=["testserver"])
+class AccessiblePresentationContractTests(SimpleTestCase):
+    def setUp(self):
+        static_root = Path(settings.BASE_DIR) / "public_web" / "static" / "public_web"
+        self.css = (static_root / "site.css").read_text(encoding="utf-8")
+        self.javascript = (static_root / "site.js").read_text(encoding="utf-8")
+
+    def valid_data(self, **overrides):
+        data = {
+            "destination": "JP",
+            "departure_date": "2026-09-10",
+            "return_date": "2026-09-10",
+        }
+        data.update(overrides)
+        return data
+
+    def test_index_has_one_heading_landmarks_skip_link_and_native_labels(self):
+        response = self.client.get(reverse("public_web:index"))
+        body = response.content.decode("utf-8")
+
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(body.count("<h1"), 1)
+        self.assertContains(response, 'class="skip-link" href="#main-content"')
+        self.assertContains(response, "<header", html=False)
+        self.assertContains(response, 'nav class="site-nav" aria-label="주요 메뉴"')
+        self.assertContains(response, 'main id="main-content"')
+        self.assertContains(response, "<footer", html=False)
+        self.assertContains(response, 'for="id_destination"')
+        self.assertContains(response, 'for="id_departure_date"')
+        self.assertContains(response, 'for="id_return_date"')
+        self.assertLess(body.index("skip-link"), body.index("<header"))
+        self.assertLess(body.index("<header"), body.index("<main"))
+        self.assertLess(body.index("<main"), body.index("<footer"))
+
+    def test_invalid_form_has_focusable_linked_error_summary(self):
+        response = self.client.post(
+            reverse("public_web:index"),
+            self.valid_data(return_date="2026-09-09"),
+        )
+
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(response, 'role="alert" aria-live="assertive"')
+        self.assertContains(response, 'tabindex="-1"')
+        self.assertContains(response, "data-error-summary")
+        self.assertContains(response, 'href="#id_return_date"')
+        self.assertContains(response, 'aria-invalid="true"')
+        self.assertContains(
+            response,
+            'aria-describedby="id_return_date_hint id_return_date_error"',
+        )
+        self.assertIn('querySelector("[data-error-summary]")', self.javascript)
+        self.assertIn("errorSummary.focus()", self.javascript)
+
+    def test_submit_state_is_textual_progressive_enhancement(self):
+        response = self.client.get(reverse("public_web:index"))
+
+        self.assertContains(response, "data-trip-form")
+        self.assertContains(response, "data-submit-button")
+        self.assertContains(response, "data-submit-label")
+        self.assertContains(response, "data-submit-status")
+        self.assertContains(response, 'aria-live="polite" aria-atomic="true"')
+        self.assertIn('setAttribute("aria-busy", "true")', self.javascript)
+        self.assertIn('setAttribute("aria-disabled", "true")', self.javascript)
+        self.assertIn(
+            'status.textContent = "게시 정보를 불러오는 중…"',
+            self.javascript,
+        )
+        for forbidden in (
+            "fetch(",
+            "XMLHttpRequest",
+            "localStorage",
+            "sessionStorage",
+            "FormData",
+            "console.",
+        ):
+            self.assertNotIn(forbidden, self.javascript)
+
+    def test_result_cards_keep_independent_markers_text_and_focus_order(self):
+        cards = {
+            "entry": {
+                "state": "ready",
+                "heading": "입국요건 사실",
+                "status_label": "게시된 source 사실",
+                "message": "공식 source의 검수·게시 사실입니다.",
+                "has_publication": False,
+            },
+            "warning": {
+                "state": "stale",
+                "heading": "여행경보",
+                "status_label": "재확인 필요",
+                "message": "마지막 검수·게시 사실입니다.",
+                "has_publication": False,
+            },
+        }
+
+        with patch(
+            "public_web.results._safe_card",
+            side_effect=lambda module, loader: cards[module],
+        ):
+            response = self.client.get(reverse("public_web:results"))
+
+        self.assertEqual(response.status_code, 200)
+        body = response.content.decode("utf-8")
+        self.assertEqual(body.count("<h1"), 1)
+        self.assertEqual(body.count("<h2"), 2)
+        self.assertContains(response, 'id="entry-card" data-state="ready"')
+        self.assertContains(response, 'id="warning-card" data-state="stale"')
+        self.assertContains(response, 'tabindex="0"', count=2)
+        self.assertContains(
+            response,
+            'aria-describedby="entry-status entry-message"',
+        )
+        self.assertContains(
+            response,
+            'aria-describedby="warning-status warning-message"',
+        )
+        self.assertContains(response, "상태: 게시된 source 사실")
+        self.assertContains(response, "상태: 재확인 필요")
+        self.assertContains(response, 'aria-hidden="true"', count=2)
+
+    def test_styles_encode_responsive_touch_focus_and_wrapping_contracts(self):
+        for required in (
+            "box-sizing: border-box",
+            "min-height: 44px",
+            "min-height: 48px",
+            "focus-visible",
+            "overflow-wrap: anywhere",
+            "word-break: keep-all",
+            "grid-template-columns: minmax(0, 1fr)",
+            "repeat(2, minmax(0, 1fr))",
+            "@media (max-width: 30rem)",
+            "@media (pointer: coarse)",
+            "@media (forced-colors: active)",
+            ".site-header a:focus-visible",
+            "outline-color: white",
+            "box-shadow: 0 0 0 0.4375rem #000000",
+        ):
+            self.assertIn(required, self.css)
+        for forbidden in ("@import", "url(http://", "url(https://"):
+            self.assertNotIn(forbidden, self.css)


