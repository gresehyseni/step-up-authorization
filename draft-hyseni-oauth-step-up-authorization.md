---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "OAuth 2.0 Step Up Authorization Challenge Protocol"
abbrev: "Step Up AuthZ"
category: std

docname: draft-hyseni-oauth-step-up-authorization-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Gresë Hyseni
    organization: Raiffeisen Bank International
    email: grese.hyseni@rbinternational.com
 -
    fullname: Yaron Zehavi
    organization: Raiffeisen Bank International
    email: yaron.zehavi@rbinternational.com
 -
    fullname: Henrik Kroll
    organization: Raiffeisen Bank International
    email: henrik.kroll@rbinternational.com

normative: RFC6749
name: RFC6749
title: "The OAuth 2.0 Authorization Framework"
target: https://www.rfc-editor.org/info/rfc6749

name: RFC6750
title: "The OAuth 2.0 Authorization Framework: Bearer Token Usage"
target: https://www.rfc-editor.org/info/rfc6750

name: RFC9068
title: "JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens"
target: https://datatracker.ietf.org/doc/rfc9068

name: RFC9396
title: "OAuth 2.0 Rich Authorization Requests"
target: https://www.rfc-editor.org/info/rfc9396

name: RFC9470
title: "OAuth 2.0 Step-Up Authentication Challenge Protocol"
target: https://www.rfc-editor.org/info/rfc9470

informative:

--- abstract
This document proposes an extension to RFC 9470 (OAuth 2.0 Step-Up Authentication Challenge Protocol) to support scenarios where the Resource Server requires transaction-specific authorization information. A new error code, insufficient_authorization_details and unment_authorization_requirements are defined, along with including of authorization_details (as per RFC 9396) or authorization_details_reference_uri in the response to enable step-up authorization based on dynamic operation bound data.


--- middle

# Introduction

OAuth 2.0 Step-Up Authentication (RFC 9470) defines a way for Resource Servers (RS) to indicate that a stronger user authentication is required by responding with the WWW-Authenticate header using insufficient_user_authentication error. However, in many real-world applications, particularly in financial or regulated domains, the challenge is not only about authentication strength level but also about whether the Access Token (AT) contains sufficient authorization.

This document defines a new error code, insufficient_authorization_details, and extends the use of WWW-Authenticate headers to be returned alongside a body that carries authorization_details (as per RFC 9396) or authorization_details_reference_uri. This enables RSs to indicate that a specific operation requires additional authorization context that must be included in a new AT obtained via OAuth 2.0 + RAR flow.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview
The following is an end-to-end sequence of a typical step up authorization scenario implemented according to this specification. The scenario assumes that, before the sequence described below takes place, the client already obtained an access token for the protected resource.

+----------+                                          +--------------+
|          |                                          |              |
|          |-----------(1) request ------------------>|              |
|          |                                          |              |
|          |<---------(2) challenge ------------------|   Resource   |
|          |                                          |    Server    |
|  Client  |                                          |              |
|          |-----------(5) request ------------------>|              |
|          |                                          |              |
|          |<-----(6) protected resource -------------|              |
|          |                                          +--------------+
|          |
|          |
|          |  +-------+                              +---------------+
|          |->|       |                              |               |
|          |  |       |--(3) authorization request-->|               |
|          |  | User  |                              |               |
|          |  | Agent |<-----------[...]------------>| Authorization |
|          |  |       |                              |     Server    |
|          |<-|       |                              |               |
|          |  +-------+                              |               |
|          |                                         |               |
|          |<-------- (4) access token --------------|               |
|          |                                         |               |
+----------+                                         +---------------+
Figure 1: Abstract Protocol Flow

1. The client requests a protected resource, presenting an access token.
2. The resource server evaluated and determines that the presented access token was obtained lacks sufficient authorization context; hence, it denies the request and returns a challenge describing (using a combination of authorization_details) what authorization requirements must be met for the resource server to authorize a request.
3. The client directs the user agent to the authorization server with an authorization request that in addition includes the authorization_details retrieved by the resource server in the previous step.
4. Then, the authorization server returns a new access token to the client. The new access token contains or references the authorization_details.
5. The client retries the original request from step 1, presenting the newly obtained access token.
The resource server validates the new access_token including the authorization_details and determines complies with its requirements and returns a successful response. When the access token is bound to a transaction id, the resource server discards the access token once used.

As seen from the flow above, the resource server has the capability to determine whether the operation requires the presence of authorization_details. It also has the capability of generating the neccessary authorization_details to be consented ba the user. This makes sense compared to a scenario where the client would construct the authorization_details for the following reasons: 
1. the client can't know without prealignment the structure of RAR,
2. the client doesn't need to learn this structure, because it is not the one who will interpret it
3. this support the use case when the RS makes a decision based not only on the endpoint called but also based on the dynamic data in the body payload: e.g. when amount higher than 1000 then require user to explicitly consent this.

