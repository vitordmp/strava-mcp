# Strava MCP

A [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server that exposes Strava data to Claude via Cloudflare Workers + OAuth 2.1.

Forked from [kw510/strava-mcp](https://github.com/kw510/strava-mcp) with one addition: an `analyzeZoneDistribution` tool aimed at training-context analysis (Zone 2 verification, VO2 Max stimulus tracking, session character classification).

## What it does

- Exposes 38+ Strava API endpoints as MCP tools (activities, segments, routes, clubs, athlete data, streams, zones)
- Handles OAuth 2.1 end-to-end — connect once via the Claude.ai connector UI, no token copy/paste
- Runs on Cloudflare Workers free tier (well under the 100k req/day limit for personal use)

## Tools added in this fork

### `analyzeZoneDistribution`

Takes an activity ID. Pulls activity zones, HR streams, and activity metadata in parallel, then computes:

- **Time-in-zone breakdown** with bpm ranges, minutes, and percentages
- **VO2 stimulus minutes** — total time at Z4 + Z5 (the standard ~8–15 min/session target for VO2 Max adaptation)
- **Z2 percentage** — for verifying whether an "easy run" was actually easy
- **Session character classification**: `zone_2_dominant` / `threshold_work` / `high_intensity_vo2_stimulus` / `tempo_or_grey_zone` / `mixed`
- **HR stats** from the second-by-second stream (avg, max, sample count)
- **Plain-language interpretation** in the `notes` field

This is the layer the upstream repo doesn't have — it provides interpreted output rather than raw zone JSON.

## Deploy

### 1. Strava API credentials

1. Go to <https://www.strava.com/settings/api>
2. Create an application:
   - **Authorization Callback Domain**: `strava-mcp.<your-cloudflare-subdomain>.workers.dev`
   - **Authorization Callback URL**: `https://strava-mcp.<your-cloudflare-subdomain>.workers.dev/callback`
3. Note your **Client ID** and **Client Secret**

### 2. Cloudflare deployment

```bash
git clone https://github.com/<your-username>/strava-mcp.git
cd strava-mcp
npm install

# Create the OAuth KV namespace
npx wrangler kv namespace create OAUTH_KV
# Copy the returned ID into wrangler.jsonc → kv_namespaces[0].id

# Push your secrets
npx wrangler secret put STRAVA_CLIENT_ID
npx wrangler secret put STRAVA_CLIENT_SECRET

# Deploy
npx wrangler deploy
```

Wrangler will print the public URL. Use that as your callback domain in step 1 (you may need to update the Strava app after the first deploy if you didn't know the subdomain yet).

### 3. Connect to Claude

In Claude → Settings → Connectors → Add custom connector:

- **URL**: `https://strava-mcp.<your-subdomain>.workers.dev/sse`

Claude opens the Strava OAuth flow in a popup. Authorize, and the tools become available.

## Local dev

```bash
# Create a separate Strava app with localhost callbacks for development
# Authorization Callback Domain: localhost
# Authorization Callback URL: http://localhost:8788/callback

# Put dev credentials in .dev.vars
echo "STRAVA_CLIENT_ID=..." > .dev.vars
echo "STRAVA_CLIENT_SECRET=..." >> .dev.vars

npx wrangler dev
# Server runs at http://localhost:8788
```

## Strava API rate limits

- 200 requests / 15 min
- 2,000 requests / day

Plenty for personal use. The `analyzeZoneDistribution` tool issues 3 parallel requests per call (activity, zones, streams).

## License

MIT, inherited from the upstream repo.
