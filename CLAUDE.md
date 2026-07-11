# AestheticFlow CRM

Single-file app (`index.html`) backed by Supabase (project `zatgwmdinqzrwxxdmxoe`). Multi-tenant SaaS: each clinic is a row in `clinics`, isolated via RLS (`clinic_id` + `is_staff()`/`my_clinic_id()`). Platform admin dashboard exists (`platform_admins` table, `is_platform_admin()`).

Live site deploys from GitHub Pages off `claude/aestheticflow-crm-design-6y67rr` (mirrors `claude/aestheticflow-phase-2-auth-ltya60`, which both get pushed together).

## Outstanding / remember for later

- **Billing is not live.** `clinics.plan`/`subscription_status` exist and are manually editable from the Platform Admin dashboard, but there's no real Stripe checkout or webhook wired up. **When the user is ready to launch/charge real customers, revisit this and set up actual Stripe billing** (needs a Stripe account + API keys + price IDs from the user — not something Claude can self-serve).
- Supabase org is on the **Free plan** — "Leaked Password Protection" (Attack Protection → Prevent use of leaked passwords) is Pro-tier-gated and can't be enabled until upgraded. Revisit before onboarding real customers at scale.
- Only the seeded Owner (Dr Sarah / `medicalvirtuals88@gmail.com`) has a real linked account. The other seeded staff (Dr James, Amy, Dr Naledi) have no email on file — decide if they're real invites or should be removed.
- Client portal doesn't expose photo_sets/documents yet (deliberate, per original scope) — revisit if clients should see their own before/after photos.
