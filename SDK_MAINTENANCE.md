# SDK Maintenance Guide

How to regenerate, update, and maintain the Parsec API SDKs.

This guide is written assuming you are working from the **main monorepo root**
(`parsecular-1/`) where the docs repo is checked out at `pc-documentation/`.

If you are working from the **docs repo directly** (`parsecular/docs`), drop the
`pc-documentation/` prefix in file paths.

## Overview

| Component | Location | Purpose |
|-----------|----------|---------|
| OpenAPI spec | `pc-documentation/openapi.yaml` | Source of truth for REST contract |
| Stainless config | `pc-documentation/stainless.yml` | Controls SDK generation (resources, naming, settings) |
| Spectral config | `pc-documentation/.spectral.yaml` | OpenAPI linting rules |
| TypeScript SDK (staging) | `stainless-sdks/parsec-api-typescript` | Stainless-managed staging repo |
| TypeScript SDK (production) | `parsecular/sdk-typescript` (GitHub) | Published to npm as `parsec-api` |
| Python SDK (staging) | `stainless-sdks/parsec-api-python` | Stainless-managed staging repo |
| Python SDK (production) | `parsecular/sdk-python` (GitHub) | Published to PyPI as `parsec_api` |
| Local clones | `stainless-sdks/sdk-typescript`, `stainless-sdks/sdk-python` | Production repo clones (gitignored) |

## How Stainless Branches Work

Stainless manages several branches in each SDK repo:

| Branch | Purpose | Who manages it |
|--------|---------|----------------|
| `generated` | Pure codegen output, no custom code | Stainless only (never touch) |
| `next` | Generated code + your custom patches merged | Stainless + you |
| `main` | Release branch | You (merge `next` when ready) |

**Key concept:** Stainless uses a semantic three-way merge. When you commit custom
code to `next`, Stainless re-applies your changes on top of each new codegen run
automatically. Your patches persist across regenerations — no cherry-picking needed.

**Never commit to:** `generated`, `codegen/**`, or `integrated/**` branches.

## Regenerating SDKs

After changing `openapi.yaml` or `stainless.yml`:

```bash
# 1. Lint the spec
cd pc-documentation && npx @stoplight/spectral-cli lint openapi.yaml && cd -

# 2. Regenerate TypeScript
stl builds create --project parsec-api \
  --openapi-spec pc-documentation/openapi.yaml \
  --stainless-config pc-documentation/stainless.yml \
  --target typescript --branch main

# 3. Regenerate Python
stl builds create --project parsec-api \
  --openapi-spec pc-documentation/openapi.yaml \
  --stainless-config pc-documentation/stainless.yml \
  --target python --branch main

# If "No changes to commit", rerun with: --allow-empty
# Builds run async. Check status:
#   stl builds retrieve --build-id <id>
```

Stainless pushes to `next` on the staging repo. Your custom patches on `next`
are preserved automatically.

## Custom Patches (Persist Automatically)

Custom patches live as commits on `next` of the **staging** repo. Stainless
preserves them through regenerations via semantic three-way merge.

If a merge conflict occurs, Stainless opens a PR on the staging repo. Resolve
it in GitHub — Stainless applies your resolution to future builds.

### TypeScript patches (`stainless-sdks/parsec-api-typescript`, branch `next`)

1. `src/resources/orderbook.ts` — `OrderbookLevel` tuple type, `bids`/`asks` typing
2. `src/resources/index.ts` — Re-export `OrderbookLevel`
3. `tests/contract.test.ts`, `tests/schema-validation.test.ts` — Live validation tests
4. `package.json` — `test:contract` script, dev deps, repo links to `parsecular/sdk-typescript`

### Python patches (`stainless-sdks/parsec-api-python`, branch `next`)

1. `src/parsec_api/types/orderbook_retrieve_response.py` — `OrderbookLevel` tuple type
2. `src/parsec_api/types/__init__.py` — Re-export `OrderbookLevel`
3. `pyproject.toml`, `README.md` — Repo links to `parsecular/sdk-python`

### Adding a new custom patch

```bash
cd stainless-sdks/parsec-api-typescript
git checkout next && git pull origin next
# Make changes...
git add -A && git commit -m "fix(orderbook): add OrderbookLevel tuple type"
git push origin next
```

Your commit persists through all future regenerations.

## Releasing to Production

After verifying `next` looks correct:

```bash
cd stainless-sdks/parsec-api-typescript
git checkout main && git pull origin main
git merge origin/next
git push origin main
```

Stainless syncs the staging `main` to the production repo (`parsecular/sdk-typescript`)
and creates release PRs there.

## Adding a New REST Endpoint

1. Implement the handler in `pc-api/src/handlers/`
2. Add the route in `pc-api/src/routes.rs`
3. Add the endpoint to `pc-documentation/openapi.yaml`
4. Add a Mintlify doc page in `pc-documentation/api/`
5. Add the page to `pc-documentation/docs.json` navigation
6. Add the resource/method mapping to `pc-documentation/stainless.yml`
7. Lint: `npx @stoplight/spectral-cli lint pc-documentation/openapi.yaml`
8. Regenerate both SDKs (see above)
9. Verify `next` branch has the new resource file
10. Merge `next` → `main` when ready to release

## Running Contract Tests

Contract tests validate SDK responses against the OpenAPI spec using a live server.

```bash
cd stainless-sdks/sdk-typescript
PARSEC_API_KEY=pk_... pnpm test:contract

# Requires: running local server at localhost:3000 (or set PARSEC_BASE_URL)
# Optional: validate against local OpenAPI file:
#   PARSEC_OPENAPI_SPEC=/absolute/path/to/openapi.yaml
```

## Publishing (When Ready)

1. Update `publish.npm` / `publish.pypi` to `true` in `stainless.yml`
2. Set version in each SDK's package manifest
3. Follow each SDK's publish workflow (see `prepublishOnly` in package.json)
