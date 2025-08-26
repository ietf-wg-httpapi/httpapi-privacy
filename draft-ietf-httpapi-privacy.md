---
title: "API Keys and Privacy"
abbrev: "privacy"
category: bcp

docname: draft-ietf-httpapi-privacy-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Building Blocks for HTTP APIs"
venue:
  group: "Building Blocks for HTTP APIs"
  type: "Working Group"
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "ietf-wg-httpapi/httpapi-privacy"
  latest: "https://ietf-wg-httpapi.github.io/draft-ietf-httpapi-privacy/draft-ietf-httpapi-privacy.html"

author:
 -
    fullname: Rich Salz
    organization: Akamai Technologies
    email: rsalz@akamai.com
 -
    fullname: Mike Bishop
    organization: Akamai Technologies
    email: mbishop@evequefou.be
 -
    fullname: Marius Kleidl
    organization: Transloadit
    email: marius@transloadit.com

normative:
  RFC6265:

informative:
  BLOG:
    target: https://jviide.iki.fi/http-redirects
    title: Your API Shouldn't Redirect HTTP to HTTPS
    date: "May, 2024"
    author:
      ins: J. Viide


--- abstract

Redirecting HTTP requests to HTTPS, a common pattern for human-facing web
resources, can be an anti-pattern for authenticated HTTP API traffic.
This document
discusses the pitfalls and makes deployment recommendations for authenticated
HTTP APIs. It does not specify a protocol.

--- middle

# Introduction

It is a common pattern for HTTP servers to prefer serving resources over HTTPS.
Because HTTPS uses TLS, clients receive authentication of the server and
confidentiality of the resource bodies supplied by the server.

In order to implement this preference, HTTP servers often listen for unencrypted
requests and respond with a 3XX status code directing the client to the
equivalent resource over an encrypted connection. For unauthenticated web
browsing, this is a reasonable user experience bridge. Users often type bare
hostnames (not URIs) into a user agent; if the user agent defaults to an
unencrypted connection, the server can correct this default and require the use
of encryption. This pattern is so well established that many HTTP server and
intermediary implementations have a prominently displayed option to enable it
automatically.

When client authentication is used, more care must be taken. The client's
initial request may include a Bearer token or other credential (such as
a Cookie); once the request
has been sent on the network, any passive attacker who can see the traffic can
acquire this credential and use it.

If the server performs a redirect in this situation, it does not mitigate
exposure of the credential. Further, because the request will ultimately succeed
if the client follows the redirect, an application developer or user who
accidentally configures an unencrypted API endpoint will not necessarily notice
the misconfiguration.

This document describes actions API servers and clients should take in order to
safeguard credentials. These recommendations are not directed at resources where
no authentication is used.

Some have wondered if this document is really necessary. After all, we have
been telling people not to send passwords and such in the clear for decades.
Regrettably, this lesson seems to be largely forgotten by those developing
Web-based APIs.  The blog post that motivated this document, {{BLOG}}, did a
spot-check in May, 2024, and found over two dozen websites that were
vulnerable to the issues listed here.


## Conventions and Definitions

{::boilerplate bcp14info}


# Server Recommendations

## Pre-Connection Redirects

Various mechanisms exist to inform clients that unencrypted requests to a server
are never appropriate:

- HTTP Strict Transport Security (HSTS) {{!RFC6797}} informs clients who make a
  successful connection over HTTPS that secure connections are a requirement in
  the future.
- HTTPS DNS records {{!RFC9460}} inform clients at connection time to use only
  secure connections to the indicated server.

Neither mechanism is foolproof. An attacker with control of the network or the
DNS server could block resolution of HTTPS records on a client connecting to a
new server, while HSTS requires a successful prior connection to the server and
relies on the client to implement persistent storage of the HSTS directive.

Used together, however, both approaches make clients less likely to send any
requests over an insecure channel.
HTTP API servers with authenticated endpoints SHOULD
employ both mechanisms.

## Connection Blocking

