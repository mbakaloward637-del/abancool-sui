

# Kiro Prompt: cPanel / WHMCS / DirectAdmin Backend Integration

Here is your complete, copy-paste-ready prompt for Kiro:

---

```text
PROJECT: Abancool Technology — cPanel / WHMCS / DirectAdmin Behind-the-Scenes Integration

Build PHP REST API endpoints that integrate with WHM (cPanel), WHMCS, and DirectAdmin APIs.
The React frontend calls these endpoints via fetch(). All responses are JSON.
These endpoints handle hosting provisioning, panel SSO, usage stats, and billing sync.

────────────────────────────────────────────
DIRECTORY STRUCTURE (add to existing backend/)
────────────────────────────────────────────

backend/
├── services/
│   ├── WHMService.php            # WHM/cPanel API wrapper
│   ├── DirectAdminService.php    # DirectAdmin API wrapper
│   └── WHMCSService.php          # WHMCS API wrapper
├── api/
│   ├── cpanel/
│   │   ├── sso.php               # GET /api/cpanel/sso
│   │   ├── stats.php             # GET /api/cpanel/stats
│   │   └── status.php            # GET /api/cpanel/status
│   ├── provisioning/
│   │   └── provision.php         # POST /api/provisioning/provision
│   ├── payments/
│   │   ├── mpesa-stk.php         # POST /api/payments/mpesa
│   │   ├── mpesa-callback.php    # POST /api/payments/mpesa/callback
│   │   ├── stripe-intent.php     # POST /api/payments/stripe/intent
│   │   └── stripe-webhook.php    # POST /api/payments/stripe/webhook
│   └── whmcs/
│       └── sync.php              # POST /api/whmcs/sync

────────────────────────────────────────────
SERVICE: WHMService.php
────────────────────────────────────────────

Class WHMService handles all WHM API calls. Constructor takes WHM_HOST, WHM_PORT, WHM_TOKEN from env.
Auth header for all requests: "Authorization: whm root:{WHM_TOKEN}"
Base URL: https://{WHM_HOST}:{WHM_PORT}/json-api/

Methods:

1. createAccount(string $username, string $domain, string $plan, string $contactemail): array
   - POST /json-api/createacct
   - Params: username, domain, plan, contactemail, reseller=0
   - Returns: { success: bool, cpanel_url: string, username: string }
   - Generate username: first 8 chars of domain name (alphanumeric only), check uniqueness

2. suspendAccount(string $username, string $reason): array
   - POST /json-api/suspendacct
   - Params: user={username}, reason={reason}

3. unsuspendAccount(string $username): array
   - POST /json-api/unsuspendacct
   - Params: user={username}

4. terminateAccount(string $username): array
   - POST /json-api/removeacct
   - Params: user={username}

5. getAccountStats(string $username): array
   - GET /json-api/accountsummary?user={username}
   - Parse and return: {
       disk_used_mb, disk_limit_mb,
       bandwidth_used_mb, bandwidth_limit_mb,
       email_accounts, email_limit,
       databases, database_limit,
       addon_domains, parked_domains,
       suspended: bool, plan: string
     }

6. createSession(string $username): string
   - POST /json-api/create_user_session
   - Params: user={username}, service=cpaneld
   - Returns: the SSO URL string (one-time login URL)
   - This is the key method — user clicks "Open cPanel" → frontend calls
     GET /api/cpanel/sso → this method runs → returns URL → frontend
     opens URL in new tab → user is auto-logged into cPanel

7. listAccounts(): array
   - GET /json-api/listaccts
   - Returns array of all accounts on server

────────────────────────────────────────────
SERVICE: DirectAdminService.php
────────────────────────────────────────────

Class DirectAdminService handles DirectAdmin API calls.
Constructor takes DA_HOST, DA_PORT (default 2222), DA_ADMIN_USER, DA_ADMIN_PASSWORD or DA_API_KEY from env.
Auth: Basic auth with admin credentials, or Login-Key header.
Base URL: https://{DA_HOST}:{DA_PORT}/

Methods:

1. createAccount(string $username, string $domain, string $package, string $email, string $password): array
   - POST /CMD_API_ACCOUNT_USER
   - Params: action=create, add=Submit, username, domain, package, email, passwd={password}, passwd2={password}, ip=shared, notify=yes
   - Returns: { success: bool, username: string }

2. suspendAccount(string $username): array
   - POST /CMD_API_SELECT_USERS
   - Params: location=CMD_SELECT_USERS, suspend=Suspend, select0={username}

3. unsuspendAccount(string $username): array
   - POST /CMD_API_SELECT_USERS
   - Params: location=CMD_SELECT_USERS, unsuspend=Unsuspend, select0={username}

4. deleteAccount(string $username): array
   - POST /CMD_API_SELECT_USERS
   - Params: confirmed=Confirm, delete=yes, select0={username}

5. getAccountStats(string $username): array
   - GET /CMD_API_SHOW_USER_USAGE?user={username}
   - Parse and return same structure as WHM stats

6. createLoginUrl(string $username): string
   - POST /CMD_API_LOGIN_KEYS
   - Create a one-time login key for the user
   - Build URL: https://{DA_HOST}:{DA_PORT}/CMD_LOGIN?username={username}&key={login_key}
   - This is the DirectAdmin equivalent of cPanel SSO
   - When user clicks "Open DirectAdmin" → this URL opens in new tab

7. listAccounts(): array
   - GET /CMD_API_SHOW_ALL_USERS
   - Returns array of all user accounts

────────────────────────────────────────────
SERVICE: WHMCSService.php
────────────────────────────────────────────

Class WHMCSService handles WHMCS API calls (headless billing sync).
Constructor takes WHMCS_URL, WHMCS_API_IDENTIFIER, WHMCS_API_SECRET from env.
All requests: POST to WHMCS_URL with params:
  identifier={WHMCS_API_IDENTIFIER}&secret={WHMCS_API_SECRET}&action={action}&responsetype=json

Methods:

1. addClient(array $data): int
   - action=AddClient
   - Params: firstname, lastname, email, phonenumber, password2 (auto-generated)
   - Returns: WHMCS client ID

2. getClientByEmail(string $email): ?array
   - action=GetClients, search={email}
   - Returns client data or null

3. addOrder(int $clientId, int $productId, string $domain, string $billingCycle, string $paymentMethod): int
   - action=AddOrder
   - Params: clientid, pid={productId}, domain, billingcycle (monthly|annually), paymentmethod (mailin)
   - Returns: WHMCS order ID

4. acceptOrder(int $orderId): bool
   - action=AcceptOrder
   - Params: orderid
   - This activates the order in WHMCS

5. addInvoicePayment(int $invoiceId, string $transactionId, float $amount, string $gateway): bool
   - action=AddInvoicePayment
   - Params: invoiceid, transid, amount, gateway (mpesa|stripe)

6. getClientProducts(int $clientId): array
   - action=GetClientsProducts
   - Params: clientid
   - Returns list of products/services

────────────────────────────────────────────
API ENDPOINT: GET /api/cpanel/sso
────────────────────────────────────────────

Requires: JWT auth (logged-in user)
Logic:
1. Get user ID from JWT
2. Query hosting_orders WHERE user_id = {user_id} AND status = 'active' LIMIT 1
3. If no active order → return { error: "No active hosting", redirect: "/client/dashboard/hosting" }
4. Read hosting_orders.panel_type column:
   - If 'cpanel' → use WHMService::createSession(cpanel_username) → return { url, panel: "cpanel" }
   - If 'directadmin' → use DirectAdminService::createLoginUrl(cpanel_username) → return { url, panel: "directadmin" }
5. Frontend receives URL → window.open(url, '_blank')
6. User is auto-logged into their panel. No credentials needed.

────────────────────────────────────────────
API ENDPOINT: GET /api/cpanel/stats
────────────────────────────────────────────

Requires: JWT auth
Logic:
1. Get active hosting order for user
2. If no active order → return null
3. Based on panel_type:
   - 'cpanel' → WHMService::getAccountStats(cpanel_username)
   - 'directadmin' → DirectAdminService::getAccountStats(cpanel_username)
4. Return JSON: {
     disk: { used_mb, limit_mb, percent },
     bandwidth: { used_mb, limit_mb, percent },
     email: { count, limit },
     databases: { count, limit },
     plan_name, status, panel_type,
     expires_at
   }

────────────────────────────────────────────
API ENDPOINT: POST /api/provisioning/provision
────────────────────────────────────────────

Called internally after payment confirmation. NOT user-facing.
Input: { hosting_order_id: int }
Logic:
1. Load hosting_order with plan details and user info
2. Generate unique username (first 8 alphanumeric chars of domain)
3. Determine panel_type from hosting_plans.panel_type column (default: 'cpanel')
4. If panel_type = 'cpanel':
   - WHMService::createAccount(username, domain, plan_slug, user_email)
   - cpanel_url = "https://{WHM_HOST}:2083"
5. If panel_type = 'directadmin':
   - Generate random password
   - DirectAdminService::createAccount(username, domain, package, email, password)
   - cpanel_url = "https://{DA_HOST}:2222"
6. Update hosting_orders SET:
   - status = 'active'
   - cpanel_username = {generated username}
   - cpanel_url = {panel url}
   - starts_at = NOW()
   - expires_at = NOW() + billing_cycle interval (1 month or 1 year)
7. Optionally sync to WHMCS:
   - WHMCSService::addClient or getClientByEmail
   - WHMCSService::addOrder
   - WHMCSService::acceptOrder
8. Log to activity_log
9. Send email confirmation to user (optional)

────────────────────────────────────────────
API ENDPOINT: POST /api/payments/mpesa (STK Push)
────────────────────────────────────────────

Requires: JWT auth
Input: { invoice_id: int, phone: string }
Logic:
1. Load invoice, verify it belongs to user and status = 'unpaid'
2. Format phone: ensure 254XXXXXXXXX format
3. Get M-Pesa OAuth token:
   GET https://api.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials
   Auth: Basic base64(MPESA_CONSUMER_KEY:MPESA_CONSUMER_SECRET)
4. Send STK push:
   POST https://api.safaricom.co.ke/mpesa/stkpush/v1/processrequest
   Body: {
     BusinessShortCode: MPESA_SHORTCODE,
     Password: base64(MPESA_SHORTCODE + MPESA_PASSKEY + timestamp),
     Timestamp: YYYYMMDDHHmmss,
     TransactionType: "CustomerPayBillOnline",
     Amount: invoice.amount,
     PartyA: phone,
     PartyB: MPESA_SHORTCODE,
     PhoneNumber: phone,
     CallBackURL: MPESA_CALLBACK_URL,
     AccountReference: invoice.invoice_number,
     TransactionDesc: "Payment for " + invoice.service_description
   }
5. Create payments record: status='pending', method='mpesa'
6. Return { success: true, checkout_request_id }

────────────────────────────────────────────
API ENDPOINT: POST /api/payments/mpesa/callback
────────────────────────────────────────────

No auth (Safaricom calls this). IP whitelist recommended.
Logic:
1. Parse Safaricom callback JSON
2. If ResultCode = 0 (success):
   a. Extract MpesaReceiptNumber, Amount, PhoneNumber
   b. Find payment by checkout_request_id
   c. Update payment: status='completed', mpesa_receipt, paid_at=NOW()
   d. Update invoice: status='paid', paid_at=NOW()
   e. Find related hosting_order
   f. Call POST /api/provisioning/provision with hosting_order_id
3. If ResultCode != 0: Update payment status='failed'
4. Return { ResultCode: 0, ResultDesc: "Accepted" }

────────────────────────────────────────────
API ENDPOINT: POST /api/payments/stripe/intent
────────────────────────────────────────────

Requires: JWT auth
Input: { invoice_id: int }
Logic:
1. Load invoice, verify ownership and unpaid status
2. Create Stripe PaymentIntent:
   \Stripe\PaymentIntent::create([
     'amount' => invoice.amount * 100 (cents),
     'currency' => 'kes',
     'metadata' => ['invoice_id' => invoice.id, 'user_id' => user.id]
   ])
3. Create payments record: status='pending', method='stripe', stripe_payment_id
4. Return { client_secret: paymentIntent->client_secret }

────────────────────────────────────────────
API ENDPOINT: POST /api/payments/stripe/webhook
────────────────────────────────────────────

No auth (Stripe calls this). Verify signature with STRIPE_WEBHOOK_SECRET.
Logic:
1. Verify webhook signature
2. If event = payment_intent.succeeded:
   a. Extract invoice_id from metadata
   b. Update payment: status='completed', paid_at=NOW()
   c. Update invoice: status='paid', paid_at=NOW()
   d. Find related hosting_order
   e. Call POST /api/provisioning/provision with hosting_order_id
3. Return 200

────────────────────────────────────────────
API ENDPOINT: POST /api/whmcs/sync
────────────────────────────────────────────

Internal only (called by provisioning flow).
Input: { user_id: int, hosting_order_id: int }
Logic:
1. Load user and hosting order with plan
2. Check if WHMCS client exists: WHMCSService::getClientByEmail(user.email)
3. If not → WHMCSService::addClient(user data) → get whmcs_client_id
4. WHMCSService::addOrder(whmcs_client_id, plan.whmcs_product_id, domain, billing_cycle, 'mailin')
5. WHMCSService::acceptOrder(order_id)
6. Store whmcs_client_id on users table (add column if needed)

────────────────────────────────────────────
DATABASE CHANGES NEEDED
────────────────────────────────────────────

ALTER TABLE hosting_plans ADD COLUMN panel_type ENUM('cpanel','directadmin') DEFAULT 'cpanel';
ALTER TABLE hosting_plans ADD COLUMN whmcs_product_id INT NULL;

ALTER TABLE hosting_orders ADD COLUMN panel_type ENUM('cpanel','directadmin') DEFAULT 'cpanel';

ALTER TABLE users ADD COLUMN whmcs_client_id INT NULL;

ALTER TABLE payments ADD COLUMN checkout_request_id VARCHAR(100) NULL;

────────────────────────────────────────────
ENVIRONMENT VARIABLES (add to .env)
────────────────────────────────────────────

# WHM/cPanel
WHM_HOST=server.abancool.com
WHM_PORT=2087
WHM_TOKEN=your-whm-api-token

# DirectAdmin
DA_HOST=da.abancool.com
DA_PORT=2222
DA_ADMIN_USER=admin
DA_ADMIN_PASSWORD=
DA_API_KEY=

# WHMCS
WHMCS_URL=https://billing.abancool.com/includes/api.php
WHMCS_API_IDENTIFIER=
WHMCS_API_SECRET=

# M-Pesa
MPESA_CONSUMER_KEY=
MPESA_CONSUMER_SECRET=
MPESA_SHORTCODE=174379
MPESA_PASSKEY=
MPESA_CALLBACK_URL=https://api.abancool.com/api/payments/mpesa/callback
MPESA_ENV=sandbox

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

────────────────────────────────────────────
FRONTEND BEHAVIOR (React — already built)
────────────────────────────────────────────

ClientCpanel.tsx changes:
1. On page load → GET /api/cpanel/status
   - If no active hosting → show "Get Hosting" paywall
   - If active → GET /api/cpanel/stats → render real usage bars
2. "Open cPanel" / "Open DirectAdmin" button:
   - GET /api/cpanel/sso → receive { url, panel }
   - window.open(url, '_blank') → user auto-logged into panel
   - Button label changes based on panel_type from stats response

ClientPayments.tsx changes:
1. M-Pesa: POST /api/payments/mpesa with { invoice_id, phone }
   - Show "STK push sent, check your phone" message
   - Poll invoice status every 5 seconds until paid
2. Stripe: POST /api/payments/stripe/intent
   - Use Stripe.js confirmCardPayment with client_secret
   - On success, redirect to dashboard

────────────────────────────────────────────
COMPLETE FLOW: User Orders Hosting → Gets Panel Access
────────────────────────────────────────────

1. User visits /hosting → clicks "Order Now" on a plan
2. Frontend creates invoice + hosting_order (pending) in DB
3. User redirected to /client/dashboard/payments
4. User pays via M-Pesa STK push or Stripe card
5. Payment callback hits /api/payments/mpesa/callback or stripe/webhook
6. Callback confirms payment → calls /api/provisioning/provision
7. Provision endpoint:
   a. Reads plan.panel_type (cpanel or directadmin)
   b. Creates account on WHM or DirectAdmin server
   c. Updates hosting_order: active, cpanel_username, cpanel_url, expires_at
   d. Syncs to WHMCS (optional)
8. User visits /client/dashboard/cpanel
9. Page loads → GET /api/cpanel/stats → shows real disk/email/db usage
10. User clicks "Open cPanel" or "Open DirectAdmin"
11. GET /api/cpanel/sso → returns auto-login URL
12. Browser opens new tab → user is inside their panel, no password needed

────────────────────────────────────────────
SECURITY
────────────────────────────────────────────

- All /api/cpanel/* and /api/provisioning/* require valid JWT
- /api/payments/mpesa/callback: whitelist Safaricom IPs
- /api/payments/stripe/webhook: verify Stripe signature
- /api/whmcs/sync: internal only, not exposed to frontend
- WHM/DA credentials never exposed to frontend
- Rate limit STK push: max 1 per minute per user
- All database queries use PDO prepared statements
```

Copy and paste this entire prompt into Kiro. It covers all three panel integrations (cPanel via WHM, DirectAdmin, WHMCS), all payment flows (M-Pesa STK push, Stripe), auto-provisioning after payment, SSO auto-login for both panels, real usage stats, and the required database schema changes.

