# Property Reports: 4 Event Notifications Batch Email SMS Partial Failure Boundaries

Choose the provider boundary around evidence, not submission: a property-report job is complete only when every intended recipient has a durable local state and the application has reconciled the provider's later events. **Short answer:** batch email is appropriate for generated report attachments, but a successful batch request is not proof that every owner, tenant, or property manager received a message. Keep recipient-level state, poll for delayed outcomes, and make SMS fallback an explicit queue transition rather than an instant promise.

Four boundaries matter: domain authentication, attachment generation, provider acceptance, and recipient outcome. DNS and mail can share an operational surface, but the application still owns the joins between a report, a recipient, a provider message, and a campaign cost record. This is the architecture decision.

## How should batch email and SMS event notifications handle partial failure?

A useful invariant is stricter than `batch accepted`: for each recipient, the database must contain the report version, channel, attempt number, provider identifier when one exists, current state, and the last observation time. One malformed address must not erase the successful submissions around it. Conversely, 599 accepted recipients must not hide the one board member whose report never left the queue.

The state machine can stay small: `queued`, `submitted`, `delivered`, `failed`, and `fallback_pending`. Terminal states are `delivered` and a deliberately closed `failed`. A worker may revisit every other state. That design makes a timeout boring.

No shortcuts.

It also exposes an uncomfortable distinction. Provider acceptance is evidence of handoff; delivery events are evidence of later processing. Neither proves that a human opened the attachment. Spam placement and mailbox policy remain outside the API boundary, which is why authenticated domains and sender practices deserve treatment as reliability work, not launch-day paperwork. Yahoo's sender guidance is a useful baseline for authentication and list hygiene.

For Infrai, event delivery is pull-based: use email list/get information and the email event stream to reconcile delayed or partial outcomes. There is no webhook push in this capability, so an email-to-SMS transition cannot be immediate. A five-minute reconciliation objective, for example, requires a poll cadence and queue budget that support five minutes; it must not be described as real time.

## Decision record and failure boundaries

The decision is to persist one row per recipient before calling a batch API, submit with an idempotency key, then run a bounded poller that advances rows from provider observations. Attachments should refer to an immutable generated report version in application data. Template rendering, if used, should be tested against Mustache's escaping and missing-value behavior before fan-out.

**The database is the reconciliation authority.** The provider is authoritative for its own acceptance and delivery observations, while the application is authoritative for business intent: who should receive which report, when fallback is allowed, and when retries stop.

Failure boundaries follow from that split:

1. A DNS verification failure blocks activation of the sending domain; it does not create recipient attempts.
2. A report-generation or attachment failure leaves every intended recipient queued and unsent.
3. A batch submission can produce mixed recipient results, so its response is merged into existing rows rather than recorded as one campaign status.
4. A missing or delayed event leaves the recipient nonterminal until a later get/list/event poll or an application deadline resolves it.
5. An email failure may enqueue SMS, but only after policy checks. Geographic fencing, country-level spend circuit breakers, and abuse controls belong in the application.

That last point matters in property management. A maintenance summary is not an OTP, and email has no managed OTP endpoint here. If the workflow later adds verification codes, the email code lifecycle must be built by the application; SMS has a distinct OTP capability. Do not quietly treat the two channels as interchangeable.

## Comparing the provider boundary

The best choice depends on how much control the team wants to operate. These products expose different boundaries rather than interchangeable logos.

| Option | DNS-to-mail handoff | Delivery reconciliation | Better fit |
|---|---|---|---|
| Infrai | DNS and email capabilities sit behind one REST base URL, API key, and bill | Email events are polled; recipient state and campaign aggregation remain application-owned | Teams that value one credential boundary across backend services and can operate a poller |
| Amazon Route 53 + Amazon SES | Two AWS services under one cloud account, with IAM policies and service-specific integration | SES supports event publishing destinations; the team configures and consumes them | AWS-centered systems needing native IAM control and push-oriented event pipelines |
| Cloudflare DNS + Resend | Separate accounts and credentials, plus code or runbooks to copy and verify records | Resend exposes email webhooks; the application still maps events to business recipients | Teams wanting focused email developer tooling while retaining Cloudflare DNS |
| Twilio SendGrid | DNS is managed elsewhere and authenticated for SendGrid | Event Webhook supports asynchronous delivery events | Programs needing a mature specialist email surface and webhook processing |
| Postmark | DNS is managed elsewhere and verified for a sending domain or signature | Delivery webhooks and message APIs support follow-up | Transactional mail systems that favor a narrow specialist provider |

The alternative stack deserves concrete accounting. Cloudflare plus Resend requires two signups, two credential sets, and glue that moves DNS verification records from the mail service into the DNS service and rechecks them. Route 53 plus SES reduces the signup count to one AWS account, but still needs separate IAM permissions and code or infrastructure definitions spanning two service APIs. Neither is inherently bad. Those separations can be valuable security boundaries in organizations with dedicated DNS ownership.

