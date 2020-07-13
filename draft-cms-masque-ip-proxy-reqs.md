---
title: Requirements for a MASQUE Protocol to Proxy IP Traffic
abbrev: IP Proxying Requirements
docname: draft-cms-masque-ip-proxy-reqs-latest
category: info

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
There is interest among MASQUE working group participants in designing a
protocol that can proxy IP traffic over HTTP. This document describes the set
of requirements for such a protocol.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing
list [](masque@ietf.org) or on the GitHub repository which contains the draft:
[](https://github.com/DavidSchinazi/masque-drafts).


--- middle

# Introduction
There exist several IETF standards for proxying IP in a way that is
authenticated and confidential, such as IKEv2/IPsec {{?IKEV2=RFC7296}}.
However, those are distinguishable from common Internet traffic and often
blocked. Additionally, large server deployments have expressed interest in
using a VPN solution that leverages existing security protocols such as QUIC
{{!QUIC=I-D.ietf-quic-transport}} or TLS {{!TLS=RFC8446}} to avoid adding
another protocol to their security posture.

This document describes the set of requirements for a protocol that can proxy
IP traffic over HTTP. The requirements outlined below are similar to the
considerations made in designing the CONNECT-UDP method
{{?CONNECT-UDP=I-D.schinazi-masque-connect-udp}}, additionally including
IP-specific requirements, such as a means of negotiating the routes that should
be advertised on either end of the connection.

Discussion of this work is encouraged to happen on the MASQUE IETF mailing list
[](masque@ietf.org) or on the GitHub repository which contains the draft:
[](https://github.com/DavidSchinazi/masque-drafts).

## Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

## Definitions

* Data Transport: The method by which IP packets are transmitted. This can
  involve streams or datagrams.

* IP Session: An association between client and server whereby both agree to
  proxy IP traffic given certain configuration properties. This is similar to a
  Child Security Association in IKEv2 terminology.

# Use Cases

There are multiple reasons to deploy an IP proxying protocol. This section
discusses some examples of use cases that MUST be supported by the protocol.

## Point to Point Connectivity

Point-to-point connectivity creates a private, encrypted and authenticated
network between two IP addresses. This is useful, for example, with container
networking to provide a virtual (overlay) network with addressing separate from
the physical transport. An example of this is Wireguard.

## Point to Network Connectivity

Point-to-Network connectivity is the more traditional remote-access "VPN" use
case, frequently used when a user needs to connect to a different network (such
as an enterprise network) for access to resources that are not exposed to the
public Internet.

## Network to Network Connectivity

Network-to-Network connectivity is also called a site-to-site VPN. Like the
point-to-network use case, the goal is to connect to a network that is not
exposed publicly. The site-to-site aspects make this transparent to the user;
the entire networks are connected to each other and route packets transparently
without a VPN client installed on the user's device. This style of connectivity
can also be used to connect devices that cannot run VPN clients through to the
network.

# Requirements

This section lists requirements for a protocol that can proxy IP over an HTTP
connection.

## IP Session Establishment

The protocol will allow the client to request establishment of an IP Session,
along with configuration options and one or more associated Data Transports.
The server will have the ability to accept or deny the client's request.

## Proxying of IP packets

The protocol will establish Data Transports, which will be able to forward IP
packets, in their unmodified entirety. The protocol will support both IPv6
{{!IPV6=RFC8200}} and IPv4 {{!IPV4=RFC0791}}.

## Maximum Transmission Unit

The protocol will allow endpoints to negotiate the Maximum Transmission Unit
(MTU) in use over a given Data Transport. This will allow avoiding IP
fragmentation, especially as IPv6 does not allow IP fragmentation by nodes
along the path.

## IP Assignment

Both the client or server may request to be assigned an IP address range. In
response to the request, the peer will respond with an IP address range of its
choosing.

## Route Negotiation

At any point in an IP Session (not limited to its initial negotiation), the
protocol will allow both client and server to request routes to specific IP
address ranges. In response to this request, the peer will have the ability to
respond with a subset of routes that it is willing to accept, or deny the
request.

## Identity

When negotiating the creation of an IP Session, the protocol will allow both
endpoints to exchange an identifier. For example, both endpoints will be able
to identify themselves by sending a fully-qualified domain name.

## Transport Security

The protocol MUST be run over a protocol that provides mutual authentication,
confidentiality and integrity. Using QUIC or TLS would meet this requirement.

## Authentication

Additionally to the authentication provided by the transport, the protocol will
have the ability to authenticate both client and server during the
establishment of the IP Session. In particular, it will be possible for the
client to offer an OAuth Access Token {{?OAUTH=RFC6749}} to the server when
requesting IP proxying, potentially through an extension of the protocol. The
protocol will also have the ability to support vendor-specific authentication
mechanisms as extensions.

## Reliable Transmission of IP Packets

While it is desirable to transmit IP packets unreliably in most cases, the
protocol will provide a mechanism to allow forwarding some packets reliably.
For example, when using HTTP/3, this can be accomplished by allowing Data
Transports to run over both DATAGRAM and STREAM frames.

## Flow Control

The protocol will allow the ability to proxy IP packets without flow control,
at least when HTTP/3 is in use. QUIC DATAGRAM frames are not flow controlled
and would meet this requirement. The document defining the protocol will
provide guidance on how best to use flow control to improve IP Session
performance.

## Indistinguishability

A passive network observer not participating in the encrypted connection should
not be able to distinguish an IP proxying session from regular encrypted HTTP
Web traffic.

## Support HTTP/2 and HTTP/3

The IP proxying protocol discussed in this document will run over HTTP. The
protocol SHOULD strongly prefer to use HTTP/3 {{!H3=I-D.ietf-quic-http}} and
SHOULD use the QUIC DATAGRAM frames {{!DGRAM=I-D.ietf-quic-datagram}} when
available to improve performance. The protocol SHOULD also support HTTP/2
{{!H2=RFC7540}} as a fallback when UDP is blocked on the network path. Proxying
IP over HTTP/2 MAY result in lower performance than over HTTP/3.

## Multiplexing

Since recent HTTP versions support concurrently running multiple requests over
the same connection, the protocol SHOULD support multiple independent instances
of IP proxying over a given HTTP connection.

## Load balancing

Clients and servers should each be able to instantiate new Data Transports.
This facilitates multi-threaded servers being able to handle a higher bandwidth
of IP proxied packets.

The IP proxying mechanisms need to support load balancing of the traffic sent
across the session, such as to another server. The document defining the new
protocol should provide guidance for when additional connections and/or
sessions should be opened, as opposed to reusing existing ones.

## Extensibility

The protocol will provide a mechanism by which clients and servers can add
extension information to the exchange that establishes the IP session. If the
solution uses an HTTP request and response, this could be accomplished using
HTTP headers.

Once the session is established, the protocol will provide a mechanism that
allows reliably exchanging vendor-specific messages in both directions at any
point in the lifetime of the IP Session.

# Non-requirements

This section discusses topics that are explicitly out of scope for the IP
Proxying protocol. These topics MAY be handled by implementers or future
extensions.

## Addressing Architecture

This document only describes the requirements for a protocol that allows IP
proxying. It does not discuss how the IPs assigned are determined, managed, or
translated. While these details are important for producing a functional
system, they do not need to be handled by the protocol beyond the ability to
convey those assignments.

## Translation

Some servers may wish to perform Network Address Translation (NAT) or any other
modification to packets they forward. Doing so is out of scope for the proxying
protocol. In particular, the ability to discover the presence of a NAT,
negotiate NAT bindings, or check connectivity through a NAT is explicitly out
of scope and left to future extensions.

## IP Packet Extraction

How packets are forwarded between the IP proxying connection and the physical
network is out of scope. This is deliberately not specified and will be left to
individual implementations.

# Security Considerations

This document only discusses requirements on a protocol that allows IP
proxying. That protocol will need to document its security considerations.

# IANA Considerations

This document requests no actions from IANA.

# Acknowledgments
{:numbered="false"}

The authors would like to thank participants of the MASQUE working group for
their feedback.
