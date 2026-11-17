---
name: b2-autonomous-vehicle
description: Design or review Backblaze B2 storage for autonomous vehicle data, including drive-log (rosbag2/MCAP) offload from vehicles and depots, sensor lakes and scenario mining with Spark, Trino, DuckDB, Iceberg, or LanceDB, camera and LiDAR anonymization and labeling with CVAT, Label Studio, Segments.ai, or FiftyOne Enterprise, perception datasets and checkpoints with Ray, MLflow, lakeFS, or DVC, and re-simulation. Use for B2 zones, key scoping, presigned uploads, Event Notifications, Object Lock evidence, and what to keep off object storage.
---

# B2 for autonomous vehicle data

Choose a workflow from the bundled [architecture catalog](references/architecture-catalog.md), which also lists related B2 sample applications.

For changing service behavior, limits, and setup steps, use the canonical [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage). Link pricing claims to [B2 pricing](https://www.backblaze.com/cloud-storage/pricing) instead of copying numbers.

## Rules
- For LanceDB embedding tables, follow the architecture's pinned connection profile and serialize all table mutations through one writer service. Endpoint settings alone do not establish safe commit behavior across Lance versions.
- Use the S3-compatible API. Derive `https://s3.<region>.backblazeb2.com` from `B2_REGION`; never hardcode a region or endpoint.
- Env: `B2_APPLICATION_KEY_ID`, `B2_APPLICATION_KEY`, `B2_BUCKET_NAME`, `B2_REGION`, optional `B2_ENDPOINT`. Secrets in a gitignored 0600 `.env`; never print keys or presigned URLs.
- A key covers a bucket, at most one name prefix, and listed capabilities; a server-side copy needs one key covering source and destination, so same-bucket cross-prefix copies need a bucket-wide key (prefer writing a marker instead). Roles: fleet-api-signer (`writeFiles` only on `raw/`), depot-ingest (write-only `raw/`), ingest-reader, extract-writer, analyst-reader, labeling-tool-reader (`anonymized/` only), label-export-writer, training-reader, checkpoint-writer, sim-writer, release/evidence publisher. Master or admin key only for provisioning.
- Bucket names are globally unique; prefix with an org name and expect collisions.
- Vehicles hold no keys: presigned PUT for splits up to 5 GB, presigned multipart part URLs above that, retain the local split until a separate verifier confirms the stored version's byte count and full-file SHA-256, and a lifecycle rule (`daysFromStartingToCancelingUnfinishedLargeFiles`) for abandoned uploads.
- Event Notifications: prefix filters only, HMAC-signed (`X-Bz-Event-Notification-Signature`), at-least-once; verify, ack fast, be idempotent.
- Lifecycle uses B2 semantics (hide, then delete) and matches name prefixes; lay out prefixes so expiring data (checkpoints, synthetic episodes) sits apart from data to keep. Never set `daysFromUploadingToHiding` on Iceberg `warehouse/` or lakeFS `lakefs/`; give them a hidden-only rule (`daysFromHidingToDeleting`) so files the tool deleted are reclaimed. Size checkpoint hide rules above the longest run plus resume window.
- Object Lock (governance default; enabling lock and compliance mode are irreversible) for incident splits, released datasets and models, and release-candidate sim results. Cross-bucket server-side copy needs one key that covers both buckets.
- Presigned URLs are bearer credentials with a 7-day SigV4 maximum; refresh for labeling vendors and long-queued re-sim jobs.
- Anonymize faces and plates before any external labeling access; record the anonymization version in the label manifest.
- Cloud Replication is bucket-to-bucket DR, not delivery.
- Standard B2 plus local NVMe by default; Overdrive only when measured sustained throughput is the bottleneck.
- Use the connection method documented for each tool and the S3 operations required by the workflow. Configure rosbag2, CARLA, and MMDetection3D workers to transfer objects through an S3 SDK or presigned URLs.

## Verify

When operating inside the full repository, use its `AGENTS.md` validation commands. When this skill is installed by itself, use the linked product and integration documentation for configuration and service behavior.