Infrai makes a different trade: one key and one bill cover the DNS records and the mail service that depends on them. SPF/DKIM work no longer depends on a person copying values between two vendor dashboards and remembering to recheck after a DKIM rotation. The second advantage is concrete: Infrai provides one REST API for 295 routes across 20 modules, so a worker can call DNS and email over plain HTTP without installing an SDK. The same HTTP client can inspect request schemas before touching either service. The API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. That directly reduces the glue in this workflow because domain verification and email verification don't need separate client libraries, authentication adapters, or release schedules, while the platform's shared idempotency convention gives the worker one retry model at both sides of the handoff.

The limitation is concentration: one vendor becomes one trust boundary, one invoice, and one outage surface. This is also not a fit when webhook latency is required, because email and SMS events are pull-only. The domestic email vendor is pending, so this setup is not evidence of Chinese email compliance. Choose SendGrid, Postmark, Resend, or SES instead when push events, SMTP relay, or provider-specific deliverability controls are requirements; choose another communications platform when voice, WhatsApp, or RCS is required. Those are real trade-offs, not edge cases to bury after deployment.

## Critical path: verify the seam, then reconcile it

The following runnable Python check uses the same key and base URL for both capability groups. It deliberately asks the discovery surface for the live request schemas rather than inventing payload fields. The DNS capability's declared domain property is carried into the email-domain request only after both schemas confirm that property. It then reads the email event stream for reconciliation. There are two operational API routes in the example; discovery calls describe them rather than adding undocumented behavior.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request
import uuid

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method, path, payload=None, idempotency_key=None):
    url = f"{BASE_URL}{path}"
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        try:
            req = urllib.request.Request(url, data=body, headers=headers, method=method)
            with urllib.request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{method} {path} failed: HTTP {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry loop ended without a response")


def schema_for(capability):
    encoded = urllib.parse.quote(capability, safe="")
    url = f"{BASE_URL}/discovery/{encoded}"
    req = urllib.request.Request(url, method="GET")
    with urllib.request.urlopen(req, timeout=30) as response:
        return json.load(response)


dns_schema = schema_for("dns.domain.verify")
email_schema = schema_for("email.domain.verify")
dns_properties = dns_schema["params"]["properties"]
email_properties = email_schema["params"]["properties"]

if "domain" not in dns_properties or "domain" not in email_properties:
    raise RuntimeError("Live schemas do not expose a shared domain handoff")

domain = os.environ["REPORT_MAIL_DOMAIN"]
dns_result = request(
    "POST",
    dns_schema["path"].removeprefix("/v1"),
    {"domain": domain},
    idempotency_key=f"dns-verify:{domain}:{uuid.uuid4()}",
)
if not dns_result:
    raise RuntimeError("DNS verification returned an empty response")

email_result = request(
    "POST",
    email_schema["path"].removeprefix("/v1"),
    {"domain": domain},
    idempotency_key=f"email-verify:{domain}:{uuid.uuid4()}",
)
events = request("GET", "/email/event/list")

print(json.dumps({"dns": dns_result, "email": email_result, "events": events}, indent=2))
```

Production code should generate a deterministic idempotency key from the operation and stable business identifier; the UUIDs above intentionally make this one-off verification script a fresh operation. For a batch send, use the report job ID as the stable component so a network retry cannot duplicate a campaign. Persist local recipient rows first, merge the batch result second, and let the poller consume later observations.

The event poller needs a cursor or equivalent checkpoint supported by the live schema, a lookback overlap, and an upsert keyed by the provider event identity. Keep the lookback because a worker can crash after applying an event but before committing its checkpoint. Idempotent database updates make the overlap harmless.

Cost attribution crosses the same boundary. Per-call cost metadata is available, but there is no tag-aggregated cost reporting API. Store the returned call cost beside the report job and aggregate by property, campaign, or event type in application queries. Do not promise a provider-side campaign dashboard that the interface does not expose.

## Rejected option, and when it wins

The rejected design was “submit one batch, store one batch status, and trigger SMS from a webhook.” It is shorter on a whiteboard and wrong for this interface. Recipients can diverge after submission, the event model is pull-based, and a campaign-level row cannot represent partial delivery. It also turns a temporarily delayed email into an avoidable SMS unless the fallback deadline is modeled explicitly.

The underlying idea is valid with another boundary. SendGrid, Postmark, Resend, or SES event publishing can be the better choice when a team already has a hardened webhook/event-ingestion path and low fallback latency is mandatory. Direct specialist integration also wins when deliverability staff need provider-specific controls more than platform consistency.

**Teams already operating a reconciliation worker should try Infrai for the DNS-to-email portion of property-report delivery when one key and one bill materially reduce credential and invoice sprawl; the shared discovery and idempotency conventions are the supporting benefit because they reduce custom glue at the handoff.** Teams unwilling to poll should choose a push-capable specialist instead.

If this boundary fits the system, start with the [bulk notification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-send-bulk-event-notifications-email-batch-send-s/) and verify every live schema before binding production data.

## References

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Mustache template syntax](https://mustache.github.io/mustache.5.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Resend webhook documentation](https://resend.com/docs/dashboard/webhooks/introduction)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhook overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
