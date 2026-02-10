# SDK Maintenance Guide

How to regenerate, update, and maintain the Parsec API SDKs.

## Overview

| Component | Location | Purpose |
|-----------|----------|---------|
| OpenAPI spec | `pc-documentation/openapi.yaml` | Source of truth for REST contract |
| Stainless config | `pc-documentation/stainless.yml` | Controls SDK generation (resources, naming, settings) |
| Spectral config | `pc-documentation/.spectral.yaml` | OpenAPI linting rules |
| TypeScript SDK | `parsecular/sdk-typescript` (GitHub) | Published to npm as `parsec-api` |
| Python SDK | `parsecular/sdk-python` (GitHub) | Published to PyPI as `parsec_api` |
| Local clones | `stainless-sdks/sdk-typescript`, `stainless-sdks/sdk-python` | Gitignored in main repo |

## Regenerating SDKs

After changing `openapi.yaml` or `stainless.yml`:

```bash
# Lint the spec first (Spectral config lives in pc-documentation/.spectral.yaml)
cd pc-documentation && npx @stoplight/spectral-cli lint openapi.yaml
cd -

# Regenerate TypeScript
stl builds create --project parsec-api \
  --openapi-spec pc-documentation/openapi.yaml \
  --stainless-config pc-documentation/stainless.yml \
  --target typescript --wait

# Regenerate Python
stl builds create --project parsec-api \
  --openapi-spec pc-documentation/openapi.yaml \
  --stainless-config pc-documentation/stainless.yml \
  --target python --wait

# If you see: "No changes to commit", rerun with:
#   --allow-empty
```

## Custom Patches (Must Re-apply After Every Regen)

Stainless updates the generated SDK repos. The following patches are **manual changes on top
of codegen**, and must be re-applied after every regeneration.

### TypeScript (`stainless-sdks/sdk-typescript`)

1. `src/resources/orderbook.ts` — Add `OrderbookLevel` tuple type, update `bids`/`asks` typing
2. `src/resources/index.ts` — Re-export `OrderbookLevel`
3. `tests/contract.test.ts`, `tests/schema-validation.test.ts` — Live contract/schema validation tests (opt-in)
4. `package.json`, `pnpm-lock.yaml` — `test:contract` script + dev deps (`ajv`, `ajv-formats`, `@apidevtools/swagger-parser`)
5. `package.json`, `README.md`, `CONTRIBUTING.md` — Repo links should point at `parsecular/sdk-typescript`

### Python (`stainless-sdks/sdk-python`)

1. `src/parsec_api/types/orderbook_retrieve_response.py` — Add `OrderbookLevel` tuple type, update `bids`/`asks` typing
2. `src/parsec_api/types/__init__.py` — Re-export `OrderbookLevel`
3. `pyproject.toml`, `README.md`, `CONTRIBUTING.md` — Repo links should point at `parsecular/sdk-python`
4. `src/parsec_api/resources/*.py` — Docstring links should point at `parsecular/sdk-python`

### How to re-apply

Keep the patches as separate commits on `main`. After regen:

```bash
cd stainless-sdks/sdk-typescript
git log --oneline  # find the patch commit hash
git cherry-pick <hash>  # or manually re-apply if conflicts
```

TODO: Once the API stabilizes, replace manual patching with a post-gen script for deterministic regen.

## Adding a New REST Endpoint

1. Implement the handler in `pc-api/src/handlers/`
2. Add the endpoint to `pc-documentation/openapi.yaml`
3. Add the resource/method mapping to `pc-documentation/stainless.yml`
4. Lint: `npx @stoplight/spectral-cli lint pc-documentation/openapi.yaml`
5. Regenerate both SDKs (see above)
6. Re-apply custom patches
7. Commit and push all three repos (docs, ts sdk, python sdk)

## Running Contract Tests

Contract tests validate SDK responses against the OpenAPI spec using a live server.
They are opt-in and disabled by default.

```bash
# TypeScript
cd stainless-sdks/sdk-typescript
PARSEC_API_KEY=pk_... pnpm test:contract

# Requires: running local server at localhost:3000 (or set PARSEC_BASE_URL)
# Optional: validate against local OpenAPI file instead of the build artifact:
#   PARSEC_OPENAPI_SPEC=/absolute/path/to/openapi.yaml
```

## Publishing (When Ready)

1. Update `publish.npm` / `publish.pypi` to `true` in `stainless.yml`
2. Set version in each SDK's package manifest
3. Follow each SDK's publish workflow (see `prepublishOnly` in package.json)
