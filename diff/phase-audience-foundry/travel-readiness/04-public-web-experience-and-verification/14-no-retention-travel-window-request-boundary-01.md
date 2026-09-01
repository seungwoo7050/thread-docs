# 저장하지 않는 여행 시간창 요청 경계

## `feat(web): add the no-retention trip form`

diff --git a/public_web/forms.py b/public_web/forms.py
new file mode 100644
index 0000000..8171fb3
--- /dev/null
+++ b/public_web/forms.py
@@ -0,0 +1,82 @@
+from django import forms
+
+
+class TripForm(forms.Form):
+    destination = forms.ChoiceField(
+        label="목적지",
+        choices=(("JP", "일본"),),
+        error_messages={
+            "required": "목적지를 선택해 주세요.",
+            "invalid_choice": "목적지는 일본만 선택할 수 있습니다.",
+        },
+        widget=forms.Select(
+            attrs={
+                "autocomplete": "off",
+                "aria-describedby": "id_destination_hint",
+            }
+        ),
+    )
+    departure_date = forms.DateField(
+        label="출국일",
+        input_formats=("%Y-%m-%d",),
+        widget=forms.DateInput(
+            format="%Y-%m-%d",
+            attrs={
+                "type": "date",
+                "autocomplete": "off",
+                "aria-describedby": "id_departure_date_hint",
+            },
+        ),
+        error_messages={
+            "required": "출국일을 입력해 주세요.",
+            "invalid": "출국일을 날짜 형식으로 입력해 주세요.",
+        },
+    )
+    return_date = forms.DateField(
+        label="귀국일",
+        input_formats=("%Y-%m-%d",),
+        widget=forms.DateInput(
+            format="%Y-%m-%d",
+            attrs={
+                "type": "date",
+                "autocomplete": "off",
+                "aria-describedby": "id_return_date_hint",
+            },
+        ),
+        error_messages={
+            "required": "귀국일을 입력해 주세요.",
+            "invalid": "귀국일을 날짜 형식으로 입력해 주세요.",
+        },
+    )
+
+    def clean(self):
+        cleaned = super().clean()
+        departure = cleaned.get("departure_date")
+        return_date = cleaned.get("return_date")
+        if (
+            departure is not None
+            and return_date is not None
+            and return_date < departure
+        ):
+            self.add_error(
+                "return_date",
+                forms.ValidationError(
+                    "귀국일은 출국일과 같거나 그 이후여야 합니다.",
+                    code="date_order",
+                ),
+            )
+        return cleaned
+
+    def mark_errors_for_accessibility(self) -> None:
+        """Link bound field errors without copying values outside the form."""
+
+        for name in self.errors:
+            if name not in self.fields:
+                continue
+            widget = self.fields[name].widget
+            widget.attrs["aria-invalid"] = "true"
+            current = widget.attrs.get("aria-describedby", "").split()
+            error_id = f"id_{name}_error"
+            if error_id not in current:
+                current.append(error_id)
+            widget.attrs["aria-describedby"] = " ".join(current)
diff --git a/public_web/middleware.py b/public_web/middleware.py
new file mode 100644
index 0000000..132b31a
--- /dev/null
+++ b/public_web/middleware.py
@@ -0,0 +1,63 @@
+from django.conf import settings
+from django.http import HttpRequest, HttpResponse
+
+
+_PUBLIC_CANONICAL_PATHS = {
+    "/": "/",
+    "/results": "/results/",
+    "/results/": "/results/",
+}
+_CSP = (
+    "default-src 'self'; base-uri 'none'; form-action 'self'; "
+    "frame-ancestors 'none'; object-src 'none'"
+)
+
+
+def _protect(
+    response: HttpResponse,
+    request: HttpRequest,
+) -> HttpResponse:
+    response.headers["Cache-Control"] = "no-store"
+    response.headers["Pragma"] = "no-cache"
+    response.headers["Content-Security-Policy"] = _CSP
+    response.headers["Referrer-Policy"] = "no-referrer"
+    response.headers["X-Content-Type-Options"] = "nosniff"
+    response.headers["X-Frame-Options"] = "DENY"
+    response.headers["Cross-Origin-Opener-Policy"] = "same-origin"
+    if request.is_secure() and settings.SECURE_HSTS_SECONDS:
+        hsts = f"max-age={settings.SECURE_HSTS_SECONDS}"
+        if settings.SECURE_HSTS_INCLUDE_SUBDOMAINS:
+            hsts += "; includeSubDomains"
+        if settings.SECURE_HSTS_PRELOAD:
+            hsts += "; preload"
+        response.headers["Strict-Transport-Security"] = hsts
+    return response
+
+
+class PublicPrivacyMiddleware:
+    """Strip trip queries before framework redirects and close public errors."""
+
+    def __init__(self, get_response):
+        self.get_response = get_response
+
+    def __call__(self, request: HttpRequest) -> HttpResponse:
+        canonical_path = _PUBLIC_CANONICAL_PATHS.get(request.path_info)
+        if canonical_path is not None and request.META.get("QUERY_STRING"):
+            return _protect(
+                HttpResponse(
+                    status=303,
+                    headers={"Location": canonical_path},
+                ),
+                request,
+            )
+
+        response = self.get_response(request)
+        if canonical_path is None:
+            return response
+        if response.status_code >= 500:
+            response = HttpResponse(
+                "일시적으로 요청을 처리할 수 없습니다.\n",
+                status=response.status_code,
+                content_type="text/plain; charset=utf-8",
+            )
+        return _protect(response, request)
diff --git a/public_web/templates/public_web/index.html b/public_web/templates/public_web/index.html
new file mode 100644
index 0000000..b504f32
--- /dev/null
+++ b/public_web/templates/public_web/index.html
@@ -0,0 +1,83 @@
+<!doctype html>
+<html lang="ko">
+<head>
+  <meta charset="utf-8">
+  <meta name="viewport" content="width=device-width, initial-scale=1">
+  <title>여행준비 — 일본 정보 확인</title>
+</head>
+<body>
+  <header>
+    <nav aria-label="주요 메뉴">
+      <a href="{% url 'public_web:index' %}" aria-current="page">여행준비</a>
+      <a href="{% url 'public_web:results' %}">게시 정보 보기</a>
+    </nav>
+  </header>
+  <main id="main-content">
+    <h1>일본 여행 정보 확인</h1>
+    <p>
+      대한민국 일반여권 여행자를 위한 공식 source 사실을 확인합니다.
+      입력은 이 요청 안에서만 검증하며 저장하지 않습니다.
+    </p>
+
+    {% if form.errors %}
+      <section role="alert" aria-labelledby="form-error-heading" tabindex="-1">
+        <h2 id="form-error-heading">입력 내용을 확인해 주세요</h2>
+        <ul>
+          {% for field in form %}
+            {% for error in field.errors %}
+              <li><a href="#{{ field.id_for_label }}">{{ field.label }}: {{ error }}</a></li>
+            {% endfor %}
+          {% endfor %}
+          {% for error in form.non_field_errors %}
+            <li>{{ error }}</li>
+          {% endfor %}
+        </ul>
+      </section>
+    {% endif %}
+
+    <form method="post" action="{% url 'public_web:index' %}" novalidate>
+      {% csrf_token %}
+      <fieldset>
+        <legend>여행 범위</legend>
+
+        <div>
+          <label for="{{ form.destination.id_for_label }}">{{ form.destination.label }}</label>
+          {{ form.destination }}
+          <p id="id_destination_hint">Phase 0 공개 범위는 일본으로 고정되어 있습니다.</p>
+          {% if form.destination.errors %}
+            <ul id="id_destination_error">
+              {% for error in form.destination.errors %}<li>{{ error }}</li>{% endfor %}
+            </ul>
+          {% endif %}
+        </div>
+
+        <div>
+          <label for="{{ form.departure_date.id_for_label }}">{{ form.departure_date.label }}</label>
+          {{ form.departure_date }}
+          <p id="id_departure_date_hint">날짜 적용성은 결과에서 계산하지 않습니다.</p>
+          {% if form.departure_date.errors %}
+            <ul id="id_departure_date_error">
+              {% for error in form.departure_date.errors %}<li>{{ error }}</li>{% endfor %}
+            </ul>
+          {% endif %}
+        </div>
+
+        <div>
+          <label for="{{ form.return_date.id_for_label }}">{{ form.return_date.label }}</label>
+          {{ form.return_date }}
+          <p id="id_return_date_hint">출국일과 같거나 이후 날짜를 입력해 주세요.</p>
+          {% if form.return_date.errors %}
+            <ul id="id_return_date_error">
+              {% for error in form.return_date.errors %}<li>{{ error }}</li>{% endfor %}
+            </ul>
+          {% endif %}
+        </div>
+      </fieldset>
+      <button type="submit">게시 정보 확인</button>
+    </form>
+  </main>
+  <footer>
+    <p>여행 목적과 날짜별 적용 여부는 공식 기관에서 다시 확인해 주세요.</p>
+  </footer>
+</body>
+</html>
diff --git a/public_web/templates/public_web/results.html b/public_web/templates/public_web/results.html
index b48a972..4a90ddd 100644
--- a/public_web/templates/public_web/results.html
+++ b/public_web/templates/public_web/results.html
@@ -8,7 +8,7 @@
 <body>
   <header>
     <nav aria-label="주요 메뉴">
