# Macaroon Technical Specification

This document provides a complete technical specification for creating, encoding,
verifying, and attenuating L402 macaroons. It is intended as an implementor's
guide.

## Background

Macaroons are HMAC-chain based bearer credentials originally described in
[Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in
the Cloud](https://research.google/pubs/pub41892/). They consist of three
components:

1. **Identifier** — public data that maps to a secret root key
2. **Caveats** — a list of predicates that restrict the macaroon's authority
3. **Signature** — an HMAC chain computed from the root key, identifier, and
   all caveats

The signature is what makes macaroons tamper-proof: any modification to the
identifier or caveats invalidates the chain.

## Identifier Structure

The macaroon identifier is the public ID field that encodes static information
about the credential. All multi-byte integers are encoded in **big-endian** byte
order.

### Version 0 Format

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 2 bytes | `version` | uint16, currently `0` |
| 2 | 32 bytes | `payment_hash` | SHA-256 hash from the Lightning invoice |
| 34 | 32 bytes | `token_id` | 32 random bytes, unique per credential |

**Total size: 66 bytes.**

The `payment_hash` links the macaroon to its corresponding Lightning invoice.
The `token_id` provides a stable user identifier across macaroon rotations
\(e.g., tier upgrades that revoke and re-mint\).

### Encoding

```
identifier = version_bytes || payment_hash || token_id
```

Where `||` denotes concatenation and `version_bytes` is a 2-byte big-endian
uint16.

### Root Key

Each macaroon MUST have its own cryptographically random root key of **32 bytes**.
The root key MUST NOT be disclosed to clients. The mapping from identifier to
root key is maintained server-side; the SHA-256 hash of the identifier MAY be
used as the lookup key.

Root keys MUST be stored securely and reliably, as they are required for both
verification and revocation.

## Minting

To mint a new L402 macaroon:

1. Generate a 32-byte cryptographically random root key.
2. Generate a 32-byte cryptographically random token ID.
3. Obtain the payment hash `H` from the Lightning invoice.
4. Construct the identifier: `encode(version=0, H, token_id)`.
5. Create the macaroon: `macaroon.New(rootKey, identifier, location, V2)`.
6. Add any initial first-party caveats \(services, capabilities, constraints\).
7. Store the root key indexed by `sha256(identifier)`.
8. Return the macaroon \(base64-encoded\) alongside the invoice.

```
WWW-Authenticate: L402 version="0", token="<base64(macaroon)>", invoice="<bolt11>"
```

## Caveat Format

Caveats are string-encoded key-value pairs separated by a single `=` character:

```
condition=value
```

The `condition` uniquely identifies the caveat type and determines how it is
evaluated. The `value` provides the parameters for evaluation.

### Standard Caveat Types

#### Preimage Caveat

Added by the client after payment to bind the proof of payment to the macaroon.

| Condition | Value | Example |
|-----------|-------|---------|
| `preimage` | hex-encoded 32-byte preimage | `preimage=abcdef0123456789...` |

The preimage caveat is a first-party caveat. When present, the verifier checks
`sha256(preimage) == payment_hash` from the identifier.

#### Services Caveat

Encodes which services the macaroon is authorized to access.

| Condition | Value | Example |
|-----------|-------|---------|
| `services` | comma-separated `name:tier` pairs | `services=lightning_loop:0,pool:0` |

- Service names are ASCII strings.
- Tiers are uint8 values starting from `0` \(base tier\).
- Multiple `services` caveats form an attenuation chain: each MUST be a
  subset of the previous \(can only remove services, never add\).

#### Capabilities Caveat

Restricts which operations are permitted for a specific service.

| Condition | Value | Example |
|-----------|-------|---------|
| `<service>_capabilities` | comma-separated capability names | `lightning_loop_capabilities=loop_out,loop_in` |

- If absent, all capabilities of the service are permitted.
- Multiple capability caveats for the same service form an attenuation chain:
  each MUST be a subset of the previous.

#### Constraint Caveats

Service-specific restrictions on individual capabilities.

| Condition | Value | Example |
|-----------|-------|---------|
| `<capability>_<constraint>` | constraint-specific | `loop_out_monthly_volume_sats=200000000` |

Common constraints:

| Constraint | Value Type | Description |
|------------|-----------|-------------|
| `_monthly_volume_sats` | integer string | Maximum monthly volume in satoshis |
| `_valid_until` | unix timestamp string | Expiration time |

Multiple caveats of the same constraint MUST be increasingly restrictive \(lower
volume, earlier expiration\).

### Caveat Example

A base-tier Lightning Loop macaroon with 2 BTC monthly Loop Out volume:

```text
identifier:
    version = 0
    payment_hash = 163102a9c88fa4ec9ac9937b6f070bc3e27249a81ad7a05f398ac5d7d16f7bea
    token_id = fed74b3ef24820f440601eff5bfb42bef4d615c4948cec8aca3cb15bd23f1013
caveats:
    services = lightning_loop:0
    lightning_loop_capabilities = loop_out,loop_in
    loop_out_monthly_volume_sats = 200000000
```

## Attenuation \(Delegation\)

A macaroon holder can create a weaker version of their credential by adding
first-party caveats. This is the primary mechanism for delegation.

**Rules:**
- New caveats can only _restrict_, never _expand_ authority.
- For list-valued caveats \(services, capabilities\), new values MUST be a
  subset of the previous caveat's values.
- For numeric constraints, new values MUST be more restrictive \(lower volume,
  earlier expiration\).

**Example:** Restricting the Loop macaroon above to only Loop In with 1 BTC
monthly volume:

```text
# Added by the delegating party:
lightning_loop_capabilities = loop_in
loop_in_monthly_volume_sats = 100000000
```

The HMAC chain ensures these caveats cannot be removed by the recipient.

## Verification

Verification is a three-step process involving two parties: the **minter**
\(who holds the root key\) and the **authorizer** \(the target service\).

### Step 1: Signature Verification \(Minter\)

1. Look up the root key using `sha256(macaroon.Id())`.
2. Recompute the HMAC chain: starting from the root key and identifier, chain
   through each caveat to produce the terminal signature.
3. Compare the computed signature to `macaroon.Signature()`.
4. If they don't match, REJECT — the macaroon has been tampered with.
5. If no root key exists for the identifier, REJECT — the macaroon has been
   revoked.

### Step 2: Payment Verification \(Minter\)

1. Extract the preimage `r` from either:
   - The `preimage` caveat in the macaroon, OR
   - The `Authorization` header \(after the `:`\)
2. Decode the identifier to extract `payment_hash` \(`H`\).
3. Verify `H == sha256(r)`.
4. If verification fails, REJECT.

### Step 3: Caveat Verification \(Authorizer\)

Each caveat is evaluated against a **satisfier** — a function that understands
the caveat's condition and can evaluate it.

For each unique caveat condition present in the macaroon:

1. Find a satisfier registered for that condition.
2. If multiple caveats share the same condition, verify that each successive
   caveat is more restrictive than the previous \(`SatisfyPrevious`\).
3. Evaluate the final \(most restrictive\) caveat against the current request
   context \(`SatisfyFinal`\).
4. If any caveat fails, REJECT.

**Unknown caveats:** If no satisfier is registered for a caveat's condition,
the caveat MUST be skipped \(not rejected\). This allows macaroon holders to
add caveats for other applications without breaking verification by the
current service.

### Built-in Satisfiers

| Satisfier | Condition | SatisfyPrevious | SatisfyFinal |
|-----------|-----------|-----------------|--------------|
| Services | `services` | New list ⊆ previous list | Target service in list |
| Capabilities | `<svc>_capabilities` | New list ⊆ previous list | Target capability in list |
| Timeout | `<svc>_valid_until` | New timestamp ≤ previous | `now() < timestamp` |

## Revocation

To revoke a macaroon, delete its corresponding root key from storage. Without
the root key, the minter cannot verify the signature, and the macaroon is
permanently invalid.

Revocation is also used during tier upgrades: the old macaroon is revoked
and a new one is minted with updated capabilities.

## Serialization Formats

### Macaroon Binary Format

The `gopkg.in/macaroon.v2` library defines two binary formats:

- **V1**: Original format, wider compatibility.
- **V2**: Compact binary format, recommended for L402.

Use `macaroon.MarshalBinary()` and `macaroon.UnmarshalBinary()` for
serialization.

### Wire Format \(Token Storage\)

For client-side storage and transmission, the full token includes:

| Field | Size | Encoding |
|-------|------|----------|
| macaroon length | 4 bytes | uint32 big-endian |
| macaroon bytes | variable | `MarshalBinary()` output |
| payment hash | 32 bytes | raw bytes |
| preimage | 32 bytes | raw bytes \(zero if pending\) |
| amount paid | 8 bytes | uint64 big-endian \(msat\) |
| routing fee paid | 8 bytes | uint64 big-endian \(msat\) |
| time created | 8 bytes | int64 big-endian \(UnixNano\) |

### HTTP Header Encoding

- `WWW-Authenticate`: macaroon is base64-encoded
- `Authorization`: macaroon is base64-encoded, preimage is hex-encoded
- gRPC `Custom-Metadata` \("token" or "macaroon"\): macaroon is hex-encoded

## Reference Implementations

- [gopkg.in/macaroon.v2](https://pkg.go.dev/gopkg.in/macaroon.v2) — Go macaroon library
- [gopkg.in/macaroon-bakery.v2](https://pkg.go.dev/gopkg.in/macaroon-bakery.v2) — Higher-level macaroon services
- [Aperture L402 package](https://github.com/lightninglabs/aperture/tree/master/l402) — Reference L402 implementation
