---
name: searchapi-grounded-web-research
description: >-
  Ground an answer in live web results using SearchApi's single search operation — pick the
  engine, page through results safely, and stay inside the account's hourly credit cap.
api: SearchApi SERP API
operations:
  - search
  - getAccount
generated: '2026-08-13'
method: generated
source:
  - openapi/searchapi-search-api-openapi.yml
  - openapi/searchapi-account-api-openapi.yml
  - conventions/searchapi-conventions.yml
  - rate-limits/searchapi-rate-limits.yml
  - errors/searchapi-problem-types.yml
---

# Grounded web research with SearchApi

SearchApi exposes exactly one search operation. Everything you can do with it is a choice of
`engine` plus parameters.

- `search` — `GET https://www.searchapi.io/api/v1/search` (POST accepted on the same path)
- `getAccount` — `GET https://www.searchapi.io/api/v1/me`

## 1. Authenticate

Send the key as `Authorization: Bearer <API_KEY>`. The `api_key` query parameter also works
but puts the credential in URLs, logs and browser history — prefer the header.

A missing or wrong key returns HTTP 401 with `{"error": "Invalid API key."}`. That is not
retryable; stop.

## 2. Check headroom before a burst

SearchApi returns **no rate-limit headers of any kind** — no `X-RateLimit-*`, no
`RateLimit-*`, no `Retry-After`. The only way to see your position against the cap is to
call `getAccount`:

```
GET /api/v1/me
-> { "account": { "remaining_credits": 5200, "monthly_allowance": 10000, ... },
     "api_usage": { "searches_this_hour": 250, "hourly_rate_limit": 200000 }, ... }
```

The published rule is that you may use **up to 20% of your plan's monthly credits each
hour**. Before issuing more than a handful of searches, call `getAccount` once and budget
against `hourly_rate_limit - searches_this_hour`. Do not discover the cap by hitting it: the
exhaustion status code and body are undocumented, so there is nothing reliable to branch on.

## 3. Run the search

Required parameters are `engine` and `q`.

```
GET /api/v1/search?engine=google&q=<query>&gl=us&hl=en
```

Useful parameters carried by the google engine: `location` or `uule` (mutually exclusive —
send one, never both), `gl` (country, default `us`), `hl` (interface language, default `en`),
`device` (`desktop` | `mobile` | `tablet`), `time_period`, `safe`, `filter`, `verbatim`,
`page`.

Pick the engine by the job, not by habit — `google` for general web, `google_news` for
recency, `google_scholar` for citable sources, `google_maps` for place data, `bing` for a
second opinion when you need corroboration from a different index. The full engine list is
in `mcp/searchapi-tool-crosswalk.yml`.

## 4. Read the response

Check `search_metadata.status` (`"Success"`) alongside the HTTP status. Then read
`organic_results` — each carries position, title, link and snippet. Depending on engine and
query the body may also contain `answer_box`, `knowledge_graph`, `local_results`, `ads`,
`shopping_ads` and `related_searches`.

**The response is polymorphic on the `engine` you asked for, and the body carries no type
tag.** Parse against the engine you requested; never sniff the shape.

Cite the `link` on each result you use. Do not present a snippet as fact without the link —
the snippet is Google's summary of a page, not the page.

## 5. Paginate carefully

Walk pages with `page`. Google web results are fixed at 10 per page — the `num` parameter was
phased out by Google in September 2025 and is now constant 10.

On deeper surfaces you will get a `next_page_token`. It **grows with depth** (2KB → 5KB →
8KB+), and once it is long enough your GET will fail with HTTP **413** (Request Entity Too
Large) or **414** (URI Too Long). The fix is not to retry — it is to switch transport:

```
POST /api/v1/search
Authorization: Bearer <API_KEY>
Content-Type: application/json

{ "engine": "google", "q": "<query>", "next_page_token": "<long token>" }
```

If you plan to page more than two or three deep, start with POST.

## 6. Budget

Billing is pay-per-success: only HTTP 200 responses consume a credit, failed requests do not.
Every page of results is a separate search and a separate credit. Decide how deep you need to
go before you start.

## 7. Errors

Errors are a flat `{"error": "<message>"}` JSON body — not RFC 9457 problem+json, no error
code, no documentation URI. Branch on HTTP status:

| Status | Meaning | Do |
| --- | --- | --- |
| 401 | Invalid API key | Stop. Not retryable. |
| 400 | Bad request | Check `engine`/`q` and engine-specific constraints (e.g. `location` vs `uule`). |
| 413 / 414 | `next_page_token` too long for a GET | Re-issue as POST with a JSON body. |

Capture `x-request-id` from the response headers on any failure — it is the only correlation
handle SearchApi gives you, and support will ask for it.
