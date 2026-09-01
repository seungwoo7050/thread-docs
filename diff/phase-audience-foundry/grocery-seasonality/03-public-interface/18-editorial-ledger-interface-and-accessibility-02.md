## `feat: introduce the ledger visual system`

diff --git a/grocery/static/grocery/app.css b/grocery/static/grocery/app.css
index c979948..83eed99 100644
--- a/grocery/static/grocery/app.css
+++ b/grocery/static/grocery/app.css
@@ -1,26 +1,49 @@
+@font-face {
+  font-family: "Gowun Batang";
+  src: url("fonts/gowun-batang-bold.woff2") format("woff2");
+  font-style: normal;
+  font-weight: 700;
+  font-display: swap;
+}
+
 :root {
   color-scheme: light;
-  --color-canvas: #f6f7f2;
-  --color-surface: #ffffff;
-  --color-text: #17201a;
+  --color-canvas: #f4f0e6;
+  --color-surface: #fffdf7;
+  --color-surface-muted: #eee8da;
+  --color-text: #1d2820;
   --color-muted: #536057;
-  --color-border: #c7cec8;
-  --color-brand: #175b3a;
-  --color-brand-strong: #0d442a;
-  --color-brand-soft: #e8f2ec;
-  --color-info: #165273;
-  --color-info-soft: #eaf4f8;
-  --color-warning: #745000;
-  --color-warning-soft: #fff4d6;
-  --color-error: #8b1e24;
-  --color-error-soft: #fff0f0;
-  --color-neutral-soft: #eef1ee;
-  --focus-ring: #0969da;
-  --shadow-card: 0 0.2rem 0.8rem rgb(23 32 26 / 8%);
-  --radius-small: 0.45rem;
-  --radius-medium: 0.8rem;
+  --color-border: #bdb6a5;
+  --color-border-strong: #736d5f;
+  --color-brand: #286442;
+  --color-brand-strong: #17472f;
+  --color-brand-soft: #e3eee5;
+  --color-info: #245b73;
+  --color-info-soft: #e8f2f5;
+  --color-warning: #704b00;
+  --color-warning-soft: #fff1c9;
+  --color-error: #8b2830;
+  --color-error-soft: #fbe9e6;
+  --color-neutral-soft: #eceee9;
+  --color-data: #245b73;
+  --color-lower: #245b73;
+  --color-higher: #245b73;
+  --color-focus: #005fcc;
+  --color-on-brand: #ffffff;
+  --space-1: 0.25rem;
+  --space-2: 0.5rem;
+  --space-3: 0.75rem;
+  --space-4: 1rem;
+  --space-5: 1.25rem;
+  --space-6: 1.5rem;
+  --space-8: 2rem;
+  --space-10: 2.5rem;
+  --space-12: 3rem;
+  --space-16: 4rem;
+  --radius-small: 0.25rem;
   --page-gutter: clamp(1rem, 4vw, 2rem);
   --measure: 44rem;
+  --page-width: 72rem;
 }
 
 *,
@@ -58,10 +81,17 @@ svg {
   max-width: 100%;
 }
 