-      <a href="/">처음으로</a>
+      <a href="{% url 'public_web:index' %}">다시 입력</a>
       <a href="{% url 'public_web:results' %}" aria-current="page">게시 정보</a>
     </nav>
   </header>
diff --git a/public_web/tests/test_privacy_form.py b/public_web/tests/test_privacy_form.py
new file mode 100644
index 0000000..87b9adc
--- /dev/null
+++ b/public_web/tests/test_privacy_form.py
@@ -0,0 +1,182 @@
+from unittest.mock import patch
+
+from django.core.cache import cache
+from django.test import Client, SimpleTestCase, override_settings
+from django.urls import reverse
+
+
+class TripFormPrivacyTests(SimpleTestCase):
+    def valid_data(self, **overrides):
+        data = {
+            "destination": "JP",
+            "departure_date": "2026-09-01",
+            "return_date": "2026-09-01",
+        }
+        data.update(overrides)
+        return data
+
+    def test_get_is_unbound_and_query_input_is_removed(self):
+        response = self.client.get(reverse("public_web:index"))
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(response, "일본 여행 정보 확인")
+        self.assertNotIn("sessionid", response.cookies)
+
+        canonical = self.client.get(
+            reverse("public_web:index"),
+            {"departure_date": "private-marker"},
+        )
+        self.assertEqual(canonical.status_code, 303)
+        self.assertEqual(canonical.headers["Location"], "/")
+        self.assertNotContains(
+            canonical,
+            "private-marker",
+            status_code=303,
+        )
+        self.assertEqual(canonical.headers["Cache-Control"], "no-store")
+
+    @override_settings(SECURE_SSL_REDIRECT=True)
+    def test_query_is_removed_before_https_and_slash_redirects(self):
+        root = self.client.get(
+            "/",
+            {"departure_date": "private-http-marker"},
+            secure=False,
+        )
+        self.assertEqual(root.status_code, 303)
+        self.assertEqual(root.headers["Location"], "/")
+        self.assertNotIn("private-http-marker", root.headers["Location"])
+        self.assertEqual(root.headers["Referrer-Policy"], "no-referrer")
+        self.assertEqual(root.headers["X-Content-Type-Options"], "nosniff")
+
+        missing_slash = self.client.get(
+            "/results",
+            {"return_date": "private-slash-marker"},
+            secure=False,
+        )
+        self.assertEqual(missing_slash.status_code, 303)
+        self.assertEqual(missing_slash.headers["Location"], "/results/")
+        self.assertNotIn(
+            "private-slash-marker",
+            missing_slash.headers["Location"],
+        )
+
+        https_redirect = self.client.get("/", secure=False)
+        self.assertEqual(https_redirect.status_code, 301)
+        self.assertNotIn("?", https_redirect.headers["Location"])
+        self.assertEqual(https_redirect.headers["Cache-Control"], "no-store")
+
+    @patch.object(cache, "set")
+    @patch("django.contrib.sessions.backends.db.SessionStore.save")
+    def test_valid_post_is_queryless_no_retention_303(
+        self,
+        session_save,
+        cache_set,
+    ):
+        response = self.client.post(
+            reverse("public_web:index"),
+            self.valid_data(),
+        )
+
+        self.assertEqual(response.status_code, 303)
+        self.assertEqual(response.headers["Location"], "/results/")
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertNotContains(
+            response,
+            "2026-09-01",
+            status_code=303,
+        )
+        self.assertNotIn("sessionid", response.cookies)
+        self.assertFalse(response.wsgi_request.session.modified)
+        session_save.assert_not_called()
+        cache_set.assert_not_called()
+
+    def test_invalid_values_are_corrected_in_the_same_request(self):
+        response = self.client.post(
+            reverse("public_web:index"),
+            self.valid_data(
+                departure_date="2026-09-10",
+                return_date="2026-09-09",
+            ),
+        )
+
+        self.assertEqual(response.status_code, 200)
+        self.assertContains(
+            response,
+            "귀국일은 출국일과 같거나 그 이후여야 합니다.",
+        )
+        self.assertContains(response, 'aria-invalid="true"')
+        self.assertContains(response, "2026-09-10")
+        self.assertContains(response, "2026-09-09")
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        self.assertNotIn("sessionid", response.cookies)
+        self.assertFalse(response.wsgi_request.session.modified)
+
+    def test_invalid_destination_and_date_shapes_do_not_redirect(self):
+        for data, expected in (
+            (self.valid_data(destination="ZZ"), "일본만 선택"),
+            (self.valid_data(departure_date="not-a-date"), "날짜 형식"),
+            (self.valid_data(return_date=""), "귀국일을 입력"),
+        ):
+            with self.subTest(data=data):
+                response = self.client.post(reverse("public_web:index"), data)
+                self.assertEqual(response.status_code, 200)
+                self.assertContains(response, expected)
+
+    def test_csrf_is_required_and_valid_token_keeps_fixed_redirect(self):
+        client = Client(enforce_csrf_checks=True)
+        rejected = client.post(
+            reverse("public_web:index"),
+            self.valid_data(),
+        )
+        self.assertEqual(rejected.status_code, 403)
+        self.assertEqual(rejected.headers["Cache-Control"], "no-store")
+
+        form_page = client.get(reverse("public_web:index"))
+        token = form_page.cookies["csrftoken"].value
+        accepted = client.post(
+            reverse("public_web:index"),
+            self.valid_data(csrfmiddlewaretoken=token),
+        )
+        self.assertEqual(accepted.status_code, 303)
+        self.assertEqual(accepted.headers["Location"], "/results/")
+        self.assertNotIn("sessionid", accepted.cookies)
+
+    def test_results_query_is_removed_and_post_is_not_supported(self):
+        query = self.client.get(
+            reverse("public_web:results"),
+            {"return_date": "private-marker"},
+        )
+        self.assertEqual(query.status_code, 303)
+        self.assertEqual(query.headers["Location"], "/results/")
+        self.assertNotContains(
+            query,
+            "private-marker",
+            status_code=303,
+        )
+
+        post = self.client.post(reverse("public_web:results"), self.valid_data())
+        self.assertEqual(post.status_code, 405)
+        self.assertEqual(post.headers["Cache-Control"], "no-store")
+
+    @override_settings(DEBUG=True)
+    def test_debug_500_never_renders_bound_trip_values_or_exception(self):
+        marker = "private-debug-trip-marker"
+        client = Client()
+        client.raise_request_exception = False
+        with patch(
+            "public_web.views.render",
+            side_effect=RuntimeError("private-exception-marker"),
+        ):
+            response = client.post(
+                reverse("public_web:index"),
+                self.valid_data(
+                    departure_date=marker,
+                    return_date="invalid-return-date",
+                ),
+            )
+
+        self.assertEqual(response.status_code, 500)
+        self.assertEqual(response.headers["Cache-Control"], "no-store")
+        body = response.content.decode("utf-8")
+        self.assertNotIn(marker, body)
+        self.assertNotIn("private-exception-marker", body)
+        self.assertIn("요청을 처리할 수 없습니다", body)
diff --git a/public_web/urls.py b/public_web/urls.py
index 5223dea..8822806 100644
--- a/public_web/urls.py
+++ b/public_web/urls.py
@@ -1,10 +1,11 @@
 from django.urls import path
 
