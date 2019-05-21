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

normative:
  RFC2119:
  RFC5705:
  RFC7540:
  RFC8174:
  RFC8446:
  RFC8484:
  I-D.ietf-quic-http:
  I-D.ietf-quic-transport:
  I-D.pauly-quic-datagram:

informative:
  RFC7427:
  RFC8441:
  RFC8471:
  I-D.ietf-httpbis-http2-secondary-certs:
  I-D.pardue-httpbis-http-network-tunnelling:
  I-D.schwartz-httpbis-helium:
  I-D.sullivan-tls-post-handshake-auth:


--- abstract

This document describes MASQUE (Multiplexed Application Substrate over QUIC
Encryption). MASQUE is a mechanism that allows co-locating and obfuscating
networking applications behind an HTTPS web server. The currently prevalent
use-case is to allow running a VPN server that is indistinguishable from an
HTTPS server to any unauthenticated observer. We do not expect major providers
and CDNs to deploy this behind their main TLS certificate, as they are not
willing to take the risk of getting blocked, as shown when domain fronting
was blocked. An expected use would be for individuals to enable this behind
their personal websites via easy to configure open-source software.

This document is a straw-man proposal. It does not contain enough details to
implement the protocol, and is currently intended to spark discussions on
the approach it is taking. As we have not yet found a home for this work,
discussion is encouraged to happen on the GitHub repository which contains
the draft: <https://github.com/DavidSchinazi/masque-drafts>.


--- middle

# Introduction

This document describes MASQUE (Multiplexed Application Substrate over QUIC
Encryption). MASQUE is a mechanism that allows co-locating and obfuscating
networking applications behind an HTTPS web server. The currently prevalent
use-case is to allow running a VPN server that is indistinguishable from an
HTTPS server to any unauthenticated observer. We do not expect major providers
and CDNs to deploy this behind their main TLS certificate, as they are not
willing to take the risk of getting blocked, as shown when domain fronting
was blocked. An expected use would be for individuals to enable this behind
their personal websites via easy to configure open-source software.

*DISCUSS*: do we want to provide a VPN service, or just an HTTP/QUIC
service? VPN services requires interacting with the network layer,
providing spearate addresses to clients, etc. This is much harder to
deploy than a proxy service for which the proxy just needs a single
port.

This document is a straw-man proposal. It does not contain enough details to
implement the protocol, and is currently intended to spark discussions on
the approach it is taking. As we have not yet found a home for this work,
discussion is encouraged to happen on the GitHub repository which contains
the draft: <https://github.com/DavidSchinazi/masque-drafts>.

MASQUE leverages the efficient head-of-line blocking prevention features of
the QUIC transport protocol {{I-D.ietf-quic-transport}} when MASQUE is used
in an HTTP/3 {{I-D.ietf-quic-http}} server. MASQUE can also run in an
HTTP/2 server {{RFC7540}} but at a performance cost.


## Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Scenarios

There are three important scenarios for deploying a QUIC proxy: the home server
scenario, the hidden client scenario, and the onion scenario.

## Home Server Scenario

It is often difficult to connect to a home server. The IP address might
change over time. Firewalls in the home router or in the network may block
incoming connections. Deploying a "proxy in the cloud" helps resolve
these issues.

In the simplest instatiation of the scenario, the home server establishes
QUIC connection with proxy server "in the cloud", and the DNS records of the
homeserver point to address of the proxy server in the cloud. The clients
of the home server will send their QUIC messages to the proxy. The proxy
will identify the final destination of the messages by looking at
the SNI in Initial packets, or the Destination CID in handshake and short packets.
The proxy forwards these packets to the QUIC server as "datagram".
The home server treat packets as if received on UDP socket.
The home server forwards its own packets to the proxy as datagram.
The proxy relays QUIC packets to the client.

Optionally, the QUIC server may use the "preferred address" mechanism to suggest
migration to a direct connection, bypassing the proxy.

Optionally, local clients may discover the local server, without using the proxy.

## Hidden client scenario

There are many clients who would rather not establish a direct connection to
web servers, for example to avoid location tracking. The clients can do that
for their QUIC connection by running through a proxy server. The web server
will only see the IP address of the proxy, not that of the client.

