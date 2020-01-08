---
title: HTTP Transport Authentication
abbrev: HTTP Transport Authentication
docname: draft-schinazi-httpbis-transport-auth
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

The most common existing authentication mechanisms for HTTP are sent with each
HTTP request, and authenticate that request instead of the underlying HTTP
connection, or transport. While these mechanisms work well for existing uses
of HTTP, they are not suitable for emerging applications that multiplex
non-HTTP traffic inside an HTTP connection. This document describes the HTTP
Transport Authentication Framework, a method of authenticating not only an
HTTP request, but also its underlying transport.


--- middle

# Introduction {#introduction}

The most common existing authentication mechanisms for HTTP are sent with each
HTTP request, and authenticate that request instead of the underlying HTTP
connection, or transport. While these mechanisms work well for existing uses
of HTTP, they are not suitable for emerging applications that multiplex
non-HTTP traffic inside an HTTP connection. This document describes the HTTP
Transport Authentication Framework, a method of authenticating not only an
HTTP request, but also its underlying transport.

Traditional HTTP semantics specify that HTTP is a stateless protocol where
each request can be understood in isolation {{!RFC7230}}. However, the
emergence of QUIC {{?I-D.ietf-quic-transport}} as a new transport protocol
that can carry HTTP {{?I-D.ietf-quic-http}} and the existence of QUIC
extensions such as the DATAGRAM frame {{?I-D.pauly-quic-datagram}} enable new
uses of HTTP such as {{?I-D.vvv-webtransport-http3}} and
{{?I-D.schinazi-masque}} where some traffic is exchanged that is disctinct
from HTTP requests and responses. In order to authenticate this traffic, it
is necessary to authenticate the underlying transport (e.g., QUIC or
TLS {{!RFC8446}}) instead of authenticate each request individually. This
mechanism aims to supplement the HTTP Authentication Framework {{?RFC7235}},
not replace it.

Note that there is currently no mechanism for origin servers to request
that user agents authenticate themselves using Transport Authentication,
this is left as future work.


## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document uses the Augmented BNF defined in {{!RFC5234}} and updated by
{{!RFC7405}} along with the "#rule" extension defined in Section 7 of
{{!RFC7230}}. The rules below are defined in {{!RFC3061}}, {{!RFC5234}},
{{!RFC7230}}, and {{!RFC7235}}:

~~~
  OWS           = <OWS, see {{!RFC7230}}, Section 3.2.3>
  quoted-string = <quoted-string, see {{!RFC7230}}, Section 3.2.6>
  token         = <token, see {{!RFC7230}}, Section 3.2.6>
  token68       = <token, see {{!RFC7235}}, Section 2.1>
  oid           = <oid, see {{!RFC3061}}, Section 2>
~~~


# Computing the Authentication Proof {#compute-proof}

This document only defines Transport Authentication for uses of HTTP with TLS.
This includes any use of HTTP over TLS as typically used for HTTP/2, or
HTTP/3 where the transport protocol uses TLS as its authentication and key
exchange mechanism {{?I-D.ietf-quic-tls}}.

The user agent leverages a TLS keying material exporter {{!RFC5705}} to
generate a nonce which can be signed using the user-id's key. The keying
material exporter uses a label that starts with the characters
"EXPORTER-HTTP-Transport-Authentication-" (see {{schemes}} for the labels
and contexts used by each scheme). The TLS keying material exporter is
used to generate a 32-byte key which is then used as a nonce.


# Header Field Definition {#header-definition}

The "Transport-Authentication" header allows a user agent to authenticate
its transport connection with an origin server.

~~~
  Transport-Authentication = transp-auth-scheme *( OWS ";" OWS parameter )
  transp-auth-scheme       = token
  parameter                = token "=" ( token / quoted-string )
~~~


## The u Directive {#directive-u}

The OPTIONAL "u" (user-id) directive specifies the user-id that the user
agent wishes to authenticate. It is encoded using
Base64 (Section 4 of {{!RFC4648}}).

~~~
    u = token68
~~~


## The p Directive {#directive-p}

The OPTIONAL "p" (proof) directive specifies the proof that the user agent
provides to attest to possessing the credential that matches its user-id.
It is encoded using Base64 (Section 4 of {{!RFC4648}}).

~~~
    p = token68
~~~


## The a Directive {#directive-a}

The OPTIONAL "a" (algorithm) directive specifies the algorithm used to compute
the proof transmitted in the "p" directive.

~~~
    a = oid
~~~


# Transport Authentication Schemes {#schemes}

The Transport Authentication Framework allows defining Transport
Authentication Schemes, which specify how to authenticate user-ids. This
documents defined the "Signature" and "HMAC" schemes.


## Signature {#signature}

