## `feat(frontend): apply market editorial design system`

diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
index f5879d1..72486b5 100644
--- a/grocery/static/grocery/app.css
+++ b/grocery/static/grocery/app.css
@@ -8,28 +8,31 @@
 
 :root {
   color-scheme: light;
-  --color-canvas: #f4f0e6;
-  --color-surface: #fffdf7;
-  --color-surface-muted: #eee8da;
-  --color-text: #1d2820;
-  --color-muted: #536057;
-  --color-border: #bdb6a5;
-  --color-border-strong: #736d5f;
-  --color-brand: #286442;
-  --color-brand-strong: #17472f;
-  --color-brand-soft: #e3eee5;
-  --color-info: #245b73;
-  --color-info-soft: #e8f2f5;
-  --color-warning: #704b00;
-  --color-warning-soft: #fff1c9;
-  --color-error: #8b2830;
-  --color-error-soft: #fbe9e6;
-  --color-neutral-soft: #eceee9;
-  --color-data: #245b73;
-  --color-lower: #245b73;
-  --color-higher: #245b73;
-  --color-focus: #005fcc;
-  --color-on-brand: #ffffff;
+  --color-canvas: #f3e9d7;
+  --color-surface: #fffdf6;
+  --color-surface-muted: #e8ddcb;
+  --color-text: #18221c;
+  --color-muted: #566058;
+  --color-border: #c6bba8;
+  --color-border-strong: #746b5d;
+  --color-brand: #a93426;
+  --color-brand-strong: #123f2d;
+  --color-brand-soft: #dce8dc;
+  --color-info: #1c5d75;
+  --color-info-soft: #dcecef;
+  --color-warning: #704700;
+  --color-warning-soft: #fff0c2;
+  --color-error: #8e2630;
+  --color-error-soft: #f8e4df;
+  --color-neutral-soft: #e8e4da;
+  --color-data: #1c5d75;
+  --color-lower: #1c5d75;
+  --color-higher: #1c5d75;
+  --color-focus: #075d92;
+  --color-on-brand: #fffdf6;
+  --color-harvest: #a93426;
+  --color-forest: #123f2d;
+  --color-ivory: #fffdf6;
   --space-1: 0.25rem;
   --space-2: 0.5rem;
   --space-3: 0.75rem;
@@ -40,7 +43,7 @@
   --space-10: 2.5rem;
   --space-12: 3rem;
   --space-16: 4rem;
-  --radius-small: 0.25rem;
+  --radius-small: 0.125rem;
   --page-gutter: clamp(1rem, 4vw, 2rem);
   --measure: 44rem;
   --page-width: 72rem;
@@ -64,7 +67,7 @@ html {
     "Noto Sans KR",
     sans-serif;
   font-size: 100%;
-  line-height: 1.6;
+  line-height: 1.55;
   overflow-wrap: anywhere;
   word-break: keep-all;
 }
@@ -73,6 +76,7 @@ body {
   min-width: 0;
   min-height: 100vh;
   margin: 0;
+  background: var(--color-canvas);
 }
 
 img,
@@ -84,6 +88,7 @@ svg {
 time,
 data,
 .comparison-field--reference strong {
+  font-variant-numeric: tabular-nums;
   overflow-wrap: normal;
   white-space: nowrap;
 }
@@ -115,36 +120,42 @@ dd {
 
 h1,
 h2,
-h3,
-h4,
 .brand__name {
   font-family: "Gowun Batang", serif;
   font-weight: 700;
-  line-height: 1.25;
+  line-height: 1.18;
+  text-wrap: balance;
+}
+
+h3,
+h4 {
+  font-family: inherit;
+  font-weight: 800;
+  line-height: 1.3;
   text-wrap: balance;
 }
 
 h1 {
-  max-width: 24ch;
-  margin-bottom: var(--space-3);
-  font-size: clamp(2rem, 7vw, 3.5rem);
-  letter-spacing: -0.045em;
+  max-width: 20ch;
+  margin-bottom: var(--space-4);
+  font-size: clamp(2.25rem, 8vw, 4.75rem);
+  letter-spacing: -0.055em;
 }
 
 h2 {
-  margin-bottom: var(--space-3);
-  font-size: clamp(1.35rem, 4vw, 1.75rem);
-  letter-spacing: -0.025em;
+  margin-bottom: var(--space-4);
+  font-size: clamp(1.55rem, 4.2vw, 2.35rem);
+  letter-spacing: -0.04em;
 }
 
 h3 {
   margin-bottom: var(--space-2);
-  font-size: 1.15rem;
+  font-size: 1.08rem;
 }
 
 h4 {
   margin-bottom: var(--space-1);
-  font-size: 1.08rem;
+  font-size: 1rem;
 }
 
 .visually-hidden {
@@ -168,7 +179,7 @@ h4 {
 
 .page-main {
   min-height: 65vh;
-  padding-block: clamp(var(--space-6), 7vw, var(--space-16));
+  padding-block: clamp(var(--space-6), 5.5vw, var(--space-12));
 }
 
 .skip-link {
@@ -193,14 +204,15 @@ h4 {
 
 .masthead,
 .site-header {
-  border-bottom: 1px solid var(--color-border);
-  background: var(--color-surface);
+  border-bottom: 4px solid var(--color-harvest);
+  background: var(--color-forest);
+  color: var(--color-ivory);
 }
 
 .masthead__inner,
 .site-header__inner {
   display: flex;
-  min-height: 4.75rem;
+  min-height: 5rem;
   align-items: center;
 }
 
@@ -215,7 +227,7 @@ h4 {
   min-height: 2.75rem;
   align-items: center;
   gap: var(--space-3);
-  color: var(--color-text);
+  color: var(--color-ivory);
   text-decoration: none;
 }
 
@@ -223,6 +235,7 @@ h4 {
   width: 2.75rem;
   height: 2.75rem;
   flex: 0 0 auto;
+  filter: brightness(0) invert(1);
 }
 
 .brand-copy {
@@ -233,13 +246,15 @@ h4 {
 }
 
 .brand__name {
-  font-size: 1.2rem;
-  letter-spacing: -0.025em;
+  font-size: 1.34rem;
+  letter-spacing: -0.045em;
 }
 
 .brand__description {
-  color: var(--color-muted);
-  font-size: 0.82rem;
+  color: #dce8dc;
+  font-size: 0.76rem;
+  font-weight: 700;
+  letter-spacing: 0.025em;
 }
 
 .site-actions {
@@ -253,7 +268,8 @@ h4 {
   align-items: center;
   gap: var(--space-1);
   padding-inline: var(--space-2);
-  color: var(--color-brand-strong);
+  border-bottom: 1px solid #789b8b;
+  color: var(--color-ivory);
   font-size: 0.88rem;
   font-weight: 800;
 }
@@ -263,20 +279,21 @@ h4 {
   min-width: 1.5rem;
   min-height: 1.5rem;
   place-items: center;
-  border: 1px solid var(--color-brand-strong);
-  border-radius: 50%;
+  border: 1px solid var(--color-ivory);
+  border-radius: var(--radius-small);
+  background: var(--color-harvest);
   line-height: 1;
 }
 
 .site-footer {
-  border-top: 1px solid var(--color-border);
-  background: var(--color-surface);
-  color: var(--color-muted);
+  border-top: 4px solid var(--color-harvest);
+  background: var(--color-forest);
+  color: #dce8dc;
   font-size: 0.9rem;
 }
 
 .site-footer__inner {
-  padding-block: var(--space-6);
+  padding-block: var(--space-8);
 }
 
 .site-footer p:last-child {
@@ -285,28 +302,32 @@ h4 {
 
 .eyebrow {
   margin-bottom: var(--space-2);
-  color: var(--color-brand-strong);
-  font-size: 0.82rem;
-  font-weight: 800;
-  letter-spacing: 0.04em;
+  color: var(--color-harvest);
+  font-size: 0.74rem;
+  font-weight: 850;
+  letter-spacing: 0.105em;
+  text-transform: uppercase;
 }
 
 .page-heading {
-  max-width: var(--measure);
-  margin-bottom: clamp(var(--space-6), 5vw, var(--space-12));
+  max-width: 58rem;
+  margin-bottom: clamp(var(--space-6), 4vw, var(--space-10));
 }
 
 .page-heading__summary {
+  max-width: 42rem;
   margin-bottom: 0;
   color: var(--color-muted);
-  font-size: clamp(1rem, 2.5vw, 1.15rem);
+  font-size: clamp(1rem, 2.5vw, 1.18rem);
+  line-height: 1.7;
 }
 
 .search-panel {
   min-width: 0;
-  margin-bottom: var(--space-6);
-  padding-block: var(--space-3);
-  border-block: 1px solid var(--color-border);
+  margin-bottom: var(--space-5);
+  padding-block: var(--space-4);
+  border-top: 3px solid var(--color-forest);
+  border-bottom: 1px solid var(--color-border-strong);
 }
 
 .search-panel h2 {
@@ -341,7 +362,7 @@ select {
   min-width: 0;
   min-height: 2.75rem;
   padding: 0.625rem var(--space-3);
-  border: 2px solid var(--color-border-strong);
+  border: 1px solid var(--color-border-strong);
   border-radius: var(--radius-small);
   background: var(--color-surface);
   color: var(--color-text);
@@ -364,7 +385,8 @@ select[aria-invalid="true"] {
   gap: var(--space-2);
   margin-bottom: var(--space-4);
   padding: var(--space-3) var(--space-4);
-  border-left: 0.3rem solid var(--color-error);
+  border-block: 1px solid var(--color-error);
+  border-left: 0.35rem solid var(--color-error);
   background: var(--color-error-soft);
   color: var(--color-error);
 }
@@ -411,7 +433,7 @@ select[aria-invalid="true"] {
   align-items: center;
   justify-content: center;
   padding: 0.625rem var(--space-4);
-  border: 2px solid transparent;
+  border: 1px solid transparent;
   border-radius: var(--radius-small);
   font: inherit;
   font-weight: 750;
@@ -431,14 +453,14 @@ select[aria-invalid="true"] {
 
 .button--secondary {
   margin-top: var(--space-2);
-  border-color: var(--color-brand-strong);
+  border-color: var(--color-border-strong);
   background: var(--color-surface);
   color: var(--color-brand-strong);
 }
 
 .category-nav {
-  margin-bottom: var(--space-3);
-  padding-bottom: var(--space-3);
+  margin-bottom: var(--space-2);
+  padding-bottom: var(--space-2);
   border-bottom: 1px solid var(--color-border);
 }
 
@@ -452,13 +474,14 @@ select[aria-invalid="true"] {
   align-items: center;
   justify-content: space-between;
   padding-block: var(--space-2);
-  color: var(--color-brand-strong);
+  color: var(--color-text);
   cursor: pointer;
   font-weight: 800;
 }
 
 .catalog-search summary::after {
   content: "+";
+  color: var(--color-harvest);
   font-size: 1.15rem;
   line-height: 1;
 }
@@ -484,7 +507,7 @@ select[aria-invalid="true"] {
   justify-content: space-between;
   gap: var(--space-3);
   padding-block: var(--space-2);
-  color: var(--color-brand-strong);
+  color: var(--color-text);
   cursor: pointer;
   font-weight: 800;
 }
@@ -530,20 +553,30 @@ select[aria-invalid="true"] {
   display: flex;
   min-width: 0;
   flex-wrap: wrap;
-  gap: var(--space-2);
+  gap: 0;
+  border: 1px solid var(--color-border-strong);
+  background: var(--color-surface);
+}
+
+.segment-list li {
+  flex: 1 1 7rem;
+}
+
+.segment-list li + li {
+  border-left: 1px solid var(--color-border-strong);
 }
 
 .segment {
+  width: 100%;
   gap: var(--space-1);
-  border-color: var(--color-border-strong);
+  border: 0;
   background: var(--color-surface);
   color: var(--color-text);
 }
 
 .segment--selected {
-  border-color: var(--color-brand-strong);
-  background: var(--color-brand-soft);
-  color: var(--color-brand-strong);
+  background: var(--color-forest);
+  color: var(--color-ivory);
 }
 
 .segment__selected-mark {
@@ -558,7 +591,8 @@ select[aria-invalid="true"] {
   gap: var(--space-3);
   margin-block: var(--space-6);
   padding: var(--space-4);
-  border: 1px solid currentcolor;
+  border-block: 1px solid currentcolor;
+  border-right: 1px solid currentcolor;
   border-left-width: 0.35rem;
   background: var(--color-surface);
 }
@@ -568,6 +602,10 @@ select[aria-invalid="true"] {
   margin-bottom: 0;
 }
 
+.state-notice__body {
+  min-width: 0;
+}
+
 .state-notice__symbol {
   display: inline-grid;
   width: 1.75rem;
@@ -606,7 +644,9 @@ select[aria-invalid="true"] {
   align-items: baseline;
   justify-content: space-between;
   gap: var(--space-4);
-  margin-bottom: var(--space-3);
+  margin-bottom: var(--space-4);
+  padding-top: var(--space-3);
+  border-top: 3px solid var(--color-forest);
 }
 
 .section-heading h2,
@@ -627,6 +667,9 @@ select[aria-invalid="true"] {
   max-width: 100%;
   flex: 0 0 auto;
   color: var(--color-muted);
+  font-size: 0.82rem;
+  font-weight: 800;
+  letter-spacing: 0.03em;
 }
 
 .publication-summary {
@@ -634,9 +677,10 @@ select[aria-invalid="true"] {
   min-width: 0;
   grid-template-columns: repeat(2, minmax(0, 1fr));
   gap: var(--space-3);
-  margin-bottom: var(--space-5);
-  padding-block: var(--space-2);
-  border-block: 1px solid var(--color-border);
+  margin-bottom: var(--space-4);
+  padding: var(--space-2) var(--space-3);
+  border-left: 3px solid var(--color-data);
+  background: var(--color-info-soft);
 }
 
 .publication-summary div,
@@ -650,8 +694,9 @@ select[aria-invalid="true"] {
 .definition-grid dt,
 .comparison-field dt {
   color: var(--color-muted);
-  font-size: 0.8rem;
+  font-size: 0.75rem;
   font-weight: 700;
+  letter-spacing: 0.025em;
 }
 
 .publication-summary dd,
@@ -672,13 +717,17 @@ select[aria-invalid="true"] {
 }
 
 .catalog-group + .catalog-group {
-  margin-top: var(--space-8);
+  margin-top: var(--space-10);
 }
 
 .catalog-group__heading {
   margin-bottom: 0;
-  padding-bottom: var(--space-2);
-  border-bottom: 2px solid var(--color-brand-strong);
+  padding: var(--space-2) var(--space-3);
+  border-bottom: 0;
+  background: var(--color-forest);
+  color: var(--color-ivory);
+  font-size: 0.86rem;
+  letter-spacing: 0.07em;
 }
 
 .ledger-column-head {
@@ -687,12 +736,12 @@ select[aria-invalid="true"] {
 
 .ledger-list {
   min-width: 0;
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
 }
 
 .ledger-row {
   min-width: 0;
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -700,7 +749,7 @@ select[aria-invalid="true"] {
   display: grid;
   grid-template-columns: repeat(2, minmax(0, 1fr));
   gap: var(--space-2) var(--space-3);
-  padding: var(--space-2);
+  padding: var(--space-3);
 }
 
 .ledger-entry__top {
@@ -732,7 +781,7 @@ select[aria-invalid="true"] {
   flex-wrap: wrap;
   gap: var(--space-1) var(--space-3);
   color: var(--color-muted);
-  font-size: 0.8rem;
+  font-size: 0.78rem;
   line-height: 1.4;
 }
 
@@ -776,9 +825,11 @@ select[aria-invalid="true"] {
 }
 
 .ledger-fact--price dd {
-  font-family: "Gowun Batang", serif;
-  font-size: 1.3rem;
-  font-weight: 700;
+  color: var(--color-forest);
+  font-family: inherit;
+  font-size: 1.35rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
   line-height: 1.25;
   overflow-wrap: normal;
   white-space: nowrap;
@@ -786,6 +837,7 @@ select[aria-invalid="true"] {
 
 .ledger-fact--date dd {
   font-size: 0.92rem;
+  font-variant-numeric: tabular-nums;
   overflow-wrap: normal;
   white-space: nowrap;
 }
@@ -803,13 +855,17 @@ select[aria-invalid="true"] {
   min-height: 2.75rem;
   align-items: center;
   color: var(--color-text);
+  font-size: 1.03rem;
+  font-weight: 850;
+  text-decoration-color: var(--color-harvest);
+  text-decoration-thickness: 0.12em;
 }
 
 .ledger-entry__action {
   display: none;
   gap: var(--space-1);
   align-self: center;
-  color: var(--color-muted);
+  color: var(--color-harvest);
   font-weight: 700;
 }
 
@@ -856,7 +912,7 @@ select[aria-invalid="true"] {
 }
 
 .status-text--current {
-  color: var(--color-brand-strong);
+  color: var(--color-forest);
 }
 
 .status-text--stale {
@@ -890,13 +946,16 @@ select[aria-invalid="true"] {
 }
 
 .breadcrumb {
-  margin-bottom: var(--space-6);
+  margin-bottom: var(--space-5);
+  font-size: 0.86rem;
+  font-weight: 800;
 }
 
 .series-nav {
   min-width: 0;
   margin: calc(-1 * var(--space-4)) 0 var(--space-8);
   border-block: 1px solid var(--color-border-strong);
+  background: var(--color-surface);
 }
 
 .series-nav ul {
@@ -919,7 +978,7 @@ select[aria-invalid="true"] {
   min-height: 2.75rem;
   align-items: center;
   padding: var(--space-2) var(--space-3);
-  border-bottom: 3px solid transparent;
+  border-bottom: 4px solid transparent;
   color: var(--color-text);
   font-size: 0.9rem;
   font-weight: 800;
@@ -927,8 +986,8 @@ select[aria-invalid="true"] {
 }
 
 .series-nav__item--current {
-  border-bottom-color: var(--color-brand-strong);
-  color: var(--color-brand-strong);
+  border-bottom-color: var(--color-harvest);
+  color: var(--color-forest);
 }
 
 .detail-actions {
@@ -979,20 +1038,28 @@ select[aria-invalid="true"] {
 
 .detail-intro {
   display: grid;
-  gap: var(--space-8);
+  gap: 0;
   margin-bottom: var(--space-10);
+  border-block: 4px solid var(--color-forest);
+  background: var(--color-surface);
+}
+
+.detail-intro__identity {
+  padding: clamp(var(--space-5), 4vw, var(--space-10));
 }
 
 .detail-signature {
   display: flex;
   min-width: 0;
   flex-wrap: wrap;
-  gap: var(--space-2) var(--space-5);
+  gap: var(--space-2) var(--space-4);
   margin-bottom: 0;
 }
 
 .detail-signature div {
   min-width: 0;
+  padding-right: var(--space-4);
+  border-right: 1px solid var(--color-border);
 }
 
 .detail-signature dt,
@@ -1012,8 +1079,24 @@ select[aria-invalid="true"] {
   display: grid;
   min-width: 0;
   gap: var(--space-2);
-  padding-block: var(--space-5);
-  border-block: 2px solid var(--color-brand-strong);
+  align-content: center;
+  padding: clamp(var(--space-5), 4vw, var(--space-10));
+  border-block: 0;
+  background: var(--color-forest);
+  color: var(--color-ivory);
+}
+
+.current-price .eyebrow {
+  color: #e8b19f;
+}
+
+.current-price h2 {
+  color: var(--color-ivory);
+  font-family: inherit;
+  font-size: 0.78rem;
+  font-weight: 800;
+  letter-spacing: 0.07em;
+  text-transform: uppercase;
 }
 
 .current-price h2,
@@ -1024,9 +1107,10 @@ select[aria-invalid="true"] {
 }
 
 .current-price__value {
-  font-family: "Gowun Batang", serif;
-  font-size: clamp(2.25rem, 9vw, 3.75rem);
-  font-weight: 700;
+  font-family: inherit;
+  font-size: clamp(2.5rem, 10vw, 4.5rem);
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
   letter-spacing: -0.045em;
   line-height: 1.1;
   overflow-wrap: normal;
@@ -1035,7 +1119,7 @@ select[aria-invalid="true"] {
 
 .current-price__date,
 .current-price__note {
-  color: var(--color-muted);
+  color: #dce8dc;
 }
 
 .current-price__note {
@@ -1048,7 +1132,7 @@ select[aria-invalid="true"] {
 .comparison-section {
   min-width: 0;
   padding-top: var(--space-5);
-  border-top: 1px solid var(--color-border-strong);
+  border-top: 3px solid var(--color-forest);
 }
 
 .identity-panel,
@@ -1067,7 +1151,7 @@ select[aria-invalid="true"] {
 .definition-grid div {
   min-width: 0;
   padding-top: var(--space-2);
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
 }
 
 .definition-grid dd {
@@ -1087,12 +1171,12 @@ select[aria-invalid="true"] {
 
 .comparison-list {
   min-width: 0;
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
 }
 
 .comparison-row {
   min-width: 0;
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -1102,6 +1186,7 @@ select[aria-invalid="true"] {
 
 .comparison-row--unavailable {
   color: var(--color-muted);
+  background: var(--color-neutral-soft);
 }
 
 .comparison-row__facts {
@@ -1197,7 +1282,7 @@ select[aria-invalid="true"] {
 .record-heading {
   min-width: 0;
   max-width: 52rem;
-  margin-bottom: var(--space-6);
+  margin-bottom: var(--space-5);
 }
 
 .record-heading h1 {
@@ -1208,15 +1293,17 @@ select[aria-invalid="true"] {
   max-width: var(--measure);
   margin-bottom: var(--space-4);
   color: var(--color-muted);
-  font-size: clamp(1rem, 2.5vw, 1.12rem);
+  font-size: clamp(1rem, 2.5vw, 1.18rem);
+  line-height: 1.7;
 }
 
 .scope-controls,
 .market-scope {
   min-width: 0;
   margin-bottom: var(--space-8);
-  padding-block: var(--space-4);
-  border-block: 1px solid var(--color-border-strong);
+  padding: var(--space-4);
+  border-block: 3px solid var(--color-forest);
+  background: var(--color-surface);
 }
 
 .scope-controls__heading {
@@ -1273,8 +1360,9 @@ select[aria-invalid="true"] {
 
 .history-chart {
   margin: var(--space-5) 0 var(--space-8);
-  padding-block: var(--space-3);
-  border-block: 1px solid var(--color-border);
+  padding: var(--space-4);
+  border-block: 1px solid var(--color-border-strong);
+  background: var(--color-surface);
 }
 
 .history-chart__svg {
@@ -1297,7 +1385,7 @@ select[aria-invalid="true"] {
   stroke: var(--color-data);
   stroke-linecap: round;
   stroke-linejoin: round;
-  stroke-width: 3;
+  stroke-width: 3.5;
 }
 
 .history-chart__point {
@@ -1362,13 +1450,13 @@ select[aria-invalid="true"] {
 }
 
 .month-list {
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
 }
 
 .month-row {
   min-width: 0;
   padding: var(--space-3) var(--space-2);
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -1378,8 +1466,9 @@ select[aria-invalid="true"] {
 
 .month-row__date {
   margin-bottom: var(--space-2);
-  font-family: "Gowun Batang", serif;
-  font-weight: 700;
+  font-family: inherit;
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
 }
 
 .month-row__facts {
@@ -1403,6 +1492,7 @@ select[aria-invalid="true"] {
 .month-row__facts dd {
   margin-bottom: 0;
   font-weight: 750;
+  font-variant-numeric: tabular-nums;
 }
 
 .month-row--unavailable,
@@ -1419,7 +1509,7 @@ select[aria-invalid="true"] {
 }
 
 .region-list {
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
 }
 
 .region-row {
@@ -1428,7 +1518,7 @@ select[aria-invalid="true"] {
   grid-template-columns: repeat(2, minmax(0, 1fr));
   gap: var(--space-2) var(--space-3);
   padding: var(--space-3) var(--space-2);
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -1453,6 +1543,7 @@ select[aria-invalid="true"] {
 .region-row__range dd {
   margin-bottom: 0;
   font-weight: 750;
+  font-variant-numeric: tabular-nums;
 }
 
 .region-row__range dl {
@@ -1519,7 +1610,7 @@ select[aria-invalid="true"] {
 }
 
 .market-list {
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
 }
 
 .market-row {
@@ -1528,7 +1619,7 @@ select[aria-invalid="true"] {
   grid-template-columns: repeat(2, minmax(0, 1fr));
   gap: var(--space-2) var(--space-3);
   padding: var(--space-3) var(--space-2);
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -1556,9 +1647,11 @@ select[aria-invalid="true"] {
 }
 
 .market-row__price dd {
-  font-family: "Gowun Batang", serif;
-  font-size: 1.15rem;
-  font-weight: 700;
+  color: var(--color-forest);
+  font-family: inherit;
+  font-size: 1.2rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
 }
 
 .market-row__date dd {
@@ -1573,8 +1666,9 @@ select[aria-invalid="true"] {
 .selection-add {
   min-width: 0;
   margin-bottom: var(--space-8);
-  padding-block: var(--space-4);
-  border-block: 1px solid var(--color-border-strong);
+  padding: var(--space-4);
+  border-block: 3px solid var(--color-forest);
+  background: var(--color-surface);
 }
 
 .selection-add__heading {
@@ -1635,7 +1729,7 @@ select[aria-invalid="true"] {
 .selection-list {
   margin: 0;
   padding: 0;
-  border-bottom: 1px solid var(--color-border-strong);
+  border-bottom: 2px solid var(--color-forest);
   list-style: none;
 }
 
@@ -1645,7 +1739,7 @@ select[aria-invalid="true"] {
   grid-template-columns: repeat(2, minmax(0, 1fr));
   gap: var(--space-2) var(--space-3);
   padding: var(--space-3) var(--space-2);
-  border-top: 1px solid var(--color-border);
+  border-top: 1px solid var(--color-border-strong);
   background: var(--color-surface);
 }
 
@@ -1691,9 +1785,11 @@ select[aria-invalid="true"] {
 }
 
 .selection-row__price dd {
-  font-family: "Gowun Batang", serif;
-  font-size: 1.2rem;
-  font-weight: 700;
+  color: var(--color-forest);
+  font-family: inherit;
+  font-size: 1.3rem;
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
 }
 
 .selection-row__comparison {
@@ -1721,7 +1817,7 @@ select[aria-invalid="true"] {
   grid-template-columns: auto minmax(0, 1fr);
   gap: var(--space-3);
   padding-block: var(--space-5);
-  border-block: 1px solid var(--color-border-strong);
+  border-block: 3px solid var(--color-forest);
 }
 
 .selection-empty__mark {
@@ -1745,8 +1841,8 @@ select[aria-invalid="true"] {
 
 .error-page {
   max-width: 42rem;
-  padding-block: var(--space-6);
-  border-block: 1px solid var(--color-border-strong);
+  padding-block: var(--space-10);
+  border-block: 4px solid var(--color-forest);
 }
 
 .error-page p:not(.eyebrow) {
@@ -1754,9 +1850,254 @@ select[aria-invalid="true"] {
   color: var(--color-muted);
 }
 
+/* Market-editorial structures introduced by the v2 information architecture. */
+.catalog-hero {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-3);
+  margin-bottom: var(--space-5);
+  padding-top: var(--space-4);
+  border-top: 4px solid var(--color-forest);
+}
+
+.catalog-hero h1,
+.catalog-hero p:last-child {
+  margin-bottom: 0;
+}
+
+.catalog-hero__title,
+.catalog-hero__summary {
+  min-width: 0;
+}
+
+.catalog-toolbar {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2);
+  margin-bottom: var(--space-4);
+  padding-block: var(--space-3);
+  border-block: 1px solid var(--color-border-strong);
+}
+
+.catalog-toolbar__tools,
+.catalog-tools {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2);
+}
+
+.catalog-meta {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  align-items: baseline;
+  justify-content: space-between;
+  gap: var(--space-2) var(--space-4);
+  margin-bottom: var(--space-3);
+}
+
+.catalog-meta .publication-summary {
+  width: 100%;
+  margin-bottom: 0;
+}
+
+.catalog-meta > :where(h2, p, dl) {
+  margin-bottom: 0;
+}
+
+.detail-intro__lead {
+  min-width: 0;
+}
+
+.detail-intro__summary {
+  max-width: 36rem;
+  margin: var(--space-4) 0 0;
+  color: var(--color-muted);
+}
+
+.detail-pathways {
+  margin: calc(-1 * var(--space-5)) 0 var(--space-8);
+  border-block: 1px solid var(--color-border-strong);
+  background: var(--color-surface);
+}
+
+.detail-pathways .series-nav {
+  margin: 0;
+  border: 0;
+}
+
+.detail-pathways .detail-actions {
+  margin: 0;
+  padding: var(--space-2) var(--space-3);
+  border-top: 1px solid var(--color-border);
+}
+
+.detail-pathways ul {
+  display: grid;
+  min-width: 0;
+  margin: 0;
+  padding: 0;
+  list-style: none;
+}
+
+.detail-pathways li + li {
+  border-top: 1px solid var(--color-border);
+}
+
+.detail-pathways a {
+  display: flex;
+  min-width: 2.75rem;
+  min-height: 3.25rem;
+  align-items: center;
+  justify-content: space-between;
+  gap: var(--space-3);
+  padding: var(--space-2) var(--space-3);
+  color: var(--color-text);
+  font-weight: 800;
+  text-decoration: none;
+}
+
+.history-summary,
+.market-summary {
+  display: grid;
+  min-width: 0;
+  margin: 0 0 var(--space-8);
+  border-block: 4px solid var(--color-forest);
+  background: var(--color-surface);
+}
+
+.history-summary__item,
+.market-summary__item {
+  min-width: 0;
+  padding: var(--space-4);
+}
+
+.history-summary__item + .history-summary__item,
+.market-summary__item + .market-summary__item {
+  border-top: 1px solid var(--color-border-strong);
+}
+
+.history-summary dt,
+.market-summary dt {
+  margin-bottom: var(--space-1);
+  color: var(--color-muted);
+  font-size: 0.72rem;
+  font-weight: 800;
+  letter-spacing: 0.055em;
+}
+
+.history-summary dd,
+.market-summary dd {
+  margin-bottom: 0;
+  color: var(--color-forest);
+  font-size: clamp(1.35rem, 5vw, 2rem);
+  font-variant-numeric: tabular-nums;
+  font-weight: 850;
+  line-height: 1.2;
+}
+
+.history-summary__period {
+  display: block;
+  margin-top: var(--space-1);
+  color: var(--color-muted);
+  font-size: 0.76rem;
+  font-weight: 650;
+}
+
+.history-year-groups {
+  display: grid;
+  gap: var(--space-5);
+}
+
+.history-year {
+  min-width: 0;
+  border-top: 3px solid var(--color-forest);
+}
+
+.history-year > summary {
+  display: flex;
+  min-height: 3.5rem;
+  align-items: center;
+  justify-content: space-between;
+  gap: var(--space-4);
+  padding: var(--space-2) var(--space-3);
+  background: var(--color-surface-muted);
+  color: var(--color-text);
+  cursor: pointer;
+  font-weight: 850;
+  list-style: none;
+}
+
+.history-year > summary::-webkit-details-marker {
+  display: none;
+}
+
+.history-year > summary::after {
+  content: "+";
+  color: var(--color-harvest);
+  font-size: 1.2rem;
+}
+
+.history-year[open] > summary::after {
+  content: "−";
+}
+
+.history-year__label {
+  font-size: 1.1rem;
+}
+
+.history-year__count {
+  margin-left: auto;
+  color: var(--color-muted);
+  font-size: 0.76rem;
+  font-weight: 700;
+}
+
+.history-year .month-ledger {
+  border-bottom: 1px solid var(--color-border-strong);
+}
+
+.region-section__intro {
+  display: grid;
+  min-width: 0;
+  gap: var(--space-2);
+  margin-bottom: var(--space-4);
+  padding-top: var(--space-3);
+  border-top: 3px solid var(--color-forest);
+}
+
+.region-section__intro > :last-child {
+  max-width: var(--measure);
+  margin-bottom: 0;
+  color: var(--color-muted);
+}
+
+.selection-results-first {
+  margin-bottom: var(--space-10);
+}
+
+.error-page__eyebrow {
+  color: var(--color-harvest);
+  font-size: 0.74rem;
+  font-weight: 850;
+  letter-spacing: 0.105em;
+}
+
+.error-page__actions {
+  display: flex;
+  flex-wrap: wrap;
+  gap: var(--space-2);
+  margin-top: var(--space-6);
+}
+
+.error-page__summary {
+  font-size: clamp(1rem, 2.5vw, 1.18rem);
+  line-height: 1.7;
+}
+
 @media (hover: hover) {
   .button--primary:hover {
-    background: var(--color-brand);
+    background: var(--color-harvest);
   }
 
   .button--secondary:hover,
@@ -1778,11 +2119,88 @@ select[aria-invalid="true"] {
     background: var(--color-brand-soft);
   }
 
+  .ledger-row:hover,
+  .comparison-row:hover,
+  .month-row:hover,
+  .region-row:hover,
+  .market-row:hover,
+  .selection-row:hover {
+    background: #f7f0e2;
+  }
+
+  .detail-pathways a:hover,
+  .history-year > summary:hover {
+    background: var(--color-brand-soft);
+  }
+
 }
 
 @media (max-width: 39.98rem) {
+  .site-header__inner {
+    min-height: 4.5rem;
+  }
+
+  .brand-mark {
+    width: 2.4rem;
+    height: 2.4rem;
+  }
+
+  .brand__name {
+    font-size: 1.2rem;
+  }
+
+  .brand__description {
+    font-size: 0.68rem;
+  }
+
+  .site-actions a {
+    font-size: 0.78rem;
+  }
+
+  .page-main {
+    padding-top: var(--space-5);
+  }
+
   .page-heading--catalog {
-    margin-bottom: var(--space-4);
+    margin-bottom: var(--space-3);
+  }
+
+  .catalog-hero {
+    gap: var(--space-2);
+    padding-top: var(--space-3);
+  }
+
+  .catalog-hero h1 {
+    margin-bottom: var(--space-2);
+    font-size: clamp(2.15rem, 11vw, 2.7rem);
+  }
+
+  .catalog-hero__summary {
+    font-size: 0.92rem;
+    line-height: 1.55;
+  }
+
+  .catalog-toolbar {
+    margin-bottom: var(--space-3);
+    padding-block: var(--space-2);
+  }
+
+  .catalog-toolbar__tools,
+  .catalog-tools {
+    grid-template-columns: repeat(2, minmax(0, 1fr));
+  }
+
+  .catalog-toolbar__tools > details[open],
+  .catalog-tools > details[open] {
+    grid-column: 1 / -1;
+  }
+
+  .catalog-toolbar .catalog-options {
+    margin-top: 0;
+  }
+
+  .catalog-toolbar .catalog-options__current {
+    display: none;
   }
 
   .search-panel h2 {
@@ -1794,7 +2212,7 @@ select[aria-invalid="true"] {
     flex-wrap: wrap;
     gap: var(--space-1) var(--space-4);
     margin-bottom: var(--space-2);
-    padding-block: var(--space-1);
+    padding: var(--space-2);
   }
 
   .publication-summary div {
@@ -1805,6 +2223,27 @@ select[aria-invalid="true"] {
     gap: var(--space-1);
   }
 
+  .catalog-results__heading {
+    margin-bottom: var(--space-2);
+    padding-top: var(--space-2);
+  }
+
+  .catalog-results__heading h2 {
+    font-size: 1.45rem;
+  }
+
+  .catalog-meta {
+    margin-bottom: var(--space-2);
+  }
+
+  .catalog-group__heading {
+    padding-block: 0.45rem;
+  }
+
+  .ledger-entry {
+    padding-block: var(--space-2);
+  }
+
   .comparison-row__facts {
     grid-template-columns: minmax(0, 1fr);
   }
@@ -1822,9 +2261,45 @@ select[aria-invalid="true"] {
   .region-row__action {
     grid-column: auto;
   }
+
+  .history-summary,
+  .market-summary {
+    margin-bottom: var(--space-6);
+  }
+
+  .history-summary__item,
+  .market-summary__item {
+    display: grid;
+    grid-template-columns: minmax(0, 0.8fr) minmax(0, 1.2fr);
+    align-items: baseline;
+    gap: var(--space-3);
+    padding-block: var(--space-3);
+  }
+
+  .history-summary dd,
+  .market-summary dd {
+    font-size: 1.35rem;
+    text-align: right;
+  }
+
+  .history-summary__period {
+    text-align: right;
+  }
 }
 
 @media (min-width: 40rem) {
+  .catalog-hero {
+    grid-template-columns: minmax(15rem, 0.95fr) minmax(0, 1.05fr);
+    align-items: end;
+    gap: clamp(var(--space-6), 5vw, var(--space-12));
+    padding-block: var(--space-5);
+  }
+
+  .catalog-hero__summary {
+    padding-bottom: var(--space-2);
+    border-bottom: 1px solid var(--color-border-strong);
+  }
+
   .search-form {
     grid-template-columns: minmax(0, 1fr) auto;
     align-items: end;
@@ -1857,6 +2332,17 @@ select[aria-invalid="true"] {
     align-items: end;
   }
 
+  .region-scope {
+    display: grid;
+    grid-template-columns: minmax(12rem, 0.75fr) minmax(0, 1.25fr);
+    align-items: end;
+    gap: var(--space-6);
+  }
+
+  .region-scope .scope-controls__heading {
+    margin-bottom: 0;
+  }
+
   .selection-add__form {
     grid-template-columns: minmax(0, 1fr) auto;
     align-items: end;
@@ -1871,9 +2357,9 @@ select[aria-invalid="true"] {
   .detail-intro {
     grid-template-columns: minmax(0, 1fr) minmax(18rem, 0.75fr);
     align-items: stretch;
-    gap: var(--space-6);
-    padding-block: var(--space-5);
-    border-block: 2px solid var(--color-brand-strong);
+    gap: 0;
+    padding-block: 0;
+    border-block: 4px solid var(--color-forest);
   }
 
   .detail-intro__identity {
@@ -1882,10 +2368,9 @@ select[aria-invalid="true"] {
 
   .current-price {
     align-content: center;
-    padding-block: 0;
-    padding-left: var(--space-6);
+    padding: clamp(var(--space-6), 4vw, var(--space-10));
     border-block: 0;
-    border-left: 1px solid var(--color-border);
+    border-left: 0;
   }
 
   .current-price__value {
@@ -1895,9 +2380,68 @@ select[aria-invalid="true"] {
   .comparison-row__facts {
     gap: var(--space-2) var(--space-4);
   }
+
+  .detail-pathways {
+    display: grid;
+    grid-template-columns: minmax(0, 1fr) auto;
+    align-items: stretch;
+  }
+
+  .detail-pathways .series-nav ul {
+    display: flex;
+  }
+
+  .detail-pathways .detail-actions {
+    display: flex;
+    align-items: center;
+    border-top: 0;
+    border-left: 1px solid var(--color-border);
+  }
+
+  .history-summary,
+  .market-summary {
+    grid-template-columns: repeat(3, minmax(0, 1fr));
+  }
+
+  .history-summary__item + .history-summary__item,
+  .market-summary__item + .market-summary__item {
+    border-top: 0;
+    border-left: 1px solid var(--color-border-strong);
+  }
 }
 
 @media (min-width: 64rem) {
+  .catalog-toolbar {
+    grid-template-columns: minmax(20rem, 0.9fr) minmax(28rem, 1.1fr);
+    align-items: start;
+    gap: var(--space-6);
+  }
+
+  .catalog-toolbar .category-nav {
+    grid-column: 1;
+    margin-bottom: 0;
+    padding-bottom: 0;
+    border-bottom: 0;
+  }
+
+  .catalog-toolbar__tools,
+  .catalog-tools {
+    grid-column: 2;
+    grid-template-columns: repeat(2, minmax(0, 1fr));
+    gap: var(--space-4);
+  }
+
+  .catalog-toolbar .catalog-search,
+  .catalog-toolbar .catalog-options {
+    margin-top: 0;
+    border-top: 0;
+    border-bottom: 1px solid var(--color-border-strong);
+  }
+
+  .catalog-toolbar > .form-error {
+    grid-column: 1 / -1;
+  }
+
   .ledger-entry__top {
     display: contents;
   }
@@ -1916,14 +2460,16 @@ select[aria-invalid="true"] {
   .ledger-column-head {
     display: grid;
     padding: var(--space-2) var(--space-3);
-    border-bottom: 1px solid var(--color-border-strong);
-    color: var(--color-muted);
+    border-bottom: 2px solid var(--color-forest);
+    background: var(--color-surface-muted);
+    color: var(--color-text);
     font-size: 0.78rem;
     font-weight: 800;
   }
 
   .ledger-entry {
     align-items: center;
+    min-height: 5.75rem;
     padding: var(--space-3);
   }
 
@@ -2003,9 +2549,10 @@ select[aria-invalid="true"] {
   .comparison-column-head {
     display: grid;
     padding: var(--space-2);
-    border-top: 2px solid var(--color-brand-strong);
+    border-top: 3px solid var(--color-forest);
     border-bottom: 1px solid var(--color-border-strong);
-    color: var(--color-muted);
+    background: var(--color-surface-muted);
+    color: var(--color-text);
     font-size: 0.78rem;
     font-weight: 800;
   }
@@ -2045,9 +2592,10 @@ select[aria-invalid="true"] {
     display: grid;
     grid-template-columns: minmax(8rem, 0.8fr) repeat(3, minmax(8rem, 1fr));
     padding: var(--space-2);
-    border-top: 2px solid var(--color-brand-strong);
+    border-top: 3px solid var(--color-forest);
     border-bottom: 1px solid var(--color-border-strong);
-    color: var(--color-muted);
+    background: var(--color-surface-muted);
+    color: var(--color-text);
     font-size: 0.78rem;
     font-weight: 800;
   }
@@ -2090,9 +2638,10 @@ select[aria-invalid="true"] {
   .region-ledger__head {
     display: grid;
     padding: var(--space-2);
-    border-top: 2px solid var(--color-brand-strong);
+    border-top: 3px solid var(--color-forest);
     border-bottom: 1px solid var(--color-border-strong);
-    color: var(--color-muted);
+    background: var(--color-surface-muted);
+    color: var(--color-text);
     font-size: 0.78rem;
     font-weight: 800;
   }
@@ -2132,9 +2681,10 @@ select[aria-invalid="true"] {
   .market-ledger__head {
     display: grid;
     padding: var(--space-2);
-    border-top: 2px solid var(--color-brand-strong);
+    border-top: 3px solid var(--color-forest);
     border-bottom: 1px solid var(--color-border-strong);
-    color: var(--color-muted);
+    background: var(--color-surface-muted);
+    color: var(--color-text);
     font-size: 0.78rem;
     font-weight: 800;
   }
@@ -2226,6 +2776,10 @@ select[aria-invalid="true"] {
   .comparison-row,
   .search-panel,
   .current-price,
+  .detail-pathways,
+  .history-summary,
+  .market-summary,
+  .history-year,
   .identity-panel,
   .provenance,
   .error-page {
@@ -2237,6 +2791,10 @@ select[aria-invalid="true"] {
     forced-color-adjust: auto;
   }
 
+  .brand-mark {
+    filter: none;
+  }
+
   .comparison-meter__rail,
   .comparison-meter__zero,
   .comparison-meter__cap {


