# Namoza Developer Assignment
## Developer — Position 1 (Client Web + Martech)
**Candidate:** Abhishek Kumar Shrivastav
**Submission Date:** 2 July 2026

---

## Overview

This repository contains all three deliverables for the
Namoza Developer Assignment. The scenario is a full digital
growth engagement for OrthoNow — a chain of 9 orthopaedic
clinics across Bengaluru, Hyderabad, and Chennai.

---

## Repository Structure
---

## Task 01 — GTM Event Schema

**File:** `task-01/gtm-schema.md`

### What's Inside
- Complete event schema table covering all 9 key
  interactions on the OrthoNow website
- Trigger type, key parameters, and GA4 report mapping
  for every event
- Full dataLayer JSON for all 3 steps of the booking form
- GA4 Funnel Exploration setup instructions for step-level
  drop-off analysis
- Google Ads conversion action selection with justification

### Key Decision
`booking_confirmed` was chosen as the Google Ads conversion
import over `call_now_click` or `consultation_form_submitted`
because it is the only event representing a verified,
server-confirmed business outcome rather than an intent
signal. Optimising toward an earlier funnel event would
teach Google Ads' Smart Bidding the wrong behaviour.

### Important Technical Note
The 3-step booking form requires custom dataLayer pushes
written by the front-end developer — GTM cannot natively
detect movement between steps of a JavaScript-driven
multi-step form. The schema includes exact briefing
language for the dev team.

---

## Task 02 — Landing Page Build

**File:** `task-02/index.html`
**Live URL:** https://glistening-cranachan-abe53f.netlify.app
**PageSpeed Mobile Score:** 100/100

### What's Built
- Single self-contained HTML file — no frameworks,
  no external CSS, no Google Fonts
- Target audience: working professionals aged 28–50
  in Bengaluru experiencing knee or back pain
- 2-field form (Name + Phone) with client-side validation
- GTM dataLayer push fires on successful form submit
  (not on page load, not on button click alone)
- Thank-you state appears without page reload
- Mobile-first responsive design

### Conversion Architecture Decisions

**Headline:** Speaks directly to the patient's pain point,
not the clinic's credentials. Working professionals
recognise their situation immediately.

**Form placement:** Above the fold on both mobile and
desktop. The target audience has already decided they
want to book — removing friction is the priority.

**2-field form only:** Name and phone. Every additional
field reduces conversion rate. Email is not collected
because it is not needed for the follow-up workflow
(OrthoNow's team calls to confirm).

**CTA copy:** "Book My Free Consultation →" instead of
"Submit". The word "free" reduces financial hesitation.
The arrow creates forward momentum.

**Trust signals:** Placed immediately below the form
(mobile) and inline with the hero (desktop) — not
buried at the bottom where hesitant users never scroll.

### GTM dataLayer Push
```javascript
window.dataLayer.push({
  'event': 'consultation_form_submitted',
  'form_name': 'orthonow_consultation_lp',
  'lead_source': 'google_ads_consultation_lp',
  'clinic_preference': 'bengaluru',
  'submission_timestamp': new Date().toISOString()
});
```
This push fires after client-side validation passes
and before the thank-you state is shown. Visible in
browser console (DevTools → Console) on form submit.

### PageSpeed Score
![PageSpeed Mobile Score](task-02/pagespeed-screenshot.png)

---

## Task 03 — Integration Design

**File:** `task-03/integration-writeup.md`

### Architecture Summary
### Why Serverless Function Over Zapier or Make
HubSpot's native deduplication is on email — not phone.
Since this form collects no email, a Zapier or native
embed flow would create duplicate contacts on every
repeat submission. The serverless function implements
a search-before-create pattern that Zapier cannot
handle cleanly.

### Critical Edge Case Handled
If two patients submit with the same phone number,
the HubSpot Search API returns the existing contact
and the function updates it via PATCH. No duplicate
contact is created. This is the correct behaviour for
Indian healthcare lead gen where email is not collected.

---

## How to Run Task 02 Locally

```bash
# No build step needed — open directly in browser
# Option 1: double click index.html in file explorer

# Option 2: use VS Code Live Server extension
# Right click index.html → Open with Live Server

# Option 3: simple Python server
cd task-02
python -m http.server 8000
# open http://localhost:8000 in browser
```

---

## Testing the dataLayer Push

1. Open `task-02/index.html` in Chrome
2. Press `F12` to open DevTools
3. Click the **Console** tab
4. Fill in Name and Phone on the page
5. Click **Book My Free Consultation**
6. You will see this logged in the console:

```javascript
✅ dataLayer push fired: [
  {
    event: 'consultation_form_submitted',
    form_name: 'orthonow_consultation_lp',
    lead_source: 'google_ads_consultation_lp',
    clinic_preference: 'bengaluru',
    submission_timestamp: '2025-01-28T10:30:00.000Z'
  }
]
```

---

## Contact

**Name:** Abhishek Kumar Shrivastav
**Email:** 21mayabhishek@gmail.com
**Phone:** +919801880205
**Loom Walkthrough:** [paste Loom link here after recording]