The "Signature" Transport Authentication Scheme uses asymmetric cyptography.
User agents possess a user-id and a public/private key pair, and origin
servers maintain a mapping of authorized user-ids to their associated public
keys. When using this scheme, the "u", "p", and "a" directives are REQUIRED.
The TLS keying material export label for this scheme is
"EXPORTER-HTTP-Transport-Authentication-Signature" and the associated
context is empty. The nonce is then signed using the selected asymmetric
signature algorithm and transmitted as the proof directive.

For example, the user-id "john.doe" authenticating using Ed25519 {{?RFC8410}}
could produce the following header (lines are folded to fit):

~~~
Transport-Authentication: Signature u="am9obi5kb2U=";a=1.3.101.112;
p="SW5zZXJ0IHNpZ25hdHVyZSBvZiBub25jZSBoZXJlIHdo
aWNoIHRha2VzIDUxMiBiaXRzIGZvciBFZDI1NTE5IQ=="
~~~


## HMAC {#hmac}

The "HMAC" Transport Authentication Scheme uses symmetric cyptography.
User agents possess a user-id and a secret key, and origin servers maintain a
mapping of authorized user-ids to their associated secret key. When using this
scheme, the "u", "p", and "a" directives are REQUIRED.
The TLS keying material export label for this scheme is
"EXPORTER-HTTP-Transport-Authentication-HMAC" and the associated
context is empty. The nonce is then HMACed using the selected HMAC algorithm
and transmitted as the proof directive.

For example, the user-id "john.doe" authenticating using
HMAC-SHA-512 {{?RFC6234}} could produce the following
header (lines are folded to fit):

~~~
Transport-Authentication: HMAC u="am9obi5kb2U=";a=2.16.840.1.101.3.4.2.3;
p="SW5zZXJ0IEhNQUMgb2Ygbm9uY2UgaGVyZSB3aGljaCB0YWtl
cyA1MTIgYml0cyBmb3IgU0hBLTUxMiEhISEhIQ=="
~~~


# Proxy Considerations {#proxy}

Since Transport Authentication authenticates the underlying transport by
leveraging TLS keying material exporters, it cannot be transparently
forwarded by proxies that terminate TLS. However it can be sent over
proxied connections when TLS is performed end-to-end (e.g., when using
HTTP CONNECT proxies).


# Security Considerations {#security}

Transport Authentication allows a user-agent to authenticate to an origin
server while guaranteeing freshness and without the need for the server
to transmit a nonce to the user agent. This allows the server to accept
authenticated clients without revealing that it supports or expects
authentication for some resources. It also allows authentication without
the user agent leaking the presence of authentication to observers due to
clear-text TLS Client Hello extensions.


# IANA Considerations {#iana}


## Transport-Authentication Header Field {#iana-header}

This document, if approved, requests IANA to register the
"Transport-Authentication" header in the "Permanent Message Header Field Names"
registry maintained at
<https://www.iana.org/assignments/message-headers/>.

~~~
  +--------------------------+----------+--------------+---------------+
  |    Header Field Name     | Protocol |    Status    |   Reference   |
  +--------------------------+----------+--------------+---------------+
  | Transport-Authentication |   http   | experimental | This document |
  +--------------------------+----------+--------------+---------------+
~~~


## Transport Authentication Schemes Registry {#iana-schemes}

This document, if approved, requests IANA to create a new HTTP Transport
Authentication Schemes Registry with the following entries:

~~~
  +---------------------------------+---------------+
  | Transport Authentication Scheme |   Reference   |
  +---------------------------------+---------------+
  |            Signature            | This document |
  +---------------------------------+---------------+
  |              HMAC               | This document |
  +---------------------------------+---------------+
~~~


## TLS Keying Material Exporter Labels {#iana-exporter-label}

This document, if approved, requests IANA to register the following entries in
the "TLS Exporter Labels" registry maintained at
<https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#exporter-labels>

~~~
  +--------------------------------------------------+
  |                       Value                      |
  +--------------------------------------------------+
  | EXPORTER-HTTP-Transport-Authentication-Signature |
  +--------------------------------------------------+
  | EXPORTER-HTTP-Transport-Authentication-HMAC      |
  +--------------------------------------------------+
~~~

Both of these entries are listed with the following qualifiers:

~~~
  +---------+-------------+---------------+
  | DTLS-OK | Recommended |   Reference   |
  +---------+-------------+---------------+
  |    N    |      Y      | This document |
  +---------+-------------+---------------+
~~~


--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}

The authors would like to thank many members of the IETF community, as this
document is the fruit of many hallway conversations. Using the OID for the
signature and HMAC algorithms was inspired by Signature Authentication in
IKEv2 {{?RFC7427}}.


