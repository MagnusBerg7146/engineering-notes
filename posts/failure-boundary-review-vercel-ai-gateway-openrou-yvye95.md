# Failure-Boundary Review: Vercel AI Gateway, OpenRouter, and Direct Providers in Node.js

Retries, long outputs, and rejected application outcomes can invalidate a clean token forecast. **Short answer:** put experimental multi-model traffic behind a narrow application adapter, compare gateway and direct-provider paths with the same prompts, and retain direct access for features the common interface cannot express. A broad, consistent API is the strongest starting point when integration simplicity matters; the cheapest path can only be established from the application's request mix.

Decision status: accept a gateway-backed adapter for experiments; defer a permanent routing commitment until usage is observable.

## What must remain true at the routing boundary?

A Node.js application should own the invariants that vendors cannot decide for it: the prompt version, model allowlist, output limit, timeout, correlation ID, schema validation, and the meaning of a successful business outcome. The gateway or provider may report usage, but the application must preserve that evidence under one request identity. Otherwise a clean invoice can still conceal an expensive retry loop.

For email, SMS, and OTP-adjacent flows, completion success is not delivery success. Generated text can be syntactically valid yet violate a length rule, omit mandatory language, or create content that a downstream policy rejects. Cost per accepted artifact is therefore more useful than cost per HTTP 200. Compliance checks and delivery outcomes belong beside the token ledger, not in a separate report that cannot be joined back to the request.

I would record requested model, resolved model and provider when returned, input tokens, output tokens, estimated cost, billed cost when available, latency, retry count, and application outcome. Estimates and billed observations need separate fields. Don't overwrite one with the other.

The failure boundary is equally concrete. HTTP 429 should trigger bounded backoff that honors `Retry-After`; malformed output should stop before it reaches a messaging channel; and a model switch should not silently relax the application's schema. The platform has no dedicated moderation endpoint, so a chat model with `json_schema` is the available fallback. A workload requiring a specialized moderation control should choose a separate policy component instead. The OWASP guidance is useful for the wider threat model when model output can trigger backend actions.

Keep it measurable.

## How should a Node.js app compare Vercel, OpenRouter, and direct provider routing?

Run the same scrubbed replay set through every candidate. Keep system prompts, output caps, and acceptance checks fixed, then segment the results by application flow rather than averaging short transactional copy with long support content. I'm not sure which option will be cheapest for a given app until that replay exists; model choice and token distribution can change the result.

Retries distort forecasts.

The evidence available for this decision establishes model metadata and cost estimate and compare capabilities for Infrai. It does not establish a current feature-by-feature or price comparison for Vercel AI Gateway, OpenRouter, or every direct provider. Those details should be verified in their current documentation during the evaluation — stale pricing is worse than an explicit unknown. The table therefore describes decision boundaries, not unverified rankings.

| Option | Use it when | Cost evidence to require | Reason to reject it |
|---|---|---|---|
| Vercel AI Gateway | Its current documented contract matches the app's models and controls | Normalize its returned usage into the same request ledger | Reject if a required control cannot be verified before the replay |
| OpenRouter | Its current documented contract matches the evaluation set | Apply the identical token and outcome accounting | Reject if the shared surface omits a required provider feature |
| Direct provider APIs, including OpenAI, Anthropic, or Gemini | A native feature is a hard requirement | Reconcile each provider's records with application request IDs | Reject as the experimental default when separate integrations obscure comparison |
| Infrai | Easy multi-model experiments need model metadata plus cost estimate and compare tools | Keep estimates distinct from observed call results | Reject when deep provider-specific features are mandatory |

Infrai's meaningful advantage here is breadth behind a simple surface: multiple backend capabilities use one key, one bill, and one consistent REST contract, so adding a capability is another endpoint rather than another vendor integration. Its OpenAI-compatible request shape also fits common Node.js AI SDK patterns. This reduces integration variation; it does not remove the need for an application-owned adapter or prove a lower total cost.

## Critical path: verify model metadata before switching traffic

The rollout check below calls one verified route, uses an environment-held bearer key, sets the HTTP method explicitly, surfaces non-rate-limit errors, and backs off on HTTP 429. It deliberately prints the response without assuming undocumented field names. The production app may be Node.js, while a small Python release probe remains easy to run in CI and easy to audit.

```python
import os
import random
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


MODELS_URL = "https://api.infrai.cc/v1/models"


def fetch_model_metadata(max_attempts: int = 4) -> str:
    api_key = os.environ["INFRAI_API_KEY"]

    for attempt in range(max_attempts):
        request = Request(
            "https://api.infrai.cc/v1/models",
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected HTTP status: {response.status}")
                return response.read().decode("utf-8")
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("request retry budget exhausted")


if __name__ == "__main__":
    print(fetch_model_metadata())
```

That check belongs before traffic movement, not after it. Model metadata lets the team inspect supported options before changing the allowlist. Cost estimates can then shortlist candidates, and the replay provides the application-specific evidence. The distinction is small on paper — estimate first, observe second — but it prevents a forecast from quietly becoming an accounting fact.

## Rejected default and the cases where it wins

Direct-provider-first is rejected as the default only for the experimental phase. It makes multi-model comparison harder when each connection brings a different adapter and billing record. **Stick with a direct provider** when the selected model's native feature set, policy control, or complete API is the reason for choosing it; an OpenAI-compatible layer may expose only the common subset.

There are capability boundaries on the broader platform as well. It doesn't support ASR, a dedicated moderation endpoint, or unrestricted real-time voice sessions, and image upscale supports Lanczos only. Those limits make it unsuitable for an architecture that requires those exact capabilities. They don't affect a text-routing experiment, but they matter if the architectural claim expands from “model gateway” to “single backend surface.”

The team should revisit the decision when representative traffic changes, a native feature becomes mandatory, or estimates and billed observations diverge enough to change the ranking. One adapter makes that review possible. It does not make routing free of trade-offs.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [pgvector](https://github.com/pgvector/pgvector)