It is also mentioned that RS may return an indicator whether the required new token shall be a one time token, this to let the client know that this token is not meant as a substitute for the original access token retrieved during the login flow. (Here look into rfc9470 and maybe refine based on that)

# Resource Server Initial Request

# Resource Server Authorization Challenge response
HTTP/1.1 403 Forbidden
This specification introduces a nre error code value to allow a RS to indicase insufficient authorization context.

insufficient_authorization_details: signals that the token is valid for general access but lacks contextual authorization for the operation.

authorization_details: follows the JSON structure of RFC 9396. OPTIONAL.

authorization_details_reference_uri: a dynamically generated URI where the RS exposes the authorization_details payload for the AS to retrieve. OPTIONAL.

once: used to indicate the client that the requested new token will be used only once and dicarded, which serves as a guidance to the client to not discard their current login tokens. OPTIONAL.

Below we show an example of RS responding with the Step Up Authorization challenge in order to proceed with authorizing a transactional operation.

HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer
  error="insufficient_authorization_details",
  error_description="User must authorize this transaction.",
  scope="init_order",
  acr_values="urn:rbi:psd2:sca",
  authorization_details=[
    {
      "type": "trading_order",
      "amount": "1000.00",
      "currency": "EUR",
      "stock": "XYZ"
    }
  ]

Figure 2: Authorization Reqirements Challenge

Note: the logic through which the resource server determines that the current request does not meet the authorization requirements of the protected resource is out of scope for this document. 

# Authorization Request
A client receiving a challenge from the resource server carrying the insufficient_authorization_details error code SHOULD parse the WWW-Authenticate header for scope, max_age and the response body for authorization_details value and use them, if present, in constructing an authorization request. This request is then conveyed to the authorization server's authorization endpoint via the user agent in order to obtain a new access token complying with the corresponding requirements. The authorization_details is defined by RFC9396. This type of error and the presence of authorization_details should tell client to not start a new [OIDC] flow with intention to substitute login tokens, rather start a only authorization OAuth2.0 + RAR flow. 

The example authorization request URI below, which might be used after receiving the challenge in Figure 2.

GET /authorize?
  response_type=code&
  client_id=wallet.app&
  redirect_uri=https://wallet.app/cb&
  scope=init_order&
  acr_values=urn:rbi:psd2:sca&
  authorization_details=%5B...%5D&
  state=xyz&
  max_age=0

Figure 3. Authorization Request + RAR

# Authorization Response
Authorization server uses the retrieved authorization_details and/or scope to let user consent and then authorize user according to the new requirements (e.g. User signs the on the authorization data presented).

{
  "access_token": "AT+RAR",
  "token_type": "Bearer",
  "expires_in": 3600,
  "authorization_details": [
    {
      "type": "trading_order",
      "status": "authorized",
      "signed_at": "2025-05-22T12:34:00Z"
    }
  ]
}

Figure 4. Access Token with authorization_details included

# Authorization Error Response
This specification introduces a new error_type to the Authorization Error Response defined by RFC9396 to indicate a situation when authorization_data and/or scope format is valid but user did not consent to it. 

unment_authorization_requirements: signals he authorization request for the given authorization_details and/or scope has failed due to AS or user decision.

# Resource Server Access Token evaluation
How the RS evaluates whether an access token meets the protected resource's authorization requirements it depends on the use case and it's not defined on this specification. 

# Transactional operations
In the use case when user must consent to explicitly on the dynamic data of an operation of a transaction nature, then the RS should indicate this data on the authorization_details and convay the transaction-bound nature by including the one_time value. 

When the RS retrieves an Access Token with the authorization_details inside it, it should compare validate the contents or authorization_details with the transaction data retrieved on the body, and return  discard it after one use. The rest of the validation is done as defined by RFC9068.

Below is an example of a succesful response after the protected endpoint is ivoked with the access token from Figure 4.

{
  "order_status": "INITIATED",
  "order_id": "12345",
  "details": {
    "stock": "XYZ",
    "amount": "1000.00",
    "currency": "EUR"
  }
}

Figure 5. Succesful response

# Authorization Server Metadata
Authorization servers can advertise their support of this specification by including in their metadata document, as defined in [RFC8414].

# Security Considerations
This specification adds to previously defined OAuth mechanisms. Their respective security considerations apply:

OAuth 2.0 [RFC6749],
JWT access tokens [RFC9068],
Bearer WWW-Authenticate [RFC6750],
token introspection [RFC7662], and
authorization server metadata [RFC8414].
OAuth 2.0 Rich Authorization Requests

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
