---
title: The MASQUE Proxy
docname: draft-schinazi-masque-proxy-latest
submissiontype: independent
number:
date:
v: 3
category: info
keyword:
  - masque
  - proxy
venue:
  repo: https://github.com/DavidSchinazi/masque-drafts
  latest: https://davidschinazi.github.io/masque-drafts/draft-schinazi-masque-proxy.html
author:
  -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    region: CA
    code: 94043
    country: United States of America
    email: dschinazi.ietf@gmail.com
informative:
  H3:
    =: RFC9114
    display: HTTP/3
  TODO:
    title: find that 20 year old email about using nested CONNECT tunnels with SSL to improve privacy
    date: false

--- abstract

MASQUE (Multiplexed Application Substrate over QUIC Encryption) is a set of
protocols and extensions to HTTP that allow proxying all kinds of Internet
traffic over HTTP. This document defines the concept of a "MASQUE Proxy", an
Internet-accessible node that can relay client traffic in order to provide
privacy guarantees.

--- middle

# Introduction

In the early days of HTTP, requests and responses weren't encrypted. In order
to add features such as caching, HTTP proxies were developed to parse HTTP
requests from clients and forward them on to other HTTP servers. As SSL/TLS
became more common, the CONNECT method was introduced {{?CONNECT=RFC2817}} to
allow proxying SSL/TLS over HTTP. That gave HTTP the ability to create tunnels
that allow proxying any TCP-based protocol. While non-TCP-based protocols were
always prevalent on the Internet, the large-scale deployment of QUIC
{{?QUIC=RFC9000}} meant that TCP no longer represented the majority of
Internet traffic. Simultaneously, the creation of HTTP/3 {{H3}} allowed
running HTTP over a non-TCP-based protocol. In particular, QUIC allows
disabling loss recovery {{?DGRAM=RFC9221}} and that can then be used in HTTP
{{?HTTP-DGRAM=RFC9297}}. This confluence of events created both the possibility
and the necessity for new proxying technologies in HTTP.

This led to the creation of MASQUE (Multiplexed Application Substrate over QUIC
Encryption). MASQUE allows proxying both UDP ({{?CONNECT-UDP=RFC9298}}) and IP
({{?CONNECT-IP=RFC9484}}) over HTTP. While MASQUE has uses beyond improving
user privacy, its focus and design are best suited for protecting sensitive
information.

# Privacy Protections

There are currently multiple usage scenarios that can benefit from using a
MASQUE Proxy.

## Protection from Web Servers

Connecting directly to Web servers allows them to access the public IP address
of the user. There are many privacy concerns relating to user IP addresses
{{?IP-PRIVACY=I-D.irtf-pearg-ip-address-privacy-considerations}}. Because of
these, many user agents would rather not establish a direct connection to
web servers. They can do that by running their traffic through a MASQUE
Proxy. The web server will only see the IP address of the MASQUE Proxy, not
that of the client.

## Protection from Network Providers

Some users may wish to obfuscate the destination of their network traffic
from their network provider. This prevents network providers from using data
harvested from this network traffic in ways the user did not intend.

## Partitioning

While routing traffic through a MASQUE proxy reduces the network provider's
ability to observe traffic, that information is transfered to the proxy
operator. This can be suitable for some threat models, but for the majority
of users transferring trust from their network provider to their proxy (or VPN)
provider is not a meaningful security improvement.

There is a technical solution that allows resolving this issue: it is possible
to nest MASQUE tunnels such that traffic flows through multiple MASQUE proxies.
This has the advantage of partitioning sensitive information to prevent
correlation {{?PARTITION=RFC9614}}.

Though the idea of nested tunnels dates back decades {{TODO}}, MASQUE now
allows running HTTP/3 end-to-end from a user agent to an origin via multiple
nested CONNECT-UDP tunnels. The proxy closest to the user can see the user's IP
address but not the origin, whereas the other proxy can see the origin without
knowing the user's IP address. If the two proxies are operated by non-colluding
entities, this allows hiding the user's IP address from the origin without the
proxies knowing the user's browsing history.

## Obfuscation

The fact that MASQUE is layered over HTTP makes it much more resilient to
detection. To network observers, the unencrypted bits in a QUIC connection
used for MASQUE are indistinguishable from those of a regular Web browsing
connection. Separately, if paired with a non-probeable HTTP authentication
scheme {{?SIGNATURE-AUTH=I-D.ietf-httpbis-unprompted-auth}}, any Web server
can also become a MASQUE proxy while remaining indistinguishable from a
regular Web server. It might still be possible to detect some level of
MASQUE usage by analyzing encrypted traffic patterns, however the cost of
performing such an analysis at scale makes it impractical.

This allows MASQUE to operate on networks that disallow VPNs by using a
combination of protocol detection and blocklists.

# Related Technologies

This section discusses how MASQUE fits in with other contemporary
privacy-focused IETF protocols.

## OHTTP

Oblivious HTTP {{?OHTTP=RFC9458}} uses a cryptographic primitive
{{?HPKE=RFC9180}} that is more lightweight than TLS {{?TLS=RFC8446}}, making it
a great fit for decorrelating HTTP requests. In traditional Web browsing, the
user agent will often make many requests to the same origin (e.g., to load
HTML, style sheets, images, scripts) and those requests are correlatable since
the origin can include identifying query parameters to join separate requests.
In such scenarios, MASQUE is a better fit since it operates at the granularity
of a connection. However, there are scenarios where a user agent might want to
make non-correlatable requests (e.g., to anonymously report telemetry); for
those, OHTTP provides better efficiency than using MASQUE with a separate
connection per request. While OHTTP and MASQUE are separate technologies that
serve different use cases, they can be colocated on the same HTTP server that
acts as both a MASQUE Proxy and an OHTTP Relay.

## DoH

DNS over HTTPS {{?DoH=RFC8484}} allows encrypting DNS traffic by sending it
through an encrypted HTTP connection. Colocating a DoH server with a MASQUE
IP proxy provides better performance than using DNS over port 53 inside the
encrypted tunnel.

# Security Considerations

Implementers of a MASQUE proxy need to review the Security Considerations of
the documents referenced by this one.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

MASQUE was originally inspired directly or indirectly by prior work from many
people. The author would like to thank {{{Nick Harper}}},
{{{Christian Huitema}}}, {{{Marcus Ihlar}}}, {{{Eric Kinnear}}},
{{{Mirja Kuehlewind}}}, {{{Brendan Moran}}}, {{{Lucas Pardue}}},
{{{Tommy Pauly}}}, {{{Zaheduzzaman Sarker}}} and {{{Ben Schwartz}}} for their
input.

In particular, the probing resistance component of MASQUE came from a
conversation with {{{Chris A. Wood}}} as we were preparing a draft for an
upcoming Thursday evening BoF.

All of the MASQUE enthusiasts and other contributors to the MASQUE working
group are to thank for the successful standardization of {{HTTP-DGRAM}},
{{CONNECT-UDP}}, and {{CONNECT-IP}}.

The author would like to express immense gratitude to Christophe A., an
inspiration and true leader of VPNs.
