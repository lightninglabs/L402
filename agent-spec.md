# L402 Agent Specification

Token-efficient reference for L402 protocol integration.

## Core Concept

L402 = HTTP 402 + Lightning invoice + macaroon. Client pays invoice, gets
preimage, presents `(macaroon, preimage)` to authenticate. Stateless
verification: macaroon commits to payment hash H, server checks
`H == sha256(preimage)`.

## Headers

**Challenge** (server → client):
```
HTTP/1.1 402 Payment Required
WWW-Authenticate: L402 macaroon="<base64>", invoice="<bolt11>"
```

**Authorization** (client → server):
```
Authorization: L402 <base64(macaroon)>:<hex(preimage)>
```

Multiple macaroons: comma-separate base64 values before the colon.

## Protocol Flow

```
Client → Server:  GET /resource
Server → Client:  402 + WWW-Authenticate: L402 macaroon="M", invoice="P"
Client → LN:      pay(P) → preimage r
Client → Server:  GET /resource + Authorization: L402 M:r
Server:           verify H == sha256(r), verify macaroon integrity
Server → Client:  200 OK + resource
```

## Macaroon Identifier

Big-endian encoding:

| Field | Size | Description |
|-------|------|-------------|
| version | 2 bytes | uint16, currently 0 |
| payment_hash | 32 bytes | sha256 hash from invoice |
| token_id | 32 bytes | random, tracks user across macaroon rotations |

## Caveats (macaroon attenuation)

Key=value strings. Three standard types:

- `services=name:tier[,name:tier,...]` (allowed services)
- `<service>_capabilities=cap1,cap2` (allowed operations)
- `<cap>_<constraint>=value` (operation constraints)

Each additional caveat of same key MUST restrict further (never widen).

## gRPC Variant

Same auth scheme. Differences:
- Server MUST return HTTP 200 (gRPC requirement)
- Challenge encoded in `grpc-status-details-bin` as serialized gRPC Status proto
- Client sends macaroon as `Custom-Metadata` key "macaroon"

## Backwards Compatibility

- Accept `LSAT` anywhere `L402` appears

## Credential Lifecycle

- Reuse until revoked (expiry, usage limit, tier upgrade)
- Revoke by deleting root key
- Upgrade: revoke old macaroon, mint new with expanded capabilities

## Implementation Checklist

Server:
1. Mint macaroon committing to payment hash
2. Return 402 + WWW-Authenticate with macaroon, invoice
3. On auth request: verify macaroon integrity, check `H == sha256(r)`
4. Accept both `L402` and `LSAT` scheme names

Client:
1. Parse WWW-Authenticate, extract macaroon + invoice
2. Validate invoice amount is acceptable
3. Pay invoice, obtain preimage
4. Send `Authorization: L402 <macaroon>:<preimage>`
5. Cache credential for reuse