If an API request succeeds despite having an unencrypted endpoint configured,
the developer or user is less likely to notice the misconfiguration. Where
possible, it is advantageous for such a misconfiguration to fail immediately so
that the error can be noticed and corrected.

HTTP API servers MAY induce such an early failure by not accepting unencrypted
connections, e.g. on port 80. This makes it impossible for a client to send a
credential over an insecure channel to the authentic server, as no such channel
can be opened.
HTTP API servers MAY alternatively restrict connections on port 80 to
network sources which are more trusted, such as a VPN or virtual network
interface.

However, this mitigation is limited against active network attackers, who can
impersonate the HTTP API server and accept the client's insecure
connection attempt.

## Credential Restriction

Whenever possible, credentials should include an indicator to clients that the
credential is restricted to secure contexts. For example, Cookie-based
authentication SHOULD include the Secure attribute described in {{Section
4.1.2.5 of RFC6265}}. Bearer tokens MAY use the format described in {{?RFC8959}}
to indicate the expected usage to the client.

## Disclosure Response

Some deployments may not find it feasible to completely block unencrypted
connections, whether because the hostname is shared with unauthenticated
endpoints or for infrastructure reasons. Therefore, HTTP API servers need
a response for
when a credential has been received over an insecure channel.

HTTP status code 403 (Forbidden) indicates that "the server understood the
request but refuses to fulfill it" {{!HTTP=RFC9110}}. While this is generally
understood to mean that "the server considers \[the credentials] insufficient to
grant access," it also states that "a request might be forbidden for reasons
unrelated to the credentials." HTTP API servers SHOULD return status
code 403 to all
requests received over an insecure channel, regardless of the validity of the
presented credentials.

Because a difference in behavior would enable attackers to guess and check
possible credentials, an HTTP API server MUST NOT return a different client
response
between a valid or invalid credential presented over an insecure connection.
Differences in behavior MUST only be visible on subsequent use of the credential
over a secure channel.

### Credential Revocation

When a request is received over an unencrypted channel, the presented credential
is potentially compromised. HTTP API servers SHOULD revoke such
credentials immediately.
When the credential is next used over a secure channel, the server MAY return an
error that indicates why the credential was revoked.

Credentials in a request can take on different forms. API keys and tokens are simple
modes for authentication, but can be abused by attackers to forge requests and hence
should be revoked if compromised. Requests can also be authenticated using derived values,
where they only include digital signatures or message authentication codes (MACs)
derived from credentials but not the credentials themselves. Since an attacker cannot
abuse the derived values to forge requests, the server MAY choose to not revoke the
credentials in this case.

# Client Recommendations

The following recommendations increase the success rate of the server
recommendations above.

## Implement Relevant Protocols

Clients SHOULD support and query for HTTPS records {{!RFC9460}} when
establishing a connection. This gives HTTP API servers an opportunity
to provide more
complete information about capabilities, some of which are security-relevant.

Clients SHOULD respect HSTS headers {{!RFC6797}} received
from a server. This includes implementing persistent storage of HSTS indications
received from the server.

## Respect Credential Restrictions

Clients MUST NOT send a Cookie with the Secure attribute {{RFC6265}} over an
insecure channel. Clients MUST NOT send an Authorization header containing a
token whose value begins with "secret-token:" over an insecure channel.

## Disallow Insecure by Default

When authentication is used, clients SHOULD require an explicit indication from
the user or caller that an insecure context is expected which is distinct from
the provided URI. Depending on the interface, this might be a UI preference or
an API flag.

Absent such an indication, clients of HTTP APIs MUST implement and use HTTPS
exclusively.

# Security Considerations

This entire document is about security of HTTP API interactions.

The behavior recommended in {{credential-revocation}} creates the potential for
a denial of service attack where an attacker guesses many possible credentials
over an unencrypted connection in hopes of discovering and revoking a valid one.
HTTP API servers implementing this mitigation MUST also guard against such attacks, such
as by limiting the number of requests before closing the connection and
rate-limiting the establishment of insecure connections.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

We are grateful to Joachim Viide for his {{BLOG}} blog posting that brought up the issue.
