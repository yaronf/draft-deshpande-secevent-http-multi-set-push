---
title: "Push-Based Delivery For Multiple Security Event Tokens (SET) Using HTTP"
abbrev: "Push-multi-SET"
category: std

docname: draft-deshpande-secevent-http-multi-set-push-latest
submissiontype: IETF
number:
date:
v: 3
area: "Security"
workgroup: "Security Events"
keyword:
 - security event
 - secevent
venue:
  group: "Security Events"
  type: "Working Group"
  mail: "id-event@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/id-event/"
  github: "appsdesh/draft-deshpande-secevent-http-multi-set-push"
  latest: "https://appsdesh.github.io/draft-deshpande-secevent-http-multi-set-push/draft-deshpande-secevent-http-multi-set-push.html"

author:
 -
    fullname: Apoorva Deshpande
    organization: Okta
    email: apoorva.deshpande@okta.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
   RFC8417:
   RFC8935:
   RFC9110:
   RFC9457:
   RFC9846:
   RFC9728:
   RFC8259:
   RFC2277:
   RFC6838:

informative:
   IANA.media-types:


--- abstract

This specification defines how multiple Security Event Tokens (SETs) can be
delivered to an intended recipient using HTTP POST over TLS.  The SETs
are transmitted in the body of an HTTP POST request to an endpoint
operated by the recipient, and the recipient indicates successful or
failed transmission via the HTTP response.

--- middle

# Introduction

This specification defines a mechanism by which a Transmitter of a Security Event Token (SET) {{RFC8417}} can deliver multiple SETs to an intended SET Recipient via HTTP POST {{RFC9110}} over TLS in a single POST request. {{RFC8935}} focuses on the delivery of the single SET to the Recipient. When sending a large number of SETs, sending them one by one is inefficient. This specification defines a way to send batches of SETs in a single POST request for more efficient transport.

Push-Based delivery for multiple SETs is intended to help in the following scenarios:

- The Transmitter of the SET has multiple outstanding SETs to be communicated to the Recipient
- The Transmitter wants to reduce the number of outbound requests to the same Recipient to optimize performance and avoid being rate-limited when the number of SETs to be communicated is high
- The Recipient wants to optimize processing multiple SETs
- The Recipient wants to acknowledge or provide error responses to previously received SETs, but wants to do so asynchronously, rather than within the response to the same HTTP POST in which it received the SET

This specification will handle all the use cases and scenarios for the {{RFC8935}} and make it more extensible to support multiple SETs per one outbound POST request.

Similar to {{RFC8935}}, this specification does not define how the Transmitter and Recipient exchange configuration metadata, such as endpoint URLs, cryptographic keys, and implementation constraints like buffer size limitations.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Push endpoint to receive multiple SETs

Each Recipient that supports this specification MUST support a new push endpoint that receives multiple SETs in a single request. This endpoint MUST be capable of serving HTTP POST {{RFC9110}} requests. This endpoint MUST be TLS {{RFC9846}} enabled and MUST reject any communication not using TLS.
How the Transmitter obtains this endpoint from the Recipient is outside the scope of this specification.


# SET Delivery Semantics

In this SET delivery using HTTP over TLS, a Transmitter delivers zero or more SETs in a JavaScript Object Notation (JSON) {{RFC8259}} document
to the SET Recipient. The Recipient either acknowledges the successful receipt of the SETs or indicates failure in processing of one or more SETs in a JSON document to the Transmitter.

The Transmitter SHOULD periodically send a request with zero SETs to allow the Recipient to respond back with an ack or err for previously transmitted SETs that have not yet been acknowledged.

After successful (acknowledged) SET delivery, SET Transmitters are not required to retain or record SETs for retransmission. Once a SET is acknowledged, the SET Recipient SHALL be responsible for retention, if needed. Transmitters may also discard undelivered SETs under deployment-specific conditions, such as if they have not been acknowledged (success or failure) for too long a period of time or if an excessive amount of storage is needed to retain them. If a Transmitter receives an acknowledgement or error for a SET it has no record of, the Transmitter MUST ignore that acknowledgement or error.

