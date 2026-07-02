---
title: "OAuth 2.0 Refresh Token and Authorization Expiration"
abbrev: "OAuth RT/Authorization Expiration"
category: info

docname: draft-ietf-oauth-refresh-token-expiration-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - refresh token
 - authorization
 - token endpoint
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "oauth-wg/rt-expiration"
  latest: "https://drafts.oauth.net/rt-expiration/draft-ietf-oauth-refresh-token-expiration.html"

author:
 -
    fullname: Nicholas Watson
    organization: Google, LLC
    email: nwatson@google.com

normative:
  RFC6749:
  RFC7662:
  RFC8414:
  RFC9700:

informative:
  RFC9396:
  OpenID:
    title: OpenID Connect Core 1.0
    target: https://openid.net/specs/openid-connect-core-1_0.html
    date: November 8, 2014
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore

--- abstract

This specification extends OAuth 2.0 [RFC6749] by adding new token endpoint
response parameters to specify refresh token expiration and user authorization
expiration.

--- middle

# Introduction

RFC6749 defines the OAuth 2.0 protocol, part of which is the ability for a
client to receive a refresh token that may be repeatedly exchanged for more
access tokens. OAuth 2.0 does not contain any normative language around
expiration or lack thereof for refresh tokens, mentioning only that they are
"typically long-lasting".

In the years since the publication of OAuth 2.0, in response to changing
security and privacy landscapes, many authorization servers have begun to issue
shorter-lived refresh tokens for two main reasons:

*   The authorization server or user may decide that the access being granted is
    too sensitive to allow indefinite access (e.g. mail or health data).
*   The authorization server enforces a maximum duration that refresh tokens may
    be held without being exchanged on the token endpoint.

Clients may wish to implement special handling for expiring refresh tokens. For
example, if the user has granted expiring access, the client may notify the user
that they will need to reauthorize access before a certain date to avoid
interruption of service.

# Requirements Notation and Conventions

{::boilerplate bcp14-tagged}

# Terminology

This specification uses terminology defined in [RFC6749]. The following terms
are used throughout this document:

Resource owner and user
:   May be used interchangeably to refer to the entity capable of granting
    access to a protected resource.

Client, application, and relying party
:   May be used interchangeably to refer to the application making protected
    resource requests on behalf of the resource owner and with its
    authorization.

Authorization
:   The resource owner's permission grant for a client to access protected
    resources on their behalf, as described in [RFC6749] Sec 4.1.1.

Access token
:   A credential used by the client to access protected resources on behalf of
    the resource owner, as referenced in [RFC6749] Sec 1.4. Access tokens
    represent proof of authorization.

Refresh token
:   A credential used by the client to obtain new access tokens without
    prompting the user, as referenced in [RFC6749] Sec 1.5. Refresh tokens do
    not grant authorization or renew authorization, they only provide a
    mechanism for obtaining new access tokens within the bounds of an existing
    authorization.

# Concepts

There are two mechanisms that can affect refresh token expiration.

## Authorization expiration

When granting authorization for an application to access their data as
referenced in [RFC6749] Sec 4.1.1, the user may opt to time-limit that
authorization, especially if the data is sensitive or they aren't sure how long
they'll continue using the application. The authorization server itself may also
impose mandatory limits on authorization duration.

## Refresh token timeout

Authorization servers may wish to define a maximum amount of time clients can
hold a refresh token without exchanging it. Beyond the security benefit provided
by expiring credentials, this also provides a convenient mechanism for
authorization servers to ensure there aren't ancient valid credentials out in
the wild, which could complicate tasks like refresh token key rotation.

# Refresh token expiration

The refresh token MUST NOT expire later than the user authorization expires. It
MAY expire earlier if the authorization server also enforces a maximum duration
between refresh token exchanges.

