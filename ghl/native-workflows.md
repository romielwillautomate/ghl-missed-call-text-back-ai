# Native GoHighLevel workflows: the recipe

The GoHighLevel side of this build runs as two native workflows inside the sub-account. They are built by hand from this recipe. The n8n workflow in `workflow/` does the AI and integration side.

- **Workflow A, "Missed call text back"**: a missed inbound call tags the caller, opens an opportunity in "New leads", texts them (one text during business hours, another after hours) and tells n8n.
- **Workflow B, "Missed call replies to n8n"**: when a contact with the `missed-call` tag texts back, it sends the message to n8n. n8n decides the answer and sends it through the Conversations API.

Every trigger, filter, action and merge field name below was checked against HighLevel's help center (help.gohighlevel.com) on October 26, 2026, and the riskiest ones (Call Details, If/Else, Customer Replied, the opportunity actions, the webhook actions, merge fields and workflow settings) again on October 27, 2026; the table at the end lists the article for each. Anything marked **Not confirmed** could not be verified in the current docs: check it in your account while you build, and test with your own phone before going live. Menu paths can differ slightly between accounts.

## Before you start

| Item | What to do | Status |
|---|---|---|
| Built-in Missed Call Text Back | Turn it off, or callers get two texts: **Phone System → Voice → Voicemail & Missed Call Text Back → Missed Call Text Back**, untick **Enable Missed Call Text Back** | Confirmed (help article 48001239140) |
| Other auto replies | Turn off anything else that answers SMS on this number, such as a Conversation AI bot, or customers get two answers | Not a doc fact, a precaution |
| Pipeline | **Opportunities → Pipelines**: a pipeline named, for example, "Service requests" with at least the stages **New leads** and **Booked** | Standard setup. n8n needs the pipeline ID and the Booked stage ID ([docs/setup.md](../docs/setup.md) shows the API call) |
| Calendar | The calendar customers book into (for example "Service visit", 60 minutes) | n8n needs its ID |
| Custom value | **Settings → Custom Values → + Add Custom Value**: name **Business Name**, value `Northside Plumbing`. It is used as `{{custom_values.business_name}}` | Confirmed: "ex: My Custom Value will be custom_values.my_custom_value" (help article 155000004705) |
| Custom fields | None needed. n8n keeps the conversation state (reply count, offered times, why they called) | |
| Tags | Created the first time an action uses them. Workflow A adds `missed-call`; n8n adds `missed-call-replied`, `missed-call-booked`, `missed-call-needs-human`, `missed-call-emergency`, `missed-call-opted-out` and `missed-call-done` | |
| Premium action | Custom Webhook is an LC Premium action: 100 free lifetime premium executions per account, then $0.01 per execution. The agency turns premium actions on under **Settings → Company settings → Workflow - Premium Features**. Each missed call and each customer text uses one execution | Confirmed (help article 155000005678) |
| Shared secret | Make up one long random string. It is `GHL_WEBHOOK_SECRET` in n8n and the Bearer token in both webhook actions | |
| Call tracking | If calls ring through to a cell phone, a missed call can still be marked "completed", which this workflow does not catch. HighLevel's answer is **Call Connect**: **Settings → Phone System**, the three dots next to the number, **Edit Configuration → Advanced Settings**, turn on **Call Connect** and **Play Whisper Message** | Confirmed: "Twilio marks all calls as "Completed" once they finish routing, even if no one answers" (help article 48001181825) |

## Workflow A: Missed call text back

**Automation → Workflows → Create Workflow → Start from scratch.** Name: `Missed call text back (n8n)`.

**Settings tab:** turn **Allow Re-entry** on, so a customer who calls again weeks later is texted again ("A contact cannot re-enter if they are still active in the workflow", help article 48001239875). Leave the **Time Window** off: the text should go out right after the call. Set the workflow **Timezone** to **Account Timezone**, so time based steps use the business's time zone and not the caller's (with "Contact Timezone", "Actions will execute based on the individual timezone of each contact", help article 48001239875).

### Trigger: Call Details

**Add New Trigger → Events → Call Details.** (The help article for this trigger used to be called "Call Status"; its old URL now opens "Workflow Trigger - Call Details".)

