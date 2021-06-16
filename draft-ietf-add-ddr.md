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
entity or cooperating entities.

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
the Unencrypted Resolver ({{by-ip}}).

2. When the hostname of an encrypted DNS server is known, the client requests
details by sending a query for a DNS SVCB record. This can be used to discover
alternate encrypted DNS protocols supported by a known server, or to provide
details if a resolver name is provisioned by a network ({{by-name}}).

Both of these approaches allow clients to confirm that a discovered Encrypted
Resolver is designated by the originally provisioned resolver. "Designated" in
this context means that the first resolver has selected the second as a suitable
alternative for clients to use automatically.

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

Compatible:
: A client and an Encrypted Resolver are considered compatible if there is
at least one encrypted DNS transport that they both support.

Validation Identity:
: An identity for the Encrypted Resolver that can be checked via TLS
certificate validation.

Private IP:
: Any IP address reserved for local network unicast use (IPv4: {{!RFC1918}}
and {{!RFC6598}}; IPv6: {{!RFC4193}}).

Public IP:
: Any IP address that is not a Private IP. (This definition is broader than
necessary, but in this context it is safest to assume an address is public if
there is doubt.)

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
     alpn=h2 dohpath=/dns-query{?dns} )
~~~

The following is an example of an SVCB record describing a DoT server:

~~~
_dns.example.net  7200  IN SVCB 1 dot.example.net (
     alpn=dot port=8530 )
~~~

If multiple Designated Resolvers are available, using one or more
encrypted DNS protocols, the resolver deployment can indicate a preference using
the priority fields in each SVCB record {{I-D.ietf-dnsop-svcb-https}}.

This document focuses on discovering DoH and DoT Designated Resolvers.
Other protocols can also use the format defined by {{!I-D.schwartz-svcb-dns}}.
However, if any protocol does not involve some form of certificate validation,
new validation mechanisms will need to be defined to support validating
designation as defined in {{validation}}.

# Security Goals

In general, the Encrypted and Unencrypted resolvers can have different IP
addresses. The client reaches these resolvers over different network paths,
respectively the "encrypted path" and the "unencrypted path". For security
analysis, we consider these paths separately, with the following goals:

* Passive attacks on either path can always be prevented.
* If the client starts with a global identity for a resolver, and knows ahead
  of time that it is compatible, it can reliably detect any active attack on
  either path (i.e. authenticated encryption).
* If there is no attacker on the unencrypted path, the unencrypted
  resolver can be configured in such a way that compatible clients can
  reliably detect active attacks on the encrypted path.
* If the attacker is on the encrypted path, their attack lasts only as long
  as they remain on-path.
* If the attacker is on the unencrypted path, their attack lasts only a
  short time after they cease to be on-path.

# Discovery Using Resolver IP Addresses {#by-ip}

When a DNS client is configured with an Unencrypted Resolver IP address, it
SHOULD query the resolver for SVCB records for "dns://resolver.arpa" before
making other queries. Specifically, the client SHOULD issue a query for
`_dns.resolver.arpa` with the SVCB resource record type (64)
{{I-D.ietf-dnsop-svcb-https}}, retransmitting as necessary (e.g. {{?RFC1536}}
Section 1), and SHOULD NOT send other queries until a reply is received.

If the recursive resolver that receives this query has one or more Designated
Resolvers, it will return the corresponding SVCB records. When responding
to these special queries for "dns://resolver.arpa", the recursive resolver
SHOULD include the A and AAAA records for the name of the Designated Resolver
in the Additional Answers section. This will allow the DNS client to make
queries over an encrypted connection without waiting to resolve the Encrypted
Resolver name per {{!I-D.ietf-dnsop-svcb-https}}. If no A/AAAA records or SVCB
IP address hints are included, clients will be forced to delay use of the
Encrypted Resolver until an additional DNS lookup for the A and AAAA records
can be made to the Unencrypted Resolver (or some other resolver the DNS client
has been configured to use).

If the recursive resolver that receives this query has no Designated Resolvers,
it SHOULD return NODATA for queries to the "resolver.arpa" SUDN.

If the Unencrypted Resolver IP address is a Public IP, that IP address is the
Validation Identity (see {{validation}}).

If the Unencrypted Resolver IP address is a Private IP, there is no Validation
Identity unless the SVCB query resolved to a CNAME whose target has the form
`_dns.$HOSTNAME`, in which case the Validation Identity is `$HOSTNAME`.  Use of
such a CNAME defends against attackers on the encrypted path, which likely
traverses the public internet.

# Discovery Using Resolver Names {#by-name}

A DNS client that already knows the name of an Encrypted Resolver can use DDR
to discover details about all supported encrypted DNS protocols. This situation
can arise if a client has been configured to use a given Encrypted Resolver, or
if a network provisioning protocol (such as DHCP or IPv6 Router Advertisements)
provides a name for an Encrypted Resolver alongside the resolver IP address.

For these cases, the client simply sends a DNS SVCB query using the known name
of the resolver. This query can be issued to the named Encrypted Resolver itself
or to any other resolver. Unlike the case of bootstrapping from an Unencrypted
Resolver ({{by-ip}}), these records SHOULD be available in the public DNS.

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

