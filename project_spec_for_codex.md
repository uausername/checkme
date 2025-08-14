# Project Spec for Codex — "Check-Status" (Чек‑Статус)

## Purpose / high-level idea

**Check-Status** is a lightweight single‑page web app (MVP) that generates attractive, shareable PNG "checks" (fake receipt/receipt‑like images) for end users. The product targets low-income users who want an inexpensive, playful way to signal small status moments (e.g., "paid mom for coffee", "donated 20 UAH"). The app *appears* utilitarian (has amount, date, category, QR‑like visual), but its main purpose is entertainment and social sharing.

Key constraints and goals:

- Digital product, extremely simple distribution (static site), minimal or no upfront marketing.
- Free base product; monetization via microsales of premium styles/skins/cosmetic packs priced very low (micropayments).
- Organic growth via built‑in watermark/CTA embedded in generated images that encourages recipients to create their own checks.
- Works on weak devices and offline-friendly when possible.
- Minimum viable server footprint: initially none needed to use base features; optional small server for payments, code issuance and redemption, analytics.

---

## Product features (MVP)

- Single page `index.html` that renders a form and a canvas preview and can export PNG.
- Inputs: amount, currency, recipient (to), note, category, date, sender name/nick, watermark CTA.
- Templates: two free templates and one visually locked premium template (placeholder).
- Download PNG and share via Web Share API (mobile).
- Watermark CTA contains link/shortcode to drive referrals.

Optional server features (recommended for monetization):

- Simple backend to create and deliver one‑time unlock codes or signed tokens after payment.
- Database to store codes, issued payments, redemptions and basic analytics.
- Webhook endpoint to process payment provider notifications and generate/deliver codes.
- Redemption endpoint to validate codes and unlock premium styles.

---

## Target tech stack (recommended)

- Frontend: plain HTML/CSS/JavaScript (single file `index.html`) — keep it dependency free so Codex can continue easily.
- Backend (optional): Node.js (Express) — minimal server template included in spec examples. Use SQLite for small scale or PostgreSQL if scaling is expected.
- Payments: Stripe (recommended), fallback options: PayPal, Paddle, Gumroad, or Telegram Payments if audience is Telegram heavy.
- Delivery: email (SendGrid/Mailgun) or Telegram Bot API for immediate code delivery.

---

## File structure (recommended repo layout)

```
/ (repo root)
  ├─ index.html                 # main SPA file (already provided)
  ├─ README.md                  # user‑facing instructions and deploy notes
  ├─ PROJECT_SPEC_FOR_CODEX.md  # (this file)
  ├─ server/                    # optional backend
  │   ├─ index.js               # express server (webhook + redeem APIs)
  │   ├─ db.js                  # db helper (SQLite or PG wrapper)
  │   ├─ package.json
  │   └─ migrations/            # SQL schema and seeds
  ├─ scripts/                   # utility scripts (generate codes, sign tokens)
  └─ assets/                    # optional images/fonts
```

---

## Database schema (SQLite minimal)