| Filter | Value | Status |
|---|---|---|
| Call Direction | Incoming | Confirmed |
| Call Status | **busy**, **no-answer** and **voicemail** (pick all three; the filter takes several values) | Confirmed in help article 48001212511 (September 2026). An older article (155000002552) writes "not answered" instead of "no-answer": pick whichever your dropdown shows |
| In Phone Number | Optional: your main business line, if the sub-account has several numbers | Confirmed filter name |

The help article's own example: "If Call Direction = Incoming and Call Status includes busy, voicemail, and no-answer, the call must be incoming and match one of the selected call-status outcomes before the trigger qualifies." The trigger has no tag filter.

### Actions, in order

**1. Remove Contact Tag** (name it "Clear the last conversation")
- Tags: `missed-call-booked`, `missed-call-needs-human`, `missed-call-emergency`, `missed-call-done`
- Why: n8n stops answering a contact with one of these tags. A new missed call starts a new conversation. `missed-call-opted-out` stays, because an opt out must hold.
- Status: action name confirmed ("Tag removal is done using the Remove Contact Tag action").

**2. Add Contact Tag** (name it "Mark as missed call")
- Tag: `missed-call`
- Status: confirmed.

**3. Create/Update Opportunity** (name it "Open in New leads")
- Pipeline: Service requests
- Stage: New leads
- Opportunity name: `{{contact.name}} (missed call)`
- Opportunity source: `Missed call`
- Status: Open
- **Allow Opportunity to Move**: off, so a repeat caller whose job is already booked does not move back to New leads
- Status: the action name and the Pipeline, Stage, Status, Value and "Allow Opportunity to Move" fields are confirmed in the pipelines guide (help article 155000005062). This action's own help article returns 404, so the **Opportunity name** and **Opportunity source** fields are taken from the separate "Create Opportunity" action (help article 155000004752): **Not confirmed** for Create/Update. If your account only offers the separate **Create Opportunity** action, use it with **Duplicate Opportunity** off.

**4. If/Else** (name it "Open now?") **Not confirmed, read this first.**
The current If/Else help article (155000002471, rechecked October 27, 2026; last modified September 1, 2026) lists no condition for the current time or day. An older official HighLevel article (48001180266, now returning 404; its text survives in search indexes) described time comparisons on the **current day of the week** and the **current hour**, with the operators **Is**, **Is Not**, **Is After**, **Is on or After**, **Is Before** and **Is on or Before**. It also warned that hours compare whole hours: "Current Hour Is After 6 PM" is only true from 7:00 PM. Third party guides show "Current day of week is not Saturday" in the If/Else condition list. Which time zone the comparison uses is not documented either. For the workflow **Timezone** setting, the docs say "All time-based actions will execute based on the timezone set for the entire location (account)" when it is Account Timezone (help article 48001239875), so that is the safe choice, but they do not say that If/Else conditions follow it.

If your If/Else offers these conditions, add two branches (example hours: Monday to Friday 7 AM to 6 PM, Saturday 8 AM to 2 PM):

| Branch | Conditions (all must match) |
|---|---|
| Weekday open | Current Day of the Week **Is Not** Saturday; Current Day of the Week **Is Not** Sunday; Current Hour **Is on or After** 7 AM; Current Hour **Is Before** 6 PM |
| Saturday open | Current Day of the Week **Is** Saturday; Current Hour **Is on or After** 8 AM; Current Hour **Is Before** 2 PM |
| None (created automatically) | After hours |

Then test both sides of each boundary with a test contact before going live.

**If your If/Else has no time conditions:** skip this step, use one text that works at any hour (step 5, the business hours text with "If we are closed, we will text you back first thing" added), and send `textSent` = `single` in step 6.

**5. Send SMS** (one in each branch)
- From Number: your business line
- In both "open" branches, the business hours text:

  ```
  Hi, this is {{custom_values.business_name}}. Sorry we missed your call! What can we help you with? Reply here and we will text you right back.
  ```

- In the None branch, the after hours text:

  ```
  Hi, this is {{custom_values.business_name}}. Sorry we missed your call. We are closed right now and open again at 7 AM. Tell us what you need and we will text you back. For flooding, a burst pipe or a gas smell, reply EMERGENCY.
  ```

