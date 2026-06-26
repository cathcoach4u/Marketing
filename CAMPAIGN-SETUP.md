# Campaign Console — setup and good habits

_Last updated: 2026-06-26_

The Marketing repo now has a private **Campaign Console** (`campaigns.html`) for running Facebook
campaigns: plan campaigns, draft and publish posts to the Page, and work the leads that come in. It reuses
the Coach4U CRM's Supabase project, so there is one home for the data, not two.

This file is the source of truth for how it is wired and what is left to switch on. Update it whenever any of
this changes.

## The shape of it

- **Front end:** `campaigns.html` in this repo, a single self-contained page (Supabase JS via CDN, no build
  step), styled to the locked Coach4U brand.
- **Backend:** the **CRM Supabase project** `uoixetfvboevjxlkfyqy`. Tables `marketing_campaigns`,
  `marketing_posts`, `marketing_leads` already exist there with RLS set to authenticated-only.
- **Facebook:** the existing edge functions `meta-facebook` (publish a Page post) and `meta-lead-webhook`
  (receive lead-form leads). Both live in the CRM repo `internal-coach4u-hub/supabase/functions/`.

## The four good habits

### 1. Connect to Supabase — DONE in code
`campaigns.html` creates the client with the same URL and **publishable** key the CRM uses:

```
SUPABASE_URL = https://uoixetfvboevjxlkfyqy.supabase.co
SUPABASE_KEY = sb_publishable_… (publishable / anon — safe to embed, reads and writes nothing without a login)
```

It reads and writes `marketing_campaigns` and `marketing_posts`, and reads `marketing_leads`. Nothing to do.

### 2. Password protection — DONE in code, needs a login
The page is gated by **Supabase Auth**. It shows a sign-in screen and only loads the console after a
successful `signInWithPassword`. Same model as the CRM: RLS is authenticated-only and **self-signup is
disabled**, so the embedded key exposes no data.

- **To use it:** sign in with an existing Coach4U Supabase login (the same email and password as the CRM).
- **To add a person:** create their user in Supabase Auth (Cath only):
  https://supabase.com/dashboard/project/uoixetfvboevjxlkfyqy/auth/users — do not enable public sign-ups.

### 3. Cloudflare — Cath, dashboard steps
Put the whole Marketing site behind Cloudflare Access so only the team can reach it, then close the public
GitHub Pages copy. This mirrors the CRM's `internal-coach4u-hub/SECURITY.md`, but simpler because the whole
marketing site is private (no public bypass paths needed).

1. **Cloudflare → Pages:** create a Pages project from this repo (`cathcoach4u/Marketing`), production
   auto-deploy ON, preview deploys = None.
2. **Cloudflare → Zero Trust → Access → Applications → Add → Self-hosted:** cover the whole host.
   - **Policy 1:** Action = Allow, Include = your Coach4U team (emails or email domain).
   - **Policy 2:** Action = Block, Include = Everyone (fail-safe).
3. **Test** in a private window: the site should demand login.
4. **Close the GitHub backdoor:** GitHub → repo → Settings → Pages → Source → **None**, so there is no
   ungated `cathcoach4u.github.io/Marketing/` copy. Update your bookmark to the Cloudflare URL.

Note: the Supabase login (habit 2) is the app-level gate and works on its own. Cloudflare is the network-level
gate on top. Both is the belt-and-braces the CRM uses.

### 4. Connect to Facebook API — Cath sets secrets, then deploy
The console's **Publish to Facebook** button calls `meta-facebook`; lead forms post to `meta-lead-webhook`.
Both are written and committed in the CRM repo but not live until the Meta side is set up.

1. **Meta (developers.facebook.com):** an app with the Coach4U Page connected; a long-lived **Page access
   token** with `pages_manage_posts` (publishing) and, for lead ads, `leadgen` + `pages_read_engagement` +
   `pages_show_list`; note the numeric **Page ID**.
2. **Supabase secrets** (Cath only — Edge Functions → Secrets):
   https://supabase.com/dashboard/project/uoixetfvboevjxlkfyqy/functions
   - `META_PAGE_ID`
   - `META_PAGE_ACCESS_TOKEN`
   - `META_WEBHOOK_VERIFY_TOKEN` (any string you choose; used by the lead webhook handshake)
3. **Deploy the functions** (Claude can do this via the Supabase MCP once the secrets exist, or Cath via CLI):
   - `meta-facebook` — deploy with **verify_jwt: true** (the console sends the signed-in user token).
   - `meta-lead-webhook` — deploy with **verify_jwt: false** (Meta calls it unauthenticated).
4. **Register the lead webhook in Meta:** Webhooks → Page → subscribe `leadgen` to
   `https://uoixetfvboevjxlkfyqy.supabase.co/functions/v1/meta-lead-webhook`, verify token =
   `META_WEBHOOK_VERIFY_TOKEN`.

Until step 1–2 are done, **Publish to Facebook** returns "Facebook Page not configured" and no leads arrive.
Everything else in the console (planning campaigns, drafting posts) works straight away.

## Paid ads vs this console
This console covers **organic Page posts + lead capture + campaign planning**. Placing **paid ads**
(campaign → ad set → ad, budgets, audiences) is still done in **Meta Ads Manager** against the campaign plan.
Automating paid ads would be a separate build on Meta's Marketing API and is not in scope here.

## Files
- `campaigns.html` — the console.
- `index.html` — links to it (nav + a hub card).
- CRM repo: `supabase/functions/meta-facebook/index.ts`, `supabase/functions/meta-lead-webhook/index.ts`,
  `supabase/migrations/20260621_marketing_hub.sql` (the tables).