Upon receiving a SET, the SET Recipient reads the SET and validates it in the manner described in {{Section 2 of RFC8935}}. The SET Recipient MUST acknowledge receipt to the SET Transmitter, and SHOULD do so in a timely fashion (e.g., milliseconds). The SET Recipient SHALL NOT use the event acknowledgement mechanism to report event errors other than those relating to the parsing and validation of the SET.

## Acknowledgement for all SETs

A Recipient MUST ensure that it includes the `jti` value of each SET it receives, either in an ack or a setErrs value, to the Transmitter from which it received the SETs. A Transmitter SHOULD retry sending the same SET again if it was never responded to either in an ack value or in a setErrs value by a Recipient in a reasonable time period. A Transmitter MAY limit the number of times it retries sending a SET. A Transmitter MAY publish the retry time period and maximum number of retries to its peers, but such publication is outside the scope of this specification.

## Uniqueness of SETs

A Transmitter MUST NOT send two SETs with the same `jti` value if the SET has been either acknowledged through ack value or produced an error indicated by a setErrs value. If a Transmitter wishes to re-send an event after it has received an error response through a setErrs value, then it MUST generate a new SET that has a new (and unique) jti value.

This specification does not mandate replay protection on the Recipient. However, if a Recipient receives a SET with a `jti` value that it has already acknowledged or reported an error for, it MAY silently drop the duplicate rather than reprocessing it.

## Transmitting SETs

To transmit a SET to a SET Recipient, the SET Transmitter makes an HTTP POST request to a TLS-enabled HTTP endpoint provided by the SET Recipient. The body of this request is of the content type `"application/secevents+json"` (see {{media-type-registration}}) and the Accept header field MUST be `"application/json"`.

A Transmitter may initiate communication with the Recipient in order to:

-  Send SETs to the Recipient
-  Receive acknowledgement of SETs in response

The body of this request MUST contain the following fields:

### The `sets` Field {#sets}

REQUIRED. A JSON object containing key-value pairs in which the key of a field is a string that contains the `jti` claim of the SET that is specified in the value of the field. This field MAY be an empty object to indicate that no SETs are being delivered by the initiator in this communication. The maximum number of SETs in a push MAY be set by the Transmitter for itself and SHOULD be communicated offline to the Recipients.


The following is a non-normative example of a request.

      POST /push HTTP/1.1
      Host: recipient.example.com
      Content-Type: application/secevents+json
      Accept: application/json

      {
        "sets": {
          "4d3559ec67504aaba65d40b0363faad8":
          "eyJhbGciOiJub25lIn0.
          eyJqdGkiOiI0ZDM1NTllYzY3NTA0YWFiYTY1ZDQwYjAzNjNmYWFkOCIsImlhdC
          I6MTQ1ODQ5NjQwNCwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
          YXVkIjpbImh0dHBzOi8vc2NpbS5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
          ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tL0Zl
          ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwiZXZlbnRzIjp7InVybj
          ppZXRmOnBhcmFtczpzY2ltOmV2ZW50OmNyZWF0ZSI6eyJyZWYiOiJodHRwczov
          L3NjaW0uZXhhbXBsZS5jb20vVXNlcnMvNDRmNjE0MmRmOTZiZDZhYjYxZTc1Mj
          FkOSIsImF0dHJpYnV0ZXMiOlsiaWQiLCJuYW1lIiwidXNlck5hbWUiLCJwYXNz
          d29yZCIsImVtYWlscyJdfX19.",
          "3d0c3cf797584bd193bd0fb1bd4e7d30":
          "eyJhbGciOiJub25lIn0.
          eyJqdGkiOiIzZDBjM2NmNzk3NTg0YmQxOTNiZDBmYjFiZDRlN2QzMCIsImlhdC
          I6MTQ1ODQ5NjAyNSwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwi
          YXVkIjpbImh0dHBzOi8vamh1Yi5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MW
          ZhNWJiYzg3OTU5M2I3NzU0IiwiaHR0cHM6Ly9qaHViLmV4YW1wbGUuY29tL0Zl
          ZWRzLzVkNzYwNDUxNmIxZDA4NjQxZDc2NzZlZTciXSwic3ViIjoiaHR0cHM6Ly
          9zY2ltLmV4YW1wbGUuY29tL1VzZXJzLzQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIx
          ZDkiLCJldmVudHMiOnsidXJuOmlldGY6cGFyYW1zOnNjaW06ZXZlbnQ6cGFzc3
          dvcmRSZXNldCI6eyJpZCI6IjQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIxZDkifSwi
          aHR0cHM6Ly9leGFtcGxlLmNvbS9zY2ltL2V2ZW50L3Bhc3N3b3JkUmVzZXRFeH
          QiOnsicmVzZXRBdHRlbXB0cyI6NX19fQ."
        }
      }

