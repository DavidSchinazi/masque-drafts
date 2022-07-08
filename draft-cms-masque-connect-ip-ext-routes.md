---
title: A Routing Extension to CONNECT-IP
abbrev: CONNECT-IP Routes
docname: draft-cms-masque-connect-ip-ext-routes-latest
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

This document describes an extension to the CONNECT-IP HTTP method. This
extension allows both endpoints to negotiate routes. This enables split-tunnel
VPN services, and network-to-network VPNs.


--- middle

# Introduction

This document describes an extension to the CONNECT-IP HTTP method
{{?CONNECT-IP=I-D.cms-masque-connect-ip}}. This extension allows both endpoints
to negotiate routes. This enables split-tunnel VPN services, and
network-to-network VPNs.

CONNECT-IP allows endpoints to set up an IP tunnel between one another but does
not allow exchanging which routes are supported though the tunnel. This
extension can be used to connect an endpoint or network to another network
without changing default routes.


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


# Routes

Endpoints have the ability to advertise and reject routes using the
ROUTE_ADVERTISEMENT ({{route-adv}}) and ROUTE_REJECTION ({{route-adv}})
capsule. Note that these capsules are purely informational: receipt of a
ROUTE_ADVERTISEMENT capsule does not require the recipient to start routing
traffic to its peer. Additionally, if an endpoint receives a ROUTE_REJECTION
for a given prefix that it had previously received a ROUTE_ADVERTISEMENT
capsule for, then the two cancel out and the endpoint MUST remove its state
from the ROUTE_ADVERTISEMENT capsule instead of installing new state for the
ROUTE_REJECTION capsule. Conversely, the same is true of a ROUTE_ADVERTISEMENT
that matches a previous ROUTE_REJECTION. Routes are handled via
longest-prefix-first preference, meaning that if a given IP prefix is covered
by multiple route advertisement and route rejections, the one with the longest
prefix is used.

When processing ROUTE_ADVERTISEMENT capsules, endpoints MUST check their local
policy before deciding whether to forward packets to their peer. Since ignoring
these capsules is allowed by the protocol, such policy decisions will not
prevent interoperability.


# Capsules

## ROUTE_ADVERTISEMENT Capsule {#route-adv}

The ROUTE_ADVERTISEMENT capsule allows an endpoint to communicate to its peer
that it is willing to route traffic to a given prefix. This indicates that the
sender has an existing route to the prefix, and notifies its peer that if the
receiver of the ROUTE_ADVERTISEMENT capsule sends IP packets for this prefix in
HTTP Datagrams, the sender of the capsule will forward them along its
preexisting route. This capsule uses a Capsule Type of 0xfff102. Its value uses
the following format:

~~~
ROUTE_ADVERTISEMENT Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #route-adv-format title="ROUTE_ADVERTISEMENT Capsule Format"}

IP Version:

: IP Version of this route advertisement. MUST be either 4 or 6.

IP Address:

: IP address of the advertised route. If the IP Version field has value 4, the
IP Address field SHALL have a length of 32 bits. If the IP Version field has
value 6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix of the advertised route, in bits. MUST be lesser or
equal to the length of the IP Address field, in bits.

Upon receiving the ROUTE_ADVERTISEMENT capsule, an endpoint MAY start routing
IP packets in that prefix to its peer.


## ROUTE_REJECTION Capsule {#route-rej}

The ROUTE_REJECTION capsule allows an endpoint to communicate to its peer that
it is not willing to route traffic to a given prefix. This capsule uses a
Capsule Type of 0xfff103. Its value uses the following format:

~~~
ROUTE_REJECTION Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #route-rej-format title="ROUTE_REJECTION Capsule Format"}

IP Version:

: IP Version of this route rejection. MUST be either 4 or 6.

IP Address:

: IP address of the rejected route. If the IP Version field has value 4, the IP
Address field SHALL have a length of 32 bits. If the IP Version field has value
6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix of the advertised route, in bits. MUST be lesser or
equal to the length of the IP Address field, in bits.

