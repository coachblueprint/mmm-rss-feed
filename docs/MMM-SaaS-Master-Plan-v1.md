# MMM SaaS Master Plan v1

**Product:** Monday Morning Motivators (MMM) as a multi-tenant SaaS
**Repo:** coachblueprint/mmm-rss-feed (SaaS app lives in `app/`; legacy feed system untouched at existing paths)
**Written:** 2026-08-19, planning session in Claude mobile chat (cloud-native, GitHub MCP)
**Status:** DRAFT - owes an adversarial plan-review from a cold context before Slice 1 starts
**Session prefix:** MMM

---

## 1. What we're building

A subscription product for inspection companies (and other realtor-facing service businesses) that sends a branded motivational email to their real estate agent list every Monday morning. The product ships with a five-year, 260-week pre-built calendar of quotes and images. Tenants customize branding (fonts, shadows, header, footer, custom message), override individual weeks, upload or AI-generate images within quota, ingest their agent contacts, and get per-agent engagement reporting. Sending is our own infrastructure (Resend), not GHL RSS campaigns.

TexInspec is tenant #1 and migrates off the legacy RSS/GHL pipeline only after the new pipeline has proven itself on a test list.

## 2. Non-disruption contract (read before every slice)

The legacy system currently sends TexInspec's Monday email and MUST NOT break:

- Google Sheet (source of truth, 260 rows)
- `.github/workflows/sunday-rotate.yml` and `monday-rotate.yml`
- `scripts/generate_feed.py`
- `mmm-feed.xml` at repo ROOT (GHL polls the raw URL of this exact path)
- The GHL preview and production campaigns

Rules:
1. **Do not move, rename, or edit any legacy file.** The raw.githubusercontent.com URL GHL polls encodes the file path; moving `mmm-feed.xml` breaks Monday's send.
2. **The repo stays PUBLIC** until the legacy pipeline is retired (Slice 10). Flipping it private 401s the raw URL. Consequence: nothing sensitive ever lands in this repo while public - no secrets (Doppler only), no customer data, no tenant lists. Code being visible is accepted temporarily.
3. The new app lives entirely under `app/` with its own Vercel project (root directory = `app/`).
4. Any slice that touches GHL, the Sheet, the Actions, or repo settings is a GATE: stop and wait for Jonathan.

## 3. Decisions already made (with Jonathan, 2026-08-19)

- **Customer:** inspection companies primarily; schema stays generic enough for other service businesses.
- **Sending:** Resend. Pro plan ($20/mo, 50k emails) with pay-as-you-go overage covers early scale; a 500-agent tenant is ~2,200 emails/mo. v1 sends from a shared MMM sending subdomain with tenant display name and tenant reply-to. **v2:** tenant-owned custom domains (tenant adds DNS records, sends from their own domain).
- **Contact ingestion v1:** CSV upload with column mapping + dedupe, manual add/edit, and "zapper style" email ingest - each tenant gets a unique inbound address (Resend inbound routing); forwarding an email adds/updates the agent. **v2:** public API for direct integration (auto-tracks every agent), GHL/ISN importers.
- **Images:** seed from Jonathan's existing library; tenants can upload their own; tenants can AI-generate via our OpenAI credits under a style guide + guardrails, quota ~5 generations/month/tenant.
- **Calendar:** all 260 weeks exist, but tenants see and edit a rolling 26-week window. Per-week overrides (swap quote, image, subject). Plus date-ranged broadcast blocks ("append this coupon to every send starting September").
- **Billing:** Stripe from day one, reusing the black-bird-reviews / past-skies integration pattern with placeholder credentials, swapped before go-live.
- **Brand:** plan under "Monday Morning Motivators"; pivot allowed near launch. Domain TBD (Slice 0 dependency).
- **Supabase:** new project, coachblueprint org, **Micro tier minimum** (Merlin Nano-tier outage lesson).
- **Secrets:** coachblueprint Doppler. `doppler run --` prefix for all env-needing commands.
- **Mail rule:** all outbound mail through a shared `sendEmail` helper hitting Resend. SMTP inside Edge Functions is a known dead end.

## 4. Architecture

- **Frontend/app:** Next.js (App Router) + TypeScript on Vercel, project root `app/`. Follows the PPP / past-skies multi-tenant reference pattern.
- **DB/auth/storage:** new Supabase project. RLS on every tenant table keyed by `tenant_id`. Supabase Storage for image library, tenant uploads, and rendered composites.
- **Compositor:** two renderers, one source of truth.
  - *Editor preview:* client-side (HTML canvas or absolutely-positioned DOM) driven by a `design_config` JSON (font, size, color, shadow x/y/blur/opacity, position, header/footer blocks, message block).
  - *Send-time render:* server-side from the SAME `design_config` (satori + sharp, or node-canvas - Slice 2 spike decides), producing a final PNG stored in Supabase Storage. Email HTML references the hosted PNG. Render on save/override, never live at send time for the whole list.
