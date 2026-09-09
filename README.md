# Probe: checksum-hash-pinning

## Probe metadata

| Field                  | Value                                       |
|------------------------|---------------------------------------------|
| PM                     | uv                                          |
| Pattern                | checksum-hash-pinning                       |
| Feature category       | checksum_signing                            |
| PM version under test  | 0.12.12                                     |
| Generated at           | 2026-09-09T17:02:09Z                        |
| Schema version         | 1.2                                         |

## Purpose

This probe exercises SHA-256 hash pinning in `uv.lock` — specifically
the `hash = "sha256:..."` fields inside `[package.sdist]` and
`[[package.wheels]]` blocks.

UV 0.12.12 introduced Authenticode code-signing and macOS notarization
of the uv CLI binary itself. The lockfile hash format is unchanged, but
this release makes it important to verify that Mend SCA can still
parse `uv.lock` hash entries correctly and report them in the
dependency tree, regardless of how the uv binary is signed or
notarized.

The probe uses `requests==2.31.0` and its transitive dependencies
(`certifi`, `charset-normalizer`, `idna`, `urllib3`), all sourced from
PyPI with known real SHA-256 hashes.

## Relevance to Mend SCA

Mend's Python resolver (pip path) operates on `uv.lock` via the
UV Project Filtering path (`MEND_SCA_UV_PROJECTS`). The lockfile's
per-artifact hash entries are the ground truth for reproducible
installs. If Mend cannot parse the `sha256:` prefixed values from
the `[package.sdist]` and `[[package.wheels]]` tables, hash
information is silently dropped from the detected tree.

This probe exists to confirm:
1. The `hashes` field in `expected-tree.json` is populated correctly.
2. Mend does not re-resolve or override the lockfile's pinned
   dependencies due to changes in how the uv binary is signed.
3. Both sdist and wheel hash entries are parsed.

## Dependency graph

```
checksum-hash-pinning (root)
└── requests 2.31.0
    ├── certifi 2024.2.2
    ├── charset-normalizer 3.3.2
    ├── idna 3.6
    └── urllib3 2.2.1
```

## Python version detection

The probe ships `.python-version` containing `3.11`. Mend reads
`.python-version` at higher precedence than `pyproject.toml`'s
`requires-python` on the PIP detection chain. Both files declare
Python 3.11 compatibility. Mend will use `.python-version`.

## Mend config

Bucket B — no `.whitesource` emitted. Python version is dynamically
detected from `.python-version` (takes precedence on the PIP chain)
and `requires-python = ">=3.11"` in `pyproject.toml`. The uv tool
itself is not in the `install-tool` list and cannot be pinned via
`scanSettings.versioning`; the probe targets uv 0.12.12 behavior but
cannot enforce that version through Mend's tool-provisioning layer.
Document this limitation: operators running this probe must ensure
uv 0.12.12 is installed in the scan environment out-of-band.

## Files

```
checksum-hash-pinning-20260909-170209/
├── .python-version          Python 3.11 pin for Mend version detection
├── pyproject.toml           PEP 621 manifest — requests as direct dep
├── uv.lock                  Lockfile with SHA-256 hashes per artifact
├── src/
│   └── checksum_hash_pinning/
│       └── __init__.py      Minimal source stub
├── README.md                This file
└── expected-tree.json       Ground truth for downstream comparison
```
