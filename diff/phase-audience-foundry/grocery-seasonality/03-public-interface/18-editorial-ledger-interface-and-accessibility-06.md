## `feat(frontend): restructure public information hierarchy`

diff --git a/grocery/templates/grocery/_state_notice.html b/grocery/templates/grocery/_state_notice.html
index 9e314bc..2486c21 100644
--- a/grocery/templates/grocery/_state_notice.html
+++ b/grocery/templates/grocery/_state_notice.html
@@ -1,7 +1,7 @@
 {% if state == "loading" %}
   <section class="state-notice state-notice--info" role="status" aria-live="polite" aria-atomic="true">
     <span class="state-notice__symbol" aria-hidden="true">…</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">조사 자료를 불러오고 있습니다</h2>
       <p>공개된 자료를 확인하는 동안 잠시 기다려 주세요.</p>
     </div>
@@ -9,7 +9,7 @@
 {% elif state == "empty" %}
   <section class="state-notice state-notice--neutral" role="status">
     <span class="state-notice__symbol" aria-hidden="true">○</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">검색 결과가 없습니다</h2>
       <p>품목명을 바꾸거나 다른 부류를 선택하세요.</p>
       <a class="button button--secondary" href="{{ recovery_url|default:'/' }}">전체 항목 보기</a>
@@ -18,7 +18,7 @@
 {% elif state == "unavailable" %}
   <section class="state-notice state-notice--warning" role="status">
     <span class="state-notice__symbol" aria-hidden="true">!</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">아직 공개된 조사 자료가 없습니다</h2>
       <p>검토를 마친 자료가 공개되면 이곳에 표시됩니다.</p>
     </div>
@@ -26,7 +26,7 @@
 {% elif state == "stale" %}
   <section class="state-notice state-notice--warning" role="status">
     <span class="state-notice__symbol" aria-hidden="true">!</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">마지막 공개 자료를 표시합니다</h2>
       <p>최근 자료 확인이 필요합니다. 마지막으로 검토를 마친 조사값을 표시합니다.</p>
     </div>
@@ -34,7 +34,7 @@
 {% elif state == "server_error" %}
   <section class="state-notice state-notice--error" role="alert">
     <span class="state-notice__symbol" aria-hidden="true">×</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">조사 자료를 불러오지 못했습니다</h2>
       <p>잠시 후 다시 불러오세요.</p>
       {% if retry_url %}<a class="button button--secondary" href="{{ retry_url }}">다시 불러오기</a>{% endif %}
@@ -43,7 +43,7 @@
 {% elif state == "selection_required" %}
   <section class="state-notice state-notice--neutral" role="status">
     <span class="state-notice__symbol" aria-hidden="true">→</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">지역을 먼저 선택하세요</h2>
       <p>월별 조사값을 확인할 지역을 선택해 주세요.</p>
     </div>
@@ -51,7 +51,7 @@
 {% elif state == "partial" %}
   <section class="state-notice state-notice--warning" role="status">
     <span class="state-notice__symbol" aria-hidden="true">!</span>
-    <div>
+    <div class="state-notice__body">
       <h2 class="state-notice__title">일부 품목을 제외했습니다</h2>
       <p>현재 공개 목록에 없는 품목 {{ excluded_count }}개는 표시하지 않았습니다.</p>
     </div>
diff --git a/grocery/templates/grocery/catalog.html b/grocery/templates/grocery/catalog.html
index b97c491..e5f5db3 100644
--- a/grocery/templates/grocery/catalog.html
+++ b/grocery/templates/grocery/catalog.html
@@ -3,138 +3,136 @@
 {% block title %}채소·과일 소매 조사값 | 초록장부{% endblock %}
 
 {% block content %}
-  <header class="page-heading page-heading--catalog">
-    <p class="eyebrow">KAMIS 소매 조사 평균</p>
-    <h1>채소·과일 소매 조사값</h1>
-    <p class="page-heading__summary">
-      품목을 둘러보거나 이름으로 찾아보세요. 품종·등급·판매 단위·조사 범위가 같은 값만
-      비교합니다.
+  <header class="page-heading page-heading--catalog catalog-hero">
+    <div class="catalog-hero__title">
+      <p class="eyebrow">KAMIS 소매 조사 평균</p>
+      <h1>채소·과일 소매 조사값</h1>
+    </div>
+    <p class="page-heading__summary catalog-hero__summary">
+      품목을 둘러보거나 이름으로 찾아보세요. 품종·등급·판매 단위·조사 범위가 같은 값만 비교합니다.
     </p>
   </header>
 
   {% if catalog_state != "loading" and catalog_state != "unavailable" and catalog_state != "server_error" %}
