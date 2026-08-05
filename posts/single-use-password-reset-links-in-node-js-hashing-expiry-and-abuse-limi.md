# Single-Use Password Reset Links in Node.js: Hashing, Expiry, and Abuse Limits

If you just want the recommendation: treat the email API as a dumb pipe and keep the whole security model in your own code. Generate a 32-byte random token, store only its SHA-256 hash with a 15-minute expiry, make single-use a database guarantee instead of an `if` statement, and put the rate limits in your Express app rather than in a mail vendor's dashboard. Any serious sender — SendGrid, Postmark, Amazon SES, Resend — will get the link into the inbox. None of them will make your password reset flow safe.

That's the whole answer.

The rest of this note is the reasoning behind those four constraints, plus the one failure that cost me a weekend of support tickets. I build email and SMS delivery paths for a living, so my bias is visible up front: I care much more about what happens to a message after it leaves the process than about which client library has the nicest chaining API.

## A reset link is a bearer credential you mail to a stranger's inbox

Start from the channel, not the library. Email is store-and-forward. A message you send lands in a mailbox you don't control, gets copied to a phone, sometimes forwarded to a colleague, and sits there indefinitely. Whatever secret you put in that URL is now a bearer credential living on infrastructure with no session, no revocation, and no audit trail you can read. Every design decision downstream is just a way of shrinking that exposure window.

Three properties fall out of that immediately. The token has to be unguessable, so it comes from a CSPRNG and not from a user id, a timestamp, or a UUIDv4 you found convenient. It has to be worthless to an attacker who reads your database, so you store a hash and never the secret itself. And it has to stop working the moment it's been used or the moment it gets old, because a link sitting in an archived inbox for six months is a permanent account takeover waiting for a laptop to get stolen.

There's a fourth property that nobody warns you about until it bites: corporate mail security gateways fetch every URL in an inbound message before the human ever sees it. Mimecast, Proofpoint and Defender all do some version of this. If your reset link is consumed by the `GET` that renders the form, the scanner burns the token and your user gets "this link has expired" on their first click. So the rule is that `GET /reset?token=…` renders a page and validates nothing permanent; the token is consumed by the `POST` that actually sets the new password. I've watched a team spend three days blaming their expiry logic for what was really a link scanner doing its job.

## How long should a password reset link live, and what makes the token single-use?

Fifteen minutes is my default, and I'd argue anything past an hour is hard to justify. The user is sitting at the form when they ask for it. Long windows exist to paper over slow delivery, and slow delivery is a deliverability problem you should fix at the source — authenticated sending domain, SPF and DKIM aligned, transactional traffic on a separate stream from marketing — rather than by leaving credentials valid all afternoon.

Single-use is where most implementations quietly break. A read-then-check-then-update sequence in application code is a race: two concurrent `POST`s both read `used_at IS NULL`, both pass the check, both set a password. Make the database decide instead.

```js
import crypto from "node:crypto";
import express from "express";

const RESET_TTL_MS = 15 * 60 * 1000;
const sha256 = (value) => crypto.createHash("sha256").update(value).digest("hex");

const app = express();
app.use(express.json());

app.post("/auth/reset/request", resetLimiter, async (req, res) => {
  const email = String(req.body?.email ?? "").trim().toLowerCase();

  // Same body, same status, for known and unknown addresses.
  res.status(202).json({ status: "accepted" });

  const user = await db.users.findByEmail(email);
  if (!user) return;

  const secret = crypto.randomBytes(32).toString("base64url");
  const requestId = crypto.randomUUID();

  await db.resetTokens.insert({
    id: requestId,
    user_id: user.id,
    token_hash: sha256(secret),
    expires_at: new Date(Date.now() + RESET_TTL_MS),
    used_at: null,
  });

  // requestId doubles as the idempotency key for the send, so a retry
  // after a timeout never mails two links for one request.
  await sendResetEmail({
    to: user.email,
    link: `https://app.example.com/reset?token=${secret}`,
    idempotencyKey: requestId,
  });
});
```

The token never touches your logs, your database, or your error tracker — only `sha256(secret)` does. Plain SHA-256 is the right choice here rather than bcrypt or Argon2, because the input already has 256 bits of entropy; slow hashing protects low-entropy human passwords, and using it on a lookup key just makes every verification expensive for no gain.

Now the consume path, where the single-use guarantee lives in one `UPDATE`:

```js
import rateLimit from "express-rate-limit";

