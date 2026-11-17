# Design Notes: draft-fletcher-oauth-txn-token-chaining-profile

This file records the design decisions, rationale, and open questions
established during the initial drafting of this specification.  It is
intended as a handoff document for contributors picking up the work,
and as an audit trail of why the draft is structured the way it is.

---

## Naming

The correct draft name is:

```
draft-fletcher-oauth-txn-token-chaining-profile
```

The `docname` field in the YAML front matter of the Markdown source,
and the Markdown filename itself, must match this name exactly:

```yaml
docname: draft-fletcher-oauth-txn-token-chaining-profile-latest
```

**The `oauth` element is deliberate and is an exception to the usual
individual-draft convention.**  Individual submissions normally carry
no working group prefix — that prefix is ordinarily reserved for
WG-adopted documents — but including `oauth` here is what causes the
draft to be assigned to the OAuth working group.  Do not remove
`oauth` from the name, and do not flag it as a naming error.

**`txn-token` is abbreviated, not spelled out.**  The IETF Datatracker
limits a draft name to 50 characters;
`draft-fletcher-oauth-transaction-token-chaining-profile` is 55 and is
rejected at submission.  The abbreviated form is 47.  Do not expand
`txn-token` back to `transaction-token`.

See "Document Naming (RESOLVED 2026-08)" below for the history of this
decision, which reversed the project's original position.

### Version Numbering Restarts at the Rename

**A new document name is a new Datatracker document, so version
numbering restarted at -00.**  Versions -00, -01, and -02 were
submitted under the original name
`draft-fletcher-transaction-token-chaining-profile`.  Under the
current name, -00 was published on 2026-09-10 and -01 on 2026-10-26;
the next submission is **-02**.  The new -00 is linked back to its
predecessor by a Replaces relationship rather than by continuing the
version sequence.  The intermediate name
`draft-fletcher-oauth-transaction-token-chaining-profile` was never
published and must not appear in any release tag.

Two consequences to keep in mind:

- **Release tags.**  A release tag is the document name plus the
  two-digit version (e.g.
  `draft-fletcher-oauth-txn-token-chaining-profile-01`).  The tags for
  -00 through -02 under the old name remain; that is correct and they
  should not be renamed or recreated.  Two distinct tags end in `-00`,
  and two end in `-01`.

- **The Replaces relationship was set automatically.**  `replaces()`
  in `lib/upload.mk` fires only when the tag ends in `-00`.  It walks
  the `git log --follow` rename chain and picks the first previous
  name that has a release tag in the repo.  That chain is
  `...-oauth-txn-token-...` → `...-oauth-transaction-token-...` →
  `...-transaction-token-...`; the middle name carries no tag, so it is
  skipped and the relationship resolves to
  `draft-fletcher-transaction-token-chaining-profile`.  Do not create a
  tag under the intermediate name.  The value is editable on the
  Datatracker document page if it ever needs correcting.

### Document History Conventions

In the draft's Document History section, a `## Since Draft NN` heading
lists the changes made **after** version NN was published — that is,
the changes that appear in the version being prepared.

The headings published in -01 and -02 were off by one: the section
labelled `Since Draft 01` in both actually described the changes that
shipped *in* -01, and -02's own changes were never recorded at all.
Both were corrected while preparing the rename, so the published -01
and -02 on the Datatracker do not match the current file.  This is
expected; do not "restore" the old headings to match them.

**Version numbers in these headings are ambiguous across the rename
and must be qualified by document name.**  Because numbering restarts
at -00 (see above), the existing `Since Draft 00` / `01` / `02`
headings refer to versions of
`draft-fletcher-transaction-token-chaining-profile`, while future
headings will reuse those same numbers for versions of
`draft-fletcher-oauth-txn-token-chaining-profile`.  Any heading naming
a bare version number is therefore unsafe on its own.

**Pending:** the Document History section has not yet been restructured
for this.  Its `Since Draft 00` / `01` / `02` headings still carry bare
numbers referring to the pre-rename document, and the new-name -01 was
published with its own changes listed under `Since Draft 02`.  The
section is to be hand-edited by the author before the next submission,
to identify which document each entry belongs to and to qualify
headings by name so the two sequences do not collide.

---

## What This Draft Is

This is a profile of:

> **draft-ietf-oauth-identity-chaining** ("OAuth Identity and
> Authorization Chaining Across Domains")

...that specifies how to use a **Transaction Token (Txn-Token)**
(`draft-ietf-oauth-transaction-tokens`) as the `subject_token` in a
Token Exchange (RFC 8693) request to obtain a **Transaction Token JWT
Authorization Grant (Txn-Token-JAG)** for crossing a trust domain
boundary.

### Terminology: Txn-Token-JAG

The grant this profile defines is called the **Txn-Token-JAG**
(introduced in PR #21, September 2026, together with the title change
to "Transaction Token JWT Authorization Grant Profile ...").  The name
parallels the ID-JAG of the sibling profile and gives the artifact a
short, unambiguous label distinct from the generic "JWT Authorization
Grant" of the base identity-chaining draft.  Use "Txn-Token-JAG" when
referring to this profile's grant; use "JWT Authorization Grant" only
for the base-spec concept.

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
**AS-A** (its own trust domain's authorization server) for a
**Txn-Token-JAG** that AS-A has curated specifically for **AS-B** (the
partner's authorization server).  Only the Txn-Token-JAG crosses the
trust boundary; the Txn-Token stays inside Trust Domain A.

### The Canonical Example (use this when explaining the draft)

An enterprise mail service receives an inbound email via SMTP.  The
mail service holds a Txn-Token representing a `mail-delivery`
transaction.  Before storing the message, it must call a partner
**spam-rating API** in the spam service's trust domain.

- Trust Domain A: the enterprise (`as.enterprise.example`)
- Trust Domain B: the spam service (`as.spamsvc.example`)
- Protected Resource: `https://api.spamsvc.example/spam-rating`
- Scope: `spam.rating.read`

This example appears in the Token Exchange request example, the
Txn-Token-JAG example, and the System-Initiated use case in the
appendix.  It was chosen deliberately because it is a
**system-initiated** transaction (no OAuth client at the SMTP server),
which is the most distinctive of the common principal types and the
hardest to address with ID-JAG.

### Initiating Principal Types (Common Examples, Not a Closed Set)

A Txn-Token may represent many originating contexts.  The draft uses
three common examples throughout; they are illustrative, and the
Txn-Token spec may define others.  The profile's rules (subject
mapping, claims transcription) apply regardless of type.

| Type | `sub` example | Txn-Token characteristics |
|---|---|---|
| Human user | External OAuth access token → user identity | `rctx` carries client ID, IP |
| Internal system | SMTP server, messaging gateway | `scope` is primary context carrier |
| Automated workload | Scheduled job, pipeline | May use SPIFFE URI as `sub` |

The ID-JAG profile handles only the human user type.  This is a key
differentiator and must be preserved in the Introduction.

Note: earlier drafts used `purp` for the transaction purpose.  The
Txn-Token spec carries this in `scope`, and the draft was aligned in
the -01 (old name) cycle.  Use `scope`, not `purp`.

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
the Txn-Token-JAG to carry a curated subset of Txn-Token context
across the domain boundary.

**Rationale:** The Txn-Token must not be forwarded externally.  But
AS-B may need some transaction context to apply authorization policy
(e.g., `scope` to distinguish a `mail-delivery` request from a
`spam-report` request; a redacted `rctx` to carry the SMTP envelope
sender for per-sender policy; selected `tctx` fields such as a ticker
symbol the partner needs to fulfil the request).  `txn_claims`
provides a structured, policy-controlled envelope for this.  See
Decision #11 for the `tctx` minimization rules.

**Requires IANA registration** of the `txn_claims` JWT claim name.

**Key constraint:** The Cross-Domain Trust Agreement MUST define
which claims may appear in `txn_claims` and their semantics.  AS-A
and AS-B must have a shared normative understanding.

### 4. `txn` Claim in the Txn-Token-JAG

**Decision:** The `txn` claim from the Txn-Token SHOULD be
transcribed into the Txn-Token-JAG.  (The draft currently states this
inconsistently — see OQ-9.)

**Rationale:** This preserves end-to-end transaction correlation
across the trust boundary, which is critical for audit and
troubleshooting.  AS-B SHOULD record it in audit logs.  The privacy
trade-off (cross-domain correlation identifier) is acknowledged in
the Privacy Considerations section.

### 5. `jti` Single-Use Enforcement

**Decision:** Promoted from SHOULD to MUST for AS-B.

**Rationale:** Txn-Token-JAGs are short-lived bearer tokens.
Replay is a real attack vector.  AS-B MUST track presented `jti`
values within the grant's validity window.  AS-A SHOULD set lifetimes
of ≤60 seconds (≤300 seconds as an outer bound).

### 6. `client_id` Not Required in the Txn-Token-JAG

**Decision:** No `client_id` claim in this profile's Txn-Token-JAG.

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
(DPoP or mTLS).  The `cnf` claim in the Txn-Token-JAG conveys the
Requesting Workload's key to AS-B.

**Rationale:** Follows Appendix B.3 of the base identity-chaining
draft (the authorization-server-as-client topology).  AS-A must
verify proof of possession of the Requesting Workload's public key
before embedding it in the grant.

### 10. Symmetric Domain Isolation — No Grant-to-Foreign-TTS / No Downstream Txn-Token Re-Minting

**Decision:** This profile deliberately stops at obtaining an access token so
the Requesting Workload can call a partner Protected Resource.  It does NOT
present the Txn-Token-JAG to Trust Domain B's Transaction Token
Service, and it does NOT define a mechanism for AS-A's grant to cause a native
Txn-Token to be minted in Trust Domain B.

**Rationale:** A Txn-Token is authoritative only within its issuing domain and
carries that domain's internal context.  Decision #3 / the Claims Minimization
section already forbid a Domain A Txn-Token from leaving Domain A; the same
invariant must hold symmetrically for Domain B — **a Domain B Txn-Token MUST
NOT leave Domain B.**  Allowing AS-A's grant to be presented to Domain B's TTS
(the "Mode B / direct exchange" pattern in
`draft-liu-oauth-cross-domain-txn-token`) risks a Domain A entity obtaining a
native Domain B Txn-Token and thereby leaking Domain B's internal
`req_wl`/`tctx`/`rctx` and topology across the boundary — especially when the
grant is bearer (no `cnf`) and not single-use.  Keeping the downstream Txn-Token
minting internal to Domain B (the access-token-mediated path) preserves
isolation on both sides: no Domain-A-held credential ever reaches Domain B's
TTS.  The full comparison and WG review text are kept as a local working note
in `reviews/draft-liu-review-response.md` (git-ignored, not distributed).

Domain B *may* still mint its own Txn-Token for the inbound call — but from the
access token, at its own perimeter, by its own policy (see Decision #12).

### 11. `tctx` May Cross Only in Minimized Form

**Decision:** `tctx` MAY be transcribed into `txn_claims`, but only the
individual fields Trust Domain B needs to authorize or perform the
transaction.  Other parameters of the same transaction, internal
authorization-model artifacts (policy/entitlement identifiers), and
classifications computed about the Initiating Principal MUST be omitted.  A
dedicated Security Considerations subsection (Transaction Context
Minimization) and a Privacy Considerations bullet cover the disclosure and
correlation risks.

**Rationale:** Some partner calls cannot be completed without transaction
parameters (e.g., the market-data provider needs the ticker).  But `tctx` is
the claim most likely to expose Trust Domain A's internal processing and
customer data, and even with a pairwise `sub`, transaction details can
re-identify the principal by correlation with data AS-B already holds.  Which
fields may be transcribed is deployment-specific and left to the Cross-Domain
Trust Agreement.  (Issue #18, PR #22, October 2026.)

### 12. Recursive Chaining with Pairwise Trust

**Decision:** Trust Domain A / B are roles, not positions.  The profile MAY
be applied recursively: Domain B's perimeter MAY exchange the inbound access
token for its own Txn-Token, and a workload in Domain B then applies this
profile unchanged toward Domain C.  Each crossing is self-contained and
governed only by the Cross-Domain Trust Agreement between the two domains
directly involved.  Whether `txn` is carried forward across further hops is a
deployment decision; deployments wanting end-to-end correlation SHOULD specify
it in their agreements, and deployments wanting to limit linkability SHOULD
use independent transaction identifiers per domain.

**Rationale:** Keeps trust strictly pairwise — no AS needs a relationship
with domains further upstream — and is consistent with Decision #10: no
Txn-Token ever leaves its domain at any hop.  The Security Considerations note
that claims in a Txn-Token-JAG may derive from context the issuer itself
received from upstream.  (Added August 2026.)

### 13. Additional Claims Permitted in the Txn-Token-JAG

**Decision:** The claim list is not closed.  A Txn-Token-JAG MAY carry
additional registered or deployment-specific claims; recipients ignore claims
they do not understand, per RFC 7519 Section 4.

**Rationale:** Consistent with the general preference against accidental
closed enumerations, and with JWT processing norms.  (Issue #4, PR #23,
October 2026.)

### 14. Motivating Dependency: WIMSE AIMS

**Decision:** The Abstract cites the AI Identity Management System draft
(`draft-ietf-wimse-aims`) as a dependent use: AI agents act as workloads that
carry authorization context in Txn-Tokens, and AIMS directs cross-domain
access through identity chaining or a profile of it such as this one.

**Rationale:** Demonstrates concrete demand from agentic deployments.
(Issue #6, PR #24, October 2026.)

---

## Reference Specifications (with version notes)

| Reference | Version used | Notes |
|---|---|---|
| draft-ietf-oauth-identity-chaining | -11 | Normative base. Check for newer versions before submission. |
| draft-ietf-oauth-transaction-tokens | -08 | Normative. Check for newer versions. |
| draft-ietf-oauth-identity-assertion-authz-grant | -03 (April 2026) | Informative. WG-adopted; replaces individual draft-parecki-* versions. The old individual draft (`draft-parecki-oauth-identity-assertion-authz-grant`) is superseded and must NOT be cited. |
| draft-ietf-wimse-arch | -07 | Informative. Referenced for SPIFFE URI / workload identity context. |
| draft-ietf-wimse-workload-creds | -00 | Informative. Listed in the YAML header but not currently cited in the body. |
| draft-ietf-wimse-aims | latest (bare reference) | Informative. Cited in the Abstract as a dependent use (Decision #14). No explicit metadata block in the YAML header, so the build fetches it from bib.ietf.org. |
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
the access token issued by AS-B, while leaving the Txn-Token-JAG as
bearer (it is short-lived and single-use).

### OQ-2: RAR / `authorization_details` Integration

The ID-JAG profile supports `authorization_details` (RFC 9396) in
the grant.  This profile does not yet define how RAR
`authorization_details` from a Txn-Token would be transcribed into
the Txn-Token-JAG.  The Relationship appendix says a future revision
MAY define it.

**Suggested resolution:** Define a `txn_claims.authorization_details`
sub-claim, or allow top-level `authorization_details` in the
Txn-Token-JAG following the same minimization rules as scope.  Now
that Decision #13 permits additional claims, a top-level claim would
not be blocked by the claim list.  Needs discussion with the OAuth WG.

### OQ-3: SPIFFE URI Subject Mapping

Subject Identifier Mapping defers all mapping rules to the Cross-Domain
Trust Agreement (which MUST cover every Initiating Principal type that
may appear).  No SPIFFE URI mapping algorithm is defined or
recommended.

**Suggested resolution:** Either (a) define a RECOMMENDED mapping
algorithm (e.g., strip the trust domain prefix and use the path
component), or (b) explicitly defer to a companion WIMSE document.
Coordinate with the WIMSE WG.

### OQ-4: Human User Sub Mapping — Pairwise vs. Opaque

For human-user-initiated transactions, the normative text defers the
mapping entirely to the Cross-Domain Trust Agreement.  The only
guidance is non-normative (the user-initiated use case mentions "the
user's email address or a pairwise identifier") plus the permission
for `email` in `txn_claims` as a supplementary identifier.  This is
underspecified for interoperability.

**Suggested resolution:** Consider defining a RECOMMENDED default
(e.g., email, given its widespread cross-domain recognizability) and
a RECOMMENDED stable pairwise identifier construction (e.g.,
PPID-style hash of `iss + sub + audience`).

### OQ-5: `txn_claims` Schema Registration

The draft registers the `txn_claims` claim name with IANA but does
not register a schema.  Claims Minimization gives per-claim rules for
`scope`, `rctx`, `tctx`, internal call chain claims, and
supplementary identity claims, and says the Cross-Domain Trust
Agreement SHOULD define the permitted set.

**Suggested resolution:** Either (a) define a small set of RECOMMENDED
`txn_claims` sub-claims (`scope`, `rctx`, `tctx`) as normative, with IANA
registration for each, or (b) explicitly state that the sub-claim
set is deployment-defined via the Cross-Domain Trust Agreement.
The draft currently leans toward (b).

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

### OQ-8: Document Naming (RESOLVED)

Resolved: the name is settled.  It was renamed to include `oauth` in
August 2026 for WG assignment, then shortened to
`draft-fletcher-oauth-txn-token-chaining-profile` in September 2026
for the Datatracker's 50-character limit.  See the Naming section at
the top of this file.  Do not reopen this.

### OQ-9: `txn` Claim — SHOULD or MUST?

The draft is internally inconsistent.  JWT Claims Requirements lists
`txn` under "The following claims SHOULD be present", while Mandatory
Transcriptions says "AS-A MUST include the `txn` claim".  Decision #4
records SHOULD.  The recursive-chaining text (Chaining Across Multiple Trust Domains) and the Privacy
Considerations (operators weigh auditability against linkability)
also assume `txn` can be omitted or replaced.

**Suggested resolution:** Keep SHOULD (consistent with Decision #4 and
the linkability guidance) and move the `txn` requirement out of
Mandatory Transcriptions, stating it once.

### OQ-10: `typ` Value vs. the Txn-Token-JAG Name

The grant was renamed Txn-Token-JAG (PR #21), but its `typ` is still
`txn-chain+jwt`.  The sibling uses `oauth-id-jag+jwt`.

**Suggested resolution:** Decide before IANA registration whether to
align (e.g. `oauth-txn-jag+jwt`) or keep `txn-chain+jwt`.  Changing it
later breaks implementations.

---

## Relationship to ID-JAG Profile — Key Comparison Points

When explaining to the OAuth WG how this profile differs from the
ID-JAG profile, lead with these points in order:

1. **Subject token type**: Txn-Token vs. ID Token / SAML assertion.
2. **Principal scope**: any Initiating Principal (user, system,
   workload, ...) vs. human-user only.
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
| 2026-08 | Pieter Kasselman and Sean O'Dell added as authors; sections on chaining across multiple trust domains added |
| 2026-08 | Document renamed to `draft-fletcher-oauth-transaction-token-chaining-profile` to secure OAuth WG assignment, reversing the earlier no-WG-prefix position (OQ-8) |
| 2026-08 | Build fixes: duplicate BCP 14 boilerplate removed and `{::boilerplate bcp14-tagged}` moved into Conventions and Definitions; `RFC2119`/`RFC8174` dropped from the YAML `normative:` block; Document History brackets escaped |
| 2026-09 | Document renamed to `draft-fletcher-oauth-txn-token-chaining-profile` (47 characters) to fit the Datatracker's 50-character limit on draft names |
| 2026-09 | Document History corrected while preparing the rename: headings relabelled to the "changes after version NN" convention, and the -02 entry backfilled (it had shipped with no history entry of its own) |
| 2026-09 | Decided to republish under the new name as -00 rather than continuing the sequence at -03, relying on the Datatracker Replaces relationship to link back to `draft-fletcher-transaction-token-chaining-profile` |
| 2026-09-10 | `draft-fletcher-oauth-txn-token-chaining-profile-00` published (PR #19) |
| 2026-09 | Title changed to "Transaction Token JWT Authorization Grant Profile ..." and the grant named "Txn-Token-JAG" throughout (PR #21) |
| 2026-10 | `tctx` transcription made explicit with minimization rules, plus new Security and Privacy Considerations text (issue #18, PR #22) |
| 2026-10 | Additional claims explicitly permitted in the Txn-Token-JAG (issue #4, PR #23) |
| 2026-10 | WIMSE AIMS draft cited in the Abstract as a dependent use (issue #6, PR #24) |
| 2026-10-26 | `draft-fletcher-oauth-txn-token-chaining-profile-01` published |
| 2026-11 | DESIGN-NOTES cleanup: `purp` → `scope`, Txn-Token-JAG terminology, Decisions #11–#14 recorded, OQ-3/4/5 refreshed against current text, OQ-9 (`txn` SHOULD/MUST) and OQ-10 (`typ` value) opened |