-from . import results
+from . import results, views
 
 
 app_name = "public_web"
 
 urlpatterns = [
+    path("", views.index, name="index"),
     path("results/", results.results, name="results"),
 ]
diff --git a/public_web/views.py b/public_web/views.py
new file mode 100644
index 0000000..b328c3e
--- /dev/null
+++ b/public_web/views.py
@@ -0,0 +1,42 @@
+from django.http import HttpRequest, HttpResponse
+from django.shortcuts import render
+from django.urls import reverse
+from django.views.decorators.debug import sensitive_post_parameters
+from django.views.decorators.http import require_http_methods
+
+from .forms import TripForm
+
+
+def _fixed_redirect(route_name: str) -> HttpResponse:
+    return HttpResponse(
+        status=303,
+        headers={
+            "Location": reverse(route_name),
+            "Cache-Control": "no-store",
+        },
+    )
+
+
+@sensitive_post_parameters(
+    "destination",
+    "departure_date",
+    "return_date",
+)
+@require_http_methods(("GET", "POST"))
+def index(request: HttpRequest) -> HttpResponse:
+    if request.method == "GET" and request.GET:
+        return _fixed_redirect("public_web:index")
+    if request.method == "POST":
+        form = TripForm(request.POST)
+        if form.is_valid():
+            return _fixed_redirect("public_web:results")
+        form.mark_errors_for_accessibility()
+    else:
+        form = TripForm()
+    response = render(
+        request,
+        "public_web/index.html",
+        {"form": form},
+    )
+    response.headers["Cache-Control"] = "no-store"
+    return response
diff --git a/travel_readiness/settings.py b/travel_readiness/settings.py
index dc876ab..8869428 100644
--- a/travel_readiness/settings.py
+++ b/travel_readiness/settings.py
@@ -57,6 +57,7 @@ AUTH_PASSWORD_VALIDATORS = [
 ]
 
 MIDDLEWARE = [
+    "public_web.middleware.PublicPrivacyMiddleware",
     "django.middleware.security.SecurityMiddleware",
     "django.contrib.sessions.middleware.SessionMiddleware",
     "django.middleware.common.CommonMiddleware",


## `fix(browser): redact trip inputs from acceptance evidence`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 0e21672..feeb876 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -1774,6 +1774,14 @@ def loading_start_javascript(*, origin: str, check: str) -> str:
     return form?.getAttribute('aria-busy') === 'true' && button?.getAttribute('aria-disabled') === 'true' && button.textContent.includes('제출 중') && status?.textContent.includes('불러오는 중') && status.getAttribute('role') === 'status' && status.getAttribute('aria-live') === 'polite';
   }});
   if (!loading) fail('loading-semantics');