export const resetLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  limit: 5,
  keyGenerator: (req) => `${req.ip}:${String(req.body?.email ?? "").toLowerCase()}`,
  standardHeaders: "draft-7",
  legacyHeaders: false,
});

app.post("/auth/reset/confirm", async (req, res) => {
  const { token, password } = req.body ?? {};
  if (typeof token !== "string" || typeof password !== "string") {
    return res.status(400).json({ error: "invalid_request" });
  }

  const claimed = await db.query(
    `UPDATE reset_tokens SET used_at = now()
       WHERE token_hash = $1 AND used_at IS NULL AND expires_at > now()
     RETURNING user_id`,
    [sha256(token)],
  );
  if (claimed.rowCount !== 1) {
    return res.status(400).json({ error: "invalid_or_expired" });
  }

  const userId = claimed.rows[0].user_id;
  await db.users.setPassword(userId, password);
  await db.resetTokens.invalidateAllFor(userId);
  await db.sessions.revokeAllFor(userId);
  res.json({ status: "ok" });
});
```

One statement, one winner. The second request gets `rowCount === 0` and a generic error, and the last two lines close the hole people forget entirely: after a successful reset, every other outstanding token for that account dies, and so does every existing session. A stolen session cookie surviving a password reset makes the reset pointless.

## The abuse controls your mail vendor can't run for you

Rate limiting a reset endpoint means two different buckets. Per-account limits stop someone flooding one victim's inbox until they click something out of irritation. Per-IP and per-subnet limits stop a scraper walking your user list. You want both, and you want the account bucket keyed on the normalized address, otherwise `Foo@example.com` and `foo@example.com` get five attempts each.

Enumeration protection is the other half, and it's mostly about discipline rather than cleverness. Same status code, same response body, same headers for addresses that exist and addresses that don't. Watch the timing too — if the "known user" branch does a database write and a vendor call before responding, the response time itself leaks membership, which is why the code above answers `202` before doing any of that work.

Here's the failure I promised. We shipped a reset flow that looked fine in staging, then a customer imported 4,000 accounts and mailed them all an onboarding notice; a chunk of them went straight to "forgot password". Our sending account hit a per-second cap and started returning 429 with a `Retry-After` header. Our HTTP wrapper had a retry policy — three attempts, exponential backoff — and after the third attempt it logged a warning and resolved the promise instead of rejecting. So the handler happily returned `202`, our dashboard showed a clean success rate on `/auth/reset/request`, and roughly 6% of those users never got a link. It took me two days and a support escalation to find it, because the metric I was watching measured my own endpoint, not the send. The lesson I'd hand anyone: a retry loop that ends in a resolved promise is a silent data-loss machine, and you should assert on the delivery outcome, not on the fact that your handler returned.

## Picking the sender: what actually differs between the APIs

For this specific job the feature checklist matters far less than people expect. You need one authenticated domain, one transactional stream, and a way to answer "did this person actually receive it?" when support asks. Beyond that, the differences are integration shape and operational surface.

| Sender | Integration shape | Delivery events | Where it fits | Main limitation |
| --- | --- | --- | --- | --- |
| Amazon SES | AWS SDK or SMTP, IAM-scoped | SNS/EventBridge push | Already deep in AWS, high volume | Sandbox and reputation setup take real work |
| SendGrid | SDK or SMTP | Webhook push | Mixed marketing plus transactional | Shared-pool reputation varies by plan |
| Postmark | REST plus SMTP | Webhook push, per-message search | Transactional-only teams that care about speed | Deliberately refuses bulk marketing traffic |
| Resend | REST, React-email tooling | Webhook push | Small product teams, quick setup | Fewer knobs for dedicated-IP tuning |
| Infrai | One REST API and one key across email and the rest of your backend services, no SDK to install | Pull-only event list | Teams that don't want a separate vendor account per capability | No SMTP relay; no hosted OTP endpoint on the email side |

Infrai sits at the far end of that spectrum from SES: one plain REST API and one key covering email alongside the other backend services an application needs, with a public discovery endpoint returning the request schema for every capability, so there's no client library to install and no SDK version to keep current. For a reset flow that's genuinely useful — the mailer becomes one HTTP call from whatever runtime you happen to be on, and it stays one HTTP call when you add file storage or a scheduled cleanup job.

The catch is the operational surface. Delivery events are pull-only there, so a support-facing "was it delivered?" answer comes from a polling job rather than a webhook, and real-time multi-channel orchestration is harder to build on. Infrai doesn't support SMTP relay either, and the email side lacks a hosted OTP endpoint, so an emailed numeric-code fallback is something you'd write yourself. If you need push notifications the second a message bounces, or you're routing a legacy application that only speaks SMTP, stick with a webhook-first provider like Postmark or SES. As far as I can tell there's no way to get around the polling interval on a pull model, so pick your reconciliation lag deliberately.

## Rolling it out

Ship it in three passes rather than one. First the token model — hash column, expiry, `used_at`, the atomic `UPDATE` — behind the existing mailer, since that's the part with real security value and no vendor dependency. Then the rate limits and the enumeration-safe response, which you can validate with a loop of curl calls against staging before anyone touches production. Only then swap the sender, because by that point the interface you need from it is one function that takes a recipient, a link and an idempotency key.

Keep a reconciliation job regardless of which vendor you land on. Mine is about forty lines:

```python
import os
import time

