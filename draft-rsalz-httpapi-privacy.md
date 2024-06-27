---
title: "API Keys and Privacy"
abbrev: "privacy"
category: info

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

informative:
  BLOG:
    target: https://jviide.iki.fi/http-redirects
    title: Your API Shouldn't Redirect HTTP to HTTPS
    date: "May, 2024"
    author:
      ins: J. Viide


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


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

# Security Considerations

This entire document is about security of HTTP API interactions.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

We are grateful to Joachim Viide for his {{BLOG}} blog posting that brought up the issue.
