# Maintenance guide

How to run, change and troubleshoot Attachment Radar once it's live.

## How each message is triggered

| Message | Flow | When it posts |
|---|---|---|
| Offer extended | A | Within minutes of a team member sending a new offer |
| Follow-up needed (reminder n of 2) | A | At each scheduled check (every 2 days) if nothing has come back |
| No more automatic reminders | A | Straight after reminder 2 if nothing has come back |
| Signed documents received from candidate | B | Within minutes of the candidate's reply with an attachment |
| Signed documents received (forwarded) | B | Within minutes of a `[SIGNED]` forward to the distro |
| Reminders stopped | A | At the **next scheduled check** after documents arrive (up to 2 days later) |

Flow A and Flow B don't talk to each other. "Signed documents received" (instant) followed later by "Reminders stopped" (next check) is expected.

## Golden rules for changes

1. **Running runs keep the version they started with.** Saving a change only affects offers sent afterwards. After any change, check **28-day run history** for runs still **Running**.
2. **Cancel only test runs.** A cancelled live run stops that candidate's reminders and its "Reminders stopped" message.
3. **Never Resubmit a cancelled false run.** It restarts it.
4. **Rename steps before editing expressions that refer to them** (spaces become `_`).
5. **After testing, put the Delay back to 2 Day** and save.

## Common changes

### Change the number of reminders
Flow A → *Wait for signed documents* → **Count** (= maximum reminders). Keep Count × Delay under 30 days (the platform's maximum run length). Update "of 2" in *Post follow-up reminder* and the line in *Post offer extended* to match.

### Change the reminder interval
Flow A → *Delay* inside the loop. Same 30-day rule applies.

### Move notifications to a different Teams chat
Update **Group chat** in every Teams step: Flow A (*Post offer extended*, *Post follow-up reminder*, *Post reminders stopped*, *Post reminders ended*) and Flow B (*Post signed forward*, *Post received from candidate*). The chat needs a name and at least one message before it appears in the list.

### Add another company email domain
Add it to *Is new offer* (sender and candidate checks) in Flow A and to *Sender is a candidate* in Flow B.

## Testing safely

- Use an **external** address (e.g. a personal mailbox) as the candidate. Flow B ignores company senders by design.
- Use a **new test subject** each time so old test emails don't interfere.
- Set Flow A's Delay to `3 Minute` for testing, then back to `2 Day`.
- To re-check how a flow treats a specific email without sending a new one: **Test → Automatically → With a recently used trigger** and pick that run.

## Troubleshooting

| Symptom | Likely cause | What to check / do |
|---|---|---|
| "Offer extended" for something that isn't an offer | Subject or recipient passed *Is new offer* | Open the run and check the subject and `varCandidateEmail`; cancel the run; tighten *Is new offer* if needed |
| Reminders continue after the candidate replied | Reply not found by *Get offer thread* | Reply is in the offers folder? Fetch Only Unread = **No**? Subject still contains the offer subject? Check *Get offer thread* raw outputs in the run |
| No "Signed documents received" | Candidate replied only to the sender, sent a link instead of an attachment, changed the subject, or used another address | Teammate forwards to the distro with `[SIGNED]` in the subject |
| "Reminders stopped" hasn't appeared | Not due yet: it posts at the next 2-day check | Check the run is still **Running** |
| Messages show your name instead of the bot | *Post as* set to **User** | Set *Post as* to **Flow bot** |
| A **For each** appeared around a step | A dynamic-content chip came from *Get emails* (a list) | Undo, and use the trigger's value or a `triggerOutputs()` expression |

## Team conventions (share with the team)

1. Subject always starts with the company name: `[Company name] - Offer of Employment for [Candidate Full Name]`.
2. **To:** candidate only · **CC:** distro.
3. Reply in the same thread for follow-ups and responses. Replies never trigger notifications.
4. Signed documents that came only to you: forward to the distro with `[SIGNED]` at the start of the subject.
5. Offer declined or withdrawn: reply in the thread to the distro with `[CLOSED]` at the start of the subject.
