---
title: The CONNECT-IP HTTP Method
abbrev: CONNECT-IP
docname: draft-cms-masque-connect-ip-latest
category: std
wg: MASQUE

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "A. Chernyakhovsky"
    name: "Alex Chernyakhovsky"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: achernya@google.com
 -
    ins: "D. McCall"
    name: "Dallas McCall"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dallasmccall@google.com
 -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dschinazi.ietf@gmail.com


--- abstract

This document describes the CONNECT-IP HTTP method. CONNECT-IP is similar to
CONNECT-UDP, but allows transmitting IP packets, without being limited to just
TCP like CONNECT or UDP like CONNECT-UDP.


--- middle

# Introduction

This document describes the CONNECT-IP HTTP method. CONNECT-IP is similar to
CONNECT-UDP, but allows transmitting IP packets, without being limited to just
TCP like CONNECT or UDP like CONNECT-UDP.

CONNECT-IP allows endpoints to set up an IP tunnel between one another. This
can be used to implement a consumer VPN, point-to-point, point-to-network, and
network-to-network capabilities as described in
{{?REQS=I-D.ietf-masque-ip-proxy-reqs}}.


## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
responds to the CONNECT-IP request. If there are
HTTP intermediaries (as defined in Section 2.3 of {{!RFC7230}}) between the
client and the proxy, those are referred to as "intermediaries" in this
document.


# The CONNECT-IP Method {#connect-ip-method}

The CONNECT-IP method establishes a stream to an endpoint server that then
permits the exchange of control data, such as IP address information, reachable
IP ranges, and other relevant information for successfully transmitting IP
datagrams between hosts.

The request-target of a CONNECT-IP request is a URI {{!URI=RFC3986}} which uses
the "https" scheme and a client-specified path. When using HTTP/2
{{!H2=RFC7540}} or later, CONNECT-IP requests use HTTP pseudo-headers with the
following requirements:

* The ":method" pseudo-header field is set to "CONNECT-IP".

* The ":scheme" pseudo-header field is set to "https".

* The ":path" pseudo-header field is set to the value provided by the
  client. That value MUST NOT be empty.

* The ":authority" pseudo-header field contains the host and port of the proxy.
  The target of a CONNECT-IP request is the server providing the CONNECT-IP
  featureset, not an individual endpoint with which a connection is desired.

A CONNECT-IP request that does not conform to these restrictions is
malformed (see {{H2}}, Section 8.1.2.6).

Any 2xx (Successful) response indicates that the proxy is willing to open an IP
tunnel between it and the client. Any response other than a successful response
indicates that the tunnel has not yet been formed.

A proxy MUST NOT send any Transfer-Encoding or Content-Length header fields in
a 2xx (Successful) response to CONNECT-IP. A client MUST treat a successful
response to CONNECT-IP containing any Content-Length or Transfer-Encoding
header fields as malformed.

A payload within a CONNECT-IP request message has no defined semantics; a
CONNECT-IP request with a non-empty payload is malformed. Note that the
CONNECT-IP stream is used to convey control messages, but they are not
semantically part of the request or response themselves.

Responses to the CONNECT-IP method are not cacheable.

The lifetime of the tunnel is tied to the CONNECT-IP stream. Closing the stream
(via the FIN bit on a QUIC STREAM frame, or a QUIC RESET_STREAM frame) closes
the associated tunnel.


# Transmitting IP Packets using HTTP Datagrams

When the HTTP connection supports HTTP/3 datagrams
{{!H3DGRAM=I-D.ietf-masque-h3-datagram}}, IP packets can be sent using them.
The HTTP/3 Datagram Payload contains a full IP packet, from the IP Version
field until the last byte of the IP Payload.


# Routes

Endpoints have the ability to advertise and reject routes using the
ROUTE_ADVERTISEMENT ({{route-adv}}) and ROUTE_REJECTION ({{route-adv}})
messages. Note that these messages are purely informational: receipt of a
ROUTE_ADVERTISEMENT message does not require the recipient to start routing
traffic to its peer. Additionally, if an endpoint receives a ROUTE_REJECTION
for a given prefix that it had previously received a ROUTE_ADVERTISEMENT
message for, then the two cancel out and the endpoint MUST remove its state
from the ROUTE_ADVERTISEMENT message instead of installing new state for the
ROUTE_REJECTION message. Conversely, the same is true of a ROUTE_ADVERTISEMENT
that matches a previous ROUTE_REJECTION. Routes are handled via
longest-prefix-first preference, meaning that if a given IP prefix is covered
by multiple route advertisement and route rejections, the one with the longest
prefix is used.