In a simple instantioation of the scenario, the client establishes a QUIC connection
with the proxy. The client sends QUIC messages to the proxy as datagrams.
The proxy decapsulates the messages, sends them to web server.
The web server replies to proxy. The proxy uses the destination CID to
determines which client should process the packet, and forwards QUIC messages
as Datagrams on the client-proxy connection.

## Onion scenario

The hidden client scenario only provides partial connection against tracking,
because the proxy knows the address of the client. Onion routing solves that
issue. Onion routing is available for TCP/TLS, but the proxy design should
make it available for QUIC as well.

In the onion scenario, the client establishes a connection to proxy, then through
that to another proxy, etc. This creates a tree of proxies rooted at the client.
QUIC connections are mapped to a specific branch of the tree.
The first proxy knows the actual address of the client,
but the other proxies only know the address of the previous proxy.
To assure reasonable privacy, the path should include at least 3 proxies.

Hidden server can similarly hide between several layers of proxy. Hidden servers
should not publish their address in the DNS. They may use an Onion DHT service
instead (see Tor ".onion"), or in fact any other mechanism.
This is out of scope in this spec.

# Requirements

This section describes the goals and requirements chosen for the MASQUE
protocol.


## Invisibility of VPN Usage

An authenticated client using the VPN appears to observers as a regular HTTPS
client. Observers only see that HTTP/3 or HTTP/2 is being used over an
encrypted channel. No part of the exchanges between client and server may
stick out. Note that traffic analysis is currently considered out of scope.


## Invisibility of the Server

To anyone without private keys, the server is indistinguishable from a regular
web server. It is impossible to send an unauthenticated probe that the server
would reply to differently than if it were a normal web server.


## Fallback to HTTP/2 over TLS over TCP

When QUIC is blocked, MASQUE can run over TCP and still satisfy
previous requirements. Note that in this scenario performance may be
negatively impacted.

## Proxy and client authentication

QUIC relies on TLS, the client or the home server will normally authenticate the proxy by verifying the proxy's certificate.
In the home server and hidden client scenarios, the proxy may also want to verify the server's identity before providing service.
In the onion scenario,the client probably wants to remain anonymous.

There are two classes of reasons for requiring authentication of the proxy's client: legal and economics.

In some jurisdictions, a proxy may be held responsible if the home server publishes "illegal" content or the hidden client accesses it.
This reasoning leads many Wi-Fi network to require some explicit consent before allowing a client to connect.
In free countries, it is enough for the client to check a box acknowledging that it won't do anything illegal.
In authoritarian countries, Wi-Fi providers are supposed to verify and log the client's identity.
We might expect proxy services to have the same type of legal requirements, but to require more automation.
The "consent" form require human intervention, which is hard to automate. Verifying a certificate is easier.

There is also an economic access to proxy provision. Providing robust proxy services to multiple clients may be costly.
Proxies probably need some kind of business model. Some may use a "subscription" model, tied to client authentication.
They will authenticate the client and verify that subscription fees have been paid before providing service.

The subscription business model may not be adequate in the "onion" scenario, which aim to hide clients' identities.
We might explore "temporary certificates" that prove that the client is authorized to use the service without disclosing the identity.
We might also explore some kind of micropayment system. The client would get anonymous "coins" from a service manager.
The proxies could collect the coins, and redeem them to pay for the service.
The agreement to "not do naything illegal" could be verified by the service manager before prividing coins.
The whole micropayment architecture is probably out of scope in the proxy design, but there should be an API to enable it. 

## Single proxy address

A proxy should not require more network access than a regular server.
A typical server only listens on a single UDP port, typically 443.
Proxy services should not require listening to additional UDP ports, or
opening additional ports in the local firewall.

## SNI routing and hidden SNI

In the "home server" scenario, the proxy will relay connection requests based on the server SNI.
The protocol should allow configuration of that SNI, typically after verifying the home server's identity.

The proxy should support ESNI. This is very much the same scenario as a server farm or CDN supporting ESNI.

Supporting ESNI adds constraints in the forwarding of incoming connections.
The proxy can decrypt the ESNI, but the home server cannot. The server will need some of the ESNI data.
The proxy must forward them to the client.

## Destination CID

The proxy will receive packets in which the destination is identified by the DCID.

