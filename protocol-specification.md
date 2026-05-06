# L402 Protocol Specification

## 1. Introduction

This document specifies the L402 authentication scheme for HTTP and gRPC.
L402 combines the [HTTP 402 Payment Required](https://tools.ietf.org/html/rfc7231#section-6.5.2)
status code with [Lightning Network](https://github.com/lightning/bolts)
invoice payments to create a challenge-response protocol for paid API access.
A server issues a _challenge_ containing an authentication credential and a
Lightning invoice; the client pays the invoice to obtain a preimage, then
presents the credential and preimage together as proof of payment.

The credential is a [macaroon](https://research.google/pubs/pub41892/) (an
HMAC-chain bearer token) that cryptographically commits to the invoice's
payment hash. This binding enables stateless verification: the server checks
`H == sha256(preimage)` against the hash embedded in the macaroon, with no
database lookup required. See [Macaroon Minting & Verification](docs/macaroons.md)
for a full treatment of the macaroon format.

For higher-level motivation and use cases, see the
[Introduction](docs/introduction.md).

## 2. Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/rfc2119)
\([RFC 2119](https://tools.ietf.org/html/rfc2119),
[RFC 8174](https://tools.ietf.org/html/rfc8174)\) when, and only when, they
appear in all capitals, as shown here.

## 3. Terminology

**L402 Credential.** A `<macaroon>:<preimage>` pair transmitted in the HTTP
`Authorization` header. The macaroon is base64-encoded; the preimage is
hex-encoded. This is the artifact a client presents to prove payment.

**Macaroon.** An HMAC-chain bearer credential minted by the server. The
macaroon's identifier commits to the payment hash of a Lightning invoice.
Macaroons support _caveats_ (restrictions) and _attenuation_ (delegation with
reduced authority). See the [Macaroon Technical Specification](macaroon-spec.md)
for construction details.

**Preimage.** The 32-byte value `r` such that `sha256(r)` equals the payment
hash of the Lightning invoice. Possession of the preimage proves the invoice
was paid.

**Payment Hash.** The SHA-256 hash `H` of the preimage, embedded in both the
Lightning invoice and the macaroon identifier. The binding `H == sha256(r)` is
the core verification primitive.

**Challenge.** The `WWW-Authenticate: L402 ...` header returned by the server
alongside HTTP 402, containing a macaroon and a Lightning invoice.

**Caveat.** A restriction appended to a macaroon's HMAC chain. Caveats can
encode service access, capabilities, expiration, volume limits, and other
constraints. Each successive caveat can only _narrow_ the macaroon's authority,
never widen it.

## 4. Protocol Overview

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant LN as Lightning Network

    C->>S: GET /resource
    S-->>C: 402 Payment Required<br/>WWW-Authenticate: L402 macaroon="M", invoice="P"
    C->>LN: pay(P)
    LN-->>C: preimage r
    C->>S: GET /resource<br/>Authorization: L402 M:r
    S->>S: verify macaroon, check H == sha256(r)
    S-->>C: 200 OK + resource
```

### 4.1. Status Code Usage

| Condition | Status | Response |
|-----------|--------|----------|
| Resource requires payment, no credential provided | 402 | `WWW-Authenticate` challenge with macaroon + invoice |
| Credential present but macaroon invalid or tampered | 401 | Unauthorized |
| Credential present but preimage does not match payment hash | 401 | Unauthorized |
| Credential valid, access granted | 200 | Requested resource |

The 402 status code is used exclusively for the initial payment challenge. Once
a client presents a credential (valid or not), the server MUST respond with 401
if verification fails, not 402. This distinction lets clients differentiate
"you need to pay" from "your credential is broken."

## 5. The L402 Authentication Scheme

The L402 scheme is registered under the HTTP Authentication Framework specified
in [RFC 7235](https://tools.ietf.org/html/rfc7235). The scheme name is "L402"
(case-insensitive).

### 5.1. Challenge (WWW-Authenticate)

When a server requires payment for a resource, it MUST respond with HTTP 402
and include a `WWW-Authenticate` header of the form:

```
WWW-Authenticate: L402 macaroon="<base64>", invoice="<bolt11>"
```

**Parameters:**

- `macaroon` (REQUIRED): The authentication macaroon, base64-encoded
  ([RFC 4648](https://tools.ietf.org/html/rfc4648)). The macaroon MUST commit
  to the payment hash `H` of the invoice `P` in its identifier
  (see [Macaroon Technical Specification](macaroon-spec.md), Section
  "Identifier Structure").

- `invoice` (REQUIRED): A [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
  payment request. The client MUST pay this invoice to obtain the preimage
  required to complete the `Authorization` header.

**Example:**

```text
HTTP/1.1 402 Payment Required
Date: Mon, 04 Feb 2014 16:50:53 GMT
WWW-Authenticate: L402 macaroon="AGIAJEemVQUTEyNCR0exk7ek90Cg==", invoice="lnbc1500n1pw5kjhmpp5fu6xhthlt2vucmzkx6c7wtlh2r625r30cyjsfqhu8rsx4xpz5lwqdpa2fjkzep6yptksct5yp5hxgrrv96hx6twvusycn3qv9jx7ur5d9hkugr5dusx6cqzpgxqr23s79ruapxc4j5uskt4htly2salw4drq979d7rcela9wz02elhypmdzmzlnxuknpgfyfm86pntt8vvkvffma5qc9n50h4mvqhngadqy3ngqjcym5a"
```

Where `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` is the macaroon the client must
include in each authorized request, and `"lnbc1500n1pw5kjhmpp..."` is the
BOLT 11 invoice the client must pay to reveal the preimage.

### 5.2. Credentials (Authorization)

The L402 credential is transmitted in the `Authorization` header using the
following syntax:

```
Authorization: L402 <base64(macaroon)>:<hex(preimage)>
```

Where:

- The macaroon is base64-encoded per [RFC 4648](https://tools.ietf.org/html/rfc4648).
  Multiple macaroons are base64-encoded individually and comma-separated before
  the colon.
- The preimage is hex-encoded per [RFC 3548, Section 6](https://tools.ietf.org/html/rfc3548#section-6).

This syntax is comparable to the
["token68" syntax](https://tools.ietf.org/html/rfc7235#section-2.1) used for
HTTP Basic auth.

**Example:**

```text
Authorization: L402 AGIAJEemVQUTEyNCR0exk7ek90Cg==:1234abcd1234abcd1234abcd
```

Since the macaroon and preimage are both binary data encoded in ASCII, there is
no issue with control characters or colons (see "CTL" in
[Appendix B.1 of RFC 5234](https://tools.ietf.org/html/rfc5234#appendix-B.1)).
If a client provides a macaroon or preimage containing control characters, the
server MUST treat it as an invalid L402 and respond with 401.

### 5.3. Grammar

```
l402-challenge   = "L402" 1*SP l402-params
l402-params      = macaroon-param "," SP invoice-param
macaroon-param   = "macaroon" "=" quoted-string
invoice-param    = "invoice" "=" quoted-string

l402-credential  = "L402" 1*SP macaroons ":" preimage
macaroons        = base64 *("," base64)
preimage         = 1*HEXDIG
base64           = 1*( ALPHA / DIGIT / "+" / "/" / "=" )
```

## 6. HTTP Protocol Flow

### 6.1. Server Flow

Upon receipt of a request for a resource that requires payment and lacks a
valid L402 credential:

1. The server SHOULD derive a price for the resource expressed in
   millisatoshis (1/1000th of a satoshi) and create a
   [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
   invoice `P` requesting that amount from its backing Lightning node.

2. The server MUST mint a new macaroon `M` for the client. The macaroon MUST
   commit to the payment hash `H` of the invoice `P` in its identifier. This
   commitment enables in-band payment verification: the server can confirm a
   client has paid using only the macaroon and preimage, with no additional
   state or backend lookup.

3. The server MUST reply with HTTP 402 (Payment Required). Officially, the
   HTTP specification marks 402 as
   ["reserved for future use"](https://tools.ietf.org/html/rfc7231#section-6.5.2),
   but this document assumes the future has arrived.

4. The server MUST include a `WWW-Authenticate` header per Section 5.1
   containing the macaroon and invoice.

Upon receiving a request with an `Authorization: L402` header:

1. The server MUST verify the cryptographic integrity of the macaroon (HMAC
   chain verification against the root key). If the macaroon is invalid, the
   server MUST return 401 Unauthorized.

2. The server MUST parse the `Authorization` header into the base64-encoded
   macaroon `M` and the hex-encoded preimage `r`.

3. The server MUST verify that the invoice tied to the macaroon has been paid:
   1. If the server committed to the payment hash `H` in the macaroon, it can
      verify that `H == sha256(r)`. This is the RECOMMENDED approach as it
      enables stateless verification.
   2. Otherwise, the server SHOULD verify that the invoice `P` has been paid
      in full via its Lightning node.

4. If verification fails, the server MUST return 401 Unauthorized. Otherwise,
   the server SHOULD process the request or forward it to the proxied backend.

It is imperative that the server ensure payment before processing the request
or forwarding it to a backend. By cryptographically committing to the payment
hash in the macaroon, the server can perform fast, stateless verification of
the payment hash + preimage relation.

### 6.2. Client Flow

Upon receiving a `WWW-Authenticate: L402` challenge:

1. The client SHOULD verify that the BOLT 11 invoice does not request an
   excessive amount of Bitcoin. If the amount exceeds a configured threshold,
   the client SHOULD abandon the request.

2. After validating the invoice, the client MUST pay the invoice over the
   Lightning Network to obtain the payment preimage `r`.

3. The client MUST construct an `Authorization` header per Section 5.2:
   ```
   Authorization: L402 <base64(macaroon)>:<hex(preimage)>
   ```

4. The client MUST re-issue the original HTTP request with the `Authorization`
   header attached.

## 7. Wallet and Payment Processor Requirements

The protocol flows in Sections 6 and 8 specify obligations for the L402 server
(the resource gatekeeper) and the L402 client (the entity constructing the
`Authorization` header). The proof chain also depends on a third actor: the
**wallet or payment processor** that settles the underlying Lightning invoice
on the client's behalf and surfaces the resulting preimage `r` back to the
client.

In typical Lightning Network routing, `r` propagates to the client's wallet
automatically through the HTLC settlement chain; no normative behavior on the
wallet's part is required. However, when both the payee and payer are hosted
by the same custodial Lightning service, the service may settle the payment
internally — crediting the payee's account and debiting the payer's account
without routing the payment over the Lightning Network. In this scenario, no
HTLC chain forms, `r` is generated at invoice creation and held by the service
throughout, and `r` is never propagated to the client by the underlying
Lightning protocol.

To preserve L402 compatibility regardless of settlement path:

1. A Lightning wallet or payment processor MUST surface the payment preimage
   `r` to the client for any settled invoice, regardless of whether settlement
   occurred via HTLC routing or via internal accounting on a single custodial
   service.

2. A wallet or payment processor that omits `r` from the payment-confirmation
   response for an internally-settled invoice SHOULD be considered
   non-compliant with this specification for L402 authentication purposes,
   since the client cannot construct a valid `Authorization` header without
   `r`.

3. A wallet or payment processor MUST NOT release the preimage `r` to the
   client prior to settlement of the corresponding invoice. In standard
   Lightning routing this is enforced by the HTLC protocol; in
   internal-settlement implementations it MUST be enforced by the wallet or
   payment processor's accounting logic. The L402 server's stateless
   `H == sha256(r)` check (Section 6.1) assumes that possession of `r` by the
   client implies the invoice has been settled.

These requirements apply equally to the HTTP flow (Section 6) and the gRPC
flow (Section 8). They impose no requirement on the underlying Lightning
settlement path; they impose a requirement only on what the wallet or payment
processor surfaces to the client after settlement.

## 8. gRPC Protocol Flow

gRPC is transmitted over HTTP/2 but uses special trailing headers for
protocol-specific information. The
[gRPC specification](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses)
requires a status code of 200 in all responses. As a result, the L402 gRPC
flow is modified to always return 200 OK at the HTTP level and instead convey
the payment challenge via gRPC trailing headers.

### 8.1. Server Flow

The server flow is identical to the HTTP flow (Section 6.1) with the following
modifications:

1. The server MUST reply with HTTP 200 OK (not 402).

2. The server MUST encode the L402 challenge as a serialized gRPC Status proto
   in the `grpc-status-details-bin` trailing header. The deserialized proto
   contains the macaroon and invoice:

   ```javascript
   {
       code: 402,
       message: "missing L402",
       details: {
           type_url: "type.googleapis.com/google.rpc.QuotaFailure",
           value: {
               macaroon: "<macaroon>",
               invoice: "<invoice>"
           }
       }
   }
   ```

3. The server MUST include the following trailing headers:
   - `Grpc-Message: missing L402`
   - `Grpc-Status: 402`

   Example:

   ```text
   HTTP/2 200 OK
   Date: Mon, 04 Feb 2014 16:50:53 GMT
   Content-Type: application/grpc
   ...
   Grpc-Message: missing L402
   Grpc-Status: 402
   Grpc-Status-Details-Bin: CJIDEgxtaXNzaW5nIExTQVQaeQ...
   ```

The L402 proxy determines whether a request is gRPC by checking whether the
`Content-Type` header begins with `application/grpc`. The proxy MUST be HTTP/2
compatible, since gRPC clients expect an HTTP/2-speaking server.

### 8.2. Client Flow

The gRPC client flow is identical to the HTTP flow (Section 6.2). Once the
client has deserialized the proto and extracted the macaroon and invoice, it
pays the invoice and constructs the L402 credential identically to the HTTP
case (concatenating base64-encoded macaroon, colon, hex-encoded preimage).

Other gRPC headers and trailers are required as normal; see the
[gRPC over HTTP2 specification](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
for details.

## 9. Credential Reuse and Revocation

L402 credentials are intended for reuse. A client SHOULD cache and reuse its
credential until the server rejects it with a new 402 challenge. An L402 can
be scoped to a single backend service or apply across all services behind the
same L402 proxy, since the proxy verifies all macaroons for all backends.

Possible revocation conditions include:

- Expiry date encoded as a caveat
- Exceeded usage count
- Volume of usage in a time period that necessitates a tier upgrade
- Explicit server-side revocation (by deleting the root key)

When a credential is revoked, the server issues a fresh 402 challenge and the
client repeats the payment flow.

## 10. Security Considerations

### 10.1. Transport Security

L402 credentials are bearer tokens. The macaroon and preimage are transmitted
as cleartext in HTTP headers and MUST be protected by TLS. Implementations
MUST use TLS 1.2 ([RFC 5246](https://tools.ietf.org/html/rfc5246)) or later;
TLS 1.3 ([RFC 8446](https://tools.ietf.org/html/rfc8446)) is RECOMMENDED.

Servers MUST NOT issue L402 challenges over unencrypted HTTP. Clients MUST NOT
send L402 credentials over unencrypted HTTP.

### 10.2. Credential Interception

If a client's L402 is intercepted by an attacker (e.g., via a compromised TLS
termination point), the attacker can reuse the credential. The L402 proxy would
not be able to distinguish this usage as illicit, since the credential is a
bearer token.

### 10.3. Spoofing by Counterfeit Servers

L402 is vulnerable to spoofing if a client connects to a malicious server
(e.g., by mistyping a URL). The malicious server could store the client's L402
and reuse it.

Because macaroons support attenuation through caveats, this class of attack can
be mitigated by binding the credential to client-specific details. A server (or
the client itself, via self-attenuation) could add caveats restricting validity
to a particular IP address, TLS client certificate fingerprint, origin domain,
or other client-identifying predicate. The server then verifies these caveats
on each request, ensuring a stolen credential cannot be replayed from a
different context.

Each binding predicate carries its own tradeoffs: an IP caveat prevents use
after a network change; a TLS client cert fingerprint requires the client to
maintain a stable key pair. Deployments SHOULD choose binding predicates
appropriate to their threat model.

### 10.4. Replay Protection

The macaroon itself does not inherently prevent replay. Replay protection comes
from the caveat and revocation mechanisms: expiry caveats, usage-count tracking,
and root key deletion all limit the window in which a stolen credential is
useful.

### 10.5. Amount Verification

Clients MUST verify that the invoice amount is reasonable for the requested
resource before paying. Malicious servers could request arbitrarily large
payments. Client implementations SHOULD enforce a configurable maximum payment
threshold.

## 11. Backwards Compatibility

The L402 protocol was formerly known as LSAT. To preserve backwards
compatibility with deployed clients and servers:

- Servers SHOULD send both `LSAT` and `L402` scheme names in `WWW-Authenticate`
  challenge headers. The `LSAT` header SHOULD appear first for compatibility
  with older client implementations.
- Clients and servers MUST accept both `LSAT` and `L402` in `Authorization`
  headers.

## 12. References

- [RFC 2119: Key words for use in RFCs](https://tools.ietf.org/html/rfc2119)
- [RFC 4648: Base Encodings](https://tools.ietf.org/html/rfc4648)
- [RFC 5234: ABNF](https://tools.ietf.org/html/rfc5234)
- [RFC 5246: TLS 1.2](https://tools.ietf.org/html/rfc5246)
- [RFC 7235: HTTP Authentication](https://tools.ietf.org/html/rfc7235)
- [RFC 7231: HTTP Semantics (402)](https://tools.ietf.org/html/rfc7231#section-6.5.2)
- [RFC 8174: RFC 2119 Clarification](https://tools.ietf.org/html/rfc8174)
- [RFC 8446: TLS 1.3](https://tools.ietf.org/html/rfc8446)
- [BOLT 11: Invoice Protocol](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
- [Macaroons: Cookies with Contextual Caveats (Google Research)](https://research.google/pubs/pub41892/)
- [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
