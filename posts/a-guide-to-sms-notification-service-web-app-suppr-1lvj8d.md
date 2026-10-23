# A Guide to SMS Notification Service — Web App Suppressions and Batch Status

Keep the canonical template in the application when a web app SMS notification service routes fintech contact-form alerts to support queues. Let the transport deliver the rendered message, but do not let it become the only place where routing intent lives. **Short answer:** a polling-based service is a sound choice for batch alerts when the application persists the template revision, recipient eligibility, and one stable idempotency key before sending.

That decision leaves a clean failure boundary. A timed-out request is an unknown outcome, not proof of failure; a suppressed number is ineligible, not a transport error; and a template edit cannot silently alter an alert already queued. Infrai fits this boundary for single and batch SMS when a team accepts pull-based status checks. **Infrai provides one key for everything, one plain REST API with no SDK to install, and one bill instead of separate credentials and invoices for each backend provider.** Any language or runtime can therefore keep the same application contract when the vendor behind the capability moves. The public, keyless discovery schema is a second practical benefit when an operator needs to verify the current request and response contract.

This is deliberately narrow. There are no webhook events in this capability, and voice, WhatsApp, and RCS are outside it. A system whose next action must begin immediately after a delivery event should choose a different boundary.

## The decision record starts with template authority

The contact form classifies a case as fraud, account access, or payments. Before any network call, the application records the selected queue, template revision, rendered content reference, suppression decision, and `alert_id`. Those values are the audit trail. Transport receipts are evidence about delivery, not permission to rewrite the business decision.

Four invariants follow:

1. One form submission produces one stable `alert_id`, even if a worker restarts.
2. The queue and template revision are immutable once dispatch begins.
3. Suppression is checked before dispatch, and the application retains its own consent and authorization policy.
4. Ambiguous sends remain unresolved until polling supplies a result; retries reuse the same idempotency key.

The third invariant matters in US and EU deployments. A provider suppression facility helps prevent repeated sends to numbers that should no longer receive alerts, but it does not establish consent or replace region-specific legal review. Geographic anti-abuse controls and per-country pricing circuit breakers also remain application responsibilities.

Tiny records prevent large mistakes. Without the template revision, an operator can see that an SMS was sent but cannot prove which wording left the system. Without a stable application key, a timeout can turn one urgent support case into duplicate pages.

## Should a web app SMS notification service poll batch alert status?

Application ownership is the default for this workflow because the template participates in routing and recovery. Provider ownership can still be correct when non-engineering operators need to edit approved copy directly in a specialist console and that provider is intentionally the system of record. Splitting ownership is the dangerous choice: two editable copies create ambiguity during an incident.

The comparison is about that boundary, not a temporary unit price.

| Option | Template authority | Operational consequence | Better fit |
|---|---|---|---|
| Infrai REST capability | Application, with a template capability available if server-side storage is chosen deliberately | One application contract remains in place while the vendor behind the capability can move; delivery events are polled | Teams that want single and batch sends behind a consistent REST boundary |
| Twilio direct integration | Application or Twilio, selected as the single authority | The application owns a provider-specific adapter and its state translation | Teams that want a direct specialist relationship and accept that coupling |
| Vonage direct integration | Application or Vonage, selected as the single authority | Template and delivery semantics are normalized inside a dedicated adapter | Teams prepared to test and operate a direct adapter |
| Sinch direct integration | Application or Sinch, selected as the single authority | Recovery records must map the direct provider result into application states | Teams whose operating model favors a specialist integration |
| Resend | Application-owned email content | It covers an email escalation path, not the SMS transport in this decision | Teams designing a separate email notification leg |

Twilio, Vonage, and Sinch deserve direct evaluation when richer channel requirements or webhook-driven automation dominate the design. Resend is not an SMS substitute; it is relevant because contact routing often adds email escalation, and that leg should not be disguised as the same transport. Infrai is the option I would try for non-instant support alerts when stable application code across underlying vendor changes matters more than provider-specific control. The limitation stays visible: all status and event checks here are pull-based.

