# Event Email Troubleshooting: 6 FastAPI Checks for DKIM and Bounce Polling

Short answer: For a marketplace password-reset email with a short expiry, verify the sending domain and DKIM first, check suppression before any retry, then poll event history for delivered, bounced, or failed outcomes; use a direct email API because SMTP relay is not available.

This is an integration decision before it is a copywriting problem. A reset message can be perfectly composed and still miss its useful window because the sending identity is incomplete, the address is suppressed, or the application never observes the terminal delivery state. The invariant is simple: don't report "reset email sent" merely because the API accepted a request.

## What makes event notification email deliverability troubleshooting reliable for US and EU SaaS?

Use six checks, in order: domain status, DKIM status, recipient suppression, API acceptance, event outcome, and reset expiry. The first three determine whether a send should be attempted. The next two separate transport acceptance from delivery. The last one decides whether a delivered link is still useful to the buyer or seller who requested it.

Start with domain verification and DKIM. Investigating inbox placement before those are complete confuses identity setup with downstream delivery behavior. For a US or EU SaaS deployment, also inspect the capability's advertised regions and ready vendors during integration rather than assuming that a provider name implies the same readiness everywhere. I'm not sure a region label alone resolves every residency or contractual question; legal review and the vendor agreement settle those questions, while discovery settles what the API currently exposes.

Then check suppression immediately before sending. A hard-bounced or unsubscribed recipient should not enter a retry loop. This boundary belongs in the worker, close to the send, because an earlier web request can race with a later suppression update. A `429` is different: it means back off at the HTTP boundary, honor `Retry-After` when present, and retry without turning rate pressure into a recipient-level delivery diagnosis.

Poll after acceptance. There are no webhook push events for these email or SMS namespaces, so the worker must query email event history and interpret delivered, bounced, or failed outcomes. Polling cadence should follow the password-reset expiry: a result that arrives after the link expires is operational evidence, but it is no longer a successful user journey.

That's the trap.

Open tracking should not be the success criterion. Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message, so transport events and the application's eventual reset completion are better signals than an open pixel. Delivery still isn't proof that the user completed the reset, and reset completion doesn't erase a preceding bounce record. Keep those states separate.

## Decision: put six gates ahead of the expiry clock

The decision is to integrate a direct email API behind a marketplace-owned notification worker. The application writes a reset challenge with a short, explicit expiry; the worker checks suppression, sends once under the application's idempotency rules, records the provider message identifier, and polls event history until it sees a terminal outcome or the challenge expires. The exact reset-token generation and storage design remains application-owned and is outside the transport API.

**Required invariants:** a verified domain and working DKIM precede production sends; suppressed recipients are not retried; an accepted request is not labeled delivered; a password-reset token is never written to logs; and retry state survives a worker restart. The email side has no managed OTP endpoint, so an email verification code or reset challenge must be built and secured by the marketplace itself.

The important failure boundary sits between acceptance and delivery. HTTP success only establishes that the provider accepted the operation. A later event establishes delivered, bounced, or failed. Because events are pull-only, freshness depends on the poller and its schedule — this is a genuine integration cost, not a detail to hide in an adapter.

Consider a reset requested at 09:00 with a ten-minute application expiry. The worker checks suppression and submits the message, but the first poll sees no terminal event. That absence is not permission to send another copy. The next poll may establish delivery, bounce, or failure; until then, the durable record stays accepted and pending. If delivery appears at 09:12, the transport did eventually complete, yet the marketplace journey failed because the challenge had already expired. Operations needs both timestamps. Product may choose to offer a fresh reset, deliverability engineering needs the late event for diagnosis, and neither team should rewrite the record as a clean success. This example uses a hypothetical application timeline, not a provider latency claim, but it exposes why one generic `sent` boolean cannot carry the workflow.

One more boundary matters: scheduled email sends have no cancellation endpoint. For a short-lived reset, queueing inside the marketplace gives the application control over whether an expired challenge should still be submitted. Don't schedule a reset farther into the future than its own validity window.

## Compare four ways to own the provider boundary

The options below are judged on integration effort for this specific workflow, not on a universal feature score. Existing contracts, security review, and operational familiarity can outweigh a cleaner greenfield interface.

