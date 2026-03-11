# Macaroon Technical Specification

This document provides a complete technical specification for creating, encoding,
verifying, and attenuating L402 macaroons. It is intended as an implementor's
guide — sufficient to build a macaroon library from scratch without reference to
any existing implementation.

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

## HMAC Chain Construction

The core cryptographic primitive is **HMAC-SHA256**. A macaroon's signature is
the result of iteratively HMACing each component in sequence, where each step's
output becomes the next step's key.

### Notation

- `HMAC(key, message)` denotes HMAC-SHA256 with the given key and message.
- `||` denotes byte concatenation.

### Initial Signature

Given a 32-byte `root_key` and a byte string `identifier`:

```
sig_0 = HMAC(root_key, identifier)
```

If the macaroon has no caveats, `sig_0` is the final signature.

### Chaining Caveats

Each first-party caveat is a byte string `caveat_id`. When a caveat is appended
to the chain, the previous signature is used as the HMAC key:

```
sig_1 = HMAC(sig_0, caveat_id_1)
sig_2 = HMAC(sig_1, caveat_id_2)
...
sig_n = HMAC(sig_{n-1}, caveat_id_n)
```

The macaroon's final **signature** is `sig_n` (or `sig_0` if there are no
caveats). This is the value stored in the macaroon and transmitted to the
verifier.

### Why the Chain Works

Because HMAC is a one-way function:

- **Tamper-proof:** Changing any caveat or the identifier produces a different
  chain. Without the root key, an attacker cannot recompute the correct
  signature.
- **Append-only:** Anyone holding a macaroon with signature `sig_k` can compute
  `sig_{k+1} = HMAC(sig_k, new_caveat)` to add a caveat. But they cannot remove
  existing caveats or recover `sig_{k-1}` from `sig_k`, because HMAC is not
  invertible.

### Worked Example

```
root_key   = 0x00...00  (32 zero bytes, for illustration)
identifier = "example-id"

sig_0 = HMAC(root_key, "example-id")
      = 0x3c5d... (32 bytes)

caveat_1 = "account = 12345"
sig_1 = HMAC(sig_0, "account = 12345")
      = 0x8a2f... (32 bytes)

caveat_2 = "valid_until = 1700000000"
sig_2 = HMAC(sig_1, "valid_until = 1700000000")
      = 0xb71c... (32 bytes)

Final macaroon signature = sig_2
```

A verifier with access to `root_key` recomputes this exact chain from
`(root_key, identifier, [caveat_1, caveat_2])` and checks that the result
matches the signature in the presented macaroon.

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

Where `version_bytes` is a 2-byte big-endian uint16.

### Root Key

Each macaroon MUST have its own cryptographically random root key of **32 bytes**.
The root key MUST NOT be disclosed to clients. The mapping from identifier to
root key is maintained server-side; the SHA-256 hash of the identifier MAY be
used as the lookup key.

Root keys MUST be stored securely and reliably, as they are required for both
verification and revocation.

## Minting

To mint a new L402 macaroon:

1. Generate a 32-byte cryptographically random `root_key`.
2. Generate a 32-byte cryptographically random `token_id`.
3. Obtain the `payment_hash` from the Lightning invoice.
4. Construct the identifier: `version_bytes || payment_hash || token_id`
   (66 bytes total).
5. Compute the initial signature: `sig_0 = HMAC(root_key, identifier)`.
6. For each initial first-party caveat `c_i` (services, capabilities,
   constraints), extend the chain: `sig_i = HMAC(sig_{i-1}, c_i)`.
7. Assemble the macaroon: `(identifier, [c_1, ..., c_n], sig_n)`.
8. Store the root key indexed by `sha256(identifier)`.
9. Return the macaroon (serialized and base64-encoded) alongside the invoice.

```
WWW-Authenticate: L402 version="0", token="<base64(macaroon)>", invoice="<bolt11>"
```

### Pseudocode

```
function mint_macaroon(payment_hash, caveats):
    root_key  = random_bytes(32)
    token_id  = random_bytes(32)
    version   = uint16_big_endian(0)
    identifier = version || payment_hash || token_id

    sig = HMAC-SHA256(root_key, identifier)
    for caveat in caveats:
        sig = HMAC-SHA256(sig, caveat.encode())

    store(sha256(identifier) -> root_key)

    return Macaroon{
        identifier: identifier,
        caveats:    caveats,
        signature:  sig,
    }
```

