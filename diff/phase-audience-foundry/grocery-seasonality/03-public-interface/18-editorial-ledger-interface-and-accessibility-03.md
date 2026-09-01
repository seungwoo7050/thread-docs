## `feat: redesign the public SSR experience`

diff --git a/grocery/templates/grocery/_state_notice.html b/grocery/templates/grocery/_state_notice.html
index 4fc6008..896f740 100644
--- a/grocery/templates/grocery/_state_notice.html
+++ b/grocery/templates/grocery/_state_notice.html
@@ -2,43 +2,42 @@
   <section class="state-notice state-notice--info" role="status" aria-live="polite" aria-atomic="true">
     <span class="state-notice__symbol" aria-hidden="true">…</span>
     <div>
-      <h2 class="state-notice__title">자료를 불러오는 중</h2>
-      <p>{{ state_message|default:"검토되어 공개된 조사값을 확인하고 있습니다." }}</p>
+      <h2 class="state-notice__title">조사 자료를 불러오고 있습니다</h2>
+      <p>공개된 자료를 확인하는 동안 잠시 기다려 주세요.</p>
     </div>
   </section>
 {% elif state == "empty" %}
   <section class="state-notice state-notice--neutral" role="status">
     <span class="state-notice__symbol" aria-hidden="true">○</span>
     <div>
-      <h2 class="state-notice__title">조건에 맞는 항목 없음</h2>
-      <p>{{ state_message|default:"검색어를 줄이거나 다른 부류를 선택해 보세요." }}</p>
+      <h2 class="state-notice__title">검색 결과가 없습니다</h2>
+      <p>품목명을 바꾸거나 다른 부류를 선택하세요.</p>
+      <a class="button button--secondary" href="{{ recovery_url|default:'/' }}">전체 항목 보기</a>
     </div>
   </section>
 {% elif state == "unavailable" %}
   <section class="state-notice state-notice--warning" role="status">
     <span class="state-notice__symbol" aria-hidden="true">!</span>
     <div>
-      <h2 class="state-notice__title">공개 조사값 없음</h2>
-      <p>{{ state_message|default:"현재 공개할 수 있는 검토 완료 자료가 없습니다." }}</p>
+      <h2 class="state-notice__title">아직 공개된 조사 자료가 없습니다</h2>
+      <p>검토를 마친 자료가 공개되면 이곳에 표시됩니다.</p>
     </div>
   </section>
 {% elif state == "stale" %}
   <section class="state-notice state-notice--warning" role="status">
     <span class="state-notice__symbol" aria-hidden="true">!</span>
     <div>
-      <h2 class="state-notice__title">마지막 검토 자료 표시 중</h2>
-      <p>
-        {{ state_message|default:"새 수집 상태를 확인하고 있습니다. 아래에는 마지막으로 검토해 공개한 조사값을 표시합니다." }}
-      </p>
+      <h2 class="state-notice__title">마지막 공개 자료를 표시합니다</h2>
+      <p>최근 자료 확인이 필요합니다. 마지막으로 검토를 마친 조사값을 표시합니다.</p>
     </div>
   </section>
 {% elif state == "server_error" %}
   <section class="state-notice state-notice--error" role="alert">
     <span class="state-notice__symbol" aria-hidden="true">×</span>
     <div>
-      <h2 class="state-notice__title">자료를 표시하지 못함</h2>
-      <p>{{ state_message|default:"잠시 후 다시 시도해 주세요." }}</p>
-      {% if retry_url %}<a class="button button--secondary" href="{{ retry_url }}">다시 시도</a>{% endif %}
+      <h2 class="state-notice__title">조사 자료를 불러오지 못했습니다</h2>
+      <p>잠시 후 다시 불러오세요.</p>
+      {% if retry_url %}<a class="button button--secondary" href="{{ retry_url }}">다시 불러오기</a>{% endif %}
     </div>
   </section>
 {% endif %}
diff --git a/grocery/templates/grocery/catalog.html b/grocery/templates/grocery/catalog.html
index 3c01cf6..1aed3e9 100644
--- a/grocery/templates/grocery/catalog.html
+++ b/grocery/templates/grocery/catalog.html
@@ -1,27 +1,31 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}채소·과일 조사값 | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}채소·과일 소매 조사값 | 초록장부{% endblock %}
 
 {% block content %}
-  <header class="page-heading">
+  <header class="page-heading page-heading--catalog">
     <p class="eyebrow">KAMIS 소매 조사 평균</p>
-    <h1>채소·과일 조사값</h1>
+    <h1>채소·과일 소매 조사값</h1>
     <p class="page-heading__summary">
-      공식 품목명으로 찾아보고, 한 품목의 품종·등급·판매 단위가 같은 제공값만 확인할 수
-      있습니다.
+      품목을 둘러보거나 이름으로 찾아보세요. 품종·등급·판매 단위·조사 범위가 같은 값만
+      비교합니다.
     </p>
   </header>
 
-  <section class="search-panel" aria-labelledby="catalog-search-heading">
-    <h2 id="catalog-search-heading">품목 찾기</h2>
+  {% if catalog_state != "loading" and catalog_state != "unavailable" and catalog_state != "server_error" %}
+    <section class="search-panel" aria-labelledby="catalog-search-heading">
+    <div class="search-panel__heading">
+      <h2 id="catalog-search-heading">품목 찾기</h2>
+    </div>
+
     {% if query_error or category_error %}
       <div id="search-error" class="form-error" role="alert">
-        <span aria-hidden="true">!</span>
+        <span class="form-error__symbol" aria-hidden="true">!</span>
         <div class="form-error__content">
-          <p><strong>입력 확인:</strong> 입력 내용을 고친 뒤 다시 검색해 주세요.</p>
+          <p class="form-error__title"><strong>입력 내용을 확인하세요</strong></p>
           <ul>
             {% if query_error %}
-              <li>{{ query_error }} <a class="form-error__link" href="#catalog-query">검색어 입력으로 이동</a></li>
+              <li>{{ query_error }} <a class="form-error__link" href="#catalog-query">품목명 입력으로 이동</a></li>
             {% endif %}
             {% if category_error %}
               <li>{{ category_error }} <a class="form-error__link" href="{{ form_action }}">부류 선택 초기화</a></li>
@@ -30,10 +34,30 @@
         </div>
       </div>
     {% endif %}
+
+    {% if categories %}
+      <nav class="category-nav" aria-label="부류 선택">
+        <ul class="segment-list">
+          {% for category in categories %}
+            <li>
+              <a
+                class="segment{% if category.selected %} segment--selected{% endif %}"
+                href="{{ category.url }}"
+                {% if category.selected %}aria-current="page"{% endif %}
+              >
+                {% if category.selected %}<span class="segment__selected-mark" aria-hidden="true">✓</span>{% endif %}
+                <span>{{ category.label }}</span>
+              </a>
+            </li>
+          {% endfor %}
+        </ul>
+      </nav>
+    {% endif %}
+
     <form class="search-form" action="{{ form_action|default:'' }}" method="get" role="search">
       <div class="field-group">
-        <label for="catalog-query">공식 품목명</label>
-        <p id="catalog-query-hint" class="field-hint">예: 배추, 사과</p>
+        <label for="catalog-query">품목명</label>
+        <p id="catalog-query-hint" class="field-hint">KAMIS 공식 품목명으로 검색하세요. 예: 배추, 사과</p>
         <input
           id="catalog-query"
           name="q"
@@ -50,85 +74,127 @@
       {% endif %}
       <button class="button button--primary" type="submit">검색</button>
     </form>
-
-    {% if categories %}
-      <nav class="category-nav" aria-label="부류 선택">
-        <ul class="chip-list">
-          {% for category in categories %}
-            <li>
-              <a
-                class="chip{% if category.selected %} chip--selected{% endif %}"
-                href="{{ category.url }}"
-                {% if category.selected %}aria-current="page"{% endif %}
-              >
-                {% if category.selected %}<span class="chip__selected-mark" aria-hidden="true">✓</span>{% endif %}
-                <span>{{ category.label }}</span>
-              </a>
-            </li>
-          {% endfor %}
-        </ul>
-      </nav>
-    {% endif %}
-  </section>
+    </section>
+  {% endif %}
 
   {% if catalog_state == "validation" %}
   {% elif catalog_state == "loading" or catalog_state == "unavailable" or catalog_state == "server_error" %}
-    {% include "grocery/_state_notice.html" with state=catalog_state state_message=status_message retry_url=retry_url %}
+    {% include "grocery/_state_notice.html" with state=catalog_state retry_url=retry_url %}
   {% else %}
     {% if catalog_state == "stale" %}
-      {% include "grocery/_state_notice.html" with state="stale" state_message=status_message %}
+      {% include "grocery/_state_notice.html" with state="stale" %}
     {% endif %}
 
     {% if results %}
       <section class="catalog-results" aria-labelledby="catalog-results-heading">
