# Build guide

Step-by-step instructions to rebuild both flows in Power Automate's **new designer**. Expressions are listed in full in [`expressions.md`](expressions.md).

## Prerequisites

1. **Offer email convention**
   - Subject: `Company - Offer of Employment for [Candidate Full Name]`
   - **To:** the candidate only · **CC:** the team distro
2. **Outlook rule** in the flow owner's mailbox: *Recipient address includes [distro]* → *Move to* **Offer of Employment**.
3. **Teams group chat** with the team. Give it a name and send one message (Power Automate only lists named chats that have messages).

## Designer basics

| To do this | Do this |
|---|---|
| Add a step | Click **⊕** → **Add an action** → search → select |
| Rename a step | Click the card → click its title in the left panel → type → Enter |
| Show hidden fields | **Advanced parameters** → **Show all** |
| Insert a value from an earlier step | Click the field → **⚡** (or type `/`) |
| Insert an expression | Click the field → **fx** → paste → **Add** |
| Copy a step | Right-click the card → **Copy action** → **⊕** → **Paste an action** |

Rename steps **before** writing expressions that refer to them (spaces become `_`, e.g. `Get offer thread` → `Get_offer_thread`).

---

## Flow A – Offer extended and reminders

**Create:** + Create → Start from blank → **Automated cloud flow** → trigger **When a new email arrives (V3)** → Create.

### 1. Trigger
| Field | Value |
|---|---|
| Folder | Offer of Employment |
| Subject Filter | `Offer of Employment` |
| Include Attachments / Only with Attachments | No / No |
| Importance | Any |

### 2. Six Initialize variable steps (top level, not inside a condition)
`varCandidateEmail`, `varOfferSubject`, `varSentBy`, `varSentTime` (String), `varDone` (Boolean, `false`), `varStopReason` (String, empty).

### 3. Condition `Is new offer`
Left: the *Is new offer* expression · **is equal to** · Right: fx `true`. Leave **False** empty. Everything below goes in **True**.

### 4. Compose `Sent time Manila`
Input: `addHours(variables('varSentTime'), 8, 'ddd d MMM yyyy, h:mm tt')`

### 5. Post message in a chat or channel `Post offer extended`
Post as **Flow bot** · Post in **Group chat** · pick the chat.
```
Offer extended
Candidate: [varCandidateEmail]
Offer: [varOfferSubject]
Sent by: [varSentBy]
Sent: [Outputs of Sent time Manila]
Up to 2 reminders will post, 2 days apart, until signed documents are received.
```

### 6. Do until `Wait for signed documents`
- Loop until `varDone` **is equal to** fx `true`
- **Count** `2` (= maximum number of reminders) · **Timeout** `P29D`

In the new designer, steps **inside** the loop go between the **Do** label and the grey condition bar at the bottom of the box.

Inside the loop, in order:

| # | Step | Settings |
|---|---|---|
| a | **Delay** | Count `2`, Unit `Day` |
| b | **Get emails (V3)** → `Get offer thread` | Folder: Offer of Employment · Subject Filter: ⚡ `varOfferSubject` · Fetch Only Unread: **No** (otherwise replies someone has opened are skipped) · Include Attachments: No · Top: `50` |
| c | **Filter array** → `Find signed or closed` | From: `body('Get_offer_thread')?['value']` · Edit in advanced mode → filter expression (single line; starts with `@` if typed in the box, no `@` if pasted via **fx**) |
| d | **Condition** → `Documents back` | `length(body('Find_signed_or_closed'))` **is greater than** `0` |

- **Documents back → True:** Set variable `varStopReason` = `first(body('Find_signed_or_closed'))?['subject']`, then Set variable `varDone` = `true`
- **Documents back → False:** copy of the Teams post → `Post follow-up reminder`:
```
Follow-up needed (reminder [fx reminder number] of 2)
No signed documents yet from [varCandidateEmail]
Offer: [varOfferSubject]
Sent by: [varSentBy] on [Outputs of Sent time Manila]
Days since offer: [fx days-since expression]
```

### 7. After the loop: Condition `Stopped by documents`
Placed under the loop's grey bar, still inside *Is new offer → True*. `varDone` **is equal to** fx `true`.
- **True:** `Post reminders stopped`: `Reminders stopped for [varCandidateEmail]` / `Found: [varStopReason]`
- **False:** `Post reminders ended`: `No more automatic reminders for [varCandidateEmail]` / `Offer: [varOfferSubject]. Please follow up manually.`

---

## Flow B – Signed documents received

**Create:** Automated cloud flow → **When a new email arrives (V3)**.

### 1. Trigger
Same as Flow A, except **Only with Attachments: Yes**.

### 2. Condition `Is SIGNED forward`
`contains(toUpper(triggerOutputs()?['body/subject']), '[SIGNED]')` **is equal to** fx `true`

- **True:** `Post signed forward`:
```
Signed documents received (forwarded)
Forwarded by: [From]
Offer: [Subject]
Received: [fx addHours(... receivedDateTime ...)]
```

### 3. False branch
1. **Get emails (V3)** → `Find offer to sender`: Folder Offer of Employment · **To:** fx `triggerOutputs()?['body/from']` · Subject Filter `Offer of Employment` · Fetch Only Unread **No** · Top `1`
2. **Condition** `Sender is a candidate`: *Sender is a candidate* expression **is equal to** fx `true`
3. **True:** `Post received from candidate` (use fx trigger expressions, not chips from *Get emails*):
```
Signed documents received from candidate
From: [fx from]
Offer: [fx subject]
Received: [fx addHours(...)]
Preview: [fx bodyPreview]
```

---

## Testing

1. In Flow A, temporarily set **Delay** to `3` **Minute**.
2. **Test → Manually**, then send a test offer to a personal address (distro in CC, PDF attached).
3. Expect: *Offer extended* → *Follow-up needed (reminder 1 of 2)* → *(reminder 2 of 2)* ~3 min apart → *No more automatic reminders*. (Run a second test and reply before reminder 2 to check that reminders stop.)
4. Reply All from the personal address with a PDF → Flow A: *Reminders stopped*; Flow B: *received from candidate*.
5. Forward the reply to the distro with `[SIGNED]` at the start of the subject → Flow B: *forwarded*.
6. **Reset Delay to 2 Day**, cancel any test runs still *Running*, and turn off any older notification flow.

## Team conventions

1. Subject always `Company - Offer of Employment for [Candidate Full Name]`.
2. **To:** candidate only · **CC:** distro.
3. Follow-ups and re-sends: reply in the same thread.
4. Signed documents that came only to you: forward to the distro with `[SIGNED]` at the start of the subject.
5. Offer declined or withdrawn: reply in the thread to the distro with `[CLOSED]` at the start of the subject.
