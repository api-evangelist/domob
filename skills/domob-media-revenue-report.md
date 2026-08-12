---
name: domob-media-revenue-report
description: >-
  Pull Domob (多盟) publisher media statistics — ad requests, bids, impressions,
  clicks, CPM and media revenue — for a single day (hour-level) or a date range
  (day-level), optionally narrowed to one ad slot.
api: domob-media-data-api
operations:
  - getMediaStats
generated: '2026-08-12'
method: generated
source: >-
  openapi/domob-media-data-api-openapi.yml, conventions/domob-conventions.yml,
  errors/domob-problem-types.yml, authentication/domob-authentication.yml
---

# Domob media revenue report

Domob's entire public API contract is one operation. This skill is how to call
it correctly, and — more importantly — how not to be misled by it.

## Before you start

- **Base URL:** `https://developer.domob.cn`
- **Operation:** `getMediaStats` — `POST /developer/api/get/stats`
- **You need a real publisher account.** There is no sandbox, no test host and
  no test key for this API. See `sandbox/domob-sandbox.yml`.

## Read this first: the credential is a password

Domob has no token layer. `user_info.username` and `user_info.password` are the
developer-platform account's **email and login password**, sent in the JSON body
on every call. There is no scoped key, no expiry and no documented rotation.

Consequences you must design around:

- Never hand these values to an agent that logs its request bodies.
- Rotating access means changing the human's platform password.
- The `Token` request header is **not** a second credential. It is
  `base64(AES/CBC/PKCS7(slot_id + end_dt + start_dt))` computed with a key Domob
  prints in its own public PDF. Anyone can compute it. Treat it as a required
  format check, not as security.

## Steps

### 1. Build the `Token` header

Concatenate, in this exact order, with no separators:

```
slot_id + end_dt + start_dt
```

Use the empty string for `slot_id` when querying all slots. So a request for
2024-10-29 through 2024-10-30 across all slots signs the string
`2024103020241029`, and the same window scoped to slot `1234` signs
`12342024103020241029`.

Encrypt with AES/CBC/PKCS7 using the key published in Domob's Media Data API
PDF, then base64-encode the ciphertext. Domob publishes a worked Golang
implementation in section 2.2 of that document.

### 2. Choose your granularity by choosing your range

Granularity is **inferred, not requested**:

| Range | Result |
| --- | --- |
| `start_dt == end_dt` | hour-level rows, `day_type: "hr"` |
| `start_dt < end_dt` | day-level rows, `day_type: "dt"` |

Dates are integers in `YYYYMMDD` form — `20241029`, not `"2024-10-29"`.
**No timezone is documented anywhere**, so do not assume the day boundary
matches yours. If a total has to reconcile with another system, say so in your
output rather than asserting the numbers line up.

### 3. Call `getMediaStats`

```
POST https://developer.domob.cn/developer/api/get/stats
Content-Type: application/json
Token: <the value from step 1>

{
  "user_info": { "username": "<account email>", "password": "<account password>" },
  "export_info": { "slot_id": "", "start_dt": 20241029, "end_dt": 20241030 }
}
```

### 4. Branch on `code`, never on the HTTP status

**This API returns HTTP 200 for failures.** Verified live: an empty body returns
`200` with `{"code":1,"data":{},"msg":"邮箱或密码信息为空","sysTime":...}`.

```
if response.status != 200      -> transport/network failure
else if body.code != 0         -> API FAILURE — surface body.msg, do not treat as data
else                           -> success
```

Any generic HTTP client, retry wrapper or agent that trusts status codes will
record every error here as a success. Handle this explicitly.

Known codes: `0` = OK; `1` = email or password missing. Domob publishes no error
reference, so treat any other non-zero `code` as an unknown failure and pass
`msg` through verbatim rather than guessing at it. See
`errors/domob-problem-types.yml`.

### 5. Read the rows through the `dim` dictionary

Every response ships its own column dictionary in `data.dim` — each entry maps a
row field `label` to its Chinese display `name`. Use it to label output rather
than hardcoding translations; it is the one genuinely agent-friendly thing about
this payload.

Row fields: `day_type`, `summary` (the time bucket), `name` (exchange),
`media_ad_slot`, `req`, `bid`, `imp`, `clk`, `cpm_price`, `media_price`,
`application_id`.

`cpm_price` and `media_price` are **CNY (元)**. There is no currency field in the
payload and no currency parameter on this API — the `cur=usd` option belonged to
the older Reporting API, whose host no longer resolves.

## Constraints to respect

- **No pagination.** The whole window comes back in one `data.data` array, with
  no page, limit or cursor parameter and no total count. Bound your requests by
  narrowing the date range, not by paging.
- **No idempotency key**, but the operation is a read and is safe to repeat.
- **No rate-limit headers.** You get no back-pressure signal at all, and a
  throttle would most likely arrive as another HTTP 200. Pace yourself
  conservatively and do not retry tightly.
- **No request id.** Nothing in the response identifies the call, so log your own
  correlation id if you need to raise a support ticket at support@domob.cn.

## Do not treat these numbers as settlement

Domob's service agreement states plainly that the electronic statement is for
reference only and that binding settlement is the stamped statement issued by
多盟智胜网络技术（北京）有限公司. If you are reporting revenue, label it as
platform-reported, not as settled. See `plans/domob-plans-pricing.yml`.
