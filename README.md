# Adsetric - Complete Changelog

## Database Structure

### Users Table
- `id` - integer, primary key
- `name` - string, required
- `email` - string, required, unique, validated email format
- `password_digest` - string, bcrypt encrypted
- `created_at` - datetime
- `updated_at` - datetime
- Index on `email` (unique)

### Ad Accounts Table
- `id` - integer, primary key
- `user_id` - integer, foreign key to users, required
- `unified_connection_id` - string, required, unique (stores mock IDs like "mock_connection_abc123")
- `platform` - string, default "facebook", required
- `name` - string, required (e.g. "Demo Facebook Ad Account")
- `account_id` - string (stores mock IDs like "mock_abc123")
- `active` - boolean, default true
- `last_synced_at` - datetime
- `created_at` - datetime
- `updated_at` - datetime
- Index on `user_id`
- Index on `unified_connection_id` (unique)
- Foreign key to `users`

### Ad Analyses Table
- `id` - integer, primary key
- `ad_account_id` - integer, foreign key to ad_accounts, required
- `summary` - text (AI-generated summary like "Your ad campaigns are performing well...")
- `key_metrics` - json, default {} (stores hash with total_spend, total_impressions, total_clicks, total_conversions, total_revenue, average_ctr, average_cpc, average_roas)
- `insights` - json, default [] (stores array of objects with type, title, description, impact)
- `actions` - json, default [] (stores array of objects with priority, action, description, expected_impact, effort)
- `raw_ad_data` - json, default {} (stores campaigns array, ad_groups array, ads array, report hash with daily_breakdown)
- `analyzed_at` - datetime, required
- `created_at` - datetime
- `updated_at` - datetime
- Index on `ad_account_id`
- Index on `analyzed_at`
- Foreign key to `ad_accounts`

### Model Relationships
- User `has_many :ad_accounts, dependent: :destroy`
- User `has_many :ad_analyses, through: :ad_accounts`
- AdAccount `belongs_to :user`
- AdAccount `has_many :ad_analyses, dependent: :destroy`
- AdAnalysis `belongs_to :ad_account`

### Model Validations
- User: email presence, uniqueness (case insensitive), format; name presence; password length min 6 on create
- AdAccount: unified_connection_id presence, platform presence, name presence
- AdAnalysis: analyzed_at presence

### Model Scopes
- AdAccount.active - `where(active: true)`
- AdAnalysis.latest - `order(analyzed_at: :desc)`
- AdAnalysis.recent - `where("analyzed_at > ?", 24.hours.ago)`

### Model Callbacks
- User: `before_save :downcase_email`

---

## Added

### Sidebar (app/views/layouts/dashboard.html.erb)
- Centered logo image at top (h-20 w-20 rounded-2xl)
- "Adsetric" title text below logo (font-display font-bold text-xl text-gray-900)
- "Ad Analytics" subtitle (text-sm text-gray-500)
- Gradient separator line (h-px bg-gradient-to-r from-transparent via-gray-200 to-transparent)
- "ACTIVE ACCOUNT" section header (text-xs font-semibold text-gray-400 uppercase tracking-wider)
- Account dropdown button showing current account name and Meta icon
- Dropdown menu listing all connected accounts
- "Add new account" button in dropdown (button_to add_mock_account_path)
- Navigation items without icon boxes - icons display directly
- Active nav state: bg-[#0a61ec] text-white shadow-lg shadow-[#0a61ec]/25
- Inactive nav state: text-gray-600 hover:bg-gray-100
- User info card at sidebar bottom (p-4 border-t border-gray-200)
- Avatar circle with user initial (w-10 h-10 bg-[#0a61ec] rounded-xl text-white font-bold)
- User name display (font-semibold text-gray-900 text-sm)
- User email display (text-xs text-gray-500)
- Logout button integrated in user card (bx bx-log-out icon)
