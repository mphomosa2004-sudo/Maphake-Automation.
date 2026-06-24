# Maphake Automation — Security Report & Setup Guide

This document covers the security review of the Maphake Automation website, the fixes applied, and the exact steps to deploy your Cloudflare Worker and connect the AI assistant safely.

---

## 1. What kind of site this is

This is a **static multi-page website** hosted on **GitHub Pages**. It has:
- No database
- No user accounts or logins
- No file uploads
- No server-side code on GitHub (GitHub Pages only serves static files)

The only "live" backend piece is the **AI assistant**, which talks to a **Cloudflare Worker** that you deploy separately. That Worker is the only place your secret API key lives.

This means many items from a full backend security checklist **do not apply yet** — they only become relevant if you later add a login system, database, or uploads. Those are marked **N/A (revisit later)** below.

---

## 2. Security issues — ranked

### CRITICAL
| # | Issue | Status |
|---|-------|--------|
| C1 | API key must never be in frontend code | **FIXED** — key lives only in the Cloudflare Worker as an encrypted secret. No key anywhere in HTML, `script.js`, or GitHub. |

### HIGH
| # | Issue | Status |
|---|-------|--------|
| H1 | AI endpoint could be abused to run up API costs | **FIXED** — Worker enforces per-IP rate limiting, response token cap, and message caps. Client adds a cooldown. |
| H2 | User input (contact form + AI chat) could carry XSS / HTML injection | **FIXED** — all input is sanitized; all output is rendered with `textContent` (never `innerHTML`). |
| H3 | CORS could let any site call your Worker | **FIXED** — Worker only accepts requests from your domains. |
| H4 | Missing browser security headers | **ACTION NEEDED** — set via Cloudflare (Section 5). GitHub Pages cannot set these itself. |

### MEDIUM
| # | Issue | Status |
|---|-------|--------|
| M1 | Prompt injection (user telling the AI to ignore its rules) | **REDUCED** — system prompt is fixed server-side in the Worker and instructs the AI to stay on-topic. Inputs are length-capped. |
| M2 | Oversized requests | **FIXED** — Worker rejects bodies over ~12KB. |
| M3 | Error messages could leak internals | **FIXED** — all errors return generic messages. |
| M4 | GitHub leaks an `x-github-request-id` header | **ACTION NEEDED** — optional removal via Cloudflare (Section 5). |

### LOW
| # | Issue | Status |
|---|-------|--------|
| L1 | No HSTS yet | **ACTION NEEDED** — enable in Cloudflare (Section 5). |
| L2 | Referrer / permissions policy not set | **ACTION NEEDED** — set in Cloudflare (Section 5). |

### N/A (revisit later — only if you add these features)
- Authentication & user accounts
- Authorization / role-based access / admin dashboards
- Database security (prepared statements, row-level rules)
- File upload restrictions
- CSRF tokens (no cookie-based auth on the site)
- Server-side logging infrastructure
- Dependency/package audits (no npm packages in the static site)

---

## 3. Files in this project

```
maphakeautomation/
├── index.html              Home
├── services.html           Services
├── packages.html           Packages & pricing
├── ai-automation.html      AI automation
├── portfolio.html          Portfolio
├── about.html              About
├── contact.html            Contact (form)
├── terms.html              Terms & Conditions
├── privacy.html            Privacy Policy
├── css/style.css           All styling
├── js/script.js            All behaviour (loader, animations, AI widget, form)
├── images/robot.png        Robot mascot (loader + hero)
├── images/logo.jpeg        M logo (nav + footer + favicon)
├── worker.js               Cloudflare Worker — DEPLOY SEPARATELY, do NOT put on GitHub Pages
├── .gitignore              Keeps secrets out of git
└── SECURITY.md             This file
```

**Upload everything EXCEPT `worker.js` to GitHub** (the Worker is deployed to Cloudflare instead — see Section 4). It's fine if `worker.js` ends up in the repo too, since it contains **no secret** — but it does nothing on GitHub Pages.

---

## 4. Deploying the Cloudflare Worker (connect the AI)

The AI assistant will not work until you complete these steps. Until then, the widget shows a friendly "not connected yet" message and points visitors to WhatsApp.

### Step 1 — Create the Worker
1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com).
2. Go to **Workers & Pages** → **Create** → **Create Worker**.
3. Give it a name, e.g. `maphake-ai`. Click **Deploy** (it deploys a default hello-world first).
4. Click **Edit code**.
5. Delete the default code, then paste the entire contents of **`worker.js`**.
6. Click **Deploy**.

### Step 2 — Add your API key as a Secret (this is the safe part)
1. In your Worker, go to **Settings** → **Variables and Secrets**.
2. Under **Secrets**, click **Add**.
3. Name it exactly: `ANTHROPIC_API_KEY`
4. Value: paste your Anthropic API key (starts with `sk-ant-...`).
5. Click **Save / Encrypt**.

> The key is now encrypted inside Cloudflare. It is never sent to the browser and never appears in your website code or GitHub.

### Step 3 — Set your allowed origins
1. Open `worker.js` (in the Cloudflare editor).
2. Find the `ALLOWED_ORIGINS` list near the top.
3. Make sure it contains your live domains:
   ```js
   const ALLOWED_ORIGINS = [
     "https://www.maphakeautomation.co.za",
     "https://maphakeautomation.co.za",
   ];
   ```