- **Scheduler:** one dispatcher cron (Vercel cron or a SINGLE pg_cron job - do not proliferate pg_cron jobs; Merlin lesson). Dispatcher finds tenants due for Sunday preview or Monday 8 AM send in the tenant's timezone and enqueues batches.
- **Email events:** Resend webhooks (delivered, opened, clicked, bounced, complained) into `send_events`. Bounces/complaints auto-write suppressions.
- **Compliance baked in:** one-click List-Unsubscribe header + footer link, per-tenant physical address in footer (CAN-SPAM), suppression respected at send time, global unsubscribe page.

## 5. Data model (first cut - Slice 1 refines)

- `tenants` - name, slug, status, plan, stripe ids, timezone, physical address, reply_to, sender display name, ingest_address
- `tenant_users` - Supabase auth users x tenant, role
- `master_calendar` - 260 rows: week_of (Monday date), quote, author, subject_line, default_image_id
- `images` - master library + tenant-scoped (tenant_id nullable), source (library|upload|ai), storage path, dimensions
- `ai_generations` - tenant_id, month, prompt, image_id (quota enforcement: count per tenant per month)
- `tenant_design` - the design_config JSON (fonts, shadow, header/footer builder output, message block), versioned
- `week_overrides` - tenant_id x week_of: quote/image/subject overrides, rendered_png path
- `broadcast_blocks` - tenant_id, html/text block, start_date, end_date nullable ("coupon on every send from September")
- `contacts` - tenant_id, name, email, phone, brokerage, source (csv|manual|ingest|api), status
- `suppressions` - tenant_id scope + global scope, reason (unsub|bounce|complaint|manual)
- `sends` - one row per tenant per week per type (preview|production), status, counts
- `send_recipients` / `send_events` - per-contact delivery + engagement trail (this is the per-agent auto-tracking)
- `billing` - subscription state mirror from Stripe webhooks

## 6. Slices

Ordered so the riskiest unknown (compositor) is proven before anything depends on it, and so every slice ends in something demonstrable. Each slice: branch per slice, green Vercel preview as the gate, PR, merge. Verification loop is listed per slice; the build session must run it before calling the slice done.

### Slice 0 - Foundations (GATED: Jonathan actions required)
- Create Supabase project (Micro), coachblueprint org. GATE: cost confirm.
- Scaffold Next.js app under `app/`, new Vercel project pointed at `app/`, connected to this repo.
- Doppler config for the project; env plumbing via `doppler run --`.
- Jonathan-only: Resend account/API key into Doppler; choose + verify the shared sending subdomain (DNS); OpenAI key into Doppler; Stripe placeholder keys into Doppler; provide the image library files and a CSV export of the 260-row Sheet.
- Verify: app deploys to Vercel preview, connects to Supabase, sends one test email via Resend from the verified subdomain.

### Slice 1 - Tenancy, auth, schema
- Full schema + RLS + migrations (applied live via Supabase MCP with permission; committed .sql is the record).
- Auth (Supabase), tenant creation, tenant_users, basic settings page (timezone, address, reply-to, display name).
- Seed `master_calendar` from the Sheet CSV and `images` from the library.
- Verify: two test tenants cannot see each other's rows through the API (adversarial RLS check, not just happy path).

### Slice 2 - Compositor spike (the unknown; do before the editor UI)
- Prove server-side render: quote text on image with font choice, configurable shadow, wrapping, safe margins, at email-friendly dimensions (target ~1200px wide, retina-aware).
- Decide satori+sharp vs node-canvas based on font fidelity and Vercel runtime fit. Record the decision and why in `app/docs/compositor-decision.md`.
- Curate the v1 font list (licensed/open fonts only, self-hosted files).
- Verify: golden-file test - 5 quote/image/font/shadow combos render deterministically; long-quote wrapping doesn't overflow.

### Slice 3 - The editor
- Full-service editor: pick week, live preview, font picker, size, color, shadow controls, text position; header builder (logo upload, colors, layout presets); footer builder (contact info, socials, compliance line locked in); custom message block.
- Save writes design_config + triggers server render; preview pixel-matches the server render within tolerance.
- Rolling 26-week calendar view with per-week override UX and broadcast blocks UX.
- Verify: editor output re-rendered server-side matches preview on the 5 golden combos; a non-technical user (Jonathan counts) can rebrand a week in under 5 minutes.

