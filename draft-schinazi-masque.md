---
title: The MASQUE Protocol
abbrev: MASQUE
docname: draft-schinazi-masque
category: exp

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "D. Schinazi"
    name: "David Schinazi"
    organization: "Google LLC"
    street: "1600 Amphitheatre Parkway"
    city: "Mountain View, California 94043"
    country: "United States of America"
    email: dschinazi.ietf@gmail.com


--- abstract

This document describes MASQUE (Multiplexed Application Substrate over QUIC
Encryption). MASQUE is a framework that allows concurrently running multiple
networking applications inside a QUIC or HTTP/3 connection. For example, MASQUE can
allow a QUIC client to negotiate proxying capability with a MASQUE server,
and subsequently make use of this functionality while concurrently processing
multiple HTTP/3 requests and responses.

This document is a straw-man proposal. It does not contain enough details to
implement the protocol, and is currently intended to spark discussions on
the approach it is taking. Discussion of this work is encouraged to happen on
the MASQUE IETF mailing list <masque@ietf.org> or on the GitHub repository
which contains the draft: <https://github.com/DavidSchinazi/masque-drafts>.


--- middle

# Introduction {#introduction}

This document describes MASQUE (Multiplexed Application Substrate over QUIC
Encryption). MASQUE is a framework that allows concurrently running multiple
networking applications inside a QUIC or HTTP/3 connection (see
{{!HTTP3=I-D.ietf-quic-http}}). For example, MASQUE can allow a QUIC client to
negotiate proxying capability with a MASQUE server, and subsequently make use
of this functionality while concurrently processing multiple HTTP/3 requests and
responses.

MASQUE Negotiation is performed using HTTP mechanisms, but MASQUE applications
can subsequently leverage QUIC {{!QUIC=I-D.ietf-quic-transport}} features
without using HTTP.

This document is a straw-man proposal. It does not contain enough details to
implement the protocol, and is currently intended to spark discussions on
the approach it is taking. Discussion of this work is encouraged to happen on
the MASQUE IETF mailing list <masque@ietf.org> or on the GitHub repository
which contains the draft: <https://github.com/DavidSchinazi/masque-drafts>.


## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Overview

A client that intents to use a MASQUE server for forwarding support to an appliaction server, when
e.g. end-to-end connectivity is not possible and additional encryption is desired, opens an HTTP/3
connection to the MASQUE server. The proxy IP address and port number (e.g. 443) are either known
or have been previously discovered which is out of scope for this document.

It then negotiates use of supported MASQUE Applications with the
MASQUE server and instructs the server to forward its end-to-end traffic to the target server respectively.
This will, when successfully negotiated, lead to a setup where an end-to-end connection between the client and
the target server is tunnelled within an outer HTTP/s or QUIC connection between the client and the proxy. 
The proxy observes and forwards the end-to-end traffic between the client and the server but cannot
alter or decrypt it as it is end-to-end security protected.

# MASQUE Negotiation {#negotiation}

In order to negotiate the use of the MASQUE protocol, the client starts by
sending a MASQUE request in the HTTP data of an HTTP POST request to
"/.well-known/masque/initial". The client can use this to request specific
MASQUE applications and advertise support for MASQUE extensions. The MASQUE
server indicates support for MASQUE by sending an HTTP status code 200 response,
and can use the data to inform the client of which MASQUE applications are now
in use, and various configuration parameters.

If the server does not reply with a 200 response, MASQUE is not supported and 
the client SHOULD close the QUIC connection and connect to the traget server
directly instead.

Both the MASQUE negotiation initial request and its response carry a list of
type-length-value fields. The type field is a number corresponding to a MASQUE
application, and is encoded as a QUIC variable-length integer. The length field
represents the length in bytes of the value field, encoded as a QUIC
variable-length integer. The contents of the value field or defined by its
corresponding MASQUE application. When parsing, endpoints MUST ignore unknown
MASQUE applications.


# MASQUE Applications {#applications}

As soon as the server has accepted the client's MASQUE initial request, it
advertises support for certain MASQUE Applications. These application can be
multiplexed over this HTTP/3 connection or directly on top of different QUIC streams
within the same underlying QUIC connection. Applications can provide forwarding as well as an
direct communication channel between the client and the MASQUE server.

## HTTP Proxy {#http-proxy}

The client can make proxied HTTP requests through the server to other
servers. In practice this will mean using the CONNECT method to establish a
stream over which to run TLS to a different remote destination. The proxy
applies back-pressure to streams in both directions.


## DNS over HTTPS {#doh}

The client can send DNS queries using DNS over HTTPS {{!DOH=RFC8484}} to the
MASQUE server.


## QUIC Proxying {#quic-proxy}

By leveraging QUIC client connection IDs, a MASQUE server can act as a QUIC
proxy while only using one UDP port. The server informs the client of a
scheme for client connection IDs (for example, random of a minimum length or
vended by the MASQUE server) and then the server can forward those packets to
further web servers.

This mechanism can elide the connection IDs on the link between the client
and MASQUE server by negotiating a mapping between DATAGRAM_IDs and the tuple
(client connection ID, server connection ID, server IP address, server port).

Compared to UDP proxying, this mode has the advantage of only requiring one UDP
port to be open on the MASQUE server, and can lower the overhead on the link
between client and MASQUE server by compressing connection IDs.


## UDP Proxying {#udp-proxy}

In order to support WebRTC or QUIC to further servers, clients need a way to
relay UDP onwards to a remote server. In practice for most widely deployed
protocols other than DNS, this involves many datagrams over the same ports.
Therefore this mechanism implements that efficiently: clients can use the
MASQUE protocol stream to request an UDP association to an IP address and
UDP port pair. In QUIC, the server would reply with a DATAGRAM_ID that the
client can then use to have UDP datagrams sent to this remote server.
Datagrams are then simply transferred between the DATAGRAMs with this ID and
the outer server. There will also be a message on the MASQUE protocol stream
to request shutdown of a UDP association to save resources when it is no
longer needed.


## IP Proxying {#ip-proxy}

For the rare cases where the previous mechanisms are not sufficient, proxying
can be performed at the IP layer. This would use a different DATAGRAM_ID and
IP datagrams would be encoded inside it without framing.


## Service Registration {#service-registration}

MASQUE can be used to make a home server accessible on the wide area. The home
server authenticates to the MASQUE server and registers a domain name it wishes
to serve. The MASQUE server can then forward any traffic it receives for that
domain name (by inspecting the TLS Server Name Indication (SNI) extension) to
the home server. This received traffic is not authenticated and it allows
non-modified clients to communicate with the home server without knowing it is
not colocated with the MASQUE server.

To help obfuscate the home server, deployments can use Encrypted Server Name
Indication {{?ESNI=I-D.ietf-tls-esni}}. That will require the MASQUE server
sending the cleartext SNI to the home server.


# Security Considerations {#security}

Here be dragons. TODO: slay the dragons.


# IANA Considerations {#iana}

This document will request IANA to register the "/.well-known/masque/" URI
(expert review)
<https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml>.

This document will request IANA to create a new MASQUE Applications registry
which governs a 62-bit space of MASQUE application types.


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

This proposal was inspired directly or indirectly by prior work from many
people. The author would like to thank
Nick Harper,
Christian Huitema,
Marcus Ihlar,
Eric Kinnear,
Mirja Kuehlewind,
Brendan Moran,
Lucas Pardue,
Tommy Pauly,
Zaheduzzaman Sarker,
Ben Schwartz,
and
Christopher A. Wood
for their input.

