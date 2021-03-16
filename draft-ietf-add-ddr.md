---
title: "Discovery of Designated Resolvers"
abbrev: DDR
docname: draft-ietf-add-ddr-latest
date:
category: std

ipr: trust200902
area: Internet
workgroup: ADD

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
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
  -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: caw@heapingbits.net
  -
    ins: P. McManus
    name: Patrick McManus
    org: Fastly
    email: mcmanus@ducksong.com
  -
    ins: T. Jensen
    name: Tommy Jensen
    org: Microsoft
    email: tojens@microsoft.com

--- abstract

This document defines Discovery of Designated Resolvers (DDR), a
mechanism for DNS clients to use DNS records to discover a resolver's encrypted
DNS configuration. This mechanism can be used to move from unencrypted DNS to
encrypted DNS when only the IP address of an encrypted resolver is known. It can
also be used to discover support for encrypted DNS protocols when the name of an
encrypted resolver is known. This mechanism is designed to be limited to cases
where unencrypted resolvers and their designated resolvers are operated by the same
entity.

--- middle

# Introduction

When DNS clients wish to use encrypted DNS protocols such as DNS-over-TLS (DoT)
{{!RFC7858}} or DNS-over-HTTPS (DoH) {{!RFC8484}}, they require additional
information beyond the IP address of the DNS server, such as the resolver's
hostname, non-standard ports, or URL paths. However, common configuration
mechanisms only provide the resolver's IP address during configuration. Such
mechanisms include network provisioning protocols like DHCP {{?RFC2132}} and
IPv6 Router Advertisement (RA) options {{?RFC8106}}, as well as manual
configuration.

This document defines two mechanisms for clients to discover designated
resolvers using DNS server Service Binding (SVCB, {{I-D.ietf-dnsop-svcb-https}})
records:

1. When only an IP address of an Unencrypted Resolver is known, the client
queries a special use domain name to discover DNS SVCB records associated with
the Unencrypted Resolver ({{bootstrapping}}).

2. When the hostname of an encrypted DNS server is known, the client requests
details by sending a query for a DNS SVCB record. This can be used to discover
alternate encrypted DNS protocols supported by a known server, or to provide
details if a resolver name is provisioned by a network ({{encrypted}}).

Both of these approaches allow clients to confirm that a discovered Encrypted
Resolver is designated by the originally provisioned resolver. "Equivalence" in
this context means that the resolvers are operated by the same entity; for
example, the resolvers are accessible on the same IP address, or there is a
certificate that claims ownership over both resolvers.

## Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{?RFC2119}} {{?RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

This document defines the following terms:

DDR:
: Discovery of Designated Resolvers. Refers to the mechanisms defined
in this document.

Designated Resolver:
: A resolver, presumably an Encrypted Resolver, designated by another resolver
for use in its own place. This designation can be authenticated with TLS certificates.

Encrypted Resolver:
: A DNS resolver using any encrypted DNS transport. This includes current
mechanisms such as DoH and DoT as well as future mechanisms.

Unencrypted Resolver:
: A DNS resolver using TCP or UDP port 53.

Private IP: Any IP address reserved for local network unicast use (IPv4: {{!RFC1918}} and {{!RFC6598}}; IPv6: {{!RFC4193}}).

Public IP: Any IP address that is not a Private IP. (This definition is broader
than necessary, but in this context it is safest to assume an address is public
if there is doubt.)

# DNS Service Binding Records

DNS resolvers can advertise one or more Designated Resolvers that
may offer support over encrypted channels and are controlled by the same
entity.

When a client discovers Designated Resolvers, it learns information
such as the supported protocols, ports, and server name to use in certificate
validation. This information is provided in Service Binding (SVCB) records for
DNS Servers, defined by {{!I-D.schwartz-svcb-dns}}.

The following is an example of an SVCB record describing a DoH server:

~~~
_dns.example.net  7200  IN SVCB 1 . (
     alpn=h2 dohpath=/dns-query{?dns} ipv4hint=x.y.z.w )
~~~

The following is an example of an SVCB record describing a DoT server:

~~~
_dns.example.net  7200  IN SVCB 1 dot.example.net (
     alpn=dot port=8530 ipv4hint=x.y.z.w )
~~~

