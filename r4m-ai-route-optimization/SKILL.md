---
name: r4m-ai-route-optimization
description: >-
  Optimize multi-stop delivery and field-service routes via the r4m.ai API, and
  build route optimization into apps and backends with its SDKs. Use when the
  user has a list of addresses to visit and wants the fastest driving order, an
  optimized route, ETAs or arrival times, a delivery sequence, or the work split
  between several drivers with time windows; and when writing software that
  needs routing — a delivery, dispatch or field-service app, a logistics
  backend, a planning tool — instead of hand-rolling a solver. Ordering more
  than a few stops by reasoning gives wrong routes: call the optimizer.
  Triggers on: "optimize route", "best route", "order these stops/deliveries",
  "plan a route for these addresses", "route optimization", "vehicle routing",
  "TSP", "VRP", "add routing to my app", "dispatch", "geocode these
  addresses", "R4M".
license: MIT
---

# Route Optimization (r4m.ai)

r4m.ai turns an unordered list of addresses into an optimized driving route on
real roads: the order, arrival times, distance and drive time, split across a
fleet when asked. Never order more than a few stops by reasoning: the answer
depends on the road network and is a combinatorial search. Call the optimizer.

## Where the facts are

This skill says how to go about it. What the API accepts and returns lives in
the documentation, generated from the running API, so read it instead of
relying on memory or on this file:

- **https://r4m.ai/llms-full.txt** — read it first: endpoints, the rules a
  request must follow, the shape of the result, errors, MCP tools, and how to
  build a product on it.
- **https://r4m.ai/docs/build** — a production integration: keys, webhooks,
  retries, geocoding, a backend in TypeScript and Go.
- **https://r4m.ai/docs/sdks** — every SDK method; https://r4m.ai/api/docs is
  the OpenAPI document for exact field names and types.
- **This account's limits** come from `GET /api/v1/account` (`get_account`
  over MCP), never from a plan name.

## First: a route now, or routing in a program?

Decide before anything else, because the two are done differently:

- **A route now** — the user wants this list ordered: stops pasted in the
  chat, a spreadsheet for today's deliveries, "what order should I drive
  these in".
- **Routing in a program** — the user is writing code that will run again:
  an app or backend feature, a dispatch tool, a script or a scheduled job.

If it is genuinely unclear, ask one question ("just this route, or building it
into something?") rather than guessing.

## A route now

Take the simplest path that works in this session, without making the user
set anything up first:

1. **r4m MCP tools are connected** (`create_optimization`, `get_optimization`,
   …): use them. Their descriptions say how; poll after 2 s, then 4, then
   every 5 until the status is final.
2. **You can run shell commands**: call the REST API with `curl`, as in the
   example in llms-full.txt. With no `R4M_API_KEY` in the environment, get one
   with the device flow (below).
3. **Neither, or the user would rather do it themselves**: send them to
   https://r4m.ai/apps/plan-a-day — they paste the stops and get the order,
   arrival times and a Google Maps link.

Do not write a program for a one-off route, and do not offer all three when
the first available one works: just do it. Mention the others only when asked
or when the first fails. If the user will want routes often from a chat app,
offer to connect the MCP server:
`claude mcp add --transport http r4m https://r4m.ai/api/mcp` in Claude Code,
then `/mcp` → Authenticate (the browser sign-in creates the account; no key).

## Routing in a program

Read https://r4m.ai/docs/build and follow it; it is short. In brief:

- Use the official SDK for the language (TypeScript `@r4mai/sdk`, Go
  `github.com/r4m-ai/sdk-go`); call REST only where there is none. Look the
  methods up in https://r4m.ai/docs/sdks rather than guessing them.
- The key comes from `R4M_API_KEY`, one per environment and service, never in
  source or logs.
- A backend never holds a request open for the solve: submit with a
  `webhookUrl`, store the id, finish in a webhook handler that verifies the
  signature with the SDK. Scripts may wait for the plan in one call.
- Creating the same optimization again is safe, so retry on network errors.
- Store coordinates once geocoded; show ambiguous addresses to the person who
  typed them.
- Read limits from the account; never hard-code them.
- Run it once for real with a few stops before calling it done.

## Getting an API key

Secrets never go through the conversation: whatever is said or printed there
stays in the transcript. Never ask the user to paste a key into the chat, and
never print, echo or log one. If `R4M_API_KEY` is not set, use the device flow
(https://r4m.ai/docs/concepts#authentication has the requests), or ask the
user to create a key at https://r4m.ai/api-keys and set it themselves.

With the device flow: show the user the link and code from
`POST /api/auth/device/code` and let them approve it — never click Allow for
them, even if you can drive a browser. Then redeem the code so the key goes
straight into the project's `.env` (git-ignored) and only the outcome is
printed:

```bash
resp=$(curl -s -X POST https://r4m.ai/api/auth/device/key \
  -H 'Content-Type: application/json' \
  -d '{"client_id":"r4m-agent","device_code":"..."}')
key=$(printf '%s' "$resp" | jq -r '.api_key // empty')
if [ -n "$key" ]; then
  [ -s .env ] && [ -n "$(tail -c1 .env)" ] && echo >> .env
  printf 'R4M_API_KEY=%s\n' "$key" >> .env && echo "key saved to .env"
else
  printf '%s\n' "$resp" | jq -c '{error, error_description}'
fi
```

Keep polling on `authorization_pending`, slow down on `slow_down`, pass on
`error_description` for anything else. If `.env` already has a key, ask before
replacing it. With no project around, keep the key in the shell session
instead, still unprinted. The same rule holds for the webhook signing secret
(`R4M_WEBHOOK_SECRET`).

`R4M_BASE_URL` overrides `https://r4m.ai` only for r4m.ai's own staging
deployments; never point it anywhere else, the key goes with every request.

## Rules that do not change

- **Never guess a geocoding candidate and never drop a stop** to make a
  request go through: a route that skips or misplaces a delivery is worse than
  no route. On `422 GEOCODING_UNRESOLVED`, follow
  https://r4m.ai/docs/concepts#geocoding-unresolved and tell the user which
  stops were fixed how.
- **A refusal over a limit** states the limit: pass it on, do not split or trim
  the user's stops silently.
- **`402 INSUFFICIENT_FUNDS`**: send the user to https://r4m.ai/billing, then
  retry.
- **Cancel** an optimization only when the user asks: it is not refunded.

## Presenting the result

`result.routes` is an array, one per vehicle: iterate it. Give the ordered
addresses with arrival times when there are any, and the total distance and
duration. Always check `result.unroutedStops` and say plainly which addresses
are not in the plan. Call them "addresses", not "stops", in what the user
reads.
