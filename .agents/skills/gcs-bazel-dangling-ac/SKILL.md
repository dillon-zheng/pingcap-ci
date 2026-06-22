---
name: gcs-bazel-dangling-ac
description: "Debug and remediate Bazel remote-cache failures where Jenkins/Prow reports `Missing digest` from the GCS-backed cache, especially after GCS CAS cleanup leaves dangling AC entries. Use this to find the exact `ac/<hash>` that references a missing `cas/<hash>` and delete only that AC."
---

# GCS Bazel Dangling AC

## When to use

Use this skill when a PingCAP CI Jenkins/Prow job fails with Bazel remote cache errors like:

```text
Failed to fetch blobs because they do not exist remotely.
Missing digest: <sha256>/<size> for bazel-out/.../<file>
```

This usually means an Action Cache (`ac/<hash>`) entry still exists and points at a missing CAS blob (`cas/<sha256>`). Deleting broad AC sets by time window is unreliable. Find and delete the exact AC entry.

## Key rules

- Do not assume all AC objects read in the failure window are bad. Bazel reads thousands of AC/CAS objects concurrently.
- Do not use `JSON_VALUE(protopayload_auditlog.requestMetadata.callerIp)`. `callerIp` is a string, and `JSON_VALUE("34.60.203.59")` can incorrectly return `34.60`. Use `protopayload_auditlog.requestMetadata.callerIp` directly.
- Use `bq query --max_rows=...`; default output can silently show only 100 rows.
- Confirm deletion with `gsutil stat` returning 404 before calling it fixed.
- If deleting an AC manually, also remove it from `gcs_cache_object_last_seen_current` so cleanup state matches live GCS.

## Constants

Default bucket and dataset for this incident class:

```bash
BUCKET=pingcap-ci-bazel-remote-cache-us-central1
PROJECT=pingcap-testing-account
DATASET=ci_bazel_cache_logs
LOG_TABLE=cloudaudit_googleapis_com_data_access
LAST_SEEN_TABLE=gcs_cache_object_last_seen_current
```

## Workflow

1. Download the Jenkins console log and extract the missing digest, output path, failing target/action, and timestamp.

```bash
curl -fsSL '<jenkins-build-url>/consoleText' -o /tmp/jenkins-console.txt
rg -n -C 8 'Missing digest|FAILED TO BUILD|remote-cache' /tmp/jenkins-console.txt
```

Capture:

- `MISSING_DIGEST`, for example `1386cd831da726c032323b7b182802b6a9c2c71631b03ebd3a812beb36ec8389`
- `MISSING_PATH`, for example `bazel-out/k8-fastbuild/bin/pkg/util/dbterror/plannererrors/plannererrors_test_/testmain.go`
- failure UTC time, for example `2026-06-18 13:17:58 UTC`

2. Find Bazel GET events for the missing CAS. Exclude local `gsutil` or `curl` checks.

```bash
bq query --nouse_legacy_sql --format=prettyjson --max_rows=1000 <<'SQL' > /tmp/missing-cas-events.json
SELECT
  timestamp,
  protopayload_auditlog.requestMetadata.callerIp AS caller_ip,
  protopayload_auditlog.authenticationInfo.principalEmail AS principal_email,
  protopayload_auditlog.requestMetadata.callerSuppliedUserAgent AS user_agent
FROM `pingcap-testing-account.ci_bazel_cache_logs.cloudaudit_googleapis_com_data_access`
WHERE resource.labels.bucket_name = "pingcap-ci-bazel-remote-cache-us-central1"
  AND timestamp >= TIMESTAMP("<start-utc>")
  AND timestamp < TIMESTAMP("<end-utc>")
  AND protopayload_auditlog.methodName = "storage.objects.get"
  AND REGEXP_EXTRACT(protopayload_auditlog.resourceName, r"/objects/(.+)$") = "cas/<MISSING_DIGEST>"
ORDER BY timestamp
SQL
```

Pick rows where `user_agent` starts with `bazel/` and principal is the CI remote-cache service account. Ignore rows from local users or `gsutil`.

3. For the Bazel failure event, list AC GETs from the exact same raw `caller_ip` in the minute before the missing CAS GET.

```bash
bq query --nouse_legacy_sql --format=csv --max_rows=100000 <<'SQL' > /tmp/ac-candidates.csv
SELECT
  timestamp,
  REGEXP_EXTRACT(protopayload_auditlog.resourceName, r"/objects/(.+)$") AS object_name
FROM `pingcap-testing-account.ci_bazel_cache_logs.cloudaudit_googleapis_com_data_access`
WHERE resource.labels.bucket_name = "pingcap-ci-bazel-remote-cache-us-central1"
  AND timestamp >= TIMESTAMP_SUB(TIMESTAMP("<failure-utc>"), INTERVAL 60 SECOND)
  AND timestamp <= TIMESTAMP("<failure-utc>")
  AND protopayload_auditlog.requestMetadata.callerIp = "<raw-caller-ip>"
  AND protopayload_auditlog.methodName = "storage.objects.get"
  AND STARTS_WITH(REGEXP_EXTRACT(protopayload_auditlog.resourceName, r"/objects/(.+)$"), "ac/")
ORDER BY timestamp, object_name
SQL
```

