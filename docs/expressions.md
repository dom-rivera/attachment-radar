# Expressions reference

Every expression used in both flows. Replace `@yourcompany.com` / `@yourcompany-alt.com` with your organisation's email domains.

## Flow A – Offer extended and reminders

### Variables (Initialize variable)

| Variable | Type | Expression | What it holds |
|---|---|---|---|
| `varCandidateEmail` | String | `trim(toLower(first(split(triggerOutputs()?['body/toRecipients'], ';'))))` | First address on the **To** line, lower-case |
| `varOfferSubject` | String | `triggerOutputs()?['body/subject']` | The offer email's subject |
| `varSentBy` | String | `triggerOutputs()?['body/from']` | Team member who sent the offer |
| `varSentTime` | String | `triggerOutputs()?['body/receivedDateTime']` | When the offer arrived (UTC) |
| `varDone` | Boolean | `false` | Becomes `true` when documents are back |
| `varStopReason` | String | *(empty)* | Subject of the email that stopped the reminders |

### Is new offer (Condition: expression *is equal to* `true`)

```
and(
  or(
    contains(toLower(triggerOutputs()?['body/from']), '@yourcompany.com'),
    contains(toLower(triggerOutputs()?['body/from']), '@yourcompany-alt.com')
  ),
  startsWith(toLower(trim(triggerOutputs()?['body/subject'])), 'your company name'),
  contains(toLower(triggerOutputs()?['body/subject']), 'offer of employment'),
  not(contains(variables('varCandidateEmail'), '@yourcompany.com')),
  not(contains(variables('varCandidateEmail'), '@yourcompany-alt.com')),
  not(contains(toUpper(triggerOutputs()?['body/subject']), '[SIGNED]')),
  not(contains(toUpper(triggerOutputs()?['body/subject']), '[CLOSED]'))
)
```
True only for a brand-new offer, i.e. all of:

| Rule | Why |
|---|---|
| Sent **from** a company address | Only the team sends offers |
| Subject **starts with** the company name (the standard offer subject) | Automatically excludes `RE:`, `FW:`, `Automatic reply:`, `Undeliverable:` and similar prefixes |
| Subject contains "offer of employment" | Confirms it's an offer email |
| Sent **to** an outside address | A real candidate is never a colleague |
| No `[SIGNED]` / `[CLOSED]` | Those are manual stop signals |

*Version 1.0 excluded only `RE:` and `FW:` prefixes. A colleague's out-of-office auto-reply ("Automatic reply: …") slipped through and was treated as a new offer, which led to this stricter version (v1.2).*

### Sent time Manila (Compose)

```
addHours(variables('varSentTime'), 8, 'ddd d MMM yyyy, h:mm tt')
```
Converts UTC to Manila time (fixed UTC+8). `convertFromUtc` was avoided because it rejects Outlook's `+00:00` timestamp format.

### Do until – Wait for signed documents

- Loop until: `varDone` *is equal to* `true`
- Count: `2` (maximum reminders) · Timeout: `P29D` (ISO 8601 duration = 29 days; a safety net, since Count ends the loop first)

To allow more reminders, raise Count and keep Count × Delay under 30 days (the platform's maximum run length).

### Find signed or closed (Filter array)

From:
```
body('Get_offer_thread')?['value']
```
Filter. If typed straight into the advanced-mode box, start with `@` and no leading spaces. If pasted into the **fx** editor, leave the `@` out:
```
@and(greater(ticks(item()?['receivedDateTime']), ticks(variables('varSentTime'))), or(contains(toUpper(item()?['subject']), '[SIGNED]'), contains(toUpper(item()?['subject']), '[CLOSED]'), and(contains(toLower(item()?['from']), variables('varCandidateEmail')), equals(item()?['hasAttachments'], true))))
```
Keeps emails received **after** the offer that are `[SIGNED]`, `[CLOSED]`, or from the candidate with an attachment. `ticks()` turns timestamps into numbers so they can be compared.

### Documents back (Condition)

```
length(body('Find_signed_or_closed'))
```
*is greater than* `0`

### Set variables (True branch)

| Variable | Value |
|---|---|
| `varStopReason` | `first(body('Find_signed_or_closed'))?['subject']` |
| `varDone` | `true` |

### Reminder number (in the reminder message)

```
add(iterationIndexes('Wait_for_signed_documents'), 1)
```
`iterationIndexes` is 0 for the loop's first round, so adding 1 gives "reminder 1 of 2", "reminder 2 of 2".

### Days since offer (in the reminder message)

```
div(sub(ticks(utcNow()), ticks(variables('varSentTime'))), 864000000000)
```
One day = 864,000,000,000 ticks (100-nanosecond units).

## Flow B – Signed documents received

### Is SIGNED forward (Condition: expression *is equal to* `true`)

```
contains(toUpper(triggerOutputs()?['body/subject']), '[SIGNED]')
```

### Find offer to sender (Get emails V3 → To field)

```
triggerOutputs()?['body/from']
```

### Sender is a candidate (Condition: expression *is equal to* `true`)

```
and(greater(length(body('Find_offer_to_sender')?['value']), 0), not(contains(toLower(triggerOutputs()?['body/from']), '@yourcompany.com')), not(contains(toLower(triggerOutputs()?['body/from']), '@yourcompany-alt.com')))
```

### Message fields

| Field | Expression |
|---|---|
| From | `triggerOutputs()?['body/from']` |
| Offer | `triggerOutputs()?['body/subject']` |
| Received | `addHours(triggerOutputs()?['body/receivedDateTime'], 8, 'ddd d MMM yyyy, h:mm tt')` |
| Preview | `triggerOutputs()?['body/bodyPreview']` |

Using trigger expressions instead of dynamic-content chips from *Get emails* prevents Power Automate from wrapping the post in an unwanted **For each** loop.