## Caveat Format

Caveats are UTF-8 string-encoded key-value pairs separated by a single `=`
character. The caveat's raw byte encoding (its UTF-8 string representation) is
what gets HMACed into the signature chain.

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

The HMAC chain for this macaroon is:

```
sig_0 = HMAC(root_key, identifier_bytes)
sig_1 = HMAC(sig_0, "services=lightning_loop:0")
sig_2 = HMAC(sig_1, "lightning_loop_capabilities=loop_out,loop_in")
sig_3 = HMAC(sig_2, "loop_out_monthly_volume_sats=200000000")

macaroon.signature = sig_3
```

## Attenuation \(Delegation\)

A macaroon holder can create a weaker version of their credential by appending
first-party caveats. This is the primary mechanism for delegation.

To attenuate, the holder computes a new signature by extending the chain using
the macaroon's current signature as the HMAC key:

```
new_sig = HMAC(old_sig, new_caveat)
```

The holder does NOT need the root key to do this — only the current signature,
which is embedded in the macaroon they already possess.

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

After attenuation, the chain extends:

```
sig_4 = HMAC(sig_3, "lightning_loop_capabilities=loop_in")
sig_5 = HMAC(sig_4, "loop_in_monthly_volume_sats=100000000")

attenuated_macaroon.signature = sig_5
```

The HMAC chain ensures these caveats cannot be removed by the recipient: doing
so would require inverting HMAC to recover `sig_3` from `sig_5`, which is
computationally infeasible.

## Verification

Verification is a three-step process involving two parties: the **minter**
\(who holds the root key\) and the **authorizer** \(the target service\).

### Step 1: Signature Verification \(Minter\)

The minter recomputes the HMAC chain from scratch and compares it to the
presented signature:

1. Extract the identifier from the macaroon.
2. Look up the root key using `sha256(identifier)` as the storage key.
3. If no root key exists, REJECT — the macaroon was never minted or has been
   revoked.
4. Recompute the chain:
   ```
   sig = HMAC(root_key, identifier)
   for each caveat in macaroon.caveats:
       sig = HMAC(sig, caveat)
   ```
5. Compare the computed `sig` to the macaroon's signature field using a
   constant-time comparison.
6. If they differ, REJECT — the macaroon has been tampered with.

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

### Verification Pseudocode

```
function verify(macaroon, root_key_store, satisfiers, request_ctx):
    # Step 1: Signature verification.
    root_key = root_key_store.get(sha256(macaroon.identifier))
    if root_key is None:
        return REJECT("unknown or revoked macaroon")

    sig = HMAC-SHA256(root_key, macaroon.identifier)
    for caveat in macaroon.caveats:
        sig = HMAC-SHA256(sig, caveat)

    if not constant_time_equal(sig, macaroon.signature):
        return REJECT("signature mismatch")

    # Step 2: Payment verification.
    payment_hash = macaroon.identifier[2:34]
    preimage = extract_preimage(macaroon, request_ctx)
    if sha256(preimage) != payment_hash:
        return REJECT("invalid preimage")

    # Step 3: Caveat verification.
    # Group caveats by condition.
    groups = group_by(macaroon.caveats, key=condition_of)
    for condition, caveat_list in groups:
        satisfier = satisfiers.get(condition)
        if satisfier is None:
            continue  # Unknown caveats are skipped.

        for i in range(1, len(caveat_list)):
            if not satisfier.satisfy_previous(caveat_list[i-1], caveat_list[i]):
                return REJECT("caveat not more restrictive")

        if not satisfier.satisfy_final(caveat_list[-1], request_ctx):
            return REJECT("caveat not satisfied")

    return ACCEPT
```

## Revocation

To revoke a macaroon, delete its corresponding root key from storage. Without
the root key, the minter cannot verify the signature, and the macaroon is
permanently invalid.

Revocation is also used during tier upgrades: the old macaroon is revoked
and a new one is minted with updated capabilities.

## Serialization Formats

### Macaroon V2 Binary Format

