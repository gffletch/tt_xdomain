---
title: "Transaction Token Authorization Grant Profile for OAuth Identity and Authorization Chaining"
abbrev: "Txn-Token Chaining Profile"
docname: draft-fletcher-oauth-transaction-token-chaining-profile-00
category: std
submissiontype: IETF
ipr: trust200902
workgroup: Web Authorization Protocol
keyword:
  - oauth
  - transaction tokens
  - identity chaining
  - cross-domain
  - workload identity

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
Authorization Grant for crossing a trust domain boundary.  By using a
Txn-Token as the input credential, this profile enables workloads
operating within a trusted domain to carry their full transaction
context — including the original initiating principal, authorization
scope, and request-specific parameters — into a cross-domain access
request, without leaking internal credentials beyond the trust
boundary.

--- note_Note_to_Readers

*RFC EDITOR: please remove this section before publication*

Discussion of this document takes place on the Web Authorization
Protocol Working Group mailing list (oauth@ietf.org), which is
archived at <https://mailarchive.ietf.org/arch/browse/oauth/>.

Source for this draft and an issue tracker can be found at
<https://github.com/george-fletcher/draft-fletcher-oauth-transaction-token-chaining-profile>.

--- middle

# Introduction

Modern multi-service architectures decompose business functionality
across many workloads that may span organizational and cloud-provider
trust boundaries.  When a workload operating inside a trust domain
needs to invoke a resource in a foreign trust domain, it must
present credentials that are meaningful to the foreign domain's
authorization server — credentials that are both trustworthy and
that faithfully convey the original initiating context.

The OAuth Identity and Authorization Chaining Across Domains
specification {{I-D.ietf-oauth-identity-chaining}} defines a
general mechanism by which a client in Trust Domain A can obtain a
JWT Authorization Grant from the Authorization Server of Trust Domain A
and present it to the Authorization Server of Trust Domain B to receive
an access token.  The base specification deliberately leaves the choice
of subject token type open, allowing profiles to constrain and
specialize the mechanism for specific deployment scenarios.

The Transaction Tokens specification
{{I-D.ietf-oauth-transaction-tokens}} defines a token format and
issuance protocol purpose-built for propagating user identity and
authorization context across workloads within a single trust domain.
A Transaction Token (Txn-Token) is a short-lived, cryptographically
signed JWT that captures the identity of the initiating principal, the
authorization scope, and immutable parameters of the original external
request.

This specification defines the additional details necessary to use a
Txn-Token as the `subject_token` in the Token Exchange request
described in Section 2.3 of {{I-D.ietf-oauth-identity-chaining}}.
Doing so creates a natural bridge: the same rich context that
Txn-Tokens propagate within a trusted domain is preserved and
transformed into a cross-domain JWT Authorization Grant that carries
this context across the domain boundary.

This profile is complementary to the Identity Assertion JWT
Authorization Grant profile
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, a companion OAuth
Working Group document that targets deployments where the Resource
Authorization Server already trusts a common IdP for SSO and subject
resolution, and where an OpenID Connect ID Token or SAML 2.0 assertion
representing an authenticated human session is the subject token.
This profile instead addresses the workload-to-workload scenario
described in the WIMSE architecture {{I-D.ietf-wimse-arch}}, where no
human end-user session is driving the cross-domain access and the
input credential is a Txn-Token representing an authorized machine
transaction.  A detailed structural comparison of the two profiles
appears in the informative appendix on Related Specifications.

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

Requesting Workload:
: A workload (as defined in {{I-D.ietf-oauth-transaction-tokens}})
  operating inside Trust Domain A that wishes to invoke a protected
  resource in Trust Domain B.  This workload acts as the client in the
  identity chaining flow.

Transaction Token Service (TTS):
: The service within Trust Domain A that mints Txn-Tokens.  The TTS
  is the logical equivalent of the Authorization Server of Trust Domain
  A for the purposes of producing the subject token that initiates the
  chaining flow.  In some deployments the TTS and the Authorization
  Server of Trust Domain A MAY be co-located.

Authorization Server of Trust Domain A (AS-A):
: The OAuth 2.0 Authorization Server within Trust Domain A that
  receives the Token Exchange request from the Requesting Workload,
  validates the presented Txn-Token, and issues the JWT Authorization
  Grant.

Authorization Server of Trust Domain B (AS-B):
: The OAuth 2.0 Authorization Server within Trust Domain B that
  receives the JWT Authorization Grant and issues an access token for
  the Protected Resource.

Protected Resource:
: The resource server in Trust Domain B that the Requesting Workload
  wishes to call.


# Overview

The flow defined in this profile extends the identity chaining flow of
{{I-D.ietf-oauth-identity-chaining}} by specifying the exact nature of
the subject token presented in Step 2 of that flow.  The complete
end-to-end sequence is:

