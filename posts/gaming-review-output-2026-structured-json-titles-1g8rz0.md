# Gaming Review Output 2026: Structured JSON Titles, Bullets, and Takeaways

Short answer: for a structured summary JSON output API, choose a mode that constrains generation to a schema, then keep a second validator at your application boundary. For gaming code reviews, the winning design is the one that rejects malformed titles, bullets, and key takeaways before they reach pull-request automation; fluent prose, low latency, and a convenient Node.js client come after contract correctness.

This decision record treats a review as data, not chat. The input is a patch plus repository context. The output has a short title, evidence-backed bullets, and key takeaways that a bot can route without guessing. A syntactically valid JSON object is only the first gate. A finding with an unknown severity, an empty file path, or a line outside the diff is still wrong.

## The acceptance corpus is the architecture

Use a provider's schema-constrained output feature behind an internal adapter, and validate the returned object again before publishing it. The adapter keeps the application contract stable even if a team changes models or providers. It also prevents provider-specific response envelopes from leaking into review logic.

The canonical object should be deliberately small. A useful contract has one bounded title, a list of finding objects, and a list of key takeaways. Each finding needs a stable identifier, severity, path, line, explanation, and remediation. Disallow unknown keys. Require every field the downstream system reads. If optionality is necessary, represent it explicitly rather than asking consumers to infer absence.

Three invariants matter most. First, the response must parse as one JSON value with no surrounding commentary. Second, it must satisfy the local schema, including enums and length bounds. Third, each claim must be grounded in the submitted patch. The first two are mechanical. The third needs deterministic checks where possible and a separate quality evaluation where it isn't.

This is where review systems resemble OTP delivery more than a writing assistant: an apparently successful request isn't the same as a completed transaction. A provider can return valid JSON while the application still has no publishable finding. Treat parse, schema validation, evidence validation, and publication as separate states, with separate counters. Don't collapse them into a single `success` metric.

It should guarantee the shape it explicitly supports, no more. The application must own semantic rules such as “the cited line belongs to the patch” and “critical means exploitable or data-destroying.” Those rules depend on the repository and can't be delegated to a generic output grammar.

For selection, run the same contract suite against every candidate API. Include a normal patch, an empty diff, a renamed file, a deleted line, a generated asset, and adversarial text embedded in a comment. Require exact key names and types. Then score evidence precision separately from schema pass rate; combining them hides why a candidate failed. I'm not sure a single model-side guarantee can ever settle repository-specific evidence quality. A replayable fixture set will.

Keep refusal and truncation outside the success object. If the upstream response indicates that generation did not complete normally, the adapter should return a typed non-success result rather than manufacturing empty bullets. That distinction matters operationally: “no findings” is a valid review outcome, while “no usable model output” should be retried or queued for a human.

A compact state machine is enough:

1. `received`: patch and policy version recorded.
2. `generated`: upstream call completed normally.
3. `validated`: JSON, schema, and patch evidence passed.
4. `published`: the review comment was written once.

Stop early on failure.

Return `422` when a completed response violates the internal review contract. Reserve `429` handling for rate limiting and apply bounded retries with jitter before moving the job to a delayed queue. A timeout is ambiguous — the caller shouldn't assume the remote side did no work — so give each review an idempotency key and make publication conditional on that key. These are application policies, not claims about one vendor's status codes.

Log the schema version, provider adapter version, model identifier, finish state, validation stage, latency, and a digest of the patch. Don't log raw proprietary source by default. For compliance and incident review, retain the minimum artifact needed to reproduce a disputed finding, apply repository access controls, and define deletion periods before launch. Gaming repositories routinely contain unreleased mechanics and anti-cheat logic; a cheerful debug dump can become the larger risk.

Watch separate rates for parse rejection, schema rejection, evidence rejection, retry exhaustion, and duplicate-publication suppression. The denominators matter. A graph saying “99% valid” is useless if skipped jobs disappeared before validation.

One sharp edge deserves a longer example. Suppose a patch deletes a bounds check at line 184 and adds a replacement at line 219. A plausible review may cite line 184, describe an out-of-bounds read, and conform perfectly to the schema. Yet a diff-aware validator can see that the cited line is deleted, while a context pass can see the replacement. Publishing the finding would create noise and train developers to ignore the reviewer. The contract therefore validates more than types: the cited path must exist in the patch, the line must be an added or contextual line under the team's policy, and the evidence text must be recoverable from the captured input. If any check fails, record the failed stage and withhold the comment.

## A failure budget exposes the real comparison