-        <div class="section-heading">
+        <div class="section-heading catalog-results__heading">
           <h2 id="catalog-results-heading">공개 항목</h2>
           {% if result_count_label %}<p class="result-count" aria-live="polite">{{ result_count_label }}</p>{% endif %}
         </div>
-        <ul class="result-list">
-          {% for item in results %}
-            <li>
-              <article class="result-card">
-                <a class="result-card__link" href="{{ item.url }}">
-                  <div class="result-card__heading">
-                    <p class="result-card__category">{{ item.category_label }}</p>
-                    <h3>{{ item.item_name }}</h3>
-                    <p class="result-card__identity">
-                      {{ item.variety_name }} · {{ item.grade_name }} · {{ item.unit_label }}
-                    </p>
-                  </div>
-                  <dl class="result-card__facts">
-                    <div>
-                      <dt>조사일 평균</dt>
-                      <dd>{{ item.current_price_label }}</dd>
-                    </div>
-                    <div>
-                      <dt>조사일</dt>
-                      <dd>
-                        {% if item.source_date_iso %}<time datetime="{{ item.source_date_iso }}">{% endif %}
-                        {{ item.source_date_label }}
-                        {% if item.source_date_iso %}</time>{% endif %}
-                      </dd>
-                    </div>
-                    <div>
-                      <dt>자료 상태</dt>
-                      <dd>
-                        <span class="status-text status-text--{{ item.freshness_state|default:'current' }}">
-                          <span aria-hidden="true">{% if item.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
-                          {{ item.freshness_label }}
-                        </span>
-                      </dd>
-                    </div>
-                  </dl>
-                  <span class="result-card__action" aria-hidden="true">자세히 보기 →</span>
-                </a>
-              </article>
-            </li>
+
+        {% if publication %}
+          <dl class="publication-summary" aria-label="공개 자료 정보">
+            <div>
+              <dt>KAMIS 자료 확인 일시</dt>
+              <dd>
+                {% if publication.checked_at_iso %}<time datetime="{{ publication.checked_at_iso }}">{% endif %}
+                {{ publication.checked_at_display }}
+                {% if publication.checked_at_iso %}</time>{% endif %}
+              </dd>
+            </div>
+            <div>
+              <dt>자료 상태</dt>
+              <dd>
+                <span class="status-text status-text--{{ publication.freshness_state|default:'current' }}">
+                  <span aria-hidden="true">{% if publication.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
+                  {{ publication.freshness_label }}
+                </span>
+              </dd>
+            </div>
+          </dl>
+        {% endif %}
+
+        {% regroup results by category_label as result_groups %}
+        <div class="catalog-ledger">
+          {% for group in result_groups %}
+            <section class="catalog-group" aria-labelledby="catalog-group-{{ forloop.counter }}">
+              <h3 id="catalog-group-{{ forloop.counter }}" class="catalog-group__heading">{{ group.grouper }}</h3>
+              <div class="ledger-column-head" aria-hidden="true">
+                <span>품목·조건</span>
+                <span>소매 조사 평균</span>
+                <span>1주 비교</span>
+                <span>조사일</span>
+                <span>이동</span>
+              </div>
+              <ul class="ledger-list">
+                {% for item in group.list %}
+                  <li class="ledger-row">
+                    <article class="ledger-entry">
+                      <header class="ledger-entry__heading">
+                        <h4>
+                          <a
+                            class="ledger-entry__link"
+                            href="{{ item.url }}"
+                            aria-label="{{ item.item_name }} 상세 보기"
+                          >{{ item.item_name }}</a>
+                        </h4>
+                        <p class="ledger-entry__identity">
+                          <span>품종 {{ item.variety_name }}</span>
+                          <span>등급 {{ item.grade_name }}</span>
+                          <span>판매 단위 {{ item.unit_label }}</span>
+                        </p>
+                      </header>
+
+                      <dl class="ledger-fact ledger-fact--price">
+                        <dt>조사일 평균</dt>
+                        <dd>{{ item.current_price_label }}</dd>
+                      </dl>
+
+                      <dl class="ledger-fact ledger-fact--comparison">
+                        <dt>1주 비교</dt>
+                        <dd>
+                          {% if item.week_comparison.available %}
+                            <span class="direction direction--{{ item.week_comparison.direction_code|lower }}">
+                              <span class="direction__symbol" aria-hidden="true">
+                                {% if item.week_comparison.direction_code == "LOWER" %}↓{% elif item.week_comparison.direction_code == "HIGHER" %}↑{% else %}＝{% endif %}
+                              </span>
+                              <span>
+                                {% if item.week_comparison.direction_code == "EQUAL" %}
+                                  1주 전 제공값과 같음 ({{ item.week_comparison.percentage_display }})
+                                {% else %}
+                                  1주 전 제공값보다 {{ item.week_comparison.difference_display }}
+                                  {{ item.week_comparison.direction_label }} ({{ item.week_comparison.percentage_display }})
+                                {% endif %}
+                              </span>
+                            </span>
+                          {% else %}
+                            <span class="status-text status-text--unavailable">
+                              <span aria-hidden="true">○</span> 1주 전 비교값 없음
+                            </span>
+                          {% endif %}
+                        </dd>
+                      </dl>
+
+                      <dl class="ledger-fact ledger-fact--date">
+                        <dt>조사일</dt>
+                        <dd>
+                          {% if item.source_date_iso %}<time datetime="{{ item.source_date_iso }}">{% endif %}
+                          {{ item.source_date_label }}
+                          {% if item.source_date_iso %}</time>{% endif %}
+                        </dd>
+                      </dl>
+
+                      <span class="ledger-entry__action" aria-hidden="true">→</span>
+                    </article>
+                  </li>
+                {% endfor %}
+              </ul>
+            </section>
           {% endfor %}
-        </ul>
+        </div>
       </section>
     {% else %}
-      {% include "grocery/_state_notice.html" with state="empty" state_message=status_message %}
+      {% include "grocery/_state_notice.html" with state="empty" recovery_url=form_action %}
     {% endif %}
   {% endif %}
 {% endblock %}
diff --git a/grocery/templates/grocery/detail.html b/grocery/templates/grocery/detail.html
index 81529ac..792cfb3 100644
--- a/grocery/templates/grocery/detail.html
+++ b/grocery/templates/grocery/detail.html
@@ -1,165 +1,242 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}{{ series.item_name|default:"조사값 상세" }} | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}{{ series.item_name|default:"조사값 상세" }} | 초록장부{% endblock %}
 
 {% block content %}
-  <nav class="breadcrumb" aria-label="현재 위치">
-    <ol>
-      <li><a href="{{ catalog_url|default:'/' }}">채소·과일 조사값</a></li>
-      <li aria-current="page">{{ series.item_name|default:"상세" }}</li>
-    </ol>
+  <nav class="breadcrumb" aria-label="목록으로 돌아가기">
+    <a href="{{ catalog_url|default:'/' }}">← 채소·과일 소매 조사값</a>
   </nav>
 
-  <header class="page-heading page-heading--detail">
-    <p class="eyebrow">{{ series.category_label|default:"공개 조사 항목" }}</p>
-    <h1>{{ series.item_name|default:"조사값 상세" }}</h1>
-    {% if series.variety_name or series.grade_name or series.unit_label %}
-      <p class="page-heading__summary identity-summary">
-        {% if series.variety_name %}<span>품종 {{ series.variety_name }}</span>{% endif %}
-        {% if series.grade_name %}<span>등급 {{ series.grade_name }}</span>{% endif %}
-        {% if series.unit_label %}<span>판매 단위 {{ series.unit_label }}</span>{% endif %}
-      </p>
+  <header class="detail-intro">
+    <div class="detail-intro__identity">
+      <p class="eyebrow">{{ series.category_label|default:"공개 조사 항목" }}</p>
+      <h1>{{ series.item_name|default:"조사값 상세" }}</h1>
+      {% if series.variety_name or series.grade_name or series.unit_label or provenance.coverage_label %}
+        <dl class="detail-signature" aria-label="품목 조건 요약">
+          {% if series.variety_name %}<div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>{% endif %}
+          {% if series.grade_name %}<div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>{% endif %}
+          {% if series.unit_label %}<div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>{% endif %}
+          {% if provenance.coverage_label %}<div><dt>조사 범위</dt><dd>{{ provenance.coverage_label }}</dd></div>{% endif %}
+        </dl>
+      {% endif %}
+    </div>
+
+    {% if detail_state != "loading" and detail_state != "unavailable" and detail_state != "server_error" %}
+      <div class="current-price" aria-labelledby="current-price-heading">
+        <div class="current-price__heading">
+          <p class="eyebrow">KAMIS 소매 조사 평균</p>
+          <h2 id="current-price-heading">조사일 평균</h2>
+        </div>
+        <p class="current-price__value">
+          {% if series.current_price_machine %}<data value="{{ series.current_price_machine }}">{% endif %}
+          {{ series.current_price_label }}
+          {% if series.current_price_machine %}</data>{% endif %}
+        </p>
+        <p class="current-price__date">
+          조사일
+          {% if provenance.source_date_iso %}<time datetime="{{ provenance.source_date_iso }}">{% endif %}
+          {{ provenance.source_date_label }}
+          {% if provenance.source_date_iso %}</time>{% endif %}
+        </p>
+        <p class="current-price__note">개별 판매처의 판매 금액이 아닌 KAMIS 소매 조사 평균입니다.</p>
+      </div>
     {% endif %}
   </header>
 
   {% if detail_state == "loading" or detail_state == "unavailable" or detail_state == "server_error" %}
