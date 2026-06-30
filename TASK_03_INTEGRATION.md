# Task 03 — Integration Design: Landing Page → HubSpot + WhatsApp + Google Ads

## End-to-End Architecture

When a patient submits the consultation form on the landing page, the following sequence executes:

**Step 1 — Client-Side (Browser):**
The form's `submit` handler fires `window.dataLayer.push({event: 'consultation_form_submitted', ...})` for GTM/GA4/Google Ads tracking, then sends the form payload (Name, Phone, Clinic Preference) via a `fetch()` POST request to a lightweight **serverless middleware function** (hosted on Cloudflare Workers or AWS Lambda behind API Gateway). I choose a custom middleware over Zapier/Make because it gives us full control over the deduplication logic, retry behaviour, and response times — critical when we have a 2-minute WhatsApp SLA. Zapier's execution queue can introduce 1–5 minute delays on lower-tier plans, which is unacceptable here.

**Step 2 — Middleware (Serverless Function):**
The middleware receives the payload and executes three operations in a controlled sequence:

1. **HubSpot Contact Upsert (Search-then-Create pattern):** HubSpot's default deduplication key is **email**, not phone. Since our form collects only Name + Phone (no email — standard in Indian healthcare lead gen), we cannot rely on native dedup. Instead, the middleware calls the **HubSpot Search API** (`POST /crm/v3/objects/contacts/search`) filtering by the `phone` property. If a match is found, it calls the **Update API** (`PATCH /crm/v3/objects/contacts/{id}`) to update the existing record with the new Name and Clinic Preference. If no match is found, it calls the **Create API** (`POST /crm/v3/objects/contacts`) with properties: `firstname`, `phone`, `clinic_preference` (custom property), `lead_source` = `"Google Ads - Consultation Landing Page"`, and `lead_status` = `"New Enquiry"`. This search-then-upsert pattern prevents duplicate contacts when the same patient submits multiple times.

2. **WhatsApp Confirmation via Karix API:** After the HubSpot upsert succeeds, the middleware makes a `POST` request to the **Karix WhatsApp Business API** endpoint with the patient's phone number and a pre-approved template message (e.g., *"Hi {{name}}, thank you for booking a consultation at OrthoNow {{clinic}}. Our team will confirm your appointment within 24 hours."*). Template messages must be pre-approved by WhatsApp/Karix — this is not a free-text message.

3. **Response:** The middleware returns a `200 OK` to the browser. The landing page shows the thank-you state.

**Step 3 — GTM/Google Ads (Client-Side, Parallel to Step 2):**
The `dataLayer.push` from Step 1 fires a GTM tag that sends the `consultation_form_submitted` event to GA4 (as a key event) and triggers a **Google Ads Conversion Tracking tag** using the Google Ads conversion ID and label. This happens entirely client-side via GTM and is independent of the server-side flow — so even if the middleware is slow, the conversion still records.

## Single Biggest Failure Point

**The HubSpot Search API call failing or timing out.** If the search fails, we cannot determine whether the contact exists, meaning we risk either creating a duplicate (if we fall back to "always create") or losing the lead entirely (if we bail out). This is the linchpin of the entire flow — every downstream action depends on it.

**Fallback:** Implement a **dead-letter queue** (e.g., an SQS queue or a simple Cloudflare KV store). If the HubSpot search/upsert fails after 2 retries with exponential backoff, the middleware writes the raw form payload to the dead-letter queue and still proceeds to fire the Karix WhatsApp message (so the patient isn't left waiting). A scheduled job (every 5 minutes) processes the dead-letter queue and retries the HubSpot upsert. An alert (Slack webhook or email) notifies the team of any queued failures so they can be manually resolved if retries exhaust.

## WhatsApp 2-Minute SLA — Risks and Monitoring

**What could break the SLA:**
- **Karix API downtime or rate limiting:** Third-party APIs have outages. Karix may throttle during high-volume periods.
- **Template message not approved:** If the WhatsApp template expires or is rejected during a re-approval cycle, all sends will silently fail.
- **Middleware cold start latency:** Serverless functions (especially on AWS Lambda with Node.js) can have 1–3 second cold starts — not a risk to the 2-minute SLA individually, but combined with retry logic and upstream latency, it adds up.
- **HubSpot upsert taking too long:** If the search-then-create takes 5+ seconds due to a large contact database, the WhatsApp send is delayed.

**Monitoring strategy:**
- **Log timestamps** at each stage in the middleware (request received, HubSpot upsert complete, Karix API called, Karix response received). Calculate `karix_response_timestamp - form_submit_timestamp` and log it as `whatsapp_delivery_latency_ms`.
- **Set a CloudWatch/Datadog alert** if `whatsapp_delivery_latency_ms` exceeds 90 seconds (giving 30 seconds of buffer before the 2-minute SLA).
- **Decouple WhatsApp from HubSpot if latency spikes:** If monitoring shows HubSpot is the bottleneck, restructure to fire the Karix WhatsApp call **in parallel** with the HubSpot upsert (instead of sequentially). The patient gets their confirmation immediately; the CRM record can settle asynchronously.
- **Karix delivery receipts:** Use Karix's webhook callback for delivery status (`sent`, `delivered`, `read`, `failed`). Log these and alert on `failed` status so the ops team can follow up manually with an SMS or phone call.