~~~ ascii-art
+----------+ +----------+ +---------+ +---------+ +---------+
| External |  |Requesting|  |  TTS /  |  | AS-B  |  |Protected|
| Endpoint |  | Workload |  |  AS-A   |  |       |  |Resource |
|(Trust A) |  |(Trust A) |  |(Trust A)|  |(TrustB|  |(Trust B)|
+----+-----+  +----+-----+  +----+----+  +---+---+  +---+----+
     |              |             |           |           |
     | (1) Access   |             |           |           |
     |   Token /   |             |           |           |
     |  API Request|             |           |           |
     |------------>|             |           |           |
     |             |             |           |           |
     |             | (2) Request |           |           |
     |             |  Txn-Token  |           |           |
     |             | [TxnToken]  |           |           |
     |             |------------>|           |           |
     |             |             |           |           |
     |             | (3) Txn-    |           |           |
     |             |   Token     |           |           |
     |             |<- - - - - - |           |           |
     |             |             |           |           |
     |             | (4) Discover|           |           |
     |             |  AS-B       |           |           |
     |             |-------------------------------+     |
     |             |             |           |           |
     |             | (5) Token Exchange      |           |
     |             |  [RFC8693]  |           |           |
     |             | subject_token = Txn-Token           |
     |             |------------>|           |           |
     |             |             |           |           |
     |             | (6) JWT Authorization   |           |
     |             |   Grant     |           |           |
     |             |<- - - - - - |           |           |
     |             |             |           |           |
     |             | (7) JWT Bearer Grant    |           |
     |             |   [RFC7523] |           |           |
     |             |---------------------------->|       |
     |             |             |           |           |
     |             | (8) Access Token        |           |
     |             |<- - - - - - - - - - - - |           |
     |             |             |           |           |
     |             | (9) API Request         |           |
     |             |------------------------------------>|
     |             |             |           |           |
~~~
{: #fig-flow title="Transaction Token Chaining Flow"}

The steps are as follows:

1. An external request arrives at the Requesting Workload's External
   Endpoint, accompanied by an OAuth 2.0 access token or other
   authorization credential.

2. The Requesting Workload exchanges the incoming access token with its
   local Transaction Token Service (TTS) for a Txn-Token covering the
   current transaction context.  This step follows the Txn-Token
   issuance procedures defined in {{I-D.ietf-oauth-transaction-tokens}}.

3. The TTS issues a Txn-Token to the Requesting Workload.

4. The Requesting Workload discovers AS-B using the mechanisms defined
   in Section 2.2 of {{I-D.ietf-oauth-identity-chaining}}, such as the
   `authorization_servers` property in OAuth 2.0 Protected Resource
   Metadata {{RFC9728}}.  The Requesting Workload obtains AS-B's issuer
   URL for use as the `audience` parameter in the subsequent Token
   Exchange request.

5. The Requesting Workload presents the Txn-Token as the `subject_token`
   in an OAuth 2.0 Token Exchange {{RFC8693}} request to AS-A.  The
   request includes AS-B's issuer URL as the `audience` parameter and,
   optionally, the target resource server URI as the `resource` parameter.
   See {{token-exchange-request-parameters}} for the full parameter
   specification.

6. AS-A validates the Txn-Token, evaluates its authorization policy,
   transcribes claims as appropriate (see {{claims-transcription}}),
   and returns a JWT Authorization Grant.

7. The Requesting Workload presents the JWT Authorization Grant to AS-B
   using the JWT Profile for OAuth 2.0 Authorization Grants {{RFC7523}}.

8. AS-B validates the JWT Authorization Grant and issues an access token
   for the Protected Resource.

9. The Requesting Workload calls the Protected Resource using the access
   token.


# Transaction Token as Subject Token

## Subject Token Requirements

When this profile is used, the `subject_token` in the Token Exchange
request (Step 5 of {{fig-flow}}) MUST be a Txn-Token as defined in
{{I-D.ietf-oauth-transaction-tokens}}.

The `subject_token_type` parameter MUST be:

~~~ abnf
subject_token_type =
    "urn:ietf:params:oauth:token-type:txn-token"
~~~

This value is registered with IANA in {{iana-oauth-uri}} of this
document.

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

## Token Exchange Request Parameters {#token-exchange-request-parameters}

In addition to the subject token requirements in
{{subject-token-requirements}}, the Token Exchange request
({{RFC8693}} Section 2.1) MUST include the following parameters when
this profile is in use.

### Identifying the Target Authorization Server

The Token Exchange request MUST indicate which authorization server
in Trust Domain B (AS-B) should be the intended audience of the
resulting JWT Authorization Grant.  This profile uses the `audience`
parameter for this purpose, following the same convention as
{{I-D.ietf-oauth-identity-assertion-authz-grant}} Section 4.3.

`audience`:
: REQUIRED.  The Issuer URL of AS-B as defined in Section 2 of
  {{RFC8414}}.  AS-A uses this value to set the `aud` claim of the
  resulting JWT Authorization Grant, so that AS-B can verify the
  grant was issued for it.  The value MUST be the `issuer`
  identifier as advertised in AS-B's authorization server metadata
  ({{RFC8414}}), or a logical name for AS-B that AS-A has been
  configured to recognize and map to AS-B's actual issuer identifier.

`resource`:
: OPTIONAL.  A URI that identifies the specific Protected Resource
  (resource server) in Trust Domain B that the Requesting Workload
  intends to call, as defined in Section 2 of {{RFC8707}}.  When
  present, AS-A SHOULD include this value in the `resource` claim of
  the JWT Authorization Grant so that AS-B can issue a resource-bound
  access token.

The two parameters serve distinct purposes that MUST NOT be confused:

- `audience` identifies the **authorization server** (AS-B) that
  will receive and validate the JWT Authorization Grant.  It
  determines the `aud` claim of the grant.

- `resource` identifies the **resource server** (the protected API)
  for which an access token is ultimately sought.  It is used by
  AS-B when issuing the access token, not for validating the grant
  itself.

This distinction aligns with {{I-D.ietf-oauth-identity-chaining}}
Section 2.3.1, which requires either `resource` or `audience` to be
present and specifies that these values identify the target
authorization server.  However, in this profile `resource` in the
Token Exchange request identifies the resource server (following
{{RFC8707}}), while `audience` identifies AS-B.  Implementations
MUST use `audience` to identify AS-B and MAY additionally use
`resource` to identify the target resource server, but MUST NOT pass
the AS-B issuer URL as the `resource` parameter.

### Remaining Parameters

`grant_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:
: REQUIRED.  The Txn-Token as described in
  {{subject-token-requirements}}.

`subject_token_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:token-type:txn-token`.

`requested_token_type`:
: OPTIONAL.  When present, the value MUST be
  `urn:ietf:params:oauth:token-type:jwt`, indicating that a JWT
  Authorization Grant is requested.  If absent, AS-A MUST still
  produce a JWT Authorization Grant conforming to this profile when
  the other parameters conform to this profile.

`scope`:
: OPTIONAL.  The space-separated list of scopes requested for
  the resulting JWT Authorization Grant, which constrains the
  access token that AS-B will ultimately issue.  AS-A MUST NOT
  issue a JWT Authorization Grant with a scope that exceeds the
  scope asserted in the `scope` claim of the presented Txn-Token.
  If the requested scope is wider than the Txn-Token's scope,
  AS-A MUST either narrow the granted scope to the intersection
  or deny the request.

The `actor_token` and `actor_token_type` parameters defined in
{{RFC8693}} are not used in this profile.

### Example Token Exchange Request

The following is a non-normative example of a Token Exchange request
conforming to this profile.  The Requesting Workload (a service in
Trust Domain A) requests a JWT Authorization Grant for AS-B
(`https://as.b.example`) in order to subsequently call the
inventory API (`https://api.b.example/inventory`).

~~~ http-message
POST /token HTTP/1.1
Host: as.a.example
Content-Type: application/x-www-form-urlencoded
Authorization: Bearer <workload-client-credential>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<txn-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atxn-token
&audience=https%3A%2F%2Fas.b.example
&resource=https%3A%2F%2Fapi.b.example%2Finventory
&scope=inventory.read
~~~

### Token Exchange Response

If the request is valid and the Requesting Workload is authorized to
receive a JWT Authorization Grant for the indicated audience, AS-A
returns a Token Exchange response as defined in Section 2.2 of
{{RFC8693}}.

`access_token`:
: REQUIRED.  The JWT Authorization Grant.  (Token Exchange uses the
  `access_token` field for the returned token for historical
  compatibility reasons, even though this is not an OAuth access
  token.)

`issued_token_type`:
: REQUIRED.  The value MUST be
  `urn:ietf:params:oauth:token-type:jwt`.

`token_type`:
: REQUIRED.  The value MUST be `N_A`, as this is not an OAuth
  access token and sender-constraining via `token_type` does not
  apply.

`expires_in`:
: RECOMMENDED.  The lifetime of the JWT Authorization Grant in
  seconds.  This value SHOULD reflect the `exp` claim of the
  returned grant JWT and SHOULD be short (see {{jwt-authorization-grant}}).

`refresh_token`:
: This parameter SHOULD NOT be present.

On error, AS-A returns an OAuth 2.0 token error response as defined
in Section 5.2 of {{RFC6749}} and Section 2.2.2 of {{RFC8693}}.

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
   specified in Section 5.1 of {{I-D.ietf-oauth-identity-chaining}} and
   Section 2.5 of {{RFC9700}}.

2. Validate the Txn-Token signature using the public keys of the TTS
   that issued it.  AS-A MUST establish a trust relationship with the
   TTS; in deployments where the TTS and AS-A are co-located, this is
   straightforward.  Where they are separate services, AS-A MUST be
   configured with the TTS's `jwks_uri` or equivalent key material.

3. Validate that the Txn-Token is not expired.

4. Validate that the `aud` claim of the Txn-Token is acceptable to
   AS-A (i.e., the Txn-Token was intended for consumption by AS-A or
   the TTS acting on AS-A's behalf).

5. Resolve the target AS-B from the `audience` parameter.  AS-A MUST
   verify that the `audience` value corresponds to a known AS-B with
   which a trust relationship has been established.  If the `audience`
   is unknown or the policy prohibits issuing a JWT Authorization Grant
   for that audience, AS-A MUST deny the request as specified in
   Section 2.2.2 of {{RFC8693}}.  AS-A MUST NOT accept a bare resource
   server URI in the `audience` parameter in place of an AS-B issuer
   identifier.

6. If the `resource` parameter is present, validate that it identifies
   a resource server within Trust Domain B that is consistent with the
   indicated `audience` (AS-B).  AS-A SHOULD propagate the `resource`
   value into the `resource` claim of the JWT Authorization Grant so
   that AS-B can issue a resource-bound access token (see
   {{RFC8707}}).

7. Validate that the requested `scope` (if present) does not exceed
   the scope of the presented Txn-Token.  AS-A MUST NOT issue a JWT
   Authorization Grant with a broader scope than the Txn-Token's scope.

8. Apply claims transcription policy as described in
   {{claims-transcription}}.

9. Construct and sign the JWT Authorization Grant as described in
   {{jwt-authorization-grant}}, setting the `aud` claim to the AS-B
   issuer identifier resolved in step 5.

10. Return the JWT Authorization Grant in the Token Exchange response
    as described in {{token-exchange-request-parameters}}.

## AS-B Processing Rules

Upon receipt of a JWT Bearer grant request ({{RFC7523}}) that
originated from this profile, AS-B MUST perform the following steps
in addition to the processing rules specified in Section 2.4.2 of
{{I-D.ietf-oauth-identity-chaining}}:

1. Validate the `typ` header of the JWT Authorization Grant.  The value
   MUST be `txn-chain+jwt` as defined in {{jwt-authorization-grant}}.

2. Validate the `aud` claim identifies AS-B.

3. Validate the `sub` claim or mapped workload identity claim against
   known or policy-authorized subjects within Trust Domain B.

4. If present, evaluate the `txn_claims` claim (see
   {{claims-transcription}}) when applying authorization policy, such
   as verifying that the original request context is consistent with
   the access being requested.

5. Issue an access token constrained by the scope asserted in the JWT
   Authorization Grant.  AS-B SHOULD NOT issue refresh tokens as
   described in Section 5.4 of {{I-D.ietf-oauth-identity-chaining}}.


# JWT Authorization Grant {#jwt-authorization-grant}

## Grant Format

The JWT Authorization Grant produced by AS-A in response to a Token
Exchange request conforming to this profile is a JWT {{RFC7519}}
that MUST conform to the JWT Authorization Grant requirements
specified in Section 2.3.3 of
{{I-D.ietf-oauth-identity-chaining}}.

### JWT Header

The JWT header MUST include the following:

typ:
: REQUIRED.  The value MUST be `txn-chain+jwt`.  Using an explicit
  type is REQUIRED by this profile, consistent with the JSON Web
  Token Best Current Practices {{RFC8725}}.

alg:
: REQUIRED.  An asymmetric signing algorithm appropriate for the
  security requirements of the deployment.  Deployments SHOULD
  use PS256, PS384, PS512, ES256, ES384, or ES512 as defined in
  {{RFC7519}}.  The use of `none` or symmetric algorithms is
  prohibited.

kid:
: RECOMMENDED.  The key identifier corresponding to the signing key.

### JWT Body Claims

The following claims MUST be present in the JWT Authorization Grant:

iss:
: REQUIRED.  The Issuer identifier of AS-A ({{RFC8414}} Section 2).

sub:
: REQUIRED.  The subject of the JWT Authorization Grant.  This
  MUST be derived from the `sub` claim of the Txn-Token.  AS-A MAY
  transcribe the identifier according to the mapping defined in
  {{claims-transcription}}.

aud:
: REQUIRED.  The Issuer URL of AS-B ({{RFC8414}} Section 2), derived
  from the `audience` parameter of the Token Exchange request.  As
  specified in Section 2.3.3 of {{I-D.ietf-oauth-identity-chaining}},
  the `aud` claim SHOULD be restricted to a single authorization server
  to prevent the grant from being replayed at an unintended AS.

iat:
: REQUIRED.  The time at which the grant was issued ({{RFC7519}}
  Section 4.1.6).

exp:
: REQUIRED.  The expiration time of the grant ({{RFC7519}} Section
  4.1.4).  The lifetime SHOULD be short — deployments SHOULD use a
  value no greater than 300 seconds (5 minutes) and SHOULD prefer
  values of 60 seconds or less.

jti:
: REQUIRED.  A unique identifier for this JWT ({{RFC7519}} Section
  4.1.7).  AS-B SHOULD enforce single-use semantics by tracking
  presented `jti` values within the grant's validity window.

scope:
: RECOMMENDED.  The space-separated list of scopes authorized for
  this grant ({{RFC6749}} Section 3.3).  The scope MUST NOT be wider
  than the scope in the `scope` claim of the source Txn-Token.

The following claims SHOULD be present:

txn:
: The unique transaction identifier of the originating Txn-Token, as
  defined in {{I-D.ietf-oauth-transaction-tokens}}.  Including this
  claim preserves the transaction correlation chain across the domain
  boundary and supports auditing.

The following claims MAY be present:

resource:
: A URI or array of URIs identifying the Protected Resource(s) in
  Trust Domain B for which access is being requested, as defined in
  Section 2 of {{RFC8707}}.  When present, this claim MUST be derived
  from the `resource` parameter of the Token Exchange request.  AS-B
  SHOULD use this value to issue a resource-bound access token.  Note
  that this claim identifies the **resource server** (the API), not AS-B;
  it is distinct from the `aud` claim, which identifies AS-B.

txn_claims:
: A JSON object containing a subset of claims transcribed from the
  Txn-Token.  The exact set of claims to include is determined by
  the AS-A's claims transcription policy (see
  {{claims-transcription}}).  Claims that reveal internal
  infrastructure details or that are not meaningful to AS-B
  SHOULD be omitted.

cnf:
: If sender-constraining is in use (see {{sender-constraining}}),
  the confirmation method claim as defined in RFC 7800.

### Example JWT Authorization Grant

The following is a non-normative example of the header and claims of a
JWT Authorization Grant produced by this profile, corresponding to the
Token Exchange request example in
{{token-exchange-request-parameters}}.  The token is presented in
decoded form.

Header:

~~~ json
{
  "typ": "txn-chain+jwt",
  "alg": "ES256",
  "kid": "as-a-2026-01"
}
~~~

Claims:

~~~ json
{
  "iss": "https://as.a.example",
  "sub": "workload-svc-1@a.example",
  "aud": "https://as.b.example",
  "iat": 1746700000,
  "exp": 1746700060,
  "jti": "8f14e45f-ceee-467a-a19e-ab8f290a1f30",
  "scope": "inventory.read",
  "resource": "https://api.b.example/inventory",
  "txn": "a9b2c3d4-e5f6-7890-abcd-ef1234567890",
  "txn_claims": {
    "purp": "inventory-check",
    "rctx": {
      "ip": "203.0.113.42"
    }
  }
}
~~~

Note that `aud` is set to `https://as.b.example` (the AS-B issuer
identifier, derived from the `audience` request parameter) and
`resource` is set to `https://api.b.example/inventory` (the resource
server URI, derived from the `resource` request parameter).  These
are distinct values serving distinct purposes.


# Claims Transcription {#claims-transcription}

This profile constrains and extends the claims transcription rules of
Section 2.5 of {{I-D.ietf-oauth-identity-chaining}} as follows.

## Mandatory Transcriptions

AS-A MUST derive the `sub` claim of the JWT Authorization Grant from
the `sub` claim of the Txn-Token.  If the subject identifier format
differs between Trust Domain A and Trust Domain B, AS-A MUST
translate the identifier accordingly.  If no mapping for the subject
can be determined, AS-A MUST reject the request.

AS-A MUST transcribe the `txn` claim (the unique transaction
identifier) from the Txn-Token into the JWT Authorization Grant.

## Constrained Scope Transcription

AS-A MUST NOT expand the scope beyond the `scope` claim of the
source Txn-Token.  AS-A MAY further restrict the scope in response
to policy or the `scope` parameter in the Token Exchange request.
The effective scope in the JWT Authorization Grant MUST be the
intersection of the requested scope and the Txn-Token's scope.

## Optional Context Claims Transcription

AS-A MAY include a `txn_claims` claim in the JWT Authorization Grant,
containing a subset of claims extracted from the Txn-Token body.
The purpose of this claim is to allow AS-B to make more informed
authorization decisions based on the original transaction context.

The following guidance applies to transcribing Txn-Token claims into
`txn_claims`:

- Claims that identify internal infrastructure, intermediate
  workloads, or internal IP addresses SHOULD be omitted unless
  Trust Domain B has an explicit need for them.

- The `purp` claim (purpose/intent of the transaction) SHOULD be
  included when it is meaningful to AS-B's authorization policy.

- The `rctx` claim (requester context) MAY be included in a
  redacted form to preserve the IP address of the original external
  endpoint while stripping internal network information.

- AS-A SHOULD NOT include claims that are specific to the internal
  workload orchestration of Trust Domain A and that have no
  normative meaning in Trust Domain B.

When AS-B receives a JWT Authorization Grant that includes
`txn_claims`, both AS-A (at grant issuance time) and AS-B (at token
issuance time) MUST agree on the semantics of the included claims.
Profiles or deployment agreements SHOULD define the set of claims
that may appear in `txn_claims` and their interpretation.

## Subject Identifier Formats

When SPIFFE Workload Identifiers {{I-D.ietf-wimse-arch}} or WIMSE
Workload Credentials {{I-D.ietf-wimse-workload-creds}} are in use in
Trust Domain A, the `sub` claim of the Txn-Token may be a SPIFFE URI
(e.g., `spiffe://trust-domain-a.example/ns/default/sa/svc1`).  AS-A
MUST map this identifier to a form that AS-B can recognize.  Possible
mappings include:

- A local identifier at AS-B if a prior federation agreement has
  established an explicit mapping.

- A transformed identifier that preserves the structural SPIFFE URI
  form (e.g., `spiffe://trust-domain-a.example/svc1`) and relies on
  AS-B's policy to reason about cross-domain SPIFFE URIs.


# Authorization Server Metadata

This profile registers the following addition to the Authorization
Server Metadata defined in {{RFC8414}} and Section 3 of
{{I-D.ietf-oauth-identity-chaining}}.

In addition to the `identity_chaining_requested_token_types_supported`
metadata parameter, an Authorization Server that supports this profile
SHOULD include the following token type in the list:

~~~
urn:ietf:params:oauth:token-type:txn-token
~~~

Inclusion of this value indicates that the Authorization Server will
accept Txn-Tokens as the `subject_token` in a Token Exchange request
conforming to this profile.


# Security Considerations

## Client Authentication

The Requesting Workload MUST authenticate to AS-A when performing the
Token Exchange request.  AS-A SHOULD follow the guidance of Section
2.5 of {{RFC9700}} for client authentication.  The use of
asymmetric key-based client authentication (e.g., {{RFC7523}} client
assertion) is RECOMMENDED.  The use of static shared secrets for
client authentication is NOT RECOMMENDED.

## Sender Constraining Tokens {#sender-constraining}

AS-B SHOULD issue sender-constrained access tokens.  Both DPoP
(OAuth 2.0 Demonstrating Proof of Possession) and Mutual-TLS
({{RFC9700}} Section 2.3) are RECOMMENDED mechanisms.

When the Authorization Server in Trust Domain A acts as client to
AS-B (as in Appendix B.2 of {{I-D.ietf-oauth-identity-chaining}}),
the delegated key binding mechanism described in Appendix B.3 of
{{I-D.ietf-oauth-identity-chaining}} SHOULD be used.  In this
case, AS-A MUST verify proof of possession of the Requesting
Workload's public key and convey the key to AS-B using the
`requested_cnf` claim in the JWT Authorization Grant, so that AS-B
can bind the resulting access token to that key.

## Txn-Token as Bearer Credential

A Txn-Token presented as a subject token is a bearer credential.
If an attacker captures a valid Txn-Token before the Token Exchange
request is made, it can potentially be used to obtain a JWT
Authorization Grant.  Deployments MUST evaluate this risk.
Mitigations include:

- Ensuring that all communication between the Requesting Workload
  and AS-A occurs over mutually authenticated TLS.

- Configuring very short Txn-Token lifetimes (on the order of
  seconds to minutes).

- Requiring that the Requesting Workload authenticate itself to AS-A
  with client credentials that cannot be stolen along with the
  Txn-Token.

## JWT Authorization Grant Replay

The JWT Authorization Grant produced by this profile is also a
bearer token.  AS-A SHOULD restrict its validity to a short
lifetime.  AS-B SHOULD enforce single-use semantics for the `jti`
claim to prevent replay.  See Section 5.5 of
{{I-D.ietf-oauth-identity-chaining}} for additional guidance.

## Txn-Token Cross-Domain Leakage

Txn-Tokens are designed for use within a single trust domain and MAY
contain information about internal workload call chains,
infrastructure topology, or sensitive operational parameters that
MUST NOT be exposed beyond the trust boundary.  AS-A MUST NOT
forward the raw Txn-Token to AS-B.  The Txn-Token is consumed by
AS-A, and only the carefully transcribed JWT Authorization Grant
(with appropriate claims minimization as defined in
{{claims-transcription}}) crosses the trust domain boundary.

## Scope Boundary Enforcement

AS-A MUST enforce that the scope in the JWT Authorization Grant does
not exceed the scope of the presented Txn-Token.  This prevents a
workload from using identity chaining to escalate its privileges
beyond what was authorized in the originating transaction.  AS-B MUST
independently verify that the scope requested in the access token
request does not exceed the scope of the presented JWT Authorization
Grant.

## Trust Domain Isolation

The trust relationship between AS-A and AS-B confers only the ability
for AS-A to issue JWT Authorization Grants that AS-B will accept.
AS-B MUST NOT allow clients in Trust Domain B to present JWT
Authorization Grants issued by AS-A on behalf of clients in Trust
Domain A as a mechanism to elevate their own privileges within Trust
Domain B.

## Refresh Tokens

AS-B SHOULD NOT issue refresh tokens in response to a JWT
Authorization Grant request conforming to this profile, consistent
with Section 5.4 of {{I-D.ietf-oauth-identity-chaining}}.  Because
Txn-Tokens are short-lived and transaction-specific, re-obtaining a
new Txn-Token and repeating the chaining flow is the correct
mechanism for renewing authorization.


# Privacy Considerations

Txn-Tokens may contain claims derived from the original external
request, including information about the initiating user, their
device, or the nature of the request.  This specification requires
AS-A to apply claims minimization (see {{claims-transcription}})
before issuing a JWT Authorization Grant.

Deployments MUST ensure that any user-identifying information
included in `txn_claims` is consistent with the privacy policies of
both Trust Domain A and Trust Domain B.  Trust relationships between
authorization servers MUST be established with user privacy in mind,
including alignment of data handling policies.

The propagation of the `txn` claim (transaction correlation
identifier) across domain boundaries enables end-to-end transaction
tracing.  This may assist auditing and debugging but also constitutes
a form of cross-domain correlation.  Deployments SHOULD evaluate
whether the auditability benefits of propagating the `txn` claim
outweigh the privacy implications in their specific context.


# IANA Considerations

## OAuth URI Registry {#iana-oauth-uri}

This specification requests registration of the following value in
the "OAuth URI" registry established by {{RFC6749}} (maintained by
IANA at <https://www.iana.org/assignments/oauth-parameters>):

- URI: `urn:ietf:params:oauth:token-type:txn-token`
- Common Name: Transaction Token
- Change Controller: IETF
- Specification Document(s): {{subject-token-requirements}} of
  this document

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

# Use Cases

## Workload-to-Workload Cross-Domain API Access

A service mesh within Trust Domain A processes an API request
initiated by an external user.  The gateway workload obtains a
Txn-Token capturing the user identity, the request scope, and the
original client IP address.  A downstream workload within the mesh
determines that it must call a partner API in Trust Domain B.  Using
this profile, the downstream workload exchanges its Txn-Token for a
JWT Authorization Grant targeting Trust Domain B's authorization
server, then presents the grant to obtain an access token for the
partner API.  The partner's resource server receives an access token
whose issuer is Trust Domain B's authorization server, containing a
mapped subject identifier and a minimal set of contextual claims.
The raw internal Txn-Token is never exposed outside Trust Domain A.

## CI/CD Pipeline Accessing External Artifact Registry

A CI/CD orchestrator in Trust Domain A manages build pipelines.
Each pipeline run is represented as a transaction within Trust Domain
A, identified by a Txn-Token that captures the pipeline identifier,
repository, and commit hash in its `txn_claims`.  When a pipeline
run needs to publish an artifact to an external registry in Trust
Domain B, the pipeline workload exchanges its Txn-Token for a JWT
Authorization Grant.  The grant includes a transcribed `txn_claims`
object containing the repository and commit hash.  The external
registry's authorization server issues a scoped upload token that
Trust Domain B's audit logs can correlate with the originating build
pipeline.

## Agentic AI Workflow Calling an External Tool

An AI orchestration platform in Trust Domain A manages AI agent
tasks on behalf of a user.  The platform issues a Txn-Token for each
agent invocation, capturing the user identity and the declared intent
(`purp`) of the agent task (e.g., "generate-billing-report").  When
the agent must call a third-party data API in Trust Domain B, the
platform uses this profile to exchange the Txn-Token for a JWT
Authorization Grant that conveys the user identity and the task
intent to Trust Domain B.  This enables the external data service to
apply intent-aware authorization policies without receiving the full
internal Txn-Token.


# Relationship to Related Specifications

This specification is one of a family of profiles of
{{I-D.ietf-oauth-identity-chaining}}.

## Identity Assertion JWT Authorization Grant

{{I-D.ietf-oauth-identity-assertion-authz-grant}} (the "ID-JAG"
specification, adopted by the OAuth Working Group as of April 2026)
targets deployments where the Resource Authorization Server (AS-B)
already trusts the issuing IdP (AS-A) for Single Sign-On (SSO) and
subject resolution.  The subject token in that profile is a human
user's Identity Assertion — typically an OpenID Connect ID Token or
a SAML 2.0 assertion.

The key structural properties of the ID-JAG profile that distinguish
it from this specification are:

Subject Token Type:
: The ID-JAG profile uses an OpenID Connect ID Token
  (`urn:ietf:params:oauth:token-type:id_token`) or SAML 2.0
  assertion (`urn:ietf:params:oauth:token-type:saml2`) as the
  `subject_token`.  This profile uses a Txn-Token
  (`urn:ietf:params:oauth:token-type:txn-token`).

Principal Model:
: The ID-JAG profile is centered on a human End-User whose
  authenticated session with the IdP drives the cross-domain
  access request.  This profile is centered on a software
  workload acting on the basis of an authorized transaction,
  which may or may not involve a human initiating principal.

`audience` and `resource` Parameters:
: Both profiles use `audience` in the Token Exchange request to
  identify the target authorization server (AS-B), following the
  convention in {{I-D.ietf-oauth-identity-assertion-authz-grant}}
  Section 4.3.  The `resource` parameter ({{RFC8707}}) in the Token
  Exchange request identifies the target resource server (the API),
  not AS-B.  This profile adopts the same parameter semantics.  Both
  profiles treat `audience` and `resource` as serving distinct
  purposes: `audience` → AS-B issuer URL → becomes `aud` in the grant;
  `resource` → resource server URI → becomes `resource` claim in the
  grant and subsequently constrains the access token issued by AS-B.

`client_id` Requirement:
: The ID-JAG includes a REQUIRED `client_id` claim identifying
  the OAuth 2.0 client at the Resource Authorization Server that
  will act on behalf of the resource owner.  This allows the
  Resource Authorization Server to enforce its own client
  registration and policy independently of the IdP.  This profile
  uses the Requesting Workload's authenticated identity (conveyed
  via client authentication to AS-A) rather than a pre-registered
  `client_id` at AS-B, which is consistent with the workload
  credential model of {{I-D.ietf-wimse-arch}}.

Multi-Tenancy:
: The ID-JAG profile defines `tenant`, `aud_tenant`, and `aud_sub`
  claims to handle multi-tenant SaaS deployments where a shared IdP
  and shared Resource Authorization Server each host multiple
  organizational tenants.  This profile does not define equivalent
  claims, as workload trust domains are typically bounded by
  organizational or cloud-provider boundaries rather than tenant
  identifiers within a shared service.

Rich Authorization Requests (RAR):
: The ID-JAG profile supports the optional `authorization_details`
  claim ({{?RFC9396}}) in the grant, enabling structured
  authorization beyond simple scope strings.  This profile does not
  currently define RAR integration; a future revision or companion
  document MAY define how RAR `authorization_details` from the
  originating Txn-Token are transcribed into the JWT Authorization
  Grant.

SAML 2.0 Interoperability:
: The ID-JAG profile includes a dedicated section on SAML 2.0
  Identity Assertion interoperability, enabling deployments that use
  SAML as the SSO protocol to participate in cross-domain access.
  This profile does not address SAML, as Txn-Tokens are JWT-based
  and operate within the OAuth/WIMSE ecosystem.

Sender Constraining:
: Both profiles discuss sender-constraining of access tokens.  The
  ID-JAG profile defines a Proof-of-Possession mechanism using a
  `cnf` claim in the ID-JAG itself to convey the client's public
  key to the Resource Authorization Server, enabling the Resource
  Authorization Server to issue a bound access token.  This profile
  adopts the delegated key binding mechanism described in Appendix
  B.3 of {{I-D.ietf-oauth-identity-chaining}} for the
  authorization-server-as-client topology and uses `requested_cnf`
  in the JWT Authorization Grant.

The two profiles are intended to be complementary.  A deployment
MAY support both simultaneously: using the ID-JAG profile for
user-initiated cross-domain access (where an authenticated human
session is the driver) and this profile for machine-initiated
cross-domain access (where a Txn-Token representing an authorized
workload transaction is the driver).  An authorization server
implementing both profiles SHOULD distinguish between the two grant
types by inspecting the `typ` header of the JWT Authorization Grant:
`oauth-id-jag+jwt` for the ID-JAG profile and `txn-chain+jwt` for
this profile.


# Acknowledgements
{:numbered="false"}

The author would like to thank Atul Tulshibagwale, Pieter Kasselman,
Aaron Parecki, Brian Campbell, Arndt Schwenkschuster, Kelley Burgin,
and the members of the IETF OAuth Working Group for their foundational
work on the specifications that this profile depends on.

The Transaction Tokens concept was originally developed by Atul
Tulshibagwale, George Fletcher, and Pieter Kasselman.  The OAuth
Identity and Authorization Chaining Across Domains specification was
authored by Arndt Schwenkschuster, Pieter Kasselman, Kelley Burgin,
Michael Jenkins, Brian Campbell, and Aaron Parecki.