-    {% include "grocery/_state_notice.html" with state=detail_state state_message=status_message retry_url=retry_url %}
+    {% include "grocery/_state_notice.html" with state=detail_state retry_url=retry_url %}
   {% else %}
     {% if detail_state == "stale" %}
-      {% include "grocery/_state_notice.html" with state="stale" state_message=status_message %}
+      {% include "grocery/_state_notice.html" with state="stale" %}
     {% endif %}
 
-    <section class="current-price" aria-labelledby="current-price-heading">
-      <div>
-        <p class="eyebrow">KAMIS 소매 조사 평균</p>
-        <h2 id="current-price-heading">조사일 평균</h2>
-      </div>
-      <p class="current-price__value">
-        {% if series.current_price_machine %}<data value="{{ series.current_price_machine }}">{% endif %}
-        {{ series.current_price_label }}
-        {% if series.current_price_machine %}</data>{% endif %}
-      </p>
-      <p class="current-price__date">
-        조사일
-        {% if provenance.source_date_iso %}<time datetime="{{ provenance.source_date_iso }}">{% endif %}
-        {{ provenance.source_date_label }}
-        {% if provenance.source_date_iso %}</time>{% endif %}
-      </p>
-    </section>
+    <div class="detail-layout">
+      <div class="detail-main">
+        <section class="comparison-section" aria-labelledby="comparison-heading">
+          <div class="section-heading section-heading--stacked">
+            <p class="eyebrow">같은 조사 조건 안에서</p>
+            <h2 id="comparison-heading">KAMIS 비교값</h2>
+            <p>현재 조사일 평균과 KAMIS가 제공한 이전 값을 비교합니다.</p>
+          </div>
+
+          <div class="comparison-ledger">
+            <div class="comparison-column-head" aria-hidden="true">
+              <span>기간</span>
+              <span>KAMIS 제공값</span>
+              <span>조사일 평균과 차이</span>
+              <span>비교 기준일</span>
+            </div>
 
-    <section class="identity-panel" aria-labelledby="identity-heading">
-      <h2 id="identity-heading">비교 대상의 정확한 조건</h2>
-      <dl class="definition-grid">
-        <div><dt>품목</dt><dd>{{ series.item_name }}</dd></div>
-        <div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>
-        <div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>
-        <div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>
-        <div><dt>조사범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
-      </dl>
-    </section>
+            <ul class="comparison-list">
+              {% for comparison in comparisons %}
+                <li class="comparison-row{% if not comparison.available %} comparison-row--unavailable{% endif %}">
+                  <dl class="comparison-row__facts">
+                    <div class="comparison-field comparison-field--period">
+                      <dt>기간</dt>
+                      <dd><strong>{{ comparison.period_label }}</strong></dd>
+                    </div>
 
-    <section class="comparison-section" aria-labelledby="comparison-heading">
-      <div class="section-heading section-heading--stacked">
-        <h2 id="comparison-heading">비교 제공값</h2>
-        <p>source가 같은 series에 제공한 값과 현재 조사 평균의 산술 차이입니다.</p>
+                    <div class="comparison-field comparison-field--reference">
+                      <dt>KAMIS 제공값</dt>
+                      <dd>
+                        {% if comparison.available %}
+                          <strong>{{ comparison.reference_price_display }}</strong>
+                        {% else %}
+                          <span class="status-text status-text--unavailable">
+                            <span aria-hidden="true">○</span> 비교값 없음
+                          </span>
+                        {% endif %}
+                      </dd>
+                    </div>
+
+                    <div class="comparison-field comparison-field--difference">
+                      <dt>조사일 평균과 차이</dt>
+                      <dd>
+                        {% if comparison.available %}
+                          <p class="direction direction--{{ comparison.direction_code|lower }}">
+                            <span class="direction__symbol" aria-hidden="true">
+                              {% if comparison.direction_code == "LOWER" %}↓{% elif comparison.direction_code == "HIGHER" %}↑{% else %}＝{% endif %}
+                            </span>
+                            <span>
+                              {% if comparison.direction_code == "EQUAL" %}
+                                조사일 평균이 비교값과 같음 ({{ comparison.percentage_display }})
+                              {% else %}
+                                조사일 평균이 비교값보다 {{ comparison.difference_display }}
+                                {{ comparison.direction_label }} ({{ comparison.percentage_display }})
+                              {% endif %}
+                            </span>
+                          </p>
+
+                          {% if comparison.microbar %}
+                            <svg
+                              class="comparison-meter comparison-meter--{{ comparison.direction_code|lower }}"
+                              viewBox="0 0 100 8"
+                              aria-hidden="true"
+                              focusable="false"
+                            >
+                              <line class="comparison-meter__rail" x1="0" y1="4" x2="100" y2="4"/>
+                              <line class="comparison-meter__zero" x1="50" y1="1" x2="50" y2="7"/>
+                              {% if comparison.microbar.is_equal %}
+                                <circle class="comparison-meter__point" cx="{{ comparison.microbar.cap_x }}" cy="4" r="2.5"/>
+                              {% else %}
+                                <rect
+                                  class="comparison-meter__value"
+                                  x="{{ comparison.microbar.x }}"
+                                  y="2"
+                                  width="{{ comparison.microbar.width }}"
+                                  height="4"
+                                  rx="2"
+                                />
+                                <circle class="comparison-meter__point" cx="{{ comparison.microbar.cap_x }}" cy="4" r="2"/>
+                                {% if comparison.microbar.capped %}
+                                  <line
+                                    class="comparison-meter__cap"
+                                    x1="{{ comparison.microbar.cap_x }}"
+                                    y1="1"
+                                    x2="{{ comparison.microbar.cap_x }}"
+                                    y2="7"
+                                  />
+                                {% endif %}
+                              {% endif %}
+                            </svg>
+                          {% endif %}
+                        {% else %}
+                          KAMIS가 이 기간의 비교값을 제공하지 않았습니다.
+                        {% endif %}
+                      </dd>
+                    </div>
+
+                    <div class="comparison-field comparison-field--date">
+                      <dt>비교 기준일</dt>
+                      <dd>
+                        {% if comparison.reference_date_unavailable %}
+                          KAMIS에서 제공하지 않음
+                        {% else %}
+                          {{ comparison.reference_date_display }}
+                        {% endif %}
+                      </dd>
+                    </div>
+                  </dl>
+                </li>
+              {% empty %}
+                <li class="comparison-row comparison-row--unavailable">
+                  <p class="comparison-row__empty">
+                    <span class="status-text status-text--unavailable"><span aria-hidden="true">○</span> 비교값 없음</span>
+                  </p>
+                </li>
+              {% endfor %}
+            </ul>
+          </div>
+        </section>
       </div>
-      <ul class="comparison-list">
-        {% for comparison in comparisons %}
-          <li class="comparison-card{% if not comparison.available %} comparison-card--unavailable{% endif %}">
-            <h3>{{ comparison.period_label }}</h3>
-            {% if comparison.available %}
-              <p class="comparison-card__reference">
-                제공값 <strong>{{ comparison.reference_value_label }}</strong>
-              </p>
-              <p class="direction direction--{{ comparison.direction_code|lower }}">
-                <span class="direction__symbol" aria-hidden="true">
-                  {% if comparison.direction_code == "LOWER" %}↓{% elif comparison.direction_code == "HIGHER" %}↑{% else %}＝{% endif %}
-                </span>
-                <span>
-                  {{ comparison.difference_label }} {{ comparison.direction_label }}
-                  ({{ comparison.percentage_label }})
-                </span>
-              </p>
-            {% else %}
-              <p class="status-text status-text--unavailable">
-                <span aria-hidden="true">○</span> 비교 정보 없음
-              </p>
-              {% if comparison.unavailable_reason_label %}
-                <p class="comparison-card__reason">{{ comparison.unavailable_reason_label }}</p>
-              {% endif %}
-            {% endif %}
-            <p class="comparison-card__date">
-              {% if comparison.reference_date_available %}
-                비교 기준일
-                {% if comparison.reference_date_iso %}<time datetime="{{ comparison.reference_date_iso }}">{% endif %}
-                {{ comparison.reference_date_label }}
-                {% if comparison.reference_date_iso %}</time>{% endif %}
-              {% else %}
-                source가 비교 기준일을 별도로 제공하지 않음
-              {% endif %}
-            </p>
-          </li>
-        {% empty %}
-          <li class="comparison-card comparison-card--unavailable">
-            <p class="status-text status-text--unavailable"><span aria-hidden="true">○</span> 비교 정보 없음</p>
-          </li>
-        {% endfor %}
-      </ul>
-    </section>
 