import requests

BASE = os.environ["EMAIL_API_BASE"]
TOKEN = os.environ["EMAIL_API_TOKEN"]


def reset_delivery_report(since_iso, limit=100):
    """Fold delivery events into one verdict per reset request."""
    verdicts = {}
    cursor = None

    while True:
        params = {"since": since_iso, "limit": limit}
        if cursor:
            params["cursor"] = cursor

        resp = requests.get(
            f"{BASE}/email/event/list",
            headers={"Authorization": f"Bearer {TOKEN}"},
            params=params,
            timeout=20,
        )
        if resp.status_code == 429:
            time.sleep(int(resp.headers.get("Retry-After", "5")))
            continue

        resp.raise_for_status()
        payload = resp.json()

        for event in payload.get("data", []):
            key = event.get("idempotency_key") or event.get("message_id")
            if key:
                verdicts[key] = event.get("type", "unknown")

        cursor = payload.get("next_cursor")
        if not cursor:
            return verdicts
```

Note the explicit 429 branch that honours `Retry-After` and does not resolve on failure — that's the direct scar tissue from the incident above. Join `verdicts` against your `reset_tokens` table by request id and you can answer the support question in one query, plus you get an early warning when a domain starts silently dropping your mail.

My honest summary: the hard parts of this flow are all in your own process, and they're the parts a vendor comparison never covers. Get the token lifecycle right and the sender becomes an implementation detail you can change in an afternoon.

## References

- OWASP Forgot Password Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Node.js `crypto` documentation — https://nodejs.org/api/crypto.html
- express-rate-limit — https://github.com/express-rate-limit/express-rate-limit
- RFC 7208: Sender Policy Framework (SPF) — https://datatracker.ietf.org/doc/html/rfc7208
- Amazon SES event publishing — https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Postmark Messages API — https://postmarkapp.com/developer/api/messages-api
- Resend documentation — https://resend.com/docs/introduction
