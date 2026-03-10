# L402 Agent Specification

Token-efficient reference for L402 protocol integration. Version 0.

## Core Concept

L402 = HTTP 402 + Lightning invoice + auth token. Client pays invoice, gets
preimage, presents `(token, preimage)` to authenticate. Stateless verification:
token commits to payment hash H, server checks `H == sha256(preimage)`.

## Headers

**Challenge** (server → client):
```
HTTP/1.1 402 Payment Required
WWW-Authenticate: L402 version="0", token="<base64>", invoice="<bolt11>"
```

**Authorization** (client → server):
```
Authorization: L402 <base64(token)>:<hex(preimage)>
```

Multiple tokens: comma-separate base64 values before the colon.

## Protocol Flow

```
Client → Server:  GET /resource
Server → Client:  402 + WWW-Authenticate: L402 version="0", token="T", invoice="P"
Client → LN:      pay(P) → preimage r
Client → Server:  GET /resource + Authorization: L402 T:r
Server:           verify H == sha256(r), verify token integrity
Server → Client:  200 OK + resource
```

## Token Requirements

- MUST commit to payment hash H of invoice P
- MUST be verifiable without backend state (stateless)
- RECOMMENDED format: macaroons (HMAC chain, supports caveats/attenuation)
- MAY use any format satisfying above constraints

## Macaroon Identifier (when using macaroons)

Big-endian encoding:

| Field | Size | Description |
|-------|------|-------------|
| version | 2 bytes | uint16, currently 0 |
| payment_hash | 32 bytes | sha256 hash from invoice |
| user_id | 32 bytes | random, tracks user across token rotations |

## Caveats (macaroon attenuation)

Key=value strings. Three standard types:

- `services=name:tier[,name:tier,...]` — allowed services
- `<service>_capabilities=cap1,cap2` — allowed operations
- `<cap>_<constraint>=value` — operation constraints

Each additional caveat of same key MUST restrict further (never widen).

## gRPC Variant

Same auth scheme. Differences:
- Server MUST return HTTP 200 (gRPC requirement)
- Add trailers: `Grpc-Message: payment required`, `Grpc-Status: 13`
- Client sends token as `Custom-Metadata` key "token"

## Backwards Compatibility

- Accept `LSAT` anywhere `L402` appears
- Accept `macaroon=` in addition to `token=` in headers
- Accept missing `version` parameter (pre-versioning clients)

## Credential Lifecycle

- Reuse until revoked (expiry, usage limit, tier upgrade)
- Revoke by deleting root key (macaroons) or invalidating token
- Upgrade: revoke old token, mint new with expanded capabilities

## Implementation Checklist

Server:
1. Mint token committing to payment hash
2. Return 402 + WWW-Authenticate with version, token, invoice
3. On auth request: verify token integrity, check `H == sha256(r)`
4. Accept both `L402` and `LSAT` scheme names
5. Accept both `token=` and `macaroon=` parameter names

Client:
1. Parse WWW-Authenticate, extract token + invoice
2. Validate invoice amount is acceptable
3. Pay invoice, obtain preimage
4. Send `Authorization: L402 <token>:<preimage>`
5. Cache credential for reuse
