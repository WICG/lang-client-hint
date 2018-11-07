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

1.  Browsers should deprecate and eventually remove the `Accept-Language` header. In order to
    quickly realize some privacy gains without a lot of compatibility impact, the header could be
    locked to something genericly appropriate (perhaps based on typical language preferences for the
    user's rough, IP-based geolocation) as an intermediate step.

2.  Browsers should introduce a new `Sec-CH-Lang` header field, representing the users'
    language preferences. This would be a [Structured Header][2] whose value is a flat list of
    language codes in the users' preferred order. For example:

    ```http
    Sec-CH-Lang: "en-US", "en", "de"
    ```

3.  Browsers should restrict the `navigator.language` and `navigator.languages` APIs to secure
    contexts, to match Client Hints' scope.

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
causing another navigation request, which will include the `Sec-CH-Lang` header.

## Should we include the `q` weighting?

The `Accept-Language` header includes a weighting mechanism (e.g. the `q=0.8` bits in the example
above). A potential application of that is expressing equal preference for two or more languages,
but the challenge of exposing such an option to users (compared to an ordered list) seems to make
practical use unlikely. I'll blithely assert that `q` weighting can be removed without impact.

## Wait a minute, I don't see this delegation stuff in the Client Hints spec...

Right. There are more than a few open PRs:

* Fetch integration of Accept-CH opt-in: [whatwg/fetch#773](whatwg/fetch#773)
* HTML integration of Accept-CH-Lifetime and the ACHL cache: [whatwg/HTML#3774](https://github.com/whatwg/html/issues/3774)
* Adding new CH features to the CH list in Fetch: [whatwg/fetch#725](https://github.com/whatwg/fetch/issues/725)
* Other PRs for adding the Feature Policy 3rd party opt-in: [whatwg/fetch#811](https://github.com/whatwg/fetch/issues/811) and [wicg/feature-folicy#220](https://github.com/wicg/feature-policy/issues/220)

## What's with the `Sec-CH-` prefix?

 Based on some discussion in [w3ctag/design-reviews#320](https://github.com/w3ctag/design-reviews/issues/320#issuecomment-435874298),
it seems reasonable to forbid access to these headers from JavaScript, and demarcate them as
browser-controlled client hints so they can be documented and included in requests without triggering
CORS preflights. A `Sec-CH-` prefix seems like a viable approach. _This bit might shift as the broader
Client Hints discussions above coalesce into something more solid that lands in specs_.
