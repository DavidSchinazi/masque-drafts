---
title: A Flow Forwarding Mode Extension to CONNECT-IP
abbrev: CONNECT-IP Flow Forwarding
docname: draft-tbd-masque-connect-ip-ext-flow-latest
category: std
wg: MASQUE

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "T. B. Determined"
    name: "To Be Determined"
    organization: "MASQUE Enthusiasts LLC"
    email: dschinazi.ietf@gmail.com


--- abstract

This document describes an extension to the CONNECT-IP HTTP method. This
extension defines a new Flow Forwarding Mode which supports optimization for
individual IP flows forwarded to the targeted peer.

* IMPORTANT NOTE: This draft exists only to demonstrate the feasibility of
  defining CONNECT-IP Flow Forwarding Mode as an extension to CONNECT-IP. The
  overwhelming majority of the content in this draft was copied from
  draft-kuehlewind-masque-connect-ip-01. A list of changes from that design is
  available in an appendix to this document. All credit for flow forwarding
  should go to the authors of that document. However, all blame for any errors
  in this draft should be attributed to its editor. The editor's intent is to
  step down as editor of this draft and have the authors of
  draft-kuehlewind-masque-connect-ip take on that role.


--- middle

# Introduction

This document describes an extension to the CONNECT-IP HTTP method
{{!CONNECT-IP=I-D.cms-masque-connect-ip}}. This extension defines a new Flow
Forwarding Mode which supports optimization for individual IP flows forwarded
to the targeted peer.

In flow forwarding mode the CONNECT-IP method establishes an outgoing IP flow,
from the MASQUE server's external address to the target server's address
specified by the client for a particular upper layer protocol. This mode also
enables reception and relaying of the reverse IP flow from the target address
to the MASQUE server to ensure that return traffic can be received by the
client. However, it does not support flow establishment by an external peer.
This specification supports forwarding of incoming traffic to one of the
clients only if an active mapping has previously been created based on an
CONNECT-IP request. Clients that need to support reception of flows established
by external peer need to use regular CONNECT-IP.

This mode allows for flow-based optimizations and a larger effective maximum
packet size through the tunnel. The target IP address is provided by the client
as part of the CONNECT-IP request. The source address is either independently
selected by the proxy or can be requested to be either the same as used in a
previous and currently active CONNECT-IP request or different from currently
requests by the same client. The client also indicates the upper layer
protocol, thus defining the three tuple used as primary selector for the flow.

In this mode the payload between the client and proxy does not contain the IP
header in order to reduce overhead. Any additional information (other than the
source and destination IP addresses and ports as well as the upper layer
protocol identifier) that is needed to construct the IP header or to inform the
client about information from received IP packets can be signalled as part of
the CONNECT-IP request or using HTTP/3 Datagram
{{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}} later.

In flow forwarding mode, usually one upper-layer end-to-end connection is
associated to one CONNECT-IP forwarding association. While it would be possible
for a client to use the same forwarding association for multiple end-to-end
connections to the same target server, as long as they all require the same
Protocol (IPv4) / Next Header (IPv6) value, this would lead to the use of the
same flow ID for all connections. As such, this is not recommended for
connection-oriented transmissions. In order to enable multiple flow forwarding
associations to the same server, the flow forwarding mode supports a way to
specify some additional upper layer protocol selectors, e.g. TCP source and
destination port, to enable multiple CONNECT-IP request for the same three
tuple, see CONN-ID header ({{conn-id}}).

The default model for address handling in this specification is that the proxy
(MASQUE Server) will have a pool of one or more IP addresses that it can lend
to the MASQUE client and routable over its external interface. Other potential
use cases and address handling are possible, potentially requiring further
extensions.

This proposal is based on the analysis provided in
{{?I-D.westerlund-masque-transport-issues}} indicating that most
information in the IP header is either IP flow related or can or even
should be provided by the proxy as the IP communication endpoint
without the need for input from the client.  The most crucial
information identified that requires client interaction is ECN
{{!RFC3168}} and ICMP {{!RFC0792}} {{!RFC4443}} handling.

This document defines the following IP header field treatment.

Required to be determined in CONNECT-IP request and response:

*  IP version

*  IP Source Address

*  IP Destination Address (target address)

