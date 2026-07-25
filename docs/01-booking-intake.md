# Workflow 1: Booking Intake & Confirmation

## The Business Problem

Without automation, every new booking request at Pawsome Stays would require a
staff member to manually check a calendar or spreadsheet for kennel space,
confirm or waitlist the customer by hand, and separately notify both the
customer and the team. This doesn't scale past a handful of bookings a day,
and it's easy for a request to get lost, double-booked, or forgotten.

This workflow automates that entire intake process — from the moment a
booking request comes in, to the moment the customer knows whether they're
confirmed or waitlisted.

## What This Workflow Does

1. **Receives the request** — a booking request (pet name, check-in/check-out
   dates, service type, owner email) arrives via a webhook, simulating a real
   website's booking form submitting to the backend.
2. **Looks up the pet** — searches Airtable for a matching pet record. This
   both resolves the pet's internal record ID (needed to link the booking)
   and catches typos or unregistered pets early.
3. **Branches on whether the pet was found:**
   - **Not found** → the request is treated as an error. Full context (the
     original request) is preserved for follow-up, and the customer is told
     to get in touch rather than being silently dropped.
   - **Found** → continues to capacity checking.
4. **Checks kennel capacity** — searches existing *Confirmed* bookings for any
   that overlap the requested date range, and counts them against total
   kennel capacity (40 slots).
5. **Branches on capacity:**
   - **Room available** → creates a new *Confirmed* booking record.
   - **Full** → creates a new *Waitlist* entry instead.
6. **Builds a status message** — each of the three outcomes (confirmed /
   waitlisted / error) produces a personalized subject line and message body.
7. **Sends the customer an email** and **responds to the original webhook
   request** with a clean JSON payload reflecting the outcome — the way a
   real booking API would.

## Architecture

_[Insert screenshot of the full n8n canvas here]_

```
[Webhook] → [Search Pets] → [IF: pet found?]
                                ├─ True → [Check Overlaps] → [Count Overlaps] → [IF: has capacity?]
                                │            ├─ True  → [Create Confirmed Booking] → [Build Confirmed Message] ─┐
                                │            └─ False → [Add to Waitlist] → [Build Waitlist Message] ───────────┤
                                └─ False → [Build Error Message] ──────────────────────────────────────────────┤
                                                                                                                   ↓
                                                                                           [Send Status Email] → [Respond to Webhook]
```

## Key Design Decisions

**Webhook-based intake, not a form.**
A webhook mirrors how a real website's booking system would talk to a backend
automation — this is the same integration pattern used by tools like
Shopify, Calendly, or Stripe when they notify external systems of new events.

**Date-overlap logic, not exact date matching.**
A naive "does this date exist in another booking" check would miss most real
overlaps (partial start, partial end, fully-contained stays). Instead, this
workflow uses the standard interval-overlap condition:

> Two date ranges overlap if `existing.checkIn < requested.checkOut` AND
> `existing.checkOut > requested.checkIn`

This single Airtable formula (using `IS_BEFORE` / `IS_AFTER`) correctly
catches every overlap scenario without needing separate cases.

**Live capacity checking, not a cached counter.**
Rather than maintaining a separate "slots remaining" field that has to be
kept in sync, this workflow queries live `Bookings` data every time. A cached
number risks going stale if bookings change outside this workflow (e.g. a
manual edit in Airtable); querying live data guarantees the capacity check is
always accurate — closer to how a real production system would handle it.

**An unrecognized pet does *not* automatically create a new pet record.**
It might seem convenient to auto-create a pet profile the moment an unknown
name comes through. This was deliberately avoided: a newly-created pet with
no health screening or risk assessment could otherwise slide straight into a
paid booking, skipping a genuinely important safety step. It also risks
silently creating duplicate records from simple typos. Instead, "pet not
found" is treated as a hard stop that routes to manual review — a more
realistic pattern for a business that cares about pet safety.

**One shared email/response path, not three separate ones.**
Each outcome branch (confirmed / waitlisted / error) only builds a status
message (subject, body, outcome label). All three converge into a single
Gmail node and a single Respond to Webhook node. This avoids duplicating the
"send email" and "respond" logic three times, and makes the workflow easier
to maintain — a change to how emails are sent only needs to happen in one
place.

## Error Handling

- **Pet not found**: caught explicitly via an IF check after the pet lookup.
  The full original request is preserved (not just the pet name) so a staff
  member has complete context if manual follow-up is needed. The customer
  receives a clear response instead of a silent failure.
- **Infrastructure/API failures** (e.g. Airtable being unreachable): handled
  by a centralized Error Workflow shared across all five automations in this
  project. _[Link to Error Workflow docs once built]_

## Tech Stack

- **n8n** (self-hosted, run locally via npm)
- **Airtable** — `Pets`, `Bookings`, and `Waitlist` tables act as the
  business's operational database
- **Gmail** (OAuth2) — customer-facing transactional emails

## Example Payloads

**Incoming webhook request:**
```json
{
  "petName": "Buddy",
  "checkInDate": "2026-08-10",
  "checkOutDate": "2026-08-15",
  "serviceType": "Boarding",
  "ownerEmail": "owner@example.com"
}
```

**Response — booking confirmed:**
```json
{
  "status": "confirmed",
  "message": "Hi! Your booking for Buddy from 2026-08-10 to 2026-08-15 is confirmed. See you soon!"
}
```

**Response — waitlisted:**
```json
{
  "status": "waitlisted",
  "message": "Hi! We're currently full for your requested dates, but Buddy has been added to our waitlist. We'll notify you if a spot opens up."
}
```

**Response — pet not found:**
```json
{
  "status": "error",
  "message": "Hi, we couldn't find a pet matching \"Zeus\" on file. Please contact us so we can help complete your pet's profile before booking."
}
```