| Option | Shape guarantee | Portability | Operational cost | Best fit |
|---|---|---|---|---|
| Schema-constrained generation | Strong candidate for exact keys, enums, and nesting when the chosen API supports the required schema subset | Medium; adapters must normalize provider envelopes and supported keywords | Contract tests plus provider integration tests | Automated review comments and queues |
| Tool or function arguments | Structured arguments tied to a declared callable contract | Medium; tool-call representations differ | Must handle tool selection, completion state, and argument validation | Workflows that execute a defined action after review |
| Prompted JSON plus local validation | No model-side structural guarantee | High at the prompt layer, lower at the retry layer | More rejection, repair, and retry logic | Low-risk prototypes and providers without constrained output |
| Self-hosted constrained decoding | Controlled by the team and model stack | High inside that stack | Serving, grammar, upgrades, and capacity belong to the team | Regulated or isolated environments with platform staff |

The table is a shortlist, not a ranking. Set separate failure budgets for malformed shape, unsupported evidence, and exhausted retries before testing it. OpenAI's function-calling guide is one primary example of declaring callable tools and receiving structured arguments. Other APIs expose different surfaces, so verify the current documentation and run the fixtures; don't infer equivalent behavior from a similarly named feature.

Latency and price belong in the evaluation, but neither rescues a weak contract. Measure full review completion, including retries and validation, rather than the first response token. Also price the rejected attempts and retained evaluation data. Your mileage may vary because patch size, schema complexity, and retry policy change the workload more than a headline per-token rate suggests.

## How can Node.js keep structured summary output inside a JSON title-and-bullets schema?

The following Python harness represents the boundary even when the calling service is Node.js. The Node.js worker can write the candidate object and patch metadata to the same internal contract; keeping this verifier at the HTTP or queue boundary makes provider swaps less invasive.

```python
from dataclasses import dataclass
from typing import Any

ALLOWED_SEVERITIES = {"low", "medium", "high", "critical"}
REQUIRED_FINDING_KEYS = {
    "id", "severity", "path", "line", "explanation", "remediation"
}


@dataclass(frozen=True)
class PatchIndex:
    reviewable_lines: dict[str, set[int]]


def validate_review(candidate: Any, patch: PatchIndex) -> dict[str, Any]:
    if not isinstance(candidate, dict):
        raise ValueError("review must be a JSON object")
    if set(candidate) != {"title", "bullets", "key_takeaways"}:
        raise ValueError("review has missing or unknown top-level keys")
    if not isinstance(candidate["title"], str) or not candidate["title"].strip():
        raise ValueError("title must be a non-empty string")
    if not isinstance(candidate["bullets"], list):
        raise ValueError("bullets must be a list")
    if not isinstance(candidate["key_takeaways"], list):
        raise ValueError("key_takeaways must be a list")

    seen_ids: set[str] = set()
    for finding in candidate["bullets"]:
        if not isinstance(finding, dict) or set(finding) != REQUIRED_FINDING_KEYS:
            raise ValueError("finding keys do not match the contract")
        if finding["severity"] not in ALLOWED_SEVERITIES:
            raise ValueError("unknown severity")
        if finding["id"] in seen_ids:
            raise ValueError("finding id must be unique")
        seen_ids.add(finding["id"])

        allowed_lines = patch.reviewable_lines.get(finding["path"], set())
        if finding["line"] not in allowed_lines:
            raise ValueError("finding is not grounded in a reviewable patch line")

    return candidate
```

The snippet intentionally doesn't repair data. Repairing an enum or inventing a missing path can turn a visible contract failure into a false accusation against a developer. Reject, record the stage, and retry only when policy says a second generation is safe. Publication should consume only the returned validated object and should use the review idempotency key.

Test this path with golden fixtures and mutation cases. Remove a required key. Add an unknown key. Change `line` from an integer to a string. Duplicate an ID. Point a finding at an unchanged file. Also test a valid empty `bullets` array, because “no issue found” must not be confused with a broken response. Five targeted mutations often reveal more than fifty happy-path snapshots.

## Rollout, rollback, and the rejected repair loop

We rejected prompted JSON with a “fix it if parsing fails” loop for automated pull-request publication. The catch is that repair adds another generative step between evidence and the developer, and a syntactic fix may silently alter meaning. It also makes rate-limit behavior, latency, and audit trails harder to reason about.

It remains suitable for an internal prototype where a human reads every result and nothing writes back to the repository. Stick with self-hosted constrained decoding when source cannot leave an isolated environment or when the team needs direct control of the serving stack. Choose function arguments instead when a valid review immediately feeds a narrowly defined action and the tool boundary is the clearer contract.

The decision can change. Re-run the fixture corpus when the schema changes, a model identifier changes, or an adapter changes its response mapping. Promote only after both structural and evidence gates meet the team's predeclared thresholds. Keep the old adapter deployable until sampled production reviews clear the same gates; rollback should change routing, not the public review schema.

No silent upgrades.

## Further reading

The primary documentation below is useful for checking one concrete function-argument mechanism. Recheck current provider documentation during evaluation because supported surfaces can change.

## References

- https://platform.openai.com/docs/guides/function-calling