-    <section class="search-panel" aria-labelledby="catalog-search-heading">
-    <div class="search-panel__heading">
-      <h2 id="catalog-search-heading">품목 찾기</h2>
-    </div>
-
-    {% if validation_errors %}
-      {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
-    {% elif query_error or category_error %}
-      <div id="search-error" class="form-error" role="alert">
-        <span class="form-error__symbol" aria-hidden="true">!</span>
-        <div class="form-error__content">
-          <p class="form-error__title"><strong>입력 내용을 확인하세요</strong></p>
-          <ul>
-            {% if query_error %}
-              <li>{{ query_error }} <a class="form-error__link" href="#catalog-query">품목명 입력으로 이동</a></li>
-            {% endif %}
-            {% if category_error %}
-              <li>{{ category_error }} <a class="form-error__link" href="{{ form_action }}">부류 선택 초기화</a></li>
-            {% endif %}
-          </ul>
+    <section class="search-panel catalog-toolbar" aria-label="품목 찾기와 표시 기준">
+      {% if validation_errors %}
+        {% include "grocery/_form_errors.html" with validation_errors=validation_errors %}
+      {% elif query_error or category_error %}
+        <div id="search-error" class="form-error" role="alert">
+          <span class="form-error__symbol" aria-hidden="true">!</span>
+          <div class="form-error__content">
+            <p class="form-error__title"><strong>입력 내용을 확인하세요</strong></p>
+            <ul>
+              {% if query_error %}
+                <li>{{ query_error }} <a class="form-error__link" href="#catalog-query">품목명 입력으로 이동</a></li>
+              {% endif %}
+              {% if category_error %}
+                <li>{{ category_error }} <a class="form-error__link" href="{{ form_action }}">부류 선택 초기화</a></li>
+              {% endif %}
+            </ul>
+          </div>
         </div>
-      </div>
-    {% endif %}
-
-    {% if categories %}
-      <nav class="category-nav" aria-label="부류 선택">
-        <ul class="segment-list">
-          {% for category in categories %}
-            <li>
-              <a
-                class="segment{% if category.selected %} segment--selected{% endif %}"
-                href="{{ category.url }}"
-                {% if category.selected %}aria-current="page"{% endif %}
+      {% endif %}
+
+      {% if categories %}
+        <nav class="category-nav" aria-label="부류 선택">
+          <ul class="segment-list">
+            {% for category in categories %}
+              <li>
+                <a
+                  class="segment{% if category.selected %} segment--selected{% endif %}"
+                  href="{{ category.url }}"
+                  {% if category.selected %}aria-current="page"{% endif %}
+                >
+                  {% if category.selected %}<span class="segment__selected-mark" aria-hidden="true">✓</span>{% endif %}
+                  <span>{{ category.label }}</span>
+                </a>
+              </li>
+            {% endfor %}
+          </ul>
+        </nav>
+      {% endif %}
+
+      <div class="catalog-toolbar__tools">
+        <details class="catalog-search"{% if search_open or query_error %} open{% endif %}>
+          <summary>품목명으로 찾기</summary>
+          <form class="search-form" action="{{ form_action|default:'' }}" method="get" role="search">
+            <div class="field-group">
+              <label for="catalog-query">품목명</label>
+              <p id="catalog-query-hint" class="field-hint">KAMIS 공식 품목명으로 검색하세요. 예: 배추, 사과</p>
+              <input
+                id="catalog-query"
+                name="q"
+                type="search"
+                maxlength="80"
+                autocomplete="off"
+                enterkeyhint="search"
+                aria-describedby="catalog-query-hint{% if query_error and validation_errors %} validation-title{% elif query_error %} search-error{% endif %}"
+                {% if query_error %}aria-invalid="true"{% endif %}
               >
-                {% if category.selected %}<span class="segment__selected-mark" aria-hidden="true">✓</span>{% endif %}
-                <span>{{ category.label }}</span>
-              </a>
-            </li>
-          {% endfor %}
-        </ul>
-      </nav>
-    {% endif %}
-
-    <details class="catalog-search"{% if search_open or query_error %} open{% endif %}>
-      <summary>품목명으로 찾기</summary>
-      <form class="search-form" action="{{ form_action|default:'' }}" method="get" role="search">
-        <div class="field-group">
-          <label for="catalog-query">품목명</label>
-          <p id="catalog-query-hint" class="field-hint">KAMIS 공식 품목명으로 검색하세요. 예: 배추, 사과</p>
-          <input
-            id="catalog-query"
-            name="q"
-            type="search"
-            maxlength="80"
-            autocomplete="off"
-            enterkeyhint="search"
-            aria-describedby="catalog-query-hint{% if query_error and validation_errors %} validation-title{% elif query_error %} search-error{% endif %}"
-            {% if query_error %}aria-invalid="true"{% endif %}
-          >
-        </div>
-        {% if selected_category %}
-          <input type="hidden" name="category" value="{{ selected_category }}">
+            </div>
+            {% if selected_category %}
+              <input type="hidden" name="category" value="{{ selected_category }}">
+            {% endif %}
+            {% if selected_period and selected_period != "week" %}<input type="hidden" name="period" value="{{ selected_period }}">{% endif %}
+            {% if selected_direction and selected_direction != "all" %}<input type="hidden" name="direction" value="{{ selected_direction }}">{% endif %}
+            {% if selected_sort and selected_sort != "name" %}<input type="hidden" name="sort" value="{{ selected_sort }}">{% endif %}
+            <button class="button button--primary" type="submit">검색</button>
+          </form>
+        </details>
+
+        {% if period_options and direction_options and sort_options %}
+          <details class="catalog-options"{% if filters_open or period_error or direction_error or sort_error %} open{% endif %}>
+            <summary>
+              <span>비교·정렬 기준</span>
+              <span class="catalog-options__current">
+                {{ selected_period_label|default:"1주 비교" }} · {{ selected_sort_label|default:"품목명 순" }}
+              </span>
+            </summary>
+            <form class="filter-form" action="{{ form_action|default:'' }}" method="get">
+              {% if selected_category %}<input type="hidden" name="category" value="{{ selected_category }}">{% endif %}
+              <div class="field-group">
+                <label for="catalog-period">비교 기간</label>
+                <select
+                  id="catalog-period"
+                  name="period"
+                  {% if period_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
+                >
+                  {% for option in period_options %}
+                    <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+                  {% endfor %}
+                </select>
+              </div>
+              <div class="field-group">
+                <label for="catalog-direction">변화 방향</label>
+                <select
+                  id="catalog-direction"
+                  name="direction"
+                  {% if direction_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
+                >
+                  {% for option in direction_options %}
+                    <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+                  {% endfor %}
+                </select>
+              </div>
+              <div class="field-group">
+                <label for="catalog-sort">표시 순서</label>
+                <select
+                  id="catalog-sort"
+                  name="sort"
+                  {% if sort_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
+                >
+                  {% for option in sort_options %}
+                    <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
+                  {% endfor %}
+                </select>
+              </div>
+              <button class="button button--secondary" type="submit">기준 적용</button>
+            </form>
+            <p class="catalog-options__note">가격이 아니라 같은 품목 조건의 변화율을 기준으로 정렬합니다.</p>
+          </details>
         {% endif %}
-        {% if selected_period and selected_period != "week" %}<input type="hidden" name="period" value="{{ selected_period }}">{% endif %}
-        {% if selected_direction and selected_direction != "all" %}<input type="hidden" name="direction" value="{{ selected_direction }}">{% endif %}
-        {% if selected_sort and selected_sort != "name" %}<input type="hidden" name="sort" value="{{ selected_sort }}">{% endif %}
-        <button class="button button--primary" type="submit">검색</button>
-      </form>
-    </details>
-
-    {% if period_options and direction_options and sort_options %}
-      <details class="catalog-options"{% if filters_open or period_error or direction_error or sort_error %} open{% endif %}>
-        <summary>
-          <span>비교·정렬 기준</span>
-          <span class="catalog-options__current">
-            {{ selected_period_label|default:"1주 비교" }}
-            · {{ selected_sort_label|default:"품목명 순" }}
-          </span>
-        </summary>
-        <form class="filter-form" action="{{ form_action|default:'' }}" method="get">
-          {% if selected_category %}<input type="hidden" name="category" value="{{ selected_category }}">{% endif %}
-          <div class="field-group">
-            <label for="catalog-period">비교 기간</label>
-            <select
-              id="catalog-period"
-              name="period"
-              {% if period_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
-            >
-              {% for option in period_options %}
-                <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
-              {% endfor %}
-            </select>
-          </div>
-          <div class="field-group">
-            <label for="catalog-direction">변화 방향</label>
-            <select
-              id="catalog-direction"
-              name="direction"
-              {% if direction_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
-            >
-              {% for option in direction_options %}
-                <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
-              {% endfor %}
-            </select>
-          </div>
-          <div class="field-group">
-            <label for="catalog-sort">표시 순서</label>
-            <select
-              id="catalog-sort"
-              name="sort"
-              {% if sort_error %}aria-invalid="true" aria-describedby="validation-title"{% endif %}
-            >
-              {% for option in sort_options %}
-                <option value="{{ option.value }}"{% if option.selected %} selected{% endif %}>{{ option.label }}</option>
-              {% endfor %}
-            </select>
-          </div>
-          <button class="button button--secondary" type="submit">기준 적용</button>
-        </form>
-        <p class="catalog-options__note">가격이 아니라 같은 품목 조건의 변화율을 기준으로 정렬합니다.</p>
-      </details>
-    {% endif %}
+      </div>
     </section>
   {% endif %}
 
@@ -149,11 +147,16 @@
     {% if results %}
       <section class="catalog-results" aria-labelledby="catalog-results-heading">
         <div class="section-heading catalog-results__heading">
-          <h2 id="catalog-results-heading">공개 항목</h2>
+          <div>
+            <p class="eyebrow">검토를 마친 공개 자료</p>
+            <h2 id="catalog-results-heading">품목별 조사값</h2>
+          </div>
           {% if result_count_label %}<p class="result-count" aria-live="polite">{{ result_count_label }}</p>{% endif %}
         </div>
 
-        {% include "grocery/_publication_summary.html" with publication=publication %}
+        <div class="catalog-meta">
+          {% include "grocery/_publication_summary.html" with publication=publication %}
+        </div>
 
         {% regroup results by category_label as result_groups %}
         <div class="catalog-ledger">
diff --git a/grocery/templates/grocery/detail.html b/grocery/templates/grocery/detail.html
index 3f8edf0..6fda882 100644
--- a/grocery/templates/grocery/detail.html
+++ b/grocery/templates/grocery/detail.html
@@ -9,8 +9,11 @@
 
   <header class="detail-intro">
     <div class="detail-intro__identity">
-      <p class="eyebrow">{{ series.category_label|default:"공개 조사 항목" }}</p>
-      <h1>{{ series.item_name|default:"조사값 상세" }}</h1>
+      <div class="detail-intro__lead">
+        <p class="eyebrow">{{ series.category_label|default:"공개 조사 항목" }}</p>
+        <h1>{{ series.item_name|default:"조사값 상세" }}</h1>
+        <p class="detail-intro__summary">같은 품목 조건에서 확인된 최근 조사값입니다.</p>
+      </div>
       {% if series.variety_name or series.grade_name or series.unit_label or provenance.coverage_label %}
         <dl class="detail-signature" aria-label="품목 조건 요약">
           {% if series.variety_name %}<div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>{% endif %}
@@ -19,21 +22,12 @@
           {% if provenance.coverage_label %}<div><dt>조사 범위</dt><dd>{{ provenance.coverage_label }}</dd></div>{% endif %}
         </dl>
       {% endif %}
-      {% if selection_add_url %}
-        <div class="detail-actions">
-          <a class="button button--secondary" href="{{ selection_add_url }}">선택 목록에 담기</a>
-        </div>
-      {% elif historical_links.selection_url %}
-        <div class="detail-actions">
-          <a class="button button--secondary" href="{{ historical_links.selection_url }}">선택 목록에 담기</a>
-        </div>
-      {% endif %}
     </div>
 
     {% if detail_state != "loading" and detail_state != "unavailable" and detail_state != "server_error" %}
-      <div class="current-price">
+      <section class="current-price" aria-labelledby="current-price-heading">
         <div class="current-price__heading">
-          <p class="eyebrow">KAMIS 소매 조사 평균</p>
+          <p class="eyebrow">최근 공개값</p>
           <h2 id="current-price-heading">조사일 평균</h2>
         </div>
         <p class="current-price__value">
@@ -48,21 +42,33 @@
           {% if provenance.source_date_iso %}</time>{% endif %}
         </p>
         <p class="current-price__note">개별 판매처의 판매 금액이 아닌 KAMIS 소매 조사 평균입니다.</p>
-      </div>
+      </section>
     {% endif %}
   </header>
 
-  {% if section_nav %}
-    {% include "grocery/_series_nav.html" with section_nav=section_nav series=series %}
-  {% elif historical_links %}
-    <nav class="series-nav" aria-label="{{ series.item_name }} 자료 보기">
-      <ul>
-        <li><span class="series-nav__item series-nav__item--current" aria-current="page">최근 조사값</span></li>
-        {% if historical_links.history_url %}<li><a class="series-nav__item" href="{{ historical_links.history_url }}">월별 기록</a></li>{% endif %}
-        {% if historical_links.regions_url %}<li><a class="series-nav__item" href="{{ historical_links.regions_url }}">지역별 조사값</a></li>{% endif %}
-      </ul>
-    </nav>
-  {% endif %}
+  <div class="detail-pathways">
+    {% if section_nav %}
+      {% include "grocery/_series_nav.html" with section_nav=section_nav series=series %}
+    {% elif historical_links %}
+      <nav class="series-nav" aria-label="{{ series.item_name }} 자료 보기">
+        <ul>
+          <li><span class="series-nav__item series-nav__item--current" aria-current="page">최근 조사값</span></li>
+          {% if historical_links.history_url %}<li><a class="series-nav__item" href="{{ historical_links.history_url }}">월별 기록</a></li>{% endif %}
+          {% if historical_links.regions_url %}<li><a class="series-nav__item" href="{{ historical_links.regions_url }}">지역별 조사값</a></li>{% endif %}
+        </ul>
+      </nav>
+    {% endif %}
+
+    {% if selection_add_url %}
+      <div class="detail-actions">
+        <a class="button button--secondary" href="{{ selection_add_url }}">선택 목록에 담기</a>
+      </div>
+    {% elif historical_links.selection_url %}
+      <div class="detail-actions">
+        <a class="button button--secondary" href="{{ historical_links.selection_url }}">선택 목록에 담기</a>
+      </div>
+    {% endif %}
+  </div>
 
   {% if detail_state == "loading" or detail_state == "unavailable" or detail_state == "server_error" %}
     {% include "grocery/_state_notice.html" with state=detail_state retry_url=retry_url %}
diff --git a/grocery/templates/grocery/history.html b/grocery/templates/grocery/history.html
index 390a679..01e2523 100644
--- a/grocery/templates/grocery/history.html
+++ b/grocery/templates/grocery/history.html
@@ -3,7 +3,7 @@
 {% block title %}{{ series.item_name|default:"품목" }} 월별 기록 | 초록장부{% endblock %}
 
 {% block content %}
-  {% include "grocery/_historical_header.html" with page_eyebrow="월별 과거 가격 패턴" page_summary="선택한 지역에서 KAMIS가 제공한 월별 소매 조사 평균과 조사 범위를 확인합니다." %}
+  {% include "grocery/_historical_header.html" with page_eyebrow="월별 조사 기록" page_summary="선택한 지역에서 KAMIS가 제공한 월별 소매 조사 평균과 조사 범위를 함께 확인합니다." %}
 
   {% if history_state == "loading" or history_state == "unavailable" or history_state == "server_error" %}
     {% include "grocery/_state_notice.html" with state=history_state retry_url=retry_url %}
@@ -54,6 +54,44 @@
           <p>선은 월 조사 평균, 옅은 범위는 같은 달의 최저·최고 조사값입니다.</p>
         </div>
 
+        {% if history_summary %}
+          <dl class="history-summary" aria-label="월별 조사값 요약">
+            <div class="history-summary__item history-summary__item--latest">
+              <dt>최근 월평균</dt>
+              <dd>
+                <data value="{{ history_summary.latest.mean_machine }}">{{ history_summary.latest.mean_label }}</data>
+                <span class="history-summary__period">
+                  {% if history_summary.latest.period_iso %}<time datetime="{{ history_summary.latest.period_iso }}">{% endif %}
+                  {{ history_summary.latest.period_label }}
+                  {% if history_summary.latest.period_iso %}</time>{% endif %}
+                </span>
+              </dd>
+            </div>
+            <div class="history-summary__item">
+              <dt>가장 낮은 월평균</dt>
+              <dd>
+                <data value="{{ history_summary.lowest.mean_machine }}">{{ history_summary.lowest.mean_label }}</data>
+                <span class="history-summary__period">
+                  {% if history_summary.lowest.period_iso %}<time datetime="{{ history_summary.lowest.period_iso }}">{% endif %}
+                  {{ history_summary.lowest.period_label }}
+                  {% if history_summary.lowest.period_iso %}</time>{% endif %}
+                </span>
+              </dd>
+            </div>
+            <div class="history-summary__item">
+              <dt>가장 높은 월평균</dt>
+              <dd>
+                <data value="{{ history_summary.highest.mean_machine }}">{{ history_summary.highest.mean_label }}</data>
+                <span class="history-summary__period">
+                  {% if history_summary.highest.period_iso %}<time datetime="{{ history_summary.highest.period_iso }}">{% endif %}
+                  {{ history_summary.highest.period_label }}
+                  {% if history_summary.highest.period_iso %}</time>{% endif %}
+                </span>
+              </dd>
+            </div>
+          </dl>
+        {% endif %}
+
         {% if history_chart %}
           <figure class="history-chart">
             <svg
@@ -89,29 +127,65 @@
           </figure>
         {% endif %}
 
-        <div class="month-ledger">
-          <div class="month-ledger__head" aria-hidden="true">
-            <span>연월</span><span>소매 조사 평균</span><span>월 최저 조사값</span><span>월 최고 조사값</span>
-          </div>
-          <ol class="month-list">
-            {% for point in monthly_points %}
-              <li class="month-row{% if not point.available %} month-row--unavailable{% endif %}">
-                <p class="month-row__date">
-                  {% if point.period_iso %}<time datetime="{{ point.period_iso }}">{% endif %}{{ point.period_label }}{% if point.period_iso %}</time>{% endif %}
-                </p>
-                {% if point.available %}
-                  <dl class="month-row__facts">
-                    <div><dt>소매 조사 평균</dt><dd><data value="{{ point.mean_machine }}">{{ point.mean_label }}</data></dd></div>
-                    <div><dt>월 최저 조사값</dt><dd><data value="{{ point.minimum_machine }}">{{ point.minimum_label }}</data></dd></div>
-                    <div><dt>월 최고 조사값</dt><dd><data value="{{ point.maximum_machine }}">{{ point.maximum_label }}</data></dd></div>
-                  </dl>
-                {% else %}
-                  <p class="month-row__missing">{{ point.unavailable_label }}</p>
-                {% endif %}
-              </li>
+        {% if history_year_groups %}
+          <div class="history-year-groups" aria-label="연도별 월 조사 기록">
+            {% for group in history_year_groups %}
+              <details class="history-year{% if group.is_latest %} history-year--latest{% endif %}"{% if group.open %} open{% endif %}>
+                <summary>
+                  <span class="history-year__label">{{ group.label }}</span>
+                  {% if group.is_latest %}<span class="history-year__count">최근 기록</span>{% endif %}
+                </summary>
+                <div class="month-ledger">
+                  <div class="month-ledger__head" aria-hidden="true">
+                    <span>연월</span><span>소매 조사 평균</span><span>월 최저 조사값</span><span>월 최고 조사값</span>
+                  </div>
+                  <ol class="month-list">
+                    {% for point in group.points %}
+                      <li class="month-row{% if not point.available %} month-row--unavailable{% endif %}">
+                        <p class="month-row__date">
+                          {% if point.period_iso %}<time datetime="{{ point.period_iso }}">{% endif %}{{ point.period_label }}{% if point.period_iso %}</time>{% endif %}
+                        </p>
+                        {% if point.available %}
+                          <dl class="month-row__facts">
+                            <div><dt>소매 조사 평균</dt><dd><data value="{{ point.mean_machine }}">{{ point.mean_label }}</data></dd></div>
+                            <div><dt>월 최저 조사값</dt><dd><data value="{{ point.minimum_machine }}">{{ point.minimum_label }}</data></dd></div>
+                            <div><dt>월 최고 조사값</dt><dd><data value="{{ point.maximum_machine }}">{{ point.maximum_label }}</data></dd></div>
+                          </dl>
+                        {% else %}
+                          <p class="month-row__missing">{{ point.unavailable_label }}</p>
+                        {% endif %}
+                      </li>
+                    {% endfor %}
+                  </ol>
+                </div>
+              </details>
             {% endfor %}
-          </ol>
-        </div>
+          </div>
+        {% else %}
+          <div class="month-ledger">
+            <div class="month-ledger__head" aria-hidden="true">
+              <span>연월</span><span>소매 조사 평균</span><span>월 최저 조사값</span><span>월 최고 조사값</span>
+            </div>
+            <ol class="month-list">
+              {% for point in monthly_points %}
+                <li class="month-row{% if not point.available %} month-row--unavailable{% endif %}">
+                  <p class="month-row__date">
+                    {% if point.period_iso %}<time datetime="{{ point.period_iso }}">{% endif %}{{ point.period_label }}{% if point.period_iso %}</time>{% endif %}
+                  </p>
+                  {% if point.available %}
+                    <dl class="month-row__facts">
+                      <div><dt>소매 조사 평균</dt><dd><data value="{{ point.mean_machine }}">{{ point.mean_label }}</data></dd></div>
+                      <div><dt>월 최저 조사값</dt><dd><data value="{{ point.minimum_machine }}">{{ point.minimum_label }}</data></dd></div>
+                      <div><dt>월 최고 조사값</dt><dd><data value="{{ point.maximum_machine }}">{{ point.maximum_label }}</data></dd></div>
+                    </dl>
+                  {% else %}
+                    <p class="month-row__missing">{{ point.unavailable_label }}</p>
+                  {% endif %}
+                </li>
+              {% endfor %}
+            </ol>
+          </div>
+        {% endif %}
       </section>
     {% else %}
       {% include "grocery/_state_notice.html" with state="server_error" retry_url=retry_url %}
diff --git a/grocery/templates/grocery/markets.html b/grocery/templates/grocery/markets.html
index d70226d..2152cbf 100644
--- a/grocery/templates/grocery/markets.html
+++ b/grocery/templates/grocery/markets.html
@@ -18,8 +18,9 @@
 
     <section class="market-scope" aria-labelledby="market-scope-heading">
       <div>
-        <p class="eyebrow">선택 지역</p>
+        <p class="eyebrow">시장 조사 범위</p>
         <h2 id="market-scope-heading">{{ selected_region.label }}</h2>
+        <p>같은 지역 안에서 확인된 개별 시장 조사값입니다.</p>
       </div>
       <form class="scope-form scope-form--date" action="{{ markets_form_action|default:'' }}" method="get">
         <div class="field-group">
@@ -44,6 +45,24 @@
           </div>
           {% if result_count_label %}<p class="result-count">{{ result_count_label }}</p>{% endif %}
         </div>
+
+        {% if market_summary %}
+          <dl class="market-summary" aria-label="시장별 조사값 요약">
+            <div class="market-summary__item market-summary__item--count">
+              <dt>확인된 시장</dt>
+              <dd>{{ market_summary.total_count_label }}</dd>
+            </div>
+            <div class="market-summary__item">
+              <dt>가장 낮은 조사값</dt>
+              <dd><data value="{{ market_summary.minimum_machine }}">{{ market_summary.minimum_label }}</data></dd>
+            </div>
+            <div class="market-summary__item">
+              <dt>가장 높은 조사값</dt>
+              <dd><data value="{{ market_summary.maximum_machine }}">{{ market_summary.maximum_label }}</data></dd>
+            </div>
+          </dl>
+        {% endif %}
+
         <p class="section-note">시장명은 KAMIS 공식 이름 순서로 표시합니다. 각 값은 개별 시장 조사값이며 지역 평균이 아닙니다.</p>
 
         <div class="market-ledger">
diff --git a/grocery/templates/grocery/regions.html b/grocery/templates/grocery/regions.html
index ef023d3..fb24420 100644
--- a/grocery/templates/grocery/regions.html
+++ b/grocery/templates/grocery/regions.html
@@ -16,9 +16,10 @@
 
     {% include "grocery/_publication_summary.html" with publication=historical_publication publication_label="지역별 공개 자료 정보" %}
 
-    <section class="scope-controls scope-controls--compact" aria-labelledby="regions-date-heading">
+    <section class="scope-controls scope-controls--compact region-scope" aria-labelledby="regions-date-heading">
       <div class="scope-controls__heading">
-        <h2 id="regions-date-heading">조사일</h2>
+        <p class="eyebrow">비교 기준</p>
+        <h2 id="regions-date-heading">조사일 선택</h2>
         <p>지역과 시장 자료가 함께 확인된 실제 조사일만 선택할 수 있습니다.</p>
       </div>
       <form class="scope-form scope-form--date" action="{{ regions_form_action|default:'' }}" method="get">
@@ -37,11 +38,13 @@
     {% if regions_state == "validation" %}
     {% elif regional_rows %}
       <section class="region-section" aria-labelledby="regions-heading">
-        <div class="section-heading section-heading--stacked">
-          <p class="eyebrow">{% if selected_date.iso %}<time datetime="{{ selected_date.iso }}">{% endif %}{{ selected_date.label }}{% if selected_date.iso %}</time>{% endif %}</p>
-          <h2 id="regions-heading">지역별 조사 범위</h2>
-          <p>지역은 KAMIS 공식 지역명 순서로 표시합니다. 선의 양 끝은 최저·최고, 점은 평균입니다.</p>
-        </div>
+        <header class="section-heading region-section__intro">
+          <div>
+            <p class="eyebrow">{% if selected_date.iso %}<time datetime="{{ selected_date.iso }}">{% endif %}{{ selected_date.label }}{% if selected_date.iso %}</time>{% endif %}</p>
+            <h2 id="regions-heading">지역별 조사 범위</h2>
+          </div>
+          <p>선의 양 끝은 최저·최고 조사값, 점은 지역 평균입니다. 지역은 KAMIS 공식 이름 순서로 표시합니다.</p>
+        </header>
 
         <div class="region-ledger">
           <div class="region-ledger__head" aria-hidden="true">
diff --git a/grocery/templates/grocery/selection.html b/grocery/templates/grocery/selection.html
index aed4c39..f5aa881 100644
--- a/grocery/templates/grocery/selection.html
+++ b/grocery/templates/grocery/selection.html
@@ -33,44 +33,13 @@
 
       {% include "grocery/_publication_summary.html" with publication=publication publication_label="선택 목록 공개 자료 정보" %}
 
-      <section class="selection-add" aria-labelledby="selection-add-heading">
-        <div class="selection-add__heading">
-          <p class="eyebrow">현재 목록에 이어서</p>
-          <h2 id="selection-add-heading">품목 추가</h2>
-          <p>품종·등급·판매 단위를 확인하고 한 품목을 더 담으세요.</p>
-        </div>
-
-        {% if selection_limit_reached %}
-          <p class="selection-add__status" role="status">
-            <strong>다섯 품목을 모두 선택했습니다.</strong>
-            다른 품목을 담으려면 먼저 목록에서 하나를 빼세요.
-          </p>
-        {% elif can_add_selection and selection_candidates %}
-          <form class="selection-add__form" action="{{ selection_form_action|default:'/selection/' }}" method="get">
-            {% for item in items %}
-              <input type="hidden" name="series" value="{{ item.series_value }}">
-            {% endfor %}
-            <div class="field-group">
-              <label for="selection-add-item">추가할 품목</label>
-              <p id="selection-add-hint" class="field-hint">현재 목록의 마지막에 추가합니다. 최대 다섯 품목까지 담을 수 있습니다.</p>
-              <select id="selection-add-item" name="series" required aria-describedby="selection-add-hint">
-                <option value="">품목을 선택하세요</option>
-                {% for candidate in selection_candidates %}
-                  <option value="{{ candidate.value }}">{{ candidate.label }}</option>
-                {% endfor %}
-              </select>
-            </div>
-            <button class="button button--primary" type="submit">선택 목록에 추가</button>
-          </form>
-        {% else %}
-          <p class="selection-add__status" role="status">현재 목록에 더 추가할 수 있는 공개 품목이 없습니다.</p>
-        {% endif %}
-      </section>
-
       {% if items %}
-        <section class="selection-section" aria-labelledby="selection-heading">
+        <section class="selection-section selection-results-first" aria-labelledby="selection-heading">
           <div class="section-heading selection-section__heading">
-            <h2 id="selection-heading">최근 조사값</h2>
+            <div>
+              <p class="eyebrow">현재 선택</p>
+              <h2 id="selection-heading">품목별 최근 조사값</h2>
+            </div>
             {% if result_count_label %}<p class="result-count">{{ result_count_label }}</p>{% endif %}
           </div>
           {% if clear_url %}
@@ -124,6 +93,40 @@
           </div>
         </section>
       {% endif %}
+
+      <section class="selection-add" aria-labelledby="selection-add-heading">
+        <div class="selection-add__heading">
+          <p class="eyebrow">선택 목록 편집</p>
+          <h2 id="selection-add-heading">품목 추가</h2>
+          <p>품종·등급·판매 단위를 확인하고 한 품목을 더 담으세요.</p>
+        </div>
+
+        {% if selection_limit_reached %}
+          <p class="selection-add__status" role="status">
+            <strong>다섯 품목을 모두 선택했습니다.</strong>
+            다른 품목을 담으려면 먼저 목록에서 하나를 빼세요.
+          </p>
+        {% elif can_add_selection and selection_candidates %}
+          <form class="selection-add__form" action="{{ selection_form_action|default:'/selection/' }}" method="get">
+            {% for item in items %}
+              <input type="hidden" name="series" value="{{ item.series_value }}">
+            {% endfor %}
+            <div class="field-group">
+              <label for="selection-add-item">추가할 품목</label>
+              <p id="selection-add-hint" class="field-hint">현재 목록의 마지막에 추가합니다. 최대 다섯 품목까지 담을 수 있습니다.</p>
+              <select id="selection-add-item" name="series" required aria-describedby="selection-add-hint">
+                <option value="">품목을 선택하세요</option>
+                {% for candidate in selection_candidates %}
+                  <option value="{{ candidate.value }}">{{ candidate.label }}</option>
+                {% endfor %}
+              </select>
+            </div>
+            <button class="button button--primary" type="submit">선택 목록에 추가</button>
+          </form>
+        {% else %}
+          <p class="selection-add__status" role="status">현재 목록에 더 추가할 수 있는 공개 품목이 없습니다.</p>
+        {% endif %}
+      </section>
     {% endif %}
   {% endif %}
 {% endblock %}
diff --git a/grocery/tests/test_vnext_public_templates.py b/grocery/tests/test_vnext_public_templates.py
index 8ec7072..b8c7474 100644
--- a/grocery/tests/test_vnext_public_templates.py
+++ b/grocery/tests/test_vnext_public_templates.py
@@ -85,6 +85,65 @@ def test_history_renders_supplementary_chart_and_exact_month_ledger() -> None:
     assert 'style="' not in html
 
 
+def test_history_leads_with_summary_and_opens_only_the_latest_year() -> None:
+    latest = {
+        "available": True,
+        "period_iso": "2026-07",
+        "period_label": "2026년 7월",
+        "mean_machine": "8000",
+        "mean_label": "8,000원",
+        "minimum_machine": "7000",
+        "minimum_label": "7,000원",
+        "maximum_machine": "9000",
+        "maximum_label": "9,000원",
+    }
+    older = {
+        **latest,
+        "period_iso": "2025-12",
+        "period_label": "2025년 12월",
+        "mean_machine": "7600",
+        "mean_label": "7,600원",
+    }
+    context = historical_base("history_state")
+    context.update(
+        {
+            "selected_region": {"label": "서울"},
+            "selected_range": {"label": "36개월"},
+            "monthly_points": [older, latest],
+            "history_summary": {
+                "latest": latest,
+                "lowest": older,
+                "highest": latest,
+            },
+            "history_year_groups": [
+                {
+                    "year": "2026",
+                    "label": "2026년",
+                    "is_latest": True,
+                    "open": True,
+                    "points": [latest],
+                },
+                {
+                    "year": "2025",
+                    "label": "2025년",
+                    "is_latest": False,
+                    "open": False,
+                    "points": [older],
+                },
+            ],
+        }
+    )
+
+    html = render_to_string("grocery/history.html", context)
+
+    assert 'class="history-summary"' in html
+    assert "최근 월평균" in html and "가장 낮은 월평균" in html
+    assert '<details class="history-year history-year--latest" open>' in html
+    assert html.index("2026년") < html.index("2025년")
+    assert html.count("<details") == 2
+    assert html.count(" open>") == 1
+
+
 def test_region_and_market_pages_keep_provider_facts_distinct() -> None:
     regions = historical_base("regions_state")
     regions.update(
@@ -113,6 +172,14 @@ def test_region_and_market_pages_keep_provider_facts_distinct() -> None:
             "selected_region": {"label": "서울"},
             "selected_date": {"iso": "2026-08-29", "label": "2026년 8월 29일"},
             "date_options": [{"value": "2026-08-29", "label": "2026년 8월 29일", "selected": True}],
+            "market_summary": {
+                "total_count": 31,
+                "total_count_label": "31곳",
+                "minimum_machine": "7900",
+                "minimum_label": "7,900원",
+                "maximum_machine": "8600",
+                "maximum_label": "8,600원",
+            },
             "market_rows": [
                 {
                     "market_name": "양곡시장",
@@ -132,6 +199,8 @@ def test_region_and_market_pages_keep_provider_facts_distinct() -> None:
     assert "시장별 값 보기" in region_html
     assert "각 값은 개별 시장 조사값이며 지역 평균이 아닙니다." in market_html
     assert "시장별 소매 조사값입니다" in market_html
+    assert 'class="market-summary"' in market_html
+    assert all(value in market_html for value in ("31곳", "7,900원", "8,600원"))
 
 
 def test_historical_blocking_states_hide_controls_and_fact_ledgers() -> None:
diff --git a/grocery/tests/test_vnext_selection_template.py b/grocery/tests/test_vnext_selection_template.py
index bdd6338..0d28697 100644
--- a/grocery/tests/test_vnext_selection_template.py
+++ b/grocery/tests/test_vnext_selection_template.py
@@ -69,6 +69,9 @@ def test_selection_add_form_appends_to_canonical_url_order() -> None:
     second = html.index('name="series" value="22222222-2222-2222-2222-222222222222"')
     candidate = html.index('id="selection-add-item"')
     assert first < second < candidate
+    assert html.index('class="selection-section selection-results-first"') < html.index(
+        'class="selection-add"'
+    )
     assert 'class="selection-add__form" action="/selection/" method="get"' in html
     assert 'name="series" required aria-describedby="selection-add-hint"' in html
     assert "양파 · 일반 · 상품 · 1kg" in html
diff --git a/templates/400.html b/templates/400.html
index e44aba6..583b05b 100644
--- a/templates/400.html
+++ b/templates/400.html
@@ -4,8 +4,11 @@
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow error-page__eyebrow">초록장부 안내</p>
     <h1 id="error-heading">요청 내용을 확인하세요</h1>
-    <p>입력한 주소나 검색 조건을 처리할 수 없습니다. 내용을 확인한 뒤 다시 시도하세요.</p>
-    <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
+    <p class="error-page__summary">입력한 주소나 검색 조건을 처리할 수 없습니다. 공개 목록에서 다시 찾아보세요.</p>
+    <div class="error-page__actions">
+      <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    </div>
   </section>
 {% endblock %}
diff --git a/templates/403.html b/templates/403.html
index 9edffc4..3dacb87 100644
--- a/templates/403.html
+++ b/templates/403.html
@@ -4,8 +4,11 @@
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow error-page__eyebrow">초록장부 안내</p>
     <h1 id="error-heading">이 페이지를 볼 수 없습니다</h1>
-    <p>공개된 조사값은 목록에서 확인할 수 있습니다.</p>
-    <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    <p class="error-page__summary">요청한 화면은 공개되어 있지 않습니다. 공개된 조사값은 목록에서 확인할 수 있습니다.</p>
+    <div class="error-page__actions">
+      <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    </div>
   </section>
 {% endblock %}
diff --git a/templates/404.html b/templates/404.html
index 58299b1..d1c9c45 100644
--- a/templates/404.html
+++ b/templates/404.html
@@ -4,8 +4,11 @@
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
+    <p class="eyebrow error-page__eyebrow">초록장부 안내</p>
     <h1 id="error-heading">페이지를 찾을 수 없습니다</h1>
-    <p>주소가 바뀌었거나 현재 공개 목록에 없는 항목입니다.</p>
-    <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    <p class="error-page__summary">주소가 바뀌었거나 현재 공개 목록에 없는 품목입니다.</p>
+    <div class="error-page__actions">
+      <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    </div>
   </section>
 {% endblock %}
diff --git a/templates/500.html b/templates/500.html
index 5b6d34a..1c96135 100644
--- a/templates/500.html
+++ b/templates/500.html
@@ -4,8 +4,11 @@
 
 {% block content %}
   <section class="error-page" role="alert" aria-labelledby="error-heading">
+    <p class="eyebrow error-page__eyebrow">초록장부 안내</p>
     <h1 id="error-heading">페이지를 표시하지 못했습니다</h1>
-    <p>요청을 완료하지 못했습니다. 잠시 후 다시 시도하세요.</p>
-    <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
+    <p class="error-page__summary">조사값을 표시하지 못했습니다. 잠시 후 공개 목록에서 다시 시도하세요.</p>
+    <div class="error-page__actions">
+      <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
+    </div>
   </section>
 {% endblock %}