_Figure 1: Example of SET Transmission_

In the above example, the Transmitter is sending 2 SETs to the Recipient.

      POST /push HTTP/1.1
      Host: recipient.example.com
      Content-Type: application/secevents+json
      Accept: application/json

      {
        "sets": {}
      }

_Figure 2: Example of empty SET transmission_

In the above example, the Transmitter is sending zero SETs to the Recipient. This placeholder/empty request allows the Recipient to respond back with ack/err for previously transmitted SETs.

The SET Transmitter MAY include in the request an Accept-Language header field to indicate to the SET Recipient the preferred language(s) in which to receive error message descriptions.

## Response Communication

A Recipient MUST respond to the communication by sending an HTTP response. The body of this response is of the content type `"application/json"`. It contains the following fields:

`ack`
REQUIRED. An array of strings, in which each string is the `jti` value of a previously received SET that is acknowledged in this object. This array MAY be empty to indicate that no previously received SETs are being acknowledged in this communication.

`setErrs`
OPTIONAL. A JSON object containing key-value pairs in which the key of a field is a string that contains the `jti` value of a previously received SET that the sender of the communication object was unable to process. The value of the field is a JSON object that has the following fields:

`err`
REQUIRED. The short reason why the specified SET failed to be processed. Error codes are described in Section 2.4 of [RFC8935]. Note that the `authentication_failed` and `access_denied` codes described therein apply to the SET Transmission Request as a whole rather than to an individual SET, and therefore MUST NOT be used in a setErrs value; such failures MUST instead be reported using the `err` field described in {{failure-response}}.

`description`
OPTIONAL. An explanation of why the SET failed to be processed.

If the response contains a `description`, then the response MUST include a Content-Language header field whose value indicates the language of the error descriptions included in the response body. If the SET Recipient can provide error descriptions in multiple languages, they SHOULD choose the language to use according to the value of the Accept-Language header field sent by the SET Transmitter in the transmission request, as described in {{Section 12.5.4 of RFC9110}}. If the SET Transmitter did not send an Accept-Language header field, or if the SET Recipient does not support any of the languages included in the header field, the SET Recipient MUST respond with messages that are understandable by an English-speaking person, as described in Section 4.5 of [RFC2277].

### Success Response {#success-response}

If the Recipient is successful in accepting the request, it MUST return the HTTP status code 202 (Accepted). The response MUST have the content-type `"application/json"`.

      HTTP/1.1 202 Accepted
      Content-type: application/json

      {
        "ack": [
          "3d0c3cf797584bd193bd0fb1bd4e7d30"
        ]
      }

_Figure 3: Example of SET Transmission response with ack_

In the above example, the Recipient acknowledges one of the SETs it previously received. There are no errors reported by the Recipient.

      HTTP/1.1 202 Accepted
      Content-type: application/json

      {
         "ack": [
          "f52901c499611ef94540242ac12000322",
          "0636e274399711ef9454-0242ac120002",
          "d563c72479a04ff0ba415657fa5e2cb11"
         ],
         "setErrs": {
          "4d3559ec67504aaba65d40b0363faad8" : {
            "err": "invalid_key",
            "description": "Failed validation"
          }
         }
      }

_Figure 4: Example of SET Transmission response, ack and errors_

In the above example, the Recipient acknowledges three of the SETs it previously received. There are errors reported by the Recipient for acknowledging one SET.

### Failure Response {#failure-response}

