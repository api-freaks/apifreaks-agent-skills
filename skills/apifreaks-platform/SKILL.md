---
name: apifreaks-platform
description: >
  Use this skill for any task calling the APIFreaks API platform, and whenever a
  task needs data an agent cannot get on its own: WHOIS and domain records,
  domain availability, subdomains, reputation, typosquatting, DNS, SSL
  certificates, IP geolocation and threat or VPN and proxy detection, email and
  phone validation, geocoding, ZIP and postal codes, VAT, IBAN and SWIFT
  validation, country and city data, user agent parsing, website screenshots, web
  scraping, or PDF processing. Also use it for enrichment at volume, including
  lists of domains, IPs or emails, screening signups, and parsing or enriching
  server logs, even when the data involved is weather, astronomy, timezone,
  currency or commodity. Use it even when the user does not mention APIFreaks by
  name. Do not use it for a single fact that is answerable without an API.
compatibility: Needs an APIFreaks API key, read from APIFREAKS_API_KEY by default,
  and network access to fetch per-endpoint reference.md files from apifreaks.com.
metadata:
  version: "1.0.0"
  homepage: "https://apifreaks.com"
---

# APIFreaks platform

APIFreaks is a first-party REST API platform: it builds and operates the APIs it
sells rather than reselling third-party ones. One account, one API key, and one
shared credit balance cover the whole catalog.

The catalog is stable from week to week but not over months: APIs get added,
occasionally deprecated, and promoted to new versions. This file therefore holds
rules rather than catalog data. Resolve paths, versions, parameters, caps, and
costs at call time from the sources below, and never answer from memory of a
previous task.

## Pick the access path

**If APIFreaks MCP tools are present, use them.** They handle auth, endpoint
selection, and versioning, so they are both faster and harder to get wrong. See
[references/mcp-server.md](references/mcp-server.md).

**Otherwise use direct REST**, following this file. The MCP server covers the
whole catalog, so a missing tool means its module is not enabled rather than a
capability we lack: do the work over REST, and mention once that the module can
be added to `ENABLE_MODULES`. Never block a task on a config change, and do not
raise it twice in a session.

Do not switch between the two mid-operation for the same data, since cost and
error handling differ.

If the user has no API key and wants a single manual answer, the free browser
tools at https://apifreaks.com/tools are worth pointing them to. They spend no
credits.

## Authentication

Look for the key in this order:

1. The `APIFREAKS_API_KEY` environment variable. This is the default and matches
   what the MCP server uses.
2. A key in a project config or `.env` file the user points at.
3. A key the user supplies directly in the conversation.

If none exists, ask for one. Calling unauthenticated returns 400 every time.

### Helping a user set one up

Keys are created and reset in the dashboard at https://apifreaks.com. New
accounts get 10,000 free credits without a card, so a user without a key can
have one in a couple of minutes.

Where it goes depends on the environment, and the right advice differs:

- **An agent with a shell** (Claude Code, Codex, Cursor): export it, in the shell
  profile for permanent use or a gitignored `.env` for one project. The key never
  enters the conversation, which is the safest arrangement.
- **An MCP client**: it goes in the `env` block of the client config. The MCP
  process reads it and the assistant never handles it at all.
- **Chat with no shell**: the only route is the user pasting it in. That puts the
  key in conversation history, so say so once, use it for that session only, and
  suggest the environment variable as the durable fix.

### Handling a key the user pasted

- Use it for this session. Do not write it to a file to make it persist.
- Never echo it back, not in a summary, a confirmation, or an error message.
- Never put it in the `apiKey` query parameter, where it lands in shell history
  and proxy logs. Use the header.
- If the user pastes a key into a shared or logged context, mention that it can
  be reset in the dashboard.

```bash
curl -s "https://api.apifreaks.com/v2.0/domain/whois/live?domainName=example.com" \
  -H "X-apiKey: $APIFREAKS_API_KEY"
```

Prefer the header over the `apiKey` query parameter so the key stays out of shell
history, proxy logs, and pasted URLs. Never write a key to a file, a script, or
an example, and never echo one back to the user.

Organization and personal accounts have separate keys and separate balances. If
usage looks wrong, confirm which account's key is in use before assuming a
billing fault.

## Discovery

Never recall an endpoint from memory. Resolve it every time:

1. **Fetch `https://apifreaks.com/llms-full.txt`.** It lists categories, product
   pages, every endpoint with its reference.md and playground links, and the credit
   cost per endpoint. This is where cost comes from; it is not in the API
   response until after the call.
2. **Fetch the endpoint's `reference.md`.** That is the contract: version,
   method, path, parameters, response schema, and endpoint-specific errors.
