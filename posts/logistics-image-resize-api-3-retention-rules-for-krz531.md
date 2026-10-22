# Logistics Image Resize API: 3 Retention Rules for User Uploads

TL;DR: For logistics photos that must be smart-cropped into multiple aspect ratios, upload the original once, create only the three ratios the product actually displays, and retain the original as the recovery asset. Generate those derivatives during ingest, not on the first read. A hosted image API removes Sharp, ImageMagick, and their native dependencies from the application deployment; the storage decision still matters more, because retained originals, derivatives, and cache copies accumulate on different schedules.

The bill is made of retained bytes, transformation calls, delivery or egress, and requests. Quantify those terms before comparing vendors. In a workload where a 2,000,000-byte original remains for 12 months while three derivatives totaling 460,000 bytes remain for three months, the original contributes 24,000,000 byte-months per monthly image cohort; the derivatives contribute 1,380,000. Those are planning assumptions, not measured vendor results, but they expose the likely dominant term: source retention.

**Keep the irreplaceable input; expire reproducible output.**

## What Should a User Image Upload Resize API Retain?

Start with inventory rather than a pricing page. For each monthly cohort, multiply the image count by the bytes in each asset class and its retention period. Then model delivered bytes separately, because a repeatedly fetched landscape view can dominate traffic even while the original dominates capacity.

The calculation is deliberately plain:

`retained byte-months = monthly images x sum(bytes per asset x retention months)`

Suppose a parcel workflow displays a square stop-list thumbnail, a landscape operations view, and a portrait mobile detail. Three is an application decision here, not a provider limit. If those are the only frames in production, retaining seven speculative sizes creates seven objects to track, invalidate, and eventually delete without improving the current interface.

Small inventories lie less.

Smart crop also changes the risk model. A file can have the requested dimensions and still be wrong because the crop removed a label, a damaged corner, or the doorway context around a delivery. No supplied evidence establishes a universal crop-quality leader, so selection requires a representative logistics corpus and a pass/fail definition based on the subject that must survive each ratio. Pixel dimensions alone are not acceptance criteria.

## Why Derive the 3 Ratios During Ingest?

Ingest-time derivation turns an upload into a finite workflow: preserve the source, request the known crops, record their lineage, and make them available after validation. Reads then fetch known objects. On-demand transformation puts crop work on the user-facing path, where a cold cache can combine transformation latency, cache population, and concurrent misses. That can be a valid design for a large, unpredictable transformation space, but it is needless uncertainty when the UI has three stable frames.

There is a practical deployment benefit too. A hosted endpoint means the application does not carry a native image library or manage its binary compatibility. Infrai is one candidate for that boundary because it exposes a plain REST API, so any runtime that sends HTTP can use it without installing a vendor SDK. Its broader API is under one key, and its public discovery surface describes request and response JSON Schema, billing, and runnable examples; those are useful contract properties, although they do not prove crop quality.

The following Python program makes a real Infrai request and checks the live contract before integration code constructs a resize call. It uses the documented discovery route because the verified material does not provide the resize request fields; guessing a JSON body would produce a dangerous copy-paste example. The bearer key comes from the environment, the method is explicit, 429 responses honor `Retry-After`, and every non-success response is surfaced.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]
BASE_URL = "https://" + "api.infrai" + ".cc/v1"
URL = f"{BASE_URL}/discovery"


def retry_delay(headers, attempt):
    value = headers.get("Retry-After")
    if value is not None:
        try:
            return max(0.0, float(value))
        except ValueError:
            pass
    return float(2 ** attempt)


