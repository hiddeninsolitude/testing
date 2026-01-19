# Adsetric - Complete Changelog

## Database Structure

### Users Table
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | integer | PRIMARY KEY | Auto-increment |
| name | string | NOT NULL | User's display name |
| email | string | NOT NULL, UNIQUE | Email address |
| password_digest | string | NOT NULL | Bcrypt encrypted password |
| created_at | datetime | NOT NULL | Record creation time |
| updated_at | datetime | NOT NULL | Record update time |

**Indexes:**
- `index_users_on_email` (unique)

**Validations:**
- email: presence, uniqueness (case insensitive), format (URI::MailTo::EMAIL_REGEXP)
- name: presence
- password: length minimum 6 (on create only)

**Callbacks:**
- before_save: downcase_email

### Ad Accounts Table
| Column | Type | Constraints | Default | Description |
|--------|------|-------------|---------|-------------|
| id | integer | PRIMARY KEY | | Auto-increment |
| user_id | integer | NOT NULL, FK | | References users.id |
| unified_connection_id | string | NOT NULL, UNIQUE | | Connection identifier |
| platform | string | NOT NULL | "facebook" | Ad platform |
| name | string | NOT NULL | | Account display name |
| account_id | string | | | Platform account ID |
| active | boolean | | true | Is account active |
| last_synced_at | datetime | | | Last data sync time |
| created_at | datetime | NOT NULL | | Record creation time |
| updated_at | datetime | NOT NULL | | Record update time |

**Indexes:**
- `index_ad_accounts_on_user_id`
- `index_ad_accounts_on_unified_connection_id` (unique)

**Foreign Keys:**
- user_id -> users.id

**Validations:**
- unified_connection_id: presence
- platform: presence
- name: presence

**Scopes:**
- `active` -> `where(active: true)`

### Ad Analyses Table
| Column | Type | Constraints | Default | Description |
|--------|------|-------------|---------|-------------|
| id | integer | PRIMARY KEY | | Auto-increment |
| ad_account_id | integer | NOT NULL, FK | | References ad_accounts.id |
| summary | text | | | AI-generated summary text |
| key_metrics | json | | {} | Hash of metric totals/averages |
| insights | json | | [] | Array of insight objects |
| actions | json | | [] | Array of action objects |
| raw_ad_data | json | | {} | Full campaign/ad data |
| analyzed_at | datetime | NOT NULL | | Analysis timestamp |
| created_at | datetime | NOT NULL | | Record creation time |
| updated_at | datetime | NOT NULL | | Record update time |

**Indexes:**
- `index_ad_analyses_on_ad_account_id`
- `index_ad_analyses_on_analyzed_at`

**Foreign Keys:**
- ad_account_id -> ad_accounts.id

**Validations:**
- analyzed_at: presence

**Scopes:**
- `latest` -> `order(analyzed_at: :desc)`
- `recent` -> `where("analyzed_at > ?", 24.hours.ago)`

### Model Relationships
```
User
  has_many :ad_accounts, dependent: :destroy
  has_many :ad_analyses, through: :ad_accounts

AdAccount
  belongs_to :user
  has_many :ad_analyses, dependent: :destroy

AdAnalysis
  belongs_to :ad_account
```

---

## Routes

### All Routes (config/routes.rb)
```ruby
# Root
root to: "public/landing#landing"

# Auth Routes
GET    /auth/register    -> public/auth/register#register    as: :register
POST   /auth/register    -> public/auth/register#create
GET    /auth/login       -> public/auth/login#login          as: :login
POST   /auth/login       -> public/auth/login#create
DELETE /logout           -> public/auth/login#destroy        as: :logout

# Dashboard Routes
GET    /dashboard/overview                    -> private/dashboard/overview#overview          as: :dashboard_overview
GET    /dashboard/analyzing                   -> private/dashboard/overview#analyzing         as: :dashboard_analyzing
GET    /dashboard/overview/analysis_status    -> private/dashboard/overview#analysis_status   as: :dashboard_analysis_status
POST   /dashboard/overview/trigger_analysis   -> private/dashboard/overview#trigger_analysis  as: :dashboard_trigger_analysis
GET    /dashboard/analytics                   -> private/dashboard/analytics#analytics        as: :dashboard_analytics
GET    /dashboard/campaigns                   -> private/dashboard/campaigns#campaigns        as: :dashboard_campaigns

# Settings Routes
GET    /dashboard/settings                        -> private/settings/account#show                as: :settings
PATCH  /dashboard/settings                        -> private/settings/account#update
DELETE /dashboard/settings/ad_accounts/:id        -> private/settings/account#disconnect_ad_account  as: :disconnect_ad_account
POST   /dashboard/settings/add_mock_account       -> private/settings/account#add_mock_account       as: :add_mock_account

# Health Check
GET    /up    -> rails/health#show    as: :rails_health_check
```