- The texts do not use `{{contact.first_name}}`, because a new caller usually has no name in GoHighLevel yet and the text would read "Hi , this is".
- HighLevel adds "Reply STOP to unsubscribe" and "Thanks, <Location Name>" to the first SMS to a new contact by itself (help article 48001213941), so the texts do not repeat it.
- Status: action and fields confirmed (help article 155000002474).

**6. Custom Webhook** (one at the end of each branch, name it "Tell n8n")
- **Send Data → Custom Webhook**
- Event: **POST** (the simple mode with Body key/value pairs; HighLevel builds the body itself)
- URL: `https://YOUR-N8N-DOMAIN/webhook/ghl-missed-call-text-back`
- Authorization: **Bearer Token**, token: your `GHL_WEBHOOK_SECRET`
- Body, key and value pairs:

  | Key | Value |
  |---|---|
  | `event` | `missed_call` |
  | `locationId` | `{{location.id}}` |
  | `contactId` | `{{contact.id}}` |
  | `textSent` | `business_hours` in the open branches, `after_hours` in the None branch (`single` if you skipped step 4) |

- Leave "Save response from this Webhook" off.
- Status: Event modes, Bearer Token authorization and the merge field picker confirmed (help article 155000003305). `{{location.id}}` and `{{contact.id}}` are in the merge field list. **Not confirmed:** the content type the simple POST mode uses (the article says "Switch to CUSTOM if your provider expects JSON", which suggests form encoding). n8n reads both form encoded and JSON bodies, and the test sends both.

**Publish** the workflow.

## Workflow B: Missed call replies to n8n

**Automation → Workflows → Create Workflow → Start from scratch.** Name: `Missed call replies to n8n`.

**Settings tab:** turn **Allow Re-entry** on. Every text must enter the workflow again.

### Trigger: Customer Replied

| Filter | Value | Status |
|---|---|---|
| Reply Channel | SMS | Confirmed ("Select SMS as the Reply Channel", help article 48001156789) |
| Has Tag | `missed-call` | Confirmed filter name (help article 155000002677) |

Do **not** add **Replied to Workflow**. It "runs only if the contact replied to a specific workflow's messages", and the assistant's texts are sent by n8n through the API, not by a workflow, so the customer's later replies would most likely be dropped. Whether Customer Replied fires for a reply to a text sent through the API is **Not confirmed** in the docs; the triggers list says it "fires when the contact replies on any connected channel". Check it in your first live test (reply twice).

### Action: Custom Webhook (name it "Send the text to n8n")

- Event: **POST** (Body key/value pairs)
- URL: `https://YOUR-N8N-DOMAIN/webhook/ghl-missed-call-text-back` (the same URL as workflow A)
- Authorization: **Bearer Token**, token: your `GHL_WEBHOOK_SECRET`
- Body:

  | Key | Value |
  |---|---|
  | `event` | `reply` |
  | `locationId` | `{{location.id}}` |
  | `contactId` | `{{contact.id}}` |
  | `body` | `{{message.body}}` (pick **Message → Message Body**) |
  | `messageType` | `SMS` |
  | `direction` | `inbound` |

- There is no merge field for the message id (the merge field list has only `{{message.body}}` and `{{message.subject}}`), so n8n finds the message in the conversation history and uses its id to skip repeat deliveries.
- If you prefer **Event: CUSTOM** (Method POST, Content-Type `application/json`, Raw Body), type the same keys as JSON. Then send yourself a test text with a double quote in it: a raw body with `{{message.body}}` inside quotes may break on it. The key/value mode avoids that.

**Publish** the workflow.

## The free alternative: the standard Webhook action

The standard **Webhook** action (help article "Workflow Action - Webhook (Outbound)") is free but cannot set headers. Use it with the same URL and Method POST, and add **Custom Data** items:

| Workflow | Custom Data items |
|---|---|
| A | `event` = `missed_call`, `contactId` = `{{contact.id}}`, `textSent` = `business_hours` or `after_hours`, `secret` = your `GHL_WEBHOOK_SECRET` |
| B | `event` = `reply`, `contactId` = `{{contact.id}}`, `messageBody` = `{{message.body}}`, `secret` = your `GHL_WEBHOOK_SECRET` |