+  const inputsCleared = await page.evaluate(() => {{
+    const departure = document.querySelector('[name="departure_date"]');
+    const returning = document.querySelector('[name="return_date"]');
+    if (!(departure instanceof HTMLInputElement) || !(returning instanceof HTMLInputElement)) return false;
+    departure.value = ''; returning.value = '';
+    return departure.value === '' && returning.value === '';
+  }});
+  if (!inputsCleared) fail('loading-input-redaction');
   {_dom_privacy_source(origin=origin, path='/')}
   {_dynamic_layout_source()}
   return {_marker(check)};
@@ -1938,6 +1946,14 @@ def validation_javascript(*, origin: str, check: str) -> str:
   const field = page.getByLabel('귀국일', {{ exact: true }});
   if (await field.getAttribute('aria-invalid') !== 'true' || !(await field.getAttribute('aria-describedby') || '').includes('id_return_date_error')) fail('validation-description');
   if (page.url() !== {json.dumps(origin + '/')}) fail('validation-location');
+  const inputsCleared = await page.evaluate(() => {{
+    const departure = document.querySelector('[name="departure_date"]');
+    const returning = document.querySelector('[name="return_date"]');
+    if (!(departure instanceof HTMLInputElement) || !(returning instanceof HTMLInputElement)) return false;
+    departure.value = ''; returning.value = '';
+    return departure.value === '' && returning.value === '' && document.activeElement?.matches('[data-error-summary]');
+  }});
+  if (!inputsCleared) fail('validation-input-redaction');
   {_dom_privacy_source(origin=origin, path='/')}
   {_static_asset_integrity_source(origin)}
   {_dynamic_layout_source()}
