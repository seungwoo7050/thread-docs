## `fix(db): enforce reviewed flight pointer closure`

diff --git a/travel_windows/migrations/0007_flight_review_lifecycle.py b/travel_windows/migrations/0007_flight_review_lifecycle.py
index cc062d4..1c19ba1 100644
--- a/travel_windows/migrations/0007_flight_review_lifecycle.py
+++ b/travel_windows/migrations/0007_flight_review_lifecycle.py
@@ -225,6 +225,124 @@ $$;
 CREATE TRIGGER travel_windows_00_candidate_publication_duration_guard
 BEFORE INSERT ON travel_windows_flightpublicationduration
 FOR EACH ROW EXECUTE FUNCTION travel_windows_guard_candidate_publication_duration();
+
+CREATE OR REPLACE FUNCTION travel_windows_guard_flight_pointer() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    candidate_generation bigint;
+    candidate_supersedes uuid;
+    publication_schedule_id uuid;
+    candidate_review_id uuid;
+    candidate_rollback_target_id uuid;
+    candidate_published_by text;
+    review_matches boolean;
+    audit_matches boolean;
+    candidate_duration_count bigint;
+    duration_closure_matches boolean;
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'flight current pointer cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.id IS DISTINCT FROM OLD.id
+       OR NEW.version <> OLD.version + 1
+       OR NEW.current_publication_id IS NULL THEN
+        RAISE EXCEPTION 'flight current pointer transition is invalid'
+            USING ERRCODE = 'check_violation';
+    END IF;
+
+    SELECT publication.generation,
+           publication.supersedes_id,
+           publication.schedule_revision_id,
+           publication.review_decision_id,
+           publication.rollback_target_id,
+           publication.published_by
+      INTO candidate_generation,
+           candidate_supersedes,
+           publication_schedule_id,
+           candidate_review_id,
+           candidate_rollback_target_id,
+           candidate_published_by
+      FROM travel_windows_flightpublication AS publication
+     WHERE publication.id = NEW.current_publication_id;
+
+    SELECT EXISTS (
+        SELECT 1
+          FROM travel_windows_flightreviewdecision AS review
+         WHERE review.id = candidate_review_id
+           AND review.schedule_revision_id = publication_schedule_id
+           AND review.decision = 'APPROVED'
+           AND review.actor_principal = candidate_published_by
+           AND (
+               (
+                   candidate_rollback_target_id IS NULL
+                   AND review.reason_code = 'APPROVE_FLIGHT_CANDIDATE'
+                   AND review.rollback_target_publication_id IS NULL
+               )
+               OR (
+                   candidate_rollback_target_id IS NOT NULL
+                   AND review.reason_code = 'ROLLBACK_FLIGHT_PUBLICATION'
+                   AND review.rollback_target_publication_id = candidate_rollback_target_id
+               )
+           )
+    ) INTO review_matches;
+
+    SELECT EXISTS (
+        SELECT 1
+          FROM travel_windows_flightauditevent AS audit
+         WHERE audit.publication_id = NEW.current_publication_id
+           AND audit.review_decision_id = candidate_review_id
+           AND audit.schedule_revision_id = publication_schedule_id
+           AND audit.outcome = 'SUCCEEDED'
+           AND audit.prior_publication_id IS NOT DISTINCT FROM OLD.current_publication_id
+           AND audit.rollback_target_publication_id
+               IS NOT DISTINCT FROM candidate_rollback_target_id
+           AND audit.action = CASE
+               WHEN candidate_rollback_target_id IS NULL THEN 'PUBLISH'
+               ELSE 'ROLLBACK'
+           END
+    ) INTO audit_matches;
+
+    SELECT COUNT(*)
+      INTO candidate_duration_count
+      FROM travel_windows_flightcandidateduration AS candidate
+     WHERE candidate.schedule_revision_id = publication_schedule_id;
+
+    SELECT NOT EXISTS (
+               SELECT 1
+                 FROM travel_windows_flightcandidateduration AS candidate
+                 LEFT JOIN travel_windows_flightpublicationduration AS published
+                   ON published.publication_id = NEW.current_publication_id
+                  AND published.duration_revision_id = candidate.duration_revision_id
+                  AND published.destination_airport_id = candidate.destination_airport_id
+                WHERE candidate.schedule_revision_id = publication_schedule_id
+                  AND published.id IS NULL
+           )
+           AND NOT EXISTS (
+               SELECT 1
+                 FROM travel_windows_flightpublicationduration AS published
+                 LEFT JOIN travel_windows_flightcandidateduration AS candidate
+                   ON candidate.schedule_revision_id = publication_schedule_id
+                  AND candidate.duration_revision_id = published.duration_revision_id
+                  AND candidate.destination_airport_id = published.destination_airport_id
+                WHERE published.publication_id = NEW.current_publication_id
+                  AND candidate.id IS NULL
+           )
+      INTO duration_closure_matches;
+
+    IF candidate_generation <> NEW.version
+       OR candidate_supersedes IS DISTINCT FROM OLD.current_publication_id
+       OR candidate_review_id IS NULL
+       OR NOT review_matches
+       OR NOT audit_matches
+       OR candidate_duration_count < 1
+       OR NOT duration_closure_matches THEN
+        RAISE EXCEPTION 'flight publication does not extend reviewed current history'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
 """
 
 
@@ -247,6 +365,36 @@ DROP TRIGGER IF EXISTS travel_windows_review_decision_immutable
     ON travel_windows_flightreviewdecision;
 DROP TRIGGER IF EXISTS travel_windows_candidate_duration_immutable
     ON travel_windows_flightcandidateduration;
+
+CREATE OR REPLACE FUNCTION travel_windows_guard_flight_pointer() RETURNS trigger
+LANGUAGE plpgsql AS $$
+DECLARE
+    candidate_generation bigint;
+    candidate_supersedes uuid;
+BEGIN
+    IF TG_OP = 'DELETE' THEN
+        RAISE EXCEPTION 'flight current pointer cannot be deleted'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    IF NEW.id IS DISTINCT FROM OLD.id
+       OR NEW.version <> OLD.version + 1
+       OR NEW.current_publication_id IS NULL THEN
+        RAISE EXCEPTION 'flight current pointer transition is invalid'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    SELECT generation, supersedes_id
+      INTO candidate_generation, candidate_supersedes
+      FROM travel_windows_flightpublication
+     WHERE id = NEW.current_publication_id;
+    IF NOT FOUND
+       OR candidate_generation <> NEW.version
+       OR candidate_supersedes IS DISTINCT FROM OLD.current_publication_id THEN
+        RAISE EXCEPTION 'flight publication does not extend current history'
+            USING ERRCODE = 'check_violation';
+    END IF;
+    RETURN NEW;
+END;
+$$;
 """
 
 