-    <aside class="provenance" aria-labelledby="provenance-heading">
-      <h2 id="provenance-heading">출처와 자료 상태</h2>
-      <dl class="definition-grid definition-grid--provenance">
-        <div>
-          <dt>출처</dt>
-          <dd>
-            {% if provenance.source_url %}
-              <a href="{{ provenance.source_url }}" rel="external noreferrer">{{ provenance.source_name }}</a>
-            {% else %}
-              {{ provenance.source_name }}
-            {% endif %}
-            {% if provenance.dataset_id %}<span class="metadata-detail">데이터셋 {{ provenance.dataset_id }}</span>{% endif %}
-          </dd>
-        </div>
-        <div>
-          <dt>조사일</dt>
-          <dd>
-            {% if provenance.source_date_iso %}<time datetime="{{ provenance.source_date_iso }}">{% endif %}
-            {{ provenance.source_date_label }}
-            {% if provenance.source_date_iso %}</time>{% endif %}
-          </dd>
-        </div>
-        <div><dt>조사범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
-        <div>
-          <dt>마지막 확인</dt>
-          <dd>
-            {% if provenance.checked_at_iso %}<time datetime="{{ provenance.checked_at_iso }}">{% endif %}
-            {{ provenance.checked_at_label }}
-            {% if provenance.checked_at_iso %}</time>{% endif %}
-          </dd>
-        </div>
-        <div>
-          <dt>공개 검토일</dt>
-          <dd>
-            {% if provenance.reviewed_at_iso %}<time datetime="{{ provenance.reviewed_at_iso }}">{% endif %}
-            {{ provenance.reviewed_at_label }}
-            {% if provenance.reviewed_at_iso %}</time>{% endif %}
-          </dd>
-        </div>
-        <div>
-          <dt>자료 상태</dt>
-          <dd>
-            <span class="status-text status-text--{{ provenance.freshness_state|default:'current' }}">
-              <span aria-hidden="true">{% if provenance.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
-              {{ provenance.freshness_label }}
-            </span>
-          </dd>
-        </div>
-      </dl>
-      <p class="provenance__note">
-        명시된 품목·품종·등급·판매 단위·조사범위가 모두 같은 값만 비교합니다. source가
-        제공하지 않은 비교 기준일은 추정하지 않습니다.
-      </p>
-    </aside>
+      <aside class="detail-aside" aria-label="조사 조건과 자료 정보">
+        <section class="identity-panel" aria-labelledby="identity-heading">
+          <h2 id="identity-heading">조사 조건</h2>
+          <dl class="definition-grid">
+            <div><dt>품목</dt><dd>{{ series.item_name }}</dd></div>
+            <div><dt>품종</dt><dd>{{ series.variety_name }}</dd></div>
+            <div><dt>등급</dt><dd>{{ series.grade_name }}</dd></div>
+            <div><dt>판매 단위</dt><dd>{{ series.unit_label }}</dd></div>
+            <div><dt>조사 범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
+          </dl>
+        </section>
+
+        <section class="provenance" aria-labelledby="provenance-heading">
+          <h2 id="provenance-heading">출처와 자료 정보</h2>
+          <dl class="definition-grid definition-grid--provenance">
+            <div>
+              <dt>출처</dt>
+              <dd>
+                {% if provenance.source_url %}
+                  <a href="{{ provenance.source_url }}" rel="external noreferrer">{{ provenance.source_name }}</a>
+                {% else %}
+                  {{ provenance.source_name }}
+                {% endif %}
+                {% if provenance.dataset_id %}<span class="metadata-detail">데이터셋 {{ provenance.dataset_id }}</span>{% endif %}
+              </dd>
+            </div>
+            <div>
+              <dt>조사일</dt>
+              <dd>
+                {% if provenance.source_date_iso %}<time datetime="{{ provenance.source_date_iso }}">{% endif %}
+                {{ provenance.source_date_label }}
+                {% if provenance.source_date_iso %}</time>{% endif %}
+              </dd>
+            </div>
+            <div><dt>조사 범위</dt><dd>{{ provenance.coverage_label }}</dd></div>
+            <div>
+              <dt>KAMIS 자료 확인 일시</dt>
+              <dd>
+                {% if provenance.checked_at_iso %}<time datetime="{{ provenance.checked_at_iso }}">{% endif %}
+                {{ provenance.checked_at_display }}
+                {% if provenance.checked_at_iso %}</time>{% endif %}
+              </dd>
+            </div>
+            <div>
+              <dt>공개 검토 일시</dt>
+              <dd>
+                {% if provenance.reviewed_at_iso %}<time datetime="{{ provenance.reviewed_at_iso }}">{% endif %}
+                {{ provenance.reviewed_at_label }}
+                {% if provenance.reviewed_at_iso %}</time>{% endif %}
+              </dd>
+            </div>
+            <div>
+              <dt>자료 상태</dt>
+              <dd>
+                <span class="status-text status-text--{{ provenance.freshness_state|default:'current' }}">
+                  <span aria-hidden="true">{% if provenance.freshness_state == "stale" %}!{% else %}✓{% endif %}</span>
+                  {{ provenance.freshness_label }}
+                </span>
+              </dd>
+            </div>
+          </dl>
+          <p class="provenance__note">
+            표시된 품목·품종·등급·판매 단위·조사 범위가 모두 같은 값만 비교합니다. KAMIS가
+            비교 기준일을 제공하지 않으면 날짜를 추정하지 않습니다.
+          </p>
+        </section>
+      </aside>
+    </div>
   {% endif %}
 {% endblock %}
diff --git a/grocery/tests/test_public_routes.py b/grocery/tests/test_public_routes.py
index cb22812..9ca65d0 100644
--- a/grocery/tests/test_public_routes.py
+++ b/grocery/tests/test_public_routes.py
@@ -33,7 +33,7 @@ def activate_publication() -> tuple[PublicationRevision, RetailPriceSnapshot, An
     )
     publisher.user_permissions.add(permission)
     publisher = type(publisher)._default_manager.get(pk=publisher.pk)
-    revision = seal_recent_publication(decision.id, "ko-v1")
+    revision = seal_recent_publication(decision.id, "ko-v3")
     transition_recent_publication(
         operation_id=uuid.uuid4(),
         actor=publisher,
@@ -58,7 +58,7 @@ def test_catalog_is_unavailable_before_activation_and_candidate_detail_is_hidden
     )
 
     assert catalog_response.status_code == 200
-    assert "공개 조사값 없음" in catalog_response.content.decode()
+    assert "아직 공개된 조사 자료가 없습니다" in catalog_response.content.decode()
     assert "X-Publication-Fact-Set" not in catalog_response
     assert detail_response.status_code == 404
     assert not PublicationChannel.objects.exists()
@@ -80,17 +80,28 @@ def test_catalog_search_and_detail_use_only_the_active_sealed_revision() -> None
     assert search_response.status_code == 200
     assert empty_response.status_code == 200
     assert detail_response.status_code == 200
+    detail_html = " ".join(detail_response.content.decode().split())
     assert catalog_response.headers["X-Publication-Fact-Set"] == revision.typed_fact_set_sha256
     assert detail_response.headers["X-Publication-Fact-Set"] == revision.typed_fact_set_sha256
     assert snapshot.series.item_name in search_response.content.decode()
-    assert "조건에 맞는 항목 없음" in empty_response.content.decode()
-    assert "KAMIS 소매 조사 평균" in detail_response.content.decode()
-    assert "2,000원 낮음" in detail_response.content.decode()
-    assert "(+25.0%)" in detail_response.content.decode()
-    assert "같음" in detail_response.content.decode()
-    assert "source가 비교 기준일을 별도로 제공하지 않음" in detail_response.content.decode()
-    assert "데이터셋 15156063" in detail_response.content.decode()
+    assert "검색 결과가 없습니다" in empty_response.content.decode()
+    assert "KAMIS 소매 조사 평균" in detail_html
+    assert "조사일 평균이 비교값보다 2,000원 낮음 (-20.0%)" in detail_html
+    assert "(+25.0%)" in detail_html
+    assert "같음" in detail_html
+    assert "KAMIS에서 제공하지 않음" in detail_html
+    assert "데이터셋 15156063" in detail_html
     assert "sessionid" not in catalog_response.cookies
+    assert catalog_response.context["publication"]["freshness_label"] == "KAMIS 자료 확인 완료"
+    assert catalog_response.context["results"][0]["week_comparison"]["period_label"] == (
+        "1주 전 제공값"
+    )
+    assert detail_response.context["publication"] == {
+        "checked_at_iso": detail_response.context["provenance"]["checked_at_iso"],
+        "checked_at_display": detail_response.context["provenance"]["checked_at_display"],
+        "freshness_state": "current",
+        "freshness_label": "KAMIS 자료 확인 완료",
+    }
 
 
 @pytest.mark.django_db
@@ -118,9 +129,10 @@ def test_invalid_mobile_search_input_returns_associated_correction_error() -> No
     assert 'role="alert"' in invalid_html
     assert 'aria-invalid="true"' in invalid_html
     assert 'aria-describedby="catalog-query-hint search-error"' in invalid_html
-    assert "검색어는 80자 이하여야 합니다." in invalid_html
+    assert "입력 내용을 확인하세요" in invalid_html
+    assert "품목명은 80자 이하로 입력하세요." in invalid_html
     assert invalid_query not in invalid_html
-    assert "조건에 맞는 항목 없음" not in invalid_html
+    assert "검색 결과가 없습니다" not in invalid_html
     assert corrected_response.status_code == 200
     assert 'aria-invalid="true"' not in corrected_response.content.decode()
 
@@ -143,7 +155,7 @@ def test_category_validation_is_distinct_and_never_reflects_query_or_choice() ->
     assert 'aria-invalid="true"' not in html
     assert query_marker not in html
     assert category_marker not in html
-    assert "조건에 맞는 항목 없음" not in html
+    assert "검색 결과가 없습니다" not in html
 
 
 @pytest.mark.django_db
@@ -191,7 +203,7 @@ def test_confirmation_age_is_separate_and_preserves_last_known_good() -> None:
     assert stale is not None
     assert stale.revision.id == revision.id
     assert stale.freshness_state == "stale"
-    assert "새 확인 필요" in stale.freshness_label
+    assert stale.freshness_label == "마지막 공개 자료 · 최근 확인 필요"
 
 
 @pytest.mark.django_db
@@ -201,20 +213,39 @@ def test_database_failure_uses_fixed_server_error_without_exception_reflection()
         response = Client().get(reverse("grocery:catalog"))
 
     assert response.status_code == 503
-    assert "자료를 표시하지 못함" in response.content.decode()
+    assert "조사 자료를 불러오지 못했습니다" in response.content.decode()
     assert marker not in response.content.decode()
 
 
 @pytest.mark.django_db
 def test_qa_state_routes_are_hard_disabled_unless_local_setting_is_explicit() -> None:
-    disabled = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": "loading"}))
-    assert disabled.status_code == 404
+    for state in ("loading", "error_400", "error_403", "error_404", "error_500"):
+        disabled = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": state}))
+        assert disabled.status_code == 404
 
     with override_settings(QA_STATE_PREVIEWS_ENABLED=True):
-        for state in ("loading", "empty", "unavailable", "stale", "server_error"):
+        expected_headings = {
+            "loading": "조사 자료를 불러오고 있습니다",
+            "empty": "검색 결과가 없습니다",
+            "unavailable": "아직 공개된 조사 자료가 없습니다",
+            "stale": "마지막 공개 자료를 표시합니다",
+            "server_error": "조사 자료를 불러오지 못했습니다",
+        }
+        for state, heading in expected_headings.items():
             response = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": state}))
             assert response.status_code == (503 if state == "server_error" else 200)
-            assert "로컬 화면 상태 검수용 미리보기" in response.content.decode()
+            assert heading in response.content.decode()
+
+        error_previews = {
+            "error_400": (400, "요청 내용을 확인하세요"),
+            "error_403": (403, "이 페이지를 볼 수 없습니다"),
+            "error_404": (404, "페이지를 찾을 수 없습니다"),
+            "error_500": (500, "페이지를 표시하지 못했습니다"),
+        }
+        for state, (status, heading) in error_previews.items():
+            response = Client().get(reverse("grocery:qa_catalog_state", kwargs={"state": state}))
+            assert response.status_code == status
+            assert f'<h1 id="error-heading">{heading}</h1>' in response.content.decode()
 
 
 @pytest.mark.django_db
diff --git a/grocery/tests/test_public_templates.py b/grocery/tests/test_public_templates.py
index 6235561..bfb96f5 100644
--- a/grocery/tests/test_public_templates.py
+++ b/grocery/tests/test_public_templates.py
@@ -31,6 +31,50 @@ def render(template_name: str, context: Mapping[str, Any] | None = None) -> str:
     return render_to_string(template_name, context or {})
 
 
+def comparison(
+    period_label: str,
+    *,
+    direction_code: str = "LOWER",
+    direction_label: str = "낮음",
+    percentage_display: str = "-20.0%",
+    available: bool = True,
+) -> dict[str, object]:
+    if not available:
+        return {
+            "period_label": period_label,
+            "available": False,
+            "reference_price_display": "",
+            "difference_display": "",
+            "percentage_display": "",
+            "direction_code": "UNAVAILABLE",
+            "direction_label": "",
+            "reference_date_display": "",
+            "reference_date_unavailable": True,
+            "unavailable_reason": "KAMIS가 이 기간의 비교값을 제공하지 않았습니다.",
+            "microbar": None,
+        }
+    is_equal = direction_code == "EQUAL"
+    return {
+        "period_label": period_label,
+        "available": True,
+        "reference_price_display": "10,000원",
+        "difference_display": "2,000원",
+        "percentage_display": percentage_display,
+        "direction_code": direction_code,
+        "direction_label": direction_label,
+        "reference_date_display": "",
+        "reference_date_unavailable": True,
+        "unavailable_reason": "",
+        "microbar": {
+            "x": 50 if is_equal else 30,
+            "width": 0 if is_equal else 20,
+            "capped": False,
+            "cap_x": 50 if is_equal else 30,
+            "is_equal": is_equal,
+        },
+    }
+
+
 def catalog_context(**overrides: object) -> dict[str, object]:
     context: dict[str, object] = {
         "home_url": "/",
@@ -43,7 +87,13 @@ def catalog_context(**overrides: object) -> dict[str, object]:
             {"label": "채소류", "url": "/?category=vegetable", "selected": True},
             {"label": "과일류", "url": "/?category=fruit", "selected": False},
         ],
-        "result_count_label": "공개 항목 1개",
+        "result_count_label": "공개 항목 2개",
+        "publication": {
+            "checked_at_iso": "2026-08-30T09:00:00+09:00",
+            "checked_at_display": "2026년 8월 30일 09:00",
+            "freshness_state": "current",
+            "freshness_label": "KAMIS 자료 확인 완료",
+        },
         "results": [
             {
                 "url": "/series/200-212-00-04/",
@@ -55,9 +105,20 @@ def catalog_context(**overrides: object) -> dict[str, object]:
                 "current_price_label": "8,000원",
                 "source_date_iso": "2026-08-29",
                 "source_date_label": "2026년 8월 29일",
-                "freshness_state": "current",
-                "freshness_label": "공개 조사일 확인됨",
-            }
+                "week_comparison": comparison("1주 전 제공값"),
+            },
+            {
+                "url": "/series/400-411-00-01/",
+                "category_label": "과일류",
+                "item_name": "사과",
+                "variety_name": "후지",
+                "grade_name": "상품",
+                "unit_label": "10개",
+                "current_price_label": "25,000원",
+                "source_date_iso": "2026-08-29",
+                "source_date_label": "2026년 8월 29일",
+                "week_comparison": comparison("1주 전 제공값", available=False),
+            },
         ],
     }
     context.update(overrides)
