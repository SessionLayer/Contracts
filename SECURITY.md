# Security policy

Report a vulnerability through GitHub's private vulnerability reporting: the
**Security** tab above, then **Report a vulnerability**. That opens a thread
only you and the maintainers can read. Do not open a public issue, pull
request, or discussion for a security finding.

[SessionLayer's vulnerability disclosure policy](https://github.com/SessionLayer/Documentation/blob/main/docs/security/vulnerability-disclosure.md)
is the single authority for every repository in this organization: what to
include in a report, full scope, embargo and credit, and how to verify that
the release you installed is the build the advisory named. Read it before
reporting.

## Scope in this repository

This repository holds the frozen contracts that cross a component boundary:
the Control Plane to Gateway gRPC contract, the Agent to Gateway and Gateway
to Gateway wire protocols, and the Control Plane's OpenAPI surface. It ships
no binary; consumers vendor a pinned tag via `contracts.lock`.

In scope: a contract that admits an unsafe state, and an N-1 negotiation
window that admits a downgrade or a silent version mismatch.

Not accepted here: a defect in how a component implements a contract. That
belongs to that component's repository, which is where the fix ships. The
policy lists the rest of the out-of-scope set, including test fixtures,
volumetric denial-of-service testing, anything starting from a credential the
threat model already assumes lost, and accepted risks already documented in
the trust model.

## Response targets

The [disclosure policy](https://github.com/SessionLayer/Documentation/blob/main/docs/security/vulnerability-disclosure.md)
carries the one timeline this organization keeps, from acknowledgement through
triage, fix and embargo, and it covers every repository including this one.
Advisories credit you unless you ask to stay anonymous, and request a CVE for
findings rated moderate or above. There is no bug bounty.
