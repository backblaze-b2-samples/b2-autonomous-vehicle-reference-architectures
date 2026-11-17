# B2 configuration examples

Configuration templates for the buckets in these [reference architectures](../docs/architecture-index.md). Run command blocks from this repository's `examples/` directory using Bash, the B2 CLI v4, and `jq`. Authenticate the CLI with a provisioning credential before creating buckets or application keys. Configuration fields are documented in the [B2 API reference](https://www.backblaze.com/apidocs).

Set `ORG` to a globally unique bucket-name prefix. Set the additional variables used by the selected command block, and replace values shown in `<angle brackets>` before running it. Choose retention periods, lifecycle settings, and application-key capabilities for the workload; lifecycle rules permanently delete expired object versions.

| Variable | Purpose |
|---|---|
| `ORG` | Globally unique bucket-name prefix. |
| `DEPOT` | Depot identifier used in the ingest-key name. |
| `NAME` | Dataset or release name used in the scoped object prefix. |
| `VERSION` | Dataset or model release version used in the scoped object prefix. |

| File | Applies to | Purpose | Architectures |
|---|---|---|---|
| `lifecycle-raw.json` | `<org>-av-raw` | Keep `raw/`, cancel unfinished multipart uploads after 7 days, expire `quarantine/` | [drive-log offload](../architectures/drive-log-offload-ingest/architecture.md) |
| `event-notifications.json` | `<org>-av-raw` | Signed webhook on new `raw/` objects. Copy it for `<org>-av-label` with prefix `labels/` | [drive-log offload](../architectures/drive-log-offload-ingest/architecture.md), [labeling](../architectures/multisensor-labeling/architecture.md) |
| `object-lock-evidence.json` | `<org>-av-evidence`, `<org>-av-release` | Default governance retention | [drive-log offload](../architectures/drive-log-offload-ingest/architecture.md), [training](../architectures/perception-training-data-checkpoints/architecture.md), [re-sim](../architectures/simulation-resim-evaluation/architecture.md) |
| `lifecycle-lake.json` | `<org>-av-lake` | Expire non-current `extracted/` versions. On `warehouse/`, delete only files Iceberg already deleted (hidden-only rule; never set `daysFromUploadingToHiding` there) | [sensor lake](../architectures/sensor-lake-scenario-mining/architecture.md) |
| `lifecycle-datasets.json` | `<org>-av-datasets` | Hidden-only rule on `lakefs/`: delete objects lakeFS garbage collection already deleted. `datasets/` has no rule | [training](../architectures/perception-training-data-checkpoints/architecture.md) |
| `cors-labeling.json` | `<org>-av-label` | Exact-origin browser GET/HEAD (with Range) for presigned media | [labeling](../architectures/multisensor-labeling/architecture.md) |
| `lifecycle-train-sim.json` | `<org>-av-train`, `<org>-av-sim` | Expire intermediate `checkpoints/` and bulk `sim/episodes/`. Each bucket contains only one of the two prefixes, so the other rule does nothing there | [training](../architectures/perception-training-data-checkpoints/architecture.md), [re-sim](../architectures/simulation-resim-evaluation/architecture.md) |
| `sse.json` | all buckets | SSE-B2 default encryption | all |

### Why `warehouse/` and `lakefs/` get hidden-only rules

B2 keeps every version. Iceberg snapshot expiry and orphan-file cleanup, and lakeFS garbage collection, call S3 `DeleteObject` without a version ID. On B2 that only hides the file, so without a rule their storage is never reclaimed. A rule with `daysFromUploadingToHiding: null` and `daysFromHidingToDeleting: N` deletes only files the tool already removed, and N days is the undo window. Never set `daysFromUploadingToHiding` on these prefixes: hiding a file a live snapshot or commit references corrupts the table or repository.

Cloud Replication is not used in these architectures. If you add a DR copy, remember that replication is bucket-to-bucket and is not a delivery mechanism.

## Provision buckets (admin key)

Bucket names are globally unique, so keep an organization prefix and expect collisions. Bucket, lifecycle, CORS, notification, and Object Lock settings require an admin key or the master key. Run provisioning once and keep that key out of runtime services.