@@ -79,32 +140,14 @@ def detail_context(**overrides: object) -> dict[str, object]:
             "current_price_label": "8,000원",
         },
         "comparisons": [
-            {
-                "period_label": "1주 전 제공값",
-                "available": True,
-                "reference_value_label": "10,000원",
-                "difference_label": "2,000원",
-                "percentage_label": "-20.0%",
-                "direction_code": "LOWER",
-                "direction_label": "낮음",
-                "reference_date_available": False,
-            },
-            {
-                "period_label": "1개월 전 제공값",
-                "available": True,
-                "reference_value_label": "10,000원",
-                "difference_label": "2,000원",
-                "percentage_label": "-20.0%",
-                "direction_code": "LOWER",
-                "direction_label": "낮음",
-                "reference_date_available": False,
-            },
-            {
-                "period_label": "1년 전 제공값",
-                "available": False,
-                "unavailable_reason_label": "source 응답에 비교값이 없습니다.",
-                "reference_date_available": False,
-            },
+            comparison("1주 전 제공값"),
+            comparison(
+                "1개월 전 제공값",
+                direction_code="EQUAL",
+                direction_label="같음",
+                percentage_display="0.0%",
+            ),
+            comparison("1년 전 제공값", available=False),
         ],
         "provenance": {
             "source_name": "한국농수산식품유통공사 KAMIS 최근일자 도·소매가격정보",
@@ -113,42 +156,66 @@ def detail_context(**overrides: object) -> dict[str, object]:
             "source_date_iso": "2026-08-29",
             "source_date_label": "2026년 8월 29일",
             "coverage_label": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
-            "checked_at_label": "2026년 8월 30일 09:00",
+            "checked_at_display": "2026년 8월 30일 09:00",
             "checked_at_iso": "2026-08-30T09:00:00+09:00",
             "reviewed_at_label": "2026년 8월 30일 09:30",
             "reviewed_at_iso": "2026-08-30T09:30:00+09:00",
             "freshness_state": "current",
-            "freshness_label": "공개 조사일 확인됨",
+            "freshness_label": "KAMIS 자료 확인 완료",
         },
     }
     context.update(overrides)
     return context
 
 
