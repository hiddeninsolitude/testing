# Adsetric - Deployment Guide

Simple guide to get Adsetric running in production.

---

## Step 1: Get Stripe API Keys

1. Go to https://dashboard.stripe.com/apikeys
2. Copy **Secret key** (starts with `sk_`) → Save as `STRIPE_SECRET_KEY`
3. Copy **Publishable key** (starts with `pk_`) → Save as `STRIPE_PUBLISHABLE_KEY`

---

## Step 2: Create Stripe Products & Prices

Go to https://dashboard.stripe.com/products

Create 3 products with monthly and yearly pricing:

**Product 1: Starter**
- Click "Add product"
- Name: `Starter`
- Add price: $17.00, Recurring, Monthly → Copy the price ID (starts with `price_`)
- Add another price: $170.00, Recurring, Yearly → Copy the price ID
- Save these as `STRIPE_PRICE_STARTER_MONTHLY` and `STRIPE_PRICE_STARTER_YEARLY`

**Product 2: Pro**
- Name: `Pro`
- Monthly: $29.00 → Save as `STRIPE_PRICE_PRO_MONTHLY`
- Yearly: $290.00 → Save as `STRIPE_PRICE_PRO_YEARLY`

**Product 3: Agency**
- Name: `Agency`
- Monthly: $97.00 → Save as `STRIPE_PRICE_AGENCY_MONTHLY`
- Yearly: $970.00 → Save as `STRIPE_PRICE_AGENCY_YEARLY`

---

## Step 3: Setup Stripe Webhook

1. Go to https://dashboard.stripe.com/webhooks
2. Click "Add endpoint"
3. Enter your webhook URL: `https://yourdomain.com/webhooks/stripe`
4. Click "Select events" and add:
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.paid`
   - `invoice.payment_failed`
5. Click "Add endpoint"
6. Click "Reveal" under Signing secret
7. Copy the secret (starts with `whsec_`) → Save as `STRIPE_WEBHOOK_SECRET`

---

## Step 4: Create Meta App

Before using Unified.to, you need to create a Meta app:

**Create the App:**
1. Go to https://developers.facebook.com/apps
2. Click "Create App"
3. Name: Enter any name you want
4. Contact Email: Enter your email
5. Click "Ads and Monetisation"
6. Select "Measure ad performance data with Marketing API"
7. Business Portfolio: Click "I don't want to connect a business portfolio yet"
8. Requirements: Not needed
9. Click "Create App"

**Configure Facebook Login for Business:**
1. In your app sidebar, go to **Facebook Login for Business** → **Settings**
2. Enable the following:
   - ✅ Client OAuth Login
   - ✅ Web OAuth Login
   - ✅ Enforce HTTPS
   - ✅ Use Strict Mode for Redirect URIs
3. In **Valid OAuth Redirect URIs**, add: `https://api.unified.to/oauth/code`
4. Click "Save Changes"

**Configure App Domains:**
1. Go to **App Settings** → **Basic**
2. In **App Domains**, add: `unified.to`
3. Click "Save Changes"

---

## Step 5: Get Unified.to API Keys

1. In Unified.to dashboard, go to **Configurations** → **API Keys**
2. Click "Create API Key"
3. Copy the **Workspace ID** → Save as `UNIFIED_WORKSPACE_ID`
4. Copy the **API Token** → Save as `UNIFIED_API_TOKEN`
5. Set environment to Production → Save as `UNIFIED_ENV=Production`

---

## Step 6: Enable Meta Ads in Unified.to

1. In Unified.to dashboard, go to **Integrations**
2. Search for **Meta Ads**
3. For **Integration categories**, select:
   - ✅ Advertising
   - ✅ Passthrough
4. Configure OAuth credentials:
   - **OAuth 2 Client ID**: 
     - Go back to your Meta app → **App Settings** → **Basic**
     - Copy the **App ID**
     - Paste it into the OAuth 2 Client ID box in Unified.to
   - **OAuth 2 Client Secret**:
     - In the same Meta app page, copy the **App Secret**
     - Paste it into the OAuth 2 Client Secret box in Unified.to
5. Configure scopes:
   - Expand **Override OAuth 2 Scopes**
   - Find and enable `ads_report_read`
   - In **Override Meta Ads Scopes**, type: `ads_read`
6. Click **Save** or **Update**

---

## Step 7: Configure Unified.to OAuth Callback

1. In Unified.to dashboard, go to Settings → OAuth (or Developers)
2. Add your callback URL: `https://yourdomain.com/connect/callback`
3. Save the configuration

---

## Step 8: Get OpenAI API Key