```bash
# Object Lock can be enabled at creation or later, but never disabled.
b2 bucket create --file-lock-enabled "$ORG-av-evidence" allPrivate
b2 bucket create --file-lock-enabled "$ORG-av-release"  allPrivate
for b in raw lake label datasets train sim; do b2 bucket create "$ORG-av-$b" allPrivate; done

for b in raw lake label datasets train sim evidence release; do
  b2 bucket update --default-server-side-encryption SSE-B2 "$ORG-av-$b"
done

# One --lifecycle-rule per rule (the plural --lifecycle-rules flag is deprecated in b2 v4.4.1).
apply_lifecycle() {  # <file> <bucket>
  local args=()
  while IFS= read -r rule; do args+=(--lifecycle-rule "$rule"); done < <(jq -c '.[]' "$1")
  b2 bucket update "${args[@]}" "$2"
}
apply_lifecycle lifecycle-raw.json        "$ORG-av-raw"
apply_lifecycle lifecycle-lake.json       "$ORG-av-lake"
apply_lifecycle lifecycle-datasets.json   "$ORG-av-datasets"
apply_lifecycle lifecycle-train-sim.json  "$ORG-av-train"
apply_lifecycle lifecycle-train-sim.json  "$ORG-av-sim"

b2 bucket update --cors-rules "$(cat cors-labeling.json)" "$ORG-av-label"
b2 bucket update --default-retention-mode governance --default-retention-period "365 days" "$ORG-av-evidence"
b2 bucket update --default-retention-mode governance --default-retention-period "365 days" "$ORG-av-release"

# Event Notifications use the native API b2_set_bucket_notification_rules (the key needs
# writeBucketNotifications). Generate the signing secret, keep it in the webhook's secret store,
# and never commit it.
```

## Scoped application keys

Each key below is limited to one bucket and at most one name prefix, with only the capabilities its role needs. `b2 key create` prints the secret once. Store it in a gitignored `0600` file or a secret manager, never in shell history, logs, or CI output.

```bash
# Drive-log offload
b2 key create --bucket "$ORG-av-raw" --name-prefix raw/        fleet-api-signer     writeFiles
b2 key create --bucket "$ORG-av-raw" --name-prefix raw/        "depot-ingest-$DEPOT" listFiles,writeFiles
b2 key create --bucket "$ORG-av-raw" --name-prefix raw/        ingest-reader        listFiles,readFiles
b2 key create --bucket "$ORG-av-raw" --name-prefix quarantine/ ingest-quarantine    writeFiles   # marker objects only
b2 key create --bucket "$ORG-av-raw" --name-prefix catalog/    catalog-exporter     listFiles,writeFiles

# Sensor lake: an extraction job holds ingest-reader (raw/), extract-writer, and iceberg-writer
b2 key create --bucket "$ORG-av-lake" --name-prefix extracted/ extract-writer       listFiles,readFiles,writeFiles
b2 key create --bucket "$ORG-av-lake" --name-prefix warehouse/ iceberg-writer       listFiles,readFiles,writeFiles,deleteFiles
b2 key create --bucket "$ORG-av-lake"                          analyst-reader       listFiles,readFiles

# Labeling (tools never get raw/)
b2 key create --bucket "$ORG-av-label" --name-prefix anonymized/ labeling-tool-reader listBuckets,listFiles,readFiles
b2 key create --bucket "$ORG-av-label" --name-prefix labels/     label-export-writer  listBuckets,listFiles,readFiles,writeFiles

# Training
b2 key create --bucket "$ORG-av-datasets" --name-prefix datasets/    training-reader   listFiles,readFiles
b2 key create --bucket "$ORG-av-train"    --name-prefix checkpoints/ checkpoint-writer listFiles,readFiles,writeFiles,deleteFiles
b2 key create --bucket "$ORG-av-datasets" --name-prefix "datasets/$NAME/$VERSION/" --duration 86400 partner-delivery-signer listFiles,readFiles

# Re-sim
b2 key create --bucket "$ORG-av-sim" --name-prefix sim/ sim-writer listFiles,writeFiles
```

A server-side copy needs one key that can read the source and write the destination. Within one bucket, a key holds at most one name prefix, so `ingest-reader` plus `ingest-quarantine` cannot copy `raw/` to `quarantine/`. The drive-log architecture therefore writes a quarantine marker instead of copying. If you must move the bytes, give the ingest worker a bucket-wide key (no `--name-prefix`) with `listFiles,readFiles,writeFiles`, or download and re-upload.

A server-side copy between buckets (for example `raw` to `evidence`, `datasets` to `release`, or `sim` to `evidence`) needs one key that can read the source and write the destination. b2 v4.4.1 accepts `--bucket` more than once. Create a dedicated publisher key for the release or evidence pipeline that covers just those two buckets and has `listFiles,readFiles,writeFiles,writeFileRetentions,writeFileLegalHolds`. This is the most privileged runtime key, so run it only in that pipeline. Create it without a name prefix and restrict it by bucket.

Governance-mode retention can be removed by a key with `bypassGovernance`, so do not give that capability to runtime keys. Compliance mode cannot be shortened. Set retention periods from your safety-case, contractual, and legal requirements; the 365-day value here is a placeholder.

## LanceDB embedding tables

Use the [LanceDB connection profile](lancedb.md) for pinned dependencies, credential scope and single-writer commit requirements.
