# Adsetric

AI-powered Meta Ads analytics platform. Connect your ad accounts, get GPT-4 analysis, and actionable insights.

## Tech Stack
- Ruby on Rails 7.1
- SQLite (development)
- Tailwind CSS
- Unified.to (Meta Ads API)
- OpenAI GPT-4o-mini

## Setup

```bash
bundle install
rails db:migrate
```

### Environment Variables (.env)
```
# Unified.to - Meta Ads API
UNIFIED_WORKSPACE_ID=your_workspace_id
UNIFIED_API_TOKEN=your_jwt_token
UNIFIED_WORKSPACE_SECRET=optional_for_signature_verification
UNIFIED_ENV=Production  # or Sandbox for testing

# OpenAI
OPENAI_API_KEY=sk-proj-...
```

---

## Database Schema

### users
| Column | Type | Description |
|--------|------|-------------|
| name | string | Display name |
| email | string | Unique email |
| password_digest | string | Bcrypt hash |

### ad_accounts
| Column | Type | Description |
|--------|------|-------------|
| user_id | integer | FK to users |
| unified_connection_id | string | Unified OAuth connection |
| unified_organization_id | string | Meta ad account ID |
| platform | string | "metaads" |
| name | string | Account display name |
| active | boolean | Is connected |
| last_synced_at | datetime | Last data pull |

### ad_analyses
| Column | Type | Description |
|--------|------|-------------|
| ad_account_id | integer | FK to ad_accounts |
| summary | text | GPT-generated summary |
| key_metrics | json | Totals (spend, clicks, etc) |
| insights | json | Array of insight objects |
| actions | json | Array of action objects |
| analysis_json | json | Full GPT response |
| analysis_version | integer | Schema version (1 = GPT) |
| raw_ad_data | json | Campaigns, ads, reports, metadata |
| analyzed_at | datetime | When analyzed |

---

## Unified.to Integration

### Service: `app/services/unified_ads_service.rb`

Handles all Meta Ads API communication via Unified.to.

#### OAuth Flow
```ruby
# 1. Generate OAuth URL
UnifiedAdsService.oauth_url(
  user: current_user,
  success_redirect: "http://localhost:3000/unified/callback",
  failure_redirect: "http://localhost:3000/settings?error=failed"
)
# Returns: https://api.unified.to/unified/integration/auth/{workspace}/metaads?...

# 2. Handle callback - verify signature, get connection_id
UnifiedAdsService.handle_callback(params)
# Returns: { success: true, connection_id: "abc123", user_id: 1 }

# 3. List ad accounts (organizations)
UnifiedAdsService.list_organizations(connection_id)
# Returns: { success: true, data: [{ id: "123", name: "My Ad Account" }, ...] }
```

#### Data Fetching
```ruby
# Main method - fetches everything in one call
result = UnifiedAdsService.fetch_all_data(
  connection_id: "abc123",
  org_id: "456789",  # Meta ad account ID
  days: 30
)

# Returns:
{
  success: true,
  needs_reconnect: false,
  data_incomplete: false,
  data_scope_warning: false,
  data: {
    campaigns: [...],      # Normalized campaign objects
    ad_groups: [...],      # Normalized ad set objects
    ads: [...],            # Normalized ad objects
    reports: [...],        # Raw report data
    metrics: {             # Aggregated from reports
      total_spend: 1234.56,
      total_impressions: 50000,
      total_clicks: 1500,
      total_conversions: 45,
      total_revenue: 3500.00,
      average_ctr: 3.0,
      average_cpc: 0.82,
      average_roas: 2.84
    },
    daily_breakdown: [...],  # Daily metrics
    date_range: { start_date: "2026-01-01", end_date: "2026-01-19", days: 30 }
  },
  metadata: {
    connection_id: "abc123",
    org_id: "456789",
    days: 30,
    start_gte: "2025-12-20T00:00:00Z",
    end_lt: "2026-01-19T00:00:00Z",
    pulled_at: "2026-01-19T12:00:00Z",
    timezone: "UTC",
    report_type: "daily",
    report_granularity: "day",
    counts: { campaigns: 5, groups: 12, ads: 30, reports: 30 },
    data_incomplete: false,
    truncated_endpoints: [],
    data_scope_warning: false,
    scope_warnings: nil
  }
}
```

#### Individual Fetch Methods (all paginated)
```ruby
UnifiedAdsService.fetch_campaigns(connection_id, org_id: org_id)
UnifiedAdsService.fetch_ad_groups(connection_id, org_id: org_id)
UnifiedAdsService.fetch_ads(connection_id, org_id: org_id)
UnifiedAdsService.fetch_reports(connection_id, org_id: org_id, start_gte: date, end_lt: date)
```