In standard code, the DCID is chosen independently by each client, and some clients
may not be using any DCID at all. The proxy could observe collisions, or be
unable to determine which client should receive a packet.

To ensure routability, the clients should synchronize their use of CID with the
proxy. This may require stating a minimum length, imposing some form of randomness,
or even getting the proxy to propose CID values.


# Overview of the Mechanism

The server runs an HTTPS server on port 443, and has a valid TLS certificate
for its domain. The client has a public/private key pair, and the server
maintains a list of authorized MASQUE clients, and their public key.
(Alternatively, clients can also be authenticated using a shared secret.)
The client starts by establishing a regular HTTPS connection to the server
(HTTP/3 over QUIC or HTTP/2 over TLS 1.3 {{RFC8446}} over TCP), and validates
the server's TLS certificate as it normally would for HTTPS. If validation
fails, the connection is aborted. The client then uses a TLS keying material
exporter {{RFC5705}} with label "EXPORTER-masque" and no context to generate a
32-byte key. This key is then used as a nonce to prevent replay attacks. The
client then sends an HTTP CONNECT request for "/.well-known/masque/initial"
with the :protocol pseudo-header field set to "masque", and a
"Masque-Authentication:" header. The MASQUE authentication header differs
from the HTTP "Authorization" header in that it applies to the underlying
connection instead of being per-request. It can use either a shared secret
or asymmetric authentication. The asymmetric variant uses authentication
method "PublicKey", and it transmits a signature of the nonce with the
client's public key encoded in base64 format, followed by other information
such as the client username and signature algorithm OID. The symmetric
variant uses authentication method "HMAC" and transmits an HMAC of the
nonce with the shared secret instead of a signature. For example this header
could look like:

~~~
Masque-Authentication: PublicKey u="am9obi5kb2U=";a=1.3.101.112;
s="SW5zZXJ0IHNpZ25hdHVyZSBvZiBub25jZSBoZXJlIHdo
aWNoIHRha2VzIDUxMiBiaXRzIGZvciBFZDI1NTE5IQ=="

