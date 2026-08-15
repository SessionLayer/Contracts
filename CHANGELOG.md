# Changelog

All notable changes to this repo's contract bundle are documented here. The
version here is a **bundle tag** for this repo, independent of (but mapped
against) the three sub-contracts' own versions — see `README.md` "Tag scheme"
and `contracts/VERSIONING.md` for what each number means and how it moves.

## v0.1.1 — the contracts stop citing documents that do not ship

Every citation of the design and requirements documents comes out of the proto
comments, the OpenAPI descriptions, the wire specifications and the contract
READMEs: `FR-*` / `NFR-*` requirement ids, bare `§x.y` and `Design §x.y`
sections, `D<n>` decision ids, and the internal invariant and build-history
notes. Those documents ship in no SessionLayer repository, so a reader who
followed one arrived nowhere — while the citation's shape claimed there was
somewhere to arrive. Where the citation was the whole content of a comment, the
rule it pointed at is written down in its place.

Resolvable references are untouched: `RFC <n>` and its sections, the OpenID
Connect specifications, `CVE` / `GHSA` / `RUSTSEC` advisories, cross-references
between the documents in this bundle (`VERSIONING.md §2`,
`agent-gateway-v1.md §3`), and the `R1`-`R5` row labels the conformance page
defines for itself.

A PATCH bump: description and comment text only. No schema, path, enum member,
`required` list, field number, RPC or wire type changed, and `info.version`
stays `0.1.0`. A consumer that regenerates sees the new text in its generated
documentation comments; no generated type moves.

## v0.1.0 — an agent-connected node can be registered with its host anchor

`RegisterNodeRequest` gains `connectorKind` (`agentless` | `agent`, default
`agentless`), and `address` moves out of `required` because an `agent` node is
reached through the Agent's own outbound channel and must not carry a dial
address. The host anchor — at least one of `hostCertificate` / `pinnedHostKey` —
is now stated as a requirement of both kinds, which is what the Gateway has
always enforced: it runs the same no-TOFU host verification on the inner leg
however it reached the node, so an agent node with no anchor aborts every
session. Before this, the only way to give one an anchor was a direct database
write.

A MINOR bump: the new property is optional and defaults to the previous
behaviour, and relaxing `address` from required widens what a server accepts
without invalidating any request an existing client sends. The OpenAPI URI major
stays `v1` and `info.version` stays `0.1.0`; no protobuf or wire contract is
touched.

## v0.0.3 — `aws_kms` signs

`CaBackend` now documents `aws_kms` as a backend that signs rather than an
unimplemented seam, and `RotateCaRequest` states the `keyReference` grammar it
requires: a KMS key ARN, with alias ARNs refused because `kms:UpdateAlias`
repoints an alias invisibly to the Control Plane and would swap the signing key
underneath a CA whose public half is already distributed to every node.

Descriptions only. The `CaBackend` enum is unchanged — `aws_kms` has been a
member since v0.0.1, because the stored backend set is deliberately wider than
the usable one and is never narrowed.

## v0.0.1 — initial release

The first tagged bundle of the SessionLayer cross-repo contracts: the CP ↔
Gateway gRPC contract (`contracts/proto/`, `ProtocolVersion` 1.1), the Agent ↔
Gateway and Gateway ↔ Gateway wire protocols (`contracts/wire/`, both frozen
at 1.0), and the complete Control Plane REST surface (`contracts/openapi/`,
URI major `v1`, `info.version` 0.1.0).

**The gRPC contract** covers connect-time version negotiation; mTLS-based
Gateway and Agent identity (enroll/renew, generation-counter revocation);
data-plane authorization with a signed decision context; outer-leg credential
resolution, including the IdP-independent break-glass path (FIDO2 primary,
offline codes fallback); session recording (short-lived WORM upload
credentials, customer-held-key sealing so the platform cannot decrypt); the
fleet-wide lock deny-list feed; JIT and break-glass access models; host-cert
signing for the ProxyJump host-identity path; HA presence and Gateway↔Gateway
coordination; and session-lease/idle-timeout enforcement.

**The wire protocols** cover the outbound-agent dial-back model
(Agent↔Gateway, `wire/agent-gateway-v1.md`) and the peer-relay byte bridge
used for HA node routing (Gateway↔Gateway, `wire/gateway-relay-v1.md`), both
framed binary protocols over mutually-authenticated WebSocket with their own
version negotiation, independent of the gRPC plane. `wire/conformance/`
carries machine-checkable golden frames and negotiation vectors so each
consumer's own CI catches wire drift without needing a peer binary.

**The OpenAPI contract** covers the full config-resource surface (rules,
roles, role-bindings, CAs, service accounts, node policies, capability/JIT/
break-glass policies, session-limit policies), runtime resources (nodes,
sessions, session leases, gateways, join-tokens, gateway-enrollment-tokens),
the audit-event and recording read/replay/export/retention surface, and the
operator-settings singleton (including the customer recording key and CA
public-key/trust-anchor exports). Conventions are uniform across the surface:
cursor pagination, an `Idempotency-Key` header on mutating operations, RFC
9457 problem-document errors, and platform-RBAC gating with audit logging on
every write.

`contracts/VERSIONING.md` is the authoritative statement of how the three
contracts version independently and stay compatible across an N-1 window.
