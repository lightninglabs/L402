# Protocol Specification

## Introduction

In this chapter, we outline the specification for the abstract L402 HTTP and gRPC protocols. This is intended to be along the lines of the document we would submit if we were submitting the L402 HTTP/gRPC protocol to a standards committee. For more details on the higher-level purpose and motivations behind L402, please [see this chapter](introduction.md).

## Specification

This section defines the "L402" authentication scheme, which transmits credentials as `<macaroon(s)>:<preimage>` pairs, where the preimage is encoded as hex and the Macaroon is encoded as base64. Multiple Macaroons are base64 encoded individually and listed comma separated before the colon.  
This scheme is not considered to be a secure method of user authentication unless used in conjunction with some external secure system such as TLS, as the Macaroon and preimage are passed over the network as cleartext.

The L402 authentication scheme is based on the model that the client needs to authenticate itself with a Macaroon and invoice preimage for each backend service it wants to access. The server will service the request only if it can validate the Macaroon and preimage for the particular backend service requested.

The L402 authentication scheme utilizes the Authentication Framework specified in [_RFC 7235_](https://tools.ietf.org/html/rfc7235) as follows.

In challenges: the scheme name is "L402". Note that the scheme name is case-insensitive. For credentials, the syntax is:

`macaroons` → [_&lt;base64 encoding&gt;_](https://tools.ietf.org/html/rfc3548#section-3), comma separated if multiple Macaroons are present  
`preimage` → [_&lt;hex encoding&gt;_](https://tools.ietf.org/html/rfc3548#section-6)  
`token` → macaroons ":" preimage

Specifically, the syntax for "token" specified above is used, which can be considered comparable to the [_"token68" syntax_](https://tools.ietf.org/html/rfc7235#section-2.1) used for HTTP basic auth.

### Reusing Credentials

L402 is intended to be reused until they are revoked and the server issues a new challenge in response to a client request containing a newly invalid L402. Possible revocation conditions include: expiry date, exceeded N usages, volume of usages in a certain time period necessitating a tier upgrade, and potentially others \(discussed further in the higher-level design document\).

L402  could be configured for use on a per-backend-service basis or for all Lightning Labs services. I.e., it’s flexible whether an L402 could apply to both the Bos score API _and_ a loop-in, or just one of them. This flexibility is afforded because all services are going to be gated by the same L402 proxy, which verifies all Macaroons for all backend services.

### Security Considerations

If a client’s L402 is intercepted by Mallory, which is possible if the transmission is not encrypted in some way such as TLS, the L402 can be used by Mallory and the L402 proxy would not be able to distinguish this usage as illicit.

L402 authentication is also vulnerable to spoofing by counterfeit servers. If a client slightly mistypes the URL of a desired backend service, they become vulnerable to spoofing attacks if connecting to a server that maliciously stores their L402 and uses it for their own purposes. This attack could be addressed by requiring the user of the L402 to have a specific IP address. However, there are downsides to this approach; for example, if a user switches WiFi networks, their credential becomes unusable.

## HTTP Specification

In this section, we specify the protocol for the HTTP portion of the L402 proxy.

Upon receipt of a request for a URI of an L402-proxied backend service that lacks credentials or contains an L402 that is invalid or insufficient in some way, the server should reply with a challenge using the 402 \(Payment Required\) status code. **Officially, in the HTTP RFC documentation, status code 402 is**  [_**"reserved for future use"**_](https://tools.ietf.org/html/rfc7231#section-6.5.2)  **-- but this document assumes the future has arrived.**

Alongside the 402 status code, the server should specify the `WWW-Authenticate` header \([_\[RFC7235\], Section 4.1_](https://tools.ietf.org/html/rfc7235#section-4.1)\) field to indicate the L402 authentication scheme and the Macaroon needed for the client to form a complete L402.

For instance:

```text
 HTTP/1.1 402 Payment Required

Date: Mon, 04 Feb 2014 16:50:53 GMT

WWW-Authenticate: L402 macaroon="AGIAJEemVQUTEyNCR0exk7ek90Cg==", invoice="lnbc1500n1pw5kjhmpp5fu6xhthlt2vucmzkx6c7wtlh2r625r30cyjsfqhu8rsx4xpz5lwqdpa2fjkzep6yptksct5yp5hxgrrv96hx6twvusycn3qv9jx7ur5d9hkugr5dusx6cqzpgxqr23s79ruapxc4j5uskt4htly2salw4drq979d7rcela9wz02elhypmdzmzlnxuknpgfyfm86pntt8vvkvffma5qc9n50h4mvqhngadqy3ngqjcym5a"
```

where `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` is the Macaroon that the client must include for each of its authorized requests and `"lnbc1500n1pw5kjhmpp..."` is the invoice the client must pay to reveal the preimage that must be included for each of its authorized requests.

In other words, to receive authorization, the client:

1. Pays the invoice from the server, thus revealing the invoice’s preimage
2. Constructs the L402 by concatenating the base64-encoded Macaroon\(s\), a single

   colon \(":"\), and the hex-encoded preimage.

Since the Macaroon and the preimage are both binary data encoded in an ASCII based format, there should be no problem with either containing control characters or colons \(see "CTL" in [_Appendix B.1 of \[RFC5234\]_](https://tools.ietf.org/html/rfc5234#appendix-B.1)\). If a user provides a Macaroon or preimage containing any of these characters, this is to be considered an invalid L402 and should result in a 402 and authentication information as specified above.

If a client wishes to send the Macaroon `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` \(already base64-encoded by the server\) and the preimage `"1234abcd1234abcd1234abcd"` \(already hex encoded by the payee's Lightning node\), they would use the following header field:

```text
Authorization: L402 AGIAJEemVQUTEyNCR0exk7ek90Cg==:1234abcd1234abcd1234abcd
```

## gRPC Protocol Specification

This section defines the "L402" gRPC authentication scheme, which, similarly to the HTTP version, transmits credentials as `<macaroon(s)>:<preimage>` pairs where the preimage is encoded as hex and the Macaroon is encoded as base64. Multiple Macaroons are base64 encoded individually and listed comma separated before the colon. As above, this scheme is not considered to be a secure method of user authentication unless used in conjunction with some external secure system such as TLS, as the Macaroon and preimage are passed over the network as cleartext.

The L402 proxy will determine whether an incoming HTTP request is gRPC by checking whether the Content-Type header begins with application/grpc, therefore gRPC clients must set this header in all requests.

Note that the L402 proxy must be HTTP/2 compatible to accommodate requests for gRPC backend services, since the gRPC client expects to be talking to a server that "speaks" HTTP/2.

Upon receipt of a request for a URI of an L402-proxied backend service that lacks L402 credentials, the server should reply with a challenge encoded in the grpc-status-details-bin HTTP header as a serialized gRPC Status proto message, to be deserialized on the client side. Once deserialized, the proto will look roughly like this object:

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

Note that deserialization is language-dependent. In Go, it looks something like this:

```go
_, err := client.AccessBackendService(ctx, &pb.BackendServiceRequest{})
If err != nil {
        st, _ := status.FromError(err)
        message := st.Message() // get message
        code := st.Code() // get code
        for _, detail := range st.Details() {
                switch t := detail.(type) {
                case *errdetails.QuotaFailure:
                for _, violation := range t.GetViolations() {
                        // parse macaroon from "macaroon:&lt;mac&gt;" format
                        // parse invoice from "invoice:&lt;inv&gt;" format
…
```

Serialization is similarly language-dependent.

Depending on the context, QuotaFailure may not be the most descriptive error message, but it fits a scenario where a user has exceeded their free "trial period" for a backend service.

Alongside the serialized status details, the server should specify status code `200 OK`, the `Content-Type` header, and the following trailers: `grpc-message` and `grpc-status`.

For instance:

```text
HTTP/2 200 OK
Date: Mon, 04 Feb 2014 16:50:53 GMT
Content-Type: application/grpc
…
Grpc-Message: missing L402
Grpc-Status: 402
Grpc-Status-Details-Bin: CJIDEgxtaXNzaW5nIExTQVQaeQ…
```

Where `"CJIDEgxtaXNzaW5nIExTQVQaeQ…"` is the serialized gRPC status proto.

Once the client has deserialized the proto and extracted the Macaroon and invoice, they may pay the invoice and construct the L402 identically to the HTTP specification, i.e. by concatenating the base64-encoded Macaroon, a single colon \(":"\), and the hex-encoded preimage.

If a client wishes to send the Macaroon `"AGIAJEemVQUTEyNCR0exk7ek90Cg=="` \(already base64-encoded by the server\) and the preimage `"1234abcd1234abcd1234abcd"` \(already hex encoded by the payee's Lightning node\), they would use the following header field:

```text
Authorization: L402 AGIAJEemVQUTEyNCR0exk7ek90Cg==:1234abcd1234abcd1234abcd
```

Note this is the same as the HTTP specification. Other gRPC headers and trailers are required; more information can be found in the [_gRPC over HTTP2 specification_](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md).