If multiple Designated Resolvers are available, using one or more
encrypted DNS protocols, the resolver deployment can indicate a preference using
the priority fields in each SVCB record {{I-D.ietf-dnsop-svcb-https}}.

This document focuses on discovering DoH and DoT Designated Resolvers.
Other protocols can also use the format defined by {{!I-D.schwartz-svcb-dns}}.
However, if any protocol does not involve some form of certificate validation,
new validation mechanisms will need to be defined to support validating
equivalence as defined in {{authenticated}}.

# Discovery Using Resolver IP Addresses {#bootstrapping}

When a DNS client is configured with an Unencrypted Resolver IP address, it
SHOULD query the resolver for SVCB records for "dns://resolver.arpa" before
making other queries. Specifically, the client SHOULD issue a query for
`_dns.resolver.arpa` with the SVCB resource record type (64)
{{I-D.ietf-dnsop-svcb-https}}, retransmitting as necessary (e.g. {{?RFC1536}}
Section 1), and SHOULD NOT send other queries until a reply is received.

If the recursive resolver that receives this query has one or more Designated
Resolvers, it will return the corresponding SVCB records. When
responding to these special queries for "dns://resolver.arpa", the SVCB records
SHOULD contain at least one "ipv4hint" and/or "ipv6hint" keys. These address
hints indicate the address on which the corresponding Encrypted Resolver can be
reached and avoid additional DNS lookup for the A and AAAA records of the
Encrypted Resolver name.

## Authenticated Discovery {#authenticated}

If the Unencrypted Resolver's IP address is a Public IP, it is considered a
global reference identity and is subject to cryptographic authentication.

In order to be considered an authenticated Designated Resolver, the
TLS certificate presented by the Encrypted Resolver MUST contain the IP address
of the designating Unencrypted
Resolver within the SubjectAlternativeName certificate field. The client MUST
check the SubjectAlternativeName field for the Unencrypted Resolver's IP
address. If the
certificate can be validated, the client SHOULD use the discovered Designated
Resolver for any cases in which it would have otherwise used the
Unencrypted Resolver. Otherwise, the client SHOULD NOT use the discovered Encrypted
Resolver. Additionally, the client SHOULD suppress any further queries for
Designated Resolvers using this Unencrypted Resolver for the length of
time indicated by the SVCB record's Time to Live (TTL).

## Opportunistic Discovery {#opportunistic}

If the Unencrypted Resolver's IP address is a Private IP, it is considered
a local reference identity that cannot be confirmed using TLS certificates.
Instead, the ability to respond to queries confirms control of the address.

The client SHOULD limit the validity of the discovered information (e.g. by
limiting the SVCB response TTL) to no more than 5 minutes, and MUST stop
using any Encrypted Resolver that is no longer designated. {{optimizations}}
describes how to avoid periods of unencrypted resolution that could result
from this.

Opportunistic Privacy is defined for DoT in Section 4.1 of {{!RFC7858}} as a
mode in which clients do not validate the name of the resolver presented in the
certificate. A client MAY use information from the SVCB record for
"dns://resolver.arpa" with this "opportunistic" approach (not validating the
names presented in the SubjectAlternativeName field of the certificate) as long
as the IP address of the Encrypted Resolver is also a Private IP (such as
those defined in {{!RFC1918}}). This approach can be used for DoT or DoH.

In order to refer the user to an Encrypted Resolver on a Public IP, the
Unencrypted Resolver MUST return a CNAME that aliases `_dns.resolver.arpa` to
`_dns.$HOSTNAME`. If the client receives such a CNAME, it MAY proceed with
name-based discovery for `$HOSTNAME` ({{encrypted}}), subject to the above TTL
limit.  Clients SHOULD NOT connect to an Encrypted Resolver on a Public IP
without TLS authentication.

Resolvers that support authenticated discovery ({{authenticated}}) can also support
opportunistic discovery via local forwarders by publishing such a CNAME record and
placing their SVCB records at `_dns.$HOSTNAME`. Local forwarders that wish to
prevent this kind of discovery can intercept queries to `_dns.resolver.arpa`.

# Discovery Using Resolver Names {#encrypted}

