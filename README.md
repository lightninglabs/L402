# Lightning Service Authentication Tokens (LSAT)

In this document, we outline the design for a Lightning Service Authentication
Token (LSAT) for future services created by Lightning Labs. The high level idea
is that before a user can interact with any of our paid services, they need to
first obtain an authentication token. In order to obtain a token, we require the
user to pay us over Lightning in order to obtain a pre-image, which itself is
the token.

The implementation of the authentication token is chosen to be macaroons, as
they allow us to package attributes and capabilities along with
the token. This system allows us to automate pricing on the fly and allows for
a number of novel constructs such as automated tier upgrades.
In another light, this can be viewed as a global HTTP 402 reverse proxy at the
load balancing level for all our services.

* [Introduction](introduction.md)
* [Authentication flow](authentication-flow.md)
* [Protocol specification](protocol-specification.md)

## Implementations

* [Kirin: A gRPC/HTTP authentication reverse proxy using LSATs](https://github.com/lightninglabs/kirin)
* [lsat-js: A utility library for working with LSATs](https://github.com/Tierion/lsat-js)

## External links / References

* [LSAT: Your Ticket Aboard the Internet's Money Rails](https://docs.google.com/presentation/d/1QSm8tQs35-ZGf7a7a2pvFlSduH3mzvMgQaf-06Jjaow/edit#slide=id.p)
  slides to Olaoluwa Osuntokun's (@roasbeef) presentation at The Lightning Conference 2019 in Berlin.
* [LSAT Playground](https://lsat-playground.bucko.now.sh/)
* [Macaroons: Cookies with Contextual Caveats](https://research.google/pubs/pub41892/)
  the 2014 paper published on Google Scholar.
* [HTTP/1.1 RFC, Section 6.5.2: 402 Payment Required](https://tools.ietf.org/html/rfc7231#section-6.5.2)
* [Proposal for OAuth style delegated authentication using LSATs](https://github.com/lightningnetwork/lnd/issues/288)