```
CREATE TABLE codes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT UNIQUE,
  style_id TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  issued_to TEXT,
  issued_by_payment_id TEXT,
  used INTEGER DEFAULT 0,
  used_at DATETIME,
  expires_at DATETIME
);

CREATE TABLE payments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  provider TEXT,
  provider_payment_id TEXT UNIQUE,
  amount INTEGER,
  currency TEXT,
  customer_email TEXT,
  metadata TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Notes:

- `issued_to` can be an email or a telegram chat id; store how delivered in metadata.
- `style_id` ties code to what it unlocks.

---

## Code issuance: two approaches

### A — Persistent codes (DB backed)

1. On webhook (payment success) generate a random code (8–12 chars), insert to `codes` with `issued_to` and `issued_by_payment_id`.
2. Deliver the code to the buyer (Telegram message or email).
3. The code is marked `used = 1` on redemption.

Pros: easy to revoke, can track usage, simple to debug. Cons: requires storing data and slight server maintenance.

### B — Signed tokens (stateless, no DB)

1. Server creates payload `{ style_id, exp, rnd }`, signs it with HMAC secret into `token = base64(payload).sig`.
2. Deliver `token` to buyer. On redemption, server verifies signature + expiry.

Pros: no DB, trivial horizontal scaling.\
Cons: cannot individually revoke tokens without blacklisting and harder to track usage.

---

## Payment flows (summary)

### Option 1 — Stripe Checkout + webhook (recommended)

- Frontend hits your server `/create-checkout-session` (or directly uses Checkout with serverless), passing `style_id` as metadata.
- User completes Stripe Checkout. Stripe calls your `/webhook` with `checkout.session.completed`.
- Server validates webhook signature, generates code / token and stores it in DB or signs token.
- Deliver code via email or Telegram.
- Provide a `/redeem` API to change `used` flag and unlock style for that user.

### Option 2 — Gumroad / Paddle / Gumroad-like (minimal code)

- Use Gumroad’s product delivery mechanism (upload digital file or use their license key feature).
- You get very low‑effort delivery but less control and fees higher.

### Option 3 — Telegram Payments or Bot-driven UX

- If your users are in Telegram, the bot can create a payment link/invoice.
- After successful payment, bot receives confirmation and posts the code directly in the user chat.

### Option 4 — PayPal (webhook or IPN) / other local providers

- Similar to Stripe but implementation details and verification differ. Use webhook/IPN signature verification.

---

## API design (suggested endpoints)

```
POST /create-checkout-session   # (optional) create Stripe Checkout session (returns url)
POST /webhook                   # payment provider webhook (stripe signature verification)
POST /redeem                    # redeem a code or token (body: { code | token, user_id })
GET  /codes?status=issued       # admin: list codes
POST /admin/revoke-code         # admin revoke
```

Behavior details:

- `/webhook` must verify signature and not allow replay (use created\_at & provider\_payment\_id uniqueness).
- `/redeem` validates code/token and returns `{ ok: true, style_id }` or `{ error }`.

---

## Security and operational notes

- Always verify webhook signatures (Stripe has `stripe-signature`).
- Keep secrets in env variables or secret manager: `STRIPE_SECRET`, `STRIPE_WEBHOOK_SECRET`, `TG_BOT_TOKEN`, `SIGN_SECRET`.
- Serve webhook over HTTPS; use `ngrok` for local dev testing.
- Rate limit `redeem` endpoint and add brute‑force protection (codes are short).
- Use random high-entropy code generation; avoid confusing characters.
- If using signed tokens, rotate signing keys and maintain ability to blacklist tokens if needed.

---

## Example code snippets (Node.js)

### Generate an 8‑char human‑friendly code

```js
const crypto = require('crypto');
function genCode(len = 8) {
  const alphabet = 'ABCDEFGHJKMNPQRSTUVWXYZ23456789';
  const bytes = crypto.randomBytes(len);
  let s = '';
  for (let i = 0; i < len; i++) s += alphabet[bytes[i] % alphabet.length];
  return s;
}
```

### Sign payload (stateless token)

```js
const crypto = require('crypto');
function signPayload(payload, secret) {
  const data = Buffer.from(JSON.stringify(payload)).toString('base64url');
  const sig = crypto.createHmac('sha256', secret).update(data).digest('base64url');
  return `${data}.${sig}`;
}
function verifyToken(token, secret) {
  const [data, sig] = token.split('.');
  const expected = crypto.createHmac('sha256', secret).update(data).digest('base64url');
  if (!timingSafeEqual(expected, sig)) return null;
  return JSON.parse(Buffer.from(data, 'base64url').toString());
}
```

---

## Frontend UX for purchases

- Prefer the following flow for best UX: user clicks "Buy style" → open a small modal asking for optional email or telegram id → call server to create Checkout session → redirect to provider → on success, show a friendly thank you screen that says "we've sent your code".
- If the user provides Telegram username/ID during purchase, add it to `session.metadata` so webhook can route delivery to that chat.

---

## Testing & QA

- Use Stripe test keys + test webhook signing secret to validate production‑like flows.
- Create unit tests around `genCode`, `signPayload`/`verifyToken` and DB operations.
- Manual tests: create checkout session, simulate webhook, ensure code is saved and delivered, redeem code in `/redeem` and verify flags.

---

## Deployment suggestions

- Static site (`index.html`) → serve via GitHub Pages / Cloudflare Pages.
- Backend server → small VPS (DigitalOcean) / serverless (AWS Lambda via Serverless framework / Vercel / Render). If using webhooks prefer an always‑on server with a stable URL and HTTPS.
- Database → start with SQLite for minimal cost, migrate to Postgres if concurrency grows.

---

## Admin & analytics

- Admin pages (simple password protected endpoint behind reverse proxy) to view issued codes, usage, totals per style and recent payments.
- Track simple events via server logs or lightweight analytics (Matomo or Plausible) to measure referral conversion from watermarks.

---

## Future features & extensions

- Offer account system so users can save their generated checks.
- Allow users to buy packs of styles or subscribe for unlimited access.
- Add SVG export, sticker packs, or animated check GIFs.
- Integrate with social networks for one‑tap sharing and custom captions.

---

## What Codex / a developer needs next

- Confirm which payment provider to implement first (Stripe recommended).
- Decide delivery method: Telegram vs email (for instant delivery Telegram is best if the audience already uses it).
- Decide whether to use DB (persistent codes) or stateless tokens.
- If DB chosen: include `server/migrations/` SQL and a tiny admin UI.

If you want, I can also generate a minimal `server/index.js` (Express) implementing:

- `/create-checkout-session` example for Stripe,
- `/webhook` verifying Stripe signature and issuing code,
- `/redeem` endpoint,
- `scripts/seed_codes.js` helper.

Provide which payment provider and delivery channel you prefer and I will produce the server code and test cases.

