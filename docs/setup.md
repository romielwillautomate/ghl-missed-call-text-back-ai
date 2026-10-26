# Setup: GoHighLevel, Claude, Slack and n8n

You need self-hosted n8n reachable over HTTPS, a GoHighLevel sub-account with a phone number that can send SMS, a calendar and a pipeline, an Anthropic API key and a Slack incoming webhook. Plan about 45 minutes: 15 for the keys and n8n, 30 for the two GoHighLevel workflows.

The order matters: n8n first (so its URL exists), then the GoHighLevel workflows that call it.

## 1. GoHighLevel keys and IDs

### Private Integration token

1. In the sub-account, go to **Settings → Private Integrations → Create new Integration** (if Private Integrations is missing, it may need to be turned on under Labs).
2. Give it a name such as "n8n missed call assistant" and these scopes:

   | Scope | Used for |
   |---|---|
   | `contacts.readonly` | Loading the contact (tags, DND, name) |
   | `contacts.write` | Adding tags and notes, and setting DND after an opt out |
   | `conversations.readonly` | Finding the contact's conversation |
   | `conversations/message.readonly` | Reading the last messages for Claude |
   | `conversations/message.write` | Sending the SMS replies |
   | `calendars.readonly` | Free slots |
   | `calendars/events.write` | Booking the appointment |
   | `opportunities.readonly` | Finding the contact's opportunity |
   | `opportunities.write` | Moving it to the booked stage, or creating one |

3. Copy the token (it starts with `pit-` and is shown once) into `GHL_API_KEY`. HighLevel recommends rotating it every 90 days.

### IDs

- **Location ID** (`GHL_LOCATION_ID`): **Settings → Business Profile**.
- **Calendar ID** (`GHL_CALENDAR_ID`): open the calendar's settings; the ID is in the page URL. Or list the calendars (calendar calls use `Version: 2021-04-15`):

  ```bash
  curl "https://services.leadconnectorhq.com/calendars/?locationId=YOUR_LOCATION_ID" \
    -H "Authorization: Bearer pit-your-token" -H "Version: 2021-04-15"
  ```

  The workflow books with `toNotify` on, so the calendar's own notifications run as for any booking. If the calendar sends its own SMS confirmation, customers get that one and the assistant's; turn one of them off. Use a calendar that assigns the team member by itself (a personal calendar or round robin), because the workflow does not pick one. Set `APPOINTMENT_MINUTES` to the calendar's own slot length: the workflow books from the start of a free slot for that many minutes, so a 60 minute visit on a calendar with 30 minute slots would cover the next slot too, and that slot could still be offered to someone else.

- **Pipeline and stage IDs** (`GHL_PIPELINE_ID`, `GHL_BOOKED_STAGE_ID`): create the pipeline first (for example "Service requests" with the stages "New leads" and "Booked"), then:

  ```bash
  curl "https://services.leadconnectorhq.com/opportunities/pipelines?locationId=YOUR_LOCATION_ID" \
    -H "Authorization: Bearer pit-your-token" -H "Version: 2021-07-28"
  ```

  Copy the pipeline's `id` and the `id` of the Booked stage. Workflow A in GoHighLevel opens the opportunity in New leads; n8n moves it to Booked when the customer books. If the contact has no open opportunity in the pipeline, n8n reopens a lost or abandoned one, and otherwise creates a new one. A won opportunity is a past job and is never reopened. Creating a second opportunity for the same contact only works if **Allow Multiple Opportunities per Contact** is on for the sub-account; if it is off, GoHighLevel refuses it, the customer is still booked and told so, and the run fails with a note to add the opportunity by hand.

Only text people who contacted you or agreed to hear from you. In the US, sending SMS through a GoHighLevel number also needs A2P 10DLC registration.

## 2. Anthropic

Create a key at **console.anthropic.com → API keys** and put it in `ANTHROPIC_API_KEY`. The default model is `claude-sonnet-5`; `claude-haiku-4-5` is cheaper. The assistant forces a tool call with thinking turned off, so `claude-opus-5-5` and `claude-fable-5-1` do not work (they refuse a forced tool call), and neither do the other Fable and Mythos models (they refuse thinking turned off). With one of them set, every call is refused with a settings error that names the problem.

## 3. Slack

Create an incoming webhook for a private channel (**api.slack.com/apps → Create New App → Incoming Webhooks**) and put its URL in `SLACK_WEBHOOK_URL`. Bookings, hand offs and emergencies go there, and so do the error alerts. It is required: without it, an emergency would reach nobody. `SLACK_EMERGENCY_MENTION` is put in front of emergency alerts so the channel gets pinged. Not set, it is `<!here>`; set to an empty value, there is no mention.

## 4. n8n

1. Copy `.env.example` into your n8n environment, fill in the values and restart n8n. The nodes read them with `$env`, so self-hosted n8n needs `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. (n8n Cloud does not allow `$env`. There, replace `$env.` with `$vars.` and create the same names under Variables.)
2. Import `workflow/error-alert-slack.json` and **activate it**. n8n 2.x only runs an error workflow that is active; an inactive one is skipped without any alert.
3. Import `workflow/ghl-missed-call-text-back-ai.json`. In its settings, choose the error alert workflow as its **Error Workflow**.
4. Activate it. It listens on `https://YOUR-N8N-DOMAIN/webhook/ghl-missed-call-text-back`.

