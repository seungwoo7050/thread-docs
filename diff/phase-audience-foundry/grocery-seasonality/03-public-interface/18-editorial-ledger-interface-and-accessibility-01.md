# 편집형 원장 인터페이스와 접근성

## `feat(public): add responsive ssr templates`

diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
new file mode 100644
index 0000000..cff63ea
--- /dev/null
+++ b/grocery/static/grocery/app.css
@@ -0,0 +1,763 @@
+:root {
+  color-scheme: light;
+  --color-canvas: #f6f7f2;
+  --color-surface: #ffffff;
+  --color-text: #17201a;
+  --color-muted: #536057;
+  --color-border: #c7cec8;
+  --color-brand: #175b3a;
+  --color-brand-strong: #0d442a;
+  --color-brand-soft: #e8f2ec;
+  --color-info: #165273;
+  --color-info-soft: #eaf4f8;
+  --color-warning: #745000;
+  --color-warning-soft: #fff4d6;
+  --color-error: #8b1e24;
+  --color-error-soft: #fff0f0;
+  --color-neutral-soft: #eef1ee;
+  --focus-ring: #0969da;
+  --shadow-card: 0 0.2rem 0.8rem rgb(23 32 26 / 8%);
+  --radius-small: 0.45rem;
+  --radius-medium: 0.8rem;
+  --page-gutter: clamp(1rem, 4vw, 2rem);
+  --measure: 44rem;
+}
+
+*,
+*::before,
+*::after {
+  box-sizing: border-box;
+}
+
+html {
+  min-width: 0;
+  background: var(--color-canvas);
+  color: var(--color-text);
+  font-family:
+    system-ui,
+    -apple-system,
+    BlinkMacSystemFont,
+    "Apple SD Gothic Neo",
+    "Noto Sans KR",
+    sans-serif;
+  font-size: 100%;
+  line-height: 1.6;
+  overflow-wrap: anywhere;
+  word-break: keep-all;
+}
+
+body {
+  min-width: 0;
+  min-height: 100vh;
+  margin: 0;
+}
+
+img,
+svg {
+  display: block;
+  max-width: 100%;
+}
+
+a {
+  color: var(--color-brand-strong);
+  text-underline-offset: 0.18em;
+  text-decoration-thickness: 0.08em;
+}
+
+a:hover {
+  text-decoration-thickness: 0.14em;
+}
+
+:focus-visible {
+  outline: 3px solid var(--focus-ring);
+  outline-offset: 3px;
+}
+
+h1,
+h2,
+h3,
+p,
+dl,
+dd {
+  margin-top: 0;
+}
+
+h1,
+h2,
+h3 {
+  line-height: 1.25;
+  text-wrap: balance;
+}
+
+h1 {
+  max-width: 22ch;
+  margin-bottom: 0.75rem;
+  font-size: clamp(2rem, 7vw, 3.75rem);
+  letter-spacing: -0.04em;
+}
+
+h2 {
+  margin-bottom: 0.75rem;
+  font-size: clamp(1.35rem, 4vw, 1.8rem);
+  letter-spacing: -0.025em;
+}
+
+h3 {
+  margin-bottom: 0.5rem;
+  font-size: 1.15rem;
+}
+
+.page-shell {
+  width: min(100%, 76rem);
+  min-width: 0;
+  margin-inline: auto;
+  padding-inline: var(--page-gutter);
+}
+
+.page-main {
+  min-height: 65vh;
+  padding-block: clamp(2rem, 7vw, 5rem);
+}
+
+.skip-link {
+  position: fixed;
+  z-index: 100;
+  top: 0.75rem;
+  left: 0.75rem;
+  min-height: 2.75rem;
+  padding: 0.65rem 1rem;
+  border-radius: var(--radius-small);
+  background: var(--color-text);
+  color: #fff;
+  transform: translateY(-200%);
+}
+
+.skip-link:focus {
+  transform: translateY(0);
+}
+
+.site-header {
+  border-bottom: 1px solid var(--color-border);
+  background: var(--color-surface);
+}
+
+.site-header__inner {
+  display: flex;
+  align-items: center;
+  min-height: 5rem;
+}
+
+.brand {
+  display: inline-flex;
+  min-width: 0;
+  min-height: 2.75rem;
+  flex-direction: column;
+  justify-content: center;
+  color: var(--color-text);
+  text-decoration: none;
+}
+
+.brand__name {
+  font-size: 1.08rem;
+  font-weight: 800;
+  letter-spacing: -0.02em;
+}
+
+.brand__description {
+  color: var(--color-muted);
+  font-size: 0.85rem;
+}
+
+.site-footer {
+  border-top: 1px solid var(--color-border);
+  background: var(--color-surface);
+  color: var(--color-muted);
+  font-size: 0.9rem;
+}
+
+.site-footer__inner {
+  padding-block: 1.75rem;
+}
+
+.site-footer p:last-child {
+  margin-bottom: 0;
+}
+
+.eyebrow {
+  margin-bottom: 0.5rem;
+  color: var(--color-brand-strong);
+  font-size: 0.83rem;
+  font-weight: 800;
+  letter-spacing: 0.04em;
+}
+
+.page-heading {
+  max-width: var(--measure);
+  margin-bottom: clamp(2rem, 5vw, 3.5rem);
+}
+
+.page-heading__summary {
+  margin-bottom: 0;
+  color: var(--color-muted);
+  font-size: clamp(1rem, 2.5vw, 1.2rem);
+}
+
+.identity-summary {
+  display: flex;
+  flex-wrap: wrap;
+  gap: 0.5rem 1rem;
+}
+
+.identity-summary span {
+  min-width: 0;
+}
+
+.search-panel,
+.identity-panel,
+.provenance,
+.current-price,
+.error-page {
+  min-width: 0;
+  border: 1px solid var(--color-border);
+  border-radius: var(--radius-medium);
+  background: var(--color-surface);
+  box-shadow: var(--shadow-card);
+}
+
+.search-panel,
+.identity-panel,
+.provenance {
+  padding: clamp(1rem, 4vw, 2rem);
+}
+
+.search-panel {
+  margin-bottom: 2rem;
+}
+
+.search-form {
+  display: grid;
+  min-width: 0;
+  gap: 0.75rem;
+}
+
+.field-group {
+  min-width: 0;
+}
+
+.field-group label {
+  display: block;
+  margin-bottom: 0.2rem;
+  font-weight: 750;
+}
+
+.field-hint {
+  margin-bottom: 0.45rem;
+  color: var(--color-muted);
+  font-size: 0.9rem;
+}
+
+input[type="search"] {
+  width: 100%;
+  min-width: 0;
+  min-height: 2.75rem;
+  padding: 0.68rem 0.8rem;
+  border: 2px solid #667269;
+  border-radius: var(--radius-small);
+  background: var(--color-surface);
+  color: var(--color-text);
+  font: inherit;
+}
+
+input[aria-invalid="true"] {
+  border-color: var(--color-error);
+}
+
+.form-error {
+  display: flex;
+  align-items: flex-start;
+  gap: 0.6rem;
+  margin-bottom: 1rem;
+  padding: 0.8rem;
+  border-left: 0.3rem solid var(--color-error);
+  background: var(--color-error-soft);
+  color: #68141a;
+}
+
+.button,
+.chip {
+  display: inline-flex;
+  min-height: 2.75rem;
+  align-items: center;
+  justify-content: center;
+  padding: 0.62rem 1rem;
+  border: 2px solid transparent;
+  border-radius: var(--radius-small);
+  font: inherit;
+  font-weight: 750;
+  line-height: 1.2;
+  text-align: center;
+  text-decoration: none;
+  cursor: pointer;
+}
+
+.button--primary {
+  background: var(--color-brand-strong);
+  color: #fff;
+}
+
+.button--secondary {
+  margin-top: 0.45rem;
+  border-color: var(--color-brand-strong);
+  background: var(--color-surface);
+  color: var(--color-brand-strong);
+}
+
+.category-nav {
+  margin-top: 1.25rem;
+  padding-top: 1.25rem;
+  border-top: 1px solid var(--color-border);
+}
+
+.chip-list,
+.result-list,
+.comparison-list,
+.breadcrumb ol {
+  margin: 0;
+  padding: 0;
+  list-style: none;
+}
+
+.chip-list {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: 0.5rem;
+}
+
+.chip {
+  border-color: var(--color-border);
+  background: var(--color-surface);
+  color: var(--color-text);
+}
+
+.chip--selected {
+  border-color: var(--color-brand-strong);
+  background: var(--color-brand-soft);
+}
+
+.state-notice {
+  display: grid;
+  min-width: 0;
+  grid-template-columns: auto minmax(0, 1fr);
+  gap: 0.8rem;
+  margin-block: 1.5rem;
+  padding: clamp(1rem, 4vw, 1.5rem);
+  border: 1px solid currentcolor;
+  border-left-width: 0.35rem;
+  border-radius: var(--radius-medium);
+}
+
+.state-notice p:last-child,
+.state-notice__title {
+  margin-bottom: 0;
+}
+
+.state-notice__symbol {
+  display: inline-grid;
+  width: 1.75rem;
+  height: 1.75rem;
+  place-items: center;
+  border: 2px solid currentcolor;
+  border-radius: 50%;
+  font-weight: 900;
+  line-height: 1;
+}
+
+.state-notice--info {
+  background: var(--color-info-soft);
+  color: var(--color-info);
+}
+
+.state-notice--warning {
+  background: var(--color-warning-soft);
+  color: var(--color-warning);
+}
+
+.state-notice--error {
+  background: var(--color-error-soft);
+  color: var(--color-error);
+}
+
+.state-notice--neutral {
+  background: var(--color-neutral-soft);
+  color: var(--color-text);
+}
+
+.section-heading {
+  display: flex;
+  min-width: 0;
+  align-items: baseline;
+  justify-content: space-between;
+  gap: 1rem;
+  margin-bottom: 1rem;
+}
+
+.section-heading h2,
+.section-heading p {
+  margin-bottom: 0;
+}
+
+.section-heading--stacked {
+  display: block;
+}
+
+.section-heading--stacked p {
+  max-width: var(--measure);
+  color: var(--color-muted);
+}
+
+.result-count {
+  flex: 0 0 auto;
+  color: var(--color-muted);
+}
+
+.result-list {
+  display: grid;
+  min-width: 0;
+  gap: 1rem;
+}
+
+.result-card,
+.result-card__link,
+.result-card__heading,
+.result-card__facts,
+.result-card__facts div {
+  min-width: 0;
+}
+
+.result-card {
+  border: 1px solid var(--color-border);
+  border-radius: var(--radius-medium);
+  background: var(--color-surface);
+  box-shadow: var(--shadow-card);
+}
+
+.result-card__link {
+  display: grid;
+  min-height: 2.75rem;
+  gap: 1rem;
+  padding: clamp(1rem, 4vw, 1.5rem);
+  color: inherit;
+  text-decoration: none;
+}
+
+.result-card__link:hover {
+  border-radius: inherit;
+  background: var(--color-brand-soft);
+}
+
+.result-card__category {
+  margin-bottom: 0.25rem;
+  color: var(--color-brand-strong);
+  font-size: 0.85rem;
+  font-weight: 800;
+}
+
+.result-card__identity {
+  margin-bottom: 0;
+  color: var(--color-muted);
+}
+
+.result-card__facts {
+  display: grid;
+  grid-template-columns: repeat(auto-fit, minmax(min(100%, 10rem), 1fr));
+  gap: 0.75rem;
+  margin-bottom: 0;
+}
+
+.result-card__facts div {
+  padding-top: 0.75rem;
+  border-top: 1px solid var(--color-border);
+}
+
+.result-card__facts dt,
+.definition-grid dt {
+  color: var(--color-muted);
+  font-size: 0.83rem;
+  font-weight: 700;
+}
+
+.result-card__facts dd,
+.definition-grid dd {
+  margin-bottom: 0;
+  font-weight: 720;
+}
+
+.result-card__action {
+  justify-self: start;
+  color: var(--color-brand-strong);
+  font-weight: 800;
+}
+
+.status-text {
+  display: inline-flex;
+  min-width: 0;
+  align-items: baseline;
+  gap: 0.35rem;
+  font-weight: 750;
+}
+
+.status-text--current {
+  color: var(--color-brand-strong);
+}
+
+.status-text--stale {
+  color: var(--color-warning);
+}
+
+.status-text--unavailable {
+  color: var(--color-muted);
+}
+
+.breadcrumb {
+  margin-bottom: 1.5rem;
+}
+
+.breadcrumb ol {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  align-items: center;
+  gap: 0.35rem;
+  color: var(--color-muted);
+  font-size: 0.9rem;
+}
+
+.breadcrumb li {
+  min-width: 0;
+}
+
+.breadcrumb li + li::before {
+  padding-right: 0.35rem;
+  content: "/";
+}
+
+.breadcrumb a {
+  display: inline-flex;
+  min-height: 2.75rem;
+  align-items: center;
+}
+
+.current-price {
+  display: grid;
+  min-width: 0;
+  gap: 0.5rem;
+  margin-bottom: 1.25rem;
+  padding: clamp(1.25rem, 5vw, 2.25rem);
+  border-color: #8fb39e;
+  background: linear-gradient(145deg, var(--color-surface), var(--color-brand-soft));
+}
+
+.current-price h2 {
+  margin-bottom: 0;
+}
+
+.current-price__value {
+  margin-bottom: 0;
+  font-size: clamp(2rem, 9vw, 3.5rem);
+  font-weight: 850;
+  letter-spacing: -0.04em;
+  line-height: 1.15;
+}
+
+.current-price__date {
+  margin-bottom: 0;
+  color: var(--color-muted);
+}
+
+.identity-panel,
+.comparison-section,
+.provenance {
+  margin-top: 1.25rem;
+}
+
+.definition-grid {
+  display: grid;
+  min-width: 0;
+  grid-template-columns: repeat(auto-fit, minmax(min(100%, 11rem), 1fr));
+  gap: 1rem;
+  margin-bottom: 0;
+}
+
+.definition-grid div {
+  min-width: 0;
+  padding-top: 0.75rem;
+  border-top: 1px solid var(--color-border);
+}
+
+.comparison-section {
+  min-width: 0;
+  padding-block: 1rem;
+}
+
+.comparison-list {
+  display: grid;
+  min-width: 0;
+  grid-template-columns: repeat(auto-fit, minmax(min(100%, 16rem), 1fr));
+  gap: 1rem;
+}
+
+.comparison-card {
+  min-width: 0;
+  padding: 1.25rem;
+  border: 1px solid var(--color-border);
+  border-radius: var(--radius-medium);
+  background: var(--color-surface);
+}
+
+.comparison-card--unavailable {
+  border-style: dashed;
+  background: var(--color-neutral-soft);
+}
+
+.comparison-card__reference {
+  margin-bottom: 0.75rem;
+  color: var(--color-muted);
+}
+
+.comparison-card__reference strong {
+  color: var(--color-text);
+  font-size: 1.12rem;
+}
+
+.direction {
+  display: flex;
+  min-width: 0;
+  align-items: flex-start;
+  gap: 0.45rem;
+  font-weight: 800;
+}
+
+.direction__symbol {
+  flex: 0 0 auto;
+}
+
+.direction--lower {
+  color: #235783;
+}
+
+.direction--higher {
+  color: #8b1e24;
+}
+
+.direction--equal {
+  color: var(--color-text);
+}
+
+.comparison-card__date,
+.comparison-card__reason {
+  color: var(--color-muted);
+  font-size: 0.9rem;
+}
+
+.comparison-card__date:last-child,
+.comparison-card__reason:last-child {
+  margin-bottom: 0;
+}
+
+.definition-grid--provenance {
+  grid-template-columns: repeat(auto-fit, minmax(min(100%, 14rem), 1fr));
+}
+
+.metadata-detail {
+  display: block;
+  color: var(--color-muted);
+  font-size: 0.88rem;
+  font-weight: 500;
+}
+
+.provenance a {
+  display: inline-flex;
+  min-height: 2.75rem;
+  align-items: center;
+}
+
+.provenance__note {
+  max-width: var(--measure);
+  margin: 1.5rem 0 0;
+  padding-top: 1rem;
+  border-top: 1px solid var(--color-border);
+  color: var(--color-muted);
+}
+
+.error-page {
+  max-width: 42rem;
+  padding: clamp(1.25rem, 5vw, 3rem);
+}
+
+.error-page p:not(.eyebrow) {
+  max-width: var(--measure);
+  color: var(--color-muted);
+}
+
+@media (min-width: 40rem) {
+  .search-form {
+    grid-template-columns: minmax(0, 1fr) auto;
+    align-items: end;
+  }
+
+  .search-form .button {
+    min-width: 7rem;
+  }
+
+  .result-card__link {
+    grid-template-columns: minmax(12rem, 1fr) minmax(18rem, 1.25fr);
+    align-items: center;
+  }
+
+  .result-card__action {
+    grid-column: 1 / -1;
+  }
+
+  .current-price {
+    grid-template-columns: minmax(0, 1fr) auto;
+    align-items: end;
+  }
+
+  .current-price__value {
+    grid-row: 1 / 3;
+    grid-column: 2;
+    text-align: right;
+  }
+}
+
+@media (min-width: 64rem) {
+  .result-card__link {
+    grid-template-columns: minmax(15rem, 1fr) minmax(25rem, 1.5fr) auto;
+  }
+
+  .result-card__action {
+    grid-row: auto;
+    grid-column: auto;
+    justify-self: end;
+  }
+}
+
+@media (prefers-reduced-motion: reduce) {
+  *,
+  *::before,
+  *::after {
+    scroll-behavior: auto !important;
+    transition-duration: 0.01ms !important;
+  }
+}
+
+@media (forced-colors: active) {
+  .button,
+  .chip,
+  .state-notice,
+  .result-card,
+  .comparison-card {
+    border-color: CanvasText;
+  }
+}
diff --git a/grocery/templates/grocery/_state_notice.html b/grocery/templates/grocery/_state_notice.html
new file mode 100644
index 0000000..4fc6008
--- /dev/null
+++ b/grocery/templates/grocery/_state_notice.html
@@ -0,0 +1,44 @@
+{% if state == "loading" %}
+  <section class="state-notice state-notice--info" role="status" aria-live="polite" aria-atomic="true">
+    <span class="state-notice__symbol" aria-hidden="true">…</span>
+    <div>
+      <h2 class="state-notice__title">자료를 불러오는 중</h2>
+      <p>{{ state_message|default:"검토되어 공개된 조사값을 확인하고 있습니다." }}</p>
+    </div>
+  </section>
+{% elif state == "empty" %}
+  <section class="state-notice state-notice--neutral" role="status">
+    <span class="state-notice__symbol" aria-hidden="true">○</span>
+    <div>
+      <h2 class="state-notice__title">조건에 맞는 항목 없음</h2>
+      <p>{{ state_message|default:"검색어를 줄이거나 다른 부류를 선택해 보세요." }}</p>
+    </div>
+  </section>
+{% elif state == "unavailable" %}
+  <section class="state-notice state-notice--warning" role="status">
+    <span class="state-notice__symbol" aria-hidden="true">!</span>
+    <div>
+      <h2 class="state-notice__title">공개 조사값 없음</h2>
+      <p>{{ state_message|default:"현재 공개할 수 있는 검토 완료 자료가 없습니다." }}</p>
+    </div>
+  </section>
+{% elif state == "stale" %}
+  <section class="state-notice state-notice--warning" role="status">
+    <span class="state-notice__symbol" aria-hidden="true">!</span>
+    <div>
+      <h2 class="state-notice__title">마지막 검토 자료 표시 중</h2>
+      <p>
+        {{ state_message|default:"새 수집 상태를 확인하고 있습니다. 아래에는 마지막으로 검토해 공개한 조사값을 표시합니다." }}
+      </p>
+    </div>
+  </section>
+{% elif state == "server_error" %}
+  <section class="state-notice state-notice--error" role="alert">
+    <span class="state-notice__symbol" aria-hidden="true">×</span>
+    <div>
+      <h2 class="state-notice__title">자료를 표시하지 못함</h2>
+      <p>{{ state_message|default:"잠시 후 다시 시도해 주세요." }}</p>
+      {% if retry_url %}<a class="button button--secondary" href="{{ retry_url }}">다시 시도</a>{% endif %}
+    </div>
+  </section>
+{% endif %}
diff --git a/grocery/templates/grocery/base.html b/grocery/templates/grocery/base.html
new file mode 100644
index 0000000..c6114bc
--- /dev/null
+++ b/grocery/templates/grocery/base.html
@@ -0,0 +1,41 @@
+{% load static %}
+<!doctype html>
+<html lang="ko">
+  <head>
+    <meta charset="utf-8">
+    <meta name="viewport" content="width=device-width, initial-scale=1">
+    <meta name="color-scheme" content="light">
+    <meta
+      name="description"
+      content="KAMIS가 제공한 채소류·과일류 소매 조사 평균과 비교 제공값을 확인합니다."
+    >
+    <title>{% block title %}농산물 조사값 살펴보기{% endblock %}</title>
+    <link rel="stylesheet" href="{% static 'grocery/app.css' %}">
+  </head>
+  <body>
+    <a class="skip-link" href="#main-content">본문으로 건너뛰기</a>
+
+    <header class="site-header">
+      <div class="page-shell site-header__inner">
+        <a class="brand" href="{{ home_url|default:'/' }}" aria-label="농산물 조사값 살펴보기 홈">
+          <span class="brand__name">농산물 조사값 살펴보기</span>
+          <span class="brand__description">KAMIS가 제공한 소매 조사 자료</span>
+        </a>
+      </div>
+    </header>
+
+    <main id="main-content" class="page-shell page-main" tabindex="-1">
+      {% block content %}{% endblock %}
+    </main>
+
+    <footer class="site-footer">
+      <div class="page-shell site-footer__inner">
+        <p>
+          표시값은 KAMIS가 제공한 소매 조사 평균이며, 개별 판매처의 판매값을 나타내지
+          않습니다.
+        </p>
+        <p>검토 후 공개된 자료만 표시하며 공개 화면에서 외부 source를 직접 호출하지 않습니다.</p>
+      </div>
+    </footer>
+  </body>
+</html>
diff --git a/grocery/templates/grocery/catalog.html b/grocery/templates/grocery/catalog.html
new file mode 100644
index 0000000..ce94ec7
--- /dev/null
+++ b/grocery/templates/grocery/catalog.html
@@ -0,0 +1,121 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}채소·과일 조사값 | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <header class="page-heading">
+    <p class="eyebrow">KAMIS 소매 조사 평균</p>
+    <h1>채소·과일 조사값</h1>
+    <p class="page-heading__summary">
+      공식 품목명으로 찾아보고, 한 품목의 품종·등급·판매 단위가 같은 제공값만 확인할 수
+      있습니다.
+    </p>
+  </header>
+
+  <section class="search-panel" aria-labelledby="catalog-search-heading">
+    <h2 id="catalog-search-heading">품목 찾기</h2>
+    {% if query_error %}
+      <div id="search-error" class="form-error" role="alert">
+        <span aria-hidden="true">!</span>
+        <span><strong>입력 확인:</strong> {{ query_error }}</span>
+      </div>
+    {% endif %}
+    <form class="search-form" action="{{ form_action|default:'' }}" method="get" role="search">
+      <div class="field-group">
+        <label for="catalog-query">공식 품목명</label>
+        <p id="catalog-query-hint" class="field-hint">예: 배추, 사과</p>
+        <input
+          id="catalog-query"
+          name="q"
+          type="search"
+          value="{{ query|default:'' }}"
+          maxlength="80"
+          autocomplete="off"
+          enterkeyhint="search"
+          aria-describedby="catalog-query-hint{% if query_error %} search-error{% endif %}"
+          {% if query_error %}aria-invalid="true"{% endif %}
+        >
+      </div>
+      {% if selected_category %}
+        <input type="hidden" name="category" value="{{ selected_category }}">
+      {% endif %}
+      <button class="button button--primary" type="submit">검색</button>
+    </form>
+
+    {% if categories %}
+      <nav class="category-nav" aria-label="부류 선택">
+        <ul class="chip-list">
+          {% for category in categories %}
+            <li>
+              <a
+                class="chip{% if category.selected %} chip--selected{% endif %}"
+                href="{{ category.url }}"
+                {% if category.selected %}aria-current="page"{% endif %}
+              >{{ category.label }}</a>
+            </li>
+          {% endfor %}
+        </ul>
+      </nav>
+    {% endif %}
+  </section>
+
+  {% if catalog_state == "loading" or catalog_state == "unavailable" or catalog_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=catalog_state state_message=status_message retry_url=retry_url %}
+  {% else %}
+    {% if catalog_state == "stale" %}
+      {% include "grocery/_state_notice.html" with state="stale" state_message=status_message %}
+    {% endif %}
+
+    {% if results %}
+      <section class="catalog-results" aria-labelledby="catalog-results-heading">
+        <div class="section-heading">
+          <h2 id="catalog-results-heading">공개 항목</h2>
+          {% if result_count_label %}<p class="result-count" aria-live="polite">{{ result_count_label }}</p>{% endif %}
+        </div>
+        <ul class="result-list">
+          {% for item in results %}
+            <li>
+              <article class="result-card">
+                <a class="result-card__link" href="{{ item.url }}">
+                  <div class="result-card__heading">
+                    <p class="result-card__category">{{ item.category_label }}</p>
+                    <h3>{{ item.item_name }}</h3>
+                    <p class="result-card__identity">
+                      {{ item.variety_name }} · {{ item.grade_name }} · {{ item.unit_label }}
+                    </p>
+                  </div>
+                  <dl class="result-card__facts">
+                    <div>
+                      <dt>조사일 평균</dt>
+                      <dd>{{ item.current_price_label }}</dd>
+                    </div>
+                    <div>
+                      <dt>조사일</dt>
+                      <dd>
+                        {% if item.source_date_iso %}<time datetime="{{ item.source_date_iso }}">{% endif %}
+                        {{ item.source_date_label }}
+                        {% if item.source_date_iso %}</time>{% endif %}
+                      </dd>
+                    </div>
+                    <div>
+                      <dt>자료 상태</dt>
+                      <dd>
+                        <span class="status-text status-text--{{ item.freshness_state|default:'current' }}">
+                          <span aria-hidden="true">{% if item.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
+                          {{ item.freshness_label }}
+                        </span>
+                      </dd>
+                    </div>
+                  </dl>
+                  <span class="result-card__action" aria-hidden="true">자세히 보기 →</span>
+                </a>
+              </article>
+            </li>
+          {% endfor %}
+        </ul>
+      </section>
+    {% else %}
+      {% include "grocery/_state_notice.html" with state="empty" state_message=status_message %}
+    {% endif %}
+  {% endif %}
+{% endblock %}
diff --git a/grocery/templates/grocery/detail.html b/grocery/templates/grocery/detail.html
new file mode 100644
index 0000000..e020447
--- /dev/null
+++ b/grocery/templates/grocery/detail.html
@@ -0,0 +1,144 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}{{ series.item_name|default:"조사값 상세" }} | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <nav class="breadcrumb" aria-label="현재 위치">
+    <ol>
+      <li><a href="{{ catalog_url|default:'/' }}">채소·과일 조사값</a></li>
+      <li aria-current="page">{{ series.item_name|default:"상세" }}</li>
+    </ol>
+  </nav>
+
+  <header class="page-heading page-heading--detail">
+    <p class="eyebrow">{{ series.category_label|default:"공개 조사 항목" }}</p>
+    <h1>{{ series.item_name|default:"조사값 상세" }}</h1>
+    {% if series.variety_name or series.grade_name or series.unit_label %}
+      <p class="page-heading__summary identity-summary">
+        {% if series.variety_name %}<span>품종 {{ series.variety_name }}</span>{% endif %}
+        {% if series.grade_name %}<span>등급 {{ series.grade_name }}</span>{% endif %}
+        {% if series.unit_label %}<span>판매 단위 {{ series.unit_label }}</span>{% endif %}
+      </p>
+    {% endif %}
+  </header>
+
+  {% if detail_state == "loading" or detail_state == "unavailable" or detail_state == "server_error" %}
+    {% include "grocery/_state_notice.html" with state=detail_state state_message=status_message retry_url=retry_url %}
+  {% else %}
+    {% if detail_state == "stale" %}
+      {% include "grocery/_state_notice.html" with state="stale" state_message=status_message %}
+    {% endif %}
+
+    <section class="current-price" aria-labelledby="current-price-heading">
+      <div>
+        <p class="eyebrow">KAMIS 소매 조사 평균</p>
+        <h2 id="current-price-heading">조사일 평균</h2>
+      </div>
+      <p class="current-price__value">
+        {% if series.current_price_machine %}<data value="{{ series.current_price_machine }}">{% endif %}
+        {{ series.current_price_label }}
+        {% if series.current_price_machine %}</data>{% endif %}
+      </p>
+      <p class="current-price__date">
+        조사일
+        {% if provenance.source_date_iso %}<time datetime="{{ provenance.source_date_iso }}">{% endif %}
+        {{ provenance.source_date_label }}
+        {% if provenance.source_date_iso %}</time>{% endif %}
+      </p>
+    </section>
+
+    <section class="identity-panel" aria-labelledby="identity-heading">
+      <h2 id="identity-heading">비교 대상의 정확한 조건</h2>
+      <dl class="definition-grid">
+        <div><dt>품목</dt><dd>{{ series.item_name }}</dd></div>
+        <div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>
+        <div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>
+        <div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>
+        <div><dt>조사범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
+      </dl>
+    </section>
+
+    <section class="comparison-section" aria-labelledby="comparison-heading">
+      <div class="section-heading section-heading--stacked">
+        <h2 id="comparison-heading">비교 제공값</h2>
+        <p>source가 같은 series에 제공한 값과 현재 조사 평균의 산술 차이입니다.</p>
+      </div>
+      <ul class="comparison-list">
+        {% for comparison in comparisons %}
+          <li class="comparison-card{% if not comparison.available %} comparison-card--unavailable{% endif %}">
+            <h3>{{ comparison.period_label }}</h3>
+            {% if comparison.available %}
+              <p class="comparison-card__reference">
+                제공값 <strong>{{ comparison.reference_value_label }}</strong>
+              </p>
+              <p class="direction direction--{{ comparison.direction_code|lower }}">
+                <span class="direction__symbol" aria-hidden="true">
+                  {% if comparison.direction_code == "LOWER" %}↓{% elif comparison.direction_code == "HIGHER" %}↑{% else %}＝{% endif %}
+                </span>
+                <span>
+                  {{ comparison.difference_label }} {{ comparison.direction_label }}
+                  ({{ comparison.percentage_label }})
+                </span>
+              </p>
+            {% else %}
+              <p class="status-text status-text--unavailable">
+                <span aria-hidden="true">○</span> 비교 정보 없음
+              </p>
+              {% if comparison.unavailable_reason_label %}
+                <p class="comparison-card__reason">{{ comparison.unavailable_reason_label }}</p>
+              {% endif %}
+            {% endif %}
+            <p class="comparison-card__date">
+              {% if comparison.reference_date_available %}
+                비교 기준일
+                {% if comparison.reference_date_iso %}<time datetime="{{ comparison.reference_date_iso }}">{% endif %}
+                {{ comparison.reference_date_label }}
+                {% if comparison.reference_date_iso %}</time>{% endif %}
+              {% else %}
+                source가 비교 기준일을 별도로 제공하지 않음
+              {% endif %}
+            </p>
+          </li>
+        {% empty %}
+          <li class="comparison-card comparison-card--unavailable">
+            <p class="status-text status-text--unavailable"><span aria-hidden="true">○</span> 비교 정보 없음</p>
+          </li>
+        {% endfor %}
+      </ul>
+    </section>
+
+    <aside class="provenance" aria-labelledby="provenance-heading">
+      <h2 id="provenance-heading">출처와 자료 상태</h2>
+      <dl class="definition-grid definition-grid--provenance">
+        <div>
+          <dt>출처</dt>
+          <dd>
+            {% if provenance.source_url %}
+              <a href="{{ provenance.source_url }}" rel="external noreferrer">{{ provenance.source_name }}</a>
+            {% else %}
+              {{ provenance.source_name }}
+            {% endif %}
+            {% if provenance.dataset_id %}<span class="metadata-detail">데이터셋 {{ provenance.dataset_id }}</span>{% endif %}
+          </dd>
+        </div>
+        <div><dt>조사일</dt><dd>{{ provenance.source_date_label }}</dd></div>
+        <div><dt>조사범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
+        <div><dt>마지막 확인</dt><dd>{{ provenance.checked_at_label }}</dd></div>
+        <div><dt>공개 검토일</dt><dd>{{ provenance.reviewed_at_label }}</dd></div>
+        <div>
+          <dt>자료 상태</dt>
+          <dd>
+            <span class="status-text status-text--{{ provenance.freshness_state|default:'current' }}">
+              <span aria-hidden="true">{% if provenance.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
+              {{ provenance.freshness_label }}
+            </span>
+          </dd>
+        </div>
+      </dl>
+      <p class="provenance__note">
+        명시된 품목·품종·등급·판매 단위·조사범위가 모두 같은 값만 비교합니다. source가
+        제공하지 않은 비교 기준일은 추정하지 않습니다.
+      </p>
+    </aside>
+  {% endif %}
+{% endblock %}
diff --git a/grocery/tests/test_public_templates.py b/grocery/tests/test_public_templates.py
new file mode 100644
index 0000000..c882d99
--- /dev/null
+++ b/grocery/tests/test_public_templates.py
@@ -0,0 +1,245 @@
+from collections.abc import Mapping
+from pathlib import Path
+from typing import Any
+
+import pytest
+from django.conf import settings
+from django.template.loader import render_to_string
+
+FORBIDDEN_PUBLIC_PHRASES = (
+    "제철",
+    "평년",
+    "저렴",
+    "비싸",
+    "가성비",
+    "사세요",
+    "추천",
+    "최저가",
+    "시장 최저",
+    "전국 평균",
+    "마트 가격",
+    "실시간",
+    "예측",
+    "절약액",
+    "품절",
+    "판매 종료",
+    "비제철",
+)
+
+
+def render(template_name: str, context: Mapping[str, Any] | None = None) -> str:
+    return render_to_string(template_name, context or {})
+
+
+def catalog_context(**overrides: object) -> dict[str, object]:
+    context: dict[str, object] = {
+        "home_url": "/",
+        "form_action": "/",
+        "catalog_state": "ready",
+        "query": "배추",
+        "selected_category": "vegetable",
+        "categories": [
+            {"label": "전체", "url": "/", "selected": False},
+            {"label": "채소류", "url": "/?category=vegetable", "selected": True},
+            {"label": "과일류", "url": "/?category=fruit", "selected": False},
+        ],
+        "result_count_label": "공개 항목 1개",
+        "results": [
+            {
+                "url": "/series/200-212-00-04/",
+                "category_label": "채소류",
+                "item_name": "아주긴한국어공식품목명이줄바꿈되어야하는배추",
+                "variety_name": "아주긴한국어공식품종명이잘리지않아야하는품종",
+                "grade_name": "상품",
+                "unit_label": "아주긴원문판매단위표시 포기 × 1",
+                "current_price_label": "8,000원",
+                "source_date_iso": "2026-08-29",
+                "source_date_label": "2026년 8월 29일",
+                "freshness_state": "current",
+                "freshness_label": "공개 조사일 확인됨",
+            }
+        ],
+    }
+    context.update(overrides)
+    return context
+
+
+def detail_context(**overrides: object) -> dict[str, object]:
+    context: dict[str, object] = {
+        "home_url": "/",
+        "catalog_url": "/?category=vegetable",
+        "detail_state": "ready",
+        "series": {
+            "category_label": "채소류",
+            "item_name": "배추",
+            "variety_name": "봄",
+            "grade_name": "상품",
+            "unit_label": "포기 × 1",
+            "current_price_machine": "8000",
+            "current_price_label": "8,000원",
+        },
+        "comparisons": [
+            {
+                "period_label": "1주 전 제공값",
+                "available": True,
+                "reference_value_label": "10,000원",
+                "difference_label": "2,000원",
+                "percentage_label": "-20.0%",
+                "direction_code": "LOWER",
+                "direction_label": "낮음",
+                "reference_date_available": False,
+            },
+            {
+                "period_label": "1개월 전 제공값",
+                "available": True,
+                "reference_value_label": "10,000원",
+                "difference_label": "2,000원",
+                "percentage_label": "-20.0%",
+                "direction_code": "LOWER",
+                "direction_label": "낮음",
+                "reference_date_available": False,
+            },
+            {
+                "period_label": "1년 전 제공값",
+                "available": False,
+                "unavailable_reason_label": "source 응답에 비교값이 없습니다.",
+                "reference_date_available": False,
+            },
+        ],
+        "provenance": {
+            "source_name": "한국농수산식품유통공사 KAMIS 최근일자 도·소매가격정보",
+            "source_url": "https://www.data.go.kr/data/15156063/openapi.do",
+            "dataset_id": "15156063",
+            "source_date_iso": "2026-08-29",
+            "source_date_label": "2026년 8월 29일",
+            "coverage_label": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
+            "checked_at_label": "2026년 8월 30일 09:00",
+            "reviewed_at_label": "2026년 8월 30일 09:30",
+            "freshness_state": "current",
+            "freshness_label": "공개 조사일 확인됨",
+        },
+    }
+    context.update(overrides)
+    return context
+
+
+def test_catalog_renders_semantic_search_and_long_identity() -> None:
+    html = render("grocery/catalog.html", catalog_context())
+
+    assert '<html lang="ko">' in html
+    assert 'href="#main-content"' in html
+    assert '<main id="main-content"' in html
+    assert 'role="search"' in html
+    assert '<label for="catalog-query">공식 품목명</label>' in html
+    assert 'name="q"' in html
+    assert 'aria-current="page"' in html
+    assert "아주긴한국어공식품목명이줄바꿈되어야하는배추" in html
+    assert "아주긴원문판매단위표시 포기 × 1" in html
+    assert "공개 조사일 확인됨" in html
+
+
+def test_catalog_validation_error_is_associated_with_input() -> None:
+    html = render(
+        "grocery/catalog.html",
+        catalog_context(query="잘못된 입력", query_error="검색어는 80자 이하여야 합니다."),
+    )
+
+    assert 'id="search-error"' in html
+    assert 'role="alert"' in html
+    assert 'aria-invalid="true"' in html
+    assert 'aria-describedby="catalog-query-hint search-error"' in html
+    assert "검색어는 80자 이하여야 합니다." in html
+
+
+@pytest.mark.parametrize(
+    ("state", "expected_copy", "expected_role"),
+    [
+        ("loading", "자료를 불러오는 중", 'role="status"'),
+        ("empty", "조건에 맞는 항목 없음", 'role="status"'),
+        ("unavailable", "공개 조사값 없음", 'role="status"'),
+        ("stale", "마지막 검토 자료 표시 중", 'role="status"'),
+        ("server_error", "자료를 표시하지 못함", 'role="alert"'),
+    ],
+)
+def test_catalog_state_has_text_and_semantic_role(
+    state: str,
+    expected_copy: str,
+    expected_role: str,
+) -> None:
+    context = catalog_context(catalog_state=state, retry_url="/")
+    if state != "stale":
+        context["results"] = []
+    html = render(
+        "grocery/catalog.html",
+        context,
+    )
+
+    assert expected_copy in html
+    assert expected_role in html
+
+
+def test_detail_renders_exact_identity_comparisons_and_provenance() -> None:
+    html = render("grocery/detail.html", detail_context())
+
+    assert "비교 대상의 정확한 조건" in html
+    assert "품종" in html and "봄" in html
+    assert "등급" in html and "상품" in html
+    assert "판매 단위" in html and "포기 × 1" in html
+    assert "조사범위" in html and "22개 도시 지역 전체 집계" in html
+    assert "2,000원 낮음" in html
+    assert "(-20.0%)" in html
+    assert "비교 정보 없음" in html
+    assert "source가 비교 기준일을 별도로 제공하지 않음" in html
+    assert "데이터셋 15156063" in html
+    assert "공개 검토일" in html
+
+
+def test_detail_direction_is_not_conveyed_by_symbol_or_color_alone() -> None:
+    html = render("grocery/detail.html", detail_context())
+
+    assert '<span class="direction__symbol" aria-hidden="true">' in html
+    assert "낮음" in html
+    assert "비교 정보 없음" in html
+
+
+@pytest.mark.parametrize(
+    ("template_name", "heading"),
+    [
+        ("400.html", "요청을 확인해 주세요"),
+        ("403.html", "이 페이지에 접근할 수 없습니다"),
+        ("404.html", "페이지를 찾을 수 없습니다"),
+        ("500.html", "지금은 페이지를 표시할 수 없습니다"),
+    ],
+)
+def test_error_templates_render_recovery_link(template_name: str, heading: str) -> None:
+    html = render(template_name, {"home_url": "/"})
+
+    assert heading in html
+    assert '<main id="main-content"' in html
+    assert 'href="/"' in html
+
+
+def test_public_templates_do_not_contain_forbidden_claims() -> None:
+    templates = [
+        *Path(settings.BASE_DIR, "grocery", "templates").rglob("*.html"),
+        *Path(settings.BASE_DIR, "templates").rglob("*.html"),
+    ]
+    rendered_templates = "\n".join(path.read_text(encoding="utf-8") for path in templates)
+
+    for phrase in FORBIDDEN_PUBLIC_PHRASES:
+        assert phrase not in rendered_templates
+
+
+def test_styles_define_small_viewport_safety_focus_and_touch_targets() -> None:
+    css = Path(settings.BASE_DIR, "grocery", "static", "grocery", "app.css").read_text(
+        encoding="utf-8"
+    )
+
+    assert "box-sizing: border-box" in css
+    assert "min-width: 0" in css
+    assert "overflow-wrap: anywhere" in css
+    assert ":focus-visible" in css
+    assert "min-height: 2.75rem" in css
+    assert "@media (min-width: 40rem)" in css
+    assert "@media (min-width: 64rem)" in css
+    assert "@media (forced-colors: active)" in css
diff --git a/templates/400.html b/templates/400.html
new file mode 100644
index 0000000..34ff8e2
--- /dev/null
+++ b/templates/400.html
@@ -0,0 +1,12 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}요청을 확인해 주세요 | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow">요청 오류 · 400</p>
+    <h1 id="error-heading">요청을 확인해 주세요</h1>
+    <p>입력한 주소나 검색 조건을 처리할 수 없습니다. 내용을 고친 뒤 다시 시도해 주세요.</p>
+    <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
+  </section>
+{% endblock %}
diff --git a/templates/403.html b/templates/403.html
new file mode 100644
index 0000000..b077f22
--- /dev/null
+++ b/templates/403.html
@@ -0,0 +1,12 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}접근할 수 없음 | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow">접근 제한 · 403</p>
+    <h1 id="error-heading">이 페이지에 접근할 수 없습니다</h1>
+    <p>공개된 조사값은 목록에서 확인할 수 있습니다.</p>
+    <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+  </section>
+{% endblock %}
diff --git a/templates/404.html b/templates/404.html
new file mode 100644
index 0000000..652c719
--- /dev/null
+++ b/templates/404.html
@@ -0,0 +1,12 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}페이지를 찾을 수 없음 | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow">찾을 수 없음 · 404</p>
+    <h1 id="error-heading">페이지를 찾을 수 없습니다</h1>
+    <p>주소가 바뀌었거나 현재 공개 목록에 없는 항목입니다.</p>
+    <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+  </section>
+{% endblock %}
diff --git a/templates/500.html b/templates/500.html
new file mode 100644
index 0000000..b6e1325
--- /dev/null
+++ b/templates/500.html
@@ -0,0 +1,12 @@
+{% extends "grocery/base.html" %}
+
+{% block title %}일시적인 오류 | 농산물 조사값 살펴보기{% endblock %}
+
+{% block content %}
+  <section class="error-page" role="alert" aria-labelledby="error-heading">
+    <p class="eyebrow">서버 오류 · 500</p>
+    <h1 id="error-heading">지금은 페이지를 표시할 수 없습니다</h1>
+    <p>요청을 완료하지 못했습니다. 잠시 후 다시 시도해 주세요.</p>
+    <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
+  </section>
+{% endblock %}


