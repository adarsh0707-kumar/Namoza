# OrthoNow — Namoza Developer Assignment

**Candidate:** Adarsh Kumar  
**Role:** Developer — Position 1 (Client Web + Martech)  
**Client:** OrthoNow, 9 orthopaedic clinics across Bengaluru, Hyderabad, and Chennai

---

## Repository Structure

```
├── task1-gtm-schema.md          # GTM event schema table + booking form dataLayer JSON
├── task2-landing-page.html      # Single-file landing page (HTML/CSS/Vanilla JS)
├── task3-integration-design.md  # HubSpot + WhatsApp + Google Ads integration writeup
└── README.md                    # This file
```

---

## Task 01 — GTM Event Schema

See [`task1-gtm-schema.md`](./task1-gtm-schema.md)

**Covers:**
- Full event schema for all 6 interaction types (booking form, call buttons, WhatsApp, PDF download, clinic page views, blog scroll depth)
- Exact dataLayer JSON for all 3 booking form steps
- Explanation of why GTM requires custom dataLayer pushes for multi-step forms
- Recommended Google Ads conversion action with justification

---

## Task 02 — Landing Page

See [`task2-landing-page.html`](./task2-landing-page.html)

**Live demo:** Open the HTML file directly in any browser — no server required.

**Key decisions:**
- Audience-specific copy for Bengaluru working professionals (28–50) with knee/back pain — not generic healthcare copy
- 2 required fields (name + phone) + 1 optional (clinic preference) — keeps friction minimal
- GTM `consultation_form_submitted` dataLayer push fires on valid submit only, not on page load
- Thank-you state shown without page reload
- Designed for mobile-first; PageSpeed target 90+

**To verify the dataLayer push (for Loom):**
1. Open the HTML file in Chrome
2. Open DevTools → Console
3. Submit the form with valid inputs
4. You will see the log: `[OrthoNow GTM] dataLayer push fired: { event: 'consultation_form_submitted', ... }`

---

## Task 03 — Integration Design

See [`task3-integration-design.md`](./task3-integration-design.md)

**Architecture:** Landing page → Serverless function → HubSpot Contacts API (with phone-first deduplication) + Karix WhatsApp API → Google Ads conversion via GTM

**Key flag:** HubSpot deduplicates on email by default. Since this form collects phone only, a search-before-write approach is used, and phone is configured as a unique property in HubSpot settings.

---
