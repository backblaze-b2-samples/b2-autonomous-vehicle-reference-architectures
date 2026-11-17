# Agent guide

Use this repository to design or review storage workflows for autonomous vehicle (AV) data on Backblaze B2: drive-log offload, sensor lakes and scenario mining, multi-sensor labeling, perception datasets and checkpoints, and re-simulation.

<!-- CODEGRAPH_START -->
## CodeGraph

When a `.codegraph/` directory exists at the repository root, use `codegraph explore "<question or symbols>"` before grep, find, or broad file reads when locating or understanding code. If the directory does not exist, skip CodeGraph; indexing is a maintainer decision.
<!-- CODEGRAPH_END -->

## Sources of truth

1. `catalog/architectures.yaml` is the machine-readable catalog of architectures, third-party integrations (with `connection_type`), and external sample applications. `schemas/architecture-catalog.schema.json` validates it.
2. `architectures/*/architecture.md` contains the detailed design, B2 zones, boundaries, and security.
3. `examples/` contains B2 configuration templates. This repository has no runnable applications; samples live in external repositories.
4. Current Backblaze product behavior is authoritative at <https://www.backblaze.com/docs/cloud-storage>. Pricing is authoritative at <https://www.backblaze.com/cloud-storage/pricing>.

Describe the documented connection for each integration: a B2 backend or provider preset, a custom S3 endpoint, presigned URLs, or an application worker using an SDK. Specify the endpoint, region, addressing style, required operations, and credential scope. Attribute product capabilities to the linked vendor documentation or source code. Describe sample applications by the workflow they implement. Use `native-b2-option` for an actual B2 backend or B2-specific provider preset; use `s3-custom-endpoint` for a configurable S3 connection.

For a single-file read of the summary and every architecture, use `llms-full.txt`.

## Generated files

Run `python3 tools/generate_catalog.py` after editing the catalog. Do not hand-edit generated sections in `README.md`, `docs/related-b2-samples.md`, `docs/architecture-index.md`, `skill/references/architecture-catalog.md`, `llms.txt`, or `llms-full.txt`. `llms-full.txt` concatenates narrative pages, so regenerate after editing `docs/autonomous-vehicles.md` or any `architecture.md`.

Validate with:

```bash
python3 tools/generate_catalog.py --check
python3 tools/check_markdown_links.py            # add --external to probe outbound links
python3 -m pytest tests
```

## Operational invariants

- Derive `https://s3.<region>.backblazeb2.com` from the bucket region; never hardcode a real region.
- Use scoped application keys: one bucket and at most one name prefix per key, with only the capabilities the role needs. Keep master or admin keys in provisioning only.
- Vehicles never hold keys; they use presigned PUT or multipart part URLs. Labeling tools never read `raw/`.
- Treat presigned URLs as bearer credentials; the SigV4 maximum is 7 days and a single presigned PUT is limited to 5 GB.
- Write dataset, label, scenario, and suite versions immutably with `manifest.json` last.
- Event Notifications are at-least-once and prefix-filtered: verify the HMAC signature and make consumers idempotent.
- Keep catalogs, metadata databases, annotation-tool state, queues, and NVMe scratch off object storage. Never set `daysFromUploadingToHiding` on Iceberg `warehouse/` or lakeFS `lakefs/`; give them a hidden-only rule (`daysFromHidingToDeleting`) so files the tool deleted are reclaimed.
- Cloud Replication is bucket-to-bucket DR, not delivery.
- Never commit credentials, real presigned URLs, or vehicle data.
