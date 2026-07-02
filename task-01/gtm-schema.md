# Task 01 — GTM Event Schema
## OrthoNow · Namoza Digital Growth Engagement

---

## Section 1 — Complete Event Schema Table

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `clinic_page_view` | Page View trigger (URL contains `/clinic/`) | `clinic_name`, `clinic_city`, `page_path` | Pages & Screens report; Audience — "Visited Specific Clinic Page" |
| `call_now_click` | Click Listener (GTM native — element matches `a[href^="tel:"]`) | `click_text`, `page_location`, `clinic_name` | Events report; Audience — "High Intent: Called Clinic" |
| `whatsapp_widget_click` | Click Listener (element matches `.whatsapp-widget` or `a[href*="wa.me"]`) | `page_location`, `click_url`, `device_category` | Events report; Audience — "WhatsApp Intent" for remarketing |
| `patient_guide_form_submit` | Custom dataLayer push (fires on gated form success) | `form_name`, `lead_source_page`, `fields_completed` | Events report; Audience — "PDF Lead — Remarketing Pool" |
| `patient_guide_download` | Custom dataLayer push (fires after gated form success, on PDF delivery) | `pdf_name`, `page_location`, `lead_source_page` | Events report; tracks actual download vs form completion |
| `blog_scroll_depth` | Scroll Depth trigger (GTM native — thresholds: 25%, 50%, 75%, 90%) | `percent_scrolled`, `article_title`, `page_path` | Engagement report — Pages & Screens sorted by scroll depth |
| `booking_step_complete` | Custom dataLayer push (one push per step — 3 total) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (see Section 2 below) |
| `booking_confirmed` | Custom dataLayer push (fires on confirmed server response — Step 3 only) | `clinic_location`, `specialty`, `appointment_date` | Conversions; imported into Google Ads (see Section 3) |
| `consultation_form_submitted` | Custom dataLayer push (landing page form submit) | `form_name`, `lead_source`, `clinic_preference`, `submission_timestamp` | Conversions; feeds "Landing Page Lead" audience |

---

### Important Notes on Schema Decisions

**Why no PII in parameters:**
Name and phone number are never sent as GA4 event parameters.
Google's measurement terms prohibit sending personally identifiable
information to GA4. Parameters only confirm that data was entered,
never what the data was.

**Why `patient_guide_form_submit` and `patient_guide_download`
are separate events:**
The PDF is gated — a user can complete the form but still not
receive the PDF if there is a delivery failure. Tracking both
events separately lets us detect drop-off between form completion
and actual download delivery.

**Why `call_now_click` uses a native Click Listener:**
GTM can natively detect clicks on `tel:` links without any custom
dataLayer push from the front-end developer. This is one of the
few events that does not require developer involvement to instrument.

---

## Section 2 — Booking Form Funnel Drop-Off Tracking

### How the 3-Step Funnel Works

GTM cannot natively detect movement between steps of a
multi-step JavaScript form. The form does not trigger a page
reload between steps — it is a single-page interaction. This
means GTM has no DOM event to listen to unless the front-end
developer explicitly pushes a custom event into `window.dataLayer`
at the exact moment each step completes.

**Who writes the dataLayer push:**
The front-end developer who builds or maintains the booking form
writes the push. The Namoza developer's job is to brief them
with the exact event name, exact parameter keys, exact values,
and the exact moment in the code where the push must fire
(after successful client-side validation, before rendering the
next step — never on button click alone, never before validation).

---

### GTM Setup for Each Step

**One Custom Event Trigger** listening for:
`Event Name equals booking_step_complete`

**One GA4 Event Tag** firing on that trigger, mapping these
parameters as GA4 event parameters (must be registered as
Custom Dimensions in GA4 Admin → Custom Definitions):
- `step_number`
- `step_name`
- `clinic_location`
- `specialty`

---

### dataLayer Push — Step 1
**When it fires:** After user selects clinic location AND
specialty, on click of "Next" button, after client-side
validation passes.

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Pain"
}
```

---

### dataLayer Push — Step 2
**When it fires:** After user enters name, phone, and preferred
date, on click of "Next" button, after client-side validation
passes.

**Note:** Name and phone are NOT included in this push.
Sending PII to GA4 violates Google's measurement terms.
We only confirm that the fields were completed.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Pain",
  "preferred_date": "2025-02-15"
}
```

---

### dataLayer Push — Step 3
**When it fires:** On confirmed SUCCESS response from the
server — NOT on button click. If this fired on button click,
failed bookings would be counted as conversions, silently
poisoning funnel data and Google Ads optimisation.

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Pain",
  "appointment_date": "2025-02-15"
}
```

---

### How to Surface Drop-Off in GA4 Funnel Exploration

1. Go to GA4 → Explore → Funnel Exploration
2. Set funnel type to **Open Funnel**
3. Add 3 steps:
   - Step 1: `event_name equals booking_step_complete`
     AND `step_number equals 1`
   - Step 2: `event_name equals booking_step_complete`
     AND `step_number equals 2`
   - Step 3: `event_name equals booking_step_complete`
     AND `step_number equals 3`
4. GA4 will show the percentage of users who dropped off
   between each step
5. Add `clinic_location` as a breakdown dimension to see
   which clinics have the worst drop-off at which step

This tells the performance marketing team exactly where
in the booking flow patients are abandoning — and which
clinic pages to prioritise fixing first.

---

## Section 3 — Google Ads Conversion Action

**Chosen event: `booking_confirmed`**

**Why this event over the others:**

`booking_confirmed` fires only when the server returns a
successful booking response at Step 3. It is the only event
in this schema that represents a completed, verified business
outcome rather than an intent signal.

Importing `call_now_click` would tell Google Ads to optimise
toward people who click a phone number — including accidental
clicks and calls that do not result in bookings. The bidding
algorithm would learn the wrong behaviour.

Importing `consultation_form_submitted` (landing page) would
optimise toward form fills — better than a click, but still
not a confirmed appointment. A user could submit the form
and then not answer the follow-up call.

`booking_confirmed` is the closest proxy to actual patient
acquisition available in this schema. It represents a user
who selected a clinic, entered their details, and received
server-side confirmation — the highest-quality signal we
can send to Google Ads' Smart Bidding algorithm.

**How to import:**
GA4 → Admin → Events → Mark `booking_confirmed` as a
conversion → Google Ads → Tools → Conversions → Import
from Google Analytics.