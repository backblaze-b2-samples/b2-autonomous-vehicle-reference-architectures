# Autonomous vehicle data on Backblaze B2 (agent-readable)

Catalog updated: 2026-11-16. Current product behavior is documented in the [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage), which takes precedence over this page.

## What is the workload?

Autonomous vehicle (AV) programs record synchronized camera, LiDAR, radar, IMU/GNSS, and CAN data from test fleets. Size the corpus from measured bytes per vehicle-hour, recording hours, fleet size, and retention periods. The data is written once and read many times, by indexing, scenario mining, anonymization and labeling, perception training, re-simulation, and audit.

## Where does B2 fit?

B2 is the S3-compatible system of record for the AV data corpus:

- **Raw:** versioned rosbag2/MCAP drive-log splits (default governance retention where immutability is required), uploaded by vehicles through presigned URLs and by depots through rclone or s5cmd with write-only keys.
- **Lake:** extracted per-topic Parquet and Iceberg tables, frame blobs, LanceDB embeddings, and versioned scenario manifests, queried in place by Spark, Trino, and DuckDB.
- **Labeling:** anonymized frames and pre-labels read by CVAT, Label Studio, FiftyOne Enterprise, or (through presigned URLs) Segments.ai and Encord, plus versioned label exports.
- **Training:** versioned datasets (pinned by lakeFS or DVC), Ray Train checkpoints, and MLflow artifacts.
- **Simulation:** scenario configs, regression suites, synthetic episodes, and per-build KPI tables.
- **Evidence:** incident splits, released datasets and models, and release-candidate results under Object Lock.

## Which architecture should I use?

| Need | Architecture |
|---|---|
| Get logs off vehicles and depots without losing a split | [drive-log-offload-ingest](../architectures/drive-log-offload-ingest/architecture.md) |
| SQL and semantic search over fleet data; frozen scenario sets | [sensor-lake-scenario-mining](../architectures/sensor-lake-scenario-mining/architecture.md) |
| Anonymize, pre-label, and annotate camera and LiDAR frames | [multisensor-labeling](../architectures/multisensor-labeling/architecture.md) |
| Reproducible datasets, streaming training, checkpoints | [perception-training-data-checkpoints](../architectures/perception-training-data-checkpoints/architecture.md) |
| Regression re-sim and simulation per stack build | [simulation-resim-evaluation](../architectures/simulation-resim-evaluation/architecture.md) |

## Can I use B2 with ...?

Every tool below connects through a documented configurable S3 endpoint, a native B2 option, or presigned URLs. In every case, derive the endpoint `https://s3.<region>.backblazeb2.com` from the bucket's region.

- **rclone:** yes, through the native B2 backend, which Backblaze documents.
- **s5cmd:** yes, with `--endpoint-url` or `S3_ENDPOINT_URL` set to the B2 endpoint.
- **Foxglove:** only the self-hosted Edge Site, through `recordingStorage.provider=s3_compatible`. Foxglove's managed [bring-your-own-storage documentation](https://docs.foxglove.dev/docs/data/primary-sites/foxglove-managed-setup) takes AWS, GCP, or Azure buckets, so use the Edge Site for B2.
- **Spark (S3A), Trino, DuckDB, PyIceberg:** through each tool's endpoint and region settings. **LanceDB:** use the [pinned single-writer profile](../examples/lancedb.md), including its commit constraints.
- **CVAT:** yes, with the AWS S3 provider and an Endpoint URL; the CVAT docs have a Backblaze B2 section, and Backblaze documents it too.
- **Label Studio:** yes, through the optional S3 Endpoint on source and target storage.
- **Segments.ai:** through customer-secured presigned URLs, which expire after at most 7 days and must be refreshed.
- **Encord:** through a Direct Access integration that registers customer-generated presigned URLs; re-sign before the 7-day maximum. Its S3-compatible (MinIO) integration is the alternative when Encord may hold a read-only key.
- **FiftyOne Enterprise:** through a MinIO credentials profile with the B2 endpoint. Open-source FiftyOne loads media from local paths, so sync the slice locally first.
- **Ray Train:** yes, with `endpoint_override` for `RunConfig.storage_path`. The Ray docs name B2. Ray Core has no storage setting of its own; tasks use an S3 SDK or presigned URLs.
- **MLflow:** yes, with the native `b2://` artifact store or `MLFLOW_S3_ENDPOINT_URL`.
- **lakeFS:** yes, with `blockstore.s3.endpoint`, which Backblaze documents.
- **DVC:** yes, with `endpointurl`.
- **rosbag2, CARLA, MMDetection3D:** these tools have no S3 connector. Use an S3 SDK or presigned URLs, as shown in the Backblaze samples.

