# Express Feedback Form with Email Receipts and Low-Rating Alerts (Mailtrap Email API)

A small Node.js feedback app: visitors leave a 1–5 rating and a comment, get a copy of their feedback by email, and low ratings alert the owner with Reply-To set to the submitter, so the owner can answer from their inbox. Built with Express 5, Node.js 24's built-in SQLite and Zod, it sends email through the [Mailtrap Email API](https://mailtrap.io/email-api/?utm_source=github&utm_medium=repo&utm_campaign=express-feedback-receipt) with native `fetch` (no SDK). During development, emails are captured in the [Mailtrap Email Sandbox](https://mailtrap.io/email-sandbox/?utm_source=github&utm_medium=repo&utm_campaign=express-feedback-receipt) instead of being delivered.

## What it does

Share a rating and a little context. You will receive a copy of your feedback for your records.

The email workflow:

- **Receipt** (`feedback.receipt`): the submitter gets a copy of their rating and message.
- **Low-rating alert** (`feedback.alert`): when the rating is at or below `FEEDBACK_ALERT_MAX_RATING` (default 2), `OWNER_EMAIL` gets the feedback with `reply_to` set to the submitter. Leave `OWNER_EMAIL` empty, or set the threshold to `0`, to turn alerts off.

## Technologies

- Express 5
- EJS
- [Mailtrap Email API](https://docs.mailtrap.io/email-api-smtp/overview?utm_source=github&utm_medium=repo&utm_campaign=express-feedback-receipt) for sending and [Mailtrap Email Sandbox](https://docs.mailtrap.io/email-sandbox/overview?utm_source=github&utm_medium=repo&utm_campaign=express-feedback-receipt) for testing, called with native `fetch` (no SDK)
- Node.js 24 and its built-in SQLite module for local persistence
- Zod for input validation

## Run locally

Install Node.js 24 or newer in the 24.x release line and npm. The built-in SQLite API is experimental in some Node 24 releases; no database server or native build tools are required.

```sh
npm ci
cp .env.example .env
# Edit .env: set ADMIN_PASSWORD, APP_URL, owner/recipient settings,
# and email credentials using docs/email-setup.md.
npm run dev
```

The app is served at http://localhost:3000. For operator access, choose a unique password of at least 16 characters. You can generate one with:

```sh
node -e "console.log(require('node:crypto').randomBytes(24).toString('base64url'))"
```

Use that value in `.env`; the operator username is `admin`. See [email configuration](docs/email-setup.md) for the default provider, development setup, and production settings. To explore locally without credentials, set `MAIL_MODE=log` while running `npm run dev`; resulting messages are written to `.data/emails.jsonl` and are **not sent**. Log mode is disabled when `NODE_ENV=production`.

## Try the workflow

1. Open http://localhost:3000 after starting the application.
2. Complete the form with realistic sample values and submit.
3. Check `/admin` for the saved record and email state.
4. Submit a rating of 1 or 2 with `OWNER_EMAIL` set; `/admin` lists a `feedback.alert` email next to the receipt.

Example form values:

```json
{
  "name": "Alex Doe",
  "email": "alex@example.com",
  "rating": "4",
  "message": "Clear and useful. A few more examples would help."
}
```



## Configuration

| Variable | Meaning |
| --- | --- |
| `APP_URL` | Exact public origin; used for email links and cross-origin checks |
| `ADMIN_PASSWORD` | Operator password, at least 16 characters |
| `OWNER_EMAIL` | Recipient of low-rating alerts; leave empty to turn alerts off |
| `FEEDBACK_ALERT_MAX_RATING` | Ratings at or below this value (1–5) alert `OWNER_EMAIL`; default `2`, `0` turns alerts off |
| `REVIEWER_EMAIL` | Fixed reviewer for review requests; otherwise unused |
| `DB_PATH` | SQLite file; defaults to `.data/app.sqlite` |
| `HOST`, `PORT` | Production listener settings where supported by the framework |
| `MAIL_MODE` | `sandbox`, `production`, or local-only `log` |

Email credential variables are documented in [email setup](docs/email-setup.md). Frameworks must be restarted after environment changes. Customize the app's public copy, fields, seeded event details, or menu in `src/core/config.js`; Nuxt also uses `app-definition.json` for its client-visible definition. Keep those two definitions consistent when editing.

## Where the main idea lives

- `src/server.js` (routes and middleware) and `views/index.ejs` (page template).
- `src/core/service.js`: the application's workflow, field validation, and atomic state changes.
- `src/core/store.js`: SQLite records, hashed tokens, rate counters, and a small email outbox.
- `src/core/mail.js`: server-side email transport and environment selection.
- `test/workflow.test.mjs`: workflow and permission edge cases.
- `test/email.test.mjs`: transport configuration tests using local doubles.

## Check and build

```sh
npm test
npm run build
npm start
```

Tests exercise the app's business rules and email configuration without sending external mail. They do not prove production delivery. To check actual sending, configure your own credentials and complete the normal workflow with an address you control.

## Email failures

Saving a valid record and recording its intended email happen in one SQLite transaction. Sending happens afterward. An email failure leaves the record intact and appears in the operator page.

```sh
npm run retry-email
```

This retries pending and known-failed messages. `accepted` means accepted by the provider, not confirmed delivery. `logged` means a local preview only. Timeouts or partial HTTP API results are `unknown`; messages interrupted by a process crash may remain `sending`. These states are not automatically retried because a first send may have succeeded. Check provider logs before reconciling them manually.

## Deployment and limits

Run one Node process behind HTTPS with a persistent writable volume for `DB_PATH`. Run the build first, then start with your configured environment. Set `APP_URL` to the real HTTPS origin; do not trust arbitrary forwarded headers. The operator uses HTTP Basic authentication and must be protected by HTTPS outside localhost. This app is not designed for ephemeral or multi-instance serverless storage.

Form submissions are limited to 12 per client per hour and 3 per recipient per hour. Behind a reverse proxy, clients may share the proxy address; configure infrastructure limits before larger deployment. There is no multi-user operator system, payment processing, distributed job queue, or production email webhook tracker.

## FAQ

### How do I send a feedback receipt email from Express with the Mailtrap Email API?

Send a `POST` with a Bearer token to the Mailtrap Email API. `src/core/mail.js` does this with native `fetch`, a 15-second timeout and a check that every recipient got a message ID:

```js
const response = await fetch('https://send.api.mailtrap.io/api/send', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${process.env.MAILTRAP_PRODUCTION_TOKEN}`
  },
  body: JSON.stringify({
    from: { email: process.env.MAIL_FROM, name: 'Feedback Receipt' },
    to: [{ email: 'alex@example.com' }],
    subject: 'A copy of your feedback',
    text: 'Hello Alex,\nThank you for sharing your feedback.\nRating: 4/5',
    category: 'feedback.receipt'
  }),
  signal: AbortSignal.timeout(15000)
});
const result = await response.json(); // { success: true, message_ids: [...] }
```

### How can the site owner reply to feedback straight from their inbox?

The low-rating alert sets `reply_to` to the submitter's address, while `from` stays on your verified sending domain. Pressing Reply in any mail client answers the person who left the feedback, and the app never sends from an address it doesn't own:

```js
{ to: [{ email: process.env.OWNER_EMAIL }], reply_to: { email: 'alex@example.com' }, category: 'feedback.alert', ... }
```

### How do I test feedback emails without sending them to real people?

Keep `MAIL_MODE=sandbox`. Emails go to `https://sandbox.api.mailtrap.io/api/send/{inbox_id}` and land in the [Mailtrap Email Sandbox](https://mailtrap.io/email-sandbox/?utm_source=github&utm_medium=repo&utm_campaign=express-feedback-receipt), where you can check the receipt and the alert, including the Reply-To header. For offline work, `MAIL_MODE=log` with `npm run dev` writes them to `.data/emails.jsonl`. See [email setup](docs/email-setup.md).

### How do I validate form input in Express with Zod?

`src/core/service.js` builds a Zod schema from the field list in `src/core/config.js`: text is trimmed and length-limited, the email is checked and lowercased, and the rating must be one of the listed options (`1`–`5`). Invalid input is rejected before anything is stored or emailed.

### What happens if an email fails to send?

The feedback and its emails are saved in one SQLite transaction before sending, so a failure never loses a submission. `/admin` shows each email's type and state, and `npm run retry-email` retries failed messages. A failed owner alert doesn't change what the submitter is told about their receipt.

## License

MIT. See [LICENSE](LICENSE).
