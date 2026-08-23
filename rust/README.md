# Shared Rust sources

Canonical copies of the TLS/mTLS code the Agent and the Gateway both compile.
They are byte-identical in both consumers, and were duplicated before this
directory existed: a fix to the pinned-certificate verifier in one repository
did not reach the other, and nothing detected that.

Vendored by each consumer's `scripts/vendor-contracts.sh` into its own `src/`,
pinned by `contracts.lock` and diffed on every CI run - the same mechanism the
`.proto` files use. They are **source files, not a crate**: nothing enters
`[dependencies]`, the SBOM, or `cargo deny`'s graph.

| File | What it is |
| --- | --- |
| `mtls.rs` | `Tls13OnlyPinnedVerifier` - TLS 1.3 only, verifies the peer against pinned trust anchors and refuses an empty anchor set (fail closed). Plus the mTLS client/connect path. |
| `tls.rs` | One-time process init of the `ring` crypto provider. |
| `secret.rs` | `serde_zeroizing_string` - serde adapter for secrets that must be zeroized on drop. |

This directory sits beside `contracts/`, not inside it: the Control Plane
vendors `contracts/` wholesale with `rsync --delete` and would otherwise
inherit Rust it never compiles.
