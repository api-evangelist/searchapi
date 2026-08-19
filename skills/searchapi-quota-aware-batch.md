---
name: searchapi-quota-aware-batch
description: >-
  Run a large batch of SearchApi searches without tripping the undocumented hourly cap, by
  budgeting against the Account API and auditing the run with the Search Analytics API.
api: SearchApi Account & Analytics API
operations:
  - getAccount
  - getSearchAnalytics
  - search
generated: '2026-08-13'
method: generated
source:
  - openapi/searchapi-account-api-openapi.yml
  - rate-limits/searchapi-rate-limits.yml
  - plans/searchapi-plans-pricing.yml
---

# Quota-aware batch searching

Use this when you are about to issue hundreds or thousands of searches — rank tracking, price
monitoring, corpus building — rather than a single grounded lookup.

The problem this skill exists to solve: **SearchApi signals its rate limit only through a
polled endpoint.** There are no rate-limit response headers and no `Retry-After`. A naive
batch runner discovers the cap by failing against it, and the failure behaviour is
undocumented. So you poll first and pace yourself.

## Operations

- `getAccount` — `GET https://www.searchapi.io/api/v1/me`
- `getSearchAnalytics` — `GET https://www.searchapi.io/api/v1/search_analytics`
- `search` — `GET|POST https://www.searchapi.io/api/v1/search`

## 1. Establish the budget

```
GET /api/v1/me
Authorization: Bearer <API_KEY>
```

```json
{
  "account":      { "current_month_usage": 4800, "monthly_allowance": 10000, "remaining_credits": 5200 },
  "api_usage":    { "searches_this_hour": 250, "hourly_rate_limit": 200000 },
  "subscription": { "period_start": "2024-12-04T00:34:59Z", "period_end": "2025-01-04T00:34:59Z" }
}
```

Two ceilings apply at once:

- **Hourly:** `hourly_rate_limit - searches_this_hour`. The published rule is 20% of the
  plan's monthly credits per hour.
- **Monthly:** `remaining_credits`, resetting at `subscription.period_end`.

Take the smaller. If your batch exceeds the hourly ceiling, split it across hours — do not
retry into the wall.

## 2. Pace the run

Re-poll `getAccount` every few hundred searches rather than every search; it is an account
endpoint, not a search, and is not documented as consuming a credit, but it is still a
request. Recompute headroom from the fresh `searches_this_hour` instead of counting locally —
other keys and other processes on the same account draw from the same bucket.

Stop the batch when headroom drops below your safety margin. Resume at the top of the next
hour.

## 3. Use POST for deep pagination

Any batch that pages beyond the first couple of pages should use `POST /api/v1/search` with a
JSON body from the start. `next_page_token` grows to 8KB+ and will otherwise produce HTTP
413 or 414 partway through a run — the ugliest possible time to discover it.

## 4. Only successes cost

Billing is pay-per-success: HTTP 200 consumes a credit, failures do not. This means a run
that is failing is not burning quota, but it is burning wall clock. Watch the ratio.

## 5. Audit the run

```
GET /api/v1/search_analytics?engine=all&bucket=hourly&time_period=last_day
```

Returns `summary` (average speed, total searches, success rate, errors),
`performance_by_engine` (omitted when you filter to one engine), and `buckets` — per-interval
speeds, counts and error rates.

Read this after every large batch. It is the only place SearchApi shows you per-engine success
rate, and engines degrade independently: a marketplace engine can be failing while the Google
engines are healthy, and nothing in an individual response would tell you.

## 6. Compliance flag

On enterprise plans, adding `zero_retention=true` to search, account and analytics requests
disables logging and storage of the request. If your batch carries personal data in the query
string, set it — and note that turning it on removes the very analytics trail step 5 relies
on.
