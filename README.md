# LSAT

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
