# B2 reference architectures for autonomous vehicles

How autonomous vehicle teams use Backblaze B2 across drive-log offload, sensor lakes and scenario mining, multi-sensor labeling, perception training data and checkpoints, and re-simulation. Each architecture documents the data flow, B2 zones and security boundaries, and links out to the third-party integrations and B2 sample applications that implement it.

Start with the [architecture index](docs/architecture-index.md) or the [agent-readable summary](docs/autonomous-vehicles.md). AI agents: see [For AI agents](#for-ai-agents). Current B2 product behavior is documented at [backblaze.com/docs](https://www.backblaze.com/docs/cloud-storage).

## Where does Backblaze B2 fit for autonomous vehicles?

AV programs record synchronized camera, LiDAR, radar, IMU/GNSS, and CAN data. Size ingest and retention from measured bytes per vehicle-hour, recording hours, fleet size, and the available offload window. B2 stores the drive logs and derived datasets for indexing, labeling, training, simulation, and audit.

- **Store in B2:** versioned raw drive logs (rosbag2/MCAP), extracted Parquet and Iceberg tables, frame blobs, embeddings, scenario sets, anonymized frames, versioned labels and datasets, checkpoints, and simulation results.
- **Read in place:** Spark, Trino, DuckDB, Iceberg, and LanceDB for the lake; CVAT, Label Studio, and FiftyOne Enterprise for labeling; Ray, MLflow, lakeFS, and DVC for training and evaluation. Each reaches B2 through a documented S3 endpoint, a native B2 option, or presigned URLs.
- **Ingest safely:** vehicles upload with short-lived presigned URLs and never hold keys. Depots use rclone or s5cmd with write-only keys scoped to `raw/`, and Fireball seeds large backfills.
- **Automate and protect:** Event Notifications start extraction and validation. Object Lock covers incident evidence, released datasets and models, and safety-case results.
- **Keep elsewhere:** metadata and lineage databases, table catalogs, low-latency vector serving, annotation-tool databases, queues and orchestration, live telemetry, and local NVMe scratch.

Pricing, egress terms, and limits change: see [B2 pricing](https://www.backblaze.com/cloud-storage/pricing) and the [B2 docs](https://www.backblaze.com/docs/cloud-storage).

## Architectures

<!-- catalog:architectures:start -->
| Architecture | Purpose | Samples and setup |
|---|---|---|
| [Offload drive logs from vehicles and depots into a versioned raw zone](architectures/drive-log-offload-ingest/architecture.md) | Upload closed rosbag2/MCAP log splits from vehicles, depot ingest stations, or shipped drives into a versioned B2 raw zone, verify each split before deleting the local copy, send only trigger-flagged event windows over the air, and start downstream extraction from object-created events. | [Rosbag2 Cloud Offload](https://github.com/backblaze-b2-samples/rosbag2-cloud-offload) · [Roboflow Inference Frame Archive](https://github.com/backblaze-b2-samples/roboflow-inference-frame-archive) · [B2 Event Broker](https://github.com/backblaze-b2-samples/b2-event-broker) · [b2listen](https://github.com/backblaze-b2-samples/b2listen) · [setup](https://www.backblaze.com/docs/cloud-storage-integrate-rclone-with-backblaze-b2) |
| [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) | Extract raw logs into per-topic Parquet/Iceberg tables and frame blobs on B2, query them in place with Spark, Trino, or DuckDB, find rare scenarios with embedding search over keyframes, and freeze results as versioned scenario-set manifests. | [CLIP Video Scene Search](https://github.com/backblaze-b2-samples/clip-video-scene-search) · [YOLO Drone and Dashcam Footage Detection](https://github.com/backblaze-b2-samples/yolo-drone-footage-detection) · [DuckDB Query In Place](https://github.com/backblaze-b2-samples/duckdb-query-in-place) · [Iceberg Feature Store](https://github.com/backblaze-b2-samples/iceberg-feature-store) · [Lance Multimodal Search](https://github.com/backblaze-b2-samples/lance-multimodal-search) · [nuScenes Sensor Data Lake](https://github.com/backblaze-b2-samples/nuscenes-sensor-data-lake) · [KISS-ICP LiDAR Archive](https://github.com/backblaze-b2-samples/kiss-icp-lidar-archive) · [setup](https://trino.io/docs/current/object-storage/file-system-s3.html) |
| [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) | Blur faces and license plates in selected camera frames, run model-assisted 2D and 3D pre-labeling, have humans refine labels in tools that read media from B2 directly or through presigned URLs, and write versioned label releases back to B2. | [SAM 2 Mask Dataset Builder](https://github.com/backblaze-b2-samples/sam2-mask-dataset-builder) · [ByteTrack Dataset Builder](https://github.com/backblaze-b2-samples/bytetrack-dataset-builder) · [YOLO11 Batch Detection Pipeline](https://github.com/backblaze-b2-samples/yolo11-batch-detection-pipeline) · [B2 Event Broker](https://github.com/backblaze-b2-samples/b2-event-broker) · [MMDetection3D LiDAR Dataset](https://github.com/backblaze-b2-samples/mmdetection3d-lidar-dataset) · [setup](https://www.backblaze.com/docs/cloud-storage-connect-backblaze-b2-cloud-storage-to-cvat) |
| [Version perception datasets, stream them to training, and track checkpoints](architectures/perception-training-data-checkpoints/architecture.md) | Freeze labeled scenarios into immutable dataset versions (manifests plus shards) on B2, pin them with lakeFS or DVC, stream shards directly into training, and persist Ray Train checkpoints and MLflow run artifacts to B2. | [WebDataset Streaming PyTorch Training](https://github.com/backblaze-b2-samples/webdataset-streaming-pytorch-training) · [lakeFS on B2 Quickstart](https://github.com/backblaze-b2-samples/lakefs-on-b2-quickstart) · [fastdup Dataset Cleaning](https://github.com/backblaze-b2-samples/fastdup-dataset-cleaning) · [nuScenes Sensor Data Lake](https://github.com/backblaze-b2-samples/nuscenes-sensor-data-lake) · [MMDetection3D LiDAR Dataset](https://github.com/backblaze-b2-samples/mmdetection3d-lidar-dataset) · [setup](https://docs.ray.io/en/latest/train/user-guides/persistent-storage.html) |
| [Re-simulate real drives and synthetic scenarios against each stack build and keep comparable KPIs](architectures/simulation-resim-evaluation/architecture.md) | Store scenario definitions and a frozen regression suite on B2, replay real log splits to re-sim workers from presigned URLs, write simulator episodes back to B2, and keep per-build KPI results immutable and comparable across releases. | [Rosbag2 Cloud Offload](https://github.com/backblaze-b2-samples/rosbag2-cloud-offload) · [CARLA Sensor Data Lake](https://github.com/backblaze-b2-samples/carla-sensor-data-lake) · [setup](https://www.backblaze.com/docs/cloud-storage-connect-backblaze-b2-to-mlflow) |
<!-- catalog:architectures:end -->

## Third-party integrations

Use the connection method and settings listed for each integration. Configure S3 clients with `https://s3.<region>.backblazeb2.com`, derived from the bucket's region; other workflows use a native B2 backend, presigned URLs, or a worker that transfers closed files.

<!-- catalog:integrations:start -->
| Tool | Role | Connection type | How it connects to B2 | Architectures |
|---|---|---|---|---|
| [rclone](https://rclone.org/b2/) | Bulk copy and sync of drive-log folders from depot NAS or ingest stations | Built-in Backblaze B2 option | Native Backblaze B2 backend (type = b2) with a write-only key scoped to raw/; the S3 backend with the B2 endpoint also works ([setup guide](https://www.backblaze.com/docs/cloud-storage-integrate-rclone-with-backblaze-b2)) | [Offload drive logs from vehicles and depots into a versioned raw zone](architectures/drive-log-offload-ingest/architecture.md) |
| [s5cmd](https://github.com/peak/s5cmd) | High-parallelism bulk upload and sync from depot ingest stations | S3 sync tool | Copy closed splits from local disk with --endpoint-url (or S3_ENDPOINT_URL) set to https://s3.<region>.backblazeb2.com and the depot's write-only key; size --numworkers, cp --concurrency, and --part-size to the depot uplink ([setup guide](https://github.com/peak/s5cmd#readme)) | [Offload drive logs from vehicles and depots into a versioned raw zone](architectures/drive-log-offload-ingest/architecture.md) |
| [Foxglove Edge Site (self-hosted)](https://docs.foxglove.dev/docs/data/edge-sites/installation) | Recording indexing, preview, and engineer triage near the data | S3 custom endpoint | recordingStorage.provider=s3_compatible with recordingStorage.s3CompatibleServiceUrl set to the B2 S3 endpoint and an HMAC key pair in the recording-storage-credentials secret; set recordingStorage.s3CompatibleServiceRegion to the bucket region | [Offload drive logs from vehicles and depots into a versioned raw zone](architectures/drive-log-offload-ingest/architecture.md) |
| [Apache Spark (Hadoop S3A)](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/index.html) | Log decoding, extraction, and large-scale scene detection | S3 custom endpoint | fs.s3a.endpoint=https://s3.<region>.backblazeb2.com with fs.s3a.endpoint.region and fs.s3a.path.style.access=true, plus per-bucket endpoint and credential overrides for the raw and lake buckets ([setup guide](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/third_party_stores.html)) | [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) |
| [Trino](https://trino.io/) | Interactive SQL over Iceberg, Hive, or Delta tables on B2 | S3 custom endpoint | Native S3 file system (fs.s3.enabled=true) with s3.endpoint set to B2, s3.region set to the bucket region, and s3.path-style-access=true in each Iceberg, Hive, or Delta Lake catalog ([setup guide](https://trino.io/docs/current/object-storage/file-system-s3.html)) | [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) |
| [DuckDB (httpfs)](https://duckdb.org/docs/current/core_extensions/httpfs/s3api.html) | Ad hoc analyst queries over Parquet in place | S3 custom endpoint | S3 secret with ENDPOINT set to the B2 host name (s3.<region>.backblazeb2.com, no scheme), REGION, URL_STYLE 'path', and a read-only lake key | [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) |
| [Apache Iceberg (PyIceberg)](https://py.iceberg.apache.org/configuration/) | Table format for extracted sensor and event tables | S3 custom endpoint | FileIO s3.endpoint set to the B2 endpoint and s3.region to the bucket region; run the SQL or REST catalog separately | [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) |
| [LanceDB / Lance](https://lancedb.com/) | Keyframe embedding table and ANN index stored next to the media | S3 custom endpoint | Use the pinned LanceDB 0.20.0 / pylance 0.23.2 connection profile in examples/lancedb.md, with the B2 endpoint, region and scoped credentials; serialize all table mutations through one writer service ([setup guide](https://github.com/lance-format/lance/blob/v0.23.2/rust/lance-table/src/io/commit.rs)) | [Decode drive logs into a queryable sensor lake and mine long-tail scenarios](architectures/sensor-lake-scenario-mining/architecture.md) |
| [CVAT](https://docs.cvat.ai/docs/workspace/attach-cloud-storage/) | 2D and 3D (point cloud) annotation and review | S3 custom endpoint | Attach cloud storage with the Amazon S3 provider, the required Endpoint URL set to the B2 endpoint, and a read-only key scoped to anonymized/ ([setup guide](https://www.backblaze.com/docs/cloud-storage-connect-backblaze-b2-cloud-storage-to-cvat)) | [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) |
| [Label Studio](https://labelstud.io/guide/storage_s3) | Image, video, and multimodal annotation with separate source and target storage | S3 custom endpoint | Amazon S3 source and target storage with the S3 Endpoint field set to the B2 endpoint; a read-only key on anonymized/ for source and a separate writer on labels/ for target | [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) |
| [Segments.ai](https://docs.segments.ai/how-to-integrate/import-data) | Multi-sensor (camera and LiDAR) labeling by a managed workforce | Presigned URLs | Customer-secured URLs; your service signs B2 GET URLs with a read-only anonymized/ key and registers them on samples, refreshing open samples before the 7-day SigV4 maximum | [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) |
| [Encord](https://docs.encord.com/platform-documentation/General/annotate-data-integrations/annotate-direct-access-integration) | Managed 2D, video, and multi-sensor annotation over customer-hosted media | Presigned URLs | Direct Access integration; register files with a JSON whose objectUrl values are presigned B2 GET URLs signed by your service, add bucket CORS for the Encord origin, and re-sign before the 7-day SigV4 maximum. Encord's S3-compatible (MinIO) integration is the alternative when Encord may hold a read-only key and sign URLs itself | [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) |
| [FiftyOne Enterprise](https://docs.voxel51.com/enterprise/installation.html) | Label QA, error mining, and dataset slicing | S3 custom endpoint | MinIO credentials profile (MINIO_ENDPOINT_URL or an ini endpoint_url) set to the B2 endpoint, plus region and a read-only key; media paths use the endpoint as their prefix | [Anonymize, pre-label, and annotate camera and LiDAR frames, then publish versioned label sets](architectures/multisensor-labeling/architecture.md) |
| [Ray (Train)](https://docs.ray.io/en/latest/train/user-guides/persistent-storage.html) | Distributed training with checkpoints and run state persisted to B2 | S3 custom endpoint | Set RunConfig storage_path to an s3:// URI with endpoint_override=https://s3.<region>.backblazeb2.com, or set AWS_ENDPOINT_URL_S3 with a plain s3:// path | [Version perception datasets, stream them to training, and track checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [MLflow](https://mlflow.org/docs/latest/self-hosting/architecture/artifact-store/) | Experiment tracking, per-build evaluation runs, and model registry artifacts | Built-in Backblaze B2 option | Native b2://<bucket>@<endpoint-host>/<path> artifact store, or an s3:// artifact root with MLFLOW_S3_ENDPOINT_URL set to the B2 endpoint; use proxied artifact access so only the tracking server holds the key ([setup guide](https://www.backblaze.com/docs/cloud-storage-connect-backblaze-b2-to-mlflow)) | [Version perception datasets, stream them to training, and track checkpoints](architectures/perception-training-data-checkpoints/architecture.md), [Re-simulate real drives and synthetic scenarios against each stack build and keep comparable KPIs](architectures/simulation-resim-evaluation/architecture.md) |
| [lakeFS](https://docs.lakefs.io/reference/configuration/) | Git-style branches and commits over dataset versions | S3 custom endpoint | Blockstore type s3 with blockstore.s3.endpoint set to the B2 endpoint, blockstore.s3.force_path_style=true, and the bucket region ([setup guide](https://www.backblaze.com/docs/cloud-storage-lakefs-integration-with-backblaze-b2)) | [Version perception datasets, stream them to training, and track checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [DVC](https://doc.dvc.org/user-guide/data-management/remote-storage/amazon-s3) | Pin dataset manifests and shards to git commits | S3 custom endpoint | S3 remote (dvc remote add) with endpointurl set to the B2 endpoint and credentials stored with --local | [Version perception datasets, stream them to training, and track checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
<!-- catalog:integrations:end -->

## Sample applications

Sample applications from [`backblaze-b2-samples`](https://github.com/backblaze-b2-samples) demonstrating these storage workflows. Follow the linked repositories for source code, setup, and operating instructions.

<!-- catalog:samples:start -->
| Sample | Demonstrates | Related architectures |
|---|---|---|
| [Rosbag2 Cloud Offload](https://github.com/backblaze-b2-samples/rosbag2-cloud-offload) | Presigned-PUT offload of closed rosbag2 splits with head_object verification before local delete, a Parquet session catalog, and a presigned-GET replay manifest | [drive-log-offload-ingest](architectures/drive-log-offload-ingest/architecture.md), [simulation-resim-evaluation](architectures/simulation-resim-evaluation/architecture.md) |
| [Roboflow Inference Frame Archive](https://github.com/backblaze-b2-samples/roboflow-inference-frame-archive) | Edge inference that archives only flagged frames and predictions to B2 with Parquet roll-ups | [drive-log-offload-ingest](architectures/drive-log-offload-ingest/architecture.md) |
| [CLIP Video Scene Search](https://github.com/backblaze-b2-samples/clip-video-scene-search) | PySceneDetect segmentation and OpenCLIP text/image scene search over a FAISS index stored on B2 | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [YOLO Drone and Dashcam Footage Detection](https://github.com/backblaze-b2-samples/yolo-drone-footage-detection) | Multipart upload of dashcam or drone video, YOLO detection on keyframes, and GPS-correlated GeoJSON detections on B2 | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [SAM 2 Mask Dataset Builder](https://github.com/backblaze-b2-samples/sam2-mask-dataset-builder) | SAM 2 mask propagation across video, written to B2 as versioned COCO RLE segmentation datasets | [multisensor-labeling](architectures/multisensor-labeling/architecture.md) |
| [ByteTrack Dataset Builder](https://github.com/backblaze-b2-samples/bytetrack-dataset-builder) | Versioned MOT tracking datasets built from raw video on B2 | [multisensor-labeling](architectures/multisensor-labeling/architecture.md) |
| [YOLO11 Batch Detection Pipeline](https://github.com/backblaze-b2-samples/yolo11-batch-detection-pipeline) | Batch 2D detection and segmentation over B2 image and video libraries producing COCO JSON pre-labels and crops | [multisensor-labeling](architectures/multisensor-labeling/architecture.md) |
| [B2 Event Broker](https://github.com/backblaze-b2-samples/b2-event-broker) | Verifies and fans out signed B2 Event Notifications to subscribers | [drive-log-offload-ingest](architectures/drive-log-offload-ingest/architecture.md), [multisensor-labeling](architectures/multisensor-labeling/architecture.md) |
| [b2listen](https://github.com/backblaze-b2-samples/b2listen) | Forwards B2 Event Notifications to local services during development (not listed in the Labs feed) | [drive-log-offload-ingest](architectures/drive-log-offload-ingest/architecture.md) |
| [DuckDB Query In Place](https://github.com/backblaze-b2-samples/duckdb-query-in-place) | SQL over Parquet, CSV, and JSON in B2 through DuckDB httpfs with a custom endpoint | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [Iceberg Feature Store](https://github.com/backblaze-b2-samples/iceberg-feature-store) | PyIceberg tables on B2 with a local SQL catalog, schema evolution, snapshot rollback, and DuckDB time travel | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [Lance Multimodal Search](https://github.com/backblaze-b2-samples/lance-multimodal-search) | A LanceDB vector table and ANN index stored directly on B2 next to the media | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [WebDataset Streaming PyTorch Training](https://github.com/backblaze-b2-samples/webdataset-streaming-pytorch-training) | Tar shards streamed from B2 into PyTorch with worker and node shard splits and no local staging | [perception-training-data-checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [lakeFS on B2 Quickstart](https://github.com/backblaze-b2-samples/lakefs-on-b2-quickstart) | lakeFS branch, commit, and merge over a B2 blockstore (public repo, not listed in the Labs feed) | [perception-training-data-checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [fastdup Dataset Cleaning](https://github.com/backblaze-b2-samples/fastdup-dataset-cleaning) | Duplicate, outlier, and broken-frame detection on B2 with quarantine instead of deletion | [perception-training-data-checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [nuScenes Sensor Data Lake](https://github.com/backblaze-b2-samples/nuscenes-sensor-data-lake) | nuScenes-format camera, LiDAR, and radar blobs on B2 indexed with the devkit and served as presigned per-keyframe training manifests | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md), [perception-training-data-checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [MMDetection3D LiDAR Dataset](https://github.com/backblaze-b2-samples/mmdetection3d-lidar-dataset) | KITTI-style LiDAR turned into 3D detection datasets with pre-labels, manifests, and checkpoints on B2 | [multisensor-labeling](architectures/multisensor-labeling/architecture.md), [perception-training-data-checkpoints](architectures/perception-training-data-checkpoints/architecture.md) |
| [KISS-ICP LiDAR Archive](https://github.com/backblaze-b2-samples/kiss-icp-lidar-archive) | KISS-ICP LiDAR odometry and incremental mapping with raw scans, poses, .ply map snapshots, and trajectories archived per session on B2 | [sensor-lake-scenario-mining](architectures/sensor-lake-scenario-mining/architecture.md) |
| [CARLA Sensor Data Lake](https://github.com/backblaze-b2-samples/carla-sensor-data-lake) | CARLA scenario configs and per-episode sensor frames on B2 served to a PyTorch DataLoader with presigned URLs | [simulation-resim-evaluation](architectures/simulation-resim-evaluation/architecture.md) |
<!-- catalog:samples:end -->

See the [full sample cross-reference](docs/related-b2-samples.md) for selection guidance.

## B2 capability map

| Area | Capabilities | Used in | Selection notes |
|---|---|---|---|
| Access | S3 API, range GETs, presigned URLs, CORS, multipart, scoped keys (one bucket, at most one prefix) | All | Presigned PUT is limited to 5 GB, so larger files use presigned multipart parts. Presigned URLs last at most 7 days. CORS applies only to browser labeling tools |
| Change and lifecycle | Event Notifications, versions, lifecycle (hide, then delete; cancel unfinished multipart) | Offload, lake, labeling, training, re-sim | Webhooks are signed and delivered at least once. Filters match prefixes only. On Iceberg `warehouse/` and lakeFS `lakefs/`, use only hidden-only rules (`daysFromHidingToDeleting`), never `daysFromUploadingToHiding` |
| Protection | SSE-B2, Object Lock, legal hold, server-side copy | Offload, training, re-sim | Governance mode by default. Enabling Object Lock and choosing compliance mode are irreversible. No SSE-C where browsers or presigned URLs read |
| Bulk movement | Fireball | Offload | Seeds historic backfills. Check current capacity on the [Fireball page](https://www.backblaze.com/cloud-storage/features/fireball-data-migration) |
| Delivery | Included egress, compute partners, presigned URLs | Training, re-sim, labeling | GPU clouds read B2 in place. Cloud Replication is bucket-to-bucket DR and is not used for delivery |
| Performance | Standard B2, [Overdrive](https://www.backblaze.com/cloud-storage/b2-overdrive), local NVMe cache | Training, lake, re-sim | Standard plus NVMe by default. Consider Overdrive only when measured sustained read throughput is the bottleneck |

## For AI agents

| Resource | Use it for |
|---|---|
| [llms.txt](llms.txt) | Curated index of architectures, integrations, samples, and official docs |
| [llms-full.txt](llms-full.txt) | The summary and every architecture in one file |
| [catalog/architectures.yaml](catalog/architectures.yaml) | Source of truth for architectures, integrations (with connection configuration), and samples; validated by [JSON Schema](schemas/architecture-catalog.schema.json) |
| [AGENTS.md](AGENTS.md) | Source hierarchy, generated files, validation commands, and operational invariants |
| [skill/SKILL.md](skill/SKILL.md) | Portable agent skill for B2 autonomous vehicle data architecture work |
| [examples/README.md](examples/README.md) | Lifecycle, CORS, Object Lock, Event Notification, encryption, and scoped-key templates |

<!-- catalog:related:start -->
Other B2 reference architecture categories: [AI Data Providers](https://github.com/backblaze-b2-samples/b2-ai-data-provider-reference-architectures), [Generative Video](https://github.com/backblaze-b2-samples/b2-gen-video-reference-architectures), [Image Generation](https://github.com/backblaze-b2-samples/b2-image-generation-reference-architectures), [Robotics](https://github.com/backblaze-b2-samples/b2-robotics-reference-architectures), [Commercial Drones](https://github.com/backblaze-b2-samples/b2-commercial-drone-reference-architectures).
<!-- catalog:related:end -->

The [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage) is authoritative for product behavior; this repository provides workflow-specific designs.

## Repository layout

- `catalog/`: architecture, integration, and sample metadata.
- `schemas/`: the catalog JSON Schema.
- `architectures/`: architecture docs and Mermaid diagrams.
- `docs/`: agent-readable pages and generated indexes.
- `examples/`: B2 configuration templates.
- `skill/`: the portable agent skill.
- `tools/`: the generator and link checker.
- `tests/`: tool tests.
- `.github/`: CI and templates.

This repository is documentation only. Runnable code lives in the linked sample repositories.

## Validate locally

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements-catalog.txt pytest
python3 tools/generate_catalog.py --check
python3 tools/check_markdown_links.py            # add --external to probe outbound links
python3 -m pytest tests
```

## Contributing, security, and license

See [CONTRIBUTING](.github/CONTRIBUTING.md). The catalog is the extension point. Report sensitive issues through the [security policy](.github/SECURITY.md). Released under the [MIT license](LICENSE); cite with [CITATION.cff](CITATION.cff).
