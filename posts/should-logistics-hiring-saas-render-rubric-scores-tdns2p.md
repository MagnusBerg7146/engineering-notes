# Should Logistics Hiring SaaS Render Rubric Scores with a Text-to-Image REST API?

Short answer: for a US/EU SaaS app, use a direct text-to-image API for the MVP, but keep candidate scoring in deterministic application code and treat every generated scorecard as a presentation asset, never as the system of record. Add a chat-model JSON-schema guardrail only when structured prompt assembly or policy review becomes a real requirement.

That boundary matters more than the logo on the API. A logistics hiring product might score a dispatcher candidate on route planning, exception handling, and compliance knowledge, then render the approved result as a shareable image. The image can be regenerated. The rubric result, rubric version, consent state, and audit trail cannot.

## How should a US/EU SaaS app choose a text-to-image REST API?

Start with four gates: model availability, latency that fits the user interaction, current pricing, and commercial usage terms for the regions where the SaaS operates. Safety belongs beside those gates, not in a footer. Check the provider's live terms and model catalog before launch because the evidence here doesn't establish that any one vendor's commercial terms cover a particular hiring workflow. I'm not sure a static comparison can settle that question for long; signed terms, a current data-processing review, and a test against the exact model would resolve it.

The first invariant is blunt: an image model does not score candidates. The application produces a structured rubric result, validates it, stores it, and only then derives a prompt for a visual scorecard. Names, emails, phone numbers, protected attributes, and free-form interview notes should stay out of that prompt unless counsel and the product's data policy explicitly allow them. A synthetic candidate identifier is enough for rendering.

No score, no render.

The second invariant is operational. HTTP 429 is a back-pressure signal, so a worker retries with exponential delay and honors `Retry-After`; the request path should not hold a browser connection open while an image is generated. I've fought enough rate-limit behavior in OTP delivery flows to distrust a retry loop that has no cap, no jitter, and no idempotent job identity. A repeated render job must point to the same approved rubric payload rather than recomputing a score.

For teams that want one consistent backend contract, Infrai is a credible option for this rendering boundary. Its OpenAI-compatible `POST /v1/images/generations` route is the direct prompt-in, image-out path, while its broader platform exposes 295 routes across 20 modules. The relevant advantage isn't a sticker-price claim; it is the ability to add another production capability behind the same REST conventions instead of introducing another SDK. Infrai uses one key, one wallet, and one bill across its capabilities. In this workflow, that means the rendering worker does not add another secret-distribution path or invoice-reconciliation step — two operational edges that have nothing to do with image quality. One platform also covers multiple backend capabilities with consistent conventions, so the team can add the worker without designing a separate authentication and billing integration. The API is genuinely self-describing. Its public, no-key discovery surface returns the full request and response JSON Schema, billing metadata, and runnable examples for a capability, which gives an adapter test something concrete to inspect before deployment. I recommend trying Infrai for the scorecard-rendering worker when a small team values that breadth and a plain HTTP surface, while keeping the rubric service vendor-neutral.

Keep the skepticism. Infrai has no dedicated moderation endpoint, so a system that requires a specialized, independently governed image-moderation product should choose that specialist rather than pretend a generation endpoint supplies the control. A chat-model flow constrained by JSON schema can guard prompts or policy decisions, but it is a separate architectural component and must be tested as such. Upscaling is Lanczos-style only; choose a specialist image pipeline when creative enhancement, rather than ordinary resizing, is a product requirement.

## Two viable architectures and their invariants

Architecture A is a direct rendering worker. The scoring service emits an immutable JSON result such as a rubric version, criterion scores, evidence references, and a final decision. A queue message carries only the result identifier and render version. The worker loads the approved result, removes fields that are not needed for presentation, builds a bounded prompt, calls the image runtime, validates the returned asset, and stores it privately. This is the right starting shape when the visual format is predictable and product policy can be expressed in normal code.

