# Changelog

All notable changes to this repo's contract bundle are documented here. The
version here is a **bundle tag** for this repo, independent of (but mapped
against) the three sub-contracts' own versions - see `README.md` "Tag scheme"
and `contracts/VERSIONING.md` for what each number means and how it moves.

## v0.3.1 - the comments say only what the schema cannot

A `PATCH` by this repo's own rule: contract-repo-only changes with no effect on any
consumer's API shape. No field, RPC, endpoint, enum value or constraint moved.

Every comment in the bundle was adjudicated individually - 449 comment blocks, each
with a recorded verdict - and survived only by stating something the generated code
does not, or by naming a catch a reasonable edit would break. 1386 comment lines
became 873. A `.proto` is the specification, so the normative rules stayed: enforced
ranges, MUST-rejects, single-use constraints, which side allocates a value, what an
empty value means. Field comments restating a field's name, type or cardinality went.

One correction, and it is why consumers should take this pin. `LockEvent.added` said
a pushed lock means "the Gateway adds it and immediately tears down any matching live
session" - unconditionally - while `LockMode` in the same file defines
`LOCK_MODE_BEST_EFFORT` as "blocks new issuance but does NOT forcibly tear down an
already-established session". An implementer following the `added` comment would tear
down exactly the sessions a BEST_EFFORT lock exists to leave running.

## v0.3.0 - the pin list answers the question an operator actually asks

Three defects found by standing a Control Plane up and driving the API, rather than
by reading the spec.

`GET /v1/pins` could not enumerate. `identity` was a required query parameter, so an
operator holding `user:manage` could ask "does this identity have pins" but never
"which pins exist" or "who still holds one" - the two questions offboarding and
incident review start from, and neither answerable without already knowing the
answer. `identity` is now optional: omitted lists every live pin, present filters to
one identity. Relaxing a required parameter invalidates no request an existing
client sends.

`ttlSeconds` was required on every rule, including a `deny`, where a grant lifetime
means nothing - a deny grants nothing and stays in force until the rule is changed.
It comes out of both `required` lists, and its description now states the real rule:
required for `effect: allow`, where it bounds the grant and its absence is a `422`
naming the field; ignored for `effect: deny`. No default is invented, because an
unbounded grant must never be inferred from silence.

The closed permission vocabulary gains `metrics:read`. The Control Plane's metrics
endpoint is authenticated but not authorized: a service account with no role
bindings at all reads the full meter set - fleet-wide live-session counts,
authorization error rates, CA-signer activity, session-limit denials - so every
machine identity the platform has ever issued can read it. Reusing `audit:read`
would be the worse trade, handing a scraper the entire audit trail to reach a gauge,
so the vocabulary widens by one member instead. It sits beside `audit:read`, the
other read-only view of the platform's own operation.

**A widened closed vocabulary is a change every copy of it must follow**: the
Control Plane's `PlatformPermissions.ALL`, the `platform_role.permissions` CHECK
constraint, the seeded admin role, and the Dashboard's role editor. A copy that
misses it rejects `metrics:read` as outside the vocabulary, and the permission is
ungrantable.

A MINOR bump, not a patch. Every change here is additive or relaxing and breaks no
existing client, but none of them is description-only: a new enum member moves the
generated Java enum and the TypeScript union, and both relaxations move generated
model and client signatures. The PATCH row requires zero effect on a consumer's
generated code, so it does not apply. The gRPC protocol stays `1.1`, the wire
protocol `1.0`, the OpenAPI URI major `v1`, and `info.version` `0.1.0`.

## v0.2.2 - six operations name the permission they enforce

`GET`/`POST /v1/pins`, `DELETE /v1/pins/{pinId}`, `POST`/`DELETE
/v1/service-accounts/{id}/credentials` and `GET /v1/jit-requests/{jitRequestId}`
are all platform-RBAC gated, and not one of them said gated on what. Four carried
no description at all; two said "Platform-RBAC gated + audited" and stopped there,
which is worse than silence - it tells a client a gate exists and withholds the one
fact needed to pass it. The five pin and credential operations name `user:manage`,
the JIT read names `request:approve`, and each was read out of the controller that
enforces it rather than inferred from its neighbours.

The four missing descriptions are written. `POST /v1/jit-requests` already stated
outright that it is open to any authenticated principal and is unchanged, but with
the JIT read naming its permission the pair no longer reads as though the whole
resource were ungated.

Every platform-RBAC-gated operation on the surface now names its permission. The
three that name none do so by design: `POST /v1/oauth2/token` and
`POST /v1/auth/device` carry their own security schemes, and the JIT submission is
deliberately open.

A PATCH bump: descriptions only. No path, schema, property, enum member or
`required` list moved, and `info.version` stays `0.1.0`.

## v0.2.1 - a validator refuses the empty anchor set, not only a reader

`NodeHostAnchorsRequest` stated in prose that at least one of `hostCertificate` /
`pinnedHostKey` is required, and left the rule for a human to apply. Over exactly
two declared properties with `additionalProperties: false`, `minProperties: 1`
says the same thing where a spec-driven client validator will actually check it -
before the request reaches a server that would only refuse it. The prose stays,
because the prose is what says why an empty set can never be accepted.

