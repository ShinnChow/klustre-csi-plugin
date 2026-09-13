# Troubleshooting

## musl/glibc mismatch: `lfs`-based probes silently fail

### Symptom

The node plugin container attempts to run host-side binaries such as `lfs --version`
via `LD_LIBRARY_PATH=/host/lib:/host/lib64:/host/usr/lib:/host/usr/lib64`. On nodes
with a glibc-linked `lfs` binary, the exec returns a relocation error or exits
silently. Downstream health checks may incorrectly report the Lustre client as absent
even when it is correctly installed and functional.

### Cause

The plugin container image is built as a statically-linked musl binary. Executing
host-side glibc-linked binaries from inside the container via `LD_LIBRARY_PATH` fails
due to a dynamic-linker (ldso) ABI mismatch between musl and glibc. This affects the
`PATH` and `LD_LIBRARY_PATH` environment variables set on the CSI node DaemonSet
(see [`manifests/daemonset-klustre-csi-node.yaml`](../manifests/daemonset-klustre-csi-node.yaml)
lines 41-44).

Lustre mounts themselves are **not affected** -- `mount.lustre` is invoked via
`nsenter`, which runs fully host-side and does not inherit the container's dynamic
linker environment.

### Workaround

- Rely on `mount.lustre` and `nsenter`-based mount operations (already the default).
- Disable or ignore `lfs`-based health probe output until the probe mechanism is
  reworked.
- Monitor the linked GitHub issue below for status on a permanent fix.

### Reference

<!-- MANUAL ACTION REQUIRED: File a GitHub issue at
     https://github.com/klustrefs/klustre-csi-plugin/issues/new
     using the template below, then replace this comment with a link to the
     resulting issue. -->

**Issue template** (paste when filing manually):

```
Title: Node plugin: lfs-based probes silently fail due to musl/glibc mismatch

Labels: bug, csi-node, follow-up

## Summary
The node plugin container (musl-linked) attempts to exec host glibc binaries via
LD_LIBRARY_PATH, which fails with a relocation error. lfs-based health probes
silently report failure even on nodes with a working Lustre client.

## Reproduction
Observed on a 3-VM KVM rig: Lustre 2.17.0 (ldiskfs), k3s node, klustre-csi-plugin
DaemonSet running. Mount operations succeed (nsenter path), but `lfs --version`
returns a glibc relocation error from inside the musl container.

Relevant manifest: `manifests/daemonset-klustre-csi-node.yaml` lines 41-44
(PATH and LD_LIBRARY_PATH envs pointing to `/host/*` paths).

## Impact
Non-blocking for mount operations. Health probes may report false negatives.

## Workaround
Use nsenter-based mounts (default). Ignore lfs-probe output.

## Acceptance criteria for fix
- `lfs` binary (or equivalent probe) works correctly from inside the musl
  container, OR
- Health probe mechanism is replaced with one that does not require exec'ing
  host binaries via `LD_LIBRARY_PATH`.

## See also
`klustrefs-plan.md` section 1: "nsenter runs fully host-side, but lfs-based health probes
silently fail. Tracked as a follow-up issue."
```
