# Introduction

In this document we aim to specify a protocol standard of what we call an `L402`. `L402` is derived from HTTP status code 402: Payment Required. L402 is a new standard protocol for authentication and paid APIs developed by Lightning Labs. An L402 can serve both as authentication, as well as a payment mechanism \(one can view it as a ticket\) for paid APIs. By leveraging L402, a service or business is able to offer a new tier of paid APIs that sit between free, and subscription: pay as you go.

One can view L402 as a fancy authentication credential or cookie. They differ from regular cookies in that they're a cryptographically verifiable bearer credential. An L402 credential _encodes_ all its capabilities within a macaroon which can only be created by the end service provider. The L402 specification uses a combating of `HTTP` as well as the Lightning Network to create a seamless end-to-end payment+authentication flow for the next-generation of paid APIs built on top of the Lightning Network.

The system described above isn't a fantasy, L402 is used _today_ by Lightning Labs to serve as an authentication+payment solution for Lightning Loop, a non-custodial on/off ramp for the Lightning Network and Lightning Pool, a non-custodial market place for liquidity in the Lightning Network. Lightning Labs, has also open sourced `aperture`, a reference L402 aware reverse-proxy used in production for all our systems. In the remainder of this section, we'll explore the motivation, lineage, and workflow of L402 at a high level. For a more detailed speciation, please see the later sections of this specification.

## The Forgotten HTTP Error Code

HTTP as we know it today uses a number of _error_ codes to allow developers to easily consume APIs created by service providers. As an example, the well known `200 OK` error code indicates a successful HTTP response. The `401 Unauthorized` is sent when a client attempts to access a page or resource that requires authentication, and so on. A large number of other error code exist, with some more commonly used than others. One error code which has widely been underutilized is: `402 Payment Required`. As the name entails, this code is returned when a client attempts to access a resource that they haven't _paid for_ yet. In most versions of the HTTP specification, this code is marked as being "reserved for future use". Many speculate that it was intended to be used by some sort of digital cash or micropayment scheme, which didn't yet exist at the time of the initial HTTP specification drafting.

However, several decades later, we _do_ have a widely used digital cash system: Bitcoin! On top of that, a new network oriented around micropayments has also arisen: the Lightning Network. Early in the lifetime of Lightning Labs, we were drawn to the potential for paid metered APIs enabled by the Lightning Network. We'd solved the payment portion with LN itself, the next challenge was to create a protocol that would be easy to drop into _existing_ APIs in an easy and extensible manner. Our solution to this is the L402 protocol.

## Authentication and API Payments in a Lightning-Native Web

Lightning has the potential to serve as the de-facto payment method to access services and resources on the web. In this new web, rather than a user being tracked across the web with invisible pixels to serve invasive ads, or users needing to give away their emails subjecting themselves to a lifetime of spam and tracking, what if a user was able to _pay_ for a service and in the process obtain a ticket/receipt which can be used for future authentication and access?

In this new web, email addresses and passwords are a thing of the past. Instead _cryptographic bearer credentials_ are purchased and presented by users to access services and resources. In this new web, credit cards no longer serve as a gatekeeper to all the amazing experiences that have been created. L402 enables the creation of a new more global, more private, more developer friendly web.

At this point, curious users may be wondering: How would such a scheme work? Are the payment and receipt steps atomic? Why can't a user just forge one of these "tickets"?

## HTTP + Macaroons + Lightning = L402

An L402 is essentially a ticket obtained over Lightning for a particular service or resource. The ticket itself _encodes_ what resource it's able to access. It can be copied, or given to a friend so they can access that same resources. It can also be _attenuated_ to provide a friend access to a slightly weaker version of that resource \(able to stream video at only 480p as an example\). On the other end, services can mint special tickets for particular users, rotate, upgrade, and even revoke the tickets.

The tickets themselves are actually _macaroons_. Macaroons are a flexible standard for API credentials which are already used by `lnd` as its default authentication mechanism. The L402 protocol allows a user to _atomically_ purchase one of these tickets for sats over the Lightning Network. Partial L402 are served over HTTP \(or HTTP/2\) when a user attempts to access a resource that requires payment \(`402 Payment Required`\) along with a Lightning _invoice_. This _partial_ L402 can then be converted into a _complete_ L402 by paying the invoice, and obtaining the payment pre-image \(the invoice pays to a payment hash: `payment_hash = sha256(pre_image)`\).

With proper integration at end clients, Lightning wallets, mobile application, browsers \(and extensions\), the above flow has potential to be even more seamless than the credit card flow users are accustomed to today. It's also more _private_ as the server doesn't need to know _who_ paid for the ticket, only that it was successfully paid for.

## Example Applications and Use Cases

The L402 standard enables a number of new use cases, pricing models, and applications to be built, all using the Lightning Network as a primary money rail. As the standard is also defined over _HTTP/2_, it can be naturally extended to also support gating access to existing _gRPC_ services. This is rather powerful as it enables a _strong decoupling_ of authentication+payment logic from application logic. Today Lightning Loop uses L402s in this very manner to provide a lightweight authentication mechanism for our users.

As L402 leverages the Lightning Network for its payment capabilities, they also enable the easy creation of _metered_ APIs. A metered API is one where the user is able to pay for the target resource or service as they go rather than needing to commit to a subscription up front. Developers can use L402 to create applications that charge users on an on going basis for resources like compute or disk space. If the user stops paying, then the resource can be suspended, collected, and re-allocated for another paying user. Once again, as the standard supports _gRPC_ which supports _bi directional streaming_ APIs, one could even create a metered streaming video or audio service as well!

Additionally, L402 also enables innovation at the API _architecture_ level. One example is automated tier upgrades. Many APIs typically offer several tiers which allow users to gain access to more or additional resources as they climb up the ladder. Typically, a user must _manually_ navigate a web-page to request an upgrade to a higher tier, or downgrade to a lower tier. With the L402 standard, tier upgrades can easily be automated: the user hits a new endpoint to obtain an _upgraded_ L402 which _encodes_ additional functionality or increased resource access compared to the prior tier. Services can even leverage L402 for A/B Testing by giving subsets of users a distinct L402 which when submitted to the service, render a slightly different version of the target resource or service.

## Conclusion

In this section we've provided an overview of the lineage, motivation, workflow, potential and uses cases for the L402 standard. In the later sections, we'll dive a bit deeper, fully specifying the L402 protocol end to end.