A PATCH bump: no property, path, enum member or `required` list moved, and the
keyword reaches neither generator - the Java models carry no constraint for it and
`openapi-typescript` emits types only - so no consumer's generated code changes.
`info.version` stays `0.1.0`.

## v0.2.0 - the refusals become visible and the anchorless node becomes repairable

`AuthorizeRequest` gains `credential_principals`, the logins the presented
outer-leg credential is scoped to. The Gateway already reduces on that scope, but
locally and before it asks - so a scoped credential used for a login outside its
scope was refused with no decision record written anywhere, and the refusal an
auditor most wants to see was the one nobody could. The field is a deny-only
reducer like `source_ip`: it can suppress an allow, never widen one, and empty
means unscoped. The Gateway keeps its local reduction as a backstop, so a caller
that omits the field still cannot obtain an out-of-scope allow - it forfeits only
the audit record, which is what it had before.

`GET` and `PUT /v1/nodes/{nodeId}/host-anchors` are the repair path for a node
with no host-identity anchor. An Agent that joins under a name nobody registered
has its node created for it with neither a host certificate nor a pinned host key,
and the Gateway never trusts a host on first use, so every session to that node
aborts. No call could fix it: the only escape was to abandon the name. `PUT`
replaces the anchor set atomically and refuses an empty one, because a node
without an anchor does not fall back to trust-on-first-use - it stops working.
Gated on `node:enroll`, the permission that writes the same anchors at
registration.

`NodeResource.health` and `owningGateway` now describe a derivation instead of a
stored value. They are computed per request from `runtime.presence` and the node's
anchors: an anchorless node is `unhealthy` before anything else is considered, an
agent node is `healthy` / `unreachable` / `unknown` by how fresh its owner's
heartbeat is, and an agentless node is `unknown` permanently - the Control Plane
holds no liveness signal for a node it dials on demand, and that `unknown` is not
a fault the reader should go looking for.

The repo's own gate now regenerates `frames.json` and fails when the committed
golden differs from what `framegen` produces. Both Rust consumers check their wire
codecs against that file, so a proto change without a regeneration left two green
suites measuring bytes that had stopped describing the contract.

A MINOR bump: one protobuf field with a fresh number, one new path, and
description text. The gRPC protocol stays `1.1` - a field added within an
already-bumped minor does not move the number again - the wire protocol stays
`1.0`, and the OpenAPI URI major stays `v1` with `info.version` `0.1.0`.

## v0.1.3 - three constraints stated in the present tense

The last of the build provenance in this half of the bundle: the wire spec's
`0x21` / `0x30` reservation notes, the versioning policy's account of how a host
anchor used to be reachable, and the enrollment-token endpoint's reference to the
raw `INSERT` the install guide once required. Each recorded how something used to
be done, which a reader cannot act on.

Where the note left a live constraint behind, the constraint is kept and stated in
the present tense; where it did not, the note goes. Prose only.

This release exists because the commit carrying it was written two minutes after
`v0.1.1` was pushed and was therefore not in the merge - a stranding this repo has
hit before, since it merges earliest and its branch keeps moving underneath.

## v0.1.2 - the relay client says why it is hand-rolled

`gateway-relay-v1.md` told the reader to "see the supply-chain rationale" for why
the reference NATS client is hand-rolled. That rationale is a page in a different
repository, named nowhere and linked nowhere, so the pointer resolved for nobody.
The document now states the reason: hand-rolling keeps an entire TLS stack out of
the dependency graph.

A PATCH bump, and the same defect as v0.1.1 in a form no id-shaped pattern could
find - the pointer had no identifier in it at all.

## v0.1.1 - the contracts stop citing documents that do not ship

Every citation of the design and requirements documents comes out of the proto
comments, the OpenAPI descriptions, the wire specifications and the contract
READMEs: `FR-*` / `NFR-*` requirement ids, bare `§x.y` and `Design §x.y`
sections, `D<n>` decision ids, and the internal invariant and build-history
notes. Those documents ship in no SessionLayer repository, so a reader who
followed one arrived nowhere - while the citation's shape claimed there was
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

## v0.1.0 - an agent-connected node can be registered with its host anchor

`RegisterNodeRequest` gains `connectorKind` (`agentless` | `agent`, default
`agentless`), and `address` moves out of `required` because an `agent` node is
reached through the Agent's own outbound channel and must not carry a dial
address. The host anchor - at least one of `hostCertificate` / `pinnedHostKey` -
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

## v0.0.3 - `aws_kms` signs

`CaBackend` now documents `aws_kms` as a backend that signs rather than an
unimplemented seam, and `RotateCaRequest` states the `keyReference` grammar it
requires: a KMS key ARN, with alias ARNs refused because `kms:UpdateAlias`
repoints an alias invisibly to the Control Plane and would swap the signing key
underneath a CA whose public half is already distributed to every node.

Descriptions only. The `CaBackend` enum is unchanged - `aws_kms` has been a
member since v0.0.1, because the stored backend set is deliberately wider than
the usable one and is never narrowed.

## v0.0.1 - initial release

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