If the user renews their authorization, the authorization server SHOULD update
the expiration time of existing refresh tokens if their lifetime was truncated
due to user authorization expiration. (This is especially true if the
authorization was updated out of band as discussed in
[User Experience Considerations](#ux-considerations).) The
authorization server MUST NOT accept expired refresh tokens for any purpose,
even if it has no way to update the expiration time of existing refresh tokens.

Access tokens MUST NOT expire later than the user authorization expires. If the
user renews their authorization, the authorization server MAY update the
expiration time of existing access tokens if possible. Resource servers MUST NOT
accept expired access tokens for any purpose, even if the authorization server
has no way to update the expiration time of existing access tokens.

# Token endpoint response {#token-endpoint}

This specification introduces two new response parameters.

## Successful response

    refresh_token_timeout
          The time in seconds that the refresh token may be held by the client
          without exchanging. For example, the value 604800 denotes that the
          refresh token will expire in one week from the time the response was
          generated. This value SHALL NOT exceed the value in
          authorization_expires_in.

    authorization_expires_in
          The lifetime in seconds of the user's authorization for the scopes
          contained in the issued or presented refresh token. For example, the
          value 2629800 denotes that the authorization will expire in one month
          from the time the response was generated. This value MAY exceed that
          of refresh_token_timeout.

An authorization server MAY return only one of these parameters. Providing
`refresh_token_timeout` without `authorization_expires_in` indicates that the
user's authorization is indefinite, but the refresh token must be used within
the specified timeout to remain valid. Providing `authorization_expires_in`
without `refresh_token_timeout` indicates that the authorization has a fixed
duration, but the refresh token has no maximum idle time and may remain valid if
the authorization is extended out of band.

If finite, the authorization server MUST return these values whenever the token
endpoint response contains the `refresh_token` field. The authorization server
MAY return these values even if the response contains no `refresh_token` field,
in which case the values correspond to the presented `refresh_token`. This can
be useful in the following example cases:

*   For `refresh_token_timeout`, the authorization server could have
    updated the existing refresh token lifetime in place.
*   For `authorization_expires_in`, the user's authorization lifetime could have
    been modified out of band.
*   In all cases, it can be convenient for the client to receive these values
    in each response.

### Relationship of `authorization_expires_in` to scopes

Though `authorization_expires_in` is returned from the token endpoint when
refresh tokens are used, it corresponds to the user's authorization for *scopes*
(or finer-grained access through RAR [RFC9396]) rather than individual tokens.
The authorization server SHOULD ensure consistent lifetimes across multiple
refresh tokens for the same scopes.

Tying authorization lifetime to scopes means it's possible to have some access
valid for one duration and other access valid for a different duration. For
example, a user could grant indefinite access for the `openid` scope and
short-lived access for a calendar scope. In situations like this, it is
RECOMMENDED that the authorization server return the minimum time that any
access granted by the refresh token is valid. This does run some risk of the
client asking the user to reauthorize prematurely. In the previous example, the
client might ask the user to reauthorize the `openid` scope because it received
an `authorization_expires_in` value corresponding to the short-lived calendar
scope.

If clients are requesting multiple scopes that can have different lifetimes,
they will ultimately need to make their own tradeoffs to decide how and when to
ask the user for reauthorization. This specification's goal is simply to provide
them with more information to make this decision.

### Indefinite Expiration

Omitted values indicate that there is no fixed upper bound on the lifetime of
the credential or authorization. If the authorization server has not declared
its support for refresh token lifetime in the Authorization Server Metadata,
omitted response fields could indicate either indefinite validity or simply lack
of support for this specification. However, indefinite expiration and lack of
information about expiration should be handled by the client in the same way.
That is to say, the client must always handle refresh token invalidation not
caused by expiration, such as by explicit user revocation. Clients MUST NOT
make any assumptions that omitted response fields in one response imply their
omission in later responses too.

Rather than omitting a response value, an authorization server may choose to
return a large arbitrary value, e.g. 315569520 for 10 years. This avoids any
ambiguity around support for indefinite values while achieving a similar
practical effect. Clients MUST treat all large values as literals and MUST NOT
make any assumptions about which may be considered indefinite.

## Error response

The existing `invalid_grant` error code already explicitly covers token
expiration and should be sufficient. Upon receiving this error code the client
SHOULD start a new authorization grant flow.

## Example

Suppose an authorization server enforces that refresh tokens must be exchanged
at least once every 7 days, and a user has granted authorization to an
application for access for 10 days. The initial authorization code grant (Day 0)
will result in the following response values:

    refresh_token_timeout: 604800  // 7 days
    authorization_expires_in: 864000  // 10 days

A refresh token grant on Day 2 will result in the following response values:

    refresh_token_timeout: 604800  // 7 days
    authorization_expires_in: 691200  // 8 days

A refresh token grant on Day 7 will result in the following response values:

    refresh_token_timeout: 259200  // 3 days
    authorization_expires_in: 259200  // 3 days

If instead, the client held the initial refresh token for 8 days (i.e. exceeding
`refresh_token_timeout` but not `authorization_expires_in`), the refresh token
grant will fail:

    error: invalid_grant
    error_description: "expired refresh token"

Note that the error description text is non-normative and for illustrative
purposes only.

# Update to Token Introspection

While Token Introspection [RFC7662] is primarily intended for resource servers
seeking information about received access tokens, the specification does permit
refresh token introspection as well. Authorization servers supporting refresh
token introspection SHOULD support the `refresh_token_timeout` and
`authorization_expires_in` parameters on the endpoint with the same semantics as
defined in [Token endpoint response](#token-endpoint). These parameters SHOULD
be returned only when the presented `token` is a refresh token.

Use of a refresh token on token introspection MUST NOT reset any
`refresh_token_timeout` duration.

# Update to Authorization Server Metadata

Support for the expiring refresh tokens SHOULD be declared in the
OAuth 2.0 Authorization Server Metadata [RFC8414] with the following
metadata:

    refresh_token_expiration_types_supported
        OPTIONAL. JSON array of supported expiration types. The possible values
        are "authorization" and "token_timeout".

If the authorization server omits expiration time response fields to indicate
indefinite validity, it MUST declare `refresh_token_expiration_types_supported`
in its metadata to indicate to the client that it's aware of this spec.

# User Experience Considerations {#ux-considerations}

While clients must be able to gracefully handle tokens' expiring at any time,
the user experience may suffer if there's an unintended interruption of service.
This degradation of experience would most likely be felt by users of clients
running in the background, such as task or travel management apps that rely on
access to a user's calendar or inbox.

If an application recognizes that its access is nearing expiration, it can
proactively prompt the user for reauthorization next time they're "in the loop"
(e.g. using a parameter like `prompt=consent` from [OpenID]), or even
communicate to the user out of band that their granted access is expiring.

Another option an authorization server could provide to the user is a management
surface where the user can go proactively extend the lifetime of their own
grant, which would update the lifetime of the client's refresh token(s) in
place. The client would discover the extended expiration on its next refresh
token grant request.

# Security Considerations

While it is possible to allow refresh token expiration to exceed that of user
authorization expiration if the authorization server checks both timestamps when
validating a refresh token, this is a potentially dangerous source of bugs in
systems with complicated user authorization models. By requiring refresh tokens
to expire no later than user authorization expires, there is less risk of bugs
that accidentally provide data access to the client beyond the term of the
user's authorization.

Authorization servers implementing token rotation on every refresh [RFC 9700]
Sec 4.14 may wish to enforce a maximum duration that a refresh token may be held
without rotation, and this specification allows that duration to be communicated
as part of the API rather than relying on documentation.

Clients may wish to maintain multiple refresh tokens with different access in
order to separate different lifetimes across different scopes. For example, a
short-lived token to access financial data and a long-lived token to access
basic user info. There is a tradeoff here, both in complexity of token
management and also in increased friction for the user to authorize multiple
tokens.

# Privacy Considerations

Allowing users to time-limit their authorization is a privacy improvement. While
this was already doable in regular OAuth implementations, the potential
interruption of service for the user may have discouraged implementation of the
feature. This specification provides a standardized way to mitigate that concern
and should lead to greater adoption of time-limited authorization.

# IANA Considerations

## OAuth Parameters Registration

This specification registers the following OAuth parameter definitions in the
IANA OAuth Parameters registry.

### Registry Contents

*   Name: refresh_token_timeout
    *   Parameter Usage Location: token response
    *   Change Controller: IETF
    *   Reference: This document
*   Name: authorization_expires_in
    *   Parameter Usage Location: token response
    *   Change Controller: IETF
    *   Reference: This document

## OAuth Authorization Server Metadata Registration

This specification registers the following Authorization Server Metadata
definitions in the IANA OAuth Authorization Server Metadata registry.

### Registry Contents

*   Metadata Name: refresh_token_expiration_types_supported
    *   Metadata Description: What types of refresh token expiration are
        supported by the authorization server
    *   Change Controller: IETF
    *   Reference: This document

## Change History

Delete this section before publication.

*   May 8, 2026:
    *   Incorporate Vanshaj's review:
        *   Add language on backwards compatibility for definite duration
            becoming finite.
        *   `refresh_token_expiration_types_supported` `"credential"` value
            renamed to `"token_timeout"`.
        *   Simplified example
        *   Linking UX considerations from RT expiration section
*   Feb 27, 2026:
    *   Address Issues 4, 5, 6 from George to discuss tradeoffs around managing
        multiple tokens or scopes with different expirations, as well as out of
        band reauthorization by the user.
    *   Rewording and clarification based on Dan's suggestions on the list.

--- back

# Acknowledgments
{:numbered="false"}

A special thanks to Aaron Parecki for his continuous feedback and invaluable
assistance in guiding the author through the IETF draft process.

The author would also like to thank Andrii Deinega, George Fletcher, Dan Moore,
and Vanshaj Singhania for their reviews and technical suggestions on the repo
and mailing list.
