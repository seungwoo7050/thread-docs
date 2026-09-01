## `feat(reviews): add the admin review boundary`

diff --git a/reviews/admin.py b/reviews/admin.py
new file mode 100644
index 0000000..11f8d3f
--- /dev/null
+++ b/reviews/admin.py
@@ -0,0 +1,568 @@
+"""Least-privilege operator boundary for immutable publication history."""
+
+from __future__ import annotations
+
+from django.contrib import admin, messages
+
+from entry_requirements.models import EntryFactRevision
+from travel_warnings.models import TravelWarningRevision
+
+from .models import (
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    PublishedEntryFacts,
+    PublishedTravelWarning,
+    ReviewDecision,
+)
+from .publication import (
+    PublicationCode,
+    publish_candidate,
+    reject_candidate,
+    rollback_publication,
+)
+
+
+_GENERIC_FAILURE_MESSAGE = (
+    "작업을 완료하지 못했습니다. 운영 상태를 확인하십시오."
+)
+_SELECTION_MESSAGE = "작업할 항목을 정확히 하나 선택하십시오."
+_SELECTION_REQUIRED = object()
+
+_OUTCOME_MESSAGES = {
+    PublicationCode.PUBLISHED: (
+        messages.SUCCESS,
+        "게시 작업이 완료되었습니다.",
+    ),
+    PublicationCode.REJECTED: (
+        messages.SUCCESS,
+        "검수 거절이 기록되었습니다.",
+    ),
+    PublicationCode.ROLLED_BACK: (
+        messages.SUCCESS,
+        "이전 게시 이력 복원이 완료되었습니다.",
+    ),
+    PublicationCode.NOT_AUTHORIZED: (
+        messages.ERROR,
+        "이 작업을 수행할 권한이 없습니다.",
+    ),
+    PublicationCode.STALE_POINTER: (
+        messages.WARNING,
+        (
+            "게시 상태가 변경되었습니다. "
+            "새로고침 후 다시 시도하십시오."
+        ),
+    ),
+    PublicationCode.INVALID_TARGET: (
+        messages.WARNING,
+        "선택한 항목으로 작업을 완료할 수 없습니다.",
+    ),
+    PublicationCode.SOURCE_GATE_FAILED: (
+        messages.WARNING,
+        (
+            "현재 출처 검증 조건을 충족하지 않아 "
+            "작업을 완료할 수 없습니다."
+        ),
+    ),
+    PublicationCode.TRANSACTION_ABORTED: (
+        messages.ERROR,
+        _GENERIC_FAILURE_MESSAGE,
+    ),
+    PublicationCode.AUDIT_UNAVAILABLE: (
+        messages.ERROR,
+        _GENERIC_FAILURE_MESSAGE,
+    ),
+}
+
+
+def _operator_has_permission(request, permission: str) -> bool:
+    try:
+        user = request.user
+        return bool(
+            user.is_active
+            and user.is_staff
+            and user.has_perm(permission)
+        )
+    except Exception:
+        return False
+
+
+def _single_primary_key(queryset):
+    selected = list(queryset.values_list("pk", flat=True)[:2])
+    if len(selected) != 1:
+        return None
+    return selected[0]
+
+
+def _single_publication_identity(queryset):
+    selected = list(queryset.values_list("pk", "module")[:2])
+    if len(selected) != 1:
+        return None
+    return selected[0]
+
+
+def _current_pointer_version(pointer_model) -> int:
+    version = pointer_model.objects.values_list("version", flat=True).get()
+    if type(version) is not int or version < 0:
+        raise RuntimeError("invalid publication pointer")
+    return version
+
+
+def _fixed_outcome_code(outcome) -> str:
+    try:
+        code = outcome.code
+    except Exception:
+        return PublicationCode.TRANSACTION_ABORTED
+    if type(code) is not str or code not in _OUTCOME_MESSAGES:
+        return PublicationCode.TRANSACTION_ABORTED
+    return code
+
+
+def _run_publish_action(*, queryset, module, actor, pointer_model):
+    try:
+        revision_id = _single_primary_key(queryset)
+        if revision_id is None:
+            return _SELECTION_REQUIRED
+        outcome = publish_candidate(
+            module=module,
+            typed_revision_id=revision_id,
+            actor=actor,
+            expected_pointer_version=_current_pointer_version(pointer_model),
+        )
+        return _fixed_outcome_code(outcome)
+    except Exception:
+        return PublicationCode.TRANSACTION_ABORTED
+
+
+def _run_reject_action(*, queryset, module, actor):
+    try:
+        revision_id = _single_primary_key(queryset)
+        if revision_id is None:
+            return _SELECTION_REQUIRED
+        outcome = reject_candidate(
+            module=module,
+            typed_revision_id=revision_id,
+            actor=actor,
+        )
+        return _fixed_outcome_code(outcome)
+    except Exception:
+        return PublicationCode.TRANSACTION_ABORTED
+
+
+def _run_rollback_action(
+    *, queryset, module, actor, pointer_model
+):
+    try:
+        identity = _single_publication_identity(queryset)
+        if identity is None:
+            return _SELECTION_REQUIRED
+        publication_id, selected_module = identity
+        if selected_module != module:
+            return PublicationCode.INVALID_TARGET
+        outcome = rollback_publication(
+            module=module,
+            target_publication_id=publication_id,
+            actor=actor,
+            expected_pointer_version=_current_pointer_version(pointer_model),
+        )
+        return _fixed_outcome_code(outcome)
+    except Exception:
+        return PublicationCode.TRANSACTION_ABORTED
+
+
+class _ReadOnlyAdmin(admin.ModelAdmin):
+    """Remove every built-in mutation path; database guards remain final."""
+
+    actions = ()
+    list_per_page = 50
+
+    def has_add_permission(self, request):
+        return False
+
+    def has_change_permission(self, request, obj=None):
+        return False
+
+    def has_delete_permission(self, request, obj=None):
+        return False
+
+    def _message_code(self, request, code: str) -> None:
+        level, message = _OUTCOME_MESSAGES.get(
+            code,
+            (messages.ERROR, _GENERIC_FAILURE_MESSAGE),
+        )
+        self.message_user(request, message, level=level)
+
+    def _message_outcome(self, request, outcome) -> None:
+        self._message_code(request, _fixed_outcome_code(outcome))
+
+    def _message_selection(self, request) -> None:
+        self.message_user(request, _SELECTION_MESSAGE, level=messages.WARNING)
+
+    def _message_action_result(self, request, result) -> None:
+        if result is _SELECTION_REQUIRED:
+            self._message_selection(request)
+            return
+        self._message_code(request, result)
+
+    def _reject_non_post(self, request) -> bool:
+        if request.method == "POST":
+            return False
+        self._message_code(request, PublicationCode.INVALID_TARGET)
+        return True
+
+
+@admin.register(EntryFactRevision)
+class EntryFactRevisionAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "country",
+        "passport_policy",
+        "ordinary_passport_period_text",
+        "basis_text",
+        "snapshot_date",
+        "first_observed_at",
+        "created_at",
+    )
+    readonly_fields = fields
+    list_display = (
+        "id",
+        "country",
+        "passport_policy",
+        "ordinary_passport_period_text",
+        "snapshot_date",
+        "first_observed_at",
+        "created_at",
+    )
+    list_select_related = ("country", "passport_policy")
+    ordering = ("-created_at", "-id")
+    actions = ("publish_entry_candidate", "reject_entry_candidate")
+
+    def has_publish_entry_permission(self, request):
+        return _operator_has_permission(request, "reviews.publish_entry")
+
+    def has_reject_entry_permission(self, request):
+        return _operator_has_permission(request, "reviews.reject_entry")
+
+    @admin.action(
+        description="선택한 입국 사실 candidate 검수 승인 및 게시",
+        permissions=("publish_entry",),
+    )
+    def publish_entry_candidate(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        if not self.has_publish_entry_permission(request):
+            self._message_code(request, PublicationCode.NOT_AUTHORIZED)
+            return
+        result = _run_publish_action(
+            queryset=queryset,
+            module=PublicationModule.ENTRY,
+            actor=request.user,
+            pointer_model=PublishedEntryFacts,
+        )
+        self._message_action_result(request, result)
+
+    @admin.action(
+        description="선택한 입국 사실 candidate 검수 거절",
+        permissions=("reject_entry",),
+    )
+    def reject_entry_candidate(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        if not self.has_reject_entry_permission(request):
+            self._message_code(request, PublicationCode.NOT_AUTHORIZED)
+            return
+        result = _run_reject_action(
+            queryset=queryset,
+            module=PublicationModule.ENTRY,
+            actor=request.user,
+        )
+        self._message_action_result(request, result)
+
+
+@admin.register(TravelWarningRevision)
+class TravelWarningRevisionAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "country",
+        "source_alarm_level_code",
+        "source_scope_type",
+        "source_scope_text",
+        "source_written_date",
+        "first_observed_at",
+        "created_at",
+    )
+    readonly_fields = fields
+    list_display = fields
+    list_select_related = ("country",)
+    ordering = ("-created_at", "-id")
+    actions = ("publish_warning_candidate", "reject_warning_candidate")
+
+    def has_publish_warning_permission(self, request):
+        return _operator_has_permission(request, "reviews.publish_warning")
+
+    def has_reject_warning_permission(self, request):
+        return _operator_has_permission(request, "reviews.reject_warning")
+
+    @admin.action(
+        description="선택한 여행경보 candidate 검수 승인 및 게시",
+        permissions=("publish_warning",),
+    )
+    def publish_warning_candidate(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        if not self.has_publish_warning_permission(request):
+            self._message_code(request, PublicationCode.NOT_AUTHORIZED)
+            return
+        result = _run_publish_action(
+            queryset=queryset,
+            module=PublicationModule.TRAVEL_WARNING,
+            actor=request.user,
+            pointer_model=PublishedTravelWarning,
+        )
+        self._message_action_result(request, result)
+
+    @admin.action(
+        description="선택한 여행경보 candidate 검수 거절",
+        permissions=("reject_warning",),
+    )
+    def reject_warning_candidate(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        if not self.has_reject_warning_permission(request):
+            self._message_code(request, PublicationCode.NOT_AUTHORIZED)
+            return
+        result = _run_reject_action(
+            queryset=queryset,
+            module=PublicationModule.TRAVEL_WARNING,
+            actor=request.user,
+        )
+        self._message_action_result(request, result)
+
+
+@admin.register(ReviewDecision)
+class ReviewDecisionAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "module",
+        "typed_revision_id",
+        "decision",
+        "decision_sequence",
+        "reason_code",
+        "actor_principal",
+        "decided_at",
+    )
+    readonly_fields = fields
+    list_display = fields
+    list_filter = ("module", "decision", "reason_code")
+    ordering = ("-decided_at", "-id")
+
+    @admin.display(description="Typed revision ID")
+    def typed_revision_id(self, obj):
+        return obj.entry_fact_revision_id or obj.travel_warning_revision_id
+
+
+@admin.register(PublicationRevision)
+class PublicationRevisionAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "module",
+        "generation",
+        "typed_revision_id",
+        "country_identity",
+        "passport_policy_identity",
+        "entry_period_text",
+        "entry_basis_text",
+        "entry_snapshot_date",
+        "warning_alarm_level_code",
+        "warning_scope_type",
+        "warning_scope_text",
+        "warning_written_date",
+        "completeness",
+        "limitation_code",
+        "state",
+        "supersedes_revision_id",
+        "rollback_target_revision_id",
+        "source_code_snapshot",
+        "source_revision",
+        "source_owner_snapshot",
+        "attribution_text_snapshot",
+        "source_first_observed_at",
+        "created_at",
+        "published_at",
+    )
+    readonly_fields = fields
+    list_display = (
+        "id",
+        "module",
+        "generation",
+        "typed_revision_id",
+        "country_identity",
+        "completeness",
+        "limitation_code",
+        "published_at",
+    )
+    list_filter = ("module", "state", "completeness")
+    ordering = ("-published_at", "-id")
+    actions = ("rollback_entry_publication", "rollback_warning_publication")
+
+    def has_rollback_entry_permission(self, request):
+        return _operator_has_permission(request, "reviews.rollback_entry")
+
+    def has_rollback_warning_permission(self, request):
+        return _operator_has_permission(request, "reviews.rollback_warning")
+
+    @admin.display(description="Typed revision ID")
+    def typed_revision_id(self, obj):
+        return obj.entry_fact_revision_id or obj.travel_warning_revision_id
+
+    @admin.display(description="국가")
+    def country_identity(self, obj):
+        country = obj.scope_country
+        return f"{country.iso_alpha2} — {country.name_ko}"
+
+    @admin.display(description="여권 정책")
+    def passport_policy_identity(self, obj):
+        policy = obj.scope_passport_policy
+        if policy is None:
+            return "—"
+        return f"{policy.code}@{policy.revision}"
+
+    @admin.display(description="입국 체류기간")
+    def entry_period_text(self, obj):
+        revision = obj.entry_fact_revision
+        return revision.ordinary_passport_period_text if revision else "—"
+
+    @admin.display(description="입국 근거")
+    def entry_basis_text(self, obj):
+        revision = obj.entry_fact_revision
+        return revision.basis_text if revision else "—"
+
+    @admin.display(description="입국 snapshot date")
+    def entry_snapshot_date(self, obj):
+        revision = obj.entry_fact_revision
+        return revision.snapshot_date if revision else None
+
+    @admin.display(description="여행경보 단계")
+    def warning_alarm_level_code(self, obj):
+        revision = obj.travel_warning_revision
+        return revision.source_alarm_level_code if revision else "—"
+
+    @admin.display(description="여행경보 범위 유형")
+    def warning_scope_type(self, obj):
+        revision = obj.travel_warning_revision
+        return revision.source_scope_type if revision else "—"
+
+    @admin.display(description="여행경보 범위")
+    def warning_scope_text(self, obj):
+        revision = obj.travel_warning_revision
+        return revision.source_scope_text if revision else "—"
+
+    @admin.display(description="출처 작성일")
+    def warning_written_date(self, obj):
+        revision = obj.travel_warning_revision
+        return revision.source_written_date if revision else None
+
+    @admin.display(description="이전 publication revision ID")
+    def supersedes_revision_id(self, obj):
+        return obj.supersedes_id
+
+    @admin.display(description="복원 대상 publication revision ID")
+    def rollback_target_revision_id(self, obj):
+        return obj.rollback_target_id
+
+    @admin.action(
+        description="선택한 과거 입국 게시 이력으로 복원",
+        permissions=("rollback_entry",),
+    )
+    def rollback_entry_publication(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        self._rollback(
+            request=request,
+            queryset=queryset,
+            module=PublicationModule.ENTRY,
+            pointer_model=PublishedEntryFacts,
+        )
+
+    @admin.action(
+        description="선택한 과거 여행경보 게시 이력으로 복원",
+        permissions=("rollback_warning",),
+    )
+    def rollback_warning_publication(self, request, queryset):
+        if self._reject_non_post(request):
+            return
+        self._rollback(
+            request=request,
+            queryset=queryset,
+            module=PublicationModule.TRAVEL_WARNING,
+            pointer_model=PublishedTravelWarning,
+        )
+
+    def _rollback(self, *, request, queryset, module, pointer_model):
+        permission = (
+            "reviews.rollback_entry"
+            if module == PublicationModule.ENTRY
+            else "reviews.rollback_warning"
+        )
+        if not _operator_has_permission(request, permission):
+            self._message_code(request, PublicationCode.NOT_AUTHORIZED)
+            return
+        result = _run_rollback_action(
+            queryset=queryset,
+            module=module,
+            actor=request.user,
+            pointer_model=pointer_model,
+        )
+        self._message_action_result(request, result)
+
+
+@admin.register(AuditEvent)
+class AuditEventAdmin(_ReadOnlyAdmin):
+    fields = (
+        "id",
+        "module",
+        "action",
+        "outcome",
+        "failure_code",
+        "typed_revision_id",
+        "review_decision_identity",
+        "publication_revision_identity",
+        "prior_publication_revision_identity",
+        "rollback_target_revision_identity",
+        "actor_principal",
+        "occurred_at",
+        "redaction_state",
+    )
+    readonly_fields = fields
+    list_display = (
+        "id",
+        "module",
+        "action",
+        "outcome",
+        "failure_code",
+        "typed_revision_id",
+        "actor_principal",
+        "occurred_at",
+    )
+    list_filter = ("module", "action", "outcome", "failure_code")
+    ordering = ("-occurred_at", "-id")
+
+    @admin.display(description="Typed revision ID")
+    def typed_revision_id(self, obj):
+        return obj.entry_fact_revision_id or obj.travel_warning_revision_id
+
+    @admin.display(description="Review decision ID")
+    def review_decision_identity(self, obj):
+        return obj.review_decision_id
+
+    @admin.display(description="Publication revision ID")
+    def publication_revision_identity(self, obj):
+        return obj.publication_revision_id
+
+    @admin.display(description="Prior publication revision ID")
+    def prior_publication_revision_identity(self, obj):
+        return obj.prior_publication_revision_id
+
+    @admin.display(description="Rollback target revision ID")
+    def rollback_target_revision_identity(self, obj):
+        return obj.rollback_target_publication_revision_id
diff --git a/reviews/tests/test_admin.py b/reviews/tests/test_admin.py
new file mode 100644
index 0000000..492b666
--- /dev/null
+++ b/reviews/tests/test_admin.py
@@ -0,0 +1,412 @@
+import uuid
+from types import SimpleNamespace
+from unittest.mock import patch
+
+from django.contrib import admin, messages
+from django.test import RequestFactory, SimpleTestCase
+
+from entry_requirements.models import EntryFactRevision
+from travel_warnings.models import TravelWarningRevision
+
+from reviews.admin import (
+    AuditEventAdmin,
+    EntryFactRevisionAdmin,
+    PublicationRevisionAdmin,
+    ReviewDecisionAdmin,
+    TravelWarningRevisionAdmin,
+    _GENERIC_FAILURE_MESSAGE,
+    _OUTCOME_MESSAGES,
+)
+from reviews.models import (
+    AuditEvent,
+    PublicationModule,
+    PublicationRevision,
+    ReviewDecision,
+)
+from reviews.publication import PublicationCode, PublicationOutcome
+
+
+class _PermissionUser:
+    is_active = True
+    is_staff = True
+
+    def __init__(self, *permissions):
+        self.permissions = set(permissions)
+
+    def has_perm(self, permission):
+        return permission in self.permissions
+
+
+class _SelectedRows:
+    def __init__(self, rows):
+        self.rows = rows
+
+    def values_list(self, *fields, flat=False):
+        if flat:
+            values = [row[0] for row in self.rows]
+        else:
+            values = self.rows
+        return values
+
+
+class AdminBoundaryTests(SimpleTestCase):
+    def setUp(self):
+        self.factory = RequestFactory()
+        self.site = admin.AdminSite(name="operator-test")
+        self.entry_admin = EntryFactRevisionAdmin(
+            EntryFactRevision,
+            self.site,
+        )
+        self.warning_admin = TravelWarningRevisionAdmin(
+            TravelWarningRevision,
+            self.site,
+        )
+        self.publication_admin = PublicationRevisionAdmin(
+            PublicationRevision,
+            self.site,
+        )
+
+    def request(self, *permissions, method="POST"):
+        request = getattr(self.factory, method.lower())("/operator/")
+        request.user = _PermissionUser(*permissions)
+        return request
+
+    def test_only_typed_candidates_and_immutable_history_are_registered(self):
+        for model in (
+            EntryFactRevision,
+            TravelWarningRevision,
+            ReviewDecision,
+            PublicationRevision,
+            AuditEvent,
+        ):
+            self.assertIn(model, admin.site._registry)
+
+    def test_admin_surfaces_exclude_transport_and_identity_hash_fields(self):
+        forbidden = {
+            "parse_run",
+            "typed_fingerprint_sha256",
+            "source_locator_snapshot",
+            "source_contract_fingerprint_sha256",
+            "parser_contract_fingerprint_sha256",
+            "schema_fingerprint_sha256",
+            "input_identity_sha256",
+            "logical_transaction_id",
+            "correlation_id",
+        }
+        model_admins = (
+            self.entry_admin,
+            self.warning_admin,
+            ReviewDecisionAdmin(ReviewDecision, self.site),
+            self.publication_admin,
+            AuditEventAdmin(AuditEvent, self.site),
+        )
+        for model_admin in model_admins:
+            exposed = set(model_admin.fields) | set(model_admin.list_display)
+            self.assertTrue(forbidden.isdisjoint(exposed))
+
+    def test_all_registered_models_remove_builtin_mutations(self):
+        request = self.request()
+        for model_admin in (
+            self.entry_admin,
+            self.warning_admin,
+            ReviewDecisionAdmin(ReviewDecision, self.site),
+            self.publication_admin,
+            AuditEventAdmin(AuditEvent, self.site),
+        ):
+            self.assertFalse(model_admin.has_add_permission(request))
+            self.assertFalse(model_admin.has_change_permission(request))
+            self.assertFalse(model_admin.has_delete_permission(request))
+
+    def test_action_visibility_requires_exact_permission(self):
+        request = self.request("reviews.publish_entry")
+        actions = self.entry_admin.get_actions(request)
+        self.assertIn("publish_entry_candidate", actions)
+        self.assertNotIn("reject_entry_candidate", actions)
+
+        wrong_module_request = self.request("reviews.publish_warning")
+        self.assertNotIn(
+            "publish_entry_candidate",
+            self.entry_admin.get_actions(wrong_module_request),
+        )
+
+    def test_each_action_rejects_get_before_permission_or_data_access(self):
+        cases = (
+            (
+                self.entry_admin,
+                self.entry_admin.publish_entry_candidate,
+                "reviews.publish_entry",
+                _SelectedRows([(uuid.uuid4(),)]),
+            ),
+            (
+                self.entry_admin,
+                self.entry_admin.reject_entry_candidate,
+                "reviews.reject_entry",
+                _SelectedRows([(uuid.uuid4(),)]),
+            ),
+            (
+                self.warning_admin,
+                self.warning_admin.publish_warning_candidate,
+                "reviews.publish_warning",
+                _SelectedRows([(uuid.uuid4(),)]),
+            ),
+            (
+                self.warning_admin,
+                self.warning_admin.reject_warning_candidate,
+                "reviews.reject_warning",
+                _SelectedRows([(uuid.uuid4(),)]),
+            ),
+            (
+                self.publication_admin,
+                self.publication_admin.rollback_entry_publication,
+                "reviews.rollback_entry",
+                _SelectedRows(
+                    [(uuid.uuid4(), PublicationModule.ENTRY)]
+                ),
+            ),
+            (
+                self.publication_admin,
+                self.publication_admin.rollback_warning_publication,
+                "reviews.rollback_warning",
+                _SelectedRows(
+                    [(uuid.uuid4(), PublicationModule.TRAVEL_WARNING)]
+                ),
+            ),
+        )
+        expected = _OUTCOME_MESSAGES[PublicationCode.INVALID_TARGET]
+        for model_admin, action, permission, rows in cases:
+            with self.subTest(action=action.__name__):
+                request = self.request(permission, method="GET")
+                with (
+                    patch("reviews.admin._operator_has_permission") as check,
+                    patch("reviews.admin._current_pointer_version") as pointer,
+                    patch("reviews.admin.publish_candidate") as publish,
+                    patch("reviews.admin.reject_candidate") as reject,
+                    patch("reviews.admin.rollback_publication") as rollback,
+                    patch.object(model_admin, "message_user") as message,
+                ):
+                    action(request, rows)
+
+                check.assert_not_called()
+                pointer.assert_not_called()
+                publish.assert_not_called()
+                reject.assert_not_called()
+                rollback.assert_not_called()
+                message.assert_called_once_with(
+                    request,
+                    expected[1],
+                    level=expected[0],
+                )
+
+    def test_entry_publish_passes_only_typed_id_actor_and_pointer_version(self):
+        revision_id = uuid.uuid4()
+        request = self.request("reviews.publish_entry")
+        outcome = PublicationOutcome(
+            PublicationCode.PUBLISHED,
+            PublicationModule.ENTRY,
+            generation=3,
+            pointer_version=3,
+        )
+        with (
+            patch("reviews.admin._current_pointer_version", return_value=2),
+            patch("reviews.admin.publish_candidate", return_value=outcome) as call,
+            patch.object(self.entry_admin, "message_user") as message,
+        ):
+            self.entry_admin.publish_entry_candidate(
+                request,
+                _SelectedRows([(revision_id,)]),
+            )
+
+        call.assert_called_once_with(
+            module=PublicationModule.ENTRY,
+            typed_revision_id=revision_id,
+            actor=request.user,
+            expected_pointer_version=2,
+        )
+        self.assertEqual(message.call_args.kwargs["level"], messages.SUCCESS)
+
+    def test_warning_reject_passes_only_typed_id_and_actor(self):
+        revision_id = uuid.uuid4()
+        request = self.request("reviews.reject_warning")
+        outcome = PublicationOutcome(
+            PublicationCode.REJECTED,
+            PublicationModule.TRAVEL_WARNING,
+        )
+        with (
+            patch("reviews.admin.reject_candidate", return_value=outcome) as call,
+            patch.object(self.warning_admin, "message_user") as message,
+        ):
+            self.warning_admin.reject_warning_candidate(
+                request,
+                _SelectedRows([(revision_id,)]),
+            )
+
+        call.assert_called_once_with(
+            module=PublicationModule.TRAVEL_WARNING,
+            typed_revision_id=revision_id,
+            actor=request.user,
+        )
+        self.assertEqual(message.call_args.kwargs["level"], messages.SUCCESS)
+
+    def test_rollback_is_single_target_and_exact_module(self):
+        publication_id = uuid.uuid4()
+        request = self.request("reviews.rollback_entry")
+        outcome = PublicationOutcome(
+            PublicationCode.ROLLED_BACK,
+            PublicationModule.ENTRY,
+            generation=4,
+            pointer_version=4,
+        )
+        with (
+            patch("reviews.admin._current_pointer_version", return_value=3),
+            patch("reviews.admin.rollback_publication", return_value=outcome) as call,
+            patch.object(self.publication_admin, "message_user"),
+        ):
+            self.publication_admin.rollback_entry_publication(
+                request,
+                _SelectedRows([(publication_id, PublicationModule.ENTRY)]),
+            )
+
+        call.assert_called_once_with(
+            module=PublicationModule.ENTRY,
+            target_publication_id=publication_id,
+            actor=request.user,
+            expected_pointer_version=3,
+        )
+
+    def test_cross_module_and_multiple_selection_never_call_service(self):
+        request = self.request("reviews.rollback_entry")
+        with (
+            patch("reviews.admin.rollback_publication") as call,
+            patch.object(self.publication_admin, "message_user") as message,
+        ):
+            self.publication_admin.rollback_entry_publication(
+                request,
+                _SelectedRows(
+                    [
+                        (uuid.uuid4(), PublicationModule.ENTRY),
+                        (uuid.uuid4(), PublicationModule.ENTRY),
+                    ]
+                ),
+            )
+            self.publication_admin.rollback_entry_publication(
+                request,
+                _SelectedRows(
+                    [(uuid.uuid4(), PublicationModule.TRAVEL_WARNING)]
+                ),
+            )
+
+        call.assert_not_called()
+        self.assertEqual(message.call_count, 2)
+
+    def test_direct_action_invocation_rechecks_permission(self):
+        request = self.request()
+        with (
+            patch("reviews.admin.publish_candidate") as call,
+            patch.object(self.entry_admin, "message_user") as message,
+        ):
+            self.entry_admin.publish_entry_candidate(
+                request,
+                _SelectedRows([(uuid.uuid4(),)]),
+            )
+
+        call.assert_not_called()
+        self.assertEqual(message.call_args.kwargs["level"], messages.ERROR)
+
+    def test_all_service_outcomes_use_a_fixed_message(self):
+        request = self.request()
+        allowed_messages = {message for _level, message in _OUTCOME_MESSAGES.values()}
+        for code in _OUTCOME_MESSAGES:
+            outcome = PublicationOutcome(code, PublicationModule.ENTRY)
+            with patch.object(self.entry_admin, "message_user") as message:
+                self.entry_admin._message_outcome(request, outcome)
+            rendered = message.call_args.args[1]
+            self.assertIn(rendered, allowed_messages)
+
+    def test_ordinary_failure_and_unknown_outcome_are_generic(self):
+        request = self.request("reviews.publish_entry")
+        detail = "sensitive-marker-that-must-not-be-rendered"
+        with (
+            patch(
+                "reviews.admin.publish_candidate",
+                side_effect=RuntimeError(detail),
+            ),
+            patch("reviews.admin._current_pointer_version", return_value=0),
+            patch.object(self.entry_admin, "message_user") as message,
+        ):
+            self.entry_admin.publish_entry_candidate(
+                request,
+                _SelectedRows([(uuid.uuid4(),)]),
+            )
+        rendered = message.call_args.args[1]
+        self.assertEqual(rendered, _GENERIC_FAILURE_MESSAGE)
+        self.assertNotIn(detail, rendered)
+
+        unknown = SimpleNamespace(code=detail)
+        with patch.object(self.entry_admin, "message_user") as message:
+            self.entry_admin._message_outcome(request, unknown)
+        self.assertEqual(message.call_args.args[1], _GENERIC_FAILURE_MESSAGE)
+
+    def test_message_backend_failure_has_no_service_exception_context(self):
+        request = self.request("reviews.publish_entry")
+        service_marker = "service-boundary-marker"
+        message_marker = "message-boundary-marker"
+        caught_exception = None
+        caught_traceback = None
+        with (
+            patch("reviews.admin._current_pointer_version", return_value=0),
+            patch(
+                "reviews.admin.publish_candidate",
+                side_effect=RuntimeError(service_marker),
+            ),
+            patch.object(
+                self.entry_admin,
+                "message_user",
+                side_effect=RuntimeError(message_marker),
+            ),
+        ):
+            try:
+                self.entry_admin.publish_entry_candidate(
+                    request,
+                    _SelectedRows([(uuid.uuid4(),)]),
+                )
+            except RuntimeError as exception:
+                caught_exception = exception
+                caught_traceback = exception.__traceback__
+
+        self.assertIsNotNone(caught_exception)
+        self.assertEqual(caught_exception.args, (message_marker,))
+        self.assertIsNone(caught_exception.__cause__)
+        self.assertIsNone(caught_exception.__context__)
+        traceback = caught_traceback
+        production_frames = []
+        while traceback is not None:
+            frame = traceback.tb_frame
+            if frame.f_code.co_filename.endswith("/reviews/admin.py"):
+                production_frames.append(frame)
+            traceback = traceback.tb_next
+        self.assertTrue(production_frames)
+        for frame in production_frames:
+            for value in frame.f_locals.values():
+                self.assertNotIn(service_marker, repr(value))
+
+    def test_process_control_is_not_absorbed_or_rendered(self):
+        request = self.request("reviews.publish_entry")
+        for exception_type in (KeyboardInterrupt, SystemExit):
+            with self.subTest(exception_type=exception_type.__name__):
+                with (
+                    patch(
+                        "reviews.admin._current_pointer_version",
+                        return_value=0,
+                    ),
+                    patch(
+                        "reviews.admin.publish_candidate",
+                        side_effect=exception_type(),
+                    ),
+                    patch.object(self.entry_admin, "message_user") as message,
+                ):
+                    with self.assertRaises(exception_type):
+                        self.entry_admin.publish_entry_candidate(
+                            request,
+                            _SelectedRows([(uuid.uuid4(),)]),
+                        )
+                message.assert_not_called()