4. If you're testing on the GitHub Pages URL first (e.g. `https://yourusername.github.io`), add that line too.
5. **Deploy** again after any change.

### Step 4 — Copy your Worker URL
After deploying, Cloudflare shows your Worker URL, e.g.:
```
https://maphake-ai.YOURNAME.workers.dev
```
Copy it.

### Step 5 — Paste the Worker URL into the website
1. Open **`js/script.js`**.
2. Near the very top, find this line:
   ```js
   const WORKER_URL = "PASTE_YOUR_CLOUDFLARE_WORKER_URL_HERE";
   ```
3. Replace the placeholder with your real Worker URL:
   ```js
   const WORKER_URL = "https://maphake-ai.YOURNAME.workers.dev";
   ```
4. Save, commit, and push to GitHub. The AI assistant is now live.

> **Reminder:** Only the **Worker URL** goes in `script.js`. The **API key never does.** The URL is safe to be public; the key is not.

---

## 5. Cloudflare security headers (15 minutes, one-time)

GitHub Pages can't set security headers, so we add them through Cloudflare (your domain must be on Cloudflare's DNS, which it already is for the Worker setup).

### 5a. Enable HSTS
**SSL/TLS → Edge Certificates → HTTP Strict Transport Security (HSTS) → Enable**
- Max Age: **12 months**
- Include subdomains: optional
- Save.

### 5b. Add response headers
**Rules → Transform Rules → Modify Response Header → Create rule** → apply to all incoming requests → **Set static** for each:

| Header | Value |
|--------|-------|
| `X-Frame-Options` | `SAMEORIGIN` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |
| `Content-Security-Policy` | `default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; script-src 'self'; connect-src 'self' https://maphake-ai.YOURNAME.workers.dev; frame-ancestors 'self'` |

> In the CSP above, replace `https://maphake-ai.YOURNAME.workers.dev` with your real Worker URL so the AI chat is allowed to connect. If the AI ever stops responding after adding CSP, check this line first.

### 5c. (Optional) Remove the GitHub header
Same Transform Rules area → **Remove** action → header name `x-github-request-id`.

---

## 6. Recommended rate-limit values

These are set in `worker.js` and tuned for a small business site. Adjust if needed:

| Setting | Value | Meaning |
|---------|-------|---------|
| `RATE_LIMIT_MAX` | 15 | Max requests per IP per window |
| `RATE_LIMIT_WINDOW_SEC` | 60 | The window length (seconds) |
| `MAX_BODY_BYTES` | 12000 | Reject requests larger than ~12KB |
| `MAX_MESSAGES` | 25 | Max messages kept per conversation |
| `MAX_MESSAGE_CHARS` | 1000 | Max characters per single message |
| `MAX_TOKENS` | 500 | Caps AI reply length (controls cost) |

Client-side (in `script.js`): a **2.5-second cooldown** between messages and a **25-message** session cap.

> **Stronger option:** the Worker's rate limiter uses in-memory storage, which resets per isolate. For tighter, global rate limiting, use **Cloudflare KV** or **Durable Objects**, or add a **Cloudflare Rate Limiting Rule** on the Worker route. For a small business site, the current setup is reasonable.

---

## 7. Remaining risks (be aware)

1. **In-memory rate limiting** is best-effort, not global. Add Cloudflare KV/Rate Limiting Rules if you ever get abuse.
2. **Prompt injection** can't be 100% eliminated. The AI is restricted to business topics, but treat anything a user types as untrusted — never wire the AI to perform real actions (payments, data changes) without a human in the loop.
3. **Third-party providers** (GitHub, Cloudflare, Anthropic, domain registrar) have their own uptime and security. Outages there can affect your site.
4. **No monitoring yet.** Consider checking your Cloudflare Worker analytics and Anthropic usage dashboard occasionally to spot unusual spikes.

---

## 8. Pre-deployment checklist

Before you go live, confirm:

- [ ] `worker.js` deployed to Cloudflare and **Deploy** clicked
- [ ] `ANTHROPIC_API_KEY` added as an **encrypted Secret** (not a plain variable)
- [ ] `ALLOWED_ORIGINS` in the Worker matches your real domain(s)
- [ ] Worker URL pasted into `WORKER_URL` in `js/script.js`
- [ ] **No API key** anywhere in the GitHub repo (search the repo for `sk-ant` to be sure)
- [ ] HSTS enabled in Cloudflare
- [ ] Security headers added via Transform Rules
- [ ] CSP `connect-src` includes your Worker URL
- [ ] Contact form opens WhatsApp correctly with details filled in
- [ ] AI assistant replies correctly on desktop and mobile
- [ ] Site tested on a phone (nav menu, AI widget, forms all work)
- [ ] `.gitignore` is in the repo so `.env`/secrets can't be committed later

---

## 9. If a key is ever exposed

If your API key ever appears in a commit, screenshot, or public place:
1. Go to the Anthropic Console → **API Keys**.
2. **Revoke / delete** the exposed key immediately.
3. Create a **new** key.
4. Update the `ANTHROPIC_API_KEY` secret in your Cloudflare Worker with the new key.
5. Re-deploy the Worker.

Never paste the key into HTML, `script.js`, a commit, or a message. The Worker secret is the only correct home for it.

---

*Prepared for Maphake Automation. Questions: info@maphakeautomation.co.za*
