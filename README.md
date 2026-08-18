# CloudcastFabric manual

The published manual for **CloudcastFabric** — a multicast-over-unicast overlay
fabric for cloud instances, with first-class AES67/PTP timing.

**Read it:** https://cloudcastsystemsau.github.io/cloudcastfabric-manual/

| File | What it is |
|---|---|
| `index.html` | The manual, generated and branded. Do not hand-edit. |
| `MANUAL.md` | The markdown source it was generated from. |
| `brand/` | Logo, favicon and lockups from the CloudcastFabric brand package. |

## How it is produced

This repository is a **publishing target**. The source of truth is
`docs/MANUAL.public.md` in the private `CloudCastFabric` repository, rendered by
`build/build-manual.mjs` there:

```bash
node build/build-manual.mjs docs/MANUAL.public.md index.html
```

`MANUAL.public.md` is the **customer edition** — a deliberate edit of the
engineering manual (`docs/MANUAL.md`) in the same repository, not a build of it.
Section 8 is scope rather than a defect list, limitations are stated as scope,
and the internal design references are dropped. When the engineering manual
gains something customers should see, it is ported across by hand.

That generator reads the brand mark straight out of `brand/logo/` rather than
copying it in, so the manual cannot drift from the brand package, and it exits
non-zero if any in-page anchor fails to resolve — which is what catches a
heading rename that would otherwise silently break the contents list.

Edits belong in the source repository, not here; anything changed directly in
this repo is lost on the next publish.

---

CloudcastFabric is a Cloudcast Systems product.