-def test_catalog_renders_semantic_search_and_long_identity() -> None:
+def test_catalog_renders_brand_semantic_search_and_grouped_ledger() -> None:
     html = render("grocery/catalog.html", catalog_context())
 
     assert '<html lang="ko">' in html
     assert 'href="#main-content"' in html
     assert '<main id="main-content"' in html
+    assert 'src="/static/grocery/brand-mark.svg"' in html
+    assert '<span class="brand__name">초록장부</span>' in html
+    assert '<span class="brand__description">채소·과일 소매 조사값</span>' in html
     assert 'role="search"' in html
-    assert '<label for="catalog-query">공식 품목명</label>' in html
+    assert '<label for="catalog-query">품목명</label>' in html
     assert 'name="q"' in html
     assert 'aria-current="page"' in html
-    assert '<span class="chip__selected-mark" aria-hidden="true">✓</span>' in html
+    assert '<span class="segment__selected-mark" aria-hidden="true">✓</span>' in html
+    assert html.count('class="catalog-group"') == 2
+    assert "품목·조건" in html
+    assert "소매 조사 평균" in html
+    assert "1주 비교" in html
+    assert "조사일" in html
+    assert "이동" in html
     assert "아주긴한국어공식품목명이줄바꿈되어야하는배추" in html
     assert "아주긴원문판매단위표시 포기 × 1" in html
-    assert "공개 조사일 확인됨" in html
+    assert 'class="ledger-entry__link"' in html
+    assert 'aria-label="아주긴한국어공식품목명이줄바꿈되어야하는배추 상세 보기"' in html
+    assert '<span class="ledger-entry__action" aria-hidden="true">→</span>' in html
+    assert "KAMIS 자료 확인 완료" in html
 
 
-def test_catalog_validation_error_is_associated_with_input() -> None:
+def test_catalog_shows_one_week_comparison_preview_without_longer_periods() -> None:
+    html = render("grocery/catalog.html", catalog_context())
+
+    assert "1주 전 제공값보다 2,000원 낮음 (-20.0%)" in " ".join(html.split())
+    assert "1주 전 비교값 없음" in html
+    assert "1개월 전" not in html
+    assert "1년 전" not in html
+    assert "10,000원" not in html
+    assert "reference_price_display" not in html
+    assert "comparison-meter" not in html
+
+
+def test_catalog_validation_error_is_associated_with_blank_input() -> None:
     private_query = "잘못된 입력"
     html = render(
         "grocery/catalog.html",
         catalog_context(
             catalog_state="validation",
             query=private_query,
-            query_error="검색어는 80자 이하여야 합니다.",
+            query_error="품목명은 80자 이하로 입력하세요.",
         ),
     )
 
@@ -156,23 +223,24 @@ def test_catalog_validation_error_is_associated_with_input() -> None:
     assert 'role="alert"' in html
     assert 'aria-invalid="true"' in html
     assert 'aria-describedby="catalog-query-hint search-error"' in html
-    assert 'href="#catalog-query">검색어 입력으로 이동</a>' in html
-    assert "검색어는 80자 이하여야 합니다." in html
+    assert 'href="#catalog-query">품목명 입력으로 이동</a>' in html
+    assert "입력 내용을 확인하세요" in html
+    assert "품목명은 80자 이하로 입력하세요." in html
     assert private_query not in html
-    assert "조건에 맞는 항목 없음" not in html
+    assert "검색 결과가 없습니다" not in html
 
 
 @pytest.mark.parametrize(
     ("state", "expected_copy", "expected_role"),
     [
-        ("loading", "자료를 불러오는 중", 'role="status"'),
-        ("empty", "조건에 맞는 항목 없음", 'role="status"'),
-        ("unavailable", "공개 조사값 없음", 'role="status"'),
-        ("stale", "마지막 검토 자료 표시 중", 'role="status"'),
-        ("server_error", "자료를 표시하지 못함", 'role="alert"'),
+        ("loading", "조사 자료를 불러오고 있습니다", 'role="status"'),
+        ("empty", "검색 결과가 없습니다", 'role="status"'),
+        ("unavailable", "아직 공개된 조사 자료가 없습니다", 'role="status"'),
+        ("stale", "마지막 공개 자료를 표시합니다", 'role="status"'),
+        ("server_error", "조사 자료를 불러오지 못했습니다", 'role="alert"'),
     ],
 )
