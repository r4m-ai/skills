---
name: route-optimization
description: >-
  Optimize multi-stop delivery and field-service routes via the r4m.ai API.
  Use when the user has a list of addresses to visit and wants the fastest
  driving order, an optimized route, ETAs, or a delivery sequence. Triggers on:
  "optimize route", "best route", "order these stops/deliveries", "plan a route
  for these addresses", "route optimization", "shortest path between
  addresses", "R4M".
license: MIT
---

# Route Optimization (r4m.ai)

This skill turns an unordered list of addresses into an optimized driving route
via r4m.ai. Optimization runs asynchronously: you submit the addresses, get a
job id back immediately, then poll until it is done.

Three ways to call it, in order of preference:

1. **MCP tools**, if an `r4m` MCP server is connected in this session — the
   tools do the HTTP for you (see "Using MCP" below).
2. **The official SDK** (`@r4mai/sdk`) when writing code in a JavaScript or
   TypeScript project.
3. **Raw HTTP** in non-JS environments or for a quick check.

## Prerequisites

The user must have an r4m.ai API key (`acc_...`). If it is not already available
in the environment, ask for it and read it from `R4M_API_KEY`. Never hardcode or
log the key.

Base URL: `https://r4m.ai` (override with `R4M_BASE_URL` for self-hosted/preview).

Two hard requirements on the request:

- **Each address needs coordinates** (`lat`, `lng`) — the optimizer does NOT
  geocode plain strings. If you only have address text, geocode it first (see
  step 1 below).
- **At least 3 addresses** per optimization.

To pick the start/end point, set `isDepot: true` on one address (position in the
array does not matter), and `roundTrip: true` in the parameters if the driver
must return to the depot.

## Using MCP (when connected)

If the session has an r4m MCP server (tools named `create_optimization`,
`get_optimization`, `bulk_geocode`, `forward_geocode`, `list_optimizations`,
`check_usage`, `list_keys`, `create_key`, `request_topup`), call those
directly instead of writing code — every tool is an exact twin of the REST
endpoint of the same name, takes the same fields and returns the same
`data` payload:

1. `bulk_geocode` with `{ "addresses": [...strings] }` → one result per
   distinct address, matched by `original` (never by position). Use the ones
   with `status: "ok"`; an `ambiguous` result lists `candidates` (ask the
   user, or add a city/state to the string and retry), `not_found` needs a
   corrected address. Only `precision: "address"` is exact enough to drive
   to. Skip this step when coordinates are already known.
2. `create_optimization` with `{ "addresses": [{ address, lat, lng, isDepot? }] }`
   → `{ id, status: "pending", priceMicros }`.
3. `get_optimization` with `{ "id" }` every few seconds until `status` is
   `completed` or `failed`, then present `result.routes` as described under
   "Present the result" below.

`check_usage` shows the prepaid balance; a `402 INSUFFICIENT_FUNDS` from any
tool means it needs topping up (`request_topup` starts a machine payment
where that is enabled, otherwise point the user at the billing page).

If no r4m MCP server is connected but the user wants one, the endpoint is
`https://r4m.ai/api/mcp` (Streamable HTTP) with the API key as a Bearer
token — the same key as for REST. For Claude Code:

```bash
claude mcp add --transport http r4m https://r4m.ai/api/mcp \
  --header "Authorization: Bearer $R4M_API_KEY"
```

## Using the SDK (in code)

Install it with the project's package manager:

```bash
npm install @r4mai/sdk   # or: pnpm add @r4mai/sdk
```

Every endpoint is a typed function. Pass `baseUrl` and your raw `acc_…` key via
`auth` on each call (the client adds the `Bearer ` prefix itself).

The SDK's generated response type flattens the `{ success, data }` envelope, so
read the payload via a small cast — `(res.data as Envelope<T>).data` — exactly as
the result objects below do.

```ts
import { bulkGeocode, createOptimization, getOptimization } from "@r4mai/sdk";

const opts = {
  baseUrl: process.env.R4M_BASE_URL ?? "https://r4m.ai",
  auth: process.env.R4M_API_KEY,
};

// Helper: the wire format is `{ success, data }`; the generated type isn't, so
// unwrap with one cast. `T` is the payload shape you expect.
const unwrap = <T>(res: { data?: unknown }) =>
  (res.data as { data?: T } | undefined)?.data;

const stops = [
  "350 5th Ave, New York, NY",
  "11 Wall St, New York, NY",
  "1 World Trade Center, New York, NY",
];

// 1. Geocode the address strings into { lat, lng }.
const geo = await bulkGeocode({ ...opts, body: { addresses: stops } });
type GeocodeResult = { original: string; status: "ok" | "ambiguous" | "not_found"; lat?: number; lng?: number; precision?: string };
const results = unwrap<{ results: GeocodeResult[] }>(geo)?.results ?? [];
// One result per distinct address, matched by `original` — never by position.
// Anything but `ok` (ambiguous, not found) needs a better address string.
const addresses = results
  .filter((r) => r.status === "ok" && r.lat != null && r.lng != null)
  .map((r, i) => ({ address: r.original, lat: r.lat!, lng: r.lng!, isDepot: i === 0 }));

// 2. Submit (returns immediately with status "pending").
const created = await createOptimization({ ...opts, body: { addresses } });
const job0 = unwrap<{ id: string; status: string; priceMicros: number }>(created);
const id = job0!.id;

// 3. Poll until terminal (typically 2–60s). Back off a few seconds between polls.
let job = job0!;
while (job.status === "pending" || job.status === "processing") {
  await new Promise((r) => setTimeout(r, 3000));
  const polled = await getOptimization({ ...opts, path: { id } });
  job = unwrap<{ id: string; status: string; result?: unknown; errorMessage?: string }>(polled)!;
}

// 4. Present the result. `routes` is always an array — one entry per vehicle.
if (job.status === "completed") {
  const result = job.result as {
    routes: Array<{ totalDistanceMeters: number; totalTimeSeconds: number; waypoints: Array<{ index: number }> }>;
    unroutedStops: Array<{ index: number }>;
  };
  for (const route of result.routes) {
    console.log(route.waypoints.map((w) => w.index), route.totalDistanceMeters, route.totalTimeSeconds);
  }
  console.log(result.unroutedStops); // stops that could not be fitted at all
} else {
  console.error(job.errorMessage);
}
```

