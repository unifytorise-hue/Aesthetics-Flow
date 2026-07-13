# AestheticFlow CRM

Single-file app (`index.html`) backed by Supabase (project `zatgwmdinqzrwxxdmxoe`). Multi-tenant SaaS: each clinic is a row in `clinics`, isolated via RLS (`clinic_id` + `is_staff()`/`my_clinic_id()`). Platform admin dashboard exists (`platform_admins` table, `is_platform_admin()`).

Live site deploys from GitHub Pages off `claude/aestheticflow-crm-design-6y67rr` (mirrors `claude/aestheticflow-phase-2-auth-ltya60`, which both get pushed together).

## Outstanding / remember for later

- **Billing is not live.** `clinics.plan`/`subscription_status` exist and are manually editable from the Platform Admin dashboard, but there's no real Stripe checkout or webhook wired up. **When the user is ready to launch/charge real customers, revisit this and set up actual Stripe billing** (needs a Stripe account + API keys + price IDs from the user — not something Claude can self-serve).
- Supabase org is on the **Free plan** — "Leaked Password Protection" (Attack Protection → Prevent use of leaked passwords) is Pro-tier-gated and can't be enabled until upgraded. Revisit before onboarding real customers at scale.
- Database is production-clean as of 2026-07-13: the old `clinic1` dev/seed clinic (mislabeled `plan: active`) and an orphaned test clinic (`clinic_5a8c77c6178d`, from testing the signup flow) were deleted — both had zero real data. The only clinic left is the real trial signup `Maeso`. Note: Dr Sarah / Dr James / Amy / Dr Naledi never existed as DB rows — they're hardcoded fallback demo data in `index.html`'s initial JS state, used only when there's no Supabase session (offline/demo mode), not something to clean up in the database. `medicalvirtuals88@gmail.com` is the platform admin account (`platform_admins` table), not clinic staff.
- Client portal now exposes `photo_sets` (before/after photos) as of 2026-07-13, via a new "clients read own photo sets" RLS policy. `documents` (consent forms etc.) are still staff-only — revisit if clients should see those too.
