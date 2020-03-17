# Macaroon Minting & Verification

## Overview

In this chapter, we outline the specification of the macaroon component of a Lightning Service Authentication Token \(LSAT\). To recap, an LSAT is composed of a macaroon, which specifies the allowed capabilities of the token, and a preimage, which serves as the token’s proof of payment. Macaroons are perfect candidates for LSATs as they are tamper-proof, support easy rotation, have attributes, and can even be further attenuated in order for applications that integrate an LSAT enabled service to delegate any additional capabilities. We’ll cover how macaroons are created, attenuated, and verified as part of an LSAT. This chapter will require an understanding of how macaroons work and how they are useful in the context of authentication. It may be useful to skim the [introductory research paper on macaroons](https://research.google/pubs/pub41892/).

## Minting Macaroons

Macaroons have three basic components: a public identifier that maps to a root key, a list of caveats \(which is how attenuation is achieved\), and a signature. Minting a new macaroon only requires the public identifier and the root key it corresponds to.

Each macaroon must have its own cryptographically random root key that must not be disclosed to prevent forgeability of macaroons. Each root key consists of 32 bytes, which provides a reasonable tradeoff between entropy and size. Root keys are essential to the verification of macaroons, so they must be stored securely and reliably.

### Macaroon Identifier

The key mapping to the secret is the SHA-256 hash of the macaroon’s public identifier, which contains static information about the macaroon itself. In the initial construction, the public identifier should include the following in the listed order:

* **Version** - A version allows for an iterative macaroon design.
* **Payment Hash** - A payment hash links an invoice’s payment request to the macaroon. Once the payment request is fulfilled, the payer receives its corresponding preimage as proof of payment. This proof can then be provided along with the macaroon to ensure an LSAT has been paid for without checking whether the invoice has been fulfilled.
* **User Identifier** - A unique user identifier allows services to track users across distinct macaroons serving useful in the context of service level metering. A user identifier of 32 random bytes is used instead of the macaroon’s identifier because the latter can be revoked, e.g., in the case of a service tier upgrade.

## Attenuation Through Caveats

Caveats are predicates that restrict a macaroon’s authority, as well as the context in which it may be successfully used. When verifying a macaroon, each caveat predicate must be evaluated and hold true. Due to their flexibility, additional context found within the request to a service may be necessary for proper evaluation.

Caveats of a version 0 macaroon are represented as key-value pairs encoded as a string where the key and value are separated by a single `=` character. The key uniquely identifies a caveat and provides context as to how it should be evaluated, while the value provides context for the evaluation itself. There aren't any further restrictions on how caveats should be formed, but Lightning Labs services will mostly impose three types of caveats which are covered below.

### Target Services

The target services the macaroon is allowed to access is represented as a caveat. The caveat key is `services`, while the value consists of a comma-separated list of target services. Each target service is composed of a two-tuple consisting of the service name and its tier. Tiers are service specific and must start from 0, which serves as the base tier. If a specific service tier has its capabilities and/or constraints updated, there needs to be a way to detect when a macaroon of the same tier with the now outdated capabilities and/or constraints is being used. By committing to the service tier, it is possible to detect such cases and seamlessly upgrade the stale macaroon \(assuming it is valid\) by revoking it and minting a new macaroon with the newer capabilities and/or constraints.

When verifying this caveat, if a macaroon is attempting to access a service that it does not commit to, then it should be considered invalid. If multiple services caveats exist, then verification should ensure each occurrence of the caveat restricts more access than the previous.

### Service Capabilities

Each service can further be restricted by the capabilities the service provides and these are also represented as another caveat. These caveats have a key ending in `_capabilities` that is prefixed with the service name, while the value consists of a comma-separated list of the allowed service capabilities. This type of caveat allows certain macaroons to only have access to a subset of a service's features.

If a capabilities caveat for a particular service is not present, then the macaroon is able to access any capability of the service. If multiple capabilities caveats exist for the same service, then verification should ensure each occurrence of the caveat restricts more access than the previous.

### Service Constraints

Each service can define its own set of constraint caveats for a given tier to further restrict the capabilities of a macaroon. Each constraint takes the form of a caveat, where the key is prefixed with the service capability it applies to, and the remainder of the key includes context for the service on how to evaluate the constraint. The caveat value specifies the parameters for the constraint evaluation.

If multiple caveats of the same constraint are found within a macaroon, then verification should ensure each occurrence of the constraint restricts more access than the previous.

As an example, a base tier Lightning Loop macaroon with access to a 2 BTC monthly volume for Loop Out and unlimited volume for Loop In would look like:

```text
identifier:
    version = 0
    user_id = fed74b3ef24820f440601eff5bfb42bef4d615c4948cec8aca3cb15bd23f1013
    payment_hash = 163102a9c88fa4ec9ac9937b6f070bc3e27249a81ad7a05f398ac5d7d16f7bea
caveats:
    services = lightning_loop:0
    lightning_loop_capabilities = loop_out,loop_in
    loop_out_monthly_volume_sats = 200000000
```

Due to the flexibility of the design, a macaroon holder is able to further attenuate a macaroon if they wish to share it with a third party under more restrictive permissions. Following the example above, the macaroon holder can restrict the macaroon’s capabilities to only allow access to Loop In \(and not Loop Out\) with a monthly volume of 1 BTC by adding the following caveats:

```text
lightning_loop_capabilities = loop_in
loop_in_monthly_volume_sats = 100000000
```

## Macaroon Verification

Verifying a macaroon consists of a three step process and involves two parties: the minter of the macaroon and the authorizer, which is the service the macaroon targets. The minter of the macaroon performs signature checks to ensure the macaroon is valid, while the authorizer ensures the macaroon has the required permissions to access the service.

The first step ensures a macaroon was minted by the minter and it has not been tampered with. This is done by computing the HMAC chain of the macaroon, starting from its root key and identifier, and including any further attenuation through its caveats to arrive at the terminal signature of the macaroon. If these do not match, the macaroon is invalid. If there isn’t a valid root key corresponding to the macaroon, it is also considered invalid.

The second step ensures a macaroon has provided a valid proof of payment \(preimage\) and is performed by the minter as well. Since macaroons commit to the payment hash of an invoice, this is a trivial step.

The final step ensures a macaroon is authorized to access a service. This is done by ensuring the service-specific caveat predicates of a macaroon hold true for the service being accessed. If only one of these caveats doesn’t hold true, then the macaroon is invalid. In the case of an unknown caveat, its evaluation must be skipped by the authorizer as the macaroon holder can further attenuate the macaroon for other applications.

## Macaroon Revocation

To prevent abusers of a macaroon-based authenticated service, a macaroon should be able to be revoked. This can be achieved by having the minter remove the macaroon’s corresponding root key. By doing so, the minter will never be able to verify the signature of a revoked macaroon, ensuring it will never reach its targeted service.

Revocation also serves useful when performing a service tier upgrade on a macaroon. The prior macaroon is revoked to ensure only the upgraded one can be used going forward.