SDK response fields are camelCase (e.g. `priceMicros`, `errorMessage`).

## Using raw HTTP (fallback)

For non-JS environments. Endpoints return the envelope
`{ "success": boolean, "data" | "error" }`; `data` fields are camelCase.

1. **Geocode** address strings to coordinates (skip if you already have
   `lat`/`lng`). `POST /api/v1/geocoding/bulk`:

   ```bash
   curl -sS -X POST "$R4M_BASE_URL/api/v1/geocoding/bulk" \
     -H "Authorization: Bearer $R4M_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{ "addresses": ["350 5th Ave, New York, NY", "11 Wall St, New York, NY"] }'
   # → data.results[]: one per distinct address, matched by `original`:
   #   { original, status: "ok", address, lat, lng, precision, confidence }
   #   { original, status: "ambiguous", candidates: [...] }  — pick one or refine the string
   #   { original, status: "not_found" }
   ```

2. **Submit** with coordinates. `POST /api/v1/optimizations`:

   ```bash
   curl -sS -X POST "$R4M_BASE_URL/api/v1/optimizations" \
     -H "Authorization: Bearer $R4M_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "addresses": [
         { "address": "350 5th Ave, New York, NY", "lat": 40.7484, "lng": -73.9857, "isDepot": true },
         { "address": "11 Wall St, New York, NY",   "lat": 40.7069, "lng": -74.0113 },
         { "address": "1 World Trade Center, NY",   "lat": 40.7127, "lng": -74.0134 }
       ]
     }'
   ```

   A `202 Accepted` returns `{ "success": true, "data": { "id": "<uuid>", "status": "pending", "priceMicros": N } }`.
   Capture `data.id` (a UUID — no fixed prefix).

3. **Poll** `GET /api/v1/optimizations/$ID` until `status` is `completed` or
   `failed`. Back off a few seconds between polls; do not hammer the API.

   ```bash
   curl -sS "$R4M_BASE_URL/api/v1/optimizations/$ID" \
     -H "Authorization: Bearer $R4M_API_KEY"
   ```

4. **Present the result.** On `completed`, `data.result.routes` holds one entry
   per vehicle — always an array, with a single entry for an ordinary
   single-vehicle run, so iterate rather than assuming one. Each route carries
   `waypoints` in driving order (`index` points back at the request's
   `addresses`), plus `totalDistanceMeters` and `totalTimeSeconds`. Also check
   `data.result.unroutedStops`: stops listed there could not be fitted into any
   route and are NOT in the plan. On `failed`, surface `data.errorMessage`.

When showing the route to the user, use "Address" (not "Stop") in user-facing
text, and report the ordered list plus total distance/duration.

## Notes

- Every request is paid from a prepaid US-dollar balance and the price scales
  with address count. Amounts in the API are integer micro-dollars (1 USD =
  1 000 000) in fields named `…Micros`. It is not a monthly allowance: nothing
  resets. A `402 INSUFFICIENT_FUNDS` means the balance cannot cover the
  request — tell the user to top up.
- A `429` means "busy", and both kinds are handled the same way: wait for
  the seconds in the `Retry-After` header, then retry. `RATE_LIMITED` is the
  per-minute safety limit; `CONCURRENCY_LIMIT_EXCEEDED` means the plan's
  optimizations-in-flight allowance (Free 1, Pro 2) is used up, and
  `error.details.activeOptimizationIds` names the ones still running — poll
  one of them until it finishes, then retry. The SDK helpers
  (`submitOptimization`, `createOptimizationAndWaitForRoutes`) do this for
  you; prefer them over hand-rolled retry loops.
- `webhookUrl` is an alternative to polling, but it must be reachable from the
  public internet — a localhost URL is never delivered. To exercise webhooks
  against a server running on the user's machine, expose it with ngrok
  (`ngrok http <port>`) and pass the https URL ngrok prints.

See the full API reference at https://r4m.ai/docs.