# Stream Chunks {#stream-chunks}

The DATA stream tied to the bidirectional stream that the CONNECT-IP request
was sent on is a sequence of CONNECT-IP Stream Chunks, which are defined as a
sequence of type-length-value tuples using the following format (using the
notation from the "Notational Conventions" section of
{{!QUIC=I-D.ietf-quic-transport}}):

~~~
CONNECT-IP Stream {
  CONNECT-IP Stream Chunk (..) ...,
}
~~~
{: #stream-chunk-format title="CONNECT-IP Stream Format"}

~~~
CONNECT-IP Stream Chunk {
  CONNECT-IP Stream Chunk Type (i),
  CONNECT-IP Stream Chunk Length (i),
  CONNECT-IP Stream Chunk Value (..),
}
~~~
{: #stream-format title="CONNECT-IP Stream Chunk Format"}

CONNECT-IP Stream Chunk Type:

: A variable-length integer indicating the Type of the CONNECT-IP Stream
Chunk. Endpoints that receive a chunk with an unknown CONNECT-IP Stream Chunk
Type MUST silently skip over that chunk.

CONNECT-IP Stream Chunk Length:

: The length of the CONNECT-IP Stream Chunk Value field following this field.
Note that this field can have a value of zero.

CONNECT-IP Stream Chunk Value:

: The payload of this chunk. Its semantics are determined by the value of the
CONNECT-IP Stream Chunk Type field.

# Messages

## IP_PACKET Message

The IP_PACKET message allows conveying IP Packets when HTTP/3 Datagrams are not
available. This message uses a CONNECT-IP Stream Chunk Type of 0x00. Its value
uses the following format:

~~~
IP_PACKET Message {
  IP Packet (...),
}
~~~
{: #ip-packet-format title="IP_PACKET Message Format"}

IP Packet:

: A full IP packet, from the IP Version field until the last byte of the IP
Payload.

Note that this message MAY still be used even when HTTP/3 datagrams are
available.


## ADDRESS_ASSIGN Message

The ADDRESS_ASSIGN message allows an endpoint to inform its peer that it has
assigned an IP address to it. It allows assigning a prefix which can contain
multiple addresses. This message uses a CONNECT-IP Stream Chunk Type of 0x01.
Its value uses the following format:

~~~
ADDRESS_ASSIGN Message {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #addr-assign-format title="ADDRESS_ASSIGN Message Format"}

IP Version:

: IP Version of this address assignment. MUST be either 4 or 6.

IP Address:

: Assigned IP address. If the IP Version field has value 4, the IP Address
field SHALL have a length of 32 bits. If the IP Version field has value 6, the
IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix assigned, in bits. MUST be lesser or equal to the
length of the IP Address field, in bits.


## ADDRESS_REQUEST Message

The ADDRESS_REQUEST message allows an endpoint to request assignment of an IP
address from its peer. It allows the endpoint to optionally indicate a
preference for which address it would get assigned. This message uses a
CONNECT-IP Stream Chunk Type of 0x02. Its value uses the following format:

~~~
ADDRESS_REQUEST Message {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #addr-req-format title="ADDRESS_REQUEST Message Format"}

IP Version:

: IP Version of this address request. MUST be either 4 or 6.

IP Address:

: Requested IP address. If the IP Version field has value 4, the IP Address
field SHALL have a length of 32 bits. If the IP Version field has value 6, the
IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix requested, in bits. MUST be lesser or equal to the
length of the IP Address field, in bits.

Upon receiving the ADDRESS_REQUEST message, an endpoint SHOULD assign an IP
address to its peer, and then respond with an ADDRESS_ASSIGN message to inform
the peer of the assignment.


## ROUTE_ADVERTISEMENT Message {#route-adv}

The ROUTE_ADVERTISEMENT message allows an endpoint to communicate to its peer
that it is willing to route traffic to a given prefix. This message uses a
CONNECT-IP Stream Chunk Type of 0x03. Its value uses the following format:

~~~
ROUTE_ADVERTISEMENT Message {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #route-adv-format title="ROUTE_ADVERTISEMENT Message Format"}

IP Version:

: IP Version of this route advertisement. MUST be either 4 or 6.

IP Address:

: IP address of the advertised route. If the IP Version field has value 4, the
IP Address field SHALL have a length of 32 bits. If the IP Version field has
value 6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix of the advertised route, in bits. MUST be lesser or
equal to the length of the IP Address field, in bits.

Upon receiving the ROUTE_ADVERTISEMENT message, an endpoint MAY start routing
IP packets in that prefix to its peer.


## ROUTE_REJECTION Message {#route-rej}

The ROUTE_REJECTION message allows an endpoint to communicate to its peer that
it is not willing to route traffic to a given prefix. This message uses a
CONNECT-IP Stream Chunk Type of 0x04. Its value uses the following format:

~~~
ROUTE_REJECTION Message {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #route-rej-format title="ROUTE_REJECTION Message Format"}

IP Version:

: IP Version of this route rejection. MUST be either 4 or 6.

IP Address:

: IP address of the rejected route. If the IP Version field has value 4, the IP
Address field SHALL have a length of 32 bits. If the IP Version field has value
6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix of the advertised route, in bits. MUST be lesser or
equal to the length of the IP Address field, in bits.

Upon receiving the ROUTE_REJECTION message, an endpoint MUST stop routing IP
packets in that prefix to its peer. Note that this message can be reordered
with DATAGRAM frames, and therefore an endpoint that receives packets for
routes it has rejected MUST NOT treat that as an error.


## ROUTE_RESET Message {#route-reset}

The ROUTE_RESET message allows an endpoint to cancel any routes it had
previously advertised or denied. This message uses a CONNECT-IP Stream Chunk
Type of 0x05. Its value uses the following format:

~~~
ROUTE_RESET Message {
}
~~~
{: #route-reset-format title="ROUTE_RESET Message Format"}

Upon receiving the ROUTE_RESET message, an endpoint MUST stop routing IP
packets to its peer. Note that this message can be reordered with DATAGRAM
frames, and therefore an endpoint that receives packets for routes it has
rejected MUST NOT treat that as an error.

The main purpose of the ROUTE_RESET message is to allow endpoints to not have
to remember the full list of routes they have shared with their peer. In
practice, it is expected that ROUTE_RESET messages will be closely followed by
ROUTE_ADVERTISEMENT messages that will refill the routing table that was just
cleared.


## SHUTDOWN Message

The SHUTDOWN message allows an endpoint to communicate to its peer that it is
about to close the CONNECT-IP stream, with a string explaining the reason for
the shutdown. This message uses a CONNECT-IP Stream Chunk Type of 0x06. Its
value uses the following format:

~~~
SHUTDOWN Message {
  Reason Phrase (..),
}
~~~
{: #shutdown-format title="SHUTDOWN Message Format"}

Reason Phrase:

: Additional diagnostic information for the shutdown. This SHOULD be
a UTF-8 encoded string {{!UTF8=RFC3629}}, though the frame does not carry
information, such as language tags, that would aid comprehension by any entity
other than the one that created the text.

Note that the SHUTDOWN message is informational, the tunnel is only closed when
its corresponding CONNECT-IP stream is closed. Endpoints MAY close the tunnel
with a reason phrase by sending the SHUTDOWN message with the FIN bit set on
the underlying QUIC STREAM frame that carried it.


## ATOMIC_START Message

The ATOMIC_START message allows an endpoint to create an atomic set of
messages. This message uses a CONNECT-IP Stream Chunk Type of 0x07. Its value
uses the following format:

~~~
ATOMIC_START Message {
}
~~~
{: #atomic-start-format title="ATOMIC_START Message Format"}

Upon receiving an ATOMIC_START message, an endpoint MUST buffer all incoming
known messages until it receives an ATOMIC_END message. Endpoints MUST NOT send
two ATOMIC_START messages without an ATOMIC_END message between them.

Endpoints MUST NOT buffer unknown messages. Endpoints MAY choose to immediately
process IP_PACKET and SHUTDOWN messages instead of buffering them. Extensions
that register new message types MAY specify that it is allowed to skip
buffering for them.

The purpose of this frame is to avoid timing issues where an endpoint installs
a route before an important route rejection was received. Endpoints SHOULD
group their initial configuration into an atomic block to allow their peer to
mark the tunnel as operational once the whole block is parsed.


## ATOMIC_END Message

The ATOMIC_END message allows an endpoint to end an atomic set of messages.
This message uses a CONNECT-IP Stream Chunk Type of 0x08. Its value uses the
following format:

~~~
ATOMIC_END Message {
}
~~~
{: #atomic-end-format title="ATOMIC_END Message Format"}

Upon receiving an ATOMIC_END message, an endpoint MUST parse all previously
buffered messages, in order of receipt. Endpoints MUST NOT send an ATOMIC_END
message without a preceding ATOMIC_START message.


# Extensibility Considerations

CONNECT-IP can be extended via multiple mechanisms to increase functionality.
There are two main ways to extend CONNECT-IP: HTTP headers and CONNECT-IP
Stream Chunk Types. For example, an authentication extension could define an
HTTP header that allows endpoints to send authentication credentials to their
peer during the creation of the tunnel. Alternatively, one could specify an
extension that defines a new CONNECT-IP Stream Chunk Type which allows
exchanging DNS configuration between endpoints.


# Security Considerations

There are significant risks in allowing arbitrary clients to establish a tunnel
to arbitrary servers, as that could allow bad actors to send traffic and have
it attributed to the proxy. Proxies that support CONNECT-IP SHOULD restrict its
use to authenticated users. The HTTP Authorization header {{?AUTH=RFC7235}} MAY
be used to authenticate clients. More complex authentication schemes are out of
scope for this document but can be implemented using CONNECT-IP extensions.

Since CONNECT-IP endpoints can proxy IP packets send by their peer, they SHOULD
follow the guidance in {{!BCP38=RFC2827}} to help prevent denial of service
attacks.


# IANA Considerations {#iana}

## HTTP Method {#iana-method}

This document will request IANA to register "CONNECT-IP" in the
HTTP Method Registry (IETF review) maintained at
<[](https://www.iana.org/assignments/http-methods)>.

~~~
  +-------------+------+------------+---------------+
  | Method Name | Safe | Idempotent |   Reference   |
  +-------------+------+------------+---------------+
  | CONNECT-IP  |  no  |     no     | This document |
  +-------------+------+------------+---------------+
~~~


## Stream Chunk Type Registration {#iana-chunk-type}

This document will request IANA to create a "CONNECT-IP Stream Chunk Type"
registry. This registry governs a 62-bit space, and follows the registration
policy for QUIC registries as defined in {{QUIC}}. In addition to the fields
required by the QUIC policy, registrations in this registry MUST include the
following fields:

Type:

: A short mnemonic for the type.

Description:

: A brief description of the type semantics, which MAY be a summary if a
specification reference is provided.

The initial contents of this registry are:

~~~
+-------+---------------------+---------------------+---------------+
| Value |        Type         |      Description    |   Reference   |
+-------+---------------------+---------------------+---------------+
| 0x00  |      IP_PACKET      | Full IP packet      | This document |
| 0x01  |   ADDRESS_ASSIGN    | Address Assignment  | This document |
| 0x02  |   ADDRESS_REQUEST   | Address Request     | This document |
| 0x03  | ROUTE_ADVERTISEMENT | Route Advertisement | This document |
| 0x04  |   ROUTE_REJECTION   | Route Rejection     | This document |
| 0x05  |     ROUTE_RESET     | Route Reset         | This document |
| 0x06  |      SHUTDOWN       | Shutdown Reason     | This document |
| 0x07  |    ATOMIC_START     | Atomic Start        | This document |
| 0x08  |     ATOMIC_END      | Atomic End          | This document |
+-------+---------------------+---------------------+---------------+
~~~

Each value of the format `41 * N + 29` for integer values of N (that is, 29,
70, 111, ...) are reserved; these values MUST NOT be assigned by IANA and MUST
NOT appear in the listing of assigned values.


--- back

# Examples

## Consumer VPN

In this scenario, the client will typically receive a single IP address that
the proxy has picked from a pool of addresses it maintains. The client will
route all traffic through the tunnel. The exchange could look as follows:

~~~
    Client                                             Server

    ADDRESS_REQUEST          -------->
      IP Version = 4
      IP Address = 0.0.0.0
      IP Prefix Length = 0

                             <--------  ADDRESS_ASSIGN
                                          IP Version = 4
                                          IP Address = 192.0.2.42
                                          IP Prefix Length = 32

                             <--------  ROUTE_ADVERTISEMENT
                                          IP Version = 4
                                          IP Address = 0.0.0.0
                                          IP Prefix Length = 0
~~~


# Acknowledgments
{:numbered="false"}

The design of CONNECT-IP was inspired by discussions in the MASQUE working
group around {{REQS}}. The authors would like to thank participants in those
discussions for their feedback.
