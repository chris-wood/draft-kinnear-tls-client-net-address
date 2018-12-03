---
title: TLS NAT Detection Extension
abbrev: TLS NAT Detection Extension
docname: draft-wood-tls-nat-detection-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
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

This document describes a TLS extension that may be used to detect the presence
of an on-path NAT or similar proxy between client and server of a TLS connection.
A motivating use case for this extension is reliable NAT detection measurements.
Secondary use cases include client-side and server-side detection of NATs for the
purpose of informed QUIC connection migration policies.

--- middle

# Introduction

NATs and other proxies may improve TLS communication privacy by masking the true
IP address of clients in a session. Modulo other cleartext signals such as session
identifiers, the anonymity set of a connection passing through a NAT is proportional
to the number of clients serviced by said NAT.

In practice, clients cannot detect NATs when establishing normal TLS connections.
This can be problematic for because clients cannot tell if their traffic can be linked via
IP-layer information, such as a stable source address. As a result, clients do not know if
privacy-driven connection migration policies, such as those prescribed by QUIC {{I-D.ietf-quic-transport}},
provide value against a passive eavesdropper. Moreover, without knowledge of the presence of a NAT,
it is unclear if using TLS over a connection-oriented protocol such as TCP introduces any
privacy regression with respect to connectionless protocols, such as UDP.

As a motivating example, consider DNS-over-TLS {{RFC7858}}{{RFC8310}} versus dnscrypt. It is
hypothesized that the latter is superior from a privacy perspective due to the lack of stable,
persistent identifiers. However, if one can show that NATs are infrequently used for encrypted
DNS, then using a connection-oriented protocol such as DoT does not induce a privacy regression.

This document describes a new TLS extension for NAT detection in TLS 1.3 {{RFC8446}}. It requires
servers that support this extension to send the perceived client address to clients. The latter may then confirm whether or
not this representation matches their known public address and, if so, conclude that no NAT or
on-path proxy was involved in the communication. Clients may also optionally send an obfuscated
version of their public IP address to servers, allowing the latter to similarly detect the presence
of a NAT or proxy.

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

# Address Extension

Endpoints may send the known or perceived client IP address to its peer using the following
"network_address" extension:

~~~
enum {
    network_address(TBD), (65535)
} ExtensionType;
~~~

Clients may send this extension in ClientHello. It contains the following structure

~~~
struct {
    opaque address<0..255>;
} NetworkAddress;
~~~

address
: A representation of the client's known or perceived address.

When sent by a client, the address may be empty. If not empty, it MUST be constructed
as NetworkAddress.address = SHA256(CH.random || ADDR), where ADDR is the network-order
byte-wise representation of the client's public IP address. It will be either 4 or 16
bytes for IPv4 or IPv6 addresses, accordingly. A server which receives a non-empty
network_address extension may use its contents and the arrival packet IP-layer information
to determine if a NAT was in use. Specifically, the server may re-compute NetworkAddress.address
using the perceived client IP address and CH.random. If the recomputed value matches
the contents of the extension, then no NAT was present.

A supporting server may echo back the NetworkAddress extension inside the EncryptedExtensions.
In this case, NetworkAddress.address = ADDR, the raw network-order byte-wise representation
of the client IP address. (Since the extension is encrypted, there is no need to obfuscate
the address for transit.) Clients which receive a non-empty NetworkAddress extension may
use it detect the presence of a NAT. This is done by comparing the contents with their known
or perceived public IP address.

Clients MUST treat empty NetworkAddress.address extensions as errors and send an Illegal Parameter
alert in response.

# IANA Considerations

IANA is requested to Create an entry, network_address(TBD), in the existing registry
for ExtensionType (defined in {{RFC8446}}), with "TLS 1.3" column values being set to
"CH, EE", and "Recommended" column being set to "Yes".

# Security Considerations

XXX

# Acknowledgments

XXX
