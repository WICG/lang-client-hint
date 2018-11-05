# Explainer: The `Lang` Client Hint

Mike West, November 2018

_©2018, Google, Inc. All rights reserved._

(_Note: This isn't a proposal that's well thought out, and stamped solidly with the Google Seal of
Approval. It's a collection of interesting ideas for discussion, nothing more, nothing less._)

## A Problem

User agents specify a set of languages as part of each HTTP request via the `Accept-Languages` header.
This header's value is critical for some kinds of content negotiation, ideally allowing servers to give
each user content in a language they prefer. My user agent sends a header along the lines of:

```http
Accept-Language: en-US,en;q=0.9,de;q=0.8
```

This shows that I prefer content in English, but accept content in German as well. That's great for
a few important use cases, but exposes quite a bit of entropy to the web at large, even though a
small subset of sites I visit will actually use the information to improve my experience.

This document proposes that we deprecate that header, and move to an opt-in model which will reveal
my language preferences to the specific sites that declare they can do something useful with it,
while keeping them off the wire for the rest.

## A Proposal

In the glorious future, servers will receive no information about the user's language preferences.
Servers can instead opt-into receiving such information via a new `Lang` [Client Hint][1]. We might
accomplish this as follows:

[1]: https://tools.ietf.org/html/draft-ietf-httpbis-client-hints
[2]: https://tools.ietf.org/html/draft-ietf-httpbis-header-structure

1.  Browsers should deprecate the `Accept-Language` header over time, initially locking the string
    to something genericly appropriate (perhaps based normal language preferences for the user's
    rough, IP-based geolocation), and eventually removing it entirely.

2.  Browsers should introduce a new `Lang` Client Hint header field, representing the users'
    language preferences. This would be a [Structured Header][2] whose value is a flat list of
    language codes in the users' preferred order. For example:

    ```http
    Lang: "en-US", "en", "de"
    ```

3.  Browsers should restrict the `navigator.languages` API to secure contexts, to match Client
    Hints' scope.

4.  Servers can opt-into the new `Lang` Client Hint by delivering an appropriate `Accept-CH`
    header in the usual way:

    ```http
    Accept-CH: Lang, DPR, Width
    ```

    Note that `Accept-CH` is only accepted over secure transport: this proposal has the effect
    of deprecating language availability over plaintext entirely. Note also that only the top-level
    document's server can opt-into Client Hints. If third-parties require the data, it can be
    delegated to them via an appropriate Feature Policy.

# FAQ

## What about the first request? How will the server know what to do?

If the server absolutely requires language information to make a reasonable decision about what
data to send to the user, it can do so by sending an `Accept-CH` header along with a 302 response,
causing another navigation request, which will include the `Lang` header.

## Should we include the `q` weighting?

The `Accept-Language` header includes a weighting mechanism (e.g. the `q=0.8` bits in the example
above). It doesn't appear to me that that has any actual meaning, and I'll blithly assert that it
can be removed without impact. Hopefully [Chesterton][3] won't mind.

[3]: https://en.wikipedia.org/wiki/Wikipedia:Chesterton%27s_fence