In the event of a general HTTP error condition that is not specific to an individual SET, the SET Recipient responds with the applicable HTTP status code, as defined in {{Section 15 of RFC9110}}.

When the SET Recipient rejects the request as a whole (for example, because the request is malformed, or the Transmitter is not authenticated or authorized), the SET Recipient SHOULD describe the failure using the problem details format defined in {{RFC9457}}. The Content-Type header field of such a response MUST be `"application/problem+json"`. In addition to the members defined in {{RFC9457}}, the response MAY include the following extension member:

`err`
OPTIONAL. A short, machine-readable code identifying the reason the request failed. This code is not specific to any individual SET; it indicates a request-level or service-level failure.

Note that failure responses in this specification are not specific to failures related to any individual SET. SET-specific errors are communicated in a success response payload as defined in the {{success-response}} Section.

Example values for the `err` extension member that can indicate request-level failures include, but are not limited to:

- `invalid_request` (request is malformed)
- `authentication_failed` (authentication token provided by the Transmitter is expired, revoked or invalid)
- `access_denied` (the Transmitter does not have adequate permissions to invoke this API)
- `too_many_sets` (the Transmitter included too many SETs in a single request; this is an indication for the Transmitter to make a request with a lower number of SETs or to comply with the maximum SET count that the Recipient published outside of this specification)


      HTTP/1.1 400 Bad Request
      Content-Language: en-US
      Content-Type: application/problem+json

      {
        "type": "https://example.com/probs/authentication-failed",
        "title": "Authentication failed",
        "status": 400,
        "detail": "Access token has expired.",
        "err": "authentication_failed"
      }

_Figure 5: Example Error Response (authentication_failed)_

The non-normative example above indicates that the access token included in the request is expired.

#### Out of order delivery

A Response may contain `jti` values in its ack or setErrs that do not correspond to the SETs received in the same Request to which the Response is being sent. They MAY consist of values received in previous Requests.

# Authentication and Authorization {#authn-and-authz}

The Transmitter MUST verify the identity of the Recipient by validating
the TLS certificate presented by the Recipient during the TLS handshake, and verifying that
it is the intended recipient of the request, before sending the SETs.

How the Transmitter and Recipient agree on authorization of the request is out of scope of this document.

This section describes server-side authentication of the Recipient by the Transmitter. Authentication of the Transmitter by the Recipient (e.g., via OAuth tokens, mutual TLS, or other mechanisms) is out of scope of this document and is expected to be defined by profiles of this specification.

# Delivery Reliability

A Transmitter MUST attempt to deliver any SETs it has previously attempted to deliver to a Recipient until:

   - It receives an acknowledgement through the ack value for that SET in a subsequent communication with the Recipient
   - It receives a setErrs object for that SET in a subsequent communication with the Recipient
   - It has attempted to deliver the SET a maximum number of times and has failed to communicate either due to communication errors or lack of inclusion in ack or setErrs in subsequent communications that were conducted for the maximum number of times. The maximum number of attempts MAY be set by the Transmitter for itself and SHOULD be communicated offline to the Recipients

Additionally consider Delivery Reliability aspects discussed in {{Section 4 of RFC8935}}.

## Event Ordering and Processing Guarantees

This specification is a transport efficiency mechanism and it does not address transactional aspects of the request. Every SET is an independent event in the request to the Recipient. The event ordering in the request does not imply any chronological dependence. For chronological dependence, the Recipient should look at the time-related event claims.

A Transmitter should not assume the ordered processing of the SETs by the Recipient sub-systems. This specification does not add any transactional requirements on the Recipient.

# Security Considerations {#security-considerations}

The Security Considerations of {{RFC8935}}, {{RFC9846}}, and {{Section 17 of RFC9110}} apply to this specification.

## Too many SETs in the request

This mechanism allows a Transmitter to send a large number of SETs in a single request. A malicious or misconfigured Transmitter could send an extremely large payload, attempting to exhaust memory or CPU resources on the Recipient during JSON parsing or SET validation.

