# Task 01 — GTM Event Schema: OrthoNow

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience it feeds |
|---|---|---|---|
| `clinic_page_view` | Page View — Trigger fires on Page Path matches RegEx `/clinic/.*` | `clinic_name`, `city`, `page_path` | Engagement → Pages and Screens (filtered by clinic); powers a "Visited Clinic X page" remarketing audience |
| `call_now_click` | Click — Just Links / All Elements, Click Classes contains `call-now-btn` | `page_location`, `clinic_name` (blank if homepage), `phone_number_clicked` | Custom conversion event "Calls initiated"; feeds a "Call intent" audience for Search remarketing |
| `whatsapp_chat_click` | Click — Trigger on Click URL contains `wa.me` | `page_location`, `page_type` (home/clinic/landing), `device_category` | Custom conversion "WhatsApp intent"; feeds Explore → Path Exploration to see where chat is opened from |
| `patient_guide_form_submit` | Form Submission trigger on the gated PDF-download form (or Custom Event `guide_download_submitted` pushed on success) | `form_name`, `lead_source`, `page_location` | Conversion event; feeds a "Downloaded guide, not booked" nurture audience |
| `blog_scroll_depth` | Scroll Depth trigger — Vertical, thresholds 25/50/75/90% | `percent_scrolled`, `page_path`, `article_title` | Engagement → Pages and Screens (scroll as secondary dim); feeds "Engaged blog readers" audience for content remarketing |
| `booking_step_view` | Custom Event trigger listening for dataLayer event `booking_step_view` | `step_number`, `step_name`, `clinic_location` | Funnel Exploration (step entry) |
| `booking_step_complete` | Custom Event trigger listening for dataLayer event `booking_step_complete` (fires 3x — once per step) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (step completion / drop-off) |
| `booking_confirmed` | Custom Event trigger listening for dataLayer event `booking_confirmed` | `clinic_location`, `specialty`, `preferred_date`, `booking_id` | **Primary conversion.** Imported to Google Ads. Feeds "Booked patients" audience for exclusion from prospecting campaigns |

---

## 2. Booking Funnel — Step-Level Drop-off Tracking

**Important context (this is the actual crux of Task 1):**
GTM cannot natively detect progress inside a JS-driven multi-step form. There's no page reload and no new URL between steps 1→2→3, so GTM has nothing to listen to on its own — no Page View, no Click on a "real" link, no Form Submission (that only fires on the *final* submit). The only way to track intermediate steps is for the **front-end developer** to explicitly push a `dataLayer.push()` at each step transition in the form's JS (e.g. inside the "Next" button's onClick handler, after validation passes). GTM's job is only to *listen* for that Custom Event trigger — it does not generate it.

**How I'd brief the front-end dev:**
"Every time a user successfully completes a step and clicks Next/Confirm, push a `dataLayer` object with `event: 'booking_step_complete'` right before you render the next step — not on step load, on step *completion*. Also push a `booking_step_view` when a step first renders, so we can see if people are dropping off before they even start filling a step, vs. abandoning mid-fill. I'll give you the exact key names and I'll add the GTM trigger + tag on my side once you confirm it's firing (I'll verify in GTM Preview mode)."

### Trigger set-up in GTM
- One **Custom Event** trigger per event name (`booking_step_view`, `booking_step_complete`) — GTM Custom Event triggers match on `event` name in the dataLayer push.
- One GA4 Event tag per trigger, mapping the dataLayer variables (`step_number`, `step_name`, `clinic_location`, `specialty`) to GA4 event parameters via Data Layer Variables in GTM.

### Actual dataLayer JSON (not pseudocode)

**Step 1 — Location + Specialty selected**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "OrthoNow Koramangala",
  "specialty": "Knee & Joint"
}
```

**Step 2 — Contact details entered**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "OrthoNow Koramangala",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-10"
}
```

**Step 3 — Booking confirmed (final)**
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "OrthoNow Koramangala",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-10",
  "booking_id": "ON-2026-04821"
}
```

### Surfacing drop-off in GA4 Funnel Exploration
1. Create a new **Funnel Exploration**.
2. Add 3 open-funnel steps, in order:
   - Step 1: `booking_step_complete` where `step_number = 1`
   - Step 2: `booking_step_complete` where `step_number = 2`
   - Step 3: `booking_confirmed`
3. Turn **"Open funnel"** on for step 1 (so we don't force entry only from the very first touchpoint) and leave it closed between steps 2→3 to measure true sequential drop-off.
4. Use the built-in **"Elapsed time"** breakdown to see how long users take between step 2 and step 3 — this is usually where drop-off spikes (phone number field, OTP, etc.), and it's the number I'd show the marketing team first.
5. Break down by `clinic_location` as a secondary dimension to see if drop-off is worse for specific clinics (could indicate a slot-availability or copy issue on that clinic's option).

---

## 3. Conversion Action to Import into Google Ads

**Import: `booking_confirmed`**

Not `call_now_click` and not `whatsapp_chat_click`, even though both look like strong intent signals. Reasoning:

- **Call and WhatsApp clicks are directional, not qualified.** A call click just means someone tapped a button — it doesn't confirm they reached a person, spoke to reception, or actually booked. Importing this as a conversion would feed Google Ads' automated bidding (Target CPA / Maximize Conversions) a noisy signal, and the algorithm would start optimizing toward people who *click to call* rather than people who *actually book*.
- **`booking_confirmed` is the one event that's unambiguous and carries full attribution data** — clinic, specialty, and a real `booking_id` I can reconcile against the CRM later to confirm it wasn't a duplicate or test booking.
- For a healthcare vertical specifically, Google Ads' Smart Bidding is only as good as the conversion it's chasing — feeding it a true bottom-of-funnel action (confirmed appointment) rather than a micro-conversion (button click) is what actually moves CPA down over time, not just conversion *volume* up.

`call_now_click` can still be tracked as a **secondary/observation conversion** (not counted toward primary bidding) so the marketing team has visibility into it without it corrupting the bid strategy.
