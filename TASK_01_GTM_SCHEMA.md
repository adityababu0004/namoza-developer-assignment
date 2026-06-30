# Task 01 — GTM Event Schema for OrthoNow

## Complete Event Schema

| # | Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|-----------|-------------|----------------|----------------------|
| 1 | `booking_step_complete` | Custom Event (dataLayer) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1); `step_number`, `step_name`, `form_field_count` (step 2); `step_number`, `step_name`, `booking_date`, `clinic_location` (step 3) | **Funnel Exploration** — 3-step booking funnel drop-off; **Audience** — users who completed step 1 but not step 3 (retargeting) |
| 2 | `consultation_form_submitted` | Custom Event (dataLayer) | `clinic_location`, `specialty`, `preferred_date` | **Conversions report** — primary conversion; **Audience** — all converters for lookalike/exclusion |
| 3 | `call_now_click` | GTM Click Trigger (CSS selector / `data-action="call-now"`) | `click_url` (tel: number), `page_location`, `button_position` (header / clinic-page / landing-page) | **Events report** — call intent volume by page; **Audience** — high-intent users who called |
| 4 | `whatsapp_chat_open` | GTM Click Trigger (outbound click to `wa.me`) | `page_location`, `device_category`, `click_text` | **Events report** — WhatsApp engagement; **Audience** — WhatsApp-preferring users |
| 5 | `patient_guide_form_submit` | Custom Event (dataLayer) | `guide_name`, `page_location`, `form_fields_filled` | **Events report** — gated content performance; **Audience** — guide downloaders for nurture |
| 6 | `patient_guide_download` | Custom Event (dataLayer — fires after form gate clears) | `guide_name`, `file_url`, `page_location` | **Events report** — actual download confirmation |
| 7 | `clinic_location_view` | GTM History Change or Pageview trigger | `page_location`, `clinic_name`, `city` | **Pages and screens report** — per-clinic traffic; **Audience** — city-specific visitors (Bengaluru / Hyderabad / Chennai) |
| 8 | `blog_scroll_depth` | GTM Scroll Depth Trigger (25%, 50%, 75%, 90%) | `percent_scrolled`, `page_title`, `article_category` | **Events report** — content engagement quality; **Audience** — deep readers (90%+) for remarketing |
| 9 | `blog_article_read` | Custom Event (dataLayer — fires at 90% scroll + 30s time on page) | `page_title`, `article_category`, `time_on_page` | **Engagement report** — qualified reads vs. bounces |

---

## 3-Step Booking Form — Funnel Tracking

### Why Custom dataLayer Pushes Are Required

GTM **cannot natively detect** transitions between steps of a client-side multi-step form. If the form is a single-page component that shows/hides steps via JavaScript (which is the standard pattern), there is no URL change, no DOM event, and no history change for GTM to listen to. The **front-end developer must write explicit `dataLayer.push()` calls** in the form's step-transition logic.

**Who writes the dataLayer push?** The developer who owns the form code. GTM only *listens* for these events — it does not generate them. As the developer on this account, I would write the `dataLayer.push()` calls directly in the form's JavaScript, or brief the client's front-end team with exact specifications if we don't own the form codebase.

### How I Would Brief the Dev Team

> **Developer Brief — Booking Form dataLayer Implementation**
>
> For each step transition in the booking form, add a `window.dataLayer.push()` call at the point in your JavaScript where the user successfully completes a step and the UI advances to the next one. Do **not** fire on page load. Do **not** fire on "back" navigation between steps. Fire **only** on forward progression after validation passes.
>
> Here are the exact payloads for each step. Copy these into your step-transition handlers. Replace the template variables (`{{...}}`) with the actual form field values at that point in time.

---

### Step 1 — Location & Specialty Selected

