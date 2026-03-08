

# cPanel / WHMCS / DirectAdmin — Behind-the-Scenes Integration Plan

## What the User Sees (Frontend)

The user never interacts with WHMCS, WHM, or DirectAdmin directly. Everything happens through the existing Abancool dashboard pages. Here is the flow:

```text
┌─────────────────────────────────────────────────────────────┐
│  PUBLIC SITE                                                │
│  /hosting → Pick plan → "Order Now"                         │
│  /domains → Search domain → "Add to Cart" → "Checkout"      │
│         ↓                                                   │
│  Not logged in? → /client/login (email/password or Google)  │
│         ↓                                                   │
│  DASHBOARD                                                  │
│  /client/dashboard/payments → Pay invoice (M-Pesa / Stripe) │
│         ↓                                                   │
│  Payment confirmed → Edge function activates everything     │
│         ↓                                                   │
│  /client/dashboard/cpanel → "Open cPanel" button            │
│  (SSO auto-login — no username/password needed)             │
└─────────────────────────────────────────────────────────────┘
```

## What Happens Behind the Scenes (Edge Functions → PHP/WHMCS)

All automation is handled by **backend functions** that call your WHMCS/WHM/DirectAdmin APIs. The React frontend never talks to these servers directly.

### Architecture

```text
React Frontend
     │
     ▼
Lovable Cloud (Edge Functions)
     │
     ├──→ WHMCS API (billing, client sync, invoices)
     ├──→ WHM API (cPanel account provisioning, stats, SSO)
     └──→ DirectAdmin API (alternative panel provisioning)
```

### Edge Functions to Build

| Function | Trigger | What It Does |
|---|---|---|
| `provision-hosting` | Called after payment confirmed | Creates cPanel/DA account via WHM or DirectAdmin API. Updates `hosting_orders` with cpanel_url, username, expiry. Optionally syncs client to WHMCS. |
| `cpanel-sso` | User clicks "Open cPanel" | Calls WHM `create_user_session` API, returns one-time login URL. Frontend opens it in new tab. |
| `cpanel-stats` | cPanel dashboard page loads | Calls WHM `accountsummary` API, returns disk/cpu/email/db usage as JSON. |
| `payment-callback` | M-Pesa/Stripe webhook | Verifies payment, marks invoice paid, triggers `provision-hosting` logic. |
| `whmcs-sync` | After signup or payment | Syncs client record and order to WHMCS for billing/support tracking. |

### Secrets Required

These secrets need to be added before implementation:

- `WHM_HOST` — WHM server hostname (e.g. `server.abancool.com`)
- `WHM_TOKEN` — WHM API token for root user
- `WHM_PORT` — Usually `2087`
- `WHMCS_URL` — WHMCS API endpoint (e.g. `https://billing.abancool.com/includes/api.php`)
- `WHMCS_API_IDENTIFIER` — WHMCS API credential
- `WHMCS_API_SECRET` — WHMCS API credential
- `DIRECTADMIN_HOST` — DirectAdmin server (optional, if using DA instead of cPanel)
- `DIRECTADMIN_API_KEY` — DirectAdmin API key
- `MPESA_CONSUMER_KEY`, `MPESA_CONSUMER_SECRET`, `MPESA_SHORTCODE`, `MPESA_PASSKEY` — For M-Pesa STK push
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` — For Stripe payments

### Detailed Flow: Purchase → Provisioning

1. **User picks plan** on `/hosting` or `/client/dashboard/hosting`
2. **Invoice created** in `invoices` table, order created in `hosting_orders` (status: `pending`)
3. **User redirected** to `/client/dashboard/payments` — sees unpaid invoice
4. **User pays** via M-Pesa STK push or Stripe
5. **Callback edge function** receives confirmation:
   - Updates `payments.status = 'completed'`
   - Updates `invoices.status = 'paid'`
   - Calls **WHM API** `createacct` to provision cPanel account (or DirectAdmin `CMD_API_ACCOUNT_USER` to create DA account)
   - Optionally calls **WHMCS API** `AddClient` + `AddOrder` + `AcceptOrder` to sync billing
   - Updates `hosting_orders`: `status = 'active'`, `cpanel_url`, `cpanel_username`, `starts_at`, `expires_at`
6. **User visits cPanel page** — sees stats and "Open cPanel" button
7. **"Open cPanel" click** → calls `cpanel-sso` edge function → WHM generates session URL → user auto-logged into cPanel in new tab

### Detailed Flow: cPanel SSO (No Password Needed)

```text
User clicks "Open cPanel"
     │
     ▼
Frontend calls edge function: cpanel-sso
     │
     ▼
Edge function calls WHM API:
  POST https://{WHM_HOST}:2087/json-api/create_user_session
  Headers: Authorization: whm root:{WHM_TOKEN}
  Params: user={cpanel_username}&service=cpaneld
     │
     ▼
WHM returns: { url: "https://server:2083/cpsess123456/..." }
     │
     ▼
Frontend opens URL in new tab → user is logged into cPanel
```

### Detailed Flow: cPanel Stats (Real Usage Data)

The current `ClientCpanel.tsx` shows hardcoded stats. This will be replaced:

```text
cPanel page loads
     │
     ▼
Frontend calls edge function: cpanel-stats
     │
     ▼
Edge function calls WHM API:
  GET https://{WHM_HOST}:2087/json-api/accountsummary?user={cpanel_username}
     │
     ▼
Returns: { disk_used, disk_limit, email_count, db_count, ... }
     │
     ▼
Frontend renders real progress bars with actual usage
```

### Frontend Changes

1. **`ClientCpanel.tsx`** — Replace hardcoded stats with edge function call to `cpanel-stats`. Replace "Open cPanel" button to call `cpanel-sso` edge function and open returned URL. Remove iframe embed (SSO is better).

2. **`ClientPayments.tsx`** — Wire "Send STK" button to call `payment-mpesa` edge function (Daraja STK push). Add Stripe card payment option calling `payment-stripe` edge function.

3. **`ClientHosting.tsx`** — No major changes needed (already creates orders/invoices).

4. **`DashboardOverview.tsx`** — Already reads from DB, will reflect real data once provisioning is live.

5. **`HostingPage.tsx`** — Already creates invoices and redirects to payment. No changes needed.

### WHMCS Sync (Optional but Recommended)

WHMCS acts as the billing backend. After each signup/payment:
- `AddClient` — create client in WHMCS
- `AddOrder` — create hosting order
- `AcceptOrder` — activate the order
- WHMCS then handles renewal reminders, suspension on non-payment, etc.

The React dashboard remains the primary UI. WHMCS runs headless.

### Database Changes

**Add UPDATE RLS policy** on `hosting_orders` and `invoices` so edge functions (using service role key) can update statuses after payment. Currently these tables only allow SELECT and INSERT.

Migration needed:
```sql
-- Allow service-role updates (edge functions use service role)
-- No user-facing UPDATE policy needed; edge functions bypass RLS with service role key
```

No new tables needed — existing schema covers everything.

### Implementation Order

1. Add required secrets (WHM, WHMCS, M-Pesa, Stripe)
2. Create `payment-callback` edge function (M-Pesa + Stripe webhooks)
3. Create `provision-hosting` edge function (WHM createacct)
4. Create `cpanel-sso` edge function
5. Create `cpanel-stats` edge function
6. Update `ClientCpanel.tsx` to use SSO + real stats
7. Update `ClientPayments.tsx` to call real payment edge functions
8. Create `whmcs-sync` edge function (optional)

