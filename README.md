[README (1).md](https://github.com/user-attachments/files/29783821/README.1.md)# Avensa Health — Automated Review Request System

A two-scenario [Make.com](https://make.com) automation that requests Google reviews from medspa clients after a completed appointment, routes responses based on sentiment, and triggers staff follow-up for unhappy clients — without gating access to the public review link based on sentiment.

## How it works

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  01 - Send Review Request    │         │  02 - Process Review Response │
│  (scheduled/triggered)       │         │  (webhook, real-time)          │
└─────────────────────────────┘         └──────────────────────────────┘
 Airtable → Twilio SMS → Airtable         Tally webhook → Airtable →
 (find completed, unsent →                Router (rating 4-5 vs 1-3) →
  send link → mark sent)                  SMS/Email/Slack + Airtable update
```

**Scenario 1** finds appointments that are `Completed` and haven't had a review request sent yet, texts the client a link to a Tally feedback form, and marks the record as requested.

**Scenario 2** listens for Tally form submissions via webhook, looks up the full client record from Airtable, and branches based on their 1–5 rating:
- **4–5 (Positive):** sends a Google review link via SMS + email, marks the record
- **1–3 (Negative):** sends a service-recovery SMS, alerts staff in Slack, flags the record

## Scenario 1 — Send Review Request

| Module | Type | Purpose |
|---|---|---|
| `Airtable — Search Records` | `airtable:ActionSearchRecords` | Pulls records where `{Status} = 'Completed'` and `{Review Requested}` is unchecked |
| `Twilio — Send SMS` | `twilio:SendSMS` | Texts the client a Tally form link with `record_id` appended as a URL param |
| `Airtable — Update Record` | `airtable:ActionUpdateRecords` | Marks `Review Requested` = true so this record isn't re-pulled next run |

**Airtable filter formula:**
```
AND({Status} = 'Completed', NOT({Review Requested}))
```

**SMS sent:**
```
Hi {First Name}! Thanks for visiting Avensa Health. Rate us: https://tally.so/r/rjWJXR?record_id={record}
```

## Scenario 2 — Process Review Response

| Module | Type | Purpose |
|---|---|---|
| `Webhook — Custom webhook` | `gateway:CustomWebHook` | Receives the Tally form submission payload |
| `Tools — Set multiple variables` | `util:SetVariables` | Extracts `record_id`, `rating`, `comment` from the Tally `fields[]` array |
| `Airtable — Update Record` | `airtable:ActionUpdateRecords` | Writes `rating`, `comment`, and submission timestamp back to the original row |
| `Airtable — Get Record` | `airtable:ActionGetRecord` | Re-fetches the full client record (First Name, Email, Phone) by `record_id` — used for personalizing outbound messages |
| `Router` | `builtin:BasicRouter` | Branches on `rating` |

### Route: Positive Feedback (rating contains 4 or 5)
1. **Twilio SMS** → sends Google review link
2. **Gmail** → sends Google review email
3. **Airtable Update Record** → marks review-sent flag

### Route: Negative Feedback (rating contains 1, 2, or 3)
1. **Twilio SMS** → service recovery message, no review link included
2. **Slack** → alerts staff channel with client name, rating, comment, record ID
3. **Airtable Update Record** → flags record as "🚨 Negative Review Detected"

## Why sentiment doesn't gate the Google link

Google's review policy prohibits **review gating** — screening customers privately and only sending the public review link to those who respond positively. This system avoids that by *not* sending the review link at all in the negative branch (rather than sending a filtered/conditional version of it), and keeps the sentiment check purely internal for service recovery. If you want the strictest-compliant version, remove the router entirely and send an identical review-request message to 100% of respondents, with negative-feedback detection happening as a separate internal-only step.

## Setup requirements

- **Airtable base** with a table containing: `Status`, `Review Requested` (checkbox), `Phone`, `First Name`, `Last Name`, `Email`, plus fields for `Rating` and `Comment`
- **Twilio account** — a verified sending number; trial accounts are limited to verified recipient numbers and enforce stricter SMS segment limits (see Known Issues)
- **Tally form** with hidden fields matching your URL query params exactly (case- and underscore-sensitive), plus a rating field and a free-text comment field
- **Gmail connection** for the positive-branch email
- **Slack connection** with a channel for negative-feedback alerts

### Tally hidden field setup
Field names in Tally must exactly match the query param names in the SMS link. A field named `first name` (with a space) will **not** populate from a URL param of `first_name` — Tally does exact-string matching and silently drops the value instead of erroring.

## Known issues / TODO

- **Gmail body still references `{{first_name}}`** in the positive-feedback branch, but the `first_name` variable was removed from the round-trip in favor of the `Airtable — Get Record` lookup. This will render as blank. Should be updated to `{{29.\`First Name\`}}` to match the rest of that branch.
- **Slack alert concatenates `{{29.\`First Name\`}}{{29.\`Last Name\`}}` with no space** between them — will render as e.g. `MariaSantos`. Add a space in the template.
- **Array-index field mapping is fragile.** The `Set Variables` step reads Tally's hidden/visible fields by array position (`fields[1].value`, `fields[2].value`, etc.). Reordering, adding, or deleting a question in the Tally form will silently shift these indices and break the mapping. Consider switching to a `label`-matching lookup if the form will be edited again.
- **Twilio trial account limitations** — outbound SMS only reaches verified numbers, all messages get a "Sent from your Twilio trial account" prefix, and message segments are capped. Upgrade the account before going live with real client traffic.

## Scenario naming convention

```
Medspa [Location Letter] - [Order] - [Purpose]
e.g. Medspa A - 01 - Send Review Request
     Medspa A - 02 - Process Review Response
```

This keeps multi-location deployments (e.g. `Medspa B`, `Medspa C`) distinguishable in the Make.com scenario list.