+time,
+data,
+.comparison-field--reference strong {
+  overflow-wrap: normal;
+  white-space: nowrap;
+}
+
 a {
   color: var(--color-brand-strong);
-  text-underline-offset: 0.18em;
   text-decoration-thickness: 0.08em;
+  text-underline-offset: 0.18em;
 }
 
 a:hover {
@@ -69,13 +99,14 @@ a:hover {
 }
 
 :focus-visible {
-  outline: 3px solid var(--focus-ring);
+  outline: 3px solid var(--color-focus);
   outline-offset: 3px;
 }
 
 h1,
 h2,
 h3,
+h4,
 p,
 dl,
 dd {
@@ -84,31 +115,52 @@ dd {
 
 h1,
 h2,
-h3 {
+h3,
+h4,
+.brand__name {
+  font-family: "Gowun Batang", serif;
+  font-weight: 700;
   line-height: 1.25;
   text-wrap: balance;
 }
 
 h1 {
-  max-width: 22ch;
-  margin-bottom: 0.75rem;
-  font-size: clamp(2rem, 7vw, 3.75rem);
-  letter-spacing: -0.04em;
+  max-width: 24ch;
+  margin-bottom: var(--space-3);
+  font-size: clamp(2rem, 7vw, 3.5rem);
+  letter-spacing: -0.045em;
 }
 
 h2 {
-  margin-bottom: 0.75rem;
-  font-size: clamp(1.35rem, 4vw, 1.8rem);
+  margin-bottom: var(--space-3);
+  font-size: clamp(1.35rem, 4vw, 1.75rem);
   letter-spacing: -0.025em;
 }
 
 h3 {
-  margin-bottom: 0.5rem;
+  margin-bottom: var(--space-2);
   font-size: 1.15rem;
 }
 
+h4 {
+  margin-bottom: var(--space-1);
+  font-size: 1.08rem;
+}
+
+.visually-hidden {
+  position: absolute !important;
+  width: 1px !important;
+  height: 1px !important;
+  padding: 0 !important;
+  margin: -1px !important;
+  overflow: hidden !important;
+  clip: rect(0, 0, 0, 0) !important;
+  white-space: nowrap !important;
+  border: 0 !important;
+}
+
 .page-shell {
-  width: min(100%, 76rem);
+  width: min(100%, var(--page-width));
   min-width: 0;
   margin-inline: auto;
   padding-inline: var(--page-gutter);
@@ -116,28 +168,22 @@ h3 {
 
 .page-main {
   min-height: 65vh;
-  padding-block: clamp(2rem, 7vw, 5rem);
-}
-
-.qa-notice {
-  margin-bottom: 1.5rem;
-  padding: 0.65rem 0.8rem;
-  border: 2px dashed var(--color-warning);
-  background: var(--color-warning-soft);
-  color: var(--color-warning);
-  font-weight: 800;
+  padding-block: clamp(var(--space-6), 7vw, var(--space-16));
 }
 
 .skip-link {
   position: fixed;
   z-index: 100;
-  top: 0.75rem;
-  left: 0.75rem;
+  top: var(--space-3);
+  left: var(--space-3);
+  display: inline-flex;
+  min-width: 2.75rem;
   min-height: 2.75rem;
-  padding: 0.65rem 1rem;
+  align-items: center;
+  padding: 0.625rem var(--space-4);
   border-radius: var(--radius-small);
   background: var(--color-text);
-  color: #fff;
+  color: var(--color-on-brand);
   transform: translateY(-200%);
 }
 
@@ -145,36 +191,50 @@ h3 {
   transform: translateY(0);
 }
 
+.masthead,
 .site-header {
   border-bottom: 1px solid var(--color-border);
   background: var(--color-surface);
 }
 
+.masthead__inner,
 .site-header__inner {
   display: flex;
+  min-height: 4.75rem;
   align-items: center;
-  min-height: 5rem;
 }
 
 .brand {
   display: inline-flex;
   min-width: 0;
   min-height: 2.75rem;
-  flex-direction: column;
-  justify-content: center;
+  align-items: center;
+  gap: var(--space-3);
   color: var(--color-text);
   text-decoration: none;
 }
 
+.brand-mark {
+  width: 2.75rem;
+  height: 2.75rem;
+  flex: 0 0 auto;
+}
+
+.brand-copy {
+  display: flex;
+  min-width: 0;
+  flex-direction: column;
+  justify-content: center;
+}
+
 .brand__name {
-  font-size: 1.08rem;
-  font-weight: 800;
-  letter-spacing: -0.02em;
+  font-size: 1.2rem;
+  letter-spacing: -0.025em;
 }
 
 .brand__description {
   color: var(--color-muted);
-  font-size: 0.85rem;
+  font-size: 0.82rem;
 }
 
 .site-footer {
@@ -185,7 +245,7 @@ h3 {
 }
 
 .site-footer__inner {
-  padding-block: 1.75rem;
+  padding-block: var(--space-6);
 }
 
 .site-footer p:last-child {
@@ -193,60 +253,39 @@ h3 {
 }
 
 .eyebrow {
-  margin-bottom: 0.5rem;
+  margin-bottom: var(--space-2);
   color: var(--color-brand-strong);
-  font-size: 0.83rem;
+  font-size: 0.82rem;
   font-weight: 800;
   letter-spacing: 0.04em;
 }
 
 .page-heading {
   max-width: var(--measure);
-  margin-bottom: clamp(2rem, 5vw, 3.5rem);
+  margin-bottom: clamp(var(--space-6), 5vw, var(--space-12));
 }
 
 .page-heading__summary {
   margin-bottom: 0;
   color: var(--color-muted);
-  font-size: clamp(1rem, 2.5vw, 1.2rem);
+  font-size: clamp(1rem, 2.5vw, 1.15rem);
 }
 
-.identity-summary {
-  display: flex;
-  flex-wrap: wrap;
-  gap: 0.5rem 1rem;
-}
-
-.identity-summary span {
-  min-width: 0;
-}
-
-.search-panel,
-.identity-panel,
-.provenance,
-.current-price,
-.error-page {
+.search-panel {
   min-width: 0;
-  border: 1px solid var(--color-border);
-  border-radius: var(--radius-medium);
-  background: var(--color-surface);
-  box-shadow: var(--shadow-card);
+  margin-bottom: var(--space-6);
+  padding-block: var(--space-3);
+  border-block: 1px solid var(--color-border);
 }
 
-.search-panel,
-.identity-panel,
-.provenance {
-  padding: clamp(1rem, 4vw, 2rem);
-}
-
-.search-panel {
-  margin-bottom: 2rem;
+.search-panel h2 {
+  margin-bottom: var(--space-4);
 }
 
 .search-form {
   display: grid;
   min-width: 0;
-  gap: 0.75rem;
+  gap: var(--space-3);
 }
 
 .field-group {
@@ -255,12 +294,12 @@ h3 {
 
 .field-group label {
   display: block;
-  margin-bottom: 0.2rem;
+  margin-bottom: var(--space-1);
   font-weight: 750;
 }
 
 .field-hint {
-  margin-bottom: 0.45rem;
+  margin-bottom: var(--space-2);
   color: var(--color-muted);
   font-size: 0.9rem;
 }
@@ -269,8 +308,8 @@ input[type="search"] {
   width: 100%;
   min-width: 0;
   min-height: 2.75rem;
-  padding: 0.68rem 0.8rem;
-  border: 2px solid #667269;
+  padding: 0.625rem var(--space-3);
+  border: 2px solid var(--color-border-strong);
   border-radius: var(--radius-small);
   background: var(--color-surface);
   color: var(--color-text);
@@ -283,13 +322,26 @@ input[aria-invalid="true"] {
 
 .form-error {
   display: flex;
+  min-width: 0;
   align-items: flex-start;
-  gap: 0.6rem;
-  margin-bottom: 1rem;
-  padding: 0.8rem;
+  gap: var(--space-2);
+  margin-bottom: var(--space-4);
+  padding: var(--space-3) var(--space-4);
   border-left: 0.3rem solid var(--color-error);
   background: var(--color-error-soft);
-  color: #68141a;
+  color: var(--color-error);
+}
+
+.form-error__symbol {
+  display: inline-grid;
+  width: 1.5rem;
+  height: 1.5rem;
+  flex: 0 0 auto;
+  place-items: center;
+  border: 2px solid currentcolor;
+  border-radius: 50%;
+  font-weight: 900;
+  line-height: 1;
 }
 
 .form-error__content {
@@ -297,30 +349,31 @@ input[aria-invalid="true"] {
 }
 
 .form-error__content p {
-  margin-bottom: 0.25rem;
+  margin-bottom: var(--space-1);
 }
 
-.form-error__link {
-  display: inline-flex;
-  min-height: 2.75rem;
-  align-items: center;
-  font-weight: 750;
+.form-error__content ul {
+  margin-block: 0;
+  padding-left: var(--space-5);
 }
 
+.form-error__link,
 .breadcrumb a,
 .provenance a {
   display: inline-flex;
+  min-width: 2.75rem;
   min-height: 2.75rem;
   align-items: center;
 }
 
 .button,
-.chip {
+.segment {
   display: inline-flex;
+  min-width: 2.75rem;
   min-height: 2.75rem;
   align-items: center;
   justify-content: center;
-  padding: 0.62rem 1rem;
+  padding: 0.625rem var(--space-4);
   border: 2px solid transparent;
   border-radius: var(--radius-small);
   font: inherit;
@@ -328,56 +381,59 @@ input[aria-invalid="true"] {
   line-height: 1.2;
   text-align: center;
   text-decoration: none;
+}
+
+.button {
   cursor: pointer;
 }
 
 .button--primary {
   background: var(--color-brand-strong);
-  color: #fff;
+  color: var(--color-on-brand);
 }
 
 .button--secondary {
-  margin-top: 0.45rem;
+  margin-top: var(--space-2);
   border-color: var(--color-brand-strong);
   background: var(--color-surface);
   color: var(--color-brand-strong);
 }
 
 .category-nav {
-  margin-top: 1.25rem;
-  padding-top: 1.25rem;
-  border-top: 1px solid var(--color-border);
+  margin-bottom: var(--space-3);
+  padding-bottom: var(--space-3);
+  border-bottom: 1px solid var(--color-border);
 }
 
-.chip-list,
-.result-list,
-.comparison-list,
-.breadcrumb ol {
+.segment-list,
+.ledger-list,
+.comparison-list {
   margin: 0;
   padding: 0;
   list-style: none;
 }
 
-.chip-list {
+.segment-list {
   display: flex;
   min-width: 0;
   flex-wrap: wrap;
-  gap: 0.5rem;
+  gap: var(--space-2);
 }
 
-.chip {
-  border-color: var(--color-border);
+.segment {
+  gap: var(--space-1);
+  border-color: var(--color-border-strong);
   background: var(--color-surface);
   color: var(--color-text);
-  gap: 0.35rem;
 }
 
-.chip--selected {
+.segment--selected {
   border-color: var(--color-brand-strong);
   background: var(--color-brand-soft);
+  color: var(--color-brand-strong);
 }
 
-.chip__selected-mark {
+.segment__selected-mark {
   flex: 0 0 auto;
   font-weight: 900;
 }
@@ -386,12 +442,12 @@ input[aria-invalid="true"] {
   display: grid;
   min-width: 0;
   grid-template-columns: auto minmax(0, 1fr);
-  gap: 0.8rem;
-  margin-block: 1.5rem;
-  padding: clamp(1rem, 4vw, 1.5rem);
+  gap: var(--space-3);
+  margin-block: var(--space-6);
+  padding: var(--space-4);
   border: 1px solid currentcolor;
   border-left-width: 0.35rem;
-  border-radius: var(--radius-medium);
+  background: var(--color-surface);
 }
 
 .state-notice p:last-child,
@@ -436,8 +492,8 @@ input[aria-invalid="true"] {
   flex-wrap: wrap;
   align-items: baseline;
   justify-content: space-between;
-  gap: 1rem;
-  margin-bottom: 1rem;
+  gap: var(--space-4);
+  margin-bottom: var(--space-3);
 }
 
 .section-heading h2,
@@ -460,89 +516,203 @@ input[aria-invalid="true"] {
   color: var(--color-muted);
 }
 
-.result-list {
+.publication-summary {
   display: grid;
   min-width: 0;
-  gap: 1rem;
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+  gap: var(--space-3);
+  margin-bottom: var(--space-5);
+  padding-block: var(--space-2);
+  border-block: 1px solid var(--color-border);
 }
 
-.result-card,
-.result-card__link,
-.result-card__heading,
-.result-card__facts,
-.result-card__facts div {
+.publication-summary div,
+.publication-summary dd {
   min-width: 0;
 }
 
-.result-card {
-  border: 1px solid var(--color-border);
-  border-radius: var(--radius-medium);
-  background: var(--color-surface);
-  box-shadow: var(--shadow-card);
+.publication-summary dt,
+.ledger-fact dt,
+.detail-signature dt,
+.definition-grid dt,
+.comparison-field dt {
+  color: var(--color-muted);
+  font-size: 0.8rem;
+  font-weight: 700;
 }
 
-.result-card__link {
-  display: grid;
-  min-height: 2.75rem;
-  gap: 1rem;
-  padding: clamp(1rem, 4vw, 1.5rem);
-  color: inherit;
-  text-decoration: none;
+.publication-summary dd,
+.ledger-fact dd,
+.detail-signature dd,
+.definition-grid dd,
+.comparison-field dd {
+  margin-bottom: 0;
 }
 
-.result-card__link:hover {
-  border-radius: inherit;
-  background: var(--color-brand-soft);
+.catalog-ledger,
+.catalog-group,
+.ledger-entry,
+.ledger-entry__heading,
+.ledger-fact {
+  min-width: 0;
 }
 
-.result-card__category {
-  margin-bottom: 0.25rem;
-  color: var(--color-brand-strong);
-  font-size: 0.85rem;
-  font-weight: 800;
+.catalog-group + .catalog-group {
+  margin-top: var(--space-8);
 }
 
-.result-card__identity {
+.catalog-group__heading {
   margin-bottom: 0;
-  color: var(--color-muted);
+  padding-bottom: var(--space-2);
+  border-bottom: 2px solid var(--color-brand-strong);
+}
+
+.ledger-column-head {
+  display: none;
+}
+
+.ledger-list {
+  min-width: 0;
+  border-bottom: 1px solid var(--color-border-strong);
+}
+
+.ledger-row {
+  min-width: 0;
+  border-top: 1px solid var(--color-border);
+  background: var(--color-surface);
 }
 
-.result-card__facts {
+.ledger-entry {
   display: grid;
-  grid-template-columns: repeat(auto-fit, minmax(min(100%, 10rem), 1fr));
-  gap: 0.75rem;
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+  gap: var(--space-2) var(--space-3);
+  padding: var(--space-2);
+}
+
+.ledger-entry__heading h4,
+.ledger-entry__identity,
+.ledger-fact,
+.ledger-fact dd {
   margin-bottom: 0;
 }
 
-.result-card__facts div {
-  padding-top: 0.75rem;
+.ledger-entry__heading {
+  display: grid;
+  grid-row: 1;
+  grid-column: 1 / -1;
+  grid-template-columns: auto minmax(0, 1fr);
+  align-items: center;
+  gap: var(--space-1) var(--space-3);
+}
+
+.ledger-entry__identity {
+  display: flex;
+  min-width: 0;
+  flex-wrap: wrap;
+  gap: var(--space-1) var(--space-3);
+  color: var(--color-muted);
+  font-size: 0.8rem;
+  line-height: 1.4;
+}
+
+.ledger-entry__identity span {
+  min-width: 0;
+}
+
+.ledger-fact {
+  display: grid;
+  grid-template-columns: auto minmax(0, 1fr);
+  align-items: baseline;
+  gap: var(--space-1);
+  padding-top: var(--space-1);
   border-top: 1px solid var(--color-border);
 }
 
-.result-card__facts dt,
-.definition-grid dt {
+.ledger-fact--price {
+  grid-row: 2;
+  grid-column: 1;
+}
+
+.ledger-fact--date {
+  grid-row: 2;
+  grid-column: 2;
+}
+
+.ledger-fact--comparison {
+  grid-row: 3;
+  grid-column: 1 / -1;
+}
+
+.ledger-fact--price,
+.ledger-fact--date {
+  display: block;
+}
+
+.ledger-fact--price dt,
+.ledger-fact--date dt {
+  display: block;
+  margin-bottom: var(--space-1);
+}
+
+.ledger-fact--price dd {
+  font-family: "Gowun Batang", serif;
+  font-size: 1.3rem;
+  font-weight: 700;
+  line-height: 1.25;
+  overflow-wrap: normal;
+  white-space: nowrap;
+}
+
+.ledger-fact--date dd {
+  font-size: 0.92rem;
+  overflow-wrap: normal;
+  white-space: nowrap;
+}
+
+.ledger-fact--comparison .direction,
+.ledger-fact--comparison .status-text {
+  font-size: 0.86rem;
+  line-height: 1.45;
+}
+
+.ledger-entry__link {
+  display: inline-flex;
+  width: fit-content;
+  min-width: 2.75rem;
+  min-height: 2.75rem;
+  align-items: center;
+  color: var(--color-text);
+}
+
+.ledger-entry__action {
+  display: none;
+  gap: var(--space-1);
+  align-self: center;
   color: var(--color-muted);
-  font-size: 0.83rem;
   font-weight: 700;
 }
 
-.result-card__facts dd,
-.definition-grid dd {
-  margin-bottom: 0;
-  font-weight: 720;
+.button:active,
+.segment:active,
+.ledger-entry__link:active {
+  transform: translateY(1px);
 }
 
-.result-card__action {
-  justify-self: start;
-  color: var(--color-brand-strong);
-  font-weight: 800;
+.button:active,
+.segment:active {
+  border-color: var(--color-text);
+}
+
+.ledger-entry__link:active {
+  background: var(--color-brand-soft);
+  text-decoration-thickness: 0.14em;
 }
 
 .status-text {
   display: inline-flex;
   min-width: 0;
   align-items: baseline;
-  gap: 0.35rem;
+  gap: var(--space-1);
   font-weight: 750;
 }
 
@@ -558,154 +728,240 @@ input[aria-invalid="true"] {
   color: var(--color-muted);
 }
 
+.direction {
+  display: flex;
+  min-width: 0;
+  align-items: flex-start;
+  gap: var(--space-2);
+  margin-bottom: 0;
+  font-weight: 800;
+}
+
+.direction__symbol {
+  flex: 0 0 auto;
+}
+
+.direction--lower,
+.direction--higher {
+  color: var(--color-data);
+}
+
+.direction--equal {
+  color: var(--color-text);
+}
+
 .breadcrumb {
-  margin-bottom: 1.5rem;
+  margin-bottom: var(--space-6);
+}
+
+.detail-intro,
+.detail-intro__identity,
+.detail-layout,
+.detail-main,
+.detail-aside {
+  min-width: 0;
 }
 
-.breadcrumb ol {
+.detail-intro {
+  display: grid;
+  gap: var(--space-8);
+  margin-bottom: var(--space-10);
+}
+
+.detail-signature {
   display: flex;
   min-width: 0;
   flex-wrap: wrap;
-  align-items: center;
-  gap: 0.35rem;
-  color: var(--color-muted);
-  font-size: 0.9rem;
+  gap: var(--space-2) var(--space-5);
+  margin-bottom: 0;
 }
 
-.breadcrumb li {
+.detail-signature div {
   min-width: 0;
 }
 
-.breadcrumb li + li::before {
-  padding-right: 0.35rem;
-  content: "/";
+.detail-signature dt,
+.detail-signature dd {
+  display: inline;
 }
 
-.breadcrumb a {
-  display: inline-flex;
-  min-height: 2.75rem;
-  align-items: center;
+.detail-signature dt::after {
+  content: " ";
+}
+
+.detail-signature dd {
+  font-weight: 750;
 }
 
 .current-price {
   display: grid;
   min-width: 0;
-  gap: 0.5rem;
-  margin-bottom: 1.25rem;
-  padding: clamp(1.25rem, 5vw, 2.25rem);
-  border-color: #8fb39e;
-  background: linear-gradient(145deg, var(--color-surface), var(--color-brand-soft));
+  gap: var(--space-2);
+  padding-block: var(--space-5);
+  border-block: 2px solid var(--color-brand-strong);
 }
 
-.current-price h2 {
+.current-price h2,
+.current-price__value,
+.current-price__date,
+.current-price__note {
   margin-bottom: 0;
 }
 
 .current-price__value {
-  margin-bottom: 0;
-  font-size: clamp(2rem, 9vw, 3.5rem);
-  font-weight: 850;
-  letter-spacing: -0.04em;
-  line-height: 1.15;
+  font-family: "Gowun Batang", serif;
+  font-size: clamp(2.25rem, 9vw, 3.75rem);
+  font-weight: 700;
+  letter-spacing: -0.045em;
+  line-height: 1.1;
+  overflow-wrap: normal;
+  white-space: nowrap;
 }
 
-.current-price__date {
-  margin-bottom: 0;
+.current-price__date,
+.current-price__note {
   color: var(--color-muted);
 }
 
+.current-price__note {
+  max-width: 36rem;
+  font-size: 0.88rem;
+}
+
+.identity-panel,
+.provenance,
+.comparison-section {
+  min-width: 0;
+  padding-top: var(--space-5);
+  border-top: 1px solid var(--color-border-strong);
+}
+
 .identity-panel,
-.comparison-section,
 .provenance {
-  margin-top: 1.25rem;
+  margin-top: var(--space-8);
 }
 
 .definition-grid {
   display: grid;
   min-width: 0;
   grid-template-columns: repeat(auto-fit, minmax(min(100%, 11rem), 1fr));
-  gap: 1rem;
+  gap: var(--space-4);
   margin-bottom: 0;
 }
 
 .definition-grid div {
   min-width: 0;
-  padding-top: 0.75rem;
+  padding-top: var(--space-2);
   border-top: 1px solid var(--color-border);
 }
 
-.comparison-section {
+.definition-grid dd {
+  font-weight: 720;
+}
+
+.comparison-ledger,
+.comparison-row,
+.comparison-row__facts,
+.comparison-field {
   min-width: 0;
-  padding-block: 1rem;
+}
+
+.comparison-column-head {
+  display: none;
 }
 
 .comparison-list {
-  display: grid;
   min-width: 0;
-  grid-template-columns: repeat(auto-fit, minmax(min(100%, 16rem), 1fr));
-  gap: 1rem;
+  border-bottom: 1px solid var(--color-border-strong);
 }
 
-.comparison-card {
+.comparison-row {
   min-width: 0;
-  padding: 1.25rem;
-  border: 1px solid var(--color-border);
-  border-radius: var(--radius-medium);
+  border-top: 1px solid var(--color-border);
   background: var(--color-surface);
 }
 
-.comparison-card--unavailable {
-  border-style: dashed;
-  background: var(--color-neutral-soft);
+.comparison-row:first-child {
+  border-top-color: var(--color-border-strong);
 }
 
-.comparison-card__reference {
-  margin-bottom: 0.75rem;
+.comparison-row--unavailable {
   color: var(--color-muted);
 }
 
-.comparison-card__reference strong {
-  color: var(--color-text);
-  font-size: 1.12rem;
+.comparison-row__facts {
+  display: grid;
+  grid-template-columns: repeat(2, minmax(0, 1fr));
+  gap: var(--space-2) var(--space-3);
+  margin-bottom: 0;
+  padding: var(--space-2);
 }
 
-.direction {
-  display: flex;
-  min-width: 0;
-  align-items: flex-start;
-  gap: 0.45rem;
-  font-weight: 800;
+.comparison-field {
+  display: grid;
+  grid-template-columns: auto minmax(0, 1fr);
+  align-items: baseline;
+  gap: var(--space-1);
+  margin-bottom: 0;
+  padding-top: 0;
+  border-top: 0;
 }
 
-.direction__symbol {
-  flex: 0 0 auto;
+.comparison-field--difference,
+.comparison-field--date {
+  grid-column: 1 / -1;
 }
 
-.direction--lower {
-  color: #235783;
+.comparison-field--difference {
+  display: block;
+  padding-top: var(--space-2);
+  border-top: 1px solid var(--color-border);
 }
 
-.direction--higher {
-  color: #8b1e24;
+.comparison-field dd > :last-child {
+  margin-bottom: 0;
 }
 
-.direction--equal {
-  color: var(--color-text);
+.comparison-row__empty {
+  margin-bottom: 0;
+  padding: var(--space-3) var(--space-2);
 }
 
-.comparison-card__date,
-.comparison-card__reason {
-  color: var(--color-muted);
-  font-size: 0.9rem;
+.comparison-meter {
+  width: 100%;
+  max-width: 18rem;
+  min-width: 8rem;
+  height: 1.25rem;
+  margin-top: var(--space-2);
+  overflow: visible;
 }
 
-.comparison-card__date:last-child,
-.comparison-card__reason:last-child {
-  margin-bottom: 0;
+.comparison-meter__rail {
+  stroke: var(--color-border-strong);
+  stroke-width: 1.5;
+}
+
+.comparison-meter__zero {
+  stroke: var(--color-text);
+  stroke-width: 1.5;
+}
+
+.comparison-meter__value,
+.comparison-meter__point {
+  fill: var(--color-data);
+}
+
+.comparison-meter__cap {
+  stroke: var(--color-data);
+  stroke-width: 2;
+}
+
+.comparison-meter--equal .comparison-meter__point {
+  fill: var(--color-text);
 }
 
 .definition-grid--provenance {
-  grid-template-columns: repeat(auto-fit, minmax(min(100%, 14rem), 1fr));
+  grid-template-columns: repeat(auto-fit, minmax(min(100%, 13rem), 1fr));
 }
 
 .metadata-detail {
@@ -715,23 +971,18 @@ input[aria-invalid="true"] {
   font-weight: 500;
 }
 
-.provenance a {
-  display: inline-flex;
-  min-height: 2.75rem;
-  align-items: center;
-}
-
 .provenance__note {
   max-width: var(--measure);
-  margin: 1.5rem 0 0;
-  padding-top: 1rem;
+  margin: var(--space-6) 0 0;
+  padding-top: var(--space-4);
   border-top: 1px solid var(--color-border);
   color: var(--color-muted);
 }
 
 .error-page {
   max-width: 42rem;
-  padding: clamp(1.25rem, 5vw, 3rem);
+  padding-block: var(--space-6);
+  border-block: 1px solid var(--color-border-strong);
 }
 
 .error-page p:not(.eyebrow) {
@@ -739,10 +990,17 @@ input[aria-invalid="true"] {
   color: var(--color-muted);
 }
 
-@media (max-width: 39.999rem) {
-  .breadcrumb li[aria-current="page"] {
-    display: none;
+@media (hover: hover) {
+  .button--primary:hover {
+    background: var(--color-brand);
+  }
+
+  .button--secondary:hover,
+  .segment:hover,
+  .ledger-entry__link:hover {
+    background: var(--color-brand-soft);
   }
+
 }
 
 @media (min-width: 40rem) {
@@ -755,37 +1013,177 @@ input[aria-invalid="true"] {
     min-width: 7rem;
   }
 
-  .result-card__link {
-    grid-template-columns: minmax(12rem, 1fr) minmax(18rem, 1.25fr);
-    align-items: center;
+  .publication-summary {
+    gap: var(--space-6);
   }
 
-  .result-card__action {
+  .ledger-entry {
+    grid-template-columns: repeat(2, minmax(0, 1fr));
+    align-items: start;
+    gap: var(--space-4);
+  }
+
+  .ledger-entry__heading {
     grid-column: 1 / -1;
   }
 
+  .detail-intro {
+    grid-template-columns: minmax(0, 1fr) minmax(18rem, 0.75fr);
+    align-items: stretch;
+    gap: var(--space-6);
+    padding-block: var(--space-5);
+    border-block: 2px solid var(--color-brand-strong);
+  }
+
+  .detail-intro__identity {
+    align-self: center;
+  }
+
   .current-price {
-    grid-template-columns: minmax(0, 1fr) auto;
-    align-items: end;
+    align-content: center;
+    padding-block: 0;
+    padding-left: var(--space-6);
+    border-block: 0;
+    border-left: 1px solid var(--color-border);
   }
 
   .current-price__value {
-    grid-row: 1 / 3;
-    grid-column: 2;
-    text-align: right;
+    font-size: clamp(2.5rem, 6vw, 3.75rem);
+  }
+
+  .comparison-row__facts {
+    gap: var(--space-2) var(--space-4);
   }
 }
 
 @media (min-width: 64rem) {
-  .result-card__link {
-    grid-template-columns: minmax(15rem, 1fr) minmax(25rem, 1.5fr) auto;
+  .ledger-column-head,
+  .ledger-entry {
+    grid-template-columns:
+      minmax(12rem, 1.45fr)
+      minmax(8rem, 0.72fr)
+      minmax(14rem, 1.2fr)
+      minmax(8rem, 0.72fr)
+      minmax(6rem, 0.46fr);
+    gap: var(--space-4);
+  }
+
+  .ledger-column-head {
+    display: grid;
+    padding: var(--space-2) var(--space-3);
+    border-bottom: 1px solid var(--color-border-strong);
+    color: var(--color-muted);
+    font-size: 0.78rem;
+    font-weight: 800;
   }
 
-  .result-card__action {
+  .ledger-entry {
+    align-items: center;
+    padding: var(--space-3);
+  }
+
+  .ledger-entry__heading,
+  .ledger-entry__action {
+    grid-column: auto;
+  }
+
+  .ledger-entry__heading {
+    display: block;
+  }
+
+  .ledger-fact {
+    padding-top: 0;
+    border-top: 0;
+  }
+
+  .ledger-fact--price,
+  .ledger-fact--comparison,
+  .ledger-fact--date {
     grid-row: auto;
     grid-column: auto;
+  }
+
+  .ledger-fact dt {
+    position: absolute;
+    width: 1px;
+    height: 1px;
+    padding: 0;
+    margin: -1px;
+    overflow: hidden;
+    clip: rect(0, 0, 0, 0);
+    white-space: nowrap;
+    border: 0;
+  }
+
+  .ledger-entry__action {
+    display: inline-flex;
     justify-self: end;
   }
+
+  .detail-layout {
+    display: block;
+  }
+
+  .detail-aside {
+    display: grid;
+    grid-template-columns: minmax(0, 0.8fr) minmax(0, 1.2fr);
+    gap: var(--space-8);
+    margin-top: var(--space-10);
+    padding-top: var(--space-5);
+    border-top: 1px solid var(--color-border-strong);
+  }
+
+  .detail-aside .identity-panel,
+  .detail-aside .provenance {
+    margin-top: 0;
+    padding-top: 0;
+    border-top: 0;
+  }
+
+  .comparison-column-head,
+  .comparison-row__facts {
+    grid-template-columns:
+      minmax(5.5rem, 0.6fr)
+      minmax(7rem, 0.8fr)
+      minmax(15rem, 1.6fr)
+      minmax(8rem, 0.9fr);
+    gap: var(--space-3);
+  }
+
+  .comparison-column-head {
+    display: grid;
+    padding: var(--space-2);
+    border-top: 2px solid var(--color-brand-strong);
+    border-bottom: 1px solid var(--color-border-strong);
+    color: var(--color-muted);
+    font-size: 0.78rem;
+    font-weight: 800;
+  }
+
+  .comparison-row__facts {
+    align-items: start;
+    padding: var(--space-3) var(--space-2);
+  }
+
+  .comparison-field,
+  .comparison-field:nth-child(2) {
+    display: block;
+    grid-column: auto;
+    padding-top: 0;
+    border-top: 0;
+  }
+
+  .comparison-field dt {
+    position: absolute;
+    width: 1px;
+    height: 1px;
+    padding: 0;
+    margin: -1px;
+    overflow: hidden;
+    clip: rect(0, 0, 0, 0);
+    white-space: nowrap;
+    border: 0;
+  }
 }
 
 @media (prefers-reduced-motion: reduce) {
@@ -794,15 +1192,38 @@ input[aria-invalid="true"] {
   *::after {
     scroll-behavior: auto !important;
     transition-duration: 0.01ms !important;
+    animation-duration: 0.01ms !important;
+    animation-iteration-count: 1 !important;
   }
 }
 
 @media (forced-colors: active) {
   .button,
-  .chip,
+  .segment,
   .state-notice,
-  .result-card,
-  .comparison-card {
+  .ledger-row,
+  .comparison-row,
+  .search-panel,
+  .current-price,
+  .identity-panel,
+  .provenance,
+  .error-page {
     border-color: CanvasText;
   }
+
+  .brand-mark,
+  .comparison-meter {
+    forced-color-adjust: auto;
+  }
+
+  .comparison-meter__rail,
+  .comparison-meter__zero,
+  .comparison-meter__cap {
+    stroke: CanvasText;
+  }
+
+  .comparison-meter__value,
+  .comparison-meter__point {
+    fill: CanvasText;
+  }
 }
diff --git a/grocery/static/grocery/brand-mark.svg b/grocery/static/grocery/brand-mark.svg
new file mode 100644
index 0000000..bb66840
--- /dev/null
+++ b/grocery/static/grocery/brand-mark.svg
@@ -0,0 +1,16 @@
+<svg
+  xmlns="http://www.w3.org/2000/svg"
+  viewBox="0 0 48 48"
+  fill="none"
+  stroke="#0d442a"
+  stroke-width="2.5"
+  stroke-linecap="round"
+  stroke-linejoin="round"
+  aria-hidden="true"
+>
+  <path d="M4.5 14.5c6.3-2.2 12.8-.9 19.5 3.5v25c-6.7-4.4-13.2-5.7-19.5-3.5Z"/>
+  <path d="M43.5 14.5c-6.3-2.2-12.8-.9-19.5 3.5v25c6.7-4.4 13.2-5.7 19.5-3.5Z"/>
+  <path d="M24 18v25"/>
+  <path d="M24 17c.2-5.8 3.4-9.5 9-11.5-.1 5.8-3.3 9.5-9 11.5Z" fill="#0d442a"/>
+  <path d="M23.8 17c-5.6-1-9.1-4.2-10-9.4 5.6.8 9 4 10 9.4Z" fill="#0d442a"/>
+</svg>
diff --git a/grocery/static/grocery/favicon.svg b/grocery/static/grocery/favicon.svg
index 89e1ce9..4266ecc 100644
--- a/grocery/static/grocery/favicon.svg
+++ b/grocery/static/grocery/favicon.svg
@@ -1,5 +1,17 @@
-<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64" role="img" aria-label="농산물 조사값">
-  <rect width="64" height="64" rx="14" fill="#0d442a"/>
-  <path d="M32 51c-10 0-18-8-18-18 0-9 6-16 14-18 0 6 3 10 8 12 5 2 10 0 14-3 1 3 2 6 2 9 0 10-9 18-20 18Z" fill="#e8f2ec"/>
-  <path d="M32 13c5 1 9 5 10 10-6 0-10-4-10-10Z" fill="#8fc7a5"/>
+<svg
+  xmlns="http://www.w3.org/2000/svg"
+  viewBox="0 0 64 64"
+  fill="none"
+  stroke="#0d442a"
+  stroke-width="4"
+  stroke-linecap="round"
+  stroke-linejoin="round"
+  role="img"
+  aria-label="초록장부"
+>
+  <path d="M6 22c8.5-3 17-1.2 26 4.8V58c-9-6-17.5-7.8-26-4.8Z"/>
+  <path d="M58 22c-8.5-3-17-1.2-26 4.8V58c9-6 17.5-7.8 26-4.8Z"/>
+  <path d="M32 27v31"/>
+  <path d="M32 25.5C32.2 17.8 36.5 12.8 44 10c-.2 7.8-4.5 12.8-12 15.5Z" fill="#0d442a"/>
+  <path d="M31.8 25.5c-7.5-1.3-12.2-5.6-13.5-12.6 7.5 1.1 12.1 5.4 13.5 12.6Z" fill="#0d442a"/>
 </svg>
diff --git a/grocery/templates/grocery/base.html b/grocery/templates/grocery/base.html
index 6b9be7d..4351780 100644
--- a/grocery/templates/grocery/base.html
+++ b/grocery/templates/grocery/base.html
@@ -7,9 +7,9 @@
     <meta name="color-scheme" content="light">
     <meta
       name="description"
-      content="KAMIS가 제공한 채소류·과일류 소매 조사 평균과 비교 제공값을 확인합니다."
+      content="KAMIS가 제공한 채소·과일 소매 조사 평균과 비교값을 확인합니다."
     >
-    <title>{% block title %}농산물 조사값 살펴보기{% endblock %}</title>
+    <title>{% block title %}초록장부 | 채소·과일 소매 조사값{% endblock %}</title>
     <link rel="icon" href="{% static 'grocery/favicon.svg' %}" type="image/svg+xml">
     <link rel="stylesheet" href="{% static 'grocery/app.css' %}">
   </head>
@@ -18,27 +18,29 @@
 
     <header class="site-header">
       <div class="page-shell site-header__inner">
-        <a class="brand" href="{{ home_url|default:'/' }}" aria-label="농산물 조사값 살펴보기 홈">
-          <span class="brand__name">농산물 조사값 살펴보기</span>
-          <span class="brand__description">KAMIS가 제공한 소매 조사 자료</span>
+        <a class="brand" href="{{ home_url|default:'/' }}" aria-label="초록장부 홈">
+          <img
+            class="brand-mark"
+            src="{% static 'grocery/brand-mark.svg' %}"
+            width="44"
+            height="44"
+            alt=""
+          >
+          <span class="brand-copy">
+            <span class="brand__name">초록장부</span>
+            <span class="brand__description">채소·과일 소매 조사값</span>
+          </span>
         </a>
       </div>
     </header>
 
     <main id="main-content" class="page-shell page-main" tabindex="-1">
-      {% if qa_preview %}
-        <p class="qa-notice" role="note">로컬 화면 상태 검수용 미리보기</p>
-      {% endif %}
       {% block content %}{% endblock %}
     </main>
 
     <footer class="site-footer">
       <div class="page-shell site-footer__inner">
-        <p>
-          표시값은 KAMIS가 제공한 소매 조사 평균이며, 개별 판매처의 판매값을 나타내지
-          않습니다.
-        </p>
-        <p>검토 후 공개된 자료만 표시하며 공개 화면에서 외부 source를 직접 호출하지 않습니다.</p>
+        <p>표시값은 KAMIS 소매 조사 평균입니다. 개별 판매처의 실제 판매 금액과 다를 수 있습니다.</p>
       </div>
     </footer>
   </body>
diff --git a/grocery/tests/test_accessibility_contrast.py b/grocery/tests/test_accessibility_contrast.py
index d675a26..3017d3f 100644
--- a/grocery/tests/test_accessibility_contrast.py
+++ b/grocery/tests/test_accessibility_contrast.py
@@ -20,27 +20,52 @@ def _contrast(foreground: str, background: str) -> float:
     return (lighter + 0.05) / (darker + 0.05)
 
 
-def test_rendered_text_palette_meets_wcag_aa_including_gradient_extremes() -> None:
+def _colors() -> dict[str, str]:
     css = Path(settings.BASE_DIR, "grocery", "static", "grocery", "app.css").read_text(
         encoding="utf-8"
     )
-    colors = {match.group("name"): match.group("value") for match in _CUSTOM_PROPERTY.finditer(css)}
+    return {match.group("name"): match.group("value") for match in _CUSTOM_PROPERTY.finditer(css)}
+
+
+def test_rendered_text_palette_meets_wcag_aa_across_ledger_and_state_surfaces() -> None:
+    colors = _colors()
     pairs = (
         (colors["color-text"], colors["color-canvas"]),
         (colors["color-text"], colors["color-surface"]),
+        (colors["color-text"], colors["color-surface-muted"]),
+        (colors["color-text"], colors["color-neutral-soft"]),
         (colors["color-muted"], colors["color-canvas"]),
         (colors["color-muted"], colors["color-surface"]),
         (colors["color-brand"], colors["color-surface"]),
         (colors["color-brand-strong"], colors["color-surface"]),
+        (colors["color-brand-strong"], colors["color-brand-soft"]),
         (colors["color-info"], colors["color-info-soft"]),
         (colors["color-warning"], colors["color-warning-soft"]),
         (colors["color-error"], colors["color-error-soft"]),
         (colors["color-text"], colors["color-brand-soft"]),
-        (colors["color-brand-strong"], colors["color-brand-soft"]),
-        ("#235783", colors["color-surface"]),
-        ("#8b1e24", colors["color-surface"]),
-        ("#ffffff", colors["color-brand-strong"]),
-        ("#68141a", colors["color-error-soft"]),
+        (colors["color-lower"], colors["color-surface"]),
+        (colors["color-higher"], colors["color-surface"]),
+        (colors["color-on-brand"], colors["color-brand"]),
+        (colors["color-on-brand"], colors["color-brand-strong"]),
     )
 
     assert min(_contrast(foreground, background) for foreground, background in pairs) >= 4.5
+
+
+def test_focus_and_interactive_boundaries_meet_non_text_contrast() -> None:
+    colors = _colors()
+    pairs = (
+        (colors["color-focus"], colors["color-canvas"]),
+        (colors["color-focus"], colors["color-surface"]),
+        (colors["color-focus"], colors["color-brand-soft"]),
+        (colors["color-border-strong"], colors["color-canvas"]),
+        (colors["color-border-strong"], colors["color-surface"]),
+    )
+
+    assert min(_contrast(foreground, background) for foreground, background in pairs) >= 3
+
+
+def test_price_direction_tokens_use_one_neutral_data_color() -> None:
+    colors = _colors()
+
+    assert colors["color-lower"] == colors["color-higher"] == colors["color-data"]
diff --git a/grocery/tests/test_static_delivery.py b/grocery/tests/test_static_delivery.py
index 5ff5949..0ed6d52 100644
--- a/grocery/tests/test_static_delivery.py
+++ b/grocery/tests/test_static_delivery.py
@@ -1,4 +1,9 @@
+import hashlib
+from pathlib import Path
+from xml.etree import ElementTree
+
 from django.conf import settings
+from django.contrib.staticfiles import finders
 from django.test import SimpleTestCase
 
 from config.settings import staticfiles_storage_backend
@@ -24,3 +29,71 @@ class StaticDeliverySettingsTests(SimpleTestCase):
             staticfiles_storage_backend(debug=True),
             "django.contrib.staticfiles.storage.StaticFilesStorage",
         )
+
+    def test_frontend_static_assets_are_local_and_discoverable(self) -> None:
+        for asset in (
+            "grocery/app.css",
+            "grocery/brand-mark.svg",
+            "grocery/favicon.svg",
+            "grocery/fonts/gowun-batang-bold.woff2",
+        ):
+            with self.subTest(asset=asset):
+                self.assertIsNotNone(finders.find(asset))
+
+        css = Path(settings.BASE_DIR, "grocery", "static", "grocery", "app.css").read_text(
+            encoding="utf-8"
+        )
+        self.assertIn('url("fonts/gowun-batang-bold.woff2")', css)
+        self.assertNotIn("@import", css)
+        self.assertNotIn("http://", css)
+        self.assertNotIn("https://", css)
+
+    def test_self_hosted_heading_font_matches_pinned_upstream_provenance(self) -> None:
+        base_dir = Path(settings.BASE_DIR)
+        font_path = base_dir / "grocery/static/grocery/fonts/gowun-batang-bold.woff2"
+        license_path = base_dir / "LICENSES/GowunBatang-OFL-1.1.txt"
+        notices_path = base_dir / "THIRD_PARTY_NOTICES.md"
+
+        self.assertEqual(
+            hashlib.sha256(font_path.read_bytes()).hexdigest(),
+            "7f3c6eff348d1e8034bbd9b4cf177e887c4a8a58b59035cfee0b1ed464d54a70",
+        )
+        license_text = license_path.read_text(encoding="utf-8")
+        self.assertIn(
+            "Copyright 2021 The Gowun Batang Project Authors",
+            license_text,
+        )
+        self.assertIn("SIL OPEN FONT LICENSE Version 1.1", license_text)
+        notices = notices_path.read_text(encoding="utf-8")
+        self.assertIn("4e73f5a9a004927220354f4b68a4c720da538147", notices)
+        self.assertIn("LICENSES/GowunBatang-OFL-1.1.txt", notices)
+
+    def test_public_frontend_contains_no_raster_photo_assets(self) -> None:
+        static_root = Path(settings.BASE_DIR, "grocery", "static", "grocery")
+        raster_suffixes = {".avif", ".gif", ".jpeg", ".jpg", ".png", ".webp"}
+
+        raster_assets = [
+            path.relative_to(static_root).as_posix()
+            for path in static_root.rglob("*")
+            if path.is_file() and path.suffix.lower() in raster_suffixes
+        ]
+
+        self.assertEqual(raster_assets, [])
+
+    def test_svg_assets_have_no_script_foreign_object_or_external_reference(self) -> None:
+        for asset_name in ("brand-mark.svg", "favicon.svg"):
+            path = Path(settings.BASE_DIR, "grocery", "static", "grocery", asset_name)
+            # This is a repository-owned static fixture, not untrusted XML input.
+            root = ElementTree.parse(path).getroot()  # noqa: S314
+            local_names = {element.tag.rsplit("}", 1)[-1] for element in root.iter()}
+
+            with self.subTest(asset=asset_name):
+                self.assertEqual(root.tag.rsplit("}", 1)[-1], "svg")
+                self.assertIn("viewBox", root.attrib)
+                self.assertNotIn("script", local_names)
+                self.assertNotIn("foreignObject", local_names)
+                for element in root.iter():
+                    for name, value in element.attrib.items():
+                        self.assertFalse(name.lower().startswith("on"))
+                        if name.rsplit("}", 1)[-1] == "href":
+                            self.assertTrue(value.startswith("#"), value)


