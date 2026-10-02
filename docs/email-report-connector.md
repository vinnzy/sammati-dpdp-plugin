# Email-report connector: backend spec

The plugin's closing step offers to email the report as a PDF and, separately, to let Sammati contact the user. This needs a small remote MCP server, because a skill cannot send email or record consent by itself. The plugin side is written and waits on this branch. It does nothing until the two tools below exist.

## Tools

### `get_report_notice()`

Returns the current privacy notice, so the wording lives in one place and is versioned with the consent record. The plugin shows it verbatim and never hard-codes legal text.

```json
{
  "notice_version": "string",
  "notice_text": "string",
  "purposes": [
    { "id": "send_report", "label": "Email me this report as a PDF", "required_for_report": true },
    { "id": "contact", "label": "A Sammati specialist may contact me about closing these gaps", "required_for_report": false }
  ]
}
```

### `send_report(input)`

```json
{
  "email": "string",
  "name": "string, optional",
  "organisation": "string, optional",
  "notice_version": "string, from get_report_notice",
  "consents": { "send_report": true, "contact": false },
  "result": {
    "mode": "pulse | complete",
    "total": 18, "max": 30,
    "sections": [{ "number": 1, "title": "string", "score": 1, "max": 2 }],
    "answers": [{ "question": 3, "answer": "yes | partial | no | unsure" }],
    "gaps": [{ "question": 31, "next_step": "string" }]
  }
}
```

Returns `{ "status": "sent | failed", "reference": "string" }`.

## What the server must do

1. Reject the call unless `consents.send_report` is true and `notice_version` is the current or a still-valid version.
2. Record each consent decision (granted or not) in the same consent store the sammati.io forms use: purpose, notice version, timestamp, and the channel as `claude-plugin`. This is the legally relevant record.
3. Build the PDF on the server from `result`. Claude cannot attach files to an email.
4. Send it with Resend from a Sammati address, with a link to withdraw consent.
5. If `consents.contact` is true, create the lead, but treat the person as contactable only after they confirm. A link in the same email (double opt-in) works, and it stops someone entering another person's address.
6. Return a reference the plugin can quote to the user.

## Abuse and privacy controls

- The tools take no login, so rate-limit by caller and by email address, and cap one email per address per day. Otherwise it is an open mail relay.
- Store only what is needed for the report and the lead. Set a retention period and state it in the notice.
- Never log pasted documents. The plugin doesn't send them.

## Wiring it into the plugin

When the server is live, add `.mcp.json` at the plugin root with its URL, bump the version, update the README's data section if anything changed, and submit the server to the directory as its own MCP connector as well as the plugin. Anthropic's directory scan flags data sent to undisclosed destinations, so the README must keep describing this flow accurately.

## Open questions

- Which existing service exposes the consent store, and what does it expect from a new channel?
- The directory has authentication requirements for connectors. Check whether an unauthenticated connector like this one is allowed before building.
