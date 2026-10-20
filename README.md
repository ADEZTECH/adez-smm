# ADEZ SMM

Version 1.0.0. A standalone Node.js application serving plain HTML, CSS and browser JavaScript. MongoDB is accessed through Mongoose. There is no React frontend and no client-side provider API key.

## Included

* Registration, password login, account editing, password changes and Resend password resets.
* First registration becomes the sole super administrator, atomically.
* A ShakerGains catalog imported across all platform categories, service-specific order forms, single and bulk orders.
* Customer wallet, deposit history, order history, status refresh, cancellation and refill requests where supported.
* Automatic wallet refunds for confirmed rejected, canceled and partial orders.
* Kenyan customers use GiftedPay M-Pesa STK. Other customers are redirected to Paystack hosted checkout, initialized with `channels: ["card"]`.
* Admin pricing shows provider rates, converted cost, selling price, margin and profit for each service. Reports show overall and by-service sales, refunds, cost and estimated profit.
* Super-admin settings manage branding, registration, maintenance, exchange rates, minimum deposits and encrypted integration keys.
* Admin payment reconciliation, uncertain order resolution, access controls, audited balance adjustments and audit history.
* Responsive layouts, local SVG icons, animations and reduced-motion support.
* `vercel.json`, `render.yaml`, `.env.example` and an automated test suite.

## Quick start

Requirements: Node.js 22 or newer and a MongoDB replica set, such as MongoDB Atlas. A standalone MongoDB server without transaction support is deliberately refused because wallet operations must be atomic.

From the extracted `adez-smm` directory:

```sh
npm install
cp .env.example .env
# Fill .env locally. Do not commit it.
npm start
```

Alternatively use pnpm. `npm run dev` starts the watch-mode server. The default port is 3000; production hosts provide `PORT`.

Required environment configuration:

| Variable | Purpose |
| --- | --- |
| `MONGODB_URI` | Atlas or another MongoDB replica-set connection |
| `SESSION_SECRET` | Stable random string of at least 32 characters |
| `SETTINGS_ENCRYPTION_KEY` | Separate stable random encryption key, recommended |
| `APP_URL` | Your actual public HTTPS origin, without a path or trailing slash |
| `CRON_SECRET` | Stable random scheduler bearer secret |

Generate each random secret separately:

```sh
node -e "process.stdout.write(require('crypto').randomBytes(48).toString('hex'))"
```

The public page can load without MongoDB. In that state the API reports `configured: false`, and authentication and financial actions return a clear configuration error. It never substitutes fake accounts, money or services.

## Owner setup

1. Connect your MongoDB database and set the stable environment secrets.
2. Register your owner account **before opening the site to other people**. The first successful registration is automatically super admin. There is no publicly accessible role-selection field.
3. Open **Admin > Settings**. Set the site name, icon URL, support email and verified Resend sender email.
4. Enter ShakerGains, Resend, GiftedPay and Paystack keys. Stored keys are AES-256-GCM encrypted and write-only. Empty key fields preserve existing values.
5. Open **Admin > Pricing** and sync the catalog. The provider's actual currency is read from its balance endpoint.
6. Set the provider-to-KES conversion in Settings, then set each selling price and enable the desired services. A price is per 1,000 units or per package, as shown on the row.
7. Select a Paystack currency supported by your merchant account, and set its conversion to KES. KES to KES must be exactly 1. Currency support depends on your Paystack account, not the customer's location.
8. Configure the Paystack webhook at your actual HTTPS origin plus `/smm-api/webhooks/paystack`.
9. Verify real small payments, refunds, provider orders and reset emails before accepting customer funds.

Optional environment keys are `SHAKERGAINS_API_KEY`, `GIFTEDPAY_API_KEY`, `PAYSTACK_SECRET_KEY` and `RESEND_API_KEY`. Admin-stored keys take precedence. Changing the encryption key without re-entering stored integration keys makes those values unreadable. Do not casually rotate the encryption key.

Only super admins can change settings, prices, user access, balances or manually reconcile uncertain transactions. Other admins can read the administration reports and run reconciliation.

## Payments and accounting

All wallet balances, ledger amounts and deposit amounts are stored as integer minor units: `10000` is KES 100. Service rates are decimal major-currency values.

The gateway is selected from the customer's stored country, not a payment request's supplied country. Once financial activity exists, the country is locked; this prevents switching payment routes mid-transaction.

* GiftedPay base URL: `https://mpesa.gifted.co.ke/api`. The server calls `/payments/process`, then `/payments/verify`. It leaves GiftedPay's callback URL at the provider default so their verification ledger can record the M-Pesa result. Customer wallet polling, scheduled reconciliation and admin verification confirm payment.
* Paystack initializes hosted checkout with **card only**, then verifies reference, amount, currency, channel and account email. Signed webhooks trigger verification; the webhook body and browser return URL never directly credit money.
* Wallet credits are atomic and idempotent. Duplicate verification and repeated callbacks cannot create duplicate deposits.
* A network timeout during an order submission is not proof of failure. Funds remain held in `Submission uncertain` until the super admin checks ShakerGains and attaches the confirmed provider ID, or confirms it was not submitted and refunds it. The application never automatically resends such an order.
* Refunds return to the customer's wallet, not to M-Pesa or their card. Partial refunds are proportional to the provider-reported remaining quantity.
* Profit is an **estimate**: sales minus refunds minus provider cost. Gateway fees, taxes and foreign-exchange spread are not included. Initial cost uses the stored rate snapshot; valid provider status charges update the order cost.
* Administrator refunds target an absolute cumulative total, making retries safe. Balance adjustments require an idempotency key and are audited.

