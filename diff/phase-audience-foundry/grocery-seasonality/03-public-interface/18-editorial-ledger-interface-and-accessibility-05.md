## `feat(frontend): add shared public navigation partials`

diff --git a/grocery/templates/grocery/_pagination.html b/grocery/templates/grocery/_pagination.html
new file mode 100644
index 0000000..81e058f
--- /dev/null
+++ b/grocery/templates/grocery/_pagination.html
@@ -0,0 +1,15 @@
+{% if pagination and pagination.has_multiple_pages %}
+  <nav class="pagination" aria-label="페이지 이동">
+    <div class="pagination__previous">
+      {% if pagination.previous_url %}
+        <a href="{{ pagination.previous_url }}" rel="prev">← 이전</a>
+      {% endif %}
+    </div>
+    <p>{{ pagination.page_label }}</p>
+    <div class="pagination__next">
+      {% if pagination.next_url %}
+        <a href="{{ pagination.next_url }}" rel="next">다음 →</a>
+      {% endif %}
+    </div>
+  </nav>
+{% endif %}
diff --git a/grocery/templates/grocery/_publication_summary.html b/grocery/templates/grocery/_publication_summary.html
new file mode 100644
index 0000000..d1ec2c1
--- /dev/null
+++ b/grocery/templates/grocery/_publication_summary.html
@@ -0,0 +1,21 @@
+{% if publication %}
+  <dl class="publication-summary" aria-label="{{ publication_label|default:'공개 자료 정보' }}">
+    <div>
+      <dt>KAMIS 자료 확인 일시</dt>
+      <dd>
+        {% if publication.checked_at_iso %}<time datetime="{{ publication.checked_at_iso }}">{% endif %}
+        {{ publication.checked_at_display }}
+        {% if publication.checked_at_iso %}</time>{% endif %}
+      </dd>
+    </div>
+    <div>
+      <dt>자료 상태</dt>
+      <dd>
+        <span class="status-text status-text--{{ publication.freshness_state|default:'current' }}">
+          <span aria-hidden="true">{% if publication.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
+          {{ publication.freshness_label }}
+        </span>
+      </dd>
+    </div>
+  </dl>
+{% endif %}
diff --git a/grocery/templates/grocery/_series_nav.html b/grocery/templates/grocery/_series_nav.html
new file mode 100644
index 0000000..977a6b4
--- /dev/null
+++ b/grocery/templates/grocery/_series_nav.html
@@ -0,0 +1,15 @@
+{% if section_nav %}
+  <nav class="series-nav" aria-label="{{ series.item_name }} 자료 보기">
+    <ul>
+      {% for item in section_nav %}
+        <li>
+          {% if item.current %}
+            <span class="series-nav__item series-nav__item--current" aria-current="page">{{ item.label }}</span>
+          {% elif item.available and item.url %}
+            <a class="series-nav__item" href="{{ item.url }}">{{ item.label }}</a>
+          {% endif %}
+        </li>
+      {% endfor %}
+    </ul>
+  </nav>
+{% endif %}
diff --git a/grocery/templates/grocery/_state_notice.html b/grocery/templates/grocery/_state_notice.html
index 896f740..9e314bc 100644
--- a/grocery/templates/grocery/_state_notice.html
+++ b/grocery/templates/grocery/_state_notice.html
@@ -40,4 +40,20 @@
       {% if retry_url %}<a class="button button--secondary" href="{{ retry_url }}">다시 불러오기</a>{% endif %}
     </div>
   </section>
+{% elif state == "selection_required" %}
+  <section class="state-notice state-notice--neutral" role="status">
+    <span class="state-notice__symbol" aria-hidden="true">→</span>
+    <div>
+      <h2 class="state-notice__title">지역을 먼저 선택하세요</h2>
+      <p>월별 조사값을 확인할 지역을 선택해 주세요.</p>
+    </div>
+  </section>
+{% elif state == "partial" %}
+  <section class="state-notice state-notice--warning" role="status">
+    <span class="state-notice__symbol" aria-hidden="true">!</span>
+    <div>
+      <h2 class="state-notice__title">일부 품목을 제외했습니다</h2>
+      <p>현재 공개 목록에 없는 품목 {{ excluded_count }}개는 표시하지 않았습니다.</p>
+    </div>
+  </section>
 {% endif %}
diff --git a/grocery/templates/grocery/base.html b/grocery/templates/grocery/base.html
index 4351780..b5089ab 100644
--- a/grocery/templates/grocery/base.html
+++ b/grocery/templates/grocery/base.html
@@ -31,6 +31,11 @@
             <span class="brand__description">채소·과일 소매 조사값</span>
           </span>
         </a>
+        {% if selection_url %}
+          <nav class="site-actions" aria-label="내 선택 목록">
+            <a href="{{ selection_url }}">선택 목록{% if selection_count %} <span aria-label="{{ selection_count }}개">{{ selection_count }}</span>{% endif %}</a>
+          </nav>
+        {% endif %}
       </div>
     </header>
 
@@ -40,7 +45,7 @@
 
     <footer class="site-footer">
       <div class="page-shell site-footer__inner">
-        <p>표시값은 KAMIS 소매 조사 평균입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.</p>
+        <p>{% block footer_note %}표시값은 KAMIS 소매 조사 평균입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.{% endblock %}</p>
       </div>
     </footer>
   </body>


## `feat(frontend): add historical page primitives`

diff --git a/grocery/templates/grocery/_form_errors.html b/grocery/templates/grocery/_form_errors.html
new file mode 100644
index 0000000..c775927
--- /dev/null
+++ b/grocery/templates/grocery/_form_errors.html
@@ -0,0 +1,16 @@
+{% if validation_errors %}
+  <div class="form-error" role="alert" aria-labelledby="validation-title">
+    <span class="form-error__symbol" aria-hidden="true">!</span>
+    <div class="form-error__content">
+      <p id="validation-title" class="form-error__title"><strong>입력 내용을 확인하세요</strong></p>
+      <ul>
+        {% for error in validation_errors %}
+          <li>
+            {{ error.message }}
+            {% if error.target %}<a class="form-error__link" href="#{{ error.target }}">입력으로 이동</a>{% endif %}
+          </li>
+        {% endfor %}
+      </ul>
+    </div>
+  </div>
+{% endif %}
diff --git a/grocery/templates/grocery/_historical_header.html b/grocery/templates/grocery/_historical_header.html
new file mode 100644
index 0000000..6dfb10c
--- /dev/null
+++ b/grocery/templates/grocery/_historical_header.html
@@ -0,0 +1,16 @@
+<nav class="breadcrumb" aria-label="이전 화면으로 돌아가기">
+  <a href="{{ back_url|default:catalog_url }}">← {{ back_label|default:"채소·과일 소매 조사값" }}</a>
+</nav>
+
+<header class="record-heading">
+  <p class="eyebrow">{{ page_eyebrow }}</p>
+  <h1>{{ series.item_name }}</h1>
+  <p class="record-heading__summary">{{ page_summary }}</p>
+  <dl class="detail-signature" aria-label="품목 조건">
+    <div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>
+    <div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>
+    <div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>
+  </dl>
+</header>
+
+{% include "grocery/_series_nav.html" with section_nav=section_nav series=series %}


