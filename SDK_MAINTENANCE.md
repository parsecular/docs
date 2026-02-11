# SDK Maintenance Guide

How to regenerate, update, and maintain the Parsec API SDKs.

This guide is written assuming you are working from the **main monorepo root**
(`parsecular-1/`) where the docs repo is checked out at `pc-documentation/`.

If you are working from the **docs repo directly** (`parsecular/docs`), drop the
`pc-documentation/` prefix in file paths.

---

## Overview

| Component | Location | Purpose |
|-----------|----------|---------|
| OpenAPI spec | `pc-documentation/openapi.yaml` | Source of truth for REST contract |
| Stainless config | `pc-documentation/stainless.yml` | Controls SDK generation (resources, naming, settings) |
| Spectral config | `pc-documentation/.spectral.yaml` | OpenAPI linting rules |
| TS SDK staging | `stainless-sdks/parsec-api-typescript` | Stainless-managed repo (commit custom patches here) |
| TS SDK production | `parsecular/sdk-typescript` (GitHub) | Published to npm as `parsec-api` |
| Python SDK staging | `stainless-sdks/parsec-api-python` | Stainless-managed repo (commit custom patches here) |
| Python SDK production | `parsecular/sdk-python` (GitHub) | Published to PyPI as `parsec_api` |
| TS local clone | `stainless-sdks/sdk-typescript` | Clone of production repo (dev/test here, gitignored) |
| Python local clone | `stainless-sdks/sdk-python` | Clone of production repo (dev/test here, gitignored) |

### Repo Naming Clarification

There are **two sets of repos** per SDK, which is confusing but necessary for the Stainless workflow:

- **`parsec-api-*`** (staging) — Stainless owns these. This is where codegen output lands and where your custom patches persist. You commit here to make changes survive regeneration.
- **`sdk-*`** (production clones) — These are local clones of the real `parsecular/sdk-*` GitHub repos. You develop and test here for convenience, then copy changes to the staging repos when ready.

The flow is: **develop in `sdk-*` → copy to `parsec-api-*` staging `next` → merge to staging `main` → Stainless syncs to production**.

---

## What's Generated vs Hand-Written

Understanding which files Stainless generates and which are hand-written is critical.
Stainless regenerates the REST client on every build. Hand-written code lives alongside
it as custom patches on the `next` branch.

### TypeScript SDK

| File | Generated? | Notes |
|------|-----------|-------|
| `src/client.ts` | **Generated** + patched | Stainless generates this. `ws()` method and `#deriveWsUrl()` are hand-written patches. |
| `src/streaming.ts` | **Hand-written** | Entire file is a custom patch. WebSocket client. |
| `src/index.ts` | **Generated** + patched | Stainless generates exports. Streaming re-exports are hand-written. |
| `src/resources/**` | **Generated** + patched | `orderbook.ts` has `OrderbookLevel` tuple type patch. |
| `tests/streaming.test.ts` | **Hand-written** | WebSocket tests. |
| `tests/contract.test.ts` | **Hand-written** | Live contract validation tests. |
| `tests/schema-validation.test.ts` | **Hand-written** | Schema validation tests. |
| `package.json` | **Generated** + patched | Stainless generates base. `isomorphic-ws`, `ws`, `@types/ws` deps are patches. |
| Everything else in `src/` | **Generated** | Don't edit — changes will be overwritten. |

### Python SDK

| File | Generated? | Notes |
|------|-----------|-------|
| `src/parsec_api/_client.py` | **Generated** + patched | Stainless generates this. `ws()` and `_derive_ws_url()` methods are hand-written patches. |
| `src/parsec_api/streaming.py` | **Hand-written** | Entire file is a custom patch. WebSocket client. |
| `src/parsec_api/__init__.py` | **Generated** + patched | Stainless generates exports. Streaming imports/exports are hand-written. |
| `src/parsec_api/types/**` | **Generated** + patched | `orderbook_retrieve_response.py` has `OrderbookLevel` tuple patch. |
| `tests/test_ws_streaming.py` | **Hand-written** | WebSocket tests. |
| `pyproject.toml` | **Generated** + patched | `websockets` dependency is a hand-written patch. |
| Everything else in `src/` | **Generated** | Don't edit — changes will be overwritten. |