4. Download candidate AC objects and search their ActionResult bytes. These are protobuf bytes, but `strings`/`rg -a` usually exposes output paths and digest hashes.

```bash
python3 - <<'PY'
import csv
objs = sorted({r["object_name"] for r in csv.DictReader(open("/tmp/ac-candidates.csv")) if r["object_name"].startswith("ac/")})
open("/tmp/ac-candidates.txt", "w").write("".join(o + "\n" for o in objs))
open("/tmp/ac-candidate-urls.txt", "w").write("".join("gs://pingcap-ci-bazel-remote-cache-us-central1/" + o + "\n" for o in objs))
print("unique_ac", len(objs))
PY

rm -rf /tmp/ac-candidates && mkdir -p /tmp/ac-candidates
xargs -n 1 -P 64 -I{} sh -c 'h=${1##*/}; gsutil -q cp "$1" "/tmp/ac-candidates/$h" 2>/dev/null || true' sh {} < /tmp/ac-candidate-urls.txt

rg -a -l '<MISSING_DIGEST>|<distinctive-path-fragment>' /tmp/ac-candidates > /tmp/bad-ac-files.txt || true
```

Use a distinctive path fragment from the error, for example `pkg/util/dbterror/plannererrors|plannererrors_test_`. If `MISSING_DIGEST` is present in an AC, that AC is the primary suspect.

5. Inspect exact hits. The bad AC should show the missing path and missing digest together.

```bash
for f in $(cat /tmp/bad-ac-files.txt); do
  echo "--- ac $(basename "$f")"
  strings "$f" | rg -C 5 '<MISSING_DIGEST>|<distinctive-path-fragment>'
done
```

Example of a true hit:

```text
--- ac 5a294a1a72fbd2a654bdcc79f176f80506de26ffd1cdf4e42eacf0864415fd13
bazel-out/k8-fastbuild/bin/pkg/util/dbterror/plannererrors/plannererrors_test_/testmain.go
@1386cd831da726c032323b7b182802b6a9c2c71631b03ebd3a812beb36ec8389
```

6. Confirm the object is live and audit its history.

```bash
BAD_AC=ac/<bad-ac-hash>
gsutil stat "gs://pingcap-ci-bazel-remote-cache-us-central1/$BAD_AC"

bq query --nouse_legacy_sql --format=prettyjson --max_rows=1000 <<SQL > /tmp/bad-ac-events.json
SELECT
  timestamp,
  protopayload_auditlog.methodName AS method,
  protopayload_auditlog.requestMetadata.callerIp AS caller_ip,
  protopayload_auditlog.authenticationInfo.principalEmail AS principal_email,
  protopayload_auditlog.requestMetadata.callerSuppliedUserAgent AS user_agent
FROM \`pingcap-testing-account.ci_bazel_cache_logs.cloudaudit_googleapis_com_data_access\`
WHERE resource.labels.bucket_name = "pingcap-ci-bazel-remote-cache-us-central1"
  AND timestamp >= TIMESTAMP("<investigation-start-utc>")
  AND REGEXP_EXTRACT(protopayload_auditlog.resourceName, r"/objects/(.+)$") = "$BAD_AC"
ORDER BY timestamp
SQL
```

If the AC has no recent `storage.objects.create`, it was not re-created; it was simply missed by prior cleanup.

7. Delete only the exact bad AC and verify.

```bash
BAD_AC=ac/<bad-ac-hash>
gsutil rm "gs://pingcap-ci-bazel-remote-cache-us-central1/$BAD_AC"
gsutil stat "gs://pingcap-ci-bazel-remote-cache-us-central1/$BAD_AC" || true
sleep 5
gsutil stat "gs://pingcap-ci-bazel-remote-cache-us-central1/$BAD_AC" || true
```

The expected result after deletion is:

```text
No URLs matched: gs://pingcap-ci-bazel-remote-cache-us-central1/ac/<bad-ac-hash>
```

8. Reconcile BigQuery last_seen for this manual delete.

```bash
bq query --nouse_legacy_sql <<SQL
DELETE FROM \`pingcap-testing-account.ci_bazel_cache_logs.gcs_cache_object_last_seen_current\`
WHERE object_name = "$BAD_AC"
SQL

bq query --nouse_legacy_sql --format=csv <<SQL
SELECT COUNT(*) AS row_count
FROM \`pingcap-testing-account.ci_bazel_cache_logs.gcs_cache_object_last_seen_current\`
WHERE object_name = "$BAD_AC"
SQL
```

## If no direct AC hit is found

- Expand the AC GET window from 60 seconds to 120 or 300 seconds for the same raw caller IP.
- Search by output path fragments first; the missing digest may be nested through Tree/Directory CAS objects.
- If needed, extract all 64-char hex digests from candidate AC objects, download those CAS blobs, and search their bytes for the missing digest/path.
- Still delete only an AC after proving it references the missing path or missing digest. Do not delete all ACs from a busy failure window.

## Root cause pattern

This failure class can happen when CAS blobs are deleted independently while AC entries remain hot. A hot AC can keep being read, but its referenced CAS blob may not be read often enough to survive CAS LRU cleanup. The result is a dangling AC: Bazel hits the AC, then fails fetching output blobs.

For steady-state cleanup design, avoid enabling independent CAS deletion unless the implementation can prevent or repair dangling AC references.
