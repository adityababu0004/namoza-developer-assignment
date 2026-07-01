# Task 01 – GTM Event Schema for OrthoNow

## 1. Event Tracking Table

The table below shows the events I would track on the OrthoNow website using Google Tag Manager.

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
| :--- | :--- | :--- | :--- |
| `booking_step_complete` | Custom Event | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration |
| `consultation_form_submitted` | Custom Event | `clinic_location`, `submission_source`, `specialty` | Conversions |
| `call_now_click` | Click Trigger | `click_url`, `page_location`, `button_position` | Events Report |
| `whatsapp_chat_open` | Click Trigger | `page_location`, `device_type`, `click_text` | Events Report |
| `patient_guide_download` | Link Click Trigger | `file_name`, `page_location`, `guide_name` | Downloads Report |
| `clinic_location_view` | Page View | `clinic_name`, `city`, `page_title` | Pages & Screens |
| `blog_scroll_depth` | Scroll Trigger | `percent_scrolled`, `page_title`, `article_category` | Engagement Report |

---

# 2. Booking Form Funnel Tracking

The booking form is a multi-step JavaScript form. Since the page does not reload between steps, GTM cannot automatically detect when the user moves from one step to another.

The front-end should send a `dataLayer.push()` after each successful step.

## Step 1 – Location & Specialty Selected

**GTM Trigger**

- Trigger Type: Custom Event
- Event Name: `booking_step_complete`

**JSON**

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

---

## Step 2 – Patient Details Entered

**GTM Trigger**

- Trigger Type: Custom Event
- Event Name: `booking_step_complete`

**JSON**

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

---

## Step 3 – Booking Confirmed

**GTM Trigger**

- Trigger Type: Custom Event
- Event Name: `booking_step_complete`

**JSON**

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

After the booking is completed, the final conversion event should also be sent.

```json
{
  "event": "consultation_form_submitted",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "submission_source": "booking_form"
}
```

---

## Who writes the dataLayer.push()?

The front-end developer writes the `window.dataLayer.push()` because it is part of the website's JavaScript. GTM does not create these events by itself. It only listens for the events sent by the front-end and forwards them to GA4.

---

## How I would brief the front-end developer

I would ask the front-end developer to add a `window.dataLayer.push()` after each booking step is successfully completed and before the form moves to the next step.

Each event should include:

- `event`
- `step_number`
- `step_name`
- `clinic_location`
- `specialty`

The JSON payloads above can be used directly during implementation.

---

## GA4 Funnel Exploration

To measure where users leave the booking process:

1. Open **GA4 → Explore → Funnel Exploration**.
2. Create three funnel steps:
   - Step 1 → Event = `booking_step_complete` where `step_number = 1`
   - Step 2 → Event = `booking_step_complete` where `step_number = 2`
   - Step 3 → Event = `booking_step_complete` where `step_number = 3`
3. Save the funnel.

This report will show how many users complete each step and where users drop off before finishing the booking process.

---

# 3. Google Ads Conversion

I would import **`consultation_form_submitted`** as the primary conversion into Google Ads.

### Why?

A completed consultation form is the strongest conversion signal because it shows that the user has successfully submitted a booking request.

Events such as `call_now_click` or `whatsapp_chat_open` only show interest. A user may click those buttons without actually contacting the clinic. Optimizing campaigns using `consultation_form_submitted` helps Google Ads focus on users who are more likely to become qualified leads.