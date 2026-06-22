---
name: pingcapmirror-gomod-hotfix
description: "Hotfix missing Go module zip files in `gs://pingcapmirror/gomod/...` when Bazel/CI fails with 404s from internal module mirrors. Use this to verify the exact module artifact, upload the correct zip from `proxy.golang.org`, and validate the mirror."
---

# Pingcapmirror Gomod Hotfix

## When to use

Use this skill when CI fails fetching a Go module zip from internal mirrors, for example:

```text
WARNING: Download from https://cache.hawkingrei.com/gomod/... failed: GET returned 404 Not Found
WARNING: Download from https://storage.googleapis.com/pingcapmirror/gomod/... failed: GET returned 404 Not Found
```

Typical Bazel error:

```text
Error downloading [https://cache.hawkingrei.com/gomod/..., https://storage.googleapis.com/pingcapmirror/gomod/...]: GET returned 404 Not Found
```

This usually means the required Go module zip exists upstream, but was never mirrored into `pingcapmirror`.

## Key rules

- Do not use a GitHub source archive zip. Bazel expects the Go module zip format, not a GitHub repo snapshot.
- Always download from `proxy.golang.org` using the module pseudo-version from `go.mod` and `DEPS.bzl`.
- Always verify the zip `sha256` matches the value in `DEPS.bzl` before upload.
- Confirm the target GCS object path exactly matches the URL Bazel is trying to fetch.
- Validate the public `storage.googleapis.com/pingcapmirror/...` URL returns `200` before calling the hotfix done.

## Example incident

The confirmed working pattern for `github.com/pingcap/tipb` was:

- module version: `v0.0.0-20260617071407-7c071244534b`
- proxy zip:
  `https://proxy.golang.org/github.com/pingcap/tipb/@v/v0.0.0-20260617071407-7c071244534b.zip`
- mirror object:
  `gs://pingcapmirror/gomod/github.com/pingcap/tipb/com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip`

## Workflow

1. Confirm the CI error is a missing internal `gomod` object, not a Bazel remote-cache `ac/` or `cas/` issue.

Look for URLs like:

```text
https://cache.hawkingrei.com/gomod/<module-path>/<object>.zip
https://storage.googleapis.com/pingcapmirror/gomod/<module-path>/<object>.zip
```

If the failing path is under `ac/` or `cas/`, use the Bazel GCS cache cleanup skills instead.

2. Inspect the PR diff and identify the exact upgraded module version.

```bash
gh pr diff <PR> --repo pingcap/tidb | sed -n '/^diff --git a\/DEPS.bzl/,/^diff --git /p;/^diff --git a\/go.mod/,/^diff --git /p;/^diff --git a\/go.sum/,/^diff --git /p'
```

Capture:

- module path, for example `github.com/pingcap/tipb`
- pseudo-version, for example `v0.0.0-20260617071407-7c071244534b`
- expected `sha256` from `DEPS.bzl`
- expected `strip_prefix`
- exact mirror object name, for example `com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip`

3. Verify the internal mirror really is missing the object.

```bash
curl -I -L --max-time 20 https://storage.googleapis.com/pingcapmirror/gomod/<module-path>/<object>.zip
curl -I -L --max-time 20 https://cache.hawkingrei.com/gomod/<module-path>/<object>.zip
```

Expected pre-hotfix result: `404`.

4. Verify the canonical Go module zip exists upstream.

```bash
curl -I -L --max-time 20 https://proxy.golang.org/<module-path>/@v/<version>.zip
curl -I -L --max-time 20 https://proxy.golang.org/<module-path>/@v/<version>.mod
curl -I -L --max-time 20 https://proxy.golang.org/<module-path>/@v/<version>.info
```

At minimum, the zip must return `200`.

5. Download the module zip from `proxy.golang.org` and verify its `sha256`.

```bash
tmpdir=$(mktemp -d)
cd "$tmpdir"

curl -fLo <object>.zip \
  https://proxy.golang.org/<module-path>/@v/<version>.zip

shasum -a 256 <object>.zip
```

The hash must exactly match the `sha256` declared in `DEPS.bzl`.

Optional sanity check:

```bash
unzip -l <object>.zip | sed -n '1,20p'
```

The root prefix should match `strip_prefix`, for example:

```text
github.com/pingcap/tipb@v0.0.0-20260617071407-7c071244534b/...
```

6. Verify local GCP auth and bucket access.

```bash
gcloud auth list --filter=status:ACTIVE --format='value(account)'
gcloud config get-value project
gcloud storage ls gs://pingcapmirror/gomod/<module-path-dir>/ | tail -n 20
```

If the user already set the project, do not change it unless necessary.

7. Upload the verified zip to `pingcapmirror`.

```bash
gcloud storage cp <object>.zip \
  gs://pingcapmirror/gomod/<module-path-dir>/<object>.zip
```

Example:

```bash
gcloud storage cp \
  /tmp/com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip \
  gs://pingcapmirror/gomod/github.com/pingcap/tipb/com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip
```

8. Verify the uploaded object metadata and public URL.

```bash
gcloud storage ls -L gs://pingcapmirror/gomod/<module-path-dir>/<object>.zip

curl -I --max-time 20 \
  https://storage.googleapis.com/pingcapmirror/gomod/<module-path-dir>/<object>.zip
```

Expected post-hotfix result: `HTTP 200`.

9. Retest the CI job.

If the failing job is in GitHub Actions:

```bash
gh run rerun <run-id> --failed
```

Or ask the PR author to comment `retest` if the job is driven by Prow/Jenkins automation.

## Path mapping rules

Given:

- module path: `github.com/pingcap/tipb`
- version: `v0.0.0-20260617071407-7c071244534b`

The URLs and object names are:

```text
proxy zip:
https://proxy.golang.org/github.com/pingcap/tipb/@v/v0.0.0-20260617071407-7c071244534b.zip

mirror GCS object:
gs://pingcapmirror/gomod/github.com/pingcap/tipb/com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip

mirror public URL:
https://storage.googleapis.com/pingcapmirror/gomod/github.com/pingcap/tipb/com_github_pingcap_tipb-v0.0.0-20260617071407-7c071244534b.zip
```

The object basename generally comes from `DEPS.bzl`, not from hand-built string guessing.

## Quick checklist

- CI is failing on `gomod/...zip` `404`, not `ac/` or `cas/`
- exact module version identified from PR diff
- `DEPS.bzl` `sha256` captured
- internal mirror URL confirmed `404`
- `proxy.golang.org` zip confirmed `200`
- downloaded zip `sha256` matches `DEPS.bzl`
- zip uploaded to exact `gs://pingcapmirror/...` path
- public mirror URL confirmed `200`
- CI rerun triggered

## Notes

- Uploading to `pingcapmirror` is a pragmatic hotfix. It unblocks CI without changing code.
- If the same artifact is also expected behind `cache.hawkingrei.com`, that layer may catch up later through its own sync path. The immediate unblock is the `storage.googleapis.com/pingcapmirror/...` fallback returning `200`.
- If multiple new module versions were introduced in the same PR, repeat the same verification and upload flow for each missing mirror object.