-def test_catalog_state_has_text_and_semantic_role(
+def test_catalog_state_has_plain_language_and_semantic_role(
     state: str,
     expected_copy: str,
     expected_role: str,
@@ -180,56 +248,101 @@ def test_catalog_state_has_text_and_semantic_role(
     context = catalog_context(catalog_state=state, retry_url="/")
     if state != "stale":
         context["results"] = []
-    html = render(
-        "grocery/catalog.html",
-        context,
-    )
+    html = render("grocery/catalog.html", context)
 
     assert expected_copy in html
     assert expected_role in html
 
 
-def test_detail_renders_exact_identity_comparisons_and_provenance() -> None:
+def test_blocking_catalog_states_do_not_render_search_or_false_results() -> None:
+    for state in ("loading", "unavailable", "server_error"):
+        html = render(
+            "grocery/catalog.html",
+            catalog_context(catalog_state=state, results=[], retry_url="/"),
+        )
+
+        assert 'role="search"' not in html
+        assert 'class="segment-list"' not in html
+        assert 'class="catalog-ledger"' not in html
+
+
+def test_detail_renders_identity_ruled_comparisons_and_provenance() -> None:
     html = render("grocery/detail.html", detail_context())
+    compact_html = " ".join(html.split())
 
-    assert "비교 대상의 정확한 조건" in html
+    assert 'class="detail-intro"' in html
+    assert 'class="current-price"' in html
+    assert "개별 판매처의 판매 금액이 아닌 KAMIS 소매 조사 평균입니다." in html
     assert "품종" in html and "봄" in html
     assert "등급" in html and "상품" in html
     assert "판매 단위" in html and "포기 × 1" in html
-    assert "조사범위" in html and "22개 도시 지역 전체 집계" in html
-    assert "2,000원 낮음" in html
-    assert "(-20.0%)" in html
-    assert "비교 정보 없음" in html
-    assert "source가 비교 기준일을 별도로 제공하지 않음" in html
+    assert "조사 범위" in html and "22개 도시 지역 전체 집계" in html
+    assert 'class="comparison-ledger"' in html
+    assert 'class="comparison-column-head"' in html
+    assert html.count('<li class="comparison-row') == 3
+    for class_name in ("period", "reference", "difference", "date"):
+        assert f"comparison-field--{class_name}" in html
+    assert "조사일 평균이 비교값보다 2,000원 낮음 (-20.0%)" in compact_html
+    assert "조사일 평균이 비교값과 같음 (0.0%)" in compact_html
+    assert "KAMIS가 이 기간의 비교값을 제공하지 않았습니다." in html
+    assert "<dt>비교 기준일</dt>" in html
+    assert "KAMIS에서 제공하지 않음" in html
+    assert "비교 기준일: KAMIS에서 제공하지 않음" not in html
     assert "데이터셋 15156063" in html
-    assert "공개 검토일" in html
+    assert "공개 검토 일시" in html
     assert '<time datetime="2026-08-29">' in html
     assert '<time datetime="2026-08-30T09:00:00+09:00">' in html
+    assert 'x="30"' in html and 'width="20"' in html and 'cx="30"' in html
+    assert 'style="' not in html
 
 
-def test_detail_direction_is_not_conveyed_by_symbol_or_color_alone() -> None:
+def test_detail_direction_is_not_conveyed_by_symbol_color_or_chart_alone() -> None:
     html = render("grocery/detail.html", detail_context())
 
     assert '<span class="direction__symbol" aria-hidden="true">' in html
     assert "낮음" in html
-    assert "비교 정보 없음" in html
+    assert "같음" in html
+    assert "비교값 없음" in html
+    assert 'class="comparison-meter' in html
+    assert 'aria-hidden="true"' in html
 
 
 @pytest.mark.parametrize(
     ("template_name", "heading"),
     [
-        ("400.html", "요청을 확인해 주세요"),
-        ("403.html", "이 페이지에 접근할 수 없습니다"),
+        ("400.html", "요청 내용을 확인하세요"),
+        ("403.html", "이 페이지를 볼 수 없습니다"),
         ("404.html", "페이지를 찾을 수 없습니다"),
-        ("500.html", "지금은 페이지를 표시할 수 없습니다"),
+        ("500.html", "페이지를 표시하지 못했습니다"),
     ],
 )
-def test_error_templates_render_recovery_link(template_name: str, heading: str) -> None:
+def test_error_templates_render_plain_recovery_without_technical_details(
+    template_name: str, heading: str
+) -> None:
     html = render(template_name, {"home_url": "/"})
 
     assert heading in html
     assert '<main id="main-content"' in html
     assert 'href="/"' in html
+    assert "<pre" not in html
+    assert "<code" not in html
+    assert "Traceback" not in html
+
+
+def test_public_templates_keep_ssr_security_and_local_visual_contract() -> None:
+    templates = [
+        *Path(settings.BASE_DIR, "grocery", "templates").rglob("*.html"),
+        *Path(settings.BASE_DIR, "templates").rglob("*.html"),
+    ]
+    template_source = "\n".join(path.read_text(encoding="utf-8") for path in templates)
+
+    assert "<script" not in template_source.lower()
+    assert "javascript:" not in template_source.lower()
+    assert "<picture" not in template_source.lower()
+    assert "style=" not in template_source.lower()
+    assert "http://" not in template_source.lower()
+    assert "source가" not in template_source
+    assert "source 응답" not in template_source
 
 
 def test_public_templates_do_not_contain_forbidden_claims() -> None:
@@ -237,22 +350,35 @@ def test_public_templates_do_not_contain_forbidden_claims() -> None:
         *Path(settings.BASE_DIR, "grocery", "templates").rglob("*.html"),
         *Path(settings.BASE_DIR, "templates").rglob("*.html"),
     ]
-    rendered_templates = "\n".join(path.read_text(encoding="utf-8") for path in templates)
+    template_source = "\n".join(path.read_text(encoding="utf-8") for path in templates)
 
     for phrase in FORBIDDEN_PUBLIC_PHRASES:
-        assert phrase not in rendered_templates
+        assert phrase not in template_source
 
 
-def test_styles_define_small_viewport_safety_focus_and_touch_targets() -> None:
+def test_styles_define_ledger_tokens_responsive_interaction_and_user_preferences() -> None:
     css = Path(settings.BASE_DIR, "grocery", "static", "grocery", "app.css").read_text(
         encoding="utf-8"
     )
 
+    assert "@font-face" in css
+    assert 'url("fonts/gowun-batang-bold.woff2")' in css
+    assert "--page-width: 72rem" in css
     assert "box-sizing: border-box" in css
     assert "min-width: 0" in css
     assert "overflow-wrap: anywhere" in css
     assert ":focus-visible" in css
+    assert "outline: 3px solid var(--color-focus)" in css
+    assert ".ledger-entry__link" in css
     assert "min-height: 2.75rem" in css
+    assert ".button:active" in css
+    assert "@media (hover: hover)" in css
     assert "@media (min-width: 40rem)" in css
     assert "@media (min-width: 64rem)" in css
+    assert "@media (prefers-reduced-motion: reduce)" in css
     assert "@media (forced-colors: active)" in css
+    assert "linear-gradient(" not in css
+    assert "radial-gradient(" not in css
+    assert "box-shadow" not in css
+    assert "--color-lower: #245b73" in css
+    assert "--color-higher: #245b73" in css
diff --git a/grocery/views.py b/grocery/views.py
index 9943303..15d0d55 100644
--- a/grocery/views.py
+++ b/grocery/views.py
@@ -4,6 +4,7 @@ from __future__ import annotations
 
 import logging
 import uuid
+from decimal import Decimal
 from typing import Final
 from urllib.parse import urlencode
 
@@ -17,26 +18,34 @@ from django.views.decorators.http import require_safe
 
 from grocery.forms import QUERY_MAX_LENGTH, SearchForm
 from grocery.observability import log_event
+from grocery.presentation import comparison_microbar
 from grocery.public_read import (
     catalog_item,
     detail_context,
     load_active_publication,
+    publication_context,
     publication_entries,
 )
 
 _LOGGER: Final = logging.getLogger("grocery.audit")
 _QA_STATES: Final = frozenset({"loading", "empty", "unavailable", "stale", "server_error"})
 _QA_DETAIL_STATES: Final = frozenset({"loading", "unavailable", "stale", "server_error"})
+_QA_ERROR_STATES: Final = {
+    "error_400": ("400.html", 400),
+    "error_403": ("403.html", 403),
+    "error_404": ("404.html", 404),
+    "error_500": ("500.html", 500),
+}
 _QUERY_ERROR_MESSAGES: Final = {
-    "max_length": f"검색어는 {QUERY_MAX_LENGTH}자 이하여야 합니다.",
-    "unsafe": "검색어에는 줄바꿈이나 제어 문자를 사용할 수 없습니다.",
+    "max_length": f"품목명은 {QUERY_MAX_LENGTH}자 이하로 입력하세요.",
+    "unsafe": "품목명은 한 줄로 입력하세요.",
 }
 _QA_STATE_MESSAGES: Final = {
-    "loading": "검토되어 공개된 조사값을 확인하고 있습니다.",
-    "empty": "검색어를 줄이거나 다른 부류를 선택해 보세요.",
-    "unavailable": "현재 공개할 수 있는 검토 완료 자료가 없습니다.",
-    "stale": "마지막 검토 자료를 표시하며 새 수집 상태를 확인하고 있습니다.",
-    "server_error": "잠시 후 다시 시도해 주세요.",
+    "loading": "공개된 자료를 확인하는 동안 잠시 기다려 주세요.",
+    "empty": "품목명을 바꾸거나 다른 부류를 선택하세요.",
+    "unavailable": "검토를 마친 자료가 공개되면 이곳에 표시됩니다.",
+    "stale": "최근 자료 확인이 필요합니다. 마지막으로 검토를 마친 조사값을 표시합니다.",
+    "server_error": "잠시 후 다시 불러오세요.",
 }
 
 
@@ -70,7 +79,7 @@ def catalog(request: HttpRequest) -> HttpResponse:
                 {
                     "catalog_state": "unavailable",
                     "results": [],
-                    "status_message": "현재 공개할 수 있는 검토 완료 자료가 없습니다.",
+                    "status_message": "검토를 마친 자료가 공개되면 이곳에 표시됩니다.",
                 }
             )
             return render(request, "grocery/catalog.html", context)
@@ -90,22 +99,23 @@ def catalog(request: HttpRequest) -> HttpResponse:
                 "status_message": (
                     active.stale_message
                     if active.stale_message
-                    else "검색 조건에 맞는 공개 항목이 없습니다."
+                    else "품목명을 바꾸거나 다른 부류를 선택하세요."
                 ),
                 "results": results,
                 "result_count_label": f"공개 항목 {len(results)}개",
+                "publication": publication_context(active),
             }
         )
         response = render(request, "grocery/catalog.html", context)
         return _publication_response(response, active.revision.typed_fact_set_sha256)
-    except DatabaseError, ValidationError:
+    except (DatabaseError, ValidationError):  # fmt: skip
         log_event(_LOGGER, "ERROR", "public.catalog.unavailable")
         context = _catalog_base_context(category=category)
         context.update(
             {
                 "catalog_state": "server_error",
                 "results": [],
-                "status_message": "잠시 후 다시 시도해 주세요.",
+                "status_message": "잠시 후 다시 불러오세요.",
                 "retry_url": reverse("grocery:catalog"),
             }
         )
@@ -133,19 +143,20 @@ def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
             "catalog_url": reverse("grocery:catalog"),
             "detail_state": active.freshness_state if active.stale_message else "ready",
             "status_message": active.stale_message,
+            "publication": publication_context(active),
             **detail_context(entry, active),
         }
         response = render(request, "grocery/detail.html", context)
         return _publication_response(response, active.revision.typed_fact_set_sha256)
     except Http404:
         raise
-    except DatabaseError, ObjectDoesNotExist, ValidationError:
+    except (DatabaseError, ObjectDoesNotExist, ValidationError):  # fmt: skip
         log_event(_LOGGER, "ERROR", "public.detail.unavailable")
         context = {
             "home_url": reverse("grocery:catalog"),
             "catalog_url": reverse("grocery:catalog"),
             "detail_state": "server_error",
-            "status_message": "잠시 후 다시 시도해 주세요.",
+            "status_message": "잠시 후 다시 불러오세요.",
             "retry_url": request.path,
         }
         return render(request, "grocery/detail.html", context, status=503)
@@ -153,7 +164,18 @@ def detail(request: HttpRequest, series_id: uuid.UUID) -> HttpResponse:
 
 @require_safe
 def qa_catalog_state(request: HttpRequest, state: str) -> HttpResponse:
