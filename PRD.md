# PRD: HealthPal — Personal Health Tracker & Companion

**Status:** v1.1 built and delivered (`HealthPal.html`)
**Owner:** Mika
**Last updated:** July 8, 2026

## 1. Summary

HealthPal is a friendly, self-contained health-tracking app built for Mika and for anyone managing a health condition that requires watching blood test trends, taking medication on schedule, and being mindful of food choices. The initial trigger was Mika's own situation: blood sugar that's elevated but not yet pre-diabetic, with a desire to catch it early through better food awareness.

The app runs as a single HTML file with no backend or account system. All data is stored locally in the browser (localStorage) on the device where it's opened.

## 2. Problem & Motivation

People managing an emerging or chronic health condition typically juggle three disconnected habits: remembering to take medication at the right time, noticing patterns in lab results over months, and judging in the moment whether a food choice fits their condition. Most people don't have a lightweight, judgment-free tool that ties these together and talks to them like a supportive friend rather than a clinical dashboard.

## 3. Goals

- Let users log blood test markers over time and see trends at a glance, without being locked into a fixed list of tests.
- Give users a place to track their medications and get genuinely delivered reminders, not just an in-app list they have to remember to check.
- Let users log what they eat (by description, optionally with a reference photo) and get an immediate, friendly, condition-aware reaction — good choice, fine in moderation, or worth watching.
- Support the emotional and lifestyle side of health: mood/energy check-ins, water and exercise, and menstrual cycle tracking.
- Make it easy to walk into a doctor's appointment with a clean summary instead of a stack of raw numbers.
- Keep the tone warm and encouraging, never guilt-inducing — this app should feel like a supportive companion, not another chore.
- Give users a calendar view of their health activity, and a real way to get medication times and appointments into Apple Calendar (and from there, iCloud).
- Let users keep actual documents (lab PDFs, doctor's notes, scans) attached to their record and download them again whenever needed.

## 4. Non-Goals (v1)

- **No real AI image recognition of food photos.** True photo-based nutrition analysis requires a backend and an AI vision API. v1 lets users attach a photo for their own visual record, but the actual recommendation is driven by a typed description, matched against condition-specific keyword rules.
- **No native push notifications from the app itself.** A static local file cannot send OS-level notifications when it isn't open. Medication reminders that need to reach the user reliably are handled separately, as recurring scheduled messages set up through Claude/Cowork.
- **No automatic multi-device sync.** Because data lives in browser localStorage, a phone and a computer each keep their own separate copy. Keeping them aligned is a manual export/import step (see §9).
- **No clinical claims.** HealthPal offers general, rule-based food guidance tied to self-reported conditions. It is not a diagnostic tool and doesn't replace medical advice.
- **No Apple Health integration.** Apple Health (HealthKit) has no public web API — a browser-based app cannot read from or write to it under any circumstances, regardless of backend investment. This isn't a v1 limitation to revisit later; it's a platform-level restriction. iCloud Calendar is the supported integration point instead (see §8).
- **No live two-way calendar sync.** Apple's calendar servers don't accept direct connections from browser JavaScript (CORS-blocked), so calendar integration is one-way, user-initiated: HealthPal generates a standard calendar file (.ics) that the user opens once to add events into Apple Calendar, which then syncs to iCloud on its own.

## 5. Target Users

1. **Primary:** Mika — borderline high blood sugar, not pre-diabetic, wants to catch early warning signs through diet awareness and track lab trends over time.
2. **Secondary:** Anyone with a condition requiring scheduled medication and/or dietary awareness (e.g., high blood pressure, high cholesterol), who wants one friendly place to track it all instead of separate apps for meds, labs, and food.

## 6. Feature Set

| Area | What it does |
|---|---|
| Onboarding | First-run setup: name, health focus areas (checkboxes for blood sugar, blood pressure, cholesterol), a free-text box for any other health conditions in the user's own words, and general notes. The checkbox conditions drive the food recommendation rules; free-text conditions are stored and shown in the profile/doctor report, and referenced in food tips as noted context, since they aren't tied to a keyword rule set. |
| Blood Markers | User defines any custom marker (name, unit, healthy range) rather than a fixed list, logs readings with dates and notes, and sees a trend chart with color-coded status (good / borderline / out of range) per marker. |
| Medications | Add medications with dosage and one or more daily times. In-app list shows what's due today. A banner explains that real reminders are delivered through a separate scheduled-message setup (see §8), since the app can't notify on its own when closed. While the tab is open, it also does a lightweight in-app check every 30 seconds and surfaces a toast if a dose time is hit. |
| Food Log & Recommendations | User types a description of what they ate (meal type, date/time, optional reference photo). A rule-based engine checks the description against keyword lists tied to the user's selected condition(s) and returns a verdict — great choice, okay in moderation, worth watching, or just "logged" if there's not enough detail — plus a short friendly explanation. Full history is kept and browsable. |
| Daily Check-in | Mood (5-point emoji scale), energy (1–5 slider), and free-text symptoms/notes, logged once per day and viewable as history. |
| Water & Activity | Quick-add buttons for water intake against a daily goal (default 8 glasses, editable), and a simple exercise log (type + duration) with a rolling weekly total. |
| Cycle Tracking | Log period start/end dates and symptoms; once at least two cycles are logged, the app estimates average cycle length and predicts the next start date. |
| Calendar | Month-view calendar with dots marking medication days, appointments, logged blood readings, and food entries; clicking a day shows the detail. An Appointments manager lets users add one-off events (title, date, time, notes). Both appointments and the recurring medication schedule can be exported as `.ics` files that import straight into Apple Calendar, which then syncs to iCloud on its own. |
| Documents | Upload any file (lab PDFs, doctor's notes, scans, images) with a title, category, date, and optional notes. Files are listed with a one-click Download link and can be deleted. Documents logged in a date range also show up in the Doctor Report. |
| Doctor Visit Report | Generates a printable summary (marker trends, active medications, nutrition snapshot, wellbeing summary, attached documents) for a chosen date range, with one-click print/save-as-PDF and a raw-data CSV export. |
| Settings | Edit profile and condition tags, set water goal, export/import a full JSON backup, and clear all data. |

## 7. Data Model (stored as one JSON object in localStorage)

- `profile` — name, selected condition tags, free-text "other conditions," free-text notes, water goal, onboarding status.
- `markers` — user-defined blood markers (name, unit, healthy low/high).
- `readings` — individual marker values tied to a date, with optional notes.
- `meds` — medication name, dosage, list of times, notes, active flag.
- `foodLogs` — date/time, meal type, description, optional photo (stored inline), computed verdict and tip, matched keyword tags.
- `checkins` — date, mood, energy (1–5), symptom notes.
- `exercise` — date, activity type, duration in minutes.
- `water` — date, glasses logged.
- `periods` — cycle start/end dates and symptoms.
- `appointments` — title, date, time, notes; source data for calendar dots and `.ics` export.
- `documents` — title, category, date, notes, original file name/type, file contents (stored inline as base64).

## 8. Reminders and Calendar Integration

Because HealthPal is a static file with no server, it cannot send a notification unless the browser tab is open at that exact moment, and it cannot connect directly to Apple's calendar servers (browser JavaScript is blocked by CORS from doing that). Two separate mechanisms cover this gap:

**Medication reminders that need to reach the user reliably:**
1. The user tells Claude (in chat) the medication name and time(s), e.g. "remind me to take Metformin at 8am and 8pm every day."
2. Claude creates a recurring scheduled task that fires a message at those times regardless of whether HealthPal is open.
3. The medication is also entered into HealthPal itself, so it shows up in the in-app list, dashboard, calendar, and doctor report.

*Status: not yet set up — pending the user providing actual medication names and times.*

**Getting medications and appointments into Apple Calendar / iCloud:**
1. In the Calendar tab (or the Medications tab), the user clicks "Export schedule" or "Export to Calendar."
2. HealthPal generates a standard `.ics` calendar file — a daily-repeating event per medication time, or a single event per appointment.
3. Opening the downloaded file adds the event(s) into Apple Calendar, which syncs to iCloud automatically from there.

This is one-way and user-initiated by design — there's no live sync, so the user re-exports if medication times or appointments change. Apple Health (HealthKit) is not part of this integration; see §4 for why that's not possible from a web app.

## 9. Multi-Device Use

Mika confirmed she plans to use HealthPal on more than one device (phone and computer). Since localStorage doesn't sync across browsers or devices, the Settings tab includes an explicit workflow: export a JSON backup from the device with the most current data, then import it on the other device. This is manual by design in v1 — there's no account system or cloud sync — and the app surfaces this clearly rather than letting the user discover a sync gap on their own.

## 10. Design & Tone Guidelines

- **Visual style:** dark theme by request — near-black backgrounds (`#101012`), dark surfaces for cards/inputs, and muted/desaturated accent colors (dusty teal, muted coral, soft gold) rather than bright, saturated tones. Rounded cards, Nunito typeface. Printed reports force a white background with dark text regardless of the in-app theme, since printing dark backgrounds wastes ink and hurts legibility.
- **Copy tone:** friendly and encouraging, but *subdued* rather than heavily emoji-driven. Emoji are reserved for functional wayfinding (tab icons, mood picker) rather than sprinkled through headings, toasts, and button labels — this was tuned down from the original build based on direct feedback.
- **No guilt-tripping:** food feedback is framed as gentle awareness ("might want to watch this one"), never as a scolding or restriction.

## 11. Constraints & Assumptions

- Single-user, single-browser-profile use per device; no login or multi-user support.
- Photo and document attachments are stored as base64 inside localStorage, which has practical size limits (a few MB depending on browser). Kept as-is per user preference in v1; could be revisited if storage limits become a problem, especially now that arbitrary documents (PDFs, scans) can be uploaded alongside food photos.
- Recommendation engine is keyword/rule-based, not a medical algorithm — it's meant to nudge awareness, not replace a doctor or dietitian.
- Calendar export is one-way (.ics generation), not a live sync; Apple Health has no integration path at all (see §4).

## 12. Open Items / Next Steps

- Set up the actual Cowork scheduled-task reminders once Mika provides medication names and times.
- Monitor whether localStorage size becomes an issue with photo/document attachments; if so, add automatic image compression and/or a file-size warning on upload.
- Consider a future version with real AI-based food photo analysis if/when a backend or API key is introduced.
- Revisit multi-device sync if manual export/import proves too friction-heavy in practice.

## 13. Version History

- **v1 (July 8, 2026):** Initial build — all features in §6 (original set) implemented and delivered as `HealthPal.html`. Follow-up revision pass: toned down copy/emoji, clarified multi-device backup workflow, fixed an onboarding-lock edge case in Settings.
- **v1.1 (July 8, 2026):** Added a free-text "other health conditions" field alongside the fixed checkboxes; switched the visual theme from bright to dark/muted; added a Calendar tab (month view + appointments + .ics export for Apple Calendar/iCloud); added a Documents tab for uploading and downloading arbitrary files (lab results, doctor's notes, etc.); documented why Apple Health cannot be integrated from a web app.
