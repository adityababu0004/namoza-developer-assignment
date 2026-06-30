# Namoza Developer Assignment — OrthoNow

**Candidate:** Aditya Raj  
**Role:** Developer - Position 1 (Client Web + Martech)  
**Client:** OrthoNow — 9 orthopaedic clinics across Bengaluru, Hyderabad & Chennai

---

## Repository Structure

```
├── README.md                    ← You are here
├── TASK_01_GTM_SCHEMA.md        ← Task 1: GTM Event Schema + dataLayer JSON
├── index.html                   ← Task 2: Landing Page (single self-contained file)
├── TASK_03_INTEGRATION.md       ← Task 3: Integration Design (written answer)
└── pagespeed-screenshot.png     ← PageSpeed Insights score (to be added)
```

---

## Task 01 — GTM Event Schema

**File:** [`TASK_01_GTM_SCHEMA.md`](./TASK_01_GTM_SCHEMA.md)

Contains:
- **Complete event schema table** covering all 9 key interactions (booking form steps, Call Now, WhatsApp, Patient Guide download, clinic page views, blog scroll depth).
- **3-step booking form funnel tracking** with exact `dataLayer.push()` JSON for each step — including the critical point that GTM cannot natively detect multi-step form transitions without custom dataLayer pushes written by the front-end developer.
- **GA4 Funnel Exploration setup** instructions for visualizing step-level drop-off.
- **Google Ads conversion action** recommendation (`consultation_form_submitted`) with justification.

### Key Technical Decision
The dataLayer pushes for the booking form are written by the developer who owns the form code — not configured in GTM. GTM only *listens* for these custom events via Custom Event triggers. The schema document includes a developer briefing template for handing off implementation to the client's front-end team.

---

## Task 02 — Landing Page

**File:** [`index.html`](./index.html)

A single self-contained HTML file. No build tools, no server, no external dependencies.

### How to Run
Open `index.html` in any modern browser. That's it.

### How to Verify the dataLayer Push
1. Open `index.html` in Chrome.
2. Open DevTools → Console (`Cmd+Option+J` / `Ctrl+Shift+J`).
3. Fill in the form with a valid name and 10-digit phone number (starting with 6-9).
4. Click "Book My Free Consultation →".
5. You should see in the console:
   ```
   [OrthoNow] dataLayer push fired: {event: "consultation_form_submitted", form_name: "consultation_booking", ...}
   [OrthoNow] Full dataLayer: [...]
   ```
6. Verify: Type `window.dataLayer` in the console to inspect the full array.

### Technical Highlights
- **Zero external requests** — no Google Fonts, no CDN CSS, no images. System font stack (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto...`) for instant text rendering.
- **Inline SVGs** for logo and icons — no HTTP image requests.
- **Mobile-first responsive** — designed for mobile, scales up to desktop. Not "desktop shrunk to mobile."
- **Form validation** — validates 10-digit Indian mobile numbers (must start with 6-9), name must be ≥2 characters.
- **dataLayer push fires ONLY on valid form submission** — not on page load, not on invalid submit.
- **Thank-you state** renders in-place without a page reload.
- **Accessibility** — semantic HTML5 elements, `aria-label`, `aria-required`, `aria-live="polite"` on thank-you state, `role="alert"` on error messages.

### PageSpeed Target
The page is designed to score **90+ on PageSpeed Insights Mobile** due to:
- Single file, no external resources (zero render-blocking requests)
- No web fonts to download (eliminates FOIT/FOUT)
- No images to load (inline SVGs only)
- Minimal CSS and JS (all inline, no parsing of external files)
- Semantic HTML structure

---

## Task 03 — Integration Design

**File:** [`TASK_03_INTEGRATION.md`](./TASK_03_INTEGRATION.md)

A 300-400 word written answer covering:

1. **End-to-end architecture:** Form → Custom serverless middleware (Cloudflare Workers) → HubSpot CRM (Search API + Create/Update) + Karix WhatsApp API + Google Ads conversion (client-side via GTM).

2. **HubSpot phone deduplication:** HubSpot's default dedup key is *email*, not phone. Since our form doesn't collect email (standard in Indian healthcare lead gen), we use the HubSpot Search API to find existing contacts by phone number before creating/updating — a search-then-upsert pattern.

3. **WhatsApp 2-minute SLA:** Risks (Karix downtime, template expiry, cold starts, upstream HubSpot latency) and monitoring (timestamp logging, latency alerts at 90s threshold, delivery receipt webhooks, option to decouple WhatsApp from HubSpot for parallel execution).

4. **Biggest failure point:** HubSpot Search API timeout — mitigated with a dead-letter queue + retry scheduler + team alerts.

---

## Loom Video

> **[Loom link to be added here]**
>
> Walkthrough structure (max 8 minutes):
> 1. GTM schema decisions (2 min)
> 2. Live demo of landing page + dataLayer push in browser console (3 min)
> 3. Integration architecture answer (3 min)

---

## Contact

Submission sent to naman@namoza.com  
Subject: `Developer Assignment - Aditya Raj`
