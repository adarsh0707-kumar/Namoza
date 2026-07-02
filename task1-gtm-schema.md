# Task 01 — GTM Event Schema for OrthoNow

## Full Event Schema Table

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration report; Remarketing audience: dropped at step 2 |
| `booking_confirmed` | Custom Event (dataLayer push) | `clinic_location`, `specialty`, `preferred_date`, `booking_id` | Conversions; Audience: confirmed bookers |
| `call_now_click` | Click — Just Links (href `tel:`) | `click_location` (homepage/clinic-page/landing-page), `clinic_name`, `page_path` | Events report; Audience: high-intent callers |
| `whatsapp_click` | Click — Just Links (href `wa.me`) | `click_location`, `page_path`, `widget_type` (floating/inline) | Events report; Audience: WhatsApp engagers |
| `patient_guide_form_submit` | Custom Event (dataLayer push) | `form_name` ('patient_guide_gate'), `page_path`, `clinic_context` | Conversions (soft); Audience: guide downloaders |
| `patient_guide_download` | Custom Event (dataLayer push, fires after form validates) | `document_name`, `page_path`, `lead_captured` (true/false) | Events report |
| `clinic_page_view` | Page View — URL contains `/clinics/` | `clinic_name`, `clinic_city`, `page_path` | Pages & Screens report; Geo audience per city |
| `blog_scroll_depth` | Scroll Depth trigger (25/50/75/90%) | `scroll_threshold`, `article_title`, `article_category`, `page_path` | Engagement report; Audience: high-engagement readers |

---

## 3-Step Booking Form — Funnel Drop-off Tracking

### Architecture Overview

GTM **cannot** natively detect when a user moves between steps in a JS-rendered multi-step form. The front-end developer must push to `window.dataLayer` at each step transition. GTM listens for these pushes via Custom Event triggers.

---

### dataLayer Pushes — Exact JSON

**Step 1 — User selects clinic location and specialty, clicks "Next":**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement"
}
```

**Step 2 — User enters name, phone, preferred date, clicks "Next":**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2025-07-15"
}
```
> Note: Do NOT push name/phone into dataLayer — PII must not flow into GA4.

**Step 3 — User clicks "Confirm Booking" and booking is submitted:**
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2025-07-15",
  "booking_id": "ON-20250715-0042"
}
```

---

### GTM Setup for Each Step

| Step | GTM Trigger | GTM Tag |
|---|---|---|
| Step 1 | Custom Event: `booking_step_complete` → filter `step_number` equals `1` | GA4 Event tag: `booking_step_complete` with params |
| Step 2 | Custom Event: `booking_step_complete` → filter `step_number` equals `2` | GA4 Event tag: `booking_step_complete` with params |
| Step 3 | Custom Event: `booking_step_complete` → filter `step_number` equals `3` | GA4 Event tag: `booking_step_complete` + separate Conversion tag |

### surfacing Drop-off in GA4 Funnel Exploration

1. Go to **Explore → Funnel Exploration**
2. Add steps using the `booking_step_complete` event, filtered by `step_number` = 1, 2, 3
3. Enable **Open Funnel** (users can enter at any step)
4. Breakdown dimension: `clinic_location` — shows which clinic funnel leaks most
5. Set date comparison to spot week-over-week drop-off changes

---

## Google Ads Conversion Action

**Recommended conversion: `booking_step_complete` where `step_number = 3` (i.e., booking confirmed)**

**Why this one over the others:**

- It represents a **completed intent action** — the user has submitted all three steps. This is the closest proxy to an actual appointment that doesn't require backend confirmation.
- `call_now_click` is noisier — many clicks won't result in a conversation, and call duration data isn't available without additional setup.
- `patient_guide_download` is a soft conversion and attracts informational-stage users, not bottom-of-funnel bookers.
- Importing `booking_confirmed` (step 3) into Google Ads lets Smart Bidding optimise toward users who complete the full funnel — not just visit or click.

**Import method:** In GTM, add a Google Ads Conversion Tracking tag firing on the step-3 Custom Event trigger, using the conversion ID and label from the Google Ads account.
