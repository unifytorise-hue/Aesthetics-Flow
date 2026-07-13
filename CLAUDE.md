# AestheticFlow CRM

Single-file app (`index.html`) backed by Supabase (project `zatgwmdinqzrwxxdmxoe`). Multi-tenant SaaS: each clinic is a row in `clinics`, isolated via RLS (`clinic_id` + `is_staff()`/`my_clinic_id()`). Platform admin dashboard exists (`platform_admins` table, `is_platform_admin()`).

Live site deploys from GitHub Pages off `claude/aestheticflow-crm-design-6y67rr` (mirrors `claude/aestheticflow-phase-2-auth-ltya60`, which both get pushed together).

## Edge functions (all deployed and live, all gated on missing secrets)

Every integration below is real, production code — not scaffolding — but each
one deliberately no-ops (`{ configured: false }`, frontend shows a clear
message) until its secret is set. Nothing can charge a card, send a real
email/SMS, or call a paid API until the user supplies the corresponding key.
Set secrets via the Supabase Dashboard (Edge Functions → Secrets) — there's
no MCP tool for it, so this always needs the user to do it themselves.

- **`create-paystack-transaction`** + **`paystack-webhook`** (replaced Stripe
  2026-07-13) — real Paystack Checkout for content-credit purchases and the
  monthly subscription. Needs `PAYSTACK_SECRET_KEY` and
  `PAYSTACK_SUBSCRIPTION_PLAN_CODE` (a Plan created in the Paystack
  Dashboard → Plans — a real pricing decision, deliberately not defaulted).
  `PAYSTACK_CURRENCY` is optional — omit it to use the Paystack account's
  default currency. Webhook: Paystack Dashboard → Settings → API Keys &
  Webhooks → Webhook URL, pointed at the `paystack-webhook` function URL —
  unlike Stripe, Paystack signs webhooks with the same secret key (no
  separate webhook secret to generate). **This is the main launch
  blocker** — needs a Paystack account + keys from the user. The
  `subscription.create`/`subscription.disable`/`invoice.payment_failed`
  handling in `paystack-webhook` is best-effort from Paystack's docs, not
  verified against a live payload yet — check `get_logs` for that function
  against a real event once the webhook is live and adjust field paths if
  Paystack's actual payload differs.
- The old `create-checkout-session` and `stripe-webhook` Stripe functions
  are retired — redeployed as inert 410-response stubs (no MCP tool exists
  to delete a deployed edge function outright). Safe to delete manually via
  the Supabase Dashboard. `clinics.stripe_customer_id`/`stripe_subscription_id`
  were renamed to `paystack_customer_code`/`paystack_subscription_code`.
  If Stripe's own dashboard still has a webhook endpoint configured, remove
  it there too so Stripe stops retrying against the stub.
- **`send-campaign`** — real bulk email (SendGrid) / SMS (Twilio) sending for
  Campaigns, replacing the mailto:/sms: hand-off. Needs `SENDGRID_API_KEY` +
  `SENDGRID_FROM_EMAIL` for email, `TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN`
  + (`TWILIO_FROM_NUMBER` or `TWILIO_MESSAGING_SERVICE_SID`) for SMS.
- **`ai-proxy`** (added 2026-07-13) — proxies all AI Assistant features
  (content ideas, social captions, campaign copy, the chat assistant) to
  Anthropic. **Fixes a real production bug**: these features previously
  called `api.anthropic.com` directly from the browser with no key attached
  — that only works in a sandboxed preview host, so on the actual GitHub
  Pages production site every AI feature silently failed 100% of the time.
  Needs `ANTHROPIC_API_KEY` as an edge function secret to go live.
- **`public-api`** — external API (`GET /clients`, `POST /appointments`)
  authenticated with per-clinic hashed keys from Settings → API Access.
  Already fully live, no secret needed.

## Outstanding / remember for later

- Supabase org is on the **Free plan** — "Leaked Password Protection" (Attack Protection → Prevent use of leaked passwords) is Pro-tier-gated and can't be enabled until upgraded. Revisit before onboarding real customers at scale.
- Database is production-clean as of 2026-07-13: the old `clinic1` dev/seed clinic (mislabeled `plan: active`) and an orphaned test clinic (`clinic_5a8c77c6178d`, from testing the signup flow) were deleted — both had zero real data. The only clinic left is the real trial signup `Maeso`. Note: Dr Sarah / Dr James / Amy / Dr Naledi never existed as DB rows — they're hardcoded fallback demo data in `index.html`'s initial JS state, used only when there's no Supabase session (offline/demo mode), not something to clean up in the database. `medicalvirtuals88@gmail.com` is the platform admin account (`platform_admins` table), not clinic staff.
- Client portal now exposes `photo_sets` (before/after photos) as of 2026-07-13, via a new "clients read own photo sets" RLS policy. `documents` (consent forms etc.) are still staff-only — revisit if clients should see those too.
- Orphaned auth user `unifytorise@gmail.com` (from signup-flow testing) still exists in `auth.users` with no linked clinic/staff/admin row. Harmless as-is; flagged in case it should be deleted.
- A stale code comment claiming "RLS is not yet configured" (predating Phase 2) was corrected on 2026-07-13 — verified via `pg_policies` that every table has proper clinic/client-scoped RLS.
- Client portal password reset (added 2026-07-13): "Forgot password?" on the client sign-in screen calls `sb.auth.resetPasswordForEmail()`; a global `onAuthStateChange` listener catches the `PASSWORD_RECOVERY` event when the user returns via the emailed link and shows a "Set New Password" screen. **Requires a manual Dashboard step**: the redirect URL (`window.location.origin + window.location.pathname` — i.e. the live GitHub Pages URL) needs to be on Supabase's Auth → URL Configuration → Redirect URLs allow-list, or the reset link won't come back to the app correctly. Not verified against a real email send in this session (this sandbox can't reach Supabase's network at all — see below); worth a real click-through once deployed.
- This session confirmed a hard sandbox limitation: browser traffic to the real Supabase project (`zatgwmdinqzrwxxdmxoe.supabase.co`) is blocked by this environment's egress policy, same as the CDN. Genuine live E2E testing isn't possible from here — verification this session used either direct SQL/schema inspection (for backend correctness) or a mocked-network browser test that drives the real app code with a stand-in `supabase.js` (for UI/client-logic correctness). A future session with real network access should do one real click-through end to end.
