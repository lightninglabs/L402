# Protocol Specification

## Introduction

In this chapter, we outline the specification for the abstract L402 HTTP and gRPC protocols. This is intended to be along the lines of the document we would submit if we were submitting the L402 HTTP/gRPC protocol to a standards committee. For more details on the higher-level purpose and motivations behind L402, please [see this chapter](introduction.md).

## Specification

This section defines the "L402" authentication scheme, which transmits credentials as `<token(s)>:<preimage>` pairs, where the preimage is encoded as hex and the token is encoded as base64. Multiple tokens are base64 encoded individually and listed comma separated before the colon.
This scheme is not considered to be a secure method of user authentication unless used in conjunction with some external secure system such as TLS, as the token and preimage are passed over the network as cleartext.

The L402 authentication scheme is based on the model that the client needs to authenticate itself with a token and invoice preimage for each backend service it wants to access. The server will service the request only if it can validate the token and preimage for the particular backend service requested. The L402 protocol is token-format agnostic: any authentication token that can commit to a payment hash may be used. Macaroons \(HMAC chain based bearer credentials\) are the RECOMMENDED token format due to their support for delegation, attenuation of capabilities, and stateless verification. See [Macaroon Minting & Verification](macaroons.md) for a full specification of the macaroon-based token format.

The L402 authentication scheme utilizes the Authentication Framework specified in [_RFC 7235_](https://tools.ietf.org/html/rfc7235) as follows.

In challenges: the scheme name is "L402". Note that the scheme name is case-insensitive. For credentials, the syntax is:

`auth-tokens` → [_&lt;base64 encoding&gt;_](https://tools.ietf.org/html/rfc3548#section-3), comma separated if multiple auth tokens are present
`preimage` → [_&lt;hex encoding&gt;_](https://tools.ietf.org/html/rfc3548#section-6)
`L402 credential` → auth-tokens ":" preimage

Specifically, the syntax for "L402 credential" specified above is used, which can be considered comparable to the [_"token68" syntax_](https://tools.ietf.org/html/rfc7235#section-2.1) used for HTTP basic auth.

### Reusing Credentials

L402 credentials are intended to be reused until they are revoked and the server issues a new challenge in response to a client request containing a newly invalid L402. Possible revocation conditions include: expiry date, exceeded N usages, volume of usages in a certain time period necessitating a tier upgrade, and potentially others \(discussed further in the higher-level design document\).

L402 could be configured for use on a per-backend-service basis or for all exposed services. As an example, the same L402 credential may commit to capabilities enabling a user to access both service `Y` and `X` from the same endpoint. This flexibility is afforded because all services are going to be gated by the same L402 proxy, which verifies all credentials for all backend services.

### Security Considerations

If a client's L402 is intercepted by Mallory, which is possible if the transmission is not encrypted in some way such as TLS, the L402 can be used by Mallory and the L402 proxy would not be able to distinguish this usage as illicit.

L402 authentication is also vulnerable to spoofing by counterfeit servers. If a client slightly mistypes the URL of a desired backend service, they become vulnerable to spoofing attacks if connecting to a server that maliciously stores their L402 and uses it for their own purposes.

Because L402 tokens support attenuation through caveats, this class of attack can be mitigated by cryptographically binding the token to client-specific details. For example, a server \(or the client itself, via self-attenuation\) could add caveats that restrict the token's validity to a particular IP address, TLS client certificate fingerprint, origin domain, or other client-identifying predicate. The server then verifies these caveats upon each request, ensuring a stolen token cannot be replayed from a different context. Each binding predicate carries its own tradeoffs: an IP caveat prevents use after a network change, while a TLS client cert fingerprint requires the client to maintain a stable key pair. Deployments should choose binding predicates appropriate to their threat model.

## HTTP Protocol Flow Specification

In this section, we specify the protocol for the HTTP portion of the L402 proxy.

### Server Flow

Upon receipt of a request for a URI of an L402-proxied backend service that lacks credentials or contains an L402 that is invalid or insufficient in some way:

1. The server SHOULD derive an economical price for the endpoint expressed in milli-satoshis \(1/1000th of a satoshi\) and create a [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) invoice `P` requesting that amount from its backing Lightning Node.

2. The server MUST create a new authentication token `T` for the client that will only become valid once the client pays the LN invoice.

   1. The token MUST commit to the payment hash `H` of the invoice `P`. This commitment is what provides in-band payment authentication: the server can verify that a client has paid using only the token and the preimage, without any backend lookup or additional state. Without this commitment, the server would need to maintain a large amount of state to validate all past, current, and future credentials.

3. The server MUST reply with a challenge using the 402 \(Payment Required\) status code. **Officially, in the HTTP RFC documentation, status code 402 is** [_**"reserved for future use"**_](https://tools.ietf.org/html/rfc7231#section-6.5.2) **-- but this document assumes the future has arrived.**

4. Alongside the 402 status code, the server MUST specify the `WWW-Authenticate` header \([_\[RFC7235\], Section 4.1_](https://tools.ietf.org/html/rfc7235#section-4.1)\) field to indicate the L402 authentication scheme and the token needed for the client to form a complete L402.

   A valid response header MUST take the form of:
   ```
   WWW-Authenticate: L402 version="0", token="T", invoice="P"
   ```

   Where:

   * `version`: A protocol version string. The current version is `"0"`. Clients MUST ignore unknown key=value parameters in the challenge header, to allow future extensions to the protocol \(e.g., alternative invoice formats\).

   * `token`: The authentication token, encoded as base64. The token MUST commit to the payment hash `H` of the invoice `P`.

   * `invoice`: A [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) payment request that the client MUST pay in order to obtain the preimage required to complete the `Authorization` header.

   An example L402 HTTP response resembles:

   ```text
   HTTP/1.1 402 Payment Required

   Date: Mon, 04 Feb 2014 16:50:53 GMT

   WWW-Authenticate: L402 version="0", token="AGIAJEemVQUTEyNCR0exk7ek90Cg==", invoice="lnbc1500n1pw5kjhmpp5fu6xhthlt2vucmzkx6c7wtlh2r625r30cyjsfqhu8rsx4xpz5lwqdpa2fjkzep6yptksct5yp5hxgrrv96hx6twvusycn3qv9jx7ur5d9hkugr5dusx6cqzpgxqr23s79ruapxc4j5uskt4htly2salw4drq979d7rcela9wz02elhypmdzmzlnxuknpgfyfm86pntt8vvkvffma5qc9n50h4mvqhngadqy3ngqjcym5a"
   ```

   where `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` is the token that the client MUST include for each of its authorized requests and `"lnbc1500n1pw5kjhmpp..."` is the BOLT 11 invoice the client must pay to reveal the preimage that must be included for each of its authorized requests.

Upon receiving an HTTP request with a valid [RFC-7235](https://datatracker.ietf.org/doc/html/rfc7235) `Authorization` header:

1. The server MUST verify the cryptographic authenticity and integrity of the credential.
   1. If a credential is invalid, then the server MUST return a 401 Unauthorized status code.
2. The server MUST parse out the `Authorization` header \(`token:preimage`\) into `T` the base64-encoded token, and `r` the hex-encoded payment preimage.
3. The server MUST verify that the BOLT 11 invoice tied to the token `T` has been paid.
   1. If the server committed to the payment hash `H` in the token `T`, then they can simply verify that `H = sha256(r)`.
   2. Otherwise, the server should verify that the invoice `P` has been paid in full.
4. If the invoice has not been paid in full, then the server MUST return a 401 Unauthorized header. Otherwise, the server SHOULD process the request itself, or pass it on to the proxied backend.

Regarding step 3.1 and 3.2, it's imperative that the server ensure that the invoice has been paid in full before processing the response, or proxying it to a backend. As stated above, by cryptographically committing to the payment hash, the server can implement fast stateless verification of the payment hash+preimage relation.

### Client Flow

Upon receiving a valid L402 `WWW-Authenticate` header the client:

1. Should verify that the contained BOLT 11 invoice doesn't request an excessive amount of Bitcoin.
   1. If so, then the client SHOULD abandon the HTTP request.
2. After validating the returned BOLT 11 invoice, the client MUST pay the invoice over the LN to receive the payment pre-image `r`.
3. The client then constructs a valid `Authorization` header with the following form:
   `Authorization: L402 <base64(token)>:<hex(preimage)>`
4. The client then re-issues the original HTTP request with the attached L402 `Authorization` header.

In other words, to receive authorization, the client:

* Pays the invoice from the server, thus revealing the invoice's preimage
* Constructs the L402 by concatenating the base64-encoded token\(s\), a single colon \(":"\), and the hex-encoded preimage.

Since the token and the preimage are both binary data encoded in an ASCII based format, there should be no problem with either containing control characters or colons \(see "CTL" in [_Appendix B.1 of \[RFC5234\]_](https://tools.ietf.org/html/rfc5234#appendix-B.1)\). If a user provides a token or preimage containing any of these characters, this is to be considered an invalid L402 and SHOULD result in a 402 and authentication information as specified above.

If a client wishes to send the token `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` \(already base64-encoded by the server\) and the preimage `"1234abcd1234abcd1234abcd"` \(already hex encoded by the payee's Lightning node\), they would use the following header field:

```text
Authorization: L402 AGIAJEemVQUTEyNCR0exk7ek90Cg==:1234abcd1234abcd1234abcd
```

## gRPC Protocol Specification

This section defines the gRPC variant of the L402 authentication scheme. While gRPC is itself transmitted over HTTP, it uses special trailing headers to delineate protocol-specific information. In addition, the [gRPC specification requires that a status code of 200 is _always_ sent in responses](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses). As a result, the protocol flow is modified slightly to always send a 200 OK status code, and instead insert the relevant L402 headers as gRPC trailing headers.

### Server Flow

The server flow when receiving a URI that lacks credentials or is invalid is identical to the HTTP protocol flow with the following modifications:

1. The server MUST reply with a challenge using the 200 OK status code.
2. Alongside the normal `WWW-Authenticate` header, the server MUST send the following trailing headers:

   * `Grpc-Message: payment required`
   * `Grpc-Status`: 13 # Internal

   As an example:
   ```text
   HTTP/2 200 OK
   Date: Mon, 04 Feb 2014 16:50:53 GMT
   Content-Type: application/grpc
   …
   Grpc-Message: payment required
   Grpc-Status: 13
   ```

### Client Flow

The gRPC flow for the client is identical to the HTTP flow, with one addition: the client MUST also transmit the final token as `Custom-Metadata`, with a key of "token" and value of the hex-encoded serialized token.

All other gRPC headers and trailers are required as normal; more information can be found in the [_gRPC over HTTP2 specification_](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md).

## Backwards Compatibility

The `L402` protocol was formerly known as `LSAT`. In order to preserve backwards compatibility with clients, anywhere `L402` is used within HTTP headers or requests, a valid protocol flow with the string `LSAT` should also be accepted.

Earlier versions of this specification used `macaroon` as the key name in the `WWW-Authenticate` challenge header \(e.g., `L402 macaroon="X", invoice="Y"`\) and in the gRPC `Custom-Metadata` key. To preserve backwards compatibility with deployed clients and servers:

* Servers SHOULD accept `macaroon=` in addition to `token=` when parsing client requests and `WWW-Authenticate` headers from upstream proxies.
* Clients SHOULD accept both `token=` and `macaroon=` when parsing `WWW-Authenticate` challenge headers from servers.
* Servers SHOULD accept the gRPC `Custom-Metadata` key "macaroon" in addition to "token".

The `version` parameter in the `WWW-Authenticate` header was introduced in this revision. Old clients that do not understand the `version` parameter will ignore it per the "unknown parameters MUST be ignored" rule. Servers SHOULD accept requests that omit the `version` parameter.
