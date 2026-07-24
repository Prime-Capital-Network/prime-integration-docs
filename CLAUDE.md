# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repo is the Mintlify-powered API documentation site for **PrimeERP** (PrimeNet), a logistics-industry ERP SaaS. All content is written in **Spanish** — match that when adding or editing pages. There is no application code here; every file is either MDX content or Mintlify configuration (`docs.json`).

The API reference pages are hand-written (not generated from an OpenAPI spec), so new endpoints require manually authoring a page that follows the existing conventions below.

## Commands

```bash
npm i -g mint          # install the Mintlify CLI (one-time)
mint dev                # local preview at http://localhost:3000 (run from repo root, where docs.json lives)
mint broken-links       # check for broken internal links
mint a11y               # check for accessibility issues
mint validate           # validate the docs build
```

There is no test suite, linter, or build step beyond the `mint` CLI commands above.

## Architecture

### Navigation is centralized in `docs.json`

`docs.json` defines the entire site: theme, colors, and the full navigation tree (two tabs: **Documentation** and **API Reference**, each broken into groups). It is the single source of truth for what pages are published — **a new `.mdx` file is invisible until it's added to the `pages` array of the appropriate group in `docs.json`.**

### Content layout

- `documentation/` — conceptual/guide pages (introduction, auth, API key generation, rate limiting). Small, flat folder.
- `api-reference/` — one subfolder per resource (`associates`, `vehicles`, `containers`, `purchase-invoices`, etc.), each holding one `.mdx` file per operation.
- `api-reference/<resource>/snippets/` — reusable MDX fragments (e.g. shared `ResponseField` blocks for a nested "detail" object) imported/reused across sibling pages within that resource. Only `purchase-invoices` and `reimbursement-liabilities` currently have snippets; add one when a resource has near-identical nested response shapes repeated across multiple pages.

### Endpoint page file naming

Files are named by **Spanish verb + resource**, one file per HTTP operation, e.g. for a resource:
- `listar-<recurso>.mdx` — GET collection
- `extraer-<recurso>-por-id.mdx` — GET single by id
- `crear-<recurso>.mdx` — POST
- `actualizar-<recurso>.mdx` — PUT (full update)
- `actualizar-parcial-<recurso>.mdx` — PATCH (partial update)
- `eliminar-<recurso>.mdx` — DELETE
- Custom actions follow the same pattern, e.g. `cancelar-vehiculo.mdx` for `POST /vehicles/{id}/cancel`.

Some resources also have a nested "detail" sub-resource (e.g. `purchase-invoices/listar-detalles.mdx`, `crear-detalle.mdx`) following the same verb pattern.

### Standard endpoint page structure

Every operation page follows this shape (see `api-reference/customs/*.mdx` or `api-reference/vehicles/cancelar-vehiculo.mdx` as references):

1. Frontmatter: `title`, `sidebarTitle` (short Spanish word: "Listar", "Crear", "Actualizar", "Cancelar"), `description`, and `api: "METHOD /path"` (see "Interactive API playground" below — this replaces the old manual ` ```http ` route block, which should not be added to new pages).
2. One-line Spanish description of what the endpoint does.
3. `<Note>` stating the required scope, e.g. `Este endpoint requiere el scope **\`Associates.Write\`**.`
4. `### Headers` with `<ParamField header="X-Tenant-Id" type="string" required>` (every endpoint needs this — see below).
5. `### Path parameters` / `### Query parameters` / `### Body parameters` using `<ParamField path|query|body="..." type="..." required>`.
6. `### Example request` — a `<CodeGroup>` with three languages, always in this order: ` ```bash curl `, ` ```javascript JavaScript `, ` ```python Python `. Use realistic Dominican-business-style example data (real-looking names/addresses), not `foo`/`bar`. **Always include the resource `{id}` in the URL** for single-resource operations (PUT/PATCH/DELETE/GET-by-id) — a recurring bug found across this repo was example URLs missing the id segment entirely.
7. `### Example response` — a raw JSON example, or `### Response` prose for `204 No Content` responses.
8. `### Response fields` using `<ResponseField name="..." type="...">`, nesting objects with `<Expandable title="...">`. Never use `<ParamField>` here — that component is for request parameters only.
9. `### Errores` — one `<ResponseField name="<status> <Reason>">` per documented error, always including the `403 Forbidden` scope-missing case with the *correct* scope name for this resource (copy-pasting this block between resources is the most common source of wrong-scope-name bugs).
10. For write operations, a trailing `<RequestExample>` block with the raw JSON body (mirrors the curl example body) — this powers Mintlify's side-panel example.

### Interactive API playground

`docs.json` has a site-wide `api.mdx` config (`server: https://api.primenetcloud.com/v1`, `auth: { method: "key", name: "X-Api-Key" }`) that powers Mintlify's built-in "Try it" playground on every page that sets `api: "METHOD /path"` in its frontmatter — no OpenAPI spec needed, it reads the page's own `<ParamField>`/`<ResponseField>` components. Because auth here requires *two* headers and `docs.json` only configures the primary one (`X-Api-Key`), every page also declares the second one manually via `<ParamField header="X-Tenant-Id" type="string" required>` (see step 4 above) so the playground exposes both input fields.

Cross-field business rules (e.g. "`billToId` must reference an associate with `isCustomer: true` and `status.id: 2`") can't be expressed in this setup — they stay as prose bullets inside the relevant `<ParamField>`, exactly as before. The playground doesn't validate them; that's expected and doesn't need a workaround.

### Query filter conventions

List endpoints reuse a shared filter vocabulary documented once in `api-reference/filtros-avanzados.mdx`: `StringSearchFilter` (exact/`*prefix`/`suffix*`/`*contains*`/`_isnull`), `RangeSearchFilter` (`_min`/`_max`, non-inclusive), `BooleanSearchFilter`, `EnumSearchFilter`. Reference these type names in `<ParamField>` types rather than re-explaining the filter semantics on every page.

### Auth model (documented in `documentation/autenticacion.mdx`)

Every request requires two headers: `X-Api-Key` (format `pnk_{keyId}.{secret}`) and `X-Tenant-Id`. API keys carry scopes (e.g. `Associates.Read`, `Associates.Write`); missing scope → `403`, bad/missing/expired key or mismatched tenant → `401`.

## Writing conventions

This repo has the Mintlify docs skill installed (`.claude/skills/mintlify`, `.agents/skills/mintlify`, tracked via `skills-lock.json`) — consult it for general Mintlify component/config reference. Repo-specific conventions on top of that:

- All prose, headings, and field descriptions are in **Spanish**; code (variable names, JSON keys) stays in English/camelCase as returned by the actual API.
- Base URL used throughout examples: `https://api.primenetcloud.com/v1`.
- Every new endpoint page must be added to the correct group's `pages` array in `docs.json`, matching the existing group naming (Spanish, e.g. "Asociados", "Vehículos").
- Match an existing sibling page in the same resource folder for structure/tone before writing a new one — consistency across the ~14 resources matters more than any single page's polish.