### Merge Conflict Risk Areas

When Stainless regenerates, it three-way merges your patches on `next`. Most custom
code is in **new files** (`streaming.ts`, `streaming.py`, test files) which never
conflict. The risk areas are files that are **both generated and patched**:

- **`client.ts` / `_client.py`** — highest risk. The `ws()` method sits inside a
  Stainless-generated class. If Stainless changes the class structure, this may conflict.
- **`index.ts` / `__init__.py`** — medium risk. Export lists can shift.
- **`package.json` / `pyproject.toml`** — low risk. Dependency additions rarely conflict.

If a merge conflict occurs, Stainless opens a `main--merge-conflict` branch/PR on
the staging repo. Resolve it in GitHub. Stainless applies your resolution going forward.

---

## How Stainless Branches Work

Stainless manages several branches in each **staging** repo:

| Branch | Purpose | Who manages it |
|--------|---------|----------------|
| `generated` | Pure codegen output, no custom code | Stainless only (never touch) |
| `next` | Generated code + your custom patches merged | Stainless + you |
| `main` | Release branch | You (merge `next` when ready) |

**Key concept:** Stainless uses a semantic three-way merge. When you commit custom
code to `next`, Stainless re-applies your changes on top of each new codegen run
automatically. Your patches persist across regenerations — no cherry-picking needed.

**Never commit to:** `generated`, `codegen/**`, or `integrated/**` branches.

---

## Developing & Testing Custom Code (e.g. WebSocket Client)

Day-to-day development happens in the **production clones** (`sdk-*`), not the staging
repos, because they have the full project setup (node_modules, venv, etc.).

### 1. Develop in production clone

```bash
# TypeScript
cd stainless-sdks/sdk-typescript
# edit src/streaming.ts, tests/streaming.test.ts, etc.
pnpm install && pnpm test

# Python
cd stainless-sdks/sdk-python
# edit src/parsec_api/streaming.py, tests/test_ws_streaming.py, etc.
# Use the monorepo venv:
/path/to/parsecular-1/.venv/bin/python -m pytest tests/test_ws_streaming.py -v -o "addopts="
```

### 2. Copy changes to staging repo and commit on `next`

Once your code works, copy the changed files to the staging repo and commit.

> **⚠ Important:** Staging repos may have generated content that your production
> clone doesn't (e.g. new resources added via Stainless). For **new files**
> (`streaming.ts`, test files), wholesale copy is safe. For **generated + patched
> files** (`client.ts`, `_client.py`, `index.ts`, `__init__.py`, `package.json`,
> `pyproject.toml`), **diff and patch** rather than replacing — otherwise you may
> lose staging-only content.

```bash
# TypeScript example
cd stainless-sdks/parsec-api-typescript
git checkout next && git pull origin next

# New files — safe to copy wholesale:
cp ../sdk-typescript/src/streaming.ts src/streaming.ts
cp ../sdk-typescript/tests/streaming.test.ts tests/streaming.test.ts

# Generated+patched files — diff and apply manually:
# e.g. add ws() method to client.ts, add exports to index.ts,
# add deps to package.json. Don't overwrite the whole file.

git add -A && git commit -m "feat(ws): add WebSocket streaming client"
git push origin next
```

```bash
# Python example
cd stainless-sdks/parsec-api-python
git checkout next && git pull origin next

# New files — safe to copy wholesale:
cp ../sdk-python/src/parsec_api/streaming.py src/parsec_api/streaming.py
cp ../sdk-python/tests/test_ws_streaming.py tests/test_ws_streaming.py

# Generated+patched files — diff and apply manually:
# e.g. add ws() to _client.py, add imports to __init__.py,
# add websockets dep to pyproject.toml. Don't overwrite the whole file.

git add -A && git commit -m "feat(ws): add WebSocket streaming client"
git push origin next
```