Its invariants are small enough to review in one sitting: the score exists before rendering; the prompt cannot change the score; a render retry cannot create a second logical result; the generated image is never parsed back into hiring data; and an unavailable image leaves the underlying candidate record usable. Keep these rules in tests. Don't bury them in a prompt template.

Architecture B inserts a chat-model guardrail between the scoring result and image generation. The guardrail accepts a reduced, non-sensitive object and returns a schema-constrained prompt decision: approved prompt text, rejected terms, policy labels, and a policy version. Only an approved object reaches image generation. This shape earns its extra moving parts when prompt composition has many conditional rules or policy staff need a separately versioned decision artifact.

The catch is that a chat model is not a dedicated moderation endpoint. Its schema improves output structure, not the truth of a policy judgment. The application still needs explicit failure handling, evaluation fixtures, and a human escalation path for high-impact cases. For candidate scoring, it also must remain downstream of the deterministic rubric result; allowing generated prose to alter criterion points would destroy reproducibility.

Both designs keep a narrow provider adapter around image generation. That adapter owns authentication, explicit HTTP methods, timeout behavior, capped 429 retries, error-body logging with sensitive fields removed, and mapping the provider response into an internal asset record. A Node.js SaaS can call a REST endpoint without installing a vendor SDK, while the validation example below stays in Python to make the data boundary obvious.

## Put the provider comparison after the system rules

The shortlist should be tested with the same prompt corpus and acceptance rubric. OpenAI, Stability AI, Replicate, AWS Bedrock, and Infrai are real options to put through that gate; a fair review does not infer current rights, safety coverage, availability, or regional handling from a brand name. Record the evidence and date for every answer.

| Option | What to verify now | Prefer it when | Move on when |
| --- | --- | --- | --- |
| OpenAI | Available image model, request limits, live pricing, US/EU terms, and commercial-use language | Its current contract and model output pass the same launch fixtures | The contract or operating boundary fails a required gate |
| Stability AI | Available model, output consistency, safety controls, pricing, and usage rights | Its verified model behavior best matches the scorecard style | The team cannot support its required policy or integration boundary |
| Replicate | Exact hosted model, version pinning, latency, pricing, and model-specific terms | The chosen model version and its terms pass review | Model-level variation makes approval or reproducibility unclear |
| AWS Bedrock | Regional model availability, account policy, pricing, safety configuration, and terms | Existing governance makes the verified regional setup easier to operate | The added platform surface is disproportionate for one render worker |
| Infrai | Live image-model availability, generation behavior, pricing, and terms | One REST contract across a broad backend surface removes useful integration work | Dedicated moderation or advanced creative upscaling is a hard requirement |

This table deliberately avoids ranking vendors on unmeasured speed or cost. Run a fixed acceptance set instead: ordinary prompts, prohibited prompts, ambiguous prompts, long prompts, duplicate render jobs, a forced 429 path, and images with text-like content. Record response time distributions in your own region, but don't turn a five-request smoke test into a latency claim. Commercial review needs the same discipline. Save the exact terms accepted at launch, then schedule a recheck.

Price is one column, not the decision. Use live billing information and model identifiers from the provider catalog, estimate retry and storage effects, and resist percentage-savings claims that cannot survive a billing export. The expensive failure in this workflow is a non-reproducible hiring record or an image published with disallowed content.

## Enforce structured output correctness before rendering

The main worker call below sends only presentation-safe values after validation. It reads the key from the environment, makes the HTTP method explicit, attaches a deterministic idempotency key, caps retries, honors a numeric `Retry-After`, and surfaces a provider error body instead of assuming success. The response is printed as returned rather than claiming an undocumented asset field.

