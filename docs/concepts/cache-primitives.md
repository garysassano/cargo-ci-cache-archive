# Cache Primitives

Cargo CI caching experiments used three different storage primitives. They are not interchangeable, even when they all make a later build faster.

## Archive Cache

Examples:

- `actions/cache`
- An `actions/cache`-compatible backend
- `Swatinem/rust-cache`, which builds on top of archive cache semantics

Changing the backend used by `actions/cache` does not change archive-cache semantics. The restored state is still selected by a key, downloaded, and extracted into the current filesystem. It is not equivalent to a mounted filesystem snapshot. See the [RunsOn guide](../deployments/runs-on/README.md) for the selected backend implementation.

Archive cache behavior:

```text
compute key
download archive if key/restore-key matches
extract files into current filesystem
optionally clean/prune paths
upload new archive under a key
```

Best for:

- Download archives.
- Cargo registry/cache data.
- Dependency-oriented target artifacts.
- Setup action tarballs.
- Tool installer caches.
- `mise-action` tool installs under `MISE_DATA_DIR`.

Limitations:

- Extraction reconstructs files into a fresh filesystem.
- The cache key decides whether a new archive can be saved.
- Archive tools and cache actions may not preserve all metadata exactly as a local filesystem would.
- A cache action can clean or prune paths before save.

## Filesystem Snapshot

Examples:

- EBS snapshot restore.
- `runs-on/snapshot`-style mounted volume workflows.

Filesystem snapshot behavior:

```text
restore volume from snapshot
mount filesystem
build using paths inside mounted filesystem
unmount/detach
snapshot resulting filesystem
```

Best for:

- Reproducing local no-op Cargo behavior.
- Preserving exact target state, dep-info, fingerprints, build script outputs, and registry source trees.
- Workflows where the operational cost of volume lifecycle is acceptable.

Limitations:

- More infrastructure and lifecycle complexity.
- Credential-bearing files must be scrubbed before snapshot save.
- Toolchains and tool caches can bloat snapshots if placed under the snapshot root.

## Network Filesystem

Examples:

- [Amazon S3 Files](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files.html).
- Other remote filesystems exposed as local mount paths.

Network filesystem behavior:

```text
mount shared filesystem
read/write Cargo state directly on the mount
optional prewarm or local copy
unmount
```

Best for:

- Shared file access workloads.
- Cases where many clients need a common filesystem namespace.

Limitations for Cargo target state:

- Cargo no-op traverses many small metadata, fingerprint, dep-info, and build-script files.
- Remote filesystem metadata/read latency can dominate even when Cargo is logically fresh.
- Prewarming often moves cost rather than removing it.

## Choosing A Primitive

| Goal | Preferred primitive |
| --- | --- |
| Avoid dependency downloads | Archive cache / `rust-cache` |
| Preserve source mtimes | Cached worktree or filesystem snapshot |
| Preserve full local target state | Filesystem snapshot or source-keyed target archive |
| Cache setup tools and toolchains | Archive cache through `mise-action` |
| Share state across workers without archives | Network filesystem, but not ideal for Cargo target no-op |
| Keep workflow simple and maintained | `mise-action` with `Swatinem/rust-cache` and mtime-preserving checkout |
