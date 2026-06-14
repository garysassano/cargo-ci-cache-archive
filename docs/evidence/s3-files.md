# S3 Files Evidence

This page records the S3 Files experiments for Cargo target and registry state.

## Question

Could an S3-backed shared filesystem preserve Cargo state across ephemeral workers while keeping repeated no-op builds faster than local archive-cache approaches?

## Test Setup

- Cargo target directory on S3 Files.
- Cargo registry on S3 Files.
- Cargo target and registry both on S3 Files.
- S3 Files target with `Swatinem/rust-cache` handling Cargo registry and home.
- Prewarming mounted paths before Cargo.
- Copying registry state from S3 Files to local disk.
- Raising import thresholds to include larger Rust artifacts.

## Observations

Cargo could be logically clean:

```text
Compiling 0
Downloaded 0
Dirty 0
fingerprint error 0
```

The remaining elapsed time came from remote traversal and reads:

| Configuration | Observed result |
| --- | --- |
| Pure S3 Files registry and target | Forced no-op around 35 to 39 seconds |
| S3 Files prewarm | Moved work earlier; one prewarm was around 41 seconds before a lower Cargo time |
| Copy registry from S3 Files to local disk | One copy path took about 932 seconds |
| `rust-cache` registry and S3 Files target | Cargo improved to around 8 to 14 seconds in some runs but remained slower than local target caching |
| Target-directory priming after mount | Reduced the Cargo step by moving work into the priming step |

The tested runner also spent around 8 seconds in mount setup. The actual mount and `findmnt` work was around 1.5 seconds; installing or upgrading helper packages consumed roughly 5 to 6 seconds.

Large Rust artifacts also required a higher S3 Files import threshold. A 10 MiB threshold excluded files such as an 84 MiB SDK `rlib`, a 48.16 MiB events `rlib`, and a 14.13 MiB Lambda bootstrap. The experiment raised the relevant threshold to 1 GiB.

## Interpretation

S3 Files could preserve a shared namespace and enough state for Cargo's freshness proof to succeed, but Cargo still traversed many small metadata, fingerprint, dep-info, and build-script files through the network filesystem. Prewarming and priming shifted that cost rather than removing it.

## Limitations

- Results depend on the tested runner image, mount helper state, network path, and S3 Files service behavior at the time of the experiment.
- These findings address Cargo target and registry no-op state, not every possible shared-filesystem workload.

## Implications

Do not use S3 Files for Cargo target or registry state in the selected workflow. See the [S3 Files approach page](../approaches/s3-files.md) for current mechanics, architecture, and the archived workflow shape.
