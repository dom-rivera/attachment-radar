# Changelog

All notable changes to Attachment Radar. Dates are when the change went live.

## v1.2 – 11 Nov 2026 · Ignore auto-replies and internal recipients

**Issue found in production:** a colleague's out-of-office auto-reply ("Automatic reply: [offer subject]") landed in the offers folder. Flow A treated it as a new offer and started a reminder run for a "candidate" who was actually a team member.

**Root cause:** the *Is new offer* check only excluded subjects starting with `RE:` or `FW:`, so the `Automatic reply:` prefix passed.

**Changes (Flow A → *Is new offer*):**
- The subject must now **start with** the company name and contain "offer of employment", i.e. the standard offer subject. This excludes every reply-style prefix (`RE:`, `FW:`, `Automatic reply:`, `Undeliverable:`, etc.) in one rule.
- The candidate (first **To** address) must be an **external** address.
- Kept: sender must be a team member; `[SIGNED]` / `[CLOSED]` emails are ignored.

**Clean-up:** cancelled the incorrect run from run history so no false reminders were posted.

**Verified:** replayed the auto-reply with *Test → With a recently used trigger*. *Is new offer* went to **False**, the run succeeded, and nothing was posted.

**Team convention added:** offer subjects must always start with the company name.

## v1.1 – 11 Nov 2026 · Cap follow-ups at two reminders

**Request:** the team wanted at most two automatic nudges per candidate instead of reminders every 2 days for up to 26 days.

**Changes (Flow A):**
- *Wait for signed documents* loop **Count** `13` → `2` (Timeout unchanged at `P29D`).
- *Post offer extended*: last line now reads "Up to 2 reminders will post, 2 days apart, until signed documents are received."
- *Post follow-up reminder*: heading now shows **reminder n of 2** using `add(iterationIndexes('Wait_for_signed_documents'), 1)`.
- *Post reminders ended*: now reads "No more automatic reminders for [candidate]. Please follow up manually."

**Rollout note:** runs that had already started kept the old settings; test runs were cancelled and live runs were allowed to finish.

## v1.0.1 – 11 Nov 2026 · First live case

- Confirmed end to end with a real candidate: **Offer extended** → **Signed documents received from candidate** within minutes of the reply.
- Confirmed that the team replying to the candidate (with or without attachments) does **not** trigger notifications.
- Documented expected behaviour: **Reminders stopped** posts at Flow A's next scheduled check, not immediately (Flow B gives the instant alert).

## v1.0 – 10 Nov 2026 · Initial release

Flow A (offer extended + reminders) and Flow B (signed documents received) built and tested. Fixes made during build and testing:

| Problem | Fix |
|---|---|
| `convertFromUtc` failed on Outlook's `+00:00` timestamps | Switched to `addHours(…, 8, format)` (Manila is fixed UTC+8) |
| Filter array: "'where' must be a template language expression" | Removed a leading space before `@` |
| `@` pasted into the fx editor caused "This expression has a problem" | Use `@` only when typing directly into advanced mode |
| Power Automate auto-wrapped a Teams post in **For each** | Replaced chips from *Get emails* with trigger expressions |
| Delay placed after the loop instead of inside | Moved it between the loop's **Do** label and condition bar |
| Do until default timeout `PT1H` | Set Timeout `P29D` |
| Reminders didn't stop after the candidate replied | *Get emails* had **Fetch Only Unread = Yes**; set to **No**, Top raised to 50, sender matched with `contains` |
| Flow B ignored test replies | Tests used a company address as the "candidate"; Flow B deliberately ignores internal senders, so tests now use an external address |