*  Upper Layer Protocol (IPv4 Protocol field / IPv6 Next Header field)

Can be chosen by Proxy on transmission:

*  IPv6 Flow label (per CONNECT-IP flow mode request)

*  IPv4 Time to live / IPv6 Hop Limit (proxy configured)

*  Diffserv Codepoint, default is set to 0 (Best Effort)

May optionally be provided on a per packet basis

*  Explicit Congestion Notification in both directions.

The consequence of this is certain limitations that future extension can
address. For packets that are sent from the target server to the client, the
client will not get any information on the actual value of TTL/Hop Count, DSCP,
or flow label when received by the proxy. Instead these field are set and
consumed by the proxy only.

Signalling of other dedicated values may be desired in certain deployments, e.g
for DCSP {{?RFC2474}}. However, DSCP is in any case a challenge due to local
domain dependency of the used DSCP values and the forwarding behavior and
traffic treatment they represent. Future use cases for DSCP, as well as new
IPv6 extension headers or destination header options {{!RFC8200}} may require
additional signaling. Therefore, it is important that the signaling is
extensible.

## Motivation of IP flow model for flow forwarding

The chosen IP flow model is selected due to several advantages:

*  Minimized per packet overhead: The per packet overhead is reduced
  to basic framing of the IP payload for each IP packet and flow
  identifiers.  This enables a larger effective Maximum Transmission
  Unit (MTU) than regular CONNECT-IP.

*  Shared functionality with CONNECT-UDP: The UDP flow proxying
  functionality of CONNECT-UDP will need to establish, store and
  process the same IP header related fields and state.  So this can
  be accomplished by simply removing the UDP specific processing of
  packets.

*  CONNECT-IP can establish a new IP flow in 0-RTT: No network
  related latencies in establishing new flow.

Disadvantages of this model are the following:

*  Client to server focused solution: Accepting non-solicited peer-
  initiated traffic is not supported.

## Definitions

*  Proxy: This document uses proxy as synonym for the MASQUE Server
  or an HTTP proxy, depending on context.

*  Client: The endpoint initiating a MASQUE tunnel and IP relaying
  with the proxy.

*  Target host: A remote endpoint the client wishes to establish bi-
  directional communication with via tunnelling over the proxy.

*  IP proxying: A proxy forwarding IP payloads to a target for an IP
  flow.  Data is decapsulate at the proxy and amended by a IP header
  before forwarding to the target.  Packet boundaries need to be
  preserved or signalled between the client and proxy.

*  IP flow: A flow of IP packets between two hosts as identified by
  their IP addresses, and where all the packets share some
  properties.  These properties include source/destination address,
  protocol / next header field, flow label (IPv6 only), and DSCP per
  direction.

*  Address = IP address

~~~
                     Target Address --+
                                       \
+--------+           +--------+         \ +--------+
|        |  Path #1  |        | Path #2  V|        |
| Client |<--------->|  Proxy |<--------->| Target |
|        |          ^|        |^          |        |
+--------+         / +--------+ \         +--------+
                  /              \
                 /                +-- Proxy's external address
                /
               +-- Proxy's service address
