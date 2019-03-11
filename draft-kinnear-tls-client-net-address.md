---
title: TLS Client Network Address Extension
abbrev: TLS Client Network Address Extension
docname: draft-kinnear-tls-client-net-address-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
  -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
  RFC2119:
  RFC3947:
  RFC7858:
  RFC8174:
  RFC8310:
  RFC8446:
  I-D.ietf-quic-transport:

--- abstract

This document describes a TLS extension that may be used to send the perceived
client network address from a TLS 1.3 server to client. This information can be
used for a variety of purposes, including: NAT detection, ASN recording, and
privacy-driven transport protocol features such as connection ID rotation in QUIC.

--- middle

# Introduction

This document describes a new TLS extension for communicating the perceived client
network address from a TLS 1.3 {{RFC8446}} server to client. This has several uses, including:
NAT detection, ASN recording, and privacy-driven transport protocol features
such as connection ID rotation in QUIC. Servers that support this extension can send the
perceived client address to clients. The latter may then confirm whether or not this
representation matches their known public address. Unlike the related NAT detection
extension for IKE {{RFC3947}}, clients do not send their perceived IP address to servers,
even in an obfuscated form. Doing so would introduce an unwanted privacy regression
for clients.

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

# Client Address Use Cases

Knowledge of a public client address can serve several purposes. First,
it allows clients to detect the presence of a NAT or other proxy involved in a
TLS connection. Such proxies may improve communication privacy by masking the public
IP address of clients in a session. Modulo other cleartext signals such as session
identifiers, the anonymity set of a connection passing through a NAT is proportional
to the number of clients serviced by said NAT.
Without NAT awareness, clients cannot determine if their traffic can be linked via
IP-layer information, such as a stable source address. As a result, clients cannot
determine if privacy-driven polciies such as never resuming TLS connections improve privacy.

Second, if clients can determine whether or not they are behind a NAT, they can make
informed decisions about connection reuse. As a motivating example, consider
DNS-over-TLS {{RFC7858}}{{RFC8310}}. Privacy-sensitive clients may wish to use fresh
connections for individual queries so as to not allow recursive resolvers the ability
of building client query histories. However, in the absence of a NAT, reusing a connection
does not pose a significant privacy regression since such clients are generally identifiable
by their IP address. (Note: it is an open question as to how reliable IP addresses
are as a client identifier.)

Lastly, client network awareness may influence privacy-driven connection
migration policies, such as those prescribed by QUIC {{I-D.ietf-quic-transport}}. For example,
if clients know they are not behind a NAT, then connection ID rotation serves little value
in preventing linkability.

# Address Extension

Endpoints may send the known or perceived client IP address to its peer using the following
"network_address" extension:

~~~
enum {
    network_address(TBD), (65535)
} ExtensionType;
~~~

When sent by a client, this extension MUST be empty. A server which receives a non-empty
network_address extension MUST terminate the connection with an Illegal Parameter alert.

Supporting servers which receive this extension may respond with a NetworkAddress extension,
shown below, inside the EncryptedExtensions.

~~~
struct {
    opaque address<32..255>;
} NetworkAddress;
~~~

address
: The client's perceived address.

In this case, NetworkAddress.address carries the raw network-order byte-wise representation
of the client IP address. (Since the extension is encrypted, there is no need to obfuscate
the address for transit.) Clients which receive a non-empty NetworkAddress extension may
use it to record their public IP address. Clients MUST treat empty NetworkAddress.address
extensions as an error and send an Illegal Parameter alert in response.

# IANA Considerations

IANA is requested to Create an entry, network_address(TBD), in the existing registry
for ExtensionType (defined in {{RFC8446}}), with "TLS 1.3" column values being set to
"CH, EE", and "Recommended" column being set to "Yes".

# Security Considerations

Since NetworkAddress extension contents are encrypted, this extension introduces
no additional security or privacy issues.

An earlier design let clients send their address to servers in an obfuscated form,
e.g., by hashing the client's perceived IP address with ClientHello.random, so that
servers could measure whether or not clients were also behind NATs. However, such
obfuscation mechanisms are subject to dictionary attacks and therefore could be
used by malicious on-path attackers to learn a client's true public address. Absent
this information, there are no explicit signals from a single (non-resumed) TLS
connection that such attackers can use to learn the client's public address.

In general, absent a mechanism to encrypt the client extensions, sending the
client's perceived address in any form therefore constitutes a privacy regression.
