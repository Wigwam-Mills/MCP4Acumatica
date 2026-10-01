# Wigwam fork: staying in sync with upstream

This repo (`Wigwam-Mills/MCP4Acumatica`) is a GitHub fork of
`hallboys/MCP4Acumatica` (Apache-2.0). This file is Wigwam-only and does not
exist upstream, so syncing never conflicts with it.

## Rules that keep sync painless

1. **Do not edit upstream files on `main` unless you must.** Every local edit
   is a future merge conflict. `wrangler.jsonc` is the one file expected to
   differ (see below).
2. **Sync with the GitHub "Sync fork" button** on the repo's main page, or locally:
   ```
   git remote add upstream https://github.com/hallboys/MCP4Acumatica.git   # once
   git fetch upstream
   git merge upstream/main
   ```
3. **Read `CHANGELOG.md` before deploying a sync.** It lists breaking changes and
   Acumatica-side requirements (for example the `acumatica/` customization
   package and the `MCPGIs` / `MCPGIFields` feed GIs).
4. **Tags are not copied by Sync fork.** Releases upstream use the form
   `25R2-<version>`. To get them: `git fetch upstream --tags`. Pushing tags to
   this fork triggers `.github/workflows/release-on-tag.yml`, which calls a
   reusable workflow in `hallboys/.github`. Whether that workflow can run from
   this fork has not been tested; skip pushing tags unless you want a release object here.

## Deployment config (`wrangler.jsonc`)

Upstream's file ships placeholders: `ACUMATICA_URL`, `ACUMATICA_TENANT`, empty
KV ids, and so on. Real values must come from one of:

- **Cloudflare dashboard variables** (Workers & Pages > mcp4acumatica >
  Settings > Variables and Secrets), **plus** `"keep_vars": true` in
  `wrangler.jsonc` so a deploy does not overwrite them. Per Cloudflare's Wrangler
  docs (as noted in the 2026-10-01 handoff), `wrangler deploy` otherwise
  overrides dashboard-set variables. Secrets are not deleted by a deploy.
- **Real values committed in `wrangler.jsonc`** on a Wigwam-only branch that you
  merge upstream into. Never commit secrets (`ACUMATICA_CLIENT_ID`,
  `ACUMATICA_CLIENT_SECRET`, `COOKIE_ENCRYPTION_KEY`, `ADMIN_SECRET`).

Either way `keep_vars` or the values are a one-time edit to upstream's file, so
expect to resolve that one hunk when a sync touches it.

## What upstream's `wrangler.jsonc` requires (as of 0.53.0)

Compare each line with your existing worker before the first deploy from this repo:

| Item | Upstream 0.53.0 value |
|---|---|
| KV bindings | `TOKEN_STORE`, `OAUTH_KV` (may point at one shared namespace) |
| Durable Objects | `MCP_OBJECT` (`AcumaticaMcpServer`), `TOKEN_MANAGER` (`TokenManager`); migrations `v1`, `v2` |
| R2 buckets | `mcp4acumatica_logs` (`mcp4acumatica-logs`), `INDEX_STORE` (`mcp4acumatica-index`) |
| Logpush | `"logpush": true`, which upstream's comment says needs a Workers Paid plan |
| Endpoint | `ACUMATICA_ENDPOINT_VERSION` `25.200.001`, `ACUMATICA_ENDPOINT_NAME` `Default` |
| Other vars | `ACUMATICA_MAX_RECORDS` `1000`, `ACUMATICA_CANARY_GI` `MCPAccess` |

Keep the worker name and hostname unchanged, or update the Connected
Application redirect URI in Acumatica (SM303010).

## Cutting over the existing worker

The deployed worker `mcp4acumatica` was created from `cchesebro-prog/mcp4acumatica`
(`package.json` 0.38.5, not a GitHub fork). To move it onto this fork:

1. Record the current dashboard variable values and KV namespace ids.
2. Apply them per the section above.
3. In Cloudflare, reconnect Workers Builds for the worker to this repo and pick the production branch.
4. Merge, let it deploy, then run `/docs/admin/preflight` and its **Authenticated checks** form (needs `ADMIN_SECRET`).

## Sources and refresh dates

- Upstream `package.json` (0.53.0), `CHANGELOG.md`, `wrangler.jsonc`, and
  `.github/workflows/*` read from this fork's `main` at commit `fa2c2e3`, 2026-10-01.
- Existing deployment facts (0.38.5, Cloudflare variable-override behavior) come
  from the 2026-10-01 handoff note and were not independently re-checked here.