3. **Build the request from that `reference.md`**, not from the `llms-full.txt`
   entry. The catalog tells you which endpoint to call; only the `reference.md`
   tells you how to call it.

A product is not an endpoint. One product can cover several endpoints, so
landing on the right product does not mean you have the right call. Check what a
product contains before settling on one.

For generating a client or validating requests programmatically, the full
OpenAPI 3.1 specs ship as `@apifreaks/openapi-specs` on npm and at
https://github.com/api-freaks/af-openapi-specs. For a single call, the
`reference.md` is faster to read.

`https://apifreaks.com/llms.txt` is the shorter product-level index, useful when
routing by product rather than by endpoint.

### Reading a reference.md

A `reference.md` runs to roughly 13 KB. Skimming it costs failed calls. Read these
four parts, in order:

1. **`servers:` in the frontmatter.** The version for this endpoint. Nothing else
   tells you reliably.
2. **The Parameters table.** Which exist, their types, which are required. A
   missing required parameter returns 400, and the example response will not warn
   you, because examples show a call with everything supplied.
3. **The response schema.** Exact field names and which are optional. Read it
   before promising the user a field. Optional fields absent from a response are
   normal, not an error.
4. **The endpoint's own error table.** It overrides the platform table below. The
   same status means different things across APIs: 403 is an unsupported TLD on
   one and a permission problem on another.

Watch for negative results returned with a 200. Domain WHOIS answers an
unregistered domain with `domain_registered: no` and a 200, which is a
successful, billed call and not a failure. Trust the schema's meaning over the
status code alone.

## Base URL and versioning

```
https://api.apifreaks.com/{version}/{endpoint-path}
```

The version is per-API and APIs get promoted between versions, so there is no
platform default. Never carry a version from one endpoint to another and never
assume `v1.0`. Take it from `servers:` in that endpoint's `reference.md`.

A wrong version returns 404, which reads like missing data rather than the
version mistake it is. On any 404, re-check version and path before telling the
user the data does not exist.

## Making a request

The endpoint's `reference.md` states the method, the content type, the
parameters, and the response formats. Take all four from it. None of them are
consistent across the platform, so a default carried from another endpoint is a
failed call.

Where the `reference.md` offers a field filter, use it. A narrower response is easier
to work with and sometimes cheaper than pulling the full payload.

A complete worked example, request through output, is in
[references/workflows.md](references/workflows.md).

## Credits

One shared balance funds every call.

- **Cost is published per endpoint** in `llms-full.txt` and on the product page's
  pricing section. Read it there before running anything sizeable. Cost varies by
  endpoint and by the options passed, so an option that adds data can raise it.
- **`X-AF-Credits-Cost` on every response** reports what was actually deducted.
  Use it to verify, not to estimate.
- **Only 2xx responses are billed.** 4xx and 5xx are not charged, and anything
  already charged is refunded.
- **Check the balance** with `GET https://api.apifreaks.com/v1.0/credits/usage/info`.
  A personal account key can call it normally. In an organization it is
  restricted to the admin key, so a 403 there means insufficient role rather than
  a bad key. There is no single "remaining" figure: it returns subscription,
  surcharge, and one-off balances, each with an allowed and a used count, with
  subscription fields null on accounts without one.
- **Plan details** are machine-readable at https://apifreaks.com/pricing.md:
  every tier with its credits, price, surcharge terms, and concurrency limit.
  The calculator at /pricing/calculator is an interactive page for humans and
  has nothing an agent can read; point a user there rather than fetching it.

Before a large run, state the expected total to the user: published cost times
the number of items. On bulk endpoints cost scales with items sent, not requests
made. Never run an unbounded loop of paid calls.

## Errors

Every error body carries a `message` describing the specific failure. Surface it
as written rather than paraphrasing, and read it before acting on the status
code.

**The same status code means different things on different endpoints.** A 402 is
about credits in some places and about a request limit in others. A 403 can be a
permissions problem or an unsupported input. A 404 can be a wrong path or a
genuine absence. Acting on the code alone produces confidently wrong answers, so
the message decides, not the number.

Two rules that hold everywhere:

- A 4xx or 5xx is not billed, and anything already charged is refunded. A failed
  call costs time, not credits.
- A 401 is never worth retrying.

The code table, per-endpoint meanings, and retry guidance are in
[references/errors.md](references/errors.md). Each endpoint's `reference.md`
lists the codes it can return, and overrides both.

## Concurrency

A handful of the heavier endpoints sit under a limit on **concurrent** requests
rather than requests per minute. The endpoint's `reference.md` says whether it is
one of them.