#### Error Handling
- `401/403` → Sets `needs_reconnect: true`, mark account inactive
- `429` → Retries with exponential backoff (2s → 4s, max 10s, 2 retries)
- `5xx` → Logs error, returns partial data if available
- Pagination safety cap: 50 pages max per endpoint

#### Org Scoping
- Always pass `org_id` in API requests (primary filter)
- `filter_by_org()` is a safety net - logs warning if items filtered out
- Tracks `scope_warnings` in metadata if mismatched items found

---

## ChatGPT Analysis

### Service: `app/services/chatgpt_analysis_service.rb`

Sends ad data to GPT-4o-mini and returns structured analysis.

#### Usage
```ruby
result = ChatgptAnalysisService.analyze(data: unified_result)

# Returns:
{
  success: true,
  version: 1,
  analysis: {
    "summary" => "Your campaigns are performing well with 2.84x ROAS...",
    "key_metrics" => {
      "spend" => 1234.56,
      "impressions" => 50000,
      "clicks" => 1500,
      "ctr" => 3.0,
      "conversions" => 45,
      "roas" => 2.84
    },
    "insights" => [
      {
        "type" => "success",        # success | warning | critical
        "title" => "Strong ROAS",
        "detail" => "Your 2.84x ROAS exceeds industry average of 2.0x",
        "evidence" => "$3,500 revenue from $1,234 spend"
      }
    ],
    "actions" => [
      {
        "priority" => "high",       # high | medium | low
        "title" => "Scale top campaign",
        "steps" => ["Increase budget by 20%", "Monitor for 3 days"],
        "expected_impact" => "+$500 weekly revenue"
      }
    ]
  }
}
```

#### What Gets Sent to GPT (compact payload)
```ruby
{
  metrics: { total_spend, total_impressions, ... },
  daily_breakdown: last_14_days,
  top_campaigns: top_5_by_spend,
  metadata: { days, campaign_count, ad_count, data_incomplete }
}
```

#### JSON Schema (enforced by OpenAI)
```json
{
  "summary": "string",
  "key_metrics": { "spend": 0, "impressions": 0, "clicks": 0, "ctr": 0, "conversions": 0, "roas": 0 },
  "insights": [{ "type": "success|warning|critical", "title": "", "detail": "", "evidence": "" }],
  "actions": [{ "priority": "high|medium|low", "title": "", "steps": [""], "expected_impact": "" }]
}
```

#### Configuration
- Model: `gpt-4o-mini` (fast, cheap, supports structured outputs)
- Temperature: `0.3` (consistent, factual)
- Max tokens: `1500`
- Version tracking: `analysis_version` column for schema migrations

---

## User Flow

### Connect Ad Account
1. User clicks "Connect Meta Ads" → `unified_connect`
2. Redirects to Unified.to OAuth → User authorizes on Meta
3. Callback with `connection_id` → `unified_callback`
4. Show organization picker → `unified_select_org`
5. User selects ad account → `unified_save_org`
6. Creates `ad_account` record, triggers `AdAccountAnalysisJob`
7. Redirects to `/dashboard/analyzing` (loading screen)
8. Job fetches Unified data → calls ChatGPT → creates `ad_analysis`
9. Analyzing page polls `/analysis_status`, redirects to overview when done

### Background Job: `AdAccountAnalysisJob`
```ruby
AdAccountAnalysisJob.perform_later(ad_account.id)
```
- Fetches data from Unified.to
- Calls ChatGPT for analysis
- Creates `ad_analysis` record
- Falls back to basic summary if GPT fails
- Creates error analysis if Unified fails

---

## Routes

```ruby
# Auth
GET/POST /auth/register
GET/POST /auth/login
DELETE    /logout

# Dashboard
GET  /dashboard/overview
GET  /dashboard/analyzing
GET  /dashboard/overview/analysis_status
POST /dashboard/overview/trigger_analysis
GET  /dashboard/analytics
GET  /dashboard/campaigns

# Settings & OAuth
GET    /dashboard/settings
PATCH  /dashboard/settings
DELETE /dashboard/settings/ad_accounts/:id
GET    /unified/connect
GET    /unified/callback
GET    /unified/select_org
POST   /unified/save_org
```

---

## Brand Colors
- Primary: `#0a61ec`
- Dark: `#0950d1`
- Light: `#badaff`
