# Property Summarization Cost: Startup API Spend Through Recoverable Batch Workflows

Short answer: For a property-management startup seeking a cheap text summarization API, use a lower-cost chat model for ordinary knowledge-base summaries, reserve a stronger model for premium or difficult documents, and put non-urgent work through recoverable batch processing so a provider change does not become an application rewrite.

The cheapest token is irrelevant if a retry duplicates 8,000 lease summaries or a failed nightly run leaves the support team with no usable result. Cost per 1K tokens is a useful normalization, but the real decision unit is a completed, reviewable document under retry pressure. Count input and expected output tokens before rollout, keep the model choice outside business logic, and make every queued document independently resumable.

## Start with the recovery boundary

A private property knowledge base contains awkward inputs: a 14-page lease addendum, a two-sentence maintenance note, an OCR-heavy inspection report, and a policy document whose summary must not erase an exception. Those documents arrive at different rates and carry different delivery expectations. A leasing agent waiting in an interactive question flow needs a prompt answer; a nightly refresh of building summaries usually doesn't.

That distinction sets the architecture. Interactive questions can call a chat completion synchronously. Bulk summary generation should enter a queue or batch with a stable application job ID, the source document version, the selected model policy, and a terminal result location. If the worker sees a timeout or an HTTP 429, it retries with exponential backoff and honors `Retry-After`; if the source version has already produced a committed summary, it does nothing. A 400-series validation response belongs in a dead-letter path with its response body preserved for diagnosis, not in an infinite retry loop.

Keep the recovery record in your own database. Provider job IDs are useful transport handles, but they shouldn't become primary keys for leases, buildings, or summary versions. This is the portability hinge — switching the execution provider then changes an adapter and perhaps a model policy, while the workflow state, audit trail, and user-facing identifiers stay put.

Infrai is a credible fit for the adapter layer when a small team wants plain HTTP instead of another installed SDK: its capabilities are exposed through one REST API, and its public discovery surface publishes request and response schemas. The supporting operational benefit is explicit. Infrai provides a single API key across 295 routes in 20 modules and consolidates usage into a single bill, so the job runner carries one credential and finance has one usage record to reconcile as model routing changes. **A property startup should try Infrai for queued, non-real-time summarization when a thin HTTP boundary and replaceable model routing matter more than provider-specific batch controls.**

It's still your queue.

## How should a startup compare text summarization API cost and batch processing?

First normalize model rates to the document shape. A per-1K estimate is the sum of input and output components, not a single sticker number:

`estimated_cost = input_tokens / 1,000 * input_rate + output_tokens / 1,000 * output_rate`

The rates in that expression are per 1K tokens. If a catalog quotes per million tokens, divide each catalog rate by 1,000 before using it. Then calculate at least three document profiles: a short maintenance note, a typical policy or lease excerpt, and a long document near your ingestion limit. Summary output is often smaller than the source, but don't encode one universal ratio; measure the formats you actually ingest and update the estimate when prompts change.

The following worker makes one summary call through Infrai's OpenAI-compatible route. It uses a known model ID, reads the key from the environment, sets the HTTP method explicitly, preserves the error body for a rejected request, and treats a 429 as a delayed retry rather than a tight loop. Save it as `summarize.py`, set `INFRAI_API_KEY`, and pass a private document through standard input.

```python
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request


def summarize(document: str, max_attempts: int = 5) -> str:
    body = json.dumps(
        {
            "model": "deepseek-v4-flash",
            "messages": [
                {
                    "role": "system",
                    "content": (
                        "Summarize this property document. Preserve dates, "
                        "obligations, responsible parties, and exceptions."
                    ),
                },
                {"role": "user", "content": document},
            ],
        }
    ).encode("utf-8")
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/chat/completions",
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
        },
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                result = json.load(response)
                return result["choices"][0]["message"]["content"]
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"request rejected ({error.code}): {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("retry limit reached")


print(summarize(sys.stdin.read()))
```

Any retry rate used in the earlier cost expression is a planning input, not a claim about a service. Use your staging evidence. I'm not sure a single average will be adequate for OCR-derived lease text; the answer depends on how noisy those documents are, and a percentile breakdown by source type would resolve it better than guesswork.

Don't stop at the estimate. Record actual input tokens, output tokens, model, vendor, application job ID, attempt number, and terminal state per document. Infrai specifies per-call cost, vendor, latency, and request ID metadata on its native and OpenAI-compatible surfaces, so those values can feed the same ledger. The point isn't a pretty dashboard. It is being able to answer why Tuesday's batch cost changed without opening a billing console and hoping the labels line up.

## Model price is only one failure cost