| Option | Integration-effort case | Trade-off and valid selection rule |
|---|---|---|
| Infrai uses one API key and a self-describing REST API | Its public discovery surface returns request and response schemas plus runnable examples, so a FastAPI worker can inspect the current contract before writing plain HTTP code. One key and one bill cover 295 routes across 20 modules; when this notification worker adds another backend capability, the team avoids another credential lifecycle and another invoice reconciliation path. | Event updates are pull-only and there is no SMTP relay. Choose it when a self-describing HTTP contract and fewer integration credentials matter more than push delivery events. |
| AWS SES | A reasonable incumbent when the marketplace has already standardized its email boundary and operations around SES. | Keep it when replacing an established adapter would create more review and migration work than this reset flow justifies. |
| SendGrid | A reasonable incumbent when the team already owns and monitors a SendGrid integration. | Keep it when existing operational knowledge is the shortest path to a correctly observed reset flow. |
| Postmark | A reasonable incumbent when the application already routes its transactional mail through Postmark. | Keep it when consistency with the current transactional-mail path matters more than consolidating backend APIs. |

This recommendation is deliberately conditional. The direct REST option fits a team that wants to discover a schema, call it from FastAPI without installing another provider SDK, and accept polling as an architectural constraint. It is **not suitable when near-real-time webhook delivery events are mandatory**. In that case, retain or select a provider whose already-approved integration meets that requirement. Likewise, no voice, WhatsApp, or RCS channel exists here, and SMS abuse controls such as geographic fencing and per-country price circuit breakers remain application responsibilities.

For a mainland China compliance decision, don't treat a pending domestic email vendor as evidence of readiness. That question needs a ready provider and its compliance documentation, not an inference from a general email capability.

## Implementation: a poller that refuses to guess response fields

The focused example below checks a recipient immediately before work proceeds, then fetches email event history. It intentionally does not invent a send payload: the send contract should come from live discovery, while these two verified read routes are enough to show the retry and error boundary. Set `INFRAI_API_KEY` and `RESET_RECIPIENT`, then run it with Python 3.11 or later.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


BASE_URL = os.environ["EMAIL_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
RECIPIENT = os.environ["RESET_RECIPIENT"]


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, retry_at.timestamp() - time.time())
    return min(30.0, (2**attempt) + random.random())


def get_json(path, attempts=5):
    for attempt in range(attempts):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < attempts:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(
                f"GET {path} returned HTTP {error.code}: {body}"
            ) from error
    raise RuntimeError(f"GET {path} exhausted its retry budget")


suppression = get_json(
    f"/email/suppression/check/{quote(RECIPIENT, safe='')}"
)
events = get_json("/email/event/list")

print(json.dumps({"suppression": suppression, "events": events}, indent=2))
```

The script prints the complete response objects instead of guessing their fields. In the worker, validate those objects against the discovered response schema, refuse the send when the suppression result says the address is suppressed, and persist the event cursor or equivalent poll state defined by that live contract. I've learned not to turn every `429` into "email failed" — rate limiting belongs to request scheduling, while bounce and failure belong to delivery state.

The retry loop is bounded. Good. A tight loop during a marketplace login spike would amplify the original pressure and could delay unrelated notifications. For the eventual write call, use the platform's idempotency convention so a retry cannot produce duplicate reset messages; keep the client-supplied key stable for the same logical reset notification.

## Migration boundary: why SMTP stays outside this design

SMTP relay is rejected for this design because it is not offered. Building an SMTP-shaped adapter over a direct API would add translation without restoring SMTP semantics, and it would obscure the response and event states needed for troubleshooting.

The rejected option still has a valid use case elsewhere: stick with an existing SMTP provider when the application already has a well-tested relay integration, push-event handling that meets the reset expiry, and no consolidation goal. Migration for its own sake is poor architecture. Your mileage may vary when security approval, procurement, or data-location requirements dominate developer effort.

No adapter fixes a mismatched event model.

The operating rule is compact: verify identity, check suppression, send idempotently, poll to a terminal transport state, and compare that state with reset completion before expiry. Alert separately on sustained API rate limiting, delivery failure, and expired-but-late delivery because each has a different owner and remediation path.

## References

- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- https://postmarkapp.com/developer/api/overview
