## `docs: document publisher operations`

diff --git a/README.md b/README.md
index 65bf06e..e3199ab 100644
--- a/README.md
+++ b/README.md
@@ -1,52 +1,153 @@
 # Content Foundry Publisher
 
-Content Foundry Publisher is a private, Git-backed editorial workspace for preparing, reviewing, approving, and publishing articles.
+Content Foundry Publisher is a private, Git-backed editorial workspace. Canonical
+articles are Markdown with YAML front matter; a site record selects exactly one
+publication engine. ContentOps is retired and no ContentOps code, migrations,
+contracts, runtime state, or Git history are part of this repository.
 
-This repository starts from a new root history. It does not continue ContentOps history and does not import ContentOps code. ContentOps is retired.
+The MVP fails closed: publication needs a committed, interactive human approval
+for the exact 40-character source commit. Approval and publication are separate
+append-only audit events. A successful publication has one deterministic receipt,
+so retrying the same article/site/engine/source tuple cannot publish it twice.
 
-## Product boundary
+## Prerequisites and local gate
 
-The product owns:
+- Node.js 22 through 24 and npm 11
+- Git
+- `fnm`, Corepack, and pnpm 11.22.0 only for the frozen Public Sites adapter
+- Docker Desktop for the supported `wp-env` publication transport
 
-- Markdown article source and YAML front matter
-- editorial status and review through private Git branches and pull requests
-- an explicit human approval gate before every publication
-- deterministic validation and target-specific publication adapters
-- publication receipts that identify the approved source commit and remote result
+From a clean checkout:
 
-The product does not own:
+```sh
+npm ci --ignore-scripts
+npm run check
+npm audit --omit=dev
+```
 
-- autonomous approval or unattended publication
-- real-time issue discovery
-- a general-purpose blog hosting platform
-- secrets committed to Git
-- direct imports from the retired ContentOps implementation
+`npm run check` validates every article and site, scans tracked and unignored
+repository files for high-confidence secret material, and runs the full test suite.
+No cloud account, provider identifier, domain, or production credential is needed.
 
-AI-assisted drafting may be added later, but generated text is always an unapproved draft. No model may approve or publish content.
+## Canonical content and sites
 
-## Initial delivery direction
+Articles live in `content/articles/*.md` and sites in `sites/*.yml`. An article
+contains only a `site` identifier. The selected site owns the engine choice:
 
-1. Run the editorial workflow locally with Decap CMS local backend.
-2. Store canonical content as Markdown and YAML in this repository.
-3. Validate content before review and again at the approved commit.
-4. Require the user to approve the pull request or equivalent local review checkpoint.
-5. Publish through exactly one adapter selected by the site configuration:
-   - frozen Public Sites release-directory/build-report interface; or
-   - local WordPress through `wp-env`, designed to move later to WordPress.com Business.
-6. Record a non-secret publication receipt without changing the approved article content.
+- `public_sites` uses the frozen release directory -> target build -> build report
+  boundary.
+- `wordpress` creates or updates a remote post whose status is always `draft`.
 
-Public Sites remains a separate frozen repository. Integration is permitted only through its public release-directory input and build-report output. Publisher code must not import Public Sites internals or modify its renderer as part of normal content publication.
+Schemas in `schemas/` reject unknown fields. Content, fixtures, logs, receipts, and
+Git must never contain credentials, tokens, cookies, recovery codes, TOTP seeds,
+or production provider identifiers.
 
-## Repository state
+## Local authoring with Decap
 
-This initial commit intentionally contains policy and product boundaries only. It makes no claim that an application, CMS, WordPress environment, cloud account, domain, or deployment exists.
+The reviewed Decap packages are exact-version dependencies; their source is not
+copied or forked. Run two terminals from this repository:
 
-Implementation begins in a new Codex session from the fixed decisions in [`docs/PRODUCT-DECISIONS.md`](docs/PRODUCT-DECISIONS.md) and the rules in [`DEVELOPMENT-RULES.md`](DEVELOPMENT-RULES.md).
+```sh
+npm run cms:proxy
+```
 
-## Security
+```sh
+npm run cms:web
+```
 
-Never commit passwords, API tokens, recovery codes, TOTP seeds, cookies, provider credentials, or production identifiers. Account creation, payment, 2FA, and secret entry are interactive user checkpoints.
+Open `http://127.0.0.1:8080`. Both services bind to loopback. Decap's editorial
+workflow writes Git branches and commits in this private repository. Review and
+merge those commits normally, then run `npm run check` on the candidate commit.
 
