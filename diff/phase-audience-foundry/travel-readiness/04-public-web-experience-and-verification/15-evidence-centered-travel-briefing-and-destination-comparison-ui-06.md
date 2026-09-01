## `feat(frontend): present ranked travel opportunities`

diff --git a/e2e/browser_scenario_server.py b/e2e/browser_scenario_server.py
index e12041f..dac416b 100644
--- a/e2e/browser_scenario_server.py
+++ b/e2e/browser_scenario_server.py
@@ -49,8 +49,8 @@ SITE_ASSETS: Final = {
     "/static/public_web/site.css": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.css",
         "text/css",
-        8_493,
-        "8ceab60867561812885e614c3aeedc563118a6ffd90bbc78c821a2e5d72f0e76",
+        15_265,
+        "e041050e5ccba01e218582072ae29c687e5cc2ad5432861e77870ffc45cec34a",
     ),
     "/static/public_web/site.js": (
         REPOSITORY_ROOT / "public_web" / "static" / "public_web" / "site.js",
diff --git a/public_web/static/public_web/site.css b/public_web/static/public_web/site.css
index 3fe49d8..18753fb 100644
--- a/public_web/static/public_web/site.css
+++ b/public_web/static/public_web/site.css
@@ -389,6 +389,346 @@ h3 {
   font-weight: 750;
 }
 
+.results-section {
+  margin-top: clamp(var(--space-16), 10vw, var(--space-24));
+  scroll-margin-top: var(--space-8);
+}
+
+.results-header {
+  max-width: var(--reading-width);
+  margin-left: auto;
+  padding-bottom: var(--space-8);
+  border-bottom: 0.125rem solid var(--header);
+}
+
+.search-window {
+  display: grid;
+  min-width: 0;
+  margin: var(--space-8) 0 0;
+  border-block: 0.0625rem solid var(--line-strong);
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+}
+
+.search-window div {
+  min-width: 0;
+  padding: var(--space-4);
+}
+
+.search-window div + div {
+  border-left: 0.0625rem solid var(--line);
+}
+
+.search-window dt {
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
+  font-weight: 800;
+}
+
+.search-window dd {
+  margin: var(--space-1) 0 0;
+  font-size: 1.0625rem;
+  font-weight: 850;
+}
+
+.flight-status {
+  margin-top: var(--space-6);
+  padding-left: var(--space-4);
+  border-left: 0.35rem solid var(--current);
+}
+
+.results-section[data-state="empty"] .flight-status {
+  border-left-color: var(--muted-ink);
+}
+
+.results-section[data-state="unavailable"] .flight-status,
+.results-section[data-state="stale"] .flight-status {
+  border-left-color: var(--stale);
+}
+
+.results-section[data-state="server-error"] .flight-status {
+  border-left-color: var(--error);
+}
+
+.flight-status p {
+  margin: 0;
+}
+
+.flight-status p + p {
+  margin-top: var(--space-1);
+  color: var(--muted-ink);
+}
+
+.flight-source {
+  margin: var(--space-4) 0 0;
+  color: var(--muted-ink);
+  font-size: 0.875rem;
+}
+
+.flight-source a {
+  display: inline-flex;
+  min-height: 44px;
+  align-items: center;
+  color: var(--link);
+  font-weight: 800;
+}
+
+.opportunity-list {
+  max-width: var(--reading-width);
+  margin: 0 0 0 auto;
+  padding: 0;
+  list-style: none;
+}
+
+.opportunity-list > li {
+  min-width: 0;
+}
+
+.opportunity {
+  min-width: 0;
+  padding-block: clamp(var(--space-12), 8vw, var(--space-16));
+  border-bottom: 0.125rem solid var(--header);
+}
+
+.opportunity-rank {
+  margin: 0 0 var(--space-2);
+  color: var(--current);
+  font-size: 0.8125rem;
+  font-weight: 850;
+  letter-spacing: 0.08em;
+}
+
+.destination-heading {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-6);
+  align-items: flex-end;
+  justify-content: space-between;
+}
+
+.destination-heading > div {
+  min-width: 0;
+}
+
+.destination-heading h3 {
+  font-size: clamp(2rem, 7vw, 3.25rem);
+  letter-spacing: -0.055em;
+}
+
+.destination-heading > div p {
+  margin: var(--space-1) 0 0;
+  color: var(--muted-ink);
+}
+
+.stay-time {
+  flex: 0 0 auto;
+  margin: 0;
+  text-align: right;
+}
+
+.stay-time span {
+  display: block;
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
+  font-weight: 750;
+}
+
+.stay-time strong {
+  display: block;
+  font-size: clamp(1.5rem, 5vw, 2.2rem);
+  letter-spacing: -0.04em;
+  line-height: 1.15;
+}
+
+.main-itinerary {
+  margin-top: var(--space-8);
+}
+
+.main-itinerary > h4,
+.calculation-note h4,
+.travel-briefs > h4 {
+  margin: 0 0 var(--space-3);
+  font-size: 0.875rem;
+  letter-spacing: 0.03em;
+}
+
+.itinerary-legs {
+  display: grid;
+  min-width: 0;
+  border-block: 0.0625rem solid var(--line-strong);
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+}
+
+.flight-leg {
+  min-width: 0;
+  padding: var(--space-4) var(--space-6) var(--space-6) 0;
+}
+
+.flight-leg + .flight-leg {
+  padding-inline: var(--space-6) 0;
+  border-left: 0.0625rem solid var(--line);
+}
+
+.flight-leg-heading {
+  display: flex;
+  min-width: 0;
+  gap: var(--space-3);
+  align-items: baseline;
+  justify-content: space-between;
+}
+
+.flight-leg-heading p {
+  margin: 0;
+  color: var(--muted-ink);
+  font-size: 0.875rem;
+}
+
+.flight-leg-heading .leg-direction {
+  color: var(--ink);
+  font-weight: 850;
+}
+
+.event-list {
+  position: relative;
+  margin: var(--space-6) 0 0;
+  padding-left: var(--space-6);
+}
+
+.event-list::before {
+  position: absolute;
+  top: 0.4rem;
+  bottom: 0.4rem;
+  left: 0.25rem;
+  width: 0.125rem;
+  content: "";
+  background: var(--line-strong);
+}
+
+.event {
+  position: relative;
+}
+
+.event + .event {
+  margin-top: var(--space-6);
+}
+
+.event::before {
+  position: absolute;
+  top: 0.45rem;
+  left: calc(-1.5rem - 0.2rem);
+  width: 0.65rem;
+  height: 0.65rem;
+  content: "";
+  background: var(--surface);
+  border: 0.125rem solid var(--current);
+  border-radius: 50%;
+}
+
+.event--estimated::before {
+  border-color: var(--muted-ink);
+  border-style: dashed;
+}
+
+.event dt {
+  color: var(--muted-ink);
+  font-size: 0.8125rem;
+  font-weight: 750;
+}
+
+.event dt span {
+  display: inline-block;
+  margin-left: var(--space-1);
+  padding-left: var(--space-2);
+  border-left: 0.0625rem solid var(--line);
+}
+
+.event--official dt span {
+  color: var(--current);
+}
+
+.event dd {
+  margin: 0;
+  font-size: 1.125rem;
+  font-weight: 850;
+}
+
+.alternatives {
+  margin-top: var(--space-6);
+  border-block: 0.0625rem solid var(--line);
+}
+
+.alternatives summary {
+  display: flex;
+  min-height: 48px;
+  align-items: center;
+  padding-block: var(--space-3);
+  color: var(--link);
+  font-weight: 850;
+  cursor: pointer;
+}
+
+.alternatives > ol {
+  margin: 0;
+  padding: 0 0 var(--space-4);
+  list-style: none;
+}
+
+.alternatives > ol > li {
+  padding: var(--space-4);
+  background: var(--surface);
+  border-left: 0.25rem solid var(--line-strong);
+}
+
+.alternatives > ol > li + li {
+  margin-top: var(--space-3);
+}
+
+.alternative-stay {
+  margin: 0 0 var(--space-2);
+}
+
+.alternative-schedule {
+  margin: 0;
+}
+
+.alternative-schedule div {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2);
+  grid-template-columns: 4rem minmax(0, 1fr);
+}
+
+.alternative-schedule div + div {
+  margin-top: var(--space-1);
+}
+
+.alternative-schedule dt {
+  color: var(--muted-ink);
+  font-size: 0.875rem;
+  font-weight: 800;
+}
+
+.alternative-schedule dd {
+  min-width: 0;
+  margin: 0;
+}
+
+.calculation-note {
+  margin-top: var(--space-8);
+  padding: var(--space-4) var(--space-6);
+  color: var(--muted-ink);
+  background: var(--surface-muted);
+  border-left: 0.25rem solid var(--stale);
+}
+
+.calculation-note p {
+  margin: 0;
+}
+
+.calculation-note p + p {
+  margin-top: var(--space-2);
+  font-size: 0.8125rem;
+}
+
 .site-footer {
   padding-block: var(--space-6);
   color: white;
@@ -424,6 +764,11 @@ h3 {
   .search-intro h1 {
     max-width: 15ch;
   }
+
+  .results-header,
+  .opportunity-list {
+    margin-left: 0;
+  }
 }
 
 @media (max-width: 30rem) {
@@ -450,12 +795,44 @@ h3 {
   input[type="datetime-local"] {
     font-size: 1rem;
   }
+
+  .search-window,
+  .itinerary-legs {
+    grid-template-columns: minmax(0, 1fr);
+  }
+
+  .search-window div + div,
+  .flight-leg + .flight-leg {
+    border-top: 0.0625rem solid var(--line);
+    border-left: 0;
+  }
+
+  .flight-leg,
+  .flight-leg + .flight-leg {
+    padding-inline: 0;
+  }
+
+  .destination-heading {
+    align-items: flex-start;
+    flex-direction: column;
+    gap: var(--space-4);
+  }
+
+  .stay-time {
+    text-align: left;
+  }
+
+  .alternative-schedule div {
+    grid-template-columns: minmax(0, 1fr);
+  }
 }
 
 @media (pointer: coarse) {
   .site-brand,
   .primary-button,
-  .error-summary a {
+  .error-summary a,
+  .flight-source a,
+  .alternatives summary {
     min-height: 48px;
   }
 }
@@ -481,6 +858,15 @@ h3 {
     border-color: CanvasText;
   }
 
+  .flight-status,
+  .opportunity,
+  .itinerary-legs,
+  .flight-leg,
+  .calculation-note,
+  .alternatives {
+    border-color: CanvasText;
+  }
+
   :where(a, button, input, select, [tabindex="0"]):focus-visible {
     outline: 0.1875rem solid Highlight;
   }
diff --git a/public_web/templates/public_web/index.html b/public_web/templates/public_web/index.html
index 2abf487..02c3cea 100644
--- a/public_web/templates/public_web/index.html
+++ b/public_web/templates/public_web/index.html
@@ -90,6 +90,10 @@
       </form>
     </div>
   </section>
+
+  {% if has_search_result %}
+    {% include "public_web/partials/search_results.html" %}
+  {% endif %}
 {% endblock %}
 
 {% block footer %}
diff --git a/public_web/templates/public_web/partials/opportunity.html b/public_web/templates/public_web/partials/opportunity.html
new file mode 100644
index 0000000..0946460
--- /dev/null
+++ b/public_web/templates/public_web/partials/opportunity.html
@@ -0,0 +1,86 @@
+<article id="destination-{{ opportunity.rank }}" class="opportunity"
+         aria-labelledby="destination-{{ opportunity.rank }}-heading">
+  <header class="opportunity-header">
+    <p class="opportunity-rank">추천 {{ opportunity.rank }}</p>
+    <div class="destination-heading">
+      <div>
+        <h3 id="destination-{{ opportunity.rank }}-heading">{{ opportunity.destination.city_name }}</h3>
+        <p>{{ opportunity.destination.country_name }} · {{ opportunity.destination.airport_name }} ({{ opportunity.destination.airport_code }})</p>
+      </div>
+      <p class="stay-time">
+        <span>예상 현지 체류시간</span>
+        <strong>{{ opportunity.estimated_local_stay_label }}</strong>
+      </p>
+    </div>
+  </header>
+
+  <section class="main-itinerary" aria-labelledby="destination-{{ opportunity.rank }}-itinerary">
+    <h4 id="destination-{{ opportunity.rank }}-itinerary">추천 일정</h4>
+    <div class="itinerary-legs">
+      <section class="flight-leg" aria-label="출국 운항 예정편">
+        <div class="flight-leg-heading">
+          <p class="leg-direction">가는 편</p>
+          <p>{{ opportunity.outbound_schedule.carrier_name }} {{ opportunity.outbound_schedule.flight_number }}</p>
+        </div>
+        <dl class="event-list">
+          <div class="event event--official">
+            <dt>인천 출발 <span>공식 운항 일정</span></dt>
+            <dd>{{ opportunity.outbound_schedule.icn_event_label }}</dd>
+          </div>
+          <div class="event event--estimated">
+            <dt>현지 도착 <span>제품 예상</span></dt>
+            <dd>{{ opportunity.outbound_schedule.estimated_destination_event_label }}</dd>
+          </div>
+        </dl>
+      </section>
+
+      <section class="flight-leg" aria-label="귀국 운항 예정편">
+        <div class="flight-leg-heading">
+          <p class="leg-direction">오는 편</p>
+          <p>{{ opportunity.inbound_schedule.carrier_name }} {{ opportunity.inbound_schedule.flight_number }}</p>
+        </div>
+        <dl class="event-list">
+          <div class="event event--estimated">
+            <dt>현지 출발 <span>제품 예상</span></dt>
+            <dd>{{ opportunity.inbound_schedule.estimated_destination_event_label }}</dd>
+          </div>
+          <div class="event event--official">
+            <dt>인천 도착 <span>공식 운항 일정</span></dt>
+            <dd>{{ opportunity.inbound_schedule.icn_event_label }}</dd>
+          </div>
+        </dl>
+      </section>
+    </div>
+  </section>
+
+  {% if opportunity.alternatives %}
+    <details class="alternatives">
+      <summary>다른 시간 조합 {{ opportunity.alternatives|length }}개 보기</summary>
+      <ol>
+        {% for alternative in opportunity.alternatives %}
+          <li>
+            <p class="alternative-stay">예상 현지 체류시간 <strong>{{ alternative.estimated_local_stay_label }}</strong></p>
+            <dl class="alternative-schedule">
+              <div>
+                <dt>가는 편</dt>
+                <dd>{{ alternative.outbound_schedule.carrier_name }} {{ alternative.outbound_schedule.flight_number }} · 인천 출발 {{ alternative.outbound_schedule.icn_event_label }}</dd>
+              </div>
+              <div>
+                <dt>오는 편</dt>
+                <dd>{{ alternative.inbound_schedule.carrier_name }} {{ alternative.inbound_schedule.flight_number }} · 인천 도착 {{ alternative.inbound_schedule.icn_event_label }}</dd>
+              </div>
+            </dl>
+          </li>
+        {% endfor %}
+      </ol>
+    </details>
+  {% endif %}
+
+  <aside class="calculation-note" aria-label="예상 체류시간 계산 기준">
+    <h4>계산 기준</h4>
+    <p>{{ opportunity.calculation_basis.notice }}</p>
+    <p>공식 운항 일정 {{ opportunity.calculation_basis.schedule_source_date|default:"날짜 없음" }} · 비행시간 참고자료 {{ opportunity.calculation_basis.duration_reference_date|default:"날짜 없음" }}</p>
+  </aside>
+
+  <div class="travel-briefs" data-travel-briefs></div>
+</article>
diff --git a/public_web/templates/public_web/partials/search_results.html b/public_web/templates/public_web/partials/search_results.html
new file mode 100644
index 0000000..2a63780
--- /dev/null
+++ b/public_web/templates/public_web/partials/search_results.html
@@ -0,0 +1,43 @@
+<section id="recommendations" class="results-section" data-state="{{ flight_state }}"
+         aria-labelledby="results-heading">
+  <header class="results-header">
+    <p class="section-kicker">검색 결과</p>
+    {% if flight_state == "ready" and opportunities %}
+      <h2 id="results-heading">갈 수 있는 도시를 찾았습니다</h2>
+    {% else %}
+      <h2 id="results-heading">찾은 결과를 확인해 주세요</h2>
+    {% endif %}
+    <dl class="search-window" aria-label="입력한 여행 가능 시간">
+      <div>
+        <dt>출발 가능</dt>
+        <dd>{{ search_window.departure_at_label }}</dd>
+      </div>
+      <div>
+        <dt>인천 도착 마감</dt>
+        <dd>{{ search_window.return_by_label }}</dd>
+      </div>
+    </dl>
+    <div class="flight-status" role="status" aria-live="polite">
+      <p><strong>운항 자료 상태: {{ flight_status_label }}</strong></p>
+      <p>{{ flight_message }}</p>
+    </div>
+    {% if flight_source_locator %}
+      <p class="flight-source">
+        운항 일정 마지막 확인 {{ flight_checked_at|default:"확인시각 없음" }} ·
+        자료 {{ flight_publication_revision|default:"리비전 없음" }}<br>
+        <a href="{{ flight_source_locator }}" rel="noopener noreferrer">운항 일정 공식 출처 열기</a>
+        {% if flight_source_attribution %}<span> — {{ flight_source_attribution }}</span>{% endif %}
+      </p>
+    {% endif %}
+  </header>
+
+  {% if opportunities %}
+    <ol class="opportunity-list" aria-label="추천 도시">
+      {% for opportunity in opportunities %}
+        <li>
+          {% include "public_web/partials/opportunity.html" %}
+        </li>
+      {% endfor %}
+    </ol>
+  {% endif %}
+</section>