~~~
{: #fig-nodes title="The nodes and their addresses"}

{{fig-nodes}} provides an overview figure of the involved nodes, i.e. client,
proxy, and target host. There are also two network paths. Path #1 is the client
to proxy path, where IP proxying is provided over an HTTP/3 session, usually
over QUIC, to tunnel IP flow(s). Path #2 is the path between the proxy and the
target.

The client will use the proxy's service address to establish a transport
connection on which to request IP proxying using HTTP/3 CONNECT-IP. The proxy
will then relay the client's IP flows to the target host. The IP header from
the proxy to the target carries the proxy's external address as source address
and the target's address as destination address.

## Conventions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Negotiating Flow Forwarding Mode

To request flow forwarding, the client sends a CONNECT-IP request to the
forwarding proxy indicating the target host and port in the Flow-Forwarding
header field ({{ffd}}). The host portion is either an IP literal encapsulated
within square brackets, an IPv4 address in dotted-decimal form, or a registered
name. Further the CONNECT-IP request MUST contain the IP-Protocol header
({{ip-proto}}) and MAY contain the IP-Address-Handling ({{ip-addr-handling}})
or the Conn-ID ({{conn-id}}) header.

If the server accepts the request, it responds with a 2xx (Successful) response
that echoes the Flow-Forwarding header field ({{ffd}}). The client can
optimistically register its IP payload context ID ({{payload-capsule}}) and
start sending IP payloads ({{payload-format}}) before it receives the server
response. If the server rejects the request or does not support flow forwarding
mode, those datagrams will be dropped.


# HTTP Datagram Multiplexing and Encoding

Flow forwarding mode requires multiplexing multiple types of HTTP Datagrams
over a single CONNECT-IP request: IP payloads and ICMP messages. To allow this,
clients register HTTP Datagram Context IDs using capsules {{HTTP-DGRAM}}.

## IP Payloads

In order to exchange IP payloads, the client starts by using its context ID
allocation service {{HTTP-DGRAM}} to allocate a context ID. It then
communicates that context ID to the server using a
REGISTER_DATAGRAM_CONTEXT_IP_PAYLOAD capsule ({{payload-capsule}}). Once that
is done, both endpoints can exchange IP Payloads ({{payload-format}}).

### IP Payload Capsule {#payload-capsule}

The REGISTER_DATAGRAM_CONTEXT_IP_PAYLOAD capsule (type=0xfff801) allows an
endpoint to inform its peer of the encoding and semantics of IP Payload
datagrams associated with a given context ID. Its Capsule Data field consists
of:

~~~
REGISTER_DATAGRAM_CONTEXT_IP_PAYLOAD Capsule {
  Context ID (i),
}
~~~
{: #payload-capsule-format title="REGISTER_DATAGRAM_CONTEXT_IP_PAYLOAD Capsule Format"}

Context ID:

: The context ID to register.

### IP Payload HTTP Datagram Format {#payload-format}

HTTP Datagrams registered to carry IP Payloads contain the IP payload. This is
defined as the payload following the IPv4 header and any options for IPv4, and
for IPv6 as the payload following the IPv6 header and any extension header.

## ICMP

In order to exchange ICMP messages, the client starts by using its context ID
allocation service {{HTTP-DGRAM}} to allocate a context ID. It then
communicates that context ID to the server using a
REGISTER_DATAGRAM_CONTEXT_ICMP capsule ({{icmp-capsule}}). Once that is done,
both endpoints can exchange ICMP messages ({{icmp-format}}).

### ICMP Capsule {#icmp-capsule}

The REGISTER_DATAGRAM_CONTEXT_ICMP capsule (type=0xfff802) allows an endpoint
to inform its peer of the encoding and semantics of ICMP datagrams associated
with a given context ID. Its Capsule Data field consists of:

~~~
REGISTER_DATAGRAM_CONTEXT_ICMP Capsule {
  Context ID (i),
}
~~~
{: #payload-icmp-format title="REGISTER_DATAGRAM_CONTEXT_ICMP Capsule Format"}

Context ID:

: The context ID to register.

### ICMP HTTP Datagram Format {#icmp-format}

HTTP Datagrams registered to carry ICMP messages contain a summary message of
the ICMP message received and validated for the respective IP flow. The message
format carries the ICMP packet for ICMPv4 {{!RFC0792}} or ICMPv6 {{!RFC4443}}.
This format is chosen for forward compatibility. From an implementation
perspective the client don't need to verify the checksum or validate the header
fields because that is done by the server. However, some type codes, like
IMCPv4 type 2, (Packet Too Big) carries an MTU field that the implementation
want to read beyond understanding the meaning of the type and code combination.

# HTTP Headers

Note: This section should be improved by clarifying if headers are in request,
response or both.

## Flow-Forwarding header for CONNECT-IP {#ffd}

Flow-Forwarding is an Item Structured Header {{RFC8941}}. Its value MUST be a
String containing either an IPv6 literal encapsulated within square brackets,
an IPv4 address in dotted-decimal form, or a registered name.

~~~
Flow-Forwarding = sf-string
~~~

## IP-Protocol Header for CONNECT-IP {#ip-proto}

In order to construct the IP header the the proxy needs to fill the "Protocol"
field in the IPv4 header or "Next header" field in the IPv6 header. As the IP
payload is otherwise mostly opaque to the proxy, this information has to be
provided by the client for each CONNECT-IP request for flow forwarding.

IP-Protocol is a Item Structured Header {{!RFC8941}}. Its value MUST be an
Integer. Its ABNF is:

~~~
IP-Protocol = sf-integer
~~~

## IP-Version header for CONNECT-IP {#ip-version}

IP-Version is a Item Structured Header {{RFC8941}}. Its value MUST be an
Integer and either 4 or 6. This information is used by the proxy to check if
the requested IP version is supported by the network that the proxy is
connected to, as well as to check the destination or source IP address for
compliance.

~~~
IP-Version = sf-integer
~~~

## IP-Address header for CONNECT-IP {#ip-addr}

IP-Address is an Item Structured Header {{RFC8941}}. Its value MUST be an
String contain an IP address or IP address range of the same IP version as
indicated in the IP-Version header. The address must be specified in the format
specified by TBD.

This header is used to request the use of a certain IP address or IP address
range by the client to be used as source IP address. If the IP-Address header
is not presented, the proxy is implicitly requested to assign an IP address or
IP address range and provide this information to the client with the HTTP
response.

If the the client does not provide an IP address or IP address range is has to
wait for the proxy response before any payload data can be sent. If the request
is denied by the proxy, any sent payload data will be discarded and a new
CONNECT-IP request has to be sent.

The header is also used as a response header from the proxy to the client to
indicate the actual IP address or IP address range that will be used by the
proxy.

~~~
IP-Address = sf-string
~~~

## IP-Address-Handling Header for CONNECT-IP {#ip-addr-handling}

This header can be used to request the use of a stable address for multiple
active flow forwarding associations. The first association will be established
with an IP selected by the proxy unless also the IP-Address header
({{ip-addr}}) is provided and accepted by proxy. However, additional forwarding
association can be requested by the client to use the same IP address as a
previous request by specifying the stream ID as value in this header. This
header can also be used to ensure that a "new", not yet for this client used
address is selected by setting a value that is larger than the maximum stream
ID.

IP-Address-Handling is a Item Structured Header {{RFC8941}}. Its value MUST be
an Integer and indicates the stream ID of the corresponding active flow
forwarding association. Its ABNF is:

~~~
IP-Address-Handling = sf-integer
~~~

## Conn-ID Header for CONNECT-IP {#conn-id}

This document further defines a new header field to be used with CONNECT-IP
"Conn-ID". The Conn-ID HTTP header field indicates the value, offset, and
length of a field in the IP payload that can be used by the proxy as a
connection identifier in addition to the IP address and protocol tuple when
multiple connections are proxied to the same target server for incoming traffic
on the service address.

Conn-ID is a Item Structured Header {{RFC8941}}. Its value MUST be a Byte
Sequence. Its ABNF is:

~~~
 Conn-ID = sf-binary
~~~

The following parameters are defined:

* A parameter whose name is "offset", and whose value is an Integer indicating
  the offset of the identifier field starting from the beginning of a datagram
  or HTTP frame on the forwarding stream.

* A parameter whose name is "length", and whose value is an Integer indicating
  the length of the identifier field starting from the offset.

Both parameters MUST be present and the header MUST be ignored if these
parameter are not present.

This function can be used to e.g. indicate the source port field in the IP
payload when containing a TCP packet.


# MASQUE server behavior

A MASQUE server that receives an CONNECT-IP request examines the request
headers to determine if this request is for flow forwarding mode or regular
CONNECT-IP. If flow forwarding mode is in use, the server determines if the
required headers are present and which of the optional headers that are
included.

A server that supports flow forwarding mode MUST echo the client's
Flow-Forwarding header to indicate support.

The proxy maintains a database with mappings between the HTTP connections and
stream IDs and the IP level selectors and Conn-ID information. Using this
database and the pool of available addresses and the requests
IP-Address-Handling, Conn-ID, IP-Version, IP-Address headers (if included) to
select a source IP address. This selection for flow forwarding mode is further
discussed below in {{addr-selection}}.

Once the mapping is successfully established, the proxy sends a HEADERS frame
containing a 2xx series status code to the client. The response MUST contain an
IP-Address header indicating the outgoing source IP address or IP address range
selected by the proxy.

All Datagram capsules received on that stream as well as all HTTP/3 datagrams
belonging to this CONNECT-IP association are processed for forwarding to the
target server. Datagrams are processed as specified in {{construct}} to produce
IP packets that can be forwarded.

IP packets received from the target server are mapped to an active forwarding
connection and are respectively forwarded in an HTTP datagram to the client
(see {{recv}}).


## Error handling

TBD (e.g. out of IP address, conn-id collision)

## IP address selection {#addr-selection}

In flow forwarding mode the proxy constructs the IP header when sending the IP
payload towards the target server and it selects an source IP address from its
pool of IP addresses that are routed to the MASQUE server.

If no additional information about a payload field that can be used as an
identifier based on Conn-ID header is provided by the client, the proxy uses
the source/destination address and protocol ID 3-tuple in order to map an
incoming IP packet to an active forwarding connection. The proxy MUST also
consider if IP-Address-Handling header ({{ip-addr-handling}}) is included and
its value. If the IP-Address-Handling header is not included and the there has
been prior request the proxy SHOULD give the client the same source Address as
the first flow forwarding request. Given these constraints the MASQUE proxy
MUST select a source IP address that leads to a unique tuple, and if that is
not possible an error response is generated. The same IP address MAY be used
for different clients when those client connect to different target servers.
However, this also means that potentially multiple IP address are used for the
same client when multiple connection to one target server are needed. This can
be problematic if the source address is used by the target as an identifier.
Therefore it is RECOMMENDED that clients are given unique addresses unless a
large fraction of the pool has been exhausted.

If the Conn-ID header is provided, the proxy should use that field as an
connection identifier together with protocol ID, source and destination
address, as a 4-tuple. In this case it is recommended to use a stable IP
address for each client, while the same IP address might still be used for
multiple clients, if not indicated differently by the client in the
configuration file. Note that if the same IP address is used for multiple
clients, this can still lead to an identifier collision and the CONNECT-IP
request MUST be reject if such a collision is detect.

Note: Are we allowing multiple client's to share the same 3-tuple when using
Conn-ID? It might be good for privacy reasons however, it significantly
increases the collision risk.

## Constructing the IP header {#construct}

To retrieve the source and destination address the proxy looks up the mapping
for the datagram flow ID or stream identifier. The IP version, flow label,
DiffServ codepoint (DSCP), and hop limit/TTL is selected by the proxy. The IPv4
Protocol or IPv6 Next Header field is set based on the information provided by
the IP-Protocol header in the CONNECT-IP request.

The proxy MUST set the Don't Fragment (DF) flag in the IPv4 header. Payload
that does not fit into one IP packet MUST be dropped. A dropping indication
should be provided to the client. Further the proxy should provide MTU
information.

The ECN field is by default set to non-ECN capable transport (non- ECT).
Further ECN handling is described in {{ecn}}.

## Receiving an IP packet {#recv}

When the proxy receives an incoming IP packet on the external interface(s), it
checks the packet selectors to find the mappings that match the given packet.

If one or more mappings exists, it further checks if this mapping contains
additional identifier information as provided by the Conn-ID Header of the
CONNECT-IP request. If this field maps as well, the IP payload is forwarded to
the client. If no active mapping is found, the IP packet is discarded.

The above is achieve by using the selector with the most number of fields that
match the packet.

# Additional signalling

## ECN {#ecn}

ECN requires coordination with the end-to-end communication points as it should
only be used if the endpoints are also capable and willing to signal congestion
notifications to the other end and react accordingly if a congestion
notification is received.

The probing and verification in the upper layer protocol of end-to-end ECN
requires per packet control over what value is set on IP packet transmission as
well as which of all values are received by the proxy. The QUIC specification
is providing one such example in Section 13.4 of {{!RFC9000}}. Thus in flow
forwarding mode the proxy needs to be able to set and read the ECN values in
sent and received IP packets respectively. This may motivate that this
functionality is optional to implement, even if supporting CONNECT-IP
implementations in general will need to handle IP packets and their fields with
fine grained control. If optional some negotiation mechanism is needed.

Possible realizations are:

* always have two bits before payload in flow forwarding model, e.g.
by including the whole Type of Service (TOS) byte, which would also
enable DSCP setting and reading.

* use 4 different context IDs depending on what ECN field value was
received or should be set.

This is work in process and will be further specified in a future
version of this document.

## ICMP handling

In flow forwarding mode a ICMP datagram format ({{icmp-format}}) is used to
send the information from some ICMP message to the client.

The proxy upon receiving an ICMP message with a destination of an IP address it
performs flow forwarding on it needs to process the ICMP message. First it
should validate that the ICMP message and find if it matches any of its IP flow
selectors (including Conn-ID). In case there are multiple matching use the IP
selector with the most number of field that matches fully.

Some messages may be applicable both to the proxy and the client. For example
an verified ICMPv6 Packet Too Big is applicable both to the proxy and the
client. Others like ICMPv6 Destination Unreachable (Type=1), Code=3 (Address
unreachable) and Code=4 (Port unreachable) is only possible to act on by the
client.

QUESTION: Which ICMP messages should be suppressed by the proxy?

If a matching IP selector was chosen, then lookup the mapping for the HTTP
connection and Stream ID which this message should be sent to. Encapsulate the
received ICMP message in the ICMP datagram format and send it to the client.

## MTU considerations

The use of QUIC as a encapsulation between the client and proxy introduces
additional overhead. If datagrams are used to encapsulate packets between the
proxy and client, the end-to-end packets must fit within one datagram but the
size of the datagrams is limited by the tunneling encapsulation overhead.

In flow forwarding mode the client is usually also the tunnel endpoint that
knows about the tunnel overhead and can therefore restrict the size of the
packets on the end-to-end connection accordingly. However, the target endpoint
is usually not aware of the tunnel overhead. Additional signalling on the
end-to-end connection from the client to the target endpoint might be needed to
restrict the packet size. If QUIC is also used as end-to-end protocol, this
could be realized by the transport parameter. In additional, signal from the
proxy to the client could be provided as an extension to indicate the tunnel
overhead more accurately and flexibly over time. Such signalling might the
realized on the HTTP layer in order to take any additional limitations by HTTP
intermediates into account.

If the proxy receives an incoming packet from a target endpoint that is too big
to fit within a datagram on the tunnel connection, the proxy MAY either forward
the packet encapsulated in the CAPSULE frames on the respective stream or, if
IPv4 with DF bit set or IPv6 is used, the proxy MAY reject the packet and send
an ICMPv4 Packet type 3 code 4, or ICMPv6 Too Big (PTB) message.

# Examples

TBD

# Security considerations

This document does currently not discuss risks that are generic to the MASQUE
approach.

Any CONNECT-IP specific risks need further consideration in future, especially
when the handling of IP functions is defined in more detail.

# IANA considerations

## HTTP Header

This document (if published as RFC) registers the following headers in the
"Permanent Message Header Field Names" registry maintained at
<https://www.iana.org/assignments/message-headers>.

~~~
+---------------------+----------+--------+---------------+
| Header Field Name   | Protocol | Status |   Reference   |
+---------------------+----------+--------+---------------+
| Flow-Forwarding     |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
| Conn-ID             |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
| IP-Protocol         |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
| IP-Address          |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
| IP-Address-Handling |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
| IP-Version          |   http   |  exp   | This document |
+---------------------+----------+--------+---------------+
~~~

--- back

# Changes from draft-kuehlewind-masque-connect-ip-01 {#changes}
{:numbered="false"}

The overwhelming majority of the content in this draft was copied from
draft-kuehlewind-masque-connect-ip-01. The editor made the following minor
changes in order to make Flow Forwarding Mode compatible with CONNECT-IP:

Negotiation of Flow Forwarding Mode:

: draft-kuehlewind-masque-connect-ip-01 uses the ":authority" pseudo-header to
differentiate between flow forwarding mode and regular CONNECT-IP. Since many
HTTP servers are responsible for multiple authorities, this document instead
uses a new HTTP header "Flow-Forwarding" to communicate whether flow forwarding
mode is in use.

HTTP Datagram Context IDs:

: draft-kuehlewind-masque-connect-ip-01 left registration of context IDs as
future work, which prevented implementing the proposal. This draft defines two
new capsules to register context IDs and register their associated formats.