-## Licensing
+## Explicit approval
 
-This private repository does not grant a general license to redistribute its original source. Third-party packages keep their own licenses and notices; see [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md). A future WordPress plugin must live in a clearly separated GPL-compatible package.
+Approval must be performed by the reviewer in an interactive terminal while the
+candidate commit is clean and checked out:
+
+```sh
+git rev-parse HEAD
+npm run approve -- \
+  --article content/articles/publisher-loop.md \
+  --source-sha <full-source-sha> \
+  --actor <reviewer-name> \
+  --decision approved
+```
+
+The command displays the immutable article, site, and SHA and accepts only the
+exact phrase `APPROVE <full-source-sha>`. There is no non-interactive approval
+flag. Commit the new `.publisher/events/approvals/*.json` file before publishing.
+A later committed rejection for the same source blocks publication.
+
+Without that committed event, the publish command exits with `APPROVAL_REQUIRED`
+before it constructs or calls an adapter.
+
+## Public Sites publication
+
+Use only an external checkout of
+`seungwoo7050/content-foundry-public-sites` detached at
+`1717326cda8262d7f7f56d544b3a9d0215b71d51`. Do not modify or commit in it.
+Set `PUBLISHER_PUBLIC_SITES_REPO` to that checkout in the current shell and run:
+
+```sh
+npm run publish -- \
+  --article content/articles/publisher-loop.md \
+  --source-sha <approved-source-sha>
+```
+
+The adapter compiles a release outside the renderer checkout, builds only
+`site-a`, verifies the frozen HEAD and tracked state, and fingerprints the target
+output into a build report. Analytics, ads, origin, and provider configuration are
+disabled. The proved interface and captured report are described in
+[`docs/PUBLIC-SITES-VIABILITY.md`](docs/PUBLIC-SITES-VIABILITY.md).
+
+## Local WordPress publication
+
+Start the Docker-backed development environment:
+
+```sh
+npm run wp:start
+```
+
+After approving and committing `content/articles/wordpress-loop.md`, run:
+
+```sh
+npm run publish -- \
+  --article content/articles/wordpress-loop.md \
+  --source-sha <approved-source-sha>
+```
+
+The default transport invokes WP-CLI through `wp-env`; it does not need a password
+in a command, file, or receipt. It creates a draft, embeds a non-secret idempotency
+marker, retains the returned post ID, recovers the same draft after receipt loss,
+and updates that post for a newly approved source. It refuses a non-draft or a post
+owned by another article. Stop it with `npm run wp:stop`.
+
+For a future remote WordPress endpoint, set `PUBLISHER_WP_TRANSPORT=rest`. The URL
+and credential variable names are declared by the site record and values must be
+injected only at runtime. Non-loopback HTTP is rejected. Do not paste credential
+values into issues, chat, Git, logs, or documentation; stop for an approved
+interactive secret-entry procedure when they are required.
+
+The repeatable create/update/failure/retry proof is in
+[`docs/WORDPRESS-WP-ENV-PROOF.md`](docs/WORDPRESS-WP-ENV-PROOF.md).
+
+## Audit and rollback
+
+- Approval/rejection: `.publisher/events/approvals/*.json`
+- Publication start/failure: `.publisher/events/publications/*.json`
+- Atomic success: `.publisher/publications/<receipt-id>/{event,receipt}.json`
+- Transient locks/work: `.publisher/locks/` and `.publisher/work/` (Git-ignored)
+
+Commit audit records after each completed operation. The receipt ID is SHA-256 of
+the versioned article/site/engine/source identity. Re-running a completed command
+returns the existing receipt without calling the adapter. A failed attempt writes
+only a redacted failure code and remains retryable. Roll back source or policy with
+a normal revert and a new review/approval; never rewrite audited Git history.
+
+## Security and dependency boundary
+
+Decap is local-only because its development dependency graph contains accepted
+legacy denial-of-service advisories. `npm audit --omit=dev` covers Publisher's
+production dependency tree. See [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md)
+and [`docs/COMPLETION-REPORT.md`](docs/COMPLETION-REPORT.md) for the pinned versions,
+evidence, and remaining environment blocker.
diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index 48bc75f..d55b94b 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -13,6 +13,12 @@ and SHA-512 integrity for direct and transitive packages.
 | `decap-cms-app` | 3.15.1 | https://github.com/decaporg/decap-cms | MIT | Local editorial UI dependency; it is neither copied nor forked. Retain upstream copyright/license when redistributed. |
 | `decap-server` | 3.10.0 | https://github.com/decaporg/decap-cms | MIT | Local-backend proxy used only during local authoring; retain upstream copyright/license when redistributed. |
 