A DNS client that already knows the name of an Encrypted Resolver can use DDR
to discover details about all supported encrypted DNS protocols. This situation
can arise if a client has been configured to use a given Encrypted Resolver, or
if a network provisioning protocol (such as DHCP or IPv6 Router Advertisements)
provides a name for an Encrypted Resolver alongside the resolver IP address.

For these cases, the client simply sends a DNS SVCB query using the known name
of the resolver. This query can be issued to the named Encrypted Resolver itself
or to any other resolver. Unlike the case of bootstrapping from an Unencrypted
Resolver ({{bootstrapping}}), these records SHOULD be available in the public
DNS.

For example, if the client already knows about a DoT server
`resolver.example.com`, it can issue an SVCB query for
`_dns.resolver.example.com` to discover if there are other encrypted DNS
protocols available. In the following example, the SVCB answers indicate that
`resolver.example.com` supports both DoH and DoT, and that the DoH server
indicates a higher priority than the DoT server.

~~~
_dns.resolver.example.com  7200  IN SVCB 1 . (
     alpn=h2 dohpath=/dns-query{?dns} )
_dns.resolver.example.com  7200  IN SVCB 2 . (
     alpn=dot )
~~~

Often, the various supported encrypted DNS protocols will be accessible using
the same hostname. In the example above, both DoH and DoT use the name
`resolver.example.com` for their TLS certficates. If a deployment uses a
different hostname for one protocol, but still wants clients to treat both DNS
servers as designated, the TLS certificates MUST include both names in the
SubjectAlternativeName fields. Note that this name verification is not related
to the DNS resolver that provided the SVCB answer.

For example, being able to discover a Designated Resolver for a known
Encrypted Resolver is useful when a client has a DoT configuration for
`foo.resolver.example.com` but is on a network that blocks DoT traffic. The
client can still send a query to any other accessible resolver (either the local
network resolver or an accessible DoH server) to discover if there is a designated
DoH server for `foo.resolver.example.com`.

# Optimizations

To avoid periods of unencrypted resolution, clients SHOULD repeat the discovery
query at least several seconds before the current SVCB record expires.  To reduce
server load, idle clients MAY defer repeating discovery until there is a pending
query, and SHOULD delay the pending query until after discovery completes.

When the client has performed Opportunistic Discovery ({{opportunistic}}), it MAY
use the SVCB record for its full TTL if the Encrypted Resolver is on the same
Private IP as the designating Unencrypted Resolver, or if the client has reason
to believe that the Encrypted Resolver is not an attacker.

# Deployment Considerations

Resolver deployments that support DDR are advised to consider the following
points.

## Caching Forwarders

If a caching forwarder consults multiple resolvers, it may be possible for it to
cache records for the "resolver.arpa" Special Use Domain Name (SUDN) for
multiple resolvers. This may result in clients sending queries intended to
discover Designated Resolvers for resolver `foo` and receiving answers
for resolvers `foo` and `bar`.

A client will successfully reject unintended connections because the
authenticated discovery will fail or the resolver addresses do not match.
Clients that attempt unauthenticated connections to resolvers discovered through
SVCB queries run the risk of connecting to the wrong server in this scenario.

To prevent unnecessary traffic from clients to incorrect resolvers, DNS caching
resolvers SHOULD NOT cache results for the "resolver.arpa" SUDN other than for
Designated Resolvers under their control.

## Certificate Management

Resolver owners that support authenticated discovery will need to list valid
referring IP addresses in their TLS certificates. This may pose challenges for
resolvers with a large number of referring IP addresses.

## Legacy Partial Forwarders

A "partial forwarder" is a DNS device that forwards some queries to an upstream
resolver, and answers some queries directly.  This includes both filtering and
split-horizon DNS behaviors.  As noted in {{opportunistic}}, such devices can
intercept queries to `_dns.resolver.arpa` to ensure that the upstream resolver
does not trigger an opportunistic upgrade. However, some networks contain legacy
partial forwarders whose behavior cannot be updated.  For compatibility with
legacy partial forwarders, clients MAY impose additional conditions on
opportunistic discovery, such as requiring that the Encrypted Resolver is on the
same Private IP as the Unencrypted Resolver.

## Limited compatibility with getaddrinfo for IP-based discovery