Masque-Authentication: HMAC u="am9obi5kb2U=";a=2.16.840.1.101.3.4.2.3;
s="SW5zZXJ0IHNpZ25hdHVyZSBvZiBub25jZSBoZXJlIHdo
aWNoIHRha2VzIDUxMiBiaXRzIGZvciBFZDI1NTE5IQ=="
~~~
{: #auth-format title="MASQUE Authentication Format Example"}

When the server receives this CONNECT request, it verifies the signature and
if that fails responds with code "405 Method Not Allowed", making sure its
response is the same as what it would return for any unexpected CONNECT
request. If the signature verifies, the server responds with code
"101 Switching Protocols", and from then on this HTTP stream is now dedicated
to the MASQUE protocol. That protocol provides a reliable bidirectional
message exchange mechanism, which is used by the client and server to
negotiate what protocol options are supported and enabled by policy, and
client VPN configuration such as IP addresses. When using QUIC, this protocol
also allows endpoints to negotiate the use of QUIC extensions, such as support
for the DATAGRAM extension {{I-D.pauly-quic-datagram}}.


# Mechanisms the Server Can Advertise to Authenticated Clients

Once a server has authenticated the client's MASQUE CONNECT request,
it advertises services that the client may use. These services allow
for example varying degrees of proxying services to help a client
obfuscate the ultimate destination of their traffic.


## HTTP Proxy

The client can make proxied HTTP requests through the server to other
servers. In practice this will mean using the CONNECT method to establish a
stream over which to run TLS to a different remote destination.


## DNS over HTTPS

The client can send DNS queries using DNS over HTTPS (DoH) {{RFC8484}}
to the MASQUE server.


## UDP Proxying

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
longer needed. When running over TCP, the client opens a new stream with a
CONNECT request to the "masque-udp-proxy" protocol and then sends datagrams
encapsulated inside the stream with a two-byte length prefix in network byte
order. The target IP and port are sent as part of the URL query. Resetting
that stream instructs the server to release any associates resources.


## IP Proxying

For the rare cases where the previous mechanisms are not sufficient, proxying
can be performed at the IP layer. This would use a different DATAGRAM_ID and
IP datagrams would be encoded inside it without framing. Over TCP, a dedicated
stream with two byte length prefix would be used. The server can inspect the
IP datagram to look for the destination address in the IP header.


## Path MTU Discovery

In the main deployment of this mechanism, QUIC will be used between client
and server, and that will most likely be the smallest MTU link in the path
due to QUIC header and authentication tag overhead. The client is responsible
for not sending overly large UDP packets and notifying the server of the low
MTU. Therefore PMTUD is currently seen as out of scope of this document.


# Security Considerations

Here be dragons. TODO: slay the dragons.


# IANA Considerations

We will need to register:

* the TLS keying material exporter label "EXPORTER-masque" (spec required)
<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#exporter-labels>

* the new HTTP header "Masque-Authentication"
<https://www.iana.org/assignments/message-headers/message-headers.xhtml>

* the "/.well-known/masque/" URI (expert review)
<https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml>

* The "masque" and "masque-udp-proxy" extended HTTP CONNECT protocols

We will also need to define the MASQUE control protocol and that will be
likely to define new registries of its own.


--- back

# Acknowledgments
{:numbered="false"}

This proposal was inspired directly or indirectly by prior work from many
people. In particular, this work is related to
{{I-D.schwartz-httpbis-helium}} and
{{I-D.pardue-httpbis-http-network-tunnelling}}. The mechanism used to
run the MASQUE protocol over HTTP/2 streams was inspired by {{RFC8441}}.
Using the OID for the signature algorithm was inspired by Signature
Authentication in IKEv2 {{RFC7427}}.

The author would like to thank Christophe A., an inspiration and true leader
of VPNs.


# Design Justifications
{:numbered="false"}

Using an exported key as a nonce allows us to prevent replay attacks (since it
depends on randomness from both endpoints of the TLS connection) without
requiring the server to send an explicit nonce before it has authenticated the
client. Adding an explicit nonce mechanism would expose the server as it would
need to send these nonces to clients that have not been authenticated yet.

The rationale for a separate MASQUE protocol stream is to allow
server-initiated messages. If we were to use HTTP semantics, we would only be
able to support the client-initiated request-response model. We could have used
WebSocket for this purpose but that would have added wire overhead and
dependencies without providing useful features.

There are many other ways to authenticate HTTP, however the authentication
used here needs to work in a single client-initiated message to meet the
requirement of not exposing the server.

The current proposal would also work with TLS 1.2, but in that case TLS false
start and renegotiation must be disabled, and the extended master secret and
renegotiation indication TLS extensions must be enabled.

If the server or client want to hide that HTTP/2 is used, the client can set
its ALPN to an older version of HTTP and then use the Upgrade header to
upgrade to HTTP/2 inside the TLS encryption.

The client authentication used here is similar to how Token Binding {{RFC8471}}
operates, but it has very different goals. MASQUE does not use token binding
directly because using token binding requires sending the token_binding TLS
extension in the TLS ClientHello, and that would stick out compared to a
regular TLS connection.

TLS post-handshake authentication {{I-D.sullivan-tls-post-handshake-auth}}
is not used by this proposal because that requires sending the
"post_handshake_auth" extension in the TLS ClientHello, and that would stick
out from a regular HTTPS connection.

Client authentication could have benefited from Secondary Certificate
Authentication in HTTP/2 {{I-D.ietf-httpbis-http2-secondary-certs}},
however that has two downsides: it requires the server advertising that
it supports it in its SETTINGS, and it cannot be sent unprompted by the
client, so the server would have to request authentication.
Both of these would make the server stick out from regular HTTP/2 servers.

MASQUE proposes a new client authentication method (as opposed to reusing
something like HTTP basic authentication) because HTTP authentication methods
are conceptually per-request (they need to be repeated on each request)
whereas the new method is bound to the underlying connection (be it QUIC or
TLS). In particular, this allows sending QUIC DATAGRAM frames without
authenticating every frame individually. Additionally, HMAC and asymmetric
keying are preferred to sending a password for client authentication since
they have a tighter security bound. Going into the design rationale, HMACs
(and signatures) need some data to sign, and to avoid replay attacks that
should be a fresh nonce provided by the remote peer. Having the server
provide an explicit nonce would leak the existence of the server so we use
TLS keying material exporters as they provide us with a nonce that contains
entropy from the server without requiring explicit communication.