Provider documentation:

* `https://shakergainske.com/api`
* `https://pay.gifted.co.ke/api-guide`
* `https://paystack.com/docs/payments/accept-payments/`
* `https://resend.com/docs/api-reference/emails/send-email`

## Service coverage

The catalog importer has no hardcoded platform list: every returned category and service is stored and visible to the administrator.

Order forms support Default, Custom Comments, Mentions, Mentions with Hashtags, Mentions Custom List, Mentions Hashtag, Mentions User Followers, Mentions Media Likers, Custom Comments Package, Comment Likes, Poll, Comment Replies, Invites from Groups and Package.

Subscriptions, SEO and unknown provider-specific types are imported but deliberately not enabled for ordering in version1. Their extra field and recurring-charge contracts were not established by the public documentation used for this build. They must not be treated as ordinary one-time orders or charged speculatively. The pricing panel explicitly marks these as unsupported. Confirm those contracts with ShakerGains before adding adapters.

Sync preserves existing operator prices and enabled state, updates provider prices and marks removed upstream services unavailable. New services are hidden until reviewed and enabled. Scheduled jobs refresh the catalog approximately every 15 minutes when time remains after order and payment reconciliation. The last successful sync timestamp is shown; failures preserve the last verified catalog.

## Render

For a standalone repository with this directory's files at its root, use the included `render.yaml` to create a Node web service. Set `MONGODB_URI` and the actual HTTPS `APP_URL`. Render generates the session, encryption and scheduler secrets.

The blueprint uses a paid Starter service; review the selected service plan before creating it. No hosting account or service has been created by this project.

If publishing the complete Replit workspace instead of the extracted standalone package, add `rootDir: artifacts/adez-smm` under the service in `render.yaml`. If your repository contains a top-level `adez-smm` directory, use `rootDir: adez-smm`.

The long-running Node process reconciles orders and payments every minute. Set `ENABLE_SCHEDULER=false` if an external scheduler is managing reconciliation.

## Vercel

Import this standalone directory as the project root. For the complete workspace, set the Vercel Root Directory to `artifacts/adez-smm`. Select **Other** as the framework and do not configure a static output directory.

The included configuration routes requests to the Express serverless handler and traces `public/` into the deployment. Set the environment values above in Vercel, then set `APP_URL` to the actual HTTPS origin and redeploy. Atlas must permit your host's database connections.

Vercel has no persistent background process. The included cron runs daily and sends `Authorization: Bearer <CRON_SECRET>`. For prompt reconciliation without a customer online, configure a supported more frequent scheduler for your hosting plan, or an external authenticated scheduler:

```text
GET /smm-api/jobs/cron
Authorization: Bearer <your CRON_SECRET>
```

Jobs use database leases, request timeouts and a bounded processing budget. Customer status refreshes and wallet polling work independently of cron; Paystack webhooks also confirm deposits. High-volume catalogs or financial reconciliation are better suited to a long-running Node service. Function timeout limits must accommodate the job budget; set the serverless function maximum duration to at least 60 seconds on a plan that supports it.

Vercel and Render deployment files are supplied, but deployments to those accounts have **not** been performed or tested.

## Security notes

* Scrypt password hashing, random database-backed sessions, hashed reset tokens and session invalidation after password changes.
* HttpOnly, SameSite cookies; Secure cookies in production.
* CSRF validation, origin checks, role checks, per-user ownership checks and database-backed rate limiting.
* Strict input validation and explicit scalar MongoDB filters against operator injection.
* Helmet security headers and Content Security Policy with no inline scripts or external executable browser dependencies.
* Public assets only. Environment files and server code are not exposed through static serving.
* No provider credentials or passwords in application request logs, customer responses or audit entries.
* Signed Paystack webhooks, independent server-side payment verification, transactions and unique ledger references.
* Review `TRUST_PROXY_HOPS` for your reverse-proxy topology. Incorrect proxy trust can weaken IP-based rate limits.
* Keep first-owner registration private until your owner account exists. Review backups, database privileges and business/provider compliance before launch.

This is not a guarantee against every attack or a substitute for a production security review.

## Tests

```sh
npm install
npm run build
npm test
```

The suite starts and stops an **isolated MongoDB replica set** and uses controlled provider responses. It does not contact or charge your live gateways. The first test run may download a MongoDB 8.0.5 binary. The host must support that binary. To use an existing compatible binary, set `MONGOMS_SYSTEM_BINARY` to its path.

The delivered build passed 36 tests. See `docs/test-report.md`. Public desktop and mobile screens were inspected. Signed-in page rendering was not verified in a browser; protected route behavior was exercised by API integration tests. Real provider credentials, email delivery, real-money payments and real ShakerGains fulfilment have **not** been live tested.

## Layout

```text
server/            Express application, Mongoose models and provider adapters
server/lib/        Input, password, encryption and CSRF helpers
public/index.html  HTML shell
public/css/        Responsive styling and animation
public/js/         Browser ES modules and pages
public/assets/     Local SVG assets
test/              Security and API integration tests
docs/              API contract and verification report
scripts/check.js   Syntax and distribution checks
vercel.json        Serverless routing
render.yaml        Long-running Node hosting
```
