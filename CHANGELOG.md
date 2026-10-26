# Changelog

## Review fixes (October 27, 2026)

An adversarial review of the workflow, the mocks and the recipe found these problems. Each one is fixed and covered by the test.

- **An emergency in other words waited until morning.** In quiet hours only the emergency word list was checked, so "water is pouring through the ceiling" at 11 PM sat in the Wait node until quiet hours ended, although the README said emergencies are handled at once. Claude now reads every quiet hours message: an emergency or an opt out, in any words, is handled at once, and only the rest waits. Such a message costs a second Claude call when it is answered.
- **Quiet hours ended an hour early on the night the clocks go back.** The end of quiet hours was counted in clock seconds, so on November 1, 2026 in `America/Chicago` a message held before 2 AM would have been answered at 7 AM, inside 21:00 to 08:00 quiet hours (and an hour late in spring). The end is now worked out as a real moment in the business time zone.
- **An opt out could weaken GoHighLevel's own STOP handling.** GoHighLevel sets DND by itself when a contact texts STOP, possibly as a permanent DND that only the contact can lift. The workflow then wrote `SMS: active` over it. An SMS DND that is already active or permanent is now kept as it is.
- **Customer controlled text was in the system prompt.** The contact name and the reason for the call (which Claude wrote from the customer's own words) went into the system prompt. Both now go in the user turn inside `<customer>`, with angle brackets removed, like the rest of the customer text.
- **A wrong `GHL_LOCATION_ID` made the assistant ignore everything without a word.** A call from another location was skipped as a success. It now gets a 400 and an alert.
- **An unknown message type dropped the text quietly.** The standard Webhook action's value for an SMS is not documented; any value the workflow did not know was skipped as "not an SMS". Known other channels are still skipped, and anything else now gets a 400 and an alert that names the value.
- **`SLACK_EMERGENCY_MENTION` had no default.** The docs promised `<!here>`, the code used none. Not set now means `<!here>`; set but empty means no mention. The test leaves it unset to check the default.
- **The test played a duplicate missed call webhook as a second missed call text.** A repeat delivery now resends only the webhook, so the conversation Claude sees has the missed call text once. The test also checked quiet hours writes only for SMS; it now checks tags and notes too.
- **Docs.** The recipe sets the workflow Timezone to Account Timezone and records what the help center says on October 27, 2026 (the If/Else time conditions and Customer Replied for API texts are still not confirmed). The setup guide says to match `APPOINTMENT_MINUTES` to the calendar's slot length, and the README lists what a static data race can really lose and that identical words within 24 hours can be taken as a repeat.

## v1 (October 26, 2026)

First build: a missed call text back for a home services business, with the GoHighLevel side as two native workflows (recipe in `ghl/native-workflows.md`) and the AI and integration side in n8n, plus an error alert workflow and an end to end mock test.

### GoHighLevel side (native workflows, built by hand)

- Workflow A: the Call Details trigger (incoming; busy, no-answer, voicemail) clears the outcome tags of an earlier conversation, adds `missed-call`, opens an opportunity in "New leads", sends a business hours or an after hours text and tells n8n through a Custom Webhook with a Bearer token.
- Workflow B: Customer Replied (SMS, has tag `missed-call`) sends each text to the same n8n webhook.
- The standard Webhook action works too, with the secret as a Custom Data item.

### n8n side

- One webhook for both workflows. The shared secret and every setting are checked first: a wrong secret gets a 401, broken settings a 500 that lists every problem, a body without an event a 400, and each of these fails the run so the error workflow alerts.
- A missed call starts a fresh conversation (reply count, offered times, why they called) and adds a note with the time and which text went out.
- A text from the customer: repeat deliveries are skipped through a message id ledger, the contact and the last 20 messages are loaded, and code decides before any AI: STOP words set SMS DND, emergency words (flooding, burst pipe, no water, gas smell and more) send the emergency text and alert Slack with a channel mention, booked, handed off and opted out contacts are left alone, the reply cap hands off to a person, and quiet hours hold the message.
- Claude (`ANTHROPIC_MODEL`, default `claude-sonnet-5`) decides through a forced, strict tool call: ask why they called, answer from `BUSINESS_FAQ`, offer times, book a picked time, hand off, raise an emergency in other words, honor an opt out in plain words, or close the conversation.
- Open times come from the GoHighLevel free slots API, 3 at a time, one per day where possible. A picked time is booked and the opportunity moves to the booked stage (open or lost reused, won never reopened, created when there is none).
- Every outcome adds tags and a note; bookings, hand offs and emergencies post to Slack. Any failure ends the run as failed with the reason.

### Found and fixed while building and testing

- **A customer picking a time could be handed off instead of booked.** The reply cap counted the text that offered the times, so a customer whose times arrived as the last allowed text was handed to a person when they answered "2". While offered times are open, 2 more texts are now allowed.
- **Quiet hours and n8n's waiting runs.** A probe on n8n 2.19.5 showed that a run resumed from a Wait node keeps the workflow memory it had when it started, and that a wait of 65 seconds or less is not saved to the database at all. A resumed run that wrote memory would overwrite whatever other runs saved in the meantime. Held messages now wait at least 70 seconds and are then posted back to the webhook as a new run, so the resumed run writes nothing.
- **Held runs all resuming at the same second.** Every message held overnight would resume at the end of quiet hours at once, and parallel runs can overwrite each other's memory. Each contact now gets a fixed offset of up to 110 seconds.
- **The webhook secret could end up in saved runs.** n8n hides the `Authorization` header in the saved webhook data, but a node that copies the headers saves them in plain text. The first Code node now passes on only the fields the workflow needs. The test scans every saved run for the secret, the GoHighLevel token and the Anthropic key. With the standard Webhook action the secret travels in the body and is saved; the docs say so.
- **The booking lost the reason for the call.** The appointment description only used the reason from the message that picked the time, which usually has none. It now uses the reason learned earlier in the conversation.
- **"Reply EMERGENCY" was not an emergency word.** The after hours text tells callers to reply EMERGENCY, so the word is now in the default list.
- **HighLevel renamed the trigger.** The "Call Status" help article now opens "Call Details", and the current If/Else article lists no time of day condition. The recipe uses the new name, marks the time conditions as not confirmed and gives a single text fallback.
- **The simple POST mode of the Custom Webhook may send a form body.** Its content type is not documented, so n8n reads JSON and form encoded bodies, and the test sends both, plus the standard Webhook action with Custom Data nested and at the top level.
- **Texts used the first name.** A new caller usually has no name in GoHighLevel yet, so "Hi {{contact.first_name}}" would read "Hi , this is". The missed call texts no longer use it.
- **The test could not see waiting runs.** n8n's public API does not list runs that sit in a Wait node, so the quiet hours test checks that those runs loaded the contact, sent nothing and are not finished yet.
