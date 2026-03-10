# L402: Lightning HTTP 402 Protocol

L402 is an open protocol for paying for and authenticating access to APIs and
services over the internet using the Lightning Network. It brings to life the
long-dormant HTTP `402 Payment Required` status code by combining cryptographic
authentication tokens with Lightning Network micropayments.

## How It Works

1. A client requests a paid resource from a server.
2. The server responds with `402 Payment Required`, including an authentication
   token and a Lightning invoice in the `WWW-Authenticate` header.
3. The client pays the invoice over Lightning, receiving a payment preimage as
   proof of payment.
4. The client re-sends the request with the token and preimage in the
   `Authorization` header.
5. The server verifies the credential and serves the resource.

The token cryptographically commits to the payment hash of the invoice, so the
server can verify payment using only the token and preimage — no database
lookups or session state required.

```
WWW-Authenticate: L402 version="0", token="<base64>", invoice="<bolt11>"
Authorization:    L402 <base64(token)>:<hex(preimage)>
```

## Why L402?

**No accounts, no passwords.** Users pay a Lightning invoice and receive a
cryptographic credential. No email, no sign-up form, no personal data collected.

**Pay-as-you-go.** Instead of choosing between free tiers and monthly
subscriptions, users pay for exactly what they use. A single API call can cost
fractions of a cent.

**Programmable credentials.** When using macaroons as the token format (the
recommended default), credentials support attenuation: a token holder can
create a weaker version of their credential to share with others, restricting
access to specific services, capabilities, or usage limits.

**Stateless verification.** Servers verify credentials using only the token and
preimage — no centralized database needed. This makes L402 naturally suited for
distributed systems and microservice architectures.

## Agents and Agentic Commerce

L402 is uniquely suited for the emerging world of AI agents and autonomous
software that needs to discover, evaluate, and pay for services without human
intervention. Because L402 credentials are:

- **Machine-readable** — structured headers that any HTTP client can parse
- **Self-contained** — no out-of-band registration or OAuth flows needed
- **Instantly obtainable** — pay an invoice, get a credential, all in one
  HTTP round-trip
- **Programmatically attenuable** — agents can delegate scoped sub-credentials
  to other agents

...they provide the ideal authentication and payment primitive for agent-to-agent
and agent-to-service interactions. An AI agent can autonomously discover an API,
pay for access with Lightning, and immediately start making authenticated
requests — all without a human in the loop.

This makes L402 a foundational building block for agentic commerce: a world
where software agents transact with services and each other using real money
over open payment rails.

## Token Format

L402 is token-format agnostic — any token that commits to a payment hash works.
[Macaroons](macaroons.md) (HMAC-chain bearer credentials) are the recommended
format because they support:

- Delegation and attenuation through caveats
- Stateless verification via HMAC chains
- Service-level access control and tier encoding

See the [Macaroon Minting & Verification](macaroons.md) chapter for the full
specification.

## What's New

This revision of the L402 specification introduces several key updates aligned
with [bLIP-0026](https://github.com/lightning/blips/blob/master/blip-0026.md):

- **Token generalization** — The protocol is now token-format agnostic. The
  `WWW-Authenticate` header uses `token=` instead of the former `macaroon=`.
  Macaroons remain the recommended format.
- **Version system** — A `version="0"` parameter is now included in the
  `WWW-Authenticate` challenge header, enabling future protocol evolution.
- **Backwards compatibility** — Explicit rules for accepting both `L402`/`LSAT`
  scheme names and `token=`/`macaroon=` parameter names.
- **Enhanced security guidance** — Expanded security considerations covering
  caveat-based token binding (IP, TLS fingerprint, origin domain).
- **Agent specification** — A new [agent-spec.md](agent-spec.md) provides the
  complete protocol in ~560 tokens (~420 words), designed for AI agent context
  windows.

## Specification

* [Introduction](introduction.md) — Motivation and high-level overview
* [Authentication Flow](authentication-flow.md) — Step-by-step walkthrough with diagrams
* [Protocol Specification](protocol-specification.md) — HTTP and gRPC protocol details
* [Macaroon Minting & Verification](macaroons.md) — Recommended token format overview
* [Macaroon Technical Specification](macaroon-spec.md) — Detailed create/verify/attenuate implementation guide
* [Agent Specification](agent-spec.md) — Complete protocol in ~560 tokens for AI agent integration

## Implementations

* [Aperture](https://github.com/lightninglabs/aperture) — gRPC/HTTP authentication reverse proxy using L402
* [lsat-js](https://github.com/Tierion/lsat-js) — JavaScript utility library for working with L402 credentials
* [boltwall](https://github.com/tierion/boltwall) — Node.js middleware-based authentication using L402
* [Fewsats](https://www.fewsats.com/) — L402-compatible API marketplace and CLI tools

## External Links

* [Builder's Guide Documentation](https://docs.lightning.engineering/the-lightning-network/l402)
* [Macaroons: Cookies with Contextual Caveats (Google Research)](https://research.google/pubs/pub41892/)
* [HTTP/1.1 RFC, Section 6.5.2: 402 Payment Required](https://tools.ietf.org/html/rfc7231#section-6.5.2)
* [bLIP-0026: L402 Protocol Specification](https://github.com/lightning/blips/blob/master/blip-0026.md)