### Routes Added
- `POST /dashboard/settings/add_mock_account` - Creates demo ad account with mock data

### Routes Removed
- `GET /oauth/unified/connect` - OAuth initiation
- `GET /oauth/unified/callback` - OAuth callback handler
- `GET /unified/status` - API status page

---

## Mock Data Structure

### key_metrics (Hash)
```ruby
{
  "total_spend" => 3247.50,
  "total_impressions" => 125430,
  "total_clicks" => 3876,
  "total_conversions" => 142,
  "total_revenue" => 8520.00,
  "average_ctr" => 3.09,
  "average_cpc" => 0.84,
  "average_roas" => 2.62
}
```

### insights (Array)
```ruby
[
  { "type" => "success", "title" => "Strong ROAS Performance", "description" => "...", "impact" => "high" },
  { "type" => "warning", "title" => "High Cost Per Click", "description" => "...", "impact" => "medium" },
  { "type" => "opportunity", "title" => "Mobile Performance Gap", "description" => "...", "impact" => "high" }
]
```

### actions (Array)
```ruby
[
  { "priority" => "high", "action" => "Increase Budget for Top Campaigns", "expected_impact" => "+$2,400 monthly revenue", "effort" => "low" },
  { "priority" => "medium", "action" => "Pause Underperforming Ad Groups", "expected_impact" => "-$450 wasted spend", "effort" => "low" },
  { "priority" => "high", "action" => "Optimize Mobile Creatives", "expected_impact" => "+35% mobile conversions", "effort" => "medium" }
]
```

### raw_ad_data.campaigns (Array - 3 items)
```ruby
{
  "id" => "camp_1",
  "name" => "Summer Sale 2026",
  "status" => "ACTIVE",
  "objective" => "Conversions",
  "budget" => 2000.00,
  "spend" => 1247.50,
  "impressions" => 48230,
  "clicks" => 1456,
  "conversions" => 58,
  "revenue" => 3480.00,
  "ctr" => 3.02,
  "cpc" => 0.86,
  "roas" => 2.79
}
```

### raw_ad_data.ad_groups (Array - 5 items)
```ruby
{
  "id" => "adg_1",
  "campaign_id" => "camp_1",
  "name" => "Audience A - 25-34",
  "status" => "ACTIVE",
  "daily_budget" => 50.00,
  "spend" => 647.50,
  "impressions" => 25120,
  "clicks" => 756,
  "conversions" => 32,
  "revenue" => 1920.00,
  "ctr" => 3.01,
  "cpc" => 0.86,
  "roas" => 2.97
}
```

### raw_ad_data.ads (Array - 10 items)
```ruby
{
  "id" => "ad_1",
  "adset_id" => "adg_1",
  "name" => "Video Ad - Product Demo",
  "status" => "ACTIVE",
  "creative_type" => "Video",
  "spend" => 347.50,
  "impressions" => 13560,
  "clicks" => 408,
  "conversions" => 18,
  "revenue" => 1080.00,
  "ctr" => 3.01,
  "cpc" => 0.85,
  "roas" => 3.11
}
```

### raw_ad_data.report.daily_breakdown (Array - 7 days)
```ruby
{
  "date" => "2026-01-13",
  "spend" => 98.50,
  "impressions" => 3850,
  "clicks" => 118,
  "conversions" => 4,
  "revenue" => 240.00,
  "ctr" => 3.06,
  "cpc" => 0.83,
  "roas" => 2.44
}
```