Check it before building the GoHighLevel side. A request without the secret must get a 401:

```bash
curl -i -X POST "https://YOUR-N8N-DOMAIN/webhook/ghl-missed-call-text-back" -H "Content-Type: application/json" -d '{}'
```

And a request with the secret but no event must get a 400 that names the problem (this one also fails the run, so Slack gets an error alert, which proves the error workflow works):

```bash
curl -i -X POST "https://YOUR-N8N-DOMAIN/webhook/ghl-missed-call-text-back" \
  -H "Authorization: Bearer YOUR_GHL_WEBHOOK_SECRET" -H "Content-Type: application/json" -d '{}'
```

A 500 instead lists every setting that is missing or wrong.

**Quiet hours** need one more thing: when they end, a message that waited is posted back to the workflow's own webhook URL from inside n8n. If n8n cannot reach its public URL from its own network (some proxies and firewalls block that), set `SELF_WEBHOOK_URL` to an address it can reach, for example `http://localhost:5678/webhook/ghl-missed-call-text-back`. Waiting messages survive an n8n restart, because n8n stores waiting runs in its database.

## 5. GoHighLevel workflows

Build the two native workflows from [ghl/native-workflows.md](../ghl/native-workflows.md): workflow A (Call Details trigger, tags, opportunity, the missed call text, Custom Webhook to n8n) and workflow B (Customer Replied trigger, Custom Webhook to n8n). Both use the URL above and `GHL_WEBHOOK_SECRET` as the Bearer token.

Treat `GHL_WEBHOOK_SECRET` like a password: anyone who has it can make the assistant text your contacts. If it leaks, change it in n8n and in both GoHighLevel workflows.

## 6. First live test

Follow "First live test" at the end of [ghl/native-workflows.md](../ghl/native-workflows.md): a missed call, a reply, a booking, an emergency and a STOP, each from your own phones.

## Tags

With the default `TAG_PREFIX`:

| Tag | Added by | Meaning |
|---|---|---|
| `missed-call` | Workflow A | In the missed call flow. Workflow B only forwards texts from contacts with this tag. Remove it to stop the assistant for a contact |
| `missed-call-replied` | n8n | Texted back at least once |
| `missed-call-booked` | n8n | Booked by text. The assistant stops replying and your team takes over |
| `missed-call-needs-human` | n8n | Needs a person: they asked, were upset, asked something the facts do not cover, hit the reply cap, or Claude failed. The assistant stops replying |
| `missed-call-emergency` | n8n | Reported a possible emergency. The team was alerted. The assistant stops replying, but further emergency words still alert the team |
| `missed-call-opted-out` | n8n | Texted STOP or asked to stop. SMS DND is on |
| `missed-call-done` | n8n | Said they do not need help anymore. The assistant stops replying |

Workflow A removes `booked`, `needs-human`, `emergency` and `done` when the contact calls again, so a new missed call starts a new conversation. `opted-out` is never removed by the workflows.

## Troubleshooting

Open the workflow's **Executions** list in n8n:

| You see | Likely cause |
|---|---|
| Run stopped at "Stop: Bad Secret" | The Bearer token (or `secret` Custom Data item) does not match `GHL_WEBHOOK_SECRET`, or `GHL_WEBHOOK_SECRET` is empty |
| Run stopped at "Stop: Bad Settings" | A setting is missing or invalid. The message lists every problem |
| Run stopped at "Stop: Bad Request" | The webhook body has no `event`, an unknown one, or no `contactId`; the message type is one the workflow does not know (the message names it); or the call came from another location than `GHL_LOCATION_ID` (fix the setting, or remove the webhook action from that sub-account). Check the body fields in the GoHighLevel workflow |
| A run failed but Slack stayed quiet | The error alert workflow is not active, is not chosen as the Error Workflow, or `SLACK_WEBHOOK_URL` is wrong |
| Run ended at "Route Event" | Ignored on purpose: not an inbound SMS, or an empty message |
| Run ended at "Route Message" | Ignored on purpose. The output of "Build Context" says why (repeat delivery, no `missed-call` tag, opted out, booked, with a person) |
| Run ended at "Replay Held Message" | The message arrived during quiet hours and was not an emergency or an opt out. It waited and was handed back as a new run when quiet hours ended |
| Run stopped at "Stop: Held Message Lost" | The waiting run could not reach the webhook. Set `SELF_WEBHOOK_URL` and answer the customer by hand |
| Run stopped at "Stop: Contact Lookup Failed" | `GHL_API_KEY` scopes (`contacts.readonly`) or a contact that was deleted |
| "without the conversation history" | The customer was answered, but the conversation could not be loaded. Check the `conversations.readonly` and `conversations/message.readonly` scopes |
| Run stopped at "Stop: Opt Out Failed" | GoHighLevel did not accept the DND update. Set SMS DND on that contact by hand right away |
| Run stopped at "Stop: Alert Not Delivered" | Slack refused the alert. The run message holds the whole alert, so read it there |
| Run stopped at "Stop: Missed Call Note Failed" | The note could not be saved (`contacts.write` scope). The caller still got the text from GoHighLevel |
| Every customer goes to a person after a few texts | `MAX_AI_REPLIES` is low, or the conversations are long. 6 is the default |