Upon receiving the ROUTE_REJECTION capsule, an endpoint MUST stop routing IP
packets in that prefix to its peer. Note that this capsule can be reordered
with DATAGRAM frames, and therefore an endpoint that receives packets for
routes it has rejected MUST NOT treat that as an error.


## ROUTE_RESET Capsule {#route-reset}

The ROUTE_RESET capsule allows an endpoint to cancel any routes it had
previously advertised or denied. This capsule uses a Capsule Type of 0xfff104.
Its value uses the following format:

~~~
ROUTE_RESET Capsule {
}
~~~
{: #route-reset-format title="ROUTE_RESET Capsule Format"}

Upon receiving the ROUTE_RESET capsule, an endpoint MUST stop routing IP
packets to its peer. Note that this capsule can be reordered with DATAGRAM
frames, and therefore an endpoint that receives packets for routes it has
rejected MUST NOT treat that as an error.

The main purpose of the ROUTE_RESET capsule is to allow endpoints to not have
to remember the full list of routes they have shared with their peer. In
practice, it is expected that ROUTE_RESET capsules will be closely followed by
ROUTE_ADVERTISEMENT capsules that will refill the routing table that was just
cleared.


## ATOMIC_START Capsule

The ATOMIC_START capsule allows an endpoint to create an atomic set of
capsules. This capsule uses a Capsule Type of 0xfff106. Its value uses the
following format:

~~~
ATOMIC_START Capsule {
}
~~~
{: #atomic-start-format title="ATOMIC_START Capsule Format"}

Upon receiving an ATOMIC_START capsule, an endpoint MUST buffer all incoming
known CONNECT-IP-specific capsules (i.e., capsules defined in this document)
until it receives an ATOMIC_END capsule. Endpoints MUST NOT send two
ATOMIC_START capsules without an ATOMIC_END capsule between them.

Endpoints MUST NOT buffer unknown capsules. Endpoints MAY choose to immediately
process IP_PACKET and SHUTDOWN capsules instead of buffering them. Capsules
defined in other documents are by default not buffered by ATOMIC_START.
Extensions that register new capsule types MAY specify that these capsules
should be buffered by ATOMIC_START, and whether it is allowed to skip buffering
for them.

The purpose of this frame is to avoid timing issues where an endpoint installs
a route before an important route rejection was received. Endpoints SHOULD
group their initial configuration into an atomic block to allow their peer to
mark the tunnel as operational once the whole block is parsed.


## ATOMIC_END Capsule

The ATOMIC_END capsule allows an endpoint to end an atomic set of capsules.
This capsule uses a Capsule Type of 0xfff107. Its value uses the following
format:

~~~
ATOMIC_END Capsule {
}
~~~
{: #atomic-end-format title="ATOMIC_END Capsule Format"}

Upon receiving an ATOMIC_END capsule, an endpoint MUST parse all previously
buffered capsules, in order of receipt. Endpoints MUST NOT send an ATOMIC_END
capsule without a preceding ATOMIC_START capsule.


# Security Considerations

In theory, endpoints could use ROUTE_ADVERTISEMENT capsules to divert traffic
from naive endpoints. To avoid this, receivers of ROUTE_ADVERTISEMENT capsules
MUST check their local policy before acting on such capsules, see {{routes}}.


# IANA Considerations {#iana}

## Capsule Type Registrations {#iana-capsule-types}

This document will request IANA to add the following values to the "HTTP
Capsule Types" registry created by {{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}}:

~~~
+----------+---------------------+---------------------+---------------+
|   Value  |        Type         |      Description    |   Reference   |
+----------+---------------------+---------------------+---------------+
| 0xfff102 | ROUTE_ADVERTISEMENT | Route Advertisement | This document |
| 0xfff103 |   ROUTE_REJECTION   | Route Rejection     | This document |
| 0xfff104 |     ROUTE_RESET     | Route Reset         | This document |
| 0xfff106 |    ATOMIC_START     | Atomic Start        | This document |
| 0xfff107 |     ATOMIC_END      | Atomic End          | This document |
+----------+---------------------+---------------------+---------------+
~~~


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
group around {{?REQS=I-D.ietf-masque-ip-proxy-reqs}}. The authors would like to
thank participants in those discussions for their feedback.
