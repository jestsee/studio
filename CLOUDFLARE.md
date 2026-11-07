# Self-hosting on Cloudflare Workers

Use this fork with Workers (OpenNext), rather than a Pages deployment. The
locked adapter is `@opennextjs/cloudflare` 1.6.5 with Wrangler 4.24.4; Next.js pages and route handlers
must use the default Node.js runtime. Keep `nodejs_compat` and a compatibility
date of at least 2024-09-23 in `wrangler.jsonc`.

## Build and deploy

```sh
npm ci
npx opennextjs-cloudflare build
npx wrangler deploy --dry-run
npx wrangler login
npx opennextjs-cloudflare deploy
```

For Cloudflare Workers Git builds, connect **your fork**, select the branch
containing this fix, use `npx opennextjs-cloudflare build` as the build command
and `npx opennextjs-cloudflare deploy` as the deploy command. The root directory
is the repository root. The Worker name must match `outerbase-studio` in
`wrangler.jsonc` (or change both together). Do not point production builds at
the unchanged upstream repository.

## Cache configuration

No KV namespace is required for the initial D1 client deployment. The configured
OpenNext static-assets adapter reads build-time cache entries from the `ASSETS`
binding. Persistent runtime cache writes and ISR updates are unavailable
in this configuration. This app does not configure ISR or on-demand revalidation.

If persistent caching is needed later, create a namespace in your own account:

```sh
npx wrangler kv namespace create NEXT_INC_CACHE_KV
```

Switch `open-next.config.ts` to the `kv-incremental-cache` adapter and put the
returned namespace ID in the `kv_namespaces` array:

```json
{ "binding": "NEXT_INC_CACHE_KV", "id": "<your-namespace-id>" }
```

The binding name is required by the locked adapter. `NEXT_CACHE_WORKERS_KV`
does not match it. Never reuse the original repository's namespace ID. Full ISR
also requires the queue/tag-cache/self-service setup described in the
[OpenNext cache documentation](https://opennext.js.org/cloudflare/caching).

## Connect an existing D1 database

1. Open `/local`, then create a Cloudflare D1 connection (direct URL:
   `/local/new-base/cloudflare-d1`).
2. Enter the Cloudflare Account ID, D1 Database ID, and an API token with
   **Account > D1 > Edit**, restricted to the intended account. Use the
   Cloudflare dashboard to create the token and enter it directly in Studio.
3. Connect and run `SELECT 1 AS ok;` to verify access without changing data.

Studio stores local connections in the browser and forwards D1 queries through
`/proxy/d1` to Cloudflare's REST API. No D1 binding, database migration, server
database, OAuth app, or Worker secret is needed for this local-connection flow.
The token is forwarded on each query; treat the Studio origin and browser profile
as trusted. Do not commit tokens or put them in build variables or URLs.

Workspace/account features use the upstream Outerbase API by default; they are
separate from local D1 connections. Analytics is disabled unless explicitly
enabled with its configuration. For private use, configure Cloudflare Access
on the Worker hostname and allow only your identity.

## Free tier and verification

Start with Workers Free; this configuration provisions no KV, R2, or Durable
Objects and does not change your subscription. Check
[current Workers limits](https://developers.cloudflare.com/workers/platform/limits/)
and [D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/).
Workers Free allows 100,000 requests/day and 10 ms CPU/request. A successful
build or upload does not prove that every query and page fits the CPU budget.
Watch Worker metrics for CPU-limit failures (1102) during normal use.

After deployment, check `/local`, `/local/new-base/cloudflare-d1`, and
`/client/s/cloudflare-d1`. A POST to `/proxy/d1` without connection headers
should return HTTP 400 with a missing-account/database error. Then use a real
connection to run the read-only query above. That final test requires your D1
token in Studio, never in chat.
