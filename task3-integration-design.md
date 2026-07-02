# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## End-to-End Architecture

When a patient submits the consultation form, the following sequence executes:

**Step 1 — Form submission (Front-end)**
The landing page fires `window.dataLayer.push({ event: 'consultation_form_submitted', ... })`, which GTM picks up and immediately fires the Google Ads conversion tag via the GTM container. This keeps the conversion signal in-browser — no server round-trip required, and it fires before anything else can fail.

**Step 2 — Data relay (Netlify / Vercel serverless function)**
On form submit, the front-end also makes a `fetch()` POST to a lightweight serverless function (e.g. Netlify Functions or a Vercel Edge Function). This function receives `name`, `phone`, and `clinic_preference`. I use a serverless function rather than a direct HubSpot Forms API call from the browser because: (a) it hides the HubSpot API key from client-side code, and (b) it lets me orchestrate both the HubSpot write and the Karix WhatsApp dispatch from one place with proper error handling.

**Step 3 — HubSpot contact creation (HubSpot Contacts API v3, not Forms API)**
The serverless function calls the **HubSpot Contacts API** directly — not the native form embed and not Zapier. Reason: the form collects phone but not email, and HubSpot's native form deduplication operates on email by default. Using the API gives us explicit control.

The deduplication call sequence:
1. `GET /crm/v3/objects/contacts?filterGroups` — search by `phone` property first
2. If a match exists → `PATCH` (update) that contact
3. If no match → `POST` (create) a new contact

Properties set on create/update:
```json
{
  "firstname": "<name>",
  "phone": "<phone>",
  "clinic_preference": "<value>",
  "lead_source": "Google Ads - Consultation Landing Page",
  "hs_lead_status": "NEW"
}
```

**Step 4 — WhatsApp confirmation (Karix API)**
Immediately after the HubSpot write resolves, the same serverless function POSTs to the Karix WhatsApp Business API, sending a pre-approved template message to the patient's number. The template reads: _"Hi [Name], your OrthoNow consultation request has been received. Our team will call you within 30 minutes. — OrthoNow [Clinic]"_

---

## Biggest Failure Point & Fallback

**The single biggest failure point is the Karix WhatsApp dispatch.**

HubSpot writes are synchronous and relatively reliable. The Karix API introduces a third-party dependency with its own uptime envelope, and Indian telecom delivery (DLT regulations, carrier filtering) adds further variability.

Fallback architecture:
- If the Karix API call fails (timeout > 5s or non-2xx response), the serverless function writes the failed job to a **retry queue** (e.g. an Upstash Redis queue or a simple Supabase table with a `status = 'pending'` flag).
- A separate cron function (running every 90 seconds) polls the queue and retries failed Karix calls up to 3 times before marking them `failed` and alerting the ops team via Slack webhook.
- The form submission is considered successful regardless — the patient sees the thank-you state immediately. The WhatsApp failure is an internal ops issue, not a patient-facing one.

---

## WhatsApp 2-Minute SLA — Risk & Monitoring

**What can break the SLA:**

1. **Karix API latency** — upstream delays on their message queue
2. **DLT template approval lag** — if the template is not pre-approved by TRAI-registered DLT, the message queues indefinitely
3. **Phone number format errors** — if the number reaches Karix without the `+91` prefix, delivery fails silently
4. **Serverless cold start** — on low-traffic periods, Netlify/Vercel cold starts can add 800ms–2s latency before the Karix call even initiates

**Monitoring setup:**
- Log a timestamp at form submit and a second timestamp when Karix returns a `200 OK` with a message ID
- Ship both timestamps to a simple monitoring dashboard (e.g. Datadog or even a Supabase table with a Retool view)
- Alert via Slack if `karix_sent_at - form_submitted_at > 90 seconds` — giving ops time to intervene before the 2-minute SLA is breached
- Weekly report on SLA adherence rate; target > 95%

---

## HubSpot Phone Deduplication — Critical Note

HubSpot's default deduplication key is **email**, not phone. This form collects no email. If two patients submit with the same phone number but different names, a naive `POST` to the Contacts API creates **duplicate contacts**, both assigned `Lead Status = New Enquiry`, and both triggering a WhatsApp message — a confusing and unprofessional patient experience.

The explicit search-before-write sequence (Step 3 above) prevents this. Phone should also be configured as a **unique property** in HubSpot Settings → Properties → Contact Properties → Phone Number → mark as unique identifier. This enforces deduplication at the HubSpot data model level as a second safety net.