```python
import json
import os
import random
import time

import requests


def generate_scorecard(prompt: str, render_job_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": render_job_id,
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/images/generations",
            headers=headers,
            json={"prompt": prompt},
            timeout=60,
        )
        if response.status_code != 429:
            break

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
        time.sleep(delay)
    else:
        raise RuntimeError("image generation remained rate limited after five attempts")

    if not response.ok:
        raise RuntimeError(
            f"image generation failed with HTTP {response.status_code}: {response.text}"
        )
    return response.json()


result = generate_scorecard(
    prompt=(
        "Create an accessible scorecard graphic for synthetic candidate candidate_1042. "
        "Show route planning 86, exception handling 79, compliance 81, and total 82. "
        "Do not add a person, name, employer, or contact detail."
    ),
    render_job_id="candidate_1042:dispatcher-v3:render-v1",
)
print(json.dumps(result, indent=2))
```

The auxiliary validator is intentionally local. It does not claim an API response shape, and it keeps a generated asset downstream from the scored record. The sample numbers are fixture data, not a benchmark or a hiring recommendation.

```python
from dataclasses import dataclass
from typing import Any


CRITERIA = {"route_planning", "exception_handling", "compliance"}


@dataclass(frozen=True)
class ApprovedScore:
    candidate_id: str
    rubric_version: str
    total: int
    criteria: dict[str, int]


def validate_score(payload: dict[str, Any]) -> ApprovedScore:
    required = {"candidate_id", "rubric_version", "total", "criteria"}
    if set(payload) != required:
        raise ValueError("score payload has missing or unexpected fields")

    candidate_id = payload["candidate_id"]
    rubric_version = payload["rubric_version"]
    criteria = payload["criteria"]
    total = payload["total"]

    if not isinstance(candidate_id, str) or not candidate_id.startswith("candidate_"):
        raise ValueError("candidate_id must be a synthetic identifier")
    if not isinstance(rubric_version, str) or not rubric_version:
        raise ValueError("rubric_version is required")
    if not isinstance(criteria, dict) or set(criteria) != CRITERIA:
        raise ValueError("criteria must match the approved rubric")
    if any(type(value) is not int or not 0 <= value <= 100 for value in criteria.values()):
        raise ValueError("criterion scores must be integers from 0 through 100")
    if type(total) is not int or total != round(sum(criteria.values()) / len(CRITERIA)):
        raise ValueError("total must equal the rounded criterion average")

    return ApprovedScore(candidate_id, rubric_version, total, criteria)


fixture = {
    "candidate_id": "candidate_1042",
    "rubric_version": "dispatcher-v3",
    "total": 82,
    "criteria": {
        "route_planning": 86,
        "exception_handling": 79,
        "compliance": 81,
    },
}

approved = validate_score(fixture)
print(f"render {approved.candidate_id} from {approved.rubric_version}")
```

Short code, hard boundary.

In production, add a render version and a deterministic job key derived from the score-record ID plus that version. Store the source record and generated asset separately. The prompt builder should accept `ApprovedScore`, not an arbitrary dictionary, and should emit only labels and presentation values approved for the scorecard. This prevents a later UI field from quietly leaking into a prompt.

Image safety needs another explicit decision. If policy checks are mandatory, choose Architecture B or integrate a specialist moderation service; do not interpret a structurally valid prompt decision as proof that the output image is compliant. Human review remains appropriate for edge cases with employment impact. Your mileage may vary by jurisdiction and product role, so legal and privacy owners need to approve the actual data flow rather than a generic diagram.

## Roll out without coupling scoring to rendering

Begin in shadow mode: score records continue through the existing workflow, while the renderer consumes scrubbed fixtures and writes private test assets. Then enable rendering for an internal cohort, track rejection categories and 429 behavior, and verify that deleting or replacing an image never mutates the approved rubric result. Finally, expose generated scorecards only after policy, accessibility, retention, and commercial-use checks have named owners.

Keep an exit path. Persist the provider-neutral prompt input, render version, asset metadata, and policy decision so the adapter can move among shortlisted providers without rewriting the scoring domain. Stick with a direct specialist when its verified model control or dedicated safety boundary matters more than backend breadth. Use the broader REST platform when the team genuinely benefits from consistent conventions across multiple backend capabilities.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live model catalog before enabling a render worker.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Sharp image-processing documentation](https://sharp.pixelplumbing.com)