L402 uses the **V2** compact binary format for macaroon serialization. The
format is TLV-like, using single-byte type tags followed by variable-length
data encoded with unsigned varints.

#### Varint Encoding

Multi-byte lengths and field sizes use **unsigned variable-length integers**
(varints), encoded as in Protocol Buffers:

- Each byte uses its lower 7 bits for data and the high bit as a continuation
  flag.
- If the high bit is `1`, another byte follows. If `0`, this is the last byte.
- Bytes are ordered least-significant first (little-endian varint).

Examples:
- `0x05` encodes 5 (single byte, high bit clear).
- `0x80 0x01` encodes 128 (first byte: 0 with continuation; second byte: 1).

#### V2 Structure

A V2 macaroon is a sequence of typed fields:

| Tag | Value | Meaning |
|-----|-------|---------|
| `0x02` | — | Version indicator (V2 format). No data follows. |
| `0x06` | varint length + bytes | **Location** (optional, UTF-8 string). |
| `0x02` | varint length + bytes | **Identifier**. |
| `0x00` | — | End-of-section marker (ends the macaroon header). |

After the header, each first-party caveat is encoded as:

| Tag | Value | Meaning |
|-----|-------|---------|
| `0x05` | varint length + bytes | Caveat **location** (optional). |
| `0x01` | varint length + bytes | Caveat **identifier** (the caveat string). |
| `0x02` | varint length + bytes | Caveat **verification ID** (empty for first-party). |
| `0x00` | — | End-of-section marker (ends this caveat). |

After all caveats, the signature is encoded as:

| Tag | Value | Meaning |
|-----|-------|---------|
| `0x06` | varint length + bytes | **Signature** (32 bytes for HMAC-SHA256). |
| `0x00` | — | End-of-section marker (ends the macaroon). |

#### Field Encoding

Each field (except bare markers) is encoded as:

```
tag_byte || varint(data_length) || data_bytes
```

The `0x00` end-of-section markers and the initial `0x02` version indicator are
bare single-byte tags with no length or data.

#### First-Party vs Third-Party Caveats

L402 uses only **first-party caveats**. A first-party caveat has a caveat
identifier (the condition string) and an empty verification ID. Third-party
caveats (used for cross-service delegation) have non-empty verification IDs and
are outside the scope of this specification.

#### Encoding Example

A macaroon with identifier `"test-id"` and one caveat `"account=12345"`:

```
0x02                          # V2 format marker
0x02 0x07 "test-id"           # Identifier: tag=0x02, length=7, data
0x00                          # End of header section

0x01 0x0e "account=12345"     # Caveat ID: tag=0x01, length=14, data
0x02 0x00                     # Verification ID: tag=0x02, length=0 (first-party)
0x00                          # End of caveat section

0x06 0x20 <32 signature bytes> # Signature: tag=0x06, length=32
0x00                          # End of macaroon
```

### Wire Format \(Token Storage\)

For client-side storage and transmission, the full token includes:

| Field | Size | Encoding |
|-------|------|----------|
| macaroon length | 4 bytes | uint32 big-endian |
| macaroon bytes | variable | V2 binary format (see above) |
| payment hash | 32 bytes | raw bytes |
| preimage | 32 bytes | raw bytes \(zero if pending\) |
| amount paid | 8 bytes | uint64 big-endian \(millisatoshis\) |
| routing fee paid | 8 bytes | uint64 big-endian \(millisatoshis\) |
| time created | 8 bytes | int64 big-endian \(UnixNano\) |

### HTTP Header Encoding

- `WWW-Authenticate`: macaroon is base64-encoded (standard, with padding)
- `Authorization`: macaroon is base64-encoded, preimage is hex-encoded
- gRPC `Custom-Metadata` \("token" or "macaroon"\): macaroon is hex-encoded

## Reference Implementations

- [gopkg.in/macaroon.v2](https://pkg.go.dev/gopkg.in/macaroon.v2) — Go macaroon library
- [libmacaroons](https://github.com/rescrv/libmacaroons) — C reference implementation
- [pymacaroons](https://github.com/ecordell/pymacaroons) — Python macaroon library
- [Aperture L402 package](https://github.com/lightninglabs/aperture/tree/master/l402) — Reference L402 implementation