-    if not settings.QA_STATE_PREVIEWS_ENABLED or state not in _QA_STATES:
+    if not settings.QA_STATE_PREVIEWS_ENABLED:
+        raise Http404
+    error_preview = _QA_ERROR_STATES.get(state)
+    if error_preview is not None:
+        template_name, status = error_preview
+        return render(
+            request,
+            template_name,
+            {"home_url": reverse("grocery:catalog"), "qa_preview": True},
+            status=status,
+        )
+    if state not in _QA_STATES:
         raise Http404
     context = _catalog_base_context(category="vegetable")
     context.update(
@@ -164,6 +186,7 @@ def qa_catalog_state(request: HttpRequest, state: str) -> HttpResponse:
             "retry_url": request.path,
             "results": _qa_results() if state == "stale" else [],
             "result_count_label": "공개 항목 1개" if state == "stale" else "공개 항목 0개",
+            "publication": _qa_publication_context() if state == "stale" else {},
         }
     )
     return render(
@@ -230,10 +253,10 @@ def _query_error(form: SearchForm) -> str:
     errors = form.errors.as_data().get("q", [])
     if not errors:
         return ""
-    return _QUERY_ERROR_MESSAGES.get(errors[0].code or "", "검색어를 확인해 주세요.")
+    return _QUERY_ERROR_MESSAGES.get(errors[0].code or "", "품목명을 확인하세요.")
 
 
-def _qa_results() -> list[dict[str, str]]:
+def _qa_results() -> list[dict[str, object]]:
     return [
         {
             "url": reverse("grocery:qa_detail_state", kwargs={"state": "stale"}),
@@ -246,7 +269,20 @@ def _qa_results() -> list[dict[str, str]]:
             "source_date_iso": "2026-08-29",
             "source_date_label": "2026년 8월 29일",
             "freshness_state": "stale",
-            "freshness_label": "마지막 검토 자료 · 새 확인 필요",
+            "freshness_label": "마지막 공개 자료 · 최근 확인 필요",
+            "week_comparison": {
+                "period_label": "1주 전 제공값",
+                "available": True,
+                "reference_price_display": "125,456원",
+                "difference_display": "2,000원",
+                "percentage_display": "-1.6%",
+                "direction_code": "LOWER",
+                "direction_label": "낮음",
+                "reference_date_display": "",
+                "reference_date_unavailable": True,
+                "unavailable_reason": "",
+                "microbar": comparison_microbar(Decimal("-1.6"), "LOWER"),
+            },
         }
     ]
 
@@ -266,28 +302,41 @@ def _qa_detail_ready_context() -> dict[str, object]:
             {
                 "period_label": "1주 전 제공값",
                 "available": True,
-                "reference_value_label": "125,456원",
-                "difference_label": "2,000원",
-                "percentage_label": "-1.6%",
+                "reference_price_display": "125,456원",
+                "difference_display": "2,000원",
+                "percentage_display": "-1.6%",
                 "direction_code": "LOWER",
                 "direction_label": "낮음",
-                "reference_date_available": False,
+                "reference_date_display": "2026년 8월 22일",
+                "reference_date_unavailable": False,
+                "unavailable_reason": "",
+                "microbar": comparison_microbar(Decimal("-1.6"), "LOWER"),
             },
             {
                 "period_label": "1개월 전 제공값",
                 "available": False,
-                "unavailable_reason_label": "source 응답에 비교 제공값이 없습니다.",
-                "reference_date_available": False,
+                "reference_price_display": "",
+                "difference_display": "",
+                "percentage_display": "",
+                "direction_code": "UNAVAILABLE",
+                "direction_label": "비교 정보 없음",
+                "reference_date_display": "",
+                "reference_date_unavailable": True,
+                "unavailable_reason": "KAMIS가 이 기간의 비교값을 제공하지 않았습니다.",
+                "microbar": None,
             },
             {
                 "period_label": "1년 전 제공값",
                 "available": True,
-                "reference_value_label": "123,456원",
-                "difference_label": "0원",
-                "percentage_label": "0.0%",
+                "reference_price_display": "123,456원",
+                "difference_display": "0원",
+                "percentage_display": "0.0%",
                 "direction_code": "EQUAL",
                 "direction_label": "같음",
-                "reference_date_available": False,
+                "reference_date_display": "",
+                "reference_date_unavailable": True,
+                "unavailable_reason": "",
+                "microbar": comparison_microbar(Decimal("0.0"), "EQUAL"),
             },
         ],
         "provenance": {
@@ -300,10 +349,20 @@ def _qa_detail_ready_context() -> dict[str, object]:
             "source_date_label": "2026년 8월 29일",
             "coverage_label": "KAMIS 소매 조사 22개 도시 지역 전체 집계",
             "checked_at_iso": "2026-08-30T12:00:00+09:00",
-            "checked_at_label": "2026년 8월 30일 12:00",
+            "checked_at_display": "2026년 8월 30일 12:00",
             "reviewed_at_iso": "2026-08-30T12:30:00+09:00",
             "reviewed_at_label": "2026년 8월 30일 12:30",
             "freshness_state": "stale",
-            "freshness_label": "마지막 검토 자료 · 새 확인 필요",
+            "freshness_label": "마지막 공개 자료 · 최근 확인 필요",
         },
+        "publication": _qa_publication_context(),
+    }
+
+
+def _qa_publication_context() -> dict[str, str]:
+    return {
+        "checked_at_iso": "2026-08-30T12:00:00+09:00",
+        "checked_at_display": "2026년 8월 30일 12:00",
+        "freshness_state": "stale",
+        "freshness_label": "마지막 공개 자료 · 최근 확인 필요",
     }
diff --git a/templates/400.html b/templates/400.html
index 34ff8e2..e44aba6 100644
--- a/templates/400.html
+++ b/templates/400.html
@@ -1,12 +1,11 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}요청을 확인해 주세요 | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}요청 내용을 확인하세요 | 초록장부{% endblock %}
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
-    <p class="eyebrow">요청 오류 · 400</p>
-    <h1 id="error-heading">요청을 확인해 주세요</h1>
-    <p>입력한 주소나 검색 조건을 처리할 수 없습니다. 내용을 고친 뒤 다시 시도해 주세요.</p>
+    <h1 id="error-heading">요청 내용을 확인하세요</h1>
+    <p>입력한 주소나 검색 조건을 처리할 수 없습니다. 내용을 확인한 뒤 다시 시도하세요.</p>
     <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
   </section>
 {% endblock %}
diff --git a/templates/403.html b/templates/403.html
index b077f22..9edffc4 100644
--- a/templates/403.html
+++ b/templates/403.html
@@ -1,11 +1,10 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}접근할 수 없음 | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}이 페이지를 볼 수 없습니다 | 초록장부{% endblock %}
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
-    <p class="eyebrow">접근 제한 · 403</p>
-    <h1 id="error-heading">이 페이지에 접근할 수 없습니다</h1>
+    <h1 id="error-heading">이 페이지를 볼 수 없습니다</h1>
     <p>공개된 조사값은 목록에서 확인할 수 있습니다.</p>
     <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
   </section>
diff --git a/templates/404.html b/templates/404.html
index 652c719..58299b1 100644
--- a/templates/404.html
+++ b/templates/404.html
@@ -1,10 +1,9 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}페이지를 찾을 수 없음 | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}페이지를 찾을 수 없습니다 | 초록장부{% endblock %}
 
 {% block content %}
   <section class="error-page" aria-labelledby="error-heading">
-    <p class="eyebrow">찾을 수 없음 · 404</p>
     <h1 id="error-heading">페이지를 찾을 수 없습니다</h1>
     <p>주소가 바뀌었거나 현재 공개 목록에 없는 항목입니다.</p>
     <a class="button button--primary" href="{{ home_url|default:'/' }}">공개 목록으로 이동</a>
diff --git a/templates/500.html b/templates/500.html
index b6e1325..5b6d34a 100644
--- a/templates/500.html
+++ b/templates/500.html
@@ -1,12 +1,11 @@
 {% extends "grocery/base.html" %}
 
-{% block title %}일시적인 오류 | 농산물 조사값 살펴보기{% endblock %}
+{% block title %}페이지를 표시하지 못했습니다 | 초록장부{% endblock %}
 
 {% block content %}
   <section class="error-page" role="alert" aria-labelledby="error-heading">
-    <p class="eyebrow">서버 오류 · 500</p>
-    <h1 id="error-heading">지금은 페이지를 표시할 수 없습니다</h1>
-    <p>요청을 완료하지 못했습니다. 잠시 후 다시 시도해 주세요.</p>
+    <h1 id="error-heading">페이지를 표시하지 못했습니다</h1>
+    <p>요청을 완료하지 못했습니다. 잠시 후 다시 시도하세요.</p>
     <a class="button button--primary" href="{{ home_url|default:'/' }}">목록으로 돌아가기</a>
   </section>
 {% endblock %}


