---
name: Sync an OpenAPI spec into an Apiable portal from CI
description: >-
  Push the current OpenAPI document into an Apiable portal's documentation entry on every deploy, so
  the published API reference never drifts from the shipped code.
api: openapi/apiable-platform-api-openapi.json
operations:
  - findAllPlans
  - findDocumentationByPlanId
  - findDocById
  - patchDocById
  - createDocByPlanIntegrationId_1
  - patchDocByPlanId_1
  - patchFullApiDocConfiguration
  - uploadFile
  - versions
---

# Sync an OpenAPI spec into an Apiable portal from CI

## Before you act

- **Scope.** This is the one flow Apiable scopes narrowly: `apiable/cicd` is sufficient for every
  operation here. Use it instead of `apiable/platform` — it is the least privilege that works.
- **Credentials.** Client id and secret, exchanged at `<portal-host>/api/oauth2/token`. Store them
  as pipeline secrets. Apiable's own GitHub Action takes them as `api_key` / `api_secret` plus an
  `api_url` naming your portal's API base.
- **You need a documentation id (`docid`)** to update, and its source must be set to
  *Continuous Delivery* in the portal. See
  https://www.apiable.io/docs/automation/ci-cd/.

## Steps

1. **Locate the documentation entry.** `findAllPlans` (`GET /api/plans`) then
   `findDocumentationByPlanId` (`GET /api/plans/{id}/docs/overview`) lists what a plan carries.
   `findDocById` (`GET /api/docs/{docId}`) reads one entry. Record the `docId` in your pipeline
   config rather than rediscovering it on every run.
2. **Push the spec.** `patchDocById` (`PATCH /api/docs/{docId}`) updates the entry the portal
   renders. For a plan-level or API-level entry that does not exist yet, create it with
   `createDocByPlanIntegrationId_1` (`POST /api/plans/{id}/docs`) and thereafter patch it with
   `patchDocByPlanId_1` (`PATCH /api/plans/{id}/docs/{docId}`).
3. **Full API reference.** `patchFullApiDocConfiguration` (`PATCH /api/docs/full-api-doc`) updates
   the portal's single combined API reference configuration.
4. **Files.** `uploadFile` (`POST /api/files/upload`) uploads a file. There is **no delete
   operation for an uploaded file** in the published contract — do not use it as scratch space.
5. **Health and version check.** `versions` (`GET /api/versions`) and `health`
   (`GET /api/health`) both accept `apiable/cicd` and are the cheapest way to prove your
   credentials and host are right before the pipeline does anything destructive.

## The ready-made action

Apiable publishes a composite GitHub Action that runs the whole flow:

```yaml
- uses: apiable/apiable-github-actions@v2
  with:
    api_key: ${{ secrets.APIABLE_CLIENT_ID }}
    api_secret: ${{ secrets.APIABLE_CLIENT_SECRET }}
    api_url: https://api.apiable.io
    open_api_spec_url: https://your-service.example.com/openapi.json
    docid: your-documentation-entry-id
```

**Verify the ref before you depend on it.** As of 2026-09-02 the public repository
`github.com/apiable/apiable-github-actions` has no tags and no releases — `@v2` and `@v1` do not
resolve on the public repo, only the `master` branch does. Pin to a commit SHA you have confirmed,
or call `patchDocById` directly, rather than trusting the documented `@v2` tag to exist.

## Errors

- **401** — token missing, expired, or without `apiable/cicd`.
- **404** — wrong `docId`, wrong plan id, or you are pointed at the wrong portal host.
- **400** — the uploaded document failed Apiable's spec validation. Apiable validates an uploaded
  spec as an API spec on upload, so a malformed OpenAPI fails here, not silently in the portal.
- There is no idempotency key and no rate-limit signalling. A patch is naturally idempotent in
  effect — the same spec pushed twice leaves the same entry — so a retry of step 2 is safe in a way
  that a retry of `createDoc*` is not.