### raw_ad_data.report.device_breakdown (Hash)
```ruby
{
  "mobile" => { "spend" => 1623.75, "impressions" => 62715, "clicks" => 1938, "conversions" => 65, "revenue" => 3900.00, ... },
  "desktop" => { "spend" => 1298.00, "impressions" => 50172, "clicks" => 1551, "conversions" => 62, "revenue" => 3720.00, ... },
  "tablet" => { "spend" => 325.75, "impressions" => 12543, "clicks" => 387, "conversions" => 15, "revenue" => 900.00, ... }
}
```

---

## Added

### UI Changes
- Redesigned sidebar with centered logo, title "Adsetric", subtitle "Ad Analytics"
- Gradient separator line between logo and account section
- User info card at sidebar bottom with avatar, name, email, logout button
- Navigation icons display directly without icon boxes
- Active nav state: white icons on blue background with shadow
- Number animations on all metric cards (count up from 0 over 2 seconds)
- Daily averages on all stat cards
- Additional metrics: CPM, Cost per Conversion, Conversion Rate, Profit
- Conversion funnel with drop-off indicators
- Device & placement performance sections
- Expandable campaign/ad set/ad cards
- Best day indicators in daily breakdown table
- Color-coded ROAS badges

### Controllers
- `AccountController#add_mock_account` - Creates demo account with full mock data

### JavaScript
- `animateNumber()` - Counter animation function
- `refreshAnalysis()` - Triggers analysis job via AJAX
- Progress bar animations with CSS transitions
- Expandable card toggle handlers
- Search and filter functionality on campaigns page

---

## Changed

### UI Changes
- Removed profile icon from top-right navbar
- Moved logout to sidebar user card
- All "Connect Account" links changed to button_to forms

### Controllers
- `AnalyticsController` - Handles nested daily_breakdown in report hash
- `AccountController` - Redirects to overview after mock account creation (not analyzing page)

### Models
- `AdAccount` scope changed from `where(status: "active")` to `where(active: true)`

### Config
- Removed ngrok host from development.rb

### User Flow
- Before: Sign up -> Auto-create demo account -> Overview
- After: Sign up -> Onboarding -> Click "Add Account" -> Overview

---

## Removed

### Files
- `app/services/unified_ads_service.rb`
- `app/services/chatgpt_analysis_service.rb`
- `app/controllers/private/oauth/unified_controller.rb`
- `app/controllers/private/unified_status_controller.rb`
- `app/views/private/unified_status/show.html.erb`
- `lib/tasks/unified_integrations.rake`
- `lib/tasks/unified_test.rake`
- `test_unified.rb`
- `UNIFIED_API_SETUP_GUIDE.md`
- `UNIFIED_API_TROUBLESHOOTING.md`
- `ANALYTICS_DATA_FLOW.md`
- `INTEGRATION_VERIFICATION.md`
- `ANALYTICS_ENHANCEMENTS_SUMMARY.md`

### Folders
- `app/services/`
- `app/controllers/private/oauth/`
- `app/views/private/unified_status/`

### Features
- Unified.to API integration
- ChatGPT/OpenAI analysis
- OAuth flow
- API Status page in settings
- Auto demo account on signup
- Background job re-enqueueing

### Code from AdAccountAnalysisJob
- UnifiedAdsService calls
- ChatgptAnalysisService calls
- Mock mode delays
- Job re-enqueueing every 10 minutes

---

## Fixed

### Scroll Blocking
- AOS library was blocking scrolling
- Added `overflow-y: scroll !important` to html/body
- Added `pointer-events: auto !important` on animated elements

### Mock Data Fields
- Added `budget` to campaigns
- Added `objective` to campaigns
- Added `daily_budget` to ad_groups
- Added `creative_type` to ads
- Changed `ad_group_id` to `adset_id` in ads

### Field Names
- Changed `avg_ctr` to `average_ctr`
- Changed `avg_cpc` to `average_cpc`
- Changed `avg_roas` to `average_roas`

### Other
- Added `summary` field to analysis
- Fixed analytics controller to handle nested daily_breakdown
- Removed `.gsub` calls on numeric values in overview

---

## Brand Colors
- Primary: `#0a61ec`
- Dark: `#0950d1`
- Light: `#badaff`
