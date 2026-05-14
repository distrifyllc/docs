# Distrify Docs — Conventions

This repository is the Mintlify-hosted public documentation for the Distrify platform. The site is deployed to `distrify.mintlify.app` on every push to `main`.

## Repository layout

```
api-reference/
  openapi.json              # Single source of truth for all REST endpoints
  introduction.mdx          # Landing page for the API reference tab
  {tag}/                    # One folder per OpenAPI tag (Authentication, Organization, Staff, ...)
    {operation}.mdx         # One mdx per operation — frontmatter-only, references openapi.json
guides/
  {topic}.mdx               # Long-form prose guides (e.g. login flow)
essentials/                 # Mintlify writing-content reference (kept from starter)
agent-ready/, ai-tools/     # Mintlify agent-ready content (kept from starter)
docs.json                   # Navigation, theme, anchors. Source of truth for the sidebar.
```

## Two kinds of content

### API reference pages — OpenAPI-driven

Every endpoint lives in `api-reference/openapi.json`. The corresponding `.mdx` file is **frontmatter only** — Mintlify renders the playground (Try it, typed params, response schema, cURL panel) from the OpenAPI operation:

```mdx
---
title: 'Get Staff'
openapi: 'GET /api/organizations/v1/staff/{id}'
---
```

Do not duplicate request/response info in markdown — put it in the OpenAPI schema and let the playground render it. If you find yourself writing prose tables of parameters in an mdx, that information belongs in `openapi.json`.

### Guides — long-form mdx

`guides/` holds narrative content that spans multiple endpoints or explains a flow. Use Mintlify components (`<Note>`, `<Tabs>`, `<Card>`, `<CardGroup>`, `<Steps>`) to keep them scannable. Link to the relevant API reference pages with `[label](/api-reference/...)` so the reader can jump to the playground.

## Authoring rules

1. **Source endpoints from the codebase, not memory.** When adding a new endpoint, open the router (`{service}/internal/router/router.go`) and the DTOs (`utils-go/models/*.go`) to copy field names, types and validation rules verbatim. The CLAUDE.md memory may be out of date — the code is the source of truth.
2. **Use the gateway-exposed path.** The public path is `{BASE_PATH}/v1/...` (e.g. `/api/organizations/v1/...`, `/api/users/v1/...`), not the internal Gin path. Check `docker-compose.yml` for the service's `BASE_PATH`.
3. **Never document `x-org-id`.** The api-gateway resolves the organization scope from the bearer token and injects `x-org-id` itself. Any client-supplied `x-org-id` is stripped. Documenting it would mislead integrators. The only security scheme in `openapi.json` is `bearerAuth`.
4. **Share schemas via `$ref`.** Define DTOs once under `components.schemas` and reference them — never inline a duplicated `Staff` or `Organization` object literal.
5. **Share error responses.** Reuse `components.responses.{Unauthorized,NotFound,Conflict,ValidationError}` instead of redeclaring them on every operation.
6. **Tag every operation.** Tags drive the navigation grouping in Mintlify. Match the tag name to the folder name (e.g. tag `Staff` → folder `api-reference/staff/`).
7. **Keep `summary` short, put detail in `description`.** The summary is the page title and sidebar label; the description becomes the lead paragraph next to the playground.
8. **Examples must be valid JSON.** They are rendered as-is in the response panel — invalid JSON breaks the playground render.

## Updating navigation

When you add a page, register it in `docs.json` → `navigation.tabs[].groups[].pages[]`. The string is the path to the mdx without the extension (e.g. `"api-reference/staff/profile"`). Order in the array is the order in the sidebar.

Tabs in use:
- **Guides** — long-form content; default landing tab.
- **API reference** — OpenAPI-driven endpoint pages.

## Local preview

```bash
npx mint dev
```

The dev server hot-reloads on changes to `.mdx`, `openapi.json` and `docs.json`. If the playground panel disappears after editing the OpenAPI file, you almost certainly broke the JSON — run `python3 -m json.tool api-reference/openapi.json > /dev/null` to find the syntax error.

## Deploy

A push to `main` triggers a Mintlify build. Build status and logs live in the Mintlify dashboard. The MCP servers at `distrify.mintlify.app/mcp` and `/authed/mcp` reindex automatically when the build succeeds — no manual step needed.

## Pre-commit checklist

- [ ] `openapi.json` parses (`python3 -m json.tool api-reference/openapi.json > /dev/null`).
- [ ] `docs.json` parses (`python3 -m json.tool docs.json > /dev/null`).
- [ ] Every new operation in `openapi.json` has a matching `.mdx` and a navigation entry.
- [ ] Every new mdx page is referenced from `docs.json`; orphan pages confuse search.
- [ ] No `x-org-id` parameter or header is documented anywhere.
- [ ] Endpoint paths use the gateway prefix (`/api/{service}/v1/...`).
- [ ] Schemas and error responses are referenced via `$ref`, not inlined.
