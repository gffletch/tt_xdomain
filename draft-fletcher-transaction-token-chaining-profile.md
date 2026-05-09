---
title: "Transaction Token Authorization Grant Profile for OAuth Identity and Authorization Chaining"
abbrev: "Txn-Token Chaining Profile"
docname: draft-fletcher-transaction-token-chaining-profile-latest
category: std
submissiontype: IETF
ipr: trust200902
workgroup: Web Authorization Protocol
keyword:
  - oauth
  - transaction tokens
  - identity chaining
  - cross-domain
  - trust domain

stand_alone: yes
smart_quotes: no
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
  - ins: G. Fletcher
    name: George Fletcher
    organization: Practical Identity LLC
    email: george@practicalidentity.com

normative:
  RFC2119:
  RFC8174:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9700:
  RFC9728:
  I-D.ietf-oauth-identity-chaining:
    title: "OAuth Identity and Authorization Chaining Across Domains"
    target: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-chaining-11
    author:
      - ins: A. Schwenkschuster
        name: Arndt Schwenkschuster
        org: Defakto Security
      - ins: P. Kasselman
        name: Pieter Kasselman
        org: Defakto Security
      - ins: K. Burgin
        name: Kelley Burgin
        org: MITRE
      - ins: M. Jenkins
        name: Michael Jenkins
        org: NSA-CCSS
      - ins: B. Campbell
        name: Brian Campbell
        org: Ping Identity
      - ins: A. Parecki
        name: Aaron Parecki
        org: Okta
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-oauth-identity-chaining-11
  I-D.ietf-oauth-transaction-tokens:
    title: "Transaction Tokens"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/
    author:
      - ins: A. Tulshibagwale
        name: Atul Tulshibagwale
        org: SGNL
      - ins: G. Fletcher
        name: George Fletcher
        org: Practical Identity LLC
      - ins: P. Kasselman
        name: Pieter Kasselman
        org: Defakto Security
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-oauth-transaction-tokens-08

informative:
  RFC9068:
  RFC9396:
  I-D.ietf-wimse-arch:
    title: "Workload Identity in a Multi System Environment (WIMSE) Architecture"
    target: https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/
    author:
      - ins: J. Salowey
        name: Joseph Salowey
        org: CyberArk
      - ins: Y. Rosomakho
        name: Yaroslav Rosomakho
        org: Zscaler
      - ins: H. Tschofenig
        name: Hannes Tschofenig
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-wimse-arch-07
  I-D.ietf-wimse-workload-creds:
    title: "WIMSE Workload Credentials"
    target: https://datatracker.ietf.org/doc/draft-ietf-wimse-workload-creds/
    author:
      - ins: B. Campbell
        name: Brian Campbell
        org: Ping Identity
      - ins: J. Salowey
        name: Joseph Salowey
        org: CyberArk
      - ins: A. Schwenkschuster
        name: Arndt Schwenkschuster
        org: Defakto Security
      - ins: Y. Sheffer
        name: Yaron Sheffer
        org: Intuit
      - ins: Y. Rosomakho
        name: Yaroslav Rosomakho
        org: Zscaler
    date: 2025
    seriesinfo:
      Internet-Draft: draft-ietf-wimse-workload-creds-00
  I-D.ietf-oauth-identity-assertion-authz-grant:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    author:
      - ins: A. Parecki
        name: Aaron Parecki
        org: Okta
      - ins: K. McGuinness
        name: Karl McGuinness
        org: Independent
      - ins: B. Campbell
        name: Brian Campbell
        org: Ping Identity
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-oauth-identity-assertion-authz-grant-03

--- abstract

This specification defines a profile of the OAuth Identity and
Authorization Chaining Across Domains
{{I-D.ietf-oauth-identity-chaining}} mechanism that uses a Transaction
Token (Txn-Token) {{I-D.ietf-oauth-transaction-tokens}} as the subject
token in a Token Exchange {{RFC8693}} request to obtain a JWT
Authorization Grant for crossing a trust domain boundary.

A Txn-Token is scoped to a single trust domain and represents the
full authorization context of an in-progress transaction, regardless
of whether that transaction was initiated by a human user calling an
external API, by an internal system event, or by an automated
workload.  This profile specifies how a service operating within that
trust domain can present its Txn-Token to obtain a JWT Authorization
Grant that carries the necessary context across a trust domain
boundary, enabling an access token to be issued for a partner service,
without exposing internal trust-domain credentials or token formats
beyond the trust boundary.

--- note_Note_to_Readers

*RFC EDITOR: please remove this section before publication*

Discussion of this document takes place on the Web Authorization
Protocol Working Group mailing list (oauth@ietf.org), which is
archived at <https://mailarchive.ietf.org/arch/browse/oauth/>.

Source for this draft and an issue tracker can be found at
<https://github.com/george-fletcher/draft-fletcher-transaction-token-chaining-profile>.

--- middle

# Introduction

Organizations routinely deploy services that, in fulfilling a
transaction for a user or an automated process, must call one or more
partner APIs that lie outside the organization's own trust boundary.
The challenge is to carry the authorization context of the original
transaction — including the identity and authorization of the
initiating principal — across that boundary in a way that is
trustworthy to the partner, without leaking internal credentials or
internal token formats.

Transaction Tokens (Txn-Tokens) {{I-D.ietf-oauth-transaction-tokens}}
address the first half of this problem.  A Txn-Token is a
short-lived, cryptographically signed JWT scoped to a single trust
domain (for example, an enterprise or a cloud service provider's
internal environment).  It is minted by a Transaction Token Service
(TTS) at the point where a transaction enters the trust domain and
captures, in immutable form, the identity of the initiating
principal, the purpose of the transaction, and relevant request
parameters.  Every workload within the trust domain that handles the
transaction receives and validates this Txn-Token, ensuring a
consistent and authoritative authorization context throughout the
internal call chain.

A Txn-Token may represent any of several originating contexts:

External User Request:
: A human user or external client calls an API exposed at the
  trust domain's perimeter (e.g., a financial services API that
  adds a stock to a watch list on behalf of the user, authenticated
  via an OAuth 2.0 access token).  The TTS mints a Txn-Token
  anchored to the user's identity and the authorized scope of that
  external access token.