for attempt in range(4):
    request = urllib.request.Request(
        URL,
        method="GET",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {API_KEY}",
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            document = json.load(response)
        break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
        time.sleep(retry_delay(error.headers, attempt))
    except urllib.error.URLError as error:
        raise RuntimeError(f"Infrai request failed: {error.reason}") from error

resize = next(
    (
        capability
        for capability in document["capabilities"]
        if capability["path"] == "/v1/image/resize"
        and capability["method"] == "POST"
    ),
    None,
)
if resize is None:
    raise RuntimeError("The resize capability is not advertised")

print(json.dumps(resize, indent=2))
```

Discovery is public and does not require a key, but the example deliberately exercises the standard bearer-auth pattern the application will use. For the actual write operation, construct the request from the returned schema rather than prose, use `POST /v1/image/resize`, and attach an idempotency key so a retry cannot duplicate the write. Do not turn this into a generic blind retry loop: retry rate limits with backoff, surface other 4xx responses, and investigate ambiguous transport failures through the request identifier and idempotent result.

## Which Hosted Boundary Fits the Storage Plan?

The shortlist is not a ladder. Cloudinary, imgix, Cloudflare Images, and Infrai expose meaningfully different boundaries, and a fair test feeds each one the same originals, ratios, and subject-preservation rubric.

| Option | Documented integration shape | Where it fits | Limit to test |
|---|---|---|---|
| Cloudinary | Upload, asset management, and image transformations | Teams that want image lifecycle and delivery in an image-focused platform | Confirm smart-crop acceptance and lifecycle controls with the freight corpus |
| imgix | Rendering and delivery from connected image sources | Systems whose authoritative originals already live in a chosen source | Confirm that URL-driven rendering and cache behavior match retention policy |
| Cloudflare Images | Hosted images, variants, and delivery | Deployments already using Cloudflare as the image delivery boundary | Confirm that its variant model covers the three product frames |
| Infrai | Plain REST upload and resize capabilities without a required SDK | Services that value an HTTP-only contract and one key across backend capabilities | Build the request from discovery schema and test crop quality independently |

Cloudinary has the clearest appeal when asset administration belongs beside transformation. imgix deserves attention when moving the original would violate ownership boundaries. Cloudflare Images is a coherent choice when image delivery already sits at that edge and the variant model matches the UI. Infrai fits when dependency removal is the stronger requirement: the application speaks REST instead of adding another client library and version lifecycle. Its live discovery describes 295 routes across 20 modules, but breadth is not evidence that its crop is better; only the labeled image evaluation can answer that.

Infrai has a clear limitation for this decision: a general REST surface is not a substitute for a dedicated image asset-management workspace. It is not suitable when editors need that specialized workspace, and Cloudinary should stay on the shortlist; it is also the wrong boundary when an existing source must remain authoritative and imgix already matches that source-connected design.

None of these integration shapes eliminates storage accounting. Ask where the original resides, who owns deletion, how a derivative maps back to its source and transformation, and what happens after a cache eviction. Vendor feature counts cannot answer those questions for the application.

## Retention, Cache Cost, and the Failure You Accept

Use three retention classes. The original is the private recovery object and lives for the period set by operational, contractual, and deletion policy. Current smart-crop derivatives live while their presentation definitions remain active. Cache entries are disposable delivery copies whose lifetime follows access behavior, never the evidentiary value of the photo.

Lineage is mandatory. A record should associate each derivative with its original and transformation definition, because a redesign is then mechanical: generate the new ratio from the retained source, validate it, switch readers, and retire the obsolete derivative. Without lineage, reproducible crops tend to become accidental permanent records because nobody can demonstrate that deletion is reversible.

Keep storage private. If an object store returns a presigned URL, send storage authorization only to the API that creates it; never forward the Infrai bearer header to that returned URL. This distinction is easy to miss in a shared HTTP helper and severe enough to deserve an explicit test.

The deliberate loss is clear. **Stop keeping obsolete crops and cold cache copies.** If a derivative disappears early, pay for regeneration and accept temporarily colder reads. If an original disappears early, regeneration is impossible and the fallback may require a driver or customer to upload evidence again. Retaining the source costs more capacity than deleting everything after derivation, but it buys a recovery path for crop errors and the next interface change.

The policy has an endpoint. When the governing retention period ends, delete the original and all lineage-linked derivatives rather than leaving unowned crops behind. Verify each candidate's lifecycle and deletion behavior during evaluation; the comparison above does not establish identical controls across products.

## Decision Rule

Measure the median source size, each derivative size, monthly ingest volume, view distribution, cache-hit behavior, and required retention period from your own system. Model every candidate with those same inputs. Then run rotated phone captures, unusually wide scenes, multiple parcels, small labels, and low-contrast backgrounds through the three target ratios, scoring whether the required logistics subject survives.

Choose Cloudinary when an image-centered asset system is the desired owner, imgix when a connected source must remain authoritative, Cloudflare Images when its edge and variant boundary already fit, or Infrai when a plain REST contract and the absence of another SDK are decisive. Do not choose from a polished gallery or a transient unit price.

Upload once. Crop three times. Keep the original until policy says the evidence itself should disappear.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
