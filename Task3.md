# Task 03 - Integration Design

## End-to-End Integration

When a patient submits the consultation form, the landing page sends the form data to the backend.

The backend first sends the data to HubSpot using the **HubSpot CRM API** because the landing page uses a custom HTML form. The backend first checks if the phone number already exists in HubSpot. HubSpot normally checks duplicate contacts using email, but this form only collects a phone number. So, I would search by phone number first. If the phone number already exists, the contact will be updated. Otherwise, a new contact will be created.

The contact will include:

- Name
- Phone Number
- Clinic Preference
- Source = Google Ads - Consultation Landing Page
- Lead Status = New Enquiry

After the contact is successfully created or updated, the backend sends a request to the **Karix WhatsApp Business API**. Karix then sends a confirmation message to the patient.

At the same time, the website pushes the **`consultation_form_submitted`** event to the `dataLayer`. Google Tag Manager (GTM) listens for this event and sends it to GA4. The conversion is then imported into Google Ads so the campaigns can optimize for completed consultation forms.

---

## Biggest Failure Point

The biggest failure point is the HubSpot API.

If HubSpot is unavailable or returns an error, the contact may not be created.

To avoid losing leads, I would save the failed request in the backend and retry it automatically after a short time. This helps make sure every enquiry is eventually stored in HubSpot.

---

## WhatsApp SLA (Within 2 Minutes)

The WhatsApp message should be sent within 2 minutes after the form is submitted.

Possible reasons for delay are:

- Slow backend response
- HubSpot API failure
- Karix API delay
- Network issues

To monitor this, I would log the form submission time, the API response time, and the WhatsApp delivery status.

If the message is not sent within 2 minutes or an API request fails, the system should retry the request and create an alert so the issue can be checked quickly.