Regardless of the SVCB records' owner name or TargetName, the Validation
Identity is always the original hostname known by the DNS client.

This discovery procedure can be used to discover a Designated Resolver for a 
known Encrypted Resolver.  For example, this might be useful when a client
has a DoT configuration for
`foo.resolver.example.com` but is on a network that blocks DoT traffic. The
client can still send a query to any other accessible resolver (either the local
network resolver or an accessible DoH server) to discover if there is a designated
DoH server for `foo.resolver.example.com`.

# Validation

If the client expects to find a compatible Encrypted Resolver, but does not
receive a SVCB response with at least one supported transport, it SHOULD
interpret this as a possible active attack.

If the client has a Validation Identity for the Encrypted Resolver, it SHOULD
validate the server's TLS certificate for this identity. If validation fails,
it SHOULD interpret this as a possible active attack.

If the client does not have a Validation Identity, or the Validation Identity
was discovered insecurely (e.g. via CNAME), the client SHOULD limit the
validity duration of the discovered information (e.g. the SVCB response
TTL) to no more than 5 minutes. The client MAY make use of this encrypted
connection, but SHOULD NOT assume that the connection is more secure
than an unencrypted connection.

When the discovery information expires, the client MUST NOT continue to rely
on it, and SHOULD repeat the discovery procedure.

## Optimizations

To avoid periods of unencrypted resolution, clients SHOULD repeat the discovery
query at least several seconds before the current SVCB record expires. To reduce
server load, idle clients MAY defer repeating discovery until there is a pending
query, and SHOULD delay the pending query until after discovery completes.

Clients MAY use the SVCB record for its full TTL, not applying the limit
described above, if the Encrypted Resolver is on the same IP address as the
designating Unencrypted Resolver, or if the client has reason to believe that
the Encrypted Resolver is not an attacker.

# Deployment Considerations

Resolver deployments that support DDR are advised to consider the following
points.

## Caching Forwarders

When using IP-based discovery ({{by-ip}}), clients may bypass DNS forwarders
that forward queries for "resolver.arpa" upstream. If this is not the
forwarder's intended behavior, it SHOULD NOT forward these queries upstream.

Operators who choose to forward queries for "resolver.arpa" upstream should note
that client behavior is never guaranteed and use of DDR by a resolver does not
communicate a requirement for clients to use the SVCB record when it cannot be
authenticated.

Some networks contain legacy forwarders whose behavior cannot be updated.
For compatibility with these networks, clients MAY impose additional conditions
on discovery from a Private IP, such as requiring that the Encrypted Resolver
is on the same IP address as the Unencrypted Resolver.

## Certificate Management

Resolver owners that support discovery via Public IPs will need to list valid
referring IP addresses in their TLS certificates. This may pose challenges for
resolvers with a large number of referring IP addresses.

## Limited compatibility with getaddrinfo for IP-based discovery

Clients that access DNS through the `getaddrinfo` API can only issue A and
AAAA queries, and can only receive these records and CNAMEs. Such clients MAY
attempt IP-based discovery ({{by-ip}}) by issuing an A or AAAA query
for `_dns.resolver.arpa`. If a CNAME record is returned, it indicates the
DNS server hostname (plus a `_dns` prefix). (The A or AAAA query is not
generally expected to succeed, and is used only to elicit the CNAME.) If the
client already knows how to connect to this server, or is able to determine
how to connect through some other mechanism, it MAY do so.

# Security Considerations

Since clients can receive DNS SVCB answers over unencrypted DNS, on-path
attackers can prevent successful discovery by dropping or modifying SVCB
responses. Clients that wish to defend against these attacks must use
name-based discovery ({{by-name}}) with local DNSSEC validation, or otherwise
have an authenticated indication that there is a compatible Encrypted
Resolver available.

While the IP address of the Unencrypted Resolver is often provisioned over
insecure mechanisms, it can also be provisioned securely, such as via manual
configuration, a VPN, or on a network with protections like RA guard
{{?RFC6105}}. An attacker who is temporarily on-path might try to gain
persistent access to the client's DNS traffic by causing the client to
discover an apparent Designated Resolver that is actually operated by the
attacker.

If the IP address of an Unencrypted Resolver is a Public IP, this attack is
prevented by using this IP as a Validation Identity. Otherwise, persistence
is prevented by imposing strict freshness requirements on the discovery
information.

# IANA Considerations {#iana}

## Special Use Domain Name "resolver.arpa"

This document calls for the creation of the "resolver.arpa" SUDN. This will
allow resolvers to respond to queries directed at themselves rather than a
specific domain name. While this document uses "resolver.arpa" to return SVCB
records indicating designated encrypted capability, the name is generic enough
to allow future reuse for other purposes where the resolver wishes to provide
information about itself to the client.

The "resolver.arpa" SUDN is similar to "ipv4only.arpa" in that the querying
client is not interested in an answer from the authoritative "arpa" name
servers. The intent of the SUDN is to allow clients to communicate with the
Unencrypted Resolver much like "ipv4only.arpa" allows for client-to-middlebox
communication. For more context, see the rationale behind "ipv4only.arpa" in
{{?RFC8880}}.

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
