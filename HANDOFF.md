# iLive Family Connect — Developer Handoff v0.1

**Owner:** IHLD MedTech Pvt. Ltd. · **Prototype:** `iLive-Family-Connect-Prototype.html` (open in any browser; every button works)

The prototype sets the look and behaviour. Match it pixel-for-pixel at a 390 pt width, and scale it down to 360 dp on Android.

---

## 1. Screens & navigation

| # | Screen | Route | Entered from | Key actions |
|---|---|---|---|---|
| 01 | Home (Guardian dashboard) | `/home/:patientId` | Launch, push | Vital card → 02 · Summary card → 03 · "View notes" → 04 · Call buttons → 05 · Menu → drawer |
| 02 | Vitals & Trends | `/vitals/:patientId/:metric` | Any vital card (opens with that metric selected) | Switch metric · 24H / 7D / 30D · min/avg/max · recent readings |
| 03 | Daily Health Summary | `/summary/:patientId/:date` | Premium summary card on Home, 9:00 AM push | Today in brief · today's vitals · medical-team review seal · read receipts · Mark as read · PDF · Talk to the medical team |
| 04 | Care Team Notes | `/notes/:patientId?role=` | "View notes" row (opens that role's tab) | Tabs: Doctor / Physio / Diet / Care team · earlier notes · request call back |
| 05 | Call sheet | `/call?to=team|parent|<familyMemberId>` | Header phone icon, bottom buttons, notes, summary | Choose recipient · Voice / Video / Call me back · in-call state |

On Home, the bottom action bar is **sticky**: it stays above the home indicator while the content scrolls under it. "Call Parent" opens screen 05 with the monitored patient already selected. "Call iLive Care Team" opens it with the team selected.

**People in the app:** the *patient* is the person being monitored (Prem Kumar Khanna). The *viewer* is the logged-in family member who subscribes (Shalin Khanna, his son). The *family circle* is everyone linked to the patient who gets updates (Shalin and his sister Arun Khanna). All wording adapts to the viewer's relation, e.g. "Hello, Shalin Khanna", "your father's health", "Call Dad". Arun logging in sees the same screens with his own name, and her brothers listed in the circle.

**Family Circle card (Home):** shows every linked member with their relation and last activity, a call button for each sibling, and "+ Invite a family member". An invite needs the patient's consent. The summary card shows which family members have read today's summary (read receipts).

Menu drawer items for v1: Switch family member, Alerts & notifications, Reports & documents, Devices, Consent & privacy, Help & support, Log out.

## 2. Design tokens
See `tokens.json`. In summary:

- **Colours:** navy `#0B1D3A` (header, primary button) · gold `#D9B871` (premium accent, used on navy only) · background `#F2F4F8` · cards `#FFFFFF`, border `#E4E9F1` · blue `#2563EB` · violet `#5B48D6` (summary)
- **Status colours:** alert `#B42318` on `#FDECEC` (the value itself in red `#D92D20`) · watch `#8A4B00` on `#FEF1DC` · normal `#11703A` on `#E3F5EA` · stale `#4B5568` on `#EDF0F5`
- **Fonts:** headings in Plus Jakarta Sans 800 · body in Inter. H1 26/31, patient name 24, card title 17, vital value 30 (letter-spacing −0.6), body 14.5, timestamps 12, section labels 12.5 bold caps with +0.1em spacing
- **Logo:** use the official iLive vector file. The prototype only approximates it.
- **Spacing:** 16 screen margin · 12 card gap · 14 section gap
- **Corner radius:** header bottom 30 · cards 18–20 · icon tiles 13–16 · buttons 18 · chips and pills fully rounded
- **Shadows:** cards `0 6 20 rgba(11,29,58,.06)` · primary button `0 10 22 rgba(11,29,58,.28)`
- **Touch targets:** at least 44×44; main buttons 56 tall

## 3. Components

| Component | Props | Notes |
|---|---|---|
| PatientHeader | name, sex, monitoringDay, programme, tags[], onCall | Navy block with subtle wave graphic. Product name "iLive Family Connect" (21 px, "Family Connect" in gold). Meta line: sex · day of monitoring. Tags wrap to a second line |
| MonitoringStatusCard | state: active, paused or offline | Active shows a green dot. Offline uses the watch colours and shows when the device was last seen. |
| VitalCard | label, value, unit, recordedAt, status, statusLabel, metric | 2-column grid. The value turns red only when status is alert. The chip is hidden when status is none. The whole card is one tap target. The heart icon appears on heart rate only. |
| StatusChip | status, label | Colour is never the only signal: the text always names the state |
| SummaryCard | generatedAt, reviewedBy, unread | A violet dot shows while the summary is unread |
| InteractionRow | role, lastCallAt, hasNotes | Icon tile colour depends on the role |
| CallActionBar | — | Primary navy button (1.35fr) and secondary outline button (1fr), both 56 tall, sticky |

## 3a. Daily Health Summary (signature feature)
- **On Home:** one compact navy card with a gold border, in the same place as before (after the readings, above Last Interactions). It shows a gold review seal, the label "DAILY HEALTH SUMMARY", the title "Today's Health Summary", "Reviewed by iLive Medical Team · Monitored 24/7 · Today, 9:00 AM", a "NEW" badge while unread, and a gold arrow. Tapping the card opens the summary screen.
- **The report screen has only three parts:**
  1. **Today in brief:** 2–3 plain-language sentences about today's vitals, written or approved by the medical team.
  2. **Today's vitals:** all readings with time and status. Tapping a reading opens its trend.
  3. **Reassurance block:** cream and gold, with the seal, "We're keeping a close watch on Dad", and the reviewer and time.
- **Deliberately left out:** care plan, medication schedule and doctor's notes. Notes stay on the Care Team Notes screen.
- **API:** `GET /v1/patients/:id/summaries/:date` → `{ generatedAt, briefText, vitals[], reviewedBy, reviewedAt, readBy[] }`

## 4. Status & freshness rules
- **The app never decides clinical status.** The backend sends `status` and `statusLabel` with each reading. These are computed from the patient's own target ranges, which the iLive Doctor sets. The app only maps each status to a colour.
- **Old readings:** if a reading is older than its metric's `staleAfterHours`, the app shows a grey "N days ago" chip instead of the status chip. Suggested defaults, all changeable from the backend: HR, HRV and steps 2 h · glucose 24 h · BP 72 h · sleep 36 h.
- **Missing readings:** show "—" with "No reading yet". Never show 0.
- **Times:** shown as "hh:mm AM · dd Mon", in IST by default. Guardians can choose another time zone (useful for NRI families).
- **"Synced" label:** the time of the last successful fetch. Home supports pull-to-refresh, and gets live updates over WebSocket or silent push.

## 5. API endpoints

| Method & path | Used by | Returns / notes |
|---|---|---|
| `GET /v1/guardian/patients` | Patient switcher | One viewer can be linked to both parents |
| `GET /v1/patients/:id/family-circle` | Family Circle card | Members, relation, lastActiveAt, summaryReadAt (also included in the dashboard call) |
| `POST /v1/patients/:id/family-circle/invites` | Invite button | body `{name, phone, relation}` → SMS/WhatsApp invite; joins only after patient consent |
| `GET /v1/guardian/patients/:id/dashboard` | Home | Everything Home needs in one call — see `dashboard.sample.json` |
| `GET /v1/patients/:id/vitals/:metric?range=24h|7d|30d` | Vitals | series[], min, avg, max, targetRange |
| `GET /v1/patients/:id/summaries/:date` | Summary | briefText, vitals[], reviewedBy, reviewedAt, readBy[], pdfUrl |
| `POST /v1/summaries/:summaryId/ack` | Mark as read | The care team sees "guardian read at …" |
| `GET /v1/patients/:id/notes?role=` | Notes | Only notes marked `shareWithGuardian = true` |
| `POST /v1/calls` | Call sheet | body `{to: team|parent, mode: voice|video|callback}` → VoIP token or callback ticket |
| `WS /v1/guardian/stream` | Home live updates | Events: `vital.updated`, `alert.raised`, `summary.ready`, `note.added` |

## 6. Behaviour
- **Push notifications:**
  - `alert.raised`: high priority; opens Vitals on that metric.
  - `summary.ready`: sent at 9:00 AM IST.
  - `note.added`.
  - Reminders when a reading is missed.
- **Lock-screen privacy:** push text must not contain health data. Use something generic, e.g. "New update about Dad".
- **Calls:**
  - Parent: native `tel:` or in-app VoIP.
  - Care Team: in-app voice or video, falling back to a normal phone call.
  - "Call me back": creates a ticket for the care desk.
- **The 112 emergency line on the call sheet is required wording.**
- **Loading:** show placeholder (skeleton) cards in the same grid.

## 7. Access, privacy, accessibility
- **Linking a guardian:** the invite comes from the patient app or the care team. The patient's consent is recorded. If consent is withdrawn, access ends immediately.
- **Login:** OTP, then biometric unlock.
- **Screen readers:** every card has a spoken label, e.g. "Heart rate 100 bpm, Elevated. Open trends". Text contrast is at least 4.5:1.
- **Languages:** English first. All text kept in string files so Hindi and Bengali can be added.

## 8. Placeholders to replace
- Trend chart data is sample data generated by the prototype.
- Items in `[brackets]` (clinician names, phone number) come from the API. The "Today in brief" text is sample wording that the medical team will write or approve each day.

## 9. v1 acceptance checklist
- [ ] Home matches the prototype at 390 pt and scales to 360 dp Android
- [ ] All 6 vitals come from the dashboard API, including old and missing readings
- [ ] Every vital card opens Vitals with that metric selected; 24H/7D/30D works
- [ ] The 9:00 AM summary push opens screen 03; Mark as read sends the acknowledgement
- [ ] Notes show only notes shared with the guardian, per role tab
- [ ] Greeting, relation wording and Family Circle come from `viewer` and `familyCircle` in the API
- [ ] Read receipts on the summary update for every family member
- [ ] Both call buttons work on iOS and Android; "Call me back" creates a ticket
- [ ] VoiceOver and TalkBack read every card and button correctly