@@ -1952,6 +1968,7 @@ def correction_javascript(*, origin: str, check: str) -> str:
   if (!await page.evaluate(() => document.activeElement?.matches('[data-error-summary]'))) fail('error-focus-start');
   await page.keyboard.press('Tab'); await page.keyboard.press('Enter');
   if (!await page.evaluate(() => document.activeElement?.id === 'id_return_date')) fail('error-link-target');
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
   await page.keyboard.press('Tab');
   if (!await page.evaluate(() => document.activeElement?.matches('[data-submit-button]'))) fail('correction-order');
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 9b5dfa0..75bebdb 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -516,6 +516,21 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("finishSubmit(200, false)", acceptance.validation_javascript(
             origin=self.origins["ready"], check="validation"
         ))
+        loading_source = acceptance.loading_start_javascript(
+            origin=self.origins["loading"], check="loading"
+        )
+        validation_source = acceptance.validation_javascript(
+            origin=self.origins["ready"], check="validation"
+        )
+        correction_source = acceptance.correction_javascript(
+            origin=self.origins["ready"], check="correction"
+        )
+        self.assertIn("loading-input-redaction", loading_source)
+        self.assertIn("validation-input-redaction", validation_source)
+        self.assertIn("departure.value = ''; returning.value = '';", loading_source)
+        self.assertIn("departure.value = ''; returning.value = '';", validation_source)
+        self.assertIn(acceptance.SYNTHETIC_DEPARTURE, correction_source)
+        self.assertIn(acceptance.SYNTHETIC_VALID_RETURN, correction_source)
         self.assertIn("await route.continue();", source)
         self.assertNotIn("route.continue({ headers", source)
         self.assertNotIn("const csrf =", source)


