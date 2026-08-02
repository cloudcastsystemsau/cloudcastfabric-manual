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
`docs/MANUAL.md` in the private `CloudCastFabric` repository, rendered by
`build/build-manual.mjs` there:

```bash
node build/build-manual.mjs          # docs/MANUAL.md -> docs/manual.html
```

That generator reads the brand mark straight out of `brand/logo/` rather than
copying it in, so the manual cannot drift from the brand package, and it exits
non-zero if any in-page anchor fails to resolve — which is what catches a
heading rename that would otherwise silently break the contents list.

Edits belong in the source repository, not here; anything changed directly in
this repo is lost on the next publish.

---

CloudcastFabric is a Cloudcast Systems product.