What the docs say about its payload (help article 155000003299): the contact fields (`first_name`, `last_name`, `full_name`, `email`, `phone`, `tags` and more), `location` with its `id`, `message` with `type`, `body`, `direction` and `status` when the trigger has a message, and `workflow` with `id` and `name`.

**Not confirmed:** whether Custom Data arrives under a `customData` key or at the top level (n8n reads both, and the test sends both), whether the payload has a contact id of its own (that is why `contactId` is sent as Custom Data), and the `message.type` value for an SMS. n8n accepts `SMS`, `TYPE_SMS`, other values with `SMS` in them and `2`, skips other known channels (email, calls, Facebook and so on) and refuses any other value with a 400 and an alert that names it, so a text is never dropped without notice; check your first live run.

The trade-off: the secret travels in the body, and n8n saves the body with every run, so anyone who can open the n8n executions can read the secret. With the Custom Webhook's Bearer Token, n8n hides the `Authorization` header in saved runs (the test checks both).

## First live test

1. Call the business line from your own phone and let it ring out. In workflow A's execution history, check the tag, the opportunity in New leads and the text. In n8n, check the run and the note on the contact.
2. Reply "Hi, my kitchen sink is clogged". Workflow B runs, and the assistant answers within a few seconds.
3. Reply "Can someone come tomorrow?", then pick a time. The appointment appears on the calendar, the opportunity moves to Booked and Slack gets an alert.
4. From a second phone: call, then reply "EMERGENCY". The emergency text comes back and Slack gets an alert with a channel mention.
5. From a third phone: call, then reply "STOP". SMS DND turns on and nothing more is sent.
6. If you use quiet hours: during them, reply "water is coming through the ceiling" from a test contact that has the `missed-call` tag. The emergency text and the Slack alert come at once; an ordinary question waits until quiet hours end.

## What was checked, and where

| Fact | Status | Source (help.gohighlevel.com article) |
|---|---|---|
| Trigger name "Call Details" (Events) | Confirmed | 48001212511 |
| Filters Call Direction (Incoming), Call Status (busy, voicemail, no-answer, several at once), In Phone Number | Confirmed | 48001212511 |
| "no-answer" versus "not answered" spelling | Docs disagree | 48001212511 and 155000002552 |
| A missed call can show as "completed" without Call Connect | Confirmed | 48001181825 |
| Built-in Missed Call Text Back and where to turn it off | Confirmed | 48001239140 |
| Customer Replied filters Reply Channel and Has Tag | Confirmed | 155000002677, 48001156789 |
| Customer Replied fires for replies to texts sent through the API | Not confirmed | 155000002292 says "any connected channel"; 155000002677 (rechecked October 27, 2026) says it "starts a workflow when a contact replies to a message" and names no exception |
| If/Else conditions on the current day of the week and hour | Not confirmed in current docs (rechecked October 27, 2026) | 48001180266 (now 404), 155000002471 |
| Which time zone If/Else time conditions use | Not confirmed (the workflow Timezone setting covers "time-based actions"; conditions are not mentioned) | 48001239875 |
| Add Contact Tag, Remove Contact Tag | Confirmed | 155000003111, 155000002294 |
| Create/Update Opportunity: Pipeline, Stage, Status, Allow Opportunity to Move | Confirmed | 155000005062 |
| Create/Update Opportunity: Opportunity name and source fields | Not confirmed (own article is 404) | 155000004752 (Create Opportunity), 155000004753 (Update Opportunity) |
| Send SMS fields, automatic STOP line on the first SMS | Confirmed | 155000002474, 48001213941 |
| Custom values and the `{{custom_values.key}}` format | Confirmed | 155000008770, 155000004705 |
| Merge fields `{{contact.id}}`, `{{location.id}}`, `{{contact.name}}`, `{{message.body}}` | Confirmed | 48001078171 |
| Custom Webhook: Event modes, Bearer Token, merge field picker | Confirmed | 155000003305 |
| Custom Webhook simple POST content type | Not confirmed | 155000003305 |
| Custom Webhook premium pricing | Confirmed | 155000005678 |
| Standard Webhook payload fields | Confirmed in part (see above) | 155000003299 |
| Allow Re-entry, Time Window, workflow Timezone (Account or Contact) | Confirmed | 48001239875 |
| STOP replies set DND by themselves | Confirmed | 48001214849 |