1. Go to https://platform.openai.com/api-keys
2. Click "Create new secret key"
3. Copy the key (starts with `sk-proj-`) → Save as `OPENAI_API_KEY`
4. Make sure you have billing set up at https://platform.openai.com/account/billing

---

## Step 9: Create Environment File

Create a `.env` file in your project root with all the keys:

```bash
# Stripe Keys
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Stripe Price IDs
STRIPE_PRICE_STARTER_MONTHLY=price_...
STRIPE_PRICE_STARTER_YEARLY=price_...
STRIPE_PRICE_PRO_MONTHLY=price_...
STRIPE_PRICE_PRO_YEARLY=price_...
STRIPE_PRICE_AGENCY_MONTHLY=price_...
STRIPE_PRICE_AGENCY_YEARLY=price_...

# Unified.to (Meta Ads)
UNIFIED_WORKSPACE_ID=...
UNIFIED_API_TOKEN=...
UNIFIED_ENV=Production

# OpenAI
OPENAI_API_KEY=sk-proj-...

# Rails
RAILS_ENV=production
SECRET_KEY_BASE=<generate by running: rails secret>
```

---

## Step 10: Deploy on VPS (Ubuntu)

**1. Install dependencies:**
```bash
sudo apt update
sudo apt install -y ruby-full build-essential postgresql postgresql-contrib libpq-dev nodejs nginx
```

**2. Clone and setup application:**
```bash
cd /var/www
git clone <your-repo-url> adsetric
cd adsetric
bundle install
```

**3. Setup database:**
```bash
sudo -u postgres createdb adsetric_production
RAILS_ENV=production rails db:migrate
```

**4. Precompile assets:**
```bash
RAILS_ENV=production rails assets:precompile
```

**5. Create systemd service** (`/etc/systemd/system/adsetric.service`):
```ini
[Unit]
Description=Adsetric
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/adsetric
Environment=RAILS_ENV=production
EnvironmentFile=/var/www/adsetric/.env
ExecStart=/usr/local/bin/bundle exec puma -C config/puma.rb
Restart=always

[Install]
WantedBy=multi-user.target
```

**6. Start service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable adsetric
sudo systemctl start adsetric
```

---

## Step 11: Setup Nginx & SSL

**1. Create Nginx config** (`/etc/nginx/sites-available/adsetric`):
```nginx
server {
  listen 80;
  server_name yourdomain.com;
  
  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

**2. Enable site:**
```bash
sudo ln -s /etc/nginx/sites-available/adsetric /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

**3. Install SSL certificate:**
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

---

## Step 12: Test Everything

**Test Stripe Checkout:**
1. Go to `https://yourdomain.com/pricing`
2. Click "Get Started" on any plan
3. Use test card: `4242 4242 4242 4242` (any future date, any CVC)
4. Complete checkout
5. Check Stripe dashboard to see the subscription

**Test Meta Ads Connection:**
1. Login to your account
2. Go to Settings or Dashboard
3. Click "Connect Meta Account"
4. You'll be redirected to Meta to authorize
5. Select your Business account
6. Grant permissions
7. Select an ad account from the list
8. Verify it's connected in your dashboard

**Check Webhooks:**
1. Go to https://dashboard.stripe.com/webhooks
2. Click on your webhook
3. Check "Recent events" - should show successful deliveries

---

## Troubleshooting

**Stripe checkout not working:**
- Make sure `STRIPE_PUBLISHABLE_KEY` is set correctly
- Check browser console for errors
- Verify webhook URL is accessible (not behind login)

**Meta Ads connection failing:**
- Verify `UNIFIED_WORKSPACE_ID` and `UNIFIED_API_TOKEN` are correct
- Make sure Meta integration is enabled in Unified.to dashboard
- Check callback URL in Unified.to matches exactly: `https://yourdomain.com/connect/callback`
- Ensure user is Admin or Advertiser on the Meta ad account
- Check user has proper permissions in Meta Business Manager

**Webhooks failing:**
- Verify `STRIPE_WEBHOOK_SECRET` matches Stripe dashboard
- Check logs: `tail -f log/production.log`
- Make sure webhook endpoint is not behind authentication

**App not starting:**
- Check all environment variables are set in `.env`
- Verify database connection
- Check logs for errors: `journalctl -u adsetric -f`

---

## Useful Commands

**View logs:**
```bash
tail -f log/production.log
journalctl -u adsetric -f
```

**Restart app:**
```bash
sudo systemctl restart adsetric
```

**Update app:**
```bash
cd /var/www/adsetric
git pull
bundle install
RAILS_ENV=production rails db:migrate
RAILS_ENV=production rails assets:precompile
sudo systemctl restart adsetric
```

**Check app status:**
```bash
sudo systemctl status adsetric
```

---

**You're done! Your app should be live at https://yourdomain.com** 🚀