**When it fires:** User selects a clinic location and specialty, then clicks "Next" / "Continue". The push fires *after* front-end validation confirms both fields are filled, and *before* or *as* the UI transitions to Step 2.

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name — e.g. 'Koramangala, Bengaluru'}}",
  "specialty": "{{specialty selected — e.g. 'Knee Replacement'}}",
  "form_id": "ortho_booking_form"
}
```

**GTM Trigger:** Custom Event trigger matching `booking_step_complete` where `step_number` equals `1`.

---

### Step 2 — Patient Details Entered

**When it fires:** User fills in name, phone, and preferred date, then clicks "Next". The push fires *after* front-end validation (name non-empty, phone is valid 10-digit Indian mobile, date is a future date).

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "{{clinic name — carried from step 1}}",
  "specialty": "{{specialty — carried from step 1}}",
  "has_preferred_date": true,
  "form_id": "ortho_booking_form"
}
```

> **Note:** We do NOT push the patient's name, phone number, or date into the dataLayer. These are PII/PHI fields. GA4 terms of service prohibit sending personally identifiable information. We push only non-PII confirmation flags like `has_preferred_date: true`.

**GTM Trigger:** Custom Event trigger matching `booking_step_complete` where `step_number` equals `2`.

---

### Step 3 — Booking Confirmed

**When it fires:** User reviews their information and clicks "Confirm Booking". The push fires *after* the API call returns a success response (or, if no API, after the UI transitions to the confirmation/thank-you state).

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name — carried from step 1}}",
  "specialty": "{{specialty — carried from step 1}}",
  "booking_date": "{{preferred date — e.g. '2026-07-15'}}",
  "form_id": "ortho_booking_form",
  "transaction_status": "success"
}
```

Additionally, fire the final conversion event:

```json
{
  "event": "consultation_form_submitted",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty}}",
  "submission_source": "booking_form"
}
```

**GTM Trigger:** Custom Event trigger matching `booking_step_complete` where `step_number` equals `3`. A separate trigger for `consultation_form_submitted` fires the GA4 conversion tag and the Google Ads conversion tag.

---

### Surfacing Step-Level Drop-Off in GA4 Funnel Exploration

1. **Navigate to:** GA4 → Explore → Funnel Exploration.
2. **Create a new exploration** with the following funnel steps:
   - **Step 1:** Event = `booking_step_complete`, Parameter condition: `step_number` = `1`
   - **Step 2:** Event = `booking_step_complete`, Parameter condition: `step_number` = `2`
   - **Step 3:** Event = `booking_step_complete`, Parameter condition: `step_number` = `3`
3. **Set the funnel to "Closed Funnel"** so only users entering at Step 1 are counted. This prevents users who somehow land on Step 2 directly from skewing completion rates.
4. **Add a breakdown dimension** of `clinic_location` to identify if specific clinics have higher drop-off (possibly due to availability issues or UX friction for that location's offerings).
5. The funnel visualization will show:
   - Step 1 → Step 2 drop-off: indicates friction in the patient details form (common: phone number validation issues, unclear date picker).
   - Step 2 → Step 3 drop-off: indicates hesitation at confirmation (common: price not shown, no cancellation policy visible).

---

## Google Ads Conversion Action

**Recommended conversion to import:** `consultation_form_submitted`

**Why this one over the others:**

1. **It represents bottom-of-funnel intent.** A completed booking form submission is the closest measurable proxy to an actual clinic visit. `call_now_click` and `whatsapp_chat_open` are intent signals, but they don't confirm engagement — a user can tap "Call" and hang up, or open WhatsApp and never send a message.

2. **It provides sufficient volume for Smart Bidding.** Google Ads' automated bidding strategies (Target CPA, Maximize Conversions) require 15–30 conversions per month to optimize effectively. A booking form submission is frequent enough to train the algorithm, unlike a downstream event like "appointment attended" which would have a significant reporting delay and lower volume.

3. **It has a clean 1:1 signal.** Each form submission is a discrete, deduplicated action (one submission per session). Click events like `call_now_click` can fire multiple times per session, inflating conversion counts and confusing the bidding algorithm.

4. **Attribution window alignment.** Form submissions happen in the same session as the ad click (or within a short window), giving Google Ads a tight feedback loop for optimizing which clicks lead to conversions.

> **Configuration:** In Google Ads, import this as a "Primary" conversion action (not "Secondary/Observation") so it directly influences automated bidding. Set the counting method to "One" (not "Every") to avoid double-counting if a user resubmits.
