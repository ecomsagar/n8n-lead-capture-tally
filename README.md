# n8n Lead Capture — Tally → Google Sheets → Telegram

Form submitted → row in Google Sheets in 5 seconds → instant Telegram alert.
No manual data entry, no copy-paste.

![Telegram alert](screenshots/telegram-alert.png)

## The problem

Most small service businesses collect leads through a form, then someone manually
types them into a spreadsheet. That's 1–2 hours a day. Worse, leads sit unseen for
hours while the customer is already talking to a competitor.

## The flow

```
Tally Form → Webhook → Set (normalize) → Google Sheets → Telegram
```

| Node | What it does |
|---|---|
| Webhook | Listens for form submissions (POST) |
| Set / Edit Fields | Flattens Tally's nested JSON into clean fields |
| Google Sheets | Appends a row |
| Telegram | Sends the alert |

## Requirements

- n8n (self-hosted or cloud)
- Tally account (free tier works)
- Google Sheets credential
- Telegram bot token + chat ID

## Setup

1. Import `workflow.json` into n8n
2. Add your own Google Sheets and Telegram credentials
3. Create a sheet with these headers in row 1:
   `Name | Phone | Product | Budget | Time`
4. Activate the workflow and copy the **Production URL** from the Webhook node
5. In Tally → Integrations → Webhooks, paste that URL

Format the phone column as **Plain text** before going live, or Google Sheets will
strip the leading `+` from international numbers.

## Gotchas worth knowing

**Multiple choice fields send IDs, not text.**
Tally sends the selected option's ID inside an array, not the label you see on the
form. To get the readable text, match the ID against the `options` array:

```javascript
{{ $json.body.data.fields
     .find(f => f.label === 'Product').options
     .find(o => o.id === $json.body.data.fields
     .find(f => f.label === 'Product').value[0]).text }}
```

**Webhooks can't be tested in a browser.**
Opening the Production URL in a browser sends a GET request and returns a 404.
The webhook only accepts POST. Test by actually submitting the form.

**Match by label, not by index.**
`fields[0].value` breaks silently if someone reorders the form — no error, just
wrong data in the wrong column. `find(f => f.label === 'Name')` survives reordering.

**Timestamps arrive in UTC.**
Tally's `createdAt` is UTC. Convert it if your reporting depends on local dates.

## Extending this

Add an IF node after the Sheets step to route leads by any condition — budget,
product type, urgency — to different people:

```
Sheets → IF (Budget > 5000) → Telegram (senior rep)
                            → Telegram (junior rep)
```

Convert Budget from string to number first, or the comparison fails silently.

## Swapping the trigger

The Webhook node is the only part tied to Tally. Replace it with a Facebook Lead
Ads, Typeform, or Gmail trigger and adjust the Set node expressions — everything
downstream stays identical. That's the whole point of normalizing early.

---

Part of a 26-week automation build series. This is week 1.
