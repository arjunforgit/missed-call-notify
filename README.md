# missed-call-notify-macrodroid

> Missed call detection & staff notification automation

## Stack
- **Trigger:** MacroDroid (Android) — 3 macros
- **Automation:** n8n (self-hosted, Docker)
- **Notifications:** Slack (`#missed-call` channel)
- **Incident Management:** PagerDuty
- **Data Storage:** n8n Data Tables

## MacroDroid Macros
- **missed-call-notify** — `Call Missed` trigger → POST `{type: "missed_call", phone, time, name}`
- **call-active** — `Call Active` trigger → POST `{type: "call_started", phone, time}`
- **call-returned** — `Call Ended` trigger → POST `{type: "call_ended", phone, time}`
- MacroDroid pro version is a one-time payment (no subscription); 3 free days available by watching a rewarded ad

All 3 → same n8n webhook URL

## n8n Flow — 3 Branches

### missed_call
`Missed Call Trigger` → `Route By Type` → `Check If Pending` → `Is Already Pending?`
- **TRUE** → `Send to Slack` → `Create PagerDuty Incident`
- **FALSE** → `Save New Missed Call` → `Send to Slack` → `Create PagerDuty Incident`

### call_started
`Route By Type` → `Save Call Start Time`

### call_ended
`Route By Type` → `Check If Was Pending` → `Was It Pending?`
- **TRUE** → `Mark As Resolved` → `Was Actually Updated?`
  - **TRUE** → `Staff On Duty` → `Override Check` → `Override Exists?`
    - **TRUE** → `Send Call Returned to Slack` → `Resolve PagerDuty Incident`
    - **FALSE** → `Roster Lookup` → `Send Call Returned to Slack` → `Resolve PagerDuty Incident`
  - **FALSE** → `No Operation`
- **FALSE** → `No Operation`

## Data Tables

### missed_calls_status
Tracks call state per guest number
- `phone`, `status` (pending/resolved), `missed_time`, `name`, `call_start_time`

### staff_schedule
Shift rotation lookup
- `receptionist_name`, `shift_slot` (even_day / even_night / odd_day / odd_night)

### shift_override
Manual leave override (add row when staff on leave)
- `shift_slot`, `override_date` (YYYY-MM-DD), `override_staff`

## Shift Rotation Logic

- **Reference:** July 27, 2026 (Monday) = Week 0
- **Even weeks:** Staff A = Day (10AM–10PM), Staff B = Night
- **Odd weeks:** Staff B = Day, Staff A = Night
- **Sunday night:** Day-shift person continues until Monday 10AM
- **Leave override:** Add row in `shift_override` table only for affected shift

### Finding shift_slot for override
Calculate weeks since July 27 → even/odd → determine slot (even_day / even_night / odd_day / odd_night)

## Slack Messages

**Missed Call:**

👤 [Name/Unknown] | [phone - clickable] | Missed Call 🔴

**Call Returned:**

👤 [Name/Unknown] | [phone - clickable] | Missed Call at [HH:MM AM/PM] - Resolved by [Staff] at [HH:MM AM/PM] 🟢

## PagerDuty
- **Incident created:** on every missed call (dedup_key = phone number)
- **Incident resolved:** on call returned
- **Routing key:** stored in n8n HTTP node body

## Key Configs

- **Webhook URL:** stored in MacroDroid HTTP action & n8n Webhook node
- **Slack webhook:** stored in `Send to Slack` node
- **PagerDuty routing key:** stored in `Create PagerDuty Incident` node
- **Call ended gap threshold:** 3 seconds (duplicate event filter)
- **MacroDroid battery:** must be set to `Unrestricted` (Android settings)
- **n8n workflow:** must be `Published/Active` at all times

## Notes
- Replace webhook URL with your own before use
- MacroDroid `Call Ended` trigger fires for both missed and answered calls — gap threshold handles dedup
- Once a call is returned and resolved, the cycle resets — any future missed call from the same number starts fresh
- `missed_calls_status` table rows persist — old resolved rows don't auto-delete
- 3-second gap threshold in "Was It Pending?" node filters out duplicate `call_ended` events that fire alongside missed calls (MacroDroid limitation)
- PagerDuty escalation interval configurable in Escalation Policy settings
