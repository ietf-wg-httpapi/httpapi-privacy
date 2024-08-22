---
title: "API Keys and Privacy"
abbrev: "privacy"
category: bcp

docname: draft-rsalz-httpapi-privacy-latest
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
  github: "richsalz/draft-rsalz-httpapi-privacy"
  latest: "https://richsalz.github.io/draft-rsalz-httpapi-privacy/draft-rsalz-httpapi-privacy.html"

author:
 -
    fullname: Rich Salz
    organization: Akamai Technologies
    email: rsalz@akamai.com
 -
    fullname: Mike Bishop
    organization: Akamai Technologies
    email: mbishop@akamai.com
 -
    fullname: Marius Kleidl
    organization: TODO
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
resources, can be an anti-pattern for authenticated API traffic. This document
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
initial request may include a Bearer token or other credential; once the request
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
requests over an insecure channel. Servers with authenticated endpoints SHOULD
employ both mechanisms.

## Connection Blocking

If an API request succeeds despite having an unencrypted endpoint configured,
the developer or user is less likely to notice the misconfiguration. Where
possible, it is advantageous for such a misconfiguration to fail immediately so
that the error can be noticed and corrected.

Servers MAY induce such an early failure by not accepting unencrypted
connections, e.g. on port 80. This makes it impossible for a client to send a
credential over an insecure channel to the authentic server, as no such channel
can be opened.

However, this mitigation is limited against active network attackers, who can
impersonate the server and accept the client's insecure connection attempt.

## Credential Restriction

Whenever possible, credentials should include an indicator to clients that the
credential is restricted to secure contexts. For example, Cookie-based
authentication SHOULD include the Secure attribute described in {{Section
4.1.2.5 of RFC6265}}. Bearer tokens MAY use the format described in {{?RFC8959}}
to indicate the expected usage to the client.

## Credential Revocation

Some deployments may not find it feasible to completely block unencrypted
connections, whether because the hostname is shared with unauthenticated
endpoints or for infrastructure reasons. Therefore, servers need a response for
when a valid credential has been received over an insecure channel.

Because a difference in behavior would enable attackers to guess and check
possible credentials, a server MUST NOT return a different client response
between a valid or invalid credential presented over an insecure connection.
Differences in behavior MUST only be visible on subsequent use of the credential
over a secure channel.

When a request is received over an unencrypted channel, the presented credential
is potentially compromised. Servers SHOULD revoke such credentials immediately.
When the credential is next used over a secure channel, a server MAY return an
error that indicates why the credential was revoked.

# Client Recommendations

The following recommendations increase the success rate of the server
recommendations above.

## Implement Relevant Protocols

Clients SHOULD support and query for HTTPS records {{!RFC9460}} when
establishing a connection and SHOULD respect HSTS headers {{!RFC6797}} received
from a server. This includes implementing persistent storage of HSTS indications
received from the server.

## Respect Credential Restrictions

Clients MUST NOT send a Cookie with the Secure attribute {{RFC6265}} over an
insecure channel. Clients MUST NOT send an Authorization header containing a
token whose value begins with "secret-token:" over an insecure channel.

## Disallow Insecure by Default

When authentication is used, clients SHOULD require an explicit indication from
the user or caller that an insecure context is expected. Without such an
indication, attempts to send credentials should fail without producing any
network traffic.

# Security Considerations

This entire document is about security of HTTP API interactions.

The behavior recommended in {{credential-revocation}} creates the potential for
a denial of service attack where an attacker guesses many possible credentials
over an unencrypted connection in hopes of discovering and revoking a valid one.
Servers implementing this mitigation MUST also guard against such attacks, such
as by limiting the number of requests before closing the connection and
rate-limiting the establishment of insecure connections.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments

We are grateful to Joachim Viide for his {{BLOG}} blog posting that brought up the issue.

# Notes

I don't disagree with the thrust of the article, but I take a different
message from it. It's not that the redirect is bad; the redirect is beside
the point.

An unencrypted authenticated request should, in general, not succeed. It
should at a minimum fail and ideally revoke the just-disclosed credentials.

The redirect is bad in this case because the subsequent request succeeded.
Returning a 200 over HTTP would be just as bad.

Browsers warn if you enter a password field on a form that sends to http.
Api clients don't, and many API's have an API key in the link or as a header field.

## From some email

I chose to explicitly address redirects in particular, as they seem to be the
status quo that most APIs implement, probably due to redirects being the
standard operating procedure" in webdev lore. Redirects are also often the
simplest option due to server software like Caddy supporting them out of the
box.

Ben Bucksch had good points. Failing unencrypted requests (or connection
attempts) is a relatively low hanging fruit for improving the aforementioned
status quo. Then at least accidental plain HTTP use can be noticed easily and
fixed early, reducing the window of opportunity for attacks. API providers
can also implement failures first and then move on to the more complex topic
of token auto-revocation. Anthropic chose an interesting approach: their API
now returns an error that also suggests rotating the API keys manually. I
think its a good quick improvement, informing the API consumer of a
recommended path forward.

As for using the HTTP status code 426, I almost implemented that for our API
when I noticed that NPM uses it. However,
https://httpwg.org/specs/rfc9110.html#status.426 states:

  "the server MUST send an Upgrade header field"

Later in section 7.8 (https://httpwg.org/specs/rfc9110.html#rfc.section.7.8):

  The Upgrade header field only applies to switching protocols on top of the
  existing connection; it cannot be used to switch the underlying connection
  (transport) protocol, nor to switch the existing communication to a
  different connection.

As the switch from HTTP to HTTPS would require a new connection, I felt that
it was not a perfect match. Thats why I ended up emulating Stripe with status
code 403, which "indicates that the server understood the request but refuses
to fulfill it." A bit more open-ended, but not a perfect match either.

