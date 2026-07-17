# Account Hygiene Score API Documentation

**Last updated:** July 2026
**Audience:** Developers integrating with the TikTok for Business Ads API
**Keywords:** Account Hygiene Score, AHS API, TikTok Ads API, account optimization score, campaign suggestions, ad group configuration, creative refresh, PRS, AOS, CMS

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Who This Doc Is For](#2-who-this-doc-is-for)
- [3. API Reference](#3-api-reference)
  - [3.1 Endpoint](#31-endpoint)
  - [3.2 Authentication](#32-authentication)
  - [3.3 Request Parameters](#33-request-parameters)
  - [3.4 Response](#34-response)
  - [3.5 suggestion_details by type](#35-suggestion_details-by-type)
  - [3.6 Example request](#36-example-request)
  - [3.7 Example response](#37-example-response)
  - [3.8 Error codes](#38-error-codes)
  - [3.9 Permissions](#39-permissions)
- [4. Implementation Guide](#4-implementation-guide)
  - [4.1 Prerequisites](#41-prerequisites)
  - [4.2 Step 1: Confirm access and permissions](#42-step-1-confirm-access-and-permissions)
  - [4.3 Step 2: Build and send the request](#43-step-2-build-and-send-the-request)
  - [4.4 Step 3: Surface the account score](#44-step-3-surface-the-account-score)
  - [4.5 Step 4: Route suggestions by type](#45-step-4-route-suggestions-by-type)
  - [4.6 Step 5: Refresh on the right cadence](#46-step-5-refresh-on-the-right-cadence)
  - [4.7 Common Implementation Challenges](#47-common-implementation-challenges)
  - [4.8 Troubleshooting](#48-troubleshooting)
- [5. Follow up Questions](#5-follow-up-questions)

---

## 1. Overview

The Account Hygiene Score (AHS) API lets advertisers and their tooling pull a health score for one or more advertiser accounts, plus a set of concrete suggestions for improving that score. The score is built from three dimensions:

- **AOS (Ad group Optimization Score):** how well ad groups are configured
- **CMS (Creative Management Score):** how often creative assets are refreshed
- **PRS (Product Recommendation Score):** how much an account is using TikTok's in-house ad products (Cost Cap, Target CPA, OCPX, PSA, ADC, TTCX/TTCC)

Each suggestion returned by the API tells you what to change, how many entities are affected, and how many points you'd gain by acting on it. This doc covers the request/response contract for the endpoint and a step by step guide for wiring it into your app.

**Meta description for search indexing:** Reference and implementation guide for the Account Hygiene Score API, which returns advertiser account health scores and optimization suggestions for TikTok Ads accounts.

---

## 2. Who This Doc Is For

This is written for backend and full stack engineers building server side integrations against the TikTok for Business Ads API, whether that's an internal tool, an agency reporting dashboard, or a product feature that surfaces account health data to end users. Specifically this assumes:

- You're comfortable making authenticated REST calls and parsing JSON, you don't need basic HTTP explained
- You already have or know how to get a TikTok for Business developer account and app credentials, since that setup process is covered in TikTok's general Authentication docs, not repeated here
- You're the one deciding how `suggestion_details` gets deserialized and handled in your codebase, since a lot of the complexity in this API is in the fact that its shape changes depending on `suggestion_type`, and that's the kind of detail that trips people up if they skim instead of read
- You're likely building something that either surfaces this data in a UI, feeds it into an automated workflow, or both, and you need the response contract to be exact rather than approximate

---

## 3. API Reference

### 3.1 Endpoint

```
GET https://business-api.tiktok.com/open_api/v1.3/account/optimization/account/
```

> Note: the tech spec's permission scope section lists the path as `/open_api/v1.3/account/hygiene/account/`, which doesn't match the endpoint path defined earlier in the same doc (`.../account/optimization/account/`). Flag this to your backend team before you ship, since one of the two is likely a typo left over from an earlier naming pass. This documentation uses `optimization` as the canonical path since that's what's defined in section 2.2.1 of the spec, but confirm against the live environment before hardcoding it.

### 3.2 Authentication

| Header | Required | Type | Description |
|---|---|---|---|
| `Access-Token` | Yes | string | Your authorized access token. See the Authentication guide for how to generate one. |

### 3.3 Request Parameters

| Field | Required/Optional | Type | Description |
|---|---|---|---|
| `advertiser_ids` | Required | string[] | List of advertiser IDs to fetch scores for. Accepts 1 to 100 IDs per call. |
| `suggestion_types` | Optional | string[] | Filters which suggestion categories to return. If omitted, all types are returned. See the enum table below. |

#### suggestion_types enum

| Value | What it means | Downstream mapping |
|---|---|---|
| `PAUSE_ENTITY_WITH_BAD_PERFORMANCE` | Pause underperforming campaigns, ad groups, or ads | `CLOSE_WITH_BAD_PERFORMANCE` |
| `CREATE_MORE_ENTITY` | Create more ad groups or dedicated campaigns | `CREATE_WITH_MORE_QUANTITY` |
| `CREATE_AD_GROUP_WITH_GOOD_PERFORMANCE_CREATIVE` | Create new ad groups using creatives that already perform well | n/a |
| `UPDATE_BUDGET_TO_PASS_LEARNING_PHASE` | Create campaigns/ad groups with a suggested budget so they clear the learning phase | `CREATE_WITH_HIGH_BUDGET` |
| `UPDATE_BUDGET_FOR_BETTER_PERFORMANCE` | Update budget on existing campaigns/ad groups | `IMPROVE_WITH_RECOMMEND_BUDGET` |
| `CREATE_AD_WITH_NEW_CREATIVE` | Create ads using new creative assets | `REFRESH_CREATIVE_WITH_CREATE` |
| `ADOPT_COST_CAP` | Recommend Cost Cap bidding on more campaigns/ad groups | n/a |
| `ADOPT_TARGET_CPA` | Recommend Target CPA bidding on more campaigns/ad groups | n/a |
| `ADOPT_OCPX` | Recommend OCPX billing on more ad groups | n/a |
| `ADOPT_PSA` | Recommend creating more Product Shopping Ads campaigns | n/a |
| `ADOPT_ADC` | Recommend creating more Advanced Dedicated Campaigns | n/a |
| `ADOPT_TTCX_TTCC` | Recommend ads using TTCX or TTCC creatives | n/a |

### 3.4 Response

Top level fields:

| Field | Type | Description |
|---|---|---|
| `code` | number | Response status code |
| `message` | string | Response message |
| `request_id` | string | Unique log ID for this request, useful when filing a support ticket |
| `data` | object | Container for the actual payload |
| `data.result_list` | object[] | One entry per advertiser ID requested |

Each entry in `result_list`:

| Field | Type | Description |
|---|---|---|
| `advertiser_id` | string | Echoes back the advertiser ID from the request |
| `account_score_info` | object | Overall score for the account, see below |
| `suggestion_info_list` | object[] | List of suggestions, see below |

**`account_score_info` object**

| Field | Type | Description |
|---|---|---|
| `current_score` | number | The account's current optimization score |
| `full_score` | number | The maximum possible score for this account |
| `lift_score` | number | Total score you'd gain if every suggestion below were adopted |

**Each item in `suggestion_info_list`**

| Field | Type | Description |
|---|---|---|
| `suggestion_type` | string | One of the enum values from section 3.3 |
| `suggestion_score_info.current_score` | number | Current score for this specific suggestion (real time) |
| `suggestion_score_info.full_score` | number | Max score for this suggestion |
| `suggestion_score_info.lift_score` | number | Points gained if this suggestion is adopted |
| `suggestion_details` | object | Payload shape varies by `suggestion_type`, see 3.5 |

### 3.5 suggestion_details by type

The shape of `suggestion_details` changes depending on `suggestion_type`. Only the fields relevant to the returned type are populated.

**When `suggestion_type = PAUSE_ENTITY_WITH_BAD_PERFORMANCE`**

| Field | Type | Refresh cadence |
|---|---|---|
| `total_problematic_entity_count` | number | Daily |
| `total_need_pause_entity_count` | number | Real time |
| `total_paused_entity_count` | number | Real time |
| `problematic_campaign_count` | number | Daily |
| `problematic_adgroup_count` | number | Daily |
| `problematic_ad_count` | number | Daily |
| `need_pause_campaign_count` | number | Real time |
| `need_pause_adgroup_count` | number | Real time |
| `need_pause_ad_count` | number | Real time |
| `paused_campaign_count` | number | Real time |
| `paused_adgroup_count` | number | Real time |
| `paused_ad_count` | number | Real time |
| `need_pause_campaign_ids` | string[] | Real time |
| `need_pause_adgroup_ids` | string[] | Real time |
| `need_pause_ad_id_list` | string[] | Real time |

**When `suggestion_type = CREATE_MORE_ENTITY`**

| Field | Type | Notes |
|---|---|---|
| `create_adgroup_count` | number | Refers only to ad groups under non dedicated campaigns. Real time. |

**When `suggestion_type = CREATE_AD_GROUP_WITH_GOOD_PERFORMANCE_CREATIVE`**

| Field | Type | Notes |
|---|---|---|
| `material_id_list` | string[] | Daily |
| `good_performance_material_id_count` | number | Materials with good performance in the last 7 days. Daily. |
| `material_info_list` | object[] | Each item includes `campaign_id`, `adgroup_id`, `ad_id`, `video_id`, `identity_bc_id`, `identity_id`, `identity_type`, `tiktok_item_id`, and `material_id`. The identity fields only populate for Spark Ads; the rest populate for non Spark Ads. Daily. |

**When `suggestion_type = UPDATE_BUDGET_TO_PASS_LEARNING_PHASE` or `UPDATE_BUDGET_FOR_BETTER_PERFORMANCE`**

| Field | Type | Notes |
|---|---|---|
| `need_adjust_budget_ad_group_count` | number | Ad groups under non dedicated campaigns only. Real time. |
| `suggestion_budget_details` | object[] | Each item has `entity_id`, `entity_type` (`CAMPAIGN` or `ADGROUP`), `current_budget`, and `suggest_budget`. Budget values refresh daily. |

**When `suggestion_type = CREATE_AD_WITH_NEW_CREATIVE`**

| Field | Type | Notes |
|---|---|---|
| `created_ad_material_count` | number | Ads already created with new materials. Real time. |
| `need_create_ad_material_count` | number | Suggested number of additional ads to create. Real time. |

**When `suggestion_type = ADOPT_TARGET_CPA`**

| Field | Type | Notes |
|---|---|---|
| `suggested_increase_spending_amount` | number | Suggested spend increase if Target CPA is adopted. Daily. |

### 3.6 Example request

```bash
curl -X GET \
  'https://business-api.tiktok.com/open_api/v1.3/account/optimization/account/?advertiser_ids=["7012345678901234567"]&suggestion_types=["PAUSE_ENTITY_WITH_BAD_PERFORMANCE","ADOPT_TARGET_CPA"]' \
  -H 'Access-Token: YOUR_ACCESS_TOKEN'
```

### 3.7 Example response

```json
{
  "code": 0,
  "message": "OK",
  "request_id": "20260716120000010203040506070809",
  "data": {
    "result_list": [
      {
        "advertiser_id": "7012345678901234567",
        "account_score_info": {
          "current_score": 68,
          "full_score": 100,
          "lift_score": 32
        },
        "suggestion_info_list": [
          {
            "suggestion_type": "PAUSE_ENTITY_WITH_BAD_PERFORMANCE",
            "suggestion_score_info": {
              "current_score": 10,
              "full_score": 20,
              "lift_score": 10
            },
            "suggestion_details": {
              "total_problematic_entity_count": 14,
              "total_need_pause_entity_count": 6,
              "total_paused_entity_count": 8,
              "problematic_campaign_count": 2,
              "problematic_adgroup_count": 5,
              "problematic_ad_count": 7,
              "need_pause_campaign_count": 1,
              "need_pause_adgroup_count": 2,
              "need_pause_ad_count": 3,
              "paused_campaign_count": 1,
              "paused_adgroup_count": 3,
              "paused_ad_count": 4,
              "need_pause_campaign_ids": ["1798000000000001"],
              "need_pause_adgroup_ids": ["1798000000000011", "1798000000000012"],
              "need_pause_ad_id_list": ["1798000000000111", "1798000000000112", "1798000000000113"]
            }
          },
          {
            "suggestion_type": "ADOPT_TARGET_CPA",
            "suggestion_score_info": {
              "current_score": 5,
              "full_score": 15,
              "lift_score": 10
            },
            "suggestion_details": {
              "suggested_increase_spending_amount": 320.50
            }
          }
        ]
      }
    ]
  }
}
```

### 3.8 Error codes

| Code | Message | Meaning |
|---|---|---|
| `40002` | Invalid Params: Missing data for required field {}. | A required field (usually `advertiser_ids`) is missing or malformed |
| `50000` | Internal service error. | Something failed downstream, retry with backoff |

### 3.9 Permissions

To call this endpoint, the calling app or user needs the **Account Optimization** permission, which sits under the level 1 permission group **Ads Management** in the TikTok Business backend. If you get a permissions error, check that this permission has been granted at the advertiser account level, not just the app level.

---

## 4. Implementation Guide

This section walks through wiring the AHS API into a typical backend integration.

### 4.1 Prerequisites

- A registered TikTok for Business developer app with a valid `App ID` and `App Secret`
- An `Access-Token` generated through the standard OAuth flow for the advertiser accounts you want to query
- The **Account Optimization** permission enabled for those advertiser accounts (see section 3.9)
- One or more `advertiser_id` values you have access to

### 4.2 Step 1: Confirm access and permissions

Before you write any integration code, confirm the access token has the Account Optimization permission scoped to the advertiser IDs you plan to query. A token that works for other Ads Management endpoints will not automatically work here if this specific permission hasn't been granted.

### 4.3 Step 2: Build and send the request

```python
import requests

url = "https://business-api.tiktok.com/open_api/v1.3/account/optimization/account/"
headers = {"Access-Token": "YOUR_ACCESS_TOKEN"}
params = {
    "advertiser_ids": '["7012345678901234567"]',
    "suggestion_types": '["PAUSE_ENTITY_WITH_BAD_PERFORMANCE","CREATE_AD_WITH_NEW_CREATIVE"]'
}

response = requests.get(url, headers=headers, params=params)
data = response.json()

if data["code"] != 0:
    raise Exception(f"AHS API error {data['code']}: {data['message']}")

results = data["data"]["result_list"]
```

If you're calling this for many advertisers, batch them into the same request rather than looping one call per advertiser. The `advertiser_ids` param takes up to 100 IDs at once, and doing it this way cuts down on rate limit pressure.

### 4.4 Step 3: Surface the account score

```python
for result in results:
    score_info = result["account_score_info"]
    print(f"Advertiser {result['advertiser_id']}: "
          f"{score_info['current_score']}/{score_info['full_score']} "
          f"(+{score_info['lift_score']} available)")
```

A common pattern here is to show this as a progress bar or gauge in a dashboard, with the lift score displayed as "you could gain X more points."

### 4.5 Step 4: Route suggestions by type

Since `suggestion_details` changes shape based on `suggestion_type`, handle each type explicitly rather than trying to read a generic schema.

```python
def handle_suggestion(suggestion):
    s_type = suggestion["suggestion_type"]
    details = suggestion["suggestion_details"]

    if s_type == "PAUSE_ENTITY_WITH_BAD_PERFORMANCE":
        ad_ids = details.get("need_pause_ad_id_list", [])
        adgroup_ids = details.get("need_pause_adgroup_ids", [])
        campaign_ids = details.get("need_pause_campaign_ids", [])
        # queue these for pause, or show them in a review UI first
        return {"action": "review_pause", "ads": ad_ids, "adgroups": adgroup_ids, "campaigns": campaign_ids}

    elif s_type == "UPDATE_BUDGET_FOR_BETTER_PERFORMANCE" or s_type == "UPDATE_BUDGET_TO_PASS_LEARNING_PHASE":
        budget_changes = details.get("suggestion_budget_details", [])
        return {"action": "review_budget", "changes": budget_changes}

    elif s_type == "ADOPT_TARGET_CPA":
        return {"action": "show_recommendation", "spend_lift": details.get("suggested_increase_spending_amount")}

    else:
        return {"action": "generic", "raw": details}

for result in results:
    for suggestion in result["suggestion_info_list"]:
        action = handle_suggestion(suggestion)
        # send `action` to your review queue, notification system, or auto-apply pipeline
```

Do not auto apply pause or budget suggestions without a human review step in most cases. Treat this API as a recommendation engine, not an autonomous account manager, unless your product has explicitly opted the advertiser into auto apply behavior.

### 4.6 Step 5: Refresh on the right cadence

Not every field updates at the same frequency. Real time fields (like `need_pause_ad_id_list`) can be polled more often, while daily fields (like `problematic_campaign_count` or `material_id_list`) don't need to be re-fetched more than once a day. Polling daily fields too often just burns rate limit budget for no benefit.

A reasonable pattern:
- Poll the whole endpoint once daily for a full refresh
- If your UI shows a "pause now" action, re-check the specific real time fields right before the action is taken, to avoid acting on stale data

### 4.7 Common Implementation Challenges

These are the issues that tend to come up while building against this API, as opposed to the error code lookup table further down. Worth reading before you start, not just when something breaks.

**The endpoint path is inconsistent in the source spec.** Section 2.2.1 defines the endpoint as `.../account/optimization/account/`, but the permission scope section later in the same doc references `.../account/hygiene/account/`. This is a documentation bug, not an API quirk, but it's the single most likely thing to cost you time if you hardcode a path early and it turns out to be the wrong one. Confirm the live path with your platform contact or in a sandbox call before you build against it, don't assume section 2.2.1 is automatically the correct one just because it's listed first.

**`suggestion_details` isn't one schema, it's several schemas sharing a field name.** Because the same JSON key holds a different shape depending on `suggestion_type`, you can't deserialize this into a single fixed struct or type the way you might with most REST responses. Model it as a tagged union (or the equivalent in your language) keyed off `suggestion_type`, and write a small dispatcher like the one in section 4.5 instead of trying to force one schema to cover every case. Trying to flatten this into one object with a bunch of optional fields works but gets messy fast once you're handling more than two or three suggestion types.

**Real time fields and daily fields will disagree with each other if you're not careful.** Some fields under `suggestion_details` update in real time (like `need_pause_ad_id_list`) while others only update once a day (like `problematic_campaign_count`). It's normal to see a real time count that doesn't match a daily count exactly, that's not a bug, it just means something changed between refresh cycles. Don't build validation logic that assumes these numbers always reconcile, and don't page someone on call for a mismatch that's expected by design.

**Batching advertiser_ids is easy to get wrong the first time.** The parameter takes up to 100 advertiser IDs in a single call, but it's tempting to just loop one advertiser per request instead, especially if you're porting logic from an older single advertiser endpoint. Looping works but burns through rate limit budget fast if you're managing more than a handful of accounts. Batch requests up front, and page through your full advertiser list in chunks of 100 rather than one at a time.

**Pause and budget suggestions look actionable but shouldn't be auto executed without a review step.** It's tempting to wire `need_pause_ad_id_list` or `suggestion_budget_details` straight into a pause or budget update call and call the integration done. In practice, most teams need a human approval step or at minimum an audit log before these get applied, since acting on stale or misunderstood data here directly affects live ad spend. Build the review step in from the start rather than bolting it on after something gets paused that shouldn't have been.

**Permission errors can look identical to auth errors.** A missing Account Optimization permission and an expired or invalid token can both come back looking like a generic failure depending on how your error handling logs things. Log the actual `code` and `message` from the response body, not just the HTTP status, or you'll spend time debugging the wrong layer.

### 4.8 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `40002` on every call | Missing `advertiser_ids` param, or malformed JSON array in the query string | Double check the param is URL encoded and is valid JSON |
| Permission denied even with a valid token | Account Optimization permission not granted for that advertiser | Re-check permission at the advertiser account level in the backend |
| `suggestion_details` fields are all null | You're reading fields that don't apply to the returned `suggestion_type` | Only read the fields documented for that specific type in section 3.5 |
| Numbers change between two calls seconds apart | You're looking at a real time field, this is expected | Only alert on this if the change is unexpected relative to actions you took |
| `50000` errors | Downstream service issue | Retry with exponential backoff, and include `request_id` when contacting support |

---

## 5. Follow up Questions

### Q2. Evaluating the existing docs for RAG consumption

> *Original question: Evaluate the existing documentation on our website (https://business-api.tiktok.com/portal/docs?id=1797738007505921). How would you restructure or rewrite this material so that an AI agent or LLM (using Retrieval-Augmented Generation) could more accurately consume and retrieve this information for a user?*

I tried pulling the live page directly (`https://business-api.tiktok.com/portal/docs?id=1797738007505921`) and it's rendered client side, so the actual doc content doesn't come through in a plain fetch, only the page shell and meta tags. That itself is worth flagging as an RAG problem: if a retrieval pipeline can't render JS, it never sees the content at all. Based on the structure of the spec doc I was given for this exercise, which mirrors how TikTok's business API docs are typically laid out, here's how I'd restructure it for RAG consumption.

The biggest issue with docs like this one is that they're built as giant nested tables meant for a human to scan visually. A person can follow the indentation and "only return when suggestion_type = X" notes because they can see the whole table at once. An LLM chunking this content for embeddings usually cannot, since chunk boundaries cut through the middle of a table and the model loses the parent context (which suggestion_type a given field belongs to) by the time it retrieves that chunk.

A few concrete changes I'd make, split into two categories.

#### Category A: Restructure the existing documentation conventions

These are changes to how the current TikTok Business API docs site itself is written and organized, no new deliverable required, just a different way of structuring the same content.

1. **Break the one giant response table into one table per suggestion_type.** Right now `suggestion_details` is one sprawling table with a `Only return when suggestion_type = X` note buried in the last column. If each suggestion_type gets its own self contained section with its own heading and its own table, a retrieval system can pull just that section and have full context, without needing the surrounding table rows to know what applies.

2. **Repeat context instead of relying on inheritance.** Human readers infer that `need_pause_ad_id_list` belongs to `PAUSE_ENTITY_WITH_BAD_PERFORMANCE` because it's in the same table block. An LLM chunk retrieved in isolation needs that stated directly, something like "this field is part of the PAUSE_ENTITY_WITH_BAD_PERFORMANCE suggestion type" written into the row or the section heading itself, not just implied by position.

3. **Use heading hierarchy that matches how developers actually search.** Right now the field descriptions live several levels deep in a table with abbreviated source column references (like `Task.params`) that mean nothing without internal context. I'd add descriptive H2/H3 headers per endpoint, per suggestion_type, and per field group, since most RAG chunking strategies split on headings, and a good split point produces a self contained, retrievable unit.

4. **Turn embedded enums into their own reference sections with anchors.** The `suggestion_types` enum is currently embedded inline inside the request parameter description. Pulling it into its own section with a stable anchor lets it get retrieved and cited on its own when someone asks "what suggestion types exist" without needing to retrieve the entire endpoint doc.

5. **Add a short plain language summary above every table.** One or two sentences in prose, ahead of each table, gives the embedding model a stronger semantic signal than a table row alone. Tables are hard to embed well since the semantic meaning is spread across column and row, which a plain language sentence restates in a way that embeds cleanly.

6. **Keep example requests and responses next to the field docs they illustrate, not in a separate examples page.** If they're separated, a retrieval system might return the field table without the example, or the example without knowing which fields it's demonstrating.

7. **Server side render or provide a static docs mirror.** This is table stakes but worth stating given what I ran into just now. If the canonical docs are JS rendered only, no crawler based RAG pipeline sees them, full stop.

#### Category B: Publish a separate machine readable documentation surface for AI agents

This is a different kind of change from points 1 through 7. Instead of restructuring the existing page, it's a new deliverable that sits alongside the human facing site.

**What it is.** The pattern that's forming around this is a convention called `llms.txt`, a plain markdown file sitting at the root of a docs domain that works something like a sitemap, except instead of pointing search engines at pages, it points AI systems at the canonical machine readable content, stripped of navigation, CSS, and JavaScript. Some teams also publish an `llms-full.txt` with the expanded version of the content inline rather than just links.

**Why it's a real production pattern, not just a proposal.** It's not a formal standard yet, more of a widely adopted convention, but it's already showing up in production. Documentation platforms like Mintlify, GitBook, and Fern now support generating it directly rather than requiring teams to hand roll it, and companies like Anthropic, Stripe, and Cursor already publish one. This isn't a niche idea either. Traffic data from a few docs platforms puts a meaningful share, in some reports close to half, of documentation page requests as coming from AI agents rather than human browsers, especially for API first products. Treating the AI reader as a second audience with its own dedicated surface, rather than just hoping a well structured human page also happens to work for retrieval, reflects where a lot of traffic already is.

**How I'd apply it here.** For this endpoint specifically:

- Keep the human facing reference doc as is
- Generate a machine readable export of the same content that applies the restructuring from points 1 through 6 above, one section per suggestion_type, context stated explicitly rather than implied by table position, examples kept next to the fields they illustrate
- Add a short `llms.txt` index that points to both versions
- Generate both versions from the same source of truth, ideally the OpenAPI spec or the tech spec doc itself, in the same build pipeline, so a change to the API doesn't update one version and silently leave the other stale
- Keep in mind that AI tools consuming these files tend to cache them, so a stale machine readable version can keep producing wrong suggestions for a while before anyone notices, which is another reason to generate it automatically rather than hand maintain it

### Q3. Documenting AI features when output isn't guaranteed

> *Original question: Traditional software and APIs return predictable responses, but AI-powered tools often generate varied or unpredictable outputs. How do you approach writing documentation, tutorials, and code samples for an AI feature when the exact result cannot be guaranteed?*

The main shift is moving away from "here's exactly what you'll get" and toward "here's the shape of what you'll get, and here's how to handle variation programmatically." A few things I lean on:

- **Show a range of example outputs, not just one.** One example implies determinism even if you don't mean it to. Showing two or three varied but valid outputs for the same input signals to the reader that variation is expected and normal, not a bug they need to chase down.
- **Document the schema strictly even when the content inside it varies.** If the field names, types, and structure are guaranteed even though the values aren't, say that explicitly. Developers can build reliable code against a stable schema even if the text inside a field is non deterministic.
- **Write code samples that validate and handle edge cases, not just the happy path.** For example, if an LLM powered suggestion field could come back empty, truncated, or in an unexpected format, the sample code should show a check for that instead of assuming the ideal case every time.
- **Separate "guaranteed" fields from "generated" fields visibly in the docs.** Something like a small badge or note next to fields whose values come from a model versus fields that are deterministic counts or IDs. This sets the right expectation at a glance.
- **Avoid over specifying exact wording in tutorials.** If a tutorial shows a exact expected model output as if it's the only correct one, readers will file a bug when they get something slightly different. Better to describe the properties a good output should have (tone, length, what it must include) rather than a single frozen example they'll expect to match exactly.
- **Call out non determinism up front, not as a footnote.** If retries with the same input can produce different results, that belongs in the first paragraph of the relevant section, not buried at the bottom.

### Q4. A workflow for scaling documentation with AI tools

> *Original question: Describe a specific workflow where you would use AI tools (e.g., LLMs, custom scripts) to automate or scale the creation of technical documentation. What are the risks of this approach, and how do you mitigate them?*

One workflow I'd actually use: pulling structured API specs (OpenAPI/Swagger, or in this case a tech spec doc like the one used for this exercise) and using an LLM to draft the first pass of reference docs, tables, and code samples, with a human doing structured review before publish rather than free form editing from scratch.

Rough steps:
1. Extract the source of truth (OpenAPI spec, tech spec doc, or even a live API's response schema) into a clean structured format.
2. Prompt an LLM with that structured input plus a fixed style guide and template, so every generated page follows the same voice, heading structure, and example format instead of drifting page to page.
3. Auto generate: field tables, one example request and response per endpoint, and a first draft implementation guide.
4. Run generated code samples against a sandbox or mock server to confirm they actually execute and the example output matches the schema, not just that it reads well.
5. Human review pass focused specifically on accuracy of business logic and edge cases the model wouldn't know about (like the mismatched endpoint path I flagged in section 3.1 of this doc), not on prose polish.
6. Publish, then track which pages generate the most support tickets or "was this helpful: no" clicks, and feed that back into the next generation pass.

Risks and how I'd mitigate them:

- **Hallucinated fields or behavior.** The model can invent a field or parameter that sounds plausible but doesn't exist. Mitigation: never let generated docs publish without diffing them against the actual spec or a live API response, ideally with an automated check, not just a human skim.
- **Stale docs that look confidently correct.** Generated prose reads polished even when it's wrong, which makes errors harder to catch than in obviously rough draft. Mitigation: tie doc generation to the same CI pipeline as the API itself, so a spec change triggers a re-generation and a diff review, instead of docs drifting silently out of sync.
- **Voice inconsistency across a large doc set.** Without a locked style guide in the prompt, tone drifts between generation runs. Mitigation: keep the style guide and templates versioned and reused for every run, don't regenerate the prompt from memory each time.
- **Over reliance replacing subject matter expert review.** It's tempting to publish straight from the model once it "looks right." Mitigation: keep a required human review step in the pipeline, specifically scoped to accuracy rather than wording, so the human isn't just re-reading prose for typos.
- **New docs quietly contradicting existing docs.** A generated page can be accurate on its own yet conflict with something already published, like a differing default value or permission rule. Mitigation: gate releases on evals that diff new pages against the existing doc corpus and block publish on contradictions.

### Q5. 你最喜欢的AI产品技术文档站点

> *Original question: What is your favorite technical documentation site or developer portal specifically for an AI product (e.g., OpenAI, Anthropic, Hugging Face, LangChain) and why? (Answer this question in Chinese)*

我最喜欢的是Anthropic的开发者文档站(docs.claude.com)。原因主要有几点:

- **参考文档和教程分开清晰**:它把参考文档和实操教程分开得很清楚,你可以很快找到你需要的那种内容,而不用在一个大而全的页面里来回翻。

- **代码示例可直接运行,多语言覆盖**:几乎每一个概念旁边都配了可以直接跑的代码示例,而且是多语言的(Python、TypeScript、curl等),这对开发者来说非常实用,不需要自己再去翻译示例代码。

- **对模型输出的不确定性写得诚实**:文档里对于模型输出的不确定性写得比较诚实,不会假装每次调用都会得到一模一样的结果,而是解释清楚哪些部分是稳定的(比如响应结构),哪些部分本质上是可变的(比如生成的文字内容)。这种诚实的写法其实解决了上面第3题提到的那个问题,值得其他AI产品文档参考。

- **更新速度快,跟得上产品迭代**:文档的更新速度也比较快,新模型或新功能上线后,文档基本能同步跟上,不会出现文档和实际产品脱节太久的情况。