Internal System Event:
: An internal system triggers processing that has no direct external
  human caller (e.g., an SMTP server receiving an inbound message
  and initiating storage of that message in the recipient's mailbox).
  The TTS mints a Txn-Token representing the system's identity and
  the purpose of the transaction.

Automated Workload Request:
: One workload within the trust domain invokes another as part of
  an automated pipeline (e.g., a scheduled job triggering a data
  aggregation service).  The Txn-Token represents the workload
  identity and the pipeline's authorization scope.

In all three cases, the Txn-Token provides a uniform, internal
representation of the authorization context.  The problem this
specification addresses is what happens when a service within the
trust domain, in the course of executing such a transaction, needs
to call a service in a *different* trust domain — a partner
organization, a SaaS provider, or a third-party API — in order to
complete the transaction.

Consider a mail service within an enterprise trust domain.  Upon
receiving an inbound message via SMTP, the mail service is issued a
Txn-Token representing the mail delivery transaction on behalf of the
recipient user.  Before storing the message, the mail service must
call a partner spam-rating API in the spam service's trust domain.
The mail service cannot present its internal Txn-Token to the spam
service — the Txn-Token is scoped to the enterprise trust domain and
carries internal context that must not be disclosed externally.
Instead, the mail service must obtain a credential that is meaningful
to the spam service's authorization server while preserving the
relevant authorization context of the original transaction.

The OAuth Identity and Authorization Chaining Across Domains
specification {{I-D.ietf-oauth-identity-chaining}} defines a general
mechanism by which a client in Trust Domain A can obtain a JWT
Authorization Grant from the Authorization Server of Trust Domain A
and present it to the Authorization Server of Trust Domain B to
receive an access token.  The base specification deliberately leaves
the choice of subject token type open, allowing profiles to constrain
and specialize the mechanism for specific deployment scenarios.

This specification defines the additional details necessary to use a
Txn-Token as the `subject_token` in the Token Exchange request
described in Section 2.3 of {{I-D.ietf-oauth-identity-chaining}}.
The Txn-Token is consumed by the Authorization Server of Trust Domain
A, which validates it, applies claims transcription and minimization
policy, and issues a JWT Authorization Grant targeted at the
Authorization Server of Trust Domain B.  The JWT Authorization Grant
crosses the trust boundary carrying only the context that Trust
Domain B is authorized to see.  The Txn-Token itself never leaves
Trust Domain A.

This profile is complementary to the Identity Assertion JWT
Authorization Grant profile
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, which targets
deployments where the Resource Authorization Server already trusts a
common IdP for SSO and subject resolution, using an OpenID Connect
ID Token or SAML 2.0 assertion as the subject token.  That profile
is optimized for the human-user, single-sign-on scenario, where the
trust relationship between AS-A and AS-B is mediated through a
shared identity provider.  This profile addresses scenarios where
the trust relationship between AS-A and AS-B is established through a
bilateral or federated Cross-Domain Trust Agreement, and where the
input credential is a Txn-Token representing any authorized
transaction within Trust Domain A.

A detailed structural comparison of the two profiles appears in
{{relationship-to-related-specifications}}.

## Requirements Language

{::boilerplate bcp14-tagged}


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all
capitals, as shown here.

## Roles

The following roles are used in this document.  They extend the role
definitions in {{I-D.ietf-oauth-identity-chaining}}.

Initiating Principal:
: The entity whose authorization context is captured in the Txn-Token.
  The Initiating Principal may be a human user who made an external
  request to Trust Domain A, an internal system acting on its own
  behalf, or an automated workload operating within Trust Domain A.
  The Initiating Principal is not necessarily the same entity as the
  Requesting Workload that performs the cross-domain token exchange.

Requesting Workload:
: A service operating inside Trust Domain A that, in the course of
  processing a transaction, needs to call a Protected Resource in
  Trust Domain B.  The Requesting Workload holds a Txn-Token
  representing the current transaction context and acts as the OAuth
  2.0 client in the Token Exchange flow with AS-A.

Transaction Token Service (TTS):
: The service within Trust Domain A that mints and signs Txn-Tokens.
  The TTS is the authoritative source of transaction authorization
  context within Trust Domain A.  In some deployments the TTS and
  AS-A MAY be co-located; in others they are separate services within
  the same trust domain.

Authorization Server of Trust Domain A (AS-A):
: The OAuth 2.0 Authorization Server within Trust Domain A that
  receives the Token Exchange request from the Requesting Workload,
  validates the presented Txn-Token, applies claims transcription and
  minimization policy, and issues the JWT Authorization Grant targeted
  at AS-B.

Authorization Server of Trust Domain B (AS-B):
: The OAuth 2.0 Authorization Server within Trust Domain B that
  receives the JWT Authorization Grant from the Requesting Workload
  and issues an access token for the Protected Resource.

Protected Resource:
: The resource server in Trust Domain B that the Requesting Workload
  needs to call in order to complete the transaction in progress in
  Trust Domain A.

## Terms

Transaction:
: A unit of work initiated by an Initiating Principal that may span
  multiple workloads within Trust Domain A and that has a single,
  coherent authorization context.  A transaction is identified by the
  `txn` claim in the Txn-Token.

Trust Domain:
: A deployment-specific security and administrative boundary within
  which services, identifiers, credentials, and policy decisions are
  mutually trusted.  Txn-Tokens are scoped to a single trust domain.
  In this specification, Trust Domain A is the trust domain in which
  the transaction originates and in which the Requesting Workload
  operates.  Trust Domain B is the trust domain in which the Protected
  Resource and AS-B operate.

Cross-Domain Trust Agreement:
: A bilateral or federated configuration through which AS-A and AS-B
  establish mutual trust, permitting AS-A to issue JWT Authorization
  Grants that AS-B will accept, and defining the subject identifier
  mappings, permitted claims, and authorization policy that apply to
  cross-domain requests.  The mechanism for establishing this trust
  is out of scope for this specification, but MUST be established
  prior to any cross-domain token exchange under this profile.


# Overview

## Transaction Token Context Within a Trust Domain

A transaction enters Trust Domain A at its perimeter.  The
initiating event may be:

(a) **An inbound API call from an external client**, in which case
  the external client presents an OAuth 2.0 access token or similar
  credential at the trust domain's API gateway;

(b) **An internal system event**, such as an SMTP server receiving
  an inbound message, where the triggering input arrives from outside
  the enterprise boundary; or

(c) **An automated workload trigger**, with no direct external caller,
  such as a scheduled job or an event-driven pipeline invocation.

In all cases, the workload that first handles the transaction requests
a Txn-Token from the TTS, presenting whatever inbound credential or
context is available.  The TTS validates the inbound context and
mints a Txn-Token that captures the Initiating Principal's identity
(which may be a user identity, a system identity, or a workload
identity), the purpose of the transaction (`scope`), and relevant
request parameters (`rctx`).  The Txn-Token is propagated to all
downstream workloads within Trust Domain A that participate in
processing the transaction.

## Cross-Domain Invocation

When a Requesting Workload within Trust Domain A determines that it
needs to call a Protected Resource in Trust Domain B in order to
complete the transaction, it follows the flow defined in this profile.
The complete end-to-end sequence is illustrated in {{fig-flow}}.

~~~ ascii-art
+----------+  +-----------+  +----------+  +---------+  +---------+
|Initiating|  |Requesting |  |  TTS /   |  |  AS-B   |  |Protected|
| Request  |  | Workload  |  |  AS-A    |  |(Trust B)|  |Resource |
|(Perimeter|  |(Trust A)  |  |(Trust A) |  |         |  |(Trust B)|
| Trust A) |  |           |  |          |  |         |  |         |
+----+-----+  +-----+-----+  +----+-----+  +----+----+  +----+----+
     |               |            |              |             |
     | (1) Inbound   |            |              |             |
     |   Request     |            |              |             |
     | (any origin)  |            |              |             |
     |-------------->|            |              |             |
     |               |            |              |             |
     |               | (2) Request|              |             |
     |               |  Txn-Token |              |             |
     |               |----------->|              |             |
     |               |            |              |             |
     |               | (3) Txn-   |              |             |
     |               |   Token    |              |             |
     |               |<- - - - - -|              |             |
     |               |            |              |             |
     |               | (4) Discover AS-B         |             |
     |               |...(RFC9728)...............|             |
     |               |            |              |             |
     |               | (5) Token Exchange        |             |
     |               |   [RFC8693]|              |             |
     |               | subject_token=Txn-Token   |             |
     |               | audience=AS-B issuer URL  |             |
     |               |----------->|              |             |
     |               |            |              |             |
     |               | (6) JWT Authorization     |             |
     |               |    Grant   |              |             |
     |               |<- - - - - -|              |             |
     |               |            |              |             |
     |               | (7) Present JWT Grant     |             |
     |               |   [RFC7523]               |             |
     |               |-------------------------->|             |
     |               |            |              |             |
     |               | (8) Access Token          |             |
     |               |<- - - - - - - - - - - - - |             |
     |               |            |              |             |
     |               | (9) Call Protected Resource             |
     |               |---------------------------------------->|
     |               |            |              |             |
~~~
{: #fig-flow title="Transaction Token Chaining Flow"}

The steps are as follows:

1. An inbound request arrives at the Requesting Workload's perimeter.
   This may be an OAuth 2.0-protected API call from an external user
   or client, an SMTP message delivery, a scheduled job trigger, or
   any other initiating event.

2. The Requesting Workload (or the first workload within Trust Domain
   A that receives the transaction) requests a Txn-Token from the TTS,
   presenting the available inbound context (e.g., the external OAuth
   2.0 access token, an internal system credential, or a workload
   identity), following the Txn-Token issuance procedures defined in
   {{I-D.ietf-oauth-transaction-tokens}}.  The TTS records the
   Initiating Principal's identity in the `sub` claim and the
   transaction purpose in the `scope` claim of the Txn-Token.

3. The TTS issues a Txn-Token to the Requesting Workload.  The
   Txn-Token is scoped to Trust Domain A and MUST NOT be presented
   to any entity outside Trust Domain A.

4. The Requesting Workload discovers AS-B using the mechanisms defined
   in Section 2.2 of {{I-D.ietf-oauth-identity-chaining}}, such as
   the `authorization_servers` metadata property in the Protected
   Resource Metadata {{RFC9728}} published by the Protected Resource.
   The Requesting Workload obtains AS-B's issuer URL for use as the
   `audience` parameter in the Token Exchange request.

5. The Requesting Workload presents the Txn-Token as the `subject_token`
   in an OAuth 2.0 Token Exchange {{RFC8693}} request to AS-A,
   identifying AS-B in the `audience` parameter and optionally
   specifying the target Protected Resource in the `resource`
   parameter.  See {{token-exchange-request-parameters}} for the
   full parameter specification.

6. AS-A validates the Txn-Token, verifies that a Cross-Domain Trust
   Agreement exists with the indicated AS-B, applies subject
   identifier mapping ({{subject-identifier-mapping}}) and claims
   minimization ({{claims-transcription}}), and issues a signed JWT
   Authorization Grant.  The Txn-Token is consumed entirely within
   Trust Domain A and is not forwarded.

7. The Requesting Workload presents the JWT Authorization Grant to
   AS-B using the JWT Profile for OAuth 2.0 Authorization Grants
   {{RFC7523}}.

8. AS-B validates the JWT Authorization Grant and issues an access
   token for the Protected Resource.

9. The Requesting Workload calls the Protected Resource with the
   access token, completing the cross-domain portion of the
   transaction.


# Transaction Token as Subject Token

## Subject Token Requirements

When this profile is used, the `subject_token` in the Token Exchange
request (Step 5 of {{fig-flow}}) MUST be a Txn-Token as defined in
{{I-D.ietf-oauth-transaction-tokens}}.

The `subject_token_type` parameter MUST be:

~~~ abnf
subject_token_type =
    "urn:ietf:params:oauth:token-type:txn_token"
~~~

This value is defined in {{I-D.ietf-oauth-transaction-tokens}}.

The Txn-Token presented as the `subject_token` MUST satisfy all of
the validity requirements specified in
{{I-D.ietf-oauth-transaction-tokens}}, including:

- The Txn-Token MUST NOT be expired.

- The Txn-Token MUST be signed and verifiable by AS-A using keys
  published by the TTS.

- The Txn-Token's `aud` claim MUST identify AS-A (or a value that
  AS-A accepts as a valid audience for presented subject tokens).

A Txn-Token that has already expired or whose signature cannot be
verified MUST be rejected as defined in Section 2.2.2 of {{RFC8693}}.

## Txn-Token Initiating Principal Context {#txn-token-initiating-principal-context}

The Txn-Token's `sub` claim identifies the Initiating Principal of
the transaction.  AS-A MUST be capable of processing Token Exchange
requests for all of the following Initiating Principal types:

Human User Identity:
: The `sub` claim identifies a human user whose identity was
  established when the transaction entered Trust Domain A via an
  OAuth 2.0-protected API call.  In this case the `sub` value is
  typically derived from the user's identity in the external access
  token presented at the API gateway, and the Txn-Token's `rctx`
  claim captures relevant attributes of the external request (such
  as the OAuth client identifier and originating IP address).

System Identity:
: The `sub` claim identifies an internal system component (such as
  an SMTP server or a messaging gateway) acting in its own right,
  with no external user as the Initiating Principal.  The `scope`
  claim is particularly significant in this case, as it conveys the
  reason for the transaction in the absence of a user-facing
  authorization context.

Workload Identity:
: The `sub` claim identifies an automated workload (such as a
  scheduled job or pipeline service).  Workload identifiers MAY take
  the form of SPIFFE URIs {{I-D.ietf-wimse-arch}} when
  WIMSE-compatible infrastructure is in use within Trust Domain A.

The claims transcription rules in {{claims-transcription}} and the
subject identifier mapping rules in {{subject-identifier-mapping}}
apply regardless of which Initiating Principal type the Txn-Token
represents.  AS-A MUST map the `sub` claim to an identifier
appropriate for Trust Domain B, applying principal-type-specific
mapping logic as defined in {{subject-identifier-mapping}}.


## Token Exchange Request Parameters {#token-exchange-request-parameters}

In addition to the subject token requirements in
{{subject-token-requirements}}, the Token Exchange request
({{RFC8693}} Section 2.1) MUST include the following parameters when
this profile is in use.

### Identifying the Target Authorization Server and Resource

The Token Exchange request MUST identify the Authorization Server in
Trust Domain B (AS-B) that should be the intended audience of the
resulting JWT Authorization Grant.  This profile uses the `audience`
parameter for this purpose, following the convention established in
{{I-D.ietf-oauth-identity-assertion-authz-grant}} Section 4.3.

`audience`:
: REQUIRED.  The Issuer URL of AS-B as defined in Section 2 of
  {{RFC8414}}.  AS-A uses this value to set the `aud` claim of the
  resulting JWT Authorization Grant, so that AS-B can verify the
  grant was issued for it.  The value MUST be the `issuer`
  identifier as advertised in AS-B's authorization server metadata,
  or a logical name for AS-B that AS-A has been configured to
  recognize and map to AS-B's actual issuer identifier.

`resource`:
: OPTIONAL.  A URI that identifies the specific Protected Resource
  (resource server) in Trust Domain B that the Requesting Workload
  intends to call, as defined in Section 2 of {{RFC8707}}.  When
  present, AS-A SHOULD include this value in the `resource` claim of
  the JWT Authorization Grant so that AS-B can issue a resource-bound
  access token.

The two parameters serve distinct purposes that MUST NOT be
conflated:

- `audience` identifies the **authorization server** (AS-B) that
  will receive and validate the JWT Authorization Grant.  Its value
  becomes the `aud` claim of the grant.

- `resource` identifies the **resource server** (the Protected
  Resource — the partner API to be called) for which an access token
  is ultimately sought.  AS-B uses it when scoping the access token;
  it does not appear in the `aud` claim.

Implementations MUST use `audience` to identify AS-B and MUST NOT
pass the AS-B issuer URL as the `resource` parameter.

### Remaining Parameters

`grant_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:
: REQUIRED.  The Txn-Token as described in
  {{subject-token-requirements}}.

`subject_token_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:token-type:txn_token`.

`requested_token_type`:
: OPTIONAL.  When present, the value MUST be
  `urn:ietf:params:oauth:token-type:jwt`.  If absent, AS-A MUST
  still produce a JWT Authorization Grant conforming to this profile
  when the other parameters conform to this profile.

`scope`:
: OPTIONAL.  The space-separated list of scopes requested for the
  resulting JWT Authorization Grant, constraining the access token
  that AS-B will ultimately issue.  AS-A MUST NOT issue a JWT
  Authorization Grant with a scope that exceeds the `scope` claim of
  the presented Txn-Token.  If the requested scope is wider than the
  Txn-Token's scope, AS-A MUST either narrow the scope to the
  intersection or deny the request.

The `actor_token` and `actor_token_type` parameters defined in
{{RFC8693}} are not used in this profile.

### Example Token Exchange Request

The following is a non-normative example conforming to this profile.
A mail service workload in an enterprise (Trust Domain A) has
received an SMTP message and holds a Txn-Token representing a
`mail-delivery` transaction.  The mail service needs to call a
spam-rating API operated by a partner spam service whose
Authorization Server is `https://as.spamsvc.example` and whose
spam-rating API is `https://api.spamsvc.example/spam-rating`.

~~~ http-message
POST /token HTTP/1.1
Host: as.enterprise.example
Content-Type: application/x-www-form-urlencoded
Authorization: Bearer <mail-service-client-credential>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<txn-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atxn_token
&audience=https%3A%2F%2Fas.spamsvc.example
&resource=https%3A%2F%2Fapi.spamsvc.example%2Fspam-rating
&scope=spam.rating.read
~~~

### Token Exchange Response

If the request is valid and the Requesting Workload is authorized to
receive a JWT Authorization Grant for the indicated audience, AS-A
returns a Token Exchange response as defined in Section 2.2 of
{{RFC8693}}.

`access_token`:
: REQUIRED.  The JWT Authorization Grant.  (Token Exchange uses the
  `access_token` field for the returned token for historical
  compatibility reasons; this is not an OAuth access token.)

`issued_token_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:token-type:jwt`.

`token_type`:
: REQUIRED.  The value MUST be `N_A`, as this is not an OAuth
  access token and token-type-based sender constraining does not
  apply at this step.

`expires_in`:
: RECOMMENDED.  The lifetime of the JWT Authorization Grant in
  seconds.  This value SHOULD reflect the `exp` claim of the
  returned grant JWT and SHOULD be short (see
  {{jwt-claims-requirements}}).

`refresh_token`:
: This parameter SHOULD NOT be present.

On error, AS-A returns an error response as defined in Section 5.2
of {{RFC6749}} and Section 2.2.2 of {{RFC8693}}.

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token": "eyJ...<JWT Authorization Grant>...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_type": "N_A",
  "expires_in": 60
}
~~~


