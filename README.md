# Adsetric - Configuration & Integration Guide

## Environment Variables

```bash
# .env

# Unified.to - Meta Ads API (https://app.unified.to/settings/api)
UNIFIED_WORKSPACE_ID=your_workspace_id
UNIFIED_API_TOKEN=your_jwt_token
UNIFIED_WORKSPACE_SECRET=optional_signature_verification
UNIFIED_ENV=Production  # or Sandbox for testing

# OpenAI (https://platform.openai.com/api-keys)
OPENAI_API_KEY=sk-proj-...
```

---

## Unified.to API Integration

**Service:** `app/services/unified_ads_service.rb`

### OAuth Flow

| Step | Route | What Happens |
|------|-------|--------------|
| 1 | `GET /dashboard/settings/unified/connect` | Generates Unified OAuth URL, redirects user |
| 2 | (Unified.to) | User logs into Meta, authorizes app |
| 3 | `GET /dashboard/settings/unified/callback` | Verifies signature, stores `connection_id` in session |
| 4 | `GET /dashboard/settings/unified/select_org` | Fetches ad accounts, shows picker UI |
| 5 | `POST /dashboard/settings/unified/save_org` | Saves ad account, triggers analysis job |

### Data Fetching

**Main method:** `UnifiedAdsService.fetch_all_data(connection_id:, org_id:, days:)`

Calls 4 Unified endpoints:
- `GET /ads/{connection_id}/campaign?org_id=...`
- `GET /ads/{connection_id}/group?org_id=...` (ad sets)
- `GET /ads/{connection_id}/ad?org_id=...`
- `GET /ads/{connection_id}/report?org_id=...&start_gte=...&end_lt=...`

**Pagination:** 100 items/page, continues until < 100 returned, 50 page safety cap.

**Rate Limiting:** On 429, retries with exponential backoff (2s → 4s, max 10s).

**Auth Errors:** On 401/403, sets `needs_reconnect: true` to prompt reconnection.

### Return Structure

```ruby
{
  success: true,
  needs_reconnect: false,
  data_incomplete: false,
  data: {
    campaigns: [...],
    ad_groups: [...],
    ads: [...],
    metrics: { total_spend, total_impressions, total_clicks, total_conversions, 
               total_revenue, average_ctr, average_cpc, average_roas },
    daily_breakdown: [...]
  },
  metadata: { connection_id, org_id, days, pulled_at, counts, truncated_endpoints }
}
```

---

## ChatGPT Analysis

**Service:** `app/services/chatgpt_analysis_service.rb`

### What Gets Sent to GPT

Compact payload to minimize cost:
- Aggregated metrics (spend, impressions, clicks, CTR, conversions, ROAS)
- Last 14 days daily breakdown (for trend analysis)
- Top 5 campaigns by spend
- Metadata (date range, counts)

### What GPT Analyzes

GPT receives the data and evaluates:

1. **Overall Performance** - Is ROAS good? How does spend efficiency look?
2. **Trends** - Are metrics improving, declining, or stable over the 14-day window?
3. **Campaign Comparison** - Which campaigns are performing best/worst?
4. **Anomalies** - Any sudden drops in CTR, spikes in CPC, or unusual patterns?

### What the User Gets

**Summary:** A 2-3 sentence overview like:
> "Your campaigns generated $3,500 revenue from $1,234 spend (2.84x ROAS) over the last 30 days. The Summer Sale campaign is your top performer, but CTR has declined 15% this week."

**Insights:** AI-identified observations with evidence:
- "Strong ROAS Performance" (success) - "Your 2.84x ROAS exceeds industry average of 2.0x"
- "Declining CTR Trend" (warning) - "CTR dropped from 3.5% to 2.9% over the past week"
- "Mobile Underperforming" (critical) - "Mobile ROAS is 1.2x vs 3.1x on desktop"

**Actions:** Prioritized recommendations with steps:
- "Refresh Ad Creatives" (high) - Steps: Create new variations, A/B test, pause underperformers
- "Scale Top Campaign" (medium) - Steps: Increase budget 20%, monitor 3 days, scale if ROAS holds

### Response Schema (enforced by OpenAI)

```json
{
  "summary": "2-3 sentence performance overview",
  "key_metrics": { "spend", "impressions", "clicks", "ctr", "conversions", "roas" },
  "insights": [{
    "type": "success|warning|critical",
    "title": "Insight title",
    "detail": "Explanation of what GPT found",
    "evidence": "Specific data backing this up"
  }],
  "actions": [{
    "priority": "high|medium|low",
    "title": "Action title",
    "steps": ["Step 1", "Step 2", "Step 3"],
    "expected_impact": "What user can expect if they do this"
  }]
}
```

### Insight Types
- `success` - Positive performance worth celebrating
- `warning` - Needs attention but not urgent
- `critical` - Urgent issue hurting performance

### Action Priorities
- `high` - Do this first, biggest impact on results
- `medium` - Important but can wait a few days
- `low` - Nice optimization, do when you have time

### Config
- Model: `gpt-4o-mini` (fast, cheap, supports structured outputs)
- Temperature: `0.3` (consistent, factual - not creative)
- Strict JSON schema enforced (GPT must return valid structure)

---

## Background Job

**Job:** `app/jobs/ad_account_analysis_job.rb`

```ruby
AdAccountAnalysisJob.perform_later(ad_account.id)
```

1. Fetches data from Unified.to
2. Sends to ChatGPT for analysis
3. Creates `ad_analysis` record
4. Falls back to basic summary if GPT fails

---

## Routes

### Auth
```
GET  /auth/register
POST /auth/register
GET  /auth/login
POST /auth/login
DELETE /logout
```

### Dashboard
```
GET  /dashboard/overview              → Main dashboard (or onboarding)
GET  /dashboard/analyzing             → Loading screen with polling
GET  /dashboard/overview/analysis_status  → JSON: { complete: true/false }
POST /dashboard/overview/trigger_analysis → Trigger re-analysis
GET  /dashboard/analytics             → Detailed metrics
GET  /dashboard/campaigns             → Campaign list
```

### Settings & OAuth
```
GET    /dashboard/settings                      → Account settings
PATCH  /dashboard/settings                      → Update settings
DELETE /dashboard/settings/ad_accounts/:id      → Disconnect account
POST   /dashboard/settings/add_mock_account     → Add demo account

GET  /dashboard/settings/unified/connect     → Start OAuth
GET  /dashboard/settings/unified/callback    → Handle OAuth callback
GET  /dashboard/settings/unified/select_org  → Organization picker
POST /dashboard/settings/unified/save_org    → Save org + trigger job
```

---

## Database Columns Added

**ad_accounts:**
- `unified_organization_id` - Meta ad account ID (user selects after OAuth)

**ad_analyses:**
- `analysis_json` - Full GPT response stored as JSON
- `analysis_version` - Schema version (1 = GPT, 0 = fallback)

---

## Key Files

| File | Purpose |
|------|---------|
| `app/services/unified_ads_service.rb` | Unified.to OAuth + data fetching |
| `app/services/chatgpt_analysis_service.rb` | GPT-4 structured analysis |
| `app/jobs/ad_account_analysis_job.rb` | Background analysis job |
| `app/controllers/private/settings/account_controller.rb` | OAuth flow + account management |
| `app/views/private/dashboard/overview/analyzing.html.erb` | Loading screen with polling |
| `app/views/private/settings/account/unified_select_org.html.erb` | Organization picker UI |

---

## Brand Colors

- Primary: `#0a61ec`
- Dark: `#0950d1`
- Light: `#badaff`