### 3. Release to production

```bash
cd stainless-sdks/parsec-api-typescript
git checkout main && git pull origin main
git merge origin/next
git push origin main
```

Stainless syncs the staging `main` to the production repo (`parsecular/sdk-typescript`)
and creates release PRs there. Repeat for Python.

---

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

### After Regeneration Checklist

1. Pull latest `next` in the staging repo
2. Verify your custom patches are still present (check `streaming.ts` / `streaming.py` exist)
3. If Stainless opened a `main--merge-conflict` branch, resolve it on GitHub
4. Pull latest into your production clone (`sdk-*`) to pick up any generated changes
5. Run tests in the production clone to verify nothing broke
6. If all good: merge `next` → `main` on staging to release

---

## Custom Patches (Persist Automatically)

Custom patches live as commits on `next` of the **staging** repo. Stainless
preserves them through regenerations via semantic three-way merge.

If a merge conflict occurs, Stainless opens a PR on the staging repo. Resolve
it in GitHub — Stainless applies your resolution to future builds.

### TypeScript patches (`stainless-sdks/parsec-api-typescript`, branch `next`)

**REST patches:**
1. `src/resources/orderbook.ts` — `OrderbookLevel` tuple type, `bids`/`asks` typing
2. `src/resources/index.ts` — Re-export `OrderbookLevel`
3. `tests/contract.test.ts`, `tests/schema-validation.test.ts` — Live validation tests
4. `package.json` — `test:contract` script, dev deps, repo links to `parsecular/sdk-typescript`

**WebSocket patches:**
5. `src/streaming.ts` — Full WS client (new file, low conflict risk)
6. `src/client.ts` — `ws()` method + `#deriveWsUrl()` on `ParsecAPI` class
7. `src/index.ts` — Re-export streaming types
8. `package.json` — `isomorphic-ws`, `ws`, `@types/ws` dependencies
9. `tests/streaming.test.ts` — WS client tests with mock server (17 tests)

### Python patches (`stainless-sdks/parsec-api-python`, branch `next`)

**REST patches:**
1. `src/parsec_api/types/orderbook_retrieve_response.py` — `OrderbookLevel` tuple type
2. `src/parsec_api/types/__init__.py` — Re-export `OrderbookLevel`
3. `pyproject.toml`, `README.md` — Repo links to `parsecular/sdk-python`

**WebSocket patches:**
4. `src/parsec_api/streaming.py` — Full WS client (new file, low conflict risk)
5. `src/parsec_api/_client.py` — `ws()` method + `_derive_ws_url()` on both `ParsecAPI` and `AsyncParsecAPI`
6. `src/parsec_api/__init__.py` — Import/export streaming types
7. `pyproject.toml` — `websockets>=12.0, <15` dependency
8. `tests/test_ws_streaming.py` — WS client tests with mock server (16 tests)

### Adding a new custom patch

```bash
cd stainless-sdks/parsec-api-typescript
git checkout next && git pull origin next
# Make changes...
git add -A && git commit -m "fix(orderbook): add OrderbookLevel tuple type"
git push origin next
```

Your commit persists through all future regenerations.

---

## Running Tests

### TypeScript SDK

```bash
cd stainless-sdks/sdk-typescript

# All tests (unit + streaming)
pnpm test

# Streaming tests only
pnpm vitest tests/streaming.test.ts

# Contract tests (requires running server + API key)
PARSEC_API_KEY=pk_... pnpm test:contract

# Contract tests with local OpenAPI validation
PARSEC_API_KEY=pk_... PARSEC_OPENAPI_SPEC=/absolute/path/to/openapi.yaml pnpm test:contract
```

### Python SDK

```bash
cd stainless-sdks/sdk-python

# All tests (uses monorepo venv, override addopts to skip -n auto if pytest-xdist missing)
/path/to/.venv/bin/python -m pytest tests/ -v -o "addopts="

# Streaming tests only
/path/to/.venv/bin/python -m pytest tests/test_ws_streaming.py -v -o "addopts="

# If using the monorepo venv, install the SDK in editable mode first:
/path/to/.venv/bin/pip install -e stainless-sdks/sdk-python
/path/to/.venv/bin/pip install pytest pytest-asyncio websockets
```