# Processing Rules

## AS-A Processing Rules

Upon receipt of a Token Exchange request conforming to this profile,
AS-A MUST perform the following steps:

1. Authenticate the client (Requesting Workload) using the mechanisms
   specified in Section 5.1 of {{I-D.ietf-oauth-identity-chaining}}
   and Section 2.5 of {{RFC9700}}.

2. Validate the Txn-Token signature using the public keys of the TTS
   that issued it.  AS-A MUST establish a trust relationship with the
   TTS; in deployments where the TTS and AS-A are co-located this is
   straightforward.  Where they are separate services, AS-A MUST be
   configured with the TTS's `jwks_uri` or equivalent key material.

3. Validate that the Txn-Token is not expired.

4. Validate that the `aud` claim of the Txn-Token is acceptable to
   AS-A (i.e., the Txn-Token was issued for consumption by AS-A or
   the TTS acting on AS-A's behalf).

5. Resolve the target AS-B from the `audience` parameter.  AS-A MUST
   verify that the `audience` value corresponds to a known AS-B with
   which a Cross-Domain Trust Agreement has been established.  If the
   `audience` is unknown or policy prohibits issuing a JWT
   Authorization Grant for that audience, AS-A MUST return an error
   as specified in Section 2.2.2 of {{RFC8693}}.  AS-A MUST NOT
   accept a resource server URI in the `audience` parameter in place
   of an AS-B issuer identifier.

6. If the `resource` parameter is present, validate that it
   identifies a Protected Resource within Trust Domain B consistent
   with the indicated AS-B.  AS-A SHOULD propagate the `resource`
   value into the `resource` claim of the JWT Authorization Grant.

7. Validate that the requested `scope`, if present, does not exceed
   the `scope` claim of the Txn-Token.  AS-A MUST NOT issue a JWT
   Authorization Grant with broader scope than the Txn-Token asserts.

8. Determine the Initiating Principal type from the Txn-Token and
   apply the appropriate subject identifier mapping as described in
   {{subject-identifier-mapping}}.  If no mapping can be determined,
   AS-A MUST return an error.

9. Apply claims transcription and minimization policy as described in
   {{claims-transcription}}.

10. Construct and sign the JWT Authorization Grant as described in
    {{jwt-authorization-grant}}, setting the `aud` claim to the AS-B
    issuer identifier resolved in step 5.

11. Return the JWT Authorization Grant in the Token Exchange response
    as described in {{token-exchange-request-parameters}}.

## AS-B Processing Rules

Upon receipt of a JWT Bearer grant request ({{RFC7523}}) conforming
to this profile, AS-B MUST perform the following steps in addition
to the processing rules specified in Section 2.4.2 of
{{I-D.ietf-oauth-identity-chaining}}:

1. Validate the `typ` header of the JWT Authorization Grant.  The
   value MUST be `txn-chain+jwt` as defined in {{jwt-authorization-grant}}.

2. Validate that the `aud` claim matches AS-B's own issuer identifier.

3. Validate that the `iss` claim identifies an AS-A with which a
   Cross-Domain Trust Agreement has been established, and validate
   the JWT signature using the public keys advertised by that AS-A.

4. Validate that the JWT is not expired and that the `jti` value has
   not been previously presented (single-use enforcement).

5. Resolve the subject.  The resolution strategy depends on the
   Initiating Principal type reflected in the `sub` claim:

   a. For human user identities, AS-B SHOULD attempt to match the
      `sub` claim (or a supplementary identifier such as `email` in
      `txn_claims`) against its local user directory or subject
      registry.

   b. For system or workload identities, AS-B SHOULD evaluate the
      `sub` claim against its configured cross-domain workload
      access policy.

   c. If subject resolution fails and the Cross-Domain Trust
      Agreement does not permit Just-In-Time provisioning, AS-B
      MUST return an error.

6. If present, evaluate the `txn_claims` claim to apply
   context-aware authorization policy (see {{claims-transcription}}),
   for example verifying that the `scope` value is consistent with
   the requested scope.

7. Issue an access token constrained by the `scope` and, if present,
   the `resource` claim in the JWT Authorization Grant.  AS-B SHOULD
   NOT issue refresh tokens, consistent with Section 5.4 of
   {{I-D.ietf-oauth-identity-chaining}}.


# JWT Authorization Grant {#jwt-authorization-grant}

## Grant Format

The JWT Authorization Grant produced by AS-A in response to a Token
Exchange request conforming to this profile is a JWT {{RFC7519}}
that MUST conform to the JWT Authorization Grant requirements
specified in Section 2.3.3 of {{I-D.ietf-oauth-identity-chaining}}.

### JWT Header

typ:
: REQUIRED.  The value MUST be `txn-chain+jwt`.  Explicit JWT
  typing is REQUIRED by this profile, consistent with JSON Web Token
  Best Current Practices {{RFC8725}}.

alg:
: REQUIRED.  An asymmetric signing algorithm.  Deployments SHOULD
  use PS256, PS384, PS512, ES256, ES384, or ES512 as defined in
  {{RFC7519}}.  The `none` algorithm and symmetric algorithms are
  prohibited.

kid:
: RECOMMENDED.  The key identifier corresponding to the signing key.

### JWT Claims Requirements {#jwt-claims-requirements}

The following claims MUST be present:

iss:
: REQUIRED.  The Issuer identifier of AS-A ({{RFC8414}} Section 2).

sub:
: REQUIRED.  The Initiating Principal's identity as mapped by AS-A
  according to {{subject-identifier-mapping}}.  The value MUST be
  meaningful to AS-B within the context of the Cross-Domain Trust
  Agreement.

aud:
: REQUIRED.  The Issuer URL of AS-B, derived from the `audience`
  parameter of the Token Exchange request.  MUST be a single value
  to prevent grant replay at an unintended authorization server.

iat:
: REQUIRED.  Issuance time ({{RFC7519}} Section 4.1.6).

exp:
: REQUIRED.  Expiration time ({{RFC7519}} Section 4.1.4).  The
  lifetime SHOULD be short.  Deployments SHOULD use a value no
  greater than 300 seconds and SHOULD prefer values of 60 seconds or
  less, consistent with the short-lived nature of Txn-Tokens.

jti:
: REQUIRED.  A unique identifier for this JWT ({{RFC7519}} Section
  4.1.7).  AS-B MUST enforce single-use semantics by tracking
  presented `jti` values within the grant's validity window.

scope:
: RECOMMENDED.  The authorized scope ({{RFC6749}} Section 3.3).
  MUST NOT be wider than the `scope` claim of the source Txn-Token.

The following claims SHOULD be present:

txn:
: The unique transaction identifier from the originating Txn-Token,
  as defined in {{I-D.ietf-oauth-transaction-tokens}}.  Preserves
  end-to-end transaction correlation across the domain boundary.
  AS-B SHOULD record this value in its audit logs.

The following claims MAY be present:

resource:
: A URI or array of URIs identifying the Protected Resource(s) in
  Trust Domain B, as defined in Section 2 of {{RFC8707}}.  Derived
  from the `resource` parameter of the Token Exchange request.  AS-B
  SHOULD use this to issue a resource-bound access token.  This
  value identifies the **resource server** (the partner API), not
  AS-B.  It is distinct from — and must never duplicate — the `aud`
  claim.

txn_claims:
: A JSON object containing a curated subset of Txn-Token claims,
  selected and minimized per the policy in {{claims-transcription}}.
  AS-B MAY use these claims for context-aware authorization decisions.

cnf:
: If sender-constraining is in use (see {{sender-constraining}}),
  the confirmation method claim conveying the Requesting Workload's
  public key, as defined in {{RFC7800}}.

### Example JWT Authorization Grant

The following is a non-normative example corresponding to the mail
service scenario in {{token-exchange-request-parameters}}.  The
Initiating Principal is the mail service's system identity
(`mail-gateway@enterprise.example`) and the Txn-Token's `rctx`
carries the SMTP envelope sender.  The `scope` and a minimized `rctx`
are transcribed into `txn_claims`.

Header:

~~~ json
{
  "typ": "txn-chain+jwt",
  "alg": "ES256",
  "kid": "as-enterprise-2026-01"
}
~~~

Claims:

~~~ json
{
  "iss": "https://as.enterprise.example",
  "sub": "mail-gateway@enterprise.example",
  "aud": "https://as.spamsvc.example",
  "iat": 1746700000,
  "exp": 1746700060,
  "jti": "8f14e45f-ceee-467a-a19e-ab8f290a1f30",
  "scope": "spam.rating.read",
  "resource": "https://api.spamsvc.example/spam-rating",
  "txn": "a9b2c3d4-e5f6-7890-abcd-ef1234567890",
  "txn_claims": {
    "scope": "mail-delivery",
    "rctx": {
      "smtp_from": "sender@external.example"
    }
  }
}
~~~

The `aud` claim (`https://as.spamsvc.example`) identifies the
authorization server of Trust Domain B.  The `resource` claim
(`https://api.spamsvc.example/spam-rating`) identifies the specific
API endpoint.  These are distinct values serving distinct purposes.


# Claims Transcription {#claims-transcription}

This profile constrains and extends the claims transcription rules of
Section 2.5 of {{I-D.ietf-oauth-identity-chaining}} as follows.

## Mandatory Transcriptions

AS-A MUST derive the `sub` claim of the JWT Authorization Grant from
the `sub` claim of the Txn-Token, applying the subject identifier
mapping defined in {{subject-identifier-mapping}}.

AS-A MUST include the `txn` claim from the Txn-Token as the `txn`
claim in the JWT Authorization Grant, preserving the transaction
correlation identifier across the domain boundary.

## Constrained Scope Transcription

The scope in the JWT Authorization Grant MUST be the intersection of
the Txn-Token's `scope` claim and the `scope` parameter of the Token
Exchange request (if present).  AS-A MUST NOT expand scope beyond the
Txn-Token's scope under any circumstances.

## Subject Identifier Mapping {#subject-identifier-mapping}

The `sub` claim of the Txn-Token identifies the Initiating Principal
within Trust Domain A's namespace.  AS-A MUST translate this
identifier to a form that is both meaningful and authorized for use
in Trust Domain B, according to the mapping rules defined in the
Cross-Domain Trust Agreement.  The appropriate strategy depends on
the Initiating Principal type:

Human User Identity:
: AS-A SHOULD map the user's `sub` to a cross-domain user
  identifier that AS-B can resolve to a local subject.  This MAY be
  an email address, a stable pairwise identifier, or another
  identifier agreed in the Cross-Domain Trust Agreement.  If AS-B
  supports Just-In-Time provisioning and the Cross-Domain Trust
  Agreement permits it, supplementary identity attributes (e.g.,
  `email`) MAY be included in the `txn_claims` object to facilitate
  subject resolution at AS-B.

System Identity:
: AS-A SHOULD map the system component's identifier to a
  cross-domain system identity agreed in the Cross-Domain Trust
  Agreement.  A common pattern is a structured identifier of the
  form `system:<component>@<trust-domain>` (e.g.,
  `system:mail-gateway@enterprise.example`).

Workload Identity:
: AS-A SHOULD map the workload identifier to a form recognized by
  AS-B.  Where SPIFFE URIs {{I-D.ietf-wimse-arch}} are in use, AS-A
  SHOULD apply the SPIFFE URI translation rules defined in the
  Cross-Domain Trust Agreement (e.g., preserving the structural
  SPIFFE URI form or mapping to a local workload identifier at AS-B).

If no mapping can be determined for the Initiating Principal, AS-A
MUST deny the Token Exchange request.

## Claims Minimization

Txn-Tokens MUST NOT be forwarded across trust domain boundaries.
The JWT Authorization Grant is the only artifact that crosses the
boundary, and AS-A MUST apply strict claims minimization.

The optional `txn_claims` object in the JWT Authorization Grant MAY
carry a curated subset of Txn-Token claims that are relevant to
AS-B's authorization policy.  AS-A MUST apply the following
minimization rules:

Purpose Claim (`scope`):
: SHOULD be included when it is meaningful to AS-B's authorization
  policy (e.g., to enable the Protected Resource to apply different
  handling based on transaction type).

Requester Context (`rctx`):
: MAY be included in a minimized form.  Information relevant to the
  cross-domain request (e.g., the originating client IP address for
  a user-initiated transaction, or the SMTP envelope sender address
  for a mail delivery transaction) MAY be included.  Internal
  network addresses, intermediate workload identifiers, and
  internal infrastructure topology details MUST be omitted.

Internal Call Chain:
: Claims that record intermediate workloads or the internal call
  chain within Trust Domain A MUST NOT be included in `txn_claims`.

Supplementary Identity Claims:
: For human user Initiating Principals, claims such as `email` MAY
  be included in `txn_claims` if the Cross-Domain Trust Agreement
  explicitly permits their disclosure and AS-B requires them for
  subject resolution.

The Cross-Domain Trust Agreement SHOULD define the set of claims
permitted to appear in `txn_claims` and their expected semantics,
to ensure that AS-A and AS-B have a shared, normative understanding
of each transcribed claim.


# Authorization Server Metadata

This profile adds to the Authorization Server Metadata framework
defined in {{RFC8414}} and Section 3 of
{{I-D.ietf-oauth-identity-chaining}}.

An Authorization Server that supports this profile MUST include the
value `urn:ietf:params:oauth:token-type:txn_token` in its
`identity_chaining_requested_token_types_supported` metadata
parameter.  This signals that the Authorization Server accepts
Txn-Tokens as the `subject_token` in a Token Exchange request
conforming to this profile.


# Security Considerations

## Client Authentication

The Requesting Workload MUST authenticate to AS-A when performing
the Token Exchange request.  The use of asymmetric key-based client
authentication (e.g., a JWT client assertion per {{RFC7523}}) is
RECOMMENDED.  Static shared secrets SHOULD NOT be used.  AS-A SHOULD
follow the client authentication guidance in Section 2.5 of
{{RFC9700}}.

## Sender Constraining Tokens {#sender-constraining}

AS-B SHOULD issue sender-constrained access tokens.  Both DPoP
(OAuth 2.0 Demonstrating Proof of Possession) and Mutual-TLS
({{RFC9700}} Section 2.3) are RECOMMENDED mechanisms.

When AS-A acts as the client toward AS-B (the authorization-server-
as-client topology described in Appendix B.2 of
{{I-D.ietf-oauth-identity-chaining}}), the delegated key binding
mechanism described in Appendix B.3 of that document SHOULD be used.
AS-A MUST verify proof of possession of the Requesting Workload's
key and convey it to AS-B using the `cnf` claim in the JWT
Authorization Grant.

## Txn-Token Confidentiality

A Txn-Token is a bearer credential scoped to Trust Domain A and MUST
NOT be forwarded to any entity outside Trust Domain A.  All
communication between the Requesting Workload and AS-A that involves
a Txn-Token MUST occur over mutually authenticated TLS.  Txn-Token
lifetimes SHOULD be short to limit the window of exposure if a token
is captured in transit.

## JWT Authorization Grant Replay Prevention

The JWT Authorization Grant is a bearer token.  AS-B MUST enforce
single-use semantics on the `jti` claim.  AS-A SHOULD set a short
validity lifetime (see {{jwt-claims-requirements}}).  Additional
guidance is provided in Section 5.5 of
{{I-D.ietf-oauth-identity-chaining}}.

## Scope Boundary Enforcement

AS-A MUST enforce that the JWT Authorization Grant scope does not
exceed the Txn-Token's scope.  AS-B MUST independently enforce that
the access token it issues does not convey scope exceeding the JWT
Authorization Grant.  These controls together prevent the chaining
mechanism from being used to escalate privileges beyond the
originating transaction's authorized scope.

## Cross-Domain Trust Agreement Integrity

Operators MUST ensure that:

- AS-A issues JWT Authorization Grants only for AS-B instances with
  which a bilateral Cross-Domain Trust Agreement has been explicitly
  established and is actively maintained.

- AS-B accepts JWT Authorization Grants only from AS-A instances
  listed in its trusted issuers configuration.

- The Cross-Domain Trust Agreement, including subject identifier
  mappings and permitted `txn_claims`, is reviewed whenever the
  participating services or their authorization policies change.

## Refresh Tokens

AS-B SHOULD NOT issue refresh tokens.  Because Txn-Tokens are
short-lived and transaction-specific, re-obtaining a new Txn-Token
and repeating the chaining flow is the correct renewal mechanism.
Issuing a refresh token would decouple the access lifetime from the
originating transaction's authorization context and create a
persistent credential outside the control of Trust Domain A.


# Privacy Considerations

Txn-Tokens may contain claims that relate to the Initiating
Principal, including personal identity information for
human-user-initiated transactions (e.g., user identifier, email
address, IP address) that may be subject to applicable privacy
regulations.

AS-A MUST apply claims minimization ({{claims-transcription}})
before issuing a JWT Authorization Grant.  Specifically:

- Only identity claims necessary for AS-B to resolve the subject and
  apply authorization policy SHOULD be included in `txn_claims`.

- Claims that could be used to reconstruct internal activity patterns
  within Trust Domain A MUST NOT be included.

- The Cross-Domain Trust Agreement MUST specify which identity claims
  AS-A is permitted to disclose to AS-B, consistent with the data
  handling and privacy policies of both organizations.

The `txn` claim enables end-to-end transaction correlation across the
domain boundary.  Operators SHOULD evaluate whether the auditability
benefits outweigh the privacy implications for their specific
deployment, particularly for human-user-initiated transactions.


# IANA Considerations

## JWT Typ Registration

This specification requests registration of the following value in
the "JSON Web Signature and Encryption Header Parameters" registry
(maintained by IANA):

- Header Parameter Name: `txn-chain+jwt`
- Header Parameter Description: JWT type for a Transaction Token
  Chaining Authorization Grant as defined in this document
- Change Controller: IETF
- Specification Document(s): {{jwt-authorization-grant}} of this
  document

## JWT Claims Registry

This specification requests registration of the following claim name
in the "JSON Web Token Claims" registry (maintained by IANA):

- Claim Name: `txn_claims`
- Claim Description: Transcribed claims from a Transaction Token,
  included in a JWT Authorization Grant to convey cross-domain
  authorization context
- Change Controller: IETF
- Specification Document(s): {{claims-transcription}} of this
  document


--- back

# Use Cases {#use-cases}

The following use cases illustrate the three Initiating Principal
types described in {{txn-token-initiating-principal-context}}, each
demonstrating a scenario where a workload within a trust domain must
call a partner service in a separate trust domain to complete the
transaction.

## User-Initiated External API Call Requiring a Partner Service

A financial services enterprise exposes a portfolio management API to
its customers.  A customer uses a mobile application to add a stock
to their watch list, calling `POST /watchlist` at the enterprise's
API gateway with an OAuth 2.0 access token.

The API gateway workload requests a Txn-Token from the TTS,
presenting the user's access token as the inbound credential.  The
TTS mints a Txn-Token with `sub` set to the user's enterprise
identifier, `scope` set to `watchlist-update`, and `rctx` capturing
the mobile client's OAuth client identifier and IP address.  This
Txn-Token propagates through the internal portfolio service call
chain.

To enrich the watch list entry with current market data, the
portfolio service must call a market-data API operated by a partner
financial data provider in Trust Domain B.  The portfolio service
exchanges the Txn-Token for a JWT Authorization Grant using this
profile.  AS-A maps the user's enterprise identifier to a
cross-domain user identifier agreed with the partner (e.g., the
user's email address or a pairwise identifier), and includes a
minimized `txn_claims` carrying `scope: watchlist-update`.

The partner's authorization server issues an access token that
identifies the user (enabling per-user rate limiting and audit
logging at the partner) without receiving the enterprise's internal
Txn-Token, internal access token, or internal user database
identifiers.

## System-Initiated Event Requiring a Partner Service

An enterprise mail service receives an inbound email message via
SMTP.  The SMTP server is an internal system component operating
under its own system credential; no external OAuth client is
involved.  The SMTP server requests a Txn-Token from the TTS with
`sub` set to its system identity (`system:mail-gateway@enterprise.example`),
`scope` set to `mail-delivery`, and `rctx` carrying the SMTP envelope
sender address and the recipient user's internal identifier.  This
Txn-Token propagates to the mail storage service workload.

Before storing the message in the recipient's mailbox, the mail
storage service must call a spam-rating API operated by a partner
spam service in Trust Domain B (whose Authorization Server is
`https://as.spamsvc.example` and whose spam-rating API is
`https://api.spamsvc.example/spam-rating`).

The mail storage service exchanges the Txn-Token for a JWT
Authorization Grant using this profile.  AS-A maps the system
identity to the cross-domain service identifier agreed with the spam
service, and includes a minimized `txn_claims` carrying
`scope: mail-delivery` and `rctx.smtp_from` (the envelope sender
address, stripped of internal routing metadata).

The spam service's authorization server issues an access token for
the spam-rating API.  The spam service can apply per-sender and
per-recipient policy based on `txn_claims`, enabling personalized
spam filtering without requiring the enterprise to expose internal
user tokens or the Txn-Token outside its trust boundary.

## Automated Workload Requiring a Partner Service

An enterprise data platform runs a nightly telemetry aggregation job.
The job is an automated workload with no direct external caller,
triggered by an internal scheduler.  The scheduler requests a
Txn-Token from the TTS with `sub` set to the job's SPIFFE workload
URI (`spiffe://enterprise.example/telemetry/nightly-agg`), `scope`
set to `telemetry-aggregation`, and no user context in `rctx`.

To complete the aggregation, the job must query a third-party
analytics API in Trust Domain B.  The job exchanges the Txn-Token
for a JWT Authorization Grant using this profile.  AS-A maps the
SPIFFE workload URI to a cross-domain workload identifier agreed with
the analytics provider, and includes `scope: telemetry-aggregation`
in `txn_claims`.

The analytics provider's authorization server issues a scoped access
token.  The `txn` claim in the JWT Authorization Grant allows the
analytics provider to correlate API calls to the originating job run
for billing and audit purposes, without receiving the internal SPIFFE
URI or other Trust Domain A infrastructure details.


# Relationship to Related Specifications {#relationship-to-related-specifications}

This specification is one of a family of profiles of
{{I-D.ietf-oauth-identity-chaining}}.

## Identity Assertion JWT Authorization Grant

{{I-D.ietf-oauth-identity-assertion-authz-grant}} (the "ID-JAG"
specification, adopted by the OAuth Working Group as of April 2026)
targets deployments where AS-B already trusts AS-A (acting as an
IdP) for Single Sign-On (SSO) and subject resolution, using an
OpenID Connect ID Token or SAML 2.0 assertion as the subject token.

The key structural differences between the two profiles are:

Subject Token Type:
: The ID-JAG profile uses an OpenID Connect ID Token or SAML 2.0
  assertion as the `subject_token`.  This profile uses a Txn-Token
  (`urn:ietf:params:oauth:token-type:txn_token`).

Initiating Principal Scope:
: The ID-JAG profile is exclusively centered on a human End-User
  whose authenticated session at the IdP drives the cross-domain
  access.  This profile supports all three Initiating Principal
  types — human user, internal system, and automated workload —
  uniformly, because Txn-Tokens capture all three.

Trust Relationship Basis:
: The ID-JAG profile relies on a pre-existing SSO trust relationship
  between AS-A (the IdP) and AS-B (the Resource AS) for the same
  user population.  This profile relies on a bilateral Cross-Domain
  Trust Agreement between AS-A and AS-B, which may exist
  independently of any shared identity provider.

`audience` and `resource` Parameters:
: Both profiles use `audience` to identify AS-B (the target
  authorization server) and `resource` ({{RFC8707}}) optionally to
  identify the target Protected Resource.  These parameters serve
  the same distinct purposes in both profiles: `audience` → AS-B
  issuer URL → `aud` in the grant; `resource` → resource server URI
  → `resource` claim in the grant.

`client_id` Requirement:
: The ID-JAG includes a REQUIRED `client_id` claim identifying the
  OAuth 2.0 client at AS-B acting on behalf of the resource owner.
  This is appropriate where the application has a pre-registered
  client relationship with AS-B.  This profile does not require a
  pre-registered `client_id` at AS-B; the Requesting Workload's
  identity is conveyed through client authentication to AS-A and the
  subject mapping in the JWT Authorization Grant.

Multi-Tenancy:
: The ID-JAG profile defines `tenant`, `aud_tenant`, and `aud_sub`
  claims for multi-tenant SaaS deployments.  This profile does not
  define equivalent tenant-scoping claims, as trust domain
  boundaries are typically organizational or service-provider
  boundaries rather than tenant partitions within a shared platform.

Rich Authorization Requests (RAR):
: The ID-JAG profile supports the optional `authorization_details`
  claim ({{RFC9396}}) in the grant.  This profile does not currently
  define RAR integration; a future revision MAY define how
  `authorization_details` from a Txn-Token are transcribed into the
  JWT Authorization Grant.

SAML 2.0 Interoperability:
: The ID-JAG profile includes SAML 2.0 identity assertion
  interoperability.  This profile addresses only JWT-based
  Txn-Tokens.

Sender Constraining:
: Both profiles use the `cnf` claim to convey a sender-constraining
  key to AS-B.  The ID-JAG profile embeds `cnf` in the ID-JAG
  itself; this profile includes `cnf` in the JWT Authorization
  Grant, derived from the Requesting Workload's client credential
  presented to AS-A (see {{sender-constraining}}).

The two profiles are complementary.  A deployment MAY support both:
the ID-JAG profile for human-user cross-domain access coordinated
through a shared identity provider, and this profile for any
transaction-driven cross-domain access (user-initiated,
system-initiated, or workload-initiated) where the trust relationship
is established through a bilateral Cross-Domain Trust Agreement.  An
Authorization Server implementing both MUST distinguish between them
by inspecting the JWT `typ` header: `oauth-id-jag+jwt` for the
ID-JAG profile and `txn-chain+jwt` for this profile.


# Acknowledgements
{:numbered="false"}

The author would like to thank Atul Tulshibagwale, Pieter Kasselman,
Aaron Parecki, Brian Campbell, Arndt Schwenkschuster, Kelley Burgin,
Karl McGuinness, and the members of the IETF OAuth Working Group for
their foundational work on the specifications that this profile
depends on.

The Transaction Tokens concept was originally developed by Atul
Tulshibagwale, George Fletcher, and Pieter Kasselman.  The OAuth
Identity and Authorization Chaining Across Domains specification was
authored by Arndt Schwenkschuster, Pieter Kasselman, Kelley Burgin,
Michael Jenkins, Brian Campbell, and Aaron Parecki.
