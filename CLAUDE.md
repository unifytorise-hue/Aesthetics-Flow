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
  2026-07-13; tiered pricing added 2026-07-18) — real Paystack Checkout for
  content-credit purchases and the monthly subscription, now split across 3
  pricing tiers (see "Pricing tiers" below). Needs `PAYSTACK_SECRET_KEY` and
  one Plan code per tier — `PAYSTACK_PLAN_CODE_SOLO` /
  `PAYSTACK_PLAN_CODE_TEAM` / `PAYSTACK_PLAN_CODE_MULTI` (each a Plan created
  in the Paystack Dashboard → Plans — real pricing decisions, deliberately
  not defaulted; a tier with no plan code set just returns
  `{configured:false}` for that tier specifically, others still work).
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

## Pricing tiers (added 2026-07-18)

Three tiers — **Solo** ($25/mo), **Team** ($60/mo), **Multi-Location**
($150/mo) — deliberately priced at the low end of what was originally
discussed, as launch pricing meant to be raised later for new signups once
there's real usage/case-study data; existing subscribers keep whatever
price they signed up at (Paystack subscriptions are tied to the specific
Plan they joined, not a live-updating price). All config lives in
`index.html` as `TIERS`/`TIER_LIMITS`/`TIER_PRICES`/`TIER_NAMES`, right after
`ROLE_PERMISSIONS`.

Tiers gate only the two multi-tenant dimensions already in the schema
(staff seats, locations) plus the two features with a real per-use cost —
SMS campaigns (Twilio charges per message) and public API access. Everything
else (clients, appointments, invoices, inventory, AI content credits) stays
unlimited/unaffected by tier for now — AI credits in particular are **not**
tier-scaled yet (still the existing flat `content_credits` balance +
top-up-purchase system); a recurring monthly credit allowance per tier would
need a scheduled job (pg_cron) and was deliberately left as a follow-up
rather than bolted on here.

