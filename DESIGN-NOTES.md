# Design Notes: draft-fletcher-transaction-token-chaining-profile

This file records the design decisions, rationale, and open questions
established during the initial drafting of this specification.  It is
intended as a handoff document for contributors picking up the work,
and as an audit trail of why the draft is structured the way it is.

---

## Naming

The correct IETF individual draft name is:

```
draft-fletcher-transaction-token-chaining-profile
```

Note that the working group prefix (`oauth`) is **not** used for
individual submissions; that prefix is reserved for WG-adopted
documents.  The `docname` field in the YAML front matter of the
Markdown source must match this name exactly:

```yaml
docname: draft-fletcher-transaction-token-chaining-profile-00
```

The document was initially drafted with an incorrect `docname` of
`draft-fletcher-oauth-transaction-token-chaining-profile-00`.
This must be corrected before submission to the IETF Datatracker.

---

## What This Draft Is

This is a profile of:

> **draft-ietf-oauth-identity-chaining** ("OAuth Identity and
> Authorization Chaining Across Domains")

...that specifies how to use a **Transaction Token (Txn-Token)**
(`draft-ietf-oauth-transaction-tokens`) as the `subject_token` in a
Token Exchange (RFC 8693) request to obtain a JWT Authorization Grant
for crossing a trust domain boundary.

The profile is analogous in structure to:

> **draft-ietf-oauth-identity-assertion-authz-grant** (the "ID-JAG"
> profile, adopted by the OAuth WG in April 2026, currently at -03)

...which uses an OpenID Connect ID Token or SAML 2.0 assertion as
the subject token.  The two profiles are complementary siblings, not
competitors.  The ID-JAG profile is optimized for human-user SSO
scenarios where AS-B already trusts AS-A as an IdP.  This profile is
optimized for any transaction originating within a single trust domain
(enterprise) that needs to call an external partner service.

---

## Core Motivation and Deployment Context

### The Fundamental Scenario

A Txn-Token is **scoped to a single trust domain** (e.g., an
enterprise).  A workload within that trust domain may need to call a
partner service in a *different* trust domain to complete a
transaction.  The Txn-Token cannot be forwarded externally — it
carries internal context and is not meaningful outside its trust
domain.

This profile defines how the workload exchanges its Txn-Token at
**AS-A** (its own trust domain's authorization server) for a **JWT
Authorization Grant** that AS-A has curated specifically for
**AS-B** (the partner's authorization server).  Only the JWT
Authorization Grant crosses the trust boundary; the Txn-Token stays
inside Trust Domain A.

### The Canonical Example (use this when explaining the draft)

An enterprise mail service receives an inbound email via SMTP.  The
mail service holds a Txn-Token representing a `mail-delivery`
transaction.  Before storing the message, it must call a partner
**spam-rating API** in the spam service's trust domain.

- Trust Domain A: the enterprise (`as.enterprise.example`)
- Trust Domain B: the spam service (`as.spamsvc.example`)
- Protected Resource: `https://api.spamsvc.example/spam-rating`
- Scope: `spam.rating.read`

This example appears in the Token Exchange request example, the JWT
Authorization Grant example, and Use Case B in the appendix.  It was
chosen deliberately because it is a **system-initiated** transaction
(no OAuth client at the SMTP server), which is the most distinctive
of the three principal types and the hardest to address with ID-JAG.

### The Three Initiating Principal Types

A Txn-Token may represent any of three originating contexts, and
this profile explicitly handles all three:

| Type | `sub` example | Txn-Token characteristics |
|---|---|---|
| Human user | External OAuth access token → user identity | `rctx` carries client ID, IP |
| Internal system | SMTP server, messaging gateway | `purp` is primary context carrier |
| Automated workload | Scheduled job, pipeline | May use SPIFFE URI as `sub` |

The ID-JAG profile handles only the human user type.  This is a key
differentiator and must be preserved in the Introduction.

---

## Key Design Decisions

### 1. `audience` vs. `resource` in the Token Exchange Request

**Decision:** `audience` REQUIRED (identifies AS-B); `resource`
OPTIONAL (identifies the resource server / partner API).

**Rationale:** RFC 8693 allows both parameters; the base
identity-chaining draft used `resource` ambiguously to mean "the
target authorization server," which conflicts with RFC 8707 (where
`resource` means a resource server URI).  The ID-JAG profile (-03)
resolved this cleanly:

- `audience` → AS-B issuer URL → becomes `aud` in the JWT
  Authorization Grant
- `resource` → resource server URI (the partner API) → becomes
  `resource` claim in the grant; used by AS-B to scope the access token

This profile adopts the same convention.  This is normatively stated
and must not be weakened or made ambiguous in future edits.

**Key rule:** Implementations MUST use `audience` to identify AS-B
and MUST NOT pass the AS-B issuer URL as `resource`.

### 2. JWT `typ` Header Value

**Decision:** `txn-chain+jwt`

**Rationale:** Explicit JWT typing is required by RFC 8725 (JWT BCP).
Choosing a distinct `typ` allows an authorization server implementing
both this profile and the ID-JAG profile to discriminate between
grant types by inspecting the header, without examining the payload.
The ID-JAG profile uses `oauth-id-jag+jwt`; this profile uses
`txn-chain+jwt`.  Both require IANA registration.

### 3. New `txn_claims` JWT Claim

**Decision:** Introduce a new claim `txn_claims` (JSON object) in
the JWT Authorization Grant to carry a curated subset of Txn-Token
context across the domain boundary.

**Rationale:** The Txn-Token must not be forwarded externally.  But
AS-B may need some transaction context to apply authorization policy
(e.g., `purp` to distinguish a `mail-delivery` request from a
`spam-report` request; a redacted `rctx` to carry the SMTP envelope
sender for per-sender policy).  `txn_claims` provides a structured,
policy-controlled envelope for this.

**Requires IANA registration** of the `txn_claims` JWT claim name.

**Key constraint:** The Cross-Domain Trust Agreement MUST define
which claims may appear in `txn_claims` and their semantics.  AS-A
and AS-B must have a shared normative understanding.

### 4. `txn` Claim in the JWT Authorization Grant

**Decision:** The `txn` claim from the Txn-Token SHOULD be
transcribed into the JWT Authorization Grant as a SHOULD-level
requirement.

**Rationale:** This preserves end-to-end transaction correlation
across the trust boundary, which is critical for audit and
troubleshooting.  AS-B SHOULD record it in audit logs.  The privacy
trade-off (cross-domain correlation identifier) is acknowledged in
the Privacy Considerations section.

### 5. `jti` Single-Use Enforcement

**Decision:** Promoted from SHOULD to MUST for AS-B.

**Rationale:** JWT Authorization Grants are short-lived bearer tokens.
Replay is a real attack vector.  AS-B MUST track presented `jti`
values within the grant's validity window.  AS-A SHOULD set lifetimes
of ≤60 seconds (≤300 seconds as an outer bound).

### 6. `client_id` Not Required in the JWT Authorization Grant

**Decision:** No `client_id` claim in this profile's JWT
Authorization Grant.

**Rationale:** The ID-JAG profile requires `client_id` because the
client (the wiki app) has a pre-registered relationship with AS-B
(the chat app's AS).  In this profile, the Requesting Workload
typically does NOT have a pre-registered OAuth client relationship
with AS-B.  The workload's identity is conveyed through client
authentication to AS-A and through the `sub` mapping in the grant.
Adding `client_id` would require AS-B to maintain a workload client
registry, which conflicts with the bilateral domain trust model.

### 7. Trust Relationship Model: Bilateral, Not IdP-Mediated

**Decision:** The trust relationship between AS-A and AS-B is a
**Cross-Domain Trust Agreement** — a bilateral or federated
configuration established independently of any shared identity
provider.

**Rationale:** Unlike the ID-JAG profile (where the trust is
mediated through a shared IdP that both AS-A and AS-B trust for SSO),
this profile's use cases often involve trust domains with no shared
IdP — e.g., an enterprise and a third-party SaaS spam service.
The term "Cross-Domain Trust Agreement" is defined as a term of art
in the Conventions section.  It subsumes the key management, subject
identifier mapping policy, and permitted `txn_claims` definitions.

### 8. No Refresh Tokens

**Decision:** AS-B SHOULD NOT issue refresh tokens.

**Rationale:** Txn-Tokens are short-lived and transaction-specific.
Re-obtaining a Txn-Token and repeating the chaining flow is the
correct renewal mechanism.  A refresh token would create a persistent
credential whose lifetime is decoupled from the originating
transaction's authorization context — defeating the purpose of the
short-lived, transaction-scoped design.

### 9. Sender Constraining

**Decision:** AS-B SHOULD issue sender-constrained access tokens
(DPoP or mTLS).  The `cnf` claim in the JWT Authorization Grant
conveys the Requesting Workload's key to AS-B.

**Rationale:** Follows Appendix B.3 of the base identity-chaining
draft (the authorization-server-as-client topology).  AS-A must
verify proof of possession of the Requesting Workload's public key
before embedding it in the grant.

---

## Reference Specifications (with version notes)

| Reference | Version used | Notes |
|---|---|---|
| draft-ietf-oauth-identity-chaining | -11 | Normative base. Check for newer versions before submission. |
| draft-ietf-oauth-transaction-tokens | -08 | Normative. Check for newer versions. |
| draft-ietf-oauth-identity-assertion-authz-grant | -03 (April 2026) | Informative. WG-adopted; replaces individual draft-parecki-* versions. The old individual draft (`draft-parecki-oauth-identity-assertion-authz-grant`) is superseded and must NOT be cited. |
| draft-ietf-wimse-arch | -07 | Informative. Referenced for SPIFFE URI / workload identity context. |
| draft-ietf-wimse-workload-creds | -00 | Informative. Referenced for workload credential model. |
| RFC 8693 | — | Token Exchange. Normative. |
| RFC 7523 | — | JWT Profile for OAuth 2.0 Authorization Grants. Normative. |
| RFC 8707 | — | Resource Indicators. Normative. Critical for `resource` parameter semantics. |
| RFC 8725 | — | JWT BCP. Normative. Drives explicit `typ` requirement. |
| RFC 9700 | — | OAuth 2.0 Security Best Current Practice. Normative. |
| RFC 9728 | — | OAuth 2.0 Protected Resource Metadata. Normative. Used for AS-B discovery. |
| RFC 9396 | — | Rich Authorization Requests (RAR). Informative only; RAR integration is an open question. |

---

## Open Questions / Future Work

These items were explicitly deferred and should be tracked as GitHub
Issues:

### OQ-1: Mandatory Sender Constraining

Should DPoP or mTLS sender-constraining be elevated from SHOULD to
MUST?  The current draft says SHOULD to allow deployments that cannot
support key management.  However, bearer access tokens crossing trust
domain boundaries present a significant risk.  The WG may push for
MUST.

**Suggested resolution:** Consider making sender-constraining MUST for
the access token issued by AS-B, while leaving the JWT Authorization
Grant as bearer (it is short-lived and single-use).

### OQ-2: RAR / `authorization_details` Integration

The ID-JAG profile supports `authorization_details` (RFC 9396) in
the grant.  This profile does not yet define how RAR
`authorization_details` from a Txn-Token would be transcribed into
the JWT Authorization Grant.

**Suggested resolution:** Define a `txn_claims.authorization_details`
sub-claim, or allow top-level `authorization_details` in the JWT
Authorization Grant following the same minimization rules as scope.
Needs discussion with the OAuth WG.

### OQ-3: SPIFFE URI Subject Mapping

The draft states that AS-A "SHOULD apply the SPIFFE URI translation
rules defined in the Cross-Domain Trust Agreement" but does not define
any normative SPIFFE URI mapping algorithm.

**Suggested resolution:** Either (a) define a RECOMMENDED mapping
algorithm (e.g., strip the trust domain prefix and use the path
component), or (b) explicitly defer to a companion WIMSE document.
Coordinate with the WIMSE WG.

### OQ-4: Human User Sub Mapping — Pairwise vs. Opaque

For human-user-initiated transactions, the draft allows `email`,
"a stable pairwise identifier," or "another identifier agreed in the
Cross-Domain Trust Agreement."  This is underspecified.

**Suggested resolution:** Consider defining a RECOMMENDED default
(e.g., email, given its widespread cross-domain recognizability) and
a RECOMMENDED stable pairwise identifier construction (e.g.,
PPID-style hash of `iss + sub + audience`).

### OQ-5: `txn_claims` Schema Registration

The draft registers the `txn_claims` claim name with IANA but does
not register a schema or enumerate permitted sub-claims beyond
informal guidance in the Claims Transcription section.

**Suggested resolution:** Either (a) define a small set of RECOMMENDED
`txn_claims` sub-claims (`purp`, `rctx`) as normative, with IANA
registration for each, or (b) explicitly state that the sub-claim
set is deployment-defined via the Cross-Domain Trust Agreement.

### OQ-6: Txn-Token `aud` Claim — TTS vs. AS-A

The draft states that the Txn-Token's `aud` claim "MUST identify
AS-A (or a value that AS-A accepts as a valid audience for presented
subject tokens)."  In deployments where the TTS and AS-A are separate
services, it is not obvious whether the Txn-Token's audience should
be the TTS or AS-A.

**Suggested resolution:** Align with the Txn-Token spec
(draft-ietf-oauth-transaction-tokens) on what the canonical `aud`
value for a Txn-Token is.  If the Txn-Token spec says `aud` is the
TTS's own URI, then AS-A must be configured to accept that value.

### OQ-7: Discovery of AS-B — AS-B Issuer vs. Protected Resource Metadata

The draft references RFC 9728 (Protected Resource Metadata) for
AS-B discovery, specifically the `authorization_servers` property.
However, RFC 9728 is relatively new and may not be widely deployed.

**Suggested resolution:** Add a fallback discovery mechanism, e.g.,
AS-A can be statically configured with the mapping from resource
server hostname to AS-B issuer URL as part of the Cross-Domain Trust
Agreement.

### OQ-8: Docname Correction (Action Required Before Submission)

The `docname` field in the YAML front matter currently reads:

```
draft-fletcher-oauth-transaction-token-chaining-profile-00
```

This must be corrected to:

```
draft-fletcher-transaction-token-chaining-profile-00
```

Individual IETF drafts do not carry the working group name in the
individual author prefix.

---

## Relationship to ID-JAG Profile — Key Comparison Points

When explaining to the OAuth WG how this profile differs from the
ID-JAG profile, lead with these points in order:

1. **Subject token type**: Txn-Token vs. ID Token / SAML assertion.
2. **Principal scope**: all three initiating types vs. human-user only.
3. **Trust basis**: bilateral Cross-Domain Trust Agreement vs.
   shared IdP SSO trust.
4. **`client_id`**: not required here vs. REQUIRED in ID-JAG.
5. **`typ`**: `txn-chain+jwt` vs. `oauth-id-jag+jwt`.
6. Both profiles use the same `audience`/`resource` parameter
   convention (this was aligned with ID-JAG -03 during drafting).

The two profiles are explicitly complementary and a single AS MAY
implement both.

---

## Drafting History

| Date | Event |
|---|---|
| 2026-05 | Initial draft produced via iterative refinement in Claude.ai chat session |
| 2026-05 | Reference to `draft-parecki-oauth-identity-assertion-authz-grant` corrected to WG-adopted `draft-ietf-oauth-identity-assertion-authz-grant-03` |
| 2026-05 | `audience`/`resource` parameter semantics fully specified and aligned with ID-JAG -03 convention; Token Exchange response section added; example request/response added |
| 2026-05 | Complete rewrite to incorporate three Initiating Principal types (human user, internal system, automated workload); canonical mail-service/spam-rating example introduced; all three use cases updated; `Cross-Domain Trust Agreement` defined as term of art; subject identifier mapping section added; docname correction noted |