### Slice 4 - Contacts
- CSV upload with column mapping, dedupe on email, error report.
- Manual add/edit/archive. Suppression list view.
- Email ingest: Resend inbound route per tenant, parser extracts contact from forwarded email, logs ambiguous parses for review instead of guessing.
- Verify: messy real-world CSV (mixed case, dupes, bad emails) imports with correct counts; forwarded email creates the right contact.

### Slice 5 - Send pipeline
- Dispatcher cron, batch sender via Resend (respect rate limits), Sunday tenant-admin preview + Monday production send in tenant timezone.
- List-Unsubscribe one-click + hosted unsubscribe page + suppression enforcement.
- Webhooks into send_events; bounce/complaint auto-suppression.
- Verify: end-to-end on a seeded test tenant with a 10-address test list (mailbox providers mixed); unsubscribe round-trips; a suppressed contact provably not sent.

### Slice 6 - Reports
- Per-send dashboard (sent/delivered/opened/clicked/unsubbed/bounced) and per-agent engagement history.
- Weekly tenant email report (uses the same send pipeline).
- Verify: numbers reconcile against Resend's dashboard for the test sends.

### Slice 7 - AI image generation
- OpenAI image generation behind the style guide prompt scaffold + guardrails (content filter on prompts, brand-safe style enforcement), 5/month/tenant quota, generated images land in the tenant library.
- Verify: quota blocks the 6th generation; a prompt-injection style prompt ("ignore style guide...") still comes out on-style or is refused.

### Slice 8 - Billing
- Stripe subscription (pattern from black-bird-reviews / past-skies), webhook mirror, gate sending on active subscription, trial state.
- Placeholder credentials; swap is a go-live task.
- Verify: test-mode subscribe/cancel flips tenant sending eligibility.

### Slice 9 - TexInspec pilot (GATED)
- TexInspec created as tenant #1, real agent list imported, branding built in the editor.
- Runs in PARALLEL with the legacy pipeline: new system sends only to a small test list while legacy keeps sending production. Minimum two clean parallel Mondays.
- Verify: side-by-side - new send matches or beats the legacy email; engagement events land correctly.

### Slice 10 - Cutover + retire legacy (GATED, Jonathan drives)
- Point TexInspec production list at the new pipeline; PAUSE (not delete) the GHL campaigns; watch two production Mondays.
- Then: disable the legacy Actions, archive `scripts/` + workflows in place with a README note, flip the repo PRIVATE (safe now - nothing polls the raw URL), rotate anything that was ever visible while public out of caution.
- Rollback path at every step: unpause GHL campaign, re-enable Actions - legacy files were never touched.

## 7. Division of labor

**Jonathan only (gates):** Supabase project cost confirm; all Doppler secret writes; Resend account + DNS records for the sending subdomain; OpenAI + Stripe keys; Sheet CSV export + image library handoff; GHL campaign pause/changes; repo visibility flip; anything that emails real agents.

**Claude builds everything else,** looping per-slice: build, run the slice's verification, fix, re-verify, PR with the verification evidence in the body, merge on green.

## 8. Risks / open items

1. **Compositor font fidelity on Vercel serverless** - the Slice 2 spike exists to kill this early. Fallback: render in a Supabase Edge Function or a small Fly/Railway render service.
2. **Deliverability on a shared subdomain** - one bad tenant list hurts everyone. Mitigations: import-time validation, complaint-rate monitoring per tenant with auto-pause, DMARC/SPF/DKIM correct from day one (bbirdr playbook). Custom domains in v2 are the real fix.
3. **Rendered-image-only emails** can trip spam filters (image-to-text ratio). Mitigation: email HTML includes real text (quote as text under the image, custom message as text), image is the hero not the whole body.
4. **Public repo during build** - accepted, but every PR must pass secret scanning and never embed tenant/customer data in fixtures.
5. **Timezone sends** - one dispatcher, tenant-timezone aware; test DST boundaries.
6. **Quota/abuse on AI images** - hard quota table, prompts logged.
7. **Brand/domain undecided** - blocks the sending subdomain choice in Slice 0; needs an answer before DNS setup, even if provisional.

## 9. Plan-review debt

This plan was authored in this session and per dev-conventions MUST receive an adversarial review from a cold context (session that did not write it) before Slice 1 begins. Expected challenge areas: slice granularity of 3 (editor may need splitting), whether Slice 9's two-parallel-Mondays gate is sufficient, and the shared-subdomain deliverability posture.
