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

## Diff: deployed repo (0.38.5) vs this fork (0.53.0) `wrangler.jsonc`

Compared `cchesebro-prog/mcp4acumatica` @ `55e3ab1` with this fork's `wrangler.jsonc`:

- **Same:** worker name, compatibility settings, `ACUMATICA_URL` / `ACUMATICA_TENANT`
  placeholders, endpoint version `25.200.001`, `ACUMATICA_MAX_RECORDS`, both Durable
  Object bindings and migrations `v1`/`v2`, Logpush, and both R2 buckets. No new
  bindings or migrations are needed for 0.53.0 on that file.
- **KV ids:** the deployed repo has real ids for `TOKEN_STORE` and `OAUTH_KV` (two
  different namespaces). This fork has empty ids. Copy the two ids from the
  deployed repo's `wrangler.jsonc` or the deploy will provision new, empty namespaces
  and users will have to sign in again.
- **Renamed var:** the deployed repo sets `ACUMATICA_MCP_ROLE`. Upstream removed it
  and replaced it with `ACUMATICA_CANARY_GI` (default `MCPAccess`), per the
  CHANGELOG. The default means the old value can be dropped.
- **R2 preview buckets:** the deployed repo adds `preview_bucket_name` to both R2
  entries; upstream does not. Optional, only affects `wrangler dev`.
- **Real Acumatica URL and tenant are not in the repo.** The deployed repo also has
  placeholders, so the live values were set in the dashboard (not verified). If
  Workers Builds redeploys without `keep_vars`, those dashboard values may be
  overwritten by the placeholders.

## Wigwam edits already applied to `wrangler.jsonc`

- `TOKEN_STORE` and `OAUTH_KV` ids copied from the deployed repo. Both exist in the
  Cloudflare account (`mcp4acumatica-app`, `mcp4acumatica-oauth`), checked 2026-10-01.
- `"keep_vars": true` added.
- Nothing else changed: the Acumatica URL and tenant are still placeholders, so
  confirm the dashboard values exist before the first deploy from this repo.

These are the only intended divergences from upstream, so expect a conflict in this
file when a sync touches the same lines.

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
- `cchesebro-prog/mcp4acumatica` `wrangler.jsonc` @ `55e3ab1` (0.38.5), read 2026-10-01.
- Existing deployment facts (0.38.5, Cloudflare variable-override behavior) come
  from the 2026-10-01 handoff note and were not independently re-checked here.