Recipients MUST protect themselves against such attacks. It is RECOMMENDED that Recipients establish and document a reasonable upper limit on both the number of SETs and the total size, in bytes, of the request body they will process in a single request. Limiting the number of SETs alone is insufficient, since a request containing few but excessively large SETs can still exhaust memory or CPU resources. The Transmitter MUST obey the maximum number of SETs and maximum request body size communicated by the Recipient. This will avoid any potential truncations/loss of information at the Recipient.

If a Recipient receives a request exceeding either limit, it SHOULD reject the entire request with a `413 Payload Too Large` HTTP status code.

How the Recipient conveys these upper limits to Transmitters is outside the scope of this specification (see {{sets}} for the `sets` field definition).


## Authentication and Authorization

The Transmitter MUST follow the procedures described in section {{authn-and-authz}} in order to securely authenticate and authorize the Recipient.

## HTTP and TLS

The Transmitter MUST use TLS {{RFC9846}} to communicate with the Recipient and is subject to the security considerations of HTTP {{Section 17 of RFC9110}}.

Failure to properly validate the Recipient's TLS certificate could allow a Transmitter to send SETs to an impersonating endpoint, resulting in the disclosure of sensitive security event information to an unauthorized party.

## Event Delivery Latency

The primary purpose of security event tokens is the timely communication of security-sensitive information. While this specification enables batching for efficiency, Transmitters MUST NOT unduly delay the transmission of events in an attempt to create larger batches.

Delaying the transmission of a time-sensitive event, such as a credential compromise or session revocation, defeats the purpose of the protocol and provides an adversary with a larger window of opportunity to act.

It is RECOMMENDED that Transmitters implement a batching policy that sends a pending batch of SETs when either of the following conditions is met:

- The number of SETs in the batch reaches a configured size limit.
- A configured amount of time (e.g., 1-2 seconds) has elapsed since the oldest SET in the batch was generated.

This ensures a balance between network efficiency and the real-time nature of the communication.

## Information Disclosure in Error Responses

The `setErrs` is designed for debugging and provides valuable feedback. However, if implemented incorrectly, it can become a source of information leakage, disclosing internal details or enabling enumeration type attacks.

It is RECOMMENDED that `setErrs` information be designed to be helpful without revealing sensitive information about internal architecture. For example, a `description` SHOULD NOT echo back tenant identifiers, internal user identifiers, or other values that could enable an attacker to enumerate valid accounts or infer details of the Recipient's internal architecture, such as stack traces or database identifiers.

Additional security considerations in {{Section 5 of RFC8935}}.

# Privacy Considerations

Privacy Considerations from {{Section 6 of RFC8935}} apply.

# IANA Considerations

This document registers the `application/secevents+json` media type in the "Media Types" registry {{IANA.media-types}}.

## Media Type Registration {#media-type-registration}

### Registry Contents

This section registers the `application/secevents+json` media type {{RFC6838}} in the "Media Types" registry {{IANA.media-types}} in the manner described in {{RFC6838}}. This media type is used to indicate that the content is a JSON {{RFC8259}} object carrying a batch of Security Event Tokens (SETs), as described in {{sets}}.

- Type name: application
- Subtype name: secevents+json
- Required parameters: N/A
- Optional parameters: N/A
- Encoding considerations: 8bit; the content is a JSON object as defined in {{RFC8259}} and is always encoded using UTF-8
- Security considerations: See {{security-considerations}} of this document and {{Section 5 of RFC8935}}
- Interoperability considerations: N/A
- Published specification: {{sets}} of this document
- Applications that use this media type: Applications that deliver batches of Security Event Tokens (SETs) over HTTP
- Fragment identifier considerations: N/A
- Additional information:
  - Magic number(s): N/A
  - File extension(s): N/A
  - Macintosh file type code(s): N/A
- Person &amp; email address to contact for further information: Apoorva Deshpande, apoorva.deshpande@okta.com
- Intended usage: COMMON
- Restrictions on usage: none
- Author: Apoorva Deshpande, apoorva.deshpande@okta.com
- Change controller: IETF
- Provisional registration? No

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to acknowledge the following individuals
who contributed ideas, feedback, and wording that shaped and formed the final specification:

Atul Tulshibagwale, Yair Sarig, Yaron Sheffer.