---

## CI / Linting Requirements

Both staging repos run CI on every push to `main` (and `next`). You must pass
these checks before merging.

### TypeScript CI

| Check | Tool | Fix command |
|-------|------|-------------|
| Formatting | **prettier** | `npx prettier --write src/ tests/` |
| Linting | **eslint** | `npx eslint src/ tests/ --fix` |
| Type checking | **tsc** (strict mode) | Fix type errors manually |
| Lockfile | **pnpm** (frozen lockfile) | `pnpm install --no-frozen-lockfile` then commit `pnpm-lock.yaml` |

Common gotchas:
- CI runs `pnpm install --frozen-lockfile`. If you add deps to `package.json`
  without updating the lockfile, CI fails. Always run `pnpm install` and commit
  the updated lockfile.
- Prettier enforces the project's formatting rules. Run it before committing.

### Python CI

| Check | Tool | Fix command |
|-------|------|-------------|
| Formatting/linting | **ruff** | `ruff check --fix src/ tests/` |
| Type checking | **pyright** (strict mode) | Fix type errors manually |

Common gotchas:
- Ruff enforces import sorting (`I001`). After adding/changing imports, run
  `ruff check --fix` to re-sort.
- Pyright strict mode flags `list[Unknown]` after `isinstance` checks. Use
  `cast(List[Any], val)` to keep types as `Any`.
- Tests are excluded from pyright strict checking (`pyproject.toml` →
  `[tool.pyright]` → `exclude`). Decorator-registered callbacks (e.g.
  `@ws.on("orderbook") async def on_book(...)`) trigger false positives for
  `reportUnusedFunction`.

---

## Releasing to Production

After verifying `next` looks correct:

```bash
# TypeScript
cd stainless-sdks/parsec-api-typescript
git checkout main && git pull origin main
git merge origin/next
git push origin main

# Python
cd stainless-sdks/parsec-api-python
git checkout main && git pull origin main
git merge origin/next
git push origin main
```

Stainless syncs the staging `main` to the production repo (`parsecular/sdk-typescript`,
`parsecular/sdk-python`) and creates release PRs there.

---

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

## Modifying the WebSocket Protocol

1. Update `pc-api/src/ws.rs` (server)
2. Update SDK WS clients:
   - TypeScript: `stainless-sdks/sdk-typescript/src/streaming.ts`
   - Python: `stainless-sdks/sdk-python/src/parsec_api/streaming.py`
3. Update tests:
   - TypeScript: `stainless-sdks/sdk-typescript/tests/streaming.test.ts`
   - Python: `stainless-sdks/sdk-python/tests/test_ws_streaming.py`
4. Update `pc-documentation/websocket/overview.mdx`
5. Copy changes to staging `next` and push (see "Developing & Testing" above)
6. Bump SDK version appropriately

---

## Publishing (When Ready)

1. Update `publish.npm` / `publish.pypi` to `true` in `stainless.yml`
2. Set version in each SDK's package manifest
3. Follow each SDK's publish workflow (see `prepublishOnly` in package.json)

---

## End-to-End Example: Full Release Flow

Here's the complete flow from making a change to having it published:

```
1. Develop & test in production clone (sdk-*)
   ↓
2. Copy changed files to staging repo (parsec-api-*)
   ↓
3. Commit on staging `next` branch, push
   ↓
4. (If REST change) Regenerate via `stl builds create`
   → Stainless merges your patches on top of new codegen
   → Verify no merge conflicts
   ↓
5. Merge staging `next` → staging `main`, push
   ↓
6. Stainless syncs staging `main` → production repo (parsecular/sdk-*)
   → Creates a release PR on GitHub
   ↓
7. Review and merge the release PR
   ↓
8. Published to npm / PyPI (if publish is enabled in stainless.yml)
```
