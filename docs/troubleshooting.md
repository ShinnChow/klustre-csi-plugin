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
- Disable or ignore `lfs`-based health probe output until the probe mechanism is reworked.

### Status

Known limitation. Fix will land in a future release — either by using a statically-linked probe or by replacing the probe mechanism with one that does not require exec'ing host glibc binaries via `LD_LIBRARY_PATH`. Non-blocking for mount operations.