+The disposable wp-env configuration downloads the official WordPress 7.1 release
+archive from `downloads.wordpress.org` at runtime. WordPress is GPL-2.0-or-later,
+is not committed or redistributed by Publisher, and remains isolated from this
+repository's original source. The supported publication proof uses wp-env's Docker
+runtime; its Playground runtime cannot execute `wp-env run` in the pinned version.
+
 Registry metadata and package contents were reviewed before pinning. `npm ci`
 verifies the lockfile integrity hashes. Transitive packages retain their own
 licenses; `node_modules` is excluded from Git and no production bundle is committed.
diff --git a/docs/COMPLETION-REPORT.md b/docs/COMPLETION-REPORT.md
new file mode 100644
index 0000000..d8d2c1c
--- /dev/null
+++ b/docs/COMPLETION-REPORT.md
@@ -0,0 +1,86 @@
+# Publisher MVP completion report
+
+Date: 2026-08-29 (Asia/Seoul)
+
+## Baseline and repository isolation
+
+Development began from clean Publisher commit
+`92d96f47068dbf0464211b9acf41dc9547562671`. The private origin is
+`seungwoo7050/content-foundry-publisher`, and the repository starts with one root
+commit. No retired ContentOps repository, code, migration, contract, runtime state,
+or history was read or imported.
+
+The Public Sites repository was verified as archived at
+`seungwoo7050/content-foundry-public-sites`; branch `public-sites` resolved to
+`1717326cda8262d7f7f56d544b3a9d0215b71d51`. Publisher uses only its release
+directory -> target build -> build report interface.
+
+## Closed-loop implementation
+
+- Strict Markdown/YAML schemas and deterministic Public Sites and WordPress
+  fixtures.
+- Exact pinned Decap local-backend editorial dependencies and loopback services.
+- TTY-only approval/rejection tied to clean immutable Git HEAD, with committed
+  approval required before adapter dispatch.
+- Separate approval, publication-start, publication-failure, and publication-
+  success audit events.
+- Deterministic receipt identity and atomic success event/receipt directory.
+- Public Sites adapter pinned to the frozen SHA and `site-a` target, with a
+  provider-free environment and verified output digest.
+- WordPress draft-only adapter with retained remote post ID, idempotency marker,
+  receipt-loss recovery, and wp-env/REST transports whose errors are redacted.
+- Repository gate covering validation, schema drift, secret detection, approval
+  bypass, rejection, retry, partial failure, receipt collision, and adapter failure.
+
+## Evidence obtained
+
+The actual Public Sites spike compiled `content/articles/publisher-loop.md` at
+Publisher commit `a62af56dde3f7dc9d6a7bee0ec7f81e83e7b2dc8`. At the exact frozen renderer
+SHA, all six prerequisite builds and the `site-a` target build succeeded. Next.js
+generated 14 routes including the article, category, and about pages. Publisher
+verified 64 output files. The committed build report has SHA-256
+`e58ee42735385d0ef3607ebca8883923602773815ac0e17a8b5dc0667e9667b9` and records
+output digest
+`sha256:28d854cccc32280182e893c0ae3c01b7d68e5fb855deda33d0c088ee2b9587fc`.
+
+The approval gate was also exercised from the real CLI at clean Publisher commit
+`d20eed695e4c5e5cba5e51e7327507eb53ebd60d`: publishing the WordPress fixture
+without an approval event exited with `APPROVAL_REQUIRED` before adapter dispatch
+and produced no publication state.
+
+Focused tests prove both adapters' success and redacted failure paths. Public Sites
+tests retry a failed build and reject work inside the frozen renderer. WordPress
+tests create one draft, recover it after receipt loss, update the same post ID for a
+new source, reject non-drafts, redact command/HTTP errors, and keep status `draft`.
+Orchestrator tests prove a successful retry calls an adapter once and a failed
+attempt remains retryable without a duplicate receipt.
+
+## Environment blocker: live wp-env mutation
+
+The requested live WordPress create/update/retry sequence was not marked as passed.
+Docker Desktop 28.0.4 was available, but two Docker-backed `wp-env start` attempts
+stalled while pulling `mysql` and `phpmyadmin`; direct image pull and manifest
+diagnostics had the same registry-path stall. The interrupted attempt left no
+WordPress, MariaDB, or phpMyAdmin container or image.
+
+As a safe alternative check, wp-env's Playground runtime downloaded and started
+the pinned WordPress 7.1 core at loopback. However, `wp-env run cli wp ...` returned
+that `run` is unsupported for Playground in `@wordpress/env` 11.14.0. Using its
+generated login values through an internal or undocumented path would violate the
+credential checkpoint and would not prove the supported transport, so the session
+stopped there. The executable closure procedure is in
+[`WORDPRESS-WP-ENV-PROOF.md`](WORDPRESS-WP-ENV-PROOF.md).
+
+This blocker is environmental, not reported as a passing gate. It prevents a claim
+that the complete two-adapter loop was demonstrated against a live WordPress
+instance in this session. No external WordPress account, Cloudflare resource,
+domain, provider identifier, payment flow, or production credential was used.
+
+## Final verification contract
+
+Before delivery, run `npm ci --ignore-scripts`, `npm run check`,
+`npm audit --omit=dev`, `git diff --check`, verify the frozen Public Sites GitHub
+archive/branch state again, push all commits, and compare clean local HEAD with
+`origin/main`. The exact final SHA and test counts are reported after that immutable
+push rather than embedded circularly in its own commit.
+
diff --git a/docs/WORDPRESS-WP-ENV-PROOF.md b/docs/WORDPRESS-WP-ENV-PROOF.md
new file mode 100644
index 0000000..9bac789
--- /dev/null
+++ b/docs/WORDPRESS-WP-ENV-PROOF.md
@@ -0,0 +1,61 @@
+# WordPress wp-env proof
+
+This procedure exercises the real local adapter without a hosted account or a
+credential value in Git, logs, commands, fixtures, or receipts. Run it only with
+Docker Desktop available and a clean private Publisher checkout.
+
+## Supported runtime
+
+The supported adapter transport is Docker-backed `wp-env` because it can execute
+WP-CLI inside the disposable environment. The optional wp-env Playground runtime
+can start WordPress, but `@wordpress/env` 11.14.0 rejects its `run` command and
+therefore cannot execute this adapter proof.
+
+```sh
+npm ci --ignore-scripts
+npm run wp:start
+node_modules/.bin/wp-env run cli wp core version
+```
+
+If an image pull, Docker daemon, login, password, token, payment, or 2FA checkpoint
+blocks these commands, stop. Do not substitute a production account or place a
+secret on the command line.
+
+## Create and identical retry
+
+1. Check out and review the WordPress fixture commit; run `npm run check`.
+2. In a TTY, approve its exact HEAD SHA with `npm run approve -- ...` and the
+   literal `APPROVE <sha>` phrase.
+3. Commit the approval event.
+4. Publish the approved source SHA. Record only the receipt ID and remote post ID.
+5. Run the identical publish command again.
+6. Verify it says `Reused publication receipt`, the receipt ID is unchanged, and
+   WP-CLI reports exactly one draft with that slug.
+
+The second command returns the atomic receipt before calling WordPress.
+
+## Update and partial-failure retry
+
+1. Commit the first publication audit records and receipt.
+2. Change the canonical article, commit it, review it, approve that new exact SHA,
+   and commit the new approval event.
+3. Stop wp-env and invoke publication once. It must fail with a redacted
+   `WP_ENV_COMMAND_FAILED` event and create no success receipt.
+4. Restart wp-env and repeat the same publish command.
+5. Verify the new receipt records the same remote post ID, status `draft`, and the
+   updated body. Verify there is still exactly one matching WordPress draft.
+
+This proves create, update, retry after a transport failure, remote identity
+retention, and no duplicate post. A source rollback is a normal Git revert followed
+by a new review and approval; it updates the retained draft instead of rewriting an
+old receipt.
+
+## Cleanup
+
+```sh
+npm run wp:stop
+```
+
+The environment is disposable local state. Do not archive its database, generated
+configuration, or runtime files in Git.
+
