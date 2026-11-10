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
  not(startsWith(toLower(triggerOutputs()?['body/subject']), 're:')),
  not(startsWith(toLower(triggerOutputs()?['body/subject']), 'fw')),
  not(contains(toUpper(triggerOutputs()?['body/subject']), '[SIGNED]')),
  not(contains(toUpper(triggerOutputs()?['body/subject']), '[CLOSED]'))
)
```
True only for a brand-new email from the team: not a reply, a forward (`FW:` / `Fwd:`), or a `[SIGNED]`/`[CLOSED]` email.

### Sent time Manila (Compose)

```
addHours(variables('varSentTime'), 8, 'ddd d MMM yyyy, h:mm tt')
```
Converts UTC to Manila time (fixed UTC+8). `convertFromUtc` was avoided because it rejects Outlook's `+00:00` timestamp format.

### Do until – Wait for signed documents

- Loop until: `varDone` *is equal to* `true`
- Count: `13` · Timeout: `P29D` (ISO 8601 duration = 29 days)

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
