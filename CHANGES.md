# Changes Made — Cloudsmith Support Assessment

This document lists the 6 issues found and fixed in the repository, followed
by the redesign of `promote_package.yml` to trigger from a Cloudsmith
webhook instead of a manual dispatch.

## Part 1: Six issues found and fixed

### 1. `pyproject.toml` — missing package `name` and `version`
**File:** `pyproject.toml`
**Problem:** The `[project]` table had no `name` or `version` field. Both
are required by the `pyproject.toml` spec / hatchling, so `python -m build`
would fail immediately with a metadata error, before a wheel or sdist was
ever produced.
**Fix:** Added `name = "example_package"` and `version = "0.0.1"` under
`[project]`, matching the package under `src/` and the version already
declared in `src/example_package/__init__.py`.

### 2. `build_package.yml` — "Build package" step has no command
**File:** `.github/workflows/build_package.yml`
**Problem:** The `Build package` step declared only a `name:`, with no
`run:` key underneath it. A step with no `run`/`uses` is invalid, so the
build job would never actually build anything (and would fail workflow
validation/execution).
**Fix:** Added `run: python -m build` to the step, using the `build`
package already installed in the previous step.

### 3. `release_package.yml` — missing `id-token: write` permission
**File:** `.github/workflows/release_package.yml`
**Problem:** The workflow authenticates to Cloudsmith via OIDC (per the
project's authentication requirement), but the job only granted
`contents: read` and `actions: read`. Without `id-token: write`, GitHub
will not issue the OIDC token the Cloudsmith CLI needs, so authentication
would fail.
**Fix:** Added `id-token: write` to the `permissions` block.

### 4. `release_package.yml` — no Cloudsmith CLI install/auth step
**File:** `.github/workflows/release_package.yml`
**Problem:** The workflow called `cloudsmith push ...` directly, but never
installed the Cloudsmith CLI or authenticated it. The `promote_package.yml`
workflow correctly used the `cloudsmith-io/cloudsmith-cli-action` action to
do both in one step (using OIDC); that step was entirely absent here, so
`cloudsmith` would not exist on the runner and the push would fail.
**Fix:** Added an `Install Cloudsmith CLI` step using
`cloudsmith-io/cloudsmith-cli-action@v1.0.1` with `oidc-namespace` /
`oidc-service-slug`, matching the pattern already used in
`promote_package.yml`, immediately before the push step.

### 5. `promote_package.yml` — staging/production repo names swapped
**File:** `.github/workflows/promote_package.yml`
**Problem:**
```yaml
CLOUDSMITH_STAGING_REPO: 'production'
CLOUDSMITH_PRODUCTION_REPO: 'staging'
```
The variable names and their values were inverted. As written, the
"promote" step would search for the package in the repository literally
named `production` (not `staging`, where packages actually land after
`release_package.yml` runs) and, if found, move it into a repository named
`staging`. This is the opposite of the intended staging → production flow
and would either fail outright (package not found in `production`) or
silently move a production package back into staging.
**Fix:** Corrected the values so `CLOUDSMITH_STAGING_REPO: 'staging'` and
`CLOUDSMITH_PRODUCTION_REPO: 'production'`.

### 6. `README.md` — wrong workflow filename referenced
**File:** `README.md`
**Problem:** The README documents the three workflows, but names the
second one `realease_package.yml` (typo), which does not match the actual
file in the repo, `release_package.yml`. This is a real repo defect: the
documentation is the source of truth a maintainer uses to find the file,
and the typo'd name will not resolve, causing confusion when trying to
locate or reference the workflow.
**Fix:** Corrected the filename in `README.md` to `release_package.yml`.

---

## Part 2: `promote_package.yml` redesign

The original pipeline was triggered manually (`workflow_dispatch`) and
required a human to supply the package version to promote. Per the new
requirements, it is now event-driven end-to-end: a Cloudsmith webhook fires
the moment a package finishes synchronising in staging, which tags the
package and hands off to a promotion job that sweeps up everything tagged
for release.

### a) Manual trigger removed
The `workflow_dispatch` trigger (and its `package_version` input) has been
removed entirely.

### b) Webhook trigger from the staging repository
Replaced with:
```yaml
on:
  repository_dispatch:
    types: [package-synchronised]
```
`repository_dispatch` is the mechanism GitHub Actions provides for
triggering a workflow from an external system. Cloudsmith itself cannot
call this API natively with a single click, so the wiring is:

1. **Cloudsmith side** — a webhook is configured on the **staging**
   repository, subscribed to the **Package Synchronised** event, with
   *Payload Format* set to **Handlebars Template** so the outgoing body can
   be shaped to match GitHub's expected schema:
   - **URL:** `https://api.github.com/repos/<owner>/<repo>/dispatches`
   - **Headers:** `Authorization: Bearer <PAT>` (a token scoped to
     `contents: write` on this repo) and `Accept: application/vnd.github+json`
   - **Body:**
     ```json
     {
       "event_type": "package-synchronised",
       "client_payload": {
         "namespace": "{{ data.namespace }}",
         "repository": "{{ data.repository }}",
         "filename": "{{ data.filename }}",
         "version": "{{ data.version }}",
         "identifier_perm": "{{ data.identifier_perm }}"
       }
     }
     ```
   - Optionally, a package search query (e.g. `format:python`) can be
     attached to the webhook so it only fires for relevant packages.

2. **GitHub side** — the `repository_dispatch` event with
   `event_type: package-synchronised` triggers `promote_package.yml`, and
   the `client_payload` fields become available as
   `github.event.client_payload.*` inside the workflow (used to identify
   exactly which package just synchronised).

This entire setup (webhook creation, PAT, secrets) is configuration on the
Cloudsmith and GitHub sides, not something expressed inside the YAML file
itself — the YAML only needs to know how to react once the event arrives,
which is what the `on: repository_dispatch:` block above does.

### c) Tag the package `ready-for-production`
Added a new first job, `tag-ready-for-production`, which runs before
promotion and tags the package identified in the webhook payload:
```yaml
cloudsmith tags add \
  "${CLOUDSMITH_NAMESPACE}/${CLOUDSMITH_STAGING_REPO}/${IDENTIFIER}" \
  "${READY_TAG}"
```
where `IDENTIFIER` comes from `github.event.client_payload.identifier_perm`
(the `identifier_perm` Cloudsmith includes in the webhook payload for the
package that triggered the event) and `READY_TAG` is `ready-for-production`.
Tagging is done as its own step/job so that "eligible for production" is a
durable, inspectable property of the package in Cloudsmith itself, rather
than only living inside a single workflow run's memory.

### d) `PACKAGE_QUERY` changed to promote everything tagged `ready-for-production`
The `promote` job (which `needs: tag-ready-for-production`, so it always
runs after tagging) now queries by tag instead of by a single filename and
version:
```yaml
PACKAGE_QUERY="tag:${READY_TAG}"
PACKAGE_DATA=$(cloudsmith list package "${CLOUDSMITH_NAMESPACE}/${CLOUDSMITH_STAGING_REPO}" -q "$PACKAGE_QUERY" -F json)
```
It then iterates over **every** matching package (not just `.data[0]`) and
promotes each one with `cloudsmith mv --yes`:
```yaml
echo "$PACKAGE_DATA" | jq -r '.data[].identifier_perm' | while read -r IDENTIFIER; do
  cloudsmith mv --yes \
    "${CLOUDSMITH_NAMESPACE}/${CLOUDSMITH_STAGING_REPO}/${IDENTIFIER}" \
    "${CLOUDSMITH_PRODUCTION_REPO}"
done
```
Querying by tag (rather than by the single package that triggered this run)
means the pipeline is self-healing: if a previous run's promotion step
failed for any reason, the tag remains on that package and it will be
picked up and retried the next time any package synchronises, instead of
being silently stranded in staging.

### Full resulting flow
1. `build_package.yml` builds the package on push/PR to `main`.
2. `release_package.yml` pushes the built package to the Cloudsmith
   `staging` repository via OIDC.
3. Cloudsmith finishes synchronising the package and fires a
   **Package Synchronised** webhook.
4. The webhook calls GitHub's `repository_dispatch` API, triggering
   `promote_package.yml`.
5. `tag-ready-for-production` tags the new package `ready-for-production`.
6. `promote` queries staging for all `ready-for-production`-tagged
   packages and moves each one into the `production` repository.