- `clinics.tier` (migration `add_tier_to_clinics`) — `'solo'|'team'|'multi'`,
  default `'solo'`, check-constrained. Distinct from the pre-existing
  `clinics.plan` column, which despite the name has always meant billing
  *status* (`'trial'|'active'|...`, redundant with `subscription_status` —
  a naming quirk that predates this feature and wasn't touched here).
- `currentTierLimits()` — the single gating source of truth. Returns
  unlimited-everything while `subscription_status === 'trialing'` (trial
  users get full access to evaluate the product, same principle as the
  billing banner already treating trial as a distinct, non-restricted
  state), otherwise returns `TIER_LIMITS[clinic.tier]`.
- **Enforcement points**: Add Staff button (Settings) checks seat count
  before opening the modal; Add Location button checks location count the
  same way; the Campaign modal's Type `<select>` disables the SMS `<option>`
  (with an inline "(upgrade to Team plan)" hint) when `!limits.sms`; the
  Settings API Access card renders a locked placeholder card instead of the
  functional key-management UI when `!limits.api`. All of these are
  client-side UX only — there's no server-side/RLS enforcement of seat or
  location counts, so a determined user could still insert rows directly;
  acceptable for now since this mirrors how every other client-side
  permission check in the app already works (`currentPerms()`), not a new
  risk class.
- **Settings → Billing** now renders a 3-card tier comparison (price,
  seats/locations/SMS/API summary, and an Upgrade/Current-Plan button per
  tier) instead of the old single "Upgrade Plan" button — also fixed the
  Plan/Status redundancy that existed before (the "Plan" row used to just
  echo the same trial/active value as "Status"; it now shows the tier name).
- **Checkout flow**: each tier's Upgrade button calls
  `create-paystack-transaction` with `{kind:'subscription', tier:'solo'|
  'team'|'multi'}`; the function picks the matching `PAYSTACK_PLAN_CODE_*`
  secret and puts `tier` in the Paystack transaction metadata.
  `paystack-webhook`'s `charge.success` handler (the only place metadata is
  available — `subscription.create`'s payload doesn't carry it) reads that
  back and sets `clinics.tier` alongside the existing status/plan updates.
  **Known gap**: upgrading tiers while already on an active paid
  subscription re-runs the same checkout flow (a fresh Paystack transaction
  tied to the new Plan) rather than modifying the existing subscription in
  place — Paystack will start billing the new Plan, but the old subscription
  isn't automatically canceled, which could double-bill a real customer.
  Fine for launch (nobody's on a paid tier yet), but needs a real
  cancel-old-subscription step added before tier upgrades are used by an
  actual paying customer.
- **Platform Admin** clinics table gained a Tier column/dropdown next to the
  existing Plan/Status ones, backed by `admin_list_clinics()` (now selects
  `tier`) and `admin_set_clinic_plan(p_clinic_id, p_plan, p_status, p_tier)`
  (new optional 4th param, `coalesce`s against the existing value so old
  callers passing only 3 args still work) — both `SECURITY DEFINER` Postgres
  functions gated on `is_platform_admin()`.
- **Not yet translated**: the new Billing tier-comparison cards and the SMS
  upgrade-hint copy are English-only, consistent with Settings content
  generally still being outside the i18n pass (see the i18n entries below).
- Verified via a mocked-network browser E2E test (17/17 checks) covering
  seat/location gating at and under the limit on Solo/Team/Multi, SMS
  option disabling, the API Access card's locked/unlocked states, trial
  bypassing all limits, the Upgrade button's Paystack call payload, and the
  Platform Admin tier dropdown reading and writing correctly.

## Affiliate referral system (added 2026-07-26)

Lets a clinic generate shareable links for social-media/referral partners;
anyone who submits their details through one becomes a tagged Lead, and the
clinic sees referral counts + commission owed once that lead becomes a
paying client. Two explicit product decisions behind this (confirmed before
building): the link goes to a **public lead-capture form**, not self-service
booking (there's no public real-time calendar-availability system to book
against); and commission is **earned when the referral becomes a paying
client** (first paid invoice), not merely on booking. A third decision
(affiliates do **not** get their own login/portal — staff track everything
internally) keeps this to one new public surface rather than a third
authentication system alongside staff and client-portal logins.

- **`affiliates` table** (migration `add_affiliates_referral_system`) —
  `id`, `clinic_id`, `name`, `code` (globally unique, used in the shareable
  link), `commission_rate`, `contact_email`, `active`. RLS mirrors every
  other clinic-scoped table (`is_staff() and clinic_id = my_clinic_id()`).
  `leads.referred_by_affiliate_id` and `clients.referred_by_affiliate_id`
  (both nullable FKs, `on delete set null`) tag who referred them; the
  lead→client conversion flow (`bindLeadsEvents`, Sales Pipeline "Completed"
  stage) copies the reference across so commission calculation only needs
  to look at `clients`, not join back through `leads`.
- **`affiliate-referral` edge function** (new, `verify_jwt: false`) — the
  one deliberately public, unauthenticated surface in the whole app.
  `GET ?code=X` resolves a code to the affiliate + clinic name (for the
  landing page's "Referred by X to Y" header); `POST` accepts
  name/phone/email/note and inserts a Lead scoped to that clinic, service-role
  bypassing RLS the same way `public-api`/`paystack-webhook` already do.
  Deliberately narrow blast radius: it can only ever create a Lead, never a
  client or anything billing-related. No CAPTCHA/rate-limiting yet — fine
  for launch, worth adding if a real link gets spammed.
- **Public landing page** (`renderAffiliateReferralPage`, `index.html?ref=
  <code>`) — checked in `init()` before any session/auth routing, so a
  stranger with no account lands here regardless of session state. Reuses
  the `.portal-center-wrap`/`.portal-login-card` auth-screen styling for
  visual consistency with the rest of the app.
- **Sales Team page** gained an "Affiliate Partners" card (owner/admin only)
  — list of affiliates with referred-lead/converted-client counts and
  commission owed (`computeAffiliateMetrics()`, calculated live from paid
  invoices exactly like the existing staff origination-commission system —
  no ledger table, no auto-payout, matching how staff commission already
  works), a Copy Link button, and Add/Edit/Deactivate/Delete. Codes are
  client-generated (`crypto.getRandomValues`, 8 base36 chars) — collision
  risk is negligible at this scale and matches the non-cryptographic
  `Date.now()`-based IDs used everywhere else in the app.
- **Not yet translated**: the whole feature (landing page + Sales Team card)
  is English-only, consistent with Sales Team and Settings generally still
  being outside the i18n pass.
- Verified via a mocked-network browser E2E test (11/11 checks, plus a
  standalone check confirming lead→client conversion propagates the
  affiliate reference) covering affiliate CRUD, commission math (confirmed
  it only counts *paid* invoices, not sent/draft ones), the Copy Link
  button's URL, the public landing page's happy path end-to-end (code
  resolution → form submit → success message), and the graceful
  inactive/invalid-code error state.

## Outstanding / remember for later

- Supabase org is on the **Free plan** — "Leaked Password Protection" (Attack Protection → Prevent use of leaked passwords) is Pro-tier-gated and can't be enabled until upgraded. Revisit before onboarding real customers at scale.
- Database is production-clean as of 2026-07-13: the old `clinic1` dev/seed clinic (mislabeled `plan: active`) and an orphaned test clinic (`clinic_5a8c77c6178d`, from testing the signup flow) were deleted — both had zero real data. The only clinic left is the real trial signup `Maeso`. Note: Dr Sarah / Dr James / Amy / Dr Naledi never existed as DB rows — they're hardcoded fallback demo data in `index.html`'s initial JS state, used only when there's no Supabase session (offline/demo mode), not something to clean up in the database. `medicalvirtuals88@gmail.com` is the platform admin account (`platform_admins` table), not clinic staff.
- Client portal now exposes `photo_sets` (before/after photos) as of 2026-07-13, via a new "clients read own photo sets" RLS policy. `documents` (consent forms etc.) are still staff-only — revisit if clients should see those too.
- Orphaned auth user `unifytorise@gmail.com` (from signup-flow testing) still exists in `auth.users` with no linked clinic/staff/admin row. Harmless as-is; flagged in case it should be deleted.
- A stale code comment claiming "RLS is not yet configured" (predating Phase 2) was corrected on 2026-07-13 — verified via `pg_policies` that every table has proper clinic/client-scoped RLS.
- Password reset (client portal added 2026-07-13, extended to staff sign-in 2026-07-26): "Forgot password?" on the client sign-in screen — and now also the staff sign-in screen — calls `sb.auth.resetPasswordForEmail()`; a global `onAuthStateChange` listener catches the `PASSWORD_RECOVERY` event when the user returns via the emailed link and shows a "Set New Password" screen (`renderSetNewPassword()`). That screen was already generic (routes back through `handleSession()`, which resolves staff vs client vs pending on its own), so no changes were needed there — only `renderStaffAuth()` needed a `forgot`/`reset-sent` mode mirroring `renderPortalAuth()`'s existing implementation, reusing the same `portal.*` translation keys (already staff-neutral in meaning) rather than duplicating new ones. Same non-enumeration behavior as the portal flow: a reset request for an unknown email shows the identical "if an account exists…" confirmation as a real one, never revealing which. **Requires a manual Dashboard step**: the redirect URL (`window.location.origin + window.location.pathname` — i.e. the live GitHub Pages URL) needs to be on Supabase's Auth → URL Configuration → Redirect URLs allow-list, or the reset link won't come back to the app correctly. Not verified against a real email send in this session (this sandbox can't reach Supabase's network at all — see below); worth a real click-through once deployed. Verified via a mocked-network browser E2E test (12/12 checks) covering the link's visibility (sign-in only, not signup), the full forgot → email-sent → back-to-sign-in loop, the no-enumeration behavior on a failed reset, and a regression check that the client portal's own flow still works.
- This session confirmed a hard sandbox limitation: browser traffic to the real Supabase project (`zatgwmdinqzrwxxdmxoe.supabase.co`) is blocked by this environment's egress policy, same as the CDN. Genuine live E2E testing isn't possible from here — verification this session used either direct SQL/schema inspection (for backend correctness) or a mocked-network browser test that drives the real app code with a stand-in `supabase.js` (for UI/client-logic correctness). A future session with real network access should do one real click-through end to end.
- **Language selector / i18n (added 2026-07-15, extended 2026-07-16)**: `LANGUAGES`/`TRANSLATIONS`/`t()`/`getLang()`/`setLang()` in `index.html`. Now 5 languages — English, Spanish, French, Portuguese, Arabic. Originally bounded to nav labels + auth screens + client portal dashboard; now being extended page-by-page through the rest of the staff CRM (see the 2026-07-18 entry below for progress). Language is a `localStorage` preference (per-browser, not per-clinic) since the auth screens render before any clinic is loaded. To extend: add a language to `LANGUAGES`+`TRANSLATIONS`, or add more translated strings by adding a key to every language block and calling `t('your.key')` at the render site. Switching language from Settings (`#setLanguage`) calls `render()`, so any already-translated view picks up the change immediately without a page reload.
- **Full CRM translation, in progress (started 2026-07-18)**: extending i18n coverage beyond the original bounded scope, one page at a time — each page gets its own `<page>.*` translation-key namespace (e.g. `dash.*`, `clients.*`, `cal.*`, `leads.*`, `inv.*`) across all 5 languages, plus canonical-value lookup namespaces for data enums that must stay in English internally but display translated (e.g. `skinType.*`, `concern.*`, `leadStage.*`, `invStatus.*`, reusing the existing `invoiceType.*` pattern) — `<select>`/`<option>` elements get an explicit `value="..."` attribute pinned to the English canonical string so save handlers, filters, and stage-comparison logic reading `.value`/the underlying data field are unaffected by the display language. Added shared `common.edit`/`common.delete` keys for the many pages that just need a plain "Edit"/"Delete" icon-button title. **Done so far**: Dashboard (`renderDashboard`), Clients list + client detail (`renderClientsList`/`renderClientsTable`/`renderClientDetail`, all 6 tabs), Calendar (`renderCalendar`), Sales Pipeline/Leads (`renderLeads`), Inventory (`renderInventory`), Invoices list + builder + detail modal (`renderInvoices`/`renderInvoiceBuilderModal`/`openInvoiceDetailModal`), Sales Team (`renderSales` — target board, pending approvals, revenue-share pie chart, agent directory). Each page verified via a mocked-network browser E2E test before commit. Translating Sales caught a real bug in the translation pass itself: the top-of-page "Origination Commission Owed" stat and the shorter per-agent-card "Commission Owed" label are two distinct English strings that got collapsed into one `sales.commissionOwed` key on the first draft — split back out into `sales.commissionOwed` (long) and `sales.commissionOwedShort` once the E2E test's per-agent-card assertion caught the wrong text. **Deliberately left English-only within Invoices**: the printable/PDF invoice document (`openPrintableInvoice`, a separate popup HTML document intended as a business record — consistent with the RTL printable-invoice exception noted above) and the invoice send toast messages ("This client has no email on file", "Email drafted — check your mail app", etc.) — `showToast()` call sites are scattered across nearly every action in the app, not specific to one page, so toast-message i18n is being treated as a separate future pass rather than folded into each page's translation. **Also done**: Team Chat (`renderTeamChat`), AI Assistant (`renderAIAssistant`/`renderChatBubble`), and the billing/trial banner shown at the top of every staff page (`renderBillingBanner`) — added shared `common.send` for the two chat "Send" buttons and `billing.*`/`teamchat.*`/`ai.*` namespaces. **Also done**: Campaigns main list view (`renderCampaigns`/`renderCampaignsList` — subtab switcher, credit badge, campaign cards, status pills via `campaignStatus.*`). **Deliberately left English-only within Campaigns**: Social Content sub-tab (`renderSocialContent`), the campaign create/edit modal (`renderCampaignModal`), and the social post modal (`renderSocialPostModal`) — these are large, deeply nested modals (AI content generation forms, platform pickers, etc.) being treated as a separate follow-up rather than folded into this pass. **Still English-only pages**: Reports, Settings content, Platform Admin.
- **RTL support (added 2026-07-16)**: `applyDirection()` sets `<html dir="rtl" lang="ar">` when Arabic is selected (`RTL_LANGS`), called on load and after every language switch. Most layout mirrors automatically since the app already relies on flexbox `row` direction (which is direction-aware per the CSS spec) — verified via screenshot that the sidebar, nav, and auth cards all flip correctly. Added explicit `[dir="rtl"]` overrides for the handful of physical (non-auto-flipping) CSS properties in the structural shell: sidebar border, nav active-item indicator, table header alignment, mobile drawer slide direction. **Not exhaustively audited**: ~15 scattered inline-style `margin-left`/spacing tweaks deeper in the app (status badges, the printable invoice template, the platform admin table) were not individually mirrored — functional but not pixel-perfect in RTL. Only reachable via Arabic today; if Hebrew or another RTL language is added later, add its code to `RTL_LANGS`.
- **Currency selector (added 2026-07-15)**: the free-text currency field in clinic Settings is now a dropdown of ~140 world currencies (`CURRENCIES`). Still stores just the symbol in `clinic_settings.currency` (unchanged column/type) — this is a UI improvement, not a switch to ISO-code-based storage. Because several currencies share a symbol (e.g. `$` for USD/MXN/ARS/...), the dropdown's pre-selected option after reload may not be the exact currency originally picked, though the displayed symbol is always correct.
- **Country selector (added 2026-07-15)**: added to both clinic Settings and the client Add/Edit modal, via `countrySelectHtml()`/`bindCountrySelect()`/`countrySelectValue()` — a dropdown of ~197 countries plus a free-text "Other" fallback for anything not listed. New `country` column on both `clinic_settings` and `clients` (migration `add_country_to_clinic_settings_and_clients`). Not surfaced anywhere in the client detail view yet (only editable via the Add/Edit Client modal) — revisit if it should be shown there too.
- Building the i18n/currency/country feature caught a real bug in its own first draft: every screen's language switcher reused the same `id="langSwitch"`, and since auth screens are hidden (not removed) when navigating away, stale duplicate IDs collided and `getElementById` could bind to the wrong element. Fixed by giving each screen's switcher a unique ID (`langSwitchStaffAuth`, `langSwitchPortalAuth`, etc.) — worth remembering as a general pattern risk if more per-screen widgets get added with reused IDs.
- **Locale-aware date/number formatting (added 2026-07-15)**: `getLocale()` maps the selected language to a BCP 47 locale (`LOCALE_BY_LANG`: en→en-GB, es→es-ES, fr→fr-FR), and every `toLocaleDateString`/`toLocaleString` call in the app (not just the bounded-translation surface — `fmtDate()`, `fmtMoney()`, the calendar week/month labels, dashboard "at risk" grouping, reports month labels, API key timestamps) now uses it instead of a hardcoded `'en-GB'`. So switching language also switches date format ("13 Jul 2026" → "13 juil. 2026") and number grouping/decimal convention ("$12,345.67" → "$12 345,67") app-wide, even on pages whose text itself is still English-only. Verified via a mocked browser test across all three languages. Note: `es-ES` legitimately omits thousands grouping on 4-digit numbers (e.g. `2500` stays `2500`, not `2.500`) per ICU's `minimumGroupingDigits` rule for that locale — grouping only kicks in at 5+ digits. Confirmed as correct `Intl` behavior, not a bug, when a test initially expected a separator there.
- **Phone number formatting + localized client-facing message templates (added 2026-07-17)**: `COUNTRY_CALLING_CODES` (197 entries, keyed identically to `COUNTRIES`) backs two new helpers — `normalizePhoneForWhatsApp(phone, countryName)` (handles `+`-prefixed, `00`-prefixed, and bare local numbers, converting to the digit string `wa.me` links need) and `phonePlaceholderForCountry(countryName)` (e.g. `+27 phone number`, shown as a live-updating placeholder hint on the Add/Edit Client modal's phone field, tied to the country dropdown). Fixes a real bug: both WhatsApp send call sites (invoice send, review request send) previously just stripped non-digits from `client.phone`, which silently produced a broken `wa.me` link for any client whose phone was saved in local format (e.g. `082 123 4567`) instead of full international format — they now go through `normalizePhoneForWhatsApp(client.phone, client.country)`. Also added `tf(key, vars)` (a `t()` variant with `{placeholder}` substitution) and `msg.*`/`invoiceType.*` translation keys across all 5 languages, and refactored `buildInvoiceMessage()`/`invoiceHeaderText()` and `openReviewRequestModal()`'s message + email subject to use them — so the invoice send message and review-request message (previously hardcoded English regardless of language setting) now localize along with the rest of the auth/portal surface. Verified via Node unit tests (phone normalization, 7/7 cases) and a mocked browser E2E test (language switch + message content across en/es/ar).