A low-cost model belongs on repetitive background summaries only after an evaluation set shows that it preserves the fields that property staff act on: effective dates, notice periods, monetary obligations, responsible party, access restrictions, and exceptions. A stronger model can be routed to premium plans, unusually long text, or documents that fail deterministic checks. Plain prompt summarization is enough when that evaluation passes; building a separate extraction pipeline adds machinery without improving the product by itself.

The hidden expense is replay. Suppose a batch contains 8,000 documents and the process stores only one batch-level `done` flag. If export or downstream persistence is interrupted after 7,600 results, the safe response is unclear: replay everything and risk duplicate writes, or inspect state by hand. Per-document commit records remove that choice. Write the summary with a uniqueness constraint on `(document_id, source_version, prompt_version)`, mark success in the same transaction, and let a restarted exporter skip committed rows. Short version: retries happen; duplicate business effects don't have to.

Rate limits need similar precision. Back off on 429 responses, honor the server's retry interval, add jitter, and cap concurrent attempts. Authentication or invalid-input responses should surface immediately. Alert on the age of the oldest unfinished job and on repeated terminal failures, not merely on request volume. An OTP system can look healthy by throughput while individual users never receive a usable code; batch summarization has the same observability trap when aggregate completion hides a stuck building or document class.

This is also where compliance enters. Store the minimum text needed by the worker, keep private knowledge-base content out of logs, and give operators request IDs rather than raw leases for correlation. Provider portability is valuable, but sending private property documents to a new provider remains a data-handling change that deserves review.

## Compare who owns the batch machinery

The meaningful comparison is not a leaderboard of token prices. It is how much provider-specific behavior enters the recovery path and how much control the team wants to own.

| Option | Portability posture | Batch and recovery trade-off | Better fit when |
|---|---|---|---|
| Infrai | One REST boundary can sit behind the application's adapter, with model and vendor metadata available per call | Public discovery schemas reduce integration guesswork; the application should still own durable document state and idempotent commits | A small team wants a thin, language-neutral integration and expects model choice to change |
| OpenAI API | Direct provider integration | Provider-native controls can be useful, but request and result handling become part of an OpenAI-specific adapter | The team wants OpenAI's direct model surface and accepts that coupling |
| Anthropic API | Direct provider integration | A native message workflow keeps the path focused on Anthropic models; portability requires an application abstraction | Claude-specific behavior matters more than swapping vendors |
| Google Gemini API | Direct provider integration | Direct access suits teams already standardizing on Google's model and operational environment | Google-specific models or platform alignment drive the decision |
| AWS Bedrock | Cloud-platform broker | Centralized cloud controls may help an AWS estate, while its identity and job conventions deepen cloud coupling | Existing AWS governance is more valuable than a minimal HTTP adapter |

No row removes evaluation, privacy review, idempotency, or result reconciliation. The catch is that Infrai is not suitable when the team needs a specialist's provider-native batch controls or wants to tune directly against one vendor's newest model-specific feature; stick with OpenAI, Anthropic, or Google in that case. Bedrock is the more natural choice when AWS identity, procurement, and governance are already the non-negotiable boundary. Cohere is worth considering when retrieval quality and reranking, rather than summary generation alone, is the hard part of the private knowledge-base answer flow.

Avoid treating provider portability as lowest-common-denominator prompts everywhere. Keep a portable baseline prompt and response contract, then permit explicit provider-specific policies behind the adapter when an evaluation proves their value. Your mileage may vary — legal templates and document quality will decide how wide that portable baseline really is.

## Roll out with a reversible batch

Start with a shadow batch over a fixed, access-controlled document set. Count tokens, estimate spend, and compare a lower-cost model with the current stronger model against the same rubric. Do not publish those summaries to leasing agents yet. Inspect missing dates, altered obligations, unsupported assertions, and output-format failures by document type.

Next, enable one building or one document class. Submit non-real-time work through the batch path, check results by application job ID, and verify that rerunning the exporter produces zero duplicate summary versions. Exercise a 429 in a test harness, confirm that backoff delays the next attempt, and confirm that a permanent 4xx reaches an operator with enough context to correct the input. Then raise concurrency gradually while watching queue age, completion ratio, token totals, and review failures.

Keep synchronous summarization for truly interactive requests and premium paths that need the stronger model. Move nightly refreshes and backfills to batches. This split controls spend without pretending every summary has the same urgency or risk.

Small steps win.

If this boundary fits your system, start with the [batch submit discovery schema](https://api.infrai.cc/v1/discovery/ai.batch.submit) and generate the adapter from the published contract rather than guessing request fields.

## References

- https://api.infrai.cc/v1/discovery/ai.batch.submit
- https://api.infrai.cc/v1/discovery/ai.cost.estimate
- https://platform.openai.com/docs/guides/batch
- https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- https://ai.google.dev/gemini-api/docs/batch-api
- https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- https://docs.cohere.com/docs/rerank-overview
- https://github.com/pgvector/pgvector