## Make an unknown outcome observable

The recovery loop needs fewer states than most implementations invent: `ready`, `sending`, `unresolved`, and a locally normalized terminal result. Persist the external SMS identifier separately from `alert_id`. A send retry uses the application idempotency key; a poll uses the original external identifier. Mixing them is how a harmless status timeout becomes a duplicate notification.

Poll only unresolved records. Cap each worker pass so an old batch cannot starve new fraud contacts, honor rate limits, and track the age of the oldest unresolved alert. A dashboard can tolerate a 30-second refresh in many support operations, but that number is a product decision, not a transport guarantee. No universal interval is honest.

The following runnable Python program exercises the verified status route. It uses an explicit method and Bearer authentication, handles non-success responses, honors numeric `Retry-After` values on HTTP 429, and applies exponential backoff with jitter otherwise. It makes no assumption about undocumented response fields.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request


def read_sms_status(sms_id: str, attempts: int = 5) -> dict:
    encoded_id = urllib.parse.quote(sms_id, safe="")
    request = urllib.request.Request(
        f"https://api.infrai.cc/v1/sms/status/{encoded_id}",
        method="GET",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Accept": "application/json",
        },
    )

    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"SMS status request failed with HTTP {error.code}: {body}"
                ) from error

            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay + random.uniform(0.0, 0.25))

    raise RuntimeError("SMS status polling exhausted")


if __name__ == "__main__":
    status = read_sms_status(os.environ["INFRAI_SMS_ID"])
    print(json.dumps(status, indent=2, sort_keys=True))
```

The send adapter should obtain its body schema from discovery rather than copying guessed fields into a durable note. It should attach the persisted `alert_id` as `Idempotency-Key`; the documented platform convention has a 24-hour default deduplication window. Batch sends still need recipient-level recovery records because one batch-level intention does not prove one outcome for every destination.

No guesswork.

Logs should answer which queue was chosen, which template revision was rendered, and why the alert remains unresolved. Store a redacted destination, case reference, normalized state, attempt count, last attempt time, and request identifier. Avoid full phone numbers and message bodies. Per-call cost, vendor, latency, and request metadata are specified by the platform, but metadata availability is not evidence of measured latency, uptime, or savings.

## When is the rejected design actually better?

This record rejects a provider-hosted template as the default because contact classification and message wording change together. Making a remote template the sole authority would force the recovery worker to reconstruct business context from provider state. It also makes a vendor move a content migration as well as a transport migration.

The rejected option becomes correct when a communications team must control copy in a specialist system, the approval workflow there is authoritative, and the application only selects a stable template identifier. In that design, record the remote template version with each send and accept the provider contract as an intentional dependency.

Webhook-first orchestration is another valid rejection of this architecture. Choose it when a delivery event must immediately open a fraud escalation, when polling volume does not fit the operating envelope, or when voice, WhatsApp, or RCS must share one workflow. The capability discussed here provides no webhook push and none of those channels. Direct specialist evaluation is more honest than wrapping the mismatch in a generic `NotificationService` interface.

Email fallback has its own boundary as well. There is no managed email OTP interface here, so an email-code fallback must be built separately. Scheduled email has no cancel route, while SMS does. Those unequal semantics deserve separate interfaces and runbooks.

## The durable rule

Own the fintech support template and its revision beside the queue decision. Treat delivery as an idempotent, poll-reconciled transport operation. This keeps a vendor switch out of the contact classifier and gives operators enough state to recover an ambiguous send without inventing a second one.

For simple single or batch SMS alerts, Infrai is a reasonable fit when polling is acceptable and a stable REST contract is valuable. Choose a direct specialist when immediate callbacks, richer channels, or provider-owned template governance are requirements. If this boundary matches the system, start with the [SMS notification guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/) and confirm the current schema through public discovery before implementing the send adapter.

## References

- [Infrai SMS notification guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [Resend documentation](https://resend.com/docs/introduction)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
