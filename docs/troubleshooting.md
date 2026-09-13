# Troubleshooting

## musl/glibc mismatch: `lfs`-based probes silently fail (fixed)

### Symptom (affects released images up to v0.1.3)

The node plugin container attempts to run host-side binaries such as `lfs --version`
via `LD_LIBRARY_PATH=/host/lib:/host/lib64:/host/usr/lib:/host/usr/lib64`. On nodes
with a glibc-linked `lfs` binary, the exec returns a relocation error or exits
silently. Downstream health checks may incorrectly report the Lustre client as absent
even when it is correctly installed and functional.

Lustre mounts themselves are **not affected** -- `mount.lustre` is invoked via
`nsenter`, which runs fully host-side and does not inherit the container's dynamic
linker environment.

### Cause

The plugin container image is built as a statically-linked musl binary. Executing
host-side glibc-linked binaries from inside the container via `LD_LIBRARY_PATH` fails
due to a dynamic-linker (ldso) ABI mismatch between musl and glibc.

### Fix

All host-side tools used by the plugin (`lfs`, `lsmod`, `modprobe`) are now executed
via `nsenter -t 1 -m` -- the same mechanism already used for `mount.lustre` -- so they
run fully host-side and never cross the musl/glibc dynamic-linker boundary
(see `src/lustre/client.rs`, `host_command`).

Consequently, the `PATH` and `LD_LIBRARY_PATH` `/host/*` environment variables and the
`/host/sbin`, `/host/usr/sbin`, `/host/lib`, `/host/lib64` volume mounts were removed
from [`manifests/daemonset-klustre-csi-node.yaml`](../manifests/daemonset-klustre-csi-node.yaml).
Deploying the fixed image with the updated manifest resolves the issue; no workaround
is needed.
