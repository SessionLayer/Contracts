# SessionLayer Contracts

The single, independently-versioned home for every contract that crosses a
component boundary in [SessionLayer](https://github.com/SessionLayer): the
CP↔Gateway gRPC contract, the Agent↔Gateway and Gateway↔Gateway wire
protocols, and the Control Plane's OpenAPI REST surface.

Rather than living inside one consumer's tree and being vendored by the others
via a sibling-checkout-path copy script — a mechanism that is a silent no-op in
CI, since CI checks out one repo at a time and the sibling path never exists
there — every consumer, including `ControlPlane` itself, vendors from this repo
by cloning a pinned git tag, and every consumer's CI genuinely re-fetches that
tag to verify the vendored copy hasn't drifted.

## Layout

The moved contract payload lives, unchanged, under [`contracts/`](contracts/):

```
contracts/
├── openapi/       # REST contract (OpenAPI 3.1)
├── proto/         # CP <-> Gateway gRPC contract (protobuf, buf-managed)
├── wire/          # Agent<->Gateway and Gateway<->Gateway wire-protocol specs
├── redocly.yaml    # OpenAPI linter config
├── lint.sh         # buf lint + buf breaking + redocly lint, in one entrypoint
└── VERSIONING.md   # the authoritative N-1 compatibility / versioning policy
```

See [`contracts/README.md`](contracts/README.md) for the full layout and
per-contract detail, and [`contracts/VERSIONING.md`](contracts/VERSIONING.md)
for the versioning policy (D33/§16A, FR-HA-9) and what each current protocol
version covers.

## Tag scheme

Each tag (`vX.Y.Z`) is a bundle version for this repo, independent of but
documented against the three sub-contracts it carries at that point (see
[`CHANGELOG.md`](CHANGELOG.md) for the exact mapping). Bundle bumps:

| Bump | When |
|---|---|
| `MAJOR` | any sub-contract had a breaking change, or the OpenAPI URI major moved |
| `MINOR` | any additive contract change shipped (new field/RPC/endpoint) |
| `PATCH` | contracts-repo-only changes (lint config, docs, README) with zero effect on any consumer's generated code |

The bundle tag is a convenience pin for consumers. It is not the same number
as the gRPC `ProtocolVersion`, the wire `ProtocolVersion`, or the OpenAPI
`info.version`, each of which is versioned independently per
`contracts/VERSIONING.md`.

## How a consumer vendors from this repo

Every consumer (`ControlPlane`, `Gateway`, `Agent`, `Dashboard`)
pins an exact tag+commit in its own `contracts.lock` and fetches with its own
`scripts/vendor-contracts.sh`:

1. `contracts.lock` names `repo=SessionLayer/Contracts`, `tag=<pinned tag>`,
   `sha=<resolved commit>`. The SHA is the ground truth: the vendor/check
   scripts resolve the tag on every run and assert it still equals the locked
   SHA, so a moved/re-pushed tag can't silently swap content.
2. `scripts/vendor-contracts.sh` does `git clone --depth 1 --branch <tag>` of
   this repo into a temp dir, verifies the resolved commit SHA matches
   `contracts.lock`, then copies the relevant subtree(s) of `contracts/` into
   that consumer's own committed vendored path, preserving each consumer's
   own on-disk layout.
3. `scripts/vendor-contracts.sh --check` does the same fetch and diff without
   writing anything, and exits non-zero on any mismatch. Every consumer's CI
   runs this as a real, network-fetching drift check.

The fetch mechanism is git-only (`git clone --depth 1 --branch <tag>`) and
needs no GitHub API token or hosted registry. It works fully offline once the
tag has been fetched.

## CI

One required check, `gate`: `buf lint` + `buf breaking` (against the previous
tag) + `redocly lint` over `contracts/`, via `contracts/lint.sh`. SHA-pinned
actions, `permissions: contents: read`. Dependabot covers GitHub Actions only —
this repo has no runtime dependencies.