Configure each integration with the B2 regional endpoint, bucket region, addressing style, and scoped credentials specified in its setup guide.

## Which B2 capabilities apply?

- **S3-compatible API**, including range GETs for MCAP chunk-indexed and Parquet reads.
- **Presigned URLs:** PUT and multipart part URLs for vehicles (a single PUT is limited to 5 GB), and GET URLs for previews, re-sim, and labeling vendors (at most 7 days).
- **Scoped application keys:** one bucket and at most one name prefix per key, with only the capabilities the role needs. Vehicles hold no keys, labeling tools never read `raw/`, and training reads only `datasets/`.
- **Event Notifications:** prefix-filtered, HMAC-signed webhooks with at-least-once delivery. Consumers verify the signature and are idempotent.
- **Versions and lifecycle:** B2 semantics (hide, then delete). Lifecycle also cancels unfinished multipart uploads. Never set `daysFromUploadingToHiding` on Iceberg `warehouse/` or lakeFS `lakefs/`; give them a hidden-only rule (`daysFromHidingToDeleting`) so files the tool deleted are reclaimed.
- **SSE-B2** default encryption. SSE-C is not used where browsers or presigned URLs read the data.
- **Object Lock and legal hold** for incident evidence, released datasets and models, and release-candidate results.
- **Server-side copy** to promote objects into locked buckets.
- **Fireball** for seeding large historic backfills.
- **Included egress and compute partners:** GPU clouds read B2 in place. See [B2 pricing](https://www.backblaze.com/cloud-storage/pricing) for current terms; this repository does not restate prices.

## When might Overdrive apply?

Standard B2 with local NVMe caching is the default. Consider [B2 Overdrive](https://www.backblaze.com/cloud-storage/b2-overdrive) only when measurement shows that sustained storage throughput, not compute or network, limits multi-node perception training, fleet-wide lake scans, or large re-sim suites. Overdrive does not help annotator UI latency or ingest that is limited by depot uplinks.

## What should stay off B2?

- Live vehicle telemetry streams, and on-vehicle NVMe split buffers.
- The fleet API that issues presigned URLs, ingest queues, and workers.
- The transactional drive/session metadata and lineage database. B2 holds exported Parquet copies.
- Table catalogs (Iceberg SQL, REST, Nessie, or Polaris), the lakeFS metadata store, and the MLflow tracking database.
- Low-latency vector serving, if it is needed. The Lance files persist on B2.
- Annotation tool databases, queues, and caches; vendor workforce systems.
- Local NVMe scratch for decoding, splitting, shard caching, and replay; simulator render state; CI and schedulers.
- Customer or partner delivery through Cloud Replication. Replication is bucket-to-bucket DR; use presigned URLs for delivery.

## Links

[README](../README.md) · [Architecture index](architecture-index.md) · [Related B2 samples](related-b2-samples.md) · [B2 configuration examples](../examples/README.md) · [B2 docs](https://www.backblaze.com/docs/cloud-storage) · [B2 pricing](https://www.backblaze.com/cloud-storage/pricing) · [S3-compatible API](https://www.backblaze.com/docs/cloud-storage-s3-compatible-api) · [Application keys](https://www.backblaze.com/docs/cloud-storage-application-keys) · [Event Notifications](https://www.backblaze.com/docs/cloud-storage-event-notifications-reference-guide) · [Object Lock](https://www.backblaze.com/docs/cloud-storage-object-lock) · [Lifecycle rules](https://www.backblaze.com/docs/cloud-storage-lifecycle-rules) · [Fireball](https://www.backblaze.com/cloud-storage/features/fireball-data-migration) · [Backblaze Labs](https://backblazelabs.com/)