Clients that access DNS through the `getaddrinfo` API can only issue A and AAAA
queries, and can only receive these records and CNAMEs. Such clients MAY
attempt IP-based discovery ({{bootstrapping}}) by issuing an A or AAAA query
for `_dns.resolver.arpa`. If a CNAME record is returned, it indicates the
DNS server hostname (plus a `_dns` prefix). (The A or AAAA query is not
generally expected to succeed, and is used only to elicit the CNAME.) If the
client already knows how to connect to this server, or is able to determine
how to connect through some other mechanism, it MAY do so.

# Security Considerations

Since clients can receive DNS SVCB answers over unencrypted DNS, on-path
attackers can prevent successful discovery by dropping SVCB packets. When using
IP-based discovery ({{bootstrapping}}), it is not possible to distinguish between resolvers
that do not have a Designated Resolver and such an active attack.
Clients that wish to defend against these attacks must use name-based discovery
({{encrypted}}) with local DNSSEC validation, or otherwise have an authenticated
indication that there is an Encrypted Resolver available.

While the IP address of the Unencrypted Resolver is often provisioned over
insecure mechanisms, it can also be provisioned securely, such as via manual
configuration, a VPN, or on a network with protections like RA guard
{{?RFC6105}}. An attacker who is temporarily in-path might try to gain persistent
access to the client's DNS traffic by causing the client to discover an apparent
Designated Resolver that is actually operated by the attacker.

If the IP address of an Unencrypted Resolver is a Public IP, clients SHOULD validate
that this IP address is covered by the SubjectAlternativeName of the Encrypted
Resolver's TLS certificate ({{authenticated}}). This ensures that the Encrypted
Resolver is authorized by the operator of the Unencrypted Resolver.

When the Unencrypted Resolver has a local IP address, use of the Designated
Resolver ({{opportunistic}}) SHOULD be subject to strict freshness
requirements to ensure that a transient attacker who forges a discovery response
cannot gain persistent access to the client's DNS queries. If the Unencrypted
Resolver is on the local network but the Encrypted Resolver is remote,
name-based authentication is required to ensure that an external attacker cannot
intercept the connection.

# IANA Considerations {#iana}

## Special Use Domain Name "resolver.arpa"

This document calls for the creation of the "resolver.arpa" SUDN. This will
allow resolvers to respond to queries directed at themselves rather than a
specific domain name. While this document uses "resolver.arpa" to return SVCB
records indicating designated encrypted capability, the name is generic enough
to allow future reuse for other purposes where the resolver wishes to provide
information about itself to the client.

--- back

# Rationale for using SVCB records {#rationale}

This mechanism uses SVCB/HTTPS resource records {{!I-D.ietf-dnsop-svcb-https}}
to communicate that a given domain designates a particular Designated
Resolver for clients to use in place of an Unencrypted Resolver (using a SUDN)
or another Encrypted Resolver (using its domain name).

There are various other proposals for how to provide similar functionality.
There are several reasons that this mechanism has chosen SVCB records:

- Discovering encrypted resolver using DNS records keeps client logic for DNS
self-contained and allows a DNS resolver operator to define which resolver names
and IP addresses are related to one another.

- Using DNS records also does not rely on bootstrapping with higher-level
application operations (such as {{?I-D.schinazi-httpbis-doh-preference-hints}}).

- SVCB records are extensible and allow definition of parameter keys. This makes
them a superior mechanism for extensibility as compared to approaches such as
overloading TXT records. The same keys can be used for discovering Designated
Resolvers of different transport types as well as those advertised by
Unencrypted Resolvers or another Encrypted Resolver.

- Clients and servers that are interested in privacy of names will already need
to support SVCB records in order to use Encrypted TLS Client Hello
{{!I-D.ietf-tls-esni}}. Without encrypting names in TLS, the value of encrypting
DNS is reduced, so pairing the solutions provides the largest benefit.

- Clients that support SVCB will generally send out three queries when accessing
web content on a dual-stack network: A, AAAA, and HTTPS queries. Discovering a
Designated Resolver as part of one of these queries, without having to
add yet another query, minimizes the total number of queries clients send. While
{{?RFC5507}} recommends adding new RRTypes for new functionality, SVCB provides
an extension mechanism that simplifies client behavior.
