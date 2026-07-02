# Task 03 — Integration Design
## OrthoNow Consultation Form → HubSpot + WhatsApp + Google Ads

---

## End-to-End Architecture

### The Flow in Order
---

### What Sits at Each Step

**Step 1 — Form Submission**
The landing page form submits via a JavaScript `fetch()` POST
request to a Netlify Serverless Function endpoint. No API keys
are exposed in the browser. The payload contains name, phone,
and clinic preference from the form fields.

**Step 2 — Serverless Function (Netlify Function)**
A Node.js serverless function receives the POST request.
I chose a serverless function over Zapier or Make for three
reasons: API keys stay server-side and are never exposed in
the browser; the search-before-create deduplication logic
for phone numbers requires conditional API calls that
no-code tools handle poorly; and execution is synchronous
and auditable with full error logging.

**Step 3 — HubSpot Integration via API (not native embed)**
I chose the HubSpot Contacts API directly over HubSpot's
native form embed or Zapier for a specific reason: phone
number deduplication.

HubSpot's default deduplication is on email address — not
phone number. Since this form collects name and phone only
(no email), a native embed or Zapier flow would create
duplicate contacts every time the same patient submits twice.

The serverless function first calls HubSpot's Search API:
`POST /crm/v3/objects/contacts/search` with a filter on
the `phone` property. If a match is found, it calls
`PATCH /crm/v3/objects/contacts/{id}` to update the
existing contact. If no match is found, it calls
`POST /crm/v3/objects/contacts` to create a new one.

In both cases, these properties are set:
- `firstname`: from form
- `phone`: from form
- `hs_lead_status`: New Enquiry
- `lead_source`: Google Ads - Consultation Landing Page
- `clinic_preference`: from form

**Step 4 — Karix WhatsApp Message**
Immediately after the HubSpot call resolves, the serverless
function makes an async POST request to the Karix API with
the patient's phone number and a pre-approved WhatsApp
message template confirming their enquiry. This call is
made asynchronously — a Karix failure does not block the
HubSpot write or the success response to the browser.

**Step 5 — Google Ads Conversion**
The serverless function returns a `200 OK` response to the
browser. The landing page JavaScript receives this success
response and fires the GTM dataLayer push:
`window.dataLayer.push({event: 'consultation_form_submitted'})`.
GTM picks this up and fires the Google Ads conversion tag.
The conversion fires client-side, after confirmed server
success — not on button click.

---

## Biggest Failure Point and Fallback

**The single biggest failure point is the Karix WhatsApp
API call.**

HubSpot is a mature platform with 99.99% uptime SLA.
Google Ads conversion tracking is client-side and
independent. But Karix, as a third-party messaging
gateway, introduces the most failure risk — API timeouts,
rate limits, template approval delays, and WhatsApp
Business API restrictions on new numbers.

**Fallback architecture:**
The serverless function wraps the Karix call in a
try-catch block. If the Karix call fails or times out
(threshold: 5 seconds), the failure is logged to a
monitoring service (such as Sentry or Google Cloud
Logging) with the patient's phone number and timestamp.
A retry queue (using a simple scheduled function that
runs every 60 seconds) picks up failed WhatsApp jobs
and retries them up to 3 times before flagging for
manual follow-up. This ensures HubSpot contact creation
and Google Ads conversion tracking are never blocked
by a WhatsApp failure.

---

## WhatsApp 2-Minute SLA — What Breaks It and How to Monitor

**What could break the 2-minute SLA:**

1. Serverless function cold start delay (first invocation
   after idle period can add 2-3 seconds — negligible)
2. Karix API response latency under load
3. WhatsApp message queue congestion on Meta's side
4. Karix rate limiting if multiple submissions arrive
   simultaneously from a paid campaign burst

**How to monitor it:**
Every WhatsApp send attempt is logged with three
timestamps: `form_submitted_at`, `karix_called_at`,
and `karix_confirmed_at`. These logs go to Google
Cloud Logging or Sentry. A simple alert fires if
the delta between `form_submitted_at` and
`karix_confirmed_at` exceeds 90 seconds — giving
the team 30 seconds of buffer before the SLA is
actually breached. Weekly SLA reports are pulled
from these logs to track p95 delivery latency
over time.

---

## Phone Number Deduplication — The Critical Edge Case

If two patients submit with the same phone number
but different names, the HubSpot Search API call
returns the existing contact. The function updates
that contact's properties using PATCH — it does
NOT create a second contact. The `firstname` field
is updated to the most recent submission.

This is the correct behaviour for Indian healthcare
lead gen where the same patient may submit multiple
times across different campaigns, and where email
is not collected. A setup that does not handle this
— such as a native HubSpot form embed or a basic
Zapier flow — would silently create duplicate
contacts, inflating lead counts and corrupting
CRM data.