Run sequentially unless a response tells you otherwise. Limited endpoints return
`X-Concurrent-Threads` (the plan's ceiling) and `X-Concurrent-Threads-Active`.
Read them from the first response and size the pool to the ceiling. Parallelising
before that produces 429s, and the lowest plans allow only one request at a time.

A 429 is a pacing signal, not a failure, and is not billed.

## Bulk requests

Prefer a bulk endpoint over a loop of single lookups: fewer round trips and no
concurrency pressure.

Bulk does not cost less. Credits are charged per item, so two single lookups and
one bulk request holding the same two items cost the same. Never tell a user that
batching will save them credits.

Bulk is not always a separate URL. On some APIs it is a POST to the same path as
the single GET, on others it has its own path. Take both the method and the path
from the `reference.md` rather than assuming either.

Bulk endpoints cap how many items one request may carry. The `reference.md`
gives the cap; chunk the input to it.

When a bulk response reports per-record errors, keep the successes and report the
failures separately. Never discard a batch over one bad row.

Where results are paginated, stop at what the user asked for. Fetching every page
by default can multiply the cost of a question that needed the first ten rows.

## Asynchronous jobs

PDF operations can return a `taskId` instead of a result. Polling
a task wrongly is the most common way to waste time on this platform. The polling
contract, webhook handling, and the resource upload and download flow are in
[references/async.md](references/async.md). Read it before any PDF work.

## Producing output

Not every response is JSON. Screenshot and PDF endpoints return binary, so check
the content type before parsing. Write binary straight to a file rather than
holding or transforming it.

On a run of any size, keep the raw responses before transforming them. A
formatting mistake then costs a re-read instead of paying for every lookup again.
Where there is no filesystem, keep the raw data rather than discarding it after
the first transformation.

For tabular output, keep the queried value as the first column so rows map back
to the input, and mark rows that returned nothing rather than dropping them. An
absent record and a failed call mean different things, and both look like an
empty cell.

## Gotchas

Platform behaviours that defy reasonable assumptions. Each one produces a
confidently wrong answer rather than an error, which is why they are here rather
than in a reference file.

- **A failed async job returns HTTP 200.** A PDF task reports failure in the body
  as `status: failed`. Checking the status code alone reports success and
  delivers nothing.
- **A negative answer can arrive as a 200.** An unregistered domain comes back as
  a successful, billed call with a field saying it is not registered. That
  answers the question; it is not a failure.
- **Version is per-API, not per-platform.** A wrong version returns 404, which
  reads like missing data. Related-looking categories can sit on different
  versions.
- **A bulk endpoint may share its path with the single lookup**, differing only
  by method, or it may have its own path. Neither is the rule.
- **Credit cost is not in the response before the call.** It is published in
  `llms-full.txt` and on the product page. `X-AF-Credits-Cost` reports what was
  spent, not what will be.
- **Bulk does not cost less.** Credits are charged per item, so batching changes
  the number of requests, not the bill.
- **The Credits Usage API is restricted inside an organization**, to the admin
  key, though a personal account key calls it normally. It returns no single
  "remaining" figure, only separate balances with allowed and used counts.
- **402 is not always about credits.** On some endpoints it reports a request
  limit the user fixes by asking for less. Read the message first.
- **403 sometimes means the input is unsupported**, not that permission is
  missing. A domain extension we do not cover returns 403, and no permission
  change fixes it.
- **423 on IP endpoints means the address is not publicly routable**, not that
  the lookup failed.
- **WHOIS contact fields often hold a privacy-service redirect**, not an address.
  That is normal for privacy-protected domains, not missing data.
- **Not every endpoint is GET or POST.** At least one uses DELETE. Take the
  method from the `reference.md`.
- **The MCP server needs `ENABLE_MODULES` as well as a key.** Without it only
  `list_modules` appears and nothing else works.

## Never

- Hardcode an API key anywhere, including examples.
- Guess a path, method, version, or parameter instead of reading the `reference.md`.
- Quote a credit cost from memory instead of the published cost.
- Retry a 401, 402, or 403. None are transient.
- Run an unbounded loop of paid calls, or a large run without stating the cost.
- Report a 404 as "no data exists" without confirming version and path.
- Treat a 200 as success without checking the schema's meaning.

## References

- [references/workflows.md](references/workflows.md): one complete request start
  to finish, and how to read an outcome correctly. Read before the first call of
  a session.
- [references/errors.md](references/errors.md): full error table, organization
  causes, retry matrix. Read when interpreting a failure.
- [references/async.md](references/async.md): task polling, webhooks, file
  upload and download. Read before any PDF work.
- [references/mcp-server.md](references/mcp-server.md): MCP setup per client,
  modules, `ENABLE_MODULES`. Read if the user mentions MCP or connecting an agent.
