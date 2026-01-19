# Adsetric

AI-powered Meta Ads analytics. Connect your ad account, get GPT-4 insights and recommendations.

## Setup

```bash
bundle install
rails db:migrate
rails server
```

### .env
```bash
UNIFIED_WORKSPACE_ID=your_workspace_id
UNIFIED_API_TOKEN=your_jwt_token
UNIFIED_ENV=Production
OPENAI_API_KEY=sk-proj-...
```

---

## How It Works

1. User connects Meta Ads via Unified.to OAuth
2. Selects which ad account (organization) to analyze
3. Background job fetches campaigns/ads/reports from Unified API
4. Sends compact data to GPT-4o-mini for analysis
5. Stores structured insights and actions in database
6. Dashboard displays summary, metrics, insights, and action items

---

## Unified.to API Integration

Unified.to abstracts the Meta Ads API into a simple REST interface.

### OAuth Flow

1. **`unified_connect`** - Generates OAuth URL, redirects user to Unified
2. User authorizes on Meta → Unified redirects back with `connection_id`
3. **`unified_callback`** - Verifies signature, stores connection_id in session
4. **`unified_select_org`** - Fetches list of ad accounts user has access to
5. **`unified_save_org`** - Saves selected ad account, triggers analysis job

### Data Fetching

**Main method:** `UnifiedAdsService.fetch_all_data(connection_id:, org_id:, days:)`

Fetches from 4 endpoints:
- `/ads/{connection_id}/campaign` - All campaigns
- `/ads/{connection_id}/group` - All ad sets
- `/ads/{connection_id}/ad` - All ads
- `/ads/{connection_id}/report` - Performance metrics by date

All endpoints pass `org_id` to scope to the selected ad account.

**Pagination:** Each endpoint fetches 100 items per page, continues until less than 100 returned. Safety cap at 50 pages.

**Rate Limiting:** On 429, waits for `Retry-After` header (or 2s), retries twice with exponential backoff.

**Auth Errors:** On 401/403, sets `needs_reconnect: true` so UI can prompt user to reconnect.

### Return Data

```ruby
{
  success: true,
  needs_reconnect: false,
  data: {
    campaigns: [...],      # Normalized campaign objects
    ad_groups: [...],      # Normalized ad set objects  
    ads: [...],            # Normalized ad objects
    metrics: {             # Aggregated totals from reports
      total_spend, total_impressions, total_clicks,
      total_conversions, total_revenue,
      average_ctr, average_cpc, average_roas
    },
    daily_breakdown: [...]  # Daily metrics for charts
  },
  metadata: {
    connection_id, org_id, days, pulled_at,
    counts: { campaigns: 5, groups: 12, ads: 30, reports: 30 }
  }
}
```

---

## ChatGPT Analysis

GPT-4o-mini analyzes the ad data and returns structured insights.

### What Gets Sent

We send a **compact payload** to keep costs low:
- Aggregated metrics (spend, impressions, clicks, CTR, conversions, ROAS)
- Last 14 days of daily breakdown (for trend analysis)
- Top 5 campaigns by spend (name, status, metrics)
- Metadata (date range, counts, completeness flags)

### Structured Output

We use OpenAI's **strict JSON schema** so GPT always returns valid, parseable JSON:

```json
{
  "summary": "2-3 sentence performance overview",
  "key_metrics": {
    "spend": 1234.56,
    "impressions": 50000,
    "clicks": 1500,
    "ctr": 3.0,
    "conversions": 45,
    "roas": 2.84
  },
  "insights": [
    {
      "type": "success|warning|critical",
      "title": "Strong ROAS Performance",
      "detail": "Your 2.84x ROAS exceeds industry average",
      "evidence": "$3,500 revenue from $1,234 spend"
    }
  ],
  "actions": [
    {
      "priority": "high|medium|low",
      "title": "Scale Top Campaign",
      "steps": ["Increase budget by 20%", "Monitor for 3 days"],
      "expected_impact": "+$500 weekly revenue"
    }
  ]
}
```

### Insight Types
- **success** - Positive performance worth highlighting
- **warning** - Needs attention but not urgent
- **critical** - Urgent issue requiring immediate action

### Action Priorities
- **high** - Do first, biggest impact
- **medium** - Important but can wait
- **low** - Nice to have, optimize later

### Configuration
- Model: `gpt-4o-mini` (fast, cheap, supports structured outputs)
- Temperature: `0.3` (consistent, factual responses)
- Max tokens: `1500`

### Fallback

If GPT fails, the job creates a basic summary without insights/actions. The `analysis_version` column tracks this (1 = GPT, 0 = fallback).

---

## Key Files

| File | Purpose |
|------|---------|
| `app/services/unified_ads_service.rb` | Unified.to API - OAuth, data fetching |
| `app/services/chatgpt_analysis_service.rb` | OpenAI API - structured analysis |
| `app/jobs/ad_account_analysis_job.rb` | Background job - fetch + analyze |
| `app/controllers/private/settings/account_controller.rb` | OAuth flow |

---

## Database

**ad_accounts:** `unified_connection_id`, `unified_organization_id`, `name`, `active`, `last_synced_at`

**ad_analyses:** `summary`, `key_metrics`, `insights`, `actions`, `analysis_json`, `analysis_version`, `raw_ad_data`

---

## User Flow

```
Sign up → Connect Meta Ads → Select ad account → Analyzing screen → Dashboard
```

The analyzing page polls `/analysis_status` every 2s until the background job completes.

---

## Routes

```
/unified/connect      → Start OAuth
/unified/callback     → Handle callback  
/unified/select_org   → Pick ad account
/unified/save_org     → Save + trigger job

/dashboard/overview   → Main dashboard
/dashboard/analyzing  → Loading screen
/dashboard/analytics  → Detailed metrics
/dashboard/campaigns  → Campaign list
```

---

## Brand Colors

- Primary: `#0a61ec`
- Dark: `#0950d1`  
- Light: `#badaff`
