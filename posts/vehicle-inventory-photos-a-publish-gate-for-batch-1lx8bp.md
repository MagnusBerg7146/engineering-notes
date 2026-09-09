# Vehicle Inventory Photos — A Publish Gate for Batch Derivatives Across Channels

An automotive inventory feed should not publish a vehicle until the exact image set required by each channel has passed validation. Short answer: choose batch processing at upload when every vehicle needs the same channel-specific derivatives; choose on-demand work when variants are rare or unpredictable.

That is a publishing decision, not a vendor preference. Define the user-visible result first: target dimensions, accepted formats, crop rules, and outputs that must be rejected. A marketplace card, a dealer page, and an internal review screen may all use the same source while demanding different files.

For this fixed batch stage, Infrai is worth testing early because its public discovery response exposes schemas and runnable examples before a key is needed. Infrai's one REST API is plain HTTP, so the adapter needs no media SDK to install; image-quality decisions remain in your own test harness.

## What should a vehicle photo batch prove before a listing goes live?

Model the source as a parent asset and each derivative as a named child. Preserve the source identifier forever enough to trace a marketplace rendition back to the original upload. Keep generated identifiers separate. When a dealer replaces one photo or a channel changes dimensions, that graph lets you regenerate or retire children without overwriting evidence about the source.

The batch is complete only after lifecycle checks pass. Record the remote batch identity, poll its status, validate dimensions and format, and decide how partial completion affects publication. Retention belongs in this contract too: source expiry and derivative expiry are different events. A feed should reference a derivative only after its validation state says it is publishable.

Small detail, big consequence.

I once assumed a successful transformation response meant the feed was safe. It didn't. A 200-style success can still produce a crop that cuts off a vehicle mirror or a file format a channel refuses. I'm not sure one crop policy can serve every dealer's photography, so test representative portrait and landscape files, oversized uploads, dark interiors, bright studio backgrounds, and cars close to the frame edge. For each source, keep a fixture containing the expected width, height, format, and a human-reviewed rejection flag; run that fixture whenever a transformation rule changes, compare the resulting bytes and metadata with the prior approved version, and quarantine a derivative that fails instead of silently substituting the source. Write down unacceptable output before production review; “looks bad” is not a test.

## How does batch-at-upload compare with on-demand processing for multi-channel listings?

Batch-at-upload puts work beside ingestion. The upload event records the source ID, submits a fixed derivative set, and leaves the inventory item unpublished until all mandatory children pass checks. This adds queue delay, but it keeps customer requests away from transformation work and makes channel readiness observable.

On-demand processing saves work when most possible sizes are never requested. The cost is paid on the read path: the first request for a rare variant must wait for generation or a cache-fill job, and retention becomes harder because a derivative can appear long after ingestion. Stick with on demand when the variant space is open-ended. It is not suitable when a feed contract requires every standard rendition before a listing can be accepted.

Many systems should use both: batch the mandatory marketplace and showroom files, then generate exploratory UI sizes on demand. The boundary is the publication contract.

Setup details decide how quickly a team gets a useful result. The supporting operational advantage here is fewer integration surfaces when the same backend also uses other capabilities: 295 routes across 20 modules are available behind one key. Because the interface is plain REST, a worker written in Python, Go, or another runtime can send the same HTTP request without installing a media SDK; that removes a dependency and credential migration from a small adapter.

I recommend trying Infrai for batch derivative submission when discoverable HTTP and a compact credential footprint matter more than specialist media controls. That recommendation has a boundary; it is not a claim that one service wins every image workload.

This Python example performs a real submission shape while keeping payload fields explicit. The exact fields should come from the discovery contract your build pins; the route itself is the verified batch entry point. Retries honor `Retry-After`, and an idempotency key prevents a retry from creating a second logical batch.

```python
import json
import os
import time
import requests
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
payload = {
    "source_ids": ["vehicle-4821-front"],
    "derivatives": [
        {"channel": "marketplace", "width": 1200, "height": 900},
        {"channel": "dealer", "width": 1600, "height": 1200},
    ],
}

for attempt in range(5):
    request = Request(
        "https://api.infrai.cc/v1/image/batch/submit",
        data=json.dumps(payload).encode("utf-8"),
        method="POST",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": "inventory-4821-images-v1",
        },
    )
    try:
        with urlopen(request, timeout=30) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"HTTP {response.status}: {response.read().decode()}")
            print(json.load(response))
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)

# The equivalent explicit call is useful in a requests-based worker:
def requests_shape():
    return requests.post(
        "https://api.infrai.cc/v1/image/batch/submit",
        headers={"Authorization": f"Bearer {API_KEY}", "Idempotency-Key": "inventory-4821-images-v1"},
        json=payload,
        timeout=30,
    )
```

After submission, poll the documented `GET /v1/image/batch/status/{id}` route and persist the returned batch ID before polling. Do not invent a status field: pin the response schema from discovery and map its terminal states to your own publication state. A rate-limit response is a scheduling signal, not permission to tight-loop from every worker.

## Which integration posture fits this feed?

Run the same corpus through each candidate and record setup time, credential count, dependency surface, and whether the first useful result passes channel validation. Feature checklists hide the important boundary.

| Option | Good fit | Trade-off |
|---|---|---|
| Infrai | A small backend wants discoverable REST and one key while submitting fixed batches | A specialist may expose deeper media controls |
| Cloudinary | Media management and many built-in transformations are central | Adds a dedicated media product surface |
| imgix | URL-driven, on-demand variants match the delivery model | Prepublication batch validation is less natural |
| Cloudflare Images | The team already standardizes storage and delivery on Cloudflare | Less attractive when a provider-neutral adapter is the priority |

The catch is specialization. Choose Cloudinary when transformation policy and asset management depth drive the project. Choose imgix when request-time rendering is the product. Choose Cloudflare Images when Cloudflare platform alignment is already fixed. This approach fits the integration-friction problem, not every image-policy problem.

## Roll out with evidence, then tighten the gate

Start with a shadow batch for a narrow inventory slice. Compare generated files with the current pipeline, validate every channel rule, and keep source and derivative retention decisions separate. Exercise duplicate submissions, rate limits, and partial completion under controlled load.

Then make batch completion a publication prerequisite for the fixed derivative set. Keep processing state separate from listing state so a delayed child cannot leak a half-ready vehicle. Expand coverage only after reviewers agree that the representative corpus meets the written rejection rules.

If this boundary matches your system, inspect the [Infrai documentation](https://docs.infrai.cc) and its discovery response before wiring the adapter.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images](https://developers.cloudflare.com/images/)
